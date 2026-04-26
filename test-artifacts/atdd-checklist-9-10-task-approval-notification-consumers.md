---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-19'
workflowType: 'bmad-testarch-atdd'
mode: 'create'
storyKey: '9-10-task-approval-notification-consumers'
detectedStack: 'fullstack'
generationMode: 'AI Generation (backend story — no browser recording)'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/9-10-task-approval-notification-consumers.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-09.md'
  - 'eusolicit-app/services/notification/src/notification/workers/opportunity_consumer.py'
  - 'eusolicit-app/services/notification/tests/worker/test_opportunity_consumer.py'
  - 'eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py'
  - 'eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py'
  - 'eusolicit-app/services/notification/src/notification/main.py'
  - 'eusolicit-app/services/notification/src/notification/config.py'
  - 'eusolicit-app/services/notification/src/notification/tasks/email.py'
  - 'eusolicit-app/_bmad/bmm/config.yaml'
testFramework: 'pytest + pytest-asyncio'
---

# ATDD Checklist: Story 9.10 — Task & Approval Notification Consumers

**Story:** `9-10-task-approval-notification-consumers`
**Date:** 2026-04-19
**TDD Phase:** 🔴 RED (all tests skipped — failing until implementation complete)
**Stack:** fullstack / backend story
**Test Framework:** pytest + pytest-asyncio (`asyncio_mode="auto"`)

---

## TDD Red Phase Summary

✅ Failing tests generated — **8 test files**, **42 tests total**

All tests are marked `@pytest.mark.skip(reason="RED PHASE: ...")`.
They will run and **pass (GREEN)** when the corresponding implementation is complete.

---

## Generated Test Files

| File | Tests | Priority | ACs Covered |
|------|-------|----------|-------------|
| `packages/eusolicit-models/tests/test_events_task_approval.py` | 10 | P1 | AC10 |
| `packages/eusolicit-common/tests/events/test_bootstrap_approvals.py` | 6 | P1 | AC9 |
| `services/notification/tests/worker/test_task_consumer.py` | 12 | P1/P2 | AC1, AC2, AC5, AC6, AC7, AC8 |
| `services/notification/tests/worker/test_approval_consumer.py` | 12 | P1/P2 | AC3, AC4, AC5, AC6, AC7, AC8, AC12 |
| `services/notification/tests/unit/test_approval_links.py` | 7 | P1 | AC3, AC12 |
| `services/notification/tests/integration/test_task_consumer_integration.py` | 3 | P1/P0 | AC1, AC2, AC5 (E09-R-002) |
| `services/notification/tests/integration/test_approval_consumer_integration.py` | 2 | P1/P0 | AC3, AC5 (E09-R-002) |
| `services/notification/tests/test_lifespan_consumers.py` | 4 | P1 | AC11 |

**Total: 56 tests** (includes sub-scenarios; ~42 unique @pytest.mark.skip test functions)

---

## Acceptance Criteria Coverage

### ✅ AC1 — TaskAssigned dispatch
- [x] `test_task_assigned_dispatches_one_email_to_assignee` (unit) — template + recipient + template_data fields
- [x] `test_task_assigned_missing_assignee_warns_and_acks` (unit) — AC6 overlap: missing user → warn + ACK
- [x] `test_task_assigned_integration_dispatches_one_email` (integration) — real Redis, mocked send_email

### ✅ AC2 — TaskOverdue dispatch + team-lead CC
- [x] `test_task_overdue_dispatches_two_emails_assignee_and_team_lead` (unit) — 2 dispatches
- [x] `test_task_overdue_missing_team_lead_skips_cc_but_still_sends_assignee_email` (unit) — partial success
- [x] `test_task_overdue_no_team_lead_dispatches_only_assignee_email` (integration) — real Redis

### ✅ AC3 — ApprovalRequested fan-out to all approvers
- [x] `test_approval_requested_fan_out_to_all_approvers` (unit) — N approvers → N dispatches
- [x] `test_approval_requested_template_data_includes_signed_urls` (unit) — approve_url, reject_url
- [x] `test_approval_requested_per_approver_setnx_key_format` (unit) — key = `notification:dispatched:approval-req:{msg_id}:{approver_id}`
- [x] `test_approval_requested_partial_setnx_only_dispatches_missing_approvers` (unit) — partial replay
- [x] `test_approval_requested_integration_fan_out_to_three_approvers` (integration) — real Redis, 3 calls
- [x] `test_approval_requested_partial_idempotency_skips_already_dispatched_approver` (integration) — SETNX crash sim

### ✅ AC4 — ApprovalDecided dispatch to proposal owner
- [x] `test_approval_decided_dispatches_one_email_to_proposal_owner` (unit) — template + fields
- [x] `test_approval_decided_missing_owner_warns_and_acks` (unit) — AC6 overlap

