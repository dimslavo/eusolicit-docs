---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-12'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 12
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md'
  - 'knowledge/risk-governance.md'
  - 'knowledge/probability-impact.md'
  - 'knowledge/test-levels-framework.md'
  - 'knowledge/test-priorities-matrix.md'
---

# Test Design: Epic 12 — Analytics, Reporting & Admin Platform

**Date:** 2026-04-12
**Author:** TEA Master Test Architect
**Status:** Final — P0 ATDD Red Phase Complete; P1–P3 ATDD Pending
**Version:** v3 (2026-04-12) — final pass, checklist-validated; supersedes v2 (2026-04-12)
**Project:** EU Solicit
**Epic Reference:** `eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md`
**System-Level Reference:** `test-design-architecture.md` (v4, 2026-04-09)
**ATDD Reference:** `atdd-checklist-e12-p0.md` (2026-04-05) — 19 P0 atomic tests, RED phase

---

## Executive Summary

**Scope:** Epic-level test design for E12 — Analytics dashboards (market intelligence, ROI tracker, team performance, competitor intelligence, pipeline forecasting, usage), report generation engine (PDF via reportlab, DOCX via python-docx), scheduled and on-demand report delivery via Celery Beat + SendGrid, VPN-restricted Admin API (tenant management, crawler management, white-label configuration, audit log, platform analytics), Enterprise API surface (API key auth, Redis token-bucket rate limiting, OpenAPI auto-documentation), performance optimization, load testing (k6), OWASP security audit, and first-run onboarding wizard.

**Sprint:** 13–14 | **Points:** 55 | **Dependencies:** E05, E06, E07, E08 | **Milestone:** MVP Launch

**Risk Summary:**

- Total risks identified: 11
- High-priority risks (score ≥ 6): 3
- Critical categories: SEC (cross-tenant analytics leakage, OWASP coverage), BUS (tier gate bypass on Professional+ dashboards)

**Coverage Summary:**

- P0 scenarios: 9 (~19 atomic tests, ~15–25 hours)
- P1 scenarios: 18 (~20–35 hours)
- P2 scenarios: 20 (~8–15 hours)
- P3 scenarios: 4 (~2–5 hours)
- **Total atomic tests:** ~61 functional + 2 infrastructure (k6 load, OWASP ZAP)
- **Total effort:** ~45–80 hours (~1.5–2.5 weeks, 1 QA)

**ATDD Status:** P0 RED-phase tests generated (2026-04-05). 19 atomic API tests across 4 spec files — all using `test.skip()`. GREEN phase begins when S12.01–S12.15 endpoints are deployed to staging.

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **KraftData AI agent output quality** | Owned by KraftData eval-runs; E12 consumes pipeline forecasting predictions, does not generate AI content | Integration tested via AI Gateway mock responses (TB-02) |
| **Stripe billing lifecycle** | Covered in E06 test design; E12 usage dashboard reads existing subscription data | Usage dashboard tests assume valid subscription state via test data seeding (TB-01) |
| **Proposal generation and export** | Covered in E07; S12.09 reuses shared document generation code from E07 | S12.09 tests report-specific templates only; E07 regression suite must be green |
| **Data pipeline crawl logic** | Covered in E04/E05; S12.12 tests admin CRUD for crawlers, not crawl parsing | Admin endpoints tested for management operations only |
| **Calendar sync (Google/Microsoft)** | Covered in E08; no calendar features in E12 scope | N/A |
| **ESPD XML conformance** | Covered in E11 test design (E11-R-018); E12 has no ESPD scope | N/A |

---

## Risk Assessment

