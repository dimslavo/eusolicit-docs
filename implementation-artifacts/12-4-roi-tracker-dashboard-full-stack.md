# Story 12.4: ROI Tracker Dashboard (Full Stack)

Status: approved

## Story

As a **paid-tier user on the EU Solicit platform**,
I want **an ROI tracker dashboard at `/analytics/roi`**,
so that **I can see aggregate return on investment (total invested, total won, ROI %) via summary cards, explore per-bid investment vs. outcome in a sortable paginated table, and view monthly ROI trends on a dual-line chart — all filterable by date range**.

## Acceptance Criteria

### Backend — ROI Summary Endpoint

1. **AC1** — `GET /api/v1/analytics/roi/summary` returns aggregate ROI metrics for the authenticated user's company. Response body is a JSON object with fields:
   - `total_invested_eur` (Decimal serialised as string) — `SUM(mv_roi_tracker.total_invested_eur)` across matching rows
   - `total_won_eur` (Decimal serialised as string) — `SUM(mv_roi_tracker.total_won_eur)` across matching rows
   - `roi_pct` (Decimal serialised as string) — `ROUND((total_won_eur − total_invested_eur) / total_invested_eur × 100, 2)` computed in Python; returns `"0"` when `total_invested_eur` is zero
   - `bid_count` (integer) — `COUNT(DISTINCT proposal_id)` across matching rows

   Supports `date_from: date | None` and `date_to: date | None` query parameters that filter on `mv_roi_tracker.month`. When no rows match, returns all-zero values with `bid_count: 0` — HTTP 200, never 404.

2. **AC2** — `GET /api/v1/analytics/roi/bids` returns a paginated per-bid breakdown. Response body is `{"items": [...], "total": N, "page": P, "page_size": S}` where each item contains:
   - `proposal_id` (UUID string)
   - `month` (ISO date string, first day of month)
   - `total_invested_eur` (Decimal string or null)
   - `total_won_eur` (Decimal string or null)
   - `roi_pct` (Decimal string or null)

   Supports `date_from`, `date_to` filters; `sort_by` parameter (choices: `total_invested_eur`, `total_won_eur`, `roi_pct`, `month`; default: `month`); `sort_dir` parameter (`asc` | `desc`; default: `desc`). Paginated via `page` (default 1) and `page_size` (default 20, max 100).

3. **AC3** — `GET /api/v1/analytics/roi/trends` returns monthly ROI trend data grouped by month. Response body is `{"items": [...]}` where each item contains `month`, `total_invested_eur`, `total_won_eur`, `roi_pct` (ROI % re-calculated per month from monthly sums — same formula as AC1). Results ordered by `month ASC`. Supports `date_from`, `date_to` filters. Not paginated.

4. **AC4** — All 3 endpoints require a valid Bearer JWT (→ 401 on missing/invalid) and an active paid subscription (→ 403 via `require_paid_tier`). The 403 body uses `ForbiddenError` with `details={"upgrade_required": True}` (same pattern as S12.02).

5. **AC5** — All 3 endpoints scope queries with `WHERE company_id = current_user.company_id` as the first, non-optional filter clause (E12-R-001 cross-tenant isolation). This clause must appear before any optional date filters in every service function.

6. **AC6** — All 3 endpoints include `Cache-Control: public, max-age=1800` response header (materialised view refreshed daily; 30-minute staleness acceptable).

