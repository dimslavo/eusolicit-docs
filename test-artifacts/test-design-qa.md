---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-12'
workflowType: 'testarch-test-design'
---

# Test Design for QA: EU Solicit System-Level

**Purpose:** Test execution recipe for QA team. Defines what to test, how to test it, and what QA needs from other teams.

**Date:** 2026-04-12
**Author:** BMad TEA Agent
**Status:** Draft
**Project:** EU Solicit

**Related:** See Architecture doc (test-design-architecture.md) for testability concerns and architectural blockers.

---

## Executive Summary

**Scope:** Full system-level testing of EU Solicit SaaS platform.

**Risk Summary:**

- Total Risks: 6 (5 high-priority score ≥6, 1 medium, 0 low)
- Critical Categories: Security (Multi-tenancy), Business (Billing), Tech (AI Hallucinations).

**Coverage Summary:**

- P0 tests: ~5 (critical paths, security)
- P1 tests: ~7 (important features, integration)
- P2 tests: ~3 (edge cases, regression)
- P3 tests: ~1 (exploratory, benchmarks)
- **Total**: ~16 tests (~2-3 weeks with 2 QAs)

---

## Not in Scope

**Components or systems explicitly excluded from this test plan:**

| Item | Reasoning | Mitigation |
| :--- | :--- | :--- |
| **External LLM Model Training** | Beyond application scope | Rely on provider SLAs + Prompt hardening |
| **Physical Infrastructure** | Cloud-native deployment | Managed by DevOps/Platform team |
| **Legacy Procurement Portals** | CAIS/TED only for Phase 1 | Future integration scope |

---

## Dependencies & Test Blockers

**CRITICAL:** QA cannot proceed without these items from other teams.

### Backend/Architecture Dependencies (Pre-Implementation)

1. **Stripe Mocking/Simulator** - Fintech/Backend - Pre-Impl
   - Need API to simulate successful/failed payments and webhooks.
   - Blocks all Billing & Subscription tests.

2. **Tenant Data Seeding API** - Backend/Auth - Pre-Impl
   - Need endpoint to create/clear test organizations and users.
   - Blocks parallel E2E execution and isolation tests.

### QA Infrastructure Setup (Pre-Implementation)

1. **Test Data Factories** - QA
   - Organization/User/Tender/Proposal factories.
   - Auto-cleanup fixtures for parallel safety using Playwright Utils.

2. **Test Environments** - DevOps/QA
   - CI/CD: GitHub Actions with Kafka and Redis services.
   - Staging: Full environment with Stripe Test Mode.

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Score | QA Test Coverage |
| :--- | :--- | :--- | :--- | :--- |
| **R-01** | **SEC** | Data Leakage (Multi-tenancy) | **6** | Cross-tenant UUID access tests (API/E2E). |
| **R-02** | **BUS** | Billing Inaccuracy | **6** | Stripe webhook simulation + VAT unit tests. |
| **R-03** | **TECH** | AI Hallucinations | **6** | Semantic evaluation of proposal drafts. |
| **R-05** | **DATA** | Tender Sync Failure | **6** | `recurse` polling on Tender Sync API. |
| **R-06** | **SEC** | LLM Prompt Injection | **6** | Adversarial prompt injection suite. |

---

## Entry Criteria

- [ ] All requirements and assumptions agreed upon by QA, Dev, PM.
- [ ] Staging environment provisioned with Kafka and PostgreSQL.
- [ ] Test data factories ready for Organization and User entities.
- [ ] Pre-implementation blockers (Stripe Mock, Seeding API) resolved.
- [ ] Solution Architecture v4 signed off.

## Exit Criteria

- [ ] All P0 tests passing (100%).
- [ ] All P1 tests passing (≥ 95%).
- [ ] No open High/Critical security vulnerabilities.
- [ ] Row Level Security (RLS) verified for all tenant-scoped tables.
- [ ] Performance baseline: Search latency < 500ms (P95).

---

## Test Coverage Plan

### P0 (Critical)

| Test ID | Requirement | Test Level | Risk Link | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **P0-001** | Multi-tenant Signup | E2E | R-01 | Verify isolation of newly created tenant. |
| **P0-002** | Subscription Purchase | E2E | R-02 | Core revenue path validation. |
| **P0-003** | AI Proposal Generation | E2E | R-03 | Unique value prop validation. |
| **P0-004** | Cross-tenant Access Block | API | R-01 | Negative test: Attempt to access other tenant data. |
| **P0-005** | Tender Data Consistency | API | R-05 | Ensure sync'd data matches source spec. |

---

### P1 (High)

| Test ID | Requirement | Test Level | Risk Link | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **P1-001** | RBAC Enforcement | API | R-01 | Admin vs User vs Guest permissions. |
| **P1-002** | VAT Calculation | Unit/API | R-02 | Accuracy across different EU countries. |
| **P1-003** | Tender Search Performance| API | R-04 | Validate < 500ms P95 latency. |
| **P1-004** | Prompt Injection Guard | API | R-06 | Adversarial input validation. |
| **P1-005** | MFA Setup & Verification| E2E | ASR-04 | Critical security journey. |
| **P1-006** | AI Output Quality | API | R-03 | Semantic check for required procurement tags. |
| **P1-007** | Kafka Consumer Idempotency| API | R-05 | Re-process same event without duplicates. |

---

### P2 (Medium)

| Test ID | Requirement | Test Level | Risk Link | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **P2-001** | Proposal Export to PDF | API | - | Verify file generation and metadata. |
| **P2-002** | Analytics Dashboard | API | - | Verify aggregate counts match DB state. |
| **P2-003** | Profile Settings Update | E2E | - | Standard CRUD validation. |

---

### P3 (Low)

| Test ID | Requirement | Test Level | Notes |
| :--- | :--- | :--- | :--- |
| **P3-001** | UI Branding (White-label) | Component | Visual regression check for colors/logos. |

---

## Execution Strategy

### Every PR: Playwright Tests (~10 min)

- All P0 and P1 tests (Unit, API, and core E2E).
- Sharded across 4 workers in GitHub Actions.
- Total: ~12 tests.

### Nightly: k6 Performance & P2 (~30 min)

- Load testing for Search API.
- All P2 and P3 tests.
- Visual regression snapshots.

### Weekly: Security & Chaos (~2 hours)

- DAST scanning for injection flaws.
- Kafka fault injection (partition loss, lag).

---

## QA Effort Estimate

| Priority | Count | Effort Range | Notes |
| :--- | :--- | :--- | :--- |
| P0 | ~5 | ~40–60 hours | Includes complex isolation & billing setup. |
| P1 | ~7 | ~30–50 hours | API contracts and security hardening. |
| P2 | ~3 | ~10–20 hours | PDF and Analytics logic. |
| P3 | ~1 | ~5 hours | UI components. |
| **Total** | **16** | **~85–135 hours** | **Estimated 2 weeks for 2 QAs.** |

---

## Appendix A: Code Examples

**Cross-tenant Isolation Test Pattern:**

```typescript
import { test } from '@seontechnologies/playwright-utils/api-request/fixtures';
import { expect } from '@playwright/test';

test('@P0 @Security tenant B cannot access tenant A data', async ({ apiRequest, authToken }) => {
  // Assuming authToken belongs to Tenant B
  const tenantADataId = 'uuid-from-tenant-a';

  const { status } = await apiRequest({
    method: 'GET',
    path: `/api/proposals/${tenantADataId}`,
    // authToken (Tenant B) should be rejected
  });

  expect(status).toBe(403); // or 404 to avoid ID leaking
});
```

---

**Generated by:** BMad TEA Agent
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