### High-Priority Risks (Score ≥ 6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E12-R-001** | **SEC** | Cross-tenant data leakage in analytics — materialized views aggregate data across companies; missing `company_id` scoping in view definitions or API queries exposes Company A metrics to Company B | 2 | 3 | **6** | Automated cross-tenant isolation tests on every analytics endpoint; verify materialized view SQL includes `company_id` WHERE clauses; DB-level verification | Backend Lead | Sprint 13 |
| **E12-R-002** | **BUS** | Tier gate bypass on Professional+ dashboards — misconfigured or missing tier middleware on competitor/pipeline endpoints allows lower-tier users to access gated features without paying | 2 | 3 | **6** | Parametrized tier boundary tests: Free, Starter, Professional, Enterprise against all gated endpoints; verify 403 + upgrade prompt for ineligible tiers; router-level middleware (not per-endpoint) | Backend Lead | Sprint 13 |
| **E12-R-003** | **SEC** | OWASP audit false confidence — incomplete checklist coverage creates false security assurance before launch; missed items surface as post-launch vulnerabilities | 2 | 3 | **6** | Independent checklist review by second engineer; automated OWASP ZAP scan against staging; document risk-accepted items with rationale and owner | Security Lead | Sprint 14 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E12-R-004 | PERF | Report generation resource exhaustion — PDF/DOCX with embedded chart images may spike Celery worker memory under concurrent generation | 2 | 2 | 4 | Worker memory limits + concurrency cap per worker; test with 10 concurrent report generations | Backend Lead |
| E12-R-005 | SEC | Enterprise API rate limit bypass — token bucket implementation flaws allow circumvention via key rotation or timing attacks | 2 | 2 | 4 | Burst testing: exceed limit rapidly, verify 429 + headers; test key rotation doesn't reset rate window | Backend Lead |
| E12-R-006 | PERF | Materialized view refresh blocking reads — if unique index is missing, `REFRESH CONCURRENTLY` fails and falls back to blocking refresh, causing dashboard timeouts during refresh cycles | 2 | 2 | 4 | Verify unique indexes exist on all materialized views; test concurrent read availability during refresh | DBA/Backend |
| E12-R-007 | OPS | Scheduled report email delivery failure — SendGrid outage or template misconfiguration leads to silent report non-delivery with no user notification | 2 | 2 | 4 | Delivery confirmation logging; retry with exponential backoff; test with SendGrid sandbox mode | Backend Lead |
| E12-R-008 | SEC | Admin API VPN/IP bypass — misconfigured IP allowlist exposes admin endpoints to the public internet | 1 | 3 | 3 | Test from non-allowlisted IP verifies 403; integration test verifying middleware ordering | DevOps |
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

**System-Level Risk Inheritance:**
- E12-R-001 extends system-level R-002 (multi-tenant data isolation) into analytics materialized views
- E12-R-002 extends system-level R-012 (tier gating accuracy) into Professional+ dashboard gates
- E12-R-005 relates to system-level R-008 (PostgreSQL shared instance) — rate limiting protects shared DB from Enterprise API overload

---

## Entry Criteria

- [ ] E05, E06, E07, E08 features deployed and stable in staging
- [ ] Test data seeding API operational (TB-01 from system-level design)
- [ ] AI Gateway mock mode available for pipeline forecasting agent responses (TB-02)
- [ ] Stripe test mode configured for usage/subscription data (TB-03)
- [ ] PostgreSQL materialized views deployed via migration (S12.01)
- [ ] Celery Beat configured and running in staging
- [ ] SendGrid sandbox mode configured for email delivery tests
- [ ] VPN/IP allowlist configured for Admin API in staging environment

## Exit Criteria

- [ ] All P0 tests passing (100%) — 19 atomic tests in 4 spec files
- [ ] All P1 tests passing (≥ 95% or failures triaged and accepted)
- [ ] No open P0/P1 severity bugs in E12 scope
- [ ] Cross-tenant analytics isolation validated across all 6 dashboard domains
- [ ] Tier gate enforcement validated on all Professional+ endpoints (4 tiers × 2 endpoint groups)
- [ ] Load test results documented with p95 < 500ms for analytics reads under 50 concurrent users
- [ ] OWASP checklist completed with all items addressed or risk-accepted with rationale
- [ ] All production secrets rotated and documented

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on if time-constrained), NOT execution timing. All functional tests run in PRs unless they require dedicated infrastructure. See "Execution Strategy" section for timing.

**Test count note:** P0 has 9 scenario rows that expand to **19 atomic test cases** in the ATDD phase (see `atdd-checklist-e12-p0.md`). Parametrized tier variants, multi-auth-state, and multi-key scenarios each produce separate atomic tests.

### P0 (Critical)

**Criteria:** Blocks core functionality + High risk (score ≥ 6) + No workaround + Affects majority of users/revenue

