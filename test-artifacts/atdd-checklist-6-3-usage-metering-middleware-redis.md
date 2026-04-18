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
storyId: 6-3-usage-metering-middleware-redis
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
executionMode: sequential
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-3-usage-metering-middleware-redis.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit-app/_bmad/bmm/config.yaml
  - eusolicit-app/_bmad/tea/config.yaml
---

# ATDD Checklist: Story 6.3 — Usage Metering Middleware (Redis)

**Date:** 2026-04-17
**Author:** TEA Master Test Architect (bmad-testarch-atdd)
**TDD Phase:** 🔴 RED (failing tests — awaiting implementation)
**Story:** eusolicit-docs/implementation-artifacts/6-3-usage-metering-middleware-redis.md
**Epic Test Design:** eusolicit-docs/test-artifacts/test-design-epic-06.md

---

## Preflight Summary

| Check | Status | Notes |
|-------|--------|-------|
| Stack detected | ✅ `backend` | Python/FastAPI/pytest; no browser testing |
| Story status | ✅ `ready-for-dev` | AC1–AC10 fully specified |
| Story has clear acceptance criteria | ✅ 10 ACs | All ACs fully specified with implementation code |
| Test framework | ✅ pytest + pytest-asyncio | conftest.py with app/test_client/redis fixtures |
| `tests/unit/` directory | ✅ exists | `__init__.py` present |
| `tests/integration/` directory | ✅ exists | `__init__.py` present |
| `core/usage_gate.py` | ❌ NOT IMPLEMENTED | Defined in story Task 1.1 — ATDD red phase |
| `fakeredis[aioredis]` | ⚠️ NOT IN pyproject.toml | Must be added to `[project.optional-dependencies] dev` |
| `testcontainers[redis]` | ⚠️ NOT IN pyproject.toml | Must be added to `[project.optional-dependencies] dev` |
| `eusolicit_common.exceptions.AppException` | ✅ EXISTS | Used in S06.02; error/status_code/details fields confirmed |
| Epic test design (E06) | ✅ LOADED | test-design-epic-06.md — risks E06-R-002/010, IDs E06-P0-004/005, E06-P1-008/009, E06-P3-004 |

---

## Generation Mode

**Mode selected:** AI Generation (backend stack — no browser recording needed)

The story provides complete implementation code and test specifications in Tasks 1–3. Tests are
adapted from the story spec with guarded imports and `@pytest.mark.skip` decorators to establish
the TDD red phase without breaking CI collection.

---

## 🔴 TDD Red Phase: Failing Tests Generated

All test functions are decorated with `@pytest.mark.skip(reason="S06.03 not yet implemented — ATDD red phase")`.

**Why `@pytest.mark.skip` (not `@pytest.mark.xfail`):**
The unimplemented module `client_api.core.usage_gate` would cause an `ImportError` at collection
time. Tests use guarded imports (`try/except ImportError`) to allow pytest to collect the files,
and `@pytest.mark.skip` on each test function to document the red phase without breaking CI
collection.

**Why guarded imports are safe:**
If the import fails, local `None` / empty-dict stubs are set. The individual test functions are
skipped before they can call the stubs, so no `AttributeError` occurs at runtime.

---

## Test Strategy

### AC → Test Level Mapping

| AC | Description | Test Level | File | Priority |
|----|-------------|-----------|------|---------|
| AC1 | `UsageGateContext` dataclass with `user_id`, `user_tier`, `_upgrade_url` fields | Unit | `tests/unit/test_usage_gate.py` | P1 |
| AC2 | Single atomic Lua script `_USAGE_LUA`; GET + conditional INCR + EXPIRE only on first | Unit | `tests/unit/test_usage_gate.py` | P0 |
| AC2 | Atomic race guard — two concurrent requests at limit−1 → exactly one success, one 429 | Integration | `tests/integration/test_usage_gate_concurrency.py` | P0 |
| AC3 | Redis key pattern `user:{user_id}:usage:{feature}:{YYYY-MM}`; EXPIRE = billing period TTL | Unit | `tests/unit/test_usage_gate.py` | P1 |
| AC4 | `FEATURE_TIER_LIMITS` constant: free=0, starter=10, professional=50, enterprise=None | Unit | `tests/unit/test_usage_gate.py` | P1 |
| AC5 | `X-Usage-Remaining` header set on success; Enterprise gets no header | Unit | `tests/unit/test_usage_gate.py` | P1 |
| AC6 | 429 `usage_limit_exceeded` with details (limit, used, resets_at, upgrade_url) + header "0" | Unit | `tests/unit/test_usage_gate.py` | P0 |
| AC7 | `get_usage_gate` is async callable (FastAPI dependency) | Unit | `tests/unit/test_usage_gate.py` | P1 |
| AC8 | fakeredis serial scenarios: Starter 1st/10th/11th, Free 1st, Enterprise, Professional 50th/51st | Unit | `tests/unit/test_usage_gate.py` | P1 |
| AC9 | Concurrency: testcontainers Redis, asyncio.gather, exactly one success + one 429, counter=limit | Integration | `tests/integration/test_usage_gate_concurrency.py` | P0 |
| AC10 | `__all__` exports `UsageGateContext`, `get_usage_gate`, `FEATURE_TIER_LIMITS` | Unit | `tests/unit/test_usage_gate.py` | P1 |

