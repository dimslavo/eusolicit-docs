# Story 28.03: Webhook Receiver Hardening + Idempotency Cache

**Status:** backlog
**Epic:** E28 — Webhook & Reconciler Hardening
**Points:** 5
**Type:** backend
**Dependencies:** E04 S04.25 (receiver scaffold) done; S28.01 (subscriptions)
**Blocks:** S25.03, S26.07
**Created:** 2026-05-15
**Source:** E28 epic §S28.03

## Story

As a **platform engineer**,
I want **the SirmaAI webhook receiver to be HMAC-verified, idempotent, schema-validated, and DLQ-aware**,
so that **silent retries, signature attacks, and poison events don't corrupt EU Solicit's canonical state**.

## Acceptance Criteria

1. Endpoint `POST /webhooks/sirmaai` (already scaffolded in E04 S04.25); harden per below.
2. **HMAC verification**: parse Standard Webhooks signature header → look up subscription's HMAC secret (current + previous during 24h rotation overlap per S28.02) → `hmac.compare_digest()` against expected. **Rule 48 — never `==`**. On mismatch → 401 + `shared.audit_log` row with `action_type='webhook_signature_mismatch'`.
3. **Idempotency cache**: Redis SETNX on `webhook_event:{sirmaai_event_id}` with 7-day TTL.
   - Duplicate → 200 (no Stream publication, increment `sirmaai_webhook_idempotency_hits_total` counter).
   - First-time → proceed.
4. **Schema validation**: validate payload against expected per-event-type schema (Pydantic models per event type). Malformed → DLQ insert (S28.06) + admin alert + 202 (acknowledge receipt to prevent SirmaAI retry storm).
5. **Successful events routed** to internal Redis Stream per S28.04 mapping.
6. **Failure-closed on Redis down**: if SETNX call fails because Redis is unreachable → 503 (encourages SirmaAI retry rather than risk dropped event).
7. **TLS enforcement**: receiver is HTTPS-only at edge (verified via nginx config).
8. **Negative tests**:
   - Signature mismatch → 401 + audit row + no Stream publication.
   - Duplicate event → 200 + idempotency hit counter increment.
   - Malformed payload → row in `gateway.webhook_dlq` + no Stream publication + 202.
   - Redis SETNX failure (Redis unavailable) → 503.
9. **Performance**: median latency < 100ms; p99 < 500ms (including Redis SETNX + DB lookup + Stream xadd).

## Dev Notes

### Pattern reuse
- E04 S04.25 receiver scaffold (already shipped).
- HMAC `compare_digest` (Rule 48).
- Redis SETNX idempotency (Epic 9 dual-layer pattern).

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py` (extend with hardenings)
- `services/sirmaai-gateway/src/sirmaai_gateway/services/webhook_idempotency.py` (new — Redis SETNX wrapper)
- `services/sirmaai-gateway/src/sirmaai_gateway/services/webhook_signature.py` (extend with rotation-overlap support)
- `services/sirmaai-gateway/src/sirmaai_gateway/schemas/webhook_events.py` (new — Pydantic per event type)
- `services/sirmaai-gateway/tests/integration/test_webhook_receiver.py`

### Out of scope
- DLQ admin surface (S28.06).
- Internal-routing detail (S28.04).
- Degraded-mode banner (S28.07).

## Risks

- **R1**: SirmaAI may use a non-Standard-Webhooks signature format — confirm against their OpenAPI; adapt verification logic.
- **R2**: Idempotency hit counter cardinality — bound to event-type label, not event-id.

## Testing

- Unit: each negative path.
- Integration: full receiver path with mocked SirmaAI signing.
- Performance: latency budget.

## See also

- Epic file §S28.03
- E04 amendment S04.25 (receiver scaffold)
- PRD amendment FR-54
