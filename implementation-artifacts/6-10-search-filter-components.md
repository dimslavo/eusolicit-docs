# Story 6.10: Search & Filter Components

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user discovering EU procurement opportunities**,
I want **a filter sidebar with CPV code multi-select, region checkboxes, budget range, deadline range, status toggles, and opportunity type selector — with active filter chips, URL-persisted filter state, and a mobile bottom sheet**,
so that **I can precisely narrow the opportunity listing to match my procurement focus area, share filtered views via URL, and efficiently filter on any device**.

## Acceptance Criteria

1. `FilterSidebar.tsx` renders as a side panel integrated into `OpportunitiesListPage` at `/opportunities`; on screens ≥ `lg` breakpoint (`hidden lg:block`) it is permanently visible alongside results; contains 6 labelled filter sections: CPV Codes, Regions, Budget Range, Deadline Range, Status, Opportunity Type.

2. **CPV Code multi-select**: `CPVCodeFilter.tsx` renders with a search input (`data-testid="cpv-filter-search"`) that filters the static `CPV_CODES` list (from `@/lib/data/cpv-codes`) client-side; results shown in a scrollable list (max-height limited, `ScrollArea`) with a `<Checkbox>` per CPV entry; selected codes appear as dismissible `<Badge>` chips inside the filter component; multi-select (multiple CPV codes can be active simultaneously); when no CPV codes match the search query the list shows a "No results" message.

3. **Region checkboxes**: `RegionFilter.tsx` renders a `<Checkbox>` per EU country from a new shared data file `@/lib/data/eu-countries.ts`; each checkbox label shows the country's flag emoji and name; a "Select all" / "Deselect all" button (`data-testid="region-select-all"`) at the top of the section toggles all checkboxes.

4. **Budget range**: `BudgetRangeFilter.tsx` renders two `<Input type="number">` fields — min EUR (`data-testid="budget-min-input"`) and max EUR (`data-testid="budget-max-input"`); both fields have EUR prefix label; when both values are set, `budget_min` must be ≤ `budget_max` (an inline validation error message is shown otherwise, and the invalid value is NOT written to the URL); 500 ms debounce before writing to URL params.

5. **Deadline range**: `DeadlineRangeFilter.tsx` renders two `<Input type="date">` fields — from (`data-testid="deadline-from-input"`) and to (`data-testid="deadline-to-input"`); `deadline_from` must be ≤ `deadline_to` when both are set (inline validation error shown otherwise); URL params update immediately on field blur/change (no debounce needed for date pickers).

6. **Status toggle buttons**: `StatusFilter.tsx` renders three `<Button>` toggle components: Open (`data-testid="status-btn-open"`), Closing Soon (`data-testid="status-btn-closing_soon"`), Closed (`data-testid="status-btn-closed"`); clicking a button sets `?status=` URL param; clicking the active button clears the param; only one status can be active at a time (matches the backend `status` enum from S06.01).

7. **Opportunity type selector**: `OpportunityTypeFilter.tsx` renders four `<Button>` toggle components: Works, Services, Supplies, Mixed; clicking sets `?opportunity_type=` URL param; only one type active at a time (matches the backend enum); all four buttons have `data-testid="type-btn-{value}"` where value is `works | services | supplies | mixed`.

8. **Filter application & debouncing**: CPV, region, status, and type filter changes update URL params immediately (driving re-fetch via `useOpportunityListing`); budget inputs debounce at 500 ms; deadline inputs update on blur; any filter change also clears `after_cursor` from URL params to reset pagination to page 1.

9. **Active filter chips**: `ActiveFilterChips.tsx` renders above the results area (`data-testid="active-filter-chips"`); each active filter category generates one or more chips — each chip shows a label and an `<X>` dismiss button (`data-testid="chip-remove-{category}"`) that clears that specific filter; a "Clear all filters" `<Button>` (`data-testid="filter-clear-all"`) appears when ≥ 1 filter is active and clears all filter URL params at once; when no filters are active the chip container renders but is empty (not hidden, preserving layout).

10. **Filter state in URL**: All filter values persist as URL query params (`?cpv_codes=`, `?regions=`, `?budget_min=`, `?budget_max=`, `?deadline_from=`, `?deadline_to=`, `?status=`, `?opportunity_type=`); navigating to a URL with these params pre-populates all filter components so the filtered view is fully shareable; existing URL params from S06.09 (`q`, `sort_by`, `sort_order`, `view`) continue to work unchanged alongside filter params.

11. **Mobile bottom sheet**: `MobileFilterSheet.tsx` renders a "Filter" trigger button (`data-testid="filter-mobile-trigger"`) visible on `< lg` viewports (`lg:hidden`); the button shows an active-filter count badge when ≥ 1 filter is active; clicking opens a `<Sheet>` (shadcn/ui `side="bottom"`) containing the full `<FilterSidebar>`; on `lg+` screens, `FilterSidebar` is rendered inline (not inside the Sheet) so filters are never duplicated in the DOM.

12. All UI strings use `useTranslations("opportunities")`; all new filter translation keys (see Dev Notes: i18n Keys section) exist in both `messages/en.json` and `messages/bg.json` under the `opportunities` namespace.

13. All filter components and interactive elements have `data-testid` attributes as specified in Dev Notes (testid reference table).

14. `OpportunityListParams` interface in `lib/api/opportunities.ts` is extended with all filter fields; `useOpportunityListing` in `lib/queries/use-opportunities.ts` passes them through to the API calls unchanged.

15. ATDD test file `__tests__/opportunities-filter-s6-10.test.ts` covers file structure, data-testid presence in source, i18n key completeness, and `OpportunityListParams` interface extension; all tests pass GREEN.

## Tasks / Subtasks

- [x] Task 1: Create `lib/data/eu-countries.ts` shared data file (AC: 3)
  - [x] 1.1 Define and export `EuCountry` interface: `{ code: string; name: string; flag: string }`
  - [x] 1.2 Export `EU_COUNTRIES: EuCountry[]` — all 27 EU member states with ISO-3166-1 alpha-2 codes, English names, and flag emoji; extract the pattern from `ConsortiumFinderPanel.tsx` and expand to a named-export constant (do not modify ConsortiumFinderPanel)
  - [x] 1.3 Export `EU_COUNTRY_MAP: Record<string, EuCountry>` keyed by country code for O(1) lookup

- [x] Task 2: Extend `OpportunityListParams` in `lib/api/opportunities.ts` (AC: 14)
  - [x] 2.1 Add optional filter fields to the `OpportunityListParams` interface (see Dev Notes: API Contract)
  - [x] 2.2 Verify `listOpportunities` and `searchOpportunities` pass `params` as-is via `apiClient.get` — no changes needed to function bodies (interface extension is sufficient)

- [x] Task 3: Add i18n translation keys (AC: 12)
  - [x] 3.1 Add all new filter keys under `opportunities` namespace to `messages/en.json` (see Dev Notes: i18n Keys section for the full key list)
  - [x] 3.2 Add matching translated keys to `messages/bg.json`
  - [x] 3.3 Run `pnpm check:i18n` and verify it passes

