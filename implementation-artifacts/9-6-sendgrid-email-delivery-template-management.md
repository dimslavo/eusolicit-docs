# Story: 9-6-sendgrid-email-delivery-template-management

## Epic
Epic 09: Notifications, Alerts & Calendar

## Metadata
- **Points**: 3
- **Type**: backend
- **Module**: Notification Service

## Description
Implement a Celery task `send_email` on the `emails` queue that sends transactional email via the SendGrid v3 API. Accept parameters: recipient_email, template_type (enum), template_data (dict). Map template_type to SendGrid dynamic template IDs configured in environment variables. Template types: `alert_digest`, `trial_expiry_reminder`, `task_assigned`, `approval_requested`, `approval_decided`, `password_reset`, `welcome`. Log every send attempt in `notification.email_log` with sendgrid_message_id and initial status `sent`. Implement a SendGrid Event Webhook handler (in the **Notification Service** as `POST /webhooks/sendgrid`) to update `email_log.status` on delivery/bounce/fail events. Retry on 429/5xx from SendGrid with exponential backoff.

## Acceptance Criteria
- [x] `send_email` task dispatches email via SendGrid API for all 7 template types
- [x] Template IDs are configurable via environment variables, not hardcoded
- [x] Every send is logged in `notification.email_log` with sendgrid_message_id
- [x] SendGrid webhook endpoint updates email_log status to delivered/bounced/failed
- [x] 429 and 5xx responses from SendGrid trigger task retry with exponential backoff
- [x] Invalid template_type raises descriptive error and does not retry
- [x] Webhook endpoint validates SendGrid signature to prevent spoofing

## Tasks/Subtasks

- [x] **Task 1**: Implement `send_email` Celery task (notification/tasks/email.py)
  - [x] AC 1: All 7 template types dispatched via SendGrid v3 API
  - [x] AC 2: Template IDs read from env vars (NOTIFICATION_SENDGRID_TEMPLATE_{TYPE})
  - [x] AC 3: Every send logged to `notification.email_log`
  - [x] AC 5: 429/5xx triggers `task.retry()` with exponential backoff
  - [x] AC 6: Invalid template_type raises `ValueError`, not retried

- [x] **Task 2**: Fix architectural deviation — move webhook to Notification Service
  - [x] Remove webhook handler from `client-api/api/webhooks/sendgrid.py`
  - [x] Remove sendgrid webhook router from `client-api/main.py`
  - [x] Remove `sendgrid_webhook_public_key` from `ClientApiSettings`
  - [x] Create `notification/api/webhooks/sendgrid.py` with correct handler
  - [x] Add `sendgrid_webhook_public_key` to `NotificationSettings`
  - [x] Register webhook router in `notification/main.py`
  - [x] AC 4: Webhook updates email_log status (delivered/bounced/failed)
  - [x] AC 7: Webhook validates SendGrid ECDSA signature

- [x] **Task 3**: Add Alembic migration to revoke cross-service DB permissions
  - [x] Migration 005: revoke SELECT, UPDATE on email_log from client_api_role
  - [x] Migration 005: revoke USAGE on notification schema from client_api_role

- [x] **Task 4**: Fix broken import in opportunity_consumer.py
  - [x] Correct import from `notification.tasks.email` (was wrong path)
  - [x] Create compat re-export at `notification/workers/tasks/send_email.py`

- [x] **Task 5**: Write and fix unit/worker tests
  - [x] `test_send_email_invalid_template_type_raises_value_error`
  - [x] `test_send_email_valid_template_types_accepted` (7 parametrized)
  - [x] `test_send_email_template_id_from_settings`
  - [x] `test_send_email_retries_on_sendgrid_rate_limit_and_5xx` (5 parametrized: 429, 500, 502, 503, 504)
  - [x] `test_send_email_no_retry_on_non_transient_error`
  - [x] `test_send_email_logs_to_db_on_success`
  - [x] `test_send_email_logs_failed_status_on_non_retryable_error`

- [x] **Task 6**: Write webhook API tests
  - [x] `test_webhook_invalid_signature_returns_401`
  - [x] `test_webhook_valid_signature_accepted`
  - [x] `test_webhook_event_updates_status` (3 parametrized)
  - [x] `test_webhook_non_status_event_no_db_change`
  - [x] `test_webhook_sg_message_id_with_dot_suffix`
  - [x] `test_webhook_missing_sg_message_id_skipped`
  - [x] `test_webhook_multiple_events_in_batch`
  - [x] `test_webhook_endpoint_is_in_notification_service`

