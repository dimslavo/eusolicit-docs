# Story 1.5: Redis Streams Event Bus Setup

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **developer on the EU Solicit platform**,
I want **a Redis Streams event bus with a lightweight publisher/consumer abstraction in `eusolicit-common`, 7 named streams, consumer groups per service, idempotent bootstrap, and dead letter handling**,
so that **services can communicate asynchronously via typed events with guaranteed at-least-once delivery, distributed tracing via correlation IDs, and automatic dead letter quarantine for failed messages — enabling all future event-driven features (notifications, pipeline triggers, billing events) to be built on a proven, tested foundation**.

## Acceptance Criteria

1. 7 Redis Streams created on startup: `eu-solicit:opportunities`, `eu-solicit:agents`, `eu-solicit:subscriptions`, `eu-solicit:notifications`, `eu-solicit:tasks`, `eu-solicit:billing`, `eu-solicit:admin`
2. Consumer groups created per service on their subscribed streams (see Consumer Group Mapping below)
3. `eusolicit-common` provides `EventPublisher` class with `async publish(stream, event)` method
4. `eusolicit-common` provides `EventConsumer` class with `async consume(stream, group, consumer_name)` method supporting both blocking and non-blocking reads
5. Events serialized as JSON with envelope: `{event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id}`
6. A Python bootstrap module creates streams and groups idempotently on service startup (uses `MKSTREAM` + `XGROUP CREATE` with error handling for existing groups)
7. Integration test: publish event to `eu-solicit:opportunities` stream, consume from consumer group, assert payload matches and all envelope fields present
8. Dead letter handling: messages that fail processing 3 times are XACK'd from the original stream and XADD'd to a `<stream>.dlq` stream with failure metadata

## Tasks / Subtasks

- [x] Task 1: Create `EventPublisher` class in `eusolicit-common` (AC: 3, 5)
  - [x] 1.1 Create `packages/eusolicit-common/src/eusolicit_common/events/publisher.py`
  - [x] 1.2 Implement `async publish(stream: str, event_type: str, payload: dict, *, source_service: str, tenant_id: str | None, correlation_id: str | None)` using `XADD`
  - [x] 1.3 Auto-generate `event_id` (UUID4) and `timestamp` (UTC ISO) in envelope
  - [x] 1.4 Serialize `payload` as JSON string within the envelope dict
  - [x] 1.5 Return the Redis message ID from `XADD`
- [x] Task 2: Create `EventConsumer` class in `eusolicit-common` (AC: 4, 8)
  - [x] 2.1 Create `packages/eusolicit-common/src/eusolicit_common/events/consumer.py`
  - [x] 2.2 Implement `async consume(stream: str, group: str, consumer_name: str, *, count: int = 10, block_ms: int | None = 2000)` using `XREADGROUP`
  - [x] 2.3 Support non-blocking mode when `block_ms=None` or `block_ms=0`
  - [x] 2.4 Implement `async ack(stream: str, group: str, message_id: str)` using `XACK`
  - [x] 2.5 Implement dead letter logic: `async process_pending(stream, group, consumer_name, max_retries=3)` — check `XPENDING` for messages exceeding retry threshold, move to `<stream>.dlq` via `XADD`, then `XACK` original
  - [x] 2.6 DLQ events include original envelope + `{dlq_reason, failure_count, dlq_timestamp, original_stream}`
- [x] Task 3: Create event bus bootstrap module (AC: 1, 2, 6)
  - [x] 3.1 Create `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py`
  - [x] 3.2 Define `STREAMS` and `CONSUMER_GROUPS` constants matching test-utils `redis_utils.py` exactly
  - [x] 3.3 Implement `async bootstrap_event_bus(redis_client)` — creates all 7 streams via `XGROUP CREATE ... MKSTREAM`, creates all consumer groups idempotently
  - [x] 3.4 Handle `BUSYGROUP` error (group already exists) gracefully — log and continue
  - [x] 3.5 Log stream/group creation using `structlog`
