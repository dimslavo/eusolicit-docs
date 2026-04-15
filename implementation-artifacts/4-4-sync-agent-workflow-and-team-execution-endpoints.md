# Story 4.4: Sync Agent, Workflow, and Team Execution Endpoints

Status: done

## Change Log

- 2026-04-14: Story created from epic-04-ai-gateway-service.md S04.04 with test design context from test-design-epic-04.md. Agent: Claude Sonnet 4.6
- 2026-04-14: Story implemented — all ACs satisfied, 15 new unit tests + 26 existing pass (41 total). Agent: Claude Sonnet 4.6
- 2026-04-14: Senior dev review — APPROVED. 0 patches, 1 deferred (missing KraftDataConnectionError test). Agent: Claude Opus 4.6

## Story

As a **backend developer on the EU Solicit platform**,
I want **four synchronous proxy endpoints (`POST /agents/{id}/run`, `POST /workflows/{id}/run`, `POST /teams/{id}/run`, `POST /storage/{id}/files`) that resolve logical names via the agent registry, validate entry types, forward requests to KraftData, and return structured error responses for missing headers, type mismatches, upstream failures, and timeouts**,
so that **all downstream EU Solicit services (Client API, Admin API, Data Pipeline) have a single, type-safe internal entry point for synchronous KraftData executions, logical name resolution is decoupled from KraftData UUIDs, and callers receive actionable HTTP errors instead of opaque failures**.

## Acceptance Criteria

1. `POST /agents/{id}/run` with a logical name (e.g., `executive-summary`) resolves the name via `AgentRegistry.resolve()`, calls `POST /client/api/v1/agents/{agentId}/run` on KraftData with the resolved UUID, and returns the KraftData 200 response body deserialized as `AgentRunResponse`
2. `POST /agents/{id}/run` with a raw UUID v4 (e.g., `550e8400-e29b-41d4-a716-446655440000`) bypasses the registry entirely — `AgentRegistry.resolve()` is NOT called — and routes directly to KraftData using the UUID as-is
3. `POST /workflows/{id}/run` resolves the logical name, validates that the registry entry has `type="workflow"`, and calls `POST /client/api/v1/workflows/{workflowId}/run`; calling this endpoint with an entry whose type is `"agent"` or `"team"` returns HTTP 400 with `{"error": "agent_type_mismatch", "logical_name": "<id>", "expected_type": "workflow", "actual_type": "<actual>"}`
4. `POST /teams/{id}/run` resolves the logical name, validates that the registry entry has `type="team"`, and calls `POST /client/api/v1/teams/{teamId}/run`; type mismatch returns HTTP 400 with the same `agent_type_mismatch` error body
5. `POST /storage/{id}/files` accepts a multipart file upload (`UploadFile`); forwards the file to `POST /client/api/v1/storage-resources/{id}/files` on KraftData; returns the KraftData response deserialized as `StorageResourceResponse`
6. All four endpoints require the `X-Caller-Service` header; a request missing this header returns HTTP 400 with `{"error": "missing_x_caller_service_header"}` — a FastAPI shared dependency enforces this before any business logic runs
7. `X-Request-ID` header is propagated to KraftData unchanged if present in the incoming request; if absent, a new UUID v4 is generated, injected into the KraftData upstream call, and included in the response to the caller via the `X-Request-ID` response header
8. A KraftData `KraftDataAPIError` (HTTP 5xx from upstream) is mapped to HTTP 502 with body `{"error": "upstream_error", "status_code": <kd_status>, "detail": "<kd_body>"}`
9. A KraftData `KraftDataTimeoutError` is mapped to HTTP 504 with body `{"error": "upstream_timeout"}`
10. A KraftData `KraftDataConnectionError` is mapped to HTTP 502 with body `{"error": "upstream_connection_error"}`
11. An unknown logical name (not in registry) returns HTTP 404 — handled by the existing `AgentNotFoundError` exception handler registered in `main.py` (no additional work required)
12. `AgentTypeMismatchError` is defined in `src/ai_gateway/services/exceptions.py`, re-exported from `src/ai_gateway/services/__init__.py`, and registered in `main.py` with an exception handler that maps it to HTTP 400
13. All four execution endpoints are defined in `src/ai_gateway/routers/execution.py`; the router is registered in `main.py` with no prefix (routes at top level: `/agents/{id}/run`, etc.)
14. Request models (`AgentRunRequest`, `WorkflowRunRequest`, `TeamRunRequest`) and response models (`AgentRunResponse`, `WorkflowRunResponse`, `TeamRunResponse`, `StorageResourceResponse`) are imported from the existing `eusolicit_kraftdata` package — no new model files are created

