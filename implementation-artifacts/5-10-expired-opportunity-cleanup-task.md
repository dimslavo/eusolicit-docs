# Story 5.10: Expired Opportunity Cleanup Task

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer implementing the data pipeline**,
I want **a `cleanup_expired_opportunities` weekly Celery Beat task that queries `pipeline.opportunities` for records where `deadline < now() - retention_period` (default 30 days, configurable via `OPPORTUNITY_RETENTION_DAYS` env var) and `deleted_at IS NULL`, soft-deletes those records by setting `deleted_at = now()`, cascades the soft-delete to related `pipeline.submission_guides` records in the same DB transaction, logs the counts of affected opportunities and guides at INFO level, and publishes an `opportunities.expired` event to the `eu-solicit:opportunities` Redis Stream using the E01 sync envelope format with the list of expired opportunity IDs**,
so that **expired opportunities are automatically hidden from all standard model queries without permanent data loss, related guides are cleaned up in sync to avoid orphaned records, downstream services are notified of expired IDs for cache invalidation or archival, and the database does not accumulate indefinitely-growing sets of stale records that inflate scoring, guide-generation, and analytics queries**.

## Acceptance Criteria

1. All `pipeline.opportunities` records where `deadline < now() - OPPORTUNITY_RETENTION_DAYS` AND `deleted_at IS NULL` have `deleted_at` set to the current UTC timestamp after the task completes
2. All `pipeline.submission_guides` records whose `opportunity_id` is in the expired set AND `deleted_at IS NULL` also have `deleted_at` set in the same database transaction as the opportunity soft-delete (cascade soft-delete, verified by E05-P1-026)
3. `OPPORTUNITY_RETENTION_DAYS` environment variable controls the retention window; defaults to `30` when absent; overriding it requires no code change (verified by E05-P2-011)
4. An `opportunities.expired` event is published to the `eu-solicit:opportunities` Redis Stream using the E01 sync envelope format: `event_id` (UUID4), `event_type="OpportunitiesExpired"`, `payload` (JSON string), `timestamp` (UTC ISO), `correlation_id` (task correlation ID), `source_service="data-pipeline"`, `tenant_id=""`; the deserialized `payload` contains `expired_ids` (list of UUID strings) and `expired_at` (UTC ISO timestamp) (verified by E05-P1-027)
5. When there are zero expired opportunities, the task exits after logging `expired_opportunity_count=0` and does **not** publish a Redis event — no empty-payload event appears on the stream
6. After the task runs, the default SQLAlchemy `Opportunity` model query (with its `deleted_at IS NULL` default criterion) returns zero rows for the soft-deleted records — they are invisible to all standard application queries (verified by E05-P2-012)
7. The counts of soft-deleted opportunities and cascaded guides are both logged at INFO level with structured fields `expired_opportunity_count` and `cascaded_guide_count`
8. All structured log lines emitted by the task include `task_name`, `retention_days`, and `correlation_id` fields

## Tasks / Subtasks

