# Story: 9-6-sendgrid-email-delivery-template-management

## Epic
Epic 09: Notifications, Alerts & Calendar

## Metadata
- **Points**: 3
- **Type**: backend
- **Module**: Notification Service

## Description
Implement a Celery task `send_email` on the `emails` queue that sends transactional email via the SendGrid v3 API. Accept parameters: recipient_email, template_type (enum), template_data (dict). Map template_type to SendGrid dynamic template IDs configured in environment variables. Template types: `alert_digest`, `trial_expiry_reminder`, `task_assigned`, `approval_requested`, `approval_decided`, `password_reset`, `welcome`. Log every send attempt in `notification.email_log` with sendgrid_message_id and initial status `sent`. Implement a SendGrid Event Webhook handler (in Client API as `POST /webhooks/sendgrid`) to update `email_log.status` on delivery/bounce/fail events. Retry on 429/5xx from SendGrid with exponential backoff.

## Acceptance Criteria
- [ ] `send_email` task dispatches email via SendGrid API for all 7 template types
- [ ] Template IDs are configurable via environment variables, not hardcoded
- [ ] Every send is logged in `notification.email_log` with sendgrid_message_id
- [ ] SendGrid webhook endpoint updates email_log status to delivered/bounced/failed
- [ ] 429 and 5xx responses from SendGrid trigger task retry with exponential backoff
- [ ] Invalid template_type raises descriptive error and does not retry
- [ ] Webhook endpoint validates SendGrid signature to prevent spoofing

## Implementation Notes
- Use `sendgrid` Python SDK. Configure `SENDGRID_API_KEY` and `SENDGRID_TEMPLATE_{TYPE}` env vars.
- Webhook signature validation uses the SendGrid Event Webhook verification library.
- Rate limit: SendGrid allows 600 requests/min on paid plans; add a `rate_limit` on the task if needed.

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
- **REVIEW: Changes Requested**
- **DEVIATION**: The story instructs to implement the SendGrid Event Webhook handler in the Client API and directly update `email_log.status`. The `email_log` table belongs to the Notification Service's schema. Direct cross-service database access violates the architecture document's mandate of "per-service DB roles with enforced access boundaries".
- **DEVIATION_TYPE**: CONTRADICTORY_SPEC
- **DEVIATION_SEVERITY**: blocking
- **Action**: Move the webhook handler to the Notification Service, or have the Client API publish a domain event to a message broker for the Notification Service to consume and update its own database.

## Status
changes_requested

## Known Deviations

### Detected by `3-code-review` at 2026-04-19T12:07:03Z

- The story specifies implementing the SendGrid Event Webhook handler in the Client API while directly updating `notification.email_log.status` via SQL. The `email_log` table belongs to the Notification Service's schema, which violates the architecture's mandate for per-service DB roles and strict schema isolation boundaries. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
