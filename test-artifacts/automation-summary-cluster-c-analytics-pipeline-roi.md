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
scope: cluster-C-analytics-pipeline-roi
batch: P3-c
storyKeys:
  - 12-4-roi-tracker-dashboard-full-stack
  - 12-7-pipeline-forecasting-dashboard-full-stack-professional-tier
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/pipeline/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/pipeline/components/PipelineForecastDashboard.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/pipeline/components/PipelineForecastTimeline.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/pipeline/components/PipelineForecastEmpty.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/pipeline/components/PipelineForecastFilters.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/roi/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/roi/components/RoiTrackerDashboard.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/roi/components/RoiSummaryCards.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/roi/components/RoiBidsTable.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/roi/components/RoiTrendChart.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/competitors/components/TierUpgradeGate.tsx
  - eusolicit-app/frontend/apps/client/lib/api/analytics.ts
  - eusolicit-app/frontend/apps/client/lib/api/error-utils.ts
---

# Automation Summary: P3-c — Analytics: Pipeline + ROI

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P3-c (cluster C, third net-new E2E coverage sub-run)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why this surface

Both dashboards shipped with **only API-level isolation tests** — no user-level
E2E. The headline gap was tier-gate visual coverage: Story 12.7 makes Pipeline
Forecasting Professional-tier-only, but no test asserts the user-facing 403→
upgrade-gate flow.

### Pipeline vs ROI tier-gate behaviour

The two dashboards handle tier gating very differently — worth recording so
future authors don't burn time looking for the wrong surface:

- **Pipeline (`12.7`)** has an explicit `pipeline-upgrade-gate` testid wrapping
  the shared `TierUpgradeGate` component (PipelineForecastDashboard.tsx:84-93).
  The query layer detects 403 via `isHttpError(forecastQuery.error, 403)` and
  short-circuits to the gate.
- **ROI (`12.4`)** has **no UI tier-gate at all**. RoiTrackerDashboard.tsx just
  renders skeletons / empty states based on each query's `isLoading` / `data`
  fields. If a query returns 403, the relevant section displays its loading
  skeleton indefinitely (no graceful fallback). Tier enforcement is purely
  server-side.

Tier-gate visual E2E coverage is therefore **Pipeline-only**. ROI gets happy
+ empty coverage. ROI's lack of a UI-level 403 surface is a separate gap to
flag for the product team — not something I'd patch in an E2E run.

---

## Files Created

### `e2e/specs/analytics/analytics-pipeline-roi.spec.ts` (NEW)

5 tests across 5 describe scopes:

| # | Dashboard | Scope | Description |
|---|---|---|---|
| 1 | Pipeline | Tier gate (P0) | Mock `GET /api/v1/analytics/pipeline/forecast` → 403; assert `pipeline-upgrade-gate` AND inner `competitor-upgrade-gate` testids visible AND CTA `competitor-upgrade-cta` href = `/settings/billing`. |
| 2 | Pipeline | Empty (P1) | 200 with `{items: [], total: 0}`; assert `pipeline-forecast-empty` visible AND `pipeline-filters` still rendered (so users can adjust criteria). |
| 3 | Pipeline | Happy path (P0) | 200 with one seeded forecast item; assert `pipeline-forecast-timeline` + `pipeline-forecast-item` visible plus filter panel + Generate-Report CTA. |
| 4 | ROI | Happy path (P0) | All three queries (`/summary`, `/bids`, `/trends`) return data; assert `roi-tracker-page`, `roi-summary-cards`, all four summary card testids (`roi-card-invested/won/roi-pct/bid-count`), `roi-bids-table`, two `roi-bid-row-{n}` rows, and `roi-trend-chart-section`. |
| 5 | ROI | Empty data (P1) | Bids returns `{items: [], total: 0}`, trends returns `{items: []}`; assert `roi-bids-empty` (RoiBidsTable.tsx:48) and `roi-trend-empty` (RoiTrendChart.tsx:33) surfaces visible AND their non-empty counterparts NOT rendered (`toHaveCount(0)`). |

### Mock helpers

- `mockCompanyProfile(page)` — `/api/v1/companies/*` GET pass-through (skips `/members` so other handlers can claim it).
- `mockPipelineForecast(page, payload)` — `/api/v1/analytics/pipeline/forecast` returns either a `{items, total}` success body or a `{status, body}` failure body (used for the 403 tier-gate test).
- `mockRoiEndpoints(page, opts)` — wires all three ROI GETs (`/summary`, `/bids`, `/trends`) with overridable payloads. Defaults give a populated happy-path scenario; tests can pass `{bids: {items: [], total: 0}, trends: {items: []}}` for the empty branch.

