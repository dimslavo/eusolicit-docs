# Story 6.8: AI Summary Generation API (SSE Streaming)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **paid-tier user who wants AI-generated intelligence on an opportunity**,
I want **`POST /api/v1/opportunities/{opportunity_id}/ai-summary` to check my usage quota atomically, call the Executive Summary Agent via the AI Gateway, stream the response back via Server-Sent Events (delta / metadata / done / error events), persist the completed summary to `client.ai_summaries`, and support `GET .../ai-summary` to retrieve a cached summary (< 24 h old) without re-metering**,
so that **I receive streaming AI analysis without quota abuse, every summary is persisted for re-use, and cached summaries are returned instantly at no quota cost**.

## Acceptance Criteria

1. `POST /api/v1/opportunities/{opportunity_id}/ai-summary` requires a valid Bearer JWT; returns 401 if absent or invalid.

2. Free-tier users receive 403 with `{"error": "tier_limit", "upgrade_url": "..."}` — the same block applied to the detail endpoint.

3. UsageGate check executes **before** the `StreamingResponse` is created:
   - Executes `check_and_increment(feature="ai_summary", redis=redis, response=response)` atomically via the Lua script already in `usage_gate.py`.
   - If limit is exceeded: returns **JSON** 429 with body `{"error": "usage_limit_exceeded", "limit": N, "used": M, "resets_at": "ISO8601", "upgrade_url": "..."}` and header `X-Usage-Remaining: 0`.
   - On success: sets `X-Usage-Remaining: N` header on the response.
   - Enterprise-tier users bypass the usage gate entirely (unlimited).

4. Opportunity data for the agent payload is fetched from `pipeline.opportunities` using a read-only session before streaming begins (reuses the existing `get_pipeline_readonly_session` dependency).

5. The endpoint calls AI Gateway `POST /agents/executive-summary/run-stream` via a new `AiGatewayClient.stream_agent()` method that uses `httpx` async streaming.

6. The response is a `StreamingResponse` with `media_type="text/event-stream"` and headers `Cache-Control: no-cache`, `X-Accel-Buffering: no`. SSE events yielded:
   - `event: delta\ndata: {"text": "<chunk>"}\n\n` — one or more text chunks
   - `event: metadata\ndata: {"model": "<str>", "tokens_used": <int>}\n\n` — exactly once, after all deltas
   - `event: done\ndata: {"summary_id": "<uuid>"}\n\n` — persisted summary UUID; signals stream end
   - `event: error\ndata: {"error": "<message>"}\n\n` — on AI Gateway failure; stream terminates

7. On receiving the `done` signal from the AI Gateway: the generator accumulates all `delta` text chunks and persists a new row to `client.ai_summaries` with `(id, opportunity_id, user_id, company_id, content, model, tokens_used, generated_at)`. The `done` event sent to the client carries the persisted `summary_id`.

8. `GET /api/v1/opportunities/{opportunity_id}/ai-summary` retrieves the most-recent cached summary for the requesting user:
   - Queries `client.ai_summaries` WHERE `opportunity_id = :opp_id AND user_id = :user_id` ORDER BY `generated_at DESC` LIMIT 1.
   - If found AND `generated_at >= NOW() - 24h` AND `pipeline.opportunities.updated_at <= generated_at` → return `{"summary_id": ..., "content": ..., "model": ..., "tokens_used": ..., "generated_at": ..., "cached": true}` with HTTP 200. **No usage decrement.**
   - If found but the opportunity's `updated_at > summary.generated_at` (stale) → return `{"cached": false, "stale": true, "reason": "opportunity_updated"}` with HTTP 200, signalling the client to trigger regeneration.
   - If not found or `generated_at < NOW() - 24h` → return `{"cached": false}` with HTTP 200.
   - Free-tier users return 403 (same tier block as POST).

9. Both endpoints appear in `/openapi.json` with correct path parameters, response schemas, and status code descriptions for 200/401/403/429.

10. Integration tests in `tests/api/test_ai_summary.py` cover:
    - E06-P0-008: cached summary (< 24 h) returned by GET without usage counter decrement
    - E06-P0-009: POST SSE streams `delta` → `metadata` → `done` events in order; summary persisted in DB with correct fields
    - E06-P1-021: GET returns `{"cached": true, ...}` without calling AI Gateway
    - E06-P1-022: AI Gateway failure streams `error` event; connection terminates gracefully
    - E06-P0-005: UsageGate 429 on limit exceeded with correct body and header
    - 401: unauthenticated request denied
    - 403: free-tier user denied (POST and GET)

