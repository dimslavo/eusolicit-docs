# E04: AI Gateway Service

**Sprint**: 3--4 | **Points**: 34 | **Dependencies**: E01 | **Milestone**: Demo

## Goal

Build an internal-only FastAPI service that acts as the single integration point between EU Solicit backend services and the KraftData Agentic AI platform (stage.sirma.ai). The AI Gateway wraps all KraftData Client API v1 endpoints behind a simplified internal API, provides circuit-breaker and retry resilience, proxies SSE streams, receives and routes KraftData webhooks, and logs every execution for observability. Deployed as a ClusterIP service -- never internet-facing -- consumed exclusively by Client API, Admin API, and Data Pipeline.

## KraftData Client API v1 Endpoints Consumed

| Internal Route | KraftData Route |
|---|---|
| `POST /agents/{id}/run` | `POST /client/api/v1/agents/{agentId}/run` |
| `POST /agents/{id}/run-stream` | `POST /client/api/v1/agents/{agentId}/run-stream` |
| `POST /workflows/{id}/run` | `POST /client/api/v1/workflows/{workflowId}/run` |
| `POST /workflows/{id}/run-stream` | `POST /client/api/v1/workflows/{workflowId}/run-stream` |
| `POST /teams/{id}/run` | `POST /client/api/v1/teams/{teamId}/run` |
| `POST /storage/{id}/files` | `POST /client/api/v1/storage-resources/{id}/files` |
| `POST /webhooks/kraftdata` | Inbound receiver -- not a KraftData call |
| -- | `POST /client/api/v1/eval-runs` (quality evaluation) |
| -- | `PUT /api/webhooks/subscriptions/{id}` (webhook management) |

## Acceptance Criteria

- [ ] FastAPI service starts, passes `/health` and `/ready` probes, connects to PostgreSQL and Redis
- [ ] All 7 internal API routes proxy correctly to their KraftData counterparts and return expected responses
- [ ] Agent registry loads 29 entries (27 agents, 1 team, 1 workflow) from YAML config; unknown logical names return 404
- [ ] httpx async client authenticates to KraftData using API key from environment/secrets
- [ ] Circuit breaker opens after 5 consecutive failures per agent, rejects calls with 503 during 30s cooldown, re-closes on success
- [ ] Retry logic applies exponential backoff (1s, 2s, 4s) up to 3 retries for 5xx and timeout errors only
- [ ] SSE stream proxy forwards KraftData events to the calling service in real time; handles connection drops and partial responses gracefully
- [ ] Webhook receiver validates KraftData signature, routes events by type, publishes to Redis Streams (`agent.execution.completed`, `agent.execution.failed`)
- [ ] Every agent/team/workflow call is logged to `gateway.agent_executions` with execution_id, agent_name, caller_service, start_time, end_time, status, latency_ms, error_message
- [ ] Webhook events are logged to `gateway.webhook_log`
- [ ] Connection pool enforces configurable concurrency limit to KraftData (default: 10)
- [ ] Integration tests cover happy path, circuit-breaker tripping, retry exhaustion, SSE stream interruption, and webhook signature rejection
- [ ] Service runs as ClusterIP only -- no Ingress, no public exposure

---

## Stories

### S04.01: FastAPI Service Scaffold and Health Probes

**Points**: 2 | **Type**: backend

Bootstrap the `ai-gateway` FastAPI service inside the monorepo. Set up project structure, dependency management, configuration loading, and health endpoints.

**Tasks**:
- Create `services/ai-gateway/` with standard layout: `app/`, `app/routers/`, `app/models/`, `app/services/`, `app/config.py`, `tests/`
- Add `pyproject.toml` with dependencies: `fastapi`, `uvicorn`, `httpx`, `pydantic`, `pydantic-settings`, `sqlalchemy`, `asyncpg`, `redis[hiredis]`, `pyyaml`
- Implement `app/config.py` using pydantic-settings: `KRAFTDATA_BASE_URL`, `KRAFTDATA_API_KEY`, `DATABASE_URL`, `REDIS_URL`, `CONCURRENCY_LIMIT` (default 10), `CIRCUIT_BREAKER_THRESHOLD` (default 5), `CIRCUIT_BREAKER_COOLDOWN` (default 30)
- Implement `GET /health` -- returns `{"status": "ok"}` (liveness)
- Implement `GET /ready` -- checks PostgreSQL and Redis connectivity, returns 200 or 503
- Add Dockerfile and docker-compose service entry
- Wire up structured JSON logging from E01 shared library

