---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-13'
inputDocuments: [
  '/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md',
  '/home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/test-design-architecture.md',
  '/home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/test-design-qa.md',
  '/home/debian/Projects/eusolicit/eusolicit-docs/test-artifacts/automation-summary-story-12-1.md',
  '/home/debian/Projects/eusolicit/_bmad/bmm/config.yaml'
]
---

# Test Design: Epic 12 — Analytics, Reporting & Admin Platform

**Date:** 2026-04-13
**Author:** Deb
**Status:** Draft (Updated — S12.01 automation complete)
**Epic:** E12 | **Sprint:** 13–14 | **Points:** 55 | **Dependencies:** E05, E06, E07, E08

---

## Executive Summary

**Scope:** Epic-level test design for Epic 12 (Analytics, Reporting & Admin Platform) — the final pre-launch epic covering 18 stories across analytics dashboards, PDF/DOCX report generation, internal admin platform, enterprise API, performance hardening, and user onboarding.

**Implementation Status:**

- **S12.01 (MV & Refresh Infrastructure):** ✅ Automated — 258 tests passing (89 notification unit, 139 client-API unit, 30 integration); 9 E2E tests in RED phase (pending S12.02–S12.08 endpoints)
- **S12.02–S12.18:** Pending implementation

**Risk Summary:**

- Total risks identified: 12
- High-priority risks (≥6): 7 (R12.1 and R12.5 partially mitigated by S12.01 automation)
- Medium-priority risks (3–5): 3
- Low-priority risks (1–2): 2
- Critical categories: SEC, PERF, DATA, OPS

**Coverage Summary:**

- P0 scenarios: ~43 test cases (~65–80 hours)
- P1 scenarios: ~149 test cases (~88–105 hours)
- P2 scenarios: ~25 test cases (~20–28 hours)
- P3 scenarios: ~10 test cases (~4–6 hours)
- **Total:** ~227 test cases · **Effort:** ~175–220 hours (~22–28 days for 1 QA / ~11–14 days for 2 QAs)
- **Already automated (S12.01):** 258 tests (unit + integration); ~28 P0 + 139 P1 + 91 P2 tests complete

---

## Not in Scope

| Item | Reasoning | Mitigation |
|:---|:---|:---|
| **External SIEM Integration** | Out of scope for MVP; admin platform uses internal Loki/PostgreSQL audit logs. | Audit log content verified via internal API assertions only. |
| **Mobile Native App Views** | Admin platform is desktop-first/responsive web; no native mobile app in scope. | Responsive web testing covers tablet + mobile viewports in browser. |
| **Real-time Cross-tenant Benchmarking** | Global aggregate benchmarking across all tenants is Phase 2 post-MVP. | Analytics scoped to per-tenant data isolation only. |
| **Historical MV Backfill Accuracy** | Accuracy of pre-existing data before MV deployment is Phase 2. | MV correctness tested with seeded data from migration point forward. |
| **Email Deliverability (SendGrid Spam)** | Email rendering and deliverability optimisation is outside test scope. | SendGrid API mock used; delivery confirmed via API response, not inbox. |

---

## Risk Assessment

### High-Priority Risks (Score ≥ 6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|:---|:---|:---|:---:|:---:|:---:|:---|:---|:---|
| **R12.1** | SEC | **Cross-tenant data leakage** in analytics materialized views — queries lacking `company_id` scoping expose other tenants' procurement/ROI data. | 2 | 3 | **6** | Automated multi-tenant isolation suite injecting wrong `company_id` context into every analytics endpoint; assert empty/401 response. Verify SQL filter in each MV query. | Backend/Security Lead | Sprint 13 |
| **R12.2** | SEC | **Admin API privilege escalation** — non-admin JWT or non-VPN request reaches admin endpoints; tier-override or tenant data exposed. | 2 | 3 | **6** | Negative RBAC test suite iterating all admin routes with standard user JWT and non-allowlisted IPs; expect 403 on every attempt. | Security Lead | Sprint 13 |
| **R12.3** | PERF | **Analytics query latency** — materialized view queries against large multi-tenant datasets exceed dashboard load SLA (>3s p95). | 2 | 3 | **6** | Performance benchmark suite with 500k–1M row seeded datasets; assert p95 < 3s for all `/analytics/*` endpoints; EXPLAIN ANALYZE on slow queries. | Perf/Backend Lead | Sprint 14 |
| **R12.4** | SEC | **Enterprise API key exposure** — hashed key storage bypassed, revoked keys still accepted, or rate limiting misconfigured allowing quota exhaustion. | 2 | 3 | **6** | Key lifecycle tests (create → use → revoke → retry must fail); rate limit saturation test returning 429 with correct headers; key hash integrity assertion. | Backend/Security Lead | Sprint 14 |
| **R12.5** | DATA | **Materialized view stale/corrupt data** — `REFRESH MATERIALIZED VIEW CONCURRENTLY` fails mid-run leaving partial view or blocking reads if unique index missing. | 2 | 3 | **6** | Integration test triggering concurrent MV refresh while issuing read queries; assert no read timeouts and view row counts consistent pre/post refresh. Verify `CONCURRENTLY` constraint (unique index present). | Backend Lead | Sprint 13 |
| **R12.9** | PERF | **Load test failure at target concurrency** — key user flows (search, analytics dashboards, proposal generation) exceed p95 500ms threshold under production load. | 2 | 3 | **6** | Locust/k6 load test scripts covering all target flows against staging; document p50/p95/p99 latencies; gate on p95 < 500ms for read endpoints. | Perf Team | Sprint 14 |
| **R12.10** | SEC | **Security audit failures (OWASP/network policy)** — launch blocked by unresolved OWASP Top 10 findings, misconfigured Kubernetes network policies, or unrotated production secrets. | 2 | 3 | **6** | Complete OWASP Top 10 checklist execution; Kubernetes network policy service-isolation verification; confirm all secrets rotated and stored in secret manager before release gate. | Security Team | Sprint 14 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|:---|:---|:---|:---:|:---:|:---:|:---|:---|
| **R12.6** | BUS | **Tier gate bypass** — users on Starter/Professional access Professional+ analytics dashboards (competitor, pipeline) or Enterprise API via URL manipulation or missing middleware. | 2 | 2 | **4** | Negative tier tests for all gated endpoints using tokens from each tier; assert 403 + upgrade message for non-qualifying tiers. | Product/Backend |
| **R12.7** | OPS | **Silent async report failure** — Celery PDF/DOCX generation task fails without surfacing to the user; download link never appears; DLQ unmonitored. | 2 | 2 | **4** | Dead Letter Queue validation: inject a report task that triggers Celery failure; assert DLQ message created and job status reflects failure. Verify user notification path. | Ops/Backend |
| **R12.8** | TECH | **Celery Beat schedule drift/miss** — daily MV refresh tasks skipped due to scheduler misconfiguration; hourly usage refresh lags >2 hours. | 2 | 2 | **4** | Integration test verifying Celery Beat schedule config is registered and fires within expected window using test task; monitor scheduler beat log. | Backend Lead |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|:---|:---|:---:|:---:|:---:|:---:|:---|
| **R12.11** | BUS | Onboarding wizard state corruption — `onboarding_completed` flag reset after wizard skip/dismiss causing re-trigger on next login. | 1 | 2 | 2 | Monitor; cover in P1 E2E regression. |
| **R12.12** | BUS | White-label subdomain collision — duplicate subdomain accepted, causing routing confusion between tenants. | 1 | 2 | 2 | Monitor; uniqueness validation test in P1. |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring, alerting)

