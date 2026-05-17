# Story 26.06: Auto-Trigger on opportunities.ingested

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 3
**Type:** backend
**Dependencies:** S26.03 (workflow), S26.04 (analyses service), S05.24 (staged-rollout) done
**Blocks:** Slice 3 end-to-end cascade
**Created:** 2026-05-15
**Source:** E26 epic §S26.06

## Story

As **Elena who hasn't opened the app yet today**,
I want **qualification analysis to run automatically on every new opportunity ingested for my tenant**,
so that **when I log in I see a triaged list with pursue/monitor/decline tags, not 30 raw items I have to read one by one**.

## Acceptance Criteria

1. Consumer in `data-pipeline` subscribed to Redis Stream `opportunities.ingested`. On each event `{company_id, opportunity_id, source}`:
   - Dispatch `start_analysis(company_id, opportunity_id, "qualification", force=False)` per S26.04 service.
   - Idempotency via S26.04 handles dupe events.
2. On qualification completion webhook (S26.07 routes it), if `recommended_action='pursue'` AND tenant is Pro+ tier: auto-dispatch `start_analysis(company_id, opportunity_id, "quantification")`.
3. **Throttle**: per-tenant max 100 concurrent in-flight analyses (separate counter from S26.05 endpoint rate-limit). Gauge: `sirmaai_concurrent_runs{company_id}`. Above threshold → push to a deferred-queue with 5-min retry delay (DON'T drop).
4. **End-to-end SLA**: opportunity-ingested event → qualification result persisted ≤ 60s p95 steady state.
5. **Cascade test**: synthetic opportunity ingestion (via E05 path) → auto-qualification → on `pursue` outcome, auto-quantification fires (Pro+ tenant) → both results visible via `GET /api/v1/opportunities/{id}/analyses`.
6. **Throttle test**: bulk ingest 150 opportunities for one tenant; 100 dispatch immediately; 50 queue; queue drains within 10 min.
7. **Tier-aware**: on Free / Starter tenant, qualification fires but quantification is skipped silently (NO 402 — just no quantification analysis is created). Verified.

## Dev Notes

### Pattern reuse
- Redis Streams consumer pattern: `services/notification/src/notification/consumers/`.
- Throttle: atomic Redis Lua key per tenant.

### Files likely touched
- `services/data-pipeline/src/data_pipeline/consumers/opportunity_ingested.py` (new)
- `services/client-api/src/client_api/services/opportunity_analysis_service.py` (extend with `auto_cascade_on_qualification_complete`)
- `services/client-api/tests/integration/test_auto_qualification_cascade.py`

### Out of scope
- User-trigger (S26.05).
- SSE notification (S26.07).
- UI (S26.08).

## Risks

- **R1**: Ingestion bursts blow throttle — deferred-queue mitigates; if backlog grows, alert operator.
- **R2**: Sub-second event volume → consumer-group lag; verify lag stays < 100 messages under normal flow.

## Testing

- Integration: end-to-end cascade.
- Throttle: 150-burst scenario.
- Tier: Pro+ vs Starter cascade differs correctly.

## See also

- Epic file §S26.06
- PRD amendment FR-47, FR-48 (auto-trigger contract)
