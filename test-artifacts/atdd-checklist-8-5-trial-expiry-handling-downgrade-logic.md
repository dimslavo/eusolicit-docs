---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-19'
workflowType: 'testarch-atdd'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/8-5-trial-expiry-handling-downgrade-logic.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-08.md'
  - 'eusolicit-docs/_bmad/bmm/config.yaml'
  - 'eusolicit-app/services/client-api/tests/unit/test_webhook_service.py'
  - 'eusolicit-app/services/client-api/tests/integration/test_stripe_webhook_flow.py'
  - 'eusolicit-app/services/client-api/src/client_api/services/webhook_service.py'
---

# ATDD Checklist: Story 8.5 — Trial Expiry Handling & Downgrade Logic

**Date:** 2026-04-19
**Author:** BMad TEA Agent (bmad-testarch-atdd)
**Status:** 🔴 RED PHASE — All tests generated as failing (TDD red phase)
**Story:** `8-5-trial-expiry-handling-downgrade-logic`
**Epic:** Epic 08 — Subscription & Billing

---

## Step 1: Preflight & Context

### Stack Detection

| Item | Value |
|---|---|
| Detected stack | `fullstack` (Python FastAPI + Next.js) |
| Effective test stack for this story | `backend` (no UI changes in Story 8.5) |
| Test framework | pytest + pytest-asyncio + pytest-mock |
| Test marker for skip | `@pytest.mark.skip(reason="🔴 RED PHASE...")` |
| Browser recording | N/A (backend-only story) |

### Prerequisites Verified

- [x] Story status: `ready-for-dev` with clear acceptance criteria (7 ACs)
- [x] Test framework: `conftest.py` confirmed in `services/client-api/tests/`
- [x] Existing patterns loaded: `test_webhook_service.py` (Story 8.4 unit) + `test_stripe_webhook_flow.py` (Story 8.4 integration)
- [x] Epic test design loaded: `test-design-epic-08.md` — P0 test `8.5-E2E-001`, P2 test `8.5-UNIT-001` identified
- [x] Current `webhook_service.py` inspected — confirmed none of Story 8.5's functions exist yet

### Inputs Loaded

| Artifact | Purpose |
|---|---|
| `8-5-trial-expiry-handling-downgrade-logic.md` | Story spec, ACs, tasks, dev notes |
| `test-design-epic-08.md` | Epic-level coverage strategy, risk links (R-001, R-006) |
| `webhook_service.py` | Current code baseline — confirms what's missing |
| `test_webhook_service.py` | Unit test patterns (factories, patch targets, async mocking) |
| `test_stripe_webhook_flow.py` | Integration test patterns (seed, HTTP invoke, DB verify) |

---

## Step 2: Generation Mode

**Mode selected:** AI generation (backend stack — no browser recording needed)

**Rationale:** Story 8.5 is a pure backend webhook handler modification. All test scenarios
are expressible via function-level unit tests and DB-seeded integration tests. Playwright
recording is not applicable.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenario Mapping

| AC | Description | Test ID | Level | Priority | Why It Will Fail |
|---|---|---|---|---|---|
| AC1 | trial_will_end → publish trial.expiring | `test_trial_will_end_publishes_trial_expiring_event` | Unit | P2 (8.5-UNIT-001) | `_handle_trial_will_end` does not exist |
| AC1 | trial_will_end → sub not found → WARNING | `test_trial_will_end_subscription_not_found_logs_warning` | Unit | P2 | `_handle_trial_will_end` does not exist |
| AC1 | trial_will_end → publish failure swallowed | `test_trial_will_end_publish_failure_is_swallowed` | Unit | P2 | `_handle_trial_will_end` does not exist |
| AC2 | is_trial=True + past_due → free/active/is_trial=False | `test_subscription_updated_past_due_with_is_trial_downgrades_to_free` | Unit | P0 | upsert has no trial expiry override |
| AC2 | is_trial=True + canceled → free/active/is_trial=False | `test_subscription_updated_canceled_with_is_trial_downgrades_to_free` | Unit | P0 | upsert has no trial expiry override |
| AC2 | is_trial=True + incomplete_expired → free/active/is_trial=False | `test_subscription_updated_incomplete_expired_with_is_trial_downgrades_to_free` | Unit | P2 | upsert has no trial expiry override |
| AC2 (regression) | is_trial=False + past_due → raw status preserved | `test_subscription_updated_past_due_without_is_trial_preserves_raw_status` | Unit | P0 | guard: fails until Task 2.1 correct |
| AC2 | deleted + is_trial=True → free/active | `test_subscription_deleted_with_is_trial_downgrades_to_free_active` | Unit | P0 | is_deleted branch sets canceled unconditionally |
| AC2 (regression) | deleted + is_trial=False → free/canceled | `test_subscription_deleted_without_is_trial_sets_free_canceled` | Unit | P1 | regression guard for Story 8.4 |
| AC5 | trial expiry → subscription.changed published | `test_trial_expiry_publishes_subscription_changed_after_commit` | Unit | P1 | process_webhook lacks trial detection |
| AC1+AC7 | trial_will_end dispatch to handler only | `test_process_webhook_trial_will_end_dispatches_to_handle_trial_will_end` | Unit | P1 | no trial_will_end branch exists |
| AC6 | trial expiry audit → billing.trial_expired | `test_process_webhook_trial_expiry_audit_entry_uses_trial_expired_action` | Unit | P1 | always uses billing.webhook_processed |
| AC2+AC3+AC5+AC6+AC7 | Full trial expiry integration flow | `test_trial_expiry_webhook_downgrades_subscription_preserves_data` | Integration | P0 (8.5-E2E-001) | all 4 tasks missing |
| AC1+AC7 | Duplicate trial_will_end deduplicated | `test_duplicate_trial_will_end_event_is_deduplicated` | Integration | P1 | trial_will_end not dispatched yet |

