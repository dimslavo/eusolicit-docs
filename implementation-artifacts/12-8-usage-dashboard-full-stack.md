# Story 12.8: Usage Dashboard (Full Stack)

Status: done

## Story

As a **paid-tier user on the EU Solicit platform**,
I want **a usage dashboard at `/analytics/usage`**,
so that **I can see my current billing period's consumption vs. tier limits for AI summaries, proposal drafts, and compliance checks via circular progress meters — with a warning indicator when I am above 80% of any limit and an upgrade CTA when I am at or above 100% of any limit**.

## Acceptance Criteria

### Backend — Usage Consumption Endpoint

1. **AC1** — `GET /api/v1/analytics/usage` returns current billing period consumption vs. limits for the authenticated user's company. Response body is a JSON object with fields:
   - `billing_period_start` (ISO date string, e.g. `"2026-04-01"`)
   - `billing_period_end` (ISO date string, e.g. `"2026-04-30"`)
   - `usage` — an array of 3 objects (one per metric type), each containing:
     - `metric_type` (`"ai_summaries"` | `"proposal_drafts"` | `"compliance_checks"`)
     - `consumed` (integer)
     - `limit` (integer) — the `limit_value` column from `mv_usage_consumption`
     - `remaining` (integer) — `GREATEST(limit_value - consumed, 0)`

   The array is ordered: `ai_summaries`, `proposal_drafts`, `compliance_checks`.

   If no current-period usage record exists for a metric type, include a placeholder entry with `consumed: 0`, `limit: 0`, `remaining: 0` so the frontend always receives all 3 types.

   When no usage data exists at all for the company, returns HTTP 200 with `billing_period_start: null`, `billing_period_end: null`, and `usage: []` — never 4xx.

2. **AC2** — The current billing period is determined by querying `mv_usage_consumption` filtered to the row(s) where `period_start <= CURRENT_DATE AND period_end >= CURRENT_DATE` for the company. If multiple rows have overlapping periods (edge case), the one with the latest `period_start` is used. This filter is applied after the mandatory `company_id` scoping clause.

3. **AC3 — Authentication & Access Control**: The endpoint requires a valid Bearer JWT (`→ 401` on missing/invalid). Access is gated by `require_paid_tier` dependency (`→ 403` with `ForbiddenError(details={"upgrade_required": True})` for Free tier users). All paid tiers (Starter, Professional, Enterprise) can access the endpoint.

4. **AC4 — Tenant Isolation (E12-R-001)**: Every SQL statement in the usage service MUST include `WHERE company_id = current_user.company_id` as the **first, non-optional clause** before any period filters or ordering. This clause must appear in the service function before any optional filters.

5. **AC5 — Cache Headers**: The endpoint returns `Cache-Control: public, max-age=1800` (30 minutes — a reasonable staleness window given the hourly MV refresh cadence).

6. **AC6 — Unit Tests** in `services/client-api/tests/api/test_analytics_usage.py` cover:
   - Happy path: paid-tier token → 200 response containing all 3 metric types with correct field names
   - `billing_period_start` and `billing_period_end` present and valid ISO date strings
   - `remaining` equals `max(limit - consumed, 0)` — verified against seeded data
   - Tenant isolation: Company B rows are not returned in Company A's results (assert mock call passes `company_id = current_user.company_id`)
   - Empty result (no usage rows): 200 with `billing_period_start: null`, `billing_period_end: null`, `usage: []`
   - Missing metric type: placeholder row with `consumed: 0`, `limit: 0`, `remaining: 0` included for absent types
   - Free tier JWT → 403 with `upgrade_required: True`
   - Unauthenticated (no JWT) → 401
   - `Cache-Control: public, max-age=1800` header present on response
   - All 3 metric types are always present in the response (in order: `ai_summaries`, `proposal_drafts`, `compliance_checks`)

---

### Frontend — Usage Dashboard

7. **AC7** — A new page is created at `apps/client/app/[locale]/(protected)/analytics/usage/page.tsx`. It is a server component shell (no `"use client"` directive) that imports and renders `<UsageDashboard />` from `./components/UsageDashboard`. The route resolves to `/{locale}/analytics/usage`.

