---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-19'
workflowType: 'bmad-testarch-atdd'
story: '8-4-stripe-webhook-endpoint-subscription-lifecycle-sync'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/8-4-stripe-webhook-endpoint-subscription-lifecycle-sync.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-08.md'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
  - 'eusolicit-docs/_bmad/bmm/config.yaml'
tddPhase: RED
detectedStack: backend
generationMode: ai
---

# ATDD Checklist: Story 8.4 — Stripe Webhook Endpoint & Subscription Lifecycle Sync

**Date:** 2026-04-19
**Author:** BMad TEA Agent (bmad-testarch-atdd)
**TDD Phase:** 🔴 RED — All tests skipped; fail until feature implemented
**Story:** [8-4-stripe-webhook-endpoint-subscription-lifecycle-sync.md](../implementation-artifacts/8-4-stripe-webhook-endpoint-subscription-lifecycle-sync.md)
**Epic Test Design:** [test-design-epic-08.md](./test-design-epic-08.md)

---

## TDD Red Phase Status

| Category | Count | Status |
|---|---|---|
| Unit tests (webhook service) | 23 | 🔴 All `@pytest.mark.skip` |
| Unit tests (endpoint) | 4 | 🔴 All `@pytest.mark.skip` |
| Integration tests | 2 | 🔴 All `@pytest.mark.skip` |
| **Total** | **29** | **🔴 RED — Failing by design** |

---

## Generated Test Files

| File | Tests | Level | Marker |
|---|---|---|---|
| `services/client-api/tests/unit/test_webhook_service.py` | 23 | Unit | `@pytest.mark.unit` |
| `services/client-api/tests/unit/test_billing_webhook_endpoint.py` | 4 | Unit | `@pytest.mark.unit` |
| `services/client-api/tests/integration/test_stripe_webhook_flow.py` | 2 | Integration | `@pytest.mark.integration` |

---

## Acceptance Criteria Coverage

### AC1 — Signature Verification (R-004 mitigation — P0)

| Test | File | Level | Priority | Covered |
|---|---|---|---|---|
| `test_verify_stripe_signature_valid_returns_event` | test_webhook_service.py | Unit | P0 | ✅ |
| `test_verify_stripe_signature_invalid_raises_error` | test_webhook_service.py | Unit | P0 | ✅ |
| `test_verify_stripe_signature_missing_sig_raises_error` | test_webhook_service.py | Unit | P0 | ✅ |
| `test_webhook_endpoint_missing_secret_returns_400` | test_billing_webhook_endpoint.py | Unit | P0 | ✅ |
| `test_webhook_endpoint_invalid_signature_returns_400` | test_billing_webhook_endpoint.py | Unit | P0 | ✅ (8.4-API-001) |
| `test_webhook_endpoint_valid_request_returns_200` | test_billing_webhook_endpoint.py | Unit | P0 | ✅ |
| `test_webhook_endpoint_reads_raw_body` | test_billing_webhook_endpoint.py | Unit | P0 | ✅ |

**Coverage:** 7 tests | Risk R-004 mitigated

### AC2 — Subscription Lifecycle Sync (R-001 — P0)

| Test | File | Level | Priority | Covered |
|---|---|---|---|---|
| `test_get_tier_from_price_id_professional` | test_webhook_service.py | Unit | P0 | ✅ |
| `test_get_tier_from_price_id_starter` | test_webhook_service.py | Unit | P0 | ✅ |
| `test_get_tier_from_price_id_enterprise` | test_webhook_service.py | Unit | P0 | ✅ |
| `test_get_tier_from_price_id_unknown_returns_none` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_get_tier_from_price_id_none_returns_none` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_handle_subscription_created_syncs_tier_and_status` | test_webhook_service.py | Unit | P0 | ✅ (8.4-API-002) |
| `test_handle_subscription_updated_updates_tier` | test_webhook_service.py | Unit | P0 | ✅ (8.4-API-003) |
| `test_handle_subscription_unknown_price_preserves_existing_tier` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_handle_subscription_not_found_returns_none_and_logs_warning` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_webhook_subscription_created_syncs_to_db` | test_stripe_webhook_flow.py | Integration | P0 | ✅ (8.4-API-002) |

**Coverage:** 10 tests | Epic P0 test 8.4-API-002 covered

### AC3 — Subscription Deleted (P0)

| Test | File | Level | Priority | Covered |
|---|---|---|---|---|
| `test_handle_subscription_deleted_sets_free_canceled` | test_webhook_service.py | Unit | P0 | ✅ |

**Coverage:** 1 test

### AC4 — Invoice Events (P0)

