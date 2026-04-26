---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-19'
workflowType: 'testarch-test-design'
inputDocuments: ['EU_Solicit_PRD_v1.md', 'EU_Solicit_Solution_Architecture_v4.md', 'EU_Solicit_Requirements_Brief_v4.md', 'EU_Solicit_User_Journeys_and_Workflows_v1.md', 'EU_Solicit_Service_Decomposition_Analysis.md']
---

# Test Design for Architecture: EU Solicit System-Level v2

**Purpose:** Architectural concerns, testability gaps, and NFR requirements for review by Architecture/Dev teams. Serves as a contract between QA and Engineering on what must be addressed before test development begins.

**Date:** 2026-04-19
**Author:** BMad TEA Agent
**Status:** Architecture Review Pending
**Project:** EU Solicit
**PRD Reference:** EU_Solicit_PRD_v1.md
**ADR Reference:** EU_Solicit_Solution_Architecture_v4.md, EU_Solicit_Service_Decomposition_Analysis.md

> **Note:** This document supersedes the 2026-04-12 version, which incorrectly referenced Kafka (replaced by Redis 7 Streams), Elasticsearch (replaced by PostgreSQL FTS), and "AI Solicit Service" / direct OpenAI/Anthropic integration (replaced by KraftData AI Gateway with 29 agents).

---

## Executive Summary

**Scope:** System-level architecture review for the EU Solicit SaaS platform, covering multi-tenancy, AI-assisted procurement proposal generation, EU grant lifecycle management, event-driven service decomposition, and subscription billing.

**Business Context** (from PRD):

- **Revenue/Impact:** SaaS subscription model with tiered pricing (Free / Starter / Professional / Enterprise), 14-day Professional trial, per-bid add-ons, usage metering, and EU VAT compliance via Stripe Tax.
- **Problem:** Manual procurement proposal drafting and EU grant application tracking are slow, error-prone, and require deep domain knowledge. EU Solicit automates this with AI-assisted drafting, real-time tender synchronisation, and compliance tooling.
- **GA Launch:** Phase 5 completion (Analytics and Polish).

**Architecture** (from ADR / Decomposition):

- **Key Decision 1:** Five independently deployable FastAPI microservices (Client API port 8000, Admin API port 8001, AI Gateway, Data Pipeline, Notification Service) with per-service PostgreSQL 16 schemas (client, pipeline, admin, notification, gateway, shared).
- **Key Decision 2:** Redis 7 Streams event bus — 7 named streams, 4 consumer groups, at-least-once delivery with DLQ. No Kafka.
- **Key Decision 3:** KraftData AI Gateway — 29 specialised agents — handles all AI orchestration via SSE streaming proxy (120 s idle timeout, 600 s total timeout, 15 s heartbeat). No direct OpenAI/Anthropic integration.
- **Key Decision 4:** PostgreSQL full-text search (FTS) for opportunity discovery. No Elasticsearch.
- **Key Decision 5:** Two-tier RBAC — company_role (5 levels) + entity_permissions table. Both layers enforced on every request.
- **Key Decision 6:** RS256 JWT authentication + Google OAuth2; HMAC webhook signature validation (hmac.compare_digest) on all KraftData callbacks.
- **Key Decision 7:** Kubernetes + Helm deployment; MinIO/S3 storage; ClamAV AV scanning; SendGrid email.

**Expected Scale:**

- Multi-tenant support for hundreds of organisations and consultants.
- High-volume tender synchronisation from AOP, TED, and EU grants portals.
- Concurrent SSE streaming sessions under AI-heavy usage peaks.

**Risk Summary:**

- **Total risks**: 18
- **High-priority (≥6)**: 6 risks requiring immediate mitigation (R-001, R-002, R-003, R-004, R-006, R-018)
- **Medium-priority (3-5)**: 8 risks
- **Low-priority (1-2)**: 4 risks
- **Test effort**: ~90–150 hours (~2–3 weeks for 2 QAs, pending blocker resolution)

---

## Quick Guide

### BLOCKERS - Team Must Decide (Can't Proceed Without)

**Pre-Implementation Critical Path** — These MUST be completed before QA can write integration or E2E tests:

1. **TB-01: Test data seeding API** — Architecture must provide `POST /api/test-data/seed` and a matching cleanup endpoint to create isolated tenant fixtures (company, users, roles, subscription tier) and tear them down after each E2E suite run. Required for tenant isolation tests and parallel E2E execution. (recommended owner: Backend / Client API)
2. **TB-02: KraftData mock service** — A local mock server covering all 29 KraftData agents with configurable, deterministic responses and SSE streaming support. Without this, every AI-dependent test (opportunity summary, draft generation, compliance, ESPD auto-fill, scoring, etc.) is blocked or non-deterministic in CI. (recommended owner: AI Gateway Lead)
3. **TB-03: Stripe CLI webhook forwarding in CI** — CI pipeline must run `stripe listen --forward-to` (or equivalent) so that subscription lifecycle events (created, updated, deleted, trial expiry, add-on purchase) can be triggered and observed deterministically in tests. Without this, all billing tests are blocked. (recommended owner: Billing / DevOps)
4. **TB-04: Redis Streams DLQ per consumer group + Prometheus monitoring** — Each consumer group must have a documented dead-letter queue (DLQ) mechanism and a Prometheus metric exposing DLQ depth. Without this, async event delivery tests cannot assert failure and recovery paths. (recommended owner: Platform / Backend)

**What we need from team:** Complete all 4 items pre-implementation or test development is blocked on the corresponding epics.

---

### HIGH PRIORITY - Team Should Validate (We Provide Recommendation, You Approve)

1. **R-002 / R-006: Cross-tenant and entity-level RBAC** — Recommendation: implement a negative-path cross-tenant test suite as mandated by `project-context.md`. The entity_permissions table must be queryable per-proposal and per-collaboration context. QA will build the cross-tenant leakage matrix. (Approve: Security Lead, before any user-facing epic exits QA)
2. **R-018: ESPD XML conformance** — Recommendation: pin the EU ESPD XSD v2.1+ schema file inside the Docker image at build time so validation is deterministic across environments. ESPD serialisation must be tested against the pinned XSD before E11 exits QA. (Approve: Compliance Lead / Backend)
3. **R-003: SSE connection pool saturation** — Recommendation: expose an SSE connection count metric in Prometheus and configure HPA on that metric before load-testing the AI Gateway. QA needs connection pool isolation per tenant before issuing concurrent load. (Approve: Platform / AI Gateway Lead)
4. **R-004: Billing accuracy** — Recommendation: the trial/add-on/VAT/usage-metering code paths must be exercised together in a single end-to-end Stripe test-clock run. Overlapping tier transitions and add-ons on the same invoice are the highest-risk scenario. (Approve: Billing Lead)

**What we need from team:** Review recommendations and approve, or suggest alternative approaches.

---

### INFO ONLY - Solutions Provided (Review, No Decisions Needed)

1. **Test strategy:** Multi-level (E2E Playwright for core user journeys, API-level pytest for contracts and business logic, unit tests for calculations and pure functions). Priority tiers P0–P3 mapped to risk scores.
2. **Tooling:** pytest + testcontainers (eusolicit-test-utils), Playwright with App Router page objects, k6 for load testing SSE and search endpoints, structlog JSON for machine-parseable log assertions.
3. **Tiered CI/CD:** PR gate (P0 / P1 smoke, unit, lint, type-check), Nightly (P2 / P3, integration, E2E), Weekly (load, security, chaos). Mirrors the existing GitHub Actions pipeline.
4. **Coverage:** ~94 prioritised test scenarios across P0–P3, mapped to risks R-001 through R-018 and to epics E01–E12.
5. **Quality gates:** P0 = 100% pass required to merge; P1 = 95% pass; coverage floor 80% (enforced by existing `make coverage`).

**What we need from team:** Review and acknowledge.

---

## For Architects and Devs - Open Topics

### Risk Assessment

**Total risks identified**: 18 (6 high-priority score ≥6, 8 medium score 3–5, 4 low score 1–2)

