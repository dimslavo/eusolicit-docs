# Runbook: PostgreSQL RDS Disk Full

**Severity**: SEV-1 (if write-failures cascading) / SEV-2 (if early warning, writes still succeeding)
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6)

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| `aws_rds_freeable_memory_average` critically low (< 500MB) | CloudWatch |
| RDS storage-full alarm in CloudWatch | CloudWatch → RDS alarms |
| Write-failure cascade — services reporting `DiskFull` or `no space left on device` | Per-service logs |
| `HighErrorBudgetBurnRate` co-firing (all write operations failing) | `error-budget-burn.md` |
| PostgreSQL `pg_stat_user_tables` showing unexpectedly large tables | psql query |
| Dead rows bloat (unvacuumed tables growing unboundedly) | psql `pgstattuple` |
| RDS `FreeStorageSpace` CloudWatch metric approaching 0 | CloudWatch |

**Alert reference**: PE.05 `alerting-rules.yaml` references `aws_rds_freeable_memory_average` for memory; watch also `FreeStorageSpace` CloudWatch metric.

---

## Triage

1. **Check RDS free storage space**:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/RDS \
     --metric-name FreeStorageSpace \
     --dimensions Name=DBInstanceIdentifier,Value=eusolicit-prod \
     --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Minimum \
     --unit Bytes
   ```
   Bytes < 1073741824 (1GB) → critical; < 5368709120 (5GB) → warning.

2. **Check storage auto-scaling status** (verify it is enabled per Story 21-2):
   ```bash
   aws rds describe-db-instances \
     --db-instance-identifier eusolicit-prod \
     --query 'DBInstances[0].{AllocatedStorage:AllocatedStorage,MaxAllocatedStorage:MaxAllocatedStorage}'
   ```
   If `MaxAllocatedStorage > AllocatedStorage` → auto-scaling will trigger automatically when free space < 10%. Check if it already triggered.

3. **Identify the largest tables** using psql:
   ```sql
   -- Connect as admin (migration_role has DDL rights)
   SELECT schemaname, tablename,
          pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
          pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
          pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size
   FROM pg_tables
   WHERE schemaname IN ('client', 'admin', 'pipeline', 'gateway', 'notification', 'shared')
   ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
   LIMIT 20;
   ```

4. **Check dead-row bloat** (primary disk-pressure cause after high-churn tables):
   ```sql
   SELECT schemaname, relname, n_dead_tup, n_live_tup, last_autovacuum, last_autoanalyze
   FROM pg_stat_user_tables
   ORDER BY n_dead_tup DESC
   LIMIT 10;
   ```
   High `n_dead_tup` with old `last_autovacuum` → autovacuum backlog → primary disk-pressure cause.

5. **Check Story 1.3 schema-isolation invariant** — no cross-schema CASCADEs can amplify disk pressure:
   - Each service schema is isolated per `infra/postgres/init/01-init-schemas-and-roles.sql`.
   - If one schema's disk pressure is cascading to others → this is a schema isolation violation (escalate immediately).

---

## Resolution

### Branch A — Auto-scaling not triggered; disk <5GB free (warning state)

1. **Verify auto-scaling is enabled**; if not, enable it:
   ```bash
   aws rds modify-db-instance \
     --db-instance-identifier eusolicit-prod \
     --max-allocated-storage 500 \
     --apply-immediately
   ```
   RDS storage auto-scales from current `AllocatedStorage` up to `max-allocated-storage` in 10GB increments.
   Auto-scale triggers when free space < 10% of allocated AND < 10GB.

2. **Trigger manual autovacuum** on the bloated table(s):
   ```sql
   VACUUM VERBOSE ANALYZE <schema>.<tablename>;
   ```
   Non-blocking; reclaims dead-row space without locking the table.

3. **Check WAL archive volume** if using RDS with PITR:
   - Transaction log archives can consume significant storage independently.
   - Verify via RDS Enhanced Monitoring → Storage breakdown.

### Branch B — Disk critically full (<1GB free); writes failing

1. **Escalate to SEV-1** immediately. Notify on-call backup.

2. **Emergency VACUUM FULL** decision tree:
   - `VACUUM FULL` reclaims dead space **and** defragments pages → requires **brief table lock** (table is read-only during VACUUM FULL).
   - Run VACUUM FULL only during low-traffic window (or immediately if writes are already blocked):
   ```sql
   -- Run as superuser / migration_role
   VACUUM FULL VERBOSE <schema>.<tablename>;
   ```
   ⚠️ `VACUUM FULL` on large tables can take minutes. Schedule during off-peak hours if writes are still possible.

3. **Read-replica-promotion as emergency write-traffic redirection**:
   - If primary disk is completely full and writes are blocked → promote the read replica to accept writes temporarily.
   - Follow `pg-failover.md` Branch B (force failover).
   - Caution: promotes replica AS new primary; original primary (disk-full) must be cleaned up separately.

4. **Delete identifiable large/safe data** (only with CTO approval for non-ephemeral data):
   - Log files, audit trails older than retention policy → `DELETE FROM <schema>.<log_table> WHERE created_at < NOW() - INTERVAL '90 days'`.
   - Run VACUUM immediately after DELETE to reclaim space.

5. **Emergency manual storage increase** (bypasses auto-scaling trigger):
   ```bash
   aws rds modify-db-instance \
     --db-instance-identifier eusolicit-prod \
     --allocated-storage <current_GB + 50> \
     --apply-immediately
   ```
   Note: storage increase is irreversible in RDS (cannot decrease). Wait 6 hours between increases.

---

## Verification

1. **Free storage space recovering**:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/RDS --metric-name FreeStorageSpace \
     --dimensions Name=DBInstanceIdentifier,Value=eusolicit-prod \
     --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Minimum --unit Bytes
   ```
   Target: > 5GB free.

2. **Write operations succeeding** — check application-level write metrics (new proposals, billing events).

3. **Dead-row bloat cleared** (`n_dead_tup` reduced for top tables after VACUUM).

4. **Auto-scaling confirmed active** — `MaxAllocatedStorage > AllocatedStorage`.

5. **Schema isolation intact** — no cross-schema data loss or corruption.

---

## Rollback

1. VACUUM / VACUUM FULL cannot be "rolled back" — they are maintenance operations. If VACUUM FULL caused a prolonged lock:
   - Use `pg_cancel_backend(<pid>)` to cancel if the lock is blocking writes.
   - Re-assess the table size and consider a partial VACUUM FULL (by partition/schema).

2. If emergency storage increase was applied → no rollback possible (RDS storage is unidirectional).

3. If read-replica promotion was triggered → follow `pg-failover.md` §Rollback.

---

## Related

- `pg-failover.md` — if disk-full triggers or requires a failover
- `rds-replica-lag.md` — replica lag during heavy VACUUM operations
- `deploy-rollback.md` — if a migration (e.g., backfill) caused the disk pressure
- `error-budget-burn.md` — burn-rate co-fires during write-failure cascade
- Story 1.3 schema-isolation invariant — `infra/postgres/init/01-init-schemas-and-roles.sql`
- PE.05 `alerting-rules.yaml` — `aws_rds_freeable_memory_average` alert
