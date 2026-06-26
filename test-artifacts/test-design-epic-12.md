---
workflowStatus: 'completed'
totalSteps: 5
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
nextStep: ''
lastSaved: '2026-05-25'
workflowType: 'testarch-test-design'
designLevel: 'epic'
epicNum: 12
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epics/epic-12-admin-platform.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/project-context.md'
  - 'risk-governance.md'
  - 'probability-impact.md'
  - 'test-levels-framework.md'
  - 'test-priorities-matrix.md'
---

# Test Design: Epic 12 - Admin Platform

**Date:** 2026-05-25
**Author:** Deb
**Status:** Draft
**Mode:** Epic-Level (sequential, single artifact)

> System operators securely manage platform-wide operations, monitor KPIs, and curate
> data to ensure ongoing platform health. This epic introduces the internal **admin
> portal** (Next.js admin app :3001) and **`admin-api`** (:8002, `admin` schema),
> protected by a network/IP allowlist and a fully-audited mutation surface.

---

## Executive Summary

**Scope:** Epic-level test design for Epic 12 (Admin Platform), covering its two stories:

- **S12.1 — Tenant Management:** list / suspend / reactivate companies and adjust tiers from the admin portal, behind a VPN/IP allowlist, with every mutating action audited.
- **S12.2 — KPI Dashboards:** live Recharts dashboards (MRR, active companies, AI cost ratio, crawler health) backed by scheduled materialized views.

**Risk Summary:**

- Total risks identified: **8**
- High-priority risks (≥6): **3** (R-001, R-002, R-003)
- Critical categories: **SEC** (admin-portal access + privilege), **DATA** (tenant state integrity + KPI freshness)

**Coverage Summary:**

- P0 scenarios: **12** (~16–24 hours)
- P1 scenarios: **14** (~14–22 hours)
- P2/P3 scenarios: **13** (~6–12 hours)
- **Total effort**: ~36–58 hours (~5–8 days for 1 QA + dev support)

**System-level inheritance:** This epic inherits **R-002 (cross-tenant isolation)** and the
two-tier RBAC contract from `test-design-architecture.md`. The admin portal is the
highest-privilege surface in the platform — a defect here can affect *every* tenant — so
SEC scenarios are treated as non-negotiable P0 gates.

---

## Not in Scope

| Item | Reasoning | Mitigation |
| ---- | --------- | ---------- |
| **VPN / IP allowlist network infrastructure** | The L3/L4 VPN concentrator and firewall allowlist are owned by the infra/platform team, not this epic. This plan covers only the **application-level** enforcement (middleware that reads `X-Forwarded-For` / trusted-proxy headers and denies disallowed IPs). | API + middleware tests assert deny/allow behaviour from spoofed/forwarded headers; a system test confirms the middleware is registered before any auth/route handler. |
| **Materialized view query/index performance tuning** | Deep tuning of MV refresh plans is deferred to a perf story; this epic verifies **functional correctness and freshness SLA**, not optimal query plans. | P2 baseline benchmark on dashboard endpoints (<5s) establishes a regression floor; deeper tuning tracked separately. |
| **Recharts visual pixel-perfection** | Exact chart pixel rendering is browser/library dependent and low business risk. | P3 visual-regression snapshots detect gross regressions only; data-integrity (UI value == DB value) is covered at P0. |
| **Real Stripe MRR computation correctness** | MRR *calculation* logic belongs to the billing epic (E08/E15). This epic verifies the dashboard **reads and displays** the materialized aggregate faithfully. | Dashboard tests assert UI matches the seeded MV/aggregate row, not the upstream Stripe math. |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
| ------- | -------- | ----------- | ----------- | ------ | ----- | ---------- | ----- | -------- |
| **R-001** | **SEC** | Admin portal reachable from a disallowed network due to misconfigured/mis-ordered IP-allowlist middleware (e.g. trusts client-supplied `X-Forwarded-For`, or middleware mounted after routes). Exposes platform-wide controls to the public internet. | 2 | 3 | **6** | Exhaustive middleware + API tests for allow/deny from trusted-proxy vs spoofed headers; assert middleware ordering (runs before auth/route); deny-by-default test. Constant-time / non-bypassable check. | Security Lead / Backend | End of S12.1 |
| **R-002** | **DATA/SEC** | Incorrect tenant mutation — suspending/reactivating the wrong company, or applying a tier change that bypasses tenant scoping — causes customer-facing outage or wrong billing. Cross-tenant write from admin context. | 2 | 3 | **6** | E2E coverage of every state transition (active→suspended→active) + tier matrix; API integrity assertions on the exact `company_id` mutated; negative cross-tenant test (operator action must target only the intended tenant); state-machine guard tests (illegal transitions rejected). | Backend / QA | End of S12.1 |
| **R-003** | **DATA** | KPI dashboards show stale/incorrect data because the scheduled materialized-view refresh (Celery Beat) fails silently or races, driving wrong leadership decisions. | 3 | 2 | **6** | Integration test of the MV refresh task (success + failure/retry paths); E2E data-assertion that UI metric == value queried from `admin` schema; freshness/health indicator surfaced and asserted; alert on refresh failure. | DEV / QA | End of S12.2 |

