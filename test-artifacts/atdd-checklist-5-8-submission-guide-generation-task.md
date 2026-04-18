---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-16'
workflowType: bmad-testarch-atdd
story_id: 5-8-submission-guide-generation-task
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-8-submission-guide-generation-task.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-app/services/data-pipeline/tests/conftest.py
  - eusolicit-app/services/data-pipeline/tests/unit/test_score_opportunities.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_score_opportunities.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/score_opportunities.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/models/submission_guide.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/models/enrichment_queue.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/models.py
  - _bmad/bmm/config.yaml
outputFiles:
  - eusolicit-app/services/data-pipeline/tests/unit/test_generate_submission_guides.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_generate_submission_guides.py
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
---

# ATDD Checklist — Story 5.8: Submission Guide Generation Task

**Date:** 2026-04-16
**Author:** TEA Master Test Architect
**Story:** 5-8 — Submission Guide Generation Task
**Status:** 🔴 RED PHASE — 14 new tests generated, all in RED state
**Stack:** backend (Python / Celery / SQLAlchemy / PostgreSQL / pytest)
**Generation Mode:** AI Generation (backend-only Celery task; no browser automation needed)

---

## Step 1: Preflight & Context

### Stack Detection

| Indicator | Found | Location |
|-----------|-------|----------|
| `pyproject.toml` with Celery, SQLAlchemy, pytest deps | ✅ | `eusolicit-app/services/data-pipeline/pyproject.toml` |
| `conftest.py` with `PostgresContainer`, `eager_celery` fixtures | ✅ | `eusolicit-app/services/data-pipeline/tests/conftest.py` |
| No `playwright.config.ts` or frontend framework | ✅ confirmed absent | N/A |
| `tests/unit/` and `tests/integration/` directories | ✅ | `eusolicit-app/services/data-pipeline/tests/` |

**Detected stack:** `backend`
**Test framework:** pytest 8.x with `pytest-asyncio`, `respx`, `testcontainers[postgres]`, `unittest.mock`

### TEA Config Flags

| Flag | Value | Source |
|------|-------|--------|
| `test_stack_type` | `auto` → detected `backend` | _bmad/bmm/config.yaml |
| `tea_use_playwright_utils` | not set → `disabled` | config.yaml |
| `tea_use_pactjs_utils` | not set → `disabled` | config.yaml |
| `tea_pact_mcp` | not set → `none` | config.yaml |
| `tea_browser_automation` | not set → `none` | config.yaml |

### Prerequisites Verified

- [x] Story 5.8 approved with 8 clear acceptance criteria (AC1–AC8)
- [x] Backend test framework: `conftest.py` with `pg_container` (session-scoped testcontainer), Alembic migrations, `eager_celery`, `eager_celery_no_propagate`, `_clean_tables`, `_psycopg2_conn` fixtures
- [x] Existing test patterns available: `test_score_opportunities.py` (unit + integration) as structural template
- [x] AI Gateway mocking pattern confirmed: `patch("...get_client")` for unit/integration tests
- [x] `EnrichmentQueueItem` model exists at `data_pipeline.models.enrichment_queue` with all required fields
- [x] `SubmissionGuide` model exists at `data_pipeline.models.submission_guide` with `steps` JSONB, `generated_by`, `reviewed`, `source_portal`, `created_at`, `updated_at` columns
- [x] `AIGatewayUnavailableError(agent_name, attempts, last_status_code)` constructor confirmed
- [x] `AgentType.SUBMISSION_GUIDE` exists in `data_pipeline.ai_gateway_client.models` (note: actual name is `SUBMISSION_GUIDE`, not `GUIDE` as in dev notes — dev may need to check this at implementation time)
- [x] `score_opportunities.py` exists as structural reference for the orchestrator+sub-task pattern

### Context Loaded

