---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-18'
workflowType: bmad-testarch-atdd
mode: create
storyId: 7-7-compliance-risk-scoring-agent-integrations
detectedStack: backend
generationMode: ai-generation
executionMode: sequential
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-7-compliance-risk-scoring-agent-integrations.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/services/client-api/tests/api/test_proposal_checklist.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 7.7 — Compliance, Risk & Scoring Agent Integrations

**Date:** 2026-04-18
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — All tests will FAIL until implementation is complete
**Story Status:** ready-for-dev
**Test File:** `services/client-api/tests/api/test_proposal_compliance_risk_scoring.py`

---

## TDD Red Phase Summary

| Metric | Value |
|--------|-------|
| **TDD Phase** | 🔴 RED (failing tests) |
| **Test File** | `test_proposal_compliance_risk_scoring.py` |
| **Total Tests** | 29 |
| **E2E Tests** | 0 (backend-only story) |
| **Test Framework** | pytest + pytest-asyncio + respx |
| **All tests will fail until** | Implementation complete (see prerequisites below) |

---

## Prerequisites for RED → GREEN Transition

The following must be implemented before tests can pass:

| # | File | Change Required |
|---|------|----------------|
| 1 | `services/client-api/alembic/versions/023_proposal_agent_results_columns.py` | **CREATE** — Alembic migration adding 3 JSONB columns to `client.proposals` |
| 2 | `services/client-api/src/client_api/models/proposal.py` | **MODIFY** — Add 3 nullable JSONB `Mapped[dict | None]` columns |
| 3 | `services/client-api/src/client_api/schemas/proposal_agent_results.py` | **CREATE** — Pydantic schemas: `ComplianceCriterion`, `ComplianceCheckRequest/Response`, `ClauseRiskItem`, `ClauseRiskRequest/Response`, `ScoreCardCriterion`, `ScoringSimulationResponse`, `NullResultResponse` |
| 4 | `services/client-api/src/client_api/services/proposal_service.py` | **MODIFY** — Add `_build_agent_payload_base`, `run_compliance_check`, `run_clause_risk`, `run_scoring_simulation`, `get_compliance_check_result`, `get_clause_risk_result`, `get_scoring_simulation_result` |
| 5 | `services/client-api/src/client_api/api/v1/proposals.py` | **MODIFY** — Add 6 new endpoints (3 POST + 3 GET) with proper role enforcement |

---

## Acceptance Criteria Coverage

### AC1 — POST /compliance-check

> `POST /api/v1/proposals/{proposal_id}/compliance-check` invokes `compliance-checker` agent via `AiGatewayClient.run_agent()`. Requires `bid_manager` role; returns 404 for cross-company or non-existent. Optional `framework_id` loads framework data. Returns HTTP 200 with criteria list and `checked_at` timestamp.

| Test ID | Test Name | Class | Priority | Epic ID |
|---------|-----------|-------|----------|---------|
| AC1-001 | `test_compliance_check_stores_and_retrieves_results` | `TestComplianceCheck` | P1 | E07-P1-012 |
| AC1-002 | `test_compliance_check_with_framework_id` | `TestComplianceCheck` | P1 | — |
| AC1-003 | `test_compliance_check_cross_company_returns_404` | `TestComplianceCheck` | P0 | E07-P0-005 |
| AC1-004 | `test_compliance_check_nonexistent_proposal_returns_404` | `TestComplianceCheck` | P1 | — |
| AC1-005 | `test_compliance_check_requires_authentication` | `TestComplianceCheck` | P1 | — |

**Coverage:** ✅ Nominal path | ✅ framework_id loading | ✅ Cross-company 404 | ✅ Non-existent 404 | ✅ Auth enforcement

---

### AC2 — POST /clause-risk

> `POST /api/v1/proposals/{proposal_id}/clause-risk` invokes `clause-risk-analyzer` agent. Requires `bid_manager` role; returns 404 for cross-company. Optional `document_ids: list[UUID]`. `risk_level` validated against enum; unknown values coerced to `"medium"`. Result stored in `clause_risk_result` JSONB.

| Test ID | Test Name | Class | Priority | Epic ID |
|---------|-----------|-------|----------|---------|
| AC2-001 | `test_clause_risk_stores_and_retrieves_clauses` | `TestClauseRisk` | P1 | E07-P1-013 |
| AC2-002 | `test_clause_risk_invalid_risk_level_coerced_to_medium` | `TestClauseRisk` | P1 | — |
| AC2-003 | `test_clause_risk_with_document_ids` | `TestClauseRisk` | P1 | — |
| AC2-004 | `test_clause_risk_cross_company_returns_404` | `TestClauseRisk` | P0 | E07-P0-005 |

**Coverage:** ✅ Nominal path | ✅ risk_level enum coercion | ✅ document_ids forwarding | ✅ Cross-company 404

