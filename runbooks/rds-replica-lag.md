# Runbook: RDS Multi-AZ Replica Lag

**Severity**: SEV-1 (if sustained >5m and data-integrity risk) / SEV-2 (if isolated spike)
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6) | **Alertmanager routing**: `severity=page` → PagerDuty

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| `HighRDSReplicaLag` alert fires (`aws_rds_replica_lag_average` > 30s for 5m) | PagerDuty page |
| AWS RDS Console → Monitoring → Replica Lag metric spikes | AWS Console |
| Read traffic overflowing to primary (connection-pool exhaustion on replica) | Per-service logs |
| NFR-15 data-integrity risk: replica lag means failover target is behind | PE.02 §Failover Drill Steps |
| Write-latency increase on primary if replica is slow to acknowledge | `high-latency.md` alert co-firing |

**Alert source**: `infra/observability/prometheus/rules/alerting-rules.yaml` lines 83–96 (PE.05).
**NFR reference**: NFR-15 — data integrity; replica lag ≤30s is the PE.02 invariant.

---

## Triage

1. **Check replica lag history in AWS Console**:
   - RDS → Databases → `eusolicit-prod` → Monitoring → `ReplicaLag` metric.
   - Is lag growing (trending up) or a transient spike (already declining)?
   - Trending up → escalate to SEV-1 (failover risk); transient → SEV-2.

2. **Check primary write load**:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/RDS \
     --metric-name WriteLatency \
     --dimensions Name=DBInstanceIdentifier,Value=eusolicit-prod \
     --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Average
   ```
   High write latency on primary = replica falls behind naturally (not a bug; capacity issue).

3. **Check for a bulk-write operation** (migration, data import):
   ```bash
   kubectl logs -n eusolicit job/<service>-migrate --tail=50
   ```
   If a migration is running → lag is expected temporarily; monitor for completion.

4. **Check CloudWatch `aws_rds_database_connections_average`**:
   - Is the replica saturated (many read connections)?
   - Services should be directing reads to primary fallback if replica is lagging.

5. **Is an automated failover already triggered?** Check RDS Events:
   ```bash
   aws rds describe-events --source-identifier eusolicit-prod --duration 60
   ```

---

## Resolution

### Branch A — Transient spike (lag declining, no failover imminent)

1. Monitor for 5–10 minutes. If lag returns to <5s naturally → close as SEV-3.
2. Check if a bulk write (migration, large import) caused the spike. If yes → expected behaviour; log the event.
3. Update PagerDuty incident with findings and resolve.

### Branch B — Sustained lag >30s (failover decision point)

1. **Prefer waiting**: AWS Multi-AZ replica lag usually self-corrects within 2–5 minutes after write spike subsides.
2. **Escalate to SEV-1** if lag has been >30s for >5 minutes AND write traffic is normal (no bulk operation).
3. **Force failover** (manual) if primary shows signs of degradation:
   ```bash
   aws rds reboot-db-instance \
     --db-instance-identifier eusolicit-prod \
     --force-failover
   ```
   ⚠️ This promotes the replica to primary. **Follow `pg-failover.md` immediately after triggering.**

4. **Verify per-service reconnect SLA** after failover: all services should reconnect within ≤30s per Story 21-2 §Failover Drill Steps.

### Branch C — Automated failover already triggered by AWS

→ Follow `pg-failover.md` for the full post-failover verification procedure.

---

## Verification

1. **Replica lag returns to <5s**:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/RDS \
     --metric-name ReplicaLag \
     --dimensions Name=DBInstanceIdentifier,Value=eusolicit-prod \
     --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Average
   ```
   Target: < 5s.

2. **All services reconnect** (if failover was triggered): check service health endpoints:
   ```bash
   for svc in client-api admin-api agenticsai-gateway data-pipeline notification integrations-api; do
     kubectl exec -n eusolicit deploy/$svc -- curl -s http://localhost:${PORT}/health | grep '"status":"ok"'
   done
   ```

3. **No transaction loss**: compare write counts before/after failover via application metrics.

4. **`HighRDSReplicaLag` alert resolves** in PagerDuty after lag drops below 30s for the `for: 5m` period.

---

## Rollback

If forced failover causes unexpected issues:

- There is no "undo" for `reboot-db-instance --force-failover` — the former replica is now primary.
- Monitor `pg-failover.md` §Verification checklist to confirm the new primary is healthy.
- If the new primary shows issues, contact AWS Support — do NOT re-trigger failover without guidance.

---

## Related

- `pg-failover.md` — full PostgreSQL failover procedure (post-trigger steps)
- `high-latency.md` — latency regression (may co-fire if reads overflow to primary)
- `error-budget-burn.md` — burn-rate alert (may co-fire during failover)
- `full-disk-on-pg.md` — disk pressure may cause write-lag spikes
- PE.05 `alerting-rules.yaml` lines 83–96 — alert definition
- Story 21-2 `pe-02-cutover-runbook.md` §Failover Drill Steps — reconnect SLA evidence
