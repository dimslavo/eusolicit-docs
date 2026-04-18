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
mode: story-level
storyId: 5-10-expired-opportunity-cleanup-task
detectedStack: backend
executionMode: sequential
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/5-10-expired-opportunity-cleanup-task.md
  - eusolicit-docs/test-artifacts/test-design-epic-05.md
  - eusolicit-app/services/data-pipeline/tests/conftest.py
  - eusolicit-app/services/data-pipeline/tests/integration/test_publish_event.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/models/opportunity.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/models/submission_guide.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/db.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/celery_app.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/beat_schedule.py
  - eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/cleanup.py
---

# ATDD Checklist: Story 5.10 — Expired Opportunity Cleanup Task

**Date:** 2026-04-17
**Author:** TEA Master Test Architect (BMAD bmad-testarch-atdd)
**TDD Phase:** 🔴 RED (failing tests — implementation pending)
**Story:** S05.10 — `cleanup_expired_opportunities` weekly Celery Beat task
**Stack:** Backend (Python / pytest / Celery / SQLAlchemy / Redis)

---

## Prerequisites

- [x] Story 5.10 is `ready-for-dev` with 8 clear acceptance criteria
- [x] Test framework: pytest + Celery eager mode (no Playwright/browser)
- [x] `pg_container` session-scoped testcontainer fixture available in `conftest.py`
- [x] `eager_celery` / `eager_celery_no_propagate` fixtures available
- [x] `_psycopg2_conn`, `_clean_tables`, `_reset_clients_and_db` fixtures available
- [x] Redis test instance at `TEST_REDIS_URL=redis://localhost:6379/1`
- [x] `_set_test_redis_url` + `_clean_opportunities_stream` autouse fixtures established in `test_publish_event.py` (replicated in `test_cleanup_expired.py`)
- [x] Existing cleanup stub: `services/data-pipeline/src/data_pipeline/workers/tasks/cleanup.py`
- [x] Opportunity model with `deleted_at IS NULL` filter via `do_orm_execute` session event
- [x] `get_sync_session()` synchronous context manager available in `db.py`
- [x] Beat schedule entry `cleanup-expired-opportunities` references correct task name `pipeline.cleanup_expired_opportunities`

### Stack Detection

- **Detected stack:** `backend`
- **Test levels generated:** Unit + Integration (no E2E/browser)
- **Generation mode:** AI generation (backend stack → sequential execution)

---

## TDD Red Phase Status

🔴 **All 12 tests will fail** because `cleanup_expired.py` does not exist yet.

Both test files import from `data_pipeline.workers.tasks.cleanup_expired` (the module to be created at S05.10 Task 1). All tests call `_require_cleanup_module()` which calls `pytest.fail("🔴 RED PHASE: ...")` with a descriptive message pointing to the implementation task.

**Why this is intentional:** Tests define the expected behavior before implementation. Once `cleanup_expired.py` is created and `celery_app.py` is updated (S05.10 Tasks 1–3), all tests should pass (green phase).

---

## Generated Test Files

| File | Type | Tests | Status |
|------|------|-------|--------|
| `eusolicit-app/services/data-pipeline/tests/unit/test_cleanup_expired.py` | Unit | 6 | 🔴 RED |
| `eusolicit-app/services/data-pipeline/tests/integration/test_cleanup_expired.py` | Integration | 6 | 🔴 RED |

**Total: 12 failing tests**

---

## Acceptance Criteria Coverage

### AC1 — Expired opportunities soft-deleted after task completes

| Test ID | Level | Test Name | Status |
|---------|-------|-----------|--------|
| ATDD-5.10-U02 | Unit | `test_cleanup_soft_deletes_expired_rows` | 🔴 RED |
| E05-P1-025 / ATDD-5.10-I01 | Integration | `test_cleanup_soft_deletes_opportunities_past_retention` | 🔴 RED |

