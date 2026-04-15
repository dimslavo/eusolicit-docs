# Story 4.7: KraftData Webhook Receiver and Redis Stream Publishing

Status: done

## Story

As a **backend developer on the EU Solicit platform**,
I want **an inbound webhook endpoint (`POST /webhooks/kraftdata`) that validates HMAC-SHA256 signatures, routes events by type, publishes to Redis Streams (`agent.execution.completed` / `agent.execution.failed`), deduplicates by `execution_id`, and logs every receive to `gateway.webhook_log`**,
so that **downstream services can react to asynchronous KraftData agent execution results reliably, securely, and without duplicate processing**.

## Acceptance Criteria

1. Valid webhook with correct `X-Kraftdata-Signature` returns 200 and message appears in the correct Redis Stream (`agent.execution.completed` or `agent.execution.failed`)
2. Invalid or missing `X-Kraftdata-Signature` returns 401 with `{"error": "invalid_signature"}`; nothing published to Redis; `gateway.webhook_log` row inserted with `signature_valid=False`
3. Duplicate `execution_id` within 1 hour returns 200 but is NOT re-published to Redis (idempotency via `SET key EX 3600 NX`)
4. Unknown event type returns 200 and logs WARN; nothing published to Redis
5. Webhook processing completes in < 50ms (from request to 200 response, using mocked Redis + DB)
6. `gateway.webhook_log` contains one entry for every received webhook (valid or invalid signature)
7. Signature comparison uses `hmac.compare_digest()` — constant-time, not `==`
8. Redis Stream message conforms to specified JSON format (see Dev Notes)
9. Webhook log DB write is async/fire-and-forget — a DB failure does NOT prevent the 200 response or Redis publish

## Tasks / Subtasks

- [ ] Task 1: Add `WEBHOOK_SECRET` to config (AC: 1, 2, 7)
  - [ ] 1.1 In `src/ai_gateway/config.py`, add field: `webhook_secret: str = ""`
  - [ ] 1.2 Document the new env var in config docstring alongside existing KraftData settings

- [ ] Task 2: Create Redis singleton module (AC: 1, 3, 8)
  - [ ] 2.1 Create `src/ai_gateway/services/redis_client.py` — module-level singleton pattern identical to `kraftdata_client.py`
  - [ ] 2.2 Implement `init_redis(settings)`, `close_redis()`, `get_redis() -> redis.asyncio.Redis`
  - [ ] 2.3 Use `redis.asyncio.from_url(settings.redis_url, decode_responses=True)` — `decode_responses=True` required so XADD fields are strings not bytes

- [ ] Task 3: Wire Redis lifecycle into `main.py` (AC: 1, 3)
  - [ ] 3.1 Import `init_redis`, `close_redis` from `ai_gateway.services.redis_client`
  - [ ] 3.2 In lifespan startup: call `await init_redis(settings)` **after** `await init_client(settings)` and **before** `await init_registry(...)`
  - [ ] 3.3 In lifespan shutdown (after `yield`): call `await close_redis()` before `await close_client()`
  - [ ] 3.4 Include `webhooks_router` in `app.include_router(webhooks_router.router)` — no prefix

- [ ] Task 4: Create `gateway.webhook_log` Alembic migration (AC: 6)
  - [ ] 4.1 Create `alembic/versions/002_webhook_log.py` (chains from `revision = "001"`, `down_revision = "001"`)
  - [ ] 4.2 Migration `upgrade()` creates table and indexes exactly per the SQL spec in Dev Notes
  - [ ] 4.3 `downgrade()` drops the table (schema="gateway")
  - [ ] 4.4 Create `src/ai_gateway/models/webhook_log.py` with SQLAlchemy mapped class `WebhookLog` (see Dev Notes for column spec)

