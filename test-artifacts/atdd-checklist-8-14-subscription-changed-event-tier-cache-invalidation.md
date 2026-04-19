---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-19'
story: 8-14-subscription-changed-event-tier-cache-invalidation
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/8-14-subscription-changed-event-tier-cache-invalidation.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - eusolicit-app/services/client-api/tests/unit/test_tier_gate.py
  - eusolicit-app/services/client-api/tests/integration/test_subscription_usage.py
  - eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/consumer.py
  - eusolicit-app/services/client-api/src/client_api/dependencies.py
  - eusolicit-app/services/client-api/tests/conftest.py
---

# ATDD Checklist: Story 8.14 — Subscription Changed Event & Tier Cache Invalidation

## TDD Red Phase (Current State)

🔴 **RED PHASE ACTIVE** — All tests are decorated with `@pytest.mark.skip`.
Failing tests define the contract. Remove `skip` markers after implementation is complete.

---

## Stack Detection

- **Detected Stack**: `fullstack` (Next.js 14 + FastAPI microservices)
- **Story Scope**: Pure backend — no new API endpoints, no UI changes
- **Generation Mode**: AI generation (no browser recording needed)
- **Test Framework**: pytest + AsyncMock (backend unit/integration)

---

## Generation Summary

| File | Type | Tests | Status |
|---|---|---|---|
| `tests/unit/test_tier_cache.py` | Unit | 11 | 🔴 RED (all skipped) |
| `tests/unit/test_tier_cache_consumer.py` | Unit | 7 | 🔴 RED (all skipped) |
| `tests/integration/test_subscription_changed_cache_invalidation.py` | Integration | 3 | 🔴 RED (all skipped) |
| **Total** | | **21** | **0 passing (RED phase)** |

---

## Acceptance Criteria Coverage

### AC1 — Redis Tier Cache Service (`tier_cache.py`)

| Test | File | Priority | Status |
|---|---|---|---|
| `test_tier_cache_prefix_constant` — TIER_CACHE_PREFIX == "tier" | `test_tier_cache.py` | P1 | 🔴 RED |
| `test_tier_cache_key_format` — returns "tier:{uuid}" | `test_tier_cache.py` | P1 | 🔴 RED |
| `test_get_cached_tier_hit_returns_decoded_string` — bytes → str | `test_tier_cache.py` | P1 | 🔴 RED |
| `test_get_cached_tier_hit_with_str_value` — str passthrough | `test_tier_cache.py` | P2 | 🔴 RED |
| `test_get_cached_tier_miss_returns_none` — None on key miss | `test_tier_cache.py` | P1 | 🔴 RED |
| `test_set_cached_tier_calls_set_with_correct_key_and_ttl` — ex=60 | `test_tier_cache.py` | P1 | 🔴 RED |
| `test_set_cached_tier_respects_custom_ttl` — custom ex= | `test_tier_cache.py` | P2 | 🔴 RED |
| `test_set_cached_tier_redis_error_never_raises` — fire-and-forget | `test_tier_cache.py` | P1 | 🔴 RED |
| `test_invalidate_cached_tier_calls_delete_with_correct_key` — DELETE | `test_tier_cache.py` | P1 | 🔴 RED |
| `test_invalidate_cached_tier_redis_error_never_raises` — fire-and-forget | `test_tier_cache.py` | P1 | 🔴 RED |

**Critical design note (AC1):** Invalidation uses `DELETE` (not `SET tier free`) to avoid a 60s window where upgraded users see stale "free" tier.

---

### AC2 — `opportunity_tier_gate.py` Uses Cache

Covered by integration test 8.14-INT-003 (read-through path) and the DB fallback sanity test 8.14-INT-002. No standalone unit tests: AC2 changes behaviour of an existing function (add cache calls) — the integration tests verify the end-to-end contract.

**Developer note:** After implementing Task 2, also verify that `test_tier_gate_context.py` still passes with the new cache layer.

---

### AC3 — `tier_gate.py` Uses Cache