| Requirement | Test Level | Risk Link | Atomic Tests | ATDD File | Notes |
|-------------|------------|-----------|--------------|-----------|-------|
| Cross-tenant isolation — Company A cannot access Company B data across all 6 analytics domains (market, ROI, team, competitor, pipeline, usage) | API | E12-R-001 | 2 | `analytics/cross-tenant-isolation.api.spec.ts` | Seed Company A + B with analytics data; authenticate as A; query all 6 domains; assert zero B records |
| Tier gate — competitor intelligence endpoints: Free → 403, Starter → 403, Professional → 200, Enterprise → 200 | API | E12-R-002 | 4 | `analytics/tier-gate-enforcement.api.spec.ts` | Parametrized per tier; verify 403 includes upgrade prompt with `required_tier` field |
| Tier gate — pipeline forecasting endpoints: Free → 403, Starter → 403, Professional → 200 | API | E12-R-002 | 3 | `analytics/tier-gate-enforcement.api.spec.ts` | Same parametrized pattern; Enterprise implicitly covered by Professional |
| Admin API — non-allowlisted IP receives 403; missing JWT receives 401 | API | E12-R-008 | 2 | `admin/admin-access-control.api.spec.ts` | Simulated non-VPN request → 403; no auth header → 401 |
| Admin API — regular user JWT receives 403; admin JWT with admin role claim receives 200 | API | E12-R-008 | 2 | `admin/admin-access-control.api.spec.ts` | Standard user role → 403; admin role claim → 200 |
| Enterprise API — valid/invalid/missing/revoked X-API-Key returns correct HTTP status | API | E12-R-005 | 4 | `enterprise/enterprise-api-auth.api.spec.ts` | Valid → 200; invalid → 401; missing → 401; revoked → 401 |
| Enterprise API — rate limiting: 429 returned when limit exceeded; rate limit headers present in all responses | API | E12-R-005 | 2 | `enterprise/enterprise-api-auth.api.spec.ts` | Burst to exceed limit → 429; verify X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset present |
| OWASP security checklist — automated ZAP scan against staging; zero high/critical findings | Infra | E12-R-003 | 1 | *(k8s/ZAP pipeline job)* | Not a Playwright test; coverage: all E12 API endpoints |
| Load test — analytics dashboard p95 < 500ms under 50 concurrent users | Perf | E12-R-006 | 1 | *(k6 script)* | Not a Playwright test; target: market + ROI + team endpoints; document p50/p95/p99 |

**Total P0:** 9 scenarios → 19 atomic tests (17 Playwright API + 2 infrastructure); ~15–25 hours

**ATDD Status:** RED phase complete (2026-04-05). All 17 Playwright API tests generated with `test.skip()`. GREEN phase begins when S12.01–S12.15 endpoints are deployed to staging.

---

### P1 (High)

