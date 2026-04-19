---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-19'
workflowType: bmad-testarch-atdd
mode: story-level
storyId: 8-9-per-bid-add-on-purchase-flow
tddPhase: RED
detectedStack: fullstack
generationMode: ai-generation
executionMode: sequential
inputDocuments:
  - eusolicit-docs/implementation-artifacts/8-9-per-bid-add-on-purchase-flow.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - eusolicit-app/e2e/specs/billing-checkout.spec.ts
  - eusolicit-app/services/client-api/tests/unit/test_security.py
  - eusolicit-app/services/client-api/tests/unit/test_addon_checkout_service.py
  - eusolicit-app/services/client-api/tests/unit/test_addon_webhook_handler.py
  - eusolicit-app/services/client-api/tests/integration/test_addon_purchase_flow.py
  - eusolicit-docs/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 8.9 — Per-Bid Add-On Purchase Flow

**Date:** 2026-04-19
**Author:** BMad TEA Agent — Master Test Architect (bmad-testarch-atdd)
**TDD Phase:** 🔴 RED (failing tests — awaiting implementation)
**Story:** eusolicit-docs/implementation-artifacts/8-9-per-bid-add-on-purchase-flow.md
**Epic Test Design:** eusolicit-docs/test-artifacts/test-design-epic-08.md

---

## Preflight Summary

| Check | Status | Notes |
|-------|--------|-------|
| Stack detected | ✅ `fullstack` | Python/FastAPI/pytest backend + Next.js 14/Playwright frontend |
| Story status | ✅ `ready-for-dev` | AC1–AC10 fully specified with implementation code |
| Story has clear acceptance criteria | ✅ 10 ACs | All ACs fully specified |
| Test framework (backend) | ✅ pytest + pytest-asyncio | Existing conftest.py patterns confirmed |
| Test framework (E2E) | ✅ Playwright | `billing-checkout.spec.ts` already exists from Story 8.6 |
| `tests/unit/` directory (client-api) | ✅ exists | `test_security.py`, `test_rbac.py`, etc. confirmed |
| `tests/integration/` directory (client-api) | ✅ exists | `test_002_migration.py`, etc. confirmed |
| `e2e/specs/billing-checkout.spec.ts` | ✅ exists | Prior billing E2E tests present from Stories 8.6–8.8 |
| `create_addon_checkout_session()` | ❌ NOT IMPLEMENTED | Defined in story Task 2 — ATDD red phase |
| `_addon_price_map()` | ❌ NOT IMPLEMENTED | Defined in story Task 1 — ATDD red phase |
| `STRIPE_PRICE_ADDON_*` settings fields | ❌ NOT IMPLEMENTED | Defined in story Task 1 — ATDD red phase |
| `POST /api/v1/billing/addon/checkout/session` | ❌ NOT IMPLEMENTED | Defined in story Task 3 — ATDD red phase |
| `GET /api/v1/billing/addon/status` | ❌ NOT IMPLEMENTED | Defined in story Task 3 — ATDD red phase |
| `_handle_addon_checkout_completed()` | ❌ NOT IMPLEMENTED | Defined in story Task 4 — ATDD red phase |
| `checkout.session.completed` webhook branch | ❌ NOT IMPLEMENTED | Defined in story Task 4 — ATDD red phase |
| `AddOnPurchase` ORM model | ✅ EXISTS | Defined in Story 8.2 — `client_api.models.add_on_purchase` |
| `add_on_purchases` UNIQUE constraint | ⚠️ CHECK REQUIRED | Story Task 5 must verify migration 029 if missing |
| `AddOnPurchaseButton` component | ❌ NOT IMPLEMENTED | Defined in story Task 8 — ATDD red phase |
| `messages/bg.json` + `messages/en.json` billing keys | ❌ NOT IMPLEMENTED | 5 new keys per story Task 8 — ATDD red phase |
| Epic test design (E08) | ✅ LOADED | test-design-epic-08.md — test ID 8.9-E2E-001 (P1) |

---

## Generation Mode

**Mode selected:** AI Generation (sequential)

Fullstack project. Backend tests use Python/pytest with `@pytest.mark.skip` (guarded
imports for unimplemented modules). E2E tests use Playwright `test.skip()` added to the
existing `billing-checkout.spec.ts`. No browser recording needed — tests generated from
story acceptance criteria and the implementation code scaffolds in Tasks 1–10.

---

