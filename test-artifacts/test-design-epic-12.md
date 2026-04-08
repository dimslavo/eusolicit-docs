---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-06'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 12
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'knowledge/risk-governance.md'
  - 'knowledge/probability-impact.md'
  - 'knowledge/test-levels-framework.md'
  - 'knowledge/test-priorities-matrix.md'
---

# Test Design: Epic 12 — Analytics, Reporting & Admin Platform

**Date:** 2026-04-06
**Author:** TEA Master Test Architect
**Status:** Draft

---

## Executive Summary

**Scope:** Epic-level test design for E12 — Analytics dashboards (market, ROI, team, competitor, pipeline, usage), report generation (PDF/DOCX), scheduled/on-demand delivery, VPN-restricted Admin API (tenants, crawlers, white-label, audit logs, platform analytics), Enterprise API (API key auth, rate limiting, OpenAPI docs), performance optimization, load testing, security audit, and user onboarding wizard.

**Sprint:** 13–14 | **Points:** 55 | **Dependencies:** E05, E06, E07, E08

**Risk Summary:**

- Total risks identified: 11
- High-priority risks (score >= 6): 3
- Critical categories: SEC (cross-tenant analytics leakage, OWASP coverage), BUS (tier gate bypass on Professional+ dashboards)

**Coverage Summary:**

- P0 scenarios: 10 (~15–25 hours)
- P1 scenarios: 18 (~20–35 hours)
- P2/P3 scenarios: 24 (~10–20 hours)
- **Total effort**: ~45–80 hours (~1.5–2.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **KraftData AI agent output quality** | Owned by KraftData eval-runs; E12 consumes predictions, does not generate AI content | Integration contract tested via AI Gateway mock responses |
| **Stripe billing lifecycle** | Covered in E06 test design; E12 usage dashboard reads existing subscription data | Usage dashboard tests assume valid subscription state via test data seeding |
| **Proposal generation/export** | Covered in E07; E12 reuses shared document generation code from E07 | S12.09 tests report-specific templates only, not proposal export |
| **Data pipeline crawl logic** | Covered in E04; S12.12 tests admin CRUD for crawlers, not crawl parsing | Admin endpoints tested for management operations, not source integration |
| **Calendar sync (Google/Microsoft)** | Covered in E08; no calendar features in E12 | N/A |

---

## Risk Assessment

### High-Priority Risks (Score >= 6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E12-R-001** | **SEC** | Cross-tenant data leakage in analytics — materialized views aggregate data across companies; missing `company_id` scoping in view definitions or API queries exposes tenant A's metrics to tenant B | 2 | 3 | **6** | Automated cross-tenant isolation tests on every analytics endpoint; verify materialized view SQL includes `company_id` WHERE clauses | Backend Lead | Sprint 13 |
| **E12-R-002** | **BUS** | Tier gate bypass on Professional+ dashboards — misconfigured or missing tier middleware on competitor/pipeline endpoints allows lower-tier users to access gated features without paying | 2 | 3 | **6** | Parametrized tier boundary tests: Free, Starter, Professional, Enterprise against all gated endpoints; verify 403 + upgrade prompt for ineligible tiers | Backend Lead | Sprint 13 |
| **E12-R-003** | **SEC** | OWASP audit false confidence — incomplete checklist coverage creates false security assurance before launch; missed items surface as post-launch vulnerabilities | 2 | 3 | **6** | Independent checklist review by second engineer; automated OWASP ZAP scan against staging; document risk-accepted items with rationale | Security Lead | Sprint 14 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E12-R-004 | PERF | Report generation resource exhaustion — PDF/DOCX with embedded chart images may spike Celery worker memory under concurrent generation | 2 | 2 | 4 | Worker memory limits + concurrency cap per worker; test with 10 concurrent report generations | Backend Lead |
| E12-R-005 | SEC | Enterprise API rate limit bypass — token bucket implementation flaws allow circumvention via key rotation or timing attacks | 2 | 2 | 4 | Burst testing: exceed limit rapidly, verify 429 + headers; test key rotation doesn't reset rate window | Backend Lead |
| E12-R-006 | PERF | Materialized view refresh blocking reads — if unique index is missing, `REFRESH CONCURRENTLY` fails and falls back to blocking refresh, causing dashboard timeouts | 2 | 2 | 4 | Verify unique indexes exist on all materialized views; test read availability during refresh | DBA/Backend |
| E12-R-007 | OPS | Scheduled report email delivery failure — SendGrid outage or template misconfiguration leads to silent report non-delivery | 2 | 2 | 4 | Delivery confirmation logging; retry with exponential backoff; test with SendGrid sandbox mode | Backend Lead |
| E12-R-008 | SEC | Admin API VPN/IP bypass — misconfigured IP allowlist exposes admin endpoints to public internet | 1 | 3 | 3 | Test from non-allowlisted IP verifies 403; integration test verifying middleware ordering | DevOps |
| E12-R-009 | SEC | API key hash storage weakness — keys stored with weak hash (MD5/SHA1) means compromised DB exposes usable keys | 1 | 3 | 3 | Code review verifying bcrypt/scrypt; test that raw key is not retrievable from DB | Backend Lead |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E12-R-010 | TECH | Onboarding wizard state persistence — if `onboarding_completed` flag fails to persist, wizard loops on every login | 2 | 1 | 2 | Monitor |
| E12-R-011 | OPS | White-label subdomain DNS validation — DNS readiness check may produce false negatives for propagating records | 1 | 1 | 1 | Monitor |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

**System-Level Risk Inheritance:** E12-R-001 extends system-level R-002 (multi-tenant data isolation) into analytics materialized views. E12-R-002 extends system-level R-012 (tier gating accuracy) into Professional+ dashboard gates.

---

## Entry Criteria

- [ ] E05, E06, E07, E08 features deployed and stable in staging
- [ ] Test data seeding API operational (TB-01 from system-level design)
- [ ] AI Gateway mock mode available for pipeline forecasting agent responses (TB-02)
- [ ] Stripe test mode configured for usage/subscription data (TB-03)
- [ ] PostgreSQL materialized views deployed via migration (S12.01)
- [ ] Celery Beat configured and running in staging
- [ ] SendGrid sandbox mode configured for email delivery tests
- [ ] VPN/IP allowlist configured for admin API staging environment

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (>= 95% or failures triaged and accepted)
- [ ] No open P0/P1 severity bugs in E12 scope
- [ ] Cross-tenant analytics isolation validated across all 6 dashboard domains
- [ ] Tier gate enforcement validated on all Professional+ endpoints
- [ ] Load test results documented with p95 < 500ms for analytics reads
- [ ] OWASP checklist completed with all items addressed or risk-accepted
- [ ] All production secrets rotated and documented

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on if time-constrained), NOT execution timing. See "Execution Strategy" for when tests run.

