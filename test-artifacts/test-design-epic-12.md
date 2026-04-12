---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-12'
---

# Test Design: Epic 12 - Analytics, Reporting & Admin Platform

**Date:** 2026-04-12
**Author:** Deb
**Status:** Approved

---

## Executive Summary

**Scope:** Full test design for Epic 12 — Analytics dashboards, report generation, admin platform, and enterprise API.

**Risk Summary:**

- Total risks identified: 12
- High-priority risks (≥6): 3
- Critical categories: SEC, BUS, PERF

**Coverage Summary:**

- P0 scenarios: 9 (21 atomic tests) (25 hours)
- P1 scenarios: 19 (35 hours)
- P2/P3 scenarios: 25 (20 hours)
- **Total effort**: 80 hours (~10 days)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **AI Content Quality** | Owned by AI service eval-runs; E12 consumes forecasting data | Integration tested via AI Gateway mocks |
| **Billing Lifecycle** | Covered in E06; E12 usage dashboard reads state only | Usage tests assume valid subscription via seeding |
| **Proposal Export Engine** | Covered in E07; S12.09 reuses shared export code | Test report-specific templates only |
| **Crawler Parsing Logic** | Covered in E04/E05; S12.12 tests management CRUD only | Admin endpoints tested for CRUD operations |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E12-R-001** | **SEC** | **Cross-tenant data leakage** in analytics — missing `company_id` scoping in materialized views or API queries exposes Tenant A data to Tenant B. | 2 | 3 | **6** | Automated isolation tests on every analytics endpoint; ORM unit tests confirm `company_id` column. | Backend Lead | Sprint 13 |
| **E12-R-002** | **BUS** | **Tier gate bypass / Stub risk** — `require_professional_plus_tier` is currently a **stub** (NotImplementedError). If S12.06/S12.07 endpoints land without real implementation, Professional+ features will return 500 errors. | 2 | 3 | **6** | Replace stub with real logic before S12.06 implementation; verify 403 response on gated endpoints. | Backend Lead | Sprint 13 |
| **E12-R-003** | **SEC** | **OWASP audit gaps** — missed items in pre-launch checklist create vulnerabilities in the admin/enterprise API surface. | 2 | 3 | **6** | Automated ZAP scan + independent checklist review by security lead. | Security Lead | Sprint 14 |

### Medium-Priority Risks (Score 3-4)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E12-R-004 | PERF | Report generation resource exhaustion (OOM) | 2 | 2 | 4 | Concurrency caps per worker; load test with 10+ concurrent reports | Backend Lead |
| E12-R-005 | SEC | Enterprise API rate limit bypass via key rotation | 2 | 2 | 4 | Test rotation doesn't reset rate window | Backend Lead |
| E12-R-006 | PERF | Materialized view refresh blocking dashboard reads | 2 | 2 | 4 | Confirm `REFRESH CONCURRENTLY` + unique indexes | DBA |
| E12-R-007 | OPS | Scheduled report email delivery failure | 2 | 2 | 4 | Exponential backoff + delivery logging | Backend Lead |

### Low-Priority Risks (Score 1-2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E12-R-008 | BUS | Minor dashboard chart rendering glitches | 1 | 2 | 2 | Visual regression tests |
| E12-R-009 | OPS | Inaccurate usage dashboard due to Celery lag | 1 | 2 | 2 | Monitor Celery queue depth |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

## Entry Criteria

- [ ] Core services (E05–E08) stable in staging.
- [ ] Test data seeding API (TB-01) operational.
- [ ] AI Gateway mock mode (TB-02) available for forecasting tests.
- [ ] Materialized views (S12.01) deployed and reachable.
- [ ] **BLOCKER**: `require_professional_plus_tier` stub replaced with real implementation.

## Exit Criteria

- [ ] 100% P0 tests passing (21 atomic tests).
- [ ] ≥ 95% P1 tests passing (failures triaged).
- [ ] Cross-tenant isolation validated across all 6 dashboards.
- [ ] Load test: p95 < 500ms for analytics reads under 50 concurrent users.
- [ ] OWASP checklist completed and signed off.

---

## Test Coverage Plan

### P0 (Critical) - Run on every commit

**Criteria**: Blocks core journey + High risk (≥6) + No workaround

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| Cross-tenant isolation | API | E12-R-001 | 2 | QA | Verify Company A cannot see B's analytics. |
| Tier gate enforcement | API | E12-R-002 | 7 | QA | Verify 403 on gated endpoints. |
| Admin Access Control | API | E12-R-003 | 4 | QA | Verify VPN/IP allowlist and Admin role JWT. |
| Enterprise API Auth | API | E12-R-005 | 6 | QA | Verify X-API-Key and Rate Limiting (429). |
| OWASP Security Scan | Infra | E12-R-003 | 1 | SEC | ZAP scan against staging endpoints. |
| Load Test (p95) | Perf | E12-R-006 | 1 | PERF | k6 load test on analytics reads. |

**Total P0**: 21 tests, 25 hours

### P1 (High) - Run on PR to main

