---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
lastStep: step-04-generate-tests
lastSaved: '2026-04-09'
workflowType: bmad-testarch-atdd
storyKey: 11-7-logframe-generator-reporting-template-agent-integrations
storyFile: eusolicit-docs/implementation-artifacts/11-7-logframe-generator-reporting-template-agent-integrations.md
tddPhase: RED
detectedStack: fullstack
testLevel: API/Integration (backend only)
inputDocuments:
  - eusolicit-docs/implementation-artifacts/11-7-logframe-generator-reporting-template-agent-integrations.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - eusolicit-app/services/client-api/tests/api/test_consortium_finder.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - _bmad/bmm/config.yaml
  - _bmad/tea/config.yaml
outputFiles:
  - eusolicit-app/services/client-api/tests/api/test_logframe_generator.py
  - eusolicit-app/services/client-api/tests/api/test_reporting_template.py
---

# ATDD Checklist: Story 11.7 — Logframe Generator & Reporting Template Agent Integrations

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — All tests are designed to FAIL until the story is implemented.
**Story:** `eusolicit-docs/implementation-artifacts/11-7-logframe-generator-reporting-template-agent-integrations.md`
**Test Files:**
- `eusolicit-app/services/client-api/tests/api/test_logframe_generator.py`
- `eusolicit-app/services/client-api/tests/api/test_reporting_template.py`

---

## Step 1: Preflight & Context Summary

### Stack Detection

| Indicator | Finding |
|-----------|---------|
| `eusolicit-app/services/client-api/` | Python backend (pytest, asyncpg, FastAPI) |
| `eusolicit-app/playwright.config.ts` | Frontend/E2E layer present |
| `eusolicit-app/services/client-api/pyproject.toml` | Backend project manifest |
| **Detected stack** | `fullstack` |
| **Test scope for this story** | `backend` only (API endpoints + DOCX export; no browser interaction) |

### Prerequisites ✅

- [x] Story 11.7 has clear acceptance criteria (AC1–AC12)
- [x] `conftest.py` with `client_api_session_factory`, `test_redis_client`, RSA fixtures
- [x] `respx` installed as dev dependency (added in S11.3)
- [x] `test_consortium_finder.py` pattern established (mirrors S11.6)
- [x] AI Gateway client (`AiGatewayClient`) already implemented from S11.4
- [x] `pytest-asyncio` and `httpx` available

### Config Flags Read

From `_bmad/tea/config.yaml`:
- `test_artifacts`: `{project-root}/eusolicit-docs/test-artifacts`
- `tea_use_playwright_utils`: `true` → not applicable (backend test, API-only profile)
- `tea_use_pactjs_utils`: `false` → not applicable
- `test_stack_type`: `auto` → auto-detected as `fullstack`; backend tests only for this story
- `tea_execution_mode`: `auto` → resolved as `sequential` (single main agent, no subagents)

### Loaded Context

- **Epic test design:** `test-design-epic-11.md` — confirms E11-P0-008, E11-P0-009, E11-P0-010,
  E11-P1-004, E11-P1-005, E11-P1-006, E11-P1-007, E11-P2-003 as relevant test IDs for this story
- **Risk links:** E11-R-001 (AI Gateway error handling), E11-R-008 (Logframe parser field completeness)
- **Pattern source:** `test_consortium_finder.py` — fixture structure, respx mock pattern,
  `aigw_env_setup` session fixture, client/session/token/company_id tuple

---

## Step 2: Generation Mode

**Mode selected:** AI Generation (Sequential)

**Reasoning:** Backend story with explicit, machine-readable acceptance criteria and well-defined
request/response contracts. Story file (Tasks 5 and 6) contains complete fixture response specs
and test method names. No browser interaction required. Pattern from `test_consortium_finder.py`
provides direct template. `tea_execution_mode: auto` resolved to `sequential` — single main agent.

**Special note — DB-reading endpoints:** S11.7 is the first story in the grants group to introduce
DB-reading endpoints (`/reporting-template`, `/reporting-template/export`). The fixture
`reporting_client_and_session` seeds a project row using `client.proposals` as a placeholder
table name. The dev must verify the actual table name and update the seed SQL before GREEN PHASE.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenarios Mapping

