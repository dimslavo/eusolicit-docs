# Story 12.2: Market Intelligence Dashboard API

Status: review-complete

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **paid-tier user on the EU Solicit platform**,
I want **Client API endpoints that query the market intelligence materialized view**,
so that **the market intelligence dashboard page (S12.03) can display procurement volumes, average contract values, top contracting authorities, and seasonal trends — all filtered by date range, sector, and country — without hitting live source tables on every request**.

## Acceptance Criteria

1. **AC1** — `GET /api/v1/analytics/market/volume` returns procurement volume by sector. Response body is `{"items": [...], "total": N, "page": P, "page_size": S}` where each item contains `sector`, `country`, `opportunity_count`, and `total_value_eur`. Supports `date_from`, `date_to`, `sector`, and `country` query parameters. Paginated via `page` (default 1) and `page_size` (default 20, max 100).

2. **AC2** — `GET /api/v1/analytics/market/values` returns average contract values by sector. Response body is `{"items": [...]}` where each item contains `sector`, `country`, and `avg_contract_value_eur`. Supports `date_from`, `date_to`, `sector`, and `country` filters. Not paginated (aggregated per sector — bounded result set).

3. **AC3** — `GET /api/v1/analytics/market/authorities` returns top contracting authorities ranked by `opportunity_count` descending. Response body is `{"items": [...], "total": N, "page": P, "page_size": S}` where each item contains `authority_name`, `sector`, `country`, `opportunity_count`, and `total_value_eur`. Supports `date_from`, `date_to`, `sector`, and `country` filters. Paginated via `page`/`page_size`.

4. **AC4** — `GET /api/v1/analytics/market/trends` returns monthly aggregate trend data. Response body is `{"items": [...]}` where each item contains `month` (ISO date string, first day of month), `sector`, `country`, `opportunity_count`, `total_value_eur`, and `avg_contract_value_eur`. Supports `date_from`, `date_to`, `sector`, and `country` filters. Not paginated (time-series data for charting).

5. **AC5** — All 4 endpoints require a valid Bearer JWT (`Authorization: Bearer <token>`). Requests with no token or an expired/invalid token return HTTP 401. Requests from users whose company has a `free` plan or no active subscription return HTTP 403 with body `{"message": "This feature requires a paid subscription.", "upgrade_required": true}`.

6. **AC6** — All 4 endpoints **always** scope queries with `WHERE company_id = current_user.company_id`. No query may return rows belonging to a different company. This is the primary defence for E12-R-001 (cross-tenant analytics leakage, risk score 6).

7. **AC7** — Empty-result responses return HTTP 200 with an empty `items` array (and `total: 0` for paginated endpoints). HTTP 4xx is never returned for legitimate requests that match zero rows.

8. **AC8** — All 4 endpoints include the response header `Cache-Control: public, max-age=1800` (30-minute cache; materialized view is refreshed daily so brief staleness is acceptable).

9. **AC9** — Unit tests in `tests/api/test_analytics_market.py` cover: each filter parameter applied independently; date range + sector + country combined filter; empty-result edge case (200 with empty items); cross-tenant isolation (Company B rows not returned for Company A token); cache header present in all responses; 401 for missing/invalid JWT; 403 for free-tier user.

## Tasks / Subtasks

- [x] Task 1: Pydantic response schemas — `schemas/analytics.py` (AC: 1, 2, 3, 4, 7)
  - [x] 1.1 Create `services/client-api/src/client_api/schemas/analytics.py`
  - [x] 1.2 Define `MarketVolumeItem(sector: str, country: str, opportunity_count: int, total_value_eur: Decimal | None)` — maps `mv_market_intelligence` columns used by `/volume`
  - [x] 1.3 Define `MarketValuesItem(sector: str, country: str, avg_contract_value_eur: Decimal | None)` — for `/values`
  - [x] 1.4 Define `MarketAuthorityItem(authority_name: str, sector: str, country: str, opportunity_count: int, total_value_eur: Decimal | None)` — for `/authorities`
  - [x] 1.5 Define `MarketTrendItem(month: date, sector: str, country: str, opportunity_count: int, total_value_eur: Decimal | None, avg_contract_value_eur: Decimal | None)` — for `/trends`
  - [x] 1.6 Define generic `PaginatedResponse[T](items: list[T], total: int, page: int, page_size: int)` as a Pydantic generic model; use for `/volume` and `/authorities`
  - [x] 1.7 Define `ListResponse[T](items: list[T])` as a simple Pydantic generic wrapper; use for `/values` and `/trends`
  - [x] 1.8 Export all schemas from `services/client-api/src/client_api/schemas/__init__.py` (add to existing exports)

