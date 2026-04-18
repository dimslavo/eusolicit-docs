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
story_id: 5-9-redis-streams-event-publishing
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-9-redis-streams-event-publishing.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-app/services/data-pipeline/tests/conftest.py
  - eusolicit-app/services/data-pipeline/tests/unit/test_generate_submission_guides.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_generate_submission_guides.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/generate_submission_guides.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/models/crawler_run.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/celery_app.py
  - _bmad/bmm/config.yaml
outputFiles:
  - eusolicit-app/services/data-pipeline/tests/unit/test_publish_event.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_publish_event.py
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
---

# ATDD Checklist — Story 5.9: Redis Streams Event Publishing

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**Story:** 5-9 — Redis Streams Event Publishing
**Status:** 🔴 RED PHASE — 11 new tests generated, all in RED state
**Stack:** backend (Python / Celery / SQLAlchemy / PostgreSQL / Redis / pytest)
**Generation Mode:** AI Generation (backend-only Celery task; no browser automation needed)

---

## Step 1: Preflight & Context

### Stack Detection

| Indicator | Found | Location |
|-----------|-------|----------|
| `pyproject.toml` with Celery, SQLAlchemy, redis, pytest deps | ✅ | `eusolicit-app/services/data-pipeline/pyproject.toml` |
| `conftest.py` with `PostgresContainer`, `eager_celery` fixtures | ✅ | `eusolicit-app/services/data-pipeline/tests/conftest.py` |
| No `playwright.config.ts` or frontend framework | ✅ confirmed absent | N/A |
| `tests/unit/` and `tests/integration/` directories | ✅ | `eusolicit-app/services/data-pipeline/tests/` |

**Detected stack:** `backend`
**Test framework:** pytest 8.x with `pytest-asyncio`, `testcontainers[postgres]`, `unittest.mock`, sync `redis`

### TEA Config Flags

| Flag | Value | Source |
|------|-------|--------|
| `test_stack_type` | `auto` → detected `backend` | _bmad/bmm/config.yaml |
| `tea_use_playwright_utils` | not set → `disabled` | config.yaml |
| `tea_use_pactjs_utils` | not set → `disabled` | config.yaml |
| `tea_pact_mcp` | not set → `none` | config.yaml |
| `tea_browser_automation` | not set → `none` | config.yaml |

### Prerequisites Verified

- [x] Story 5.9 approved with 8 clear acceptance criteria (AC1–AC8)
- [x] Backend test framework: `conftest.py` with `pg_container`, `eager_celery`, `eager_celery_no_propagate`, `_psycopg2_conn`, `_clean_tables`, `_reset_clients_and_db` fixtures
- [x] Existing test patterns available: `test_generate_submission_guides.py` (unit + integration) as structural template
- [x] `CrawlerRun` model confirmed: `errors` Mapped[dict] JSONB, `status` String, `found`, `new_count` ("new" col), `updated` Integer columns
- [x] `generate_submission_guides.py` exists but does NOT yet call `publish_ingested_event.apply_async` (correct RED state for AC7 / Task 2)
- [x] Celery conftest uses `redis://localhost:6379/1` as `TEST_REDIS_URL` — same instance used for stream integration tests
- [x] E01 envelope format confirmed from `eusolicit-common/events/publisher.py` reference in story dev notes

### Key Implementation Gap

`publish_event.py` module does **not yet exist**. The module will be created at:
`eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/publish_event.py`

Additionally, `generate_submission_guides.py` must be updated to call
`publish_ingested_event.apply_async(args=[crawl_result])` after `guide_group.apply_async()`
(AC7 / Task 2).

### Context Loaded

- **Story 5.9:** `publish_ingested_event` Celery task using sync `redis.Redis` client (not asyncio). Publishes E01 envelope to `eu-solicit:opportunities` Redis Stream. Celery isolated retry (max 3, exponential back-off). `_record_publish_failure` helper writes to `crawler_runs.errors` JSONB on exhausted retries without altering `crawler_runs.status`.
- **Epic test design:** `test-design-epic-05.md` — E05-P0-007 (Redis publish happy path), E05-P0-008 (isolated publish retry), E05-P1-024 (summary sum cross-validation), E05-R-003 (Redis publish-or-lose risk mitigation)
- **Risks mitigated:** E05-R-003 (Redis Streams publish loss, Score 6): tests E05-P0-007 and E05-P0-008 are both P0 and must pass before sprint close.

---

## Step 2: Generation Mode

**Mode selected:** AI Generation

**Reason:** `detected_stack = backend`. All scenarios are Celery task execution tests, Redis stream assertions, DB state assertions, and mock-based unit tests — no browser automation or UI interaction needed.

---

## Step 3: Test Strategy

### AC → Test ID Mapping

