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

# Test Design for Architecture: EU Solicit

**Purpose:** Architectural concerns, testability gaps, and NFR requirements for review by Architecture/Dev teams. Serves as a contract between QA and Engineering on what must be addressed before test development begins.

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**Status:** Architecture Review Pending
**Project:** EU Solicit
**PRD Reference:** EU_Solicit_PRD_v1.md
**ADR Reference:** EU_Solicit_Solution_Architecture_v4.md (ADRs 1.1–1.15)
**Version:** v3 (2026-04-09) — updated agent inventory count, new medium risks R-016/R-017 from epic-level learnings, risk register expanded to 17 risks

---

## Executive Summary

**Scope:** Full platform — 5 FastAPI/Celery services, Next.js 14 frontend, PostgreSQL 16 (6 schemas), Redis 7 (7 streams, 4 consumer groups), KraftData Agentic AI (29 entities: 27 agents + 1 team + 1 workflow, 7 vector stores), Stripe billing (trials, add-ons, EU VAT, usage metering), Google Calendar + Outlook sync, data pipeline crawlers (AOP, TED, EU Grant), proposal collaboration, task orchestration, approval workflows, ESPD auto-fill, submission guides.

**Business Context** (from PRD):

- **Revenue/Impact:** SaaS platform with 4 tiers (Free → Enterprise); revenue from Starter (€29/mo), Professional (€59/mo), Enterprise (€109+/mo), and per-bid add-ons; 14-day free trial drives Professional conversion
- **Target milestones:** Demo at Sprint 8 / Week 16, Beta at Sprint 12 / Week 24, MVP Launch at Sprint 14 / Week 28 (22-week timeline)

**Architecture** (ADR v4):

- **ADR 1.2:** Hybrid SOA — 5 services (Client API, Admin API, Data Pipeline, AI Gateway, Notification Service)
- **ADR 1.3:** REST + Redis Streams async events; 7 named streams, 4 consumer groups; `traceparent` + `trace_id` propagated across all services and stream events
- **ADR 1.5:** Single PostgreSQL instance, 6 schemas, per-service DB roles with enforced access boundaries; 5 DB roles × 6 schemas
- **ADR 1.7:** All AI capabilities delegated to KraftData Agentic AI platform; 29 entities (27 agents, 1 team, 1 workflow) referenced by logical names in agent registry; EU Solicit owns orchestration only
- **ADR 1.13:** Entity-level RBAC — company-wide role sets ceiling, per-entity permission narrows; 5 entity roles (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only)
- **ADR 1.14:** Billing includes trials (no CC), per-bid add-ons (Stripe one-time), EU VAT via Stripe Tax

**Risk Summary:**

- **Total risks:** 17 (5 high-priority ≥6, 8 medium, 4 low)
- **Test effort:** ~92 scenarios (~6–9 weeks for 1 QA, ~3–5 weeks for 2 QAs)

---

## Quick Guide

### 🚨 BLOCKERS — Team Must Decide (Can't Proceed Without)

**Pre-Implementation Critical Path** — Must be completed before QA can write integration tests:

1. **TB-01: Test Data Seeding API** — `POST /api/test-data/seed` and `DELETE /api/test-data/cleanup` endpoints (dev/staging only). Must support: users (all 5 entity roles), companies (all 4 tiers + trial states), opportunities (various CPV/region/budget/status combinations), proposals (with versions, collaborators, section locks), tasks (with DAG dependencies), and approval workflows (configurable stages). Without this, every integration test must manually create data through the full API chain — slow, brittle, and parallelization-unsafe. **Owner:** Backend Lead. **Required by:** Sprint 3.

2. **TB-02: KraftData Mock Service** — AI Gateway mock mode returning deterministic, fixture-based responses for all 29 KraftData entity types (27 agents, 1 team, 1 workflow). Mock must support: streaming SSE responses (Proposal Generator Workflow), synchronous responses (all agents), configurable failure injection (for circuit-breaker and graceful-degradation tests), and agent-type-specific response schemas. Without this, every AI feature test requires live KraftData connectivity — slow, non-deterministic, expensive, and unavailable in CI. **Owner:** Backend Lead. **Required by:** Sprint 3.