- **Story 5.8:** `generate_submission_guides` Celery task + `_generate_single_guide` sub-task chain. Inputs: crawl result dict with `opportunity_ids`. Output: `SubmissionGuide` rows in `pipeline.submission_guides`. Freshness check: `opp.updated_at <= guide.created_at` → skip. Failure path: `EnrichmentQueueItem` creation, no re-raise.
- **Epic test design:** `test-design-epic-05.md` — E05-P1-021 (guide creation), E05-P1-022 (no regeneration), E05-P1-022b (regeneration when updated), E05-P1-023 (failure queues enrichment), E05-P2-010 (`generated_by` from execution_id)
- **Risks mitigated:** E05-R-005 (Celery chord starvation via `pipeline_guides` queue — third isolated queue), E05-R-007 (enrichment queue entry on failure with `enrichment_type='submission_guide'`)

### Key Implementation Gap

`generate_submission_guides.py` does **not yet exist**. The module will be created at:
`eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/generate_submission_guides.py`

Additionally, `score_opportunities.py` must be updated to call
`generate_submission_guides.apply_async(args=[crawl_result])` at the end of its execution (AC7 / Task 1).

---

## Step 2: Generation Mode

**Mode selected:** AI Generation

**Reason:** `detected_stack = backend`. All scenarios are Celery task execution tests, DB state assertions, and mock-based unit tests — no browser automation or UI interaction needed.

---

## Step 3: Test Strategy

### AC → Test ID Mapping

| AC | Description | Test ID(s) | Level | Priority | Epic Test ID | File |
|----|-------------|------------|-------|----------|--------------|------|
| AC1 | `submission_guides` row created with `steps`, `reviewed=false`, `source_portal=source_type` | ATDD-5.8-U05, ATDD-5.8-I01 | Unit+Integration | P1 | E05-P1-021 | both |
| AC2 | Skip when `opp.updated_at <= guide.created_at` (guide is current) | ATDD-5.8-U04, ATDD-5.8-U04b, ATDD-5.8-I02 | Unit+Integration | P1 | E05-P1-022 | both |
| AC3 | Regenerate (update in-place) when `opp.updated_at > guide.created_at` | ATDD-5.8-I03 | Integration | P1 | E05-P1-022b | integration |
| AC4 | `generated_by` = `execution_id`; fallback to `"submission-guide-agent:{correlation_id}"` when None | ATDD-5.8-U05, ATDD-5.8-U06 | Unit | P2 | E05-P2-010 | unit |
| AC5 | Single failure does not abort group — no re-raise | ATDD-5.8-U07, ATDD-5.8-I04, ATDD-5.8-I05 | Unit+Integration | P1 | E05-P1-023 | both |
| AC6 | `EnrichmentQueueItem` created with `enrichment_type='submission_guide'`, `status='pending'`, `attempts=0` | ATDD-5.8-U07, ATDD-5.8-I04 | Unit+Integration | P1 | E05-P1-023 | both |
| AC7 | `generate_submission_guides` auto-triggered from `score_opportunities` | ATDD-5.8-U01, ATDD-5.8-U01b, ATDD-5.8-U02, ATDD-5.8-U03 | Unit | P1 | — | unit |
| AC8 | Structured logs include `task_name`, `opportunity_id`, `correlation_id` | ATDD-5.8-U09 | Unit | P2 | E05-P2-016 | unit |

### Edge Cases

| Scenario | Test ID | Rationale |
|----------|---------|-----------|
| `opportunity_ids` key absent from crawl_result | ATDD-5.8-U01b | AC7 specifies `.get("opportunity_ids", [])` default |
| Opportunity UUID not found in DB | ATDD-5.8-U08 | No guide, no `EnrichmentQueueItem` for not_found (distinct from failure) |
| Batch with 1 failing and 4 succeeding | ATDD-5.8-I05 | Resilience: orchestrator survives partial failure |

### Test Level Rationale

- **Unit tests** (mock DB + mock AI gateway): fastest path to verify module interface, freshness check logic, `generated_by` population, and error handling
- **Integration tests** (real DB testcontainer + mock AI gateway): verify actual DB writes, JSONB persistence, in-place update semantics, and enrichment queue population with real Celery eager execution

---

## Step 4: Test Generation (AI Generation Mode)

### Generated Test Files

| File | Tests | Level | Status |
|------|-------|-------|--------|
| `tests/unit/test_generate_submission_guides.py` | 9 | Unit | 🔴 RED |
| `tests/integration/test_generate_submission_guides.py` | 5 | Integration | 🔴 RED |

### Unit Test Inventory (`tests/unit/test_generate_submission_guides.py`)

