---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-18'
workflowType: 'testarch-test-design'
inputDocuments: ['epic-07-proposal-generation.md', 'test-design-architecture.md']
---

# Test Design: Epic 07 - Proposal Generation & Document Intelligence

**Date:** 2026-04-18
**Author:** BMad TEA Agent
**Status:** Draft
**Project:** EU Solicit

---

## Executive Summary

**Scope:** Epic-level test design for Epic 07: Proposal Generation & Document Intelligence. Covers the end-to-end AI-powered proposal workspace, including real-time AI generation via SSE, rich text editing with auto-save, compliance and scoring agent integrations, and PDF/DOCX export capabilities.

**Risk Summary:**

- Total risks identified: 6
- High-priority risks (≥6): 4
- Critical categories: TECH, SEC, DATA

**Coverage Summary:**

- P0 scenarios: 12 (24 hours)
- P1 scenarios: 18 (18 hours)
- P2/P3 scenarios: 25 (12.5 hours)
- **Total effort**: 55 hours (~7 days)

---

## Not in Scope

| Item | Reasoning | Mitigation |
| :--- | :--- | :--- |
| **AI Model Quality Validation** | Validating the persuasiveness or linguistic quality of the AI generated text is subjective and outside deterministic test boundaries. | Addressed through Human-in-the-loop review and UAT phases. Automated testing focuses on structural correctness and data binding. |
| **External AI Gateway Provisioning** | Managing the lifecycle of the OpenAI/Anthropic backing services. | We assume the AI Gateway (E04) provides reliable access or mocked endpoints for test environments. |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-001 | SEC | Data Leakage: Cross-tenant access to proposals or content blocks. | 2 | 3 | 6 | E2E Isolation Suite testing RLS policies and API scoping for `company_id`. | Security Lead | Sprint 7 |
| R-002 | TECH | AI SSE Stream Interruptions: Network drops or timeout during real-time section generation. | 3 | 2 | 6 | Mocked SSE failures; test auto-resume and partial save states in the UI. | Frontend Lead | Sprint 8 |
| R-003 | DATA | Content Overwrites: Concurrent edits or stale clients overwriting proposal data. | 2 | 3 | 6 | Enforce optimistic locking (version/hash) on `PATCH/PUT` content endpoints. | Backend Lead | Sprint 7 |
| R-004 | TECH | AI Output Determinism: Brittleness in CI due to unpredictable agent responses. | 3 | 2 | 6 | Implement a test mode/mocking strategy for the AI Gateway to return fixed fixtures. | AI Lead | Sprint 7 |

### Medium-Priority Risks (Score 3-4)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-005 | PERF | Document Export Timeouts: Large proposal payload causing PDF/DOCX generation to fail. | 2 | 2 | 4 | Load test export APIs; stream responses; add background processing if necessary. | Backend Lead | Sprint 8 |

### Low-Priority Risks (Score 1-2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-006 | BUS | Visual glitches in Tiptap editor during SSE streaming. | 2 | 1 | 2 | Monitor |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

## Entry Criteria

- [ ] AI Gateway (E04) endpoints are accessible in the test environment.
- [ ] Database schema migrations for `proposals`, `proposal_versions`, and `content_blocks` are applied.
- [ ] Mock responses for all AI agents (Compliance, Clause Risk, Scoring, Pricing, Win Themes) are available.

## Exit Criteria

- [ ] All P0 tests passing.
- [ ] All P1 tests passing (or failures triaged).
- [ ] Real-time SSE AI generation works seamlessly in E2E environments.
- [ ] Multi-tenant isolation verified with 100% pass rate.
- [ ] PDF and DOCX exports generate valid, parsable documents.

## Project Team (Optional)

| Name | Role | Testing Responsibilities |
| :--- | :--- | :--- |
| QA Team | Test Engineers | E2E test implementation, API contract testing, AI Agent mock validation. |
| Dev Team | Backend/Frontend | Unit tests, SSE integration tests, Tiptap editor component tests. |

