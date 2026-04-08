# Story 1.6: eusolicit-common Shared Package

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **developer on the EU Solicit platform**,
I want **the `eusolicit-common` shared package to provide configuration management, structured logging, base middleware classes (auth, audit, rate limit), a standardized health check endpoint, exception handlers, and Pydantic base models**,
so that **every service starts from a consistent, tested foundation — eliminating boilerplate, enforcing observability standards (structured JSON logs with correlation IDs), and ensuring uniform error responses, health reporting, and middleware behavior across all 5 services**.

## Acceptance Criteria

1. **Config loader**: reads from environment variables with `.env` fallback, validates with Pydantic `BaseSettings`, supports per-service overrides via `env_prefix`
2. **Logging**: `structlog` configured for JSON output in production, pretty-print in development; includes `correlation_id`, `service_name`, `tenant_id` in every log line
3. **Auth middleware base**: abstract class that extracts and validates JWT from `Authorization` header, populates request state with user context (`user_id`, `company_id`, `tenant_id`, `role`)
4. **Audit middleware**: logs every request/response with method, path, status code, duration, user ID, and tenant ID
5. **Rate limit middleware base**: abstract class with pluggable backend (Redis-based), configurable per-route limits
6. **Health check endpoint**: `GET /healthz` returns `{status, version, uptime, checks: {db, redis, ...}}` with dependency checks
7. **Exception handlers**: maps domain exceptions to HTTP responses (400/401/403/404/409/422/500) with consistent error envelope `{error, message, details, correlation_id}`
8. **Pydantic base models**: `BaseSchema` (with `model_config` for camelCase alias generation), `PaginatedResponse[T]`, `ErrorResponse`, `HealthResponse`
9. Unit tests for config loader, exception handlers, and Pydantic models (>= 80% coverage on this package)

## Tasks / Subtasks

- [x] Task 1: Create Pydantic base models (AC: 8)
  - [x] 1.1 Create `packages/eusolicit-common/src/eusolicit_common/schemas.py`
  - [x] 1.2 Implement `BaseSchema` with `model_config = ConfigDict(populate_by_name=True, alias_generator=to_camel)` for camelCase JSON keys
  - [x] 1.3 Implement `PaginatedResponse[T]` generic model with `items: list[T]`, `total: int`, `page: int`, `page_size: int`, `total_pages: int`
  - [x] 1.4 Implement `ErrorResponse` with `error: str`, `message: str`, `details: dict[str, Any] | None`, `correlation_id: str`
  - [x] 1.5 Implement `HealthResponse` with `status: str`, `version: str`, `uptime: float`, `checks: dict[str, HealthCheckResult]`
  - [x] 1.6 Implement `HealthCheckResult` with `status: str`, `latency_ms: float | None`, `error: str | None`

- [x] Task 2: Create config loader (AC: 1)
  - [x] 2.1 Create `packages/eusolicit-common/src/eusolicit_common/config.py`
  - [x] 2.2 Implement `BaseServiceSettings(BaseSettings)` with common fields: `service_name: str`, `environment: str` (default "development"), `debug: bool` (default False), `log_level: str` (default "INFO"), `database_url: str | None`, `redis_url: str | None`, `version: str` (default "0.1.0")
  - [x] 2.3 Configure `model_config = SettingsConfigDict(env_file=".env", env_file_encoding="utf-8", extra="ignore")` for `.env` fallback
  - [x] 2.4 Support per-service overrides via `env_prefix` — e.g., `CLIENT_API_DATABASE_URL` overrides `DATABASE_URL` when `env_prefix="CLIENT_API_"`
  - [x] 2.5 Add `get_settings()` cached factory function returning the settings singleton

- [x] Task 3: Create structured logging (AC: 2)
  - [x] 3.1 Create `packages/eusolicit-common/src/eusolicit_common/logging.py`
  - [x] 3.2 Implement `setup_logging(service_name: str, *, environment: str = "development", log_level: str = "INFO")` that configures structlog
  - [x] 3.3 Production mode: JSON renderer (`structlog.processors.JSONRenderer`)
  - [x] 3.4 Development mode: pretty-print renderer (`structlog.dev.ConsoleRenderer`)
  - [x] 3.5 Add processors for: `correlation_id` (from context var), `service_name` (bound), `tenant_id` (from context var), `timestamper` (ISO format, UTC)
  - [x] 3.6 Implement `get_logger(name: str | None = None)` that returns a bound structlog logger
  - [x] 3.7 Implement context var helpers: `set_correlation_id(cid)`, `get_correlation_id()`, `set_tenant_id(tid)`, `get_tenant_id()`

- [x] Task 4: Create exception handlers (AC: 7)
  - [x] 4.1 Create `packages/eusolicit-common/src/eusolicit_common/exceptions.py`
  - [x] 4.2 Define domain exception hierarchy: `AppException(Exception)` base with `status_code`, `error`, `message`, `details`
  - [x] 4.3 Subclasses: `BadRequestError(400)`, `UnauthorizedError(401)`, `ForbiddenError(403)`, `NotFoundError(404)`, `ConflictError(409)`, `ValidationError(422)`, `InternalError(500)`
  - [x] 4.4 Implement `app_exception_handler(request, exc)` → JSONResponse using `ErrorResponse` envelope
  - [x] 4.5 Implement `unhandled_exception_handler(request, exc)` → 500 with generic message (no stack trace leak)
  - [x] 4.6 Implement `register_exception_handlers(app: FastAPI)` utility that wires all handlers to the app

