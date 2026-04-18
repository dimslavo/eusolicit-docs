# Story 5.9: Redis Streams Event Publishing

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer implementing the data pipeline**,
I want **a `publish_ingested_event` Celery task, chained from `generate_submission_guides` after guide dispatch, that publishes an `OpportunitiesIngested` event to the `eu-solicit:opportunities` Redis Stream using the E01 event-envelope format (event_id, event_type, JSON-serialised payload, timestamp, correlation_id, source_service, tenant_id), with the payload containing `crawler_type`, `run_id`, `opportunity_ids`, `timestamp`, and `summary` (counts of new/updated/unchanged), with Celery-level isolated retry on XADD failure (max 3, exponential back-off), and with ERROR-level logging plus `crawler_runs.errors` JSONB update on exhausted retries**,
so that **downstream services receive real-time notification of newly ingested opportunities via the shared event bus, the publish step can be retried in isolation without re-running the expensive crawl-normalize-score-guide chain, and any permanent publish failures are observable and recoverable through the `crawler_runs` audit record**.

## Acceptance Criteria

1. Event published to the `eu-solicit:opportunities` Redis Stream with the full E01 envelope format: `event_id` (UUID4), `event_type="OpportunitiesIngested"`, `payload` (JSON string), `timestamp` (UTC ISO), `correlation_id` (set to `run_id` for end-to-end traceability), `source_service="data-pipeline"`, `tenant_id=""`
2. The deserialized `payload` contains all five required fields: `crawler_type` (str), `run_id` (str), `opportunity_ids` (list of UUID strings), `timestamp` (UTC ISO string), and `summary` dict with `new`, `updated`, and `unchanged` integer keys
3. An integration test Redis client can read the event from `eu-solicit:opportunities` after the task completes and deserialize `opportunity_ids` back to a Python list (verifying E05-P0-007)
4. Publish failure (simulated XADD error) triggers Celery isolated retry without re-executing the crawl; `crawler_runs` record status is `"completed"` before any publish retry attempt begins (verifying E05-P0-008)
5. After all retries are exhausted, log at `ERROR` level with structured fields `run_id`, `stream_name`, and `opportunity_ids_count`; update `crawler_runs.errors` JSONB with `{"stream_publish_failure": {"stream": ..., "run_id": ..., "opportunity_ids_count": N, "error": "..."}}` for compensating recovery
6. `summary.new + summary.updated + summary.unchanged == crawler_runs.found` for the corresponding `run_id` (verifying E05-P1-024); `unchanged = found - new_count - updated`
7. `publish_ingested_event` is automatically dispatched from `generate_submission_guides`: after `guide_group.apply_async()`, call `publish_ingested_event.apply_async(args=[crawl_result])` — fires only on the success path, never inside an exception handler
8. All structured log lines emitted by `publish_ingested_event` include `task_name`, `run_id`, and `correlation_id` fields

## Tasks / Subtasks