3. **TB-03: Stripe Test Mode Configuration** — Stripe test-mode keys configured across all environments, Stripe CLI webhook forwarding documented and runnable locally, Stripe test clocks configured for trial lifecycle simulation (14-day expiry, `customer.subscription.trial_will_end`, grace period, downgrade to Free). Must cover: subscription creation, trial creation at registration, trial expiry → Free downgrade, add-on Checkout Sessions (one-time payments), subscription webhook events (`checkout.session.completed`, `customer.subscription.updated`, `payment_intent.payment_failed`), and EU VAT edge cases (B2B reverse charge, B2C, missing tax_id, Enterprise custom invoicing). **Owner:** Backend Lead. **Required by:** Sprint 4.

### ⚠️ HIGH PRIORITY — Team Should Validate

1. **R-002: Multi-tenant data isolation** — Shared PostgreSQL with 6 schemas and cross-schema reads. Client API reads `pipeline` schema (opportunities); Admin API reads `client` and `pipeline` schemas; Notification Service reads `client` schema. Each requires validation that DB roles enforce exact boundaries (no cross-schema write access, no cross-company row access). Recommend automated DB-role isolation tests + automated cross-tenant API sweep. **Owner:** Architect + Backend Lead. **Phase:** Pre-implementation.

2. **R-003: SSE stream saturation** — AI Gateway proxies long-lived SSE connections (Proposal Generator Workflow, Executive Summary Agent) while also handling synchronous REST requests. Under concurrent load, SSE connections risk starving the REST connection pool. Recommend: separate uvicorn worker pool or asyncio task group for SSE endpoints; HPA scaling metric based on active SSE connection count (not just CPU/memory); per-tenant connection limit. **Owner:** Backend Lead. **Phase:** Implementation.

3. **R-006: Entity-level RBAC enforcement** — Two-tier permission model: company role sets ceiling, entity permission (`entity_permissions` table) narrows per opportunity/proposal. Edge cases: bid_manager on Proposal A but read_only on Proposal B; user with technical_writer company role but no entity permission on a specific proposal; permission grants for deleted entities; collaborator role inheritance when proposal is duplicated. Recommend exhaustive permission matrix tests covering all combinations. **Owner:** Backend. **Phase:** Implementation.

### 📋 INFO ONLY — Solutions Provided

- **Test strategy:** API-heavy (60% API, 25% E2E, 10% Unit, 5% Performance) — API-first architecture makes API testing the highest-value level; all business logic accessible via REST without UI coupling
- **Tooling:** Playwright (E2E + API tests), pytest + pytest-asyncio (unit/integration), k6 (performance)
- **CI/CD cadence:** Every PR (~10–15 min Playwright + pytest), Nightly (k6 performance), Weekly (chaos/DR/ZAP security)
- **Coverage:** ~92 test scenarios P0–P3; full plan in companion QA doc (test-design-qa.md)
- **Observability:** W3C Trace Context (`traceparent` header + `trace_id` in Redis stream payloads per Architecture §15) enables cross-service trace stitching in Jaeger — strong foundation for debugging test failures

---

## For Architects and Devs — Open Topics 👷

### Risk Assessment

**Total risks identified:** 17 (5 high-priority ≥6, 8 medium, 4 low)

