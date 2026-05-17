# Story 25.07: Tier-Gated Quota + Workspace KB Dashboard

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 5
**Type:** fullstack
**Dependencies:** S25.02 (upload), S25.04 (search + download)
**Blocks:** Slice 2 DoD
**Created:** 2026-05-15
**Source:** E25 epic §S25.07

## Story

As **Elena managing my company's KB artefacts**,
I want **a workspace KB Dashboard with quota meter, filterable artefact list, replace/archive actions, and an upgrade CTA when I'm approaching quota**,
so that **I can curate my KB efficiently and understand my tier's storage limits**.

## Acceptance Criteria

1. **Endpoint** `GET /api/v1/kb/files` returns paginated artefact list filtered by current tenant + optional `?category=<>` + sortable by `created_at | size_bytes`. Response includes per-row metadata (filename, category, processing-state, size, uploaded_by display name, tags) + `meta: {total_size_bytes, quota_limit_bytes, current_count}`.
2. **Endpoint** `GET /api/v1/kb/usage` returns `{used_bytes, quota_bytes, tier, files_count}` for the current tenant — drives the dashboard quota meter without a full list scan.
3. **Frontend page** at `/workspace/[workspaceId]/kb`:
   - **Quota meter** (top-right): radial progress bar with `used / quota` bytes; color bands (green <80%, yellow 80–95%, red >95%); click-through to billing if Pro tier and over quota.
   - **Filter rail** (left, collapsible on mobile): by `artefact_category`, by tag, by processing state (Processing / Ready / Archived).
   - **Artefact list** (right): card or table toggle; each row shows filename, category badge, processing badge, size, uploaded-at, action menu (replace / archive / search-within / download).
   - **Empty state**: large drop-zone + "Boost AI quality — upload past proposals or your company profile" copy (matches UX amendment §3 J1 step 4a).
   - **Upload zone**: drag-and-drop + keyboard fallback (button + file picker per WCAG 2.1 AA).
4. **Over-quota upload UX**:
   - Upload-zone blocks new file additions when at quota; shows inline "Upgrade to Professional for 5 GB of KB storage" CTA.
   - 402 from S25.02 endpoint surfaces a toast + the upgrade CTA.
5. **WCAG 2.1 AA**:
   - Keyboard navigation across full page (tab order: quota → filters → list → upload zone).
   - `aria-live="polite"` on quota meter updates (when an upload completes).
   - Drag-and-drop has keyboard fallback (Tab + Enter on button).
   - Sufficient contrast on processing badges + quota color bands.
   - Drag-and-drop affordance is announced to screen readers.
6. **E2E Playwright test** (`tests/e2e/kb-lifecycle.spec.ts`):
   - Pro tenant: upload PDF → see in list → archive → restore (within 30d) → search → result shown.
   - Free tenant: upload up to 50MB → 51MB blocked → upgrade-CTA visible.
7. **Tier-quota enforcement**: backend (S25.02) is canonical; frontend reflects backend state. UI never claims "you have space" when backend disagrees.

## Dev Notes

### Pattern reuse
- Existing pages under `frontend/apps/client/app/(protected)/workspace/[workspaceId]/` for layout.
- Upgrade modal: existing pattern for tier-gated features.
- Drag-and-drop + keyboard fallback: existing pattern in proposal-upload component (Epic 6 S06.12).

### Files likely touched
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/kb/page.tsx` (new — main page)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/kb/components/KbDashboard.tsx` (new)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/kb/components/QuotaMeter.tsx` (new)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/kb/components/ArtefactList.tsx` (new)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/kb/components/UploadZone.tsx` (new)
- `services/client-api/src/client_api/routers/kb.py` (extend with list + usage endpoints)
- `frontend/apps/client/messages/en.json` + `bg.json` (i18n keys for `kb.*`)
- `tests/e2e/kb-lifecycle.spec.ts` (new)

### Out of scope
- AI-assisted KB curation (tagging, deduplication) — post-launch.
- Cross-workspace KB sharing — post-launch (would need a `tenant_scope=true` flag per artefact).
- Bulk delete / batch operations.

## Risks

- **R1**: Quota meter eventual-consistency — small race after upload before usage endpoint reflects. Mitigate via optimistic UI (assume just-uploaded size is in total).
- **R2**: i18n parity — must add Bulgarian translations matching existing pattern; `pnpm check:i18n` gate enforces.

## Testing

- Unit: quota-meter color bands; over-quota CTA logic.
- E2E Playwright: full flow as in AC6.
- Cross-tenant: tenant-A page never shows tenant-B files.
- Accessibility: axe-core + manual keyboard nav.

## See also

- Epic file §S25.07
- UX amendment §5.11 (KB Dashboard state rules)
- PRD amendment FR-49 (tier-gated quotas)