- [ ] Task 1: Create `publish_ingested_event` Celery task (AC: 1, 2, 4, 5, 8)
  - [ ] 1.1 Create `services/data-pipeline/src/data_pipeline/workers/tasks/publish_event.py`. Implement:
    ```python
    @shared_task(
        name="pipeline.publish_ingested_event",
        bind=True,
        max_retries=3,
        default_retry_delay=5,
        retry_backoff=True,
        queue="pipeline_crawl",
    )
    def publish_ingested_event(self, crawl_result: dict[str, Any]) -> dict[str, Any]:
    ```
    Steps:
    - Extract fields: `run_id = crawl_result.get("run_id", "")`, `crawler_type = crawl_result.get("crawler_type", "unknown")`, `opportunity_ids: list[str] = crawl_result.get("opportunity_ids", [])`, `found = crawl_result.get("found", 0)`, `new_count = crawl_result.get("new_count", 0)`, `updated = crawl_result.get("updated", 0)`.
    - Bind structlog context: `task_name="pipeline.publish_ingested_event"`, `run_id=run_id`, `correlation_id=run_id`.
    - Compute `unchanged = max(0, found - new_count - updated)`.
    - Build event payload:
      ```python
      payload = {
          "crawler_type": crawler_type,
          "run_id": run_id,
          "opportunity_ids": opportunity_ids,
          "timestamp": datetime.now(UTC).isoformat(),
          "summary": {"new": new_count, "updated": updated, "unchanged": unchanged},
      }
      ```
    - Build E01 envelope (sync, no asyncio):
      ```python
      envelope = {
          "event_id": str(uuid4()),
          "event_type": "OpportunitiesIngested",
          "payload": json.dumps(payload),
          "timestamp": datetime.now(UTC).isoformat(),
          "correlation_id": run_id,
          "source_service": "data-pipeline",
          "tenant_id": "",
      }
      ```
    - Obtain a sync Redis client:
      ```python
      redis_url = os.environ.get("REDIS_URL", os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/0"))
      r = redis.Redis.from_url(redis_url, decode_responses=True)
      ```
    - Call `XADD` in a try/except:
      ```python
      try:
          stream = "eu-solicit:opportunities"
          message_id = r.xadd(stream, envelope)
          bound_log.info(
              "publish_ingested_event.published",
              stream=stream,
              message_id=message_id,
              opportunity_count=len(opportunity_ids),
          )
          return {"message_id": message_id, "stream": stream, "run_id": run_id}
      except Exception as exc:
          bound_log.warning(
              "publish_ingested_event.xadd_failed",
              error=str(exc),
              attempt=self.request.retries + 1,
          )
          try:
              raise self.retry(exc=exc)
          except MaxRetriesExceededError:
              # All retries exhausted — log ERROR and update crawler_runs
              bound_log.error(
                  "publish_ingested_event.retries_exhausted",
                  stream=stream,
                  opportunity_ids_count=len(opportunity_ids),
                  error=str(exc),
              )
              _record_publish_failure(run_id, stream, opportunity_ids, str(exc))
              return {"message_id": None, "stream": stream, "run_id": run_id, "error": str(exc)}
      finally:
          r.close()
      ```

  - [ ] 1.2 Implement `_record_publish_failure(run_id: str, stream: str, opportunity_ids: list[str], error: str) -> None` as a module-level helper. Opens a sync DB session, loads the `CrawlerRun` by `run_id`, and merges `{"stream_publish_failure": {"stream": stream, "run_id": run_id, "opportunity_ids_count": len(opportunity_ids), "error": error}}` into `crawler_runs.errors` JSONB. Wraps the DB update in a broad `try/except` — if this secondary write also fails, log at `ERROR` level with the error details and return without raising (the original publish failure is already logged).

- [ ] Task 2: Chain `publish_ingested_event` from `generate_submission_guides` (AC: 7)
  - [ ] 2.1 Edit `services/data-pipeline/src/data_pipeline/workers/tasks/generate_submission_guides.py`:
    - Add import at top: `from data_pipeline.workers.tasks.publish_event import publish_ingested_event`
    - In `generate_submission_guides`, after `guide_group.apply_async()` and before `bound_log.info("generate_submission_guides.dispatched", ...)`, add:
      ```python
      publish_ingested_event.apply_async(args=[crawl_result])
      bound_log.debug(
          "generate_submission_guides.event_publish_dispatched",
          run_id=crawl_result.get("run_id"),
      )
      ```
    - The call must be outside any exception handler — fires only on the success path.

- [ ] Task 3: Add `crawler_type` to the crawl result dict so the publisher can include it (AC: 2)
  - [ ] 3.1 Check `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py`: confirm `result` dict contains `"crawler_type": "aop"`. If absent, add `"crawler_type": "aop"` to the `result` dict before calling `score_opportunities.apply_async(args=[result])`.
  - [ ] 3.2 Check `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py`: confirm or add `"crawler_type": "ted"` to the result dict.
  - [ ] 3.3 Check `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py`: confirm or add `"crawler_type": "eu_grants"` to the result dict.

- [ ] Task 4: Register task and queue routing in Celery app (AC: 4)
  - [ ] 4.1 Edit `services/data-pipeline/src/data_pipeline/workers/celery_app.py`:
    - Add `"data_pipeline.workers.tasks.publish_event"` to the `include` list.
    - Add to `task_routes`:
      ```python
      "pipeline.publish_ingested_event": {"queue": "pipeline_crawl"},
      ```
    - This routes the publish task to the same `pipeline_crawl` queue as crawlers, keeping it separated from scoring/guide tasks. No new worker process required.

