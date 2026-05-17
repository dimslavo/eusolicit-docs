# Story 28.05: Reconciler Hardening + Observability

**Status:** backlog
**Epic:** E28 — Webhook & Reconciler Hardening
**Points:** 3
**Type:** backend
**Dependencies:** E04 S04.26 (reconciler scaffold) done
**Blocks:** Slice 5 DoD (run-state integrity)
**Created:** 2026-05-15
**Source:** E28 epic §S28.05

## Story

As a **platform engineer**,
I want **the 5-minute reconciler to be the authoritative source of run-state truth with backoff, thundering-herd protection, and rich observability**,
so that **a webhook outage never silently loses a run AND the reconciler itself doesn't melt SirmaAI under load**.

## Acceptance Criteria

1. Builds on E04 S04.26 (basic 5-min reconciler). Hardenings:
   - **Skip rows** where `last_polled_at < now() - INTERVAL '30 seconds'` — prevents thundering herd on staggered deploys.
   - **Exponential backoff** on SirmaAI rate-limit (429): per-affected-tenant backoff to 10-min cadence. Metric `sirmaai_reconciler_rate_limit_backoff_active{company_id}`.
   - **Bounded parallelism** within a single reconciler tick: max 20 concurrent SirmaAI status calls.
2. **Prometheus exports**:
   - Gauge `sirmaai_workflow_runs_nonterminal_total{run_type}` — count of pending/running runs by type.
   - Gauge `sirmaai_webhook_dlq_depth_total` — DLQ row count.
   - Counter `sirmaai_reconciler_polls_per_minute_total`.
   - Histogram `sirmaai_run_to_terminal_seconds` — time from `started_at` to terminal status.
3. **Alert rule**: `sirmaai_workflow_runs_nonterminal_total > 100` sustained for 30 minutes fires `ticket`-severity alert via Alertmanager (per onprem-03 Slack/email/Telegram receivers per ADR-010 launch posture).
4. **Reconciler scan SLA**: completes within the 5-minute cadence at 10,000 non-terminal rows (load test required).
5. **Rate-limit handling test**: simulate SirmaAI 429 → reconciler backs off to 10-min cadence for that tenant; metric `sirmaai_reconciler_rate_limit_backoff_active=1`.
6. **Alert firing test**: synthetic backlog → alert fires after 30-min sustained → alert resolves after backlog drains.
7. **Idempotency** with webhook (per S28.04): reconciler `UPDATE ... WHERE status IN ('pending','running')` guard prevents double-update.

## Dev Notes

### Pattern reuse
- Existing reconciler from E04 S04.26.
- Prometheus gauges + histograms via `prometheus_client`.
- `asyncio.Semaphore(20)` for bounded parallelism.

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/tasks/workflow_runs_reconciler.py` (extend)
- `infra/observability/prometheus/rules/sirmaai-alerts.yaml` (extend)
- `infra/observability/grafana/dashboards/sirmaai-reconciler.json` (new)
- `services/sirmaai-gateway/tests/integration/test_reconciler_hardening.py`

### Out of scope
- DLQ admin surface (S28.06).
- Degraded-mode banner (S28.07).

## Risks

- **R1**: Reconciler scan time exceeds 5-min budget at scale — load test catches; tune batch size / concurrency.

## Testing

- Integration: rate-limit backoff path; idempotency-race with webhook.
- Performance: 10K non-terminal rows scan within 5min.

## See also

- Epic file §S28.05
- E04 amendment S04.26 (scaffold)
- PRD amendment FR-55, NFR-26
