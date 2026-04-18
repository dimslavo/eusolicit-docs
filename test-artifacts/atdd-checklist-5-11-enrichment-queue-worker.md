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
storyId: 5-11-enrichment-queue-worker
storyFile: eusolicit-docs/implementation-artifacts/5-11-enrichment-queue-worker.md
detectedStack: backend
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-11-enrichment-queue-worker.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-app/services/data-pipeline/tests/unit/test_cleanup_expired.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_cleanup_expired.py
  - eusolicit-app/services/data-pipeline/tests/conftest.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_score_opportunities.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_generate_submission_guides.py
  - eusolicit-app/services/data-pipeline/tests/unit/test_celery_beat.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 5.11 — Enrichment Queue Worker

**Date:** 2026-04-17
**Author:** BMad TEA Master Test Architect
**Story:** S05.11 — Enrichment Queue Worker (`process_enrichment_queue`)
**Status:** 🔴 TDD RED PHASE — Tests generated, awaiting implementation

---

## TDD Red Phase Status

🔴 **All tests are in RED phase** — they will fail with `ImportError` or `pytest.fail()` until
`process_enrichment_queue.py` is created (S05.11 Task 1) and the Beat schedule is updated
(S05.11 Task 6).

### Generated Test Files

| File | Type | Tests | Status |
|------|------|-------|--------|
| `services/data-pipeline/tests/unit/test_process_enrichment_queue.py` | Unit | 12 | 🔴 RED |
| `services/data-pipeline/tests/integration/test_process_enrichment_queue.py` | Integration | 6 | 🔴 RED |

**Total: 18 failing tests** (all will fail until implementation is complete)

---

## Stack Detection

- **Detected stack**: `backend` (Python, Celery, SQLAlchemy, PostgreSQL, Redis)
- **Generation mode**: AI generation (no browser recording needed)
- **Test framework**: pytest + testcontainers + unittest.mock

---

## Test Strategy

### Acceptance Criteria → Test Level Mapping

| AC | Description | Unit Tests | Integration Tests |
|----|-------------|-----------|-------------------|
| AC1 | Pending items fetched in FIFO order, batch size from env var | U01, U02, U10 | I01 |
| AC2 | Successful relevance_scoring: opp updated, item deleted, no event | U07 (negative) | I02 |
| AC3 | Successful submission_guide: guide upserted, item deleted, no event | — | I03 |
| AC4 | 3 failed attempts → status='failed', enrichment.failed event | U07, U08, U09 | I04 |
| AC5 | Single item failure does not abort batch | U06 | I06 |
| AC6 | Soft-deleted parent → skip and hard-delete item | U03, U04, U05 | I05 |
| AC7 | Beat schedule entry 'enrichment-queue-worker' exists, 15-min interval | U12 | — |
| AC8 | Structured logs include required fields at all log levels | U11 | — |

### Test Level Rationale

- **Unit tests** cover: business logic isolation (batch loop, counter accumulation, event envelope
  shape, log fields, retry threshold logic, skip conditions), env var configuration, Beat schedule
  configuration. Mocked: `get_sync_session`, `get_client`, `redis.Redis.from_url`.

- **Integration tests** cover: actual DB state transitions (relevance_scores JSONB updated, item
  hard-deleted, submission_guides upserted), Redis stream event publishing and payload validation,
  FIFO ordering with real timestamps, partial batch failure isolation. Uses: testcontainers
  PostgreSQL + Redis + mock AI Gateway.

---

## Unit Tests (12 tests)

### ATDD-5.11-U01: No pending items → zero counts (AC1)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_no_pending_items_returns_zero_counts`
- **AC**: AC1
- **Priority**: P1
- **Assertions**:
  - [ ] Result is `{"processed": 0, "succeeded": 0, "failed": 0, "skipped": 0}`
  - [ ] `redis.Redis.from_url` NOT called (no event on empty batch)
  - [ ] Task completes without raising

---

### ATDD-5.11-U02: ENRICHMENT_BATCH_SIZE env var controls SELECT LIMIT (AC1, E05-P2-013)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_batch_size_from_env_var`
- **AC**: AC1, E05-P2-013
- **Priority**: P2
- **Assertions**:
  - [ ] `ENRICHMENT_BATCH_SIZE=3` → SELECT compiled with LIMIT 3
  - [ ] Task completes successfully (returns 0 items, no error)

---

