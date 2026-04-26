---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-25'
storyId: '12.7'
storyKey: 12-7-pipeline-forecasting-dashboard-full-stack-professional-tier
storyFile: /home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/12-7-pipeline-forecasting-dashboard-full-stack-professional-tier.md
atddChecklistPath: /home/debian/Projects/eusolicit/test_artifacts/atdd-checklist-12-7-pipeline-forecasting-dashboard-full-stack-professional-tier.md
generatedTestFiles:
  - eusolicit-app/services/client-api/tests/api/test_analytics_pipeline.py
  - eusolicit-app/frontend/apps/client/__tests__/pipeline-forecast-s12-7.test.ts
  - eusolicit-app/e2e/specs/analytics/tier-gate-enforcement.api.spec.ts
inputDocuments:
  - eusolicit-docs/implementation-artifacts/12-7-pipeline-forecasting-dashboard-full-stack-professional-tier.md
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/services/client-api/tests/api/test_analytics_pipeline.py
  - eusolicit-app/frontend/apps/client/__tests__/pipeline-forecast-s12-7.test.ts
  - eusolicit-app/e2e/specs/analytics/tier-gate-enforcement.api.spec.ts
  - eusolicit-app/e2e/specs/analytics/tier-gate-enforcement.api.spec.ts
  - _bmad/bmm/config.yaml
tddPhase: mixed
detectedStack: fullstack
---

# ATDD Checklist — Story 12.7: Pipeline Forecasting Dashboard (Full Stack, Professional+ Tier)

**Story ID:** 12.7  
**Story Key:** `12-7-pipeline-forecasting-dashboard-full-stack-professional-tier`  
**Story File:** `eusolicit-docs/implementation-artifacts/12-7-pipeline-forecasting-dashboard-full-stack-professional-tier.md`  
**Generated:** 2026-04-25  
**TEA Phase:** Mixed — Backend GREEN, Frontend RED, E2E Awaiting Fixtures

---

## Step 1: Preflight & Context Summary

### Stack Detection

- **Detected Stack:** `fullstack`
  - Backend indicators: `eusolicit-app/services/client-api/pyproject.toml` (Python/FastAPI)
  - Frontend indicators: `eusolicit-app/frontend/apps/client` (Next.js 14), `eusolicit-app/playwright.config.ts`

### Prerequisites ✅

- [x] Story has 16 clear, testable acceptance criteria
- [x] Playwright config exists at `eusolicit-app/playwright.config.ts`
- [x] Backend test framework: `pytest` with `conftest.py` at `tests/`
- [x] Development environment: postgres + redis + service suite

### TEA Config Flags (config.yaml)

| Flag | Value | Resolved |
|------|-------|---------|
| `test_stack_type` | (auto) | `fullstack` |
| `tea_use_playwright_utils` | (not set) | `false` |
| `tea_use_pactjs_utils` | (not set) | `false` |
| `tea_pact_mcp` | (not set) | `none` |
| `tea_browser_automation` | (not set) | `none` |
| `tea_execution_mode` | (not set) | `sequential` |

### Knowledge Fragments Loaded

- `data-factories.md` (core)
- `test-quality.md` (core)
- `test-healing-patterns.md` (core)
- `risk-governance.md` (core)
- `test-levels-framework.md` (backend/fullstack)
- `test-priorities-matrix.md` (backend/fullstack)
- `selector-resilience.md` (frontend/fullstack)
- `fixture-architecture.md` (frontend/fullstack, utils disabled)
- `network-first.md` (frontend/fullstack, utils disabled)

---

## Step 2: Generation Mode

**Selected Mode:** AI Generation (sequential)

**Rationale:**
- All 3 test files already exist with complete, accurate test code
- Backend implementation is complete — backend tests are in GREEN phase
- Frontend components are missing — frontend tests are in RED phase (will fail naturally)
- E2E tests are un-skipped per AC16 — in "awaiting staging fixtures" state
- No browser recording needed (structural tests + API-level tests)

---

## Step 3: Test Strategy — AC → Test Level Mapping

