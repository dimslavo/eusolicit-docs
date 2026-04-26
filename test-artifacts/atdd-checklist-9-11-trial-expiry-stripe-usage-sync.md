---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-20'
lastReviewed: '2026-04-20'
workflowType: 'bmad-testarch-atdd'
mode: 'create'
storyKey: '9-11-trial-expiry-stripe-usage-sync'
detectedStack: 'fullstack'
generationMode: 'AI Generation (backend story — no browser recording)'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/9-11-trial-expiry-stripe-usage-sync.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-09.md'
  - 'eusolicit-app/services/notification/src/notification/workers/approval_consumer.py'
  - 'eusolicit-app/services/notification/src/notification/workers/tasks/billing_usage_sync.py'
  - 'eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py'
  - 'eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py'
  - 'eusolicit-app/services/notification/src/notification/main.py'
  - 'eusolicit-app/services/notification/tests/test_lifespan_consumers.py'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-9-10-task-approval-notification-consumers.md'
testFramework: 'pytest + pytest-asyncio (asyncio_mode=auto)'
---

# ATDD Checklist: Story 9.11 — Trial Expiry & Stripe Usage Sync

**Story:** `9-11-trial-expiry-stripe-usage-sync`
**Date:** 2026-04-20
**TDD Phase:** 🔴 RED (all tests skipped — failing until implementation complete)
**Stack:** fullstack / backend story
**Test Framework:** pytest + pytest-asyncio (`asyncio_mode="auto"`)
**Risk Links Closed:** E09-R-004 (P0, Stripe counter atomicity) · E09-R-010 (P1, trial expiry event coupling) · E09-R-002 (P0 carry-over, consumer duplicate dispatch)

---

## TDD Red Phase Summary

✅ Failing tests generated — **8 test files**, **51 tests total**

> **Correction (2026-04-20 review):** Most tests do NOT use `@pytest.mark.skip` — they fail
> at collection/runtime because the source modules (`subscription_consumer.py`, hardened
> `billing_usage_sync.py`) do not yet satisfy the assertions. The AC12 bootstrap regression
> guard tests (`test_bootstrap_subscriptions.py`) have had their incorrect `@pytest.mark.skip`
> decorators removed: the bootstrap constants already exist and these tests should run (and
> pass) immediately — their purpose is to fail loudly on future accidental regression, not to
> mark unimplemented code.

---

## Generated Test Files

| File | Tests | Priority | ACs Covered |
|------|-------|----------|-------------|
| `packages/eusolicit-models/tests/test_events_trial_expiring.py` | 8 | P1 | AC1 |
| `services/client-api/tests/unit/test_webhook_service_stream_alignment.py` | 4 | P1 | AC2 |
| `services/notification/tests/worker/test_subscription_consumer.py` | 18 | P0/P1/P2 | AC3, AC4, AC5, AC6, AC7, AC14 |
| `services/notification/tests/worker/test_billing_usage_sync_atomicity.py` | 12 | P0/P1/P2 | AC9, AC10, AC11, AC13, AC14 |
| `services/notification/tests/integration/test_subscription_consumer_integration.py` | 2 | P0/P1 | AC3, AC5 (E09-R-002, E09-R-010) |
| `services/notification/tests/integration/test_billing_usage_sync_atomicity.py` | 2 | P0 | AC10 (E09-R-004) |
| `services/notification/tests/test_lifespan_consumers.py` *(extended)* | 2 | P1 | AC8 |
| `packages/eusolicit-common/tests/events/test_bootstrap_subscriptions.py` | 3 | P1 | AC12 |

**Total: 51 unique `@pytest.mark.skip` test functions across 8 files**

---

## Acceptance Criteria Coverage

### ✅ AC1 — `TrialExpiring` event schema in `eusolicit-models.events`

