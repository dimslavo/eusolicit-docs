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
scope: cluster-C-analytics-rest
batch: P3-d
storyKeys:
  - 12-2-market-intelligence-dashboard-api
  - 12-3-market-intelligence-dashboard-frontend
  - 12-5-team-performance-dashboard-full-stack
  - 12-6-competitor-intelligence-dashboard-full-stack-professional-tier
  - 12-8-usage-dashboard-full-stack
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/market/components/MarketIntelligenceDashboard.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/market/components/AuthoritiesTable.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/team/components/TeamPerformanceDashboard.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/team/components/TeamLeaderboardTable.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/team/components/TeamActivityChart.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/competitors/components/CompetitorIntelligenceDashboard.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/competitors/components/CompetitorProfileCards.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/competitors/components/TierUpgradeGate.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/usage/components/UsageDashboard.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/usage/components/UsageMetersSection.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/analytics/usage/components/UsageMeter.tsx
  - eusolicit-app/frontend/apps/client/lib/api/analytics.ts
---

# Automation Summary: P3-d — Analytics: Market + Team + Competitors + Usage

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P3-d (cluster C, fourth net-new E2E coverage sub-run)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Triage

Four dashboards in one sub-run. The shared mock pattern from P3-c
(`jsonResponder` + per-dashboard wiring) carried over cleanly so the spec stays
under 500 lines while covering both happy + empty paths for each board, plus
the Competitors tier-gate (the only one of the four with a UI 403 surface).

### Tier-gate behaviour matrix

| Dashboard | UI tier-gate? | Detection |
|---|---|---|
| Market | ❌ No | Server-side only; 403 → skeletons stuck |
| Team | ❌ No | Server-side only |
| Competitors | ✅ **Yes** | `CompetitorIntelligenceDashboard.tsx:70` calls `isHttpError(profilesQuery.error, 403)` → renders `<TierUpgradeGate />` |
| Usage | ❌ No | Paid-tier matches nav guard upstream; 403 not expected |

So tier-gate visual coverage in P3-d is **Competitors-only**. (Pipeline
covered separately in P3-c.) ROI's tier-gate gap was already flagged in P3-c.

---

## Files Created

### `e2e/specs/analytics/analytics-rest.spec.ts` (NEW)

8 tests across 8 describe scopes (one happy + one variant per dashboard):

| # | Dashboard | Scope | Description |
|---|---|---|---|
| 1 | Market | Happy path (P0) | Mock all 4 GETs (`/volume`, `/values`, `/authorities`, `/trends`) → assert `market-intelligence-page`, `market-volume-chart-section`, `market-trend-chart-section`, `market-authorities-section`, `market-authorities-table`, `market-auth-row-0`. |
| 2 | Market | Empty authorities (P1) | `/authorities` returns `{items: [], total: 0}` → assert `market-authorities-empty` visible AND `market-authorities-table` absent. |
| 3 | Team | Happy path (P0) | Leaderboard + activity + user GETs all return data → assert `team-filters`, `team-leaderboard-table`, `team-activity-chart-section`. |
| 4 | Team | Empty leaderboard (P1) | Leaderboard returns empty → `team-leaderboard-empty` visible, table absent. |
| 5 | Competitors | Tier gate (P0) | `/competitors/profiles` returns 403 → `competitor-upgrade-gate` visible, CTA href = `/settings/billing`, `competitor-profile-cards` and `competitor-filters` both absent (component branch at CompetitorIntelligenceDashboard.tsx:114-116). |
| 6 | Competitors | Happy path (P0) | Profiles + benchmarks return data → `competitor-filters`, `competitor-profile-cards`, `competitor-card-acme-holding`, `competitor-card-beta-group`. Slug derivation per `CompetitorProfileCards.tsx:79`. Asserts upgrade-gate **not** visible. |
| 7 | Usage | Happy path (P0) | `/usage` returns 3 meters with consumption under limits → `usage-billing-period`, `usage-meters`, all 3 `usage-meter-{type}` testids; upgrade CTA NOT shown (no metric at limit). |
| 8 | Usage | Empty (P1) | `/usage` returns `{billing_period_start: null, ..., usage: []}` → `usage-empty` visible, `usage-meters` absent. |

### Mock helpers

- `mockCompanyProfile(page)` — same pattern as P3-a/b/c.
- `jsonResponder(payload)` — small reusable factory that returns either a 200
  with JSON body or a `{status, body?}` failure. Covers all eight tests.
- Per-dashboard `mockMarketEndpoints` / `mockTeamEndpoints` /
  `mockCompetitorEndpoints` / `mockUsageEndpoint` accept overrides for the
  endpoints they care about and use the `jsonResponder` helper for handlers.

### Rationale for "8 tests, 1 spec file"

