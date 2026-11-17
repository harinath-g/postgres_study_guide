# Advanced PostgreSQL Production Engineering Curriculum

**Audience:** Software/Data Engineer with ~5 years of engineering experience.  
**Outcome:** Operate, debug, recover, and scale mission-critical PostgreSQL systems.  
**Version baseline:** PostgreSQL 15–17; verify version-specific behavior before production changes.

This curriculum deliberately bypasses entry-level SQL. Labs should use a disposable primary/replica cluster, PgBouncer, production-like data distributions, and metrics/logging.

---

## Module 1 — Process Architecture, Memory, and Connection Lifecycle

### 1. Core Architectural Concepts & Mechanics

PostgreSQL is process-per-connection. The postmaster accepts a connection and manages backend processes, which coordinate through shared memory.

```text
Client → Postmaster → Backend
                       ├─ parser → rewriter → planner → executor
                       ├─ shared_buffers → cached data pages
                       ├─ WAL buffers → WAL writer → pg_wal
                       └─ local memory: work_mem, temp_buffers

Background: checkpointer | bgwriter | WAL writer | autovacuum | archiver
```

- `shared_buffers` is the PostgreSQL shared page cache; OS page cache also matters.
- `work_mem` is per sort/hash operation, not per session.
- WAL flush behavior and `synchronous_commit` determine commit durability.
- A pooler prevents backend process and memory explosion.

### 2. Day-to-Day Operational Use Cases

- Set a connection budget across apps, workers, BI, admin, replication, and autovacuum.
- Use transaction pooling for stateless applications.
- Model maximum concurrent executor memory before changing `max_connections` or `work_mem`.
- Separate application, reporting, migration, and batch roles/pools.

### 3. Production Incidents & Mitigation Steps

**Connection exhaustion / storm**

```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY 1 ORDER BY 2 DESC;
SELECT usename, application_name, client_addr, count(*)
FROM pg_stat_activity GROUP BY 1,2,3 ORDER BY 4 DESC;
```

1. Preserve an admin connection with `superuser_reserved_connections`.
2. Identify the source deployment or job.
3. Terminate only verified noncritical idle connections:

```sql
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE state = 'idle' AND usename = 'app_user' AND backend_type = 'client backend';
```

4. Rate-limit/rollback the source and correct pool sizing.
5. Add safeguards: `idle_in_transaction_session_timeout`, `statement_timeout`, and `lock_timeout`.

**OOM caused by per-backend memory**

```sql
SELECT pid, usename, state, query FROM pg_stat_activity WHERE state = 'active';
SELECT datname, temp_files, pg_size_pretty(temp_bytes) FROM pg_stat_database;
```

Cancel the offending query first with `SELECT pg_cancel_backend(<pid>);`; terminate only if necessary with `pg_terminate_backend`. Reduce role-specific `work_mem`, then recalculate concurrency capacity.

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_stat_activity`, `pg_stat_database`, `pg_stat_bgwriter`, `pg_stat_io`, `pg_settings`, `pg_roles`.  
**Configuration:** `max_connections`, `superuser_reserved_connections`, `shared_buffers`, `work_mem`, `maintenance_work_mem`, `temp_buffers`, `effective_cache_size`, `idle_in_transaction_session_timeout`, `statement_timeout`, `log_connections`, `log_disconnections`, `log_line_prefix`.

---

## Module 2 — MVCC, Visibility, Locks, and Concurrency

### 1. Core Architectural Concepts & Mechanics

MVCC retains row versions and evaluates them against a transaction snapshot.

```text
UPDATE row
  old tuple: xmin=100, xmax=250
  new tuple: xmin=250, xmax=0

Reader sees a tuple if xmin committed before its snapshot
and xmax is absent or not committed before that snapshot.
```

```text
SQL operation
 ├─ heavyweight relation/transaction/advisory locks
 ├─ row-level locks
 ├─ LWLocks for shared internal structures
 └─ predicate locks for SERIALIZABLE isolation