---

### AC3 — POST /scoring-simulation

> `POST /api/v1/proposals/{proposal_id}/scoring-simulation` invokes `scoring-simulator` agent. Requires `bid_manager` role; returns 404 for cross-company or non-existent. Payload includes proposal sections and opportunity evaluation criteria. Result stored in `scoring_simulation_result` JSONB.

| Test ID | Test Name | Class | Priority | Epic ID |
|---------|-----------|-------|----------|---------|
| AC3-001 | `test_scoring_simulation_stores_and_retrieves_scorecard` | `TestScoringSimulation` | P1 | E07-P1-014 |
| AC3-002 | `test_scoring_simulation_cross_company_returns_404` | `TestScoringSimulation` | P0 | E07-P0-005 |
| AC3-003 | `test_scoring_simulation_nonexistent_proposal_returns_404` | `TestScoringSimulation` | P1 | — |

**Coverage:** ✅ Nominal path | ✅ Cross-company 404 | ✅ Non-existent 404

---

### AC4 — GET endpoints (all 3)

> GET endpoints retrieve stored results. Any authenticated company user may call GET (no `bid_manager` requirement). Returns 404 for cross-company. Returns `HTTP 200` with `{"result": null}` if no result generated yet (never 404 for missing result).

| Test ID | Test Name | Class | Priority | Epic ID |
|---------|-----------|-------|----------|---------|
| AC4-001 | `test_get_compliance_check_no_result_returns_null` | `TestGetEndpoints` | P1 | — |
| AC4-002 | `test_get_clause_risk_no_result_returns_null` | `TestGetEndpoints` | P1 | — |
| AC4-003 | `test_get_scoring_simulation_no_result_returns_null` | `TestGetEndpoints` | P1 | — |
| AC4-004 | `test_get_compliance_check_cross_company_returns_404` | `TestGetEndpoints` | P0 | E07-P0-005 |
| AC4-005 | `test_get_clause_risk_cross_company_returns_404` | `TestGetEndpoints` | P0 | E07-P0-005 |
| AC4-006 | `test_get_scoring_simulation_cross_company_returns_404` | `TestGetEndpoints` | P0 | E07-P0-005 |

**Coverage:** ✅ Null result → 200 (not 404) | ✅ Cross-company 404 on all 3 GET endpoints

> **Note:** GET retrieve-after-POST is covered inline in AC1-001, AC2-001, and AC3-001.

---

### AC5 — Error Handling

> `AiGatewayTimeoutError` → HTTP 504 `{"error": "agent_timeout", "retry_after": 30}`. `AiGatewayUnavailableError` → HTTP 503 `{"error": "agent_unavailable"}`. Malformed agent response → HTTP 200 with empty list + warning log (no crash, no partial store).

| Test ID | Test Name | Class | Priority | Epic ID |
|---------|-----------|-------|----------|---------|
| AC5-001 | `test_compliance_check_timeout_returns_504` | `TestErrorHandling` | P2 | E07-P2-005 |
| AC5-002 | `test_compliance_check_gateway_unavailable_returns_503` | `TestErrorHandling` | P1 | — |
| AC5-003 | `test_compliance_check_malformed_response_returns_empty_criteria` | `TestErrorHandling` | P1 | — |
| AC5-004 | `test_clause_risk_malformed_response_returns_empty_clauses` | `TestErrorHandling` | P1 | — |
| AC5-005 | `test_scoring_simulation_malformed_response_returns_empty_criteria` | `TestErrorHandling` | P1 | — |
| AC5-006 | `test_clause_risk_timeout_returns_504` | `TestErrorHandling` | P2 | E07-R-007 |
| AC5-007 | `test_scoring_simulation_timeout_returns_504` | `TestErrorHandling` | P2 | E07-R-007 |

**Coverage:** ✅ Timeout 504 (all 3 agents) | ✅ Unavailable 503 | ✅ Malformed response → empty list (all 3 agents)

---

### AC6 — Alembic Migration 023

> Migration `023_proposal_agent_results_columns` adds 3 nullable JSONB columns to `client.proposals`. `revision = "023_proposal_agent_results_columns"`, `down_revision = "022_proposal_checklist"`. Reversible.

| Test ID | Test Name | Class | Priority | Epic ID |
|---------|-----------|-------|----------|---------|
| AC6-001 | `test_proposals_has_compliance_check_result_column` | `TestMigrationSchema` | P2 | — |
| AC6-002 | `test_proposals_has_clause_risk_result_column` | `TestMigrationSchema` | P2 | — |
| AC6-003 | `test_proposals_has_scoring_simulation_result_column` | `TestMigrationSchema` | P2 | — |
| AC6-004 | `test_all_three_columns_default_to_null` | `TestMigrationSchema` | P2 | — |

