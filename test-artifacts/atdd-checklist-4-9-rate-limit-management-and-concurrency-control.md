---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-23'
storyId: '4.9'
storyKey: '4-9-rate-limit-management-and-concurrency-control'
storyFile: '/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/4-9-rate-limit-management-and-concurrency-control.md'
atddChecklistPath: '/home/debian/Projects/eusolicit/test_artifacts/atdd-checklist-4-9-rate-limit-management-and-concurrency-control.md'
generatedTestFiles:
  - '/home/debian/Projects/eusolicit/eusolicit-app/services/ai-gateway/tests/unit/test_rate_limiter_atdd_s04_09.py'
inputDocuments:
  - '/home/debian/Projects/eusolicit/_bmad/bmm/config.yaml'
  - '/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/4-9-rate-limit-management-and-concurrency-control.md'
  - '/home/debian/Projects/eusolicit/eusolicit-app/services/ai-gateway/pyproject.toml'
  - '/home/debian/Projects/eusolicit/eusolicit-app/services/ai-gateway/tests/unit/test_rate_limiter.py'
  - '/home/debian/Projects/eusolicit/eusolicit-app/services/ai-gateway/tests/unit/test_admin_rate_limit.py'
  - '/home/debian/Projects/eusolicit/eusolicit-app/services/ai-gateway/tests/integration/test_e2e_webhooks_ratelimit.py'
tddPhase: 'RED'
detectedStack: 'backend'
---

# ATDD Checklist — Story 4.9: Rate Limit Management and Concurrency Control

**Date:** 2026-04-23
**Story:** S04.09 — Rate Limit Management and Concurrency Control
**Status:** `ready-for-dev`
**Stack:** Backend (Python FastAPI, pytest, `asyncio_mode = "auto"`)
**Service:** `services/ai-gateway`

---

## TDD Red Phase Summary

🔴 **Red-phase scaffolds generated** (gap-filling tests for ACs 3, 5, 8, 9)

The story's dev notes include a complete implementation spec and pre-written test patterns.
Existing unit test files (`test_rate_limiter.py`, `test_admin_rate_limit.py`) cover ACs 1, 2, 3 (partial), 4, 6, 7 but are **not marked `@pytest.mark.skip`** — they were pre-generated alongside the implementation skeleton.

A new ATDD scaffold file `test_rate_limiter_atdd_s04_09.py` was created with `@pytest.mark.skip` markers for the identified coverage gaps: **AC 3 (endpoint-level latency=0), AC 5 (streaming 429 before SSE headers), AC 8 (env var loading), AC 9 (exception handler)**.

---

## Step 1: Preflight & Context Loading

| Item | Status |
|------|--------|
| Story file loaded | ✅ |
| Story approved with acceptance criteria | ✅ (9 ACs) |
| `conftest.py` found at `tests/conftest.py` | ✅ |
| `pyproject.toml` found with `asyncio_mode = "auto"` | ✅ |
| Detected stack | `backend` |
| Test framework | `pytest` + `pytest-asyncio` (auto mode) |

---

## Step 2: Generation Mode

**Mode:** AI generation (backend stack — no browser recording applicable)

Rationale: Pure asyncio logic (rate limiter) and FastAPI ASGI endpoints are best tested
via in-process `ASGITransport` + `AsyncMock` patterns. No live service or browser required.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Level Mapping

| AC | Description | Test Level | Priority |
|----|------------|-----------|---------|
| AC 1 | `GatewayRateLimiter` structure: global semaphore, per-agent dict, `stats()` | Unit | P1 |
| AC 2 | `CONCURRENCY_LIMIT=2`, 3 concurrent → 2 proceed, 3rd queues, all return 200 | Unit | P1 |
| AC 3 | Queue timeout → 429, `rate_limited` in `gateway.agent_executions`, `latency_ms=0` | Unit + Integration | P1 |
| AC 4 | Per-agent `max_concurrent` enforced; 4th call → 429 even if global has capacity | Unit | P2 |
| AC 5 | Streaming endpoint acquires slot BEFORE `StreamingResponse`; 429 before SSE headers | Endpoint/Unit | P2 |
| AC 6 | `GET /admin/rate-limit` returns live counters | Unit (endpoint) | P2 |
| AC 7 | WARN log at queue > 50%; ERROR log on rejection | Unit | P2 |
| AC 8 | `AIGatewaySettings.queue_timeout` loaded from `QUEUE_TIMEOUT` env var | Unit (config) | P2 |
| AC 9 | `main.py` lifespan inits rate limiter; `RateLimitError` → 429 handler registered | Unit (endpoint) | P2 |

### Red Phase: All tests must FAIL before implementation

New ATDD scaffold tests are marked `@pytest.mark.skip`. Remove the skip decorator for each
task as it is implemented to activate the test and verify the green phase.

---

## Step 4: Test Generation

### Existing Tests (Pre-Generated — NOT skip-tagged)

These tests were pre-generated from the story's dev notes. They will **PASS** once the
implementation is in place (and the implementation was also pre-generated). The developer
should verify they pass after completing each task.

