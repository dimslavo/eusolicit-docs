---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-09'
workflowType: 'testarch-test-design'
inputDocuments:
  - 'eusolicit-docs/EU_Solicit_PRD_v1.md'
  - 'eusolicit-docs/EU_Solicit_Requirements_Brief_v4.md'
  - 'eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md'
  - 'eusolicit-docs/EU_Solicit_User_Journeys_and_Workflows_v1.md'
---

# Test Design for QA: EU Solicit

**Purpose:** Test execution recipe for QA team. Defines what to test, how to test it, and what QA needs from other teams.

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**Status:** Draft
**Project:** EU Solicit
**Version:** v5 (2026-04-09) — expanded to 92 test scenarios; added R-016 (locale routing) and R-017 (token refresh race) to risk table and P1 coverage; corrected KraftData entity count to 29; incorporated epic-level learnings from E01/E02/E03

**Related:** See Architecture doc (test-design-architecture.md) for testability concerns, architectural blockers, risk mitigation plans, and assumptions.

---

## Executive Summary

**Scope:** Full-platform system-level test design for EU Solicit — 5 backend services (FastAPI/Celery), Next.js 14 frontend, KraftData AI integration (29 entities: 27 agents, 1 team, 1 workflow; 7 vector stores), Stripe billing (trials, add-ons, EU VAT, usage metering), Google Calendar + Outlook sync, data pipeline crawlers (AOP, TED, EU Grant portals), proposal collaboration + task orchestration + approval workflows, entity-level RBAC, ESPD auto-fill, submission guides, analytics dashboards + reporting.

**Risk Summary:**

- Total Risks: 17 (5 high-priority ≥6, 8 medium, 4 low)
- Critical Categories: TECH (KraftData SPOF, Redis Streams DLQ), SEC (multi-tenant isolation, entity RBAC, token security), BUS (billing lifecycle), PERF (SSE saturation, PostgreSQL FTS)

**Coverage Summary:**

- P0 tests: ~20 (auth, billing, data isolation, core AI flows, service health)
- P1 tests: ~27 (opportunity discovery, proposal lifecycle, collaboration, alerts, calendar, locale routing, token refresh)
- P2 tests: ~30 (edge cases, analytics, admin features, calendar edge cases, compliance, ESPD, submission guides)
- P3 tests: ~15 (E2E persona journeys, k6 performance benchmarks, i18n, white-label)
- **Total: ~92 tests (~6–9 weeks with 1 QA, ~3–5 weeks with 2 QAs)**

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| Electronic bid submission | Out of MVP scope per PRD §3 / ADR 1.8 | Platform provides submission guides; users submit externally via portal |
| Microsoft OAuth2 login | Deferred post-MVP per ADR 1.8 | Google OAuth2 + email/password covered; authlib infra ready for post-MVP |
| E-procurement portal connectors | Post-MVP per Requirements Brief §11 | Crawlers only; no direct portal integration; `SubmissionPort` interface defined |
| ERP/CRM integrations | Post-MVP per Requirements Brief §11 | No Salesforce/SAP/HubSpot/1C testing |
| Digital signatures (eIDAS) | Post-MVP per ADR 1.8 | Document export as PDF/DOCX tested instead; `SignatureProviderPort` interface defined |
| Elasticsearch faceted search | Post-MVP per Architecture §4.3 | PostgreSQL FTS tested; Elasticsearch migration path tested when introduced |
| KraftData agent output quality | Owned by KraftData via eval-runs | EU Solicit tests integration contract and mock-mode correctness, not AI output quality |
| Reporting Template Generator (grant winners) | Enterprise feature; post-MVP usage | Basic export path tested; post-award template generation deferred |
| Consultant marketplace | Post-MVP per Requirements Brief §11 | Company profiles vault exists; marketplace feature deferred |

---

## Dependencies & Test Blockers

**CRITICAL:** QA cannot proceed without these items from other teams.

### Backend/Architecture Dependencies (Pre-Implementation)

**Source:** See Architecture doc "Quick Guide" for detailed mitigation plans

1. **Test Data Seeding API (TB-01)** — Backend Lead — Sprint 3
   - QA needs: `POST /api/test-data/seed` supporting: users (all 5 entity roles, all 5 company roles), companies (free/starter/professional/enterprise/trial states), opportunities (various CPV codes, regions BG/EU, budget ranges, statuses), proposals (with versions, collaborators, section locks, comments), tasks (with DAG dependencies and templates), approval workflows (multi-stage configurable sequences)
   - QA needs: `DELETE /api/test-data/cleanup` with transaction rollback or cascaded delete for test isolation
   - Blocks: All integration and E2E tests; without this, every test requires full registration → profile → subscription setup chain (~5 min per test)

2. **KraftData Mock Service (TB-02)** — Backend Lead — Sprint 3
   - QA needs: AI Gateway mock mode activated via env flag (`AI_GATEWAY_MOCK=true`) returning deterministic fixture responses for all 29 entity types (27 agents, 1 team, 1 workflow)
   - Mock must support: synchronous agent responses, SSE streaming responses (Proposal Generator Workflow), configurable per-agent-type delay simulation, failure injection (for circuit-breaker tests), and agent-type-specific response schemas matching production contracts
   - Blocks: All AI feature tests — proposal generation (SSE), document analysis, compliance checking, scoring simulation, bid/no-bid decision, analytics agents; without mock, all AI paths require live KraftData = non-deterministic, slow, CI-unavailable

