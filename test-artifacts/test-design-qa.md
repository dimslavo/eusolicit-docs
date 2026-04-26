---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-19'
workflowType: 'testarch-test-design'
inputDocuments: ['EU_Solicit_Solution_Architecture_v4.md', 'project-context.md', 'test-design-architecture.md']
---

# Test Design for QA: EU Solicit System-Level v2

**Purpose:** Test execution recipe for the QA team. Defines what to test, how to test it, what tooling is required, and what QA needs from other teams before testing can begin. This document is the authoritative test plan for system-level and cross-epic validation.

**Date:** 2026-04-19
**Author:** BMad TEA Agent
**Status:** Active
**Project:** EU Solicit

**Related:** See Architecture doc (`test-design-architecture.md`) for testability concerns and architectural blockers.

---

## Executive Summary

**Scope:** Full system-level testing of the EU Solicit SaaS platform — 5 FastAPI microservices, 2 Next.js 14 frontends, PostgreSQL 16 with per-service schemas, Redis 7 Streams event bus, KraftData AI Gateway (29 agents), Stripe billing, SendGrid email, MinIO/S3 storage, ClamAV virus scanning, and dual-layer RS256 JWT + Google OAuth2 authentication.

**Risk Summary:**

- Total Risks: 18 (6 high-priority score ≥6, 9 medium score 3–5, 3 low)
- Critical Categories: Security (cross-tenant isolation, RBAC), Business (billing accuracy, SSE saturation), Tech (KraftData reliability, ESPD conformance)

**Coverage Summary:**

- P0 tests: 8 (critical paths, security, billing, AI SSE)
- P1 tests: 12 (RBAC matrix, metering, circuit breaker, section locking, audit trail)
- P2 tests: 8 (DLQ, notifications, export, analytics, iCal, tasks, content blocks, submission guide)
- P3 tests: 4 (visual regression, i18n, FTS performance, white-label E2E)
- **Total**: 32 tests (~3–5 weeks with 1 QA)

---

## Not in Scope

**Components or systems explicitly excluded from this test plan:**

| Item | Reasoning | Mitigation |
| :--- | :--- | :--- |
| **KraftData LLM model training and weights** | Third-party ML model outside application scope | Rely on KraftData SLA; validate agent responses through contract mocks (TB-02) |
| **Stripe payment form UI** | Stripe-hosted Checkout is outside our DOM | Validate redirect URL correctness and webhook-driven state via Stripe CLI simulation; do not test Stripe's own form |
| **VIES external service uptime** | EU-operated service known for intermittent downtime | Test fallback path (flag for manual review, do not block checkout); assert retry logic |
| **Physical infrastructure and cloud provisioning** | Managed by DevOps/Platform team; outside QA scope | Covered by infrastructure-as-code review and smoke tests post-deploy |
| **Legacy procurement portal crawlers (AOP, TED, EU Grants)** | Crawler fidelity depends on external portal HTML changes | Smoke-test ingestion pipeline health; full crawler contract tests out of scope for system QA |
| **Browser-level OAuth2 consent screens** | Google-hosted UI outside our DOM | Validate token exchange and session state after callback; do not test Google's own consent form |

**Note:** Items listed here have been reviewed and accepted as out-of-scope by QA, Dev, and PM.

---

## Dependencies & Test Blockers

**CRITICAL:** QA cannot proceed on the listed scenarios without these items from other teams.

### Backend / Architecture Dependencies (Pre-Implementation)

**Source:** See Architecture doc "Quick Guide" for detailed mitigation plans.

1. **TB-01: Stripe CLI Webhook Simulator** — Backend/Fintech — Sprint 9
   - Need `stripe trigger` event automation integrated into CI and staging.
   - Blocks all P0-006, P1-003, P1-004, P1-005 billing and metering tests.

2. **TB-02: KraftData Mock Service** — AI Platform — Sprint 10
   - Need a contract-faithful mock of the 29-agent KraftData API (SSE streaming, HMAC callbacks, circuit-breaker scenarios).
   - Blocks P0-005, P0-008, P1-006, P1-008, P1-010, P2-008 AI tests.

