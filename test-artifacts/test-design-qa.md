---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-07'
workflowType: 'testarch-test-design'
inputDocuments:
  - 'eusolicit-docs/EU_Solicit_PRD_v1.md'
  - 'eusolicit-docs/EU_Solicit_Requirements_Brief_v4.md'
  - 'eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md'
  - 'eusolicit-docs/EU_Solicit_User_Journeys_and_Workflows_v1.md'
---

# Test Design for QA: EU Solicit

**Purpose:** Test execution recipe for QA team. Defines what to test, how to test it, and what QA needs from other teams.

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**Status:** Draft
**Project:** EU Solicit

**Related:** See Architecture doc (test-design-architecture.md) for testability concerns, architectural blockers, and risk mitigation plans.

---

## Executive Summary

**Scope:** Full-platform system-level test design for EU Solicit — 5 backend services (FastAPI/Celery), Next.js 14 frontend, KraftData AI integration (22 agents/teams/workflows), Stripe billing, calendar sync, data pipeline.

**Risk Summary:**

- Total Risks: 15 (5 high-priority ≥6, 6 medium, 4 low)
- Critical Categories: TECH (KraftData dependency), SEC (multi-tenant isolation, RBAC, SSE saturation), BUS (billing complexity)

**Coverage Summary:**

- P0 tests: ~18 (auth, billing, data isolation, core AI flows)
- P1 tests: ~25 (opportunity discovery, proposal lifecycle, collaboration, alerts)
- P2 tests: ~30 (edge cases, analytics, admin features, calendar sync)
- P3 tests: ~15 (exploratory, performance benchmarks, i18n)
- **Total: ~88 tests (~6–9 weeks with 1 QA)**

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| Electronic bid submission | Out of MVP scope per PRD Section 7 | Platform provides submission guides; users submit externally |
| Microsoft OAuth2 login | Deferred post-MVP per ADR 1.8 | Google OAuth2 + email/password covered |
| E-procurement portal connectors | Post-MVP per Requirements Brief Section 11 | Crawlers only; no direct portal integration |
| ERP/CRM integrations | Post-MVP per Requirements Brief Section 11 | No Salesforce/SAP/HubSpot testing |
| Digital signatures (eIDAS) | Post-MVP per ADR 1.8 | Document export as PDF/DOCX tested instead |
| Elasticsearch faceted search | Post-MVP per Architecture Section 4.3 | PostgreSQL FTS tested |
| KraftData agent output quality | Owned by KraftData via eval-runs | EU Solicit tests integration contract, not AI output quality |

---

## Dependencies & Test Blockers

**CRITICAL:** QA cannot proceed without these items from other teams.

### Backend/Architecture Dependencies (Pre-Implementation)

**Source:** See Architecture doc "Quick Guide" for detailed mitigation plans

1. **Test Data Seeding API (TB-01)** — Backend Lead — Pre-implementation
   - QA needs: `POST /api/test-data/seed` with support for users, companies, subscriptions (all tiers + trial states), opportunities, proposals, tasks, and approval workflows
   - Blocks: All integration and E2E tests

2. **KraftData Mock Service (TB-02)** — Backend Lead — Pre-implementation
   - QA needs: AI Gateway mock mode returning deterministic fixture responses for all 22 agent types (summaries, proposals, compliance checks, scoring)
   - Blocks: All AI feature tests (proposal generation, scoring simulator, compliance validator, document parser)

3. **Stripe Test Configuration (TB-03)** — Backend Lead — Pre-implementation
   - QA needs: Stripe test-mode keys, webhook simulation (Stripe CLI), test clocks for trial lifecycle
   - Blocks: All billing tests (subscription creation, trial expiry, add-on purchases, VAT)

### QA Infrastructure Setup (Pre-Implementation)

1. **Test Data Factories** — QA
   - User factory (all roles: admin, bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only)
   - Company factory (all tiers: free, starter, professional, enterprise, trial)
   - Opportunity factory (various statuses, CPV codes, regions, budgets)
   - Proposal factory (with versions, collaborators, comments, section locks)
   - Auto-cleanup fixtures for parallel safety