**Criteria:** Important features + Medium risk (score 3–5) + Common workflows + Workaround difficult

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| S12.01: Materialized views — all 5 domains (market, ROI, team, competitor, usage) created with correct schema and unique indexes for CONCURRENTLY | API | E12-R-006 | 1 | QA | Query each view; verify column presence, data types, and unique index |
| S12.01: Celery Beat refresh schedule — daily for market/ROI/team/competitor, hourly for usage | API | E12-R-006 | 1 | QA | Trigger manual refresh via management command; verify view data updated post-trigger |
| S12.01: REFRESH CONCURRENTLY — concurrent reads not blocked during refresh | API | E12-R-006 | 1 | QA | Start refresh; concurrent read returns data without lock wait timeout |
| S12.02: Market intelligence API — volume, values, authorities, trends with filter combinations; empty-result edge case | API | — | 1 | QA | Date range + sector + country filter combos; empty result returns 200 with empty array |
| S12.04: ROI tracker API — summary (total invested, total won, ROI %), per-bid breakdown, trend over time | API | — | 1 | QA | Verify ROI % = (won − invested) / invested × 100; pagination on per-bid endpoint |
| S12.05: Team performance API — leaderboard all metrics; individual user row-level enforcement | API | — | 1 | QA | Company admin sees all users; regular user sees only own metrics (403 on other user IDs) |
| S12.06: Competitor intelligence API — profiles, patterns, benchmarks data structure (Professional+ only) | API | E12-R-002 | 1 | QA | Via staging seed data; verify competitor profile fields; tier gate enforced per P0 |
| S12.07: Pipeline forecasting API — predictions with confidence scores in range [0.0–1.0] (Professional+ only) | API | E12-R-002 | 1 | QA | Via AI Gateway mock (TB-02); verify confidence score bounds and prediction structure |
| S12.08: Usage dashboard API — consumed/limit/remaining per usage type; billing period start/end dates | API | — | 1 | QA | Verify consumed + remaining = limit; billing period dates present; all paid tiers |
| S12.09: Report generation — PDF output: non-empty file, correct MIME (application/pdf), headers + tables present | API | E12-R-004 | 1 | QA | Generate pipeline summary PDF via Celery task; download via signed S3 URL (24h expiry) |
| S12.09: Report generation — DOCX output: non-empty file, correct MIME, headers + tables present | API | E12-R-004 | 1 | QA | Same template as PDF via python-docx; verify shared generation code reuse from E07 |
| S12.10: Scheduled report delivery — Celery Beat trigger generates report and emails via SendGrid sandbox | API | E12-R-007 | 1 | QA | Configure weekly schedule; trigger via management command; verify SendGrid sandbox receipt with attachment |
| S12.10: On-demand report — async job → signed S3 URL with 24h expiry returned when ready; in-app download link | API | — | 1 | QA | POST generate request; poll status endpoint; verify URL has 24h expiry; download resolves |
| S12.11: Admin tenant management — list (paginated + filtered), detail (usage + subscription), tier override with audit log entry | API | — | 1 | QA | CRUD operations; verify audit log entry created on tier override with reason field |
| S12.13: Admin audit log — search by user/action/entity/date; CSV export streaming response | API | — | 1 | QA | Filter by each dimension; verify streaming CSV Content-Type and disposition headers |
| S12.15: Enterprise API — OpenAPI spec auto-generated; Swagger UI at `/v1/docs`; Redoc at `/v1/redoc` accessible | API | — | 1 | QA | GET both docs endpoints return 200 with correct Content-Type (text/html) |
| S12.15: API key CRUD — create returns full key once; list returns masked (last 4 chars); revoke → 401 on next call | API | E12-R-009 | 1 | QA | Create → full key in response + bcrypt hash in DB (not raw); list → masked display; revoked key → 401 |
| S12.17: Load test — search, opportunity detail, proposal generation, analytics under target concurrency (extended suite) | Perf | — | 1 | QA | k6 script; document p50/p95/p99 and error rates; p95 < 500ms target for read endpoints |

**Total P1:** 18 tests, ~20–35 hours

---

### P2 (Medium)

