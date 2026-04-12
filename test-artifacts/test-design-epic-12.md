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
  - 'eusolicit-docs/implementation-artifacts/12-1-analytics-materialized-views-refresh-infrastructure.md'
  - 'eusolicit-docs/implementation-artifacts/12-2-market-intelligence-dashboard-api.md'
  - 'eusolicit-docs/implementation-artifacts/12-3-market-intelligence-dashboard-frontend.md'
  - 'knowledge/risk-governance.md'
  - 'knowledge/probability-impact.md'
  - 'knowledge/test-levels-framework.md'
  - 'knowledge/test-priorities-matrix.md'
---

# Test Design: Epic 12 — Analytics, Reporting & Admin Platform

**Date:** 2026-04-12
**Author:** TEA Master Test Architect
**Status:** Final — P0 ATDD Red Phase Complete; P1–P3 ATDD Pending; S12.01–S12.02 Implemented
**Version:** v5 (2026-04-12) — incorporates implementation artifacts from S12.01 (MV names, index columns, management command) and S12.02 (tier gate code paths, exact response schemas, `require_professional_plus_tier` stub risk); supersedes v4 (2026-04-12)
**Project:** EU Solicit
**Epic Reference:** `eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md`
**System-Level Reference:** `test-design-architecture.md` (v4, 2026-04-09)
**ATDD Reference:** `atdd-checklist-e12-p0.md` (2026-04-05) — 19 Playwright API tests, 4 spec files, RED phase complete

---

## Executive Summary

**Scope:** Epic-level test design for E12 — Analytics dashboards (market intelligence, ROI tracker, team performance, competitor intelligence, pipeline forecasting, usage), report generation engine (PDF via reportlab, DOCX via python-docx), scheduled and on-demand report delivery via Celery Beat + SendGrid, VPN-restricted Admin API (tenant management, crawler management, white-label configuration, audit log, platform analytics), Enterprise API surface (API key auth, Redis token-bucket rate limiting, OpenAPI auto-documentation), performance optimization, load testing (k6), OWASP security audit, and first-run onboarding wizard.

**Sprint:** 13–14 | **Points:** 55 | **Dependencies:** E05, E06, E07, E08 | **Milestone:** MVP Launch

**Implementation Progress (as of 2026-04-12):**
- S12.01 Analytics Materialized Views: ✅ **done** — `mv_market_intelligence`, `mv_roi_tracker`, `mv_team_performance`, `mv_competitor_intelligence`, `mv_usage_consumption` deployed; unique indexes and `REFRESH CONCURRENTLY` confirmed in migration `011_analytics_materialized_views.py`; tea automation: 🔄 in-progress
- S12.02 Market Intelligence Dashboard API: ✅ **done** — All 4 endpoints deployed; `require_paid_tier` dependency implemented; company_id scoping at service layer confirmed
- S12.03 Market Intelligence Dashboard Frontend: 🔄 **in-progress** — data-testids defined in story, implementation underway
- S12.04–S12.18: ⏳ backlog

**Risk Summary:**

- Total risks identified: 12 (updated from v4: +1 new E12-R-012 for `require_professional_plus_tier` stub)
- High-priority risks (score ≥ 6): 3 (unchanged)
- Critical categories: SEC (cross-tenant analytics leakage, OWASP coverage), BUS (tier gate bypass on Professional+ dashboards, including stub risk)

**Coverage Summary:**

- P0 scenarios: 9 (~21 atomic tests: 19 Playwright API + 2 infra, ~15–25 hours) — RED phase complete
- P1 scenarios: 19 (~20–35 hours) — +1 from v4 for stub gate verification
- P2 scenarios: 20 (~8–15 hours)
- P3 scenarios: 4 (~2–5 hours)
- **Total atomic tests:** ~64 (62 Playwright functional + 2 infrastructure)
- **Total effort:** ~45–80 hours (~1.5–2.5 weeks, 1 QA)

**ATDD Status:** P0 RED-phase tests generated (2026-04-05). 19 Playwright API tests across 4 spec files — all using `test.skip()`. GREEN phase begins when S12.01–S12.15 endpoints are deployed to staging. P0 for S12.01/S12.02 preconditions are now satisfied in staging (stories done); tier gate tests can be un-skipped when S12.06 Professional+ gate is implemented.

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **KraftData AI agent output quality** | Owned by KraftData eval-runs; E12 consumes pipeline forecasting predictions via AI Gateway, does not generate AI content | Integration tested via AI Gateway mock responses (TB-02) |
| **Stripe billing lifecycle** | Covered in E06 test design; E12 usage dashboard reads existing subscription data | Usage dashboard tests assume valid subscription state via test data seeding (TB-01) |
| **Proposal generation and export** | Covered in E07; S12.09 reuses shared document generation code from E07 (`python-docx`, `reportlab`) | S12.09 tests report-specific templates only; E07 regression suite must be green |
| **Data pipeline crawl logic** | Covered in E04/E05; S12.12 tests admin CRUD for crawlers, not crawl parsing | Admin endpoints tested for management operations only |
| **Calendar sync (Google/Microsoft)** | Covered in E08; no calendar features in E12 scope | N/A |
| **ESPD XML conformance** | Covered in E11 test design; E12 has no ESPD scope | N/A |

---

## Risk Assessment