3. **TB-03: Tenant Data Seeding / Teardown API** — Backend/Auth — Sprint 8
   - Need a test-only endpoint (disabled in production) to create isolated tenant fixtures and clean up after each test run.
   - Blocks parallel E2E execution and all cross-tenant isolation tests (P0-001, P0-003).

4. **TB-04: ClamAV Test Container Image** — DevOps — Sprint 8
   - ClamAV must be available in CI via testcontainers or a pre-built Docker image that accepts EICAR strings.
   - Blocks P0-007 virus-scan tests.

### QA Infrastructure Setup (Pre-Implementation)

1. **Test Data Factories** — QA
   - `CompanyFactory`, `UserFactory`, `OpportunityFactory`, `ProposalFactory`, `SubscriptionFactory` using `faker` + `eusolicit-test-utils`.
   - Auto-cleanup via transaction-rollback fixtures for parallel safety.

2. **Test Environments** — QA / DevOps
   - Local: `docker compose up` with ClamAV, Redis 7, PostgreSQL 16, MinIO.
   - CI/CD: GitHub Actions with `testcontainers` services; Stripe CLI sidecar; KraftData mock (TB-02).
   - Staging: Full environment with Stripe Test Mode, SendGrid sandbox, real MinIO bucket.

---

## Risk Assessment

**Note:** Full risk details in Architecture doc. This section maps the 18-risk register to QA test coverage.

### High-Priority Risks (Score ≥6) — MITIGATE before release

| Risk ID | Category | Description | Score | QA Test Coverage |
| :--- | :--- | :--- | :--- | :--- |
| **R-001** | **TECH** | KraftData AI Gateway reliability — circuit breaker, retry storms, per-agent concurrency exhaustion | **8** | P1-006 (circuit breaker opens on failure, 503 returned, resets); P1-008 (mock SSE stream with heartbeats); P2-008 (submission guide caching) |
| **R-002** | **SEC** | Cross-tenant data isolation — JWT from Tenant B must not read Tenant A proposals, opportunities, or documents | **9** | P0-003 (cross-tenant API negative tests across all resource types) |
| **R-003** | **TECH** | SSE connection saturation — 120s idle / 600s total limits; heartbeats every 15s; too many concurrent streams exhaust file descriptors | **6** | P0-005 (full SSE stream lifecycle); P1-008 (heartbeat cadence and stream termination) |
| **R-004** | **BUS** | Billing accuracy — Redis usage counters must stay in sync with Stripe usage records; trial downgrade must not lose data | **8** | P0-006 (14-day trial lifecycle); P1-003 (metering drift); P1-004 (per-bid add-on) |
| **R-006** | **SEC** | Entity-level RBAC bypass — company_role check is necessary but not sufficient; entity_permissions table must also be enforced | **7** | P0-004 (tier gating serialization); P1-001 (RBAC matrix); P1-002 (entity permission 403) |
| **R-018** | **DATA** | ESPD XML conformance — auto-fill agent must produce schema-valid XML against EU procurement standard | **6** | P1-010 (ESPD profile auto-fill field population) |

### Medium-Priority Risks (Score 3–5) — MONITOR

