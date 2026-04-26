# Story 12.7: Pipeline Forecasting Dashboard (Full Stack, Professional+ Tier)

Status: review

## Story

As a **Professional or Enterprise tier user on the EU Solicit platform**,
I want **a pipeline forecasting dashboard at `/analytics/pipeline`**,
so I can **view AI-generated predictions of upcoming tender opportunities matching my sectors.**

## Acceptance Criteria

1. **AC1 — Data Model**: `client.pipeline_predictions` table created via Alembic (012). Columns: `id` (UUID), `company_id` (UUID), `title` (text), `sector` (text), `estimated_value_min_eur` (numeric), `estimated_value_max_eur` (numeric), `predicted_publication_date` (date), `confidence_score` (numeric 0–100), `basis` (text), `generated_at` (timestamptz), `expires_at` (timestamptz).
2. **AC2 — SQLAlchemy Model**: Table registered in `models/analytics_views.py`.
3. **AC3 — Backend API**: `GET /api/v1/analytics/pipeline/forecast` returns `PaginatedResponse[PipelineForecastItem]`. `confidence_level` is derived in Python: high (≥75), medium (40–74), low (<40).
4. **AC4 — Tier Gate**: Endpoint requires Professional+ tier. Error message is generic: `"This feature requires a Professional or Enterprise subscription."`.
5. **AC5 — Tenant Isolation**: All queries scoped to `company_id` as the first filter.
6. **AC6 — Cache Headers**: Returns `Cache-Control: public, max-age=900`.
7. **AC7 — Backend Unit Tests**: Cover happy path, filters, isolation, tier gate, and confidence derivation.
8. **AC8 — Frontend Page**: Page shell at `/analytics/pipeline` renders the dashboard.
9. **AC9 — Filter Panel**: Sector, Value Min/Max, and Confidence Min filters with Apply/Clear.
10. **AC10 — Timeline View**: Month-grouped, date-sorted list of predictions with pagination.
11. **AC11 — Confidence Badges**: Color-coded badges: Green (High), Amber (Medium), Red (Low).
12. **AC12 — Empty State**: Renders when no predictions exist for the company.
13. **AC13 — Tier Upgrade Gate**: Renders 403 error as an upgrade prompt (reusing `TierUpgradeGate`).
14. **AC14 — i18n**: All 22 dashboard keys present in `en.json` and `bg.json`.
15. **AC15 — Frontend Structural Tests**: Verify components, testids, API logic, and i18n keys.
16. **AC16 — E2E P0 Tests**: Pipeline tests un-skipped in `tier-gate-enforcement.api.spec.ts` with fixed assertions.

## Tasks / Subtasks

### Backend

- [x] Task 1: Alembic migration 012 — create `client.pipeline_predictions` table (AC: 1)
  - [x] 1.1 Create `services/client-api/alembic/versions/012_pipeline_predictions.py` with `down_revision = "011"`
  - [x] 1.2 `upgrade()` — `CREATE TABLE IF NOT EXISTS client.pipeline_predictions` with all columns and CHECK constraint on `confidence_score` (see AC1)
  - [x] 1.3 `upgrade()` — create 3 indexes: `ix_pipeline_predictions_company_id`, `ix_pipeline_predictions_company_sector`, `ix_pipeline_predictions_company_date`
  - [x] 1.4 `downgrade()` — drop 3 indexes then `DROP TABLE IF EXISTS client.pipeline_predictions`

- [x] Task 2: SQLAlchemy Core Table model (AC: 2)
  - [x] 2.1 Add `pipeline_predictions` `sa.Table` definition to `services/client-api/src/client_api/models/analytics_views.py`
  - [x] 2.2 Export `pipeline_predictions` from `__all__` in `analytics_views.py`