### High-Priority Risks (Score ≥ 6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E12-R-001** | **SEC** | Cross-tenant data leakage in analytics — materialized views aggregate data across companies; missing `company_id` scoping in view definitions or API queries exposes Company A metrics to Company B. **Implementation note:** S12.02 confirms company_id scoping is enforced at service layer via `WHERE company_id = :company_id` in all SQLAlchemy queries against `mv_market_intelligence`. Similar enforcement must be verified for all remaining MV-backed endpoints (S12.04–S12.08). | 2 | 3 | **6** | Automated cross-tenant isolation tests on every analytics endpoint; verify materialized view SQL includes `company_id` WHERE clauses; DB-level query log verification; service-layer code review for all analytics service functions | Backend Lead | Sprint 13 |
| **E12-R-002** | **BUS** | Tier gate bypass on Professional+ dashboards — `require_professional_plus_tier` in `tier_gate.py` is a **stub** (raises `NotImplementedError`) as of S12.02 implementation. S12.06 and S12.07 must replace this stub with real implementation before competitor/pipeline endpoints go live. If stub reaches production, any request will result in 500 rather than 403 — effectively blocking all users including Professional+ | 2 | 3 | **6** | Verify stub is replaced before S12.06 endpoint wires it in; parametrized tier boundary tests (Free → 403, Starter → 403, Professional → 200, Enterprise → 200) on all gated endpoints; router-level middleware prevents per-endpoint accidents | Backend Lead | Sprint 13 (S12.06) |
| **E12-R-003** | **SEC** | OWASP audit false confidence — incomplete checklist coverage creates false security assurance before launch; missed items surface as post-launch vulnerabilities | 2 | 3 | **6** | Independent checklist review by second engineer; automated OWASP ZAP scan against staging; document risk-accepted items with rationale and owner | Security Lead | Sprint 14 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E12-R-004 | PERF | Report generation resource exhaustion — PDF/DOCX with embedded chart images may spike Celery worker memory under concurrent generation | 2 | 2 | 4 | Worker memory limits + concurrency cap per worker; test with 10 concurrent report generations | Backend Lead |
| E12-R-005 | SEC | Enterprise API rate limit bypass — token bucket implementation flaws allow circumvention via key rotation or timing attacks | 2 | 2 | 4 | Burst testing: exceed limit rapidly, verify 429 + headers; test key rotation doesn't reset rate window | Backend Lead |
| E12-R-006 | PERF | Materialized view refresh blocking reads — if unique index is missing, `REFRESH CONCURRENTLY` falls back to blocking refresh, causing dashboard timeouts. **Implementation note:** S12.01 confirms unique indexes are created: `uq_mv_market_intelligence(company_id, sector, country, month)`, `uq_mv_roi_tracker(company_id, proposal_id)`, `uq_mv_team_performance(company_id, user_id, month)`, `uq_mv_competitor_intelligence(company_id, competitor_name, sector)`, `uq_mv_usage_consumption(company_id, metric_type, period_start)` | 2 | 2 | 4 | Verify all 5 unique indexes exist via migration verification test; test concurrent read availability during simulated refresh; inspect Alembic migration `011_analytics_materialized_views.py` | DBA/Backend |
| E12-R-007 | OPS | Scheduled report email delivery failure — SendGrid outage or template misconfiguration leads to silent report non-delivery with no user notification | 2 | 2 | 4 | Delivery confirmation logging; retry with exponential backoff; test with SendGrid sandbox mode | Backend Lead |
| E12-R-008 | SEC | Admin API VPN/IP bypass — misconfigured IP allowlist exposes admin endpoints to the public internet | 1 | 3 | 3 | Test from non-allowlisted IP verifies 403; integration test verifying middleware ordering | DevOps |
| E12-R-009 | SEC | API key hash storage weakness — keys stored with weak hash (MD5/SHA1) means compromised DB exposes usable keys | 1 | 3 | 3 | Code review verifying bcrypt/scrypt; test that raw key is not retrievable from DB | Backend Lead |
| E12-R-010 | PERF | Celery Beat refresh schedule drift — daily refresh tasks for 4 domains staggered 2:00–2:45 AM per implementation; if Celery Beat worker restarts during the window, some views may miss their refresh cycle, causing stale data for up to 48h on daily-refresh views | 1 | 2 | 2 | Verify Beat schedule survives worker restart in staging; add Celery Beat task history logging; alert on missed task executions | Backend Lead |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E12-R-011 | TECH | Onboarding wizard state persistence — if `onboarding_completed` flag fails to persist, wizard loops on every login | 2 | 1 | 2 | Monitor |
| E12-R-012 | OPS | White-label subdomain DNS validation — DNS readiness check may produce false negatives for propagating records | 1 | 1 | 1 | Monitor |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

**System-Level Risk Inheritance:**
- E12-R-001 extends system-level R-002 (multi-tenant data isolation) into analytics materialized views
- E12-R-002 extends system-level R-012 (tier gating accuracy) into Professional+ dashboard gates; now includes stub-replacement risk from S12.02 implementation
- E12-R-005 relates to system-level R-008 (PostgreSQL shared instance) — rate limiting protects shared DB from Enterprise API overload

---

## Entry Criteria

- [ ] E05, E06, E07, E08 features deployed and stable in staging
- [ ] Test data seeding API operational (TB-01 from system-level design)
- [ ] AI Gateway mock mode available for pipeline forecasting agent responses (TB-02)
- [ ] Stripe test mode configured for usage/subscription data (TB-03)
- [ ] PostgreSQL materialized views deployed via migration `011_analytics_materialized_views.py` (S12.01) ✅ done
- [ ] Celery Beat configured and running in staging with daily/hourly refresh schedule
- [ ] SendGrid sandbox mode configured for email delivery tests
- [ ] VPN/IP allowlist configured for Admin API in staging environment
- [ ] `require_professional_plus_tier` stub replaced with real implementation (gate for tier gate P0 tests)

## Exit Criteria

- [ ] All P0 tests passing (100%) — 21 atomic tests (19 Playwright in 4 spec files + ZAP scan + k6 load test)
- [ ] All P1 tests passing (≥ 95% or failures triaged and accepted)
- [ ] No open P0/P1 severity bugs in E12 scope
- [ ] Cross-tenant analytics isolation validated across all 6 dashboard domains via `cross-tenant-isolation.api.spec.ts`
- [ ] Tier gate enforcement validated on all Professional+ endpoints (4 tiers × 2 endpoint groups) via `tier-gate-enforcement.api.spec.ts`
- [ ] Load test results documented with p95 < 500ms for analytics reads under 50 concurrent users
- [ ] OWASP checklist completed with all items addressed or risk-accepted with rationale
- [ ] All production secrets rotated and documented
- [ ] `require_professional_plus_tier` confirmed as real implementation (not stub) before S12.06 endpoints deploy

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on if time-constrained), NOT execution timing. All functional tests run in PRs unless they require dedicated infrastructure. See "Execution Strategy" section for timing.

