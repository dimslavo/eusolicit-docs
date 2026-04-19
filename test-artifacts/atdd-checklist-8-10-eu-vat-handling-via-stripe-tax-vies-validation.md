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
storyId: 8-10-eu-vat-handling-via-stripe-tax-vies-validation
tddPhase: RED
detectedStack: fullstack
generationMode: ai-generation
executionMode: sequential
inputDocuments:
  - eusolicit-docs/implementation-artifacts/8-10-eu-vat-handling-via-stripe-tax-vies-validation.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - eusolicit-app/services/client-api/tests/unit/test_billing_service.py
  - eusolicit-app/services/client-api/tests/unit/test_billing_webhook_endpoint.py
  - eusolicit-app/services/client-api/tests/unit/test_checkout_session.py
  - eusolicit-app/services/client-api/tests/integration/test_stripe_webhook_flow.py
  - eusolicit-app/services/client-api/tests/integration/test_stripe_customer_provisioning.py
  - eusolicit-app/e2e/specs/billing-checkout.spec.ts
  - eusolicit-docs/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 8.10 — EU VAT Handling via Stripe Tax & VIES Validation

**Date:** 2026-04-19
**Author:** BMad TEA Agent — Master Test Architect (bmad-testarch-atdd)
**TDD Phase:** 🔴 RED (failing tests — awaiting implementation)
**Story:** eusolicit-docs/implementation-artifacts/8-10-eu-vat-handling-via-stripe-tax-vies-validation.md
**Epic Test Design:** eusolicit-docs/test-artifacts/test-design-epic-08.md

---

## Preflight Summary

| Check | Status | Notes |
|-------|--------|-------|
| Stack detected | ✅ `fullstack` | Python 3.12/FastAPI/pytest backend + Next.js 14/Playwright frontend |
| Story status | ✅ `ready-for-dev` | AC1–AC10 fully specified with implementation scaffolds |
| Story has clear acceptance criteria | ✅ 10 ACs | All ACs fully specified |
| Test framework (backend) | ✅ pytest + pytest-asyncio | Existing conftest.py + testcontainers patterns confirmed |
| Test framework (E2E) | ✅ Playwright | billing-checkout.spec.ts exists from Story 8.6 |
| `tests/unit/` directory (client-api) | ✅ exists | test_billing_service.py, test_checkout_session.py, etc. confirmed |
| `tests/integration/` directory (client-api) | ✅ exists | test_stripe_webhook_flow.py, etc. confirmed |
| `e2e/specs/` directory | ✅ exists | billing-checkout.spec.ts present from Stories 8.6–8.9 |
| `vies_service.py` | ❌ NOT IMPLEMENTED | Defined in story Task 2 — ATDD red phase |
| `ViesValidationResult` dataclass | ❌ NOT IMPLEMENTED | Defined in story Task 2 — ATDD red phase |
| `_parse_vat_number()` | ❌ NOT IMPLEMENTED | Defined in story Task 2 — ATDD red phase |
| Migration `030_vat_validation_status.py` | ❌ NOT IMPLEMENTED | Defined in story Task 1 — ATDD red phase |
| `Company.vat_validation_status` ORM field | ❌ NOT IMPLEMENTED | Defined in story Task 1.2 — ATDD red phase |
| `POST /api/v1/billing/vat/validate` | ❌ NOT IMPLEMENTED | Defined in story Task 3 — ATDD red phase |
| `VatValidateRequest` Pydantic model | ❌ NOT IMPLEMENTED | Defined in story Task 3.1 — ATDD red phase |
| `automatic_tax` on checkout sessions | ❌ NOT IMPLEMENTED | Defined in story Task 4 — ATDD red phase |
| Company profile VAT background hook | ❌ NOT IMPLEMENTED | Defined in story Task 5 — ATDD red phase |
| `CompanyProfileResponse.vat_validation_status` | ❌ NOT IMPLEMENTED | Defined in story Task 5.4 — ATDD red phase |
| `VatNumberInput` component | ❌ NOT IMPLEMENTED | Defined in story Task 8 — ATDD red phase |
| i18n keys (bg.json + en.json) | ❌ NOT IMPLEMENTED | 9 keys per story Task 8.3/8.4 — ATDD red phase |
| Company profile page VatNumberInput integration | ❌ NOT IMPLEMENTED | Defined in story Task 9 — ATDD red phase |
| Epic test design (E08) | ✅ LOADED | test-design-epic-08.md — 8.10-API-001/002/003 (P1, R-002) |