7. **AC7** — Unit tests in `services/client-api/tests/api/test_analytics_roi.py` cover: happy path for all 3 endpoints; `date_from`/`date_to` filter narrows results correctly; empty result returns HTTP 200 (not 4xx); cross-tenant isolation (other company's rows not returned); `Cache-Control` header present on all 3 endpoints; 401 for unauthenticated request; 403 with `upgrade_required` for free-tier company; `roi_pct` calculation correctness (`(won − invested) / invested × 100`); zero-invested edge case returns `roi_pct = "0"`; `bid_count` correct in `/summary`; `sort_by`/`sort_dir` parameters affect row order in `/bids`; pagination `page`/`page_size` respected.

---

### Frontend — ROI Tracker Dashboard

8. **AC8** — A new page is created at `apps/client/app/[locale]/(protected)/analytics/roi/page.tsx`. It is a server component shell (no `"use client"` directive) that imports and renders `<RoiTrackerDashboard />` from `./components/RoiTrackerDashboard`. The route resolves to `/{locale}/analytics/roi`.

9. **AC9** — A filter panel renders at the top of the dashboard with `data-testid="roi-filters"`. It contains:
   - **Date From** — `<Input type="date" data-testid="roi-filter-date-from" />` with label `t("analytics.roi.filterDateFrom")`
   - **Date To** — `<Input type="date" data-testid="roi-filter-date-to" />` with label `t("analytics.roi.filterDateTo")`
   - **Apply button** — `<Button data-testid="roi-filter-apply-btn">` with label `t("analytics.roi.filterApplyBtn")`, disabled when any query is loading
   - **Clear button** — `<Button variant="outline" data-testid="roi-filter-clear-btn">` with label `t("analytics.roi.filterClearBtn")`

   Filter state uses the draft/applied split pattern from S12.03: two `useState` objects (`draftFilters` for controlled inputs, `appliedFilters` for the active query params). Filters are applied only when Apply is clicked, not on every keystroke.

10. **AC10** — Four summary cards render below the filters inside `<div data-testid="roi-summary-cards" className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4 mb-6">`:
    - **Total Invested** — `data-testid="roi-card-invested"`: heading `t("analytics.roi.cardInvestedTitle")`, value formatted as `€{N,NNN.NN}`
    - **Total Won** — `data-testid="roi-card-won"`: heading `t("analytics.roi.cardWonTitle")`, value formatted as `€{N,NNN.NN}`
    - **ROI %** — `data-testid="roi-card-roi-pct"`: heading `t("analytics.roi.cardRoiTitle")`, value formatted as `{N.NN}%`. Text colour: `text-green-600` when `parseFloat(roi_pct) > 0`, `text-red-600` when `< 0`, `text-slate-600` when `=== 0`.
    - **Bids Tracked** — `data-testid="roi-card-bid-count"`: heading `t("analytics.roi.cardBidCountTitle")`, integer value.

    While data is loading, `<SkeletonCard data-testid="roi-summary-skeleton" />` renders in place of the cards row.

11. **AC11** — A per-bid table section renders below the cards with `data-testid="roi-bids-section"`. It has a heading `t("analytics.roi.bidsTableTitle")` and contains:
    - `<table data-testid="roi-bids-table">` with four columns: Month, Total Invested, Total Won, ROI %.
    - Column headers are clickable for **server-side** sorting. Each `<th>` has `data-testid="roi-col-{key}"` where key is `month`, `total_invested_eur`, `total_won_eur`, `roi_pct`. Clicking a `<th>` triggers a sort state update and re-query (not client-side sort). An arrow indicator (▲/▼) reflects the active sort column and direction.
    - Table rows: `<tr data-testid="roi-bid-row-{index}">` with cells: month as `MMM YYYY`, invested/won as `€N,NNN.NN`, ROI % as `N.NN%` (text `text-red-600` when negative).
    - Pagination controls render below the table: `<Button data-testid="roi-bids-prev-btn">` (disabled on page 1), `<span data-testid="roi-bids-page-indicator">` showing page and total, `<Button data-testid="roi-bids-next-btn">` (disabled on last page).
    - Loading → `<SkeletonTable rows={5} data-testid="roi-bids-skeleton" />`. Empty → `<EmptyState data-testid="roi-bids-empty" title={t("analytics.roi.bidsEmptyTitle")} />`.

12. **AC12** — A trend chart section renders below the bids table with `data-testid="roi-trend-chart-section"`. It has a heading `t("analytics.roi.trendChartTitle")` and contains a Recharts `<LineChart>` in `<ResponsiveContainer width="100%" height={280}>`:
    - X-axis `<XAxis dataKey="month">` with month labels formatted as `MMM YYYY`.
    - Two lines: `<Line dataKey="total_invested_eur" stroke="#3b82f6" dot={false} />` and `<Line dataKey="total_won_eur" stroke="#10b981" dot={false} />`.
    - `<Tooltip>` showing month, invested, won, and ROI % on hover. `<CartesianGrid strokeDasharray="3 3" />` and `<Legend />`.
    - Uses data from `useRoiTrends(appliedFilters)`. Loading → `<SkeletonCard data-testid="roi-trend-skeleton" />`. Empty → `<EmptyState data-testid="roi-trend-empty" title={t("analytics.roi.trendEmptyTitle")} />`.

13. **AC13** — `apps/client/lib/api/analytics.ts` (created in S12.03) is **updated** (not replaced) to add ROI types and fetch functions:
    - `RoiFilters` interface: `{ dateFrom?: string; dateTo?: string }`
    - `RoiSummary` interface: `{ total_invested_eur: string; total_won_eur: string; roi_pct: string; bid_count: number }`
    - `RoiBidItem` interface: `{ proposal_id: string; month: string; total_invested_eur: string | null; total_won_eur: string | null; roi_pct: string | null }`
    - `RoiTrendItem` interface: `{ month: string; total_invested_eur: string | null; total_won_eur: string | null; roi_pct: string | null }`
    - `fetchRoiSummary(filters: RoiFilters): Promise<RoiSummary>` — calls `GET /api/v1/analytics/roi/summary`
    - `fetchRoiBids(filters: RoiFilters, sortBy?: string, sortDir?: string, page?: number, pageSize?: number): Promise<PaginatedResponse<RoiBidItem>>` — calls `GET /api/v1/analytics/roi/bids` with `sort_by`, `sort_dir`, `page`, `page_size` query params
    - `fetchRoiTrends(filters: RoiFilters): Promise<ListResponse<RoiTrendItem>>` — calls `GET /api/v1/analytics/roi/trends`

14. **AC14** — A new file `apps/client/lib/queries/use-roi-analytics.ts` is created with `"use client"` directive. It exports:
    - `useRoiSummary(filters: RoiFilters)` — `queryKey: ["roi-summary", filters]`, staleTime: `1_800_000`
    - `useRoiBids(filters: RoiFilters, sortBy: string, sortDir: string, page: number)` — `queryKey: ["roi-bids", filters, sortBy, sortDir, page]`, staleTime: `1_800_000`
    - `useRoiTrends(filters: RoiFilters)` — `queryKey: ["roi-trends", filters]`, staleTime: `1_800_000`

15. **AC15** — An `roi` sub-key is added to the `analytics` object in both `apps/client/messages/en.json` and `apps/client/messages/bg.json`:
    ```json
    "roi": {
      "pageTitle": "ROI Tracker",
      "pageDescription": "Track your investment vs. return across bids.",
      "filterDateFrom": "Date From",
      "filterDateTo": "Date To",
      "filterApplyBtn": "Apply Filters",
      "filterClearBtn": "Clear",
      "cardInvestedTitle": "Total Invested",
      "cardWonTitle": "Total Won",
      "cardRoiTitle": "ROI",
      "cardBidCountTitle": "Bids Tracked",
      "bidsTableTitle": "Per-Bid Breakdown",
      "bidsEmptyTitle": "No bid data",
      "colMonth": "Month",
      "colInvested": "Total Invested",
      "colWon": "Total Won",
      "colRoi": "ROI %",
      "trendChartTitle": "ROI Trend Over Time",
      "trendInvestedLabel": "Invested",
      "trendWonLabel": "Won",
      "trendEmptyTitle": "No trend data",
      "pagingLabel": "Page {page} of {total}"
    }
    ```
    All `t("analytics.roi.*")` calls must resolve without TypeScript errors.

16. **AC16** — A unit test file `apps/client/__tests__/roi-tracker-s12-4.test.ts` is created (Vitest, `environment: 'node'`) covering:
    - Page shell exists at `analytics/roi/page.tsx` and does NOT contain `"use client"`
    - `RoiTrackerDashboard.tsx` exists in the `components/` subdirectory
    - `RoiFilters.tsx`, `RoiSummaryCards.tsx`, `RoiBidsTable.tsx`, `RoiTrendChart.tsx` all exist
    - `lib/api/analytics.ts` exports `fetchRoiSummary`, `fetchRoiBids`, `fetchRoiTrends`
    - `lib/api/analytics.ts` contains `interface RoiFilters`, `interface RoiSummary`, `interface RoiBidItem`, `interface RoiTrendItem`
    - `lib/queries/use-roi-analytics.ts` exports `useRoiSummary`, `useRoiBids`, `useRoiTrends` and contains `1_800_000`
    - `en.json` and `bg.json` contain all `analytics.roi.*` keys from AC15
    - `RoiTrackerDashboard.tsx` source contains `data-testid="roi-filters"`, `data-testid="roi-summary-cards"`, `data-testid="roi-bids-section"`, `data-testid="roi-trend-chart-section"`

---

## Tasks / Subtasks

### Backend

- [ ] Task 1: Pydantic ROI schemas — append to `schemas/analytics.py` (AC: 1, 2, 3)
  - [ ] 1.1 Open `services/client-api/src/client_api/schemas/analytics.py` and append below existing market schemas
  - [ ] 1.2 Add `from uuid import UUID` import if not present
  - [ ] 1.3 Define `RoiSummaryResponse(BaseModel)` with `model_config = ConfigDict(from_attributes=True)` and fields: `total_invested_eur: Decimal`, `total_won_eur: Decimal`, `roi_pct: Decimal`, `bid_count: int`
  - [ ] 1.4 Define `RoiBidItem(BaseModel)` with `ConfigDict(from_attributes=True)` and fields: `proposal_id: UUID`, `month: date`, `total_invested_eur: Decimal | None = None`, `total_won_eur: Decimal | None = None`, `roi_pct: Decimal | None = None`
  - [ ] 1.5 Define `RoiTrendItem(BaseModel)` with `ConfigDict(from_attributes=True)` and fields: `month: date`, `total_invested_eur: Decimal | None = None`, `total_won_eur: Decimal | None = None`, `roi_pct: Decimal | None = None`
  - [ ] 1.6 Add `RoiSummaryResponse`, `RoiBidItem`, `RoiTrendItem` to `__all__` in `schemas/analytics.py`
  - [ ] 1.7 Export the three new schemas from `services/client-api/src/client_api/schemas/__init__.py`

- [ ] Task 2: ROI analytics service — `services/analytics_roi_service.py` (AC: 1, 2, 3, 5)
  - [ ] 2.1 Create `services/client-api/src/client_api/services/analytics_roi_service.py`
  - [ ] 2.2 Import `mv_roi_tracker` from `client_api.models.analytics_views`
  - [ ] 2.3 Define `_apply_roi_filters(stmt, date_from: date | None, date_to: date | None)` helper — adds optional `mv_roi_tracker.c.month >= date_from` and `mv_roi_tracker.c.month <= date_to` clauses (analogous to `_apply_market_filters` in `analytics_market_service.py`)
  - [ ] 2.4 Define `_compute_roi_pct(total_won: Decimal, total_invested: Decimal) -> Decimal` — returns `round((total_won - total_invested) / total_invested * 100, 2)` when `total_invested > 0`, else `Decimal("0")`
  - [ ] 2.5 Implement `async get_roi_summary(company_id: UUID, date_from: date | None, date_to: date | None, session: AsyncSession) -> RoiSummaryResponse`:
    - SELECT `func.sum(mv_roi_tracker.c.total_invested_eur)`, `func.sum(mv_roi_tracker.c.total_won_eur)`, `func.count(mv_roi_tracker.c.proposal_id.distinct())` WHERE `company_id = :company_id` + optional date filters
    - Compute `roi_pct` in Python via `_compute_roi_pct`
    - If no rows (SUM returns None), return `RoiSummaryResponse(total_invested_eur=Decimal("0"), total_won_eur=Decimal("0"), roi_pct=Decimal("0"), bid_count=0)`
  - [ ] 2.6 Implement `async get_roi_bids(company_id: UUID, date_from: date | None, date_to: date | None, sort_by: str, sort_dir: str, page: int, page_size: int, session: AsyncSession) -> tuple[list[RoiBidItem], int]`:
    - Validate `sort_by` in `{"total_invested_eur", "total_won_eur", "roi_pct", "month"}` — default to `"month"` if invalid
    - Build base stmt: SELECT all view columns WHERE `company_id = :company_id` + optional date filters
    - Count query: `select(func.count()).select_from(base_stmt.subquery())`
    - Order rows by `getattr(mv_roi_tracker.c, sort_by).desc()` or `.asc()` based on `sort_dir`; apply OFFSET/LIMIT
    - Return `(items, total_count)`
  - [ ] 2.7 Implement `async get_roi_trends(company_id: UUID, date_from: date | None, date_to: date | None, session: AsyncSession) -> list[RoiTrendItem]`:
    - SELECT `mv_roi_tracker.c.month`, `func.sum(total_invested_eur)`, `func.sum(total_won_eur)` WHERE `company_id = :company_id` + optional date filters; GROUP BY `month`; ORDER BY `month ASC`
    - Compute `roi_pct` per row in Python after fetching (call `_compute_roi_pct` per row)
    - Return list of `RoiTrendItem`
  - [ ] 2.8 ALL service functions MUST have `company_id: UUID` as the first parameter and `WHERE mv_roi_tracker.c.company_id == company_id` as the first WHERE clause — mandatory, non-optional (E12-R-001)

- [ ] Task 3: ROI analytics router — `api/v1/analytics_roi.py` (AC: 1, 2, 3, 4, 6)
  - [ ] 3.1 Create `services/client-api/src/client_api/api/v1/analytics_roi.py` with `router = APIRouter(prefix="/analytics/roi", tags=["analytics"])` and `_CACHE_CONTROL = "public, max-age=1800"`
  - [ ] 3.2 Implement `GET /summary` endpoint: `date_from: date | None = Query(None)`, `date_to: date | None = Query(None)`. Dependencies: `current_user: Annotated[CurrentUser, Depends(require_paid_tier)]`, `session: Annotated[AsyncSession, Depends(get_db_session)]`. Inject `response: Response`. Return type: `RoiSummaryResponse`. Set `Cache-Control` header then call `analytics_roi_service.get_roi_summary(company_id=current_user.company_id, ...)`.
  - [ ] 3.3 Implement `GET /bids` endpoint: add `sort_by: str = Query(default="month")`, `sort_dir: str = Query(default="desc")`, `page: int = Query(default=1, ge=1)`, `page_size: int = Query(default=20, ge=1, le=100)` on top of date filters. Return type: `PaginatedResponse[RoiBidItem]`. Set `Cache-Control` header.
  - [ ] 3.4 Implement `GET /trends` endpoint: `date_from`, `date_to` params only. Return type: `ListResponse[RoiTrendItem]`. Set `Cache-Control` header.
  - [ ] 3.5 All three endpoints extract `company_id = current_user.company_id` and pass to the corresponding service function

- [ ] Task 4: Register ROI router in `main.py` (AC: 1, 2, 3)
  - [ ] 4.1 Open `services/client-api/src/client_api/main.py`
  - [ ] 4.2 Add `from client_api.api.v1 import analytics_roi as analytics_roi_v1` import after the existing `analytics_market` import
  - [ ] 4.3 Add `api_v1_router.include_router(analytics_roi_v1.router)` after `api_v1_router.include_router(analytics_market_v1.router)`
  - [ ] 4.4 Verify full paths resolve to `/api/v1/analytics/roi/summary`, `/api/v1/analytics/roi/bids`, `/api/v1/analytics/roi/trends`

- [ ] Task 5: Backend unit tests — `tests/api/test_analytics_roi.py` (AC: 7)
  - [ ] 5.1 Create `services/client-api/tests/api/test_analytics_roi.py`
  - [ ] 5.2 Fixture `roi_context`: seed a company with Starter plan (active subscription); seed 3 `mv_roi_tracker` rows via direct INSERT as `migration_role` (see Dev Notes for seeding approach):
    - Row A: `proposal_id=UUID_A`, `month=date(2025, 1, 1)`, `total_invested_eur=1000`, `total_won_eur=1500`, `roi_pct=50.00`
    - Row B: `proposal_id=UUID_B`, `month=date(2025, 2, 1)`, `total_invested_eur=2000`, `total_won_eur=1800`, `roi_pct=-10.00`
    - Row C: `proposal_id=UUID_C`, `month=date(2025, 1, 1)`, `total_invested_eur=500`, `total_won_eur=0`, `roi_pct=-100.00`
    Return auth headers and seeded proposal UUIDs
  - [ ] 5.3 Fixture `other_company_row`: seed 1 `mv_roi_tracker` row for a different company (cross-tenant check)
  - [ ] 5.4 **GET /summary — happy path**: assert 200; `bid_count = 3`; `total_invested_eur` == "3500.00"; `total_won_eur` == "3300.00"; `roi_pct` ≈ "-5.71" (or "−5.71" depending on rounding — accept `Decimal` comparison within 0.01)
  - [ ] 5.5 **GET /summary — date filter**: `?date_from=2025-02-01`; assert `bid_count = 1`; totals reflect Row B only
  - [ ] 5.6 **GET /summary — empty result**: `?date_from=2030-01-01`; assert 200, `bid_count = 0`, `total_invested_eur == "0"`, `roi_pct == "0"`
  - [ ] 5.7 **GET /summary — zero-invested edge case**: (same as empty result) verify `roi_pct` is `"0"`, not a division error
  - [ ] 5.8 **GET /summary — cross-tenant isolation**: with `other_company_row` seeded; query without filters; assert `bid_count` is still 3 (other company row not counted)
  - [ ] 5.9 **GET /summary — cache header**: assert `response.headers["Cache-Control"] == "public, max-age=1800"`
  - [ ] 5.10 **GET /summary — unauthenticated**: no `Authorization` header; assert 401
  - [ ] 5.11 **GET /summary — free-tier**: free plan company; assert 403, body contains `upgrade_required: true`
  - [ ] 5.12 **GET /bids — happy path**: assert 200; `total = 3`; each item has `proposal_id`, `month`, `total_invested_eur`, `total_won_eur`, `roi_pct`
  - [ ] 5.13 **GET /bids — default sort (month desc)**: assert first item has `month = date(2025, 2, 1)` (Row B, most recent)
  - [ ] 5.14 **GET /bids — sort by roi_pct asc**: `?sort_by=roi_pct&sort_dir=asc`; assert first item is Row C (`roi_pct = -100.00`)
  - [ ] 5.15 **GET /bids — sort by total_invested_eur desc**: `?sort_by=total_invested_eur&sort_dir=desc`; assert first item is Row B (2000 invested)
  - [ ] 5.16 **GET /bids — date filter**: `?date_from=2025-02-01`; assert `total = 1`, only Row B returned
  - [ ] 5.17 **GET /bids — pagination**: `?page=1&page_size=2`; assert `total = 3`, `len(items) = 2`; `?page=2&page_size=2`; assert `len(items) = 1`
  - [ ] 5.18 **GET /bids — empty result**: `?date_from=2030-01-01`; assert 200, `items = []`, `total = 0`
  - [ ] 5.19 **GET /bids — cross-tenant isolation**: assert other company row not in results
  - [ ] 5.20 **GET /bids — cache header, 401, 403**: same patterns as 5.9, 5.10, 5.11
  - [ ] 5.21 **GET /trends — happy path**: assert 200; 2 items (Jan 2025, Feb 2025); ordered by month ASC
  - [ ] 5.22 **GET /trends — Jan 2025 aggregation**: `total_invested_eur` = "1500.00" (Row A + Row C), `total_won_eur` = "1500.00"; `roi_pct` = "0.00" (1500−1500)/1500×100)
  - [ ] 5.23 **GET /trends — Feb 2025**: `total_invested_eur` = "2000.00", `total_won_eur` = "1800.00", `roi_pct` = "-10.00"
  - [ ] 5.24 **GET /trends — date filter**: `?date_from=2025-02-01&date_to=2025-02-28`; assert 1 item (Feb only)
  - [ ] 5.25 **GET /trends — empty result, cache header, cross-tenant, 401, 403**: same coverage pattern