#### High-Priority Risks (Score ≥6) — IMMEDIATE ATTENTION

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **R-001** | TECH | KraftData single point of failure — all 29 AI entities (27 agents, 1 team, 1 workflow) depend on one external platform; 7 vector stores also externally managed | 3 | 3 | **9** | Circuit breaker (fail-open after 5 failures) + graceful degradation UX + deterministic mock for all 29 entity types | Backend Lead | Pre-implementation (mock); Implementation (degradation UX) |
| **R-002** | SEC | Multi-tenant data isolation — shared PostgreSQL, cross-schema reads, entity-level RBAC complexity across 5 roles × all entity types | 2 | 3 | **6** | DB role enforcement tests + automated cross-tenant API sweep + entity permission matrix | Architect + Backend Lead | Pre-implementation |
| **R-003** | PERF | SSE stream saturation — AI Gateway proxies long-lived SSE connections; connection pool starvation risk under concurrent load (Proposal Generator + Executive Summary simultaneous users) | 2 | 3 | **6** | Separate SSE connection pool + HPA SSE-count metric + per-tenant connection limit | Backend Lead | Implementation |
| **R-004** | BUS | Stripe billing complexity — 4 flows (subscriptions, trials with no-CC, per-bid add-ons, EU VAT via Stripe Tax), usage metering, Enterprise custom invoicing, trial→Free downgrade on expiry | 2 | 3 | **6** | Stripe test clocks + webhook simulation + all lifecycle event paths covered | Backend Lead | Implementation |
| **R-006** | SEC | Entity-level RBAC edge cases — two-tier model (company role ceiling + entity permission) with 5 roles, 2 entity types (opportunity, proposal), multiple permission levels; grant/revoke/inherit edge cases | 2 | 3 | **6** | Exhaustive permission matrix: all 5 roles × 2 entity types × 4 permission levels + edge cases | Backend | Implementation |

#### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| R-005 | DATA | Data pipeline staleness — crawlers implemented as KraftData agents; if KraftData is unavailable, no new opportunities ingested; users see stale results | 2 | 2 | 4 | AI Gateway circuit breaker serves existing data gracefully; stale-data UX indicator | Backend |
| R-007 | TECH | Redis Streams dead letter handling — 7 streams, 4 consumer groups; no dead letter queue documented; failed events (notification delivery, calendar sync, cache invalidation) may be silently lost | 2 | 2 | 4 | Add DLQ stream per consumer group + retry policy + Prometheus alert on DLQ depth | Backend Lead |
| R-008 | PERF | PostgreSQL shared instance contention — 6 schemas, FTS on 10K+ opportunities, materialized view refresh (6 dashboards), concurrent analytics queries | 2 | 2 | 4 | Index optimization + connection pool per service + FTS index benchmarking + materialized view refresh scheduling | DBA/Backend |
| R-009 | OPS | Calendar sync OAuth token lifecycle — Google + Microsoft OAuth2 tokens expire and must be refreshed; revocation must propagate to calendar sync jobs | 2 | 2 | 4 | OAuth mock (token refresh/revocation) + Celery beat sync integration tests | Backend |
| R-012 | BUS | Tier gating accuracy — complex access matrix: 4 tiers × region (BG/EU) × CPV code × budget range × feature type (summary/proposal/compliance); free vs. paid field visibility | 2 | 2 | 4 | Parametrized tier boundary tests covering all tier × feature × region combinations | Backend |
| R-015 | SEC | Proposal section locking race conditions — pessimistic TTL-based locking (15-min auto-release); concurrent overwrite risk on TTL boundary; lock holder disconnects without explicit unlock | 2 | 2 | 4 | Concurrent lock acquisition tests + TTL boundary tests + 423 response validation | Backend |
| **R-016** ★NEW | TECH | Frontend locale middleware routing — Next.js middleware handles BG/EN locale routing for all paths; misconfiguration causes incorrect locale assignment, broken deep links, or redirect loops for all users | 2 | 2 | 4 | Locale routing Playwright tests covering all locale/path combinations; middleware unit tests; cookie + Accept-Language fallback tests | Frontend Lead |
| **R-017** ★NEW | SEC | JWT refresh race condition at frontend — concurrent 401 responses trigger multiple simultaneous refresh attempts; without deduplication, race results in double-refresh, token invalidation, or user logout on concurrent tab usage | 2 | 2 | 4 | Deduplication gate in apiClient (singleton in-flight refresh promise); Playwright concurrent-tab test; token replay prevention | Frontend Lead |

#### Low-Priority Risks (Score 1–3)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| R-010 | SEC | ClamAV file scanning reliability — malware scanning integration; ClamAV startup delay in CI; virus DB freshness | 1 | 3 | 3 | Monitor; test with EICAR test file; health-check ClamAV before upload tests |
| R-011 | DATA | Document retention lifecycle — soft-delete + 30-day grace + S3 hard delete; Celery Beat task correctness; audit log immutability | 1 | 2 | 2 | Monitor; add retention lifecycle integration test |
| R-013 | TECH | i18n coverage — Bulgarian + English UI completeness; AI output language alignment with user preference; missing translation keys at runtime | 2 | 1 | 2 | Monitor; i18n key coverage test in CI; manual BG QA session |
| R-014 | OPS | White-label subdomain isolation — Enterprise-only; *.eusolicit.com wildcard routing; brand/logo isolation between tenants | 1 | 2 | 2 | Monitor; white-label isolation test for Enterprise tenant |