- [x] Task 1: Implement `cleanup_expired_opportunities` Celery task (AC: 1, 2, 3, 5, 7, 8)
  - [x] 1.1 Create `services/data-pipeline/src/data_pipeline/workers/tasks/cleanup_expired.py`. Implement:
    ```python
    @shared_task(
        name="pipeline.cleanup_expired_opportunities",
        bind=True,
        queue="pipeline_crawl",
    )
    def cleanup_expired_opportunities(self) -> dict[str, Any]:
    ```
    Steps:
    - Read retention window: `retention_days = int(os.environ.get("OPPORTUNITY_RETENTION_DAYS", "30"))`.
    - Compute cutoff: `cutoff_date = datetime.now(UTC) - timedelta(days=retention_days)`.
    - Generate a task correlation ID: `correlation_id = str(uuid4())`.
    - Bind structlog context:
      ```python
      bound_log = log.bind(
          task_name="pipeline.cleanup_expired_opportunities",
          retention_days=retention_days,
          correlation_id=correlation_id,
      )
      bound_log.info("cleanup_expired_opportunities.started", cutoff_date=cutoff_date.isoformat())
      ```
    - Open a sync DB session via `get_sync_session()`:
      ```python
      with get_sync_session() as session:
          # Step 1: Find expired opportunity IDs (not yet soft-deleted)
          # Use with_loader_criteria bypass or explicit column filter to find active-but-expired rows.
          # Because the model's default criterion already applies deleted_at IS NULL,
          # a normal query on Opportunity already returns only non-deleted rows.
          # Filter further by deadline < cutoff_date.
          expired_ids_result = session.execute(
              select(Opportunity.id).where(Opportunity.deadline < cutoff_date)
          ).scalars().all()
          expired_ids: list[uuid.UUID] = list(expired_ids_result)
      ```
    - If `expired_ids` is empty:
      ```python
      bound_log.info(
          "cleanup_expired_opportunities.no_expired",
          expired_opportunity_count=0,
          cascaded_guide_count=0,
      )
      return {"expired_opportunity_count": 0, "cascaded_guide_count": 0}
      ```
    - Otherwise, within the same session, perform the batch soft-delete in two steps:
      ```python
      now_utc = datetime.now(UTC)

      # Step 2: Soft-delete expired opportunities (bulk UPDATE)
      session.execute(
          update(Opportunity)
          .where(Opportunity.id.in_(expired_ids))
          .values(deleted_at=now_utc)
          .execution_options(synchronize_session="fetch")
      )

      # Step 3: Cascade soft-delete to submission_guides in same transaction
      guide_result = session.execute(
          update(SubmissionGuide)
          .where(
              SubmissionGuide.opportunity_id.in_(expired_ids),
              SubmissionGuide.deleted_at.is_(None),
          )
          .values(deleted_at=now_utc)
          .execution_options(synchronize_session="fetch")
      )
      guide_count = guide_result.rowcount

      session.commit()
      ```
    - Build the list of expired ID strings: `expired_id_strs = [str(eid) for eid in expired_ids]`.
    - Log outcome:
      ```python
      bound_log.info(
          "cleanup_expired_opportunities.completed",
          expired_opportunity_count=len(expired_ids),
          cascaded_guide_count=guide_count,
      )
      ```
    - Publish the expired event (call helper defined in Task 2):
      ```python
      _publish_expired_event(
          expired_id_strs=expired_id_strs,
          correlation_id=correlation_id,
          bound_log=bound_log,
      )
      ```
    - Return `{"expired_opportunity_count": len(expired_ids), "cascaded_guide_count": guide_count}`.

- [x] Task 2: Implement `_publish_expired_event` helper (AC: 4, 5)
  - [x] 2.1 Add `_publish_expired_event` as a module-level helper in `cleanup_expired.py`:
    ```python
    def _publish_expired_event(
        expired_id_strs: list[str],
        correlation_id: str,
        bound_log: Any,
    ) -> None:
        """Publish OpportunitiesExpired event to Redis Stream using sync client.
        
        Uses the same E01 envelope format as publish_event.py.
        Best-effort: publish failure is logged at ERROR level but does not
        re-trigger the cleanup task.
        """
        if not expired_id_strs:
            return  # AC5: no empty-payload event

        payload = {
            "expired_ids": expired_id_strs,
            "expired_at": datetime.now(UTC).isoformat(),
        }
        envelope = {
            "event_id": str(uuid4()),
            "event_type": "OpportunitiesExpired",
            "payload": json.dumps(payload),
            "timestamp": datetime.now(UTC).isoformat(),
            "correlation_id": correlation_id,
            "source_service": "data-pipeline",
            "tenant_id": "",
        }
        redis_url = os.environ.get(
            "REDIS_URL",
            os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/0"),
        )
        r = redis.Redis.from_url(redis_url, decode_responses=True)
        stream = "eu-solicit:opportunities"
        try:
            message_id = r.xadd(stream, envelope)
            bound_log.info(
                "cleanup_expired_opportunities.event_published",
                stream=stream,
                message_id=message_id,
                expired_count=len(expired_id_strs),
            )
        except Exception as exc:  # noqa: BLE001
            bound_log.error(
                "cleanup_expired_opportunities.event_publish_failed",
                stream=stream,
                expired_count=len(expired_id_strs),
                error=str(exc),
            )
        finally:
            r.close()
    ```
    Note: Unlike `publish_ingested_event` (S05.09), the cleanup event is best-effort with a single attempt. The soft-delete is already committed; the event is informational. If publishing fails, the ERROR log is the recovery signal. Celery-level retry for the entire cleanup task is not appropriate since the DB write is already committed.