8. **AC8 — Usage Meters Section**: The dashboard renders a meters section with `data-testid="usage-meters"` containing one progress meter component per usage type. The section heading uses `t("analytics.usage.metersTitle")`. Three meters are always rendered:
   - **AI Summaries** — `data-testid="usage-meter-ai-summaries"`
   - **Proposal Drafts** — `data-testid="usage-meter-proposal-drafts"`
   - **Compliance Checks** — `data-testid="usage-meter-compliance-checks"`

   Each meter displays:
   - The usage type label (e.g. `t("analytics.usage.aiSummariesLabel")`)
   - A circular or radial progress indicator showing `consumed / limit` as a percentage
   - Text: `{consumed} / {limit}` (e.g. `"45 / 100"`) with `data-testid="usage-meter-{metric_type}-count"`
   - A percentage label (e.g. `"45%"`) with `data-testid="usage-meter-{metric_type}-pct"`

   When loading, each meter's position renders `<SkeletonCard data-testid="usage-meters-skeleton" />`. When `usage` is an empty array, renders `<EmptyState data-testid="usage-empty" title={t("analytics.usage.emptyTitle")} />` instead of the meters section.

9. **AC9 — Billing Period Display**: A billing period badge renders above the meters with `data-testid="usage-billing-period"`. Displays `t("analytics.usage.billingPeriodLabel")` followed by the period dates formatted as `MMM d – MMM d, YYYY` (e.g. `"Apr 1 – Apr 30, 2026"`). Hidden when `billing_period_start` is null.

10. **AC10 — Warning Indicator**: For each usage meter where `consumed / limit >= 0.80` (i.e. 80% or above but below 100%), render a warning badge with `data-testid="usage-warning-{metric_type}"` containing `t("analytics.usage.warningLabel")` (e.g. `"80% reached"`). The badge uses Tailwind classes `bg-amber-100 text-amber-800` (or equivalent `variant="warning"`). The warning is NOT shown at 100% — the upgrade CTA replaces it.

11. **AC11 — Upgrade CTA**: When any usage type has `consumed >= limit` (i.e. 100% or above), render an upgrade call-to-action section below the meters with `data-testid="usage-upgrade-cta"`. Contents:
    - Heading: `t("analytics.usage.ctaTitle")`
    - Description: `t("analytics.usage.ctaDescription")`
    - A link button: `t("analytics.usage.ctaButton")` with `href="/settings/billing"` and `data-testid="usage-upgrade-cta-link"`

    The CTA is shown once (not once per limit-exceeded type). Each individual meter at 100% renders `data-testid="usage-limit-reached-{metric_type}"` indicator (e.g. a red badge with `bg-red-100 text-red-800`) instead of the warning badge.

12. **AC12 — API Client**: `apps/client/lib/api/analytics.ts` is **updated** (not replaced) to add:
    - `UsageMetric` interface: `{ metric_type: "ai_summaries" | "proposal_drafts" | "compliance_checks"; consumed: number; limit: number; remaining: number }`
    - `UsageSummary` interface: `{ billing_period_start: string | null; billing_period_end: string | null; usage: UsageMetric[] }`
    - `fetchUsageSummary(): Promise<UsageSummary>` — calls `GET /api/v1/analytics/usage`

13. **AC13 — TanStack Query Hook**: Create `apps/client/lib/queries/use-usage-analytics.ts` with `"use client"` directive. Exports:
    - `useUsageSummary()` — `queryKey: ["usage-summary"]`, `staleTime: 1_800_000` (30 minutes, matching Cache-Control max-age=1800)

14. **AC14 — i18n**: Add `analytics.usage` sub-key to `apps/client/messages/en.json` and `apps/client/messages/bg.json`. Required keys:
    - `pageTitle`, `pageDescription`
    - `metersTitle`
    - `billingPeriodLabel`
    - `aiSummariesLabel`, `proposalDraftsLabel`, `complianceChecksLabel`
    - `warningLabel`
    - `ctaTitle`, `ctaDescription`, `ctaButton`
    - `emptyTitle`, `emptyDescription`
    - `limitReachedLabel`