**Coverage:** ✅ All 3 JSONB columns exist | ✅ Columns are nullable | ✅ Fresh proposals have NULL defaults

> **Note:** Migration reversibility (`alembic downgrade -1`) is manual verification — see Dev Notes in story file.

---

### AC7 — Integration Test Suite

> Integration tests at `tests/api/test_proposal_compliance_risk_scoring.py` covering nominal paths, malformed responses, cross-company 404, and AI Gateway timeout. All 131 existing tests must continue to pass.

| Covered By | Status |
|------------|--------|
| This entire test file (24 tests) | ✅ Authored |
| Regression: existing 131 tests unchanged | ✅ No existing files modified |

---

## Test Coverage Matrix

| Scenario | Test(s) | Priority | Epic ID | Status |
|----------|---------|----------|---------|--------|
| Compliance check: nominal POST + GET retrieve | AC1-001 | P1 | E07-P1-012 | 🔴 RED |
| Compliance check: framework_id payload | AC1-002 | P1 | — | 🔴 RED |
| Compliance check: cross-company 404 (POST) | AC1-003 | P0 | E07-P0-005 | 🔴 RED |
| Compliance check: non-existent 404 | AC1-004 | P1 | — | 🔴 RED |
| Compliance check: auth enforcement | AC1-005 | P1 | — | 🔴 RED |
| Clause risk: nominal POST + GET retrieve | AC2-001 | P1 | E07-P1-013 | 🔴 RED |
| Clause risk: invalid risk_level → "medium" | AC2-002 | P1 | — | 🔴 RED |
| Clause risk: document_ids forwarded | AC2-003 | P1 | — | 🔴 RED |
| Clause risk: cross-company 404 (POST) | AC2-004 | P0 | E07-P0-005 | 🔴 RED |
| Scoring sim: nominal POST + GET retrieve | AC3-001 | P1 | E07-P1-014 | 🔴 RED |
| Scoring sim: cross-company 404 (POST) | AC3-002 | P0 | E07-P0-005 | 🔴 RED |
| Scoring sim: non-existent 404 | AC3-003 | P1 | — | 🔴 RED |
| GET compliance-check: null result | AC4-001 | P1 | — | 🔴 RED |
| GET clause-risk: null result | AC4-002 | P1 | — | 🔴 RED |
| GET scoring-sim: null result | AC4-003 | P1 | — | 🔴 RED |
| GET compliance-check: cross-company 404 | AC4-004 | P0 | E07-P0-005 | 🔴 RED |
| GET clause-risk: cross-company 404 | AC4-005 | P0 | E07-P0-005 | 🔴 RED |
| GET scoring-sim: cross-company 404 | AC4-006 | P0 | E07-P0-005 | 🔴 RED |
| Timeout → 504 (compliance-check) | AC5-001 | P2 | E07-P2-005 | 🔴 RED |
| Unavailable → 503 (compliance-check) | AC5-002 | P1 | — | 🔴 RED |
| Malformed → empty (compliance-check) | AC5-003 | P1 | — | 🔴 RED |
| Malformed → empty (clause-risk) | AC5-004 | P1 | — | 🔴 RED |
| Malformed → empty (scoring-sim) | AC5-005 | P1 | — | 🔴 RED |
| Timeout → 504 (clause-risk) | AC5-006 | P2 | E07-R-007 | 🔴 RED |
| Timeout → 504 (scoring-sim) | AC5-007 | P2 | E07-R-007 | 🔴 RED |
| Migration: compliance_check_result column | AC6-001 | P2 | — | 🔴 RED |
| Migration: clause_risk_result column | AC6-002 | P2 | — | 🔴 RED |
| Migration: scoring_simulation_result column | AC6-003 | P2 | — | 🔴 RED |
| Migration: fresh proposal → all NULL | AC6-004 | P2 | — | 🔴 RED |

**Total: 29 tests (0 passing in RED phase)**

---

## Risk Coverage

| Risk ID | Description | Tests Covering |
|---------|-------------|---------------|
| **E07-R-003** | RLS bypass via cross-company ID enumeration | AC1-003, AC2-004, AC3-002, AC4-004, AC4-005, AC4-006 (6 tests for all 6 endpoints) |
| **E07-R-007** | AI agent cascade failure with no per-endpoint timeout | AC5-001, AC5-006, AC5-007 |

> **E07-P0-005 coverage:** All 6 new endpoints (3 POST + 3 GET) verified to return 404 (not 403) for cross-company access. Covers the E07-P0-005 extension for this story.

---

## Priority Summary