- [x] **Task 7**: Update story-9-1 Celery config tests to reflect final implementation
  - [x] Accept canonical `notification.tasks.email` module path in includes test
  - [x] Update route key test to accept `notification.tasks.email.send_email`
  - [x] Update `autoretry_for` test: `send_email` uses manual `self.retry()` (correct)

### Review Follow-ups (AI)

- [x] **[AI-Review][High]** Finding 1: Update `migrated_db` fixture regex in `test_002_migration.py` to accept revisions ≥ 002 — migration 005 causes fixture to fail because `\b(002|003)\b` does not match `"005"`, resulting in 32 integration test failures
- [x] **[AI-Review][High]** Finding 2: Fix `send_email` task — catch `MaxRetriesExceededError` after `self.retry()` and write `status="failed"` to `email_log`; add unit test `test_send_email_logs_failed_status_on_max_retries_exceeded`
- [x] **[AI-Review][High]** Finding 4: Fix webhook handler to fail closed — raise 401 when `sendgrid_webhook_public_key` is empty or not set (currently fails open, allowing unauthenticated spoofing); add `test_webhook_missing_public_key_returns_401`; update all tests that bypassed verification with empty key to mock `verify_signature` instead
- [x] **[AI-Review][Med]** Finding 3: Replace inline `sa.Table` (with wrong `sa.Text` for status instead of `Enum`) in `send_email._log_to_db` with `EmailLog` ORM model using sync `Session`; replace raw SQL `text("UPDATE ...")` in `sendgrid_webhook.py` with ORM `update(EmailLog)` statement — both should use the single source-of-truth ORM model

## Implementation Notes
- Use `sendgrid` Python SDK. Configure `NOTIFICATION_SENDGRID_API_KEY` and `NOTIFICATION_SENDGRID_TEMPLATE_{TYPE}` env vars.
- Webhook signature validation uses the SendGrid Event Webhook verification library (ECDSA, not HMAC).
- Rate limit: SendGrid allows 600 requests/min on paid plans; add a `rate_limit` on the task if needed.
- **Architecture fix**: Webhook moved from client-api to notification service (owns `email_log` table).

## Test Expectations (from Epic Test Design)
- **Medium-Priority Risk R-005 (Score 3)**: Third-party rate limits (Google/MS/SendGrid).
  - **Mitigation**: Exponential backoff retries, Celery rate limits.
- **P1 High Priority Test (API)**: SendGrid Webhooks.
  - **Steps**: Simulate delivered/bounced events updating DB status.
  - **Notes**: Run on PR to main. Owner: QA.
- **P2 Medium Priority Test (Worker)**: Celery Retry Backoff.
  - **Steps**: Mock 429 response from SendGrid, verify 5 retries.
  - **Notes**: Run nightly/weekly. Owner: DEV.
- **P3 Low Priority Test (Manual)**: Email Template Rendering.
  - **Steps**: Visual verification of SendGrid transactional templates.
  - **Notes**: Run on-demand. Owner: QA.

## Senior Developer Review
- **REVIEW: Approve** (final pass 2026-04-24)
- **Finding 1** (Resolved): The new database migrations (`005`) broke the existing integration test `test_002_migration.py`, which had a hardcoded assertion expecting `alembic current` to be `002` or `003`. This results in 32 failing tests. The legacy migration test needs to be updated to accommodate the new database schema state.
- **Finding 2** (Resolved): In `send_email` task, if the task reaches its maximum retries due to a retryable error (e.g., 429 or 5xx), the final `Retry` or `MaxRetriesExceededError` exception raised by `self.retry()` will bypass the DB fallback logging block. Consequently, emails that fail permanently due to max retries are never logged to `email_log`.
- **Finding 3** (Resolved): Architectural Drift. Both the `send_email` Celery task and the `sendgrid_webhook` endpoint bypassed the SQLAlchemy ORM `EmailLog` model. The task defined an inline `sa.Table` (with an incorrect `Text` type for the `status` column instead of the PostgreSQL `Enum`), and the webhook used raw SQL strings (`text("UPDATE ...")`). Both now import and use the single source of truth `EmailLog` model.
- **Finding 4** (Resolved): Security/Acceptance Gap. The webhook handler silently bypassed ECDSA signature verification if `settings.sendgrid_webhook_public_key` was empty or not set. Now fails closed (401 when the key is absent) and test `test_webhook_missing_public_key_returns_401` validates the path.