| Risk ID | Category | Description | Score | QA Test Coverage |
| :--- | :--- | :--- | :--- | :--- |
| R-005 | DATA | Crawler staleness — opportunities not refreshed within SLA window | 4 | P1-009 (Redis Streams event delivery chain) |
| R-007 | TECH | Redis Streams DLQ — poison messages blocking consumer groups | 4 | P2-001 (DLQ routing + Prometheus counter) |
| R-008 | PERF | PostgreSQL FTS latency — full-text search > 200ms P95 under load | 4 | P3-003 (k6 load test, 50 VUs, P95 < 200ms) |
| R-010 | SEC | ClamAV scan bypass or timeout — malicious upload reaching S3 | 5 | P0-007 (EICAR rejection + valid PDF acceptance) |
| R-011 | TECH | Proposal section locking race — two users acquiring same section lock | 4 | P1-007 (lock acquired, 423 on conflict, TTL expiry) |
| R-012 | BUS | Tier gating staleness — stale Redis cache granting premium features post-downgrade | 4 | P0-004 (Free/Starter/Professional/Enterprise field sets); P1-003 (429 on limit) |
| R-013 | OPS | Migration rollback failure — Alembic down-migration corrupting schema state | 3 | Covered by CI migration smoke; not in system QA scope |
| R-014 | SEC | HMAC webhook signature bypass — naive string comparison vulnerable to timing attack | 5 | P0-008 (invalid sig → 403; valid sig → processed) |
| R-015 | SEC | Prompt injection via proposal content — adversarial content reaching KraftData agent | 4 | Covered by security review; contract test via TB-02 mock |
| R-017 | SEC | JWT refresh race condition — concurrent refresh requests issuing multiple valid tokens | 4 | P0-002 (JWT lifecycle including refresh and revoke) |

### Low-Priority Risks (Score 1–2) — DOCUMENT

| Risk ID | Category | Description | Score | QA Test Coverage |
| :--- | :--- | :--- | :--- | :--- |
| R-009 | OPS | Calendar OAuth token expiry causing iCal feed breaks | 2 | P2-005 (iCal feed token auth enforced) |
| R-016 | TECH | Next-intl locale routing mismatch — wrong locale served on direct URL | 2 | P3-002 (i18n completeness) |

---

## Entry Criteria

**QA testing cannot begin until ALL of the following are met:**

- [ ] All requirements and assumptions agreed upon by QA, Dev, and PM
- [ ] Staging environment provisioned: PostgreSQL 16, Redis 7, MinIO, ClamAV, all 5 services running
- [ ] `eusolicit-test-utils` factories ready: Company, User, Opportunity, Proposal, Subscription
- [ ] TB-01 (Stripe CLI) available in CI and staging
- [ ] TB-03 (Tenant seeding API) deployed to test environment (disabled in production)
- [ ] TB-04 (ClamAV testcontainer) available in CI pipeline
- [ ] Solution Architecture v4 signed off by Architect and PM

## Exit Criteria

**Testing phase is complete when ALL of the following are met:**

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95%; any failures triaged and accepted with ticket)
- [ ] No open High/Critical security vulnerabilities (R-002, R-006, R-014, R-015)
- [ ] Cross-tenant isolation verified for all tenant-scoped resources (proposals, opportunities, documents)
- [ ] Billing accuracy verified: trial lifecycle, metering sync, per-bid add-on
- [ ] PostgreSQL FTS P95 latency < 200ms under 50 concurrent users (P3-003)
- [ ] Coverage gate: pytest unit + integration ≥80% (enforced by `make coverage`)

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on if time-constrained), NOT execution timing. See "Execution Strategy" for when tests actually run.

### P0 (Critical)

**Criteria:** Blocks core functionality + High risk (score ≥6) + No workaround + Affects majority of users

