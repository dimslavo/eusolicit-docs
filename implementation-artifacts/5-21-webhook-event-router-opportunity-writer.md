# Story 5.21: Webhook Event Router → Opportunity Writer

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As the **data-pipeline service**,
I want to **consume `sirmaai.workflow.completed` events from Redis Streams, upsert the normalised opportunity records they carry into `pipeline.opportunities`, and publish `opportunities.ingested` so downstream consumers continue to react in real time**,
so that **the SirmaAI N8N workflows (S05.20) become the authoritative ingestion path while preserving the existing `pipeline.opportunities` schema, `(source_id, source_type)` dedup contract, and `OpportunitiesIngested` event contract that the rest of the platform depends on**.

This is the **consumer side** of the E05 amendment cutover: S05.20 ships the producer (three N8N templates emitting `workflow.completed` via SirmaAI → S04.25 receiver → `sirmaai.workflow.completed` Redis Stream); S05.21 closes the loop by routing those events into the canonical opportunity store.

## Acceptance Criteria

1. **AC1 — Consumer wiring**: An async background task `run_workflow_event_consumer` runs inside the `data-pipeline` FastAPI lifespan, subscribed to the Redis Stream `sirmaai.workflow.completed` under the consumer group `data-pipeline:workflow-event-router` with consumer name `data-pipeline-router-1`. The group is created idempotently (`XGROUP CREATE … MKSTREAM`, swallowing `BUSYGROUP`). [Pattern source: `services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py`]
2. **AC2 — Event envelope parsing**: The consumer parses each message's outer Redis Streams fields (`webhook_id`, `subscription_id`, `event_type`, `received_at`, `payload`). `payload` is a JSON string containing the full SirmaAI Standard Webhooks envelope (`{type, timestamp, data}`). The consumer extracts `data.client_reference_id` (= eusolicit run UUID), `data.status`, `data.sirmaai_project_id`, `data.source_type` (one of `aop|ted|eu_grants`), and `data.opportunities` (list of normalised opportunity records).
3. **AC3 — Status gate**: Only events whose mapped status (per `sirmaai_gateway.services.workflow_run_repository.map_status`) equals `succeeded` proceed to the upsert path. Events with `failed`, `cancelled`, or any other terminal-but-not-successful status are acked and logged at INFO with no write. Events whose `event_type` field on the outer envelope is not `workflow.completed` are acked and skipped (defence-in-depth — the receiver already routes by type).
4. **AC4 — Idempotency via run-id dedup**: Before any write, the consumer attempts `SETNX data_pipeline:workflow_router:idempotency:{eusolicit_run_id} <iso_timestamp>` with a 7-day TTL via `EXPIRE`. If `SETNX` returns 0 (key already exists), the message is acked without writing and a `workflow_router.idempotent_skip` log line is emitted with `eusolicit_run_id` and `webhook_id`. This makes duplicate webhook delivery (Standard Webhooks at-least-once) a no-op.
5. **AC5 — Tenant resolution and cross-tenant guard**: Resolve `data.sirmaai_project_id` → `(company_id, provisioning_status)` via raw SQL `SELECT company_id, provisioning_status FROM client.sirmaai_projects WHERE sirmaai_project_id = :pid` (no ORM cross-import — precedent: `rate_limit_sync_consumer._resolve_project`). If the row does not exist OR `provisioning_status != 'provisioned'`, the event is DLQ'd with reason `unknown_or_unprovisioned_project` and acked. If `data.opportunities[*].company_id` (when present) does not match the resolved `company_id`, the event is DLQ'd with reason `cross_tenant_violation` — never written.
6. **AC6 — Atomic upsert via existing helper**: Successful events are upserted into `pipeline.opportunities` using the existing `upsert_opportunities(session, records, source_type)` helper at `data_pipeline.workers.tasks._upsert`. The helper provides single-statement `INSERT … ON CONFLICT (source_id, source_type) DO UPDATE … RETURNING id` semantics — DO NOT reimplement dedup. The returned `(new_count, updated_count, opportunity_ids)` tuple is captured for the downstream event.
7. **AC7 — Soft-delete preservation**: Upsert MUST NOT resurrect soft-deleted rows. The `_upsert.py` helper's `ON CONFLICT DO UPDATE` set columns do NOT touch `deleted_at`, so rows with `deleted_at IS NOT NULL` stay soft-deleted even on a re-crawl that produces the same `(source_id, source_type)`. Add an explicit unit test asserting this invariant.
8. **AC8 — Publish `opportunities.ingested`**: After successful upsert, publish an `OpportunitiesIngested` envelope to the Redis Stream `eu-solicit:opportunities` byte-for-byte compatible with `publish_event.publish_ingested_event` (S05.09). Envelope fields: `event_id` (uuid4), `event_type='OpportunitiesIngested'`, `payload` (JSON-encoded inner dict with `crawler_type` = `data.source_type`, `run_id` = `eusolicit_run_id`, `opportunity_ids` (list of UUIDs from upsert), `timestamp`, `summary={new, updated, unchanged}`), `timestamp`, `correlation_id` (= run_id), `source_service='data-pipeline'`, `tenant_id=''` (preserve legacy envelope shape for downstream parity).
9. **AC9 — Publish isolation**: The XADD on `eu-solicit:opportunities` is wrapped in try/except. On XADD failure (transient Redis error), the consumer does NOT ack the source message — leaving it in the PEL so the `retry_pending`/`process_pending` loop re-delivers it. The upsert is NOT re-executed because `SETNX` idempotency from AC4 already short-circuits the second delivery. Result: at-least-once event publish without duplicate writes.
10. **AC10 — DLQ via PEL**: Use `EventConsumer.process_pending` (from `eusolicit_common.events.consumer`) with `max_retries=3` and default `min_idle_ms=30_000` to move stuck messages to `sirmaai.workflow.completed.dlq` automatically after the PEL retry budget is exhausted. Do NOT create a new DB-backed DLQ table — the stream-based DLQ is sufficient and matches the precedent. Schema-parse failures (e.g. missing `data`, malformed JSON in `payload`) ack immediately after writing a `workflow_router.parse_error` log line — retrying malformed JSON will never succeed.
11. **AC11 — Loop resilience**: The consumer main loop mirrors `rate_limit_sync_consumer.run_rate_limit_sync_consumer`: `process_pending` → `retry_pending` (with `min_idle_ms=0` for event-driven reclaim) → `consume` with `block_ms=2000, count=10`. Loop-level `asyncio.CancelledError` re-raises; any other exception logs `workflow_router.loop_error` and `asyncio.sleep(5)` before continuing. NEVER let the consumer task die — a wedged loop blocks all ingestion.
12. **AC12 — Async session factory**: Add `data_pipeline/db_async.py` exporting `get_async_session_factory()` returning an `async_sessionmaker[AsyncSession]` bound to `DATABASE_URL` (converted to `postgresql+asyncpg://`). The session factory is lifespan-scoped (one engine per process); never share `AsyncSession` instances between iterations. Existing sync session helper (`db.py:get_sync_session`) stays untouched — Celery tasks still use it.
13. **AC13 — Lifespan wiring**: `data_pipeline/main.py` gains a FastAPI lifespan that (a) instantiates an async Redis client via `redis.asyncio.from_url(REDIS_URL)`, (b) calls `bootstrap_event_bus(redis)` (already imports from `eusolicit_common.events`), (c) launches `run_workflow_event_consumer` as an `asyncio.create_task`, and (d) on shutdown cancels the task and awaits it with `asyncio.wait_for(..., timeout=10)`. Add the new `data-pipeline:workflow-event-router` group to the canonical `CONSUMER_GROUPS` dict in `eusolicit_common.events.bootstrap` so it is created on every service startup.
14. **AC14 — Template payload extension**: The three N8N workflow templates shipped by S05.20 (`infra/n8n-templates/crawl-{aop,ted,eu-grants}-v1.json`) currently emit only `summary.opportunity_ids` and counts in their `Build Workflow Completed Event` node. Extend each template's Set node to additionally emit `data.opportunities` (the full normalised+scored record list passed through earlier agent nodes) and `data.sirmaai_project_id` (already available as `$json.projectId`) and `data.source_type` (literal `"aop"`, `"ted"`, or `"eu_grants"`). This is an **additive minor bump** — version files stay `*-v1.json` (per S05.20 policy: minor in-place permitted, major requires new file). Re-run `python scripts/sync_n8n_templates.py --apply` and update the audit-log via the script's --force-reason-aware path if drift is detected; otherwise plain apply.
15. **AC15 — Metrics**: Extend `data_pipeline.metrics` with a new Counter `pipeline_workflow_events_total{source_type, outcome}` where `outcome ∈ {processed, idempotent_skip, dlq, parse_error, cross_tenant_violation, status_skip, publish_failure}`. Increment after each terminal disposition. Existing `pipeline_opportunities_total{source_type, action}` is also incremented on successful upsert (action ∈ {new, updated}) — preserve the S05.12 contract.
16. **AC16 — Structured logging**: Every log line from the consumer module includes `task_name='workflow_event_consumer'`, `eusolicit_run_id` (when known), `correlation_id` (= `eusolicit_run_id`), `webhook_id`, `subscription_id`, and `source_type`. NEVER log the full webhook payload, the SirmaAI bearer token, or any field whose key matches the existing eusolicit redaction regex.
17. **AC17 — Negative test: cross-tenant**: Integration test seeds two companies A and B with their own `sirmaai_projects` rows. Construct a `workflow.completed` event where the outer `data.sirmaai_project_id` belongs to Company A but `data.opportunities[0].company_id` is Company B's UUID. Assert: the message is DLQ'd, NO rows are written to `pipeline.opportunities`, and the counter `pipeline_workflow_events_total{outcome="cross_tenant_violation"}` increments by 1.
18. **AC18 — Negative test: idempotent re-delivery**: Integration test XADD's the same valid event twice with the same `client_reference_id`. Assert: exactly one row created in `pipeline.opportunities`, exactly one `OpportunitiesIngested` envelope on `eu-solicit:opportunities`, the second delivery results in `pipeline_workflow_events_total{outcome="idempotent_skip"}` increment.
19. **AC19 — Negative test: invalid project**: Integration test sends an event whose `sirmaai_project_id` does not exist in `client.sirmaai_projects`. Assert: DLQ with reason `unknown_or_unprovisioned_project`, no DB write, no `OpportunitiesIngested` publish.
20. **AC20 — Negative test: malformed JSON**: Integration test XADD's a message whose `payload` field is `"{not-json"`. Assert: parse-error log line emitted, message acked (no PEL retention), DLQ counter not incremented (parse errors are explicitly NOT DLQ'd per AC10).
21. **AC21 — DoD per project standards**: `make lint`, `make type-check`, `make test-service SVC=data-pipeline` (unit + integration) all green; coverage on `data_pipeline/services/workflow_event_consumer.py` ≥ 90%. No new `print`, no bare `except:`, no `from … import *`, every external call has an explicit timeout. Project-level `make coverage` must remain ≥ 80%.

