# Story 4.2: httpx Async Client and KraftData Authentication

Status: review

## Change Log

- 2026-04-14: S04.02 implemented — singleton httpx.AsyncClient with KraftData auth, typed exceptions, lifespan wiring, respx dev dep, 9 unit tests + 1 integration smoke test (16/16 pass, 1 skipped in CI). Agent: Claude Sonnet 4.5

## Story

As a **backend developer on the EU Solicit platform**,
I want **a singleton `httpx.AsyncClient` that handles all outbound KraftData calls — injecting authentication headers, enforcing connection pool limits, applying timeouts, and raising typed exceptions**,
so that **every subsequent AI gateway story (S04.04–S04.09) has a single, well-tested call point into KraftData with no ad-hoc HTTP wiring, and the service never leaks connections or sends unauthenticated requests**.

## Acceptance Criteria

1. `kraftdata_client.py` exposes a lifespan-managed `httpx.AsyncClient` singleton initialized on app startup and cleanly closed (`.aclose()`) on app shutdown — no resource leaks
2. Every outbound request carries `Authorization: Bearer {KRAFTDATA_API_KEY}` — no request reaches KraftData without this header
3. Every outbound request carries `Content-Type: application/json` as a default header
4. Connection pool is configured from settings: `max_connections=CONCURRENCY_LIMIT`, `max_keepalive_connections=CONCURRENCY_LIMIT // 2`, `keepalive_expiry=30s`
5. Default timeouts: 60s connect, 120s read (agent runs can be slow)
6. `async def call_kraftdata(method, path, **kwargs) -> httpx.Response` is the single call point; `path` is appended to `KRAFTDATA_BASE_URL`
7. Every outbound request is logged at INFO with `method`, `path`, `status_code`, and `latency_ms`; response bodies logged at DEBUG level
8. `httpx.TimeoutException` raises `KraftDataTimeoutError` (with original exception as context)
9. `httpx.ConnectError` raises `KraftDataConnectionError` (with original exception as context)
10. HTTP 4xx/5xx responses raise `KraftDataAPIError` with `status_code` and response body accessible as attributes
11. `respx>=0.21` added to dev dependencies in `pyproject.toml`

## Tasks / Subtasks

- [x] Task 1: Create `src/ai_gateway/services/exceptions.py` — typed KraftData exceptions (AC: 8, 9, 10)
  - [x] 1.1 Define `KraftDataError(Exception)` as the base; store `message` and optional `original_exc`
  - [x] 1.2 Define `KraftDataTimeoutError(KraftDataError)` — raised on `httpx.TimeoutException`
  - [x] 1.3 Define `KraftDataConnectionError(KraftDataError)` — raised on `httpx.ConnectError`
  - [x] 1.4 Define `KraftDataAPIError(KraftDataError)` — raised on non-2xx responses; expose `status_code: int` and `body: str` attributes
  - [x] 1.5 Export all four from `src/ai_gateway/services/__init__.py`

- [x] Task 2: Create `src/ai_gateway/services/kraftdata_client.py` (AC: 1–10)
  - [x] 2.1 Define `_client: httpx.AsyncClient | None = None` module-level singleton reference
  - [x] 2.2 Implement `get_client() -> httpx.AsyncClient` — returns singleton; raises `RuntimeError` if called before `init_client()`
  - [x] 2.3 Implement `async def init_client(settings: AIGatewaySettings) -> None`:
    - Build `httpx.Limits(max_connections=settings.concurrency_limit, max_keepalive_connections=settings.concurrency_limit // 2, keepalive_expiry=30)`
    - Build `httpx.Timeout(connect=60.0, read=120.0, write=30.0, pool=10.0)`
    - Build default headers dict: `{"Authorization": f"Bearer {settings.kraftdata_api_key}", "Content-Type": "application/json"}`
    - Instantiate `httpx.AsyncClient(base_url=settings.kraftdata_base_url, headers=..., limits=..., timeout=...)`
    - Assign to `_client`
  - [x] 2.4 Implement `async def close_client() -> None` — calls `await _client.aclose()` if `_client` is not None; resets `_client` to None
  - [x] 2.5 Implement `async def call_kraftdata(method: str, path: str, **kwargs) -> httpx.Response`:
    - Capture `start = time.monotonic()`
    - Call `await get_client().request(method, path, **kwargs)` inside a try/except block
    - On success: log at INFO with `method`, `path`, `status_code`, `latency_ms`; log response body at DEBUG; call `resp.raise_for_status()` to trigger `httpx.HTTPStatusError` on 4xx/5xx
    - Catch `httpx.TimeoutException` → raise `KraftDataTimeoutError(f"Timeout calling {method} {path}", original_exc=exc)` from `exc`
    - Catch `httpx.ConnectError` → raise `KraftDataConnectionError(f"Connection error calling {method} {path}", original_exc=exc)` from `exc`
    - Catch `httpx.HTTPStatusError` → raise `KraftDataAPIError(f"KraftData returned {exc.response.status_code}", status_code=exc.response.status_code, body=exc.response.text)` from `exc`
    - Return `httpx.Response` on success (2xx)