## Tasks / Subtasks

- [x] Task 1: Add `stream_agent()` to `AiGatewayClient` in `ai_gateway_client.py` (AC: 5)
  - [x] 1.1 Add async context-manager method `stream_agent(agent_name, payload)` that opens `httpx.AsyncClient.stream("POST", f"{base_url}/agents/{agent_name}/run-stream", json=payload, headers={"X-Caller-Service": "client-api"})` with `Timeout(connect=60.0, read=None, write=30.0, pool=10.0)` and yields raw SSE lines as `AsyncGenerator[str, None]`.
  - [x] 1.2 Parse the byte stream line-by-line; yield complete non-empty lines; skip heartbeat lines (`event: heartbeat`).
  - [x] 1.3 Propagate `httpx.TimeoutException` as `AiGatewayTimeoutError` (already defined).
  - [x] 1.4 Propagate `httpx.ConnectError` and `httpx.RemoteProtocolError` as `AiGatewayUnavailableError` (already defined).

- [x] Task 2: Add `AISummaryGetResponse` schema to `schemas/opportunities.py` (AC: 8, 9)
  - [x] 2.1 Appended `AISummaryGetResponse` Pydantic model to `schemas/opportunities.py`.

- [x] Task 3: Add AI summary service functions to `opportunity_service.py` (AC: 4, 7, 8)
  - [x] 3.1 Add `get_cached_ai_summary(pipeline_session, client_session, opportunity_id, user_id)` — queries `client.ai_summaries` for the most recent summary, fetches `pipeline.opportunities.updated_at`, applies 24 h TTL and staleness logic, returns `AISummaryGetResponse`.
  - [x] 3.2 Add `persist_ai_summary(session, opportunity_id, user_id, company_id, content, model, tokens_used)` — inserts a row to `client.ai_summaries`, returns the new UUID.
  - [x] 3.3 Add `build_executive_summary_payload(opportunity_id, pipeline_session)` — fetches opportunity row from `pipeline.opportunities`, returns the agent payload dict with title, description, contracting_authority, budget_min, budget_max, deadline, cpv_codes, region, opportunity_type.

- [x] Task 4: Add `POST` and `GET` routes to `opportunities.py` router (AC: 1–9)
  - [x] 4.1 Add `POST /{opportunity_id}/ai-summary` SSE endpoint:
    - Uses JWT `current_user.subscription_tier` directly (not DB-backed `get_opportunity_tier_gate`) for tier check — consistent with document endpoints; allows test users without DB subscriptions.
    - Blocks free-tier (AC2) before any async work.
    - Creates `UsageGateContext` manually and calls `check_and_increment("ai_summary", redis, response)` before creating `StreamingResponse` (AC3). Catches `AppException(429)` and returns flat `JSONResponse` with `X-Usage-Remaining: 0`.
    - Fetches opportunity data via `opportunity_service.build_executive_summary_payload(opportunity_id, pipeline_session)`.
    - Acquires SSE semaphore (E06-R-004 mitigation) with 50 ms timeout; returns 503 if cap reached.
    - Forwards `X-Usage-Remaining` from FastAPI `response.headers` into `StreamingResponse` headers.
    - Returns `StreamingResponse(_generate_summary_stream(...), media_type="text/event-stream", headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no", ...})`.

  - [x] 4.2 Implement `_generate_summary_stream()` as an async generator:
    - Opens `gw_client.stream_agent("executive-summary", payload)` context.
    - Parses incoming SSE lines, accumulates `delta` text, captures `metadata` fields.
    - Yields `delta`, `metadata` events to client as-is.
    - On `done` from AI Gateway: calls `persist_ai_summary(...)` with a fresh `get_session_factory()` session; yields `event: done\ndata: {"summary_id": "<uuid>"}\n\n`.
    - On `error` from AI Gateway or exception: yields `event: error\ndata: {"error": "<msg>"}\n\n`.
    - Releases semaphore in `finally` block.

  - [x] 4.3 Add `GET /{opportunity_id}/ai-summary` route:
    - Blocks free-tier (403).
    - Calls `opportunity_service.get_cached_ai_summary(...)`.
    - Returns `AISummaryGetResponse`.

