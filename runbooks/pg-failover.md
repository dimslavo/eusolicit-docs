# Runbook: PostgreSQL Multi-AZ Failover

**Severity**: SEV-1 (if primary unreachable) / SEV-2 (if planned/manual failover)
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6) — lifts operator-facing content from PE.02 `pe-02-cutover-runbook.md` §Failover Drill Steps

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| RDS Multi-AZ automated failover event in AWS Console | AWS RDS → Events |
| `HighRDSReplicaLag` alert co-firing before failover | `rds-replica-lag.md` |
| All services reporting DB connection errors simultaneously | Per-service logs + Grafana |
| `aws_rds_database_connections_average` drops to 0 | CloudWatch |
| Health endpoints returning unhealthy during reconnect window | `kubectl get pods -n eusolicit` |
| Alembic migration job failing with connection error | Job logs |

---

## Triage

1. **Confirm failover status in AWS Console**:
   ```bash
   aws rds describe-events \
     --source-identifier eusolicit-prod \
     --duration 60 \
     --event-categories failover
   ```
   Expected event: `"Finished Multi-AZ DB instance failover to eusolicit-prod"`.

2. **Check which instance is now primary**:
   ```bash
   aws rds describe-db-instances \
     --db-instance-identifier eusolicit-prod \
     --query 'DBInstances[0].{Endpoint:Endpoint.Address,MultiAZ:MultiAZ,Status:DBInstanceStatus}'
   ```

3. **Check per-service reconnect status**:
   ```bash
   for svc in client-api admin-api ai-gateway data-pipeline notification integrations-api; do
     echo "=== $svc ===" && kubectl logs -n eusolicit deploy/$svc --since=5m | grep -E "(connection|reconnect|error)" | tail -5
   done
   ```
   Expected: connection errors followed by successful reconnect within ≤30s.

4. **Was this automated (AWS-triggered) or manual?** Check RDS events for the trigger:
   - `"Multi-AZ instance failover started"` = automated (primary failure).
   - `"DB instance restarted"` with `--force-failover` = manual (operator-triggered).

---

## Resolution

### Branch A — Automated failover (AWS triggered; primary failed)

AWS Multi-AZ handles promotion automatically. The platform engineer's job is verification, not execution.

1. **Wait for AWS to complete promotion** (typically 30–120 seconds).
2. **Verify per-service reconnect** — all 6 services MUST reconnect within ≤30s per PE.02 §Failover Drill Steps:
   ```bash
   kubectl get events -n eusolicit --sort-by='.lastTimestamp' | tail -20
   ```
   Look for pod restarts (normal; connection pool re-establishment) followed by `"Ready"` state.
3. **Run per-service health checks**:
   ```bash
   for svc in client-api admin-api ai-gateway data-pipeline notification integrations-api; do
     kubectl exec -n eusolicit deploy/$svc -- curl -sf http://localhost:8000/health && echo "$svc: OK" || echo "$svc: FAIL"
   done
   ```
4. **Verify Alembic migration connectivity** (if a migration was running):
   ```bash
   kubectl logs -n eusolicit job/<service>-migrate --tail=20
   ```
   If the migration failed mid-run → check whether it needs to be re-run (idempotent migrations are safe to re-run).

### Branch B — Manual failover (operator-initiated; primary degraded but not failed)

1. **Trigger failover**:
   ```bash
   aws rds reboot-db-instance \
     --db-instance-identifier eusolicit-prod \
     --force-failover
   ```
   ⚠️ This promotes the replica. All active transactions on the primary are rolled back.

2. **Start the 30-second reconnect timer**. Services use SQLAlchemy connection pools with reconnect logic. Monitor pod logs.

3. **Proceed with Branch A verification steps** above.

### Branch C — Connection pool exhaustion without failover

If services report DB errors but no failover event exists:

1. Check connection count: `aws_rds_database_connections_average`.
2. Check for connection leaks in service logs (unclosed sessions, missing `finally` blocks).
3. Emergency: restart the service with high connection count:
   ```bash
   kubectl rollout restart deploy/<service> -n eusolicit
   ```

---

## Verification

1. **All services reconnect within ≤30s** (PE.02 §Failover Drill Steps invariant):
   - Measure: timestamp first connection error → timestamp first successful health check.
   - Acceptable: ≤30s total reconnect window per service.

2. **Zero transaction loss**: check application-level metrics (e.g., no orphaned `proposals` in `DRAFT` state that should be `SUBMITTED`).

3. **DB write path healthy**:
   ```bash
   kubectl exec -n eusolicit deploy/client-api -- python -c \
     "from client_api.db import engine; import asyncio; asyncio.run(engine.execute('SELECT 1'))"
   ```

4. **Replica re-established**: AWS will automatically provision a new standby replica. Monitor in RDS Console → Configuration → Multi-AZ.
   - New standby typically ready within 10–30 minutes.

5. **`HighRDSReplicaLag` clears** after new standby catches up.

---

## Rollback

There is no "undo" for a Multi-AZ failover — the former replica is now primary. If the new primary shows unexpected issues:

1. **Do not force another failover immediately** — AWS needs time to provision the new standby.
2. Contact AWS Support if the new primary is unstable.
3. If a deploy was in-flight during failover → the deploy must be re-assessed:
   - Check if Alembic migration completed on the new primary.
   - If uncertain → use `alembic current` to check applied revisions.
   - Follow `deploy-rollback.md` if the application code rollback is needed.

---

## Related

- `rds-replica-lag.md` — precursor alert (lag >30s → failover decision)
- `full-disk-on-pg.md` — disk pressure may trigger or follow a failover
- `deploy-rollback.md` — Helm + Alembic rollback if deploy was in-flight
- `error-budget-burn.md` — burn-rate impact during reconnect window
- Story 21-2 `pe-02-cutover-runbook.md` §Failover Drill Steps — reconnect SLA evidence (dev-side audit trail)
- NFR-15 — data integrity: replica lag ≤30s, RPO ≤24h
