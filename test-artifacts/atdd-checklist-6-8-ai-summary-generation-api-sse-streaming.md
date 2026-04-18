---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-17'
workflowType: bmad-testarch-atdd
mode: story-level
storyId: 6-8-ai-summary-generation-api-sse-streaming
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-8-ai-summary-generation-api-sse-streaming.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/services/client-api/src/client_api/core/usage_gate.py
  - eusolicit-app/services/client-api/src/client_api/services/ai_gateway_client.py
  - eusolicit-app/services/client-api/src/client_api/models/client_ai_summary.py
  - eusolicit-app/services/client-api/tests/api/test_document_download.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/pyproject.toml
---

# ATDD Checklist: Story 6.8 — AI Summary Generation API (SSE Streaming)

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**Story:** S06.08 | **Status:** ready-for-dev
**Test File:** `eusolicit-app/services/client-api/tests/api/test_ai_summary.py`
**TDD Phase:** 🔴 RED — All tests marked `@pytest.mark.skip` until S06.08 is implemented

---

## Step 1: Preflight & Context

### Stack Detection

| Indicator | Found | Conclusion |
|-----------|-------|------------|
| `pyproject.toml` | ✅ `eusolicit-app/services/client-api/pyproject.toml` | Backend present |
| `playwright.config.ts` / `next.config.*` | ✅ (Next.js frontend exists) | Frontend present |
| **Detected stack** | — | `fullstack` |
| **Test scope** | Backend Python API (story targets `tests/api/test_ai_summary.py`) | Backend test mode |

### Prerequisites

- [x] Story `6-8-ai-summary-generation-api-sse-streaming.md` status: `ready-for-dev` with 10 clear ACs
- [x] Test framework: `pytest` + `pytest-asyncio` + `httpx` (confirmed by existing test files)
- [x] `client.ai_summaries` table: migration `018` already applied (S06.05)
- [x] `UsageGateContext` / `get_usage_gate`: implemented in `core/usage_gate.py`
- [x] `AiGatewayClient`: implemented in `services/ai_gateway_client.py` (needs `stream_agent()`)
- [x] `conftest.py` fixtures: `client_api_session_factory`, `rsa_test_key_pair`, `test_redis_client`
- [x] FK lesson from S06.07: `_OWNER_USER_ID` and `_OWNER_COMPANY_ID` match seeded test DB rows

### Key Architectural Notes Loaded

- **SSE vs HTTP Response Lifecycle**: UsageGate check MUST execute before `StreamingResponse` is created to ensure 429 is returned as JSON (not embedded in SSE stream).
- **DB Session in SSE Generator**: Generator must create its own session from session_factory — request-scoped session may be torn down before generator completes.
- **`stream_agent()` interface**: Will be `@asynccontextmanager` method on `AiGatewayClient` — mocked via `patch.object(AiGatewayClient, "stream_agent", side_effect=...)`.
- **Enterprise bypasses usage gate**: `limit=None` in `FEATURE_TIER_LIMITS["ai_summary"]["enterprise"]` → no Redis INCR, no header.

---

## Step 2: Generation Mode

**Selected mode:** AI Generation (backend story, no browser recording needed)

**Rationale:** All 10 ACs target Python FastAPI backend endpoints and DB/Redis integrations. No browser UI interactions involved in S06.08.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenarios Mapping