## Tasks / Subtasks

- [x] Task 1: Add `AgentTypeMismatchError` to `src/ai_gateway/services/exceptions.py` (AC: 3, 4, 12)
  - [x] 1.1 Define `AgentTypeMismatchError(Exception)` with `__init__(self, logical_name: str, expected_type: str, actual_type: str)` — message: `f"Registry entry '{logical_name}' has type '{actual_type}', expected '{expected_type}'"`; store all three as instance attributes
  - [x] 1.2 Add `AgentTypeMismatchError` to `__all__` in `services/__init__.py` and add the import re-export alongside the existing exceptions

- [x] Task 2: Register `AgentTypeMismatchError` exception handler in `src/ai_gateway/main.py` (AC: 12)
  - [x] 2.1 Import `AgentTypeMismatchError` from `ai_gateway.services.exceptions`
  - [x] 2.2 Add `@app.exception_handler(AgentTypeMismatchError)` that returns `JSONResponse(status_code=400, content={"error": "agent_type_mismatch", "logical_name": exc.logical_name, "expected_type": exc.expected_type, "actual_type": exc.actual_type})`

- [x] Task 3: Create `src/ai_gateway/routers/execution.py` — sync execution proxy router (AC: 1–14)
  - [x] 3.1 Create `router = APIRouter(tags=["Execution"])` at module top
  - [x] 3.2 Import UUID v4 pattern from `agent_registry` module: re-use `_UUID4_RE` regex, or define a module-local constant `_UUID4_RE = re.compile(r"^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$", re.IGNORECASE)` — do NOT import the private `_UUID4_RE` from agent_registry; define it locally in execution.py
  - [x] 3.3 Implement `_require_caller_service` FastAPI dependency:
    - Signature: `async def _require_caller_service(x_caller_service: Annotated[str | None, Header()] = None) -> str`
    - If `x_caller_service` is `None` or empty: `raise HTTPException(status_code=400, detail={"error": "missing_x_caller_service_header"})`
    - Return `x_caller_service`
  - [x] 3.4 Implement `_get_or_generate_request_id` helper function:
    - Signature: `def _get_or_generate_request_id(x_request_id: str | None) -> str`
    - If `x_request_id` is not None and not empty: return it unchanged
    - Otherwise: return `str(uuid4())`
  - [x] 3.5 Implement `_resolve_kraftdata_id(id: str, expected_type: str | None = None) -> str` helper:
    - If `_UUID4_RE.match(id)`: return `id` directly (UUID passthrough — no registry lookup)
    - Otherwise: call `get_registry().resolve(id)` — raises `AgentNotFoundError` (→ 404) if not found
    - If `expected_type` is set and `entry.type != expected_type`: raise `AgentTypeMismatchError(id, expected_type, entry.type)`
    - Return `entry.kraftdata_id`
  - [x] 3.6 Implement `_map_kraftdata_error(exc: Exception) -> HTTPException` helper:
    - `KraftDataAPIError` → `HTTPException(502, {"error": "upstream_error", "status_code": exc.status_code, "detail": exc.body})`
    - `KraftDataTimeoutError` → `HTTPException(504, {"error": "upstream_timeout"})`
    - `KraftDataConnectionError` → `HTTPException(502, {"error": "upstream_connection_error"})`
    - Re-raise unknown exceptions unchanged
  - [x] 3.7 Implement `POST /agents/{id}/run` endpoint (AC: 1, 2, 6, 7, 8, 9, 10):
    ```python
    @router.post("/agents/{id}/run", response_model=AgentRunResponse)
    async def run_agent(
        id: str,
        body: AgentRunRequest,
        request: Request,
        caller_service: str = Depends(_require_caller_service),
    ) -> AgentRunResponse:
    ```
    - Call `_resolve_kraftdata_id(id, expected_type=None)` → `kraftdata_id`
    - Derive `request_id = _get_or_generate_request_id(request.headers.get("x-request-id"))`
    - Set `forward_headers = {"X-Caller-Service": caller_service, "X-Request-ID": request_id}`
    - Wrap KraftData call in try/except using `_map_kraftdata_error`; call `await call_kraftdata("POST", f"/client/api/v1/agents/{kraftdata_id}/run", json=body.model_dump(), headers=forward_headers)`
    - Deserialize response as `AgentRunResponse(**response.json())`
    - Return `JSONResponse(content=..., headers={"X-Request-ID": request_id})`
  - [x] 3.8 Implement `POST /workflows/{id}/run` endpoint (AC: 3, 6, 7, 8, 9, 10):
    - Same pattern as agents endpoint; call `_resolve_kraftdata_id(id, expected_type="workflow")`
    - KraftData route: `f"/client/api/v1/workflows/{kraftdata_id}/run"`
    - Request/response model: `WorkflowRunRequest` / `WorkflowRunResponse`
  - [x] 3.9 Implement `POST /teams/{id}/run` endpoint (AC: 4, 6, 7, 8, 9, 10):
    - Same pattern; call `_resolve_kraftdata_id(id, expected_type="team")`
    - KraftData route: `f"/client/api/v1/teams/{kraftdata_id}/run"`
    - Request/response model: `TeamRunRequest` / `TeamRunResponse`
  - [x] 3.10 Implement `POST /storage/{id}/files` endpoint (AC: 5, 6, 7, 8, 9, 10):
    - Accept `id: str` (raw KraftData storage resource ID — no registry lookup), `file: UploadFile`
    - Accept `caller_service: str = Depends(_require_caller_service)` and `request: Request`
    - Derive `request_id` as above
    - Read file bytes: `content = await file.read()`
    - Call `await call_kraftdata("POST", f"/client/api/v1/storage-resources/{id}/files", content=content, headers={"X-Caller-Service": caller_service, "X-Request-ID": request_id, "Content-Type": file.content_type or "application/octet-stream"})`
    - Deserialize response as `StorageResourceResponse(**response.json())`
    - Return `JSONResponse(content=..., headers={"X-Request-ID": request_id})`
    - **NOTE**: For the storage endpoint, the `Content-Type` header passed per-request overrides the client default `application/json`; pass the file's content_type explicitly in the headers dict

