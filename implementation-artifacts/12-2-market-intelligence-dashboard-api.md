# Story 12.2: Market Intelligence Dashboard API

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **paid-tier company user on the EU Solicit platform**,
I want **a company-scoped Market Intelligence API that serves procurement-volume, contract-value, top-authority, and monthly-trend aggregates from the analytics materialized views**,
so that **the Market Intelligence dashboard (S12.3 frontend) can render Recharts visualizations of my market without running expensive live aggregations on every request**.

## Context & Scope Clarification

This story is the **backend API** half of the Market Intelligence dashboard. The matching frontend is **Story 12.3** (`12-3-market-intelligence-dashboard-frontend`). **Do not build any frontend in this story** — only the `client-api` endpoints, service layer, and Pydantic schemas.

This story belongs to the **analytics dashboard lineage** (S12.02–S12.08) that reads the materialized views created by the now-`done` **Story 12.1** (`12-1-analytics-materialized-views-refresh-infrastructure`). It reads `client.mv_market_intelligence` from the **`client` schema** and is served by **`client-api` (:8001)** — it is *company-scoped tenant analytics*, **not** the admin/leadership KPI surface.

> **Noted variance (surface, do not silently resolve):** The epic file `epic-12-admin-platform.md` describes S12.2 as "KPI Dashboards" on the *admin portal* (MRR, AI cost ratio, crawler health via admin-api). The implementation lineage in `eusolicit-docs/implementation-artifacts/` splits Epic 12 into 18 granular stories where **12-2 is the company-facing Market Intelligence API on `client-api`**, and the leadership/platform KPI surface is handled separately under `12-13-admin-api-audit-log-platform-analytics`. This story follows the implementation-artifacts lineage (consistent with 12-1, which explicitly states it enables "S12.02 … API routes in client-api"). If the operator intends the admin/MRR interpretation instead, raise it before dev — do not merge the two.

> **Implementation already scaffolded:** The router, service, schemas, and a large API test suite for this story **already exist in the repo** (committed Apr–May 2026; see File List). The dev work here is to **verify the existing implementation against these ACs, close any gaps, and bring the full Definition-of-Done gate suite to green** — not to build from a blank slate. Read the existing files first; do not reinvent or duplicate them.

## Acceptance Criteria

The four endpoints are all mounted under the router prefix `/analytics/market` (full path `/api/v1/analytics/market/...`).

1. **AC1 — `GET /api/v1/analytics/market/volume`** returns procurement volume by sector, **paginated**. Each item contains `sector`, `country`, `opportunity_count`, `total_value_eur`. Response shape is `PaginatedResponse[MarketVolumeItem]` (`items`, `total`, `page`, `page_size`). Supports optional query filters `date_from`, `date_to`, `sector`, `country`, and pagination `page` (≥1), `page_size` (1–100, default 20).

2. **AC2 — `GET /api/v1/analytics/market/values`** returns average contract values per sector/country, **non-paginated** (`ListResponse[MarketValuesItem]`). Each item contains `sector`, `country`, `avg_contract_value_eur`. Supports `date_from`, `date_to`, `sector`, `country` filters.

3. **AC3 — `GET /api/v1/analytics/market/authorities`** returns top contracting authorities **ranked by `opportunity_count` DESC**, **paginated** (`PaginatedResponse[MarketAuthorityItem]`). Each item contains `authority_name`, `sector`, `country`, `opportunity_count`, `total_value_eur`.

4. **AC4 — `GET /api/v1/analytics/market/trends`** returns monthly aggregate trend data **ordered by `month` ASC**, **non-paginated** (`ListResponse[MarketTrendItem]`). Each item contains `month` (ISO date), `sector`, `country`, `opportunity_count`, `total_value_eur`, `avg_contract_value_eur`.

5. **AC5 — AuthN:** Every endpoint requires a valid Bearer JWT. Missing/expired/invalid token → **401** before any handler logic runs.

6. **AC6 — Tier gate:** Every endpoint requires an **active paid subscription** (Starter or above) via `require_paid_tier`. A free-tier (or no-subscription) company → **403** with an "upgrade required" message. The gate runs before the handler body.

7. **AC7 — Cross-tenant isolation (E12-R-001, P0):** Every service query is scoped with `WHERE company_id = current_user.company_id` as the **first** filter. Company A must never receive Company B's rows. A negative cross-tenant test must assert this for at least the paginated endpoints.

