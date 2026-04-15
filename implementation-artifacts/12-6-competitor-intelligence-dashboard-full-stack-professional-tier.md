# Story 12.6: Competitor Intelligence Dashboard (Full Stack, Professional+ Tier)

Status: done

## Story

As a **Professional or Enterprise tier user on the EU Solicit platform**,
I want **a competitor intelligence dashboard at `/analytics/competitors`**,
so that **I can explore competitor bidding profiles, compare 2–4 competitors side by side, visualise per-sector bidding patterns, and review pricing benchmarks — all powered by the `mv_competitor_intelligence` materialized view, gated behind a Professional+ subscription**.

## Acceptance Criteria

### Backend — Competitor Analytics Endpoints

1. **AC1** — `GET /api/v1/analytics/competitors/profiles` returns a paginated list of competitor profiles for the authenticated user's company. Response body is `{"items": [...], "total": N, "page": P, "page_size": S}`. Each item (`CompetitorProfileItem`) contains:
   - `competitor_name` (string) — natural key from `mv_competitor_intelligence.competitor_name`
   - `total_bids` (integer) — `SUM(total_bid_count)` grouped by `competitor_name`
   - `avg_win_rate` (Decimal string) — `AVG(avg_win_rate)` across all sectors for that competitor; `"0"` if no data
   - `sectors` (list[str]) — distinct `sector` values for that competitor, sorted alphabetically
   - `avg_bid_value_eur` (Decimal string or null) — `AVG(avg_bid_value_eur)` across sectors

   Supports optional query parameters:
   - `sector` (string, optional) — filter to competitors active in this sector
   - `sort_by` (`Literal["total_bids", "avg_win_rate", "competitor_name"]`; default: `"total_bids"`)
   - `sort_dir` (`Literal["asc", "desc"]`; default: `"desc"`)
   - `page` (int ≥ 1; default: 1), `page_size` (int 1–100; default: 20)

2. **AC2** — `GET /api/v1/analytics/competitors/{competitor_name}/patterns` returns the per-sector breakdown for a single competitor. Path parameter `competitor_name` is URL-decoded by FastAPI. Response body is `{"items": [...]}` — a `ListResponse[CompetitorPatternItem]` where each item contains:
   - `sector` (string)
   - `bid_count` (integer) — `total_bid_count` for this competitor in this sector
   - `win_rate` (Decimal string or null) — `avg_win_rate` for this competitor in this sector
   - `avg_bid_value_eur` (Decimal string or null)

   Returns **HTTP 404** (`{"error": "not_found", "message": "Competitor not found"}`) when no rows exist for the given `competitor_name` and `company_id`. Results ordered by `bid_count DESC` (the bar chart renders highest-activity sectors first).

3. **AC3** — `GET /api/v1/analytics/competitors/benchmarks` returns pricing benchmark aggregations across all competitors in the company's view, grouped by sector. Response body is `{"items": [...]}` — a `ListResponse[CompetitorBenchmarkItem]` where each item contains:
   - `sector` (string)
   - `competitor_count` (integer) — distinct number of competitors in this sector
   - `avg_bid_value_eur` (Decimal string or null) — `AVG(avg_bid_value_eur)` across all competitors in sector
   - `max_bid_value_eur` (Decimal string or null) — `MAX(avg_bid_value_eur)` per sector
   - `min_bid_value_eur` (Decimal string or null) — `MIN(avg_bid_value_eur)` per sector

   Results ordered by `competitor_count DESC`.

4. **AC4 — Professional+ Tier Gate (implement `require_professional_plus_tier`)**:
   - The existing stub in `client_api/core/tier_gate.py` MUST be replaced with a real implementation.
   - `PROFESSIONAL_PLUS_PLANS: frozenset[str] = frozenset({"professional", "enterprise"})` — add to `tier_gate.py`.
   - The dependency queries `client.subscriptions` for `company_id = current_user.company_id AND plan IN ('professional', 'enterprise') AND status = 'active'`.
   - If no qualifying row is found, raise `ForbiddenError("Competitor intelligence requires a Professional or Enterprise subscription.", details={"upgrade_required": True})` → HTTP 403.
   - All 3 competitor endpoints (`/profiles`, `/{competitor_name}/patterns`, `/benchmarks`) use `require_professional_plus_tier` as their auth dependency.

