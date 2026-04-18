---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-16'
workflowType: bmad-testarch-atdd
story_id: 5-6-eu-grants-crawler-task
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-6-eu-grants-crawler-task.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-app/services/data-pipeline/tests/conftest.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_crawl_aop.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py
  - _bmad/bmm/config.yaml
outputFiles:
  - eusolicit-app/services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_crawl_eu_grants.py
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
---

# ATDD Checklist — Story 5.6: EU Grants Crawler Task

**Date:** 2026-04-16
**Author:** TEA Master Test Architect
**Story:** 5-6 — EU Grants Crawler Task
**Status:** 🔴 RED PHASE — 14 existing tests GREEN; 4 new RED stubs added for uncovered scenarios
**Stack:** backend (Python / Celery / SQLAlchemy / PostgreSQL / pytest)
**Generation Mode:** AI Generation (no recording needed — backend-only task)

---

## Step 1: Preflight & Context

### Stack Detection

- **Detected stack:** `backend`
- **Indicators:** `pyproject.toml` with `celery`, `sqlalchemy`, `alembic`, `testcontainers` deps; `conftest.py` with `PostgresContainer` and `eager_celery` fixtures; no `playwright.config.ts` or frontend framework indicators in `services/data-pipeline/`
- **Test framework:** pytest 8.x with `pytest-asyncio`, `respx`, `testcontainers[postgres]`, `freezegun`

### Prerequisites Verified

- [x] Story 5.6 approved with 8 clear acceptance criteria (AC1–AC8)
- [x] Backend test framework configured: `conftest.py` with `PostgresContainer`, Alembic migrations, `eager_celery`, `eager_celery_no_propagate`, `_clean_tables`, `_latest_run`, `_opp_count` fixtures
- [x] Existing test patterns available: `test_crawl_aop.py` (direct structural template for EU Grants), `test_crawl_ted.py`
- [x] AI Gateway mocking in place: `respx` router configured at `http://ai-gateway:8004`
- [x] Fixture generators available: `_make_eu_grants_raw`, `_make_eu_grants_normalized` in `conftest.py`

### Context Loaded

- **Story:** 5.6 — EU Grants Crawler Task — `crawl_eu_grants` Celery task with two-hop agent chain, EU-Grants-specific normalizer, grant-specific JSONB fields, budget parser, `crawler_runs` audit trail, on_failure signal handler
- **Epic test design:** `test-design-epic-05.md` — E05-P0-006 (field completeness), E05-P1-016 (budget parsing), E05-P1-017 (opportunity_type=grant), E05-P2-007 (crawler_runs accuracy)
- **Fixture infrastructure:** `conftest.py` provides `pg_container` (session-scoped testcontainer with Alembic migrations), `_psycopg2_conn`, `_opp_count`, `_latest_run`, `eager_celery`, `eager_celery_no_propagate`, `_make_eu_grants_raw(n)`, `_make_eu_grants_normalized(n)`
- **Agent response shape:** EU Grant Portal Agent → single call (no pagination), returns `{"opportunities": [...]}` with `budget_value`/`budget_range` (not standard `budget` dict), plus `evaluation_criteria` JSONB and `mandatory_documents` list
- **Key differences from AOP/TED:** no pagination, flexible budget formats, two grant-specific JSONB columns, `opportunity_type` must be in `update_cols`

---

## Step 2: Generation Mode

**Mode selected:** AI Generation

**Reason:** `detected_stack = backend`. All scenarios are Celery task execution tests, normalizer unit tests, and database state assertions — no browser automation or UI interaction needed. Full AI generation from acceptance criteria, story dev notes, and existing test patterns.

---

## Step 3: Test Strategy

### AC → Test ID Mapping

