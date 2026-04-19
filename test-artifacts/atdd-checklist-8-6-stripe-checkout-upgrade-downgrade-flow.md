---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-19'
workflowType: 'testarch-atdd'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/8-6-stripe-checkout-upgrade-downgrade-flow.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-08.md'
  - 'eusolicit-docs/_bmad/bmm/config.yaml'
  - 'eusolicit-app/services/client-api/tests/unit/test_billing_service.py'
  - 'eusolicit-app/services/client-api/tests/unit/test_webhook_service.py'
  - 'eusolicit-app/services/client-api/tests/integration/test_stripe_webhook_flow.py'
  - 'eusolicit-app/services/client-api/tests/integration/test_trial_expiry_flow.py'
  - 'eusolicit-app/services/client-api/src/client_api/api/v1/billing.py'
  - 'eusolicit-app/services/client-api/src/client_api/services/billing_service.py'
---

# ATDD Checklist: Story 8.6 — Stripe Checkout Upgrade/Downgrade Flow

**Date:** 2026-04-19
**Author:** BMad TEA Agent (bmad-testarch-atdd)
**Status:** 🔴 RED PHASE — All tests generated as failing (TDD red phase)
**Story:** `8-6-stripe-checkout-upgrade-downgrade-flow`
**Epic:** Epic 08 — Subscription & Billing

---

## Step 1: Preflight & Context

### Stack Detection

| Item | Value |
|---|---|
| Detected stack | `fullstack` (Python FastAPI + Next.js) |
| Effective test stack for this story | `fullstack` (backend endpoints + frontend pages both introduced) |
| Backend test framework | pytest + pytest-asyncio + pytest-mock |
| Frontend test framework | Playwright (E2E) |
| Test marker for skip (backend) | `@pytest.mark.skip(reason="🔴 RED PHASE...")` |
| Test marker for skip (frontend) | `test.skip(...)` |

### Prerequisites Verified

- [x] Story status: `ready-for-dev` with clear acceptance criteria (10 ACs)
- [x] Test framework: `conftest.py` confirmed in `services/client-api/tests/`
- [x] Existing patterns loaded: `test_billing_service.py`, `test_webhook_service.py`, `test_stripe_webhook_flow.py`, `test_trial_expiry_flow.py`
- [x] Epic test design loaded: `test-design-epic-08.md` — P0 test `8.6-E2E-001` (R-006), P2 test `8.6-API-001` identified
- [x] Current `billing.py` inspected — confirmed only `POST /billing/webhooks/stripe` exists; GET /subscription and POST /checkout/session absent
- [x] Current `billing_service.py` inspected — confirmed `create_checkout_session()` not yet implemented

### Inputs Loaded

| Artifact | Purpose |
|---|---|
| `8-6-stripe-checkout-upgrade-downgrade-flow.md` | Story spec, 10 ACs, tasks, dev notes |
| `test-design-epic-08.md` | Epic-level coverage strategy, risk links (R-006), P0/P2 test IDs |
| `billing.py` (current state) | Code baseline — confirms GET/POST endpoints absent |
| `billing_service.py` (current state) | Code baseline — confirms `create_checkout_session()` absent |
| `test_billing_service.py` | Unit test patterns (factory helpers, async mock patterns) |
| `test_webhook_service.py` | Unit test patterns for webhook service |
| `test_stripe_webhook_flow.py` | Integration test patterns (seed SQL, ASGITransport, httpx, DB verify) |
| `test_trial_expiry_flow.py` | Integration test patterns (session_factory fixture, multi-session verify) |

---

## Step 2: Generation Mode

**Mode selected:** AI generation (fullstack — backend tests generated directly; E2E tests generated from story spec and AC descriptions)

**Rationale:** Story 8.6 is a fullstack story. The backend portions (AC1–3, AC7–9) are expressible as pure unit and integration tests. The frontend portions (AC4–6, AC10) are page/component behaviors that require E2E Playwright tests. Because the frontend pages do not yet exist (RED phase), browser recording is not applicable — E2E tests are generated from AC specifications with `test.skip` scaffolds and Playwright route interception patterns.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenario Mapping