15. **AC15 — Frontend Unit Tests**: `apps/client/__tests__/usage-dashboard-s12-8.test.ts` verifies:
    - Structural existence of all components and their `data-testid` attributes
    - API fetch function `fetchUsageSummary` exported from `lib/api/analytics.ts`
    - Hook `useUsageSummary` exported from `lib/queries/use-usage-analytics.ts` with `staleTime = 1_800_000`
    - All `analytics.usage.*` keys present in both `en.json` and `bg.json`
    - `UsageMeter` component file exists and contains the 3 metric-type testid patterns
    - `UsageUpgradeCta` (or equivalent) component exports and contains `data-testid="usage-upgrade-cta"`
    - Follows the same structural verification pattern used in `pipeline-forecast-s12-7.test.ts`

---

### E2E ATDD — Cross-Tenant Isolation

16. **AC16 — Cross-Tenant P0 test already covers `/api/v1/analytics/usage`** in `e2e/specs/analytics/cross-tenant-isolation.api.spec.ts` (the second test iterates over `analyticsEndpoints` including `/api/v1/analytics/usage`). No new ATDD file is needed. When S12.08 is implemented, the existing P0 test must pass GREEN for the usage endpoint — verify that company A's usage response contains no `company_id` matching company B.

---

## Tasks / Subtasks

### Backend

- [x] Task 1: Pydantic schemas — append to `schemas/analytics.py` (AC: 1)
  - [x] 1.1 Add `# ─── Usage Dashboard (S12.08) ──────` section to `schemas/analytics.py`
  - [x] 1.2 Define `UsageMetricItem(metric_type: Literal["ai_summaries", "proposal_drafts", "compliance_checks"], consumed: int, limit: int, remaining: int)`
  - [x] 1.3 Define `UsageSummaryResponse(billing_period_start: date | None, billing_period_end: date | None, usage: list[UsageMetricItem])`
  - [x] 1.4 Add both schemas to `__all__` in `schemas/analytics.py`

- [x] Task 2: Usage analytics service — `services/analytics_usage_service.py` (AC: 1, 2, 4)
  - [x] 2.1 Create `services/client-api/src/client_api/services/analytics_usage_service.py`
  - [x] 2.2 Implement `async get_usage_summary(company_id: UUID, session: AsyncSession) -> UsageSummaryResponse`
  - [x] 2.3 Base query: `SELECT metric_type, consumed, limit_value, GREATEST(limit_value - consumed, 0) AS remaining, period_start, period_end FROM client.mv_usage_consumption WHERE company_id = :company_id` — `company_id` MUST be the first WHERE clause
  - [x] 2.4 Filter to current billing period: add `AND period_start <= CURRENT_DATE AND period_end >= CURRENT_DATE`; if multiple rows per type, take latest `period_start` (use `DISTINCT ON (metric_type) ORDER BY metric_type, period_start DESC`)
  - [x] 2.5 Determine billing period dates from any returned row's `period_start` / `period_end` (all rows in a single period share the same dates)
  - [x] 2.6 Build response: ensure all 3 metric types are present — if a type is missing from query results, insert a placeholder `UsageMetricItem(metric_type=<type>, consumed=0, limit=0, remaining=0)`
  - [x] 2.7 Order final `usage` array: `["ai_summaries", "proposal_drafts", "compliance_checks"]`
  - [x] 2.8 Return `UsageSummaryResponse(billing_period_start=None, billing_period_end=None, usage=[])` when no rows found (empty result, no exception)

- [x] Task 3: Usage analytics router — `api/v1/analytics_usage.py` (AC: 3, 5)
  - [x] 3.1 Create `services/client-api/src/client_api/api/v1/analytics_usage.py`
  - [x] 3.2 Single route: `GET /` (root of the router) with `require_paid_tier` as auth dependency
  - [x] 3.3 Call `analytics_usage_service.get_usage_summary(company_id=current_user.company_id, session=session)`
  - [x] 3.4 Set `Cache-Control: public, max-age=1800` response header
  - [x] 3.5 Return `UsageSummaryResponse`

- [x] Task 4: Register router in `main.py` (AC: 3)
  - [x] 4.1 Import `analytics_usage` router and include with prefix `/api/v1/analytics/usage`

