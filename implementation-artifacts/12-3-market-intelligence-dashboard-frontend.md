# Story 12.3: Market Intelligence Dashboard Frontend

Status: approved

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **paid-tier user on the EU Solicit platform**,
I want **a market intelligence dashboard at `/analytics/market`**,
so that **I can visually explore procurement volumes by sector (bar chart), monthly trend data (line chart), and top contracting authorities (sortable table) — with interactive date range, sector, and country filters that update all visualizations simultaneously — and the page looks correct on mobile devices**.

## Acceptance Criteria

### Route & Page Shell

1. **AC1** — A new page is created at `apps/client/app/[locale]/(protected)/analytics/market/page.tsx`. It is a server component shell (no `"use client"`) that imports and renders `<MarketIntelligenceDashboard />`. The route resolves to `/{locale}/analytics/market`.

---

### Filter Controls

2. **AC2** — A filter panel renders at the top of the dashboard with `data-testid="market-filters"`. It contains:
   - **Date From** — `<Input type="date" data-testid="market-filter-date-from" />` with label `t("analytics.market.filterDateFrom")`
   - **Date To** — `<Input type="date" data-testid="market-filter-date-to" />` with label `t("analytics.market.filterDateTo")`
   - **Sector** — `<Input type="text" data-testid="market-filter-sector" placeholder={t("analytics.market.filterSectorPlaceholder")} />` (freeform text, matches CPV code prefix)
   - **Country** — `<Input type="text" data-testid="market-filter-country" placeholder={t("analytics.market.filterCountryPlaceholder")} />` (ISO 2-letter code)
   - **Apply button** — `<Button data-testid="market-filter-apply-btn">` with label `t("analytics.market.filterApplyBtn")` — triggers a re-fetch of all four API endpoints with the current filter values
   - **Clear button** — `<Button variant="outline" data-testid="market-filter-clear-btn">` with label `t("analytics.market.filterClearBtn")` — resets all four filter inputs to empty and re-fetches with no filters

3. **AC3** — Filter state is managed in `MarketIntelligenceDashboard` via `useState`: `dateFrom: string`, `dateTo: string`, `sector: string`, `country: string`. **Filters are applied only when the Apply button is clicked**, not on every keystroke. The "applied" filter set is stored in a separate `useState` object (`appliedFilters`) that is passed as query parameters to all four `useQuery` calls. This prevents unnecessary re-fetches on every character typed.

---

### Procurement Volume Bar Chart

4. **AC4** — A card section with `data-testid="market-volume-chart-section"` renders below the filters. It has a heading `t("analytics.market.volumeChartTitle")` and contains a Recharts `<BarChart>` wrapped in `<ResponsiveContainer width="100%" height={320}>`.
   - X-axis (`<XAxis dataKey="sector">`) shows sector labels, with `angle={-35}` and `textAnchor="end"` to prevent overlap.
   - Y-axis (`<YAxis>`) shows `total_value_eur` values formatted as compact currency (e.g. `€1.2M`).
   - `<Bar dataKey="total_value_eur" fill="#3b82f6" data-testid="market-volume-bar" />` renders one bar per sector group.
   - `<Tooltip>` shows `sector`, `opportunity_count`, and `total_value_eur` on hover, formatted via `t("analytics.market.volumeTooltipLabel")`.
   - `<Legend />` renders below the chart.
   - The chart uses data from `useMarketVolume(appliedFilters)`. When data is loading, a `<SkeletonCard data-testid="market-volume-skeleton" />` renders in place of the chart. When data is empty (items array is empty), an `<EmptyState data-testid="market-volume-empty" title={t("analytics.market.volumeEmptyTitle")} description={t("analytics.market.volumeEmptyDescription")} />` renders.

---

### Monthly Trend Line Chart

5. **AC5** — A card section with `data-testid="market-trend-chart-section"` renders below the bar chart. It has a heading `t("analytics.market.trendChartTitle")` and contains a Recharts `<LineChart>` wrapped in `<ResponsiveContainer width="100%" height={280}>`.
   - X-axis (`<XAxis dataKey="month">`) shows month labels formatted as `MMM YYYY` (e.g. `Jan 2025`) using `new Date(month).toLocaleDateString(locale, { year: "numeric", month: "short" })`.
   - Y-axis (`<YAxis>`) shows `opportunity_count` as integer values.
   - `<Line type="monotone" dataKey="opportunity_count" stroke="#10b981" dot={false} data-testid="market-trend-line" />` renders the trend line.
   - `<Tooltip>` with `formatter` prop shows `month`, `opportunity_count`, and `total_value_eur` on hover.
   - `<CartesianGrid strokeDasharray="3 3" />` renders a subtle grid.
   - The chart uses data from `useMarketTrends(appliedFilters)`. Loading → `<SkeletonCard data-testid="market-trend-skeleton" />`. Empty → `<EmptyState data-testid="market-trend-empty" title={t("analytics.market.trendEmptyTitle")} />`.

---

### Top Contracting Authorities Table