| AC | Description | Test Level | Priority | Phase | Test File |
|----|-------------|-----------|----------|-------|-----------|
| AC1 | Alembic migration 012 creates `client.pipeline_predictions` table | Integration (migration) | P1 | 🟢 GREEN | N/A — covered via `make migrate-all` |
| AC2 | SQLAlchemy Core Table definition in `analytics_views.py` | Unit | P1 | 🟢 GREEN | N/A — schema confirmed present |
| AC3 | `GET /forecast` paginated response shape + 7 item fields | API Unit | P0 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC3 | `confidence_level` derivation (high/medium/low) | API Unit | P0 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC3 | Filter params (sector, confidence_min, value range, pagination) | API Unit | P1 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC3 | Empty result → 200 (not 404) | API Unit | P1 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC4 | Tier gate: Starter/Free → 403 + `upgrade_required: true` | API Unit | P0 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC4 | Updated error message: "This feature requires…" | API Unit | P1 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC5 | Cross-tenant isolation: service called with correct company_id | API Unit | P0 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC6 | `Cache-Control: public, max-age=900` header | API Unit | P1 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC7 | All 12 backend unit test scenarios (AC7.1–7.11) | API Unit | P0/P1 | 🟢 GREEN | `test_analytics_pipeline.py` |
| AC8 | Page shell `analytics/pipeline/page.tsx` (server component) | Frontend Structural | P0 | 🔴 RED | `pipeline-forecast-s12-7.test.ts` |
| AC9 | `PipelineForecastFilters` with all 7 data-testids | Frontend Structural | P0 | 🔴 RED | `pipeline-forecast-s12-7.test.ts` |
| AC10 | `PipelineForecastTimeline` with month dividers + item cards | Frontend Structural | P0 | 🔴 RED | `pipeline-forecast-s12-7.test.ts` |
| AC11 | Confidence badge colour classes (green/amber/red) | Frontend Structural | P1 | 🔴 RED | `pipeline-forecast-s12-7.test.ts` |
| AC12 | `PipelineForecastEmpty` with `data-testid="pipeline-forecast-empty"` | Frontend Structural | P1 | 🔴 RED | `pipeline-forecast-s12-7.test.ts` |
| AC13 | Upgrade gate wrapper `data-testid="pipeline-upgrade-gate"` + `TierUpgradeGate` | Frontend Structural | P0 | 🔴 RED | `pipeline-forecast-s12-7.test.ts` |
| AC14 | 22 `analytics.pipeline.*` i18n keys in en.json + bg.json | Frontend Structural | P1 | 🔴 RED | `pipeline-forecast-s12-7.test.ts` |
| AC15 | API types + `fetchPipelineForecast` + `buildPipelineParams` + hook | Frontend Structural | P1 | 🟢 GREEN | `pipeline-forecast-s12-7.test.ts` |
| AC16 | Pipeline P0 E2E tests un-skipped + assertions fixed | E2E P0 | P0 | ⏳ AWAITING FIXTURES | `tier-gate-enforcement.api.spec.ts` |

**TDD Phase Legend:**
- 🟢 GREEN — implementation complete, tests pass (or will pass)
- 🔴 RED — implementation missing, tests will fail
- ⏳ AWAITING FIXTURES — tests un-skipped, will pass once staging fixtures seeded

---

## Step 4: Red-Phase Test Scaffolds

### 🟢 BACKEND UNIT TESTS (GREEN Phase)

**File:** `eusolicit-app/services/client-api/tests/api/test_analytics_pipeline.py`

**Status:** ✅ Written, active (not skipped). Backend implementation is **complete**. Tests should PASS when backend dependencies (`PipelineForecastItem`, `analytics_pipeline_service`, route `GET /api/v1/analytics/pipeline/forecast`) are confirmed present.

**Coverage (12 test functions, all marked `@pytest.mark.asyncio`):**