| AC | Description | Test ID | Level | Priority | Why It Will Fail |
|---|---|---|---|---|---|
| AC1 | POST /checkout/session → checkout_url + session_id returned | `test_create_checkout_session_returns_url` | Unit | P0 | `create_checkout_session()` does not exist in billing_service.py |
| AC1 | POST endpoint returns 200 + checkout_url | `test_post_checkout_session_returns_checkout_url` | Unit | P0 | `POST /checkout/session` endpoint does not exist in billing.py |
| AC1 | success_url contains `{CHECKOUT_SESSION_ID}` placeholder | `test_create_checkout_session_success_url_contains_session_id_placeholder` | Unit | P0 | `create_checkout_session()` does not exist |
| AC2 | Missing stripe_customer_id → billing_not_configured | `test_create_checkout_session_missing_customer_returns_error` | Unit | P0 | Service function does not exist |
| AC2 | No subscription row → billing_not_configured | `test_create_checkout_session_missing_subscription_returns_error` | Unit | P0 | Service function does not exist |
| AC2 | 422 response when billing_not_configured | `test_post_checkout_session_returns_422_on_billing_not_configured` | Unit | P0 | Endpoint does not exist |
| AC3 | Unknown tier → tier_not_configured | `test_create_checkout_session_unknown_tier_returns_error` | Unit | P0 | Service function does not exist |
| AC3 | 422 response when tier_not_configured | `test_post_checkout_session_returns_422_on_tier_not_configured` | Unit | P0 | Endpoint does not exist |
| AC4 | Billing settings page calls API, redirects to Stripe | `P0 8.6-E2E-001` (in E2E file) | E2E | P0 (R-006) | Frontend page does not exist |
| AC5 | Success page polls for tier change, shows confirmation | `8.6-E2E-001` + `AC5 timeout test` | E2E | P0 | Frontend page does not exist |
| AC6 | Cancel page shows cancellation toast + back CTA | `AC6 cancel page test` | E2E | P2 | Frontend page does not exist |
| AC7 | GET /subscription returns current state | `test_get_subscription_returns_current_tier` | Unit | P1 | Endpoint does not exist |
| AC7 | GET /subscription → 404 when no row | `test_get_subscription_returns_404_when_no_row` | Unit | P1 | Endpoint does not exist |
| AC7 | GET /subscription requires auth | `test_get_subscription_requires_auth` | Unit | P1 | Endpoint does not exist |
| AC8 | stripe.api_key guard returns billing_not_configured | `test_create_checkout_session_no_stripe_api_key_returns_error` | Unit | P2 | Service function does not exist |
| AC8 | Proration: downgrade webhook updates local tier | `test_checkout_session_downgrade_webhook_updates_tier` | Integration | P2 (8.6-API-001) | Story 8.6 Tasks 1–3 not implemented |
| AC9 | Audit trail written on checkout session creation | `test_create_checkout_session_writes_audit_entry` | Unit | P2 | Service function does not exist |
| AC1 resilience | StripeError caught, returns error dict | `test_create_checkout_session_stripe_error_returns_error` | Unit | P2 | Service function does not exist |
| Security | POST /checkout/session requires auth | `test_post_checkout_session_requires_auth` | Unit | P1 | Endpoint does not exist |
| Pydantic | Invalid tier → 422 before service called | `test_post_checkout_session_invalid_tier_returns_422` | Unit | P1 | Endpoint/model does not exist |
| AC10 | i18n — BG locale, no hardcoded English | `AC10 i18n test` | E2E | P2 | Frontend page does not exist |

---

## Step 4: Generated Test Files

### 4.1 Test File Summary

| File | Level | Test Count | Tests | Priority |
|---|---|---|---|---|
| `services/client-api/tests/unit/test_checkout_session.py` | Unit | 15 | Service + endpoint tests | P0–P2 |
| `services/client-api/tests/integration/test_checkout_upgrade_flow.py` | Integration | 1 | 8.6-API-001 downgrade webhook | P2 |
| `e2e/specs/billing-checkout.spec.ts` | E2E | 4 | 8.6-E2E-001 + AC5/AC6/AC10 | P0–P2 |

**Total tests generated: 20** (all in RED PHASE with skip decorators)

---

### 4.2 Unit Tests: `test_checkout_session.py`

#### Class: `TestCreateCheckoutSession` (7 tests)

| Test | AC | Priority | Why It Fails |
|---|---|---|---|
| `test_create_checkout_session_returns_url` | AC1 | P0 | `create_checkout_session()` not in billing_service.py |
| `test_create_checkout_session_missing_customer_returns_error` | AC2 | P0 | Function absent |
| `test_create_checkout_session_missing_subscription_returns_error` | AC2 | P0 | Function absent |
| `test_create_checkout_session_unknown_tier_returns_error` | AC3 | P0 | Function absent |
| `test_create_checkout_session_stripe_error_returns_error` | AC1 resilience | P2 | Function absent |
| `test_create_checkout_session_writes_audit_entry` | AC9 | P2 | Function absent |
| `test_create_checkout_session_success_url_contains_session_id_placeholder` | AC1 | P0 | Function absent |
| `test_create_checkout_session_no_stripe_api_key_returns_error` | AC8 | P2 | Function absent |