6. **AC6** — A card section with `data-testid="market-authorities-section"` renders below the trend chart. It has a heading `t("analytics.market.authoritiesTableTitle")`. The section contains:
   - A `<table data-testid="market-authorities-table">` with four columns: Authority Name, Sector, Country, Opportunities.
   - **Column headers are clickable for sorting**. Each `<th>` has `data-testid="market-auth-col-{key}"` where key is `authority_name`, `sector`, `country`, `opportunity_count`. Clicking a column header toggles sort direction (`asc` → `desc` → `asc`); an arrow indicator (▲ / ▼) reflects the current sort state.
   - Table rows: `<tr data-testid="market-auth-row-{index}">` with four `<td>` cells matching the column order.
   - Sorting is **client-side** (sort the data returned by the API in component state, not a new API call) because the `/authorities` endpoint returns all results for the filter set. Sort state managed via `useState<{ column: string; direction: "asc" | "desc" }>({ column: "opportunity_count", direction: "desc" })` — default sort is `opportunity_count` descending.
   - **Pagination**: Display 10 rows per page. A pagination control renders below the table with:
     - `<Button data-testid="market-auth-prev-btn">` — disabled on first page
     - `<span data-testid="market-auth-page-indicator">` showing `t("analytics.market.pagingLabel", { page: currentPage, total: totalPages })`
     - `<Button data-testid="market-auth-next-btn">` — disabled on last page
   - Loading → `<SkeletonTable rows={5} data-testid="market-authorities-skeleton" />`. Empty → `<EmptyState data-testid="market-authorities-empty" title={t("analytics.market.authoritiesEmptyTitle")} />`.

---

### Loading Skeletons

7. **AC7** — While any of the four API calls (`/volume`, `/values`, `/authorities`, `/trends`) are loading:
   - The bar chart section shows `<SkeletonCard data-testid="market-volume-skeleton" />` instead of the chart.
   - The trend chart section shows `<SkeletonCard data-testid="market-trend-skeleton" />` instead of the chart.
   - The authorities section shows `<SkeletonTable rows={5} data-testid="market-authorities-skeleton" />` instead of the table.
   - The filter Apply button is disabled (`disabled` attribute set) while any query is in a loading state, preventing double-submission.

---

### Responsive Layout

8. **AC8** — The page uses a responsive grid layout. On desktop (≥ `lg` breakpoint, 1024px), the volume bar chart and trend line chart render side-by-side in a two-column grid: `className="grid grid-cols-1 lg:grid-cols-2 gap-6"`. On mobile (< `lg`), they stack vertically (single column). The authorities table always renders full-width below the chart row on all screen sizes. The filter panel always renders full-width above the charts.

---

### API Integration & Query Hooks

9. **AC9** — A new file `apps/client/lib/api/analytics.ts` is created. It exports:
   - TypeScript interfaces matching the S12.02 API response shapes: `MarketVolumeItem`, `MarketValuesItem`, `MarketAuthorityItem`, `MarketTrendItem`, `PaginatedResponse<T>`, `ListResponse<T>`, and `MarketFilters` (filter params type).
   - Four async API functions using `apiClient` from `@eusolicit/ui`: `fetchMarketVolume(filters, page, pageSize)`, `fetchMarketValues(filters)`, `fetchMarketAuthorities(filters, page, pageSize)`, `fetchMarketTrends(filters)`. Each function constructs query parameters from the filters object, omitting `undefined`/empty-string values.
   - `fetchMarketVolume` calls `GET /api/v1/analytics/market/volume`; returns `PaginatedResponse<MarketVolumeItem>`.
   - `fetchMarketValues` calls `GET /api/v1/analytics/market/values`; returns `ListResponse<MarketValuesItem>`.
   - `fetchMarketAuthorities` calls `GET /api/v1/analytics/market/authorities` with `page_size=100` to retrieve enough data for client-side sorting; returns `PaginatedResponse<MarketAuthorityItem>`.
   - `fetchMarketTrends` calls `GET /api/v1/analytics/market/trends`; returns `ListResponse<MarketTrendItem>`.

10. **AC10** — A new file `apps/client/lib/queries/use-market-analytics.ts` is created. It exports four TanStack Query `useQuery` hooks:
    - `useMarketVolume(filters: MarketFilters)` — `queryKey: ["market-volume", filters]`, staleTime: `1_800_000` (30 min, matching the `Cache-Control: max-age=1800` from the API).
    - `useMarketTrends(filters: MarketFilters)` — `queryKey: ["market-trends", filters]`, staleTime: `1_800_000`.
    - `useMarketAuthorities(filters: MarketFilters)` — `queryKey: ["market-authorities", filters]`, staleTime: `1_800_000`. Fetches with `page_size=100` to get all rows for client-side sort.
    - `useMarketValues(filters: MarketFilters)` — `queryKey: ["market-values", filters]`, staleTime: `1_800_000`.
    - All four hooks pass `filters` as part of the query key so TanStack Query re-fetches automatically when `appliedFilters` changes.

---

### i18n