| AC | Description | Test ID(s) | Level | Priority | Epic Test ID | File |
|----|-------------|------------|-------|----------|--------------|------|
| AC1 | opportunity_type='grant' set on every upserted record | E05-S6-001 | Integration | P0 | E05-P1-017 | integration |
| AC1 | evaluation_criteria JSONB preserved from agent response | E05-S6-002 | Unit | P0 | E05-P0-006 | unit |
| AC1 | evaluation_criteria populated in DB after full crawl | E05-S6-003 | Integration | P0 | E05-P0-006 | integration |
| AC1 | opportunity_type='grant' preserved after ON CONFLICT re-crawl | E05-S6-004 | Integration | P0 | E05-P1-017 | integration (**RED**) |
| AC2 | mandatory_documents JSONB preserved from agent response | E05-S6-005 | Unit | P1 | E05-P0-006 | unit |
| AC2 | mandatory_documents populated in DB after full crawl | E05-S6-006 | Integration | P1 | E05-P0-006 | integration |
| AC2 | mandatory_documents=NULL when omitted by agent | E05-S6-007 | Unit | P2 | — | unit |
| AC3 | budget_value=scalar → budget_min=budget_max=scalar | E05-S6-008 | Unit | P1 | E05-P1-016 | unit |
| AC3 | budget_range="X-Y" → budget_min=X, budget_max=Y | E05-S6-009 | Unit | P1 | E05-P1-016 | unit |
| AC3 | budget_range with whitespace stripped correctly | E05-S6-010 | Unit | P2 | E05-P1-016 | unit |
| AC3 | standard budget dict {min, max, currency} parsed | E05-S6-011 | Unit | P2 | — | unit |
| AC3 | no budget fields → budget_min=None, budget_max=None | E05-S6-012 | Unit | P2 | — | unit |
| AC4 | successful crawl → opportunities in DB with source_type='eu_grants' | E05-S6-013 | Integration | P0 | E05-P2-007 | integration |
| AC4 | crawler_runs record: status='completed', ended_at populated | E05-S6-014 | Integration | P0 | E05-P2-007 | integration |
| AC4 | crawler_runs counts: found, new_count, updated accurate | E05-S6-015 | Integration | P0 | E05-P2-007 | integration |
| AC5 | same source_id+source_type → ON CONFLICT update, no duplicate row | E05-S6-016 | Integration | P0 | E05-P0-001 | integration |
| AC5 | second run: new_count=0, found=N (not creating new rows) | E05-S6-017 | Integration | P0 | E05-P2-007 | integration |
| AC6 | AIGatewayUnavailableError triggers Celery retry mechanism | E05-S6-018 | Integration | P0 | — | integration (**RED**) |
| AC6 | crawler_runs.status='retrying' set during retry attempt | E05-S6-019 | Integration | P1 | — | integration (**RED**) |
| AC6 | after 3 retries exhausted: status='failed', errors populated | E05-S6-020 | Integration | P0 | E05-R-002 | integration |
| AC7 | on_failure handler transitions run to 'failed' for non-retriable exception | E05-S6-021 | Integration | P0 | E05-R-002 | integration (**RED**) |
| AC7 | no crawler_runs record remains in 'running' or 'retrying' after task finalization | E05-S6-022 | Integration | P1 | E05-R-002 | integration |
| AC8 | integration test — happy path with grant-specific fields populated | E05-S6-023 | Integration | P0 | E05-P0-006 | integration |
| AC8 | integration test — gateway-down → status='failed' after max retries | E05-S6-024 | Integration | P0 | E05-R-002 | integration |
| AC8 | unit test suite covers all budget parsing cases and field passthrough | E05-S6-025 | Unit | P1 | E05-P1-016 | unit |

### Test Level Selection

- **Unit tests (10 tests):** Pure function tests of `normalize_eu_grants_response()` and `_parse_eu_grants_budget()` — no DB, no Celery, fast execution
- **Integration tests (8 existing + 4 RED stubs):** Full Celery task execution with testcontainers PostgreSQL — verifies DB state, `crawler_runs` accuracy, AI Gateway mock interaction
- **No E2E tests:** Backend-only story; no frontend/API endpoints involved
- **No Pact tests:** EU Grants doesn't expose new HTTP endpoints; it is a consumer of AI Gateway (covered by AC coverage + respx mocks)

### Red Phase Requirements

All 4 RED stub tests must fail at collection/execution time. They use `pytest.fail()` with descriptive messages explaining what the test should verify once implemented.

---

## Step 4: Generated Tests

### 4A — Unit Tests (Backend)

**File:** `services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py`