| Test ID | Requirement | Test Level | Risk Link | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **P0-001** | Full registration → company creation → Professional trial auto-provisioned → protected page accessible | E2E (Playwright) | R-002, R-004 | Covers entire onboarding happy path; verifies AuthGuard + middleware dual-layer auth |
| **P0-002** | JWT auth lifecycle — issue, refresh, expiry returns 401, revoke | API | R-017 | Parameterize: valid refresh, concurrent refresh race, expired token, revoked token |
| **P0-003** | Cross-tenant isolation — Tenant B JWT cannot access Tenant A proposals / opportunities / documents | API | R-002 | Assert 403 or 404 (no ID leakage) for each resource type; use two isolated tenant fixtures |
| **P0-004** | Tier-gated serialization — Free / Starter / Professional / Enterprise see correct field sets for opportunities | API | R-006, R-012 | Parameterize across 4 tiers; verify locked fields absent from Free/Starter responses |
| **P0-005** | AI summary generation via SSE stream — opportunity → SSE connect → content chunks → completion event; usage counter increments | E2E (Playwright) | R-003, R-001 | Requires TB-02 mock; assert 15s heartbeat cadence; assert usage counter in Redis post-stream |
| **P0-006** | Stripe trial lifecycle — registration → trial active → 14-day expiry → auto-downgrade to Free; data preserved | API | R-004 | Use Stripe CLI time-travel; assert proposal data intact after downgrade; assert Free field set |
| **P0-007** | Document upload security — EICAR test file rejected (422); valid PDF accepted → MinIO presigned URL returned | API | R-010 | Requires TB-04; assert ClamAV scan result in response; assert presigned URL is time-limited |
| **P0-008** | HMAC webhook validation — invalid KraftData webhook signature returns 403; valid signature processes correctly | API | R-014 | Use `hmac.compare_digest` constant-time assertion; tamper payload byte-by-byte |

**Total P0:** 8 tests

---

### P1 (High)

**Criteria:** Important features + Medium risk (3–5) + Common workflows + Workaround exists but difficult

| Test ID | Requirement | Test Level | Risk Link | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **P1-001** | Two-tier RBAC parameterized matrix — company_role × endpoint × operation → expected HTTP status | API | R-006 | Parametrize ≥20 combos: Owner/Admin/Editor/Viewer/Guest × create/read/update/delete |
| **P1-002** | Entity-level permission enforcement on proposals — collaborator without entity permission gets 403 | API | R-006 | Add collaborator at company level but not entity level; assert 403 on proposal read/write |
| **P1-003** | Stripe usage metering — Redis counter increments → Stripe sync → 429 returned when limit reached with upgrade prompt | API | R-004, R-012 | Assert `Retry-After` header; assert upgrade URL in 429 body |
| **P1-004** | Per-bid add-on purchase flow — Stripe one-time charge → access granted within same session | API | R-004 | Use Stripe CLI `payment_intent.succeeded` event; assert feature flag set post-webhook |
| **P1-005** | EU VAT handling — VIES validation success/failure; reverse charge mechanics for non-BG EU companies | API | R-004 | Parameterize: BG company (local VAT), DE company (reverse charge), invalid VIES (fallback to manual) |
| **P1-006** | KraftData circuit breaker — simulate gateway failure → circuit opens → 503 returned → circuit resets after configured timeout | API | R-001 | Requires TB-02; inject 5 consecutive failures; assert circuit OPEN; wait TTL; assert circuit HALF-OPEN then CLOSED |
| **P1-007** | Proposal section locking — User A locks section → User B gets 423 Locked; lock TTL expires → lock reacquirable by User B | API | R-011 | Assert `Retry-After` header on 423; fast-forward Redis TTL; assert User B can acquire |
| **P1-008** | AI draft generation SSE via KraftData mock — connect, receive heartbeats at 15s cadence, receive content chunks, receive completion event | API | R-001, R-003 | Requires TB-02; assert no chunk arrives after `event: done`; assert connection closed by server at 600s |
| **P1-009** | Redis Streams event delivery — publish `opportunities.ingested` event → notification consumer processes → alert matched → SendGrid API called with correct recipients | API | R-005, R-007 | Use SendGrid sandbox mode; assert email envelope fields match alert preferences |
| **P1-010** | ESPD profile auto-fill — profile created → KraftData ESPD agent called → response fields populated in DB correctly | API | R-018 | Requires TB-02; assert mandatory ESPD fields present; validate XML snippet is schema-valid |
| **P1-011** | Audit trail completeness — mutate proposal, company profile, user → verify `shared.audit_log` entry (actor, action, entity_type, entity_id, timestamp) | API | R-002 | Use direct DB assertion via test session; verify fire-and-forget does not block response |
| **P1-012** | Approval workflow state machine — task created → submitted → approved through all configured stages | API | R-011 | Assert invalid transitions return 422; assert final state idempotent on re-approval attempt |