### Medium-Priority Risks (Score 3-4)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| ------- | -------- | ----------- | ----------- | ------ | ----- | ---------- | ----- |
| **R-004** | **SEC** | Audit trail for admin actions is incomplete, missing the actor/before-after/timestamp, or not written transactionally with the mutation — hindering incident investigation and compliance. | 2 | 2 | **4** | API tests asserting every mutating tenant action writes exactly one audit row with correct actor, action, target `company_id`, before/after state; verify audit write is in the same transaction (no mutation without audit). No PII/secrets logged. | DEV |
| **R-005** | **PERF** | Dashboard aggregate endpoints are slow (>5s) under realistic data volume, degrading leadership UX. | 2 | 2 | **4** | API-level baseline benchmark on each dashboard endpoint with seeded volume; assert MV-backed reads (not live aggregation); regression floor in CI. | DEV |
| **R-006** | **OPS** | Materialized views drift / migration ordering: MV definitions or refresh schedule not applied after `make reset-db` + `make migrate-all`, leaving empty dashboards in fresh environments. | 2 | 2 | **4** | Migration smoke test confirms MV objects + Beat schedule exist post-`migrate-all`; empty-state dashboard renders gracefully (no crash) when MV not yet refreshed. | DEV / Platform |

### Low-Priority Risks (Score 1-2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
| ------- | -------- | ----------- | ----------- | ------ | ----- | ------ |
| **R-007** | **SEC** | Non-operator internal user (or a regular tenant JWT) discovers and hits admin-api endpoints directly, bypassing the portal UI. | 1 | 2 | **2** | RBAC/authz tests assert non-operator principals get 403 on every admin-api route (defence in depth behind the allowlist). |
| **R-008** | **BUS** | Recharts minor-version breaking change causes dashboard render failure. | 1 | 2 | **2** | Pin Recharts version; P3 visual-regression + render smoke catch breakage. Monitor. |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

## Entry Criteria

- [ ] Epic 12 acceptance criteria for S12.1 and S12.2 approved by PM.
- [ ] `admin-api` (:8002) deployed to the test environment with `admin` schema migrated (`make migrate-all`).
- [ ] Admin Next.js app (:3001) builds and serves the `[locale]` routes.
- [ ] IP-allowlist middleware configuration documented (trusted-proxy header, allowlist source).
- [ ] Test data factories available: `CompanyFactory`, `UserFactory` (operator + non-operator roles), tier fixtures.
- [ ] An "Operator"-role test principal and at least one non-operator principal available.
- [ ] Materialized views + Celery Beat refresh schedule defined and applied.

## Exit Criteria

- [ ] All P0 tests passing (100%).
- [ ] All P1 tests passing, or failures triaged and waived.
- [ ] No open high-priority / high-severity bugs in Epic 12 functionality.
- [ ] `admin-api` line coverage ≥ **80%** (`make coverage`).
- [ ] High-priority mitigations (R-001, R-002, R-003) implemented and green.
- [ ] SEC-category scenarios pass **100%** (no waivers).
- [ ] `make lint` + `make type-check` clean; `pnpm lint && pnpm type-check` clean for the admin app (`pnpm check:i18n` if strings added).

---

## Test Coverage Plan

> **Test-level discipline:** SEC bypass and data-integrity logic is exercised at the **API/integration** level (deterministic, fast, schema-asserted). E2E is reserved for the operator's critical journey and UI↔DB data-assertion. No duplicate coverage of the same logic across levels.

### P0 (Critical) - Run on every commit

