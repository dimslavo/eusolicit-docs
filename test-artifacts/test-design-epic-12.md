---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-12'
---

# Test Design: Epic 12 - Analytics, Reporting & Admin Platform

**Date:** 2026-04-12
**Author:** Deb
**Status:** Draft

---

## Executive Summary

**Scope:** Epic-level test design for Epic 12 (Analytics, Reporting & Admin Platform)

**Risk Summary:**

- Total risks identified: 6
- High-priority risks (≥6): 4
- Critical categories: DATA, SEC, PERF

**Coverage Summary:**

- P0 scenarios: 3 (~20–30 hours)
- P1 scenarios: 4 (~35–50 hours)
- P2/P3 scenarios: 3 (~20–35 hours)
- **Total effort**: ~75–115 hours (~10–15 days)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|:---|:---|:---|
| **External SIEM Integration** | Out of scope for MVP admin platform. | Audit logs stored in internal Loki/PostgreSQL. |
| **Mobile App Admin Views** | Admin platform is desktop-first/responsive web only. | Responsive web testing on desktop browsers. |
| **Real-time Global Benchmarking** | Aggregating data across all tenants is Phase 2. | Analytics restricted to per-tenant data isolation. |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|:---|:---|:---|:---:|:---:|:---:|:---|:---|:---|
| R12.1 | DATA | **Cross-tenant data leakage** in analytics Materialized Views. | 2 | 3 | **6** | Row-level security (RLS) and multi-tenant isolation suites. | backend-team | Sprint 13 |
| R12.2 | SEC | **Admin API unauthorized access**: Privilege escalation to platform admin. | 2 | 3 | **6** | Negative RBAC testing and JWT claim validation. | security-team | Sprint 13 |
| R12.3 | PERF | **Analytics Query Latency**: MV refreshes taking >30s for large tenants. | 3 | 2 | **6** | Performance benchmarks with 1M+ record datasets. | performance-team | Sprint 14 |
| R12.4 | SEC | **Enterprise API Exposure**: Leakage of API keys or lack of rate limiting. | 2 | 3 | **6** | Rate-limiting middleware validation and key lifecycle tests. | backend-team | Sprint 14 |

### Medium-Priority Risks (Score 3-4)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|:---|:---|:---|:---:|:---:|:---:|:---|:---|
| R12.5 | BUS | **Tier Gate Bypass**: Free tier users accessing Enterprise features. | 2 | 2 | **4** | Subscription gate middleware integration tests. | product-team |
| R12.6 | OPS | **Report Job Failure**: Async report generation failing silently in queue. | 2 | 2 | **4** | Dead Letter Queue monitoring and job retry validation. | ops-team |

### Low-Priority Risks (Score 1-2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|:---|:---|:---|:---:|:---:|:---:|:---|
| R12.7 | TECH | Minor UI glitches in dashboard charts on specific browsers. | 1 | 2 | 2 | Monitor |
| R12.8 | BUS | Analytics data stale by < 5 minutes (MV refresh lag). | 1 | 1 | 1 | Monitor |

---

## Entry Criteria

- [ ] Requirements for Epic 12 finalized and signed off.
- [ ] Staging environment with Citus/PostgreSQL Materialized View support ready.
- [ ] Test data factories for multi-tenant data seeding available.
- [ ] Playwright Utils (`apiRequest`, `recurse`) configured in the monorepo.

## Exit Criteria

- [ ] 100% of P0 Tenant Isolation and Security tests passing.
- [ ] 95% of P1 Functional and Performance tests passing.
- [ ] Analytics dashboard query response time < 3s for 95th percentile.
- [ ] No open Critical or High severity defects.

---

## Test Coverage Plan

### P0 (Critical) - Run on every commit

**Criteria**: Blocks core journey + High risk (≥6) + No workaround

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| R12.1: Tenant Isolation | API | R12.1 | 5 | QA | Validate SQL filters and RLS. |
| R12.2: Admin RBAC | API | R12.2 | 8 | QA | Negative testing for all Admin endpoints. |
| R12.4: API Key Gate | API | R12.4 | 4 | QA | Verify revocation and rate-limiting. |

