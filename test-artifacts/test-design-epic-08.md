---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-18'
workflowType: 'testarch-test-design'
inputDocuments: ['epic-08-subscription-billing.md', 'test-design-architecture.md', 'config.yaml']
---

# Test Design: Epic 08 — Subscription & Billing

**Date:** 2026-04-18
**Author:** BMad TEA Agent
**Status:** Draft
**Sprint:** 9–10 | **Points:** 55 | **Dependencies:** E02 (Auth/Registration), E06 (Feature Gating)

---

## Executive Summary

**Scope:** Epic-Level test design for Epic 08 — Subscription & Billing

Epic 08 introduces the full billing lifecycle using Stripe as the payment backbone: tiered subscription management (Free / Starter / Professional / Enterprise), a 14-day no-card trial, per-bid add-on purchases, EU VAT handling via Stripe Tax + VIES, enterprise custom invoicing, Redis-backed usage metering, and cache-coherent tier gating. The architecture integrates with two already-shipped epics (E02 Registration, E06 Tier Gating), relying on Stripe webhooks and Redis Streams as the event spine.

**Risk Summary:**

- Total risks identified: 8
- High-priority risks (≥6): 4
- Critical categories: DATA, BUS, TECH, SEC

**Coverage Summary:**

- P0 scenarios: 7 (~14–20 hours)
- P1 scenarios: 9 (~9–15 hours)
- P2 scenarios: 8 (~6–10 hours)
- P3 scenarios: 2 (~2–4 hours)
- **Total**: 26 tests, **~31–49 hours** (~4–6 days, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
| --- | --- | --- |
| **Stripe Platform Uptime** | Third-party infrastructure outside our control | Handled by Stripe's SLA; monitor via Stripe Statuspage |
| **VIES External Service Uptime** | Third-party EU service known for occasional downtime | Implement async retry with fallback to manual VAT review (tested separately) |
| **Stripe Dashboard Configuration** | Configuring the Customer Portal via Stripe Dashboard is manual | Treat as prerequisite; document required portal configuration |
| **Browser-level Payment Form** | Stripe-hosted Checkout UI is outside our DOM | Validate redirect URL correctness and webhook-driven state, not Stripe's own form |

---

## Risk Assessment

### High-Priority Risks (Score ≥6) — MITIGATE before release

| Risk ID | Category | Description | Story | Probability | Impact | Score | Mitigation | Owner | Timeline |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| R-001 | DATA | Webhook race conditions causing duplicate subscriptions or dropped events | S08.04 | 3 | 2 | **6** | Idempotency via `webhook_events` deduplication table; wrap processing in DB transaction that fails on duplicate `stripe_event_id` | Backend | Sprint 9 |
| R-002 | BUS | EU VAT miscalculation or VIES validation blocking legitimate checkouts | S08.10 | 2 | 3 | **6** | VIES fallback on 503 (flag for manual review, do not block registration); exhaustive reverse-charge matrix tests | Fintech | Sprint 10 |
| R-003 | TECH | Redis usage metering drift vs. Stripe usage records leading to over/under-billing | S08.08 | 2 | 3 | **6** | Atomic `INCR` operations; nightly Celery reconciliation task; post-sync diff check | Backend | Sprint 10 |
| R-004 | SEC | Stripe webhook signature bypass — missing or bypassable HMAC validation allows fraudulent event injection | S08.04 | 2 | 3 | **6** | Enforce `stripe.Webhook.construct_event` on every request; test with tampered signatures | Backend | Sprint 9 |

### Medium-Priority Risks (Score 3–5) — MONITOR

| Risk ID | Category | Description | Story | Probability | Impact | Score | Mitigation | Owner |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| R-005 | SEC | Trial manipulation — company registers multiple times to obtain repeated trials | S08.03 | 2 | 2 | **4** | DB unique constraint on one trial per `company_id`; test attempted second-trial creation | Backend |
| R-006 | TECH | Cache invalidation failure on tier change leaves stale tier in middleware, granting unauthorized access to premium features | S08.14 | 2 | 2 | **4** | Redis Streams `subscription.changed` consumer invalidates cache; DB fallback on cache miss (never trust stale cache) | Backend |

### Low-Priority Risks (Score 1–2) — DOCUMENT

| Risk ID | Category | Description | Story | Probability | Impact | Score | Action |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R-007 | TECH | Stripe API version drift — a pinned Stripe client version mismatch causes webhook payload changes to break event parsing | S08.04 | 1 | 2 | **2** | Pin `stripe` SDK version in `pyproject.toml`; monitor Stripe changelog | Monitor |
| R-008 | OPS | Enterprise custom invoice delayed sending or amount mismatch | S08.11 | 1 | 2 | **2** | Admin review workflow; `invoice.paid` webhook sync test | Monitor |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

---

## Entry Criteria

- [ ] Stripe Test Mode API keys (`STRIPE_TEST_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`) provisioned in staging environment
- [ ] Stripe CLI installed in CI for local webhook forwarding
- [ ] Redis cluster available in test environment (for usage metering and Streams)
- [ ] E02 Registration flow deployed (trial provisioning hook depends on it)
- [ ] E06 Tier Gating middleware deployed (downgrade enforcement and cache invalidation depend on it)
- [ ] DB migrations for `subscriptions`, `tier_access_policies`, `add_on_purchases`, `webhook_events` tables applied
- [ ] `tier_access_policies` seeded with all four tier definitions

## Exit Criteria

- [ ] All P0 tests passing with no exceptions
- [ ] All P1 tests passing or failures triaged with approved waivers
- [ ] No open HIGH-severity bugs (severity ≥ critical)
- [ ] Webhook idempotency tests validate zero duplicate records under concurrent replay
- [ ] Usage metering reconciles accurately (Redis count matches Stripe usage records after Celery sync)
- [ ] R-001 through R-004 mitigations verified by passing tests
- [ ] Trial manipulation (R-005) rejected at DB constraint layer
- [ ] Cache invalidation (R-006) verified with end-to-end tier change test

---

## Test Coverage Plan

### P0 (Critical) — Run on every commit

**Criteria:** Blocks core billing journey + High risk (≥6) + No workaround

| Story | Requirement | Test ID | Test Level | Risk Link | Test Count | Owner | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| S08.03 | Auto-provision 14-day Professional trial on new company registration | 8.3-E2E-001 | E2E | R-005 | 1 | QA | No payment method; assert `status: trialing` in DB + Redis Stream event published |
| S08.04 | Webhook signature verification rejects tampered or missing signatures | 8.4-API-001 | API | R-004 | 1 | QA | Send request with invalid `Stripe-Signature` header; expect 400 |
| S08.04 | Webhook `customer.subscription.created` syncs tier and dates to local DB | 8.4-API-002 | API | R-001 | 1 | QA | Fire Stripe test event; verify `subscriptions` row upserted correctly |
| S08.04 | Webhook `customer.subscription.updated` reflects plan change in local DB | 8.4-API-003 | API | R-001 | 1 | QA | Fire upgrade event; verify tier field updated |
| S08.04 | Webhook `invoice.payment_failed` marks subscription `past_due` | 8.4-API-004 | API | R-001 | 1 | QA | Fire failed invoice event; verify `status: past_due` |
| S08.05 | Trial expiry downgrades company to Free tier and enforces feature limits | 8.5-E2E-001 | E2E | — | 1 | QA | Simulate trial-end webhook; assert Free-tier limits active, data preserved |
| S08.06 | Upgrade via Stripe Checkout redirects and unlocks premium features on success | 8.6-E2E-001 | E2E | R-006 | 1 | QA | Mock Checkout session; assert success page shows confirmation, tier updated |

**Total P0:** 7 tests, **~14–20 hours**

---

### P1 (High) — Run on PR to main

**Criteria:** Important features + Medium risk (3–5) + Common workflows

| Story | Requirement | Test ID | Test Level | Risk Link | Test Count | Owner | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| S08.04 | Webhook idempotency — replaying the same `stripe_event_id` produces no duplicate records | 8.4-API-005 | API | R-001 | 1 | QA | POST same webhook payload twice; assert exactly one record in DB |
| S08.07 | Stripe Customer Portal session URL is generated and returned to authenticated user | 8.7-API-001 | API | — | 1 | QA | Authenticated request; assert response contains valid Stripe portal URL |
| S08.08 | Redis usage counter increments atomically for AI summaries, proposal drafts, compliance checks | 8.8-API-001 | API | R-003 | 1 | QA | Call `usage.increment()` N times; verify Redis `INCR` key == N |
| S08.08 | `/api/v1/subscription/usage` returns current consumption vs. tier limits | 8.8-API-002 | API | R-003 | 1 | QA | Authenticated request; assert consumed and limit fields match DB and Redis |
| S08.09 | Per-bid add-on purchase completes via Checkout and unlocks feature for specific opportunity | 8.9-E2E-001 | E2E | — | 1 | QA | Mock `checkout.session.completed`; assert `add_on_purchases` row inserted, feature accessible |
| S08.10 | VIES-valid VAT number syncs to Stripe customer tax_ids and stores `status: valid` locally | 8.10-API-001 | API | R-002 | 1 | QA | Provide valid EU VAT; mock VIES success; assert Stripe and DB updated |
| S08.10 | VIES-invalid VAT number returns validation error to client | 8.10-API-002 | API | R-002 | 1 | QA | Provide invalid VAT; assert 422 with clear error |
| S08.10 | VIES service downtime does not block registration; fallback flag is set | 8.10-API-003 | API | R-002 | 1 | QA | Mock VIES 503; assert registration succeeds, flag `vat_validation_status: pending` |
| S08.14 | `subscription.changed` Redis Stream event triggers tier cache invalidation in middleware | 8.14-INT-001 | Integration | R-006 | 1 | QA | Trigger tier change; assert Redis cache key evicted; next request reads from DB |

**Total P1:** 9 tests, **~9–15 hours**

---

### P2 (Medium) — Run nightly/weekly

**Criteria:** Secondary features + Low risk (1–2) + Edge cases

| Story | Requirement | Test ID | Test Level | Risk Link | Test Count | Owner | Notes |
| --- | --- | --- | --- | --- | --- | --- | --- |
| S08.01 | Idempotent Stripe customer creation — duplicate calls do not create multiple Stripe customers | 8.1-API-001 | API | — | 1 | DEV | Call provisioning twice with same `company_id`; assert single customer in Stripe mock |
| S08.03 | Second trial attempt is rejected with clear error | 8.3-API-001 | API | R-005 | 1 | DEV | Attempt trial provisioning on company that already has a subscription record |
| S08.05 | `customer.subscription.trial_will_end` webhook triggers `trial.expiring` event to notification service | 8.5-UNIT-001 | Unit | — | 1 | DEV | Mock time; fire webhook; assert notification event published |
| S08.06 | Proration on mid-cycle downgrade is handled by Stripe; local subscription state reflects new tier | 8.6-API-001 | API | — | 1 | DEV | Fire `customer.subscription.updated` with downgrade; assert local tier field |
| S08.08 | Celery daily sync task reports Redis counters as Stripe usage records | 8.8-UNIT-001 | Unit | R-003 | 1 | DEV | Mock Stripe API; run task; assert `stripe.SubscriptionItem.create_usage_record` called with correct values |
| S08.08 | Period rollover resets Redis usage counters and preserves prior period totals | 8.8-UNIT-002 | Unit | R-003 | 1 | DEV | Advance billing period; verify counter keys reset; prior period archived |
| S08.11 | Admin can create Enterprise custom invoice with NET 30 terms via admin API | 8.11-API-001 | API | R-008 | 1 | QA | Admin creates invoice; assert Stripe draft invoice created with `days_until_due: 30` |
| S08.12 | Pricing page renders all four tier comparison rows with correct feature limits from API | 8.12-COMP-001 | Component | — | 1 | DEV | Render page with mocked `tier_access_policies` response; assert tier names and feature limits visible |

**Total P2:** 8 tests, **~6–10 hours**

---

### P3 (Low) — Run on-demand

**Criteria:** Nice-to-have + Exploratory + Performance benchmarks

| Story | Requirement | Test ID | Test Level | Test Count | Owner | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| S08.08 | 10,000 concurrent usage INCR operations produce accurate Redis count | 8.8-PERF-001 | API (k6) | 1 | QA | k6 script; assert Redis count == 10,000 after concurrent run; no dropped increments |
| S08.02 | Alembic migrations apply cleanly with all indexes and FK constraints verified | 8.2-INT-001 | Integration | 1 | DEV | Apply migrations to clean schema; inspect `information_schema` for expected indexes and FK |

**Total P3:** 2 tests, **~2–4 hours**

---

## Execution Order

### Smoke Tests (<5 min)

**Purpose:** Fast feedback — catch broken Stripe connectivity or misconfigured environment

- [ ] Stripe Test connection health check (verify API key reachability)
- [ ] Company registration creates Stripe customer and trial subscription (S08.01 + S08.03)

**Total:** 2 scenarios

---

### P0 Tests (<15 min)

**Purpose:** Critical path and high-risk scenario validation

- [ ] 8.3-E2E-001 — Trial auto-provisioned on registration (E2E)
- [ ] 8.4-API-001 — Invalid webhook signature rejected (API)
- [ ] 8.4-API-002 — `customer.subscription.created` syncs to DB (API)
- [ ] 8.4-API-003 — `customer.subscription.updated` reflects tier change (API)
- [ ] 8.4-API-004 — `invoice.payment_failed` marks `past_due` (API)
- [ ] 8.5-E2E-001 — Trial expiry downgrades to Free tier (E2E)
- [ ] 8.6-E2E-001 — Checkout upgrade unlocks premium features (E2E)

**Total:** 7 scenarios

---

### P1 Tests (<30 min)

**Purpose:** Important feature coverage including idempotency, metering, add-ons, VAT, cache

- [ ] 8.4-API-005 — Webhook idempotency: no duplicate records on replay (API)
- [ ] 8.7-API-001 — Customer Portal URL generated for authenticated user (API)
- [ ] 8.8-API-001 — Redis usage counter increments atomically (API)
- [ ] 8.8-API-002 — Usage endpoint returns consumed vs. limit (API)
- [ ] 8.9-E2E-001 — Per-bid add-on purchase unlocks feature (E2E)
- [ ] 8.10-API-001 — Valid VAT synced to Stripe and stored (API)
- [ ] 8.10-API-002 — Invalid VAT returns 422 (API)
- [ ] 8.10-API-003 — VIES downtime does not block registration (API)
- [ ] 8.14-INT-001 — `subscription.changed` invalidates tier cache (Integration)

**Total:** 9 scenarios

---

### P2/P3 Tests (<60 min)

**Purpose:** Full regression, edge cases, performance

- [ ] 8.1-API-001 — Idempotent Stripe customer creation (API)
- [ ] 8.3-API-001 — Second trial rejected (API)
- [ ] 8.5-UNIT-001 — Trial reminder email event triggered at T-3 days (Unit)
- [ ] 8.6-API-001 — Proration state correct on downgrade (API)
- [ ] 8.8-UNIT-001 — Celery sync reports usage to Stripe (Unit)
- [ ] 8.8-UNIT-002 — Period rollover resets counters (Unit)
- [ ] 8.11-API-001 — Enterprise invoice creation with NET 30 (API)
- [ ] 8.12-COMP-001 — Pricing page tier comparison renders correctly (Component)
- [ ] 8.8-PERF-001 — 10k concurrent INCR accuracy (k6)
- [ ] 8.2-INT-001 — DB migration schema validation (Integration)

**Total:** 10 scenarios

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test (range) | Total Hours (range) | Notes |
| --- | --- | --- | --- | --- |
| P0 | 7 | 2.0–3.0 | **14–20** | Stripe mock setup, webhook simulation, E2E Checkout flow |
| P1 | 9 | 1.0–1.5 | **9–15** | API contracts, Redis validation, VIES mock |
| P2 | 8 | 0.75–1.25 | **6–10** | Unit mocks, component render, Celery task stubs |
| P3 | 2 | 1.0–2.0 | **2–4** | k6 script, schema inspection |
| **Total** | **26** | — | **~31–49** | **~4–6 QA days** |

### Prerequisites

**Test Data:**

- Stripe Test Mode cards (success: `4242 4242 4242 4242`; decline: `4000 0000 0000 0002`)
- EU VAT numbers: valid (e.g., `DE123456789`), invalid format, VIES-offline simulation
- Test Stripe Price IDs for Free, Starter, Professional, Enterprise
- Test company factory (`CompanyFactory`) with `stripe_customer_id` field

**Tooling:**

- **Stripe CLI** — webhook forwarding (`stripe listen`) and test event firing (`stripe trigger`)
- **Playwright** — E2E Checkout redirect flows, subscription management page, trial banner
- **pytest + testcontainers** — API and integration tests (DB + Redis)
- **k6** — Usage metering load test (P3)
- **pytest-mock** — VIES service mocking, Celery task isolation

**Environment:**

- Staging with `STRIPE_TEST_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` configured
- Redis 7 available (for usage counters and Streams consumer tests)
- Celery worker or task-executor available for beat task validation

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions; release blocked until fixed)
- **P1 pass rate:** ≥95% (written waiver with owner and expiry required for any failure)
- **P2/P3 pass rate:** ≥90% (informational; track as tech debt)
- **High-risk mitigations (R-001 to R-004):** 100% complete or formally waived before GA