| AC | Description | Test ID(s) | Level | Priority | Epic Test ID | File |
|----|-------------|------------|-------|----------|--------------|------|
| AC1 | E01 envelope: event_id, event_type="OpportunitiesIngested", payload (JSON), timestamp, correlation_id=run_id, source_service="data-pipeline", tenant_id="" | ATDD-5.9-U01, ATDD-5.9-I01 | Unit+Integration | P0 | E05-P0-007 | both |
| AC2 | Payload: crawler_type, run_id, opportunity_ids (list), timestamp (ISO), summary{new,updated,unchanged} | ATDD-5.9-U02, ATDD-5.9-I01 | Unit+Integration | P0 | E05-P0-007 | both |
| AC3 | Redis client reads event and deserializes opportunity_ids back to Python list | ATDD-5.9-I01 | Integration | P0 | E05-P0-007 | integration |
| AC4 | Publish failure → isolated Celery retry; crawler_runs stays "completed"; no crawl re-run | ATDD-5.9-U06, ATDD-5.9-I02 | Unit+Integration | P0 | E05-P0-008 | both |
| AC5 | Retries exhausted → ERROR log + crawler_runs.errors JSONB update via _record_publish_failure | ATDD-5.9-U05 (log), ATDD-5.9-U06 (record call), ATDD-5.9-I04 | Unit+Integration | P0 | E05-R-003 | both |
| AC6 | summary.new + updated + unchanged == crawler_runs.found | ATDD-5.9-U03, ATDD-5.9-U04, ATDD-5.9-I03 | Unit+Integration | P1 | E05-P1-024 | both |
| AC7 | publish_ingested_event auto-dispatched from generate_submission_guides (success path only) | ATDD-5.9-U07 | Unit | P1 | — | unit |
| AC8 | All log lines include task_name, run_id, correlation_id | ATDD-5.9-U05 | Unit | P2 | E05-P2-016 | unit |

### Edge Cases

| Scenario | Test ID | Rationale |
|----------|---------|-----------|
| unchanged computation with various found/new/updated combos | ATDD-5.9-U04 (parametrized) | AC6 — ensures no arithmetic error in unchanged = max(0, found-new-updated) |
| stream cleanup between tests | `_clean_opportunities_stream` fixture | Cross-test stream pollution prevention |
| _record_publish_failure called directly (unit-level AC5 path) | ATDD-5.9-I04 | Tests DB write logic independently of retry plumbing |

### Test Level Rationale

- **Unit tests** (mock Redis + mock DB): fastest path to verify E01 envelope construction, payload field presence, unchanged arithmetic, log field binding, and chain dispatch assertion
- **Integration tests** (real DB testcontainer + real Redis for stream tests): verify actual XADD call produces readable stream message, `crawler_runs.errors` JSONB update with real PostgreSQL JSONB merge, and `crawler_runs.status` isolation guarantee

---

## Step 4: Test Generation

### Generated Test Files

| File | Tests | Level | Status |
|------|-------|-------|--------|
| `tests/unit/test_publish_event.py` | 7 | Unit | 🔴 RED |
| `tests/integration/test_publish_event.py` | 4 | Integration | 🔴 RED |

### Unit Test Inventory (`tests/unit/test_publish_event.py`)

| Test ID | Test Function | AC | Epic ID | RED Reason |
|---------|--------------|-----|---------|-----------|
| ATDD-5.9-U01 | `test_publish_ingested_event_happy_path` | AC1 | E05-P0-007 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-U02 | `test_publish_ingested_event_payload_fields` | AC2 | E05-P0-007 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-U03 | `test_publish_summary_unchanged_calculated` | AC6 | E05-P1-024 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-U04 | `test_publish_summary_matches_crawler_runs` | AC6 | E05-P1-024 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-U05 | `test_publish_ingested_event_logs_task_name_run_id_correlation_id` | AC5, AC8 | E05-P2-016 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-U06 | `test_publish_ingested_event_retries_on_xadd_failure` | AC4, AC5 | E05-P0-008 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-U07 | `test_generate_submission_guides_dispatches_publish_ingested_event` | AC7 | — | `generate_submission_guides.py` does not yet call `publish_ingested_event.apply_async` → `AssertionError` |

### Integration Test Inventory (`tests/integration/test_publish_event.py`)

| Test ID | Test Function | AC | Epic ID | RED Reason |
|---------|--------------|-----|---------|-----------|
| ATDD-5.9-I01 | `test_publish_event_happy_path_event_on_stream` | AC1, AC2, AC3 | E05-P0-007 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-I02 | `test_publish_failure_triggers_isolated_retry_not_crawl_rerun` | AC4 | E05-P0-008 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-I03 | `test_publish_summary_matches_crawler_runs` | AC6 | E05-P1-024 | Module missing → `pytest.fail()` (RED) |
| ATDD-5.9-I04 | `test_publish_exhausted_retries_updates_crawler_runs_errors` | AC5 | E05-R-003 | Module missing → `pytest.fail()` (RED) |