| Test Function | Test ID | AC | Priority | Epic ID | Status |
|---------------|---------|-----|----------|---------|--------|
| `test_eu_grants_budget_single_value` | E05-S6-008 | AC3 | P1 | E05-P1-016 | ✅ GREEN |
| `test_eu_grants_budget_range_string` | E05-S6-009 | AC3 | P1 | E05-P1-016 | ✅ GREEN |
| `test_eu_grants_budget_range_with_whitespace` | E05-S6-010 | AC3 | P2 | E05-P1-016 | ✅ GREEN |
| `test_eu_grants_budget_standard_dict` | E05-S6-011 | AC3 | P2 | — | ✅ GREEN |
| `test_eu_grants_budget_absent` | E05-S6-012 | AC3 | P2 | — | ✅ GREEN |
| `test_eu_grants_opportunity_type_always_grant` | E05-S6-001 | AC1 | P0 | E05-P1-017 | ✅ GREEN |
| `test_eu_grants_evaluation_criteria_preserved` | E05-S6-002 | AC1 | P0 | E05-P0-006 | ✅ GREEN |
| `test_eu_grants_mandatory_documents_preserved` | E05-S6-005 | AC2 | P1 | E05-P0-006 | ✅ GREEN |
| `test_eu_grants_source_type_always_eu_grants` | E05-S6-013 | AC4 | P0 | — | ✅ GREEN |
| `test_eu_grants_missing_optional_fields` | E05-S6-007/E05-S6-012 | AC1,2,3 | P2 | — | ✅ GREEN |

**Unit test count:** 10 functions | **Status:** 10 GREEN, 0 RED

**Coverage provided:**
- Full `_parse_eu_grants_budget()` — all three input cases (scalar, range string, standard dict) plus absent budget
- `normalize_eu_grants_response()` — opportunity_type hardcoding, source_type hardcoding, JSONB field passthrough, missing-field defaults

### 4B — Integration Tests (Backend)

**File:** `services/data-pipeline/tests/integration/test_crawl_eu_grants.py`

**Fixture requirements:**
- `pg_container` (session-scoped PostgresContainer with Alembic migrations)
- `eager_celery` / `eager_celery_no_propagate` (sync task execution)
- `_clean_tables` (TRUNCATE before each test)
- `_opp_count(source_type)`, `_latest_run(crawler_type)`, `_psycopg2_conn()`
- `_make_eu_grants_raw(n)`, `_make_eu_grants_normalized(n)` (in `conftest.py`)
- `respx.mock(base_url=AI_GW_BASE)` (inline per test)

**Integration test functions:**

| Test Function | Test ID | AC | Priority | Epic ID | Status |
|---------------|---------|-----|----------|---------|--------|
| `test_crawl_eu_grants_field_completeness` | E05-S6-003,6,13,14,15,23 | AC1,2,4,8 | P0 | E05-P0-006, E05-P2-007 | ✅ GREEN |
| `test_crawl_eu_grants_opportunity_type_grant` | E05-S6-001 | AC1 | P1 | E05-P1-017 | ✅ GREEN |
| `test_crawl_eu_grants_crawler_runs_accuracy` | E05-S6-015,16,17,22 | AC4,5,7 | P2 | E05-P2-007 | ✅ GREEN |
| `test_crawl_eu_grants_gateway_down` | E05-S6-020,24 | AC6,7,8 | P0 | E05-R-002 | ✅ GREEN |
| `test_eu_grants_opportunity_type_preserved_on_reconflict` | E05-S6-004 | AC5 | P0 | E05-P1-017 | 🔴 RED |
| `test_eu_grants_retry_mechanism_triggered` | E05-S6-018 | AC6 | P0 | — | 🔴 RED |
| `test_eu_grants_status_retrying_during_retry_attempt` | E05-S6-019 | AC6 | P1 | — | 🔴 RED |
| `test_eu_grants_on_failure_handler_for_non_retriable_exception` | E05-S6-021 | AC7 | P0 | E05-R-002 | 🔴 RED |

**Integration test count:** 8 functions (4 existing GREEN + 4 new RED stubs) | **Status:** 4 GREEN, 4 RED

---

## Step 5: Validation & Completion

### Coverage Validation