- [x] Task 5: ATDD tests in `tests/api/test_ai_summary.py` — pre-written; removed all 13 `@pytest.mark.skip` decorators (GREEN phase). All 15 tests pass.

## Dev Notes

### Architecture Overview

S06.08 sits at the intersection of four existing subsystems: UsageGate (S06.03), OpportunityTierGate (S06.02), AI Gateway client (E04/S11.03), and `client.ai_summaries` DB table (migration 018 from S06.05). All four are complete. This story wires them together with a new SSE endpoint.

### Critical: SSE vs HTTP Response Lifecycle

**The UsageGate check MUST happen before `StreamingResponse` is created.** Once `StreamingResponse` is returned to FastAPI, HTTP headers (200 + `Content-Type: text/event-stream`) are committed to the client. A 429 AppException raised inside the generator AFTER headers are sent cannot change the HTTP status code — the client would receive a 200 stream followed by an error event, which looks like a success.

**Correct pattern:**
```python
@router.post("/{opportunity_id}/ai-summary")
async def generate_ai_summary(
    ...
    usage_gate: Annotated[UsageGateContext, Depends(get_usage_gate)],
    redis: Annotated[Redis, Depends(get_redis_client)],
    response: Response,
    ...
) -> StreamingResponse:
    # 1. Tier check (raises 403 AppException → JSON before headers sent)
    if tier_gate.user_tier not in _PAID_TIERS:
        raise AppException(..., status_code=403)

    # 2. Usage gate (raises 429 AppException → JSON before headers sent)
    await usage_gate.check_and_increment("ai_summary", redis, response)

    # 3. Only NOW create StreamingResponse (headers committed after return)
    return StreamingResponse(
        _generate_summary_stream(...),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},
    )
```

The `X-Usage-Remaining` header is set by `check_and_increment()` on the FastAPI `response` object. Since we return a `StreamingResponse`, the header may not merge automatically — check whether FastAPI propagates `response.headers` onto `StreamingResponse` or whether the header must be passed explicitly in `StreamingResponse(headers={...})`. Prefer explicitly forwarding the header from `response.headers` into the `StreamingResponse`.

### DB Session in SSE Generator

FastAPI's `get_db_session` dependency creates a session scoped to the HTTP request. The `_generate_summary_stream` generator is called by Starlette *after* the route handler returns — by that time the dependency teardown may have begun. **Do NOT pass the request-scoped session into the generator.**

Instead, import the session factory from `dependencies.py` and create a fresh session inside the generator:

```python
from client_api.dependencies import _session_factory  # module-level factory

async def _generate_summary_stream(...) -> AsyncGenerator[bytes, None]:
    ...
    async with _session_factory() as db:
        async with db.begin():
            summary_id = await opportunity_service.persist_ai_summary(
                db, opportunity_id=opportunity_id, ...
            )
```

Alternatively, expose a `get_session_factory()` helper from `dependencies.py` and import that. Follow the pattern used by `_seed_document` in `test_document_download.py` — the same "session factory" pattern is established.

### `AiGatewayClient.stream_agent()` Implementation

The existing `run_agent()` uses a fresh `httpx.AsyncClient` per call. `stream_agent()` must similarly create a fresh client but use the `stream()` context manager. The AI Gateway SSE endpoint is `POST /agents/{agent_name}/run-stream`.

```python
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager

@asynccontextmanager
async def stream_agent(
    self, agent_name: str, payload: dict
) -> AsyncGenerator[str, None]:
    """Stream SSE events from AI Gateway agent execution.

    Yields raw SSE line strings (e.g., 'event: delta', 'data: {"text": "..."}', '').
    Heartbeat lines ('event: heartbeat') should be skipped by the caller.
    """
    url = f"{self._base_url}/agents/{agent_name}/run-stream"
    stream_timeout = httpx.Timeout(connect=60.0, read=None, write=30.0, pool=10.0)
    try:
        async with httpx.AsyncClient(timeout=stream_timeout) as client:
            async with client.stream(
                "POST", url, json=payload,
                headers={"X-Caller-Service": "client-api"},
            ) as resp:
                if resp.status_code >= 400:
                    raise AiGatewayUnavailableError(resp.status_code, "")
                async for line in resp.aiter_lines():
                    yield line
    except httpx.TimeoutException as exc:
        raise AiGatewayTimeoutError(f"Timeout streaming agent {agent_name}") from exc
    except (httpx.ConnectError, httpx.RemoteProtocolError) as exc:
        raise AiGatewayUnavailableError(0, str(exc)) from exc
```