### ✅ AC5 — Idempotent consumption (SETNX Redis guard)
- [x] `test_task_assigned_setnx_returns_false_skips_dispatch` (unit) — replay skipped
- [x] `test_task_assigned_setnx_key_format` (unit) — `notification:dispatched:task:{msg_id}`
- [x] `test_task_assigned_setnx_ttl_set_to_86400_seconds` (unit) — 24h TTL
- [x] `test_task_consumer_run_calls_retry_pending_before_consume` (unit) — loop order
- [x] `test_task_assigned_idempotency_no_duplicate_on_crash_redelivery` (integration) — E09-R-002
- [x] `test_approval_requested_partial_setnx_only_dispatches_missing_approvers` (unit) — per-approver
- [x] `test_approval_requested_partial_idempotency_skips_already_dispatched_approver` (integration)

### ✅ AC6 — Graceful handling of missing/unknown events
- [x] `test_unknown_event_type_logs_info_and_acks` (unit — TaskConsumer)
- [x] `test_task_assigned_missing_assignee_warns_and_acks` (unit)
- [x] `test_approval_consumer_unknown_event_type_logs_info_and_acks` (unit — ApprovalConsumer)
- [x] `test_approval_decided_missing_owner_warns_and_acks` (unit)

### ✅ AC7 — Malformed payload handling
- [x] `test_malformed_json_payload_logs_error_and_acks` (unit — TaskConsumer)
- [x] `test_pydantic_validation_failure_logs_error_and_acks` (unit — TaskConsumer)
- [x] `test_approval_consumer_malformed_payload_logs_error_and_acks` (unit — ApprovalConsumer)

### ✅ AC8 — Async-safe Celery dispatch (asyncio.to_thread)
- [x] `test_task_assigned_uses_asyncio_to_thread_for_celery_dispatch` (unit)
- [x] `test_approval_decided_uses_asyncio_to_thread` (unit)

### ✅ AC9 — Bootstrap additions
- [x] `test_streams_dict_includes_approvals_key` — `STREAMS["approvals"] == "eu-solicit:approvals"`
- [x] `test_existing_streams_unchanged` — no regression on existing streams
- [x] `test_consumer_groups_notification_svc_includes_approvals`
- [x] `test_consumer_groups_existing_notification_streams_intact`
- [x] `test_bootstrap_event_bus_creates_approvals_consumer_group`
- [x] `test_bootstrap_event_bus_tolerates_busygroup_on_approvals`
- [x] `test_bootstrap_event_bus_second_call_does_not_raise`

### ✅ AC10 — Event-schema contract
- [x] `test_task_overdue_schema_exists` — fields + discriminator
- [x] `test_task_overdue_opportunity_id_optional`
- [x] `test_approval_requested_schema_exists` — fields + discriminator
- [x] `test_approval_requested_required_approver_ids_is_list`
- [x] `test_approval_decided_schema_approved` — fields + discriminator
- [x] `test_approval_decided_schema_rejected_with_note`
- [x] `test_approval_decided_decision_rejects_invalid_value` — Pydantic validation
- [x] `test_service_event_parses_task_overdue` — union discriminator
- [x] `test_service_event_parses_approval_requested`
- [x] `test_service_event_parses_approval_decided`
- [x] `test_task_overdue_round_trip` — model_dump_json → validate_json
- [x] `test_approval_requested_round_trip`
- [x] `test_approval_decided_round_trip`

### ✅ AC11 — Consumer lifecycle
- [x] `test_lifespan_creates_task_consumer_task`
- [x] `test_lifespan_creates_approval_consumer_task`
- [x] `test_lifespan_opportunity_consumer_task_still_exists` (regression)
- [x] `test_lifespan_cancels_consumer_tasks_on_shutdown`

### ✅ AC12 — Security: no plaintext secrets in logs
- [x] `test_approval_secret_never_logged_during_dispatch` (unit — ApprovalConsumer)
- [x] `test_signed_url_does_not_contain_plaintext_secret` (unit — approval_links)

### Additional Coverage
- [x] `test_task_assigned_does_not_dispatch_to_different_user` — cross-tenant isolation (Story 9.9 lesson)
- [x] `test_build_signed_url_structure_approve` / `test_build_signed_url_structure_reject` — URL format
- [x] `test_build_signed_url_is_deterministic` — HMAC determinism
- [x] `test_approve_and_reject_urls_have_different_signatures` — action affects HMAC
- [x] `test_hmac_matches_manual_computation` — HMAC algorithm verification
- [x] `test_approval_link_default_ttl_is_seven_days`

---

## Not-yet-covered (defer to Implementation / bmad-testarch-automate)

