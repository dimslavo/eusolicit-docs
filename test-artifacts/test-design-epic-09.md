---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-19'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 9
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-06.md'
  - 'eusolicit-docs/test-artifacts/test-design-progress.md'
  - 'eusolicit/_bmad/bmm/config.yaml'
---

# Test Design: Epic 9 — Notifications, Alerts & Calendar

**Date:** 2026-04-19
**Author:** TEA Master Test Architect
**Status:** Draft
**Epic:** E09 | **Sprint:** 9–10 | **Points:** 55 | **Dependencies:** E01, E05, E06

---

## Executive Summary

**Scope:** Epic-level test design for E09 — Notifications, Alerts & Calendar. This epic delivers the Notification Service as a purely event-driven Celery worker (no HTTP ingress) that handles all outbound communications: alert digest emails matched against user CPV/region/budget preferences, Google Calendar and Microsoft Outlook OAuth2 synchronization, iCal feed generation, task/approval lifecycle emails, trial expiry reminders, Stripe usage metering, and scheduled report delivery. Functional scope spans: Notification Service scaffold + Celery configuration (S09.01), notification schema migrations (S09.02), alert preferences CRUD in Client API (S09.03), alert matching + immediate dispatch via Redis Streams (S09.04), daily/weekly digest assembly (S09.05), SendGrid email delivery + webhook status tracking (S09.06), iCal feed generation (S09.07), Google Calendar OAuth2 + sync (S09.08), Microsoft Outlook OAuth2 + sync via Graph API (S09.09), task and approval notification consumers (S09.10), trial expiry + Stripe usage sync (S09.11), alert preferences frontend page (S09.12), calendar connection management frontend page (S09.13), and scheduled report generation task (S09.14).

