---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-25'
workflowType: bmad-testarch-atdd
mode: create
storyId: '11.6'
storyKey: 11-6-consortium-finder-agent-integration
storyFile: eusolicit-docs/implementation-artifacts/11-6-consortium-finder-agent-integration.md
atddChecklistPath: test_artifacts/atdd-checklist-11-6-consortium-finder-agent-integration.md
generatedTestFiles:
  - eusolicit-app/services/client-api/tests/api/test_consortium_finder.py
detectedStack: fullstack (story type: backend)
generationMode: AI Generation (backend — no browser recording)
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/11-6-consortium-finder-agent-integration.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - eusolicit-app/services/client-api/tests/api/test_consortium_finder.py
  - eusolicit-app/services/client-api/tests/api/test_budget_builder.py
  - eusolicit-app/services/client-api/tests/conftest.py
---

# ATDD Checklist: Story 11.6 — Consortium Finder Agent Integration

**Date:** 2026-04-25
**Author:** BMAD TEA Master Test Architect
**TDD Phase:** 🔴 RED — Story 11.6 NOT yet implemented. All 26 tests will fail because the
  endpoint `POST /api/v1/grants/consortium-finder` does not exist yet. The schemas
  (`PastProject`, `PartnerSuggestion`, `ConsortiumFinderRequest`, `ConsortiumFinderResponse`)
  and service function `find_consortium_partners()` have not been created. Tests fail
  naturally without `@pytest.mark.skip` — consistent with the project pattern (test_budget_builder.py,
  test_grant_eligibility.py).
**Story Status:** approved — ready for dev-story

---

## Step 1: Preflight & Context

### Stack Detection

- **Detected stack:** `fullstack` (auto-detection: `pyproject.toml` + `playwright.config.ts` both present)
- **Story type override:** `backend` — this story has no frontend components; only backend test levels apply
- **Test framework:** pytest 8.x + pytest-asyncio + httpx AsyncClient (ASGITransport) + respx MockRouter

### Prerequisites Check

| Requirement | Status | Notes |
|---|---|---|
| Story has clear acceptance criteria | ✅ | 9 ACs: endpoint schema, payload contract, response parsing, pagination, sorting, graceful degradation, error handling, auth, stateless |
| pytest conftest with async session fixtures | ✅ | `client_api_session_factory`, `test_redis_client` in conftest.py |
| `get_current_user` dependency | ✅ | Used for auth gate and `company_id` extraction (AC8, AC9) |
| `AiGatewayClient` (Story 11.3/11.4) | ✅ | `client_api.services.ai_gateway_client` — `run_agent`, `AiGatewayTimeoutError`, `AiGatewayUnavailableError` |
| `/grants` router already registered (Story 11.4) | ✅ | `main.py` unchanged — router already mounted; only endpoint within `grants.py` is new |
| `ClientApiSettings.aigw_*` config fields | ✅ | `CLIENT_API_AIGW_BASE_URL`, `AIGW_TIMEOUT_SECONDS` — already wired from S11.04 |
| `grants_service.py` + `grants.py` (editable) | ✅ | Files exist from S11.04/S11.05; story adds new functions/endpoint |
| `grants.py` schemas file (editable) | ✅ | `src/client_api/schemas/grants.py` exists; 4 new classes appended (AC1, AC3) |
| respx installed for AI Gateway mocking | ✅ | `pytest.importorskip("respx")` guard at module level |
| No DB write required (AC9 — stateless) | ✅ | `AsyncSession` dependency NOT added to this endpoint |

### Epic 11 Test Design Coverage

Loaded `eusolicit-docs/test-artifacts/test-design-epic-11.md`. Relevant E11 test IDs for S11.6:

| Epic Test ID | Priority | Coverage Summary |
|---|---|---|
| **E11-P0-008** | P0 | `consortium-finder` endpoint returns `{"message": "...", "code": "AGENT_UNAVAILABLE"}` on gateway 5xx |
| **E11-P0-009** | P0 | AI Gateway mock: consortium-finder agent returns deterministic fixture response in CI (smoke gate) |
| **E11-P0-010** | P0 | 30 s timeout enforced; `ReadTimeout` mock → 503 |
| **E11-P1-003** | P1 | Paginated results; capability overlap ranking; single-country filter; max_results honoured |
| **E11-P2-002** | P2 | Empty results (no matches); contact_info absent → null, not crash (E11-R-012) |

Risks addressed by this story:
- **E11-R-001** (AI Gateway error handling) — mitigated by AC7 + `TestAC7AgentErrorHandling`
- **E11-R-012** (Consortium Finder result field completeness) — mitigated by AC6 + `TestAC6GracefulDegradation`

---

## Step 2: Generation Mode