**Criteria:** Secondary features + Low risk (score 1–2) + Edge cases + Regression prevention

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| S12.03: Market intelligence frontend — bar chart, line chart, sortable authorities table, filter controls, responsive layout | E2E | — | 1 | QA | Charts render with data; date/sector/country filters update all visualizations; mobile stacking viewport |
| S12.04: ROI tracker frontend — summary cards (invested/won/ROI%), per-bid table (sortable), trend chart with date range filter | E2E | — | 1 | QA | Cards display correct metrics; table sorting by investment/outcome/ROI; chart updates on filter |
| S12.05: Team performance frontend — user cards, sortable leaderboard, activity bar chart; admin vs regular user view difference | E2E | — | 1 | QA | Leaderboard column sorting; admin sees all team members; regular user sees only self |
| S12.06: Competitor intelligence frontend — profile cards, 2–4 competitor comparison table, pattern chart (Professional+ only) | E2E | — | 1 | QA | Select 2–4 competitors for side-by-side comparison; pattern chart renders; tier gate reflected in UI |
| S12.07: Pipeline forecasting frontend — timeline/calendar view, color-coded confidence indicators, filters, empty state | E2E | — | 1 | QA | Green/amber/red confidence badges; sector/value/threshold filters; empty state when no predictions |
| S12.08: Usage dashboard frontend — circular meters, 80% warning indicator, upgrade CTA at 100% | E2E | — | 1 | QA | Warning shown at >80% of limit; upgrade CTA links to billing page; available on all paid tiers |
| S12.10: Report schedule configuration — company admin form (weekly/monthly, type, format, recipients) and reports list page | E2E | — | 1 | QA | Configure schedule; verify persistence; update recipients; past reports list shows download links + timestamps |
| S12.14: Admin tenant management frontend — searchable table, tier badges, detail drawer, tier override form | E2E | — | 1 | QA | Search by name; filter by tier; detail drawer shows subscription + usage; override with reason reflects immediately |
| S12.14: Admin crawler management frontend — run history table with status badges, schedule config form, manual trigger | E2E | — | 1 | QA | Status badges (success/failed/running); manual trigger shows confirmation dialog; run detail modal |
| S12.14: Admin white-label frontend — logo upload, color pickers, subdomain input, live preview panel | E2E | — | 1 | QA | Change primary color → preview updates in real time; logo upload shows preview; subdomain validation inline |
| S12.14: Admin audit log frontend — filter bar, paginated results, CSV export download | E2E | — | 1 | QA | Apply user + action + date filters; results update; CSV export triggers download |
| S12.14: Admin platform analytics frontend — funnel visualization, tier distribution pie, MRR line chart, metric cards | E2E | — | 1 | QA | All chart types render with data; metric cards display MRR, churn rate; funnel shows stage progression |
| S12.16: Enterprise API documentation page — embedded Swagger UI or Redoc renders and interactive exploration works | E2E | — | 1 | QA | Interactive endpoint exploration functional; page gated to Enterprise tier admins |
| S12.16: API key management frontend — list masked keys, create (show full key once + clipboard), revoke with confirmation | E2E | — | 1 | QA | Full key shown exactly once on creation; copy-to-clipboard; revoke requires confirmation dialog |
| S12.12: Admin crawler schedule — GET/PUT schedule per crawler type persists correctly | API | — | 1 | QA | PUT schedule → GET returns updated cron expression + enabled state |
| S12.12: Admin crawler trigger — manual crawl starts and returns run_id; status polling tracks progress | API | — | 1 | QA | POST trigger → run_id in response; GET /admin/crawlers/runs/{run_id} transitions to completed |
| S12.12: Admin white-label — subdomain uniqueness enforced (409 on duplicate); DNS readiness check | API | E12-R-011 | 1 | QA | Duplicate subdomain → 409; DNS check returns readiness boolean with detail message |
| S12.13: Admin platform analytics — funnel counts, tier distribution, usage aggregates, revenue metrics present | API | — | 1 | QA | Verify non-null structure; MRR calculation consistent with subscription data |
| S12.15: Rate limit headers — X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset on every Enterprise API response | API | E12-R-005 | 1 | QA | Headers present on both 200 and 429 responses |
| S12.17: Performance optimization — EXPLAIN ANALYZE on all analytics queries; N+1 queries resolved | Code Review | — | 1 | QA | Code review gate: index coverage report + N+1 detection via query log analysis |

**Total P2:** 20 tests, ~8–15 hours

---

### P3 (Low)

**Criteria:** Nice-to-have + Polish + Exploratory

| Requirement | Test Level | Test Count | Owner | Notes |
|-------------|------------|------------|-------|-------|
| S12.18: Onboarding wizard — first login detection triggers overlay; wizard steps (company profile, first search, first opportunity, feature tour) | E2E | 1 | QA | First login → wizard appears; step 1 spotlighted; navigating wizard steps works |
| S12.18: Onboarding wizard — skip/dismiss persists; completion sets `onboarding_completed` flag; no reappearance | E2E | 1 | QA | Skip → absent on next login; complete → flag set → no reappearance |
| S12.18: Empty states — all E12 pages show appropriate empty state when data is absent | E2E | 1 | QA | Analytics with no data shows message + CTA; reports list with no reports shows empty state |
| S12.18: Error pages (404/500/403) — branded templates with correct status codes; public pages have correct meta/OG tags | E2E | 1 | QA | Navigate to invalid route → branded 404; direct 403/500 pages render; meta tags present on public pages |

**Total P3:** 4 tests, ~2–5 hours

---

## Execution Strategy

**Philosophy:** Run all functional tests in every PR unless they require dedicated infrastructure (load runners, security scanners). Playwright with parallelization handles the full functional suite in ~10–15 min.

### Every PR: Playwright + pytest Tests (~10–15 min)

All P0 (17 Playwright API tests), P1 (18 tests), P2 (20 tests), and P3 (4 tests) functional tests run via Playwright and pytest with parallelization across 4 shards. Total: ~59 functional tests.

**Why run in PRs:** Fast feedback; no expensive infrastructure needed; API tests have no browser overhead; frontend E2E tests complete in parallelized Playwright execution.

### Nightly: k6 Performance Tests (~30–60 min)