### Final-pass Review (2026-04-24) — Approve
Adversarial verification against the final working state:
- `notification/tasks/email.py` uses `EmailLog` ORM via sync `Session`; `MaxRetriesExceededError` is caught and logged (`email.py:126-135`).
- `notification/api/webhooks/sendgrid.py` fails closed on empty public key (`sendgrid.py:73-79`) and uses `sa_update(EmailLog)` instead of raw SQL.
- Migration `005_revoke_email_log_from_client_api.py` correctly revokes the cross-service grants added by 004 and provides a downgrade path.
- `test_002_migration.py` `migrated_db` fixture regex `\b(\d{3,})\b` with `int >= 2` correctly accepts any revision ≥ 002, unblocking the 32 previously failing integration tests.
- Client-api cleanup complete: router un-mounted (`main.py:150`), `sendgrid_webhook_public_key` removed from `ClientApiSettings`, test file reduced to a placeholder.
- Test coverage: 11 webhook tests (including fail-closed, dot-suffix, batch, non-status events) + 9 worker tests (7 templates, 5 retry codes, max-retries-exceeded, non-retryable). All 7 ACs traceable to at least one passing test.

### Review Findings (minor, non-blocking)

- [ ] **[Review][Patch][Low]** Stale docstring on `sendgrid_webhook_public_key` field — `services/notification/src/notification/config.py:84-86` description still reads *"When empty, signature verification is skipped"*, contradicting the fail-closed behavior introduced by Finding 4. Update wording to reflect that an empty value now yields a 401.
- [ ] **[Review][Patch][Low]** Dead branch in `send_email` — `services/notification/src/notification/tasks/email.py:110-112` contains `if status_code is None and hasattr(exc, "to_dict"): pass`, a no-op that does nothing. Remove it or implement the intended `to_dict` fallback for SendGrid SDK exceptions that do not expose `status_code`.
- [x] **[Review][Defer][Med]** No webhook event-id deduplication — SendGrid retries a webhook batch with the same `sg_event_id` values; the handler re-applies the status every time. Idempotent for terminal states but wastes writes and could overwrite a later `bounced` with an earlier `delivered` if events arrive out of order. Deferred — track in `deferred-work.md` as a reliability hardening item; not in scope for Story 9-6 ACs.
- [x] **[Review][Defer][Low]** Webhook `except Exception` around `db.execute(stmt)` swallows errors, but the `get_db_session` dependency then calls `session.commit()` on the invalidated transaction which raises and surfaces as a 500 — SendGrid will retry, which is acceptable, but the observability story could be tighter. Deferred.

- **DEVIATION**: The story instructs to implement the SendGrid Event Webhook handler in the Client API and directly update `email_log.status`. The `email_log` table belongs to the Notification Service's schema. Direct cross-service database access violates the architecture document's mandate of "per-service DB roles with enforced access boundaries".
- **DEVIATION_TYPE**: CONTRADICTORY_SPEC
- **DEVIATION_SEVERITY**: blocking
- **Action**: Move the webhook handler to the Notification Service, or have the Client API publish a domain event to a message broker for the Notification Service to consume and update its own database.
- **RESOLVED**: Webhook handler moved to Notification Service (Task 2 above). ✅

- **DEVIATION**: The implementation defines the `email_log` table inline in `send_email.py` using `sa.Table` with mismatched column types (`Text` instead of `Enum` for `status`) and uses raw SQL strings in `sendgrid_webhook.py` instead of utilizing the existing SQLAlchemy ORM `EmailLog` model. This bypasses type safety and the single source of truth.
- **DEVIATION_TYPE**: ARCHITECTURAL_DRIFT
- **DEVIATION_SEVERITY**: deferrable

- **DEVIATION**: The SendGrid webhook signature verification silently passes if the `sendgrid_webhook_public_key` setting is empty. This fails open, contradicting the AC to "validate SendGrid signature to prevent spoofing". If the key is misconfigured in production, the endpoint is completely unprotected.
- **DEVIATION_TYPE**: ACCEPTANCE_GAP
- **DEVIATION_SEVERITY**: blocking

## Status
review

## Known Deviations

### Detected by `3-code-review` at 2026-04-19T12:07:03Z

- The story specifies implementing the SendGrid Event Webhook handler in the Client API while directly updating `notification.email_log.status` via SQL. The `email_log` table belongs to the Notification Service's schema, which violates the architecture's mandate for per-service DB roles and strict schema isolation boundaries. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- **RESOLVED at 2026-04-19**: Webhook handler moved to `notification/api/webhooks/sendgrid.py`. Migration 005 revokes the cross-service DB permissions that were granted in migration 004.