| AC | Scenario | Priority | Level | Test Function |
|----|----------|----------|-------|---------------|
| AC1 | POST without auth → 401 | P0 | API | `test_post_ai_summary_unauthenticated_returns_401` |
| AC1 | GET without auth → 401 | P0 | API | `test_get_ai_summary_unauthenticated_returns_401` |
| AC2 | POST free-tier → 403 + tier_limit + upgrade_url | P0 | API | `test_post_ai_summary_free_tier_returns_403` |
| AC2/AC8 | GET free-tier → 403 + tier_limit + upgrade_url | P0 | API | `test_get_ai_summary_free_tier_returns_403` |
| AC3/E06-P0-005 | POST Starter quota exhausted → 429 body + X-Usage-Remaining: 0 | P0 | API | `test_post_ai_summary_usage_limit_exceeded_returns_429` |
| AC3 | POST Starter success → X-Usage-Remaining: 9 header | P1 | API | `test_post_ai_summary_sets_x_usage_remaining_header` |
| AC3 | POST Enterprise → no X-Usage-Remaining, stream completes | P1 | Integration | `test_post_ai_summary_enterprise_bypasses_usage_gate` |
| AC6/AC7/E06-P0-009 | POST SSE: delta(s) → metadata → done; summary persisted in DB | P0 | Integration | `test_post_ai_summary_sse_event_order_and_db_persistence` |
| AC8/E06-P0-008 | GET cached summary < 24 h → 200 cached=true, counter unchanged | P0 | Integration | `test_get_ai_summary_cached_no_counter_decrement` |
| AC8/E06-P1-021 | GET cached summary → no AI Gateway call | P1 | API | `test_get_ai_summary_cached_no_ai_gateway_call` |
| AC8 | GET no summary → 200 cached=false | P1 | API | `test_get_ai_summary_no_summary_returns_cached_false` |
| AC8 | GET expired summary (> 24 h) → 200 cached=false | P1 | API | `test_get_ai_summary_expired_returns_cached_false` |
| AC8/E06-R-007 | GET stale: opp updated after summary → cached=false, stale=true | P2 | Integration | `test_get_ai_summary_stale_when_opportunity_updated_after_summary` |
| AC6/E06-P1-022 | POST AI Gateway error → error event streamed; no DB row | P1 | Integration | `test_post_ai_summary_ai_gateway_error_streams_error_event` |
| AC9 | Both endpoints in /openapi.json with correct status codes | P2 | API | `test_ai_summary_endpoints_in_openapi` |

**Total tests:** 15
**Priority breakdown:** P0: 5 | P1: 7 | P2: 3

### Tests NOT covered here (deferred)

| Test ID | Reason for Deferral |
|---------|---------------------|
| E06-P0-004 | Atomic counter race (2 concurrent requests at limit−1) — requires testcontainers Redis + asyncio.gather; separate integration test module recommended |
| E06-P2-007 | Stale cache staleness-triggered regeneration with metering — complex multi-step flow; depends on E06-P0-004 infrastructure |
| E06-P3-001 | OpenAPI schema coverage for all 8 endpoints — epic-level, not story-level |

---

## Step 4: TDD Red Phase — Test Generation

### Mode

**Execution Mode:** Sequential (backend Python project, single agent)
**TDD Phase:** 🔴 RED — Tests use `@pytest.mark.skip` to document intent

### Python TDD Red-Phase Convention

Python equivalent of TypeScript `test.skip()`:

```python
@pytest.mark.skip(reason="TDD RED PHASE: S06.08 not implemented — <specific reason>")
@pytest.mark.asyncio
async def test_<name>(...):
    ...
```

When the developer removes `@pytest.mark.skip`, tests will FAIL (red) because:
- `POST /api/v1/opportunities/{opportunity_id}/ai-summary` returns 404 (not registered)
- `GET /api/v1/opportunities/{opportunity_id}/ai-summary` returns 404 (not registered)
- `AiGatewayClient.stream_agent()` raises `AttributeError` (method not implemented)
- `AISummaryGetResponse` schema not importable (class not defined)

### TDD Red Phase Compliance Check

- [x] All 15 tests decorated with `@pytest.mark.skip`
- [x] All tests assert EXPECTED behavior (not placeholders like `assert True`)
- [x] No `@pytest.mark.skip` removed prematurely
- [x] No placeholder assertions (`assert True`, `assert 1 == 1`)
- [x] Realistic test data (real seeded user/company IDs, valid JWT payloads)
- [x] Tests structured to verify both happy-path and error scenarios from ACs

### File Written

| File | Status |
|------|--------|
| `eusolicit-app/services/client-api/tests/api/test_ai_summary.py` | ✅ Created |

---

## Step 4C: Aggregation

### Test Infrastructure

#### Mock Helpers (in test file)

| Helper | Purpose |
|--------|---------|
| `_make_jwt(rsa_test_key_pair, *, user_id, company_id, tier)` | RS256 JWT with arbitrary tier — mirrors `test_document_download.py` |
| `_mock_sse_stream_success()` | `@asynccontextmanager` → delta → delta → metadata → done |
| `_mock_sse_stream_error()` | `@asynccontextmanager` → delta → error |
| `_stream_agent_success(*args)` | Side-effect function for `patch.object(AiGatewayClient, "stream_agent", ...)` |
| `_stream_agent_error(*args)` | Side-effect function for error path |
| `_make_pipeline_session_override(updated_at)` | Returns `get_pipeline_readonly_session` override with mock result |
| `_seed_ai_summary(session_factory, ...)` | Inserts `client.ai_summaries` row; returns `summary_id` string |