5. **AC5 — Tenant Isolation** (E12-R-001): Every SQL statement in the competitor service MUST include `WHERE company_id = current_user.company_id` as the **first, non-optional clause** before any optional filters.

6. **AC6 — Cache Headers**: All 3 endpoints return `Cache-Control: public, max-age=1800` (daily MV refresh; 30-minute client-side staleness is acceptable).

7. **AC7 — Routing Order**: In `api/v1/analytics_competitors.py`, the `/profiles` and `/benchmarks` routes MUST be declared **before** the `/{competitor_name}/patterns` route to prevent FastAPI matching `"profiles"` or `"benchmarks"` as a `competitor_name` path parameter.

8. **AC8 — Unit Tests** in `services/client-api/tests/api/test_analytics_competitors.py` cover:
   - Happy path for all 3 endpoints (Professional+ token → correct response shape)
   - Tier gate: Starter tier JWT → 403 with `upgrade_required: True` on all 3 endpoints
   - Tier gate: Free/unauthenticated → 401/403 on all 3 endpoints
   - Cross-tenant isolation: Company B rows are not included in Company A's results
   - Empty result: returns HTTP 200 with `{"items": [], "total": 0, ...}` for profiles; `{"items": []}` for benchmarks
   - Competitor not found: `GET /competitors/unknown-name/patterns` → 404
   - Pagination: `page=2` with `page_size=5` returns correct window of results
   - `sort_by`/`sort_dir` `Literal` validation: invalid values → 422
   - `Cache-Control: public, max-age=1800` header present on all 3 endpoints

---

### Frontend — Competitor Intelligence Dashboard

9. **AC9** — A new page is created at `apps/client/app/[locale]/(protected)/analytics/competitors/page.tsx`. It is a server component shell that renders `<CompetitorIntelligenceDashboard />`.

10. **AC10 — Filter Panel**: Renders at the top with `data-testid="competitor-filters"`. Contains:
    - **Sector** input — `<Input data-testid="competitor-filter-sector" />` with label `t("analytics.competitors.filterSectorPlaceholder")`
    - **Sort By** select — `data-testid="competitor-filter-sort-by"`, options: Total Bids / Avg Win Rate / Name
    - **Apply** button — `data-testid="competitor-filter-apply"`
    - **Clear** button — `data-testid="competitor-filter-clear"`

11. **AC11 — Competitor Profile Cards**: Renders with `data-testid="competitor-profile-cards"`. Each card has `data-testid="competitor-card-{competitor_name}"` (slugified). Card displays:
    - `competitor_name` — card heading
    - `total_bids` — labelled `t("analytics.competitors.cardTotalBids")`
    - `avg_win_rate` formatted as `x.x%` — labelled `t("analytics.competitors.cardAvgWinRate")`
    - `sectors` — displayed as pill/badge tags
    - `avg_bid_value_eur` — formatted as currency string

    Clicking a card selects it and populates the pattern chart (AC13).

12. **AC12 — Comparison Table**: Renders with `data-testid="competitor-comparison-table"`. Users can select 2–4 competitors for side-by-side comparison using checkboxes on each profile card. The table shows selected competitor names as columns and rows for: Total Bids, Avg Win Rate, Avg Bid Value. The table is hidden when fewer than 2 competitors are selected. A clear-all-selection button (`data-testid="competitor-compare-clear"`) resets the selection.

13. **AC13 — Pattern Chart**: Renders with `data-testid="competitor-pattern-chart"`. A Recharts `<BarChart>` populated from the `/{competitor_name}/patterns` response. X-axis: sector names. Y-axis: bid count. Shows the currently selected competitor's per-sector activity. Renders an empty state (`data-testid="competitor-pattern-empty"`) with `t("analytics.competitors.patternEmptyTitle")` when no competitor is selected or no data returned.

14. **AC14 — Professional+ Upgrade Gate**: When the API returns HTTP 403 on initial load, instead of the dashboard render `data-testid="competitor-upgrade-gate"` containing:
    - An upgrade message: `t("analytics.competitors.upgradeMessage")`
    - A CTA button linking to `/settings/billing`: `data-testid="competitor-upgrade-cta"`

