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

# Test Design for Architecture: EU Solicit

**Purpose:** Architectural concerns, testability gaps, and NFR requirements for review by Architecture/Dev teams. Serves as a contract between QA and Engineering on what must be addressed before test development begins.

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**Status:** Architecture Review Pending
**Project:** EU Solicit
**PRD Reference:** EU_Solicit_PRD_v1.md
**ADR Reference:** EU_Solicit_Solution_Architecture_v4.md (ADRs 1.1–1.15)

---

## Executive Summary

**Scope:** Full platform — 5 FastAPI/Celery services, Next.js 14 frontend, PostgreSQL 16 (6 schemas), Redis 7 (7 streams, 4 consumer groups), KraftData Agentic AI (22 agents/teams/workflows, 7 vector stores), Stripe billing (trials, add-ons, EU VAT), Google Calendar + Outlook sync, data pipeline crawlers.

**Business Context** (from PRD):

- **Revenue/Impact:** SaaS platform with 4 tiers (Free → Enterprise); revenue from Starter (€29), Professional (€59), Enterprise (€109+), and per-bid add-ons
- **Target milestones:** Demo at Sprint 8 / Week 16, Beta at Sprint 12 / Week 24, MVP Launch at Sprint 14 / Week 28

**Architecture** (ADR v4):

- **ADR 1.2:** Hybrid SOA — 5 services (Client API, Admin API, Data Pipeline, AI Gateway, Notification Service)
- **ADR 1.3:** REST + Redis Streams async events; 7 named streams, 4 consumer groups
- **ADR 1.5:** Single PostgreSQL instance, 6 schemas, per-service DB roles with enforced access boundaries
- **ADR 1.7:** All AI capabilities delegated to KraftData Agentic AI platform

**Risk Summary:**

- **Total risks:** 15 (5 high-priority ≥6, 6 medium, 4 low)
- **Test effort:** ~88 scenarios (~6–9 weeks for 1 QA, ~3–5 weeks for 2 QAs)

---

## Quick Guide

### 🚨 BLOCKERS — Team Must Decide (Can't Proceed Without)

**Pre-Implementation Critical Path** — Must be completed before QA can write integration tests:

1. **TB-01: Test Data Seeding API** — `POST /api/test-data/seed` and `DELETE /api/test-data/cleanup` endpoints (dev/staging only) supporting: users, companies, subscriptions (all tiers + trial states), opportunities, proposals, tasks, and approval workflows. Owner: Backend Lead. Required by: Sprint 3.
2. **TB-02: KraftData Mock Service** — AI Gateway mock mode returning deterministic fixture responses for all 22 agent types. Without this, every AI feature test requires live KraftData connectivity. Owner: Backend Lead. Required by: Sprint 3.
3. **TB-03: Stripe Test Mode Configuration** — Stripe test-mode keys, webhook simulation (Stripe CLI), and test clocks for trial lifecycle, add-on purchases, and subscription lifecycle. Owner: Backend Lead. Required by: Sprint 4.

### ⚠️ HIGH PRIORITY — Team Should Validate

1. **R-002: Multi-tenant data isolation** — Validate schema-level + entity-level RBAC enforces strict cross-company isolation. Recommend automated isolation tests at the DB role level.
2. **R-003: SSE stream saturation** — Validate AI Gateway can handle concurrent SSE connections without starving the REST pool. Recommend separate connection pool + HPA metric on active SSE count.
3. **R-006: Entity-level RBAC enforcement** — Validate the two-tier permission model (company role ceiling + entity permission) handles all edge cases (e.g., bid manager on Proposal A but read-only on Proposal B).

### 📋 INFO ONLY — Solutions Provided

- **Test strategy:** API-heavy (60% API, 25% E2E, 10% Unit, 5% Performance) — API-first architecture makes API testing the highest-value level
- **Tooling:** Playwright (E2E + API), pytest (unit/integration), k6 (performance)
- **CI/CD cadence:** Every PR (~10–15 min Playwright), Nightly (k6 performance), Weekly (chaos/DR)
- **Coverage:** ~88 test scenarios P0–P3; full plan in companion QA doc (test-design-qa.md)

---

## For Architects and Devs — Open Topics 👷

### Risk Assessment

**Total risks identified:** 15 (5 high-priority ≥6, 6 medium, 4 low)

#### High-Priority Risks (Score ≥6) — IMMEDIATE ATTENTION

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **R-001** | TECH | KraftData single point of failure — all AI features (22 agents) depend on one external platform | 3 | 3 | **9** | Circuit breaker + graceful degradation UX + deterministic mock for testing | Backend Lead | Pre-implementation |
| **R-002** | SEC | Multi-tenant data isolation — shared PostgreSQL with cross-schema reads, entity-level RBAC complexity | 2 | 3 | **6** | DB role enforcement tests + automated isolation validation per schema boundary | Architect | Pre-implementation |
| **R-003** | PERF | SSE stream saturation — AI Gateway proxies long-lived connections; starvation risk under concurrent load | 2 | 3 | **6** | Connection pool isolation + HPA based on active SSE count metric | Backend Lead | Implementation |
| **R-004** | BUS | Stripe billing complexity — trials (no CC), per-bid add-ons, EU VAT (reverse charge), usage metering | 2 | 3 | **6** | Stripe test mode + clock testing + webhook simulation for all lifecycle events | Backend Lead | Implementation |
| **R-006** | SEC | Entity-level RBAC — two-tier permission model (company role ceiling + entity permission) edge cases | 2 | 3 | **6** | Exhaustive permission matrix: all 5 entity roles × all entity types × all permission levels | Backend | Implementation |

#### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| R-005 | DATA | Data pipeline staleness — crawlers depend on KraftData; no local fallback | 2 | 2 | 4 | AI Gateway circuit breaker (existing data serves clients) | Backend |
| R-007 | TECH | Redis Streams delivery — no dead letter handling documented; 7 streams, 4 consumer groups | 2 | 2 | 4 | Add dead letter stream + monitoring per consumer group | Backend |
| R-008 | PERF | PostgreSQL shared instance — 6 schemas, FTS on 10K+ opportunities | 2 | 2 | 4 | Index optimization + connection pool tuning | DBA/Backend |
| R-009 | OPS | Calendar sync OAuth — token refresh/revocation handling for Google + Microsoft | 2 | 2 | 4 | OAuth mock + token lifecycle tests | Backend |
| R-012 | BUS | Tier gating accuracy — complex region/CPV/budget access matrix across 4 tiers | 2 | 2 | 4 | Parametrized tier boundary tests | Backend |
| R-015 | SEC | Proposal collaboration — pessimistic lock TTL expiry + concurrent overwrite risk | 2 | 2 | 4 | Lock race condition testing | Backend |

#### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| R-010 | SEC | File upload ClamAV — malware scanning integration reliability | 1 | 3 | 3 | Monitor |
| R-011 | DATA | Document retention — soft delete + S3 purge lifecycle correctness | 1 | 2 | 2 | Monitor |
| R-013 | TECH | i18n — Bulgarian + English coverage in UI and AI outputs | 2 | 1 | 2 | Monitor |
| R-014 | OPS | White-label subdomain isolation — Enterprise only, limited customers at launch | 1 | 2 | 2 | Monitor |

**Risk Category Legend:** TECH (architecture/integration), SEC (security), PERF (performance), DATA (integrity), BUS (business impact), OPS (deployment/operations)

---

### Testability Concerns and Architectural Gaps

**🚨 ACTIONABLE CONCERNS — Architecture Team Must Address**

#### Blockers to Fast Feedback

| Concern | Impact | What Architecture Must Provide | Owner | Timeline |
|---------|--------|-------------------------------|-------|----------|
| No test data seeding API | Cannot create users, companies, subscriptions, or opportunities in specific states | `POST /api/test-data/seed` + `DELETE /api/test-data/cleanup` (dev/staging only) | Backend Lead | Pre-implementation |
| No KraftData mock/stub | Every AI feature test requires live KraftData — slow, non-deterministic, expensive | AI Gateway mock mode returning deterministic fixture responses for all 22 agent types | Backend Lead | Pre-implementation |
| No Stripe webhook simulator | Cannot test trial expiry, subscription changes, add-on purchases, or VAT edge cases | Stripe CLI test mode + webhook forward configuration documented | Backend Lead | Pre-implementation |
| No Redis Streams test harness | Cannot verify event-driven flows (alert delivery, calendar sync, subscription changes) | Event test helper to publish/consume streams with deterministic assertions | Backend Lead | Implementation |

#### Architectural Improvements Needed

1. **Redis Streams dead letter handling**
   - **Current problem:** No dead letter stream documented across any of the 7 streams or 4 consumer groups; failed events may be lost silently
   - **Required change:** Add dead letter stream per consumer group with retry policy and alerting
   - **Impact if not fixed:** Silent event loss → missed notifications, stale caches, calendar sync failures
   - **Owner:** Backend Lead | **Timeline:** Implementation phase

2. **W3C Trace Context propagation**
   - **Current problem:** OpenTelemetry + Jaeger in the stack, but cross-service propagation via W3C Trace Context headers in REST calls and Redis Streams event headers is not documented
   - **Required change:** Ensure trace context propagated in all 5 services + Redis Streams event headers
   - **Impact if not fixed:** Cannot trace requests across service boundaries during debugging
   - **Owner:** Backend Lead | **Timeline:** Implementation phase

---

### Testability Assessment Summary

**What Works Well**

- API-first design (FastAPI) — 100% of business logic accessible via REST; no UI coupling required
- Schema isolation per service — clear data ownership boundaries enable isolated testing
- Circuit breaker on AI Gateway — graceful degradation when KraftData unavailable
- Stateless services with HPA — horizontally scalable; no session state to manage in tests
- Pydantic validation at all service boundaries — strong typing reduces integration contract errors
- Separate VPN-restricted ingress for Admin API — clean security boundary for admin testing
- Redis-backed usage metering with atomic INCR — deterministic rate limit testing

**Accepted Trade-offs (No Action Required)**

