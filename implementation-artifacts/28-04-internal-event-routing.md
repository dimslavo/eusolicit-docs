# Story 28.04: Internal Event Routing

**Status:** backlog
**Epic:** E28 — Webhook & Reconciler Hardening
**Points:** 3
**Type:** backend
**Dependencies:** S28.03 (receiver)
**Blocks:** S25.03, S26.07 (downstream consumers)
**Created:** 2026-05-15
**Source:** E28 epic §S28.04

## Story

As a **platform engineer**,
I want **the webhook receiver to route SirmaAI events to the correct internal Redis Stream(s) for downstream consumers**,
so that **`data-pipeline`, `client-api`, and `notification` services don't need to talk to SirmaAI directly — they just consume from a Stream**.

## Acceptance Criteria

1. Map SirmaAI event types to internal Redis Streams per architecture amendment §5.3:
   - `sirmaai.workflow.completed` → `data-pipeline.opportunity_completion` + `client-api.user_notifications`
   - `sirmaai.agent.run.completed` → `gateway.workflow_runs_converge` + `notification.run_completion_alerts`
   - `sirmaai.kb.file.processed` → `client-api.kb_file_processed`
   - `sirmaai.policy.violation` → `notification.policy_alerts` + `shared.audit_log` write
2. **Idempotency-discipline rule** documented in route-handler docstring: consumers MUST guard updates with `UPDATE ... WHERE status IN ('pending','running')` since the reconciler may have already converged.
3. **Consumer-side idempotency test**: webhook + reconciler race → final state correct exactly once.
4. **Stream consumer-group lag** exported as Prometheus metric `redis_stream_consumer_group_lag{stream, group}`.
5. **Routing config** in `services/sirmaai-gateway/src/sirmaai_gateway/services/event_router.py` is data-driven (a dict/yaml mapping), NOT hardcoded if/else — makes adding new event types a one-line change.
6. **Each event type → expected consumer Stream(s)** verified via integration test.

## Dev Notes

### Pattern reuse
- Existing Redis Streams `xadd` pattern.

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/services/event_router.py` (new)
- `services/sirmaai-gateway/src/sirmaai_gateway/config/event_routing.yaml` (new — declarative mapping)
- `services/sirmaai-gateway/tests/integration/test_event_routing.py`
- Consumer-side downstream tests (modular per consumer)

### Out of scope
- Consumer implementations (those live in their respective services).
- Reconciler (S28.05).

## Risks

- **R1**: A consumer that doesn't honour the idempotency rule could double-update. Cross-service code review enforces.

## Testing

- Integration: each event → expected Stream(s).
- Idempotency race: webhook + reconciler interleaved → final state correct.

## See also

- Epic file §S28.04
- Architecture amendment §5.3