**Test count note:** P0 has 9 scenario rows that expand to **21 atomic test cases** in total: 19 Playwright API tests (see `atdd-checklist-e12-p0.md` — 4 spec files, 19 tests) plus 2 infrastructure tests (OWASP ZAP scan, k6 load test). Parametrized tier variants and multi-key scenarios each produce separate atomic tests.

### P0 (Critical)

**Criteria:** Blocks core functionality + High risk (score ≥ 6) + No workaround + Affects majority of users/revenue

| Requirement | Test Level | Risk Link | Atomic Tests | ATDD File | Notes |
|-------------|------------|-----------|--------------|-----------|-------|
| Cross-tenant isolation — Company A cannot access Company B data across all 6 analytics domains (market, ROI, team, competitor, pipeline, usage) | API | E12-R-001 | 2 | `analytics/cross-tenant-isolation.api.spec.ts` | Seed Company A + B with analytics data via TB-01 `crossTenantPair()` factory; authenticate as A; query all 6 domains; assert zero B records. S12.02 confirmed company_id scoping at service layer for market domain — validate same pattern propagates to ROI, team, competitor, pipeline, usage (S12.04–S12.08) |
| Tier gate — competitor intelligence endpoints: Free → 403, Starter → 403, Professional → 200, Enterprise → 200 | API | E12-R-002 | 4 | `analytics/tier-gate-enforcement.api.spec.ts` | Parametrized per tier via `companyWithTier(tier)` factory; verify 403 includes upgrade prompt with `required_tier` field; **prerequisite: `require_professional_plus_tier` stub must be replaced with real implementation before un-skipping these tests** |
| Tier gate — pipeline forecasting endpoints: Free → 403, Starter → 403, Professional → 200 | API | E12-R-002 | 3 | `analytics/tier-gate-enforcement.api.spec.ts` | Enterprise implicitly covered by Professional gate (same router middleware); AI Gateway mock (TB-02) needed for Professional/Enterprise 200 responses |
| Admin API — non-allowlisted IP receives 403; missing JWT receives 401 | API | E12-R-008 | 2 | `admin/admin-access-control.api.spec.ts` | Simulated non-VPN request via request headers → 403; no auth header → 401 |
| Admin API — regular user JWT receives 403; admin JWT with admin role claim receives 200 | API | E12-R-008 | 2 | `admin/admin-access-control.api.spec.ts` | Standard user role → 403; admin role claim in JWT → 200 via `adminUser()` factory |
| Enterprise API — valid/invalid/missing/revoked X-API-Key returns correct HTTP status | API | E12-R-005 | 4 | `enterprise/enterprise-api-auth.api.spec.ts` | Valid → 200; invalid → 401; missing → 401; revoked → 401 via `revokedApiKey()` factory |
| Enterprise API — rate limiting: 429 returned when limit exceeded; rate limit headers present in all responses | API | E12-R-005 | 2 | `enterprise/enterprise-api-auth.api.spec.ts` | Burst to exceed limit → 429; verify `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` present in both 200 and 429 responses |
| OWASP security checklist — automated ZAP scan against staging; zero high/critical findings | Infra | E12-R-003 | 1 | *(k8s/ZAP pipeline job)* | Not a Playwright test; coverage: all E12 API endpoints; weekly CI job |
| Load test — analytics dashboard p95 < 500ms under 50 concurrent users | Perf | E12-R-006 | 1 | *(k6 script)* | Not a Playwright test; target: market + ROI + team endpoints; document p50/p95/p99 |

**Total P0:** 9 scenarios → 21 atomic tests (19 Playwright API + 2 infrastructure); ~15–25 hours

**ATDD Status:** RED phase complete (2026-04-05). All 19 Playwright API tests generated with `test.skip()` across 4 spec files:
- `cross-tenant-isolation.api.spec.ts` — 2 tests (S12.01/S12.02 preconditions satisfied; un-skip when S12.04–S12.08 complete for full domain sweep)
- `tier-gate-enforcement.api.spec.ts` — 7 tests (4 competitor + 3 pipeline; un-skip after `require_professional_plus_tier` stub is replaced in S12.06)
- `admin-access-control.api.spec.ts` — 4 tests (un-skip when S12.11–S12.13 Admin API deployed)
- `enterprise-api-auth.api.spec.ts` — 6 tests (4 key auth + 2 rate limit; un-skip when S12.15 deployed)

GREEN phase begins domain-by-domain as endpoints land in staging.

---

### P1 (High)