## 🔴 TDD Red Phase: Failing Tests Generated

All Python test functions are decorated with `@pytest.mark.skip(reason="S08.09 not yet implemented — ATDD red phase")`.
All Playwright test functions use `test.skip(...)`.

**Why `@pytest.mark.skip` (not `@pytest.mark.xfail`):**
The unimplemented modules (`billing_service.create_addon_checkout_session`,
`webhook_service._handle_addon_checkout_completed`, new billing endpoints) would cause
`ImportError` or `404` failures at collection/request time. Tests use guarded imports
(`try/except ImportError`) to prevent CI collection errors while clearly documenting
the expected behavior via assertions.

---

## Test Strategy

### AC Coverage → Test Level Mapping

| AC | Description | Test Level | Test IDs / File | Priority |
|----|-------------|------------|-----------------|----------|
| AC1 | Checkout session endpoint (auth, role gate, response) | Unit + Integration | `TestCreateAddonCheckoutSession`, `TestAddonCheckoutEndpointRBAC` | P1 |
| AC2 | Add-on price config; 422 when not configured | Unit + Integration | `TestAddonPriceMap`, `TestAddonPriceSettings`, `test_addon_checkout_endpoint_missing_price_config_returns_422` | P2 |
| AC3 | `checkout.session.completed` webhook handler | Unit + Integration | `TestHandleAddonCheckoutCompleted`, `TestProcessWebhookRouting`, `TestWebhookAddonCheckoutCompleted` | P1 |
| AC4 | Add-on status query endpoint | Integration | `TestAddonStatusEndpoint` | P1 |
| AC5 | `add_on_purchases` DB constraints (UNIQUE + index) | Integration | `test_webhook_addon_checkout_completed_is_idempotent` | P2 |
| AC6 | Frontend: Purchase Add-On buttons | E2E | `8.9-E2E-001` in `billing-checkout.spec.ts` | P1 |
| AC7 | Frontend: Success return flow (toast, URL clean, badge) | E2E | `8.9-E2E-001` in `billing-checkout.spec.ts` | P1 |
| AC8 | Tier-gating (Included in Plan badge) | E2E | `8.9-AC8` in `billing-checkout.spec.ts` | P2 |
| AC9 | Audit trail (write_audit_entry; fire-and-forget) | Unit | `test_handle_addon_checkout_completed_writes_audit_entry`, `test_handle_addon_checkout_audit_failure_does_not_raise` | P2 |
| AC10 | i18n (all new keys in bg.json + en.json) | E2E | `8.9-AC10` in `billing-checkout.spec.ts` | P3 |

---

## Generated Test Files

### 1. Backend Unit Tests (🔴 RED)

**File:** `eusolicit-app/services/client-api/tests/unit/test_addon_checkout_service.py`

| Test Class | Test Name | AC | Priority | Failure Reason |
|------------|-----------|-----|----------|----------------|
| `TestCreateAddonCheckoutSession` | `test_create_addon_checkout_session_returns_checkout_url` | AC1 | P1 | `ImportError`: function not implemented |
| `TestCreateAddonCheckoutSession` | `test_create_addon_checkout_session_no_subscription_returns_error` | AC1 | P1 | `ImportError`: function not implemented |
| `TestCreateAddonCheckoutSession` | `test_create_addon_checkout_session_unknown_addon_type_returns_error` | AC2 | P2 | `ImportError`: function not implemented |
| `TestCreateAddonCheckoutSession` | `test_create_addon_checkout_session_stripe_error_returns_error` | AC1 | P2 | `ImportError`: function not implemented |
| `TestCreateAddonCheckoutSession` | `test_create_addon_checkout_session_embeds_metadata` | AC1 | P1 | `ImportError`: function not implemented |
| `TestAddonPriceMap` | `test_addon_price_map_returns_only_configured_entries` | AC2 | P2 | `ImportError`: function not implemented |
| `TestAddonPriceMap` | `test_addon_price_map_returns_empty_when_all_none` | AC2 | P2 | `ImportError`: function not implemented |
| `TestAddonPriceSettings` | `test_client_api_settings_has_addon_price_fields` | AC2 | P2 | `AssertionError`: fields missing from `ClientApiSettings` |

**File:** `eusolicit-app/services/client-api/tests/unit/test_addon_webhook_handler.py`