- Analytics dashboard latency under load (P0 load test — 50 concurrent users)
- Core user flow latency benchmarks: search → opportunity detail → proposal generation (P1)
- Report generation concurrency stress test: 10 concurrent PDF + DOCX generations (E12-R-004)
- Enterprise API rate limit burst test under realistic key concurrency (E12-R-005)

**Why defer to nightly:** Requires dedicated staging environment and k6 runner; meaningful results need stable baseline environment; 30+ min runtime unsuitable for PR gates.

### Weekly: Security Scan (~1–2 hours)

- OWASP ZAP authenticated scan against staging covering all E12 API endpoints (P0 — E12-R-003)
- Network policy verification for Admin API VPN/IP restriction (E12-R-008)
- API key hash storage code review gate (bcrypt/scrypt verification — E12-R-009)

**Why defer to weekly:** Long-running; requires ZAP infrastructure; stable baseline environment needed; security landscape changes slowly.

---

## Resource Estimates

### Test Development Effort

| Priority | Scenarios | Atomic Tests | Effort Range | Status | Notes |
|----------|-----------|--------------|-------------|--------|-------|
| P0 | 9 | 19 (17 Playwright + 2 infra) | ~15–25 hours | RED phase complete | Cross-tenant isolation, tier gates, security scans, load tests — complex setup |
| P1 | 18 | 18 | ~20–35 hours | Not started | API endpoint validation, report generation, admin CRUD — standard complexity |
| P2 | 20 | 20 | ~8–15 hours | Not started | Frontend E2E, admin pages, secondary API tests — straightforward |
| P3 | 4 | 4 | ~2–5 hours | Not started | Onboarding wizard, empty states, error pages — simple E2E |
| **Total** | **51** | **~61** | **~45–80 hours** | | **~1.5–2.5 weeks (1 QA)** |

### Test Data Factories Required

| Factory | Purpose | Status |
|---------|---------|--------|
| `companyWithTier(tier)` | Seed company with specific subscription tier + auth token | Needed (tracked in ATDD checklist) |
| `crossTenantPair()` | Seed two companies with analytics data in all 6 domains | Needed |
| `adminUser()` | Platform admin JWT from VPN-allowlisted context | Needed |
| `enterpriseApiKey(options)` | Valid hashed API key; supports `{ revoked: true }` variant | Needed |
| `analyticsData(companyId, domain)` | Seed materialized view source records per domain | Needed |
| `reportSchedule(options)` | Weekly/monthly report schedule with type, format, recipients | Needed |
| `crawlerRun(status)` | Crawler run history records for admin tests | Needed |

### Tooling

- **Playwright** — E2E + API tests (functional suite)
- **pytest + pytest-asyncio** — Backend integration tests (Celery task, DB-level assertions)
- **k6** — Performance/load tests
- **OWASP ZAP** — Automated security scanning
- **SendGrid sandbox** — Email delivery testing

### Environment Prerequisites

| Requirement | Required By | Status |
|-------------|-------------|--------|
| E05–E08 features stable in staging | Sprint 13 start | Prerequisite |
| Test data seeding API (TB-01) | Sprint 13 start | From system-level design |
| AI Gateway mock mode (TB-02) | Sprint 13 (S12.07) | From system-level design |
| Stripe test mode (TB-03) | Sprint 13 | From system-level design |
| PostgreSQL materialized views (S12.01 migration) | Sprint 13 | E12 scope |
| Celery Beat running in staging | Sprint 13 | E12 scope |
| SendGrid sandbox mode configured | Sprint 13 (S12.10) | E12 scope |
| VPN/IP allowlist on staging Admin API | Sprint 13 | E12 scope |
| k6 runner (cloud or self-hosted) | Sprint 14 (S12.17) | E12 scope |
| OWASP ZAP tooling | Sprint 14 (S12.17) | E12 scope |

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions — 19 atomic tests must all pass)
- **P1 pass rate:** ≥ 95% (waivers required for any failure)
- **P2/P3 pass rate:** ≥ 90% (informational; block on P2 failures in admin auth area)
- **High-risk mitigations:** 100% complete or documented waivers before sprint-end

### Coverage Targets

- **Critical paths (analytics isolation, tier gates, admin auth):** ≥ 80%
- **Security scenarios (SEC category):** 100%
- **Business logic (calculations, aggregations):** ≥ 70%
- **Edge cases (empty data, boundary values, filter combinations):** ≥ 50%

### Non-Negotiable Requirements