---

## Entry Criteria

- [ ] Epic 12 requirements finalized and signed off by PM and Tech Lead.
- [ ] Staging environment running with PostgreSQL (MV support + unique index constraints), Redis, Celery Beat, and S3-compatible storage (LocalStack or AWS staging).
- [ ] Test data factories available: `TenantDataFactory` (10k–1M records), `AdminUserFactory` (admin role JWT), `ApiKeyFactory` (valid/revoked keys), `CompanyProfileFactory`.
- [ ] Playwright `apiRequest` and `recurse` helpers configured in the monorepo.
- [ ] Admin Service, Client Analytics API, and Enterprise API services scaffolded and accessible in staging.
- [ ] VPN/IP allowlist configured for admin endpoint testing (or IP allowlist mock in staging config).
- [ ] System-level test design reviewed and blockers B-01, B-02, B-03 addressed.

## Exit Criteria

- [ ] 100% of P0 tests pass — zero exceptions (tenant isolation, admin RBAC, enterprise API key gate).
- [ ] ≥95% of P1 tests pass (failures triaged and waived with owner sign-off).
- [ ] Analytics dashboard p95 response time < 3s under standard staging load.
- [ ] Load test p95 < 500ms for read endpoints at target concurrency — documented results.
- [ ] OWASP Top 10 checklist fully completed; all High findings resolved or risk-accepted by Security Lead.
- [ ] No open Critical or High severity defects unresolved.
- [ ] Enterprise API documentation (Swagger/Redoc) verified against implementation.
- [ ] All production secrets rotated and confirmed in secret manager.

---

## Test Coverage Plan

> **Priority labels (P0/P1/P2/P3) indicate risk level and test importance, NOT execution timing.** When to run each suite is defined separately in the Execution Strategy section below. A P2 test does not mean "run nightly" — it means lower risk, secondary coverage.

> **Test level notation:** "API" = Playwright `apiRequest`/`request` (no browser); "E2E" = Playwright browser + UI; "Integration" = service + DB/queue interaction; "Unit" = isolated logic/function. Per `test-levels-framework.md`, business logic is tested at Unit/API level; user journeys at E2E.

---

### P0 (Critical)

**Criteria:** Blocks core journey + High risk (≥6) + No workaround

| Story | Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---|:---:|:---|:---|
| S12.01 | MV schema exists for all 5 domains post-migration; indexes on filter columns present | API/DB | R12.5 | 2 | QA/DEV | ✅ **AUTOMATED** — `test_011_migration.py::TestE12DB002ViewsExist`, `TestE12DB003UniqueIndexes` (30 integration tests passing) |
| S12.01 | `REFRESH MATERIALIZED VIEW CONCURRENTLY` does not block concurrent reads | Integration | R12.5 | 2 | DEV | ✅ **AUTOMATED** — `test_011_migration.py::TestE12DB006RefreshConcurrently`, `TestE12DB007ConcurrentRead` |
| S12.02 | Company A token cannot retrieve Company B's market analytics data | API | R12.1 | 3 | QA | Inject wrong `company_id`; assert empty results or 404 — never Company B data |
| S12.04 | Company A token cannot retrieve Company B's ROI data | API | R12.1 | 2 | QA | Same pattern — cross-tenant ROI isolation |
| S12.05 | Standard user token cannot retrieve other users' team metrics | API | R12.1 | 3 | QA | `GET /analytics/team/user/{other_user_id}` with standard JWT → 403 |
| S12.06 | Starter/Professional tier token receives 403 + upgrade message on competitor endpoints | API | R12.6 | 3 | QA | All 3 competitor endpoints tested with each lower tier |
| S12.07 | Starter/Professional tier token receives 403 + upgrade message on pipeline forecast | API | R12.6 | 2 | QA | Forecast endpoint + tier gate message validated |
| S12.09 | Async report Celery task completes; S3 object exists; signed URL returned | Integration | R12.7 | 3 | QA/DEV | Use `recurse` to poll job status; assert URL resolves and returns valid file |
| S12.11 | All admin tenant endpoints return 403 for non-allowlisted IP | API | R12.2 | 3 | QA | Test from non-VPN IP; assert 403 on all tenant routes |
| S12.11 | All admin tenant endpoints return 401/403 for standard user JWT (no admin claim) | API | R12.2 | 4 | QA | Iterate all tenant admin endpoints with standard token |
| S12.12 | All admin crawler/white-label endpoints return 403 for non-allowlisted IP | API | R12.2 | 3 | QA | Same IP-restriction pattern for crawler routes |
| S12.13 | All admin audit/analytics endpoints return 403 for non-allowlisted IP | API | R12.2 | 2 | QA | IP restriction on audit log + platform analytics |
| S12.14 | Admin frontend pages blocked for non-admin role (route guard redirects) | E2E | R12.2 | 3 | QA | Attempt direct URL nav as standard user; assert redirect to /403 or /dashboard |
| S12.15 | `X-API-Key` missing or invalid returns 401 on all `/v1/*` routes | API | R12.4 | 3 | QA | Missing header, tampered key, unknown key — all must return 401 |
| S12.15 | Revoked API key returns 401 (not 200) | API | R12.4 | 1 | QA | Create key → revoke → retry → assert 401 |
| S12.15 | Rate limit exceeded returns 429 with `X-RateLimit-*` headers | API | R12.4 | 3 | QA | Burst requests beyond per-key limit; assert 429 + correct headers |
| S12.17 | Load test: p95 read latency < 500ms under target concurrency on staging | Performance | R12.9 | 1 | Perf | k6/Locust suite across search, analytics, proposal flows |
| S12.17 | OWASP Top 10 checklist items all addressed or risk-accepted by Security Lead | Security Audit | R12.10 | 1 | Security | Checklist evidence attached to release gate |