#### Class: `TestGetSubscriptionEndpoint` (3 tests)

| Test | AC | Priority | Why It Fails |
|---|---|---|---|
| `test_get_subscription_returns_current_tier` | AC7 | P1 | `GET /subscription` endpoint absent from billing.py |
| `test_get_subscription_returns_404_when_no_row` | AC7 | P1 | Endpoint absent |
| `test_get_subscription_requires_auth` | AC7 | P1 | Endpoint absent |

#### Class: `TestCreateCheckoutSessionEndpoint` (5 tests)

| Test | AC | Priority | Why It Fails |
|---|---|---|---|
| `test_post_checkout_session_returns_checkout_url` | AC1 | P0 | `POST /checkout/session` absent from billing.py |
| `test_post_checkout_session_returns_422_on_billing_not_configured` | AC2 | P0 | Endpoint absent |
| `test_post_checkout_session_returns_422_on_tier_not_configured` | AC3 | P0 | Endpoint absent |
| `test_post_checkout_session_requires_auth` | Security | P1 | Endpoint absent |
| `test_post_checkout_session_invalid_tier_returns_422` | AC1 (Pydantic) | P1 | `CheckoutSessionRequest` model absent |

---

### 4.3 Integration Tests: `test_checkout_upgrade_flow.py`

#### Class: `TestCheckoutDowngradeViaWebhook` (1 test)

| Test | AC | Epic Test ID | Priority | Why It Fails |
|---|---|---|---|---|
| `test_checkout_session_downgrade_webhook_updates_tier` | AC8 | 8.6-API-001 | P2 | Story 8.6 Tasks 1–3 not implemented (skip guard) |

**Note:** This test exercises the webhook handler from Story 8.4 — it will pass once Story 8.6 Tasks 1–3 are complete and the skip decorator is removed. The underlying webhook logic (tier mapping via `stripe_starter_price_id`) was implemented in Story 8.4 and should work correctly with the mock settings provided.

---

### 4.4 E2E Tests: `billing-checkout.spec.ts`

| Test | AC | Epic Test ID | Priority | Why It Fails |
|---|---|---|---|---|
| `P0 8.6-E2E-001: upgrade via Stripe Checkout redirects and shows success banner` | AC4, AC5 | 8.6-E2E-001 (R-006) | P0 | `/settings/billing/page.tsx` absent; `/settings/billing/success/page.tsx` absent |
| `AC5: success page shows processing message after polling timeout` | AC5 | — | P1 | `/settings/billing/success/page.tsx` absent |
| `AC6: cancel page shows cancellation message and back-to-billing CTA` | AC6 | — | P2 | `/settings/billing/cancel/page.tsx` absent |
| `AC10: billing settings page renders with BG locale translations` | AC10 | — | P2 | `/settings/billing/page.tsx` absent |

---

## Step 5: Validation & Completion

### Validation Checklist

- [x] All acceptance criteria (AC1–AC10) mapped to at least one test
- [x] All P0 tests from epic test design covered (8.6-E2E-001)
- [x] P2 test from epic test design covered (8.6-API-001)
- [x] All tests decorated with skip markers (RED phase compliance)
- [x] Unit tests: imports inside test bodies (ImportError on RED phase)
- [x] Integration test: follows `test_stripe_webhook_flow.py` patterns (raw SQL seed, httpx, ASGITransport, `session_factory` fixture)
- [x] E2E tests: `test.skip(...)` scaffold with documented steps and route mocking
- [x] Rich assertion messages with implementation task references
- [x] No passing tests (all RED phase)
- [x] No orphaned browser sessions (no recording sessions used)
- [x] Test files saved to correct locations (not temp dirs)
- [x] Checklist saved to `test-artifacts/`

### Coverage Map: Acceptance Criteria