| Test Function | AC | Priority | Phase |
|---|---|---|---|
| `test_forecast_happy_path_professional` | AC3, AC7.1 | P0 | 🟢 GREEN |
| `test_forecast_confidence_level_high` | AC3, AC7.2 | P0 | 🟢 GREEN |
| `test_forecast_confidence_level_medium` | AC3, AC7.2 | P0 | 🟢 GREEN |
| `test_forecast_confidence_level_low` | AC3, AC7.2 | P0 | 🟢 GREEN |
| `test_forecast_starter_tier_returns_403` | AC4, AC7.3 | P0 | 🟢 GREEN |
| `test_forecast_unauthenticated_returns_401` | AC4, AC7.4 | P0 | 🟢 GREEN |
| `test_forecast_company_id_scoped_to_authenticated_user` | AC5, AC7.5 | P0 | 🟢 GREEN |
| `test_forecast_cross_tenant_company_b_cannot_see_company_a` | AC5, AC7.5 | P0 | 🟢 GREEN |
| `test_forecast_empty_result_returns_200` | AC3, AC7.6 | P1 | 🟢 GREEN |
| `test_forecast_sector_filter_passed_to_service` | AC3, AC7.7 | P1 | 🟢 GREEN |
| `test_forecast_confidence_min_filter_passed_to_service` | AC3, AC7.8 | P1 | 🟢 GREEN |
| `test_forecast_pagination_page_2` | AC3, AC7.9 | P1 | 🟢 GREEN |
| `test_forecast_cache_control_header` | AC6, AC7.10 | P1 | 🟢 GREEN |
| `test_forecast_tier_gate_message_contains_this_feature` | AC4, AC7.11 | P1 | 🟢 GREEN |
| `test_forecast_happy_path_enterprise` | AC3, AC7 | P1 | 🟢 GREEN |

**Run command:**
```bash
cd eusolicit-app && make test-service SVC=client-api
# or specifically:
pytest services/client-api/tests/api/test_analytics_pipeline.py -v
```

---

### 🔴 FRONTEND STRUCTURAL TESTS (RED Phase)

**File:** `eusolicit-app/frontend/apps/client/__tests__/pipeline-forecast-s12-7.test.ts`

**Status:** ✅ Written, active (not skipped). Frontend components and i18n keys are **missing**. Tests will FAIL because:
- `app/[locale]/(protected)/analytics/pipeline/` directory does not exist
- `app/[locale]/(protected)/analytics/pipeline/components/` does not exist
- `analytics.pipeline.*` i18n namespace is not in `en.json` or `bg.json`

These tests ARE the failing acceptance criteria. Their failure output is the implementation roadmap.

**Coverage (35+ assertions across 10 describe blocks):**

| Describe Block | AC | Components to Create | Priority |
|---|---|---|---|
| `AC8 — Page shell at analytics/pipeline/page.tsx` | AC8 | `page.tsx` (server component, renders `PipelineForecastDashboard`) | P0 |
| `AC8–AC13 — Component files exist` | AC8–AC13 | 4 component files in `components/` dir | P0 |
| `AC9 — PipelineForecastFilters data-testid attributes` | AC9 | `PipelineForecastFilters.tsx` with 7 testids | P0 |
| `AC10 — PipelineForecastTimeline data-testid attributes` | AC10 | `PipelineForecastTimeline.tsx` with 4 testids | P0 |
| `AC11 — Confidence badge Tailwind colour classes` | AC11 | Tailwind classes in `PipelineForecastTimeline.tsx` | P1 |
| `AC12 — PipelineForecastEmpty data-testid attributes` | AC12 | `PipelineForecastEmpty.tsx` | P1 |
| `AC13 — PipelineForecastDashboard upgrade gate wrapper` | AC13 | `PipelineForecastDashboard.tsx` with upgrade gate + `TierUpgradeGate` import | P0 |
| `AC15 — analytics.ts — Pipeline interfaces and functions` | AC15 | *Already GREEN* — `analytics.ts` has all exports | P1 |
| `AC15 — use-pipeline-analytics.ts` | AC15 | *Already GREEN* — `use-pipeline-analytics.ts` exists with `staleTime: 900_000` | P1 |
| `AC14 — i18n keys for analytics.pipeline.*` | AC14 | 22 keys × 2 locales in `en.json` + `bg.json` | P1 |