### Test Level Rationale

- **Unit tests** (10 tests): Mock-based; exercise individual functions (`_handle_trial_will_end`,
  `_handle_subscription_upsert`, `process_stripe_webhook`). No external dependencies.
- **Integration tests** (2 tests): Require real DB session (testcontainers from conftest).
  Cover full stack: function → DB write → DB verify.
- **No E2E (Playwright)**: Story 8.5 modifies only backend webhook handlers and has no UI changes.
  The epic test design note confirms "P0 test 8.5-E2E-001 is verifiable as an integration test".

### Priority Distribution

| Priority | Count | Coverage Focus |
|---|---|---|
| P0 | 6 | Core trial expiry downgrade path (AC2), regression guards, full integration flow |
| P1 | 5 | subscription.changed publish, audit action type, dispatch, deletion regression, dedup |
| P2 | 4 | trial_will_end handler (AC1), incomplete_expired status |
| P3 | 0 | — |
| **Total** | **12** | All 7 ACs covered |

---

## Step 4: TDD Red Phase — Generated Failing Tests

### 🔴 TDD Red Phase Compliance

All tests use `@pytest.mark.skip(reason="🔴 RED PHASE: ...")` — they are intentionally failing.

```
✅ TDD Red Phase Validation: PASS
- All 12 tests decorated with @pytest.mark.skip
- All tests assert EXPECTED behaviour (not placeholder assertions)
- All tests marked with reason explaining what must be implemented
- No tests with placeholder assertions (assert True, etc.)
```

### Generated Test Files

#### 1. Unit Tests — `services/client-api/tests/unit/test_trial_expiry_handling.py`

| Test Method | Class | Priority | AC | Fails Because |
|---|---|---|---|---|
| `test_trial_will_end_publishes_trial_expiring_event` | `TestHandleTrialWillEnd` | P2 | AC1 | `_handle_trial_will_end` missing |
| `test_trial_will_end_subscription_not_found_logs_warning` | `TestHandleTrialWillEnd` | P2 | AC1 | `_handle_trial_will_end` missing |
| `test_trial_will_end_publish_failure_is_swallowed` | `TestHandleTrialWillEnd` | P2 | AC1 | `_handle_trial_will_end` missing |
| `test_subscription_updated_past_due_with_is_trial_downgrades_to_free` | `TestTrialExpiryDowngrade` | P0 | AC2 | upsert no trial override |
| `test_subscription_updated_canceled_with_is_trial_downgrades_to_free` | `TestTrialExpiryDowngrade` | P0 | AC2 | upsert no trial override |
| `test_subscription_updated_incomplete_expired_with_is_trial_downgrades_to_free` | `TestTrialExpiryDowngrade` | P2 | AC2 | upsert no trial override |
| `test_subscription_updated_past_due_without_is_trial_preserves_raw_status` | `TestTrialExpiryDowngrade` | P0 | AC2 regression | guard for Story 8.4 |
| `test_subscription_deleted_with_is_trial_downgrades_to_free_active` | `TestTrialExpiryDowngrade` | P0 | AC2 | is_deleted sets canceled unconditionally |
| `test_subscription_deleted_without_is_trial_sets_free_canceled` | `TestTrialExpiryDowngrade` | P1 | AC2 regression | guard for Story 8.4 |
| `test_trial_expiry_publishes_subscription_changed_after_commit` | `TestTrialExpiryPublishesSubscriptionChanged` | P1 | AC5 | no trial expiry detection |
| `test_process_webhook_trial_will_end_dispatches_to_handle_trial_will_end` | `TestProcessWebhookDispatch` | P1 | AC1+AC7 | no trial_will_end branch |
| `test_process_webhook_trial_expiry_audit_entry_uses_trial_expired_action` | `TestProcessWebhookDispatch` | P1 | AC6 | wrong audit action type |

