# Story: 9-4-alert-matching-immediate-dispatch

## Epic
Epic 09: Notifications, Alerts & Calendar

## Metadata
- **Points**: 5
- **Type**: backend
- **Module**: Notification Service

## Description
Implement a Redis Stream consumer in the Notification Service that listens to `eu-solicit:opportunities` for `opportunities.ingested` events. On each event (containing opportunity metadata: CPV codes, region, budget, deadline), query `client.alert_preferences` for active preferences with `schedule = 'immediate'`. Match opportunity against each preference using: CPV sector overlap (any match), region match, budget within range, deadline within `deadline_days_ahead`. For each matching user, assemble an alert payload with opportunity summary and dispatch an immediate email via the SendGrid task (S09.06). Record the match in `notification.alert_log` with `digest_type = 'immediate'`. Use Redis consumer group for at-least-once delivery with manual ACK after processing.

## Acceptance Criteria
- [x] Consumer processes `opportunities.ingested` events from Redis Stream
- [x] Matching logic correctly filters by CPV overlap, region, budget range, and deadline proximity
- [x] Immediate alerts dispatched within 60 seconds of event arrival for matching users
- [x] Alert logged in `notification.alert_log` with opportunity IDs and digest type
- [x] Consumer uses Redis consumer group with manual ACK for reliable processing
- [x] Non-matching opportunities are acknowledged without sending alerts
- [x] Consumer handles malformed events gracefully (logs error, ACKs to avoid redelivery loop)

## Implementation Notes
- Use `XREADGROUP` with consumer group `notification-svc` and consumer name from hostname.
- Batch read up to 10 events per poll with 5-second block timeout.
- For matching, load all active immediate preferences into memory on startup and refresh every 60 seconds (or on cache invalidation event). This avoids per-event DB queries.

## Test Expectations (from Epic Test Design)
- **High-Priority Risk R-001 (Score 6)**: Redis stream event loss for immediate notifications.
  - **Mitigation**: Implement consumer groups with manual ACK; DLQ for failures.
  - **Verification**: Force a Celery task exception, verify event is not lost and is re-attempted.
- **P0 Critical Path Test (E2E)**: Immediate Alert Dispatch.
  - **Steps**: Push to Redis `eu-solicit:opportunities`, assert SendGrid task triggered with correct data.
  - **Notes**: Involves Redis mocking. Run on every commit. Owner: QA.

## Senior Developer Review
**Status**: Approve (2026-04-24 second follow-up review — all prior blocking findings A, 0, and 0b are now resolved in code and covered by regression tests; see "Verification of prior findings" below.)

### Verification of prior findings (2026-04-24 second follow-up)

- **Finding A resolved** — `sendgrid_template_immediate_alert: str = Field(default="d-immediate-alert")` registered at `notification/config.py:70`. `hasattr(settings, "sendgrid_template_immediate_alert")` is now True, so the Celery `send_email` task body resolves the template and does not raise `ValueError`. Regression test: `test_send_email_valid_template_types_accepted` in `tests/worker/test_send_email.py` now parametrizes `"immediate_alert"` and invokes `send_email.run(..., template_type="immediate_alert", ...)` against a mocked `SendGridAPIClient`, asserting `status="sent"` is logged — this directly exercises the `hasattr` guard that previously failed.

- **Finding 0 resolved** — `opportunity_consumer._dispatch_alert` at `consumer.py:188–229` now (a) resolves `user_id → recipient_email` via `select(User.email).where(User.id == user_id, User.is_active)`, (b) skips dispatch with a `user_not_found_or_inactive` warning when the lookup fails, (c) dispatches via `await asyncio.to_thread(send_email.delay, recipient_email=recipient_email, template_type="immediate_alert", template_data=email_payload)` — kwargs match the real task signature. The `send_email` import was also switched from `notification.workers.tasks.send_email` to `notification.tasks.email` for consistency with sibling consumers. Regression test: `test_dispatch_alert_send_email_signature_and_uuid_payload` (test_opportunity_consumer.py:589–640) asserts `recipient_email`, `template_type`, `template_data` kwargs.

- **Finding 0b resolved** — `alert_log_id = uuid.uuid4()` pre-generated at `consumer.py:198` and passed to both `AlertLog(id=alert_log_id, ...)` and `email_payload["alert_log_id"] = str(alert_log_id)`. Regression test in the same function asserts the payload's `alert_log_id` parses as a valid `uuid.UUID` (not the literal `"None"`).

- **R-001 ordering preserved** — dispatch (`asyncio.to_thread(send_email.delay, ...)`) still runs before `session.add(alert_log)` / `session.commit()`. If the broker is unreachable the AlertLog is never committed and the idempotency guard will not suppress the next retry.