### ATDD-5.11-U03: Soft-deleted opportunity → item skipped + hard-deleted (AC6)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_item_skipped_when_opportunity_soft_deleted`
- **AC**: AC6
- **Priority**: P1
- **Assertions**:
  - [ ] `session.delete(item)` called once
  - [ ] AI Gateway `get_client()` NOT called
  - [ ] Result `skipped=1, succeeded=0`

---

### ATDD-5.11-U04: Missing opportunity (None) → item skipped + hard-deleted (AC6)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_item_skipped_when_opportunity_not_found`
- **AC**: AC6
- **Priority**: P1
- **Assertions**:
  - [ ] `session.delete(item)` called
  - [ ] AI Gateway NOT called
  - [ ] Result `skipped=1`

---

### ATDD-5.11-U05: Unknown enrichment_type → item skipped (AC5 edge)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_unknown_enrichment_type_skipped`
- **AC**: AC5 (edge case)
- **Priority**: P2
- **Assertions**:
  - [ ] Item deleted, AI Gateway NOT called
  - [ ] Result `skipped=1`

---

### ATDD-5.11-U06: Single item failure does not abort batch (AC5)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_single_item_failure_does_not_abort_batch`
- **AC**: AC5
- **Priority**: P1
- **Assertions**:
  - [ ] `_process_single_item` called 3 times (once per item)
  - [ ] Result `processed=3, succeeded=2, failed=1`
  - [ ] Task does NOT raise

---

### ATDD-5.11-U07: Item marked failed after 3 attempts, Redis event (AC4, E05-P1-029)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_item_marked_failed_after_3_attempts`
- **AC**: AC4
- **Priority**: P1 (E05-P1-029)
- **Assertions**:
  - [ ] `item.status == "failed"`
  - [ ] `item.attempts == 3`
  - [ ] `item.last_error` non-empty
  - [ ] `redis.xadd` called exactly once

---

### ATDD-5.11-U08: Failed event payload matches E01 envelope format (AC4)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_failed_event_payload_structure`
- **AC**: AC4
- **Priority**: P1
- **Assertions**:
  - [ ] Stream = `eu-solicit:enrichment`
  - [ ] Outer envelope: `event_type="EnrichmentFailed"`, `source_service="data-pipeline"`, `tenant_id=""`
  - [ ] Inner payload: `item_id`, `opportunity_id`, `enrichment_type`, `attempts=3`, `last_error`, `failed_at`

---

### ATDD-5.11-U09: Item pending after <3 attempts, no Redis event (AC4)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_item_pending_restored_after_failed_attempt`
- **AC**: AC4
- **Priority**: P1
- **Assertions**:
  - [ ] `item.status == "pending"` (not "failed")
  - [ ] `item.attempts == 2` (1 prior + 1)
  - [ ] `item.last_error` set
  - [ ] `redis.Redis.from_url` NOT called

---

### ATDD-5.11-U10: SELECT query orders by created_at ASC (AC1)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_batch_processing_order_fifo`
- **AC**: AC1
- **Priority**: P1
- **Assertions**:
  - [ ] Compiled SELECT SQL contains `ORDER BY`, `CREATED_AT`, `ASC`

---

### ATDD-5.11-U11: Log fields include required keys (AC8)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_log_fields_include_required_keys`
- **AC**: AC8
- **Priority**: P2
- **Assertions**:
  - [ ] `task_name="pipeline.process_enrichment_queue"` in bound log context
  - [ ] `correlation_id` (non-empty) in bound log context
  - [ ] Batch completion log includes `batch_size`, `processed`, `succeeded`, `failed`, `skipped`

---

### ATDD-5.11-U12: Beat schedule entry 'enrichment-queue-worker' exists (AC7)

- **File**: `tests/unit/test_process_enrichment_queue.py`
- **Function**: `test_beat_schedule_entry_exists`
- **AC**: AC7
- **Priority**: P1
- **Assertions**:
  - [ ] `"enrichment-queue-worker"` in `app.conf.beat_schedule`
  - [ ] `task == "pipeline.process_enrichment_queue"`
  - [ ] `schedule == timedelta(minutes=15)` (default)
  - [ ] `task_routes["pipeline.process_enrichment_queue"]["queue"] == "pipeline_crawl"`

---

## Integration Tests (6 tests)

### E05-P1-028 / ATDD-5.11-I01: FIFO processing and batch size (AC1, E05-P2-013)

