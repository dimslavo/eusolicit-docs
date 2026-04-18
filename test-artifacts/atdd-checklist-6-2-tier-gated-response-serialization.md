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
storyId: 6-2-tier-gated-response-serialization
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-2-tier-gated-response-serialization.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/src/client_api/schemas/opportunities.py
  - eusolicit-app/services/client-api/src/client_api/core/tier_gate.py
  - eusolicit-app/packages/eusolicit-common/src/eusolicit_common/exceptions.py
  - eusolicit-app/services/client-api/src/client_api/config.py
  - eusolicit-app/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.2 — Tier-Gated Response Serialization

**Date:** 2026-04-17
**Author:** TEA Master Test Architect (bmad-testarch-atdd)
**TDD Phase:** 🔴 RED (failing tests — awaiting implementation)
**Story:** eusolicit-docs/implementation-artifacts/6-2-tier-gated-response-serialization.md
**Epic Test Design:** eusolicit-docs/test-artifacts/test-design-epic-06.md

---

## Preflight Summary

| Check | Status | Notes |
|-------|--------|-------|
| Stack detected | ✅ `backend` | Python/FastAPI/pytest; no browser testing |
| Story has clear acceptance criteria | ✅ 10 ACs | AC1–AC10 fully specified |
| Test framework configured | ✅ pytest + pytest-asyncio | conftest.py with app/test_client fixtures |
| `tests/unit/` directory | ✅ exists | `__init__.py` present |
| `tests/api/` directory | ✅ exists | conftest.py not separate; inherits from tests/conftest.py |
| `OpportunityFreeResponse` | ❌ NOT IMPLEMENTED | Defined in story Task 1.1 |
| `OpportunityFullResponse` | ❌ NOT IMPLEMENTED | Defined in story Task 1.1 |
| `opportunity_tier_gate.py` | ❌ NOT IMPLEMENTED | Defined in story Task 2.1 |
| `GET /search` TierGate integration | ❌ NOT IMPLEMENTED | Defined in story Task 3.1 |
| `eusolicit_common.exceptions.AppException` | ✅ EXISTS | Verified in exceptions.py |

---

## 🔴 TDD Red Phase: Failing Tests Generated

All test functions are decorated with `@pytest.mark.skip(reason="S06.02 not yet implemented — ATDD red phase")`.

**Why `@pytest.mark.skip` (not `@pytest.mark.xfail`):**
The unimplemented modules (`client_api.core.opportunity_tier_gate`, `OpportunityFreeResponse`,
`OpportunityFullResponse`) would cause an ImportError at collection time. The tests use guarded
imports (try/except) to allow pytest to collect the file, and `@pytest.mark.skip` on each test
function to document the red phase without breaking CI collection.

---

## Generated Test Files

### File 1: Unit Tests

**Path:** `services/client-api/tests/unit/test_tier_gate_context.py`
**Status:** 🔴 Created (RED phase — all tests skipped)
**Test count:** 23 test functions

| Test Function | AC | Test ID | Priority |
|--------------|-----|---------|---------|
| `test_free_response_contains_only_allowed_fields` | AC1 | E06-P1-005 | P1 |
| `test_free_response_serialized_field_values_match_source_item` | AC1 | — | P1 |
| `test_free_response_has_exactly_six_model_fields` | AC1 | E06-P1-005 | P1 |
| `test_full_response_includes_all_fields` | AC2 | E06-P1-006 | P1 |
| `test_full_response_jsonb_fields_are_dicts` | AC2 | E06-P1-006 | P1 |
| `test_serialize_item_returns_correct_model_for_each_tier` (×4) | AC5, AC6 | E06-P1-007 | P1 |
| `test_starter_bulgaria_within_budget_is_in_scope` | AC4 | — | P0 |
| `test_starter_france_is_out_of_scope_and_check_scope_raises_403` | AC4 | E06-P0-003 | P0 |
| `test_starter_over_budget_is_out_of_scope` | AC4 | — | P0 |
| `test_starter_over_budget_check_scope_raises_403` | AC4 | — | P0 |
| `test_starter_budget_max_none_is_treated_as_in_scope` | AC4 | — | P1 (edge case) |
| `test_starter_region_check_is_case_insensitive` | AC4 | — | P1 (edge case) |
| `test_starter_null_region_is_out_of_scope` | AC4 | — | P1 (edge case) |
| `test_professional_any_region_under_budget_is_in_scope` (×6) | AC4 | — | P1 |
| `test_professional_over_budget_is_out_of_scope` | AC4 | — | P1 |
| `test_professional_at_exact_budget_limit_is_in_scope` | AC4 | — | P1 (boundary) |
| `test_professional_over_budget_check_scope_raises_403` | AC4 | — | P1 |
| `test_enterprise_always_in_scope` (×5) | AC4 | — | P1 |
| `test_free_tier_is_always_in_scope` (×3) | AC4 | — | P1 |
| `test_is_in_scope_equals_inverse_of_check_scope` (×7) | AC5 | — | P1 |
| `test_check_scope_exception_contains_non_empty_upgrade_url` | AC10 | — | P1 |
| `test_check_scope_upgrade_url_includes_billing_path` | AC10 | — | P1 |

