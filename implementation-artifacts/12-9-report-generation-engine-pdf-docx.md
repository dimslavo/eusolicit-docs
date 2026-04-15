# Story 12.9: Report Generation Engine (PDF & DOCX)

Status: review-approved

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer on the EU Solicit platform**,
I want **a Celery-backed report generation engine that produces formatted PDF (reportlab) and DOCX (python-docx) documents for four report types — pipeline summary, bid performance summary, team activity, and custom date range — stores output in S3, and returns a signed 24-hour download URL**,
so that **the scheduled and on-demand report delivery stories (S12.10) have a reliable, template-driven document generation foundation to build on, and the existing proposal export (E07/S07.10) is refactored to reuse the same shared rendering stack**.

## Acceptance Criteria

### Infrastructure — `client.report_jobs` Table

1. **AC1 — Alembic migration 013** creates the `client.report_jobs` table. Migration file: `services/client-api/alembic/versions/013_report_jobs.py` with `down_revision = "012"`. The `upgrade()` creates:

   ```sql
   CREATE TABLE IF NOT EXISTS client.report_jobs (
       id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
       company_id        UUID NOT NULL REFERENCES client.companies(id) ON DELETE CASCADE,
       requested_by_id   UUID NOT NULL REFERENCES client.users(id),
       report_type       TEXT NOT NULL CHECK (report_type IN (
                             'pipeline_summary', 'bid_performance_summary',
                             'team_activity', 'custom_date_range')),
       format            TEXT NOT NULL CHECK (format IN ('pdf', 'docx')),
       status            TEXT NOT NULL DEFAULT 'pending'
                             CHECK (status IN ('pending', 'processing', 'complete', 'failed')),
       s3_key            TEXT,
       download_url      TEXT,
       url_expires_at    TIMESTAMPTZ,
       date_from         DATE,
       date_to           DATE,
       error_message     TEXT,
       created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
       completed_at      TIMESTAMPTZ
   );
   ```

   Followed by:
   ```sql
   CREATE INDEX ix_report_jobs_company_id ON client.report_jobs (company_id);
   CREATE INDEX ix_report_jobs_company_status ON client.report_jobs (company_id, status);
   CREATE INDEX ix_report_jobs_created_at ON client.report_jobs (created_at DESC);
   ```

   Permission grants in `upgrade()`:
   ```sql
   GRANT SELECT, INSERT, UPDATE ON client.report_jobs TO client_api_role;
   GRANT SELECT, INSERT, UPDATE ON client.report_jobs TO notification_role;
   ```

   The `downgrade()` drops the three indexes then `DROP TABLE IF EXISTS client.report_jobs`. Migration is fully reversible.

2. **AC2 — SQLAlchemy Core Table** definition added to `services/client-api/src/client_api/models/report_job.py`. Define `report_jobs` as an `sa.Table` with `schema="client"` and `info={"is_view": False}`. Export from `services/client-api/src/client_api/models/__init__.py`. Columns mirror AC1 exactly.

---

### Shared Document Generation Module

3. **AC3 — `eusolicit_common.document_generation` package** is created at `packages/eusolicit-common/src/eusolicit_common/document_generation/`. The package exposes:

   - `__init__.py` — re-exports: `render_pdf`, `render_docx`, `ReportSection`, `HeaderSection`, `TextSection`, `TableSection`, `ChartSection`, `generate_chart_png`
   - `models.py` — Pydantic v2 models:

     ```python
     class ReportMetadata(BaseModel):
         title: str
         subtitle: str | None = None
         company_name: str
         generated_at: datetime
         date_from: date | None = None
         date_to: date | None = None

     class HeaderSection(BaseModel):
         type: Literal["header"] = "header"
         level: Literal[1, 2, 3] = 1
         text: str

     class TextSection(BaseModel):
         type: Literal["text"] = "text"
         body: str

     class TableSection(BaseModel):
         type: Literal["table"] = "table"
         caption: str | None = None
         columns: list[str]          # column headers
         rows: list[list[str]]       # each row is a list of cell strings

     class ChartSection(BaseModel):
         type: Literal["chart"] = "chart"
         caption: str | None = None
         png_bytes: bytes            # pre-rendered PNG image bytes

     ReportSection = Annotated[
         HeaderSection | TextSection | TableSection | ChartSection,
         Field(discriminator="type")
     ]

     class ReportDocument(BaseModel):
         metadata: ReportMetadata
         sections: list[ReportSection]
     ```

   - `pdf_renderer.py` — `render_pdf(doc: ReportDocument) -> bytes`:
     Uses reportlab `SimpleDocTemplate` + `Platypus` flowables.
     - Title page: company name (Heading1), report title (Heading2), generated_at date, optional date range
     - `HeaderSection(level=1)` → `Paragraph(text, styles["Heading1"])`; level 2/3 → `Heading2`/`Heading3`
     - `TextSection` → `Paragraph(body, styles["Normal"])`
     - `TableSection` → `Table([columns] + rows)` with `TableStyle` grid, header row shaded `#4F81BD`
     - `ChartSection` → `Image(BytesIO(png_bytes), width=400, height=250)` with optional caption paragraph
     - Page numbers added via `PageTemplate` with frame and `onLaterPages` callback
     - Returns PDF bytes; raises `DocumentRenderError` on failure

   - `docx_renderer.py` — `render_docx(doc: ReportDocument) -> bytes`:
     Uses `python_docx.Document` with default styles.
     - Title page: `document.add_heading(company_name, 0)`, `document.add_heading(title, 1)`, date paragraph
     - `HeaderSection` → `document.add_heading(text, level)`
     - `TextSection` → `document.add_paragraph(body)`
     - `TableSection` → `document.add_table(rows=len(rows)+1, cols=len(columns))` with header row bold + shaded `#4F81BD`
     - `ChartSection` → `document.add_picture(BytesIO(png_bytes), width=Inches(5.5))` with optional caption
     - Returns DOCX bytes via `BytesIO`; raises `DocumentRenderError` on failure

   - `charts.py` — `generate_chart_png(chart_type: str, labels: list[str], values: list[float], title: str = "", xlabel: str = "", ylabel: str = "") -> bytes`:
     Uses `matplotlib.pyplot` with `Agg` backend (no display).
     - `chart_type = "bar"` → horizontal bar chart
     - `chart_type = "line"` → line chart with markers
     - `chart_type = "pie"` → pie chart with percentage labels
     - Returns PNG bytes via `BytesIO`; always closes figure to avoid memory leak
     - Thread-safe: uses `matplotlib.use("Agg")` at module import

   - `exceptions.py` — `DocumentRenderError(Exception)` with message field

