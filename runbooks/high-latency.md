# Runbook: High Latency (p95 NFR-2 Violation)

**Severity**: SEV-2
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6) | **Alertmanager routing**: `severity=ticket` → Slack `#platform-alerts`

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| `HighLatencyP95` alert fires (p95 > 0.2s for 10m) | Slack `#platform-alerts` |
| Grafana per-service endpoint-breakdown panel shows slow endpoints | Grafana → `<service>-dashboard.json` → Endpoint Breakdown |
| `latency_p95:slo:rate5m` exceeds 0.2 | Prometheus |
| Customer reports of slow page loads / timeouts | Support channel |
| Error-budget slow-burn alert co-firing | `error-budget-burn.md` |

**NFR reference**: NFR-2 — REST p95 < 200ms.
**Baseline reference**: Story 21-1 `load-test-results.md` — baseline p95 at 30 VUs and 50 VUs.
**Alert source**: `infra/observability/prometheus/rules/alerting-rules.yaml` lines 73–82 (PE.05).

---

## Triage

1. **Identify the slow service**: open Grafana → per-service dashboard → Endpoint Breakdown panel.
   - Sort by p95 descending. Which endpoint is the outlier?
   ```promql
   histogram_quantile(0.95, sum by (service, path, le) (rate(http_request_duration_seconds_bucket[5m])))
   ```

2. **Compare against baseline** from `load-test-results.md`:
   - client-api `POST /proposals` baseline p95 = ~90ms at 30 VUs.
   - If now >200ms with same VU count → regression, not load-driven.

3. **Check for slow queries** using `pg_stat_statements_seconds_total`:
   ```sql
   -- Run from psql / admin panel against the relevant schema
   SELECT query, mean_exec_time, calls, total_exec_time
   FROM pg_stat_statements
   ORDER BY mean_exec_time DESC
   LIMIT 20;
   ```
   A new full-table scan (missing index, plan change after migration) is the most common cause.

4. **Check recent migrations**: was an Alembic migration run in the last 30 min for this service?
   - `kubectl logs -n eusolicit job/<service>-migrate --tail=50`
   - If yes and the migration added/removed an index → evaluate plan regression.

5. **Check agenticsai-gateway TTFB** for AgenticSAI-dependent endpoints:
   - `kubectl logs -n eusolicit deploy/agenticsai-gateway --since=10m | grep 'agenticsai'`
   - If agenticsai-gateway TTFB elevated → **SLA-EXEMPT** per architecture.md line 762 → follow `agenticsai-outage.md`.

6. **Check HPA state**: is the service at max replicas?
   ```bash
   kubectl get hpa -n eusolicit
   ```

---

## Resolution

### Branch A — Query-plan regression (slow query identified in pg_stat_statements)

1. Confirm the slow query with `EXPLAIN ANALYZE`:
   ```sql
   EXPLAIN ANALYZE <slow query from pg_stat_statements>;
   ```
2. If a sequential scan where an index is expected → check if a recent migration dropped or changed the index.
3. **If migration is the cause → rollback** → follow `deploy-rollback.md`:
   - `helm rollback <release> <previous-revision>`
   - Then `alembic downgrade <previous_rev>` if DDL change must be reverted.
4. If no migration → add the missing index (forward-fix preferred for additive changes):
   ```sql
   CREATE INDEX CONCURRENTLY idx_<table>_<column> ON <schema>.<table>(<column>);
   ```
   `CONCURRENTLY` avoids table lock in production.

### Branch B — Runtime degradation (no slow queries; load increased or HPA at ceiling)

1. Scale the service manually if HPA is at max:
   ```bash
   kubectl scale deploy/<service> -n eusolicit --replicas=<N+2>
   ```
2. After load subsides, let HPA scale back. Investigate root cause (load spike from a customer? bot traffic?).
3. If sustained load growth → capacity planning ticket (sprint follow-up).

### Branch C — agenticsai-gateway / AgenticSAI TTFB (SLA-EXEMPT path)

→ Follow `agenticsai-outage.md`. This is SLA-EXEMPT per architecture.md line 762.

---

## Verification

1. **Prometheus query** — p95 returns below 0.2s:
   ```promql
   latency_p95:slo:rate5m
   ```
   Target: < 0.2 across all services.

2. **Grafana** — Endpoint Breakdown panel shows all p95 values below 200ms.

3. **pg_stat_statements** — no query with `mean_exec_time > 100ms` for the affected service's common paths.

4. **Story 21-1 baseline sanity** — rerun a quick k6 smoke test at 30 VUs and confirm p95 ≤ 90ms for `client-api`.

5. **Alert cleared** — `HighLatencyP95` resolves after the `for: 10m` period passes below threshold.

---

## Rollback

If §Resolution steps worsen latency (e.g., index creation causes more I/O under load):

1. Cancel the `CREATE INDEX CONCURRENTLY` if still running:
   ```sql
   SELECT pid, query FROM pg_stat_activity WHERE query LIKE '%CREATE INDEX%';
   SELECT pg_cancel_backend(<pid>);
   ```
2. Revert via `deploy-rollback.md` if a migration-based rollback is needed.
3. Scale down temporarily if scaling increased load on DB (check `aws_rds_database_connections_average`).

---

## Related

- `error-budget-burn.md` — burn-rate alert (may co-fire with this)
- `deploy-rollback.md` — Helm rollback + Alembic downgrade
- `agenticsai-outage.md` — agenticsai-gateway TTFB (SLA-EXEMPT)
- `pg-failover.md` — if DB health is the root cause
- `rds-replica-lag.md` — if replica lag is causing read traffic to overflow to primary
- Story 21-1 `load-test-results.md` — baseline p95 numbers for sanity comparison
- PE.05 `alerting-rules.yaml` lines 73–82 — alert definition