**Total P1:** 12 tests

---

### P2 (Medium)

**Criteria:** Secondary features + Lower risk (1–3) + Edge cases + Regression prevention

| Test ID | Requirement | Test Level | Risk Link | Notes |
| :--- | :--- | :--- | :--- | :--- |
| **P2-001** | Redis Streams DLQ — inject poison message → verify DLQ routing after max retries; DLQ Prometheus counter increments | API | R-007 | Assert consumer group ACK is withheld; assert `dlq.messages.total` counter +1 |
| **P2-002** | Notification digest — daily digest Celery task assembles matched opportunities per user; SendGrid API called with correct recipient list | API | R-005 | Use Celery `apply()` in tests (not async worker); assert SendGrid sandbox receives correct envelope |
| **P2-003** | PDF/DOCX export — proposal export generates valid document; metadata correct; download presigned URL accessible | API | — | Assert MIME type; assert filename includes proposal title; assert URL expires |
| **P2-004** | Analytics materialized views — materialized view refresh → aggregate counts match source data | API | R-002 | Assert all view queries include `WHERE company_id = :cid` scoping (cross-tenant leakage check) |
| **P2-005** | iCal feed — tender deadlines appear in valid `.ics`; token auth enforced; correct VEVENT fields populated | API | R-009 | Assert `Content-Type: text/calendar`; assert unauthenticated request → 401; assert DTSTART / SUMMARY correct |
| **P2-006** | Task orchestration — task create → dependency chain → status transitions → completion signal | API | — | Assert dependent task cannot START while prerequisite is PENDING; assert completion emits Redis event |
| **P2-007** | Content blocks library — CRUD operations + full-text search + insertion into proposal generation context | API | — | Assert FTS returns block on keyword; assert block included in generation context payload |
| **P2-008** | Submission guide generation — portal-specific instructions generated via KraftData agent; response cached per opportunity | API | R-001 | Requires TB-02; assert second call returns cached response without calling agent again |

**Total P2:** 8 tests

---

### P3 (Low)

**Criteria:** Nice-to-have + Exploratory + Performance benchmarks + i18n / visual completeness

| Test ID | Requirement | Test Level | Notes |
| :--- | :--- | :--- | :--- |
| **P3-001** | UI visual regression — shadcn/ui design tokens, white-label branding overrides, mobile responsive breakpoints (375px, 768px, 1280px) | Component (Playwright) | Snapshot against baseline; diff threshold 0.1%; run nightly |
| **P3-002** | i18n completeness — all UI strings present in both BG and EN locale JSON; zero missing `next-intl` keys at runtime | Component (Playwright) | Assert no `[missing: ...]` rendered strings; check both `/bg/` and `/en/` route prefixes |
| **P3-003** | PostgreSQL FTS search latency < 200ms P95 under 50 concurrent users | Perf (k6) | Run nightly; 50 VUs, 5-minute ramp; assert P95 < 200ms, P99 < 500ms; assert zero 5xx |
| **P3-004** | Enterprise white-label — custom domain + logo + colour scheme applied consistently across all pages of both client and admin apps | E2E (Playwright) | Parameterize across 2 white-label configurations; assert CSS variable overrides applied |

**Total P3:** 4 tests

---

## Execution Strategy

**Philosophy:** Run the full functional suite in every PR using pytest + Playwright parallelization. Defer only genuinely infrastructure-heavy tests (k6 load, visual regression snapshots) to nightly.

### Every PR: pytest + Playwright (~15 min)

**All functional tests** (P0 through P2):

- pytest: unit + integration markers via `make test` (testcontainers spun up per session)
- Playwright: P0 and P1 E2E + API tests, sharded across 4 workers
- Stripe CLI sidecar active for billing tests
- KraftData mock (TB-02) active for AI tests
- Total: ~28 tests (P0 + P1 + P2 excluding P3-003)

**Why run in PRs:** pytest testcontainers are fast; Playwright API tests run in-process; no expensive k6 infrastructure needed.