- [x] Task 2: Paid-tier gate dependency — `core/tier_gate.py` (AC: 5)
  - [x] 2.1 Create `services/client-api/src/client_api/core/tier_gate.py`
  - [x] 2.2 Define module-level constant `PAID_PLANS: frozenset[str] = frozenset({"starter", "professional", "enterprise"})` — these are the plan values stored in `client.subscriptions.plan`
  - [x] 2.3 Implement async FastAPI dependency `require_paid_tier(current_user: CurrentUser, session: AsyncSession) -> CurrentUser`: query `SELECT id FROM client.subscriptions WHERE company_id = :company_id AND plan = ANY(:plans) AND status = 'active' LIMIT 1` via SQLAlchemy; if no row found, raise `HTTPException(status_code=403, detail={"message": "This feature requires a paid subscription.", "upgrade_required": True})`
  - [x] 2.4 Use `Annotated[CurrentUser, Depends(get_current_user)]` and `Annotated[AsyncSession, Depends(get_db_session)]` as parameters — do NOT call `get_current_user` separately; require_paid_tier depends on it
  - [x] 2.5 Log denied access via `structlog.get_logger()` at INFO level with `company_id` and `plan` (or `None` if no subscription row found) — useful for upgrade funnel analytics
  - [x] 2.6 Add `require_professional_plus_tier` stub (raises `NotImplementedError`) as a placeholder for S12.06/S12.07 Professional+ gate — do NOT implement; just reserve the name so future stories can fill it in without naming conflicts

- [x] Task 3: Analytics market service — `services/analytics_market_service.py` (AC: 1, 2, 3, 4, 6, 7)
  - [x] 3.1 Create `services/client-api/src/client_api/services/analytics_market_service.py`
  - [x] 3.2 Define internal `_apply_market_filters(stmt, date_from, date_to, sector, country)` helper — applies optional WHERE clauses on `mv_market_intelligence.c.month >= date_from`, `month <= date_to`, `sector = sector`, `country = country`; skip clauses where parameter is `None`
  - [x] 3.3 Implement `async get_market_volume(company_id, date_from, date_to, sector, country, page, page_size, session) -> tuple[list[MarketVolumeItem], int]`: SELECT `sector`, `country`, `SUM(opportunity_count)`, `SUM(total_value_eur)` FROM `client.mv_market_intelligence` WHERE `company_id = :company_id` + optional filters; GROUP BY `sector, country`; ORDER BY `total_value_eur DESC NULLS LAST`; return `(items, total_count)`. Use two queries: count query first, then paginated result query (OFFSET/LIMIT).
  - [x] 3.4 Implement `async get_market_values(company_id, date_from, date_to, sector, country, session) -> list[MarketValuesItem]`: SELECT `sector`, `country`, `AVG(avg_contract_value_eur)` FROM `client.mv_market_intelligence` WHERE `company_id = :company_id` + optional filters; GROUP BY `sector, country`; ORDER BY `avg_contract_value_eur DESC NULLS LAST`
  - [x] 3.5 Implement `async get_market_authorities(company_id, date_from, date_to, sector, country, page, page_size, session) -> tuple[list[MarketAuthorityItem], int]`: SELECT `authority_name`, `sector`, `country`, `SUM(opportunity_count)`, `SUM(total_value_eur)` FROM `client.mv_market_intelligence` WHERE `company_id = :company_id` + optional filters; GROUP BY `authority_name, sector, country`; ORDER BY `SUM(opportunity_count) DESC NULLS LAST`; paginated
  - [x] 3.6 Implement `async get_market_trends(company_id, date_from, date_to, sector, country, session) -> list[MarketTrendItem]`: SELECT `month`, `sector`, `country`, `SUM(opportunity_count)`, `SUM(total_value_eur)`, `AVG(avg_contract_value_eur)` FROM `client.mv_market_intelligence` WHERE `company_id = :company_id` + optional filters; GROUP BY `month, sector, country`; ORDER BY `month ASC`
  - [x] 3.7 All service functions MUST use `sqlalchemy.select()` from `client_api.models.analytics_views.mv_market_intelligence` (the `sa.Table` object) — do NOT write raw SQL strings in service functions; use SQLAlchemy Core expressions
  - [x] 3.8 All service functions MUST include `company_id` in the WHERE clause as the first and non-optional filter (E12-R-001); if this clause is accidentally removed the unit tests will catch it via the cross-tenant isolation test

