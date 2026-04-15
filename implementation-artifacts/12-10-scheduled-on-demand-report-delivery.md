# Story 12.10: Scheduled & On-Demand Report Delivery

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **company admin or authenticated user on the EU Solicit platform**,
I want **to configure scheduled weekly/monthly reports that are emailed automatically and to generate on-demand reports from any analytics page with an in-app download link when ready**,
so that **I receive actionable business intelligence reports without manual effort, and can generate ad-hoc reports whenever needed with immediate feedback on their status**.

## Acceptance Criteria

### Infrastructure — `client.report_schedules` Table

1. **AC1 — Alembic migration 014** creates the `client.report_schedules` table. Migration file: `services/client-api/alembic/versions/014_report_schedules.py` with `down_revision = "013"`. The `upgrade()` creates:

   ```sql
   CREATE TABLE IF NOT EXISTS client.report_schedules (
       id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
       company_id      UUID NOT NULL REFERENCES client.companies(id) ON DELETE CASCADE,
       created_by_id   UUID NOT NULL REFERENCES client.users(id),
       report_type     TEXT NOT NULL CHECK (report_type IN (
                           'pipeline_summary', 'bid_performance_summary',
                           'team_activity', 'custom_date_range')),
       format          TEXT NOT NULL CHECK (format IN ('pdf', 'docx')),
       frequency       TEXT NOT NULL CHECK (frequency IN ('weekly', 'monthly')),
       recipients      TEXT[] NOT NULL DEFAULT '{}',
       is_active       BOOLEAN NOT NULL DEFAULT TRUE,
       created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
       updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
   );
   ```

   Followed by:
   ```sql
   CREATE INDEX ix_report_schedules_company_id     ON client.report_schedules (company_id);
   CREATE INDEX ix_report_schedules_company_active  ON client.report_schedules (company_id, is_active);
   CREATE INDEX ix_report_schedules_frequency       ON client.report_schedules (frequency, is_active);
   ```

   Permission grants in `upgrade()`:
   ```sql
   GRANT SELECT, INSERT, UPDATE, DELETE ON client.report_schedules TO client_api_role;
   GRANT SELECT ON client.report_schedules TO notification_role;
   ```

   The `downgrade()` drops indexes then `DROP TABLE IF EXISTS client.report_schedules`. Migration is fully reversible.

2. **AC2 — SQLAlchemy Core Table** definition added to `services/client-api/src/client_api/models/report_schedule.py`. Define `report_schedules` as an `sa.Table` with `schema="client"` and `info={"is_view": False}`. Export from `services/client-api/src/client_api/models/__init__.py`. Columns mirror AC1 exactly. `recipients` column uses `sa.ARRAY(sa.Text())`.

---

### Client API — Celery Producer

3. **AC3 — Celery producer** configured in `services/client-api/src/client_api/celery_producer.py`:

   ```python
   from celery import Celery
   from client_api.config import settings

   celery_producer = Celery(
       "client_api_producer",
       broker=settings.CELERY_BROKER_URL,
   )

   def dispatch_generate_report(
       job_id: str,
       company_id: str,
       report_type: str,
       format: str,
       date_from: str | None = None,
       date_to: str | None = None,
   ) -> None:
       """Dispatch generate_report task to the notification service worker."""
       celery_producer.send_task(
           "notification.tasks.report_generation.generate_report",
           args=[job_id, company_id, report_type, format, date_from, date_to],
       )
   ```

   `CELERY_BROKER_URL` added to `ClientApiSettings` in `services/client-api/src/client_api/config.py`:
   ```python
   CELERY_BROKER_URL: str = Field(default="redis://redis:6379/0", env="CELERY_BROKER_URL")
   ```

   > The client-api acts as a **Celery producer only** — no workers, no beat. It uses `send_task()` by name so the notification service's task code is never imported directly.

---

### Client API — On-Demand Report Endpoints

4. **AC4 — `POST /api/v1/reports`** — Create an on-demand report job. Requires valid Bearer JWT (`→ 401` on missing/invalid). All paid tiers can trigger reports; Free tier receives `403` with `{"upgrade_required": true}`.

   Request body (JSON):
   ```json
   {
     "report_type": "pipeline_summary" | "bid_performance_summary" | "team_activity" | "custom_date_range",
     "format": "pdf" | "docx",
     "date_from": "2026-01-01" | null,
     "date_to": "2026-03-31" | null
   }
   ```

   Execution flow:
   - Validate `report_type` and `format` values (422 on invalid)
   - If `report_type == "custom_date_range"`, `date_from` and `date_to` are required (422 otherwise)
   - Create `client.report_jobs` row with `status='pending'`, `company_id=current_user.company_id`, `requested_by_id=current_user.id`
   - Call `dispatch_generate_report(job_id, company_id, report_type, format, date_from, date_to)` from `celery_producer`
   - Write audit log entry: `entity_type="report_job"`, `action_type="create"`, `entity_id=job_id`, `after={"report_type": ..., "format": ...}`
   - Return HTTP 202:
     ```json
     {
       "job_id": "<uuid>",
       "status": "pending",
       "report_type": "pipeline_summary",
       "format": "pdf",
       "created_at": "2026-04-13T06:00:00Z"
     }
     ```

5. **AC5 — `GET /api/v1/reports/{job_id}`** — Retrieve report job status and download URL. Requires valid JWT. Returns:

   ```json
   {
     "job_id": "<uuid>",
     "report_type": "pipeline_summary",
     "format": "pdf",
     "status": "pending" | "processing" | "complete" | "failed",
     "download_url": "<signed-s3-url>" | null,
     "url_expires_at": "2026-04-14T06:00:00Z" | null,
     "error_message": null | "<string>",
     "created_at": "2026-04-13T06:00:00Z",
     "completed_at": "2026-04-13T06:02:30Z" | null
   }
   ```

   - **Tenant isolation (E12-R-001):** Query includes `WHERE company_id = current_user.company_id AND id = job_id`. Return `404` if not found or belongs to a different company — never expose cross-tenant job data.
   - When `status='complete'` but `url_expires_at < NOW()`, regenerate the signed URL, update `report_jobs`, return the new URL.

6. **AC6 — `GET /api/v1/reports`** — List company report jobs (paginated). Requires valid JWT.

   Query parameters: `page` (default 1), `page_size` (default 20, max 100), `status` (optional filter).

   Response:
   ```json
   {
     "items": [<report job objects per AC5>],
     "total": 42,
     "page": 1,
     "page_size": 20
   }
   ```

   - Filter: `WHERE company_id = current_user.company_id` — mandatory, non-optional.
   - Order: `created_at DESC`.
   - Only the authenticated company's jobs are returned, even if `status` filter could match other companies.

---

### Client API — Report Schedule Endpoints

All schedule endpoints require:
- Valid Bearer JWT (`→ 401`)
- Company admin role (`company_role == 'admin'`; `→ 403` for non-admins)
- All queries scope to `company_id = current_user.company_id`

7. **AC7 — `POST /api/v1/reports/schedules`** — Create report schedule.

   Request body:
   ```json
   {
     "report_type": "pipeline_summary" | "bid_performance_summary" | "team_activity" | "custom_date_range",
     "format": "pdf" | "docx",
     "frequency": "weekly" | "monthly",
     "recipients": ["email@example.com", ...]
   }
   ```

   - Validate `recipients` is a non-empty list of valid email strings (422 on invalid email; 422 if empty list).
   - Max 20 recipients per schedule (422 if exceeded).
   - Create `report_schedules` row with `is_active=True`.
   - Write audit log: `entity_type="report_schedule"`, `action_type="create"`, `after=<full schedule object>`.
   - Return HTTP 201 with the created schedule (including `id`, `is_active`, `created_at`).

8. **AC8 — `GET /api/v1/reports/schedules`** — List company report schedules.

   Response: `{"items": [<schedule objects>], "total": <int>}` (not paginated — companies are unlikely to have more than a few dozen schedules).

9. **AC9 — `PUT /api/v1/reports/schedules/{schedule_id}`** — Update schedule.

   - Updateable fields: `report_type`, `format`, `frequency`, `recipients`, `is_active`.
   - Partial updates allowed (PATCH semantics via PUT — only provided fields are updated).
   - `updated_at` set to `NOW()` on every update.
   - Tenant isolation: `WHERE company_id = current_user.company_id AND id = schedule_id` (`→ 404` if not found).
   - Write audit log: `entity_type="report_schedule"`, `action_type="update"`, `before=<old>`, `after=<new>`.
   - Return updated schedule.