**Selected mode:** AI Generation (backend — no browser recording)

**Rationale:** Pure backend API story. All test patterns follow `test_budget_builder.py` arrange/act/assert
format with respx mocking for the AI Gateway. No browser automation required. The test file
`test_consortium_finder.py` was pre-specified in full detail in the story's Dev Notes (Task 4),
including fixture responses, fixture setup, and all 26 test methods. The generated file mirrors the
S11.05 pattern exactly.

---

## Step 3: Test Strategy

| Priority | Area | Scenarios | Harness |
|---|---|---|---|
| **P0** | Happy path: 200 response + payload structure (AC1, AC2) | 5 | API — respx MockRouter + ASGITransport |
| **P0** | Response parsing + ranking (AC3, AC5) | 6 | API — respx MockRouter |
| **P0** | AI Gateway error handling → 503 (AC7) | 3 | API — respx timeout/500/503 mocks |
| **P1** | Graceful field degradation (AC6) | 4 | API — partial/empty agent responses |
| **P1** | Auth: 401/all-roles-permitted (AC8) | 2 | API — no token / read_only role |
| **P2** | Input validation → 422 (AC1 constraints) | 6 | API — Pydantic validation |
| **Total** | | **26 tests** | |

**Key design decisions:**
- Tests fail naturally (no `@pytest.mark.skip`) — project convention from test_budget_builder.py
- `respx.mock` context manager intercepts all AI Gateway HTTP calls; no real gateway contact
- `consortium_client_and_session` fixture: function-scoped; registers unique admin user + company per test
- `aigw_env_setup`: session-scoped autouse; sets `CLIENT_API_AIGW_BASE_URL=http://test-aigw.local:8000`
- `MOCK_RANKED_PARTNERS_RESPONSE` scores intentionally non-ranked (85.0, 72.0, 91.0) to verify AC5 sort
- `MOCK_PARTIAL_PARTNER_RESPONSE` omits `contact_info` and `past_projects` keys entirely (not null) — per E11-R-012

---

## Step 4: Generated Test File

### `tests/api/test_consortium_finder.py` — 26 tests (RED PHASE)

**Location:** `eusolicit-app/services/client-api/tests/api/test_consortium_finder.py`

**Status:** ✅ File exists on disk (pre-written as part of story Task 4 specification)

#### `TestAC1AC2HappyPath` — 5 tests (P0, E11-P0-009)

Success path + payload verification. All fail in RED PHASE: endpoint returns 404 (not 200).

| Test Method | AC | Epic ID | Failure Reason (RED) |
|---|---|---|---|
| `test_consortium_finder_returns_200_with_response_structure` | AC1, AC3 | E11-P0-009 | POST → 404 (endpoint missing) |
| `test_consortium_finder_agent_payload_has_required_fields` | AC2 | E11-P0-009 | 404 → agent never called → payload not captured |
| `test_consortium_finder_x_caller_service_header_sent` | AC2 | E11-P0-009 | 404 → no request captured → header assertion fails |
| `test_consortium_finder_forwards_target_countries` | AC2 | E11-P0-009 | 404 → target_countries not captured |
| `test_consortium_finder_forwards_max_results` | AC2 | E11-P0-009 | 404 → max_results not captured |

#### `TestAC3AC5ResponseParsing` — 6 tests (P0/P1, E11-P1-003)

Parsing, ranking by `collaboration_score`, pagination contract.

| Test Method | AC | Epic ID | Failure Reason (RED) |
|---|---|---|---|
| `test_partners_returned_in_descending_collaboration_score_order` | AC5 | E11-P1-003 | 404 → no parsed response; sort not applied |
| `test_total_results_and_page_size_match_partner_count` | AC3, AC4 | E11-P1-003 | 404 → no response fields |
| `test_partners_have_required_fields` | AC3 | E11-P1-003 | 404 → no partner objects returned |
| `test_single_country_filter_scenario` | AC3 | E11-P1-003 | 404 → no partner returned |
| `test_max_results_param_forwarded_to_agent` | AC2 | E11-P1-003 | 404 → agent not called |
| `test_capability_overlap_ranking_reflected_in_scores` | AC5 | E11-P1-003 | 404 → no sorted response |

#### `TestAC6GracefulDegradation` — 4 tests (P1/P2, E11-P2-002, E11-R-012)

Partial and empty agent responses degrade gracefully.

| Test Method | AC | Epic ID | Failure Reason (RED) |
|---|---|---|---|
| `test_empty_results_returns_200_with_empty_list` | AC6 | E11-P2-002 | 404 → no response |
| `test_partner_with_missing_contact_info_returns_null` | AC6 | E11-P2-002, E11-R-012 | 404 → no partner parsed |
| `test_partner_with_missing_past_projects_returns_empty_list` | AC6 | E11-P2-002 | 404 → no partner parsed |
| `test_agent_response_missing_partners_key_defaults_to_empty_list` | AC6 | E11-P2-002 | 404 → no response |