3. **Stripe Test Configuration (TB-03)** — Backend Lead — Sprint 4
   - QA needs: Stripe test-mode API keys configured in all test environments; `stripe listen --forward-to` setup documented and runnable in CI; Stripe test clock API enabled on test account; webhook endpoint verification key
   - Covers: subscription creation (checkout.session.completed), trial lifecycle (trial_will_end → expiry → downgrade to Free), add-on Checkout Sessions (mode=payment), payment failure (payment_intent.payment_failed), subscription cancellation, EU VAT (Stripe Tax test mode with VIES mock)
   - Blocks: All billing tests — P0-005 through P0-007 cannot run; trial/add-on/VAT edge case coverage blocked

### QA Infrastructure Setup (Pre-Implementation)

1. **Test Data Factories** — QA (Sprint 3)
   - `UserFactory` — all 5 entity roles (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only) + platform_admin; faker-randomized email/name
   - `CompanyFactory` — all 4 tiers (free, starter, professional, enterprise) + trial state; includes `tax_id` for VAT tests
   - `OpportunityFactory` — various CPV codes, regions (BG, EU), budget ranges (< €500K, €500K–€5M, > €5M), statuses (open, closing_soon, closed, awarded); source (aop, ted, eu_grants)
   - `ProposalFactory` — with versions, collaborators (multi-role), section locks, comments; includes `proposal_section_locks` for locking tests
   - `TaskFactory` — with DAG dependencies; includes templates per opportunity type
   - `ApprovalWorkflowFactory` — configurable stage sequences; includes decisions
   - Auto-cleanup fixtures using pytest's `yield` fixture pattern with `DELETE /api/test-data/cleanup` in teardown; parallel-safe using unique `company_id` per test
   - `ContentBlockFactory` — various categories, tags, approval states

2. **Test Environments** — QA/DevOps (Sprint 2)
   - **Local:** Docker Compose with all 5 services + PostgreSQL 16 + Redis 7 + MinIO + ClamAV; `.env.test` with `AI_GATEWAY_MOCK=true` + Stripe test keys + `DEBUG_AUTH=true` for JWT override in tests
   - **CI/CD:** GitHub Actions with service containers (PostgreSQL + Redis + MinIO); Stripe CLI webhook forwarding via `stripe listen --forward-to $APP_URL/api/v1/webhooks/stripe`; AI Gateway mock mode ON
   - **Staging:** Shared Kubernetes namespace; Stripe test mode + KraftData sandbox; smoke tests only (not full regression)

**Example factory + Playwright API test pattern:**

```python
# tests/conftest.py (pytest/backend)
import pytest
import httpx

@pytest.fixture
async def company_professional(test_client):
    """Create a Professional-tier company and clean up after test."""
    resp = await test_client.post('/api/test-data/seed', json={
        'company': {'tier': 'professional', 'tax_id': 'BG123456789'},
        'users': [{'role': 'bid_manager'}, {'role': 'technical_writer'}],
    })
    data = resp.json()
    yield data
    await test_client.delete(f"/api/test-data/cleanup/{data['company']['id']}")
```

```typescript
// tests/e2e/fixtures.ts (Playwright/E2E)
import { test, expect } from '@playwright/test';
import { faker } from '@faker-js/faker';

test('@P0 @API @Security cross-tenant isolation: company A cannot read company B data', async ({ request }) => {
  // Seed two companies
  const companyA = await request.post('/api/test-data/seed', { data: { company: { tier: 'professional' } } });
  const companyB = await request.post('/api/test-data/seed', { data: { company: { tier: 'professional' } } });

  const tokenA = (await companyA.json()).users[0].jwt;
  const proposalB_id = (await companyB.json()).proposals[0].id;

  // Company A's token should NOT access Company B's proposal
  const response = await request.get(`/api/v1/proposals/${proposalB_id}`, {
    headers: { Authorization: `Bearer ${tokenA}` }
  });
  expect(response.status()).toBe(404); // Not 403, to prevent enumeration
});
```

---

## Risk Assessment

**Note:** Full risk details and mitigation plans in Architecture doc (test-design-architecture.md).

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Score | QA Test Coverage |
|---------|----------|-------------|-------|-----------------|
| **R-001** | TECH | KraftData single point of failure (29 AI entities) | **9** | Mock mode: all 29 entity types respond deterministically; chaos test: KraftData unreachable → circuit opens → graceful 503 with cached result; no 500s permitted |
| **R-002** | SEC | Multi-tenant data isolation (shared PostgreSQL, RBAC) | **6** | Cross-tenant API sweep: Company A JWT cannot access Company B data at any of 10+ endpoints; DB role permission denial tests at PostgreSQL level |
| **R-003** | PERF | SSE stream saturation (AI Gateway concurrent connections) | **6** | k6 load test: 100 concurrent SSE + 100 concurrent REST → REST p95 < 200ms; per-tenant SSE limit (429 response on 4th connection) |
| **R-004** | BUS | Stripe billing complexity (trials, add-ons, EU VAT) | **6** | Full lifecycle: registration → trial → expiry → Free downgrade; add-on Checkout → webhook → feature unlock; all 5 webhook event types; B2B/B2C VAT; Enterprise invoicing |
| **R-006** | SEC | Entity-level RBAC (two-tier permission model edge cases) | **6** | Permission matrix: 5 roles × 2 entity types × 4 permission levels; cross-proposal isolation; grant/revoke atomicity; ceiling enforcement (read_only cannot be granted edit) |

### Medium/Low-Priority Risks