| AC | Gap | Reason |
|----|-----|--------|
| AC9 | `eusolicit-test-utils/redis_utils.py` bootstrap mirror | Out of scope for consumer-level ATDD; Story 9.10 Task 2 covers implementation |
| AC3 | HMAC token expiry actually set 7 days from event.timestamp | Deferred to integration; requires time-frozen integration scenario |
| AC11 | `app.state.task_consumer_task` cancellation verified via asyncio inspection | TestClient makes task inspection complex; covered by smoke in production |

---

## Risk Traceability

| Risk | Test | Status |
|------|------|--------|
| E09-R-002 (Consumer duplicate dispatch) | `test_task_assigned_idempotency_no_duplicate_on_crash_redelivery` (integration) | 🔴 RED |
| E09-R-002 (Consumer duplicate dispatch) | `test_approval_requested_partial_idempotency_skips_already_dispatched_approver` (integration) | 🔴 RED |
| E09-R-002 (SETNX ordering: check before dispatch, set after) | `test_task_assigned_setnx_returns_false_skips_dispatch` (unit) | 🔴 RED |
| AC12 / Secret exposure | `test_approval_secret_never_logged_during_dispatch` | 🔴 RED |
| Cross-tenant isolation (Story 9.9 lesson) | `test_task_assigned_does_not_dispatch_to_different_user` | 🔴 RED |

---

## TDD Red → Green Protocol

When implementing Story 9.10, follow this test-by-test activation sequence:

### Phase 1: Event schemas + bootstrap (Stories 9.10 Tasks 1–2)
```bash
# Remove @pytest.mark.skip from:
# packages/eusolicit-models/tests/test_events_task_approval.py
# packages/eusolicit-common/tests/events/test_bootstrap_approvals.py
cd eusolicit-app
make test-service SVC=eusolicit-models
make test-service SVC=eusolicit-common
```

### Phase 2: Approval link signer (Task 4)
```bash
# Remove @pytest.mark.skip from:
# services/notification/tests/unit/test_approval_links.py
make test-service SVC=notification
```

### Phase 3: Unit tests — TaskConsumer (Task 5)
```bash
# Remove @pytest.mark.skip from:
# services/notification/tests/worker/test_task_consumer.py
make test-service SVC=notification
```

### Phase 4: Unit tests — ApprovalConsumer (Task 6)
```bash
# Remove @pytest.mark.skip from:
# services/notification/tests/worker/test_approval_consumer.py
make test-service SVC=notification
```

### Phase 5: Lifespan wiring (Task 7)
```bash
# Remove @pytest.mark.skip from:
# services/notification/tests/test_lifespan_consumers.py
make test-service SVC=notification
```

### Phase 6: Integration tests (Task 8) — requires `make infra`
```bash
# Remove @pytest.mark.skip from:
# services/notification/tests/integration/test_task_consumer_integration.py
# services/notification/tests/integration/test_approval_consumer_integration.py
make test-integration
```

---

## Quality Gate

- [ ] All 8 test files pass (GREEN) with `@pytest.mark.skip` removed
- [ ] `make lint` — ruff passes on all new test files
- [ ] `make type-check` — mypy passes on all new test files  
- [ ] Coverage ≥ 80% on `task_consumer.py`, `approval_consumer.py`, `approval_links.py`
- [ ] E09-R-002 idempotency integration tests pass (100% — non-negotiable)
- [ ] AC12 secret-not-in-logs test passes
- [ ] No regression on `test_opportunity_consumer.py` (all existing tests still green)

---

## Implementation Files Expected (New)

| File | Required By |
|------|-------------|
| `services/notification/src/notification/workers/task_consumer.py` | AC1, AC2, AC5–AC8 |
| `services/notification/src/notification/workers/approval_consumer.py` | AC3, AC4, AC5–AC8, AC12 |
| `services/notification/src/notification/core/approval_links.py` | AC3 |
| `services/notification/src/notification/models/user.py` | AC1, AC2, AC3, AC4 |
| `packages/eusolicit-models/tests/__init__.py` | Test discovery |
| `packages/eusolicit-common/tests/events/__init__.py` | Test discovery |

## Modified Files Expected

| File | Change |
|------|--------|
| `packages/eusolicit-models/src/eusolicit_models/events.py` | Add TaskOverdue, ApprovalRequested, ApprovalDecided + extend ServiceEvent |
| `packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py` | Add "approvals" to STREAMS + CONSUMER_GROUPS["notification-svc"] |
| `services/notification/src/notification/main.py` | Start task + approval consumer tasks in lifespan |
| `services/notification/src/notification/config.py` | Add approval_link_secret + app_base_url |

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Story:** 9-10-task-approval-notification-consumers
**Date:** 2026-04-19