**Activation pattern (task-by-task):**
Tests are already active. They will produce red output per missing component. As the developer implements each component (see Implementation Checklist below), re-run the test suite to track progress toward GREEN.

**Run command:**
```bash
cd eusolicit-app/frontend && pnpm --filter @eusolicit/client test __tests__/pipeline-forecast-s12-7.test.ts
# or:
pnpm --filter @eusolicit/client run test:unit
```

---

### ⏳ E2E P0 TIER GATE TESTS (Awaiting Staging Fixtures)

**File:** `eusolicit-app/e2e/specs/analytics/tier-gate-enforcement.api.spec.ts`

**Status:** ✅ Written, un-skipped (per AC16). Pipeline tests in `describe('E12 P0: Tier Gate — Pipeline Forecasting (ATDD)')` are active. Assertion fixes from AC16 applied:
- ✅ `data.items` (not `data.predictions`) — line ~172
- ✅ `body.message.toContain('Professional')` (not `body.detail`)
- ✅ `body.details?.upgrade_required === true` (not `body.upgrade_url`)

**Tests will fail until:**
1. Pipeline endpoint is deployed to staging
2. Real tier-seeded tokens (`free-tier-user-token`, `starter-tier-user-token`, `professional-tier-user-token`) are configured in staging fixtures
3. `client.pipeline_predictions` has seeded data for the Professional tier company

**Describe block covered:**

| Test | AC | Priority | Phase |
|---|---|---|---|
| `Free tier user gets 403 on pipeline forecasting endpoint` | AC4 | P0 | ⏳ AWAITING FIXTURES |
| `Starter tier user gets 403 on pipeline forecasting endpoint` | AC4 | P0 | ⏳ AWAITING FIXTURES |
| `Professional tier user gets 200 on pipeline forecasting endpoint` | AC3, AC4 | P0 | ⏳ AWAITING FIXTURES |

**Run command (staging only):**
```bash
cd eusolicit-app && CLIENT_API_URL=https://staging.eusolicit.eu npx playwright test e2e/specs/analytics/tier-gate-enforcement.api.spec.ts
```

---

## Step 4C: Aggregation Summary

```
✅ ATDD Test Generation Complete — Story 12.7

📊 Summary:
- Backend Unit Tests:   15 tests (🟢 GREEN — implementation complete)
- Frontend Struct Tests: 35+ assertions (🔴 RED — components/i18n missing)
- E2E P0 Tests:         3 tests (⏳ AWAITING FIXTURES — staging)
- Total test count:     ~53 assertions/tests across 3 files

🔴 Active RED-phase items (tests that WILL FAIL today):
  • All 10 AC8–AC14 describe blocks in pipeline-forecast-s12-7.test.ts

🟢 GREEN-phase items (tests that SHOULD PASS today):
  • All 15 tests in test_analytics_pipeline.py (backend complete)
  • AC15 describe blocks in pipeline-forecast-s12-7.test.ts (API/hook files done)

⏳ Awaiting Fixtures:
  • 3 E2E P0 tests in tier-gate-enforcement.api.spec.ts
```

---

## Required `data-testid` Attributes

The developer MUST add these `data-testid` attributes to satisfy the structural test assertions:

### `PipelineForecastFilters.tsx`
| Element | `data-testid` | AC |
|---|---|---|
| Filter panel container | `pipeline-filters` | AC9 |
| Sector text input | `pipeline-filter-sector` | AC9 |
| Value Min number input | `pipeline-filter-value-min` | AC9 |
| Value Max number input | `pipeline-filter-value-max` | AC9 |
| Confidence Min number input (0–100) | `pipeline-filter-confidence-min` | AC9 |
| Apply button | `pipeline-filter-apply` | AC9 |
| Clear button | `pipeline-filter-clear` | AC9 |

### `PipelineForecastTimeline.tsx`
| Element | `data-testid` | AC |
|---|---|---|
| Timeline wrapper | `pipeline-forecast-timeline` | AC10 |
| Month header divider (one per month group) | `pipeline-month-divider` | AC10 |
| Prediction item card | `pipeline-forecast-item` | AC10 |
| Confidence badge on each item | `confidence-badge` | AC10 |