---

## Generation Mode

**Mode selected:** AI Generation (sequential)

Fullstack project with Python/FastAPI backend and Next.js 14 frontend. Backend tests use
Python/pytest with `@pytest.mark.skip` (guarded imports for unimplemented modules). E2E tests
use Playwright `test.skip()` in new `billing-vat.spec.ts`. No browser recording needed —
tests generated from story ACs and the implementation scaffolds in Tasks 1–10.

---

## Test Strategy

### Acceptance Criteria → Test Level Mapping

| AC | Description | Test Level | Priority | Test File |
|----|-------------|-----------|----------|-----------|
| AC1 | `automatic_tax={"enabled": True}` on checkout sessions | Unit | P2 | `test_checkout_stripe_tax.py` |
| AC2 | Migration 030: `vat_validation_status` column | Integration (migration) | P3 | (covered by migration framework) |
| AC3 | VIES service: parse, validate, retry, never raise | Unit | P1 | `test_vies_service.py` |
| AC4 | `POST /api/v1/billing/vat/validate` endpoint | Unit + Integration | P1 | `test_vat_validate_endpoint.py`, `test_vat_validation_flow.py` |
| AC5 | Company profile PUT/PATCH triggers background VAT validation | Integration | P1 | `test_vat_validation_flow.py` |
| AC6 | `CompanyProfileResponse` includes `vat_validation_status` | Unit | P2 | (in `test_vat_validate_endpoint.py`) |
| AC7 | `VatNumberInput` component with status indicator | E2E | P1 | `billing-vat.spec.ts` |
| AC8 | Company profile page integrates `VatNumberInput` | E2E | P1 | `billing-vat.spec.ts` |
| AC9 | i18n keys in bg.json + en.json | E2E | P2 | `billing-vat.spec.ts` |
| AC10 | Audit trail: `billing.vat_validated` on POST | Unit | P1 | `test_vat_validate_endpoint.py` |

### TDD Red Phase Requirements

All Python test functions: `@pytest.mark.skip(reason="S08.10 not yet implemented — ATDD red phase")`
All Playwright test functions: `test.skip(...)`

These tests **define expected behavior** before implementation. They will fail until:
- `vies_service.py` exists with `validate_vat_number()` and `validate_and_sync_company_vat()`
- Migration 030 and `Company.vat_validation_status` ORM field exist
- `POST /api/v1/billing/vat/validate` is registered in `billing.py`
- `automatic_tax={"enabled": True}` added to both checkout session create calls
- Company profile service hooks background validation on `tax_id` change
- `VatNumberInput` component exists and is integrated in the company profile page

---

## 🔴 TDD Red Phase: Failing Tests Generated

All Python test functions are decorated with `@pytest.mark.skip(reason="S08.10 not yet implemented — ATDD red phase")`.
All Playwright test functions use `test.skip(...)`.

### ⚠️ Critical Implementation Note

**VIES REST API response uses `"valid"` key (bool), NOT `"isValid"`.**
See story Dev Notes ⚠️ section. The test mocks use:
```python
{"countryCode": "DE", "vatNumber": "123456789", "valid": True, "name": "Test GmbH"}
```
NOT `{"isValid": true}`. Implement `data.get("valid", False)` in `vies_service.py`.

---

## Test Files Generated

### Backend: Unit Tests

#### 1. `test_vies_service.py`
**Path:** `services/client-api/tests/unit/test_vies_service.py`
**Activates on:** Story 8.10 Task 2 complete