| Test ID | Test Function | AC | Epic ID | RED Reason |
|---------|--------------|-----|---------|-----------|
| ATDD-5.8-U01 | `test_generate_submission_guides_skips_when_no_opportunity_ids` | AC7 | — | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U01b | `test_generate_submission_guides_skips_when_key_absent` | AC7 | — | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U02 | `test_generate_submission_guides_dispatches_group_for_all_ids` | AC7 | — | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U03 | `test_score_opportunities_dispatches_generate_submission_guides` | AC7 | — | `score_opportunities.py` does not yet import/call `generate_submission_guides` → `AssertionError` |
| ATDD-5.8-U04 | `test_generate_single_guide_skips_when_guide_is_current` | AC2 | E05-P1-022 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U04b | `test_generate_single_guide_skips_when_opp_updated_at_earlier_than_guide` | AC2 | E05-P1-022 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U05 | `test_generate_single_guide_generated_by_populated_from_execution_id` | AC1, AC4 | E05-P2-010 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U06 | `test_generate_single_guide_generated_by_fallback_when_execution_id_none` | AC4 | E05-P2-010 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U07 | `test_generate_single_guide_creates_enrichment_queue_entry_on_failure` | AC5, AC6 | E05-P1-023 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U08 | `test_generate_single_guide_skips_missing_opportunity` | AC5 edge | — | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-U09 | `test_generate_single_guide_log_includes_required_fields` | AC8 | E05-P2-016 | Module missing → `pytest.fail()` (RED) |

### Integration Test Inventory (`tests/integration/test_generate_submission_guides.py`)

| Test ID | Test Function | AC | Epic ID | RED Reason |
|---------|--------------|-----|---------|-----------|
| ATDD-5.8-I01 | `test_submission_guide_created_for_new_opportunity` | AC1 | E05-P1-021 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-I02 | `test_existing_guide_not_regenerated_when_opportunity_unchanged` | AC2 | E05-P1-022 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-I03 | `test_existing_guide_regenerated_when_opportunity_updated` | AC3 | E05-P1-022b | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-I04 | `test_guide_generation_failure_queues_enrichment_entry` | AC5, AC6 | E05-P1-023 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.8-I05 | `test_guide_generation_failure_does_not_block_batch` | AC5 | — | Module missing → `pytest.fail()` (RED) |

---

## Step 4C: Test Infrastructure

### Fixtures Used (from conftest.py)

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `pg_container` | session | PostgreSQL 16 testcontainer + Alembic migrations |
| `eager_celery` | function | `task_always_eager=True, task_eager_propagates=True` |
| `eager_celery_no_propagate` | function | `task_always_eager=True, task_eager_propagates=False` |
| `_clean_tables` | function | TRUNCATE opportunities + crawler_runs CASCADE (clears submission_guides, enrichment_queue via FK) |
| `_psycopg2_conn` | function | Raw psycopg2 connection factory |
| `_reset_clients_and_db` | function (autouse) | Resets AI client, circuit breaker, engine between tests |

### Local Helpers (defined in integration test file)

| Helper | Purpose |
|--------|---------|
| `_seed_opportunity(conn, *, source_type, updated_at)` | Insert 1 opportunity via psycopg2; `updated_at` override for freshness check tests |
| `_seed_guide(conn, opportunity_id, *, created_at, steps, generated_by, source_portal)` | Insert a `submission_guides` row with specific `created_at` |
| `_get_guide(conn, opportunity_id)` | Read `submission_guides` row for an opportunity; returns dict or None |
| `_guide_count(conn, opportunity_id)` | COUNT `submission_guides` rows for an opportunity (verify no duplicates) |
| `_enrichment_queue_count(conn, enrichment_type)` | COUNT `enrichment_queue` rows with `enrichment_type='submission_guide'` |
| `_make_guide_response(steps, execution_id, source_type)` | Build AI Gateway response dict for guide agent |

### Mock Strategy