- [ ] Task 5: Implement webhook router (AC: 1–9)
  - [ ] 5.1 Create `src/ai_gateway/routers/webhooks.py`
  - [ ] 5.2 Implement `POST /webhooks/kraftdata`:
    - [ ] 5.2.1 Read raw body via `await request.body()` BEFORE parsing JSON (body bytes needed for HMAC)
    - [ ] 5.2.2 Validate signature from `X-Kraftdata-Signature` header using `_verify_signature(secret, body, header)` — see Dev Notes for exact implementation
    - [ ] 5.2.3 Reject missing/invalid signature: return 401 `{"error": "invalid_signature"}`; fire-and-forget DB log with `signature_valid=False`; return immediately
    - [ ] 5.2.4 Parse body as `KraftDataWebhookPayload` Pydantic model (see Dev Notes)
    - [ ] 5.2.5 Idempotency check: `SET webhook:dedup:{execution_id} 1 EX 3600 NX` → if returns `None` (key existed), return 200 `{"status": "duplicate"}`; fire-and-forget DB log; return
    - [ ] 5.2.6 Route by `event_type`: `"execution.completed"` → stream `agent.execution.completed`; `"execution.failed"` → stream `agent.execution.failed`; unknown → log WARN, return 200 `{"status": "ignored"}`; fire-and-forget DB log
    - [ ] 5.2.7 Publish to Redis Stream via `XADD` with message JSON (see Dev Notes for exact fields)
    - [ ] 5.2.8 Fire-and-forget: `asyncio.create_task(_log_webhook_to_db(...))` — DB write must not block response
    - [ ] 5.2.9 Return 200 `{"status": "ok"}`

- [ ] Task 6: Create unit tests (AC: 1–9)
  - [ ] 6.1 Create `tests/unit/test_webhooks.py`
  - [ ] 6.2 Tests required (see Dev Notes for test patterns):
    - `test_valid_signature_returns_200_and_publishes`
    - `test_invalid_signature_returns_401_no_publish`
    - `test_missing_signature_header_returns_401`
    - `test_constant_time_comparison_used` (inspect source uses `hmac.compare_digest`)
    - `test_duplicate_execution_id_returns_200_no_republish`
    - `test_execution_completed_routes_to_correct_stream`
    - `test_execution_failed_routes_to_correct_stream`
    - `test_unknown_event_type_returns_200_not_published`
    - `test_redis_xadd_failure_returns_500` (only case where response may fail — Redis publish is synchronous path)
    - `test_db_failure_does_not_block_response`
    - `test_redis_message_format_correct_fields`

## Dev Notes

### Architecture: How This Story Fits

S04.07 is the **inbound event receiver** that completes the KraftData integration loop. All outbound calls (S04.02–S04.06) push work TO KraftData; this story receives completion signals FROM KraftData asynchronously. After this story, Redis Streams `agent.execution.completed` and `agent.execution.failed` carry agent results downstream to any consumer (E05 Data Pipeline, E11 Grants compliance agents, etc.).

**Critical dependency ordering:**
- S04.07 runs **after** S04.06 (circuit breaker) — circuit breaker module (`circuit_breaker.py`, `retry.py`) already exist
- S04.07 runs **before** S04.08 (execution logging/DB schema) — `gateway.webhook_log` table does NOT yet exist
- This story MUST create the `gateway.webhook_log` migration (002_webhook_log.py) because AC6 requires the table to be populated
- S04.08 will create `gateway.agent_executions` in migration 003 and may add additional indexes to webhook_log

### Existing Service Structure (What's Already Built)

```
services/ai-gateway/
├── src/ai_gateway/
│   ├── config.py              # AIGatewaySettings — ADD webhook_secret field here
│   ├── main.py                # FastAPI app + lifespan — ADD Redis init + webhooks router
│   ├── models/
│   │   └── __init__.py        # empty — ADD webhook_log.py here
│   ├── routers/
│   │   ├── admin.py           # GET /admin/registry, GET /admin/circuits
│   │   ├── execution.py       # POST /agents/{id}/run, /workflows/{id}/run, etc. (S04.04+05)
│   │   └── health.py          # GET /health, GET /ready
│   └── services/
│       ├── agent_registry.py  # AgentRegistry, AgentEntry, resolve(), init_registry()
│       ├── circuit_breaker.py # AgentCircuit, CircuitOpenError, init_circuits(), get_circuit()
│       ├── exceptions.py      # KraftDataError hierarchy, AgentNotFoundError, AgentTypeMismatchError
│       ├── kraftdata_client.py # call_kraftdata(), init_client(), close_client(), get_client()
│       └── retry.py           # with_retry() exponential backoff
├── alembic/versions/
│   └── 001_initial.py         # creates gateway._migrations_meta only (revision="001")
└── tests/
    ├── conftest.py             # gateway_engine, gateway_session fixtures
    └── unit/
        ├── test_admin_circuits.py
        ├── test_agent_registry.py
        ├── test_circuit_breaker.py
        ├── test_execution_router.py
        ├── test_health.py
        ├── test_kraftdata_client.py
        ├── test_retry.py
        └── test_sse_proxy.py
```