**Criteria:** Important features + Medium risk (score 3–5) + Common workflows + Workaround difficult

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| S12.01: Materialized views — all 5 domains created with correct schema, column types, and unique indexes for CONCURRENTLY. Verify: `uq_mv_market_intelligence(company_id, sector, country, month)`, `uq_mv_roi_tracker(company_id, proposal_id)`, `uq_mv_team_performance(company_id, user_id, month)`, `uq_mv_competitor_intelligence(company_id, competitor_name, sector)`, `uq_mv_usage_consumption(company_id, metric_type, period_start)` | API | E12-R-006 | 1 | QA | Query `pg_indexes` or `information_schema.table_constraints` for each unique index; verify column presence and data types via `information_schema.columns` |
| S12.01: Celery Beat refresh schedule — daily (2:00–2:45 AM staggered) for market/ROI/team/competitor, hourly for usage | API | E12-R-006, E12-R-010 | 1 | QA | Trigger ad-hoc full refresh via `scripts/refresh_analytics_views.py --all`; verify view data updated post-trigger; inspect Beat schedule config for cron expressions |
| S12.01: REFRESH CONCURRENTLY — concurrent reads not blocked during refresh; notification_role owns all 5 views (required for REFRESH without MAINTAIN privilege) | API | E12-R-006 | 1 | QA | Start refresh on one view; concurrent SELECT from same view returns data without lock wait timeout (pg_locks check); verify `pg_matviews.matviewowner = 'notification_role'` for all 5 |
| **S12.02/S12.06: `require_professional_plus_tier` stub replacement** — verify stub raises `NotImplementedError` in current implementation; confirm real implementation is wired before S12.06 endpoints deploy | API | E12-R-002 | 1 | QA | Inspect `services/client-api/src/client_api/core/tier_gate.py`: stub must be replaced with real check (analogous to `require_paid_tier`) before S12.06 deploys; this is a pre-condition gate for tier-gate P0 tests |
| S12.02: Market intelligence API — 4 endpoints respond correctly with filter combinations; empty-result edge case returns 200 with `{"items": [], "total": 0}` (paginated) or `{"items": []}` (list); company_id scoping verified | API | — | 1 | QA | Date range + sector + country filter combos via `tests/api/test_analytics_market.py` patterns; empty result returns 200 not 4xx; `Cache-Control: public, max-age=1800` header present on all 4 endpoints |
| S12.02: Free-tier rejection — 403 with `{"message": "This feature requires a paid subscription.", "upgrade_required": true}` body on all 4 market endpoints; missing/invalid JWT → 401 | API | E12-R-002 | 1 | QA | Test against `require_paid_tier` dependency path; verify exact JSON response body matches schema; confirm `upgrade_required: true` field present |
| S12.04: ROI tracker API — `GET /analytics/roi/summary` returns total invested, total won, ROI %; ROI % = `(won − invested) / invested × 100`; per-bid pagination; trend over time | API | — | 1 | QA | Seed bid_preparation_logs + bid_outcomes via TB-01; verify ROI % calculation correctness; per-bid endpoint paginates |
| S12.05: Team performance API — leaderboard all metrics; individual user row-level enforcement (admin sees all; regular user sees only own via company-role check) | API | — | 1 | QA | Company admin JWT → all users in leaderboard; regular user JWT → `GET /analytics/team/user/{other_user_id}` returns 403 |
| S12.06: Competitor intelligence API — profiles, patterns, benchmarks (Professional+ only) | API | E12-R-002 | 1 | QA | Via staging seed data; verify competitor profile fields: name, bid_count, estimated_win_rate, sectors; tier gate enforced per P0; stub replacement confirmed |
| S12.07: Pipeline forecasting API — predictions with confidence scores in range [0.0–1.0] (Professional+ only) | API | E12-R-002 | 1 | QA | Via AI Gateway mock (TB-02) returning deterministic prediction fixture; verify confidence score bounds; tier gate enforced |
| S12.08: Usage dashboard API — consumed/limit/remaining per usage type (AI summaries, proposal drafts, compliance checks); billing period start/end dates present | API | — | 1 | QA | Verify consumed + remaining = limit arithmetic; billing period start < end; all paid tiers can access; free tier returns 403 |
| S12.09: Report generation — PDF output: non-empty file, correct MIME (`application/pdf`), signed S3 URL with 24h expiry; headers + tables present in document | API | E12-R-004 | 1 | QA | Generate pipeline summary PDF via Celery task; download via signed S3 URL; verify MIME type and non-zero byte size; assert URL expiry behaviour |
| S12.09: Report generation — DOCX output: non-empty file, correct MIME (`application/vnd.openxmlformats-officedocument.wordprocessingml.document`), shared generation code reuse from E07 | API | E12-R-004 | 1 | QA | Same template as PDF via python-docx; verify shared extraction from E07 generation stack; test all 4 template types (pipeline summary, bid performance, team activity, custom date range) |
| S12.10: Scheduled report delivery — Celery Beat trigger generates report and emails via SendGrid sandbox; attachment present in email | API | E12-R-007 | 1 | QA | Configure weekly schedule; trigger via management command; verify SendGrid sandbox receipt with attachment; assert attachment MIME type and byte size > 0 |
| S12.10: On-demand report — async job tracks to completion; signed S3 URL with 24h expiry returned; in-app download link available | API | — | 1 | QA | POST generate → poll status endpoint until `completed`; verify URL has 24h expiry; download resolves to non-empty file |
| S12.11: Admin tenant management — paginated list with search/tier filter; detail (subscription + usage + activity); tier override creates audit log entry with reason field | API | — | 1 | QA | Verify pagination metadata (`total`, `page`, `page_size`); detail returns current-period usage meters; override → `POST /admin/tenants/{id}/tier-override` with `reason` → audit log entry created |
| S12.13: Admin audit log — search by user/action_type/entity_type/date; CSV export streaming response with `Content-Disposition: attachment; filename="audit_log.csv"` | API | — | 1 | QA | Filter by each dimension independently; combined filter; streaming CSV Content-Type `text/csv`; verify at least one data row |
| S12.15: Enterprise API — OpenAPI spec auto-generated; Swagger UI at `/v1/docs`; Redoc at `/v1/redoc` accessible | API | — | 1 | QA | GET both docs endpoints return 200 with `Content-Type: text/html`; spec includes request/response examples |
| S12.15: API key CRUD — create returns full key exactly once; list returns masked (last 4 chars visible); key hash stored as bcrypt/scrypt (not raw); revoked key → 401 | API | E12-R-009 | 1 | QA | Create key → full key in response; verify DB hash is NOT equal to raw key (bcrypt/scrypt prefix); list → masked; revoke → subsequent request with revoked key → 401 |

**Total P1:** 19 tests, ~20–35 hours

---

### P2 (Medium)