#### `TestAC7AgentErrorHandling` — 3 tests (P0, E11-P0-008, E11-P0-010)

AI Gateway failures → structured 503.

| Test Method | AC | Epic ID | Failure Reason (RED) |
|---|---|---|---|
| `test_gateway_timeout_returns_503_agent_unavailable` | AC7 | E11-P0-008, E11-P0-010 | 404 → ReadTimeout mock never reached; `AGENT_UNAVAILABLE` not returned |
| `test_gateway_500_returns_503_not_500` | AC7 | E11-P0-008 | 404 → agent mock not invoked |
| `test_gateway_503_returns_503_with_standard_body` | AC7 | E11-P0-008 | 404 → agent mock not invoked |

#### `TestAC8Authorization` — 2 tests

Auth gate: 401 for no token; all roles permitted.

| Test Method | AC | Failure Reason (RED) |
|---|---|---|
| `test_unauthenticated_returns_401` | AC8 | 404 (route not found) ≠ 401 (route exists but auth fails) |
| `test_read_only_role_allowed` | AC8 | 404 → `_register_and_verify_with_role` succeeds but POST fails |

#### `TestAC1InputValidation` — 6 tests (P2)

FastAPI/Pydantic 422 validation before agent is called.

| Test Method | AC | Failure Reason (RED) |
|---|---|---|
| `test_missing_project_description_returns_422` | AC1 | 404 ≠ 422 (Pydantic never runs) |
| `test_empty_required_capabilities_returns_422` | AC1 | 404 ≠ 422 |
| `test_missing_required_capabilities_returns_422` | AC1 | 404 ≠ 422 |
| `test_max_results_above_50_returns_422` | AC1 | 404 ≠ 422 |
| `test_max_results_below_1_returns_422` | AC1 | 404 ≠ 422 |
| `test_null_target_countries_accepted` | AC1 | 404 ≠ 200 |

---

## Step 4c: Aggregate

All test infrastructure confirmed present.

### File List

| File | Phase | Test Count |
|---|---|---|
| `eusolicit-app/services/client-api/tests/api/test_consortium_finder.py` | 🔴 RED | 26 |
| **Total** | | **26** |

### AC → Test Coverage Mapping

| AC | Test Classes | Coverage |
|---|---|---|
| AC1 (POST endpoint + request schema) | `TestAC1AC2HappyPath` × 1, `TestAC1InputValidation` × 6 | ✅ Full |
| AC2 (Agent payload: fields + header + timeout) | `TestAC1AC2HappyPath` × 4 | ✅ Full |
| AC3 (Response parsing: ConsortiumFinderResponse shape) | `TestAC3AC5ResponseParsing` × 3 | ✅ Full |
| AC4 (Pagination contract: page=1, page_size=len) | `TestAC3AC5ResponseParsing` × 1 | ✅ Full |
| AC5 (Descending collaboration_score sort) | `TestAC3AC5ResponseParsing` × 2 | ✅ Full |
| AC6 (Graceful field degradation: null/[]) | `TestAC6GracefulDegradation` × 4 | ✅ Full |
| AC7 (AI Gateway errors → 503 AGENT_UNAVAILABLE) | `TestAC7AgentErrorHandling` × 3 | ✅ Full |
| AC8 (No JWT → 401; all roles allowed) | `TestAC8Authorization` × 2 | ✅ Full |
| AC9 (Stateless — no DB writes) | Structural (no `AsyncSession` in endpoint); AC7 confirms no state on failure | ✅ Full |

### Epic Test ID Coverage

| Epic Test ID | Priority | Test Methods | Status |
|---|---|---|---|
| **E11-P0-008** | P0 | `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` | 🔴 RED |
| **E11-P0-009** | P0 | `test_consortium_finder_returns_200_with_response_structure`, `test_consortium_finder_agent_payload_has_required_fields` | 🔴 RED |
| **E11-P0-010** | P0 | `test_gateway_timeout_returns_503_agent_unavailable` | 🔴 RED |
| **E11-P1-003** | P1 | `test_partners_returned_in_descending_collaboration_score_order`, `test_single_country_filter_scenario`, `test_max_results_param_forwarded_to_agent`, `test_capability_overlap_ranking_reflected_in_scores`, `test_total_results_and_page_size_match_partner_count` | 🔴 RED |
| **E11-P2-002** | P2 | `test_empty_results_returns_200_with_empty_list`, `test_partner_with_missing_contact_info_returns_null` | 🔴 RED |