**New files to create:**
- `src/ai_gateway/services/redis_client.py`
- `src/ai_gateway/routers/webhooks.py`
- `src/ai_gateway/models/webhook_log.py`
- `alembic/versions/002_webhook_log.py`
- `tests/unit/test_webhooks.py`

**Files to modify:**
- `src/ai_gateway/config.py` — add `webhook_secret`
- `src/ai_gateway/main.py` — add Redis lifecycle + webhooks router

### Redis Client Module Pattern

Model `redis_client.py` exactly on `kraftdata_client.py` — module-level singleton, init/close/get functions:

```python
"""Singleton redis.asyncio.Redis client for AI Gateway.

Lifecycle — call init_redis() on startup, close_redis() on shutdown.
Use get_redis() in request handlers.
"""
from __future__ import annotations

import redis.asyncio as aioredis
import structlog
from ai_gateway.config import AIGatewaySettings

log = structlog.get_logger(__name__)
_redis: aioredis.Redis | None = None


def get_redis() -> aioredis.Redis:
    if _redis is None:
        raise RuntimeError("Redis not initialised. Call init_redis() during startup.")
    return _redis


async def init_redis(settings: AIGatewaySettings) -> None:
    global _redis
    _redis = aioredis.from_url(
        settings.redis_url or "redis://localhost:6379/0",
        decode_responses=True,   # CRITICAL: strings, not bytes — needed for XADD field values
    )
    await _redis.ping()
    log.info("redis.connected", url=settings.redis_url)


async def close_redis() -> None:
    global _redis
    if _redis is not None:
        await _redis.aclose()
        _redis = None
        log.info("redis.disconnected")
```

**`decode_responses=True` is critical** — without it, `XADD` field values are bytes, which breaks JSON serialization downstream.

### Signature Validation Implementation

```python
import hashlib
import hmac

def _verify_signature(secret: str, body: bytes, received_header: str | None) -> bool:
    """Constant-time HMAC-SHA256 signature verification.

    KraftData may send signature as "sha256=<hex>" or raw "<hex>".
    Use hmac.compare_digest() — NOT == — to prevent timing attacks (E04-R-001).
    """
    if not received_header:
        return False
    normalized = received_header.removeprefix("sha256=")
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, normalized)
```

**Security note from test design (E04-R-001, score=6 HIGH):**
- Timing attack risk: naive `==` comparison leaks signature length via timing → MUST use `hmac.compare_digest()`
- Log rejected attempts at WARN with timestamp only (NOT the signature value itself — forensic data without exposure)
- Validate the header BEFORE parsing body JSON (reject immediately if missing/invalid to avoid processing cost)

### Webhook Payload Model

```python
from pydantic import BaseModel

class KraftDataWebhookPayload(BaseModel):
    event_type: str
    execution_id: str
    agent_id: str
    result: dict | None = None
    error: str | None = None
```

Place this in `webhooks.py` directly (no need for a separate models file for this simple model).

### Redis Stream Message Format (AC 8)

Published via `await redis.xadd(stream_name, fields_dict)` where fields_dict is:

```json
{
  "execution_id": "<from payload>",
  "agent_id": "<from payload>",
  "agent_name": "<resolved logical name OR empty string if not resolvable>",
  "event_type": "<from payload>",
  "payload": "<full JSON-serialized payload as string>",
  "received_at": "<ISO 8601 UTC datetime>"
}
```