**Criteria:** Secondary features + Low risk (score 1–2) + Edge cases + Regression prevention

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| S12.03: Market intelligence frontend — bar chart renders procurement volume by sector; line chart renders monthly trends; sortable/filterable authorities table; date/sector/country filters update all visualizations; responsive layout (charts stack vertically on mobile) | E2E | — | 1 | QA | Playwright E2E; charts render with data (check Recharts canvas or svg elements); filter → data updates; viewport 390px → charts stack. data-testids: `market-filters`, `market-filter-date-from`, `market-filter-date-to`, `market-filter-sector`, `market-filter-country`, `market-filter-apply-btn`, `market-filter-clear-btn`; loading skeletons visible before data |
| S12.04: ROI tracker frontend — summary cards (invested/won/ROI%), per-bid table (sortable), trend chart with date range filter | E2E | — | 1 | QA | Cards display correct metrics; table sorting by investment, outcome, and ROI columns; trend chart updates on date range filter |
| S12.05: Team performance frontend — user cards with metrics, sortable leaderboard table, activity bar chart; admin vs regular user view difference | E2E | — | 1 | QA | Leaderboard column sorting; admin sees all team members; regular user sees only self (other rows absent) |
| S12.06: Competitor intelligence frontend — profile cards, 2–4 competitor comparison table, pattern chart (Professional+ only); lower tier UI shows upgrade prompt | E2E | — | 1 | QA | Select 2–4 competitors for side-by-side comparison; pattern chart renders; non-Professional tier sees upgrade prompt, not data |
| S12.07: Pipeline forecasting frontend — timeline/calendar view, color-coded confidence indicators (green/amber/red), filters (sector, value range, confidence threshold), empty state | E2E | — | 1 | QA | Green/amber/red confidence badges; sector/value/threshold filters functional; empty state displayed when no predictions |
| S12.08: Usage dashboard frontend — circular progress meters; warning indicator at >80% of limit; upgrade CTA at ≥100% links to billing page | E2E | — | 1 | QA | Warning visible at >80%; CTA links to billing; meters render for AI summaries, proposal drafts, compliance checks; available on all paid tiers |
| S12.10: Report schedule configuration — company admin can configure weekly/monthly schedule (type, format, recipients) via settings form; reports list page shows past reports with download links + timestamps | E2E | — | 1 | QA | Form persists schedule; update recipients; past reports list shows download links and generation timestamps |
| S12.14: Admin tenant management frontend — searchable company table with tier badges, detail drawer (subscription info, usage meters, activity summary), tier override form with reason field | E2E | — | 1 | QA | Search by name; filter by tier; detail drawer opens; override with reason reflects change immediately |
| S12.14: Admin crawler management frontend — run history table with status badges (success/failed/running), schedule config form, manual trigger with confirmation dialog, run detail modal | E2E | — | 1 | QA | Status badges render correctly; manual trigger shows confirmation dialog before proceeding; run detail modal opens |
| S12.14: Admin white-label frontend — logo upload with preview, color pickers for primary/accent, subdomain input, live preview panel updates in real time | E2E | — | 1 | QA | Change primary color → preview updates immediately without save; logo upload shows preview; subdomain inline validation |
| S12.14: Admin audit log frontend — filter bar (user search, action type, entity type, date range), paginated results, CSV export download | E2E | — | 1 | QA | Apply user + action + date filters; results update; CSV export triggers browser download with non-empty file |
| S12.14: Admin platform analytics frontend — funnel visualization, tier distribution pie chart, MRR line chart, metric cards (MRR, churn rate) | E2E | — | 1 | QA | All chart types render with data; metric cards show MRR and churn rate; funnel shows stage progression |
| S12.16: Enterprise API documentation page — embedded Swagger UI or Redoc renders; interactive endpoint exploration works; page gated to Enterprise tier company admins | E2E | — | 1 | QA | Interactive exploration functional; non-Enterprise tier sees 403/upgrade prompt |
| S12.16: API key management frontend — list masked keys (last 4 visible), create (full key shown exactly once + copy-to-clipboard), revoke with confirmation dialog, rate limit tier + usage stats displayed | E2E | — | 1 | QA | Full key shown exactly once on creation; copy-to-clipboard works; revoke requires confirmation; stats display |
| S12.12: Admin crawler schedule — GET/PUT schedule per crawler type persists correctly (cron expression + enabled state) | API | — | 1 | QA | PUT schedule → GET returns updated cron expression + enabled state |
| S12.12: Admin crawler trigger — manual crawl starts, returns run_id; status polling tracks progress to completion | API | — | 1 | QA | POST trigger → run_id in response; GET `/admin/crawlers/runs/{run_id}` transitions to `completed` |
| S12.12: Admin white-label — subdomain uniqueness enforced (409 on duplicate); DNS readiness check returns readiness boolean with detail | API | E12-R-012 | 1 | QA | Duplicate subdomain → 409; DNS check returns `{"ready": bool, "detail": string}` |
| S12.13: Admin platform analytics — funnel counts, tier distribution, usage aggregates, revenue metrics all present and non-null | API | — | 1 | QA | Verify non-null structure; MRR calculation consistent with subscription data |
| S12.15: Rate limit headers — `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` present on both 200 and 429 Enterprise API responses | API | E12-R-005 | 1 | QA | Headers present on every response; Remaining decrements on successive calls; Reset is a valid Unix timestamp |
| S12.17: Performance optimization — EXPLAIN ANALYZE on all analytics queries; N+1 queries resolved in API endpoints | Code Review | — | 1 | QA | Code review gate: index coverage report; N+1 detection via query log analysis (SQLAlchemy echo or pg_stat_statements); confirm no more than 2 DB queries per endpoint |

**Total P2:** 20 tests, ~8–15 hours

---

### P3 (Low)

**Criteria:** Nice-to-have + Polish + Exploratory

| Requirement | Test Level | Test Count | Owner | Notes |
|-------------|------------|------------|-------|-------|
| S12.18: Onboarding wizard — first login triggers overlay; wizard steps (company profile, first search, first opportunity, feature tour) with spotlight and instructional tooltips | E2E | 1 | QA | First login → wizard overlay appears; step 1 spotlights company profile area; navigating steps works; skip/close available |
| S12.18: Onboarding wizard — skip/dismiss sets `onboarding_completed` flag; wizard absent on subsequent login; completion does not reappear | E2E | 1 | QA | Skip → flag set → no reappearance on re-login; complete wizard → same |
| S12.18: Empty states — all E12 pages show appropriate empty state when data is absent (analytics with no data, reports list with no reports) | E2E | 1 | QA | Analytics with no seeded data shows empty state message + CTA; reports list with no reports shows empty state |
| S12.18: Error pages (404/500/403) — branded templates with correct status codes; public pages include correct meta tags and OG tags | E2E | 1 | QA | Navigate to invalid route → branded 404; trigger 403/500 pages → branded; public pages have `og:title`, `og:description`, canonical meta tags |