**Note:** Use `@asynccontextmanager` so the caller can `async with gw_client.stream_agent(...) as lines:` and iterate `lines`. This keeps the `httpx` connection alive for the full duration.

### SSE Event Parsing in Generator

The AI Gateway yields raw SSE bytes. Each SSE event consists of lines like:
```
event: delta
data: {"text": "The tender..."}

event: metadata
data: {"model": "gpt-4o", "tokens_used": 1234}

event: done
data: {}
```

Parse the `asynccontextmanager` generator line by line:

```python
event_type = None
delta_chunks: list[str] = []
metadata: dict = {}

async with gw_client.stream_agent("executive-summary", payload) as lines:
    async for line in lines:
        if line.startswith("event:"):
            event_type = line[len("event:"):].strip()
        elif line.startswith("data:"):
            data_str = line[len("data:"):].strip()
            data = json.loads(data_str) if data_str else {}
            if event_type == "delta":
                delta_chunks.append(data.get("text", ""))
                yield f"event: delta\ndata: {data_str}\n\n".encode()
            elif event_type == "metadata":
                metadata = data
                yield f"event: metadata\ndata: {data_str}\n\n".encode()
            elif event_type == "done":
                # Persist to DB
                content = "".join(delta_chunks)
                summary_id = await persist_ai_summary(...)
                done_data = json.dumps({"summary_id": str(summary_id)})
                yield f"event: done\ndata: {done_data}\n\n".encode()
            elif event_type == "error":
                yield f"event: error\ndata: {data_str}\n\n".encode()
                return
            elif event_type == "heartbeat":
                pass  # skip
        elif line == "":
            event_type = None  # reset for next event
```

### `build_executive_summary_payload` Payload Structure

The agent payload should mirror what E04 expects for the Executive Summary Agent. Based on `agents.yaml` (`executive-summary` → "Generates executive summary from tender documents"), include:

```python
{
    "opportunity_id": str(opportunity_id),
    "title": row["title"],
    "description": row["description"] or "",
    "contracting_authority": row["contracting_authority"] or "",
    "budget_min": str(row["budget_min"]) if row["budget_min"] else None,
    "budget_max": str(row["budget_max"]) if row["budget_max"] else None,
    "currency": row["currency"] or "EUR",
    "deadline": row["deadline"].isoformat() if row["deadline"] else None,
    "cpv_codes": row["cpv_codes"] or [],
    "region": row["region"] or "",
    "opportunity_type": row["opportunity_type"],
    "evaluation_criteria": row["evaluation_criteria"] or {},
    "mandatory_documents": row["mandatory_documents"] or {},
}
```

Return 404 if `opportunity_id` is not found in `pipeline.opportunities`.

### Cache Logic (GET Endpoint)

```python
# Query most-recent summary for (opportunity_id, user_id)
stmt = (
    select(ai_t)
    .where(ai_t.c.opportunity_id == opportunity_id)
    .where(ai_t.c.user_id == user_id)
    .order_by(ai_t.c.generated_at.desc())
    .limit(1)
)
row = (await client_session.execute(stmt)).mappings().one_or_none()

if row is None:
    return AISummaryGetResponse(cached=False)

# 24h TTL check
age = datetime.now(UTC) - row["generated_at"]
if age > timedelta(hours=24):
    return AISummaryGetResponse(cached=False)

# Staleness check: fetch opportunity updated_at from pipeline
opp_stmt = select(opp_t.c.updated_at).where(opp_t.c.id == opportunity_id)
opp_row = (await pipeline_session.execute(opp_stmt)).mappings().one_or_none()
if opp_row and opp_row["updated_at"] and opp_row["updated_at"] > row["generated_at"]:
    return AISummaryGetResponse(cached=False, stale=True, reason="opportunity_updated")

return AISummaryGetResponse(
    cached=True,
    summary_id=row["id"],
    content=row["content"],
    model=row["model"],
    tokens_used=row["tokens_used"],
    generated_at=row["generated_at"],
)
```