| ATDD-5.6 | Description | Test Function(s) | Level | Status |
|----------|-------------|-----------------|-------|--------|
| ATDD-5.6-01 | opportunity_type='grant' on every upserted record | `test_eu_grants_opportunity_type_always_grant` (unit) + `test_crawl_eu_grants_opportunity_type_grant` (integration) | Unit+Integration | ✅ GREEN |
| ATDD-5.6-02 | evaluation_criteria JSONB preserved | `test_eu_grants_evaluation_criteria_preserved` (unit) + `test_crawl_eu_grants_field_completeness` (integration) | Unit+Integration | ✅ GREEN |
| ATDD-5.6-03 | opportunity_type='grant' preserved after ON CONFLICT re-crawl | `test_eu_grants_opportunity_type_preserved_on_reconflict` | Integration | 🔴 RED |
| ATDD-5.6-04 | mandatory_documents JSONB preserved | `test_eu_grants_mandatory_documents_preserved` (unit) + `test_crawl_eu_grants_field_completeness` (integration) | Unit+Integration | ✅ GREEN |
| ATDD-5.6-05 | mandatory_documents=NULL when omitted | `test_eu_grants_missing_optional_fields` | Unit | ✅ GREEN |
| ATDD-5.6-06 | budget_value=scalar → budget_min=budget_max | `test_eu_grants_budget_single_value` | Unit | ✅ GREEN |
| ATDD-5.6-07 | budget_range="X-Y" → correct split | `test_eu_grants_budget_range_string` | Unit | ✅ GREEN |
| ATDD-5.6-08 | budget_range with whitespace stripped | `test_eu_grants_budget_range_with_whitespace` | Unit | ✅ GREEN |
| ATDD-5.6-09 | standard budget dict parsed | `test_eu_grants_budget_standard_dict` | Unit | ✅ GREEN |
| ATDD-5.6-10 | no budget fields → NULL | `test_eu_grants_budget_absent` | Unit | ✅ GREEN |
| ATDD-5.6-11 | crawler_runs record created with correct crawler_type and status | `test_crawl_eu_grants_field_completeness` + `test_crawl_eu_grants_crawler_runs_accuracy` | Integration | ✅ GREEN |
| ATDD-5.6-12 | crawler_runs counts (found, new, updated) accurate | `test_crawl_eu_grants_crawler_runs_accuracy` + `test_crawl_eu_grants_field_completeness` | Integration | ✅ GREEN |
| ATDD-5.6-13 | 5 new opportunities → new_count=5, updated=0 | `test_crawl_eu_grants_field_completeness` | Integration | ✅ GREEN |
| ATDD-5.6-14 | same data again → no new rows created | `test_crawl_eu_grants_crawler_runs_accuracy` (2nd run) | Integration | ✅ GREEN |
| ATDD-5.6-15 | AIGatewayUnavailableError triggers Celery retry | `test_eu_grants_retry_mechanism_triggered` | Integration | 🔴 RED |
| ATDD-5.6-16 | status='retrying' during retry attempt | `test_eu_grants_status_retrying_during_retry_attempt` | Integration | 🔴 RED |
| ATDD-5.6-17 | after 3 retries → status='failed', errors populated | `test_crawl_eu_grants_gateway_down` | Integration | ✅ GREEN |
| ATDD-5.6-18 | non-retriable exception → on_failure → status='failed' | `test_eu_grants_on_failure_handler_for_non_retriable_exception` | Integration | 🔴 RED |
| ATDD-5.6-19 | integration happy path — all grant-specific fields + counts correct | `test_crawl_eu_grants_field_completeness` | Integration | ✅ GREEN |
| ATDD-5.6-20 | integration gateway-down → status='failed' after max retries | `test_crawl_eu_grants_gateway_down` | Integration | ✅ GREEN |
| ATDD-5.6-21 | unit test suite covers all budget parsing cases | `test_crawl_eu_grants_normalizer.py` (10 tests) | Unit | ✅ GREEN |

### AC Coverage Summary

| AC | Tests | RED stubs | Status |
|----|-------|-----------|--------|
| AC1: opportunity_type + evaluation_criteria | 4 | 1 (ATDD-5.6-03) | Partial — ON CONFLICT re-crawl check missing |
| AC2: mandatory_documents JSONB | 3 | 0 | ✅ Full |
| AC3: budget parsing (all formats) | 5 | 0 | ✅ Full |
| AC4: crawl execution + counts | 4 | 0 | ✅ Full |
| AC5: atomic deduplication | 2 | 1 (ATDD-5.6-03) | Partial — opportunity_type in update_cols not verified |
| AC6: retry on AIGatewayUnavailableError | 2 | 2 (ATDD-5.6-15, 5.6-16) | Partial — retry mechanism + retrying status not verified |
| AC7: terminal state guarantee | 2 | 1 (ATDD-5.6-18) | Partial — on_failure for non-retriable not tested |
| AC8: integration test coverage | 3 | 0 | ✅ Full |

### Epic Test ID Traceability