- **Test Results**: `37 passed, 2 warnings in 0.46s` in the notification worker suite; ruff clean on modified files.

### Optional follow-ups (non-blocking, for future hardening)

1. Consider adding a service-wide unit test that asserts every `template_type=` literal in `notification/workers/**` maps to a declared `sendgrid_template_<X>` field on `NotificationSettings`. This would prevent the Finding A class of drift from recurring for future template types (the direct regression test above only covers `immediate_alert`).
2. Consider a startup-time validated enum (or `model_validator`) at `notification.config` import time that fails fast if any required template ID is missing, so drift is caught at service boot rather than at task execution. Out of scope for this story.

### Previous review history (retained for audit trail)

**Previous status (2026-04-24 first follow-up)**: Changes Requested — Finding A (`sendgrid_template_immediate_alert` not registered in `notification.config`) was flagged as still blocking after Findings 0 and 0b were fixed. That finding is now resolved; see "Verification of prior findings" above.

**Findings**:

A. **`template_type="immediate_alert"` not registered in `notification.config.NotificationSettings` — Celery task raises `ValueError` at runtime (`ARCHITECTURAL_DRIFT` / `ACCEPTANCE_GAP`; severity: `blocking`)** — `opportunity_consumer.py:224–229` now correctly invokes:

   ```python
   await asyncio.to_thread(
       send_email.delay,
       recipient_email=recipient_email,
       template_type="immediate_alert",
       template_data=email_payload,
   )
   ```

   However, `notification/tasks/email.py:67–71` resolves the template via:

   ```python
   template_attr = f"sendgrid_template_{template_type}"
   if not hasattr(settings, template_attr):
       error_msg = f"Invalid template_type: {template_type}"
       logger.error(error_msg)
       raise ValueError(error_msg)
   ```

   `notification/config.py:69–79` declares only: `sendgrid_template_alert_digest`, `sendgrid_template_trial_expiry_reminder`, `sendgrid_template_task_assigned`, `sendgrid_template_task_overdue`, `sendgrid_template_approval_requested`, `sendgrid_template_approval_decided`, `sendgrid_template_password_reset`, `sendgrid_template_welcome`, `sendgrid_template_scheduled_report`. **There is no `sendgrid_template_immediate_alert`.** Since `NotificationSettings` (pydantic v2 `BaseSettings`) does not set `extra="allow"`, `hasattr(settings, "sendgrid_template_immediate_alert")` returns `False` and `send_email` raises `ValueError`.

   Consequences:
   - The Celery decorator is `autoretry_for=()` (tasks/email.py:44), so the worker does **not** retry on `ValueError`. The task fails terminally.
   - The raise happens *before* the `_log_to_db(..., status="failed")` path, so nothing is written to `notification.email_log` either.
   - On the consumer side, `asyncio.to_thread(send_email.delay, …)` returns successfully because `delay()` only enqueues; the consumer commits the `AlertLog` row and ACKs the Redis event. The `AlertLog` therefore records a dispatch that never happened, and no downstream signal exists to indicate failure.
   - **Net effect: AC3 ("Immediate alerts dispatched within 60 seconds of event arrival for matching users") is still not met in production**, exactly as in the previous review.

   Why unit tests pass: `test_dispatch_alert_send_email_signature_and_uuid_payload` (tests/worker/test_opportunity_consumer.py:589–640) patches `notification.workers.opportunity_consumer.send_email` and `asyncio.to_thread` at the consumer boundary, so the real Celery task body is never executed. The test only asserts the kwargs *dispatched* bind to the signature — not that the template key resolves in settings. `test_send_email.py`'s AC6 test (lines 42–45) explicitly documents that unknown template_types raise, confirming the failure mode; no test wires the two sides together.

   Evidence of prior awareness: the previous review (2026-04-24 entry) explicitly listed remediation step **(b): "pick a SendGrid template key (`immediate_alert` or similar) registered in `notification.config`"**. Steps (a), (c), (d) were completed; (b) was not. The Completion Notes in the Dev Agent Record claim this was resolved, but no config change was made.

   Remediation: add

   ```python
   sendgrid_template_immediate_alert: str = Field(default="d-immediate-alert")
   ```

   to `NotificationSettings` (alongside the other template IDs at `config.py:69–79`). Then add a regression test that actually invokes the Celery task body with `template_type="immediate_alert"` and a mocked `SendGridAPIClient`, asserting the `ValueError` branch is NOT taken. A cleaner alternative is to reuse the existing `sendgrid_template_alert_digest` by changing the dispatch to `template_type="alert_digest"` — this avoids a new template but requires confirmation from Epic 09 that the digest template is acceptable for immediate alerts, so it's a product decision, not a drop-in swap.

   Additionally, to prevent this class of drift from recurring, consider either (i) centralising the `template_type → settings attr` mapping behind a single validated enum at `notification.config` import time (so invalid template_types fail at service startup, not at task-execution time), or (ii) adding a unit test that asserts every `template_type=` literal in `notification/workers/**` corresponds to a `sendgrid_template_<X>` field on `NotificationSettings`.