**Acceptance**:
- `GET /health` returns 200 with `{"status": "ok"}`
- `GET /ready` returns 200 when DB and Redis are up, 503 when either is down
- Service starts in under 3 seconds locally
- Configuration loads from environment variables and `.env` file

---

### S04.02: httpx Async Client and KraftData Authentication

**Points**: 3 | **Type**: backend

Build the core HTTP client that all outbound KraftData calls flow through. Manages connection pooling, authentication, base URL routing, and request/response logging.

**Tasks**:
- Create `app/services/kraftdata_client.py` with an `httpx.AsyncClient` singleton (lifespan-managed)
- Configure connection pool: `max_connections` from `CONCURRENCY_LIMIT`, `max_keepalive_connections` = `CONCURRENCY_LIMIT // 2`, `keepalive_expiry` = 30s
- Inject `Authorization: Bearer {KRAFTDATA_API_KEY}` header on every request
- Set default timeout: 60s connect, 120s read (agent runs can be slow)
- Add `Content-Type: application/json` default header
- Implement `async def call_kraftdata(method, path, **kwargs) -> httpx.Response` as the single call point
- Log every outbound request (method, path, status, latency) at INFO level; log response bodies at DEBUG level
- Handle `httpx.TimeoutException`, `httpx.ConnectError`, `httpx.HTTPStatusError` with typed internal exceptions (`KraftDataTimeoutError`, `KraftDataConnectionError`, `KraftDataAPIError`)

**Acceptance**:
- Client authenticates successfully against KraftData stage environment
- Connection pool respects `CONCURRENCY_LIMIT` (verified with concurrent request test)
- Timeout errors raise `KraftDataTimeoutError` with original context
- 4xx responses raise `KraftDataAPIError` with status code and body
- Client is cleanly shut down on app shutdown (no resource leaks)

**Tests**:
- Unit: mock httpx transport, verify auth header sent, verify timeout handling, verify pool limits
- Integration: real call to KraftData `/health` or equivalent (skipped in CI if no credentials)

---

### S04.03: Agent Registry -- Logical Name to KraftData UUID Mapping

**Points**: 2 | **Type**: backend

Create a configuration-driven registry that maps logical agent names used by EU Solicit services to KraftData UUIDs. This decouples business logic from KraftData identifiers and makes agent swaps a config change.

**Tasks**:
- Create `app/services/agent_registry.py` with `AgentRegistry` class
- Define YAML schema in `config/agents.yaml`:
  ```yaml
  agents:
    executive-summary:
      kraftdata_id: "uuid-here"
      type: agent  # agent | team | workflow
      description: "Generates executive summary from tender documents"
      timeout_override: 180  # optional, seconds
    # ... 29 entries total
  ```
- Load and validate on startup using Pydantic models; fail fast on missing/duplicate entries
- Expose `registry.resolve(logical_name) -> AgentEntry` -- returns entry or raises `AgentNotFoundError`
- Expose `registry.list_agents() -> list[AgentEntry]` for admin introspection
- Support hot-reload via `SIGHUP` or admin endpoint `POST /admin/registry/reload`

**Acceptance**:
- Registry loads 29 entries from YAML without error
- `resolve("executive-summary")` returns correct UUID and type
- `resolve("nonexistent")` raises `AgentNotFoundError`
- Duplicate logical names in YAML cause startup failure with clear error message
- Reload endpoint updates registry without service restart

**Tests**:
- Unit: load valid YAML, load YAML with duplicates (expect error), resolve known name, resolve unknown name
- Unit: verify timeout_override falls back to default when not specified

---

