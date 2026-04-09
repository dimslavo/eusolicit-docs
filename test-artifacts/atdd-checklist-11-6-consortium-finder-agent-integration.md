---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
lastStep: step-04-generate-tests
lastSaved: '2026-04-09'
workflowType: bmad-testarch-atdd
storyKey: 11-6-consortium-finder-agent-integration
storyFile: eusolicit-docs/implementation-artifacts/11-6-consortium-finder-agent-integration.md
tddPhase: RED
detectedStack: fullstack
testLevel: API/Integration (backend only)
inputDocuments:
  - eusolicit-docs/implementation-artifacts/11-6-consortium-finder-agent-integration.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - eusolicit-app/services/client-api/tests/api/test_budget_builder.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - _bmad/bmm/config.yaml
  - resources/knowledge/data-factories.md
  - resources/knowledge/test-quality.md
  - resources/knowledge/test-healing-patterns.md
  - resources/knowledge/test-levels-framework.md
  - resources/knowledge/test-priorities-matrix.md
  - resources/knowledge/error-handling.md
outputFiles:
  - eusolicit-app/services/client-api/tests/api/test_consortium_finder.py
---

# ATDD Checklist: Story 11.6 — Consortium Finder Agent Integration

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — All tests are designed to FAIL until the story is implemented.
**Story:** `eusolicit-docs/implementation-artifacts/11-6-consortium-finder-agent-integration.md`
**Test File:** `eusolicit-app/services/client-api/tests/api/test_consortium_finder.py`

---

## Step 1: Preflight & Context Summary

### Stack Detection

| Indicator | Finding |
|-----------|---------|
| `eusolicit-app/services/client-api/` | Python backend (pytest, asyncpg, FastAPI) |
| `eusolicit-app/playwright.config.ts` | Frontend/E2E layer present |
| `eusolicit-app/services/client-api/pyproject.toml` | Backend project manifest |
| **Detected stack** | `fullstack` |
| **Test scope for this story** | `backend` only (stateless API endpoint, no browser interaction) |

### Prerequisites ✅

- [x] Story 11.6 has clear acceptance criteria (AC1–AC9)
- [x] `conftest.py` with `client_api_session_factory`, `test_redis_client`, RSA fixtures
- [x] `respx` installed as dev dependency (added in S11.3)
- [x] `test_budget_builder.py` pattern established (mirrors S11.5)
- [x] AI Gateway client (`AiGatewayClient`) already implemented from S11.4

### Config Flags Read

From `_bmad/bmm/config.yaml`:
- `test_artifacts`: `{project-root}/eusolicit-docs/test-artifacts`
- `tea_use_playwright_utils`: not set → not applicable (backend test)
- `tea_use_pactjs_utils`: not set → not applicable
- `test_stack_type`: not set → auto-detected as `fullstack`; backend tests only for this story

### Loaded Context

- **Epic test design:** `test-design-epic-11.md` — confirms E11-P0-008, E11-P0-009, E11-P0-010,
  E11-P1-003, E11-P2-002 as the relevant test IDs for this story
- **Risk links:** E11-R-001 (AI Gateway error handling), E11-R-012 (Consortium Finder field completeness)
- **Pattern source:** `test_budget_builder.py` — fixture structure, respx mock pattern,
  `_register_and_verify_with_role` helper, `aigw_env_setup` session fixture

---

## Step 2: Generation Mode

**Mode selected:** AI Generation

**Reasoning:** Backend story with explicit, machine-readable acceptance criteria and well-defined
request/response contracts. No browser interaction required. Complete fixture response specs provided
in the story file (Task 4.3–4.4). Pattern from `test_budget_builder.py` provides direct template.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenarios Mapping