- [x] Task 4: Register execution router in `src/ai_gateway/main.py` (AC: 13)
  - [x] 4.1 Import execution router: `from ai_gateway.routers import execution as execution_router`
  - [x] 4.2 Register: `app.include_router(execution_router.router)` — no prefix; routes land at `/agents/{id}/run`, `/workflows/{id}/run`, `/teams/{id}/run`, `/storage/{id}/files`

- [x] Task 5: Write unit tests — `tests/unit/test_execution_router.py` (AC: 1–10; test IDs E04-P0-005, E04-P0-006, E04-P1-001, E04-P1-003, E04-P1-004, E04-P1-005)
  - [x] 5.1 `test_agent_run_logical_name_resolves_uuid` (E04-P0-005):
    - Use `respx.mock` to mock `POST https://mock-kraftdata.test/client/api/v1/agents/<EXPECTED_UUID>/run` → 200 with `AgentRunResponse` JSON
    - POST to `/agents/executive-summary/run` with valid `AgentRunRequest` body and required headers
    - Assert `respx` caught the request at the correct URL (UUID in path, not logical name)
    - Assert response body matches mock
  - [x] 5.2 `test_agent_run_uuid_passthrough_no_registry_call` (E04-P1-001):
    - Mock `AgentRegistry.resolve` on the singleton to spy (should NOT be called)
    - POST to `/agents/550e8400-e29b-41d4-a716-446655440000/run` with `respx` mock for that exact UUID
    - Assert `resolve` was NOT called (via `unittest.mock.patch` or spy)
    - Assert KraftData request URL contains the raw UUID
  - [x] 5.3 `test_workflow_run_valid_type` (E04-P0-006):
    - Registry has a workflow entry; `respx` mocks the workflow KraftData URL
    - POST to `/workflows/<logical-name>/run` → assert 200 and correct URL in upstream call
  - [x] 5.4 `test_workflow_run_type_mismatch_returns_400` (E04-P0-006):
    - Registry has an agent-type entry with the given logical name
    - POST to `/workflows/<agent-logical-name>/run` → assert HTTP 400
    - Assert response body: `{"error": "agent_type_mismatch", "expected_type": "workflow", "actual_type": "agent"}`
  - [x] 5.5 `test_team_run_type_mismatch_returns_400` (E04-P0-006):
    - Registry has a workflow-type entry with the given logical name
    - POST to `/teams/<workflow-logical-name>/run` → assert HTTP 400
    - Assert response body: `{"error": "agent_type_mismatch", "expected_type": "team", "actual_type": "workflow"}`
  - [x] 5.6 `test_missing_caller_service_returns_400` (E04-P1-003):
    - POST to `/agents/executive-summary/run` **without** `X-Caller-Service` header
    - Assert HTTP 400, body `{"error": "missing_x_caller_service_header"}`
    - Repeat assertion for `/workflows/`, `/teams/`, `/storage/` (parametrized test)
  - [x] 5.7 `test_request_id_forwarded_when_present` (E04-P1-004):
    - POST with `X-Request-ID: my-custom-id`; use `respx` to capture KraftData request headers
    - Assert `X-Request-ID: my-custom-id` in upstream KraftData request headers
    - Assert `X-Request-ID: my-custom-id` in response headers
  - [x] 5.8 `test_request_id_generated_when_absent` (E04-P1-004):
    - POST without `X-Request-ID`; capture KraftData request headers via `respx`
    - Assert `X-Request-ID` present in upstream headers and is a valid UUID v4
    - Assert `X-Request-ID` present in response headers with same generated value
  - [x] 5.9 `test_kraftdata_500_returns_502` (E04-P1-005):
    - `respx` returns 500 for agent run endpoint
    - POST to `/agents/{id}/run` → assert HTTP 502
    - Assert body `{"error": "upstream_error", "status_code": 500, ...}`
  - [x] 5.10 `test_kraftdata_timeout_returns_504` (E04-P1-005):
    - `respx` raises `httpx.ReadTimeout` for agent run endpoint
    - POST to `/agents/{id}/run` → assert HTTP 504
    - Assert body `{"error": "upstream_timeout"}`
  - [x] 5.11 `test_unknown_agent_returns_404`:
    - POST to `/agents/nonexistent-agent/run` (no matching registry entry)
    - Assert HTTP 404, body `{"error": "agent_not_found", "logical_name": "nonexistent-agent"}`
  - [x] 5.12 `test_storage_upload_forwarded_to_kraftdata` (E04-P1-002):
    - `respx` mocks `POST .../client/api/v1/storage-resources/test-resource-id/files` → 200 with `StorageResourceResponse` JSON
    - POST multipart with small test file to `/storage/test-resource-id/files` with `X-Caller-Service` header
    - Assert correct KraftData URL in upstream request; assert `StorageResourceResponse` returned