### Nightly: k6 Performance + Visual Regression (~45 min)

**All performance and visual tests** (P3):

- k6: P3-003 FTS latency under 50 VUs (5-minute ramp; 5-minute steady state)
- Playwright: P3-001 visual regression snapshots; P3-002 i18n key completeness; P3-004 white-label E2E
- Total: 4 P3 tests

**Why defer to nightly:** k6 requires dedicated load infrastructure; visual snapshot diffs are noise-prone in PR context.

### Weekly: Security & Chaos (~2 hours)

**Special resilience and adversarial tests:**

- Stripe webhook replay and signature tampering suite (extended variants of P0-008)
- Redis Streams partition failure injection (extended DLQ chaos beyond P2-001)
- ClamAV timeout simulation (malformed archive stress beyond P0-007)
- JWT concurrent refresh race under load (extended P0-002 concurrency)

**Why defer to weekly:** Fault injection requires infrastructure-level control not available in standard CI.

**Manual tests (excluded from automation):**

- DevOps validation: Kubernetes rolling deploy, Helm rollback
- Finance validation: Stripe Dashboard subscription state reconciliation
- ESPD XML schema submission to test portal (requires portal sandbox access)

---

## QA Effort Estimate

**QA test development effort only** (excludes DevOps, Backend, Fintech work):

| Priority | Count | Effort Range | Notes |
| :--- | :--- | :--- | :--- |
| P0 | 8 | ~40–60 hours | Complex setup: Stripe CLI, ClamAV, SSE capture, cross-tenant fixtures |
| P1 | 12 | ~50–70 hours | RBAC matrix parametrization, circuit breaker simulation, audit trail assertions |
| P2 | 8 | ~20–30 hours | Edge cases, DLQ injection, export format validation |
| P3 | 4 | ~10–15 hours | k6 script, visual snapshots, i18n crawler |
| **Total** | **32** | **~120–175 hours** | **~3–5 weeks, 1 QA engineer full-time** |

**Assumptions:**

- Includes test design, implementation, debugging, and CI integration
- Excludes ongoing maintenance (~10% effort per sprint)
- Assumes TB-01 through TB-04 delivered on schedule
- Assumes `eusolicit-test-utils` factories available before P0 work begins

**Dependencies from other teams:**

- See "Dependencies & Test Blockers" section for TB-01 through TB-04 items blocking QA.

---

## Tooling & Access

| Tool or Service | Purpose | Access Required | Status |
| :--- | :--- | :--- | :--- |
| **pytest + eusolicit-test-utils** | Unit, integration, API tests with testcontainers and transaction-rollback fixtures | Repo access | Ready |
| **Playwright** | E2E and API-level tests; SSE stream capture; visual regression snapshots | Repo access; staging URL | Ready |
| **k6** | PostgreSQL FTS load test (P3-003); 50 VUs ramp | k6 Cloud or self-hosted runner | Pending (DevOps) |
| **Stripe CLI** | Webhook event simulation (TB-01); time-travel for trial expiry | Stripe Test Mode API key | Pending (Fintech) |
| **KraftData Mock Service** | Contract-faithful AI gateway mock with SSE and HMAC support (TB-02) | Internal service endpoint | Pending (AI Platform) |
| **ClamAV testcontainer** | EICAR test file scanning in CI (TB-04) | Docker Hub pull or internal registry | Pending (DevOps) |
| **SendGrid Sandbox** | Email delivery assertion without real sends | SendGrid API key (sandbox mode) | Pending (Backend) |
| **MinIO (local)** | S3-compatible presigned URL testing | `docker compose up` | Ready |

**Access requests needed:**

- [ ] Stripe Test Mode API key for CI/staging environments (TB-01)
- [ ] KraftData mock service endpoint and contract spec (TB-02)
- [ ] ClamAV testcontainer image pushed to internal registry (TB-04)
- [ ] SendGrid sandbox API key for CI environment

---

## Interworking & Regression

**Services impacted by system-level test changes:**