**Risk Category Legend:** TECH (architecture/integration), SEC (security/access control), PERF (performance/SLA), DATA (data integrity/loss), BUS (business impact/logic), OPS (deployment/operations)

---

### Testability Concerns and Architectural Gaps

**🚨 ACTIONABLE CONCERNS — Architecture Team Must Address**

#### Blockers to Fast Feedback

| Concern | Impact | What Architecture Must Provide | Owner | Timeline |
|---------|--------|-------------------------------|-------|----------|
| No test data seeding API | Cannot create users/companies/subscriptions/opportunities/proposals in controlled states; every test must go through full registration/setup flow — slow, brittle, non-parallelizable | `POST /api/test-data/seed` + `DELETE /api/test-data/cleanup` (dev/staging only); must support all entity types and states | Backend Lead | Pre-implementation (Sprint 3) |
| No KraftData mock service | Every AI feature test (29 entity types) requires live KraftData connectivity — non-deterministic, slow, expensive, CI-unavailable; proposal generation, compliance checks, scoring all untestable without live connection | AI Gateway mock mode: deterministic fixture responses for all 29 entity types; configurable error injection; SSE streaming mock | Backend Lead | Pre-implementation (Sprint 3) |
| No Stripe test mode setup | Cannot test trial lifecycle (14-day, no-CC, expiry → Free downgrade), add-on purchases, subscription webhooks, EU VAT edge cases; billing is a P0 risk area | Stripe CLI webhook forward + test clocks + test mode keys documented across all environments | Backend Lead | Pre-implementation (Sprint 4) |
| No Redis Streams test harness | Cannot verify event-driven flows: `opportunities.ingested` → alert matching → email delivery; `subscription.changed` → tier cache invalidation; `agent.execution.completed` → UI update | Event test helper: publish to stream + assert consumer group acknowledgement + verify downstream effect, with deterministic assertions | Backend Lead | Implementation (Sprint 5) |

#### Architectural Improvements Needed

1. **Redis Streams dead letter queue**
   - **Current problem:** No DLQ documented across any of the 7 streams or 4 consumer groups. Failed events in `eu-solicit:notifications`, `eu-solicit:tasks`, or `eu-solicit:subscriptions` may be silently lost — missed notifications, stale tier caches, missed calendar sync triggers.
   - **Required change:** Add a `eu-solicit:dlq` stream (or per-consumer-group DLQ). Implement retry policy (3 attempts, exponential backoff). Add Prometheus counter on DLQ depth per consumer group, with Grafana alert at depth > 0.
   - **Impact if not fixed:** Silent event loss → unreliable alert delivery, broken calendar sync, stale subscription tier cache
   - **Owner:** Backend Lead | **Timeline:** Implementation phase (Sprint 5)

2. **Per-consumer-group DLQ monitoring**
   - **Current problem:** Even if a DLQ is added, without monitoring per consumer group (`client-api`, `notification-svc`, `data-pipeline`, `ai-gateway`), dead letters accumulate silently.
   - **Required change:** Prometheus gauge `eusolicit_streams_dlq_depth{consumer_group}` + Grafana alert rule at depth > 0.
   - **Owner:** Backend Lead | **Timeline:** Implementation phase

3. **Locale routing middleware test coverage**
   - **Current problem:** Next.js middleware locale routing (BG/EN) is untested at the system level; incorrect routing affects all platform users. Epic-level E03 analysis identified this as a high-risk area.
   - **Required change:** Middleware unit tests + Playwright locale routing smoke test in PR suite.
   - **Owner:** Frontend Lead | **Timeline:** Implementation phase (Sprint 3)

---

### Testability Assessment Summary

**What Works Well**