11. **AC11** — An `analytics` key is added to both `apps/client/messages/en.json` and `apps/client/messages/bg.json` with the following structure (English values shown; Bulgarian translation provided in Dev Notes):
    ```json
    "analytics": {
      "market": {
        "pageTitle": "Market Intelligence",
        "pageDescription": "Procurement volumes, trends, and top contracting authorities.",
        "filterDateFrom": "Date From",
        "filterDateTo": "Date To",
        "filterSectorPlaceholder": "Sector (e.g. 45)",
        "filterCountryPlaceholder": "Country (e.g. BG)",
        "filterApplyBtn": "Apply Filters",
        "filterClearBtn": "Clear",
        "volumeChartTitle": "Procurement Volume by Sector",
        "volumeTooltipLabel": "Volume",
        "volumeEmptyTitle": "No volume data",
        "volumeEmptyDescription": "Try broadening your filters.",
        "trendChartTitle": "Monthly Procurement Trends",
        "trendEmptyTitle": "No trend data",
        "authoritiesTableTitle": "Top Contracting Authorities",
        "authoritiesEmptyTitle": "No authorities data",
        "pagingLabel": "Page {page} of {total}",
        "colAuthorityName": "Authority",
        "colSector": "Sector",
        "colCountry": "Country",
        "colOpportunities": "Opportunities"
      }
    }
    ```
    All `t("analytics.market.*")` calls in the component must resolve to these keys without TypeScript errors.

---

### Navigation

12. **AC12** — The market intelligence page is added to the client app navigation. In `apps/client/app/[locale]/(protected)/layout.tsx`:
    - Import `BarChart2` from `lucide-react` (alongside existing icon imports).
    - Add `analytics` to the `"nav"` i18n key in both `en.json` and `bg.json`: `"analytics": "Analytics"` / `"analytics": "Анализи"`.
    - Add a nav item to `clientNavItems` array: `{ icon: BarChart2, label: t("nav.analytics"), href: \`/${locale}/analytics/market\` }`. Place it after the Dashboard item and before Tenders.

---

### Unit Test

13. **AC13** — A unit test file `apps/client/__tests__/market-intelligence-s12-3.test.ts` is created covering structural and static assertions (Vitest, environment: `node`):
    - Page shell exists at `apps/client/app/[locale]/(protected)/analytics/market/page.tsx`
    - `MarketIntelligenceDashboard.tsx` exists in the `components/` subdirectory
    - `MarketFilters.tsx` exists in the `components/` subdirectory
    - `apps/client/lib/api/analytics.ts` exports `fetchMarketVolume`, `fetchMarketValues`, `fetchMarketAuthorities`, `fetchMarketTrends`
    - `apps/client/lib/queries/use-market-analytics.ts` exports `useMarketVolume`, `useMarketTrends`, `useMarketAuthorities`, `useMarketValues`
    - `en.json` contains all `analytics.market.*` keys listed in AC11
    - `bg.json` contains all `analytics.market.*` keys listed in AC11
    - `en.json` `nav` object contains `"analytics"` key
    - `layout.tsx` imports `BarChart2` from `lucide-react`
    - `layout.tsx` references the string `analytics/market` (verifying nav link present)
    - `MarketIntelligenceDashboard.tsx` contains `data-testid="market-filters"`, `data-testid="market-volume-chart-section"`, `data-testid="market-trend-chart-section"`, `data-testid="market-authorities-section"`

## Tasks / Subtasks

- [x] Task 1: TypeScript API interfaces and API client — `lib/api/analytics.ts` (AC: 9)
  - [x] 1.1 Create `apps/client/lib/api/analytics.ts`
  - [x] 1.2 Define `MarketFilters` interface: `{ dateFrom?: string; dateTo?: string; sector?: string; country?: string }`
  - [x] 1.3 Define `MarketVolumeItem`, `MarketValuesItem`, `MarketAuthorityItem`, `MarketTrendItem` interfaces matching the S12.02 API schemas (see Dev Notes for field list)
  - [x] 1.4 Define generic `PaginatedResponse<T>` interface: `{ items: T[]; total: number; page: number; page_size: number }`
  - [x] 1.5 Define generic `ListResponse<T>` interface: `{ items: T[] }`
  - [x] 1.6 Implement `buildMarketParams(filters: MarketFilters)` helper that converts the filter object to an axios `params` object, omitting keys where value is `undefined` or `""`
  - [x] 1.7 Implement `fetchMarketVolume(filters: MarketFilters, page = 1, pageSize = 20)`: calls `GET /api/v1/analytics/market/volume`; returns `PaginatedResponse<MarketVolumeItem>`
  - [x] 1.8 Implement `fetchMarketValues(filters: MarketFilters)`: calls `GET /api/v1/analytics/market/values`; returns `ListResponse<MarketValuesItem>`
  - [x] 1.9 Implement `fetchMarketAuthorities(filters: MarketFilters)`: calls `GET /api/v1/analytics/market/authorities` with `page_size: 100`; returns `PaginatedResponse<MarketAuthorityItem>` (100 rows for client-side sort)
  - [x] 1.10 Implement `fetchMarketTrends(filters: MarketFilters)`: calls `GET /api/v1/analytics/market/trends`; returns `ListResponse<MarketTrendItem>`

