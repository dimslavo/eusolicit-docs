---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-05'
workflowType: 'testarch-atdd'
storyId: 'e12-p0'
inputDocuments:
  - 'eusolicit-docs/test-artifacts/test-design-epic-12.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-app/e2e/support/fixtures/index.ts'
  - 'eusolicit-app/e2e/support/factories/index.ts'
---

# ATDD Checklist: E12 P0 — Analytics, Admin & Enterprise API Security

## Story Summary

**Epic:** E12 — Analytics, Reporting & Admin Platform
**Scope:** P0 (Critical) acceptance tests from epic-level test design
**Sprint:** 13–14 | **Priority:** P0 | **Risk-Driven:** Yes

**Focus Areas:**
- Cross-tenant data isolation in analytics materialized views (E12-R-001, Score 6)
- Tier gate enforcement on Professional+ dashboards (E12-R-002, Score 6)
- Admin API VPN/IP restriction and admin role authentication (E12-R-008)
- Enterprise API key authentication and rate limiting (E12-R-005)

---

## TDD Red Phase (Current)

**Status:** RED — All tests use `test.skip()` and will fail until features are implemented.

| Test Type | File | Test Count | Status |
|-----------|------|------------|--------|
| API | `e2e/specs/analytics/cross-tenant-isolation.api.spec.ts` | 2 | Skipped (RED) |
| API | `e2e/specs/analytics/tier-gate-enforcement.api.spec.ts` | 7 | Skipped (RED) |
| API | `e2e/specs/admin/admin-access-control.api.spec.ts` | 4 | Skipped (RED) |
| API | `e2e/specs/enterprise/enterprise-api-auth.api.spec.ts` | 6 | Skipped (RED) |
| **Total** | **4 files** | **19 tests** | **All skipped** |

**Not generated as test code (infrastructure-level):**
- OWASP ZAP automated scan — requires ZAP tooling, not a Playwright test
- k6 load test — requires k6 scripts, not a Playwright test

---

## Acceptance Criteria Coverage

### Cross-Tenant Isolation (E12-R-001)

| AC | Test | File | Priority |
|----|------|------|----------|
| Company A cannot see Company B market intelligence data | `Company A cannot see Company B market intelligence data` | cross-tenant-isolation.api.spec.ts | P0 |
| Company A cannot see Company B data across ROI, team, competitor, pipeline, usage | `Company A cannot see Company B data across ROI, team, competitor, pipeline, and usage domains` | cross-tenant-isolation.api.spec.ts | P0 |

### Tier Gate Enforcement (E12-R-002)

| AC | Test | File | Priority |
|----|------|------|----------|
| Free tier → 403 on competitor endpoints | `Free tier user gets 403 on competitor intelligence endpoints` | tier-gate-enforcement.api.spec.ts | P0 |
| Starter tier → 403 on competitor endpoints | `Starter tier user gets 403 on competitor intelligence endpoints` | tier-gate-enforcement.api.spec.ts | P0 |
| Professional tier → 200 on competitor endpoints | `Professional tier user gets 200 on competitor intelligence endpoints` | tier-gate-enforcement.api.spec.ts | P0 |
| Enterprise tier → 200 on competitor endpoints | `Enterprise tier user gets 200 on competitor intelligence endpoints` | tier-gate-enforcement.api.spec.ts | P0 |
| Free tier → 403 on pipeline forecast | `Free tier user gets 403 on pipeline forecasting endpoint` | tier-gate-enforcement.api.spec.ts | P0 |
| Starter tier → 403 on pipeline forecast | `Starter tier user gets 403 on pipeline forecasting endpoint` | tier-gate-enforcement.api.spec.ts | P0 |
| Professional tier → 200 on pipeline forecast | `Professional tier user gets 200 on pipeline forecasting endpoint` | tier-gate-enforcement.api.spec.ts | P0 |

### Admin API Access Control (E12-R-008)

| AC | Test | File | Priority |
|----|------|------|----------|
| Non-allowlisted IP → 403 on all admin endpoints | `Non-allowlisted IP receives 403 on all admin endpoints` | admin-access-control.api.spec.ts | P0 |
| Regular user JWT → 403 on admin endpoints | `Regular user JWT receives 403 on admin endpoints` | admin-access-control.api.spec.ts | P0 |
| Admin JWT → 200 on admin endpoints | `Admin JWT with correct role receives 200 on admin endpoints` | admin-access-control.api.spec.ts | P0 |
| Missing JWT → 401 on admin endpoints | `Missing JWT receives 401 on admin endpoints` | admin-access-control.api.spec.ts | P0 |

### Enterprise API Auth & Rate Limiting (E12-R-005)

| AC | Test | File | Priority |
|----|------|------|----------|
| Valid API key → 200 | `Valid API key returns 200 on enterprise endpoints` | enterprise-api-auth.api.spec.ts | P0 |
| Invalid API key → 401 | `Invalid API key returns 401` | enterprise-api-auth.api.spec.ts | P0 |
| Missing API key → 401 | `Missing API key returns 401` | enterprise-api-auth.api.spec.ts | P0 |
| Revoked API key → 401 | `Revoked API key returns 401` | enterprise-api-auth.api.spec.ts | P0 |
| Rate limit exceeded → 429 | `Rate limit enforced — 429 returned when limit exceeded` | enterprise-api-auth.api.spec.ts | P0 |
| Rate limit headers on every response | `Rate limit headers present on every response` | enterprise-api-auth.api.spec.ts | P0 |

---

## Fixture Needs

The following fixtures are needed for GREEN phase (not created yet — tracked for implementation):