### S04.04: Sync Agent, Workflow, and Team Execution Endpoints

**Points**: 5 | **Type**: backend

Implement the core synchronous execution proxy endpoints. Each endpoint resolves the logical name or UUID, calls the corresponding KraftData endpoint, and returns the response.

**Tasks**:
- Implement `POST /agents/{id}/run`:
  - Accept `id` as logical name or raw UUID
  - Resolve via agent registry (if logical name) or pass through (if UUID)
  - Forward request body to `POST /client/api/v1/agents/{agentId}/run`
  - Map KraftData response to internal schema; return 200 on success
  - Return 404 if agent not found in registry, 502 if KraftData returns error, 504 if timeout
- Implement `POST /workflows/{id}/run`:
  - Same pattern, routes to `/client/api/v1/workflows/{workflowId}/run`
  - Validate that resolved entry has `type: workflow`
- Implement `POST /teams/{id}/run`:
  - Same pattern, routes to `/client/api/v1/teams/{teamId}/run`
  - Validate that resolved entry has `type: team`
- Implement `POST /storage/{id}/files`:
  - Accept multipart file upload
  - Forward to `/client/api/v1/storage-resources/{id}/files`
  - Return KraftData response (file ID, status)
- Define Pydantic request/response models for each endpoint
- Add `X-Request-ID` header propagation (generate if not present)
- Add `X-Caller-Service` header requirement on all endpoints (used for execution logging)

**Acceptance**:
- `POST /agents/executive-summary/run` resolves to correct UUID and returns KraftData response
- `POST /agents/{raw-uuid}/run` works without registry lookup
- `POST /workflows/{id}/run` rejects if registry entry type is not `workflow`
- `POST /storage/{id}/files` successfully uploads a test PDF
- Missing `X-Caller-Service` header returns 400
- KraftData 500 response returns 502 to caller with error details
- KraftData timeout returns 504 to caller

**Tests**:
- Unit: mock KraftData client, verify correct endpoint routing for each type
- Unit: verify logical name resolution and UUID passthrough
- Unit: verify type mismatch rejection (e.g., calling workflow endpoint with agent type)
- Integration: end-to-end call with mocked KraftData responses via `respx`

---

### S04.05: SSE Stream Proxy Endpoints

**Points**: 5 | **Type**: backend

Implement streaming execution endpoints that proxy KraftData Server-Sent Events to the calling service in real time. This is critical for responsive UX during long-running agent executions.

**Tasks**:
- Implement `POST /agents/{id}/run-stream`:
  - Resolve agent as in S04.04
  - Open SSE connection to `POST /client/api/v1/agents/{agentId}/run-stream`
  - Return `StreamingResponse` with `text/event-stream` content type
  - Forward each SSE event as received: `data:`, `event:`, `id:` fields
  - On KraftData stream completion, close response cleanly
- Implement `POST /workflows/{id}/run-stream`:
  - Same pattern for workflow streaming
- Handle connection drops:
  - If KraftData drops the connection mid-stream, send a final `event: error` with `data: {"error": "upstream_disconnected"}` and close
  - If the calling service disconnects, cancel the KraftData request (avoid orphaned connections)
- Handle timeouts:
  - Stream idle timeout: 120s with no events triggers disconnect
  - Total stream timeout: 600s maximum duration
- Implement heartbeat: send `event: heartbeat` every 15s during idle periods to keep the connection alive and detect dead clients
- Buffer partial SSE frames: KraftData may send partial `data:` lines; reassemble before forwarding

**Acceptance**:
- Client receives SSE events in real time (latency < 100ms per event after KraftData sends)
- Stream completes cleanly on normal agent completion
- Client disconnect cancels upstream KraftData connection within 5s
- KraftData disconnect sends error event to client
- Idle stream times out after 120s with timeout event
- Heartbeat events arrive every ~15s during idle periods
- Partial SSE frames are correctly reassembled

**Tests**:
- Unit: mock SSE source, verify events forwarded in order
- Unit: simulate upstream disconnect, verify error event sent
- Unit: simulate client disconnect, verify upstream cancelled
- Unit: verify heartbeat timing with frozen clock
- Integration: end-to-end SSE proxy with mock KraftData SSE server