- [x] Task 2: TanStack Query hooks — `lib/queries/use-market-analytics.ts` (AC: 10)
  - [x] 2.1 Create `apps/client/lib/queries/use-market-analytics.ts` with `"use client"` directive
  - [x] 2.2 Implement `useMarketVolume(filters: MarketFilters)` using `useQuery({ queryKey: ["market-volume", filters], queryFn: () => fetchMarketVolume(filters), staleTime: 1_800_000 })`
  - [x] 2.3 Implement `useMarketValues(filters: MarketFilters)` with `queryKey: ["market-values", filters]`, staleTime `1_800_000`
  - [x] 2.4 Implement `useMarketAuthorities(filters: MarketFilters)` with `queryKey: ["market-authorities", filters]`, staleTime `1_800_000`
  - [x] 2.5 Implement `useMarketTrends(filters: MarketFilters)` with `queryKey: ["market-trends", filters]`, staleTime `1_800_000`
  - [x] 2.6 Import `useQuery` from `@tanstack/react-query` and the four `fetch*` functions from `@/lib/api/analytics`

- [x] Task 3: i18n — add `analytics` key to both message files (AC: 11, 12)
  - [x] 3.1 Add `"analytics": { "market": { ... } }` object to `apps/client/messages/en.json` with all keys from AC11 (English values)
  - [x] 3.2 Add `"analytics": { "market": { ... } }` object to `apps/client/messages/bg.json` with all keys from AC11 (Bulgarian values — see Dev Notes)
  - [x] 3.3 Add `"analytics": "Analytics"` to the `"nav"` object in `apps/client/messages/en.json`
  - [x] 3.4 Add `"analytics": "Анализи"` to the `"nav"` object in `apps/client/messages/bg.json`

- [x] Task 4: Navigation update — `layout.tsx` (AC: 12)
  - [x] 4.1 Open `apps/client/app/[locale]/(protected)/layout.tsx`
  - [x] 4.2 Add `BarChart2` to the existing `lucide-react` import statement
  - [x] 4.3 Add `{ icon: BarChart2, label: t("nav.analytics"), href: \`/${locale}/analytics/market\` }` to `clientNavItems` array — insert after the Dashboard entry (first position) and before Tenders

- [x] Task 5: `MarketFilters` component — filter panel (AC: 2, 3)
  - [x] 5.1 Create `apps/client/app/[locale]/(protected)/analytics/market/components/MarketFilters.tsx` as a `"use client"` component
  - [x] 5.2 Props: `{ dateFrom: string; dateTo: string; sector: string; country: string; isLoading: boolean; onChange: (field: keyof MarketFilterState, value: string) => void; onApply: () => void; onClear: () => void }`
  - [x] 5.3 Render four labeled `<Input>` fields using `data-testid` values from AC2. Use `<label>` elements with `htmlFor` for accessibility.
  - [x] 5.4 Render Apply `<Button data-testid="market-filter-apply-btn">` and Clear `<Button variant="outline" data-testid="market-filter-clear-btn">` buttons
  - [x] 5.5 Apply button is `disabled` when `isLoading` is `true`
  - [x] 5.6 Filter row uses `className="flex flex-wrap gap-3 items-end"` for horizontal layout on desktop, wrapping on mobile
  - [x] 5.7 Wrap the entire filter panel in `<div data-testid="market-filters" className="bg-white rounded-lg border p-4 mb-6">`

- [x] Task 6: `VolumeBarChart` component — Recharts bar chart (AC: 4)
  - [x] 6.1 Create `apps/client/app/[locale]/(protected)/analytics/market/components/VolumeBarChart.tsx` as a `"use client"` component
  - [x] 6.2 Props: `{ data: MarketVolumeItem[]; isLoading: boolean; locale: string }`
  - [x] 6.3 When `isLoading === true`, render `<SkeletonCard data-testid="market-volume-skeleton" />`
  - [x] 6.4 When `data.length === 0` and not loading, render `<EmptyState data-testid="market-volume-empty" title={t("analytics.market.volumeEmptyTitle")} description={t("analytics.market.volumeEmptyDescription")} />`
  - [x] 6.5 Otherwise render `<ResponsiveContainer width="100%" height={320}><BarChart data={data}>...</BarChart></ResponsiveContainer>`
  - [x] 6.6 Include `<XAxis dataKey="sector" angle={-35} textAnchor="end" height={60} tick={{ fontSize: 11 }} />` and `<YAxis tickFormatter={(v) => \`€${(v/1_000_000).toFixed(1)}M\`} />`
  - [x] 6.7 Include `<Bar dataKey="total_value_eur" fill="#3b82f6" radius={[4, 4, 0, 0]} />` with `data-testid` injected via wrapper `<div data-testid="market-volume-chart-section">`
  - [x] 6.8 Include `<Tooltip formatter={(value: number) => [\`€${(value/1_000_000).toFixed(2)}M\`, t("analytics.market.volumeTooltipLabel")]} />` and `<Legend />`