---

### Frontend

- [ ] Task 6: TypeScript interfaces and fetch functions — update `lib/api/analytics.ts` (AC: 13)
  - [ ] 6.1 Open `apps/client/lib/api/analytics.ts` (created in S12.03; do NOT recreate — append only)
  - [ ] 6.2 Append `RoiFilters` interface: `export interface RoiFilters { dateFrom?: string; dateTo?: string }`
  - [ ] 6.3 Append `RoiSummary` interface: `export interface RoiSummary { total_invested_eur: string; total_won_eur: string; roi_pct: string; bid_count: number }`
  - [ ] 6.4 Append `RoiBidItem` interface: `export interface RoiBidItem { proposal_id: string; month: string; total_invested_eur: string | null; total_won_eur: string | null; roi_pct: string | null }`
  - [ ] 6.5 Append `RoiTrendItem` interface: `export interface RoiTrendItem { month: string; total_invested_eur: string | null; total_won_eur: string | null; roi_pct: string | null }`
  - [ ] 6.6 Implement `function buildRoiParams(filters: RoiFilters)` — maps `{ dateFrom?, dateTo? }` to `{ date_from?, date_to? }`, omitting undefined/empty-string values (analogous to `buildMarketParams`)
  - [ ] 6.7 Implement `export async function fetchRoiSummary(filters: RoiFilters): Promise<RoiSummary>` — calls `GET /api/v1/analytics/roi/summary` via `apiClient` with `buildRoiParams(filters)` as params; returns `response.data`
  - [ ] 6.8 Implement `export async function fetchRoiBids(filters: RoiFilters, sortBy = "month", sortDir = "desc", page = 1, pageSize = 20): Promise<PaginatedResponse<RoiBidItem>>` — calls `GET /api/v1/analytics/roi/bids` with `{ ...buildRoiParams(filters), sort_by: sortBy, sort_dir: sortDir, page, page_size: pageSize }` as params
  - [ ] 6.9 Implement `export async function fetchRoiTrends(filters: RoiFilters): Promise<ListResponse<RoiTrendItem>>` — calls `GET /api/v1/analytics/roi/trends` with `buildRoiParams(filters)` as params