## Tasks / Subtasks

### Task 1 — Async session + Redis plumbing (AC12, AC13)
- [x] **1.1** Add `services/data-pipeline/src/data_pipeline/db_async.py` exporting `get_async_session_factory()` (returns `async_sessionmaker[AsyncSession]`). Convert `DATABASE_URL` to `postgresql+asyncpg://` if needed. Module-level lazy singleton — one engine per process. Add a `dispose_async_engine()` for test teardown.
- [x] **1.2** Add `services/data-pipeline/src/data_pipeline/services/__init__.py` (new package) and `services/redis_client.py` exporting `get_redis()` (lifespan-bound `redis.asyncio.Redis` singleton). Mirror `sirmaai_gateway.services.redis_client` shape: factory + `init_redis(app)` + `close_redis(app)` to be called from lifespan.
- [x] **1.3** Wire a `lifespan` context manager into `data_pipeline/main.py`. Inside lifespan: (a) `init_redis(app)`, (b) `await bootstrap_event_bus(redis)`, (c) launch `run_workflow_event_consumer` as `asyncio.create_task` and stash the handle on `app.state.workflow_router_task`, (d) on shutdown: `task.cancel()` + `await asyncio.wait_for(task, timeout=10)` with `asyncio.TimeoutError` swallowed-and-logged, (e) `await close_redis(app)`. Pass `lifespan=` to the `FastAPI(...)` constructor.

### Task 2 — Register the new consumer group (AC1, AC13)
- [x] **2.1** Edit `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py`: add the new consumer group entry. Either (a) extend the existing `"data-pipeline"` entry's stream list with `"sirmaai.workflow.completed"`, OR (b) add a new entry `"data-pipeline-workflow-router": ["sirmaai.workflow.completed"]`. **Choose (b)** so the consumer name and group name align (one group → one consumer process responsibility) and so future operator dashboards can distinguish the router group from the legacy `eu-solicit:admin` listener.
- [x] **2.2** Update the test-utils canonical map in `packages/eusolicit-test-utils/src/eusolicit_test_utils/redis_utils.py` (and any other place the spec calls out as "must match exactly") with the new entry. Search-grep `"data-pipeline"` and `"eu-solicit:admin"` to locate every authoritative copy.
- [x] **2.3** Add a unit test asserting `CONSUMER_GROUPS["data-pipeline-workflow-router"] == ["sirmaai.workflow.completed"]` so future edits to the dict trip CI.

### Task 3 — Consumer module skeleton (AC1, AC11, AC16)
- [x] **3.1** Create `services/data-pipeline/src/data_pipeline/services/workflow_event_consumer.py`. Top-level structure mirrors `sirmaai_gateway/services/rate_limit_sync_consumer.py`:
  - Module-level constants: `_STREAM = "sirmaai.workflow.completed"`, `_GROUP = "data-pipeline-workflow-router"`, `_CONSUMER = "data-pipeline-router-1"`, `_BLOCK_MS = 2000`, `_COUNT = 10`, `_DLQ_MAX_RETRIES = 3`, `_IDEMPOTENCY_KEY_PREFIX = "data_pipeline:workflow_router:idempotency:"`, `_IDEMPOTENCY_TTL_SECONDS = 7 * 24 * 3600`.
  - `_ensure_consumer_group(redis)` helper that swallows `BUSYGROUP`.
  - `_handle_message(event, consumer, session_factory, redis, metrics)` coroutine — extracted for testability (mirror precedent).
  - `run_workflow_event_consumer(app)` main loop coroutine: `_ensure_consumer_group` → infinite `while True` with `process_pending` → `retry_pending(min_idle_ms=0)` → `consume(block_ms=2000)` → per-event dispatch → `asyncio.CancelledError` re-raise → other-exception `sleep(5)`.
- [x] **3.2** Bind structlog with `task_name="workflow_event_consumer"` once at the top of `_handle_message` so every log line carries it. Use `bound_log = log.bind(...)` pattern — don't shadow the module-level `log`.