### P0 (Critical)

**Criteria:** Blocks core functionality + High risk (score >= 6) + No workaround + Affects majority of users

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| Cross-tenant isolation — analytics endpoints (market, ROI, team, competitor, pipeline, usage) | API | E12-R-001 | 2 | QA | Seed Company A + B analytics data; authenticate as A; query all 6 domains; assert zero B records |
| Tier gate — competitor intelligence endpoints return 403 for Free/Starter | API | E12-R-002 | 1 | QA | Parametrized: Free -> 403, Starter -> 403, Professional -> 200, Enterprise -> 200 |
| Tier gate — pipeline forecasting endpoints return 403 for Free/Starter | API | E12-R-002 | 1 | QA | Same parametrized pattern as competitor gate |
| Admin API VPN/IP restriction — non-allowlisted IP rejected | API | E12-R-008 | 1 | QA | Request from non-VPN IP -> 403 on all admin endpoints |
| Admin API authentication — admin role required | API | E12-R-008 | 1 | QA | Regular user JWT -> 403 on admin endpoints |
| Enterprise API — X-API-Key authentication validates correctly | API | E12-R-005 | 1 | QA | Valid key -> 200; invalid/missing/revoked key -> 401 |
| Enterprise API — rate limiting enforced, 429 returned | API | E12-R-005 | 1 | QA | Exceed limit -> 429 with X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset headers |
| OWASP security checklist — automated scan validation | API | E12-R-003 | 1 | QA | ZAP scan against staging; verify no high/critical findings |
| Load test — analytics dashboard p95 < 500ms under concurrency | Perf | E12-R-006 | 1 | QA | k6 script targeting market + ROI + team endpoints under 50 concurrent users |

