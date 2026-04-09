---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-09'
workflowType: 'atdd'
storyId: '11-4-grant-eligibility-agent-integration'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/11-4-grant-eligibility-agent-integration.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-11.md'
  - 'eusolicit-app/services/client-api/tests/api/test_espd_autofill_export.py'
  - 'eusolicit-app/services/client-api/src/client_api/services/ai_gateway_client.py'
  - 'eusolicit-app/services/client-api/src/client_api/models/company.py'
  - 'eusolicit-app/services/client-api/src/client_api/config.py'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
---

# ATDD Checklist: Story 11.4 — Grant Eligibility Agent Integration

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Tests written; feature not yet implemented
**Story File:** `eusolicit-docs/implementation-artifacts/11-4-grant-eligibility-agent-integration.md`
**Test File:** `eusolicit-app/services/client-api/tests/api/test_grant_eligibility.py`

---

## Preflight Summary (Step 1)

### Stack Detection

| Check | Result |
|-------|--------|
| `pyproject.toml` found | ✅ `eusolicit-app/services/client-api/pyproject.toml` |
| `playwright.config.ts` / `cypress.config.ts` | ❌ Not present |
| **Detected stack** | **`backend`** |
| Test framework | `pytest` + `pytest-asyncio` |
| AI Gateway mock library | `respx>=0.21` (already in `[project.optional-dependencies] dev`) |

### Prerequisites Check

| Requirement | Status |
|-------------|--------|
| Story approved with clear acceptance criteria (7 ACs) | ✅ |
| `conftest.py` with session factory, RSA fixtures, Redis fixtures | ✅ |
| `AiGatewayClient` implemented (Story 11.3) | ✅ `src/client_api/services/ai_gateway_client.py` |
| `ClientApiSettings.aigw_base_url` / `aigw_timeout_seconds` | ✅ `src/client_api/config.py` |
| `Company` ORM model | ✅ `src/client_api/models/company.py` |
| Reference test pattern | ✅ `tests/api/test_espd_autofill_export.py` |
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

Acceptance criteria are clear and detailed. All scenarios are standard API/service patterns (POST endpoint, auth middleware, AI Gateway mock with `respx`). Full AI generation from story AC specification.

---

## Test Strategy (Step 3)

### Acceptance Criteria → Test Scenario Mapping

| AC | Description | Scenarios | Level | Priority | Epic Test IDs |
|----|-------------|-----------|-------|----------|---------------|
| **AC1** | POST endpoint accepts optional filters, loads company, returns 200 | Happy path; empty body `{}`; company_id echoed | Integration/API | P0 | E11-P0-004, E11-P0-009 |
| **AC2** | Agent payload: company_profile + filters + X-Caller-Service + 30s timeout | Payload structure captured; header captured; filter forwarding | Integration/API | P0 | E11-P0-004, E11-P0-009 |
| **AC3** | Response parsed into GrantEligibilityResponse | Full-match (2 programmes); partial-match (1); no-match (0); missing key defaults | Integration/API | P1 | E11-P1-001 |
| **AC4** | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE | Timeout (ReadTimeout); HTTP 500; HTTP 503 | Integration/API | P0 | E11-P0-004, E11-P0-008, E11-P0-010 |
| **AC5** | Company not found → 404 | Company deleted after JWT issued | Integration/API | P1 | — |
| **AC6** | Unauthenticated → 401; all roles allowed | No token → 401; read_only role → 200 | Integration/API | P0 | — |
| **AC7** | No client-side filtering; null when absent | Filter forwarding verbatim; null for absent params | Integration/API | P1 | E11-P1-001 |

### Test Level Selection (backend stack)

- **Integration** (function-scope, real DB session via rollback): all tests
- **No E2E** (backend stack — no browser required)
- **No Unit** (business logic is thin service delegation; integration tests give better coverage)

### Priority Assignment

| Priority | Tests | Rationale |
|----------|-------|-----------|
| P0 | AC1 happy path, AC2 payload/header, AC4 error handling, AC6 auth | Core functionality + E11-R-001 (AI Gateway error handling risk score 6) |
| P1 | AC3 parsing variants, AC5 edge case, AC7 filter forwarding | Important features, standard risk |

---

## TDD Red Phase (Step 4)

### 🔴 Generated Failing Tests

**Test file:** `eusolicit-app/services/client-api/tests/api/test_grant_eligibility.py`

All test methods carry `@pytest.mark.skip(reason="🔴 RED PHASE: Story 11.4 not yet implemented — ...")`.

### Test Classes and Methods

#### `TestAC1AC2HappyPath` — AC1, AC2 | E11-P0-004, E11-P0-009 | **P0**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_eligibility_check_returns_200_with_response_structure` | 200 + `matched_programmes`, `total_matched==2`, `company_id`, `checked_at` | Endpoint → 404 (route not registered) |
| `test_eligibility_check_agent_payload_has_company_profile_and_filters` | Captured payload has `company_profile.company_id` as str; `filters.programme_type == null` | Endpoint → 404, no agent call |
| `test_eligibility_check_with_programme_type_filter` | `filters.programme_type == "horizon_europe"` forwarded verbatim | Endpoint → 404, no agent call |
| `test_eligibility_check_with_funding_range_filter` | `filters.funding_range_min_eur == 100000.0`, `funding_range_max_eur == 5000000.0` | Endpoint → 404, no agent call |
| `test_eligibility_check_x_caller_service_header_sent` | `x-caller-service: client-api` header on agent request | Endpoint → 404, no agent call |