| AC | Description | Test Class | Test Level | Priority |
|----|-------------|-----------|------------|---------|
| AC1 | `POST /api/v1/grants/consortium-finder` accepts required/optional fields, returns 200 | `TestAC1InputValidation`, `TestAC1AC2HappyPath` | API Integration | P0 |
| AC2 | Agent payload structure + `X-Caller-Service: client-api` header + timeout | `TestAC1AC2HappyPath` | API Integration | P0 |
| AC3 | `ConsortiumFinderResponse` structure: partners, total_results, page, page_size | `TestAC3AC5ResponseParsing` | API Integration | P1 |
| AC4 | Single-page contract: `page` always 1, `page_size == len(partners)` | `TestAC3AC5ResponseParsing` | API Integration | P1 |
| AC5 | Partners sorted descending by `collaboration_score` after parsing | `TestAC3AC5ResponseParsing` | API Integration | P1 |
| AC6 | Partial response graceful degradation: missing fields → null/[] not crash | `TestAC6GracefulDegradation` | API Integration | P2 |
| AC7 | Gateway timeout/5xx → 503 `AGENT_UNAVAILABLE`; raw error not forwarded | `TestAC7AgentErrorHandling` | API Integration | P0 |
| AC8 | Unauthenticated → 401; all roles permitted (no role restriction) | `TestAC8Authorization` | API Integration | P0 |
| AC9 | Stateless — no DB writes (implicit: endpoint has no `AsyncSession` dep) | (covered by architecture constraints in test fixture design) | — | — |

### Test Level Decisions

- **API/Integration only** — no unit tests generated at this stage.
  - The parsing logic (`_parse_partner_suggestion`, `_parse_past_project`) is best tested
    through the full endpoint test (respx mock → endpoint → parser → response assertion)
    rather than isolated unit tests. This follows the pattern established in S11.5.
  - E2E tests (browser) are deferred: the Consortium Finder Panel (story 11.6 frontend)
    is referenced in E11-P2-013 (epic-level E2E) and is a separate story scope.

### Priority Assignments

| Priority | Test Count | Test IDs |
|----------|-----------|---------|
| P0 | 9 | `test_consortium_finder_returns_200_with_response_structure`, `test_consortium_finder_agent_payload_has_required_fields`, `test_consortium_finder_x_caller_service_header_sent`, `test_gateway_timeout_returns_503_agent_unavailable`, `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body`, `test_unauthenticated_returns_401`, `test_missing_project_description_returns_422`, `test_missing_required_capabilities_returns_422` |
| P1 | 7 | `test_partners_returned_in_descending_collaboration_score_order`, `test_total_results_and_page_size_match_partner_count`, `test_partners_have_required_fields`, `test_single_country_filter_scenario`, `test_max_results_param_forwarded_to_agent`, `test_capability_overlap_ranking_reflected_in_scores`, `test_consortium_finder_forwards_target_countries` |
| P2 | 6 | `test_empty_results_returns_200_with_empty_list`, `test_partner_with_missing_contact_info_returns_null`, `test_partner_with_missing_past_projects_returns_empty_list`, `test_agent_response_missing_partners_key_defaults_to_empty_list`, `test_max_results_above_50_returns_422`, `test_max_results_below_1_returns_422` |
| P2 (auth) | 2 | `test_read_only_role_allowed`, `test_null_target_countries_accepted` (`test_consortium_finder_forwards_max_results` is P1) |

### Red Phase Requirements

All tests in `test_consortium_finder.py` are designed to **fail** until:

1. `PastProject`, `PartnerSuggestion`, `ConsortiumFinderRequest`, `ConsortiumFinderResponse`
   are added to `src/client_api/schemas/grants.py`
2. `_parse_past_project()`, `_parse_partner_suggestion()`, `find_consortium_partners()`
   are added to `src/client_api/services/grants_service.py`
3. `POST /consortium-finder` endpoint is wired in `src/client_api/api/v1/grants.py`

**Primary failure mode in RED phase:** All tests that POST to `ENDPOINT` receive `404 Not Found`
instead of the expected status code (200, 422, 401, 503). The `aigw_env_setup` fixture handles
ImportError gracefully so collection does not fail in RED phase.

---

## Step 4: Generated Tests

### Test File

**Path:** `eusolicit-app/services/client-api/tests/api/test_consortium_finder.py`
**TDD Phase:** 🔴 RED

### Test Inventory

#### Module-Level Setup