```

Long transactions retain old snapshots and stop vacuum from removing obsolete versions.

### 2. Day-to-Day Operational Use Cases

- Keep request transactions short and prevent `idle in transaction` sessions.
- Use `READ COMMITTED` by default; adopt stronger isolation deliberately.
- Implement retry policy for serialization failures/deadlocks.
- Apply `lock_timeout` to migrations and use advisory locks for singleton workers.

### 3. Production Incidents & Mitigation Steps

**Lock queue blocks traffic**

```sql
SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid,
       blocked.query AS blocked_query, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid AND NOT bl.granted
JOIN pg_locks kl ON kl.locktype = bl.locktype
 AND kl.database IS NOT DISTINCT FROM bl.database
 AND kl.relation IS NOT DISTINCT FROM bl.relation
 AND kl.pid <> bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = kl.pid;
```

1. Identify the object, blocker, transaction age, and owner.
2. Cancel safely: `SELECT pg_cancel_backend(<blocking_pid>);`.
3. If needed, terminate after assessing rollback impact: `SELECT pg_terminate_backend(<blocking_pid>);`.
4. Prevent recurrence with migration lock budgets and concurrent index builds.

**Long transaction prevents vacuum**

```sql
SELECT pid, usename, state, xact_start, now()-xact_start AS age, backend_xmin, query
FROM pg_stat_activity WHERE xact_start IS NOT NULL ORDER BY xact_start;
```

Verify the session is not required, have it commit/rollback or terminate it, then run `VACUUM (VERBOSE, ANALYZE) public.affected_table;`.

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_stat_activity`, `pg_locks`, `pg_class`, `pg_namespace`, `pg_stat_database_conflicts`, `pg_prepared_xacts`.  
**Configuration:** `default_transaction_isolation`, `lock_timeout`, `deadlock_timeout`, `idle_in_transaction_session_timeout`, `max_locks_per_transaction`, `max_pred_locks_per_transaction`, `log_lock_waits`.

---

## Module 3 — WAL, Checkpointing, Durability, and Crash Recovery

### 1. Core Architectural Concepts & Mechanics

```text
INSERT/UPDATE
  │
  ├─ dirty shared-buffer page → bgwriter/checkpointer → relation files
  └─ WAL record → WAL buffers → WAL writer → pg_wal → fsync

Crash: control file + checkpoint → replay later WAL → consistent cluster
```

WAL reaches durable storage before changed data pages. Checkpoints establish recovery points; poor pacing creates storage write storms. Full-page writes guard against torn pages.

### 2. Day-to-Day Operational Use Cases

- Measure WAL generation and retention.
- Tune checkpoints against actual write rate and recovery objective.
- Separate WAL/data storage when the platform supports it.
- Size archival and replication throughput from WAL volume.

### 3. Production Incidents & Mitigation Steps

**`pg_wal` fills a volume**

```sql
SELECT slot_name, slot_type, active, restart_lsn, confirmed_flush_lsn,
 pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
SELECT application_name, state, sync_state, sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
```

1. Identify lagging replicas or inactive slots.
2. Restore a required consumer or replica.
3. Drop only a verified obsolete slot: `SELECT pg_drop_replication_slot('obsolete_slot');`.
4. Never manually delete files from `pg_wal`.
5. Alert on retention and capacity; set a conscious `max_slot_wal_keep_size` limit.

**Checkpoint I/O storm**

```sql
SELECT checkpoints_timed, checkpoints_req, checkpoint_write_time,
 checkpoint_sync_time, buffers_checkpoint, buffers_clean, maxwritten_clean
FROM pg_stat_bgwriter;
```

Confirm correlation, then tune incrementally:

```conf
max_wal_size = '16GB'
checkpoint_timeout = '15min'
checkpoint_completion_target = 0.9
```

Reload with `SELECT pg_reload_conf();` and observe a complete workload cycle.

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_stat_bgwriter`, `pg_stat_wal`, `pg_stat_replication`, `pg_replication_slots`, `pg_control_checkpoint()`, `pg_current_wal_lsn()`.  
**Configuration:** `wal_level`, `synchronous_commit`, `fsync`, `full_page_writes`, `wal_compression`, `max_wal_size`, `min_wal_size`, `checkpoint_timeout`, `checkpoint_completion_target`, `wal_buffers`, `archive_mode`, `archive_command`.

---

## Module 4 — Autovacuum, Bloat, Freezing, and Transaction ID Safety

### 1. Core Architectural Concepts & Mechanics

```text
MVCC update/delete → dead tuples
  ├─ VACUUM: reusable space, index cleanup, visibility map
  ├─ ANALYZE: refreshed planner statistics
  └─ FREEZE: transaction-ID wraparound protection