- [x] Task 3: Register task in Celery app and verify Beat schedule entry (AC: 1)
  - [x] 3.1 Edit `services/data-pipeline/src/data_pipeline/workers/celery_app.py`:
    - Add `"data_pipeline.workers.tasks.cleanup_expired"` to the `include` list.
    - Add to `task_routes`:
      ```python
      "pipeline.cleanup_expired_opportunities": {"queue": "pipeline_crawl"},
      ```
  - [x] 3.2 Verify the Beat schedule entry `cleanup-expired-opportunities` created in S05.02 references `pipeline.cleanup_expired_opportunities` as the task name. The entry should look like:
    ```python
    "cleanup-expired-opportunities": {
        "task": "pipeline.cleanup_expired_opportunities",
        "schedule": crontab(
            hour=int(os.environ.get("CLEANUP_HOUR", "4")),
            minute=0,
            day_of_week=os.environ.get("CLEANUP_DAY_OF_WEEK", "0"),  # Sunday = 0
        ),
    },
    ```
    If the Beat schedule was defined with a placeholder task name, update it to `"pipeline.cleanup_expired_opportunities"`.

- [x] Task 4: Write unit tests (AC: 1, 3, 5, 7, 8)
  - [x] 4.1 Create `services/data-pipeline/tests/unit/test_cleanup_expired.py`. Use `eager_celery` fixture (`task_always_eager=True`). Mock the DB session and Redis client at their import boundaries.
    - **`test_cleanup_no_expired_opportunities_no_event`** — mock `get_sync_session()` to return a session where `select(Opportunity.id).where(...)` yields an empty list; mock `redis.Redis.from_url`; call `cleanup_expired_opportunities.apply()`; assert `redis.Redis.from_url` was NOT called (no event published); assert result `expired_opportunity_count == 0`.
    - **`test_cleanup_soft_deletes_expired_rows`** — mock session returning 2 expired opportunity UUIDs; assert `session.execute` called with an `update(Opportunity)` statement that sets `deleted_at`; assert `session.commit()` called.
    - **`test_cleanup_cascades_soft_delete_to_guides`** — mock session returning 2 expired UUIDs; assert a second `update(SubmissionGuide)` statement was executed with `deleted_at` and the correct `opportunity_id.in_()` filter.
    - **`test_cleanup_retention_days_from_env_var`** — patch `OPPORTUNITY_RETENTION_DAYS=7`; capture the `where(Opportunity.deadline < cutoff_date)` call; assert `cutoff_date ≈ now() - timedelta(days=7)` (within 5s tolerance using `freezegun`).
    - **`test_cleanup_logs_required_fields`** — use `caplog` or structlog mock; trigger the task; assert log records contain `task_name="pipeline.cleanup_expired_opportunities"`, `retention_days`, and `correlation_id`.
    - **`test_cleanup_event_payload_structure`** — mock Redis; run task with 3 mocked expired IDs; capture `xadd` call args; parse `json.loads(envelope["payload"])`; assert `payload["expired_ids"]` is a list of 3 strings; assert `payload["expired_at"]` is a non-empty ISO timestamp string; assert `envelope["event_type"] == "OpportunitiesExpired"`; assert `envelope["source_service"] == "data-pipeline"`.

