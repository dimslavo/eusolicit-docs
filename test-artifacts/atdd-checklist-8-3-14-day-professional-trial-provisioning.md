---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-18'
workflowType: 'bmad-testarch-atdd'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/8-3-14-day-professional-trial-provisioning.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-08.md'
  - 'eusolicit-app/services/client-api/src/client_api/services/billing_service.py'
  - 'eusolicit-app/services/client-api/src/client_api/core/tier_gate.py'
  - 'eusolicit-app/services/client-api/tests/unit/test_billing_service.py'
  - 'eusolicit-app/services/client-api/tests/unit/test_tier_gate.py'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
  - '_bmad/bmm/config.yaml'
---

# ATDD Checklist: Story 8.3 — 14-Day Professional Trial Provisioning

**Story Key:** `8-3-14-day-professional-trial-provisioning`
**Date Generated:** 2026-04-18
**TDD Phase:** 🔴 RED (failing tests — implementation not yet done)
**Stack:** fullstack (tests cover backend only — no UI for this story)
**Test Framework:** pytest + pytest-asyncio + unittest.mock

---

## TDD Red Phase Summary

| Metric | Value |
|---|---|
| Total Tests Generated | 28 |
| Unit Tests | 24 |
| Integration Tests | 4 |
| All Tests Expected to FAIL | ✅ Yes |
| Fixture Needs | AsyncMock sessions, Stripe mocks, Redis mocks |

### Why Tests Fail (Red Phase)

| Test File | Fail Reason |
|---|---|
| `test_trial_provisioning.py` | `ImportError: cannot import name 'provision_professional_trial' from 'client_api.services.billing_service'` (Task 2 not implemented) |
| `test_tier_gate_trial.py` | `AttributeError: module 'client_api.core.tier_gate' has no attribute '_ALLOWED_STATUSES'` + SQL assertion: "trialing" not in compiled `status = 'active'` (Task 5 not implemented) |
| `test_new_company_billing_bg.py` | `ImportError: cannot import name 'provision_new_company_billing_bg' from 'client_api.services.billing_service'` (Task 3 not implemented) |
| `test_trial_provisioning_flow.py` | Same `ImportError` on `provision_new_company_billing_bg` (Tasks 3+4 not implemented) |

---

## Generated Test Files

| File | Tests | Priority Coverage | AC Coverage |
|---|---|---|---|
| `tests/unit/test_trial_provisioning.py` | 11 | P0: 8, P1: 2, P2: 1 | AC1, AC2, AC3, AC5 |
| `tests/unit/test_tier_gate_trial.py` | 10 | P0: 6, P1: 4 | AC7 |
| `tests/unit/test_new_company_billing_bg.py` | 9 | P0: 5, P1: 3, P2: 1 | AC4, AC6 |
| `tests/integration/test_trial_provisioning_flow.py` | 4 | P0: 2, P2: 2 | AC2, AC3, AC4, AC6, AC8 |

---

## Acceptance Criteria Coverage

| AC | Description | Test(s) | Priority | Status |
|---|---|---|---|---|
| **AC1** | Stripe subscription created with correct params: customer, items, trial_end (now+14d Unix), payment_behavior="default_incomplete", idempotency_key=f"trial-provisioning-{company_id}" | `test_trial_provisioned_creates_stripe_subscription_and_updates_db`, `test_stripe_subscription_create_uses_correct_customer_and_price`, `test_stripe_subscription_create_uses_payment_behavior_default_incomplete`, `test_trial_provisioning_uses_correct_idempotency_key`, `test_trial_provisioning_trial_end_is_14_days_from_now` | P0 | 🔴 RED |
| **AC2** | DB subscription row updated atomically: tier="professional", status="trialing", trial_start/end, is_trial=True, started_at; stripe_customer_id preserved | `test_trial_provisioned_creates_stripe_subscription_and_updates_db`, `test_stripe_customer_id_preserved_after_trial_provisioning`, `test_provision_new_company_billing_bg_creates_trial_subscription`, `test_provision_new_company_billing_bg_trial_end_is_14_days` | P0 | 🔴 RED |
| **AC3** | One-trial-per-company: existing stripe_subscription_id → skip with INFO "trial_already_provisioned"; no duplicate Stripe call | `test_trial_provisioning_idempotent_existing_subscription`, `test_trial_already_provisioned_logs_info`, `test_second_provision_call_does_not_call_stripe_subscription_create_again` | P0/P2 | 🔴 RED |
| **AC4** | subscription.changed event published to Redis Streams AFTER session.commit(); payload: {company_id, old_tier="free", new_tier="professional", timestamp}; event failure non-fatal | `test_provision_new_company_billing_bg_publishes_event_after_commit`, `test_provision_new_company_billing_bg_no_event_when_trial_skipped`, `test_provision_new_company_billing_bg_event_failure_is_non_fatal`, `test_provision_new_company_billing_bg_publishes_redis_stream_event` | P0 | 🔴 RED |
| **AC5** | Graceful degradation: no stripe_customer_id → WARNING skip; no price_id → WARNING skip; StripeError → ERROR swallowed | `test_trial_provisioning_skips_when_no_customer_id`, `test_trial_provisioning_skips_when_no_price_id`, `test_trial_provisioning_stripe_error_logged_and_swallowed`, `test_trial_provisioning_skips_when_no_subscription_row` | P0 | 🔴 RED |
| **AC6** | auth.py calls provision_new_company_billing_bg() (combined task); combined function runs customer+trial in single session; provision_stripe_customer_bg() retained deprecated | `test_provision_new_company_billing_bg_calls_customer_then_trial`, `test_provision_new_company_billing_bg_passes_customer_id_to_trial`, `test_auth_module_imports_provision_new_company_billing_bg`, `test_provision_stripe_customer_bg_still_exists_deprecated`, `test_register_endpoint_background_task_is_provision_new_company_billing_bg` | P0 | 🔴 RED |
| **AC7** | tier_gate.py: all three gate functions use status.in_(_ALLOWED_STATUSES); _ALLOWED_STATUSES=["active","trialing"]; opportunity_tier_gate.py also updated | `test_allowed_statuses_constant_exists`, `test_allowed_statuses_constant_contains_active_and_trialing`, `test_allowed_statuses_does_not_include_cancelled_or_past_due`, `test_require_paid_tier_sql_includes_trialing_status`, `test_require_professional_plus_tier_sql_includes_trialing_status`, `test_require_enterprise_tier_sql_includes_trialing_status`, `test_get_opportunity_tier_gate_sql_includes_trialing_status` | P0 | 🔴 RED |
| **AC8** | Unit + Integration tests in specified files cover all above ACs | *(all tests above)* | P0 | 🔴 RED |