| Class | Test | AC | Priority |
|-------|------|----|----------|
| `TestParseVatNumber` | `test_parse_valid_de_vat_number` | AC3 | P1 |
| `TestParseVatNumber` | `test_parse_valid_bg_vat_number` | AC3 | P1 |
| `TestParseVatNumber` | `test_parse_strips_spaces_and_hyphens` | AC3 | P1 |
| `TestParseVatNumber` | `test_parse_rejects_non_eu_country_code` | AC3 | P1 |
| `TestParseVatNumber` | `test_parse_rejects_too_short_input` | AC3 | P1 |
| `TestParseVatNumber` | `test_parse_northern_ireland_xi_prefix` | AC3 | P2 |
| `TestParseVatNumber` | `test_parse_case_insensitive_input` | AC3 | P2 |
| `TestValidateVatNumber` | `test_valid_vat_returns_valid_status` | AC3 | P1 |
| `TestValidateVatNumber` | `test_invalid_vat_returns_invalid_status` | AC3 | P1 |
| `TestValidateVatNumber` | `test_vies_503_returns_pending_after_all_retries` | AC3, R-002 | P1 |
| `TestValidateVatNumber` | `test_vies_timeout_returns_pending` | AC3, R-002 | P1 |
| `TestValidateVatNumber` | `test_malformed_vat_returns_invalid_without_calling_vies` | AC3 | P1 |
| `TestValidateVatNumber` | `test_retry_with_backoff_on_503_then_success` | AC3 | P1 |
| `TestValidateVatNumber` | `test_vies_connection_error_returns_pending` | AC3 | P2 |
| `TestValidateAndSyncCompanyVat` | `test_updates_db_vat_validation_status_on_valid` | AC5 | P1 |
| `TestValidateAndSyncCompanyVat` | `test_never_raises_on_exception` | AC5 | P1 |
| `TestValidateAndSyncCompanyVat` | `test_skips_stripe_sync_when_no_stripe_customer_id` | AC3, AC5 | P1 |

**Total unit tests (VIES service):** 17

---

#### 2. `test_vat_validate_endpoint.py`
**Path:** `services/client-api/tests/unit/test_vat_validate_endpoint.py`
**Activates on:** Story 8.10 Tasks 2 + 3 complete

| Class | Test | AC | Priority |
|-------|------|----|----------|
| `TestVatValidateEndpointValid` | `test_vat_validate_valid_vat_returns_200` | AC4 | P1 |
| `TestVatValidateEndpointValid` | `test_vat_validate_syncs_to_stripe_when_valid` | AC4 | P1 |
| `TestVatValidateEndpointValid` | `test_vat_validate_skips_stripe_sync_when_no_subscription` | AC4 | P1 |
| `TestVatValidateEndpointInvalid` | `test_vat_validate_invalid_vat_returns_422` | AC4 | P1 |
| `TestVatValidateEndpointPending` | `test_vat_validate_pending_returns_200_with_pending_status` | AC4, R-002 | P1 |
| `TestVatValidateEndpointPending` | `test_vat_validate_pending_does_not_call_stripe` | AC4 | P1 |
| `TestVatValidateEndpointAuth` | `test_vat_validate_requires_authentication` | AC4 | P1 |
| `TestVatValidateEndpointAuditTrail` | `test_vat_validate_writes_audit_entry_on_valid` | AC10 | P1 |
| `TestVatValidateEndpointAuditTrail` | `test_vat_validate_audit_failure_does_not_raise` | AC10 | P1 |
| `TestVatValidateEndpointDBUpdate` | `test_vat_validate_updates_company_vat_validation_status` | AC4 | P1 |

**Total unit tests (endpoint):** 10

---

#### 3. `test_checkout_stripe_tax.py`
**Path:** `services/client-api/tests/unit/test_checkout_stripe_tax.py`
**Activates on:** Story 8.10 Task 4 complete

