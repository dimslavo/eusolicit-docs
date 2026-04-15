---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-14'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 4
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-04-ai-gateway-service.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/test-design-progress.md'
---

# Test Design: Epic 4 — AI Gateway Service

**Date:** 2026-04-14
**Author:** TEA Master Test Architect
**Status:** Draft
**Epic:** E04 | **Sprint:** 3–4 | **Points:** 34 | **Dependency:** E01

---

## Executive Summary

**Scope:** Epic-level test design for E04 — AI Gateway Service. Covers the full lifecycle of the internal-only FastAPI service that proxies EU Solicit backend requests to the KraftData Agentic AI platform (stage.sirma.ai). Functional scope includes: service scaffold and health probes (S04.01), httpx async client and authentication (S04.02), agent/team/workflow registry (S04.03), synchronous execution endpoints (S04.04), SSE stream proxy (S04.05), circuit breaker and retry logic (S04.06), inbound webhook receiver and Redis Stream publishing (S04.07), execution audit logging and DB schema (S04.08), concurrency and rate limiting (S04.09), and integration test suite (S04.10).

This epic is a **critical platform dependency**: every AI-assisted feature (ESPD auto-fill, grant eligibility, budget builder, consortium finder, logframe generator) routes through this gateway. Failures here silently degrade or block the entire AI-assisted proposal workflow for all tenants.

**Risk Summary:**

- Total risks identified: 9
- High-priority risks (≥6): 3 (webhook signature bypass, SSE proxy reliability, circuit breaker distributed state gap)
- Critical categories: SEC (webhook bypass), TECH (SSE edge cases, circuit state), DATA (Redis stream loss), PERF (semaphore starvation)

**Coverage Summary:**