15. **AC15 — i18n**: Add `analytics.competitors` keys to both `messages/en.json` and `messages/bg.json`. Required keys:
    - `pageTitle`, `pageDescription`
    - `filterSectorPlaceholder`, `filterApplyBtn`, `filterClearBtn`
    - `filterSortLabel`, `sortOptionTotalBids`, `sortOptionAvgWinRate`, `sortOptionName`
    - `cardTotalBids`, `cardAvgWinRate`, `cardSectors`, `cardAvgBidValue`
    - `comparisonTableTitle`, `comparisonColTotalBids`, `comparisonColAvgWinRate`, `comparisonColAvgBidValue`
    - `patternChartTitle`, `patternEmptyTitle`, `patternXAxisLabel`, `patternYAxisLabel`
    - `upgradeMessage`, `upgradeCta`
    - `pagingLabel`, `paginationPrevious`, `paginationNext`
    - `profilesEmptyTitle`

16. **AC16 — Frontend Unit Tests**: `apps/client/__tests__/competitor-intelligence-s12-6.test.ts` verifies structural existence of all components, `data-testid` attributes, API fetch functions, TanStack Query hooks, and i18n keys. Follows the same structural verification pattern used in `team-performance-s12-5.test.ts`.

---

## Tasks / Subtasks

### Backend

- [x] Task 1: Implement `require_professional_plus_tier` in `core/tier_gate.py`
  - [x] Define `PROFESSIONAL_PLUS_PLANS: frozenset[str] = frozenset({"professional", "enterprise"})`
  - [x] Replace `raise NotImplementedError(...)` stub with real subscription query (same pattern as `require_paid_tier`)
  - [x] Raise `ForbiddenError("Competitor intelligence requires a Professional or Enterprise subscription.", details={"upgrade_required": True})`
  - [x] Add `PROFESSIONAL_PLUS_PLANS` to `__all__`
- [x] Task 2: Competitor Pydantic schemas — append to `schemas/analytics.py`
  - [x] Define `CompetitorProfileItem` (competitor_name, total_bids, avg_win_rate, sectors, avg_bid_value_eur)
  - [x] Define `CompetitorPatternItem` (sector, bid_count, win_rate, avg_bid_value_eur)
  - [x] Define `CompetitorBenchmarkItem` (sector, competitor_count, avg_bid_value_eur, max_bid_value_eur, min_bid_value_eur)
  - [x] Add all 3 to `__all__`
- [x] Task 3: Competitor analytics service — `services/analytics_competitor_service.py`
  - [x] Implement `get_competitor_profiles(company_id, sector, sort_by, sort_dir, page, page_size, session)` → `(list[CompetitorProfileItem], int)`
  - [x] Implement `get_competitor_patterns(company_id, competitor_name, session)` → `list[CompetitorPatternItem]`
  - [x] Implement `get_competitor_benchmarks(company_id, session)` → `list[CompetitorBenchmarkItem]`
  - [x] Ensure `company_id` scoping is the first WHERE clause in every query
- [x] Task 4: Competitor analytics router — `api/v1/analytics_competitors.py`
  - [x] Declare routes in order: `/profiles`, `/benchmarks`, `/{competitor_name}/patterns`
  - [x] All 3 routes use `require_professional_plus_tier` dependency
  - [x] `sort_by` typed as `Literal["total_bids", "avg_win_rate", "competitor_name"]`
  - [x] `sort_dir` typed as `Literal["asc", "desc"]`
  - [x] Set `Cache-Control: public, max-age=1800` on all responses
- [x] Task 5: Register router in `main.py` — prefix `/api/v1`
- [x] Task 6: Backend unit tests — `tests/api/test_analytics_competitors.py`

### Frontend

- [x] Task 7: Update `lib/api/analytics.ts` with Competitor interfaces and fetch functions
  - [x] `CompetitorFilters`, `CompetitorProfileItem`, `CompetitorPatternItem`, `CompetitorBenchmarkItem`
  - [x] `fetchCompetitorProfiles(filters, page, pageSize)`, `fetchCompetitorPatterns(competitorName)`, `fetchCompetitorBenchmarks()`