| Test Class | Test Name | AC | Priority | Failure Reason |
|------------|-----------|-----|----------|----------------|
| `TestHandleAddonCheckoutCompleted` | `test_handle_addon_checkout_completed_inserts_add_on_purchase_row` | AC3 | P1 | `ImportError`: function not implemented |
| `TestHandleAddonCheckoutCompleted` | `test_handle_addon_checkout_completed_is_idempotent` | AC3 | P2 | `ImportError`: function not implemented |
| `TestHandleAddonCheckoutCompleted` | `test_handle_addon_checkout_completed_writes_audit_entry` | AC9 | P2 | `ImportError`: function not implemented |
| `TestHandleAddonCheckoutCompleted` | `test_handle_addon_checkout_audit_failure_does_not_raise` | AC9 | P2 | `ImportError`: function not implemented |
| `TestHandleAddonCheckoutCompleted` | `test_handle_addon_checkout_completed_missing_metadata_returns_gracefully` | AC3 | P2 | `ImportError`: function not implemented |
| `TestProcessWebhookRouting` | `test_process_webhook_routes_addon_checkout_event` | AC3 | P1 | `ImportError`: branch not in process_stripe_webhook |
| `TestProcessWebhookRouting` | `test_process_webhook_ignores_subscription_checkout_session` | AC3 | P2 | `ImportError`: branch not in process_stripe_webhook |
| `TestProcessWebhookRouting` | `test_process_webhook_ignores_non_addon_payment_checkout` | AC3 | P2 | `ImportError`: branch not in process_stripe_webhook |

### 2. Backend Integration Tests (🔴 RED)

**File:** `eusolicit-app/services/client-api/tests/integration/test_addon_purchase_flow.py`

| Test Class | Test Name | AC | Epic Test ID | Priority | Failure Reason |
|------------|-----------|-----|-------------|----------|----------------|
| `TestAddonStatusEndpoint` | `test_addon_status_returns_not_purchased_when_no_row` | AC4 | — | P1 | `404`: endpoint not implemented |
| `TestAddonStatusEndpoint` | `test_addon_status_returns_purchased_after_insert` | AC4 | 8.9-API-002 | P1 | `404`: endpoint not implemented |
| `TestAddonStatusEndpoint` | `test_addon_status_requires_authentication` | AC4 | — | P1 | `404`: endpoint not implemented |
| `TestAddonCheckoutEndpointRBAC` | `test_addon_checkout_endpoint_requires_bid_manager_role` | AC1 | — | P1 | `404`: endpoint not implemented |
| `TestAddonCheckoutEndpointRBAC` | `test_addon_checkout_endpoint_contributor_role_returns_403` | AC1 | — | P1 | `404`: endpoint not implemented |
| `TestAddonCheckoutEndpointRBAC` | `test_addon_checkout_endpoint_bid_manager_returns_checkout_url` | AC1 | — | P1 | `404`: endpoint not implemented |
| `TestAddonCheckoutEndpointRBAC` | `test_addon_checkout_endpoint_missing_price_config_returns_422` | AC2 | — | P2 | `404`: endpoint not implemented |
| `TestWebhookAddonCheckoutCompleted` | `test_webhook_addon_checkout_completed_creates_add_on_purchase_row` | AC3 | 8.9-API-001 | P1 | `AssertionError`: no row created |
| `TestWebhookAddonCheckoutCompleted` | `test_webhook_addon_checkout_completed_is_idempotent` | AC3+AC5 | 8.9-UNIT-001 | P2 | `AssertionError`: duplicate row or wrong count |

### 3. Frontend E2E Tests (🔴 RED) — Added to `billing-checkout.spec.ts`

**File:** `eusolicit-app/e2e/specs/billing-checkout.spec.ts`

| Test Name | AC | Epic Test ID | Priority | Failure Reason |
|-----------|-----|-------------|----------|----------------|
| `P1 8.9-E2E-001: per-bid add-on purchase completes via Checkout and unlocks feature for opportunity` | AC6, AC7 | 8.9-E2E-001 | P1 | `Error`: `AddOnPurchaseButton` component not found on opportunity page |
| `8.9-AC8: enterprise tier shows "Included in Plan" badge instead of purchase button` | AC8 | — | P2 | `Error`: Add-on section not rendered on opportunity detail page |
| `8.9-AC10: add-on buttons use i18n keys (no hardcoded English in BG locale)` | AC10 | — | P3 | `Error`: component not rendered |

---

## Acceptance Criteria Coverage Map

