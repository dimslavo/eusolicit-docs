# Story 26.07: Webhook-Driven Result Persistence + SSE Notification

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 3
**Type:** backend + frontend
**Dependencies:** S26.04 (analyses service), S28.03 (receiver hardening), S28.04 (event routing)
**Blocks:** S26.08 (UI consumes SSE)
**Created:** 2026-05-15
**Source:** E26 epic §S26.07

## Story

As **Elena watching an in-flight quantification run**,
I want **the result panel to flip from "Running…" to "Completed" within 5 seconds of the agent finishing**,
so that **I don't have to refresh the page or wait for the next poll cycle**.

## Acceptance Criteria

1. Consumer in `sirmaai-gateway` (or `client-api`, per E28 routing) subscribed to internal Redis Stream `gateway.workflow_runs_converge`. On `agent.run.completed` events:
   - Look up `client.opportunity_analyses` row by `eusolicit_run_id`.
   - Validate response payload via Pydantic against analysis_type's structured-output schema.
   - On valid: `record_completion(eusolicit_run_id, payload, kb_citations)`.
   - On invalid: `record_failure(eusolicit_run_id, error_message)` with parsing details.
2. **SSE endpoint** in `client-api`: `GET /api/v1/opportunities/{id}/analyses/stream` opens an SSE stream scoped to the calling tenant. Pushes `{eusolicit_run_id, analysis_type, status, payload?}` events on `record_completion` + `record_failure`. ADR-005 SSE invariants observed (quota check before stream, fresh session, terminal event).
3. **Concurrency**: webhook + reconciler can both fire for the same run — `record_completion` is idempotent via `UPDATE ... WHERE status IN ('pending','running')` guard (per E28 idempotency rules).
4. **Performance**: webhook → DB update → SSE push < 5s p95.
5. **Failure-mode test**: malformed payload → `failed` status set + `error_message` populated + SSE pushes failure event.
6. **SSE lifecycle**: terminal event after completion; client closes connection; server-side cleanup.

## Dev Notes

### Pattern reuse
- SSE: existing pattern (ADR-005).
- Redis Stream consumer.
- Pydantic validation.

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/consumers/agent_run_completed.py` (new)
- `services/client-api/src/client_api/routers/opportunities.py` (extend with SSE)
- `services/client-api/tests/integration/test_analyses_sse.py`

### Out of scope
- The webhook receiver itself (E28 S28.03).
- The UI consumer (S26.08).
- Reconciler (E28 S28.05).

## Risks

- **R1**: SSE timeout on long quantification runs (3 min) — heartbeat events every 30s prevent proxy-close.
- **R2**: Webhook + reconciler race → idempotency guard required (UPDATE with status-filter WHERE clause).

## Testing

- Integration: completion event end-to-end.
- Failure mode: malformed payload.
- SSE lifecycle: terminal event observed.

## See also

- Epic file §S26.07
- ADR-005 (SSE)
- E28 §S28.03 + §S28.04