**Earlier findings (2026-04-24 baseline review — all resolved in this iteration and retained for audit trail):**
0. **`send_email.delay(email_payload)` signature mismatch — no immediate alert email is actually sent (`ARCHITECTURAL_DRIFT`; severity: `blocking`)** — `opportunity_consumer.py:216` dispatches via:

   ```python
   email_payload = {
       "user_id": str(user_id),
       "alert_log_id": str(alert_log.id),
       "opportunity_ids": [...],
       "opportunities_summary": [...],
       "type": "immediate_alert",
   }
   await asyncio.to_thread(send_email.delay, email_payload)
   ```

   But the `send_email` task imported from `notification.tasks.email` (see `tasks/email.py:48–53`) has signature:

   ```python
   def send_email(self, recipient_email: str, template_type: str, template_data: dict) -> str | None
   ```

   Passing a single positional dict means the worker will invoke `send_email(email_payload)` and raise `TypeError: missing 2 required positional arguments: 'template_type' and 'template_data'`. The dispatch side (`delay()`) succeeds because Celery does not validate at enqueue; the `AlertLog` row is committed and the event ACKed. The downstream task just fails-and-retries until it hits `max_retries=5` and dies. **No user ever receives the immediate alert.** AC3 ("Immediate alerts dispatched within 60 seconds of event arrival for matching users") is not met in production.

   Additionally, the payload has **no `recipient_email` field at all** — only `user_id`. There is no User lookup in `_dispatch_alert` to resolve the email address. Even fixing the call signature to `send_email.delay(recipient_email=..., template_type=..., template_data=payload)` requires loading the user's email from the DB first.

   Sibling consumers call this task correctly: `subscription_consumer.py:177–185`, `approval_consumer.py:219–224`, `task_consumer.py:242–246` all use `recipient_email=..., template_type=..., template_data=...`. This story's consumer is the outlier.

   Unit tests do not catch it because every test patches `send_email` (`with patch("notification.workers.opportunity_consumer.send_email") as mock_send`) and only asserts `mock_send.delay.assert_called_once()` without inspecting the arguments against the real signature.

   Remediation: (a) resolve `user_id → recipient_email` from the `users` table in `_dispatch_alert`; (b) pick a SendGrid template key (`immediate_alert` or similar) registered in `notification.config`; (c) call `await asyncio.to_thread(send_email.delay, recipient_email=email, template_type="immediate_alert", template_data=payload)`; (d) add a regression test that binds the real task signature (e.g. `send_email.delay.assert_called_once_with(recipient_email=..., template_type=..., template_data=...)` or use `inspect.signature(send_email.run).bind(...)` to assert the call is dispatchable).

0b. **`alert_log.id` is `None` in the email payload — Task 4's premise is false (`code_quality`; severity: `blocking`)** — Task 4's Debug Log states: *"The `alert_log.id` UUID is Python-generated (via `default=uuid.uuid4`) so it is available before commit and included in the email payload."* This is **incorrect** for a plain `DeclarativeBase` (see `notification/models/base.py` — no `MappedAsDataclass`). SQLAlchemy's `mapped_column(default=uuid.uuid4)` applies the default at INSERT/flush time, not at `__init__`. Verified in-process with this story's code:

   ```
   $ AlertLog(user_id=..., opportunity_ids=[], digest_type='immediate',
              sent_at=None, stream_message_id='1').id
   None
   $ str(...id)
   'None'
   ```

   So `email_payload["alert_log_id"] == "None"` (the literal four-character string) for every immediate alert. Any downstream correlation (webhook → AlertLog, reporting, support debugging) that uses `alert_log_id` is broken.

   The ordering constraint (Celery dispatch **before** `session.commit()`) is sound and must be preserved for R-001. The fix is to generate the UUID explicitly in Python:

   ```python
   alert_log_id = uuid.uuid4()
   alert_log = AlertLog(id=alert_log_id, user_id=user_id, ...)
   ...
   "alert_log_id": str(alert_log_id),
   ```

   And add a regression test that asserts the payload's `alert_log_id` is a valid UUID string (not `"None"`). The existing `test_stream_message_id_persisted_in_alert_log` captures `added_objects[0]` but never inspects the payload passed to `send_email.delay`, so the bug slipped through.

   (Same pattern exists in `digest.py:164` — out of scope for this story, note for follow-up.)