| Item | Value |
|------|-------|
| `AIGW_TEST_BASE_URL` | `"http://test-aigw.local:8000"` |
| `AIGW_CONSORTIUM_URL` | `"{AIGW_TEST_BASE_URL}/agents/consortium-finder/run"` |
| `ENDPOINT` | `"/api/v1/grants/consortium-finder"` |
| `AGENT_UNAVAILABLE_CODE` | `"AGENT_UNAVAILABLE"` |
| `AGENT_UNAVAILABLE_MESSAGE` | `"AI features are temporarily unavailable. Please try again."` |

#### Mock Response Fixtures

| Fixture | Description | AC Coverage |
|---------|-------------|------------|
| `MOCK_RANKED_PARTNERS_RESPONSE` | 3 partners in non-ranked agent order (85.0, 72.0, 91.0) → expect sorted (91.0, 85.0, 72.0) | AC3, AC5 |
| `MOCK_SINGLE_COUNTRY_RESPONSE` | 1 partner, country="BG" | AC1, E11-P1-003 |
| `MOCK_EMPTY_RESULTS_RESPONSE` | `{"partners": []}` — no matches | AC6, E11-P2-002 |
| `MOCK_PARTIAL_PARTNER_RESPONSE` | 1 partner, `contact_info` and `past_projects` keys absent | AC6, E11-R-012 |

#### Session-Scoped Fixtures

| Fixture | Scope | Description |
|---------|-------|-------------|
| `aigw_env_setup` | session, autouse | Sets `CLIENT_API_AIGW_BASE_URL`, clears `get_settings` + `get_ai_gateway_client` LRU caches; graceful ImportError on RED phase |
| `rsa_env_setup` | session, autouse (from conftest) | RSA key pair for JWT signing |
| `test_redis_client` | session (from conftest) | Redis DB index 1 for rate-limit isolation |

#### Function-Scoped Fixtures

| Fixture | Scope | Description |
|---------|-------|-------------|
| `consortium_client_and_session` | function | Register unique admin user + company → verify email → login → yield `(client, session, token, company_id_str)` → rollback |

#### Helper Functions

| Helper | Description |
|--------|-------------|
| `_register_and_verify_with_role(client, session, target_company_id, role)` | Creates secondary user with specified role in target company; returns `(access_token, user_id)` |

#### Test Classes and Methods

---

### `TestAC1AC2HappyPath` — Happy Path: Response Structure + Agent Payload

**Coverage:** AC1, AC2, E11-P0-009

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 1 | `test_consortium_finder_returns_200_with_response_structure` | 200 | AC1, AC3 | E11-P0-009 |
| 2 | `test_consortium_finder_agent_payload_has_required_fields` | 200 | AC2 | E11-P0-009 |
| 3 | `test_consortium_finder_x_caller_service_header_sent` | 200 | AC2 | E11-P0-009 |
| 4 | `test_consortium_finder_forwards_target_countries` | 200 | AC2 | E11-P1-003 |
| 5 | `test_consortium_finder_forwards_max_results` | 200 | AC2 | E11-P1-003 |

**Key Assertions:**
- `resp.status_code == 200`
- Response has `partners`, `total_results`, `page`, `page_size`
- `total_results == 3`, `page == 1`, `page_size == 3`
- Captured payload has `project_description`, `required_capabilities`, `company_id` (str), `max_results` (int)
- `target_countries is None` when not supplied; forwarded verbatim when supplied
- Header `x-caller-service: client-api` present in agent request

**RED Phase failure mode:** `404 Not Found` (endpoint not yet created)

---

### `TestAC3AC5ResponseParsing` — Parsing + Collaboration Score Ranking

**Coverage:** AC3, AC5, E11-P1-003

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 6 | `test_partners_returned_in_descending_collaboration_score_order` | 200 | AC5 | E11-P1-003 |
| 7 | `test_total_results_and_page_size_match_partner_count` | 200 | AC3, AC4 | E11-P1-003 |
| 8 | `test_partners_have_required_fields` | 200 | AC3 | E11-P1-003 |
| 9 | `test_single_country_filter_scenario` | 200 | AC1, AC3 | E11-P1-003 |
| 10 | `test_max_results_param_forwarded_to_agent` | 200 | AC2 | E11-P1-003 |
| 11 | `test_capability_overlap_ranking_reflected_in_scores` | 200 | AC5 | E11-P1-003 |