**Total P0:** ~43 test cases across 18 scenarios · Estimated effort: **~65–80 hours**

---

### P1 (High)

**Criteria:** Core features + Medium/High risk (3–5) + Common user workflows

#### S12.01 — Analytics Materialized Views & Refresh Infrastructure ✅ AUTOMATED (258 tests)

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| Celery Beat daily refresh fires for market/ROI/team/competitor views (verified via task execution log) | Integration | R12.8 | 2 | DEV | ✅ `test_refresh_analytics_views.py::TestBeatSchedule` + `test_refresh_analytics_views_extended.py` |
| Hourly usage view refresh schedule is registered and fires within window | Integration | R12.8 | 1 | DEV | ✅ `test_refresh_analytics_views.py::TestBeatSchedule` — hourly config verified |
| Manual refresh management command updates MV data correctly | API/CLI | — | 1 | DEV | ✅ `test_refresh_script.py` — 34 tests covering all CLI paths |
| Migration rollback leaves no orphaned views or indexes | DB | R12.5 | 1 | DEV | ✅ `test_011_migration.py::TestE12DB008Downgrade` — rollback verified |
| MV indexes on filter columns (`sector`, `country`, `date_range`) improve query plans | DB | R12.3 | 1 | DEV | ✅ `test_011_migration.py::TestE12DB003UniqueIndexes` — index presence confirmed |

**S12.01 P1 subtotal:** 6 tests (all automated)

#### S12.02 — Market Intelligence Dashboard API

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /analytics/market/volume` returns procurement volume grouped by sector with date range + country filter | API | R12.3 | 2 | QA | Test filter combinations; verify pagination |
| `GET /analytics/market/values` returns average contract values by sector | API | — | 1 | QA | Verify calculation against seeded data |
| `GET /analytics/market/authorities` returns top authorities ranked by activity (paginated) | API | — | 2 | QA | Ranking order + pagination metadata |
| `GET /analytics/market/trends` returns monthly aggregates with correct date grouping | API | — | 2 | QA | Verify monthly bucket boundaries |
| Empty-result edge cases return empty arrays (not 500) for all 4 endpoints | API | — | 1 | QA | Seed company with no bids; assert `[]` with 200 |
| Cache headers (`Cache-Control`, `ETag`) present on market responses | API | — | 1 | QA | Verify response headers per spec |

**S12.02 P1 subtotal:** 9 tests

#### S12.03 — Market Intelligence Dashboard Frontend

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| Bar chart renders procurement volume grouped by sector (Recharts SVG visible) | E2E | — | 1 | QA | Assert `<svg>` Recharts element rendered with bar data |
| Line chart renders monthly trend data with hover tooltip functional | E2E | — | 1 | QA | Hover tooltip appears on data point |
| Sortable table displays top contracting authorities with pagination | E2E | — | 2 | QA | Sort by activity column; pagination navigates |
| Date range, sector, and country filters update all 3 visualizations | E2E | — | 2 | QA | Change filter → assert API re-called and charts re-render |
| Responsive layout: charts stack vertically on mobile viewport (375px) | E2E | — | 1 | QA | Screenshot assertion at 375px width |

**S12.03 P1 subtotal:** 7 tests

#### S12.04 — ROI Tracker Dashboard (Full Stack)

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /analytics/roi/summary` returns correct total invested, won, ROI % from seeded data | API | — | 2 | QA | Verify ROI formula: `(won - invested) / invested × 100` |
| `GET /analytics/roi/bids` returns per-bid breakdown paginated, sortable | API | — | 2 | QA | Sort by investment desc; verify page_size |
| `GET /analytics/roi/trends` returns ROI over time grouped by month | API | — | 1 | QA | Verify time series grouping |
| Frontend summary cards display aggregate metrics correctly | E2E | — | 1 | QA | Assert card values match API response |
| Per-bid table sortable by investment, outcome, and ROI columns | E2E | — | 1 | QA | Click each sortable column header |
| Trend chart renders with date range filter applied | E2E | — | 1 | QA | Apply 3-month filter; assert chart re-renders |

**S12.04 P1 subtotal:** 8 tests

#### S12.05 — Team Performance Dashboard (Full Stack)

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /analytics/team/leaderboard` returns all team members sorted by win rate for company admin | API | — | 2 | QA | Admin JWT; verify sort order |
| `GET /analytics/team/user/{user_id}` returns individual user metrics | API | — | 1 | QA | Own user_id returns data; verified fields |
| Frontend leaderboard table sortable by any metric column | E2E | — | 2 | QA | Sort by bids submitted, win rate, avg prep time |
| User cards display all 4 metrics: bids submitted, win rate, avg prep time, proposals generated | E2E | — | 1 | QA | Assert all metric labels and values visible |
| Activity chart shows team-level bids submitted over time | E2E | — | 1 | QA | Assert chart renders with correct data |

**S12.05 P1 subtotal:** 7 tests

#### S12.06 — Competitor Intelligence Dashboard (Professional+ Tier)

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /analytics/competitors/profiles` returns paginated competitor profiles for Professional+ | API | R12.6 | 2 | QA | Valid Professional+ token; verify fields |
| `GET /analytics/competitors/{id}/patterns` returns bidding pattern data | API | — | 1 | QA | Valid competitor_id; verify pattern fields |
| `GET /analytics/competitors/benchmarks` returns pricing benchmark aggregations | API | — | 1 | QA | Verify benchmark structure |
| Frontend competitor profile cards show all required fields | E2E | — | 1 | QA | Name, bid count, win rate, active sectors visible |
| Comparison table: select 2–4 competitors side by side | E2E | — | 2 | QA | Select 2 and 4 competitors; table renders |
| Pattern chart renders bidding frequency trends | E2E | — | 1 | QA | Assert chart element present with data |

**S12.06 P1 subtotal:** 8 tests