```

Autovacuum thresholds are approximately `autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor × estimated rows)`. High-churn large tables require per-table settings. Standard vacuum usually does not return disk space to the OS.

### 2. Day-to-Day Operational Use Cases

- Tune high-churn tables individually.
- Track freeze age, dead tuples, autovacuum duration, and bloat.
- Run `VACUUM (ANALYZE)` after substantial modifications.
- Plan online table rewrites with approved tooling such as `pg_repack`.

### 3. Production Incidents & Mitigation Steps

**Transaction ID wraparound risk**

```sql
SELECT datname, age(datfrozenxid) AS xid_age,
       2147483647-age(datfrozenxid) AS xid_remaining
FROM pg_database ORDER BY xid_age DESC;

SELECT n.nspname, c.relname, age(c.relfrozenxid) AS xid_age,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE c.relkind IN ('r','m') ORDER BY age(c.relfrozenxid) DESC LIMIT 30;
```

1. Treat as an availability emergency.
2. Clear safe blockers: long transactions, stale slots, prepared transactions.
3. Vacuum oldest relations first: `VACUUM (FREEZE, VERBOSE, ANALYZE) public.critical_table;`.
4. If necessary, run `VACUUM (FREEZE, VERBOSE);` database-wide.
5. Temporarily raise measured capacity through `autovacuum_max_workers` and `autovacuum_vacuum_cost_limit`.

**Severe bloat**

```sql
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 30;
```

Find retention blockers and churn source, tune autovacuum, run standard vacuum for reusable space, and plan an online rewrite or maintenance-window `VACUUM FULL` only when OS-level reclamation is necessary.

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_stat_user_tables`, `pg_stat_all_tables`, `pg_stat_progress_vacuum`, `pg_database`, `pg_class`, `pg_stat_user_indexes`, `pg_visibility`.  
**Configuration:** `autovacuum`, `autovacuum_max_workers`, `autovacuum_naptime`, `autovacuum_vacuum_scale_factor`, `autovacuum_analyze_scale_factor`, `autovacuum_vacuum_cost_limit`, `autovacuum_vacuum_cost_delay`, `vacuum_freeze_min_age`, `autovacuum_freeze_max_age`.

---

## Module 5 — Query Planning, Statistics, Indexes, and Performance Forensics

### 1. Core Architectural Concepts & Mechanics

```text
SQL → Parse → Rewrite → Planner → Plan → Executor
                         │
                         ├─ table statistics / MCVs / histograms
                         ├─ index metadata
                         ├─ cost model
                         └─ join alternatives
```

Bad cardinality estimates commonly cause bad plans. `EXPLAIN (ANALYZE, BUFFERS)` compares estimated rows and costs with actual work and I/O.

### 2. Day-to-Day Operational Use Cases

- Use `pg_stat_statements` as the workload inventory.
- Review high total-time, high-call, high-temp-I/O, and high-read queries.
- Design partial, expression, covering, BRIN, and multicolumn indexes from workload shape.
- Use extended statistics for correlated predicates.

### 3. Production Incidents & Mitigation Steps

**Unindexed query saturates CPU/I/O**

```sql
SELECT pid, now()-query_start AS runtime, wait_event_type, wait_event, query
FROM pg_stat_activity WHERE state='active' ORDER BY query_start;
SELECT queryid, calls, total_exec_time, mean_exec_time, rows,
 shared_blks_read, temp_blks_written, query
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 20;
```