#### High-Priority Risks (Score ≥6) - IMMEDIATE ATTENTION

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **R-001** | **TECH** | KraftData AI Gateway: non-deterministic agent outputs + circuit breaker failure complexity | 2 | 3 | **6** | Mock service for all 29 agents (TB-02); circuit breaker chaos tests; semantic evaluation hooks for non-determinism | AI Gateway Lead | Pre-E04 |
| **R-002** | **SEC** | Cross-tenant data isolation failure (two-tier RBAC edge cases, entity_permissions misapplication) | 2 | 3 | **6** | Automated cross-tenant negative test suite; DB-level schema isolation; entity_permissions query validation | Security Lead | Pre-E02 exit |
| **R-003** | **PERF** | SSE connection pool saturation under peak AI usage load | 2 | 3 | **6** | SSE connection count Prometheus metric; HPA on SSE count; per-tenant pool isolation before load testing | Platform / AI Gateway Lead | Pre-E04 load test |
| **R-004** | **BUS** | Billing inaccuracy: overlapping trial/add-on/VAT/usage-metering logic in Stripe | 2 | 3 | **6** | Stripe CLI in CI (TB-03); end-to-end test-clock run covering trial + add-on + VAT on same invoice | Billing Lead | Pre-E08 exit |
| **R-006** | **SEC** | Entity-level RBAC bypass on per-proposal collaborator permissions | 2 | 3 | **6** | Entity-level permission matrix test; proposal-scoped collaborator negative-path suite | Security Lead | Pre-E10 exit |
| **R-018** | **TECH/BUS** | ESPD XML serialisation non-conformance to EU XSD v2.1+ standard | 2 | 3 | **6** | Pin ESPD XSD v2.1+ in Docker image at build time; CI validation step against pinned XSD before E11 ESPD export tested | Compliance Lead / Backend | Pre-E11 exit |

#### Medium-Priority Risks (Score 3-5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-005 | DATA | Data pipeline crawler failures producing stale/missing opportunity data | 2 | 2 | 4 | Graceful degradation tests; stale-data health-check endpoint; retry/backoff tests for AOP, TED, EU grants crawlers | Data Pipeline Lead |
| R-007 | TECH | Redis Streams at-least-once delivery: DLQ gaps, duplicate event processing | 2 | 2 | 4 | DLQ per consumer group + Prometheus metric (TB-04); idempotency tests for all 7 streams | Platform / Backend |
| R-008 | PERF | PostgreSQL full-text search latency under concurrent load | 2 | 2 | 4 | tsvector index coverage tests; k6 concurrent-search load test; query plan review | Backend / DBA |
| R-011 | TECH | Proposal section locking deadlock under concurrent multi-user editing | 2 | 2 | 4 | Section lock TTL tests; concurrent-edit race condition simulation; deadlock detection in integration tests | Backend / Collaboration Lead |
| R-012 | BUS | Tier gating enforcement gap: serialization missing for degraded-tier API responses | 2 | 2 | 4 | Tier-gated serialization tests for all 4 tiers; regression gate on opportunity and proposal endpoints | Backend / Product |
| R-015 | SEC | LLM prompt injection via user-supplied tender/proposal content | 2 | 2 | 4 | Input sanitization layer before KraftData dispatch; adversarial prompt suite in E2E; HMAC validation on all callbacks | AI Gateway Lead / Security |
| R-017 | SEC | JWT refresh race condition under concurrent browser tabs | 2 | 2 | 4 | Refresh token deduplication (mutex or atomic Redis flag); concurrent-tab simulation in Playwright | Auth Lead / Frontend |
| R-010 | SEC | File upload security: ClamAV scan bypass or race condition | 1 | 3 | 3 | EICAR test-file upload assertions; scan-before-serve enforced in storage pipeline; race condition test (scan vs. download) | Backend / Storage |

#### Low-Priority Risks (Score 1-2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| R-009 | OPS | Google Calendar OAuth token expiry / refresh for calendar sync | 1 | 2 | 2 | Monitor; add token-refresh integration test when Calendar sync is implemented |
| R-013 | OPS | Alembic migration rollback failure causing schema drift across services | 1 | 3 | 3 | Migration smoke test in CI (`make migrate-all`); per-service rollback dry-run in staging |
| R-014 | SEC | HMAC webhook signature timing attack (== comparison instead of hmac.compare_digest) | 1 | 3 | 3 | Code review gate: enforce `hmac.compare_digest` in all webhook handlers; static analysis rule |
| R-016 | TECH | Next.js locale routing middleware regression on BG/EN locale switching | 1 | 2 | 2 | Monitor; Playwright locale-switching smoke test on every PR |

#### Risk Category Legend

- **TECH**: Technical / Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

### Testability Concerns and Architectural Gaps

**ACTIONABLE CONCERNS - Architecture Team Must Address**

#### 1. Blockers to Fast Feedback (WHAT WE NEED FROM ARCHITECTURE)