| Risk ID | Category | Description | Score | QA Test Coverage |
|---------|----------|-------------|-------|-----------------|
| R-005 | DATA | Data pipeline staleness | 4 | Verify stale-data UX indicator displayed when pipeline circuit breaker open; no 500s served |
| R-007 | TECH | Redis Streams DLQ gaps | 4 | Event flow tests: publish event to each of 7 streams → assert consumer group acknowledgement → verify downstream effect; DLQ depth = 0 after processing |
| R-008 | PERF | PostgreSQL shared instance bottleneck | 4 | k6: opportunity search with 10K+ seeded records; FTS p95 < 200ms; concurrent analytics + search → no contention; materialized view refresh within window |
| R-009 | OPS | Calendar sync OAuth token lifecycle | 4 | OAuth mock: token refresh cycle (access token expires → refresh token used → new access token); revocation: token revoked → sync stops gracefully; no orphaned sync jobs |
| R-010 | SEC | ClamAV file scanning | 3 | Upload EICAR test file → expect 422 (malware detected); upload clean PDF → expect 201; ClamAV health-check passes before upload tests begin |
| R-012 | BUS | Tier gating accuracy | 4 | Parametrized: Free (name/deadline/location/type/status only) vs. Starter/Professional/Enterprise (full data, AI features, region limits); usage meter limits enforced per tier |
| R-015 | SEC | Proposal section locking race | 4 | Concurrent lock acquisition (2 users, same section, same instant) → only 1 acquires lock; TTL boundary test (lock expires → section becomes available); 423 response with lock holder identity |
| **R-016** ★NEW | TECH | Frontend locale middleware routing (BG/EN) | 4 | Playwright: all platform routes accessible in both locales; cookie-based locale override; Accept-Language fallback; no redirect loops; deep links resolve to correct locale |
| **R-017** ★NEW | SEC | JWT refresh race condition at frontend | 4 | Playwright: open 2 browser tabs → both make authenticated requests → single token refresh triggered (not two); no user logout; both requests complete successfully after refresh |
| R-011 | DATA | Document retention lifecycle | 2 | Integration test: create document → advance time past retention window → verify soft-delete → advance 30 days → verify S3 purge; audit log immutable throughout |
| R-013 | TECH | i18n coverage (BG + EN) | 2 | CI: i18n key coverage check (no missing keys in either locale); smoke test in BG locale (key pages render without missing-key errors); AI output language matches user preference setting |
| R-014 | OPS | White-label subdomain isolation | 2 | Enterprise tenant white-label subdomain: correct logo/brand/colors rendered; no bleed from other tenants' white-label config |

---

## Entry Criteria

- [ ] All requirements and acceptance criteria agreed upon by QA, Dev, PM
- [ ] Test environments provisioned (Docker Compose local + CI namespace + staging)
- [ ] Test data factories implemented (User, Company, Opportunity, Proposal, Task, ApprovalWorkflow, ContentBlock)
- [ ] TB-01 (Seeding API) delivered and functional
- [ ] TB-02 (KraftData mock) operational for all 29 entity types (or Playwright `route()` stub fallback)
- [ ] TB-03 (Stripe test mode) configured with webhook forwarding
- [ ] Docker Compose test profile starts all 5 services + PostgreSQL + Redis + MinIO + ClamAV
- [ ] GitHub Actions CI matrix runs all 5 service tests in parallel

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95%, or failures triaged and accepted by QA Lead + PM)
- [ ] No open P0/P1 severity bugs (or all accepted as known risk with owner)
- [ ] Cross-tenant isolation validated (0 cross-company data leaks)
- [ ] Performance baselines met: REST p95 < 200ms, SSE TTFB < 500ms, FTS on 10K+ records p95 < 200ms
- [ ] All 5 Stripe webhook event types tested and handlers verified
- [ ] Entity RBAC permission matrix coverage: all 5 roles × 2 entity types × 4 permission levels

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on when time-constrained), NOT execution timing. See "Execution Strategy" for when tests run.

### P0 (Critical)