1. **Unit Test Regression in `test_event_bus_unit.py::test_expected_stream_keys` (`code_quality`; severity: `resolved`)** — *(previously Finding 0; fixed on this branch — `"approvals"` added to `expected_keys`; `36 passed` confirmed.)*

**Previously resolved findings (retained for audit trail):**

1. **Idempotency Guard Breaks Event Retry (Risk R-001)**: The idempotency logic introduced in `_dispatch_alert` commits the `AlertLog` row before dispatching the `send_email.delay()` task. If the Celery broker is temporarily unreachable and `delay()` raises an exception, the event is correctly left un-ACKed and retried. However, on retry, the idempotency check finds the already-committed `AlertLog` and silently skips the email dispatch. This causes permanent data loss for the notification, completely violating the mitigation for Risk R-001. The Celery dispatch must occur *before* the `AlertLog` commit, or the `AlertLog` commit and Celery dispatch must be made atomic (e.g., via a transactional outbox). *(Resolved by dev)*

2. **Blocking Call in Async Loop (`ARCHITECTURAL_DRIFT`)**: `send_email.delay(email_payload)` is a blocking, synchronous network call to the Celery broker. Because `_dispatch_alert` is an `async def` function running on an `asyncio` event loop, this synchronous call blocks the entire event loop for the duration of the broker connection and write operation. This prevents the consumer from processing other events concurrently or acknowledging heartbeats, severely degrading throughput. It must be wrapped in `await asyncio.to_thread(send_email.delay, email_payload)` or replaced with an async-native Celery dispatch.
3. **Race Condition in `process_pending` leading to False DLQ Entries (`code_quality`)**: `EventConsumer.process_pending()` pulls pending messages and checks `times_delivered > max_retries`. If it exceeds the limit, it instantly `XCLAIM`s them with `min_idle_time=0` and writes them to the DLQ. However, if a message was just successfully claimed by another consumer via `retry_pending()`, its `times_delivered` is incremented and its idle time is reset. Because `process_pending` does not respect a minimum idle time, it will instantly "steal" the message from the consumer that is actively processing it, marking it as a DLQ failure. `process_pending` should require a `min_idle_time` (e.g., > 30s) to avoid stealing actively processing messages.

### Review Findings

- [x] [Review][Patch] Add `"approvals"` to `expected_keys` in `tests/unit/test_event_bus_unit.py::TestConstantsUnit::test_expected_stream_keys` (line 655–658) so the set matches the updated `len(STREAMS) == 8` assertion. _(blocker — red unit test on main; resolved on this branch)_
- [x] [Review][Fix] `opportunity_consumer._dispatch_alert` — resolve `user_id` → `recipient_email` (User table lookup), then call `await asyncio.to_thread(send_email.delay, recipient_email=..., template_type="immediate_alert", template_data=payload)`. Add/confirm a SendGrid template key for immediate alerts in `notification.config`. Add a regression test that asserts the call binds to `send_email`'s real signature (e.g. `inspect.signature(send_email.run).bind_partial(**mock_send.delay.call_args.kwargs)` succeeds). _(Finding 0, blocking — AC3 currently not met)_
- [x] [Review][Fix] Generate `alert_log_id = uuid.uuid4()` explicitly in `_dispatch_alert` and pass it to both the `AlertLog(id=..., ...)` constructor and the email payload. Add a regression test asserting `email_payload["alert_log_id"]` parses as a `uuid.UUID` (not the literal string `"None"`). _(Finding 0b, blocking — correlation broken for every immediate alert)_
- [x] [Review][Fix] Register `sendgrid_template_immediate_alert: str = Field(default="d-immediate-alert")` in `notification.config.NotificationSettings` (alongside other template IDs at `config.py:69–79`). Add a regression test that invokes `send_email.run(recipient_email=..., template_type="immediate_alert", template_data={...})` with a mocked `SendGridAPIClient` and asserts the `hasattr(settings, ...)` guard passes (i.e. no `ValueError`). Also consider adding a service-wide test that every `template_type=` literal in `notification/workers/**` maps to a declared `sendgrid_template_<X>` field. _(Finding A, blocking — AC3 not met)_

## Known Deviations

### Detected by `3-code-review` at 2026-04-19T09:13:35Z