| Concern | Impact on Testing | What Architecture Must Provide | Owner | Timeline |
| :--- | :--- | :--- | :--- | :--- |
| **No test data seeding API (TB-01)** | Cannot parallelise E2E runs; tenant isolation tests cannot be automated | `POST /api/test-data/seed` with tenant fixture payload + matching DELETE cleanup endpoint | Backend / Client API | Pre-E01 exit |
| **No KraftData mock service (TB-02)** | All AI-dependent tests (summary, draft, compliance, ESPD, scoring) are non-deterministic or require live KraftData in CI | Local mock server covering all 29 agents with configurable SSE responses and error injection | AI Gateway Lead | Pre-E04 |
| **No Stripe CLI in CI (TB-03)** | All billing lifecycle tests (trial, upgrade, downgrade, add-on, VAT) cannot be triggered deterministically | `stripe listen` forwarding in CI Docker Compose or equivalent webhook injection harness | Billing / DevOps | Pre-E08 |
| **No Redis Streams DLQ or monitoring (TB-04)** | Async delivery failure and recovery paths cannot be tested; duplicate-event handling is unverifiable | DLQ per consumer group (documented stream name + consumer group name); Prometheus gauge for DLQ depth | Platform / Backend | Pre-E09 |

#### 2. Architectural Improvements Needed (WHAT SHOULD BE CHANGED)

1. **KraftData test/sandbox mode**
   - **Current problem**: KraftData AI Gateway has no documented test mode or prompt snapshotting mechanism. Every CI run against a live gateway is non-deterministic and potentially billable.
   - **Required change**: Define a `KRAFTDATA_ENV=test` mode that returns pre-recorded responses, or provide a contract-test harness with recorded cassettes per agent.
   - **Impact if not fixed**: AI tests will remain brittle and unsuitable for PR gates; semantic drift in agent outputs will go undetected.
   - **Owner**: AI Gateway Lead
   - **Timeline**: Pre-E04

2. **Redis Stream event envelope missing correlation_id**
   - **Current problem**: The Redis 7 Streams event envelope does not include a `correlation_id` field. This makes it impossible to trace a single user action through all downstream events in tests or in production observability.
   - **Required change**: Add `correlation_id` (UUID, propagated from the originating HTTP request) to every stream event envelope alongside `event_type`, `tenant_id`, and `timestamp`.
   - **Impact if not fixed**: Async test assertions must use fragile timing windows; distributed tracing across services is broken; DLQ root-cause analysis is impractical.
   - **Owner**: Platform / Backend
   - **Timeline**: Pre-E05

3. **No tenant cleanup endpoint for E2E teardown**
   - **Current problem**: E2E test suites that create companies, users, proposals, and subscriptions have no reliable way to remove all fixtures after a run, leading to DB pollution in shared environments.
   - **Required change**: Provide a `DELETE /api/test-data/tenant/{company_id}` endpoint (test/staging environments only, guarded by an env flag) that cascades deletion across all schemas for the given tenant.
   - **Impact if not fixed**: Test isolation degrades over time; nightly E2E runs accumulate stale data; tenant-count assertions become unreliable.
   - **Owner**: Backend / Client API
   - **Timeline**: Pre-E01 exit (same as TB-01)

4. **PDF proposal export lacks JSON alternative for structured assertion**
   - **Current problem**: Exported proposals are PDF and DOCX only. Validating section content (e.g., confirming a specific AI-generated clause is present) requires fragile PDF text extraction.
   - **Required change**: Expose a `GET /api/v1/proposals/{id}/export?format=json` endpoint returning structured proposal content (sections, metadata, version) to allow deterministic assertion in tests, alongside PDF/DOCX generation.
   - **Impact if not fixed**: Proposal content tests rely on PDF parsing or manual review; regression coverage for export is severely limited.
   - **Owner**: Backend / Proposal Lead
   - **Timeline**: Pre-E07

5. **Celery Beat digest schedules lack test-trigger endpoint**
   - **Current problem**: Notification Service uses Celery Beat for scheduled digest delivery. Tests cannot trigger a scheduled task immediately without manipulating system time or waiting for the next beat interval.
   - **Required change**: Provide a `POST /api/v1/notifications/digests/trigger` endpoint (test/staging only) that fires the digest Celery task synchronously for a given tenant, bypassing the beat schedule.
   - **Impact if not fixed**: Notification digest tests are slow (must wait for beat interval) or require fragile time mocking; CI nightly runs are unreliable.
   - **Owner**: Notification Service Lead
   - **Timeline**: Pre-E09