### Coverage Targets

- **Critical billing paths (trial, upgrade, downgrade, webhook sync):** ≥80%
- **Security scenarios (SEC category — R-004, R-005):** 100%
- **Business logic (BUS — R-002 VAT matrix):** ≥80%
- **Edge cases (idempotency, fallbacks, period rollover):** ≥60%

### Non-Negotiable Requirements

- [ ] All P0 tests pass
- [ ] No HIGH-severity (≥6) risk items unmitigated or unwaived
- [ ] Stripe webhook signature validation enforced and passing (R-004)
- [ ] Idempotency tests confirm zero duplicate records under concurrent webhook replay (R-001)
- [ ] Stripe Test Mode keys are never cross-pollinated with production config
- [ ] Payment gateway integration verified only against Stripe Test Mode (never live keys)

---

## Mitigation Plans

### R-001: Webhook race conditions causing duplicate subscriptions or dropped events (Score: 6)

**Mitigation Strategy:** Log `stripe_event_id` into a `webhook_events` DB table on first receipt. Wrap all event processing in a DB transaction that fails gracefully (idempotent upsert) on duplicate event ID. Use `SELECT FOR UPDATE` to prevent concurrent processing of the same event.
**Owner:** Backend
**Timeline:** Sprint 9
**Status:** Planned
**Verification:** Fire identical webhook payloads concurrently (10 threads, same `stripe_event_id`); assert exactly one `subscriptions` row and one `webhook_events` row created. Zero errors in application logs.