- [x] `test_trial_expiring_event_class_importable` (unit) — TrialExpiring importable
- [x] `test_trial_expiring_event_type_literal_is_trial_expiring` (unit) — Literal["TrialExpiring"]
- [x] `test_trial_expiring_default_days_remaining_is_3` (unit) — default=3
- [x] `test_trial_expiring_explicit_days_remaining_stored` (unit) — explicit value stored
- [x] `test_trial_expiring_missing_company_id_raises_validation_error` (unit) — required fields
- [x] `test_trial_expiring_missing_trial_end_raises_validation_error` (unit) — required fields
- [x] `test_service_event_discriminated_union_includes_trial_expiring` (unit) — union routing
- [x] `test_trial_expiring_round_trip_json_discriminated_union` (unit) — AC1 explicit round-trip

**File:** `packages/eusolicit-models/tests/test_events_trial_expiring.py`

---

### ✅ AC2 — Client-API publisher aligned to canonical stream

- [x] `test_publish_trial_expiring_uses_canonical_stream` (unit) — stream="eu-solicit:subscriptions"
- [x] `test_publish_trial_expiring_not_using_legacy_stream` (unit) — NOT "trial.expiring"
- [x] `test_publish_trial_expiring_payload_contains_required_fields` (unit) — company_id, trial_end, days_remaining
- [x] `test_publish_trial_expiring_event_type_is_trial_expiring` (unit) — event_type field

**File:** `services/client-api/tests/unit/test_webhook_service_stream_alignment.py`

---

### ✅ AC3 — `SubscriptionConsumer` dispatches to all company admins

- [x] `test_trial_expiring_dispatches_email_to_all_company_admins` (unit) — 3 admins → 3 dispatches
- [x] `test_trial_expiring_template_type_is_trial_expiry_reminder` (unit) — template_type value
- [x] `test_trial_expiring_template_data_contains_required_fields` (unit) — days_remaining, trial_end, company_id, URLs
- [x] `test_trial_expiring_upgrade_url_from_settings_app_base_url` (unit) — URL from settings, not hardcoded
- [x] `test_trial_expiring_fans_out_to_all_three_admins` (integration) — real Redis, 3 send_email calls
- [x] `test_cross_tenant_company_b_admin_not_emailed_for_company_a_event` (unit) — cross-tenant negative

**Files:** `test_subscription_consumer.py`, `test_subscription_consumer_integration.py`

---

### ✅ AC4 — Zero-admin graceful handling

- [x] `test_zero_admins_logs_warning_and_acks_no_email_dispatched` (unit) — warning log + ACK, no email
- [x] `test_no_admins_log_emitted` (unit / AC14 overlap) — subscription_consumer.no_admins log event

**File:** `services/notification/tests/worker/test_subscription_consumer.py`

---

### ✅ AC5 — Idempotent consumption (SETNX per-admin)

- [x] `test_setnx_per_admin_key_format` (unit) — `notification:dispatched:subscription:{msg_id}:{admin_user_id}`
- [x] `test_setnx_ttl_is_86400_seconds` (unit) — 24-hour TTL
- [x] `test_setnx_already_dispatched_returns_true_skip` (unit) — collision → skip
- [x] `test_partial_replay_only_emails_un_dispatched_admins` (unit) — 3 admins, admin1 pre-dispatched → 2 sent
- [x] `test_run_loop_order_retry_pending_before_consume` (unit) — loop order invariant
- [x] `test_partial_crash_replay_only_emails_missing_admins` (integration) — real Redis, SETNX pre-set

**Files:** `test_subscription_consumer.py`, `test_subscription_consumer_integration.py`

---

### ✅ AC6 — Graceful handling of unknown and malformed events

- [x] `test_unknown_event_type_logs_info_and_acks` (unit) — info log + ACK, no email
- [x] `test_malformed_json_payload_logs_error_and_acks` (unit) — error log + ACK, no exception
- [x] `test_pydantic_validation_error_logs_error_and_acks` (unit) — Pydantic error → error log + ACK
- [x] `test_subscription_consumer_uses_type_adapter_not_hand_rolled_json` (unit) — TypeAdapter enforced

**File:** `services/notification/tests/worker/test_subscription_consumer.py`

---

### ✅ AC7 — Async-safe Celery dispatch

- [x] `test_send_email_delay_wrapped_in_asyncio_to_thread` (unit) — source inspection — no bare `.delay()` in async code