#### S12.07 — Pipeline Forecasting Dashboard (Professional+ Tier)

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /analytics/pipeline/forecast` returns predicted opportunities with confidence scores | API | — | 2 | QA | Verify all 5 fields: title, sector, value range, date, confidence |
| Frontend timeline view plots predicted opportunities on time axis | E2E | — | 1 | QA | Assert timeline/calendar items render |
| Confidence color coding: green (high), amber (medium), red (low) indicators | E2E | — | 1 | QA | Assert CSS color classes per confidence level |
| Sector and confidence threshold filters update forecast results | E2E | — | 2 | QA | Apply each filter; assert list updates |

**S12.07 P1 subtotal:** 6 tests

#### S12.08 — Usage Dashboard (Full Stack)

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /analytics/usage` returns consumed/limit/remaining for all 3 usage types | API | — | 2 | QA | Verify AI summaries, proposal drafts, compliance checks |
| Response includes billing period start/end dates | API | — | 1 | QA | Assert date fields present and valid |
| Frontend circular progress meters render for each usage type | E2E | — | 1 | QA | Assert 3 meter elements with correct labels |
| Warning indicator shown when usage exceeds 80% of limit | E2E | — | 1 | QA | Seed usage at 85% limit; assert warning UI element |
| Upgrade CTA displayed when any usage type is at or above limit | E2E | — | 1 | QA | Seed usage at 100% limit; assert CTA link to billing |

**S12.08 P1 subtotal:** 6 tests

#### S12.09 — Report Generation Engine (PDF & DOCX)

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| PDF report generated via reportlab: has headers, summary section, and data table | Integration | R12.7 | 2 | DEV | Assert PDF bytes > 0; parse structure via reportlab reader |
| DOCX report generated via python-docx: has headers, summary, and embedded chart image | Integration | R12.7 | 2 | DEV | Assert DOCX bytes > 0; verify document structure |
| S3 upload stores report object; signed URL generated with 24h expiry | Integration | — | 2 | DEV | Assert S3 object exists; URL TTL verified |
| All 4 templates produce valid output: pipeline summary, bid performance, team activity, custom date range | Integration | — | 4 | DEV | One test per template; assert non-empty output |
| Shared generation code between E07 proposal export and E12 report engine (no duplication) | Unit | — | 1 | DEV | Assert common renderer module imported in both paths |

**S12.09 P1 subtotal:** 11 tests

#### S12.10 — Scheduled & On-Demand Report Delivery

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| Company admin can configure weekly/monthly schedule with type, format, and recipients via API | API | — | 2 | QA | POST schedule config; GET confirms it persisted |
| Celery Beat triggers scheduled report and sends email via SendGrid (mock) | Integration | R12.7/R12.8 | 2 | DEV | Mock SendGrid; assert task fired and email payload correct |
| On-demand report request returns async job ID; status polling via `recurse` reaches "complete" | E2E/API | — | 2 | QA | Playwright `recurse` polling job endpoint until status=complete |
| Download link appears in-app when report is ready | E2E | — | 1 | QA | Assert download link element visible after job completes |
| Reports list page shows past reports with download links and timestamps | E2E | — | 1 | QA | Assert table row with correct metadata |
| Email delivery includes report as attachment with summary in body | Integration | — | 1 | DEV | Mock SendGrid; assert attachment MIME type correct |

**S12.10 P1 subtotal:** 9 tests

#### S12.11 — Admin API — Tenant Management

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /admin/tenants` returns paginated list with search by name and tier filter | API | — | 3 | QA | Search partial name; filter by tier; assert pagination metadata |
| `GET /admin/tenants/{company_id}` returns subscription, current usage, activity summary | API | — | 2 | QA | Verify all summary fields present |
| `POST /admin/tenants/{company_id}/tier-override` updates tier and creates audit log entry | API | — | 2 | QA | Assert tier changed; assert audit_log row created with reason |
| Admin JWT with admin role claim accepted on all tenant routes | API | — | 1 | QA | Positive: valid admin JWT → 200 on all routes |

**S12.11 P1 subtotal:** 8 tests

#### S12.12 — Admin API — Crawler & White-Label Management

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /admin/crawlers/runs` returns paginated run history with status, timestamps | API | — | 2 | QA | Verify pagination + status field values |
| `GET/PUT /admin/crawlers/schedule/{crawler_type}` reads and updates Celery Beat schedule | API | — | 2 | QA | Read current schedule; update; re-read confirms change |
| `POST /admin/crawlers/trigger/{crawler_type}` starts crawl and returns valid run_id | API | — | 1 | QA | Assert run_id UUID format; GET run status shows "running" |
| `GET/PUT /admin/tenants/{id}/white-label` reads/updates branding settings | API | R12.12 | 2 | QA | Update subdomain, logo, colors; GET confirms persisted |
| Subdomain uniqueness: duplicate subdomain rejected with 409 | API | R12.12 | 1 | QA | Create two tenants with same subdomain; assert 409 on second |
| `GET /admin/crawlers/runs/{run_id}` returns run status, result counts, error details | API | — | 1 | QA | Assert all detail fields present |

**S12.12 P1 subtotal:** 9 tests

#### S12.13 — Admin API — Audit Log & Platform Analytics

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `GET /admin/audit-logs` returns paginated filtered results for all 5 filter dimensions | API | — | 3 | QA | Filter by user_id, action_type, entity_type, date range individually |
| `GET /admin/audit-logs/export` returns streaming CSV response with correct MIME type | API | — | 2 | QA | Assert `Content-Type: text/csv`; parse 5 header columns |
| `GET /admin/analytics/funnel` returns signup funnel stage counts (registered, trial, paid) | API | — | 1 | QA | Assert all 3 funnel stages present with integer counts |
| `GET /admin/analytics/revenue` returns MRR, growth rate, churn rate | API | — | 2 | QA | Verify revenue metric fields and numeric types |
| `GET /admin/analytics/tiers` returns tier distribution counts | API | — | 1 | QA | Assert tier distribution sums to total tenant count |

**S12.13 P1 subtotal:** 9 tests

#### S12.14 — Admin Frontend Pages

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| Tenant management table: searchable, filterable by tier, detail drawer opens with correct data | E2E | — | 3 | QA | Search by partial name; open drawer; assert subscription info |
| Tier override form: submits with reason field; tier badge updates immediately in table | E2E | — | 2 | QA | Assert optimistic update and confirmation toast |
| Crawler run history: status badges displayed; manual trigger button fires with confirmation | E2E | — | 2 | QA | Click trigger; confirm dialog; assert new run row appears |
| Audit log page: filter bar, date range picker, paginated results, CSV export downloads file | E2E | — | 3 | QA | Apply date filter; click export; assert file download initiated |
| Platform analytics page: funnel chart, tier pie chart, revenue line chart, metric cards all render | E2E | — | 2 | QA | Assert 4 chart/card elements visible with non-zero data |