**Verification approach:**
- Unit: mock `get_sync_session`, assert `session.execute` called 3× (SELECT + 2 UPDATEs) with `update(Opportunity).values(deleted_at=...)` as one of the calls; assert `session.commit()` called
- Integration: insert 3 opps (2 expired at 60d/45d, 1 within 10d); call task; raw psycopg2 query verifies `deleted_at IS NOT NULL` for the 2 expired, `deleted_at IS NULL` for the 1 in-window

### AC2 — Cascade soft-delete to submission_guides in same transaction

| Test ID | Level | Test Name | Status |
|---------|-------|-----------|--------|
| ATDD-5.10-U03 | Unit | `test_cleanup_cascades_soft_delete_to_guides` | 🔴 RED |
| E05-P1-026 / ATDD-5.10-I02 | Integration | `test_cleanup_cascades_soft_delete_to_submission_guides` | 🔴 RED |

**Verification approach:**
- Unit: mock session returns 2 expired IDs; assert `update(SubmissionGuide)` executed with `deleted_at` and `opportunity_id.in_()` filter; assert `session.commit()` called; assert `cascaded_guide_count` in result equals guide `rowcount`
- Integration: seed 1 expired opportunity + 1 linked guide; call task; assert both have `deleted_at IS NOT NULL` via raw psycopg2

### AC3 — OPPORTUNITY_RETENTION_DAYS env var controls window (default 30)

| Test ID | Level | Test Name | Status |
|---------|-------|-----------|--------|
| ATDD-5.10-U04 | Unit | `test_cleanup_retention_days_from_env_var` | 🔴 RED |
| E05-P2-011 / ATDD-5.10-I04 | Integration | `test_cleanup_retention_period_configurable_via_env` | 🔴 RED |

**Verification approach:**
- Unit: monkeypatch `OPPORTUNITY_RETENTION_DAYS=7`; mock `datetime.now` to return `FROZEN_NOW`; capture SELECT statement; compile + extract bound `deadline_X` param; assert `|actual_cutoff - (FROZEN_NOW - timedelta(7))| < 5s`
- Integration: monkeypatch `RETENTION_DAYS=7`; insert opp at deadline=now-20d (should be deleted) and deadline=now-5d (should be kept); call task; assert psycopg2 results

### AC4 — opportunities.expired event published with E01 envelope

| Test ID | Level | Test Name | Status |
|---------|-------|-----------|--------|
| ATDD-5.10-U06 | Unit | `test_cleanup_event_payload_structure` | 🔴 RED |
| E05-P1-027 / ATDD-5.10-I03 | Integration | `test_cleanup_publishes_expired_event_with_ids` | 🔴 RED |

**Verification approach:**
- Unit: mock Redis; run with 3 mocked expired IDs; capture `xadd` call args; assert stream=`eu-solicit:opportunities`, `event_type="OpportunitiesExpired"`, `source_service="data-pipeline"`, `tenant_id=""`, non-empty `event_id`/`correlation_id`/`timestamp`; deserialize `payload`; assert `expired_ids` list of 3 matching UUID strings + `expired_at` non-empty ISO string
- Integration: seed 2 expired opps; call task; `xrevrange`; verify all E01 envelope fields + inner payload with correct 2 IDs

### AC5 — Zero expired → no event published

| Test ID | Level | Test Name | Status |
|---------|-------|-----------|--------|
| ATDD-5.10-U01 | Unit | `test_cleanup_no_expired_opportunities_no_event` | 🔴 RED |
| ATDD-5.10-I06 | Integration | `test_cleanup_no_expired_does_not_publish_event` | 🔴 RED |

**Verification approach:**
- Unit: mock session returns empty list; assert `redis.Redis.from_url` NOT called; assert result `expired_opportunity_count=0`, `cascaded_guide_count=0`
- Integration: seed only future/recent opps; call task; assert `xlen(stream) == 0`

### AC6 — Soft-deleted rows invisible to default SQLAlchemy queries

| Test ID | Level | Test Name | Status |
|---------|-------|-----------|--------|
| E05-P2-012 / ATDD-5.10-I05 | Integration | `test_soft_deleted_rows_excluded_from_model_queries` | 🔴 RED |