- [ ] Task 5: Write unit tests (AC: 1, 2, 4, 5, 6, 8)
  - [ ] 5.1 Create `services/data-pipeline/tests/unit/test_publish_event.py`. Use `eager_celery` fixture (Celery `task_always_eager=True`). Mock Redis client at the `redis.Redis.from_url` import level.
    - **`test_publish_ingested_event_happy_path`** — mock `redis.Redis.from_url` to return a mock client whose `xadd` returns `"1716000000000-0"`; call `publish_ingested_event.apply(args=[{"run_id": "abc", "crawler_type": "aop", "found": 10, "new_count": 8, "updated": 2, "opportunity_ids": ["id-1", "id-2"]}])`; assert `result.result["message_id"] == "1716000000000-0"`; assert `xadd` was called once with stream `"eu-solicit:opportunities"`; assert envelope contains `event_type="OpportunitiesIngested"`, `source_service="data-pipeline"`, `correlation_id="abc"`.
    - **`test_publish_ingested_event_payload_fields`** — call with full crawl_result; deserialize `json.loads(envelope["payload"])` from the xadd call kwargs; assert all five required fields present (`crawler_type`, `run_id`, `opportunity_ids`, `timestamp`, `summary`); assert `summary == {"new": 8, "updated": 2, "unchanged": 0}`.
    - **`test_publish_summary_unchanged_calculated`** — call with `found=10, new_count=3, updated=4`; assert `summary["unchanged"] == 3`.
    - **`test_publish_summary_matches_crawler_runs`** — assert `summary["new"] + summary["updated"] + summary["unchanged"] == found` for various (found, new_count, updated) combinations via parametrize.
    - **`test_publish_ingested_event_logs_task_name_run_id_correlation_id`** — use `caplog` or mock structlog; trigger the task; assert log record contains `task_name="pipeline.publish_ingested_event"`, `run_id`, `correlation_id`.
    - **`test_publish_ingested_event_retries_on_xadd_failure`** — with `eager_celery_no_propagate`; mock `xadd` to raise `redis.ConnectionError("down")`; call task; assert task was retried (verify `self.retry` called or exception propagated); assert `crawler_runs.status` is not changed (publish retry is isolated).

- [ ] Task 6: Write integration tests (AC: 1, 2, 3, 4, 5, 6)
  - [ ] 6.1 Create `services/data-pipeline/tests/integration/test_publish_event.py`. Use session-scoped `pg_container` fixture and `eager_celery` from `conftest.py`. For Redis, use a `redis.Redis` connection from `CELERY_BROKER_URL` (available in test environment) or mock `redis.Redis.from_url` at module level for isolation.
    - **E05-P0-007 `test_publish_event_happy_path_event_on_stream`** — seed a `CrawlerRun` record with `status="completed"`, `found=5`, `new_count=3`, `updated=2` via psycopg2; call `publish_ingested_event.apply(args=[{"run_id": str(run_id), "crawler_type": "aop", "found": 5, "new_count": 3, "updated": 2, "opportunity_ids": ["id-1", "id-2", "id-3"]}])`; read from the Redis stream using `r.xrevrange("eu-solicit:opportunities", count=1)`; assert a message was returned; parse `json.loads(msg["payload"])`; assert `payload["crawler_type"] == "aop"`; assert `payload["run_id"] == str(run_id)`; assert `isinstance(payload["opportunity_ids"], list)` and `len(payload["opportunity_ids"]) == 3`; assert `payload["summary"]["new"] == 3`; assert `payload["summary"]["updated"] == 2`; assert `payload["summary"]["unchanged"] == 0`.
    - **E05-P0-008 `test_publish_failure_triggers_isolated_retry_not_crawl_rerun`** — seed a `CrawlerRun` record with `status="completed"`; patch `redis.Redis.from_url` at `data_pipeline.workers.tasks.publish_event` to raise `redis.ConnectionError` on `xadd`; configure `eager_celery_no_propagate`; call `publish_ingested_event.apply(args=[...])`; assert the `CrawlerRun` record still has `status="completed"` (crawl was NOT re-run); assert no new `CrawlerRun` rows created for `crawler_type="aop"`.
    - **E05-P1-024 `test_publish_summary_matches_crawler_runs`** — seed a `CrawlerRun` with `found=12, new_count=7, updated=3`; call task with matching counts; read event from stream; assert `summary["new"] + summary["updated"] + summary["unchanged"] == 12`.
    - **`test_publish_exhausted_retries_updates_crawler_runs_errors`** — configure `maxRetries=0` via monkeypatch or use a wrapper that calls `_record_publish_failure` directly; seed `CrawlerRun` with `errors={}`; call `_record_publish_failure(run_id, "eu-solicit:opportunities", ["id-1"], "connection refused")`; query `crawler_runs.errors`; assert `errors["stream_publish_failure"]["opportunity_ids_count"] == 1`; assert `errors["stream_publish_failure"]["error"]` is non-empty.