- [x] Task 4: Create `ActiveFilterChips.tsx` (AC: 9)
  - [x] 4.1 Accept props: `filters: FilterState` and `onClearFilter(category: FilterCategory): void` and `onClearAll(): void`
  - [x] 4.2 Render a chip for each active filter: CPV (one chip per code), Region (one per country), Budget (single chip "€X – €Y"), Deadline (single chip "From X to Y"), Status (single chip), Type (single chip)
  - [x] 4.3 Each chip uses `<Badge>` with an `<X>` icon button; clicking calls `onClearFilter` with the appropriate category
  - [x] 4.4 "Clear all" button only visible when `activeFilterCount > 0`; `data-testid="filter-clear-all"`
  - [x] 4.5 Container: `data-testid="active-filter-chips"`; each chip: `data-testid="chip-{category}-{value}"` where category is `cpv|region|budget|deadline|status|type`; remove button: `data-testid="chip-remove-{category}"`

- [x] Task 5: Create `CPVCodeFilter.tsx` (AC: 2)
  - [x] 5.1 Accept props: `selectedCodes: string[]` and `onChange(codes: string[]): void`
  - [x] 5.2 Local `searchQuery` state (string); filter `CPV_CODES` (from `@/lib/data/cpv-codes`) by `searchQuery` (case-insensitive match on `code` or `description`); show max 50 results in `<ScrollArea className="h-48">`
  - [x] 5.3 Each result row: `<Checkbox>` checked when code is in `selectedCodes`; clicking calls `onChange` with updated array (add or remove); label shows code + description truncated to 40 chars
  - [x] 5.4 Selected codes appear as dismissible `<Badge>` chips above the search input; clicking `X` on chip removes that code via `onChange`
  - [x] 5.5 Section header "CPV Codes" with count badge when codes selected; "No CPV codes found" empty state when search yields no results
  - [x] 5.6 `data-testid="cpv-filter-search"` on search input; `data-testid="cpv-filter-list"` on the scrollable list container; each row `data-testid="cpv-option-{code}"`; selected chips container `data-testid="cpv-selected-chips"`

- [x] Task 6: Create `RegionFilter.tsx` (AC: 3)
  - [x] 6.1 Accept props: `selectedRegions: string[]` and `onChange(regions: string[]): void`
  - [x] 6.2 Render a `<Checkbox>` for each country in `EU_COUNTRIES` (from `@/lib/data/eu-countries`); label shows `{flag} {name}`; checked when `name` is in `selectedRegions`; clicking toggles the country in the array
  - [x] 6.3 "Select all" / "Deselect all" button at top: `data-testid="region-select-all"`; label changes based on whether all are selected
  - [x] 6.4 Render in a `<ScrollArea className="h-56">` for scrollability
  - [x] 6.5 Each checkbox: `data-testid="region-checkbox-{code}"` where code is the ISO country code

- [x] Task 7: Create `BudgetRangeFilter.tsx` (AC: 4)
  - [x] 7.1 Accept props: `budgetMin: number | null`, `budgetMax: number | null`, `onChange(min: number | null, max: number | null): void`
  - [x] 7.2 Two `<Input type="number" min="0">` fields with "EUR" prefix labels; local state tracks input values; 500 ms debounce before calling `onChange`
  - [x] 7.3 Validation: when both set and `min > max`, show inline error `<p className="text-destructive text-xs">` and do NOT call `onChange`
  - [x] 7.4 `data-testid="budget-min-input"` and `data-testid="budget-max-input"`; error: `data-testid="budget-range-error"`

- [x] Task 8: Create `DeadlineRangeFilter.tsx` (AC: 5)
  - [x] 8.1 Accept props: `deadlineFrom: string | null`, `deadlineTo: string | null`, `onChange(from: string | null, to: string | null): void`
  - [x] 8.2 Two `<Input type="date">` fields; call `onChange` immediately on change (no debounce for date pickers)
  - [x] 8.3 Validation: when both set and `from > to`, show inline error and do NOT call `onChange`
  - [x] 8.4 `data-testid="deadline-from-input"` and `data-testid="deadline-to-input"`; error: `data-testid="deadline-range-error"`

- [x] Task 9: Create `StatusFilter.tsx` (AC: 6)
  - [x] 9.1 Accept props: `status: "open" | "closing_soon" | "closed" | null`, `onChange(status: "open" | "closing_soon" | "closed" | null): void`
  - [x] 9.2 Three `<Button variant="outline">` components; active button uses `variant="default"` (or custom `bg-primary text-primary-foreground` classes)
  - [x] 9.3 Clicking the already-active button calls `onChange(null)` (deselect); clicking a different button calls `onChange(value)`
  - [x] 9.4 `data-testid="status-btn-open"`, `data-testid="status-btn-closing_soon"`, `data-testid="status-btn-closed"`

- [x] Task 10: Create `OpportunityTypeFilter.tsx` (AC: 7)
  - [x] 10.1 Accept props: `opportunityType: string | null`, `onChange(type: string | null): void`
  - [x] 10.2 Four `<Button variant="outline">` components: Works, Services, Supplies, Mixed
  - [x] 10.3 Active button uses `variant="default"`; clicking active button deselects (`onChange(null)`)
  - [x] 10.4 `data-testid="type-btn-works"`, `data-testid="type-btn-services"`, `data-testid="type-btn-supplies"`, `data-testid="type-btn-mixed"`

- [x] Task 11: Create `FilterSidebar.tsx` (AC: 1, 8)
  - [x] 11.1 Accept props: `filters: FilterState`, `onFilterChange(partial: Partial<FilterState>): void`
  - [x] 11.2 Render all 6 filter sections in vertical stack, each in a collapsible `<details>`/`<summary>` or always-open card section; sections separated by `<Separator />`
  - [x] 11.3 Section order: CPV Codes → Regions → Budget Range → Deadline Range → Status → Opportunity Type
  - [x] 11.4 Each section heading shows the filter name and (when active) a count badge
  - [x] 11.5 `data-testid="filter-sidebar"` on root; `data-testid="filter-section-{name}"` per section (cpv, regions, budget, deadline, status, type)

- [x] Task 12: Create `MobileFilterSheet.tsx` (AC: 11)
  - [x] 12.1 Accept props: `filters: FilterState`, `onFilterChange(partial: Partial<FilterState>): void`, `activeFilterCount: number`
  - [x] 12.2 Trigger `<Button variant="outline" className="lg:hidden">` with `<Filter />` (Lucide icon) + "Filter" label + count badge when `activeFilterCount > 0`; `data-testid="filter-mobile-trigger"`
  - [x] 12.3 `<Sheet>` with `side="bottom"` containing `<SheetContent>`; inside renders full `<FilterSidebar>` with same props
  - [x] 12.4 `<SheetHeader>` shows "Filters" title + "Close" button

- [x] Task 13: Modify `OpportunitiesListPage.tsx` to integrate filter sidebar (AC: 1, 8, 9, 10, 11)
  - [x] 13.1 Read all filter URL params from `useSearchParams()`: `cpv_codes`, `regions`, `budget_min`, `budget_max`, `deadline_from`, `deadline_to`, `status`, `opportunity_type`; parse using `readFilterState()` helper (see Dev Notes)
  - [x] 13.2 Parse filter values into `FilterState` object; pass all fields (including filter fields) to `useOpportunityListing` via extended `OpportunityListParams`
  - [x] 13.3 Implement `handleFilterChange(partial: Partial<FilterState>)` that merges partial into current filter state, removes `after_cursor` from params (pagination reset), and calls `router.replace`
  - [x] 13.4 Compute `activeFilterCount` as sum of active filter categories
  - [x] 13.5 Layout restructure: wrap existing content in `<div className="flex gap-6">` with `<aside className="hidden lg:block w-72 shrink-0">` for desktop sidebar and `<div className="flex-1 min-w-0">` for main content
  - [x] 13.6 Render `<ActiveFilterChips>` above the search bar + view toggle row
  - [x] 13.7 Render `<MobileFilterSheet>` (visible on `< lg`) as a fixed/sticky trigger (below search bar area)
  - [x] 13.8 Pass `filters` and `onFilterChange` (and `activeFilterCount`) to `<FilterSidebar>`, `<MobileFilterSheet>`, and `<ActiveFilterChips>`