4. **AC4 — E07 proposal export refactored**: The existing proposal document export in `services/client-api/src/client_api/utils/document_export.py` (or equivalent) is updated to import `render_pdf`, `render_docx`, and `generate_chart_png` from `eusolicit_common.document_generation` rather than using inline reportlab/python-docx logic. The E07 export behaviour is preserved (branding, section headers, page numbers, TOC); only the rendering backend moves to the shared module. A unit test asserts that `document_export.py` imports from `eusolicit_common.document_generation`.

   > **Note**: If the E07 export has not yet been implemented (S07.10 may not yet be done), create a stub `document_export.py` that imports from the shared module so the import assertion test passes. The full E07 implementation is out of scope for this story.

---

### Report Templates (Data Assemblers)

Each template is a pure function that accepts analytics data (Python dicts from MV queries) and returns a `ReportDocument`. Templates live in `services/notification/src/notification/report_templates/`.

5. **AC5 — Pipeline Summary template** — `services/notification/src/notification/report_templates/pipeline_summary.py`:

   Function signature:
   ```python
   def build_pipeline_summary(
       company_name: str,
       date_from: date | None,
       date_to: date | None,
       forecast_rows: list[dict],   # rows from mv_competitor_intelligence or pipeline_predictions
       generated_at: datetime,
   ) -> ReportDocument:
   ```

   Produces a `ReportDocument` with:
   - Metadata: title `"Pipeline Summary Report"`, company_name, generated_at, date_from, date_to
   - `HeaderSection(level=1, text="Pipeline Summary")`
   - `TextSection` with date range description
   - `TableSection` — columns: `["Opportunity", "Sector", "Est. Value", "Predicted Date", "Confidence"]` — rows from `forecast_rows` (each row mapped to string cells; confidence formatted as `"High"` / `"Medium"` / `"Low"`)
   - `ChartSection` — bar chart of opportunity count per sector (generated via `generate_chart_png`)
   - Handles empty `forecast_rows` gracefully: emits `TextSection` "No pipeline predictions available for the selected period." instead of empty table

6. **AC6 — Bid Performance Summary template** — `services/notification/src/notification/report_templates/bid_performance_summary.py`:

   Function signature:
   ```python
   def build_bid_performance_summary(
       company_name: str,
       date_from: date | None,
       date_to: date | None,
       roi_rows: list[dict],       # rows from mv_roi_tracker
       generated_at: datetime,
   ) -> ReportDocument:
   ```

   Produces:
   - Metadata: title `"Bid Performance Summary"`, company_name, generated_at
   - `HeaderSection(level=1, text="Bid Performance Summary")`
   - Summary `TextSection`: total bids, total invested (formatted currency €), total won (€), aggregate ROI %
   - `TableSection` — columns: `["Bid / Proposal", "Invested (€)", "Won (€)", "ROI (%)"]` — rows from `roi_rows`
   - `ChartSection` — line chart of ROI % over time (monthly buckets from `roi_rows`)
   - Empty state handled as in AC5

7. **AC7 — Team Activity template** — `services/notification/src/notification/report_templates/team_activity.py`:

   Function signature:
   ```python
   def build_team_activity(
       company_name: str,
       date_from: date | None,
       date_to: date | None,
       team_rows: list[dict],      # rows from mv_team_performance
       generated_at: datetime,
   ) -> ReportDocument:
   ```

   Produces:
   - Metadata: title `"Team Activity Report"`, company_name, generated_at
   - `HeaderSection(level=1, text="Team Activity")`
   - `TableSection` — columns: `["Team Member", "Bids Submitted", "Win Rate (%)", "Avg. Prep Time (hrs)", "Proposals Generated"]` — rows from `team_rows`
   - `ChartSection` — bar chart of bids submitted per team member

8. **AC8 — Custom Date Range template** — `services/notification/src/notification/report_templates/custom_date_range.py`:

   Function signature:
   ```python
   def build_custom_date_range(
       company_name: str,
       date_from: date,
       date_to: date,
       market_rows: list[dict],    # from mv_market_intelligence
       roi_rows: list[dict],       # from mv_roi_tracker
       team_rows: list[dict],      # from mv_team_performance
       generated_at: datetime,
   ) -> ReportDocument:
   ```

   Produces a multi-section report combining Market, ROI, and Team data for the given date range:
   - Metadata: title `"Analytics Report"` with subtitle `f"{date_from} to {date_to}"`
   - Three major sections each with a `HeaderSection(level=2)`, summary `TextSection`, and a compact `TableSection`
   - One `ChartSection` per section (market volume bar, ROI trend line, team bids bar)

---

### Celery Task

9. **AC9 — `generate_report` Celery task** in `services/notification/src/notification/tasks/report_generation.py`:

   ```python
   @celery_app.task(
       name="notification.tasks.report_generation.generate_report",
       bind=True,
       max_retries=3,
       default_retry_delay=30,
       acks_late=True,
   )
   def generate_report(
       self,
       job_id: str,          # UUID string of the report_jobs row
       company_id: str,
       report_type: str,
       format: str,          # "pdf" or "docx"
       date_from: str | None = None,   # ISO date string or None
       date_to: str | None = None,
   ) -> None:
   ```

   Task execution flow:
   - **Step 1 — Mark processing**: `UPDATE client.report_jobs SET status = 'processing' WHERE id = :job_id`
   - **Step 2 — Query analytics data**: run the appropriate materialized view query for `company_id` (see Dev Notes for query details per template). Date range filtering applied when `date_from`/`date_to` are provided.
   - **Step 3 — Assemble ReportDocument**: call the appropriate template builder function (AC5–AC8) with queried data
   - **Step 4 — Render document**: call `render_pdf(doc)` or `render_docx(doc)` based on `format`
   - **Step 5 — Upload to S3**: upload bytes to key `reports/{company_id}/{report_type}/{job_id}.{format}` (see Dev Notes for S3 client setup)
   - **Step 6 — Generate signed URL**: `s3_client.generate_presigned_url("get_object", Params={"Bucket": bucket, "Key": s3_key}, ExpiresIn=86400)`
   - **Step 7 — Mark complete**: `UPDATE client.report_jobs SET status = 'complete', s3_key = :s3_key, download_url = :signed_url, url_expires_at = :expiry, completed_at = NOW() WHERE id = :job_id`
   - **On exception**: catch all exceptions; `UPDATE client.report_jobs SET status = 'failed', error_message = str(e), completed_at = NOW() WHERE id = :job_id`; then `self.retry(exc=e)` (up to `max_retries=3`). After final retry exhaustion, status remains `'failed'`; the exception is NOT re-raised to Celery's default handler (log and suppress).

   The task uses a synchronous SQLAlchemy `Session` (not async) for simplicity in Celery context, consistent with other notification service tasks.

