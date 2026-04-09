---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-09'
workflowType: 'atdd'
storyId: '11-5-budget-builder-agent-integration'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/11-5-budget-builder-agent-integration.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-11.md'
  - 'eusolicit-app/services/client-api/tests/api/test_grant_eligibility.py'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
---

# ATDD Checklist: Story 11.5 — Budget Builder Agent Integration

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Tests written; feature not yet implemented
**Story File:** `eusolicit-docs/implementation-artifacts/11-5-budget-builder-agent-integration.md`
**Test File:** `eusolicit-app/services/client-api/tests/api/test_budget_builder.py`

---

## Preflight Summary (Step 1)

### Stack Detection

| Check | Result |
|-------|--------|
| `pyproject.toml` found | ✅ `eusolicit-app/services/client-api/pyproject.toml` |
| `playwright.config.ts` (top-level) | ✅ Present (frontend E2E stack) |
| Backend indicators (client-api service) | ✅ `pyproject.toml` + `tests/conftest.py` with pytest-asyncio |
| **Detected stack** | **`backend`** (this story is a client-api service test — no browser involvement) |
| Test framework | `pytest` + `pytest-asyncio` |
| AI Gateway mock library | `respx>=0.21` (in `[project.optional-dependencies] dev`) |

### Prerequisites Check

| Requirement | Status |
|-------------|--------|
| Story approved with clear acceptance criteria (8 ACs) | ✅ |
| `conftest.py` with session factory, RSA fixtures, Redis fixtures | ✅ |
| `AiGatewayClient` implemented (Story 11.3) | ✅ `src/client_api/services/ai_gateway_client.py` |
| `ClientApiSettings.aigw_base_url` / `aigw_timeout_seconds` | ✅ `src/client_api/config.py` |
| `/grants` router registered in `main.py` (Story 11.4) | ✅ (no change needed to main.py — AC8 note) |
| Reference test pattern | ✅ `tests/api/test_grant_eligibility.py` (S11.04) |
| `respx>=0.21` in dev dependencies | ✅ |

### TEA Config Flags (from `_bmad/bmm/config.yaml`)

| Flag | Value | Effect |
|------|-------|--------|
| `test_stack_type` | not set → `auto` → **`backend`** | Backend test patterns only |
| `tea_use_playwright_utils` | not set → disabled | No Playwright Utils loading |
| `tea_use_pactjs_utils` | not set → disabled | No Pact.js CDC tests |
| `tea_pact_mcp` | not set → `none` | No Pact MCP |
| `tea_browser_automation` | not set → disabled | No browser automation |

### Knowledge Fragments Loaded

**Core (always):**
- `data-factories.md` — fixture patterns, teardown via `session.rollback()`
- `test-quality.md` — definition of done, isolation rules
- `test-healing-patterns.md` — failure mode mitigation
- `test-levels-framework.md` — backend level selection guidance
- `test-priorities-matrix.md` — P0–P3 assignment criteria

**Backend-specific:**
- `ci-burn-in.md` — CI execution strategy

---

## Generation Mode (Step 2)

**Mode selected: AI Generation** (backend stack — no browser recording needed)

Acceptance criteria are clear and detailed. All scenarios are standard API/service patterns (POST endpoint, AI Gateway mock with `respx`, arithmetic validation). Full AI generation from story AC specification and reference test pattern (`test_grant_eligibility.py`).

---

## Test Strategy (Step 3)

### Acceptance Criteria → Test Scenario Mapping