- [x] Task 3: Update tier gate error message (AC: 4)
  - [x] 3.1 In `services/client-api/src/client_api/core/tier_gate.py`, change the `ForbiddenError` message inside `require_professional_plus_tier` from `"Competitor intelligence requires a Professional or Enterprise subscription."` to `"This feature requires a Professional or Enterprise subscription."`
  - [x] 3.2 Update the docstring `Raises` section to reflect the new generic message
  - [x] 3.3 Update the existing competitor unit test that asserts the old message (in `test_analytics_competitors.py` — find and update the assertion that checks the tier gate message)

- [x] Task 4: Pipeline Pydantic schemas — append to `schemas/analytics.py` (AC: 3)
  - [x] 4.1 Define `PipelineForecastItem(id: UUID, title: str, sector: str, estimated_value_min_eur: Decimal | None, estimated_value_max_eur: Decimal | None, predicted_publication_date: date, confidence_score: Decimal, confidence_level: Literal["high", "medium", "low"])`
  - [x] 4.2 `confidence_level` is a computed field (`@computed_field`) or a `model_validator` derived from `confidence_score`: ≥75 → `"high"`, ≥40 → `"medium"`, else `"low"`
  - [x] 4.3 Add `PipelineForecastItem` to `__all__` in `schemas/analytics.py`

- [x] Task 5: Pipeline analytics service — `services/analytics_pipeline_service.py` (AC: 3, 5)
  - [x] 5.1 Create `services/client-api/src/client_api/services/analytics_pipeline_service.py`
  - [x] 5.2 Implement `async get_pipeline_forecast(company_id: UUID, sector: str | None, value_min: Decimal | None, value_max: Decimal | None, confidence_min: Decimal | None, page: int, page_size: int, session: AsyncSession) -> tuple[list[PipelineForecastItem], int]`
  - [x] 5.3 Base query: `SELECT * FROM client.pipeline_predictions WHERE company_id = :company_id` — `company_id` scoping MUST be the first WHERE clause
  - [x] 5.4 Optional filters applied sequentially:
    - `sector`: `AND sector = :sector`
    - `confidence_min`: `AND confidence_score >= :confidence_min`
    - `value_min`: `AND (estimated_value_max_eur >= :value_min OR (estimated_value_max_eur IS NULL AND estimated_value_min_eur >= :value_min))`
    - `value_max`: `AND (estimated_value_min_eur <= :value_max OR (estimated_value_min_eur IS NULL AND estimated_value_max_eur <= :value_max))`
  - [x] 5.5 Order by: `predicted_publication_date ASC, confidence_score DESC`
  - [x] 5.6 Count query first (for `total`), then paginated result with `LIMIT :page_size OFFSET (:page - 1) * :page_size`
  - [x] 5.7 Build `PipelineForecastItem` from each row; `confidence_level` computed in Python from `confidence_score`

- [x] Task 6: Pipeline analytics router — `api/v1/analytics_pipeline.py` (AC: 3, 4, 6)
  - [x] 6.1 Create `services/client-api/src/client_api/api/v1/analytics_pipeline.py`
  - [x] 6.2 Single route: `GET /forecast` with `require_professional_plus_tier` as auth dependency
  - [x] 6.3 Query params: `sector: str | None`, `value_min: Decimal | None`, `value_max: Decimal | None`, `confidence_min: Decimal | None`, `page: int = 1`, `page_size: int = 20` (validate `page_size` 1–100)
  - [x] 6.4 Set `Cache-Control: public, max-age=900` on response
  - [x] 6.5 Return `PaginatedResponse[PipelineForecastItem]`

- [x] Task 7: Register router in `main.py` (AC: 3)
  - [x] 7.1 Import `analytics_pipeline` router and include with prefix `/api/v1/analytics/pipeline`

