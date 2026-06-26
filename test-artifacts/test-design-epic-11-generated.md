---
workflowStatus: 'completed'
totalSteps: 5
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
nextStep: ''
lastSaved: '2026-05-23'
---

# Test Design: Epic 11 - Compliance & Grants

**Date:** 2026-05-23
**Author:** Deb
**Status:** Draft

---

## Executive Summary

**Scope:** Epic-Level test design for Epic 11, covering automated compliance checks, ESPD generation, and EU grant budget validation.

**Risk Summary:**

- Total risks identified: 6
- High-priority risks (Score ≥6): 4
- Critical categories: BUS, DATA, SEC

**Coverage Summary:**

- P0 scenarios: 2 (~15-25 hours)
- P1 scenarios: 2 (~20-30 hours)
- P2/P3 scenarios: 2 (~10-15 hours)
- **Total effort**: ~45-70 hours

---

## Not in Scope

| Item                                     | Reasoning                                                                      | Mitigation                                                                                                          |
|------------------------------------------|--------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| **External Service Performance/SLA**     | Dependencies like SirmaAI are external. Their performance is out of scope. | Implement and test resilience patterns (circuit breaker, retries).                                                 |
| **Regulatory Framework Content**         | The correctness of the compliance rules themselves is assumed. We test the *application* of the rules. | Business stakeholders are responsible for validating the content of the regulatory frameworks.               |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description                                                                                                | P | I | Score | Mitigation                                                                                                                                                                                                                                                                                            |
|---------|----------|------------------------------------------------------------------------------------------------------------|---|---|-------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **BUS-03**  | BUS      | Incorrect EU grant co-financing calculations, especially with floating-point arithmetic (`_ARITHMETIC_TOLERANCE`), lead to rejected grant applications. | 3 | 3 | 9     | **Mitigation:** Exhaustive **Unit Tests** for all calculation logic, covering floating-point edge cases, boundary conditions, and the specific tolerance requirement. API-level **Integration Tests** with realistic budget data.                                                                      |
| **DATA-01** | DATA     | Incorrect data mapping from the application's data models to the generated ESPD XML/PDF format leads to non-compliant or invalid submissions. | 2 | 3 | 6     | **Mitigation:** API-level schema validation. **Unit Tests** for all data transformation logic. **E2E Tests** with Playwright to validate the final generated PDF content against the source data, asserting specific fields and text.                                     |
| **BUS-01**  | BUS      | Incorrect ZOP compliance validation logic (pass/fail/warning) from external services like SirmaAI leads to bidders submitting invalid proposals. | 2 | 3 | 6     | **Mitigation:** **Integration Tests** for the SirmaAI client using `respx` to mock various API responses (success, failure, warnings). **E2E Tests** for the full user flow, asserting that UI components like progress rings and accordions correctly reflect the validation results. |
| **SEC-01**  | SEC      | Sensitive proposal or company data is exposed via insecure endpoints in the new features, violating multi-tenancy rules. | 2 | 3 | 6     | **Mitigation:** Mandatory **cross-tenant negative tests** for all new endpoints as per project conventions. Security-focused code review ensuring `check_entity_access` is used correctly.                                                                                     |

### Medium-Priority Risks (Score 3-5)

| Risk ID | Category | Description                                                               | P | I | Score | Mitigation                                                                                                                                          |
|---------|----------|---------------------------------------------------------------------------|---|---|-------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| **TECH-01** | TECH     | The external SirmaAI service for compliance checks is unreliable, causing timeouts or frequent errors, degrading the user experience. | 2 | 2 | 4     | **Mitigation:** Dedicated **Integration Tests** to verify the implementation of resilience patterns (circuit breakers, retries) as specified in the project context, using `respx` to simulate network failures. |
| **UX-01**   | UX       | The UI for displaying complex compliance results or grant budget errors is confusing or fails to render correctly, leading to user frustration. | 2 | 2 | 4     | **Mitigation:** **E2E tests** that specifically assert that error and warning states are displayed clearly in the UI, including proper rendering of `<QueryGuard>` states and component-level error messages. |

---

## Test Coverage Plan

### Test Coverage Matrix

| Risk ID     | Scenario                                                                                       | Test Level  | Priority | Rationale & Key Assertions                                                                                                                              |
|-------------|------------------------------------------------------------------------------------------------|-------------|----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **BUS-03**  | **Grant Calc Logic:** Validate grant budget calculation with diverse inputs (zero, floats, tolerance boundaries). | Unit        | P0       | Mitigates critical calculation risk at the lowest level. Assertions will use `toBeCloseTo` for float comparisons.                                       |
| **SEC-01**  | **Security:** User from Company A cannot access/modify Company B's compliance, ESPD, or grant data. | Integration | P0       | Mitigates critical security risk. For all new endpoints, authenticate as User A, attempt to access User B's resources, and assert a 403/404 response. |
| **DATA-01** | **ESPD Generation:** Verify the generated ESPD PDF contains the correct data from the user's profile. | E2E         | P1       | Mitigates high data integrity risk. Trigger the ESPD download and verify the content of the downloaded file.                                          |
| **BUS-01**  | **Compliance Client:** Test the SirmaAI client's handling of various API responses (pass, fail, error). | Integration | P1       | Mitigates high business risk. Use `respx` to mock the external service and assert that the client processes the responses correctly.                |
| **TECH-01** | **Resilience:** The SirmaAI client correctly implements retry/circuit breaker patterns on network failure. | Integration | P2       | Mitigates medium technical risk. Use `respx` to simulate timeouts and 5xx errors to ensure resilience patterns engage as expected.                   |
| **UX-01**   | **UI Feedback:** API errors for grant/compliance checks are displayed clearly to the user in the UI. | E2E         | P2       | Mitigates medium UX risk. Mock API error responses and assert that user-friendly error messages and states appear correctly in the UI.             |

---

## Execution Strategy

- **On Pull Request:** All `Unit` and `Integration` tests will be executed to provide fast feedback.
- **Nightly Build:** All `E2E` tests will run against a deployed environment to ensure full user journeys are functional.

---

## Resource Estimates

- **P0 Scenarios:** ~15-25 hours
- **P1 Scenarios:** ~20-30 hours
- **P2 Scenarios:** ~10-15 hours
- **Total Estimated Effort:** 45-70 hours

---

## Quality Gates

- **P0 Test Pass Rate:** 100%
- **P1 Test Pass Rate:** >= 95%
- **Code Coverage (New Modules):** >= 80%
- All high-priority risks (Score >= 6) must have their corresponding mitigation tests implemented and passing before release.

---
**Generated by**: Gemini Test Architect Agent
**Workflow**: `bmad-testarch-test-design`