- P0 scenarios: 13 (~25–40 hours)
- P1 scenarios: 20 (~20–30 hours)
- P2 scenarios: 17 (~12–20 hours)
- P3 scenarios: 5 (~4–8 hours)
- **Total effort:** ~61–98 hours (~2–2.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **Direct testing of KraftData stage environment** | External dependency; credentials-gated; not controlled by EU Solicit | Use `respx` mock router for all outbound KraftData calls in CI; real-credential tests tagged `@skip-ci` |
| **Redis Streams consumer validation** | Downstream epics (E05+) own consumer logic; E04 only owns the publish side | Verify `XADD` was called with correct stream name and payload; consumer behavior is out-of-scope |
| **Kubernetes ClusterIP networking / Ingress** | Infrastructure concern owned by DevOps; service is ClusterIP-only per spec | Confirm no Ingress resource created (manifest check); runtime routing is platform-level |
| **KraftData eval-runs and webhook subscription management** | Not in E04 internal routes; future epic scope | Document as known gap; add to backlog |
| **KraftData response payload content validation** | EU Solicit is a proxy; KraftData is responsible for response correctness | Verify status codes and structural shape only; not semantic content |
| **PostgreSQL performance under high load** | Database performance is an E01/platform NFR; E08 logging is async fire-and-forget | Cover basic logging latency in P3; full load testing deferred to NFR assessment |
| **Webhook event consumer fanout** | Redis Streams consumers are downstream; this epic only publishes | Functional consumer tests belong to the consuming epic's test design |
| **UI/frontend integration** | E04 is ClusterIP; no direct UI integration in this epic | Client API and Admin API (E02) mediate between frontend and AI Gateway |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E04-R-001** | **SEC** | Webhook signature bypass — HMAC-SHA256 validation of `X-Kraftdata-Signature` header must use constant-time comparison; naive string equality is vulnerable to timing attacks; a successful bypass allows arbitrary `execution.completed` or `execution.failed` events to be injected, potentially triggering false downstream state transitions for any tenant's active proposal | 2 | 3 | **6** | Use `hmac.compare_digest()` (constant-time) for signature comparison; reject requests with missing or malformed signature header before parsing body; unit test: valid signature passes, tampered signature rejected, missing header rejected | Backend / QA | Sprint 3 |
| **E04-R-002** | **TECH** | SSE stream proxy edge-case fragility — partial SSE frame reassembly (KraftData may send `data:` across multiple TCP packets), client disconnect cancellation racing against in-flight writes, and the three-layer timeout (15s heartbeat, 120s idle, 600s total) interact in non-obvious ways; incorrect handling causes goroutine/coroutine leaks, orphaned KraftData connections, or silent data loss for streaming executions | 3 | 2 | **6** | Explicit SSE frame buffer with newline-delimited reassembly; `asyncio.CancelledError` propagation on client disconnect; independent timer tasks with `asyncio.wait_for`; unit tests with frozen clock and mock SSE source; integration test with controlled timing | Backend / QA | Sprint 3–4 |
| **E04-R-003** | **DATA** | Redis Stream publish-or-acknowledge gap — webhook endpoint returns 200 immediately after `XADD`; if Redis connection fails or `XADD` raises mid-flight, the webhook is acknowledged to KraftData (200) but never published to the stream; downstream consumers never learn the execution completed; the execution row in `gateway.webhook_log` will show `processed=False` permanently, but no retry mechanism exists | 2 | 3 | **6** | Wrap `XADD` in try/except with structured error log at ERROR level; populate `webhook_log.processed=False` + `error_message` on failure; add observability alert criteria; acceptance test: mock Redis XADD failure → verify 200 returned to KraftData but error logged; add compensating monitoring (dead-letter alerting) to backlog | Backend / QA | Sprint 3 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E04-R-004 | PERF | Semaphore starvation under streaming load — asyncio.Semaphore is FIFO; long-running SSE streams hold their permit for the full stream duration (up to 600s); a burst of concurrent streaming requests fills the semaphore before sync calls can acquire permits, causing short sync calls to wait up to 30s then return 429; no priority lanes exist | 2 | 2 | 4 | Document known limitation in ADR; monitor active/queued counts via `GET /admin/rate-limit`; suggest separate semaphore pools for sync vs. stream in Phase 2; P2 integration test verifies queue behavior | Backend |
| E04-R-005 | TECH | Circuit breaker per-instance state — in-memory state acceptable for single replica (Sprint 3–4) but becomes a correctness issue if auto-scaled; Instance A may have OPEN circuit while Instance B has CLOSED, leading to inconsistent rejection behavior and misleading `GET /admin/circuits` output | 1 | 3 | 3 | Accept as known limitation for initial deployment (single replica); document in ADR; tag as pre-scale backlog item (migrate state to Redis when multi-replica planned) | Backend |
| E04-R-006 | TECH | Registry hot-reload race condition — `SIGHUP`-triggered reload replaces `AgentRegistry._entries` dict while concurrent `resolve()` calls read it; Python's GIL does not protect against partial updates on dict replacement; if reload fails validation, startup may proceed with empty registry | 2 | 2 | 4 | Use `asyncio.Lock` around registry replacement; validate new entries fully before swapping reference; unit test: concurrent resolve() during reload; if validation fails, keep old registry and log error | Backend / QA |
| E04-R-007 | OPS | Execution log async write backlog — fire-and-forget DB writes to `gateway.agent_executions` use an unbounded task queue; under sustained load, pending writes could exhaust connection pool or memory; failed writes are swallowed with an error log but are unrecoverable | 2 | 2 | 4 | Log DB write failures at ERROR level with execution_id; add backpressure (bounded asyncio queue or connection pool timeout); P2 test: mock DB failure → verify call succeeds, error logged | Backend |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E04-R-008 | OPS | testcontainers CI startup time — PostgreSQL + Redis via testcontainers may add 30–60s to CI integration test startup; flaky container health checks cause intermittent test failures | 2 | 1 | 2 | Monitor — configure `wait_for_service` with health check retry; add `@pytest.mark.slow` tag; consider Docker layer caching |
| E04-R-009 | DATA | YAML agents.yaml drift — 29 entries loaded at startup; KraftData UUIDs may be rotated without updating the file; stale UUID causes all calls to that agent to return 404 or 502 until file is updated and service reloaded | 1 | 2 | 2 | Monitor — add startup log of all loaded entries; `GET /admin/registry/reload` makes reloading explicit; document UUID rotation runbook |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

### System-Level Risk Inheritance

- **E04-R-001** (webhook signature bypass) → extends system **R-06** (LLM Prompt Injection, score 6). System-level hardening focuses on prompt injection at the LLM interface; E04 adds the webhook attack surface. A spoofed `execution.completed` event for a non-existent execution could corrupt proposal state downstream.
- **E04-R-003** (Redis publish gap) → extends system **R-05** (Tender Sync Failure, score 6). Both risks concern event-driven reliability. Redis Stream loss for agent execution events has the same consequence as Kafka consumer failure: downstream processors miss state transitions.
- **E04-R-005** (circuit breaker per-instance) → new epic-level risk; flags a pre-scale concern not covered in system-level design. Should be captured in the architecture doc for Phase 2.

---

## Entry Criteria

- [ ] E01 complete: monorepo scaffold operational; `services/ai-gateway/` directory structure created; `pyproject.toml` with required dependencies present
- [ ] PostgreSQL and Redis available in CI (Docker Compose services or testcontainers)
- [ ] `respx` mock router configured for KraftData outbound calls (no real KraftData credentials required for CI)
- [ ] `config/agents.yaml` seeded with at least 5 representative entries (including one agent, one team, one workflow type) for test fixtures; full 29-entry file not required until S04.03 complete
- [ ] E01 shared logging library (`eusolicit-common`) available for structured JSON logging wiring
- [ ] KraftData stage credentials available in a secure secrets store for manual integration smoke tests (tagged `@skip-ci`)

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95% or failures triaged and accepted with PM sign-off)
- [ ] No open high-severity bugs in circuit breaker, webhook signature validation, or execution logging
- [ ] Line coverage ≥85% on `app/services/` and `app/routers/` (S04.10 target)
- [ ] Integration test suite (S04.10) passes in under 60 seconds in CI
- [ ] `GET /health` and `GET /ready` probes passing in Docker Compose environment
- [ ] No orphaned KraftData connections after SSE client disconnect test (verified via active connection count)
- [ ] `gateway.agent_executions` row created and updated correctly for at least one sync and one streaming execution in integration tests

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on if time-constrained), NOT execution timing. See "Execution Strategy" for when tests run.