All tests use `authenticatedPage` from `e2e/support/fixtures` — these dashboards rely only on the `isAuthenticated: true` flag and a stable `companyId` (mocked via `/api/v1/companies/*`). No role gating in the UI itself, so the standard fixture suffices.

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for the new file | ✅ 0 errors |
| `npx playwright test --list` | ✅ **5 tests collected** in `e2e/specs/analytics/analytics-pipeline-roi.spec.ts` |
| Active vs fixme split | 5 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| Route `/en/workspace/default/analytics/pipeline` exists | `app/[locale]/(protected)/workspace/[workspaceId]/analytics/pipeline/page.tsx` | ✅ |
| Route `/en/workspace/default/analytics/roi` exists | `app/[locale]/(protected)/workspace/[workspaceId]/analytics/roi/page.tsx` | ✅ |
| `pipeline-upgrade-gate` (outer wrapper for 403 branch) | `PipelineForecastDashboard.tsx:86` | ✅ |
| `competitor-upgrade-gate` (inner shared gate) | `TierUpgradeGate.tsx:26` | ✅ |
| `competitor-upgrade-cta` href = `/settings/billing` | `TierUpgradeGate.tsx:18,53-54` | ✅ |
| `pipeline-forecast-empty` empty branch | `PipelineForecastEmpty.tsx:14` | ✅ |
| `pipeline-forecast-timeline` happy branch | `PipelineForecastTimeline.tsx:120,135` | ✅ |
| `pipeline-forecast-item` per row | `PipelineForecastTimeline.tsx:153` | ✅ |
| `pipeline-filters` filter panel | `PipelineForecastFilters.tsx:30` | ✅ |
| `roi-tracker-page` outer wrapper | `RoiTrackerDashboard.tsx:71` | ✅ |
| 4× ROI summary card testids | `RoiSummaryCards.tsx:39,45,51,57` | ✅ |
| `roi-bids-table` and `roi-bid-row-{n}` | `RoiBidsTable.tsx:87,128` | ✅ |
| `roi-bids-empty` empty branch | `RoiBidsTable.tsx:48` | ✅ |
| `roi-trend-chart-section` happy chart | `RoiTrendChart.tsx:62` | ✅ |
| `roi-trend-empty` empty branch | `RoiTrendChart.tsx:33` | ✅ |
| GET `/api/v1/analytics/pipeline/forecast` | `lib/api/analytics.ts:451-459` | ✅ |
| GET `/api/v1/analytics/roi/{summary,bids,trends}` | `lib/api/analytics.ts:189,200,225` | ✅ |
| 403 detection via `isHttpError(error, 403)` | `lib/api/error-utils.ts:20` | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/analytics/analytics-pipeline-roi.spec.ts \
    --project=client-chromium --reporter=list
```

### Risks / open questions for the CI pass

1. **Pipeline timeline `.first()` selector** — `PipelineForecastTimeline.tsx`
   has TWO elements with `data-testid="pipeline-forecast-timeline"` (lines 120
   and 135 — outer wrapper and inner list). The happy-path test uses
   `.first()` to stay stable. If the React component is refactored to use
   distinct testids (e.g. `pipeline-forecast-timeline-outer` and `…-list`),
   update the selector accordingly.

2. **Mock-handler shadowing** — `mockCompanyProfile(page)` registers
   `**/api/v1/companies/*` first, then explicitly falls through if the URL
   contains `/members`. Approvals/Tasks specs have the same pattern (P3-a/b)
   and it works, but if Playwright's route-matching order changes, the
   `/members` calls might erroneously hit the company handler. Fallback fix:
   register `/api/v1/companies/*/members` route first.

3. **ROI summary "0 values" rendering** — empty-data test (#5) asserts the
   summary cards still render even when bids+trends are empty. The summary
   query is mocked with the default non-empty payload, so cards have real
   numbers. If a test wanted to assert the "all-zero" state, it'd need to
   pass `{summary: {total_invested_eur: 0, total_won_eur: 0, roi_pct: 0,
   bid_count: 0}}` — left for a future sub-run.

4. **Generate-Report CTA happy-path assertion** — Pipeline test #3 asserts
   `generate-report-btn` visible. ROI also has a `generate-report-btn` in
   `RoiTrackerDashboard.tsx:79` — not asserted in test #4 to avoid duplicating
   coverage. Sufficient for the surface check; deeper "click → modal opens"
   is deferred (Generate Report modal lives in `@eusolicit/ui` and would need
   mock wiring for createOnDemandReport / getReportJob).

5. **Tier-gate test (#1) does NOT validate the actual user tier** — the test
   merely mocks a 403 response and asserts the page renders the gate. It
   does NOT exercise the chain: free-tier user → JWT → API checks tier →
   403. That's an end-to-end concern best left to integration tests at the
   API layer (those specs already exist for analytics tier-gate per epic
   12 sprint review). The user-facing E2E confirms that **once a 403 lands**,
   the gate surface renders correctly.

---

## Coverage Delta

- Before P3-c: **0** Playwright tests for either dashboard (only API isolation
  / tier-gate-enforcement specs at `e2e/specs/analytics/cross-tenant-isolation.api.spec.ts`
  and `tier-gate-enforcement.api.spec.ts`).
- After P3-c: **5 active** user-level E2E tests covering tier-gate visual,
  empty, and happy-path scopes for both dashboards.

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P3-d** — Analytics: Market + Team + Competitors + Usage (4 remaining
  dashboards in one sub-run; share a tier-gated fixture pattern from this
  spec).
- **Deferred follow-ups** for P3-c:
  - ROI dashboard's missing UI tier-gate is a product gap (not a test gap) —
    file separately in product issue tracker.
  - Generate-Report modal "click → submit" flow (separate sub-run, modal
    lives cross-feature in `@eusolicit/ui`).
  - Filter-and-pagination interaction tests (apply filter → API receives
    qs, table re-renders with new page).
- After P3-c runs green in CI, update the rollout-plan checklist line.