- [x] Task 14: Create ATDD test file `__tests__/opportunities-filter-s6-10.test.ts` (AC: 15)
  - [x] 14.1 AC1/AC11: assert `FilterSidebar.tsx`, `MobileFilterSheet.tsx`, `ActiveFilterChips.tsx` exist in `components/`
  - [x] 14.2 AC2: assert `CPVCodeFilter.tsx` exists; contains `data-testid="cpv-filter-search"` and `data-testid="cpv-filter-list"` and `CPV_CODES` import
  - [x] 14.3 AC3: assert `RegionFilter.tsx` exists; contains `data-testid="region-select-all"` and `EU_COUNTRIES` import
  - [x] 14.4 AC4/AC5: assert `BudgetRangeFilter.tsx` contains `budget-min-input`, `budget-max-input` testids; assert `DeadlineRangeFilter.tsx` contains `deadline-from-input`, `deadline-to-input` testids
  - [x] 14.5 AC6: assert `StatusFilter.tsx` contains `status-btn-open`, `status-btn-closing_soon`, `status-btn-closed` testids
  - [x] 14.6 AC7: assert `OpportunityTypeFilter.tsx` contains `type-btn-works`, `type-btn-services`, `type-btn-supplies`, `type-btn-mixed` testids
  - [x] 14.7 AC9: assert `ActiveFilterChips.tsx` contains `active-filter-chips` and `filter-clear-all` testids
  - [x] 14.8 AC10: assert `OpportunitiesListPage.tsx` reads `cpv_codes`, `regions`, `budget_min`, `budget_max`, `deadline_from`, `deadline_to`, `status`, `opportunity_type` from `useSearchParams()` and passes through
  - [x] 14.9 AC11: assert `MobileFilterSheet.tsx` contains `filter-mobile-trigger` testid and `Sheet` import
  - [x] 14.10 AC12: assert all new filter i18n keys exist in both `en.json` and `bg.json`
  - [x] 14.11 AC14: assert `lib/api/opportunities.ts` contains `cpv_codes`, `budget_min`, `budget_max`, `deadline_from`, `deadline_to`, `status`, `opportunity_type` fields in `OpportunityListParams`
  - [x] 14.12 AC1/AC10: assert `lib/data/eu-countries.ts` exists and exports `EU_COUNTRIES`

## Dev Notes

### Architecture Overview

S06.10 is a pure frontend story that extends the S06.09 listing page with a filter sidebar. All backend filter params are already supported by `GET /api/v1/opportunities` and `GET /api/v1/opportunities/search` (S06.01). This story only adds the frontend UI layer that writes filter values into the existing URL params the backend already accepts.

**Data sources:**
- **CPV codes**: Static file `@/lib/data/cpv-codes.ts` (already exists, ~35 top-level divisions); no API call needed.
- **EU countries**: New shared static file `@/lib/data/eu-countries.ts` (27 EU member states); no API call needed.
- **Filter options (status, type)**: Hardcoded constants (enums from S06.01).

**State management pattern**: All filter state lives in URL query params (same as `sort_by`, `sort_order`, `view`, `q` in S06.09). No new Zustand stores. `FilterSidebar` receives current filter state as props from `OpportunitiesListPage` and calls `onFilterChange` to request URL updates.

### Extended `OpportunityListParams` Interface

In `lib/api/opportunities.ts`, extend the existing interface:

```typescript
export interface OpportunityListParams {
  // existing (S06.09)
  sort_by?: "deadline" | "relevance_score" | "budget" | "published_date";
  sort_order?: "asc" | "desc";
  after_cursor?: string;
  before_cursor?: string;
  limit?: number;
  q?: string;
  // NEW (S06.10)
  cpv_codes?: string;          // comma-separated CPV codes, e.g. "45000000,72000000"
  regions?: string;             // comma-separated EU country names, e.g. "Bulgaria,Germany"
  budget_min?: number;
  budget_max?: number;
  deadline_from?: string;       // ISO date YYYY-MM-DD
  deadline_to?: string;         // ISO date YYYY-MM-DD
  status?: "open" | "closing_soon" | "closed";
  opportunity_type?: "works" | "services" | "supplies" | "mixed";
}
```

No changes to `listOpportunities` or `searchOpportunities` function bodies — `apiClient.get` passes `params` through as query string automatically.

### `FilterState` Type and URL Helpers

Define these in `opportunity-utils.tsx` (already exists — add to it):

```typescript
export interface FilterState {
  cpvCodes: string[];
  regions: string[];
  budgetMin: number | null;
  budgetMax: number | null;
  deadlineFrom: string | null;    // YYYY-MM-DD
  deadlineTo: string | null;      // YYYY-MM-DD
  status: "open" | "closing_soon" | "closed" | null;
  opportunityType: "works" | "services" | "supplies" | "mixed" | null;
}

export type FilterCategory = keyof FilterState;

export function EMPTY_FILTERS(): FilterState {
  return {
    cpvCodes: [], regions: [],
    budgetMin: null, budgetMax: null,
    deadlineFrom: null, deadlineTo: null,
    status: null, opportunityType: null,
  };
}

/** Parse URL search params into FilterState */
export function readFilterState(searchParams: ReadonlyURLSearchParams): FilterState {
  return {
    cpvCodes: searchParams.get("cpv_codes")?.split(",").filter(Boolean) ?? [],
    regions: searchParams.get("regions")?.split(",").filter(Boolean) ?? [],
    budgetMin: searchParams.get("budget_min") ? Number(searchParams.get("budget_min")) : null,
    budgetMax: searchParams.get("budget_max") ? Number(searchParams.get("budget_max")) : null,
    deadlineFrom: searchParams.get("deadline_from") ?? null,
    deadlineTo: searchParams.get("deadline_to") ?? null,
    status: (searchParams.get("status") as FilterState["status"]) ?? null,
    opportunityType: (searchParams.get("opportunity_type") as FilterState["opportunityType"]) ?? null,
  };
}

/** Write FilterState into URLSearchParams, deleting empty values */
export function writeFilterToParams(params: URLSearchParams, filters: Partial<FilterState>): void {
  const set = (key: string, value: string | null) =>
    value ? params.set(key, value) : params.delete(key);
  if ("cpvCodes" in filters)
    set("cpv_codes", filters.cpvCodes?.length ? filters.cpvCodes.join(",") : null);
  if ("regions" in filters)
    set("regions", filters.regions?.length ? filters.regions.join(",") : null);
  if ("budgetMin" in filters)
    set("budget_min", filters.budgetMin != null ? String(filters.budgetMin) : null);
  if ("budgetMax" in filters)
    set("budget_max", filters.budgetMax != null ? String(filters.budgetMax) : null);
  if ("deadlineFrom" in filters) set("deadline_from", filters.deadlineFrom ?? null);
  if ("deadlineTo" in filters) set("deadline_to", filters.deadlineTo ?? null);
  if ("status" in filters) set("status", filters.status ?? null);
  if ("opportunityType" in filters) set("opportunity_type", filters.opportunityType ?? null);
}

/** Count how many filter categories have at least one active value */
export function countActiveFilters(filters: FilterState): number {
  return [
    filters.cpvCodes.length > 0,
    filters.regions.length > 0,
    filters.budgetMin != null || filters.budgetMax != null,
    filters.deadlineFrom != null || filters.deadlineTo != null,
    filters.status != null,
    filters.opportunityType != null,
  ].filter(Boolean).length;
}
```