2. **Test Environments** — QA/DevOps
   - Local: Docker Compose with all 5 services + PostgreSQL + Redis + MinIO + ClamAV
   - CI/CD: Kubernetes namespace per PR with ephemeral databases
   - Staging: Shared environment with Stripe test mode + KraftData sandbox

**Example factory pattern:**

```python
# tests/factories.py
import factory
from faker import Faker

fake = Faker()

class UserFactory(factory.Factory):
    class Meta:
        model = dict

    email = factory.LazyFunction(fake.email)
    name = factory.LazyFunction(fake.name)
    role = "user"
    company_id = factory.LazyFunction(fake.uuid4)

class CompanyFactory(factory.Factory):
    class Meta:
        model = dict

    name = factory.LazyFunction(fake.company)
    tier = "professional"
    tax_id = factory.LazyFunction(lambda: f"BG{fake.random_number(digits=9)}")
```

```typescript
// tests/e2e/fixtures.ts — example P0 API test
import { test, expect } from '@playwright/test';
import { faker } from '@faker-js/faker';

test('@P0 @API user registration returns 201', async ({ request }) => {
  const userData = {
    email: faker.internet.email(),
    password: 'TestPass123!',
    company_name: faker.company.name(),
  };

  const response = await request.post('/api/v1/auth/register', { data: userData });
  expect(response.status()).toBe(201);

  const body = await response.json();
  expect(body.user.email).toBe(userData.email);
  expect(body.token).toBeTruthy();
});
```

---

## Risk Assessment

**Note:** Full risk details and mitigation plans in Architecture doc. This section summarises risks relevant to QA test planning.

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Score | QA Test Coverage |
|---------|----------|-------------|-------|-----------------|
| **R-001** | TECH | KraftData single point of failure | **9** | Test with mock mode ON and OFF; verify graceful degradation UX when circuit breaker opens |
| **R-002** | SEC | Multi-tenant data isolation | **6** | Cross-tenant access tests: Company A cannot see Company B data at every endpoint |
| **R-003** | PERF | SSE stream saturation | **6** | k6 load test: concurrent SSE + REST under sustained load |
| **R-004** | BUS | Stripe billing complexity | **6** | Full lifecycle: free → trial → paid → cancel; add-on purchases; VAT edge cases |
| **R-006** | SEC | Entity-level RBAC enforcement | **6** | Permission matrix: all 5 roles × all entity types × all permission levels |

### Medium/Low-Priority Risks

| Risk ID | Category | Description | Score | QA Test Coverage |
|---------|----------|-------------|-------|-----------------|
| R-005 | DATA | Data pipeline staleness | 4 | Verify stale-data UX when pipeline is down |
| R-007 | TECH | Redis Streams delivery | 4 | Event flow integration tests across all 7 streams |
| R-008 | PERF | PostgreSQL shared instance bottleneck | 4 | k6 search/listing load test with 10K+ opportunities |
| R-009 | OPS | Calendar sync OAuth | 4 | OAuth mock + token refresh + revocation tests |
| R-010 | SEC | File upload ClamAV | 3 | Upload clean file (201) + infected file (422) |
| R-012 | BUS | Tier gating accuracy | 4 | Parametrized tier boundary tests (region/CPV/budget) |
| R-015 | SEC | Proposal collaboration locking | 4 | Concurrent edit race condition tests |

---

## Entry Criteria

- [ ] All requirements and assumptions agreed upon by QA, Dev, PM
- [ ] Test environments provisioned (Docker Compose local + CI namespace)
- [ ] Test data factories implemented (user, company, opportunity, proposal)
- [ ] Pre-implementation blockers resolved (TB-01, TB-02, TB-03)
- [ ] AI Gateway mock mode operational
- [ ] Stripe test mode configured with webhook forwarding

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95% or failures triaged and accepted)
- [ ] No open P0/P1 severity bugs
- [ ] Cross-tenant isolation validated
- [ ] Performance baselines met (p95 < 200ms REST, < 500ms SSE TTFB)

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on when time-constrained), NOT execution timing. See "Execution Strategy" for when tests run.