---

### S3 Storage & Signed URL

10. **AC10 — S3 upload and signed URL** behaviour:
    - S3 bucket name read from `settings.REPORT_S3_BUCKET` (env var `NOTIFICATION_REPORT_S3_BUCKET`)
    - S3 key pattern: `reports/{company_id}/{report_type}/{job_id}.{format}` (e.g. `reports/abc-123/pipeline_summary/xyz-789.pdf`)
    - `ContentType` set on upload: `application/pdf` for PDF, `application/vnd.openxmlformats-officedocument.wordprocessingml.document` for DOCX
    - Signed URL expires in **86400 seconds** (24 hours). `url_expires_at` stored as `NOW() + INTERVAL '24 hours'` in `report_jobs`
    - In local development (LocalStack), the `AWS_ENDPOINT_URL` env var overrides the S3 endpoint. The boto3 client reads this automatically if configured in settings (see Dev Notes)
    - The S3 upload uses `put_object` (not multipart) — reports are expected to be < 50 MB

---

### Tenant Isolation

11. **AC11 — Tenant isolation (E12-R-001)**: Every SQL query inside `generate_report` that reads analytics data MUST include `WHERE company_id = :company_id` as the **first, non-optional WHERE clause** before any date range or ordering clauses. This applies to queries against `mv_market_intelligence`, `mv_roi_tracker`, `mv_team_performance`, and `pipeline.pipeline_predictions`. The `report_jobs` update also includes `AND id = :job_id` (implying company_id is always scoped via the job row's stored company_id — the task MUST NOT trust the caller-supplied `company_id` blindly; it re-reads the job row first and uses `job_row.company_id`).

---

### Tests

12. **AC12 — Backend integration tests** in `services/notification/tests/test_report_generation.py`:
    - Test: PDF task completes for each of the 4 report types; `client.report_jobs.status` is `'complete'` after task execution (use direct task call, not async Celery execution)
    - Test: DOCX task completes for each of the 4 report types; status `'complete'`
    - Test: S3 object exists at the expected key after task completes (use LocalStack mock or `moto`)
    - Test: Signed URL is stored in `report_jobs.download_url` with `url_expires_at` ≈ NOW + 24h (within ±5 seconds)
    - Test: task updates status to `'processing'` before generating the document (assert intermediate state by patching render step)
    - Test: exception during render → status set to `'failed'`; `error_message` non-empty; no exception propagated to caller
    - Test: company_id isolation — seeding a report_job for Company B and calling the task with Company B's company_id returns data only for Company B (Company A's MV rows not in output)

13. **AC13 — Unit tests for document generation** in `packages/eusolicit-common/tests/test_document_generation.py`:
    - Test: `render_pdf(doc)` returns non-empty bytes for a `ReportDocument` with one of each section type
    - Test: `render_docx(doc)` returns non-empty bytes for the same input
    - Test: `generate_chart_png(chart_type="bar", ...)` returns valid PNG bytes (starts with `b"\x89PNG"`)
    - Test: `generate_chart_png(chart_type="line", ...)` returns valid PNG bytes
    - Test: `render_pdf` with empty `sections` list returns valid (non-empty) PDF (title page only)
    - Test: `render_docx` with empty sections list returns valid DOCX
    - Test: `DocumentRenderError` is raised when `TableSection.rows` contains a row with mismatched column count

14. **AC14 — Unit tests for templates** in `services/notification/tests/test_report_templates.py`:
    - Test: `build_pipeline_summary(...)` with non-empty `forecast_rows` returns `ReportDocument` with `TableSection` and `ChartSection`
    - Test: `build_pipeline_summary(...)` with empty `forecast_rows` returns `ReportDocument` with `TextSection` (empty-state message) and no `TableSection`
    - Test: `build_bid_performance_summary(...)` includes summary text with correct aggregate figures
    - Test: `build_team_activity(...)` includes `TableSection` with 5 columns and correct row count
    - Test: `build_custom_date_range(...)` includes 3 `HeaderSection(level=2)` entries (Market, ROI, Team)
    - Test: All 4 templates pass their output to `render_pdf` without raising (smoke test)
    - Test: All 4 templates pass their output to `render_docx` without raising (smoke test)

15. **AC15 — Import refactor test** in `services/client-api/tests/test_proposal_export_shared_module.py` (or nearest equivalent):
    - Test: `from client_api.utils.document_export import render_pdf` resolves to `eusolicit_common.document_generation.pdf_renderer.render_pdf` (i.e., the client-api re-exports from the shared module rather than defining its own implementation)
    - Test: `from eusolicit_common.document_generation import render_pdf, render_docx, generate_chart_png` all succeed (package importable)

---

## Tasks / Subtasks

### Infrastructure

- [x] Task 1: Alembic migration 013 — create `client.report_jobs` table (AC: 1)
  - [x] 1.1 Create `services/client-api/alembic/versions/013_report_jobs.py` with `down_revision = "012"`
  - [x] 1.2 `upgrade()` — `CREATE TABLE IF NOT EXISTS client.report_jobs` with all columns and CHECK constraints (see AC1)
  - [x] 1.3 `upgrade()` — create 3 indexes: `ix_report_jobs_company_id`, `ix_report_jobs_company_status`, `ix_report_jobs_created_at`
  - [x] 1.4 `upgrade()` — `GRANT SELECT, INSERT, UPDATE ON client.report_jobs TO client_api_role; GRANT SELECT, INSERT, UPDATE ON client.report_jobs TO notification_role;`
  - [x] 1.5 `downgrade()` — drop indexes (reverse order), then `DROP TABLE IF EXISTS client.report_jobs`