**Total P0:** 10 tests, ~15–25 hours

---

### P1 (High)

**Criteria:** Important features + Medium risk (3–5) + Common workflows + Workaround difficult

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| Materialized views — all 5 domains created with correct schema | API | E12-R-006 | 1 | QA | Query each view; verify column presence and data types |
| Celery Beat — daily refresh for market/ROI/team/competitor, hourly for usage | API | E12-R-006 | 1 | QA | Trigger manual refresh; verify view data updated |
| `REFRESH CONCURRENTLY` — reads not blocked during refresh | API | E12-R-006 | 1 | QA | Start refresh; concurrent read returns data without lock wait timeout |
| Market intelligence API — volume, values, authorities, trends with filters | API | — | 1 | QA | Date range, sector, country filter combinations; empty result edge case |
| ROI tracker API — summary, per-bid breakdown, trends | API | — | 1 | QA | Verify calculation correctness: total invested vs total won, ROI % |
| Team performance API — leaderboard and individual user metrics | API | — | 1 | QA | Admin sees all users; regular user sees only self |
| Competitor intelligence API — profiles, patterns, benchmarks (Professional+) | API | E12-R-002 | 1 | QA | Verify data correctness + tier gate enforced |
| Pipeline forecasting API — predictions with confidence scores (Professional+) | API | E12-R-002 | 1 | QA | Via AI Gateway mock; verify confidence score ranges and prediction structure |
| Usage dashboard API — consumption vs limits per usage type | API | — | 1 | QA | Verify consumed/limit/remaining values; billing period start/end dates |
| Report generation — PDF output with headers, tables, chart images | API | E12-R-004 | 1 | QA | Generate pipeline summary PDF; verify non-empty file, correct MIME type |
| Report generation — DOCX output with headers, tables, chart images | API | E12-R-004 | 1 | QA | Generate same report as DOCX; verify non-empty file, correct MIME type |
| Scheduled report delivery — Celery Beat trigger + SendGrid email | API | E12-R-007 | 1 | QA | Configure weekly schedule; trigger; verify email sent via SendGrid sandbox |
| On-demand report — async generation with download link | API | — | 1 | QA | Request generation; poll status; verify signed S3 URL returned with 24h expiry |
| Admin tenant management — list, detail, tier override with audit log | API | — | 1 | QA | CRUD operations; verify audit log entry created on tier override |
| Admin audit log — search, filter, CSV export | API | — | 1 | QA | Filter by user/action/entity/date; verify streaming CSV response |
| Enterprise API — OpenAPI spec auto-generated and accessible | API | — | 1 | QA | GET /v1/docs returns Swagger UI; GET /v1/redoc returns Redoc |
| API key CRUD — create, list, revoke | API | E12-R-009 | 1 | QA | Create returns full key once; list returns masked (last 4 chars); revoked key returns 401 |
| Load test — search, opportunity detail, proposal generation under target concurrency | Perf | — | 1 | QA | k6 script; document p50/p95/p99 latencies and error rates |

**Total P1:** 18 tests, ~20–35 hours

---

### P2 (Medium)