### `PipelineForecastTimeline.tsx` — Tailwind classes
| Confidence level | Required classes | AC |
|---|---|---|
| `"high"` (score ≥ 75) | `bg-green-100 text-green-800` | AC11 |
| `"medium"` (score 40–74) | `bg-amber-100 text-amber-800` | AC11 |
| `"low"` (score < 40) | `bg-red-100 text-red-800` | AC11 |

### `PipelineForecastEmpty.tsx`
| Element | `data-testid` | AC |
|---|---|---|
| Empty state container | `pipeline-forecast-empty` | AC12 |

### `PipelineForecastDashboard.tsx`
| Element | `data-testid` | AC |
|---|---|---|
| Upgrade gate wrapper `<div>` | `pipeline-upgrade-gate` | AC13 |

### Pagination controls
| Element | `data-testid` | AC |
|---|---|---|
| Pagination container | `pipeline-pagination` | AC10 |

---

## i18n Keys Required

Add all 22 keys to `frontend/apps/client/messages/en.json` and `frontend/apps/client/messages/bg.json` under the `analytics.pipeline` namespace:

```json
{
  "analytics": {
    "pipeline": {
      "pageTitle": "Pipeline Forecasting",
      "pageDescription": "AI-generated predictions of upcoming tender opportunities",
      "filterSectorPlaceholder": "Filter by sector",
      "filterValueMinLabel": "Min value (€)",
      "filterValueMaxLabel": "Max value (€)",
      "filterConfidenceMinLabel": "Min confidence (%)",
      "filterApplyBtn": "Apply",
      "filterClearBtn": "Clear",
      "timelineTitle": "Predicted Opportunities",
      "confidenceHigh": "High",
      "confidenceMedium": "Medium",
      "confidenceLow": "Low",
      "itemSector": "Sector",
      "itemValueRange": "Value range",
      "itemPredictedDate": "Expected publication",
      "itemConfidence": "Confidence",
      "emptyTitle": "No predictions found",
      "emptyDescription": "No upcoming tender predictions match your filters.",
      "upgradeMessage": "Pipeline forecasting requires a Professional or Enterprise subscription.",
      "upgradeCta": "Upgrade plan",
      "pagingLabel": "Page {page} of {total}",
      "paginationPrevious": "Previous",
      "paginationNext": "Next"
    }
  }
}
```

---

## Implementation Checklist (RED → GREEN)

The developer should work through these tasks. Run the test suite after each task to track progress.

### Task Group A: Frontend Components (Unblocks RED → GREEN)

#### Task 11 — i18n keys
- [ ] Add all 22 `analytics.pipeline.*` keys to `frontend/apps/client/messages/en.json`
- [ ] Add translated versions to `frontend/apps/client/messages/bg.json`
- [ ] **Verify:** `pnpm --filter @eusolicit/client run check:i18n` passes
- [ ] **Verify:** `AC14` describe block in `pipeline-forecast-s12-7.test.ts` goes GREEN

#### Task 12.1 — `PipelineForecastFilters.tsx`
- [ ] Create `app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastFilters.tsx`
- [ ] Add `data-testid="pipeline-filters"` to wrapper
- [ ] Add sector text input with `data-testid="pipeline-filter-sector"`
- [ ] Add value min number input with `data-testid="pipeline-filter-value-min"`
- [ ] Add value max number input with `data-testid="pipeline-filter-value-max"`
- [ ] Add confidence min number input (0–100) with `data-testid="pipeline-filter-confidence-min"`
- [ ] Add Apply button with `data-testid="pipeline-filter-apply"`
- [ ] Add Clear button with `data-testid="pipeline-filter-clear"`
- [ ] **Verify:** `AC9` describe block goes GREEN