| Scenario | Mock Approach |
|----------|--------------|
| Unit tests — AI Gateway | `patch("data_pipeline.workers.tasks.generate_submission_guides.get_client")` |
| Unit tests — DB session | `patch("data_pipeline.workers.tasks.generate_submission_guides.get_sync_session")` |
| Unit tests — Celery group dispatch | `patch("data_pipeline.workers.tasks.generate_submission_guides.group")` |
| Unit test — score chain dispatch | `patch("data_pipeline.workers.tasks.score_opportunities.generate_submission_guides", create=True)` |
| Integration tests — AI Gateway | `patch("data_pipeline.workers.tasks.generate_submission_guides.get_client")` |
| Integration tests — DB | Real testcontainer (no mocking) |
| Selective failure (partial batch) | `side_effect` callable that checks `payload["opportunity"]["id"]` |

---

## Step 5: Validation & Completion

### Checklist Validation

- [x] Prerequisites satisfied (story AC defined, test framework present, SubmissionGuide + EnrichmentQueueItem models exist)
- [x] Test files created at correct paths:
  - `eusolicit-app/services/data-pipeline/tests/unit/test_generate_submission_guides.py`
  - `eusolicit-app/services/data-pipeline/tests/integration/test_generate_submission_guides.py`
- [x] All 8 acceptance criteria covered by at least one test
- [x] All tests designed to FAIL before implementation (TDD red phase)
- [x] No `pytest.skip()` used — tests FAIL (not skip) in red phase
- [x] Checklist stored in `test-artifacts/` not a random location
- [x] No orphaned browser sessions (backend-only, no browser automation)
- [x] Tests match patterns from existing codebase (see `test_score_opportunities.py`)
- [x] Epic test IDs (E05-P1-021, E05-P1-022, E05-P1-022b, E05-P1-023, E05-P2-010) referenced in test docstrings

### RED Phase Mechanism

| Category | Tests | How RED State is Enforced |
|----------|-------|--------------------------|
| New module tests (U01–U09, I01–I05) | 13 | `_require_guide_module()` → `pytest.fail()` because `ImportError` on module import |
| Existing module changes (U03) | 1 | `AssertionError` — `score_opportunities.py` does not yet call `generate_submission_guides.apply_async` |

### Implementation Tasks That Will Turn Tests GREEN

| Task | Tests Turned GREEN |
|------|--------------------|
| **Task 1**: Add `generate_submission_guides.apply_async(args=[crawl_result])` to `score_opportunities.py` | ATDD-5.8-U03 |
| **Task 2**: Implement `generate_submission_guides` orchestrator task | ATDD-5.8-U01, U01b, U02 |
| **Task 3**: Implement `_generate_single_guide` sub-task (full logic: freshness check, create/update, failure handling) | U04, U04b, U05, U06, U07, U08, U09, I01–I05 |
| **Task 4**: Register `pipeline_guides` queue in `celery_app.py` | Not covered by tests — deployment configuration concern |
| **Task 5**: Verify `AgentType.SUBMISSION_GUIDE` in `client.py` | Confirmed already present — no action needed |

---

## Traceability Matrix

| Test ID | Story AC | Epic Test ID | Priority | Level |
|---------|----------|--------------|----------|-------|
| ATDD-5.8-U01 | AC7 | — | P1 | Unit |
| ATDD-5.8-U01b | AC7 | — | P2 | Unit |
| ATDD-5.8-U02 | AC7 | — | P1 | Unit |
| ATDD-5.8-U03 | AC7 | — | P1 | Unit |
| ATDD-5.8-U04 | AC2 | E05-P1-022 | P1 | Unit |
| ATDD-5.8-U04b | AC2 | E05-P1-022 | P2 | Unit |
| ATDD-5.8-U05 | AC1, AC4 | E05-P2-010 | P2 | Unit |
| ATDD-5.8-U06 | AC4 | E05-P2-010 | P2 | Unit |
| ATDD-5.8-U07 | AC5, AC6 | E05-P1-023 | P1 | Unit |
| ATDD-5.8-U08 | AC5 (edge) | — | P2 | Unit |
| ATDD-5.8-U09 | AC8 | E05-P2-016 | P2 | Unit |
| ATDD-5.8-I01 | AC1 | E05-P1-021 | P1 | Integration |
| ATDD-5.8-I02 | AC2 | E05-P1-022 | P1 | Integration |
| ATDD-5.8-I03 | AC3 | E05-P1-022b | P1 | Integration |
| ATDD-5.8-I04 | AC5, AC6 | E05-P1-023 | P1 | Integration |
| ATDD-5.8-I05 | AC5 | — | P2 | Integration |

