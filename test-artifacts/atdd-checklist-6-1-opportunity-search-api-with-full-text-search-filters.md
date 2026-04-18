---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-17'
workflowType: bmad-testarch-atdd
mode: create
storyId: 6-1-opportunity-search-api-with-full-text-search-filters
detectedStack: backend
generationMode: ai-generation
executionMode: sequential
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-1-opportunity-search-api-with-full-text-search-filters.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.1 — Opportunity Search API with Full-Text Search & Filters

**Date:** 2026-04-17
**Author:** TEA Master Test Architect (bmad-testarch-atdd)
**TDD Phase:** 🔴 RED — All tests generated as failing (feature not yet implemented)
**Stack:** Backend (Python/FastAPI/pytest)
**Story:** `eusolicit-docs/implementation-artifacts/6-1-opportunity-search-api-with-full-text-search-filters.md`

---

## Step 1: Preflight & Context

### Stack Detection
- **Detected stack:** `backend`
- **Indicators:** `pyproject.toml`, Python FastAPI service at `eusolicit-app/services/client-api/`
- **No frontend indicators:** No `playwright.config.ts`, no `package.json` in service root
- **Test framework:** pytest + pytest-asyncio (confirmed from existing conftest.py patterns)
- **TEA config flags:** Not explicitly set in `config.yaml`; all defaults applied

### Prerequisites Verified
- [x] Story has clear acceptance criteria (10 ACs specified)
- [x] Test framework present: `pyproject.toml` + `conftest.py` with pytest-asyncio fixtures
- [x] Epic-level test design loaded: `test-design-epic-06.md` (E06-P1-001 through E06-P2-002 identified)
- [x] Existing test patterns reviewed: `tests/api/test_espd_profile.py`, `test_auth_middleware.py`
- [x] Available conftest fixtures identified: `app`, `test_client`, `free_user_token`, `migration_engine`

### Loaded Artifacts
| Artifact | Purpose |
|----------|---------|
| Story 6.1 markdown | Acceptance criteria, task breakdown, test code guidance |
| `test-design-epic-06.md` | Epic-level test IDs (E06-P1-001 through E06-P3-005) and risk links |
| `services/client-api/tests/conftest.py` | Fixture inventory: app, test_client, free_user_token, migration_engine |
| `eusolicit_test_utils/auth.py` | `generate_test_jwt()` signature for token generation |

### Implementation Status (Confirmed ABSENT — RED phase valid)
- ❌ `services/client-api/src/client_api/api/v1/opportunities.py` — does not exist
- ❌ `services/client-api/src/client_api/services/opportunity_service.py` — does not exist
- ❌ `services/client-api/src/client_api/models/pipeline_opportunity.py` — does not exist
- ❌ `services/client-api/src/client_api/schemas/opportunities.py` — does not exist
- ❌ `get_pipeline_readonly_session` in `dependencies.py` — does not exist
- ❌ `api/v1/opportunities` router in `main.py` — not registered

---

## Step 2: Generation Mode

**Mode selected:** AI Generation (backend stack — no browser recording needed)
**Rationale:** All ACs are backend API specifications (HTTP status codes, JSON shapes, SQL semantics). AI generation from acceptance criteria and test design is appropriate.

---

## Step 3: Test Strategy

### AC → Test Mapping

