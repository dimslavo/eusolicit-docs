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
scope: cluster-C-tasks-kanban
batch: P3-a
storyKeys:
  - 10-5-task-crud-api
  - 10-6-task-dependencies-dag-validation
  - 10-14-task-kanban-board-detail-modal
  - 10-15-task-template-manager-ui
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/services/client-api/src/client_api/api/v1/tasks.py
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/TasksBoardPage.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/TaskBoard.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/TaskColumn.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/TaskCard.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/TaskBoardHeader.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/TaskDetailModal.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/BoardEmptyState.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/BoardErrorState.tsx
  - eusolicit-app/frontend/apps/client/lib/api/tasks.ts
  - eusolicit-app/frontend/apps/client/lib/queries/use-tasks.ts
  - eusolicit-app/frontend/apps/client/lib/api/members.ts
  - eusolicit-app/frontend/apps/client/messages/en.json
---

# Automation Summary: P3-a — Tasks/Kanban (cluster C, sub-run 1)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P3-a (cluster C, first net-new E2E coverage sub-run)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why this surface

Cluster C of the rollout plan covers user-facing features that shipped without
E2E coverage. Tasks/Kanban is the highest-DAU surface in epic 10:

- 4 epic-10 stories implement it (10-5, 10-6, 10-14, 10-15).
- ~15 React components ship under `app/[locale]/(protected)/workspace/[workspaceId]/tasks/components/`.
- 7 backend routes under `services/client-api/src/client_api/api/v1/tasks.py`.
- **Zero existing Playwright tests** before this sub-run.

The traceability matrix only covers epic 11, so there's no formal AC inventory
for tasks. I scoped 5 user-journey tests covering the testable surface:
empty/error/happy/permission/detail-modal. Drag-and-drop reordering and
dependency-DAG validation are deferred to a follow-up run because they need
deterministic input timing + cross-task PATCH mocking that's its own design.

---

## Files Created

### `e2e/specs/tasks/tasks-kanban.spec.ts` (NEW)

5 tests, 1 describe per scope:

| # | Scope | Description |
|---|---|---|
| 1 | Empty board | Mock `GET /api/v1/tasks` → `{items: [], total: 0}`; assert `task-board-empty` testid + `tasks.emptyBoardTitle` ("No tasks yet") + the "New task" CTA is enabled (admin role). |
| 2 | Error state | Mock `GET /api/v1/tasks` → 500; assert `task-board-error` testid surfaces. (Retry-button assertion deferred — `BoardErrorState` button text comes from the i18n bundle without a stable testid; brittle to selector drift.) |
| 3 | Happy path | Mock 4 tasks across all 4 statuses (pending / in_progress / blocked / completed); assert all 4 `task-column-{status}` columns render and each `task-card-{id}` lands in the correct column. Plus card-title surface. |
| 4 | Permission gating (read_only) | Manually seed `eusolicit-auth-store` with `role: read_only` (instead of `authenticatedPage` fixture which mints role=member); assert "New task" button is rendered but disabled. Validates the `canMutate` branch in `TaskBoardHeader.tsx:38-49`. |
| 5 | Detail modal opens | Mock 1 task + the `GET /api/v1/tasks/{id}` single-fetch the modal makes; click the card; assert `task-detail-modal` visible with `task-modal-title` populated to the seeded title. |

### Mock helpers

- `mockCompanyProfile(page)` — pass-through for `/api/v1/companies/*` GETs (skips `/members` so they fall through to `mockMembers`).
- `mockMembers(page)` — `/api/v1/companies/{id}/members` returns one admin member so the assignee picker has data.
- `mockTasksList(page, payload)` — handles both the bare `/api/v1/tasks` and the qs-suffixed variant. Accepts a success payload `{items, total}` or a `{status, body}` failure payload for error-state coverage.
- `buildTask(overrides)` — defensive default-shape factory so the spec only specifies fields under test; fills in nullables and timestamps.

### Standard fixtures used