- [x] Task 4: Create `events` package structure and exports (AC: 3, 4)
  - [x] 4.1 Create `packages/eusolicit-common/src/eusolicit_common/events/__init__.py` — export `EventPublisher`, `EventConsumer`, `bootstrap_event_bus`, `STREAMS`, `CONSUMER_GROUPS`
  - [x] 4.2 Update `packages/eusolicit-common/src/eusolicit_common/__init__.py` — add events subpackage to top-level exports
- [x] Task 5: Integration tests (AC: 1, 2, 7, 8)
  - [x] 5.1 Create `tests/integration/test_redis_event_bus.py`
  - [x] 5.2 Test: bootstrap creates all 7 streams (verify via `XINFO STREAM`)
  - [x] 5.3 Test: bootstrap creates all consumer groups (verify via `XINFO GROUPS`)
  - [x] 5.4 Test: publish event to `eu-solicit:opportunities`, consume from group, assert all envelope fields match
  - [x] 5.5 Test: dead letter — force 3+ delivery attempts, verify event in `<stream>.dlq` with failure metadata
  - [x] 5.6 Test: idempotent bootstrap — run `bootstrap_event_bus` twice without error; streams/groups unchanged
  - [x] 5.7 Test: non-blocking consume returns empty list when no messages
  - [x] 5.8 Test: multiple consumers in same group receive different messages (load balancing)

## Dev Notes

### CRITICAL: Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `docker-compose.yml` | Redis 7 container on port 6379 with health checks | **DO NOT TOUCH** |
| `.env.example` | `REDIS_URL=redis://redis:6379/0` and `TEST_REDIS_URL=redis://localhost:6379/1` | **DO NOT TOUCH** |
| `Makefile` | Docker + test + migration targets | **DO NOT TOUCH** (no new targets needed) |
| `packages/eusolicit-test-utils/.../redis_utils.py` | Test event helpers, STREAMS/CONSUMER_GROUPS dicts, `build_event_envelope()` | **DO NOT TOUCH** — production code must MATCH these exactly |
| `tests/conftest.py` | `redis_client` and `clean_redis` fixtures using `TEST_REDIS_URL` | **DO NOT TOUCH** — reuse these fixtures |
| `packages/eusolicit-common/pyproject.toml` | Already includes `"redis>=5.0"` dependency | **DO NOT TOUCH** — no new deps needed |
| `services/*/src/*/main.py` | Minimal FastAPI skeletons with `/healthz` | **DO NOT TOUCH** — bootstrap integration deferred to Story 1.6 |

### CRITICAL: Align with Test Utilities — Single Source of Truth

The `redis_utils.py` in `eusolicit-test-utils` already defines the canonical stream names and consumer group mappings. Production code MUST match exactly:

**Stream Names** (from `redis_utils.py`):
```python
STREAMS = {
    "opportunities": "eu-solicit:opportunities",
    "agents": "eu-solicit:agents",
    "subscriptions": "eu-solicit:subscriptions",
    "notifications": "eu-solicit:notifications",
    "tasks": "eu-solicit:tasks",
    "billing": "eu-solicit:billing",
    "admin": "eu-solicit:admin",
}
```

All streams use the `eu-solicit:` prefix. The epic AC says just `opportunities`, `agents`, etc. — use the prefixed names as established in the codebase.

**Consumer Group Mapping** (from `redis_utils.py`):
```python
CONSUMER_GROUPS = {
    "client-api": ["eu-solicit:opportunities", "eu-solicit:agents", "eu-solicit:subscriptions", "eu-solicit:billing"],
    "notification-svc": ["eu-solicit:opportunities", "eu-solicit:notifications", "eu-solicit:tasks", "eu-solicit:subscriptions"],
    "data-pipeline": ["eu-solicit:admin"],
    "ai-gateway": ["eu-solicit:admin"],
}
```

The epic AC says consumer groups named `pipeline-workers`, `notification-workers`, `ai-workers`, `admin-workers`. The test-utils uses service names (`client-api`, `notification-svc`, `data-pipeline`, `ai-gateway`) instead. **Use the test-utils names** — they are already established in test code and represent the actual service architecture (each service has its own consumer group on its subscribed streams).

### Event Envelope Format

The canonical envelope format is already defined in `redis_utils.py::build_event_envelope()`. Production `EventPublisher` MUST produce identical structure:

```python
{
    "event_id": str(uuid.uuid4()),          # Auto-generated UUID4
    "event_type": "OpportunitiesIngested",   # Discriminator string
    "payload": json.dumps(payload),          # JSON-serialized payload dict
    "timestamp": datetime.now(tz=UTC).isoformat(),  # UTC ISO-8601
    "correlation_id": correlation_id or str(uuid.uuid4()),  # For distributed tracing
    "source_service": "data-pipeline",       # Originating service name
    "tenant_id": str(tenant_uuid),           # Multi-tenant context
}
```

All values are strings (Redis Streams stores field-value pairs as strings). The `payload` field is a JSON string — consumers must `json.loads()` it after reading.

### EventPublisher Implementation Pattern

```python
class EventPublisher:
    """Publishes events to Redis Streams."""

    def __init__(self, redis_client: redis.asyncio.Redis) -> None:
        self._redis = redis_client

    async def publish(
        self,
        stream: str,
        event_type: str,
        payload: dict[str, Any],
        *,
        source_service: str,
        tenant_id: str | None = None,
        correlation_id: str | None = None,
    ) -> str:
        """Publish event to stream. Returns Redis message ID."""
        envelope = {
            "event_id": str(uuid.uuid4()),
            "event_type": event_type,
            "payload": json.dumps(payload),
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "correlation_id": correlation_id or str(uuid.uuid4()),
            "source_service": source_service,
            "tenant_id": tenant_id or "",
        }
        return await self._redis.xadd(stream, envelope)
```

Key constraints:
- Use `redis.asyncio.Redis` (async client) — all services use async
- `redis>=5.0` is already in pyproject.toml — use `redis.asyncio` module
- Return the message ID string from `XADD`
- `tenant_id` defaults to empty string (not None) since Redis stores strings

### EventConsumer Implementation Pattern

```python
class EventConsumer:
    """Consumes events from Redis Streams consumer groups."""

    def __init__(self, redis_client: redis.asyncio.Redis) -> None:
        self._redis = redis_client

    async def consume(
        self,
        stream: str,
        group: str,
        consumer_name: str,
        *,
        count: int = 10,
        block_ms: int | None = 2000,
    ) -> list[dict[str, Any]]:
        """Read new messages from consumer group. Returns parsed events."""
        ...

    async def ack(self, stream: str, group: str, message_id: str) -> None:
        """Acknowledge message processing."""
        await self._redis.xack(stream, group, message_id)

    async def process_pending(
        self,
        stream: str,
        group: str,
        consumer_name: str,
        *,
        max_retries: int = 3,
    ) -> list[dict[str, Any]]:
        """Move messages exceeding max_retries to DLQ."""
        ...
```

**Blocking vs non-blocking**: `block_ms=None` or `block_ms=0` = non-blocking (immediate return, may return empty list). `block_ms=2000` = block up to 2 seconds waiting for new messages.

### Dead Letter Queue (DLQ) Design

**Flow**: Consumer calls `XREADGROUP` → processing fails → message stays in PEL (Pending Entry List) → next `process_pending()` call checks `XPENDING` for messages with delivery count > `max_retries` → those messages get:
1. `XCLAIM`'d to the current consumer
2. `XADD`'d to `<stream>.dlq` with enriched metadata
3. `XACK`'d from the original stream

**DLQ stream naming**: Append `.dlq` to the original stream name. E.g., `eu-solicit:opportunities.dlq`.

**DLQ envelope**: Original event fields + metadata:
```python
{
    # Original fields preserved
    "event_id": ..., "event_type": ..., "payload": ..., "timestamp": ...,
    "correlation_id": ..., "source_service": ..., "tenant_id": ...,
    # DLQ metadata
    "dlq_reason": "max_retries_exceeded",
    "failure_count": "3",  # String, not int (Redis stores strings)
    "dlq_timestamp": datetime.now(tz=UTC).isoformat(),
    "original_stream": "eu-solicit:opportunities",
    "original_message_id": "1234567890-0",
}
```

**XPENDING usage**: `XPENDING stream group - + count` returns entries with `[message_id, consumer_name, idle_time, delivery_count]`. Filter for `delivery_count > max_retries`.