**Unit test count: 10** (TestHandleTrialWillEnd: 3, TestTrialExpiryDowngrade: 6, TestTrialExpiryPublishesSubscriptionChanged: 1, TestProcessWebhookDispatch: 2)

#### 2. Integration Tests — `services/client-api/tests/integration/test_trial_expiry_flow.py`

| Test Method | Class | Priority | Epic Test ID | Fails Because |
|---|---|---|---|---|
| `test_trial_expiry_webhook_downgrades_subscription_preserves_data` | `TestTrialExpiryFlow` | P0 | 8.5-E2E-001 | all 4 tasks missing |
| `test_duplicate_trial_will_end_event_is_deduplicated` | `TestTrialExpiryFlow` | P1 | AC1+AC7 | trial_will_end not dispatched |

**Integration test count: 2**

**Total test count: 12** (10 unit + 2 integration)

### Fixture Needs

Unit tests use inline mock factories — no shared fixtures needed beyond what exists in
`tests/unit/__init__.py` and `tests/conftest.py`.

Integration tests require (from existing conftest.py):
- `db_session: AsyncSession` — per-test async DB session
- `async_session_factory: async_sessionmaker` — for seeding and verifying DB state

No new fixtures needed for the ATDD red phase.

---

## Step 5: Acceptance Criteria Coverage Map

| AC | Description | Test(s) | Priority | Status |
|---|---|---|---|---|
| AC1 | trial_will_end → publish trial.expiring to Redis Streams | `test_trial_will_end_publishes_trial_expiring_event`, `test_trial_will_end_subscription_not_found_logs_warning`, `test_trial_will_end_publish_failure_is_swallowed`, `test_process_webhook_trial_will_end_dispatches_*`, `test_duplicate_trial_will_end_*` | P1/P2 | 🔴 RED |
| AC2 | Trial expiry detection: is_trial=True + bad status → free/active | `test_subscription_updated_past_due_*`, `test_subscription_updated_canceled_*`, `test_subscription_updated_incomplete_expired_*`, `test_subscription_deleted_with_is_trial_*`, integration test | P0/P2 | 🔴 RED |
| AC2 (regression) | Non-trial past_due → raw status preserved | `test_subscription_updated_past_due_without_is_trial_*`, `test_subscription_deleted_without_is_trial_*` | P0/P1 | 🔴 RED |
| AC3 | Data preservation (no rows deleted) | Integration test: asserts sub row still exists | P0 | 🔴 RED |
| AC4 | Free tier enforcement via E06 | Covered by AC2 correctness (tier=free written to DB) | P0 | N/A (no code change needed) |
| AC5 | subscription.changed published after commit | `test_trial_expiry_publishes_subscription_changed_after_commit`, integration test | P0/P1 | 🔴 RED |
| AC6 | Audit trail: billing.trial_expired action type | `test_process_webhook_trial_expiry_audit_entry_uses_trial_expired_action`, integration test | P1 | 🔴 RED |
| AC7 | Unit tests per story spec (all scenarios) | All 10 unit tests + 2 integration tests | P0–P2 | 🔴 RED |

---

## Next Steps: TDD Green Phase

After implementing Story 8.5 Tasks 1–4 in `webhook_service.py`:

### Task → Test Mapping

| Task | What to Implement | Tests That Should Turn Green |
|---|---|---|
| Task 1 | Add `_handle_trial_will_end()` + `_publish_trial_expiring()` to `webhook_service.py` | `TestHandleTrialWillEnd` (3 unit tests) |
| Task 2.1 | Modify `_handle_subscription_upsert()` — add trial expiry detection in both `is_deleted` and `else` branches | `TestTrialExpiryDowngrade` (6 unit tests) |
| Task 2.2 | Modify `process_stripe_webhook()` — detect trial expiry, use `action_type="billing.trial_expired"` | `TestProcessWebhookDispatch.test_...audit_entry_uses_trial_expired_action` |
| Task 3 | Add `trial_will_end` dispatch branch in `process_stripe_webhook()` | `TestProcessWebhookDispatch.test_...dispatches_*`, `TestTrialExpiryPublishesSubscriptionChanged` |
| Task 4 | Unit test file exists (created here as red-phase) | Integration tests turn green |

### Green Phase Steps

1. Implement Tasks 1–4 in `services/client-api/src/client_api/services/webhook_service.py`
2. Remove `@pytest.mark.skip` decorators from all tests in both test files
3. Run unit tests: `make test-unit` (from `eusolicit-app/`)
4. Run integration tests: `make test-integration` (requires infra)
5. Verify **all 12 tests pass** (green phase)
6. Run regression suite for Story 8.4: `pytest tests/unit/test_webhook_service.py tests/integration/test_stripe_webhook_flow.py -v`
7. Coverage must remain ≥80%