| AC | Description | Test File | Test Class | Priority |
|----|-------------|-----------|------------|---------|
| AC1 | `POST /logframe-generate` accepts `project_narrative` (required, min_length=1), optional `target_programme`; returns 200 | `test_logframe_generator.py` | `TestAC1AC2HappyPath`, `TestAC1InputValidation` | P0 |
| AC2 | Agent payload: `project_narrative`, `target_programme`, `company_id` (str); header `X-Caller-Service: client-api`; timeout 30s | `test_logframe_generator.py` | `TestAC1AC2HappyPath` | P0 |
| AC3 | `LogframeResponse` structure: all 4 fields with correct sub-types | `test_logframe_generator.py` | `TestAC3AC4ResponseParsing` | P1 |
| AC4 | `gantt_data: null` when key absent from agent response; `gantt_data: []` when key present but empty (distinct!) | `test_logframe_generator.py` | `TestAC3AC4ResponseParsing` | P1 |
| AC5 | Gateway timeout/5xx → HTTP 503 `AGENT_UNAVAILABLE`; raw error not forwarded | `test_logframe_generator.py` | `TestAC5AgentErrorHandling` | P0 |
| AC6 | Unauthenticated → 401; all roles allowed; stateless (no DB writes) | `test_logframe_generator.py` | `TestAC6Authorization` | P0 |
| AC7 | `POST /reporting-template` accepts `project_id` (UUID, required); returns 200 or 404 | `test_reporting_template.py` | `TestAC7AC8HappyPath`, `TestAC8NotFound`, `TestInputValidation` | P0 |
| AC8 | Service loads project from DB with company RLS; builds agent payload; calls `reporting-template-generator` | `test_reporting_template.py` | `TestAC7AC8HappyPath` | P1 |
| AC9 | `ReportingTemplateResponse` structure: `project_id`, `milestones` (list), `sections` (list) | `test_reporting_template.py` | `TestAC7AC8HappyPath` | P1 |
| AC10 | Gateway timeout/5xx → HTTP 503 `AGENT_UNAVAILABLE`; 404 propagates unchanged | `test_reporting_template.py` | `TestAC10AgentErrorHandling` | P0 |
| AC11 | `POST /reporting-template/export` returns DOCX; Content-Type + Content-Disposition correct | `test_reporting_template.py` | `TestAC11DOCX`, `TestAuthorization` | P1 |
| AC12 | DOCX includes heading, sections, milestones table, budget overview, consortium summary | `test_reporting_template.py` | `TestAC11DOCX` | P2 |

### Test Level Decisions

- **API/Integration only** — no unit tests generated at this stage.
  - Parsing logic (`_parse_logical_framework_row`, `_parse_gantt_task`, etc.) is best validated
    through full endpoint tests (respx mock → endpoint → parser → response assertion).
  - E2E tests (Gantt chart rendering, DOCX download UI) are deferred:
    E11-P2-014 (Logframe Panel E2E) and E11-P2-003 (DOCX frontend) are separate story scopes.
  - The AC4 gantt_data null/empty distinction is tested via two dedicated integration scenarios
    (`test_gantt_data_absent_returns_null_not_error` + `test_gantt_data_present_empty_list_not_null`)
    because this is a critical parsing constraint (E11-R-008) with high failure risk.

### Priority Assignments

**test_logframe_generator.py:**

| Priority | Count | Test IDs |
|----------|-------|---------|
| P0 | 7 | `test_logframe_generate_returns_200_with_complete_structure`, `test_logframe_agent_payload_has_required_fields`, `test_logframe_x_caller_service_header_sent`, `test_gateway_timeout_returns_503_agent_unavailable`, `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body`, `test_unauthenticated_returns_401` |
| P1 | 7 | `test_logframe_null_target_programme_forwarded`, `test_all_four_logframe_fields_present_in_response`, `test_logical_framework_rows_have_required_fields`, `test_work_packages_have_required_fields`, `test_gantt_tasks_have_required_fields`, `test_deliverables_have_required_fields`, `test_gantt_data_absent_returns_null_not_error` |
| P1 (critical) | 1 | `test_gantt_data_present_empty_list_not_null` (AC4 null/empty distinction — E11-R-008) |
| P2 | 2 | `test_missing_project_narrative_returns_422`, `test_empty_project_narrative_returns_422` |
| **Total** | **17** | 5 classes |

