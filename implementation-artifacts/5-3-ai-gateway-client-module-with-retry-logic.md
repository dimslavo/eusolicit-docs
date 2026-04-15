# Story 5.3: AI Gateway Client Module with Retry Logic

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer implementing the data pipeline's crawler and enrichment tasks**,
I want **a shared `ai_gateway_client` module that wraps all HTTP calls from the data pipeline to the AI Gateway service with structured retry (exponential back-off, max 3 attempts, jitter), per-agent-type timeout configuration, request/response logging at DEBUG level, correlation ID injection, a circuit-breaker pattern (open after 5 consecutive failures, half-open after 60 s), and typed Pydantic response models**,
so that **every downstream story (S05.04–S05.08) can call KraftData agents through a single hardened client without duplicating retry/circuit-breaker logic, and Celery can handle task-level retries separately by catching `AIGatewayUnavailableError`**.

## Acceptance Criteria

1. Unit tests verify retry counts (max 3 retries = 4 total attempts), exponential back-off timing (1 s → 2 s → 4 s base delays with ±25 % jitter), and circuit-breaker state transitions (`CLOSED → OPEN → HALF_OPEN → CLOSED`)
2. Timeout and retry parameters configurable via environment variables: `AI_GATEWAY_TIMEOUT_CRAWLER` (default 120 s), `AI_GATEWAY_TIMEOUT_SCORING` (default 60 s), `AI_GATEWAY_MAX_RETRIES` (default 3), `AI_GATEWAY_CIRCUIT_THRESHOLD` (default 5), `AI_GATEWAY_CIRCUIT_COOLDOWN` (default 60 s)
3. All outbound calls include a `X-Correlation-ID` header; the value is generated per-call as `uuid4` if not already present in the Celery task context
4. `AIGatewayUnavailableError` is raised when all retries are exhausted; the exception is importable from `data_pipeline.ai_gateway_client` and registered in the crawl task stubs' `autoretry_for` tuple (replacing the placeholder in S05.02 task stubs)
5. Client returns typed Pydantic response models; caller receives the parsed `output` dict from the AI Gateway's JSON response

## Tasks / Subtasks

- [x] Task 1: Create `ai_gateway_client` package (AC: 3, 4, 5)
  - [x] 1.1 Create `services/data-pipeline/src/data_pipeline/ai_gateway_client/__init__.py` — re-export `AIGatewayClient`, `AIGatewayUnavailableError`, `AgentType` so callers only need one import
  - [x] 1.2 Create `services/data-pipeline/src/data_pipeline/ai_gateway_client/exceptions.py` — define `AIGatewayUnavailableError(Exception)` with attributes: `agent_name: str`, `attempts: int`, `last_status_code: int | None`
  - [x] 1.3 Create `services/data-pipeline/src/data_pipeline/ai_gateway_client/models.py` — define `AgentType(str, Enum)` with values `CRAWLER`, `NORMALIZATION`, `SCORING`, `SUBMISSION_GUIDE`; define `AgentCallResponse(BaseModel)` with fields `output: dict[str, Any]`, `execution_id: str`, `agent_type: AgentType`

- [x] Task 2: Implement circuit breaker (AC: 1, 2)
  - [x] 2.1 Create `services/data-pipeline/src/data_pipeline/ai_gateway_client/circuit_breaker.py` — in-memory `CircuitBreaker` class mirroring the pattern in `ai_gateway/services/circuit_breaker.py` but standalone (no ai_gateway imports); states: `CLOSED`, `OPEN`, `HALF_OPEN`; configurable `threshold` and `cooldown`; use `threading.Lock` (not asyncio — Celery tasks run in threads); open after `threshold` consecutive failures; half-open after `cooldown` seconds; on OPEN, raise `CircuitOpenError(agent_name)` without making HTTP call; module-level `_breakers: dict[str, CircuitBreaker]` registry with `get_breaker(agent_name)` and `reset_all_breakers()` (test teardown only)

- [x] Task 3: Implement retry logic (AC: 1, 2)
  - [x] 3.1 Create `services/data-pipeline/src/data_pipeline/ai_gateway_client/retry.py` — sync `with_retry(fn, *, agent_name, max_retries, base_delay, jitter_factor)` function; retries only on HTTP 5xx and connection errors; raises `AIGatewayUnavailableError` on exhaustion; calculates delay as `base_delay * (2 ** attempt)` + jitter `random.uniform(-delay * jitter_factor, delay * jitter_factor)`; sleeps between attempts using `time.sleep`; logs each retry attempt at WARNING with structlog `agent_name`, `attempt`, `delay_s`, `status_code`