---

### R-002: EU VAT miscalculation or VIES blocking legitimate checkouts (Score: 6)

**Mitigation Strategy:** Implement async VIES validation with 3-retry exponential backoff. If VIES returns 503 or times out after retries, set `vat_validation_status: pending` (do NOT block registration) and flag for async manual review. Test B2B reverse-charge scenario: valid VAT from another EU country → Stripe Tax applies 0%.
**Owner:** Fintech / Backend
**Timeline:** Sprint 10
**Status:** Planned
**Verification:** (a) Mock VIES 503 → registration succeeds, `vat_validation_status: pending` in DB. (b) Valid DE VAT + German billing address → Stripe Tax rate = 0% (reverse charge). (c) Valid DE VAT + B2C customer → normal VAT rate applied.

---

### R-003: Redis usage metering drift vs. Stripe usage records (Score: 6)

**Mitigation Strategy:** Use Redis `INCR` (atomic) for all counter operations. Nightly Celery Beat task reads all active counters and calls `stripe.SubscriptionItem.create_usage_record` with `action: increment`. After sync, compare Redis value with Stripe usage total; alert on diff > 0.
**Owner:** Backend
**Timeline:** Sprint 10
**Status:** Planned
**Verification:** Simulate 10,000 concurrent `usage.increment()` calls across 10 processes. After Celery sync, assert `Redis counter == Stripe usage record`. Repeat for period-rollover boundary (T+1s past period end).