- [ ] All 19 P0 atomic tests pass (cross-tenant isolation, tier gates, admin auth, enterprise API)
- [ ] No high-risk (score ≥ 6) items unmitigated at sprint end
- [ ] Security tests (E12-R-001, R-002, R-003, R-005, R-008) pass 100%
- [ ] Load test targets met (p95 < 500ms for analytics reads under 50 concurrent users)
- [ ] OWASP checklist completed with all items addressed or risk-accepted with rationale

---

## Mitigation Plans

### E12-R-001: Cross-Tenant Data Leakage in Analytics (Score: 6)

**Mitigation Strategy:**
1. **Code review:** Verify all materialized view SQL definitions include `company_id` in WHERE clauses; verify each Celery refresh task scopes to a single company or uses per-company partitioned refresh
2. **Automated cross-tenant API tests:** Seed Company A and Company B with analytics data across all 6 domains; authenticate as Company A; query all analytics endpoints; assert zero Company B records returned
3. **DB-level verification:** Query materialized views directly with a mismatched `company_id` to confirm row-level scoping at the PostgreSQL layer

**Owner:** Backend Lead | **Timeline:** Sprint 13, before dashboard API implementation | **Status:** Planned
**Verification:** 2 P0 Playwright API tests in `cross-tenant-isolation.api.spec.ts` — GREEN phase confirms isolation

---

### E12-R-002: Tier Gate Bypass on Professional+ Dashboards (Score: 6)

**Mitigation Strategy:**
1. **Router-level middleware:** Tier gate applied at FastAPI router group level (not per-endpoint) for competitor (`/analytics/competitors/*`) and pipeline (`/analytics/pipeline/*`) route groups — ensures no endpoint is accidentally unguarded
2. **Parametrized API tests:** Every tier (Free, Starter, Professional, Enterprise) × every gated endpoint — 4 competitor + 3 pipeline = 7 test cases
3. **403 response structure:** Verify 403 includes upgrade prompt message and tier requirement field (`"required_tier": "professional"`)

**Owner:** Backend Lead | **Timeline:** Sprint 13, concurrent with S12.06 and S12.07 | **Status:** Planned
**Verification:** 7 P0 Playwright API tests in `tier-gate-enforcement.api.spec.ts` — GREEN phase confirms fix

---

### E12-R-003: OWASP Audit False Confidence (Score: 6)

**Mitigation Strategy:**
1. **OWASP ZAP authenticated scan:** Weekly automated scan against staging covering all E12 endpoints (analytics, admin, enterprise API, report generation)
2. **Manual checklist items:** Items ZAP cannot automate (business logic bypasses, insecure direct object references in materialized views) reviewed manually
3. **Second-engineer review:** Independent QA review of ZAP findings and checklist completeness
4. **Risk-accepted documentation:** All items marked "accept" must include rationale, owner, and re-evaluation date — no silent skips

**Owner:** Security Lead | **Timeline:** Sprint 14 (S12.17) | **Status:** Planned
**Verification:** ZAP scan report with zero high/critical findings; signed-off checklist document committed to repo

---

## Assumptions and Dependencies

### Assumptions

1. Materialized views from S12.01 are deployed and populated with representative test data before dashboard API testing begins — manual refresh via S12.01 management command available
2. Shared report generation code from E07 (proposal export) is stable; E12 tests cover report-specific templates only, not the underlying generation engine
3. KraftData Pipeline Forecasting Agent returns deterministic mock responses for dashboard testing via AI Gateway mock mode (TB-02); real KraftData only in staging smoke tests
4. SendGrid sandbox mode supports attachment delivery testing (email sent to sandbox inbox, not real users)
5. VPN/IP allowlist in staging uses the same enforcement mechanism as production (different IPs, same middleware code path)
6. Materialized view refresh does not lock the view table for reads during the refresh (unique indexes present per S12.01 acceptance criteria)

### Dependencies

| Dependency | Owner | Required By | Epic/Story |
|------------|-------|-------------|------------|
| E05–E08 stability in staging | Full Team | Sprint 13 start | E05–E08 |
| Test data seeding API (TB-01) | Backend Lead | Sprint 13 start | System-level |
| AI Gateway mock mode (TB-02) | Backend Lead | Sprint 13 (S12.07) | System-level |
| Stripe test mode (TB-03) | Backend Lead | Sprint 13 | System-level |
| SendGrid sandbox configuration | DevOps | Sprint 13 (S12.10) | S12.10 |
| k6 cloud or self-hosted runner | DevOps | Sprint 14 (S12.17) | S12.17 |
| OWASP ZAP tooling | DevOps | Sprint 14 (S12.17) | S12.17 |