- [x] Task 4: Analytics market API router — `api/v1/analytics_market.py` (AC: 1, 2, 3, 4, 5, 6, 7, 8)
  - [x] 4.1 Create `services/client-api/src/client_api/api/v1/analytics_market.py` with `router = APIRouter(prefix="/analytics/market", tags=["analytics"])`
  - [x] 4.2 Implement `GET /volume` endpoint: parameters `date_from: date | None = Query(None)`, `date_to: date | None = Query(None)`, `sector: str | None = Query(None)`, `country: str | None = Query(None)`, `page: int = Query(1, ge=1)`, `page_size: int = Query(20, ge=1, le=100)`. Dependencies: `current_user: Annotated[CurrentUser, Depends(require_paid_tier)]`, `session: Annotated[AsyncSession, Depends(get_db_session)]`. Return type: `PaginatedResponse[MarketVolumeItem]`. Call `analytics_market_service.get_market_volume(...)`. Set `Cache-Control` header via `Response` parameter.
  - [x] 4.3 Implement `GET /values` endpoint: same date/sector/country filters (no pagination). Dependencies: same as `/volume`. Return type: `ListResponse[MarketValuesItem]`. Call `analytics_market_service.get_market_values(...)`. Set `Cache-Control` header.
  - [x] 4.4 Implement `GET /authorities` endpoint: same filters + pagination. Return type: `PaginatedResponse[MarketAuthorityItem]`. Call `analytics_market_service.get_market_authorities(...)`. Set `Cache-Control` header.
  - [x] 4.5 Implement `GET /trends` endpoint: same date/sector/country filters (no pagination). Return type: `ListResponse[MarketTrendItem]`. Call `analytics_market_service.get_market_trends(...)`. Set `Cache-Control` header.
  - [x] 4.6 Set the `Cache-Control` response header on all 4 endpoints: inject `response: Response` as a function parameter (FastAPI pattern) and call `response.headers["Cache-Control"] = "public, max-age=1800"` before returning
  - [x] 4.7 The `require_paid_tier` dependency returns `CurrentUser` — extract `company_id` from it for all service calls: `current_user.company_id`. Do NOT use a separate `get_current_user` dependency in these endpoints.

- [x] Task 5: Register analytics market router in `main.py` (AC: 1, 2, 3, 4)
  - [x] 5.1 Import `from client_api.api.v1 import analytics_market as analytics_market_v1` in `services/client-api/src/client_api/main.py`
  - [x] 5.2 Add `api_v1_router.include_router(analytics_market_v1.router)` after the existing `grants_v1` router inclusion
  - [x] 5.3 Verify that the full endpoint paths resolve correctly to `/api/v1/analytics/market/volume`, `/api/v1/analytics/market/values`, `/api/v1/analytics/market/authorities`, `/api/v1/analytics/market/trends`