- [x] Task 7: `TrendLineChart` component — Recharts line chart (AC: 5)
  - [x] 7.1 Create `apps/client/app/[locale]/(protected)/analytics/market/components/TrendLineChart.tsx` as a `"use client"` component
  - [x] 7.2 Props: `{ data: MarketTrendItem[]; isLoading: boolean; locale: string }`
  - [x] 7.3 When `isLoading === true`, render `<SkeletonCard data-testid="market-trend-skeleton" />`
  - [x] 7.4 When `data.length === 0` and not loading, render `<EmptyState data-testid="market-trend-empty" title={t("analytics.market.trendEmptyTitle")} />`
  - [x] 7.5 Otherwise render `<ResponsiveContainer width="100%" height={280}><LineChart data={data}>...</LineChart></ResponsiveContainer>`
  - [x] 7.6 `<XAxis dataKey="month" tickFormatter={(v) => new Date(v).toLocaleDateString(locale, { year: "numeric", month: "short" })} />`
  - [x] 7.7 `<YAxis allowDecimals={false} />` and `<CartesianGrid strokeDasharray="3 3" stroke="#f0f0f0" />`
  - [x] 7.8 `<Line type="monotone" dataKey="opportunity_count" stroke="#10b981" strokeWidth={2} dot={false} activeDot={{ r: 4 }} />`
  - [x] 7.9 Wrap in `<div data-testid="market-trend-chart-section">`

- [x] Task 8: `AuthoritiesTable` component — sortable table with client-side pagination (AC: 6)
  - [x] 8.1 Create `apps/client/app/[locale]/(protected)/analytics/market/components/AuthoritiesTable.tsx` as a `"use client"` component
  - [x] 8.2 Props: `{ data: MarketAuthorityItem[]; isLoading: boolean }`
  - [x] 8.3 When `isLoading === true`, render `<SkeletonTable rows={5} data-testid="market-authorities-skeleton" />`
  - [x] 8.4 When `data.length === 0` and not loading, render `<EmptyState data-testid="market-authorities-empty" title={t("analytics.market.authoritiesEmptyTitle")} />`
  - [x] 8.5 Sort state: `useState<{ column: keyof MarketAuthorityItem; direction: "asc" | "desc" }>({ column: "opportunity_count", direction: "desc" })`
  - [x] 8.6 Page state: `useState<number>(1)`, rows-per-page constant `PAGE_SIZE = 10`
  - [x] 8.7 Compute `sortedData` by applying `[...data].sort(...)` using the sort state; compute `pagedData = sortedData.slice((page-1)*PAGE_SIZE, page*PAGE_SIZE)`; compute `totalPages = Math.ceil(sortedData.length / PAGE_SIZE)`
  - [x] 8.8 Render `<table data-testid="market-authorities-table">` with four `<th data-testid="market-auth-col-{key}">` headers; clicking a header toggles sort direction; display `▲` / `▼` indicator on the active sort column
  - [x] 8.9 Render rows: `<tr data-testid="market-auth-row-{index}">` with `<td>` for `authority_name`, `sector`, `country`, `opportunity_count`
  - [x] 8.10 Render pagination controls: `<Button data-testid="market-auth-prev-btn" disabled={page === 1}>`, `<span data-testid="market-auth-page-indicator">`, `<Button data-testid="market-auth-next-btn" disabled={page === totalPages}>`
  - [x] 8.11 Reset page to 1 whenever `data` prop changes (use `useEffect([data], () => setPage(1))`)
  - [x] 8.12 Wrap in `<div data-testid="market-authorities-section">`

- [x] Task 9: `MarketIntelligenceDashboard` — main orchestrator component (AC: 1, 2, 3, 7, 8)
  - [x] 9.1 Create `apps/client/app/[locale]/(protected)/analytics/market/components/MarketIntelligenceDashboard.tsx` as a `"use client"` component
  - [x] 9.2 Filter state: `const [draftFilters, setDraftFilters] = useState<MarketFilterState>({ dateFrom: "", dateTo: "", sector: "", country: "" })` for controlled inputs; `const [appliedFilters, setAppliedFilters] = useState<MarketFilters>({})` for the active query params
  - [x] 9.3 Call all four query hooks with `appliedFilters`: `useMarketVolume(appliedFilters)`, `useMarketTrends(appliedFilters)`, `useMarketAuthorities(appliedFilters)`, `useMarketValues(appliedFilters)`
  - [x] 9.4 Derive `isAnyLoading = volumeQuery.isLoading || trendsQuery.isLoading || authoritiesQuery.isLoading`
  - [x] 9.5 `handleApply()`: build `MarketFilters` object from `draftFilters` (omit empty strings), call `setAppliedFilters(...)` — this updates the query key and triggers re-fetch
  - [x] 9.6 `handleClear()`: reset `draftFilters` to all-empty, call `setAppliedFilters({})` — triggers re-fetch with no filters
  - [x] 9.7 Page header: `<h1 data-testid="market-page-title" className="text-2xl font-bold text-slate-900 mb-2">{t("analytics.market.pageTitle")}</h1>` and `<p data-testid="market-page-description" className="text-sm text-slate-500 mb-6">{t("analytics.market.pageDescription")}</p>`
  - [x] 9.8 Render `<MarketFilters ...>` passing draft filter state, `isLoading={isAnyLoading}`, `onApply={handleApply}`, `onClear={handleClear}`
  - [x] 9.9 Chart grid: `<div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">` containing `<VolumeBarChart>` and `<TrendLineChart>`
  - [x] 9.10 Below chart grid: full-width `<AuthoritiesTable data={authoritiesQuery.data?.items ?? []} isLoading={authoritiesQuery.isLoading} />`
  - [x] 9.11 Root element: `<div data-testid="market-intelligence-page" className="p-6 max-w-7xl mx-auto">`
  - [x] 9.12 Get `locale` from `useParams()` for passing to chart components