- [x] Task 8: TanStack Query hooks — `lib/queries/use-competitor-analytics.ts`
  - [x] `useCompetitorProfiles(filters, page, pageSize)`
  - [x] `useCompetitorPatterns(competitorName)` — enabled only when `competitorName` is non-null
  - [x] `useCompetitorBenchmarks()`
- [x] Task 9: Add `analytics.competitors` keys to `messages/en.json` and `messages/bg.json`
- [x] Task 10: Create components in `app/[locale]/(protected)/analytics/competitors/components/`
  - [x] `CompetitorFilters.tsx` — filter panel (data-testid="competitor-filters")
  - [x] `CompetitorProfileCards.tsx` — card grid with selection (data-testid="competitor-profile-cards")
  - [x] `CompetitorComparisonTable.tsx` — 2–4 competitor side-by-side table (data-testid="competitor-comparison-table")
  - [x] `CompetitorPatternChart.tsx` — Recharts BarChart per-sector (data-testid="competitor-pattern-chart")
  - [x] `TierUpgradeGate.tsx` — 403 state with upgrade CTA (data-testid="competitor-upgrade-gate")
- [x] Task 11: Dashboard orchestrator `CompetitorIntelligenceDashboard.tsx` — coordinates state: active filters, selected competitor for pattern chart, comparison selection (2–4 names)
- [x] Task 12: Page shell `app/[locale]/(protected)/analytics/competitors/page.tsx`
- [x] Task 13: Frontend unit tests — `__tests__/competitor-intelligence-s12-6.test.ts`

---

## Dev Notes

### Architecture & Materialized View

- **Materialized view**: `client.mv_competitor_intelligence`
- **Schema columns** (from `models/analytics_views.py`):
  - `company_id` UUID — tenant scope
  - `competitor_name` Text — natural competitor key (use as URL path param, decoded automatically by FastAPI)
  - `sector` Text — one row per competitor × sector combination
  - `total_bid_count` BigInteger — bids in this sector by this competitor
  - `avg_win_rate` Numeric — average win rate for this competitor in this sector
  - `avg_bid_value_eur` Numeric — average bid value in EUR for this sector

- The view has **one row per (company_id, competitor_name, sector) combination**. The `/profiles` endpoint must GROUP BY `competitor_name` to produce one profile card per competitor.

### Critical: Implementing `require_professional_plus_tier` (Task 1)

The stub in `core/tier_gate.py` currently raises `NotImplementedError`. This MUST be replaced with a real implementation following the same pattern as `require_paid_tier` but using `PROFESSIONAL_PLUS_PLANS = frozenset({"professional", "enterprise"})` instead of `PAID_PLANS`.

The ForbiddenError message for this gate should be specific: `"Competitor intelligence requires a Professional or Enterprise subscription."` — distinct from the generic paid-tier message.

Implementing this correctly will unblock the 7 P0 ATDD tests in `e2e/specs/analytics/tier-gate-enforcement.api.spec.ts` that are currently in `test.skip()` RED phase, specifically:
- `Free tier user gets 403 on competitor intelligence endpoints`
- `Starter tier user gets 403 on competitor intelligence endpoints`
- `Professional tier user gets 200 on competitor intelligence endpoints`
- `Enterprise tier user gets 200 on competitor intelligence endpoints`

### Routing Conflict Prevention (AC7)

FastAPI routes are matched top-to-bottom within a router. If `/{competitor_name}/patterns` is declared before `/benchmarks`, then `GET /competitors/benchmarks` would be matched with `competitor_name = "benchmarks"` before reaching the `/benchmarks` route. Always declare static routes (`/profiles`, `/benchmarks`) before parameterised routes (`/{competitor_name}/patterns`).

### `competitor_name` Path Parameter

Since `mv_competitor_intelligence` has no surrogate `competitor_id`, the `competitor_name` text value is used as the natural key in the URL. FastAPI automatically URL-decodes path parameters, so `"ABC Corp"` transmitted as `ABC%20Corp` will be received as `"ABC Corp"` in the handler. The service layer uses an exact string match: `WHERE competitor_name = :competitor_name AND company_id = :company_id`.