**`agent_name` resolution:** Try `get_registry().resolve_by_kraftdata_id(agent_id)` — if no match, use `""`. Do NOT raise if unresolvable. The agent_registry already has a `resolve()` by logical name; check if it also supports reverse lookup (kraftdata_id → name). If not, do a linear scan of `registry.list_agents()` comparing `entry.kraftdata_id`. Use try/except to handle any registry error gracefully.

**Redis XADD call:**
```python
import json
from datetime import datetime, timezone

stream_name = "agent.execution.completed"  # or "agent.execution.failed"
await redis.xadd(stream_name, {
    "execution_id": payload.execution_id,
    "agent_id": payload.agent_id,
    "agent_name": resolved_name,   # "" if not found
    "event_type": payload.event_type,
    "payload": json.dumps(payload.model_dump()),
    "received_at": datetime.now(timezone.utc).isoformat(),
})
```

### Idempotency Implementation

```python
redis = get_redis()
dedup_key = f"webhook:dedup:{payload.execution_id}"
is_new = await redis.set(dedup_key, "1", ex=3600, nx=True)
# set() with nx=True returns True if key was set, None if key already existed
if not is_new:
    # Duplicate — acknowledge but don't republish
    asyncio.create_task(_log_webhook_to_db(payload, signature_valid=True, processed=False))
    return JSONResponse(status_code=200, content={"status": "duplicate"})
```

### `gateway.webhook_log` Table and Model

**Alembic migration 002_webhook_log.py:**
```python
revision = "002"
down_revision = "001"

def upgrade() -> None:
    op.create_table(
        "webhook_log",
        sa.Column("id", postgresql.UUID(as_uuid=True), primary_key=True,
                  server_default=sa.text("gen_random_uuid()")),
        sa.Column("execution_id", sa.String(255), nullable=True),
        sa.Column("event_type", sa.String(100), nullable=False),
        sa.Column("signature_valid", sa.Boolean, nullable=False),
        sa.Column("payload", postgresql.JSONB, nullable=True),
        sa.Column("received_at", sa.DateTime(timezone=True), nullable=False,
                  server_default=sa.func.now()),
        sa.Column("processed", sa.Boolean, nullable=False, server_default="false"),
        schema="gateway",
    )
    op.create_index("idx_webhook_log_execution_id", "webhook_log",
                    ["execution_id"], schema="gateway")
    op.create_index("idx_webhook_log_event_type", "webhook_log",
                    ["event_type"], schema="gateway")

def downgrade() -> None:
    op.drop_index("idx_webhook_log_event_type", table_name="webhook_log", schema="gateway")
    op.drop_index("idx_webhook_log_execution_id", table_name="webhook_log", schema="gateway")
    op.drop_table("webhook_log", schema="gateway")
```

**SQLAlchemy model (`src/ai_gateway/models/webhook_log.py`):**
```python
from __future__ import annotations
import uuid
from datetime import datetime
from sqlalchemy import Boolean, DateTime, String, func
from sqlalchemy.dialects.postgresql import JSONB, UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class WebhookLog(Base):
    __tablename__ = "webhook_log"
    __table_args__ = {"schema": "gateway"}

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    execution_id: Mapped[str | None] = mapped_column(String(255), nullable=True)
    event_type: Mapped[str] = mapped_column(String(100), nullable=False)
    signature_valid: Mapped[bool] = mapped_column(Boolean, nullable=False)
    payload: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    received_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    processed: Mapped[bool] = mapped_column(Boolean, default=False, nullable=False)
```

### Fire-and-Forget DB Logging Pattern

DB writes MUST NOT block the response (AC 9). Follow the same fire-and-forget pattern used in execution logging:

```python
async def _log_webhook_to_db(
    payload: KraftDataWebhookPayload | None,
    signature_valid: bool,
    processed: bool,
    event_type: str = "",
) -> None:
    """Non-blocking DB log — swallows errors to protect response path."""
    try:
        # Create SQLAlchemy async session and insert WebhookLog row
        # Use gateway_session_factory from app state or create ephemeral session
        ...
    except Exception:
        log.exception("webhook_log.db_write_failed", execution_id=getattr(payload, "execution_id", None))
```