**test_reporting_template.py:**

| Priority | Count | Test IDs |
|----------|-------|---------|
| P0 | 7 | `test_reporting_template_returns_200_with_json_structure`, `test_reporting_template_gateway_timeout_returns_503`, `test_reporting_template_gateway_500_returns_503`, `test_export_gateway_timeout_returns_503`, `test_unauthenticated_returns_401`, `test_unauthenticated_export_returns_401`, `test_unknown_project_id_returns_404` |
| P1 | 8 | `test_reporting_template_milestones_have_required_fields`, `test_reporting_template_sections_have_required_fields`, `test_reporting_template_agent_payload_includes_project_data`, `test_reporting_template_x_caller_service_header_sent`, `test_export_returns_docx_content_type`, `test_export_has_content_disposition_attachment`, `test_export_body_is_non_empty`, `test_export_unknown_project_id_returns_404` |
| P2 | 3 | `test_export_is_valid_word_document`, `test_missing_project_id_returns_422`, `test_invalid_project_id_format_returns_422` |
| **Total** | **18** | 6 classes |

### Red Phase Requirements

All tests in both test files are designed to **fail** until:

**Logframe Generator (test_logframe_generator.py):**
1. `LogicalFrameworkRow`, `WorkPackage`, `GanttTask`, `DeliverableItem`, `LogframeRequest`, `LogframeResponse`
   added to `src/client_api/schemas/grants.py`
2. `_parse_logical_framework_row()`, `_parse_work_package()`, `_parse_gantt_task()`,
   `_parse_deliverable_item()`, `generate_logframe()` added to `grants_service.py`
3. `POST /logframe-generate` endpoint wired in `src/client_api/api/v1/grants.py`

**Reporting Template (test_reporting_template.py):**
1. `ReportingTemplateMilestone`, `ReportingTemplateSection`, `ReportingTemplateRequest`,
   `ReportingTemplateExportRequest`, `ReportingTemplateResponse` added to `grants.py`
2. `_load_project_data()`, `_parse_reporting_template_milestone()`, `_parse_reporting_template_section()`,
   `_parse_reporting_template_response()`, `generate_reporting_template()`,
   `build_report_docx()`, `export_reporting_template_docx()` added to `grants_service.py`
3. `POST /reporting-template/export` (before `/reporting-template`) and `POST /reporting-template`
   endpoints wired in `grants.py` (with `AsyncSession` dependency via `get_async_session`)
4. Actual project table name verified and fixture seed SQL updated

**Primary failure mode in RED phase:** All tests that POST to any of the 3 endpoints receive
`404 Not Found` instead of the expected status code. The `aigw_env_setup` fixture handles
`ImportError` gracefully so collection does not fail in RED phase.

---

## Step 4: Generated Tests

### Test File 1: Logframe Generator

**Path:** `eusolicit-app/services/client-api/tests/api/test_logframe_generator.py`
**TDD Phase:** 🔴 RED

#### Module-Level Setup

| Item | Value |
|------|-------|
| `AIGW_TEST_BASE_URL` | `"http://test-aigw.local:8000"` |
| `AIGW_LOGFRAME_URL` | `"{AIGW_TEST_BASE_URL}/agents/logframe-generator/run"` |
| `ENDPOINT` | `"/api/v1/grants/logframe-generate"` |
| `AGENT_UNAVAILABLE_CODE` | `"AGENT_UNAVAILABLE"` |
| `AGENT_UNAVAILABLE_MESSAGE` | `"AI features are temporarily unavailable. Please try again."` |

#### Mock Response Fixtures

| Fixture | Description | AC Coverage |
|---------|-------------|------------|
| `MOCK_FULL_LOGFRAME_RESPONSE` | All 4 fields populated; `gantt_data` key present with 2 tasks | AC1, AC3, E11-P0-009, E11-P1-004 |
| `MOCK_PARTIAL_LOGFRAME_RESPONSE` | `gantt_data` key **intentionally absent** | AC4, E11-P1-005, E11-R-008 |
| `MOCK_EMPTY_GANTT_RESPONSE` | `gantt_data` key present but value is `[]` | AC4 (empty ≠ null) |