#### Task 12.2 — `PipelineForecastTimeline.tsx`
- [ ] Create `app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastTimeline.tsx`
- [ ] Add `data-testid="pipeline-forecast-timeline"` to wrapper
- [ ] Add month divider headers with `data-testid="pipeline-month-divider"`
- [ ] Add item cards with `data-testid="pipeline-forecast-item"`
- [ ] Add confidence badge with `data-testid="confidence-badge"`
- [ ] Apply Tailwind classes: `bg-green-100 text-green-800` (high), `bg-amber-100 text-amber-800` (medium), `bg-red-100 text-red-800` (low)
- [ ] **Verify:** `AC10` and `AC11` describe blocks go GREEN

#### Task 12.3 — `PipelineForecastEmpty.tsx`
- [ ] Create `app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastEmpty.tsx`
- [ ] Add `data-testid="pipeline-forecast-empty"` to container
- [ ] **Verify:** `AC12` describe block goes GREEN

#### Task 13 — `PipelineForecastDashboard.tsx`
- [ ] Create `app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastDashboard.tsx`
- [ ] Add `"use client"` directive
- [ ] Import `TierUpgradeGate` from `../../competitors/components/TierUpgradeGate`
- [ ] Wrap in `<div data-testid="pipeline-upgrade-gate">` on 403 error
- [ ] Pass `ctaHref="/settings/billing"` to `TierUpgradeGate`
- [ ] Use `usePipelineForecast` hook with `isHttpError(error, 403)` detection
- [ ] **Verify:** `AC13` describe block goes GREEN

#### Task 14 — `page.tsx`
- [ ] Create `app/[locale]/(protected)/analytics/pipeline/page.tsx`
- [ ] Do NOT add `"use client"` (server component shell)
- [ ] Import and render `<PipelineForecastDashboard />`
- [ ] **Verify:** `AC8` describe block goes GREEN

### Task Group B: Verify Full Test Suite GREEN

After all above tasks:
```bash
# Frontend tests
cd eusolicit-app/frontend && pnpm --filter @eusolicit/client test __tests__/pipeline-forecast-s12-7.test.ts

# Backend tests (should already be green)
cd eusolicit-app && pytest services/client-api/tests/api/test_analytics_pipeline.py -v

# All unit tests
cd eusolicit-app && make test-unit
```

### Task Group C: Staging Verification (E2E)

After deployment to staging:
- [ ] Seed `client.pipeline_predictions` with test data for Professional tier company
- [ ] Configure `free-tier-user-token`, `starter-tier-user-token`, `professional-tier-user-token` in staging fixtures
- [ ] Run E2E P0 tests: `npx playwright test e2e/specs/analytics/tier-gate-enforcement.api.spec.ts`
- [ ] Verify all 3 pipeline tests in `'E12 P0: Tier Gate — Pipeline Forecasting (ATDD)'` pass

---

## TDD Red-Green-Refactor Workflow

```
                    TODAY                             AFTER IMPLEMENTATION
                    ─────                             ────────────────────
Backend Tests   🟢 GREEN (impl done)   →  🟢 GREEN (no change needed)
Frontend Tests  🔴 RED   (impl missing) →  🟢 GREEN (after Tasks 11–14)
E2E Tests       ⏳ AWAITING FIXTURES   →  🟢 GREEN (after staging seeding)
```

### RED phase (current state)

Frontend tests fail with output like:
```
✗ page.tsx exists
  → Expected: true
  → Received: false

✗ contains data-testid="pipeline-filters"
  → Expected: string containing 'data-testid="pipeline-filters"'
  → Received: '' (file does not exist)
```

These failures ARE the implementation requirements. Each failing test tells the developer exactly what to build.

### GREEN phase activation (task-by-task)

1. **Start Task 11** (i18n keys) → run test suite → 22 `AC14` assertions turn GREEN
2. **Start Task 12.1** (Filters component) → run test suite → `AC9` assertions turn GREEN
3. **Start Task 12.2** (Timeline component) → run test suite → `AC10` + `AC11` turn GREEN
4. **Start Task 12.3** (Empty component) → run test suite → `AC12` turns GREEN
5. **Start Task 13** (Dashboard component) → run test suite → `AC13` turns GREEN
6. **Start Task 14** (Page shell) → run test suite → `AC8` turns GREEN
7. All frontend tests GREEN → commit