| Test | File | Level | Priority | Covered |
|---|---|---|---|---|
| `test_handle_invoice_paid_sets_active` | test_webhook_service.py | Unit | P0 | ✅ |
| `test_handle_invoice_payment_failed_sets_past_due` | test_webhook_service.py | Unit | P0 | ✅ (8.4-API-004) |
| `test_handle_invoice_no_subscription_id_logs_warning` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_handle_invoice_subscription_not_found_logs_warning` | test_webhook_service.py | Unit | P1 | ✅ |

**Coverage:** 4 tests | Epic P0 test 8.4-API-004 covered

### AC5 — Idempotency (R-001 mitigation — P1)

| Test | File | Level | Priority | Covered |
|---|---|---|---|---|
| `test_record_event_if_new_returns_true_for_new_event` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_record_event_if_new_returns_false_for_duplicate` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_process_webhook_duplicate_event_returns_early` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_webhook_duplicate_event_no_duplicate_rows` | test_stripe_webhook_flow.py | Integration | P1 | ✅ (8.4-API-005) |

**Coverage:** 4 tests | Epic P1 test 8.4-API-005 covered | Risk R-001 mitigated

### AC6 — subscription.changed Event (P1)

| Test | File | Level | Priority | Covered |
|---|---|---|---|---|
| `test_process_webhook_subscription_changed_not_published_on_invoice_event` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_webhook_duplicate_event_no_duplicate_rows` (xadd call count assert) | test_stripe_webhook_flow.py | Integration | P1 | ✅ |

**Coverage:** 2 tests (publish-after-commit behavior tested indirectly via Redis mock)

### AC7 — Audit Logging (P1)

| Test | File | Level | Priority | Covered |
|---|---|---|---|---|
| `test_process_webhook_unhandled_event_type_returns_200` | test_webhook_service.py | Unit | P1 | ✅ |
| `test_process_webhook_audit_entry_written_on_success` | test_webhook_service.py | Unit | P1 | ✅ |

**Coverage:** 2 tests

### AC8 — Tests (meta-AC)

**This checklist and the 29 generated tests ARE the fulfillment of AC8.**

---

## Epic Test Design Coverage Map

| Test ID | Description | Priority | Status |
|---|---|---|---|
| 8.4-API-001 | Webhook signature verification rejects tampered/missing signatures → HTTP 400 | **P0** | ✅ Covered by `test_webhook_endpoint_invalid_signature_returns_400` |
| 8.4-API-002 | `customer.subscription.created` syncs tier and dates to local DB | **P0** | ✅ Covered by `test_handle_subscription_created_syncs_tier_and_status` + integration |
| 8.4-API-003 | `customer.subscription.updated` reflects plan change in local DB | **P0** | ✅ Covered by `test_handle_subscription_updated_updates_tier` |
| 8.4-API-004 | `invoice.payment_failed` marks subscription `past_due` | **P0** | ✅ Covered by `test_handle_invoice_payment_failed_sets_past_due` |
| 8.4-API-005 | Idempotency: replaying same `stripe_event_id` → exactly one record | **P1** | ✅ Covered by `test_webhook_duplicate_event_no_duplicate_rows` |

**All 5 epic-level test scenarios covered.**

---

## Risk Mitigation Verification

| Risk | Score | Mitigation | Tests |
|---|---|---|---|
| R-004 — Webhook signature bypass | 6 (HIGH) | `stripe.Webhook.construct_event` enforced on every request; missing/invalid → 400 | `test_webhook_endpoint_invalid_signature_returns_400`, `test_webhook_endpoint_missing_secret_returns_400`, `test_verify_stripe_signature_*` (3 tests) |
| R-001 — Webhook race conditions / duplicate subscriptions | 6 (HIGH) | INSERT ON CONFLICT DO NOTHING on `webhook_events.stripe_event_id` UNIQUE constraint | `test_record_event_if_new_*`, `test_process_webhook_duplicate_event_returns_early`, `test_webhook_duplicate_event_no_duplicate_rows` |

---

## Test Strategy

### Stack & Mode
- **Detected Stack:** `backend` (FastAPI/Python 3.12+, pytest + pytest-asyncio)
- **Generation Mode:** AI (no browser recording needed — pure API/backend story)
- **Execution Mode:** Sequential

### Test Level Selection

| Level | Rationale | Count |
|---|---|---|
| **Unit** | Isolate pure functions (`_get_tier_from_price_id`, `verify_stripe_signature`, `_record_event_if_new`, handlers) and endpoint security with all deps mocked | 27 |
| **Integration** | Full DB round-trip: verify ORM writes, UNIQUE constraint enforcement, and endpoint-to-DB-to-Redis chain | 2 |
| **No E2E** | Backend-only story; no browser UI involved | 0 |

### Why No E2E Tests
Story 8.4 is a pure backend feature. The Stripe webhook endpoint has no browser UI. E2E coverage for the billing user journey is handled by Story 8.6 (Checkout upgrade) and Story 8.5 (trial expiry), which test the downstream effects of these webhook events.

### Red Phase Design
All tests are decorated with `@pytest.mark.skip` and will raise `ImportError` (target modules don't exist) when unskipped before implementation. This is the canonical RED phase signal for pytest-based TDD in this project.

---

## Next Steps — TDD Green Phase

After implementing Story 8.4 Tasks 1–6:

### 1. Apply Migration 028
```bash
cd eusolicit-app
make migrate-service SVC=client-api
```

### 2. Verify New Files Exist
```
services/client-api/alembic/versions/028_webhook_events_table.py
services/client-api/src/client_api/models/webhook_event.py
services/client-api/src/client_api/services/webhook_service.py
services/client-api/src/client_api/api/v1/billing.py
```

### 3. Remove Skip Decorators
```bash
# Unit tests — remove @pytest.mark.skip from all 27 tests in:
#   tests/unit/test_webhook_service.py
#   tests/unit/test_billing_webhook_endpoint.py