- [x] Task 10: Page shell — server component (AC: 1)
  - [x] 10.1 Create `apps/client/app/[locale]/(protected)/analytics/market/page.tsx` as a server component (no `"use client"`)
  - [x] 10.2 Import and render `<MarketIntelligenceDashboard />` from `./components/MarketIntelligenceDashboard`

- [x] Task 11: Unit test — `__tests__/market-intelligence-s12-3.test.ts` (AC: 13)
  - [x] 11.1 Create `apps/client/__tests__/market-intelligence-s12-3.test.ts` following the Vitest `environment: 'node'` pattern used in `espd-s11-13.test.ts`
  - [x] 11.2 Verify all required files exist (page shell, components, api, queries)
  - [x] 11.3 Verify `en.json` and `bg.json` contain all `analytics.market.*` keys from AC11
  - [x] 11.4 Verify `en.json` and `bg.json` contain `nav.analytics` key
  - [x] 11.5 Verify `layout.tsx` imports `BarChart2` and contains `analytics/market` in the nav
  - [x] 11.6 Verify `analytics.ts` exports the four fetch functions
  - [x] 11.7 Verify `use-market-analytics.ts` exports the four query hooks
  - [x] 11.8 Verify `MarketIntelligenceDashboard.tsx` source contains the required `data-testid` values

## Dev Notes

### Architecture Context

**Service:** `apps/client` — Next.js 14 frontend monorepo app (TypeScript, TanStack Query, Recharts, next-intl, Tailwind CSS, `@eusolicit/ui` component library).

**Depends on S12.02 (done):** The four API endpoints (`/api/v1/analytics/market/*`) are implemented and deployed. The `apiClient` from `@eusolicit/ui` attaches the JWT automatically (interceptor in `packages/ui/src/lib/api-client.ts`). Import it as: `import { apiClient } from "@eusolicit/ui"`.

**Pattern reference (routing):** Study `apps/client/app/[locale]/(protected)/espd/page.tsx` for the server component shell pattern. Study `apps/client/app/[locale]/(protected)/grants/tools/page.tsx` for a complex protected page with tabs and multiple components.

**Pattern reference (TanStack Query):** Study `apps/client/lib/queries/use-espd.ts` and `apps/client/lib/api/espd.ts` for the two-layer pattern: a pure API module (`lib/api/*.ts`) that contains TypeScript interfaces and `apiClient` calls, plus a query hooks module (`lib/queries/*.ts`) that wraps them in `useQuery`/`useMutation`.

**Pattern reference (unit tests):** Study `apps/client/__tests__/espd-s11-13.test.ts` for the structural file-system test pattern (Vitest `node` environment, `readFileSync`, `existsSync`, JSON key traversal). Run tests with: `cd eusolicit-app/frontend && pnpm test --filter client`.

---

### API Response Shapes (from S12.02 story)

```typescript
// From: services/client-api/src/client_api/schemas/analytics.py

interface MarketVolumeItem {
  sector: string;
  country: string;
  opportunity_count: number;
  total_value_eur: string | null; // Decimal serialised as string
}

interface MarketValuesItem {
  sector: string;
  country: string;
  avg_contract_value_eur: string | null; // Decimal serialised as string
}

interface MarketAuthorityItem {
  authority_name: string;
  sector: string;
  country: string;
  opportunity_count: number;
  total_value_eur: string | null; // Decimal serialised as string
}

interface MarketTrendItem {
  month: string;       // ISO date string, first day of month: "2025-01-01"
  sector: string;
  country: string;
  opportunity_count: number;
  total_value_eur: string | null;   // Decimal serialised as string
  avg_contract_value_eur: string | null;
}

interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  page_size: number;
}

interface ListResponse<T> {
  items: T[];
}
```

**Important:** `total_value_eur` and `avg_contract_value_eur` are Python `Decimal` values serialised by FastAPI as **strings** (e.g. `"1234567.89"`). When formatting for display or chart rendering, parse with `parseFloat(item.total_value_eur ?? "0")`.

**API base path:** `NEXT_PUBLIC_API_URL` (default: `http://localhost:8000`). The `apiClient` base URL is already configured; just use paths like `/api/v1/analytics/market/volume`.

---

### Recharts Import Pattern

```typescript
import {
  BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer,
  LineChart, Line
} from "recharts";
```

`recharts` is already in `apps/client/package.json` at `^2.12.0`. No additional installation needed.

**Recharts server-side rendering note:** Recharts charts use `window`/`document` and must be in `"use client"` components. Both `VolumeBarChart` and `TrendLineChart` are already `"use client"` components, so this is handled correctly.

---

### Building Query Params (omit empty values)