### P0 (Critical)

**Criteria:** Blocks core AI gateway function + High risk (≥6) + No workaround + All downstream AI feature epics depend on it

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E04-P0-001** | `GET /health` returns `{"status": "ok"}` with HTTP 200 unconditionally; service does not crash on startup with valid config | Integration | — | Run against Docker Compose; also verify in testcontainers fixture; assert exact JSON body |
| **E04-P0-002** | `GET /ready` returns 200 when both PostgreSQL and Redis are reachable; returns 503 when either is down | Integration | — | Use testcontainers: (a) both up → 200; (b) PostgreSQL stopped → 503; (c) Redis stopped → 503; assert response body includes which dependency failed |
| **E04-P0-003** | httpx client injects `Authorization: Bearer {KRAFTDATA_API_KEY}` on every outbound request — no request reaches KraftData without the auth header | Unit | — | Mock httpx transport with `respx`; call `call_kraftdata()` with arbitrary path; assert `Authorization` header present and matches configured key; also verify `Content-Type: application/json` default header |
| **E04-P0-004** | Agent registry loads all entries from `config/agents.yaml`; `resolve("executive-summary")` returns correct UUID and type; `resolve("nonexistent-agent")` raises `AgentNotFoundError` (mapped to 404 at HTTP layer) | Unit | — | Load a fixture YAML with 5 entries (1 agent, 1 team, 1 workflow + 2 more agents); assert entry count; assert resolve returns expected UUID; assert resolve of unknown name raises `AgentNotFoundError`; assert HTTP layer maps to 404 |
| **E04-P0-005** | `POST /agents/{id}/run` with logical name resolves to correct KraftData UUID and forwards request body; KraftData 200 response returned to caller with correct body | Integration | — | `respx` mock: verify URL contains resolved UUID, not logical name; assert request body forwarded verbatim; assert response body matches KraftData mock response |
| **E04-P0-006** | `POST /workflows/{id}/run` and `POST /teams/{id}/run` proxy to correct KraftData endpoints; type validation enforced (calling workflow endpoint with agent-type entry returns 400) | Integration | — | Three sub-cases: (a) valid workflow → correct URL; (b) valid team → correct URL; (c) workflow endpoint called with agent-type registry entry → 400 with clear error |
| **E04-P0-007** | Circuit breaker opens after 5 consecutive failures to agent X: 6th call returns 503 immediately without contacting KraftData | Unit | — | Call mock that always returns 500; assert circuit transitions CLOSED→OPEN after 5 failures; assert 6th call returns `CircuitOpenError` without HTTP call; verify with spy that `call_kraftdata()` not invoked on 6th call |
| **E04-P0-008** | Circuit breaker half-open recovery: after 30s cooldown, next call allowed through (HALF_OPEN); success → CLOSED; subsequent calls proceed normally | Unit | — | Freeze clock; trip circuit; advance 30s; verify next call is attempted (HALF_OPEN); mock success → assert CLOSED; assert following call proceeds normally |
| **E04-P0-009** | Retry logic: KraftData returns 500 once then 200 on retry; call succeeds; `execution_log.retry_count = 1`; retry delay ≈1s (with ±25% jitter bounds) | Unit | — | Mock respx: first call 500, second call 200; assert final result is 200; assert retry delay in range [0.75s, 1.25s]; assert execution log shows retry_count=1 |
| **E04-P0-010** | Webhook receiver: valid `X-Kraftdata-Signature` returns 200; correct stream name and message published to Redis; `gateway.webhook_log` entry created | Integration | E04-R-001 | Generate valid HMAC-SHA256 signature with configured secret; POST to `/webhooks/kraftdata`; assert 200; assert Redis `XADD` called with `agent.execution.completed` or `agent.execution.failed`; assert message JSON matches expected format; assert webhook_log row inserted |
| **E04-P0-011** | Webhook receiver: invalid `X-Kraftdata-Signature` returns 401; nothing published to Redis | Integration | E04-R-001 | POST with tampered signature; assert 401; assert Redis `XADD` NOT called; assert webhook_log row inserted with `signature_valid=False` |
| **E04-P0-012** | Webhook signature uses constant-time comparison (`hmac.compare_digest`); not vulnerable to early-exit string comparison | Unit | E04-R-001 | Inspect source for `hmac.compare_digest` usage; unit test verifies both valid and invalid (off-by-one byte) signatures with identical timing characteristics (timing assertion: time difference < 1ms for same-length tokens) |
| **E04-P0-013** | Every sync proxy call creates exactly one `gateway.agent_executions` row with accurate `start_time`, `end_time`, `status=success`, `latency_ms>0`, `caller_service` populated | Integration | — | Make proxy call through testcontainers stack; query `agent_executions` table after call; assert exactly 1 row; assert all fields non-null and within expected ranges |

**Total P0:** 13 tests, ~25–40 hours

---

### P1 (High)