### Security & Tenant Isolation (E12-R-001)

- **Cross-tenant isolation**: Every service function MUST apply `WHERE company_id = current_user.company_id` as the primary, unconditional filter. This must appear before sector filters or sort clauses.
- **Tier gate (E12-R-002)**: `require_professional_plus_tier` must be used on ALL 3 endpoints. The P0 tests cover all 3 endpoints for each disqualifying tier — do not miss any route.

### Performance (E12-PERF, E12-R-003)

- Analytics reads target p95 < 500ms. The `mv_competitor_intelligence` view is refreshed daily by Celery Beat (S12.01 infrastructure).
- The GROUP BY aggregation for `/profiles` operates on the materialized view, not the raw tables — this should be fast. If slow on large datasets, add a DB-level index on `(company_id, competitor_name)`.
- Run `EXPLAIN ANALYZE` on the grouped profiles query during staging validation.

### Frontend State Management

`CompetitorIntelligenceDashboard.tsx` manages:
1. **Filter state** — `sector`, `sortBy`, `sortDir` — passed down to `CompetitorFilters` and used in `useCompetitorProfiles` query
2. **Selected competitor for pattern chart** — `selectedCompetitorName: string | null` — set when a profile card is clicked; passed to `useCompetitorPatterns` (enabled only when non-null)
3. **Comparison selection** — `compareNames: string[]` (max 4) — maintained as a Set-like array; toggled by card checkboxes; passed to `CompetitorComparisonTable`

The `useCompetitorPatterns` hook must use `enabled: !!selectedCompetitorName` to avoid sending empty-string requests.

### TDD Phase — P0 E2E Tests

The P0 ATDD tests for tier-gate enforcement on competitor endpoints are in RED phase in `e2e/specs/analytics/tier-gate-enforcement.api.spec.ts`. After implementing `require_professional_plus_tier` and deploying the 3 endpoints:

1. Remove `test.skip()` from the competitor-related tests in that file
2. Run: `npx playwright test e2e/specs/analytics/tier-gate-enforcement.api.spec.ts`
3. Verify GREEN

The cross-tenant isolation test at `e2e/specs/analytics/cross-tenant-isolation.api.spec.ts` covers multiple analytics domains in a single assertion. Ensure competitor data isolation is verified by the seeded cross-tenant pair used in that test.

### Upgrade Gate UI

When `useCompetitorProfiles` (or any competitor hook) returns a 403 response, `CompetitorIntelligenceDashboard` should render `TierUpgradeGate` instead of the normal dashboard. The upgrade gate component should accept `message` and `ctaHref` props. `ctaHref` should be `/settings/billing` (consistent with Usage Dashboard S12.08 upgrade CTA pattern).

---

## File List

### Backend
- `services/client-api/src/client_api/core/tier_gate.py` — implement `require_professional_plus_tier`, add `PROFESSIONAL_PLUS_PLANS`
- `services/client-api/src/client_api/schemas/analytics.py` — add `CompetitorProfileItem`, `CompetitorPatternItem`, `CompetitorBenchmarkItem`
- `services/client-api/src/client_api/services/analytics_competitor_service.py` — service layer (get_competitor_profiles, get_competitor_patterns, get_competitor_benchmarks)
- `services/client-api/src/client_api/api/v1/analytics_competitors.py` — router with Literal-typed sort params, routing order, Cache-Control headers
- `services/client-api/src/client_api/main.py` — register analytics_competitors router
- `services/client-api/tests/api/test_analytics_competitors.py` — backend unit tests