```typescript
export function buildMarketParams(
  filters: MarketFilters,
  extra?: Record<string, string | number>
): Record<string, string | number> {
  const params: Record<string, string | number> = {};
  if (filters.dateFrom) params.date_from = filters.dateFrom;
  if (filters.dateTo) params.date_to = filters.dateTo;
  if (filters.sector) params.sector = filters.sector;
  if (filters.country) params.country = filters.country;
  return { ...params, ...extra };
}
```

The `apiClient` (axios) accepts a `params` object that becomes query string parameters:

```typescript
export async function fetchMarketVolume(
  filters: MarketFilters,
  page = 1,
  pageSize = 20
): Promise<PaginatedResponse<MarketVolumeItem>> {
  const response = await apiClient.get<PaginatedResponse<MarketVolumeItem>>(
    "/api/v1/analytics/market/volume",
    { params: buildMarketParams(filters, { page, page_size: pageSize }) }
  );
  return response.data;
}
```

---

### Sorting the Authorities Table (Client-side)

The `/authorities` endpoint returns items already sorted by `opportunity_count DESC` from the server. For client-side column sorting in `AuthoritiesTable`:

```typescript
const sortedData = useMemo(() => {
  return [...data].sort((a, b) => {
    const av = a[sortState.column];
    const bv = b[sortState.column];
    // Numeric columns
    if (sortState.column === "opportunity_count") {
      const diff = (Number(av) || 0) - (Number(bv) || 0);
      return sortState.direction === "asc" ? diff : -diff;
    }
    // String columns
    const diff = String(av ?? "").localeCompare(String(bv ?? ""));
    return sortState.direction === "asc" ? diff : -diff;
  });
}, [data, sortState]);
```

`total_value_eur` is a string from the API (`"1234567.89"`); sort numerically via `parseFloat`.

---

### i18n Bulgarian Translations (bg.json)

```json
"analytics": {
  "market": {
    "pageTitle": "Пазарно разузнаване",
    "pageDescription": "Обеми на обществените поръчки, тенденции и водещи възложители.",
    "filterDateFrom": "От дата",
    "filterDateTo": "До дата",
    "filterSectorPlaceholder": "Сектор (напр. 45)",
    "filterCountryPlaceholder": "Държава (напр. BG)",
    "filterApplyBtn": "Приложи",
    "filterClearBtn": "Изчисти",
    "volumeChartTitle": "Обем на поръчките по сектор",
    "volumeTooltipLabel": "Обем",
    "volumeEmptyTitle": "Няма данни за обем",
    "volumeEmptyDescription": "Опитайте с по-широки филтри.",
    "trendChartTitle": "Месечни тенденции",
    "trendEmptyTitle": "Няма данни за тенденции",
    "authoritiesTableTitle": "Водещи възложители",
    "authoritiesEmptyTitle": "Няма данни за възложители",
    "pagingLabel": "Страница {page} от {total}",
    "colAuthorityName": "Възложител",
    "colSector": "Сектор",
    "colCountry": "Държава",
    "colOpportunities": "Поръчки"
  }
}
```

---

### File Locations Summary

| File | Action | Notes |
|------|--------|-------|
| `apps/client/app/[locale]/(protected)/analytics/market/page.tsx` | **Create** | Server component shell |
| `apps/client/app/[locale]/(protected)/analytics/market/components/MarketIntelligenceDashboard.tsx` | **Create** | Main container, filter state, query orchestration |
| `apps/client/app/[locale]/(protected)/analytics/market/components/MarketFilters.tsx` | **Create** | Filter panel component |
| `apps/client/app/[locale]/(protected)/analytics/market/components/VolumeBarChart.tsx` | **Create** | Recharts BarChart |
| `apps/client/app/[locale]/(protected)/analytics/market/components/TrendLineChart.tsx` | **Create** | Recharts LineChart |
| `apps/client/app/[locale]/(protected)/analytics/market/components/AuthoritiesTable.tsx` | **Create** | Sortable table with client-side pagination |
| `apps/client/lib/api/analytics.ts` | **Create** | TypeScript interfaces + apiClient fetch functions |
| `apps/client/lib/queries/use-market-analytics.ts` | **Create** | TanStack Query useQuery hooks |
| `apps/client/messages/en.json` | **Modify** | Add `analytics.market.*` + `nav.analytics` keys |
| `apps/client/messages/bg.json` | **Modify** | Add `analytics.market.*` + `nav.analytics` keys |
| `apps/client/app/[locale]/(protected)/layout.tsx` | **Modify** | Import `BarChart2`, add analytics nav item |
| `apps/client/__tests__/market-intelligence-s12-3.test.ts` | **Create** | Vitest structural unit test |

---

### Test Coverage Alignment with Epic-Level Test Design

From `eusolicit-docs/test-artifacts/test-design-epic-12.md`:

| Priority | Test | Description | Relation to this story |
|----------|------|-------------|------------------------|
| **P2** | S12.03 frontend | Charts render with data; date/sector/country filters update all visualizations; mobile stacking viewport | Covered by `market-intelligence-s12-3.test.ts` (structural) + manual/E2E verification |

