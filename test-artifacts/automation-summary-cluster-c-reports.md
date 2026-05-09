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
scope: cluster-C-reports
batch: P3-e
storyKeys:
  - 12-9-report-generation-engine-pdf-docx
  - 12-10-scheduled-on-demand-report-delivery
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/reports/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/reports/components/ReportsList.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/settings/reports/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/settings/reports/components/ReportScheduleSettings.tsx
  - eusolicit-app/frontend/apps/client/lib/api/reports.ts
---

# Automation Summary: P3-e — Reports (cluster C, sub-run 5)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P3-e (cluster C, fifth net-new E2E coverage sub-run)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why this surface

Stories 12.9 (PDF/DOCX generation engine) + 12.10 (scheduled & on-demand
delivery) ship two distinct user surfaces:

- **`/reports`** — on-demand report job list with status polling (3s interval
  while any job is pending/processing) and per-row download links.
- **`/settings/reports`** — admin-only schedule management page (create, list,
  toggle, delete).

Before this sub-run: 0 Playwright specs for either surface. Both are
high-stakes — billing artifacts (PDFs delivered to recipients) and admin-only
configuration that affects every user in a workspace.

---

## Files Created

### `e2e/specs/reports/reports.spec.ts` (NEW)

5 tests across 5 describe scopes:

| # | Surface | Scope | Description |
|---|---|---|---|
| 1 | /reports | Empty (P1) | `GET /api/v1/reports/` returns 0 items → `reports-empty` visible, `reports-table` absent. |
| 2 | /reports | Happy path (P0) | Two seeded jobs (one complete with download URL, one processing). Assert both `report-row-{id}` testids visible, both `report-status-{id}` badges visible, download link visible for complete job and absent for processing job. |
| 3 | /settings/reports | Non-admin forbidden (P1) | Seed `role: member` auth-store. Page short-circuits at `ReportScheduleSettings.tsx:101-102` → `reports-schedule-forbidden` visible, neither the form nor the page wrapper render. Defensive 403 mock on `/schedules` covers the case where the gate is bypassed. |
| 4 | /settings/reports | Admin happy path (P0) | Seed `role: admin` auth-store, return 1 existing schedule. Assert `reports-schedule-page`, `schedule-row-{id}`, toggle + delete controls per row, plus the `report-schedule-form` create panel + `schedule-submit`. |
| 5 | /settings/reports | Admin creates schedule (P0) | Empty schedules list, fill the form (type=team_activity via Radix Select, format=pdf, frequency=monthly, single recipient email), click submit, assert POST captured with the exact payload shape (`{report_type, format, frequency, recipients}`). |

### Mock helpers

- `mockCompanyProfile(page)` — same pattern as P3-a/b/c/d.
- `seedAuthStore(page, { role })` — pins `user.role` ('admin' | 'member' | 'read_only')
  via the v1 `eusolicit-auth-store` localStorage key. Required because the schedule
  page reads `useAuthStore((s) => s.user)` directly and renders the forbidden
  branch when `role !== 'admin'`. The standard `authenticatedPage` fixture only
  mints `role=member`.
- `mockReportsList(page, payload)` — registers GET handlers for `/api/v1/reports/`,
  `/api/v1/reports/?…`, and the bare `/api/v1/reports**` glob, with explicit
  fall-through for POSTs and for sub-paths like `/reports/{jobId}` and
  `/reports/schedules` so the schedule mocks can claim them.
- `mockSchedulesList(page, payload)` — `/api/v1/reports/schedules` GET handler
  for the schedule-list pages.

### Why /reports has multiple matching globs