- [x] Task 5: Create auth middleware base (AC: 3)
  - [x] 5.1 Create `packages/eusolicit-common/src/eusolicit_common/middleware/__init__.py`
  - [x] 5.2 Create `packages/eusolicit-common/src/eusolicit_common/middleware/auth.py`
  - [x] 5.3 Implement `AbstractAuthMiddleware(BaseHTTPMiddleware)` with abstract `async verify_token(token: str) -> UserContext` method
  - [x] 5.4 Define `UserContext` dataclass/model: `user_id: str`, `company_id: str | None`, `tenant_id: str | None`, `role: str`, `email: str | None`
  - [x] 5.5 Middleware extracts `Bearer` token from `Authorization` header, calls `verify_token()`, sets `request.state.user` with UserContext
  - [x] 5.6 On missing/invalid token: raises `UnauthorizedError` (from exceptions.py)
  - [x] 5.7 Support configurable `exclude_paths` (list of paths to skip auth, e.g., `/healthz`, `/docs`, `/openapi.json`)

- [x] Task 6: Create audit middleware (AC: 4)
  - [x] 6.1 Create `packages/eusolicit-common/src/eusolicit_common/middleware/audit.py`
  - [x] 6.2 Implement `AuditMiddleware(BaseHTTPMiddleware)` that logs every request/response
  - [x] 6.3 Log fields: `method`, `path`, `status_code`, `duration_ms`, `user_id` (from request.state if available), `tenant_id` (from request.state if available), `correlation_id`
  - [x] 6.4 Extract or generate `correlation_id` from `X-Request-ID` header, set in context var for downstream use
  - [x] 6.5 Set `X-Request-ID` response header with the correlation_id
  - [x] 6.6 Use structlog for all audit log output

- [x] Task 7: Create rate limit middleware base (AC: 5)
  - [x] 7.1 Create `packages/eusolicit-common/src/eusolicit_common/middleware/rate_limit.py`
  - [x] 7.2 Implement `AbstractRateLimitMiddleware(BaseHTTPMiddleware)` with abstract `async check_rate_limit(key: str, limit: int, window_seconds: int) -> RateLimitResult`
  - [x] 7.3 Define `RateLimitResult` dataclass: `allowed: bool`, `remaining: int`, `reset_at: float`
  - [x] 7.4 Implement `NoOpRateLimiter` backend that always allows (initial placeholder)
  - [x] 7.5 Support per-route configuration via a `rate_limits: dict[str, tuple[int, int]]` mapping (path pattern → (limit, window))

- [x] Task 8: Create health check endpoint (AC: 6)
  - [x] 8.1 Create `packages/eusolicit-common/src/eusolicit_common/health.py`
  - [x] 8.2 Implement `HealthCheck` class with `register_check(name: str, check_fn: Callable)` for pluggable dependency checks
  - [x] 8.3 Implement `async run_checks() -> HealthResponse` that runs all registered checks, measures latency, catches errors
  - [x] 8.4 Implement `create_health_router(health_check: HealthCheck) -> APIRouter` returning a FastAPI router with `GET /healthz`
  - [x] 8.5 Built-in check factories: `create_db_check(engine)`, `create_redis_check(redis_client)` — each executes a lightweight ping query
  - [x] 8.6 Track `uptime` from service start time (module-level `_start_time`)
  - [x] 8.7 Status is "healthy" if all checks pass, "degraded" if any non-critical check fails, "unhealthy" if any critical check fails

- [x] Task 9: Update package exports (AC: all)
  - [x] 9.1 Update `packages/eusolicit-common/src/eusolicit_common/__init__.py` — export all new public symbols
  - [x] 9.2 Update `packages/eusolicit-common/src/eusolicit_common/middleware/__init__.py` — export middleware classes

- [x] Task 10: Unit tests (AC: 9)
  - [x] 10.1 Create `tests/unit/test_eusolicit_common_config.py` — test config loader: valid config loads, missing required var raises, .env fallback works, env_prefix overrides
  - [x] 10.2 Create `tests/unit/test_eusolicit_common_exceptions.py` — test exception handlers: each domain exception maps to correct HTTP status, error envelope matches `ErrorResponse` schema, unhandled exceptions return 500 without stack trace
  - [x] 10.3 Create `tests/unit/test_eusolicit_common_schemas.py` — test Pydantic base models: `BaseSchema` camelCase alias round-trip, `PaginatedResponse[T]` serialization, `ErrorResponse` and `HealthResponse` validation
  - [x] 10.4 Create `tests/unit/test_eusolicit_common_logging.py` — test logging setup: JSON mode vs pretty-print mode, context var binding (correlation_id, service_name, tenant_id)
  - [x] 10.5 Create `tests/unit/test_eusolicit_common_health.py` — test health check: all-pass → healthy, one-fail → degraded, critical-fail → unhealthy, `HealthResponse` format correct
  - [x] 10.6 Create `tests/unit/test_eusolicit_common_middleware.py` — test audit middleware: logs correct fields, generates correlation_id if missing, sets response header; test auth middleware: abstract method called, UserContext populated, excluded paths skipped; test rate limit: NoOpRateLimiter always allows
  - [x] 10.7 Achieve >= 80% coverage on `packages/eusolicit-common/`

## Dev Notes

### CRITICAL: Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `docker-compose.yml` | Full Docker Compose stack (9+ containers) | **DO NOT TOUCH** |
| `.env.example` | All env vars for services, DB, Redis, MinIO, ClamAV, auth | **DO NOT TOUCH** |
| `Makefile` | Docker + test + migration targets | **DO NOT TOUCH** |
| `packages/eusolicit-common/src/eusolicit_common/events/` | EventPublisher, EventConsumer, bootstrap (Story 1.5) | **DO NOT TOUCH** |
| `packages/eusolicit-common/pyproject.toml` | Dependencies: fastapi>=0.115, structlog>=24.0, pydantic>=2.0, pydantic-settings>=2.0, redis>=5.0 | **DO NOT TOUCH** — all deps needed already present |
| `packages/eusolicit-test-utils/` | Test factories, auth helpers, redis/db utils | **DO NOT TOUCH** — reuse, don't duplicate |
| `tests/conftest.py` | Session/function-scoped fixtures for DB, Redis, HTTP clients, auth | **DO NOT TOUCH** — reuse these fixtures |
| `services/*/src/*/main.py` | Minimal FastAPI skeletons with placeholder `/healthz` | **DO NOT TOUCH** — service integration deferred; this story creates reusable components only |