**Criteria:** Secondary features + Low risk (1–2) + Edge cases + Regression prevention

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| Market intelligence frontend — bar chart, line chart, table, filters | E2E | — | 1 | QA | Charts render; filters update visualizations; responsive layout stacks on mobile |
| ROI tracker frontend — summary cards, per-bid table, trend chart | E2E | — | 1 | QA | Metrics display; table sorting; date range filter |
| Team performance frontend — user cards, leaderboard, activity chart | E2E | — | 1 | QA | Leaderboard sorting; admin vs regular user view difference |
| Competitor intelligence frontend — profile cards, comparison table, pattern chart | E2E | — | 1 | QA | Select 2–4 competitors side by side |
| Pipeline forecasting frontend — timeline view, confidence indicators, filters | E2E | — | 1 | QA | Color-coded confidence (green/amber/red); sector/value/threshold filters; empty state |
| Usage dashboard frontend — circular meters, 80% warning, upgrade CTA | E2E | — | 1 | QA | Warning indicator at 80%; upgrade CTA at 100% |
| Report schedule configuration — admin form for weekly/monthly | E2E | — | 1 | QA | Configure schedule with type, format, recipients; verify persistence |
| Reports list page — past reports with download links | E2E | — | 1 | QA | Download links functional; timestamps displayed |
| Admin tenant management frontend — search, detail drawer, tier override | E2E | — | 1 | QA | Search by name; filter by tier; override with reason; change reflected |
| Admin crawler management frontend — run history, schedule config, manual trigger | E2E | — | 1 | QA | Trigger manual crawl; status badges (success/failed/running) displayed |
| Admin white-label frontend — logo upload, color pickers, live preview | E2E | — | 1 | QA | Change colors -> preview updates in real time; subdomain validated |
| Admin audit log frontend — filter bar, results table, CSV download | E2E | — | 1 | QA | Apply filters; export CSV; verify download |
| Admin platform analytics frontend — funnel, pie chart, revenue chart | E2E | — | 1 | QA | Charts render with data; metric cards display |
| Enterprise API documentation page — embedded Swagger UI or Redoc | E2E | — | 1 | QA | Interactive API exploration functional |
| API key management frontend — list, create, revoke | E2E | — | 1 | QA | Full key shown once on creation; copy-to-clipboard; revoke with confirmation |
| Admin crawler schedule — read/update Celery Beat schedule per type | API | — | 1 | QA | PUT schedule -> GET returns updated values |
| Admin crawler trigger — manual crawl returns run_id + status polling | API | — | 1 | QA | POST trigger -> GET run status shows progress |
| Admin white-label — subdomain uniqueness + DNS readiness check | API | E12-R-011 | 1 | QA | Duplicate subdomain -> 409; DNS check returns readiness status |
| Admin platform analytics — funnel, tiers, usage, revenue metrics | API | — | 1 | QA | Verify metric calculations and response structure |
| Rate limit headers — X-RateLimit-Limit, Remaining, Reset present | API | E12-R-005 | 1 | QA | Verify headers on every Enterprise API response |

**Total P2:** 20 tests, ~8–15 hours

---

### P3 (Low)

**Criteria:** Nice-to-have + Exploratory + Polish items

| Requirement | Test Level | Test Count | Owner | Notes |
|-------------|------------|------------|-------|-------|
| Onboarding wizard — first login detection, step-by-step flow | E2E | 1 | QA | Wizard appears on first login; each step spotlights relevant UI area with tooltip |
| Onboarding wizard — skip/dismiss, completion persists | E2E | 1 | QA | Skip -> wizard doesn't reappear; complete -> onboarding_completed flag set |
| Empty states — reviewed across all new E12 pages | E2E | 1 | QA | Analytics pages with no data show appropriate empty state messages |
| Error pages (404/500/403) — branded templates | E2E | 1 | QA | Navigate to invalid route -> branded 404 page |

**Total P3:** 4 tests, ~2–5 hours

---

## Execution Strategy

**Philosophy:** Run everything in PRs unless significant infrastructure overhead. Playwright with parallelization handles the functional test suite in ~10–15 min.

**Organized by tool type:**

### Every PR: Playwright + pytest Tests (~10–15 min)

All P0, P1, P2, P3 functional tests (API + E2E) run via Playwright and pytest with parallelization across 4 shards. Total: ~50 functional tests.

**Why run in PRs:** Fast feedback; no expensive infrastructure needed.

### Nightly: k6 Performance Tests (~30–60 min)

- Analytics dashboard latency under load (P0 load test)
- Core user flow latency benchmarks (P1 load test)
- Report generation concurrency stress test

**Why defer to nightly:** Requires dedicated staging environment, k6 runner, 30+ min runtime.

### Weekly: Security Scan (~1–2 hours)

- OWASP ZAP automated scan against staging
- Network policy verification for admin API isolation

