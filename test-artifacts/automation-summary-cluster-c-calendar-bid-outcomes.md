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
scope: cluster-C-calendar-bid-outcomes
batch: P3-f
storyKeys:
  - 9-13-calendar-connection-management-frontend-page
  - 9-7-ical-feed
  - 9-8-google-calendar-oauth
  - 9-9-outlook-calendar-oauth
  - 14-x-bid-outcome-capture-and-lessons-learned
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/settings/calendar/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/opportunities/[id]/outcome/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/opportunities/[id]/outcome/components/BidOutcomePage.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/opportunities/[id]/outcome/components/BidOutcomeForm.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/opportunities/[id]/outcome/components/BidOutcomeLessonsPanel.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/opportunities/[id]/outcome/components/BidOutcomeEmptyState.tsx
  - eusolicit-app/frontend/apps/client/lib/api/bid-outcomes.ts
  - eusolicit-app/frontend/apps/client/lib/queries/use-bid-outcomes.ts
---

# Automation Summary: P3-f — Calendar settings + Bid outcomes (cluster C, sub-run 6)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P3-f (cluster C, sixth net-new E2E coverage sub-run)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why these surfaces

Two distinct settings/per-record surfaces with **zero** prior Playwright
coverage; both are user-visible state machines that branch on backend state:

- **Calendar connections** (`/settings/calendar`) — three independent
  integration sections (iCal, Google OAuth, Outlook OAuth) that flip between
  *connect/generate* and *disconnect/regenerate* CTAs based on
  `GET /api/v1/calendar/connections`. Wrong state → users get duplicate tokens
  or confused about whether sync is running.
- **Bid outcome capture** (`/opportunities/{id}/outcome`) — a 3-state page
  (no proposals → empty CTA, proposals + no outcome → capture form, outcome
  recorded → read-only summary + lessons-learned panel). The lessons panel
  itself has 3 lifecycle states (`pending`, `failed`, `completed`) plus a
  `not_applicable` short-circuit.

---

## Files Created

### `e2e/specs/calendar/calendar-settings.spec.ts` (NEW)

3 tests across 3 describe scopes:

| # | State | Scope | Description |
|---|---|---|---|
| 1 | All disconnected | P0 | `connections` payload returns `ical_token: null` + both OAuth providers `connected: false`. Asserts page wrapper + 3 sections visible + 3 CTAs (`calendar-ical-generate-btn`, `calendar-google-connect-btn`, `calendar-microsoft-connect-btn`) + `toHaveCount(0)` for url-input and disconnect buttons. |
| 2 | iCal token provisioned | P1 | `ical_token: 'tok_…'`, OAuth still disconnected. Asserts URL input + copy button + instructions card + regenerate CTA all visible; the disconnected-state generate CTA must NOT render (`toHaveCount(0)`). |
| 3 | Both OAuth connected | P1 | Google + Microsoft `connected: true` with sync metadata (created/updated/deleted counters, last_synced_at, connected_at, no `sync_error`). Asserts `connected-at`, `last-synced`, `events-created`, `disconnect-btn` for each provider; connect CTAs and `sync-error` testids absent. |

### `e2e/specs/bid-outcomes/bid-outcomes.spec.ts` (NEW)

3 tests across 3 describe scopes:

| # | State | Scope | Description |
|---|---|---|---|
| 1 | No proposals linked | P1 | `/proposals` returns `items: []`, `/outcome` returns 404. Asserts `bid-outcome-page` wrapper + `bid-outcome-empty-state-no-proposals` empty CTA visible; both `bid-outcome-form` and `bid-outcome-lessons-panel` absent. |
| 2 | Capture form visible | P0 | One linked proposal, `/outcome` returns 404. Asserts `bid-outcome-form` + 3 outcome radios (`won`/`lost`/`withdrawn`) + feedback textarea + submit CTA visible; empty-state and lessons panel absent. Validates the 404-on-outcome-alone does NOT trigger the error banner (per `BidOutcomePage.tsx:55`). |
| 3 | Recorded outcome + completed lessons | P1 | `/outcome` returns 200 with `lessons_learned_status: 'completed'` + populated `lessons_learned` payload. Asserts `bid-outcome-lessons-panel` + 5 collapsible section cards (`strengths`, `improvementAreas`, `keyTakeaways`, `recommendedActions`, `summary`); form, empty-state, and pending/failed status badges all absent. |

### Mock helpers

**Calendar spec:**
- `mockCompanyProfile(page)` — same pattern as P3-{a..e}.
- `mockConnections(page, payload)` — typed `CalendarConnections` interface;
  registers `GET /api/v1/calendar/connections` with full Google + Microsoft
  fixture shape (`connected`, `connected_at`, `last_synced_at`, sync counters,
  `sync_error`).

**Bid outcome spec:**
- `mockCompanyProfile(page)` — same pattern.
- `mockOpportunityDetail(page, payload)` — `GET /api/v1/opportunities/{id}`,
  permissive shape (page only reads `name`).
- `mockOpportunityProposals(page, opportunityId, items)` —
  `GET /api/v1/opportunities/{id}/proposals` returning `{items}`.
- `mockOpportunityOutcome(page, opportunityId, {status, body})` —
  `GET /api/v1/opportunities/{id}/outcome`, parameterised so the same helper
  serves the 404 (no outcome yet) and 200 (recorded) branches.

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for new files | ✅ 0 errors |
| `npx playwright test --list` filtered to new files | ✅ **6 tests collected** in 2 spec files |
| Active vs fixme split | 6 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Production-state cross-check

**Calendar:**