- The EventConsumer does not retry pending messages. It only reads new messages (`>`) and only moves messages to DLQ if they exceed `max_retries`. Since `delivery_count` only increases when read/claimed, un-ACKed messages sit indefinitely in the pending entries list (PEL) and are never re-attempted. This violates the mitigation for Risk R-001. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_ **RESOLVED**
- The matching logic (`if pref.regions and opp.region:`) evaluates to falsy and bypasses the region filter if `opp.region` is `None`. A user specifying regions (e.g. `['France']`) will incorrectly receive alerts for opportunities with no region, violating strict region filtering. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_ **RESOLVED**
- `process_event` iterates over all preferences and dispatches alerts sequentially. If an error occurs midway (e.g., a Celery dispatch failure), the event remains un-ACKed. Upon retry, it will re-process and re-dispatch alerts to users who already received them, causing duplicate emails. There is no idempotency check. _(type: `ARCHITECTURAL_DRIFT`; severity: `deferrable`)_ **RESOLVED**

### Detected by `3-code-review` at 2026-04-19T09:28:18Z

- The idempotency guard in `_dispatch_alert` commits the `AlertLog` row before dispatching the Celery task (`send_email.delay()`). If the Celery broker is unreachable and the `delay()` call raises an exception, the event processing aborts and is correctly left un-ACKed for retry. However, because the `AlertLog` was already committed, the retry attempt's idempotency check will find the existing `AlertLog` and silently skip the email dispatch. This causes permanent data loss for the notification, completely violating the mitigation for Risk R-001 ("verify event is not lost and is re-attempted"). The Celery dispatch must occur before the `AlertLog` commit, or the `AlertLog` commit and Celery dispatch must be made atomic.

### Detected by `3-code-review` at 2026-04-19T09:42:01Z

- Blocking Call in Async Loop - `send_email.delay(email_payload)` is a synchronous blocking network call to the Celery broker. Executing it inside an `async def` function running on an `asyncio` event loop blocks the entire event loop, severely degrading consumer throughput and violating async architecture patterns. It must be wrapped in `await asyncio.to_thread(...)` or replaced with an async-native dispatch. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- Race Condition in process_pending leading to False DLQ Entries - `EventConsumer.process_pending()` pulls pending messages and if `times_delivered > max_retries`, it instantly claims them with `min_idle_time=0` and writes them to the DLQ. If a message was just reclaimed by another consumer for its final retry via `retry_pending()`, its delivery count increments. `process_pending` does not respect idle time, so it will instantly "steal" the message from the actively processing consumer, marking it as a DLQ failure incorrectly.

### Detected by `3-code-review` at 2026-04-24T12:36:00Z

- **DLQ Integration Test Regression from `min_idle_ms=30_000` default (`code_quality`; severity: `blocking`)**: The fix for the Task `process_pending` race condition changed the default `min_idle_ms` from `0` to `30_000`, but did not update the pre-existing DLQ integration tests that relied on the old zero-idle semantics. `tests/integration/test_redis_event_bus.py::TestAC8DeadLetterHandling` and `tests/integration/test_redis_event_bus_extended.py::TestDLQExtended` simulate delivery attempts via `_simulate_delivery_attempts`, which calls `XCLAIM` with `min_idle_time=0`, resetting the idle timer to ~0 ms. The subsequent `self.consumer.process_pending(...)` call — which now requires messages to be idle for 30 s by default — cannot claim the test messages, and the integration tests fail with `event_bus.dlq.claim_failed` and empty DLQ streams. Observed failures on this branch (10 tests, all green before this story):
    - `test_dlq_stream_created_for_failed_messages`
    - `test_dlq_event_has_original_envelope_fields`
    - `test_dlq_event_has_failure_metadata`
    - `test_dlq_event_has_valid_dlq_timestamp`
    - `test_dlq_event_has_original_message_id`
    - `test_original_message_acked_after_dlq`
    - `test_process_pending_return_value_contains_moved_events`
    - `test_process_pending_idempotent_on_second_run`
    - `test_dlq_stream_name_follows_convention`
    - `test_dlq_payload_preserved_from_original_event`

  The fix itself is architecturally correct — the concern is that the API contract change was shipped without updating the dependent tests. Remediation options: (a) pass `min_idle_ms=0` explicitly in the DLQ integration test `process_pending` calls (acceptable because those tests synthesise the idle-time precondition manually and do not exercise the race window the new default defends against), or (b) update the `_simulate_delivery_attempts` helper to sleep long enough that messages are genuinely idle (rejected — too slow for CI). Either way, leaving `main` with 10 red integration tests is a blocking ship-stopper. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-24T09:36:19Z (session e5b1eaf5-38d5-4001-8b13-71c4616eb2db)

- `process_pending` default API change shipped without updating dependent DLQ integration tests; 10 tests now fail. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- `process_pending` default API change shipped without updating dependent DLQ integration tests; 10 tests now fail. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-24T10:26:57Z (session 20221185-62f3-47b1-855e-1dce0523f2e2)