10. **AC10 — `DELETE /api/v1/reports/schedules/{schedule_id}`** — Delete schedule.

    - Hard delete (no soft delete — the schedule row is removed).
    - Tenant isolation: `WHERE company_id = current_user.company_id AND id = schedule_id` (`→ 404` if not found).
    - Write audit log: `entity_type="report_schedule"`, `action_type="delete"`, `before=<deleted schedule>`, `after=None`.
    - Return HTTP 204 No Content.

---

### Notification Service — Scheduled Report Delivery

11. **AC11 — `check_scheduled_reports` Celery task** in `services/notification/src/notification/tasks/scheduled_report_delivery.py`:

    ```python
    @celery_app.task(
        name="notification.tasks.scheduled_report_delivery.check_scheduled_reports",
        bind=False,
    )
    def check_scheduled_reports() -> None:
    ```

    Execution logic:
    - Determine `today = date.today()` in UTC.
    - Build `frequencies_due: list[str]`:
      - Append `"weekly"` if `today.weekday() == 0` (Monday)
      - Append `"monthly"` if `today.day == 1` (1st of month)
    - If `frequencies_due` is empty: log `"no schedules due today"` and return immediately.
    - Using a synchronous `get_sync_session()`:
      - For each `frequency` in `frequencies_due`:
        - Query all active schedules:
          ```python
          schedules = session.execute(
              sa.select(report_schedules)
              .where(report_schedules.c.frequency == frequency)
              .where(report_schedules.c.is_active == True)
          ).all()
          ```
        - For each schedule:
          - Generate `job_id = str(uuid4())`
          - Insert `report_jobs` row:
            ```python
            session.execute(sa.insert(report_jobs).values(
                id=job_id,
                company_id=schedule.company_id,
                requested_by_id=schedule.created_by_id,
                report_type=schedule.report_type,
                format=schedule.format,
                status="pending",
            ))
            session.commit()
            ```
          - Dispatch Celery chain:
            ```python
            from celery import chain as celery_chain
            from notification.tasks.report_generation import generate_report
            from notification.tasks.scheduled_report_delivery import send_scheduled_report_email

            celery_chain(
                generate_report.si(
                    job_id,
                    str(schedule.company_id),
                    schedule.report_type,
                    schedule.format,
                ),
                send_scheduled_report_email.si(job_id, str(schedule.id)),
            ).apply_async()
            ```
          - Log `"dispatched scheduled report"` with `job_id`, `schedule_id`, `frequency`

12. **AC12 — `send_scheduled_report_email` Celery task** in the same file (`scheduled_report_delivery.py`):

    ```python
    @celery_app.task(
        name="notification.tasks.scheduled_report_delivery.send_scheduled_report_email",
        bind=True,
        max_retries=3,
        default_retry_delay=60,
        acks_late=True,
    )
    def send_scheduled_report_email(self, job_id: str, schedule_id: str) -> None:
    ```

    Execution flow:
    - Load `report_jobs` row for `job_id`. If `status != 'complete'`, raise `ValueError(f"job {job_id} not complete — status={status}")` to trigger retry.
    - Load `report_schedules` row for `schedule_id`. If not found or `is_active=False`: log warning and return (schedule deleted while task was queued).
    - Download report bytes from S3:
      ```python
      s3_obj = get_s3_client().get_object(Bucket=settings.REPORT_S3_BUCKET, Key=job_row.s3_key)
      report_bytes = s3_obj["Body"].read()
      ```
    - Determine `filename = f"{job_row.report_type}_{job_row.completed_at.strftime('%Y-%m-%d')}.{job_row.format}"`
    - Determine `content_type`:
      - `"pdf"` → `"application/pdf"`
      - `"docx"` → `"application/vnd.openxmlformats-officedocument.wordprocessingml.document"`
    - Compose SendGrid `Mail` object:
      - From: `settings.SENDGRID_FROM_EMAIL`
      - To: each email in `schedule_row.recipients`
      - Subject: `f"[EU Solicit] {report_type_label} Report — {today_str}"`
      - Plain text body (see Dev Notes for format)
      - Attachment: base64-encoded `report_bytes`, `filename`, `content_type`, `disposition="attachment"`
    - Send via `SendGridAPIClient(api_key=settings.SENDGRID_API_KEY).send(message)`
    - On `Exception`: `self.retry(exc=e)` (up to `max_retries=3`). After exhaustion log error and suppress.

13. **AC13 — Celery Beat schedule and include registration**:

    In `services/notification/src/notification/workers/celery_app.py`:

    - Add `"notification.tasks.scheduled_report_delivery"` to the `include` list.
    - Add to `app.conf.beat_schedule`:
      ```python
      from celery.schedules import crontab
      # ...
      "check-scheduled-reports-daily": {
          "task": "notification.tasks.scheduled_report_delivery.check_scheduled_reports",
          "schedule": crontab(hour=6, minute=0),   # 06:00 UTC daily
      },
      ```

    > **Note:** The Celery Beat schedule map may already be populated by S12.01 (MV refresh) tasks. Extend it — do not overwrite.

---

### Frontend — API Client

14. **AC14 — `apps/client/lib/api/reports.ts`** (new file) — exports all report-related API functions using the existing `apiClient` singleton from `apps/client/lib/api/client.ts`:

    ```typescript
    // Types
    export type ReportType = 'pipeline_summary' | 'bid_performance_summary' | 'team_activity' | 'custom_date_range';
    export type ReportFormat = 'pdf' | 'docx';
    export type ReportStatus = 'pending' | 'processing' | 'complete' | 'failed';
    export type ReportFrequency = 'weekly' | 'monthly';

    export interface ReportJob {
      job_id: string;
      report_type: ReportType;
      format: ReportFormat;
      status: ReportStatus;
      download_url: string | null;
      url_expires_at: string | null;
      error_message: string | null;
      created_at: string;
      completed_at: string | null;
    }

    export interface ReportSchedule {
      id: string;
      report_type: ReportType;
      format: ReportFormat;
      frequency: ReportFrequency;
      recipients: string[];
      is_active: boolean;
      created_at: string;
      updated_at: string;
    }

    export interface CreateReportPayload {
      report_type: ReportType;
      format: ReportFormat;
      date_from?: string;
      date_to?: string;
    }

    export interface CreateSchedulePayload {
      report_type: ReportType;
      format: ReportFormat;
      frequency: ReportFrequency;
      recipients: string[];
    }

    // Functions
    export async function createOnDemandReport(payload: CreateReportPayload): Promise<ReportJob>
    export async function getReportJob(jobId: string): Promise<ReportJob>
    export async function listReports(params?: { page?: number; page_size?: number; status?: ReportStatus }): Promise<{ items: ReportJob[]; total: number; page: number; page_size: number }>
    export async function createReportSchedule(payload: CreateSchedulePayload): Promise<ReportSchedule>
    export async function listReportSchedules(): Promise<{ items: ReportSchedule[]; total: number }>
    export async function updateReportSchedule(id: string, payload: Partial<CreateSchedulePayload & { is_active: boolean }>): Promise<ReportSchedule>
    export async function deleteReportSchedule(id: string): Promise<void>
    ```

---

### Frontend — Zod Schemas

15. **AC15 — `apps/client/lib/schemas/reports.ts`** (new file):

    ```typescript
    import { z } from 'zod';

    export const reportTypeEnum = z.enum([
      'pipeline_summary',
      'bid_performance_summary',
      'team_activity',
      'custom_date_range',
    ]);

    export const generateReportSchema = z.object({
      report_type: reportTypeEnum,
      format: z.enum(['pdf', 'docx']),
      date_from: z.string().optional(),
      date_to: z.string().optional(),
    }).superRefine((data, ctx) => {
      if (data.report_type === 'custom_date_range') {
        if (!data.date_from) {
          ctx.addIssue({ code: 'custom', path: ['date_from'], message: 'Required for custom date range' });
        }
        if (!data.date_to) {
          ctx.addIssue({ code: 'custom', path: ['date_to'], message: 'Required for custom date range' });
        }
      }
    });

    export const reportScheduleSchema = z.object({
      report_type: reportTypeEnum,
      format: z.enum(['pdf', 'docx']),
      frequency: z.enum(['weekly', 'monthly']),
      recipients: z
        .array(z.string().email('Invalid email address'))
        .min(1, 'At least one recipient is required')
        .max(20, 'Maximum 20 recipients'),
    });

    export type GenerateReportFormValues = z.infer<typeof generateReportSchema>;
    export type ReportScheduleFormValues = z.infer<typeof reportScheduleSchema>;
    ```