**Total P3:** 4 tests, ~2–5 hours

---

## Execution Strategy

**Philosophy:** Run all functional tests in every PR unless they require dedicated infrastructure (load runners, security scanners). Playwright with parallelization handles the full functional suite in ~10–15 min.

### Every PR: Playwright + pytest Tests (~10–15 min)

All P0 (19 Playwright API tests), P1 (19 tests), P2 (20 tests), and P3 (4 tests) functional tests run via Playwright and pytest with parallelization across 4 shards. Total: ~62 functional tests.

**Why run in PRs:** Fast feedback; no expensive infrastructure needed; API tests have no browser overhead; frontend E2E tests complete in parallelized Playwright execution.

### Nightly: k6 Performance Tests (~30–60 min)

- Analytics dashboard latency under load (P0 load test — 50 concurrent users against `mv_market_intelligence`, `mv_roi_tracker`, `mv_team_performance`)
- Core user flow latency benchmarks: search → opportunity detail → proposal generation (P1)
- Report generation concurrency stress test: 10 concurrent PDF + DOCX generations (E12-R-004)
- Enterprise API rate limit burst test under realistic key concurrency (E12-R-005)
- Celery Beat task refresh timing verification — confirm daily/hourly schedule adherence

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
| P0 | 9 | 21 (19 Playwright + 2 infra) | ~15–25 hours | RED phase complete (2026-04-05) | Cross-tenant isolation, tier gates, security scans, load tests — complex setup |
| P1 | 19 | 19 | ~20–35 hours | Not started (+1 from v4 for stub-gate check) | API endpoint validation, report generation, admin CRUD — standard complexity |
| P2 | 20 | 20 | ~8–15 hours | Not started | Frontend E2E, admin pages, secondary API tests — straightforward |
| P3 | 4 | 4 | ~2–5 hours | Not started | Onboarding wizard, empty states, error pages — simple E2E |
| **Total** | **52** | **~64** | **~45–80 hours** | | **~1.5–2.5 weeks (1 QA)** |

### Test Data Factories Required

| Factory | Purpose | Status |
|---------|---------|--------|
| `companyWithTier(tier)` | Seed company with specific subscription tier (`free`, `starter`, `professional`, `enterprise`) + auth token | Needed (tracked in ATDD checklist) |
| `crossTenantPair()` | Seed two companies with analytics data in all 6 domains across all 5 materialized views | Needed |
| `adminUser()` | Platform admin JWT from VPN-allowlisted context | Needed |
| `enterpriseApiKey(options)` | Valid hashed API key; supports `{ revoked: true }` variant | Needed |
| `analyticsData(companyId, domain)` | Seed materialized view source records per domain; trigger ad-hoc refresh via `scripts/refresh_analytics_views.py` | Needed |
| `reportSchedule(options)` | Weekly/monthly report schedule with type (`pipeline_summary`, `bid_performance`, `team_activity`, `custom_date_range`), format (`pdf`, `docx`), recipients | Needed |
| `crawlerRun(status)` | Crawler run history records (`success`, `failed`, `running`) for admin tests | Needed |

### Tooling

- **Playwright** — E2E + API tests (functional suite)
- **pytest + pytest-asyncio** — Backend integration tests (Celery task, DB-level assertions, pg_indexes verification)
- **k6** — Performance/load tests
- **OWASP ZAP** — Automated security scanning
- **SendGrid sandbox** — Email delivery testing
- **scripts/refresh_analytics_views.py** — Ad-hoc materialized view refresh in tests (S12.01 management command)

### Environment Prerequisites

| Requirement | Required By | Status |
|-------------|-------------|--------|
| E05–E08 features stable in staging | Sprint 13 start | Prerequisite |
| Test data seeding API (TB-01) | Sprint 13 start | From system-level design |
| AI Gateway mock mode (TB-02) | Sprint 13 (S12.07) | From system-level design |
| Stripe test mode (TB-03) | Sprint 13 | From system-level design |
| PostgreSQL materialized views (S12.01 migration `011`) | Sprint 13 | ✅ done |
| Celery Beat running in staging with daily/hourly refresh | Sprint 13 | E12 scope |
| SendGrid sandbox mode configured | Sprint 13 (S12.10) | E12 scope |
| VPN/IP allowlist on staging Admin API | Sprint 13 | E12 scope |
| `require_professional_plus_tier` real implementation | Sprint 13 (S12.06) | ⚠️ currently stub — must be replaced |
| k6 runner (cloud or self-hosted) | Sprint 14 (S12.17) | E12 scope |
| OWASP ZAP tooling | Sprint 14 (S12.17) | E12 scope |

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions — 21 atomic tests must all pass: 19 Playwright + ZAP zero high/critical + k6 p95 < 500ms)
- **P1 pass rate:** ≥ 95% (waivers required for any failure)
- **P2/P3 pass rate:** ≥ 90% (informational; block on P2 failures in admin auth area)
- **High-risk mitigations:** 100% complete or documented waivers before sprint-end

### Coverage Targets

- **Critical paths (analytics isolation, tier gates, admin auth):** ≥ 80%
- **Security scenarios (SEC category):** 100%
- **Business logic (calculations, aggregations):** ≥ 70%
- **Edge cases (empty data, boundary values, filter combinations):** ≥ 50%

### Non-Negotiable Requirements

- [ ] All 19 Playwright P0 API tests pass (cross-tenant isolation, tier gates, admin auth, enterprise API)
- [ ] `require_professional_plus_tier` stub replaced with real implementation before S12.06 deploys
- [ ] ZAP scan: zero high/critical OWASP findings (or all risk-accepted with documented rationale)
- [ ] k6 load test: p95 < 500ms for analytics reads under 50 concurrent users
- [ ] No high-risk (score ≥ 6) items unmitigated at sprint end
- [ ] Security tests (E12-R-001, R-002, R-003, R-005, R-008) pass 100%