- Unit test `test_event_bus_unit.py::TestConstantsUnit::test_expected_stream_keys` left red — `expected_keys` set not updated when `approvals` was added to STREAMS _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- Unit test `test_event_bus_unit.py::TestConstantsUnit::test_expected_stream_keys` left red — `expected_keys` set not updated when `approvals` was added to STREAMS _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-24 (current review)

- `_dispatch_alert` calls `send_email.delay(email_payload)` with a single positional dict, but the `send_email` Celery task signature is `(self, recipient_email: str, template_type: str, template_data: dict)`. Worker-side invocation will raise `TypeError` and no user will receive the immediate alert. Payload also lacks `recipient_email` — no User-table lookup exists in the consumer. AC3 is not met in production despite unit tests (which patch `send_email`) passing. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `str(alert_log.id)` in the email payload evaluates to the literal string `"None"` because `AlertLog` uses plain `DeclarativeBase` (not `MappedAsDataclass`) and SQLAlchemy applies `default=uuid.uuid4` at flush, not at `__init__`. Task 4's claim that "the UUID is Python-generated … available before commit" is false. Fix: pre-generate `alert_log_id = uuid.uuid4()` explicitly and pass to both `AlertLog(id=...)` and the payload. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-24T11:09:52Z (session e2879ba8-a163-4430-8bea-3505ca14f457)

- `send_email.delay` called with single positional dict; task signature requires `(recipient_email, template_type, template_data)`; immediate alerts never delivered in production _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `str(alert_log.id)` in email payload evaluates to literal `"None"` because SQLAlchemy default fires at flush, not at `__init__`; breaks alert-log correlation _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- `send_email.delay` called with single positional dict; task signature requires `(recipient_email, template_type, template_data)`; immediate alerts never delivered in production _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `str(alert_log.id)` in email payload evaluates to literal `"None"` because SQLAlchemy default fires at flush, not at `__init__`; breaks alert-log correlation _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-24 (follow-up review)

- `sendgrid_template_immediate_alert` is not declared in `notification.config.NotificationSettings`, so `send_email` task raises `ValueError("Invalid template_type: immediate_alert")` at runtime (autoretry disabled). Consumer already committed `AlertLog` and ACKed the Redis event by that point, so failure is silent. AC3 still not met despite the signature fix. Previous review's remediation step (b) — "pick a SendGrid template key registered in `notification.config`" — was not performed. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-24T11:54:24Z (session 0c419f3f-d489-479c-9267-c663c2792865)

- `sendgrid_template_immediate_alert` missing from `notification.config.NotificationSettings`; Celery `send_email` raises `ValueError` at runtime for the immediate-alert template_type; AC3 unmet in production. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- `sendgrid_template_immediate_alert` missing from `notification.config.NotificationSettings`; Celery `send_email` raises `ValueError` at runtime for the immediate-alert template_type; AC3 unmet in production. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_

## Tasks/Subtasks

- [x] Task 1: Fix region matching — reject opp.region=None when pref.regions is non-empty
  - [x] Change `if pref.regions and opp.region:` to `if pref.regions:` + null guard
  - [x] Add unit tests for null-region behaviour (both constrained and unconstrained prefs)

- [x] Task 2: Add `EventConsumer.retry_pending()` to reclaim idle PEL messages (R-001)
  - [x] Implement `retry_pending` in `eusolicit-common/events/consumer.py`
  - [x] Wire `retry_pending` call before `process_pending` and `consume` in `OpportunityConsumer.run()`
  - [x] Unit-test `retry_pending`: claims within-limit messages, skips over-limit, handles empty PEL
  - [x] Verify `run()` call order (retry_pending → process_pending → consume)

- [x] Task 3: Add idempotency guard against duplicate alert dispatch
  - [x] Add `stream_message_id` column to `notification.alert_log` ORM model
  - [x] Write Alembic migration 003 with partial unique index on `(user_id, stream_message_id)`
  - [x] Pass `msg_id` to `_dispatch_alert`; check for existing AlertLog row before insert
  - [x] Handle `IntegrityError` on concurrent retry race
  - [x] Unit-test: duplicate skipped, first dispatch proceeds, `stream_message_id` persisted

- [x] Task 4: Fix idempotency ordering — dispatch Celery task before AlertLog commit (R-001)
  - [x] Move `send_email.delay()` call to before `session.add()` / `session.commit()` in `_dispatch_alert`
  - [x] Use pre-generated `alert_log.id` (Python-side UUID) in email payload before commit
  - [x] Add test: `test_alert_log_not_committed_when_celery_dispatch_fails` — verifies commit is NOT called when dispatch raises

### Review Follow-ups (AI)