---

### S04.06: Circuit Breaker and Retry Logic

**Points**: 5 | **Type**: backend

Implement per-agent circuit breaker and retry policies to protect both EU Solicit and KraftData from cascading failures. Circuit breaker prevents hammering a failing agent; retries handle transient errors.

**Tasks**:
- Create `app/services/circuit_breaker.py`:
  - Per-agent circuit state: `CLOSED` (normal), `OPEN` (rejecting), `HALF_OPEN` (testing)
  - State transitions: CLOSED -> OPEN after `threshold` (default 5) consecutive failures; OPEN -> HALF_OPEN after `cooldown` (default 30s); HALF_OPEN -> CLOSED on success, HALF_OPEN -> OPEN on failure
  - Track state in memory (not Redis -- this is per-instance, which is acceptable for single-replica initial deployment)
  - When OPEN, reject with `CircuitOpenError` immediately (no KraftData call)
  - Expose circuit state via `GET /admin/circuits` for observability
- Create `app/services/retry.py`:
  - Retry on: HTTP 5xx, `httpx.TimeoutException`, `httpx.ConnectError`
  - Do NOT retry on: HTTP 4xx, `CircuitOpenError`, client disconnect
  - Backoff schedule: 1s, 2s, 4s (exponential, base=1, factor=2)
  - Max retries: 3 (4 total attempts)
  - Add jitter: +/- 25% randomization on each delay to prevent thundering herd
  - Log each retry attempt at WARN level with attempt number and error
- Integrate into `call_kraftdata()`: retry wraps the HTTP call, circuit breaker wraps the retry
- Emit structured log events on circuit state transitions (WARN for OPEN, INFO for CLOSED)
- Emit metrics-ready log fields: `circuit_state`, `retry_attempt`, `agent_name`

**Acceptance**:
- After 5 consecutive failures to agent X, the 6th call returns 503 immediately without contacting KraftData
- After 30s cooldown, the next call to agent X is allowed through (half-open)
- If half-open call succeeds, circuit closes and subsequent calls proceed normally
- If half-open call fails, circuit reopens for another 30s
- Transient 500 from KraftData triggers retry; succeeds on 2nd attempt with ~1s delay
- 3 retries exhausted logs final error and returns 502 to caller
- 400 from KraftData is NOT retried (returns immediately)
- `GET /admin/circuits` returns current state of all agents with failure counts and last failure time

**Tests**:
- Unit: verify state machine transitions (closed->open->half_open->closed, closed->open->half_open->open)
- Unit: verify threshold count resets on success
- Unit: verify cooldown timing with frozen clock
- Unit: verify retry backoff delays (1s, 2s, 4s) with jitter bounds
- Unit: verify non-retryable errors are not retried
- Unit: verify retry success on 2nd attempt resets circuit failure count

---

### S04.07: KraftData Webhook Receiver and Redis Stream Publishing

**Points**: 3 | **Type**: backend

Implement the inbound webhook endpoint that KraftData calls when asynchronous agent executions complete or fail. Validate signatures, route events, and publish to Redis Streams for downstream processing.