- **File**: `tests/integration/test_process_enrichment_queue.py`
- **Function**: `test_fifo_processing_and_batch_size`
- **AC**: AC1
- **Priority**: P1 (E05-P1-028), P2 (E05-P2-013)
- **Fixtures**: `pg_container`, `eager_celery`, `_clean_tables`, `monkeypatch`
- **Setup**: 5 pending items with created_at spaced 1s apart, ENRICHMENT_BATCH_SIZE=3
- **Assertions**:
  - [ ] `result.result["processed"] == 3`
  - [ ] Oldest 3 items hard-deleted (FIFO)
  - [ ] 2 newest items remain with status='pending'
  - [ ] 2 pending items remain in DB

---

### E05-P1-030 / ATDD-5.11-I02: Successful scoring retry updates opportunity (AC2)

- **File**: `tests/integration/test_process_enrichment_queue.py`
- **Function**: `test_successful_scoring_retry_updates_opportunity_and_deletes_item`
- **AC**: AC2
- **Priority**: P1 (E05-P1-030)
- **Fixtures**: `pg_container`, `eager_celery`, `_clean_tables`
- **Setup**: 1 opportunity (empty relevance_scores), 2 company profiles, 1 pending queue item (attempts=1)
- **Assertions**:
  - [ ] EnrichmentQueueItem hard-deleted
  - [ ] `opportunity.relevance_scores` has 2 keys (one per company_id)
  - [ ] Each score value is a float
  - [ ] `xlen(eu-solicit:enrichment) == 0` (no event on success)

---

### E05-P1-030 / ATDD-5.11-I03: Successful guide retry creates/updates guide (AC3)

- **File**: `tests/integration/test_process_enrichment_queue.py`
- **Function**: `test_successful_guide_retry_creates_or_updates_guide_and_deletes_item`
- **AC**: AC3
- **Priority**: P1 (E05-P1-030)
- **Fixtures**: `pg_container`, `eager_celery`, `_clean_tables`
- **Setup**: 1 opportunity, 1 pending submission_guide queue item (attempts=1)
- **Assertions**:
  - [ ] EnrichmentQueueItem hard-deleted
  - [ ] SubmissionGuide row exists with populated steps (≥1 step)
  - [ ] `guide.reviewed == False`
  - [ ] No event on stream

---

### E05-P1-029 / ATDD-5.11-I04: Item failed after 3 attempts emits event (AC4)

- **File**: `tests/integration/test_process_enrichment_queue.py`
- **Function**: `test_item_marked_failed_after_3_attempts_emits_event`
- **AC**: AC4
- **Priority**: P1 (E05-P1-029)
- **Fixtures**: `pg_container`, `eager_celery`, `_clean_tables`
- **Setup**: 1 pending item (attempts=2), AI Gateway mocked to raise `AIGatewayUnavailableError`
- **Assertions**:
  - [ ] `item.status == "failed"` (DB)
  - [ ] `item.attempts == 3` (DB)
  - [ ] `item.last_error` non-empty (DB)
  - [ ] 1 message on `eu-solicit:enrichment`
  - [ ] Full E01 envelope validation (event_type, source_service, tenant_id, event_id, etc.)
  - [ ] Inner payload: item_id, opportunity_id, enrichment_type, attempts=3, last_error, failed_at

---

### ATDD-5.11-I05: Soft-deleted opportunity → item skipped (AC6)

- **File**: `tests/integration/test_process_enrichment_queue.py`
- **Function**: `test_soft_deleted_opportunity_skips_item`
- **AC**: AC6
- **Priority**: P1
- **Fixtures**: `pg_container`, `eager_celery`, `_clean_tables`
- **Setup**: 1 soft-deleted opportunity (deleted_at set), 1 pending queue item
- **Assertions**:
  - [ ] `result.result["skipped"] == 1`
  - [ ] EnrichmentQueueItem hard-deleted
  - [ ] `get_client()` NOT called
  - [ ] No event on stream

---

### ATDD-5.11-I06: Partial batch failure — batch continues (AC5, AC4)

- **File**: `tests/integration/test_process_enrichment_queue.py`
- **Function**: `test_partial_failure_does_not_abort_batch`
- **AC**: AC5, AC4
- **Priority**: P1
- **Fixtures**: `pg_container`, `eager_celery`, `_clean_tables`
- **Setup**: Items A (attempts=0), B (attempts=2), C (attempts=0); AI Gateway fails for B's opportunity
- **Assertions**:
  - [ ] `result.result["succeeded"] == 2, failed == 1, skipped == 0`
  - [ ] Item A hard-deleted (success)
  - [ ] Item B: `status="failed"`, `attempts=3` in DB
  - [ ] Item C hard-deleted (success)
  - [ ] Exactly 1 event on stream (for item B only)
  - [ ] Event `payload.item_id` matches item B

---

## Test Design Traceability

