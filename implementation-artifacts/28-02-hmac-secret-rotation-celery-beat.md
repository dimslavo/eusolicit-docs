# Story 28.02: HMAC Secret Rotation Celery Beat

**Status:** backlog
**Epic:** E28 — Webhook & Reconciler Hardening
**Points:** 3
**Type:** backend
**Dependencies:** S28.01 (subscriptions exist)
**Blocks:** safe long-term webhook ops
**Created:** 2026-05-15
**Source:** E28 epic §S28.02

## Story

As a **platform engineer responsible for the long-term security posture**,
I want **a Beat task that rotates each subscription's HMAC shared secret on a 90-day cadence with a 24-hour double-validation overlap**,
so that **a compromised HMAC secret has a bounded blast radius AND rotation doesn't drop in-flight signed events**.

## Acceptance Criteria

1. Celery Beat task `rotate_webhook_hmac_secrets` runs daily at 02:00 UTC.
2. For each `gateway.webhook_subscriptions` row WHERE `hmac_rotated_at IS NULL OR hmac_rotated_at < now() - INTERVAL '83 days'`:
   - (1) Generate new secret via `secrets.token_urlsafe(32)`.
   - (2) Register a NEW subscription on SirmaAI side with the new secret (does NOT replace existing yet).
   - (3) Send a test event via `POST /api/webhooks/subscriptions/{new_id}/test`; verify it round-trips through EU Solicit's receiver. On failure → abort rotation; alert.
   - (4) On successful test: deregister OLD subscription via `DELETE /api/webhooks/subscriptions/{old_id}`. EU Solicit's `gateway.webhook_subscriptions` row updated: `hmac_secret_encrypted=new`, `hmac_rotated_at=now()`, `sirmaai_subscription_id=new_id`.
3. **Double-validation overlap**: minimum 24h window where receiver accepts BOTH old + new secrets. Receiver code (S28.03) holds the previous secret in a separate column or shadow record for this purpose; S28.02 updates the active one without instantly purging the previous.
4. **Failure path**: rotation failure (e.g. SirmaAI 500 on register OR test event fails) → existing subscription unchanged; new subscription orphaned (cleanup task picks it up); alert admin.
5. **Audit log**: each rotation step writes `shared.audit_log` row with `action_type='webhook_secret_rotation_step'`.
6. **End-to-end test on staging**: synthetic 1-day-cadence override → rotation completes successfully + receiver continues accepting events during the overlap.

## Dev Notes

### Pattern reuse
- Celery Beat scheduling.
- Fernet encrypt.

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/tasks/hmac_rotation.py` (new)
- `services/sirmaai-gateway/src/sirmaai_gateway/celery_app.py` (Beat entry)
- `services/sirmaai-gateway/src/sirmaai_gateway/models/webhook_subscription.py` (extend with `previous_hmac_secret_encrypted` column + `previous_secret_valid_until`)
- `services/sirmaai-gateway/alembic/versions/<next>_webhook_previous_secret.py` (migration)
- `services/sirmaai-gateway/tests/integration/test_hmac_rotation.py`

### Out of scope
- Per-event-type rotation cadence override (single 90-day default).

## Risks

- **R1**: Concurrent rotation + webhook delivery — receiver accepts both during the 24h window; safe.

## Testing

- Integration: end-to-end rotation on staging.
- Failure: register failure / test-event failure / deregister failure paths.

## See also

- Epic file §S28.02
- PRD amendment NFR-24 (HMAC rotation)
