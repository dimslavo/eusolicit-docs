# Runbook: KraftData Ingest Outage

**Severity**: SEV-2 (platform-visible; SLA-EXEMPT from platform SLO)
**SLA-Scope**: EXEMPT (vendor outage)
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6) — consolidates Epic 4 partial KraftData runbook content

---

> **SLA-EXEMPT notice**: KraftData is an upstream vendor. KraftData outages are
> **explicitly excluded from the platform 99.9% SLA scope** per architecture.md
> line 762 and the Epic 4 KraftData isolation precedent. The platform circuit
> breaker (`AgentCircuit`) protects downstream services; AI-Gateway outages from
> KraftData incidents do NOT count against the platform error budget.
> See `severity-definitions.md` SLA-scope table.

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| `KraftDataDependentHighBurnRate` alert fires (5m + 1h windows >14.4×) | PagerDuty page (separate from platform SLO page) |
| AI-Gateway 5xx rate elevated on KraftData-dependent endpoints | Grafana → ai-gateway dashboard |
| `slo_target="kraftdata-dependent"` recording rule promoted | `kraftdata-isolation-rules.yaml` |
| `AgentCircuit.state = OPEN` for one or more agents | AI-Gateway logs |
| KraftData procurement data ingest stopped (no new opportunities in last 30 min) | Admin API → data-pipeline logs |
| Opportunity-matching Celery workers timing out | data-pipeline service logs |

**Alert source**: `infra/observability/prometheus/rules/kraftdata-isolation-rules.yaml` (PE.05).
**Epic 4 circuit-breaker code**: `services/ai-gateway/src/ai_gateway/services/circuit_breaker.py` (`AgentCircuit` class).

---

## Triage

1. **Check KraftData status page**: https://status.kraftdata.eu (or operator-known status URL).
   - Is there a known incident? → Confirm EXEMPT status; no platform action needed until KraftData recovers.

2. **Inspect circuit-breaker state**:
   ```bash
   # AI-Gateway exposes circuit-breaker state in logs
   kubectl logs -n eusolicit deploy/ai-gateway --since=10m | grep -i "circuit"
   ```
   Look for: `"state": "open"` or `"state": "half_open"` in structured log lines.

3. **Check circuit-breaker state via AgentCircuit**:
   - `AgentCircuit.state` enum: `CLOSED` (normal) → `OPEN` (failing; rejecting calls) → `HALF_OPEN` (testing recovery).
   - State machine per `circuit_breaker.py`: CLOSED → OPEN after `threshold` consecutive failures;
     OPEN → HALF_OPEN after `cooldown` seconds (`settings.circuit_breaker_cooldown`);
     HALF_OPEN → CLOSED on next successful call.

4. **Confirm platform services are NOT impacted**:
   - Non-KraftData endpoints (auth, billing, ESPD forms) should be healthy.
   - Check `slo_target="platform"` recording rules — these must NOT be burning.
   ```promql
   sum(rate(http_request_errors_total{slo_target="platform"}[5m]))
   ```
   Target: near 0. If elevated → separate SEV-1; follow `error-budget-burn.md`.

5. **Estimate KraftData outage duration** from KraftData status page / incident history.

---

## Resolution

**Primary action: WAIT.** KraftData is an upstream vendor; the platform cannot resolve their outage.

1. **Acknowledge the PagerDuty alert** with `acknowledged` status and add note:
   > "KraftData upstream outage confirmed at <time>. Circuit breaker active. Platform SLO unaffected per architecture.md line 762. Monitoring KraftData status page for recovery. ETA: <KraftData estimate or 'unknown'>."

2. **Monitor circuit-breaker recovery**: once KraftData recovers, the AI-Gateway circuit will automatically transition:
   - `OPEN` → `HALF_OPEN` after `circuit_breaker_cooldown` seconds (default 30s per `kraftdata_resilient.py` line 73).
   - `HALF_OPEN` → `CLOSED` on next successful call.
   - No manual intervention required unless cooldown is misconfigured.

3. **If outage sustained >30 min** AND customer-facing impact is visible (opportunity discovery fully broken):
   - Open Slack `#platform-incidents` with status update.
   - Update status page using `status-page-comms-templates.md` SEV-1 Initial template (scope: "opportunity discovery temporarily unavailable due to upstream vendor outage").
   - Notify affected customers if known via email.

4. **If circuit breaker is stuck OPEN** after KraftData recovery (i.e., the half-open probe keeps failing):
   ```bash
   kubectl rollout restart deploy/ai-gateway -n eusolicit
   ```
   This resets the in-memory circuit state (acceptable because circuit state is intentionally in-memory per Epic 4 design; multi-replica state sharing via Redis is a pre-scale backlog item per circuit_breaker.py docstring).

5. **Opportunity data backfill**: once KraftData recovers, the Celery workers in data-pipeline will resume ingest automatically. No manual backfill needed unless there is a gap >24h.

---

## Verification

1. **Circuit-breaker transitions to CLOSED**:
   ```bash
   kubectl logs -n eusolicit deploy/ai-gateway --since=5m | grep -i "circuit" | tail -10
   ```
   Look for: `"state": "closed"`.

2. **KraftData-dependent error rate returns to 0**:
   ```promql
   sum(rate(http_request_errors_total{slo_target="kraftdata-dependent"}[5m]))
   ```
   Target: 0.

3. **Opportunity ingest resumes**: check data-pipeline logs for successful KraftData API calls.

4. **Platform SLO unaffected**: confirm `slo_target="platform"` burn rate < 1×:
   ```promql
   (1 - availability:slo:rate1h{slo_target="platform"}) / (1 - 0.999)
   ```

5. **`KraftDataDependentHighBurnRate` alert resolves** in PagerDuty.

---

## Rollback

Not applicable — this runbook covers an upstream vendor outage. Platform-side actions are limited to:
- Circuit breaker reset (AI-Gateway restart if stuck).
- Status page and customer-comms update.
- No code rollback is applicable; the circuit breaker IS the rollback mechanism.

If an AI-Gateway restart worsened something → follow `deploy-rollback.md`.

---

## Related

- `error-budget-burn.md` — platform SLO burn (distinct from KraftData-dependent burn)
- `deploy-rollback.md` — if AI-Gateway rollout is needed
- `severity-definitions.md` — SLA-scope table (KraftData listed as EXEMPT)
- Epic 4 `circuit_breaker.py` — `AgentCircuit` state machine
- Epic 4 `kraftdata_resilient.py` lines 21–73 — resilience composition (rate limiter → CB → retry)
- architecture.md line 762 — KraftData SLO isolation precedent
- PE.05 `kraftdata-isolation-rules.yaml` — alert definition