### `OpportunitiesListPage.tsx` Changes

In `handleFilterChange`, always clear `after_cursor` when any filter changes (pagination reset):

```typescript
function handleFilterChange(partial: Partial<FilterState>) {
  const params = new URLSearchParams(searchParamsRef.current.toString());
  params.delete("after_cursor");   // reset pagination on filter change
  writeFilterToParams(params, partial);
  router.replace(`${pathname}?${params.toString()}`);
}

function handleClearFilter(category: FilterCategory) {
  const reset: Partial<FilterState> = {};
  switch (category) {
    case "cpvCodes":       reset.cpvCodes = []; break;
    case "regions":        reset.regions = []; break;
    case "budgetMin":      reset.budgetMin = null; break;
    case "budgetMax":      reset.budgetMax = null; break;
    case "deadlineFrom":   reset.deadlineFrom = null; break;
    case "deadlineTo":     reset.deadlineTo = null; break;
    case "status":         reset.status = null; break;
    case "opportunityType": reset.opportunityType = null; break;
  }
  handleFilterChange(reset);
}

function handleClearAll() {
  handleFilterChange(EMPTY_FILTERS());
}
```

Pass filter values to `useOpportunityListing`:

```typescript
const filters = readFilterState(searchParams);

const { data, isLoading, ... } = useOpportunityListing({
  sort_by: sortBy,
  sort_order: sortOrder,
  q: qParam || undefined,
  cpv_codes: filters.cpvCodes.length ? filters.cpvCodes.join(",") : undefined,
  regions: filters.regions.length ? filters.regions.join(",") : undefined,
  budget_min: filters.budgetMin ?? undefined,
  budget_max: filters.budgetMax ?? undefined,
  deadline_from: filters.deadlineFrom ?? undefined,
  deadline_to: filters.deadlineTo ?? undefined,
  status: filters.status ?? undefined,
  opportunity_type: filters.opportunityType ?? undefined,
});
```

### Layout Restructure in `OpportunitiesListPage.tsx`

```tsx
return (
  <div className="space-y-4">
    {/* Search + view toggle row */}
    <div className="flex items-center gap-3">
      {/* MobileFilterSheet trigger — visible on < lg */}
      <MobileFilterSheet
        filters={filters}
        onFilterChange={handleFilterChange}
        activeFilterCount={activeFilterCount}
      />
      {/* existing search input */}
      <Input ... />
      {/* existing view toggle */}
    </div>

    {/* Active filter chips */}
    <ActiveFilterChips
      filters={filters}
      onClearFilter={handleClearFilter}
      onClearAll={handleClearAll}
    />

    {/* Two-column layout: sidebar (lg+) + results */}
    <div className="flex gap-6 items-start">
      {/* Desktop sidebar — hidden on mobile */}
      <aside className="hidden lg:block w-72 shrink-0">
        <FilterSidebar
          filters={filters}
          onFilterChange={handleFilterChange}
        />
      </aside>

      {/* Results area */}
      <div className="flex-1 min-w-0">
        {/* existing: loading / empty / table / card view + load more */}
      </div>
    </div>
  </div>
);
```

**Do NOT render `<FilterSidebar>` inside both the `<aside>` and `<MobileFilterSheet>`** — this would duplicate the DOM. Only render it inside the Sheet on mobile. On desktop, render it in the `<aside>`. Use conditional rendering:

```tsx
{/* Desktop sidebar */}
<aside className="hidden lg:block w-72 shrink-0">
  <FilterSidebar filters={filters} onFilterChange={handleFilterChange} />
</aside>

{/* Mobile: Sheet trigger + Sheet (FilterSidebar rendered inside Sheet only) */}
<MobileFilterSheet
  filters={filters}
  onFilterChange={handleFilterChange}
  activeFilterCount={activeFilterCount}
/>
```

The `MobileFilterSheet` renders its trigger button as `lg:hidden` and the Sheet itself is only shown when the trigger is clicked — so `FilterSidebar` is in the Sheet's DOM but not visible unless opened. This avoids hydration issues while keeping a single instance of the sidebar at a time.

### `eu-countries.ts` Data File

```typescript
// lib/data/eu-countries.ts
export interface EuCountry {
  code: string;   // ISO-3166-1 alpha-2
  name: string;   // English name
  flag: string;   // Flag emoji
}

export const EU_COUNTRIES: EuCountry[] = [
  { code: "AT", name: "Austria", flag: "🇦🇹" },
  { code: "BE", name: "Belgium", flag: "🇧🇪" },
  { code: "BG", name: "Bulgaria", flag: "🇧🇬" },
  { code: "HR", name: "Croatia", flag: "🇭🇷" },
  { code: "CY", name: "Cyprus", flag: "🇨🇾" },
  { code: "CZ", name: "Czech Republic", flag: "🇨🇿" },
  { code: "DK", name: "Denmark", flag: "🇩🇰" },
  { code: "EE", name: "Estonia", flag: "🇪🇪" },
  { code: "FI", name: "Finland", flag: "🇫🇮" },
  { code: "FR", name: "France", flag: "🇫🇷" },
  { code: "DE", name: "Germany", flag: "🇩🇪" },
  { code: "GR", name: "Greece", flag: "🇬🇷" },
  { code: "HU", name: "Hungary", flag: "🇭🇺" },
  { code: "IE", name: "Ireland", flag: "🇮🇪" },
  { code: "IT", name: "Italy", flag: "🇮🇹" },
  { code: "LV", name: "Latvia", flag: "🇱🇻" },
  { code: "LT", name: "Lithuania", flag: "🇱🇹" },
  { code: "LU", name: "Luxembourg", flag: "🇱🇺" },
  { code: "MT", name: "Malta", flag: "🇲🇹" },
  { code: "NL", name: "Netherlands", flag: "🇳🇱" },
  { code: "PL", name: "Poland", flag: "🇵🇱" },
  { code: "PT", name: "Portugal", flag: "🇵🇹" },
  { code: "RO", name: "Romania", flag: "🇷🇴" },
  { code: "SK", name: "Slovakia", flag: "🇸🇰" },
  { code: "SI", name: "Slovenia", flag: "🇸🇮" },
  { code: "ES", name: "Spain", flag: "🇪🇸" },
  { code: "SE", name: "Sweden", flag: "🇸🇪" },
];

export const EU_COUNTRY_MAP: Record<string, EuCountry> =
  Object.fromEntries(EU_COUNTRIES.map(c => [c.code, c]));
```

### CPV Filter Pattern (Reference: `Step2CPVSectors.tsx`)