---

## Step 4C: Test Infrastructure

### Fixtures Used (from conftest.py)

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `pg_container` | session | PostgreSQL 16 testcontainer + Alembic migrations |
| `eager_celery` | function | `task_always_eager=True, task_eager_propagates=True` |
| `eager_celery_no_propagate` | function | `task_always_eager=True, task_eager_propagates=False` |
| `_clean_tables` | function | TRUNCATE opportunities + crawler_runs CASCADE |
| `_psycopg2_conn` | function | Raw psycopg2 connection factory |
| `_reset_clients_and_db` | function (autouse) | Resets AI client, circuit breaker, engine between tests |

### Local Helpers (defined in integration test file)

| Helper | Purpose |
|--------|---------|
| `_set_test_redis_url` (autouse) | Sets `REDIS_URL=TEST_REDIS_URL` (redis://localhost:6379/1) |
| `_clean_opportunities_stream` (autouse) | Deletes `eu-solicit:opportunities` stream before/after each test |
| `_seed_crawler_run(conn, *, crawler_type, status, found, new_count, updated, errors)` | Insert a `crawler_runs` row in a terminal state |
| `_get_crawler_run(conn, run_id)` | Read `crawler_runs` row by UUID |
| `_make_crawl_result(run_id, ...)` | Build crawl_result dict |
| `_read_latest_stream_message(redis_url)` | `XREVRANGE` on `eu-solicit:opportunities` stream; returns msg dict or None |

### Mock Strategy

| Scenario | Mock Approach |
|----------|--------------|
| Unit tests — Redis | `patch("data_pipeline.workers.tasks.publish_event.redis.Redis.from_url")` |
| Unit tests — _record_publish_failure (retry test) | `patch("data_pipeline.workers.tasks.publish_event._record_publish_failure")` |
| Unit tests — structlog | `patch.object(structlog, "get_logger")` to capture bind() kwargs |
| Unit test — chain dispatch | `patch("data_pipeline.workers.tasks.generate_submission_guides.publish_ingested_event", create=True)` |
| Integration tests I01/I03 — Redis | Real Redis at `TEST_REDIS_URL` (redis://localhost:6379/1) |
| Integration test I02 — Redis failure | `patch("data_pipeline.workers.tasks.publish_event.redis.Redis.from_url")` with `side_effect=ConnectionError` |
| Integration tests — DB | Real testcontainer (no DB mocking) |

---

## Step 5: Validation & Completion

### Checklist Validation

- [x] Prerequisites satisfied (story AC defined, test framework present, CrawlerRun model confirmed)
- [x] Test files created at correct paths:
  - `eusolicit-app/services/data-pipeline/tests/unit/test_publish_event.py`
  - `eusolicit-app/services/data-pipeline/tests/integration/test_publish_event.py`
- [x] All 8 acceptance criteria covered by at least one test
- [x] All tests designed to FAIL before implementation (TDD red phase)
- [x] No `pytest.skip()` used — tests FAIL (not skip) in red phase
- [x] Checklist stored in `test-artifacts/` not a random location
- [x] No orphaned browser sessions (backend-only, no browser automation)
- [x] Tests match patterns from existing codebase (see `test_generate_submission_guides.py`)
- [x] Epic test IDs (E05-P0-007, E05-P0-008, E05-P1-024) referenced in test docstrings

### RED Phase Mechanism

| Category | Tests | How RED State is Enforced |
|----------|-------|--------------------------|
| New module tests (U01–U06, I01–I04) | 10 | `_require_publish_module()` → `pytest.fail()` because `ImportError` on module import |
| Existing module chain tests (U07) | 1 | `AssertionError` — `generate_submission_guides.py` does not yet import/call `publish_ingested_event.apply_async` |

### Implementation Tasks That Will Turn Tests GREEN

| Task | Tests Turned GREEN |
|------|--------------------|
| **Task 1**: Create `publish_event.py` with `publish_ingested_event` + `_record_publish_failure` | ATDD-5.9-U01, U02, U03, U04, U05, U06, I01, I02, I03, I04 |
| **Task 2**: Add `publish_ingested_event.apply_async(args=[crawl_result])` to `generate_submission_guides.py` | ATDD-5.9-U07 |
| **Task 3**: Add `crawler_type` to crawl_result dicts in `crawl_aop.py`, `crawl_ted.py`, `crawl_eu_grants.py` | Not covered by 5.9 tests — verified by integration tests in S05.04–S05.06 |
| **Task 4**: Register `pipeline.publish_ingested_event` in `celery_app.py` `include` list and `task_routes` | Not covered by unit tests — deployment/routing concern |

---

## Traceability Matrix

| Test ID | Story AC | Epic Test ID | Priority | Level |
|---------|----------|--------------|----------|-------|
| ATDD-5.9-U01 | AC1 | E05-P0-007 | P0 | Unit |
| ATDD-5.9-U02 | AC2 | E05-P0-007 | P0 | Unit |
| ATDD-5.9-U03 | AC6 | E05-P1-024 | P1 | Unit |
| ATDD-5.9-U04 | AC6 | E05-P1-024 | P1 | Unit |
| ATDD-5.9-U05 | AC5, AC8 | E05-P2-016 | P2 | Unit |
| ATDD-5.9-U06 | AC4, AC5 | E05-P0-008 | P0 | Unit |
| ATDD-5.9-U07 | AC7 | — | P1 | Unit |
| ATDD-5.9-I01 | AC1, AC2, AC3 | E05-P0-007 | P0 | Integration |
| ATDD-5.9-I02 | AC4 | E05-P0-008 | P0 | Integration |
| ATDD-5.9-I03 | AC6 | E05-P1-024 | P1 | Integration |
| ATDD-5.9-I04 | AC5 | E05-R-003 | P0 | Integration |

### AC Coverage Summary

| AC | Description | Tests | Covered? |
|----|-------------|-------|----------|
| AC1 | E01 envelope with all 7 required fields | U01, I01 | ✅ |
| AC2 | Payload with all 5 required fields + correct types | U02, I01 | ✅ |
| AC3 | Redis client reads event; opportunity_ids deserializes to Python list | I01 | ✅ |
| AC4 | Isolated Celery retry on XADD failure; crawler_runs stays completed | U06, I02 | ✅ |
| AC5 | ERROR log + crawler_runs.errors update on exhausted retries | U05, U06, I04 | ✅ |
| AC6 | summary.new + updated + unchanged == crawler_runs.found | U03, U04, I03 | ✅ |
| AC7 | publish_ingested_event dispatched from generate_submission_guides | U07 | ✅ |
| AC8 | Log lines include task_name, run_id, correlation_id | U05 | ✅ |

---

## Risk Mitigations Verified by Tests

| Risk | Score | Mitigation | Verified By |
|------|-------|-----------|-------------|
| E05-R-003 (Redis Streams publish-or-lose) | 6 | XADD try/except with Celery isolated retry; ERROR log + `crawler_runs.errors` JSONB on exhausted retries | ATDD-5.9-I01 (happy path), ATDD-5.9-I02 (retry isolation), ATDD-5.9-I04 (error record) |

---

## Known Assumptions & Dependencies

| Assumption | Impact | Mitigation |
|------------|--------|-----------|
| Real Redis available at `TEST_REDIS_URL` (redis://localhost:6379/1) for integration tests I01 and I03 | Tests I01/I03 fail if Redis not reachable | `_read_latest_stream_message` returns None on Redis errors; tests assert msg is not None → clear failure message guides CI setup |
| `crawler_type` field populated in `crawl_result` dict by Task 3 | `payload["crawler_type"]` would be "unknown" without Task 3 | Tests use explicit `crawler_type="aop"` in `_make_crawl_result` — not dependent on crawler task implementation |
| Celery eager mode with `task_eager_propagates=True` retries synchronously (recursive) | With `xadd` always failing, task exhausts 3 retries and calls `_record_publish_failure` | Unit test U06 patches `_record_publish_failure` to avoid DB interaction; integration test I02 patches `from_url` |
| `CrawlerRun.new_count` maps to `"new"` DB column | psycopg2 cursor must use `"new"` (quoted) in raw SQL | Confirmed by model — `mapped_column("new", ...)` |

---

## Next Steps (TDD Green Phase)

After developer implements S05.09:

1. **Run tests to verify RED state:**
   ```bash
   cd eusolicit-app/services/data-pipeline
   python -m pytest tests/unit/test_publish_event.py tests/integration/test_publish_event.py -v
   # Expected: all 11 tests FAIL (RED phase confirmed)
   ```

2. **Implement S05.09 (Tasks 1–4)** following the story spec.

3. **Run tests again to verify GREEN state:**
   ```bash
   python -m pytest tests/unit/test_publish_event.py tests/integration/test_publish_event.py -v
   # Expected: all 11 tests PASS (GREEN phase)
   ```

4. **Run full test suite regression:**
   ```bash
   python -m pytest tests/ -v --timeout=120
   # Expected: existing tests still pass; no regressions in guide generation chain
   ```

5. **Commit passing tests** once all tests are GREEN.

6. **Run `/bmad-testarch-automate`** to expand coverage beyond the ATDD scenarios.

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** BMad v6