### P0 (Critical)

**Criteria:** Blocks core functionality + High risk (≥6) + No workaround + Affects majority of users

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **P0-001** | User registration (email/password) | API | R-002 | Validate company creation + Professional trial subscription initialised |
| **P0-002** | Google OAuth2 login | API + E2E | R-002 | OAuth flow + JWT issuance + session creation |
| **P0-003** | JWT RS256 validation — expired token rejected | API | R-006 | 401 on expired/invalid/missing token |
| **P0-004** | JWT RS256 validation — insufficient permissions | API | R-006 | 403 on role mismatch |
| **P0-005** | Entity-level RBAC — company role ceiling enforcement | API | R-006 | bid_manager cannot escalate to admin at entity level |
| **P0-006** | Entity-level RBAC — cross-proposal isolation | API | R-002, R-006 | User with edit on Proposal A receives 403 on Proposal B |
| **P0-007** | Cross-tenant data isolation — Company A vs Company B | API | R-002 | Full endpoint sweep: opportunities, proposals, tasks, analytics |
| **P0-008** | Free-tier limited view — only name, deadline, location, type, status | API | R-012 | Verify gated fields return null/omitted for free users |
| **P0-009** | Paid-tier access boundaries — region/CPV/budget enforcement | API | R-012 | Starter: BG only, 1 CPV sector, ≤€500K |
| **P0-010** | Stripe Checkout — subscription creation | API | R-004 | Webhook: checkout.session.completed → subscription active |
| **P0-011** | Trial lifecycle — 14-day Professional, no CC, downgrade on expiry | API | R-004 | Stripe test clock: advance 15 days → tier = Free |
| **P0-012** | Usage metering — AI summary limit enforcement | API | R-004, R-012 | Starter at 10/10 → 429 with upgrade prompt |
| **P0-013** | AI summary generation — streaming SSE response | API + E2E | R-001 | Via AI Gateway mock; verify SSE stream + usage counter increment |
| **P0-014** | Proposal draft generation — multi-agent workflow streaming | API | R-001 | Via AI Gateway mock; verify proposal version created |
| **P0-015** | Compliance validator — framework-specific validation | API | R-001 | Via AI Gateway mock; verify checklist output |
| **P0-016** | Document upload — ClamAV virus scan + S3 storage | API | R-010 | Upload clean file (201) + infected file (422) |
| **P0-017** | Opportunity search with filters — CPV, region, budget, deadline | API | — | Full-text search + filter combination |
| **P0-018** | Audit trail — immutable log for all platform actions | API | R-002 | Verify audit_log entries on create/update/delete |

**Total P0:** ~18 tests

---

### P1 (High)