---

### Frontend — GenerateReportModal Component

16. **AC16 — `packages/ui/src/components/GenerateReportModal.tsx`** (new file in `packages/ui`). Export from `packages/ui/src/index.ts`.

    Props:
    ```typescript
    interface GenerateReportModalProps {
      isOpen: boolean;
      onClose: () => void;
      defaultReportType?: ReportType;
      showDateRange?: boolean;    // show date_from/date_to fields
    }
    ```

    Behaviour:
    - Uses `useZodForm(generateReportSchema)` with `defaultValues.report_type = defaultReportType ?? 'pipeline_summary'`.
    - On submit: calls `createOnDemandReport(formValues)`, stores returned `job_id` in local state.
    - After `job_id` is set, switches to **polling mode** using `useQuery`:
      ```typescript
      const { data: jobStatus } = useQuery({
        queryKey: ['report-job', jobId],
        queryFn: () => getReportJob(jobId!),
        enabled: !!jobId,
        refetchInterval: (query) => {
          const status = query.state.data?.status;
          return status === 'complete' || status === 'failed' ? false : 2000;
        },
      });
      ```
    - While polling: shows spinner with `data-testid="report-job-status"` and status text (e.g. `t('reports.modal.status.processing')`).
    - When `status='complete'`: shows download button:
      ```tsx
      <a
        href={jobStatus.download_url!}
        target="_blank"
        rel="noopener noreferrer"
        data-testid="report-download-link"
      >
        {t('reports.modal.downloadButton')}
      </a>
      ```
    - When `status='failed'`: shows error message from `jobStatus.error_message` or fallback `t('reports.modal.generationFailed')`.
    - On modal close (while job still pending/processing): the query is not cancelled — user can reopen the reports list page to find the job.

    Required `data-testid` attributes:
    - `data-testid="generate-report-modal"` — on the modal root
    - `data-testid="report-type-select"` — report type selector
    - `data-testid="report-format-pdf"`, `data-testid="report-format-docx"` — format radio inputs
    - `data-testid="report-date-from"`, `data-testid="report-date-to"` — date range inputs (when shown)
    - `data-testid="generate-report-submit"` — submit button
    - `data-testid="report-job-status"` — status display area during polling
    - `data-testid="report-download-link"` — download anchor when complete
    - `data-testid="report-generation-error"` — error display when failed

---

### Frontend — Reports List Page

17. **AC17 — Reports list page** at `apps/client/app/[locale]/(protected)/reports/page.tsx` (server component shell, renders `<ReportsList />`).

    `apps/client/app/[locale]/(protected)/reports/components/ReportsList.tsx`:

    - Uses TanStack Query: `queryKey: ['reports']`, `queryFn: () => listReports()`.
    - Auto-refresh while any job is `pending` or `processing`:
      ```typescript
      refetchInterval: (query) => {
        const hasPending = query.state.data?.items.some(
          (j) => j.status === 'pending' || j.status === 'processing'
        );
        return hasPending ? 3000 : false;
      }
      ```
    - Wraps query state with `<QueryGuard isLoading={...} isError={...} isEmpty={items.length === 0}>`.
    - Table columns: Report Type (formatted label), Format, Status (badge), Created (relative time), Download.
    - Status badge colours:
      - `pending` → grey
      - `processing` → blue (with animated spinner icon)
      - `complete` → green
      - `failed` → red
    - Download cell: when `status='complete'`, renders `<a href={download_url} target="_blank" data-testid="report-download-{job_id}">Download</a>`; otherwise empty.
    - Expired URL handling: if `url_expires_at` is in the past, display `t('reports.list.linkExpired')` instead of download link.
    - Empty state: `<EmptyState data-testid="reports-empty" title={t('reports.list.emptyTitle')} />`

    Required `data-testid` attributes:
    - `data-testid="reports-list-page"` — page root
    - `data-testid="reports-table"` — table element
    - `data-testid="report-row-{job_id}"` — each table row
    - `data-testid="report-status-{job_id}"` — status badge per row
    - `data-testid="report-download-{job_id}"` — download link per row (when complete)
    - `data-testid="reports-empty"` — empty state

---

### Frontend — Report Schedule Settings Page

18. **AC18 — Report schedule settings page** at `apps/client/app/[locale]/(protected)/settings/reports/page.tsx` (server component shell, renders `<ReportScheduleSettings />`).

    `apps/client/app/[locale]/(protected)/settings/reports/components/ReportScheduleSettings.tsx`:

    - **Admin gate:** Check `authStore.user?.companyRole === 'admin'`. If not admin, render `<ForbiddenState data-testid="reports-schedule-forbidden" message={t('settings.reports.adminOnly')} />` and return.
    - **Schedules list:** Uses `useQuery({ queryKey: ['report-schedules'], queryFn: listReportSchedules })`. Renders existing schedules in a card list with edit/delete/toggle controls.
    - **Create form:** Uses `useZodForm(reportScheduleSchema)`. Fields:
      - Report type: `<Select data-testid="schedule-type-select">`
      - Format: radio group (`data-testid="schedule-format-pdf"`, `data-testid="schedule-format-docx"`)
      - Frequency: radio group (`data-testid="schedule-frequency-weekly"`, `data-testid="schedule-frequency-monthly"`)
      - Recipients: tag-style input where pressing Enter or comma adds a validated email tag. Uses `packages/ui/src/components/TagInput.tsx` if available, otherwise a controlled textarea with comma-split validation. `data-testid="schedule-recipients-input"`.
    - On submit: `createReportSchedule(formValues)`, invalidate `['report-schedules']` query, show success toast.
    - **Toggle active:** `<Switch data-testid="schedule-toggle-{id}">` calls `updateReportSchedule(id, { is_active: !current })`.
    - **Delete:** confirmation dialog before `deleteReportSchedule(id)`. `data-testid="schedule-delete-{id}"`.
    - Mutations use `useMutation` + `onSuccess: () => queryClient.invalidateQueries(['report-schedules'])`.

    Required `data-testid` attributes:
    - `data-testid="reports-schedule-page"` — page root
    - `data-testid="report-schedule-form"` — create form
    - `data-testid="schedule-submit"` — submit button
    - `data-testid="schedule-row-{id}"` — each schedule row
    - `data-testid="schedule-toggle-{id}"` — active toggle per row
    - `data-testid="schedule-delete-{id}"` — delete button per row

---

### Frontend — Analytics Page Integration

19. **AC19 — "Generate Report" button** added to each analytics dashboard page. A `<Button>` that opens `<GenerateReportModal>` pre-set to the relevant `defaultReportType`:

    | Page (component file) | `defaultReportType` |
    |---|---|
    | `analytics/market/` | `'pipeline_summary'` |
    | `analytics/roi/` | `'bid_performance_summary'` |
    | `analytics/team/` | `'team_activity'` |
    | `analytics/competitor/` | `'pipeline_summary'` |
    | `analytics/pipeline/` | `'pipeline_summary'` |
    | `analytics/usage/` | `'team_activity'` |

    Each page client component:
    - Imports and renders `<GenerateReportModal>` from `packages/ui`.
    - Adds a state var `const [isReportModalOpen, setReportModalOpen] = useState(false)`.
    - Renders a `<Button data-testid="generate-report-btn" onClick={() => setReportModalOpen(true)}>` near the page heading.
    - Passes `defaultReportType` per the table above.

---

### Frontend — i18n Keys