**Key Assertions:**
- `partners[0].collaboration_score == 91.0` (highest, "Paris Research Institute")
- `partners[1].collaboration_score == 85.0`
- `partners[2].collaboration_score == 72.0`
- Each partner: `organisation_name` (non-empty str), `collaboration_score` (float 0–100), `relevant_capabilities` (list), `past_projects` (list)
- `total_results == len(partners)`, `page_size == len(partners)`, `page == 1`
- Top partner matches all 3 requested capabilities when all overlap

**RED Phase failure mode:** `404 Not Found`; even if endpoint existed, sorting not implemented → score order wrong

---

### `TestAC6GracefulDegradation` — Partial Response Handling

**Coverage:** AC6, E11-P2-002, E11-R-012

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 12 | `test_empty_results_returns_200_with_empty_list` | 200 | AC6 | E11-P2-002 |
| 13 | `test_partner_with_missing_contact_info_returns_null` | 200 | AC6 | E11-P2-002, E11-R-012 |
| 14 | `test_partner_with_missing_past_projects_returns_empty_list` | 200 | AC6 | E11-P2-002 |
| 15 | `test_agent_response_missing_partners_key_defaults_to_empty_list` | 200 | AC6 | — |

**Key Assertions:**
- Empty `{"partners": []}` → 200 with `partners == []`, `total_results == 0`, `page_size == 0`
- Missing `contact_info` key in agent dict → `partner.contact_info is None` (NOT KeyError crash)
- Missing `past_projects` key in agent dict → `partner.past_projects == []` (NOT KeyError crash)
- Missing `partners` key in root → `partners == []`, `total_results == 0` (NOT 500)

**RED Phase failure mode:** `404 Not Found`; even if endpoint existed, missing `_parse_partner_suggestion` defensive defaults → `KeyError` → 500

---

### `TestAC7AgentErrorHandling` — AI Gateway Error Scenarios

**Coverage:** AC7, E11-P0-008, E11-P0-010 (E11-R-001 mitigation)

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 16 | `test_gateway_timeout_returns_503_agent_unavailable` | 503 | AC7 | E11-P0-008, E11-P0-010 |
| 17 | `test_gateway_500_returns_503_not_500` | 503 | AC7 | E11-P0-008 |
| 18 | `test_gateway_503_returns_503_with_standard_body` | 503 | AC7 | E11-P0-008 |

**Key Assertions:**
- `resp.status_code == 503` for timeout, HTTP 500, and HTTP 503 from agent
- `body["code"] == "AGENT_UNAVAILABLE"`
- `body["message"] == "AI features are temporarily unavailable. Please try again."`
- Raw agent error body NOT forwarded to caller (e.g. `"Internal agent error"` not in resp.text)

**Mock technique:** `respx.post(AIGW_CONSORTIUM_URL).mock(side_effect=httpx.ReadTimeout(...))` for timeout; `httpx.Response(500, ...)` / `httpx.Response(503, ...)` for HTTP error codes

**RED Phase failure mode:** `404 Not Found` (endpoint missing, respx never intercepts); error handling code not implemented

---

### `TestAC8Authorization` — Authentication and Role Access

**Coverage:** AC8

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 19 | `test_unauthenticated_returns_401` | 401 | AC8 | — |
| 20 | `test_read_only_role_allowed` | 200 | AC8 | — |

**Key Assertions:**
- No `Authorization` header → 401
- `read_only` role JWT → 200 (all roles permitted; no `require_role` gate)

**RED Phase failure mode:** `404 Not Found` for both; auth middleware never reached

---

### `TestAC1InputValidation` — Pydantic Request Validation

**Coverage:** AC1

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 21 | `test_missing_project_description_returns_422` | 422 | AC1 | — |
| 22 | `test_empty_required_capabilities_returns_422` | 422 | AC1 | — |
| 23 | `test_missing_required_capabilities_returns_422` | 422 | AC1 | — |
| 24 | `test_max_results_above_50_returns_422` | 422 | AC1 | — |
| 25 | `test_max_results_below_1_returns_422` | 422 | AC1 | — |
| 26 | `test_null_target_countries_accepted` | 200 | AC1 | — |