| AC | Description | Scenarios | Level | Priority | Epic Test IDs |
|----|-------------|-----------|-------|----------|---------------|
| **AC1** | POST accepts JSON body; returns 200 + BudgetBuilderResponse | Happy path solo; response structure; required field validation | Integration/API | P0 | E11-P0-009 |
| **AC2** | Agent payload: all required fields + X-Caller-Service + 30s timeout; optional params forwarded | Payload fields; header captured; optional params null when absent; optional params forwarded | Integration/API | P0/P2 | E11-P0-009, E11-P2-001 |
| **AC3** | Response parsed: cost_categories, overhead_calculation, co_financing_split, per_partner_breakdown | Field presence + types on each sub-object; graceful empty agent response | Integration/API | P1 | — |
| **AC4** | Budget arithmetic validation before serving caller | Valid → 200; inconsistent line items → 422; inconsistent co-financing → 422; inconsistent overhead → 422 | Integration/API | P0 | E11-P0-005 |
| **AC5** | Per-partner breakdown required when consortium_size > 1 | Consortium → per_partner present; missing when consortium → 422; solo → null accepted | Integration/API | P1 | E11-P1-002 |
| **AC6** | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE | ReadTimeout → 503; HTTP 500 → 503; HTTP 503 → 503 | Integration/API | P0 | E11-P0-008, E11-P0-010 |
| **AC7** | Unauthenticated → 401; all roles permitted | No token → 401; read_only → 200 | Integration/API | P0 | — |
| **AC8** | Stateless: no DB writes | Implicit: fixture only needs JWT — no session required by service; verified via no SQL insert assertions | Integration/API | P2 | — |

### Test Level Selection (backend stack)

- **Integration** (function-scope, real DB session via rollback): all tests
- **No E2E** (backend stack — no browser required)
- **No Unit** (arithmetic validation logic is thin; integration tests provide better coverage at API contract level)

### Priority Assignment

| Priority | Tests | Rationale |
|----------|-------|-----------|
| P0 | AC1 happy path, AC2 payload/header, AC4 arithmetic validation, AC6 error handling, AC7 auth | Core functionality + E11-R-001 (AI Gateway error handling) + E11-R-004 (budget arithmetic, score 6) |
| P1 | AC3 parsing variants, AC5 consortium | Important features, standard risk |
| P2 | AC2 optional params, input validation | Edge cases, low risk |

---

## TDD Red Phase (Step 4)

### 🔴 Generated Failing Tests

**Test file:** `eusolicit-app/services/client-api/tests/api/test_budget_builder.py`

All test methods carry `@pytest.mark.skip(reason="🔴 RED PHASE: Story 11.5 not yet implemented — ...")`.

### Test Classes and Methods

#### `TestAC1AC2HappyPath` — AC1, AC2 | E11-P0-009 | **P0**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_budget_builder_returns_200_with_response_structure` | 200 + `cost_categories` (6), `total_budget`, `overhead_calculation`, `co_financing_split`; `per_partner_breakdown` is null | Endpoint → 404 (route not registered) |
| `test_budget_builder_agent_payload_has_required_fields` | Captured payload has `project_description`, `duration_months`, `total_requested_funding`, `consortium_size`, `company_id` (str); `overhead_rate` is null when not supplied | Endpoint → 404, no agent call |
| `test_budget_builder_x_caller_service_header_sent` | Captured request headers contain `x-caller-service: client-api` | Endpoint → 404, no agent call |
| `test_budget_builder_with_all_optional_params` | Captured payload has `overhead_rate == 0.25` and `target_programme == "horizon_europe"` | Endpoint → 404, no agent call |

#### `TestAC3ResponseParsing` — AC3 | **P1**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_cost_categories_have_required_fields` | Each `cost_category` has `name` (non-empty str), `amount` (float ≥ 0), `justification` (str or null) | Endpoint → 404 |
| `test_overhead_calculation_fields_present` | `overhead_calculation` has `base_direct_costs`, `overhead_rate`, `overhead_amount` as numeric values | Endpoint → 404 |
| `test_co_financing_split_fields_present` | `co_financing_split` has `eu_contribution`, `own_contribution`, `eu_rate`, `total_requested_funding` | Endpoint → 404 |
| `test_agent_response_missing_cost_categories_key_returns_graceful_error` | Agent returns `{}` → 422 (arithmetic check fails; empty list sums to 0 ≠ default total) | Endpoint → 404 |