**Criteria:** Important gateway features + Medium risk (3–5) + Common call paths + Workaround exists but impairs AI feature functionality

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E04-P1-001** | `POST /agents/{raw-uuid}/run` works without registry lookup — UUID passthrough bypasses `AgentRegistry.resolve()` and routes directly | Unit | — | Detect UUID format in path param; assert `resolve()` not called; assert KraftData URL uses the raw UUID verbatim |
| **E04-P1-002** | `POST /storage/{id}/files` multipart upload forwarded to KraftData `/client/api/v1/storage-resources/{id}/files`; response (file ID, status) returned to caller | Integration | — | Use `respx` to mock KraftData storage endpoint; POST multipart with small test PDF; assert correct KraftData URL; assert file ID returned |
| **E04-P1-003** | Missing `X-Caller-Service` header on any execution endpoint returns 400 with descriptive error message | Unit | — | Call each of the 5 execution endpoints without `X-Caller-Service`; assert 400; assert error body mentions the missing header |
| **E04-P1-004** | `X-Request-ID` header propagated to KraftData; generated and injected if not present in incoming request | Unit | — | (a) Incoming request has `X-Request-ID`: verify same value forwarded to KraftData; (b) No `X-Request-ID`: verify generated UUID injected in KraftData request header |
| **E04-P1-005** | KraftData 500 response returns 502 to caller with error details; KraftData timeout returns 504 | Integration | — | `respx` mock: (a) 500 → assert 502; (b) raise `httpx.TimeoutException` → assert 504; both cases: assert error body forwarded or wrapped |
| **E04-P1-006** | Retry exhaustion: 3 retries all fail with 500; final response is 502; `execution_log.retry_count = 3` | Unit | — | Mock: 4× 500; assert 502 returned; assert retry_count=3 in execution log; assert retry delays ≈1s, 2s, 4s (with jitter bounds ±25%) |
| **E04-P1-007** | 4xx responses from KraftData are NOT retried; returned immediately to caller; retry_count stays 0 | Unit | — | Mock: single 404 response; assert no retry attempt; assert call count=1 via spy; assert 404 (or mapped 502) returned immediately |
| **E04-P1-008** | Circuit breaker `GET /admin/circuits` returns current state for all agents: failure counts, last failure time, circuit state | Integration | — | Trip circuit for one agent; call `/admin/circuits`; assert OPEN state shown for that agent; assert other agents show CLOSED; assert failure count and timestamp fields present |
| **E04-P1-009** | SSE stream proxy happy path: `POST /agents/{id}/run-stream` returns `StreamingResponse` with `text/event-stream` content type; 3+ SSE events forwarded in order with correct `data:`, `event:`, `id:` fields | Integration | E04-R-002 | Use `respx` to mock SSE source with 3 test events; call stream endpoint; collect events; assert count, order, and field values |
| **E04-P1-010** | SSE upstream disconnect: KraftData drops connection mid-stream → final `event: error` with `data: {"error": "upstream_disconnected"}` sent to client; stream closed cleanly | Unit | E04-R-002 | Mock SSE source that emits 2 events then raises `httpx.RemoteProtocolError`; assert error event sent; assert stream closed; assert no exception propagated to caller |
| **E04-P1-011** | SSE client disconnect: calling service drops connection → KraftData upstream request cancelled within 5s; no orphaned connection | Unit | E04-R-002 | Mock client disconnect via `asyncio.CancelledError` on response write; assert upstream `httpx.AsyncClient.aclose()` or cancel called; assert no coroutine leaks (check active tasks before and after) |
| **E04-P1-012** | Webhook idempotency: duplicate `execution_id` within 1 hour is acknowledged (200 returned) but NOT re-published to Redis Stream | Integration | — | POST webhook twice with same `execution_id`; assert both return 200; assert Redis `XADD` called only once (spy/mock assertion); assert `webhook_log` has 2 rows (both logged) |
| **E04-P1-013** | Unknown event type in webhook returns 200 and logs WARN; nothing published to Redis | Unit | — | POST webhook with `event_type: "execution.unknown"`; assert 200; assert Redis `XADD` NOT called; assert WARN log emitted |
| **E04-P1-014** | Concurrency limit: with `CONCURRENCY_LIMIT=2`, 3 simultaneous sync calls — 2 proceed, 3rd queues and proceeds when a slot opens; all 3 eventually succeed | Integration | E04-R-004 | Use asyncio to fire 3 concurrent requests; mock KraftData with 200ms delay; assert all 3 succeed (no 429); assert max 2 active simultaneously via semaphore count |
| **E04-P1-015** | Rate limit queue timeout: with `CONCURRENCY_LIMIT=2` and `QUEUE_TIMEOUT=0.1s` (test override), 3 concurrent requests with slow KraftData (2s) — 3rd returns 429 after queue timeout | Unit | — | Mock semaphore full scenario with short timeout; assert 3rd call returns 429; assert `rate_limited` status in execution log |
| **E04-P1-016** | Circuit breaker HALF_OPEN → OPEN on failure: half-open test call fails → circuit reopens for another cooldown period | Unit | — | Trip circuit; advance clock 30s (HALF_OPEN); make call that returns 500; assert circuit re-opens (OPEN); assert subsequent call immediately returns 503 without HTTP attempt |
| **E04-P1-017** | Duplicate logical names in `agents.yaml` cause startup failure with clear error message; service does not start | Unit | — | Load YAML fixture with duplicate key; assert `AgentRegistry` raises `ValueError` or `StartupError` on construction; assert error message mentions the duplicate name |
| **E04-P1-018** | Streaming execution creates `agent_executions` row; `is_streaming=True`; `end_time` set when stream completes | Integration | — | Make streaming call through testcontainers; after stream ends, query `agent_executions`; assert `is_streaming=True`; assert `end_time IS NOT NULL`; assert `status=success` |
| **E04-P1-019** | Circuit-open rejection logged to `agent_executions` with `status=circuit_open` and `latency_ms=0` | Unit | — | Trip circuit; make call; assert execution log row has `status=circuit_open`, `latency_ms=0`, `error_message` describing circuit state |
| **E04-P1-020** | httpx client shuts down cleanly on app shutdown — no resource leaks; connection pool closed | Unit | — | Start FastAPI app; trigger lifespan shutdown; assert `httpx.AsyncClient.aclose()` called; assert no pending connections (verify via mock transport connection count) |