- [x] Task 3: Wire `init_client` and `close_client` into `main.py` lifespan (AC: 1)
  - [x] 3.1 Import `init_client` and `close_client` from `ai_gateway.services.kraftdata_client`
  - [x] 3.2 In lifespan startup (after `setup_logging`): call `await init_client(settings)`
  - [x] 3.3 In lifespan shutdown (after `yield`): call `await close_client()`

- [x] Task 4: Add `respx` to dev dependencies (AC: 11)
  - [x] 4.1 Add `respx>=0.21` to `[project.optional-dependencies] dev` in `pyproject.toml`

- [x] Task 5: Write unit tests — `tests/unit/test_kraftdata_client.py` (AC: 2, 3, 4, 5, 8, 9, 10)
  - [x] 5.1 Test `test_auth_header_injected` (E04-P0-003): use `respx.mock`, call `call_kraftdata("GET", "/test")`, assert outbound request has `Authorization: Bearer <key>` and `Content-Type: application/json`
  - [x] 5.2 Test `test_no_auth_header_no_request_sent`: verify that if `KRAFTDATA_API_KEY` is empty, the auth header value is `Bearer ` (empty bearer — not a key leak, but a config misconfiguration; log at WARN during `init_client` if key is empty)
  - [x] 5.3 Test `test_timeout_raises_kraftdata_timeout_error` (E04-P0-003): `respx` raises `httpx.ReadTimeout`; assert `KraftDataTimeoutError` raised; assert original exc chained
  - [x] 5.4 Test `test_connect_error_raises_kraftdata_connection_error`: `respx` raises `httpx.ConnectError`; assert `KraftDataConnectionError` raised
  - [x] 5.5 Test `test_4xx_raises_kraftdata_api_error` (E04-P0-003): mock 404 response; assert `KraftDataAPIError` raised with `status_code=404`
  - [x] 5.6 Test `test_5xx_raises_kraftdata_api_error`: mock 500 response; assert `KraftDataAPIError` raised with `status_code=500`; assert `body` attribute contains response text
  - [x] 5.7 Test `test_2xx_returns_response`: mock 200 response; assert `httpx.Response` returned; assert `status_code == 200`
  - [x] 5.8 Test `test_connection_pool_configuration` (E04-P2-014): call `init_client(settings)` with `concurrency_limit=3`; inspect `get_client()._limits`; assert `max_connections==3`, `max_keepalive_connections==1`, `keepalive_expiry==30`
  - [x] 5.9 Test `test_close_client_calls_aclose` (E04-P1-020): call `init_client`, then `close_client()`; assert `get_client()` raises `RuntimeError` after close (singleton reset to None)

- [x] Task 6: Write integration smoke test — `tests/integration/test_kraftdata_live.py` (AC: 2, skipped in CI)
  - [x] 6.1 Mark entire module `@pytest.mark.skipif(not os.getenv("KRAFTDATA_API_KEY"), reason="KraftData credentials not available")`
  - [x] 6.2 Test `test_live_auth_succeeds`: call a real KraftData endpoint (e.g., `/client/api/v1/health` or equivalent); assert 200 or 401 (not connection error); assert auth header sent