| AC | Description | Test Level | Priority | Test ID |
|----|-------------|------------|----------|---------|
| AC1 | Endpoint registered at `/api/v1/opportunities/search` in OpenAPI | Integration | P3 | `S06-AC1` |
| AC2 (FTS) | `q` param uses `websearch_to_tsquery`; absent `q` = browse mode | Integration | P1 | `E06-P1-001` |
| AC2 (browse) | No `q` = browse mode, returns all matching | Integration | P1 | `S06-AC2-browse` |
| AC3 (filters) | All six filter params as AND conditions | Integration | P1 | `E06-P1-002` |
| AC3 (validation) | Invalid enum values return 422 or 400 | Integration | P2 | `S06-AC3a` |
| AC4 (default) | Default limit=25 | Integration | P2 | `S06-AC4a` |
| AC4 (boundary) | limit 1–100 accepted; >100 → 422 | Integration | P1 | `E06-P1-004` |
| AC4 (pagination) | after_cursor/before_cursor pagination, total_count stable | Integration | P1 | `E06-P1-003` |
| AC5 (cursor format) | Cursor is opaque base64 JSON {published_at, id} | Integration | P2 | `S06-AC5a` |
| AC5 (invalid cursor) | Invalid cursor → 400 with error=invalid_cursor | Integration | P2 | `E06-P2-002` |
| AC6 (read-only) | Uses `get_pipeline_readonly_session` dependency | Integration | P1 | (via app fixture) |
| AC6 (soft-delete) | `deleted_at IS NOT NULL` rows excluded | Integration | P1 | `S06-AC6` |
| AC7 (auth) | Unauthenticated → 401 | Integration | P1 | `S06-AC7` |
| AC8 (empty) | No matches → 200 with `{results:[], total_count:0, cursors:null}` | Integration | P2 | `E06-P2-001` |
| AC9 (fields) | All required response fields present | Integration | P1 | `S06-AC9b` |
| AC9 (rank) | `relevance_rank` non-null when `q` provided; null in browse | Integration | P1 | `S06-AC9a` |

### Test Levels Selected
- **All tests: Integration** (in-process FastAPI via HTTPX `ASGITransport`)
- **Rationale:** Story 6.1 is a new API endpoint with no existing frontend. Integration tests against the running FastAPI app (with dependency overrides for pipeline session and auth) are the right level — they exercise the full request-response cycle including SQL query generation, cursor encoding, and filter composition without requiring a live Docker Compose environment.
- **No unit tests** for this story: Business logic in `opportunity_service.py` is tightly coupled to SQLAlchemy Core expressions; testing the SQL generation requires a real PostgreSQL to verify `websearch_to_tsquery` and array overlap semantics.
- **No E2E tests**: Frontend for opportunity discovery (S06.09–S06.14) is a separate story. Backend-only ATDD applies here.

### Red Phase Requirements Confirmed
All generated tests:
- ✅ Are marked with `@pytest.mark.skip(reason="🔴 ATDD RED PHASE — S06.01 not implemented")`
- ✅ Contain correct, complete assertions for expected behavior
- ✅ Will naturally FAIL (ImportError, 404, or assertion error) when skip is removed before implementation
- ✅ Reference real fixtures from conftest.py (`app`, `test_client`, `free_user_token`)
- ✅ Use realistic test data (not placeholder assertions like `assert True`)

---

## Step 4: Generated Test Files

### 🔴 TDD Red Phase — Failing Tests Generated

**Test file:** `services/client-api/tests/api/test_opportunity_search.py`