- `authenticatedPage` from `e2e/support/fixtures` (drops `eusolicit-session` cookie + provisions a member-role user via `/api/v1/auth/test-login`).
- Permission-gating test (#4) uses plain `page` and seeds the auth-store directly so it can pin role=read_only — `authenticatedPage` doesn't expose role override.

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for the new file | ✅ 0 errors |
| `npx playwright test --list` | ✅ **5 tests collected** in `e2e/specs/tasks/tasks-kanban.spec.ts` |
| Active vs fixme split | 5 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| Route `/en/workspace/default/tasks` exists | `app/[locale]/(protected)/workspace/[workspaceId]/tasks/page.tsx` | ✅ |
| `data-testid="tasks-board-page"` outer wrapper | `TasksBoardPage.tsx:93` | ✅ |
| `data-testid="task-board"` (when tasks exist) | `TaskBoard.tsx:104` | ✅ |
| `data-testid="task-board-empty"` (when 0 items) | `TasksBoardPage.tsx:111` | ✅ |
| `data-testid="task-board-error"` (on fetch error) | `TasksBoardPage.tsx:107` | ✅ |
| `data-testid="task-column-{status}"` per column | `TaskColumn.tsx:51` | ✅ |
| `data-testid="task-card-{id}"` per card | `TaskCard.tsx:56` | ✅ |
| `data-testid="task-title-{id}"` on card title | `TaskCard.tsx:85` | ✅ |
| `data-testid="task-detail-modal"` | `TaskDetailModal.tsx:241` | ✅ |
| `data-testid="task-modal-title"` (input element) | `TaskDetailModal.tsx:290` | ✅ |
| "No tasks yet" empty copy | `en.json:1324` (`tasks.emptyBoardTitle`) | ✅ |
| "New task" CTA label | `en.json:1322` (`tasks.newTaskBtn`) | ✅ |
| `canCreate=false` renders disabled wrapper | `TaskBoardHeader.tsx:38-49` (admin/bid_manager only per `useAuthStore` selector at `TasksBoardPage.tsx:63-66`) | ✅ |
| GET URL `/api/v1/tasks` (with qs) | `lib/api/tasks.ts:102` (`/api/v1/tasks${qs ? '?'+qs : ''}`) | ✅ |
| GET URL `/api/v1/tasks/{id}` for modal | `lib/api/tasks.ts:112` | ✅ |
| GET URL `/api/v1/companies/{id}/members` | `lib/api/members.ts:14-15` | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up                                                    # frontend (3000) + client-api (8001)
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/tasks/tasks-kanban.spec.ts \
    --project=client-chromium --reporter=list
# Burn-in 3x:
for i in 1 2 3; do
  npx playwright test --config=playwright.config.ts \
      e2e/specs/tasks/tasks-kanban.spec.ts \
      --project=client-chromium --reporter=line || break
done
```

### Risks / open questions for the CI pass

1. **Auth-store role hydration in test #4**: the read_only gating test pins
   `role: "read_only"` in the v1 auth-store entry, then relies on the live
   migration path to copy it to v2. If the migration filters out roles outside
   the standard set or normalises them, the assertion may fail. Workaround: seed
   directly into the v2 key (`eusolicit-client-auth-store-v2`). Saved this as a
   fallback note so a future run can switch on first CI failure.

2. **Page-level redirect for missing default workspace**: tests use the literal
   `default` workspace segment which middleware accepts (`WORKSPACE_ID_RE` at
   `middleware.ts:46`). The tasks page itself reads from a `useWorkspace*` hook
   chain — if the default workspace doesn't resolve, the page may render an
   "access denied" or redirect away. The `authenticatedPage` fixture provisions
   a fresh user via `/api/v1/auth/test-login`, which auto-creates a default
   workspace per the standard registration flow. Should be fine.

3. **Empty/error-state assertions and skeleton timing**: page renders
   `task-board-skeleton` first while the query is in-flight. The mocks resolve
   immediately, so the skeleton flips to empty/error in the same tick. The
   `expect.toBeVisible({ timeout: 15_000 })` should ride out any auth-bootstrap
   work. If CI flakes, raise to 30s or wait explicitly for skeleton dismissal.

4. **Detail-modal i18n and form values**: the modal reads `task-modal-title`
   off an `<input>` (per the screenshot of the tsx I read), so `toHaveValue(...)`
   is the right matcher. If the modal switches to an `<input readOnly>` for
   members without edit permission, the assertion still works since the
   underlying value is the same.

---

## Coverage Delta

- Before P3-a: **0** Playwright tests for Tasks/Kanban.
- After P3-a: **5 active** tests covering empty / error / happy / permission /
  detail-modal scopes.

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P3-b** — Approvals UI (epic 10 stories 10-8/10-9/10-16). Compliance-critical
  workflow. Same testid + mock pattern as Tasks/Kanban.
- **Deferred follow-up** for P3-a:
  - Drag-and-drop reordering (PATCH `/api/v1/tasks/{id}` status change). Needs
    deterministic Playwright drag input + status-update assertion.
  - Dependencies DAG validation surface (story 10-6) — blocked-by counts,
    cycle-prevention error toast.
  - Empty-filtered branch (`emptyFilteredTitle` copy when filters are active
    but match nothing).
- After P3-a runs green in CI, update the rollout-plan checklist line.