The existing `Step2CPVSectors.tsx` already implements a near-identical CPV multi-select pattern. **Reference it directly** when building `CPVCodeFilter.tsx`:
- Path: `app/[locale]/(protected)/setup/components/Step2CPVSectors.tsx`
- Key differences from that component: `CPVCodeFilter` is controlled (props-driven, no local selection state), uses `onChange` callback, and is designed for sidebar width constraints

```tsx
// CPVCodeFilter.tsx simplified pattern
"use client";
import { useState } from "react";
import { CPV_CODES } from "@/lib/data/cpv-codes";
import { Input, Checkbox, Badge, ScrollArea } from "@eusolicit/ui";
import { X } from "lucide-react";

export function CPVCodeFilter({
  selectedCodes,
  onChange,
}: {
  selectedCodes: string[];
  onChange: (codes: string[]) => void;
}) {
  const [searchQuery, setSearchQuery] = useState("");
  const filtered = CPV_CODES.filter(c =>
    !searchQuery ||
    c.code.includes(searchQuery) ||
    c.description.toLowerCase().includes(searchQuery.toLowerCase())
  ).slice(0, 50);

  return (
    <div className="space-y-2" data-testid="cpv-filter-section">
      {/* Selected chips */}
      {selectedCodes.length > 0 && (
        <div className="flex flex-wrap gap-1" data-testid="cpv-selected-chips">
          {selectedCodes.map(code => (
            <Badge key={code} variant="secondary" className="text-xs">
              {code}
              <button onClick={() => onChange(selectedCodes.filter(c => c !== code))}>
                <X className="h-3 w-3 ml-1" />
              </button>
            </Badge>
          ))}
        </div>
      )}
      {/* Search */}
      <Input
        data-testid="cpv-filter-search"
        value={searchQuery}
        onChange={e => setSearchQuery(e.target.value)}
        placeholder="Search CPV codes..."
        className="h-8 text-sm"
      />
      {/* List */}
      <ScrollArea className="h-48" data-testid="cpv-filter-list">
        {filtered.length === 0 ? (
          <p className="text-sm text-muted-foreground p-2">No CPV codes found</p>
        ) : (
          <ul className="space-y-1 p-1">
            {filtered.map(cpv => (
              <li key={cpv.code} data-testid={`cpv-option-${cpv.code}`}
                  className="flex items-start gap-2 p-1 rounded hover:bg-muted cursor-pointer"
                  onClick={() => {
                    const isSelected = selectedCodes.includes(cpv.code);
                    onChange(isSelected
                      ? selectedCodes.filter(c => c !== cpv.code)
                      : [...selectedCodes, cpv.code]
                    );
                  }}>
                <Checkbox
                  checked={selectedCodes.includes(cpv.code)}
                  className="mt-0.5 shrink-0"
                  readOnly
                />
                <span className="text-xs leading-tight">
                  <span className="font-mono text-slate-500 mr-1">{cpv.code}</span>
                  <span className="line-clamp-2">{cpv.description}</span>
                </span>
              </li>
            ))}
          </ul>
        )}
      </ScrollArea>
    </div>
  );
}
```

### `FilterSidebar.tsx` Section Structure

Each filter section wraps in a collapsible pattern. Use a simple `useState(true)` to track open/closed state per section (not `<details>` — React state is more predictable with dynamic content):

```tsx
function FilterSection({
  title, children, activeCount, testId
}: { title: string; children: React.ReactNode; activeCount: number; testId: string }) {
  const [open, setOpen] = useState(true);
  return (
    <div data-testid={`filter-section-${testId}`}>
      <button
        className="flex items-center justify-between w-full py-2 text-sm font-medium"
        onClick={() => setOpen(!open)}
      >
        <span>{title}</span>
        <div className="flex items-center gap-1">
          {activeCount > 0 && (
            <Badge variant="secondary" className="h-5 text-xs">{activeCount}</Badge>
          )}
          <ChevronDown className={cn("h-4 w-4 transition-transform", open && "rotate-180")} />
        </div>
      </button>
      {open && <div className="pb-3">{children}</div>}
      <Separator />
    </div>
  );
}
```

### `ActiveFilterChips.tsx` Budget and Deadline Chip Labels

Budget chip: `"€{min} – €{max}"`, `"≥ €{min}"` (when only min), `"≤ €{max}"` (when only max).
Deadline chip: `"{from} → {to}"`, `"From {from}"`, `"Until {to}"`.

For CPV chips: show the code (e.g., "45000000") since descriptions are long.
For Region chips: show country name.
For Status/Type chips: show human-readable label (Open, Closing Soon, etc.).

### `MobileFilterSheet` Import Pattern

`Sheet`, `SheetContent`, `SheetHeader`, `SheetTitle` are all from `@eusolicit/ui` (re-exported from `sheet.tsx` in the packages/ui). The `Filter` icon is from `lucide-react`:

```tsx
import { Sheet, SheetContent, SheetHeader, SheetTitle } from "@eusolicit/ui";
import { Filter } from "lucide-react";
```

### File Structure

```
eusolicit-app/frontend/apps/client/
├── lib/
│   ├── data/
│   │   └── eu-countries.ts                        # NEW — EU country data
│   └── api/
│       └── opportunities.ts                        # MODIFIED — extend OpportunityListParams
├── app/[locale]/(protected)/opportunities/
│   └── components/
│       ├── OpportunitiesListPage.tsx               # MODIFIED — integrate filter sidebar
│       ├── opportunity-utils.tsx                   # MODIFIED — add FilterState, helpers
│       ├── FilterSidebar.tsx                       # NEW — sidebar container
│       ├── ActiveFilterChips.tsx                   # NEW — removable chips row
│       ├── CPVCodeFilter.tsx                       # NEW — CPV multi-select
│       ├── RegionFilter.tsx                        # NEW — EU region checkboxes
│       ├── BudgetRangeFilter.tsx                   # NEW — budget min/max inputs
│       ├── DeadlineRangeFilter.tsx                 # NEW — deadline date range
│       ├── StatusFilter.tsx                        # NEW — status toggle buttons
│       ├── OpportunityTypeFilter.tsx               # NEW — type toggle buttons
│       └── MobileFilterSheet.tsx                   # NEW — mobile bottom sheet wrapper
├── messages/
│   ├── en.json                                     # MODIFIED — add filter keys
│   └── bg.json                                     # MODIFIED — add filter keys
└── __tests__/
    └── opportunities-filter-s6-10.test.ts          # NEW — ATDD tests
```

**Do NOT modify:**
- `lib/queries/use-opportunities.ts` — already passes `params` through; interface extension is sufficient
- `lib/data/cpv-codes.ts` — use as-is; do not add API fetching
- `ConsortiumFinderPanel.tsx` — do not refactor it to use the new `eu-countries.ts`; that's out of scope

### i18n Keys Required

Add the following under the existing `opportunities` namespace in both `en.json` and `bg.json`:

```json
{
  "filterTitle": "Filters",
  "filterButton": "Filter",
  "filterClear": "Clear all",
  "filterNoActive": "No active filters",
  "filterActiveCount": "{count} active",
  "cpvLabel": "CPV Codes",
  "cpvPlaceholder": "Search CPV codes...",
  "cpvNoResults": "No CPV codes found",
  "cpvSelected": "{count} selected",
  "regionLabel": "Regions",
  "regionSelectAll": "Select all",
  "regionDeselectAll": "Deselect all",
  "budgetLabel": "Budget Range",
  "budgetMin": "Min (EUR)",
  "budgetMax": "Max (EUR)",
  "budgetRangeError": "Min must be less than or equal to Max",
  "deadlineLabel": "Deadline Range",
  "deadlineFrom": "From",
  "deadlineTo": "To",
  "deadlineRangeError": "From date must be before To date",
  "statusLabel": "Status",
  "typeLabel": "Opportunity Type",
  "typeWorks": "Works",
  "typeServices": "Services",
  "typeSupplies": "Supplies",
  "typeMixed": "Mixed",
  "chipCpv": "CPV: {value}",
  "chipRegion": "Region: {value}",
  "chipBudgetBoth": "Budget: €{min} – €{max}",
  "chipBudgetMin": "Budget: ≥ €{min}",
  "chipBudgetMax": "Budget: ≤ €{max}",
  "chipDeadlineBoth": "Deadline: {from} → {to}",
  "chipDeadlineFrom": "From: {from}",
  "chipDeadlineTo": "Until: {to}",
  "chipStatus": "Status: {value}",
  "chipType": "Type: {value}",
  "removeFilter": "Remove filter"
}
```

### data-testid Reference Table

| `data-testid` value | Component | Description |
|---|---|---|
| `filter-sidebar` | `FilterSidebar` | Root element of sidebar |
| `filter-section-cpv` | `FilterSidebar` | CPV section wrapper |
| `filter-section-regions` | `FilterSidebar` | Regions section wrapper |
| `filter-section-budget` | `FilterSidebar` | Budget section wrapper |
| `filter-section-deadline` | `FilterSidebar` | Deadline section wrapper |
| `filter-section-status` | `FilterSidebar` | Status section wrapper |
| `filter-section-type` | `FilterSidebar` | Type section wrapper |
| `active-filter-chips` | `ActiveFilterChips` | Chips container |
| `filter-clear-all` | `ActiveFilterChips` | Clear all button |
| `chip-remove-cpv` | `ActiveFilterChips` | Remove CPV filter chips |
| `chip-remove-region` | `ActiveFilterChips` | Remove region filter |
| `chip-remove-budget` | `ActiveFilterChips` | Remove budget filter |
| `chip-remove-deadline` | `ActiveFilterChips` | Remove deadline filter |
| `chip-remove-status` | `ActiveFilterChips` | Remove status filter |
| `chip-remove-type` | `ActiveFilterChips` | Remove type filter |
| `cpv-filter-search` | `CPVCodeFilter` | CPV search input |
| `cpv-filter-list` | `CPVCodeFilter` | Scrollable CPV list |
| `cpv-option-{code}` | `CPVCodeFilter` | Individual CPV list item |
| `cpv-selected-chips` | `CPVCodeFilter` | Selected codes chip row |
| `region-select-all` | `RegionFilter` | Select all/deselect all button |
| `region-checkbox-{code}` | `RegionFilter` | Per-country checkbox |
| `budget-min-input` | `BudgetRangeFilter` | Min budget input |
| `budget-max-input` | `BudgetRangeFilter` | Max budget input |
| `budget-range-error` | `BudgetRangeFilter` | Validation error message |
| `deadline-from-input` | `DeadlineRangeFilter` | From date input |
| `deadline-to-input` | `DeadlineRangeFilter` | To date input |
| `deadline-range-error` | `DeadlineRangeFilter` | Validation error message |
| `status-btn-open` | `StatusFilter` | Open status button |
| `status-btn-closing_soon` | `StatusFilter` | Closing soon button |
| `status-btn-closed` | `StatusFilter` | Closed status button |
| `type-btn-works` | `OpportunityTypeFilter` | Works type button |
| `type-btn-services` | `OpportunityTypeFilter` | Services type button |
| `type-btn-supplies` | `OpportunityTypeFilter` | Supplies type button |
| `type-btn-mixed` | `OpportunityTypeFilter` | Mixed type button |
| `filter-mobile-trigger` | `MobileFilterSheet` | Mobile Sheet trigger button |

### Test Design Coverage (from `eusolicit-docs/test-artifacts/test-design-epic-06.md`)

The following test IDs from the Epic 6 test design are targeted by this story:

| Test ID | Level | Scenario | Notes for ATDD |
|---------|-------|----------|----------------|
| **E06-P1-025** | Component | Filter sidebar: CPV autocomplete loads taxonomy; region checkboxes with select-all; budget slider; deadline date range picker — all filter changes fire debounced API calls | ATDD: verify `cpv-filter-search` testid in `CPVCodeFilter.tsx`; verify `CPV_CODES` import; verify `region-select-all` testid; verify `budget-min-input` + `budget-max-input`; verify `deadline-from-input` + `deadline-to-input`; verify `OpportunitiesListPage.tsx` reads filter params from `useSearchParams()` (driving re-fetch) |
| **E06-P1-026** | Component | Active filter chips render above results; clicking chip removes filter and re-fetches; "Clear all" removes all chips | ATDD: verify `active-filter-chips` + `filter-clear-all` testids in `ActiveFilterChips.tsx`; verify `chip-remove-{category}` testids; verify `ActiveFilterChips` import in `OpportunitiesListPage.tsx` |
| **E06-P2-012** | Component | Filter state persisted in URL query params — apply filters, copy URL, navigate in fresh context, verify filters restored | ATDD: verify `readFilterState` function exported from `opportunity-utils.tsx`; verify `OpportunitiesListPage.tsx` calls `readFilterState(searchParams)` and passes result to `useOpportunityListing` |
| **E06-P2-014** | Component | Mobile responsive: filter sidebar renders as bottom sheet on mobile | ATDD: verify `filter-mobile-trigger` testid in `MobileFilterSheet.tsx`; verify `lg:hidden` class on trigger; verify `FilterSidebar` inside `SheetContent`; verify `hidden lg:block` on desktop `<aside>` in `OpportunitiesListPage.tsx` |

**Playwright E2E spec** (`e2e/specs/opportunities/opportunities-listing.spec.ts`): Add a filter test block to the existing spec:
- GREEN: Filter button is visible on mobile viewport (< 1024px)
- GREEN: Filter sidebar `data-testid="filter-sidebar"` is present in DOM
- Skip (S06.14 pending): Active filter chips trigger upgrade modal on tier-restricted filter attempts

### Critical Mistakes to Prevent

1. **Do NOT render `<FilterSidebar>` twice** — once in the `<aside>` for desktop and again inside `<MobileFilterSheet>`. Render it only in the `<aside>` on desktop. `MobileFilterSheet` renders its own instance inside the Sheet, which only appears when the Sheet is open. The CSS visibility ensures only one is visible at a time.

2. **Do NOT modify `use-opportunities.ts`** — extending `OpportunityListParams` is sufficient. The hook already spreads all `params` into the `queryKey` and `queryFn` — filter fields will be picked up automatically.

3. **Do NOT implement CPV taxonomy API call** — the CPV codes are in the static local file `@/lib/data/cpv-codes.ts`. Calling an API endpoint for CPV codes would add latency and an unnecessary dependency. Use the existing static data.

4. **Do NOT forget to reset `after_cursor` on filter change** — when a filter changes, the cursor from a previous page fetch is stale. Always `params.delete("after_cursor")` in `handleFilterChange`. Failure to do this causes the filter to start from a mid-page position, returning unexpected results.