| Class | Test | AC | Priority |
|-------|------|----|----------|
| `TestCreateCheckoutSessionIncludesAutomaticTax` | `test_create_checkout_session_includes_automatic_tax` | AC1 | P2 |
| `TestCreateCheckoutSessionIncludesAutomaticTax` | `test_create_checkout_session_automatic_tax_is_dict_enabled_true` | AC1 | P2 |
| `TestCreateAddonCheckoutSessionIncludesAutomaticTax` | `test_create_addon_checkout_session_includes_automatic_tax` | AC1 | P2 |

**Total unit tests (Stripe Tax):** 3

---

### Backend: Integration Tests

#### 4. `test_vat_validation_flow.py`
**Path:** `services/client-api/tests/integration/test_vat_validation_flow.py`
**Activates on:** Story 8.10 Tasks 1–3 complete (migration + vies_service + endpoint)

| Class | Test | Epic Test ID | AC | Priority |
|-------|------|-------------|----|----|
| `TestVatValidationFlowValid` | `test_8_10_api_001_valid_vat_syncs_to_stripe_and_db` | **8.10-API-001** | AC4, R-002 | **P1** |
| `TestVatValidationFlowInvalid` | `test_8_10_api_002_invalid_vat_returns_422` | **8.10-API-002** | AC4, R-002 | **P1** |
| `TestVatValidationFlowPending` | `test_8_10_api_003_vies_downtime_does_not_block_returns_pending` | **8.10-API-003** | AC4, R-002 | **P1** |
| `TestVatValidationCrossTenantIsolation` | `test_vat_validate_endpoint_cross_tenant_isolation` | — | AC4 | P1 |
| `TestCompanyProfileVatHook` | `test_company_profile_put_triggers_background_vat_validation` | — | AC5 | P1 |

**Total integration tests:** 5

**Coverage of epic test design P1 scenarios (8.10-API-001/002/003):** ✅ 3/3

---

### Frontend: E2E Tests

#### 5. `billing-vat.spec.ts`
**Path:** `eusolicit-app/e2e/specs/billing-vat.spec.ts`
**Activates on:** Story 8.10 Tasks 7–9 complete (VatNumberInput + profile page + i18n)

| Test | Epic Test ID | AC | Priority |
|------|-------------|----|----|
| `P1 8.10-API-001: valid EU VAT shows "Valid" badge and success toast` | 8.10-API-001 | AC7, AC8 | P1 |
| `P1 8.10-API-002: invalid EU VAT shows "Invalid" badge and error toast` | 8.10-API-002 | AC7, AC8 | P1 |
| `P1 8.10-API-003: VIES downtime shows "Pending" badge and info toast` | 8.10-API-003 | AC7, AC8 | P1 |
| `AC7: VatNumberInput shows pre-existing status from profile on page load` | — | AC7 | P2 |
| `AC9: VAT input component uses i18n — no hardcoded English in BG locale` | — | AC9 | P2 |

**Total E2E tests:** 5

---

## Test Count Summary

| Level | File | Tests | Priority Distribution |
|-------|------|-------|----------------------|
| Unit | `test_vies_service.py` | 17 | P1: 13, P2: 4 |
| Unit | `test_vat_validate_endpoint.py` | 10 | P1: 10 |
| Unit | `test_checkout_stripe_tax.py` | 3 | P2: 3 |
| Integration | `test_vat_validation_flow.py` | 5 | P1: 5 |
| E2E | `billing-vat.spec.ts` | 5 | P1: 3, P2: 2 |
| **Total** | **5 files** | **40** | **P1: 31, P2: 9** |

---

## Epic Test Coverage