- [x] Task 8: Backend unit tests — `tests/api/test_analytics_pipeline.py` (AC: 7)
  - [x] 8.1 Create test file; mock `analytics_pipeline_service.get_pipeline_forecast`
  - [x] 8.2 Override `require_professional_plus_tier` with controlled `CurrentUser` (Professional+ and Starter tier variants)
  - [x] 8.3 Test: Professional+ token → 200 with correct `PaginatedResponse` shape + all 7 item fields
  - [x] 8.4 Test: `confidence_level` computed correctly for scores 80, 55, 25
  - [x] 8.5 Test: Starter tier → 403 with `upgrade_required: True`
  - [x] 8.6 Test: unauthenticated (no JWT) → 401
  - [x] 8.7 Test: cross-tenant isolation — service called with `company_id = current_user.company_id` (assert mock call args)
  - [x] 8.8 Test: empty result → 200 with `{"items": [], "total": 0, "page": 1, "page_size": 20}`
  - [x] 8.9 Test: sector filter passed to service layer
  - [x] 8.10 Test: confidence_min filter passed to service layer
  - [x] 8.11 Test: pagination window — `page=2, page_size=5`
  - [x] 8.12 Test: `Cache-Control: public, max-age=900` header present

### Frontend

- [x] Task 9: Update `lib/api/analytics.ts` with Pipeline interfaces and fetch function (AC: 3, 8)
  - [x] 9.1 Add `// ─── Pipeline Forecasting (S12.07) ──────` section to `analytics.ts`
  - [x] 9.2 Define `PipelineFilters { sector?: string; valueMin?: number; valueMax?: number; confidenceMin?: number }`
  - [x] 9.3 Define `PipelineForecastItem { id: string; title: string; sector: string; estimated_value_min_eur: string | null; estimated_value_max_eur: string | null; predicted_publication_date: string; confidence_score: string; confidence_level: "high" | "medium" | "low" }`
  - [x] 9.4 Define `buildPipelineParams(filters: PipelineFilters, extra?: Record<string, string | number>): Record<string, string | number>` helper
  - [x] 9.5 Define `fetchPipelineForecast(filters: PipelineFilters, page = 1, pageSize = 20): Promise<PaginatedResponse<PipelineForecastItem>>`; URL: `"/api/v1/analytics/pipeline/forecast"`

- [x] Task 10: TanStack Query hook — `lib/queries/use-pipeline-analytics.ts` (AC: 8)
  - [x] 10.1 Create `apps/client/lib/queries/use-pipeline-analytics.ts`
  - [x] 10.2 Export `usePipelineForecast(filters: PipelineFilters, page: number, pageSize = 20)` — `queryKey: ["pipeline-forecast", filters, page, pageSize]`; `staleTime: 900_000` (15 min, matches Cache-Control max-age=900)

- [x] Task 11: Add `analytics.pipeline` i18n keys to `en.json` and `bg.json` (AC: 14)
  - [x] 11.1 Add all 22 keys listed in AC14 to `frontend/apps/client/messages/en.json`
  - [x] 11.2 Add the same 22 keys (translated) to `frontend/apps/client/messages/bg.json`

- [x] Task 12: Create components in `app/[locale]/(protected)/analytics/pipeline/components/` (AC: 9–13)
  - [x] 12.1 `PipelineForecastFilters.tsx` — filter panel with sector, value min/max, confidence min inputs + Apply/Clear buttons; `data-testid="pipeline-filters"`
  - [x] 12.2 `PipelineForecastTimeline.tsx` — date-sorted, month-grouped list of `PipelineForecastItem` cards; renders `data-testid="pipeline-forecast-timeline"` and `data-testid="pipeline-month-divider"` headers; each card has `data-testid="pipeline-forecast-item"` and `data-testid="confidence-badge"`
  - [x] 12.3 `PipelineForecastEmpty.tsx` — empty state component with `data-testid="pipeline-forecast-empty"`, title, and description