---

### Testability Assessment Summary

**CURRENT STATE - FYI**

#### What Works Well

- **testcontainers via eusolicit-test-utils**: PostgreSQL, Redis, and MinIO spin up in isolation per test session with no shared state. Integration tests are fully self-contained.
- **Transaction rollback fixtures**: Per-service database fixtures roll back after each test, eliminating test-order dependencies and enabling fast parallel execution.
- **FastAPI OpenAPI for contract testing**: All 5 services expose machine-readable OpenAPI schemas, enabling contract tests and client generation without manual maintenance.
- **structlog JSON output**: All services emit structured JSON logs in CI, making log assertions in tests precise and machine-parseable.
- **Per-service DB schemas**: PostgreSQL schema-per-service isolation means a test that corrupts the `client` schema cannot affect `pipeline` or `notification` schemas — critical for parallel integration runs.
- **ClamAV EICAR test file support**: The ClamAV sidecar accepts the EICAR standard test string, enabling deterministic antivirus scan pass/fail assertions without uploading real malware.

#### Accepted Trade-offs (No Action Required)

For EU Solicit Phase 1–3, the following trade-offs are acceptable:

- **AI output subjectivity**: Validating the "quality" or "persuasiveness" of KraftData-generated proposal sections remains partially manual (UAT / human review). Automated tests assert structural correctness (sections present, length thresholds, required fields populated) but not semantic quality. Accepted for GA launch.
- **PostgreSQL FTS eventual index update**: When a new opportunity is ingested by the Data Pipeline, there is a brief window (sub-second under normal load) before the FTS index reflects it. Integration tests use polling with a short back-off rather than synchronous assertion. This is acceptable and expected behaviour.
- **Google OAuth2 token refresh coverage**: Calendar sync token refresh paths (R-009) are lower priority than core authentication flows and will be covered in a follow-up sprint once Calendar sync is fully implemented.

---

### Risk Mitigation Plans (High-Priority Risks ≥6)

**Purpose**: Detailed mitigation strategies for all 6 high-priority risks (score ≥6). These risks MUST be addressed before GA launch.

#### R-001: KraftData AI Gateway — Non-deterministic Agent Outputs + Circuit Breaker Failure Complexity (Score: 6) - CRITICAL

**Mitigation Strategy:**

1. Deliver the KraftData mock service (TB-02) covering all 29 agents with recorded SSE response fixtures and configurable error injection (timeout, 503, partial stream).
2. Define `KRAFTDATA_ENV=test` environment variable that routes all AI Gateway calls to the mock in CI. Production routing is unaffected.
3. Implement circuit breaker chaos tests: simulate KraftData unavailability for >N consecutive calls and assert that the breaker opens, the fallback response is returned, and the metric is incremented.
4. Add semantic-drift monitoring: record a baseline response for a fixed prompt and alert (not fail) when cosine similarity drops below threshold in nightly runs.

**Owner:** AI Gateway Lead
**Timeline:** Pre-E04 (Infrastructure and AI Gateway epic)
**Status:** Planned
**Verification:** CI gate passes for all P0/P1 AI tests using mock only; circuit breaker test asserts open state within configured threshold.

---

#### R-002: Cross-Tenant Data Isolation Failure — Two-Tier RBAC Edge Cases (Score: 6) - CRITICAL

**Mitigation Strategy:**

1. Implement a cross-tenant negative test suite as mandated by `project-context.md`. For every resource type (opportunities, proposals, documents, audit logs, subscription data), attempt access using a valid JWT from a different tenant and assert 403 or 404.
2. Enforce `company_id` scoping in all SQLAlchemy query filters via a shared query guard in `eusolicit-common`. Missing `company_id` filter must raise a linting error.
3. Validate entity_permissions table for per-proposal collaborator scenarios: assert that a collaborator with `READ` permission cannot call mutation endpoints, and that removing a collaborator immediately revokes access.
4. Run the cross-tenant suite as a P0 gate on every PR touching auth, RBAC, or data-access layers.

**Owner:** Security Lead
**Timeline:** Pre-E02 exit (Authentication and Identity epic)
**Status:** Planned
**Verification:** Cross-tenant negative test suite passes with 100% success; zero 200-status responses on cross-tenant resource requests.