**S12.14 P1 subtotal:** 12 tests

#### S12.15 — Enterprise API — Key Auth, Rate Limiting & Docs

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| `api.eusolicit.com/v1/*` routes proxy correctly to Client API endpoints | API | — | 2 | QA | Assert `/v1/analytics/market/volume` resolves same data as client API |
| `POST /v1/api-keys` creates new API key (hashed in DB); `GET` lists it; `DELETE` revokes it | API | R12.4 | 3 | QA | Full CRUD lifecycle; assert DB hash differs from plain key |
| Valid API key header authenticates successfully on all `/v1/*` routes | API | R12.4 | 1 | QA | Positive path: valid key → 200 |
| Swagger UI accessible at `/v1/docs`; Redoc at `/v1/redoc` | API | — | 2 | QA | Assert 200 with HTML body containing spec UI |

**S12.15 P1 subtotal:** 8 tests

#### S12.16 — Enterprise API Documentation & Usage Frontend

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| API key management page: lists masked active keys (last 4 chars visible only) | E2E | — | 1 | QA | Assert key display masked with `****xxxx` format |
| New key creation: full key shown exactly once with copy-to-clipboard; subsequent view masked | E2E | R12.4 | 2 | QA | Create key; capture display; navigate away; return; verify masked |
| Key revocation: confirmation dialog; key status changes to revoked immediately | E2E | — | 1 | QA | Revoke; retry API call; assert 401 |
| Rate limit tier and usage stats displayed on management page | E2E | — | 1 | QA | Assert rate limit info section visible |

**S12.16 P1 subtotal:** 5 tests

#### S12.17 — Performance Optimization, Load Testing & Security Audit

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| EXPLAIN ANALYZE on all analytics queries shows no sequential scans on large datasets | DB | R12.3 | 2 | DEV | Assert index scan in query plan for all `/analytics/*` endpoints |
| N+1 queries resolved in API endpoints (verified via query count assertion) | Unit/Integration | R12.3 | 2 | DEV | Assert max N DB queries per API request using query counter |
| CORS configuration: allowed origins restricted; preflight returns correct headers | API | R12.10 | 1 | QA | Assert CORS headers on preflight; non-whitelisted origin rejected |
| All public endpoints have rate limiting configured (non-Enterprise API) | API | R12.4/R12.10 | 1 | QA | Assert 429 on saturation of public API endpoints |
| All production secrets documented as rotated in secret manager | Security Audit | R12.10 | 1 | Security | Evidence checklist: DB passwords, JWT keys, API keys rotated |
| Kubernetes network policies verify service-to-service isolation (cross-namespace blocked) | Security/Infra | R12.10 | 1 | Security | Test network policy rules in staging cluster |

**S12.17 P1 subtotal:** 8 tests

#### S12.18 — User Onboarding Flow & Launch Polish

| Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| First login triggers onboarding wizard overlay for users with `onboarding_completed=false` | E2E | R12.11 | 1 | QA | New user login; assert wizard overlay visible |
| Wizard steps complete in order: profile → search → opportunity → feature tour | E2E | — | 3 | QA | Step through each; assert spotlight and tooltip per step |
| Skip/dismiss sets `onboarding_completed=true` immediately via API | E2E/API | R12.11 | 2 | QA | Skip wizard; assert flag set via GET user API |
| Wizard does not reappear on subsequent login after completion or dismissal | E2E | R12.11 | 1 | QA | Login again; assert no overlay visible |
| Error pages (404, 500, 403) use branded templates with correct status codes | E2E | — | 2 | QA | Navigate to non-existent route; trigger 403; verify branded design |

**S12.18 P1 subtotal:** 9 tests

---

**Total P1: ~149 test cases across 62 scenario groups · Estimated effort: ~88–105 hours**

---

### P2 (Medium)

**Criteria:** Secondary flows + Low risk + Edge cases + UI polish

| Story | Scenario | Test Level | Risk Link | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| S12.01 | MV data remains consistent after simulated partial refresh failure (rollback to last full state) | Integration | R12.5 | 2 | DEV | Simulate mid-refresh failure; assert last valid state preserved |
| S12.02 | Market dashboard response includes correct cache headers per CDN caching spec | API | — | 1 | QA | Verify `Cache-Control: max-age=300` on market routes |
| S12.03 | Loading skeletons displayed during API fetch on market dashboard | E2E | — | 1 | QA | Throttle network; assert skeleton animation visible |
| S12.04 | ROI per-bid table: columns sortable in ascending and descending order | E2E | — | 1 | QA | Click investment column twice; assert order reversal |
| S12.05 | Team activity bar chart renders with correct time axis grouping | E2E | — | 1 | QA | Assert bar chart with date-bucketed data |
| S12.06 | Competitor pattern chart renders bidding frequency over time | E2E | — | 1 | QA | Assert chart SVG with time axis visible |
| S12.07 | Empty state displayed when no pipeline predictions available | E2E | — | 1 | QA | Seed company with no forecast data; assert empty state UI |
| S12.07 | Confidence threshold filter removes low-confidence items from timeline | E2E | — | 1 | QA | Apply high-confidence filter; assert low items hidden |
| S12.08 | Usage dashboard accessible to all paid tiers (Starter, Professional, Enterprise) | API | — | 1 | QA | Test with each tier JWT; assert 200 |
| S12.09 | All 4 report templates produce non-empty output (one test per template type) | Integration | — | 2 | DEV | Verify pipeline summary, bid perf, team activity, custom |
| S12.10 | Signed S3 download URL expires after 24 hours | Integration | — | 1 | DEV | Assert URL signed with 86400s TTL in metadata |
| S12.11 | Pagination metadata correct: `total_count`, `page`, `page_size` in all admin list responses | API | — | 1 | QA | Verify pagination envelope on `/admin/tenants` |
| S12.12 | DNS readiness check returns informative error when subdomain CNAME unresolvable | API | — | 1 | QA | Mock DNS failure; assert error message includes domain name |
| S12.13 | `GET /admin/analytics/usage` returns aggregate usage metrics across all tenants | API | — | 1 | QA | Assert response contains multi-tenant aggregate |
| S12.14 | White-label live preview panel updates in real time as logo/color settings change | E2E | — | 2 | QA | Change logo URL; change primary color; assert preview updates |
| S12.14 | Platform analytics funnel chart and revenue line chart render with correct data shape | E2E | — | 1 | QA | Assert chart components visible with non-empty series |
| S12.15 | OpenAPI spec includes request/response examples for all endpoints | API | — | 1 | QA | Parse OpenAPI JSON; assert examples present on each route |
| S12.16 | Enterprise API documentation page embeds Swagger UI or Redoc component | E2E | — | 1 | QA | Assert iframe or embedded component visible in API docs page |
| S12.17 | Response compression enabled (gzip/br): `Content-Encoding` header present on large responses | API | — | 1 | DEV | Assert compressed response on analytics endpoints >1KB |
| S12.18 | Empty states reviewed across all 5 main pages: no broken/missing empty state UI | E2E | — | 2 | QA | Navigate to each section with no data; assert styled empty state |
| S12.18 | Email templates render correctly (verified via HTML structure assertions in integration test) | Integration | — | 1 | QA | Assert key email sections present in rendered HTML |