- [x] Task 2: SQLAlchemy Core Table definition for `report_jobs` (AC: 2)
  - [x] 2.1 Create `services/client-api/src/client_api/models/report_job.py` with `sa.Table("report_jobs", metadata, ..., schema="client", info={"is_view": False})`; define all columns matching AC1
  - [x] 2.2 Export `report_jobs` table from `services/client-api/src/client_api/models/__init__.py`

### Shared Document Generation Module

- [x] Task 3: Create `eusolicit_common.document_generation` package (AC: 3)
  - [x] 3.1 Create `packages/eusolicit-common/src/eusolicit_common/document_generation/__init__.py` with re-exports
  - [x] 3.2 Create `packages/eusolicit-common/src/eusolicit_common/document_generation/exceptions.py` — define `DocumentRenderError`
  - [x] 3.3 Create `packages/eusolicit-common/src/eusolicit_common/document_generation/models.py` — define `ReportMetadata`, `HeaderSection`, `TextSection`, `TableSection`, `ChartSection`, `ReportSection`, `ReportDocument` (Pydantic v2 models as per AC3)
  - [x] 3.4 Create `packages/eusolicit-common/src/eusolicit_common/document_generation/charts.py` — implement `generate_chart_png(chart_type, labels, values, title, xlabel, ylabel) -> bytes` using `matplotlib` with `Agg` backend; include `"bar"`, `"line"`, and `"pie"` types; close figure after render
  - [x] 3.5 Create `packages/eusolicit-common/src/eusolicit_common/document_generation/pdf_renderer.py` — implement `render_pdf(doc: ReportDocument) -> bytes` using `reportlab`; handle `HeaderSection` (levels 1–3), `TextSection`, `TableSection` (grid style, blue header row), `ChartSection` (embed PNG via `reportlab.platypus.Image`); include page numbers
  - [x] 3.6 Create `packages/eusolicit-common/src/eusolicit_common/document_generation/docx_renderer.py` — implement `render_docx(doc: ReportDocument) -> bytes` using `python_docx`; handle all section types; use `Inches(5.5)` width for chart images
  - [x] 3.7 Add `reportlab`, `python-docx`, and `matplotlib` to `packages/eusolicit-common/pyproject.toml` dependencies (if not already present)

- [x] Task 4: Refactor E07 proposal export to use shared module (AC: 4)
  - [x] 4.1 Locate existing proposal document export code (likely `services/client-api/src/client_api/utils/document_export.py`)
  - [x] 4.2 If S07.10 is implemented: replace inline reportlab/python-docx rendering with calls to `eusolicit_common.document_generation.render_pdf` / `render_docx`; preserve existing branding and formatting by mapping proposal sections to `ReportSection` objects
  - [x] 4.3 If S07.10 is not yet implemented: create stub `document_export.py` that imports and re-exports `render_pdf`, `render_docx`, `generate_chart_png` from `eusolicit_common.document_generation`
  - [x] 4.4 Verify that `services/client-api/pyproject.toml` includes `eusolicit-common` as a dependency (should already be the case from earlier stories)

### Report Templates

- [x] Task 5: Create notification service `report_templates/` package (AC: 5–8)
  - [x] 5.1 Create `services/notification/src/notification/report_templates/__init__.py` (empty)
  - [x] 5.2 Create `services/notification/src/notification/report_templates/pipeline_summary.py` — implement `build_pipeline_summary(company_name, date_from, date_to, forecast_rows, generated_at) -> ReportDocument` (AC5)
  - [x] 5.3 Create `services/notification/src/notification/report_templates/bid_performance_summary.py` — implement `build_bid_performance_summary(company_name, date_from, date_to, roi_rows, generated_at) -> ReportDocument` (AC6)
  - [x] 5.4 Create `services/notification/src/notification/report_templates/team_activity.py` — implement `build_team_activity(company_name, date_from, date_to, team_rows, generated_at) -> ReportDocument` (AC7)
  - [x] 5.5 Create `services/notification/src/notification/report_templates/custom_date_range.py` — implement `build_custom_date_range(company_name, date_from, date_to, market_rows, roi_rows, team_rows, generated_at) -> ReportDocument` (AC8)

### Celery Task

- [x] Task 6: Create `generate_report` Celery task (AC: 9, 10, 11)
  - [x] 6.1 Create `services/notification/src/notification/tasks/report_generation.py`
  - [x] 6.2 Define `generate_report` as a `@celery_app.task(bind=True, max_retries=3, ...)` function with the signature in AC9
  - [x] 6.3 Implement Step 1 (mark processing): sync SQLAlchemy session UPDATE on `client.report_jobs`; add `company_id` re-read guard (fetch job row, assert `job_row.company_id == company_id`, else raise `ValueError`)
  - [x] 6.4 Implement Step 2 (query analytics data): dispatch to the correct query function based on `report_type`:
    - `pipeline_summary` → query `pipeline.pipeline_predictions` WHERE `company_id = :company_id` (+ optional date filter)
    - `bid_performance_summary` → query `client.mv_roi_tracker` WHERE `company_id = :company_id`
    - `team_activity` → query `client.mv_team_performance` WHERE `company_id = :company_id`
    - `custom_date_range` → query `mv_market_intelligence`, `mv_roi_tracker`, `mv_team_performance` each WHERE `company_id = :company_id`
  - [x] 6.5 Implement Step 3 (assemble document): call appropriate template builder with queried data; fetch company name from `client.companies WHERE id = :company_id`
  - [x] 6.6 Implement Step 4 (render): `render_pdf(doc)` or `render_docx(doc)` from `eusolicit_common.document_generation`
  - [x] 6.7 Implement Step 5 (S3 upload): `boto3.client("s3").put_object(Bucket=settings.REPORT_S3_BUCKET, Key=s3_key, Body=doc_bytes, ContentType=content_type)` where `s3_key = f"reports/{company_id}/{report_type}/{job_id}.{format}"`
  - [x] 6.8 Implement Step 6 (signed URL): `generate_presigned_url("get_object", Params={"Bucket": bucket, "Key": s3_key}, ExpiresIn=86400)`
  - [x] 6.9 Implement Step 7 (mark complete): UPDATE `report_jobs` with `status='complete'`, `s3_key`, `download_url`, `url_expires_at = NOW() + timedelta(days=1)`, `completed_at = NOW()`
  - [x] 6.10 Add exception handler: catch all exceptions; UPDATE `report_jobs` with `status='failed'`, `error_message = str(exc)`, `completed_at = NOW()`; call `self.retry(exc=exc)` if retries remain; after max retries, log and suppress (do not propagate)
  - [x] 6.11 Add `REPORT_S3_BUCKET` to `services/notification/src/notification/config.py` (`NotificationSettings`): env var `NOTIFICATION_REPORT_S3_BUCKET`, default `"eusolicit-reports-dev"` for local dev
  - [x] 6.12 Register `report_generation` task module in the Celery app's `include` list (in `services/notification/src/notification/celery_app.py` or equivalent)