**Criteria:** Blocks core functionality + High risk (≥6) + No workaround + Affects majority of users

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **P0-001** | User registration (email/password) | API | R-002 | Validate: company created, Professional trial subscription initialised, `trial_start` set, JWT RS256 issued |
| **P0-002** | User login + JWT issuance | API | R-002 | Valid credentials → 200 + JWT; invalid password → 401; revoked token → 401 |
| **P0-003** | JWT verification across services | API | R-002 | Valid JWT on Client API, Admin API (with platform_admin role); expired JWT → 401; tampered JWT → 401 |
| **P0-004** | Google OAuth2 social login | API | R-002 | OAuth2 flow mock (authlib stub): new user → company created + trial; existing user → login; state CSRF validated |
| **P0-005** | Subscription creation via Stripe Checkout | API | R-004 | Checkout Session created → `checkout.session.completed` webhook → subscription active; tier updated |
| **P0-006** | Trial lifecycle: Professional → Free downgrade on expiry | API | R-004 | Stripe test clock: registration → 14-day trial → advance clock past expiry → `customer.subscription.updated` (past_due) → Client API webhook handler → tier = Free; features gated |
| **P0-007** | Per-bid add-on purchase | API | R-004 | Checkout Session (mode=payment) → `checkout.session.completed` → `add_on_purchases` record created → feature unlocked for specific opportunity (not others) |
| **P0-008** | Cross-tenant data isolation: opportunities | API | R-002 | Company A JWT cannot retrieve Company B opportunities, proposals, tasks, analytics, documents at any endpoint; response = 404 (not 403, prevent enumeration) |
| **P0-009** | Cross-tenant data isolation: proposals | API | R-002 | As above for proposals; entity permission grants are company-scoped; no cross-company leakage |
| **P0-010** | Admin API VPN/IP restriction | API | R-002 | Request from non-VPN IP → 403/blocked at ingress; `platform_admin` role required; regular JWT rejected |
| **P0-011** | Entity-level RBAC: bid_manager on Proposal A, read_only on Proposal B | API | R-006 | bid_manager actions (edit, assign collaborators) succeed on A; blocked on B; no role bleed |
| **P0-012** | Entity RBAC: company role ceiling enforcement | API | R-006 | User with `read_only` company role cannot be granted `edit` entity permission; API returns 422 |
| **P0-013** | AI Gateway: circuit breaker opens after 5 failures | API | R-001 | Mock 5 consecutive KraftData failures → circuit opens → subsequent requests return 503 with graceful message; after 30s retry → circuit closes |
| **P0-014** | Proposal generation via KraftData (mock mode) | API | R-001 | `POST /workflows/proposal-generator/run-stream` with `AI_GATEWAY_MOCK=true` → deterministic SSE stream; all chunks received; final result stored |
| **P0-015** | AI Gateway: graceful degradation UX when circuit open | E2E | R-001 | With circuit open: AI summary button shows "AI features temporarily unavailable"; cached result displayed if available; no 500 error page |
| **P0-016** | Tier gating: Free user gets limited opportunity view | API | R-012 | Free JWT → opportunity detail returns `OpportunityFreeResponse` (name, deadline, location, type, status only); `OpportunityFullResponse` fields absent |
| **P0-017** | Tier gating: Starter user gets full BG opportunity + AI summary | API | R-012 | Starter JWT → full BG opportunity detail; AI summary generation returns usage meter INCR; over-limit → 429 with upgrade prompt |
| **P0-018** | All 5 service health checks pass | API | R-001 | `GET /health` returns 200 on all 5 services (client-api, admin-api, ai-gateway, data-pipeline, notification); DB + Redis connectivity verified |
| **P0-019** | DB role isolation: service cannot write to another service's schema | API | R-002 | As `client_api_role`: INSERT into `pipeline` schema → permission denied; as `pipeline_role`: INSERT into `client` schema → permission denied; all 5 × non-owned schema combinations |
| **P0-020** | Usage metering: atomic INCR + limit enforcement | API | R-012 | Increment usage counter to limit → 429 on next call; DECR on failed request (no phantom usage); concurrent INCR is atomic (no double-count) |

**Total P0:** ~20 tests

---

### P1 (High)

**Criteria:** Important features + Medium/high risk + Common workflows + Workaround exists but costly

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **P1-001** | Opportunity search: filter by CPV, region, budget, deadline | API | R-008, R-012 | Multiple filter combinations; FTS on title/description; pagination correct; tier-gated field visibility per filter result |
| **P1-002** | Smart matching: relevance score computed against company profile | API | R-001 | Mock Relevance Scoring Agent → deterministic score returned; score displayed on listing and detail |
| **P1-003** | Document upload: PDF/DOCX accepted; ZIP accepted; file >100MB rejected | API | — | 201 on valid upload; S3 storage confirmed; ClamAV scan triggered; 413 on oversized file |
| **P1-004** | Document upload: ClamAV malware detection | API | R-010 | Upload EICAR test file → 422 with malware error; upload clean PDF → 201 |
| **P1-005** | Alert preferences: create/update/delete preference | API | — | CRUD operations on alert preferences; preference stored; `alert_preference.updated` event published to Redis Streams |
| **P1-006** | Alert digest: email sent matching user preferences | Integration | R-007 | Seed opportunity matching user CPV/region preferences → publish `opportunities.ingested` event → Notification Service consumes → `generate_alerts` task runs → SendGrid mock called with correct recipient + matching opportunity |
| **P1-007** | Proposal collaboration: assign bid_manager role to user | API | R-006 | `proposal_collaborators` record created; JWT of assigned user now has bid_manager access; grantor identity logged |
| **P1-008** | Proposal section locking: acquire, display, TTL expiry | API | R-015 | User A acquires lock on section X; User B attempts edit → 423 with lock holder name; lock expires after 15 min inactivity → section available |
| **P1-009** | Proposal section locking: concurrent acquisition | API | R-015 | Simultaneous lock requests from User A and User B → exactly 1 succeeds; 423 returned to the other; no double-lock |
| **P1-010** | Task orchestration: create task with DAG dependencies | API | — | Task with `finish_to_start` dependency on another task; dependent task cannot move to `in_progress` until prerequisite is `completed`; transition blocked returns 422 |
| **P1-011** | Task assignment notification via Redis Streams | Integration | R-007 | Assign task → `task.assigned` event published → Notification Service consumes → email sent to assignee |
| **P1-012** | Approval workflow: multi-stage sign-off sequence | API | — | Configure 3-stage workflow (Technical → Legal → Management); stage 1 approved → stage 2 unlocked; stage 2 rejected → proposal returned to editor; decision logged in `approval_decisions` |
| **P1-013** | iCal feed: deadlines exported for tracked opportunities | API | — | `GET /api/v1/calendar/ical` returns valid iCal format; each tracked opportunity appears as event with correct deadline |
| **P1-014** | Google Calendar sync: connect + first sync | Integration | R-009 | OAuth mock: Google OAuth2 flow → `calendar.connected` event → Notification Svc consumes → `sync_calendars` task → Google Calendar API mock called with correct events; `calendar_events` records created |
| **P1-015** | Google Calendar sync: token refresh + revocation | Integration | R-009 | Access token expires → Celery Beat task uses refresh token → new access token obtained; revoke refresh token → sync stops; no orphaned sync jobs |
| **P1-016** | Redis Streams event delivery: `opportunities.ingested` end-to-end | Integration | R-007 | Data Pipeline publishes `opportunities.ingested` → `notification-svc` consumer group ACKs; `client-api` consumer group invalidates opportunity cache; verify both groups process event; DLQ depth = 0 |
| **P1-017** | Redis Streams event delivery: `subscription.changed` tier cache | Integration | R-007 | Client API publishes `subscription.changed` → all services consuming `eu-solicit:subscriptions` receive and update tier cache; verify tier gate enforces new tier on next request |
| **P1-018** | Stripe EU VAT: B2B reverse charge | API | R-004 | Company with valid EU VAT number (BG123456789) → Stripe Tax → 0% VAT applied; invoice includes reverse charge note "Article 196 Council Directive 2006/112/EC" |
| **P1-019** | Stripe EU VAT: B2C with local VAT rate | API | R-004 | Company without VAT number in EU country → Stripe Tax → local VAT rate applied; invoice includes gross + net amounts + VAT amount |
| **P1-020** | Trial: one trial per company (second registration blocked) | API | R-004 | Register user under Company A (trial started); register second user under Company A → trial NOT started again; `subscriptions.trial_start IS NOT NULL` gate enforced |
| **P1-021** | Compliance framework: CRUD + assignment to opportunity | API | — | Admin creates framework; assigns to opportunity; Client API applies framework validation; `compliance_framework.assigned` event published → Data Pipeline + AI Gateway consume |
| **P1-022** | ESPD auto-fill: profile management + form generation | API | R-001 | ESPD profile CRUD; mock ESPD Auto-Fill Agent → structured form returned; company profile data mapped to ESPD sections |
| **P1-023** | Content blocks: CRUD + approval + tagging | API | — | Create content block with category + tags; submit for approval; approve → `is_approved = true`; block available for inclusion in Proposal Generator via RAG |
| **P1-024** | Frontend locale routing: all platform routes in BG and EN | E2E | R-016 | Playwright: navigate all primary routes in `/bg/` and `/en/`; no 404s; cookie-based locale switch; Accept-Language fallback |
| **P1-025** | Frontend locale routing: deep link to BG locale resolves correctly | E2E | R-016 | Direct URL to `/bg/opportunities/[id]` → correct BG locale rendered; no redirect loop; title/nav in Bulgarian |
| **P1-026** | JWT refresh race: concurrent tabs issue single refresh | E2E | R-017 | Playwright: 2 browser tabs; expire access token; both tabs make authenticated requests → single refresh call in network log; both requests complete after refresh; user stays logged in |
| **P1-027** | Submission guides: auto-generated + displayed on opportunity detail | API | — | Data Pipeline ingests opportunity → Submission Guide Agent mock → `submission_guides` record created; Client API serves read-only to user on opportunity detail |