---

## Mitigation Plans

### E12-R-001: Cross-Tenant Data Leakage in Analytics (Score: 6)

**Mitigation Strategy:**
1. **Code review:** Verify all 5 materialized view SQL definitions include `company_id` in WHERE/GROUP BY clauses. S12.02 confirmed `WHERE company_id = :company_id` in `analytics_market_service.py` — same pattern must be audited in future analytics service files for ROI (S12.04), team (S12.05), competitor (S12.06), pipeline (S12.07), and usage (S12.08).
2. **Automated cross-tenant API tests:** Seed Company A and Company B with analytics data across all 6 domains via `crossTenantPair()` factory + `analyticsData()` + ad-hoc refresh command; authenticate as Company A; query all analytics endpoints; assert zero Company B records returned.
3. **DB-level verification:** Query materialized views directly with a mismatched `company_id` to confirm row-level scoping at the PostgreSQL layer; use `pg_stat_activity` to monitor for cross-company query bleed.

**Owner:** Backend Lead | **Timeline:** Sprint 13, before S12.04–S12.08 implementation | **Status:** Partial (S12.02 market domain verified)
**Verification:** 2 P0 Playwright API tests in `cross-tenant-isolation.api.spec.ts` — GREEN phase covers full 6-domain sweep

---

### E12-R-002: Tier Gate Bypass / Stub Risk on Professional+ Dashboards (Score: 6)

**Mitigation Strategy:**
1. **Stub replacement gate:** `require_professional_plus_tier` in `tier_gate.py` currently raises `NotImplementedError`. This must be replaced with a real implementation mirroring `require_paid_tier` — checking for `plan IN ('professional', 'enterprise')` — before S12.06 or S12.07 endpoints wire it in. Merging the stub as production code for S12.06 would return 500 (not 403) for all users, blocking paid access entirely.
2. **Router-level middleware:** Tier gate applied at FastAPI router group level (not per-endpoint) for competitor (`/analytics/competitors/*`) and pipeline (`/analytics/pipeline/*`) route groups.
3. **Parametrized API tests:** Every tier (Free, Starter, Professional, Enterprise) × every gated endpoint — 4 competitor + 3 pipeline = 7 test cases in `tier-gate-enforcement.api.spec.ts`.
4. **403 response structure:** Verify 403 includes upgrade prompt message and tier requirement field (`"required_tier": "professional"`).

**Owner:** Backend Lead | **Timeline:** Sprint 13, before S12.06 implementation | **Status:** Planned (stub identified in S12.02 implementation)
**Verification:** 7 P0 Playwright API tests in `tier-gate-enforcement.api.spec.ts` + P1 stub-gate verification test — GREEN phase confirms real implementation

---

### E12-R-003: OWASP Audit False Confidence (Score: 6)

**Mitigation Strategy:**
1. **OWASP ZAP authenticated scan:** Weekly automated scan against staging covering all E12 endpoints (analytics, admin, enterprise API, report generation, onboarding)
2. **Manual checklist items:** Items ZAP cannot automate (business logic bypasses, insecure direct object references in materialized views, materialized view owner privilege scope) reviewed manually
3. **Second-engineer review:** Independent QA review of ZAP findings and checklist completeness
4. **Risk-accepted documentation:** All items marked "accept" must include rationale, owner, and re-evaluation date — no silent skips

**Owner:** Security Lead | **Timeline:** Sprint 14 (S12.17) | **Status:** Planned
**Verification:** ZAP scan report with zero high/critical findings; signed-off checklist document committed to repo

---

## Assumptions and Dependencies

### Assumptions

1. Materialized views from S12.01 are deployed and populated with representative test data before dashboard API testing begins — manual refresh via `scripts/refresh_analytics_views.py --all` available ✅ (S12.01 management command implemented)
2. Shared report generation code from E07 (proposal export) is stable; E12 tests cover report-specific templates only, not the underlying generation engine
3. KraftData Pipeline Forecasting Agent returns deterministic mock responses for dashboard testing via AI Gateway mock mode (TB-02); real KraftData only in staging smoke tests
4. SendGrid sandbox mode supports attachment delivery testing
5. VPN/IP allowlist in staging uses the same enforcement mechanism as production (different IPs, same middleware code path)
6. `require_paid_tier` in `tier_gate.py` is the established pattern for `require_professional_plus_tier` implementation — analogous logic checking plan IN ('professional', 'enterprise')
7. Celery Beat task ownership: materialized views are owned by `notification_role`; REFRESH tasks run under that role without requiring MAINTAIN privilege ✅ (confirmed in S12.01 migration)

### Dependencies

| Dependency | Owner | Required By | Status |
|------------|-------|-------------|--------|
| Test data seeding API (TB-01) | Backend Lead | Sprint 13 start | System-level |
| AI Gateway mock mode (TB-02) | Backend Lead | Sprint 13 (S12.07) | System-level |
| Stripe test mode (TB-03) | Backend Lead | Sprint 13 | System-level |
| `require_professional_plus_tier` real implementation | Backend Lead | Sprint 13 (S12.06) | ⚠️ stub — needs replacement |
| SendGrid sandbox configuration | DevOps | Sprint 13 (S12.10) | S12.10 scope |
| k6 cloud or self-hosted runner | DevOps | Sprint 14 (S12.17) | S12.17 scope |
| OWASP ZAP tooling | DevOps | Sprint 14 (S12.17) | S12.17 scope |

### Risks to Plan

- **Risk:** `require_professional_plus_tier` stub lands in production on S12.06 without replacement
  **Impact:** All users (including Professional+) receive 500 on competitor/pipeline endpoints — revenue-impacting outage
  **Contingency:** Code review gate: S12.06 PR must not be merged if `tier_gate.py` still raises `NotImplementedError`; add P1 test to CI that verifies stub is replaced

- **Risk:** Materialized views contain stale or incomplete data in staging → false test failures on dashboard APIs
  **Contingency:** Manual refresh command `scripts/refresh_analytics_views.py --all` in test setup + dedicated analytics seed data factory via TB-01