- [ ] Task 7: TanStack Query hooks — `lib/queries/use-roi-analytics.ts` (AC: 14)
  - [ ] 7.1 Create `apps/client/lib/queries/use-roi-analytics.ts` with `"use client"` directive at the top
  - [ ] 7.2 Import `useQuery` from `@tanstack/react-query`; import `fetchRoiSummary`, `fetchRoiBids`, `fetchRoiTrends`, `RoiFilters` from `@/lib/api/analytics`
  - [ ] 7.3 Export `useRoiSummary(filters: RoiFilters)`: `useQuery({ queryKey: ["roi-summary", filters], queryFn: () => fetchRoiSummary(filters), staleTime: 1_800_000 })`
  - [ ] 7.4 Export `useRoiBids(filters: RoiFilters, sortBy: string, sortDir: string, page: number)`: `useQuery({ queryKey: ["roi-bids", filters, sortBy, sortDir, page], queryFn: () => fetchRoiBids(filters, sortBy, sortDir, page), staleTime: 1_800_000 })`
  - [ ] 7.5 Export `useRoiTrends(filters: RoiFilters)`: `useQuery({ queryKey: ["roi-trends", filters], queryFn: () => fetchRoiTrends(filters), staleTime: 1_800_000 })`

- [ ] Task 8: i18n — add `analytics.roi` keys to both message files (AC: 15)
  - [ ] 8.1 Open `apps/client/messages/en.json`; locate the `"analytics"` object added in S12.03; add `"roi": { ... }` sub-key with all English values from AC15
  - [ ] 8.2 Open `apps/client/messages/bg.json`; add `"roi": { ... }` sub-key with Bulgarian values (see Dev Notes for Bulgarian strings)