**Criteria:** Important features + Medium risk (3–5) + Common workflows + Workaround exists but difficult

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **P1-001** | Opportunity listing — paginated, sorted, tier-gated | API | R-012 | Verify pagination, sort, and tier-appropriate response shape |
| **P1-002** | Opportunity detail — full documentation for paid tiers | API + E2E | R-012 | All fields present for Professional tier |
| **P1-003** | Smart matching — relevance score against company profile | API | R-001 | Via AI Gateway mock; verify 0-100 score returned |
| **P1-004** | Email alert configuration — CPV, region, budget, frequency | API | — | Save preferences + verify preference_updated event published |
| **P1-005** | Email digest delivery — matching opportunities in digest | API | R-007 | Via notification service; verify SendGrid call |
| **P1-006** | Proposal versioning — create, update, rollback | API | — | Version history preserved; diff capability |
| **P1-007** | Proposal rich text editing — Tiptap content save/load | E2E | — | Content persists across page reload |
| **P1-008** | Proposal export — PDF and DOCX generation | API | — | Verify file download + content integrity |
| **P1-009** | Per-proposal role assignment — all 5 entity roles | API | R-006 | Assign role → verify access scoping |
| **P1-010** | Section locking — pessimistic lock acquire/release/TTL expiry | API | R-015 | Lock, verify 423 for other user, wait TTL, verify auto-released |
| **P1-011** | Proposal comments — create, resolve, carry forward across versions | API | — | Comment anchored to section + version |
| **P1-012** | Task orchestration — create, assign, dependencies (DAG validation) | API | — | Circular dependency rejected; overdue detection |
| **P1-013** | Approval workflows — stage sequence, sign-off, rejection | API | — | technical → legal → management flow |
| **P1-014** | Bid/no-bid decision — structured scoring + AI recommendation | API | R-001 | Via AI Gateway mock; override logged in audit trail |
| **P1-015** | Bid outcome recording + preparation time logging | API | — | Won/lost/withdrawn + hours/costs |
| **P1-016** | Content blocks library — CRUD, tagging, versioning | API | — | Create block → use in proposal → verify insertion |
| **P1-017** | ESPD auto-fill — company profile → ESPD form mapping | API | — | Structured form populated from company data |
| **P1-018** | Submission guides — portal-specific step-by-step instructions | API | — | Guide linked to opportunity; steps accessible |
| **P1-019** | Subscription upgrade/downgrade — Stripe Customer Portal | API | R-004 | Webhook: subscription_updated → tier change |
| **P1-020** | Per-bid add-on purchase — one-time Stripe Checkout | API | R-004 | checkout.session.completed → add-on unlocked for specific opportunity |
| **P1-021** | EU VAT calculation — B2B reverse charge + B2C with VAT | API | R-004 | Verify tax_id validation + correct VAT on invoice |
| **P1-022** | Company profile management — credentials, team members | API | — | CRUD + team member invitation |
| **P1-023** | Data pipeline — crawl trigger → normalize → enrich → upsert | API | R-005 | Via AI Gateway mock; verify opportunity created in pipeline schema |
| **P1-024** | Document parser — extract requirements from tender package | API | R-001 | Via AI Gateway mock; verify structured extraction |
| **P1-025** | Rate limiting — per-user, per-IP enforcement | API | R-008 | Exceed limit → 429 with Retry-After header |

**Total P1:** ~25 tests

---

### P2 (Medium)

**Criteria:** Secondary features + Low risk (1–2) + Edge cases + Regression prevention

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **P2-001** | Competitor tracking — award data aggregation | API | — | Via AI Gateway mock |
| **P2-002** | Pipeline forecasting — predicted opportunities | API | — | Via AI Gateway mock |
| **P2-003** | Scoring simulator — per-criterion scorecard | API | R-001 | Via AI Gateway mock |
| **P2-004** | Pricing assistant — competitive pricing suggestion | API | R-001 | Via AI Gateway mock |
| **P2-005** | Grant eligibility matcher | API | R-001 | Via AI Gateway mock |
| **P2-006** | Budget builder — EU cost category rules | API | R-001 | Via AI Gateway mock |
| **P2-007** | Logframe generator | API | R-001 | Via AI Gateway mock |
| **P2-008** | Consortium finder | API | R-001 | Via AI Gateway mock |
| **P2-009** | iCal feed generation — all tiers | API | — | Subscribe URL → valid iCal content |
| **P2-010** | Google Calendar sync — two-way | API | R-009 | OAuth mock + event creation/update/delete |
| **P2-011** | Microsoft Outlook sync — two-way | API | R-009 | OAuth mock + event creation/update/delete |
| **P2-012** | Market intelligence dashboard | API + E2E | — | Sector analytics data display |
| **P2-013** | ROI tracker — preparation cost vs. contract value | API | — | Calculation correctness |
| **P2-014** | Team performance metrics | API | — | Per-user win rate, avg prep time |
| **P2-015** | Usage dashboard — current period vs. limits | API + E2E | R-012 | Correct counts displayed for all paid tiers |
| **P2-016** | Report export — PDF/DOCX scheduled/on-demand | API | — | File download + content |
| **P2-017** | Admin — compliance framework CRUD + assignment | API | — | Create framework → assign to opportunity |
| **P2-018** | Admin — crawler management (schedule, trigger, history) | API | — | Trigger manual crawl → verify run record |
| **P2-019** | Admin — tenant management (view companies, subscriptions) | API | — | List/filter tenants |
| **P2-020** | Admin — audit log viewer (search, filter, export) | API | — | Search by user, action, entity |
| **P2-021** | Admin — white-label config (subdomain, logo, colours, email) | API | R-014 | Enterprise tenant only |
| **P2-022** | Admin — platform analytics (signup funnel, churn) | API | — | Metrics data accuracy |
| **P2-023** | Document retention — soft delete + S3 purge after grace period | API | R-011 | Celery Beat task execution |
| **P2-024** | Clause risk flagging — high-risk clause identification | API | R-001 | Via AI Gateway mock |
| **P2-025** | Requirement checklist builder — auto-generated compliance checklist | API | R-001 | Via AI Gateway mock |
| **P2-026** | Redis Streams event flow — opportunities.ingested → Notification Svc | API | R-007 | Event published → consumed → alert generated |
| **P2-027** | Trial expiry reminder — email 3 days before end | API | R-004 | Stripe webhook: trial_will_end → email sent |
| **P2-028** | Approval notifications — email on request + decision | API | R-007 | Event → email delivery |
| **P2-029** | Task notifications — email on assignment + overdue | API | R-007 | Event → email delivery |
| **P2-030** | Enterprise API — OpenAPI spec auto-generated + accessible | API | — | Verify spec endpoint accessible for Enterprise tier |