- [x] Task 4: Implement `AIGatewayClient` (AC: 2, 3, 4, 5)
  - [x] 4.1 Create `services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py`:
    - Class `AIGatewayClient` with `__init__(self, *, base_url: str, api_key: str | None = None)` — reads `AI_GATEWAY_BASE_URL` from env if not provided
    - Class-level `_timeout_map: dict[AgentType, float]` populated from env vars at module import: `AI_GATEWAY_TIMEOUT_CRAWLER` → `AgentType.CRAWLER`; `AI_GATEWAY_TIMEOUT_SCORING` → `AgentType.SCORING`; default 60 s for all types
    - Method `call_agent(self, agent_name: str, payload: dict, *, agent_type: AgentType, correlation_id: str | None = None) -> AgentCallResponse`:
      - Generate `correlation_id = str(uuid4())` if not provided
      - Resolve timeout from `_timeout_map[agent_type]`
      - Call through `get_breaker(agent_name).call(...)` wrapping `with_retry(...)` wrapping the raw `httpx.post(...)` call
      - Raw call: `POST {base_url}/workflows/{agent_name}/run` with header `X-Correlation-ID: {correlation_id}`, JSON body `{"input": payload}`, timeout from map
      - On success: parse response JSON → `AgentCallResponse`; log at DEBUG `agent_name`, `correlation_id`, `status_code`, truncated response body (≤2000 chars)
      - On `CircuitOpenError`: re-raise as `AIGatewayUnavailableError(agent_name, attempts=0, last_status_code=None)` with `circuit_open=True` attribute
      - On exhausted retries: `AIGatewayUnavailableError` already raised by `with_retry`
    - Module-level singleton: `_default_client: AIGatewayClient | None = None`; `get_client() -> AIGatewayClient` lazy initialiser reading `AI_GATEWAY_BASE_URL` from env; `reset_client()` for tests

- [x] Task 5: Update task stubs to reference `AIGatewayUnavailableError` (AC: 4)
  - [x] 5.1 Update `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` — add `autoretry_for=(AIGatewayUnavailableError,)` to `@shared_task` decorator; add import `from data_pipeline.ai_gateway_client import AIGatewayUnavailableError`
  - [x] 5.2 Update `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` — same as 5.1
  - [x] 5.3 Update `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` — same as 5.1
  - [x] 5.4 Update `services/data-pipeline/src/data_pipeline/workers/tasks/cleanup.py` — same as 5.1

- [x] Task 6: Add dev dependencies (AC: 1)
  - [x] 6.1 Add `respx>=0.21` and `freezegun>=1.5` to `[project.optional-dependencies] dev` in `services/data-pipeline/pyproject.toml`

- [x] Task 7: Write unit tests (AC: 1, 2, 3, 4)
  - [x] 7.1 Create `services/data-pipeline/tests/unit/test_ai_gateway_client.py`:
    - **E05-P1-008**: `test_retry_on_5xx` — configure `with_retry` with `max_retries=3`; using `respx` mock 2 responses returning 503 then 200; assert 3 total calls made; assert `freezegun`-controlled delays match expected backoff schedule (1 s × factor, 2 s × factor)
    - **E05-P1-009**: `test_circuit_breaker_state_transitions` — fire 5 consecutive failures on a `CircuitBreaker`; assert `state == OPEN`; assert 6th call raises `CircuitOpenError` immediately without HTTP call; mock `time.monotonic` advancing 60 s; assert state transitions to `HALF_OPEN`; fire successful call; assert state returns to `CLOSED`
    - **E05-P1-010**: `test_unavailable_error_after_retries_exhausted` — mock 4 consecutive 5xx responses; assert `AIGatewayUnavailableError` raised; assert exception is an instance type that Celery's `autoretry_for` tuple will catch
    - **E05-P1-011**: `test_correlation_id_in_all_requests` — mock AI Gateway with `respx`; call `AIGatewayClient.call_agent(...)` without explicit correlation_id; capture the outbound request; assert `X-Correlation-ID` header present and is a valid UUID v4 string
    - **E05-P2-004**: `test_timeout_configurable_per_agent_type` — patch env vars `AI_GATEWAY_TIMEOUT_CRAWLER=90`, `AI_GATEWAY_TIMEOUT_SCORING=30`; reimport client module; assert `_timeout_map[AgentType.CRAWLER] == 90` and `_timeout_map[AgentType.SCORING] == 30`; verify `respx` mock receives request with matching timeout value

## Dev Notes

### Architecture Context

S05.03 creates the **pipeline-side client** that calls the AI Gateway HTTP API (E04). It is NOT a client that calls KraftData directly — the AI Gateway (port 8004) is the single entrypoint for all KraftData agent invocations.

**Call chain:**

```
Celery Task (data-pipeline)
    └── AIGatewayClient.call_agent(agent_name, payload, agent_type=AgentType.CRAWLER)
            └── CircuitBreaker.call(...)          ← pipeline-side guard
                    └── with_retry(...)           ← pipeline-side retry
                            └── httpx.post("http://ai-gateway:8004/workflows/{agent_name}/run")
                                    └── AI Gateway (E04)
                                            ├── its own retry (retry.py)
                                            ├── its own circuit breaker (circuit_breaker.py)
                                            └── KraftData API
```

The pipeline client adds **a second layer of resilience**: if the AI Gateway itself is down (not just KraftData), the pipeline handles it gracefully. The E04 circuit breaker guards KraftData; the pipeline circuit breaker guards the AI Gateway.

[Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#7.3-Data-Pipeline-Service]

### CRITICAL: Sync vs Async

Celery tasks run in **synchronous** threads, not async coroutines. The `AIGatewayClient` must use **synchronous `httpx`** (`httpx.post`, not `httpx.AsyncClient`). Use `threading.Lock` for circuit breaker state, not `asyncio.Lock`. Use `time.sleep` for retry delays, not `await asyncio.sleep`.

This is the key difference from the E04 implementation (which is async FastAPI). Do NOT port the async patterns from `ai_gateway/services/circuit_breaker.py` directly — they will deadlock or behave unpredictably in Celery workers.

### AI Gateway Endpoint to Call

All pipeline → AI Gateway calls go through the **workflow endpoint**:

```
POST http://ai-gateway:8004/workflows/{agent_name}/run
Content-Type: application/json
Authorization: Bearer {AI_GATEWAY_API_KEY}      # optional internal auth
X-Correlation-ID: {uuid4}

{
    "input": { ...payload... }
}
```

Response:
```json
{
    "execution_id": "...",
    "status": "completed",
    "output": { ...agent-specific data... },
    "error": null,
    "execution_time_ms": 1234.5
}
```

The `output` dict is the raw agent response. Each crawler task (S05.04–S05.06) is responsible for interpreting `output` per the agent's schema.

[Source: eusolicit-app/services/ai-gateway/src/ai_gateway/routers/execution.py — POST /workflows/{id}/run]

### Agent Names (Logical Names)

These are the logical names registered in the AI Gateway's agent registry (`config/agents.yaml`). The pipeline client passes them as-is:

| AgentType | Logical Name | Used by Story |
|---|---|---|
| `CRAWLER` | `aop-crawler-agent` | S05.04 |
| `CRAWLER` | `ted-crawler-agent` | S05.05 |
| `CRAWLER` | `eu-grants-crawler-agent` | S05.06 |
| `NORMALIZATION` | `data-normalization-team` | S05.04–S05.06 |
| `SCORING` | `relevance-scoring-agent` | S05.07 |
| `SUBMISSION_GUIDE` | `submission-guide-agent` | S05.08 |

The `AgentType` enum drives timeout selection — `CRAWLER` agents get longer timeouts because crawling can be slow; `SCORING` agents get shorter timeouts.

### `client.py` — Implementation Pattern

```python
"""AI Gateway client for the Data Pipeline service.

Wraps all pipeline → AI Gateway HTTP calls with:
- Retry (exponential back-off, max 3 attempts, jitter)
- Circuit breaker (open after 5 failures, half-open after 60 s)
- Correlation ID injection
- Per-agent-type timeout configuration
- Structured DEBUG logging

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.03]
"""
from __future__ import annotations

import os
import uuid
from typing import Any

import httpx
import structlog

from data_pipeline.ai_gateway_client.circuit_breaker import CircuitOpenError, get_breaker
from data_pipeline.ai_gateway_client.exceptions import AIGatewayUnavailableError
from data_pipeline.ai_gateway_client.models import AgentCallResponse, AgentType
from data_pipeline.ai_gateway_client.retry import with_retry

log = structlog.get_logger(__name__)

_AI_GATEWAY_BASE_URL = os.environ.get("AI_GATEWAY_BASE_URL", "http://ai-gateway:8004")
_AI_GATEWAY_API_KEY = os.environ.get("AI_GATEWAY_API_KEY", "")
_MAX_RETRIES = int(os.environ.get("AI_GATEWAY_MAX_RETRIES", "3"))

# Timeout map: read once at module import; override via env vars
_TIMEOUT_MAP: dict[AgentType, float] = {
    AgentType.CRAWLER: float(os.environ.get("AI_GATEWAY_TIMEOUT_CRAWLER", "120")),
    AgentType.NORMALIZATION: float(os.environ.get("AI_GATEWAY_TIMEOUT_NORMALIZATION", "60")),
    AgentType.SCORING: float(os.environ.get("AI_GATEWAY_TIMEOUT_SCORING", "60")),
    AgentType.SUBMISSION_GUIDE: float(os.environ.get("AI_GATEWAY_TIMEOUT_SUBMISSION_GUIDE", "90")),
}


class AIGatewayClient:
    """Synchronous HTTP client for calling KraftData agents via the AI Gateway."""

    def __init__(self, *, base_url: str = _AI_GATEWAY_BASE_URL, api_key: str = _AI_GATEWAY_API_KEY) -> None:
        self.base_url = base_url.rstrip("/")
        self.api_key = api_key

    def call_agent(
        self,
        agent_name: str,
        payload: dict[str, Any],
        *,
        agent_type: AgentType,
        correlation_id: str | None = None,
    ) -> AgentCallResponse:
        """Call a KraftData agent through the AI Gateway with retry and circuit breaker."""
        if correlation_id is None:
            correlation_id = str(uuid.uuid4())

        timeout = _TIMEOUT_MAP.get(agent_type, 60.0)
        url = f"{self.base_url}/workflows/{agent_name}/run"

        headers = {
            "Content-Type": "application/json",
            "X-Correlation-ID": correlation_id,
        }
        if self.api_key:
            headers["Authorization"] = f"Bearer {self.api_key}"

        breaker = get_breaker(agent_name)

        def _make_request() -> AgentCallResponse:
            with httpx.Client(timeout=timeout) as client:
                response = client.post(url, json={"input": payload}, headers=headers)
                if response.status_code >= 500:
                    raise _GatewayHTTPError(response.status_code, response.text[:2000])
                response.raise_for_status()
                data = response.json()
                log.debug(
                    "ai_gateway.response",
                    agent_name=agent_name,
                    correlation_id=correlation_id,
                    status_code=response.status_code,
                    body_preview=str(data)[:2000],
                )
                return AgentCallResponse(
                    output=data.get("output", {}),
                    execution_id=data.get("execution_id", ""),
                    agent_type=agent_type,
                )

        try:
            return breaker.call(
                lambda: with_retry(
                    _make_request,
                    agent_name=agent_name,
                    max_retries=_MAX_RETRIES,
                )
            )
        except CircuitOpenError:
            raise AIGatewayUnavailableError(
                agent_name=agent_name,
                attempts=0,
                last_status_code=None,
                circuit_open=True,
            )
```

### `circuit_breaker.py` — Sync Implementation

**CRITICAL**: Celery tasks run in threads — use `threading.Lock`, NOT `asyncio.Lock`.

```python
from __future__ import annotations

import threading
import time
from enum import Enum

import structlog

log = structlog.get_logger(__name__)


class CircuitState(str, Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"


class CircuitOpenError(Exception):
    def __init__(self, agent_name: str) -> None:
        self.agent_name = agent_name
        super().__init__(f"Circuit for '{agent_name}' is OPEN")


class CircuitBreaker:
    def __init__(self, agent_name: str, *, threshold: int = 5, cooldown: float = 60.0) -> None:
        self.agent_name = agent_name
        self.threshold = threshold
        self.cooldown = cooldown
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time: float | None = None
        self._lock = threading.Lock()

    def call(self, fn):
        with self._lock:
            # Check if OPEN → HALF_OPEN transition is due
            if self.state == CircuitState.OPEN and self.last_failure_time is not None:
                if (time.monotonic() - self.last_failure_time) >= self.cooldown:
                    self.state = CircuitState.HALF_OPEN
                    log.info("circuit.half_open", agent_name=self.agent_name)
            if self.state == CircuitState.OPEN:
                raise CircuitOpenError(self.agent_name)

        try:
            result = fn()
        except Exception:
            with self._lock:
                self.failure_count += 1
                if self.state == CircuitState.HALF_OPEN or self.failure_count >= self.threshold:
                    self.state = CircuitState.OPEN
                    self.last_failure_time = time.monotonic()
                    log.warning("circuit.opened", agent_name=self.agent_name, failure_count=self.failure_count)
            raise
        else:
            with self._lock:
                if self.state in (CircuitState.HALF_OPEN, CircuitState.CLOSED):
                    self.failure_count = 0
                    self.state = CircuitState.CLOSED
            return result


_breakers: dict[str, CircuitBreaker] = {}
_registry_lock = threading.Lock()


def get_breaker(agent_name: str, *, threshold: int | None = None, cooldown: float | None = None) -> CircuitBreaker:
    import os
    _threshold = threshold or int(os.environ.get("AI_GATEWAY_CIRCUIT_THRESHOLD", "5"))
    _cooldown = cooldown or float(os.environ.get("AI_GATEWAY_CIRCUIT_COOLDOWN", "60"))
    with _registry_lock:
        if agent_name not in _breakers:
            _breakers[agent_name] = CircuitBreaker(agent_name, threshold=_threshold, cooldown=_cooldown)
        return _breakers[agent_name]


def reset_all_breakers() -> None:
    """Clear all circuit state — test teardown only."""
    with _registry_lock:
        _breakers.clear()
```

### `retry.py` — Sync Retry

```python
from __future__ import annotations

import random
import time
from typing import Callable, TypeVar

import structlog

from data_pipeline.ai_gateway_client.exceptions import AIGatewayUnavailableError

log = structlog.get_logger(__name__)
T = TypeVar("T")

_BASE_DELAY: float = 1.0
_FACTOR: float = 2.0
_JITTER: float = 0.25


class _GatewayHTTPError(Exception):
    def __init__(self, status_code: int, body: str) -> None:
        self.status_code = status_code
        self.body = body
        super().__init__(f"AI Gateway returned HTTP {status_code}")


def with_retry(
    fn: Callable[[], T],
    *,
    agent_name: str = "unknown",
    max_retries: int = 3,
) -> T:
    last_exc: Exception | None = None
    last_status: int | None = None

    for attempt in range(max_retries + 1):
        try:
            return fn()
        except _GatewayHTTPError as exc:
            last_exc = exc
            last_status = exc.status_code
        except (ConnectionError, TimeoutError) as exc:
            last_exc = exc

        if attempt >= max_retries:
            log.warning("retry.exhausted", agent_name=agent_name, attempts=attempt + 1)
            break

        delay = _BASE_DELAY * (_FACTOR ** attempt)
        jitter_range = delay * _JITTER
        actual_delay = delay + random.uniform(-jitter_range, jitter_range)
        log.warning("retry.attempt", agent_name=agent_name, attempt=attempt + 1, delay_s=round(actual_delay, 3))
        time.sleep(actual_delay)

    raise AIGatewayUnavailableError(
        agent_name=agent_name,
        attempts=max_retries + 1,
        last_status_code=last_status,
    )
```

**NOTE**: `_GatewayHTTPError` is a **package-private** exception used only within `ai_gateway_client` to signal retryable HTTP errors to `with_retry`. It is NOT `AIGatewayUnavailableError`. The separation is important: `_GatewayHTTPError` is raised inside `_make_request()` and caught by `with_retry`; `AIGatewayUnavailableError` is what callers (Celery tasks) see.

### `exceptions.py`

```python
from __future__ import annotations


class AIGatewayUnavailableError(Exception):
    """Raised when all retries to the AI Gateway are exhausted.

    Celery tasks register this in ``autoretry_for=(AIGatewayUnavailableError,)``
    so Celery handles task-level retry separately from the HTTP-level retry
    already attempted inside the client.

    [Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.03]
    """

    def __init__(
        self,
        agent_name: str,
        *,
        attempts: int = 0,
        last_status_code: int | None = None,
        circuit_open: bool = False,
    ) -> None:
        self.agent_name = agent_name
        self.attempts = attempts
        self.last_status_code = last_status_code
        self.circuit_open = circuit_open
        reason = "circuit OPEN" if circuit_open else f"{attempts} attempts exhausted"
        super().__init__(f"AI Gateway unavailable for agent '{agent_name}' ({reason})")
```

### `models.py`

```python
from __future__ import annotations

from enum import Enum
from typing import Any

from pydantic import BaseModel


class AgentType(str, Enum):
    """Agent types — drives timeout selection in AIGatewayClient."""
    CRAWLER = "crawler"
    NORMALIZATION = "normalization"
    SCORING = "scoring"
    SUBMISSION_GUIDE = "submission_guide"


class AgentCallResponse(BaseModel):
    """Parsed response from a successful AI Gateway agent call."""
    output: dict[str, Any]
    execution_id: str
    agent_type: AgentType
```

### Task Stub Updates (Task 5)

Update the existing stub tasks in `workers/tasks/` to wire in `AIGatewayUnavailableError`:

```python
# In crawl_aop.py, crawl_ted.py, crawl_eu_grants.py, cleanup.py
from data_pipeline.ai_gateway_client import AIGatewayUnavailableError

@shared_task(
    name="pipeline.crawl_aop",
    bind=True,
    max_retries=3,
    default_retry_delay=60,
    autoretry_for=(AIGatewayUnavailableError,),   # ADD THIS LINE
    retry_backoff=True,
)
def crawl_aop(self) -> None:
    ...
```

This single change lets S05.04–S05.06 add the real implementation without touching the decorator again.

### Test Expectations from Epic-Level Test Design

| Priority | Test ID | Description | Implementation |
|---|---|---|---|
| **P1** | E05-P1-008 | Retry on 5xx — 2 failures then success, verify 3 total calls, check backoff delays | `respx` mock + `freezegun.freeze_time` or mock `time.sleep`; assert call count and delay values |
| **P1** | E05-P1-009 | Circuit breaker transitions: `CLOSED → OPEN → HALF_OPEN → CLOSED` | Mock `time.monotonic` advancing 60 s; verify `CircuitOpenError` raised on 6th call |
| **P1** | E05-P1-010 | `AIGatewayUnavailableError` raised after 3 retries; Celery sees it as retriable | Assert type; assert `isinstance(exc, AIGatewayUnavailableError)` |
| **P1** | E05-P1-011 | Correlation ID header present in all outbound calls | `respx` captures request; assert `X-Correlation-ID` in request headers; assert valid UUID4 format |
| **P2** | E05-P2-004 | Timeout configurable per agent type via env vars | `monkeypatch.setenv` + `importlib.reload(client_module)`; assert `_TIMEOUT_MAP[AgentType.CRAWLER]` |

**Pattern for E05-P1-008** (respx + mock sleep):
```python
import respx
import httpx
from unittest.mock import patch

def test_retry_on_5xx(monkeypatch):
    call_count = 0
    sleep_delays = []

    def fake_sleep(delay):
        sleep_delays.append(delay)

    monkeypatch.setattr("data_pipeline.ai_gateway_client.retry.time.sleep", fake_sleep)

    with respx.mock:
        # First 2 calls return 503, 3rd returns 200
        respx.post("http://ai-gateway:8004/workflows/aop-crawler-agent/run").mock(
            side_effect=[
                httpx.Response(503, json={"error": "unavailable"}),
                httpx.Response(503, json={"error": "unavailable"}),
                httpx.Response(200, json={"execution_id": "abc", "status": "completed", "output": {}}),
            ]
        )
        from data_pipeline.ai_gateway_client.client import AIGatewayClient
        from data_pipeline.ai_gateway_client.models import AgentType
        client = AIGatewayClient(base_url="http://ai-gateway:8004")
        result = client.call_agent("aop-crawler-agent", {}, agent_type=AgentType.CRAWLER)
        assert result.execution_id == "abc"
        assert len(respx.calls) == 3   # 1 initial + 2 retries
        assert len(sleep_delays) == 2  # 2 sleeps between 3 attempts
        # Verify exponential backoff: delays should be ~1s and ~2s (±jitter)
        assert 0.75 <= sleep_delays[0] <= 1.25
        assert 1.5 <= sleep_delays[1] <= 2.5
```

**Pattern for E05-P1-009** (circuit breaker transitions):
```python
import time
from unittest.mock import patch, MagicMock

def test_circuit_breaker_transitions():
    from data_pipeline.ai_gateway_client.circuit_breaker import CircuitBreaker, CircuitOpenError, CircuitState

    breaker = CircuitBreaker("test-agent", threshold=5, cooldown=60.0)

    # Fire 5 failures to open the circuit
    for _ in range(5):
        with pytest.raises(Exception):
            breaker.call(lambda: (_ for _ in ()).throw(ConnectionError("down")))

    assert breaker.state == CircuitState.OPEN

    # 6th call rejected immediately without calling fn
    called = []
    with pytest.raises(CircuitOpenError):
        breaker.call(lambda: called.append(1))
    assert len(called) == 0  # fn never called

    # Advance time past cooldown
    monotonic_start = breaker.last_failure_time
    with patch("time.monotonic", return_value=monotonic_start + 61.0):
        # Next call should attempt (HALF_OPEN)
        breaker.call(lambda: None)   # successful
    assert breaker.state == CircuitState.CLOSED
    assert breaker.failure_count == 0
```

### Module Coverage Target

From test design: ≥85% line coverage on `services/data-pipeline/src/data_pipeline/ai_gateway_client/`

### `pyproject.toml` Additions

Add to `[project.optional-dependencies] dev`:
```toml
"respx>=0.21",
"freezegun>=1.5",
```

### Risk Mitigations Addressed in This Story

| Risk | Mitigation in S05.03 |
|---|---|
| **E05-R-002** (AI Gateway cascade failure) | Circuit breaker: after 5 consecutive gateway failures, opens circuit and raises `AIGatewayUnavailableError` without HTTP calls; Celery tasks can then retry at task level with `autoretry_for=(AIGatewayUnavailableError,)` |
| **E05-R-002** (orphaned `crawler_runs`) | `AIGatewayUnavailableError` is a clear exception type that S05.04–S05.06 can catch in `on_failure` handlers to transition `crawler_runs.status` to `failed` |

### Project Structure After This Story

```
eusolicit-app/
└── services/data-pipeline/
    ├── pyproject.toml                                # MODIFY — add respx, freezegun to dev deps
    └── src/data_pipeline/
        ├── ai_gateway_client/                        # + NEW package
        │   ├── __init__.py                           # + NEW — re-exports
        │   ├── circuit_breaker.py                    # + NEW — sync CircuitBreaker, get_breaker, reset_all_breakers
        │   ├── client.py                             # + NEW — AIGatewayClient, _TIMEOUT_MAP, get_client
        │   ├── exceptions.py                         # + NEW — AIGatewayUnavailableError
        │   ├── models.py                             # + NEW — AgentType, AgentCallResponse
        │   └── retry.py                              # + NEW — with_retry, _GatewayHTTPError
        └── workers/tasks/
            ├── crawl_aop.py                          # MODIFY — add autoretry_for import
            ├── crawl_ted.py                          # MODIFY — add autoretry_for import
            ├── crawl_eu_grants.py                    # MODIFY — add autoretry_for import
            └── cleanup.py                            # MODIFY — add autoretry_for import
    └── tests/
        └── unit/
            └── test_ai_gateway_client.py             # + NEW
```

**DO NOT TOUCH:**
- `services/ai-gateway/` — E04 code; the pipeline client calls its API, does not modify it
- `services/data-pipeline/alembic/` — migration chain must stay intact
- `services/data-pipeline/src/data_pipeline/models/` — from S05.01, do not modify
- `services/data-pipeline/src/data_pipeline/workers/celery_app.py` — no changes needed
- `services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` — no changes needed

### Cross-Story Dependencies

| Story | Status | Relevance to S05.03 |
|---|---|---|
| S05.01 — Pipeline Schema Migration | **DONE** | Models exist; `CrawlerRun` model from S05.01 is written by S05.04+ |
| S05.02 — Celery App Bootstrap | **DONE** | Task stubs exist in `workers/tasks/`; S05.03 updates their decorators |
| E04 — AI Gateway Service | **DONE** | AI Gateway running at `http://ai-gateway:8004`; `AI_GATEWAY_BASE_URL` env var set in Docker Compose |
| S05.04 — AOP Crawler Task | NEXT | Imports `AIGatewayClient` from this module; this story is a hard dependency |
| S05.05 — TED Crawler Task | NEXT | Same dependency |
| S05.06 — EU Grants Crawler Task | NEXT | Same dependency |
| S05.07 — Relevance Scoring Task | NEXT | Uses `AgentType.SCORING` timeout configuration |
| S05.08 — Submission Guide Task | NEXT | Uses `AgentType.SUBMISSION_GUIDE` timeout configuration |

### Important Implementation Notes

**No async in this module**: Celery tasks invoke the client synchronously from worker threads. Using async would require running a new event loop per task call, which is fragile and adds overhead. The synchronous `httpx.Client` (context manager per call) is correct here.

**No connection pool singleton**: Unlike the AI Gateway's singleton `_client`, the data pipeline client creates a new `httpx.Client` context manager per call. This is acceptable for Celery workers (calls are infrequent; each task is a heavyweight unit of work). A singleton async client is not needed and would require lifecycle management that complicates the Celery task model.

**Circuit breaker is per-agent, not per-service**: Each `agent_name` gets its own `CircuitBreaker` instance in `_breakers`. This means the AOP crawler can be marked as failing independently of the normalization team agent. The module-level `reset_all_breakers()` is for test teardown only — never call it in production code.

**Env var reload pattern**: `_TIMEOUT_MAP` is built at **module import time** from `os.environ`. For test overrides, use `monkeypatch.setenv(...)` + `importlib.reload(client)`. The test in E05-P2-004 demonstrates this pattern — same as E05-P1-006 for beat_schedule.

**`respx` mocking**: `respx` intercepts `httpx` calls at the transport layer without requiring a running server. Use `respx.mock` as a context manager. The pipeline client creates a new `httpx.Client` per call — this works naturally with `respx.mock`.

### Environment Variables Reference

| Variable | Default | Description |
|---|---|---|
| `AI_GATEWAY_BASE_URL` | `http://ai-gateway:8004` | AI Gateway base URL |
| `AI_GATEWAY_API_KEY` | `""` | Optional internal auth key |
| `AI_GATEWAY_MAX_RETRIES` | `3` | Max retry attempts per call |
| `AI_GATEWAY_TIMEOUT_CRAWLER` | `120` | Timeout (s) for crawler agents |
| `AI_GATEWAY_TIMEOUT_NORMALIZATION` | `60` | Timeout (s) for normalization team |
| `AI_GATEWAY_TIMEOUT_SCORING` | `60` | Timeout (s) for scoring agent |
| `AI_GATEWAY_TIMEOUT_SUBMISSION_GUIDE` | `90` | Timeout (s) for submission guide agent |
| `AI_GATEWAY_CIRCUIT_THRESHOLD` | `5` | Failures before circuit opens |
| `AI_GATEWAY_CIRCUIT_COOLDOWN` | `60` | Seconds before half-open transition |

### References

- [Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.03]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#P1-E05-P1-008,009,010,011, P2-E05-P2-004]
- [Source: eusolicit-app/services/ai-gateway/src/ai_gateway/services/circuit_breaker.py] — pattern template (async); adapt to sync for Celery
- [Source: eusolicit-app/services/ai-gateway/src/ai_gateway/services/retry.py] — retry pattern template; adapt to sync
- [Source: eusolicit-app/services/ai-gateway/src/ai_gateway/routers/execution.py] — POST /workflows/{id}/run endpoint being called
- [Source: eusolicit-docs/implementation-artifacts/5-2-celery-app-bootstrap-and-beat-schedule-configuration.md] — task stub pattern and decorator conventions
- [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py] — current stub to update in Task 5

---

## Dev Agent Record

### Implementation Plan

Implemented the `ai_gateway_client` package as a fully synchronous, thread-safe HTTP client for calling the AI Gateway from Celery worker tasks.

Key decisions:
- `_GatewayHTTPError` defined in `retry.py` and imported by `client.py` — keeps the retry module self-contained while avoiding circular imports.
- `_TIMEOUT_MAP` is a module-level dict populated at import time from `os.environ`, enabling `importlib.reload(client)` to pick up env-var overrides in tests (matching the E05-P1-006 pattern used in beat_schedule tests).
- `threading.Lock` used throughout circuit breaker — not `asyncio.Lock` — because Celery workers are threaded, not async.
- `respx.mock` context manager used for HTTP mocking; call count captured inside the context before it resets (respx clears `calls` on context exit).
- Added `pytest.ini_options.markers` for `unit`, `integration`, `smoke`, `slow`, `cross_service` to eliminate `PytestUnknownMarkWarning`.

### Completion Notes

✅ All 7 tasks and 16 subtasks completed.  
✅ 13 unit tests passing (E05-P1-008, E05-P1-009, E05-P1-010, E05-P1-011, E05-P2-004 + 8 additional edge-case tests).  
✅ Full regression suite: 31/31 tests pass — zero regressions.  
✅ All 5 acceptance criteria satisfied.  
✅ `respx 0.23.1` and `freezegun 1.5.5` installed and verified.

---

## File List

### New Files
- `eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/__init__.py`
- `eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/circuit_breaker.py`
- `eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py`
- `eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/exceptions.py`
- `eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/models.py`
- `eusolicit-app/services/data-pipeline/src/data_pipeline/ai_gateway_client/retry.py`
- `eusolicit-app/services/data-pipeline/tests/unit/test_ai_gateway_client.py`

### Modified Files
- `eusolicit-app/services/data-pipeline/pyproject.toml` — added `respx>=0.21`, `freezegun>=1.5` to dev deps; added pytest markers
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` — `autoretry_for=(AIGatewayUnavailableError,)` + `retry_backoff=True`
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` — same
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` — same
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/cleanup.py` — same

---

## Senior Developer Review

**Review Date:** 2026-04-14
**Reviewer:** Adversarial Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** APPROVED (all blocking findings resolved)

### Review Findings

- [x] [Review][Patch] BUG-001: Wrong exception types in `retry.py` — httpx network/timeout errors are NOT retried [retry.py:67-68]
- [x] [Review][Patch] BUG-001b: Add test for httpx transport error retry path [test_ai_gateway_client.py]
- [x] [Review][Defer] OBS-001: 4xx errors increment circuit breaker failure count [circuit_breaker.py] — deferred, pre-existing design choice
- [x] [Review][Defer] OBS-002: Test uses raw `os.environ.pop` instead of monkeypatch [test_ai_gateway_client.py] — deferred, minor test hygiene

### Blocking Finding

#### BUG-001: Wrong exception types in `retry.py` — httpx network/timeout errors are NOT retried

**File:** `services/data-pipeline/src/data_pipeline/ai_gateway_client/retry.py`, line 67
**Severity:** Blocking
**AC affected:** AC1, AC4
**Confirmed by:** Coverage report shows lines 67-68 at 0% coverage (dead code)

**Problem:** `with_retry` catches `(ConnectionError, TimeoutError)` — Python built-in exceptions — but `httpx` raises `httpx.ConnectError` and `httpx.TimeoutException`, which are **not subclasses** of those built-ins. Verified:

```python
>>> issubclass(httpx.TimeoutException, TimeoutError)
False
>>> issubclass(httpx.ConnectError, ConnectionError)
False
```

httpx exception hierarchy:
- `httpx.ConnectError` → `NetworkError` → `TransportError` → `RequestError` → `HTTPError` → `Exception`
- `httpx.TimeoutException` → `TransportError` → `RequestError` → `HTTPError` → `Exception`

**Impact:** When the AI Gateway is down (connection refused) or slow (timeout), the retry logic does NOT retry. The exception propagates as an unhandled `httpx.ConnectError` or `httpx.TimeoutException`. Since it's not `AIGatewayUnavailableError`, Celery's `autoretry_for` won't catch it either. The entire resilience chain — retry → circuit breaker → Celery autoretry — is broken for the most common real-world failure mode.

**Root cause:** The story spec (Task 3.1) prescribes `ConnectionError`/`TimeoutError` as the retryable types. This is correct for `urllib3`/`requests` but incorrect for `httpx`, which has its own exception hierarchy.

**Fix required:**

1. In `retry.py` line 67, change:
   ```python
   except (ConnectionError, TimeoutError) as exc:
   ```
   to:
   ```python
   except httpx.TransportError as exc:
   ```
   This covers all transport-level failures: DNS resolution, connect refused, read/write/pool timeout, connection reset. Add `import httpx` to the file.

2. Add a test (`test_retry_on_transport_error`) that raises `httpx.ConnectError` and verifies retry behavior. Add a second test (`test_retry_on_timeout`) that raises `httpx.TimeoutException` and verifies retry + `AIGatewayUnavailableError` on exhaustion.

### Non-Blocking Observations (Deferrable)

#### OBS-001: 4xx errors increment circuit breaker failure count

`response.raise_for_status()` on 4xx raises `httpx.HTTPStatusError`. The circuit breaker's generic `except Exception:` counts this as a failure. Repeated 400 Bad Request errors (a client-side bug) could trip the circuit open, masking the real issue. Consider catching and re-raising 4xx errors before entering the circuit breaker, or only counting 5xx/transport errors as circuit failures. **Deferrable** — unlikely in practice since payloads are controlled by pipeline code.

#### OBS-002: Test uses raw `os.environ.pop` instead of monkeypatch

`test_timeout_default_values` calls `os.environ.pop(var, None)` directly. If the test fails mid-execution, env vars may leak to subsequent tests. Use `monkeypatch.delenv(var, raising=False)` instead. **Deferrable** — minor test hygiene.

### Metrics

| Metric | Result | Target |
|--------|--------|--------|
| Tests passing | 15/15 | All |
| Regression suite | 36/36 | All |
| Line coverage (package) | 98% overall; retry.py 100% (dead-code lines fixed) | ≥85% |
| Architecture alignment | ✅ Sync httpx, threading.Lock, per-agent breakers | — |
| AC satisfaction | 5/5 (BUG-001 resolved: AC1 and AC4 now fully met) | 5/5 |

---

## Change Log

- 2026-04-14: Implemented S05.03 — created `ai_gateway_client` package with `AIGatewayClient`, `CircuitBreaker`, `with_retry`, `AIGatewayUnavailableError`, `AgentType`, `AgentCallResponse`; updated four task stubs with `autoretry_for`; added 13 unit tests covering E05-P1-008/009/010/011 and E05-P2-004; added respx+freezegun dev deps. (Dev Agent)
- 2026-04-14: Resolved BUG-001 (blocking review finding) — fixed `retry.py` to catch `httpx.TransportError` instead of Python built-in `(ConnectionError, TimeoutError)`; added `import httpx`; added two new tests `test_retry_on_transport_error` and `test_retry_on_timeout_exhausted` (BUG-001b). Result: 15/15 unit tests pass, 36/36 full regression suite passes, `retry.py` coverage 100%, package coverage 98%. All 5 ACs satisfied. (Dev Agent)