This epic introduces three distinct **high-severity attack surfaces**: OAuth2 token storage with Fernet encryption (single encryption key guards all users' external calendar write access), SendGrid webhook receiver (unauthenticated endpoint that mutates audit state), and Stripe usage counter atomicity (crash between report and key deletion causes double-billing or revenue data loss). Additionally, the Redis Streams consumer group architecture requires explicit idempotency design to prevent duplicate alert dispatch — a silent bug with direct user experience impact.

**Risk Summary:**

- Total risks identified: 12
- High-priority risks (≥6): 4 (Fernet key exposure, Redis consumer duplicate dispatch, SendGrid webhook spoof, Stripe counter race)
- Critical categories: SEC (OAuth2 encryption, webhook validation), DATA (Redis ACK idempotency, Stripe atomicity), TECH (calendar diff correctness, Graph API throttling)

**Coverage Summary:**

- P0 scenarios: 33 (~16–28 hours)
- P1 scenarios: 71 (~28–48 hours)
- P2 scenarios: 41 (~12–20 hours)
- P3 scenarios: 5 (~3–6 hours)
- **Total effort:** ~59–102 hours (~1.5–2.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **SendGrid template rendering/design correctness** | Template HTML/CSS and visual rendering are a product design concern; E09 verifies that the correct template ID is referenced and dynamic fields are populated | Assert template_id and dynamic_template_data fields in the SendGrid API request payload; visual rendering tested in UAT |
| **KraftData AI Agent behavior** | No direct AI agent calls in E09; scheduled report content depends on E12 analytics data | S09.14 creates the pipeline with placeholder sections; content quality tested when E12 delivers analytics |
| **Google Calendar / Microsoft Outlook UI** | Third-party calendar apps are out of scope; E09 verifies API-level sync correctness (event create/update/delete) | Assert calendar event payloads sent to Google/Graph APIs via mock interceptors; real calendar rendering is platform behavior |
| **Stripe webhook receiver** | E09 only reads from Redis counters and calls `SubscriptionItem.create_usage_record()`; Stripe event handling is a billing-epic concern | Test the outbound Stripe API call and its error handling; incoming Stripe webhooks tested in the billing epic |
| **Email deliverability and spam scoring** | SendGrid reputation, DNS settings (SPF/DKIM), and inbox placement are infrastructure/ops concerns | Verify only that the API call succeeds and `email_log` is populated; deliverability is an ops responsibility |
| **`eu-solicit:subscriptions` stream publishing** | The `subscription.trial_expiring` event is published by a subscriptions subsystem (E06 dependency); E09 only tests consumption | Mock the Redis Stream event in consumer tests; E06 tests the publish side |
| **Prometheus / alerting for Celery worker health** | Infrastructure concern; E09 verifies the health-check heartbeat task exists | Assert heartbeat task is registered and callable; scrape config and alerting rules are DevOps artifacts |
| **WeasyPrint PDF visual fidelity** | HTML-to-PDF rendering quality is a design concern | Verify PDF is generated without errors, has non-zero byte size, and passes basic structural parse |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E09-R-001** | **SEC** | Fernet calendar token encryption key exposure — `client.calendar_connections` stores Google and Microsoft OAuth2 access/refresh tokens encrypted with a single Fernet key (`CALENDAR_ENCRYPTION_KEY`). If this key is leaked (misconfigured secret, log line, env dump), every stored token for every user is trivially decryptable using the open-source Fernet spec, granting calendar write access to all users' personal and corporate calendars. The blast radius of a single key leak is the entire calendar-connected user base. | 2 | 3 | **6** | Never log the raw Fernet key or decrypted token value at any log level; ensure `CALENDAR_ENCRYPTION_KEY` is sourced from Vault/secrets manager (not .env in production); add a unit test asserting stored token bytes differ from the plaintext token; add an integration test asserting the key is not present in any structured log output during the OAuth flow; rotate key in CI using a synthetic test key distinct from any real credential | Backend Lead | Sprint 9 |
| **E09-R-002** | **DATA** | Redis Streams consumer group duplicate dispatch on redelivery — S09.04 uses `XREADGROUP` with manual ACK (ACK only after full processing). If the worker crashes after dispatching the email but before calling `XACK`, the event is redelivered and a second identical alert is sent to the user. Without an idempotency guard keyed on `(user_id, opportunity_id)` in `notification.alert_log`, every crash-and-redelivery scenario produces a duplicate email. With high-volume ingestion events (TED publishes ~500 opportunities/day), the duplicate rate can be significant. | 2 | 3 | **6** | Before dispatching, check `notification.alert_log` for an existing row with matching `(user_id, opportunity_id, digest_type='immediate')` and `sent_at > threshold`; only dispatch if no recent row exists; use `INSERT ... ON CONFLICT DO NOTHING` pattern inside the same DB transaction that precedes email dispatch; integration test: simulate crash-before-ACK, verify redelivery does not produce second email row in `alert_log` | Backend Lead | Sprint 9 |
| **E09-R-003** | **SEC** | SendGrid Event Webhook signature bypass — `POST /webhooks/sendgrid` receives delivery events (delivered/bounced/failed) that update `notification.email_log.status`. If the endpoint does not validate the `X-Twilio-Email-Event-Webhook-Signature` header using SendGrid's ECDSA verification library, any unauthenticated caller can POST a fabricated `delivered` event for any sendgrid_message_id, silently masking real bounces or failures from operations monitoring. | 2 | 3 | **6** | Implement SendGrid webhook signature validation using `sendgrid.helpers.eventwebhook.EventWebhook.verify_signature()`; return 403 on invalid signature; use raw request body (not parsed JSON) for signature computation; unit test: POST with invalid signature → 403; POST with valid signature → 200 + `email_log.status` updated | Backend Lead | Sprint 9 |
| **E09-R-004** | **DATA** | Stripe usage counter atomicity — S09.11 reads Redis usage counters and calls `stripe.SubscriptionItem.create_usage_record()`, then clears the counters. If the Celery task crashes between the Stripe API call returning success and the Redis `DEL`/`GETDEL` completing, usage is double-reported on the next run. Conversely, if DEL is issued before the Stripe call completes, successful usage data is permanently lost. Either path directly corrupts billing records and may violate Stripe's idempotency requirements. | 2 | 3 | **6** | Strictly enforce operation order: (1) read counter; (2) call Stripe with idempotency key = `f"{subscription_item_id}:{billing_period_key}"`; (3) only delete Redis key on confirmed 200 response; never delete before API call completes; on retry, Stripe silently deduplicates via same idempotency key; integration test: mock Stripe failure → verify counter intact; mock Stripe success → verify counter deleted | Backend Lead | Sprint 10 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E09-R-005 | DATA | Digest window calculation edge case — `MAX(sent_at)` per user falls back to 24h/7d if no prior `alert_log` row exists. If a prior digest run matched zero opportunities (producing no row), the fallback anchor resets, potentially creating overlapping windows. Alternatively, if a zero-match row is written, consecutive digests may produce disjoint windows with gaps. | 2 | 2 | 4 | Explicitly define: `alert_log` rows are written only when at least one opportunity matches; zero-match runs leave no trace; fallback anchor for first-time users is `now() - interval`; integration tests for three consecutive digest windows assert no opportunity_id appears in more than one digest for the same user | Backend Lead | Sprint 9 |
| E09-R-006 | TECH | Alert preference in-memory cache staleness — the consumer refreshes cached preferences every 60 seconds. A newly activated preference may miss immediate opportunities for up to 60 seconds. Combined with the 60-second dispatch SLA from the acceptance criteria, the first immediate alert for a newly enabled preference may functionally violate the SLA. | 2 | 2 | 4 | Force synchronous cache refresh on alert preference CRUD writes (via Redis pub/sub invalidation or reduced TTL 10–15s); or document the accepted one-cycle latency with a product waiver; add end-to-end latency assertion: create preference → publish event → measure dispatch time ≤ 60s | Backend Lead | Sprint 9 |
| E09-R-007 | TECH | Calendar sync diff logic correctness under partial failure — if a sync run commits some calendar API calls but fails before completing all, `client.calendar_events` is partially updated. The next sync diffs from this corrupted state and may generate phantom creates, missed updates, or redundant deletes. | 2 | 2 | 4 | Wrap each sync run's DB updates in a transaction; roll back `calendar_events` changes if not all API calls confirm; mark each row `sync_status = in_progress` → `committed` only after batch confirms; test: inject API failure mid-batch, verify `calendar_events` is consistent with pre-sync state | Backend Lead | Sprint 10 |
| E09-R-008 | TECH | Microsoft Graph API 429 thundering-herd — if the sync task ignores the `Retry-After` header and uses default exponential backoff, every 15-minute sync cycle will retry too early, amplifying the rate limit violation. With max 5 retries per task and N_users connections, a Graph API throttle could block sync for hours. | 2 | 2 | 4 | Parse `Retry-After` header; pass value to `self.retry(countdown=retry_after)`; integration test: mock Graph returning 429 with `Retry-After: 30`; assert retry fires at ~30s; assert no more than 1 retry within the rate limit window | Backend Lead | Sprint 10 |
| E09-R-009 | OPS | Celery Beat timezone drift — Beat and worker are separate Docker Compose services; if `CELERY_TIMEZONE=UTC` is not set in both, crontab expressions evaluate against OS local time, silently shifting digest delivery windows. | 1 | 3 | 3 | Explicitly set `CELERY_TIMEZONE = 'UTC'` and `CELERY_ENABLE_UTC = True` in both worker and beat Celery app config; verify in CI with a unit test that crontab `hour=7` fires at 07:00 UTC via `celery.schedules.crontab` evaluation against a fixed mock time | DevOps / Backend | Sprint 9 |
| E09-R-010 | BUS | Trial expiry reminder event coupling — `subscription.trial_expiring` is published by E06; if E06 never publishes this event or publishes with incorrect timing/fields, users receive no upgrade reminder. The Notification Service is purely reactive and cannot compensate. | 2 | 2 | 4 | E09 tests consumer in isolation with mocked stream events; add cross-service contract test verifying E06 publishes the event 3 days before trial end with required fields; document as dependency risk in E06 interworking | QA / Backend | Sprint 9–10 |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E09-R-011 | PERF | Digest assembly N+1 risk — per-user DB queries instead of batch load causes O(N_users) queries per digest run; timeout risk at scale | 1 | 2 | 2 | Verify implementation uses batch queries; add integration test with 100 seeded users asserting task completes in < 30s; query count assertion via SQLAlchemy event listener |
| E09-R-012 | SEC | iCal token exposed in access logs — URL path `/api/v1/calendar/ical/{token}` embeds token in access logs without scrubbing | 1 | 2 | 2 | Document log scrubbing requirement for this route; add structured logging with path redaction; include in security review checklist |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

### System-Level Risk Inheritance

- **E09-R-001** (Fernet key exposure) → extends system **R-04** (Data Encryption at Rest). Calendar tokens represent externally-scoped OAuth credentials; a key leak grants write access to third-party systems beyond EU Solicit's perimeter.
- **E09-R-002** (Consumer group duplicate dispatch) → extends system **R-02** (Event Bus Reliability) from E01. Idempotency is the consumer-side guarantee that complements at-least-once delivery.
- **E09-R-003** (SendGrid webhook spoof) → extends system **R-03** (Webhook Integrity / HMAC validation). The same pattern as KraftData webhook validation (E04) must be applied to SendGrid.
- **E09-R-004** (Stripe counter atomicity) → extends system **R-05** (Billing Accuracy). This is the revenue-critical complement to E06's usage metering (E06-R-002); both must be airtight for billing to be trustworthy.

---

## Entry Criteria

- [ ] E01 Redis Streams event bus operational with `eu-solicit:opportunities`, `eu-solicit:tasks`, `eu-solicit:approvals`, `eu-solicit:subscriptions` streams confirmed publishing
- [ ] E05 data pipeline publishing `opportunities.ingested` events with required metadata fields (CPV codes, region, budget, deadline)
- [ ] E06 `client.alert_preferences` and `client.calendar_connections` table schemas confirmed; read-only access granted to Notification Service DB role
- [ ] SendGrid account configured with dynamic templates for all 7 template types; template IDs available for environment variable configuration
- [ ] Google OAuth2 credentials (`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`) available in test environment
- [ ] Microsoft OAuth2 credentials (`MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`, `MICROSOFT_TENANT_ID`) available in test environment
- [ ] `CALENDAR_ENCRYPTION_KEY` Fernet key generated and stored in test secrets manager
- [ ] `STRIPE_API_KEY` (test mode) available; Stripe test subscriptions seeded
- [ ] Docker Compose `notification` service definition reviewed and running
- [ ] Test data factories for alert preferences, calendar connections, and email log seeded

## Exit Criteria

- [ ] All P0 tests passing (100% pass rate)
- [ ] All P1 tests passing or failures triaged with approved waivers (≥95%)
- [ ] E09-R-001 through E09-R-004 (high-priority risks) verified as mitigated
- [ ] SendGrid webhook signature validation confirmed operational in test environment
- [ ] Fernet token encryption verified: stored bytes differ from plaintext in DB
- [ ] Immediate alert dispatch end-to-end latency measured and confirmed ≤ 60 seconds
- [ ] No P0 or P1 open defects without approved waivers
- [ ] Coverage ≥ 80% on Celery task modules and Redis consumer modules
- [ ] iCal feed validated as parseable by `icalendar` library and structural content verified

---

## Test Coverage Plan

### P0 (Critical) — Run on every commit

**Criteria:** Blocks core notification delivery + High risk (≥6) + No workaround

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| S09.01: Celery worker starts, connects Redis broker, routes tasks to correct queues (`alerts`, `emails`, `calendar-sync`, `usage-reporting`) | Integration | E09-R-002 | 3 | QA | `testcontainers` Redis; verify queue routing via `app.tasks` registry + queue inspection |
| S09.04: Alert matching logic — CPV sector overlap, region match, budget range, deadline proximity (all 4 dimensions + boundaries) | Unit | — | 8 | Dev | Pure Python; parametrize across 4 match dimensions + boundary values |
| S09.04: Redis consumer group ACK + idempotency guard — crash-before-ACK redelivery does not produce duplicate `alert_log` row or duplicate email | Integration | E09-R-002 | 4 | QA | Simulate worker crash before XACK; verify `INSERT … ON CONFLICT DO NOTHING` prevents second email |
| S09.06: SendGrid webhook — valid ECDSA signature → 200 + `email_log.status` updated; invalid signature → 403 | API | E09-R-003 | 3 | QA | SendGrid test ECDSA key pair; verify raw body used for sig computation; test forged POST rejected |
| S09.08: Fernet token encryption — stored bytes differ from plaintext; `decrypt(encrypt(t)) == t` | Unit | E09-R-001 | 2 | Dev | Synthetic `CALENDAR_ENCRYPTION_KEY`; never logs plaintext |
| S09.11: Stripe counter atomicity — counter cleared only after confirmed Stripe success; crash before DEL leaves counter intact for retry | Integration | E09-R-004 | 4 | QA | Mock `stripe.SubscriptionItem.create_usage_record`; inject success/failure; verify Redis state |
| S09.03: Max 5 alert preferences per user enforced; 6th attempt returns 409 | API | — | 2 | QA | Create 5 preferences; verify 6th → 409; verify check is user-scoped |
| S09.06: `send_email` task dispatches for all 7 template types with correct `template_id` and `email_log` entry | Integration | — | 7 | Dev | Mock SendGrid API; assert correct env-var-mapped template_id per type; assert `email_log` row created |

**Total P0:** 33 tests, ~16–28 hours

---

### P1 (High) — Run on PR to main

**Criteria:** Critical paths + medium/high risk + common user workflows

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| S09.02: Migration creates `notification.alert_log`, `notification.email_log`, `notification.sync_log` with correct column types, constraints, indexes; migration is reversible | Integration | — | 4 | Dev | Testcontainers Postgres; verify via `information_schema`; verify downgrade drops tables cleanly |
| S09.03: Full CRUD — POST create, GET list, PUT update, DELETE, PATCH toggle; all require auth; scoped to current user | API | — | 7 | QA | `httpx` + `pytest-asyncio`; assert 401 unauthenticated; assert 404 (not 403) for other user's preference |
| S09.03: CPV validation (invalid code → 422), region validation (non-EU country → 422), budget range (min ≥ 0, max > min) | API | — | 5 | QA | Parametrize with known bad CPV codes, non-EU country codes, inverted budget range |
| S09.04: Non-matching opportunities — ACKed via XACK without `send_email` dispatch | Unit | — | 3 | Dev | Mock `send_email`; assert NOT called for no-match across CPV / region / budget / deadline scenarios |
| S09.04: Malformed event bytes — logs structlog `error`, issues XACK, does not redelivery-loop | Unit | — | 2 | Dev | Inject invalid JSON bytes; assert log call; assert XACK issued; no exception raised |
| S09.05: Daily digest Celery task triggers at 07:00 UTC; processes all daily-schedule preferences in correct window | Integration | E09-R-005 | 4 | QA | `freezegun` clock at 07:00 UTC; call task directly; verify correct `MAX(sent_at)` window query; assert email dispatch count |
| S09.05: Weekly digest triggers Monday 07:00 UTC; zero-match users skipped (no email) | Integration | E09-R-005 | 3 | QA | Freeze to Monday 07:00; seed matching and non-matching users; verify only matching users receive email |
| S09.05: Digest groups multiple matched opportunities into single email per user | Unit | — | 2 | Dev | Seed 5 matching opportunities for 1 user; assert `send_email` called once with all 5 opp IDs in payload |
| S09.06: `email_log` row created with `sendgrid_message_id` and initial `status='sent'` after dispatch | Integration | — | 2 | Dev | Assert `notification.email_log` has row with correct fields after task completion |
| S09.06: Retry on 429 and 5xx from SendGrid with exponential backoff | Unit | — | 3 | Dev | Mock SendGrid: 429 → 500 → 200; assert Celery `retry()` called with backoff; assert eventual success |
| S09.07: `GET /api/v1/calendar/ical/{token}` returns valid `.ics` parseable by `icalendar`; VEVENT count matches tracked opportunities; correct Content-Type | API | — | 4 | QA | Parse response with `icalendar` library; verify VEVENT count; assert `text/calendar` Content-Type |
| S09.07: Invalid/revoked token → 404 (not 401); `POST .../generate-token` rotates token; old token → 404 | API | — | 3 | QA | Assert 404 on unknown token; call generate-token twice; assert first token is invalidated |
| S09.08: Google Calendar periodic sync — creates, updates, and deletes events based on diff against `client.calendar_events` | Integration | E09-R-007 | 5 | QA | Mock `google-api-python-client`; seed `calendar_events`; assert correct API call types per create/update/delete scenario |
| S09.08: Google token refresh — access token expired → transparent refresh via stored refresh token before sync proceeds | Unit | — | 2 | Dev | Mock Google token endpoint; set `expires_at` in past; trigger sync; assert refresh exchange called first |
| S09.09: Microsoft Outlook OAuth2 callback — exchanges code for tokens; stores Fernet-encrypted token with `provider='microsoft'` | Integration | — | 3 | QA | Mock MSAL token exchange; verify encrypted token in `client.calendar_connections`; verify Fernet applied |
| S09.09: Microsoft Graph sync creates/updates/deletes events; 429 respects `Retry-After` header | Integration | E09-R-008 | 4 | QA | Mock Graph 429 with `Retry-After: 30`; assert `self.retry(countdown=30)`; assert no immediate retry |
| S09.10: `task.assigned` event → `send_email` called with `task_assigned` template and assignee email | Integration | — | 2 | QA | Publish mock event to consumer; assert correct template + recipient |
| S09.10: `approval.requested` event → `send_email` called for each required approver | Integration | — | 2 | QA | Seed 3 approvers; publish event; assert 3 `send_email` calls with `approval_requested` template |
| S09.10: `approval.decided` event → `send_email` called with `approval_decided` template to proposal owner | Integration | — | 2 | QA | Assert correct template + proposal owner email |
| S09.11: `subscription.trial_expiring` event → sends reminder email with `days_remaining`, tier comparison, upgrade CTA | Integration | E09-R-010 | 3 | QA | Mock stream event; assert `send_email` with `trial_expiry_reminder` template + `days_remaining` in template_data |
| S09.12: Alert preferences page renders existing preferences as editable cards; 5-preference limit message shown correctly | Component | — | 3 | Dev | Testing Library; mock API with 3 preferences; assert 3 cards; assert create button disabled at limit |
| S09.12: Schedule selector; enable/disable toggle calls PATCH with only `is_active` field | Component | — | 3 | Dev | Simulate toggle click; assert PATCH called with `{is_active: bool}` only |
| S09.13: Google Calendar connect button initiates OAuth; page shows connected state on return with `?connected=google` | Component/E2E | — | 3 | QA | Mock OAuth redirect; assert connected state with last-synced timestamp and disconnect button |
| S09.13: Disconnect shows confirmation dialog; confirm calls DELETE API and removes connection display | Component | — | 2 | Dev | Simulate disconnect click; assert dialog; confirm; assert DELETE API called |

**Total P1:** 71 tests, ~28–48 hours

---

### P2 (Medium) — Run nightly / weekly

**Criteria:** Secondary flows + low risk + edge cases + validation boundaries

| Requirement | Test Level | Risk Link | Test Count | Owner | Notes |
|-------------|------------|-----------|------------|-------|-------|
| S09.03: `deadline_days_ahead` boundary — 0 → 422, 1 → 201, 90 → 201, 91 → 422 | API | — | 3 | QA | Parametrize boundary values |
| S09.03: Ownership isolation — GET/PUT/DELETE on another user's preference returns 404, not 403 | API | — | 2 | QA | Seed preference for user A; call with user B JWT; assert 404 |
| S09.04: In-memory preference cache refreshes within cycle; new preference included after 60s advance | Unit | E09-R-006 | 3 | Dev | `freezegun`; create preference; advance time 65s; assert cache includes new preference |
| S09.05: Digest deduplication — same opportunity_id does not appear in two consecutive digest `alert_log` rows for same user | Integration | E09-R-005 | 2 | QA | Run two consecutive daily digests with shared opportunity in both windows; assert single occurrence in `alert_log` |
| S09.05: `alert_log` `opportunity_ids` ARRAY contains all matched IDs after digest dispatch | Unit | — | 2 | Dev | Seed 5 matching opportunities; assert all 5 IDs in `alert_log.opportunity_ids` |
| S09.06: Invalid `template_type` raises `ValueError`; task does not invoke Celery `retry()` | Unit | — | 2 | Dev | Pass `template_type='invalid'`; assert `ValueError`; assert `max_retries` not invoked |
| S09.07: VEVENT fields — `summary`, `dtstart`/`dtend` (all-day), `description`, `url`, `uid = '{opp_id}@eusolicit.com'` deterministic | API | — | 4 | QA | Seed 2 opportunities with known data; parse VEVENTs; assert field values + deterministic UID formula |
| S09.07: Regenerate token creates new token; old token → 404 | API | — | 2 | QA | Generate; regenerate; assert old → 404; new → 200 |
| S09.08: `notification.sync_log` row created after sync with correct `events_created`, `events_updated`, `events_deleted` counts | Integration | — | 2 | Dev | Run sync with 2 creates + 1 update; assert `sync_log` counts |
| S09.08: Google disconnect — revokes Google token via API + deletes `calendar_connections` row | API | — | 2 | QA | Mock Google revoke endpoint; assert connection deleted and revoke called |
| S09.09: Microsoft disconnect — revokes token + deletes connection row | API | — | 2 | QA | Mirror Google disconnect test for Microsoft Graph revoke |
| S09.10: `task.overdue` event → email to assignee AND team lead | Integration | — | 2 | QA | Publish event; assert 2 `send_email` calls with correct recipients |
| S09.10: Missing user data (user deleted) — consumer logs structlog `warning`, issues XACK, no exception | Unit | — | 2 | Dev | Mock `client.users` query returning None; assert warning log; assert XACK called; no exception |
| S09.11: Stripe 429 → task retry; `InvalidRequestError` (invalid subscription) → skip without retry | Unit | — | 3 | Dev | Mock Stripe SDK exceptions; assert retry on 429; assert no retry on `InvalidRequestError` |
| S09.11: Redis counter value after confirmed Stripe success is cleared | Integration | E09-R-004 | 2 | QA | Complement of P0-004; mock Stripe success; assert Redis key deleted |
| S09.12: Delete preference — confirm dialog shown; cancel does not call DELETE API | Component | — | 2 | Dev | Click delete; dialog shown; click cancel; assert DELETE not called |
| S09.13: iCal URL copy-to-clipboard calls `navigator.clipboard.writeText`; regenerate warning text present | Component | — | 2 | Dev | Assert clipboard API called on copy; assert warning text visible before regenerate |
| S09.13: Sync status 30-second poll updates last-synced display without full page reload | Component | — | 2 | Dev | `jest.useFakeTimers()`; advance 30s; assert API called; assert display updated |
| S09.14: PDF generated without errors; non-zero byte size; stored in S3 (localstack); `report_metadata` row created | Integration | — | 2 | QA | Mock analytics query; call task; assert S3 object + metadata row |
| S09.14: Report generation exception logged; worker does not crash | Unit | — | 2 | Dev | Mock WeasyPrint to raise; assert structlog `error`; assert Celery failure handled |

**Total P2:** 41 tests, ~12–20 hours

---

### P3 (Low) — Run on-demand

**Criteria:** Nice-to-have + exploratory + benchmark + manual verification

| Requirement | Test Level | Test Count | Owner | Notes |
|-------------|------------|------------|-------|-------|
| S09.12: CPV multi-select search renders 10,000 codes without UI freeze (< 2s render) | Component/Performance | 1 | Dev | Testing Library with full CPV JSON; assert render completes < 2s |
| S09.01: Failed task (max 5 retries exhausted) lands in DLQ stream; DLQ entry contains original payload | Manual/Integration | 1 | QA | Force Celery task to fail 5 times; verify DLQ stream in Redis; inspect payload |
| E2E: Full alert flow — create preference → publish `opportunities.ingested` → email dispatched within 60s | E2E | 1 | QA | Requires full Notification Service + Redis + SendGrid mock; tagged `@slow @skip-ci` |
| S09.14: Report emailed to all company admin users; metadata row includes S3 path for retrieval | Integration | 1 | QA | Seed company with 3 admins; run task; assert 3 `send_email` calls; assert metadata with S3 path |
| S09.13: Microsoft Outlook error state (expired token) displays reconnect CTA | E2E | 1 | QA | Seed expired-token connection; visit page; assert error state and reconnect button |

**Total P3:** 5 tests, ~3–6 hours

---

## Execution Order

### Smoke Tests (< 5 min)

**Purpose:** Detect deployment-breaking issues in Notification Service startup

- [ ] Celery worker process starts and logs heartbeat task registration (15s)
- [ ] Celery Beat process starts and logs crontab schedule for daily/weekly digest (10s)
- [ ] `POST /api/v1/alerts/preferences` returns 422 with empty body (10s)
- [ ] `GET /api/v1/calendar/ical/invalid-token` returns 404 (10s)
- [ ] `POST /webhooks/sendgrid` without signature header returns 403 (10s)

**Total:** 5 smoke checks, ~1 minute

### P0 Tests (< 20 min)

**Purpose:** Critical path validation — idempotency, security, delivery correctness

- [ ] Alert matching unit parametrize × 8 (Unit, ~2 min)
- [ ] Fernet encryption roundtrip (Unit, ~30s)
- [ ] Consumer group idempotency guard (Integration, ~4 min)
- [ ] SendGrid webhook signature accept/reject (API, ~1 min)
- [ ] Stripe counter atomicity crash/success scenarios (Integration, ~3 min)
- [ ] Max 5 preferences enforcement (API, ~1 min)
- [ ] `send_email` × 7 template types (Integration, ~3 min)
- [ ] Celery queue routing (Integration, ~2 min)

**Total:** 33 P0 tests, ~17 min

### P1 Tests (< 50 min)

**Purpose:** Full functional coverage of all notification paths

- [ ] Schema migration integrity (Integration, ~3 min)
- [ ] Alert preferences CRUD + validation (API, ~5 min)
- [ ] Redis stream consumer happy paths + error handling (Integration, ~8 min)
- [ ] Daily/weekly digest assembly (Integration, ~6 min)
- [ ] iCal feed + token management (API, ~4 min)
- [ ] Google Calendar sync + token refresh (Integration, ~6 min)
- [ ] Microsoft Graph sync + throttle handling (Integration, ~5 min)
- [ ] Task / approval / trial consumers (Integration, ~4 min)
- [ ] Frontend alert preferences page (Component, ~4 min)
- [ ] Frontend calendar connection page (Component, ~3 min)

**Total:** 71 P1 tests, ~48 min

### P2/P3 Tests (nightly, < 90 min)

**Purpose:** Full regression, edge cases, and boundary coverage

- [ ] Validation boundaries + ownership isolation (API, ~6 min)
- [ ] In-memory cache refresh cycle (Unit, ~2 min)
- [ ] Digest deduplication (Integration, ~4 min)
- [ ] VEVENT field correctness + token rotation (API, ~4 min)
- [ ] Sync logging + disconnect flows (Integration, ~6 min)
- [ ] Worker error handling + malformed event edge cases (Unit, ~5 min)
- [ ] Frontend polish + polling tests (Component, ~4 min)
- [ ] PDF generation + S3 storage (Integration, ~4 min)
- [ ] P3 exploratory and performance (Various, ~20 min)

**Total:** 46 P2/P3 tests, ~55 min

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Hours/Test (avg) | Total Hours | Notes |
|----------|-------|-----------------|-------------|-------|
| P0 | 33 | 0.65 | ~16–28 | Integration setup (Redis consumer groups, Stripe mocks, Fernet) dominates |
| P1 | 71 | 0.50 | ~28–48 | OAuth2 flow mocking, Google/Graph API intercepts add setup overhead |
| P2 | 41 | 0.35 | ~12–20 | Edge cases and boundary tests; lighter setup |
| P3 | 5 | 0.70 | ~3–6 | E2E requires full notification stack |
| **Total** | **150** | **—** | **~59–102** | **~1.5–2.5 weeks, 1 QA** |

### Prerequisites

**Test Data Factories:**

- `AlertPreferenceFactory` — generates `client.alert_preferences` rows with configurable CPV sectors, regions, budget, schedule, is_active flags
- `CalendarConnectionFactory` — generates `client.calendar_connections` rows for google/microsoft/ical providers with Fernet-encrypted mock tokens
- `OpportunityEventFactory` — generates `opportunities.ingested` Redis Stream event payloads matching E05 envelope schema
- `EmailLogFactory` — generates `notification.email_log` rows for status transition tests
- `SyncLogFactory` — generates `notification.sync_log` rows for calendar sync audit tests

**Tooling:**

- `pytest-asyncio` + `testcontainers[redis,postgresql]` for integration tests
- `fakeredis` for isolated Redis unit tests (serial only — NOT for consumer group idempotency concurrency tests)
- `respx` for mocking `httpx` calls to Google Calendar API, Microsoft Graph API
- `unittest.mock` / `pytest-mock` for Celery task interception and SendGrid SDK
- `freezegun` for Celery Beat schedule assertions and cache TTL tests
- `icalendar` Python library for `.ics` response parsing and VEVENT field assertions
- `stripe-mock` (or `pytest-mock` patching) for Stripe SDK
- Synthetic Fernet test key (32-byte base64) via `TEST_CALENDAR_ENCRYPTION_KEY` env var
- SendGrid test ECDSA key pair for webhook signature validation
- `localstack` for S3 presigned URL and PDF storage tests (S09.14)

**Environment Requirements:**

- `make infra` running (PostgreSQL + Redis 7 available); `testcontainers` provisions isolated instances per test session
- Docker Compose `notification` Celery worker + Beat reachable for smoke tests
- `SENDGRID_API_KEY=SG.test_*` (test mode, no real sends)
- `STRIPE_API_KEY=sk_test_*` (test mode, no real charges)
- Google OAuth test app credentials (revoke-safe, `calendar.events` scope approved in test project)
- Microsoft test tenant credentials (`Calendars.ReadWrite` scope available)

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions; any P0 failure blocks sprint completion)
- **P1 pass rate:** ≥95% (failures require triaged issue with owner and resolution ETA)
- **P2/P3 pass rate:** ≥90% (informational; failures tracked but do not block release)
- **High-risk mitigations (E09-R-001–R-004):** 100% verified complete before Sprint 10 close

### Coverage Targets

- **Celery task modules** (`tasks/alerts.py`, `tasks/email.py`, `tasks/calendar.py`, `tasks/usage.py`): ≥80%
- **Redis consumer modules:** ≥80%
- **Security-critical paths** (Fernet encryption, webhook signature, XACK idempotency, Stripe atomicity): 100% test coverage
- **Alert matching logic:** 100% branch coverage (all 4 matching dimensions)
- **Frontend components** (alert preferences page, calendar connection page): ≥70%

### Non-Negotiable Requirements

- [ ] All P0 tests pass
- [ ] E09-R-001: stored token bytes differ from plaintext in `calendar_connections` — verified by unit test
- [ ] E09-R-002: crash-redelivery produces no duplicate `alert_log` row — verified by integration test
- [ ] E09-R-003: forged SendGrid webhook POST returns 403 — verified by API test
- [ ] E09-R-004: Stripe counter intact after Stripe failure; cleared after success — verified by integration test
- [ ] Celery Beat UTC timezone verified by unit test against mock clock

---

## Mitigation Plans

### E09-R-001: Fernet Calendar Token Encryption Key Exposure (Score: 6)

**Mitigation Strategy:** Validate encryption in tests (stored != plaintext, roundtrip passes); enforce no-log policy for key and decrypted token values in structlog config; source `CALENDAR_ENCRYPTION_KEY` from Vault in production (not `.env`); add integration assertion that OAuth callback structured logs contain no token substring; rotate key quarterly

**Owner:** Backend Lead
**Timeline:** Sprint 9 — before S09.08 implementation ships
**Status:** Planned
**Verification:** `test_token_encryption_roundtrip` passes; manual log review during OAuth flow shows no plaintext token

---

### E09-R-002: Redis Consumer Group Duplicate Dispatch (Score: 6)

**Mitigation Strategy:** Add idempotency guard in `dispatch_immediate_alert()` using `INSERT INTO notification.alert_log (user_id, opportunity_id, digest_type, ...) ON CONFLICT (user_id, opportunity_id, digest_type) WHERE sent_at > now() - interval '1 hour' DO NOTHING`; check `affected_rows == 1` before calling `send_email`; this makes the dispatch conditional on a successful row insert

**Owner:** Backend Lead
**Timeline:** Sprint 9 — S09.04 implementation
**Status:** Planned
**Verification:** Integration test `test_consumer_crash_before_ack_no_duplicate_email` passes

---

### E09-R-003: SendGrid Webhook Signature Bypass (Score: 6)

**Mitigation Strategy:** Implement `EventWebhook.verify_signature(payload, signature, timestamp, public_key)` from `sendgrid.helpers.eventwebhook`; use raw request body (not parsed JSON) for verification; store public key in `SENDGRID_WEBHOOK_PUBLIC_KEY` env var; return 403 immediately on failure before any DB access

**Owner:** Backend Lead
**Timeline:** Sprint 9 — S09.06 implementation
**Status:** Planned
**Verification:** `test_sendgrid_webhook_invalid_signature_returns_403` and `test_sendgrid_webhook_valid_updates_email_log` both pass

---

### E09-R-004: Stripe Usage Counter Atomicity (Score: 6)

**Mitigation Strategy:** Enforce strict operation order: read counter → call Stripe with `idempotency_key=f"{subscription_item_id}:{billing_period}"` → `GETDEL` Redis key only on confirmed 200 response; never delete before API confirmation; Stripe silently deduplicates on retry via same idempotency key

**Owner:** Backend Lead
**Timeline:** Sprint 10 — S09.11 implementation
**Status:** Planned
**Verification:** `test_stripe_counter_intact_on_failure` and `test_stripe_counter_cleared_on_success` both pass

---

## Assumptions and Dependencies

### Assumptions

1. E01 Redis Streams publishes events in the standard envelope format (`event_type`, `payload`, `timestamp`, `source_service`) as established in E01's Redis Streams design (E01-S1.5)
2. E05 `opportunities.ingested` event payload includes all fields required for alert matching: `cpv_codes[]`, `region`, `budget_amount`, `deadline_date`
3. E06 `client.alert_preferences` and `client.calendar_connections` table schemas are finalized; read-only access is granted to the Notification Service DB role before Sprint 9 starts
4. SendGrid account has 7 dynamic templates provisioned; template IDs are deterministic and available as environment variables
5. Google OAuth test app has `calendar.events` scope approved in the test GCP project; Microsoft test tenant has `Calendars.ReadWrite` scope available
6. WeasyPrint is installable in the Python 3.12 container without OS-level dependency conflicts (C library dependencies managed in Dockerfile)
7. The Notification Service accesses `client.users`, `client.subscriptions`, and pipeline tables read-only; it does not own or run migrations on these tables

### Dependencies

1. **E01 Redis Streams event bus** — Required for all consumer tests; `testcontainers` Redis simulates stream events in CI
2. **E05 `opportunities.ingested` event schema** — Must be stable before S09.04 ATDD; breaking schema changes require test fixture updates
3. **E06 `client.alert_preferences` schema** — Must be frozen before S09.03 API tests; schema changes require migration + fixture updates
4. **SendGrid dynamic template IDs** — Must be configured in test env before S09.06 ATDD in Sprint 9
5. **Google/Microsoft OAuth test credentials** — Must be provisioned in test secrets manager before Sprint 10 calendar sync testing

### Risks to Plan

- **Risk:** E06 `client.alert_preferences` schema changes after S09.03 is implemented
  - **Impact:** All alert matching + CRUD tests require schema fixture updates
  - **Contingency:** Pin E06 table schema with a migration snapshot in E09 test fixtures; add schema change CI detection

- **Risk:** SendGrid template IDs not available for Sprint 9 development
  - **Impact:** `send_email` task built but not integration-tested with real template IDs
  - **Contingency:** Mock SendGrid API entirely in CI; add template ID smoke test run against real SendGrid sandbox post-sprint

- **Risk:** Google OAuth test credentials expire during Sprint 10
  - **Impact:** S09.08 integration tests fail in CI
  - **Contingency:** All Google/Graph API calls mocked at `httpx` level in CI; only `@skip-ci` smoke tests use real credentials

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|-------------------|--------|-----------------|
| **E01 — Redis Streams event bus** | E09 consumes 4 streams published by E01; changes to stream names, event envelope, or consumer group config break E09 consumers | Verify stream names and envelope schema in cross-service smoke tests; use shared `eusolicit-models` event envelope Pydantic models |
| **E05 — Data Pipeline** | `opportunities.ingested` events feed S09.04 alert matching; CPV/region/budget field name or type changes silently break matching | Add cross-service schema contract test: E05 event fixture must match E09 consumer expectation via shared `eusolicit-models` |
| **E06 — Client API** | Alert preferences CRUD (S09.03), iCal/calendar endpoints (S09.07–S09.09), and the SendGrid webhook (S09.06) all live in Client API; `client.alert_preferences` and `client.calendar_connections` read-only by Notification Service | Run E06 full API test suite on any E09 schema migration PR; verify FK references `notification.alert_log → client.users` remain intact |
| **E02 — Authentication** | Calendar OAuth2 endpoints and alert preferences CRUD require valid JWT; auth middleware changes may affect protected route behavior | Include 1 authenticated + 1 unauthenticated test per E09 endpoint in the auth regression matrix |
| **E12 — Analytics** | S09.14 scheduled report queries analytics data (placeholder in E09; populated by E12); E12 must not break the query interface | S09.14 tests use mocked analytics queries; after E12 ships, update mocks to match real query signatures |

---

## Follow-on Workflows (Manual)

- Run `*atdd` to generate failing P0 tests for S09.04 (alert matching + consumer idempotency), S09.06 (SendGrid webhook signature), and S09.08 (Fernet encryption) — separate workflow, not auto-run.
- Run `*automate` for broader P1/P2 coverage automation once Celery task implementation exists (end of Sprint 9).
- Run `*nfr` to assess performance characteristics of digest assembly at scale (P3: N+1 query risk per E09-R-011).

---

## Approval

**Test Design Approved By:**

- [ ] Product Manager: — Date: —
- [ ] Tech Lead: — Date: —
- [ ] QA Lead: — Date: —

**Comments:**

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification framework (P×I scoring, OPEN/MITIGATED/WAIVED lifecycle)
- `probability-impact.md` — Risk scoring methodology (1–3 scale definitions)
- `test-levels-framework.md` — Test level selection (Unit / Integration / API / Component / E2E)
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

### Related Documents

- Epic: `eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md`
- Architecture: `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md`
- Project Context: `eusolicit-docs/project-context.md`
- System Test Design (Architecture): `eusolicit-docs/test-artifacts/test-design-architecture.md`
- System Test Design (QA): `eusolicit-docs/test-artifacts/test-design-qa.md`
- Upstream Epic Test Design (E06): `eusolicit-docs/test-artifacts/test-design-epic-06.md`

---

**Generated by**: BMad TEA Agent - Test Architect Module
**Workflow**: `bmad-testarch-test-design`
**Version**: 4.0 (BMad v6)