### Tests

- [x] Task 7: Unit tests for `eusolicit_common.document_generation` (AC: 13)
  - [x] 7.1 Create `packages/eusolicit-common/tests/test_document_generation.py`
  - [x] 7.2 Test: `render_pdf(doc)` returns `bytes` starting with `%PDF` for full `ReportDocument` with all section types
  - [x] 7.3 Test: `render_docx(doc)` returns non-empty `bytes` (valid zip/DOCX magic bytes `PK\x03\x04`)
  - [x] 7.4 Test: `generate_chart_png("bar", ...)` returns bytes starting with PNG magic `b"\x89PNG"`
  - [x] 7.5 Test: `generate_chart_png("line", ...)` returns valid PNG bytes
  - [x] 7.6 Test: `render_pdf` with `sections=[]` returns valid PDF bytes (title page only; no exception)
  - [x] 7.7 Test: `render_docx` with `sections=[]` returns valid DOCX bytes
  - [x] 7.8 Test: `DocumentRenderError` raised when `TableSection.rows` contains a row with fewer cells than `columns` (validate in renderer, not in Pydantic model)

- [x] Task 8: Unit tests for report templates (AC: 14)
  - [x] 8.1 Create `services/notification/tests/test_report_templates.py`
  - [x] 8.2 Test: `build_pipeline_summary` with 3 forecast rows → `ReportDocument` has `TableSection` with 5 columns and `ChartSection`
  - [x] 8.3 Test: `build_pipeline_summary` with `forecast_rows=[]` → `ReportDocument` has `TextSection` (no `TableSection`)
  - [x] 8.4 Test: `build_bid_performance_summary` with seeded roi_rows → summary `TextSection` contains aggregate ROI % string
  - [x] 8.5 Test: `build_team_activity` with 4 team_rows → `TableSection` has 4 data rows and 5 columns
  - [x] 8.6 Test: `build_custom_date_range` → `ReportDocument.sections` contains exactly 3 `HeaderSection(level=2)` entries
  - [x] 8.7 Test (smoke): `render_pdf(build_pipeline_summary(...))` does not raise
  - [x] 8.8 Test (smoke): `render_docx(build_bid_performance_summary(...))` does not raise
  - [x] 8.9 Test (smoke): `render_pdf(build_team_activity(...))` does not raise
  - [x] 8.10 Test (smoke): `render_docx(build_custom_date_range(...))` does not raise

- [x] Task 9: Integration tests for `generate_report` task (AC: 12)
  - [x] 9.1 Create `services/notification/tests/test_report_generation.py`
  - [x] 9.2 Test: task called directly with `report_type="pipeline_summary"`, `format="pdf"` → `report_jobs.status` is `'complete'` in DB; S3 key exists (use `moto` S3 mock)
  - [x] 9.3 Test: task with `format="docx"` → `report_jobs.status` is `'complete'`; S3 key ends with `.docx`
  - [x] 9.4 Test (one per remaining template): `bid_performance_summary` PDF, `team_activity` DOCX, `custom_date_range` PDF — all reach `'complete'`
  - [x] 9.5 Test: S3 object exists at `reports/{company_id}/{report_type}/{job_id}.pdf` after task completion
  - [x] 9.6 Test: `download_url` in `report_jobs` is non-empty string; `url_expires_at` is within 24h ± 5s of task start
  - [x] 9.7 Test: patching `render_pdf` to raise `DocumentRenderError` → task sets status `'failed'`, `error_message` non-empty; no exception propagated from `generate_report()`
  - [x] 9.8 Test: company_id isolation — seed two companies' MV rows; run task for Company A; assert Company B's rows are NOT in the rendered document (mock template builder to capture received data)

- [x] Task 10: Import refactor test (AC: 15)
  - [x] 10.1 Create `services/client-api/tests/test_proposal_export_shared_module.py`
  - [x] 10.2 Test: `import client_api.utils.document_export` succeeds without error
  - [x] 10.3 Test: `from eusolicit_common.document_generation import render_pdf, render_docx, generate_chart_png` all succeed

---

## Dev Notes

### Migration 013 Placement

Migration `013_report_jobs.py` follows `012_pipeline_predictions.py` (from S12.07). Confirm with `alembic history` that `012` is the current head before creating `013`. If a different story has introduced a migration between 012 and 013 in the meantime, adjust `down_revision` accordingly.

### Shared Module: Package Declaration

`packages/eusolicit-common/pyproject.toml` already declares `reportlab` and `python-docx` if S07.10 (proposal export) was already implemented. If not, add them:

```toml
[project.dependencies]
# ... existing deps ...
reportlab = ">=4.0"
python-docx = ">=1.1"
matplotlib = ">=3.8"
```

`matplotlib` with `Agg` backend requires no GUI libraries. In Docker, ensure `libfreetype6` is installed (usually already present). Add to Dockerfile if missing:
```dockerfile
RUN apt-get install -y libfreetype6-dev
```

### Celery App in Notification Service

The notification service's Celery app is configured in `services/notification/src/notification/celery_app.py`. The `include` list already contains `notification.tasks.refresh_analytics_views` (from S12.01). Add the new task module:

```python
app = Celery(
    "notification",
    include=[
        "notification.tasks.refresh_analytics_views",
        "notification.tasks.report_generation",     # ← add this
    ],
)
```

### Synchronous DB Access in Celery Tasks

The notification service's Celery tasks use synchronous SQLAlchemy sessions (not async), consistent with `refresh_analytics_views.py`. Use the pattern:

```python
from notification.db import get_sync_session   # or equivalent sync session factory

with get_sync_session() as session:
    job_row = session.execute(
        sa.select(report_jobs).where(
            report_jobs.c.id == job_id_uuid,
        )
    ).one_or_none()
    if job_row is None:
        logger.warning("report_job not found", job_id=job_id)
        return
    # Use job_row.company_id, never trust caller's company_id
    ...
```

### S3 Client Configuration in Notification Service

```python
import boto3
from notification.config import settings

def get_s3_client():
    kwargs = {}
    if settings.AWS_ENDPOINT_URL:   # set to LocalStack URL in dev
        kwargs["endpoint_url"] = settings.AWS_ENDPOINT_URL
    return boto3.client(
        "s3",
        region_name=settings.AWS_REGION,
        aws_access_key_id=settings.AWS_ACCESS_KEY_ID,
        aws_secret_access_key=settings.AWS_SECRET_ACCESS_KEY,
        **kwargs,
    )
```

`AWS_ENDPOINT_URL` is set to `http://localstack:4566` in `docker-compose.yml` for local development.

### Analytics Data Queries per Report Type

All queries MUST scope to `company_id` first. Example for bid performance summary:

```python
query = (
    sa.select(mv_roi_tracker)
    .where(mv_roi_tracker.c.company_id == company_id_uuid)  # MUST be first
)
if date_from:
    query = query.where(mv_roi_tracker.c.bid_date >= date_from)
if date_to:
    query = query.where(mv_roi_tracker.c.bid_date <= date_to)
```

For `pipeline_summary`, query `pipeline.pipeline_predictions` (created in migration 012, S12.07). The notification service must have `SELECT` on `pipeline.pipeline_predictions` — check that S12.07's migration granted this to `notification_role`. If not, add the grant in migration 013's `upgrade()`:

```sql
GRANT USAGE ON SCHEMA pipeline TO notification_role;
GRANT SELECT ON pipeline.pipeline_predictions TO notification_role;
```

For `custom_date_range`, the task runs 3 separate queries (market, ROI, team). Each must include `WHERE company_id = :company_id`.

### Chart Image Sizing

For PDF (reportlab), `Image(BytesIO(png_bytes), width=400, height=250)` fits within an A4 page with 1-inch margins. For DOCX, `document.add_picture(BytesIO(png_bytes), width=Inches(5.5))` fills most of the A4 text area width.

When calling `generate_chart_png`, use `dpi=150` for a 1200×750 pixel PNG — this renders cleanly at the document sizes above without excess memory usage.

### Empty-State Handling in Templates

When input data lists are empty, templates must NOT produce empty `TableSection` (which would render as a table with only headers). Instead emit a `TextSection` with a clear message. This is tested in AC14 (Task 8.3).

### Report Job company_id Re-read Guard (AC11 detail)

The task accepts `company_id` as a parameter (for Celery serialisation convenience) but MUST re-read it from the DB job row as the authoritative source:

```python
if str(job_row.company_id) != company_id:
    raise ValueError(
        f"company_id mismatch: job belongs to {job_row.company_id}, "
        f"task was called with {company_id}"
    )
```

This prevents a compromised caller from substituting a different `company_id`.

### `moto` S3 Mock for Integration Tests

Use `moto` library to mock S3 in integration tests:

```python
import boto3
from moto import mock_aws

@mock_aws
def test_report_task_uploads_to_s3():
    conn = boto3.client("s3", region_name="eu-west-1")
    conn.create_bucket(Bucket="eusolicit-reports-dev", ...)
    # call generate_report.apply(args=[...])
    # then assert conn.get_object(Bucket=..., Key=...) does not raise
```

### P0 Test Coverage (from test-design-epic-12.md)

The P0 test from the epic-level test design for S12.09 is:

> **"Async report Celery task completes; S3 object exists; signed URL returned"** (Integration, R12.7, 3 tests)  
> *Use `recurse` to poll job status; assert URL resolves and returns valid file*

This maps to the integration tests in Task 9 (AC12). The `recurse` polling pattern will be used in the Playwright E2E layer (S12.10), not in the unit/integration tests here. For the integration tests in this story, `generate_report` is called directly (not via Celery broker) so polling is not needed — the task runs synchronously in test context.

The risk being mitigated (R12.7) is **silent async report failure** — the Celery task fails without surfacing to the user. The integration test in Task 9.7 (AC12, error path) directly covers this risk by asserting that `status='failed'` is set and `error_message` is populated.

### Test Design Coverage for S12.09

From `test-design-epic-12.md`, the following scenarios cover S12.09:

**P0 (3 tests):**
- Async report Celery task completes; S3 object exists; signed URL returned (Integration, R12.7) — covered by Tasks 9.2–9.6

**P1 (11 tests total):**
- PDF report generated via reportlab: has headers, summary section, and data table (Integration, R12.7, 2 tests) — covered by Tasks 7.2, 8.7, 9.2
- DOCX report generated via python-docx: has headers, summary, and embedded chart image (Integration, R12.7, 2 tests) — covered by Tasks 7.3, 8.8, 9.3
- S3 upload stores report object; signed URL generated with 24h expiry (Integration, 2 tests) — covered by Tasks 9.5–9.6
- All 4 templates produce valid output: pipeline summary, bid performance, team activity, custom date range (Integration, 4 tests) — covered by Tasks 9.2–9.4 and 8.7–8.10
- Shared generation code between E07 proposal export and E12 report engine (no duplication) (Unit, 1 test) — covered by Task 10

**P2 (3 tests):**
- All 4 report templates produce non-empty output (Integration, 2 tests) — covered by Tasks 9.2–9.4
- Signed S3 download URL expires after 24 hours (Integration, 1 test) — covered by Task 9.6 (`url_expires_at` assertion)

The unit tests in AC13 and AC14 cover the P1 document structure and template correctness scenarios. The P0 integration test (S3 object exists, signed URL returned) is covered by the integration test suite in AC12.

---

## Scope Boundaries

This story delivers the **generation engine only** — no API endpoints, no scheduled Celery Beat tasks, no frontend. The following are explicitly **out of scope** and belong to S12.10:

