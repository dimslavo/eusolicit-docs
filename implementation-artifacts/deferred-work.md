# Deferred Work

## Deferred from: code review of story 6-12-document-upload-component (2026-04-17)

- **Orphaned S3 files on confirm failure** — If `confirmDocumentUpload` fails after a successful S3 PUT, the file exists on S3 but is never confirmed or scanned. Requires server-side orphan cleanup (TTL-based S3 lifecycle rule) or client-side confirm retry logic.
- **No cleanup on component unmount** — Navigating away from the Documents tab mid-upload leaves XHR requests running and `onUploadComplete`/`refetch()` calls firing on stale closures. Requires `useEffect` cleanup with `AbortController` tracking via a `useRef<Map<string, AbortController>>`.
- **0-byte files pass client-side validation** — An empty file (0 bytes) passes all validation checks. Consider adding a minimum file size check or an `uploadFileEmpty` i18n error key.
- **Drag-and-drop flickering on child elements** — `onDragLeave` fires when hovering over child elements inside the drop zone, causing rapid toggling of the highlight. Fix with a `dragCounter` ref using paired `dragEnter`/`dragLeave` events.
- **`removeFromQueue` is dead code** — Defined via `useCallback` but never called; `handleCancel` uses inline filtering. Remove or use the existing helper.

## Deferred from: code review of story 6-13-ai-summary-panel-with-sse-streaming (2026-04-17)

- **`parseSSEStream` does not concatenate multi-line `data:` fields** — SSE spec allows multiple `data:` lines per event block (joined by newlines). The parser at `opportunities.ts:280` keeps only the last `data:` line per block. Backend S06.08 sends single-line data, so no current impact. Fix if backend SSE format evolves.
- **Elapsed time via `Date.now()` inaccurate under tab suspension** — `AISummaryPanel.tsx:77` records stream start time with `Date.now()`. Browser tab throttling/suspension inflates the elapsed value. Server-side timing in the `metadata` SSE event would be more accurate.

## Deferred from: code review of story 7-5-ai-draft-generation-backend-sse-integration (2026-04-17)

- **`espd_profile_summary` omitted from company payload (AC2)** — `build_generation_payload()` omits `espd_profile_summary` from the company data sent to the AI Gateway. AC2 specifies it but the Company model lacks the field. → **Story created: `dw-01-proposal-backend-schema-alignment.md`**
- **Celery task `reset_stuck_proposals_task()` lacks `@celery_app.task` decorator (AC7)** — Function exists but cannot be discovered by Celery Beat scheduler. Requires Celery app infrastructure decisions (which app instance, Beat configuration location). Service function works and is tested. → **Story created: `dw-02-celery-task-infrastructure-fixes.md`**
- **Error SSE event forwards raw AI Gateway data instead of fixed message (AC6)** — Spec says `{"error": "AI Gateway error"}`; implementation forwards actual gateway error details. More useful for debugging but deviates from spec and may expose internal details. → **Resolved: story 7-17** (single-point `_sanitize_generation_error()` helper, all error paths emit fixed strings)
- **`CancelledError` not caught in `_run_generation_task`** — On server shutdown (SIGTERM), task is cancelled but `generation_status` is not set to `failed`. Proposal stays in `generating` until cleanup task resets it (up to 300s). → **Resolved: story 7-17** (explicit `except asyncio.CancelledError` block added, sets status=failed and re-raises)
- **`set_generation_status` parameter lacks `Literal` type validation** — `status: str` accepts any string; invalid values cause unhandled DB CHECK constraint IntegrityError. All current callers pass valid values.
- **`_version_write_locks` dict grows unboundedly** — Every unique `proposal_id` adds a lock that is never evicted. Pre-existing S07.03 pattern; lock objects are ~100 bytes; not a practical concern at current scale.
- **`asyncio.run()` in Celery cleanup task incompatible with async workers** — `reset_stuck_proposals_task()` uses `asyncio.run()` which fails if an event loop is already running. Current infrastructure uses prefork pool (safe).

## Deferred from: code review of story 7-11-proposal-workspace-page-layout-navigation (2026-04-18)

- **Breakpoint threshold 1280px vs spec 1024px (AC5)** — `useBreakpoint` from `@eusolicit/ui` defines `isDesktop` at ≥1280px (Tailwind `xl`). AC5 requires panels expand at ≥1024px (Tailwind `lg`). Panels stay collapsed on 1024–1279px viewports. Requires shared UI hook change affecting all consumers. → **Story created: `dw-03-ui-breakpoint-hook-fix.md`**
- **`current_version_number` field absent from backend schema (AC11)** — Frontend `ProposalResponse` declares `current_version_number: number | null` but backend `ProposalDetailResponse` does not include this field. Runtime value is always `undefined` (shows fallback "v—"). Needs backend schema alignment. → **Story created: `dw-01-proposal-backend-schema-alignment.md`**
- **`generation_status` field missing from frontend interface (AC11)** — Backend returns `generation_status: str = "idle"` but frontend `ProposalResponse` interface omits it. Not needed by S07.11 UI; will be consumed in S07.13. → **Story created: `dw-01-proposal-backend-schema-alignment.md`**
- **`created_by` nullable mismatch (AC11)** — Backend has `created_by: UUID | None`, frontend declares `created_by: string` (non-nullable). Not rendered in S07.11 UI.
- **`status` type narrower than backend (AC11)** — Frontend uses `"draft" | "active" | "archived"` union but backend declares `status: str` (any string). Degrades gracefully for unknown values.
- **Window resize resets user's manual panel toggles** — `useEffect` with `[mounted, isDesktop]` deps resets panel collapse state on breakpoint crossing, overriding user's manual toggle. UX improvement, not a spec requirement.
- **No error boundary in server shell** — `proposals/[id]/page.tsx` renders `<ProposalWorkspacePage />` without `Suspense` or error boundary. Framework segment-level `error.tsx` may catch.
- **Hardcoded English in list placeholder** — `proposals/page.tsx` renders "Proposals list — future story" without i18n. Placeholder page will be fully replaced in a future story.
- **Unsafe `params.id` cast** — `params?.id as string` doesn't guard against `undefined` or array values. Works functionally due to `enabled: Boolean(id)` guard in hook.
- **Error handling in title PATCH silently swallowed** — `catch` block reverts `titleValue` but never shows error feedback. By design for S07.11; save status indicator will be wired in S07.12.
- **Enter+blur double-PATCH guard is closure-based, not ref-based** — The `if (!editingTitle) return;` guard uses React state captured in closures. Under React 18 batching, if blur fires from the same render cycle, the guard may not prevent a duplicate idempotent PATCH. A `useRef`-based guard or delegating Enter to `blur()` only would be deterministic. Low priority — impact is benign.
- **No client-side title maxLength** — Backend enforces `max_length=500` but the `<input>` has no `maxLength` attribute. Overlong titles are rejected server-side and silently reverted client-side with no user feedback.