**Total P2:** ~30 tests

---

### P3 (Low)

**Criteria:** Nice-to-have + Exploratory + Performance benchmarks

| Test ID | Requirement | Test Level | Notes |
|---------|-------------|------------|-------|
| **P3-001** | i18n — Bulgarian language toggle | E2E | UI text renders in Bulgarian |
| **P3-002** | i18n — AI outputs in user's preferred language | API | Via AI Gateway mock |
| **P3-003** | Free discovery → upgrade conversion flow | E2E | Full Journey 1 (Maria persona) |
| **P3-004** | Starter monitoring → AI summary flow | E2E | Full Journey 2 (Georgi persona) |
| **P3-005** | Professional proposal lifecycle flow | E2E | Full Journey 3 (Elena persona) |
| **P3-006** | Enterprise multi-client management flow | E2E | Full Journey 4 (Nikolai persona) |
| **P3-007** | SEO — public landing pages SSR with metadata | E2E | Verify SSR content in page source |
| **P3-008** | Onboarding wizard — company profile, CPV prefs, region | E2E | First-login experience |
| **P3-009** | k6: API latency — p95 < 200ms for REST endpoints | Perf | 100 concurrent users, 5-min sustained |
| **P3-010** | k6: SSE TTFB — p95 < 500ms for streaming | Perf | 50 concurrent SSE streams |
| **P3-011** | k6: Opportunity search under load — 10K+ tenders | Perf | Full-text search + filters |
| **P3-012** | k6: Concurrent proposal generation — 20 simultaneous streams | Perf | AI Gateway SSE saturation point |
| **P3-013** | Lessons learned agent — post-bid analysis | API | Via AI Gateway mock |
| **P3-014** | Regulation tracker — regulatory change detection | API | Via AI Gateway mock |
| **P3-015** | Reporting template generator — post-award templates | API | Via AI Gateway mock |

**Total P3:** ~15 tests

---

## Execution Strategy

**Philosophy:** Run everything in PRs unless there's significant infrastructure overhead. Playwright with parallelisation handles 100s of tests in ~10–15 min.

**Organised by TOOL TYPE:**

### Every PR: Playwright + pytest Tests (~10–15 min)

**All functional tests** (from any priority level):

- All E2E, API, and integration tests using Playwright + pytest
- Parallelised across 4 shards (Playwright) and pytest-xdist workers
- Total: ~73 functional tests (P0–P3 excluding k6 perf tests)

**Why run in PRs:** Fast feedback; no expensive infrastructure needed.

### Nightly: k6 Performance Tests (~30–60 min)