| Test Function | Test ID | AC | Priority | Expected failure mode |
|--------------|---------|-----|----------|----------------------|
| `test_fts_search_returns_matching_opportunities` | E06-P1-001 | AC2, AC9 | P1 | `ImportError` → 404 on endpoint |
| `test_all_six_filters_compose_as_and` | E06-P1-002 | AC3 | P1 | `ImportError` → 404 on endpoint |
| `test_cursor_pagination_forward_and_backward` | E06-P1-003 | AC4, AC5 | P1 | `ImportError` → 404 on endpoint |
| `test_limit_boundary_values[1-200]` | E06-P1-004 | AC4 | P1 | `ImportError` → 404 on endpoint |
| `test_limit_boundary_values[25-200]` | E06-P1-004 | AC4 | P1 | `ImportError` → 404 on endpoint |
| `test_limit_boundary_values[100-200]` | E06-P1-004 | AC4 | P1 | `ImportError` → 404 on endpoint |
| `test_limit_boundary_values[101-422]` | E06-P1-004 | AC4 | P1 | `ImportError` → 404 returns wrong code |
| `test_empty_results_returns_200_with_empty_shape` | E06-P2-001 | AC8 | P2 | `ImportError` → 404 on endpoint |
| `test_invalid_cursor_returns_400[...]` × 4 | E06-P2-002 | AC5 | P2 | `ImportError` → wrong status code |
| `test_endpoint_registered_in_openapi` | S06-AC1 | AC1 | P3 | Path missing from OpenAPI schema |
| `test_invalid_status_enum_returns_422_or_400` | S06-AC3a | AC3 | P2 | `ImportError` → 404 returns wrong code |
| `test_soft_deleted_rows_never_appear` | S06-AC6 | AC6 | P1 | `ImportError` → soft-deleted rows visible |
| `test_unauthenticated_request_returns_401` | S06-AC7 | AC7 | P1 | `ImportError` → 404 instead of 401 |
| `test_relevance_rank_present_when_q_provided` | S06-AC9a | AC9 | P1 | `ImportError` → missing field |
| `test_response_fields_include_all_required_ac9_fields` | S06-AC9b | AC9 | P1 | `ImportError` → missing fields |
| `test_default_limit_is_25` | S06-AC4a | AC4 | P2 | `ImportError` → wrong count |
| `test_cursor_is_opaque_base64_json` | S06-AC5a | AC5 | P2 | `ImportError` → cursor shape wrong |
| `test_browse_mode_returns_all_visible_rows` | S06-AC2 | AC2 | P1 | `ImportError` → wrong count |

**Total tests: 18** (including 4 parametrize cases for invalid cursor and 4 for limit boundary)  
**All tests: 🔴 SKIPPED (ATDD RED PHASE)**

### New Fixtures Defined in Test File
The following fixtures are defined locally in `test_opportunity_search.py` (module scope) and
should be promoted to the shared `tests/conftest.py` after S06.01 ships:

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `superuser_engine` | module | create_async_engine with migration_role credentials (full schema access) |
| `superuser_session_factory` | module | async_sessionmaker backed by superuser_engine |
| `superuser_session` | module | long-lived session for seeding pipeline.opportunities |
| `seeded_pipeline_opportunities` | module | inserts 11 rows (10 visible + 1 soft-deleted), tears down after module |
| `search_app` | function | overrides `get_pipeline_readonly_session` dep with superuser factory |

### Seed Data Summary
| Row ID | source_id | Type | Status | FTS match | CPV | Region | Visible? |
|--------|-----------|------|--------|-----------|-----|--------|---------|
| opp-001 | aop | tender | open | "road construction" | 45233100-0 | Bulgaria | ✅ |
| opp-002 | ted | tender | open | "software development" | 72000000-5 | Romania | ✅ |
| opp-003 | eu_grants | grant | open | "green energy" | 09300000-2 | EU | ✅ |
| opp-004 | aop | tender | closed | "water infrastructure" | 65100000-4 | Bulgaria | ✅ |
| opp-deleted | aop | tender | open | "Deleted road tender" | 45233100-0 | Bulgaria | ❌ (deleted_at set) |
| opp-page-005…010 | aop | tender | open | "Pagination test N" | 45000000-7 | Bulgaria | ✅ ×6 |

**Visible rows: 10** | **Total seeded: 11** (including 1 soft-deleted)

---

## Step 5: Validation & Completion

### Pre-implementation Checklist (RED Phase)

- [x] All tests marked with `@pytest.mark.skip(reason="🔴 ATDD RED PHASE — S06.01 not implemented")`
- [x] All tests contain realistic, correct assertions (not `assert True` placeholders)
- [x] All 10 ACs from Story 6.1 are covered by at least one test
- [x] Epic-level test IDs from `test-design-epic-06.md` are referenced: E06-P1-001, E06-P1-002, E06-P1-003, E06-P1-004, E06-P2-001, E06-P2-002
- [x] Soft-deleted row seeded to prove AC6 exclusion
- [x] Pagination test uses ≥9 non-deleted rows (10 seeded) to ensure ≥3 pages at limit=3
- [x] Fixture needs documented (superuser_session, search_app, seeded_pipeline_opportunities)
- [x] Pipeline schema prerequisite documented (migration 002_pipeline_tables.py)
- [x] No temp files left in random locations — test file is in canonical location