**Why defer to weekly:** Long-running; requires security tooling infrastructure.

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Effort Range | Notes |
|----------|-------|-------------|-------|
| P0 | 10 | ~15–25 hours | Cross-tenant isolation, tier gates, security scans, load tests — complex setup |
| P1 | 18 | ~20–35 hours | API endpoint validation, report generation, admin CRUD — standard complexity |
| P2 | 20 | ~8–15 hours | Frontend E2E, admin pages, secondary API tests — straightforward |
| P3 | 4 | ~2–5 hours | Onboarding wizard, empty states, error pages — simple E2E |
| **Total** | **52** | **~45–80 hours** | **~1.5–2.5 weeks (1 QA)** |

### Prerequisites

**Test Data:**

- Company factory (all tiers: free, starter, professional, enterprise)
- User factory (all roles: admin, bid_manager, contributor, reviewer, read_only)
- Analytics seed data factory (materialized view source records: bid_preparation_logs, bid_outcomes, usage_records)
- Report schedule factory (weekly/monthly configurations)
- API key factory (enterprise keys with rate limit tiers)
- Competitor data factory (mock competitor profiles and bidding patterns)

**Tooling:**

- Playwright for E2E + API tests
- pytest + pytest-asyncio for backend integration tests
- k6 for performance/load tests
- OWASP ZAP for automated security scanning
- SendGrid sandbox for email delivery testing

**Environment:**

- Staging environment with all E05–E08 features deployed
- VPN/IP allowlist configured for admin API testing
- Celery Beat running with test schedules
- MinIO (S3-compatible) for report storage
- Redis for rate limiting and caching

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate**: 100% (no exceptions)
- **P1 pass rate**: >= 95% (waivers required for failures)
- **P2/P3 pass rate**: >= 90% (informational)
- **High-risk mitigations**: 100% complete or approved waivers

### Coverage Targets

- **Critical paths (analytics isolation, tier gates, admin auth)**: >= 80%
- **Security scenarios (SEC category)**: 100%
- **Business logic (calculations, aggregations)**: >= 70%
- **Edge cases (empty data, boundary values)**: >= 50%

### Non-Negotiable Requirements

- [ ] All P0 tests pass
- [ ] No high-risk (>= 6) items unmitigated
- [ ] Security tests (SEC category) pass 100%
- [ ] Load test targets met (p95 < 500ms for analytics reads)
- [ ] OWASP checklist completed

---

## Mitigation Plans

### E12-R-001: Cross-Tenant Data Leakage in Analytics (Score: 6)

**Mitigation Strategy:**
1. Code review of all materialized view SQL definitions — verify `company_id` in every view and refresh query
2. Automated cross-tenant API tests: seed Company A and Company B analytics data, authenticate as Company A, query all 6 analytics domains, assert zero Company B records
3. DB-level verification: query materialized views directly with wrong `company_id` to confirm row-level scoping

**Owner:** Backend Lead
**Timeline:** Sprint 13 (before dashboard API implementation)
**Status:** Planned
**Verification:** P0 cross-tenant isolation tests passing in CI

### E12-R-002: Tier Gate Bypass on Professional+ Dashboards (Score: 6)

**Mitigation Strategy:**
1. Tier gate middleware applied at router level (not per-endpoint) for competitor and pipeline route groups
2. Parametrized API tests: every tier (Free, Starter, Professional, Enterprise) x every gated endpoint
3. Verify 403 response includes upgrade prompt message and correct tier requirement

**Owner:** Backend Lead
**Timeline:** Sprint 13 (concurrent with S12.06, S12.07)
**Status:** Planned
**Verification:** P0 tier gate tests passing with all 4 tier variations

### E12-R-003: OWASP Audit False Confidence (Score: 6)

**Mitigation Strategy:**
1. OWASP ZAP automated scan against staging covering all E12 endpoints
2. Manual review of checklist items that cannot be automated (logic flaws, business logic bypasses)
3. Second-engineer independent review of checklist completion
4. All risk-accepted items documented with rationale, owner, and re-evaluation date

**Owner:** Security Lead
**Timeline:** Sprint 14 (S12.17)
**Status:** Planned
**Verification:** ZAP scan report with zero high/critical findings; signed-off checklist document

---

## Assumptions and Dependencies

### Assumptions