#### Session-Scoped Fixtures

| Fixture | Scope | Description |
|---------|-------|-------------|
| `aigw_env_setup` | session, autouse | Sets `CLIENT_API_AIGW_BASE_URL`; clears `get_settings` + `get_ai_gateway_client` LRU caches; graceful ImportError in RED phase |

#### Function-Scoped Fixtures

| Fixture | Scope | Description |
|---------|-------|-------------|
| `logframe_client_and_session` | function | Register unique admin user + company → verify email → login → yield `(client, session, token, company_id_str)` → rollback |

#### Test Classes

---

### `TestAC1AC2HappyPath` — Happy Path: Response Structure + Agent Payload

**Coverage:** AC1, AC2, E11-P0-009

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 1 | `test_logframe_generate_returns_200_with_complete_structure` | 200 | AC1, AC3 | E11-P0-009 |
| 2 | `test_logframe_agent_payload_has_required_fields` | 200 | AC2 | E11-P0-009 |
| 3 | `test_logframe_x_caller_service_header_sent` | 200 | AC2 | E11-P0-009 |
| 4 | `test_logframe_null_target_programme_forwarded` | 200 | AC1, AC2 | — |

**Key Assertions:**
- `resp.status_code == 200`
- Response has `logical_framework` (len=2), `work_packages` (len=2), `gantt_data` (not None, len=2), `deliverable_table` (len=2)
- Captured payload has `project_narrative` (str), `target_programme` (str), `company_id` (str matching JWT)
- Header `x-caller-service: client-api` present in agent request
- `target_programme` is null in payload when omitted from request

**RED Phase failure mode:** `404 Not Found` (endpoint not yet created)

---

### `TestAC3AC4ResponseParsing` — Parsing + gantt_data null/empty distinction

**Coverage:** AC3, AC4, E11-P1-004, E11-P1-005, E11-R-008

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 5 | `test_all_four_logframe_fields_present_in_response` | 200 | AC3 | E11-P1-004 |
| 6 | `test_logical_framework_rows_have_required_fields` | 200 | AC3 | E11-P1-004 |
| 7 | `test_work_packages_have_required_fields` | 200 | AC3 | E11-P1-004 |
| 8 | `test_gantt_tasks_have_required_fields` | 200 | AC3 | E11-P1-004 |
| 9 | `test_deliverables_have_required_fields` | 200 | AC3 | E11-P1-004 |
| 10 | `test_gantt_data_absent_returns_null_not_error` | 200 | AC4 | E11-P1-005, E11-R-008 |
| 11 | `test_gantt_data_present_empty_list_not_null` | 200 | AC4 | E11-P1-005 |

**Key Assertions (test 10 — CRITICAL AC4):**
- Mock `MOCK_PARTIAL_LOGFRAME_RESPONSE` (no `gantt_data` key) → 200
- `response.json()["gantt_data"] is None` — **NOT an empty list**
- Other fields (`logical_framework`, `work_packages`, `deliverable_table`) still present

**Key Assertions (test 11):**
- Mock `MOCK_EMPTY_GANTT_RESPONSE` (`gantt_data: []`) → 200
- `response.json()["gantt_data"] == []` — **NOT null**

**RED Phase failure mode:** `404 Not Found`; even if endpoint existed, wrong parser default
(`.get("gantt_data", [])` instead of `.get("gantt_data")`) would fail test 10

---

### `TestAC5AgentErrorHandling` — AI Gateway Error Scenarios

**Coverage:** AC5, E11-P0-008, E11-P0-010, E11-R-001

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 12 | `test_gateway_timeout_returns_503_agent_unavailable` | 503 | AC5 | E11-P0-008, E11-P0-010 |
| 13 | `test_gateway_500_returns_503_not_500` | 503 | AC5 | E11-P0-008 |
| 14 | `test_gateway_503_returns_503_with_standard_body` | 503 | AC5 | E11-P0-008 |