**Total P1:** ~27 tests

---

### P2 (Medium)

**Criteria:** Secondary features + Low risk (1–2) + Edge cases + Regression prevention

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **P2-001** | Opportunity search: no results → empty state displayed | E2E | — | Search with zero-match CPV code → empty state component shown, no error |
| **P2-002** | Opportunity search: FTS performance with 10K+ records | API | R-008 | k6 spike test: 50 concurrent searches → p95 < 200ms; full-text search index utilised (EXPLAIN confirms) |
| **P2-003** | Document export: proposal as PDF | API | — | `POST /api/v1/proposals/{id}/export?format=pdf` → 200 with PDF content-type; `python-docx + reportlab` pipeline executed |
| **P2-004** | Document export: proposal as DOCX | API | — | Same flow, `format=docx`; DOCX file headers correct; section content present |
| **P2-005** | Bid/no-bid decision: AI scorecard + manual override | API | R-001 | Mock Bid/No-Bid Decision Agent → scorecard returned; user overrides AI recommendation; override logged in `bid_decisions` |
| **P2-006** | Bid outcome tracking: won/lost/withdrawn + evaluator feedback | API | — | Record outcome; feedback stored; outcome feeds ROI tracker materialized view on next refresh |
| **P2-007** | Preparation time logging for ROI tracker | API | — | Log hours + cost per proposal; aggregate visible in ROI tracker dashboard API endpoint |
| **P2-008** | Analytics: materialized view refresh (ROI tracker) | Integration | R-008 | Celery Beat `refresh_materialized_views` task runs; `roi_tracker` view updated; API returns fresh data after refresh |
| **P2-009** | Analytics: usage dashboard (Redis → materialized view) | Integration | — | Usage INCR in Redis; hourly materialized view refresh; usage dashboard API returns current period totals |
| **P2-010** | Report generation: on-demand PDF report | API | — | `POST /api/v1/analytics/reports` → Notification Svc `generate_reports` task → PDF generated; download URL returned |
| **P2-011** | Report generation: scheduled weekly report | Integration | — | Celery Beat triggers weekly report generation; SendGrid mock called with attached PDF report |
| **P2-012** | Admin: tenant management — view and manage company subscription | API | — | `GET /admin/api/v1/tenants` → company list with tier + usage; admin can force-downgrade a tenant's tier |
| **P2-013** | Admin: crawler management — view history + trigger manual run | API | — | `GET /admin/api/v1/crawlers` → run history; `POST /admin/api/v1/crawlers/aop/trigger` → KraftData mock AOP Crawler Agent invoked |
| **P2-014** | Admin: audit log viewer — searchable by user/action/entity | API | — | Audit log entries created for all P0 actions; `GET /admin/api/v1/audit?user_id=X` → filtered results; audit log is append-only (no DELETE possible) |
| **P2-015** | Compliance: framework auto-suggestion based on opportunity metadata | API | R-001 | Mock Framework Suggestion Agent → suggestion returned for opportunity (country + funding type); admin confirms or overrides |
| **P2-016** | ESPD profile: multiple profiles per company (consortium) | API | — | Create two ESPD profiles per company (Default + Joint Venture); select correct profile for ESPD auto-fill submission |
| **P2-017** | Company profile: `tax_id` stored + synced to Stripe Customer | API | R-004 | Update `companies.tax_id` → Stripe Customer `customer.tax_ids` updated via Stripe API; next invoice includes VAT ID |
| **P2-018** | White-label: subdomain + logo + brand colours for Enterprise tenant | E2E | R-014 | `GET https://client-name.eusolicit.com` → frontend renders with client-name logo, brand colours; no EU Solicit branding; other tenant's white-label config not visible |
| **P2-019** | Document retention: soft-delete + S3 purge lifecycle | Integration | R-011 | Create document; advance time past retention window (2 years after opportunity close); Celery Beat task soft-deletes; advance 30 days → S3 purge triggered; audit log entry immutable throughout |
| **P2-020** | Calendar sync: Outlook (Microsoft Graph API) connect + sync | Integration | R-009 | Same as P1-014 but for Microsoft Graph API; `microsoft_graph.py` client called; OAuth mock for Microsoft |
| **P2-021** | Calendar: iCal feed updates when opportunity status changes | API | — | Opportunity status changes → iCal feed regenerated; calendar client re-fetches → updated event |
| **P2-022** | Proposal comments: threaded + version-scoped + carry-forward | API | — | Add comment to Section A, Version 1; create Version 2 → unresolved comment carries forward to Version 2; resolved comment archived |
| **P2-023** | Task template: create template + apply to new opportunity | API | — | Create task template (tender type, relative deadlines); apply to opportunity with known submission deadline → tasks created with computed absolute deadlines |
| **P2-024** | Approval notification: sent on stage request + decision | Integration | R-007 | Approval requested → `approval.requested` event → Notification Svc → email to required-role approver; decision logged → `approval.decided` event → email to proposal team |
| **P2-025** | Stripe Customer Portal: upgrade, downgrade, cancel, update payment method | E2E | R-004 | Navigate to Customer Portal via Stripe-generated URL; verify self-service flows render correctly (Stripe-hosted, so UI smoke test only; API-level subscription state verified) |
| **P2-026** | Usage gate: 429 on limit exceeded + upgrade prompt | API | R-012 | Increment AI summary counter to Starter limit (e.g., 20/month) → 21st request → 429 with `{ "upgrade_required": true, "tier": "starter", "limit": 20 }` |
| **P2-027** | Enterprise API: OpenAPI spec auto-generated + accessible | API | — | `GET /api/v1/openapi.json` returns valid OpenAPI 3.x spec; Enterprise JWT required; endpoints documented match implemented routes |
| **P2-028** | Competitor tracking: publicly available award data aggregated | API | R-001 | Mock Competitor Analysis Agent → competitor records ingested; `GET /api/v1/analytics/competitor` returns aggregated data for Professional+ tier |
| **P2-029** | Pipeline forecasting: predicted tenders on dashboard | API | R-001 | Mock Pipeline Forecasting Agent → forecast data stored; `GET /api/v1/analytics/forecast` returns Professional+ tier forecast |
| **P2-030** | Proposal version control: diff capability + rollback | API | — | Create v1 → v2 with changes; `GET /api/v1/proposals/{id}/versions/{v1_id}/diff/{v2_id}` → structured diff; rollback to v1 → proposal reverts |