Cancel the immediate offender if needed, inspect the controlled plan, then create a validated index concurrently:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;
CREATE INDEX CONCURRENTLY idx_orders_customer_created
ON public.orders (customer_id, created_at DESC);
```

**Stale statistics cause a catastrophic join plan**

```sql
SELECT schemaname, relname, n_live_tup, last_analyze, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_mod_since_analyze DESC;
```

Run `ANALYZE VERBOSE public.target_table;`, raise column statistics where appropriate, and create dependency statistics for correlated filters:

```sql
ALTER TABLE public.target_table ALTER COLUMN status SET STATISTICS 1000;
CREATE STATISTICS target_dependency_stats (dependencies)
ON region, status FROM public.target_table;
ANALYZE public.target_table;
```

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_stat_statements`, `pg_stat_activity`, `pg_stats`, `pg_statistic`, `pg_stat_user_tables`, `pg_stat_user_indexes`, `pg_index`, `pg_indexes`, `pg_stat_progress_create_index`.  
**Configuration:** `shared_preload_libraries`, `pg_stat_statements.track`, `track_io_timing`, `track_wal_io_timing`, `default_statistics_target`, `random_page_cost`, `effective_io_concurrency`, `jit`, `log_min_duration_statement`, `auto_explain.*`.

---

## Module 6 — Schema Change Engineering and Zero-Downtime Delivery

### 1. Core Architectural Concepts & Mechanics

DDL is transactional but can acquire blocking locks, rewrite tables, create WAL volume, and break version compatibility.

```text
Expand → migrate/backfill → verify → contract
1. Add compatible structure
2. Backfill in bounded batches
3. Validate constraints/create indexes concurrently
4. Switch reads/writes
5. Remove deprecated structure later
```

### 2. Day-to-Day Operational Use Cases

- Add columns, indexes, constraints, partitions, and enum values safely.
- Backfill under explicit WAL, replica-lag, lock, and SLO budgets.
- Use `NOT VALID` constraints and later validation.
- Ensure concurrent index creation is not wrapped in a transaction block.

### 3. Production Incidents & Mitigation Steps

**Migration blocks traffic**

```sql
SELECT a.pid, a.query, a.wait_event_type, l.mode, l.granted, c.relname
FROM pg_locks l JOIN pg_stat_activity a ON a.pid=l.pid
LEFT JOIN pg_class c ON c.oid=l.relation
WHERE c.relname='target_table' ORDER BY l.granted, a.query_start;
```

Cancel queued migrations, clear the true blocker, then retry with bounded waits:

```sql
SET lock_timeout = '3s';
SET statement_timeout = '10min';
ALTER TABLE public.target_table ADD COLUMN new_flag boolean;
```

**Backfill overwhelms primary/replication**

Pause the job, then use small committed primary-key batches and throttle on CPU, WAL rate, replica lag, and lock waits:

```sql
UPDATE public.orders SET new_status=status
WHERE id >= :start_id AND id < :end_id AND new_status IS NULL;
```

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_locks`, `pg_stat_activity`, `pg_stat_progress_create_index`, `pg_constraint`, `pg_attribute`, `pg_depend`, `pg_stat_replication`.  
**Configuration:** `lock_timeout`, `statement_timeout`, `maintenance_work_mem`, `max_wal_size`, `wal_compression`, `log_lock_waits`, `deadlock_timeout`.

---

## Module 7 — Replication, High Availability, and Failover Architecture

### 1. Core Architectural Concepts & Mechanics

```text
Primary WAL → WAL sender → network → WAL receiver → standby pg_wal
                                                   → startup process replay
                                                   → read-only replica / promotion candidate

Application → pooler/router → primary endpoint
                           └→ replica endpoint(s)
HA control plane → quorum + fencing + promotion + endpoint update
```

Physical replication streams WAL. Logical replication streams decoded changes for selective distribution and migrations. HA also requires reliable fencing to prevent split brain.

### 2. Day-to-Day Operational Use Cases

- Measure lag in bytes and time.
- Select sync/async replication based on explicit RPO/RTO.
- Test promotion, rejoin, failback, and application reconnect behavior.
- Route reads with a clear staleness policy.
- Manage slots, timelines, and replica rebuilds.

### 3. Production Incidents & Mitigation Steps

**Replica lag threatens RPO**

```sql
SELECT application_name, state, sync_state,
 pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
 write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```

On the standby:

```sql
SELECT pg_is_in_recovery(), now()-pg_last_xact_replay_timestamp() AS replay_time_lag,
       pg_last_wal_replay_lsn();