- ✅ **API-first design (FastAPI)** — 100% of business logic accessible via REST; no UI coupling required for backend testing; OpenAPI auto-generation aids contract testing
- ✅ **Schema isolation per service** — Each service has a dedicated DB schema + DB role; enables isolated unit testing of DB access patterns without cross-service interference
- ✅ **W3C Trace Context propagation** — `traceparent` header in REST + `trace_id` in Redis Stream events (Architecture §15) enables cross-service trace stitching in Jaeger; debugging test failures across service boundaries is tractable
- ✅ **AI Gateway circuit breaker** — Fail-open pattern (5 consecutive failures → open, retry after 30s) enables graceful-degradation tests without live KraftData; deterministic mock mode will remove KraftData dependency from CI entirely
- ✅ **Stateless services with HPA** — No session state to manage in tests; services can be independently scaled and restarted; parallel test execution is safe at service level
- ✅ **Pydantic validation at all service boundaries** — Strict typing reduces integration contract errors; Pydantic DTOs in `eusolicit-models` are testable in isolation
- ✅ **Redis-backed usage metering with atomic INCR** — Usage limits are deterministically testable; atomic INCR + DECR on failure allows predictable rate-limit test scenarios
- ✅ **Separate VPN-restricted ingress for Admin API** — Clean security boundary; admin auth tests are isolated from user auth tests; network policy tests are straightforward
- ✅ **structlog + correlation IDs** — `request_id`, `company_id`, `tier`, `service_name` in every log line; test failures produce searchable, structured evidence

**Accepted Trade-offs (No Action Required)**

- **Pessimistic locking (not CRDT) for proposal collaboration** — Simpler to test than real-time sync; 15-min TTL provides clear lock acquisition/release semantics; 423 response is deterministic. Acceptable for MVP.
- **Single PostgreSQL instance** — Acceptable for MVP scale; schema isolation provides adequate separation. Revisit after load tests if contention detected.
- **KraftData agent management via admin UI (not IaC)** — EU Solicit's responsibility is orchestration only; agent configuration lives in KraftData. Agent registry (logical name → KraftData ID mapping) is the only testable interface.
- **Daily materialized view refresh for analytics dashboards** — Analytics data has an acceptable staleness window; testing uses seeded materialized views rather than real-time computation.

---

### Risk Mitigation Plans (High-Priority Risks ≥6)

#### R-001: KraftData Single Point of Failure (Score: 9) — CRITICAL

**Mitigation Strategy:**

1. **Circuit breaker** (already in ADR 1.7): Fail-open after 5 consecutive KraftData failures; retry after 30s. Test: simulate 5 failures → verify circuit opens → verify REST/UI returns graceful degradation response, not 500.
2. **Graceful degradation UX**: All AI-dependent endpoints return a structured `503` with `{ "message": "AI features temporarily unavailable", "cached_result": {...} }` when circuit is open. UI shows "AI features temporarily unavailable" with cached last result where available.
3. **KraftData mock mode (TB-02)**: AI Gateway mock mode returns deterministic fixture responses for all 29 entity types. Configurable per agent type. Enables 100% CI coverage of AI feature paths without live KraftData connectivity.
4. **Vector store coverage**: 7 vector stores must also be mocked; mock fixtures represent typical RAG retrieval results (RFP Templates, Regulatory Knowledge, Company Profiles, Historical Bids, Market Data, EU Programmes, Consortium Partners).

**Owner:** Backend Lead | **Timeline:** Mock mode pre-implementation; degradation UX in implementation | **Status:** Planned  
**Verification:** E2E test with AI Gateway mock mode ON (all 29 entity types respond); chaos test with KraftData URL unreachable (circuit opens, graceful 503 returned, no 500s)

---

#### R-002: Multi-Tenant Data Isolation (Score: 6) — HIGH

**Mitigation Strategy:**