- [ ] Task 9: `RoiFilters` component — filter panel (AC: 9)
  - [ ] 9.1 Create `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiFilters.tsx` as `"use client"` component
  - [ ] 9.2 Props: `{ dateFrom: string; dateTo: string; isLoading: boolean; onChange: (field: "dateFrom" | "dateTo", value: string) => void; onApply: () => void; onClear: () => void }`
  - [ ] 9.3 Render two labeled `<Input type="date">` fields with `data-testid="roi-filter-date-from"` and `data-testid="roi-filter-date-to"`, controlled via `value` and `onChange` props
  - [ ] 9.4 Render Apply `<Button data-testid="roi-filter-apply-btn" disabled={isLoading} onClick={onApply}>` and Clear `<Button variant="outline" data-testid="roi-filter-clear-btn" onClick={onClear}>`
  - [ ] 9.5 Wrap entire filter panel in `<div data-testid="roi-filters" className="bg-white rounded-lg border p-4 mb-6 flex flex-wrap gap-3 items-end">`

- [ ] Task 10: `RoiSummaryCards` component — KPI cards (AC: 10)
  - [ ] 10.1 Create `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiSummaryCards.tsx` as `"use client"` component
  - [ ] 10.2 Props: `{ data: RoiSummary | undefined; isLoading: boolean }`
  - [ ] 10.3 When `isLoading`, render `<div className="mb-6"><SkeletonCard data-testid="roi-summary-skeleton" /></div>`
  - [ ] 10.4 When `data` is present, render `<div data-testid="roi-summary-cards" className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4 mb-6">` with four cards:
    - Each card: `<div className="bg-white rounded-lg border p-4"><p className="text-sm font-medium text-slate-500 mb-1">{heading}</p><p className="text-2xl font-bold ...">{value}</p></div>`
    - Total Invested: `data-testid="roi-card-invested"`, value `€{parseFloat(data.total_invested_eur).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`
    - Total Won: `data-testid="roi-card-won"`, same euro format
    - ROI %: `data-testid="roi-card-roi-pct"`, class `text-green-600` / `text-red-600` / `text-slate-600` based on sign; value `{parseFloat(data.roi_pct).toFixed(2)}%`
    - Bid Count: `data-testid="roi-card-bid-count"`, value `{data.bid_count}`

- [ ] Task 11: `RoiBidsTable` component — server-sorted paginated table (AC: 11)
  - [ ] 11.1 Create `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiBidsTable.tsx` as `"use client"` component
  - [ ] 11.2 Props: `{ data: RoiBidItem[]; total: number; isLoading: boolean; sortBy: string; sortDir: "asc" | "desc"; page: number; pageSize: number; onSort: (col: string) => void; onPage: (page: number) => void }`
  - [ ] 11.3 When `isLoading`, render `<SkeletonTable rows={5} data-testid="roi-bids-skeleton" />`
  - [ ] 11.4 When `data.length === 0` and not loading, render `<EmptyState data-testid="roi-bids-empty" title={t("analytics.roi.bidsEmptyTitle")} />`
  - [ ] 11.5 Render `<table data-testid="roi-bids-table">` with four `<th data-testid="roi-col-{key}" onClick={() => onSort(key)} className="cursor-pointer select-none">` headers. Active sort column shows ▲ (`sortDir === "asc"`) or ▼ (`sortDir === "desc"`).
  - [ ] 11.6 Render rows: `<tr data-testid="roi-bid-row-{index}">` with cells:
    - Month: `new Date(item.month + "T00:00:00").toLocaleDateString(undefined, { year: "numeric", month: "short" })`
    - Invested/Won: `€{parseFloat(item.total_invested_eur ?? "0").toLocaleString(undefined, { minimumFractionDigits: 2 })}`
    - ROI %: `<span className={parseFloat(item.roi_pct ?? "0") < 0 ? "text-red-600" : ""}>{parseFloat(item.roi_pct ?? "0").toFixed(2)}%</span>`
  - [ ] 11.7 Compute `totalPages = Math.ceil(total / pageSize)`. Render pagination: `<Button data-testid="roi-bids-prev-btn" disabled={page === 1} onClick={() => onPage(page - 1)}>`, `<span data-testid="roi-bids-page-indicator">{t("analytics.roi.pagingLabel", { page, total: totalPages })}</span>`, `<Button data-testid="roi-bids-next-btn" disabled={page >= totalPages} onClick={() => onPage(page + 1)}>`
  - [ ] 11.8 Wrap entire section in `<div data-testid="roi-bids-section">`