## Dev Notes

### Architecture Context

S05.09 is the **final step of the pipeline enrichment chain**, firing after guide generation is dispatched from `generate_submission_guides`. The chain at this point is:

```
Beat → crawl_{source} (pipeline_crawl queue)
           └── calls score_opportunities.apply_async([result])
                   ├── dispatches group(_score_single_opportunity) → pipeline_scoring queue
                   └── calls generate_submission_guides.apply_async([crawl_result])
                           ├── dispatches group(_generate_single_guide) → pipeline_guides queue
                           └── calls publish_ingested_event.apply_async([crawl_result])
                                   └── XADD eu-solicit:opportunities
                                   └── (on failure) Celery retry (isolated, max 3)
                                   └── (retries exhausted) ERROR log + crawler_runs.errors update
```

The publish task fires **after guides are dispatched** (fire-and-forget), not after they complete. This is intentional: guide generation and scoring are concurrent enrichment steps that may take minutes; consumers should learn about newly ingested opportunities without waiting for enrichment to finish. The `opportunity_ids` list in the event payload covers all upserted IDs regardless of enrichment status.

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.09]
[Source: eusolicit-docs/implementation-artifacts/5-8-submission-guide-generation-task.md#Architecture-Context]

### CRITICAL: At-Least-Once Semantics — Publish Retry Must Be Isolated (AC: 4)

S05.09 AC explicitly states: "if publishing fails, the task retries the publish step only (data is already persisted)." The `crawl_result` dict is the same object that was already returned by `crawl_aop`/`crawl_ted`/`crawl_eu_grants` — data is in PostgreSQL before the publish task even runs. Therefore:

1. `crawl_aop.result` dict passed through `score_opportunities → generate_submission_guides → publish_ingested_event` is **read-only from the publisher's perspective**
2. Publishing failure must never trigger a crawl re-run
3. `crawler_runs.status` must remain `"completed"` during all publish retry attempts — do NOT change its status during retry
4. Only update `crawler_runs.errors` on **exhausted** retries (not on each individual retry attempt)

Incorrect pattern (never do this):
```python
# ❌ This would re-run the crawl on XADD failure
crawl_aop.apply_async()
```

Correct pattern (per AC4, E05-P0-008):
```python
# ✅ Only the publish step retries
publish_ingested_event.apply_async(args=[already_computed_result])
# Celery handles retry via raise self.retry(exc=exc)
```

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.09]
[Source: test-design-epic-05.md#E05-P0-008]

### CRITICAL: Use Sync Redis Client — Do Not Use `asyncio.run()` (E05 Design Constraint)

All Celery tasks in the data-pipeline service use **synchronous** execution. The `EventPublisher` from `eusolicit_common.events` uses `redis.asyncio` and is therefore async-only. Do NOT call `asyncio.run(publisher.publish(...))` in a Celery task — this is unsafe under gevent/eventlet worker pools (raises `RuntimeError` if an event loop is already running; see deferred-work.md S11.10 note).

Instead, use the synchronous `redis.Redis` client directly and replicate the E01 envelope format:

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
    "event_type": "OpportunitiesIngested",
    "payload": json.dumps(payload),       # JSON string, matches EventPublisher format
    "timestamp": datetime.now(UTC).isoformat(),
    "correlation_id": run_id,             # run_id doubles as correlation_id
    "source_service": "data-pipeline",
    "tenant_id": "",
}
message_id = r.xadd("eu-solicit:opportunities", envelope)
r.close()
```

This is byte-for-byte compatible with what `EventPublisher.publish()` produces — consumers expecting the async format can deserialize the envelope identically.

[Source: eusolicit-docs/implementation-artifacts/1-5-redis-streams-event-bus-setup.md#EventPublisher]
[Source: eusolicit-docs/implementation-artifacts/deferred-work.md#Deferred-from-code-review-of-story-11-10]

### Event Envelope Format (Must Match E01 `EventPublisher.publish()`)

The sync Redis `XADD` call must produce an envelope that matches the `EventPublisher` output format defined in `packages/eusolicit-common/src/eusolicit_common/events/publisher.py`:

```python
# EventPublisher output (for reference — async version)
envelope = {
    "event_id": str(uuid.uuid4()),          # ← new UUID per event
    "event_type": event_type,               # ← "OpportunitiesIngested"
    "payload": json.dumps(payload),         # ← JSON string of the inner dict
    "timestamp": datetime.now(UTC).isoformat(),
    "correlation_id": correlation_id or str(uuid.uuid4()),
    "source_service": source_service,       # ← "data-pipeline"
    "tenant_id": tenant_id or "",           # ← "" for pipeline (no tenant context)
}
```

The inner `payload` (after JSON-deserializing the `payload` field) must contain:

```json
{
  "crawler_type": "aop",
  "run_id": "uuid-...",
  "opportunity_ids": ["uuid-1", "uuid-2"],
  "timestamp": "2026-04-17T10:00:00+00:00",
  "summary": {
    "new": 8,
    "updated": 2,
    "unchanged": 0
  }
}
```

Consumer test validation (E05-P0-007):
```python
messages = r.xrevrange("eu-solicit:opportunities", count=1)
assert messages, "No events on stream"
msg_id, msg_data = messages[0]
envelope = msg_data  # dict with string keys/values (decode_responses=True)
assert envelope["event_type"] == "OpportunitiesIngested"
assert envelope["source_service"] == "data-pipeline"
inner = json.loads(envelope["payload"])
assert inner["crawler_type"] == "aop"
assert isinstance(inner["opportunity_ids"], list)
assert all(isinstance(oid, str) for oid in inner["opportunity_ids"])
assert inner["summary"]["new"] + inner["summary"]["updated"] + inner["summary"]["unchanged"] == inner["summary"].get("found", found)
```

### `crawler_type` Field in crawl_result

The `publish_ingested_event` task reads `crawler_type` from the `crawl_result` dict. Verify that each crawler task populates this field:

```python
# Expected crawl_result structure passed through the chain:
result = {
    "run_id": str(run_id),          # ← already present in crawl_aop, crawl_ted, crawl_eu_grants
    "found": len(normalized_records),
    "new_count": new_count,
    "updated": updated_count,
    "opportunity_ids": opportunity_ids,
    "crawler_type": "aop",          # ← Task 3 ensures this is populated
}
```

If `crawler_type` is missing from an existing result dict, `crawl_result.get("crawler_type", "unknown")` will fall back to `"unknown"` — this is acceptable for safety but must be avoided in production by completing Task 3.

### `_record_publish_failure` Helper — Error Recording on Exhausted Retries (AC: 5)

```python
def _record_publish_failure(
    run_id: str,
    stream: str,
    opportunity_ids: list[str],
    error: str,
) -> None:
    """Update crawler_runs.errors JSONB with publish failure metadata.
    
    Called only after all publish retries are exhausted.
    Does NOT change crawler_runs.status (must remain 'completed').
    Wraps DB write in try/except — a secondary failure here must not
    mask the original ERROR log.
    """
    from data_pipeline.db import get_sync_session
    from data_pipeline.models.crawler_run import CrawlerRun
    import uuid as _uuid
    
    try:
        run_uuid = _uuid.UUID(run_id)
    except (ValueError, AttributeError):
        log.error("publish_ingested_event.invalid_run_id", run_id=run_id)
        return
    
    try:
        with get_sync_session() as session:
            run_obj = session.get(CrawlerRun, run_uuid)
            if run_obj:
                existing_errors = run_obj.errors or {}
                existing_errors["stream_publish_failure"] = {
                    "stream": stream,
                    "run_id": run_id,
                    "opportunity_ids_count": len(opportunity_ids),
                    "error": error,
                }
                run_obj.errors = existing_errors
                session.commit()
    except Exception as db_exc:  # noqa: BLE001
        log.error(
            "publish_ingested_event.error_record_failed",
            run_id=run_id,
            db_error=str(db_exc),
        )