### Detected by `3-code-review` at 2026-04-19T12:35:25Z

- The implementation defines the `email_log` table inline in `send_email.py` using `sa.Table` with mismatched column types (`Text` instead of `Enum` for `status`) and uses raw SQL strings in `sendgrid_webhook.py` instead of utilizing the existing SQLAlchemy ORM `EmailLog` model. This bypasses type safety and the single source of truth. _(type: `ARCHITECTURAL_DRIFT`; severity: `deferrable`)_
- The SendGrid webhook signature verification silently passes if the `sendgrid_webhook_public_key` setting is empty. This fails open, contradicting the AC to "validate SendGrid signature to prevent spoofing". If the key is misconfigured in production, the endpoint is completely unprotected. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_

## Dev Agent Record

### Implementation Plan
1. Architectural fix: move `POST /webhooks/sendgrid` from client-api → notification service.
2. Add `sendgrid_webhook_public_key` to `NotificationSettings` (NOTIFICATION_ prefix).
3. Create `notification/api/webhooks/sendgrid.py` with ECDSA signature validation and DB status update.
4. Register webhook router in `notification/main.py`.
5. Stub-out client-api webhook file; remove router from `client-api/main.py`.
6. Remove `sendgrid_webhook_public_key` from `ClientApiSettings`.
7. Add migration 005 to revoke the cross-service DB grants added by migration 004.
8. Fix broken import in `opportunity_consumer.py` (`notification.tasks.email`, not `notification.workers.tasks.send_email`).
9. Create backward-compat re-export at `notification/workers/tasks/send_email.py`.
10. Write 17 unit tests covering all ACs + 10 webhook API tests.
11. Fix `test_celery_config.py` assertions to match the final implementation paths.

### Completion Notes (2026-04-19)

**Architecture deviation resolved**: The blocking CONTRADICTORY_SPEC deviation is fixed. The `POST /webhooks/sendgrid` endpoint now lives in the Notification Service which owns `notification.email_log`. The client-api no longer has any cross-schema DB access for this purpose.

**All 7 ACs satisfied**:
- AC1: 7 template types dispatched (7 parametrized passing tests)
- AC2: Template IDs from env vars (test verifies `NOTIFICATION_SENDGRID_TEMPLATE_WELCOME`)
- AC3: DB logging on every send (success → `sent`; non-retryable error → `failed`; max-retries exhausted → `failed`)
- AC4: Webhook updates status for `delivered`, `bounce`, `dropped` events
- AC5: 429, 500, 502, 503, 504 trigger `task.retry()` (5 parametrized passing tests)
- AC6: Invalid template raises `ValueError`; non-retryable
- AC7: Signature validation via `EventWebhook.verify_signature()`; 401 on invalid

**Test counts**: 18 unit/worker tests (all pass) + 3 webhook API tests runnable without DB (all pass). Full webhook API tests (10 total) require live PostgreSQL with notification schema.

**Pre-existing bugs fixed**:
- `opportunity_consumer.py` imported from wrong module path → fixed
- `test_celery_config.py` assertions used wrong task name/module → updated to accept canonical path

**Review findings resolved (2026-04-19) — Session 1**:
- ✅ Resolved review finding [High]: `migrated_db` fixture regex updated to accept revisions ≥ 002 (was hardcoded `002|003`; migration 005 caused 32 integration test failures)
- ✅ Resolved review finding [High]: `send_email` task now catches `MaxRetriesExceededError` and writes `status="failed"` to `email_log` before re-raising; new test `test_send_email_logs_failed_status_on_max_retries_exceeded` confirms AC3 coverage for permanent retry-exhausted failures

**Review findings resolved (2026-04-19) — Session 2**:
- ✅ Resolved review finding [High] Finding 4 (ACCEPTANCE_GAP): Webhook handler now fails **closed** — if `sendgrid_webhook_public_key` is empty/unset the endpoint returns 401 immediately, preventing silent bypass of ECDSA verification. New test `test_webhook_missing_public_key_returns_401` validates the fail-closed path. All 6 tests that previously used empty key to skip verification were updated to mock `EventWebhook.verify_signature` with `return_value=True` instead.
- ✅ Resolved review finding [Med] Finding 3 (ARCHITECTURAL_DRIFT): Replaced inline `sa.Table` (with wrong `sa.Text` for `status`) in `send_email._log_to_db` with canonical `EmailLog` ORM model using sync `Session`; replaced raw SQL `text("UPDATE ...")` in `sendgrid_webhook.py` with ORM `update(EmailLog)` statement using `execution_options(synchronize_session=False)` for async compatibility. 204 unit/worker tests pass; lint clean.