20. **AC20 — i18n keys** added to `apps/client/messages/en.json` and `apps/client/messages/bg.json` (identical key structure, BG values translated). New namespaces:

    **`reports` namespace:**
    ```json
    "reports": {
      "modal": {
        "title": "Generate Report",
        "reportTypeLabel": "Report Type",
        "formatLabel": "Format",
        "dateFromLabel": "From",
        "dateToLabel": "To",
        "generateButton": "Generate",
        "downloadButton": "Download Report",
        "status": {
          "pending": "Queued...",
          "processing": "Generating report...",
          "complete": "Report ready",
          "failed": "Generation failed"
        },
        "generationFailed": "Report generation failed. Please try again.",
        "closeButton": "Close"
      },
      "list": {
        "title": "Reports",
        "emptyTitle": "No reports yet",
        "emptyDescription": "Generate a report from any analytics page.",
        "columns": {
          "type": "Report Type",
          "format": "Format",
          "status": "Status",
          "created": "Created",
          "download": "Download"
        },
        "statusLabels": {
          "pending": "Queued",
          "processing": "Processing",
          "complete": "Ready",
          "failed": "Failed"
        },
        "linkExpired": "Link expired",
        "downloadLink": "Download"
      },
      "types": {
        "pipeline_summary": "Pipeline Summary",
        "bid_performance_summary": "Bid Performance",
        "team_activity": "Team Activity",
        "custom_date_range": "Custom Date Range"
      }
    }
    ```

    **`settings.reports` namespace:**
    ```json
    "settings": {
      "reports": {
        "title": "Scheduled Reports",
        "adminOnly": "Only company admins can manage report schedules.",
        "createTitle": "New Schedule",
        "frequencyLabels": { "weekly": "Weekly (Monday)", "monthly": "Monthly (1st)" },
        "scheduleCreated": "Schedule created",
        "scheduleUpdated": "Schedule updated",
        "scheduleDeleted": "Schedule deleted",
        "deleteConfirmTitle": "Delete Schedule",
        "deleteConfirmBody": "This schedule will stop sending reports. This cannot be undone.",
        "columns": {
          "type": "Type", "format": "Format", "frequency": "Frequency",
          "recipients": "Recipients", "active": "Active", "actions": "Actions"
        }
      }
    }
    ```

---

### Tenant Isolation

21. **AC21 — Tenant isolation (E12-R-001):** Every SQL query that reads `report_jobs` or `report_schedules` in both Client API and Notification service MUST include `company_id = current_user.company_id` (or `schedule_row.company_id`) as the **first, non-optional WHERE clause**. The `check_scheduled_reports` task queries schedules by `frequency` and `is_active` only — this is safe because it processes all companies' schedules simultaneously (multi-tenant batch). The `send_scheduled_report_email` task loads the job and schedule by their primary IDs — no company_id filter needed since the IDs are generated internally and not user-supplied.

---

### Tests

22. **AC22 — Unit tests** in `services/client-api/tests/api/test_reports.py`:
    - `POST /api/v1/reports` → 202; response contains `job_id` (UUID) and `status='pending'`; `report_jobs` row exists in DB with `status='pending'`; Celery producer `send_task` was called with correct args (mock `celery_producer.send_task`)
    - `POST /api/v1/reports` with `report_type='custom_date_range'` and missing `date_from` → 422
    - `POST /api/v1/reports` with free-tier JWT → 403 with `upgrade_required: true`
    - `POST /api/v1/reports` without JWT → 401
    - `GET /api/v1/reports/{job_id}` with valid JWT returns job owned by company → 200 with all fields
    - `GET /api/v1/reports/{job_id}` for job belonging to a different company → 404 (tenant isolation)
    - `GET /api/v1/reports/{job_id}` for completed job with expired `url_expires_at` → 200 with refreshed `download_url` (assert S3 presign was called again)
    - `GET /api/v1/reports` returns paginated list scoped to requesting company only
    - `GET /api/v1/reports?status=complete` filters by status correctly

23. **AC23 — Unit tests** in `services/client-api/tests/api/test_report_schedules.py`:
    - `POST /api/v1/reports/schedules` with company admin JWT → 201; schedule row in DB; audit log row created
    - `POST /api/v1/reports/schedules` with non-admin user JWT → 403
    - `POST /api/v1/reports/schedules` with empty `recipients` list → 422
    - `POST /api/v1/reports/schedules` with invalid email in `recipients` → 422
    - `POST /api/v1/reports/schedules` with 21 recipients → 422 (max 20)
    - `GET /api/v1/reports/schedules` returns only the requesting company's schedules (tenant isolation: seed Company B schedule; assert it is NOT in response)
    - `PUT /api/v1/reports/schedules/{id}` updates `is_active=false`; DB row reflects change; audit log created
    - `PUT /api/v1/reports/schedules/{id}` for another company's schedule → 404
    - `DELETE /api/v1/reports/schedules/{id}` removes row; returns 204; audit log created
    - `DELETE /api/v1/reports/schedules/{id}` for another company's schedule → 404

24. **AC24 — Integration tests** in `services/notification/tests/test_scheduled_report_delivery.py`:
    - `check_scheduled_reports()` called with today mocked to Monday → finds weekly schedules → `report_jobs` row created for each; `celery_chain.apply_async` called (mock chain dispatch)
    - `check_scheduled_reports()` called with today mocked to 1st of month → finds monthly schedules → jobs dispatched
    - `check_scheduled_reports()` called with today mocked to a Tuesday non-1st → `frequencies_due` is empty; no DB writes; no task dispatch
    - `check_scheduled_reports()` with Monday AND 1st (e.g. 2026-06-01) → both weekly and monthly schedules dispatched
    - `check_scheduled_reports()` with no active schedules → completes without error, no tasks dispatched
    - `send_scheduled_report_email()` with a completed job → loads S3 bytes (mock `get_s3_client`), calls SendGrid client with correct recipients, attachment filename, and content type (mock `SendGridAPIClient`)
    - `send_scheduled_report_email()` with a job in `'processing'` state → raises `ValueError`, triggers retry
    - `send_scheduled_report_email()` with a deleted schedule (`schedule_id` not found) → logs warning and returns (does not raise)
    - SendGrid email body includes `download_url` from the job row
    - Attachment MIME type is `application/pdf` for PDF jobs and correct DOCX MIME type for DOCX jobs

---

## Tasks / Subtasks

### Infrastructure

- [x] Task 1: Alembic migration 014 — `client.report_schedules` table (AC: 1)
  - [x] 1.1 Create `services/client-api/alembic/versions/014_report_schedules.py` with `down_revision = "013"`
  - [x] 1.2 `upgrade()` — `CREATE TABLE IF NOT EXISTS client.report_schedules` with all columns, CHECK constraints, and `TEXT[]` for recipients
  - [x] 1.3 `upgrade()` — create 3 indexes: `ix_report_schedules_company_id`, `ix_report_schedules_company_active`, `ix_report_schedules_frequency`
  - [x] 1.4 `upgrade()` — `GRANT SELECT, INSERT, UPDATE, DELETE ON client.report_schedules TO client_api_role; GRANT SELECT ON client.report_schedules TO notification_role;`
  - [x] 1.5 `downgrade()` — drop indexes, then `DROP TABLE IF EXISTS client.report_schedules`
  - [x] 1.6 Verify `down_revision = "013"` matches the actual head from `alembic history` before committing

- [x] Task 2: SQLAlchemy Core model for `report_schedules` (AC: 2)
  - [x] 2.1 Create `services/client-api/src/client_api/models/report_schedule.py` — `sa.Table("report_schedules", metadata, ..., schema="client", info={"is_view": False})`; use `sa.ARRAY(sa.Text())` for `recipients`
  - [x] 2.2 Export `report_schedules` from `services/client-api/src/client_api/models/__init__.py`
  - [x] 2.3 Add `"report_schedules"` to `_EXCLUDED_TABLE_NAMES` in `services/client-api/alembic/env.py` (consistent with `report_jobs` from S12.09)

### Client API — Celery Producer

- [x] Task 3: Configure Celery producer in client-api (AC: 3)
  - [x] 3.1 Add `CELERY_BROKER_URL` field to `ClientApiSettings` in `services/client-api/src/client_api/config.py` (env: `CELERY_BROKER_URL`, default: `"redis://redis:6379/0"`)
  - [x] 3.2 Create `services/client-api/src/client_api/celery_producer.py` — `Celery("client_api_producer", broker=settings.CELERY_BROKER_URL)` singleton + `dispatch_generate_report()` function
  - [x] 3.3 Add `celery>=5.3` to `services/client-api/pyproject.toml` dependencies if not already present