**XCLAIM**: Before XACK + DLQ add, use `XCLAIM stream group consumer_name min_idle_time message_id` to take ownership of the stuck message.

### Bootstrap Module Design

```python
async def bootstrap_event_bus(redis_client: redis.asyncio.Redis) -> None:
    """Create all streams and consumer groups idempotently."""
    for group_name, stream_keys in CONSUMER_GROUPS.items():
        for stream_key in stream_keys:
            try:
                await redis_client.xgroup_create(
                    stream_key, group_name, id="0", mkstream=True
                )
                log.info("event_bus.group_created", stream=stream_key, group=group_name)
            except redis.ResponseError as e:
                if "BUSYGROUP" in str(e):
                    log.debug("event_bus.group_exists", stream=stream_key, group=group_name)
                else:
                    raise
```

**`mkstream=True`**: Creates the stream if it doesn't exist. This means we don't need separate stream creation — `XGROUP CREATE ... MKSTREAM` handles both stream and group creation.

**Idempotency**: `BUSYGROUP` error means group already exists — catch and log at debug level.

**Bootstrap invocation**: This story creates the bootstrap function. Integration into service startup (`main.py`) is deferred to Story 1.6 (eusolicit-common shared package) which implements the health check lifecycle. For this story, bootstrap is called explicitly in tests.

### Files to CREATE (New)

| File | Purpose |
|------|---------|
| `packages/eusolicit-common/src/eusolicit_common/events/__init__.py` | Events package — exports `EventPublisher`, `EventConsumer`, `bootstrap_event_bus`, `STREAMS`, `CONSUMER_GROUPS` |
| `packages/eusolicit-common/src/eusolicit_common/events/publisher.py` | `EventPublisher` class with `publish()` method |
| `packages/eusolicit-common/src/eusolicit_common/events/consumer.py` | `EventConsumer` class with `consume()`, `ack()`, `process_pending()` methods |
| `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py` | `STREAMS`, `CONSUMER_GROUPS` constants + `bootstrap_event_bus()` function |
| `tests/integration/test_redis_event_bus.py` | Integration tests for event bus (streams, groups, publish/consume, DLQ, idempotency) |

### Files to MODIFY (Existing)

| File | Change |
|------|--------|
| `packages/eusolicit-common/src/eusolicit_common/__init__.py` | Add `from eusolicit_common.events import EventPublisher, EventConsumer, bootstrap_event_bus` to exports |

### Redis Library Usage — `redis.asyncio`

The `redis>=5.0` package provides async support via `redis.asyncio`. Key API:

```python
import redis.asyncio as aioredis

# Creating a client (tests use the conftest fixture)
client = aioredis.from_url("redis://localhost:6379/1", decode_responses=True)

# Stream operations
await client.xadd(stream_name, field_dict)  # Returns message ID str
await client.xreadgroup(groupname=..., consumername=..., streams={stream: ">"}, count=10, block=2000)
await client.xack(stream_name, group_name, message_id)
await client.xgroup_create(stream_name, group_name, id="0", mkstream=True)
await client.xpending_range(stream_name, group_name, min="-", max="+", count=100)
await client.xclaim(stream_name, group_name, consumer_name, min_idle_time=0, message_ids=[msg_id])
await client.xinfo_stream(stream_name)  # Returns dict with stream metadata
await client.xinfo_groups(stream_name)  # Returns list of group dicts
```

**`decode_responses=True`**: The test conftest creates Redis client with `decode_responses=True`, meaning all values are returned as strings (not bytes). The `EventPublisher`/`EventConsumer` classes should work with string values throughout.

### Testing Strategy

**Test file**: `tests/integration/test_redis_event_bus.py`

Tests require Docker Compose Redis running (port 6379). Use `TEST_REDIS_URL` (database 1) for isolation.

**Fixtures from root conftest.py** (DO NOT recreate):
- `redis_client` — async Redis client (session-scoped, `decode_responses=True`)
- `clean_redis` — per-test fixture that flushes DB before/after

**Key tests**:

1. **Bootstrap creates streams**: Call `bootstrap_event_bus(redis)` → `XINFO STREAM` for all 7 stream names → verify each exists
2. **Bootstrap creates consumer groups**: After bootstrap → `XINFO GROUPS` for each stream → verify expected groups present
3. **Idempotent bootstrap**: Call `bootstrap_event_bus` twice → no errors; stream/group counts unchanged
4. **Publish/consume round-trip**: Create `EventPublisher` → publish to `eu-solicit:opportunities` → create `EventConsumer` → consume from `client-api` group → assert all envelope fields (`event_id`, `event_type`, `payload`, `timestamp`, `correlation_id`, `source_service`, `tenant_id`) match; `payload` is valid JSON that deserializes to original dict
5. **DLQ after 3 failures**: Publish event → read via `XREADGROUP` (don't ACK) → simulate 3 delivery attempts → call `process_pending(max_retries=3)` → verify event in `<stream>.dlq` with DLQ metadata; original message ACK'd
6. **Non-blocking consume**: Call `consume(block_ms=0)` on empty stream → returns empty list immediately
7. **Multiple consumers load balance**: Publish 2 events → consume with consumer A (count=1) → consume with consumer B (count=1) → each gets different message
8. **Envelope validation**: Published event has all required fields; `event_id` is valid UUID; `timestamp` is valid ISO-8601; `correlation_id` is UUID

**Mark tests**: `@pytest.mark.integration` and `@pytest.mark.asyncio`

### Test Expectations (from Epic-Level Test Design)

This story maps to the following scenarios from `test-design-epic-01.md`:

| Priority | Test Description | Risk Link | Verification |
|----------|-----------------|-----------|--------------|
| **P0** | Redis Streams publish/consume smoke test | E01-R-002 (Score 6) | Publish JSON event to `eu-solicit:opportunities` → consume from group → assert payload match + envelope fields |
| **P1** | Redis Streams dead letter handling | E01-R-002 (Score 6) | Force consumer failure 3 times → event appears in `<stream>.dlq` stream with original payload |
| **P1** | Redis Streams 7 streams + 4 consumer groups exist | E01-R-007 (Score 4) | Bootstrap → `XINFO STREAM` for all 7 streams; `XINFO GROUPS` for all groups |
| **P2** | Redis Streams idempotent bootstrap | — | Run `bootstrap_event_bus` twice → no errors; streams/groups unchanged |

**Risk E01-R-002 (HIGH, Score 6):** Redis Streams dead letter handling gaps — failed events silently lost when consumer fails 3+ times; no DLQ stream verification. Mitigated by: `EventConsumer.process_pending()` checks `XPENDING` for stuck messages, moves to `<stream>.dlq`, integration test proves the flow.

**Risk E01-R-007 (MEDIUM, Score 4):** Redis Streams event serialization inconsistency — JSON envelope schema drift between EventPublisher and EventConsumer. Mitigated by: both classes share the same envelope field names, Pydantic validation in future Story 1.7 event schemas, round-trip integration test.

**Quality gate**: P0 tests 100% pass rate; P1 tests >= 95% pass rate. DLQ flow verified end-to-end.

### Cross-Story Dependencies

- **Story 1.1** (DONE): Created monorepo structure, `packages/eusolicit-common/` directory, `pyproject.toml` with `redis>=5.0`
- **Story 1.2** (DONE): Created `docker-compose.yml` with Redis 7 container, `.env.example` with `REDIS_URL`/`TEST_REDIS_URL`, test conftest with `redis_client`/`clean_redis` fixtures
- **Story 1.3** (DONE): PostgreSQL schemas/roles — no direct dependency, but established patterns for idempotent infrastructure setup
- **Story 1.4** (DONE): Alembic migration scaffold — no direct dependency
- **Story 1.6** (NEXT): eusolicit-common shared package — will integrate `bootstrap_event_bus()` into service startup lifecycle and health checks
- **Story 1.7** (FUTURE): eusolicit-models — will define Pydantic event schemas (`BaseEvent`, `OpportunitiesIngested`, etc.) that type the `payload` and `event_type` fields

### Previous Story Intelligence (from Story 1.4)

Key learnings from Story 1.4 implementation:

1. **DO NOT TOUCH enforcement**: Story 1.4 was approved with no violations of DO NOT TOUCH files. Follow the same discipline — the Makefile, docker-compose.yml, .env.example, and test-utils are hands-off.

2. **Test structure**: Integration tests go in `tests/integration/` with `@pytest.mark.integration` and `@pytest.mark.asyncio` markers. Use `pytest_asyncio.fixture` for async fixtures.

3. **Existing test count**: 629+ tests in the regression suite from previous stories. Run `make test` after implementation to verify no regressions.

4. **Code review findings from 1.4**: Subshell pattern recommended for Makefile targets (not relevant here). Offline mode gaps found (not relevant here). The key takeaway is: handle error cases explicitly, don't swallow errors silently.

5. **`psycopg2-binary` pattern**: Story 1.4 added sync dependencies alongside async ones. For this story, `redis>=5.0` already provides both sync and async clients — no additional dependency needed.

6. **Schema-per-service isolation**: The pattern of per-service isolation extends to event bus — each service has its own consumer group. Don't create shared/cross-cutting consumer groups.

### Anti-Patterns to Avoid

1. **DO NOT use Celery, RQ, or any task framework** — use `redis-py` directly (`XADD`/`XREADGROUP`/`XACK`). The epic explicitly requires this.
2. **DO NOT create synchronous versions** — all methods must be `async`. The entire platform is async (FastAPI + asyncio).
3. **DO NOT hardcode Redis URLs** — `EventPublisher`/`EventConsumer` receive a `redis.asyncio.Redis` client instance via constructor injection. Connection management is external.
4. **DO NOT create new test fixtures** — reuse `redis_client` and `clean_redis` from `tests/conftest.py`.
5. **DO NOT modify `redis_utils.py` in test-utils** — production code must match its constants, not the other way around.
6. **DO NOT integrate bootstrap into service `main.py`** — that's Story 1.6's responsibility. This story provides the function; 1.6 wires it into startup.
7. **DO NOT add Pydantic models for event schemas** — that's Story 1.7 (`eusolicit-models`). This story uses plain dicts for the envelope.

### Project Structure Notes

**Target directory tree changes from this story** (new items marked with `+`):

```
eusolicit-app/
├── packages/
│   └── eusolicit-common/
│       ├── pyproject.toml                              # NO CHANGE (redis>=5.0 already present)
│       └── src/
│           └── eusolicit_common/
│               ├── __init__.py                         # MODIFY — add events exports
│               └── events/                             # + NEW package
│                   ├── __init__.py                     # + NEW — package exports
│                   ├── publisher.py                    # + NEW — EventPublisher class
│                   ├── consumer.py                     # + NEW — EventConsumer class
│                   └── bootstrap.py                    # + NEW — STREAMS, CONSUMER_GROUPS, bootstrap_event_bus()
│
└── tests/
    └── integration/
        └── + test_redis_event_bus.py                   # + NEW — integration tests
```

### References

- [Source: eusolicit-docs/planning-artifacts/epic-01-infrastructure-foundation.md#S01.05]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-01.md#P0-Redis-Streams-publish-consume, E01-R-002, E01-R-007]
- [Source: eusolicit-docs/test-artifacts/test-design-architecture.md#R-007-Redis-Streams-delivery]
- [Source: eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/redis_utils.py#STREAMS, CONSUMER_GROUPS, build_event_envelope]
- [Source: eusolicit-app/tests/conftest.py#redis_client, clean_redis fixtures]
- [Source: eusolicit-app/packages/eusolicit-common/pyproject.toml#redis>=5.0]
- [Source: eusolicit-app/.env.example#REDIS_URL, TEST_REDIS_URL]
- [Source: eusolicit-docs/implementation-artifacts/1-4-alembic-migration-scaffold.md#Previous-Story-Intelligence]

## Dev Agent Record

### Agent Model Used

Claude Opus 4 (claude-sonnet-4-20250514)

### Debug Log References

- Event loop scope fix: pytest-asyncio 1.3.0 requires `asyncio_default_fixture_loop_scope = "session"` and `asyncio_default_test_loop_scope = "session"` in pyproject.toml for session-scoped async fixtures (redis_client) to work with function-scoped clean_redis fixture. This was the first story to exercise the redis_client/clean_redis fixtures from conftest.py.

### Completion Notes List

- All 72 ATDD tests pass (72/72)
- Zero regressions in existing 1370 test suite (22 pre-existing failures from uninstalled eusolicit_kraftdata/eusolicit_models packages)
- All 7 acceptance criteria met:
  - AC1: 7 Redis Streams created by bootstrap (verified via XINFO STREAM)
  - AC2: 4 consumer groups created on subscribed streams (10 stream-group pairs)
  - AC3: EventPublisher.publish() returns Redis message ID, adds events to streams
  - AC4: EventConsumer.consume() with blocking/non-blocking modes, ack(), process_pending()
  - AC5: Event envelope with all 7 fields (event_id, event_type, payload, timestamp, correlation_id, source_service, tenant_id)
  - AC6: Idempotent bootstrap (BUSYGROUP handled gracefully)
  - AC7: Full publish/consume round-trip with envelope validation
  - AC8: Dead letter handling after 3+ delivery failures with DLQ metadata
- P0 quality gate: 100% pass rate
- P1 quality gate: 100% pass rate
- DO NOT TOUCH files: zero violations
- Required pyproject.toml config addition for pytest-asyncio loop scope (not in DO NOT TOUCH list)
- eusolicit-common package installed in editable mode (`pip install -e`)

### File List

**Created:**
- `packages/eusolicit-common/src/eusolicit_common/events/__init__.py` — Events package exports (EventPublisher, EventConsumer, bootstrap_event_bus, STREAMS, CONSUMER_GROUPS)
- `packages/eusolicit-common/src/eusolicit_common/events/publisher.py` — EventPublisher class with async publish() method
- `packages/eusolicit-common/src/eusolicit_common/events/consumer.py` — EventConsumer class with consume(), ack(), process_pending() methods
- `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py` — STREAMS/CONSUMER_GROUPS constants + bootstrap_event_bus() function

**Modified:**
- `packages/eusolicit-common/src/eusolicit_common/__init__.py` — Added events subpackage exports (EventPublisher, EventConsumer, bootstrap_event_bus)
- `pyproject.toml` — Added asyncio_default_fixture_loop_scope and asyncio_default_test_loop_scope settings for pytest-asyncio 1.3.0 compatibility

## Senior Developer Review

**Reviewed:** 2026-04-06
**Reviewer:** Claude Opus 4 (bmad-code-review — Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** APPROVE
**Summary:** 0 `decision-needed`, 0 `patch`, 7 `defer`, 8 dismissed as noise.

All 8 acceptance criteria verified conformant. STREAMS/CONSUMER_GROUPS match canonical test-utils exactly. 72 ATDD tests pass. No DO NOT TOUCH violations. Implementation follows spec patterns precisely.

### Review Findings

- [x] [Review][Defer] Non-atomic DLQ move (xadd then xack) — crash between ops produces duplicate DLQ entries [consumer.py:143-146] — deferred, at-least-once semantics by design per spec
- [x] [Review][Defer] process_pending hardcoded count=100 — messages beyond 100 in PEL are not inspected per sweep [consumer.py:93] — deferred, pagination enhancement for production hardening
- [x] [Review][Defer] No per-entry error handling in process_pending loop — one Redis failure mid-loop aborts remaining entries [consumer.py:102-156] — deferred, robustness enhancement
- [x] [Review][Defer] STREAMS dict not referenced by CONSUMER_GROUPS — values match today but hardcoded strings could drift [bootstrap.py:34-49] — deferred, maintenance risk
- [x] [Review][Defer] No payload size validation before xadd — oversized payloads cause untyped Redis error [publisher.py:59] — deferred, production hardening
- [x] [Review][Defer] block_ms accepts negative values — undefined Redis behavior for BLOCK with negative int [consumer.py:47-50] — deferred, minor input validation
- [x] [Review][Defer] xclaim returns None data if message was MAXLEN-trimmed — dict(None) would crash process_pending [consumer.py:128] — deferred, relevant when MAXLEN trimming is configured