- [x] Task 5: Write integration tests (AC: 1, 2, 4, 5, 6)
  - [x] 5.1 Create `services/data-pipeline/tests/integration/test_cleanup_expired.py`. Use session-scoped `pg_container` and `eager_celery` fixtures from `conftest.py`. For Redis, use `TEST_REDIS_URL` with a `_clean_opportunities_stream` autouse fixture (same pattern as `test_publish_event.py`).
    - **E05-P1-025 `test_cleanup_soft_deletes_opportunities_past_retention`** — using `psycopg2` / `OpportunityFactory`, insert 3 opportunities:
      - `opp_1`: `deadline = now() - timedelta(days=60)` (expired, 60 days past deadline > 30-day retention → eligible)
      - `opp_2`: `deadline = now() - timedelta(days=45)` (expired, 45 days past deadline > 30 days → eligible)
      - `opp_3`: `deadline = now() - timedelta(days=10)` (recent, within retention window → not eligible)
      Call `cleanup_expired_opportunities.apply()`. Query `pipeline.opportunities` directly (bypassing model filter) using raw SQL; assert `deleted_at IS NOT NULL` for `opp_1` and `opp_2`; assert `deleted_at IS NULL` for `opp_3`.
    - **E05-P1-026 `test_cleanup_cascades_soft_delete_to_submission_guides`** — insert 1 expired opportunity (`deadline = now() - timedelta(days=60)`) and 1 `SubmissionGuide` row linked to it; call task; query `pipeline.submission_guides` directly; assert guide row has `deleted_at IS NOT NULL`; assert the opportunity itself also has `deleted_at IS NOT NULL`.
    - **E05-P1-027 `test_cleanup_publishes_expired_event_with_ids`** — insert 2 expired opportunities; call task; read stream using `r.xrevrange("eu-solicit:opportunities", count=1)`; assert a message exists; assert `msg["event_type"] == "OpportunitiesExpired"`; parse `json.loads(msg["payload"])`; assert `len(payload["expired_ids"]) == 2`; assert all IDs in `expired_ids` are valid UUID strings; assert `payload["expired_at"]` is a non-empty ISO timestamp.
    - **E05-P2-011 `test_cleanup_retention_period_configurable_via_env`** — patch `OPPORTUNITY_RETENTION_DAYS=7`; insert 1 opportunity at `deadline = now() - timedelta(days=20)` (past 7-day window but within 30-day default); insert 1 at `deadline = now() - timedelta(days=5)` (within 7-day window); call task; assert the 20-day-old opportunity has `deleted_at IS NOT NULL`; assert the 5-day-old opportunity is untouched.
    - **E05-P2-012 `test_soft_deleted_rows_excluded_from_model_queries`** — insert 2 opportunities (`deadline = now() - timedelta(days=60)`) + 1 active; call task; using the SQLAlchemy `Opportunity` model query (session-scoped fixture), assert only 1 row returned (the active one); assert the 2 expired IDs are not present in any model query results.
    - **`test_cleanup_no_expired_does_not_publish_event`** — insert 1 opportunity with `deadline = now() + timedelta(days=30)` (future deadline, never eligible); call task; assert `r.xlen("eu-solicit:opportunities") == 0` (no event published).

## Dev Notes

### Architecture Context

S05.10 is a **standalone weekly maintenance task** defined in the Beat schedule during S05.02. Unlike the crawl tasks (S05.04–06), the cleanup task has no upstream chain and no downstream tasks — it is fired directly by Beat, executes a DB batch update, and optionally publishes one Redis event. There is no AI Gateway involvement.

The pipeline task chain for context (cleanup is independent of the main chain):
```
Beat → crawl_{source} → score_opportunities → generate_submission_guides → publish_ingested_event
Beat → cleanup_expired_opportunities (independent, weekly)
                └── UPDATE pipeline.opportunities SET deleted_at = now()
                └── UPDATE pipeline.submission_guides SET deleted_at = now()  (cascade)
                └── XADD eu-solicit:opportunities (OpportunitiesExpired event)
```

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.10]
[Source: eusolicit-docs/implementation-artifacts/5-2-celery-app-bootstrap-and-beat-schedule-configuration.md]

### CRITICAL: Model Default Criterion and the Soft-Delete Query