---

## Test Coverage Plan

### P0 (Critical) - Run on every commit

**Criteria**: Blocks core journey + High risk (≥6) + No workaround

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Proposal CRUD & RLS Access | API | R-001 | 4 | QA | Ensure strict tenant isolation |
| AI Draft Generation via SSE | E2E | R-002, R-004 | 3 | QA | Mock SSE stream and verify UI updates |
| Editor Auto-save & Locking | API/E2E | R-003 | 3 | QA | Test HTTP 409 conflict responses |
| Agent Integrations (Compliance, Scoring) | API | - | 2 | QA | Validate mock agent data binding |

**Total P0**: 12 tests, 24.0 hours

### P1 (High) - Run on PR to main

**Criteria**: Important features + Medium risk (3-4) + Common workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Version History & Rollback | API/E2E | - | 5 | QA | Validate section diffs and version restores |
| Content Blocks CRUD & Search | API | R-001 | 4 | QA | Test TSVector full-text search and RLS |
| Proposal Export (PDF/DOCX) | API | R-005 | 4 | QA | Ensure output files are valid formats |
| Tiptap Formatting & Insertion | Component | - | 5 | DEV | Test rich text mechanics and block insertion |

**Total P1**: 18 tests, 18.0 hours

### P2 (Medium) - Run nightly/weekly

**Criteria**: Secondary features + Low risk (1-2) + Edge cases

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Clause Risk & Pricing Agents | API | - | 5 | QA | Validate metadata storage and retrieval |
| Requirement Checklist Toggle | API/UI | - | 5 | DEV | Test optimistic UI updates |
| Editor SSE Visual Glitches | UI | R-006 | 5 | DEV | Ensure cursor position is maintained |
| Archiving & Soft Deletes | API | - | 10 | QA | Verify archived proposals hide from default lists |

**Total P2**: 25 tests, 12.5 hours

---

## Execution Order

### Smoke Tests (<5 min)

**Purpose**: Fast feedback, catch build-breaking issues

- [ ] Create Proposal via API (30s)
- [ ] Read Proposal and Active Version (30s)
- [ ] Trigger AI Generation Endpoint (mocked) (45s)

**Total**: 3 scenarios

### P0 Tests (<10 min)

**Purpose**: Critical path validation

- [ ] E2E: User generates proposal draft via SSE, accepts all, and views content.
- [ ] API: Cross-tenant unauthorized access attempt on `GET /proposals/:id`.
- [ ] API: Trigger auto-save conflict and verify 409 response.

**Total**: 12 scenarios

### P1 Tests (<30 min)

**Purpose**: Important feature coverage

- [ ] E2E: User views version history, compares diffs, and rolls back to v1.
- [ ] API: Export proposal to PDF and validate content structure (if JSON endpoint available as per System Test Design).
- [ ] API: Search content blocks by tag and insert into proposal.

**Total**: 18 scenarios

### P2/P3 Tests (<60 min)

**Purpose**: Full regression coverage

- [ ] API: Clause Risk Analyzer full lifecycle.
- [ ] UI: Verify Pricing Simulator radar chart rendering.

**Total**: 25 scenarios

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test | Total Hours | Notes |
| :--- | :--- | :--- | :--- | :--- |
| P0 | 12 | 2.0 | 24.0 | Complex SSE/Locking setups |
| P1 | 18 | 1.0 | 18.0 | Standard API/UI flows |
| P2/P3 | 25 | 0.5 | 12.5 | Edge cases |
| **Total** | **55** | **-** | **54.5** | **~7 days** |

### Prerequisites

**Test Data:**

- User and Organization factory with valid session tokens.
- Mock Opportunity records and Company Profile setups.
- Content Block seed library.

**Tooling:**

- Playwright for E2E and visual regression (SSE streaming UI).
- Playwright Utils (Auth, Recurse).
- Mocking server for AI Gateway streaming responses.

