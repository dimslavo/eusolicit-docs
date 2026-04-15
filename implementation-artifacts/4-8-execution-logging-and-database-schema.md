# Story 4.8: Execution Logging and Database Schema

Status: review

## Story

As a **backend developer on the EU Solicit platform**,
I want **a `gateway.agent_executions` database table, a SQLAlchemy model, and an async fire-and-forget execution logger that records every agent/team/workflow call with accurate timing, status, and retry metadata — plus a `GET /admin/executions` endpoint for filtered, paginated access to those records**,
so that **every AI gateway call is observable, debuggable, and traceable for SLA analysis without ever blocking the proxy response path**.

## Acceptance Criteria

1. Alembic migration `003_agent_executions.py` creates `gateway.agent_executions` with all 15 columns and 4 indexes (`agent_name`, `status`, `start_time`, `caller_service`) exactly per the SQL spec in Dev Notes; `downgrade()` reverses cleanly
2. Every sync proxy call (`POST /agents/{id}/run`, `/workflows/{id}/run`, `/teams/{id}/run`, `/storage/{id}/files`) creates exactly one `agent_executions` row with accurate `start_time`, `end_time`, `status=success`, `latency_ms>0`, `agent_name`, `agent_kraftdata_id`, `agent_type`, and `caller_service` populated
3. Every streaming execution (`/run-stream` endpoints) creates a row with `is_streaming=True`; `end_time` and `status` are set when the stream completes (not when it starts)
4. Circuit-open rejections are logged with `status=circuit_open`, `latency_ms=0`, and `error_message` describing the circuit state — without contacting KraftData
5. Logging failure (DB error) does NOT cause the proxy call to fail — the error is swallowed, logged at ERROR level with `execution_id`, and the 200/502/504 response is returned normally
6. `GET /admin/executions` accepts optional query filters `agent_name`, `status`, `caller_service`, `start_after` (ISO 8601), `end_before` (ISO 8601), plus pagination params `limit` (default 50, max 200) and `offset` (default 0); returns matching rows as JSON
7. Shared async session factory is wired into `main.py` lifespan — `init_db(settings)` on startup, `close_db()` on shutdown — replacing the per-request lazy engine used in S04.07 webhook logging
8. Every `agent_executions` row includes `retry_count` reflecting the actual number of retry attempts made by the retry layer (0 for first-attempt successes)

## Tasks / Subtasks

- [x] Task 1: Add retry count ContextVar to `retry.py` (AC: 8)
  - [x] 1.1 At the top of `src/ai_gateway/services/retry.py`, add: `from contextvars import ContextVar` and `_retry_count: ContextVar[int] = ContextVar("retry_count", default=0)`
  - [x] 1.2 At the start of `with_retry()` (before the first attempt), reset: `_retry_count.set(0)`
  - [x] 1.3 Inside the retry loop, after each failed attempt, increment: `_retry_count.set(_retry_count.get() + 1)`
  - [x] 1.4 Expose `def get_retry_count() -> int: return _retry_count.get()` as a module-level function

- [x] Task 2: Introduce shared Base and wire shared session factory (AC: 7)
  - [x] 2.1 In `src/ai_gateway/models/__init__.py`, define a shared `Base(DeclarativeBase)` and import `WebhookLog` and `AgentExecution` models so they register against it (see Dev Notes for exact code)
  - [x] 2.2 Update `src/ai_gateway/models/webhook_log.py` to import `Base` from `ai_gateway.models` instead of defining its own local `Base(DeclarativeBase)` — this ensures both models share a single metadata registry
  - [x] 2.3 Create `src/ai_gateway/services/db.py` with `init_db(settings)`, `close_db()`, `get_session_factory() -> async_sessionmaker[AsyncSession]` — module-level singleton pattern, matching `redis_client.py` (see Dev Notes for full code)
  - [x] 2.4 In `src/ai_gateway/main.py` lifespan startup: call `await init_db(settings)` after `await init_redis(settings)` and before `await init_registry(...)`. In shutdown: call `await close_db()` after `await close_redis()`.
  - [x] 2.5 Update `src/ai_gateway/routers/webhooks.py` `_log_webhook_to_db()` to use `get_session_factory()` from `ai_gateway.services.db` instead of the lazy per-request engine — dispose the lazy `_db_engine` reference if it was initialised (set to None)

- [x] Task 3: Create Alembic migration `003_agent_executions.py` (AC: 1)
  - [x] 3.1 Create `alembic/versions/003_agent_executions.py` with `revision = "003"`, `down_revision = "002"`
  - [x] 3.2 `upgrade()` creates `gateway.agent_executions` with all 15 columns exactly per the SQL spec in Dev Notes; `gen_random_uuid()` as `id` default; `NOT NULL` constraints honoured
  - [x] 3.3 `upgrade()` creates all 4 indexes: `idx_agent_executions_agent_name`, `idx_agent_executions_status`, `idx_agent_executions_start_time`, `idx_agent_executions_caller`
  - [x] 3.4 `downgrade()` drops all 4 indexes then drops the table

- [x] Task 4: Create `AgentExecution` SQLAlchemy model (AC: 1, 2, 3, 4)
  - [x] 4.1 Create `src/ai_gateway/models/agent_execution.py` with `class AgentExecution(Base)` importing `Base` from `ai_gateway.models`
  - [x] 4.2 Map all 15 columns matching the migration schema: `id`, `execution_id` (String 255, unique, non-null), `agent_name`, `agent_kraftdata_id` (UUID), `agent_type`, `caller_service`, `request_id`, `is_streaming`, `start_time`, `end_time`, `status`, `latency_ms`, `error_message`, `retry_count`, `created_at` (see Dev Notes for full mapping)