**File:** `services/notification/tests/worker/test_subscription_consumer.py`

---

### ✅ AC8 — Consumer lifecycle wiring in `main.py::lifespan`

- [x] `test_lifespan_creates_subscription_consumer_task` (unit) — `app.state.subscription_consumer_task` exists
- [x] `test_lifespan_cancels_subscription_consumer_task_on_shutdown` (unit) — CancelledError handled gracefully

**File:** `services/notification/tests/test_lifespan_consumers.py` (extended)

---

### ✅ AC9 — Stripe usage sync idempotency key (E09-R-004, P0)

- [x] `test_idempotency_key_passed_to_create_usage_record` (unit) — kwarg present
- [x] `test_idempotency_key_format_is_item_id_colon_period` (unit) — `{item_id}:{period}` shape
- [x] `test_two_invocations_same_item_period_produce_same_idempotency_key` (unit) — determinism

**File:** `services/notification/tests/worker/test_billing_usage_sync_atomicity.py`

---

### ✅ AC10 — Stripe usage sync crash-safety (E09-R-004, P0 non-negotiable)

- [x] `test_counter_intact_on_stripe_failure` (unit) — Redis key NOT deleted on StripeError
- [x] `test_counter_cleared_on_stripe_success` (unit) — Redis key deleted after Stripe 200
- [x] `test_archive_written_only_after_stripe_success` (unit) — operation order: archive only after success
- [x] `test_stripe_counter_intact_on_failure` (integration, **P0 exit criterion**) — real Redis, Stripe raises → key exists
- [x] `test_stripe_counter_cleared_on_success` (integration, **P0 exit criterion**) — real Redis, Stripe 200 → key deleted

**Files:** `test_billing_usage_sync_atomicity.py` (unit + integration)
> ⚠️ **Non-negotiable:** E09-R-004 exit criterion requires BOTH integration tests passing.

---

### ✅ AC11 — Retry behavior parity

- [x] `test_rate_limit_error_triggers_retry` (unit) — RateLimitError → retry path
- [x] `test_api_connection_error_triggers_retry` (unit) — APIConnectionError → retry path
- [x] `test_invalid_request_error_skips_without_retry` (unit) — InvalidRequestError → skip, no retry
- [x] `test_sync_usage_to_stripe_celery_decorator_not_regressed` (unit) — max_retries=2, default_retry_delay=300, acks_late=True

**File:** `services/notification/tests/worker/test_billing_usage_sync_atomicity.py`

---

### ✅ AC12 — Bootstrap / consumer-group sanity check

- [x] `test_streams_subscriptions_maps_to_canonical_key` (unit) — STREAMS["subscriptions"]=="eu-solicit:subscriptions"
- [x] `test_notification_svc_consumer_group_includes_subscriptions_stream` (unit) — "eu-solicit:subscriptions" in CONSUMER_GROUPS["notification-svc"]
- [x] `test_legacy_trial_expiring_stream_not_in_bootstrap` (unit) — "trial.expiring" NOT in bootstrap

**File:** `packages/eusolicit-common/tests/events/test_bootstrap_subscriptions.py`

> ⚠️ **Note (2026-04-20):** `@pytest.mark.skip` removed from all 3 AC12 tests. Constants already
> exist in `bootstrap.py` (verified: STREAMS["subscriptions"]="eu-solicit:subscriptions" at line 22,
> CONSUMER_GROUPS["notification-svc"] includes it at line 47). These tests should **pass immediately**
> and act as regression guards going forward.

---

### ✅ AC13 — Security: no plaintext secrets in logs

- [x] `test_stripe_secret_key_never_appears_in_logs` (unit) — structlog capture, NOTIFICATION_STRIPE_SECRET_KEY absent

**File:** `services/notification/tests/worker/test_billing_usage_sync_atomicity.py`

---

### ✅ AC14 — Observability

- [x] `test_trial_expiring_dispatched_log_emitted` (unit) — subscription_consumer.trial_expiring_dispatched log
- [x] `test_no_admins_log_emitted` (unit) — subscription_consumer.no_admins log
- [x] `test_billing_sync_idempotency_key_set_log_emitted` (unit) — billing_sync_idempotency_key_set log