| Fixture | Purpose | Status |
|---------|---------|--------|
| `companyWithTier(tier)` | Seed company with specific subscription tier + auth token | Needed |
| `crossTenantPair()` | Seed two companies with analytics data in all domains | Needed |
| `adminUser()` | Platform admin JWT from VPN-allowlisted context | Needed |
| `enterpriseApiKey()` | Valid hashed API key for enterprise client | Needed |
| `revokedApiKey()` | API key that has been created and then revoked | Needed |
| `analyticsData(companyId)` | Seed materialized view source data for a company | Needed |

---

## Mock Requirements

| Service | Mock Type | Details |
|---------|-----------|---------|
| AI Gateway | Mock mode | Pipeline forecasting agent returns deterministic predictions (TB-02) |
| Test Data Seeding API | Required | `POST /api/test-data/seed` for companies, users, subscriptions, analytics data (TB-01) |

---

## Required data-testid Attributes

_Not applicable for P0 — all tests are API-level (no browser UI interactions)._

---

## Implementation Guidance

### Endpoints to Implement

**Analytics API (S12.02–S12.08):**
- `GET /api/v1/analytics/market/volume`
- `GET /api/v1/analytics/market/values`
- `GET /api/v1/analytics/market/authorities`
- `GET /api/v1/analytics/market/trends`
- `GET /api/v1/analytics/roi/summary`
- `GET /api/v1/analytics/team/leaderboard`
- `GET /api/v1/analytics/competitors/profiles`
- `GET /api/v1/analytics/competitors/benchmarks`
- `GET /api/v1/analytics/pipeline/forecast`
- `GET /api/v1/analytics/usage`

**Admin API (S12.11–S12.13):**
- `GET /api/v1/admin/tenants`
- `GET /api/v1/admin/crawlers/runs`
- `GET /api/v1/admin/audit-logs`
- `GET /api/v1/admin/analytics/funnel`
- `GET /api/v1/admin/analytics/revenue`

**Enterprise API (S12.15):**
- `GET /api/v1/opportunities` (via api.eusolicit.com/v1/* with X-API-Key auth)
- `POST /v1/api-keys`, `GET /v1/api-keys`, `DELETE /v1/api-keys/{key_id}`

### Infrastructure Requirements

- PostgreSQL materialized views (S12.01) must be deployed
- Tier gate middleware applied at router level for competitor and pipeline routes
- VPN/IP allowlist middleware on Admin API ingress
- Redis token bucket for Enterprise API rate limiting
- API key hashing (bcrypt/scrypt) storage

---

## Red-Green-Refactor Workflow

### RED Phase (Complete — TEA)

- [x] 19 failing API tests generated across 4 spec files
- [x] All tests use `test.skip()` to document intentional failure
- [x] Tests assert EXPECTED behavior against unimplemented endpoints
- [x] Tests organized by security domain (isolation, tier gates, admin auth, enterprise API)

### GREEN Phase (Next — DEV Team)

After implementing the feature endpoints:

1. Remove `test.skip()` from test files one domain at a time
2. Run tests for that domain:
   ```bash
   npx playwright test --grep "@P0" e2e/specs/analytics/
   npx playwright test --grep "@P0" e2e/specs/admin/
   npx playwright test --grep "@P0" e2e/specs/enterprise/
   ```
3. Verify tests PASS (green phase)
4. If any tests fail:
   - **Implementation bug**: Fix the endpoint/middleware
   - **Test bug**: Fix the test assertion (update this checklist)
5. Commit passing tests

### REFACTOR Phase (DEV Team)

- Extract common auth helpers (tier token generation, API key creation)
- Create shared fixtures for cross-tenant test scenarios
- Add test data cleanup hooks

---

## Execution Commands

```bash
# Run all E12 P0 tests (currently all skipped)
npx playwright test --grep "@P0" e2e/specs/analytics/ e2e/specs/admin/ e2e/specs/enterprise/

# Run specific domain
npx playwright test e2e/specs/analytics/cross-tenant-isolation.api.spec.ts
npx playwright test e2e/specs/analytics/tier-gate-enforcement.api.spec.ts
npx playwright test e2e/specs/admin/admin-access-control.api.spec.ts
npx playwright test e2e/specs/enterprise/enterprise-api-auth.api.spec.ts

# Run in headed mode (for debugging)
npx playwright test --headed e2e/specs/admin/admin-access-control.api.spec.ts

# Run specific test by name
npx playwright test -g "Free tier user gets 403"
```

---

## Completion Summary

| Metric | Value |
|--------|-------|
| **Story ID** | E12-P0 |
| **Primary test level** | API |
| **Total tests** | 19 |
| **API test files** | 4 |
| **E2E test files** | 0 (all P0 are API-level) |
| **Fixtures tracked** | 6 (to be created in GREEN phase) |
| **Mock requirements** | 2 (AI Gateway mock, test data seeding API) |
| **data-testid requirements** | 0 (no UI tests) |
| **Execution mode** | Sequential |
| **TDD phase** | RED (all tests skipped) |

### Knowledge Base References Applied

- `data-factories.md` — Faker-based factory patterns with Partial overrides
- `test-levels-framework.md` — API-level test selection (no E2E for auth/access control)
- `test-priorities-matrix.md` — P0 criteria (revenue/security/compliance critical)
- `test-quality.md` — No hard waits, deterministic assertions, parallel-safe

### Next Steps

1. **Implement S12.01** — Materialized views (prerequisite for analytics tests)
2. **Implement S12.06, S12.07** — Tier gate middleware (enables tier gate tests)
3. **Implement S12.11–S12.13** — Admin API endpoints (enables admin auth tests)
4. **Implement S12.15** — Enterprise API auth + rate limiting
5. **Remove `test.skip()`** — One domain at a time, verify GREEN phase
6. **Run `bmad-testarch-automate`** — Generate P1/P2/P3 tests after implementation

---

**Generated by:** BMad TEA Agent — ATDD Module
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