- [ ] Task 12: `RoiTrendChart` component — dual-line Recharts chart (AC: 12)
  - [ ] 12.1 Create `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiTrendChart.tsx` as `"use client"` component
  - [ ] 12.2 Props: `{ data: RoiTrendItem[]; isLoading: boolean; locale: string }`
  - [ ] 12.3 When `isLoading`, render `<SkeletonCard data-testid="roi-trend-skeleton" />`
  - [ ] 12.4 When `data.length === 0` and not loading, render `<EmptyState data-testid="roi-trend-empty" title={t("analytics.roi.trendEmptyTitle")} />`
  - [ ] 12.5 Otherwise render:
    ```tsx
    <div data-testid="roi-trend-chart-section">
      <h2 className="text-lg font-semibold text-slate-800 mb-4">{t("analytics.roi.trendChartTitle")}</h2>
      <ResponsiveContainer width="100%" height={280}>
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" stroke="#f0f0f0" />
          <XAxis dataKey="month" tickFormatter={(v) => new Date(v + "T00:00:00").toLocaleDateString(locale, { year: "numeric", month: "short" })} />
          <YAxis tickFormatter={(v) => `€${(v / 1_000).toFixed(0)}K`} />
          <Tooltip ... />
          <Legend />
          <Line type="monotone" dataKey="total_invested_eur" stroke="#3b82f6" dot={false} name={t("analytics.roi.trendInvestedLabel")} />
          <Line type="monotone" dataKey="total_won_eur" stroke="#10b981" dot={false} name={t("analytics.roi.trendWonLabel")} />
        </LineChart>
      </ResponsiveContainer>
    </div>
    ```
  - [ ] 12.6 Tooltip `formatter`: display `€{parseFloat(value).toLocaleString(undefined, { minimumFractionDigits: 2 })}` for monetary values; include ROI % from the data payload
  - [ ] 12.7 Import `ResponsiveContainer`, `LineChart`, `Line`, `XAxis`, `YAxis`, `CartesianGrid`, `Tooltip`, `Legend` from `recharts`

- [ ] Task 13: `RoiTrackerDashboard` — orchestrator component (AC: 8, 9, 10, 11, 12)
  - [ ] 13.1 Create `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiTrackerDashboard.tsx` as `"use client"` component
  - [ ] 13.2 Draft/applied filter state:
    ```ts
    const [draftFilters, setDraftFilters] = useState({ dateFrom: "", dateTo: "" })
    const [appliedFilters, setAppliedFilters] = useState<RoiFilters>({})
    ```
  - [ ] 13.3 Sort/page state:
    ```ts
    const [sortBy, setSortBy] = useState("month")
    const [sortDir, setSortDir] = useState<"asc" | "desc">("desc")
    const [page, setPage] = useState(1)
    ```
  - [ ] 13.4 Call all three hooks:
    ```ts
    const summaryQuery = useRoiSummary(appliedFilters)
    const bidsQuery = useRoiBids(appliedFilters, sortBy, sortDir, page)
    const trendsQuery = useRoiTrends(appliedFilters)
    ```
  - [ ] 13.5 `handleApply`: build `RoiFilters` from `draftFilters` (omit empty strings → omit the key entirely); call `setAppliedFilters(...)`; `setPage(1)`
  - [ ] 13.6 `handleClear`: reset `draftFilters` to `{ dateFrom: "", dateTo: "" }`; call `setAppliedFilters({})`; `setPage(1)`
  - [ ] 13.7 `handleSort(col: string)`: if `col === sortBy` toggle `sortDir`; else `setSortBy(col)`, `setSortDir("desc")`; `setPage(1)`
  - [ ] 13.8 `isAnyLoading = summaryQuery.isLoading || bidsQuery.isLoading || trendsQuery.isLoading`
  - [ ] 13.9 Render order:
    1. Page header with `data-testid="roi-page-title"` and `data-testid="roi-page-description"`
    2. `<RoiFilters dateFrom={draftFilters.dateFrom} dateTo={draftFilters.dateTo} isLoading={isAnyLoading} onChange={...} onApply={handleApply} onClear={handleClear} />`
    3. `<RoiSummaryCards data={summaryQuery.data} isLoading={summaryQuery.isLoading} />`
    4. `<RoiBidsTable data={bidsQuery.data?.items ?? []} total={bidsQuery.data?.total ?? 0} isLoading={bidsQuery.isLoading} sortBy={sortBy} sortDir={sortDir} page={page} pageSize={20} onSort={handleSort} onPage={setPage} />`
    5. `<RoiTrendChart data={trendsQuery.data?.items ?? []} isLoading={trendsQuery.isLoading} locale={locale} />`
  - [ ] 13.10 Root element: `<div data-testid="roi-tracker-page" className="p-6 max-w-7xl mx-auto">`
  - [ ] 13.11 Get `locale` from `useParams()` (cast as `string`) for the chart component

- [ ] Task 14: Page shell — server component (AC: 8)
  - [ ] 14.1 Create `apps/client/app/[locale]/(protected)/analytics/roi/page.tsx` as a server component — no `"use client"` directive
  - [ ] 14.2 Import and render `<RoiTrackerDashboard />` from `./components/RoiTrackerDashboard`

- [ ] Task 15: Frontend unit test — `__tests__/roi-tracker-s12-4.test.ts` (AC: 16)
  - [ ] 15.1 Create `apps/client/__tests__/roi-tracker-s12-4.test.ts` using the Vitest `environment: 'node'` pattern from `market-intelligence-s12-3.test.ts`
  - [ ] 15.2 Define path constants:
    - `ROI_DIR = join(PROTECTED_DIR, "analytics", "roi")`
    - `COMPONENTS_DIR = join(ROI_DIR, "components")`
  - [ ] 15.3 Page shell: verify `analytics/roi/page.tsx` exists and does NOT contain `"use client"`
  - [ ] 15.4 Components: verify `RoiTrackerDashboard.tsx`, `RoiFilters.tsx`, `RoiSummaryCards.tsx`, `RoiBidsTable.tsx`, `RoiTrendChart.tsx` all exist in `COMPONENTS_DIR`
  - [ ] 15.5 API module: verify `lib/api/analytics.ts` exports `fetchRoiSummary`, `fetchRoiBids`, `fetchRoiTrends` (check `export async function {name}` substring)
  - [ ] 15.6 API module: verify `lib/api/analytics.ts` contains `interface RoiFilters`, `interface RoiSummary`, `interface RoiBidItem`, `interface RoiTrendItem`
  - [ ] 15.7 Query hooks: verify `lib/queries/use-roi-analytics.ts` exports `useRoiSummary`, `useRoiBids`, `useRoiTrends` (check `export function {name}`) and contains `1_800_000`
  - [ ] 15.8 i18n: verify all `analytics.roi.*` keys from AC15 exist in both `en.json` and `bg.json` using `resolveNestedKey` helper
  - [ ] 15.9 Dashboard testids: verify `RoiTrackerDashboard.tsx` source contains all four required `data-testid` values

---

## Dev Notes

### Architecture Context

**Backend service:** `services/client-api` — FastAPI on port 8001, async SQLAlchemy (asyncpg driver). Pattern files to study: `services/client-api/src/client_api/api/v1/analytics_market.py` (router pattern), `services/client-api/src/client_api/services/analytics_market_service.py` (service pattern with `_apply_*_filters` helper and `_compute_*` helper pattern).

**Frontend app:** `apps/client` — Next.js 14, TanStack Query, Recharts, next-intl, Tailwind CSS. Pattern files: `apps/client/app/[locale]/(protected)/analytics/market/` (the entire S12.03 implementation — directly mirrors what S12.04 must build for ROI). Reuse the same component structure.