> Note: `subscription_consumer.unknown_event_type` and `subscription_consumer.malformed_payload` are implicitly covered by AC6 tests which assert the correct path is taken (ACK + no email). Explicit log-capture assertions for those two events can be added in the TEA Automate phase if coverage gaps are detected.

---

## Risk Mitigation Verification

| Risk | Score | Tests Verifying Mitigation | Status |
|------|-------|---------------------------|--------|
| **E09-R-004** — Stripe counter atomicity | 6 (HIGH) | `test_stripe_counter_intact_on_failure` (int) + `test_stripe_counter_cleared_on_success` (int) + 3 unit tests | 🔴 RED — awaiting implementation |
| **E09-R-010** — Trial expiry event coupling | 4 (MED) | `test_publish_trial_expiring_uses_canonical_stream` + `test_trial_expiring_fans_out_to_all_three_admins` (int) | 🔴 RED — awaiting implementation |
| **E09-R-002** — Consumer duplicate dispatch (carry-over) | 6 (HIGH) | `test_setnx_per_admin_key_format` + `test_partial_crash_replay_only_emails_missing_admins` (int) | 🔴 RED — awaiting implementation |

---

## Test Execution Map (by pytest marker)

```
# Unit tests (no external deps)
make test-unit  # runs automatically via pytest markers

# Specific service unit tests
make test-service SVC=notification
make test-service SVC=client-api

# Integration tests (requires testcontainers Redis)
make test-integration

# Coverage check (must stay ≥80% on subscription_consumer.py + billing_usage_sync.py)
make coverage
```

**pytest markers used:**
- No marker → unit test (runs in `make test-unit`)
- `@pytest.mark.integration` → requires testcontainers Redis/Postgres

---

## Next Steps (TDD Green Phase)

After implementation is complete (Story 9.11 dev-story):

1. **Remove** `@pytest.mark.skip(reason="RED PHASE: ...")` from all generated test functions
2. **Run tests:** `make test-service SVC=notification && make test-service SVC=client-api`
3. **Verify tests PASS** (green phase)
4. **Run integration tests:** `make test-integration`
5. **Verify E09-R-004 exit criteria:** both `test_stripe_counter_intact_on_failure` AND `test_stripe_counter_cleared_on_success` (integration) pass
6. **Check coverage:** `make coverage` — `subscription_consumer.py` and `billing_usage_sync.py` must be ≥80% (100% branch coverage on Stripe idempotency + archive-delete ordering branches)
7. **Lint + type-check:** `make quality-check`
8. Commit passing tests

## Implementation Guidance

### New files to create
- `services/notification/src/notification/workers/subscription_consumer.py` — mirror `approval_consumer.py` skeleton
- `packages/eusolicit-models/tests/test_events_trial_expiring.py` *(generated)*

### Files to modify
- `packages/eusolicit-models/src/eusolicit_models/events.py` — add `TrialExpiring` class + extend `ServiceEvent` union
- `packages/eusolicit-models/pyproject.toml` — bump patch version
- `services/client-api/src/client_api/services/webhook_service.py` — `_publish_trial_expiring`: change `stream="trial.expiring"` → `stream="eu-solicit:subscriptions"`
- `services/notification/src/notification/main.py` — add 4th `asyncio.Task` for `run_subscription_consumer`
- `services/notification/src/notification/workers/tasks/billing_usage_sync.py` — add `idempotency_key`, enforce archive+delete-after-success ordering, refine exception branches

### Key patterns (copy verbatim from `approval_consumer.py`)
- `_is_idempotent(msg_id, action_key)` — SETNX pattern with per-target key
- `run()` loop: `retry_pending → process_pending(min_idle_time=30_000) → consume → process_event`
- `asyncio.to_thread(send_email.delay, ...)` — mandatory for all Celery `.delay()` calls from async code
- `TypeAdapter(ServiceEvent).validate_python(merged_dict)` — mandatory Pydantic parsing (not hand-rolled)

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