**Tasks**:
- Implement `POST /webhooks/kraftdata`:
  - Verify webhook signature from `X-Kraftdata-Signature` header using HMAC-SHA256 with configured webhook secret
  - Reject invalid signatures with 401 (log the attempt at WARN)
  - Parse webhook payload: extract `event_type`, `execution_id`, `agent_id`, `result`, `error`
  - Route by event type:
    - `execution.completed` -> publish to Redis Stream `agent.execution.completed`
    - `execution.failed` -> publish to Redis Stream `agent.execution.failed`
    - Unknown event types -> log at WARN, return 200 (don't break the webhook)
  - Return 200 immediately after publishing (KraftData expects fast response)
- Redis Stream message format:
  ```json
  {
    "execution_id": "...",
    "agent_id": "...",
    "agent_name": "logical-name-if-resolvable",
    "event_type": "...",
    "payload": "...(JSON string)...",
    "received_at": "ISO8601"
  }
  ```
- Log every webhook to `gateway.webhook_log` table (async, don't block response)
- Implement idempotency: deduplicate by `execution_id` using Redis SET with 1h TTL

**Acceptance**:
- Valid webhook with correct signature returns 200 and message appears in Redis Stream
- Invalid signature returns 401, nothing published to Redis
- Duplicate `execution_id` within 1 hour is acknowledged (200) but not re-published
- Unknown event type returns 200 and logs warning
- Webhook processing completes in < 50ms (measured from request to response)
- `gateway.webhook_log` contains entry for every received webhook (valid or not)

**Tests**:
- Unit: verify HMAC signature validation (valid, invalid, missing header)
- Unit: verify correct Redis Stream routing by event type
- Unit: verify idempotency deduplication
- Integration: publish webhook, verify Redis Stream message with correct format

---

### S04.08: Execution Logging and Database Schema

**Points**: 3 | **Type**: backend

Create the gateway database schema and implement execution logging middleware that records every agent/team/workflow call for observability, debugging, and SLA tracking.

**Tasks**:
- Create Alembic migration for `gateway` schema:
  ```sql
  CREATE TABLE gateway.agent_executions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id VARCHAR(255) UNIQUE NOT NULL,
    agent_name VARCHAR(100) NOT NULL,
    agent_kraftdata_id UUID NOT NULL,
    agent_type VARCHAR(20) NOT NULL,  -- agent, team, workflow
    caller_service VARCHAR(50) NOT NULL,
    request_id VARCHAR(255),
    is_streaming BOOLEAN DEFAULT FALSE,
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL,  -- pending, success, failed, timeout, circuit_open
    latency_ms INTEGER,
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
  );

  CREATE INDEX idx_agent_executions_agent_name ON gateway.agent_executions(agent_name);
  CREATE INDEX idx_agent_executions_status ON gateway.agent_executions(status);
  CREATE INDEX idx_agent_executions_start_time ON gateway.agent_executions(start_time);
  CREATE INDEX idx_agent_executions_caller ON gateway.agent_executions(caller_service);

  CREATE TABLE gateway.webhook_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id VARCHAR(255),
    event_type VARCHAR(100) NOT NULL,
    signature_valid BOOLEAN NOT NULL,
    payload JSONB,
    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed BOOLEAN DEFAULT FALSE
  );

  CREATE INDEX idx_webhook_log_execution_id ON gateway.webhook_log(execution_id);
  CREATE INDEX idx_webhook_log_event_type ON gateway.webhook_log(event_type);
  ```
- Create SQLAlchemy models for both tables
- Implement execution logging as FastAPI middleware or dependency:
  - Before call: insert row with `status=pending`, record `start_time`
  - After call: update with `end_time`, `status`, `latency_ms`, `error_message`, `retry_count`
  - Use async database writes -- never block the response on logging
- Implement `GET /admin/executions` with filters: `agent_name`, `status`, `caller_service`, `start_time` range, pagination (limit/offset)
- Add partition-readiness comment: table will be partitioned by `start_time` monthly when volume warrants it

**Acceptance**:
- Migration creates both tables with all indexes
- Every sync execution creates exactly one `agent_executions` row with accurate timing
- Every streaming execution creates a row; `end_time` is set when stream completes
- Circuit-open rejections are logged with `status=circuit_open` and `latency_ms=0`
- `GET /admin/executions?agent_name=executive-summary&status=failed` returns filtered results
- Logging failure does not cause the proxy call to fail (fire-and-forget with error log)

**Tests**:
- Unit: verify execution row created on call start, updated on completion
- Unit: verify circuit-open calls are logged correctly
- Unit: verify logging failure is swallowed (mock DB error, call still succeeds)
- Integration: make proxy call, query `agent_executions` table, verify all fields populated

---

### S04.09: Rate Limit Management and Concurrency Control

**Points**: 3 | **Type**: backend

Implement concurrency control to prevent overwhelming KraftData with too many simultaneous requests. Uses a semaphore-based approach with configurable limits and queue behavior.

**Tasks**:
- Create `app/services/rate_limiter.py`:
  - `asyncio.Semaphore` with `CONCURRENCY_LIMIT` permits (default 10)
  - When semaphore is full, requests queue (up to `QUEUE_TIMEOUT` = 30s), then return 429
  - Track active/queued counts for observability
- Integrate into `call_kraftdata()` pipeline: rate limit -> circuit breaker -> retry -> HTTP call
- Expose `GET /admin/rate-limit` returning:
  ```json
  {
    "concurrency_limit": 10,
    "active_requests": 7,
    "queued_requests": 2,
    "total_rejected": 15
  }
  ```
- Implement per-agent soft limits (optional config in agents.yaml):
  ```yaml
  agents:
    executive-summary:
      kraftdata_id: "..."
      max_concurrent: 3  # optional, defaults to no per-agent limit
  ```
- Log at WARN when queue depth exceeds 50% of concurrency limit
- Log at ERROR and emit alert-ready event when requests are rejected (429)

**Acceptance**:
- With `CONCURRENCY_LIMIT=2`, 3 simultaneous calls: 2 proceed, 1 queues and proceeds when a slot opens
- With `CONCURRENCY_LIMIT=2`, 3 simultaneous calls with 30s+ execution: 3rd returns 429 after queue timeout
- Per-agent limit of 3 prevents 4th concurrent call to same agent even if global limit has capacity
- `GET /admin/rate-limit` returns accurate active and queued counts in real time
- Streaming requests hold their semaphore permit for the duration of the stream
- Rate limit rejection is logged to `agent_executions` with `status=rate_limited`

**Tests**:
- Unit: verify semaphore blocks at limit, releases on completion
- Unit: verify queue timeout triggers 429
- Unit: verify per-agent limit enforcement
- Unit: verify streaming holds permit until stream ends
- Integration: concurrent load test with concurrency limit of 2, verify correct queuing behavior

---

### S04.10: Integration Tests and End-to-End Validation

**Points**: 3 | **Type**: backend

Comprehensive integration test suite that validates the full AI Gateway request lifecycle, including failure scenarios. Uses `respx` for KraftData mocking and `testcontainers` for PostgreSQL and Redis.

**Tasks**:
- Set up test infrastructure:
  - `conftest.py` with testcontainers for PostgreSQL and Redis
  - `respx` mock router for KraftData API
  - Factory fixtures for agent registry, test requests, and webhook payloads
- Test scenarios:
  1. **Happy path sync**: call agent by logical name -> verify KraftData receives correct UUID -> verify response -> verify execution log row
  2. **Happy path stream**: call agent stream -> verify SSE events forwarded -> verify execution log with streaming flag
  3. **Circuit breaker trip**: 5 consecutive failures -> verify 6th call returns 503 without KraftData contact -> wait cooldown -> verify recovery
  4. **Retry success**: KraftData returns 500 once then 200 -> verify retry happened -> verify execution log shows retry_count=1
  5. **Retry exhaustion**: KraftData returns 500 four times -> verify 502 returned -> verify execution log shows retry_count=3
  6. **Webhook valid**: POST webhook with valid signature -> verify Redis Stream message -> verify webhook_log entry
  7. **Webhook invalid signature**: POST webhook with bad signature -> verify 401 -> verify webhook_log with `signature_valid=false`
  8. **Rate limit**: flood with concurrent requests exceeding limit -> verify queuing and 429
  9. **Unknown agent**: call with nonexistent logical name -> verify 404
  10. **Type mismatch**: call workflow endpoint with agent-type entry -> verify 400
- Generate test coverage report; target 85%+ line coverage on `app/services/` and `app/routers/`

**Acceptance**:
- All 10 test scenarios pass in CI
- Tests complete in under 60 seconds
- No flaky tests (run 3x in CI to verify)
- Coverage report generated and exceeds 85% on core modules
- Tests use no real KraftData credentials (fully mocked)

**Tests**:
- All tests listed above, plus edge cases discovered during implementation