1. Materialized views from S12.01 are deployed and populated with representative test data before dashboard API testing begins
2. Shared report generation code from E07 (proposal export) is stable and does not require retesting for E12
3. KraftData Pipeline Forecasting Agent returns deterministic mock responses for dashboard testing via AI Gateway mock mode
4. SendGrid sandbox mode supports attachment delivery testing (email sent to test inbox, not real users)
5. VPN/IP allowlist in staging matches production configuration pattern (different IPs, same middleware)

### Dependencies

1. **E05–E08 stability** — Analytics dashboards depend on data created by earlier epics (opportunities, proposals, bids, tasks). Required by: Sprint 13 start
2. **Test data seeding API (TB-01)** — Required for creating multi-tenant analytics test scenarios. Required by: Sprint 13 start
3. **AI Gateway mock mode (TB-02)** — Required for pipeline forecasting dashboard tests. Required by: Sprint 13 (S12.07)
4. **SendGrid sandbox configuration** — Required for scheduled report delivery tests. Required by: Sprint 13 (S12.10)
5. **k6 Cloud or self-hosted runner** — Required for load testing. Required by: Sprint 14 (S12.17)
6. **OWASP ZAP tooling** — Required for automated security scanning. Required by: Sprint 14 (S12.17)

### Risks to Plan

- **Risk**: Materialized views contain stale or incomplete data in staging, causing false test failures
  - **Impact**: Dashboard API tests return unexpected empty results or wrong aggregations
  - **Contingency**: Manual refresh command (S12.01 AC) + dedicated analytics seed data factory

- **Risk**: Load test infrastructure not available by Sprint 14
  - **Impact**: S12.17 load testing cannot complete; performance targets unverified at launch
  - **Contingency**: Run k6 locally against staging with reduced concurrency; document as limitation

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|-------------------|--------|-----------------|
| **Client API** | Analytics endpoints added (market, ROI, team, competitor, pipeline, usage); report generation; onboarding | Existing auth, opportunity, proposal tests must still pass |
| **Admin API** | New tenant management, crawler management, white-label, audit log, platform analytics endpoints | Existing compliance framework admin tests must still pass |
| **Notification Service** | Scheduled report delivery via Celery Beat + SendGrid | Existing alert/digest delivery tests must still pass |
| **Data Pipeline** | Crawler management endpoints expose run history and schedule control | Existing crawl pipeline tests must still pass |
| **AI Gateway** | Pipeline forecasting dashboard consumes prediction agent responses | Existing AI summary and proposal generation tests must still pass |
| **Enterprise API** | New API surface at api.eusolicit.com/v1/* with API key auth and rate limiting | New surface; no regression from prior epics |

---

## Follow-on Workflows (Manual)

- Run `*atdd` to generate failing P0 tests (separate workflow; not auto-run).
- Run `*automate` for broader coverage once implementation exists.

---

## Approval

**Test Design Approved By:**

- [ ] Product Manager: ________ Date: ________
- [ ] Tech Lead: ________ Date: ________
- [ ] QA Lead: ________ Date: ________

**Comments:**

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification framework (P x I scoring, gate decisions)
- `probability-impact.md` — Risk scoring methodology (1–3 scales, threshold rules)
- `test-levels-framework.md` — Test level selection (E2E vs API vs Unit)
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

### Related Documents

- PRD: `eusolicit-docs/EU_Solicit_PRD_v1.md`
- Epic: `eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md`
- Architecture: `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md`
- System-Level Test Design (Architecture): `eusolicit-docs/test-artifacts/test-design-architecture.md`
- System-Level Test Design (QA): `eusolicit-docs/test-artifacts/test-design-qa.md`

### System-Level Risk Cross-References

| E12 Risk | System-Level Risk | Relationship |
|----------|-------------------|-------------|
| E12-R-001 (Cross-tenant analytics leakage) | R-002 (Multi-tenant data isolation) | E12 extends R-002 into materialized view aggregations |
| E12-R-002 (Tier gate bypass) | R-012 (Tier gating accuracy) | E12 extends R-012 into Professional+ gated dashboards |
| E12-R-003 (OWASP audit coverage) | — | New; pre-launch security hardening |
| E12-R-005 (Rate limit bypass) | R-008 (PostgreSQL shared instance) | Rate limiting protects shared DB from overload |

---

**Generated by:** BMad TEA Agent — Test Architect Module
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