# Integration tests — remove @pytest.mark.skip from 2 tests in:
#   tests/integration/test_stripe_webhook_flow.py
```

### 4. Run Tests
```bash
cd eusolicit-app

# Unit tests only (fast, no DB required)
make test-unit

# Integration tests (DB + Redis required — run infra first)
make infra
make test-integration

# Single service (all levels)
make test-service SVC=client-api
```

### 5. Verify Green Phase
All 29 tests should pass. If any fail:
- **ImportError** → module/function not yet implemented (check task checklist)
- **404 on endpoint** → billing router not registered in main.py (Task 6)
- **DB error on webhook_events** → migration 028 not applied (Task 1)
- **AssertionError on tier** → `_get_tier_from_price_id` mapping wrong (Task 4.2)
- **Duplicate row in integration test** → INSERT ON CONFLICT not using correct index (Task 4.4)

### 6. Commit Passing Tests
```bash
git add services/client-api/tests/unit/test_webhook_service.py
git add services/client-api/tests/unit/test_billing_webhook_endpoint.py
git add services/client-api/tests/integration/test_stripe_webhook_flow.py
git commit -m "test: Story 8.4 ATDD green phase — stripe webhook lifecycle tests passing"
```

---

## Implementation Dependencies

These test files assume the following implementations are complete (from story task list):

| Task | Description | Tests Depend On |
|---|---|---|
| Task 1 | Migration 028: `client.webhook_events` table | Integration tests (INSERT fails without table) |
| Task 2 | `WebhookEvent` ORM model | `_record_event_if_new` unit tests |
| Task 3 | Config: `stripe_webhook_secret`, `stripe_starter_price_id`, `stripe_enterprise_price_id` | All tests (settings mocked, but config must exist for import) |
| Task 4 | `webhook_service.py` (all helpers + `process_stripe_webhook`) | All unit tests in test_webhook_service.py |
| Task 5 | `api/v1/billing.py` (endpoint router) | All tests in test_billing_webhook_endpoint.py + integration |
| Task 6 | Register billing router in `main.py` | All endpoint tests (404 until registered) |

---

## Assumptions & Notes

1. **`@pytest.mark.skip` as RED phase marker** — follows the project convention established in `test_billing_service.py`, `test_trial_provisioning.py`, and `test_stripe_customer_provisioning.py`.

2. **Import inside test body** — functions are imported inside each test body (e.g., `from client_api.services.webhook_service import verify_stripe_signature`) so that `ImportError` is raised only when the skip is removed, not when the test file is collected.

3. **Integration test DB seeding** — uses raw SQL via `text()` to avoid FK ordering issues. Assumes `client.companies` and `client.subscriptions` tables exist (from migrations 026–027, Story 8.2).

4. **`db_session` fixture** — expects the `db_session` fixture from `tests/conftest.py` (session-scoped AsyncSession). If the fixture name differs, update imports.

5. **Redis mock** — integration tests mock `get_redis_client()` return value and assert `xadd` call count. The Redis Streams `subscription.changed` event is non-fatal; a Redis connection failure in tests would not affect assertions (mocked).

6. **Concurrent idempotency test (R-001)** — the epic test design calls for a 10-thread concurrent replay test (`test_webhook_duplicate_event_no_duplicate_rows` at R-001 mitigation plan). The integration test here covers the sequential duplicate case. The concurrent variant should be added to `bmad-testarch-automate` when infrastructure for concurrent integration tests is available.

7. **`session_factory` fixture** — referenced in integration test signatures for completeness. Tests use `db_session` directly for verification queries. If the fixture is not available from conftest, the test can use `db_session` only.

---

**Generated by:** BMad TEA Agent — ATDD Module
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
