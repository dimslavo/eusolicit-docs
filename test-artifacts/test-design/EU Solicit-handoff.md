---
title: 'TEA Test Design → BMAD Handoff Document'
version: '1.0'
workflowType: 'testarch-test-design-handoff'
inputDocuments: ['test-design-architecture.md', 'test-design-qa.md']
sourceWorkflow: 'testarch-test-design'
generatedBy: 'TEA Master Test Architect'
generatedAt: '2026-04-12T14:30:00Z'
projectName: 'EU Solicit'
---

# TEA → BMAD Integration Handoff

## Purpose

This document bridges TEA's test design outputs with BMAD's epic/story decomposition workflow (`create-epics-and-stories`). It provides structured integration guidance so that quality requirements, risk assessments, and test strategies flow into implementation planning.

## TEA Artifacts Inventory

| Artifact | Path | BMAD Integration Point |
| :--- | :--- | :--- |
| Test Design Architecture | `test-artifacts/test-design-architecture.md` | Epic quality requirements, architectural blockers |
| Test Design QA | `test-artifacts/test-design-qa.md` | Story acceptance criteria, test strategy |
| Risk Assessment | (embedded in test-design-architecture.md) | Epic risk classification, story priority |

## Epic-Level Integration Guidance

### Risk References
- **Multi-tenancy Security (R-01)**: Must be a P0 gate for the Auth/Tenant management epic.
- **Billing Accuracy (R-02)**: Must be a P0 gate for the Billing/Subscription epic.
- **AI Hallucinations (R-03)**: Must be a P1 gate for the AI Proposal epic.
- **Tender Sync Reliability (R-05)**: Must be a P1 gate for the Data/Sync epic.

### Quality Gates
- **Isolation Gate**: 100% pass on cross-tenant access tests.
- **Compliance Gate**: GDPR data residency verification (ASR-01).
- **Performance Gate**: Search latency < 500ms (ASR-03).

## Story-Level Integration Guidance

### P0/P1 Test Scenarios → Story Acceptance Criteria
- **Signup Story**: AC: "Verification of tenant isolation (UUID access block)."
- **Purchase Story**: AC: "Stripe webhook successful processing and subscription activation."
- **AI Proposal Story**: AC: "Proposal draft contains all required procurement tags from source tender."
- **Search Story**: AC: "Search results returned in < 500ms for dataset > 100k records."

### Data-TestId Requirements
- `signup-form`, `org-domain-input`, `purchase-btn`, `proposal-draft-content`, `search-input`, `search-result-item`.

## Risk-to-Story Mapping

| Risk ID | Category | P×I | Recommended Story/Epic | Test Level |
| :--- | :--- | :--- | :--- | :--- |
| **R-01** | SEC | 6 | Auth: Tenant Isolation | API/E2E |
| **R-02** | BUS | 6 | Billing: Stripe Integration | E2E |
| **R-03** | TECH | 6 | AI: Proposal Generation | E2E/API |
| **R-05** | DATA | 6 | Data: Tender Sync Service | API |
| **R-06** | SEC | 6 | AI: Prompt Hardening | API |

## Recommended BMAD → TEA Workflow Sequence

1. **TEA Test Design** (`TD`) → produces this handoff document.
2. **BMAD Create Epics & Stories** → consumes this handoff, embeds quality requirements.
3. **TEA ATDD** (`AT`) → generates acceptance tests per story.
4. **BMAD Implementation** → developers implement with test-first guidance.
5. **TEA Automate** (`TA`) → generates full test suite.
6. **TEA Trace** (`TR`) → validates coverage completeness.

## Phase Transition Quality Gates

| From Phase | To Phase | Gate Criteria |
| :--- | :--- | :--- |
| Test Design | Epic/Story Creation | All P0 risks (R-01, R-02) have mitigation strategy. |
| Epic/Story Creation | ATDD | Stories have AC extracted from P0/P1 test scenarios. |
| ATDD | Implementation | Failing acceptance tests exist for P0 signup and purchase. |
| Implementation | Release | 100% P0 pass rate + RLS verification complete. |