| Epic Test ID | Level | Priority | Test Method | File | Status |
|-------------|-------|----------|-------------|------|--------|
| 8.10-API-001 | Integration | P1 | `test_8_10_api_001_valid_vat_syncs_to_stripe_and_db` | `test_vat_validation_flow.py` | 🔴 RED |
| 8.10-API-001 (UI) | E2E | P1 | `P1 8.10-API-001: valid EU VAT shows...` | `billing-vat.spec.ts` | 🔴 RED |
| 8.10-API-002 | Integration | P1 | `test_8_10_api_002_invalid_vat_returns_422` | `test_vat_validation_flow.py` | 🔴 RED |
| 8.10-API-002 (UI) | E2E | P1 | `P1 8.10-API-002: invalid EU VAT shows...` | `billing-vat.spec.ts` | 🔴 RED |
| 8.10-API-003 | Integration | P1 | `test_8_10_api_003_vies_downtime_does_not_block_returns_pending` | `test_vat_validation_flow.py` | 🔴 RED |
| 8.10-API-003 (UI) | E2E | P1 | `P1 8.10-API-003: VIES downtime shows...` | `billing-vat.spec.ts` | 🔴 RED |

**Epic P1 coverage for S08.10:** ✅ 3/3 test IDs covered (backend integration + E2E)

---

## Risk Mitigation Coverage

### R-002: EU VAT miscalculation or VIES validation blocking legitimate checkouts (Score: 6)

| Scenario | Test | Status |
|----------|------|--------|
| Valid VAT → Stripe B2B reverse-charge (0%) | `test_8_10_api_001_...` + `test_vat_validate_syncs_to_stripe_when_valid` | 🔴 RED |
| Invalid VAT → 422, clear error, DB flagged | `test_8_10_api_002_...` + `test_vat_validate_invalid_vat_returns_422` | 🔴 RED |
| VIES 503 → 200 "pending", never blocks | `test_8_10_api_003_...` + `test_vies_503_returns_pending_after_all_retries` | 🔴 RED |
| VIES timeout → "pending", never raises | `test_vies_timeout_returns_pending` | 🔴 RED |
| ConnectError → "pending", never raises | `test_vies_connection_error_returns_pending` | 🔴 RED |
| Malformed VAT → "invalid" without calling VIES | `test_malformed_vat_returns_invalid_without_calling_vies` | 🔴 RED |
| Retry then success (2×503 + 1×200) | `test_retry_with_backoff_on_503_then_success` | 🔴 RED |

**R-002 mitigation coverage:** ✅ Full (all three VIES scenarios + all edge cases)

---

## AC Coverage Checklist

- [x] **AC1** — `automatic_tax={"enabled": True}` on subscription checkout session (`test_checkout_stripe_tax.py`)
- [x] **AC1** — `automatic_tax={"enabled": True}` on add-on (payment) checkout session (`test_checkout_stripe_tax.py`)
- [x] **AC2** — Migration 030 `vat_validation_status` column (covered by migration framework; `test_vat_validation_flow.py` depends on it)
- [x] **AC3** — `_parse_vat_number()` format pre-validation (7 unit tests in `test_vies_service.py`)
- [x] **AC3** — `validate_vat_number()` VIES REST API with retry/backoff (7 unit tests)
- [x] **AC3** — `validate_and_sync_company_vat()` background task (3 unit tests)
- [x] **AC4** — `POST /api/v1/billing/vat/validate` happy path → 200 valid (unit + integration)
- [x] **AC4** — `POST /api/v1/billing/vat/validate` invalid → 422 (unit + integration)
- [x] **AC4** — `POST /api/v1/billing/vat/validate` pending → 200 (unit + integration)
- [x] **AC4** — Stripe sync: `Customer.modify` with `tax_ids` + `tax_exempt="reverse"` (unit)
- [x] **AC4** — Skips Stripe sync when no `stripe_customer_id` (unit)
- [x] **AC4** — Requires authentication (unit)
- [x] **AC4** — Cross-tenant isolation (integration)
- [x] **AC5** — PUT company profile with changed `tax_id` triggers `asyncio.create_task` (integration)
- [x] **AC5** — `validate_and_sync_company_vat` never raises (unit)
- [x] **AC6** — `CompanyProfileResponse.vat_validation_status` (implicitly covered by integration tests reading DB state)
- [x] **AC7** — `VatNumberInput` status badges: valid/invalid/pending (E2E)
- [x] **AC7** — Pre-existing status shown from profile response on page load (E2E)
- [x] **AC8** — Company profile page integrates `VatNumberInput` (E2E)
- [x] **AC9** — i18n: BG locale shows Bulgarian strings (E2E)
- [x] **AC10** — Audit entry `billing.vat_validated` written (unit)
- [x] **AC10** — Audit failure does not propagate (unit)