**Criteria**: Blocks core journey + High risk (≥6) + No workaround

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ----------- | ---------- | --------- | ---------- | ----- | ----- |
| S12.1 — IP allowlist enforcement | API / Middleware | R-001 | 4 | QA | Allowed-IP→pass; disallowed-IP→deny; spoofed `X-Forwarded-For`→deny; middleware runs before auth (deny precedes 401). |
| S12.1 — Tenant state transitions | API + E2E | R-002 | 5 | QA | suspend, reactivate, illegal transition rejected, exact `company_id` mutated, cross-tenant target isolation (no collateral mutation). |
| S12.1 — Tier adjustment integrity | API | R-002 | 1 | QA | Tier change persists for the correct company only; tier-cache invalidation event emitted. |
| S12.2 — Dashboard data integrity | E2E | R-003 | 2 | QA | UI MRR + active-companies values == values queried directly from `admin` schema MV. |

**Total P0**: 12 tests, ~16–24 hours

### P1 (High) - Run on PR to main

**Criteria**: Important features + Medium risk (3-4) + Common workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ----------- | ---------- | --------- | ---------- | ----- | ----- |
| S12.1 — Audit logging completeness | API / Integration | R-004 | 5 | DEV | Each mutating action writes one audit row (actor, action, target, before/after, ts); audit written in same transaction; no secrets/PII in payload. |
| S12.1 — Admin-api authz (non-operator) | API | R-007 | 2 | DEV | Non-operator + regular-tenant JWT → 403 on every admin route. |
| S12.2 — Materialized view refresh task | Integration | R-003 | 3 | DEV | Beat task refreshes MV (success); failure → retry/backoff + alert; freshness timestamp updated. |
| S12.2 — Crawler health + AI cost ratio metrics | API | R-003 | 2 | DEV | Endpoint returns correct aggregate shape; degraded/zero-data state handled. |
| S12.2 — Dashboard render states | E2E | R-008 | 2 | QA | Recharts renders MRR/active/AI-cost/crawler charts; empty + error states render without crash. |

**Total P1**: 14 tests, ~14–22 hours

### P2 (Medium) - Run nightly/weekly

**Criteria**: Secondary features + Low risk (1-2) + Edge cases

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ----------- | ---------- | --------- | ---------- | ----- | ----- |
| S12.1 — Tenant list pagination/sort/filter | API | - | 4 | DEV | Pagination bounds, sort keys, status/tier filters, empty result. |
| S12.2 — Dashboard endpoint performance baseline | API | R-005 | 2 | DEV | <5s with seeded volume; assert MV-backed (not live aggregation). |
| S12.2 — MV bootstrap after fresh migrate | Integration | R-006 | 2 | DEV | MV objects + Beat schedule exist post-`migrate-all`; empty MV renders gracefully. |
| General — Admin portal responsiveness | E2E | - | 2 | QA | Layout at desktop/tablet widths. |

**Total P2**: 10 tests, ~5–9 hours

### P3 (Low) - Run on-demand

**Criteria**: Nice-to-have + Exploratory + Performance benchmarks

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
| ----------- | ---------- | --------- | ---------- | ----- | ----- |
| S12.2 — KPI dashboard visual regression | E2E | R-008 | 2 | QA | Screenshot compare to detect unintended chart UI changes. |
| S12.1 — Audit log export (if implemented) | API | - | 1 | DEV | Export endpoint shape/permissions, only if feature lands. |

**Total P3**: 3 tests, ~1–2 hours

---

## Execution Order

### Smoke Tests (<5 min)

**Purpose**: Fast feedback, catch build-breaking issues

- [ ] `admin-api` boots and `/health` responds.
- [ ] Admin app builds and serves `[locale]` layout.
- [ ] Operator can log in to the admin portal.
- [ ] Tenant management page loads; KPI dashboard page loads.

**Total**: 4 scenarios

### P0 Tests (<15 min)

**Purpose**: Critical path validation

- [ ] Disallowed IP (incl. spoofed `X-Forwarded-For`) is denied before auth (API/middleware).
- [ ] Operator suspends then reactivates a company; only that `company_id` changes (E2E + API).
- [ ] Illegal state transition rejected; tier change isolates to target tenant (API).
- [ ] Dashboard MRR & active-companies values match `admin` schema MV (E2E data-assertion).

**Total**: 12 scenarios

### P1 Tests (<30 min)

**Purpose**: Important feature coverage

- [ ] Every mutating tenant action emits one correct, transactional audit row (API).
- [ ] Non-operator principals receive 403 across all admin-api routes (API).
- [ ] MV refresh Beat task succeeds; failure path retries + alerts (Integration).
- [ ] Charts render with data, empty, and error states (E2E).

**Total**: 14 scenarios

### P2/P3 Tests (<60 min)

**Purpose**: Full regression coverage

- [ ] Full `admin-api` API regression (`make test-service SVC=admin-api`).
- [ ] Full admin-portal E2E on Chromium (`make test-e2e-chromium`).
- [ ] Visual-regression snapshots of KPI dashboards.

