# E09: Notifications, Alerts & Calendar

**Sprint**: 9–10 | **Points**: 55 | **Dependencies**: E01, E05, E06 | **Milestone**: Beta

## Goal

Deliver the Notification Service as a Celery-based worker (no HTTP ingress, purely event-driven) that handles all outbound communications for EU Solicit: email alert digests matched against user-defined CPV/region/budget preferences, calendar synchronization with Google Calendar and Microsoft Outlook via OAuth2, iCal feed generation, task and approval lifecycle emails, trial expiry reminders, Stripe usage reporting, and scheduled report delivery. The service consumes events from Redis Streams (published by E01's event bus), uses SendGrid transactional templates for email, and stores audit logs in the `notification` schema. Alert preference CRUD, calendar OAuth flows, and the iCal feed endpoint live in the Client API; the Notification Service reads those tables read-only and focuses exclusively on background processing.

## Acceptance Criteria

- [ ] Notification Service Celery worker starts, connects to Redis broker, and processes events from `eu-solicit:opportunities`, `eu-solicit:tasks`, `eu-solicit:approvals`, and `eu-solicit:subscriptions` streams
- [ ] Celery Beat scheduler triggers daily digest, weekly digest, calendar sync (every 15 min), Stripe usage report (daily), and scheduled report generation tasks on correct cadences
- [ ] Alert preferences CRUD endpoints in Client API allow users to configure CPV sectors, regions, budget range, deadline proximity, and schedule (immediate/daily/weekly) with validation
- [ ] Immediate alerts are dispatched within 60 seconds of `opportunities.ingested` event for matching users; daily and weekly digests are batched and sent on schedule
- [ ] SendGrid transactional emails render correctly for all template types: alert digest, trial expiry reminder, task assigned, approval requested, approval decided, password reset, welcome email
- [ ] iCal feed endpoint returns valid `.ics` file with deadlines for tracked opportunities, regenerated on opportunity status change
- [ ] Google Calendar OAuth2 flow completes, tokens are stored encrypted in `client.calendar_connections`, and periodic sync creates/updates/deletes calendar events for tracked opportunity deadlines
- [ ] Microsoft Outlook OAuth2 flow completes via Microsoft Graph API with the same sync behavior as Google Calendar
- [ ] Task notification consumer sends email to assigned user on `task.assigned` and `task.overdue` events
- [ ] Approval notification consumer sends email to required approvers on `approval.requested` and to proposal owner on `approval.decided`
- [ ] Trial expiry consumer sends reminder email 3 days before trial end on `subscription.trial_expiring` event
- [ ] Daily Celery task reads Redis usage counters and reports to Stripe Usage Records API
- [ ] All email sends are logged in `notification.email_log` with SendGrid message ID and delivery status
- [ ] Calendar syncs are logged in `notification.sync_log` with event counts and error details
- [ ] Frontend alert preferences page allows CPV multi-select, region checkboxes, budget range, deadline slider, schedule selector, and enable/disable toggle
- [ ] Frontend calendar connection page supports Google OAuth, Outlook OAuth, iCal URL copy, sync status display, and disconnect

## Stories

### S09.01: Notification Service Scaffold & Celery Configuration
**Points**: 3 | **Type**: backend
**Description**: Create the Notification Service project structure as a Celery worker application. Configure Celery with Redis as the broker (connecting to the shared Redis instance from E01). Set up Celery Beat for periodic task scheduling. Define task routing with dedicated queues: `alerts`, `emails`, `calendar-sync`, `usage-reporting`. Configure retry policies (exponential backoff, max 5 retries) and dead-letter handling. Add health check task that logs heartbeat. Set up Alembic for the `notification` schema migrations. Include Docker Compose service definition and `.env` configuration.
**Acceptance Criteria**:
- [ ] Celery worker starts and connects to Redis broker
- [ ] Celery Beat scheduler is configured and triggers a test periodic task
- [ ] Task routing delivers tasks to correct queues (`alerts`, `emails`, `calendar-sync`, `usage-reporting`)
- [ ] Retry policy applies exponential backoff with max 5 retries on task failure
- [ ] Alembic migration creates `notification` schema
- [ ] Docker Compose service runs worker and beat processes
**Implementation Notes**: Use `celery[redis]` with `CELERY_BROKER_URL` pointing to the shared Redis. Separate Beat process from worker process in Docker Compose. Use `task_routes` dict for queue assignment. Dead-letter queue stores failed tasks for inspection.

---

### S09.02: Notification Schema Database Migrations
**Points**: 2 | **Type**: backend
**Description**: Create Alembic migrations for the `notification` schema tables: `alert_log` (id UUID PK, user_id UUID FK, opportunity_ids UUID[], digest_type enum immediate/daily/weekly, sent_at timestamptz, sendgrid_message_id text), `email_log` (id UUID PK, recipient_email text, template_type text, sendgrid_message_id text, status enum sent/delivered/bounced/failed, sent_at timestamptz), and `sync_log` (id UUID PK, calendar_connection_id UUID FK, provider enum google/microsoft, sync_type enum full/incremental, events_created int, events_updated int, events_deleted int, started_at timestamptz, completed_at timestamptz, error_message text). Add indexes on `alert_log.user_id`, `alert_log.sent_at`, `email_log.sendgrid_message_id`, `email_log.status`, and `sync_log.calendar_connection_id`.
**Acceptance Criteria**:
- [ ] Migration creates `notification.alert_log`, `notification.email_log`, `notification.sync_log` tables
- [ ] All columns have correct types, constraints, and defaults
- [ ] Indexes exist on user_id, sent_at, sendgrid_message_id, status, and calendar_connection_id
- [ ] Migration is reversible (downgrade drops tables)
**Implementation Notes**: Use `sa.Enum` for digest_type, status, provider, sync_type with `schema='notification'` to scope enums. UUID primary keys with `server_default=gen_random_uuid()`. `opportunity_ids` uses `ARRAY(UUID)` column type.

---

### S09.03: Alert Preferences CRUD API (Client API)
**Points**: 3 | **Type**: backend
**Description**: Implement CRUD endpoints in the Client API for alert preferences: `POST /api/v1/alerts/preferences` (create), `GET /api/v1/alerts/preferences` (list user's preferences), `PUT /api/v1/alerts/preferences/{id}` (update), `DELETE /api/v1/alerts/preferences/{id}` (delete), `PATCH /api/v1/alerts/preferences/{id}/toggle` (enable/disable). Request/response models: cpv_sectors (text[], validated against known CPV codes), regions (text[], validated against EU country list), budget_min/budget_max (decimal, min >= 0, max > min), deadline_days_ahead (int, 1-90), schedule (enum: immediate/daily/weekly), is_active (bool). Enforce max 5 alert preferences per user. Write to `client.alert_preferences` table. Alembic migration for the table if not already present.
**Acceptance Criteria**:
- [ ] All CRUD endpoints work with proper validation and error responses
- [ ] CPV sectors validated against known code list; invalid codes return 422
- [ ] Regions validated against EU member states; invalid regions return 422
- [ ] Budget range enforced: min >= 0, max > min
- [ ] Maximum 5 alert preferences per user enforced; 6th returns 409
- [ ] Toggle endpoint flips `is_active` without requiring full payload
- [ ] Endpoints require authentication and scope to current user's preferences
**Implementation Notes**: Store `client.alert_preferences` with composite index on `(user_id, is_active)` for efficient active-preference lookup by the Notification Service. Return 404 for preferences owned by other users (not 403, to avoid leaking existence).

---

### S09.04: Alert Matching & Immediate Dispatch
**Points**: 5 | **Type**: backend
**Description**: Implement a Redis Stream consumer in the Notification Service that listens to `eu-solicit:opportunities` for `opportunities.ingested` events. On each event (containing opportunity metadata: CPV codes, region, budget, deadline), query `client.alert_preferences` for active preferences with `schedule = 'immediate'`. Match opportunity against each preference using: CPV sector overlap (any match), region match, budget within range, deadline within `deadline_days_ahead`. For each matching user, assemble an alert payload with opportunity summary and dispatch an immediate email via the SendGrid task (S09.06). Record the match in `notification.alert_log` with `digest_type = 'immediate'`. Use Redis consumer group for at-least-once delivery with manual ACK after processing.
**Acceptance Criteria**:
- [ ] Consumer processes `opportunities.ingested` events from Redis Stream
- [ ] Matching logic correctly filters by CPV overlap, region, budget range, and deadline proximity
- [ ] Immediate alerts dispatched within 60 seconds of event arrival for matching users
- [ ] Alert logged in `notification.alert_log` with opportunity IDs and digest type
- [ ] Consumer uses Redis consumer group with manual ACK for reliable processing
- [ ] Non-matching opportunities are acknowledged without sending alerts
- [ ] Consumer handles malformed events gracefully (logs error, ACKs to avoid redelivery loop)
**Implementation Notes**: Use `XREADGROUP` with consumer group `notification-svc` and consumer name from hostname. Batch read up to 10 events per poll with 5-second block timeout. For matching, load all active immediate preferences into memory on startup and refresh every 60 seconds (or on cache invalidation event). This avoids per-event DB queries.

---

### S09.05: Daily & Weekly Digest Assembly
**Points**: 3 | **Type**: backend
**Description**: Implement Celery Beat periodic tasks for daily digest (runs at 07:00 UTC) and weekly digest (runs Monday 07:00 UTC). Each task: queries `client.alert_preferences` for active preferences with matching schedule, collects opportunities ingested since last digest (tracked via `notification.alert_log.sent_at`), matches each opportunity against user preferences using the same logic as S09.04, groups matched opportunities per user into a digest payload (sorted by relevance/deadline), and dispatches a single digest email per user via the SendGrid task. Skip users with zero matches. Record in `alert_log` with `digest_type = 'daily'` or `'weekly'` and all matched `opportunity_ids`.
**Acceptance Criteria**:
- [ ] Daily digest task triggers at 07:00 UTC and processes all daily-schedule preferences
- [ ] Weekly digest task triggers Monday 07:00 UTC and processes all weekly-schedule preferences
- [ ] Digest groups multiple matched opportunities into a single email per user
- [ ] Users with zero matches receive no email
- [ ] `alert_log` records all opportunity IDs included in each digest
- [ ] Digest window is correctly calculated from last sent digest per user (no duplicates, no gaps)
**Implementation Notes**: Use `celery.schedules.crontab` for scheduling. For the digest window, query `MAX(sent_at)` from `alert_log` per user per digest_type; fall back to 24h/7d ago if no prior digest. Batch DB queries: load all active preferences, then batch-fetch opportunities in the window, then match in Python to avoid N+1 queries.

---

### S09.06: SendGrid Email Delivery & Template Management
**Points**: 3 | **Type**: backend
**Description**: Implement a Celery task `send_email` on the `emails` queue that sends transactional email via the SendGrid v3 API. Accept parameters: recipient_email, template_type (enum), template_data (dict). Map template_type to SendGrid dynamic template IDs configured in environment variables. Template types: `alert_digest`, `trial_expiry_reminder`, `task_assigned`, `approval_requested`, `approval_decided`, `password_reset`, `welcome`. Log every send attempt in `notification.email_log` with sendgrid_message_id and initial status `sent`. Implement a SendGrid Event Webhook handler (in Client API as `POST /webhooks/sendgrid`) to update `email_log.status` on delivery/bounce/fail events. Retry on 429/5xx from SendGrid with exponential backoff.
**Acceptance Criteria**:
- [ ] `send_email` task dispatches email via SendGrid API for all 7 template types
- [ ] Template IDs are configurable via environment variables, not hardcoded
- [ ] Every send is logged in `notification.email_log` with sendgrid_message_id
- [ ] SendGrid webhook endpoint updates email_log status to delivered/bounced/failed
- [ ] 429 and 5xx responses from SendGrid trigger task retry with exponential backoff
- [ ] Invalid template_type raises descriptive error and does not retry
- [ ] Webhook endpoint validates SendGrid signature to prevent spoofing
**Implementation Notes**: Use `sendgrid` Python SDK. Configure `SENDGRID_API_KEY` and `SENDGRID_TEMPLATE_{TYPE}` env vars. Webhook signature validation uses the SendGrid Event Webhook verification library. Rate limit: SendGrid allows 600 requests/min on paid plans; add a `rate_limit` on the task if needed.

---

### S09.07: iCal Feed Generation (Client API)
**Points**: 3 | **Type**: backend
**Description**: Implement `GET /api/v1/calendar/ical/{token}` in the Client API that returns a valid `.ics` file containing VEVENT entries for all deadlines of opportunities tracked by the user. The `{token}` is a per-user opaque token (UUID stored in `client.calendar_connections` with `provider = 'ical'`) enabling unauthenticated subscription from calendar apps. Each VEVENT includes: summary (opportunity name), dtstart/dtend (deadline date, all-day event), description (opportunity type, budget, contracting authority), url (link to opportunity detail page), uid (deterministic from opportunity_id). Set `Content-Type: text/calendar` and `Content-Disposition: attachment; filename="eusolicit.ics"`. Regenerate on each request (no caching) to ensure freshness. Add endpoint `POST /api/v1/calendar/ical/generate-token` to create/rotate the iCal token.
**Acceptance Criteria**:
- [ ] Endpoint returns valid iCal file parseable by Google Calendar, Apple Calendar, Outlook
- [ ] Each tracked opportunity deadline appears as a VEVENT with correct summary, date, description, and URL
- [ ] Token-based access works without authentication (for calendar app subscription)
- [ ] Invalid or revoked token returns 404 (not 401, to avoid information leakage)
- [ ] Token generation endpoint creates new token and invalidates previous one
- [ ] Response headers set correct Content-Type and Content-Disposition
**Implementation Notes**: Use `icalendar` Python library. Generate deterministic UID as `{opportunity_id}@eusolicit.com` so calendar apps can detect updates. Query `client.tracked_opportunities` joined with `pipeline.opportunities` for deadline data. Token is UUID v4 stored in `calendar_connections` row with `provider = 'ical'`.

---

### S09.08: Google Calendar OAuth2 & Sync
**Points**: 5 | **Type**: backend
**Description**: Implement Google Calendar integration with two parts. (1) OAuth2 flow in Client API: `GET /api/v1/calendar/google/connect` redirects to Google consent screen requesting `calendar.events` scope. `GET /api/v1/calendar/google/callback` exchanges auth code for access/refresh tokens, encrypts them with Fernet (key from env), and stores in `client.calendar_connections` with `provider = 'google'`. (2) Celery periodic task in Notification Service (every 15 minutes): for each active Google calendar connection, fetch tracked opportunity deadlines, diff against `client.calendar_events` to determine creates/updates/deletes, and execute via Google Calendar API. Handle token refresh transparently using the stored refresh token. Update `calendar_events.external_event_id` and `synced_at`. Log sync results in `notification.sync_log`. Support disconnect via `DELETE /api/v1/calendar/google/disconnect` which revokes the token and removes the connection.
**Acceptance Criteria**:
- [ ] OAuth2 flow redirects to Google, handles callback, and stores encrypted tokens
- [ ] Periodic sync task creates calendar events for new tracked opportunity deadlines
- [ ] Sync task updates events when opportunity details change (deadline, name)
- [ ] Sync task deletes events when opportunity is untracked or cancelled
- [ ] Token refresh works transparently when access token expires
- [ ] Sync results logged in `notification.sync_log` with event counts
- [ ] Disconnect endpoint revokes Google token and deletes connection record
- [ ] Sync handles Google API errors (rate limits, invalid grant) gracefully with appropriate logging
**Implementation Notes**: Use `google-auth` and `google-api-python-client` libraries. Encrypt tokens with `cryptography.fernet.Fernet` using `CALENDAR_ENCRYPTION_KEY` env var. For diffing, maintain `calendar_events` as the source of truth: compare opportunity deadline set vs existing events. Use batch API for efficiency when syncing multiple events. Set `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` from env.

---

### S09.09: Microsoft Outlook OAuth2 & Sync
**Points**: 4 | **Type**: backend
**Description**: Implement Microsoft Outlook calendar integration mirroring the Google Calendar flow (S09.08) but using Microsoft Graph API. (1) OAuth2 flow in Client API: `GET /api/v1/calendar/microsoft/connect` redirects to Microsoft identity platform with `Calendars.ReadWrite` scope. Callback exchanges code for tokens, encrypts and stores with `provider = 'microsoft'`. (2) Celery periodic task (shared schedule with Google sync, every 15 min): for each active Microsoft connection, sync tracked opportunity deadlines as Outlook calendar events via `POST/PATCH/DELETE /me/events` on Microsoft Graph. Handle token refresh via the Microsoft refresh token flow. Log in `notification.sync_log`. Support disconnect via `DELETE /api/v1/calendar/microsoft/disconnect`.
**Acceptance Criteria**:
- [ ] OAuth2 flow redirects to Microsoft, handles callback, and stores encrypted tokens
- [ ] Periodic sync creates/updates/deletes Outlook calendar events correctly
- [ ] Token refresh works when access token expires
- [ ] Sync results logged in `notification.sync_log`
- [ ] Disconnect revokes token and removes connection
- [ ] Microsoft Graph API errors handled gracefully (throttling via Retry-After header, invalid grant)
**Implementation Notes**: Use `msal` (Microsoft Authentication Library) for OAuth2 and `httpx` for Graph API calls. Microsoft Graph base URL: `https://graph.microsoft.com/v1.0`. Event create: `POST /me/events`, update: `PATCH /me/events/{id}`, delete: `DELETE /me/events/{id}`. Respect `Retry-After` header on 429 responses. Set `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`, `MICROSOFT_TENANT_ID` from env.

---

### S09.10: Task & Approval Notification Consumers
**Points**: 3 | **Type**: backend
**Description**: Implement Redis Stream consumers in the Notification Service for task and approval lifecycle events. (1) Task consumer: listens to `eu-solicit:tasks` stream for `task.assigned` and `task.overdue` events. On `task.assigned`, sends email to assigned user with task details, opportunity context, and link to task page. On `task.overdue`, sends reminder email to assigned user and CC to team lead. (2) Approval consumer: listens to `eu-solicit:approvals` stream for `approval.requested` and `approval.decided` events. On `approval.requested`, sends email to each required approver with proposal summary and approve/reject link. On `approval.decided`, sends email to proposal owner with decision outcome. All emails dispatched via the `send_email` task (S09.06).
**Acceptance Criteria**:
- [ ] Task consumer processes `task.assigned` events and sends email to assignee
- [ ] Task consumer processes `task.overdue` events and sends email to assignee and team lead
- [ ] Approval consumer processes `approval.requested` and sends email to all required approvers
- [ ] Approval consumer processes `approval.decided` and sends email to proposal owner
- [ ] All emails use correct SendGrid templates with populated dynamic data
- [ ] Consumers use Redis consumer groups with manual ACK
- [ ] Consumers handle missing user data gracefully (log warning, ACK event)
**Implementation Notes**: Each consumer runs as a separate Celery task started on worker boot using `worker_ready` signal. Use shared consumer group logic from S09.04. Look up user email from `client.users` table (read-only access). For approval links, generate a signed URL with expiry for one-click approve/reject.

---

### S09.11: Trial Expiry & Stripe Usage Sync
**Points**: 3 | **Type**: backend
**Description**: Implement two consumers/tasks: (1) Trial expiry consumer: listens to `eu-solicit:subscriptions` stream for `subscription.trial_expiring` events (published 3 days before trial end). Sends trial expiry reminder email via `send_email` task with days remaining, feature summary of paid tiers, and upgrade CTA link. (2) Stripe usage reporting task: Celery periodic task (daily at 02:00 UTC) reads Redis usage counters (`user:{user_id}:usage:{feature}:{billing_period}`) for all active subscriptions, aggregates per Stripe subscription item, and reports to Stripe Usage Records API via `stripe.SubscriptionItem.create_usage_record()`. Log success/failure per subscription. Clear reported counters after successful reporting.
**Acceptance Criteria**:
- [ ] Trial expiry consumer sends reminder email on `subscription.trial_expiring` event
- [ ] Email includes days remaining, tier comparison, and upgrade link
- [ ] Stripe usage task runs daily at 02:00 UTC
- [ ] Task reads Redis usage counters and reports to Stripe Usage Records API
- [ ] Successful reports are logged; failed reports are retried on next run
- [ ] Redis counters are cleared only after successful Stripe reporting
- [ ] Task handles Stripe API errors (rate limit, invalid subscription) gracefully
**Implementation Notes**: Use `stripe` Python SDK. Map Redis usage keys to Stripe subscription items via `client.subscriptions` table (read-only). Use `action='set'` on usage records (not increment) since we report the total. For counter clearing, use Redis `GETDEL` or `GET` then `DEL` in a pipeline to avoid race conditions. Only clear the specific billing period key that was reported.

---

### S09.12: Alert Preferences Frontend Page
**Points**: 3 | **Type**: frontend
**Description**: Build the alert preferences settings page in the Client API frontend. UI components: CPV sector multi-select dropdown (searchable, with CPV code + description), EU region checkboxes (grouped by geographic area: Western, Eastern, Northern, Southern Europe), budget range dual slider with min/max numeric inputs, deadline proximity slider (1-90 days), schedule radio buttons (Immediate / Daily digest / Weekly digest), enable/disable toggle per preference, create new preference button (max 5, show count), edit and delete actions per preference card. Page loads existing preferences via `GET /api/v1/alerts/preferences` and renders as cards. Form validation mirrors backend rules. Show toast on save success/failure.
**Acceptance Criteria**:
- [ ] Page displays existing alert preferences as editable cards
- [ ] CPV multi-select supports search and displays code + description
- [ ] Region checkboxes grouped by geographic area with select-all per group
- [ ] Budget range and deadline proximity inputs validate correctly (min < max, 1-90 days)
- [ ] Schedule selector shows Immediate / Daily digest / Weekly digest options
- [ ] Enable/disable toggle calls PATCH endpoint without full form submission
- [ ] Create new preference respects 5-preference limit with clear messaging
- [ ] Delete preference shows confirmation dialog before calling DELETE endpoint
- [ ] Form validation errors display inline; API errors display as toast
**Implementation Notes**: Use the design system components from E03. CPV code list can be a static JSON loaded at build time (codes rarely change). Region groups: Western (FR, DE, NL, BE, LU, AT), Eastern (PL, CZ, SK, HU, RO, BG), Northern (SE, DK, FI, EE, LV, LT), Southern (IT, ES, PT, GR, HR, SI, CY, MT), Other (IE). Budget slider uses logarithmic scale for better UX across wide range.

---

### S09.13: Calendar Connection Management Frontend Page
**Points**: 3 | **Type**: frontend
**Description**: Build the calendar connection management page in the Client API frontend. Sections: (1) Google Calendar: "Connect Google Calendar" OAuth button (redirects to `/api/v1/calendar/google/connect`), shows connection status (connected/disconnected), last synced timestamp, sync event count, "Disconnect" button with confirmation. (2) Microsoft Outlook: identical UI to Google section but for Outlook OAuth. (3) iCal Feed: "Generate iCal URL" button, copyable URL field with copy-to-clipboard button, instructions text for subscribing in various calendar apps, "Regenerate URL" button with warning that old URL will stop working. All sections show appropriate empty/connected/error states. Poll sync status every 30 seconds when page is open.
**Acceptance Criteria**:
- [ ] Google Calendar connect button initiates OAuth flow and returns to page with connected state
- [ ] Microsoft Outlook connect button initiates OAuth flow and returns to page with connected state
- [ ] Connected calendars show last synced time, events synced count, and disconnect button
- [ ] Disconnect shows confirmation dialog and removes connection on confirm
- [ ] iCal URL generation creates and displays a copyable subscription URL
- [ ] Copy button copies URL to clipboard with visual feedback
- [ ] Regenerate URL warns user and invalidates previous URL
- [ ] Sync status polls every 30 seconds and updates UI without full page reload
- [ ] Error states display clearly (e.g., token expired, sync failed) with retry/reconnect actions
**Implementation Notes**: Use the design system card components from E03. OAuth redirect returns to this page with a `?connected=google` or `?connected=microsoft` query param to trigger a success toast. iCal URL format: `https://app.eusolicit.com/api/v1/calendar/ical/{token}`. For polling, use `setInterval` with cleanup on unmount; consider switching to WebSocket in future iteration.

---

### S09.14: Scheduled Report Generation Task
**Points**: 2 | **Type**: backend
**Description**: Implement a Celery periodic task for generating and emailing scheduled analytics reports (ties into E12 Analytics). Task runs on a configurable schedule per company (default: weekly Monday 08:00 UTC). Queries analytics data (opportunity pipeline summary, team activity, success rates) for the reporting period. Generates report as PDF using a template engine (WeasyPrint or ReportLab). Attaches PDF to email and dispatches via `send_email` task to company admin users. Store generated report metadata (report_type, period, generated_at, file_path) for future download from the platform. This is a foundational story; full analytics content depends on E12.
**Acceptance Criteria**:
- [ ] Celery periodic task generates reports on configured schedule per company
- [ ] Report contains placeholder analytics sections (to be populated by E12)
- [ ] PDF generation produces valid, styled document with EU Solicit branding
- [ ] Report emailed to all admin users of the company
- [ ] Report metadata stored for future retrieval
- [ ] Task handles generation errors gracefully and logs failures
**Implementation Notes**: Use WeasyPrint for HTML-to-PDF conversion (supports CSS styling). Report HTML template uses Jinja2. Store generated PDFs in S3 with path `reports/{company_id}/{year}/{month}/report-{date}.pdf`. This story creates the pipeline; E12 will fill in the actual analytics queries and report sections. Keep report template modular so sections can be added independently.