---

### R-004: Stripe webhook signature bypass (Score: 6)

**Mitigation Strategy:** Enforce `stripe.Webhook.construct_event(payload, sig_header, STRIPE_WEBHOOK_SECRET)` on every incoming webhook request — including local development. Reject any request that raises `stripe.error.SignatureVerificationError` with HTTP 400. Unit test signature validation in isolation.
**Owner:** Backend
**Timeline:** Sprint 9
**Status:** Planned
**Verification:** (a) Send request with valid signature → processed normally. (b) Send request with tampered body (valid sig for original) → rejected 400. (c) Send request with no `Stripe-Signature` header → rejected 400.

---

## Assumptions and Dependencies

### Assumptions

1. Blocker B-01 from the System-Level design (Stripe Mocking Strategy) is resolved: Stripe CLI is available in CI/CD for webhook event simulation, and `stripe-mock` (or Stripe Test Mode) is used for unit-level API responses.
2. The `tier_access_policies` table is seeded with all four tiers before any billing test runs.
3. Stripe Customer Portal is configured via the Stripe Dashboard (plan changes, cancellation, payment method updates) as a one-time setup prerequisite — not automated by this epic.
4. VIES API rate limits are accommodated in the implementation via retry logic and circuit-breaker pattern.

### Dependencies