### File 2: Integration Tests

**Path:** `services/client-api/tests/api/test_opportunity_tier_gate.py`
**Status:** 🔴 Created (RED phase — all tests skipped)
**Test count:** 7 test functions

| Test Function | AC | Test ID | Priority |
|--------------|-----|---------|---------|
| `test_free_tier_search_returns_only_six_fields` | AC1, AC7 | E06-P0-002 | P0 |
| `test_free_tier_search_returns_200_with_results_array` | AC7 | — | P0 |
| `test_starter_tier_filters_out_france_opportunity` | AC4, AC7 | E06-P0-003 | P0 |
| `test_starter_tier_returns_status_200_not_403_when_scope_filters` | AC7 | — | P0 |
| `test_starter_tier_receives_full_response_for_bulgaria_opportunity` | AC5, AC6 | E06-P1-007 | P1 |
| `test_starter_tier_shows_bulgaria_but_not_france_in_combined_region_search` | AC4, AC7 | E06-P0-003 | P0 |
| `test_free_tier_search_returns_200_with_results_array` | AC7 | — | P0 |

---

## Acceptance Criteria Coverage

| AC | Description | Covered By | Priority | Status |
|----|-------------|-----------|---------|--------|
| AC1 | `OpportunityFreeResponse` has exactly 6 fields; all other fields absent | Unit: 3 tests + Integration: 1 test | P0/P1 | 🔴 RED |
| AC2 | `OpportunityFullResponse` has all `OpportunitySearchItem` fields; JSONB as dict | Unit: 2 tests | P1 | 🔴 RED |
| AC3 | `OpportunityTierGate` is an async FastAPI dependency | Unit: serialize_item tests indirectly verify context construction | P1 | ⚠️ Partial — full DI injection tested via integration override |
| AC4 | `check_scope` raises `AppException(error="tier_limit", 403)` for out-of-scope paid users | Unit: 6 tests (parametrized covers 7 scenarios) + Integration: 3 tests | P0 | 🔴 RED |
| AC5 | `is_in_scope` = inverse of `check_scope`; never raises | Unit: parametrized test covers 7 tier/scope combinations | P1 | 🔴 RED |
| AC6 | `serialize_item` returns correct model per tier | Unit: parametrized for free/starter/pro/enterprise | P1 | 🔴 RED |
| AC7 | `GET /search` applies tier-gated serialization; out-of-scope silently removed | Integration: 5 tests | P0 | 🔴 RED |
| AC8 | Unit test file at `tests/unit/test_tier_gate_context.py` | ✅ File created | P0 | 🔴 RED |
| AC9 | Integration tests at `tests/api/test_opportunity_tier_gate.py` | ✅ File created | P0 | 🔴 RED |
| AC10 | `upgrade_url` is non-empty from `frontend_url + "/billing/upgrade"` | Unit: 2 tests | P1 | 🔴 RED |

**Coverage summary:** 10/10 ACs covered (AC3 partially via integration override pattern).

---

## Priority Coverage Summary

| Priority | Count | Test IDs |
|----------|-------|---------|
| **P0** | 9 tests | E06-P0-002 (1), E06-P0-003 (3), AC4 unit scope tests (4), AC7 listing 200 tests (1) |
| **P1** | 21 tests | E06-P1-005 (3), E06-P1-006 (2), E06-P1-007 (5), AC4 edge cases (7), AC5 inverse (1), AC10 (2), AC7 full response (1) |
| **Total** | **30 test functions** | (parametrized markers expand to additional runs) |

---

## Implementation Checklist (for Developer)

Complete these tasks before removing `@pytest.mark.skip`:

### Task 1: Response Models (AC1, AC2)

- [ ] Add `OpportunityFreeResponse` to `services/client-api/src/client_api/schemas/opportunities.py`
  - [ ] Exactly 6 fields: `id`, `title`, `deadline`, `region`, `opportunity_type`, `status`
  - [ ] No other fields declared (not even as `Optional[...] = None`)
  - [ ] `model_config = ConfigDict(from_attributes=True, populate_by_name=True)`
- [ ] Add `OpportunityFullResponse` to the same file
  - [ ] All `OpportunitySearchItem` fields
  - [ ] `evaluation_criteria`, `mandatory_documents`, `relevance_scores` typed as `dict | None`
- [ ] Export both models from `services/client-api/src/client_api/schemas/__init__.py`

### Task 2: Dependency (AC3, AC4, AC5, AC6, AC10)

