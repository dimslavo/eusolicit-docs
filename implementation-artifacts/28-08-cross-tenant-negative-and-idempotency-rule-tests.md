# Story 28.08: Cross-Tenant Negative + Idempotency-Rule Tests

**Status:** backlog
**Epic:** E28 — Webhook & Reconciler Hardening
**Points:** 2
**Type:** integration
**Dependencies:** S28.01..S28.07 done
**Blocks:** Sprint 5 quality gate
**Created:** 2026-05-15
**Source:** E28 epic §S28.08

## Story

As **Murat (TEA)**,
I want **a regression test suite that asserts cross-tenant isolation, webhook/reconciler race-safety, HMAC rotation overlap correctness, and DLQ replay end-to-end**,
so that **the webhook + reconciler hardening doesn't silently regress on future refactors**.

## Acceptance Criteria

1. Test suite at `services/sirmaai-gateway/tests/integration/test_webhook_reconciler_invariants.py` covering ≥10 scenarios:
   - **Cross-tenant**: webhook for tenant A's Project routes only to tenant-A-scoped consumers; tenant B subscriber cannot read tenant A's events from Redis Streams (existing scope-filter pattern).
   - **Webhook + reconciler race**: race scenarios where both fire → consumer guard prevents double-update.
   - **Webhook dropped + reconciler recovers**: synthetic webhook drop → reconciler converges within 5 minutes.
   - **HMAC rotation overlap**: events signed with old secret during 24h overlap → accepted. Events signed with neither secret AFTER overlap → 401.
   - **Idempotency duplicate**: same event_id arriving twice → second is 200 (no Stream publication; hit counter +1).
   - **Malformed payload → DLQ**: payload missing required field → 202 + row in `gateway.webhook_dlq`.
   - **DLQ replay**: fix mock, call replay → event processed successfully.
   - **Signature mismatch attack**: random signature → 401 + audit row.
   - **Redis-down failure-closed**: SETNX fails → 503.
   - **TLS-only**: HTTP request → rejected at edge (nginx config test).
2. **Load test**: 1000 concurrent webhooks + reconciler scan in parallel → no data corruption, no double-update.
3. **Tests run on every CI build** as part of integration suite.

## Dev Notes

### Pattern reuse
- pytest fixtures for cross-tenant test setup (existing).
- Mock SirmaAI client for signing webhooks.

### Files likely touched
- `services/sirmaai-gateway/tests/integration/test_webhook_reconciler_invariants.py` (new)
- `services/sirmaai-gateway/tests/load/test_webhook_concurrent_load.py` (new — k6-style or Locust)

### Out of scope
- E2E user-facing tests (those live in client-api/frontend).

## Testing

- Self-testing.

## See also

- Epic file §S28.08
- All E28 stories