- **Risk:** Load test infrastructure (k6) not available by Sprint 14 → performance targets unverified at launch
  **Contingency:** Run k6 locally against staging with reduced concurrency (20 users vs 50); document as limitation with plan to retest post-launch

- **Risk:** SendGrid sandbox does not support attachment delivery in test mode
  **Contingency:** Verify email sent via SendGrid webhook event + Celery task status; assert attachment byte size > 0 from task result; skip email client rendering test

---

## Interworking & Regression

| Service/Component | E12 Changes | Regression Scope |
|-------------------|------------|-----------------|
| **Client API** | New analytics endpoints (market ✅, ROI, team, competitor, pipeline, usage); report generation; onboarding flow; `require_paid_tier` + `require_professional_plus_tier` tier gate dependencies in `core/tier_gate.py` | Existing auth, opportunity search, proposal endpoints must pass post-E12; tier gate dependency must not break existing paid-feature gates |
| **Admin API** | New tenant management, crawler management, white-label, audit log, platform analytics endpoints | Existing E02 admin auth tests and E11 compliance admin tests must still pass |
| **Notification Service** | Scheduled report delivery via Celery Beat + SendGrid; new analytics materialized view refresh tasks (daily/hourly) ✅ configured | Existing alert delivery, digest notifications must still pass; Beat schedule must not conflict with existing tasks |
| **Data Pipeline** | Crawler management endpoints expose run history and Celery Beat schedule control | Existing E04/E05 crawl pipeline tests must still pass |
| **AI Gateway** | Pipeline forecasting dashboard consumes prediction agent responses | Existing AI summary, proposal generation, compliance check tests must still pass |
| **Enterprise API** | New surface at `api.eusolicit.com/v1/*` with API key auth + rate limiting; `api-keys` CRUD in Client API | New surface; no regression from prior epics; verify not accessible without `X-API-Key` |
| **`analytics_views.py` ORM models** | 5 new read-only `Table` definitions in `client_api.models`; `info={"is_view": True}` flag prevents Alembic DDL conflicts ✅ | Existing Alembic `alembic check` CI gate must still pass (views excluded from DDL comparison) |

---

## Follow-on Workflows

### ATDD (P0 — Complete / P1–P3 — Pending)

- **P0 RED phase:** Complete — 19 Playwright API tests in 4 spec files (`atdd-checklist-e12-p0.md`, 2026-04-05)
- **tea_automation status:** `12-1` = in-progress (per sprint-status.yaml); `12-2`, `12-3` = not yet started
- **Next (P0 GREEN phase):** Remove `test.skip()` domain by domain as endpoints deploy:
  - Analytics (cross-tenant, tier gates): after S12.06 stub replaced + S12.04–S12.08 deployed
  - Admin (access control): after S12.11–S12.13 deployed
  - Enterprise API (auth + rate limit): after S12.15 deployed
- **P1/P2/P3 ATDD:** Run `*atdd` workflow after S12.04–S12.18 implementation; reference this test design as input

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

# Run analytics domain in headed mode (debugging)
npx playwright test --headed e2e/specs/analytics/cross-tenant-isolation.api.spec.ts

# Trigger ad-hoc materialized view refresh for test setup
python scripts/refresh_analytics_views.py --all
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

### v5 Changes Summary (from v4)

1. **+1 new risk (E12-R-010):** Celery Beat schedule drift risk (PERF, score 2) — daily refresh tasks staggered 2:00–2:45 AM; identified from S12.01 implementation schedule details
2. **E12-R-002 description updated:** Now explicitly identifies `require_professional_plus_tier` stub (`NotImplementedError`) as an implementation risk discovered in S12.02's `tier_gate.py`; mitigation plan expanded to include stub-replacement gate
3. **E12-R-006 updated:** Unique index column combinations now specified from S12.01 implementation (`uq_mv_market_intelligence(company_id, sector, country, month)`, etc.)
4. **+1 P1 test:** Stub-gate verification test added — confirms `require_professional_plus_tier` is a real implementation before S12.06 PR merges (total P1: 18 → 19)
5. **P1 market API test:** Updated with concrete details — exact response schema, `Cache-Control: public, max-age=1800`, 403 body `{"message": ..., "upgrade_required": true}` from S12.02 implementation
6. **P2 S12.03 frontend test:** data-testid attributes added from S12.03 story spec (`market-filters`, `market-filter-date-from`, etc.)
7. **ATDD status:** tea_status for `12-1` = in-progress; P0 GREEN phase trigger conditions updated per implementation progress
8. **Entry criteria:** Added `require_professional_plus_tier` stub replacement as a new entry gate
9. **Dependencies table:** Added `require_professional_plus_tier` as explicit dependency with ⚠️ status
10. **Interworking:** Added `analytics_views.py` ORM models row + `core/tier_gate.py` regression note; updated notification service row for materialized view refresh tasks
11. **Total atomic tests corrected:** 62 functional + 2 infra = **64** (from 61+2=63 in v4, due to +1 P1 test)

### Related Documents

| Document | Path |
|----------|------|
| Epic 12 | `eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md` |
| System-Level Test Design (Architecture) | `eusolicit-docs/test-artifacts/test-design-architecture.md` |
| System-Level Test Design (QA) | `eusolicit-docs/test-artifacts/test-design-qa.md` |
| E12 P0 ATDD Checklist | `eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md` |
| S12.01 Implementation | `eusolicit-docs/implementation-artifacts/12-1-analytics-materialized-views-refresh-infrastructure.md` |
| S12.02 Implementation | `eusolicit-docs/implementation-artifacts/12-2-market-intelligence-dashboard-api.md` |
| S12.03 Implementation | `eusolicit-docs/implementation-artifacts/12-3-market-intelligence-dashboard-frontend.md` |
| PRD | `eusolicit-docs/EU_Solicit_PRD_v1.md` |
| Architecture | `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md` |

---

**Generated by:** BMad TEA Agent — Test Architect Module
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