| Service / Component | Impact | Regression Scope | Validation Steps |
| :--- | :--- | :--- | :--- |
| **client-api (8000)** | Primary surface for auth, opportunities, proposals, RBAC | All P0 + P1 API tests | `make test-service SVC=client-api` before merge |
| **admin-api (8001)** | Audit trail, compliance, user management | P1-011 audit trail | `make test-service SVC=admin-api` before merge |
| **ai-gateway** | KraftData circuit breaker, SSE proxy, HMAC validation | P0-005, P0-008, P1-006, P1-008 | Requires TB-02 mock active |
| **notification** | Redis Streams consumer, Celery Beat digest, SendGrid dispatch | P1-009, P2-002 | `make test-service SVC=notification` nightly |
| **data-pipeline** | Crawler ingestion, relevance scoring, Redis publish | P1-009 event chain | Smoke test on event publish/consume round-trip |
| **Next.js client + admin frontends** | AuthGuard + middleware dual-layer auth, i18n routing | P0-001, P3-001, P3-002, P3-004 | `make test-e2e-chromium` before merge |
| **shared.audit_log** | Cross-service audit writes | P1-011 | Assert entries from client-api and admin-api mutations |

**Regression test strategy:**

- Every PR must pass `make quality-check` (ruff + mypy) and `make test-unit` before Playwright suite runs.
- Service-specific integration tests (`make test-service SVC=<name>`) are gated per service PR.
- Cross-service tests (`make test` with `cross_service` marker) run in nightly pipeline.
- E2E Playwright suite (`make test-e2e-chromium`) is the final gate before merge to `main`.

---

## Appendix A: Code Examples

### Example 1 — Cross-Tenant Isolation Test (pytest + eusolicit-test-utils)

```python
# tests/integration/test_cross_tenant_isolation.py
import pytest
from httpx import AsyncClient
from eusolicit_test_utils.factories import CompanyFactory, UserFactory, ProposalFactory
from eusolicit_test_utils.fixtures import auth_headers


@pytest.mark.integration
@pytest.mark.parametrize("resource_path", [
    "/api/v1/proposals/{id}",
    "/api/v1/opportunities/{id}",
    "/api/v1/documents/{id}",
])
async def test_tenant_b_cannot_access_tenant_a_resource(
    async_client: AsyncClient,
    db_session,
    resource_path: str,
):
    """Tenant B JWT must not read Tenant A resources — asserts 403 or 404 (no ID leakage)."""

    # Arrange: create two fully isolated tenants within the test transaction
    company_a = await CompanyFactory.create(session=db_session)
    user_a = await UserFactory.create(session=db_session, company_id=company_a.id)
    proposal_a = await ProposalFactory.create(session=db_session, company_id=company_a.id)

    company_b = await CompanyFactory.create(session=db_session)
    user_b = await UserFactory.create(session=db_session, company_id=company_b.id)

    # Act: Tenant B user attempts to access Tenant A resource
    headers_b = await auth_headers(user_b, db_session)
    url = resource_path.format(id=str(proposal_a.id))
    response = await async_client.get(url, headers=headers_b)

    # Assert: must be 403 or 404 — never 200 and never 500
    assert response.status_code in (403, 404), (
        f"Expected 403 or 404 for cross-tenant access, got {response.status_code}. "
        f"Tenant B ({company_b.id}) must not read Tenant A ({company_a.id}) resource."
    )
    # Assert no internal IDs leaked in error body
    body_text = response.text
    assert str(proposal_a.id) not in body_text, "Resource ID must not appear in error response (ID leakage)"
    assert str(company_a.id) not in body_text, "Company ID must not appear in error response"
```

### Example 2 — SSE Streaming Test Capture (pytest-asyncio + httpx)