**Verification approach:**
- Integration: seed 2 expired + 1 active; call task; raw psycopg2 verifies 3 rows exist (soft-delete doesn't remove); `get_sync_session() + select(Opportunity.id)` returns exactly 1 row (the active one)

### AC7 — INFO log with expired_opportunity_count and cascaded_guide_count

| Test ID | Level | Test Name | Status |
|---------|-------|-----------|--------|
| ATDD-5.10-U05 | Unit | `test_cleanup_logs_required_fields` | 🔴 RED |

**Verification approach:**
- Unit: mock structlog; capture all `bound.info(...)` calls; assert `expired_opportunity_count` and `cascaded_guide_count` appear in at least one INFO log call

### AC8 — All log lines include task_name, retention_days, correlation_id

| Test ID | Level | Test Name | Status |
|---------|-------|-----------|--------|
| ATDD-5.10-U05 | Unit | `test_cleanup_logs_required_fields` | 🔴 RED |

**Verification approach:**
- Unit: same test as AC7; assert `log.bind()` called with `task_name="pipeline.cleanup_expired_opportunities"`, `retention_days` (any int), `correlation_id` (non-empty string)

---

## Test Coverage Summary

| Priority | Count | Test IDs |
|----------|-------|----------|
| P1 | 8 | U01, U02, U03, U05, U06, I01, I02, I03 |
| P2 | 4 | U04, I04, I05, I06 |
| **Total** | **12** | |

### Priority Rationale
- **P1**: Core cleanup behavior (soft-delete, cascade, event publish, no-event guard, logging)
- **P2**: Configuration (retention days), model filter enforcement, no-event integration

---

## Test Design Traceability

| Test ID (ATDD) | Epic Test Design ID | Story AC | Description |
|----------------|---------------------|----------|-------------|
| ATDD-5.10-U01 | — | AC5 | No expired → no Redis event (unit) |
| ATDD-5.10-U02 | — | AC1 | Bulk UPDATE on Opportunity (unit) |
| ATDD-5.10-U03 | — | AC2 | Cascade UPDATE on SubmissionGuide (unit) |
| ATDD-5.10-U04 | E05-P2-011 | AC3 | Retention days env var (unit) |
| ATDD-5.10-U05 | E05-P2-016 | AC7, AC8 | Structured log fields (unit) |
| ATDD-5.10-U06 | E05-P1-027 | AC4 | Event envelope + payload shape (unit) |
| ATDD-5.10-I01 | E05-P1-025 | AC1 | Soft-delete past-retention opps (integration) |
| ATDD-5.10-I02 | E05-P1-026 | AC2 | Cascade soft-delete to guides (integration) |
| ATDD-5.10-I03 | E05-P1-027 | AC4 | Event published with correct IDs (integration) |
| ATDD-5.10-I04 | E05-P2-011 | AC3 | Retention period via env (integration) |
| ATDD-5.10-I05 | E05-P2-012 | AC6 | Model queries exclude soft-deleted (integration) |
| ATDD-5.10-I06 | — | AC5 | No event when no expired (integration) |

---

## Risk Mitigations Tested

| Risk | Score | Test(s) |
|------|-------|---------|
| E05-R-006 — Soft-delete filter bypass | 4 | ATDD-5.10-I05 (E05-P2-012): verifies ORM filter enforced after cleanup |
| E05-R-003 — Redis publish loss | 6 | ATDD-5.10-U01/I06: no empty event; ATDD-5.10-U06/I03: correct payload |

---

## Implementation Guidance

### What needs to be created

**Task 1: `cleanup_expired.py`** (primary RED-phase target)

```
eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/cleanup_expired.py
```

Must implement:
1. `@shared_task(name="pipeline.cleanup_expired_opportunities", bind=True, queue="pipeline_crawl")` decorator
2. Read `OPPORTUNITY_RETENTION_DAYS` env var (default `"30"`)
3. Compute `cutoff_date = datetime.now(UTC) - timedelta(days=retention_days)`
4. Open `get_sync_session()` and query: `select(Opportunity.id).where(Opportunity.deadline < cutoff_date)`
5. If empty: log + return `{"expired_opportunity_count": 0, "cascaded_guide_count": 0}` (no Redis call)
6. Otherwise: bulk `update(Opportunity).where(id.in_(expired_ids)).values(deleted_at=now_utc)`
7. Bulk `update(SubmissionGuide).where(opportunity_id.in_(expired_ids), deleted_at.is_(None)).values(deleted_at=now_utc)`
8. `session.commit()` (atomic transaction)
9. Call `_publish_expired_event(expired_id_strs, correlation_id, bound_log)` (best-effort, single attempt)
10. Log at INFO with `task_name`, `retention_days`, `correlation_id`, `expired_opportunity_count`, `cascaded_guide_count`

**Task 2: `_publish_expired_event` helper** (in same file)

Must implement:
- Guard: `if not expired_id_strs: return` (no empty-payload event)
- Build E01 envelope: `event_id` (UUID4), `event_type="OpportunitiesExpired"`, `payload` (JSON with `expired_ids` + `expired_at`), `timestamp`, `correlation_id`, `source_service="data-pipeline"`, `tenant_id=""`
- Sync Redis client: `redis.Redis.from_url(redis_url, decode_responses=True)`
- `r.xadd("eu-solicit:opportunities", envelope)` — best-effort, single attempt (no retry)
- Log success at INFO or failure at ERROR

**Task 3: Update `celery_app.py`**

```python
# Replace "data_pipeline.workers.tasks.cleanup" with:
"data_pipeline.workers.tasks.cleanup_expired",
```

### Key implementation details
- Use `structlog.get_logger(__name__)` and `log.bind(task_name=..., retention_days=..., correlation_id=...)` pattern
- The Opportunity model's `deleted_at IS NULL` default criterion applies to `select()` but NOT to `update()` (UPDATE bypasses the session event listener)
- Use `execution_options(synchronize_session="fetch")` on both UPDATE statements
- The sync Redis client pattern must match `publish_event.py`: `redis.Redis.from_url(os.environ.get("REDIS_URL", os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/0")), decode_responses=True)`
- `payload` field must be `json.dumps({"expired_ids": [...], "expired_at": "..."})` (serialized string, not dict)

---

## Next Steps (TDD Green Phase)

After implementing S05.10 Tasks 1–3:

1. Run unit tests: `cd eusolicit-app/services/data-pipeline && python -m pytest tests/unit/test_cleanup_expired.py -v`
2. Run integration tests: `python -m pytest tests/integration/test_cleanup_expired.py -v` (requires Redis + PostgreSQL testcontainer)
3. All 12 tests should pass (green phase)
4. If any fail: fix implementation (task bug) or fix test (test bug)
5. Commit passing tests with implementation

### Recommended follow-on workflows
- `/bmad-testarch-automate` — expand coverage beyond the ATDD P1/P2 scenarios
- `/bmad-testarch-ci` — ensure `test_cleanup_expired` is included in CI pipeline stages

---

## Validation Checklist (Step 5)

- [x] Both test files created at correct paths
- [x] All 12 tests will fail with ImportError (RED phase — `cleanup_expired` module missing)
- [x] All tests call `_require_cleanup_module()` at top (descriptive RED-phase failure)
- [x] No placeholder assertions (all assertions target expected behavior)
- [x] `_set_test_redis_url` + `_clean_opportunities_stream` autouse fixtures replicated from `test_publish_event.py`
- [x] `_get_conn(pg_container)` helper pattern consistent with existing integration tests
- [x] Unit tests mock at module boundary (`data_pipeline.workers.tasks.cleanup_expired.*`)
- [x] Integration tests use psycopg2 for raw reads (bypasses ORM soft-delete filter)
- [x] All 8 acceptance criteria covered by at least one test
- [x] Test IDs traceable to epic test design (E05-P1-025/026/027, E05-P2-011/012)
- [x] Priority assignments consistent with epic test design (E05-P1 → P1, E05-P2 → P2)
- [x] No CLI browser sessions to clean up (backend-only stack)
- [x] ATDD checklist saved to `eusolicit-docs/test-artifacts/`

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