5. **Budget validation only blocks URL update — do NOT show red globally** — the validation error in `BudgetRangeFilter` is an inline message below the inputs. The filter only blocks calling `onChange` when invalid. The user can still navigate away, use other filters, etc. Do not disable the entire form.

6. **Do NOT use `useState` for filter state in `OpportunitiesListPage`** — all filter state is URL-driven (same as sort, view, search in S06.09). Use `readFilterState(searchParams)` to derive state. Never `useState` for `FilterState` in the page component.

7. **Regions URL param stores country NAMES, not codes** — the backend filter param `regions` (from S06.01) matches against opportunity location/region names like "Bulgaria", not ISO codes. Store names in the URL and send names to the API. Use `EuCountry.name` not `EuCountry.code` as the filter value. The checkbox label shows `{flag} {name}` and the value toggled is `name`.

8. **CPV URL param stores numeric codes without trailing zeros** — check the `CpvCode.code` format in `@/lib/data/cpv-codes.ts` before assuming format (e.g., `"45000000"` vs `"45000000-7"`). Use the exact `code` field value from the data file as the URL param value.

9. **`ScrollArea` from `@eusolicit/ui`** — use it for CPV list and region list. Do not use `overflow-y: scroll` on raw divs; the shadcn `ScrollArea` handles cross-browser scrollbar styling.

10. **`Sheet` from `@eusolicit/ui`** — import `Sheet`, `SheetContent`, `SheetHeader`, `SheetTitle` from `@eusolicit/ui`. The `side="bottom"` prop creates a bottom sheet on mobile.

### References

- Story 6.9 (listing page base): `eusolicit-docs/implementation-artifacts/6-9-opportunity-listing-page-table-card-view.md` — patterns for URL-driven state, search debounce, mobile responsive CSS
- `Step2CPVSectors` (CPV multi-select reference): `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/setup/components/Step2CPVSectors.tsx`
- `ConsortiumFinderPanel` (EU countries reference): `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/grants/tools/components/ConsortiumFinderPanel.tsx`
- `opportunity-utils.tsx` (to be extended): `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/opportunity-utils.tsx`
- Static CPV data: `eusolicit-app/frontend/apps/client/lib/data/cpv-codes.ts`
- Epic 6 test design (filter test IDs): `eusolicit-docs/test-artifacts/test-design-epic-06.md` — E06-P1-025, E06-P1-026, E06-P2-012, E06-P2-014
- Architecture v4: `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md`
- i18n messages: `eusolicit-app/frontend/apps/client/messages/en.json` + `bg.json`
- UI components: `eusolicit-app/frontend/packages/ui/src/components/ui/` (sheet.tsx, checkbox.tsx, scroll-area.tsx, badge.tsx, button.tsx, input.tsx, separator.tsx)

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (claude-sonnet-4-6)

### Debug Log References

- Iteration 1: FilterSidebar used template literals `filter-section-${testId}` — test checks for literal strings. Fixed to use explicit `data-testid="filter-section-cpv"` etc. values passed as props.
- All 185 ATDD tests pass after the fix (185/185 GREEN).
- 7 pre-existing test failures in other stories (auth-pages-s3-8, wizard-s3-9, grant-tools-s11-11, espd-s11-13, roi-tracker-s12-4) — unrelated to this story.

### Completion Notes List

- ✅ Created `lib/data/eu-countries.ts` — 27 EU member states with ISO codes, English names, flag emoji; exports `EuCountry`, `EU_COUNTRIES`, `EU_COUNTRY_MAP`
- ✅ Extended `OpportunityListParams` in `lib/api/opportunities.ts` with 8 new optional filter fields (cpv_codes, regions, budget_min, budget_max, deadline_from, deadline_to, status, opportunity_type)
- ✅ Added 37 new filter i18n keys to both `messages/en.json` and `messages/bg.json` under `opportunities` namespace; key-parity test passes
- ✅ Created `ActiveFilterChips.tsx` — dismissible badge chips per active filter; chip-remove-{category} testids; filter-clear-all button
- ✅ Created `CPVCodeFilter.tsx` — controlled multi-select with search, ScrollArea, dismissible chips; CPV_CODES import
- ✅ Created `RegionFilter.tsx` — EU_COUNTRIES import, per-country checkboxes, select-all/deselect-all toggle
- ✅ Created `BudgetRangeFilter.tsx` — number inputs with EUR prefix, 500ms debounce, inline validation blocking onChange when min > max
- ✅ Created `DeadlineRangeFilter.tsx` — date inputs, blur/change triggered, inline validation blocking onChange when from > to
- ✅ Created `StatusFilter.tsx` — three toggle buttons (open/closing_soon/closed), clicking active deselects
- ✅ Created `OpportunityTypeFilter.tsx` — four toggle buttons (works/services/supplies/mixed), clicking active deselects
- ✅ Created `FilterSidebar.tsx` — all 6 sections in collapsible FilterSection wrappers with hardcoded data-testid values; imports all 6 sub-filters
- ✅ Created `MobileFilterSheet.tsx` — lg:hidden trigger button, Sheet side="bottom", FilterSidebar inside SheetContent
- ✅ Extended `opportunity-utils.tsx` with FilterState interface, EMPTY_FILTERS, readFilterState, writeFilterToParams, countActiveFilters
- ✅ Restructured `OpportunitiesListPage.tsx` — two-column layout (hidden lg:block aside + flex-1 results), filter state from URL (no useState), handleFilterChange always deletes after_cursor, filter params passed to useOpportunityListing
- ✅ ATDD test file was pre-written; all 185 tests pass GREEN

### File List

