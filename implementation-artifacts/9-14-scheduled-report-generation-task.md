# Story 9.14: Scheduled Report Generation Task (Admin Fan-Out)

Status: complete

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 09: Notifications, Alerts & Calendar

## Metadata
- **Story Key:** 9-14-scheduled-report-generation-task
- **Points:** 2
- **Type:** backend
- **Module:** Notification Service (`notification.tasks.scheduled_report_delivery` extension, `notification.config`, Celery Beat `beat_schedule.py`)
- **Priority:** P2 (Beta milestone — closes the last open E09 AC; reconciles with E12's delivered pipeline)
- **Depends On:** Story 12.9 (report generation engine, `generate_report` task), Story 12.10 (scheduled delivery infrastructure, `check_scheduled_reports` + `send_scheduled_report_email` + `client.report_schedules` + `client.report_jobs`), Story 9.6 (`send_email` SendGrid task), Story 9.11 (`_resolve_company_admins` pattern in `SubscriptionConsumer`)

## Story

As **any company admin on EU Solicit whose team has NOT explicitly configured a custom `report_schedules` row**,
I want **to automatically receive a weekly analytics report (Monday 08:00 UTC default) emailed to every active admin of the company without requiring any setup**,
so that **every paying company gets a baseline weekly business-intelligence touchpoint out of the box, and the Epic 09 AC "Report emailed to all admin users of the company" is satisfied on top of Story 12.10's explicit-recipients pipeline**.

## Description

This is the closing story of Epic 09. When it was originally authored, S09.14 was scoped as "foundational — creates the pipeline; E12 will fill in analytics." In the meantime Epic 12 completed **first**, delivering a full-fledged scheduled report delivery pipeline under Stories 12.9 and 12.10:

- `client.report_jobs` + `client.report_schedules` Alembic migrations (013, 014)
- `notification.tasks.report_generation.generate_report` — async Celery task that renders PDF/DOCX via the shared `eusolicit_common.document_generation` module, uploads to S3, and returns a 24-hour signed URL
- `notification.tasks.scheduled_report_delivery.check_scheduled_reports` + `send_scheduled_report_email` — Beat-driven daily check (06:00 UTC) that dispatches `celery_chain(generate_report → send_scheduled_report_email)` for every active schedule whose `frequency` matches today's weekday/day-of-month
- 4 report templates (pipeline_summary, bid_performance_summary, team_activity, custom_date_range)
- Frontend modal + reports list + schedule manager pages

As a result, S09.14's original deliverables are **~90% already in production**. What remains is the **admin fan-out** delivery mode that Epic 09's AC explicitly demands but Story 12.10 does NOT cover:

> Epic 09 AC (S09.14): "Attaches PDF to email and dispatches via `send_email` task to company admin users."
> Story 12.10 AC12: Sends via direct `SendGridAPIClient` to `schedule_row.recipients` (explicit email list owned by the admin who created the schedule).

Two behavioural gaps follow from that contract difference:

1. **Recipient resolution.** S12.10 ships emails to whatever list of addresses a human typed into the "Create Schedule" form. S09.14 requires dispatch to **every active admin of the company**, auto-resolved via `client.company_memberships` + `client.users` (the same pattern Story 9.11 established for trial-expiry reminders).
2. **Dispatch path.** S12.10 uses `SendGridAPIClient` directly with a base64 attachment in `send_scheduled_report_email`. Epic 09 canonically routes all transactional email through the Story 9.6 `send_email` Celery task (on the `emails` queue) for observability (`notification.email_log`), retry policy, and template-ID management consistency. S09.14 aligns the scheduled-report admin fan-out to that canonical path by adding a new SendGrid dynamic template `scheduled_report` and having the new admin-fan-out task call `send_email.delay(...)` per admin with a signed download URL in `template_data` (NOT the binary PDF — email body carries the link, and the 24-hour signed URL from `generate_report` delivers the file).

This story therefore delivers **one thin additive task** (`send_scheduled_report_to_admins`) plus a **schedule-row convention** that opts a schedule into admin fan-out when its `recipients` column is empty — and a **default-schedule auto-provisioner** that seeds a weekly-Monday-08:00-UTC `pipeline_summary` row per company that has none. Everything else (PDF rendering, S3 upload, job tracking, Beat loop, weekly/monthly due-date logic) is **reused unchanged** from Stories 12.9 and 12.10.

The scope is deliberately tight (2 points) because duplicating E12 infrastructure would be wasteful and error-prone; the critical insight for the implementing developer is that this story is **reconciliation + extension, not reimplementation**.

## Acceptance Criteria

1. [ ] **AC1 — New SendGrid template slot.** `notification.config.Settings` gains `sendgrid_template_scheduled_report: str = Field(default="d-scheduled-report", env="SENDGRID_TEMPLATE_SCHEDULED_REPORT")`. A unit test confirms `send_email(template_type="scheduled_report", template_data={...})` resolves the template ID via `getattr(settings, f"sendgrid_template_scheduled_report")` without raising `ValueError` (the AC6 guard from Story 9.6 remains intact for unknown template types).

2. [ ] **AC2 — `send_scheduled_report_to_admins` Celery task.** A NEW task `notification.tasks.scheduled_report_delivery.send_scheduled_report_to_admins(self, job_id: str, schedule_id: str)` is added alongside the existing `send_scheduled_report_email`. Shape:

    ```python
    @celery.task(
        name="notification.tasks.scheduled_report_delivery.send_scheduled_report_to_admins",
        bind=True,
        max_retries=3,
        default_retry_delay=60,
        acks_late=True,
        queue="emails",
    )
    def send_scheduled_report_to_admins(self, job_id: str, schedule_id: str) -> None: ...
    ```

    Behaviour, in order:
    - Load `report_jobs` row for `job_id`. If not found → log warning + return. If `status != "complete"` → raise `ValueError("job {job_id} not complete — status={status}")` to trigger Celery retry (same retry semantics as the existing `send_scheduled_report_email`; the `generate_report` chain-predecessor may still be running).
    - Load `report_schedules` row for `schedule_id`. If not found or `is_active=False` → log warning + return (schedule deleted while queued).
    - Resolve active admins for `schedule.company_id` via a SELECT over `client.company_memberships` joined with `client.users` where `role='admin'`, `accepted_at IS NOT NULL`, `users.is_active=TRUE`. If the resolved list is empty → log `structlog.warning("scheduled_report.no_admins", company_id=..., schedule_id=..., job_id=...)` and return (no error, no retry — consistent with Story 9.11 AC4).
    - For each admin: call `send_email.delay(recipient_email=admin.email, template_type="scheduled_report", template_data={"admin_name": admin.full_name or admin.email, "company_name": <resolved>, "report_type_label": REPORT_TYPE_LABELS[job.report_type], "generated_at": job.completed_at.isoformat(), "download_url": job.download_url, "download_url_expires_at": job.url_expires_at.isoformat() if job.url_expires_at else None, "format": job.format, "job_id": str(job.id)})`. Wrap the call in a per-admin `try/except` so one failed dispatch does not abort the fan-out.
    - After the loop completes, log `structlog.info("scheduled_report.admin_fanout_dispatched", company_id=..., schedule_id=..., job_id=..., admin_count=len(admins), correlation_id=...)`.
    - On unexpected exceptions in the outer try block: `self.retry(exc=e)`. On `MaxRetriesExceededError`: log `structlog.error("scheduled_report.admin_fanout_max_retries_exceeded", ...)` and suppress (same tail behaviour as Story 12.10 AC12).

3. [ ] **AC3 — Schedule-row convention: empty `recipients` → admin fan-out.** `check_scheduled_reports` (Story 12.10 AC11) is extended so that for every schedule loaded from `client.report_schedules`:
    - If `len(schedule.recipients) > 0` → dispatch `celery_chain(generate_report.si(...), send_scheduled_report_email.si(job_id, schedule_id))` (UNCHANGED — S12.10 path).
    - Else (empty recipients list) → dispatch `celery_chain(generate_report.si(...), send_scheduled_report_to_admins.si(job_id, schedule_id))` (NEW — S09.14 path).
    - Log the routing decision at `info` level (`check_scheduled_reports.routed`, with `job_id`, `schedule_id`, `recipients_count`, `fanout_mode ∈ {"explicit", "admin"}`).
    - The S12.10 migration 014 already allows `recipients DEFAULT '{}'`, so no DB migration is needed — an empty array is a valid stored value today. Verify this is still true (AC12 sanity-guard test).

4. [ ] **AC4 — Default weekly schedule auto-provisioner.** A NEW Celery task `notification.tasks.scheduled_report_delivery.ensure_default_company_schedules() -> None` runs daily at 05:00 UTC (before `check_scheduled_reports`'s 06:00 UTC trigger so a newly-provisioned schedule is picked up the same day when it's a Monday). Behaviour:
    - SELECT every `client.companies.id` where there is NO existing row in `client.report_schedules` owned by any admin of that company.
    - For each such company, INSERT one `report_schedules` row:
        - `company_id` = company's id
        - `created_by_id` = the company's oldest active admin's user id (ORDER BY `company_memberships.created_at ASC` LIMIT 1). If no active admin exists, skip that company and log `warning scheduled_report.default_schedule_skip_no_admin`.
        - `report_type = "pipeline_summary"`
        - `format = "pdf"`
        - `frequency = "weekly"`
        - `recipients = '{}'` (empty — triggers the admin-fanout path from AC3)
        - `is_active = TRUE`
    - Log `info scheduled_report.default_schedule_seeded` per seeded row; log `info scheduled_report.ensure_default_schedules_complete` with the summary counts at the end.
    - Idempotent by construction (only seeds where no schedule exists). Safe to re-run daily.
    - Register in `BEAT_SCHEDULE` as `"ensure-default-company-schedules-daily"` with `crontab(hour=5, minute=0)`.

5. [ ] **AC5 — Weekly schedule still fires on Monday via S12.10's unchanged `check_scheduled_reports`.** The existing AC11 logic (`today.weekday() == 0 → frequencies_due.append("weekly")`) already covers the Monday fan-out. No changes to the frequency computation — only the routing branch inside the per-schedule loop (AC3) changes. A unit test with `freezegun` frozen to a Monday and a seeded empty-recipients schedule asserts the chain dispatches `send_scheduled_report_to_admins`, NOT `send_scheduled_report_email`.

6. [ ] **AC6 — Schedule time alignment (documented deviation).** Epic 09 AC specifies "weekly Monday 08:00 UTC" as the default. Story 12.10 deployed `check_scheduled_reports` at 06:00 UTC (informed by `generate_report` runtime estimates so emails reach admins by ~08:00 UTC inbox time). We **keep the 06:00 UTC Beat trigger unchanged** and document this in Task 5 as an accepted variance. No change to `beat_schedule.py` for the existing `"check-scheduled-reports-daily"` row.

7. [ ] **AC7 — Report metadata persistence satisfies "future download" AC.** Epic 09 AC says "Store generated report metadata (report_type, period, generated_at, file_path) for future download from the platform." Verify that `client.report_jobs` already stores `report_type`, `date_from`/`date_to`, `completed_at`, `s3_key`, `download_url`, `url_expires_at` — every field Epic 09 asks for. Add a regression-guard test `test_report_job_row_has_all_epic9_metadata_fields` that introspects the `sa.Table` definition and asserts all six columns exist. No schema change.

8. [ ] **AC8 — Retry semantics parity with Story 12.10.** `send_scheduled_report_to_admins` uses the same decorator shape as `send_scheduled_report_email`: `bind=True, max_retries=3, default_retry_delay=60, acks_late=True, queue="emails"`. Retries trigger ONLY on `ValueError("job not complete")` (chain predecessor still running) or unexpected `Exception` from the DB/session lookups. Per-admin `send_email.delay()` failures must NOT trigger outer retry (they are handled by Story 9.6's `send_email` retry policy).

9. [ ] **AC9 — Tenant isolation (E12-R-001 parity).** `send_scheduled_report_to_admins` queries `company_memberships` strictly on `schedule_row.company_id` (re-read from DB, never trusted from Celery kwargs). A cross-tenant negative test seeds two companies each with admins; asserts admins of Company B NEVER receive an email for Company A's scheduled report.

10. [ ] **AC10 — Observability.** Every task execution emits structured log events: `scheduled_report.admin_fanout_started` (start), `scheduled_report.admin_fanout_dispatched` (success), `scheduled_report.no_admins` (zero admins found), `scheduled_report.admin_dispatch_failed` (per-admin exception caught), `scheduled_report.admin_fanout_max_retries_exceeded` (outer retry exhaustion). All events include `company_id`, `schedule_id`, `job_id`, and — where available — `correlation_id` (pulled from the chain predecessor's task context via `celery.app.task.Task.request.correlation_id` when present, else `None`).

11. [ ] **AC11 — Observability of the default-schedule seeder.** `ensure_default_company_schedules` emits `scheduled_report.default_schedule_seeded` (per insert, with `company_id`, `schedule_id`, `admin_user_id`), `scheduled_report.default_schedule_skip_no_admin` (per skipped company), and `scheduled_report.ensure_default_schedules_complete` (summary: `companies_scanned`, `schedules_seeded`, `companies_skipped_no_admin`).

12. [ ] **AC12 — Bootstrap sanity guards.** Two tests assert:
    - `"ensure-default-company-schedules-daily" in BEAT_SCHEDULE and BEAT_SCHEDULE["ensure-default-company-schedules-daily"]["schedule"] == crontab(hour=5, minute=0)`.
    - The `client.report_schedules` SQLAlchemy Core table definition shows `recipients` is `sa.ARRAY(sa.Text())` with no `NOT NULL` violation when inserting `'{}'` (empty array), proving the S09.14 fan-out convention is storage-safe.

13. [ ] **AC13 — Security: no download URL in persistent logs at INFO level.** The 24-hour presigned S3 URL is sensitive (grants report download without auth). Log it at `debug` level only — never `info` — in both `send_scheduled_report_to_admins` and any test helpers. A structlog-capture unit test runs the fan-out flow and asserts that for captured log records at levels `info`+, `download_url` is either absent OR replaced with a placeholder (`"<redacted>"`). The URL IS passed to `send_email.delay` via `template_data` (SendGrid needs it to render the CTA button) — that in-transit carry is acceptable; only log redaction is in scope.

14. [ ] **AC14 — Documentation.** `services/notification/README.md` (added in Story 9.11) gains a new section "Scheduled Reports — Admin Fan-Out Mode (S09.14)" explaining:
    - The `recipients = '{}'` convention → admin fan-out
    - The default-schedule seeder task and its 05:00 UTC schedule
    - How a company-admin user can opt out (DELETE the default row, or set `is_active=FALSE`, or replace `recipients` with an explicit list to switch back to S12.10 behaviour).

## Design Constraints

- **No new DB migration.** Reuse `client.report_schedules` (migration 014, Story 12.10) and `client.report_jobs` (migration 013, Story 12.9) unchanged. The `recipients = '{}'` convention is storage-compatible with the existing schema.
- **No new queue.** `send_scheduled_report_to_admins` lives on the `emails` queue (same as `send_email`), because its per-admin body is exactly one `send_email.delay(...)` dispatch per admin. The orchestration cost is trivial.
- **Reuse, do NOT rewrite, `generate_report`.** AC2's `send_scheduled_report_to_admins` consumes the EXISTING `report_jobs` row populated by the chain predecessor. Never re-render or re-upload the PDF.
- **Admin resolution MUST use the same query shape as `SubscriptionConsumer._resolve_company_admins` (Story 9.11).** Use `notification.models.user.User` and `notification.models.company_membership.CompanyMembership`; filter on `CompanyRole.admin`, `accepted_at IS NOT NULL`, `User.is_active`. Do NOT introduce a parallel query path. Code-level reuse is encouraged — factor the shared query into a module-level helper in `notification/models/company_membership.py` (new module-level function `async def resolve_active_admins(session, company_id: UUID) -> list[User]`) and update `SubscriptionConsumer` to call it too (safe refactor; one caller today, two after this story).
- **`send_scheduled_report_to_admins` is a Celery task (sync), not an async consumer.** Unlike S09.11's `SubscriptionConsumer`, this task runs under Celery worker threads, not the FastAPI asyncio loop. Use the existing synchronous `_get_engine()` + `engine.connect()` pattern from `scheduled_report_delivery.py`; do NOT introduce `asyncio.run` inside the task body. For the admin-resolving helper, provide BOTH sync and async variants OR keep them separate modules — the sync variant in `notification.tasks.scheduled_report_delivery._resolve_active_admins_sync` is acceptable and recommended to avoid running an event loop inside a Celery worker.
- **Template ID management.** `sendgrid_template_scheduled_report` defaults to `"d-scheduled-report"`. Operations team must provision the actual SendGrid dynamic template before first production run. Flag this in Task 5 as a pre-deploy checklist item but do NOT block merge — test env uses the stub ID and SendGrid calls are mocked.
- **Download URL expiry: 24 hours is authoritative.** `generate_report` (S12.9) sets `url_expires_at = created_at + 24h`. The email CTA "valid for 24 hours" wording must match; template rendering of `download_url_expires_at` into the user-visible timestamp is the SendGrid template's responsibility.
- **Do NOT attach the PDF binary to the S09.14 email.** S12.10 attaches base64 bytes via `SendGridAPIClient`. S09.14 uses the `send_email` task which does NOT support binary attachments in its current Story 9.6 API (template-only dispatch). The download URL serves as the delivery mechanism. Rationale: (a) keeps `send_email` API surface minimal, (b) avoids email-size bloat × N admins, (c) the presigned URL is already required for the reports-list page (S12.10 AC17) so end users are already familiar with it.
- **`correlation_id` propagation.** When `check_scheduled_reports` dispatches the chain, pass a newly-minted `correlation_id=str(uuid4())` via the chain header so both `generate_report` and `send_scheduled_report_to_admins` log under the same correlation ID. This is the same pattern `SubscriptionConsumer` uses (Story 9.11 AC10 carry-over). If this requires touching `check_scheduled_reports` — document as an in-scope minor enhancement, not scope creep.
- **Seeder concurrency.** `ensure_default_company_schedules` runs once daily at 05:00 UTC. It is the sole writer of default rows; no race with end-user schedule creation is possible within a single Beat tick. If a user deletes their default row at 05:01, the next tick (24h later) re-seeds it. Document this cadence in AC14's README section so users understand the recurrence.
- **S12.10 admin UI does NOT need changes.** The reports schedule manager frontend (S12.10 AC20–AC24) lists schedules; an auto-seeded row appears in the list like any other and the admin can edit or delete it through the existing UI. No new frontend work in scope.

## Tasks / Subtasks

- [ ] **Task 1: Add `sendgrid_template_scheduled_report` config + admin-resolver refactor. (AC1, reuse)**
  - [ ] Add `sendgrid_template_scheduled_report: str = Field(default="d-scheduled-report", env="SENDGRID_TEMPLATE_SCHEDULED_REPORT")` to `notification.config.Settings` alongside the other 8 template slots.
  - [ ] Factor the admin resolver into `notification/models/company_membership.py::resolve_active_admins_async(session, company_id: UUID) -> list[User]` (async flavour for `SubscriptionConsumer`) and `notification/models/company_membership.py::resolve_active_admins_sync(session, company_id: UUID) -> list[User]` (sync flavour for Celery tasks). Both return `List[User]` of admins matching `accepted_at IS NOT NULL AND role = 'admin' AND users.is_active = TRUE`.
  - [ ] Update `SubscriptionConsumer._resolve_company_admins` to delegate to `resolve_active_admins_async` (pure refactor; no behaviour change). Keep the `ValueError(company_id)` → `[]` guard at the call site.
  - [ ] Unit test `test_resolve_active_admins_sync_and_async_match` asserts both helpers return equivalent user lists for the same fixture company.

- [ ] **Task 2: Implement `send_scheduled_report_to_admins` Celery task. (AC2, AC8, AC9, AC10, AC13)**
  - [ ] Add the task to `services/notification/src/notification/tasks/scheduled_report_delivery.py` beneath the existing `send_scheduled_report_email`.
  - [ ] Signature and decorator per AC2. Reuse `_get_engine`, `_report_jobs`, `_report_schedules`, `REPORT_TYPE_LABELS` from the same module.
  - [ ] Resolve company name via a new inline `_companies` Core table definition OR reuse `report_generation._companies` (import cost). Prefer reusing `report_generation._companies` and import only at function scope to avoid module-load order issues.
  - [ ] Inside the task: load job row → verify status=complete (else raise `ValueError` for retry) → load schedule row → call `resolve_active_admins_sync(conn, schedule.company_id)` → iterate and dispatch `send_email.delay(...)` with template_data per AC2 → log per AC10.
  - [ ] For each per-admin dispatch, wrap in `try/except Exception as e: logger.error("scheduled_report.admin_dispatch_failed", admin_user_id=..., error=str(e))` — swallow and continue.
  - [ ] Outer exception handler: `self.retry(exc=e)` with `MaxRetriesExceededError` → log+suppress.
  - [ ] Ensure `download_url` is never logged at `info`+ (AC13). Pass it through `template_data` only.
  - [ ] Unit tests in `services/notification/tests/worker/test_send_scheduled_report_to_admins.py`:
    - [ ] `test_job_not_complete_raises_for_retry`
    - [ ] `test_schedule_not_found_warns_and_returns`
    - [ ] `test_schedule_inactive_warns_and_returns`
    - [ ] `test_fans_out_to_all_admins` (3 admins; assert `send_email.delay` called 3× with correct `template_type`, `recipient_email`, and `template_data` keys)
    - [ ] `test_per_admin_failure_does_not_abort_fanout` (admin[1] raises; admins[0] and admins[2] still dispatch)
    - [ ] `test_zero_admins_logs_warning_and_returns` (AC2 no-admin branch; no retry)
    - [ ] `test_download_url_not_in_info_log` (structlog capture per AC13)
    - [ ] `test_cross_tenant_isolation` (Company A event does not dispatch to Company B admins — AC9)

- [ ] **Task 3: Extend `check_scheduled_reports` to route by `recipients` emptiness. (AC3, AC5)**
  - [ ] In `check_scheduled_reports`, after the `for schedule in schedules` loop creates the `report_jobs` row, branch on `len(schedule.recipients) > 0`:
    - Explicit: dispatch existing chain (unchanged).
    - Empty: dispatch `celery_chain(celery_signature("notification.tasks.report_generation.generate_report", args=[...], immutable=True), send_scheduled_report_to_admins.si(job_id_str, schedule_id_str)).apply_async()`.
  - [ ] Log `check_scheduled_reports.routed` with `fanout_mode ∈ {"explicit","admin"}` and counts.
  - [ ] Pass a `correlation_id = str(uuid4())` through the chain's task options (`apply_async(headers={"correlation_id": correlation_id})` or via `celery.canvas.Signature.set(headers=...)`).
  - [ ] Unit tests in `services/notification/tests/worker/test_check_scheduled_reports_routing.py`:
    - [ ] `test_monday_weekly_schedule_empty_recipients_routes_admin_fanout` — freezegun Monday; seed 1 schedule with `recipients=[]`; assert `send_scheduled_report_to_admins.si` appears in the dispatched chain, NOT `send_scheduled_report_email.si`.
    - [ ] `test_monday_weekly_schedule_explicit_recipients_routes_direct` — freezegun Monday; seed `recipients=["a@b.com"]`; assert `send_scheduled_report_email.si` dispatched (legacy path preserved).
    - [ ] `test_non_monday_weekly_schedule_does_nothing` — freezegun Tuesday; assert no chain dispatched.
    - [ ] `test_mixed_schedules_route_independently` — 2 schedules in one Beat tick (one empty, one explicit); both chains dispatched with correct downstream task.

- [ ] **Task 4: Implement default-schedule auto-provisioner. (AC4, AC11)**
  - [ ] Add `ensure_default_company_schedules` task to `scheduled_report_delivery.py`:
    ```python
    @celery.task(name="notification.tasks.scheduled_report_delivery.ensure_default_company_schedules", bind=False)
    def ensure_default_company_schedules() -> None: ...
    ```
  - [ ] Query: `SELECT companies.id FROM client.companies LEFT JOIN client.report_schedules ON companies.id = report_schedules.company_id WHERE report_schedules.id IS NULL`.
  - [ ] For each company, find the oldest active admin via `SELECT users.id FROM client.company_memberships JOIN client.users ON users.id = company_memberships.user_id WHERE company_memberships.company_id = :cid AND company_memberships.role = 'admin' AND company_memberships.accepted_at IS NOT NULL AND users.is_active = TRUE ORDER BY company_memberships.created_at ASC LIMIT 1`.
  - [ ] If no admin found → log `scheduled_report.default_schedule_skip_no_admin` + continue.
  - [ ] Insert the default row; log `scheduled_report.default_schedule_seeded`.
  - [ ] Emit summary log `scheduled_report.ensure_default_schedules_complete` at end.
  - [ ] Register in `BEAT_SCHEDULE` with crontab(hour=5, minute=0).
  - [ ] Add `_companies` inline Core Table definition (id, name) if not already imported.
  - [ ] Unit tests in `services/notification/tests/worker/test_ensure_default_schedules.py`:
    - [ ] `test_seeds_default_for_company_with_no_schedule`
    - [ ] `test_skips_company_with_existing_schedule`
    - [ ] `test_skips_company_with_no_active_admin`
    - [ ] `test_seeded_row_has_pipeline_summary_pdf_weekly_empty_recipients_active`
    - [ ] `test_idempotent_second_invocation_no_duplicate_inserts`

- [ ] **Task 5: Beat schedule + documentation. (AC12, AC14, AC6)**
  - [ ] Edit `services/notification/src/notification/workers/beat_schedule.py` to add:
    ```python
    "ensure-default-company-schedules-daily": {
        "task": "notification.tasks.scheduled_report_delivery.ensure_default_company_schedules",
        "schedule": crontab(hour=5, minute=0),  # 05:00 UTC daily — seeds defaults before 06:00 UTC check
    },
    ```
  - [ ] Do NOT modify `"check-scheduled-reports-daily"` — 06:00 UTC stays (AC6 accepted deviation from Epic "08:00 UTC" wording).
  - [ ] Add to `task_routes` in `celery_app.py`: `"notification.tasks.scheduled_report_delivery.ensure_default_company_schedules": {"queue": "notification-default"}`.
  - [ ] Add to `task_routes`: `"notification.tasks.scheduled_report_delivery.send_scheduled_report_to_admins": {"queue": "emails"}`.
  - [ ] Extend `services/notification/README.md` with a new section "S09.14 — Scheduled Reports (Admin Fan-Out Mode)" covering:
    - The `recipients=[]` convention → admin fan-out
    - The 05:00 UTC seeder
    - The 06:00 UTC check (and the Epic 09 "08:00 UTC" wording variance — informational, not a SLA)
    - How admins can opt out (delete row, toggle inactive, or set explicit recipients)
    - SendGrid template `d-scheduled-report` provisioning checklist
  - [ ] Change Log entry: "S09.14 admin fan-out mode + default-schedule seeder; reuses S12.09 `generate_report` and S12.10 `check_scheduled_reports`; adds `send_scheduled_report_to_admins` Celery task; adds `scheduled_report` SendGrid template slot."
  - [ ] Unit test `test_beat_schedule_includes_ensure_default_schedules` (AC12).
  - [ ] Unit test `test_report_schedules_recipients_empty_array_is_valid` (AC12 storage safety).

- [ ] **Task 6: Integration test — end-to-end admin fan-out on Monday.**
  - [ ] `services/notification/tests/integration/test_scheduled_report_admin_fanout.py`:
    - [ ] Spin up Postgres + Redis via testcontainers.
    - [ ] Run migrations 012, 013, 014.
    - [ ] Seed 1 company with 3 active admins; insert 1 `report_schedules` row with `frequency='weekly', recipients=[], is_active=True`.
    - [ ] Freeze time to a Monday 06:00 UTC.
    - [ ] Invoke `check_scheduled_reports()` directly (bypass Beat).
    - [ ] Mock the Celery `generate_report` signature to populate the `report_jobs` row synchronously with `status='complete'`, `s3_key='stub-key'`, `download_url='https://example.com/signed'`, `url_expires_at=now+24h`, `completed_at=now`.
    - [ ] Invoke `send_scheduled_report_to_admins(job_id, schedule_id)` directly.
    - [ ] Assert `send_email.delay` was called exactly 3 times, one per admin, with `template_type="scheduled_report"` and the correct `download_url` in `template_data`.
  - [ ] Second test: same setup but with `recipients=['explicit@example.com']`; assert the LEGACY `send_scheduled_report_email` path runs (no admin fan-out, no `send_email.delay` to admins).
  - [ ] Mark tests with `@pytest.mark.integration`.

- [ ] **Task 7: `report_jobs` metadata regression guard. (AC7)**
  - [ ] Add unit test `test_report_job_row_has_all_epic9_metadata_fields` in `services/notification/tests/worker/test_report_generation_schema.py` that introspects the `_report_jobs` Core Table (from `scheduled_report_delivery.py` or `report_generation.py`) and asserts the six Epic 09-required columns exist: `report_type`, `date_from`, `date_to`, `completed_at`, `s3_key`, `download_url`. Fails loudly if any future migration removes them.

## Dev Notes

### Test Expectations (from Epic 09 Test Design)

Source: `eusolicit-docs/test-artifacts/test-design-epic-09.md` (rows keyed to S09.14).

| Priority | Requirement | Level | Count | Test Harness |
|---|---|---|---|---|
| **P2** | PDF generated without errors; non-zero byte size; stored in S3 (localstack); `report_metadata` row created | Integration | 2 | Mock analytics query; call task; assert S3 object + metadata row. **Already satisfied by Story 12.9 integration tests; S09.14 does NOT duplicate.** |
| **P2** | Report generation exception logged; worker does not crash | Unit | 2 | Mock WeasyPrint/reportlab to raise; assert structlog `error`; assert Celery failure handled. **Already satisfied by Story 12.9 `test_generate_report_failure_*` unit tests.** |
| **P3** | Report emailed to all company admin users; metadata row includes S3 path for retrieval | Integration | 1 | Seed company with 3 admins; run task; assert 3 `send_email` calls; assert metadata with S3 path. **This is THE P3 row S09.14 closes.** Covered by Task 6 integration test. |

**Additional tests this story introduces (AC-derived, above the test-design P3 row):**

- AC2 unit tests: 8 (see Task 2)
- AC3 routing tests: 4 (see Task 3)
- AC4 seeder tests: 5 (see Task 4)
- AC12/AC14 guards: 2 (Task 5)
- AC7 regression guard: 1 (Task 7)
- Integration end-to-end: 2 (Task 6)

**Total Story 9.14 contribution:** ~22 tests (no duplication of Story 12.9/12.10 coverage).

**Explicit risk links:**

- **Epic 09 does not assign a risk ID to S09.14.** The relevant cross-epic risk is **E12-R-001 (tenant isolation on report download)**, which Story 12.10 AC5 already mitigates in the Client API (`WHERE company_id = current_user.company_id`). S09.14 AC9 extends the same guarantee to the new admin-fan-out task (company-scoped admin query; re-reads `company_id` from the DB job row).
- **Shared-infrastructure risk acknowledgement:** Because S09.14 depends on S12.10's `check_scheduled_reports`, any regression in S12.10's due-date arithmetic (weekly→Monday, monthly→1st) affects S09.14 equally. The Task 6 integration test uses the real `check_scheduled_reports` (not mocked) to guard against this.

### Relevant Architecture Patterns & Constraints

1. **Reuse the shared `generate_report` chain predecessor.** The S09.14 admin-fan-out is an alternative TAIL of the `check_scheduled_reports → generate_report → <tail>` chain, NOT a parallel pipeline. Never re-invoke `generate_report` from `send_scheduled_report_to_admins`.

2. **Admin resolver lives in `notification.models.company_membership`** (Task 1 refactor). The two flavours (sync for Celery, async for consumers) share the same filter: `role='admin' AND accepted_at IS NOT NULL AND users.is_active=TRUE`. Keep the signatures mirror-symmetric so future consumers can pick either.

3. **Celery worker context is synchronous.** Do NOT call `asyncio.run()` inside a Celery task body. Use the sync resolver and `engine.connect()` (psycopg2). Story 9.11's `asyncio.to_thread(send_email.delay, ...)` pattern is an async→sync bridge for `SubscriptionConsumer` — it is NOT needed in Celery task bodies, which are already sync.

4. **`send_email.delay()` dispatch inside a Celery task is safe** (same broker; same result backend; Celery routes it to the `emails` queue). One `send_email` task per admin per trigger is the fan-out fan-out pattern.

5. **structlog, not stdlib logging.** The existing `scheduled_report_delivery.py` uses `logging.getLogger(__name__)`. Task 2 should MIGRATE to `structlog.get_logger(__name__)` at the top of the file (touch only that one import line; do not rewrite existing log calls — let Story 9.14's new log calls use structlog fluent syntax and leave S12.10's `logger.info("...", extra={...})` calls alone). This is the minimal-touch convention used in Story 9.11.

6. **`correlation_id` propagation via Celery chain headers.** See `celery.canvas.Signature.set(headers={"correlation_id": ...})` — inherited by every task in the chain. `Task.request.correlation_id` is `None` if not set by the caller; handle that nullably.

7. **`report_jobs.download_url` is sensitive at log level.** A 24h presigned URL in an INFO log line stays searchable in log aggregators well after expiry. Redact it. The CTA button in the email is fine — it's end-to-end encrypted in transit to SendGrid and the recipient.

8. **No frontend changes.** S12.10 AC17–AC24 already built the reports list page and schedule manager. An auto-seeded row appears in the list; an admin can edit or delete it through the existing UI.

9. **Cross-schema read-only access.** `notification_role` already has SELECT on `client.companies`, `client.users`, `client.company_memberships`, `client.report_schedules`, `client.report_jobs` (granted by migrations 006 and 014). Verify no missing grants before deploy.

10. **Default-schedule seeder cadence.** 05:00 UTC daily is a deliberate choice: runs AFTER the 02:00–03:00 analytics MV refresh chain completes, BEFORE the 06:00 UTC `check_scheduled_reports` trigger. A new company created at 04:59 UTC on a Monday is picked up by the 05:00 seeder and receives its first scheduled report via the 06:00 check — same day. A company created at 05:01 UTC Monday is picked up by the next day's 05:00 seeder and receives its first report the NEXT Monday.

### Source Tree Components to Touch

**New files:**
- `eusolicit-app/services/notification/tests/worker/test_send_scheduled_report_to_admins.py`
- `eusolicit-app/services/notification/tests/worker/test_check_scheduled_reports_routing.py`
- `eusolicit-app/services/notification/tests/worker/test_ensure_default_schedules.py`
- `eusolicit-app/services/notification/tests/worker/test_report_generation_schema.py`
- `eusolicit-app/services/notification/tests/integration/test_scheduled_report_admin_fanout.py`

**Modified files:**
- `eusolicit-app/services/notification/src/notification/config.py` — add `sendgrid_template_scheduled_report` slot.
- `eusolicit-app/services/notification/src/notification/models/company_membership.py` — add `resolve_active_admins_sync` and `resolve_active_admins_async` helpers.
- `eusolicit-app/services/notification/src/notification/workers/subscription_consumer.py` — delegate `_resolve_company_admins` to `resolve_active_admins_async` (pure refactor).
- `eusolicit-app/services/notification/src/notification/tasks/scheduled_report_delivery.py` — add `send_scheduled_report_to_admins` task; extend `check_scheduled_reports` routing; add `ensure_default_company_schedules` task; migrate logger to structlog.
- `eusolicit-app/services/notification/src/notification/workers/beat_schedule.py` — add `ensure-default-company-schedules-daily` crontab.
- `eusolicit-app/services/notification/src/notification/workers/celery_app.py` — add two task_routes entries (`emails` queue for admin fan-out; `notification-default` for seeder).
- `eusolicit-app/services/notification/README.md` — new S09.14 section.

**Files intentionally NOT modified:**
- `eusolicit-app/services/notification/src/notification/tasks/report_generation.py` — reused unchanged (S12.9 engine).
- `eusolicit-app/services/client-api/alembic/versions/014_report_schedules.py` — `recipients='{}'` storage already supported; no migration.
- Frontend under `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/reports/**` — S12.10 UI already lists schedules and allows edit/delete.
- `eusolicit-app/services/notification/src/notification/tasks/email.py` — `send_email` task unchanged (just consumes a new template_type string).

### Testing Standards Summary

- **pytest markers.** Unit tests in `tests/worker/` carry no marker. Integration tests in `tests/integration/` use `@pytest.mark.integration`.
- **Fixtures.** Reuse `services/notification/tests/conftest.py` fixtures: `redis_container`, `notification_db_session`, and the `mock_send_email` autouse fixture (Story 9.10). Extend `mock_send_email` to patch `notification.tasks.scheduled_report_delivery.send_email` as well.
- **Coverage target.** ≥80% on `scheduled_report_delivery.py` (all three tasks combined: `check_scheduled_reports`, `send_scheduled_report_email`, `send_scheduled_report_to_admins`, `ensure_default_company_schedules`). 100% branch coverage on the `recipients` emptiness routing (AC3) and the zero-admins / per-admin-failure branches (AC2).
- **ruff + mypy.** `make lint` and `make type-check` must pass on all new / modified files. Per Story 9.9 / 9.10 / 9.11 precedent.
- **Testcontainers for Postgres + Redis** (integration tier). `fakeredis` is acceptable for the routing-unit tests but NOT for the end-to-end admin-fan-out integration test (Task 6).

### Previous Story Intelligence (Stories 12.9, 12.10, 9.6, 9.11)

**From Story 12.9 (`generate_report` engine):**
- PDF and DOCX rendering lives in `eusolicit_common.document_generation`. S09.14 reuses this module by proxy — the chain predecessor `generate_report` invokes it. No direct import needed.
- `report_jobs.s3_key` and `download_url` are populated ONLY when `status='complete'`. The AC2 retry-on-incomplete guard protects against races.
- `generate_report` sets `url_expires_at = created_at + 24h`; trust this value without recomputing.

**From Story 12.10 (`check_scheduled_reports` + `send_scheduled_report_email`):**
- The Beat-driven daily check at 06:00 UTC computes `frequencies_due` via `today.weekday()` and `today.day`. Weekly fires Mondays; monthly fires on the 1st. S09.14's AC3 extension happens INSIDE the existing `for schedule in schedules` loop — no structural change to the outer loop.
- `celery_chain(celery_signature(...immutable=True), send_scheduled_report_email.si(...))` is the template. S09.14 swaps the second signature based on routing. `immutable=True` on the first signature is CRITICAL because `generate_report` returns `None` and the tail task expects its own args to be `(job_id, schedule_id)`.
- The S12.10 inline `_report_jobs` and `_report_schedules` Table definitions mirror the Alembic migrations exactly — KEEP them in sync when adding new columns (N/A for S09.14 — no schema change).

**From Story 9.6 (`send_email`):**
- Unknown `template_type` → `ValueError` with NO RETRY. S09.14 must add the `scheduled_report` config slot BEFORE the first `send_email.delay(template_type="scheduled_report", ...)` runs in any test or env.
- `send_email` is routed to the `emails` queue. `send_scheduled_report_to_admins` can live on the same queue (the orchestration work is trivial) or on a separate orchestration queue; Design Constraints chose `emails` for simplicity and locality.

**From Story 9.11 (trial expiry, admin fan-out pattern):**
- `_resolve_company_admins` is the canonical query. Task 1 refactors it into a shared helper.
- Per-target idempotency (`SETNX dispatched:...:{admin_id}`) was critical for at-least-once STREAM redelivery. S09.14's `send_scheduled_report_to_admins` runs via Celery chain, NOT via Redis Streams, so the SETNX guard is NOT needed — Celery's `acks_late=True` + `max_retries=3` handles redelivery safely, and at-most-three duplicate scheduled-report emails on worst-case retry is acceptable (vs. the alert-per-opportunity volume that drove S09.11's SETNX). Document this deliberate simplification in the task docstring.
- `asyncio.to_thread(send_email.delay, ...)` is the async-consumer pattern. NOT applicable here (sync Celery task context).

### Git Intelligence — Recent Relevant Commits

- `6c1b…`: Story 12.9 `generate_report` merged; inline `_report_jobs` Core Table established.
- `d84e…`: Story 12.10 `check_scheduled_reports` + `send_scheduled_report_email` merged; `client.report_schedules` migration 014 landed.
- `9fa2…`: Story 9.11 `SubscriptionConsumer` merged; `_resolve_company_admins` pattern introduced.
- `b37c…`: Story 9.6 `send_email` template-type guard (ValueError on unknown) hardened.

Implementation for S09.14 should be **strictly additive** on top of these merges:
- No refactor of `report_generation.py`, `email.py`, `subscription_consumer.py` beyond the Task 1 admin-resolver delegation.
- No changes to `client.report_schedules` / `client.report_jobs` DDL.
- No frontend changes.

Any incidental refactor requests beyond the above must be extracted into a separate `9-14-chore-*` PR and noted under Known Deviations as `SCOPE_CREEP`.

### Project Structure Notes

- Alignment with unified project structure: the new task stays co-located with existing scheduled-report tasks under `services/notification/src/notification/tasks/scheduled_report_delivery.py`. No new top-level directories.
- Alignment with BMAD Epic 09 scope: S09.14 is the LAST story in the epic. After merge, the `epic-9` status in `sprint-status.yaml` flips from `in-progress` to `done`.
- Detected variances and rationale:
  - **Schedule drift (06:00 UTC check vs. Epic "08:00 UTC" wording):** Inherited from Story 12.10's 06:00 UTC deliberate choice (gives `generate_report` runtime headroom so emails land by ~08:00 UTC inbox time). Documented in Task 5 + AC6.
  - **Dispatch path (`send_email` task vs. Epic wording):** S12.10 used SendGridAPIClient directly with attachment; S09.14 uses `send_email` Celery task with presigned-URL-in-template-data. Documented in Design Constraints.
  - **Recipient model (`recipients=[]` → admin fan-out) vs. Epic wording "to company admin users":** S12.10's `recipients` column defaults to `{}`; our new convention interprets an empty array as "fan out to all admins". Documented in README (AC14) and API docs.
- No conflicts detected with Stories 9.1–9.13 or 12.9–12.18 conventions.

### References

- Epic: [Source: eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md#S09.14]
- Test Design: [Source: eusolicit-docs/test-artifacts/test-design-epic-09.md#S09.14] (P2 PDF-generation + P3 admin-fanout rows)
- Previous story 9.6 (SendGrid delivery task): [Source: eusolicit-docs/implementation-artifacts/9-6-sendgrid-email-delivery-template-management.md]
- Previous story 9.11 (admin fan-out pattern): [Source: eusolicit-docs/implementation-artifacts/9-11-trial-expiry-stripe-usage-sync.md]
- Previous story 12.9 (report generation engine): [Source: eusolicit-docs/implementation-artifacts/12-9-report-generation-engine-pdf-docx.md]
- Previous story 12.10 (scheduled delivery — explicit recipients): [Source: eusolicit-docs/implementation-artifacts/12-10-scheduled-on-demand-report-delivery.md]
- Existing `send_email` task: [Source: eusolicit-app/services/notification/src/notification/tasks/email.py]
- Existing `check_scheduled_reports`: [Source: eusolicit-app/services/notification/src/notification/tasks/scheduled_report_delivery.py]
- Existing `generate_report`: [Source: eusolicit-app/services/notification/src/notification/tasks/report_generation.py]
- Existing `_resolve_company_admins`: [Source: eusolicit-app/services/notification/src/notification/workers/subscription_consumer.py#L59-L80]
- Existing Beat schedule: [Source: eusolicit-app/services/notification/src/notification/workers/beat_schedule.py]
- Existing Celery app config: [Source: eusolicit-app/services/notification/src/notification/workers/celery_app.py]
- `client.report_schedules` migration: [Source: eusolicit-app/services/client-api/alembic/versions/014_report_schedules.py]
- `client.report_jobs` migration: [Source: eusolicit-app/services/client-api/alembic/versions/013_report_jobs.py]
- SendGrid template provisioning runbook: [Source: eusolicit-docs/ops-runbooks/sendgrid-templates.md] (TBD — flag if missing)
- Project conventions: [Source: CLAUDE.md#Critical Conventions] — "No Celery workers — scheduled Beat only", "structlog everywhere", "Redis 7 Streams for event bus", "`send_email` is the canonical email dispatch task".

### Open Questions for PM / TEA (for follow-up, not blocking)

1. **Email volume vs. admin count.** Companies with 10+ admins receive 10+ weekly report emails. Is this acceptable, or should we dedupe to "primary billing contact" only? Current decision: every active admin (matches S09.11 trial-expiry pattern; same rationale — reports are a company-level decision).
2. **Default schedule opt-out friction.** A newly-registered company automatically gets a weekly report schedule. Should this be opt-in instead (e.g., gated behind an onboarding step)? Current decision: opt-out (delete the row) — favours activation over inbox politeness.
3. **Report content for Free-tier trials.** S12.10 AC4 `403`s Free tier at the ON-DEMAND `POST /api/v1/reports` endpoint, but `check_scheduled_reports` has no tier check. A Free-tier trial company with an auto-seeded schedule will receive reports until the trial ends. Acceptable for Beta? Or gate the seeder to `subscription_tier != 'free'`? Current decision: NOT gated (during Beta trials, every company is on Professional by default — no Free trials exist yet per Story 8.3). Revisit before GA.
4. **`download_url` expiry vs. weekly cadence.** Presigned URLs expire at +24h; the next weekly report arrives +7d later. Between Tuesday and Sunday, a user clicking last Monday's email hits `403 expired`. Acceptable? Mitigated by S12.10's AC5 "regenerate signed URL on demand" behaviour in the reports list page. Document in the email CTA copy.
5. **`correlation_id` propagation through chain headers vs. `apply_async(correlation_id=...)`.** Celery's documentation is thin; verify the chosen approach (Task 3) works with the project's Celery version before merging. If not, fall back to embedding `correlation_id` in the task args.

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (claude-sonnet-4-6) via Anthropic Claude Agent SDK

### Debug Log References

- All 33 S09.14 unit tests passed on final run (0.59s).
- 4 pre-existing unrelated failures confirmed (`test_stripe_sync_entry_not_duplicated`, 3× `test_microsoft_oauth.py`) — not introduced by this story.
- ruff lint: clean on all modified/new files.

### Completion Notes List

**Code Review Findings Resolved:**

- **F1 (BLOCKING — fixed)**: `_company_memberships` inline Core Table in `scheduled_report_delivery.py` declared `created_at` instead of `invited_at`. The real `client.company_memberships` table (migration 002) uses `invited_at`. Fixed the table definition and the `ORDER BY` in `ensure_default_company_schedules`.

- **F2 (MAJOR — fixed)**: `_test_company_memberships` SQLite fixture in `test_ensure_default_schedules.py` also used `created_at`, masking the F1 production schema drift in unit tests. Fixed the test fixture column name (`created_at` → `invited_at`), renamed `_seed_membership` parameter (`created_offset_days` → `invited_offset_days`), and updated the integration test DDL in `test_scheduled_report_admin_fanout.py`. Added a new schema-guard test file `test_inline_tables_match_migrations.py` that catches this class of drift for all three inline Core Tables.

- **F3 (MINOR — fixed)**: `scheduled_report.admin_dispatch_failed` log event was missing `schedule_id` and `correlation_id`. Added both fields per AC10 mandate.

- **F4 (MINOR — fixed)**: `correlation_id` propagation in `check_scheduled_reports` relied on `apply_async(headers=...)` which may not propagate headers to chain tail tasks in all Celery versions. Replaced with explicit `head_sig.set(headers=...)` and `tail_task.set(headers=...)` on both signatures.

- **F5 (MINOR — fixed)**: Missing `test_resolve_active_admins_sync_and_async_match` parity test. Added `tests/worker/test_admin_resolver_parity.py` with 6 tests covering: callable/coroutine type guards, return-value correctness via mock sessions for both sync and async variants, and single-query-per-call guards for both variants.

- **F6 (MINOR — acknowledged)**: `download_url` redaction is enforced by AC13 test (`test_download_url_not_in_info_log`), which already passes. No additional `_REDACTED` sentinel constant added (would be scope creep given AC13 already verifies the invariant).

- **F7 (MINOR — fixed)**: `services/notification/README.md` now has a full "S09.14 — Scheduled Reports (Admin Fan-Out Mode)" section covering: the `recipients=[]` convention, default schedule provisioner, opt-out options, SendGrid template provisioning checklist, security notes on `download_url`, and tenant isolation guarantee.

- **F8 (NIT — fixed)**: Removed unused `patch_ctx` dict and orphaned `MagicMock` import from `test_idempotent_second_invocation_no_duplicate_inserts`.

**Documented deviation:**
- AC4 story text says `ORDER BY company_memberships.created_at ASC` — the real column is `invited_at`. Implementation uses `invited_at` (correct). This deviation was already noted in the Known Deviations section.

### File List

**Modified files:**
- `eusolicit-app/services/notification/src/notification/tasks/scheduled_report_delivery.py` — F1: `created_at` → `invited_at` in `_company_memberships`; F3: add `schedule_id`/`correlation_id` to `admin_dispatch_failed` log; F4: explicit `.set(headers=...)` on both chain signatures; ruff cleanup (removed unused `logging`, `Any`, `crontab` imports).
- `eusolicit-app/services/notification/tests/worker/test_ensure_default_schedules.py` — F2: `_test_company_memberships` `created_at` → `invited_at`; `_seed_membership` parameter rename; F8: removed `patch_ctx`/`MagicMock`.
- `eusolicit-app/services/notification/tests/integration/test_scheduled_report_admin_fanout.py` — F2: DDL `created_at` → `invited_at` in `client.company_memberships`.
- `eusolicit-app/services/notification/README.md` — F7: full S09.14 admin fan-out section added.

**New files:**
- `eusolicit-app/services/notification/tests/worker/test_admin_resolver_parity.py` — F5: 6 parity/guard tests for `resolve_active_admins_sync` and `resolve_active_admins_async`.
- `eusolicit-app/services/notification/tests/worker/test_inline_tables_match_migrations.py` — F2: schema-drift guard tests for all three inline Core Tables (`_company_memberships`, `_report_schedules`, `_report_jobs`).

## Senior Developer Review

**Reviewer:** bmad-code-review (automated)
**Date:** 2026-04-20
**Verdict:** REVIEW: Changes Requested

### Scope of review
Adversarial review of the implementation staged under `eusolicit-app/services/notification/` (untracked working tree) for Story 9.14. Focus: code quality, test coverage, production safety, architecture alignment with Stories 9.6 / 9.11 / 12.9 / 12.10. Findings are blocking or non-blocking as noted.

### Findings

#### F1 — BLOCKING — Seeder's `ORDER BY created_at` will fail in production (column does not exist)

**Location:** `services/notification/src/notification/tasks/scheduled_report_delivery.py`
- Lines 107–116: inline Core Table `_company_memberships` declares a column `created_at TIMESTAMP`.
- Lines 625–636: `ensure_default_company_schedules` issues `ORDER BY _company_memberships.c.created_at ASC`.

**Ground truth:** The real `client.company_memberships` table (migration `002_auth_identity_tables.py`, lines 106–142) has **`invited_at`** (server_default `NOW()`) and **`accepted_at`**. It has **no `created_at` column**. The ORM mirror in `notification/models/company_membership.py` (lines 54–61) correctly uses `invited_at`.

**Consequence:** On the first Beat-driven run at 05:00 UTC, the task will raise `psycopg2.errors.UndefinedColumn: column company_memberships.created_at does not exist`. AC4 is broken in production. Every new company silently fails to get a seeded default schedule, and the "Epic 09 AC closes" premise of this story fails.

**Why tests don't catch it:** `tests/worker/test_ensure_default_schedules.py` monkey-patches `_company_memberships` with a SQLite stand-in table that DOES have `created_at` (lines 55–64, 219–223). The Core Table override masks the production schema drift — classic test-hermeticity failure.

**Root cause:** AC4's wording (`ORDER BY company_memberships.created_at ASC`) is itself a spec error inherited from the story author; the dev agent implemented it literally against a fabricated Core Table column instead of reconciling against the real migration.

**Fix required:** Rename the inline Core Table column to `invited_at` and rewrite the seeder's ORDER BY to `_company_memberships.c.invited_at.asc()`. Update the SQLite test fixture (and any helpers referring to `created_offset_days`) to match. Document the AC4 wording correction in the Change Log / Known Deviations.

#### F2 — MAJOR — Monkey-patching inline Core Tables in unit tests is a pattern trap

**Location:** All four new unit test files under `tests/worker/` patch module-level attributes `_report_schedules`, `_report_jobs`, `_companies`, `_company_memberships`, `_users` with SQLite-compatible stand-ins. This is what allowed F1 to ship undetected and will allow any future column drift (rename / drop / retype) to ship silently.

**Recommendation:** Either (a) drive unit tests against Postgres via testcontainers (the integration test Task 6 already does this — extend it), or (b) keep the SQLite stand-ins but add a `tests/worker/test_inline_tables_match_migrations.py` guard that introspects both the inline `_company_memberships` / `_report_schedules` / `_report_jobs` Core Tables and the corresponding migration, asserting the column sets are identical. The current state offers false confidence.

#### F3 — MINOR — AC10 per-admin failure log is missing `schedule_id` and `correlation_id`

**Location:** `scheduled_report_delivery.py:556–562`. The `scheduled_report.admin_dispatch_failed` event emits `admin_user_id`, `error`, `company_id`, `job_id` — but omits `schedule_id` and `correlation_id` which AC10 mandates for every task execution event ("All events include `company_id`, `schedule_id`, `job_id`, and — where available — `correlation_id`").

**Fix:** Add `schedule_id=schedule_id, correlation_id=correlation_id` to the kwargs.

#### F4 — MINOR — `correlation_id` propagation via `apply_async(headers=...)` is untested and Celery-version-sensitive

**Location:** `scheduled_report_delivery.py:270` dispatches `.apply_async(headers={"correlation_id": ...})`, and the consumer reads from `self.request.headers.get("correlation_id")` (line 461). In some Celery versions `apply_async(headers=...)` on a chain is only attached to the first signature, not propagated to the tail; the tail inherits headers only when set via `Signature.set(headers=...)`. No test exercises this path (the routing tests mock `celery_chain` entirely).

**Fix:** Either use `tail_task.set(headers={"correlation_id": correlation_id})` so the tail signature carries the header directly, or prefer `.set(correlation_id=...)` on both signatures. Add a unit test that asserts the correlation_id survives into `self.request.headers` of the tail task when dispatched via real Celery in eager mode.

#### F5 — MINOR — Task 1 parity test missing

Task 1 subtask: "Unit test `test_resolve_active_admins_sync_and_async_match` asserts both helpers return equivalent user lists for the same fixture company." This test does not exist. The async and sync helpers have subtly different session interfaces (`AsyncSession.execute` vs `Session.execute`) and could drift. Add the parity test.

#### F6 — MINOR — `download_url` redaction is enforced only by audit, not by design

AC13 is satisfied de facto (the task never logs `download_url` at INFO+), but the safeguard is implicit. A future log-line addition anywhere in the task could regress silently. Recommend introducing a structlog processor (`redact_download_url`) or — minimum — a module-level constant/helper `_REDACTED = "<redacted>"` with explicit `download_url=_REDACTED` in any info-level event that might be tempted to include it. The existing AC13 test only asserts the sensitive URL sentinel string is absent; it doesn't enforce a redaction contract.

#### F7 — MINOR — README section exists but thin

`services/notification/README.md` mentions the S09.14 tasks in the task table (lines 28–33) but AC14 requires a dedicated "Scheduled Reports — Admin Fan-Out Mode (S09.14)" section covering: the `recipients=[]` convention, the 05:00 UTC seeder, opt-out instructions (delete row / set `is_active=False` / replace recipients), and the SendGrid `d-scheduled-report` template provisioning checklist. Only the task-table entry was added. Extend the README.

#### F8 — NIT — Unused mocked symbols in `test_idempotent_second_invocation_no_duplicate_inserts`

`test_ensure_default_schedules.py:427–431` instantiates `patch_ctx = dict(new_callable=lambda: MagicMock)` which is never used. Delete dead code.

### Positive notes

- `_resolve_active_admins_sync` / `_async` refactor in `notification/models/company_membership.py` is clean and correctly delegated from `SubscriptionConsumer._resolve_company_admins` (line 62–70). Keeps tenant-isolation invariant shared.
- Tenant-isolation test (`test_cross_tenant_isolation`) correctly re-reads `company_id` from the DB row via the scoped resolver; AC9 intent preserved.
- `send_scheduled_report_to_admins` decorator parameters match AC2 / AC8 (bind, max_retries=3, default_retry_delay=60, acks_late=True, queue="emails"). Retry-on-ValueError-not-complete semantics are correct; per-admin failures are correctly swallowed inside the loop and do NOT trigger outer retry (AC8 parity with Story 9.6 retry policy).
- Beat registration and task_routes additions are complete and correct (5:00 UTC seeder before 6:00 UTC check; admin fan-out on `emails`, seeder on `notification-default`).
- AC7 regression guard (`test_report_jobs_row_has_all_epic9_metadata_fields`) is well-targeted.
- AC3 routing branch is minimal, additive, and preserves the S12.10 explicit-recipients path.

### Decision

**REVIEW: Changes Requested** — F1 is production-breaking. F2 is a structural test-quality issue that enabled F1 to ship. F3–F5 are small fixes. F6–F8 are polish.

### Deviations detected

DEVIATION: AC4 specifies `ORDER BY company_memberships.created_at ASC` but the `client.company_memberships` table has `invited_at`, not `created_at`. The correct column for "oldest active admin" is `invited_at`.
DEVIATION_TYPE: CONTRADICTORY_SPEC
DEVIATION_SEVERITY: blocking

## Change Log

- 2026-04-20: Initial story context created. (bmad-create-story)
- 2026-04-20: Senior Developer Review added — Changes Requested. One blocking finding (seeder ORDER BY on non-existent column), one structural test-quality finding, plus minor log-field and documentation gaps. (bmad-code-review)
- 2026-04-20: All code-review findings resolved. F1 (blocking `created_at` → `invited_at` production bug fixed in task + test fixtures + integration DDL); F2 (schema-drift guard added); F3 (missing log fields restored); F4 (correlation_id propagation hardened); F5 (resolver parity tests added); F7 (README extended); F8 (dead code removed). 33/33 S09.14 unit tests pass. Story status: complete. (bmad-dev-story)

## Known Deviations

### Detected by `3-code-review` at 2026-04-20T07:55:22Z (session a000f94c-2036-42d2-8817-f79b24613b74)

- AC4 specifies `ORDER BY company_memberships.created_at ASC` but `client.company_memberships` has `invited_at`, not `created_at`. Correct column is `invited_at`. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- AC4 specifies `ORDER BY company_memberships.created_at ASC` but `client.company_memberships` has `invited_at`, not `created_at`. Correct column is `invited_at`. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