#### `TestAC3ResponseParsing` — AC3 | E11-P1-001 | **P1**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_full_match_programmes_have_required_fields` | Each programme has non-empty `programme_name`, int `eligibility_score` 0–100, non-empty `requirements_summary` | Endpoint → 404 |
| `test_partial_match_scenario` | `total_matched==1`; `eligibility_score==61`; `call_reference==null` | Endpoint → 404 |
| `test_no_match_scenario` | 200; `matched_programmes==[]`; `total_matched==0` | Endpoint → 404 |
| `test_agent_response_missing_matched_programmes_key_defaults_to_empty_list` | Agent returns `{}` → 200; graceful degradation to `matched_programmes==[]` | Endpoint → 404 |

#### `TestAC4AgentErrorHandling` — AC4 | E11-P0-004, E11-P0-008, E11-P0-010 | **P0**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_gateway_timeout_returns_503_agent_unavailable` | `httpx.ReadTimeout` → 503; `code=="AGENT_UNAVAILABLE"`; correct message | Endpoint → 404 |
| `test_gateway_500_returns_503_not_500` | Gateway 500 → 503; raw body not forwarded | Endpoint → 404 |
| `test_gateway_503_returns_503_with_standard_body` | Gateway 503 → 503 with standard AGENT_UNAVAILABLE body | Endpoint → 404 |

#### `TestAC5CompanyNotFound` — AC5 | **P1**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_company_not_found_returns_404` | After SQL DELETE company → 404; `detail=="Company not found"` | Endpoint → 404 for wrong reason (route missing); body mismatch |

#### `TestAC6Authorization` — AC6 | **P0**

| Method | Assertion | Why It Fails (RED) |
|--------|-----------|-------------------|
| `test_unauthenticated_returns_401` | No token → 401 | Endpoint → 404 (route missing, middleware never reached) |
| `test_read_only_role_allowed` | `read_only` role user → 200 (no role restriction beyond valid JWT) | Endpoint → 404 |

### Test Infrastructure

| Component | Detail |
|-----------|--------|
| `aigw_env_setup` fixture | Session-scoped autouse; sets `CLIENT_API_AIGW_BASE_URL=http://test-aigw.local:8000`; clears `get_settings` + `get_ai_gateway_client` LRU caches |
| `eligibility_client_and_session` fixture | Function-scoped; register → verify email → login → yield `(client, session, token, company_id_str)` → rollback |
| `_register_and_verify_with_role()` helper | Creates secondary user in target company with specified role (mirrors autofill test pattern) |
| `respx.mock` context manager | All agent HTTP calls intercepted; no real AI Gateway contact |
| `session.rollback()` teardown | No persistent state between tests |

### Coverage Statistics (RED Phase)

| Metric | Count |
|--------|-------|
| Total test methods | **15** |
| All skipped (`@pytest.mark.skip`) | ✅ 15 / 15 |
| P0 tests | 9 |
| P1 tests | 6 |
| Acceptance criteria covered | 7 / 7 (AC1–AC7) |
| Epic test IDs covered | E11-P0-004, E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-001 |

---

## Acceptance Criteria Coverage Matrix

| AC | Description | Test Methods | Status |
|----|-------------|--------------|--------|
| AC1 | POST endpoint, optional filters, company loaded, 200 + GrantEligibilityResponse | `test_eligibility_check_returns_200_with_response_structure` | 🔴 RED |
| AC2 | Agent payload structure, X-Caller-Service header, 30s timeout, no client retry | `test_eligibility_check_agent_payload_has_company_profile_and_filters`, `test_eligibility_check_with_programme_type_filter`, `test_eligibility_check_with_funding_range_filter`, `test_eligibility_check_x_caller_service_header_sent` | 🔴 RED |
| AC3 | GrantEligibilityResponse parsed, total_matched, company_id, checked_at | `test_full_match_programmes_have_required_fields`, `test_partial_match_scenario`, `test_no_match_scenario`, `test_agent_response_missing_matched_programmes_key_defaults_to_empty_list` | 🔴 RED |
| AC4 | Gateway timeout/5xx → 503 AGENT_UNAVAILABLE, no raw error forwarded | `test_gateway_timeout_returns_503_agent_unavailable`, `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` | 🔴 RED |
| AC5 | Company not found → 404 `{"detail": "Company not found"}` | `test_company_not_found_returns_404` | 🔴 RED |
| AC6 | Unauthenticated → 401; all roles permitted | `test_unauthenticated_returns_401`, `test_read_only_role_allowed` | 🔴 RED |
| AC7 | No client-side filtering; null for absent params; verbatim forwarding | `test_eligibility_check_with_programme_type_filter`, `test_eligibility_check_with_funding_range_filter`, `test_eligibility_check_agent_payload_has_company_profile_and_filters` | 🔴 RED |