1. **E02 Registration Flow** — Stripe customer provisioning and trial creation hook into the post-registration pipeline; E02 must be deployed.
2. **E06 Tier Gating Middleware** — Trial expiry downgrade and cache-invalidation logic depends on the E06 middleware reading tier from DB on cache miss.
3. **Redis Streams** — `subscription.changed` and `trial.expiring` events require Redis 7 Streams configured; the notification service must have a consumer group registered.

### Risks to Plan

- **Risk:** Stripe API version bump mid-sprint changes webhook payload structure (R-007).
  - **Impact:** Event parsing breaks silently; subscription sync produces incorrect tier records.
  - **Contingency:** Pin `stripe==X.Y.Z` in all service `pyproject.toml` files; add a changelog review step to the sprint kick-off checklist.

- **Risk:** VIES integration delay causes S08.10 to slip to Sprint 11.
  - **Impact:** EU VAT tests blocked; cannot validate reverse-charge scenarios.
  - **Contingency:** Stub VIES behind a feature flag (`VIES_ENABLED=false`); run VAT tests against stub. Promote to live VIES when available.

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
| --- | --- | --- |
| **Auth Service (E02)** | Registration hook triggers Stripe customer + trial provisioning | E02 Registration E2E: assert company created with `stripe_customer_id` and `status: trialing` |
| **Feature Gating Middleware (E06)** | Tier upgrades unlock features; trial expiry enforces Free limits | E06 Tier Limits: re-run tier access tests with subscriptions table populated |
| **Opportunity Detail Page (E06 UI)** | Add-on purchase buttons appear for opportunities when feature not included in tier | Opportunity Detail page: verify buttons visible below-tier, hidden above-tier |
| **Notification Service** | `trial.expiring` event triggers reminder email; `subscription.changed` consumed by email template | Notification: assert event published on Redis Streams; email smoke test |
| **AI Gateway / Usage Endpoints** | Usage metering counters are incremented by AI summary and proposal draft requests | AI Gateway: call AI summary endpoint, verify Redis counter incremented |