- [x] Task 5: Implement `execution_logger.py` service (AC: 2, 3, 4, 5, 8)
  - [x] 5.1 Create `src/ai_gateway/services/execution_logger.py` with two public functions: `log_execution_start(...)` and `log_execution_complete(...)` (see Dev Notes for full signatures)
  - [x] 5.2 `log_execution_start()` inserts a row with `status="pending"` and `start_time=now()`; uses `asyncio.create_task()` for the DB write (fire-and-forget); returns a `str` tracking id (`request_id`) used to correlate the update
  - [x] 5.3 `log_execution_complete()` updates the row matched by `execution_id=tracking_id` with `end_time`, `status`, `latency_ms`, `error_message`, `retry_count`; also fire-and-forget via `asyncio.create_task()`
  - [x] 5.4 Both functions wrap the DB operation in `try/except Exception`: on failure, emit `log.error("execution_log.db_write_failed", execution_id=..., error=...)` and return without raising (AC 5)
  - [x] 5.5 For `circuit_open` calls, provide `log_circuit_open_rejection(agent_name, agent_kraftdata_id, agent_type, caller_service, request_id)` which calls `log_execution_start()` immediately followed by `log_execution_complete()` with `status="circuit_open"`, `latency_ms=0`, `retry_count=0`

- [x] Task 6: Integrate execution logging into `execution.py` endpoints (AC: 2, 3, 4)
  - [x] 6.1 Import `log_execution_start`, `log_execution_complete`, `log_circuit_open_rejection` from `ai_gateway.services.execution_logger`; import `get_retry_count` from `ai_gateway.services.retry`
  - [x] 6.2 In `run_agent()`, `run_workflow()`, `run_team()`: immediately before `call_kraftdata()`, call `log_execution_start(agent_name=id, agent_kraftdata_id=kraftdata_id, agent_type=..., caller_service=caller_service, request_id=request_id, is_streaming=False)` storing `start_time = datetime.now(timezone.utc)`. In the success path after `call_kraftdata()`, call `log_execution_complete(execution_id=request_id, status="success", start_time=start_time, retry_count=get_retry_count())`. In the exception path, call `log_execution_complete(execution_id=request_id, status=..., error_message=str(exc), ...)` before re-raising
  - [x] 6.3 In `run_agent()` and `run_workflow()` exception handlers: when `CircuitOpenError` is caught, call `log_circuit_open_rejection(...)` instead of the standard complete call
  - [x] 6.4 For `upload_storage_file()`: same pattern; `agent_type="storage"`, `agent_name=id` (no registry lookup)
  - [x] 6.5 For `run_agent_stream()` and `run_workflow_stream()`: `log_execution_start()` before creating the `StreamingResponse`; pass `is_streaming=True`. Yield `StreamingResponse` wrapped in a generator that calls `log_execution_complete()` in a `finally` block after the stream ends — see Dev Notes for streaming logger wrapper pattern
  - [x] 6.6 Map exception types to status strings: `CircuitOpenError` → `"circuit_open"`, `KraftDataTimeoutError` → `"timeout"`, `KraftDataAPIError` → `"failed"`, `KraftDataConnectionError` → `"failed"`, uncaught exception → `"failed"`

- [x] Task 7: Implement `GET /admin/executions` endpoint (AC: 6)
  - [x] 7.1 In `src/ai_gateway/routers/admin.py`, import `AsyncSession`, `select` and the `AgentExecution` model; add `get_session_factory` import from `ai_gateway.services.db`
  - [x] 7.2 Add `GET /executions` route with query params: `agent_name: str | None = None`, `status: str | None = None`, `caller_service: str | None = None`, `start_after: datetime | None = None`, `end_before: datetime | None = None`, `limit: int = Query(50, ge=1, le=200)`, `offset: int = Query(0, ge=0)`
  - [x] 7.3 Build SQLAlchemy `select(AgentExecution)` with `where()` clauses applied only for non-None params; add `.order_by(AgentExecution.start_time.desc())`, `.limit(limit)`, `.offset(offset)`
  - [x] 7.4 Return JSON list with all fields serialised (use `model_dump()` or dict comprehension with ISO 8601 datetimes and UUID strings)

- [x] Task 8: Create unit tests (AC: 1–8)
  - [x] 8.1 Create `tests/unit/test_execution_logger.py`
  - [x] 8.2 Tests required (see Dev Notes for patterns):
    - `test_log_execution_creates_pending_row`
    - `test_log_execution_complete_updates_row`
    - `test_circuit_open_logged_with_zero_latency`
    - `test_db_failure_swallowed_call_still_succeeds`
    - `test_retry_count_surfaced_in_log_row`
    - `test_streaming_row_has_is_streaming_true`
    - `test_streaming_end_time_set_on_stream_close`
  - [x] 8.3 Add `test_admin_executions_filters` to `tests/unit/test_execution_router.py` (or create `tests/unit/test_admin_executions.py`): verify filter params produce correct `WHERE` clauses via mock session

## Dev Notes

### Architecture: How This Story Fits

S04.08 is the **audit and observability layer** that wraps every outbound KraftData call. All previous stories (S04.02–S04.07) make calls to KraftData but log nothing beyond a structlog line. After this story, every execution is persisted to `gateway.agent_executions` for SLA tracking, debugging, and operational dashboards.

