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
story_id: 5-7-relevance-scoring-post-processing-task
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-7-relevance-scoring-post-processing-task.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-app/services/data-pipeline/tests/conftest.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_crawl_eu_grants.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/models/enrichment_queue.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/models/opportunity.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/exceptions.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/models.py
  - _bmad/bmm/config.yaml
outputFiles:
  - eusolicit-app/services/data-pipeline/tests/unit/test_score_opportunities.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_score_opportunities.py
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
---

# ATDD Checklist — Story 5.7: Relevance Scoring Post-Processing Task

**Date:** 2026-04-16
**Author:** TEA Master Test Architect
**Story:** 5-7 — Relevance Scoring Post-Processing Task
**Status:** 🔴 RED PHASE — 16 new tests generated, all in RED state
**Stack:** backend (Python / Celery / SQLAlchemy / PostgreSQL / pytest)
**Generation Mode:** AI Generation (no recording needed — backend-only Celery task)

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

- [x] Story 5.7 approved with 8 clear acceptance criteria (AC1–AC8)
- [x] Backend test framework: `conftest.py` with `pg_container` (session-scoped testcontainer), Alembic migrations, `eager_celery`, `eager_celery_no_propagate`, `_clean_tables`, `_psycopg2_conn`, `_latest_run`, `_opp_count` fixtures
- [x] Existing test patterns available: `test_crawl_eu_grants.py`, `test_crawl_aop.py` as structural templates
- [x] AI Gateway mocking pattern confirmed: `patch("...get_client")` for unit tests, `respx` for integration
- [x] `EnrichmentQueueItem` model exists at `data_pipeline.models.enrichment_queue` with all required fields
- [x] `Opportunity.relevance_scores` JSONB column exists in model
- [x] `AIGatewayUnavailableError(agent_name, attempts, last_status_code)` constructor confirmed
- [x] `AgentType.SCORING` exists in `data_pipeline.ai_gateway_client.models`

### Context Loaded