### `client.ai_summaries` Table

Migration `018` (already applied from S06.05) created the table with:
- `id` UUID PK (gen_random_uuid())
- `opportunity_id` UUID (no FK — cross-schema FK avoided)
- `user_id` UUID FK → `client.users.id` ON DELETE CASCADE
- `company_id` UUID FK → `client.companies.id` ON DELETE CASCADE
- `content` TEXT NOT NULL
- `model` VARCHAR(100)
- `tokens_used` INTEGER
- `generated_at` TIMESTAMPTZ NOT NULL DEFAULT NOW()
- `created_at` TIMESTAMPTZ NOT NULL DEFAULT NOW()
- Index: `(opportunity_id, user_id)` and `(company_id)`

**No new migration required.** The model is at `client_api/models/client_ai_summary.py` (import as `ai_summaries_table`).

### Router Mount / URL Structure

The `opportunities.py` router has prefix `/opportunities`. The AI summary routes are:
- `POST /opportunities/{opportunity_id}/ai-summary`
- `GET /opportunities/{opportunity_id}/ai-summary`

These sit alongside the detail endpoint `GET /opportunities/{opportunity_id}`. FastAPI resolves these by path suffix + method. Ensure the `ai-summary` routes are defined **before** the `{opportunity_id}` wildcard GET route in the router file, or note that FastAPI actually matches by full path so ordering here should be fine — `/opportunities/{opportunity_id}/ai-summary` is more specific than `/opportunities/{opportunity_id}` and FastAPI routes will match correctly regardless of declaration order.

### Concurrency Cap (E06-R-004 Mitigation)

The test design flags SSE connection exhaustion as a high-risk scenario (score 6). For Sprint 6, add a configurable env var `MAX_CONCURRENT_SSE_STREAMS` (default 10) and implement a semaphore:

```python
# In config.py
max_concurrent_sse_streams: int = Field(default=10, env="MAX_CONCURRENT_SSE_STREAMS")

# In opportunities.py or a module-level singleton
_sse_semaphore: asyncio.Semaphore | None = None

def get_sse_semaphore() -> asyncio.Semaphore:
    global _sse_semaphore
    if _sse_semaphore is None:
        _sse_semaphore = asyncio.Semaphore(get_settings().max_concurrent_sse_streams)
    return _sse_semaphore
```

Acquire the semaphore before creating `StreamingResponse`; release it in the generator's `finally` block. If semaphore cannot be acquired immediately (or within a short timeout), return 503 with `Retry-After: 5`.

### Imports and File Locations

| File | Action |
|------|--------|
| `src/client_api/services/ai_gateway_client.py` | Add `stream_agent()` async context manager |
| `src/client_api/schemas/opportunities.py` | Append `AISummaryGetResponse` Pydantic model |
| `src/client_api/services/opportunity_service.py` | Add `get_cached_ai_summary`, `persist_ai_summary`, `build_executive_summary_payload` |
| `src/client_api/api/v1/opportunities.py` | Add `POST /{opportunity_id}/ai-summary` + `GET /{opportunity_id}/ai-summary` routes |
| `src/client_api/config.py` | Add `max_concurrent_sse_streams: int = Field(default=10)` |
| `tests/api/test_ai_summary.py` | Create new test file |

**Do NOT** create new model files, new service files, or new Alembic migrations — all infrastructure reused from S06.03/S06.05.

### Existing Dependencies to Inject

From `usage_gate.py`:
- `get_usage_gate` → `UsageGateContext` (inject as `Annotated[UsageGateContext, Depends(get_usage_gate)]`)
- `FEATURE_TIER_LIMITS["ai_summary"]` is already defined

From `dependencies.py`:
- `get_redis_client` → `Redis`
- `get_db_session` → `AsyncSession` (client schema writes)
- `get_pipeline_readonly_session` → `AsyncSession` (pipeline schema reads)

From `ai_gateway_client.py`:
- `get_ai_gateway_client` → `AiGatewayClient`

From `opportunity_tier_gate.py`:
- `get_opportunity_tier_gate` → `OpportunityTierGateContext`
- `_PAID_TIERS` frozenset already in `opportunities.py` (reuse)

### Test Patterns