**Environment:**

- Clean database state per test suite.
- Mock Stripe integration (from System Architecture Blocker B-01).

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate**: 100% (no exceptions)
- **P1 pass rate**: ≥95% (waivers required for failures)
- **P2/P3 pass rate**: ≥90% (informational)
- **High-risk mitigations**: 100% complete or approved waivers

### Coverage Targets

- **Critical paths**: ≥80%
- **Security scenarios**: 100%
- **Business logic**: ≥70%
- **Edge cases**: ≥50%

### Non-Negotiable Requirements

- [ ] All P0 tests pass
- [ ] No high-risk (≥6) items unmitigated
- [ ] Security tests (SEC category) pass 100%
- [ ] Multi-tenant RLS isolation tests pass 100%

---

## Mitigation Plans

### R-001: Data Leakage: Cross-tenant access (Score: 6)

**Mitigation Strategy:** Implement DB-level Row Level Security (RLS). QA will develop an automated API suite that attempts to read, write, and list `proposals` and `content_blocks` using tokens from a different `company_id`.
**Owner:** Security Lead
**Timeline:** Sprint 7
**Status:** Planned
**Verification:** Automated E2E isolation suite must pass with 100% success.

### R-002: AI SSE Stream Interruptions (Score: 6)

**Mitigation Strategy:** Build robust UI error handling for broken EventSource connections, ensuring generated chunks are retained and allowing the user to resume or discard.
**Owner:** Frontend Lead
**Timeline:** Sprint 8
**Status:** Planned
**Verification:** Induce network failure during Playwright SSE test and verify UI state recovery.

### R-003: Content Overwrites (Score: 6)

**Mitigation Strategy:** Implement version/hash-based optimistic locking on `PATCH` / `PUT` endpoints.
**Owner:** Backend Lead
**Timeline:** Sprint 7
**Status:** Planned
**Verification:** API tests mimicking concurrent updates must verify the second request receives a 409 Conflict.

### R-004: AI Output Determinism (Score: 6)

**Mitigation Strategy:** Use a test mode that bypasses actual LLM inference and returns static fixture data via SSE.
**Owner:** AI Lead
**Timeline:** Sprint 7
**Status:** Planned
**Verification:** Test environment configurations securely default to static mocked responses.

---

## Assumptions and Dependencies

### Assumptions

1. AI Gateway mock architecture is supported and capable of returning reliable fixture streams.
2. Content extraction from generated PDF/DOCX for test validation will use JSON alternative endpoints (from System Level Architecture recommendations) to avoid brittle parsing.

### Dependencies

1. AI Gateway Service (E04) - Must be stable for integration tests.
2. Tenant Auth/Seeding (System Blocker B-02) - Required for parallel test execution.

### Risks to Plan

- **Risk**: AI mocked outputs are overly simplistic and don't uncover real parsing errors in the Tiptap editor.
  - **Impact**: UI bugs discovered late in UAT.
  - **Contingency**: Include complex markdown and formatted text in the mock fixtures to stretch the editor's capabilities.

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
| :--- | :--- | :--- |
| **Auth Service** | Verifies token extraction and RLS claims. | Run all core Auth E2E tests against the new proposal endpoints. |
| **AI Gateway (E04)** | Primary orchestrator for prompt generation. | Verify prompt structure passes E04 guardrails. |
| **Search / Elastic (E06)**| Required for Content Blocks full text search. | Re-run E06 index validation tests. |

---

## Appendix

### Knowledge Base References

- `risk-governance.md` - Risk classification framework
- `probability-impact.md` - Risk scoring methodology
- `test-levels-framework.md` - Test level selection
- `test-priorities-matrix.md` - P0-P3 prioritization

### Related Documents

- PRD: EU_Solicit_PRD_v1.md
- Epic: epic-07-proposal-generation.md
- Architecture: EU_Solicit_Solution_Architecture_v4.md

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 5.0 (Step-File Architecture)