- [x] Task 13: Dashboard orchestrator `PipelineForecastDashboard.tsx` (AC: 8–13)
  - [x] 13.1 Create `components/PipelineForecastDashboard.tsx` — `"use client"` component
  - [x] 13.2 Manage state: `filters: PipelineFilters`, `page: number`
  - [x] 13.3 Use `usePipelineForecast(filters, page)` hook; handle `isLoading`, `isError`, `data`, `error.status`
  - [x] 13.4 On HTTP 403 error: render `<div data-testid="pipeline-upgrade-gate"><TierUpgradeGate message={t("analytics.pipeline.upgradeMessage")} ctaHref="/settings/billing" /></div>` — import `TierUpgradeGate` from `../competitors/components/TierUpgradeGate`
  - [x] 13.5 When `data.items.length === 0` and no active error: render `<PipelineForecastEmpty />`
  - [x] 13.6 Otherwise: render `<PipelineForecastFilters />` + `<PipelineForecastTimeline items={data.items} />` + pagination controls

- [x] Task 14: Page shell `app/[locale]/(protected)/analytics/pipeline/page.tsx` (AC: 8)
  - [x] 14.1 Create server component file (no `"use client"`)
  - [x] 14.2 Set Next.js metadata: `title: t("analytics.pipeline.pageTitle")`, `description: t("analytics.pipeline.pageDescription")`
  - [x] 14.3 Render `<PipelineForecastDashboard />`

- [x] Task 15: Frontend unit tests — `apps/client/__tests__/pipeline-forecast-s12-7.test.ts` (AC: 15)
  - [x] 15.1 Structural tests for `PipelineForecastFilters`: file exists, exports component, `data-testid="pipeline-filters"` present, sector/confidence inputs present
  - [x] 15.2 Structural tests for `PipelineForecastTimeline`: file exists, exports component, `data-testid="pipeline-forecast-timeline"` present, `data-testid="pipeline-forecast-item"` present, `data-testid="confidence-badge"` present
  - [x] 15.3 Structural tests for `PipelineForecastEmpty`: file exists, `data-testid="pipeline-forecast-empty"` present
  - [x] 15.4 Structural test for `PipelineForecastDashboard`: renders upgrade gate wrapper `data-testid="pipeline-upgrade-gate"`
  - [x] 15.5 API function tests: `fetchPipelineForecast` exported from `lib/api/analytics.ts`; `buildPipelineParams` serialises `confidenceMin` to `confidence_min`
  - [x] 15.6 Hook test: `usePipelineForecast` exported from `lib/queries/use-pipeline-analytics.ts`; `staleTime` is `900_000`
  - [x] 15.7 i18n key tests: all 22 `analytics.pipeline.*` keys present in both `en.json` and `bg.json`

### E2E ATDD

- [x] Task 16: Un-skip pipeline P0 tests in `tier-gate-enforcement.api.spec.ts` (AC: 16)
  - [x] 16.1 Remove `test.skip(` from all 3 pipeline tests in the `'E12 P0: Tier Gate — Pipeline Forecasting (ATDD)'` describe block
  - [x] 16.2 Fix assertion at line 159: `expect(data.predictions).toBeDefined()` → `expect(data.items).toBeDefined()`
  - [x] 16.3 Fix 403 body assertions in Free/Starter tests: `expect(body.detail).toContain('Professional')` → `expect(body.message).toContain('Professional')`; `expect(body.upgrade_url).toBeTruthy()` → `expect(body.details?.upgrade_required).toBe(true)`

---

## Dev Notes

### Data Source: `client.pipeline_predictions`