1. **DB role enforcement**: Verify each service's DB role cannot write to other services' schemas. Test at the PostgreSQL level: connect as `client_api_role`, attempt INSERT into `pipeline` schema → expect permission denied. Run for all 5 role × non-owned schema combinations.
2. **Row-level isolation**: Verify every Client API query includes `company_id` scoping. Automated test: create Company A (user A) and Company B (user B) with identical data; assert Company A's JWT cannot retrieve Company B's opportunities, proposals, tasks, analytics, or documents at any endpoint.
3. **Cross-schema read validation**: Admin API reads `client` schema read-only. Verify `admin_api_role` cannot INSERT/UPDATE/DELETE `client` rows.
4. **Entity permission isolation**: Verify entity-level permissions are scoped to company_id; a user from Company A cannot be granted permissions on Company B's entities.

**Owner:** Architect + Backend Lead | **Timeline:** DB role setup pre-implementation; API isolation tests in Sprint 3 | **Status:** Planned  
**Verification:** Automated cross-tenant data access sweep: ~10 API endpoints, 2 companies, 2 JWT tokens; 0 cross-company data leaks permitted

---

#### R-003: SSE Stream Saturation (Score: 6) — HIGH

**Mitigation Strategy:**

1. **Separate connection pool**: SSE-streaming endpoints (`POST /agents/{id}/run-stream`, `POST /workflows/{id}/run-stream`) use a dedicated uvicorn worker pool or asyncio streaming response handler, isolated from the standard REST request pool.
2. **HPA SSE-count metric**: Add `eusolicit_ai_gateway_active_sse_connections` Prometheus gauge; configure HPA to scale AI Gateway pods on this metric (not just CPU/memory). Scale threshold: > 50 active connections per pod.
3. **Per-tenant SSE connection limit**: Max 3 concurrent SSE connections per company (configurable); 429 response with `Retry-After` when limit exceeded. Prevents noisy-neighbour saturation.
4. **SSE heartbeat + timeout**: Implement 30s heartbeat (`: keep-alive\n\n` SSE comment); client-side reconnect with exponential backoff. Auto-close abandoned SSE connections after 5m of inactivity.

**Owner:** Backend Lead | **Timeline:** Implementation phase (Sprint 7) | **Status:** Planned  
**Verification:** k6 load test: 100 concurrent SSE connections + 100 concurrent REST requests → both must complete within SLA; no REST request degradation with SSE load

---

#### R-004: Stripe Billing Complexity (Score: 6) — HIGH

**Mitigation Strategy:**

1. **Stripe test clocks**: Simulate trial lifecycle: registration → trial_start → trial_end (advance clock) → `customer.subscription.trial_will_end` webhook → 3-day reminder email → trial_end → `customer.subscription.updated` (past_due) → Client API downgrades to Free tier.
2. **Webhook simulation** (Stripe CLI): Test all webhook event handlers: `checkout.session.completed` (subscription + add-on), `customer.subscription.trial_will_end`, `customer.subscription.updated`, `payment_intent.payment_failed`, `customer.subscription.deleted`.
3. **Add-on purchase flow**: Create Stripe Checkout Session (one-time payment, mode=payment) → simulate `checkout.session.completed` → verify `add_on_purchases` record created with `status: succeeded` → verify feature unlocked for specific opportunity.
4. **EU VAT edge cases**: B2B reverse charge (valid EU VAT number → 0% VAT + reverse charge note), B2C (valid address → local VAT rate), missing tax_id (VAT charged), Enterprise custom invoicing (NET 30/60 terms).
5. **One trial per company**: Verify second registration under same company cannot start a new trial; `subscriptions.trial_start IS NOT NULL` gate.

**Owner:** Backend Lead | **Timeline:** Implementation phase (Sprint 4) | **Status:** Planned  
**Verification:** API tests covering full subscription lifecycle, add-on purchases, all webhook event types, and EU VAT edge cases; Stripe CLI `stripe listen --forward-to localhost` in CI

---

#### R-006: Entity-Level RBAC Enforcement (Score: 6) — HIGH

**Mitigation Strategy:**

1. **Company role ceiling enforcement**: Test that a user with `read_only` company role cannot be granted `edit` entity permission on a proposal; entity permission cannot exceed company ceiling.
2. **Entity permission matrix**: Parametrized tests covering: all 5 entity roles (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only) × 2 entity types (opportunity, proposal) × 4 permission levels (view, comment, edit, manage) × HTTP methods (GET, POST, PATCH, DELETE).
3. **Cross-proposal isolation**: User is bid_manager on Proposal A but read_only on Proposal B → verify bid_manager actions on A succeed, read_only restrictions on B are enforced, neither role bleeds into the other.
4. **Permission grant/revoke atomicity**: Grant permission → verify access → revoke permission → verify denied; no window between revoke and enforcement.
5. **Deleted entity handling**: Verify entity permissions are cleaned up when opportunity/proposal is soft-deleted; no orphaned grants allow access to deleted entities.