#### `TestAC4ArithmeticValidation` — AC4 | E11-P0-005 | **P0**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_valid_solo_budget_arithmetic_passes` | `MOCK_SOLO_BUDGET_RESPONSE` (all invariants hold) → 200 | Endpoint → 404 |
| `test_inconsistent_line_item_total_returns_422` | Line items sum (937500) ≠ `total_budget` (999999) → 422; `code == "BUDGET_ARITHMETIC_INCONSISTENT"` | Endpoint → 404 |
| `test_inconsistent_co_financing_sum_returns_422` | `eu_contribution + own_contribution` (900000) ≠ `total_requested_funding` (1000000) → 422 | Endpoint → 404 |
| `test_overhead_inconsistency_returns_422` | `overhead_amount` (300000) ≠ `overhead_rate × base_direct_costs` (0.25 × 1000000 = 250000) → 422 | Endpoint → 404 |

#### `TestAC5ConsortiumBudget` — AC5 | E11-P1-002 | **P1**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_consortium_budget_returns_per_partner_breakdown` | 200; `per_partner_breakdown` has 3 items; each has `partner_name`, `subtotal`, `cost_categories` | Endpoint → 404 |
| `test_consortium_budget_per_partner_arithmetic_passes` | Partner subtotals sum = `total_budget` = 2312500.0 → 200 (per-partner arithmetic valid) | Endpoint → 404 |
| `test_consortium_budget_missing_per_partner_returns_422` | `per_partner_breakdown: null` + `consortium_size=3` → 422; `code == "MISSING_PARTNER_BREAKDOWN"` | Endpoint → 404 |
| `test_solo_budget_accepts_null_per_partner` | Solo request (default `consortium_size=1`) → 200; `per_partner_breakdown` is null | Endpoint → 404 |

#### `TestAC6AgentErrorHandling` — AC6 | E11-P0-008, E11-P0-010 | **P0**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_gateway_timeout_returns_503_agent_unavailable` | `httpx.ReadTimeout` → 503; `code == "AGENT_UNAVAILABLE"`; `message == AGENT_UNAVAILABLE_MESSAGE` | Endpoint → 404 |
| `test_gateway_500_returns_503_not_500` | Gateway 500 → 503; `code == "AGENT_UNAVAILABLE"`; raw body not forwarded | Endpoint → 404 |
| `test_gateway_503_returns_503_with_standard_body` | Gateway 503 → 503; standard `AGENT_UNAVAILABLE` body | Endpoint → 404 |

#### `TestAC7Authorization` — AC7 | **P0**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_unauthenticated_returns_401` | No token → 401 | Endpoint → 404 (route missing, middleware never reached) |
| `test_read_only_role_allowed` | `read_only` role user → 200 (no role restriction) | Endpoint → 404 |

#### `TestAC2OptionalParams` — AC2, AC1 validation | E11-P2-001 | **P2**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_missing_overhead_rate_accepted` | No `overhead_rate` → 200; captured payload `overhead_rate == null` | Endpoint → 404 |
| `test_missing_target_programme_accepted` | No `target_programme` → 200; captured payload `target_programme == null` | Endpoint → 404 |
| `test_missing_consortium_size_defaults_to_one` | No `consortium_size` → 200; captured payload `consortium_size == 1` | Endpoint → 404 |
| `test_invalid_overhead_rate_above_one_returns_422` | `overhead_rate=1.5` → 422 (FastAPI Pydantic validation) | Endpoint → 404 (FastAPI never validates body when route missing) |
| `test_missing_project_description_returns_422` | No `project_description` → 422 | Endpoint → 404 |
| `test_missing_total_requested_funding_returns_422` | No `total_requested_funding` → 422 | Endpoint → 404 |

### Test Infrastructure

| Component | Detail |
|-----------|--------|
| `aigw_env_setup` fixture | Session-scoped autouse; sets `CLIENT_API_AIGW_BASE_URL=http://test-aigw.local:8000`; clears `get_settings` + `get_ai_gateway_client` LRU caches; restores on teardown |
| `budget_client_and_session` fixture | Function-scoped; register → verify email → login → yield `(client, session, token, company_id_str)` → rollback |
| `_register_and_verify_with_role()` helper | Creates secondary user in target company with specified role (mirrors S11.04 pattern) |
| `respx.mock` context manager | All agent HTTP calls intercepted; no real AI Gateway contact |
| `capture_and_respond` side-effect functions | Used by payload inspection tests; captures `request.content` → JSON parse |
| `session.rollback()` teardown | No persistent state between tests |