- `POST /analytics/reports` endpoint (on-demand report request)
- `GET /analytics/reports/{job_id}` endpoint (job status polling)
- Celery Beat schedule configuration for weekly/monthly reports
- SendGrid email delivery of completed reports
- Frontend "Generate Report" button and reports list page

---

## File List

### New Files

- `services/client-api/alembic/versions/013_report_jobs.py` — Alembic migration 013; creates `client.report_jobs` table, 3 indexes, GRANTs to `client_api_role` and `notification_role`; downgrade uses `DROP TABLE ... CASCADE`
- `services/client-api/src/client_api/models/report_job.py` — SQLAlchemy Core Table definition for `report_jobs` with `schema="client"` and `info={"is_view": False}`
- `services/client-api/src/client_api/utils/__init__.py` — empty package init
- `services/client-api/src/client_api/utils/document_export.py` — stub re-exporting all symbols from `eusolicit_common.document_generation`
- `services/client-api/tests/test_proposal_export_shared_module.py` — AC15 tests (9 tests) verifying import identity
- `packages/eusolicit-common/src/eusolicit_common/document_generation/__init__.py` — package re-exports
- `packages/eusolicit-common/src/eusolicit_common/document_generation/exceptions.py` — `DocumentRenderError`
- `packages/eusolicit-common/src/eusolicit_common/document_generation/models.py` — Pydantic v2 models (`ReportMetadata`, `HeaderSection`, `TextSection`, `TableSection`, `ChartSection`, `ReportSection`, `ReportDocument`)
- `packages/eusolicit-common/src/eusolicit_common/document_generation/charts.py` — `generate_chart_png` (bar/line/pie, matplotlib Agg)
- `packages/eusolicit-common/src/eusolicit_common/document_generation/pdf_renderer.py` — `render_pdf` using reportlab Platypus
- `packages/eusolicit-common/src/eusolicit_common/document_generation/docx_renderer.py` — `render_docx` using python-docx
- `packages/eusolicit-common/tests/__init__.py` — empty test package init
- `packages/eusolicit-common/tests/test_document_generation.py` — AC13 tests (18 tests)
- `services/notification/src/notification/config.py` — `NotificationSettings` with `REPORT_S3_BUCKET`, AWS settings via pydantic-settings
- `services/notification/src/notification/tasks/__init__.py` — empty package init
- `services/notification/src/notification/tasks/report_generation.py` — `generate_report` Celery task (7-step flow, retry, tenant isolation)
- `services/notification/src/notification/report_templates/__init__.py` — empty package init
- `services/notification/src/notification/report_templates/pipeline_summary.py` — `build_pipeline_summary`
- `services/notification/src/notification/report_templates/bid_performance_summary.py` — `build_bid_performance_summary`
- `services/notification/src/notification/report_templates/team_activity.py` — `build_team_activity`
- `services/notification/src/notification/report_templates/custom_date_range.py` — `build_custom_date_range`
- `services/notification/tests/test_report_templates.py` — AC14 tests (24 tests)
- `services/notification/tests/test_report_generation.py` — AC12 integration tests (12 tests, SQLite + moto S3)

### Modified Files

- `services/client-api/src/client_api/models/__init__.py` — added `report_jobs` to exports
- `services/client-api/alembic/env.py` — added `"report_jobs"` to `_EXCLUDED_TABLE_NAMES` (table created via raw SQL, not ORM metadata)
- `packages/eusolicit-common/pyproject.toml` — added `reportlab>=4.0`, `python-docx>=1.1`, `matplotlib>=3.8`
- `services/notification/pyproject.toml` — added `boto3>=1.34` to deps; `moto[s3]>=5.0` to dev deps
- `services/notification/src/notification/workers/celery_app.py` — added `"notification.tasks.report_generation"` to `include` list

---

## Dev Agent Record

### Implementation Notes