**Total P1:** 20 tests, ~20–30 hours

---

### P2 (Medium)

**Criteria:** Secondary flows + Low/Medium risk (1–4) + Edge cases + Admin/observability paths

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E04-P2-001** | Agent registry hot-reload via `POST /admin/registry/reload` updates registry without service restart; new entries resolve correctly; old entries no longer resolve after reload with fresh YAML | Integration | E04-R-006 | Write initial YAML with agent A; hot-reload with YAML containing agent B (not A); assert resolve(B) succeeds; assert resolve(A) raises AgentNotFoundError |
| **E04-P2-002** | Hot-reload with invalid YAML (e.g., duplicate keys) keeps existing registry intact; returns 400 with error message | Unit | E04-R-006 | Trigger reload with invalid fixture YAML; assert existing registry unchanged; assert endpoint returns 400 with error details |
| **E04-P2-003** | Concurrent `resolve()` calls during hot-reload do not panic or return partial data | Unit | E04-R-006 | Launch 10 concurrent resolve tasks; trigger reload mid-execution; assert all tasks complete without exception; no data race |
| **E04-P2-004** | SSE heartbeat: `event: heartbeat` sent every ~15s during idle stream; heartbeat timing within ±2s of configured interval | Unit | E04-R-002 | Freeze clock; mock SSE source with no events; advance time 15s; assert heartbeat event sent; advance another 15s; assert second heartbeat |
| **E04-P2-005** | SSE partial frame reassembly: KraftData sends `data:` field split across two TCP chunks; gateway buffers and forwards complete event | Unit | E04-R-002 | Mock transport yields `b"data: partial"` then `b" content\n\n"`; assert single complete event forwarded with correct `data: partial content` |
| **E04-P2-006** | SSE idle timeout: stream with no events for 120s triggers disconnect with `event: timeout`; connection cleaned up | Unit | E04-R-002 | Freeze clock; mock SSE source that yields nothing; advance 120s; assert timeout event sent; assert stream closed |
| **E04-P2-007** | Per-agent concurrency limit: agent configured with `max_concurrent: 2` in agents.yaml; 3rd concurrent call to that agent returns 429 even if global semaphore has capacity | Unit | E04-R-004 | Set `CONCURRENCY_LIMIT=10`, `max_concurrent=2` on test agent; fire 3 concurrent calls to that agent; assert 3rd returns 429; assert only 2 active simultaneously |
| **E04-P2-008** | `GET /admin/rate-limit` returns accurate `active_requests`, `queued_requests`, `total_rejected`, `concurrency_limit` in real time | Integration | — | Use asyncio to hold 3 requests in-flight while querying admin endpoint; assert `active_requests` and `queued_requests` match expected counts |
| **E04-P2-009** | `GET /admin/executions` with `agent_name` and `status` filters returns only matching rows; pagination (limit/offset) works correctly | Integration | — | Seed 10 execution rows with varied agent names and statuses; query with `agent_name=executive-summary&status=failed`; assert only matching rows returned; query with `limit=3&offset=3`; assert correct page |
| **E04-P2-010** | Execution logging failure does not cause proxy call to fail — mock DB insert error; call still returns 200; error logged at ERROR level | Unit | E04-R-007 | Mock `SessionLocal.execute()` to raise `asyncpg.PostgresError`; make proxy call; assert 200 returned; assert error log emitted with execution_id; assert no exception propagated to caller |
| **E04-P2-011** | Webhook `gateway.webhook_log` entry created for every received webhook, valid or invalid; `signature_valid` field correctly reflects validation result | Integration | — | POST valid webhook → assert `signature_valid=True`; POST invalid webhook → assert `signature_valid=False`; both rows present in `webhook_log` |
| **E04-P2-012** | Retry jitter: backoff delays randomized in range [delay * 0.75, delay * 1.25] for each attempt; assert over 10 samples that delays fall within bounds | Unit | — | Patch `random.uniform`; assert retry delays for attempts 1 (0.75–1.25s), 2 (1.5–2.5s), 3 (3.0–5.0s) are within jitter bounds |
| **E04-P2-013** | `CircuitOpenError` is NOT retried; call returns immediately with 503; retry_count=0 in log | Unit | — | Trip circuit; make call; assert retry decorator does not call httpx; assert retry_count=0; assert 503 returned |
| **E04-P2-014** | Connection pool enforces `CONCURRENCY_LIMIT`: with limit=3, max_connections=3 configured on httpx client; verify via mock transport connection count | Unit | — | Inspect httpx.AsyncClient limits configuration; assert `max_connections=CONCURRENCY_LIMIT`; assert `max_keepalive_connections=CONCURRENCY_LIMIT//2`; assert `keepalive_expiry=30` |
| **E04-P2-015** | Alembic migration creates both `gateway.agent_executions` and `gateway.webhook_log` tables with correct columns and all specified indexes | Integration | — | Run migration against testcontainers PostgreSQL; assert both tables exist; assert each index exists (`idx_agent_executions_agent_name`, etc.); assert column types and nullability constraints |
| **E04-P2-016** | `POST /workflows/{id}/run-stream` SSE proxy works with workflow-type registry entry; rejects if entry type is `agent` | Integration | — | Mock workflow SSE source; verify events forwarded; call stream endpoint with agent-type entry → assert 400 |
| **E04-P2-017** | Streaming holds semaphore permit for duration of stream; permit released only after stream closes; no other requests wait unnecessarily after stream ends | Unit | E04-R-004 | Start stream; assert semaphore `_value` decremented; close stream; assert semaphore `_value` restored; assert subsequent sync call proceeds immediately |