```python
# tests/integration/test_sse_ai_summary.py
import asyncio
import pytest
import json
from httpx import AsyncClient
from eusolicit_test_utils.factories import CompanyFactory, UserFactory, OpportunityFactory
from eusolicit_test_utils.fixtures import auth_headers


@pytest.mark.integration
async def test_ai_summary_sse_stream_lifecycle(
    async_client: AsyncClient,
    db_session,
    kraftdata_mock,  # TB-02 fixture: starts KraftData mock service
    redis_client,
):
    """
    Verify the full SSE stream lifecycle for AI summary generation:
    - Server sends heartbeat every ~15s
    - Server sends content chunks
    - Server sends completion event
    - Usage counter increments in Redis after completion
    """
    # Arrange
    company = await CompanyFactory.create(session=db_session, tier="professional")
    user = await UserFactory.create(session=db_session, company_id=company.id)
    opportunity = await OpportunityFactory.create(session=db_session, company_id=company.id)
    headers = await auth_headers(user, db_session)

    # Configure mock to return: heartbeat → 3 content chunks → done event
    kraftdata_mock.configure_sse_sequence(
        agent="opportunity-summary",
        sequence=[
            {"event": "heartbeat", "data": ""},
            {"event": "chunk", "data": json.dumps({"text": "EU grant opportunity "})},
            {"event": "chunk", "data": json.dumps({"text": "for SMEs in renewable energy."})},
            {"event": "done", "data": json.dumps({"usage": {"tokens": 42}})},
        ],
    )

    # Act: stream SSE events
    events = []
    async with async_client.stream(
        "GET",
        f"/api/v1/opportunities/{opportunity.id}/ai-summary",
        headers=headers,
        timeout=30.0,
    ) as response:
        assert response.status_code == 200
        assert "text/event-stream" in response.headers["content-type"]

        async for line in response.aiter_lines():
            if line.startswith("event:"):
                current_event = line.split(":", 1)[1].strip()
            elif line.startswith("data:") and line != "data:":
                data = line.split(":", 1)[1].strip()
                events.append({"event": current_event, "data": data})
                if current_event == "done":
                    break

    # Assert event sequence
    event_types = [e["event"] for e in events]
    assert "heartbeat" in event_types, "SSE stream must include heartbeat events"
    assert event_types.count("chunk") >= 1, "SSE stream must include content chunks"
    assert event_types[-1] == "done", "SSE stream must terminate with 'done' event"

    # Assert chunks arrive before done
    chunk_indices = [i for i, e in enumerate(events) if e["event"] == "chunk"]
    done_index = event_types.index("done")
    assert all(i < done_index for i in chunk_indices), "All chunks must arrive before 'done' event"

    # Assert usage counter incremented in Redis
    usage_key = f"usage:{company.id}:ai_summary"
    counter = await redis_client.get(usage_key)
    assert counter is not None, "Usage counter must be set in Redis after stream completion"
    assert int(counter) >= 1, "Usage counter must have incremented"
```

---

## Appendix B: Knowledge Base References

Knowledge fragments consulted during this test design:

| Fragment | Purpose | Applied In |
| :--- | :--- | :--- |
| `risk-governance.md` | Risk identification methodology, category definitions (TECH/SEC/PERF/DATA/BUS/OPS) | Step 3: Risk Assessment |
| `probability-impact.md` | P×I scoring scale (1–3 each), threshold rules for High/Medium/Low | Step 3: Risk Assessment |
| `test-levels-framework.md` | E2E vs API vs Component vs Unit selection criteria | Step 4: Coverage Plan |
| `test-priorities-matrix.md` | P0–P3 classification rules; "blocks core + high risk + no workaround" for P0 | Step 4: Coverage Plan |
| `adr-quality-readiness-checklist.md` | Architecture quality gates required before test development begins | Step 2: Context Loading |
| `test-quality.md` | Anti-patterns (duplicate coverage, brittle selectors, non-deterministic assertions) | Step 4: Coverage Plan |
| `fixture-architecture.md` | Transaction rollback fixture design; session vs function scope decisions | Appendix A patterns |
| `data-factories.md` | Factory design for Company, User, Proposal, Opportunity, Subscription | Appendix A patterns |

---

**Generated by:** BMad TEA Agent
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