**Critical dependency ordering:**
- S04.08 runs **after** S04.07 (webhook receiver) — `gateway.webhook_log` already exists as migration `002`
- S04.08 creates `gateway.agent_executions` as migration `003`
- S04.08 also delivers the **shared session factory** that S04.07 deferred (S04.07's `_log_webhook_to_db` used a lazy per-request engine as a stopgap; this story replaces it)
- S04.09 (rate limiter) will add `status=rate_limited` rows using the same logger

### Existing Service Structure

```
services/ai-gateway/
├── src/ai_gateway/
│   ├── config.py              # AIGatewaySettings (no changes needed)
│   ├── main.py                # ADD: init_db/close_db lifecycle
│   ├── models/
│   │   ├── __init__.py        # currently empty — ADD shared Base here
│   │   └── webhook_log.py     # MODIFY: import Base from ai_gateway.models
│   ├── routers/
│   │   ├── admin.py           # ADD: GET /admin/executions
│   │   ├── execution.py       # MODIFY: integrate logging in all 6 endpoints
│   │   ├── health.py          # no changes
│   │   └── webhooks.py        # MODIFY: switch to shared session factory
│   └── services/
│       ├── agent_registry.py  # no changes
│       ├── circuit_breaker.py # no changes
│       ├── db.py              # NEW: shared async session factory
│       ├── exceptions.py      # no changes
│       ├── execution_logger.py # NEW: async fire-and-forget logger
│       ├── kraftdata_client.py # no changes
│       ├── redis_client.py    # no changes
│       └── retry.py           # MODIFY: add ContextVar retry count tracking
├── alembic/versions/
│   ├── 001_initial.py         # no changes (creates gateway._migrations_meta)
│   ├── 002_webhook_log.py     # no changes (creates gateway.webhook_log)
│   └── 003_agent_executions.py # NEW: creates gateway.agent_executions
└── tests/
    ├── conftest.py             # no changes needed
    └── unit/
        ├── test_execution_logger.py   # NEW
        └── [all existing tests unchanged]
```

### Shared Base in `models/__init__.py`

```python
"""AI Gateway ORM models package.

Defines a single shared DeclarativeBase so all models register
against the same SQLAlchemy MetaData instance (required for Alembic
autogenerate and testcontainers migration application).
"""
from __future__ import annotations

from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass


# Import models so they register against Base.metadata
from ai_gateway.models.webhook_log import WebhookLog  # noqa: E402, F401
from ai_gateway.models.agent_execution import AgentExecution  # noqa: E402, F401

__all__ = ["Base", "AgentExecution", "WebhookLog"]
```

**Update `webhook_log.py` import:**
```python
# Before (S04.07):
from sqlalchemy.orm import DeclarativeBase
class Base(DeclarativeBase): pass

# After (S04.08):
from ai_gateway.models import Base  # shared Base
```

Remove the local `Base` class from `webhook_log.py` entirely.

### `db.py` — Shared Async Session Factory

```python
"""Shared async SQLAlchemy session factory for the AI Gateway (S04.08).

Lifecycle:
    init_db(settings)   — called during lifespan startup
    close_db()          — called during lifespan shutdown
    get_session_factory() — used by execution_logger.py and routers
"""
from __future__ import annotations

import structlog
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from ai_gateway.config import AIGatewaySettings

log = structlog.get_logger(__name__)

_engine = None
_session_factory: async_sessionmaker[AsyncSession] | None = None


def get_session_factory() -> async_sessionmaker[AsyncSession]:
    if _session_factory is None:
        raise RuntimeError("DB not initialised. Call init_db() during startup.")
    return _session_factory


async def init_db(settings: AIGatewaySettings) -> None:
    global _engine, _session_factory
    if not settings.database_url:
        log.warning("db.no_database_url_configured", note="execution logging disabled")
        return
    _engine = create_async_engine(
        settings.database_url,
        pool_size=5,
        max_overflow=10,
        pool_pre_ping=True,
        echo=False,
    )
    _session_factory = async_sessionmaker(
        _engine, class_=AsyncSession, expire_on_commit=False
    )
    log.info("db.connected", url=settings.database_url.split("@")[-1])  # hide credentials


async def close_db() -> None:
    global _engine, _session_factory
    if _engine is not None:
        await _engine.dispose()
        _engine = None
        _session_factory = None
        log.info("db.disconnected")
```

**`database_url` not set → `init_db` no-ops gracefully** — `get_session_factory()` will raise `RuntimeError`, which `execution_logger.py` catches and silences (AC 5). This ensures unit tests without a real DB work out of the box.

### `003_agent_executions.py` Migration

```python
"""Create gateway.agent_executions table (S04.08).

Revision ID: 003
Revises: 002
Create Date: 2026-04-14
"""
from __future__ import annotations

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql

revision = "003"
down_revision = "002"
branch_labels = None
depends_on = None

# Partition-readiness note: this table will be partitioned by start_time monthly
# when write volume warrants it (estimated >10M rows/month threshold).


def upgrade() -> None:
    op.create_table(
        "agent_executions",
        sa.Column(
            "id",
            postgresql.UUID(as_uuid=True),
            primary_key=True,
            server_default=sa.text("gen_random_uuid()"),
        ),
        sa.Column("execution_id", sa.String(255), nullable=False, unique=True),
        sa.Column("agent_name", sa.String(100), nullable=False),
        sa.Column("agent_kraftdata_id", postgresql.UUID(as_uuid=True), nullable=False),
        sa.Column("agent_type", sa.String(20), nullable=False),
        sa.Column("caller_service", sa.String(50), nullable=False),
        sa.Column("request_id", sa.String(255), nullable=True),
        sa.Column("is_streaming", sa.Boolean, nullable=False, server_default="false"),
        sa.Column("start_time", sa.DateTime(timezone=True), nullable=False),
        sa.Column("end_time", sa.DateTime(timezone=True), nullable=True),
        sa.Column("status", sa.String(20), nullable=False),
        sa.Column("latency_ms", sa.Integer, nullable=True),
        sa.Column("error_message", sa.Text, nullable=True),
        sa.Column("retry_count", sa.Integer, nullable=False, server_default="0"),
        sa.Column(
            "created_at",
            sa.DateTime(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        schema="gateway",
    )
    op.create_index(
        "idx_agent_executions_agent_name", "agent_executions", ["agent_name"], schema="gateway"
    )
    op.create_index(
        "idx_agent_executions_status", "agent_executions", ["status"], schema="gateway"
    )
    op.create_index(
        "idx_agent_executions_start_time", "agent_executions", ["start_time"], schema="gateway"
    )
    op.create_index(
        "idx_agent_executions_caller", "agent_executions", ["caller_service"], schema="gateway"
    )


def downgrade() -> None:
    op.drop_index("idx_agent_executions_caller", table_name="agent_executions", schema="gateway")
    op.drop_index("idx_agent_executions_start_time", table_name="agent_executions", schema="gateway")
    op.drop_index("idx_agent_executions_status", table_name="agent_executions", schema="gateway")
    op.drop_index("idx_agent_executions_agent_name", table_name="agent_executions", schema="gateway")
    op.drop_table("agent_executions", schema="gateway")
```

### `agent_execution.py` SQLAlchemy Model

```python
"""SQLAlchemy ORM model for gateway.agent_executions (S04.08)."""
from __future__ import annotations

import uuid
from datetime import datetime

from sqlalchemy import Boolean, DateTime, Integer, String, Text, func
from sqlalchemy.dialects.postgresql import UUID
from sqlalchemy.orm import Mapped, mapped_column

from ai_gateway.models import Base


class AgentExecution(Base):
    """One row per agent/team/workflow/storage call through the gateway."""

    __tablename__ = "agent_executions"
    __table_args__ = {"schema": "gateway"}

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    execution_id: Mapped[str] = mapped_column(String(255), unique=True, nullable=False)
    agent_name: Mapped[str] = mapped_column(String(100), nullable=False)
    agent_kraftdata_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    agent_type: Mapped[str] = mapped_column(String(20), nullable=False)
    caller_service: Mapped[str] = mapped_column(String(50), nullable=False)
    request_id: Mapped[str | None] = mapped_column(String(255), nullable=True)
    is_streaming: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False)
    start_time: Mapped[datetime] = mapped_column(DateTime(timezone=True), nullable=False)
    end_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    status: Mapped[str] = mapped_column(String(20), nullable=False)
    latency_ms: Mapped[int | None] = mapped_column(Integer, nullable=True)
    error_message: Mapped[str | None] = mapped_column(Text, nullable=True)
    retry_count: Mapped[int] = mapped_column(Integer, nullable=False, default=0)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), nullable=False
    )
```

**`agent_kraftdata_id` type:** Note that `agent_kraftdata_id` is a UUID in PostgreSQL but callers pass a string from `_resolve_kraftdata_id()`. Use `uuid.UUID(kraftdata_id_str)` when constructing the `AgentExecution` object. For storage endpoint calls, use `uuid.UUID(int=0)` as a sentinel (zero UUID) since storage resources are not in the agent registry.

### `execution_logger.py` — Fire-and-Forget Logger

```python
"""Async fire-and-forget execution logger for gateway.agent_executions (S04.08).

All DB writes happen inside asyncio.create_task() so they never block the
proxy response path.  DB errors are swallowed after ERROR-level logging.
"""
from __future__ import annotations

import asyncio
import uuid
from datetime import datetime, timezone

import structlog

from ai_gateway.models.agent_execution import AgentExecution

log = structlog.get_logger(__name__)

# Status constants
STATUS_PENDING = "pending"
STATUS_SUCCESS = "success"
STATUS_FAILED = "failed"
STATUS_TIMEOUT = "timeout"
STATUS_CIRCUIT_OPEN = "circuit_open"


def log_execution_start(
    *,
    execution_id: str,
    agent_name: str,
    agent_kraftdata_id: str,
    agent_type: str,
    caller_service: str,
    request_id: str | None,
    is_streaming: bool,
    start_time: datetime,
) -> None:
    """Fire-and-forget: insert status=pending row. Non-blocking."""
    asyncio.create_task(
        _insert_execution(
            execution_id=execution_id,
            agent_name=agent_name,
            agent_kraftdata_id=agent_kraftdata_id,
            agent_type=agent_type,
            caller_service=caller_service,
            request_id=request_id,
            is_streaming=is_streaming,
            start_time=start_time,
        )
    )


def log_execution_complete(
    *,
    execution_id: str,
    status: str,
    end_time: datetime,
    latency_ms: int,
    error_message: str | None = None,
    retry_count: int = 0,
) -> None:
    """Fire-and-forget: update existing row with final status. Non-blocking."""
    asyncio.create_task(
        _update_execution(
            execution_id=execution_id,
            status=status,
            end_time=end_time,
            latency_ms=latency_ms,
            error_message=error_message,
            retry_count=retry_count,
        )
    )


async def _insert_execution(**kwargs) -> None:
    try:
        from ai_gateway.services.db import get_session_factory  # late import avoids circular
        factory = get_session_factory()
        # Convert string UUID to uuid.UUID object
        kraftdata_uuid = _to_uuid(kwargs.get("agent_kraftdata_id", ""))
        async with factory() as session:
            async with session.begin():
                session.add(
                    AgentExecution(
                        execution_id=kwargs["execution_id"],
                        agent_name=kwargs["agent_name"],
                        agent_kraftdata_id=kraftdata_uuid,
                        agent_type=kwargs["agent_type"],
                        caller_service=kwargs["caller_service"],
                        request_id=kwargs.get("request_id"),
                        is_streaming=kwargs.get("is_streaming", False),
                        start_time=kwargs["start_time"],
                        status=STATUS_PENDING,
                    )
                )
    except Exception:  # noqa: BLE001
        log.error("execution_log.insert_failed", execution_id=kwargs.get("execution_id"))


async def _update_execution(**kwargs) -> None:
    try:
        from ai_gateway.services.db import get_session_factory
        from sqlalchemy import update as sa_update
        factory = get_session_factory()
        async with factory() as session:
            async with session.begin():
                await session.execute(
                    sa_update(AgentExecution)
                    .where(AgentExecution.execution_id == kwargs["execution_id"])
                    .values(
                        status=kwargs["status"],
                        end_time=kwargs["end_time"],
                        latency_ms=kwargs["latency_ms"],
                        error_message=kwargs.get("error_message"),
                        retry_count=kwargs.get("retry_count", 0),
                    )
                )
    except Exception:  # noqa: BLE001
        log.error("execution_log.update_failed", execution_id=kwargs.get("execution_id"))


def _to_uuid(value: str) -> uuid.UUID:
    """Convert string to UUID; return zero UUID sentinel on failure."""
    try:
        return uuid.UUID(value)
    except (ValueError, AttributeError):
        return uuid.UUID(int=0)
```

### Integrating Logging into `execution.py` Endpoints

**Sync endpoint pattern** (shown for `run_agent`; apply identically to `run_workflow`, `run_team`, `upload_storage_file`):

```python
from datetime import datetime, timezone
from ai_gateway.services.execution_logger import (
    log_execution_start, log_execution_complete,
    STATUS_SUCCESS, STATUS_FAILED, STATUS_TIMEOUT, STATUS_CIRCUIT_OPEN,
)
from ai_gateway.services.retry import get_retry_count

@router.post("/agents/{id}/run", response_model=AgentRunResponse)
async def run_agent(id, body, request, caller_service) -> JSONResponse:
    kraftdata_id = _resolve_kraftdata_id(id, expected_type=None)
    request_id = _get_or_generate_request_id(request.headers.get("x-request-id"))
    forward_headers = {"X-Caller-Service": caller_service, "X-Request-ID": request_id}

    start_time = datetime.now(timezone.utc)
    log_execution_start(
        execution_id=request_id,
        agent_name=id,
        agent_kraftdata_id=kraftdata_id,
        agent_type="agent",
        caller_service=caller_service,
        request_id=request_id,
        is_streaming=False,
        start_time=start_time,
    )

    try:
        resp = await call_kraftdata(
            "POST", f"/client/api/v1/agents/{kraftdata_id}/run",
            json=body.model_dump(), headers=forward_headers, agent_name=id,
        )
    except (CircuitOpenError, KraftDataAPIError, KraftDataTimeoutError, KraftDataConnectionError) as exc:
        end_time = datetime.now(timezone.utc)
        latency_ms = int((end_time - start_time).total_seconds() * 1000)
        status = STATUS_CIRCUIT_OPEN if isinstance(exc, CircuitOpenError) else (
            STATUS_TIMEOUT if isinstance(exc, KraftDataTimeoutError) else STATUS_FAILED
        )
        log_execution_complete(
            execution_id=request_id, status=status, end_time=end_time,
            latency_ms=0 if isinstance(exc, CircuitOpenError) else latency_ms,
            error_message=str(exc), retry_count=get_retry_count(),
        )
        raise _handle_kraftdata_error(exc)

    end_time = datetime.now(timezone.utc)
    latency_ms = int((end_time - start_time).total_seconds() * 1000)
    log_execution_complete(
        execution_id=request_id, status=STATUS_SUCCESS, end_time=end_time,
        latency_ms=latency_ms, retry_count=get_retry_count(),
    )
    result = AgentRunResponse(**resp.json())
    return JSONResponse(content=result.model_dump(mode="json"), headers={"X-Request-ID": request_id})
```

**Agent type for each endpoint:**
- `run_agent` / `run_agent_stream`: `agent_type="agent"`
- `run_workflow` / `run_workflow_stream`: `agent_type="workflow"`
- `run_team`: `agent_type="team"`
- `upload_storage_file`: `agent_type="storage"`, `agent_kraftdata_id=id` (raw UUID, no registry)

**For `upload_storage_file`**, the `id` parameter is already a raw UUID, so pass it directly as `agent_kraftdata_id`. Use `agent_name=f"storage:{id}"` for the log row.

### Streaming Endpoint Logging Pattern

The challenge with streaming: `log_execution_complete()` must be called after the generator finishes, not when `StreamingResponse` is returned.

```python
@router.post("/agents/{id}/run-stream")
async def run_agent_stream(id, body, request, caller_service) -> StreamingResponse:
    kraftdata_id = _resolve_kraftdata_id(id, expected_type=None)
    request_id = _get_or_generate_request_id(request.headers.get("x-request-id"))
    forward_headers = {"X-Caller-Service": caller_service, "X-Request-ID": request_id}

    start_time = datetime.now(timezone.utc)
    log_execution_start(
        execution_id=request_id, agent_name=id, agent_kraftdata_id=kraftdata_id,
        agent_type="agent", caller_service=caller_service, request_id=request_id,
        is_streaming=True, start_time=start_time,
    )

    async def _logging_generator() -> AsyncGenerator[bytes, None]:
        try:
            async for chunk in _sse_stream_generator(
                f"/client/api/v1/agents/{kraftdata_id}/run-stream",
                body.model_dump(), forward_headers,
            ):
                yield chunk
            end_time = datetime.now(timezone.utc)
            log_execution_complete(
                execution_id=request_id, status=STATUS_SUCCESS, end_time=end_time,
                latency_ms=int((end_time - start_time).total_seconds() * 1000),
                retry_count=0,
            )
        except Exception as exc:  # noqa: BLE001
            end_time = datetime.now(timezone.utc)
            log_execution_complete(
                execution_id=request_id, status=STATUS_FAILED, end_time=end_time,
                latency_ms=int((end_time - start_time).total_seconds() * 1000),
                error_message=str(exc), retry_count=0,
            )
            raise

    return StreamingResponse(
        _logging_generator(), media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no", "X-Request-ID": request_id},
    )
```

This pattern wraps the SSE generator with a logging wrapper. The `log_execution_complete()` call is in the `try/finally` semantics of the generator body — it fires after the last chunk is yielded. Since it uses `asyncio.create_task()` internally, it's fire-and-forget and never blocks the last chunk delivery.

### `GET /admin/executions` Endpoint

```python
from datetime import datetime
from sqlalchemy import select
from fastapi import Query
from ai_gateway.models.agent_execution import AgentExecution
from ai_gateway.services.db import get_session_factory

@router.get("/executions", summary="Query execution log with filters")
async def list_executions(
    agent_name: str | None = None,
    status: str | None = None,
    caller_service: str | None = None,
    start_after: datetime | None = None,
    end_before: datetime | None = None,
    limit: int = Query(50, ge=1, le=200),
    offset: int = Query(0, ge=0),
) -> list[dict]:
    stmt = select(AgentExecution)
    if agent_name:
        stmt = stmt.where(AgentExecution.agent_name == agent_name)
    if status:
        stmt = stmt.where(AgentExecution.status == status)
    if caller_service:
        stmt = stmt.where(AgentExecution.caller_service == caller_service)
    if start_after:
        stmt = stmt.where(AgentExecution.start_time >= start_after)
    if end_before:
        stmt = stmt.where(AgentExecution.end_time <= end_before)
    stmt = stmt.order_by(AgentExecution.start_time.desc()).limit(limit).offset(offset)

    try:
        factory = get_session_factory()
        async with factory() as session:
            result = await session.execute(stmt)
            rows = result.scalars().all()
            return [_serialise_execution(r) for r in rows]
    except RuntimeError:
        # DB not configured — return empty list gracefully
        return []


def _serialise_execution(row: AgentExecution) -> dict:
    return {
        "id": str(row.id),
        "execution_id": row.execution_id,
        "agent_name": row.agent_name,
        "agent_kraftdata_id": str(row.agent_kraftdata_id),
        "agent_type": row.agent_type,
        "caller_service": row.caller_service,
        "request_id": row.request_id,
        "is_streaming": row.is_streaming,
        "start_time": row.start_time.isoformat() if row.start_time else None,
        "end_time": row.end_time.isoformat() if row.end_time else None,
        "status": row.status,
        "latency_ms": row.latency_ms,
        "error_message": row.error_message,
        "retry_count": row.retry_count,
        "created_at": row.created_at.isoformat() if row.created_at else None,
    }
```

### Retry Count ContextVar in `retry.py`

Add to the **top** of the existing `retry.py`:
```python
from contextvars import ContextVar

_retry_count: ContextVar[int] = ContextVar("retry_count", default=0)


def get_retry_count() -> int:
    """Return retry attempt count for the current async context."""
    return _retry_count.get()
```

Inside `with_retry()`, at the start of the function (before the first attempt):
```python
_retry_count.set(0)
```

Inside the retry loop, after a failure triggers a retry:
```python
_retry_count.set(_retry_count.get() + 1)
```

**ContextVar isolation:** In an asyncio application, each request task has its own context copy, so concurrent requests don't interfere. `asyncio.create_task()` creates a child context — set the ContextVar BEFORE calling `create_task()` if the child task needs to read it.

### `main.py` Changes

In the lifespan startup block, add **in this order** (after `await init_redis(settings)`, before `await init_registry(...)`):
```python
from ai_gateway.services.db import close_db, init_db
# ...
await init_db(settings)   # S04.08: initialise shared async session factory
```

In the lifespan shutdown block (after `yield`), add:
```python
await close_db()   # S04.08: dispose SQLAlchemy engine
```

Shutdown order should be:
1. `await close_db()`
2. `await close_redis()`
3. `await close_client()`

### Unit Test Patterns for `test_execution_logger.py`

```python
"""Unit tests for execution_logger.py (S04.08)."""
from __future__ import annotations

import asyncio
from datetime import datetime, timezone
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

START_TIME = datetime(2026, 4, 14, 12, 0, 0, tzinfo=timezone.utc)


@pytest.fixture
def mock_session_factory():
    """Mock async_sessionmaker that yields a mock AsyncSession."""
    session = AsyncMock()
    session.__aenter__ = AsyncMock(return_value=session)
    session.__aexit__ = AsyncMock(return_value=False)
    session.begin = MagicMock(return_value=AsyncMock(__aenter__=AsyncMock(return_value=None), __aexit__=AsyncMock()))
    session.add = MagicMock()
    session.execute = AsyncMock()

    factory = MagicMock(return_value=session)
    return factory, session


@pytest.mark.asyncio
async def test_log_execution_creates_pending_row(mock_session_factory):
    """log_execution_start() fires an insert task with status='pending'."""
    factory, session = mock_session_factory
    with patch("ai_gateway.services.execution_logger.get_session_factory", return_value=factory):
        from ai_gateway.services.execution_logger import log_execution_start
        log_execution_start(
            execution_id="req-001", agent_name="executive-summary",
            agent_kraftdata_id="550e8400-e29b-41d4-a716-446655440000",
            agent_type="agent", caller_service="client-api", request_id="req-001",
            is_streaming=False, start_time=START_TIME,
        )
        # Allow the create_task coroutine to execute
        await asyncio.sleep(0)
        session.add.assert_called_once()
        added = session.add.call_args[0][0]
        assert added.status == "pending"
        assert added.execution_id == "req-001"
        assert added.is_streaming is False


@pytest.mark.asyncio
async def test_db_failure_swallowed_call_still_succeeds(mock_session_factory):
    """DB insert failure must NOT propagate to caller."""
    factory, session = mock_session_factory
    session.begin.side_effect = Exception("DB connection failed")
    with patch("ai_gateway.services.execution_logger.get_session_factory", return_value=factory):
        from ai_gateway.services.execution_logger import log_execution_start
        # Must not raise — fire-and-forget swallows
        log_execution_start(
            execution_id="req-002", agent_name="test-agent",
            agent_kraftdata_id="00000000-0000-4000-8000-000000000001",
            agent_type="agent", caller_service="test", request_id="req-002",
            is_streaming=False, start_time=START_TIME,
        )
        await asyncio.sleep(0)  # Let task run
        # No exception raised — test passes if we reach here
```

Test for circuit-open logging:
```python
@pytest.mark.asyncio
async def test_circuit_open_logged_with_zero_latency(mock_session_factory):
    """circuit_open status uses latency_ms=0 regardless of wall time."""
    factory, session = mock_session_factory
    updated_values = {}

    async def capture_execute(stmt):
        # Capture the values from the update statement
        updated_values.update(stmt.compile().params if hasattr(stmt, 'compile') else {})
    session.execute = AsyncMock(side_effect=capture_execute)

    with patch("ai_gateway.services.execution_logger.get_session_factory", return_value=factory):
        from ai_gateway.services.execution_logger import log_execution_complete, STATUS_CIRCUIT_OPEN
        log_execution_complete(
            execution_id="req-003", status=STATUS_CIRCUIT_OPEN,
            end_time=datetime.now(timezone.utc), latency_ms=0, retry_count=0,
        )
        await asyncio.sleep(0)
        session.execute.assert_called_once()
```

### Test Design Priority References (from `test-design-epic-04.md`)

**P0 — Must pass before story can close:**
- **E04-P0-013**: Every sync proxy call creates exactly one `agent_executions` row with accurate `start_time`, `end_time`, `status=success`, `latency_ms>0`, `caller_service` populated — *integration test with testcontainers, implemented in S04.10*

**P1 — High priority:**
- **E04-P1-018**: Streaming execution creates row with `is_streaming=True`; `end_time` set when stream completes — *integration test with testcontainers, S04.10*
- **E04-P1-019**: Circuit-open rejection logged with `status=circuit_open` and `latency_ms=0` — *unit test in this story*

**P2 — Medium priority:**
- **E04-P2-009**: `GET /admin/executions?agent_name=&status=` returns filtered results; pagination works — *unit test in this story*
- **E04-P2-010**: Logging failure doesn't cause proxy call to fail — *unit test in this story*
- **E04-P2-015**: Alembic migration creates both tables with correct columns and all indexes — *integration test, testcontainers, S04.10*

**P0-013, P1-018, P2-015** rely on testcontainers (real PostgreSQL) and are deferred to S04.10 integration test suite. The unit tests in this story cover **P1-019, P2-009, P2-010** and all AC items verifiable with mocked DB.

### Risk Mitigation from `test-design-epic-04.md`

**E04-R-007 — Execution log async write backlog (score=4, OPS):** Fire-and-forget DB writes use `asyncio.create_task()`; under sustained load, pending tasks could exhaust the connection pool. Mitigation in this story:
- Use a bounded connection pool (`pool_size=5, max_overflow=10`) in `db.py`
- The pool timeout will cause `_insert_execution()` to raise `TimeoutError` → caught by `except Exception` → logged at ERROR, silently dropped
- This is acceptable for initial deployment; backpressure enhancement is backlog

**E04-R-003 follow-up (Redis publish gap):** After S04.08 wires the shared session factory, `webhooks.py` should update its `_log_webhook_to_db()` to use `get_session_factory()` instead of the lazy per-request engine (Task 2.5 above). This removes the per-request engine overhead from the webhook path and ensures consistent DB access.

### Config Env Var Summary

No new env vars required for this story. Uses existing `DATABASE_URL` from `BaseServiceSettings`. If `DATABASE_URL` is empty (unit test environments), `init_db()` no-ops and `get_session_factory()` raises `RuntimeError`, which execution_logger catches and silences.

### References

- Epic story spec: `eusolicit-docs/planning-artifacts/epic-04-ai-gateway-service.md#S04.08`
- Test design (E04-P0-013, P1-018/019, P2-009/010/015): `eusolicit-docs/test-artifacts/test-design-epic-04.md`
- Risk E04-R-007 (async write backlog): `eusolicit-docs/test-artifacts/test-design-epic-04.md#Medium-Priority-Risks`
- S04.07 completion notes (lazy engine to replace): `eusolicit-docs/implementation-artifacts/4-7-kraftdata-webhook-receiver-and-redis-stream-publishing.md#Completion-Notes`
- S04.09 rate limiter story will add `status=rate_limited` using the same `execution_logger.py` API

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (claude-sonnet-4-6)

### Debug Log References

- Fixed `test_webhooks.py` to patch `ai_gateway.services.db.get_session_factory` (RuntimeError) instead of removed `ai_gateway.routers.webhooks._get_db_engine` (S04.07 stopgap removed in 2.5)
- Fixed test `test_execution_logger.py` patch target: `get_session_factory` is late-imported inside `_insert_execution`/`_update_execution`, so patch target is `ai_gateway.services.db.get_session_factory` not the logger module
- Fixed `KraftDataAPIError("...", status_code=500)` signature in test (keyword-only `status_code` arg)
- Fixed import order in `main.py` (`ruff` I001 + F401 for `get_all_circuits` unused import removed)

### Completion Notes List

- **Task 1**: Added `ContextVar[int]` `_retry_count` and `get_retry_count()` to `retry.py`; `with_retry()` resets to 0 at start, increments after each failed attempt
- **Task 2**: Created `services/db.py` singleton factory; updated `models/__init__.py` with shared `Base`; migrated `webhook_log.py` to use shared Base; wired `init_db`/`close_db` into `main.py` lifespan; replaced lazy engine in `webhooks.py` with shared factory
- **Task 3**: Created `003_agent_executions.py` migration with 15 columns and 4 indexes; downgrade drops indexes then table
- **Task 4**: Created `models/agent_execution.py` with all 15 ORM columns mapped to PostgreSQL types
- **Task 5**: Created `execution_logger.py` with fire-and-forget `log_execution_start()`, `log_execution_complete()`, `log_circuit_open_rejection()`; all DB errors swallowed via `except Exception`
- **Task 6**: Rewrote `execution.py` integrating execution logging in all 6 endpoints; sync endpoints use `_exc_to_status()` helper; streaming endpoints use `_logging_generator()` wrapper pattern
- **Task 7**: Added `GET /admin/executions` to `admin.py` with 5 filters + pagination; returns `[]` gracefully when DB not configured (RuntimeError)
- **Task 8**: 9 tests in `test_execution_logger.py`; 8 tests in `test_admin_executions.py`; updated `test_webhooks.py` to use new patch targets; **105 tests pass, 0 failures**

### File List

**New files:**
- `eusolicit-app/services/ai-gateway/alembic/versions/003_agent_executions.py`
- `eusolicit-app/services/ai-gateway/src/ai_gateway/models/agent_execution.py`
- `eusolicit-app/services/ai-gateway/src/ai_gateway/services/db.py`
- `eusolicit-app/services/ai-gateway/src/ai_gateway/services/execution_logger.py`
- `eusolicit-app/services/ai-gateway/tests/unit/test_execution_logger.py`
- `eusolicit-app/services/ai-gateway/tests/unit/test_admin_executions.py`

**Modified files:**
- `eusolicit-app/services/ai-gateway/src/ai_gateway/models/__init__.py` — shared Base + model imports
- `eusolicit-app/services/ai-gateway/src/ai_gateway/models/webhook_log.py` — import Base from `ai_gateway.models`
- `eusolicit-app/services/ai-gateway/src/ai_gateway/services/retry.py` — add ContextVar retry count + `get_retry_count()`
- `eusolicit-app/services/ai-gateway/src/ai_gateway/main.py` — add `init_db`/`close_db` lifecycle
- `eusolicit-app/services/ai-gateway/src/ai_gateway/routers/execution.py` — integrate execution logging in all 6 endpoints
- `eusolicit-app/services/ai-gateway/src/ai_gateway/routers/admin.py` — add `GET /admin/executions`
- `eusolicit-app/services/ai-gateway/src/ai_gateway/routers/webhooks.py` — switch `_log_webhook_to_db` to shared session factory
- `eusolicit-app/services/ai-gateway/tests/unit/test_webhooks.py` — update patch target from `_get_db_engine` to `get_session_factory`

### Change Log

- 2026-04-14: Implemented S04.08 execution logging and database schema. Created `gateway.agent_executions` table (Alembic 003), SQLAlchemy model, shared DB session factory, async fire-and-forget execution logger, `GET /admin/executions` endpoint, integrated logging into all 6 proxy endpoints, added ContextVar retry count tracking. 17 new tests; 105 total unit tests passing.