**Session management for fire-and-forget:**
The DB session factory is NOT yet wired into the AI Gateway (S04.08 will add it as middleware). For this story, create an ephemeral `AsyncSession` directly:
```python
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from ai_gateway.config import get_settings

async def _log_webhook_to_db(...) -> None:
    try:
        settings = get_settings()
        engine = create_async_engine(settings.database_url or "")
        async with async_sessionmaker(engine, class_=AsyncSession)() as session:
            async with session.begin():
                session.add(WebhookLog(...))
        await engine.dispose()
    except Exception:
        log.exception("webhook_log.db_write_failed")
```

**Alternative (preferred if simpler):** If creating a per-request engine is too expensive, accept that DB logging will fail gracefully until S04.08 wires the shared session factory. Use a module-level lazy engine:
```python
_db_engine = None

def _get_db_engine():
    global _db_engine
    if _db_engine is None:
        _db_engine = create_async_engine(get_settings().database_url or "")
    return _db_engine
```

Initialize and dispose this engine in the lifespan (alongside Redis and httpx client).

### Webhook Router Skeleton

```python
"""KraftData inbound webhook receiver (S04.07).

POST /webhooks/kraftdata
"""
from __future__ import annotations

import asyncio
import hashlib
import hmac
import json
import structlog
from datetime import datetime, timezone
from typing import Any

from fastapi import APIRouter, Header, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel

from ai_gateway.config import get_settings
from ai_gateway.services.agent_registry import get_registry
from ai_gateway.services.redis_client import get_redis

log = structlog.get_logger(__name__)
router = APIRouter(tags=["Webhooks"])

STREAM_COMPLETED = "agent.execution.completed"
STREAM_FAILED = "agent.execution.failed"


class KraftDataWebhookPayload(BaseModel):
    event_type: str
    execution_id: str
    agent_id: str
    result: dict[str, Any] | None = None
    error: str | None = None


def _verify_signature(secret: str, body: bytes, received: str | None) -> bool:
    if not received:
        return False
    normalized = received.removeprefix("sha256=")
    expected = hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, normalized)


@router.post("/webhooks/kraftdata")
async def receive_kraftdata_webhook(
    request: Request,
    x_kraftdata_signature: str | None = Header(default=None),
) -> JSONResponse:
    body = await request.body()
    settings = get_settings()

    if not _verify_signature(settings.webhook_secret, body, x_kraftdata_signature):
        log.warning("webhook.signature_invalid", timestamp=datetime.now(timezone.utc).isoformat())
        asyncio.create_task(_log_webhook_to_db(None, signature_valid=False, event_type="unknown"))
        return JSONResponse(status_code=401, content={"error": "invalid_signature"})

    payload = KraftDataWebhookPayload.model_validate_json(body)
    redis = get_redis()

    # Idempotency check
    is_new = await redis.set(f"webhook:dedup:{payload.execution_id}", "1", ex=3600, nx=True)
    if not is_new:
        log.info("webhook.duplicate", execution_id=payload.execution_id)
        asyncio.create_task(_log_webhook_to_db(payload, signature_valid=True, processed=False))
        return JSONResponse(status_code=200, content={"status": "duplicate"})

    # Route by event type
    if payload.event_type == "execution.completed":
        stream = STREAM_COMPLETED
    elif payload.event_type == "execution.failed":
        stream = STREAM_FAILED
    else:
        log.warning("webhook.unknown_event_type", event_type=payload.event_type)
        asyncio.create_task(_log_webhook_to_db(payload, signature_valid=True, processed=False))
        return JSONResponse(status_code=200, content={"status": "ignored"})

    # Resolve agent logical name (best-effort, non-blocking on failure)
    agent_name = _resolve_agent_name(payload.agent_id)

    # Publish to Redis Stream (synchronous — this IS the core function)
    await redis.xadd(stream, {
        "execution_id": payload.execution_id,
        "agent_id": payload.agent_id,
        "agent_name": agent_name,
        "event_type": payload.event_type,
        "payload": json.dumps(payload.model_dump()),
        "received_at": datetime.now(timezone.utc).isoformat(),
    })

    asyncio.create_task(_log_webhook_to_db(payload, signature_valid=True, processed=True))
    return JSONResponse(status_code=200, content={"status": "ok"})


def _resolve_agent_name(kraftdata_id: str) -> str:
    """Reverse-lookup kraftdata_id → logical name. Returns '' if not found."""
    try:
        for entry in get_registry().list_agents():
            if entry.kraftdata_id == kraftdata_id:
                return entry.name
    except Exception:  # noqa: BLE001
        pass
    return ""
```