**Key Assertions:**
- `resp.status_code == 503` for timeout, HTTP 500, and HTTP 503 from agent
- `body["code"] == "AGENT_UNAVAILABLE"`
- `body["message"] == "AI features are temporarily unavailable. Please try again."`
- Raw agent error body NOT forwarded

**Mock technique:** `respx.post(AIGW_LOGFRAME_URL).mock(side_effect=httpx.ReadTimeout(...))` for timeout

---

### `TestAC6Authorization` — Authentication

**Coverage:** AC6

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 15 | `test_unauthenticated_returns_401` | 401 | AC6 | — |

---

### `TestAC1InputValidation` — Pydantic Request Validation

**Coverage:** AC1

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 16 | `test_missing_project_narrative_returns_422` | 422 | AC1 | — |
| 17 | `test_empty_project_narrative_returns_422` | 422 | AC1 | — |

---

### Test File 2: Reporting Template Generator

**Path:** `eusolicit-app/services/client-api/tests/api/test_reporting_template.py`
**TDD Phase:** 🔴 RED

#### Module-Level Setup

| Item | Value |
|------|-------|
| `AIGW_TEST_BASE_URL` | `"http://test-aigw.local:8000"` |
| `AIGW_REPORTING_URL` | `"{AIGW_TEST_BASE_URL}/agents/reporting-template-generator/run"` |
| `ENDPOINT` | `"/api/v1/grants/reporting-template"` |
| `EXPORT_ENDPOINT` | `"/api/v1/grants/reporting-template/export"` |
| `DOCX_MIME_TYPE` | `"application/vnd.openxmlformats-officedocument.wordprocessingml.document"` |

#### Mock Response Fixtures

| Fixture | Description | AC Coverage |
|---------|-------------|------------|
| `MOCK_REPORTING_TEMPLATE_RESPONSE` | 2 milestones, 2 sections, budget_overview, consortium_summary | AC9, E11-P0-009, E11-P1-006, E11-P1-007 |

#### Function-Scoped Fixtures

| Fixture | Scope | Description |
|---------|-------|-------------|
| `reporting_client_and_session` | function | Register user + company → verify email → login → seed project row (placeholder table) → yield `(client, session, token, company_id_str, project_id_str)` → rollback |

#### Test Classes

---

### `TestAC7AC8HappyPath` — Happy Path: JSON Structure + Agent Payload

**Coverage:** AC7, AC8, AC9, E11-P0-009, E11-P1-006

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 1 | `test_reporting_template_returns_200_with_json_structure` | 200 | AC7, AC9 | E11-P0-009, E11-P1-006 |
| 2 | `test_reporting_template_milestones_have_required_fields` | 200 | AC9 | E11-P1-006 |
| 3 | `test_reporting_template_sections_have_required_fields` | 200 | AC9 | E11-P1-006 |
| 4 | `test_reporting_template_agent_payload_includes_project_data` | 200 | AC8 | E11-P0-009 |
| 5 | `test_reporting_template_x_caller_service_header_sent` | 200 | AC8 | E11-P0-009 |

**Key Assertions:**
- Response has `project_id` (str), `milestones` (list, len=2), `sections` (list, len=2)
- Captured payload has `project_id`, `company_id` (str), `milestones`, `budget_summary`
- Header `x-caller-service: client-api` present in agent request

---

### `TestAC11DOCX` — DOCX Export

**Coverage:** AC11, AC12, E11-P1-007, E11-P2-003

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 6 | `test_export_returns_docx_content_type` | 200 | AC11 | E11-P1-007 |
| 7 | `test_export_has_content_disposition_attachment` | 200 | AC11 | E11-P1-007 |
| 8 | `test_export_body_is_non_empty` | 200 | AC11 | E11-P1-007 |
| 9 | `test_export_is_valid_word_document` | 200 | AC12 | E11-P2-003 |

**Key Assertions:**
- `Content-Type` starts with `application/vnd.openxmlformats-officedocument.wordprocessingml.document`
- `Content-Disposition` contains `"attachment"` and `".docx"`
- Response body non-empty
- `Document(io.BytesIO(resp.content))` parses without exception; `len(doc.paragraphs) > 0`

---

### `TestAC10AgentErrorHandling` — AI Gateway Errors (Reporting Template)