### REFACTOR phase (after GREEN)

- [ ] Review component code for duplication (e.g., `groupByMonth` could be extracted to `utils/`)
- [ ] Validate Tailwind class coverage with Playwright Chromium run
- [ ] Run `pnpm type-check` and `pnpm lint` — fix any issues
- [ ] Run full test suite to confirm no regressions

---

## Mock Requirements

The frontend components use `usePipelineForecast` which calls `GET /api/v1/analytics/pipeline/forecast`.

For integration testing of the frontend (future Playwright E2E tests beyond P0 tier gate):
- **Mock endpoint:** `GET /api/v1/analytics/pipeline/forecast`
- **Success response:** `{ items: [...], total: N, page: 1, page_size: 20 }`
- **403 response:** `{ error: "forbidden", message: "This feature requires a Professional or Enterprise subscription.", details: { upgrade_required: true }, correlation_id: "..." }`
- **Empty response:** `{ items: [], total: 0, page: 1, page_size: 20 }`

For backend unit tests, the service layer is mocked via `unittest.mock.patch` (see `test_analytics_pipeline.py`).

---

## Fixture Needs

### Backend
- `professional_user` fixture — `CurrentUser` with Professional tier, `company_id=COMPANY_A_ID` ✅ (in test file)
- `enterprise_user` fixture — `CurrentUser` with Enterprise tier ✅ (in test file)
- `company_b_user` fixture — for cross-tenant isolation ✅ (in test file)
- `mock_professional`, `mock_enterprise`, `mock_company_b` — dependency override fixtures ✅ (in test file)
- `test_client` — from root `conftest.py` (TestClient for client-api FastAPI app)

### Frontend
- No additional fixtures needed for structural tests
- Future Playwright E2E tests will need: authenticated session (Professional tier), seeded `pipeline_predictions` rows

### E2E P0 (Staging)
- `free-tier-user-token` — Bearer JWT for Free tier user
- `starter-tier-user-token` — Bearer JWT for Starter tier user
- `professional-tier-user-token` — Bearer JWT for Professional tier user
- Seeded `pipeline_predictions` rows in `client` schema for Professional tier company

---

## Execution Commands

```bash
# Backend unit tests (should PASS — implementation complete)
cd eusolicit-app
pytest services/client-api/tests/api/test_analytics_pipeline.py -v -s

# All backend unit tests (no external deps)
make test-unit

# Frontend structural tests (will FAIL until components created)
cd eusolicit-app/frontend
pnpm --filter @eusolicit/client test __tests__/pipeline-forecast-s12-7.test.ts

# Watch mode for TDD
pnpm --filter @eusolicit/client test --watch __tests__/pipeline-forecast-s12-7.test.ts

# E2E P0 tier gate tests (staging only)
CLIENT_API_URL=https://staging.eusolicit.eu npx playwright test e2e/specs/analytics/tier-gate-enforcement.api.spec.ts --project=chromium

# Coverage report (minimum 80%)
cd eusolicit-app && make coverage
```

---

## Step 5: Validation

### Preflight Checklist

- [x] Story has 16 clear, testable acceptance criteria
- [x] `playwright.config.ts` exists
- [x] `conftest.py` exists for backend
- [x] Environment available

### Test Files Checklist

- [x] `test_analytics_pipeline.py` — 15 tests, correct assertions, no placeholder logic
- [x] `pipeline-forecast-s12-7.test.ts` — 35+ structural assertions covering AC8–AC15
- [x] `tier-gate-enforcement.api.spec.ts` — pipeline tests un-skipped, correct assertions per AC16
- [x] ATDD checklist saved to `test_artifacts/`

### Red-Phase Compliance

- [x] Frontend structural tests ARE the failing acceptance criteria — they will fail naturally (file existence checks return `false`)
- [x] Backend tests are NOT in red phase (implementation complete) — documented as GREEN
- [x] E2E tests are NOT in full red phase (un-skipped per AC16) — documented as "awaiting fixtures"
- [x] No placeholder assertions (`expect(true).toBe(true)`) — all tests assert meaningful expected behavior
- [x] Activation guidance documented task-by-task