---

#### R-003: SSE Connection Pool Saturation Under Peak AI Usage Load (Score: 6) - HIGH

**Mitigation Strategy:**

1. Expose `sse_connections_active` as a Prometheus gauge in the AI Gateway service, labelled by tenant.
2. Configure Kubernetes HPA on the SSE connection count metric with a defined scale-out threshold.
3. Implement per-tenant SSE connection cap (configurable, default 10 concurrent per tenant) enforced at the proxy layer, returning 429 with `Retry-After` when exceeded.
4. Run a k6 load test that opens N concurrent SSE connections across M tenants and asserts: (a) no connections are silently dropped, (b) the 120 s idle timeout and 600 s total timeout are honoured, (c) 15 s heartbeat frames arrive on all open connections.

**Owner:** Platform / AI Gateway Lead
**Timeline:** Pre-E04 load test sign-off
**Status:** Planned
**Verification:** k6 load test passes at target concurrency; HPA fires at configured threshold; per-tenant cap returns 429 correctly.

---

#### R-004: Billing Inaccuracy — Overlapping Trial/Add-On/VAT/Usage-Metering Logic in Stripe (Score: 6) - HIGH

**Mitigation Strategy:**

1. Integrate Stripe CLI (`stripe listen --forward-to`) in CI Docker Compose as a sidecar (TB-03), enabling deterministic webhook delivery for all subscription lifecycle events.
2. Write end-to-end billing lifecycle tests using Stripe test clocks: (a) company registers → Professional trial created, (b) trial expires after 14 days (test-clock advance) → downgraded to Free with data preserved, (c) upgrade to Starter → Stripe Checkout session completed → tier cache invalidated, (d) add-on purchase on active subscription → invoice line item present, (e) EU VAT applied via Stripe Tax for BG company → VAT line item correct.
3. Assert usage-metering counters (Redis) are reset on tier change and that the Stripe usage record matches the Redis counter at billing cycle end.
4. Include a VAT edge case: Bulgarian company with a valid VIES VAT number should have VAT removed (reverse charge); without a valid number, VAT should be applied.

**Owner:** Billing Lead
**Timeline:** Pre-E08 exit (Subscription and Billing epic)
**Status:** Planned
**Verification:** All billing lifecycle E2E scenarios pass with Stripe test clocks; VAT edge cases validated; usage-metering counter matches Stripe usage record.

---

#### R-006: Entity-Level RBAC Bypass on Per-Proposal Collaborator Permissions (Score: 6) - CRITICAL

**Mitigation Strategy:**

1. Build a permission matrix test covering all combinations of company_role (5 levels) × entity_permission (READ, WRITE, ADMIN, OWNER) × proposal operation (view, edit, export, delete, share). Assert correct allow/deny for each cell.
2. Negative-path scenarios: a collaborator with READ permission attempts a PUT/POST/DELETE on a proposal section → assert 403; a removed collaborator attempts any access → assert 403 or 404.
3. Verify that entity_permissions checks occur in endpoint logic (not only middleware) for proposal collaboration endpoints, as mandated by `project-context.md`.
4. Include concurrent permission change tests: while a collaborator is mid-edit, their permission is revoked → assert the next auto-save returns 403 and the client receives a clear error.

**Owner:** Security Lead
**Timeline:** Pre-E10 exit (Collaboration epic)
**Status:** Planned
**Verification:** Permission matrix test passes for all 5×4×N cells; concurrent revocation test asserts 403 on next mutation after removal.

---

#### R-018: ESPD XML Serialisation Non-Conformance to EU XSD v2.1+ Standard (Score: 6) - HIGH

**Mitigation Strategy:**

1. Pin the EU ESPD XSD v2.1+ schema file inside the Docker image for the Client API service at build time (not fetched at runtime). This ensures deterministic validation regardless of upstream schema changes.
2. Add a CI step that runs `xmllint --schema espd-v2.1.xsd output.xml` (or equivalent Python lxml validation) against every ESPD XML generated by the test suite.
3. Write schema conformance tests covering: required ESPD fields populated correctly, optional fields serialised only when present, namespace prefixes match XSD expectations, and multi-criterion exclusion grounds serialise to the correct XSD enumeration values.
4. Include a regression test for the Bulgarian-language ESPD profile: character encoding (UTF-8), Cyrillic content in free-text fields, and date format (ISO 8601) must all validate against the XSD.