| AC | Test IDs | Level | Priority | Status |
|---|---|---|---|---|
| AC1 — Checkout Session creation | `test_create_checkout_session_returns_url`, `test_post_checkout_session_returns_checkout_url`, `test_create_checkout_session_success_url_contains_session_id_placeholder`, `test_create_checkout_session_stripe_error_returns_error` | Unit | P0/P2 | 🔴 RED |
| AC2 — Missing customer guard | `test_create_checkout_session_missing_customer_returns_error`, `test_create_checkout_session_missing_subscription_returns_error`, `test_post_checkout_session_returns_422_on_billing_not_configured` | Unit | P0 | 🔴 RED |
| AC3 — Unknown tier guard | `test_create_checkout_session_unknown_tier_returns_error`, `test_post_checkout_session_returns_422_on_tier_not_configured` | Unit | P0 | 🔴 RED |
| AC4 — Frontend redirect flow | `8.6-E2E-001` (E2E) | E2E | P0 (R-006) | 🔴 RED |
| AC5 — Success page polling | `8.6-E2E-001` + AC5 timeout test | E2E | P0/P1 | 🔴 RED |
| AC6 — Cancel page toast | AC6 cancel test | E2E | P2 | 🔴 RED |
| AC7 — Subscription status endpoint | `test_get_subscription_returns_current_tier`, `test_get_subscription_returns_404_when_no_row`, `test_get_subscription_requires_auth` | Unit | P1 | 🔴 RED |
| AC8 — Proration (api_key guard) | `test_create_checkout_session_no_stripe_api_key_returns_error` | Unit | P2 | 🔴 RED |
| AC8 — Proration (webhook downgrade) | `test_checkout_session_downgrade_webhook_updates_tier` (8.6-API-001) | Integration | P2 | 🔴 RED |
| AC9 — Audit trail | `test_create_checkout_session_writes_audit_entry` | Unit | P2 | 🔴 RED |
| AC10 — i18n | AC10 i18n E2E test | E2E | P2 | 🔴 RED |

### Known Gaps / Assumptions

1. **`create_checkout_session_svc` import alias** — Unit tests for `TestCreateCheckoutSessionEndpoint` mock `client_api.api.v1.billing.create_checkout_session_svc`. This requires the developer to import `create_checkout_session` from `billing_service` as `create_checkout_session_svc` in `billing.py` (Task 3.3). The mock patch path must match this alias exactly.

2. **`async_session_factory` vs `session_factory`** — The integration test uses `session_factory` fixture (consistent with `test_stripe_webhook_flow.py`). Some newer tests use `async_session_factory`. If conftest is updated to rename this fixture, update the integration test accordingly.

3. **Frontend `data-testid` attributes** — E2E tests use `data-testid` selectors (`current-plan`, `upgrade-btn-professional`, `billing-success-loading`, `billing-success-banner`, etc.). Developer must add these attributes to the React components (Tasks 5–7). The test selectors serve as the contract for component implementation.

4. **Stripe redirect interception** — The E2E test for `8.6-E2E-001` mocks `POST /api/v1/billing/checkout/session` to return a local success URL instead of a real Stripe URL. This avoids needing Stripe CLI or an actual Stripe Test Mode in the E2E environment.

5. **`sessionStorage` prior-tier pattern** — The AC5 timeout test calls `page.evaluate()` to seed `eusolicit-checkout-prior-tier` in sessionStorage. Developer must ensure the success page reads this key to detect tier change (Task 6.2–6.3).

6. **Company FK for integration tests** — The integration test seeds companies via raw SQL with `ON CONFLICT (id) DO NOTHING`. This requires the `client.companies` table to have a `name` column. If the schema differs, update the seed SQL accordingly.

---

## Completion Summary

### Files Created

| File | Type | Tests | Status |
|---|---|---|---|
| `eusolicit-app/services/client-api/tests/unit/test_checkout_session.py` | Unit | 15 | 🔴 Created (RED) |
| `eusolicit-app/services/client-api/tests/integration/test_checkout_upgrade_flow.py` | Integration | 1 | 🔴 Created (RED) |
| `eusolicit-app/e2e/specs/billing-checkout.spec.ts` | E2E | 4 | 🔴 Created (RED) |

### ATDD Checklist Output

- **Path:** `eusolicit-docs/test-artifacts/atdd-checklist-8-6-stripe-checkout-upgrade-downgrade-flow.md`
- **Status:** Complete

### Key Risks

| Risk | Mitigation |
|---|---|
| R-006 (Cache invalidation on tier change) | Covered by 8.6-E2E-001 (P0) — tier change confirmed via polling before success banner shown |
| `create_checkout_session_svc` alias mismatch | Developer must follow Task 3.3 import alias; unit test mock path hard-wires the alias |
| E2E `data-testid` attributes not added | Developer must add `data-testid` attributes per the test contracts in `billing-checkout.spec.ts` |
| Stripe `{CHECKOUT_SESSION_ID}` double-brace escaping | Unit test `test_create_checkout_session_success_url_contains_session_id_placeholder` enforces the correct single-brace form |

### Next Recommended Workflow

- **`bmad-agent-dev`** — Implement Story 8.6 Tasks 1–10 (backend endpoints + frontend pages)
- **`bmad-testarch-automate`** — Expand E2E coverage once pages are built (activate `test.skip` scaffolds)
- **`bmad-testarch-nfr`** — Assess NFRs for checkout flow: redirect latency, polling frequency, session timeout handling

---

**Generated by:** BMad TEA Agent — ATDD Module
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