- [x] Task 5: Backend unit tests — `tests/api/test_analytics_usage.py` (AC: 6)
  - [x] 5.1 Create test file; mock `analytics_usage_service.get_usage_summary`
  - [x] 5.2 Override `require_paid_tier` with controlled `CurrentUser` (paid and free tier variants)
  - [x] 5.3 Test: paid-tier token → 200 with `UsageSummaryResponse` shape; all 3 metric types present
  - [x] 5.4 Test: `billing_period_start` and `billing_period_end` are valid ISO date strings (not null when data present)
  - [x] 5.5 Test: `remaining` correctly computed as `max(limit - consumed, 0)` with seeded data
  - [x] 5.6 Test: tenant isolation — assert service called with `company_id = current_user.company_id` (mock call args)
  - [x] 5.7 Test: no usage rows → 200 with `billing_period_start: None`, `billing_period_end: None`, `usage: []`
  - [x] 5.8 Test: missing metric type → placeholder row with `consumed=0, limit=0, remaining=0` in response
  - [x] 5.9 Test: free tier JWT → 403 with `upgrade_required: True`
  - [x] 5.10 Test: unauthenticated (no JWT) → 401
  - [x] 5.11 Test: `Cache-Control: public, max-age=1800` header present on 200 response
  - [x] 5.12 Test: usage array is always ordered `ai_summaries`, `proposal_drafts`, `compliance_checks`

### Frontend

- [x] Task 6: Update `lib/api/analytics.ts` with Usage interfaces and fetch function (AC: 12)
  - [x] 6.1 Add `// ─── Usage Dashboard (S12.08) ──────` section to `analytics.ts`
  - [x] 6.2 Define `UsageMetric { metric_type: "ai_summaries" | "proposal_drafts" | "compliance_checks"; consumed: number; limit: number; remaining: number }`
  - [x] 6.3 Define `UsageSummary { billing_period_start: string | null; billing_period_end: string | null; usage: UsageMetric[] }`
  - [x] 6.4 Define `fetchUsageSummary(): Promise<UsageSummary>`; URL: `"/api/v1/analytics/usage"` (no query params)

- [x] Task 7: TanStack Query hook — `lib/queries/use-usage-analytics.ts` (AC: 13)
  - [x] 7.1 Create `apps/client/lib/queries/use-usage-analytics.ts` with `"use client"` directive
  - [x] 7.2 Export `useUsageSummary()` — `queryKey: ["usage-summary"]`, `staleTime: 1_800_000`

- [x] Task 8: Add `analytics.usage` i18n keys to `en.json` and `bg.json` (AC: 14)
  - [x] 8.1 Add all required keys (listed in AC14) to `frontend/apps/client/messages/en.json`
  - [x] 8.2 Add the same keys (translated to Bulgarian) to `frontend/apps/client/messages/bg.json`

- [x] Task 9: Create components in `app/[locale]/(protected)/analytics/usage/components/` (AC: 8–11)
  - [x] 9.1 `UsageMeter.tsx` — renders a single circular/radial progress meter for one `UsageMetric`; accepts `metric: UsageMetric` and `metricLabel: string` props; contains `data-testid="usage-meter-{metric_type}"`, `data-testid="usage-meter-{metric_type}-count"`, `data-testid="usage-meter-{metric_type}-pct"`; applies warning badge (`data-testid="usage-warning-{metric_type}"`) at 80–99% and limit badge (`data-testid="usage-limit-reached-{metric_type}"`) at 100%+
  - [x] 9.2 `UsageMetersSection.tsx` — renders the `data-testid="usage-meters"` wrapper and maps over `usage` array to render 3 `<UsageMeter />` instances; handles loading skeleton and empty state
  - [x] 9.3 `UsageBillingPeriod.tsx` — renders `data-testid="usage-billing-period"` with formatted period dates; returns null when `billing_period_start` is null
  - [x] 9.4 `UsageUpgradeCta.tsx` — renders `data-testid="usage-upgrade-cta"` with heading, description, and CTA link to `/settings/billing`; only rendered when any metric has `consumed >= limit`

- [x] Task 10: Dashboard orchestrator `UsageDashboard.tsx` (AC: 7–11)
  - [x] 10.1 Create `components/UsageDashboard.tsx` — `"use client"` component
  - [x] 10.2 Use `useUsageSummary()` hook; handle `isLoading`, `isError`, `data`
  - [x] 10.3 When `isLoading`: render `<SkeletonCard data-testid="usage-meters-skeleton" />` in meters position
  - [x] 10.4 When `data.usage.length === 0`: render `<EmptyState data-testid="usage-empty" title={t("analytics.usage.emptyTitle")} />`
  - [x] 10.5 Otherwise: render `<UsageBillingPeriod />` + `<UsageMetersSection />` + conditionally `<UsageUpgradeCta />` (when any usage `consumed >= limit`)

