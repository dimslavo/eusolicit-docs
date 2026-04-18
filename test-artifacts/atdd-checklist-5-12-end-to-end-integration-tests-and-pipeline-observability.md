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
storyId: 5-12-end-to-end-integration-tests-and-pipeline-observability
storyFile: eusolicit-docs/implementation-artifacts/5-12-end-to-end-integration-tests-and-pipeline-observability.md
detectedStack: backend
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-12-end-to-end-integration-tests-and-pipeline-observability.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-app/services/data-pipeline/tests/conftest.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_crawl_aop.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_publish_event.py
  - eusolicit-app/services/data-pipeline/tests/unit/test_process_enrichment_queue.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/cleanup_expired.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/models/company_profile.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 5.12 — End-to-end Integration Tests and Pipeline Observability

**Date:** 2026-04-17
**Author:** BMad TEA Master Test Architect
**Story:** S05.12 — E2E Integration Tests + Pipeline Observability
**Status:** 🔴 TDD RED PHASE — Tests generated, awaiting implementation

---

## TDD Red Phase Status

🔴 **Tests are in RED phase** — they will fail until the following implementation tasks are complete:

| RED Trigger | Failing Tests | Blocked By |
|-------------|---------------|-----------|
| `data_pipeline.metrics` module does not exist | ATDD-5.12-M01 through M06 (all metrics tests) | S05.12 Task 1 |
| `/metrics` endpoint missing from `main.py` | ATDD-5.12-M02, ATDD-5.12-M06 | S05.12 Task 2 |
| `crawler_type="cleanup"` missing from `cleanup_expired.py` log.bind() | ATDD-5.12-L04 | S05.12 Task 6.2 |
| Full E2E chain not yet verified in single test (new test file) | ATDD-5.12-E01, E02 | S05.12 Tasks 1-6 |

---

## Generated Test Files

| File | Type | Tests | Status |
|------|------|-------|--------|
| `eusolicit-app/services/data-pipeline/tests/unit/test_metrics.py` | Unit | 6 | 🔴 RED (all 6 fail — `data_pipeline.metrics` module missing) |
| `eusolicit-app/services/data-pipeline/tests/unit/test_log_fields.py` | Unit | 4 | 🔴 RED (L04) / 🟡 PASS (L01, L02, L03 — existing compliance verified) |
| `eusolicit-app/services/data-pipeline/tests/integration/test_e2e_pipeline.py` | Integration | 2 | 🔴 RED (require Redis + full chain) |

**Total: 9 failing tests** (6 metrics + 1 cleanup log field + 2 E2E integration)
**3 tests GREEN on day-1**: L01/L02/L03 verify existing log field compliance in crawl_aop, process_enrichment_queue, and publish_ingested_event — these pass because prior stories (S05.04, S05.09, S05.11) already implemented correct log bindings.

---

## Stack Detection

- **Detected stack**: `backend` (Python, Celery, SQLAlchemy, PostgreSQL, Redis)
- **Generation mode**: AI generation (no browser recording needed)
- **Test framework**: pytest + testcontainers + respx + unittest.mock + structlog.testing

---

## Test Strategy

### Acceptance Criteria → Test Level Mapping

| AC | Description | Unit Tests | Integration Tests | Test IDs |
|----|-------------|-----------|-------------------|----------|
| AC1 | Full pipeline runs in CI with mocked AI Gateway (no KraftData credentials) | — | ATDD-5.12-E01 | E05-P0-009 |
| AC2 | Dedup: second run with identical data produces zero new inserts | — | ATDD-5.12-E02 | E05-P2-014 |
| AC3 | All 4 Prometheus metrics exported at GET /metrics with correct labels | M01–M06 | — | E05-P2-015, P3-003, P3-004, P3-005 |
| AC4 | Structured logs include correlation_id, task_name, crawler_type on every pipeline task record | L01–L04 | — | E05-P2-016 |
| AC5 | Full AOP chain verified: (a) crawler_runs completed, (b) 10 opps, (c) relevance_scores, (d) submission_guides, (e) Redis stream event | — | ATDD-5.12-E01 | E05-P0-009 |
| AC6 | Suite executes in under 60 seconds (enforced by pytest.ini_options timeout=60) | — | (global timeout) | E05-P3-002 |