| AC | Covered By | Status |
|----|-----------|--------|
| AC1 — Checkout session endpoint | `test_create_addon_checkout_session_*`, `TestAddonCheckoutEndpointRBAC` | ✅ RED |
| AC2 — Price config + 422 | `TestAddonPriceMap`, `TestAddonPriceSettings`, `test_addon_checkout_endpoint_missing_price_config_returns_422` | ✅ RED |
| AC3 — Webhook handler + idempotency | `TestHandleAddonCheckoutCompleted`, `TestProcessWebhookRouting`, `TestWebhookAddonCheckoutCompleted` | ✅ RED |
| AC4 — Status query endpoint | `TestAddonStatusEndpoint` | ✅ RED |
| AC5 — DB constraints (UNIQUE + index) | `test_webhook_addon_checkout_completed_is_idempotent` | ✅ RED |
| AC6 — Frontend purchase buttons | `8.9-E2E-001` | ✅ RED |
| AC7 — Success return flow | `8.9-E2E-001` | ✅ RED |
| AC8 — Tier-gating UI | `8.9-AC8` | ✅ RED |
| AC9 — Audit trail | `test_handle_addon_checkout_completed_writes_audit_entry`, `test_handle_addon_checkout_audit_failure_does_not_raise` | ✅ RED |
| AC10 — i18n | `8.9-AC10` | ✅ RED |

---

## TDD Red Phase Summary

```
🔴 TDD RED PHASE: Failing Tests Generated

📊 Summary:
- Total Tests: 19 (all skipped/RED phase)
  - Backend Unit Tests: 13 tests (all @pytest.mark.skip)
    - test_addon_checkout_service.py: 8 tests (AC1, AC2)
    - test_addon_webhook_handler.py: 8 tests (AC3, AC9)
  - Backend Integration Tests: 9 tests (all @pytest.mark.skip)
    - test_addon_purchase_flow.py: 9 tests (AC1–AC5)
  - E2E Tests: 3 tests added to billing-checkout.spec.ts (all test.skip)
    - 8.9-E2E-001 (P1), 8.9-AC8 (P2), 8.9-AC10 (P3)

📋 All tests assert EXPECTED behavior
📋 All tests will FAIL until feature implemented
📋 This is INTENTIONAL (TDD red phase)
```

---

## Epic Test Design Alignment

| Epic Test ID | Level | Priority | Risk | ATDD Coverage |
|-------------|-------|----------|------|----------------|
| **8.9-E2E-001** | E2E | P1 | — | `P1 8.9-E2E-001` in `billing-checkout.spec.ts` |
| **8.9-API-001** (derived) | Integration | P1 | — | `test_webhook_addon_checkout_completed_creates_add_on_purchase_row` |
| **8.9-API-002** (derived) | Integration | P1 | — | `test_addon_status_returns_purchased_after_insert` |
| **8.9-UNIT-001** (derived) | Unit | P2 | R-001 | `test_handle_addon_checkout_completed_is_idempotent` + `test_webhook_addon_checkout_completed_is_idempotent` |
| **8.9-UNIT-002** (derived) | Unit | P2 | — | `test_process_webhook_ignores_subscription_checkout_session` |

**R-001 Mitigation (Webhook race / duplicate records):**
Covered by two idempotency tests:
1. Unit: `test_handle_addon_checkout_completed_is_idempotent` — verifies `ON CONFLICT DO NOTHING` path skips audit
2. Integration: `test_webhook_addon_checkout_completed_is_idempotent` — verifies exactly 1 row after 2 identical webhook posts

---

## Risk Assessment

| Risk | Category | Mitigation in Tests |
|------|----------|---------------------|
| Duplicate add-on purchase rows (R-001 derivative) | DATA | Idempotency tests at unit + integration level |
| Stripe price IDs hardcoded in source | BUS | `test_addon_price_map_returns_only_configured_entries` verifies env var → price mapping |
| Webhook audit failure blocks response | BUS | `test_handle_addon_checkout_audit_failure_does_not_raise` verifies fire-and-forget |
| Subscription mode misrouted as add-on | DATA | `test_process_webhook_ignores_subscription_checkout_session` |
| RBAC bypass on checkout endpoint | SEC | `TestAddonCheckoutEndpointRBAC` covers read_only + contributor + bid_manager |

---

## Implementation Checklist (For Developer)

Use this as a verification guide when implementing Story 8.9 Tasks:

### Backend Tasks

- [ ] **Task 1**: Add `stripe_price_addon_*` fields to `ClientApiSettings`
  - Verified by: `test_client_api_settings_has_addon_price_fields`
  - Verified by: `test_addon_price_map_*`

- [ ] **Task 2**: Implement `create_addon_checkout_session()` + `_addon_price_map()` in `billing_service.py`
  - Verified by: all `TestCreateAddonCheckoutSession` tests

- [ ] **Task 3**: Add `AddOnCheckoutRequest`, `POST /billing/addon/checkout/session`, `GET /billing/addon/status` to `billing.py`
  - Verified by: `TestAddonCheckoutEndpointRBAC`, `TestAddonStatusEndpoint`

- [ ] **Task 4**: Add `_handle_addon_checkout_completed()` + `checkout.session.completed` branch to `webhook_service.py`
  - Verified by: `TestHandleAddonCheckoutCompleted`, `TestProcessWebhookRouting`, `TestWebhookAddonCheckoutCompleted`

- [ ] **Task 5**: Verify/create migration 029 for `add_on_purchases` constraints
  - Verified by: `test_webhook_addon_checkout_completed_is_idempotent` (integration)

### Frontend Tasks

- [ ] **Task 8**: Create `AddOnPurchaseButton.tsx` + i18n keys in `bg.json`/`en.json`
  - Verified by: `8.9-AC10` E2E test

- [ ] **Task 9**: Integrate add-on section into opportunity detail page
  - Verified by: `8.9-E2E-001`, `8.9-AC8` E2E tests

---

## Next Steps (TDD Green Phase)

After implementing the feature:

### Backend

```bash
# Remove @pytest.mark.skip from test files, then run:
cd eusolicit-app

# Unit tests
pytest services/client-api/tests/unit/test_addon_checkout_service.py -v
pytest services/client-api/tests/unit/test_addon_webhook_handler.py -v

# Integration tests (requires DB + testcontainers)
pytest services/client-api/tests/integration/test_addon_purchase_flow.py -v

# Full billing suite regression check
pytest services/client-api/tests/unit/test_billing*.py \
       services/client-api/tests/unit/test_webhook*.py \
       services/client-api/tests/unit/test_addon*.py \
       services/client-api/tests/integration/test_addon*.py -v
```

### Frontend E2E

```bash
# Remove test.skip from 8.9-* tests in billing-checkout.spec.ts, then run:
make test-e2e-chromium
# or: npx playwright test e2e/specs/billing-checkout.spec.ts --project=chromium
```

### Activation Order

1. Implement Task 1 → activate `TestAddonPriceSettings` + `TestAddonPriceMap`
2. Implement Task 2 → activate `TestCreateAddonCheckoutSession`
3. Implement Task 3 → activate `TestAddonStatusEndpoint` + `TestAddonCheckoutEndpointRBAC`
4. Implement Task 4 → activate `TestHandleAddonCheckoutCompleted` + `TestProcessWebhookRouting` + `TestWebhookAddonCheckoutCompleted`
5. Implement Task 5 → `test_webhook_addon_checkout_completed_is_idempotent` integration test
6. Implement Tasks 8–9 → activate E2E tests in `billing-checkout.spec.ts`

---

## Validation

| Check | Status |
|-------|--------|
| All Python tests have `@pytest.mark.skip` | ✅ |
| All Playwright tests use `test.skip()` | ✅ |
| All tests assert expected behavior (not placeholders) | ✅ |
| No `expect(True).toBe(True)` or `assert True` placeholders | ✅ |
| Guarded imports (`try/except ImportError`) on all service imports | ✅ |
| Priority tags (P1/P2/P3) assigned | ✅ |
| AC1–AC10 all covered | ✅ |
| Epic test IDs 8.9-E2E-001, 8.9-API-001, 8.9-API-002 explicitly covered | ✅ |
| R-001 idempotency mitigation covered | ✅ |
| No hardcoded Stripe price IDs in tests | ✅ |
| `asyncio.to_thread` pattern used for Stripe SDK calls | ✅ (mocked via `patch("asyncio.to_thread", ...)`) |
| `mode='payment'` vs `mode='subscription'` discrimination tested | ✅ |
| Audit trail fire-and-forget behavior tested | ✅ |

---

**Generated by:** BMad TEA Agent — Master Test Architect (bmad-testarch-atdd)
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