- **Migration 013 downgrade**: Uses `DROP TABLE IF EXISTS client.report_jobs CASCADE` (not explicit index drops) to avoid `InsufficientPrivilege` errors when the migration user doesn't own individual indexes (consistent with how other migrations work in test environments where tables may have been created by a different DB user).
- **alembic/env.py exclusion**: `report_jobs` added to `_EXCLUDED_TABLE_NAMES` because it is created via raw SQL in migration 013, not through ORM `Base.metadata`. Without this, `alembic check` incorrectly proposes a DROP TABLE migration.
- **Celery task retry behaviour**: In tests, `self.retry()` raises a `Retry` exception when called outside a real Celery worker context. Tests patch `generate_report.retry` with `side_effect=MaxRetriesExceededError()` to simulate exhausted retries, triggering the suppress-and-log path.
- **SQLite UUID columns**: Integration tests use `sa.UUID(as_uuid=True)` for test table columns to avoid Python `uuid.UUID` vs string comparison issues in SQLite (SQLite doesn't auto-cast UUID objects).
- **matplotlib thread safety**: `matplotlib.use("Agg")` called at import time in `charts.py` to avoid GUI backend issues in Celery worker threads.

### Test Results

All 63 new tests pass across 4 test files:
- `packages/eusolicit-common/tests/test_document_generation.py` — 18 passed
- `services/notification/tests/test_report_templates.py` — 24 passed
- `services/notification/tests/test_report_generation.py` — 12 passed
- `services/client-api/tests/test_proposal_export_shared_module.py` — 9 passed

Regressions introduced: 1 (`alembic check` test) — fixed by adding `report_jobs` to `_EXCLUDED_TABLE_NAMES`.

Pre-existing failures confirmed unchanged (all unrelated to this story):
- `test_009_migration.py` (4 FAILED + 11 ERROR) — RED PHASE tests awaiting story 11 ESPD migration
- `test_autofill_gateway_500_returns_503_not_500` — pre-existing bug in ESPD autofill story
- `test_downgrade_base_rolls_back_completely` — pre-existing local DB state issue (tables owned by `eusolicit` superuser vs `migration_role`; same issue exists for migration 012)

---

## Senior Developer Review

### Review v1.0 — 2026-04-13

**Verdict:** CHANGES REQUESTED

**Summary:** Implementation architecturally sound but two missing database GRANTs (`client.pipeline_predictions` and `client.companies` for `notification_role`) would cause `InsufficientPrivilege` in production. Additionally, duplicate `_to_float()`/`_to_int()` helpers across 3 template files.

**Blocking findings:** F1 (missing GRANT on `client.pipeline_predictions`), F2 (missing GRANT on `client.companies`). **Deferrable:** F3 (DRY violation in helpers), F4 (`format` shadows builtin), F5 (PDF page numbers only on later pages).

All blocking findings were resolved in v1.1.

---

### Review v1.1 — 2026-04-13

**Verdict:** APPROVED ✅

#### Summary

Re-review of v1.1 implementation after F1/F2/F3 fixes. All 28 files verified (23 new, 5 modified). The previous blocking findings (F1, F2) are confirmed fixed — migration 013 now includes `GRANT SELECT ON client.pipeline_predictions TO notification_role` and `GRANT SELECT ON client.companies TO notification_role` with corresponding `REVOKE` in `downgrade()`. F3 (DRY) was also resolved: `_utils.py` created with `to_float()`/`to_int()`, all 3 templates updated to import from shared util.

No new blocking issues found. Implementation is production-ready.

#### Verification of v1.0 Fixes

- **F1 (GRANT pipeline_predictions) ✅** — migration 013 line 76: `GRANT SELECT ON client.pipeline_predictions TO notification_role`; downgrade line 86: `REVOKE`. Confirmed migration 012 creates table in `client` schema.
- **F2 (GRANT companies) ✅** — migration 013 line 80: `GRANT SELECT ON client.companies TO notification_role`; downgrade line 85: `REVOKE`.
- **F3 (DRY helpers) ✅** — `report_templates/_utils.py` created with `to_float()` and `to_int()`. All 3 templates (`bid_performance_summary.py`, `team_activity.py`, `custom_date_range.py`) import from `notification.report_templates._utils`.

#### Adversarial Review Findings (v1.1)

##### F6 — DOCX `document.save()` outside try/except (DEFERRABLE)

**Location:** `packages/eusolicit-common/src/eusolicit_common/document_generation/docx_renderer.py:128-131`
**Severity:** 🟡 Low
**Category:** Code Quality / Error Handling

`document.save(buf)` (the heavyweight DOCX serialization step) is placed after the `except` block. If serialization fails, the raw exception propagates instead of being wrapped in `DocumentRenderError`. Compare to `pdf_renderer.py` where `pdf.build()` is inside the try/except. In the current Celery task context, the outer exception handler in `generate_report` catches it, so no user-visible impact. Fix by moving `document.save(buf)` inside the existing try block.

##### F7 — `_pipeline_predictions` schema spec inconsistency (INFORMATIONAL)

**Location:** Story spec AC5 / Dev Notes reference `pipeline.pipeline_predictions`
**Severity:** ⚪ Informational

Story spec and Dev Notes reference `pipeline.pipeline_predictions`, but migration 012 creates the table in the `client` schema (`client.pipeline_predictions`). The implementation correctly targets `client.pipeline_predictions` (`schema="client"` in `report_generation.py:135`). No code change needed — the spec text has an error, not the code.

##### F8 — Global `_engine` lazy init has no thread-safety guard (INFORMATIONAL)

**Location:** `report_generation.py:141-151`
**Severity:** ⚪ Informational

`_get_engine()` uses a module-level global `_engine` without locking. Under Celery's default prefork pool this is safe (separate processes). Under eventlet/gevent, the worst case is two engines created; SQLAlchemy handles this gracefully. No fix required unless worker pool model changes.

#### Code Quality Assessment ✅

- **Migration 013**: Correct DDL, proper CREATE TABLE/INDEX/GRANT structure, reversible downgrade with CASCADE
- **SQLAlchemy Core model** (`report_job.py`): Columns mirror migration exactly, `info={"is_view": False}` correct, exported from `__init__.py`
- **Shared `document_generation` package**: Clean discriminated union model, proper `__all__` exports, thread-safe matplotlib, table validation in both renderers
- **Report templates**: All 4 handle empty state correctly, proper section composition, consistent use of shared `_utils.py`
- **Celery task**: 7-step flow matches AC9, tenant isolation guard correct (re-reads company_id from DB), retry/suppress logic correct, S3 key pattern and content types match AC10
- **Config**: pydantic-settings with env prefix, LRU-cached singleton, LocalStack endpoint override

#### Test Coverage Assessment ✅

- **63 tests across 4 files** — count verified: 18 + 24 + 12 + 9 = 63
- AC12 integration tests: SQLite + moto S3 mock, full task flow coverage
- AC12-T8 (processing status): Clever render-side capture pattern confirms intermediate state
- AC12-T9 (error path): MaxRetriesExceededError simulation exercises suppress-and-log path
- AC12-T10 (tenant isolation): Captures template builder args, asserts Company B rows excluded
- AC13 unit tests: Magic byte validation, empty doc, column mismatch error
- AC14 template tests: Section type assertions, smoke renders, empty state
- AC15 import tests: Object identity (`is`) assertions confirm re-export, not reimplementation
- Additional tests beyond AC requirements: company_id mismatch → failed, job not found → silent exit

#### Architecture Alignment ✅

- Shared `eusolicit_common.document_generation` package — correct per architecture
- E07 stub re-export pattern — appropriate for unimplemented upstream story (S07.10)
- Celery task registration in `workers/celery_app.py` — consistent with `refresh_analytics_views`
- Sync SQLAlchemy session pattern in Celery — consistent with notification service conventions
- S3 key pattern `reports/{company_id}/{report_type}/{job_id}.{format}` — matches AC10
- MV ownership by `notification_role` (migration 011) — confirms SELECT access to analytics views

---

## Change Log

| Date | Version | Description | Author |
|------|---------|-------------|--------|
| 2026-04-13 | 1.0 | Initial implementation: migration 013, shared document_generation package, 4 report templates, generate_report Celery task, 63 tests, alembic env.py exclusion fix | Dev Agent |
| 2026-04-13 | 1.1 | Review fixes: F1 — added `GRANT SELECT ON client.pipeline_predictions TO notification_role` in migration 013 upgrade() + REVOKE in downgrade(); F2 — added `GRANT SELECT ON client.companies TO notification_role` + REVOKE; F3 — extracted duplicate `_to_float()`/`_to_int()` helpers to `report_templates/_utils.py`, updated bid_performance_summary, team_activity, custom_date_range to import from shared util. All 63 tests pass. | Dev Agent |