Covered by integration tests (8.14-INT-002 and 8.14-INT-003 verify the read-through pattern). Unit coverage is ensured because `_get_tier_with_cache` delegates to `get_cached_tier` / `set_cached_tier` which are fully unit-tested in `test_tier_cache.py`.

**Developer note:** After implementing Task 3, verify all existing tier gate tests in `test_tier_gate.py`, `test_tier_gate_trial.py`, `test_tier_gate_context.py` still pass. Existing mocks need `get_redis_client` patched.

---

### AC4 — `subscription.changed` Consumer Background Task

| Test | File | Priority | Status |
|---|---|---|---|
| `test_consumer_invalidates_tier_cache_on_event` — invalidate called with correct company_id | `test_tier_cache_consumer.py` | P1 | 🔴 RED |
| `test_consumer_acks_message_after_invalidation` — ACK after invalidate | `test_tier_cache_consumer.py` | P1 | 🔴 RED |
| `test_consumer_processes_multiple_events_per_batch` — up to count=10 | `test_tier_cache_consumer.py` | P2 | 🔴 RED |
| `test_consumer_skips_event_with_missing_company_id` — no invalidate, ACK | `test_tier_cache_consumer.py` | P1 | 🔴 RED |
| `test_consumer_skips_event_with_empty_string_company_id` — empty treated as missing | `test_tier_cache_consumer.py` | P2 | 🔴 RED |
| `test_consumer_continues_after_invalidate_error` — loop survives exception | `test_tier_cache_consumer.py` | P1 | 🔴 RED |
| `test_consumer_group_creation_silences_busygroup_error` — BUSYGROUP not fatal | `test_tier_cache_consumer.py` | P2 | 🔴 RED |
| `test_consumer_group_creation_mkstream_on_first_create` — mkstream=True | `test_tier_cache_consumer.py` | P2 | 🔴 RED |

---

### AC5 — Consumer Registered in Lifespan

Not tested with dedicated unit tests (testing asyncio lifespan internals is fragile and adds low value). Verified indirectly by:
- 8.14-INT-001 running the consumer via `create_task`
- Smoke tests (existing) verifying the app starts without errors after Task 5

---

### AC6 — DB Fallback Guarantees Consistency

| Test | File | Priority | Status |
|---|---|---|---|
| `test_get_cached_tier_redis_error_returns_none_never_raises` — RedisError → None | `test_tier_cache.py` | P1 | 🔴 RED |
| `test_get_cached_tier_generic_exception_returns_none_never_raises` — any exc → None | `test_tier_cache.py` | P2 | 🔴 RED |
| `test_tier_gate_falls_back_to_db_on_redis_error` (8.14-INT-002) — integration | `test_subscription_changed_cache_invalidation.py` | P1 | 🔴 RED |

---

### AC7 — Tests (Defined by This Story)

This checklist satisfies AC7. The tests specified in the story map to:

| Story AC7 Requirement | Test Location | Status |
|---|---|---|
| `test_tier_cache.py`: hit/miss/error; TTL; delete; key format | `tests/unit/test_tier_cache.py` | 🔴 RED |
| `test_tier_cache_consumer.py`: event→invalidate; missing id; exception survives | `tests/unit/test_tier_cache_consumer.py` | 🔴 RED |
| 8.14-INT-001 (P1, R-006): publish→evict→DB read | `tests/integration/test_subscription_changed_cache_invalidation.py` | 🔴 RED |

---

## Test Design Coverage (from Epic 8 Test Design)

| Test Design ID | Priority | Risk | Scenario | Coverage File |
|---|---|---|---|---|
| **8.14-INT-001** | P1 | R-006 | `subscription.changed` invalidates tier cache; next request reads from DB | `test_subscription_changed_cache_invalidation.py::test_subscription_changed_invalidates_tier_cache_end_to_end` |
| *8.14-INT-002* | P1 | R-006 | DB fallback on Redis error (AC6 integration) | `test_subscription_changed_cache_invalidation.py::test_tier_gate_falls_back_to_db_on_redis_error` |
| *8.14-INT-003* | P2 | R-006 | Read-through: cache miss → DB → repopulate with TTL | `test_subscription_changed_cache_invalidation.py::test_cache_miss_triggers_db_read_and_repopulation` |