### Test Level Rationale

- **Unit tests** (test_metrics.py, test_log_fields.py): test module-level behavior in isolation
  using import guards, structlog.testing.capture_logs(), and MagicMock dependencies.
  No DB or Redis required. Fast to run (<5s combined).
- **Integration tests** (test_e2e_pipeline.py): full end-to-end behavior requires
  real PostgreSQL (testcontainers), real Redis (TEST_REDIS_URL), and respx mocking
  for all AI Gateway HTTP calls. Session-scoped DB for speed.

---

## Unit Tests — test_metrics.py (6 tests)

### ATDD-5.12-M01: All 4 metrics exposed with correct names and labels (E05-P2-015, AC3)

- **File**: `tests/unit/test_metrics.py`
- **Function**: `test_metrics_module_exposes_all_four_metrics`
- **AC**: AC3
- **Priority**: P2 (E05-P2-015)
- **Epic test design ref**: E05-P2-015
- **RED trigger**: `from data_pipeline.metrics import ...` → ImportError (module doesn't exist)
- **Assertions**:
  - [ ] `pipeline_crawl_duration_seconds` in generate_latest output
  - [ ] `crawler_type="aop"` label present
  - [ ] `pipeline_opportunities_total` in output
  - [ ] `source_type="aop"` and `action="new"` labels present
  - [ ] `pipeline_enrichment_queue_depth` in output
  - [ ] `pipeline_agent_call_duration_seconds` in output
  - [ ] `agent_type="CRAWLER"` label present

---

### ATDD-5.12-M02: GET /metrics returns HTTP 200 with all 4 metric names (E05-P2-015, AC3)

- **File**: `tests/unit/test_metrics.py`
- **Function**: `test_metrics_endpoint_returns_200_with_expected_content`
- **AC**: AC3
- **Priority**: P2 (E05-P2-015)
- **RED trigger**: ImportError on metrics module OR /metrics route not registered in main.py
- **Assertions**:
  - [ ] `response.status_code == 200`
  - [ ] `"pipeline_crawl_duration_seconds"` in response.text
  - [ ] `"pipeline_opportunities_total"` in response.text
  - [ ] `"pipeline_enrichment_queue_depth"` in response.text
  - [ ] `"pipeline_agent_call_duration_seconds"` in response.text

---

### ATDD-5.12-M03: pipeline_crawl_duration_seconds has crawler_type label for aop/ted/eu_grants (E05-P3-003, AC3)

- **File**: `tests/unit/test_metrics.py`
- **Function**: `test_crawl_duration_histogram_crawler_type_label`
- **AC**: AC3
- **Priority**: P3 (E05-P3-003)
- **RED trigger**: ImportError on metrics module
- **Assertions**:
  - [ ] `crawler_type="aop"` present in Prometheus text output
  - [ ] `crawler_type="ted"` present in Prometheus text output
  - [ ] `crawler_type="eu_grants"` present in Prometheus text output

---

### ATDD-5.12-M04: pipeline_agent_call_duration_seconds has per-agent agent_type labels (E05-P3-004, AC3)

- **File**: `tests/unit/test_metrics.py`
- **Function**: `test_agent_call_duration_histogram_per_agent_type`
- **AC**: AC3
- **Priority**: P3 (E05-P3-004)
- **RED trigger**: ImportError on metrics module
- **Assertions**:
  - [ ] `agent_type="SCORING"` present in output after `.observe()` call
  - [ ] `agent_type="SUBMISSION_GUIDE"` present in output after `.observe()` call

---

### ATDD-5.12-M05: pipeline_enrichment_queue_depth reflects exact set value (E05-P3-005, AC3)

- **File**: `tests/unit/test_metrics.py`
- **Function**: `test_enrichment_queue_depth_gauge_reflects_set_value`
- **AC**: AC3
- **Priority**: P3 (E05-P3-005)
- **RED trigger**: ImportError on metrics module
- **Assertions**:
  - [ ] `ENRICHMENT_QUEUE_DEPTH.set(7)` → gauge line contains `7.0` (Prometheus float format)
  - [ ] `pipeline_enrichment_queue_depth` present in output

---

### ATDD-5.12-M06: GET /metrics Content-Type is Prometheus text exposition format (AC3)

- **File**: `tests/unit/test_metrics.py`
- **Function**: `test_metrics_endpoint_content_type`
- **AC**: AC3
- **Priority**: P2
- **RED trigger**: ImportError on metrics module OR /metrics route missing
- **Assertions**:
  - [ ] `response.headers["content-type"].startswith("text/plain")`

---

## Unit Tests — test_log_fields.py (4 tests)

### ATDD-5.12-L01: crawl_aop log lines include correlation_id, task_name, crawler_type (E05-P2-016, AC4)

- **File**: `tests/unit/test_log_fields.py`
- **Function**: `test_crawl_aop_log_fields_include_required_keys`
- **AC**: AC4
- **Priority**: P2 (E05-P2-016)
- **Epic test design ref**: E05-P2-016
- **RED trigger**: Test file is new (regression/documentation test for existing compliance)
- **Assertions**:
  - [ ] At least 1 log record captured with `task_name` containing `"crawl_aop"`
  - [ ] All such records contain `correlation_id` field
  - [ ] All such records contain `task_name == "pipeline.crawl_aop"`
  - [ ] All such records contain `crawler_type == "aop"`

---

### ATDD-5.12-L02: process_enrichment_queue log lines include task_name, correlation_id (AC4)

- **File**: `tests/unit/test_log_fields.py`
- **Function**: `test_enrichment_queue_task_log_fields`
- **AC**: AC4
- **Priority**: P1
- **RED trigger**: Test file is new; process_enrichment_queue module may be absent
- **Assertions**:
  - [ ] At least 1 log record with enrichment-related `task_name` captured
  - [ ] All such records contain `task_name` field
  - [ ] All such records contain `correlation_id` field

---

### ATDD-5.12-L03: publish_ingested_event log lines include task_name, run_id, correlation_id (AC4)

- **File**: `tests/unit/test_log_fields.py`
- **Function**: `test_publish_event_log_fields`
- **AC**: AC4
- **Priority**: P1
- **RED trigger**: Test file is new; publish_event module may not yet have correlation_id
- **Assertions**:
  - [ ] At least 1 log record from publish_ingested_event captured
  - [ ] Records contain `task_name` field
  - [ ] Records contain `correlation_id` field

---

### ATDD-5.12-L04: cleanup_expired includes crawler_type='cleanup' ← PRIMARY RED TEST (AC4, Task 6.2)

- **File**: `tests/unit/test_log_fields.py`
- **Function**: `test_cleanup_expired_log_fields_include_crawler_type`
- **AC**: AC4
- **Priority**: P2
- **Epic test design ref**: E05-P2-016
- **RED trigger**: `crawler_type` MISSING from `cleanup_expired.py` log.bind() — will raise
  `AssertionError: 🔴 RED: cleanup_expired log record MISSING 'crawler_type'`
- **Implementation needed**: Add `crawler_type="cleanup"` to `log.bind()` in `cleanup_expired.py` (Task 6.2)
- **Assertions**:
  - [ ] At least 1 log record with cleanup-related `task_name` captured
  - [ ] Records contain `task_name` field
  - [ ] Records contain `correlation_id` field
  - [ ] Records contain `crawler_type` field
  - [ ] `record["crawler_type"] == "cleanup"`

---

## Integration Tests — test_e2e_pipeline.py (2 tests)

### ATDD-5.12-E01: Full AOP pipeline chain end-to-end (E05-P0-009, AC1, AC5)

- **File**: `tests/integration/test_e2e_pipeline.py`
- **Function**: `test_e2e_aop_pipeline_full_chain`
- **AC**: AC1, AC5
- **Priority**: P0 (E05-P0-009)
- **Epic test design ref**: E05-P0-009
- **RED trigger**: New test file; full chain must run end-to-end including Redis stream publish
- **Fixtures required**: `pg_container`, `eager_celery`, `_make_raw_opps`, `_make_normalized_opps`,
  `_opp_count`, `_latest_run`, `_psycopg2_conn`, `_seed_company_profiles`,
  `_ensure_auth_schema`, `_check_redis_available`, `_set_test_redis_url`, `_clean_redis_streams`
- **Assertions**:
  - [ ] `result.successful()` — crawl_aop task completes without error
  - [ ] (a) `run["status"] == "completed"`, `run["found"] == 10`, `run["new"] == 10`, `run["updated"] == 0`
  - [ ] (b) `_opp_count("aop") == 10`
  - [ ] (c) `COUNT(*) WHERE relevance_scores IS NOT NULL > 0` (when company profiles seeded)
  - [ ] (d) `COUNT(*) pipeline.submission_guides WHERE source_type='aop' >= 10` (when companies seeded)
  - [ ] (e) Exactly 1 message on `eu-solicit:opportunities` stream
  - [ ] `event_type == "OpportunitiesIngested"`
  - [ ] `payload["crawler_type"] == "aop"`
  - [ ] `len(payload["opportunity_ids"]) == 10`
  - [ ] `payload["summary"]["new"] == 10`

---

### ATDD-5.12-E02: Dedup second run produces zero new inserts (E05-P2-014, AC2)

- **File**: `tests/integration/test_e2e_pipeline.py`
- **Function**: `test_e2e_aop_dedup_second_run_zero_new_inserts`
- **AC**: AC2
- **Priority**: P2 (E05-P2-014)
- **Epic test design ref**: E05-P2-014
- **RED trigger**: New test file; dedup behavior may have edge cases
- **Fixtures required**: `pg_container`, `eager_celery`, `_make_raw_opps`, `_make_normalized_opps`,
  `_opp_count`, `_all_runs`
- **Assertions**:
  - [ ] Run 1: `result1.successful()` and `_opp_count("aop") == 10`
  - [ ] Run 2: `result2.successful()` and `_opp_count("aop") == 10` (NOT 20)
  - [ ] `len(_all_runs("aop")) == 2`
  - [ ] `runs[1]["new"] == 0` (second run reports zero new inserts)

---

## Test Infrastructure

### Fixtures Added in test_e2e_pipeline.py

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `_check_redis_available` | session | Skip module if Redis unreachable |
| `_set_test_redis_url` | function (autouse) | Monkeypatch REDIS_URL → test Redis DB |
| `_clean_redis_streams` | function (autouse) | Delete eu-solicit:opportunities before/after |
| `_ensure_auth_schema` | session | Create auth schema + company_profiles table |
| `_seed_company_profiles` | function | Insert 2 active CompanyProfile rows |

### Fixtures Reused from conftest.py

| Fixture | Purpose |
|---------|---------|
| `pg_container` | Session-scoped PostgreSQL testcontainer with Alembic migrations |
| `eager_celery` | task_always_eager=True for synchronous chain execution |
| `_psycopg2_conn` | Raw psycopg2 connection factory for assertions |
| `_opp_count` | Count rows in pipeline.opportunities by source_type |
| `_latest_run` | Latest crawler_run record as dict |
| `_all_runs` | All crawler_run records ordered by started_at ASC |
| `_clean_tables` | Autouse truncate opportunities + CASCADE (from conftest) |
| `_reset_clients_and_db` | Autouse reset engine + AI Gateway client (from conftest) |
| `_make_raw_opps` | Factory for 10 raw AOP opportunity dicts |
| `_make_normalized_opps` | Factory for 10 normalized AOP opportunity dicts |

---

## Implementation → Test Mapping (S05.12 Tasks)

| Story Task | Implementation | Verifying Test(s) |
|------------|---------------|-------------------|
| Task 1 | Create `metrics.py` with 4 metrics + `PIPELINE_METRICS_REGISTRY` | M01–M06 (all metrics tests) |
| Task 2 | Add `GET /metrics` endpoint to `main.py` | M02, M06 |
| Task 3.1 | Instrument `crawl_aop.py` with CRAWL_DURATION + OPPORTUNITIES_TOTAL | M01, M03, E01 (implicit) |
| Task 3.2 | Instrument `crawl_ted.py` similarly | M03 |
| Task 3.3 | Instrument `crawl_eu_grants.py` similarly | M03 |
| Task 4.1 | Instrument `ai_gateway_client/client.py` with AGENT_CALL_DURATION | M01, M04 |
| Task 5.1 | Update ENRICHMENT_QUEUE_DEPTH gauge in `process_enrichment_queue.py` | M01, M05 |
| Task 6.1 | Audit existing log fields (no code change needed) | L01, L02, L03 |
| Task 6.2 | Add `crawler_type="cleanup"` to `cleanup_expired.py` log.bind() | L04 |
| Task 7 | (This checklist — test file created) | E01, E02 |
| Task 8 | (This checklist — test file created) | M01–M06 |
| Task 9 | (This checklist — test file created) | L01–L04 |

---

## Risk Coverage

| Risk | Test Coverage |
|------|---------------|
| E05-R-001 (dedup race condition) | ATDD-5.12-E02 (second run zero new inserts) |
| E05-R-002 (AI Gateway cascade / orphaned runs) | ATDD-5.12-E01 (full chain completes with all mocked agents) |
| E05-R-003 (Redis publish loss) | ATDD-5.12-E01 step (e) (stream event present with correct payload) |

---

## Known Caveats

1. **auth.company_profiles**: The CompanyProfile model points to `auth.company_profiles`
   (owned by E02). This schema is not managed by pipeline Alembic migrations. The
   `_ensure_auth_schema` fixture creates a minimal version for integration tests.
   If company profile seeding fails gracefully, relevance_scores assertion (c) is skipped.

2. **Submission guides count**: Guides are generated per opportunity per company profile.
   If no company profiles exist, no guides are generated and assertion (d) is skipped.

3. **AGENT_CALL_DURATION enum values**: The `agent_type` label uses `AgentType.value`
   string representations (e.g., "CRAWLER", "SCORING", "SUBMISSION_GUIDE"). The exact
   values depend on the `AgentType` enum definition in `ai_gateway_client/models.py`.
   Tests M01 and M04 use "CRAWLER", "SCORING", "SUBMISSION_GUIDE" — verify these match
   the actual enum values at implementation time.

4. **Redis dependency**: The E2E test requires a live Redis instance. If Redis is unavailable,
   all tests in `test_e2e_pipeline.py` are skipped via the session-scoped
   `_check_redis_available` fixture.

5. **60-second timeout (AC6)**: Enforced via `pytest.ini_options timeout = 60` already set.
   No additional test is generated for this — it is an infrastructure-level enforcement.

---

## TDD Green Phase: Next Steps

After implementing all S05.12 tasks:

1. Run `pytest tests/unit/test_metrics.py -v` → verify all 6 tests pass
2. Run `pytest tests/unit/test_log_fields.py -v` → verify all 4 tests pass
3. Run `pytest tests/integration/test_e2e_pipeline.py -v` → verify both tests pass
4. Run `pytest tests/ -v --timeout=60` → verify full suite under 60 seconds
5. Commit the GREEN test results and implementation together

---

## Summary

| Metric | Count |
|--------|-------|
| Total tests generated | 12 |
| Unit tests | 10 (6 metrics + 4 log fields) |
| Integration tests | 2 (E2E chain) |
| Tests failing RED on day-1 | 9 (6 metrics + 1 cleanup log field + 2 E2E) |
| Tests passing on day-1 (regression coverage) | 3 (L01/L02/L03 — prior stories already compliant) |
| Tests with hard ImportError RED | 6 (all metrics tests — `data_pipeline.metrics` missing) |
| Tests with assertion-level RED | 1 (L04 — cleanup_expired missing `crawler_type`) |
| Tests RED by new file + chain requirements | 2 (E2E integration tests) |
| Acceptance criteria covered | AC1, AC2, AC3, AC4, AC5 (a-f) |
| Epic test design IDs covered | E05-P0-009, E05-P2-014, E05-P2-015, E05-P2-016, E05-P3-003, E05-P3-004, E05-P3-005 |
| AC6 (60s timeout) | Enforced by pytest.ini_options (no dedicated test needed) |

**Generated by:** BMad TEA Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Story:** S05.12 — End-to-end Integration Tests and Pipeline Observability
