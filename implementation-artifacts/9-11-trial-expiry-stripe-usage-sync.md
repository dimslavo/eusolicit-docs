# Story 9.11: Trial Expiry & Stripe Usage Sync

Status: completed

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 09: Notifications, Alerts & Calendar

## Metadata
- **Story Key:** 9-11-trial-expiry-stripe-usage-sync
- **Points:** 3
- **Type:** backend
- **Module:** Notification Service (+ `eusolicit-models` event schema, + `client-api` publisher alignment)
- **Priority:** P1 (Beta milestone — trial reminder is a revenue-protection feature; Stripe sync hardening is a billing-accuracy safeguard)

## Story

As a **Professional-tier trial user and the EU Solicit billing system**,
I want **to receive a reminder email 3 days before my trial ends AND have my metered usage reliably reported to Stripe each day without double-billing or data loss**,
so that **I can convert to a paid plan before losing access, and the company's billing records stay accurate and reconciled with Stripe**.

## Description

This story delivers the final purely-backend consumer in the Notification Service plus a hardening pass on the daily Stripe usage-metering task.

**Part 1 — Trial Expiry Consumer (`SubscriptionConsumer`, new):**

- Listens to the `eu-solicit:subscriptions` Redis Stream (already registered in `notification-svc` consumer groups since Epic 1).
- On a `TrialExpiring` event (published by `client-api` when Stripe fires `customer.subscription.trial_will_end`, i.e. ~3 days before trial end), dispatches a `trial_expiry_reminder` email to every company admin of the affected company via the existing `send_email` Celery task (Story 9.6, template already configured as `sendgrid_template_trial_expiry_reminder` → `d-trial-expiry`).
- Follows the identical `OpportunityConsumer` / `TaskConsumer` / `ApprovalConsumer` skeleton (retry_pending → process_pending → consume) established in Stories 9.4 and 9.10.
- Uses Redis `SETNX` idempotency guard (same pattern as Story 9.10) to prevent duplicate dispatch on crash-before-ACK redelivery.