#### Fixtures (in test file)

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `ai_summary_app` | function | Extends base `app`; overrides `get_db_session` with commit sessions; clears AI GW singleton cache |
| `cleanup_ai_summaries` | function (autouse) | Deletes test-seeded `ai_summaries` rows for `(_TEST_OPP_ID, _OWNER_USER_ID)` after each test |

#### Conftest Fixtures Used (from `tests/conftest.py`)

| Fixture | Used For |
|---------|---------|
| `app` | Base FastAPI app with test Redis + stub email service |
| `client_api_session_factory` | DB operations (seed + verify) |
| `rsa_test_key_pair` | RSA keys for JWT signing |
| `rsa_env_setup` (autouse) | Injects test keys into security module |
| `test_redis_client` | Real Redis DB index 1 (base app override) |
| `flush_rate_limit_keys` (autouse) | Clears rate-limit Redis keys between tests |

### Fixture Needs for Green Phase

When implementing S06.08 and entering the GREEN phase, the following additional fixtures / setups may be needed:

- `get_pipeline_readonly_session` dependency override in `ai_summary_app` fixture (to avoid requiring real `pipeline.opportunities` table in test DB)
- Mocked `build_executive_summary_payload` or a seeded `pipeline.opportunities` row for full integration tests
- testcontainers Redis for E06-P0-004 (concurrency race — not in this checklist)

---

## Step 5: Validation & Completion

### Validation Checklist

- [x] **Prerequisites satisfied**: conftest.py has all required base fixtures; `ai_summaries_table` model exists; `usage_gate.py` implements `FEATURE_TIER_LIMITS`
- [x] **Test file created correctly**: `tests/api/test_ai_summary.py` exists with 15 test functions
- [x] **All tests marked skip**: Every test has `@pytest.mark.skip(reason="TDD RED PHASE: ...")`
- [x] **No placeholder assertions**: All assertions verify specific expected behavior
- [x] **AC coverage complete**: All 10 ACs have ≥1 test (E06-P0-004 deferred with rationale)
- [x] **FK constraints respected**: DB-seeded rows use `_OWNER_USER_ID` + `_OWNER_COMPANY_ID` (real seeded IDs); JWT-only actors use arbitrary UUIDs
- [x] **Follows S06.07 test patterns**: mirrors `test_document_download.py` structure (`_make_jwt`, `_seed_*`, autouse cleanup, `download_app` → `ai_summary_app`)
- [x] **Temp artifacts**: none (no CLI/MCP browser sessions used — backend-only story)

### Acceptance Criteria Coverage Summary

| AC | Description | Tests Covering | Status |
|----|-------------|---------------|--------|
| AC1 | 401 for unauthenticated POST + GET | 2 tests | ✅ |
| AC2 | 403 + tier_limit for free-tier POST | 1 test | ✅ |
| AC3 | UsageGate: 429 body/header; X-Usage-Remaining on success; Enterprise bypass | 3 tests | ✅ |
| AC4 | Opportunity data fetched via get_pipeline_readonly_session | Covered via mock in SSE tests | ✅ |
| AC5 | AiGatewayClient.stream_agent() called | Covered implicitly in SSE tests (mocked + asserted) | ✅ |
| AC6 | SSE events: delta → metadata → done → error format | 2 tests (P0-009 + P1-022) | ✅ |
| AC7 | Summary persisted with correct fields on done | 1 test (P0-009) | ✅ |
| AC8 | GET cached < 24h no decrement; stale; no summary; expired; free-tier 403 | 6 tests | ✅ |
| AC9 | OpenAPI schema includes both endpoints with status codes | 1 test | ✅ |
| AC10 | Integration tests exist covering all specified scenarios | This file IS AC10 | ✅ |

### Implementation Guidance for Developer

#### Files to Create/Modify (from story Tasks)