The `Opportunity` SQLAlchemy model has a `deleted_at IS NULL` default criterion configured via `__mapper_args__` (implemented in S05.01). This means **any standard `select(Opportunity)` or `session.query(Opportunity)` call automatically filters out soft-deleted rows** — you will never accidentally retrieve deleted records through the ORM.

For the cleanup task this is correct behaviour for the _read_ phase: `select(Opportunity.id).where(Opportunity.deadline < cutoff_date)` only returns active (non-deleted) expired rows because the default criterion applies. No extra `AND deleted_at IS NULL` clause is needed in application code.

For the _write_ phase (bulk `UPDATE`), however, use `update(Opportunity).where(Opportunity.id.in_(expired_ids))` — this bypasses the ORM default criterion and updates by primary key. Since we already collected the IDs from a filtered query, no expired+already-deleted rows will be in the list.

For `SubmissionGuide`, the cascade update must explicitly add `SubmissionGuide.deleted_at.is_(None)` to avoid hitting any already-deleted guide rows (in case previous cleanup runs partially soft-deleted them). `SubmissionGuide` may or may not have the same default criterion — add the explicit filter regardless.

Incorrect pattern (never do this):
```python
# ❌ Application-level loop — N+1 queries, race-prone
for opp in expired_opps:
    opp.deleted_at = now_utc
session.commit()
```

Correct pattern (single atomic bulk UPDATE per table):
```python
# ✅ Atomic batch UPDATE by primary key list
session.execute(
    update(Opportunity)
    .where(Opportunity.id.in_(expired_ids))
    .values(deleted_at=now_utc)
    .execution_options(synchronize_session="fetch")
)
session.execute(
    update(SubmissionGuide)
    .where(
        SubmissionGuide.opportunity_id.in_(expired_ids),
        SubmissionGuide.deleted_at.is_(None),
    )
    .values(deleted_at=now_utc)
    .execution_options(synchronize_session="fetch")
)
session.commit()
```