- **Pessimistic locking (not CRDT)** for proposal collaboration — simpler to test than real-time sync; 15-min TTL sufficient for MVP
- **Single PostgreSQL instance** — acceptable for MVP scale; schema isolation provides adequate separation. Revisit if load tests show contention
- **KraftData agent management via admin UI (not IaC)** — EU Solicit's responsibility is orchestration; agent configuration lives in KraftData

---

### Risk Mitigation Plans (High-Priority Risks ≥6)

#### R-001: KraftData Single Point of Failure (Score: 9) — CRITICAL

1. AI Gateway circuit breaker: fail-open after 5 consecutive failures, retry after 30s (already in ADR 1.7)
2. Graceful degradation UX: "AI features temporarily unavailable" with last-cached results where possible
3. AI Gateway mock mode: deterministic fixture responses for all 22 agent types (enables CI/CD testing without KraftData connectivity)

**Owner:** Backend Lead | **Timeline:** Pre-implementation (mock mode), Implementation (graceful degradation UX) | **Status:** Planned
**Verification:** E2E test with AI Gateway in mock mode; chaos test with KraftData unavailable

#### R-002: Multi-Tenant Data Isolation (Score: 6) — HIGH

1. PostgreSQL DB role enforcement: each service's role verified against the DB role access matrix (5 roles × 6 schemas)
2. Entity-level RBAC: `company_id` scoping validated on every query in Client API
3. Automated isolation test: create Company A and Company B; verify Company A cannot access Company B data at any endpoint

**Owner:** Architect + Backend Lead | **Timeline:** Pre-implementation (DB role setup), Implementation (RBAC validation) | **Status:** Planned
**Verification:** Automated cross-tenant data access sweep at API level

#### R-003: SSE Stream Saturation (Score: 6) — HIGH

1. Separate connection pool for SSE streams vs. REST requests in AI Gateway
2. HPA scaling metric based on active SSE connection count (not just CPU/memory)
3. Per-tenant connection limit to prevent noisy-neighbour saturation

**Owner:** Backend Lead | **Timeline:** Implementation phase | **Status:** Planned
**Verification:** k6 load test simulating concurrent SSE + REST requests under sustained load

#### R-004: Stripe Billing Complexity (Score: 6) — HIGH

1. Stripe test clocks for trial lifecycle simulation (registration → 14-day professional → downgrade to Free)
2. Webhook event simulation for all subscription lifecycle events: `checkout.session.completed`, `customer.subscription.trial_will_end`, `customer.subscription.updated`, `payment_intent.payment_failed`
3. EU VAT edge cases: B2B reverse charge, B2C with VAT, missing tax ID, Enterprise custom invoicing (NET 30/60)

**Owner:** Backend Lead | **Timeline:** Implementation phase | **Status:** Planned
**Verification:** API tests covering full subscription lifecycle, add-on purchases, and VAT calculation

#### R-006: Entity-Level RBAC Enforcement (Score: 6) — HIGH

1. Permission matrix test: all 5 entity roles (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only) × all entity types (opportunity/proposal) × all permission levels (view/comment/edit/manage)
2. Verify company-wide role ceiling cannot be exceeded at entity level
3. Verify per-proposal isolation: user with edit on Proposal A receives 403 on Proposal B

**Owner:** Backend | **Timeline:** Implementation phase | **Status:** Planned
**Verification:** Exhaustive parametrized permission matrix test suite

---

### Assumptions and Dependencies

#### Assumptions

1. KraftData platform meets 99.5% availability SLA and supports concurrent agent execution per tenant
2. PostgreSQL replication lag (if added post-MVP) stays under 100ms for read replicas
3. Stripe Tax handles all EU member state VAT rates correctly (EU Solicit does not maintain its own VAT table)
4. All 22 KraftData agents/teams/workflows are pre-configured and available before E2E testing begins
5. AWS eu-central-1 region meets data residency requirements for all EU member states

#### Dependencies

| Dependency | Required By | Blocks |
|-----------|-------------|--------|
| KraftData API contracts finalized (agent IDs + endpoints stable) | Sprint 4 | AI Gateway integration tests |
| Stripe account + test mode + webhook endpoint configuration | Sprint 6 | All billing tests |
| Google + Microsoft OAuth app registration for dev/staging | Sprint 8 | Calendar sync tests |
| ClamAV sidecar container deployable in all environments | Sprint 6 | File upload tests |
| SendGrid test API key + templates | Sprint 7 | Email alert + notification tests |

#### Risks to Plan

- **KraftData API changes break AI Gateway integration mid-sprint** → All AI-dependent tests fail; proposal generation blocked. **Contingency:** AI Gateway mock mode enables testing independently of KraftData availability.

---

**Next Steps for Architecture Team:**

1. Review 🚨 BLOCKERS (TB-01, TB-02, TB-03) and assign owners + timelines
2. Validate ⚠️ HIGH PRIORITY items (R-002, R-003, R-006) and confirm recommendations
3. Confirm assumptions and dependency schedule with Platform team
4. Provide feedback to QA on any testability gaps not captured here

**Companion document:** `test-design-qa.md` — test execution recipe, coverage matrix, code examples, effort estimates.
