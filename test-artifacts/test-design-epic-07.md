---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-18'
workflowType: 'testarch-test-design'
inputDocuments: ['epic-07-proposal-generation.md', 'test-design-architecture.md', 'config.yaml']
---

# Test Design: Epic 07 - Proposal Generation & Document Intelligence

**Date:** 2026-04-18
**Author:** BMad TEA Agent
**Status:** Draft

---

## Executive Summary

**Scope:** Epic-level test design for Epic 07

**Risk Summary:**

- Total risks identified: 6
- High-priority risks (≥6): 4
- Critical categories: TECH, SEC, DATA

**Coverage Summary:**

- P0 scenarios: 12 (24.0 hours)
- P1 scenarios: 18 (18.0 hours)
- P2/P3 scenarios: 25 (12.5 hours)
- **Total effort**: 54.5 hours (~7 days)

---

## Not in Scope

| Item | Reasoning | Mitigation |
| :--- | :--- | :--- |
| **AI Model Subjectivity Validation** | Evaluating text persuasiveness is subjective and non-deterministic. | Human-in-the-loop QA during UAT phase. |
| **Third-party AI Gateway Provisioning** | Out of scope for this epic's functionality testing. | Ensure Gateway provides mock/fixture mode for CI determinism. |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-001 | SEC | Cross-tenant data leakage in proposals and content blocks | 2 | 3 | 6 | E2E Isolation tests with RLS validation | Security Lead | Sprint 7 |
| R-002 | TECH | Non-deterministic AI causing CI test brittleness | 3 | 2 | 6 | AI test mode with static prompt snapshots/fixtures | AI Lead | Sprint 7 |
| R-003 | DATA | Concurrent edits overwriting proposal versions | 2 | 3 | 6 | Optimistic locking on PATCH/PUT and conflict tests | Backend Lead | Sprint 7 |
| R-004 | TECH | SSE Stream interruption causing partial data loss | 3 | 2 | 6 | Frontend retry logic and progressive chunk saving | Frontend Lead | Sprint 8 |

### Medium-Priority Risks (Score 3-4)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-005 | PERF | Document Export timeouts for large payloads | 2 | 2 | 4 | Background processing / streaming API + load testing | Backend Lead |

### Low-Priority Risks (Score 1-2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-006 | BUS | Visual layout shifts in Tiptap during SSE load | 2 | 1 | 2 | Monitor |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

## Entry Criteria

- [ ] AI Gateway (E04) endpoints are accessible and test fixtures are available.
- [ ] DB Migrations for `proposals` and `proposal_versions` are executed.
- [ ] Test environment data factory ready for Company and Opportunity contexts.

## Exit Criteria

- [ ] All P0 tests passing with 100% rate.
- [ ] All P1 tests passing (failures triaged/waivers).
- [ ] RLS / Security multi-tenant suite executed with zero leakages.

## Project Team (Optional)

| Name | Role | Testing Responsibilities |
| :--- | :--- | :--- |
| QA Team | Test Engineers | E2E scenarios, Security Isolation, API testing |
| Dev Team | Backend/Frontend | Unit testing, SSE stream component tests |

---

## Test Coverage Plan

### P0 (Critical) - Run on every commit

**Criteria**: Blocks core journey + High risk (≥6) + No workaround

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Proposal CRUD & RLS Policies | API | R-001 | 4 | QA | Strictly validate tenant isolation. |
| AI Draft Generation SSE (Backend) | API | R-002 | 3 | QA | Use mocked AI Gateway streams. |
| Auto-save Content + Full Save | API | R-003 | 3 | QA | Validate optimistic locking conflicts (409). |
| Tiptap SSE Real-time Rendering | E2E | R-004 | 2 | DEV | Test UI behavior during chunk stream. |

**Total P0**: 12 tests, 24.0 hours

### P1 (High) - Run on PR to main

**Criteria**: Important features + Medium risk (3-4) + Common workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Version History Diffs & Rollback | API/E2E | - | 5 | QA | Validate section diff accuracy. |
| Compliance, Scoring & Agent Mocks | API | - | 5 | QA | Validate metadata bindings. |
| Document Export (PDF/DOCX) | API | R-005 | 4 | QA | Test validity of exported formats. |
| Content Blocks CRUD & Insert | UI/API | R-001 | 4 | DEV | TSVector search validation. |

**Total P1**: 18 tests, 18.0 hours

### P2 (Medium) - Run nightly/weekly