| Test File | Tests | ACs Covered |
|-----------|-------|------------|
| `tests/unit/test_rate_limiter.py` | 9 tests | AC 1, 2, 3, 4, 5 (slot mechanics), 7 |
| `tests/unit/test_admin_rate_limit.py` | 2 tests | AC 6 |

**Tests in `test_rate_limiter.py`:**

| Test | AC | Priority | Traceability |
|------|----|----------|-------------|
| `test_semaphore_blocks_at_limit_then_proceeds` | AC 2 | P1 | E04-P1-014 |
| `test_queue_timeout_raises_rate_limit_error` | AC 3 | P1 | E04-P1-015 |
| `test_total_rejected_incremented_on_timeout` | AC 3 | P1 | — |
| `test_per_agent_limit_enforced` | AC 4 | P2 | E04-P2-007 |
| `test_warn_logged_at_queue_50_percent` | AC 7 | P2 | — |
| `test_stream_slot_holds_permit` | AC 5 (slot) | P2 | E04-P2-017 |
| `test_stats_returns_accurate_active_count` | AC 1, 6 | P2 | E04-P2-008 |
| `test_configure_agent_creates_per_agent_semaphore` | AC 1 | P2 | — |
| `test_stream_slot_release_is_idempotent` | AC 5 | P3 | — |

**Tests in `test_admin_rate_limit.py`:**

| Test | AC | Priority | Traceability |
|------|----|----------|-------------|
| `test_admin_rate_limit_endpoint` | AC 6 | P2 | E04-P2-008 |
| `test_admin_rate_limit_endpoint_when_not_initialised` | AC 6 | P3 | — |

---

### New ATDD Scaffold Tests (RED PHASE — `@pytest.mark.skip`)

**File:** `tests/unit/test_rate_limiter_atdd_s04_09.py`

| Test | AC | Priority | Description |
|------|----|----------|-------------|
| `test_run_agent_logs_rate_limited_with_zero_latency` | AC 3 | P1 | Endpoint sets `latency_ms=0` when `RateLimitError` raised; `log_execution_complete(status='rate_limited', latency_ms=0)` |
| `test_streaming_agent_429_before_sse_headers` | AC 5 | P1 | `POST /agents/{id}/run-stream` → 429 JSON (not 200 SSE) when rate limited; verifies eager slot acquisition |
| `test_streaming_workflow_429_before_sse_headers` | AC 5 | P2 | Same as above for workflow stream endpoint |
| `test_queue_timeout_default_is_30` | AC 8 | P2 | `AIGatewaySettings.queue_timeout == 30.0` by default |
| `test_queue_timeout_loaded_from_env_var` | AC 8 | P2 | `QUEUE_TIMEOUT=15` env var sets `queue_timeout=15.0` |
| `test_rate_limit_exception_handler_maps_to_429` | AC 9 | P1 | `@app.exception_handler(RateLimitError)` returns 429 with `{"error": "rate_limit_exceeded", "agent_name": ..., "detail": ...}` |
| `test_rate_limit_exception_handler_null_agent_name` | AC 9 | P2 | Handler serialises `agent_name=None` as JSON `null` |

**Total new tests:** 7
**All 7 tests marked `@pytest.mark.skip` (TDD red phase)**

---

## Step 4C: Coverage Aggregation

### Complete AC Coverage Matrix

| AC | Pre-Generated Tests | New ATDD Scaffolds | Coverage |
|----|--------------------|--------------------|---------|
| AC 1 | `test_configure_agent_creates_per_agent_semaphore`, `test_stats_returns_accurate_active_count` | — | ✅ Full |
| AC 2 | `test_semaphore_blocks_at_limit_then_proceeds` | — | ✅ Full |
| AC 3 | `test_queue_timeout_raises_rate_limit_error`, `test_total_rejected_incremented_on_timeout` | `test_run_agent_logs_rate_limited_with_zero_latency` | ✅ Full |
| AC 4 | `test_per_agent_limit_enforced` | — | ✅ Full |
| AC 5 | `test_stream_slot_holds_permit` (slot mechanics only) | `test_streaming_agent_429_before_sse_headers`, `test_streaming_workflow_429_before_sse_headers` | ✅ Full |
| AC 6 | `test_admin_rate_limit_endpoint`, `test_admin_rate_limit_endpoint_when_not_initialised` | — | ✅ Full |
| AC 7 | `test_warn_logged_at_queue_50_percent` | — | ✅ Full |
| AC 8 | — | `test_queue_timeout_default_is_30`, `test_queue_timeout_loaded_from_env_var` | ✅ Full |
| AC 9 | — | `test_rate_limit_exception_handler_maps_to_429`, `test_rate_limit_exception_handler_null_agent_name` | ✅ Full |

**P0 coverage:** N/A (no P0 tests in this story)
**P1 coverage:** 100% (ACs 2, 3, 5 — new AC 3 + AC 5 ATDD tests)
**P2 coverage:** 100% (ACs 1, 4, 6, 7, 8, 9)
**Overall:** 100% (9/9 ACs covered)