### CRITICAL: Do NOT Wire Into Services Yet

This story creates the reusable library components. **Do NOT** modify any `services/*/src/*/main.py` files. Services will integrate these components in later stories (E02+). The current minimal `healthz` endpoint in each service is intentional — it will be replaced when services adopt eusolicit-common's health router.

### Config Loader Design

The config loader uses Pydantic v2 `BaseSettings` with `pydantic-settings` (already in pyproject.toml).

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class BaseServiceSettings(BaseSettings):
    """Base configuration shared by all EU Solicit services."""
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",  # Ignore unknown env vars
    )

    service_name: str = "unknown"
    environment: str = "development"  # development | staging | production
    debug: bool = False
    log_level: str = "INFO"
    version: str = "0.1.0"

    # Infrastructure — overridable per-service via subclass + env_prefix
    database_url: str | None = None
    redis_url: str | None = None
```

**Per-service override pattern**: Each service creates a subclass:
```python
class ClientApiSettings(BaseServiceSettings):
    model_config = SettingsConfigDict(env_prefix="CLIENT_API_")
    service_name: str = "client-api"
```

This allows `CLIENT_API_DATABASE_URL` to populate `database_url`. The `.env.example` already defines per-service env vars (e.g., `CLIENT_API_DATABASE_URL`, `REDIS_URL`).

**Cached singleton**: Use `@functools.lru_cache` for `get_settings()` so config is loaded once.

### Structured Logging Design

Use `structlog>=24.0` (already in pyproject.toml). Key design:

```python
import structlog
from contextvars import ContextVar

_correlation_id_var: ContextVar[str | None] = ContextVar("correlation_id", default=None)
_tenant_id_var: ContextVar[str | None] = ContextVar("tenant_id", default=None)