### Run Commands

```bash
# From eusolicit-app/

# Unit tests only (fast, no external deps)
pytest services/client-api/tests/unit/test_trial_expiry_handling.py -v --no-header

# Integration tests (requires DB + Redis)
pytest services/client-api/tests/integration/test_trial_expiry_flow.py -v --no-header

# Story 8.4 regression suite
pytest services/client-api/tests/unit/test_webhook_service.py \
       services/client-api/tests/integration/test_stripe_webhook_flow.py -v

# All billing-related tests
pytest services/client-api/tests/ -k "billing or webhook or trial or subscription" -v
```

---

## Risk Mitigations Addressed

| Risk ID | Description | Tests Covering |
|---|---|---|
| R-001 | Webhook race conditions / duplicate events | `test_duplicate_trial_will_end_event_is_deduplicated` (integration) |
| R-006 | Cache invalidation failure on tier change | AC5 coverage: subscription.changed verified published (triggers cache eviction in E06) |

---

## Implementation Guidance for Developer

### Files to Modify (NO new files except tests)

```
services/client-api/src/client_api/services/webhook_service.py
  ├─ Add: _handle_trial_will_end(stripe_sub_obj, session) → Subscription | None
  ├─ Add: _publish_trial_expiring(company_id, trial_end_iso) — thin wrapper around EventPublisher
  ├─ Modify: _handle_subscription_upsert() — trial expiry detection in is_deleted + else branches
  └─ Modify: process_stripe_webhook() — trial_will_end dispatch + billing.trial_expired audit
```

### Critical Constraints (from Dev Notes)

- `status="active"` on trial expiry is intentional — company retains Free-tier access
- `days_remaining=3` is a constant (Stripe always sends trial_will_end 3 days before end)
- `incomplete_expired` must be in the detection set alongside `past_due` and `canceled`
- Publish `trial.expiring` AFTER `session.commit()` in `process_stripe_webhook()` (not inside `_handle_trial_will_end`)
- Override ONLY when `sub.is_trial == True` — paid subscriptions going `past_due` must NOT be intercepted

### Patch Targets for Mock Verification

```python
# EventPublisher publish
"client_api.services.webhook_service.EventPublisher.publish"

# subscription.changed publisher
"client_api.services.webhook_service._publish_subscription_changed"

# trial.expiring publisher (new helper)
"client_api.services.webhook_service._publish_trial_expiring"

# audit writer
"client_api.services.webhook_service.write_audit_entry"

# settings
"client_api.services.webhook_service.get_settings"

# new handlers
"client_api.services.webhook_service._handle_trial_will_end"
"client_api.services.webhook_service._handle_subscription_upsert"
```

---

## Summary Statistics

```
✅ ATDD Generation Complete (TDD RED PHASE)

🔴 TDD Red Phase: Failing Tests Generated

📊 Summary:
- Total Tests: 12 (all with @pytest.mark.skip)
  - Unit Tests: 10 (RED)
  - Integration Tests: 2 (RED)
- Fixtures Created: 0 new (reusing existing conftest.py)
- All tests will FAIL until Story 8.5 Tasks 1–4 implemented

✅ Acceptance Criteria Coverage:
  AC1 — ✅ 5 tests (trial_will_end handler, dispatch, deduplication)
  AC2 — ✅ 6 tests (downgrade detection: 3 statuses + regression guards + deleted)
  AC3 — ✅ 1 test (integration: sub row existence verified after webhook)
  AC4 — ✅ Covered by AC2 correctness (tier=free written to DB, E06 reads it)
  AC5 — ✅ 2 tests (subscription.changed published after commit)
  AC6 — ✅ 2 tests (billing.trial_expired audit action)
  AC7 — ✅ All 12 tests cover story-specified test scenarios

📂 Generated Files:
  - services/client-api/tests/unit/test_trial_expiry_handling.py (10 unit tests)
  - services/client-api/tests/integration/test_trial_expiry_flow.py (2 integration tests)
  - eusolicit-docs/test-artifacts/atdd-checklist-8-5-trial-expiry-handling-downgrade-logic.md

📝 Next Steps:
  1. Implement Story 8.5 Tasks 1–4 in webhook_service.py
  2. Remove @pytest.mark.skip from all tests
  3. Run: pytest services/client-api/tests/unit/test_trial_expiry_handling.py -v
  4. Run: pytest services/client-api/tests/integration/test_trial_expiry_flow.py -v
  5. Verify all 12 tests PASS (green phase)
  6. Run Story 8.4 regression suite to confirm no regressions
```

---

**Generated by:** BMad TEA Agent — ATDD Workflow (bmad-testarch-atdd)
**Workflow:** `bmad-testarch-atdd` (Create mode, sequential execution)
**Version:** BMad v6