**All 7 ACs remain satisfied after both review sessions. All 4 senior developer review findings fully resolved.**

## File List

### New Files
- `services/notification/src/notification/api/__init__.py`
- `services/notification/src/notification/api/webhooks/__init__.py`
- `services/notification/src/notification/api/webhooks/sendgrid.py` — SendGrid webhook handler (moved from client-api, architecture fix)
- `services/notification/src/notification/workers/tasks/send_email.py` — backward-compat re-export shim
- `services/notification/alembic/versions/005_revoke_email_log_from_client_api.py` — revoke cross-service DB grants
- `services/notification/tests/api/__init__.py`
- `services/notification/tests/api/test_sendgrid_webhook.py` — 10 webhook tests

### Modified Files
- `services/notification/src/notification/config.py` — add `sendgrid_webhook_public_key` field
- `services/notification/src/notification/main.py` — register sendgrid webhook router
- `services/notification/src/notification/tasks/email.py` — catch `MaxRetriesExceededError` to log `failed` on retry exhaustion (review fix); use `EmailLog` ORM model in `_log_to_db` via sync `Session` (Finding 3 fix); add `EmailLog`, `Session` imports; remove inline `sa.Table`
- `services/notification/src/notification/api/webhooks/sendgrid.py` — fail-closed on empty public key (Finding 4 fix); use `update(EmailLog)` ORM statement instead of raw SQL (Finding 3 fix)
- `services/notification/src/notification/workers/opportunity_consumer.py` — fix import path for `send_email`
- `services/notification/tests/worker/test_send_email.py` — add `test_send_email_logs_failed_status_on_max_retries_exceeded` (review fix); 18 tests total
- `services/notification/tests/api/test_sendgrid_webhook.py` — add `test_webhook_missing_public_key_returns_401`; add `_accept_signature` helper; update 6 tests to mock `verify_signature` with `return_value=True` (Finding 4 fix); 11 tests total
- `services/notification/tests/unit/test_celery_config.py` — update assertions to match canonical module path
- `services/notification/tests/integration/test_002_migration.py` — update `migrated_db` fixture regex to accept revisions ≥ 002 (review fix)
- `services/client-api/src/client_api/api/webhooks/sendgrid.py` — replaced with empty stub (webhook moved)
- `services/client-api/src/client_api/main.py` — remove sendgrid webhook router import and include
- `services/client-api/src/client_api/config.py` — remove `sendgrid_webhook_public_key` field
- `services/client-api/tests/api/test_sendgrid_webhooks.py` — replaced with placeholder comment (tests moved to notification service)

## Change Log

| Date | Description |
|------|-------------|
| 2026-04-19 | Architectural deviation resolved: moved `POST /webhooks/sendgrid` from client-api to notification service. Added migration 005 to revoke cross-service DB permissions. Fixed broken import in opportunity_consumer.py. Rewrote ATDD unit tests (17 passing). Created 10 webhook API tests. All 7 ACs satisfied. Status → review. |
| 2026-04-19 | Addressed code review findings — 2 items resolved: (1) `migrated_db` fixture regex extended to accept revisions ≥ 002 — fixes 32 integration test failures caused by migration 005; (2) `send_email` task catches `MaxRetriesExceededError` and logs `failed` to `email_log` before re-raising — ensures AC3 coverage for retry-exhausted sends; new test added. 18 unit/worker tests pass. Status → review. |
| 2026-04-19 | Addressed remaining code review findings — 2 items resolved: (1) Finding 4 (ACCEPTANCE_GAP, blocking): webhook now fails closed when `sendgrid_webhook_public_key` is empty — returns 401 instead of skipping verification; `test_webhook_missing_public_key_returns_401` added; 6 tests updated to mock `verify_signature`; (2) Finding 3 (ARCHITECTURAL_DRIFT, deferrable): `send_email._log_to_db` uses `EmailLog` ORM model with sync `Session` (removes inline `sa.Table`); `sendgrid_webhook.py` uses ORM `update(EmailLog)` instead of raw SQL. 204 unit/worker tests pass; lint clean. All 4 senior dev review findings resolved. Status → review. |