**Coverage:** AC10, E11-P0-008, E11-P0-010

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 10 | `test_reporting_template_gateway_timeout_returns_503` | 503 | AC10 | E11-P0-008, E11-P0-010 |
| 11 | `test_reporting_template_gateway_500_returns_503` | 503 | AC10 | E11-P0-008 |
| 12 | `test_export_gateway_timeout_returns_503` | 503 | AC10, AC11 | E11-P0-008 |

---

### `TestAC8NotFound` — 404 for Unknown Project

**Coverage:** AC8, AC11

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 13 | `test_unknown_project_id_returns_404` | 404 | AC8 | — |
| 14 | `test_export_unknown_project_id_returns_404` | 404 | AC11 | — |

**Key Assertion:** `response.json()["detail"]` contains `"Project not found"` (AC8)

---

### `TestAuthorization` — Authentication

**Coverage:** AC7, AC11

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 15 | `test_unauthenticated_returns_401` | 401 | AC7 | — |
| 16 | `test_unauthenticated_export_returns_401` | 401 | AC11 | — |

---

### `TestInputValidation` — Pydantic Validation

**Coverage:** AC7

| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 17 | `test_missing_project_id_returns_422` | 422 | AC7 | — |
| 18 | `test_invalid_project_id_format_returns_422` | 422 | AC7 | — |

---

## Summary

### Test Count by Priority

**test_logframe_generator.py:**

| Priority | Count | Test Classes |
|----------|-------|-------------|
| P0 | 7 | `TestAC1AC2HappyPath` (3), `TestAC5AgentErrorHandling` (3), `TestAC6Authorization` (1) |
| P1 | 8 | `TestAC1AC2HappyPath` (1), `TestAC3AC4ResponseParsing` (7) |
| P2 | 2 | `TestAC1InputValidation` (2) |
| **Total** | **17** | 5 classes |

**test_reporting_template.py:**

| Priority | Count | Test Classes |
|----------|-------|-------------|
| P0 | 7 | `TestAC7AC8HappyPath` (1), `TestAC10AgentErrorHandling` (3), `TestAC8NotFound` (1), `TestAuthorization` (2) |
| P1 | 8 | `TestAC7AC8HappyPath` (4), `TestAC11DOCX` (3), `TestAC8NotFound` (1) |
| P2 | 3 | `TestAC11DOCX` (1), `TestInputValidation` (2) |
| **Total** | **18** | 6 classes |

**Grand Total: 35 tests across 2 files, 11 classes**

### Epic Test ID Coverage

| Epic Test ID | Priority | Description | Covered By |
|-------------|----------|-------------|-----------|
| **E11-P0-008** | P0 | All E11 agent-backed endpoints return `AGENT_UNAVAILABLE` on gateway 5xx | `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body`, `test_reporting_template_gateway_500_returns_503`, `test_export_gateway_timeout_returns_503` |
| **E11-P0-009** | P0 | AI Gateway mock: deterministic fixture responses in CI (smoke gate) | `test_logframe_generate_returns_200_with_complete_structure`, `test_logframe_agent_payload_has_required_fields`, `test_reporting_template_returns_200_with_json_structure`, `test_reporting_template_agent_payload_includes_project_data` |
| **E11-P0-010** | P0 | 30s timeout enforced (via ReadTimeout mock) | `test_gateway_timeout_returns_503_agent_unavailable`, `test_reporting_template_gateway_timeout_returns_503` |
| **E11-P1-004** | P1 | Logframe: all 4 output fields present | `test_all_four_logframe_fields_present_in_response`, `test_logical_framework_rows_have_required_fields`, `test_work_packages_have_required_fields`, `test_gantt_tasks_have_required_fields`, `test_deliverables_have_required_fields` |
| **E11-P1-005** | P1 | Logframe: gantt_data absent → null (not 500, not []) | `test_gantt_data_absent_returns_null_not_error`, `test_gantt_data_present_empty_list_not_null` |
| **E11-P1-006** | P1 | Reporting Template: project loaded from DB, agent called, JSON returned | `test_reporting_template_returns_200_with_json_structure`, `test_reporting_template_agent_payload_includes_project_data` |
| **E11-P1-007** | P1 | Reporting Template DOCX: correct MIME type, download succeeds | `test_export_returns_docx_content_type`, `test_export_has_content_disposition_attachment`, `test_export_body_is_non_empty` |
| **E11-P2-003** | P2 | Reporting Template DOCX parseable by python-docx | `test_export_is_valid_word_document` |