---

## Acceptance Criteria Coverage Map

Every acceptance criterion from the epic is mapped to at least one test:

| AC | Description | Test IDs | Priority |
| --- | --- | --- | --- |
| AC-01 | New company creates Stripe customer + 14-day Professional trial (no card) | 8.3-E2E-001 | P0 |
| AC-02 | One trial per company; duplicate attempts rejected | 8.3-API-001 | P2 |
| AC-03 | Trial expiry → Free tier, data preserved, premium gated | 8.5-E2E-001 | P0 |
| AC-04 | `customer.subscription.trial_will_end` → reminder email event at T-3 | 8.5-UNIT-001 | P2 |
| AC-05 | Upgrade/downgrade via Stripe Checkout redirect and success/cancel pages | 8.6-E2E-001 | P0 |
| AC-06 | Customer Portal link accessible for self-service | 8.7-API-001 | P1 |
| AC-07 | All subscription webhook events sync to `subscriptions` table correctly | 8.4-API-002, 8.4-API-003, 8.4-API-004, 8.4-API-005 | P0/P1 |
| AC-08 | Per-bid add-on via Checkout; `checkout.session.completed` creates record + unlocks feature | 8.9-E2E-001 | P1 |
| AC-09 | EU VAT via Stripe Tax; tax ID synced and validated via VIES | 8.10-API-001, 8.10-API-002, 8.10-API-003 | P1 |
| AC-10 | Enterprise custom invoices via admin API with NET 30/60 terms | 8.11-API-001 | P2 |
| AC-11 | Usage metering via Redis INCR; daily Celery sync to Stripe; `/subscription/usage` endpoint | 8.8-API-001, 8.8-API-002, 8.8-UNIT-001, 8.8-UNIT-002, 8.8-PERF-001 | P1/P2/P3 |
| AC-12 | `subscription.changed` on Redis Streams on every tier change → cache invalidation | 8.14-INT-001 | P1 |
| AC-13 | Pricing page with accurate tier comparison table | 8.12-COMP-001 | P2 |
| AC-14 | Trial banner shows days remaining + upgrade CTA during trialing status | 8.13-E2E (covered in 8.6-E2E-001 setup) | P1 (bundled) |
| AC-15 | Subscription management page: tier, usage meters, portal link, billing history | 8.8-API-002 (API); expand to E2E in ATDD phase | P1 |

