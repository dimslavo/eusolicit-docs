---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate']
lastStep: 'step-04c-aggregate'
lastSaved: '2026-04-18'
workflow: 'bmad-testarch-atdd'
storyId: '7-13-ai-draft-generation-panel-with-sse-streaming'
detectedStack: 'fullstack'
generationMode: 'ai'
executionMode: 'sequential'
testsGenerated:
  total: 11
  api: 4
  e2e: 1
  component: 6
tddPhase: 'RED'
---

# ATDD Checklist: Story 7.13 — AI Draft Generation Panel with SSE Streaming

**Date:** 2026-04-18
**Author:** TEA Master Test Architect
**Status:** Completed (Red Phase)

## 1. Summary & Next Steps

**ATDD Test Generation Complete (TDD RED PHASE)**

This process has generated a suite of failing acceptance tests that define the implementation requirements for Story 7.13. All generated tests are intentionally skipped (`test.skip` or `@pytest.mark.skip`) to prevent CI failures while the feature is in development.

### 1.1. Generated Artifacts

- **API Tests (Failing):** `eusolicit-app/tests/api/test_proposal_generation_atdd.py`
- **E2E Test (Failing):** `eusolicit-app/e2e/proposal_generation.spec.ts`
- **Component Tests (Failing):** `eusolicit-app/frontend/apps/client/src/components/AIDraftGenerationPanel.test.tsx`
- **Fixtures (Placeholder):** `eusolicit-app/tests/api/conftest.py`

### 1.2. Developer Workflow (TDD Green Phase)

1.  **Implement the Feature:** Build the necessary backend endpoints and frontend components to satisfy the user story.
2.  **Enable Tests:** Once the implementation is ready for testing, remove the `test.skip()` and `@pytest.mark.skip` decorators from the generated test files.
3.  **Run Tests:** Execute the test suite (`npm test` and `pytest`).
4.  **Verify PASS:** All tests should now pass. If any fail, it indicates either a bug in the implementation or an incorrect assumption in the test. Fix the code until all tests are green.
5.  **Commit:** Commit the implemented feature along with the now-passing acceptance tests.

## 2. Test Strategy & Coverage

The following table maps the story's acceptance criteria to the generated test scenarios.

| Priority | Test ID | Scenario | Test Level | Generated In |
|---|---|---|---|---|
| **P0** | E07-P0-012 | **Happy Path E2E:** User clicks "Generate Draft", sees progress, content streams into editor, and can accept the draft. | E2E (Playwright) | `proposal_generation.spec.ts` |
| **P1** | S13-C-001 | **Initial State:** Panel renders with a visible and enabled "Generate Draft" button. | Component (Vitest) | `AIDraftGenerationPanel.test.tsx` |
| **P1** | S13-C-002 | **Loading State:** Clicking "Generate Draft" disables the button and shows a progress indicator/spinner. | Component | `AIDraftGenerationPanel.test.tsx` |
| **P1** | S13-C-003 | **Streaming State:** While generation is active, a "Stop Generating" button is visible and the editor is read-only. | Component | `AIDraftGenerationPanel.test.tsx` |
| **P1** | S13-C-004 | **Completion State:** On `done` event, progress indicator is hidden, "Accept/Discard" controls appear, and editor is writable. | Component | `AIDraftGenerationPanel.test.tsx` |
| **P1** | S13-I-001 | **API Trigger:** Clicking "Generate Draft" triggers a `POST` to `/api/v1/proposals/{id}/generate`. | API (Pytest) | `test_proposal_generation_atdd.py` |
| **P1** | S13-I-002 | **API Conflict:** Attempting to generate while a job is already running results in a 409 Conflict. | API (Pytest) | `test_proposal_generation_atdd.py` |
| **P2** | S13-C-005 | **Stop Action:** Clicking "Stop Generating" returns the panel to its initial state. | Component | `AIDraftGenerationPanel.test.tsx` |
| **P2** | S13-C-006 | **API Error Handling:** Mocks a 409/503 API response; asserts a toast notification is displayed. | Component | `AIDraftGenerationPanel.test.tsx` |
| **P2** | S13-I-004 | **API Auth:** Unauthenticated requests to the generate endpoint are rejected with a 401. | API (Pytest) | `test_proposal_generation_atdd.py` |
| **P2** | S13-I-005 | **API Not Found:** Requests to a non-existent proposal return a 404. | API (Pytest) | `test_proposal_generation_atdd.py` |

## 3. Context & Knowledge Base

This plan was formulated using the following context:

- **Story File:** `7-13-ai-draft-generation-panel-with-sse-streaming.md`
- **Epic Test Design:** `test-design-epic-07.md`
- **Knowledge Base:** `data-factories.md`, `component-tdd.md`, `test-quality.md`, `test-healing-patterns.md`, `selector-resilience.md`, `timing-debugging.md`, `fixture-architecture.md`, `network-first.md`, `test-levels-framework.md`, `test-priorities-matrix.md`, `ci-burn-in.md`