**New files:**
- `eusolicit-app/frontend/apps/client/lib/data/eu-countries.ts`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/FilterSidebar.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/ActiveFilterChips.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/CPVCodeFilter.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/RegionFilter.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/BudgetRangeFilter.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/DeadlineRangeFilter.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/StatusFilter.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/OpportunityTypeFilter.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/MobileFilterSheet.tsx`
- `eusolicit-app/frontend/apps/client/__tests__/opportunities-filter-s6-10.test.ts`

**Modified files:**
- `eusolicit-app/frontend/apps/client/lib/api/opportunities.ts` — extend `OpportunityListParams` with filter fields
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/opportunity-utils.tsx` — add `FilterState`, `readFilterState`, `writeFilterToParams`, `countActiveFilters`, `EMPTY_FILTERS`
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/OpportunitiesListPage.tsx` — layout restructure, filter state integration
- `eusolicit-app/frontend/apps/client/messages/en.json` — add filter i18n keys
- `eusolicit-app/frontend/apps/client/messages/bg.json` — add filter i18n keys

## Senior Developer Review

**REVIEW: Approve**

Review Date: 2026-04-17 (Re-review)
Reviewer: Claude Opus 4.6 (Adversarial Code Review)
ATDD Tests: 185/185 GREEN
Previous Review: Changes Requested (7 findings) — ALL RESOLVED

---

### Prior Finding Verification (All Resolved)

| Finding | Status | Verified In |
|---------|--------|-------------|
| **B1** readOnly on Checkbox | RESOLVED | CPVCodeFilter.tsx:87-90, RegionFilter.tsx:59-62 — replaced with `onCheckedChange` + `tabIndex={-1}` + `aria-hidden="true"` |
| **B2** Chip removes entire category | RESOLVED | ActiveFilterChips.tsx:48-49, :70-72 — per-item filter: `cpvCodes: filters.cpvCodes.filter(c => c !== code)` |
| **B3** Double URL update on chip clear | RESOLVED | ActiveFilterChips.tsx:97, :121 — single `onFilterChange({ budgetMin: null, budgetMax: null })` call |
| **R1** NaN guard on budget params | RESOLVED | opportunity-utils.tsx:44-48 — `parseNumberParam()` helper with `isNaN()` check |
| **R2** before_cursor not cleared | RESOLVED | OpportunitiesListPage.tsx:85-86 — deletes both `after_cursor` and `before_cursor` |
| **R3** Stale closure in debounce | RESOLVED | BudgetRangeFilter.tsx:38-39 — `useRef(onChange)` pattern |
| **R4** No enum validation on URL params | RESOLVED | opportunity-utils.tsx:38-41, :62-67 — `VALID_STATUSES` / `VALID_OPPORTUNITY_TYPES` whitelists with `.includes()` |

---

### Re-Review: New Findings

No new blocking or required findings. Full line-by-line review of all 13 implementation files confirms:

- **Architecture alignment:** Pure frontend story; all filter state is URL-driven (no new stores); `useOpportunityListing` passes filter params through unchanged; `use-opportunities.ts` was not modified (correct per spec).
- **Component structure:** 10 new components + 1 data file; all follow the established pattern from S06.09. FilterSidebar is not duplicated in DOM — Sheet only renders content when open.
- **i18n completeness:** 37 filter keys present in both `en.json` and `bg.json` with proper Bulgarian translations. Key parity verified by ATDD test.
- **data-testid coverage:** All 35+ test IDs from the spec reference table are present in source. ATDD tests validate all of them.
- **URL param contract:** `OpportunityListParams` extended with 8 filter fields matching backend S06.01 API contract.
- **Debounce correctness:** Budget uses 500ms debounce with `useRef` for stable callback; deadline fires on change/blur (no debounce, per spec); CPV/region/status/type update immediately.
- **Pagination reset:** `handleFilterChange` clears both `after_cursor` and `before_cursor`.
- **Validation:** Budget NaN guard via `parseNumberParam()`; status/type validated against whitelist arrays; budget min > max and deadline from > to blocked from propagating to URL.

---

### Advisory Findings (Carried Forward — Deferrable)

These items from the initial review remain unresolved. They are non-blocking and tracked for future improvement:

#### A1. Missing `aria-pressed` on Toggle Buttons (StatusFilter.tsx, OpportunityTypeFilter.tsx)
WCAG 4.1.2 — screen readers cannot distinguish selected button. Add `aria-pressed={status === value}`.

#### A2. No Keyboard Handlers on Clickable List Items (CPVCodeFilter.tsx, RegionFilter.tsx)
WCAG 2.1.1 — `<li onClick>` lacks `tabIndex`, `onKeyDown`, `role`. Keyboard-only users cannot interact.

#### A3. `OpportunityTypeFilter` Prop Type Looser Than Necessary
Props interface uses `string | null` instead of `OpportunityTypeValue | null`; parent compensates with `as` cast (FilterSidebar.tsx:150).

#### A4. CPV Search is English-Only
`CPVCodeFilter` filters on `c.description` (English) only. Bulgarian users cannot search CPV codes in their language.

#### A5. Hardcoded `aria-label="Remove"` in CPVCodeFilter.tsx:52
Should use `t("removeFilter")` for consistency with `ActiveFilterChips.tsx`.

#### A6. Tests Are Purely Static String-Matching
All 185 ATDD tests use `.toContain()` on source strings. Acceptable as structural scaffold; no behavioral coverage.

---

### Summary

| Category | Count |
|----------|-------|
| Blocking fixes | 0 |
| Required fixes | 0 |
| Advisory / deferrable (carried forward) | 6 |

**Verdict: Approve.** All 3 blocking and 4 required findings from the initial review have been verified as resolved in source. The implementation correctly follows the story spec, architecture patterns, and URL-driven state management conventions from S06.09. The 6 advisory items (accessibility, type safety, i18n gaps) are deferrable and should be tracked for a future polish pass. 185/185 ATDD tests pass GREEN.

## Change Log

- **2026-04-17 (Story created):** Story file authored for S06.10; informed by Epic 6 test design (`test-design-epic-06.md`), S06.09 implementation patterns, existing `CPV_CODES` static data, `Step2CPVSectors` reference component, and available shadcn/ui components (sheet, checkbox, scroll-area, badge, button, input, separator).
- **2026-04-17 (Implementation):** Full implementation by Claude Sonnet 4.6. Created 10 new component files and 1 data file; modified 5 existing files. Added 37 i18n keys to both en.json and bg.json. All 185 ATDD tests pass GREEN. Story set to review.
- **2026-04-17 (Review fixes):** All 7 code review findings resolved by Claude Opus 4.6:
  - **B1 (TypeScript compile errors):** Removed invalid `readOnly` prop from `<Checkbox>` in CPVCodeFilter.tsx and RegionFilter.tsx; replaced with `onCheckedChange` handler + `tabIndex={-1}` + `aria-hidden="true"` for proper accessible delegation to parent `<li onClick>`.
  - **B2 (Per-item chip removal):** Refactored ActiveFilterChips from `onClearFilter(category)` to `onFilterChange(partial)` API, enabling per-item CPV/region chip removal instead of clearing entire category.
  - **B3 (Double URL update):** Budget and deadline chip removal now uses single `onFilterChange({ budgetMin: null, budgetMax: null })` call instead of two sequential `onClearFilter` calls.
  - **R1 (NaN guard):** Added `parseNumberParam()` helper in opportunity-utils.tsx that returns null for NaN/non-numeric budget URL params.
  - **R2 (before_cursor):** `handleFilterChange` now deletes both `after_cursor` and `before_cursor` on filter change.
  - **R3 (Stale closure):** BudgetRangeFilter now uses `useRef(onChange)` pattern to avoid stale closure in 500ms debounce timer.
  - **R4 (Enum validation):** Added `VALID_STATUSES` and `VALID_OPPORTUNITY_TYPES` whitelist arrays; `readFilterState` validates URL params against them instead of raw `as` casts.
  - All 185 ATDD tests pass GREEN. Zero new TypeScript errors (20 pre-existing in other stories). Story set to done.
- **2026-04-17 (Re-review):** Adversarial re-review by Claude Opus 4.6. Verified all 7 prior findings (B1-B3, R1-R4) resolved in source via line-by-line inspection of all 13 implementation files. 185/185 ATDD tests confirmed GREEN. No new blocking or required findings. 6 advisory items (A1-A6) carried forward as deferrable. **REVIEW: Approve.**

## Known Deviations

### Detected by `3-code-review` at 2026-04-17T09:18:42Z — ALL RESOLVED

All blocking and required deviations have been resolved in the review fixes pass. Confirmed by re-review on 2026-04-17.

### Advisory Items (Deferrable — tracked for future)
- A1: Missing `aria-pressed` on toggle buttons (WCAG 4.1.2)
- A2: No keyboard handlers on clickable list items (WCAG 2.1.1)
- A3: OpportunityTypeFilter prop type looser than necessary
- A4: CPV search is English-only (i18n gap)
- A5: Hardcoded `aria-label="Remove"` in CPVCodeFilter (should use i18n key)
- A6: Tests are purely static string-matching (no behavioral testing)