**Depends on S12.01 (done):** `mv_roi_tracker` materialised view is deployed in migration `011_analytics_materialized_views.py`. SQLAlchemy `Table` definition exists in `services/client-api/src/client_api/models/analytics_views.py`.

**Depends on S12.02 (done):** `require_paid_tier` dependency in `core/tier_gate.py` is implemented. `PaginatedResponse[T]` and `ListResponse[T]` generic schemas exist in `schemas/analytics.py`. Import these — do not redefine.

**Depends on S12.03 (done/review):** `apps/client/lib/api/analytics.ts` and `apps/client/lib/queries/use-market-analytics.ts` exist. This story **appends to** `analytics.ts` — do not overwrite the market functions.

---

### `mv_roi_tracker` — Column Reference

```
services/client-api/src/client_api/models/analytics_views.py
```

| Column | SQLAlchemy Type | Description |
|--------|----------------|-------------|
| `company_id` | `UUID(as_uuid=True)` | Tenant key — **every query must filter by this** |
| `proposal_id` | `UUID(as_uuid=True)` | One row per proposal (unique composite key with company_id) |
| `month` | `Date` | `date_trunc('month', proposals.created_at)::date` — first day of month |
| `total_invested_eur` | `Numeric` | `SUM(bpl.hours * 50 + COALESCE(bpl.cost_eur, 0))` across bid_preparation_logs |
| `total_won_eur` | `Numeric` | `MAX(CASE WHEN bo.status = 'won' THEN bo.contract_value_eur ELSE 0 END)` |
| `roi_pct` | `Numeric` | Pre-computed ROI % in the view (may be stale until next refresh). The `/summary` and `/trends` endpoints **re-compute** roi_pct in Python from fresh SUM aggregates — do not trust the view's `roi_pct` for aggregate endpoints. |

**Import:**
```python
from client_api.models.analytics_views import mv_roi_tracker
```

---

### CRITICAL: Company-ID Scoping (E12-R-001, Risk Score 6)

Every query against `mv_roi_tracker` MUST include `WHERE company_id = :company_id` as the first, non-optional filter. This is the primary defence against cross-tenant analytics leakage (E12-R-001 — highest-scored risk in E12). The P0 Playwright test `cross-tenant-isolation.api.spec.ts` seeds Company A and Company B and will sweep all analytics domains (including `/analytics/roi/*`) once S12.04–S12.08 are deployed.

**Service function signature convention:**
```python
async def get_roi_summary(
    company_id: UUID,        # FIRST parameter — always required
    date_from: date | None,
    date_to: date | None,
    session: AsyncSession,
) -> RoiSummaryResponse:
    stmt = (
        select(...)
        .where(mv_roi_tracker.c.company_id == company_id)  # FIRST WHERE — mandatory
        # ... optional date filters follow
    )
```

---

### ROI % Calculation — Formula and Edge Cases

The view stores a pre-computed `roi_pct` per proposal row, but the `/summary` and `/trends` endpoints aggregate across multiple rows and **must re-compute** from the fresh aggregated sums:

```python
from decimal import Decimal, ROUND_HALF_UP

def _compute_roi_pct(total_won: Decimal, total_invested: Decimal) -> Decimal:
    if not total_invested or total_invested == Decimal("0"):
        return Decimal("0")
    return (
        (total_won - total_invested) / total_invested * Decimal("100")
    ).quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
```

**Edge cases the unit tests check:**
- `total_invested = 0`: must return `Decimal("0")`, not raise `ZeroDivisionError`
- `total_won = 0, total_invested > 0`: returns negative ROI (−100%)
- `total_won > total_invested`: returns positive ROI

For the `/bids` endpoint, rows are returned directly from the view — the view's `roi_pct` column is used as-is (already computed per-proposal). The service does NOT re-compute it for the bids list.

---

### sort_by Validation — Allowlist Pattern

The `/bids` endpoint accepts a `sort_by` query parameter. **Validate it against an allowlist** to prevent SQL injection through column name interpolation:

```python
_VALID_SORT_COLUMNS = frozenset({"total_invested_eur", "total_won_eur", "roi_pct", "month"})

async def get_roi_bids(
    ...,
    sort_by: str,
    sort_dir: str,
    ...
):
    if sort_by not in _VALID_SORT_COLUMNS:
        sort_by = "month"  # silently default — no 400 error per spec
    col = getattr(mv_roi_tracker.c, sort_by)
    order_clause = col.desc().nullslast() if sort_dir == "desc" else col.asc().nullsfirst()
```

---

### Backend: Service Query Patterns — SQLAlchemy Core

**Summary aggregation (no GROUP BY):**
```python
from sqlalchemy import func, select

stmt = (
    select(
        func.sum(mv_roi_tracker.c.total_invested_eur).label("total_invested_eur"),
        func.sum(mv_roi_tracker.c.total_won_eur).label("total_won_eur"),
        func.count(mv_roi_tracker.c.proposal_id.distinct()).label("bid_count"),
    )
    .where(mv_roi_tracker.c.company_id == company_id)
)
stmt = _apply_roi_filters(stmt, date_from, date_to)
row = (await session.execute(stmt)).mappings().one_or_none()
```

If `row` is None or all SUM values are None (empty result), return all-zero `RoiSummaryResponse`.

**Trend aggregation (GROUP BY month):**
```python
stmt = (
    select(
        mv_roi_tracker.c.month,
        func.sum(mv_roi_tracker.c.total_invested_eur).label("total_invested_eur"),
        func.sum(mv_roi_tracker.c.total_won_eur).label("total_won_eur"),
    )
    .where(mv_roi_tracker.c.company_id == company_id)
)
stmt = _apply_roi_filters(stmt, date_from, date_to)
stmt = stmt.group_by(mv_roi_tracker.c.month).order_by(mv_roi_tracker.c.month.asc())
rows = (await session.execute(stmt)).mappings().all()
# Compute roi_pct per row in Python:
items = [
    RoiTrendItem(
        month=row["month"],
        total_invested_eur=row["total_invested_eur"],
        total_won_eur=row["total_won_eur"],
        roi_pct=_compute_roi_pct(
            row["total_won_eur"] or Decimal("0"),
            row["total_invested_eur"] or Decimal("0"),
        ),
    )
    for row in rows
]
```

---

### Test Data Seeding for Backend Unit Tests

`mv_roi_tracker` is a PostgreSQL materialised view — cannot INSERT via `client_api_role` (SELECT-only). Use the same direct-INSERT bypass pattern established in S12.02 tests:

```python
# Run as migration_role (has full CRUD)
await migration_session.execute(text("""
    INSERT INTO client.mv_roi_tracker
        (company_id, proposal_id, month, total_invested_eur, total_won_eur, roi_pct)
    VALUES
        (:company_id, :proposal_id_a, '2025-01-01', 1000, 1500, 50.00),
        (:company_id, :proposal_id_b, '2025-02-01', 2000, 1800, -10.00),
        (:company_id, :proposal_id_c, '2025-01-01', 500, 0, -100.00)
"""), {...})
await migration_session.commit()
```

Check whether the `migration_role` session fixture already exists in `services/client-api/tests/conftest.py` before creating one — S12.02 tests (`test_analytics_market.py`) likely introduced it. Reuse it.

Add `@pytest.mark.integration` to any test class/method that uses real PostgreSQL connections (see S12.02 review finding F5 — `TestCrossTenantServiceSQLIsolation` should have had this marker).

Add `finally: await session.rollback()` in any fixture's cleanup block (see S12.02 review finding F6).

---

### Frontend: Pattern References

| Concern | Reference File |
|---------|---------------|
| Server component shell | `apps/client/app/[locale]/(protected)/analytics/market/page.tsx` |
| Client dashboard orchestrator | `apps/client/app/[locale]/(protected)/analytics/market/components/MarketIntelligenceDashboard.tsx` |
| Filter component (draft/applied pattern) | `apps/client/app/[locale]/(protected)/analytics/market/components/MarketFilters.tsx` |
| Line chart component | `apps/client/app/[locale]/(protected)/analytics/market/components/TrendLineChart.tsx` |
| TanStack Query two-layer pattern | `apps/client/lib/api/analytics.ts` + `apps/client/lib/queries/use-market-analytics.ts` |
| Vitest node-env structural test | `apps/client/__tests__/market-intelligence-s12-3.test.ts` |
| `apiClient` import | `import { apiClient } from "@eusolicit/ui"` |

The `RoiBidsTable` uses **server-side sort** (re-queries with new `sort_by`/`sort_dir`) unlike S12.03's `AuthoritiesTable` which sorts client-side. This is intentional — the `/bids` endpoint can paginate thousands of rows; client-side sorting is impractical.

---

### Frontend: Bulgarian i18n Strings (`bg.json`)

```json
"roi": {
  "pageTitle": "Проследяване на ROI",
  "pageDescription": "Проследявайте инвестициите и приходите от офертите.",
  "filterDateFrom": "От дата",
  "filterDateTo": "До дата",
  "filterApplyBtn": "Приложи филтри",
  "filterClearBtn": "Изчисти",
  "cardInvestedTitle": "Общо инвестирано",
  "cardWonTitle": "Общо спечелено",
  "cardRoiTitle": "ROI",
  "cardBidCountTitle": "Проследени оферти",
  "bidsTableTitle": "Разбивка по оферта",
  "bidsEmptyTitle": "Няма данни за оферти",
  "colMonth": "Месец",
  "colInvested": "Инвестирано",
  "colWon": "Спечелено",
  "colRoi": "ROI %",
  "trendChartTitle": "Тенденция на ROI",
  "trendInvestedLabel": "Инвестирано",
  "trendWonLabel": "Спечелено",
  "trendEmptyTitle": "Няма данни за тенденции",
  "pagingLabel": "Страница {page} от {total}"
}
```

---

### File Locations Summary

**Backend — New files:**

| File | Action |
|------|--------|
| `services/client-api/src/client_api/schemas/analytics.py` | **Modify** — append 3 ROI schemas |
| `services/client-api/src/client_api/schemas/__init__.py` | **Modify** — add ROI schema exports |
| `services/client-api/src/client_api/services/analytics_roi_service.py` | **Create** |
| `services/client-api/src/client_api/api/v1/analytics_roi.py` | **Create** |
| `services/client-api/src/client_api/main.py` | **Modify** — register ROI router |
| `services/client-api/tests/api/test_analytics_roi.py` | **Create** |

**Frontend — New files:**

| File | Action |
|------|--------|
| `apps/client/lib/api/analytics.ts` | **Modify** — append ROI interfaces + fetch functions |
| `apps/client/lib/queries/use-roi-analytics.ts` | **Create** |
| `apps/client/messages/en.json` | **Modify** — add `analytics.roi` sub-key |
| `apps/client/messages/bg.json` | **Modify** — add `analytics.roi` sub-key |
| `apps/client/app/[locale]/(protected)/analytics/roi/page.tsx` | **Create** |
| `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiTrackerDashboard.tsx` | **Create** |
| `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiFilters.tsx` | **Create** |
| `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiSummaryCards.tsx` | **Create** |
| `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiBidsTable.tsx` | **Create** |
| `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiTrendChart.tsx` | **Create** |
| `apps/client/__tests__/roi-tracker-s12-4.test.ts` | **Create** |

---

### Test Coverage Alignment with Epic-Level Test Design

From `eusolicit-docs/test-artifacts/test-design-epic-12.md`:

| Priority | Scenario | Maps to this story |
|----------|----------|-------------------|
| **P0** | Cross-tenant isolation sweep (analytics domains) | AC5 + Task 5.8/5.19 — un-skip `cross-tenant-isolation.api.spec.ts` after S12.04–S12.08 complete |
| **P0** | Load test — analytics p95 < 500ms under 50 concurrent users | `mv_roi_tracker` included in k6 load test targets (S12.17) |
| **P1** | S12.04 ROI tracker API — `/summary` totals, ROI % formula, per-bid pagination, trends | AC1–AC3, AC7 (Task 5 unit tests) |
| **P2** | S12.04 ROI tracker frontend — summary cards, sortable bids table, trend chart | AC8–AC12 (Tasks 9–14) |

**E12-R-001 mitigation for this story:** S12.04 adds 3 new analytics endpoints. The `company_id` WHERE clause (Task 2.8) must be verified in code review. The cross-tenant unit tests (Task 5.8, 5.19) provide automated guard at the unit level. The P0 Playwright `cross-tenant-isolation.api.spec.ts` test covers the `/analytics/roi/*` domain once S12.04 is deployed — un-skipping is managed in S12.08 (after all analytics domains complete).

**E12-R-006 note:** `mv_roi_tracker` refresh is covered by S12.01 automation (`automation-summary-story-12-1.md`). The k6 load test in S12.17 targets this view.

---

## Senior Developer Review

**Status:** Approved
**Reviewed:** 2026-04-12
**Reviewer:** Gemini (bmad-code-review)

### Verdict: APPROVED

### Findings

| # | Severity | Category | Finding |
|---|---|---|---|
| F1 | **Deferrable** | Missing Requirement | `RoiBidsTable.tsx` uses translation keys `paginationPrevious` and `paginationNext`, which were not specified in AC15 (though the implementation correctly added them to `en.json` and `bg.json`). |