### Acceptance Criteria Coverage Matrix

| AC | Description | Tests |
|----|-------------|-------|
| AC1 | Endpoint in OpenAPI schema | `test_endpoint_registered_in_openapi` |
| AC2 | FTS with websearch_to_tsquery; browse mode | `test_fts_search_returns_matching_opportunities`, `test_browse_mode_returns_all_visible_rows` |
| AC3 | Six filter params as AND; invalid enum → 4xx | `test_all_six_filters_compose_as_and`, `test_invalid_status_enum_returns_422_or_400` |
| AC4 | Cursor pagination; limit 1–100; default 25 | `test_cursor_pagination_forward_and_backward`, `test_limit_boundary_values[*]`, `test_default_limit_is_25` |
| AC5 | Cursor encoding; invalid cursor → 400 | `test_cursor_is_opaque_base64_json`, `test_invalid_cursor_returns_400[*]` |
| AC6 | Read-only session; deleted_at exclusion | `test_soft_deleted_rows_never_appear` (+ `search_app` fixture verifies session override) |
| AC7 | Bearer JWT required; 401 if absent | `test_unauthenticated_request_returns_401` |
| AC8 | Empty result → 200 with zeros/nulls | `test_empty_results_returns_200_with_empty_shape` |
| AC9 | All required fields; relevance_rank | `test_relevance_rank_present_when_q_provided`, `test_response_fields_include_all_required_ac9_fields` |
| AC10 | (Integration tests defined in this file cover P1-001–P2-002) | All test_* functions in the file |

### Risk Mitigations Verified by These Tests

| Risk ID | Risk | Tests that verify mitigation |
|---------|------|------------------------------|
| E06-R-005 | Cursor forgery (score 4) | `test_invalid_cursor_returns_400[*]` — invalid cursors return 400, no 500 |

---

## TDD Green Phase Instructions

When Story 6.1 implementation is complete, unskip tests in this order:

### Phase 1: Foundation (Tasks 1–3)
After implementing `dependencies.py` additions, `models/pipeline_opportunity.py`, and `schemas/opportunities.py`:

```bash
# Run only the OpenAPI test to confirm router registration
pytest services/client-api/tests/api/test_opportunity_search.py::test_endpoint_registered_in_openapi -v
```

Remove skip from:
- [ ] `test_endpoint_registered_in_openapi`
- [ ] `test_unauthenticated_request_returns_401`

### Phase 2: Core Search (Tasks 4–5)
After implementing `opportunity_service.py` and `api/v1/opportunities.py`:

```bash
pytest services/client-api/tests/api/test_opportunity_search.py -k "fts or browse or filter" -v
```

Remove skip from:
- [ ] `test_fts_search_returns_matching_opportunities`
- [ ] `test_browse_mode_returns_all_visible_rows`
- [ ] `test_all_six_filters_compose_as_and`
- [ ] `test_invalid_status_enum_returns_422_or_400`
- [ ] `test_soft_deleted_rows_never_appear`
- [ ] `test_response_fields_include_all_required_ac9_fields`
- [ ] `test_relevance_rank_present_when_q_provided`

### Phase 3: Pagination & Cursors
After verifying cursor encoding (`_encode_cursor`, `_decode_cursor` in `opportunity_service.py`):

```bash
pytest services/client-api/tests/api/test_opportunity_search.py -k "cursor or limit or pagination or empty or default" -v
```

Remove skip from:
- [ ] `test_cursor_pagination_forward_and_backward`
- [ ] `test_limit_boundary_values` (all 4 parametrize cases)
- [ ] `test_empty_results_returns_200_with_empty_shape`
- [ ] `test_invalid_cursor_returns_400` (all 4 parametrize cases)
- [ ] `test_default_limit_is_25`
- [ ] `test_cursor_is_opaque_base64_json`

