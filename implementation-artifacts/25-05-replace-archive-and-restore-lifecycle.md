# Story 25.05: KB Replace + Archive + Restore Lifecycle

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 3
**Type:** backend + frontend
**Dependencies:** S25.02
**Blocks:** S24.07 (erasure iterates over archived files too)
**Created:** 2026-05-15
**Source:** E25 epic §S25.05

## Story

As **Elena who needs to swap an outdated profile document for a refreshed version**,
I want **a replace flow that preserves the file row's UUID + tags + category, and an archive/restore flow with a 30-day restore window**,
so that **historical links don't break, but I can also clean up unused artefacts without permanently losing recovery options**.

## Acceptance Criteria

1. **Endpoint** `PUT /api/v1/kb/files/{kb_file_id}` accepts a new multipart file body. Behavior:
   - Validate caller is the tenant scope; archived file → 410.
   - Delete the old SirmaAI file via `DELETE /files/{sirmaai_file_id}`.
   - Upload new body to SirmaAI via the same streaming pattern as S25.02.
   - Update `client.sirmaai_kb_files`: same UUID, new `sirmaai_file_id`, new `sha256`, new `size_bytes`, `parsed_text_available_at=NULL`, `updated_at=now()`. Preserve `tags`, `artefact_category`, `uploaded_by_user_id` (originally), filename (or update if explicitly different).
2. **Endpoint** `DELETE /api/v1/kb/files/{kb_file_id}`: soft-delete. Body: `{reason?: string}` optional.
   - Set `archived_at = now()`, `archived_reason = body.reason`.
   - SirmaAI-side: call `DELETE /files/{sirmaai_file_id}` to remove the artefact body within 10s.
   - Tier-quota recalculates excluding archived files.
3. **Endpoint** `POST /api/v1/kb/files/{kb_file_id}/restore`:
   - Valid only if `archived_at IS NOT NULL` AND `now() - archived_at ≤ 30 days`.
   - Body: required multipart file body (because the SirmaAI artefact body is GONE — caller must re-upload).
   - On valid window + valid body: re-upload to SirmaAI (returns new `sirmaai_file_id`); update `client.sirmaai_kb_files` row to clear `archived_at`, set new `sirmaai_file_id`, `sha256`, `size_bytes`, `parsed_text_available_at=NULL`.
   - On expired window (>30 days): return 410 Gone with `{error_code: "RESTORE_WINDOW_EXPIRED", archived_at, days_elapsed}`.
4. **Frontend**: KB Dashboard row actions menu adds Replace + Archive; archived list view has Restore button (warns on re-upload requirement).
5. **Cross-tenant**: scope check on all three endpoints; tenant A cannot touch tenant B's files.
6. **Audit log**: replace + archive + restore each write `shared.audit_log` row with `action_type='kb_{replace|archive|restore}'`.
7. **Integration test** covers: replace preserves UUID; archive deletes SirmaAI body within 10s; restore-within-30d succeeds; restore-beyond-30d returns 410.

## Dev Notes

### Pattern reuse
- Upload streaming pattern from S25.02.
- Soft-delete + restore pattern from existing project (Epic 14 partial-index).

### Files likely touched
- `services/client-api/src/client_api/routers/kb.py` (extend with 3 endpoints)
- `services/client-api/src/client_api/services/sirmaai_kb_service.py` (extend with `replace_file`, `archive_file`, `restore_file`)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/kb/page.tsx` (extend with action menu)
- `services/client-api/tests/integration/test_kb_lifecycle.py`

### Out of scope
- Self-service "permanent delete before 30d expires" — not in v1 (sweep + manual ops).
- Multi-file batch operations — single-file only for v1.
- Profile-update re-index (S25.06).

## Risks

- **R1**: Replace mid-search results — caller may see stale results briefly. Mitigate via SirmaAI re-indexing budget (~5min per E25 ACs); UX should warn.
- **R2**: Restore flow requires user to KEEP a local copy of the body — UX modal must spell this out at archive time.

## Testing

- Unit: replace flow happy path + UUID preservation; archive + 10s SirmaAI delete; restore window logic.
- Integration: full lifecycle with mocked SirmaAI.

## See also

- Epic file §S25.05
- PRD amendment FR-52
- UX amendment §5.11 (Artefact detail state rules)