**Total P2:** ~30 tests

---

### P3 (Low)

**Criteria:** Nice-to-have + Exploratory + Performance benchmarks + Documentation validation

| Test ID | Requirement | Test Level | Notes |
|---------|-------------|------------|-------|
| **P3-001** | E2E journey: Free Explorer (Maria) — browse tenders, see upgrade prompt | E2E | Full persona flow: register → free tier → browse opportunities (limited view) → AI summary blocked → upgrade prompt displayed |
| **P3-002** | E2E journey: Starter User (Georgi) — monitor BG tenders, use AI summary | E2E | Register → trial → add alert for BG CPV → opportunity appears in listing → AI summary generated (mock) → alert email triggered |
| **P3-003** | E2E journey: Professional User (Elena) — full proposal lifecycle | E2E | Professional tier → discover opportunity → AI summary → create proposal → assign collaborator → task DAG → bid/no-bid → generate proposal (SSE mock) → export PDF |
| **P3-004** | E2E journey: Enterprise User (Nikolai) — white-label + Enterprise API | E2E | Enterprise tier → white-label config → access via subdomain → Enterprise API token → API returns full data + OpenAPI spec |
| **P3-005** | E2E journey: Platform Admin — manage compliance + tenants | E2E | Admin login (VPN mock) → create compliance framework → assign to opportunity → view tenant list → view audit log |
| **P3-006** | k6 performance: REST API p95 latency under sustained load | Performance | 50 concurrent users for 5 min; `GET /api/v1/opportunities` p95 < 200ms; `GET /api/v1/proposals/{id}` p95 < 200ms |
| **P3-007** | k6 performance: SSE TTFB under concurrent load | Performance | 20 concurrent SSE streams (Proposal Generator mock); TTFB p95 < 500ms; stream completes without timeout |
| **P3-008** | k6 performance: opportunity FTS with 10K+ records | Performance | Seed 10K opportunities; FTS search p95 < 200ms; 50 concurrent searches; PostgreSQL index confirmed |
| **P3-009** | k6 performance: Stripe webhook processing throughput | Performance | Simulate 100 concurrent `checkout.session.completed` webhooks; all processed; no queue backup; p95 < 200ms handler time |
| **P3-010** | i18n: BG locale — all primary pages render without missing-key errors | E2E | Playwright: navigate all primary routes with BG locale; no `[missing translation: ...]` rendered; no console errors for missing keys |
| **P3-011** | i18n: AI output language matches user preference | API | Set user preference to Bulgarian → Executive Summary Agent mock returns BG output; set to English → EN output |
| **P3-012** | i18n: BG locale — all form validation messages in Bulgarian | E2E | Submit forms with invalid data while BG locale active; error messages appear in Bulgarian |
| **P3-013** | Chaos: Data Pipeline down — Client API serves existing data | Chaos | Stop Data Pipeline service; `GET /api/v1/opportunities` continues to serve (stale but not 503); stale-data indicator visible in UI |
| **P3-014** | Chaos: Notification Service down — alert preferences still saved | Chaos | Stop Notification Service; update alert preferences → `alert_preference.updated` event published to Redis Streams → event sits in stream until service recovers → on restart, events processed; no events lost (DLQ validates) |
| **P3-015** | White-label subdomain: brand isolation between Enterprise tenants | E2E | Enterprise Tenant A has logo A; Enterprise Tenant B has logo B; `client-a.eusolicit.com` shows only logo A; `client-b.eusolicit.com` shows only logo B |