**Mock SSE stream from AI Gateway:**
```python
from contextlib import asynccontextmanager
from unittest.mock import patch, MagicMock

@asynccontextmanager
async def mock_sse_stream():
    async def _lines():
        yield "event: delta"
        yield 'data: {"text": "Executive summary: "}'
        yield ""
        yield "event: delta"
        yield 'data: {"text": "this is a test tender."}'
        yield ""
        yield "event: metadata"
        yield 'data: {"model": "gpt-4o", "tokens_used": 100}'
        yield ""
        yield "event: done"
        yield 'data: {}'
        yield ""
    yield _lines()

with patch.object(gw_client, "stream_agent", return_value=mock_sse_stream()):
    ...
```

**Parse SSE response:**
```python
async with httpx.AsyncClient(app=ai_summary_app, base_url="http://test") as ac:
    async with ac.stream("POST", f"/api/v1/opportunities/{opp_id}/ai-summary",
                         headers={"Authorization": f"Bearer {token}"}) as r:
        events = []
        async for line in r.aiter_lines():
            if line.startswith("event:"):
                current_event = line.split(":", 1)[1].strip()
            elif line.startswith("data:"):
                data = json.loads(line.split(":", 1)[1].strip() or "{}")
                events.append({"event": current_event, "data": data})

# Assert order: delta(s) → metadata → done
delta_events = [e for e in events if e["event"] == "delta"]
assert len(delta_events) >= 1
metadata_events = [e for e in events if e["event"] == "metadata"]
assert len(metadata_events) == 1
done_events = [e for e in events if e["event"] == "done"]
assert len(done_events) == 1
assert "summary_id" in done_events[0]["data"]
```

**fakeredis for UsageGate tests:**
```python
import fakeredis.aioredis as fakeredis

@pytest_asyncio.fixture
async def redis_client():
    client = fakeredis.FakeRedis()
    yield client
    await client.aclose()
```

**E06-P0-004 (concurrency race) — testcontainers Redis required.** fakeredis alone cannot reproduce the race. Use `asyncio.gather` with two concurrent POST requests at limit-1 and assert exactly one succeeds. This test is complex and may be deferred to the TEA automate phase, but the story should note the requirement.

### Test Fixtures — FK Constraint Lesson from S06.07

**Critical learning from Story 6.7:** Hardcoded fake UUIDs in test fixtures violate FK constraints in `client.ai_summaries` (→ `client.users`, → `client.companies`). The `user_id` and `company_id` used to seed or verify `ai_summaries` rows **must match real seeded rows** in the test DB.

Use the same seeded test DB values established by previous stories:
- Use the conftest-level `test_user` and `test_company` fixtures for `user_id` and `company_id` in any DB-seeded rows.
- For JWT-only actors (checking 403, 401), fake UUIDs are acceptable since they never hit the DB.
- Follow the `_make_jwt` pattern from `test_document_download.py` which uses `jwt.encode()` directly (not `create_access_token`) so arbitrary `subscription_tier` claims can be embedded.

### Known Deviation Pattern from S06.07

S06.07 required adding `"detail"` as an alias key in `AppException` handler because ATDD tests assert `body["detail"]` but the handler returned `body["message"]`. **Verify this fix is present** in `eusolicit_common/exceptions.py` before writing any assertions for `response.json()["detail"]`.

### Test Design Coverage Matrix

| Test ID | Scenario | Story | Level | Implementation Hint |
|---------|----------|-------|-------|---------------------|
| E06-P0-004 | UsageGate atomic counter race | S06.03, S06.08 | Integration | testcontainers Redis + asyncio.gather |
| E06-P0-005 | 429 with correct body + header | S06.03 | API | fakeredis at limit |
| E06-P0-008 | Cached summary returned, counter unchanged | S06.08 | Integration | Seed ai_summaries, call GET + verify Redis unchanged |
| E06-P0-009 | SSE event order: delta→metadata→done | S06.08 | Integration | Mock gw_client.stream_agent |
| E06-P1-021 | GET returns cached=true without AI Gateway call | S06.08 | API | respx assert zero outbound calls |
| E06-P1-022 | Error event streamed on AI Gateway failure | S06.08 | Integration | mock_sse_stream yields error event |

### Risk Mitigations Embedded in Implementation