| Priority | Count | Tests |
|----------|-------|-------|
| **P0** | 6 | AC1-003, AC2-004, AC3-002, AC4-004, AC4-005, AC4-006 |
| **P1** | 13 | AC1-001, AC1-002, AC1-004, AC1-005, AC2-001, AC2-002, AC2-003, AC3-001, AC3-003, AC4-001, AC4-002, AC4-003, AC5-002 |
| **P2** | 10 | AC5-001, AC5-003, AC5-004, AC5-005, AC5-006, AC5-007, AC6-001, AC6-002, AC6-003, AC6-004 |
| **P3** | 0 | — |
| **Total** | **29** | — |

---

## Test Infrastructure

### Fixtures

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `aigw_url_env` | session (autouse) | Set `CLIENT_API_AIGW_BASE_URL=http://test-aigw:8000`; clear LRU cache |
| `agent_results_proposal` | function | Register user, create proposal, seed 3 sections; yield (client, session, token, proposal_id) |
| `_make_pipeline_session_override()` | helper fn | Mock pipeline session returning None for opportunity data |
| `_register_and_login_second_company()` | helper fn | Create Company B user; return JWT token for cross-company 404 tests |

### Mock Agent URLs

| Agent | Mock URL |
|-------|----------|
| `compliance-checker` | `POST http://test-aigw:8000/agents/compliance-checker/run` |
| `clause-risk-analyzer` | `POST http://test-aigw:8000/agents/clause-risk-analyzer/run` |
| `scoring-simulator` | `POST http://test-aigw:8000/agents/scoring-simulator/run` |

### Default Mock Responses

| Agent | Mock Data |
|-------|----------|
| `compliance-checker` | `_COMPLIANCE_CRITERIA`: 3 items (2 passed=True, 1 passed=False) |
| `clause-risk-analyzer` | `_CLAUSE_RISK_ITEMS`: 2 clauses (high, medium risk levels) |
| `scoring-simulator` | `_SCORING_CRITERIA`: 3 criteria (8.5/10, 7.0/10, 6.0/10) |

---

## Running the Tests

```bash
cd eusolicit-app/services/client-api

# Story 7.7 tests only (will all FAIL in RED phase):
pytest tests/api/test_proposal_compliance_risk_scoring.py -v

# Full regression suite (131 existing + 24 new):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py \
       tests/api/test_proposal_compliance_risk_scoring.py -v

# P0 tests only (cross-company isolation):
pytest tests/api/test_proposal_compliance_risk_scoring.py -v \
       -k "cross_company"
```

---

## TDD Transition Plan

### Current State: 🔴 RED

All 24 tests fail because the endpoints do not exist yet.

### GREEN Phase Steps

After implementation:

1. ✅ Run migration: `alembic upgrade head`
2. ✅ Verify ORM model: check `compliance_check_result`, `clause_risk_result`, `scoring_simulation_result` columns on `Proposal`
3. ✅ Run tests: `pytest tests/api/test_proposal_compliance_risk_scoring.py -v`
4. ✅ Fix any failures (implementation bugs, not test bugs)
5. ✅ Run full regression: all 131 + 29 = 160 tests must pass
6. ✅ Verify migration reversibility: `alembic downgrade -1` then `alembic upgrade head`

### REFACTOR Phase

After GREEN:
- Run `/bmad-testarch-automate` to expand coverage to edge cases not in ATDD scope
- Consider `pytest-cov` to verify ≥80% line coverage on new service functions

---

## Architecture Constraints Verified

Tests enforce the following critical constraints from the story:

| Constraint | Enforced By |
|------------|-------------|
| `run_agent()` NOT `stream_agent()` — synchronous calls | Mock URLs do not use SSE pattern; assert HTTP 200 JSON response |
| `company_id` exclusively from JWT (never request body) | Cross-company tests use Company B JWT with Company A proposal_id |
| Return 404 (not 403) for cross-company — prevents ID enumeration | All 6 cross-company tests assert `status_code == 404` |
| `session.flush()` never `session.commit()` | Verified by DB assertions within same session context |
| `bid_manager` for POST, `get_current_user` for GET | Auth tests (AC1-005) + GET endpoints accept any auth user |
| `AiGatewayTimeoutError` → 504 with `retry_after: 30` | AC5-001, AC5-006, AC5-007 |
| Malformed response → empty list + log warning (no crash) | AC5-003, AC5-004, AC5-005 |

---

## Assumptions and Dependencies

1. **conftest.py** provides `client_api_session_factory` and `test_redis_client` fixtures (same as `test_proposal_checklist.py`)
2. **`client.compliance_frameworks`** table exists (created in E11/Story 11-1); AC1-002 seeds a framework row directly via SQL
3. **`respx`** is available as a dev dependency (`pip install respx`)
4. **Pipeline session** is safely mockable (no real pipeline DB required in CI)
5. **LRU cache clearing** pattern from Story 7-6 lesson is applied: both `get_settings.cache_clear()` and `get_ai_gateway_client.cache_clear()` called in fixture setup/teardown

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
**Story:** 7.7 — Compliance, Risk & Scoring Agent Integrations