8. **AC8 — Empty & cache behavior:** When the materialized view has no rows for the company, each endpoint returns **200** with `items=[]` (and `total=0` for paginated). Every successful response sets header `Cache-Control: public, max-age=1800` (30-minute cache).

9. **AC9 — DoD gates green:** `make lint`, `make type-check`, and `make test-service SVC=client-api` pass; `client-api` line coverage stays **≥ 80%** (`make coverage`). No cross-schema joins are introduced (queries stay within the `client` schema MV).

## Tasks / Subtasks

- [ ] **Task 1: Verify the data source & prerequisite (AC: 1–4, 7)**
  - [ ] 1.1 Confirm Story 12.1 is `done` and `client.mv_market_intelligence` exists (`make infra && make migrate-all`; the view is created by `services/client-api/alembic/versions/011_analytics_materialized_views.py`). If the MV is absent, this is a blocking prerequisite — see Dev Notes "Prerequisite".
  - [ ] 1.2 Confirm the read-only ORM Table `mv_market_intelligence` in `services/client-api/src/client_api/models/analytics_views.py` matches the live view columns (`company_id`, `sector`, `country`, `month`, `opportunity_count`, `avg_contract_value_eur`, `total_value_eur`, `authority_name`).

- [ ] **Task 2: Review & confirm the API router (AC: 1–6, 8)**
  - [ ] 2.1 Read `services/client-api/src/client_api/api/v1/analytics_market.py`. Confirm all four routes (`/volume`, `/values`, `/authorities`, `/trends`) exist with the correct response models, query params, `Depends(require_paid_tier)`, `Depends(get_db_session)`, and the `Cache-Control: public, max-age=1800` header.
  - [ ] 2.2 Confirm the router is registered in `services/client-api/src/client_api/main.py` (`api_v1_router.include_router(analytics_market_v1.router)`).

- [ ] **Task 3: Review & confirm the service layer (AC: 1–4, 7)**
  - [ ] 3.1 Read `services/client-api/src/client_api/services/analytics_market_service.py`. Confirm `get_market_volume`, `get_market_values`, `get_market_authorities`, `get_market_trends` each (a) take `company_id` as the first parameter, (b) apply `.where(mv_market.c.company_id == company_id)` as the first WHERE clause, (c) build queries with SQLAlchemy Core against the MV Table (no raw SQL / no cross-schema joins), (d) apply the shared `_apply_market_filters` for `date_from`/`date_to`/`sector`/`country`, (e) sort authorities by `opportunity_count` DESC and trends by `month` ASC.
  - [ ] 3.2 Confirm pagination math (`offset = (page-1)*page_size`, `limit = page_size`) and that the `total` count query is separate from the rows query.

- [ ] **Task 4: Review & confirm the Pydantic schemas (AC: 1–4)**
  - [ ] 4.1 Read `services/client-api/src/client_api/schemas/analytics.py`. Confirm `PaginatedResponse[T]`, `ListResponse[T]`, `MarketVolumeItem`, `MarketValuesItem`, `MarketAuthorityItem`, `MarketTrendItem` exist with `ConfigDict(from_attributes=True)` and `Decimal` (never `float`) for monetary fields.

- [ ] **Task 5: Verify & complete the test suite (AC: 5–9)**
  - [ ] 5.1 Run `make test-service SVC=client-api` for `tests/api/test_analytics_market.py` and confirm all classes pass (`TestGetVolume`, `TestGetValues`, `TestGetAuthorities`, `TestGetTrends`).
  - [ ] 5.2 Confirm each endpoint class covers: happy-path 200, each filter forwarded to the service, pagination params forwarded (paginated endpoints), empty-result 200, `Cache-Control` header present, 401 unauthenticated, 401 invalid token, 403 free tier, and company_id-scoping assertion (cross-tenant).
  - [ ] 5.3 If any AC lacks a test (especially the AC7 cross-tenant negative for both paginated endpoints), add it. Use root-`conftest.py` fixtures and the existing `paid_market_context` / `free_market_context` / `unauth_client` fixtures in the test file; **never `commit()`**; clear `app.dependency_overrides` in `finally`.

- [ ] **Task 6: Run the full Definition-of-Done gate suite (AC: 9)**
  - [ ] 6.1 `make lint` (ruff) — clean.
  - [ ] 6.2 `make type-check` (mypy) — clean.
  - [ ] 6.3 `make test-service SVC=client-api` — green.
  - [ ] 6.4 `make coverage` — `client-api` line coverage ≥ 80%; record the number in Completion Notes.
  - [ ] 6.5 Update the File List and Completion Notes with the verified state and any gaps closed.

