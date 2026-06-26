---
stepsCompleted: [step-01-preflight-and-context, step-04-generate-tests]
lastStep: 'step-04-generate-tests'
lastSaved: ''
workflowType: 'testarch-atdd'
storyId: '11.23'
storyKey: '11-23-kb-grounded-espd-and-grant-tests'
storyFile: '/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/11-23-kb-grounded-espd-and-grant-tests.md'
atddChecklistPath: 'eusolicit-docs/test-artifacts/atdd-checklist-11-23-kb-grounded-espd-and-grant-tests.md'
generatedTestFiles:
  - 'eusolicit-app/services/client-api/tests/integration/test_kb_grounded_agents.py'
inputDocuments: []
---

# ATDD Checklist - Epic 11, Story 11.23: KB-grounded ESPD and grant tests

**Date:** 2026-05-24
**Author:** BMad Test Architect
**Primary Test Level:** Integration (API)

---

## Story Summary

As a Developer, I want to add integration tests for the knowledge-base-grounded agent responses for ESPD and Grant functionalities, so that we can ensure the SirmaAI agent's responses are correctly handled and integrated into our system.

**As a** Developer
**I want** to add integration tests for the knowledge-base-grounded agent responses for ESPD and Grant functionalities
**So that** we can ensure the SirmaAI agent's responses are correctly handled and integrated into our system.

---

## Acceptance Criteria

1.  A new integration test suite is added at `services/client-api/tests/integration/test_kb_grounded_agents.py`.
2.  The new suite includes tests for the ESPD auto-fill agent endpoint (`/api/v1/agent/espd-auto-fill`), covering happy path, error cases (404, unauthorized), and graceful handling of external agent failures (503).
3.  The new suite includes tests for the grant eligibility check agent endpoint (`/api/v1/agent/grant-eligibility-check`), covering happy path, invalid payload (422), unauthorized access, and graceful handling of external agent failures.

---

## Red-Phase Test Scaffolds Created

### API Tests (8 tests)

**File:** `eusolicit-app/services/client-api/tests/integration/test_kb_grounded_agents.py`

- ✅ **Test:** `test_espd_auto_fill_happy_path`
  - **Status:** RED - Not yet implemented
  - **Verifies:** Correctly handles a successful response from the SirmaAI ESPD agent.
- ✅ **Test:** `test_espd_auto_fill_not_found`
  - **Status:** RED - Not yet implemented
  - **Verifies:** The endpoint returns a 404 for non-existent resources.
- ✅ **Test:** `test_espd_auto_fill_unauthorized`
  - **Status:** RED - Not yet implemented
  - **Verifies:** The endpoint requires authentication.
- ✅ **Test:** `test_espd_auto_fill_agent_error`
  - **Status:** RED - Not yet implemented
  - **Verifies:** The endpoint gracefully handles a 500 error from the SirmaAI agent.
- ✅ **Test:** `test_grant_eligibility_check_happy_path`
  - **Status:** RED - Not yet implemented
  - **Verifies:** Correctly handles a successful response from the SirmaAI Grant agent.
- ✅ **Test:** `test_grant_eligibility_check_invalid_payload`
  - **Status:** RED - Not yet implemented
  - **Verifies:** The endpoint returns a 422 for invalid request payloads.
- ✅ **Test:** `test_grant_eligibility_check_unauthorized`
  - **Status:** RED - Not yet implemented
  - **Verifies:** The endpoint requires authentication.
- ✅ **Test:** `test_grant_eligibility_check_agent_error`
  - **Status:** RED - Not yet implemented
  - **Verifies:** The endpoint gracefully handles a 500 error from the SirmaAI agent.

---

## Mock Requirements

### SirmaAI Agent Mock

**Endpoint:** `POST https://sirma-ai-api.example.com/api/v1/agent/espd-grounded-completion`
**Success Response:**
```json
{
  "content": "This is the auto-filled content from the KB."
}
```
**Failure Response:**
```json
{
  "error": "Internal Server Error"
}
```

**Endpoint:** `POST https://sirma-ai-api.example.com/api/v1/agent/grant-eligibility`
**Success Response:**
```json
{
  "is_eligible": true,
  "reasons": []
}
```
**Failure Response:**
```json
{
  "error": "Internal Server Error"
}
```
---

## Implementation Checklist

### Test: `test_espd_auto_fill_happy_path`
**File:** `eusolicit-app/services/client-api/tests/integration/test_kb_grounded_agents.py`
**Tasks to make this test pass:**
- [ ] Implement the `/api/v1/agent/espd-auto-fill` endpoint.
- [ ] Implement the `AgenticsaiProject` and `AgenticsaiKbFile` models.
- [ ] Implement the logic to call the SirmaAI agent.
- [ ] Run test: `make test-service SVC=client-api`
- [ ] ✅ Test passes (green phase)

### Test: `test_grant_eligibility_check_happy_path`
**File:** `eusolicit-app/services/client-api/tests/integration/test_kb_grounded_agents.py`
**Tasks to make this test pass:**
- [ ] Implement the `/api/v1/agent/grant-eligibility-check` endpoint.
- [ ] Implement the logic to call the SirmaAI agent for grant eligibility.
- [ ] Run test: `make test-service SVC=client-api`
- [ ] ✅ Test passes (green phase)

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

**TEA Agent Responsibilities:**
- ✅ All tests written as red-phase scaffolds with `pytest.fail()`.

**Verification:**
- All generated tests are present and will fail due to `pytest.fail("Not yet implemented")`.

### GREEN Phase (DEV Team - Next Steps)

**DEV Agent Responsibilities:**
1. Pick one scaffolded test.
2. Remove `pytest.fail()` and confirm it fails.
3. Implement minimal code to make that specific test pass.
4. Repeat for all tests.
