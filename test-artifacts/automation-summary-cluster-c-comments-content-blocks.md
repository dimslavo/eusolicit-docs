---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-identify-targets
  - step-03-generate-tests
  - step-04-validate-and-summarize
lastStep: step-04-validate-and-summarize
lastSaved: '2026-05-09'
workflowType: bmad-testarch-automate
mode: bmad-integrated
scope: cluster-C-comments-content-blocks
batch: P3-g
storyKeys:
  - 10-13-comments-sidebar
  - 7-16-content-blocks-library
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/components/ProposalWorkspacePage.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/components/CommentsSidebar.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/components/CommentList.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/components/ContentBlocksLibraryPanel.tsx
  - eusolicit-app/frontend/apps/client/lib/api/proposal-comments.ts
  - eusolicit-app/frontend/apps/client/lib/api/proposals.ts
  - eusolicit-app/frontend/apps/client/lib/queries/use-proposal-comments.ts
  - eusolicit-app/frontend/apps/client/lib/queries/use-proposals.ts
---

# Automation Summary: P3-g — Comments sidebar + Content Blocks library (cluster C, sub-run 7)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P3-g (cluster C, seventh net-new E2E coverage sub-run)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why these surfaces

Both surfaces sit inside the proposal workspace shell at
`/[locale]/workspace/[workspaceId]/proposals/[id]` and were entirely
uncovered by Playwright before this run:

- **Comments sidebar** (Story 10.13) — opened from `toolbar-btn-comments`,
  renders 5 distinct states (no-section / loading / error / empty / populated),
  drives a write path with optimistic updates and resolution toggling, and
  feeds the toolbar count badge via `useCommentsSummary`.
- **Content Blocks library** (Story 7.16) — left-panel tab that lists
  reusable boilerplate, supports 300ms-debounced search, category-chip
  filtering, and inline insertion into the active Tiptap section. Sources
  blocks from `/api/v1/content-blocks` (list) and `/api/v1/content-blocks/search`
  (search).

The proposal workspace itself is one of the heaviest Next.js client surfaces
in the app — driving these tests through the live shell exercises the
toolbar wiring, panel toggles, and Sheet portal rendering that earlier specs
(approvals, generate-error-display) bypassed by visiting sub-routes.

---

## Files Created

### `e2e/specs/proposals/comments-content-blocks.spec.ts` (NEW)

4 tests across 2 describe scopes:

| # | Surface | Scope | Description |
|---|---|---|---|
| 1 | Comments sidebar | P1 | Open the sidebar via `toolbar-btn-comments`. Assert `comments-sidebar` shell + title + close button visible; the no-section empty CTA `comments-sidebar-empty-noselection` renders because no editor section is active on first paint. Skeleton/error/empty-nocomments testids absent. |
| 2 | Content Blocks library | P0 | Mock 2 blocks (one approved + categorised "pricing", one unapproved + "compliance"). Toggle left panel + switch to content-library tab. Assert `content-blocks-search-input`, `category-chip-all`, two category-chip-{name}, two `content-block-card-{id}`, two `btn-insert-block-{id}`, `content-block-approved-{id}` only on the approved block. Loading/error/empty branches absent. |
| 3 | Content Blocks library | P1 | Empty list response → `content-blocks-empty` CTA visible; no cards/error/loading. |
| 4 | Content Blocks library | P1 | Mock returns 500 on every list/search hit. After react-query exhausts retries, `content-blocks-error` + `btn-content-blocks-retry` are visible; `content-blocks-empty` absent. |

### Mock helpers

- `mockCompanyProfile(page)` — same pattern as P3-{a..f}.
- `mockProposal(page, opts?)` — `GET /api/v1/proposals/{id}` returning a
  minimal `ProposalResponse` with one Tiptap section (`executive-summary`),
  pre-set `current_version_id`, and `current_version_number: 1`. The body
  shape mirrors `lib/api/proposals.ts:21-31`.
- `mockProposalSidecars(page)` — collaborators (empty), section locks
  (empty), and the comments endpoint (empty list, total=0). The comments
  glob `**/api/v1/proposals/{id}/comments**` deliberately swallows BOTH the
  full-summary call and section-scoped calls because both go through the
  same path with different query params.
- `mockContentBlocks(page, payload)` — single glob `**/api/v1/content-blocks**`
  serves both the list (`/content-blocks`) and search (`/content-blocks/search`)
  endpoints. Switch shape `{ok: [...]}` for happy responses, `{error: true}`
  for the 500-error branch.