- [x] Task 11: Page shell `app/[locale]/(protected)/analytics/usage/page.tsx` (AC: 7)
  - [x] 11.1 Create server component file (no `"use client"`)
  - [x] 11.2 Set Next.js metadata: `title: t("analytics.usage.pageTitle")`, `description: t("analytics.usage.pageDescription")` — implemented via `generateMetadata` with `getTranslations` from `next-intl/server`
  - [x] 11.3 Render `<UsageDashboard />`

- [x] Task 12: Frontend unit tests — `apps/client/__tests__/usage-dashboard-s12-8.test.ts` (AC: 15)
  - [x] 12.1 Structural test: `UsageMeter.tsx` file exists, exports component, `data-testid="usage-meter-ai-summaries"` present in source, `data-testid="usage-warning-ai-summaries"` present, `data-testid="usage-limit-reached-ai-summaries"` present
  - [x] 12.2 Structural test: `UsageMetersSection.tsx` file exists, exports component, `data-testid="usage-meters"` present
  - [x] 12.3 Structural test: `UsageBillingPeriod.tsx` file exists, `data-testid="usage-billing-period"` present
  - [x] 12.4 Structural test: `UsageUpgradeCta.tsx` file exists, `data-testid="usage-upgrade-cta"` present, `data-testid="usage-upgrade-cta-link"` present
  - [x] 12.5 API function test: `fetchUsageSummary` exported from `lib/api/analytics.ts`
  - [x] 12.6 Hook test: `useUsageSummary` exported from `lib/queries/use-usage-analytics.ts`; `staleTime` is `1_800_000`
  - [x] 12.7 i18n key tests: all `analytics.usage.*` keys listed in AC14 present in both `en.json` and `bg.json`

### Review Follow-ups (AI)

- [x] [AI-Review] F1 — [High] UsageMeter.tsx: Replace hardcoded "Limit reached" and "80% reached" badge text with `t("limitReachedLabel")` and `t("warningLabel")` using `useTranslations("analytics.usage")`
- [x] [AI-Review] F2 — [High] page.tsx: Replace static `metadata` export with async `generateMetadata` using `getTranslations({ locale, namespace: "analytics.usage" })` from `next-intl/server`

---

## Dev Notes

### Data Source: `client.mv_usage_consumption` (Created in S12.01)

The `mv_usage_consumption` materialized view was created in S12.01 (migration `011`) and is refreshed hourly by a Celery Beat task. Its source is `shared.usage_meters` — a real table updated by the billing/subscription system (E08) whenever AI features are consumed.

The MV columns:
```sql
company_id    UUID
metric_type   TEXT   -- "ai_summaries" | "proposal_drafts" | "compliance_checks"
period_start  DATE
period_end    DATE
consumed      INTEGER
limit_value   INTEGER
remaining     INTEGER  -- GREATEST(limit_value - consumed, 0)
updated_at    TIMESTAMPTZ
```

Unique index: `(company_id, metric_type, period_start)` — used for CONCURRENT refresh and enforces one row per company/type/period.

### Current Period Query Logic (Task 2.4)

The `shared.usage_meters` table may retain historical period rows (prior billing periods). The service MUST filter to the current billing period to avoid returning stale data. Use the pattern:

```python
query = (
    sa.select(mv_usage_consumption)
    .where(
        mv_usage_consumption.c.company_id == company_id,  # MUST be first
        sa.func.current_date().between(
            mv_usage_consumption.c.period_start,
            mv_usage_consumption.c.period_end,
        ),
    )
    .distinct(mv_usage_consumption.c.metric_type)
    .order_by(
        mv_usage_consumption.c.metric_type,
        mv_usage_consumption.c.period_start.desc(),
    )
)
```

`DISTINCT ON (metric_type)` with `ORDER BY metric_type, period_start DESC` guarantees one row per metric type, taking the most recent period if there are overlapping rows (edge case in month boundary transitions).