**Key Assertions:**
- Missing required fields → 422 (FastAPI Pydantic validation)
- `required_capabilities=[]` → 422 (`Field(..., min_length=1)`)
- `max_results=51` → 422 (`Field(..., le=50)`)
- `max_results=0` → 422 (`Field(..., ge=1)`)
- Omitted `target_countries` → 200; captured payload `target_countries is None`

**RED Phase failure mode:** `404 Not Found` for all; validation models not yet created

---

## Summary

### Test Count by Priority

| Priority | Count | Test Class(es) |
|----------|-------|---------------|
| P0 | 9 | `TestAC1AC2HappyPath` (3), `TestAC7AgentErrorHandling` (3), `TestAC8Authorization` (1 of 2), `TestAC1InputValidation` (2 of 6) |
| P1 | 7 | `TestAC3AC5ResponseParsing` (6), `TestAC1AC2HappyPath` (2 forwarding tests) |
| P2 | 10 | `TestAC6GracefulDegradation` (4), `TestAC8Authorization` (1), `TestAC1InputValidation` (4 of 6) |
| **Total** | **26** | 6 classes |

### Epic Test ID Coverage

| Epic Test ID | Priority | Description | Covered By |
|-------------|----------|-------------|-----------|
| **E11-P0-008** | P0 | `consortium-finder` returns `AGENT_UNAVAILABLE` on gateway 5xx | `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` |
| **E11-P0-009** | P0 | AI Gateway mock: consortium-finder agent returns deterministic fixture in CI | `test_consortium_finder_returns_200_with_response_structure`, `test_consortium_finder_agent_payload_has_required_fields` |
| **E11-P0-010** | P0 | 30s timeout enforced | `test_gateway_timeout_returns_503_agent_unavailable` (via `httpx.ReadTimeout` mock) |
| **E11-P1-003** | P1 | Paginated results; capability overlap ranking; single-country filter; max_results honoured | `test_partners_returned_in_descending_collaboration_score_order`, `test_single_country_filter_scenario`, `test_max_results_param_forwarded_to_agent`, `test_capability_overlap_ranking_reflected_in_scores`, `test_total_results_and_page_size_match_partner_count` |
| **E11-P2-002** | P2 | Empty results; `contact_info` absent → null, not crash | `test_empty_results_returns_200_with_empty_list`, `test_partner_with_missing_contact_info_returns_null` |

### Risk Coverage

| Risk ID | Description | Test Coverage |
|---------|-------------|--------------|
| **E11-R-001** | AI Gateway error handling (timeout, 5xx) | `TestAC7AgentErrorHandling` (3 tests) |
| **E11-R-012** | Consortium Finder field completeness (graceful degradation) | `TestAC6GracefulDegradation` (4 tests) |

### Implementation Checklist (dev reference)

Before removing RED phase and marking GREEN:

- [ ] `PastProject(BaseModel)` added to `src/client_api/schemas/grants.py`
- [ ] `PartnerSuggestion(BaseModel)` added with all fields + `extra="ignore"`
- [ ] `ConsortiumFinderRequest(BaseModel)` added with validation constraints
- [ ] `ConsortiumFinderResponse(BaseModel)` added
- [ ] `_parse_past_project()` added to `grants_service.py`
- [ ] `_parse_partner_suggestion()` added with `.get()` defensive defaults for all optional fields
- [ ] `find_consortium_partners()` added with agent call, error handling, sort-by-score, response build
- [ ] `POST /consortium-finder` endpoint added to `api/v1/grants.py` (all roles via `get_current_user`)
- [ ] All 26 tests in `test_consortium_finder.py` pass
- [ ] `pytest -m integration tests/api/test_consortium_finder.py` exits 0

---

*Generated by TEA Master Test Architect — BMAD ATDD workflow (bmad-testarch-atdd)*
*TDD Cycle: 🔴 RED → (implement) → 🟢 GREEN → (refactor) → 🔵 REFACTOR*