The Pipeline Forecasting Agent (an AI agent, not yet in this story's scope to build) periodically generates predictions and writes them to `client.pipeline_predictions`. This story creates the table and reads from it. The table will be empty in staging until the agent is deployed and run; the empty-state UI and seeded test data in unit tests cover this scenario.

The `expires_at` column allows stale predictions to be cleaned up (future implementation: a Celery task removes rows where `expires_at < now()`). This story does not implement the cleanup task — the API should not filter by `expires_at` unless explicitly specified in a future story.

### Tier Gate — Existing `require_professional_plus_tier` (Task 3 Critical)

`require_professional_plus_tier` was implemented in S12.06 with a message specific to competitor intelligence: `"Competitor intelligence requires a Professional or Enterprise subscription."`. Since S12.07 shares the same gate, **update the message to a generic one** (Task 3.1). This update does not break the P0 ATDD tests (which only check the message contains `'Professional'`) but does require updating the unit test in `test_analytics_competitors.py` that asserts the specific old message (Task 3.3 — grep for `"Competitor intelligence requires"` in that file).

### Routing: Single Endpoint, No Routing Order Issue

Unlike the competitor router (which needed careful ordering to avoid FastAPI matching `/profiles` as a path parameter), the pipeline router has only one endpoint (`/forecast`), so routing order is not a concern.

### Confidence Level Derivation

`confidence_level` is computed from `confidence_score`:
```python
def _compute_confidence_level(score: Decimal) -> Literal["high", "medium", "low"]:
    if score >= 75:
        return "high"
    if score >= 40:
        return "medium"
    return "low"
```

Implement this as a `@model_validator(mode="after")` in `PipelineForecastItem`, or as a standalone function called during item construction in the service layer. Either approach is acceptable; choose whichever integrates cleanly with the Pydantic model.

### Value Range Filter Logic (Task 5.4)

The value range filter involves nullable columns:
- `value_min` filter: an opportunity is included if its `estimated_value_max_eur >= value_min`, OR if `estimated_value_max_eur` IS NULL but `estimated_value_min_eur >= value_min` (upper bound not set — include if lower bound qualifies)
- `value_max` filter: an opportunity is included if its `estimated_value_min_eur <= value_max`, OR if `estimated_value_min_eur` IS NULL but `estimated_value_max_eur <= value_max`

This is a simplified range-overlap check. Simplify further if needed: for MVP, `value_min` and `value_max` can filter on `estimated_value_min_eur` alone if nullable handling is complex. Document the simplification in a comment.

### Frontend: Reusing `TierUpgradeGate` from S12.06

`TierUpgradeGate.tsx` exists at `analytics/competitors/components/TierUpgradeGate.tsx` and accepts `message: string` and `ctaHref: string` props. Import it in `PipelineForecastDashboard.tsx`:

```typescript
import TierUpgradeGate from '../../competitors/components/TierUpgradeGate';
```

This cross-feature import is intentional for MVP. A future refactor can move `TierUpgradeGate` to `analytics/components/TierUpgradeGate.tsx` and update both competitors and pipeline to import from the shared location.

### Frontend: Upgrade Gate testid

`TierUpgradeGate` has `data-testid="competitor-upgrade-gate"` internally (from S12.06). Wrap the import in a `<div data-testid="pipeline-upgrade-gate">` in `PipelineForecastDashboard` so that tests can target the pipeline-specific testid without modifying the shared component.

### Frontend: Timeline View — Month Grouping

Group predictions by `predicted_publication_date` month/year:

```typescript
function groupByMonth(items: PipelineForecastItem[]) {
  return items.reduce((groups, item) => {
    const key = item.predicted_publication_date.substring(0, 7); // "YYYY-MM"
    if (!groups[key]) groups[key] = [];
    groups[key].push(item);
    return groups;
  }, {} as Record<string, PipelineForecastItem[]>);
}
```

Render a month header (`data-testid="pipeline-month-divider"`) before each group. Use `Intl.DateTimeFormat` with `"en-IE"` locale for consistent date formatting (consistent with `competitor-utils.ts` locale choice).

### isHttpError Utility

Use the `isHttpError(error, status)` utility from `lib/api/error-utils.ts` (created in S12.06) to detect 403 responses and show the upgrade gate. Do not use fragile type casts.

### Security: Cross-Tenant Isolation (E12-R-001)

`company_id` scoping is the primary defence against cross-tenant leakage. The `client.pipeline_predictions` table must always be queried with `WHERE company_id = current_user.company_id` as the **first** clause. This is an unconditional clause — it must never be wrapped in a conditional or made optional.

### Performance

`client.pipeline_predictions` is a small table per company (predictions for a rolling 6–12 month window). With the index on `(company_id, predicted_publication_date)`, queries should be fast even at scale. No materialized view is needed. Analytics target p95 < 500ms under staging load applies.

### ATDD Artifacts

- **Checklist:** `test_artifacts/atdd-checklist-12-7-pipeline-forecasting-dashboard-full-stack-professional-tier.md`
- **Backend unit tests:** `eusolicit-app/services/client-api/tests/api/test_analytics_pipeline.py` (15 tests, 🟢 GREEN — backend complete)
- **Frontend structural tests:** `eusolicit-app/frontend/apps/client/__tests__/pipeline-forecast-s12-7.test.ts` (81 tests, 🟢 GREEN — components verified)
- **E2E P0 tests:** `eusolicit-app/e2e/specs/analytics/tier-gate-enforcement.api.spec.ts` (3 tests, ⏳ awaiting staging fixtures)

**Active GREEN phase:** All unit and structural tests are passing. Verified backend implementation, database migrations, and frontend components.

---

### P0 ATDD Phase

The pipeline P0 tests in `tier-gate-enforcement.api.spec.ts` are in **Awaiting Fixtures phase** (un-skipped). Task 16 un-skipped them and fixed assertion bugs. The tests will not actually pass (GREEN phase) until:
1. The pipeline endpoint is deployed to staging
2. Real tier-seeded tokens are configured in the test fixtures
3. `client.pipeline_predictions` has seeded data for the Professional tier company

This is expected — the P0 ATDD pattern across E12 is that implementation stories un-skip the tests (moving from RED to "awaiting fixtures"), and the full GREEN phase is verified in staging. See `atdd-checklist-e12-p0.md` for the lifecycle documentation.

---

## File List

### Backend
- `services/client-api/alembic/versions/012_pipeline_predictions.py` — migration: create `client.pipeline_predictions` table and indexes
- `services/client-api/src/client_api/models/analytics_views.py` — add `pipeline_predictions` Table definition; export from `__all__`
- `services/client-api/src/client_api/core/tier_gate.py` — update `require_professional_plus_tier` error message to generic
- `services/client-api/src/client_api/schemas/analytics.py` — add `PipelineForecastItem` schema with `confidence_level` derivation
- `services/client-api/src/client_api/services/analytics_pipeline_service.py` — `get_pipeline_forecast` service function
- `services/client-api/src/client_api/api/v1/analytics_pipeline.py` — router: single `GET /forecast` route with tier gate and Cache-Control
- `services/client-api/src/client_api/main.py` — register analytics_pipeline router with prefix `/api/v1/analytics/pipeline`
- `services/client-api/tests/api/test_analytics_pipeline.py` — 12 backend unit tests (happy path, tier gate, isolation, filters, pagination, cache header)
- `services/client-api/tests/api/test_analytics_competitors.py` — update tier gate message assertion (3.3)

### Frontend
- `frontend/apps/client/lib/api/analytics.ts` — add `PipelineFilters`, `PipelineForecastItem`, `buildPipelineParams`, `fetchPipelineForecast`
- `frontend/apps/client/lib/queries/use-pipeline-analytics.ts` — `usePipelineForecast` hook (staleTime: 900_000)
- `frontend/apps/client/messages/en.json` — add `analytics.pipeline` namespace (22 keys)
- `frontend/apps/client/messages/bg.json` — add `analytics.pipeline` namespace (22 keys)
- `frontend/apps/client/app/[locale]/(protected)/analytics/pipeline/page.tsx` — server component page shell
- `frontend/apps/client/app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastDashboard.tsx` — orchestrator (filter state, page state, isHttpError gate, empty/timeline/upgrade routing)
- `frontend/apps/client/app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastFilters.tsx` — filter panel (`data-testid="pipeline-filters"`)
- `frontend/apps/client/app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastTimeline.tsx` — month-grouped timeline list (`data-testid="pipeline-forecast-timeline"`)
- `frontend/apps/client/app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastEmpty.tsx` — empty state (`data-testid="pipeline-forecast-empty"`)
- `frontend/apps/client/__tests__/pipeline-forecast-s12-7.test.ts` — frontend structural ATDD tests (81 assertions)

### E2E
- `eusolicit-app/e2e/specs/analytics/tier-gate-enforcement.api.spec.ts` — remove `test.skip()` from 3 pipeline tests; fix `data.predictions` → `data.items`; fix `body.detail` → `body.message`; fix `body.upgrade_url` → `body.details?.upgrade_required`

---

## Senior Developer Review

**Reviewer:** Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Date:** 2026-04-13
**Verdict:** ✅ APPROVED — All 16 acceptance criteria met.

### Review Findings

- [x] [Review][Defer] `Cache-Control: public` on tenant-scoped authenticated data — AC6 specifies `public`, but tenant-scoped responses risk cross-tenant leakage via shared CDN caches. Consider changing to `private` in a future story. [analytics_pipeline.py:29] — deferred, spec-level concern
- [x] [Review][Defer] No generic error state for non-403 API errors — 500/502/network errors render empty timeline with no error indication. No AC requires this. [PipelineForecastDashboard.tsx:107] — deferred, future UX improvement
- [x] [Review][Defer] Both value columns NULL silently excluded by value range filters — rows with both `estimated_value_min_eur` and `estimated_value_max_eur` NULL are excluded when any value filter is active. Documented as simplified MVP logic. [analytics_pipeline_service.py:84-102] — deferred, pre-existing design
- [x] [Review][Patch] NaN propagation from `Number()` conversion in `handleFilterChange` — non-numeric paste into number inputs produces NaN, sent to API as invalid Decimal → 422. Guarded with `Number.isNaN()` check in S12.07 implementation. [PipelineForecastDashboard.tsx:57-58]
- [x] [Review][Defer] `confidence_min` query param missing `ge=0, le=100` server-side bounds — out-of-range values (e.g., 150) return empty results silently. [analytics_pipeline.py:43] — deferred, non-breaking
- [x] [Review][Defer] `value_min`/`value_max` missing `ge=0` server-side bounds — negative monetary values accepted. HTML `min=0` is advisory only. [analytics_pipeline.py:41-42] — deferred, non-breaking
- [x] [Review][Defer] Page metadata uses static strings not `generateMetadata()` with i18n — Bulgarian users see English metadata in browser tab. Next.js static `Metadata` export limitation. [page.tsx:4-8] — deferred, requires `generateMetadata()` refactor
- [x] [Review][Defer] `value_min > value_max` not validated server-side — contradictory range silently returns 0 results. [analytics_pipeline.py:40-43] — deferred, UX polish

**Summary:** 0 `decision-needed`, 0 `patch`, 8 `defer`, 14 dismissed as noise.
All acceptance criteria are satisfied.

---

## Dev Agent Record

**Implemented by:** Gemini 2.0 Pro Experimental + session-0a896e6c-ae07-4954-bd00-6196e9cd1888
**File List:**
- **Modified:**
  - `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastDashboard.tsx` (Applied NaN guard patch)
  - `eusolicit-docs/implementation-artifacts/12-7-pipeline-forecasting-dashboard-full-stack-professional-tier.md` (Updated status and task list)
  - `eusolicit-docs/implementation-artifacts/sprint-status.yaml` (Updated status to review)
**Test Results:** 
- Backend: `15 passed, 7 warnings in 2.30s`
- Frontend: `81 passed` (Vitest)
- Competitor Regression: `22 passed, 7 warnings in 2.36s`

---

## Change Log

- **2026-04-13**: Story created from Epic 12 S12.07 requirements. Data model designed as `client.pipeline_predictions` (Alembic 012). Tier gate message update noted as Task 3. P0 ATDD un-skip documented in AC16 with assertion fixes. Frontend reuses `TierUpgradeGate` from S12.06.
- **2026-04-25**: Verified existing implementation. Applied NaN guard patch to `PipelineForecastDashboard.tsx`. Confirmed all unit and structural tests pass. Un-skipped P0 E2E tests verified in source. Status moved to `review`.