Splitting into 4 files (one per dashboard) would have duplicated the
`mockCompanyProfile` + `jsonResponder` infrastructure. A single spec keeps
the surface area for future maintainers small without coupling tests
together — each `test.describe(...)` is independent, and Playwright runs
each test in its own browser context.

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for the new file | ✅ 0 errors |
| `npx playwright test --list` | ✅ **8 tests collected** in `e2e/specs/analytics/analytics-rest.spec.ts` |
| Active vs fixme split | 8 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| Routes exist for all 4 dashboards | `app/[locale]/(protected)/workspace/[workspaceId]/analytics/{market,team,competitors,usage}/page.tsx` | ✅ |
| **MARKET** | | |
| `market-intelligence-page` outer wrapper | `MarketIntelligenceDashboard.tsx:76` | ✅ |
| `market-volume-chart-section` / `market-trend-chart-section` / `market-authorities-section` | lines 118, 133, 149 | ✅ |
| `market-authorities-table` / `market-auth-row-{n}` | `AuthoritiesTable.tsx:86, 129` | ✅ |
| `market-authorities-empty` empty branch | `AuthoritiesTable.tsx:77` | ✅ |
| GETs `/api/v1/analytics/market/{volume,values,authorities,trends}` | `lib/api/analytics.ts:95, 109, 125, 139` | ✅ |
| **TEAM** | | |
| `team-filters` / `team-leaderboard-table` / `team-activity-chart-section` | `TeamFilters.tsx:34`, `TeamLeaderboardTable.tsx:64`, `TeamActivityChart.tsx:46` | ✅ |
| `team-leaderboard-empty` empty branch | `TeamLeaderboardTable.tsx:50` | ✅ |
| GETs `/api/v1/analytics/team/{leaderboard,user/{id},activity}` | `lib/api/analytics.ts:284, 306, 314` | ✅ |
| **COMPETITORS** | | |
| `competitor-upgrade-gate` for 403 branch | `TierUpgradeGate.tsx:26` (component re-rendered by `CompetitorIntelligenceDashboard.tsx:115`) | ✅ |
| `competitor-upgrade-cta` href | `TierUpgradeGate.tsx:53-54` (href = `/settings/billing`) | ✅ |
| `competitor-profile-cards` / `competitor-card-{slug}` | `CompetitorProfileCards.tsx:40, 79` | ✅ |
| `competitor-filters` | `CompetitorFilters.tsx:31` | ✅ |
| Slug derivation `acme-holding` / `beta-group` | per `CompetitorProfileCards.tsx:8` (lowercase + hyphenate) | ✅ |
| GETs `/api/v1/analytics/competitors/{profiles,patterns,benchmarks}` | `lib/api/analytics.ts:381, 396, 407` | ✅ |
| **USAGE** | | |
| `usage-billing-period` / `usage-meters` / `usage-meter-{type}` (3 types) | `UsageBillingPeriod.tsx:43`, `UsageMetersSection.tsx:49`, `UsageMeter.tsx` doc lines 16-18 | ✅ |
| `usage-empty` empty branch | `UsageDashboard.tsx:92` | ✅ |
| `usage-upgrade-cta` not shown when usage < limit | `UsageDashboard.tsx:25-27` (`anyAtLimit` flag) | ✅ |
| GET `/api/v1/analytics/usage/` | `lib/api/analytics.ts:487` | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/analytics/analytics-rest.spec.ts \
    --project=client-chromium --reporter=list
```

### Risks / open questions for the CI pass

1. **Usage URL routing** — `/api/v1/analytics/usage/` (with trailing slash, per
   `lib/api/analytics.ts:487`). The mock registers two routes — `/usage/**`
   AND `/usage**` — so Playwright matches whichever the apiClient sends.
   Slight overhead but deterministic.

2. **Competitor card slugs** — `acme-holding` and `beta-group` are derived from
   the seeded competitor names. If the `slugify` helper at
   `CompetitorProfileCards.tsx:8` ever changes (e.g. emits unicode-NFKD'd
   variants), update the seed names or the testid expectation. Slug behavior
   is well-tested in unit tests for that component.

3. **Team activity chart with empty data** — test #4 asserts
   `team-leaderboard-empty` but doesn't separately probe the activity chart's
   empty surface. `TeamActivityChart.tsx:32` has its own `team-activity-empty`
   testid worth covering in a follow-up if activity-only-empty becomes a
   distinct user-visible state.

4. **Pattern endpoint defensive mock** — Competitors happy-path test mocks
   `/competitors/*/patterns` to return `{items: []}` so the page doesn't
   404 in the background if the user happened to click a card. The visible
   click-to-select behaviour is not asserted here (would need a follow-up
   sub-run that mocks a real pattern dataset and checks `competitor-pattern-chart`).

5. **Filters interaction tests deferred** — applying date-from / sector
   filters → asserting query string sent to the API → table re-renders
   with new data. Same pattern across all four dashboards. Worth a
   dedicated follow-up sub-run because all four share the staged-vs-applied
   filter pattern (handleApply / handleClear) — could share a helper.

6. **Generate-Report modal flows** — every dashboard surfaces a
   `generate-report-btn`. Click → modal opens → fill type → submit. Cross-feature
   E2E (modal lives in `@eusolicit/ui`) deferred per P3-c notes.

---

## Coverage Delta

- Before P3-d: **0** Playwright tests for these 4 dashboards (only API isolation
  + tier-gate-enforcement specs).
- After P3-d: **8 active** user-level E2E tests (2 per dashboard).

---

## Cluster C running totals (through P3-d)

| Sub-run | Spec file | Active | Fixme |
|---|---|---:|---:|
| P3-a | `tasks-kanban.spec.ts` | 5 | 0 |
| P3-b | `approvals.spec.ts` | 5 | 0 |
| P3-c | `analytics-pipeline-roi.spec.ts` | 5 | 0 |
| P3-d | `analytics-rest.spec.ts` | 8 | 0 |
| **Total** | **4 spec files** | **23** | **0** |

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P3-e** — Reports (epic 12 stories 12-9, 12-10): PDF/DOCX scheduled reports
  + on-demand report generation. Cross-cuts proposals.
- **Deferred follow-ups** for P3-d:
  - Filters interaction tests (apply → URL qs → re-render).
  - Generate Report modal click-and-submit flow (cross-feature, deferred since P3-c).
  - Competitor pattern chart click-to-select interaction.
  - Usage upgrade CTA when a metric hits limit.
  - Team activity-chart standalone empty state (`team-activity-empty`).
- After P3-d runs green in CI, update the rollout-plan checklist line.