### Placeholder Rows for Missing Metric Types (Task 2.6)

A company on a plan that does not track a particular metric may have no row for that type in the current period. The service must fill in placeholders so the frontend always renders all 3 meters. After querying, build a dict keyed by `metric_type` and supplement missing keys:

```python
METRIC_TYPES = ["ai_summaries", "proposal_drafts", "compliance_checks"]
result_map = {row.metric_type: row for row in rows}
usage_items = []
for metric_type in METRIC_TYPES:
    if metric_type in result_map:
        row = result_map[metric_type]
        usage_items.append(UsageMetricItem(
            metric_type=metric_type,
            consumed=row.consumed,
            limit=row.limit_value,
            remaining=row.remaining,
        ))
    else:
        usage_items.append(UsageMetricItem(
            metric_type=metric_type, consumed=0, limit=0, remaining=0
        ))
```

### No New Migration Needed

Unlike S12.07 (which required a new table), S12.08 has **no Alembic migration**. The `mv_usage_consumption` view already exists from S12.01 (migration `011`). The `analytics_views.py` SQLAlchemy model for the view was also created in S12.01. This story only adds a new service + router that reads from the existing view.

### `require_paid_tier` Dependency

`require_paid_tier` is the same dependency used in S12.02, S12.03, S12.04 (market, ROI analytics). It enforces that the user has an active paid subscription (Starter or above). Free tier users receive a 403 with `ForbiddenError(details={"upgrade_required": True})`. Import from `core/tier_gate.py`.

This endpoint is **NOT** gated by `require_professional_plus_tier` — the usage dashboard is available to all paid tiers, consistent with the epic AC: "Dashboard available to all paid tiers."

### Circular Progress Meter Implementation

The `UsageMeter` component should use a circular or radial progress indicator. Implementation options (in order of preference):
1. **Custom SVG `<circle>` with stroke-dasharray** — no new dependency; matches the existing pattern in the design system; fully accessible
2. **Radix UI Progress** (`@radix-ui/react-progress`) with CSS transform for circular appearance — if already in the dependency tree
3. **Recharts `RadialBarChart`** — available since Recharts is already used for other dashboards

For MVP, a clean SVG implementation is recommended:
```tsx
const radius = 40;
const circumference = 2 * Math.PI * radius;
const pct = limit > 0 ? Math.min(consumed / limit, 1) : 0;
const strokeDashoffset = circumference * (1 - pct);
// <circle strokeDasharray={circumference} strokeDashoffset={strokeDashoffset} ... />
```

Color the progress stroke: `stroke-blue-500` (normal), `stroke-amber-500` (≥80%), `stroke-red-500` (≥100%).

### Warning vs. Limit-Reached State Logic

Two distinct UI states to implement in `UsageMeter`:

| Condition | UI State | Badge testid | Badge color |
|:---|:---|:---|:---|
| `consumed / limit >= 0.80 AND consumed < limit` | Warning | `usage-warning-{type}` | `bg-amber-100 text-amber-800` |
| `consumed >= limit` | Limit Reached | `usage-limit-reached-{type}` | `bg-red-100 text-red-800` |
| `consumed / limit < 0.80` | Normal | — | — |

Note: when `limit === 0` (placeholder row), render normal state with `"0 / 0"` and no warning — avoid division-by-zero in the percentage calculation.

### Frontend: Upgrade CTA Visibility

The `<UsageUpgradeCta />` is conditionally rendered at the **dashboard level** (in `UsageDashboard.tsx`), not inside each meter. It appears once when ANY of the 3 metrics has `consumed >= limit`. Individual meters still show the `usage-limit-reached-{type}` badge for per-type visual indication.

```typescript
const anyAtLimit = data.usage.some(m => m.consumed >= m.limit && m.limit > 0);
```

### Billing Period Date Formatting

Use `Intl.DateTimeFormat` with `"en-IE"` locale (consistent with other dashboards in this epic). Format: `"Apr 1 – Apr 30, 2026"`. Helper function:

```typescript
function formatBillingPeriod(start: string, end: string): string {
  const fmt = new Intl.DateTimeFormat("en-IE", { month: "short", day: "numeric" });
  const fmtYear = new Intl.DateTimeFormat("en-IE", { month: "short", day: "numeric", year: "numeric" });
  return `${fmt.format(new Date(start))} – ${fmtYear.format(new Date(end))}`;
}
```