- [x] Task 6: Unit tests — `tests/api/test_analytics_market.py` (AC: 9)
  - [x] 6.1 Create `services/client-api/tests/api/test_analytics_market.py`
  - [x] 6.2 Test fixture `paid_auth_headers(test_client, client_session, rsa_test_key_pair)`: seed a company with Starter plan (active subscription), create a user, issue a JWT, return headers dict `{"Authorization": "Bearer <token>"}`. Seed `mv_market_intelligence` with 3 rows for this company across 2 sectors, 2 countries, 3 months.
  - [x] 6.3 Test fixture `free_auth_headers(test_client, client_session, rsa_test_key_pair)`: seed a company with `plan = 'free'` (no active paid subscription), return auth headers.
  - [x] 6.4 Test fixture `other_company_rows(client_session)`: seed 2 `mv_market_intelligence` rows for a *different* company_id. Used for cross-tenant isolation assertions.
  - [x] 6.5 **GET /volume — happy path**: authenticated paid user; assert 200, response has `items` list, `total >= 0`, `page = 1`, `page_size = 20`; each item has `sector`, `country`, `opportunity_count`, `total_value_eur`
  - [x] 6.6 **GET /volume — sector filter**: pass `?sector=<value_in_seeded_data>`; assert response items all have matching sector; no cross-sector leakage
  - [x] 6.7 **GET /volume — country filter**: pass `?country=<value>`; assert all items match country
  - [x] 6.8 **GET /volume — date range filter**: pass `?date_from=<start>&date_to=<end>` spanning only some of the seeded months; assert only months in range returned
  - [x] 6.9 **GET /volume — combined filters**: pass `?date_from=...&date_to=...&sector=...&country=...`; assert items match all constraints simultaneously
  - [x] 6.10 **GET /volume — empty result**: pass `?sector=nonexistent_sector`; assert 200, `items = []`, `total = 0`
  - [x] 6.11 **GET /volume — cross-tenant isolation**: with `paid_auth_headers` and `other_company_rows` seeded; call `/volume` without sector filter; assert none of the returned items contain data from `other_company_rows` (check by verifying no `authority_name` or value from the other company's seed rows appears)
  - [x] 6.12 **GET /volume — pagination**: seed > 20 rows for the paid company; call `?page=2&page_size=5`; assert `page = 2`, `page_size = 5`, `len(items) <= 5`, `total > 5`
  - [x] 6.13 **GET /volume — cache header**: assert response has `Cache-Control: public, max-age=1800` header
  - [x] 6.14 **GET /volume — unauthenticated**: call with no `Authorization` header; assert 401
  - [x] 6.15 **GET /volume — free tier**: call with `free_auth_headers`; assert 403, response body contains `"upgrade_required": true`
  - [x] 6.16 **GET /values — happy path, filters, empty result, cache header**: repeat same test structure as volume (6.5, 6.6, 6.7, 6.10, 6.13 equivalents) for `/values`; no pagination fields in response
  - [x] 6.17 **GET /authorities — happy path, sort order, pagination, empty result, cache header, cross-tenant, 401, 403**: same test coverage as volume (6.5–6.15) for `/authorities`; verify items are sorted by `opportunity_count DESC`
  - [x] 6.18 **GET /trends — happy path, month ordering, filters, empty result, cache header, cross-tenant, 401, 403**: same structure for `/trends`; verify `month` field is ISO date string and items are ordered by `month ASC`
  - [x] 6.19 Test `require_paid_tier` in isolation (unit, not API): mock session returning no subscription row; assert `HTTPException(status_code=403)` is raised; mock session returning a `professional` plan row; assert `CurrentUser` returned

## Dev Notes

### Architecture Context

**Service:** `services/client-api` — FastAPI on port 8001, async SQLAlchemy (asyncpg driver).

**Depends on S12.01 (done):** Migration `011_analytics_materialized_views.py` has already created `client.mv_market_intelligence` and the supporting unique/filter indexes. The SQLAlchemy `Table` definition already exists in `services/client-api/src/client_api/models/analytics_views.py`.

**Pattern reference:** Study `services/client-api/src/client_api/api/v1/grants.py` and `services/client-api/src/client_api/services/grants_service.py` for the dependency injection, router structure, and service call patterns used in this project.

---

### `mv_market_intelligence` — Column Reference

| Column | SQLAlchemy Type | Description |
|--------|----------------|-------------|
| `company_id` | `UUID(as_uuid=True)` | Tenant key — **every query must filter by this** |
| `sector` | `Text` | CPV code prefix (e.g. `"45"`) or `"unknown"` |
| `country` | `Text` | ISO 2-letter country code or `"unknown"` |
| `month` | `Date` | First day of the aggregation month (DATE type) |
| `opportunity_count` | `BigInteger` | COUNT(DISTINCT opportunity.id) in this group |
| `avg_contract_value_eur` | `Numeric` | AVG(estimated_value_eur) in this group |
| `total_value_eur` | `Numeric` | SUM(estimated_value_eur) in this group |
| `authority_name` | `Text` | `contracting_authority` text from opportunities |

Import the table object from the models module:
```python
from client_api.models.analytics_views import mv_market_intelligence as mv_market
```

---

### CRITICAL: Company-ID Scoping (E12-R-001, Risk Score 6)

**Every query against `mv_market_intelligence` MUST include `WHERE company_id = :company_id` as the first non-optional filter.** This is the primary defence against cross-tenant analytics leakage (E12-R-001 — highest-scored risk in E12).

The P0 Playwright test `e2e/specs/analytics/cross-tenant-isolation.api.spec.ts` seeds two companies (A and B), authenticates as Company A, calls `GET /api/v1/analytics/market/volume`, and asserts that zero rows with Company B's data are returned. If the `company_id` filter is ever missing from a service function, this test will fail with cross-tenant data exposure.

**Service function signature convention:**
```python
async def get_market_volume(
    company_id: UUID,        # FIRST parameter — always required
    date_from: date | None,
    date_to: date | None,
    ...
) -> ...:
    stmt = (
        select(mv_market.c.sector, mv_market.c.country, ...)
        .where(mv_market.c.company_id == company_id)  # first WHERE clause
        ...
    )
```

[Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#E12-R-001, atdd-checklist-e12-p0.md]

---

### Tier Gate: Paid Subscription Check

The `Subscription` model is in `services/client-api/src/client_api/models/subscription.py`:
- Table: `client.subscriptions`
- Key columns: `company_id`, `plan` (string), `status` (string)
- Paid plan values: `"starter"`, `"professional"`, `"enterprise"` (lowercase, as stored)
- Free plan value: `"free"` — or `plan IS NULL` / no row → also denied
- Active status value: `"active"` — trialing users may also be considered paid; product confirmation needed; for now, `status = 'active'` only

**Dependency chain:**
```
GET /analytics/market/volume
  → require_paid_tier(current_user, session)
      → get_current_user(credentials) → CurrentUser
      → query subscriptions WHERE company_id = current_user.company_id
        AND plan IN ('starter', 'professional', 'enterprise')
        AND status = 'active'
      → raises 403 if no row
      → returns CurrentUser if row found
  → analytics_market_service.get_market_volume(company_id=current_user.company_id, ...)
```

**Note:** `require_paid_tier` is new infrastructure created in this story. It is a simpler gate than the Professional+ gate needed in S12.06/S12.07. Future stories must not break this gate. The `require_professional_plus_tier` stub (Task 2.6) reserves the name so S12.06 can implement it without a naming conflict.

---

### Cache-Control Header Strategy

Materialized views are refreshed daily (nightly). Within any given day, the view data is stable. A 30-minute `max-age` cache is appropriate:
- Short enough that users see updated data within ~30 min of a refresh
- Long enough for CDN/browser to benefit from caching on dashboards with multiple tabs or repeated loads

**Implementation pattern (FastAPI):**
```python
from fastapi import Response

@router.get("/volume", response_model=PaginatedResponse[MarketVolumeItem])
async def get_volume(
    ...,
    response: Response,  # injected by FastAPI — no Depends() needed
) -> PaginatedResponse[MarketVolumeItem]:
    response.headers["Cache-Control"] = "public, max-age=1800"
    ...
```

---

### Service Query Patterns — SQLAlchemy Core

Use SQLAlchemy Core (`sa.select(...)`) with the `sa.Table` object from `analytics_views.py`. Do not use raw SQL strings. For aggregation:

```python
from sqlalchemy import select, func
from client_api.models.analytics_views import mv_market_intelligence as mv_market

# Aggregate example — get_market_volume
count_stmt = (
    select(func.count())
    .select_from(
        select(
            mv_market.c.sector,
            mv_market.c.country,
        )
        .where(mv_market.c.company_id == company_id)
        # ... optional filters from _apply_market_filters
        .group_by(mv_market.c.sector, mv_market.c.country)
        .subquery()
    )
)
total = (await session.execute(count_stmt)).scalar_one()

rows_stmt = (
    select(
        mv_market.c.sector,
        mv_market.c.country,
        func.sum(mv_market.c.opportunity_count).label("opportunity_count"),
        func.sum(mv_market.c.total_value_eur).label("total_value_eur"),
    )
    .where(mv_market.c.company_id == company_id)
    # ... optional filters
    .group_by(mv_market.c.sector, mv_market.c.country)
    .order_by(func.sum(mv_market.c.total_value_eur).desc().nullslast())
    .offset((page - 1) * page_size)
    .limit(page_size)
)
rows = (await session.execute(rows_stmt)).mappings().all()
```

**Numeric precision:** `avg_contract_value_eur` and `total_value_eur` are `Numeric` (Decimal) in the view. Pydantic response models should use `Decimal | None` (import from `decimal`). FastAPI serialises `Decimal` correctly to JSON strings if `use_enum_values=True` is not set. Use `model_config = ConfigDict(from_attributes=True)` on all response schemas to allow attribute-based construction from SQLAlchemy row mappings.

---

### File Locations Summary

| File | Action | Notes |
|------|--------|-------|
| `services/client-api/src/client_api/schemas/analytics.py` | **Create** | All analytics Pydantic schemas (S12.02 only adds market schemas; future stories append to this file) |
| `services/client-api/src/client_api/core/tier_gate.py` | **Create** | `require_paid_tier` dependency (and stub for `require_professional_plus_tier`) |
| `services/client-api/src/client_api/services/analytics_market_service.py` | **Create** | Market analytics query functions |
| `services/client-api/src/client_api/api/v1/analytics_market.py` | **Create** | Market analytics FastAPI router |
| `services/client-api/src/client_api/main.py` | **Modify** | Import and include `analytics_market_v1.router` |
| `services/client-api/src/client_api/schemas/__init__.py` | **Modify** | Export new analytics schemas |
| `services/client-api/tests/api/test_analytics_market.py` | **Create** | Unit tests per AC9 |

---

### Test Data Seeding for Unit Tests

The unit tests in `tests/api/test_analytics_market.py` must seed the `mv_market_intelligence` view directly via the `client_session` fixture. Because materialized views are read-only in PostgreSQL (cannot INSERT), tests must seed the **source tables** (`client.bid_decisions`, `pipeline.opportunities`) and then manually populate the view by calling `REFRESH MATERIALIZED VIEW CONCURRENTLY client.mv_market_intelligence` — or alternatively, use a direct INSERT bypass via a test-only helper.

**Recommended approach for test speed:** Insert rows directly into `mv_market_intelligence` via a raw SQL bypass only enabled in the test environment:
```sql
INSERT INTO client.mv_market_intelligence (company_id, sector, country, month, opportunity_count, avg_contract_value_eur, total_value_eur, authority_name)
VALUES (:company_id, :sector, :country, :month, :count, :avg_val, :total_val, :authority);
```
This requires running as `migration_role` (which has full CRUD) rather than `client_api_role` (which is SELECT-only on views). Check whether the test conftest already has a `migration_role` session fixture before creating a new one.

**Alternatively:** If inserting into the view is not possible from the test DB role, seed the source tables (bid_decisions + opportunities) and call `REFRESH MATERIALIZED VIEW CONCURRENTLY client.mv_market_intelligence` synchronously in the fixture. Check migration `011` to confirm what role owns the view and can refresh it.

[Source: eusolicit-docs/implementation-artifacts/12-1-analytics-materialized-views-refresh-infrastructure.md — AC5 management command; the `notification_role` owns the view and has REFRESH permission]

---

### Pagination Pattern

Use `page`/`page_size` offset-based pagination (consistent with other project patterns):
- `page: int = Query(default=1, ge=1)` — 1-indexed
- `page_size: int = Query(default=20, ge=1, le=100)` — max 100 per page
- `OFFSET = (page - 1) * page_size`
- Response includes `total` (count of all matching rows before pagination), `page`, `page_size`, `items`

This pagination pattern is used for `/volume` and `/authorities`. The `/values` and `/trends` endpoints do not paginate (their result sets are bounded by sector count and monthly time range respectively).

---

### Test Coverage Alignment with Epic-Level Test Design

From `eusolicit-docs/test-artifacts/test-design-epic-12.md`:

| Priority | Test ID | Description | Maps to this story's AC |
|----------|---------|-------------|------------------------|
| **P0** | cross-tenant-isolation | Company A market data not visible to Company B | AC6, AC9 test 6.11 |
| **P1** | S12.02 | Volume/values/authorities/trends with filter combos + empty-result edge case | AC1–AC4, AC7, AC9 |
| **P2** | S12.03 (frontend) | Market dashboard renders charts — depends on these API endpoints being correct | Indirect dependency |

The P0 cross-tenant test (`cross-tenant-isolation.api.spec.ts`) is already generated in RED phase and will exercise the `/analytics/market/volume` endpoint directly. When this story is implemented, the test must go GREEN. This means the `company_id` WHERE clause must be present and the fixture infrastructure (`analyticsData(companyId)`) must seed the materialized view correctly.

The P1 test for S12.02 (single test row in the epic-level test design, ~1 QA hour) will be generated separately by TEA after this story is deployed to staging. The unit tests created in Task 6 are the primary automated coverage for this story.

---

## Senior Developer Review

**Review Date:** 2026-04-12 (Re-review)
**Verdict:** REVIEW: Approve
**Summary:** Previous 3 `patch` items (F1–F3) all resolved. 3 new minor `patch` items (test-level only), 2 `defer`, 3 dismissed (resolved). Production code is clean, architecturally aligned, and correctly implements all 9 acceptance criteria. Cross-tenant isolation verified at both API-layer (mock) and DB-layer (integration test). Approving — remaining patches are test hygiene improvements that do not affect production correctness.

### Review Findings — Round 1 (Resolved)

- [x] [Review][Patch] **F1 — tier_gate uses HTTPException instead of project's ForbiddenError** — ✅ RESOLVED. `tier_gate.py:83` now raises `ForbiddenError` from `eusolicit_common.exceptions`. Test assertions updated to check `body["details"]["upgrade_required"]`.

- [x] [Review][Patch] **F2 — Cross-tenant isolation test only verifies parameter forwarding** — ✅ RESOLVED. `TestCrossTenantServiceSQLIsolation` class added (lines 1180–1374) with full DB-level verification: seeds source tables for two companies via `migration_role`, refreshes the materialized view via `notification_role`, queries via `client_api_role`, asserts Company B's data absent from Company A's results.

- [x] [Review][Patch] **F3 — Deprecated `asyncio.get_event_loop().run_until_complete()` in test** — ✅ RESOLVED. `test_require_professional_plus_tier_stub_raises_not_implemented` now uses `async def` + `@pytest.mark.asyncio` + `await`.

- [x] [Review][Defer] **F4 — No date_from > date_to validation** [`services/client-api/src/client_api/services/analytics_market_service.py:38-59`] — deferred, not in spec
  - If a client passes `date_from=2025-12-01&date_to=2025-01-01`, the query silently returns empty results. A 400 error with a message would be more user-friendly. Not specified in the AC — defer to a future usability pass.

### Review Findings — Round 2 (Current)

- [ ] [Review][Patch] **F5 — Missing `@pytest.mark.integration` on DB cross-tenant test** [`services/client-api/tests/api/test_analytics_market.py:1199`]
  - `TestCrossTenantServiceSQLIsolation` creates real PostgreSQL connections to 3 different DB roles (`migration_role`, `notification_role`, `client_api_role`) and issues DDL/DML operations. Per project-context.md rule 17 and the test level definitions, this is an integration test requiring live Docker services.
  - Without the `@pytest.mark.integration` marker, `pytest -m "not integration"` (or unit-only CI stages) will attempt to run this test and fail with connection errors.
  - **Fix:** Add `@pytest.mark.integration` to the class or the test method.

- [ ] [Review][Patch] **F6 — Test fixtures missing `await session.rollback()` in finally block** [`services/client-api/tests/api/test_analytics_market.py:259,346`]
  - `paid_market_context` (line 259) and `free_market_context` (line 346) only call `fastapi_app.dependency_overrides.clear()` in their `finally` blocks. The project's gold standard pattern from Epic 2 retrospective requires both `dependency_overrides.clear()` AND `await session.rollback()`. The `async with session_factory()` context manager handles session closure, but the explicit rollback is the established convention for guaranteed test isolation.
  - **Fix:** Add `await session.rollback()` after `dependency_overrides.clear()` in both fixtures' `finally` blocks.

- [ ] [Review][Patch] **F7 — Cross-tenant DB test has no cleanup guarantee on assertion failure** [`services/client-api/tests/api/test_analytics_market.py:1199-1374`]
  - The cleanup steps (Step 5: DELETE from source tables + final REFRESH) at lines 1342–1374 only execute if assertions in Step 4 pass. If `assert total_a > 0` or the cross-tenant assertions fail, the `AssertionError` propagates and cleanup is skipped, leaving orphaned rows in `pipeline.opportunities` and `client.bid_decisions`.
  - **Fix:** Wrap the entire test body in an outer `try/finally` that guarantees cleanup runs regardless of assertion outcome. Alternatively, use a pytest fixture with teardown for the seeding.

- [x] [Review][Defer] **F8 — `_apply_market_filters.stmt` parameter lacks type annotation** [`services/client-api/src/client_api/services/analytics_market_service.py:39`] — deferred, cosmetic
  - The `stmt` parameter has no type annotation (should be `Select` from `sqlalchemy`). Project runs mypy, but this doesn't cause a type error because the parameter is used as a generic chainable object. Low-priority consistency improvement.