**Total P0**: 17 tests, ~25 hours

### P1 (High) - Run on PR to main

**Criteria**: Important features + Medium risk (3-4) + Common workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| S12.01: MV Refresh | API | R12.1 | 6 | DEV | Verify data consistency after refresh. |
| R12.3: Analytics Perf | Load | R12.3 | 3 | QA | Benchmarks with 1M+ rows. |
| S12.10: Report Gen | E2E | R12.6 | 4 | QA | Use `recurse` to poll for completion. |
| S12.11: Admin CRUD | API | - | 12 | DEV | Standard CRUD for users/tenants. |

**Total P1**: 25 tests, ~45 hours

### P2 (Medium) - Run nightly/weekly

**Criteria**: Secondary features + Low risk (1-2) + Edge cases

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| S12.12: UI Branding | Component | - | 10 | DEV | Verify logo/color injection in UI. |
| S12.15: Public API Docs | API | - | 5 | QA | Verify Swagger matches implementation. |
| S12.18: Onboarding | E2E | - | 2 | QA | Verify first-time user guided tour. |

**Total P2**: 17 tests, ~20 hours

---

## Execution Order

### Smoke Tests (<5 min)

- [ ] Admin Login (30s)
- [ ] Analytics Dashboard Load (45s)
- [ ] Admin User Search (30s)

### P0 Tests (<10 min)

- [ ] Cross-tenant data isolation check (API)
- [ ] Admin endpoint privilege check (API)
- [ ] Enterprise API Key validation (API)

### P1 Tests (<30 min)

- [ ] Materialized View refresh logic (API)
- [ ] Report generation async flow (E2E)
- [ ] Admin user management CRUD (API)

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test | Total Hours | Notes |
|:---|:---|:---:|:---|:---|
| P0 | 17 | 1.5 | ~25 | Focus on security/isolation. |
| P1 | 25 | 1.8 | ~45 | Async logic and performance setup. |
| P2 | 17 | 1.2 | ~20 | UI and documentation. |
| **Total** | **59** | **-** | **~90** | **~11-12 days** |

### Prerequisites

**Test Data:**
- `TenantDataFactory`: Generates 10k-1M records per tenant.
- `AdminUserFactory`: Generates users with specific RBAC roles.

**Tooling:**
- `Playwright apiRequest`: For all P0/P1 backend validation.
- `Playwright recurse`: For async report generation polling.

---

## Quality Gate Criteria

### Pass/Fail Thresholds
- **P0 pass rate**: 100%
- **P1 pass rate**: ≥95%
- **Security scenarios**: 100% pass for isolation and RBAC.

### Non-Negotiable Requirements
- [ ] All Tenant Isolation tests pass (Zero tolerance for leakage).
- [ ] Admin RBAC prevents non-admin access (Zero tolerance for escalation).
- [ ] Performance benchmarks meet <3s target for dashboard load.

---

## Mitigation Plans

### R12.1: Cross-tenant data leakage (Score: 6)
**Mitigation Strategy:** Implement automated multi-tenant security suite that injects 'wrong' tenant context into all analytics queries and verifies empty/error response.
**Owner:** Security/Backend Lead
**Status:** In Progress
**Verification:** Run `test-design-epic-12.md` P0 suite.

### R12.2: Admin API unauthorized access (Score: 6)
**Mitigation Strategy:** Implement 'Negative Security Suite' that iterates through all Admin endpoints using non-privileged tokens.
**Owner:** Security Lead
**Status:** Planned
**Verification:** Verify 403 Forbidden for all unauthorized attempts.

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|:---|:---|:---|
| **Auth Service** | Affected by Admin RBAC changes. | Existing login and token refresh tests must pass. |
| **Analytics Service** | Core component of Epic 12. | Materialized View refresh logic and query speed. |
| **S3 Storage** | Used for report persistence. | Verify S3 connectivity and file upload/download. |

---

## Appendix

### Knowledge Base References
- `risk-governance.md`
- `probability-impact.md`
- `test-levels-framework.md`
- `test-priorities-matrix.md`

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 4.0 (BMad v6)