**All performance tests** (P3-009 through P3-012):

- Load, stress, and spike tests against staging environment
- Total: ~4 k6 test scenarios

**Why defer to nightly:** Requires dedicated staging environment + k6 cloud runner + 30+ min runtime.

### Weekly: Chaos & Long-Running (~hours)

- KraftData unavailability simulation (circuit breaker validation)
- Database failover (if configured post-MVP)
- Redis Streams recovery after broker restart
- Document retention lifecycle (end-to-end S3 purge validation)

**Why defer to weekly:** Very long-running; requires infrastructure fault injection.

---

## QA Effort Estimate

**QA test development effort only** (excludes DevOps, Backend, Data Eng work):

| Priority | Count | Effort Range | Notes |
|----------|-------|-------------|-------|
| P0 | ~18 | ~2–3 weeks | Complex setup: auth flows, billing lifecycle, multi-tenant isolation, AI mock integration |
| P1 | ~25 | ~2–3 weeks | Standard API/integration: proposal lifecycle, collaboration, notifications |
| P2 | ~30 | ~1–2 weeks | Straightforward API tests: admin features, edge cases, event flow |
| P3 | ~15 | ~3–5 days | E2E user journeys + k6 performance scripts |
| **Total** | **~88** | **~6–9 weeks** | **1 QA engineer, full-time** |

**Assumptions:**
- Includes test design, implementation, debugging, CI integration
- Excludes ongoing maintenance (~10% effort)
- Test infrastructure (factories, fixtures, mock services) ready before test development begins
- Effort decreases to ~3–5 weeks with 2 QA engineers working in parallel

---

## Implementation Planning Handoff

| Work Item | Owner | Target Sprint | Dependencies/Notes |
|-----------|-------|--------------|-------------------|
| Test data seeding API | Backend | Sprint 3 | TB-01; blocks all QA test development |
| AI Gateway mock mode | Backend | Sprint 3 | TB-02; blocks all AI feature tests |
| Stripe test configuration | Backend | Sprint 4 | TB-03; blocks billing tests |
| Test data factories (Python + TS) | QA | Sprint 4 | Depends on seeding API |
| Docker Compose test environment | QA/DevOps | Sprint 3 | All 5 services + deps |
| P0 test implementation | QA | Sprint 5–6 | Depends on factories + mocks |
| P1 test implementation | QA | Sprint 7–8 | Depends on P0 baseline |
| P2/P3 test implementation | QA | Sprint 9–10 | Ongoing alongside feature dev |
| k6 performance scripts | QA | Sprint 10 | Requires stable staging environment |

---

## Tooling & Access

| Tool or Service | Purpose | Access Required | Status |
|----------------|---------|-----------------|--------|
| Playwright | E2E + API testing | npm install | Ready |
| pytest + pytest-asyncio | Unit + integration testing | pip install | Ready |
| k6 | Performance/load testing | CLI install + k6 Cloud account | Pending |
| Stripe CLI | Webhook simulation + test clocks | Stripe account access | Pending |
| KraftData Sandbox | AI agent testing (when not using mocks) | API key for dev environment | Pending |
| SendGrid | Email delivery testing | Test API key | Pending |
| MinIO | Local S3-compatible storage | Docker Compose | Ready |
| ClamAV | Virus scan testing | Docker sidecar | Ready |

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope | Validation Steps |
|-------------------|--------|-----------------|-----------------|
| **Client API** | Primary test target — all user-facing features | Full P0 + P1 suite | All Playwright + pytest tests pass |
| **AI Gateway** | Integration point for all AI features | Agent execution + SSE proxy tests | Mock mode tests + circuit breaker validation |
| **Data Pipeline** | Opportunity data freshness | Crawl → normalize → upsert flow | Pipeline integration tests with mock agents |
| **Notification Service** | Alert delivery + calendar sync | Event consumption → email/calendar tests | Redis Streams event flow validation (7 streams) |
| **Admin API** | Compliance frameworks + tenant management | Admin-specific API tests | VPN-restricted endpoint access validation |

