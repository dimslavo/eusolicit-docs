# Story 25.03: Storage-Processed Webhook Handler + UI State Badge

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 3
**Type:** backend + frontend
**Dependencies:** S25.02 (KB row exists with sirmaai_file_id), S28.01 (subscription bootstrap), S28.03 (webhook receiver hardening)
**Blocks:** S25.07 (UI completeness)
**Created:** 2026-05-15
**Source:** E25 epic §S25.03

## Story

As **Elena who just uploaded a tender PDF**,
I want **the KB list row's "Processing…" badge to flip to "Ready" within a few seconds of SirmaAI finishing parsing**,
so that **I know my file is queryable by agents (qualifying, drafting) and not still pending**.

## Acceptance Criteria

1. **Backend consumer** in `sirmaai-gateway` (or `data-pipeline` per E28 §4 routing): subscribe to internal Redis Stream `client-api.kb_file_processed` (routed by E28 S28.04 from `sirmaai.kb.file.processed` SirmaAI webhook).
2. On event `{sirmaai_file_id, sirmaai_storage_resource_id, processed_at}`:
   - Look up `client.sirmaai_kb_files` row by `sirmaai_file_id` (scoped to company via row metadata).
   - Set `parsed_text_available_at = processed_at`.
   - Idempotent: duplicate webhook is a no-op (`UPDATE ... WHERE parsed_text_available_at IS NULL`).
3. **Reconciler safety net**: if SirmaAI webhook is dropped, the E28 reconciler converges the file's `processed_at` from a SirmaAI status poll within 5 minutes (NOT this story's responsibility — but document the interop).
4. **Frontend** KB List page (added in S25.07):
   - Subscribes to TanStack Query invalidation on `kb.file.processed` SSE channel: `GET /api/v1/kb/files/stream` (added by S25.04 or this story — coordinate).
   - Row state badge transitions visually: "Processing…" (yellow with spinner) → "Ready" (green with checkmark).
   - aria-live="polite" announces transitions for screen readers.
5. **SSE channel** in `client-api`: `GET /api/v1/kb/files/stream` opens an SSE stream filtered to the calling tenant's `company_id`. Pushes `{kb_file_id, parsed_text_available_at}` events on processed-state mutations.
6. **Performance**: SirmaAI webhook delivery → DB update → SSE push → UI badge flip < 5s p95 on dev environment.
7. **Test scenario**: file uploaded; SirmaAI webhook delayed by 60s; reconciler (S28.05) catches the missing state and converges; UI badge eventually flips. Verified end-to-end with mocked SirmaAI.
8. **Cross-tenant**: tenant A's webhook routes only to tenant-A SSE subscribers. Verified via two-tenant test.

## Dev Notes

### Pattern reuse
- SSE pattern: see `services/client-api/src/client_api/routers/proposals.py` for `/proposals/{id}/draft/stream` (ADR-005 SSE invariants).
- Redis Stream consumer: see `services/notification/src/notification/consumers/subscription_changed.py`.
- TanStack Query invalidation on SSE: see existing `frontend/apps/client/lib/hooks/useSSE.ts`.

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/consumers/kb_file_processed.py` (new)
- `services/client-api/src/client_api/routers/kb.py` (extend with SSE endpoint)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/kb/page.tsx` (badge + SSE subscription)
- `frontend/packages/ui/src/components/kb/FileProcessingBadge.tsx` (new)
- `services/client-api/tests/integration/test_kb_processed_webhook.py`

### Out of scope
- Webhook receiver itself (E28 S28.03 handles HMAC + idempotency + routing)
- Full KB Dashboard UI (S25.07)
- Reconciler convergence (E28 S28.05)

## Risks

- **R1**: SSE connection limits at edge proxy — typical limit is ~6 concurrent per origin per browser; not a concern for v1 (1 KB stream per session).
- **R2**: Webhook arriving before DB row exists (race between S25.02 commit and webhook) — handle via retry-once-then-dlq pattern; if no row after 30s retry, route to webhook_dlq.

## Testing

- Unit: idempotency on duplicate webhook; row-not-found handling.
- Integration: full upload → webhook → SSE → row update flow.
- Cross-tenant: scoped SSE delivery.

## See also

- Epic file §S25.03
- E28 §S28.03 + §S28.05 (receiver + reconciler dependencies)
- ADR-005 (SSE lifecycle invariants)