**Total**: 13 scenarios

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test | Total Hours | Notes |
| -------- | ----- | ---------- | ----------- | ----- |
| P0 | 12 | ~1.5–2.0 | ~16–24 | Security + cross-tenant + data-assertion setup |
| P1 | 14 | ~1.0–1.5 | ~14–22 | Audit, MV task, authz, render states |
| P2 | 10 | ~0.5 | ~5–9 | Pagination, perf baseline, responsiveness |
| P3 | 3 | ~0.25–0.5 | ~1–2 | Visual regression / optional export |
| **Total** | **39** | **-** | **~36–58** | **~5–8 days (1 QA + dev support)** |

### Prerequisites

**Test Data:**

- `CompanyFactory` (multiple companies, varied tier/status) and `UserFactory` (operator + non-operator roles) from root `tests/conftest.py`.
- Seeded `admin` schema materialized-view rows / aggregate fixtures for deterministic dashboard assertions.
- `create_company_pair` for cross-tenant isolation negative tests.

**Tooling:**

- `pytest` markers: `@pytest.mark.api` / `@pytest.mark.integration` for admin-api; Playwright (Chromium) for portal E2E.
- Trusted-proxy / `X-Forwarded-For` header injection harness for allowlist tests.
- Celery Beat task trigger (synchronous test invocation) for MV refresh, per system-level testability ask.

**Environment:**

- `make infra` (postgres + redis) then `make migrate-all` for integration tier.
- `admin-api` (:8002) + admin app (:3001) running for `api`/E2E tiers.
- Override `get_db_session` / `get_redis_client` via `app.dependency_overrides`, cleared in `finally`. `clean_redis` flushes DB 1; app uses DB 0. Never `commit()` in tests.

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate**: 100% (no exceptions)
- **P1 pass rate**: ≥95% (waivers required for failures)
- **P2/P3 pass rate**: ≥90% (informational)
- **High-risk mitigations**: 100% complete or approved waivers for R-001, R-002, R-003

### Coverage Targets

- **Critical paths (tenant state, dashboard data integrity)**: ≥90%
- **Security scenarios (IP allowlist, admin authz, cross-tenant)**: **100%**
- **`admin-api` service line coverage**: ≥80%
- **Edge cases**: ≥50%

### Non-Negotiable Requirements

- [x] All P0 tests pass.
- [x] No high-risk (≥6) item unmitigated (R-001, R-002, R-003).
- [x] SEC-category tests pass 100% — the admin portal is the platform-wide privilege surface.
- [x] Every mutating admin action has a verified audit record (R-004).
- [x] No cross-schema joins introduced; admin-api stays within the `admin` schema.

---

## Mitigation Plans

### R-001: IP-Allowlist Bypass on Admin Portal (Score: 6)

**Mitigation Strategy:** Test the allowlist middleware exhaustively at the middleware/API
level: allowed IP → pass; disallowed → deny; client-supplied/spoofed `X-Forwarded-For` →
deny (only the configured trusted proxy hop is honoured); deny-by-default when config is
empty/missing. Assert middleware is registered **before** auth and route handlers so a
disallowed IP is rejected without reaching authenticated logic. Pair with R-007 authz as
defence in depth.
**Owner:** Security Lead / Backend
**Timeline:** End of S12.1
**Status:** Planned
**Verification:** Existing `services/admin-api/tests/middleware/test_ip_allowlist.py` extended to cover spoofed-header and ordering cases; 100% pass, zero allow on disallowed source.

### R-002: Incorrect / Cross-Tenant Tenant Mutation (Score: 6)