**Owner:** Compliance Lead / Backend
**Timeline:** Pre-E11 exit (EU Grant Specialisation and Compliance epic)
**Status:** Planned
**Verification:** All ESPD XML outputs validate against pinned XSD v2.1+ with zero schema errors in CI; Bulgarian-language regression test passes.

---

### Assumptions and Dependencies

#### Assumptions

1. The KraftData AI Gateway contract (29 agents, SSE streaming protocol, webhook HMAC scheme) remains stable through Phase 4. Any breaking changes require re-evaluation of TB-02 mock cassettes.
2. Stripe test mode and Stripe test clocks remain available and functionally equivalent to production for all billing scenarios.
3. The EU ESPD XSD v2.1+ schema does not receive breaking changes between project start and E11 delivery. The pinned version in Docker will need a deliberate upgrade process if the EU updates the standard.
4. PostgreSQL 16 per-service schema isolation is maintained; no cross-schema direct joins are introduced (all cross-service reads go through APIs as per architecture).
5. Redis 7 Streams DLQ semantics (XACK failures retained in PEL) are sufficient for at-least-once delivery guarantees without an external message broker.
6. The `eusolicit-test-utils` testcontainers package provides stable PostgreSQL 16 and Redis 7 images; container startup time does not exceed CI timeout budgets.

#### Dependencies

1. **TB-01 (Test data seeding API)** — Required before E01 exits QA. Blocks all tenant-isolated E2E tests.
2. **TB-02 (KraftData mock service)** — Required before E04 begins QA. Blocks all AI-dependent test scenarios across E04, E06, E07, E11.
3. **TB-03 (Stripe CLI in CI)** — Required before E08 begins QA. Blocks all billing lifecycle and webhook tests.
4. **TB-04 (Redis DLQ + Prometheus monitoring)** — Required before E09 begins QA. Blocks async event delivery failure and recovery tests.
5. **ESPD XSD v2.1+ pinned in Docker image** — Required before E11 ESPD export is tested. Blocks all ESPD XML conformance assertions.
6. **`correlation_id` in Redis Stream event envelope** — Required before E05 (Data Pipeline) integration tests are finalised. Blocks distributed trace assertions and DLQ root-cause tests.

#### Risks to Plan

- **Risk**: KraftData mock service delivery slips past E04 start.
  - **Impact**: All AI Gateway tests must run against live KraftData, making PR gates non-deterministic and potentially incurring API costs.
  - **Contingency**: Agree on a reduced P0 gate (structural assertions only, no AI content assertions) for E04 stories until mock is available; track as a sprint blocker.

- **Risk**: Stripe test clock behaviour diverges from production for add-on/VAT overlap scenarios.
  - **Impact**: Billing tests may pass in test mode but fail in production during first real billing cycle.
  - **Contingency**: Add a manual QA sign-off step for billing edge cases before Beta launch; review Stripe changelog for test-clock known limitations.

- **Risk**: ESPD XSD v2.1+ is updated by the EU before E11 delivery.
  - **Impact**: Pinned XSD becomes outdated; conformance tests pass but production exports may fail validation.
  - **Contingency**: Monitor EU ESPD GitHub repository for schema releases; treat XSD version as a tracked dependency with a defined upgrade path in the Helm chart.

---

**End of Architecture Document**

**Next Steps for Architecture Team:**

1. Review Quick Guide (BLOCKERS / HIGH PRIORITY / INFO ONLY) and assign owners for all 4 pre-implementation blockers (TB-01 through TB-04).
2. Approve or suggest alternatives for the 4 high-priority recommendations (R-002, R-006, R-018, R-003).
3. Review the 5 architectural improvements in "Testability Concerns" and confirm delivery timelines.
4. Validate assumptions and flag any that do not hold for the current architecture.
5. Provide feedback to QA on any gaps or changes before epic implementation begins.

**Next Steps for QA Team:**

1. Wait for TB-01 through TB-04 to be resolved before writing integration and E2E tests for blocked epics.
2. Refer to companion QA doc (`test-design-qa.md`) for test scenario details, P0–P3 mapping, and coverage plan.
3. Begin test infrastructure setup (factories, fixtures, testcontainers configs) and Playwright page object scaffolding for E01 and E02 which are unblocked.