**Total P3:** ~15 tests

---

## Execution Strategy

**Philosophy:** Run all functional tests in PRs via Playwright + pytest parallelization. Defer only infrastructure-expensive tests to nightly/weekly.

### Every PR: Playwright + pytest Tests (~10–15 min)

**All functional tests** (P0 + P1 + P2 functional):

- Playwright API tests: ~65 tests parallelized across 4 shards
- pytest backend unit/integration: ~20 tests per service in parallel matrix (5 services)
- Total: ~85 tests in ~10–15 min
- Tags: `@P0 @API @Security @Integration` available for selective re-run

```bash
# Run P0 only (security gate):
npx playwright test --grep @P0

# Run P0 + P1 (standard PR gate):
npx playwright test --grep "@P0|@P1"

# Run all Playwright tests:
npx playwright test

# Backend pytest:
pytest services/client-api/tests/ services/admin-api/tests/ -n auto
```

### Nightly: k6 Performance Tests (~30–45 min)

**All performance tests** (P3-006 through P3-009):

- k6 scenarios: opportunity FTS (10K records), REST load, SSE concurrent, Stripe webhook throughput
- Runs against staging environment with Stripe test mode

### Weekly: Chaos + Long-Running (~2–4 hours)

**Special infrastructure tests** (P3-013, P3-014):

- Data Pipeline failure: requires K8s pod kill, not available in Docker Compose
- Notification Service recovery: requires DLQ drain validation after service restart
- Runs against staging Kubernetes namespace

---

## QA Effort Estimate

**QA test development effort** (excludes DevOps, Backend infrastructure, Stripe/KraftData integration setup):

| Priority | Count | Effort Range | Notes |
|----------|-------|--------------|-------|
| P0 | ~20 | ~25–40 hours | Complex security + billing + circuit-breaker scenarios; multi-step setup |
| P1 | ~27 | ~30–50 hours | Integration flows (Redis Streams, OAuth); concurrent scenarios; locale/token tests |
| P2 | ~30 | ~25–45 hours | Edge cases, secondary admin features, calendar/retention/export |
| P3 | ~15 | ~10–20 hours | Persona journeys + performance configs + chaos orchestration |
| **Total** | **~92** | **~90–155 hours** (~6–9 weeks, 1 QA) | Includes design, implementation, CI integration, debugging |

**Assumptions:**
- Includes test design, implementation, debugging, CI pipeline integration
- Excludes ongoing maintenance (~10% ongoing)
- Assumes test infrastructure (factories, seeding API, KraftData mock) ready by Sprint 3
- Assumes 2 QAs: reduce to ~3–5 weeks (parallel P0+P1 / P2+P3 split)

---

## Implementation Planning Handoff

| Work Item | Owner | Target Sprint | Dependencies/Notes |
|-----------|-------|--------------|-------------------|
| Test data seeding API (TB-01) | Backend Lead | Sprint 3 | Blocks all integration + E2E tests; must support all entity types |
| KraftData mock service (TB-02) | Backend Lead | Sprint 3 | Blocks all AI feature tests; must cover all 29 entity types |
| Stripe test mode + webhook CLI (TB-03) | Backend Lead | Sprint 4 | Blocks all billing tests; Stripe CLI must run in CI |
| QA test data factories | QA | Sprint 3 | Required for parallel-safe test isolation |
| Redis Streams DLQ | Backend Lead | Sprint 5 | Required for event flow tests (P1-016, P1-017) |
| Playwright route() KraftData stubs | QA | Sprint 3 | Contingency if TB-02 delayed; covers SSE mock at HTTP level |
| CI matrix: 5-service parallel pytest | DevOps/QA | Sprint 2 | Required for PR gate; each service tested independently |
| k6 staging pipeline | DevOps/QA | Sprint 8 | Required for performance tests (P3-006 through P3-009) |

---

## Tooling & Access