- **E06-R-002** (Redis race): Mitigated by existing Lua script in `usage_gate.py`. No further action needed in this story.
- **E06-R-004** (SSE exhaustion): Add `MAX_CONCURRENT_SSE_STREAMS` semaphore (AC implied, see concurrency cap section above).
- **E06-R-007** (stale cache): Implemented in `get_cached_ai_summary()` by comparing `ai_summaries.generated_at` vs `pipeline.opportunities.updated_at`.

### Project Structure Notes

- All new Python files follow `from __future__ import annotations` at top.
- All imports use the `client_api.*` package path.
- Test files go in `eusolicit-app/services/client-api/tests/api/` (API-level tests with mocked external dependencies).
- The `_session_factory` used in the SSE generator is the private singleton from `dependencies.py` — either expose it via a getter function or import it directly (check how conftest does it).
- `ai_gateway_client.py` currently uses `functools.lru_cache` singleton. The new `stream_agent()` is a method on `AiGatewayClient` — no singleton change needed.

### References

- Story 6.7 (predecessor): `eusolicit-docs/implementation-artifacts/6-7-document-download-api.md`
- Story 6.6 (upload): `eusolicit-docs/implementation-artifacts/6-6-document-upload-api-s3-clamav.md` (S3 pattern, session factory pattern)
- Epic AC: `eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.08`
- Test design: `eusolicit-docs/test-artifacts/test-design-epic-06.md` (E06-P0-004/005/008/009, E06-P1-021/022, E06-P2-007, E06-R-002/004/007)
- Usage gate: `src/client_api/core/usage_gate.py` (UsageGateContext, get_usage_gate, FEATURE_TIER_LIMITS)
- Opportunity tier gate: `src/client_api/core/opportunity_tier_gate.py` (OpportunityTierGateContext, get_opportunity_tier_gate)
- AI Gateway client: `src/client_api/services/ai_gateway_client.py` (AiGatewayClient, AiGatewayTimeoutError, AiGatewayUnavailableError, get_ai_gateway_client)
- AI summaries model: `src/client_api/models/client_ai_summary.py` (ai_summaries_table)
- AI summaries migration: `alembic/versions/018_ai_summaries.py`
- Opportunities router: `src/client_api/api/v1/opportunities.py`
- Opportunity service: `src/client_api/services/opportunity_service.py`
- Opportunity schemas: `src/client_api/schemas/opportunities.py`
- Config: `src/client_api/config.py`
- Dependencies: `src/client_api/dependencies.py`
- Agent registry (agent name): `eusolicit-app/services/ai-gateway/config/agents.yaml` → `executive-summary`
- AI Gateway SSE endpoint: `eusolicit-app/services/ai-gateway/src/ai_gateway/routers/execution.py` → `POST /agents/{id}/run-stream`

## Senior Developer Review

**Date:** 2026-04-17
**Verdict:** Changes Requested
**Acceptance Criteria:** All 10 ACs PASS
**Review Layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor
**Findings:** 4 patch, 3 defer, 6 dismissed

### Review Findings