```

Determine whether the bottleneck is network, receive, replay CPU, storage, or a conflicting replica query. Correct it, reduce write pressure, scale the replica, or rebuild it if it cannot catch up within retention. Do not promote a stale asynchronous standby without explicit data-loss acceptance.

**Split-brain risk during failover**

1. Fence the old primary at network/storage/orchestrator level.
2. Verify only one node accepts writes.
3. Promote through the approved HA control plane and verify `SELECT pg_is_in_recovery();`.
4. Repoint writers via a stable endpoint.
5. Rebuild/rejoin the old primary; never simply re-enable it as writable.

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_stat_replication`, `pg_stat_wal_receiver`, `pg_replication_slots`, `pg_stat_subscription`, `pg_subscription_rel`, `pg_is_in_recovery()`, `pg_last_wal_receive_lsn()`, `pg_last_wal_replay_lsn()`.  
**Configuration:** `wal_level`, `max_wal_senders`, `max_replication_slots`, `wal_keep_size`, `max_slot_wal_keep_size`, `hot_standby`, `hot_standby_feedback`, `max_standby_streaming_delay`, `synchronous_standby_names`, `primary_conninfo`, `restore_command`.

---

## Module 8 — Backup, PITR, Disaster Recovery, and Validation

### 1. Core Architectural Concepts & Mechanics

```text
Base backup + archived WAL → isolated restored cluster → replay WAL
                                                    ├─ latest consistent state
                                                    └─ target time / LSN / restore point
```

PITR requires a valid base backup and uninterrupted archived-WAL coverage. A backup is only valid operationally once a restore has been tested.

### 2. Day-to-Day Operational Use Cases

- Use pgBackRest, WAL-G, or managed backups with documented WAL semantics.
- Define RPO/RTO per service tier.
- Test restores monthly and after major platform changes.
- Retain logical backups for selective recovery and migration.

### 3. Production Incidents & Mitigation Steps

**Accidental destructive change**

1. Stop the writer/deployment and record incident time in UTC.
2. Do not issue broad corrective writes before agreeing recovery strategy.
3. Restore an isolated cluster from a valid base backup.
4. Replay WAL to immediately before the incident.
5. Validate counts and business invariants, then extract data or execute approved cutover.

Create a restore point before known risky work:

```sql
SELECT pg_create_restore_point('before_critical_migration');
```

**Backup exists but restoration fails**

Treat as a DR-control failure. Capture the archive/permission/encryption/version failure, correct the pipeline, complete a full restoration test, document achieved RTO/RPO, and alert on archive or backup freshness failures.

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_stat_archiver`, `pg_backup_start()`, `pg_backup_stop()`, `pg_control_checkpoint()`, `pg_walfile_name()`, `pg_create_restore_point()`.  
**Configuration:** `archive_mode`, `archive_command`, `archive_timeout`, `restore_command`, `recovery_target_time`, `recovery_target_lsn`, `recovery_target_name`, `recovery_target_action`.

---

## Module 9 — Partitioning, Data Lifecycle, and Large-Table Operations

### 1. Core Architectural Concepts & Mechanics

```text
parent: events
 ├─ events_2026_08
 ├─ events_2026_09
 ├─ events_2026_10
 └─ events_default

INSERT → partition routing → child table/indexes
SELECT with time predicate → partition pruning → selected children
```

Partitioning is a lifecycle/manageability technique, not an automatic replacement for indexing, data modeling, or bounded queries.

### 2. Day-to-Day Operational Use Cases

- Create future partitions before ingestion reaches the boundary.
- Use detach/drop for partition-aligned retention rather than mass deletes.
- Maintain appropriate per-child indexes and statistics.
- Limit partition count to avoid planning overhead.

### 3. Production Incidents & Mitigation Steps

**Missing partition stops inserts**

```sql
SELECT inhrelid::regclass AS partition FROM pg_inherits
WHERE inhparent='public.events'::regclass ORDER BY 1;
```

Create the partition and corresponding indexes/grants:

```sql
CREATE TABLE public.events_2026_10 PARTITION OF public.events
FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
```

Then automate advance partition creation and alert on the newest upper bound.

**Retention delete causes bloat/WAL storm**

Stop broad deletes. Confirm partition alignment, then after retention approval detach/drop expired partitions:

```sql
ALTER TABLE public.events DETACH PARTITION public.events_2025_08;
DROP TABLE public.events_2025_08;
```

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_partitioned_table`, `pg_inherits`, `pg_class`, `pg_constraint`, `pg_stat_user_tables`, `pg_stat_user_indexes`.  
**Configuration:** `constraint_exclusion`, `enable_partition_pruning`, `maintenance_work_mem`, table-level `autovacuum_*`, `default_statistics_target`.

