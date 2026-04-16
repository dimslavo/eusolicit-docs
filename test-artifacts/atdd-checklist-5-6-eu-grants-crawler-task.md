---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-load-test-design', 'step-03-generate-failing-tests']
lastStep: 'step-03-generate-failing-tests'
lastSaved: '2026-04-16'
workflowType: 'testarch-atdd'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/5-6-eu-grants-crawler-task.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-05.md'
---

# ATDD Checklist - Epic 5, Story 6: EU Grants Crawler Task

**Date:** 2026-04-16
**Author:** TEA Master Test Architect
**Primary Test Level:** Unit & Integration

---

## Story Summary

A fully implemented `crawl_eu_grants` Celery task that coordinates the EU Grant Portal Agent and Data Normalization Team agent.

**As a** backend developer implementing the data pipeline
**I want** a fully implemented `crawl_eu_grants` Celery task
**So that** EU grant opportunities are automatically crawled daily, correctly typed as grants, stored with their scoring criteria, and completely audited.

---

## Acceptance Criteria

1. Grant opportunities stored with `opportunity_type='grant'` and populated `evaluation_criteria` JSONB (scoring matrix structure from the agent response) on every upserted record
2. `mandatory_documents` JSONB contains the structured list of required documents per grant call, as returned by the EU Grant Portal Agent and preserved through normalization
3. Budget range correctly parsed into `budget_min` / `budget_max`
4. Successful crawl inserts/updates opportunities with `source_type='eu_grants'` and records accurate counts in `crawler_runs` (`found`, `new_count`, `updated`, `started_at`, `ended_at`, `status=completed`)
5. Duplicate source records are updated via atomic `INSERT ... ON CONFLICT DO UPDATE`
6. AI Gateway failure triggers Celery-level retry with `max_retries=3`
7. `crawler_runs.status` is always transitioned to a terminal state (`completed` or `failed`)
8. Integration tests cover: happy path, gateway-down, `crawler_runs` record accuracy

---

## Failing Tests Created (RED Phase)

### Component Tests (10 tests)

**File:** `services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py` (44 lines)

- ✅ **Test:** test_eu_grants_budget_single_value
  - **Status:** RED - NotImplementedError
  - **Verifies:** Budget scalar values mapped to min/max
- ✅ **Test:** test_eu_grants_budget_range_string
  - **Status:** RED - NotImplementedError
  - **Verifies:** Budget range string split to min/max
- ✅ **Test:** test_eu_grants_budget_standard_dict
  - **Status:** RED - NotImplementedError
  - **Verifies:** Budget dict parsing matches existing format
- ✅ **Test:** test_eu_grants_budget_absent
  - **Status:** RED - NotImplementedError
  - **Verifies:** Missing budget gracefully handles None defaults
- ✅ **Test:** test_eu_grants_opportunity_type_always_grant
  - **Status:** RED - NotImplementedError
  - **Verifies:** Ensure grant opportunity_type
- ✅ **Test:** test_eu_grants_evaluation_criteria_preserved
  - **Status:** RED - NotImplementedError
  - **Verifies:** JSONB matrix passed through
- ✅ **Test:** test_eu_grants_mandatory_documents_preserved
  - **Status:** RED - NotImplementedError
  - **Verifies:** JSONB list passed through
- ✅ **Test:** test_eu_grants_source_type_always_eu_grants
  - **Status:** RED - NotImplementedError
  - **Verifies:** Override source type returned from agent
- ✅ **Test:** test_eu_grants_missing_optional_fields
  - **Status:** RED - NotImplementedError
  - **Verifies:** Handle sparse agent responses
- ✅ **Test:** test_eu_grants_budget_range_with_whitespace
  - **Status:** RED - NotImplementedError
  - **Verifies:** Whitespace ignored in parsing

### Integration Tests (4 tests)

**File:** `services/data-pipeline/tests/integration/test_crawl_eu_grants.py` (98 lines)

- ✅ **Test:** test_crawl_eu_grants_field_completeness
  - **Status:** RED - NotImplementedError
  - **Verifies:** All fields are mapped and DB schema handles JSONB payload
- ✅ **Test:** test_crawl_eu_grants_opportunity_type_grant
  - **Status:** RED - NotImplementedError
  - **Verifies:** Opportunity type mapped and successfully flushed to DB