### main.py Changes

In the lifespan startup block, add **in this order**:
1. `await init_redis(settings)` — after `await init_client(settings)`, before `await init_registry(...)`
2. Add Redis shutdown in lifespan shutdown: `await close_redis()` before `await close_client()`

In the router registration section, add:
```python
from ai_gateway.routers import webhooks as webhooks_router
app.include_router(webhooks_router.router)  # no prefix — /webhooks/kraftdata is the full path
```

### Unit Test Patterns

```python
"""Unit tests for POST /webhooks/kraftdata (S04.07)."""
import hashlib
import hmac
import json
from unittest.mock import AsyncMock, MagicMock, patch

import pytest
from fastapi.testclient import TestClient
from httpx import AsyncClient

# Use the FastAPI test client with overridden dependencies
# Mock redis_client.get_redis() to return AsyncMock
# Mock agent_registry.get_registry() to return fixture registry

TEST_SECRET = "test-webhook-secret-abc123"
TEST_EXECUTION_ID = "exec-12345"
TEST_AGENT_ID = "550e8400-e29b-41d4-a716-446655440000"


def _make_signature(secret: str, body: bytes) -> str:
    return hmac.new(secret.encode(), body, hashlib.sha256).hexdigest()


def _make_payload(**kwargs) -> dict:
    defaults = {
        "event_type": "execution.completed",
        "execution_id": TEST_EXECUTION_ID,
        "agent_id": TEST_AGENT_ID,
        "result": {"output": "test"},
    }
    return {**defaults, **kwargs}


@pytest.fixture
def mock_redis():
    r = AsyncMock()
    r.set.return_value = True   # nx=True, key is new
    r.xadd.return_value = "1234-0"
    return r
```

Test for constant-time comparison:
```python
def test_verify_signature_uses_compare_digest():
    """Source inspection: _verify_signature must call hmac.compare_digest."""
    import inspect
    from ai_gateway.routers.webhooks import _verify_signature
    source = inspect.getsource(_verify_signature)
    assert "compare_digest" in source, "Must use hmac.compare_digest() not =="
```

Test for Redis message format:
```python
async def test_redis_message_format_correct_fields(mock_redis):
    # After posting valid webhook, assert xadd was called with correct stream and all required fields
    body = json.dumps(_make_payload()).encode()
    sig = _make_signature(TEST_SECRET, body)
    # ... post request and assert mock_redis.xadd.call_args
    call_args = mock_redis.xadd.call_args
    stream_name, fields = call_args[0]
    assert stream_name == "agent.execution.completed"
    assert "execution_id" in fields
    assert "agent_id" in fields
    assert "agent_name" in fields
    assert "event_type" in fields
    assert "payload" in fields
    assert "received_at" in fields
```

### Test Design Priorities for S04.07 (from test-design-epic-04.md)

**P0 (blocking — must pass):**
- **E04-P0-010**: Valid webhook → 200 + Redis XADD to correct stream + webhook_log row inserted
- **E04-P0-011**: Invalid signature → 401 + NO Redis XADD + webhook_log with `signature_valid=False`
- **E04-P0-012**: Signature uses `hmac.compare_digest` (timing attack prevention — HIGH RISK E04-R-001, score=6)

**P1 (high priority):**
- **E04-P1-012**: Duplicate `execution_id` within 1h → 200 (acknowledged) + XADD called only once (idempotency)
- **E04-P1-013**: Unknown event type → 200 + WARN logged + XADD NOT called

**P2 (medium):**
- **E04-P2-011**: `webhook_log` entry for every webhook (valid + invalid), `signature_valid` field correct

All 3 P0 tests and both P1 tests for this story **must pass before the story can be marked done**.

### Key Risk to Mitigate: Redis Publish-or-Acknowledge Gap (E04-R-003, score=6)