## Dev Notes

### Prerequisite (verify before coding)

This story **reads** `client.mv_market_intelligence`, created by **Story 12.1** (`done`). The view + its unique/`company_id` indexes are defined in `services/client-api/alembic/versions/011_analytics_materialized_views.py` and refreshed daily (02:00 UTC) by a Celery Beat task in the Notification Service. If after `make migrate-all` the view is missing or empty:
- Missing view → emit `PREREQ_UNMET: client.mv_market_intelligence absent` / `PREREQ_TYPE: MISSING_MIGRATION` / `PREREQ_ROUTE: make migrate-all` / `PREREQ_DETAIL: run services/client-api migration 011`.
- Empty view in a fresh env is expected (R-006); AC8 empty-state handling covers it. Tests mock the service layer, so they do not depend on live MV rows.

[Source: eusolicit-docs/implementation-artifacts/12-1-analytics-materialized-views-refresh-infrastructure.md]

### Existing implementation map (read these first)

The feature is already scaffolded. Confirm/complete rather than recreate:

| File | Role | Status |
|------|------|--------|
| `services/client-api/src/client_api/api/v1/analytics_market.py` | Router — 4 GET endpoints under `/analytics/market` | EXISTS |
| `services/client-api/src/client_api/services/analytics_market_service.py` | Service — `get_market_volume/values/authorities/trends` (SQLAlchemy Core on the MV, company-scoped) | EXISTS |
| `services/client-api/src/client_api/schemas/analytics.py` | Pydantic response DTOs + generic `PaginatedResponse`/`ListResponse` | EXISTS |
| `services/client-api/src/client_api/models/analytics_views.py` | Read-only `mv_market_intelligence` Table (separate `MetaData`, `info={"is_view": True}`) | EXISTS (from 12.1) |
| `services/client-api/src/client_api/main.py` | Router registration (line ~165) | EXISTS |
| `services/client-api/tests/api/test_analytics_market.py` | ~40 API tests across 4 endpoint classes | EXISTS |

### Architecture & pattern constraints (DEV GUARDRAILS)

- **Service ownership:** `client-api` (:8001), `client` schema. **Never join across schemas** — the MV already pre-aggregates `pipeline.opportunities` data into the `client` schema; query only `client.mv_market_intelligence`.
- **Company scoping is non-negotiable (E12-R-001, highest epic risk):** `company_id` is the first parameter of every service function and the first WHERE clause of every query. The value comes from `current_user.company_id` (decoded from the RS256 JWT), never from a request body/param.
- **Tier gating:** Use the existing `require_paid_tier` dependency from `client_api.core.tier_gate` — do **not** invent an inline tier check. It read-through-caches the tier in Redis (key `tier:{company_id}`, TTL 60s) then falls back to a `client.subscriptions` query; `PAID_TIERS = {starter, professional, pro_plus, enterprise}`. Free/none → `ForbiddenError` (403).
- **AuthN / `is_active`:** `require_paid_tier` depends on the standard current-user resolution (`get_current_user_or_internal`), which enforces JWT validity. The active-principal check lives in that auth layer — do not bypass it.
- **`from __future__ import annotations`** at the top of every module; `structlog` for any logging (no `print`); read config via `get_settings()`, never `os.environ` in business logic.
- **Money = `Decimal`**, never `float`. Schemas use `ConfigDict(from_attributes=True)` to hydrate from SQLAlchemy rows via `.model_validate(row)`.
- **Caching:** `Cache-Control: public, max-age=1800` set explicitly on each response. Data freshness is bounded by the daily MV refresh (R-003) — the 30-min browser/CDN cache is acceptable on top of day-grained data.
- **ruff:** line length 120, rules `I E W F UP`. Note the existing `# noqa: UP046` on the generic `PaginatedResponse`/`ListResponse` classes — keep it (intentional generic-syntax suppression).