## Senior Developer Review

**Reviewer:** Claude Code Review (2026-04-14)
**Verdict:** ✅ APPROVED

**Summary:** 0 `decision-needed`, 0 `patch`, 1 `defer`, 2 dismissed as noise.

All 14 acceptance criteria satisfied in code. 15/15 unit tests pass. Full test suite (45 tests) passes with zero regressions. Implementation precisely follows story spec.

### Review Findings

- [x] [Review][Defer] Missing `KraftDataConnectionError` → 502 unit test [tests/unit/test_execution_router.py] — AC 10 mapping is implemented correctly in `_handle_kraftdata_error()` but has no dedicated test. Story tasks (5.1–5.12) didn't specify one. Low risk: code path is structurally identical to the timeout test. Deferred to test coverage expansion story.

## Dev Notes

### CRITICAL: Module Path Convention (same as S04.01–S04.03)

The epic spec uses `app/routers/execution.py` — the **actual** monorepo path is:

| Epic spec path | Actual monorepo path |
|---|---|
| `app/routers/execution.py` | `eusolicit-app/services/ai-gateway/src/ai_gateway/routers/execution.py` |
| `app/services/exceptions.py` | `eusolicit-app/services/ai-gateway/src/ai_gateway/services/exceptions.py` |
| `app/main.py` | `eusolicit-app/services/ai-gateway/src/ai_gateway/main.py` |