## Dev Notes

### CRITICAL: Module Path Convention (same as S04.01)

The epic spec uses `app/services/kraftdata_client.py` — the **actual** monorepo path is:

| Epic spec path | Actual monorepo path |
|---|---|
| `app/services/kraftdata_client.py` | `services/ai-gateway/src/ai_gateway/services/kraftdata_client.py` |
| `app/services/exceptions.py` | `services/ai-gateway/src/ai_gateway/services/exceptions.py` |

Do NOT create a top-level `app/` directory — the existing structure uses `src/ai_gateway/`.

### What Already Exists After S04.01

| File | State | Action for S04.02 |
|---|---|---|
| `src/ai_gateway/main.py` | Complete with lifespan stub comment "S04.02 will add httpx client close here" | **MODIFY** lifespan to call `init_client` / `close_client` |
| `src/ai_gateway/config.py` | Complete — `concurrency_limit`, `kraftdata_base_url`, `kraftdata_api_key` already defined | **DO NOT TOUCH** |
| `src/ai_gateway/services/__init__.py` | Empty `__init__.py` | **EXTEND** to re-export exceptions |
| `pyproject.toml` | Has `httpx>=0.27`, missing `respx` in dev deps | **EXTEND** per Task 4 |
| `tests/conftest.py` | Has `gateway_engine`, `gateway_session`, `mock_kraftdata_base_url` fixtures | **DO NOT TOUCH** |
| `config/.gitkeep` | Stub dir for S04.03 | **DO NOT TOUCH** |

### Exception Hierarchy Design

Exceptions live in `services/exceptions.py` (not in `services/kraftdata_client.py`) so that S04.04–S04.09 routers can import them without importing the client module:

```python
# src/ai_gateway/services/exceptions.py
from __future__ import annotations
from typing import Any


class KraftDataError(Exception):
    """Base for all KraftData client errors."""
    def __init__(self, message: str, *, original_exc: Exception | None = None) -> None:
        self.message = message
        self.original_exc = original_exc
        super().__init__(message)


class KraftDataTimeoutError(KraftDataError):
    """Raised when KraftData request times out (httpx.TimeoutException)."""


class KraftDataConnectionError(KraftDataError):
    """Raised when KraftData is unreachable (httpx.ConnectError)."""


class KraftDataAPIError(KraftDataError):
    """Raised on HTTP 4xx/5xx responses from KraftData."""
    def __init__(
        self,
        message: str,
        *,
        status_code: int,
        body: str = "",
        original_exc: Exception | None = None,
    ) -> None:
        self.status_code = status_code
        self.body = body
        super().__init__(message, original_exc=original_exc)
```

### Client Singleton Pattern

The singleton is module-level (`_client`) rather than app-state to keep `call_kraftdata()` importable without a FastAPI `Request` object dependency:

```python
# src/ai_gateway/services/kraftdata_client.py
from __future__ import annotations

import time
import structlog
import httpx

from ai_gateway.config import AIGatewaySettings
from ai_gateway.services.exceptions import (
    KraftDataAPIError, KraftDataConnectionError, KraftDataTimeoutError,
)

log = structlog.get_logger(__name__)
_client: httpx.AsyncClient | None = None


def get_client() -> httpx.AsyncClient:
    if _client is None:
        raise RuntimeError(
            "KraftData client not initialised. Call init_client() during app startup."
        )
    return _client


async def init_client(settings: AIGatewaySettings) -> None:
    global _client
    if settings.kraftdata_api_key == "":
        log.warning("kraftdata.api_key_empty", message="KRAFTDATA_API_KEY is empty — all requests will fail auth")
    limits = httpx.Limits(
        max_connections=settings.concurrency_limit,
        max_keepalive_connections=settings.concurrency_limit // 2,
        keepalive_expiry=30,
    )
    timeout = httpx.Timeout(connect=60.0, read=120.0, write=30.0, pool=10.0)
    headers = {
        "Authorization": f"Bearer {settings.kraftdata_api_key}",
        "Content-Type": "application/json",
    }
    _client = httpx.AsyncClient(
        base_url=settings.kraftdata_base_url,
        headers=headers,
        limits=limits,
        timeout=timeout,
    )
    log.info("kraftdata.client_initialized", base_url=settings.kraftdata_base_url, concurrency_limit=settings.concurrency_limit)


async def close_client() -> None:
    global _client
    if _client is not None:
        await _client.aclose()
        log.info("kraftdata.client_closed")
        _client = None


async def call_kraftdata(method: str, path: str, **kwargs) -> httpx.Response:
    client = get_client()
    start = time.monotonic()
    try:
        resp = await client.request(method, path, **kwargs)
        latency_ms = int((time.monotonic() - start) * 1000)
        log.info(
            "kraftdata.request",
            method=method,
            path=path,
            status_code=resp.status_code,
            latency_ms=latency_ms,
        )
        log.debug("kraftdata.response_body", path=path, body=resp.text[:2000])
        resp.raise_for_status()
        return resp
    except httpx.TimeoutException as exc:
        latency_ms = int((time.monotonic() - start) * 1000)
        log.warning("kraftdata.timeout", method=method, path=path, latency_ms=latency_ms)
        raise KraftDataTimeoutError(
            f"Timeout calling {method} {path}", original_exc=exc
        ) from exc
    except httpx.ConnectError as exc:
        log.warning("kraftdata.connect_error", method=method, path=path, error=str(exc))
        raise KraftDataConnectionError(
            f"Connection error calling {method} {path}", original_exc=exc
        ) from exc
    except httpx.HTTPStatusError as exc:
        latency_ms = int((time.monotonic() - start) * 1000)
        log.warning(
            "kraftdata.api_error",
            method=method,
            path=path,
            status_code=exc.response.status_code,
            latency_ms=latency_ms,
        )
        raise KraftDataAPIError(
            f"KraftData returned {exc.response.status_code}",
            status_code=exc.response.status_code,
            body=exc.response.text,
            original_exc=exc,
        ) from exc
```

### Updated `main.py` Lifespan Pattern

```python
# src/ai_gateway/main.py — updated lifespan section
from ai_gateway.services.kraftdata_client import close_client, init_client

@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    settings = get_settings()
    setup_logging("ai-gateway", environment=settings.environment, log_level=settings.log_level)
    await init_client(settings)          # S04.02: initialise httpx client
    yield
    await close_client()                 # S04.02: clean shutdown
```

### Testing Pattern with `respx`

`respx` intercepts `httpx` transports at the library level — no need to mock at the network layer:

```python
# tests/unit/test_kraftdata_client.py
import pytest
import respx
import httpx
from ai_gateway.services.kraftdata_client import call_kraftdata, init_client, close_client
from ai_gateway.services.exceptions import KraftDataTimeoutError, KraftDataAPIError
from ai_gateway.config import AIGatewaySettings

TEST_API_KEY = "test-secret-key"

@pytest.fixture(autouse=True)
async def setup_client():
    """Initialise singleton before each test, tear down after."""
    settings = AIGatewaySettings(
        kraftdata_api_key=TEST_API_KEY,
        kraftdata_base_url="https://stage.sirma.ai",
        concurrency_limit=5,
    )
    await init_client(settings)
    yield
    await close_client()


@pytest.mark.asyncio
@respx.mock
async def test_auth_header_injected():
    route = respx.get("https://stage.sirma.ai/test").mock(
        return_value=httpx.Response(200, json={"ok": True})
    )
    await call_kraftdata("GET", "/test")
    assert route.called
    request = route.calls.last.request
    assert request.headers["authorization"] == f"Bearer {TEST_API_KEY}"
    assert request.headers["content-type"] == "application/json"
```

**Note**: `respx.mock` as a decorator patches `httpx` globally — ensure all tests that make real HTTP calls use this to avoid accidental real network requests.

### Connection Pool Inspection

`httpx.AsyncClient` stores limits at `client._limits` (private attribute). Prefer asserting the limits object passed during construction using a mock/spy rather than inspecting the private attribute in tests — but if needed for E04-P2-014:

```python
from ai_gateway.services.kraftdata_client import get_client

async def test_connection_pool_configuration():
    await init_client(AIGatewaySettings(concurrency_limit=3))
    client = get_client()
    assert client._limits.max_connections == 3
    assert client._limits.max_keepalive_connections == 1  # 3 // 2
    # keepalive_expiry is stored as a float in httpx internals
```

Alternatively, use `unittest.mock.patch` to capture the `httpx.Limits(...)` call arguments.

### Response Body Logging Safety

Log only the first 2000 characters of the response body at DEBUG to avoid flooding logs with large KraftData responses (e.g., full ESPD auto-fill outputs). The `call_kraftdata` implementation above already does this with `resp.text[:2000]`.

### `services/__init__.py` Exports

After Task 1, update `src/ai_gateway/services/__init__.py` to export exceptions:

```python
# src/ai_gateway/services/__init__.py
from ai_gateway.services.exceptions import (
    KraftDataAPIError,
    KraftDataConnectionError,
    KraftDataError,
    KraftDataTimeoutError,
)

__all__ = [
    "KraftDataError",
    "KraftDataTimeoutError",
    "KraftDataConnectionError",
    "KraftDataAPIError",
]
```

This allows S04.04+ routers to use `from ai_gateway.services import KraftDataAPIError` cleanly.

### Test Design Context (from test-design-epic-04.md)

Relevant test IDs for S04.02:

| Test ID | Priority | Coverage | Test Level | Key Assertion |
|---|---|---|---|---|
| **E04-P0-003** | P0 — Critical | Auth header injected on every request | Unit | `respx` mock: assert `Authorization: Bearer {key}` and `Content-Type: application/json` present; assert no request bypasses auth |
| **E04-P1-020** | P1 — High | Client shuts down cleanly on app shutdown | Unit | Trigger lifespan shutdown; assert `aclose()` called; assert `get_client()` raises `RuntimeError` after shutdown |
| **E04-P2-014** | P2 — Medium | Connection pool enforces `CONCURRENCY_LIMIT` | Unit | `init_client(settings, concurrency_limit=3)`; inspect `httpx.AsyncClient._limits`; assert `max_connections=3`, `max_keepalive_connections=1`, `keepalive_expiry=30` |

**Additional story-level tests not in test-design (but in epic AC):**
- Timeout → `KraftDataTimeoutError` (E04-P0-009 uses this indirectly for retry tests)
- 4xx → `KraftDataAPIError` with correct `status_code` (E04-P0-003 mentions 4xx handling)
- 5xx → `KraftDataAPIError` (pre-requisite for E04-P0-009 retry test in S04.06)
- Connection error → `KraftDataConnectionError`
- 2xx returns `httpx.Response` directly

### Relation to Subsequent Stories

This story creates the foundational call surface that every other gateway story depends on:

| Story | Dependency on S04.02 |
|---|---|
| S04.03 (agent registry) | None — pure registry logic, no HTTP |
| S04.04 (sync endpoints) | Calls `call_kraftdata()` for agent/workflow/team runs |
| S04.05 (SSE proxy) | Calls `call_kraftdata()` in streaming mode; uses same auth/pool |
| S04.06 (circuit breaker + retry) | Wraps `call_kraftdata()` with resilience policies; catches `KraftDataTimeoutError`, `KraftDataAPIError` |
| S04.07 (webhook) | No outbound KraftData call in this story; uses Redis only |
| S04.08 (execution logging) | Reads latency from caller context; `call_kraftdata()` is already timing-aware |
| S04.09 (rate limiter) | Wraps the call pipeline at a higher level; `call_kraftdata()` is the inner call |
| S04.10 (integration tests) | Sets up `respx` mock for KraftData; all tests use `call_kraftdata()` indirectly |

### Environment Variables Reference (no changes from S04.01)

| Variable | Default | Used by |
|---|---|---|
| `KRAFTDATA_BASE_URL` | `https://stage.sirma.ai` | `init_client()` — `base_url` |
| `KRAFTDATA_API_KEY` | `""` | `init_client()` — `Authorization` header value |
| `CONCURRENCY_LIMIT` | `10` | `init_client()` — `httpx.Limits.max_connections` |