---

## Activation Checklist (Red → Green)

To remove `@pytest.mark.skip` / `test.skip()`, the following tasks must be complete:

### Backend Unit Tests

| File | Remove skip when... |
|------|---------------------|
| `test_vies_service.py` — `TestParseVatNumber`, `TestValidateVatNumber` | Task 2: `vies_service.py` implemented |
| `test_vies_service.py` — `TestValidateAndSyncCompanyVat` | Task 2: `validate_and_sync_company_vat` implemented |
| `test_vat_validate_endpoint.py` — all classes | Tasks 2 + 3: endpoint + vies_service |
| `test_checkout_stripe_tax.py` — all classes | Task 4: `automatic_tax` added to both checkout session calls |

### Backend Integration Tests

| File | Remove skip when... |
|------|---------------------|
| `test_vat_validation_flow.py` — all classes | Tasks 1 + 2 + 3: migration + vies_service + endpoint |
| `test_vat_validation_flow.py::TestCompanyProfileVatHook` | Also requires Task 5: company profile hook |

### E2E Tests

| File | Remove skip when... |
|------|---------------------|
| `billing-vat.spec.ts` — all tests | Tasks 7 + 8 + 9: VatNumberInput + profile page + i18n |

---

## Assumptions & Known Risks

1. **VIES API `"valid"` key**: Tests mock `{"valid": true/false}` not `{"isValid": true/false}`.
   Implement `data.get("valid", False)` in `vies_service.py`. See story Dev Notes ⚠️.

2. **`asyncio.to_thread` wrapping**: All Stripe SDK calls must use `asyncio.to_thread()`.
   Unit tests patch `asyncio.to_thread` to capture kwargs. If the implementation uses
   a different async wrapper, the assertions will need adjustment.

3. **`_billing_session()` context manager**: The VAT endpoint must use `_billing_session()`
   (not `Depends(get_db_session)`). This is a mandatory billing.py pattern (see Dev Notes).

4. **`require_auth` runtime pattern**: The endpoint uses `await require_auth(credentials)`
   inside the body, not as a FastAPI `Depends()`. Tests patch `billing.require_auth` directly.

5. **Migration 030 schema**: Integration tests require the `vat_validation_status` column.
   Until migration 030 is applied, integration tests will fail with a DB column error.

6. **Company profile URL**: E2E tests assume `/settings/company` (or `/[locale]/settings/company`).
   If the actual route differs, update the `COMPANY_PROFILE_URL` constant in `billing-vat.spec.ts`.

7. **`asyncio.create_task` in integration tests**: The background VAT validation hook uses
   `asyncio.create_task()`. The integration test patches `asyncio.create_task` and captures
   the coroutine without running it to avoid actual VIES calls in the test.

---

## Next Steps

1. **Implementation**: Developer implements Story 8.10 Tasks 1–10 per story specification
2. **Activate tests**: After each task group, remove `@pytest.mark.skip` from the corresponding test class
3. **Verify green phase**: Run `make test-unit` and `make test-integration` to confirm tests pass
4. **E2E activation**: After Tasks 7–9, run `make test-e2e-chromium` to activate billing-vat.spec.ts
5. **Follow-on**: `bmad-testarch-automate` — expand coverage for P2 scenarios (AC2 migration schema, AC6 response shape)

---

**Generated by:** BMad TEA Agent — Master Test Architect (bmad-testarch-atdd)
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