### Coverage Statistics (RED Phase)

| Metric | Count |
|--------|-------|
| Total test methods | **27** |
| All skipped (`@pytest.mark.skip`) | ✅ 27 / 27 |
| P0 tests | 12 |
| P1 tests | 8 |
| P2 tests | 7 |
| Acceptance criteria covered | 8 / 8 (AC1–AC8) |
| Epic test IDs covered | E11-P0-005, E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-002, E11-P2-001 |

---

## Acceptance Criteria Coverage Matrix

| AC | Description | Test Methods | Status |
|----|-------------|--------------|--------|
| AC1 | POST endpoint accepts JSON body; returns 200 + BudgetBuilderResponse | `test_budget_builder_returns_200_with_response_structure`, `test_missing_overhead_rate_accepted`, `test_missing_target_programme_accepted`, `test_missing_consortium_size_defaults_to_one`, `test_invalid_overhead_rate_above_one_returns_422`, `test_missing_project_description_returns_422`, `test_missing_total_requested_funding_returns_422` | 🔴 RED |
| AC2 | Agent payload: all required fields + X-Caller-Service + company_id string; optional params forwarded | `test_budget_builder_agent_payload_has_required_fields`, `test_budget_builder_x_caller_service_header_sent`, `test_budget_builder_with_all_optional_params`, `test_missing_overhead_rate_accepted`, `test_missing_target_programme_accepted`, `test_missing_consortium_size_defaults_to_one` | 🔴 RED |
| AC3 | BudgetBuilderResponse fully parsed: cost_categories, overhead_calculation, co_financing_split, per_partner_breakdown | `test_cost_categories_have_required_fields`, `test_overhead_calculation_fields_present`, `test_co_financing_split_fields_present`, `test_agent_response_missing_cost_categories_key_returns_graceful_error`, `test_consortium_budget_returns_per_partner_breakdown` | 🔴 RED |
| AC4 | Budget arithmetic validation: line items, co-financing, overhead, per-partner | `test_valid_solo_budget_arithmetic_passes`, `test_inconsistent_line_item_total_returns_422`, `test_inconsistent_co_financing_sum_returns_422`, `test_overhead_inconsistency_returns_422`, `test_consortium_budget_per_partner_arithmetic_passes` | 🔴 RED |
| AC5 | Per-partner breakdown required when consortium_size > 1; null accepted when solo | `test_consortium_budget_returns_per_partner_breakdown`, `test_consortium_budget_per_partner_arithmetic_passes`, `test_consortium_budget_missing_per_partner_returns_422`, `test_solo_budget_accepts_null_per_partner` | 🔴 RED |
| AC6 | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE; raw response not forwarded | `test_gateway_timeout_returns_503_agent_unavailable`, `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` | 🔴 RED |
| AC7 | Unauthenticated → 401; all authenticated roles permitted | `test_unauthenticated_returns_401`, `test_read_only_role_allowed` | 🔴 RED |
| AC8 | Stateless: no DB writes on success or failure | Implicit: no `session.flush()`/`session.commit()` in service; fixture only uses session for test setup; verified by fixture teardown rollback | 🔴 RED |

---

## Epic Test ID Coverage

| Epic Test ID | Priority | Description | Test Methods |
|-------------|----------|-------------|--------------|
| **E11-P0-005** | P0 | Budget Builder: line items sum to total_budget; overhead correctly applied; co-financing split sums to total_requested_funding | `test_valid_solo_budget_arithmetic_passes`, `test_inconsistent_line_item_total_returns_422`, `test_inconsistent_co_financing_sum_returns_422`, `test_overhead_inconsistency_returns_422` |
| **E11-P0-008** | P0 | AGENT_UNAVAILABLE on gateway 5xx (consistent error shape) | `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` |
| **E11-P0-009** | P0 | AI Gateway mock: budget-builder agent returns deterministic fixture in CI (smoke gate) | `test_budget_builder_returns_200_with_response_structure`, `test_budget_builder_agent_payload_has_required_fields` |
| **E11-P0-010** | P0 | 30s timeout enforced; request does not hang past timeout | `test_gateway_timeout_returns_503_agent_unavailable` (via `httpx.ReadTimeout` mock) |
| **E11-P1-002** | P1 | Budget Builder: per-partner breakdown present when consortium_size > 1; co_financing_split visualizable | `test_consortium_budget_returns_per_partner_breakdown`, `test_consortium_budget_per_partner_arithmetic_passes`, `test_consortium_budget_missing_per_partner_returns_422`, `test_solo_budget_accepts_null_per_partner` |
| **E11-P2-001** | P2 | Budget Builder: missing optional params → defaults or clear validation error | `test_missing_overhead_rate_accepted`, `test_missing_target_programme_accepted`, `test_missing_consortium_size_defaults_to_one`, `test_invalid_overhead_rate_above_one_returns_422`, `test_missing_project_description_returns_422`, `test_missing_total_requested_funding_returns_422` |