### Risk Coverage

| Risk ID | Score | Mitigation | Tests |
|---------|-------|-----------|-------|
| E06-R-002 | 6 (HIGH) | Atomic Lua script; no separate GET + INCR round-trips | `test_concurrent_requests_at_limit_minus_one`, `test_burst_of_concurrent_requests_respects_limit`, `test_expire_not_reset_on_subsequent_incr`, `test_lua_script_defined_as_module_level_constant` |
| E06-R-010 | 1 (LOW) | fakeredis acceptable for serial; testcontainers for concurrency | E06-P3-004 via `test_starter_sequential_remaining_header` |

---

## Generated Test Files

### File 1: Unit Tests (fakeredis, serial)

**Path:** `eusolicit-app/services/client-api/tests/unit/test_usage_gate.py`
**Status:** 🔴 Created (RED phase — all tests skipped)
**Test count:** 22 test functions

| Test Function | AC | Test ID | Priority |
|--------------|-----|---------|---------|
| `test_feature_tier_limits_has_all_tiers` | AC4 | — | P1 |
| `test_feature_tier_limits_ai_summary_values` | AC4 | — | P1 |
| `test_usage_gate_context_is_dataclass` | AC1 | — | P1 |
| `test_usage_gate_context_fields` | AC1 | — | P1 |
| `test_usage_gate_context_instantiation` | AC1 | — | P1 |
| `test_module_all_exports` | AC10 | — | P1 |
| `test_billing_period_ttl_mid_month` | AC3 | — | P1 |
| `test_billing_period_ttl_december_year_rollover` | AC3 | — | P1 |
| `test_billing_period_ttl_last_second_of_month` | AC3 | — | P1 |
| `test_redis_key_contains_user_id_feature_and_billing_period` | AC3 | — | P1 |
| `test_starter_sequential_remaining_header` | AC5, AC8 | E06-P1-009, E06-P3-004 | P1 |
| `test_starter_first_call_header_is_nine` | AC5, AC8 | E06-P1-009 | P1 |
| `test_starter_tenth_call_remaining_zero` | AC5, AC8 | — | P1 |
| `test_starter_eleventh_call_raises_429` | AC6, AC8 | E06-P0-005 | P0 |
| `test_free_tier_first_call_raises_429` | AC6, AC8 | E06-P0-005 | P0 |
| `test_enterprise_no_metering_no_header_no_exception` | AC4, AC5, AC8 | — | P1 |
| `test_enterprise_multiple_calls_no_exception` | AC4, AC8 | — | P1 |
| `test_professional_50th_call_succeeds_51st_raises` | AC8 | — | P1 |
| `test_expire_set_to_billing_period_ttl` | AC3 | E06-P1-008 | P1 |
| `test_expire_not_reset_on_subsequent_incr` | AC2, AC3 | E06-P1-008 | P1 |
| `test_lua_script_defined_as_module_level_constant` | AC2 | — | P0 |
| `test_get_usage_gate_is_callable` | AC7 | — | P1 |

### File 2: Concurrency Integration Tests (testcontainers Redis)

**Path:** `eusolicit-app/services/client-api/tests/integration/test_usage_gate_concurrency.py`
**Status:** 🔴 Created (RED phase — all tests skipped)
**Test count:** 3 test functions