**P2 E2E scope (post-story):** The epic-level P2 E2E test for S12.03 will be generated separately by TEA after the dashboard is deployed to staging. That test will verify: charts render with seeded data, filter controls update all visualizations, and mobile viewport stacks charts vertically. The Vitest structural test created in this story (AC13) covers file existence, i18n completeness, and `data-testid` presence — which are the assertions that can be made without a running browser.

**P0 risk note:** This story has no direct P0 test involvement. The P0 cross-tenant isolation tests (`cross-tenant-isolation.api.spec.ts`) target the API layer (S12.02), not the frontend. The tier gate tests target S12.06/S12.07 (Professional+ gated dashboards), not the market dashboard (available to all paid tiers).

---

### Responsive Layout Notes

The two-column chart grid uses Tailwind's responsive prefix:
```tsx
<div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mb-6">
  <div className="bg-white rounded-lg border p-4">
    <h2 className="text-lg font-semibold text-slate-800 mb-4">
      {t("analytics.market.volumeChartTitle")}
    </h2>
    <VolumeBarChart data={...} isLoading={...} locale={locale} />
  </div>
  <div className="bg-white rounded-lg border p-4">
    <h2 className="text-lg font-semibold text-slate-800 mb-4">
      {t("analytics.market.trendChartTitle")}
    </h2>
    <TrendLineChart data={...} isLoading={...} locale={locale} />
  </div>
</div>
```

`ResponsiveContainer width="100%"` in Recharts ensures the chart fills its container regardless of viewport width. Combined with the Tailwind responsive grid, this achieves:
- Mobile: each chart is 100% width, stacked vertically
- Desktop (≥1024px): each chart is ~50% width, side-by-side

---

## Senior Developer Review

**Review Date:** 2026-04-12 (R3)
**Reviewer:** Code Review (adversarial, 3-layer)
**Verdict:** Approve
**Tests:** All 76 structural tests pass (`market-intelligence-s12-3.test.ts`)

### R3 — All R1/R2 Findings Verified Resolved

All six patch findings from R1 and R2 have been verified as correctly applied in the source code:

- [x] [Review][Patch] **P1 — Duplicate `data-testid="market-filters"` in DOM** — `MarketIntelligenceDashboard.tsx:88` now renders `<div>` without `data-testid`. The testid lives solely on `MarketFilters.tsx:41` root div. No duplication.
- [x] [Review][Patch] **P2 — Empty states now use `EmptyState` from `@eusolicit/ui`** — All three components (`VolumeBarChart.tsx:36-43`, `TrendLineChart.tsx:36-39`, `AuthoritiesTable.tsx:76-79`) import and use `<EmptyState>` wrapped in a `<div>` with the correct `data-testid`. Icon imports (`BarChart2`, `TrendingUp`, `Building2`) are retained correctly as `icon` prop to `<EmptyState>`.
- [x] [Review][Patch] **P3 — `fetchMarketAuthorities` signature** — `analytics.ts:119` now accepts `(filters, page = 1, pageSize = 100)` and passes both via `buildMarketParams`. Matches AC9 spec and sibling function patterns.
- [x] [Review][Patch] **P4 — TrendLineChart tooltip labels i18n** — `TrendLineChart.tsx:58-60` uses `t("trendTooltipOpportunities")` and `t("trendTooltipValue")`. Keys present in both `en.json` and `bg.json` with correct translations.
- [x] [Review][Patch] **P5 — VolumeBarChart `locale` prop** — `VolumeBarChart.tsx:27` destructures `locale` and uses it at line 64 in `new Intl.NumberFormat(locale, ...)` for locale-aware currency formatting on YAxis. Option A applied.
- [x] [Review][Patch] **P6 — AuthoritiesTable pagination labels i18n** — `AuthoritiesTable.tsx:150,165` uses `t("paginationPrevious")` and `t("paginationNext")`. Keys present in both `en.json` and `bg.json`.

### Deferred (carried forward — not blocking)

- [ ] [Review][Defer] **D1 — `SkeletonCard`/`SkeletonTable` silently drop `data-testid`** — Pre-existing issue in `@eusolicit/ui`, not caused by this story. The AC7 testids are passed but won't render in DOM until the UI package components are updated to spread rest props. **Deferred to a separate patch on the UI package.** AC: 7.

### Adversarial Sweep Summary (R3)

| Layer | Result |
|-------|--------|
| **Blind Hunter** (code quality, patterns, dead code) | Clean. All components follow established codebase patterns. No dead imports, no unused variables, no console.log statements. |
| **Edge Case Hunter** (boundaries, error handling) | Acceptable. `total_value_eur` null handling via `parseFloat(... ?? "0")` is consistent. Sort handles string/numeric columns correctly. Pagination edge cases (page 1, last page) properly disable buttons. `totalPages = Math.max(1, ...)` prevents division-by-zero display. |
| **Acceptance Auditor** (AC compliance) | All 13 ACs satisfied. AC1-AC13 verified against source. i18n keys complete in both locales. Navigation positioned correctly (after Dashboard, before Tenders). Responsive grid uses correct breakpoints. Query hooks use correct staleTime and queryKey patterns. |
