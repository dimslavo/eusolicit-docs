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
**Status**: Approved

**Findings**:
1. **Idempotency Guard Breaks Event Retry (Risk R-001)**: The idempotency logic introduced in `_dispatch_alert` commits the `AlertLog` row before dispatching the `send_email.delay()` task. If the Celery broker is temporarily unreachable and `delay()` raises an exception, the event is correctly left un-ACKed and retried. However, on retry, the idempotency check finds the already-committed `AlertLog` and silently skips the email dispatch. This causes permanent data loss for the notification, completely violating the mitigation for Risk R-001. The Celery dispatch must occur *before* the `AlertLog` commit, or the `AlertLog` commit and Celery dispatch must be made atomic (e.g., via a transactional outbox). *(Resolved by dev)*

2. **Blocking Call in Async Loop (`ARCHITECTURAL_DRIFT`)**: `send_email.delay(email_payload)` is a blocking, synchronous network call to the Celery broker. Because `_dispatch_alert` is an `async def` function running on an `asyncio` event loop, this synchronous call blocks the entire event loop for the duration of the broker connection and write operation. This prevents the consumer from processing other events concurrently or acknowledging heartbeats, severely degrading throughput. It must be wrapped in `await asyncio.to_thread(send_email.delay, email_payload)` or replaced with an async-native Celery dispatch.
3. **Race Condition in `process_pending` leading to False DLQ Entries (`code_quality`)**: `EventConsumer.process_pending()` pulls pending messages and checks `times_delivered > max_retries`. If it exceeds the limit, it instantly `XCLAIM`s them with `min_idle_time=0` and writes them to the DLQ. However, if a message was just successfully claimed by another consumer via `retry_pending()`, its `times_delivered` is incremented and its idle time is reset. Because `process_pending` does not respect a minimum idle time, it will instantly "steal" the message from the consumer that is actively processing it, marking it as a DLQ failure. `process_pending` should require a `min_idle_time` (e.g., > 30s) to avoid stealing actively processing messages.

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
- All 4 tasks + 2 review follow-up tasks resolved (6 total).
- Tasks 1-3: 2 blocking + 1 deferrable from first code review.
- Task 4: ordering bug from second code review.
- Review Follow-up 1 (Finding 2, third review): `asyncio.to_thread` wrapping for non-blocking Celery dispatch.
- Review Follow-up 2 (Finding 3, third review): `min_idle_ms=30_000` guard in `process_pending` DLQ claim.
- 90 unit tests pass (73 pre-existing notification + 17 new for this story).
- ruff clean on all modified files.
- ✅ Resolved review finding [blocking]: Blocking call in async loop — `send_email.delay()` now dispatched via `asyncio.to_thread`.
- ✅ Resolved review finding [blocking]: Race condition in `process_pending` — `min_idle_time` set to 30 000 ms (30 s) to prevent stealing actively-processing messages.

## File List
- `eusolicit-app/services/notification/src/notification/workers/opportunity_consumer.py` — modified
- `eusolicit-app/services/notification/src/notification/models/alert_log.py` — modified (stream_message_id column)
- `eusolicit-app/services/notification/alembic/versions/003_alert_log_stream_message_id.py` — new
- `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/consumer.py` — modified (retry_pending; process_pending min_idle_ms fix)
- `eusolicit-app/services/notification/tests/worker/__init__.py` — new
- `eusolicit-app/services/notification/tests/worker/test_opportunity_consumer.py` — modified (4 new tests for review follow-ups)

## Change Log
- 2026-04-19: Addressed 3 code-review findings (blocking: region null-guard, PEL retry; deferrable: idempotency). Added migration 003, 12 new unit tests. 177/177 tests pass.
- 2026-04-19: Fixed idempotency ordering bug (second review finding): `send_email.delay()` now called before `session.commit()` in `_dispatch_alert`. Added 1 new regression test (`test_alert_log_not_committed_when_celery_dispatch_fails`). 86/86 unit tests pass.
- 2026-04-19: Addressed 2 blocking review follow-ups from third code review: (1) wrapped `send_email.delay()` in `asyncio.to_thread()` to avoid blocking event loop; (2) fixed `process_pending` to use `min_idle_ms=30_000` instead of `0` to prevent false DLQ entries from race condition. Added 4 new unit tests. 90/90 unit tests pass.

## Status
review