### Client API — On-Demand Report Endpoints

- [x] Task 4: Implement on-demand report service (AC: 4, 5, 6)
  - [x] 4.1 Create `services/client-api/src/client_api/services/report_service.py`:
    - `create_report_job(session, user, payload) -> dict` — inserts report_jobs row, calls `dispatch_generate_report`, writes audit log, returns job dict
    - `get_report_job(session, user, job_id) -> dict | None` — queries with company_id filter; refreshes signed URL if expired
    - `list_report_jobs(session, user, page, page_size, status_filter) -> PaginatedResult` — company-scoped, ordered by `created_at DESC`
  - [x] 4.2 Implement signed URL refresh in `get_report_job`: if `url_expires_at < datetime.now(UTC)` and `status='complete'`, call `generate_presigned_url` (same pattern as S12.09), update `report_jobs.download_url` and `url_expires_at`

- [x] Task 5: Implement report schedule service (AC: 7, 8, 9, 10)
  - [x] 5.1 Create `services/client-api/src/client_api/services/report_schedule_service.py`:
    - `create_schedule(session, user, payload) -> dict` — inserts, audit log, returns schedule
    - `list_schedules(session, user) -> list[dict]` — company-scoped
    - `update_schedule(session, user, schedule_id, payload) -> dict | None` — update with `updated_at=NOW()`, audit log
    - `delete_schedule(session, user, schedule_id) -> bool` — hard delete, audit log; returns `False` if not found

- [x] Task 6: Create `services/client-api/src/client_api/routers/reports.py` (AC: 4–10)
  - [x] 6.1 Define `router = APIRouter(prefix="/api/v1/reports", tags=["reports"])`
  - [x] 6.2 `POST /` → 202 on-demand report creation (uses `create_report_job` service; paid-tier gate)
  - [x] 6.3 `GET /{job_id}` → job status (uses `get_report_job`; 404 if not found for company)
  - [x] 6.4 `GET /` → paginated report list (uses `list_report_jobs`)
  - [x] 6.5 `POST /schedules` → create schedule (uses `create_schedule`; admin-only gate via `require_company_admin`)
  - [x] 6.6 `GET /schedules` → list schedules (admin-only)
  - [x] 6.7 `PUT /schedules/{schedule_id}` → update schedule (admin-only; 404 if not found)
  - [x] 6.8 `DELETE /schedules/{schedule_id}` → 204 delete (admin-only; 404 if not found)
  - [x] 6.9 Register router in `services/client-api/src/client_api/main.py` (add `app.include_router(reports_router)`)

### Notification Service — Scheduled Delivery

- [x] Task 7: Create scheduled delivery tasks (AC: 11, 12)
  - [x] 7.1 Create `services/notification/src/notification/tasks/scheduled_report_delivery.py`
  - [x] 7.2 Implement `check_scheduled_reports()` task: determine due frequencies, query `report_schedules`, create `report_jobs` rows, dispatch celery chains
  - [x] 7.3 Implement `send_scheduled_report_email(job_id, schedule_id)` task: load job+schedule, download S3 bytes, compose SendGrid Mail with attachment, send
  - [x] 7.4 Import `report_jobs` and `report_schedules` SQLAlchemy table objects in the notification service context (these are in the `client_api` package — confirm `services/notification/pyproject.toml` has `eusolicit-models` or appropriate cross-service path, OR duplicate minimal inline `sa.Table` definitions; see Dev Notes)
  - [x] 7.5 Add `sendgrid>=6.11` to `services/notification/pyproject.toml` if not already present (should be there from E09 notifications)

- [x] Task 8: Register new tasks in Celery Beat (AC: 13)
  - [x] 8.1 Add `"notification.tasks.scheduled_report_delivery"` to `include` list in `services/notification/src/notification/workers/celery_app.py`
  - [x] 8.2 Add `"check-scheduled-reports-daily"` to `app.conf.beat_schedule` using `crontab(hour=6, minute=0)`
  - [x] 8.3 Verify existing beat schedule entries from S12.01 (MV refresh tasks) are preserved

### Frontend — API Client and Schemas

- [x] Task 9: Create `apps/client/lib/api/reports.ts` (AC: 14)
  - [x] 9.1 Define `ReportJob`, `ReportSchedule`, `CreateReportPayload`, `CreateSchedulePayload` TypeScript interfaces
  - [x] 9.2 Implement all 7 API functions using `apiClient` from `apps/client/lib/api/client.ts`
  - [x] 9.3 Ensure `createOnDemandReport` uses `POST /api/v1/reports` and `createReportSchedule` uses `POST /api/v1/reports/schedules` (different paths — do not confuse)

- [x] Task 10: Create `apps/client/lib/schemas/reports.ts` (AC: 15)
  - [x] 10.1 Define `generateReportSchema` with `superRefine` for `custom_date_range` date validation
  - [x] 10.2 Define `reportScheduleSchema` with email array validation
  - [x] 10.3 Export inferred types `GenerateReportFormValues`, `ReportScheduleFormValues`

### Frontend — Components and Pages

- [x] Task 11: Create `GenerateReportModal` in `packages/ui` (AC: 16)
  - [x] 11.1 Create `packages/ui/src/components/GenerateReportModal.tsx` with all props, polling logic, and `data-testid` attributes per AC16
  - [x] 11.2 Export `GenerateReportModal` from `packages/ui/src/index.ts`
  - [x] 11.3 The component is `"use client"` — mark with directive at top

