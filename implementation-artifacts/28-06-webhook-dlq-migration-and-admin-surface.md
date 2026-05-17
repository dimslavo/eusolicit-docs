# Story 28.06: Webhook DLQ Migration + Admin Surface

**Status:** backlog
**Epic:** E28 — Webhook & Reconciler Hardening
**Points:** 3
**Type:** backend + frontend
**Dependencies:** S28.03 (receiver writes to DLQ on malformed events)
**Blocks:** operator playbook for poison events
**Created:** 2026-05-15
**Source:** E28 epic §S28.06 (closes readiness Concern #2)

## Story

As **Operator Ivan**,
I want **a DLQ admin endpoint to inspect, replay, or mark-resolved webhook events that failed schema validation**,
so that **poison events from SirmaAI don't sit forever AND I can fix-and-replay them after a schema bug is patched**.

## Acceptance Criteria

1. **Alembic migration** creates `gateway.webhook_dlq`:
   ```sql
   CREATE TABLE gateway.webhook_dlq (
       id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
       sirmaai_event_id    TEXT NOT NULL,
       sirmaai_event_type  TEXT NOT NULL,
       payload             JSONB NOT NULL,
       received_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
       failed_reason       TEXT NOT NULL,
       retry_count         INT NOT NULL DEFAULT 0,
       last_retry_at       TIMESTAMPTZ,
       resolved_at         TIMESTAMPTZ,
       resolved_by         UUID,
       resolution_note     TEXT
   );
   CREATE INDEX ix_webhook_dlq_unresolved
       ON gateway.webhook_dlq(received_at)
       WHERE resolved_at IS NULL;
   ```
2. **Admin endpoints**:
   - `GET /api/v1/admin/webhook-dlq?status=unresolved&limit=50` — list with payload excerpt + failure reason.
   - `GET /api/v1/admin/webhook-dlq/{id}` — full payload + failure-reason history + retry log.
   - `POST /api/v1/admin/webhook-dlq/{id}/replay` — re-enqueue (clear idempotency cache first; increment retry_count; record replay in audit).
   - `POST /api/v1/admin/webhook-dlq/{id}/mark-resolved` — manual close with `resolved_by` (admin user_id) and `resolution_note` (required).
3. **Admin UI page** lists DLQ entries with filter (status, event-type, time-range) + replay/resolve buttons.
4. **Replay round-trip test**: malformed event → DLQ → fix schema bug → call replay → successfully processed.
5. **Mark-resolved auditable**: every mark-resolved action writes audit row including resolution_note.
6. **Performance**: DLQ list query (50 rows) < 200ms with 10K total DLQ rows.

## Dev Notes

### Pattern reuse
- Admin-API router pattern.
- Existing audit-log helper.

### Files likely touched
- `services/sirmaai-gateway/alembic/versions/<next>_webhook_dlq.py` (new)
- `services/admin-api/src/admin_api/routers/webhook_dlq.py` (new)
- `services/sirmaai-gateway/src/sirmaai_gateway/services/dlq_service.py` (new)
- `frontend/apps/admin/app/(authenticated)/webhook-dlq/page.tsx` (new)
- `services/sirmaai-gateway/tests/integration/test_dlq_admin.py`

### Out of scope
- Auto-replay on schema upgrade (manual operator action).
- Bulk-replay UI (single-replay for v1).

## Risks

- **R1**: Replay infinite loop — increment `retry_count` + alert if >5.

## Testing

- Integration: full replay round-trip.
- Performance: 10K DLQ rows query.

## See also

- Epic file §S28.06 (closes readiness Concern #2)
- E04 amendment §webhook_dlq table reference