- `mountWorkspace(page)` — composer that wires company + proposal + sidecars
  in one call.

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for new file | ✅ 0 errors |
| `npx playwright test --list` filtered to new file | ✅ **4 tests collected** in 1 spec file |
| Active vs fixme split | 4 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| Route `/en/workspace/default/proposals/{id}` exists | `app/.../proposals/[id]/page.tsx` | ✅ |
| `proposal-toolbar` + `toolbar-btn-comments` button | `ProposalWorkspacePage.tsx:180` | ✅ |
| `left-panel`, `left-panel-toggle`, `left-tab-content-library` | `ProposalWorkspacePage.tsx` left-aside block | ✅ |
| `comments-sidebar` Sheet wrapper | `CommentsSidebar.tsx:49` | ✅ |
| `comments-sidebar-title`, `comments-sidebar-close-btn` | `CommentsSidebarHeader.tsx:34, 49` | ✅ |
| `comments-sidebar-empty-noselection` (no active section) | `CommentList.tsx:34` (gated on `!sectionKey`) | ✅ |
| `content-blocks-search-input` | `ContentBlocksLibraryPanel.tsx:58` | ✅ |
| `category-chip-all` + per-category chips | `ContentBlocksLibraryPanel.tsx:70, 84` | ✅ |
| `content-block-card-{id}` + `content-block-preview-{id}` + `btn-insert-block-{id}` | `ContentBlocksLibraryPanel.tsx:139, 154, 172` | ✅ |
| `content-block-approved-{id}` only when `approved_at` is non-null | `ContentBlocksLibraryPanel.tsx:144-151` (conditional on `block.approved_at`) | ✅ |
| `content-blocks-empty` for blocks=[] | `ContentBlocksLibraryPanel.tsx:130` (gated on `blocks.length === 0`) | ✅ |
| `content-blocks-error` + `btn-content-blocks-retry` for 500 | `ContentBlocksLibraryPanel.tsx:113, 118` (gated on `isError && !isLoading`) | ✅ |
| GET `/api/v1/proposals/{id}` (detail) | `lib/api/proposals.ts:67` | ✅ |
| GET `/api/v1/proposals/{id}/comments?…` (list + summary share path) | `lib/api/proposal-comments.ts:72` | ✅ |
| GET `/api/v1/content-blocks` (list) | `lib/api/proposals.ts:479` | ✅ |
| GET `/api/v1/content-blocks/search?q=…` (search) | `lib/api/proposals.ts:485` | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/proposals/comments-content-blocks.spec.ts \
    --project=client-chromium --reporter=list
```

### Risks / open questions for the CI pass

1. **Tiptap editor mount timing**: the proposal workspace dynamically imports
   `ProposalEditor` (SSR off). With version content set, the editor mounts
   asynchronously. The toolbar testids render synchronously from the proposal
   detail response, so the `toolbar-btn-comments` click fires reliably before
   the editor settles. If a future change moves the toolbar inside the editor
   subtree, these tests will need an explicit `proposal-editor-area` wait.

2. **Content blocks search debounce**: `ContentBlocksLibraryPanel.tsx:23`
   debounces 300ms before swapping `listQuery` → `searchQuery`. Tests don't
   type into the search box, so only the list endpoint is hit. The single
   `**/api/v1/content-blocks**` glob would also catch the search call
   if a future test exercises typing.

3. **Sheet portal rendering**: `comments-sidebar` lives inside a Radix
   `<Sheet>` portal. `page.getByTestId` traverses portals, so visibility
   assertions work — but `toHaveCount(0)` for related testids relies on the
   Sheet only mounting children when `open=true`. Closing the Sheet via
   `comments-sidebar-close-btn` is asserted as visible-on-open only.

4. **`useMyProposalRole` with empty collaborators + member auth**: the
   default `authenticatedPage` mints `role=member`. With an empty
   collaborator list, `useMyProposalRole.role === null` → `canMutate=false`,
   so `NewCommentForm` renders the `new-comment-readonly-note` branch. The
   existing test only asserts the no-section empty CTA, so this is benign;
   a future "comment write" test should seed an admin auth-store (same
   pattern as P3-b approvals).

5. **`/comments**` glob aggressiveness**: this single handler claims every
   GET on the comments path including the carry-forward banner's summary
   call (`use-proposal-comments.ts:73`). Returning the same empty payload
   for all paths is safe because `bySection` is omitted, so the banner +
   toolbar count badge both render the no-comments branch.

6. **Deferred follow-ups for P3-g**:
   - Comments — populated list with a fixed `activeSectionKey` (would need
     to drive the editor or seed `useProposalEditorStore`), plus resolve /
     unresolve PATCH captures and the carry-forward banner branch.
   - Content blocks — debounced search (verify the search endpoint is hit
     after typing 3+ chars), category-chip filter active state via
     `data-selected="true"`, and Tiptap insert assertion (`btn-insert-block`
     click → editor body now contains the block body).

---

## Coverage Delta

- Before P3-g: **0** Playwright tests for either surface (comments sidebar
  was tested only by Vitest unit specs; content blocks had a single
  `__tests__/version-history-content-blocks-export-s7-16.test.ts` Vitest
  file, no E2E).
- After P3-g: **4 active** user-level E2E tests covering sidebar-open /
  blocks-populated / blocks-empty / blocks-error scopes.

---

## Cluster C running totals (through P3-g)

| Sub-run | Spec file(s) | Active | Fixme |
|---|---|---:|---:|
| P3-a | `tasks-kanban.spec.ts` | 5 | 0 |
| P3-b | `approvals.spec.ts` | 5 | 0 |
| P3-c | `analytics-pipeline-roi.spec.ts` | 5 | 0 |
| P3-d | `analytics-rest.spec.ts` | 8 | 0 |
| P3-e | `reports/reports.spec.ts` | 5 | 0 |
| P3-f | `calendar/calendar-settings.spec.ts` + `bid-outcomes/bid-outcomes.spec.ts` | 6 | 0 |
| P3-g | `proposals/comments-content-blocks.spec.ts` | 4 | 0 |
| **Total** | **8 spec files** | **38** | **0** |

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P3-h** — Admin UI (separate base URL + auth — own run).
- **P4-{a..c}** — API gap fill (NPS / billing / cross-service notification).

After P3-g runs green in CI, update the rollout-plan checklist line.