### P0 Cross-Tenant Test Coverage (AC16)

The existing `cross-tenant-isolation.api.spec.ts` includes `/api/v1/analytics/usage` in its `analyticsEndpoints` array (line 95). When S12.08 is fully implemented and the test fixture `companyWithTier(tier)` / `analyticsData(companyId)` is seeded with current-period usage data for both companies, the P0 isolation test will verify that Company A's usage response contains no records belonging to Company B. No additional ATDD file needs to be created for this story.

### Test Design Coverage for S12.08

From `test-design-epic-12.md`, the following scenarios cover S12.08:

**P1 (6 tests total):**
- `GET /analytics/usage` returns consumed/limit/remaining for all 3 usage types (API, 2 tests — filter combinations and data shape)
- Response includes billing period start/end dates (API, 1 test — assert date fields present and valid)
- Frontend circular progress meters render for each usage type (E2E, 1 test — assert 3 meter elements with correct labels)
- Warning indicator shown when usage exceeds 80% of limit (E2E, 1 test — seed usage at 85% limit; assert warning UI element)
- Upgrade CTA displayed when any usage type is at or above limit (E2E, 1 test — seed usage at 100% limit; assert CTA link to billing)

**P2 (1 test):**
- Usage dashboard accessible to all paid tiers (Starter, Professional, Enterprise) — test with each tier JWT; assert 200

The unit tests in AC6 and AC15 cover the P1 API and structural scenarios. The E2E scenarios are tracked in the ATDD backlog for the TEA/QA sprint.

---

## Senior Developer Review

**Review Date:** 2026-04-13
**Verdict:** CHANGES REQUESTED (deferrable) — findings resolved 2026-04-13

### Test Results

- Backend: **13/13 PASS** (`test_analytics_usage.py`)
- Frontend: **69/69 PASS** (`usage-dashboard-s12-8.test.ts`)

### AC Coverage Summary

All 16 acceptance criteria verified. 14/16 fully satisfied. 2 deferrable gaps identified below.

### Findings

#### F1 — UsageMeter.tsx: Hardcoded badge text breaks i18n (AC10, AC14) — DEFERRABLE

`UsageMeter.tsx` hardcodes English badge strings instead of consuming i18n translation keys:

- Line 125: `"Limit reached"` → should be `t("limitReachedLabel")`
- Line 133: `"80% reached"` → should be `t("warningLabel")`

The component does not import `useTranslations`. The i18n keys `analytics.usage.limitReachedLabel` and `analytics.usage.warningLabel` are defined in both `en.json` and `bg.json` but are never used. Bulgarian users will see English text in warning/limit badges.

**Fix:** Import `useTranslations("analytics.usage")` in `UsageMeter.tsx` (or accept translated strings as props) and replace hardcoded strings.

#### F2 — page.tsx: Static English metadata instead of translated metadata (Task 11.2) — DEFERRABLE

The page shell exports static `Metadata` with hardcoded English title/description. Task 11.2 specifies using `t("analytics.usage.pageTitle")`. The correct pattern is `generateMetadata` with `getTranslations` from `next-intl/server`.

**Fix:** Replace static `metadata` export with async `generateMetadata` using `getTranslations("analytics.usage")`.

### Architecture Alignment

- ✅ Reads from existing `mv_usage_consumption` (S12.01, migration 011) — no new migrations
- ✅ Tenant isolation (E12-R-001): `company_id` is first, non-optional WHERE clause
- ✅ `require_paid_tier` (not `require_professional_plus_tier`) — all paid tiers have access
- ✅ Router registered in `main.py` at `/api/v1/analytics/usage`
- ✅ Cross-tenant P0 test includes `/api/v1/analytics/usage` endpoint (AC16)
- ✅ No scope creep detected

### Review Finding Resolutions (2026-04-13)

- ✅ Resolved review finding [deferrable]: F1 — UsageMeter.tsx now imports `useTranslations("analytics.usage")` and uses `t("limitReachedLabel")` and `t("warningLabel")` instead of hardcoded English strings. Bulgarian users now see translated badge text.
- ✅ Resolved review finding [deferrable]: F2 — page.tsx now exports `generateMetadata` (async) using `getTranslations({ locale, namespace: "analytics.usage" })` from `next-intl/server`. Page title/description are now locale-aware.