---

## Implementation Guidance for Developer

### Files to Modify

| File | Action |
|------|--------|
| `src/client_api/schemas/grants.py` | **Edit** — append 6 new schema classes: `CostCategory`, `OverheadCalculation`, `CoFinancingSplit`, `PartnerBudget`, `BudgetBuilderRequest`, `BudgetBuilderResponse` |
| `src/client_api/services/grants_service.py` | **Edit** — append `_parse_cost_category()`, `_parse_partner_budget()`, `_validate_budget_arithmetic()`, `build_budget()` + add imports |
| `src/client_api/api/v1/grants.py` | **Edit** — add `POST /budget-builder` endpoint + imports |
| `main.py` | **No change** — `/grants` router already registered by S11.04 |

### Critical Implementation Notes

1. **Arithmetic tolerance**: `_ARITHMETIC_TOLERANCE = 0.01` (one Euro cent) — define as module constant in `grants_service.py`
2. **Validation order**: `MISSING_PARTNER_BREAKDOWN` check BEFORE `_validate_budget_arithmetic` — per-partner check fires first
3. **UUID serialisation**: `str(current_user.company_id)` in payload — httpx JSON does not handle `uuid.UUID` 
4. **Router catch block**: `except HTTPException as exc` in router must handle both `status_code == 503` and `status_code == 422` with `isinstance(exc.detail, dict)` → return flat `JSONResponse` (same pattern as S11.04 eligibility endpoint)
5. **Stateless**: no `AsyncSession` dependency in router or service; `get_current_user` provides `company_id` from JWT only
6. **Role guard**: use `Depends(get_current_user)` (not `require_role`) — all roles allowed (AC7)
7. **Graceful parser**: `_parse_cost_category` and `_parse_partner_budget` return `None` on error + filter → never raise from parser code
8. **Overhead check condition**: only validate overhead when `overhead_calculation.overhead_rate > 0` (skip check if rate is 0 to avoid false positives on zero-overhead programmes)

### Fixture Arithmetic Verification

**MOCK_SOLO_BUDGET_RESPONSE** (pre-verified consistent):
- Line items: 750000 + 200000 + 50000 + 100000 + 50000 + 287500 = **1 437 500** ✓ = `total_budget`
- Overhead: 1 150 000 × 0.25 = **287 500** ✓ = `overhead_amount`; base = 750000+200000+50000+100000+50000 = **1 150 000** ✓
- Co-financing: 1 150 000 + 287 500 = **1 437 500** ✓ = `total_requested_funding`

**MOCK_CONSORTIUM_BUDGET_RESPONSE** (pre-verified consistent):
- Line items: 1200000 + 300000 + 80000 + 200000 + 70000 + 462500 = **2 312 500** ✓ = `total_budget`
- Overhead: 1 850 000 × 0.25 = **462 500** ✓; base = 1200000+300000+80000+200000+70000 = **1 850 000** ✓
- Co-financing: 1 850 000 + 462 500 = **2 312 500** ✓ = `total_requested_funding`
- Partners: 750 000 + 750 000 + 812 500 = **2 312 500** ✓ = `total_budget`

**MOCK_INCONSISTENT_BUDGET_RESPONSE** (deliberately broken — line items):
- Line items: 750000 + 187500 = 937500 ≠ `total_budget` (999999) — triggers `BUDGET_ARITHMETIC_INCONSISTENT`