- **Story 5.7:** `score_opportunities` Celery task + `_score_single_opportunity` sub-task chain. Inputs: crawl result dict with `opportunity_ids`. Output: `relevance_scores` JSONB per opportunity. Failure path: `EnrichmentQueueItem` creation, no re-raise.
- **Epic test design:** `test-design-epic-05.md` — E05-P1-018 (parallelism), E05-P1-019 (failure doesn't block), E05-P1-020 (failure queued), E05-P2-008 (JSONB structure), E05-P2-009 (partial failure 3-of-5)
- **Risks mitigated:** E05-R-005 (Celery chord starvation via pipeline_scoring queue), E05-R-007 (enrichment queue entry creation on failure)

### Key Implementation Gap

`score_opportunities.py` does **not yet exist**. The module will be created at:
`eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/score_opportunities.py`

Additionally, `_upsert.py` returns `tuple[int, int]` but must return `tuple[int, int, list[str]]` (AC6), and all three crawl tasks must dispatch `score_opportunities.apply_async` (AC5).

---

## Step 2: Generation Mode

**Mode selected:** AI Generation

**Reason:** `detected_stack = backend`. All scenarios are Celery task execution tests, DB state assertions, and mock-based unit tests — no browser automation or UI interaction needed.

---

## Step 3: Test Strategy

### AC → Test ID Mapping

| AC | Description | Test ID(s) | Level | Priority | Epic Test ID | File |
|----|-------------|------------|-------|----------|--------------|------|
| AC1 | `relevance_scores` JSONB = `{company_id: score}` for all active companies | ATDD-5.7-I01, ATDD-5.7-U08 | Integration / Unit | P1 | E05-P1-018, E05-P2-008 | both |
| AC2 | Celery group to `pipeline_scoring` queue; `SCORING_CONCURRENCY` controls concurrency | ATDD-5.7-U06, ATDD-5.7-U07, ATDD-5.7-I01, ATDD-5.7-I05 | Unit / Integration | P1/P2 | E05-P1-018 | both |
| AC3 | Single-opportunity failure does not abort the batch — no re-raise | ATDD-5.7-U09, ATDD-5.7-I02, ATDD-5.7-I03 | Unit / Integration | P1 | E05-P1-019 | both |
| AC4 | `EnrichmentQueueItem` created with correct fields on failure | ATDD-5.7-U09, ATDD-5.7-I03, ATDD-5.7-I04 | Unit / Integration | P1 | E05-P1-020, E05-P2-009 | both |
| AC5 | `score_opportunities.apply_async` called at end of each crawler task | ATDD-5.7-U03, ATDD-5.7-U04, ATDD-5.7-U05 | Unit | P1 | E05-P1-018 | unit |
| AC6 | Crawl result dict includes `opportunity_ids: list[str]` | ATDD-5.7-U01, ATDD-5.7-U01b, ATDD-5.7-U02 | Unit | P1 | — | unit |
| AC7 | Integration test: 5 opps, 2 failures → 3 scores + 2 enrichment entries | ATDD-5.7-I04 | Integration | P2 | E05-P2-009 | integration |
| AC8 | Structured log lines include `task_name`, `opportunity_id`, `correlation_id` | ATDD-5.7-U11 | Unit | P2 | E05-P2-016 | unit |

### Edge Cases

| Scenario | Test ID | Rationale |
|----------|---------|-----------|
| `opportunity_ids` key absent from crawl_result | ATDD-5.7-U06b | AC2 specifies `.get("opportunity_ids", [])` default |
| Opportunity UUID not found in DB | ATDD-5.7-U10 | AC4 says EnrichmentQueueItem only created on AI Gateway failure, not on not_found |
| `dispatched` count in return dict | ATDD-5.7-I05 | AC2 return contract verification |

### Test Level Rationale

- **Unit tests** (mock DB + mock AI gateway): fastest path to verify module interface, JSONB structure, error handling, and chain dispatch logic
- **Integration tests** (real DB testcontainer + mock AI gateway): verify actual DB writes, JSONB persistence, enrichment queue population, and no-exception guarantees with real Celery eager execution

---

## Step 4: Test Generation (AI Generation Mode)

### Generated Test Files

| File | Tests | Level | Status |
|------|-------|-------|--------|
| `tests/unit/test_score_opportunities.py` | 11 | Unit | 🔴 RED |
| `tests/integration/test_score_opportunities.py` | 5 | Integration | 🔴 RED |

### Unit Test Inventory (`tests/unit/test_score_opportunities.py`)

| Test ID | Test Function | AC | Epic ID | RED Reason |
|---------|--------------|-----|---------|-----------|
| ATDD-5.7-U01 | `test_upsert_returns_opportunity_ids` | AC6 | — | `upsert_opportunities` returns 2-tuple; unpacking 3-tuple fails with `ValueError` |
| ATDD-5.7-U01b | `test_upsert_returns_ids_for_all_upserted_rows` | AC6 | — | Same as above |
| ATDD-5.7-U02 | `test_crawl_aop_result_includes_opportunity_ids` | AC6 | — | `crawl_aop` return dict has no `opportunity_ids` key → `AssertionError` |
| ATDD-5.7-U03 | `test_crawl_aop_dispatches_score_opportunities` | AC5 | — | `crawl_aop.py` does not import or call `score_opportunities.apply_async` → `AssertionError` |
| ATDD-5.7-U04 | `test_crawl_ted_dispatches_score_opportunities` | AC5 | — | Same as above for `crawl_ted.py` |
| ATDD-5.7-U05 | `test_crawl_eu_grants_dispatches_score_opportunities` | AC5 | — | Same as above for `crawl_eu_grants.py` |
| ATDD-5.7-U06 | `test_score_opportunities_skips_when_no_opportunity_ids` | AC2 | E05-P1-018 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-U06b | `test_score_opportunities_skips_when_key_absent` | AC2 | — | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-U07 | `test_score_opportunities_dispatches_group_for_all_ids` | AC2 | E05-P1-018 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-U08 | `test_score_single_opportunity_relevance_scores_keyed_by_company_id` | AC1 | E05-P2-008 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-U09 | `test_score_single_opportunity_creates_enrichment_entry_on_failure` | AC3, AC4 | E05-P1-020 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-U10 | `test_score_single_opportunity_handles_missing_opportunity` | AC3 edge | — | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-U11 | `test_score_single_opportunity_log_includes_required_fields` | AC8 | E05-P2-016 | Module missing → `pytest.fail()` (RED) |

### Integration Test Inventory (`tests/integration/test_score_opportunities.py`)

| Test ID | Test Function | AC | Epic ID | RED Reason |
|---------|--------------|-----|---------|-----------|
| ATDD-5.7-I01 | `test_score_opportunities_parallelism` | AC1, AC2 | E05-P1-018 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-I02 | `test_single_opportunity_failure_does_not_block_batch` | AC3 | E05-P1-019 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-I03 | `test_failed_scoring_queued_in_enrichment_queue` | AC4 | E05-P1-020 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-I04 | `test_partial_failure_batch_3_of_5` | AC7 | E05-P2-009 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.7-I05 | `test_score_opportunities_dispatched_count_matches_input` | AC2 | E05-P1-018 | Module missing → `pytest.fail()` (RED) |

---

## Step 4C: Test Infrastructure

### Fixtures Used (from conftest.py)

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `pg_container` | session | PostgreSQL 16 testcontainer + Alembic migrations |
| `eager_celery` | function | `task_always_eager=True, task_eager_propagates=True` |
| `eager_celery_no_propagate` | function | `task_always_eager=True, task_eager_propagates=False` |
| `_clean_tables` | function | TRUNCATE opportunities + crawler_runs CASCADE (also clears enrichment_queue via FK) |
| `_psycopg2_conn` | function | Raw psycopg2 connection factory |
| `_make_eu_grants_raw` / `_make_eu_grants_normalized` | function | EU Grants mock data generators |
| `_make_raw_opps` / `_make_normalized_opps` | function | AOP mock data generators |
| `_reset_clients_and_db` | function (autouse) | Resets AI client, circuit breaker, engine between tests |

### Local Helpers (defined in integration test file)

| Helper | Purpose |
|--------|---------|
| `_seed_opportunities(conn, n)` | Insert n active opportunities via psycopg2; return list of UUIDs |
| `_seed_company_profiles(conn, n)` | Insert n active company profiles into auth.company_profiles; return list of UUIDs |
| `_ensure_company_profiles_table(conn)` | CREATE TABLE IF NOT EXISTS auth.company_profiles (minimal schema) |
| `_get_relevance_scores(conn, opp_id)` | Read relevance_scores JSONB for one opportunity; returns None if NULL |
| `_enrichment_queue_count(conn, enrichment_type)` | COUNT enrichment_queue rows by type |
| `_clean_company_profiles(conn)` | TRUNCATE auth.company_profiles between tests |

### Mock Strategy

| Scenario | Mock Approach |
|----------|--------------|
| Unit tests — AI Gateway | `patch("data_pipeline.workers.tasks.score_opportunities.get_client")` |
| Unit tests — DB session | `patch("data_pipeline.workers.tasks.score_opportunities.get_sync_session")` |
| Unit tests — Celery group dispatch | `patch("data_pipeline.workers.tasks.score_opportunities.group")` |
| Integration tests — AI Gateway | `patch("data_pipeline.workers.tasks.score_opportunities.get_client")` |
| Integration tests — DB | Real testcontainer (no mocking) |
| Selective failure (partial batch) | `side_effect` callable that checks `payload["opportunity"]["id"]` |
| Crawl task dispatch tests | `patch("data_pipeline.workers.tasks.crawl_{x}.score_opportunities", create=True)` |

---

## Step 5: Validation & Completion

### Checklist Validation

- [x] Prerequisites satisfied (story AC defined, test framework present)
- [x] Test files created at correct paths:
  - `eusolicit-app/services/data-pipeline/tests/unit/test_score_opportunities.py`
  - `eusolicit-app/services/data-pipeline/tests/integration/test_score_opportunities.py`
- [x] All 8 acceptance criteria covered by at least one test
- [x] All tests designed to FAIL before implementation (TDD red phase)
- [x] No `pytest.skip()` used — tests FAIL (not skip) in red phase
- [x] Temp artifacts stored in `test-artifacts/` not random locations
- [x] No orphaned browser sessions (backend-only, no browser automation)
- [x] Tests match patterns from existing codebase (see `test_crawl_eu_grants.py`)
- [x] Epic test IDs (E05-P1-018, E05-P1-019, E05-P1-020, E05-P2-008, E05-P2-009) referenced in test docstrings

### RED Phase Mechanism

| Category | Tests | How RED State is Enforced |
|----------|-------|--------------------------|
| New module tests (U06–U11, I01–I05) | 11 | `_require_score_module()` → `pytest.fail()` because `ImportError` on module import |
| Existing module changes (U01–U05) | 5 | AssertionError — modules exist but return wrong data structure / missing behaviour |

### Implementation Tasks That Will Turn Tests GREEN

When developer implements the story tasks, the following tests will turn GREEN:

| Task | Tests Turned GREEN |
|------|--------------------|
| **Task 1**: Extend `upsert_opportunities` to return 3-tuple | ATDD-5.7-U01, ATDD-5.7-U01b |
| **Task 1.2**: Update crawl task callers + add `opportunity_ids` to return dict | ATDD-5.7-U02 |
| **Task 2**: Chain `score_opportunities.apply_async` in all 3 crawlers | ATDD-5.7-U03, ATDD-5.7-U04, ATDD-5.7-U05 |
| **Task 3**: Implement `score_opportunities` orchestrator task | ATDD-5.7-U06, ATDD-5.7-U06b, ATDD-5.7-U07 |
| **Task 4**: Implement `_score_single_opportunity` sub-task | ATDD-5.7-U08, ATDD-5.7-U09, ATDD-5.7-U10, ATDD-5.7-U11, I01–I05 |
| **Task 5**: Register `pipeline_scoring` queue (deployment config) | Not covered by tests — deployment concern |

---

## Traceability Matrix

| Test ID | Story AC | Epic Test ID | Priority | Level |
|---------|----------|--------------|----------|-------|
| ATDD-5.7-U01 | AC6 | — | P1 | Unit |
| ATDD-5.7-U01b | AC6 | — | P1 | Unit |
| ATDD-5.7-U02 | AC6 | — | P1 | Unit |
| ATDD-5.7-U03 | AC5 | — | P1 | Unit |
| ATDD-5.7-U04 | AC5 | — | P1 | Unit |
| ATDD-5.7-U05 | AC5 | — | P1 | Unit |
| ATDD-5.7-U06 | AC2 | E05-P1-018 | P1 | Unit |
| ATDD-5.7-U06b | AC2 | — | P2 | Unit |
| ATDD-5.7-U07 | AC2 | E05-P1-018 | P1 | Unit |
| ATDD-5.7-U08 | AC1 | E05-P2-008 | P2 | Unit |
| ATDD-5.7-U09 | AC3, AC4 | E05-P1-020 | P1 | Unit |
| ATDD-5.7-U10 | AC3 (edge) | — | P2 | Unit |
| ATDD-5.7-U11 | AC8 | E05-P2-016 | P2 | Unit |
| ATDD-5.7-I01 | AC1, AC2 | E05-P1-018 | P1 | Integration |
| ATDD-5.7-I02 | AC3 | E05-P1-019 | P1 | Integration |
| ATDD-5.7-I03 | AC4 | E05-P1-020 | P1 | Integration |
| ATDD-5.7-I04 | AC7 | E05-P2-009 | P2 | Integration |
| ATDD-5.7-I05 | AC2 | E05-P1-018 | P1 | Integration |

### AC Coverage Summary

| AC | Description | Tests | Covered? |
|----|-------------|-------|----------|
| AC1 | `relevance_scores` JSONB `{company_id: score}` for all active companies | U08, I01, I04 | ✅ |
| AC2 | Celery group to `pipeline_scoring`; `SCORING_CONCURRENCY` env var | U06, U06b, U07, I01, I05 | ✅ |
| AC3 | Single failure doesn't abort batch — no re-raise | U09, I02, I03 | ✅ |
| AC4 | `EnrichmentQueueItem` created with correct fields | U09, I03, I04 | ✅ |
| AC5 | `score_opportunities.apply_async` after each crawler | U03, U04, U05 | ✅ |
| AC6 | Crawl result includes `opportunity_ids` | U01, U01b, U02 | ✅ |
| AC7 | Integration test: 5 opps, 2 fail → 3 scores + 2 queue entries | I04 | ✅ |
| AC8 | Structured logs include `task_name`, `opportunity_id`, `correlation_id` | U11 | ✅ |

---

## Risk Mitigations Verified by Tests

| Risk | Score | Mitigation | Verified By |
|------|-------|-----------|-------------|
| E05-R-005 (Celery chord starvation) | 4 | `pipeline_scoring` queue isolation | ATDD-5.7-U07 (group dispatch verified); Task 5 (queue config — deployment only) |
| E05-R-007 (Enrichment queue unbounded failure) | 4 | `EnrichmentQueueItem` on scoring failure | ATDD-5.7-U09, ATDD-5.7-I03, ATDD-5.7-I02 |

---

## Known Assumptions & Dependencies

| Assumption | Impact | Mitigation |
|------------|--------|-----------|
| `auth.company_profiles` table exists after `alembic upgrade head` | Integration tests need it for seeding | `_ensure_company_profiles_table()` creates minimal schema if absent |
| `auth.company_profiles` schema has at minimum `(id UUID, is_active bool, profile_data JSONB)` | Seeding would fail if schema differs significantly | Based on dev notes S05.07; `CREATE TABLE IF NOT EXISTS` is a no-op if full E02 table exists |
| `Celery group` with `task_always_eager=True` runs sub-tasks synchronously | Integration tests rely on sync execution for DB state assertions | Confirmed by existing test patterns (`test_crawl_aop.py`) |
| `score_opportunities` uses `get_sync_session()` for DB access (not async) | Mock pattern requires patching synchronous session | Consistent with all existing Celery tasks in this service |
| `CompanyProfile.to_scoring_dict()` returns `{"id": str(id), "profile": {...}}` | Mock companies implement this method | Dev notes specify exact shape |

---

## Next Steps (TDD Green Phase)

After developer implements S05.07:

1. **Run tests to verify RED state:**
   ```bash
   cd eusolicit-app/services/data-pipeline
   python -m pytest tests/unit/test_score_opportunities.py tests/integration/test_score_opportunities.py -v
   # Expected: all 18 tests FAIL (RED phase confirmed)
   ```

2. **Implement S05.07 (Tasks 1–5)** following the story spec.

3. **Run tests again to verify GREEN state:**
   ```bash
   python -m pytest tests/unit/test_score_opportunities.py tests/integration/test_score_opportunities.py -v
   # Expected: all 18 tests PASS (GREEN phase)
   ```

4. **Run full test suite regression:**
   ```bash
   python -m pytest tests/ -v --timeout=120
   # Expected: existing tests still pass; no regressions in crawl tasks
   ```

5. **Commit passing tests** once all tests are GREEN.

6. **Run `/bmad-testarch-automate`** to expand coverage beyond the ATDD scenarios.

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** BMad v6