**Total P2:** ~25 test cases · Estimated effort: **~20–28 hours**

---

### P3 (Low)

**Criteria:** Nice-to-have + Exploratory + Non-blocking polish

| Story | Scenario | Test Level | Count | Owner | Notes |
|:---|:---|:---|:---:|:---|:---|
| S12.03 | Recharts charts render without visual regression in Firefox and Safari | E2E/Visual | 2 | QA | Browser matrix run (chromium covers CI; cross-browser on schedule) |
| S12.01 | MV data freshness monitoring: verify dashboard shows "last updated" timestamp within 1hr for usage MV | E2E | 1 | QA | Assert timestamp label on usage dashboard |
| S12.14 | White-label live preview: visual accuracy of applied branding in preview vs live render | E2E/Visual | 1 | QA | Screenshot comparison of preview vs actual branded page |
| S12.17 | Chaos: analytics service restart during active MV refresh — assert graceful degradation | Chaos | 2 | Ops | Manual chaos injection; assert system recovers without data loss |
| S12.18 | Public pages (landing, pricing) include correct meta tags and OG image tags | E2E | 1 | QA | Assert `<meta og:*>` tags present and populated on public routes |
| S12.10 | Analytics data freshness: usage MV lag < 2h during peak load (monitor metric) | E2E/Monitor | 1 | Ops | Assert `last_refreshed` within threshold during load test |
| Cross-epic | Full regression of E05/E06/E07/E08 integration points after Epic 12 deployment | E2E | 2 | QA | Smoke test: tender search, proposal generation, billing still operational |

**Total P3:** ~10 test cases · Estimated effort: **~4–6 hours**

---

## Execution Strategy

> **Philosophy**: Run everything in PRs if the total suite completes in under 15 minutes. With Playwright parallelization, 190+ API and E2E tests complete in ~10–12 minutes. Defer only expensive, long-running, or infrastructure-heavy suites to nightly/weekly.

### Every PR

Run full Playwright functional test suite (P0 + P1):
- All API tests: tenant isolation, admin RBAC, enterprise key/rate-limit, tier gates, analytics endpoints, admin CRUD, onboarding
- All E2E browser tests: admin frontend pages, analytics dashboards, wizard flow, API docs page
- Duration: ~10–15 min with parallelization

### Nightly

Suites too slow or noisy for PR gates:
- P2 edge case + visual/responsive suite (~20 tests, ~15–20 min)
- Performance benchmark suite: EXPLAIN ANALYZE, N+1 detection, p95 latency against seeded datasets (~20 min)

### Weekly / Pre-release

Infrastructure-heavy or manual suites:
- k6/Locust full load test against staging (50–200 VUs, ~45 min)
- OWASP Top 10 security audit execution (manual + automated, ~4h)
- Kubernetes network policy verification (manual, ~1h)
- Chaos engineering suite: service restart, MV refresh failure (~30 min, ops-assisted)
- Cross-browser matrix: Firefox + Safari chart rendering (P3, ~15 min)

---

## Resource Estimates

### Test Development Effort

| Priority | Scenarios (est.) | Test Cases (est.) | Total Hours (range) | Notes |
|:---|:---:|:---:|:---:|:---|
| P0 | 18 | ~43 | ~60–85h | Security/isolation setup; IP mocking; rate limit harness (~20h one-time setup) |
| P1 | 62 | ~149 | ~85–110h | Standard API + E2E; async polling via `recurse` |
| P2 | 21 | ~25 | ~18–28h | Edge cases + visual polish |
| P3 | 7 | ~10 | ~4–6h | Exploratory + chaos (manual) |
| **Total** | **~108** | **~227** | **~170–230h** | **~22–29 days (1 QA) / ~11–15 days (2 QAs)** |

> Ranges are intentionally wide to account for async complexity (Celery, S3, PDF/DOCX assertions), security harness setup, and fixture creation uncertainty.

### Prerequisites

**Test Data Factories:**
- `TenantDataFactory`: Generates 10k–1M MV records per tenant for performance testing.
- `AdminUserFactory`: Creates users with admin JWT claims; generates non-admin tokens for negative tests.
- `ApiKeyFactory`: Generates valid, revoked, and expired enterprise API keys.
- `ScheduledReportFactory`: Creates company report schedules (weekly/monthly) for delivery testing.
- `CrawlerRunFactory`: Seeds crawler run history with mixed statuses.

**Tooling:**
- Playwright `apiRequest` — All P0/P1 backend and API assertions.
- Playwright `recurse` — Async job polling for report generation (S12.09/S12.10) and crawler run status (S12.12).
- Redis CLI / RedisInsight — Verify per-key rate-limiting token bucket state.
- k6 or Locust — Load test scripts covering 4 key user flows.
- reportlab / python-docx assertion utilities — PDF/DOCX structure validation.
- boto3 / LocalStack — S3 signed URL verification.

**Environment:**
- Staging: PostgreSQL with materialized view support + unique index support (`CONCURRENTLY`).
- Redis configured with per-key TTL for rate limiting.
- Celery + Celery Beat running with 1-minute test schedules (configurable via env).
- S3-compatible storage (LocalStack in staging, AWS S3 in pre-prod).
- VPN/IP allowlist mock: staging config accepts a test IP header for admin endpoint testing.
- SendGrid mock (webhook receiver or mock server) for email delivery assertions.

---

## Quality Gate Criteria

### Pass/Fail Thresholds