```

Note: `CrawlerRun.errors` is typed as `Mapped[dict[str, Any]]` with `server_default="{}"` — always safe to merge into.

[Source: eusolicit-docs/implementation-artifacts/5-1-pipeline-schema-migration-and-model-layer.md#CrawlerRun]

### File Locations

| Purpose | Path |
|---------|------|
| New publish event task module | `services/data-pipeline/src/data_pipeline/workers/tasks/publish_event.py` |
| Generate submission guides (add chain call in Task 2) | `services/data-pipeline/src/data_pipeline/workers/tasks/generate_submission_guides.py` |
| Celery app (add task module + routing in Task 4) | `services/data-pipeline/src/data_pipeline/workers/celery_app.py` |
| Crawl tasks (add `crawler_type` to result dict in Task 3) | `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py`, `crawl_ted.py`, `crawl_eu_grants.py` |
| CrawlerRun model (read/update for `_record_publish_failure`) | `services/data-pipeline/src/data_pipeline/models/crawler_run.py` |
| Sync DB session | `services/data-pipeline/src/data_pipeline/db.py` |
| Unit tests | `services/data-pipeline/tests/unit/test_publish_event.py` |
| Integration tests | `services/data-pipeline/tests/integration/test_publish_event.py` |
| conftest fixtures (reuse `pg_container`, `eager_celery`, `eager_celery_no_propagate`, `_psycopg2_conn`) | `services/data-pipeline/tests/conftest.py` |

### Redis Connection in Tests

The integration tests need a real Redis connection to verify `XADD` and `XREVRANGE`. The Celery conftest uses `redis://localhost:6379/1` (`TEST_REDIS_URL`) as the broker; use the same URL for the publisher in tests:

```python
# In integration tests — override REDIS_URL to point at the test Redis DB
@pytest.fixture(autouse=True)
def _set_test_redis_url(monkeypatch):
    test_url = os.environ.get("TEST_REDIS_URL", "redis://localhost:6379/1")
    monkeypatch.setenv("REDIS_URL", test_url)

# After each test — clean the opportunities stream to avoid cross-test pollution
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

For unit tests, mock `redis.Redis.from_url` at the `data_pipeline.workers.tasks.publish_event` module level using `unittest.mock.patch`.

### Test Design Traceability

Tests in this story directly verify the following scenarios from the Epic 5 test design:

| Test ID | Priority | Scenario |
|---------|----------|----------|
| E05-P0-007 | P0 | `opportunities.ingested` event published after AOP crawl cycle with `crawler_type`, `run_id`, `opportunity_ids`, `timestamp`, `summary` |
| E05-P0-008 | P0 | Redis XADD failure triggers isolated publish retry — `crawler_runs` already `completed`, crawl not re-executed |
| E05-P1-024 | P1 | `summary.new + summary.updated + summary.unchanged == crawler_runs.found` cross-validation |

Mitigations verified:

- **E05-R-003** (Redis Streams publish-or-lose, Score 6): `XADD` wrapped in try/except with Celery retry (max 3, isolated from crawl); on exhausted retries, ERROR log with `run_id` and `opportunity_ids_count`, plus `crawler_runs.errors` update for compensating recovery. E05-P0-007 and E05-P0-008 must both pass before sprint close.

Deferred mitigations (handled by other stories):
- **E05-R-001** (Dedup race): owned by S05.01/S05.04–06; publisher only reads already-upserted `opportunity_ids`
- **E05-R-002** (AI Gateway cascade / orphaned runs): owned by crawler tasks and S05.11; publisher runs after crawl is already `completed`
- **E05-R-005** (Celery chord starvation): `publish_ingested_event` routes to `pipeline_crawl` queue — no additional queue separation needed since it runs after the scoring/guide groups are dispatched

[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#E05-R-003]
[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#E05-P0-007]
[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#E05-P0-008]

---

## Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-17 | BMad Orchestrator (bmad-create-story) | Story file created from epic-05 S05.09 spec and test-design-epic-05.md |