### Risks to Plan

- **Risk:** Materialized views contain stale or incomplete data in staging → false test failures on dashboard APIs
  **Contingency:** Manual refresh command (S12.01 AC) + dedicated analytics seed data factory via TB-01

- **Risk:** Load test infrastructure (k6) not available by Sprint 14 → performance targets unverified at launch
  **Contingency:** Run k6 locally against staging with reduced concurrency (20 users vs 50); document as limitation with plan to retest post-launch

- **Risk:** SendGrid sandbox does not support attachment delivery in test mode → scheduled report email tests cannot verify attachment content
  **Contingency:** Verify email sent (SendGrid webhook event) + Celery task status; assert attachment byte size > 0 from task result; skip email client rendering test

---

## Interworking & Regression

| Service/Component | E12 Changes | Regression Scope |
|-------------------|------------|-----------------|
| **Client API** | New analytics endpoints (market, ROI, team, competitor, pipeline, usage); report generation; onboarding flow | Existing auth, opportunity search, proposal endpoints must pass post-E12 |
| **Admin API** | New tenant management, crawler management, white-label, audit log, platform analytics endpoints | Existing E02 admin auth tests and E11 compliance admin tests must still pass |
| **Notification Service** | Scheduled report delivery via Celery Beat + SendGrid; new report generation Celery tasks | Existing alert delivery, digest notifications must still pass |
| **Data Pipeline** | Crawler management endpoints expose run history and Celery Beat schedule control | Existing E04/E05 crawl pipeline tests must still pass |
| **AI Gateway** | Pipeline forecasting dashboard consumes prediction agent responses | Existing AI summary, proposal generation, compliance check tests must still pass |
| **Enterprise API** | New surface at `api.eusolicit.com/v1/*` with API key auth + rate limiting | New surface; no regression from prior epics; verify not accessible without X-API-Key |

---

## Follow-on Workflows

### ATDD (P0 — Complete / P1–P3 — Pending)

- **P0 RED phase:** Complete — 19 atomic tests in 4 spec files (`atdd-checklist-e12-p0.md`)
- **Next:** GREEN phase — remove `test.skip()` one domain at a time as endpoints are implemented; run `npx playwright test --grep "@P0"` per domain
- **P1/P2/P3 ATDD:** Run `*atdd` workflow after S12.01–S12.18 implementation; reference this test design as input

### Automation Expansion

- Run `*automate` workflow after implementation to expand P1/P2 coverage into full automation suite

### Execution Commands (P0)

```bash
# Run all E12 P0 tests (currently all skipped — GREEN phase)
npx playwright test --grep "@P0" e2e/specs/analytics/ e2e/specs/admin/ e2e/specs/enterprise/

# Run specific domain
npx playwright test e2e/specs/analytics/cross-tenant-isolation.api.spec.ts
npx playwright test e2e/specs/analytics/tier-gate-enforcement.api.spec.ts
npx playwright test e2e/specs/admin/admin-access-control.api.spec.ts
npx playwright test e2e/specs/enterprise/enterprise-api-auth.api.spec.ts

# Run in headed mode (for debugging)
npx playwright test --headed e2e/specs/admin/admin-access-control.api.spec.ts
```

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

- `risk-governance.md` — Risk classification framework (P × I scoring, gate decisions)
- `probability-impact.md` — Risk scoring methodology (1–3 scales, DOCUMENT/MONITOR/MITIGATE/BLOCK thresholds)
- `test-levels-framework.md` — Test level selection (E2E vs API vs Unit)
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

### Related Documents

| Document | Path |
|----------|------|
| Epic 12 | `eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md` |
| System-Level Test Design (Architecture) | `eusolicit-docs/test-artifacts/test-design-architecture.md` |
| System-Level Test Design (QA) | `eusolicit-docs/test-artifacts/test-design-qa.md` |
| E12 P0 ATDD Checklist | `eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md` |
| PRD | `eusolicit-docs/EU_Solicit_PRD_v1.md` |
| Architecture | `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md` |

---

**Generated by:** BMad TEA Agent — Test Architect Module
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