---

## Dev Agent Record

### Implementation Plan

Story 12.8 implements a full-stack Usage Dashboard (backend + frontend) for the `/analytics/usage` route available to all paid-tier users.

**Backend approach:**
- Pydantic schemas added to `schemas/analytics.py` — `UsageMetricItem` and `UsageSummaryResponse`
- Service `analytics_usage_service.py` queries `client.mv_usage_consumption` with tenant-first isolation (`company_id` as first WHERE clause), current-period filter, and DISTINCT ON for deduplication; fills placeholder rows for missing metric types
- FastAPI router at `api/v1/analytics_usage.py` — `GET /` protected by `require_paid_tier`, sets `Cache-Control: public, max-age=1800`
- Router registered in `main.py` at prefix `/api/v1/analytics/usage`

**Frontend approach:**
- API interfaces and `fetchUsageSummary()` added to `lib/api/analytics.ts`
- TanStack Query hook `useUsageSummary` in `lib/queries/use-usage-analytics.ts` with `staleTime: 1_800_000`
- Four components in `analytics/usage/components/`: `UsageMeter` (SVG circular progress), `UsageMetersSection` (3-meter grid + skeleton/empty), `UsageBillingPeriod` (formatted billing badge), `UsageUpgradeCta` (limit-exceeded CTA)
- `UsageDashboard.tsx` orchestrator handles loading/error/empty/data states
- Server-component page shell with `generateMetadata` for locale-aware metadata

**Review follow-up fixes (2026-04-13):**
- F1: Added `useTranslations("analytics.usage")` to `UsageMeter.tsx`; replaced hardcoded "Limit reached" and "80% reached" with `t("limitReachedLabel")` / `t("warningLabel")`
- F2: Replaced static `metadata` export in `page.tsx` with async `generateMetadata` using `getTranslations({ locale, namespace: "analytics.usage" })` from `next-intl/server`

### Completion Notes

All 16 acceptance criteria satisfied. Tests: 13/13 backend pass, 69/69 frontend pass. All review findings resolved.

---

## File List

### Backend
- `eusolicit-app/services/client-api/src/client_api/schemas/analytics.py` (modified — UsageMetricItem, UsageSummaryResponse added)
- `eusolicit-app/services/client-api/src/client_api/services/analytics_usage_service.py` (new)
- `eusolicit-app/services/client-api/src/client_api/api/v1/analytics_usage.py` (new)
- `eusolicit-app/services/client-api/src/client_api/main.py` (modified — router registered)
- `eusolicit-app/services/client-api/tests/api/test_analytics_usage.py` (new)

### Frontend
- `eusolicit-app/frontend/apps/client/lib/api/analytics.ts` (modified — UsageMetric, UsageSummary, fetchUsageSummary added)
- `eusolicit-app/frontend/apps/client/lib/queries/use-usage-analytics.ts` (new)
- `eusolicit-app/frontend/apps/client/messages/en.json` (modified — analytics.usage keys added)
- `eusolicit-app/frontend/apps/client/messages/bg.json` (modified — analytics.usage keys added)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/analytics/usage/page.tsx` (new — updated to generateMetadata for review fix F2)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/analytics/usage/components/UsageDashboard.tsx` (new)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/analytics/usage/components/UsageMeter.tsx` (new — updated with useTranslations for review fix F1)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/analytics/usage/components/UsageMetersSection.tsx` (new)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/analytics/usage/components/UsageBillingPeriod.tsx` (new)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/analytics/usage/components/UsageUpgradeCta.tsx` (new)
- `eusolicit-app/frontend/apps/client/__tests__/usage-dashboard-s12-8.test.ts` (new)

---

## Change Log

- 2026-04-13: Initial full-stack implementation — backend service/router/tests + frontend components/hook/i18n/tests (13/13 backend pass, 69/69 frontend pass)
- 2026-04-13: Addressed code review findings — 2 items resolved:
  - F1: UsageMeter.tsx i18n badges (useTranslations replacing hardcoded strings)
  - F2: page.tsx generateMetadata with getTranslations from next-intl/server