### Full Run (Green Phase Target)
```bash
pytest services/client-api/tests/api/test_opportunity_search.py -v --tb=short
```

Expected: **18 passed**, 0 failed, 0 skipped

---

## Implementation Dependencies

### Files to Create (Story 6.1 Tasks)
| File | Story Task | Purpose |
|------|-----------|---------|
| `services/client-api/src/client_api/dependencies.py` (edit) | Task 1 | Add `get_pipeline_engine()`, `get_pipeline_readonly_session()` |
| `services/client-api/src/client_api/models/pipeline_opportunity.py` | Task 2 | SQLAlchemy Core `Table` mirroring `pipeline.opportunities` columns |
| `services/client-api/src/client_api/schemas/opportunities.py` | Task 3 | `OpportunitySearchItem`, `OpportunitySearchResponse` Pydantic models |
| `services/client-api/src/client_api/services/opportunity_service.py` | Task 4 | `search_opportunities()` — FTS + filter + cursor pagination logic |
| `services/client-api/src/client_api/api/v1/opportunities.py` | Task 5 | Router with `GET /search` endpoint |
| `services/client-api/src/client_api/main.py` (edit) | Task 6 | Register opportunities router |

### Environment Variables Required
| Variable | Default | Purpose |
|---------|---------|---------|
| `CLIENT_API_PIPELINE_DATABASE_URL` | falls back to `CLIENT_API_DATABASE_URL` | Read-only pipeline DB connection |
| `TEST_DATABASE_URL` | `postgresql+asyncpg://migration_role:...` | Test DB with pipeline schema |

### Database Prerequisite
```sql
-- Must exist before tests run (applied by data-pipeline migration 002_pipeline_tables.py)
CREATE SCHEMA IF NOT EXISTS pipeline;
CREATE TABLE pipeline.opportunities (
    id UUID PRIMARY KEY,
    source_id TEXT NOT NULL,
    source_type VARCHAR(20) NOT NULL,
    title TEXT,
    description TEXT,
    opportunity_type VARCHAR(20),
    status VARCHAR(20),
    deadline TIMESTAMPTZ,
    budget_min NUMERIC(18,2),
    budget_max NUMERIC(18,2),
    currency VARCHAR(3),
    country TEXT,
    region TEXT,
    contracting_authority TEXT,
    cpv_codes TEXT[],
    evaluation_criteria JSONB,
    mandatory_documents JSONB,
    relevance_scores JSONB,
    published_at TIMESTAMPTZ,
    deleted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (source_id, source_type)
);
```

---

## Notes & Assumptions

1. **`generate_test_jwt` uses HS256** — the conftest's `free_user_token` fixture calls `generate_test_jwt(role="member")` which produces an HS256 token. The `get_current_user` dependency in `core/security.py` uses RS256. The tests in this file inherit `free_user_token` from the shared conftest and the `rsa_env_setup` autouse fixture. If the security middleware validates HS256 in test mode, the fixture works as-is. If not, the test file's `search_app` fixture may need to override `get_current_user` to accept the test token format. This should be resolved during the implementation phase.

2. **Module-scoped seed** — `seeded_pipeline_opportunities` uses module scope so seed data is inserted once per test module run (not per test). This avoids 11 INSERT/DELETE pairs per test. Tests are designed to be non-destructive to the seeded rows.

3. **Pipeline schema ownership** — The `pipeline` schema and `opportunities` table are owned by the data-pipeline service. The client-api must use a SQLAlchemy Core `Table` definition (not the data-pipeline ORM model) to avoid service coupling. This is enforced by the test architecture (the client-api test imports only from `client_api.*` packages).

4. **Fixtures to promote** — `superuser_engine`, `superuser_session_factory`, `superuser_session` defined locally in this test file should be promoted to `tests/conftest.py` (session scope) or a `tests/pipeline_conftest.py` when more E06 stories are implemented (S06.04, S06.05 will also need pipeline schema access).

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd` (Create mode, sequential execution)
**Story:** 6.1 — Opportunity Search API with Full-Text Search & Filters