---

## Module 10 — Observability, Capacity, Security, and Operational Governance

### 1. Core Architectural Concepts & Mechanics

```text
Application: connections, latency, errors, query tags
PostgreSQL: transactions, locks, WAL, vacuum, replication, cache
Host/storage: CPU, memory, disk latency/queue, filesystem capacity
Business: freshness, reconciliation, queue depth, data invariants
```

```text
Storage = data + indexes + WAL + backup retention + bloat + maintenance headroom
Memory  = peak concurrency × per-operation memory + shared memory + OS cache
```

### 2. Day-to-Day Operational Use Cases

- Dashboard latency, locks, connections, replication, checkpoints, vacuum, disk, and backup freshness.
- Write alerts that have an owner and a concrete action.
- Run restore, failover, and load tests quarterly.
- Apply least privilege, TLS, secret rotation, and required auditing.
- Keep runbooks, ownership labels, change windows, and migration standards current.

### 3. Production Incidents & Mitigation Steps

**“Database is slow”**

```sql
SELECT pid, usename, state, wait_event_type, wait_event,
 now()-query_start AS runtime, query
FROM pg_stat_activity WHERE state <> 'idle' ORDER BY query_start;
SELECT mode, granted, count(*) FROM pg_locks GROUP BY 1,2 ORDER BY 3 DESC;
SELECT datname, blks_read, blks_hit, xact_commit, xact_rollback,
 temp_files, temp_bytes FROM pg_stat_database;
```

Classify the condition: lock queue, runaway query, I/O, CPU, connection saturation, replication, or maintenance backlog. Stabilize first, preserve evidence, correct root cause, and document prevention.

**Privileged role or endpoint exposure**

Restrict network access, rotate credentials, and inspect active sessions:

```sql
SELECT pid, usename, client_addr, application_name, state, query
FROM pg_stat_activity ORDER BY backend_start DESC;
```

Revoke unneeded privileges, validate role inheritance, preserve logs, and execute organizational incident response.

### 4. System References & Documentation Marks

**Catalogs/views:** `pg_stat_activity`, `pg_stat_database`, `pg_stat_io`, `pg_stat_bgwriter`, `pg_stat_wal`, `pg_stat_archiver`, `pg_stat_replication`, `pg_roles`, `pg_auth_members`, `pg_hba_file_rules`, `pg_file_settings`, `pg_settings`.  
**Configuration:** `logging_collector`, `log_min_duration_statement`, `log_lock_waits`, `log_checkpoints`, `log_autovacuum_min_duration`, `log_connections`, `log_disconnections`, `log_line_prefix`, `ssl`, `password_encryption`, `shared_preload_libraries`, `track_io_timing`, `compute_query_id`.

---

# Capstone — Operate a Production-Like PostgreSQL Platform

```text
Application load generator → PgBouncer → Primary PostgreSQL
                                         ├─ synchronous standby
                                         ├─ asynchronous read replica
                                         ├─ WAL archive / backup repository
                                         └─ metrics, logs, traces, query analytics
```

Demonstrate:

1. Zero-downtime expand/backfill/contract migration.
2. Connection-storm containment with PgBouncer and role limits.
3. Query-plan diagnosis and concurrent index rollout.
4. Autovacuum tuning for a high-churn relation.
5. Lock queue recovery.
6. Replica-lag diagnosis and controlled failover.
7. Point-in-time recovery after accidental deletion.
8. Automated partition lifecycle and retention.
9. Written SLOs, RPO/RTO, alert map, and incident runbooks.

## Completion Standard

The engineer is complete when they can explain not just **what command to run**, but how it interacts with WAL, MVCC visibility, locks, replication, storage, and recovery guarantees.