**Total P2:** 17 tests, ~12–20 hours

---

### P3 (Low)

**Criteria:** Nice-to-have + Observability benchmarks + Timing assertions + Manual/nightly only

| Test ID | Requirement | Test Level | Notes |
|---------|-------------|------------|-------|
| **E04-P3-001** | SSE total stream timeout: stream running >600s triggers disconnect with `event: timeout`; total elapsed time verified | Unit | Freeze clock; advance 601s; assert stream closed with timeout event; validate 600s threshold via config |
| **E04-P3-002** | Service startup time <3s in Docker Compose with cold start (no warm caches) | Integration | Docker Compose `up --wait`; measure time from container start to first successful `GET /health`; assert <3s |
| **E04-P3-003** | Webhook processing time <50ms: from HTTP request to 200 response (mocked Redis + DB) | Unit | Use mock Redis/DB; measure wall clock from request to response in unit test; assert <50ms average over 100 runs |
| **E04-P3-004** | `GET /admin/executions` pagination: 1000-row dataset; query with limit=50 completes in <200ms | Integration | Seed 1000 rows via factory; time paginated query; assert within latency target |
| **E04-P3-005** | `GET /admin/circuits` lists all 29 agent entries after full registry load; all show `CLOSED` state on fresh start | Integration | Load full 29-entry agents.yaml; call `/admin/circuits`; assert 29 entries; assert all `circuit_state=CLOSED`; assert `failure_count=0` for all |

**Total P3:** 5 tests, ~4–8 hours

---

## Execution Strategy

**PR (every pull request):** All unit tests, and integration tests using testcontainers (PostgreSQL + Redis) + `respx` KraftData mocks. Expected ~8–12 min with pytest-xdist parallelization. This includes all P0, P1, and P2 tests. Backend-only stack means no browser overhead.

**Nightly:** P3 timing benchmarks (E04-P3-002 startup time, E04-P3-003 webhook latency, E04-P3-004 pagination latency). These require controlled environments free from CI resource contention.

**Manual / Credentials-gated:** Real KraftData stage smoke tests (tagged `@skip-ci`): one live call to `/agents/{id}/run`, one live SSE stream, one real webhook receive. These are optional validation steps before Demo milestone and require `KRAFTDATA_API_KEY` in a secure secrets store.

**Philosophy:** Run everything in PRs if <15 min. Only defer timing/performance benchmarks and real-credential tests to nightly/manual.

---

## Resource Estimates

| Priority | Count | Total Hours | Notes |
|----------|-------|-------------|-------|
| P0 | 13 | ~25–40 hours | Webhook signature + circuit breaker setup; testcontainers infrastructure for integration tests |
| P1 | 20 | ~20–30 hours | SSE stream tests need careful async mock design; retry timing tests need clock freezing |
| P2 | 17 | ~12–20 hours | Registry reload, admin endpoints, log failure tests are straightforward |
| P3 | 5 | ~4–8 hours | Timing benchmarks; skip in early sprints if behind |
| **Total** | **55** | **~60–98 hours** | **~2–2.5 weeks, 1 QA** |

### Prerequisites

**Test Data:**

- `AgentRegistryFixture` — loads fixture YAML with 5–29 entries; auto-cleanup
- `KraftDataMockRouter` — `respx` router with pre-configured routes for all 7 endpoints; factory methods for 200/500/timeout/SSE responses
- `WebhookPayloadFactory` — generates signed webhook payloads (`execution.completed`, `execution.failed`, unknown event types); uses test HMAC secret
- `ExecutionLogFactory` — seeds `gateway.agent_executions` rows for pagination and filter tests