def setup_logging(
    service_name: str,
    *,
    environment: str = "development",
    log_level: str = "INFO",
) -> None:
    """Configure structlog for the service."""
    shared_processors = [
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.StackInfoRenderer(),
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        _add_service_context,  # Binds service_name, correlation_id, tenant_id
    ]

    if environment == "development":
        renderer = structlog.dev.ConsoleRenderer()
    else:
        renderer = structlog.processors.JSONRenderer()

    structlog.configure(
        processors=[*shared_processors, renderer],
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.getLevelName(log_level.upper())
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

**Context var injection**: The `_add_service_context` processor reads `_correlation_id_var` and `_tenant_id_var` to inject into every log line. The audit middleware (Task 6) sets these context vars per-request.

**`get_logger()`**: Returns `structlog.get_logger()` — callers bind additional context as needed.

### Exception Handlers Design

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class AppException(Exception):
    """Base for all domain exceptions."""
    def __init__(
        self,
        message: str,
        *,
        error: str = "app_error",
        status_code: int = 500,
        details: dict[str, Any] | None = None,
    ) -> None:
        self.message = message
        self.error = error
        self.status_code = status_code
        self.details = details
        super().__init__(message)

class NotFoundError(AppException):
    def __init__(self, message: str = "Resource not found", **kwargs):
        super().__init__(message, error="not_found", status_code=404, **kwargs)

# ... BadRequestError(400), UnauthorizedError(401), ForbiddenError(403),
#     ConflictError(409), ValidationError(422), InternalError(500)
```

**Handler registration**:
```python
def register_exception_handlers(app: FastAPI) -> None:
    app.add_exception_handler(AppException, app_exception_handler)
    app.add_exception_handler(Exception, unhandled_exception_handler)
```

`app_exception_handler` → returns `ErrorResponse` JSON with the exception's fields + correlation_id from context var.
`unhandled_exception_handler` → logs the full traceback via structlog, returns generic 500 `ErrorResponse` with no stack trace.

### Auth Middleware Design

```python
from abc import ABC, abstractmethod
from starlette.middleware.base import BaseHTTPMiddleware
from dataclasses import dataclass

@dataclass
class UserContext:
    user_id: str
    company_id: str | None = None
    tenant_id: str | None = None
    role: str = "member"
    email: str | None = None

class AbstractAuthMiddleware(BaseHTTPMiddleware, ABC):
    def __init__(self, app, *, exclude_paths: list[str] | None = None):
        super().__init__(app)
        self.exclude_paths = exclude_paths or ["/healthz", "/docs", "/openapi.json"]

    @abstractmethod
    async def verify_token(self, token: str) -> UserContext:
        """Verify JWT and return user context. Raise UnauthorizedError on failure."""
        ...

    async def dispatch(self, request, call_next):
        if request.url.path in self.exclude_paths:
            return await call_next(request)
        auth = request.headers.get("Authorization", "")
        if not auth.startswith("Bearer "):
            raise UnauthorizedError("Missing or invalid Authorization header")
        token = auth[7:]  # Strip "Bearer "
        user_ctx = await self.verify_token(token)
        request.state.user = user_ctx
        # Set tenant_id context var for logging
        set_tenant_id(user_ctx.tenant_id)
        return await call_next(request)
```

**Abstract only** — no concrete JWT validation. Concrete implementation deferred to auth epic (Google OAuth2 + JWT RS256 per architecture). The test-utils `auth.py` already uses HS256 for test JWTs — this middleware will work with any signing algorithm since verification is delegated to the abstract method.

### Audit Middleware Design

```python
import time
import uuid
from starlette.middleware.base import BaseHTTPMiddleware

class AuditMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        # Extract or generate correlation_id
        correlation_id = request.headers.get("X-Request-ID") or str(uuid.uuid4())
        set_correlation_id(correlation_id)

        start = time.perf_counter()
        response = await call_next(request)
        duration_ms = (time.perf_counter() - start) * 1000

        # Extract user context if auth middleware ran
        user_id = getattr(getattr(request.state, "user", None), "user_id", None)
        tenant_id = get_tenant_id()

        log.info(
            "http.request",
            method=request.method,
            path=request.url.path,
            status_code=response.status_code,
            duration_ms=round(duration_ms, 2),
            user_id=user_id,
            tenant_id=tenant_id,
            correlation_id=correlation_id,
        )

        response.headers["X-Request-ID"] = correlation_id
        return response
```

**Middleware ordering**: Audit middleware should be added FIRST (outermost), so it captures the full request lifecycle including auth middleware time. Services will add middleware in order: `app.add_middleware(AuditMiddleware)` then `app.add_middleware(AuthMiddleware)` — Starlette executes them LIFO (last-added runs first in the request path), so audit wraps auth.

### Rate Limit Middleware Design

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class RateLimitResult:
    allowed: bool
    remaining: int
    reset_at: float  # Unix timestamp

class AbstractRateLimitMiddleware(BaseHTTPMiddleware, ABC):
    def __init__(self, app, *, rate_limits: dict[str, tuple[int, int]] | None = None):
        super().__init__(app)
        self.rate_limits = rate_limits or {}  # path_pattern → (limit, window_seconds)

    @abstractmethod
    async def check_rate_limit(self, key: str, limit: int, window_seconds: int) -> RateLimitResult:
        ...

class NoOpRateLimiter(AbstractRateLimitMiddleware):
    """Placeholder that always allows requests."""
    async def check_rate_limit(self, key: str, limit: int, window_seconds: int) -> RateLimitResult:
        return RateLimitResult(allowed=True, remaining=limit, reset_at=0.0)
```

The architecture specifies Redis INCR-based rate limiting for usage gates. The concrete Redis implementation will be built in the subscription/tier gating epic. This story establishes the interface only.

### Health Check Design

```python
from fastapi import APIRouter
from collections.abc import Callable, Awaitable
import time

class HealthCheck:
    def __init__(self, version: str = "0.1.0") -> None:
        self._checks: dict[str, tuple[Callable[[], Awaitable[None]], bool]] = {}
        self._version = version
        self._start_time = time.time()

    def register_check(
        self, name: str, check_fn: Callable[[], Awaitable[None]], *, critical: bool = True
    ) -> None:
        self._checks[name] = (check_fn, critical)

    async def run_checks(self) -> HealthResponse:
        # Run all checks, capture latency and errors
        # Determine overall status: healthy / degraded / unhealthy
        ...

def create_health_router(health_check: HealthCheck) -> APIRouter:
    router = APIRouter()

    @router.get("/healthz")
    async def healthz() -> HealthResponse:
        return await health_check.run_checks()

    return router
```

**Built-in check factories** (convenience wrappers):
```python
def create_db_check(engine: AsyncEngine) -> Callable:
    async def check():
        async with engine.connect() as conn:
            await conn.execute(text("SELECT 1"))
    return check

def create_redis_check(redis_client) -> Callable:
    async def check():
        await redis_client.ping()
    return check
```

These are provided as helpers but NOT wired — each service decides which checks to register based on its dependencies.

### Pydantic Base Models — camelCase Alias Pattern

```python
from pydantic import BaseModel, ConfigDict
from pydantic.alias_generators import to_camel
from typing import Generic, TypeVar

T = TypeVar("T")

class BaseSchema(BaseModel):
    model_config = ConfigDict(
        populate_by_name=True,
        alias_generator=to_camel,
        from_attributes=True,
    )
```

**`populate_by_name=True`**: Allows both `snake_case` (Python) and `camelCase` (JSON) field access.
**`from_attributes=True`**: Enables `BaseSchema.model_validate(orm_object)` for SQLAlchemy models.

`PaginatedResponse[T]` is generic — services instantiate it as `PaginatedResponse[OpportunityDTO]`.

### Files to CREATE (New)

| File | Purpose |
|------|---------|
| `packages/eusolicit-common/src/eusolicit_common/config.py` | `BaseServiceSettings` with Pydantic BaseSettings, `.env` fallback, `get_settings()` |
| `packages/eusolicit-common/src/eusolicit_common/logging.py` | `setup_logging()`, `get_logger()`, context var helpers for correlation_id/tenant_id |
| `packages/eusolicit-common/src/eusolicit_common/exceptions.py` | Domain exception hierarchy + FastAPI exception handlers + `register_exception_handlers()` |
| `packages/eusolicit-common/src/eusolicit_common/schemas.py` | `BaseSchema`, `PaginatedResponse[T]`, `ErrorResponse`, `HealthResponse`, `HealthCheckResult` |
| `packages/eusolicit-common/src/eusolicit_common/health.py` | `HealthCheck` class, `create_health_router()`, `create_db_check()`, `create_redis_check()` |
| `packages/eusolicit-common/src/eusolicit_common/middleware/__init__.py` | Package exports: `AbstractAuthMiddleware`, `AuditMiddleware`, `AbstractRateLimitMiddleware`, `NoOpRateLimiter`, `UserContext`, `RateLimitResult` |
| `packages/eusolicit-common/src/eusolicit_common/middleware/auth.py` | `AbstractAuthMiddleware`, `UserContext` dataclass |
| `packages/eusolicit-common/src/eusolicit_common/middleware/audit.py` | `AuditMiddleware` — logs all requests with timing, correlation_id, user context |
| `packages/eusolicit-common/src/eusolicit_common/middleware/rate_limit.py` | `AbstractRateLimitMiddleware`, `RateLimitResult`, `NoOpRateLimiter` |
| `tests/unit/test_eusolicit_common_config.py` | Unit tests for config loader |
| `tests/unit/test_eusolicit_common_exceptions.py` | Unit tests for exception handlers |
| `tests/unit/test_eusolicit_common_schemas.py` | Unit tests for Pydantic base models |
| `tests/unit/test_eusolicit_common_logging.py` | Unit tests for structured logging |
| `tests/unit/test_eusolicit_common_health.py` | Unit tests for health check |
| `tests/unit/test_eusolicit_common_middleware.py` | Unit tests for all three middleware classes |

### Files to MODIFY (Existing)

| File | Change |
|------|--------|
| `packages/eusolicit-common/src/eusolicit_common/__init__.py` | Add exports for all new modules: config, logging, exceptions, schemas, health, middleware |

### Testing Strategy

**Test markers**: `@pytest.mark.unit` for all tests in this story. No `@pytest.mark.integration` — these are pure unit tests with mocks.

**Framework**: `pytest` + `pytest-asyncio` (asyncio_mode = "auto" per root pyproject.toml). Use `unittest.mock.AsyncMock` for async dependencies.

**Test fixtures from root conftest.py** (DO NOT recreate):
- **Do NOT** use `redis_client`, `db_engine`, etc. for unit tests — mock all external dependencies
- Unit tests must run without Docker Compose

**Coverage**: `pytest --cov=packages/eusolicit-common --cov-report=term-missing` — target >= 80%.

**Key test scenarios**:

1. **Config loader tests**:
   - Valid env vars load correctly
   - Missing optional vars use defaults
   - `.env` file fallback works (use `tmp_path` fixture to create temp .env)
   - `env_prefix` overrides work for per-service settings subclass
   - Invalid values raise `ValidationError`

2. **Exception handler tests**:
   - Each `AppException` subclass has correct `status_code`
   - `app_exception_handler` returns `ErrorResponse` JSON with matching fields
   - `unhandled_exception_handler` returns 500 with generic message, no stack trace
   - `correlation_id` included in all error responses
   - `register_exception_handlers` wires both handlers to a FastAPI app

3. **Schema tests**:
   - `BaseSchema` serializes to camelCase JSON and deserializes from both camelCase and snake_case
   - `PaginatedResponse[T]` generic instantiation with a concrete type works
   - `ErrorResponse` round-trip serialization
   - `HealthResponse` with nested `HealthCheckResult` validates correctly

4. **Logging tests**:
   - `setup_logging("test-svc", environment="production")` → JSON output
   - `setup_logging("test-svc", environment="development")` → pretty-print
   - Context vars (`correlation_id`, `tenant_id`) appear in log output
   - `get_logger()` returns a usable structlog logger

5. **Health check tests**:
   - All checks pass → `status: "healthy"`
   - Non-critical check fails → `status: "degraded"`
   - Critical check fails → `status: "unhealthy"`
   - `uptime` is a positive float
   - `create_health_router()` returns an `APIRouter` with `/healthz` route
   - `create_db_check`/`create_redis_check` factories return callables

6. **Middleware tests** (use `httpx.ASGITransport` + `httpx.AsyncClient` for testing middleware):
   - **Audit**: Generates correlation_id if `X-Request-ID` missing; uses provided `X-Request-ID` if present; sets `X-Request-ID` response header; logs method/path/status/duration
   - **Auth**: Excluded paths skip authentication; missing Authorization header → 401; invalid Bearer format → 401; successful token → `request.state.user` populated with `UserContext`; sets tenant_id context var
   - **Rate limit**: `NoOpRateLimiter` always returns `allowed=True`

### Test Expectations (from Epic-Level Test Design)

This story maps to the following scenarios from `test-design-epic-01.md`:

| Priority | Test Description | Risk Link | Verification |
|----------|-----------------|-----------|--------------|
| **P1** | eusolicit-common config loader — env vars + .env fallback + Pydantic validation | E01-R-008 (Score 4) | Valid config loads; missing required var raises; .env fallback works |
| **P1** | eusolicit-common health endpoint — `/healthz` returns status + version + checks | E01-R-008 (Score 4) | Response matches `HealthResponse` schema; db/redis checks present |
| **P1** | eusolicit-common exception handlers — domain exceptions → HTTP error envelope | E01-R-008 (Score 4) | Map 400/401/403/404/409/422/500; verify `{error, message, details, correlation_id}` |
| **P1** | eusolicit-common Pydantic base models — serialization round-trip | — | `BaseSchema` camelCase alias, `PaginatedResponse[T]`, `ErrorResponse` |
| **P2** | eusolicit-common structured logging — JSON output + correlation_id + service_name | — | Production mode: JSON; dev mode: pretty-print; context fields present |
| **P2** | eusolicit-common audit middleware — logs method, path, status, duration, user_id | — | Request/response logged with all required fields |
| **P2** | eusolicit-common rate limit middleware — abstract interface defined | — | Interface exists; NoOpRateLimiter instantiates without error |
| **P2** | eusolicit-common auth middleware — abstract JWT extraction populates request state | — | Abstract class instantiable with mock implementation |

**Risk E01-R-008 (MEDIUM, Score 4):** eusolicit-common middleware contract misalignment — abstract base classes may not match concrete implementation needs in later epics. Mitigated by: unit tests for abstract interface contracts; review against PRD feature requirements; concrete implementations in later epics can extend without breaking the interface.

**Quality gate**: P1 tests 100% pass rate. >= 80% coverage on eusolicit-common package.

### Cross-Story Dependencies

- **Story 1.1** (DONE): Created monorepo structure, `packages/eusolicit-common/` directory, pyproject.toml
- **Story 1.2** (DONE): Created Docker Compose, `.env.example` with service URLs and DB/Redis config, test conftest fixtures
- **Story 1.3** (DONE): Created PostgreSQL schemas and roles — config loader must align with env var naming in `.env.example`
- **Story 1.4** (DONE): Alembic migration scaffold — no direct dependency
- **Story 1.5** (DONE): Created `events/` subpackage in eusolicit-common — do not break imports; events already exported from `__init__.py`
- **Story 1.7** (FUTURE): eusolicit-models — will use `BaseSchema` as parent for all DTOs and event schemas
- **Story 1.8** (FUTURE): CI pipeline — will run tests with coverage check >= 80%

### Previous Story Intelligence (from Story 1.5)

Key learnings from Story 1.5 implementation:

1. **DO NOT TOUCH enforcement**: Story 1.5 was approved with zero violations of DO NOT TOUCH files. Follow the same discipline.

2. **Test structure**: Unit tests go in `tests/unit/` with `@pytest.mark.unit` marker. Integration tests go in `tests/integration/` with `@pytest.mark.integration`. Use `pytest_asyncio.fixture` for async fixtures.

3. **Existing test count**: 1370+ tests in the regression suite from previous stories (plus 22 pre-existing failures from uninstalled eusolicit_kraftdata/eusolicit_models packages). Run `make test` after implementation to verify no regressions.

4. **pyproject.toml config**: `asyncio_mode = "auto"` and `asyncio_default_fixture_loop_scope = "session"` already configured — async tests work out of the box.

5. **Package exports pattern**: Story 1.5 added events exports to `__init__.py`. Follow the same pattern — import from submodules and re-export at package root level.

6. **Code review findings from 1.5**: Handle error cases explicitly (don't swallow errors silently). Non-atomic operations are acceptable per at-least-once semantics but should be documented.

7. **eusolicit-common installed in editable mode**: `pip install -e packages/eusolicit-common` — new modules are immediately importable.

### Anti-Patterns to Avoid

1. **DO NOT modify service `main.py` files** — this story creates reusable components only. Service integration is E02+.
2. **DO NOT create concrete JWT validation** — auth middleware is abstract. Concrete implementation deferred to auth epic (JWT RS256 per architecture ADR 1.1/Section 12).
3. **DO NOT create concrete Redis rate limiter** — the abstract interface + NoOpRateLimiter is sufficient. Redis INCR-based implementation is in the subscription/tier gating epic.
4. **DO NOT add new dependencies to pyproject.toml** — all needed deps (fastapi, structlog, pydantic, pydantic-settings, redis) are already present.
5. **DO NOT break existing events imports** — `from eusolicit_common import EventPublisher, EventConsumer, bootstrap_event_bus` must continue to work.
6. **DO NOT use `logging.getLogger()`** — all logging MUST go through structlog. The architecture mandates this (Section 15).
7. **DO NOT hardcode service names or connection strings** — all config comes from environment variables via the config loader.
8. **DO NOT create integration tests that require Docker Compose** — all tests for this story should be pure unit tests with mocked dependencies.
9. **DO NOT use `BaseHTTPMiddleware` for high-throughput paths in production** — it wraps the entire request in a threadpool for streaming responses. For Story 1.6, `BaseHTTPMiddleware` is acceptable (matches the epic's implementation notes). If performance becomes a concern, middleware can be refactored to pure ASGI middleware later.

### Project Structure Notes

**Target directory tree changes from this story** (new items marked with `+`):

```
eusolicit-app/
├── packages/
│   └── eusolicit-common/
│       ├── pyproject.toml                              # NO CHANGE
│       └── src/
│           └── eusolicit_common/
│               ├── __init__.py                         # MODIFY — add all new exports
│               ├── config.py                           # + NEW — BaseServiceSettings, get_settings()
│               ├── logging.py                          # + NEW — setup_logging(), get_logger(), context vars
│               ├── exceptions.py                       # + NEW — AppException hierarchy, handlers, register_exception_handlers()
│               ├── schemas.py                          # + NEW — BaseSchema, PaginatedResponse[T], ErrorResponse, HealthResponse
│               ├── health.py                           # + NEW — HealthCheck, create_health_router(), check factories
│               ├── events/                             # NO CHANGE (Story 1.5)
│               │   ├── __init__.py
│               │   ├── publisher.py
│               │   ├── consumer.py
│               │   └── bootstrap.py
│               └── middleware/                          # + NEW package
│                   ├── __init__.py                     # + NEW — exports all middleware classes
│                   ├── auth.py                         # + NEW — AbstractAuthMiddleware, UserContext
│                   ├── audit.py                        # + NEW — AuditMiddleware
│                   └── rate_limit.py                   # + NEW — AbstractRateLimitMiddleware, NoOpRateLimiter, RateLimitResult
│
└── tests/
    └── unit/
        ├── test_event_bus_unit.py                      # NO CHANGE (Story 1.5)
        ├── + test_eusolicit_common_config.py           # + NEW
        ├── + test_eusolicit_common_exceptions.py       # + NEW
        ├── + test_eusolicit_common_schemas.py          # + NEW
        ├── + test_eusolicit_common_logging.py          # + NEW
        ├── + test_eusolicit_common_health.py           # + NEW
        └── + test_eusolicit_common_middleware.py        # + NEW
```

### References

- [Source: eusolicit-docs/planning-artifacts/epic-01-infrastructure-foundation.md#S01.06]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section-6-Shared-Libraries]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section-10-Tier-Gating-Usage-Enforcement]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section-12-Authentication-Authorization]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section-15-Observability]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-01.md#P1-config-loader-health-exceptions-models]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-01.md#P2-logging-middleware]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-01.md#E01-R-008]
- [Source: eusolicit-app/packages/eusolicit-common/pyproject.toml#dependencies]
- [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/__init__.py#current-exports]
- [Source: eusolicit-app/.env.example#per-service-env-vars]
- [Source: eusolicit-app/tests/conftest.py#test-fixtures]
- [Source: eusolicit-docs/implementation-artifacts/1-5-redis-streams-event-bus-setup.md#Previous-Story-Intelligence]

## Senior Developer Review

### Review Round 1 (2026-04-06)

**REVIEW: Changes Requested**

**Reviewer:** BMad Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Date:** 2026-04-06
**Test results:** 88/88 PASSED (0.40s) — pytest 9.0.2, Python 3.13.5
**DO NOT TOUCH violations:** 0 (all infrastructure files intact, events imports verified)

5 findings raised (F-1 MEDIUM, F-2 MEDIUM, F-3 LOW, F-4 LOW, F-5 MINOR). All 5 resolved by dev agent.

---

### Review Round 2 (2026-04-06) — Re-Review After Fixes

**REVIEW: Approve**

**Reviewer:** BMad Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Date:** 2026-04-06
**Test results:** 91/91 PASSED (0.39s) — pytest 9.0.2, Python 3.13.5, asyncio auto
**Coverage:** 82.11% on `packages/eusolicit-common/` (≥ 80% threshold met)
**DO NOT TOUCH violations:** 0

---

#### Prior Findings — Resolution Verification

| # | Finding | Status |
|---|---------|--------|
| F-1 | `env_file=".env"` missing from BaseServiceSettings | ✅ Resolved — `config.py:35` now has `env_file=".env"` |
| F-2 | Coverage gate unverifiable | ✅ Resolved — pytest-cov installed, 82.11% verified |
| F-3 | Error envelope key format inconsistency | ✅ Resolved — documented as intentional per AC 7 snake_case |
| F-4 | Non-idiomatic `create_db_check` | ✅ Resolved — uses `async with engine.connect() as conn:` |
| F-5 | Rate limit 429 empty correlation_id | ✅ Resolved — now uses `get_correlation_id() or ""` |

---

#### New Adversarial Findings (Round 2)

**Three-layer review executed: Blind Hunter, Edge Case Hunter, Acceptance Auditor.**
91 tests passed · 82.11% coverage · 6 patch · 5 defer · 6 dismiss

##### F-6 [MEDIUM — Non-blocking] Auth middleware: no catch-all for non-AppException

**File:** `middleware/auth.py:95`
**Source:** Blind Hunter + Edge Case Hunter

The `dispatch()` method catches `AppException` but not generic exceptions. If a concrete `verify_token()` raises `ConnectionError`/`TimeoutError` (likely with real auth providers), it propagates unhandled above FastAPI's exception handler layer, producing a raw 500 without the error envelope.

**Impact:** Deferred — the abstract class has no concrete implementation yet. Services implementing `verify_token()` should wrap provider calls in try/except. When a concrete auth middleware is built (auth epic), the catch-all should be added. The code comment at line 96-98 already acknowledges the ASGI stack issue, which is good awareness.

**Suggested future fix:** Add `except Exception` block after `except AppException` that logs and returns a 500 JSONResponse with error envelope.

##### F-7 [MEDIUM — Non-blocking] Health checks: no timeout on hanging checks

**File:** `health.py:82`
**Source:** Edge Case Hunter

`run_checks()` awaits `check_fn()` without any timeout. A hanging dependency check (DNS resolution stuck, socket with no timeout) blocks `/healthz` indefinitely, causing Kubernetes liveness probes to fail.

**Impact:** Deferred — services don't wire health checks yet (story scope: "creates reusable components only"). When services register checks (E02+), `asyncio.wait_for(check_fn(), timeout=5.0)` should be added.

##### F-8 [LOW — Non-blocking] Auth middleware: empty Bearer token passed to verify_token

**File:** `middleware/auth.py:84-88`
**Source:** Edge Case Hunter

`Authorization: Bearer ` (trailing space, no token) passes `startswith("Bearer ")`, sending an empty string to `verify_token`. Concrete implementations should handle this, but a guard `if not token: raise UnauthorizedError(...)` would be more defensive.

##### F-9 [LOW — Non-blocking] Dead code: unused module-level `_start_time`

**File:** `health.py:29`
**Source:** Blind Hunter

`_start_time: float = time.time()` at module level is never referenced. `HealthCheck` uses `self._start_time` (line 44). Harmless dead code.

##### F-10 [LOW — Non-blocking] Rate limiter: exact path matching only

**File:** `middleware/rate_limit.py:66`
**Source:** Blind Hunter + Edge Case Hunter

`if path in self.rate_limits:` uses exact dict lookup. Trailing-slash variants bypass rate limiting. Deferred by design — concrete Redis implementation (subscription/tier-gating epic) will need pattern matching. The `NoOpRateLimiter` always allows regardless, so no runtime impact now.

---

#### Acceptance Criteria Verification

| AC | Requirement | Status |
|----|-------------|--------|
| 1 | Config loader: env vars with .env fallback, Pydantic BaseSettings, env_prefix | ✅ Pass |
| 2 | Logging: structlog JSON/pretty-print, correlation_id/service_name/tenant_id | ✅ Pass |
| 3 | Auth middleware base: abstract JWT extraction, UserContext, exclude_paths | ✅ Pass |
| 4 | Audit middleware: logs method/path/status/duration/user_id/tenant_id/correlation_id | ✅ Pass |
| 5 | Rate limit middleware base: abstract with pluggable backend, per-route config | ✅ Pass |
| 6 | Health check endpoint: GET /healthz with status/version/uptime/checks | ✅ Pass |
| 7 | Exception handlers: domain exceptions → HTTP responses with error envelope | ✅ Pass |
| 8 | Pydantic base models: BaseSchema, PaginatedResponse[T], ErrorResponse, HealthResponse | ✅ Pass |
| 9 | Unit tests ≥ 80% coverage on eusolicit-common | ✅ Pass (82.11%) |

---

#### Positive Observations

- **Code quality is excellent**: comprehensive NumPy-style docstrings, `__all__` exports, clean type annotations with `from __future__ import annotations`, proper use of `ABC`/`abstractmethod` for extension points, `@dataclass` for lightweight data containers.
- **Architecture alignment is perfect**: abstract bases for auth/rate-limit (concrete impl deferred), factory functions for health checks, singleton config, context vars for request-scoped data — all matching the solution architecture v4 ADRs.
- **DO NOT TOUCH discipline is flawless**: zero modifications to docker-compose, .env.example, Makefile, events/, pyproject.toml, conftest.py, or service main.py files.
- **Events backward compatibility verified**: `from eusolicit_common import EventPublisher, EventConsumer, bootstrap_event_bus` confirmed working.
- **Auth middleware correctly handles ASGI stack concern**: catches `AppException` in dispatch and converts to JSONResponse since middleware sits above FastAPI's `ExceptionMiddleware`.
- **Health check status logic robust**: healthy/degraded/unhealthy states with critical vs non-critical distinction, well-tested.
- **Test quality is high**: 91 tests using `httpx.ASGITransport` for middleware testing, proper mock isolation, parametrized exception tests, context var verification.

---

#### Verdict Summary

| # | Severity | Finding | Blocking? |
|---|----------|---------|-----------|
| F-6 | MEDIUM | Auth middleware: no catch-all for non-AppException | No — deferred to auth epic |
| F-7 | MEDIUM | Health check: no timeout on hanging checks | No — deferred to service integration |
| F-8 | LOW | Auth middleware: empty Bearer token guard | No |
| F-9 | LOW | Dead code: unused module-level `_start_time` | No |
| F-10 | LOW | Rate limiter: exact path matching only | No — deferred by design |

**All 9 ACs satisfied. All 5 prior findings resolved. No blocking issues found in re-review.**
**New findings are non-blocking pattern improvements for future stories.**

---

## Dev Agent Record

### Agent Model Used

Claude Opus 4.6 (claude-sonnet-4-20250514)

### Debug Log References

- Initial implementation completed by previous session (all 10 tasks, 88 tests passing)
- Code review returned 5 findings (F-1 MEDIUM, F-2 MEDIUM, F-3 LOW, F-4 LOW, F-5 MINOR)
- This session addressed all 5 review findings

### Completion Notes List

- ✅ Resolved review finding [MEDIUM] F-1: Added `env_file=".env"` to `BaseServiceSettings.model_config` — base config now reads `.env` fallback as AC 1 requires. Updated config test `test_env_file_fallback` to test `BaseServiceSettings` directly via `monkeypatch.chdir`. Fixed all config tests to use `monkeypatch.chdir(tmp_path)` to isolate from project `.env`.
- ✅ Resolved review finding [MEDIUM] F-2: Installed `pytest-cov`, verified coverage at **82.11%** (>= 80% threshold). Added 3 dispatch tests for rate_limit middleware (unconfigured path, configured path, 429 denied) to boost coverage.
- ✅ Resolved review finding [LOW] F-3: Error envelope keys use snake_case per AC 7 specification (`{error, message, details, correlation_id}`). This is intentional — the `ErrorResponse` model supports both snake_case and camelCase via `populate_by_name=True`.
- ✅ Resolved review finding [LOW] F-4: Refactored `create_db_check()` in `health.py` to use idiomatic `async with engine.connect() as conn:` instead of two-step `await engine.connect()` + `async with connection`.
- ✅ Resolved review finding [MINOR] F-5: Rate limit 429 response now uses `get_correlation_id() or ""` instead of hardcoded empty string, ensuring correlation ID propagation from audit middleware.
- All 91 unit tests pass (88 original + 3 new rate_limit dispatch tests)
- Full regression suite: 851 passed, 0 failures
- Coverage: 82.11% on eusolicit-common package (exceeds 80% threshold)
- DO NOT TOUCH discipline: zero violations — no modifications to docker-compose.yml, .env.example, Makefile, events/, pyproject.toml, tests/conftest.py, or service main.py files
- Events backward compatibility: `from eusolicit_common import EventPublisher, EventConsumer, bootstrap_event_bus` verified working

### File List

#### New Files (Story 1.6)
- `packages/eusolicit-common/src/eusolicit_common/schemas.py` — BaseSchema, PaginatedResponse[T], ErrorResponse, HealthCheckResult, HealthResponse
- `packages/eusolicit-common/src/eusolicit_common/config.py` — BaseServiceSettings, get_settings()
- `packages/eusolicit-common/src/eusolicit_common/logging.py` — setup_logging(), get_logger(), context var helpers
- `packages/eusolicit-common/src/eusolicit_common/exceptions.py` — AppException hierarchy, handlers, register_exception_handlers()
- `packages/eusolicit-common/src/eusolicit_common/health.py` — HealthCheck, create_health_router(), create_db_check(), create_redis_check()
- `packages/eusolicit-common/src/eusolicit_common/middleware/__init__.py` — Middleware package exports
- `packages/eusolicit-common/src/eusolicit_common/middleware/auth.py` — AbstractAuthMiddleware, UserContext
- `packages/eusolicit-common/src/eusolicit_common/middleware/audit.py` — AuditMiddleware
- `packages/eusolicit-common/src/eusolicit_common/middleware/rate_limit.py` — AbstractRateLimitMiddleware, NoOpRateLimiter, RateLimitResult
- `tests/unit/test_eusolicit_common_config.py` — 8 tests for config loader
- `tests/unit/test_eusolicit_common_exceptions.py` — 17 tests for exception handlers
- `tests/unit/test_eusolicit_common_schemas.py` — 16 tests for Pydantic base models
- `tests/unit/test_eusolicit_common_logging.py` — 13 tests for structured logging
- `tests/unit/test_eusolicit_common_health.py` — 14 tests for health check
- `tests/unit/test_eusolicit_common_middleware.py` — 23 tests for middleware (auth, audit, rate limit)

#### Modified Files (Story 1.6)
- `packages/eusolicit-common/src/eusolicit_common/__init__.py` — Added exports for all new modules (config, logging, exceptions, schemas, health, middleware)

### Change Log

- **2026-04-06**: Initial implementation of all 10 tasks — schemas, config, logging, exceptions, health, auth/audit/rate-limit middleware, package exports, unit tests (88 tests passing)
- **2026-04-06**: Addressed code review findings — 5 items resolved (F-1 env_file, F-2 coverage verification, F-3 envelope key docs, F-4 idiomatic db check, F-5 rate limit correlation_id). Added 3 new rate_limit dispatch tests. 91 tests passing, 82.11% coverage.