**Owner:** Backend | **Timeline:** Implementation phase (Sprint 4–5) | **Status:** Planned  
**Verification:** Permission matrix test suite (parametrized): ~40 scenarios covering all combinations; 0 unauthorized access permitted

---

### Assumptions and Dependencies

#### Assumptions

1. **KraftData SLA**: KraftData platform offers ≥99.5% availability in production; circuit breaker handles transient failures but not extended outages.
2. **Stripe Tax handles all EU VAT**: All EU VAT calculation, VIES validation, and reverse-charge logic is handled by Stripe Tax; EU Solicit only stores and syncs `tax_id` to Stripe Customer.
3. **All 29 KraftData entities pre-configured before E2E testing begins**: Agent registry (logical name → KraftData entity ID) populated before any E2E tests run. Mock mode covers CI; staging uses real KraftData.
4. **Google Calendar + Microsoft Graph APIs available in test environment**: OAuth tokens for test accounts provisioned; calendar sync tests use sandboxed OAuth apps.
5. **PostgreSQL and Redis Docker images support required features in CI**: pg_vector extension not required for MVP; Redis Streams v7 supported by Docker image.

#### Dependencies

1. **TB-01: Test data seeding API** — Backend Lead — Sprint 3 (Required for: all integration and E2E tests)
2. **TB-02: KraftData mock service** — Backend Lead — Sprint 3 (Required for: all AI feature tests)
3. **TB-03: Stripe test mode** — Backend Lead — Sprint 4 (Required for: all billing tests)
4. **Redis Streams DLQ** — Backend Lead — Sprint 5 (Required for: event-driven flow tests)
5. **QA test data factories** — QA — Sprint 3 (Required for: test parallelization and isolation)

#### Risks to Plan

- **Risk:** KraftData mock mode not ready by Sprint 3
  - **Impact:** All AI feature tests (E04–E08) are blocked; P0-013 through P0-017 cannot run in CI
  - **Contingency:** Playwright `route()` interceptor at HTTP level to stub KraftData responses until AI Gateway mock mode is ready

- **Risk:** Stripe CLI not available in CI environment
  - **Impact:** Billing webhook tests cannot run; P0-005/P0-006 blocked
  - **Contingency:** Stripe webhook payload factory (hardcoded event payloads) posted directly to webhook endpoint; lose end-to-end Stripe validation but gain coverage of webhook handler logic

- **Risk:** PostgreSQL DB roles not enforced per spec
  - **Impact:** R-002 cross-tenant isolation tests may pass incorrectly; silent security gap
  - **Contingency:** Add DB role verification as a gate in Sprint 1 infra setup; fail CI if role matrix deviates from architecture spec

---

**End of Architecture Document**

**Next Steps for Architecture Team:**

1. Review Quick Guide (🚨/⚠️/📋) and prioritize blockers TB-01, TB-02, TB-03
2. Assign owners and timelines for high-priority risks (R-001 through R-006)
3. Confirm KraftData agent inventory (29 entities per Architecture §11.2) and ensure mock mode covers all types
4. Design and implement Redis Streams DLQ before Sprint 5
5. Validate locale routing middleware test strategy with Frontend Lead (R-016)

**Next Steps for QA Team:**

1. Wait for pre-implementation blockers TB-01–TB-03 to be resolved
2. Refer to companion QA doc (test-design-qa.md) for full test scenario catalogue
3. Begin test infrastructure setup: factories, fixtures, Docker Compose test profile
4. Implement Playwright `route()` stubs for KraftData as TB-02 contingency

**Companion Document:** `test-design-qa.md` (test execution recipe, full coverage plan)  
**Handoff Document:** `test-design/eu-solicit-handoff.md` (BMAD integration guidance)