| Epic Test ID | Priority | Scenario | Test Function | Status |
|--------------|----------|----------|---------------|--------|
| E05-P0-006 | P0 | EU Grants field completeness — evaluation_criteria + mandatory_documents populated | `test_crawl_eu_grants_field_completeness` | ✅ GREEN |
| E05-P1-016 | P1 | Budget range parsing — scalar value and range string | `test_eu_grants_budget_single_value`, `test_eu_grants_budget_range_string`, `test_eu_grants_budget_range_with_whitespace` | ✅ GREEN |
| E05-P1-017 | P1 | opportunity_type='grant' — all EU Grant records | `test_crawl_eu_grants_opportunity_type_grant` (integration), `test_eu_grants_opportunity_type_always_grant` (unit), `test_eu_grants_opportunity_type_preserved_on_reconflict` | ✅/🔴 Partial |
| E05-P2-007 | P2 | crawler_runs record accuracy — correct source_type, timestamps, counts | `test_crawl_eu_grants_crawler_runs_accuracy` | ✅ GREEN |
| E05-R-001 | — | Dedup race — atomic upsert, no duplicate rows | `test_crawl_eu_grants_crawler_runs_accuracy` (2nd run checks 0 new) | ✅ GREEN |
| E05-R-002 | — | AI Gateway cascade / orphaned runs | `test_crawl_eu_grants_gateway_down`, `test_eu_grants_on_failure_handler_for_non_retriable_exception` | ✅/🔴 Partial |

### Quality Gate Assessment

| Criterion | Target | Current | Status |
|-----------|--------|---------|--------|
| AC coverage | 100% | 17 of 21 ATDD items GREEN | ⚠️ 4 RED stubs remaining |
| P0 test pass rate | 100% | 4 P0 GREEN, 2 P0 RED stubs | ❌ P0 incomplete (ATDD-5.6-03, 5.6-18) |
| P1 test pass rate | ≥95% | 5 P1 GREEN, 2 P1 RED stubs | ⚠️ 71% (ATDD-5.6-15, 5.6-16) |
| Unit coverage ≥80% | `workers/tasks/` | 10 unit tests on normalizer | ✅ Normalizer fully covered |
| Integration audit trail | All 3 crawler types | EU Grants covered | ✅ |
| E05-R-002 mitigation | on_failure handler verified | gateway_down covers partial; non-retriable path missing | ⚠️ Partial |

---

## Summary Statistics

```
🔴 TDD ATDD Generation — Story 5.6 EU Grants Crawler Task

📊 Test Inventory:
  Unit tests (test_crawl_eu_grants_normalizer.py):   10  ✅ GREEN
  Integration tests (test_crawl_eu_grants.py):
    Existing passing:                                  4  ✅ GREEN
    New RED stubs (this workflow):                     4  🔴 RED
  ─────────────────────────────────────────────────────
  Total tests:                                        18
  GREEN (implementation complete):                    14
  RED (requiring implementation/completion):           4

📋 AC Coverage:
  Fully covered (GREEN):   AC2, AC3, AC4, AC8
  Partially covered:       AC1, AC5, AC6, AC7
  Uncovered:               none

🎯 Epic Test IDs covered:
  E05-P0-006 ✅ | E05-P1-016 ✅ | E05-P1-017 ⚠️ | E05-P2-007 ✅

📂 Files:
  services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py  (10 tests, all GREEN)
  services/data-pipeline/tests/integration/test_crawl_eu_grants.py      (8 tests, 4 GREEN + 4 RED)

📝 Next Steps (TDD Green Phase for RED stubs):
  1. ATDD-5.6-03: Implement test asserting opportunity_type='grant' in DB after 2nd crawl run
     → verify _upsert.py update_cols includes opportunity_type
  2. ATDD-5.6-15: Implement test verifying Celery retry count (mock call invocations = 4)
     → respx mock for 4 consecutive 503s; assert mock.call_count == 4
  3. ATDD-5.6-16: Implement test verifying crawler_runs.status='retrying' between retries
     → side-effect callback on retry mock reads DB before re-raise
  4. ATDD-5.6-18: Implement test verifying on_failure handler for non-AIGateway exceptions
     → monkeypatch crawl internals to raise RuntimeError; assert run.status='failed'
  5. Once implemented, remove pytest.fail() and run: cd services/data-pipeline && pytest tests/ -v
  6. Verify all 18 tests pass (green phase complete)
  7. Run: bmad-testarch-automate for broader coverage expansion
```

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