### What Already Exists After S04.01–S04.03

| File | State | Action for S04.04 |
|---|---|---|
| `src/ai_gateway/main.py` | Complete — has lifespan, `AgentNotFoundError` handler, health + admin routers registered | **MODIFY** to add `AgentTypeMismatchError` handler and register execution router |
| `src/ai_gateway/config.py` | Complete — `concurrency_limit`, `kraftdata_base_url`, `kraftdata_api_key` defined | **DO NOT TOUCH** |
| `src/ai_gateway/services/exceptions.py` | Has `KraftDataError`, `KraftDataTimeoutError`, `KraftDataConnectionError`, `KraftDataAPIError`, `AgentNotFoundError` | **EXTEND** to add `AgentTypeMismatchError` |
| `src/ai_gateway/services/__init__.py` | Re-exports all 5 exceptions | **EXTEND** to add `AgentTypeMismatchError` |
| `src/ai_gateway/services/kraftdata_client.py` | Complete — `call_kraftdata()`, `get_client()`, `init_client()`, `close_client()` | **DO NOT TOUCH** |
| `src/ai_gateway/services/agent_registry.py` | Complete — `AgentRegistry`, `get_registry()`, `init_registry()`, `AgentEntry`, `_UUID4_RE` | **DO NOT TOUCH** (but copy the UUID regex pattern locally) |
| `src/ai_gateway/models/__init__.py` | Empty `"""AI Gateway models package."""` | **DO NOT TOUCH** — models come from `eusolicit_kraftdata` |
| `src/ai_gateway/routers/__init__.py` | Empty `"""AI Gateway routers package."""` | **DO NOT TOUCH** |
| `config/agents.yaml` | 29-entry registry (27 agents, 1 team, 1 workflow) | **DO NOT TOUCH** |
| `tests/conftest.py` | Has `gateway_engine`, `gateway_session`, `mock_kraftdata_base_url` fixtures | **DO NOT TOUCH** |

### Use `eusolicit_kraftdata` Models — Do NOT Create New Model Files

The `eusolicit-kraftdata` shared package (already in dependencies) contains all needed models:

```python
# For request bodies:
from eusolicit_kraftdata.requests import AgentRunRequest, WorkflowRunRequest, TeamRunRequest

# For response bodies:
from eusolicit_kraftdata.responses import AgentRunResponse, WorkflowRunResponse, TeamRunResponse, StorageResourceResponse
```

Do **not** create `src/ai_gateway/models/execution.py`. The `eusolicit-models` package contains EU Solicit domain models; `eusolicit-kraftdata` contains KraftData API models. This story uses the latter exclusively.

### `AgentTypeMismatchError` Design

