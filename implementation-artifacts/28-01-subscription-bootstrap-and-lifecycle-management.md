# Story 28.01: Webhook Subscription Bootstrap + Lifecycle Management

**Status:** backlog
**Epic:** E28 — Webhook & Reconciler Hardening
**Points:** 3
**Type:** backend
**Dependencies:** S04.20 done
**Blocks:** S28.02 (rotation), S28.03 (receiver), every webhook consumer
**Created:** 2026-05-15
**Source:** E28 epic §S28.01

## Story

As a **platform engineer deploying EU Solicit**,
I want **Standard Webhooks subscriptions to be registered against SirmaAI declaratively at deploy time (not manually via dashboard)**,
so that **rebuilds and rollouts don't depend on operator memory and every environment has identical subscription state**.

## Acceptance Criteria

1. On service deploy (or first start if no subscriptions exist), `sirmaai-gateway` calls `POST /api/webhooks/subscriptions` with the 4 event types: `workflow.completed`, `agent.run.completed`, `storage.file.processed`, `policy.violation`.
2. Each subscription persists to `gateway.webhook_subscriptions` table (already exists per E04 amendment S04.21 — extend if needed). Fields: `id`, `event_type`, `sirmaai_subscription_id`, `hmac_secret_encrypted` (Fernet), `hmac_rotated_at`, `is_active`, `created_at`, `updated_at`.
3. **Idempotency**: re-deploy with existing subscriptions = no-op. Lookup by `event_type` before re-creating.
4. **Admin endpoint** `GET /api/v1/admin/webhook-subscriptions` returns subscription list with status (active, HMAC last rotated, sirmaai_subscription_id).
5. **Hot-restart safety**: existing subscriptions are NOT deregistered on restart; only new event types trigger registration.
6. **Cold-start test** with no existing subscriptions → 4 subscriptions registered + 4 rows in `gateway.webhook_subscriptions`.
7. **Hot-restart test** with existing subscriptions → no duplicates.
8. **Failure mode**: if SirmaAI returns 5xx on subscription POST, retry with exponential backoff (1m, 5m, 30m). After 3 failures, log error + alert + DO NOT block service startup (degraded webhook ingest is acceptable; reconciler covers state).

## Dev Notes

### Pattern reuse
- Service-startup hook: FastAPI `lifespan` context manager pattern.
- Fernet encryption: existing canonical.

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/services/subscription_bootstrap.py` (new)
- `services/sirmaai-gateway/src/sirmaai_gateway/main.py` (lifespan call)
- `services/admin-api/src/admin_api/routers/sirmaai_subscriptions.py` (new — list endpoint)
- `services/sirmaai-gateway/tests/integration/test_subscription_bootstrap.py`

### Out of scope
- HMAC rotation (S28.02).
- Receiver hardening (S28.03).
- Per-tenant subscriptions (per ADR-018, subscriptions are platform-level under singleton Org).

## Risks

- **R1**: SirmaAI dashboard-edited subscriptions could drift from code state — admin endpoint surfaces this.

## Testing

- Cold-start, hot-restart per AC6+AC7.
- Failure path: SirmaAI 503 on bootstrap → retry budget + degraded warning.

## See also

- Epic file §S28.01
- PRD amendment FR-54
- ADR-018 (platform-level Org webhook subscriptions)