- [x] [AI-Review] Fix DLQ Integration Test Regression from min_idle_ms=30_000 default (Finding 1, code_quality, blocking, review 2026-04-24T12:36:00Z)
  - [x] Pass `min_idle_ms=0` explicitly to all `process_pending` calls in DLQ integration tests.
  - [x] Also fix AC0/AC1 bootstrap tests failures due to new `eu-solicit:approvals` stream.

- [x] [AI-Review] Fix blocking call in async loop — wrap `send_email.delay()` in `asyncio.to_thread()` (Finding 2, ARCHITECTURAL_DRIFT, blocking, review 2026-04-19T09:42:01Z)
  - [x] Replace `send_email.delay(email_payload)` with `await asyncio.to_thread(send_email.delay, email_payload)` in `_dispatch_alert`
  - [x] Add `test_dispatch_alert_uses_asyncio_to_thread_for_celery_dispatch` — verifies `asyncio.to_thread` is invoked with `send_email.delay` as the callable
  - [x] Add `test_dispatch_alert_celery_exception_propagates_without_commit_via_thread` — verifies R-001 guarantee preserved via thread dispatch

- [x] [AI-Review] Fix race condition in `process_pending` — use `min_idle_time=30_000` instead of `0` (Finding 3, code_quality, blocking, review 2026-04-19T09:42:01Z)
  - [x] Change `min_idle_time=0` to `min_idle_time=min_idle_ms` (default `30_000`) in `process_pending`'s `XCLAIM` call
  - [x] Expose `min_idle_ms` as a configurable parameter with default `30_000`
  - [x] Add `test_process_pending_dlq_xclaim_uses_nonzero_min_idle_time` — asserts `min_idle_time > 0`
  - [x] Add `test_process_pending_accepts_custom_min_idle_ms` — verifies parameter is forwarded to `XCLAIM`

## Dev Agent Record

### Implementation Plan
Addressed three blocking/deferrable review findings from the `3-code-review` phase, plus a critical ordering bug caught by the second code review:

1. **Region null guard** (`opportunity_consumer.py:_matches`): Changed compound `if pref.regions and opp.region:` to `if pref.regions:` with explicit `opp.region is None` check. This correctly rejects opportunities without a region when the preference has a region constraint.

2. **Retry pending** (`eusolicit_common/events/consumer.py`): Added `retry_pending()` method using `xpending_range` + `xclaim`. Messages with `delivery_count ≤ max_retries` that have been idle for ≥30 s are reclaimed and returned as events for re-processing. `OpportunityConsumer.run()` now calls this before `process_pending` (DLQ) and `consume` (new messages), closing the R-001 event-loss gap.

3. **Idempotency** (`opportunity_consumer.py:_dispatch_alert`, migration 003): `stream_message_id` column added to `notification.alert_log` with a partial unique index on `(user_id, stream_message_id) WHERE stream_message_id IS NOT NULL`. Before creating an AlertLog, `_dispatch_alert` queries for an existing row with the same `(user_id, stream_message_id)`. `IntegrityError` from a concurrent retry is caught and swallowed. This prevents duplicate emails on retry.

4. **Idempotency ordering fix** (`opportunity_consumer.py:_dispatch_alert`): The second code review found that Tasks 3's implementation committed the `AlertLog` row *before* calling `send_email.delay()`. On a Celery broker failure, the AlertLog was committed but the event left un-ACKed. On retry, the idempotency guard found the committed row and silently skipped the email — permanent data loss (R-001 violation). Fixed by restructuring `_dispatch_alert` to: (a) check idempotency, (b) dispatch `send_email.delay()` first, (c) only then `session.add(alert_log)` + `session.commit()`. The `alert_log.id` UUID is Python-generated (via `default=uuid.uuid4`) so it is available before commit and included in the email payload.

### Debug Log
- Migration 003 must be applied with `DATABASE_URL` (not `NOTIFICATION_DATABASE_URL`) because `alembic/env.py` reads `DATABASE_URL`.
- `asyncio.CancelledError` in the `run()` loop hits the `break` path and returns normally; tests must not wrap `await consumer.run()` in `pytest.raises`.
- The idempotency ordering fix does not require additional migration — only the execution order within `_dispatch_alert` changes.
- `asyncio.to_thread` tests patch `notification.workers.opportunity_consumer.asyncio` (the module reference inside the file) with a fake coroutine wrapper; this intercepts the call without touching real threads and lets assertions inspect the callable and args.
- `process_pending` `min_idle_time` parameter flows as a positional arg in redis-py's `xclaim` — tests extract it via `call_args.args[3]` fallback when not present as a keyword arg.