### Risk Coverage

| Risk ID | Description | Test Coverage |
|---------|-------------|--------------|
| **E11-R-001** | AI Gateway error handling (timeout, 5xx) across all agent endpoints | `TestAC5AgentErrorHandling` (3 tests), `TestAC10AgentErrorHandling` (3 tests) |
| **E11-R-008** | Logframe parser field completeness — gantt_data absent → null not crash | `test_gantt_data_absent_returns_null_not_error`, `test_gantt_data_present_empty_list_not_null` |

### Implementation Checklist (dev reference)

Before removing RED phase and marking GREEN:

**Schemas (`src/client_api/schemas/grants.py`):**
- [ ] `LogicalFrameworkRow(BaseModel)` added with `extra="ignore"`
- [ ] `WorkPackage(BaseModel)` added with `extra="ignore"`
- [ ] `GanttTask(BaseModel)` added with `extra="ignore"`
- [ ] `DeliverableItem(BaseModel)` added with `extra="ignore"`
- [ ] `LogframeRequest(BaseModel)` with `project_narrative: str = Field(..., min_length=1)`
- [ ] `LogframeResponse(BaseModel)` with `gantt_data: list[GanttTask] | None` (no default)
- [ ] `ReportingTemplateMilestone(BaseModel)` added with `extra="ignore"`
- [ ] `ReportingTemplateSection(BaseModel)` added with `extra="ignore"`
- [ ] `ReportingTemplateRequest(BaseModel)` with `project_id: UUID`
- [ ] `ReportingTemplateExportRequest(BaseModel)` with `project_id: UUID`
- [ ] `ReportingTemplateResponse(BaseModel)` added

**Service (`src/client_api/services/grants_service.py`):**
- [ ] `_parse_logical_framework_row()` with `.get()` defaults + try/except
- [ ] `_parse_work_package()` with `.get()` defaults + try/except
- [ ] `_parse_gantt_task()` with `.get()` defaults + try/except
- [ ] `_parse_deliverable_item()` with `.get()` defaults + try/except
- [ ] `generate_logframe()` — CRITICAL: `raw_gantt = agent_response.get("gantt_data")` (NO default)
- [ ] `_load_project_data()` — verify actual table name from models/ before implementing
- [ ] `_parse_reporting_template_milestone()` with `.get()` defaults + try/except
- [ ] `_parse_reporting_template_section()` with `.get()` defaults + try/except
- [ ] `_parse_reporting_template_response()` composite parser
- [ ] `generate_reporting_template()` with company RLS project load + agent call
- [ ] `build_report_docx()` using `python-docx` (verify `python-docx` in pyproject.toml)
- [ ] `export_reporting_template_docx()` reusing `generate_reporting_template` + `build_report_docx`

**Router (`src/client_api/api/v1/grants.py`):**
- [ ] `POST /logframe-generate` endpoint (stateless — no `AsyncSession`)
- [ ] `POST /reporting-template/export` registered **BEFORE** `/reporting-template` (routing order)
- [ ] `POST /reporting-template` endpoint (requires `AsyncSession` via `get_async_session`)
- [ ] `get_async_session` import path verified (check S11.02 ESPD CRUD router for correct path)

**Test fixtures:**
- [ ] `reporting_client_and_session` seed SQL updated with actual project table name

**Verification:**
- [ ] All 17 tests in `test_logframe_generator.py` pass
- [ ] All 18 tests in `test_reporting_template.py` pass
- [ ] `pytest -m integration tests/api/test_logframe_generator.py` exits 0
- [ ] `pytest -m integration tests/api/test_reporting_template.py` exits 0

---

*Generated by TEA Master Test Architect — BMAD ATDD workflow (bmad-testarch-atdd)*
*TDD Cycle: 🔴 RED → (implement) → 🟢 GREEN → (refactor) → 🔵 REFACTOR*