INT-002 and INT-003 are extensions beyond the test design; they add AC6 and AC2/AC3 integration coverage respectively.

---

## Test Files Generated (RED Phase)

```
eusolicit-app/services/client-api/
└── tests/
    ├── unit/
    │   ├── test_tier_cache.py              ← NEW (AC1, AC6) — 11 tests, all @skip
    │   └── test_tier_cache_consumer.py     ← NEW (AC4) — 7 tests, all @skip
    └── integration/
        └── test_subscription_changed_cache_invalidation.py  ← NEW (AC2,AC3,AC4,AC6) — 3 tests, all @skip
```

---

## TDD Compliance Verification

- ✅ All 21 tests use `@pytest.mark.skip(reason=SKIP_REASON)`
- ✅ All tests assert **expected behaviour** (not placeholder `assert True`)
- ✅ All tests will **fail** when `skip` is removed before implementation (import errors)
- ✅ No passing tests generated in RED phase
- ✅ `SKIP_REASON` strings identify the exact Task to implement

---

## Next Steps (TDD Green Phase)

After implementing all Tasks 1–5:

1. **Remove `@pytest.mark.skip`** from all test functions
2. **Run unit tests**: `make test-unit SVC=client-api` — verify all 18 unit tests pass
3. **Run integration tests**: `make test-integration SVC=client-api` — verify 3 integration tests pass
4. **Verify existing tests still pass**: `make test-service SVC=client-api`
   - `test_tier_gate.py`, `test_tier_gate_trial.py`, `test_tier_gate_context.py`
     need `get_redis_client` patched in their mocks after Task 2 and 3
5. **Coverage check**: `make coverage` — must remain ≥80%

---

## Implementation Guidance for Developer

### Task 1 (AC1): Create `tier_cache.py`

Implement at `services/client-api/src/client_api/core/tier_cache.py`:
- `TIER_CACHE_PREFIX = "tier"`
- `tier_cache_key(company_id: UUID) -> str` → `f"tier:{company_id}"`
- `get_cached_tier(redis_client, company_id) -> str | None` — decode bytes, catch all exceptions → return None
- `set_cached_tier(redis_client, company_id, tier, ttl=60) -> None` — `redis.set(key, tier, ex=ttl)`, never raises
- `invalidate_cached_tier(redis_client, company_id) -> None` — `redis.delete(key)`, never raises

### Task 4 (AC4): Create `tier_cache_consumer.py`

Implement at `services/client-api/src/client_api/services/tier_cache_consumer.py`:
- Constants: `_STREAM = "subscription.changed"`, `_GROUP = "client-api:tier-cache-invalidator"`, `_CONSUMER = "client-api-invalidator-1"`, `_BLOCK_MS = 2000`, `_COUNT = 10`
- `_ensure_consumer_group(redis_client)` — XGROUP CREATE with mkstream=True; BUSYGROUP silently ignored
- `run_tier_cache_invalidator(app: FastAPI) → None` — infinite loop; CancelledError re-raised; all other exceptions logged at WARNING and loop continues

### Existing Test Impact (Tasks 2 & 3)

After updating `tier_gate.py` and `opportunity_tier_gate.py` to call `get_redis_client()`, existing unit tests that mock the session must also patch `get_redis_client`:

```python
with patch("client_api.core.tier_gate.get_redis_client") as mock_get_redis:
    mock_get_redis.return_value = AsyncMock()
    # ... existing test logic ...
```

---

## Key Risk Mitigation Summary

**R-006 (TECH):** Cache invalidation failure on tier change leaves stale tier in middleware, granting unauthorized access to premium features.

**Mitigation verified by this ATDD suite:**
1. Consumer receives `subscription.changed` → calls `invalidate_cached_tier` (DELETE) → no stale bytes remain
2. Next tier gate call → `get_cached_tier` returns None (cache miss) → DB query always executes → fresh accurate tier
3. Redis error → `get_cached_tier` returns None → DB fallback → no bypass possible
4. Cache TTL=60s → even without invalidation, maximum stale window is 60s (bounded risk)