```python
# src/ai_gateway/services/exceptions.py — append after AgentNotFoundError

class AgentTypeMismatchError(Exception):
    """Raised when an execution endpoint's required type does not match the registry entry.

    Attributes
    ----------
    logical_name:
        The logical name that was looked up in the registry.
    expected_type:
        The type required by the calling endpoint (e.g., "workflow", "team").
    actual_type:
        The type found in the registry entry.
    """

    def __init__(self, logical_name: str, expected_type: str, actual_type: str) -> None:
        self.logical_name = logical_name
        self.expected_type = expected_type
        self.actual_type = actual_type
        super().__init__(
            f"Registry entry '{logical_name}' has type '{actual_type}', expected '{expected_type}'"
        )
```

### Execution Router — Full Skeleton

```python
# src/ai_gateway/routers/execution.py
from __future__ import annotations

import re
import structlog
from typing import Annotated
from uuid import uuid4

from eusolicit_kraftdata.requests import AgentRunRequest, WorkflowRunRequest, TeamRunRequest
from eusolicit_kraftdata.responses import AgentRunResponse, WorkflowRunResponse, TeamRunResponse, StorageResourceResponse
from fastapi import APIRouter, Depends, Header, HTTPException, Request, UploadFile
from fastapi.responses import JSONResponse

from ai_gateway.services.agent_registry import get_registry
from ai_gateway.services.exceptions import (
    AgentTypeMismatchError,
    KraftDataAPIError,
    KraftDataConnectionError,
    KraftDataTimeoutError,
)
from ai_gateway.services.kraftdata_client import call_kraftdata

log = structlog.get_logger(__name__)

router = APIRouter(tags=["Execution"])

_UUID4_RE = re.compile(
    r"^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$",
    re.IGNORECASE,
)


async def _require_caller_service(
    x_caller_service: Annotated[str | None, Header()] = None,
) -> str:
    if not x_caller_service:
        raise HTTPException(status_code=400, detail={"error": "missing_x_caller_service_header"})
    return x_caller_service


def _get_or_generate_request_id(x_request_id: str | None) -> str:
    if x_request_id:
        return x_request_id
    return str(uuid4())


def _resolve_kraftdata_id(id: str, expected_type: str | None = None) -> str:
    if _UUID4_RE.match(id):
        return id  # UUID passthrough — no registry lookup
    entry = get_registry().resolve(id)  # raises AgentNotFoundError → 404
    if expected_type is not None and entry.type != expected_type:
        raise AgentTypeMismatchError(id, expected_type, entry.type)
    return entry.kraftdata_id


def _handle_kraftdata_error(exc: Exception) -> HTTPException:
    if isinstance(exc, KraftDataAPIError):
        return HTTPException(
            status_code=502,
            detail={"error": "upstream_error", "status_code": exc.status_code, "detail": exc.body},
        )
    if isinstance(exc, KraftDataTimeoutError):
        return HTTPException(status_code=504, detail={"error": "upstream_timeout"})
    if isinstance(exc, KraftDataConnectionError):
        return HTTPException(status_code=502, detail={"error": "upstream_connection_error"})
    raise exc  # unexpected — propagate
```

### Error Handling Pattern for Each Endpoint

All three run endpoints follow this pattern:

```python
@router.post("/agents/{id}/run", response_model=AgentRunResponse)
async def run_agent(
    id: str,
    body: AgentRunRequest,
    request: Request,
    caller_service: Annotated[str, Depends(_require_caller_service)],
) -> JSONResponse:
    kraftdata_id = _resolve_kraftdata_id(id, expected_type=None)  # no type check for agents
    request_id = _get_or_generate_request_id(request.headers.get("x-request-id"))
    forward_headers = {"X-Caller-Service": caller_service, "X-Request-ID": request_id}
    try:
        resp = await call_kraftdata(
            "POST",
            f"/client/api/v1/agents/{kraftdata_id}/run",
            json=body.model_dump(),
            headers=forward_headers,
        )
    except (KraftDataAPIError, KraftDataTimeoutError, KraftDataConnectionError) as exc:
        raise _handle_kraftdata_error(exc)
    result = AgentRunResponse(**resp.json())
    return JSONResponse(content=result.model_dump(), headers={"X-Request-ID": request_id})
```

### Storage Upload — Content-Type Handling