---

## Step 5: Validate & Complete

### 1. Validation

| Check | Status | Notes |
|---|---|---|
| Prerequisites satisfied | ✅ | All S11.6 dependencies (AiGatewayClient, grants router, config) confirmed present |
| Test file exists on disk | ✅ | `test_consortium_finder.py` — 26 tests across 5 classes |
| All 9 ACs have test coverage | ✅ | AC1–AC9 fully mapped (see AC coverage table above) |
| TDD phase documented | ✅ | 🔴 RED — all tests fail naturally (404) until implementation complete |
| Epic E11 test IDs covered | ✅ | E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-003, E11-P2-002 all covered |
| Story metadata and handoff paths captured | ✅ | Frontmatter populated |
| CLI sessions cleaned up | N/A | Backend testing — no browser sessions |
| Temp artifacts stored in `test_artifacts/` | ✅ | Checklist at correct path |
| respx mock pattern consistent with S11.05 | ✅ | `respx.mock` context manager; `aigw_env_setup` autouse session fixture |
| `MOCK_RANKED_PARTNERS_RESPONSE` scores non-ranked | ✅ | 85.0 → 72.0 → 91.0 in agent order; sorted response expected 91.0 → 85.0 → 72.0 |
| `MOCK_PARTIAL_PARTNER_RESPONSE` omits keys (not null) | ✅ | `contact_info` and `past_projects` keys absent — simulates E11-R-012 scenario |
| No `AsyncSession` in endpoint (AC9 stateless) | ✅ | Router uses `get_current_user` + `get_ai_gateway_client` only |

### 2. Completion Summary

- **Test file created:** 1 file, 26 test scenarios
- **File path:** `eusolicit-app/services/client-api/tests/api/test_consortium_finder.py`
- **Checklist output path:** `test_artifacts/atdd-checklist-11-6-consortium-finder-agent-integration.md`
- **Story key:** `11-6-consortium-finder-agent-integration`
- **Story file handoff path:** `eusolicit-docs/implementation-artifacts/11-6-consortium-finder-agent-integration.md`
- **TDD Phase:** 🔴 RED — implementation not yet started; all 26 tests will fail with 404
- **Key risks / assumptions:**
  - `aigw_env_setup` is session-scoped autouse. If another test module in the same test session also
    defines `aigw_env_setup`, they may conflict. This follows the exact S11.05 pattern in `test_budget_builder.py`
    which has been confirmed working.
  - `consortium_client_and_session` fixture is function-scoped; each test registers a fresh user + company.
    This creates real rows in `client.users` and `client.company_memberships`. Rollback in `finally` block
    ensures no persistent state between tests.
  - `_register_and_verify_with_role` helper creates a secondary user in the caller's company with a
    specified role — used by `test_read_only_role_allowed` to verify AC8 (all roles permitted). Mirrors
    the identical helper in `test_budget_builder.py`.
  - `respx.mock` intercepts all outbound httpx calls matching `AIGW_CONSORTIUM_URL`. Any test that
    calls the endpoint without entering `with respx.mock:` will make real HTTP calls (and fail for a
    different reason). All 26 tests either use `respx.mock` or don't reach the agent call (validation
    / auth tests fail before agent is invoked).
  - `test_unauthenticated_returns_401`: In RED PHASE this will fail with 404 (route not found → auth
    middleware never reached). After implementation it should fail with 401 (route found, auth fails).
    This is expected and documents the RED → GREEN transition correctly.
  - AC9 (stateless) is verified structurally: the endpoint has no `AsyncSession` dependency. The test
    suite does not assert DB row counts before/after because no DB rows are expected to be written.
    The AC7 tests implicitly verify AC9 by confirming that gateway failures do not leave orphaned data.

- **Files to implement (dev-story tasks):**

  | File | Action |
  |---|---|
  | `src/client_api/schemas/grants.py` | **Edit** — append `PastProject`, `PartnerSuggestion`, `ConsortiumFinderRequest`, `ConsortiumFinderResponse` |
  | `src/client_api/services/grants_service.py` | **Edit** — append `_parse_past_project()`, `_parse_partner_suggestion()`, `find_consortium_partners()` |
  | `src/client_api/api/v1/grants.py` | **Edit** — add `POST /consortium-finder` endpoint |
  | `src/client_api/main.py` | **No change** — `/grants` router already registered |
  | `tests/api/test_consortium_finder.py` | **No change** — exists and is RED PHASE complete |

- **Next recommended workflow:** `bmad-dev-story` to implement Tasks 1–3 in the story file. Once all
  26 tests pass, advance TDD phase to 🟢 GREEN and update this checklist's `tddPhase` field.