| Suite | Pass Rate | Policy |
|:---|:---|:---|
| **P0 (Security + Isolation)** | **100%** | Zero exceptions; any failure blocks release |
| **P0 (Load Test)** | **100%** | p95 < 500ms for read endpoints; documented results required |
| **P1 (Features)** | **≥95%** | Failures require owner waiver with root cause |
| **P2/P3** | **≥90%** | Informational; waivers self-serve |
| **OWASP Checklist** | **100% addressed** | All High findings resolved or risk-accepted in writing |

### Non-Negotiable Requirements

- [ ] **Zero cross-tenant data leakage** — any leak = release blocked (R12.1)
- [ ] **Zero admin RBAC bypass** — privilege escalation = release blocked (R12.2)
- [ ] **Enterprise API key lifecycle enforced** — revoked keys must fail 100% (R12.4)
- [ ] **Analytics p95 < 3s** on staging under 50 concurrent users (R12.3)
- [ ] **Load test p95 < 500ms** for read endpoints (R12.9)
- [ ] **All production secrets rotated** before release (R12.10)
- [ ] **OWASP Top 10** checklist evidence attached to release gate ticket (R12.10)

### Coverage Targets

- **Security scenarios (P0 SEC):** 100%
- **Tier gating (P0 BUS):** 100%
- **Core analytics endpoints (P1):** ≥80% AC coverage
- **Admin CRUD (P1):** ≥80% AC coverage
- **UI components (P2):** ≥60% AC coverage

---

## Mitigation Plans

### R12.1: Cross-tenant data leakage in analytics MV (Score: 6)

**Mitigation Strategy:**
1. Verify `company_id` WHERE clause on all 5 analytics MV queries using `EXPLAIN ANALYZE` output.
2. Build automated "cross-tenant injection suite": seed data for Tenant A and Tenant B; use Tenant A's JWT to call all `/analytics/*` endpoints with Tenant B's `company_id` injected; assert empty arrays or 404, never Tenant B data.
3. Add DB-level Row Level Security (RLS) policy on materialized view base tables as defence in depth.

**Owner:** Backend Lead + Security Lead
**Timeline:** Sprint 13 — gate on story S12.01 and S12.02 completion
**Status:** Partially Mitigated (S12.01 automation confirms all 5 MVs have `company_id` column at ORM and DB schema levels — 38 tests in `test_analytics_views_models.py`; 30 integration tests in `test_011_migration.py` verify schema. E2E cross-tenant injection suite pending S12.02 endpoint deployment.)
**Verification:** Automated P0 cross-tenant suite passes 100%; RLS policy in DB migration reviewed by DBA.

---

### R12.2: Admin API privilege escalation (Score: 6)

**Mitigation Strategy:**
1. Implement "Negative Admin Suite": iterate all 15+ admin routes using (a) standard user JWT, (b) no JWT, (c) expired JWT, (d) non-allowlisted IP header. Assert 401/403 on every attempt.
2. Admin JWT requires explicit `role: admin` claim — verify middleware rejects any JWT without this claim.
3. IP allowlist middleware tested in integration with configurable test IP.

**Owner:** Security Lead
**Timeline:** Sprint 13 — gate on story S12.11 completion
**Status:** Planned
**Verification:** Negative suite passes 100% — no admin route accessible without admin claim + allowlisted IP.

---

### R12.3: Analytics query latency > 3s p95 (Score: 6)

**Mitigation Strategy:**
1. Add indexes on all MV filter columns (`sector`, `country`, `date_from`, `date_to`, `company_id`) in S12.01 migration.
2. Run `EXPLAIN ANALYZE` benchmark test for each analytics endpoint against 500k-row dataset; block story completion if sequential scan detected.
3. Nightly performance test suite monitors p95 latency regression against baseline.

**Owner:** Backend Lead + Performance Team
**Timeline:** Sprint 13 (indexes) / Sprint 14 (load test gate)
**Status:** Planned
**Verification:** P0 performance benchmark passes with p95 < 3s at 50 concurrent users on staging.

---

### R12.4: Enterprise API key exposure / rate limit gap (Score: 6)

**Mitigation Strategy:**
1. API key stored hashed (bcrypt/SHA-256) — test verifies DB value differs from plaintext key.
2. Revoked key test: create → revoke → retry → assert 401 immediately (no caching of valid state).
3. Rate limit saturation test: burst N+1 requests above per-tier limit; assert 429 with correct `X-RateLimit-*` headers.
4. Single-display test: new key shown exactly once in UI; assert subsequent views are masked.

**Owner:** Backend Lead + Security Lead
**Timeline:** Sprint 14 — gate on story S12.15 completion
**Status:** Planned
**Verification:** Full API key lifecycle test suite passes 100%; rate limit saturation test returns 429.

---

### R12.5: Materialized view stale/corrupt data (Score: 6)

**Mitigation Strategy:**
1. Verify unique index exists on each MV before `CONCURRENTLY` refresh (required by PostgreSQL) — integration test asserts `\d view` confirms unique index.
2. Concurrent refresh stress test: trigger `REFRESH CONCURRENTLY` while 100 parallel read queries run; assert zero read errors and final row counts consistent.
3. Partial failure simulation: kill Celery refresh worker mid-task; assert last-valid MV state preserved (CONCURRENTLY guarantee).

**Owner:** Backend Lead + DBA
**Timeline:** Sprint 13 — gate on S12.01 completion
**Status:** Mitigated (S12.01 automation: `TestE12DB003UniqueIndexes` verifies unique indexes on all 5 MVs; `TestE12DB006RefreshConcurrently` verifies concurrent refresh SQL; `TestE12DB007ConcurrentRead` verifies parallel reads are not blocked during refresh. All 30 integration tests passing.)
**Verification:** Concurrent refresh integration test passes without read errors; row count pre- and post-refresh consistent.

---

### R12.9: Load test failure at target concurrency (Score: 6)

**Mitigation Strategy:**
1. Identify and resolve N+1 queries in all API endpoints before load test execution (S12.17 story).
2. Enable response compression (gzip/br) and CDN cache headers for static assets.
3. Load test scripts cover 4 flows: (1) search, (2) opportunity detail, (3) analytics dashboard, (4) proposal generation. Run against staging at 50/100/200 VUs.
4. Document p50/p95/p99 latencies; fix bottlenecks before release gate; document residual risks for post-launch monitoring.

**Owner:** Performance Team + Backend Lead
**Timeline:** Sprint 14 — load test must pass before release gate
**Status:** Planned
**Verification:** k6/Locust results documented; p95 < 500ms for read endpoints at 100 VUs confirmed.

---

### R12.10: Security audit failures — OWASP / network policy (Score: 6)