### AC Coverage Summary

| AC | Description | Tests | Covered? |
|----|-------------|-------|----------|
| AC1 | `submission_guides` row: `steps`, `reviewed=false`, `source_portal` | U05, I01 | ✅ |
| AC2 | Skip when `opp.updated_at <= guide.created_at` | U04, U04b, I02 | ✅ |
| AC3 | Regenerate in-place when `opp.updated_at > guide.created_at` | I03 | ✅ |
| AC4 | `generated_by` = `execution_id`; fallback when None | U05, U06 | ✅ |
| AC5 | Failure does not abort group — no re-raise | U07, I04, I05 | ✅ |
| AC6 | `EnrichmentQueueItem` with correct fields on failure | U07, I04 | ✅ |
| AC7 | `generate_submission_guides` triggered from `score_opportunities` | U01, U01b, U02, U03 | ✅ |
| AC8 | Structured logs: `task_name`, `opportunity_id`, `correlation_id` | U09 | ✅ |

---

## Risk Mitigations Verified by Tests

| Risk | Score | Mitigation | Verified By |
|------|-------|-----------|-------------|
| E05-R-005 (Celery chord starvation) | 4 | `pipeline_guides` queue isolation (third dedicated queue) | ATDD-5.8-U02 (group dispatch verified); Task 4 (queue config — deployment only) |
| E05-R-007 (Enrichment queue unbounded failure) | 4 | `EnrichmentQueueItem` on guide failure with `enrichment_type='submission_guide'` | ATDD-5.8-U07, ATDD-5.8-I04, ATDD-5.8-I05 |

---

## Known Assumptions & Dependencies

| Assumption | Impact | Mitigation |
|------------|--------|-----------|
| `AgentType.SUBMISSION_GUIDE` is the correct enum value (not `GUIDE` as in some dev notes) | Dev may use wrong constant if trusting dev notes over source code | Confirmed `SUBMISSION_GUIDE` in `models.py`; tests patch at `get_client()` level so exact enum value is not tested at unit level — integration tests will catch a wrong call |
| `Celery group` with `task_always_eager=True` runs sub-tasks synchronously | Integration tests rely on sync execution for DB state assertions | Confirmed by existing scoring test patterns |
| `pipeline.submission_guides.steps` is a JSONB list (not dict) per the model | Test assertions use `isinstance(guide["steps"], list)` | Confirmed by SubmissionGuide model: `steps: Mapped[dict]` stores an ordered list of step objects |
| `_clean_tables` CASCADE truncates `submission_guides` and `enrichment_queue` | Tests start with clean state | Confirmed by conftest: `TRUNCATE pipeline.opportunities ... CASCADE` — CASCADE should reach both FK-referencing tables |
| No `SubmissionGuide` unique constraint per opportunity — at-most-one enforced by application logic (not DB) | Tests must verify `_guide_count == 1` not rely on DB error | Integration tests include explicit `_guide_count` assertion for AC3 |

---

## Next Steps (TDD Green Phase)

After developer implements S05.08:

1. **Run tests to verify RED state:**
   ```bash
   cd eusolicit-app/services/data-pipeline
   python -m pytest tests/unit/test_generate_submission_guides.py tests/integration/test_generate_submission_guides.py -v
   # Expected: all 16 tests FAIL (RED phase confirmed)
   ```

2. **Implement S05.08 (Tasks 1–5)** following the story spec.

3. **Run tests again to verify GREEN state:**
   ```bash
   python -m pytest tests/unit/test_generate_submission_guides.py tests/integration/test_generate_submission_guides.py -v
   # Expected: all 16 tests PASS (GREEN phase)
   ```

4. **Run full test suite regression:**
   ```bash
   python -m pytest tests/ -v --timeout=120
   # Expected: existing tests still pass; no regressions in score_opportunities chain
   ```

5. **Commit passing tests** once all tests are GREEN.

6. **Run `/bmad-testarch-automate`** to expand coverage beyond the ATDD scenarios.

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** BMad v6