**Tooling:**

- `pytest` + `pytest-asyncio` for async test execution
- `httpx` + `respx` for KraftData API mocking (no real network calls in CI)
- `testcontainers-python` for PostgreSQL + Redis in integration tests
- `freezegun` or `unittest.mock.patch("asyncio.sleep")` for circuit breaker cooldown and SSE timeout tests
- `pytest-xdist` for parallel test execution in CI

**Environment:**

- `services/ai-gateway/` with `pyproject.toml` dependencies installed
- `config/agents-test.yaml` with 5+ entries (fast tests) and `config/agents-full.yaml` with 29 entries (P3 tests)
- CI: GitHub Actions with `services: postgres:`, `services: redis:` or testcontainers auto-pull
- Env vars: `KRAFTDATA_BASE_URL=http://mock-kraftdata`, `KRAFTDATA_API_KEY=test-key`, `WEBHOOK_SECRET=test-webhook-secret`

---

## Quality Gate Criteria

- **P0 pass rate:** 100% (no exceptions; webhook signature and circuit breaker tests are hard-fail)
- **P1 pass rate:** ≥95% (failures require triage and PM sign-off; SSE timing flakiness must be resolved, not skipped)
- **Security tests (E04-R-001 tests: P0-010, P0-011, P0-012):** 100% — all three webhook security tests must pass before Demo milestone
- **Line coverage:** ≥85% on `app/services/` and `app/routers/` (measured by `pytest-cov`; enforced as CI quality gate)
- **Integration test suite runtime:** <60 seconds (S04.10 target); if exceeded, investigate testcontainers startup or add pytest-xdist sharding
- **SSE leak check (E04-P1-011):** 100% — no orphaned connections test must pass; verified via active task count before/after

---

## Mitigation Plans

### E04-R-001: Webhook Signature Bypass (Score: 6)

**Mitigation Strategy:**
1. Use `hmac.compare_digest(computed_signature, received_signature)` — constant-time comparison prevents timing attacks
2. Validate `X-Kraftdata-Signature` header before parsing body; reject with 401 + `{"error": "invalid_signature"}` on missing or invalid header
3. Use `hmac.new(WEBHOOK_SECRET.encode(), body, hashlib.sha256).hexdigest()` for digest computation; validate both `sha256=<hex>` format and raw hex
4. Log rejected attempts at WARN with IP, timestamp, and signature prefix (not full value) for forensic analysis
5. Integration test E04-P0-010 (valid), E04-P0-011 (invalid), E04-P0-012 (constant-time) must all pass before Sprint 4

**Owner:** Backend Lead
**Timeline:** Sprint 3 (S04.07 implementation)
**Status:** Planned
**Verification:** E04-P0-010, E04-P0-011, E04-P0-012; manual code review confirms `hmac.compare_digest` used (not `==`)

---

### E04-R-002: SSE Stream Proxy Edge-Case Fragility (Score: 6)

**Mitigation Strategy:**
1. Explicit SSE frame buffer: accumulate bytes until `\n\n` boundary before forwarding; handle split frames atomically
2. Three independent asyncio Timer tasks: heartbeat (15s), idle reset (120s on each event received), total timeout (600s); cancel on stream close
3. On `asyncio.CancelledError` during write (client disconnect): catch, cancel upstream httpx request, log at INFO, return cleanly
4. On `httpx.RemoteProtocolError` or connection close from KraftData: send `event: error\ndata: {"error": "upstream_disconnected"}\n\n`; close stream
5. Unit tests with frozen clock (E04-P1-010, P1-011, P2-004, P2-005, P2-006, P3-001) cover all edge cases deterministically

**Owner:** Backend Lead
**Timeline:** Sprint 3–4 (S04.05 implementation)
**Status:** Planned
**Verification:** E04-P1-009 through E04-P1-011, E04-P2-004 through E04-P2-006, E04-P3-001; post-test connection count assertion for leak detection

---

### E04-R-003: Redis Stream Publish-or-Acknowledge Gap (Score: 6)

**Mitigation Strategy:**
1. Wrap `redis.xadd(...)` in `try/except` block; on failure: log at ERROR with `execution_id`, `event_type`, and full exception; set `webhook_log.processed=False` + `webhook_log.error_message=str(exception)`
2. Return 200 to KraftData regardless (fast webhook acknowledgement is contractually required)
3. Add compensating monitoring: alert if `webhook_log.processed=False` count exceeds threshold (backlog item for E05+)
4. Do NOT silently swallow the error — the ERROR log is the observable signal for on-call

**Owner:** Backend Lead
**Timeline:** Sprint 3 (S04.07 implementation)
**Status:** Planned
**Verification:** E04-P0-010 (happy path), E04-P0-011 (invalid sig), E04-P1-012 (idempotency); add manual test case: mock Redis `XADD` to raise → verify 200 + ERROR log + `webhook_log.processed=False`

---

## Assumptions and Dependencies

### Assumptions