- ✅ **Test:** test_crawl_eu_grants_crawler_runs_accuracy
  - **Status:** RED - NotImplementedError
  - **Verifies:** Full audit log coverage including status transition and dedup counting
- ✅ **Test:** test_crawl_eu_grants_gateway_down
  - **Status:** RED - NotImplementedError
  - **Verifies:** AIGatewayUnavailableError correctly processed

---

## Fixtures Created

### EU Grants Raw & Normalized Fixtures

**File:** `services/data-pipeline/tests/integration/test_crawl_eu_grants.py`

**Fixtures/Helpers:**
- `_make_eu_grants_raw` - Generates mock raw agent responses for EU grants.
- `_make_eu_grants_normalized` - Generates mock normalized responses for EU grants.
- `eager_celery_no_propagate` - Overrides Celery to run in eager mode without exception propagation.

---

## Mock Requirements

### EU Grant Portal Agent Mock

**Endpoint:** `POST http://ai-gateway:8004/workflows/eu-grant-portal-agent/run`
**Requirements:** Returns list of raw unnormalized items.

### Data Normalization Team Mock

**Endpoint:** `POST http://ai-gateway:8004/workflows/data-normalization-team/run`
**Requirements:** Returns list of structured dict items mapping core column fields.

---

## Implementation Checklist

### Task 1: Add `opportunity_type` to upsert helper

**File:** `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py`

**Tasks to make tests pass:**
- [ ] Add `row.setdefault("opportunity_type", None)` so every row has consistent key
- [ ] Add `"opportunity_type": stmt.excluded.opportunity_type` to `update_cols`

### Task 2: EU Grants normalizer function

**File:** `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py`

**Tasks to make tests pass:**
- [ ] Add `_parse_eu_grants_budget(item)` helper parsing rules
- [ ] Add `normalize_eu_grants_response(raw_output)` logic handling field maps
- [ ] Run test: `pytest services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py`
- [ ] ✅ Unit tests pass (green phase)

### Task 3: Replace `crawl_eu_grants` stub with full implementation

**File:** `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py`

**Tasks to make tests pass:**
- [ ] Implement DB execution and CELERY agent orchestrator loop
- [ ] Add `_on_crawl_eu_grants_failure` signal handler logic
- [ ] Implement structured logging with `structlog`
- [ ] Run test: `pytest services/data-pipeline/tests/integration/test_crawl_eu_grants.py`
- [ ] ✅ Integration tests pass (green phase)

**Estimated Effort:** 4-6 hours

---

## Running Tests

```bash
# Run all failing tests for this story
cd /home/debian/Projects/eusolicit/eusolicit-app && pytest services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py services/data-pipeline/tests/integration/test_crawl_eu_grants.py

# Run only unit tests
cd /home/debian/Projects/eusolicit/eusolicit-app && pytest services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py

# Run only integration tests
cd /home/debian/Projects/eusolicit/eusolicit-app && pytest services/data-pipeline/tests/integration/test_crawl_eu_grants.py
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

**TEA Agent Responsibilities:**
- ✅ All tests written and failing
- ✅ Fixtures and factories created with auto-cleanup
- ✅ Mock requirements documented
- ✅ Implementation checklist created

**Verification:**
- All tests run and fail as expected
- Failure messages are clear and actionable

### GREEN Phase (DEV Team - Next Steps)
1. Pick one failing test from implementation checklist.
2. Read the test to understand expected behavior.
3. Implement minimal code.
4. Check off the task and move to the next test.

---

## Test Execution Evidence

### Initial Test Run (RED Phase Verification)

**Command:** `pytest services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py services/data-pipeline/tests/integration/test_crawl_eu_grants.py`

**Results:**

```
============================= test session starts ==============================
collected 14 items                                                             

services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py FFF [ 21%]
FFFFFFF                                                                  [ 71%]
services/data-pipeline/tests/integration/test_crawl_eu_grants.py FFFF    [100%]

=================================== FAILURES ===================================
...
============================== 14 failed in 2.89s ==============================
```

**Summary:**
- Total tests: 14
- Passing: 0 (expected)
- Failing: 14 (expected)
- Status: ✅ RED phase verified

**Expected Failure Messages:**
- `NotImplementedError`

---

## Next Steps
1. Share this checklist and failing tests with the dev workflow.
2. Run failing tests to confirm RED phase.
3. Begin implementation using checklist as guide.

---

**Generated by BMad TEA Agent** - 2026-04-16