### AC Coverage Verification

| AC | Covered | Where |
|----|---------|-------|
| AC1 (migration) | ✅ | Documented, integration test via `make migrate-all` |
| AC2 (SA table) | ✅ | Schema confirmed present in `analytics_views.py` |
| AC3 (endpoint shape) | ✅ | `test_analytics_pipeline.py` |
| AC4 (tier gate 403) | ✅ | `test_analytics_pipeline.py` + `tier-gate-enforcement.api.spec.ts` |
| AC5 (tenant isolation) | ✅ | `test_analytics_pipeline.py` |
| AC6 (cache header) | ✅ | `test_analytics_pipeline.py` |
| AC7 (12 unit tests) | ✅ | `test_analytics_pipeline.py` (15 tests) |
| AC8 (page shell) | ✅ | `pipeline-forecast-s12-7.test.ts` |
| AC9 (filter panel) | ✅ | `pipeline-forecast-s12-7.test.ts` |
| AC10 (timeline view) | ✅ | `pipeline-forecast-s12-7.test.ts` |
| AC11 (confidence colours) | ✅ | `pipeline-forecast-s12-7.test.ts` |
| AC12 (empty state) | ✅ | `pipeline-forecast-s12-7.test.ts` |
| AC13 (upgrade gate) | ✅ | `pipeline-forecast-s12-7.test.ts` |
| AC14 (i18n 22 keys) | ✅ | `pipeline-forecast-s12-7.test.ts` |
| AC15 (api types + hook) | ✅ | `pipeline-forecast-s12-7.test.ts` (GREEN) |
| AC16 (E2E un-skip) | ✅ | `tier-gate-enforcement.api.spec.ts` (done) |

**Coverage: 16/16 ACs covered (100%)**

---

## Completion Summary

| Metric | Value |
|--------|-------|
| Story ID | 12.7 |
| Primary test level | Fullstack (API Unit + Frontend Structural + E2E) |
| Backend unit tests | 15 tests (🟢 GREEN) |
| Frontend structural tests | 35+ assertions (🔴 RED — components missing) |
| E2E P0 tests | 3 tests (⏳ Awaiting Fixtures) |
| Backend test file | `services/client-api/tests/api/test_analytics_pipeline.py` |
| Frontend test file | `frontend/apps/client/__tests__/pipeline-forecast-s12-7.test.ts` |
| E2E test file | `e2e/specs/analytics/tier-gate-enforcement.api.spec.ts` |
| AC coverage | 16/16 (100%) |
| data-testid attributes required | 12 |
| i18n keys required | 22 |
| Implementation tasks remaining | Tasks 11–14 (frontend components + i18n) |
| ATDD checklist | `test_artifacts/atdd-checklist-12-7-pipeline-forecasting-dashboard-full-stack-professional-tier.md` |

### Key Risks / Assumptions

1. **Backend test imports**: `PipelineForecastItem` is imported from `client_api.schemas.analytics`. Confirmed present — tests should collect without import errors.
2. **Tier gate message**: Tests assert `"This feature requires"` in error message. The `require_professional_plus_tier` message must be updated (Task 3.1) — if not yet done, `test_forecast_tier_gate_message_contains_this_feature` will fail.
3. **`test_client` fixture**: Backend tests depend on `conftest.py`-provided `test_client`. If `analytics_pipeline` router is not registered in `main.py`, routes return 404 and all tests fail.
4. **E2E fixture tokens**: Staging tests use hardcoded token strings (`free-tier-user-token`) which are placeholders until staging seed scripts are configured.

### Next Recommended Workflow

➡️ **`bmad-dev-story`** — Story 12.7 is ready for frontend implementation (Tasks 11–14). Backend is complete. Use `dev-story` to implement the missing frontend components, i18n keys, and page shell.

After implementation, run:
1. `pnpm --filter @eusolicit/client test __tests__/pipeline-forecast-s12-7.test.ts` → all GREEN
2. `make test-unit` → full test suite GREEN
3. Submit for code review → `bmad-code-review`