---

## Implementation Tasks Required to Turn GREEN

When the developer completes the following tasks, all tests should pass.
Remove ATDD RED PHASE comments from test docstrings at that time.

| Task | File(s) to Change | Tests Unlocked |
|---|---|---|
| **Task 1**: Add `stripe_professional_price_id` to `ClientApiSettings` | `services/client-api/src/client_api/config.py` | AC5 tests (price_id guard) |
| **Task 2**: Implement `provision_professional_trial()` | `services/client-api/src/client_api/services/billing_service.py` | All `test_trial_provisioning.py` tests |
| **Task 3**: Implement `provision_new_company_billing_bg()` | `services/client-api/src/client_api/services/billing_service.py` | All `test_new_company_billing_bg.py` tests |
| **Task 4**: Update `auth.py` to use `provision_new_company_billing_bg` | `services/client-api/src/client_api/api/v1/auth.py` | `test_auth_module_imports_provision_new_company_billing_bg`, `test_register_endpoint_*` |
| **Task 5**: Update `tier_gate.py` with `_ALLOWED_STATUSES` + `.in_()` filter | `services/client-api/src/client_api/core/tier_gate.py`, `core/opportunity_tier_gate.py` | All `test_tier_gate_trial.py` tests |
| **Task 6+7**: *(these ARE the ATDD tests — already created above)* | *(this file)* | *(n/a)* |

---

## Test Execution Commands

```bash
# Run all Story 8.3 unit tests (RED phase — all will fail)
cd eusolicit-app
make test-unit  # or:
pytest services/client-api/tests/unit/test_trial_provisioning.py -v
pytest services/client-api/tests/unit/test_tier_gate_trial.py -v
pytest services/client-api/tests/unit/test_new_company_billing_bg.py -v

# Run integration tests (require DB + Redis)
pytest services/client-api/tests/integration/test_trial_provisioning_flow.py -v -m integration

# Run all 8.3 tests together
pytest services/client-api/tests/unit/test_trial_provisioning.py \
       services/client-api/tests/unit/test_tier_gate_trial.py \
       services/client-api/tests/unit/test_new_company_billing_bg.py \
       services/client-api/tests/integration/test_trial_provisioning_flow.py \
       -v
```

---

## TDD Green Phase Checklist

After implementing Tasks 1–5, run the tests and verify they pass:

- [ ] `test_trial_provisioning.py` — all 11 tests GREEN
- [ ] `test_tier_gate_trial.py` — all 10 tests GREEN
- [ ] `test_new_company_billing_bg.py` — all 9 tests GREEN
- [ ] `test_trial_provisioning_flow.py` — all 4 tests GREEN (requires running DB)
- [ ] `make coverage` — coverage remains ≥80%
- [ ] `make lint` — ruff passes (no new violations)
- [ ] `make type-check` — mypy passes

---

## Risk Mitigation Coverage

| Risk ID | Description | Mitigated By |
|---|---|---|
| **R-005** | Trial manipulation — company registers multiple times for repeated trials | `test_trial_provisioning_idempotent_existing_subscription` (AC3), `test_second_provision_call_does_not_call_stripe_subscription_create_again` (integration) |
| **NIT-4** | `status == "active"` in tier_gate.py blocks trial users from Professional features | `test_require_paid_tier_sql_includes_trialing_status`, `test_require_professional_plus_tier_sql_includes_trialing_status`, `test_require_enterprise_tier_sql_includes_trialing_status` |

---

## Epic Test Design Coverage

From `test-design-epic-08.md`:

| Epic Test ID | Priority | Covered By |
|---|---|---|
| **8.3-E2E-001** | P0 | `test_provision_new_company_billing_bg_creates_trial_subscription` + `test_provision_new_company_billing_bg_publishes_redis_stream_event` (integration) |
| **8.3-API-001** | P2 | `test_second_provision_call_does_not_call_stripe_subscription_create_again` (integration) |

---

## Known Assumptions and Deferred Items

| Item | Notes |
|---|---|
| **NIT-7** (enterprise_api_keys.py queries plan) | Out of scope for 8.3 — deferred to cleanup story per Dev Notes |
| **NIT-5** (admin-api platform_analytics_service) | Out of scope for 8.3 — deferred to cleanup story per Dev Notes |
| **Opportunity_tier_gate test** | `test_get_opportunity_tier_gate_sql_includes_trialing_status` has `pytest.skip` fallback if dependency signature mismatch — manual verification may be needed |
| **E2E UI test** | No browser-level E2E test for trial banner or upgrade CTA (story is backend service logic only) |

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Story:** 8-3-14-day-professional-trial-provisioning