### Frontend
- `frontend/apps/client/lib/api/analytics.ts` — Competitor interfaces and fetch functions
- `frontend/apps/client/lib/queries/use-competitor-analytics.ts` — TanStack Query hooks (useCompetitorProfiles, useCompetitorPatterns with enabled guard, useCompetitorBenchmarks)
- `frontend/apps/client/messages/en.json` — analytics.competitors i18n keys (EN)
- `frontend/apps/client/messages/bg.json` — analytics.competitors i18n keys (BG)
- `frontend/apps/client/app/[locale]/(protected)/analytics/competitors/page.tsx` — server component page shell
- `frontend/apps/client/app/[locale]/(protected)/analytics/competitors/components/CompetitorIntelligenceDashboard.tsx` — dashboard orchestrator (filter/selection state)
- `frontend/apps/client/app/[locale]/(protected)/analytics/competitors/components/CompetitorFilters.tsx` — filter panel (data-testid="competitor-filters")
- `frontend/apps/client/app/[locale]/(protected)/analytics/competitors/components/CompetitorProfileCards.tsx` — card grid with click-select and compare checkboxes (data-testid="competitor-profile-cards")
- `frontend/apps/client/app/[locale]/(protected)/analytics/competitors/components/CompetitorComparisonTable.tsx` — 2–4 competitor side-by-side comparison (data-testid="competitor-comparison-table")
- `frontend/apps/client/app/[locale]/(protected)/analytics/competitors/components/CompetitorPatternChart.tsx` — Recharts BarChart (data-testid="competitor-pattern-chart")
- `frontend/apps/client/app/[locale]/(protected)/analytics/competitors/components/TierUpgradeGate.tsx` — 403 upgrade prompt (data-testid="competitor-upgrade-gate")
- `frontend/apps/client/__tests__/competitor-intelligence-s12-6.test.ts` — frontend ATDD unit tests

---

## Change Log

- **2026-04-13**: Story created — draft. Tier gate P0 ATDD in RED phase awaiting implementation.
- **2026-04-13**: Full-stack implementation complete — all 13 tasks done; 22 backend + 93 frontend tests passing; e2e tier-gate ATDD unskipped (GREEN phase for S12.06 competitor endpoints).
- **2026-04-13**: Senior Developer Review patches applied — 5 patch items resolved: (1) i18n for hardcoded "Clear"/"Metric" strings in CompetitorComparisonTable; (2) `"en-EU"` → `"en-IE"` locale fix; (3) `formatWinRate`/`formatCurrency` extracted to shared `competitor-utils.ts`; (4) typed `isHttpError` guard in `lib/api/error-utils.ts` replaces fragile cast; (5) `cardSectors` i18n key now used in CompetitorProfileCards. All 93 frontend tests still passing.

---

## Dev Agent Record

### Implementation Plan

Implemented full-stack competitor intelligence dashboard following existing patterns from S12.05 (Team Performance). Key decisions:

1. **Tier gate**: Replaced `NotImplementedError` stub in `tier_gate.py` with a real subscription query using `PROFESSIONAL_PLUS_PLANS = frozenset({"professional", "enterprise"})`. Updated the one existing test that validated the old stub behaviour.

2. **Backend service**: `analytics_competitor_service.py` queries `mv_competitor_intelligence` materialized view. `/profiles` groups by `competitor_name` using SQL aggregates; sectors are fetched in a separate query per competitor (avoids JSON aggregation dialect issues). `company_id` is always the first WHERE clause (E12-R-001).

3. **Routing order (AC7)**: `/profiles` and `/benchmarks` are declared before `/{competitor_name}/patterns` in the router — verified by two dedicated routing order tests.

4. **Frontend**: Follows team-performance S12.05 patterns exactly. `CompetitorIntelligenceDashboard.tsx` manages all state (filters, selected competitor, comparison names). `useCompetitorPatterns` uses `enabled: !!competitorName` guard. `TierUpgradeGate` renders on 403 response.

5. **Tests**: 22 backend unit tests (all ACs covered) + 93 frontend structural/ATDD tests. Pre-existing test failures in audit-trail, espd, grant-tools confirmed unchanged.

### Completion Notes