| Tool or Service | Purpose | Access Required | Status |
|----------------|---------|----------------|--------|
| Playwright (latest) | E2E + API tests | npm install | Ready (E01/E02/E03 tests existing) |
| pytest + pytest-asyncio | Backend unit/integration | pip install | Ready (existing test infra) |
| k6 | Performance testing | k6 binary in CI | Pending (staging pipeline) |
| Stripe CLI | Webhook simulation in CI | `stripe listen` binary | Pending (TB-03) |
| faker (JS + Python) | Test data generation | npm/pip install | Ready |
| SendGrid sandbox | Email delivery testing | SendGrid account flag | Pending |
| Google OAuth2 test app | Calendar sync testing | Google Cloud Console | Pending |
| Microsoft Graph test app | Outlook sync testing | Azure AD Console | Pending |

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope | Validation Steps |
|------------------|--------|-----------------|-----------------|
| **Client API** | Core revenue surface; all user-facing features | P0-001 through P0-020 + P1-001 through P1-027 | Full PR gate suite passes; no regression on auth/billing/RBAC |
| **Admin API** | Platform operations; VPN-restricted | P0-010 + P2-012 through P2-015 | Admin auth + CRUD operations pass; VPN restriction enforced |
| **AI Gateway** | All AI features; KraftData integration | P0-013 through P0-015 + P1-022 | Circuit breaker opens/closes correctly; mock mode deterministic |
| **Data Pipeline** | Opportunity data freshness; crawl orchestration | P1-016 + P2-013 | `opportunities.ingested` event flow intact; no new opportunities lost |
| **Notification Service** | Alerts, calendar sync, billing notifications | P1-006 + P1-011 + P1-014 | All outbound delivery paths intact; Redis Streams consumer groups ACK events |
| **eusolicit-common** | Auth + middleware shared across all services | P0-002 + P0-003 | JWT verification consistent across all 5 services |
| **eusolicit-models** | DTO contracts between services | P1-016 + P1-017 | Event schema changes do not break cross-service consumers |

**Regression test strategy:**
- Epic completion gate: P0 + P1 tests for impacted services must all pass before epic sign-off
- Schema changes: run DB role isolation tests (P0-019) on every Alembic migration
- KraftData agent registry changes: re-run all P0/P1 AI tests with updated mock fixtures
- Stripe webhook handler changes: re-run P0-005 through P0-007 + P2-017 + P2-025

---

## Appendix A: Code Examples & Tagging

**Playwright test tag conventions:**

```typescript
// P0 cross-tenant isolation test
test('@P0 @API @Security Company A cannot access Company B proposal', async ({ request }) => {
  const seedA = await request.post('/api/test-data/seed', { data: { company: { tier: 'professional' }, proposals: [{}] } });
  const seedB = await request.post('/api/test-data/seed', { data: { company: { tier: 'professional' }, proposals: [{}] } });
  const { jwt: tokenA } = (await seedA.json()).users[0];
  const { id: proposalB } = (await seedB.json()).proposals[0];

  const response = await request.get(`/api/v1/proposals/${proposalB}`, {
    headers: { Authorization: `Bearer ${tokenA}` }
  });
  expect(response.status()).toBe(404);

  // Cleanup
  await request.delete(`/api/test-data/cleanup/${(await seedA.json()).company.id}`);
  await request.delete(`/api/test-data/cleanup/${(await seedB.json()).company.id}`);
});

// P1 Redis Streams event delivery test
test('@P1 @Integration opportunities.ingested event triggers alert matching', async ({ request }) => {
  // Seed user with alert preference matching AOP CPV 72000000
  const { jwt, alert_preference_id } = (await (await request.post('/api/test-data/seed', {
    data: { company: { tier: 'starter' }, alert_preferences: [{ cpv_codes: ['72000000'], regions: ['BG'] }] }
  })).json()).users[0];

  // Trigger ingestion event (mock Data Pipeline → Redis Streams)
  await request.post('/api/test-data/trigger-event', { data: {
    stream: 'eu-solicit:opportunities',
    event: 'opportunities.ingested',
    payload: { opportunity_ids: ['test-opp-1'], source: 'aop' }
  }});

  // Poll for alert log entry (max 5s)
  await expect.poll(async () => {
    const r = await request.get(`/api/v1/alerts/log?preference_id=${alert_preference_id}`, {
      headers: { Authorization: `Bearer ${jwt}` }
    });
    return (await r.json()).items.length;
  }, { timeout: 5000 }).toBeGreaterThan(0);
});
```

**Run commands:**

```bash
# P0 security gate (CI):
npx playwright test --grep @P0 --reporter=github

# P0 + P1 PR gate (full):
npx playwright test --grep "@P0|@P1" --workers=4

# All tests:
npx playwright test --workers=4

# Backend service tests (parallel):
pytest services/client-api/tests/ -n auto --dist=worksteal
pytest services/admin-api/tests/ -n auto
pytest services/ai-gateway/tests/ -n auto

# Performance (nightly):
k6 run tests/performance/opportunity-search.js --vus=50 --duration=5m
```

---

## Appendix B: Knowledge Base References

- **Risk Governance**: `risk-governance.md` — P×I scoring methodology; gate decisions; mitigation tracking
- **Test Priorities Matrix**: `test-priorities-matrix.md` — P0–P3 criteria and classification rules
- **Test Levels Framework**: `test-levels-framework.md` — E2E vs API vs Unit selection decision matrix
- **Test Quality**: `test-quality.md` — Definition of Done (no hard waits, <300 lines per test file, <1.5 min per test)
- **Architecture Doc**: `test-design-architecture.md` — Risk mitigation plans, testability gaps, blocker details
- **Handoff Doc**: `test-design/eu-solicit-handoff.md` — BMAD epic/story integration guidance

---

**Generated by:** BMad TEA Agent  
**Workflow:** `bmad-testarch-test-design`  
**Version:** v5 (2026-04-09) — system-level regeneration with full Architecture v4 context