- [x] [Review][Patch] `asyncio.shield(semaphore.acquire())` permanently leaks semaphore permits on timeout [`opportunities.py:651`] — When `wait_for` times out, `asyncio.shield` prevents cancellation of the inner `acquire()` coroutine. The shielded acquire may complete in the background after the timeout, decrementing the semaphore counter with no corresponding `release()`. Each occurrence permanently reduces available SSE concurrency by 1. Under sustained load with contention spikes, all semaphore slots leak and every subsequent POST returns 503. **Fix:** Remove `asyncio.shield` — use `await asyncio.wait_for(semaphore.acquire(), timeout=0.05)` directly. CPython's `asyncio.Semaphore.acquire()` is cancel-safe (waiter is removed from queue on cancellation).
- [x] [Review][Patch] Usage gate increments quota before opportunity existence check — 404 wastes quota [`opportunities.py:627-644`] — `check_and_increment` atomically increments the Redis counter (line 627) BEFORE `build_executive_summary_payload` is called (line 644). If the opportunity doesn't exist (404), the quota is consumed but no summary is generated. A starter-tier user (limit 10) can have their entire monthly quota burned by sending 10 requests with invalid UUIDs. **Fix:** Move `build_executive_summary_payload(opportunity_id, pipeline_session)` before the `check_and_increment` call. The AC3 constraint is that usage gate must execute before `StreamingResponse`, not before the payload fetch.
- [x] [Review][Patch] `stream_agent` asynccontextmanager doesn't close inner generator [`ai_gateway_client.py:97-120`] — The `@asynccontextmanager` yields `_line_gen()` but doesn't call `aclose()` on it when the context exits. If the consumer stops iteration (error, return, or cancellation), the inner generator's httpx resources (`read=None` = infinite timeout) may not be cleaned up deterministically. **Fix:** Store the generator and close it explicitly: `gen = _line_gen(); try: yield gen; finally: await gen.aclose()`
- [x] [Review][Patch] No terminal SSE event when AI Gateway stream ends without "done" event [`opportunities.py:500-546`] — If the AI Gateway stream connection closes without sending a "done" or "error" event, the `async for` loop exits silently, the semaphore is released, but the client receives no terminal event. The client is left with partial delta chunks and no indication the stream ended. **Fix:** After the `async for` loop, check whether a terminal event was emitted. If not, yield `event: error\ndata: {"error": "AI Gateway stream ended unexpectedly"}\n\n`.
- [x] [Review][Defer] Usage quota consumed on AI Gateway failure or client disconnect — deferred, design tradeoff inherent to check-before-stream architecture (AC3 requirement). A future enhancement could implement quota refund on AI Gateway 5xx errors.
- [x] [Review][Defer] DB persistence failure after delta chunks already streamed — deferred, requires saga/retry pattern beyond current story scope. Current broad except handler sends error event, which is the best behavior without a compensating transaction.
- [x] [Review][Defer] Clock skew can cause negative cache age treated as permanently fresh — deferred, pre-existing infrastructure concern requiring NTP consistency. Guard `age < timedelta(0)` could be added defensively.

## Dev Agent Record

### Agent Model Used

Claude Opus 4.6

### Debug Log References

- fakeredis 2.35.1 does not support redis.eval() (Lua scripts). Tests requiring Lua-based usage gate (429 and X-Usage-Remaining header tests) were updated to use real test Redis (conftest test_redis_client on DB index 1) instead of fakeredis.
- Reordering build_executive_summary_payload before check_and_increment (Patch 2) required adding pipeline session mocks to the 429 test and removing fakeredis overrides from the X-Usage-Remaining test.

### Completion Notes List

- ✅ Resolved review finding [Patch]: Removed `asyncio.shield()` wrapping `semaphore.acquire()` — CPython's asyncio.Semaphore.acquire() is cancel-safe; the shield was causing permanent permit leaks on timeout.
- ✅ Resolved review finding [Patch]: Moved `build_executive_summary_payload()` before `check_and_increment()` — opportunity existence (404) is now validated before quota is consumed, protecting starter-tier users from wasting quota on invalid UUIDs.
- ✅ Resolved review finding [Patch]: Added explicit `await gen.aclose()` in `stream_agent()` finally block — ensures httpx resources are deterministically released when the consumer stops iteration.
- ✅ Resolved review finding [Patch]: Added terminal error event when AI Gateway stream ends without "done" or "error" — the `_generate_summary_stream` generator now tracks `terminal_event_sent` and emits `event: error` with "AI Gateway stream ended unexpectedly" if the `async for` loop exits without a terminal event.
- ✅ Updated test fixtures: `test_post_ai_summary_usage_limit_exceeded_returns_429` and `test_post_ai_summary_sets_x_usage_remaining_header` switched from fakeredis (which doesn't support Lua eval) to real test Redis with proper cleanup.
- All 15 ATDD tests pass. 653 API-level tests pass with 0 regressions.

### Change Log

- 2026-04-17: Addressed 4 code review patch findings (semaphore leak, quota-before-404, generator cleanup, terminal SSE event). Updated 2 test fixtures for Redis eval compatibility.

### File List

- `src/client_api/api/v1/opportunities.py` — (modified) Removed asyncio.shield from semaphore acquire; reordered payload fetch before usage gate; added terminal_event_sent tracking with fallback error event in SSE generator
- `src/client_api/services/ai_gateway_client.py` — (modified) Added explicit gen.aclose() in stream_agent finally block
- `tests/api/test_ai_summary.py` — (modified) Updated 429 and X-Usage-Remaining tests to use real Redis instead of fakeredis; added pipeline session mock to 429 test