[Source: /home/debian/Projects/eusolicit/CLAUDE.md#Architecture; project delivery instructions — Authorization, Security, Isolation & boundaries]

### Test design alignment (Epic 12)

From `eusolicit-docs/test-artifacts/test-design-epic-12.md`, the risks this story must keep mitigated:

- **R-003 (DATA, score 6) — stale/incorrect KPI data:** Dashboards read MV-backed aggregates, not live queries (perf + freshness). This API must read the MV (not re-aggregate live) — confirm the service queries `mv_market_intelligence`, satisfying the "MV-backed (not live aggregation)" P2 perf check (R-005).
- **R-005 (PERF, score 4):** Dashboard endpoints baseline < 5s with seeded volume. MV-backed reads + pagination keep this comfortably met; do not introduce N+1 or live joins.
- **R-001 / cross-tenant (inherited, P0):** company-scoped queries — covered by AC7 and the `test_cross_tenant_*` tests.
- **R-008 (Recharts):** out of scope here (frontend S12.3).

Test level for this story is **API/integration** (`@pytest.mark.api` style with `app.dependency_overrides`), per the test design's "SEC/data logic exercised at API level" discipline. E2E data-assertion (UI value == MV value) belongs to S12.3.

[Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#Risk-Assessment, #P0, #P1, #P2]

### Testing standards

- Markers/fixtures: the existing test file uses `@pytest.mark.asyncio` + `@pytest_asyncio.fixture`, overriding `get_db_session` and `get_redis_client` on the app and seeding an active Starter subscription, then clearing `dependency_overrides` in teardown. **Never call `commit()`** (per-test rollback contract). `clean_redis` flushes DB 1; app uses DB 0.
- Service functions are typically **mocked** in the API-tier tests (assert correct args incl. `company_id`, response shaping, status codes, headers) so they do not require live MV rows.
- Run target: `make test-service SVC=client-api`. For a single file during dev: `pytest services/client-api/tests/api/test_analytics_market.py -v`.

### Project Structure Notes

- Routers live in `services/client-api/src/client_api/api/v1/` (this repo uses `api/v1/`, **not** `infrastructure/api/routers/` — the 12.1 dev-note reference to the latter is stale; follow the actual `api/v1/` layout).
- Sibling analytics stories share `schemas/analytics.py` and the `analytics_*` router/service naming: ROI (12.4), Team (12.5), Competitors (12.6 — Professional tier), Pipeline (12.7 — Professional tier), Usage (12.8). Keep schema additions in the shared file; don't fork it.

### References

- [Source: eusolicit-docs/planning-artifacts/epics/epic-12-admin-platform.md#Story-12.2]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#R-003, #R-005, #P0, #P1]
- [Source: eusolicit-docs/implementation-artifacts/12-1-analytics-materialized-views-refresh-infrastructure.md] (MV definition + prerequisite)
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/analytics_market.py]
- [Source: eusolicit-app/services/client-api/src/client_api/services/analytics_market_service.py]
- [Source: eusolicit-app/services/client-api/src/client_api/schemas/analytics.py]
- [Source: eusolicit-app/services/client-api/src/client_api/core/tier_gate.py] (`require_paid_tier`, `PAID_TIERS`)
- [Source: eusolicit-app/services/client-api/tests/api/test_analytics_market.py]
- [Project Context: eusolicit-docs/project-context.md]

## Dev Agent Record

### Agent Model Used

{{agent_model_name_version}}

### Debug Log References

### Completion Notes List

### File List

## Senior Developer Review

**Reviewer:** Claude (bmad-code-review, adversarial)
**Date:** 2026-05-25
**Outcome:** Changes Requested

### Summary

The core feature code for S12.2 is genuinely good: the four endpoints, the
SQLAlchemy-Core service layer, and the Pydantic schemas are correctly written
and pass ruff in isolation. Company scoping (E12-R-001) is correct — `company_id`
is the first parameter and first `WHERE` clause in every service function;
money is `Decimal`; `Cache-Control: public, max-age=1800` is set on each route;
the existing `require_paid_tier` gate is reused; no cross-schema joins or raw
SQL are introduced. A comprehensive (~40-test) API suite exists.

**However, the story cannot pass its own Definition of Done (AC9) and is not
complete.** The client-api app does not import, which makes the entire
client-api test suite uncollectable — so AC5/AC6/AC7/AC8 are currently
*unverifiable*, not just unverified. The story file is also still
`Status: ready-for-dev` with an empty Dev Agent Record (no File List, no
Completion Notes), i.e. no dev pass was actually recorded.

### Blocking findings

1. **[BLOCKER] `client_api/main.py` does not import — whole client-api suite
   fails collection.** Line 21 comments out
   `# from client_api.api.v1 import analytics_pipeline as analytics_pipeline_v1`
   but line 169 still calls `api_v1_router.include_router(analytics_pipeline_v1.router)`.
   Result: `NameError: name 'analytics_pipeline_v1' is not defined` (also caught
   by ruff as `F821`). `make test-service SVC=client-api` aborts with **8
   collection errors** before a single test runs. Root cause is the incomplete
   sibling story S12.7 (pipeline) — `analytics_pipeline_service.py:19` also
   references a non-existent `pipeline_predictions` MV attribute — but
   regardless of origin, S12.2's AC9 gate is red until `main.py` imports
   cleanly. Fix: either restore the import (if S12.7's router exists and is
   safe) or comment out line 169 to match line 21.

2. **[BLOCKER] `tests/api/test_analytics_market_gaps.py` breaks collection.**
   Line 32 sets `pytest_plugins = ("client_api.tests.api.test_analytics_market",)`
   — an invalid module path (`client_api.tests` is not an importable package;
   tests live outside the `src` package). This yields
   `ImportError: No module named 'client_api.tests'`. Additionally,
   `pytest_plugins` is only honored in a top-level/rootdir conftest, not in an
   arbitrary test module. Remove the `pytest_plugins` line; the sibling-file
   imports on lines 21–28 already pull in the shared helpers, and fixtures
   resolve via the package `conftest`.

3. **[MAJOR / AC2–AC4 coverage gap, Task 5.3 not done] ATDD gap tests remain
   skipped.** Every test in `test_analytics_market_gaps.py` is
   `@pytest.mark.skip(reason="RED-PHASE …")`, covering the explicitly-listed
   gaps: AC2 `/values` date-range filter, AC3 `/authorities` date/sector/country
   filters, AC4 `/trends` sector/country filters. Task 5.3 required these to be
   un-skipped and made green. They were never closed. (The underlying service
   `_apply_market_filters` already supports these filters, so un-skipping should
   be low-effort — but it must actually be done and run.)

### Gate results (run during review)

- `make lint` → **FAIL**, 80 errors. 2 are in this lineage
  (`main.py` `F821` undefined `analytics_pipeline_v1` + an `I001` import-order);
  the rest are pre-existing in unrelated files (SirmaAI rename, models, other
  test modules). The story's own four feature files pass ruff cleanly.
- `make type-check` → **FAIL**, 35 errors across 11 files. In-lineage:
  `main.py:169` undefined name; `analytics_pipeline_service.py:19` missing
  `pipeline_predictions` MV attr (S12.7). Remainder unrelated (notification,
  enterprise-api, integrations-api).
- `make test-service SVC=client-api` → **FAIL** — 8 collection errors, 0 tests
  executed (see Blocker 1 & 2).
- `make coverage` → not runnable (suite cannot collect).

### Required to approve

- [ ] Fix `main.py` so the app imports (align line 169 with the line-21 import
      decision). `make test-service SVC=client-api` must collect and run.
- [ ] Fix/remove the invalid `pytest_plugins` in `test_analytics_market_gaps.py`.
- [ ] Un-skip and pass the AC2/AC3/AC4 filter gap tests (Task 5.3).
- [ ] Re-run AC9 gates: `make lint`, `make type-check`, `make test-service
      SVC=client-api`, `make coverage` (client-api ≥ 80%). Record numbers in
      Completion Notes and populate the File List; move Status to `review`.

### Notes on scope

Blockers 1 and the type/lint errors in `analytics_pipeline_service.py` originate
from sibling story S12.7 being left half-wired into `main.py`. This is an
**architectural-drift / integration** break that lands on S12.2 because S12.2's
DoD is the whole client-api suite. Flagging it here rather than silently
patching across story boundaries.

DEVIATION: client-api app fails to import — main.py:169 uses analytics_pipeline_v1 whose import (line 21) is commented out, breaking the entire client-api test suite collection (8 errors)
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

DEVIATION: AC2/AC3/AC4 filter-coverage ATDD gap tests in test_analytics_market_gaps.py remain @pytest.mark.skip and the file's pytest_plugins path is invalid (Task 5.3 incomplete)
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: blocking

FAILURE_REASON: client-api app fails to import (NameError analytics_pipeline_v1 at main.py:169) so AC9 test/coverage gates cannot run; AC2-AC4 gap tests left skipped; story still ready-for-dev with empty Dev Agent Record
FAILURE_CATEGORY: test_coverage
SUGGESTED_FIX: Align main.py:169 with the commented import at line 21 (restore import or remove the include_router call); remove invalid pytest_plugins in test_analytics_market_gaps.py; un-skip and pass the AC2/AC3/AC4 filter gap tests; re-run make lint/type-check/test-service/coverage and record results, then set Status: review