| Test ID | Epic Test ID | Priority | Requirement |
|---------|-------------|----------|-------------|
| ATDD-5.11-I01 | E05-P1-028 | P1 | FIFO processing + batch size cap |
| ATDD-5.11-I04 | E05-P1-029 | P1 | 3 failures → failed status + enrichment.failed event |
| ATDD-5.11-I02 | E05-P1-030 | P1 | Successful retry updates opportunity + deletes item |
| ATDD-5.11-I03 | E05-P1-030 | P1 | Successful guide retry upserts guide + deletes item |
| ATDD-5.11-U02 | E05-P2-013 | P2 | ENRICHMENT_BATCH_SIZE env var controls batch |

### Risk Mitigations Covered

| Risk ID | Risk Description | Tests Verifying |
|---------|-----------------|-----------------|
| E05-R-007 | Enrichment queue unbounded failure | U07, U08, U09, I04, I06 |
| E05-R-006 | Soft-delete filter bypass (AI quota waste) | U03, U04, I05 |

---

## Red Phase Failure Analysis

All 18 tests will fail due to **ImportError** at collection time:

```
ImportError: No module named 'data_pipeline.workers.tasks.process_enrichment_queue'
```

This triggers the conditional import guard, setting `_EQ_MODULE_AVAILABLE = False`.
Each test then calls `_require_eq_module()` which immediately calls:

```python
pytest.fail(
    "🔴 RED PHASE: process_enrichment_queue module not yet implemented. ..."
)
```

**Additionally**, `test_beat_schedule_entry_exists` (ATDD-5.11-U12) will fail even if the module
exists, because `"enrichment-queue-worker"` is not yet in `app.conf.beat_schedule` (requires
S05.11 Task 6).

---

## Implementation Guidance

Tasks required to turn tests green (in dependency order):

| Step | Story Task | Tests Unblocked |
|------|-----------|-----------------|
| 1 | Create `process_enrichment_queue.py` with `process_enrichment_queue` Celery task | All unit tests (U01-U11) |
| 2 | Implement `_process_single_item` helper | U03, U04, U05, U06, U07, U08, U09 |
| 3 | Implement `_retry_relevance_scoring` helper | U07, I01, I02, I06 |
| 4 | Implement `_retry_submission_guide` helper | I03 |
| 5 | Implement `_publish_enrichment_failed_event` helper | U07, U08, I04, I06 |
| 6 | Register task and Beat schedule in `celery_app.py` | U12 |

---

## Next Steps (TDD Green Phase)

After implementing S05.11 Tasks 1–6:

1. **Run unit tests**: `pytest tests/unit/test_process_enrichment_queue.py -v`
   - Expected: 12/12 pass (all green)
2. **Run integration tests**: `pytest tests/integration/test_process_enrichment_queue.py -v --timeout=60`
   - Expected: 6/6 pass (all green)
3. If any tests fail — either fix the implementation or tighten the test based on findings
4. Verify `ENRICHMENT_BATCH_SIZE` env var override works in integration test I01
5. Verify Redis stream cleanup fixture prevents cross-test pollution
6. Run full E05 suite to verify no regressions: `pytest tests/ -m "unit or integration" -v`

---

## Notes and Assumptions

1. **`eusolicit_models.company_profile.CompanyProfile`** does not exist in the package yet.
   The integration tests (I01, I02, I04, I06) depend on `_retry_relevance_scoring` querying
   `CompanyProfile` via ORM. When `process_enrichment_queue.py` is created, the developer must
   also ensure `eusolicit_models.company_profile` is importable with the ORM model.

2. **`_clean_tables` CASCADE** — The conftest `_clean_tables` fixture truncates
   `pipeline.opportunities CASCADE`, which should also truncate `pipeline.enrichment_queue`
   due to the FK with ON DELETE CASCADE. If not, integration tests may need a dedicated
   enrichment_queue truncate.

3. **AI Gateway mock** — Integration tests use `patch("...get_client")` following the existing
   pattern in `test_score_opportunities.py`. The story mentions `respx`, but the codebase mocks
   at the `get_client()` boundary for all integration tests.

4. **`with_for_update(skip_locked=True)`** — Not directly tested in unit tests (it's a
   PostgreSQL-specific feature). The FIFO integration test (I01) implicitly validates it works
   correctly by verifying only 3 of 5 items are processed.

5. **Beat schedule env var override** — `ENRICHMENT_QUEUE_INTERVAL_MINUTES` override is not
   explicitly tested by a separate unit test (deferred to P3). The default 15-minute interval
   is verified in U12.

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