---

## Epic Test ID Coverage

| Epic Test ID | Priority | Description | Test Methods |
|-------------|----------|-------------|--------------|
| **E11-P0-004** | P0 | Grant Eligibility: structured 503 on timeout; structured list on success | `test_eligibility_check_returns_200_with_response_structure`, `test_gateway_timeout_returns_503_agent_unavailable` |
| **E11-P0-008** | P0 | AGENT_UNAVAILABLE on gateway 5xx (consistent error shape) | `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` |
| **E11-P0-009** | P0 | Deterministic fixture response in CI (smoke gate) | `test_eligibility_check_returns_200_with_response_structure` |
| **E11-P0-010** | P0 | 30s timeout enforced; request does not hang past timeout | `test_gateway_timeout_returns_503_agent_unavailable` (via `httpx.ReadTimeout` mock) |
| **E11-P1-001** | P1 | Full-match, partial-match, no-match scenarios; filter params applied | `test_full_match_programmes_have_required_fields`, `test_partial_match_scenario`, `test_no_match_scenario`, `test_eligibility_check_with_programme_type_filter`, `test_eligibility_check_with_funding_range_filter` |

---

## Implementation Guidance for Developer

### Files to Create

| File | Action |
|------|--------|
| `src/client_api/schemas/grants.py` | **Create** — `GrantEligibilityRequest`, `MatchedProgramme`, `GrantEligibilityResponse` |
| `src/client_api/services/grants_service.py` | **Create** — `check_grant_eligibility()`, `_load_company()`, `_build_company_payload()`, `_parse_matched_programme()` |
| `src/client_api/api/v1/grants.py` | **Create** — `POST /eligibility-check` router |
| `src/client_api/main.py` | **Edit** — add 2 lines: import + `api_v1_router.include_router(grants_v1.router)` |

### Critical Implementation Notes

1. **UUID serialisation**: `str(company.id)` in `_build_company_payload()` — httpx JSON does not handle `uuid.UUID`
2. **Null filter forwarding**: absent request body fields → `null` in agent payload (AC7, not omitted)
3. **Graceful parser defaults**: `raw.get("matched_programmes", [])` when key absent
4. **Error mapping**: `AiGatewayTimeoutError` + `AiGatewayUnavailableError` → both → `HTTPException(503, detail={...})`
5. **Stateless**: no `session.flush()` / `session.commit()` on success path
6. **Role guard**: use `Depends(get_current_user)` (not `require_role`) — all roles allowed

---

## Next Steps (TDD Green Phase)

After implementing all tasks in Story 11.4:

1. **Remove `@pytest.mark.skip`** from all 14 test methods in `test_grant_eligibility.py`
2. **Run tests:**
   ```bash
   cd eusolicit-app/services/client-api
   pytest tests/api/test_grant_eligibility.py -v --tb=short
   ```
3. **Verify all 14 tests PASS** (green phase)
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
- [x] Stack detected as `backend` (correct — no frontend config files)
- [x] Generation mode: AI generation (correct for backend)
- [x] All 7 acceptance criteria mapped to test scenarios
- [x] All 5 epic test IDs (E11-P0-004, E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-001) covered
- [x] 15 test methods generated — all with `@pytest.mark.skip` (RED phase)
- [x] No placeholder assertions (`assert True`, `pass`) — all assert expected behaviour
- [x] `respx.mock` used consistently — no real AI Gateway calls
- [x] `aigw_env_setup` session-scoped autouse fixture — mirrors S11.03 pattern
- [x] `eligibility_client_and_session` function-scoped — no cross-test pollution
- [x] `session.rollback()` teardown — no persistent DB state
- [x] `_register_and_verify_with_role()` helper — follows established pattern
- [x] Test file written to disk: `tests/api/test_grant_eligibility.py`
- [x] ATDD checklist written to disk: `test-artifacts/atdd-checklist-11-4-grant-eligibility-agent-integration.md`
- [x] No temp artifacts in random locations — all in `test-artifacts/`

### Key Assumptions and Risks

| Assumption | Risk | Mitigation |
|------------|------|------------|
| `Company` model fields (`industry`, `cpv_sectors`, `regions`, `certifications`, `address`) remain stable | Low — model is from S2.08 | Tests don't assert on specific company field values |
| `respx` mock intercepting httpx calls works identically to S11.03 pattern | Low — verified pattern in autofill tests | Direct code reuse of `aigw_env_setup` fixture |
| `company_id_str` from `/auth/register` matches JWT claim | Established by S11.03 tests — low risk | Same registration + login flow used |
| AC5 (company delete) test may fail if FK constraints prevent deletion | Medium — company_memberships FK | If FK error: adjust test to use SQL to set id to non-existent UUID instead of DELETE |

---

*Generated by TEA Master Test Architect | BMAD ATDD Workflow | 2026-04-09*