---

## Follow-on Workflows (Manual)

- **`bmad-testarch-atdd`** — Generate failing P0 acceptance tests from this design (run after implementation stubs are available; not auto-run by this workflow).
- **`bmad-testarch-automate`** — Expand test automation coverage once implementation exists (particularly P2 Celery tasks and P3 k6 load test).
- **`bmad-testarch-nfr`** — Assess NFRs for billing: payment latency SLA, webhook processing throughput, Redis counter performance at scale.

---

## Approval

**Test Design Approved By:**

- [ ] Product Manager: — Date: —
- [ ] Tech Lead: — Date: —
- [ ] QA Lead: — Date: —

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification framework (P×I scoring, gate decisions)
- `probability-impact.md` — Risk scoring methodology (1-3 scale, DOCUMENT/MONITOR/MITIGATE/BLOCK thresholds)
- `test-levels-framework.md` — Test level selection (Unit / Integration / API / E2E / Component)
- `test-priorities-matrix.md` — P0–P3 prioritization rules

### Related Documents

- Epic: `eusolicit-docs/planning-artifacts/epic-08-subscription-billing.md`
- Architecture: `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md`
- System-Level Test Design: `eusolicit-docs/test-artifacts/test-design-architecture.md`
- Project Context: `eusolicit-docs/project-context.md`

---

**Generated by:** BMad TEA Agent — Test Architect Module
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