| Test Function | AC | Test ID | Priority |
|--------------|-----|---------|---------|
| `test_concurrent_requests_at_limit_minus_one` | AC2, AC9 | E06-P0-004 | P0 |
| `test_burst_of_concurrent_requests_respects_limit` | AC2, AC9 | E06-P0-004 (extended) | P0 |
| `test_concurrent_enterprise_requests_no_failures` | AC4, AC9 | — | P1 |

---

## Acceptance Criteria Coverage

| AC | Description | Tests | Coverage |
|----|-------------|-------|---------|
| AC1 | `UsageGateContext` dataclass fields | `test_usage_gate_context_is_dataclass`, `test_usage_gate_context_fields`, `test_usage_gate_context_instantiation` | ✅ Full |
| AC2 | Atomic Lua script `_USAGE_LUA`; GET + conditional INCR; E06-R-002 mitigation | `test_lua_script_defined_as_module_level_constant`, `test_expire_not_reset_on_subsequent_incr`, `test_concurrent_requests_at_limit_minus_one`, `test_burst_of_concurrent_requests_respects_limit` | ✅ Full |
| AC3 | Key pattern `user:{user_id}:usage:{feature}:{YYYY-MM}`; TTL ≤ billing period end | `test_redis_key_contains_user_id_feature_and_billing_period`, `test_expire_set_to_billing_period_ttl`, `test_billing_period_ttl_*` (3 tests) | ✅ Full |
| AC4 | `FEATURE_TIER_LIMITS` constant with correct values for all tiers | `test_feature_tier_limits_has_all_tiers`, `test_feature_tier_limits_ai_summary_values` | ✅ Full |
| AC5 | `X-Usage-Remaining` on success; no header for Enterprise | `test_starter_sequential_remaining_header`, `test_starter_first_call_header_is_nine`, `test_starter_tenth_call_remaining_zero`, `test_enterprise_no_metering_no_header_no_exception` | ✅ Full |
| AC6 | 429 `usage_limit_exceeded` body schema (limit, used, resets_at, upgrade_url) + header | `test_starter_eleventh_call_raises_429`, `test_free_tier_first_call_raises_429` | ✅ Full |
| AC7 | `get_usage_gate` is async FastAPI dependency returning `UsageGateContext` | `test_get_usage_gate_is_callable` | ✅ Structural |
| AC8 | fakeredis serial scenarios: all 7 specified sub-scenarios | 10+ unit tests across tiers | ✅ Full |
| AC9 | testcontainers concurrency: pre-set to limit−1, asyncio.gather, one success + one 429, counter=limit | `test_concurrent_requests_at_limit_minus_one` | ✅ Full |
| AC10 | `__all__` exports | `test_module_all_exports` | ✅ Full |

**Coverage score: 10/10 ACs covered**

---

## Test Design Traceability

| Test ID | Priority | Scenario | AC | Test Function |
|---------|----------|----------|----|--------------|
| E06-P0-004 | P0 | Concurrent race at limit−1 → exactly one 200, one 429 (testcontainers Redis) | AC2, AC9 | `test_concurrent_requests_at_limit_minus_one` |
| E06-P0-005 | P0 | 429 body schema: error, limit, used, resets_at, upgrade_url; header X-Usage-Remaining: 0 | AC6 | `test_starter_eleventh_call_raises_429`, `test_free_tier_first_call_raises_429` |
| E06-P1-008 | P1 | INCR sets EXPIRE ≤ billing period TTL | AC3 | `test_expire_set_to_billing_period_ttl` |
| E06-P1-009 | P1 | X-Usage-Remaining decrements correctly: 9, 8, 7 for Starter | AC5 | `test_starter_sequential_remaining_header` |
| E06-P3-004 | P3 | fakeredis serial parity — sequential counter values correct | AC2, AC5 | `test_starter_sequential_remaining_header` |

---

## Missing Dependencies (Action Required Before Green Phase)

The following packages are specified in the story but are **not** in `pyproject.toml` dev extras:

| Package | Required By | Add To |
|---------|-------------|--------|
| `fakeredis[aioredis]>=2.21` | `tests/unit/test_usage_gate.py` | `[project.optional-dependencies] dev` |
| `testcontainers[redis]>=4.7` | `tests/integration/test_usage_gate_concurrency.py` | `[project.optional-dependencies] dev` |

Add before running tests:
```toml
[project.optional-dependencies]
dev = [
    ...existing deps...,
    "fakeredis[aioredis]>=2.21",
    "testcontainers[redis]>=4.7",
]
```