**Mitigation Strategy:** Cover the tenant state machine (active↔suspended, reactivate) and
the tier matrix with API integrity assertions on the exact `company_id` mutated. Reject
illegal transitions. Add a negative cross-tenant test: an operator action targeting company
A must leave company B untouched (assert B's status/tier unchanged). Confirm tier change
emits the tier-cache invalidation event rather than cross-schema writes.
**Owner:** Backend / QA
**Timeline:** End of S12.1
**Status:** Planned
**Verification:** State-transition + tier matrix green; cross-tenant collateral-mutation test asserts zero unintended changes; audit row matches the mutated target (links R-004).

### R-003: Stale / Incorrect KPI Dashboard Data (Score: 6)

**Mitigation Strategy:** Integration-test the Celery Beat MV refresh task for success and
failure (retry/backoff + alert) paths, asserting the freshness timestamp advances. E2E
data-assertion cross-references each headline metric (MRR, active companies) against the
value queried directly from the `admin` schema MV. Surface and assert a dashboard
freshness/health indicator so stale data is visible, not silent.
**Owner:** DEV / QA
**Timeline:** End of S12.2
**Status:** Planned
**Verification:** MV refresh task test passes both paths; UI value == DB MV value within freshness window; refresh failure raises an alert and flags staleness in UI.

---

## Assumptions and Dependencies

### Assumptions

1. VPN/firewall network controls are owned and operated by infra; this epic validates only the application-level allowlist middleware.
2. MRR / unit-economics *calculations* originate in the billing epics; dashboards display pre-aggregated MV values faithfully.
3. Materialized views and the Celery Beat refresh schedule are defined as part of S12.2 and applied via `make migrate-all`.
4. `admin-api` stays strictly within the `admin` schema; all cross-service reads go through APIs or events (no cross-schema joins).
5. Recharts version is pinned; chart breakage is detectable via render smoke + visual regression.

### Dependencies

1. Operator + non-operator role fixtures and `CompanyFactory`/`UserFactory` — required before S12.1 API tests.
2. Synchronous test-trigger for the MV refresh Celery task — required before S12.2 integration tests (per system-level testability ask #5).
3. Trusted-proxy header configuration documented — required before R-001 allowlist tests.
4. Seeded `admin`-schema MV/aggregate fixtures — required before S12.2 dashboard data-integrity E2E.

### Risks to Plan

- **Risk**: MV refresh has no synchronous test-trigger and tests must wait for the beat interval.
  - **Impact**: S12.2 integration tests slow/flaky.
  - **Contingency**: Invoke the Celery task function directly in-test (call the task, not the schedule), or add a test-only trigger endpoint guarded by env flag.
- **Risk**: Allowlist trusted-proxy semantics differ between local/CI and the real ingress.
  - **Impact**: Tests pass locally but the real proxy hop is mis-trusted in prod.
  - **Contingency**: Pin the trusted-proxy hop count/config in test, and add a staging smoke from a known-disallowed source before release.
- **Risk**: Dashboard fixtures diverge from real MV shape.
  - **Impact**: Data-integrity E2E green but real dashboards wrong.
  - **Contingency**: Build fixtures by running the actual MV refresh against seeded base data rather than hand-crafting aggregate rows.

---

## Follow-on Workflows (Manual)

- Run `*atdd` to generate failing P0 tests (IP allowlist deny, tenant suspend/reactivate isolation, dashboard data-assertion).
- Run `*automate` for broader coverage once S12.1/S12.2 implementation exists.
- Run `*trace` after implementation to build the Epic-12 traceability matrix and gate decision.

---

## Approval

**Test Design Approved By:**

- [ ] Product Manager: {name} Date: {date}
- [ ] Tech Lead: {name} Date: {date}
- [ ] QA Lead: {name} Date: {date}

**Comments:**

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
| ----------------- | ------ | ---------------- |
| **admin-api (:8002, `admin` schema)** | New tenant-management + KPI endpoints, IP-allowlist middleware, audit writes. | Existing `admin-api` integration suite (stage-mapping, pricing-tiers admin-only, tenant_service) must stay green. |
| **client-api (:8001)** | Tier change triggers tier-cache invalidation consumed by client-api. | Tier-gating / subscription tests must pass after a tier change made from admin. |
| **Audit trail (shared/admin)** | New mutating actions must produce audit rows. | Existing audit-trail middleware tests must pass; no schema bleed. |
| **Notification (:8005)** | MV-refresh failure / suspension may emit alerts/notifications. | Event-bus consumer tests unaffected; new alert path additive. |
| **Admin Next.js app (:3001)** | New tenant-management + dashboard routes. | `pnpm lint`/`type-check`, locale routing smoke, existing admin E2E. |

---

## Appendix

### Knowledge Base References

- `risk-governance.md` - Risk classification framework
- `probability-impact.md` - Risk scoring methodology (P×I, 1–3 scale)
- `test-levels-framework.md` - Test level selection (E2E / API / Integration / Unit)
- `test-priorities-matrix.md` - P0–P3 prioritization

### Related Documents

- Epic: `eusolicit-docs/planning-artifacts/epics/epic-12-admin-platform.md`
- System-level (architecture): `eusolicit-docs/test-artifacts/test-design-architecture.md` (inherits R-002 cross-tenant, two-tier RBAC, R-014 HMAC discipline)
- System-level (QA): `eusolicit-docs/test-artifacts/test-design-qa.md`
- Project context: `eusolicit-docs/project-context.md`

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 4.0 (BMad v6)