- [x] Task 12: Create reports list page (AC: 17)
  - [x] 12.1 Create `apps/client/app/[locale]/(protected)/reports/page.tsx` (server component shell)
  - [x] 12.2 Create `apps/client/app/[locale]/(protected)/reports/components/ReportsList.tsx` with polling, `<QueryGuard>`, table, status badges, download links, and all `data-testid` attributes
  - [x] 12.3 Add `reports` route to sidebar navigation (in `packages/ui/src/components/Sidebar.tsx` or the app's navigation config)

- [x] Task 13: Create report schedule settings page (AC: 18)
  - [x] 13.1 Create `apps/client/app/[locale]/(protected)/settings/reports/page.tsx`
  - [x] 13.2 Create `apps/client/app/[locale]/(protected)/settings/reports/components/ReportScheduleSettings.tsx` with admin gate, schedules list, create form, toggle, delete
  - [x] 13.3 Add `settings/reports` link to Settings navigation

- [x] Task 14: Add "Generate Report" button to existing analytics pages (AC: 19)
  - [x] 14.1 Add `<GenerateReportModal>` + trigger `<Button data-testid="generate-report-btn">` to market analytics page component
  - [x] 14.2 Same for ROI, Team, Competitor, Pipeline, Usage pages
  - [x] 14.3 Use the `defaultReportType` mapping from AC19

- [x] Task 15: Add i18n keys (AC: 20)
  - [x] 15.1 Add all `reports.*` and `settings.reports.*` keys to `apps/client/messages/en.json`
  - [x] 15.2 Add same keys with Bulgarian translations to `apps/client/messages/bg.json`
  - [x] 15.3 Verify key-parity between BG and EN files (run key-parity test if available)

### Tests

- [x] Task 16: Unit tests for reports endpoints (AC: 22)
  - [x] 16.1 Create `services/client-api/tests/api/test_reports.py`
  - [x] 16.2 Implement all 9 test scenarios from AC22 (mock `celery_producer.send_task`)

- [x] Task 17: Unit tests for report schedule endpoints (AC: 23)
  - [x] 17.1 Create `services/client-api/tests/api/test_report_schedules.py`
  - [x] 17.2 Implement all 10 test scenarios from AC23

- [x] Task 18: Integration tests for scheduled delivery (AC: 24)
  - [x] 18.1 Create `services/notification/tests/test_scheduled_report_delivery.py`
  - [x] 18.2 Implement all 9 scenarios from AC24 (mock `date.today()`, mock `get_s3_client`, mock `SendGridAPIClient`, mock celery chain dispatch)

### Review Findings

**Review Date:** 2026-04-13
**Reviewer:** claude-opus-4-5 (Senior Developer Review)
**Verdict:** Changes Requested — 5 patch items, 2 deferred

#### Patch Items (must fix before merge)

- [x] [Review][Patch] **E12-R-001 tenant isolation gap in re-fetch after update** [`services/client-api/src/client_api/services/report_schedule_service.py:201-204`] — After updating a schedule, the re-fetch query uses only `WHERE id = schedule_id` without the mandatory `company_id` filter. Every other query in both report services enforces `company_id` as the first WHERE clause per E12-R-001. Fix: add `.where(report_schedules.c.company_id == user.company_id)` before `.where(report_schedules.c.id == schedule_id)` on the re-fetch.

- [x] [Review][Patch] **Date range inputs unreachable for custom_date_range** [`packages/ui/src/components/GenerateReportModal.tsx:251`] — Date inputs are gated solely by the `showDateRange` prop. None of the 6 analytics page integrations pass `showDateRange=true`. If a user selects `custom_date_range` from the report type dropdown, Zod validation rejects the form (date_from/date_to required) but no input fields are rendered. Fix: change condition to `{(showDateRange || form.watch("report_type") === "custom_date_range") && (…)}`.

- [x] [Review][Patch] **Report type labels hardcoded in English in modal** [`packages/ui/src/components/GenerateReportModal.tsx:177-182`] — The `reportTypeOptions` array uses hardcoded English strings. The `reports.types.*` i18n namespace exists per AC20. Fix: add a second `useTranslations("reports.types")` call and reference translated labels.

- [x] [Review][Patch] **Submit button uses wrong i18n key** [`apps/client/…/settings/reports/components/ReportScheduleSettings.tsx:361`] — Button label is `t("scheduleCreated")` → "Schedule created" (a success message, not a call-to-action). Fix: add a new i18n key `settings.reports.createButton` = "Create Schedule" (EN) / "Създай график" (BG) and use it for the idle button state.

- [x] [Review][Patch] **Missing success toast after schedule creation** [`apps/client/…/settings/reports/components/ReportScheduleSettings.tsx:121-131`] — AC18 requires "show success toast" on submit. The `createMutation.onSuccess` invalidates the query cache and resets the form but does not trigger a toast. Fix: add `toast.success(t("scheduleCreated"))` (or equivalent) in `onSuccess`.

#### Deferred Items (pre-existing patterns, not blocking)

- [x] [Review][Defer] **Hardcoded English strings in ReportScheduleSettings** [`apps/client/…/settings/reports/components/ReportScheduleSettings.tsx`] — "Confirm", "Cancel", "Failed to create schedule…", "Loading schedules…", "Failed to load schedules.", and placeholder text are hardcoded English. These follow a pre-existing pattern in the codebase where secondary UI text is not fully translated. — deferred, pre-existing pattern
- [x] [Review][Defer] **Schedule form uses local schema without email validation** [`apps/client/…/settings/reports/components/ReportScheduleSettings.tsx:62-74`] — AC18 specifies `useZodForm(reportScheduleSchema)` (shared schema with per-email `.email()` validation). Implementation uses a local schema with `recipients_raw: z.string()` — no client-side email format check. Server validates (AC7: 422 on invalid email), so this is functional. — deferred, UX improvement

---

## Dev Notes

### Migration 014 Placement

Confirm with `alembic history` in the `services/client-api` directory that `013` is the current head before creating `014`. If another story has introduced a migration between 013 and 014, adjust `down_revision` accordingly.

### SQLAlchemy `ARRAY` Column for Recipients

PostgreSQL `TEXT[]` maps to SQLAlchemy `sa.ARRAY(sa.Text())`:

```python
sa.Column("recipients", sa.ARRAY(sa.Text()), nullable=False, server_default="{}")
```

When inserting via Core, pass `recipients` as a Python `list[str]`. asyncpg handles array serialization automatically.

### Celery Producer — Why `send_task()` and Not Direct Import

The `generate_report` task lives in the notification service codebase. The client-api codebase cannot import notification service modules (different Python package). Using `celery_producer.send_task("notification.tasks.report_generation.generate_report", args=[...])` sends the task by name over the shared Redis broker without needing to import the task module. This is the correct pattern for cross-service Celery dispatch.

### Celery Chain for Scheduled Reports

The `celery.chain` (imported as `celery_chain` to avoid shadowing builtins) chains `generate_report` and `send_scheduled_report_email`. The `.si()` "immutable signature" call ensures the chained task does NOT receive the previous task's return value as its first argument — both tasks take explicit `job_id` parameters:

```python
from celery import chain as celery_chain
from notification.tasks.report_generation import generate_report
from notification.tasks.scheduled_report_delivery import send_scheduled_report_email

celery_chain(
    generate_report.si(job_id, company_id_str, report_type, format_str),
    send_scheduled_report_email.si(job_id, schedule_id_str),
).apply_async()
```

### Accessing `report_schedules` Table from Notification Service

The `report_schedules` table is defined in the client-api package. The notification service needs to read it for `check_scheduled_reports`. Options:

**Option A (preferred for MVP):** Define a minimal inline `sa.Table` in `scheduled_report_delivery.py` that mirrors the schema:
```python
_metadata = sa.MetaData()
_report_schedules = sa.Table(
    "report_schedules", _metadata, schema="client",
    sa.Column("id", sa.UUID),
    sa.Column("company_id", sa.UUID),
    sa.Column("created_by_id", sa.UUID),
    sa.Column("report_type", sa.Text),
    sa.Column("format", sa.Text),
    sa.Column("frequency", sa.Text),
    sa.Column("recipients", sa.ARRAY(sa.Text())),
    sa.Column("is_active", sa.Boolean),
)
```
This avoids a cross-package import dependency. The notification service already does this pattern for `report_jobs` (from S12.09's task, which queries `client.report_jobs`).

**Option B:** Move shared table definitions to `eusolicit-models` package. This is the architectural preference for long-term but is additional work beyond this story's scope.

Use Option A for S12.10. Note Option B as deferred work in `deferred-work.md`.

### SendGrid Integration in Notification Service

From E09 (notifications), `NotificationSettings` should already have:
- `SENDGRID_API_KEY: str`
- `SENDGRID_FROM_EMAIL: str`

If these are missing (E09 not yet implemented or merged), add them to `services/notification/src/notification/config.py`:
```python
SENDGRID_API_KEY: str = Field(env="SENDGRID_API_KEY")
SENDGRID_FROM_EMAIL: str = Field(default="reports@eusolicit.com", env="SENDGRID_FROM_EMAIL")
```

Email composition with attachment:
```python
import base64
from sendgrid import SendGridAPIClient
from sendgrid.helpers.mail import (
    Mail, To, Attachment, FileContent, FileName, FileType, Disposition
)

def build_report_email(recipients, subject, body_text, report_bytes, filename, content_type):
    message = Mail(
        from_email=settings.SENDGRID_FROM_EMAIL,
        to_emails=[To(r) for r in recipients],
        subject=subject,
        plain_text_content=body_text,
    )
    attachment = Attachment(
        file_content=FileContent(base64.b64encode(report_bytes).decode()),
        file_name=FileName(filename),
        file_type=FileType(content_type),
        disposition=Disposition("attachment"),
    )
    message.attachment = attachment
    return message
```

Email body template:
```
EU Solicit Report — {report_type_label}
Generated: {completed_at}
Period: {date_from} to {date_to}   (omit if None)

Your report is attached to this email.
You can also download it here (valid for 24 hours):
{download_url}

—
EU Solicit Platform
```

### Report Type Labels for Email Subject

Map `report_type` to a human-readable label for email subjects:
```python
REPORT_TYPE_LABELS = {
    "pipeline_summary": "Pipeline Summary",
    "bid_performance_summary": "Bid Performance Summary",
    "team_activity": "Team Activity",
    "custom_date_range": "Analytics Report",
}
```

### Celery Beat App File Location

**IMPORTANT:** S12.09's Dev Notes refer to the Celery app at `services/notification/src/notification/celery_app.py` but S12.09's File List (the authoritative source for what was actually created) says the file is `services/notification/src/notification/workers/celery_app.py`. Verify the actual file path in the codebase before modifying:
```bash
find services/notification -name "celery_app.py"
```
Use whichever file actually exists. If both exist, use the one referenced in S12.01's implementation (older story).

### Synchronous DB Access in Notification Tasks

All notification service Celery tasks use synchronous SQLAlchemy sessions (not async). Use `get_sync_session()` — the same pattern as `generate_report` from S12.09:

```python
from notification.db import get_sync_session

with get_sync_session() as session:
    # sync queries here
    session.commit()
```

### Admin Role Check in Client API

Company admin check should use the existing RBAC pattern from the project (established in E02/E10). Use `require_company_admin` dependency or check inline:

```python
from client_api.dependencies.auth import get_current_user

async def create_schedule(
    payload: ScheduleCreateRequest,
    current_user: User = Depends(get_current_user),
    session: AsyncSession = Depends(get_db),
):
    if current_user.company_role not in ("admin",):
        raise ForbiddenError("Company admin role required")
    ...
```

Pattern: check `company_role` attribute on the authenticated user object. The RBAC hierarchy is `admin > bid_manager > contributor > reviewer > read_only`. Only `admin` can manage report schedules.

### Paid Tier Gate for On-Demand Reports

Use the same `require_paid_tier` dependency already established for analytics endpoints (e.g., in S12.08 `GET /analytics/usage`). This gates Free tier users with 403 + `{"upgrade_required": true}`.

### TanStack Query Polling for Report Status

In `GenerateReportModal`, `refetchInterval` as a function (not a number) is the TanStack Query v5 pattern for conditional polling:

```typescript
refetchInterval: (query) => {
  const status = query.state.data?.status;
  if (!status || status === 'pending' || status === 'processing') return 2000;
  return false;  // stop polling when complete or failed
},
```

Do NOT use a fixed 2000ms interval — this would continue polling indefinitely even after the job finishes.

### Query Invalidation After On-Demand Report Creation

After submitting a report generation in the modal, invalidate the `['reports']` query so the reports list page refreshes to show the new pending job:

```typescript
const queryClient = useQueryClient();
// after createOnDemandReport() resolves:
queryClient.invalidateQueries({ queryKey: ['reports'] });
```

### P0 Test Coverage Context

From `test-design-epic-12.md`, the S12.09 P0 test is:
> "Async report Celery task completes; S3 object exists; signed URL returned" (Integration, R12.7)

The `GET /api/v1/reports/{job_id}` endpoint implemented in **this story (S12.10)** is the HTTP interface that the Playwright E2E `recurse` polling pattern uses for the P1 E2E tests. The P0 integration test for S12.09 invokes `generate_report` directly (bypassing HTTP) — but the Playwright E2E layer for on-demand reports (S12.10 P1 test) uses `recurse` to poll `GET /api/v1/reports/{job_id}` until `status='complete'`.

### Test Design Coverage for S12.10

From `test-design-epic-12.md` (S12.10 P1 section — 9 tests total):

| Scenario | Test Level | Coverage |
|---|---|---|
| Admin configures weekly/monthly schedule (POST + GET confirm) | API | Tasks 17.1–17.2 |
| Celery Beat triggers scheduled report + SendGrid mock | Integration | Tasks 18.1–18.2 |
| On-demand job ID returned + `recurse` polling to 'complete' | E2E/API | Playwright RED tests in `atdd-checklist-e12-p0.md` |
| Download link visible in-app when ready | E2E | Playwright test (`data-testid="report-download-link"`) |
| Reports list page: past reports, timestamps, download links | E2E | Playwright test (`data-testid="reports-table"`) |
| Email with attachment (SendGrid mock, MIME type) | Integration | Tasks 18.1–18.2 |

P2 scenario: "Signed S3 download URL expires after 24h" → covered by the URL-refresh logic in `get_report_job` (AC5) + test in AC22 (expired URL triggers S3 presign call).

### Scope Boundaries

This story (S12.10) delivers:
- `client.report_schedules` table + model
- Client API: on-demand report job creation + status polling + list + schedule CRUD
- Notification service: scheduled delivery task + email dispatch
- Frontend: `GenerateReportModal`, reports list page, schedule settings page, analytics page integrations

**Out of scope (future stories):**
- S12.11–S12.13: Admin API endpoints
- S12.14: Admin frontend pages
- S12.15–S12.16: Enterprise API
- WebSocket-based real-time status updates (polling is used instead per epic spec)

### Project Structure Notes

- **Notification service Celery tasks** → `services/notification/src/notification/tasks/` (established in S12.01, S12.09)
- **Client API routers** → `services/client-api/src/client_api/routers/` (follow naming like `analytics.py`, `team.py`)
- **Client API services** → `services/client-api/src/client_api/services/` (follows pattern from E07, E11 stories)
- **Frontend pages** → `apps/client/app/[locale]/(protected)/` (all protected routes under this layout)
- **Shared UI components** → `packages/ui/src/components/` (barrel-exported from `packages/ui/src/index.ts`)
- **Frontend API functions** → `apps/client/lib/api/` (one file per domain: `analytics.ts`, `reports.ts`)
- **Frontend Zod schemas** → `apps/client/lib/schemas/` (one file per domain)

### References

- [Source: epic-12-analytics-admin-platform.md#S12.10] — Story requirements, ACs, scope
- [Source: implementation-artifacts/12-9-report-generation-engine-pdf-docx.md] — `report_jobs` table schema, `generate_report` task signature, S3 pattern, sync session pattern
- [Source: test-artifacts/test-design-epic-12.md#S12.10] — P1 test scenarios (9 tests), P2 test (URL expiry)
- [Source: test-artifacts/atdd-checklist-e12-p0.md] — P0 ATDD tests (RED phase) for E12; `recurse` polling pattern
- [Source: eusolicit-docs/project-context.md] — Tech stack, RBAC, audit trail, frontend patterns, `useZodForm`, `QueryGuard`, `apiClient`
- [Source: implementation-artifacts/12-8-usage-dashboard-full-stack.md] — Pattern for fullstack analytics story (AC structure, frontend data-testid conventions, paid-tier gate, `require_paid_tier` dependency)

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

None.

### Completion Notes List

- Implemented all 18 tasks covering AC1–AC24 in full autopilot mode.
- Router placed at `src/client_api/api/v1/reports.py` (not `routers/reports.py`) to match the project's existing `api/v1/` structure; registered via `api_v1_router.include_router()` in `main.py`.
- Endpoint ordering in the router is critical: all static `/schedules` routes registered before `/{job_id}` to prevent the parameterised route capturing `/schedules` requests.
- `_require_admin = require_role("admin")` stored at module level in `reports.py` so tests can patch it via `reports_module._require_admin`.
- `recipients` uses `sa.ARRAY(sa.Text())` in the PostgreSQL model and `sa.JSON()` in SQLite-based integration tests (SQLite has no ARRAY support).
- Celery `celery_broker_url` field uses `env="CELERY_BROKER_URL"` (no prefix) to receive the bare `CELERY_BROKER_URL` env var, consistent with AC3 spec.
- Notification service uses Option A (inline `sa.Table` definitions) for accessing `report_schedules` from the scheduled delivery task, consistent with `report_generation.py` pattern.

**Review Patch Fixes (2026-04-13):**
- ✅ Resolved review finding [Patch]: E12-R-001 tenant isolation gap — added `company_id` filter to re-fetch query in `update_schedule` (`report_schedule_service.py`).
- ✅ Resolved review finding [Patch]: Date range inputs now shown when `report_type === "custom_date_range"` even without `showDateRange` prop (`GenerateReportModal.tsx`).
- ✅ Resolved review finding [Patch]: Report type labels now use `useTranslations("reports.types")` instead of hardcoded English strings (`GenerateReportModal.tsx`).
- ✅ Resolved review finding [Patch]: Added `createButton` i18n key ("Create Schedule" / "Създай график"); button now uses `t("createButton")` and `scheduleCreated` restored to "Schedule created" success message (`ReportScheduleSettings.tsx`, `en.json`, `bg.json`).
- ✅ Resolved review finding [Patch]: Added `toast.success(t("scheduleCreated"))` in `createMutation.onSuccess` (`ReportScheduleSettings.tsx`).
- Fixed pre-existing test bugs: `job_id = uuid4()` (UUID object) for DB insert in `check_scheduled_reports`; string-to-UUID conversion in `send_scheduled_report_email` queries; replaced deferred `from notification.tasks.report_generation import generate_report` with `celery_signature()` by name to avoid matplotlib import chain in tests; removed `date-fns` dependency from `ReportsList.tsx` (not installed); fixed `EmptyState` missing `icon` prop in `ReportsList.tsx`.
- All 29 tests pass: 19 client-api (AC22 + AC23) + 10 notification (AC24).
- Analytics dashboard components: `user?.role === 'admin'` used (not `companyRole`) to match the actual auth store `User` interface in the codebase.
- SendGrid settings (`sendgrid_api_key`, `sendgrid_from_email`) added to `NotificationSettings` with safe defaults.
- Beat schedule entry fires at `crontab(hour=6, minute=0)` UTC daily; date logic in task determines which frequencies are due.
- `GenerateReportModal` uses injected `createReport`/`getReportJob` props to avoid circular imports between `packages/ui` and `apps/client`.
- All 9 AC22 tests (on-demand endpoints) and all 10 AC23 tests (schedule endpoints) implemented with fixture-based dependency override pattern.
- All 9 AC24 integration tests implemented with SQLite in-memory engine + module-level patch pattern matching `test_report_generation.py`.

### File List

#### New Files

**Backend:**
- `services/client-api/alembic/versions/014_report_schedules.py` — Migration 014: `client.report_schedules` table, 3 indexes, GRANTs
- `services/client-api/src/client_api/models/report_schedule.py` — SQLAlchemy Core Table definition
- `services/client-api/src/client_api/celery_producer.py` — Celery producer singleton + `dispatch_generate_report()`
- `services/client-api/src/client_api/services/report_service.py` — on-demand report creation, status, list
- `services/client-api/src/client_api/services/report_schedule_service.py` — CRUD for report schedules
- `services/client-api/src/client_api/api/v1/reports.py` — 7 endpoints: on-demand CRUD + schedule CRUD
- `services/client-api/tests/api/test_reports.py` — AC22 unit tests (9 tests)
- `services/client-api/tests/api/test_report_schedules.py` — AC23 unit tests (10 tests)
- `services/notification/src/notification/tasks/scheduled_report_delivery.py` — `check_scheduled_reports` + `send_scheduled_report_email` tasks
- `services/notification/tests/test_scheduled_report_delivery.py` — AC24 integration tests (9 tests)

**Frontend:**
- `apps/client/lib/api/reports.ts` — API client functions for reports and schedules
- `apps/client/lib/schemas/reports.ts` — Zod schemas for generate-report and schedule forms
- `packages/ui/src/components/GenerateReportModal.tsx` — shared modal with polling
- `apps/client/app/[locale]/(protected)/reports/page.tsx` — reports list page (server shell)
- `apps/client/app/[locale]/(protected)/reports/components/ReportsList.tsx` — list component with auto-refresh
- `apps/client/app/[locale]/(protected)/settings/reports/page.tsx` — schedule settings page (server shell)
- `apps/client/app/[locale]/(protected)/settings/reports/components/ReportScheduleSettings.tsx` — schedule management form + list

#### Modified Files

- `services/client-api/src/client_api/models/__init__.py` — export `report_schedules`
- `services/client-api/alembic/env.py` — add `"report_schedules"` to `_EXCLUDED_TABLE_NAMES`
- `services/client-api/src/client_api/config.py` — add `celery_broker_url` + S3/AWS settings
- `services/client-api/src/client_api/main.py` — register `reports_v1.router`
- `services/client-api/pyproject.toml` — add `celery>=5.3` and `boto3>=1.34`
- `services/notification/src/notification/workers/celery_app.py` — add `scheduled_report_delivery` to `include` list
- `services/notification/src/notification/workers/beat_schedule.py` — add `check-scheduled-reports-daily` entry
- `services/notification/src/notification/config.py` — add `sendgrid_api_key` and `sendgrid_from_email` fields
- `packages/ui/index.ts` — export `GenerateReportModal` and `GenerateReportModalProps`
- `apps/client/messages/en.json` — add `reports.*` and `settings.reports.*` keys
- `apps/client/messages/bg.json` — add same keys with BG translations
- `apps/client/app/[locale]/(protected)/analytics/market/components/MarketIntelligenceDashboard.tsx` — add `GenerateReportModal` trigger
- `apps/client/app/[locale]/(protected)/analytics/roi/components/RoiTrackerDashboard.tsx` — add `GenerateReportModal` trigger
- `apps/client/app/[locale]/(protected)/analytics/team/components/TeamPerformanceDashboard.tsx` — add `GenerateReportModal` trigger
- `apps/client/app/[locale]/(protected)/analytics/competitor/components/CompetitorIntelligenceDashboard.tsx` — add `GenerateReportModal` trigger
- `apps/client/app/[locale]/(protected)/analytics/pipeline/components/PipelineForecastDashboard.tsx` — add `GenerateReportModal` trigger
- `apps/client/app/[locale]/(protected)/analytics/usage/components/UsageDashboard.tsx` — add `GenerateReportModal` trigger

### Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-04-13 | Full implementation: Tasks 1–18 completed in autopilot mode | claude-sonnet-4-5 |
| 2026-04-13 | Addressed code review findings — 5 patch items resolved: tenant isolation re-fetch, date range visibility, i18n report type labels, createButton key, success toast; also fixed pre-existing test bugs (UUID handling, matplotlib import chain, date-fns dependency, EmptyState icon prop) | claude-sonnet-4-5 |

#### New Files

**Backend:**
- `services/client-api/alembic/versions/014_report_schedules.py` — Migration 014: `client.report_schedules` table, 3 indexes, GRANTs
- `services/client-api/src/client_api/models/report_schedule.py` — SQLAlchemy Core Table definition
- `services/client-api/src/client_api/celery_producer.py` — Celery producer singleton + `dispatch_generate_report()`
- `services/client-api/src/client_api/services/report_service.py` — on-demand report creation, status, list
- `services/client-api/src/client_api/services/report_schedule_service.py` — CRUD for report schedules
- `services/client-api/src/client_api/routers/reports.py` — 7 endpoints: on-demand CRUD + schedule CRUD
- `services/client-api/tests/api/test_reports.py` — AC22 unit tests (9 tests)
- `services/client-api/tests/api/test_report_schedules.py` — AC23 unit tests (10 tests)
- `services/notification/src/notification/tasks/scheduled_report_delivery.py` — `check_scheduled_reports` + `send_scheduled_report_email` tasks
- `services/notification/tests/test_scheduled_report_delivery.py` — AC24 integration tests (9 tests)

**Frontend:**
- `apps/client/lib/api/reports.ts` — API client functions for reports and schedules
- `apps/client/lib/schemas/reports.ts` — Zod schemas for generate-report and schedule forms
- `packages/ui/src/components/GenerateReportModal.tsx` — shared modal with polling
- `apps/client/app/[locale]/(protected)/reports/page.tsx` — reports list page (server shell)
- `apps/client/app/[locale]/(protected)/reports/components/ReportsList.tsx` — list component with auto-refresh
- `apps/client/app/[locale]/(protected)/settings/reports/page.tsx` — schedule settings page (server shell)
- `apps/client/app/[locale]/(protected)/settings/reports/components/ReportScheduleSettings.tsx` — schedule management form + list

#### Modified Files

- `services/client-api/src/client_api/models/__init__.py` — export `report_schedules`
- `services/client-api/alembic/env.py` — add `"report_schedules"` to `_EXCLUDED_TABLE_NAMES`
- `services/client-api/src/client_api/config.py` — add `CELERY_BROKER_URL`
- `services/client-api/src/client_api/main.py` — register `reports_router`
- `services/client-api/pyproject.toml` — add `celery>=5.3` if not present
- `services/notification/src/notification/workers/celery_app.py` — add `scheduled_report_delivery` to `include`, add beat schedule entry
- `services/notification/pyproject.toml` — add `sendgrid>=6.11` if not present
- `packages/ui/src/index.ts` — export `GenerateReportModal`
- `apps/client/messages/en.json` — add `reports.*` and `settings.reports.*` keys
- `apps/client/messages/bg.json` — add same keys with BG translations
- Analytics page components (6 files) — add `<GenerateReportModal>` trigger button per AC19