If `XADD` raises mid-flight (Redis failure), the 200 response has NOT yet been sent, so KraftData will NOT receive the acknowledgment. But if we return 200 BEFORE `XADD` completes, we get the inverse problem (KraftData thinks it was processed when it wasn't).

**Decision for this story**: `XADD` is synchronous (not fire-and-forget) — it is on the critical path. If Redis `XADD` raises, the endpoint returns 500 (letting KraftData retry). Only the DB log is fire-and-forget.

Log with:
```python
except Exception as exc:
    log.error("webhook.redis_publish_failed",
              execution_id=payload.execution_id, error=str(exc))
    raise  # re-raise → FastAPI returns 500; KraftData will retry
```

This is consistent with E04-R-003 mitigation: "populate `webhook_log.processed=False` + `error_message` on failure; add observability alert criteria".

### Project Structure Compliance

- Router file goes in `src/ai_gateway/routers/` (consistent with `execution.py`, `admin.py`, `health.py`)
- Service file goes in `src/ai_gateway/services/` (consistent with `kraftdata_client.py`, `retry.py`)
- Model goes in `src/ai_gateway/models/` (extends the currently empty package)
- Alembic migration in `alembic/versions/` with format `002_webhook_log.py`
- Tests go in `tests/unit/test_webhooks.py`
- No new dependencies needed — `redis[hiredis]>=5.0` already in `pyproject.toml`
- Import style: `from __future__ import annotations` at top of every file (established pattern)
- Logger pattern: `log = structlog.get_logger(__name__)` (established pattern)

### Config Env Var Summary

| Var | Setting Field | Default | Notes |
|-----|--------------|---------|-------|
| `WEBHOOK_SECRET` | `webhook_secret` | `""` | HMAC-SHA256 signing secret; empty string → all webhooks fail validation |
| `REDIS_URL` | `redis_url` | inherited from BaseServiceSettings | Already exists |

### References

- Epic story spec: [Source: eusolicit-docs/planning-artifacts/epic-04-ai-gateway-service.md#S04.07]
- Risk E04-R-001 (webhook signature bypass): [Source: eusolicit-docs/test-artifacts/test-design-epic-04.md#High-Priority-Risks]
- Risk E04-R-003 (Redis publish gap): [Source: eusolicit-docs/test-artifacts/test-design-epic-04.md#High-Priority-Risks]
- Test scenarios P0-010, P0-011, P0-012, P1-012, P1-013, P2-011: [Source: eusolicit-docs/test-artifacts/test-design-epic-04.md#Test-Coverage-Plan]
- Redis singleton pattern: modeled on `kraftdata_client.py`
- health.py Redis connection: `redis.asyncio.from_url()` (confirms library version)
- Existing circuit breaker + retry integration: [Source: eusolicit-docs/implementation-artifacts/4-6-circuit-breaker-and-retry-logic.md#Implementation-Notes]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (Claude Code)

### Debug Log References

- Fixed `test_redis_xadd_failure_returns_500`: httpx ASGITransport re-raises server exceptions
  by default; resolved by passing `raise_app_exceptions=False` to ASGITransport in that test.

### Completion Notes List

- All 6 story tasks completed; 88/88 unit tests pass (12 new webhook tests).
- `redis.set()` with `nx=True` returns `True` on new key (not `1`) — adjusted dedup check
  to `if not is_new:` which handles both `True`/`None` semantics correctly.
- Fire-and-forget DB log uses a lazy module-level async engine; safe no-op when
  `DATABASE_URL` is empty (unit-test environment).  Full shared session factory wired in S04.08.
- `_verify_signature` normalises `sha256=<hex>` prefix (KraftData may use either form).

### File List

**New files:**
- `src/ai_gateway/services/redis_client.py`
- `src/ai_gateway/routers/webhooks.py`
- `src/ai_gateway/models/webhook_log.py`
- `alembic/versions/002_webhook_log.py`
- `tests/unit/test_webhooks.py`

**Modified files:**
- `src/ai_gateway/config.py` — added `webhook_secret: str = ""`
- `src/ai_gateway/main.py` — added Redis lifecycle (`init_redis`/`close_redis`) and webhooks router