`apiClient` (axios) in this repo sends GET to `/api/v1/reports/` (trailing
slash, per `lib/api/reports.ts:96`). Playwright's `page.route` does not
auto-handle trailing-slash variants. The spec registers three matching globs
and lets the most-specific handler win — matches both shapes plus the
filtered/paginated form (`?…`).

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for the new file | ✅ 0 errors |
| `npx playwright test --list` | ✅ **5 tests collected** in `e2e/specs/reports/reports.spec.ts` |
| Active vs fixme split | 5 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| Route `/en/workspace/default/reports` exists | `app/[locale]/(protected)/workspace/[workspaceId]/reports/page.tsx` | ✅ |
| Route `/en/workspace/default/settings/reports` exists | `app/[locale]/(protected)/workspace/[workspaceId]/settings/reports/page.tsx` | ✅ |
| `reports-list-page` outer wrapper | `ReportsList.tsx:68` | ✅ |
| `reports-table` and `report-row-{id}` | `ReportsList.tsx:95, 111` | ✅ |
| `report-status-{id}` per row | `ReportsList.tsx:120` | ✅ |
| `report-download-{id}` for complete jobs only | `ReportsList.tsx:147` (conditional on `status === 'complete'` and non-expired URL) | ✅ |
| `reports-empty` empty branch | `ReportsList.tsx:88` (passed to EmptyState) | ✅ |
| `reports-schedule-page` admin wrapper | `ReportScheduleSettings.tsx:193` | ✅ |
| `reports-schedule-forbidden` non-admin gate | `ReportScheduleSettings.tsx:169` | ✅ |
| `report-schedule-form` create form | `ReportScheduleSettings.tsx:271` | ✅ |
| `schedule-{type-select,format-pdf,format-docx,frequency-weekly,frequency-monthly,recipients-input,submit}` testids | `ReportScheduleSettings.tsx:282, 306, 314, 333, 343, 356, 383` | ✅ |
| `schedule-{row,toggle,delete}-{id}` per existing schedule | `ReportScheduleSettings.tsx:212, 228, 255` | ✅ |
| `user.role === 'admin'` page-level gate | `ReportScheduleSettings.tsx:101` | ✅ |
| GET `/api/v1/reports/` (list with trailing slash) | `lib/api/reports.ts:96` | ✅ |
| GET `/api/v1/reports/schedules` (admin list) | `lib/api/reports.ts:124` | ✅ |
| POST `/api/v1/reports/schedules` (create) | `lib/api/reports.ts:109` | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/reports/reports.spec.ts \
    --project=client-chromium --reporter=list
```

### Risks / open questions for the CI pass

1. **Radix Select interaction (test #5)**: clicking `schedule-type-select` opens
   a portaled list. The test then targets `getByRole('option', { name: /team
   activity/i })`. If the option label text differs from the i18n string in
   `messages/en.json` (`reports.types.team_activity` = "Team activity"), the
   selector misses. Easy fix — replace with a more specific testid if the
   component is updated to add one.

2. **Recipient form-level validation**: the schema (`lib/schemas/reports.ts`)
   validates emails per-item plus min=1 / max=20. Test #5 fills a single valid
   email so the form should accept and submit. If the textarea-to-array
   parsing helper (`parseRecipients`) ever requires a trailing newline or
   stricter format, adjust the input.

3. **Polling race in /reports happy-path test**: ReportsList uses
   `refetchInterval` of 3s while any job is pending/processing. The test mocks
   the list endpoint to return a populated list including a `processing` job;
   the mock will be re-hit every 3s and respond identically. No state mutation
   needed for the test to pass — assertions just check the initial render.

4. **Auth-store role hydration timing**: same as P3-a / P3-b / P3-d. The v1→v2
   migration runs at module load via `addInitScript`-seeded localStorage. If
   that ever flakes, switch to seeding the v2 key directly.

5. **Forbidden state DOM count assertions**: test #3 uses `toHaveCount(0)` on
   the page-wrapper / form testids to confirm the admin branch did NOT render.
   That's a stricter assertion than `not.toBeVisible()` (which can pass even
   if an element is offscreen). If the page renders BOTH branches (e.g. wraps
   the forbidden state inside a layout that also has the wrapper), the assertion
   tightens to find any drift.

6. **Deferred follow-ups for P3-e**:
   - Polling transition test: a `processing` job advances to `complete` after
     a few polls, and the download link appears. Needs sequential mock state.
   - Schedule toggle + delete interactions (PUT and DELETE captures).
   - Generate Report modal click → submit flow (cross-feature, deferred since P3-c).
   - Failed job rendering — `status: 'failed'` badge surface.

---

## Coverage Delta

- Before P3-e: **0** Playwright tests for either Reports surface.
- After P3-e: **5 active** user-level E2E tests covering empty / happy /
  forbidden / admin happy / admin create scopes.

---

## Cluster C running totals (through P3-e)

| Sub-run | Spec file | Active | Fixme |
|---|---|---:|---:|
| P3-a | `tasks-kanban.spec.ts` | 5 | 0 |
| P3-b | `approvals.spec.ts` | 5 | 0 |
| P3-c | `analytics-pipeline-roi.spec.ts` | 5 | 0 |
| P3-d | `analytics-rest.spec.ts` | 8 | 0 |
| P3-e | `reports/reports.spec.ts` | 5 | 0 |
| **Total** | **5 spec files** | **28** | **0** |

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P3-f** — Calendar settings + Bid outcomes (settings surfaces, share fixture).
- **P3-g** — Comments sidebar + Content blocks (proposal-editor surfaces).
- **P3-h** — Admin UI (separate base URL + auth — own run).
- **P4-{a..c}** — API gap fill (NPS / billing / cross-service).

After P3-e runs green in CI, update the rollout-plan checklist line.