- [ ] Create `services/client-api/src/client_api/core/opportunity_tier_gate.py`
  - [ ] `OpportunityTierGateContext` dataclass with `user_tier` and `_upgrade_url`
  - [ ] `serialize_item(item) -> OpportunityFreeResponse | OpportunityFullResponse`
  - [ ] `is_in_scope(item) -> bool` (never raises)
  - [ ] `check_scope(item) -> None` (raises `AppException(error="tier_limit", 403)`)
  - [ ] Starter: region == "Bulgaria" (case-insensitive) AND budget_max ≤ 500_000
  - [ ] Professional: budget_max ≤ 5_000_000; no region restriction
  - [ ] Enterprise: no restrictions (always in scope)
  - [ ] Free: always in scope; `check_scope` never raises
  - [ ] `budget_max=None` treated as in-scope (no evidence of over-budget)
  - [ ] `upgrade_url` = `get_settings().frontend_url.rstrip("/") + "/billing/upgrade"`
  - [ ] **CRITICAL**: Use `AppException(error="tier_limit")` — NOT `ForbiddenError`
- [ ] `get_opportunity_tier_gate` async dependency queries `client.subscriptions`
  - [ ] Falls back to `"free"` when no active subscription or unknown plan

### Task 3: Search Endpoint Integration (AC7)

- [ ] Update `services/client-api/src/client_api/api/v1/opportunities.py`
  - [ ] Add `Depends(get_opportunity_tier_gate)` to `search_opportunities` handler
  - [ ] Apply `is_in_scope` + `serialize_item` loop to every result item
  - [ ] Out-of-scope items silently omitted (not 403'd on listing)
  - [ ] Return `dict` with `results: list[dict]` (avoid response_model type union issues)
  - [ ] `total_count` reflects unfiltered count from DB

---

## Green Phase Instructions

Once all tasks above are complete:

1. **Remove `@pytest.mark.skip`** from every test function in both files
2. **Remove guarded imports** (the `try/except ImportError` blocks at module top)
3. **Run unit tests:**
   ```bash
   cd eusolicit-app/services/client-api
   python -m pytest tests/unit/test_tier_gate_context.py -v
   ```
4. **Run integration tests:**
   ```bash
   python -m pytest tests/api/test_opportunity_tier_gate.py -v
   ```
5. **Verify all tests PASS** (green phase)
6. If any tests fail: either fix the implementation (feature bug) or fix the test (spec mismatch — escalate to story owner)
7. Commit passing tests

---

## Key Implementation Constraints (Critical)

### 1. `AppException` vs `ForbiddenError`

> **CRITICAL:** `check_scope` MUST raise `AppException(error="tier_limit")`, not `ForbiddenError`.

`ForbiddenError` hardcodes `error="forbidden"`. The test `test_starter_france_is_out_of_scope_and_check_scope_raises_403` asserts `exc.error == "tier_limit"`. Any use of `ForbiddenError` will fail this assertion.

### 2. `OpportunityFreeResponse` — 6 Fields Only, No Nulls

The model must have **only** the 6 declared fields. The test `test_free_response_has_exactly_six_model_fields` asserts:
```python
assert set(OpportunityFreeResponse.model_fields.keys()) == {"id", "title", "deadline", "region", "opportunity_type", "status"}
```
Adding `budget_min: Decimal | None = None` would make this test fail.

### 3. `budget_max=None` is In-Scope for Starter

The implementation must treat `None` as "no declared upper bound → in-scope":
```python
budget_ok = item.budget_max is None or item.budget_max <= STARTER_BUDGET_MAX
```
The test `test_starter_budget_max_none_is_treated_as_in_scope` verifies this.

### 4. Region Check is Case-Insensitive

```python
region_ok = item.region is not None and item.region.lower() == "bulgaria"
```
The test `test_starter_region_check_is_case_insensitive` verifies "BULGARIA" → in-scope.

### 5. `is_in_scope` Never Raises

`is_in_scope` must catch its own logic exceptions and return `False` rather than re-raising.
The test `test_is_in_scope_equals_inverse_of_check_scope` calls `is_in_scope` on out-of-scope
items and expects `False`, not an exception.

---

## Risk Traceability

| Risk ID | Mitigated By | Tests |
|---------|-------------|-------|
| E06-R-001 (TierGate bypass) | AC1 field exclusion enforcement | E06-P0-002 (integration), E06-P1-005 (unit) |
| E06-R-001 (scope bypass) | AC4 scope enforcement | E06-P0-003 (integration + unit) |

---

## Execution Strategy

| When | Tests | Expected Duration |
|------|-------|-----------------|
| Every PR (post-implementation) | Unit tests (23 functions) | < 5 seconds |
| Every PR (post-implementation) | Integration tests (7 functions) | ~10–30 seconds (DB seeding) |

---

**Generated by:** BMad TEA Agent — Master Test Architect (bmad-testarch-atdd)
**Story:** eusolicit-docs/implementation-artifacts/6-2-tier-gated-response-serialization.md
**Epic Test Design:** eusolicit-docs/test-artifacts/test-design-epic-06.md