**Criteria**: Secondary features + Low risk (1-2) + Edge cases

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Clause Risk & Pricing Panel UI | UI | - | 10 | DEV | Layout & formatting assertions. |
| Soft Delete / Archive API | API | - | 5 | QA | Filtering hidden records. |

**Total P2**: 15 tests, 7.5 hours

### P3 (Low) - Run on-demand

**Criteria**: Nice-to-have + Exploratory + Performance benchmarks

| Requirement | Test Level | Test Count | Owner | Notes |
| :--- | :--- | :--- | :--- | :--- |
| Export large proposal perf | E2E | 5 | QA | Load limits |
| Tiptap extreme formatting | UI | 5 | DEV | Formatting edge cases |

**Total P3**: 10 tests, 5.0 hours

---

## Execution Order

### Smoke Tests (<5 min)

**Purpose**: Fast feedback, catch build-breaking issues

- [ ] Create Proposal via API (30s)
- [ ] Read Active Version (30s)
- [ ] Trigger AI Draft mock endpoint (45s)

**Total**: 3 scenarios

### P0 Tests (<10 min)

**Purpose**: Critical path validation

- [ ] E2E: Create proposal, stream SSE sections, accept all
- [ ] API: Cross-tenant 403 authorization check
- [ ] API: Concurrent edit 409 conflict test

**Total**: 12 scenarios

### P1 Tests (<30 min)

**Purpose**: Important feature coverage

- [ ] E2E: Rollback to previous version
- [ ] API: Export PDF check
- [ ] API: Agent mock triggers (Compliance, Score)

**Total**: 18 scenarios

### P2/P3 Tests (<60 min)

**Purpose**: Full regression coverage

- [ ] API: Archive proposals
- [ ] UI: Win Themes and Pricing visualization

**Total**: 25 scenarios

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test | Total Hours | Notes |
| :--- | :--- | :--- | :--- | :--- |
| P0 | 12 | 2.0 | 24.0 | Includes SSE & Security |
| P1 | 18 | 1.0 | 18.0 | Standard API/UI flows |
| P2 | 15 | 0.5 | 7.5 | Edge cases |
| P3 | 10 | 0.5 | 5.0 | Exploratory |
| **Total** | **55** | **-** | **54.5** | **~7 days** |

### Prerequisites

**Test Data:**
- `tenant` factory with seeded DB isolation
- `mock_opportunities` with full requirements

**Tooling:**
- Playwright (UI)
- Playwright Utils (Auth, Recurse)

**Environment:**
- AI Gateway (E04) mocked

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

---

## Mitigation Plans

### R-001: Cross-tenant data leakage in proposals and content blocks (Score: 6)

**Mitigation Strategy:** RLS enabled in DB; E2E API tests cross-fetching tenant IDs.
**Owner:** Security Lead
**Timeline:** Sprint 7
**Status:** Planned
**Verification:** Automated E2E isolation suite 100% pass

### R-002: Non-deterministic AI causing CI test brittleness (Score: 6)

**Mitigation Strategy:** Provide mock fixtures via AI Gateway bypass.
**Owner:** AI Lead
**Timeline:** Sprint 7
**Status:** Planned
**Verification:** CI runs against mock endpoints explicitly.

---

## Assumptions and Dependencies

### Assumptions

1. AI Gateway mock architecture is supported in the dev environment.
2. PDF/DOCX rendering uses known parsable formatting, or JSON equivalents exist for assertions.

### Dependencies

1. AI Gateway Service (E04) - Required by Sprint 7
2. Auth Service Seeding - Required by Sprint 7

### Risks to Plan

- **Risk**: Editor rendering is unparsable by Playwright without extensive manual tags
  - **Impact**: Slow E2E test authoring
  - **Contingency**: Dev team strictly adds `data-testid` to Tiptap sections.

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
| :--- | :--- | :--- |
| **AI Gateway (E04)** | Proxy for all agent workflows | Prompt structure tests |
| **Auth Service** | RLS / JWT scoping | Cross-tenant API boundaries |
| **Search (E06)** | Content Blocks search | Full-text query tests |

---

## Appendix

### Knowledge Base References

- `risk-governance.md` - Risk classification framework
- `probability-impact.md` - Risk scoring methodology
- `test-levels-framework.md` - Test level selection
- `test-priorities-matrix.md` - P0-P3 prioritization

### Related Documents

- PRD: `EU_Solicit_PRD_v1.md`
- Epic: `epic-07-proposal-generation.md`
- Architecture: `EU_Solicit_Solution_Architecture_v4.md`

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 5.0 (Step-File Architecture)