| File | Action | Task |
|------|--------|------|
| `src/client_api/services/ai_gateway_client.py` | Add `stream_agent()` `@asynccontextmanager` method | Task 1 |
| `src/client_api/schemas/opportunities.py` | Append `AISummaryGetResponse` Pydantic model | Task 2 |
| `src/client_api/services/opportunity_service.py` | Add `get_cached_ai_summary`, `persist_ai_summary`, `build_executive_summary_payload` | Task 3 |
| `src/client_api/api/v1/opportunities.py` | Add `POST /{opportunity_id}/ai-summary` + `GET /{opportunity_id}/ai-summary` routes | Task 4 |
| `src/client_api/config.py` | Add `max_concurrent_sse_streams: int = Field(default=10)` | Task 4 |

#### TDD Green Phase Instructions

After implementing S06.08:

1. **Remove `@pytest.mark.skip` decorators** from all 15 tests in `test_ai_summary.py`
2. **Run tests** (will initially fail — RED):
   ```bash
   cd eusolicit-app/services/client-api
   pytest tests/api/test_ai_summary.py -v
   ```
3. **Fix implementation** until all tests pass (GREEN)
4. **Key failure patterns to expect**:
   - `404 Not Found` → routes not registered in `opportunities.py`
   - `AttributeError: 'AiGatewayClient' object has no attribute 'stream_agent'` → Task 1 missing
   - `ImportError: cannot import name 'AISummaryGetResponse'` → Task 2 missing
   - `AssertionError: cached=false when expected cached=true` → staleness logic bug in Task 3
   - `X-Usage-Remaining` header absent → header forwarding from `response.headers` to `StreamingResponse` missing (see Dev Notes)
5. **Commit passing tests** with story reference

### Next Recommended Workflow

After S06.08 implementation:

1. **`/bmad-testarch-automate`** — expand coverage beyond P0/P1 (add E06-P0-004 concurrency race with testcontainers Redis, P2-007 stale cache + re-metering, full SSE concurrency cap test)
2. **`/bmad-testarch-ci`** — wire `test_ai_summary.py` into CI pipeline alongside other E06 tests

---

## Summary

```
✅ ATDD Test Generation Complete (TDD RED PHASE)

🔴 TDD Red Phase: 15 Failing Tests Generated

📊 Summary:
- Total Tests: 15 (all with @pytest.mark.skip)
  - P0: 5 (auth + tier gate + usage gate 429 + SSE event order + cache no-decrement)
  - P1: 7 (usage remaining header + enterprise bypass + cached no-GW-call + no-summary + expired + error event)
  - P2: 3 (stale check + OpenAPI schema)
- API Tests: 8 (pure HTTP assertion, no DB side-effects)
- Integration Tests: 7 (DB seed/verify + Redis state + mock SSE stream)
- Tests Deferred: 3 (E06-P0-004 concurrency race, E06-P2-007 stale regen, E06-P3-001 OpenAPI all endpoints)

📂 Generated Files:
- tests/api/test_ai_summary.py (15 tests, all @pytest.mark.skip — TDD RED PHASE)
- eusolicit-docs/test-artifacts/atdd-checklist-6-8-ai-summary-generation-api-sse-streaming.md

✅ Acceptance Criteria Coverage:
- AC1: 2 tests (401 auth guard — POST + GET)
- AC2: 1 test (403 free-tier — POST); AC8 free: 1 test
- AC3: 3 tests (429 with correct body/header; X-Usage-Remaining; Enterprise bypass)
- AC4: Mocked pipeline session in SSE tests
- AC5: stream_agent() call verified via mock assertion
- AC6: 2 tests (SSE event sequence; error event)
- AC7: 1 test (DB persistence in E06-P0-009)
- AC8: 6 tests (cached; no-GW-call; no-summary; expired; stale; free-tier 403)
- AC9: 1 test (OpenAPI schema)
- AC10: This file IS the AC10 test suite

🚦 TDD Red Phase Compliance: ALL PASS
- All 15 tests decorated with @pytest.mark.skip
- All assertions verify specific EXPECTED behavior
- No placeholder assertions
- FK constraints respected (seeded user/company IDs used for DB rows)

📝 Next Steps:
1. Implement S06.08 (see story Tasks 1–5)
2. Remove @pytest.mark.skip from all tests in test_ai_summary.py
3. Run: pytest tests/api/test_ai_summary.py -v → verify FAIL (RED)
4. Fix implementation until tests PASS (GREEN)
5. Commit: "feat: S06.08 AI summary SSE streaming — tests passing"
```

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