1. `respx` version ≥0.20 is compatible with `httpx.AsyncClient` used in S04.02; lifespan-managed client can be overridden via transport injection in tests
2. `testcontainers-python` is available and Docker is present in CI runner; container pull adds <30s to test startup
3. `freezegun` or equivalent async-compatible clock freezing works with `asyncio.sleep`; if not, circuit breaker and SSE timeout tests use `unittest.mock.patch("asyncio.sleep", return_value=None)` with time tracking
4. `config/agents.yaml` uses the YAML schema defined in S04.03; schema changes before test development begins will require fixture updates
5. PostgreSQL schema (`gateway` schema, `agent_executions`, `webhook_log`) is stable after S04.08 migration; schema changes require test fixture regeneration
6. The httpx client singleton is accessible from test fixtures via FastAPI's `app.state` or a lifespan-exposed dependency for test transport injection

### Dependencies

| Dependency | Required By | Blocks |
|-----------|-------------|--------|
| S04.01 scaffold complete (FastAPI app, health endpoints) | Sprint 3 Week 1 | E04-P0-001, E04-P0-002, all integration tests |
| S04.02 httpx client (`call_kraftdata()` function) | Sprint 3 Week 1 | E04-P0-003, all proxy endpoint tests |
| S04.03 agent registry loaded | Sprint 3 Week 1 | E04-P0-004, E04-P0-005, E04-P0-006, E04-P1-001 |
| S04.08 DB schema migration (Alembic) | Sprint 3 Week 2 | E04-P0-013, E04-P1-018, E04-P1-019, E04-P2-009, E04-P2-015 |
| S04.07 webhook receiver + Redis | Sprint 3 Week 2 | E04-P0-010 through E04-P0-012, E04-P1-012, E04-P1-013 |
| E01 shared logging library (`eusolicit-common`) | Sprint 3 Week 1 | Structured JSON log assertions in integration tests |

### Risks to Plan

- **httpx transport injection complexity** → if `httpx.AsyncClient` singleton is not easily replaceable for test transport injection, wrap in a `get_kraftdata_client()` dependency and override via `app.dependency_overrides` in test fixtures
- **testcontainers flakiness in CI** → if container startup is unstable, add `@pytest.mark.retry(3)` on integration tests; investigate GitHub Actions resource limits; fall back to Docker Compose pre-started services
- **asyncio timing sensitivity** → SSE timeout tests using `asyncio.sleep` mocks may behave differently across Python versions; pin to Python 3.11 for consistent `asyncio` behavior; use `asyncio.wait_for` with explicit timeout rather than sleep-based timing

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|-------------------|--------|------------------|
| **E01 (monorepo scaffold, shared libs)** | `eusolicit-common` logging library; Docker Compose stack for AI Gateway | Shared library import must succeed; Docker Compose `ai-gateway` service entry must start cleanly; any E01 infra change requires E04-P0-001/P0-002 re-run |
| **E02 (Auth API / Client API)** | Client API calls AI Gateway using `X-Caller-Service: client-api`; Admin API calls using `X-Caller-Service: admin-api` | Regression: `X-Caller-Service` header requirement (E04-P0-003 notes); any E02 API contract change that affects AI-proxied calls must be retested against E04 endpoints |
| **E05+ (AI feature epics: ESPD, grant eligibility, budget builder, etc.)** | All AI-assisted features route through AI Gateway; agent logical names in `agents.yaml` must match what E05+ epics expect | Regression: agent registry logical name contract (`config/agents.yaml` as integration contract); any logical name rename or UUID change breaks E05+ callers |
| **KraftData platform (stage.sirma.ai)** | External dependency; AI Gateway wraps all KraftData endpoints | `respx` mocks are the CI proxy for KraftData; real credential smoke tests (`@skip-ci`) are the manual regression layer before each Demo milestone; KraftData API version changes require respx mock updates |
| **Redis Streams (E05+ event bus)** | `agent.execution.completed` and `agent.execution.failed` streams published by E04; consumed by downstream event processors | E04 publishes; E04 tests verify publish occurred; consumer regression belongs to E05+ test designs; stream name changes are a breaking interface change |

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk scoring (P×I), gate decision rules, mitigation lifecycle
- `probability-impact.md` — Probability and impact scale definitions
- `test-levels-framework.md` — Unit vs. Integration vs. E2E selection criteria
- `test-priorities-matrix.md` — P0–P3 criteria, coverage targets, execution ordering

### Related Documents

- **Epic:** `eusolicit-docs/planning-artifacts/epic-04-ai-gateway-service.md`
- **System-level architecture test design:** `eusolicit-docs/test-artifacts/test-design-architecture.md`
- **System-level QA test design:** `eusolicit-docs/test-artifacts/test-design-qa.md`
- **E01 test design (infra dependency):** `eusolicit-docs/test-artifacts/test-design-epic-01.md`
- **E03 test design (reference format):** `eusolicit-docs/test-artifacts/test-design-epic-03.md`

---

## Follow-on Workflows

- Run `*atdd` to generate failing P0 tests (separate workflow; not auto-run).
- Run `*automate` for broader coverage once E04 stories are implemented.

---

**Generated by:** BMad TEA Agent — Test Architect Module
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