✅ Task 1 — `require_professional_plus_tier` implemented + `PROFESSIONAL_PLUS_PLANS` defined  
✅ Task 2 — `CompetitorProfileItem`, `CompetitorPatternItem`, `CompetitorBenchmarkItem` schemas added  
✅ Task 3 — `analytics_competitor_service.py` with 3 service functions, `company_id` first clause  
✅ Task 4 — Router with static routes before parameterised route, `Literal` sort params, Cache-Control  
✅ Task 5 — Router registered in `main.py`  
✅ Task 6 — 22 backend unit tests: happy path, tier gate, cross-tenant, pagination, 422, routing  
✅ Task 7 — `analytics.ts` updated with Competitor interfaces and fetch functions  
✅ Task 8 — `use-competitor-analytics.ts` with 3 hooks including `enabled` guard  
✅ Task 9 — All 27 i18n keys added to both `en.json` and `bg.json`  
✅ Task 10 — 5 components: Filters, ProfileCards, ComparisonTable, PatternChart, TierUpgradeGate  
✅ Task 11 — `CompetitorIntelligenceDashboard.tsx` orchestrator  
✅ Task 12 — Page shell `competitors/page.tsx` (server component, no "use client")  
✅ Task 13 — 93 frontend structural tests: all pass  
✅ E2E ATDD — removed `test.skip()` from competitor tier-gate tests in `tier-gate-enforcement.api.spec.ts`  
✅ Ruff linting — all checks pass on new/modified files  
✅ No regressions in backend or frontend test suites

---

## Senior Developer Review

**Date:** 2026-04-13
**Verdict:** Changes Requested
**All 16 ACs:** PASS | **Architecture Alignment:** PASS — no drift

### Review Findings

- [x] [Review][Patch] Hardcoded strings in CompetitorComparisonTable — "Clear" button text and "Metric" table header are hardcoded in English instead of using `t()` i18n functions. Violates i18n completeness intent of AC15. [`CompetitorComparisonTable.tsx`] — fixed: "Clear" uses `t("filterClearBtn")`; "Metric" uses `t("comparisonMetricHeader")` (new key added to en.json and bg.json)
- [x] [Review][Patch] Invalid `"en-EU"` locale in `Intl.NumberFormat` — `"en-EU"` is not a valid BCP 47 locale tag; currency formatting will silently fall back to default locale, producing inconsistent EUR display. Use `"en-IE"` or `"de-DE"` instead. [`CompetitorProfileCards.tsx`, `CompetitorComparisonTable.tsx`] — fixed: `formatCurrency` extracted to `competitor-utils.ts` with `"en-IE"` locale
- [x] [Review][Patch] Duplicated utility functions — `formatWinRate` and `formatCurrency` are copy-pasted identically in both `CompetitorProfileCards.tsx` and `CompetitorComparisonTable.tsx`. Extract to a shared utils module. [`CompetitorProfileCards.tsx`, `CompetitorComparisonTable.tsx`] — fixed: extracted to `components/competitor-utils.ts`; both components import from there
- [x] [Review][Patch] Fragile 403 detection for upgrade gate — The cast `(profilesQuery.error as { status?: number })?.status === 403` depends on error shape from apiClient. Consider adding a typed error guard utility for robustness. [`CompetitorIntelligenceDashboard.tsx`] — fixed: created `lib/api/error-utils.ts#isHttpError(error, status)` which safely navigates both `error.response.status` (AxiosError) and `error.status`; used in Dashboard
- [x] [Review][Patch] Unused `cardSectors` i18n key — Key exists in both en.json and bg.json but is never referenced in `CompetitorProfileCards.tsx`. Sectors are rendered without a label, unlike other card fields. Either use the key or remove it. [`CompetitorProfileCards.tsx`] — fixed: added `t("cardSectors"):` label before sector pill badges, consistent with other card fields
- [x] [Review][Defer] N+1 query for sectors in `get_competitor_profiles` — Each competitor in the page triggers a separate query for distinct sectors (up to 100 extra queries at max page_size). Consider `array_agg(DISTINCT sector)` for single-query approach. Bounded by page_size, not blocking. [`analytics_competitor_service.py`] — deferred, performance optimization
- [x] [Review][Defer] No rate limiting on competitor analytics endpoints — No server-side rate limiting. Pre-existing architectural pattern across all analytics endpoints. [`analytics_competitors.py`] — deferred, pre-existing
- [x] [Review][Defer] Tier gate subscription query on every request — `require_professional_plus_tier` hits the DB per request with no caching. Follows same pattern as `require_paid_tier`. [`tier_gate.py`] — deferred, pre-existing