**Criteria**: Important features + Medium risk (3-4) + Common workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| Materialized View Refresh | API/DB | E12-R-006 | 3 | QA | Verify Celery Beat schedule and concurrent refresh. |
| Analytics Endpoints | API | E12-R-001 | 6 | QA | Market, ROI, Team, Competitor, Pipeline, Usage. |
| Report Generation | API | E12-R-004 | 4 | QA | Format, MIME type, S3 signed URL expiry. |
| Report Delivery | API | E12-R-007 | 2 | QA | SendGrid integration + attachments. |
| Admin CRUD | API | E12-R-003 | 4 | QA | Search, pagination, and data integrity. |

**Total P1**: 19 tests, 35 hours

### P2 (Medium) - Run nightly/weekly

**Criteria**: Secondary features + Low risk (1-2) + Edge cases

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| Dashboard Visuals | E2E | E12-R-008 | 6 | QA | Chart rendering, tooltips, loading skeletons. |
| Frontend Filtering | E2E | - | 4 | QA | State consistency on Apply click. |
| Responsive Layout | E2E | - | 2 | QA | Stacked charts on mobile viewports. |
| Admin UI Workflows | E2E | - | 5 | QA | Detail drawers, color pickers, override forms. |
| Enterprise API Docs | E2E | - | 2 | QA | Swagger UI / Redoc accessibility. |

**Total P2**: 19 tests, 15 hours

### P3 (Low) - Run on-demand

**Criteria**: Nice-to-have + Exploratory + Performance benchmarks

| Requirement | Test Level | Test Count | Owner | Notes |
|-------------|------------|------------|-------|-------|
| Onboarding Wizard | E2E | 2 | QA | Spotlight effects, state persistence. |
| Empty States & 404/500 | E2E | 2 | QA | Branded error pages and empty states. |

**Total P3**: 4 tests, 5 hours

---

## Execution Order

### Smoke Tests (<5 min)

- [ ] Health check analytics endpoints (30s)
- [ ] Admin login (VPN check) (45s)
- [ ] Enterprise API key validation (1min)

**Total**: 3 scenarios

### P0 Tests (<10 min)

- [ ] Cross-tenant isolation (API)
- [ ] Tier gate enforcement (API)
- [ ] Rate limiting (API)

**Total**: 21 atomic tests

### P1 Tests (<30 min)

- [ ] Materialized view refresh (DB)
- [ ] Report generation async (Celery)
- [ ] Admin tenant management (API)

**Total**: 19 scenarios

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test | Total Hours | Notes |
|----------|-------|------------|-------------|-------|
| P0 | 21 | 1.2 | 25 | Complex security/isolation |
| P1 | 19 | 1.8 | 35 | Async flows, generation logic |
| P2 | 19 | 0.8 | 15 | UI/Visual tests |
| P3 | 4 | 1.2 | 5 | Onboarding wizard complexity |
| **Total** | **63** | **-** | **80** | **~10 days** |

### Prerequisites

**Test Data:**
- Tenant isolation factory
- Analytics data seeder (Historical data)
- Competitor profile generator

**Tooling:**
- Playwright (UI/API)
- k6 (Performance)
- OWASP ZAP (Security)

**Environment:**
- Staging with VPN access
- S3 bucket for reports
- SendGrid test account

---

## Quality Gate Criteria

### Pass/Fail Thresholds
- **P0 pass rate**: 100% (no exceptions)
- **P1 pass rate**: ≥95%
- **P2/P3 pass rate**: ≥90%
- **High-risk mitigations**: 100% complete

### Coverage Targets
- **Critical paths**: 100%
- **Security scenarios**: 100%
- **Business logic**: ≥80%

---

## Mitigation Plans

### E12-R-001: Cross-tenant data leakage (Score: 6)
**Mitigation Strategy:** Automated isolation tests on every analytics endpoint; ORM unit tests confirm `company_id` column.
**Owner:** Backend Lead
**Timeline:** Sprint 13
**Status:** In Progress
**Verification:** Isolation test suite pass rate.

### E12-R-002: Tier gate bypass / Stub risk (Score: 6)
**Mitigation Strategy:** Replace stub with real logic before S12.06 implementation; verify 403 response on gated endpoints.
**Owner:** Backend Lead
**Timeline:** Sprint 13
**Status:** Planned
**Verification:** API tests for gated endpoints.

---

## Assumptions and Dependencies

### Assumptions
1. PostgreSQL Materialized Views support concurrent refresh as planned.
2. SendGrid test environment supports attachment verification via API.

### Dependencies
1. **Tier Logic Readiness**: Required by Sprint 13 for gating tests.
2. **VPN Access for CI**: Required for Admin API testing.

---

## Follow-on Workflows (Manual)
- Run `bmad-testarch-test-design-validate` to confirm output quality.

---

## Approval
- [ ] Product Manager: Deb Date: 2026-04-12
- [ ] Tech Lead: Backend Lead Date: 2026-04-12

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|-------------------|--------|------------------|
| **Proposal Service** | Report generation reuses export logic | Proposal PDF/DOCX export |
| **Notification Service** | Scheduled reports trigger emails | Email delivery queue |

---

## Appendix

### Knowledge Base References
- `risk-governance.md`
- `probability-impact.md`
- `test-levels-framework.md`
- `test-priorities-matrix.md`

### Related Documents
- Epic: `eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md`
- Architecture: `test-design-architecture.md`

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 4.0 (BMad v6)