---

## Integration Test Coverage

**Scenario 8** in `tests/integration/test_e2e_webhooks_ratelimit.py` provides
end-to-end validation (S04.10) for:
- AC 2: 2 concurrent requests succeed, 3rd returns 429 (testcontainers + respx)
- AC 3: `gateway.agent_executions` row with `status='rate_limited'`, `latency_ms=0`
- AC 6: `GET /admin/rate-limit` returns `total_rejected >= 1`

This integration test requires testcontainers (`pytest.mark.integration`).

---

## Step 5: Validation & Completion Summary

### TDD Cycle Instructions

**For each task during implementation, follow this cycle:**

1. **Task 1 (config.py queue_timeout):**
   Remove `@pytest.mark.skip` from `test_queue_timeout_default_is_30` and `test_queue_timeout_loaded_from_env_var`
   → Run: `pytest tests/unit/test_rate_limiter_atdd_s04_09.py::test_queue_timeout_default_is_30 -v`
   → Verify RED (fails), implement, verify GREEN (passes)

2. **Task 2 (exceptions.py RateLimitError):**
   Existing tests in `test_rate_limiter.py` and `test_admin_rate_limit.py` cover this.
   → Run: `pytest tests/unit/test_rate_limiter.py -v`

3. **Task 3 (rate_limiter.py service):**
   All tests in `test_rate_limiter.py` become active.
   → Run: `pytest tests/unit/test_rate_limiter.py -v`

4. **Task 5 (main.py lifespan + exception handler):**
   Remove `@pytest.mark.skip` from `test_rate_limit_exception_handler_maps_to_429` and `test_rate_limit_exception_handler_null_agent_name`
   → Run: `pytest tests/unit/test_rate_limiter_atdd_s04_09.py::test_rate_limit_exception_handler_maps_to_429 -v`

5. **Task 6+7 (execution.py rate limit handling):**
   Remove `@pytest.mark.skip` from `test_run_agent_logs_rate_limited_with_zero_latency`
   → Run: `pytest tests/unit/test_rate_limiter_atdd_s04_09.py::test_run_agent_logs_rate_limited_with_zero_latency -v`
   Remove `@pytest.mark.skip` from streaming tests
   → Run: `pytest tests/unit/test_rate_limiter_atdd_s04_09.py -k "streaming" -v`

6. **All tasks complete:** Run full unit suite:
   `pytest tests/unit/ -v --tb=short`

---

### Generated Files

| File | Type | Status |
|------|------|--------|
| `tests/unit/test_rate_limiter.py` | Pre-generated unit tests (not skip-tagged) | Exists |
| `tests/unit/test_admin_rate_limit.py` | Pre-generated unit tests (not skip-tagged) | Exists |
| `tests/unit/test_rate_limiter_atdd_s04_09.py` | **ATDD red-phase scaffolds** (all skip-tagged) | **NEW** |
| `tests/integration/test_e2e_webhooks_ratelimit.py` | Integration tests (S04.10) | Exists |

---

### Key Risks and Notes

1. **Pre-generated implementation:** Both the source code (`rate_limiter.py`, `execution.py` modifications, `admin.py` route, `main.py` handler) and the unit tests in `test_rate_limiter.py` were pre-generated from the story's dev notes. The developer should verify these match the final implementation, not just accept them as-is.

2. **Streaming slot release (E04-R-004):** `asyncio.Semaphore` is FIFO; streaming requests hold permits for up to 600s. A burst of streaming requests can starve sync callers. This is documented in `rate_limiter.py` as a known limitation and tracked via `GET /admin/rate-limit`. The tests in this story do not cover starvation scenarios (deferred to a future story).

3. **WARN log test reliability (AC 7):** `test_warn_logged_at_queue_50_percent` uses `capsys` to check structlog output to stdout. If structlog is configured to output JSON (not text) or to a different handler, the assertion `"queue_depth_warning" in captured.out` may fail. Verify structlog output format matches the assertion.

4. **STATUS_RATE_LIMITED constant:** `execution_logger.py` defines `STATUS_RATE_LIMITED = "rate_limited"`. The integration test (Scenario 8) verifies the DB row has `status == "rate_limited"`. Ensure the constant is not renamed.

---

### Recommended Next Steps

1. **Activate ATDD tests** task-by-task by removing `@pytest.mark.skip` as each task is implemented
2. **Run full unit suite** after all tasks: `pytest tests/unit/ --tb=short`
3. **Run integration Scenario 8** after all tasks (requires `make infra`): `pytest tests/integration/test_e2e_webhooks_ratelimit.py::test_scenario8_rate_limit -v -s`
4. **Next workflow:** `bmad-dev-story` — implement Story 4.9 following the task order in the story file

---

*Generated by: bmad-testarch-atdd workflow*
*Story: 4.9 — Rate Limit Management and Concurrency Control*
*Epic: 4 — AI Gateway*
