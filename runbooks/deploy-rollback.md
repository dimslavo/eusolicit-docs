# Runbook: Deploy Rollback

**Severity**: SEV-1 (if production is broken) / SEV-2 (if canary/partial)
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6)

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| Post-deploy error-rate spike (`HighErrorBudgetBurnRate` alert fires) | PagerDuty + `error-budget-burn.md` |
| Post-deploy p95 latency regression (`HighLatencyP95` alert fires) | Slack `#platform-alerts` + `high-latency.md` |
| Canary deployment health-check failures | Helm rollout status |
| Service CrashLoopBackOff after deploy | `kubectl get pods -n eusolicit` |
| Customer reports of new regressions correlating to deploy time | Support channel |
| `helm history <release>` shows recent revision | Helm CLI |

---

## Triage

1. **Confirm the deploy timing correlates with the incident**:
   ```bash
   # Check deploy time from Helm history
   for svc in client-api admin-api agenticsai-gateway data-pipeline notification integrations-api; do
     echo "=== $svc ===" && helm history $svc -n eusolicit --max 3
   done
   ```
   Compare the most recent revision timestamp with the incident start time.

2. **Identify the failing service**:
   ```bash
   kubectl get pods -n eusolicit
   kubectl get events -n eusolicit --sort-by='.lastTimestamp' | tail -20
   ```

3. **Check if a database migration was applied**:
   ```bash
   kubectl logs -n eusolicit job/<service>-migrate --tail=50
   ```
   ⚠️ **Critical**: if a migration ran, `helm rollback` alone is NOT sufficient (DDL changes are NOT auto-reverted by Helm).
   - **Additive migrations** (new columns, new tables with defaults): forward-fix is preferred; rollback is usually unnecessary.
   - **Breaking migrations** (column rename, column type change, column drop): rollback requires `alembic downgrade`.

4. **Assess rollback vs. forward-fix**:
   - **Rollback preferred when**: migration is breaking; forward-fix would take >30 min; SEV-1 active.
   - **Forward-fix preferred when**: migration is additive; regression is in application code only; fix can be deployed in <15 min.

---

## Resolution

### Branch A — Application code regression only (no migration applied)

1. **Roll back the Helm release**:
   ```bash
   # Identify the previous revision
   helm history <service> -n eusolicit --max 5
   # Roll back to previous revision
   helm rollback <service> <previous-revision-number> -n eusolicit --wait --timeout=5m
   ```

2. **Verify rollback completed**:
   ```bash
   helm status <service> -n eusolicit
   kubectl get pods -n eusolicit | grep <service>
   ```
   Expected: `STATUS = deployed` (not `failed`); pods all `Running`.

3. **Verify error rate returns to baseline**:
   ```promql
   sum(rate(http_request_errors_total{service="<service>"}[5m]))
   ```

### Branch B — Migration-breaking-change + application code regression (rollback required)

⚠️ **Order matters**: rollback application code FIRST, then rollback migration. Never the reverse.

1. **Step 1: Roll back application code**:
   ```bash
   helm rollback <service> <previous-revision-number> -n eusolicit --wait --timeout=5m
   ```

2. **Step 2: Roll back Alembic migration** (Story 1.4 downgrade pattern):
   ```bash
   # Identify the migration revision to roll back TO
   kubectl exec -n eusolicit deploy/<service> -- alembic history --verbose | head -20
   kubectl exec -n eusolicit deploy/<service> -- alembic current
   # Downgrade to the previous revision
   kubectl exec -n eusolicit deploy/<service> -- alembic downgrade <previous_revision>
   ```
   ⚠️ **Only run downgrade if the migration is reversible** (has a `downgrade()` function in the migration file).
   ⚠️ **Data loss risk**: migrations that drop columns or tables lose data on downgrade. Confirm with CTO before proceeding.

3. **Verify migration state**:
   ```bash
   kubectl exec -n eusolicit deploy/<service> -- alembic current
   ```
   Expected: previous revision ID.

### Branch C — Forward-fix (additive migration; code fix available quickly)

1. Fix the bug in the code.
2. Deploy the fix as a new Helm revision (do NOT roll back; the additive migration is harmless).
3. Monitor error rate after deploy.

### Branch D — Canary failure (canary deployed to a subset of traffic)

1. **Abort the canary** and route all traffic back to the stable version:
   ```bash
   helm rollback <service> <stable-revision-number> -n eusolicit --wait
   ```
2. The canary's Helm revision is abandoned. The stable revision takes over.

---

## Verification

1. **Error rate returns to Story 21-1 baseline** (p95 and error-rate panel in Grafana):
   ```promql
   latency_p95:slo:rate5m
   sum(rate(http_request_errors_total{slo_target="platform"}[5m]))
   ```

2. **Helm release status is `deployed`** (not `failed` or `pending-rollback`):
   ```bash
   helm list -n eusolicit
   ```

3. **Alembic revision at expected state** (if migration was downgraded):
   ```bash
   kubectl exec -n eusolicit deploy/<service> -- alembic current
   ```

4. **Database schema integrity** — spot-check affected tables via psql to ensure schema matches expected state.

5. **All service health endpoints return 200**:
   ```bash
   for svc in client-api admin-api agenticsai-gateway data-pipeline notification integrations-api; do
     kubectl exec -n eusolicit deploy/$svc -- curl -sf http://localhost:8000/health && echo "$svc: OK"
   done
   ```

6. **`HighErrorBudgetBurnRate` alert resolves** after the `for:` period passes below threshold.

---

## Rollback of the Rollback (if rollback itself causes issues)

If `helm rollback` fails or creates a new problem:

1. Check rollback logs:
   ```bash
   helm status <service> -n eusolicit
   kubectl get events -n eusolicit | grep <service>
   ```

2. Try rolling back to an earlier revision (not just N-1):
   ```bash
   helm history <service> -n eusolicit --max 10
   helm rollback <service> <earlier-stable-revision> -n eusolicit --wait
   ```

3. If Helm state is corrupted → re-install the service with a known-good chart version:
   ```bash
   helm uninstall <service> -n eusolicit
   helm install <service> ./<service-chart> -n eusolicit --set image.tag=<stable-tag>
   ```
   ⚠️ Uninstall destroys the Helm release history. Use only as last resort.

---

## Related

- `error-budget-burn.md` — burn-rate alert that triggers rollback decision
- `high-latency.md` — latency regression (may require rollback)
- `pg-failover.md` — if a migration caused DB connectivity issues
- `rds-replica-lag.md` — if a migration with large backfill caused replica lag
- Story 1.4 — Alembic migration scaffold + `downgrade()` pattern
- architecture.md ADR-002 — RBAC: ensure rollback doesn't break auth (check auth service health post-rollback)