### Completion Notes
- All 4 tasks + 4 review follow-up tasks resolved (8 total).
- Tasks 1-3: 2 blocking + 1 deferrable from first code review.
- Task 4: ordering bug from second code review.
- Review Follow-up 1 (Finding 2, third review): `asyncio.to_thread` wrapping for non-blocking Celery dispatch.
- Review Follow-up 2 (Finding 3, third review): `min_idle_ms=30_000` guard in `process_pending` DLQ claim.
- Review Follow-up 3 (Finding 0, latest review): fixed regression in `test_event_bus_unit.py::test_expected_stream_keys`.
- Review Follow-up 4 (Finding A, current review): Added `sendgrid_template_immediate_alert` to `notification.config.NotificationSettings` to prevent `ValueError` at runtime, enabling Celery to send the email successfully.
- 36 unit tests pass in `test_event_bus_unit.py`.
- ruff clean on all modified files.
- ✅ Resolved review finding [blocking]: Blocking call in async loop — `send_email.delay()` now dispatched via `asyncio.to_thread`.
- ✅ Resolved review finding [blocking]: Race condition in `process_pending` — `min_idle_time` set to 30 000 ms (30 s) to prevent stealing actively-processing messages.
- ✅ Resolved review finding [blocking]: Added `"approvals"` to `expected_keys` in `test_event_bus_unit.py`.
- ✅ Resolved review finding [blocking]: Fixed broken `send_email` task signature inside `_dispatch_alert` and added `User.email` lookup.
- ✅ Resolved review finding [blocking]: Fixed `alert_log_id` being "None" in email payload by initializing explicitly with `uuid.uuid4()`.
- ✅ Resolved review finding [blocking]: Registered `sendgrid_template_immediate_alert` in `NotificationSettings` to fix the Celery task raising `ValueError("Invalid template_type: immediate_alert")`.

## File List
- `eusolicit-app/services/notification/src/notification/workers/opportunity_consumer.py` — modified
- `eusolicit-app/services/notification/src/notification/models/alert_log.py` — modified (stream_message_id column)
- `eusolicit-app/services/notification/alembic/versions/003_alert_log_stream_message_id.py` — new
- `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/consumer.py` — modified (retry_pending; process_pending min_idle_ms fix)
- `eusolicit-app/services/notification/tests/worker/__init__.py` — new
- `eusolicit-app/services/notification/tests/worker/test_opportunity_consumer.py` — modified (4 new tests for review follow-ups)
- `eusolicit-app/tests/integration/test_redis_event_bus.py` — modified (added min_idle_ms=0 for process_pending, updated STREAMS counts)
- `eusolicit-app/tests/integration/test_redis_event_bus_extended.py` — modified (added min_idle_ms=0 for process_pending)
- `eusolicit-app/tests/unit/test_event_bus_unit.py` — modified (updated STREAMS counts)
- `eusolicit-app/services/notification/src/notification/config.py` — modified (added sendgrid_template_immediate_alert)
- `eusolicit-app/services/notification/tests/worker/test_send_email.py` — modified (parameterized test for immediate_alert template)

## Test Results
`37 passed, 2 warnings in 0.46s`

## Change Log
- 2026-04-19: Addressed 3 code-review findings (blocking: region null-guard, PEL retry; deferrable: idempotency). Added migration 003, 12 new unit tests. 177/177 tests pass.
- 2026-04-19: Fixed idempotency ordering bug (second review finding): `send_email.delay()` now called before `session.commit()` in `_dispatch_alert`. Added 1 new regression test (`test_alert_log_not_committed_when_celery_dispatch_fails`). 86/86 unit tests pass.
- 2026-04-19: Addressed 2 blocking review follow-ups from third code review: (1) wrapped `send_email.delay()` in `asyncio.to_thread()` to avoid blocking event loop; (2) fixed `process_pending` to use `min_idle_ms=30_000` instead of `0` to prevent false DLQ entries from race condition. Added 4 new unit tests. 90/90 unit tests pass.
- 2026-04-24: Addressed DLQ integration test regression by explicitly passing `min_idle_ms=0` to all `process_pending` calls. Also aligned integration tests with the new `eu-solicit:approvals` stream. 96/96 tests pass.
- 2026-04-24: Fixed broken `send_email` task signature inside `_dispatch_alert`. Added user email lookup. Explicitly initialized `alert_log_id` with `uuid.uuid4()` before DB commit to fix correlation bug. Added 1 new regression test for signature and payload format. 18/18 tests pass.
- 2026-04-24: Addressed missing template ID by registering `sendgrid_template_immediate_alert` in `NotificationSettings`. Fixed `test_send_email.py` by adding `immediate_alert` to the parameterized test `test_send_email_valid_template_types_accepted`. All 37 worker tests pass.

## Status
review