### Task 4 — Event parsing + status gate + idempotency (AC2, AC3, AC4, AC10)
- [x] **4.1** Inside `_handle_message`, extract `webhook_id`, `subscription_id`, `event_type`, `payload` from the Redis Streams message dict. If `event_type != "workflow.completed"`: log INFO `workflow_router.unexpected_event_type`, ack, return.
- [x] **4.2** `json.loads(payload)` inside a try/except for `(ValueError, TypeError)`. On parse failure: emit `workflow_router.parse_error` log line, ack the message, increment metric `pipeline_workflow_events_total{outcome="parse_error"}`, return. **Do not raise** — leave-in-PEL would loop forever.
- [x] **4.3** Extract `data = parsed_payload.get("data") or {}`. Pull `client_reference_id`, `status`, `sirmaai_project_id`, `source_type`, `opportunities`. Validate types: `client_reference_id` parses as UUID, `source_type ∈ {"aop","ted","eu_grants"}`, `opportunities` is a list. On any validation failure: emit `workflow_router.schema_validation_failed` log line with the specific field name (NEVER the value), ack, increment `outcome="parse_error"`, return.
- [x] **4.4** Call `map_status(status)` — import from `sirmaai_gateway.services.workflow_run_repository`. **CRITICAL**: cross-service imports are forbidden under the schema isolation rule for DB writes, but pure-function imports are allowed. Confirm by checking that `map_status` is a stateless mapping (it is — it's a dict lookup). If status != `"succeeded"`: log INFO `workflow_router.status_skipped`, ack, increment `outcome="status_skip"`, return.
- [x] **4.5** Implement `_check_and_set_idempotency(redis, run_id) -> bool`. Returns True if newly claimed, False if duplicate. Use `await redis.set(key, iso_now, nx=True, ex=_IDEMPOTENCY_TTL_SECONDS)` (single round-trip — combines SETNX + EXPIRE). If False (duplicate): log INFO `workflow_router.idempotent_skip`, ack, increment `outcome="idempotent_skip"`, return.

### Task 5 — Tenant resolution + cross-tenant guard (AC5)
- [x] **5.1** Implement `_resolve_project(session, sirmaai_project_id) -> tuple[UUID, str] | None` — raw SQL `SELECT company_id, provisioning_status FROM client.sirmaai_projects WHERE sirmaai_project_id = :pid LIMIT 1`. Return `(company_id, provisioning_status)` or `None`. **NO ORM import** from client-api (cross-service ORM is the forbidden direction; raw SQL read is the precedent — `sirmaai_gateway/services/rate_limit_sync_consumer._resolve_project`).
- [x] **5.2** If `_resolve_project` returns `None` OR `provisioning_status != "provisioned"`: write DLQ via direct XADD on `sirmaai.workflow.completed.dlq` with envelope `{original_payload..., dlq_reason: "unknown_or_unprovisioned_project", dlq_timestamp, original_message_id, eusolicit_run_id}`, ack original message, increment `outcome="dlq"`, return.
- [x] **5.3** For each record in `data.opportunities`: if `record.get("company_id")` is set AND `record["company_id"] != str(resolved_company_id)`: DLQ with reason `cross_tenant_violation`, ack, increment `outcome="cross_tenant_violation"`, return. **The DLQ write happens BEFORE any DB write** — no partial writes on a cross-tenant violation.

### Task 6 — Upsert + publish (AC6, AC7, AC8, AC9)
- [x] **6.1** With `async with session_factory() as session: ... await session.commit()`: call `upsert_opportunities(session, data.opportunities, data.source_type)` — but note: the existing helper is **synchronous** (`Session` not `AsyncSession`). Two options:
  - (a) Wrap the call in `asyncio.to_thread(...)` using a sync session via `get_sync_session()` — simplest, no helper rewrite.
  - (b) Add an `await`-able sibling `upsert_opportunities_async(async_session, ...)` mirroring the sync version.
  
  **Choose (a)** for this story — the upsert is short and DB-bound; `to_thread` avoids duplicating SQL-level logic. Document the decision inline in the consumer module so reviewers don't ask. Verify with a load test in a follow-up that the thread-pool default size is adequate (CPython default = `min(32, os.cpu_count()+4)` — far larger than our consumer concurrency of 1).
- [x] **6.2** Sanity-check that `_upsert.py:upsert_opportunities`'s `update_cols` does NOT include `deleted_at`. Confirmed by reading the file — `deleted_at` is not in the `update_cols` dict, so soft-deleted rows stay soft-deleted on re-upsert. Add a unit test (`test_upsert_does_not_resurrect_soft_deleted`) asserting this with a fixture row whose `deleted_at IS NOT NULL`.
- [x] **6.3** Capture `(new_count, updated_count, opportunity_ids)` from the upsert. Build the `OpportunitiesIngested` envelope (copy the exact shape from `publish_event.publish_ingested_event` lines 99-116). Use `summary.unchanged = max(0, len(data.opportunities) - new_count - updated_count)`.
- [x] **6.4** XADD the envelope to `eu-solicit:opportunities` inside a try/except. On success: ack the source message, increment `pipeline_workflow_events_total{outcome="processed"}` and `pipeline_opportunities_total{source_type=..., action=...}` for each new/updated record. On XADD failure: log `workflow_router.publish_failed` (WARNING), increment `outcome="publish_failure"`, **DO NOT ack** — let PEL retry. The S04.25 SETNX from AC4 protects against a duplicate upsert on the next delivery.

### Task 7 — Metrics (AC15)
- [x] **7.1** Open `services/data-pipeline/src/data_pipeline/metrics.py`. Add a Counter `pipeline_workflow_events_total` with labels `["source_type", "outcome"]` registered on `PIPELINE_METRICS_REGISTRY`. Export via `__all__`. Document the canonical outcome values in the docstring.
- [x] **7.2** In `workflow_event_consumer.py`, import the new counter and call `.labels(source_type=..., outcome=...).inc()` at each terminal disposition. Where `source_type` is unavailable (e.g. parse_error before payload extraction), use the literal `"unknown"`.

### Task 8 — N8N template payload extension (AC14)
- [x] **8.1** For each of `infra/n8n-templates/crawl-aop-v1.json`, `crawl-ted-v1.json`, `crawl-eu-grants-v1.json`: locate the `Build Workflow Completed Event` Set node. Add three assignments to the `parameters.assignments.assignments` array:
  - `data.opportunities` ← `={{$json.opportunities}}` (type: array) — the normalised+scored record list flowing through from the Submission Guide Agent step.
  - `data.sirmaai_project_id` ← `={{$json.projectId}}` (type: string).
  - `data.source_type` ← literal `"aop"` / `"ted"` / `"eu_grants"` (type: string, per template).
- [x] **8.2** Bump the canonical hash baseline that `tests/unit/test_n8n_template_payloads.py` asserts on (S05.20 ships an exact-match payload test). Update the expected hash fixtures so CI passes — DO NOT add `--force` to the deploy command unless directed.
- [x] **8.3** Run `python services/data-pipeline/scripts/sync_n8n_templates.py --check` locally to confirm drift detection still fires. Then `--apply` in a separate operator step at deploy time — not in this PR's CI. (CI never `--apply`s; only `--check`s per S05.20 contract.)

### Task 9 — Tests (AC17–AC21)
- [x] **9.1** Unit tests in `services/data-pipeline/tests/unit/test_workflow_event_consumer.py`:
  - Event-type-not-workflow-completed → ack + skip.
  - JSON parse error → ack + parse_error metric.
  - Schema validation failure (missing field, non-UUID run_id, invalid source_type) → ack + parse_error metric.
  - Status != succeeded (failed, cancelled, running) → ack + status_skip metric, no DB call.
  - Idempotent re-delivery via mocked `redis.set(nx=True)` returning `None` → ack + idempotent_skip metric, no DB call.
  - Soft-delete invariant: `test_upsert_does_not_resurrect_soft_deleted` — seed a row with `deleted_at` set, run the upsert path, assert `deleted_at` is unchanged.
- [x] **9.2** Integration tests in `services/data-pipeline/tests/integration/test_workflow_event_consumer.py` (marker: `@pytest.mark.integration` — needs postgres + redis from `make infra`):
  - Happy path: XADD valid event → row appears in `pipeline.opportunities` → `OpportunitiesIngested` envelope readable from `eu-solicit:opportunities`.
  - Cross-tenant negative (AC17): two companies, mismatched `company_id` → DLQ + no DB write + `cross_tenant_violation` metric.
  - Idempotent re-delivery (AC18): XADD same event twice → 1 row + 1 published envelope + idempotent_skip metric on second.
  - Invalid project (AC19): unknown `sirmaai_project_id` → DLQ with `unknown_or_unprovisioned_project` reason.
  - Malformed payload (AC20): `payload="{not-json"` → parse_error log + immediate ack + no DLQ.
- [x] **9.3** Use the `db_session` fixture for the per-test transaction rollback contract — NEVER `commit()` in tests. For the integration tests that exercise actual XADD/XREAD, override `data_pipeline.services.redis_client.get_redis` via `app.dependency_overrides` (or direct monkeypatch in lifespan), and use the `clean_redis` fixture (DB 1 — application uses DB 0; never write test data to DB 0).
- [x] **9.4** Bonus: a regression test that asserts the new consumer group `data-pipeline-workflow-router` is present in `CONSUMER_GROUPS` so a future contributor cannot silently remove it.

### Task 10 — Definition of Done (AC21)
- [x] **10.1** `make lint` clean on `services/data-pipeline`, `packages/eusolicit-common`, `packages/eusolicit-test-utils`.
- [x] **10.2** `make type-check` clean for the same packages (`mypy services/data-pipeline packages/eusolicit-common packages/eusolicit-test-utils`).
- [x] **10.3** `make test-service SVC=data-pipeline` — unit + integration green. `make coverage` reports `data_pipeline/services/workflow_event_consumer.py ≥ 90%`.
- [x] **10.4** Update story `File List` with every file created or modified.

## Dev Notes

### Producer/consumer contract (the source of truth)

**Producer**: `services/sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py` lines 687–712. On `event_type == "workflow.completed"` after handler success, the receiver XADD's to Redis Stream `sirmaai.workflow.completed` with fields:

```
{
  "webhook_id":      "<svix-msg-id-or-uuid>",
  "subscription_id": "<sirmaai_webhook_subscriptions.id>",
  "event_type":      "workflow.completed",
  "received_at":     "<iso-8601 utc>",
  "payload":         "<json-stringified-full-sirmaai-webhook-envelope>"
}
```

The inner `payload` (when JSON-parsed) follows the Standard Webhooks envelope:

```
{
  "type": "workflow.completed",
  "timestamp": "<iso>",
  "data": {
    "client_reference_id": "<eusolicit run uuid>",
    "status": "SUCCEEDED" | "FAILED" | "CANCELLED" | ...,
    "completedAt": "<iso>",
    "error": null | "<message>",
    "sirmaai_project_id": "<sirmaai project uuid>",   // ← added by S05.21 template extension (Task 8)
    "source_type": "aop" | "ted" | "eu_grants",        // ← added by S05.21 template extension
    "opportunities": [                                   // ← added by S05.21 template extension
      {
        "source_id": "...",
        "title": "...",
        "description": "...",
        "deadline": "<iso>",
        "budget_min": 1000.00,
        "budget_max": 5000.00,
        "currency": "EUR",
        "country": "...",
        "region": "...",
        "contracting_authority": "...",
        "cpv_codes": ["..."],
        "evaluation_criteria": {...},
        "mandatory_documents": [...],
        "opportunity_type": "tender" | "grant",
        "raw_data": {...},
        "published_at": "<iso>"
        // company_id is OPTIONAL — when set, must match resolved company (cross-tenant guard)
      },
      ...
    ]
  }
}
```

`map_status` (from `sirmaai_gateway.services.workflow_run_repository`) maps SirmaAI's uppercase status to the lowercase internal vocabulary. Only `"succeeded"` advances to the upsert path.

### Why a FastAPI lifespan task, not a Celery task

Celery tasks are designed for finite work units (a few seconds to a few minutes). A Redis Streams consumer is a **persistent loop** that blocks on XREADGROUP for seconds at a time and runs indefinitely. The precedent (`rate_limit_sync_consumer` in `sirmaai-gateway`) uses the FastAPI lifespan pattern for exactly this reason: the loop is bound to the service process lifetime, cancellation flows through `asyncio.CancelledError`, and the consumer participates in graceful shutdown.

Side benefit: the consumer reads from Redis with `redis.asyncio`, which doesn't need the Celery process model — the `data-pipeline` service container's existing FastAPI app on port 8003 already runs an asyncio event loop.

### Idempotency choice: Redis SETNX over a DB dedup table

Three alternatives considered:
1. **DB table `pipeline.workflow_event_dedup(run_id PK, processed_at)`** — durable, but adds a migration, an INSERT on the hot path, and a cleanup task. The 7-day TTL behaviour requires either a sweeper or a partial index.
2. **`gateway.workflow_runs.status` query** — already exists. But the writer is `sirmaai-gateway`, not `data-pipeline`, and cross-schema reads from `data-pipeline` to `gateway` would establish a new boundary violation.
3. **Redis SETNX with 7-day TTL** — matches the S04.25 idempotency cache precedent, no migration, automatic expiry, single Redis round-trip via `set(nx=True, ex=604800)`. **Chosen.**

The 7-day TTL is the same retention window as Standard Webhooks' max re-delivery age (per S04.25 docstring), so the dedup horizon covers the full re-delivery window.

### Cross-tenant guard precedent

The `client.sirmaai_projects` table has `company_id UNIQUE`, `sirmaai_project_id UNIQUE`, and `provisioning_status` check-constrained to `('pending','provisioned','failed','archived')`. The consumer enforces:
- The event's `data.sirmaai_project_id` resolves to a row.
- That row's `provisioning_status == 'provisioned'`.
- Every `opportunities[].company_id` (where present) matches the resolved `company_id`.

This is **the** cross-tenant invariant for the consumer side of the SirmaAI pipeline. AC17 negative test is non-negotiable per the global "tenant-scoped resources need a negative authorization test" rule in `Delivery Instructions`.

### Reuse, not reinvent

**MUST reuse:**
- `data_pipeline.workers.tasks._upsert.upsert_opportunities` — atomic INSERT … ON CONFLICT DO UPDATE … RETURNING id. No new dedup logic.
- `eusolicit_common.events.consumer.EventConsumer` — `consume`, `ack`, `retry_pending`, `process_pending`. No raw `XREADGROUP` calls in the consumer module.
- `eusolicit_common.events.bootstrap.bootstrap_event_bus` — idempotent group/stream creation.
- `sirmaai_gateway.services.workflow_run_repository.map_status` — SirmaAI uppercase → internal lowercase status mapping. (Imported as a pure function.)
- The S05.09 `OpportunitiesIngested` envelope shape (lines 99–116 of `publish_event.py`) — preserve byte-for-byte for downstream parity.

**MUST NOT:**
- Reimplement `(source_id, source_type)` dedup — exists in `_upsert.py`.
- Reimplement HMAC verification or webhook idempotency — that's S04.25's job; we trust the upstream stream.
- Write to `pipeline.crawler_runs` — that table is retired (read-only history per amendment).
- Cross-import `client_api` ORM models. Raw SQL only for `client.sirmaai_projects` reads (matches sirmaai-gateway's `rate_limit_sync_consumer` precedent).
- Touch `gateway.workflow_runs` from data-pipeline — that's sirmaai-gateway's domain.

### Files to UPDATE (not create) and what must be preserved

| File | What changes | What must NOT break |
|---|---|---|
| `services/data-pipeline/src/data_pipeline/main.py` | Add `lifespan=` to `FastAPI(...)`. Keep `/healthz`, `/metrics`, `/pipeline-health` endpoints intact. | The existing `MetricsMiddleware` registration and the `generate_latest()` concatenation (PIPELINE_METRICS_REGISTRY + _http_metrics_registry + CELERY_METRICS_REGISTRY + OUTBOUND_METRICS_REGISTRY) — see AP-GUARD-1 comment, do not regress. |
| `services/data-pipeline/src/data_pipeline/metrics.py` | Add `pipeline_workflow_events_total` Counter. | Existing `pipeline_crawl_duration_seconds`, `pipeline_opportunities_total`, `pipeline_enrichment_queue_depth`, `pipeline_agent_call_duration_seconds` (S05.12 contract — verbatim per AP-GUARD-1). |
| `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py` | Add `"data-pipeline-workflow-router": ["sirmaai.workflow.completed"]` entry to `CONSUMER_GROUPS`. | The existing `"data-pipeline"` group entry (subscribed to `eu-solicit:admin`) — it has live consumers; do NOT replace. |
| `packages/eusolicit-test-utils/src/eusolicit_test_utils/redis_utils.py` (or equivalent canonical map) | Mirror the new consumer group. | Test fixtures referencing the existing groups. |
| `infra/n8n-templates/crawl-{aop,ted,eu-grants}-v1.json` | Extend `Build Workflow Completed Event` Set node with three new assignments (per AC14). Bump canonical-hash fixture in `tests/unit/test_n8n_template_payloads.py`. | The earlier nodes (Cron, Feature Flag Guard, the four SirmaAI agents) and node `id` values — additive change only. Version stays `v1` (minor in-place per S05.20 policy). |

### Files to CREATE

- `services/data-pipeline/src/data_pipeline/db_async.py`
- `services/data-pipeline/src/data_pipeline/services/__init__.py`
- `services/data-pipeline/src/data_pipeline/services/redis_client.py`
- `services/data-pipeline/src/data_pipeline/services/workflow_event_consumer.py`
- `services/data-pipeline/tests/unit/test_workflow_event_consumer.py`
- `services/data-pipeline/tests/integration/test_workflow_event_consumer.py`

No Alembic migration in this story. (Idempotency is Redis-backed; no new tables.)

### Security must-dos (per project Delivery Instructions)

- All `httpx`/Redis calls have explicit timeouts (no network here, only Redis — `socket_connect_timeout=5`, `socket_timeout=10` on the Redis client).
- No `==` for any signature or token comparison. (No comparisons in this story — the receiver already verified the HMAC; we trust the stream.)
- `User.is_active` — N/A (no user context on this path).
- Never log the SirmaAI bearer token, the full webhook payload, or any opportunity `raw_data` blob at DEBUG or above. Bind only IDs (`run_id`, `webhook_id`, `subscription_id`, `source_type`).
- `pipeline.opportunities.deleted_at` is NEVER written by the consumer; preserved by AC7 + soft-delete unit test.

### Testing standards

- Use the markers correctly: unit (`@pytest.mark.unit` — no I/O, mocks everywhere), integration (`@pytest.mark.integration` — needs postgres + redis from `make infra`).
- Per-test transaction rollback via `db_session` — NEVER `commit()` in a test.
- `clean_redis` fixture flushes DB 1; application uses DB 0; integration tests use DB 1 only.
- For lifespan-bound state, use `app.dependency_overrides` or `monkeypatch` and clean up in `finally`.

### Project Structure Notes

Conforms to the documented data-pipeline layout (`src/data_pipeline/{models,workers/tasks,services}`). Adds a new `services/` sub-package for service-layer (non-Celery) code — mirrors `sirmaai-gateway`'s `src/sirmaai_gateway/services/` precedent.

The `from __future__ import annotations` header is required at the top of every new Python module per project rules (Python 3.12, async throughout).

### References

- E05 amendment (S05.21 row): [Source: eusolicit-docs/planning-artifacts/epics/E05-data-pipeline-ingestion.md#stories--amendment-delta] line 224.
- E04 amendment, S04.25 receiver: [Source: eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/routers/webhooks.py lines 630–782] — producer of `sirmaai.workflow.completed`.
- E04 amendment, S04.26 reconciler: [Source: architecture-amendment-2026-05-12-sirmaai.md §4.4] — webhooks are strictly latency optimisation; the reconciler is authoritative.
- S05.09 envelope contract: [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/publish_event.py lines 44–177] — the `OpportunitiesIngested` envelope to preserve.
- S05.20 N8N templates + canonical-hash policy: [Source: eusolicit-docs/implementation-artifacts/5-20-n8n-workflow-templates-and-commit-to-git-deploy.md] — template-amendment workflow + drift-refusal CI gate.
- Cross-service consumer precedent: [Source: eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/services/rate_limit_sync_consumer.py] — extract `_handle_message`, loop structure, raw-SQL DB resolution, ack semantics.
- Atomic upsert helper: [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py] — DO NOT reimplement.
- EventConsumer abstraction: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/consumer.py].
- Event bus bootstrap and canonical maps: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py].
- SirmaAIProject model + tenant resolution: [Source: eusolicit-app/services/client-api/src/client_api/models/sirmaai_project.py].
- Pipeline schema isolation rule and Redis DB-0/DB-1 split: [Source: eusolicit-app/CLAUDE.md and project root CLAUDE.md].
- Architecture amendment §3.1 (sirmaai-gateway as the only SirmaAI integration point): respected — this consumer only reads from a Redis Stream that sirmaai-gateway already wrote; it never calls SirmaAI directly.

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (2026-05-14) — dev-story-review-fix phase

### Debug Log References

N/A — no external service calls required. All implementation uses mocked I/O in unit tests; integration tests use testcontainers (postgres) + localhost Redis DB 1.

### Completion Notes List

1. **Status gate mapping deviation (AC3)**: AC3 specifies `mapped_status == "succeeded"` but `sirmaai_gateway.services.workflow_run_repository.map_status("COMPLETED")` returns `"completed"` (not `"succeeded"`). The implementation uses `mapped_status == "completed"` to match the actual mapping function. See Known Deviations.

2. **Upsert uses `asyncio.to_thread` (AC6 option a)**: The existing `upsert_opportunities` helper is synchronous. Per the story's explicit guidance, we wrap it with `asyncio.to_thread` + a sync session rather than rewriting the helper. Rationale is documented inline in the consumer module.

3. **F1+F2 fix (idempotency ordering)**: Following code review, the idempotency key is now deleted on all leave-in-PEL exit paths (DB resolve error, upsert error, XADD failure). This prevents the "sticky SETNX" failure mode where a transient DB error causes subsequent retries to be incorrectly skipped as `idempotent_skip`. Module docstring updated with "Idempotency key lifecycle" section explaining the design.

4. **F4 fix (publish-failure test)**: Added UNIT-013 covering the XADD failure path — verifies no ack, `publish_failure` metric increment, and idempotency key deletion via `redis.delete`.

5. **F5 fix (soft-delete test)**: Replaced `inspect.getsource` unit test with an SQL-compilation behavioral check (compiles the actual SQLAlchemy statement and asserts `deleted_at` absent from the SET clause). Added a behavioral integration test `test_upsert_soft_delete_preserved` that seeds a real row with `deleted_at IS NOT NULL`, re-upserts, and asserts `deleted_at` is preserved.

6. **F3 fix (metric assertions)**: Integration tests INT-002, INT-003, INT-004 now assert the specific `pipeline_workflow_events_total` metric label increments mandated by AC17, AC18, AC19 respectively, using a before/after delta pattern.

### File List

**New:**
- `services/data-pipeline/src/data_pipeline/db_async.py`
- `services/data-pipeline/src/data_pipeline/services/__init__.py`
- `services/data-pipeline/src/data_pipeline/services/redis_client.py`
- `services/data-pipeline/src/data_pipeline/services/workflow_event_consumer.py`
- `services/data-pipeline/tests/unit/test_workflow_event_consumer.py`
- `services/data-pipeline/tests/integration/test_workflow_event_consumer.py`

**Modified:**
- `services/data-pipeline/src/data_pipeline/main.py` — added lifespan context manager, consumer task wiring
- `services/data-pipeline/src/data_pipeline/metrics.py` — added `WORKFLOW_EVENTS_TOTAL` counter
- `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py` — added `data-pipeline-workflow-router` consumer group + `sirmaai_workflow_completed` stream
- `packages/eusolicit-test-utils/src/eusolicit_test_utils/redis_utils.py` — mirrored new consumer group + stream
- `infra/n8n-templates/crawl-aop-v1.json` — extended Build Workflow Completed Event node with `data.opportunities`, `data.sirmaai_project_id`, `data.source_type`
- `infra/n8n-templates/crawl-ted-v1.json` — same extension
- `infra/n8n-templates/crawl-eu-grants-v1.json` — same extension

### Test Results

```
13 passed in 1.00s   (services/data-pipeline/tests/unit/test_workflow_event_consumer.py — 13 unit tests including new UNIT-013)
6 passed in 5.04s    (services/data-pipeline/tests/integration/test_workflow_event_consumer.py — all 6 integration tests including new soft-delete behavioral test)
66 passed in 1.27s   (services/data-pipeline/tests/unit/ — full unit suite, -m unit)
47 passed, 12 skipped in 202.46s  (services/data-pipeline/tests/integration/ — full integration suite, -m integration)
```

### Known Deviations

#### Known Deviation (AC3) — Status gate: "succeeded" vs "completed"

**What AC3 demanded:** "Only events whose mapped status (per `sirmaai_gateway.services.workflow_run_repository.map_status`) equals `succeeded` proceed to the upsert path."

**What was implemented:** The status gate checks `mapped_status == "completed"`.

**Why:** `map_status("COMPLETED")` returns `"completed"` (not `"succeeded"`) per the actual implementation in `workflow_run_repository.py`. The literal string `"succeeded"` does not appear in `_SIRMAAI_TO_INTERNAL_STATUS`. AC3 contains a terminology inconsistency — "succeeded" was likely intended to describe the semantic meaning (a successful run), not a literal internal status string. Using `"completed"` is correct given the implementation.

**Follow-up:** No new story required — this is a spec wording issue, not a functional gap. If the `map_status` function is ever updated to add a `"SUCCEEDED"` mapping, the consumer condition should be updated accordingly.

### Detected by `3-code-review` at 2026-05-14T15:53:40Z (session 0869e156-5163-4fcd-bcb1-30ce7a27228f)

- Idempotency SETNX is claimed before tenant resolution and upsert; a transient DB error on either path leaves the SETNX key set without an upsert, so the retry will be incorrectly marked `idempotent_skip` and the data is never written — directly contradicting AC18's "exactly one row created" guarantee. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- AC9 publish-failure isolation does not deliver at-least-once `OpportunitiesIngested` publish — the same early-SETNX ordering causes the retry to short-circuit before re-XADD. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- Integration tests for AC17/AC18/AC19 do not assert the `pipeline_workflow_events_total` metric increments the ACs explicitly require; AC9 publish-failure isolation has no test at all; the soft-delete unit test is a `inspect.getsource` text scan rather than a behavioral assertion. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- Integration tests have never been executed (Completion Notes admit "needs make infra to run"); per project DoD, tests for the changed surface must actually run and pass before completion. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- Idempotency SETNX is claimed before tenant resolution and upsert; a transient DB error on either path leaves the SETNX key set without an upsert, so the retry will be incorrectly marked `idempotent_skip` and the data is never written — directly contradicting AC18's "exactly one row created" guarantee.
- AC9 publish-failure isolation does not deliver at-least-once `OpportunitiesIngested` publish — the same early-SETNX ordering causes the retry to short-circuit before re-XADD.
- Integration tests for AC17/AC18/AC19 do not assert the `pipeline_workflow_events_total` metric increments the ACs explicitly require; AC9 publish-failure isolation has no test at all; the soft-delete unit test is a `inspect.getsource` text scan rather than a behavioral assertion.
- Integration tests have never been executed (Completion Notes admit "needs make infra to run"); per project DoD, tests for the changed surface must actually run and pass before completion. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_

## Senior Developer Review

**Reviewer:** claude-sonnet (bmad-code-review)
**Date:** 2026-05-14
**Outcome:** REVIEW: Changes Requested

The implementation tracks the spec closely and most ACs are mechanically satisfied. Code is well-structured, mirrors the `rate_limit_sync_consumer` precedent, and lint/style/security defaults are respected. However, there are two correctness concerns where the implementation either breaks an invariant the spec promises (AC18) or faithfully implements a spec that contains an internal contradiction (AC9), plus several test-coverage gaps against the explicit AC17–AC20 wording. None of these is a structural rewrite, but they need to be resolved before merging.

### Findings

#### F1 — HIGH — Idempotency SETNX claimed before tenant resolution / upsert breaks AC18 under transient errors

**File:** `services/data-pipeline/src/data_pipeline/services/workflow_event_consumer.py` lines 363–387, 429–441.

The flow is:
1. `_check_and_set_idempotency` (SETNX with 7-day TTL) — line 363
2. Tenant resolve via async DB — line 379, on transient `Exception` returns **without ack** (line 386)
3. `_upsert_sync` via `asyncio.to_thread` — line 430, on transient `Exception` returns **without ack** (line 440)

If either (2) or (3) raises (asyncpg pool exhaustion, brief postgres restart, etc.), the message stays in the PEL for retry, but the SETNX key has already been claimed. On the next delivery, `_check_and_set_idempotency` returns `False`, the message is acked as `idempotent_skip`, and the data is **never** upserted. Within the 7-day TTL window this is permanent data loss for that run_id, even though the upstream stream and PEL both behaved correctly.

This directly contradicts AC18 ("Assert: exactly one row created in `pipeline.opportunities`") under realistic transient-failure conditions. The precedent `rate_limit_sync_consumer` does not have this hazard because it has no client-side idempotency cache — every retry re-attempts the SirmaAI PUT.

**Recommended fix (pick one):**
- (a) Move SETNX to *after* a successful upsert (immediately before XADD), so a transient DB error never claims the slot.
- (b) On any "leave-in-PEL" exit path (lines 386–387, 440–441), `await redis_client.delete(idempotency_key)` before returning, releasing the slot for the retry. Keep the early SETNX for non-retried fast-paths.
- (c) Use a two-phase token: SETNX an "in-progress" sentinel that is overwritten with a "completed" sentinel only after publish; treat in-progress sentinels as expired-after-N-seconds so retries can claim them. Heavier; only worth it if (a)/(b) are infeasible.

(a) is simplest and matches typical at-least-once semantics. Document the chosen approach inline.

#### F2 — MEDIUM-HIGH — AC9 "at-least-once publish" guarantee is broken by the same idempotency ordering

**File:** Same module, lines 363 vs 473–484.

AC9 explicitly states: "On XADD failure (transient Redis error), the consumer does NOT ack the source message — leaving it in the PEL so `retry_pending`/`process_pending` re-delivers it. … Result: at-least-once event publish without duplicate writes."

The XADD branch (lines 473–484) correctly leaves the message in the PEL on failure. But when the message is re-delivered, `_handle_message` reaches `_check_and_set_idempotency` first (line 363), finds the key already set, increments `idempotent_skip`, and acks — without re-running publish. Net effect: **the downstream `OpportunitiesIngested` envelope is silently dropped** whenever an XADD fails on the first delivery. This is at-most-once, not at-least-once.

The spec itself is internally contradictory (it claims "at-least-once publish without duplicate writes" but the SETNX-first design cannot deliver both). Fixing F1 above with option (a) (SETNX after upsert, before publish) also solves F2 — a re-delivery after a publish failure would re-run the upsert (which is itself idempotent at the DB level via `ON CONFLICT (source_id, source_type) DO UPDATE`), and then attempt the XADD again.

**Recommended fix:** Same change as F1 option (a). Add an explicit test that simulates an XADD failure on the first delivery, then a successful retry, asserting exactly one `OpportunitiesIngested` envelope is published. This is currently uncovered.

#### F3 — MEDIUM — AC17/AC18/AC19 metric assertions are absent from the integration tests

**File:** `services/data-pipeline/tests/integration/test_workflow_event_consumer.py`.

AC17 (cross-tenant) explicitly says: "the counter `pipeline_workflow_events_total{outcome="cross_tenant_violation"}` increments by 1". The corresponding test (`test_cross_tenant_violation_dlq_no_db_write`) asserts the DLQ envelope and no DB write but does not touch the metric.

AC18 mandates the `idempotent_skip` metric assertion on the second delivery; `test_idempotent_redeliver_produces_one_row_and_one_event` only checks row count + envelope count.

AC19 mandates the `dlq` outcome counter increment; the test only checks the DLQ envelope.

The unit tests handle parse_error / status_skip / idempotent_skip metric counters via mocked counters, but the integration suite — which is where AC17–AC19 explicitly call out metric increments — does not. Add `WORKFLOW_EVENTS_TOTAL._metrics` (or `.labels(...)._value.get()`) sampling before and after `_handle_message` to assert the deltas.

#### F4 — MEDIUM — AC9 publish-isolation path has zero test coverage

**Files:** `tests/unit/test_workflow_event_consumer.py`, `tests/integration/test_workflow_event_consumer.py`.

The XADD-failure branch (lines 473–484 of the consumer) — including the contract that the message must **not** be acked when publish fails, and that `outcome="publish_failure"` is incremented — is not exercised by any test. A simple unit test that injects an `AsyncMock` whose `.xadd` side-effect raises `ConnectionError` (or a generic `Exception`) is sufficient to cover the AC. Without it, any future regression that accidentally acks before XADD would slip through.

#### F5 — MEDIUM — Soft-delete unit test is a text scan, not a behavioral assertion

**File:** `tests/unit/test_workflow_event_consumer.py` lines 329–345.

`test_upsert_does_not_resurrect_soft_deleted` checks `inspect.getsource(upsert_opportunities)` for the literal string `"deleted_at"`. This passes if the function source happens not to contain the literal `deleted_at` and would fail if (say) someone added a comment mentioning `deleted_at` even while preserving the invariant. Conversely, a future bug that resurrects soft-deleted rows via a different mechanism (e.g., a separate `UPDATE ... SET deleted_at=NULL` outside the helper) would not be caught.

AC7 says "Add an explicit unit test asserting this invariant." A proper behavioral test seeds a row with `deleted_at IS NOT NULL`, runs `upsert_opportunities` with a record sharing the same `(source_id, source_type)`, and asserts the `deleted_at` value is unchanged after the call. This is straightforward with the `db_session` fixture.

#### F6 — LOW — Integration tests have never been executed

The Test Results section reports "5 collected" for the integration suite; the Completion Notes admit "They cannot be run from the host venv without `make infra`". Per the project DoD, integration tests for the changed surface must actually pass, not merely collect. This needs to be run in CI (or on a host with `make infra` up) before merge. Without it, AC17–AC20 are unverified at the runtime level despite the test code being present.

This is a process gap, not a code defect, but per the global Definition-of-Done rule ("Never claim a story complete on the basis that 'tests should pass.' Run them.") it must be addressed before approval.

#### F7 — LOW — Redundant consumer-group bootstrap

`bootstrap_event_bus(redis)` in the lifespan creates the `data-pipeline-workflow-router` group (via `bootstrap.py`'s `CONSUMER_GROUPS`), and `_ensure_consumer_group` inside `run_workflow_event_consumer` then creates the same group again. Both are idempotent (BUSYGROUP catch), so functionally fine, but the local `_ensure_consumer_group` is now dead defense given the centralized bootstrap. Either remove it or document that it's a belt-and-braces guard for environments where `bootstrap_event_bus` was skipped.

#### F8 — LOW — `OPPORTUNITIES_TOTAL` could double-count if both ingestion paths run concurrently

The Celery `ingest_opportunities` path increments `OPPORTUNITIES_TOTAL{source_type, action}`. The new SirmaAI path also increments it (lines 493–496). If both paths process the same crawl during the cutover (legacy KraftData crawler still enabled in some envs per S04.20 flag-off), counts will double. Not a correctness issue for the writer itself, just a metrics interpretation concern. Document the cutover assumption inline, or gate the increment on a flag.

#### F9 — LOW — Module-level `from sirmaai_gateway import map_status` re-imported on every message

Line 340 imports `map_status` inside `_handle_message`. With per-message overhead this is negligible (Python caches the module), and the inline comment explains the deliberate cross-service import. Consider lifting to module scope to make the cross-service dependency surface visible at import time (any failure to import surfaces at startup rather than per-event). Not a blocker.

#### F10 — INFO — STREAMS dict drift between `bootstrap.py` and `redis_utils.py`

`bootstrap.py` STREAMS lists `sirmaai_gateway_key_rotated` (S04.21); `redis_utils.py` STREAMS does not. The new entry `sirmaai_workflow_completed` was added to both, but the pre-existing drift remains. Not introduced by this story — flagging for awareness. The comment at the top of both files ("must match canonical definitions in `eusolicit-test-utils` `redis_utils.py`") is already a lie.

### What's good

- Module structure mirrors `rate_limit_sync_consumer` precedent cleanly; `_handle_message` extraction makes testing tractable.
- Raw-SQL tenant resolution avoids the cross-schema ORM import correctly.
- Cross-tenant guard runs **before** the upsert (AC5).
- DLQ writer redacts PII (line 411 comment is explicit) and serializes only non-private fields.
- Async Redis client has explicit `socket_connect_timeout` and `socket_timeout` (security must-do).
- Lifespan task cancellation has a 10s wait_for + TimeoutError swallow — graceful shutdown.
- `WORKFLOW_EVENTS_TOTAL` registered on the dedicated `PIPELINE_METRICS_REGISTRY` (prevents duplicate-timeseries pytest hazard).
- `decode_responses=True` on the Redis client (string fields for XADD).
- N8N template extension follows S05.20's "minor in-place" version policy.
- AC3 status-string deviation is well-documented in Known Deviations.

### Required for approval

1. Resolve F1 + F2 together (single ordering fix: SETNX after upsert, or delete-on-error). Add a unit/integration test that simulates a transient DB error on the upsert path and asserts the retry succeeds in upserting + publishing.
2. Add the missing metric-delta assertions (F3) and the publish-failure test (F4).
3. Replace the soft-delete unit test with a behavioral assertion against a fixture row (F5).
4. Actually run the integration suite via `make infra` + `make test-integration SVC=data-pipeline` (or CI) and update the Test Results section with the pytest summary line (F6).

The other findings (F7–F10) are non-blocking; address opportunistically or in a follow-up cleanup.

---

## Senior Developer Review — Follow-up (Round 2)

**Reviewer:** claude-sonnet (bmad-code-review)
**Date:** 2026-05-14
**Outcome:** REVIEW: Approve

The four blockers from the Round 1 review (F1–F2 idempotency ordering, F3 metric-delta assertions, F4 publish-failure coverage, F5 soft-delete behavioral test, F6 integration tests actually run) are all addressed. Implementation now correctly delivers AC18's "exactly one row" guarantee under transient DB / Redis errors while preserving at-least-once publish semantics for `OpportunitiesIngested`.

### Verification of Round 1 fixes

| Finding | Resolution verified |
|---|---|
| F1 (early SETNX blocks AC18 under transient DB error) | `workflow_event_consumer.py:399, 455` — idempotency key is `await redis_client.delete(idempotency_key)`'d on both the `_resolve_project` exception path and the `_upsert_sync` exception path before returning. Retry can re-claim the slot and proceed. Module docstring lines 24–33 document the lifecycle explicitly. |
| F2 (at-least-once publish broken by SETNX-first) | `workflow_event_consumer.py:500` — XADD failure path also deletes the idempotency key, so the PEL retry re-runs upsert (idempotent via `ON CONFLICT DO UPDATE`) and re-attempts XADD. |
| F3 (missing metric-delta assertions in integration tests) | `test_workflow_event_consumer.py:400-434, 483-520, 554-585` — INT-002 / INT-003 / INT-004 now use the `_get_metric_value` helper with before/after delta asserts on `cross_tenant_violation`, `idempotent_skip`, and `dlq` outcomes respectively. |
| F4 (no publish-failure test) | `tests/unit/test_workflow_event_consumer.py:407-454` — new UNIT-013 injects `redis.xadd.side_effect = ConnectionError(...)` and asserts (a) no ack, (b) `publish_failure` counter increment, (c) `redis.delete` of the idempotency key. |
| F5 (text-scan soft-delete test) | Unit test (lines 331-376) now compiles the actual SQLAlchemy `INSERT ... ON CONFLICT DO UPDATE` statement and asserts `deleted_at` is absent from the compiled SQL (behavioral on the SQL surface). Complemented by `test_upsert_soft_delete_preserved` integration test (lines 634-696) which seeds a real `deleted_at IS NOT NULL` row, re-upserts, and asserts the column is preserved against PostgreSQL. |
| F6 (integration suite never run) | Test Results section now reports `13 passed in 1.00s` (unit) and `6 passed in 5.04s` (integration for this story); full data-pipeline integration suite `47 passed, 12 skipped in 202.46s`. |

### Spot checks (new pass)

- `map_status` semantics: confirmed via `workflow_run_repository.py:62-66` — `_SIRMAAI_TO_INTERNAL_STATUS["COMPLETED"] = "completed"`. The AC3 deviation to gate on `mapped_status == "completed"` (not `"succeeded"`) matches the actual mapping. Inline comment at consumer line 357-359 and the Known Deviations entry document this clearly.
- `_upsert.py` soft-delete invariant: `grep -n "deleted_at" services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py` returns no matches — the helper does not touch `deleted_at` in `update_cols`, so re-upsert cannot resurrect a soft-deleted row.
- Consumer-group registration: `data-pipeline-workflow-router → ["sirmaai.workflow.completed"]` is mirrored in both `packages/eusolicit-common/.../bootstrap.py:60` and `packages/eusolicit-test-utils/.../redis_utils.py:43`. UNIT-012 asserts the contract.
- Lifespan wiring: `main.py:44-89` initialises Redis → bootstraps event-bus → launches `run_workflow_event_consumer` as `asyncio.create_task` → cancels with 10s `wait_for` on shutdown → swallows `TimeoutError` / `CancelledError` cleanly.
- Async Redis client: explicit `socket_connect_timeout=5`, `socket_timeout=10`, retry on `RedisConnectionError`/`RedisTimeoutError` (security must-do — explicit timeouts).
- Cross-tenant guard ordering: violation check (lines 419-435) runs **after** tenant resolution and **before** any DB write (AC5 invariant preserved).

### Residual observations (non-blocking, informational)

These are minor and explicitly not required for approval — flagged for awareness or follow-up cleanup:

- **R1 (LOW)** — `workflow_event_consumer.py:50-56` docstring (the "Publish isolation" section) still says "The SETNX from AC 4 already short-circuits the second delivery at the upsert path, ensuring at-least-once publish without duplicate writes." This contradicts the actual behavior (key is deleted on publish failure so the retry **does** re-run upsert and re-publish — relying on `ON CONFLICT DO UPDATE` for idempotent re-upsert rather than on SETNX for short-circuit). The Round 1-fix docstring at lines 24-33 ("Idempotency key lifecycle") has the correct description. Recommend updating lines 50-56 to match.
- **R2 (LOW)** — DLQ XADD failure leaves idempotency key claimed. If `_write_dlq` raises a transient Redis error (lines 408, 427) after the SETNX has already been claimed (line 375), the exception propagates without releasing the key. PEL retry would then `idempotent_skip` and the DLQ entry is permanently lost for that run_id (until the 7-day TTL expires). Mitigation: wrap `_write_dlq` calls with an except-clause that deletes the idempotency key before re-raising, mirroring the `_resolve_project` / `_upsert_sync` / XADD-publish patterns. Very low real-world likelihood (Redis is up if we got this far) — opportunistic fix only.
- **R3 (LOW, carried)** — F7 from Round 1 still applies: `_ensure_consumer_group` inside `run_workflow_event_consumer` duplicates work that `bootstrap_event_bus` already performs via `CONSUMER_GROUPS`. Both are idempotent (BUSYGROUP catch) so functionally fine; treat as belt-and-braces.
- **R4 (LOW, carried)** — F9: `map_status` import is still inside `_handle_message` (line 351). Lifting to module scope would surface a cross-service import failure at process start instead of per-event. Non-blocking.
- **R5 (LOW)** — `db_async.py:dispose_async_engine` is declared "FOR TEST TEARDOWN (and lifespan shutdown) ONLY" but `main.py:lifespan` does not call it on shutdown. Engine pool relies on process exit for release. Negligible in practice; consider calling it in the lifespan `finally` for completeness.

### Outcome

REVIEW: Approve. All Round-1 blockers verified resolved; the residual observations above are non-blocking and can be picked up opportunistically or in a follow-up story. The implementation faithfully delivers AC1–AC21 (with the AC3 wording-vs-implementation deviation cleanly documented in Known Deviations) and meets the project DoD on lint, type-check, unit + integration coverage, and metric assertions on the explicit AC17/AC18/AC19 paths.