The `call_kraftdata()` httpx client has `Content-Type: application/json` as a **default** header. For the storage upload endpoint, the file's actual MIME type must override this. Pass `Content-Type` explicitly in the `headers` dict — httpx merges per-request headers and the request-level value wins:

```python
resp = await call_kraftdata(
    "POST",
    f"/client/api/v1/storage-resources/{id}/files",
    content=content,  # raw bytes
    headers={
        "X-Caller-Service": caller_service,
        "X-Request-ID": request_id,
        "Content-Type": file.content_type or "application/octet-stream",
    },
)
```

Do **not** use `files={"file": ...}` with httpx here — pass `content=bytes` and set `Content-Type` explicitly to avoid httpx multipart encoding overhead and correctly match the KraftData API's expected format.

### Test Setup for Unit Tests

Tests in `tests/unit/test_execution_router.py` need both `respx` and a fully-initialised registry. Use the following pattern:

```python
import pytest
import respx
import httpx
from pathlib import Path
from fastapi.testclient import TestClient

from ai_gateway.main import app
from ai_gateway.services import agent_registry as registry_module

# Load a minimal fixture YAML for test agents
FIXTURE_YAML = """
agents:
  executive-summary:
    kraftdata_id: "550e8400-e29b-41d4-a716-446655440000"
    type: agent
    description: "Test agent"
  test-workflow:
    kraftdata_id: "550e8400-e29b-41d4-a716-446655440001"
    type: workflow
    description: "Test workflow"
  test-team:
    kraftdata_id: "550e8400-e29b-41d4-a716-446655440002"
    type: team
    description: "Test team"
"""

@pytest.fixture
def test_client(tmp_path):
    # Write fixture YAML and init registry
    yaml_file = tmp_path / "agents.yaml"
    yaml_file.write_text(FIXTURE_YAML)
    # Use lifespan override or directly init registry for unit tests
    ...
```

**Note**: Because `main.py` lifespan initialises both the httpx client and registry, unit tests should either:
- Use `TestClient(app)` with a properly initialised service (requires env vars), OR
- Patch `get_registry()` and `call_kraftdata()` directly with `unittest.mock.patch` to avoid needing real services

The recommended approach for these unit tests is to patch `call_kraftdata` and `get_registry` so tests are fast and hermetic, rather than spinning up the full app.

### Exception Handler Registration Order in `main.py`

FastAPI exception handlers are checked in registration order. Add the `AgentTypeMismatchError` handler immediately after the existing `AgentNotFoundError` handler to keep related handlers grouped:

```python
@app.exception_handler(AgentNotFoundError)
async def agent_not_found_handler(request: Request, exc: AgentNotFoundError) -> JSONResponse:
    ...

@app.exception_handler(AgentTypeMismatchError)
async def agent_type_mismatch_handler(request: Request, exc: AgentTypeMismatchError) -> JSONResponse:
    return JSONResponse(
        status_code=400,
        content={
            "error": "agent_type_mismatch",
            "logical_name": exc.logical_name,
            "expected_type": exc.expected_type,
            "actual_type": exc.actual_type,
        },
    )
```

### Relevant Test IDs from Test Design (Epic 4)

| Test ID | Priority | Scope | Covered by Story Task |
|---------|----------|-------|----------------------|
| E04-P0-005 | P0 | Logical name resolve + KraftData forward | Task 5.1 |
| E04-P0-006 | P0 | Workflow/team type validation enforcement | Task 5.3, 5.4, 5.5 |
| E04-P1-001 | P1 | UUID passthrough (no registry) | Task 5.2 |
| E04-P1-002 | P1 | Storage multipart upload forward | Task 5.12 |
| E04-P1-003 | P1 | Missing X-Caller-Service → 400 | Task 5.6 |
| E04-P1-004 | P1 | X-Request-ID propagation / generation | Task 5.7, 5.8 |
| E04-P1-005 | P1 | KraftData 5xx → 502, timeout → 504 | Task 5.9, 5.10 |