**MOCK_INCONSISTENT_CO_FINANCING_RESPONSE** (deliberately broken — co-financing):
- eu_contribution + own_contribution: 800000 + 100000 = 900000 ≠ `total_requested_funding` (1000000)

**MOCK_INCONSISTENT_OVERHEAD_RESPONSE** (deliberately broken — overhead):
- overhead_amount (300000) ≠ overhead_rate × base_direct_costs (0.25 × 1000000 = 250000)

---

## Next Steps (TDD Green Phase)

After implementing all tasks in Story 11.5:

1. **Remove `@pytest.mark.skip`** from all 27 test methods in `test_budget_builder.py`
2. **Run tests:**
   ```bash
   cd eusolicit-app/services/client-api
   pytest tests/api/test_budget_builder.py -v --tb=short
   ```
3. **Verify all 27 tests PASS** (green phase)
4. If any test fails:
   - Fix implementation bug (if assertion is correct)
   - Fix test expectation (if assertion is wrong)
5. **Run full suite** to verify no regressions:
   ```bash
   pytest tests/ -v -x
   ```
6. **Commit** passing test file alongside implementation

---

## Validation Checklist (Step 5)

- [x] Prerequisites verified (framework, dependencies, reference patterns)
- [x] Stack detected as `backend` (correct — client-api service, no browser tests in scope)
- [x] Generation mode: AI generation (correct for backend)
- [x] All 8 acceptance criteria mapped to test scenarios
- [x] All 6 epic test IDs (E11-P0-005, E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-002, E11-P2-001) covered
- [x] 27 test methods generated — all with `@pytest.mark.skip` (RED phase)
- [x] No placeholder assertions (`assert True`, `pass`) — all assert expected behaviour
- [x] `respx.mock` context manager used consistently — no real AI Gateway calls
- [x] `aigw_env_setup` session-scoped autouse fixture — mirrors S11.04 pattern exactly
- [x] `budget_client_and_session` function-scoped — no cross-test pollution
- [x] `session.rollback()` teardown — no persistent DB state
- [x] `_register_and_verify_with_role()` helper — follows established S11.04 pattern
- [x] 5 deterministic fixture responses defined: solo (consistent), consortium (consistent), inconsistent line-items, inconsistent co-financing, inconsistent overhead
- [x] Fixture arithmetic pre-verified in checklist and code comments
- [x] Test file written to disk: `tests/api/test_budget_builder.py`
- [x] ATDD checklist written to disk: `test-artifacts/atdd-checklist-11-5-budget-builder-agent-integration.md`
- [x] No temp artifacts in random locations — all in `test-artifacts/`

### Key Assumptions and Risks

| Assumption | Risk | Mitigation |
|------------|------|------------|
| `grants.py` router in `main.py` is already registered from S11.04 | Low — documented in story Dev Notes (no change to main.py needed) | Verified in story tasks: "No changes required to `main.py`" |
| `respx` mock intercepting httpx calls works identically to S11.04 pattern | Low — verified pattern in eligibility tests | Direct code reuse of `aigw_env_setup` fixture |
| `company_id_str` from `/auth/register` matches JWT claim | Low — established by S11.03/S11.04 tests | Same registration + login flow used |
| `AiGatewayTimeoutError` and `AiGatewayUnavailableError` are already defined in `ai_gateway_client.py` | Low — implemented in S11.03 | Error class names referenced in story task 2.5 |
| Overhead arithmetic check skipped when `overhead_rate == 0.0` (zero-overhead programmes) | Low — documented in story tasks | Test uses `overhead_rate = 0.25` in all arithmetic tests; edge case covered by absence |
| `MISSING_PARTNER_BREAKDOWN` check fires before `BUDGET_ARITHMETIC_INCONSISTENT` | Medium — ordering matters; test assertions assume `MISSING_PARTNER_BREAKDOWN` code | Developer must follow ordering in task 2.5: per-partner check before arithmetic validation |

---

*Generated by TEA Master Test Architect | BMAD ATDD Workflow | 2026-04-09*