**Mitigation Strategy:**
1. Assign security engineer to execute OWASP Top 10 checklist against staging environment in Sprint 14.
2. Kubernetes network policy test: attempt cross-namespace service calls; assert blocked by policy.
3. Secret rotation checklist: DB passwords, JWT signing keys, API keys, S3 credentials — all rotated and stored in secret manager with rotation timestamps.
4. CORS verification: assert non-whitelisted origins receive 403 on all public endpoints.

**Owner:** Security Team + Platform Lead
**Timeline:** Sprint 14 — must complete before release gate
**Status:** Planned
**Verification:** OWASP checklist attached to release gate PR; all High findings closed or risk-accepted by CTO; secret manager audit log confirms rotation.

---

## Assumptions and Dependencies

### Assumptions

1. Staging environment has PostgreSQL with full materialized view support including `CONCURRENTLY` and unique index constraints.
2. Celery Beat schedules are configurable via environment variable for testing (e.g., `CELERY_ANALYTICS_REFRESH_INTERVAL=60` for 1-minute test cycles).
3. Redis is running in staging with configurable per-key rate limit windows for integration testing.
4. S3-compatible storage (LocalStack or AWS staging) available with public URL signing support.
5. SendGrid is mockable in staging — either via webhook receiver or HTTP stub.
6. VPN/IP allowlist middleware is configurable in staging to accept a test-mode header (`X-Test-IP: allowlisted`) without requiring actual VPN.
7. System-level blockers B-01 (Stripe mock), B-02 (tenant seeding API), B-03 (AI determinism) from architecture test design are resolved — these are required for E07/E08 regressions in P3.

### Dependencies

1. **E05/E06 data pipeline** — Materialized view data sourced from tender and opportunity tables seeded by E05/E06. Test data factory must be able to seed `tender`, `opportunity`, and `bid_preparation_logs` records directly.
2. **E07 proposal export** — S12.09 reuses document generation code from E07; shared module must be extracted and importable before S12.09 implementation begins.
3. **E08 subscription/billing** — Tier gate middleware from E08 used in S12.06, S12.07, S12.15, S12.16; billing service must be healthy in staging for tier validation.
4. **Admin JWT claim issuance** — Auth service (E02) must support `role: admin` claim in JWT. Required for all S12.11–S12.14 admin tests.
5. **PostgreSQL unique indexes on MVs** — Required before `CONCURRENTLY` refresh tests in S12.01. Migration must be deployed before S12.01 P0 suite runs.

### Risks to Plan

- **Risk**: Celery Beat schedule drift in staging due to worker restarts during testing.
  - **Impact**: S12.10 scheduled report delivery tests may be non-deterministic.
  - **Contingency**: Trigger scheduled jobs manually via admin command; assert task execution rather than scheduler timing.

- **Risk**: LocalStack S3 signed URL TTL behaviour differs from AWS in edge cases.
  - **Impact**: S12.09 signed URL expiry tests may produce false positives in staging.
  - **Contingency**: Test TTL assertion in pre-prod (AWS) environment; staging test focuses on URL generation and accessibility.

- **Risk**: Load test environment (staging) has different hardware profile from production, risking misleading p95 results.
  - **Impact**: Load test gate pass may not reflect production behaviour at scale.
  - **Contingency**: Document staging hardware specs alongside results; schedule production load test canary in Sprint 15.

---

## Interworking & Regression

| Service/Component | Impacted By Epic 12 | Regression Scope |
|:---|:---|:---|
| **Auth Service (E02)** | Admin JWT claim (`role: admin`) required for S12.11–S12.14. | Existing E02 JWT issuance and token refresh tests must still pass. |
| **Notification Service** | Celery Beat tasks for MV refresh and report delivery share worker pool. | E09 notification delivery smoke test must pass after Epic 12 deployment. |
| **Analytics Service (new)** | Core new service; materialized view refresh and query endpoints. | Full P0/P1 suite; monitor for task queue contention with E09. |
| **Proposal Service (E07)** | S12.09 reuses document generation code from E07 proposal export. | E07 proposal PDF export regression must pass (shared code path). |
| **Billing Service (E08)** | Tier gate middleware used by competitor/pipeline/enterprise API. | E08 subscription upgrade/downgrade smoke tests must still pass. |
| **S3 Storage** | Report artifacts stored and retrieved; signed URLs generated. | Verify S3 connectivity and upload/download in pre-deployment smoke test. |
| **Enterprise API Gateway** | New `api.eusolicit.com/v1/*` routing layer added. | Existing Client API endpoints unaffected — regression smoke test on `/api/v1/*`. |
| **Redis** | Rate limiting token bucket added alongside session cache. | Verify existing session management unaffected; Redis key namespace isolation. |

---

## Follow-on Workflows

- Run `*atdd` (bmad-testarch-atdd) to generate failing P0 acceptance tests for S12.01–S12.02 (tenant isolation + MV safety) — recommended first ATDD target.
- Run `*automate` (bmad-testarch-automate) after S12.09 and S12.10 are implemented to expand async report coverage.
- Run `*nfr` (bmad-testarch-nfr) in Sprint 14 to re-assess PERF and SEC NFRs against load test results.
- Run `*trace` (bmad-testarch-trace) post-sprint to verify traceability between all 14 AC-level acceptance criteria and test cases.

---

## Approval

**Test Design Approved By:**

- [ ] Product Manager: — Date: —
- [ ] Tech Lead: — Date: —
- [ ] QA Lead: — Date: —

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification, scoring (P×I), and gate decision framework
- `probability-impact.md` — Risk scoring methodology (1–3 scale)
- `test-levels-framework.md` — Test level selection (Unit / Integration / API / E2E)
- `test-priorities-matrix.md` — P0–P3 prioritization rules and decision tree
- `recurse.md` — Async polling pattern for Celery job status validation
- `api-request.md` — Playwright `apiRequest` usage for backend contract testing

### Related Documents

- PRD: `eusolicit-docs/EU_Solicit_PRD_v1.md`
- Epic: `eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md`
- Architecture: `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md`
- System-level Test Design (Architecture): `eusolicit-docs/test-artifacts/test-design-architecture.md`
- System-level Test Design (QA): `eusolicit-docs/test-artifacts/test-design-qa.md`
- Traceability Matrix: `eusolicit-docs/test-artifacts/traceability-matrix.md`
- NFR Report: `eusolicit-docs/test-artifacts/nfr-report.md`

---

**Generated by**: BMad TEA Agent — Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 4.0 (BMad v6)
**Mode**: Epic-Level (Phase 4)
