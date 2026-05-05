# Runbook: Error Budget Burn Rate

**Severity**: SEV-1 (fast-burn) / SEV-2 (slow-burn)
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6) | **Alertmanager routing**: `severity=page` → PagerDuty

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| `HighErrorBudgetBurnRate` alert fires (fast-burn: >14.4× on 1h + 5m windows) | PagerDuty page + Slack `#platform-alerts` |
| `SustainedErrorBudgetBurnRate` alert fires (slow-burn: >6× on 6h + 30m windows) | PagerDuty page |
| 4-window burn-rate panel in Grafana `platform-slo.json` shows red | Grafana → Platform SLO dashboard |
| `availability:slo:rate1h{slo_target="platform"}` dipping below 0.999 | Prometheus query |
| Error-rate spike or latency regression in service dashboards | Grafana → per-service dashboard |

**Alert source**: `infra/observability/prometheus/rules/alerting-rules.yaml` lines 39–68 (PE.05).

---

## Triage

1. **Open the Grafana Platform SLO dashboard** (`platform-slo.json`). Identify which of the 4 windows is burning:
   - 1h + 5m windows elevated → fast-burn → **treat as SEV-1; page backup if primary not responding in 5 min**.
   - 6h + 30m windows elevated, 1h/5m within budget → slow-burn → **treat as SEV-2; resolve within 1h triage**.

2. **Identify the burn driver**: open the per-service error-rate panels. Which service has elevated 5xx or timeout rate?
   ```promql
   sum by (service) (rate(http_request_errors_total{slo_target="platform"}[5m]))
   / sum by (service) (rate(http_requests_total{slo_target="platform"}[5m]))
   ```

3. **Check recent deploys**: `helm history <release>` for the affected service — was a deploy made in the last 30 min?

4. **Check for latency regression** (burn may be latency-driven, not error-rate-driven):
   ```promql
   latency_p95:slo:rate5m
   ```
   If p95 > 0.2s → follow `high-latency.md`.

5. **Verify it is NOT KraftData-isolated burn**: check `slo_target="kraftdata-dependent"` label.
   If alert has `slo_target="kraftdata-dependent"` → **SLA-EXEMPT** — follow `kraftdata-outage.md` instead.

---

## Resolution

### Branch A — Error-rate burn (HTTP 5xx elevated on a service)

1. Check service pod logs: `kubectl logs -n eusolicit deploy/<service> --since=10m | tail -100`
2. Check if a recent deploy is the cause → if yes, **rollback immediately** → follow `deploy-rollback.md`.
3. If no recent deploy: check DB / Redis connectivity from the service pod:
   ```bash
   kubectl exec -n eusolicit deploy/<service> -- python -c "from app.db import engine; engine.execute('SELECT 1')"
   ```
4. If DB connectivity issue → follow `pg-failover.md` or `full-disk-on-pg.md` based on RDS console.
5. If Redis connectivity issue → follow `redis-failover.md` or `redis-evictions.md`.

### Branch B — Latency burn (p95 regression, low error rate)

→ Follow `high-latency.md`.

### Branch C — Infrastructure event (node drain, ingress restart)

→ Follow `node-drain.md` or `ingress-controller-restart.md` based on the triggering event.

---

## Verification

After applying fix:

1. **Prometheus query** — burn rate should return to ≤1× (normal consumption):
   ```promql
   (1 - availability:slo:rate1h{slo_target="platform"}) / (1 - 0.999)
   ```
   Target: value < 14.4 (fast-burn gate), ideally < 1.

2. **Grafana**: all 4-window panels on `platform-slo.json` return to green.

3. **Error budget remaining**: confirm > 0% remaining for the calendar month.
   ```promql
   (1 - (sum(rate(http_request_errors_total{slo_target="platform"}[30d])) / sum(rate(http_requests_total{slo_target="platform"}[30d])))) / 0.001
   ```

4. **PagerDuty**: resolve the incident. Await auto-resolve if the PromQL expression drops below threshold for the `for:` period.

---

## Rollback

If §Resolution steps worsen the situation:

1. If a deploy was rolled back and rollback itself caused issues → follow `deploy-rollback.md` §Rollback section.
2. If DB failover was triggered → monitor per-service reconnect SLA ≤30s per `pg-failover.md` §Verification.
3. Revert any `maxmemory-policy` changes if Redis was modified → `redis-evictions.md` §Rollback.

---

## Related

- `high-latency.md` — p95 latency violation (NFR-2)
- `pg-failover.md` — PostgreSQL Multi-AZ failover
- `redis-failover.md` — Redis ElastiCache failover
- `deploy-rollback.md` — Helm release rollback
- `kraftdata-outage.md` — KraftData SLA-EXEMPT burn
- `severity-definitions.md` — SEV-1/2 response SLAs
- PE.05 `alerting-rules.yaml` lines 39–68 — alert definitions
- Grafana `platform-slo.json` — 4-window burn-rate panel