---

## Implementation Notes for Dev Agent

These notes capture test-relevant implementation constraints from the story:

1. **Lua script must be module-level constant `_USAGE_LUA`** — not a function-local string. Tests
   assert `hasattr(mod, "_USAGE_LUA")` and that it contains GET, INCR, EXPIRE.

2. **`redis.eval()` call signature** — `await redis.eval(script, 1, key, str(limit), str(ttl))`.
   ARGV values must be strings. Result is `[count, exceeded_flag]`; always cast with `int()`.

3. **Free tier (limit=0) semantics** — the Lua script receives `ARGV[1] = "0"`. With `current=0`
   and `current >= 0 == True`, it returns `{0, 1}` (exceeded). `check_and_increment` sees
   `exceeded=1`, sets `X-Usage-Remaining: 0`, and raises 429 with `used=0, limit=0`.

4. **Enterprise bypass** — `FEATURE_TIER_LIMITS["ai_summary"]["enterprise"] = None`. When `limit
   is None`, return early with no Redis call, no header. Tests assert no keys written.

5. **EXPIRE guard** — Lua script must only call EXPIRE when `new_count == 1` (first creation).
   Subsequent INCRs must not reset TTL. `test_expire_not_reset_on_subsequent_incr` verifies this.

6. **`_billing_period_ttl()` must be exported** — tests import it directly for TTL assertions.
   It is a private helper but must be importable from `client_api.core.usage_gate`.

7. **AppException signature** — `AppException(message, error=..., status_code=..., details={...})`.
   Assert `exc.error`, `exc.status_code`, `exc.details["limit"]` (NOT `exc.details` top-level keys).

---

## 🔴 TDD Red Phase Verification

Before implementing S06.03, run:

```bash
cd eusolicit-app/services/client-api
pip install fakeredis[aioredis] testcontainers[redis]
pytest tests/unit/test_usage_gate.py tests/integration/test_usage_gate_concurrency.py -v
```

**Expected output (red phase):** All tests should show `SKIPPED` — not `ERROR` or `FAILED`.
If any test shows `ERROR`, there is a problem with the guarded import or fixture setup.

---

## ✅ Next Steps (TDD Green Phase)

After implementing `core/usage_gate.py` (Story Task 1.1):

1. **Add missing dependencies** to `pyproject.toml`:
   - `fakeredis[aioredis]>=2.21`
   - `testcontainers[redis]>=4.7`

2. **Remove `@pytest.mark.skip`** from all test functions in both files.

3. **Run unit tests** (fast, fakeredis, no Docker required):
   ```bash
   pytest tests/unit/test_usage_gate.py -v
   ```

4. **Run concurrency integration tests** (requires Docker):
   ```bash
   pytest tests/integration/test_usage_gate_concurrency.py -v -m "not skip"
   ```

5. **Verify all tests PASS** (green phase). If any fail:
   - Implementation bug → fix `core/usage_gate.py`
   - Test assumption wrong → update test and document the deviation in story Dev Agent Record

6. **CI coverage target**: `core/usage_gate.py` must reach ≥90% line coverage (security-critical path per E06 test design exit criteria).

---

## Summary Statistics

| Metric | Value |
|--------|-------|
| TDD phase | 🔴 RED |
| Stack detected | `backend` |
| Generation mode | AI generation (sequential) |
| Total test functions | 25 |
| Unit tests | 22 (in `tests/unit/test_usage_gate.py`) |
| Integration tests | 3 (in `tests/integration/test_usage_gate_concurrency.py`) |
| P0 tests | 5 (`test_starter_eleventh_call_raises_429`, `test_free_tier_first_call_raises_429`, `test_lua_script_defined_as_module_level_constant`, `test_concurrent_requests_at_limit_minus_one`, `test_burst_of_concurrent_requests_respects_limit`) |
| P1 tests | 20 |
| ACs covered | 10/10 (100%) |
| Epic test IDs addressed | E06-P0-004, E06-P0-005, E06-P1-008, E06-P1-009, E06-P3-004 |
| Risk mitigations verified | E06-R-002 (score 6, HIGH), E06-R-010 (score 1, LOW) |
| Missing dev dependencies | `fakeredis[aioredis]`, `testcontainers[redis]` |
| All tests skipped (red phase) | ✅ Yes |

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Story:** 6.3 Usage Metering Middleware (Redis)