[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#E05-R-006]
[Source: eusolicit-docs/implementation-artifacts/5-1-pipeline-schema-migration-and-model-layer.md]

### CRITICAL: Use Sync Redis Client — Same Pattern as S05.09

The cleanup task is a synchronous Celery task. Follow the exact same sync Redis pattern established in `publish_event.py` (S05.09). Do NOT call `asyncio.run()` or use `eusolicit_common.events.EventPublisher` (which is async-only and unsafe under gevent/eventlet worker pools).

```python
import json
import os
import redis
from datetime import UTC, datetime
from uuid import uuid4

redis_url = os.environ.get("REDIS_URL", os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/0"))
r = redis.Redis.from_url(redis_url, decode_responses=True)

envelope = {
    "event_id": str(uuid4()),
    "event_type": "OpportunitiesExpired",
    "payload": json.dumps({"expired_ids": expired_id_strs, "expired_at": now_iso}),
    "timestamp": datetime.now(UTC).isoformat(),
    "correlation_id": correlation_id,
    "source_service": "data-pipeline",
    "tenant_id": "",
}
r.xadd("eu-solicit:opportunities", envelope)
r.close()
```

[Source: eusolicit-docs/implementation-artifacts/5-9-redis-streams-event-publishing.md#CRITICAL-Use-Sync-Redis-Client]

### Event Envelope Format — `OpportunitiesExpired`

The `OpportunitiesExpired` event follows the same E01 envelope format as `OpportunitiesIngested` (S05.09), differing only in `event_type` and payload structure.

Outer envelope (fields set on the Redis stream entry):
```python
{
    "event_id": "uuid4-string",
    "event_type": "OpportunitiesExpired",
    "payload": "<JSON-string of inner payload>",   # Must be json.dumps(...)
    "timestamp": "2026-04-20T04:00:00.123456+00:00",
    "correlation_id": "task-uuid-or-run-id",
    "source_service": "data-pipeline",
    "tenant_id": "",
}
```

Inner `payload` (after `json.loads(envelope["payload"])`):
```json
{
    "expired_ids": ["uuid-1", "uuid-2", "uuid-3"],
    "expired_at": "2026-04-20T04:00:00.123456+00:00"
}
```

Consumer test validation (E05-P1-027):
```python
messages = r.xrevrange("eu-solicit:opportunities", count=1)
assert messages, "No events on stream"
msg_id, msg_data = messages[0]
assert msg_data["event_type"] == "OpportunitiesExpired"
assert msg_data["source_service"] == "data-pipeline"
inner = json.loads(msg_data["payload"])
assert isinstance(inner["expired_ids"], list)
assert all(isinstance(eid, str) for eid in inner["expired_ids"])
assert inner["expired_at"]  # non-empty ISO timestamp
```

### Best-Effort Event Publish — No Isolated Celery Retry (Design Difference from S05.09)

Unlike `publish_ingested_event` (S05.09), the cleanup task's event publish is **best-effort** without Celery-level isolated retry. The rationale:

1. The DB soft-delete is already committed by the time the publish runs — re-queuing the entire cleanup task would find zero records to delete (all `deleted_at` already set) and would just produce duplicate publish attempts
2. The `expired` event is informational (cache invalidation, archival triggers) rather than a primary data-flow event — a missed expired event does not cause data inconsistency in the pipeline itself
3. Recovery: the ERROR log containing `expired_count` and `correlation_id` is the recovery signal; an operator can manually query `WHERE deleted_at IS NOT NULL AND deleted_at > ?` to reconstruct the expired ID list

If stronger at-least-once semantics are needed in a later sprint, the approach is to commit soft-deletes and then immediately enqueue a dedicated `publish_expired_event` task (same isolation pattern as S05.09). This can be added without changing the DB schema.

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.10]

### Beat Schedule Verification (S05.02 Dependency)

The Beat schedule entry `cleanup-expired-opportunities` was defined in S05.02. Verify it uses the correct Celery task name `"pipeline.cleanup_expired_opportunities"`. The expected crontab is Sunday at 04:00 UTC:

```python
"cleanup-expired-opportunities": {
    "task": "pipeline.cleanup_expired_opportunities",
    "schedule": crontab(
        hour=int(os.environ.get("CLEANUP_HOUR", "4")),
        minute=0,
        day_of_week=os.environ.get("CLEANUP_DAY_OF_WEEK", "0"),  # Sunday
    ),
},
```

Unit test E05-P1-005 (S05.02) already asserts this entry appears in `app.conf.beat_schedule`. No new schedule definition is needed in this story — only confirm the task name is wired correctly.

[Source: eusolicit-docs/implementation-artifacts/5-2-celery-app-bootstrap-and-beat-schedule-configuration.md]

### File Locations

| Purpose | Path |
|---------|------|
| New cleanup task module | `services/data-pipeline/src/data_pipeline/workers/tasks/cleanup_expired.py` |
| Celery app (add task module + routing) | `services/data-pipeline/src/data_pipeline/workers/celery_app.py` |
| Opportunity model (read `deleted_at` default criterion) | `services/data-pipeline/src/data_pipeline/models/opportunity.py` |
| SubmissionGuide model (cascade delete target) | `services/data-pipeline/src/data_pipeline/models/submission_guide.py` |
| Sync DB session | `services/data-pipeline/src/data_pipeline/db.py` |
| Unit tests | `services/data-pipeline/tests/unit/test_cleanup_expired.py` |
| Integration tests | `services/data-pipeline/tests/integration/test_cleanup_expired.py` |
| conftest fixtures (reuse `pg_container`, `eager_celery`, `_psycopg2_conn`) | `services/data-pipeline/tests/conftest.py` |

### Redis Stream Cleanup in Tests

The integration tests use `eu-solicit:opportunities` for both `OpportunitiesIngested` (S05.09) and `OpportunitiesExpired` events. Use the same `_clean_opportunities_stream` autouse fixture established in `test_publish_event.py` to flush the stream before and after each test:

```python
@pytest.fixture(autouse=True)
def _set_test_redis_url(monkeypatch):
    test_url = os.environ.get("TEST_REDIS_URL", "redis://localhost:6379/1")
    monkeypatch.setenv("REDIS_URL", test_url)

@pytest.fixture(autouse=True)
def _clean_opportunities_stream(_set_test_redis_url):
    r = redis.Redis.from_url(os.environ["REDIS_URL"], decode_responses=True)
    try:
        r.delete("eu-solicit:opportunities")
    except Exception:
        pass
    finally:
        r.close()
    yield
    r2 = redis.Redis.from_url(os.environ["REDIS_URL"], decode_responses=True)
    try:
        r2.delete("eu-solicit:opportunities")
    finally:
        r2.close()
```

To read back raw (including soft-deleted) opportunity rows in integration tests, bypass the SQLAlchemy model using `psycopg2` directly:
```python
# Read raw rows including soft-deleted records (bypasses model default criterion)
_psycopg2_conn.cursor().execute(
    "SELECT id, deleted_at FROM pipeline.opportunities WHERE id = ANY(%s)",
    ([str(eid) for eid in expired_ids],)
)
rows = _psycopg2_conn.cursor().fetchall()
assert all(row[1] is not None for row in rows)  # all have deleted_at set
```

### Imports Required in `cleanup_expired.py`

```python
import json
import os
import uuid
from datetime import UTC, datetime, timedelta
from typing import Any
from uuid import uuid4

import redis
import structlog
from celery import shared_task
from sqlalchemy import select, update

from data_pipeline.db import get_sync_session
from data_pipeline.models.opportunity import Opportunity
from data_pipeline.models.submission_guide import SubmissionGuide

log = structlog.get_logger(__name__)
```

### Test Design Traceability

Tests in this story directly verify the following scenarios from the Epic 5 test design:

| Test ID | Priority | Scenario |
|---------|----------|----------|
| E05-P1-025 | P1 | Cleanup soft-deletes opportunities past deadline + retention (3 opps: 2 expired, 1 within window) |
| E05-P1-026 | P1 | Cascade soft-delete to `submission_guides` (opportunity + guide both have `deleted_at` set) |
| E05-P1-027 | P1 | `opportunities.expired` event published with affected IDs |
| E05-P2-011 | P2 | Retention period configurable via `OPPORTUNITY_RETENTION_DAYS=7` |
| E05-P2-012 | P2 | Default model query filter excludes soft-deleted rows after cleanup |

Mitigations verified:

- **E05-R-006** (Soft-delete filter bypass, Score 4): Cascade soft-delete implemented as a bulk UPDATE; `SubmissionGuide.deleted_at.is_(None)` guard prevents double-deletion. E05-P2-012 verifies that the model's default criterion excludes soft-deleted rows from all standard queries.

Deferred mitigations (handled by other stories):
- **E05-R-001** (Dedup race): owned by S05.01/S05.04–06; cleanup operates on already-upserted rows only
- **E05-R-002** (AI Gateway cascade): not applicable — cleanup makes no AI Gateway calls
- **E05-R-003** (Redis publish loss): informational event only; no isolated Celery retry required (see design rationale above)

[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#E05-P1-025]
[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#E05-P1-026]
[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#E05-P1-027]
[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#E05-R-006]

---

## Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-17 | BMad Orchestrator (bmad-create-story) | Story file created from epic-05 S05.10 spec and test-design-epic-05.md |
| 2026-04-17 | BMad Dev Agent (bmad-dev-story) | Implemented cleanup_expired.py (full task + _publish_expired_event helper); updated celery_app.py include list; updated test_ai_gateway_client.py to reflect cleanup no longer being a stub (no AI Gateway calls); all 96 unit tests + 30 integration tests pass |

## Known Deviations

### Detected by `3-code-review` at 2026-04-16T22:36:35Z (session c793ef3a-c55a-49f0-b246-faaa14905407)

- EnrichmentQueueItem rows for soft-deleted opportunities not cleaned up — potential orphaned enrichment work after cleanup` _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- EnrichmentQueueItem rows for soft-deleted opportunities not cleaned up — potential orphaned enrichment work after cleanup` _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