| Test asserts | Live source | Match? |
|---|---|---|
| Route `/en/workspace/default/settings/calendar` exists | `app/.../settings/calendar/page.tsx` | ✅ |
| `calendar-connections-page` outer wrapper + page-title | settings/calendar page component | ✅ |
| `calendar-section-{ical,google,microsoft}` sub-sections | settings/calendar component tree | ✅ |
| `calendar-ical-{generate,regenerate,url-input,copy,instructions}-btn` testids | iCal section component | ✅ |
| `calendar-{google,microsoft}-{connect,disconnect}-btn` testids | OAuth provider components | ✅ |
| `calendar-{google,microsoft}-{connected-at,last-synced,events-created}` badges | OAuth provider components | ✅ |
| `GET /api/v1/calendar/connections` (state) | calendar settings query hook | ✅ |

**Bid outcomes:**

| Test asserts | Live source | Match? |
|---|---|---|
| Route `/en/workspace/default/opportunities/{id}/outcome` exists | `app/.../opportunities/[id]/outcome/page.tsx` | ✅ |
| `bid-outcome-page` outer wrapper | `BidOutcomePage.tsx:78` | ✅ |
| `bid-outcome-empty-state-no-proposals` empty CTA | `BidOutcomeEmptyState.tsx:18` | ✅ |
| `bid-outcome-form` + radios `bid-outcome-radio-{won,lost,withdrawn}` | `BidOutcomeForm.tsx:174, 233, 239, 245` | ✅ |
| `bid-outcome-feedback` textarea + `bid-outcome-submit` CTA | `BidOutcomeForm.tsx:304, 321` | ✅ |
| `bid-outcome-lessons-panel` + 5 section cards | `BidOutcomeLessonsPanel.tsx:116, 132, 141, 150, 159, 169` | ✅ |
| `bid-outcome-lessons-status-{pending,failed}` testids exist | `BidOutcomeLessonsPanel.tsx:83, 104` | ✅ |
| Error gate logic — 404 on /outcome alone does NOT trigger error banner | `BidOutcomePage.tsx:55` (`isErrorOutcome && Boolean(outcome)`) | ✅ |
| GET `/api/v1/opportunities/{id}` (detail) | `lib/api/opportunities.ts:170` | ✅ |
| GET `/api/v1/opportunities/{id}/proposals` | `lib/api/opportunities.ts:192` | ✅ |
| GET `/api/v1/opportunities/{id}/outcome` (404 = unrecorded, 200 = recorded) | `lib/api/bid-outcomes.ts:82` | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/calendar/calendar-settings.spec.ts \
    e2e/specs/bid-outcomes/bid-outcomes.spec.ts \
    --project=client-chromium --reporter=list
```

### Risks / open questions for the CI pass

1. **Calendar i18n drift**: Three section testids (`calendar-section-{ical,
   google,microsoft}`) and the CTAs are asserted by testid only — not by
   visible label — so locale changes won't break the spec. Same for sync
   metadata badges.

2. **Bid outcome — `lessons_learned_status` enum** (line 6 of `bid-outcomes.ts`):
   the API type is `"pending" | "completed" | "failed" | "not_applicable"`. Test
   #3 uses `completed` (not `complete`); a typo here would slot into the
   "no panel renders" else-branch. Fixture is correct.

3. **Bid outcome — proposal selector pre-select**: with one proposal in the
   list, `BidOutcomeForm.tsx:84` pre-selects it and `:171` disables the
   selector. Test #2 doesn't drive a submit so this is benign — but a future
   "submit happy path" test must remember the disabled state and assert via
   the captured POST payload rather than driving the Radix select.

4. **404 mock body shape**: the BidOutcome query hook checks `axios.isAxiosError
   && err.response?.status === 404` to skip retries; any 404 status with any
   body satisfies that branch. Spec uses `{detail: 'Outcome not recorded'}`
   to mirror FastAPI's typical shape.

5. **`bid-outcome-lessons-panel` testid only renders for the catch-all branch**
   (`BidOutcomeLessonsPanel.tsx:116`) — `pending` and `failed` use their own
   status testids and `not_applicable` returns null. Test #3 fixture fits the
   panel-render branch by setting `lessons_learned_status: 'completed'` AND
   providing a non-null `lessons_learned`.

6. **Deferred follow-ups for P3-f**:
   - Calendar — iCal generate POST flow (`POST /calendar/ical/generate-token`),
     OAuth disconnect DELETE captures, sync error fixture rendering.
   - Bid outcome — submit-form happy path (POST capture + toast),
     `lessons_learned_status: 'pending'` polling state, server-error 409/422
     inline display, evaluator-score-table add/remove rows.

---

## Coverage Delta

- Before P3-f: **0** Playwright tests for either surface.
- After P3-f: **6 active** user-level E2E tests covering disconnected /
  iCal-only / both-OAuth / no-proposals / form-visible / recorded-outcome
  scopes.

---

## Cluster C running totals (through P3-f)

| Sub-run | Spec file(s) | Active | Fixme |
|---|---|---:|---:|
| P3-a | `tasks-kanban.spec.ts` | 5 | 0 |
| P3-b | `approvals.spec.ts` | 5 | 0 |
| P3-c | `analytics-pipeline-roi.spec.ts` | 5 | 0 |
| P3-d | `analytics-rest.spec.ts` | 8 | 0 |
| P3-e | `reports/reports.spec.ts` | 5 | 0 |
| P3-f | `calendar/calendar-settings.spec.ts` + `bid-outcomes/bid-outcomes.spec.ts` | 6 | 0 |
| **Total** | **7 spec files** | **34** | **0** |

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P3-g** — Comments sidebar + Content blocks (proposal-editor surfaces).
- **P3-h** — Admin UI (separate base URL + auth — own run).
- **P4-{a..c}** — API gap fill (NPS / billing / cross-service).

After P3-f runs green in CI, update the rollout-plan checklist line.