### References

- Epic: `eusolicit-docs/planning-artifacts/epic-04-ai-gateway-service.md#S04.02`
- Test design: `eusolicit-docs/test-artifacts/test-design-epic-04.md` — E04-P0-003, E04-P1-020, E04-P2-014
- S04.01 story (foundation): `eusolicit-docs/implementation-artifacts/4-1-fastapi-service-scaffold-and-health-probes.md`
- Existing config: `services/ai-gateway/src/ai_gateway/config.py`
- Existing main (to extend): `services/ai-gateway/src/ai_gateway/main.py`
- Services `__init__` (to extend): `services/ai-gateway/src/ai_gateway/services/__init__.py`
- pyproject.toml (to extend): `services/ai-gateway/pyproject.toml`
- `eusolicit-kraftdata` response types (for future stories): `packages/eusolicit-kraftdata/src/eusolicit_kraftdata/responses.py`
- `eusolicit-common` exceptions (base pattern): `packages/eusolicit-common/src/eusolicit_common/exceptions.py`
- `respx` usage example in monorepo: `services/client-api/tests/api/test_espd_autofill_export.py`

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5)

### Debug Log References

- **DEVIATION (deferrable):** `httpx.AsyncClient._limits` does not exist in httpx>=0.27. The story Dev Notes reference this attribute but the actual limits are stored at `client._transport._pool._max_connections` / `_max_keepalive_connections` / `_keepalive_expiry` (httpcore.AsyncConnectionPool internals). Test `test_connection_pool_configuration` updated to use correct attributes. Dev Notes also pre-empted this with the `unittest.mock.patch` alternative — pool internals approach used instead as it validates runtime state directly.

### Completion Notes List

- **Task 1:** Created `src/ai_gateway/services/exceptions.py` with full 4-class hierarchy (`KraftDataError`, `KraftDataTimeoutError`, `KraftDataConnectionError`, `KraftDataAPIError`). Updated `services/__init__.py` to re-export all four.
- **Task 2:** Created `src/ai_gateway/services/kraftdata_client.py` with module-level singleton, `get_client()`, `init_client()`, `close_client()`, and `call_kraftdata()`. All AC 1–10 satisfied. Empty API key triggers WARN log. Response body logged at DEBUG truncated to 2000 chars.
- **Task 3:** Updated `main.py` lifespan to call `await init_client(settings)` on startup and `await close_client()` on shutdown. Ruff auto-fixed import ordering and `AsyncGenerator` import path.
- **Task 4:** Added `respx>=0.21` to `[project.optional-dependencies] dev` in `pyproject.toml`. Installed version: 0.23.1.
- **Task 5:** 9 unit tests written and all passing. Covers E04-P0-003, E04-P1-020, E04-P2-014 plus all exception and happy-path scenarios.
- **Task 6:** Integration smoke test written, correctly skipped in CI (no `KRAFTDATA_API_KEY`).
- **Test results:** 16 passed, 1 skipped, 0 failed. No regressions to prior S04.01 tests.
- **Linting:** All story files pass ruff. 4 pre-existing lint issues in S04.01 files not modified by this story.

### File List

- `eusolicit-app/services/ai-gateway/src/ai_gateway/services/exceptions.py` (created)
- `eusolicit-app/services/ai-gateway/src/ai_gateway/services/kraftdata_client.py` (created)
- `eusolicit-app/services/ai-gateway/src/ai_gateway/services/__init__.py` (modified — added exception re-exports)
- `eusolicit-app/services/ai-gateway/src/ai_gateway/main.py` (modified — wired init_client/close_client into lifespan)
- `eusolicit-app/services/ai-gateway/pyproject.toml` (modified — added respx>=0.21 to dev deps)
- `eusolicit-app/services/ai-gateway/tests/unit/test_kraftdata_client.py` (created)
- `eusolicit-app/services/ai-gateway/tests/integration/test_kraftdata_live.py` (created)