**Part 2 — Stripe usage-metering atomicity hardening (audit + extend Story 8.8's `sync_usage_to_stripe`):**

Story 8.8 already delivered the daily Stripe usage sync (`notification.workers.tasks.billing_usage_sync.sync_usage_to_stripe`, scheduled at 03:00 UTC via Celery Beat). This story does **not** re-implement that task; instead it closes the three remaining gaps that the Epic 09 test design calls out as **high-severity risk E09-R-004 (Stripe usage counter atomicity)**:

1. Pass an explicit Stripe `idempotency_key=f"{subscription_item_id}:{billing_period}"` to every `stripe.SubscriptionItem.create_usage_record(...)` call. Stripe will silently deduplicate on retry.
2. Guarantee the strict operation order: (a) read counter → (b) call Stripe with the idempotency key → (c) only on confirmed 200 response, archive the counter and `DEL` the Redis key; never delete before the Stripe call confirms.
3. Add integration tests (testcontainers Redis + mocked Stripe SDK) that cover the two crash scenarios: Stripe failure must leave the counter intact for the next run, and Stripe success must leave the counter cleared.

**Part 3 — Publisher alignment (`client-api`):**

The existing `_publish_trial_expiring` in `client-api` publishes to an ad-hoc stream key `"trial.expiring"` which is NOT a member of the canonical stream set in `eusolicit_common.events.bootstrap.STREAMS`, and NOT in `CONSUMER_GROUPS["notification-svc"]`. As written, the event would never reach the notification service's consumer group. This story realigns the publisher to use the canonical `eu-solicit:subscriptions` stream (already in both maps), and promotes `TrialExpiring` to a first-class discriminated event in `eusolicit-models.events` so both sides validate against the same schema.

All emails route through the existing `notification.tasks.email.send_email` Celery task on the `emails` queue. No new queues, no new migrations required (Redis SETNX for idempotency, consistent with Story 9.10).

## Acceptance Criteria

1. [x] **AC1 — `TrialExpiring` event schema.** `eusolicit-models.events` gains a new `TrialExpiring(BaseEvent)` class with `event_type: Literal["TrialExpiring"]`, `company_id: str`, `trial_end: str` (ISO-8601), `days_remaining: int` (default 3). `ServiceEvent` discriminated union is extended to include `TrialExpiring`. A round-trip unit test (`model_dump_json()` → `TypeAdapter(ServiceEvent).validate_json()`) returns a `TrialExpiring` instance.

2. [x] **AC2 — Client-API publisher aligned to canonical stream.** `client_api.services.webhook_service._publish_trial_expiring` is updated to publish with `stream="eu-solicit:subscriptions"` (NOT `"trial.expiring"`). Payload fields match the `TrialExpiring` event contract (`company_id`, `trial_end`, `days_remaining`). All existing webhook-service tests continue to pass; at least one new unit test asserts the publish is directed to `eu-solicit:subscriptions`.

3. [x] **AC3 — `SubscriptionConsumer` dispatch to all company admins.** When a `TrialExpiring` event is published to `eu-solicit:subscriptions`, the subscription consumer resolves all active `admin`-role members of the company via the `client.company_memberships` + `client.users` join (same read-only pattern as Story 9.10's team-lead resolver, but returning **every** admin, not just one), and dispatches one `send_email` call per admin with:
   - `template_type="trial_expiry_reminder"`
   - `recipient_email` = admin's email
   - `template_data` containing at minimum `days_remaining` (int), `trial_end` (ISO-8601), `company_id` (str), `tier_comparison_url` (computed from `settings.app_base_url` + `/en/settings/billing/pricing`), `upgrade_url` (computed from `settings.app_base_url` + `/en/settings/billing/upgrade`), and `admin_name` (nullable).

4. [x] **AC4 — Zero-admin graceful handling.** If the company has no active admin membership (possible only in corrupt data; should never happen in production), the consumer logs a `structlog.warning` with `company_id` and the event's `_message_id`, ACKs the event, and does NOT raise. No email is dispatched.

5. [x] **AC5 — Idempotent consumption (E09-R-002 carry-over).** `SubscriptionConsumer` uses the Redis consumer group `notification-svc` with manual ACK and invokes `EventConsumer.retry_pending()` → `process_pending()` → `consume()` in that order on every loop iteration — identical to `OpportunityConsumer.run()`. Re-delivery after a crash-before-ACK must not produce a duplicate dispatch. Use a Redis `SETNX` guard keyed on `notification:dispatched:subscription:{msg_id}:{admin_user_id}` with a 24-hour TTL, checked BEFORE dispatch. Partial-failure redelivery (e.g. 2 of 3 admins emailed before crash) must only redeliver the missing dispatches, not the already-sent ones.

6. [x] **AC6 — Graceful handling of unknown and malformed events.** Events whose `event_type` is not `TrialExpiring` are logged at `structlog.info` with `consumer`, `event_type`, `message_id` and ACKed (forward-compatibility). Events that fail Pydantic validation against the `ServiceEvent` discriminated union are logged at `structlog.error` with raw payload truncated to 256 chars and ACKed. Neither path must enter a redelivery loop. Parse MUST use `TypeAdapter(ServiceEvent).validate_python(...)` — do NOT hand-roll `json.loads + dict.get` (mandatory carry-over from Story 9.10 review finding B1).

7. [x] **AC7 — Async-safe Celery dispatch.** Every call to `send_email.delay(...)` from within `SubscriptionConsumer` async code is wrapped in `await asyncio.to_thread(send_email.delay, ...)`. Under no circumstances may `send_email.delay(...)` be called synchronously inside an `async def`. This is the explicit regression-guard lesson from Stories 9.4 and 9.10.

8. [x] **AC8 — Consumer lifecycle.** `notification/main.py::lifespan` starts `SubscriptionConsumer` as a background `asyncio.Task` alongside the three existing consumers, stores the handle on `app.state.subscription_consumer_task`, and cancels it on shutdown with the same `try/except CancelledError` pattern. A `TestClient` context entry test asserts `app.state.subscription_consumer_task` exists.

9. [x] **AC9 — Stripe usage sync idempotency key (E09-R-004, P0).** Every `stripe.SubscriptionItem.create_usage_record(...)` call in `notification.workers.tasks.billing_usage_sync` includes an explicit `idempotency_key=f"{item_id}:{period}"` keyword argument. A unit test asserts that (a) the idempotency_key is passed, (b) it matches the `{item_id}:{period}` pattern, and (c) two invocations with the same item_id + period produce two Stripe calls with the same idempotency_key (relying on Stripe's server-side deduplication).

10. [x] **AC10 — Stripe usage sync crash-safety (E09-R-004, P0).** The task must enforce the invariant: *a Redis usage counter key is DELETEed only after the Stripe API call for that counter's period has returned successfully.* When Stripe raises any `stripe.error.StripeError` (including `RateLimitError`, `InvalidRequestError`, `APIConnectionError`) or times out, the Redis counter key is NOT deleted and NOT archived, and the next scheduled run retries the report for the same period with the same idempotency_key. Integration tests assert both paths:
    - `test_stripe_counter_intact_on_failure`: mock Stripe raises; assert Redis usage key still exists after task returns.
    - `test_stripe_counter_cleared_on_success`: mock Stripe returns 200; assert Redis usage key is deleted after task returns.

11. [x] **AC11 — Retry behavior parity.** Stripe `RateLimitError` / `APIConnectionError` trigger a Celery task retry via the existing `max_retries=2, default_retry_delay=300` configuration on the `sync_usage_to_stripe` task (already set by Story 8.8 — verify not regressed). `InvalidRequestError` (permanent; bad subscription ID) is logged at `error` and skipped for that company without retry. A unit test covers each branch.

12. [x] **AC12 — Bootstrap / consumer-group sanity check.** `eusolicit_common.events.bootstrap.STREAMS["subscriptions"]` == `"eu-solicit:subscriptions"` AND `"eu-solicit:subscriptions" in CONSUMER_GROUPS["notification-svc"]` (both already true — verify with a test so future refactors can't silently regress). No changes to `bootstrap.py` or `redis_utils.py` expected; test fails loudly if anyone removes the mapping.

13. [x] **AC13 — Security: no plaintext secrets in logs.** `NOTIFICATION_STRIPE_SECRET_KEY` is never logged in plaintext at any log level. A structlog-capture unit test runs the full `sync_usage_to_stripe` dispatch flow (mocked Stripe) and asserts the secret value does not appear in any captured log record. Mirrors the AC12 lesson from Story 9.10.

14. [x] **AC14 — Observability.** The consumer emits structured log events on every dispatch decision: `subscription_consumer.trial_expiring_dispatched` (with `company_id`, `admin_count`, `correlation_id`), `subscription_consumer.no_admins` (AC4), `subscription_consumer.unknown_event_type` (AC6), `subscription_consumer.malformed_payload` (AC6). The billing usage sync already logs `billing_sync_usage_reported` and `billing_sync_complete` — verify no regression; add one new log `billing_sync_idempotency_key_set` that echoes the key value (non-sensitive: it's `"{item_id}:{period}"`).

## Design Constraints

- **No new Celery tasks.** `SubscriptionConsumer` dispatches to the existing `send_email` task (Story 9.6). The daily Stripe sync remains the existing `sync_usage_to_stripe` task (Story 8.8); only hardening changes are in scope.
- **No new DB migrations.** Idempotency uses Redis SETNX (consistent with Story 9.10). If a developer proposes a DB-backed guard instead, it MUST ship with a reversible Alembic migration under `services/notification/alembic/versions/` — but this is discouraged because it adds a table with no long-term analytic value.
- **Do NOT restructure `billing_usage_sync.py`.** This is a hardening pass, not a rewrite. The SCAN-based key discovery, per-company grouping, archive-hash TTL, and audit-event emission from Story 8.8 must remain intact. Only the Stripe call-site and its error-handling / counter-delete ordering change.
- **Do NOT change the Beat schedule time.** `sync-usage-to-stripe-daily` runs at 03:00 UTC (Story 8.8's deliberate choice, placed AFTER the 02:00 analytics MV refreshes to avoid DB contention). Epic 09's AC wording of "02:00 UTC" is superseded by Story 8.8's production-validated schedule; document this variance in the Change Log (not a defect).
- **Multi-admin dispatch for AC3.** Unlike Story 9.10's team-lead CC (single recipient), trial expiry fans out to ALL active admins so the company-wide billing decision maker sees the reminder. Rationale: trial expiry is a company-level billing event, not a task-level notification; missing one admin could cost a paying customer.
- **Signed deep-link URLs are NOT in scope.** The upgrade CTA URL is a plain unsigned URL (`{app_base_url}/en/settings/billing/upgrade`); authentication happens at the Client API via the user's existing session. Story 9.10's HMAC signing is for one-click approve/reject, which has no analogue here.
- **Schema discovery.** `eusolicit-models.events` currently has `SubscriptionChanged` and `BillingEvent` — but NOT `TrialExpiring`. The current `_publish_trial_expiring` publishes with `event_type="TrialExpiring"` as a raw string, which the consumer cannot validate against the discriminated union. AC1 closes this gap.
- **Stream-name realignment is a light-touch fix.** `_publish_trial_expiring` in `client-api` currently writes to `stream="trial.expiring"` (ad-hoc, not in bootstrap). AC2 realigns it to `stream="eu-solicit:subscriptions"`. Because Story 8 is `done` but the event publisher lives under `webhook_service.py` (modified in Story 8.5), this is a legitimate Epic 09 fix, not a regression. Backfill of previously-published events is NOT required (the ad-hoc stream was never consumed, so no historical data is lost).
- **Tier-comparison URL** uses `settings.app_base_url` (already in `config.py` since Story 9.10) concatenated with the pricing page path. Do NOT hardcode `https://app.eusolicit.com`.

## Tasks / Subtasks

- [x] **Task 1: Extend `eusolicit-models.events` with `TrialExpiring` event. (AC1)**
  - [x] Add `TrialExpiring(BaseEvent)` class to `packages/eusolicit-models/src/eusolicit_models/events.py` with fields `event_type: Literal["TrialExpiring"]`, `company_id: str`, `trial_end: str`, `days_remaining: int = 3`.
  - [x] Extend the `ServiceEvent` discriminated union to include `TrialExpiring`.
  - [x] Add unit tests in `packages/eusolicit-models/tests/test_events_trial_expiring.py`: discriminator round-trip; default `days_remaining=3`; reject `event_type="TrialExpiring"` payload missing required fields.
  - [x] Bump `pyproject.toml` version of `eusolicit-models` to the next patch (follow the convention established in Story 9.10).
- [x] **Task 2: Realign `client-api` trial-expiring publisher to canonical stream. (AC2)**
  - [x] Edit `services/client-api/src/client_api/services/webhook_service.py::_publish_trial_expiring`: change `stream="trial.expiring"` → `stream="eu-solicit:subscriptions"`.
  - [x] Confirm the payload already includes `company_id`, `trial_end`, `days_remaining` (it does per current code); no new fields.
  - [x] Add / update a unit test in `services/client-api/tests/unit/test_webhook_service.py` that asserts `publisher.publish` is called with `stream="eu-solicit:subscriptions"` when `customer.subscription.trial_will_end` is processed.
  - [x] Run `make test-service SVC=client-api` to confirm no webhook regression tests break.
- [x] **Task 3: Add `SubscriptionConsumer`. (AC3–AC8, AC14)**
  - [x] Create `services/notification/src/notification/workers/subscription_consumer.py`. Class `SubscriptionConsumer` with constructor that instantiates `EventConsumer(get_redis_client())` plus a session factory, following the `TaskConsumer` / `ApprovalConsumer` shape exactly.
  - [x] Constants `_STREAM = "eu-solicit:subscriptions"`, `_GROUP = "notification-svc"`, `_CONSUMER = socket.gethostname()`, `_BLOCK_MS = 5000`, `_COUNT = 10`, `_IDEMPOTENCY_TTL = 86400`.
  - [x] Module-level `_service_event_adapter = TypeAdapter(ServiceEvent)` (same import pattern as `approval_consumer.py` B1 fix).
  - [x] `_is_idempotent(msg_id, admin_user_id)` using `SETNX notification:dispatched:subscription:{msg_id}:{admin_user_id} EX 86400` (mirrors Story 9.10's per-approver idempotency for partial-failure safety).
  - [x] `_resolve_company_admins(company_id) -> list[User]` joins `client.company_memberships` + `client.users` where `role == CompanyRole.admin`, `accepted_at IS NOT NULL`, `User.is_active == True`. Returns all matching admins; does NOT limit to one. Uses the same read-only session pattern as `_resolve_team_lead` in `task_consumer.py` — reuse `notification.models.user.User` and `notification.models.company_membership.CompanyMembership` from Story 9.10 (no new ORM needed).
  - [x] `async def process_event(self, event)`:
    - Normalize `_message_id` from `bytes` → `str` at the top (N6 carry-over from Story 9.10).
    - Parse with `TypeAdapter(ServiceEvent).validate_python(merged_envelope_and_payload)` (B1 carry-over). On `PydanticValidationError` → log `error` with truncated raw payload + ACK.
    - Dispatch by `event_type`:
      - `TrialExpiring` → `_handle_trial_expiring`
      - any other → log `info` + ACK.
  - [x] `_handle_trial_expiring`:
    1. Resolve admins via `_resolve_company_admins(company_id)`.
    2. If `len(admins) == 0` → log `warning subscription_consumer.no_admins` + ACK + return (AC4).
    3. For each admin: check `_is_idempotent(msg_id, admin.id)` — if already dispatched, skip; else build `template_data` dict and call `await asyncio.to_thread(send_email.delay, admin.email, "trial_expiry_reminder", template_data)`.
    4. Log `info subscription_consumer.trial_expiring_dispatched` with `company_id`, `admin_count`, `correlation_id`.
    5. ACK the event (single ACK after all admin fan-out completes; per-admin idempotency guards duplicate replay).
  - [x] `async def run(self)`: `xgroup_create` (tolerate BUSYGROUP) → outer `while True` loop with `retry_pending → process_pending → consume → process_event`, copied verbatim from `task_consumer.run` (30s min_idle_time in process_pending is non-negotiable per Story 9.4 lesson).
  - [x] Entry point `async def run_subscription_consumer(app=None)` — instantiates and runs; to be wired into `main.lifespan` (Task 4).
  - [x] Unit tests in `services/notification/tests/worker/test_subscription_consumer.py`: happy-path multi-admin fan-out, zero admins, unknown event_type, malformed payload, Pydantic schema mismatch, SETNX replay (per-admin partial replay), cross-tenant negative (event for Company A does not email Company B admin — mandatory carry-over from Story 9.10 N2).
- [x] **Task 4: Wire `SubscriptionConsumer` into `notification/main.py::lifespan`. (AC8)**
  - [x] Import `run_subscription_consumer`; create a fourth `asyncio.Task` alongside the three existing consumers; store on `app.state.subscription_consumer_task`.
  - [x] Extend the shutdown loop to include `("subscription_consumer", subscription_task)`.
  - [x] Add a smoke test in `services/notification/tests/test_lifespan_consumers.py` asserting `app.state.subscription_consumer_task` exists after `TestClient` context entry. Un-skip any previously-skipped entries that block this (per Story 9.10 B2 lesson — ATDD must be green).
- [x] **Task 5: Harden Stripe usage-sync atomicity. (AC9, AC10, AC11, AC14)**
  - [x] Edit `services/notification/src/notification/workers/tasks/billing_usage_sync.py` — locate the existing `stripe.SubscriptionItem.create_usage_record(...)` call inside the `for item in metered_items` loop.
  - [x] Add keyword argument `idempotency_key=f"{item['id']}:{period}"`. Log the key at `info` as `billing_sync_idempotency_key_set` (safe — not a secret).
  - [x] Audit the operation order so that `r.hset(archive_key, ...)` and the `r.delete(counter_key)` loop are REACHED ONLY when all `create_usage_record` calls for that `(company_id, period)` tuple succeeded. Today, the try/except on line 314 catches `stripe.error.StripeError` then `continue`s — verify this leaves the archive + delete path unreached (it does, but make the invariant explicit with a `stripe_success = True` sentinel initialized before the metered-items loop and set False on any exception; skip archive+delete when `stripe_success is False`).
  - [x] Verify `max_retries=2, default_retry_delay=300, acks_late=True` are not regressed on the `@celery.task` decorator.
  - [x] Add keyword handling for `stripe.error.InvalidRequestError` — log at `error` and `continue` (skip for that company, do NOT retry); differentiate from `RateLimitError` / `APIConnectionError` which currently fall through to the generic `StripeError` branch — make the retry behavior explicit by re-raising `RateLimitError` and `APIConnectionError` so Celery's retry kicks in, and catching `InvalidRequestError` / generic `StripeError` as terminal-skip.
  - [x] Unit tests in `services/notification/tests/worker/test_billing_usage_sync_atomicity.py`:
    - [x] `test_idempotency_key_passed` — assert kwarg shape `{item_id}:{period}`.
    - [x] `test_counter_intact_on_stripe_failure` — mock Stripe raises `InvalidRequestError`; run task; assert counter key still exists in mock Redis.
    - [x] `test_counter_cleared_on_stripe_success` — mock Stripe returns; assert counter key deleted.
    - [x] `test_rate_limit_triggers_retry` — mock Stripe raises `RateLimitError`; assert `self.retry()` path is reached (or the exception propagates so Celery retries).
    - [x] `test_invalid_request_skips_without_retry` — mock Stripe raises `InvalidRequestError`; assert no retry scheduled (caught, logged, company skipped).
    - [x] `test_stripe_secret_never_in_logs` (AC13) — structlog capture; assert `settings.stripe_secret_key` value does not appear in any log record.
- [x] **Task 6: Integration tests for `SubscriptionConsumer` + billing-sync atomicity.**
  - [x] `services/notification/tests/integration/test_subscription_consumer_integration.py`:
    - [x] Spin up Redis via testcontainers; run `bootstrap_event_bus` so the `notification-svc` group exists on `eu-solicit:subscriptions`.
    - [x] Seed `client.users` + `client.company_memberships` with 3 admins for one company.
    - [x] `xadd` a `TrialExpiring` envelope to `eu-solicit:subscriptions`; start `SubscriptionConsumer` in a background task for 2 seconds then cancel.
    - [x] Assert `send_email.delay` was called 3 times (mock the Celery task), each with `template_type="trial_expiry_reminder"` and the correct admin email.
    - [x] Second test: simulate crash-before-ACK by pre-setting SETNX for admin 2; replay; assert only 2 of 3 admins receive email (AC5).
  - [x] `services/notification/tests/integration/test_billing_usage_sync_atomicity.py`:
    - [x] Real Redis via testcontainers; seed `usage:{company_id}:2026-04:{metric}` keys + `billing:{company_id}:meta` hash.
    - [x] Mock Stripe SDK; run `sync_usage_to_stripe()`:
      - [x] Success path: assert archive hash populated + counter keys deleted + audit stream entry written.
      - [x] Failure path: mock `create_usage_record` raises `stripe.error.StripeError` on the first metered item; assert counter keys NOT deleted AND archive hash NOT written (full rollback of the archive+delete pair).
  - [x] Register all new integration tests under `@pytest.mark.integration`.
- [x] **Task 7: `TrialBanner` / upgrade URL coverage (frontend sanity check — optional, non-blocking).**
  - [x] Grep `frontend/apps/client` for `/settings/billing/upgrade` and `/settings/billing/pricing` — confirm those routes exist (they do per Story 8.12 / 8.13 grep results). If missing, raise as open question but do NOT block this story.
  - [x] No frontend code changes required. This task exists only to ensure the `template_data` URLs resolve when users click the email CTA.
- [x] **Task 8: Documentation + observability polish.**
  - [x] Update `services/notification/README.md` (or equivalent) to list the four consumer loops (`opportunity`, `task`, `approval`, `subscription`) with their streams, groups, and owner stories — extend the Story 9.10 README addition.
  - [x] Add an entry in the service's Change Log noting:
    1. The `_publish_trial_expiring` stream-name correction from `"trial.expiring"` → `"eu-solicit:subscriptions"`.
    2. The deliberate 03:00 UTC Beat schedule (contradicts epic AC wording of "02:00 UTC" for operational reasons — placed AFTER analytics MV refreshes).
  - [x] Emit one structured log on billing-sync task entry confirming the Stripe idempotency-key hardening is active (`billing_sync_idempotency_hardened_v2`) — useful forensic marker for post-deploy log search.

## Dev Notes

### Test Expectations (from Epic 09 Test Design)

Source: `eusolicit-docs/test-artifacts/test-design-epic-09.md` (rows keyed to S09.11 and the high-priority risk E09-R-004 that this story explicitly mitigates).

| Priority | Requirement | Level | Count | Risk Link | Test Harness |
|---|---|---|---|---|---|
| **P0** | Stripe counter atomicity — counter cleared only after confirmed Stripe success; crash before DEL leaves counter intact for retry | Integration | 4 | E09-R-004 | Mock `stripe.SubscriptionItem.create_usage_record`; inject success/failure; verify Redis state |
| **P1** | `subscription.trial_expiring` event → sends reminder email with `days_remaining`, tier comparison, upgrade CTA | Integration | 3 | E09-R-010 | Mock stream event; assert `send_email` with `trial_expiry_reminder` template + `days_remaining` in `template_data` |
| **P2** | Stripe 429 → task retry; `InvalidRequestError` (invalid subscription) → skip without retry | Unit | 3 | — | Mock Stripe SDK exceptions; assert retry on 429; assert no retry on `InvalidRequestError` |
| **P2** | Redis counter value after confirmed Stripe success is cleared | Integration | 2 | E09-R-004 | Complement of P0-004; mock Stripe success; assert Redis key deleted |

**Total Story 9.11 contribution:** ~12 tests directly against S09.11 ACs + 4 P0 E09-R-004 scenario tests (counted above, not duplicated) + the shared consumer-idempotency P0 tests already passing from Story 9.4 (reused infra).

**Explicit high-risk links this story closes:**

- **E09-R-004 (Stripe usage counter atomicity, score 6, HIGH):** Story 8.8 established the daily sync; this story adds the explicit `idempotency_key` and the strict archive+delete-only-after-success ordering that the Epic 09 test design mandates. The non-negotiable exit criterion is `test_stripe_counter_intact_on_failure` AND `test_stripe_counter_cleared_on_success` both passing.
- **E09-R-010 (Trial expiry reminder event coupling, score 4, MEDIUM):** This story consumes the event on the notification side; the publisher fix in Task 2 closes the silent-coupling defect where `_publish_trial_expiring` wrote to an unreferenced stream key.
- **E09-R-002 (Consumer duplicate dispatch, score 6, HIGH — carry-over):** The Redis SETNX idempotency guard in `SubscriptionConsumer` mirrors the Story 9.10 pattern. The same Story 9.10 trade-off (data-loss corner if `send_email.delay` raises after SETNX succeeds) applies here; operational mitigation is DLQ monitoring via `process_pending`. Document in the consumer's docstring (N5 carry-over).

### Relevant Architecture Patterns & Constraints

1. **Consumer loop skeleton (copy verbatim from `task_consumer.py` / `approval_consumer.py`):** `retry_pending → process_pending (min_idle_time=30_000) → consume → process_event`. The 30s `min_idle_time` is non-negotiable per Story 9.4 lesson. `asyncio.CancelledError` breaks the outer loop cleanly; generic exceptions are logged and retried after `asyncio.sleep(5)`.
2. **`asyncio.to_thread(send_email.delay, ...)` is mandatory** (Story 9.4 review finding #2). Never call `.delay()` directly in async code — it's a synchronous broker network call that blocks the event loop.
3. **Event envelope parsing via Pydantic discriminator.** Use `TypeAdapter(ServiceEvent).validate_python(...)` — never hand-roll `json.loads + dict.get` (Story 9.10 B1 fix). Catch `pydantic.ValidationError` as the AC6 malformed branch.
4. **structlog, not stdlib logging.** Every consumer and billing-sync log line uses `structlog.get_logger()` and includes `correlation_id` (when available from the envelope) and `message_id`.
5. **Audit trail (billing sync only).** The existing Story 8.8 `_emit_audit_event` emits `billing.usage_synced` to the `audit_events` stream for each synced `(company_id, period)`. Preserve this call-site unchanged. Trial-expiry consumer dispatch is NOT a mutation of a sensitive entity and does NOT emit to `shared.audit_log` — consistent with Story 9.10's decision per project-context rule #44.
6. **Cross-schema read-only access.** `notification.models.user.User` and `notification.models.company_membership.CompanyMembership` (both from Story 9.10) are reused. The existing migration `006_grant_notification_select_on_users.py` already grants `notification_role` SELECT on `client.users`. Verify analogous grant exists for `client.company_memberships`; it was added in Story 9.10's migration 006 — confirm before deploy.
7. **i18n.** Email template rendering is out of scope (SendGrid dynamic template owns content). The consumer only populates `template_data`. The `tier_comparison_url` and `upgrade_url` include `/en/` path prefix for the default locale; the SendGrid template should render locale-aware copy based on an additional `locale` field (future enhancement — not blocking for Beta).
8. **Stream-name convention.** All streams follow the `eu-solicit:<domain>` prefix. The pre-existing `"trial.expiring"` ad-hoc key was a Story 8.5 oversight; Task 2 realigns it. No other consumers subscribed to `"trial.expiring"` (grep `CONSUMER_GROUPS` — it's not referenced), so the realignment cannot break any existing subscriber.
9. **Stripe idempotency-key semantics.** Per Stripe API docs, `idempotency_key` is accepted as a keyword argument on any mutating API call including `SubscriptionItem.create_usage_record(...)`. Stripe retains idempotency keys for 24 hours — long enough to cover a Celery `default_retry_delay=300` retry cycle plus any operator-triggered manual re-run within the same day. The `{item_id}:{period}` shape guarantees uniqueness per (subscription-item, billing-period) tuple, which is exactly the idempotent unit of work.
10. **No Celery workers beyond scheduled Beat + already-defined queues.** Project rule per `CLAUDE.md`: "No Celery workers — async tasks use FastAPI BackgroundTasks or Celery Beat for scheduling only. Redis 7 Streams for event bus." The existing `send_email` and `sync_usage_to_stripe` tasks are accepted deviations; this story does not extend the pattern.

### Source Tree Components to Touch

**New files:**
- `eusolicit-app/services/notification/src/notification/workers/subscription_consumer.py`
- `eusolicit-app/services/notification/tests/worker/test_subscription_consumer.py`
- `eusolicit-app/services/notification/tests/worker/test_billing_usage_sync_atomicity.py`
- `eusolicit-app/services/notification/tests/integration/test_subscription_consumer_integration.py`
- `eusolicit-app/services/notification/tests/integration/test_billing_usage_sync_atomicity.py`
- `eusolicit-app/packages/eusolicit-models/tests/test_events_trial_expiring.py`
- `eusolicit-app/services/notification/README.md`

**Modified files:**
- `eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py` — add `TrialExpiring`; extend `ServiceEvent`.
- `eusolicit-app/packages/eusolicit-models/pyproject.toml` — bump patch version.
- `eusolicit-app/services/client-api/src/client_api/services/webhook_service.py` — realign `_publish_trial_expiring` stream to `eu-solicit:subscriptions`.
- `eusolicit-app/services/client-api/tests/unit/test_webhook_service.py` — assert new stream name.
- `eusolicit-app/services/notification/src/notification/main.py` — start 4th background task in `lifespan`.
- `eusolicit-app/services/notification/src/notification/workers/tasks/billing_usage_sync.py` — add `idempotency_key`, enforce archive+delete-only-after-success ordering, refine exception branches.
- `eusolicit-app/services/notification/tests/test_lifespan_consumers.py` — add `subscription_consumer_task` assertion.

**Files intentionally NOT modified:**
- `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py` — `"eu-solicit:subscriptions"` is ALREADY in `STREAMS` and `CONSUMER_GROUPS["notification-svc"]` (verified in discovery). AC12 is a "don't regress" guard, not a diff.
- `eusolicit-app/services/notification/src/notification/models/user.py` / `company_membership.py` — reused as-is from Story 9.10.
- `eusolicit-app/services/notification/src/notification/config.py` — reuses `app_base_url` (Story 9.10) and `stripe_secret_key` (Story 8.8). No new settings.
- `eusolicit-app/services/notification/src/notification/workers/beat_schedule.py` — `sync-usage-to-stripe-daily` schedule (03:00 UTC) remains.

### Testing Standards Summary

- **pytest markers.** Unit tests under `services/notification/tests/worker/` carry no marker. Integration tests under `services/notification/tests/integration/` use `@pytest.mark.integration`.
- **Fixtures.** Reuse session-scoped fixtures from `services/notification/tests/conftest.py`: `redis_container`, `notification_db_session`, and the `mock_send_email` autouse fixture introduced in Story 9.10 (extend it to patch `notification.workers.subscription_consumer.send_email` as well).
- **Coverage target.** ≥80% on `subscription_consumer.py` and `billing_usage_sync.py` (celery task + consumer modules per epic quality gate). 100% branch coverage on the Stripe idempotency and archive-delete-ordering branches per E09-R-004 non-negotiable.
- **ruff + mypy.** `make lint` and `make type-check` must pass on all new / modified files before marking the story done. Per Story 9.9 / 9.10 precedent.
- **`asyncio` testing.** Use `pytest-asyncio` with `asyncio_mode="auto"` (already configured).
- **Testcontainers for Redis + Postgres** (integration tier). The `fakeredis` path is acceptable for the SETNX unit tests but NOT for the AC10 crash-safety integration tests — real Redis is required because counter-key visibility across the archive+delete path is non-trivial to simulate accurately with the fake client.

### Previous Story Intelligence (Stories 9.4, 9.10, and 8.8)

**From Story 9.4 (alert matching + immediate dispatch):**
- E09-R-002 idempotency lesson: dispatch order is `SETNX` → `send_email.delay()` → `ACK`. Never reorder. If `send_email.delay()` raises after SETNX succeeds, the message stays in the PEL and the outer loop's retry_pending path picks it up — the SETNX key prevents duplicate dispatch, but a persistent broker outage falls back to DLQ monitoring.
- `min_idle_time=30_000` on `XCLAIM` in `process_pending` is non-negotiable.
- Every `send_email.delay` wrapped in `asyncio.to_thread`.

**From Story 9.10 (task + approval consumers):**
- Per-target idempotency key pattern (Story 9.10 fan-out used `approval-req:{msg_id}:{approver_id}`). Story 9.11 uses the same shape: `subscription:{msg_id}:{admin_user_id}`. Half-sent replay is safe.
- Cross-tenant negative tests are MANDATORY (Story 9.9 / 9.10 carry-over). Add at least one test asserting a `TrialExpiring` event for Company A does NOT cause an email to any Company B admin.
- `TypeAdapter(ServiceEvent).validate_python(...)` — do NOT hand-roll JSON parsing. Merge envelope fields with the `payload` JSON before calling `validate_python` (see `approval_consumer.py` for the exact pattern).
- Normalize `_message_id` from `bytes` → `str` at the top of `process_event` (N6).
- Structured logging on every dispatch decision, including `correlation_id` from the envelope.

**From Story 8.8 (usage metering + Stripe sync):**
- `sync_usage_to_stripe` task is the source of truth for Stripe usage reporting. It uses SCAN (not KEYS) for key discovery. It stores a 90-day archive hash before deleting counters. It emits `billing.usage_synced` audit events to the shared `audit_events` Redis Stream. DO NOT replace or rewrite — ONLY add the idempotency key and refine the archive-delete ordering invariant.
- The `@celery.task` decorator already has `acks_late=True`, `max_retries=2`, `default_retry_delay=300`. Verify these are preserved.
- The `stripe-python` dependency is pinned to `<9` because v9+ removed `SubscriptionItem.create_usage_record`. A compat-shim exists (lines 37–65 of `billing_usage_sync.py`) — leave it untouched.

### Git Intelligence — Recent Relevant Commits

`git log --oneline -10` (last 10 commits that touch the notification stack):

Because Stories 9.4 / 9.6 / 9.8 / 9.9 / 9.10 established the consumer skeleton + SendGrid dispatch + Fernet encryption + HMAC signing + Pydantic discriminator parsing, implementation for 9.11 should be **strictly additive** — no refactors of `opportunity_consumer.py`, `task_consumer.py`, `approval_consumer.py`, `email.py`, or the alembic migration sequence. Any incidental refactor must be extracted into a separate `9-11-chore-*` PR and noted under Known Deviations as `SCOPE_CREEP`. The `billing_usage_sync.py` hardening pass is the one exception — it is an explicit Story 9.11 deliverable, not a refactor.

### Project Structure Notes

- Alignment with unified project structure: the new consumer lives under `services/notification/src/notification/workers/subscription_consumer.py` alongside `opportunity_consumer.py`, `task_consumer.py`, `approval_consumer.py`. No new top-level directories. Stream naming follows the established `eu-solicit:<domain>` prefix.
- Detected variances and rationale:
  - **Schedule drift (03:00 UTC vs epic AC "02:00 UTC"):** Story 8.8 placed the sync at 03:00 to run AFTER the 02:00 analytics MV refresh window. Epic 09 AC wording is superseded. Documented in Task 8.
  - **Stream-name realignment (`trial.expiring` → `eu-solicit:subscriptions`):** Fixes a silent bug where the event was never reaching the notification service. Not a regression — the old stream had no consumers. Documented in Task 8 and in Known Deviations.
- No conflicts detected with Stories 9.1–9.10 conventions.

### References

- Epic: [Source: eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md#S09.11]
- Test Design: [Source: eusolicit-docs/test-artifacts/test-design-epic-09.md#S09.11] (P1 trial expiry + P0/P2 Stripe atomicity rows)
- Risk E09-R-004 (Stripe usage counter atomicity): [Source: eusolicit-docs/test-artifacts/test-design-epic-09.md#High-Priority Risks]
- Risk E09-R-010 (Trial expiry reminder event coupling): [Source: eusolicit-docs/test-artifacts/test-design-epic-09.md#Medium-Priority Risks]
- Risk E09-R-002 (Consumer duplicate dispatch, inherited): [Source: eusolicit-docs/test-artifacts/test-design-epic-09.md#High-Priority Risks]
- Previous story 9.4 (consumer skeleton): [Source: eusolicit-docs/implementation-artifacts/9-4-alert-matching-immediate-dispatch.md]
- Previous story 9.10 (consumer parity — task/approval): [Source: eusolicit-docs/implementation-artifacts/9-10-task-approval-notification-consumers.md]
- Previous story 8.8 (existing Stripe usage sync): [Source: eusolicit-docs/implementation-artifacts/8-8-usage-metering-with-redis-counters-stripe-sync.md]
- Previous story 8.5 (trial expiry handling — publisher origin): [Source: eusolicit-docs/implementation-artifacts/8-5-trial-expiry-handling-downgrade-logic.md]
- Project conventions: [Source: CLAUDE.md#Critical Conventions] — "No Celery workers", "Redis 7 Streams for event bus", "structlog everywhere", "Pydantic v2 models in eusolicit-models".
- AI agent rules: [Source: eusolicit-docs/project-context.md#Redis Streams] — EventPublisher/EventConsumer envelope format; at-least-once delivery; DLQ via `process_pending`.
- Existing event schemas: [Source: eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py]
- Existing bootstrap: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/bootstrap.py] — `"eu-solicit:subscriptions"` already in `STREAMS` and `CONSUMER_GROUPS["notification-svc"]`.
- Existing publisher code to update: [Source: eusolicit-app/services/client-api/src/client_api/services/webhook_service.py#L466–507] (`_publish_trial_expiring`).
- Existing consumers to mirror: [Source: eusolicit-app/services/notification/src/notification/workers/task_consumer.py], [Source: eusolicit-app/services/notification/src/notification/workers/approval_consumer.py].
- Existing `send_email` task: [Source: eusolicit-app/services/notification/src/notification/tasks/email.py]
- Existing template env var: [Source: eusolicit-app/services/notification/src/notification/config.py#L70] (`sendgrid_template_trial_expiry_reminder = "d-trial-expiry"`).
- Existing billing-sync task (to harden): [Source: eusolicit-app/services/notification/src/notification/workers/tasks/billing_usage_sync.py]
- Stripe idempotency-key docs: https://docs.stripe.com/api/idempotent_requests (verified 2026-04-20).

### Open Questions for PM / TEA (for follow-up, not blocking)

1. **Multi-admin email volume.** AC3 fans out one email per company admin. For companies with 5–10 admins, this produces 5–10 emails per trial-expiry event. PM confirm acceptable inbox noise vs. single-email-to-primary-contact alternative? Current decision: every admin receives the email to maximize conversion chance.
2. **Beat schedule alignment.** Epic AC says 02:00 UTC; Story 8.8 deployed 03:00 UTC. Leaving 03:00 UTC unchanged per Task 8 — confirm with PM that the epic AC wording is informational (not a SLA commitment). If PM insists on 02:00 UTC, note this requires reordering the analytics MV refresh chain.
3. **Locale-aware email templates.** `template_data` currently hardcodes `/en/` in the URLs. For BG-locale trial users, the upgrade link should route to `/bg/settings/billing/upgrade`. Pending E07 / next-intl integration decision — captured as a follow-up story, not blocking 9.11 Beta.
4. **Admin-role query performance.** For large companies (Enterprise tier with 50+ admins), the `_resolve_company_admins` join runs per-event. Not a bottleneck now (trial expiry is ~few-per-day), but profile before Enterprise GA.
5. **`TrialExpiring` consumed anywhere else?** Notification Service is the only consumer today. If client-api or admin-api later want to consume the same event, `notification-svc` consumer group naming must be generalized — out of scope for this story.

## Dev Agent Record

### Agent Model Used

gemini-2.0-flash-thinking-exp-01-21 (autopilot)

### Debug Log References

- [Tests Passed] all 8 unit tests in `eusolicit-models` passed.
- [Tests Passed] all 4 unit tests in `client-api` for stream alignment passed.
- [Tests Passed] all 18 unit tests in `notification` for `SubscriptionConsumer` passed.
- [Tests Passed] all 6 lifespan tests in `notification` passed.
- [Tests Passed] all 12 unit tests in `notification` for billing sync atomicity passed.
- [Tests Passed] all 4 integration tests in `notification` passed.

### Completion Notes List

- Verified `TrialExpiring` event schema and `ServiceEvent` union in `eusolicit-models`.
- Bumped `eusolicit-models` version to `0.1.3`.
- Verified `_publish_trial_expiring` stream realignment in `client-api` from `"trial.expiring"` to `"eu-solicit:subscriptions"`.
- Verified `SubscriptionConsumer` implementation with SETNX idempotency and multi-admin dispatch.
- Verified `SubscriptionConsumer` wiring in `notification/main.py` lifespan.
- Verified Stripe usage sync hardening with `idempotency_key` and atomic Redis counter deletion.
- Created `services/notification/README.md` with consumer loop registry.
- Updated Story status to `completed` and added Change Log.

### File List

- `eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py`
- `eusolicit-app/packages/eusolicit-models/pyproject.toml`
- `eusolicit-app/services/client-api/src/client_api/services/webhook_service.py`
- `eusolicit-app/services/notification/src/notification/workers/subscription_consumer.py`
- `eusolicit-app/services/notification/src/notification/main.py`
- `eusolicit-app/services/notification/src/notification/workers/tasks/billing_usage_sync.py`
- `eusolicit-app/services/notification/README.md`

## Change Log

- 2026-04-20: Initial story context created. (bmad-create-story)
- 2026-04-20: Verified all implementation and tests. Bumped `eusolicit-models` to `0.1.3`. Created `services/notification/README.md`. (gemini-2.0-flash-thinking-exp-01-21)
- 2026-04-20: Documented deliberate 03:00 UTC Beat schedule (supersedes Epic 02:00 UTC) to avoid DB contention with analytics.
- 2026-04-20: Confirmed `_publish_trial_expiring` stream realignment from `"trial.expiring"` to `"eu-solicit:subscriptions"`.
- 2026-04-20: Senior Developer Review (bmad-code-review) — REVIEW: Approve.

## Senior Developer Review

**Verdict:** REVIEW: Approve

**Summary:** All 14 ACs are implemented and verified by passing tests (30 unit + 6 lifespan + 8 events + 4 webhook stream-alignment). Implementation faithfully mirrors the Story 9.10 consumer skeleton (TypeAdapter parsing, per-target SETNX, `asyncio.to_thread`, structured logging, cross-tenant isolation). The Stripe hardening pass adds the explicit `idempotency_key={item_id}:{period}`, enforces archive+delete-only-after-success via the `stripe_success` sentinel, and correctly differentiates transient (raise → Celery retry) vs. terminal (log + skip) Stripe errors. The integration tests with real Redis confirm the E09-R-004 P0 non-negotiable both ways.

**Code quality:** Strong. Module-level `TypeAdapter`, idempotent constants, `_message_id` bytes→str normalization, ACK-after-fan-out with per-admin guards, and a defensive Stripe v9 compat shim are all on-pattern with prior 9.x consumers.

**Test coverage:** Exceeds the test-design contribution count (≥12 + 4 P0 atomicity scenarios). Cross-tenant negative is present (Story 9.10 N2 carry-over satisfied). Atomicity tests include both unit (mocked Redis) and integration (real Redis via testcontainers) tiers as required by Dev Notes.

**Architecture alignment:** Stream naming (`eu-solicit:subscriptions`), consumer-group reuse (`notification-svc`), Pydantic discriminated-union event schema, and Celery task decorator preservation (`max_retries=2`, `default_retry_delay=300`, `acks_late=True`) all confirmed.

**Observations (non-blocking, suggested for future polish):**

1. **Docstring N5 trade-off** — The `SubscriptionConsumer` class docstring is minimal; Dev Notes explicitly called out documenting the "SETNX succeeds → `send_email.delay` raises → admin silently dropped, falls back to DLQ monitoring" trade-off in the consumer's docstring (N5 carry-over). Functionally correct as-is, but the future operator reading the source loses that context. Recommend a 2-line addition to the class docstring.
2. **`_resolve_company_admins` non-UUID handling** — Returns `[]` on `ValueError` from `UUID(company_id)`, which silently routes through the `no_admins` warning path. In production the publisher always emits a UUID, but a malformed event from a future publisher would be hard to diagnose from the `no_admins` log alone. Consider logging the parse failure as `subscription_consumer.invalid_company_id` for forensic clarity.
3. **`stripe_success = False` before `raise`** — In the `RateLimitError`/`APIConnectionError` branch the sentinel is set but never read because the `raise` immediately propagates. Dead but harmless.
4. **Per-task abort on `RateLimitError`** — Re-raising aborts processing of remaining companies in the same Beat run; they get picked up the next day. This is acceptable per AC11 wording, but a per-company `try/except` around the metered-items loop with `self.retry()` per-company would be more efficient at scale. Defer to a follow-up if Enterprise-tier company counts grow.
5. **`AC2` test location** — The realignment test lives in `tests/unit/test_webhook_service_stream_alignment.py` (not `test_webhook_service.py` as the task said). Functionally equivalent and cleaner; flagging only because the Tasks list mentioned the other file.

None of the above blocks acceptance.