**Regression strategy:** Every PR runs full Playwright + pytest suite (~10–15 min). Cross-service integration tests run against Docker Compose environment. Breaking changes detected by Pydantic DTO validation across service boundaries.

---

## Appendix A: Code Examples & Tagging

**Playwright Tags for Selective Execution:**

```typescript
import { test, expect } from '@playwright/test';
import { faker } from '@faker-js/faker';

// P0 critical — authentication
test('@P0 @API @Auth user registration creates company and subscription', async ({ request }) => {
  const userData = {
    email: faker.internet.email(),
    password: 'SecurePass123!',
    company_name: faker.company.name(),
  };

  const response = await request.post('/api/v1/auth/register', { data: userData });
  expect(response.status()).toBe(201);

  const body = await response.json();
  expect(body.user.email).toBe(userData.email);
  expect(body.company).toBeTruthy();
  expect(body.subscription.tier).toBe('professional'); // 14-day trial
  expect(body.subscription.trial_end).toBeTruthy();
  expect(body.token).toBeTruthy();
});

// P0 critical — cross-tenant isolation
test('@P0 @API @Security company A cannot access company B proposals', async ({ request }) => {
  const companyA = await seedCompany(request, { tier: 'professional' });
  const companyB = await seedCompany(request, { tier: 'professional' });
  const proposal = await seedProposal(request, { company_id: companyB.id });
  const tokenA = await getAuthToken(request, companyA.user.email);

  const response = await request.get(`/api/v1/proposals/${proposal.id}`, {
    headers: { Authorization: `Bearer ${tokenA}` },
  });

  expect(response.status()).toBe(403);
});

// P1 — billing lifecycle
test('@P1 @API @Billing trial expiry downgrades to free tier', async ({ request }) => {
  const company = await seedCompany(request, { tier: 'professional', trial: true });

  await simulateStripeWebhook(request, 'customer.subscription.updated', {
    subscription_id: company.stripe_subscription_id,
    status: 'past_due',
  });

  const response = await request.get(`/api/v1/companies/${company.id}/subscription`);
  const subscription = await response.json();
  expect(subscription.tier).toBe('free');
});
```

**pytest Tags for Backend:**

```python
# tests/api/test_auth.py
import pytest
from httpx import AsyncClient

@pytest.mark.p0
@pytest.mark.auth
async def test_expired_jwt_returns_401(client: AsyncClient, expired_token: str):
    response = await client.get(
        "/api/v1/opportunities",
        headers={"Authorization": f"Bearer {expired_token}"},
    )
    assert response.status_code == 401
    assert "expired" in response.json()["detail"].lower()

@pytest.mark.p0
@pytest.mark.security
async def test_cross_tenant_isolation(
    client: AsyncClient,
    company_a_token: str,
    company_b_proposal_id: str,
):
    response = await client.get(
        f"/api/v1/proposals/{company_b_proposal_id}",
        headers={"Authorization": f"Bearer {company_a_token}"},
    )
    assert response.status_code == 403
```

**Run by tag:**

```bash
# Playwright — run by priority
npx playwright test --grep @P0
npx playwright test --grep "@P0|@P1"

# Playwright — run by domain
npx playwright test --grep @Auth
npx playwright test --grep @Billing

# pytest — run by priority
pytest -m p0
pytest -m "p0 or p1"

# pytest — run by domain
pytest -m auth
pytest -m security
```

---

## Appendix B: Knowledge Base References

- **Risk Governance:** `risk-governance.md` — Risk scoring methodology (P × I, thresholds ≥6)
- **ADR Quality Readiness:** `adr-quality-readiness-checklist.md` — 8-category, 29-criteria testability framework
- **Test Levels Framework:** `test-levels-framework.md` — E2E vs API vs Unit selection rules
- **Test Quality:** `test-quality.md` — Definition of Done (no hard waits, <300 lines, <1.5 min, self-cleaning)

---

**Generated by:** BMad TEA Agent
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.1 (BMad v6) — Updated 2026-04-07: corrected risk classification (R-003 HIGH, R-006 HIGH), updated agent count to 22
