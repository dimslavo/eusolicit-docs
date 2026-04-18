# Story 6.6: Document Upload API (S3 + ClamAV)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer building the document management feature**,
I want **`POST /api/v1/opportunities/{opportunity_id}/documents/upload` to return an S3 presigned PUT URL for direct browser upload after validating MIME type, file size (≤100MB), and total package size (≤500MB via `SELECT … FOR UPDATE`), and `POST .../documents/{doc_id}/confirm` to set `scan_status = pending` and dispatch a ClamAV async Celery task, with an internal `POST .../documents/{doc_id}/scan-result` callback endpoint that updates `scan_status` to `clean` or `infected` and soft-deletes the S3 object on infection — all metadata persisted in the new `client.documents` table**,
so that **the document upload component (S06.12) can perform secure browser-to-S3 uploads, the download endpoint (S06.07) can gate access on `scan_status = clean`, and the security surface for malware upload is fully closed before any document becomes downloadable**.

## Acceptance Criteria

1. Alembic migration `019_documents.py` creates `client.documents` with columns: `id UUID PK DEFAULT gen_random_uuid()`, `opportunity_id UUID NOT NULL`, `user_id UUID NOT NULL REFERENCES client.users(id) ON DELETE CASCADE`, `company_id UUID NOT NULL REFERENCES client.companies(id) ON DELETE CASCADE`, `filename VARCHAR(255) NOT NULL`, `size BIGINT NOT NULL`, `mime_type VARCHAR(100) NOT NULL`, `s3_key VARCHAR(500) NOT NULL`, `scan_status VARCHAR(20) NOT NULL DEFAULT 'pending'` (one of `pending`, `clean`, `infected`, `failed`), `scan_completed_at TIMESTAMPTZ`, `uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`, `deleted_at TIMESTAMPTZ`; with composite index on `(opportunity_id, company_id)` WHERE `deleted_at IS NULL` and single-column index on `company_id`.

2. `POST /api/v1/opportunities/{opportunity_id}/documents/upload` accepts a JSON body `{"filename": str, "size": int, "mime_type": str}`, requires a valid Bearer JWT (returns 401 if absent/invalid), and requires a paid tier (`starter`, `professional`, `enterprise`; returns 403 with `{"error": "tier_limit", "upgrade_url": "..."}` for free-tier users).

3. Upload request returns 400 with `{"error": "validation_error", "detail": "..."}` when:
   - `mime_type` is not one of `application/pdf`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (DOCX), `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` (XLSX), `application/zip`, `application/x-zip-compressed`.
   - `size` exceeds 100 MB (104,857,600 bytes).

4. Upload request uses `SELECT SUM(size) … FOR UPDATE` on `client.documents` (filtered by `opportunity_id`, `company_id`, `deleted_at IS NULL`) inside the active database transaction before inserting the new document record; if `current_total + request.size > 500 MB (524,288,000 bytes)`, returns 400 with `{"error": "package_size_limit", "detail": "Total package size would exceed 500 MB."}`. The `FOR UPDATE` lock prevents concurrent inserts from both passing the size check simultaneously.

5. On a valid upload request: generates a UUID `doc_id`; constructs S3 key `documents/{company_id}/{opportunity_id}/{doc_id}/{filename}`; generates a presigned PUT URL via `boto3` with `ExpiresIn = 900` seconds (15 minutes) and `ContentType` condition matching the requested `mime_type`; inserts a `client.documents` row with `scan_status = 'pending'`; returns `{"doc_id": "<uuid>", "presigned_url": "<url>", "expires_in": 900}` with HTTP 200.

6. `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/confirm` requires a valid Bearer JWT; verifies the document exists and belongs to the requesting user (`user_id` matches) and has `scan_status = 'pending'` (returns 404 if not found, 422 if not in `pending` state); updates `uploaded_at = NOW()`; dispatches `dispatch_document_scan(doc_id=str, s3_key=str, s3_bucket=str)` via the existing Celery producer; returns `{"doc_id": "<uuid>", "scan_status": "pending", "message": "Scan initiated."}`.

7. `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/scan-result` is an internal callback endpoint; requires `X-Internal-Service-Key` header matching `settings.internal_service_secret` (returns 401 if absent or invalid); accepts body `{"status": "clean" | "infected"}`; updates `client.documents` row: sets `scan_status`, `scan_completed_at = NOW()`; if `status = infected`, also sets `deleted_at = NOW()` (soft-delete) and calls `_soft_delete_s3_object(s3_key, s3_bucket)` to tag or delete the S3 object; returns `{"doc_id": "<uuid>", "scan_status": "<new_status>"}`.

8. `dispatch_document_scan` is added to `celery_producer.py` following the existing `dispatch_generate_report` pattern; dispatches task `"notification.tasks.document_scan.scan_document"` with args `[doc_id, s3_key, s3_bucket]` via `celery_producer.send_task()`.

9. All three endpoints appear in `/openapi.json` with correct path, path parameter schemas, request body schemas, and response descriptions.

10. Integration tests in `tests/api/test_document_upload.py` cover:
    - MIME type validation: 4 allowed types accepted; `image/jpeg` and `application/x-msdownload` rejected with 400 (E06-P1-016)
    - File size > 100MB rejected with 400 (E06-P2-005)
    - Presigned PUT URL returned with `expires_in = 900` after valid upload request; `client.documents` row inserted with `scan_status = pending` after confirm (E06-P1-017)
    - `scan_status = pending` → download endpoint returns 422 (E06-P0-006 — verified via direct DB state, S06.07 integration covered in that story)
    - ClamAV `infected` callback: `scan_status = infected`, `deleted_at IS NOT NULL`, S3 soft-delete called (E06-P0-007)
    - Concurrent upload package size race: two concurrent 300MB requests against 200MB existing usage; verify only one is accepted (E06-P2-004)
    - Unauthenticated → 401; free-tier → 403

## Tasks / Subtasks

- [x] Task 1: Create `client.documents` Alembic migration (AC: 1)
  - [x] 1.1 Create `eusolicit-app/services/client-api/alembic/versions/019_documents.py`:

    ```python
    """Create client.documents table for S06.06.

    Revision ID: 019_documents
    Revises: 018_ai_summaries
    Create Date: 2026-04-17

    client.documents stores document upload metadata per opportunity per user.
    opportunity_id is stored as a bare UUID (no FK) to avoid cross-schema FK
    constraints — the client_api_role has read-only access to pipeline schema.
    scan_status transitions: pending → clean (ClamAV pass) | infected (ClamAV fail) | failed (timeout).
    deleted_at is set on infected documents (soft-delete); S3 object is tagged/deleted separately.
    """
    from __future__ import annotations

    import sqlalchemy as sa
    from alembic import op

    revision = "019_documents"
    down_revision = "018_ai_summaries"
    branch_labels = None
    depends_on = None


    def upgrade() -> None:
        op.create_table(
            "documents",
            sa.Column(
                "id",
                sa.UUID(as_uuid=True),
                primary_key=True,
                server_default=sa.text("gen_random_uuid()"),
            ),
            sa.Column("opportunity_id", sa.UUID(as_uuid=True), nullable=False),
            sa.Column(
                "user_id",
                sa.UUID(as_uuid=True),
                sa.ForeignKey("client.users.id", ondelete="CASCADE"),
                nullable=False,
            ),
            sa.Column(
                "company_id",
                sa.UUID(as_uuid=True),
                sa.ForeignKey("client.companies.id", ondelete="CASCADE"),
                nullable=False,
            ),
            sa.Column("filename", sa.String(255), nullable=False),
            sa.Column("size", sa.BigInteger, nullable=False),
            sa.Column("mime_type", sa.String(100), nullable=False),
            sa.Column("s3_key", sa.String(500), nullable=False),
            sa.Column(
                "scan_status",
                sa.String(20),
                nullable=False,
                server_default=sa.text("'pending'"),
            ),
            sa.Column("scan_completed_at", sa.DateTime(timezone=True), nullable=True),
            sa.Column(
                "uploaded_at",
                sa.DateTime(timezone=True),
                nullable=False,
                server_default=sa.text("NOW()"),
            ),
            sa.Column(
                "created_at",
                sa.DateTime(timezone=True),
                nullable=False,
                server_default=sa.text("NOW()"),
            ),
            sa.Column("deleted_at", sa.DateTime(timezone=True), nullable=True),
            schema="client",
        )
        # Composite index for package-size aggregation queries (filtered to non-deleted rows)
        op.create_index(
            "ix_documents_opp_company",
            "documents",
            ["opportunity_id", "company_id"],
            schema="client",
            postgresql_where=sa.text("deleted_at IS NULL"),
        )
        # Single-column index for company-scoped document listing
        op.create_index(
            "ix_documents_company_id",
            "documents",
            ["company_id"],
            schema="client",
        )
        # Partial index for scan timeout job (only pending rows)
        op.create_index(
            "ix_documents_pending",
            "documents",
            ["scan_status"],
            schema="client",
            postgresql_where=sa.text("scan_status = 'pending'"),
        )


    def downgrade() -> None:
        op.drop_index("ix_documents_pending", table_name="documents", schema="client")
        op.drop_index("ix_documents_company_id", table_name="documents", schema="client")
        op.drop_index("ix_documents_opp_company", table_name="documents", schema="client")
        op.drop_table("documents", schema="client")
    ```

- [x] Task 2: Create `client_document.py` Core table definition (AC: 1, 4, 5, 6, 7)
  - [x] 2.1 Create `eusolicit-app/services/client-api/src/client_api/models/client_document.py`:

    ```python
    """SQLAlchemy Core table definition for client.documents (S06.06).

    client.documents stores document upload metadata per (opportunity, user, company).
    Used by:
      S06.06 — upload request, confirm, scan-result callback
      S06.07 — download (verifies scan_status = clean)
      S06.12 — frontend upload component polls scan_status

    opportunity_id is stored as bare UUID (no FK) — cross-schema FK to
    pipeline.opportunities is not permitted from client_api_role.

    scan_status values: 'pending' | 'clean' | 'infected' | 'failed'
    deleted_at is set on infected documents (soft-delete).
    """
    from __future__ import annotations

    from sqlalchemy import (
        BigInteger,
        Column,
        DateTime,
        MetaData,
        String,
        Table,
        text,
    )
    from sqlalchemy.dialects.postgresql import UUID

    client_metadata = MetaData(schema="client")

    documents_table = Table(
        "documents",
        client_metadata,
        Column("id", UUID(as_uuid=True), primary_key=True),
        Column("opportunity_id", UUID(as_uuid=True), nullable=False),
        Column("user_id", UUID(as_uuid=True), nullable=False),
        Column("company_id", UUID(as_uuid=True), nullable=False),
        Column("filename", String(255), nullable=False),
        Column("size", BigInteger, nullable=False),
        Column("mime_type", String(100), nullable=False),
        Column("s3_key", String(500), nullable=False),
        Column(
            "scan_status",
            String(20),
            nullable=False,
            server_default=text("'pending'"),
        ),
        Column("scan_completed_at", DateTime(timezone=True), nullable=True),
        Column(
            "uploaded_at",
            DateTime(timezone=True),
            nullable=False,
            server_default=text("NOW()"),
        ),
        Column(
            "created_at",
            DateTime(timezone=True),
            nullable=False,
            server_default=text("NOW()"),
        ),
        Column("deleted_at", DateTime(timezone=True), nullable=True),
    )
    ```

    Note: This file uses its own `client_metadata = MetaData(schema="client")` instance,
    separate from any pipeline-schema MetaData — matching the same isolation pattern used
    in `client_ai_summary.py`. Do NOT share this MetaData with pipeline tables.

- [x] Task 3: Add document upload settings to `config.py` (AC: 3, 4, 5)
  - [x] 3.1 Edit `eusolicit-app/services/client-api/src/client_api/config.py` — append the following block inside `ClientApiSettings` AFTER the existing `cors_allowed_origins` field:

    ```python
        # Document upload settings (Story S06.06)
        # Set via CLIENT_API_DOCUMENTS_S3_BUCKET, etc.
        documents_s3_bucket: str = Field(
            default="eusolicit-documents-dev",
            env="DOCUMENTS_S3_BUCKET",
            description="S3 bucket for user-uploaded opportunity documents.",
        )
        document_presigned_put_expiry: int = Field(
            default=900,
            env="DOCUMENT_PRESIGNED_PUT_EXPIRY",
            description="Presigned PUT URL lifetime in seconds (default 15 minutes).",
        )
        max_document_size_bytes: int = Field(
            default=104_857_600,  # 100 MB
            env="MAX_DOCUMENT_SIZE_BYTES",
            description="Maximum single-file upload size in bytes (default 100 MB).",
        )
        max_package_size_bytes: int = Field(
            default=524_288_000,  # 500 MB
            env="MAX_PACKAGE_SIZE_BYTES",
            description="Maximum total package size per (opportunity, company) in bytes (default 500 MB).",
        )
    ```

- [x] Task 4: Create `schemas/documents.py` request and response schemas (AC: 2, 3, 5, 6, 7)
  - [x] 4.1 Create `eusolicit-app/services/client-api/src/client_api/schemas/documents.py`:

    ```python
    """Pydantic request / response schemas for document upload (S06.06) and download (S06.07).

    DocumentUploadRequest   — body for POST .../documents/upload
    DocumentUploadResponse  — response for POST .../documents/upload
    DocumentConfirmResponse — response for POST .../documents/{doc_id}/confirm
    ScanResultRequest       — body for POST .../documents/{doc_id}/scan-result (internal)
    ScanResultResponse      — response for POST .../documents/{doc_id}/scan-result (internal)

    Allowed MIME types are defined here (not in the route handler) so S06.12 frontend
    docs can reference the same canonical list without duplication.

    Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.06
    """
    from __future__ import annotations

    from uuid import UUID

    from pydantic import BaseModel, Field

    # ---------------------------------------------------------------------------
    # Constants
    # ---------------------------------------------------------------------------

    ALLOWED_MIME_TYPES: frozenset[str] = frozenset(
        {
            "application/pdf",
            # DOCX — Microsoft Word 2007+
            "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
            # XLSX — Microsoft Excel 2007+
            "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
            # ZIP (RFC 1951)
            "application/zip",
            # Alternate ZIP MIME type sent by some browsers/OS
            "application/x-zip-compressed",
        }
    )

    ALLOWED_MIME_TYPES_DISPLAY = "PDF, DOCX, XLSX, ZIP"

    # Scan status values (matches DB column constraint)
    SCAN_STATUS_PENDING = "pending"
    SCAN_STATUS_CLEAN = "clean"
    SCAN_STATUS_INFECTED = "infected"
    SCAN_STATUS_FAILED = "failed"

    VALID_SCAN_STATUSES = frozenset(
        {SCAN_STATUS_PENDING, SCAN_STATUS_CLEAN, SCAN_STATUS_INFECTED, SCAN_STATUS_FAILED}
    )

    # ---------------------------------------------------------------------------
    # Upload request / response
    # ---------------------------------------------------------------------------

    class DocumentUploadRequest(BaseModel):
        """File metadata for presigned URL generation.

        The client sends this BEFORE uploading to S3.
        The API validates MIME type and sizes, then issues a presigned PUT URL.
        The file bytes are never sent to the API — they go directly to S3.
        """

        filename: str = Field(
            ...,
            min_length=1,
            max_length=255,
            description="Original filename (e.g. 'technical_proposal.pdf').",
        )
        size: int = Field(
            ...,
            gt=0,
            description="File size in bytes.",
        )
        mime_type: str = Field(
            ...,
            description="MIME type of the file (must be one of: PDF, DOCX, XLSX, ZIP).",
        )


    class DocumentUploadResponse(BaseModel):
        """Presigned S3 PUT URL returned to the browser for direct upload.

        The browser must HTTP PUT the file bytes to `presigned_url` within
        `expires_in` seconds.  After the PUT completes, call the confirm endpoint.
        """

        doc_id: UUID
        presigned_url: str
        expires_in: int = Field(description="Seconds until presigned URL expires (900 = 15 min).")


    # ---------------------------------------------------------------------------
    # Confirm response
    # ---------------------------------------------------------------------------

    class DocumentConfirmResponse(BaseModel):
        """Response after the browser confirms the S3 upload is complete."""

        doc_id: UUID
        scan_status: str
        message: str


    # ---------------------------------------------------------------------------
    # Internal scan-result callback
    # ---------------------------------------------------------------------------

    class ScanResultRequest(BaseModel):
        """ClamAV scan result delivered by the async Celery worker via internal callback.

        Protected by X-Internal-Service-Key header (see document_service.verify_scan_callback_auth).
        """

        status: str = Field(
            ...,
            description="Scan result: 'clean' or 'infected'.",
        )


    class ScanResultResponse(BaseModel):
        """Response from the internal scan-result callback endpoint."""

        doc_id: UUID
        scan_status: str
    ```

- [x] Task 5: Create `services/document_service.py` (AC: 4, 5, 6, 7, 8)
  - [x] 5.1 Create `eusolicit-app/services/client-api/src/client_api/services/document_service.py`:

    ```python
    """Document upload service — Story S06.06.

    Provides service functions for:
      request_upload       — validate, size-lock, create document record, generate presigned PUT URL
      confirm_upload       — mark uploaded_at, dispatch ClamAV Celery task
      record_scan_result   — update scan_status; soft-delete S3 object on infection

    S3 presigned URL generation reuses the same boto3 pattern established in report_service.py.
    Package size enforcement uses SELECT ... FOR UPDATE to prevent concurrent insert races
    (E06-R-006 mitigation).

    Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.06
    Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-R-006
    """
    from __future__ import annotations

    import secrets
    import uuid
    from datetime import UTC, datetime
    from uuid import UUID

    import structlog
    from sqlalchemy import func, select, text, update
    from sqlalchemy.ext.asyncio import AsyncSession

    from client_api.celery_producer import dispatch_document_scan
    from client_api.config import get_settings
    from client_api.models.client_document import documents_table as doc_t
    from client_api.schemas.documents import (
        ALLOWED_MIME_TYPES,
        ALLOWED_MIME_TYPES_DISPLAY,
        SCAN_STATUS_CLEAN,
        SCAN_STATUS_INFECTED,
        SCAN_STATUS_PENDING,
        VALID_SCAN_STATUSES,
        DocumentConfirmResponse,
        DocumentUploadResponse,
        ScanResultResponse,
    )
    from eusolicit_common.exceptions import AppException

    log = structlog.get_logger()

    # ---------------------------------------------------------------------------
    # S3 helpers (reuse same pattern as report_service._get_s3_client)
    # ---------------------------------------------------------------------------


    def _get_s3_client():
        """Return a boto3 S3 client, honouring AWS_ENDPOINT_URL for LocalStack / moto."""
        import boto3  # noqa: PLC0415

        settings = get_settings()
        kwargs: dict = {}
        if settings.aws_endpoint_url:
            kwargs["endpoint_url"] = settings.aws_endpoint_url
        return boto3.client(
            "s3",
            region_name=settings.aws_region,
            aws_access_key_id=settings.aws_access_key_id,
            aws_secret_access_key=settings.aws_secret_access_key,
            **kwargs,
        )


    def _generate_presigned_put_url(s3_key: str, mime_type: str, expiry: int) -> str:
        """Generate an S3 presigned PUT URL with ContentType condition.

        Parameters
        ----------
        s3_key:
            Full S3 object key (e.g. 'documents/{company_id}/.../{filename}').
        mime_type:
            MIME type the client must use as Content-Type when issuing the PUT.
        expiry:
            Presigned URL lifetime in seconds.

        Returns
        -------
        str
            Pre-signed URL for HTTP PUT upload.
        """
        settings = get_settings()
        s3 = _get_s3_client()
        return s3.generate_presigned_url(
            "put_object",
            Params={
                "Bucket": settings.documents_s3_bucket,
                "Key": s3_key,
                "ContentType": mime_type,
            },
            ExpiresIn=expiry,
        )


    def _soft_delete_s3_object(s3_key: str) -> None:
        """Tag the S3 object as infected (soft-delete via object tag).

        Uses the `infected=true` tag to signal lifecycle rules without immediately
        destroying audit evidence.  Production S3 lifecycle rules can be configured
        to expire objects with this tag after a retention period.

        Falls back to a DELETE if the tagging operation fails, to ensure no infected
        object becomes accessible.
        """
        settings = get_settings()
        s3 = _get_s3_client()
        try:
            s3.put_object_tagging(
                Bucket=settings.documents_s3_bucket,
                Key=s3_key,
                Tagging={"TagSet": [{"Key": "infected", "Value": "true"}]},
            )
            log.info("s3_object_tagged_infected", s3_key=s3_key)
        except Exception as exc:  # noqa: BLE001
            log.warning("s3_tag_failed_deleting", s3_key=s3_key, error=str(exc))
            try:
                s3.delete_object(Bucket=settings.documents_s3_bucket, Key=s3_key)
                log.info("s3_object_deleted_infected", s3_key=s3_key)
            except Exception as del_exc:  # noqa: BLE001
                log.error("s3_delete_failed", s3_key=s3_key, error=str(del_exc))


    # ---------------------------------------------------------------------------
    # request_upload
    # ---------------------------------------------------------------------------


    async def request_upload(
        *,
        db: AsyncSession,
        opportunity_id: UUID,
        user_id: UUID,
        company_id: UUID,
        filename: str,
        size: int,
        mime_type: str,
    ) -> DocumentUploadResponse:
        """Validate upload request, enforce size limits, create document record, return presigned URL.

        Steps:
          1. Validate MIME type (400 if not in ALLOWED_MIME_TYPES).
          2. Validate per-file size ≤ MAX_DOCUMENT_SIZE_BYTES (400 if exceeded).
          3. SELECT SUM(size) … FOR UPDATE on client.documents for this (opportunity, company)
             to lock concurrent inserts.
          4. Check current_total + size ≤ MAX_PACKAGE_SIZE_BYTES (400 if exceeded).
          5. Generate doc_id and S3 key.
          6. Insert document record with scan_status='pending'.
          7. Generate presigned PUT URL.
          8. Return DocumentUploadResponse.

        Parameters
        ----------
        db:
            Active async session (get_db_session).  The INSERT and FOR UPDATE lock both
            run inside this session's transaction — committed by get_db_session on success.
        opportunity_id, user_id, company_id:
            From the JWT and path parameter.
        filename, size, mime_type:
            From the validated request body.

        Raises
        ------
        AppException (400)
            MIME type not allowed, per-file size exceeded, or package size exceeded.
        """
        settings = get_settings()

        # 1. Validate MIME type
        if mime_type not in ALLOWED_MIME_TYPES:
            raise AppException(
                f"File type not allowed. Accepted types: {ALLOWED_MIME_TYPES_DISPLAY}.",
                error="validation_error",
                status_code=400,
                details={"field": "mime_type", "allowed": sorted(ALLOWED_MIME_TYPES)},
            )

        # 2. Validate per-file size
        if size > settings.max_document_size_bytes:
            limit_mb = settings.max_document_size_bytes // (1024 * 1024)
            raise AppException(
                f"File size exceeds the {limit_mb} MB per-file limit.",
                error="validation_error",
                status_code=400,
                details={"field": "size", "max_bytes": settings.max_document_size_bytes},
            )

        # 3. SELECT SUM(size) FOR UPDATE — prevents concurrent insert race (E06-R-006)
        size_stmt = (
            select(func.coalesce(func.sum(doc_t.c.size), 0))
            .where(doc_t.c.opportunity_id == opportunity_id)
            .where(doc_t.c.company_id == company_id)
            .where(doc_t.c.deleted_at.is_(None))
            .with_for_update()
        )
        current_total: int = (await db.execute(size_stmt)).scalar_one()

        # 4. Check total package size
        if current_total + size > settings.max_package_size_bytes:
            limit_mb = settings.max_package_size_bytes // (1024 * 1024)
            remaining = max(0, settings.max_package_size_bytes - current_total)
            raise AppException(
                f"Total package size would exceed the {limit_mb} MB limit.",
                error="package_size_limit",
                status_code=400,
                details={"remaining_bytes": remaining, "requested_bytes": size},
            )

        # 5. Generate identifiers
        doc_id = uuid.uuid4()
        s3_key = f"documents/{company_id}/{opportunity_id}/{doc_id}/{filename}"

        # 6. Insert document record
        await db.execute(
            doc_t.insert().values(
                id=doc_id,
                opportunity_id=opportunity_id,
                user_id=user_id,
                company_id=company_id,
                filename=filename,
                size=size,
                mime_type=mime_type,
                s3_key=s3_key,
                scan_status=SCAN_STATUS_PENDING,
                created_at=datetime.now(UTC),
                uploaded_at=datetime.now(UTC),
            )
        )

        # 7. Generate presigned PUT URL (after INSERT so the record is locked)
        try:
            presigned_url = _generate_presigned_put_url(
                s3_key=s3_key,
                mime_type=mime_type,
                expiry=settings.document_presigned_put_expiry,
            )
        except Exception as exc:  # noqa: BLE001
            log.error("presigned_url_generation_failed", doc_id=str(doc_id), error=str(exc))
            raise AppException(
                "Failed to generate upload URL. Please try again.",
                error="s3_error",
                status_code=503,
            ) from exc

        log.info(
            "document_upload_requested",
            doc_id=str(doc_id),
            opportunity_id=str(opportunity_id),
            company_id=str(company_id),
            mime_type=mime_type,
            size=size,
        )

        return DocumentUploadResponse(
            doc_id=doc_id,
            presigned_url=presigned_url,
            expires_in=settings.document_presigned_put_expiry,
        )


    # ---------------------------------------------------------------------------
    # confirm_upload
    # ---------------------------------------------------------------------------


    async def confirm_upload(
        *,
        db: AsyncSession,
        opportunity_id: UUID,
        doc_id: UUID,
        user_id: UUID,
    ) -> DocumentConfirmResponse:
        """Mark document as uploaded and dispatch ClamAV scan task.

        Verifies ownership (user_id must match) and that scan_status is still 'pending'.
        Updates uploaded_at, then dispatches the Celery scan task.

        Raises
        ------
        AppException (404)
            Document not found, or doesn't belong to this user, or is soft-deleted.
        AppException (422)
            Document is not in 'pending' state (already confirmed, infected, etc.).
        """
        settings = get_settings()

        # Fetch document — must belong to this user and this opportunity
        fetch_stmt = (
            select(
                doc_t.c.id,
                doc_t.c.scan_status,
                doc_t.c.s3_key,
                doc_t.c.user_id,
            )
            .where(doc_t.c.id == doc_id)
            .where(doc_t.c.opportunity_id == opportunity_id)
            .where(doc_t.c.deleted_at.is_(None))
        )
        row = (await db.execute(fetch_stmt)).mappings().first()

        if row is None:
            raise AppException(
                "Document not found.",
                error="not_found",
                status_code=404,
            )

        if row["user_id"] != user_id:
            # Return 404 to avoid leaking existence of another user's document
            raise AppException(
                "Document not found.",
                error="not_found",
                status_code=404,
            )

        if row["scan_status"] != SCAN_STATUS_PENDING:
            raise AppException(
                f"Document is not in a confirmable state (current status: {row['scan_status']}).",
                error="invalid_state",
                status_code=422,
            )

        # Update uploaded_at timestamp
        await db.execute(
            update(doc_t)
            .where(doc_t.c.id == doc_id)
            .values(uploaded_at=datetime.now(UTC))
        )

        # Dispatch ClamAV scan task
        s3_key = row["s3_key"]
        try:
            dispatch_document_scan(
                doc_id=str(doc_id),
                s3_key=s3_key,
                s3_bucket=settings.documents_s3_bucket,
            )
        except Exception as exc:  # noqa: BLE001
            log.error(
                "celery_dispatch_failed",
                doc_id=str(doc_id),
                error=str(exc),
            )
            # Do not fail the confirm endpoint if Celery is unreachable in dev/test.
            # scan_status stays 'pending'; the ClamAV timeout job will mark it 'failed'.

        log.info("document_scan_dispatched", doc_id=str(doc_id), s3_key=s3_key)

        return DocumentConfirmResponse(
            doc_id=doc_id,
            scan_status=SCAN_STATUS_PENDING,
            message="Scan initiated.",
        )


    # ---------------------------------------------------------------------------
    # record_scan_result
    # ---------------------------------------------------------------------------


    async def record_scan_result(
        *,
        db: AsyncSession,
        doc_id: UUID,
        status: str,
    ) -> ScanResultResponse:
        """Update scan_status from ClamAV callback result.

        Called by the internal scan-result endpoint after ClamAV reports a verdict.
        If status is 'infected': sets deleted_at (soft-delete) and calls S3 object tagging.
        If status is 'clean': clears any infection markers (should not exist, but be safe).

        Raises
        ------
        AppException (400)
            Invalid status value (not 'clean' or 'infected').
        AppException (404)
            Document not found (may have been deleted or never created).
        AppException (422)
            Document is already in a terminal state (clean/infected/failed).
        """
        if status not in (SCAN_STATUS_CLEAN, SCAN_STATUS_INFECTED):
            raise AppException(
                f"Invalid scan status '{status}'. Must be 'clean' or 'infected'.",
                error="validation_error",
                status_code=400,
            )

        # Fetch current state
        fetch_stmt = (
            select(doc_t.c.id, doc_t.c.scan_status, doc_t.c.s3_key)
            .where(doc_t.c.id == doc_id)
        )
        row = (await db.execute(fetch_stmt)).mappings().first()

        if row is None:
            raise AppException(
                "Document not found.",
                error="not_found",
                status_code=404,
            )

        # Idempotent: if already in terminal state, return current state
        if row["scan_status"] in (SCAN_STATUS_CLEAN, SCAN_STATUS_INFECTED):
            log.warning(
                "scan_result_already_terminal",
                doc_id=str(doc_id),
                current=row["scan_status"],
                received=status,
            )
            return ScanResultResponse(doc_id=doc_id, scan_status=row["scan_status"])

        now = datetime.now(UTC)
        update_values: dict = {
            "scan_status": status,
            "scan_completed_at": now,
        }

        if status == SCAN_STATUS_INFECTED:
            update_values["deleted_at"] = now

        await db.execute(
            update(doc_t).where(doc_t.c.id == doc_id).values(**update_values)
        )

        # Soft-delete S3 object for infected files (non-blocking — log errors but don't fail)
        if status == SCAN_STATUS_INFECTED:
            s3_key = row["s3_key"]
            log.warning("clamav_infected_detected", doc_id=str(doc_id), s3_key=s3_key)
            _soft_delete_s3_object(s3_key)

        log.info("scan_result_recorded", doc_id=str(doc_id), status=status)

        return ScanResultResponse(doc_id=doc_id, scan_status=status)


    # ---------------------------------------------------------------------------
    # Internal auth helper
    # ---------------------------------------------------------------------------


    def verify_scan_callback_auth(x_internal_service_key: str | None) -> None:
        """Verify X-Internal-Service-Key header for the ClamAV scan-result callback.

        Raises UnauthorizedError if the key is absent or does not match
        settings.internal_service_secret.

        Called by the scan-result route handler before processing the callback body.
        """
        from eusolicit_common.exceptions import UnauthorizedError  # noqa: PLC0415

        expected = get_settings().internal_service_secret
        if not expected:
            raise UnauthorizedError("Internal service authentication is not configured.")
        if not x_internal_service_key or not secrets.compare_digest(
            x_internal_service_key, expected
        ):
            raise UnauthorizedError("Invalid or missing X-Internal-Service-Key.")
    ```

- [x] Task 6: Add `dispatch_document_scan` to `celery_producer.py` (AC: 8)
  - [x] 6.1 Edit `eusolicit-app/services/client-api/src/client_api/celery_producer.py` — append after `dispatch_generate_report`:

    ```python

    def dispatch_document_scan(
        doc_id: str,
        s3_key: str,
        s3_bucket: str,
    ) -> None:
        """Dispatch scan_document Celery task to the document scan worker.

        The task downloads the file from S3, scans with ClamAV, and calls the
        internal scan-result callback endpoint with the verdict.

        Parameters
        ----------
        doc_id:
            UUID string of the client.documents row to update.
        s3_key:
            Full S3 object key for the uploaded file.
        s3_bucket:
            S3 bucket name (from settings.documents_s3_bucket).
        """
        celery_producer.send_task(
            "notification.tasks.document_scan.scan_document",
            args=[doc_id, s3_key, s3_bucket],
        )
    ```

- [x] Task 7: Create `api/v1/documents.py` route handlers (AC: 2, 3, 4, 5, 6, 7, 9)
  - [x] 7.1 Create `eusolicit-app/services/client-api/src/client_api/api/v1/documents.py`:

    ```python
    """Document upload API router — Epic 6 (S06.06).

    Exposes:
      POST /api/v1/opportunities/{opportunity_id}/documents/upload      — request presigned URL
      POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/confirm  — confirm S3 upload
      POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/scan-result  — internal ClamAV callback

    Upload and confirm require a valid Bearer JWT (get_current_user) and a paid subscription tier.
    scan-result requires X-Internal-Service-Key header (internal ClamAV worker callback).

    Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.06
    """
    from __future__ import annotations

    from typing import Annotated
    from uuid import UUID

    from fastapi import APIRouter, Depends, Header
    from sqlalchemy.ext.asyncio import AsyncSession

    from client_api.core.security import CurrentUser, get_current_user
    from client_api.dependencies import get_db_session
    from client_api.schemas.documents import (
        DocumentConfirmResponse,
        DocumentUploadRequest,
        DocumentUploadResponse,
        ScanResultRequest,
        ScanResultResponse,
    )
    from client_api.services import document_service
    from eusolicit_common.exceptions import AppException

    router = APIRouter(prefix="/{opportunity_id}/documents", tags=["documents"])

    # Paid tiers that may upload / confirm documents
    _PAID_TIERS = frozenset({"starter", "professional", "enterprise"})


    # ---------------------------------------------------------------------------
    # POST /upload — request presigned PUT URL (S06.06)
    # ---------------------------------------------------------------------------


    @router.post(
        "/upload",
        summary="Request presigned S3 PUT URL for document upload",
        description=(
            "Validates file metadata (MIME type, size) and package size, then issues a "
            "presigned S3 PUT URL for direct browser-to-S3 upload. "
            "Accepted MIME types: application/pdf, DOCX, XLSX, application/zip. "
            "Per-file limit: 100 MB. Total package limit: 500 MB per opportunity+company. "
            "Requires paid subscription tier (Starter, Professional, Enterprise). "
            "Returns doc_id, presigned_url, and expires_in (seconds)."
        ),
        response_model=DocumentUploadResponse,
    )
    async def request_document_upload(
        opportunity_id: UUID,
        body: DocumentUploadRequest,
        current_user: Annotated[CurrentUser, Depends(get_current_user)] = None,  # type: ignore[assignment]
        db: Annotated[AsyncSession, Depends(get_db_session)] = None,  # type: ignore[assignment]
    ) -> DocumentUploadResponse:
        """Issue presigned S3 PUT URL after validation and package-size lock.

        AC2: Requires Bearer JWT; paid tier only (free → 403 tier_limit).
        AC3: MIME type and per-file size validated; 400 on failure.
        AC4: Package size enforced via SELECT SUM FOR UPDATE.
        AC5: Document record inserted with scan_status='pending'; presigned URL returned.
        """
        from client_api.config import get_settings  # noqa: PLC0415

        # Tier check: free users may not upload documents
        if current_user.subscription_tier not in _PAID_TIERS:
            settings = get_settings()
            upgrade_url = settings.frontend_url.rstrip("/") + "/billing/upgrade"
            raise AppException(
                "Document upload requires a paid subscription.",
                error="tier_limit",
                status_code=403,
                details={"upgrade_url": upgrade_url},
            )

        return await document_service.request_upload(
            db=db,
            opportunity_id=opportunity_id,
            user_id=current_user.user_id,
            company_id=current_user.company_id,
            filename=body.filename,
            size=body.size,
            mime_type=body.mime_type,
        )


    # ---------------------------------------------------------------------------
    # POST /{doc_id}/confirm — confirm S3 upload, dispatch ClamAV (S06.06)
    # ---------------------------------------------------------------------------


    @router.post(
        "/{doc_id}/confirm",
        summary="Confirm S3 upload completion and trigger ClamAV scan",
        description=(
            "Called by the browser after the presigned PUT to S3 succeeds. "
            "Updates uploaded_at and dispatches an async ClamAV scan task (Celery). "
            "Returns scan_status='pending'. "
            "Returns 404 if doc_id not found or doesn't belong to the calling user. "
            "Returns 422 if document is not in 'pending' state."
        ),
        response_model=DocumentConfirmResponse,
    )
    async def confirm_document_upload(
        opportunity_id: UUID,
        doc_id: UUID,
        current_user: Annotated[CurrentUser, Depends(get_current_user)] = None,  # type: ignore[assignment]
        db: Annotated[AsyncSession, Depends(get_db_session)] = None,  # type: ignore[assignment]
    ) -> DocumentConfirmResponse:
        """Confirm upload complete, dispatch scan.

        AC6: Validates ownership and pending state; dispatches Celery scan task.
        """
        return await document_service.confirm_upload(
            db=db,
            opportunity_id=opportunity_id,
            doc_id=doc_id,
            user_id=current_user.user_id,
        )


    # ---------------------------------------------------------------------------
    # POST /{doc_id}/scan-result — internal ClamAV callback (S06.06)
    # ---------------------------------------------------------------------------


    @router.post(
        "/{doc_id}/scan-result",
        summary="Internal ClamAV scan-result callback",
        description=(
            "Internal-only endpoint called by the ClamAV Celery worker after scanning. "
            "Requires X-Internal-Service-Key header matching the configured shared secret. "
            "Accepts status='clean' or 'infected'. "
            "On 'infected': sets scan_status=infected, soft-deletes DB record and S3 object. "
            "On 'clean': sets scan_status=clean; document becomes downloadable."
        ),
        response_model=ScanResultResponse,
    )
    async def document_scan_result(
        opportunity_id: UUID,  # noqa: ARG001 — kept for consistent URL namespace
        doc_id: UUID,
        body: ScanResultRequest,
        x_internal_service_key: Annotated[
            str | None,
            Header(alias="X-Internal-Service-Key"),
        ] = None,
        db: Annotated[AsyncSession, Depends(get_db_session)] = None,  # type: ignore[assignment]
    ) -> ScanResultResponse:
        """Record ClamAV scan result; soft-delete on infection.

        AC7: Internal auth via X-Internal-Service-Key; updates scan_status.
        """
        document_service.verify_scan_callback_auth(x_internal_service_key)

        return await document_service.record_scan_result(
            db=db,
            doc_id=doc_id,
            status=body.status,
        )
    ```

- [x] Task 8: Register documents router in `main.py` (AC: 9)
  - [x] 8.1 Edit `eusolicit-app/services/client-api/src/client_api/main.py`:
    - Add import at the top (with other router imports):
      ```python
      from client_api.api.v1 import documents as documents_v1
      ```
    - Add router registration AFTER the opportunities router line (`api_v1_router.include_router(opportunities_v1.router)`):
      ```python
      api_v1_router.include_router(documents_v1.router, prefix="/opportunities")
      ```
    This mounts `documents_v1.router` (which has prefix `/{opportunity_id}/documents`)
    under `/api/v1/opportunities`, producing:
    - `POST /api/v1/opportunities/{opportunity_id}/documents/upload`
    - `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/confirm`
    - `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/scan-result`

- [x] Task 9: Write integration tests (AC: 10)
  - [x] 9.1 Create `eusolicit-app/services/client-api/tests/api/test_document_upload.py`:

    ```python
    """Integration tests for Document Upload API — Story S06.06.

    Test IDs covered:
        E06-P1-016  MIME type validation: allowed types accepted; JPEG and EXE rejected (400)
        E06-P1-017  Valid upload request: presigned URL returned, expires_in=900;
                    confirm sets scan_status=pending in DB
        E06-P0-006  scan_status=pending blocks download (partial — verifies DB state;
                    full download gate tested in test_document_download.py S06.07)
        E06-P0-007  ClamAV infected callback: scan_status=infected, deleted_at set, S3 soft-delete called
        E06-P2-004  Concurrent upload package size race (asyncio.gather)
        E06-P2-005  File size > 100MB rejected with 400 before presigned URL issuance
        (Implicit)  Unauthenticated → 401; free-tier → 403 tier_limit

    Fixtures:
        upload_app — overrides get_db_session + boto3 S3 (moto mock)
        seeded_upload_opportunity — seeds pipeline.opportunities row for the test opportunity_id

    Stack: backend / FastAPI / pytest-asyncio / httpx AsyncClient

    Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#P0-006, P0-007, P1-016,
            P1-017, P2-004, P2-005
    """
    from __future__ import annotations

    import asyncio
    import uuid
    from unittest.mock import MagicMock, patch

    import pytest
    import pytest_asyncio
    from sqlalchemy import select, text

    from client_api.models.client_document import documents_table as doc_t

    # ---------------------------------------------------------------------------
    # Constants
    # ---------------------------------------------------------------------------

    _TEST_OPP_ID = str(uuid.uuid4())
    _STARTER_COMPANY_ID = "12345678-0000-0000-0000-000000000001"
    _INTERNAL_SECRET = "test-internal-secret-s06-06"

    _MAX_FILE_BYTES = 104_857_600   # 100 MB
    _MAX_PKG_BYTES = 524_288_000    # 500 MB

    _ALLOWED_TYPES = [
        "application/pdf",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        "application/zip",
    ]
    _REJECTED_TYPES = [
        "image/jpeg",
        "application/x-msdownload",  # .exe
    ]


    # ---------------------------------------------------------------------------
    # Fixtures
    # ---------------------------------------------------------------------------

    @pytest_asyncio.fixture
    async def upload_app(app, client_api_session_factory):
        """Override get_db_session and mock boto3 S3 for upload tests.

        Uses moto (boto3 mock) for presigned URL generation so no real AWS
        credentials are required in CI.
        """
        import os  # noqa: PLC0415

        from client_api.dependencies import get_db_session  # noqa: PLC0415

        # Set required env vars for moto / S3 mock
        os.environ.setdefault("AWS_DEFAULT_REGION", "eu-central-1")
        os.environ.setdefault("AWS_ACCESS_KEY_ID", "test")
        os.environ.setdefault("AWS_SECRET_ACCESS_KEY", "test")
        os.environ.setdefault("CLIENT_API_DOCUMENTS_S3_BUCKET", "eusolicit-documents-test")
        os.environ.setdefault("CLIENT_API_INTERNAL_SERVICE_SECRET", _INTERNAL_SECRET)

        # Clear settings cache so new env vars are picked up
        from client_api.config import get_settings  # noqa: PLC0415
        get_settings.cache_clear()

        async def override_db():
            async with client_api_session_factory() as s:
                async with s.begin():
                    yield s
                    await s.rollback()

        app.dependency_overrides[get_db_session] = override_db
        yield app
        app.dependency_overrides.pop(get_db_session, None)


    @pytest_asyncio.fixture(scope="module")
    async def seeded_upload_opportunity(superuser_session_factory):
        """Seed a pipeline.opportunities row for the upload test opportunity_id."""
        async with superuser_session_factory() as session:
            await session.execute(text("""
                INSERT INTO pipeline.opportunities (
                    id, source_id, source_type, title, opportunity_type, status,
                    deadline, budget_max, currency, country, region, cpv_codes,
                    published_at, created_at, updated_at
                ) VALUES (
                    :id, :source_id, 'aop', 'Document Upload Test Opportunity', 'tender',
                    'open', NOW() + INTERVAL '30 days', 400000, 'EUR', 'Bulgaria', 'Bulgaria',
                    '{45000000-7}'::text[], NOW(), NOW(), NOW()
                )
                ON CONFLICT (source_id, source_type) DO NOTHING
            """), {"id": _TEST_OPP_ID, "source_id": "doc-upload-test-opp"})
            await session.commit()

        yield _TEST_OPP_ID

        async with superuser_session_factory() as session:
            await session.execute(text(
                "DELETE FROM pipeline.opportunities WHERE source_id = 'doc-upload-test-opp'"
            ))
            await session.commit()


    @pytest.fixture
    def starter_headers(starter_user_token):
        return {"Authorization": f"Bearer {starter_user_token}"}


    @pytest.fixture
    def free_headers(free_user_token):
        return {"Authorization": f"Bearer {free_user_token}"}


    @pytest.fixture
    def internal_headers():
        return {"X-Internal-Service-Key": _INTERNAL_SECRET}


    def _upload_body(
        *,
        filename: str = "proposal.pdf",
        size: int = 1024 * 1024,  # 1 MB default
        mime_type: str = "application/pdf",
    ) -> dict:
        return {"filename": filename, "size": size, "mime_type": mime_type}


    # ---------------------------------------------------------------------------
    # Helpers
    # ---------------------------------------------------------------------------

    def _mock_presigned_url(monkeypatch):
        """Patch boto3 S3 client to return a fake presigned URL without real AWS calls."""
        mock_s3 = MagicMock()
        mock_s3.generate_presigned_url.return_value = "https://s3.example.com/mock-presigned-url"
        mock_s3.put_object_tagging.return_value = {}
        mock_s3.delete_object.return_value = {}
        return mock_s3


    # ---------------------------------------------------------------------------
    # E06-P2-005: File size > 100MB rejected (400) before presigned URL
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_upload_rejects_oversized_file(
        upload_app, test_client, starter_headers, seeded_upload_opportunity
    ):
        """E06-P2-005: File size > 100MB returns 400 before presigned URL issuance."""
        opp_id = seeded_upload_opportunity
        body = _upload_body(size=_MAX_FILE_BYTES + 1)  # 100MB + 1 byte
        resp = await test_client.post(
            f"/api/v1/opportunities/{opp_id}/documents/upload",
            headers=starter_headers,
            json=body,
        )
        assert resp.status_code == 400
        data = resp.json()
        assert data.get("error") == "validation_error"
        assert "size" in str(data).lower() or "100" in str(data)


    # ---------------------------------------------------------------------------
    # E06-P1-016: MIME type validation
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.parametrize("mime_type", _ALLOWED_TYPES)
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_upload_allowed_mime_types(
        upload_app, test_client, starter_headers, seeded_upload_opportunity, monkeypatch, mime_type
    ):
        """E06-P1-016: Allowed MIME types return 200 with presigned URL."""
        opp_id = seeded_upload_opportunity

        with patch("client_api.services.document_service._get_s3_client") as mock_get_s3:
            mock_get_s3.return_value = _mock_presigned_url(monkeypatch)
            ext = mime_type.split("/")[-1].replace(
                "vnd.openxmlformats-officedocument.wordprocessingml.document", "docx"
            ).replace(
                "vnd.openxmlformats-officedocument.spreadsheetml.sheet", "xlsx"
            )
            body = _upload_body(
                filename=f"test.{ext}",
                mime_type=mime_type,
            )
            resp = await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/upload",
                headers=starter_headers,
                json=body,
            )
        assert resp.status_code == 200, (
            f"Expected 200 for MIME type {mime_type}, got {resp.status_code}: {resp.text}"
        )
        data = resp.json()
        assert "doc_id" in data
        assert "presigned_url" in data
        assert data["expires_in"] == 900


    @pytest.mark.asyncio
    @pytest.mark.parametrize("mime_type", _REJECTED_TYPES)
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_upload_rejected_mime_types(
        upload_app, test_client, starter_headers, seeded_upload_opportunity, mime_type
    ):
        """E06-P1-016: JPEG and EXE MIME types return 400 validation_error."""
        opp_id = seeded_upload_opportunity
        body = _upload_body(filename="evil.exe", mime_type=mime_type)
        resp = await test_client.post(
            f"/api/v1/opportunities/{opp_id}/documents/upload",
            headers=starter_headers,
            json=body,
        )
        assert resp.status_code == 400, (
            f"Expected 400 for rejected MIME type {mime_type}, got {resp.status_code}"
        )
        data = resp.json()
        assert data.get("error") == "validation_error"


    # ---------------------------------------------------------------------------
    # Unauthenticated → 401
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_upload_requires_authentication(
        upload_app, test_client, seeded_upload_opportunity
    ):
        """Unauthenticated request to upload endpoint returns 401."""
        opp_id = seeded_upload_opportunity
        resp = await test_client.post(
            f"/api/v1/opportunities/{opp_id}/documents/upload",
            json=_upload_body(),
        )
        assert resp.status_code == 401


    # ---------------------------------------------------------------------------
    # Free tier → 403 tier_limit
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_upload_free_tier_blocked(
        upload_app, test_client, free_headers, seeded_upload_opportunity
    ):
        """Free-tier user receives 403 tier_limit on document upload endpoint."""
        opp_id = seeded_upload_opportunity
        resp = await test_client.post(
            f"/api/v1/opportunities/{opp_id}/documents/upload",
            headers=free_headers,
            json=_upload_body(),
        )
        assert resp.status_code == 403
        data = resp.json()
        assert "tier_limit" in str(data)
        assert "upgrade_url" in str(data)


    # ---------------------------------------------------------------------------
    # E06-P1-017: Valid upload → presigned URL; confirm → scan_status=pending
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_upload_presigned_url_and_confirm(
        upload_app, test_client, starter_headers, seeded_upload_opportunity,
        client_api_session_factory, monkeypatch
    ):
        """E06-P1-017: Valid upload request returns presigned URL with expires_in=900;
        confirm endpoint sets scan_status=pending in client.documents.
        """
        opp_id = seeded_upload_opportunity

        with patch("client_api.services.document_service._get_s3_client") as mock_get_s3:
            mock_get_s3.return_value = _mock_presigned_url(monkeypatch)

            # Step 1: Request presigned URL
            resp = await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/upload",
                headers=starter_headers,
                json=_upload_body(filename="technical_proposal.pdf", size=5 * 1024 * 1024),
            )
        assert resp.status_code == 200, f"Upload request failed: {resp.text}"
        data = resp.json()
        assert "doc_id" in data
        assert "presigned_url" in data
        assert data["presigned_url"].startswith("https://"), (
            "presigned_url should be an HTTPS URL"
        )
        assert data["expires_in"] == 900, (
            f"Expected expires_in=900 (15 min), got {data['expires_in']}"
        )
        doc_id = data["doc_id"]

        # Step 2: Confirm upload (mock Celery dispatch)
        with patch("client_api.services.document_service.dispatch_document_scan") as mock_dispatch:
            confirm_resp = await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/{doc_id}/confirm",
                headers=starter_headers,
            )
            assert mock_dispatch.called, "dispatch_document_scan must be called on confirm"

        assert confirm_resp.status_code == 200, f"Confirm failed: {confirm_resp.text}"
        confirm_data = confirm_resp.json()
        assert confirm_data["scan_status"] == "pending"
        assert confirm_data["doc_id"] == doc_id

        # Step 3: Verify client.documents record has scan_status=pending
        async with client_api_session_factory() as session:
            row = (
                await session.execute(
                    select(doc_t.c.scan_status, doc_t.c.s3_key)
                    .where(doc_t.c.id == uuid.UUID(doc_id))
                )
            ).mappings().first()

        assert row is not None, f"Document record not found in DB for doc_id={doc_id}"
        assert row["scan_status"] == "pending", (
            f"Expected scan_status=pending after confirm, got {row['scan_status']}"
        )
        # Verify S3 key includes company_id, opp_id, and filename
        assert "technical_proposal.pdf" in row["s3_key"]
        assert opp_id in row["s3_key"]


    # ---------------------------------------------------------------------------
    # E06-P0-006: scan_status=pending → document not downloadable (DB state check)
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_pending_document_not_downloadable_state(
        upload_app, test_client, starter_headers, seeded_upload_opportunity,
        client_api_session_factory, monkeypatch
    ):
        """E06-P0-006: After confirm, scan_status=pending — document is NOT in clean state.

        This test verifies the DB state directly.  The S06.07 download endpoint test
        (test_document_download.py) verifies the 422 HTTP response from the download endpoint.
        """
        opp_id = seeded_upload_opportunity

        with patch("client_api.services.document_service._get_s3_client") as mock_get_s3:
            mock_get_s3.return_value = _mock_presigned_url(monkeypatch)
            resp = await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/upload",
                headers=starter_headers,
                json=_upload_body(filename="pending_check.pdf", size=1024),
            )
        assert resp.status_code == 200
        doc_id = resp.json()["doc_id"]

        with patch("client_api.services.document_service.dispatch_document_scan"):
            await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/{doc_id}/confirm",
                headers=starter_headers,
            )

        # Verify: scan_status is 'pending', NOT 'clean'
        async with client_api_session_factory() as session:
            row = (
                await session.execute(
                    select(doc_t.c.scan_status).where(doc_t.c.id == uuid.UUID(doc_id))
                )
            ).mappings().first()

        assert row is not None
        assert row["scan_status"] == "pending", (
            "scan_status must remain 'pending' until ClamAV scan completes — "
            f"download must be blocked. Got: {row['scan_status']}"
        )


    # ---------------------------------------------------------------------------
    # E06-P0-007: ClamAV infected callback → scan_status=infected, deleted_at set
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_infected_scan_marks_deleted(
        upload_app, test_client, starter_headers, seeded_upload_opportunity,
        client_api_session_factory, internal_headers, monkeypatch
    ):
        """E06-P0-007: ClamAV infected callback sets scan_status=infected,
        deleted_at IS NOT NULL, and triggers S3 soft-delete (tag or delete).
        """
        opp_id = seeded_upload_opportunity

        # Create document record
        with patch("client_api.services.document_service._get_s3_client") as mock_get_s3:
            mock_s3_client = _mock_presigned_url(monkeypatch)
            mock_get_s3.return_value = mock_s3_client

            resp = await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/upload",
                headers=starter_headers,
                json=_upload_body(filename="infected.pdf", size=2048),
            )
        assert resp.status_code == 200
        doc_id = resp.json()["doc_id"]

        # Confirm to move to pending state
        with patch("client_api.services.document_service.dispatch_document_scan"):
            await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/{doc_id}/confirm",
                headers=starter_headers,
            )

        # Send infected scan result (simulates ClamAV callback without going through Celery)
        with patch("client_api.services.document_service._get_s3_client") as mock_s3_for_tag:
            mock_s3_for_tag.return_value = mock_s3_client

            scan_resp = await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/{doc_id}/scan-result",
                headers=internal_headers,
                json={"status": "infected"},
            )

        assert scan_resp.status_code == 200, f"scan-result failed: {scan_resp.text}"
        scan_data = scan_resp.json()
        assert scan_data["scan_status"] == "infected"
        assert scan_data["doc_id"] == doc_id

        # Verify DB: scan_status=infected, deleted_at IS NOT NULL
        async with client_api_session_factory() as session:
            row = (
                await session.execute(
                    select(doc_t.c.scan_status, doc_t.c.deleted_at)
                    .where(doc_t.c.id == uuid.UUID(doc_id))
                )
            ).mappings().first()

        assert row is not None
        assert row["scan_status"] == "infected", (
            f"Expected scan_status=infected, got {row['scan_status']}"
        )
        assert row["deleted_at"] is not None, (
            "deleted_at must be set (soft-delete) when scan result is infected"
        )

        # Verify S3 soft-delete was triggered (tag or delete called)
        assert (
            mock_s3_client.put_object_tagging.called
            or mock_s3_client.delete_object.called
        ), "S3 soft-delete (tagging or delete) must be triggered for infected documents"


    # ---------------------------------------------------------------------------
    # Clean scan result → scan_status=clean
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_clean_scan_marks_downloadable(
        upload_app, test_client, starter_headers, seeded_upload_opportunity,
        client_api_session_factory, internal_headers, monkeypatch
    ):
        """ClamAV clean callback sets scan_status=clean; deleted_at remains NULL."""
        opp_id = seeded_upload_opportunity

        with patch("client_api.services.document_service._get_s3_client") as mock_get_s3:
            mock_get_s3.return_value = _mock_presigned_url(monkeypatch)
            resp = await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/upload",
                headers=starter_headers,
                json=_upload_body(filename="clean.pdf", size=1024),
            )
        assert resp.status_code == 200
        doc_id = resp.json()["doc_id"]

        with patch("client_api.services.document_service.dispatch_document_scan"):
            await test_client.post(
                f"/api/v1/opportunities/{opp_id}/documents/{doc_id}/confirm",
                headers=starter_headers,
            )

        scan_resp = await test_client.post(
            f"/api/v1/opportunities/{opp_id}/documents/{doc_id}/scan-result",
            headers=internal_headers,
            json={"status": "clean"},
        )
        assert scan_resp.status_code == 200
        assert scan_resp.json()["scan_status"] == "clean"

        async with client_api_session_factory() as session:
            row = (
                await session.execute(
                    select(doc_t.c.scan_status, doc_t.c.deleted_at)
                    .where(doc_t.c.id == uuid.UUID(doc_id))
                )
            ).mappings().first()

        assert row["scan_status"] == "clean"
        assert row["deleted_at"] is None, "deleted_at must be NULL for clean documents"


    # ---------------------------------------------------------------------------
    # Scan-result endpoint: missing / invalid internal key → 401
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_scan_result_requires_internal_key(
        upload_app, test_client, seeded_upload_opportunity, monkeypatch
    ):
        """scan-result endpoint returns 401 without or with wrong internal key."""
        opp_id = seeded_upload_opportunity
        fake_doc_id = str(uuid.uuid4())

        # No key
        resp_no_key = await test_client.post(
            f"/api/v1/opportunities/{opp_id}/documents/{fake_doc_id}/scan-result",
            json={"status": "clean"},
        )
        assert resp_no_key.status_code == 401

        # Wrong key
        resp_wrong_key = await test_client.post(
            f"/api/v1/opportunities/{opp_id}/documents/{fake_doc_id}/scan-result",
            headers={"X-Internal-Service-Key": "wrong-secret"},
            json={"status": "clean"},
        )
        assert resp_wrong_key.status_code == 401


    # ---------------------------------------------------------------------------
    # E06-P2-004: Concurrent upload package size race
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    @pytest.mark.skip(reason="RED PHASE: S06.06 not yet implemented")
    async def test_concurrent_package_size_race(
        upload_app, test_client, starter_headers, seeded_upload_opportunity,
        client_api_session_factory, superuser_session_factory, monkeypatch
    ):
        """E06-P2-004: Two concurrent 300MB requests against 200MB existing usage.

        Only one should be accepted; total package size stays ≤ 500MB.
        Uses SELECT ... FOR UPDATE to prevent the race (E06-R-006 mitigation).

        Note: This test requires testcontainers PostgreSQL for proper locking behaviour
        (see E06-R-010 note — fakeredis is acceptable for serial tests but real DB
        is required for concurrency locking tests).
        """
        opp_id = seeded_upload_opportunity
        company_id = uuid.UUID(_STARTER_COMPANY_ID)

        # Seed an existing 200MB document for this company+opportunity
        existing_doc_id = uuid.uuid4()
        # Look up a real user_id for the starter company
        async with superuser_session_factory() as session:
            result = await session.execute(text("""
                SELECT u.id as user_id
                FROM client.users u
                JOIN client.company_memberships m ON m.user_id = u.id
                WHERE m.company_id = :company_id
                LIMIT 1
            """), {"company_id": str(company_id)})
            row = result.mappings().first()

        if row is None:
            pytest.skip("No Starter company user found — seed E02 test data first")

        user_id = row["user_id"]

        # Insert 200MB existing document
        async with superuser_session_factory() as session:
            await session.execute(
                doc_t.insert().values(
                    id=existing_doc_id,
                    opportunity_id=uuid.UUID(opp_id),
                    user_id=user_id,
                    company_id=company_id,
                    filename="existing.pdf",
                    size=200 * 1024 * 1024,  # 200MB
                    mime_type="application/pdf",
                    s3_key=f"documents/{company_id}/{opp_id}/{existing_doc_id}/existing.pdf",
                    scan_status="clean",
                )
            )
            await session.commit()

        try:
            # Fire two concurrent 300MB requests
            with patch("client_api.services.document_service._get_s3_client") as mock_get_s3:
                mock_get_s3.return_value = _mock_presigned_url(monkeypatch)

                results = await asyncio.gather(
                    test_client.post(
                        f"/api/v1/opportunities/{opp_id}/documents/upload",
                        headers=starter_headers,
                        json=_upload_body(filename="large_a.pdf", size=300 * 1024 * 1024),
                    ),
                    test_client.post(
                        f"/api/v1/opportunities/{opp_id}/documents/upload",
                        headers=starter_headers,
                        json=_upload_body(filename="large_b.pdf", size=300 * 1024 * 1024),
                    ),
                    return_exceptions=True,
                )

            statuses = [r.status_code for r in results if not isinstance(r, Exception)]
            success_count = sum(1 for s in statuses if s == 200)
            reject_count = sum(1 for s in statuses if s == 400)

            assert success_count == 1, (
                f"Exactly one 300MB upload should succeed with 200MB existing. "
                f"Got statuses: {statuses}"
            )
            assert reject_count == 1, (
                f"Exactly one 300MB upload should be rejected (package_size_limit). "
                f"Got statuses: {statuses}"
            )

            # Verify total package size ≤ 500MB
            async with client_api_session_factory() as session:
                total = (
                    await session.execute(text("""
                        SELECT COALESCE(SUM(size), 0) as total
                        FROM client.documents
                        WHERE opportunity_id = :opp_id
                          AND company_id = :company_id
                          AND deleted_at IS NULL
                    """), {"opp_id": opp_id, "company_id": str(company_id)})
                ).scalar()

            assert total <= _MAX_PKG_BYTES, (
                f"Total package size {total} bytes exceeds 500MB limit {_MAX_PKG_BYTES} bytes"
            )
        finally:
            # Cleanup seeded 200MB document
            async with superuser_session_factory() as session:
                await session.execute(text(
                    "DELETE FROM client.documents WHERE id = :id"
                ), {"id": str(existing_doc_id)})
                await session.commit()
    ```

## Dev Notes

### Architecture Context

S06.06 introduces the document management layer to the opportunity discovery surface.
Its relationship to the surrounding E06 stories:

| Story | Role | Endpoint(s) |
|-------|------|------------|
| S06.05 | Detail endpoint (reads documents list from `client.documents`) | `GET /api/v1/opportunities/{id}` |
| **S06.06** | **Document upload (this story)** | **`POST .../documents/upload`, `.../confirm`, `.../scan-result`** |
| S06.07 | Document download | `GET .../documents/{doc_id}/download` |
| S06.12 | Frontend upload component | wires to S06.06 endpoints |

S06.06 introduces:
- Alembic migration `019_documents` — the `client.documents` table (also consumed by S06.07)
- `models/client_document.py` — Core table definition
- Config settings: `documents_s3_bucket`, `document_presigned_put_expiry`, `max_document_size_bytes`, `max_package_size_bytes`
- `schemas/documents.py` — request/response schemas plus ALLOWED_MIME_TYPES constant
- `services/document_service.py` — upload, confirm, scan-result logic
- `api/v1/documents.py` — FastAPI endpoints (separate router registered under `/api/v1/opportunities`)
- `celery_producer.dispatch_document_scan()` function

### CRITICAL: `client.documents` MetaData Isolation

`client_document.py` uses its own `client_metadata = MetaData(schema="client")` instance —
the **same** isolation pattern as `client_ai_summary.py`. Do NOT share the pipeline MetaData.
Both `client.documents` and `client.ai_summaries` need their own `MetaData(schema="client")`
instances to ensure correct schema-qualified SQL generation.

If you need to query both tables in the same service function, import them separately:

```python
from client_api.models.client_document import documents_table as doc_t
from client_api.models.client_ai_summary import ai_summaries_table as ai_t
```

Each table uses its own MetaData instance — no schema collision.

### CRITICAL: Package Size Race — SELECT … FOR UPDATE

The `SELECT SUM(size) … FOR UPDATE` pattern (AC: 4) is the core concurrency safety mechanism.
It locks all non-deleted document rows for the `(opportunity_id, company_id)` pair within the
current transaction, preventing two concurrent requests from both reading the same pre-insert
total and both passing the 500MB check.

The lock is held until the transaction commits (via `get_db_session`). This means:
1. Request A reads SUM = 200MB (FOR UPDATE — other requests wait)
2. Request A inserts 300MB document (total = 500MB)
3. Request A commits → FOR UPDATE lock released
4. Request B reads SUM = 500MB → 400 package_size_limit

The `with_for_update()` SQLAlchemy API generates `SELECT … FOR UPDATE` in PostgreSQL.
In tests, this requires a real PostgreSQL connection — `fakeredis` is irrelevant here;
the standard `client_api_session_factory` backed by `postgresql+asyncpg` provides the
correct PostgreSQL locking semantics.

**Do NOT** use a Python-level in-memory lock (threading.Lock, asyncio.Lock) — it would only
protect against races within a single process, not across gunicorn/uvicorn worker processes.

### CRITICAL: Documents Router Mounting in `main.py`

The `documents_v1.router` has `prefix="/{opportunity_id}/documents"` (relative).
It must be registered with `prefix="/opportunities"` on the `api_v1_router`:

```python
api_v1_router.include_router(documents_v1.router, prefix="/opportunities")
```

This produces the full paths:
- `POST /api/v1/opportunities/{opportunity_id}/documents/upload`
- `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/confirm`
- `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/scan-result`

The `opportunity_id` in the documents router is a path parameter that is **not** validated
against `pipeline.opportunities` (no existence check) — the upload endpoint is decoupled from
the pipeline schema. Validation that the opportunity exists is the responsibility of the frontend
(which navigates to the detail page first) and the download endpoint (S06.07).

### CRITICAL: `subscription_tier` on `CurrentUser`

The tier check in `documents.py` accesses `current_user.subscription_tier`. Verify that
`CurrentUser` (in `core/security.py`) exposes this attribute — it was added in S06.02 when
`OpportunityTierGateContext` was implemented. If `CurrentUser` only has `user_id`, `company_id`,
`role`, a new field `subscription_tier` may need to be added. Check the JWT claim name
(`subscription_tier`) and the `CurrentUser` dataclass/model.

If `CurrentUser` does not have `subscription_tier`, use the `OpportunityTierGateContext`
dependency (same as the detail endpoint) instead of a direct `current_user.subscription_tier`
check — this avoids duplicating the tier-reading logic.

### CRITICAL: `opportunity_id` Parameter in `scan-result` Route

The `opportunity_id` path parameter appears in the scan-result route signature (for URL
namespace consistency: `POST .../documents/{doc_id}/scan-result`) but is not used by the
service function — the ClamAV worker identifies the document by `doc_id` alone.

Mark the parameter with `# noqa: ARG001` or use `_opportunity_id` naming to suppress
unused-parameter linter warnings. The parameter must remain in the path for URL consistency
with the upload and confirm endpoints and to satisfy the OpenAPI schema (AC9).

### CRITICAL: Celery Task Dispatch Failure Handling

`confirm_upload()` catches all exceptions from `dispatch_document_scan()` and logs them
without re-raising. This is intentional:

- In CI / dev environments, the Celery broker (Redis) may be unavailable.
- The `scan_status` stays `pending` even if dispatch fails.
- A background timeout job (E06-R-008 mitigation, S06.06 scope extension or separate task)
  will eventually mark stuck `pending` documents as `failed`.
- The download endpoint (S06.07) blocks all non-`clean` documents, so a stuck `pending`
  document is inaccessible but does not expose a security gap.

In production, Celery dispatch failures should trigger an alert (e.g. via Celery beat health
check or a dead-letter queue). This is an operational concern not in scope for S06.06.

### CRITICAL: S3 Client Reuse Pattern

`document_service._get_s3_client()` and `report_service._get_s3_client()` are identical
in implementation. This duplication is intentional — the two services use different S3 buckets
(`documents_s3_bucket` vs `report_s3_bucket`) and may evolve independently. A future
refactor can extract to `utils/s3.py`, but that is out of scope for S06.06.

When mocking in tests, patch `client_api.services.document_service._get_s3_client` (not
`boto3.client` directly) to ensure the mock is scoped correctly.

### CRITICAL: Internal Scan-Result Authentication

The scan-result endpoint uses `verify_scan_callback_auth()` (in `document_service.py`) which
calls `secrets.compare_digest()` against `settings.internal_service_secret`. This is the same
`internal_service_secret` setting used by the enterprise-API proxy auth (`internal_auth.py`).

In tests, set `CLIENT_API_INTERNAL_SERVICE_SECRET=test-internal-secret-s06-06` via env var
in the `upload_app` fixture. The `get_settings.cache_clear()` call in the fixture ensures the
new value is picked up by the cached singleton.

### CRITICAL: `scan_status` Never Defaults to `clean`

The DB column has `DEFAULT 'pending'` — not `DEFAULT 'clean'`. The `request_upload` service
function explicitly sets `scan_status = SCAN_STATUS_PENDING` in the INSERT. The `confirm_upload`
function does NOT change `scan_status` — it stays `pending` until the ClamAV callback arrives.

This is the core security invariant: **no document is ever downloadable (S06.07 checks
`scan_status = clean`) without an explicit ClamAV clean verdict** (E06-R-003 mitigation).

### CRITICAL: `scan-result` Idempotency

`record_scan_result()` is idempotent: if the document is already in `clean` or `infected`
state when the callback arrives, the function logs a warning and returns the current state
without modification. This handles duplicate Celery deliveries safely.

### Test Infrastructure Notes

**Moto / boto3 mock:**
The `upload_app` fixture patches `document_service._get_s3_client` at the module level.
Tests that exercise S3 interaction (E06-P1-017, E06-P0-007) use `patch()` as a context
manager around the API calls. The mock returns a `MagicMock` with `generate_presigned_url`
returning a fake HTTPS URL and `put_object_tagging`/`delete_object` no-ops.

**Alternative (moto library):**
If `moto` is available (`pip install moto[s3]`), you can use:
```python
from moto import mock_s3
@mock_s3
def my_test():
    boto3.client("s3", ...).create_bucket(Bucket="eusolicit-documents-test")
    ...
```
This provides a real in-memory S3 that generates valid presigned URLs. If `moto` is available
in the project's test dependencies, prefer it over manual `MagicMock`.

**Concurrent test (E06-P2-004):**
Uses `asyncio.gather()` to fire two simultaneous requests. Requires a real PostgreSQL
connection (testcontainers or the dev DB) — the `SELECT … FOR UPDATE` lock test is meaningless
without a real relational DB transaction manager.

### File Locations

| Purpose | Path |
|---------|------|
| Alembic migration (`client.documents`) | `eusolicit-app/services/client-api/alembic/versions/019_documents.py` |
| Core table definition | `eusolicit-app/services/client-api/src/client_api/models/client_document.py` |
| Config additions | `eusolicit-app/services/client-api/src/client_api/config.py` |
| Pydantic schemas | `eusolicit-app/services/client-api/src/client_api/schemas/documents.py` |
| Document service | `eusolicit-app/services/client-api/src/client_api/services/document_service.py` |
| Route handlers | `eusolicit-app/services/client-api/src/client_api/api/v1/documents.py` |
| Celery dispatch function | `eusolicit-app/services/client-api/src/client_api/celery_producer.py` |
| Router registration | `eusolicit-app/services/client-api/src/client_api/main.py` |
| Integration tests | `eusolicit-app/services/client-api/tests/api/test_document_upload.py` |
| Existing internal auth (reference) | `eusolicit-app/services/client-api/src/client_api/core/internal_auth.py` |
| Previous migration (down_revision) | `eusolicit-app/services/client-api/alembic/versions/018_ai_summaries.py` |

### Test Design Traceability

| Test ID | Priority | Scenario | AC |
|---------|----------|----------|----|
| E06-P0-006 | P0 | scan_status=pending in DB after confirm — download endpoint must block | AC6 |
| E06-P0-007 | P0 | ClamAV infected callback: scan_status=infected, deleted_at set, S3 soft-delete | AC7 |
| E06-P1-016 | P1 | MIME type validation: 4 allowed types (200); JPEG + EXE rejected (400) | AC3 |
| E06-P1-017 | P1 | Valid upload → presigned URL expires_in=900; confirm → scan_status=pending | AC5, AC6 |
| E06-P2-004 | P2 | Concurrent 300MB+300MB against 200MB existing; only one accepted (SELECT FOR UPDATE) | AC4 |
| E06-P2-005 | P2 | File > 100MB rejected (400) before presigned URL issuance | AC3 |
| (Implicit) | — | Unauthenticated → 401 | AC2 |
| (Implicit) | — | Free tier → 403 tier_limit | AC2 |
| (Implicit) | — | scan-result missing internal key → 401 | AC7 |
| (Implicit) | — | Clean scan → scan_status=clean, deleted_at=NULL | AC7 |

Risk mitigations relevant to this story:
- **E06-R-003** (S3/ClamAV pre-scan gap — score 6): `scan_status` defaults to `pending` (never `clean`); the confirm endpoint does NOT change `scan_status` to `clean`; S06.07 download endpoint checks `scan_status = clean` before issuing presigned GET URL. Tests E06-P0-006 (pending blocks download) and E06-P0-007 (infected → soft-delete) both verify this invariant.
- **E06-R-006** (concurrent package size race — score 4): `SELECT SUM(size) … FOR UPDATE` prevents two concurrent uploads from both passing the 500MB check. Test E06-P2-004 verifies with `asyncio.gather` against a real PostgreSQL connection.
- **E06-R-008** (ClamAV async timeout — score 4): `scan_status` never transitions to `clean` on timeout; a future background job (out of scope for S06.06) will mark stuck pending documents as `failed`. The DB partial index `ix_documents_pending` on `scan_status = 'pending'` supports efficient timeout job queries.

### References

- [Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.06] — S3 presigned URL, ClamAV scan, `client.documents` schema, 100MB/500MB limits, allowed MIME types
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P0-006] — P0: scan pending blocks download
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P0-007] — P0: infected scan → soft-delete + download blocked
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P1-016] — P1: MIME type validation
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P1-017] — P1: presigned URL 15-min expiry; scan_status=pending after confirm
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P2-004] — P2: concurrent package size race
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-P2-005] — P2: 100MB per-file limit
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-R-003] — ClamAV pre-scan gap (score 6)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-R-006] — Package size race (score 4)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#E06-R-008] — ClamAV timeout (score 4)
- [Source: eusolicit-app/services/client-api/src/client_api/services/report_service.py] — `_get_s3_client()` S3 pattern reused
- [Source: eusolicit-app/services/client-api/src/client_api/celery_producer.py] — `dispatch_generate_report` pattern for `dispatch_document_scan`
- [Source: eusolicit-app/services/client-api/src/client_api/core/internal_auth.py] — Internal service key auth pattern reused for scan-result callback
- [Source: eusolicit-app/services/client-api/alembic/versions/018_ai_summaries.py] — Previous migration (down_revision for 019)
- [Source: eusolicit-app/services/client-api/src/client_api/models/client_ai_summary.py] — MetaData isolation pattern for client-schema tables

## Dev Agent Record

### Implementation Plan

Addressed 7 code review patch findings from BMad Code Review:
1. **P1** — Filename path traversal sanitization via Pydantic `field_validator`
2. **P2** — `FOR UPDATE` on `record_scan_result` SELECT to prevent TOCTOU race
3. **P3** — `FOR UPDATE` on `confirm_upload` SELECT to prevent duplicate ClamAV dispatches
4. **P4** — `pg_advisory_xact_lock` for first-upload race protection on empty result sets
5. **P5** — `asyncio.to_thread()` for blocking S3 I/O in `record_scan_result`
6. **P6** — `Literal["clean", "infected"]` type on `ScanResultRequest.status`
7. **P7** — `application/x-zip-compressed` added to test parametrize list

### Completion Notes

All 7 review patch findings resolved. 21 tests pass (5 new filename validation + 1 new MIME type + 15 existing). Zero regressions introduced. Ruff linting clean. Story ready for final review.

## File List

| Action | Path |
|--------|------|
| Modified | `eusolicit-app/services/client-api/src/client_api/schemas/documents.py` |
| Modified | `eusolicit-app/services/client-api/src/client_api/services/document_service.py` |
| Modified | `eusolicit-app/services/client-api/tests/api/test_document_upload.py` |

## Senior Developer Review

### Review 2 (Final)

**Date:** 2026-04-17
**Reviewer:** BMad Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** APPROVED

**Summary:** 0 patch, 0 decision-needed, 0 new findings. All 7 previous patch findings verified resolved. 7 deferred items remain (pre-existing or architectural). Clean review — all three adversarial layers passed.

**Verification of Previous Patch Fixes:**

- [x] **P1 — Filename path traversal**: `field_validator` on `DocumentUploadRequest.filename` rejects `/`, `\`, `..`, null bytes, and control characters. 5 parametrized tests confirm rejection with 422. ✅
- [x] **P2 — TOCTOU in `record_scan_result`**: `.with_for_update()` added to fetch_stmt. Prevents duplicate Celery delivery race (infected/clean overwrite). ✅
- [x] **P3 — TOCTOU in `confirm_upload`**: `.with_for_update()` added to fetch_stmt. Prevents duplicate ClamAV dispatch. ✅
- [x] **P4 — Empty result set race**: `pg_advisory_xact_lock` keyed on MD5 hash of `(opportunity_id, company_id)` serialises concurrent first-uploads even when no rows exist. ✅
- [x] **P5 — Blocking S3 I/O**: `await asyncio.to_thread(_soft_delete_s3_object, s3_key)` wraps synchronous boto3 calls. ✅
- [x] **P6 — Literal type**: `ScanResultRequest.status` changed to `Literal["clean", "infected"]`. OpenAPI schema correctly documents enum. ✅
- [x] **P7 — Missing test MIME type**: `application/x-zip-compressed` added to `_ALLOWED_TYPES` test list. All 5 MIME types covered. ✅

**Acceptance Criteria Audit (all 10 ACs verified):**

| AC | Status | Evidence |
|----|--------|----------|
| AC1 | ✅ | Migration `019_documents.py` — all columns, types, FKs, composite + single + partial indexes |
| AC2 | ✅ | JWT auth via `get_current_user`; paid tier gate via `_PAID_TIERS` set; tested 401 + 403 |
| AC3 | ✅ | 5 MIME types in `ALLOWED_MIME_TYPES`; 100MB per-file check; tested with parametrize |
| AC4 | ✅ | FOR UPDATE subquery + `pg_advisory_xact_lock`; 500MB package limit; concurrent race test |
| AC5 | ✅ | Presigned PUT URL with ExpiresIn=900 + ContentType; INSERT with `scan_status='pending'` |
| AC6 | ✅ | Ownership + pending check; uploaded_at update; Celery dispatch; 404/422 errors |
| AC7 | ✅ | `X-Internal-Service-Key` via `secrets.compare_digest`; status update; infected → soft-delete |
| AC8 | ✅ | `dispatch_document_scan` follows `dispatch_generate_report` pattern; correct task name |
| AC9 | ✅ | Router registered with descriptions, response_models, path params in OpenAPI |
| AC10 | ✅ | 21 tests: MIME (7), size (1), auth (2), flow (3), scan (3), race (1), filename (5) |

**Deferred Items (unchanged from Review 1 — no action required for this story):**

- [x] [Review][Defer] Migration revision naming convention break — deferred, documented design decision
- [x] [Review][Defer] Config `env=` parameter inconsistency — deferred, pre-existing pattern
- [x] [Review][Defer] `internal_service_secret` alias bypasses `CLIENT_API_` prefix — deferred, pre-existing
- [x] [Review][Defer] `confirm_upload` does not verify S3 object exists via `HEAD` — deferred, design tradeoff
- [x] [Review][Defer] No `Content-Length` constraint on presigned PUT URL — deferred, architectural decision needed
- [x] [Review][Defer] `uploaded_at` set at INSERT then overwritten at confirm — deferred, semantic clarification
- [x] [Review][Defer] `create_access_token` does not include `subscription_tier` in JWT payload — deferred, pre-existing

### Review 1

**Date:** 2026-04-17
**Reviewer:** BMad Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** CHANGES REQUESTED

**Summary:** 7 patch, 7 defer, 9 dismissed as noise.

### Review 1 Findings (all resolved)

- [x] [Review][Patch] **P1: Filename path traversal — no sanitization before S3 key construction** — `document_service.py:213` constructs S3 key via `f"documents/{company_id}/{opportunity_id}/{doc_id}/{filename}"` with no validation for path separators (`/`, `\`), `..` sequences, null bytes, or control characters. While S3 treats keys literally (no true directory traversal), unsanitized filenames create misleading key hierarchies, risk header injection in the future download endpoint's `Content-Disposition`, and violate defense-in-depth. Fix: add a Pydantic `field_validator` on `DocumentUploadRequest.filename` that rejects path separators, `..`, null bytes, and control characters (e.g., `if '/' in v or '\\' in v or '..' in v or any(ord(c) < 32 for c in v): raise ValueError`). [document_service.py:213, schemas/documents.py:63]
- [x] [Review][Patch] **P2: TOCTOU race in `record_scan_result` — no row-level locking** — `document_service.py:390-424` does a plain `SELECT` (no `FOR UPDATE`) to check `scan_status`, then `UPDATE`s. Two concurrent scan-result callbacks (duplicate Celery delivery) can both read `pending`, both pass the terminal-state check, and the last writer wins — potentially overwriting `infected` with `clean`. Fix: add `.with_for_update()` to the `fetch_stmt` in `record_scan_result`. [document_service.py:390]
- [x] [Review][Patch] **P3: TOCTOU race in `confirm_upload` — no row-level locking** — `document_service.py:290-301` does a plain `SELECT` for the document. Two concurrent confirm calls both pass the `pending` check and both dispatch ClamAV Celery tasks, causing duplicate scans. Fix: add `.with_for_update()` to the `fetch_stmt` in `confirm_upload`. [document_service.py:290]
- [x] [Review][Patch] **P4: `FOR UPDATE` ineffective on empty result set — first-upload race** — `document_service.py:190-198` — `SELECT ... FOR UPDATE` on zero rows acquires no locks. Two concurrent first-uploads for a new `(opportunity_id, company_id)` pair both see `SUM=0` and both INSERT, potentially exceeding the 500MB package limit. Test E06-P2-004 only tests with a pre-existing document, not the empty-table case. Fix: add a `pg_advisory_xact_lock` keyed on a hash of `(opportunity_id, company_id)` before the SUM query to serialize even when no rows exist. [document_service.py:188]
- [x] [Review][Patch] **P5: Blocking synchronous S3 I/O on asyncio event loop** — `document_service.py:430` calls `_soft_delete_s3_object()` which makes synchronous boto3 network calls (`put_object_tagging`, `delete_object`) from within `async def record_scan_result`. This blocks the event loop for the duration of the S3 API calls. Fix: wrap the call with `await asyncio.to_thread(_soft_delete_s3_object, s3_key)`. [document_service.py:430]
- [x] [Review][Patch] **P6: `ScanResultRequest.status` should use `Literal` type** — `schemas/documents.py:116` types `status` as bare `str`. The service layer validates at line 382 but the schema itself accepts arbitrary strings, weakening defense-in-depth and OpenAPI documentation. Fix: change to `status: Literal["clean", "infected"]`. [schemas/documents.py:116]
- [x] [Review][Patch] **P7: Test `_ALLOWED_TYPES` list missing `application/x-zip-compressed`** — `test_document_upload.py:48-53` parametrizes 4 MIME types but `ALLOWED_MIME_TYPES` in `schemas/documents.py` contains 5 (including `application/x-zip-compressed`). The 5th type is untested. Fix: add `"application/x-zip-compressed"` to `_ALLOWED_TYPES` test list. [test_document_upload.py:48]

## Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-17 | BMad Orchestrator (bmad-create-story) | Story file created from epic-06 S06.06 spec and test-design-epic-06.md; includes migration 019_documents, client_document Core table, config additions (documents_s3_bucket, presigned PUT expiry, size limits), schemas/documents.py (ALLOWED_MIME_TYPES, request/response models), document_service.py (request_upload with SELECT FOR UPDATE race protection, confirm_upload with Celery dispatch, record_scan_result with S3 soft-delete), api/v1/documents.py (upload, confirm, scan-result routes), celery_producer.dispatch_document_scan, main.py registration, and integration tests; test design traceability to E06-P0-006, P0-007, P1-016, P1-017, P2-004, P2-005; critical notes on MetaData isolation, FOR UPDATE race protection, scan_status never defaulting to clean, and Celery dispatch failure tolerance |
| 2026-04-17 | Dev Agent (Claude) | Implemented all 9 tasks. Key implementation decisions: (1) `down_revision = "018"` (not "018_ai_summaries" — actual revision ID of previous migration is "018"); (2) FOR UPDATE on subquery pattern to avoid PostgreSQL "FOR UPDATE not allowed with aggregate functions" error; (3) `subscription_tier` added to `CurrentUser` dataclass (default "free") extracted from JWT `payload.get("subscription_tier", "free")`; (4) conftest.py token fixtures upgraded from HS256 to RS256 using session-scoped `rsa_test_key_pair` (required for `get_current_user` RS256 validation); (5) `starter_user_token` fixture uses real user_id `190d12d0-e227-463e-9bd8-289b3276a274` (FK constraint on client.documents.user_id); (6) `upload_app` fixture changed from rollback-per-request to commit-per-request with `cleanup_documents` autouse fixture for test isolation (rollback broke multi-step tests: upload→confirm→verify-DB); (7) `INTERNAL_SERVICE_SECRET` env var (no CLIENT_API_ prefix) — pydantic-settings v2 uses alias directly; (8) concurrent test temporarily overrides `CLIENT_API_MAX_DOCUMENT_SIZE_BYTES=500MB` so 300MB test files pass per-file check. All 15 tests pass; 32 pre-existing failures in E12/E11 migrations and proposal-export module are unrelated. |
| 2026-04-17 | BMad Code Review (bmad-code-review) | Review: Changes Requested. 7 patch findings, 7 deferred, 9 dismissed. Critical: (P1) filename path traversal — no sanitization before S3 key construction; (P2-P3) TOCTOU races in record_scan_result and confirm_upload — no FOR UPDATE on SELECT; (P4) FOR UPDATE ineffective on empty result set — first-upload race unprotected; (P5) blocking S3 I/O on asyncio event loop; (P6) ScanResultRequest.status should use Literal type; (P7) test missing application/x-zip-compressed MIME type. Status → in-progress. |
| 2026-04-17 | Dev Agent (Claude) | Addressed all 7 code review patch findings: (P1) Added `field_validator` on `DocumentUploadRequest.filename` rejecting path separators, `..`, null bytes, and control characters — 5 parametrized tests added; (P2) Added `.with_for_update()` to `record_scan_result` fetch_stmt preventing TOCTOU race on duplicate Celery delivery; (P3) Added `.with_for_update()` to `confirm_upload` fetch_stmt preventing duplicate ClamAV dispatches; (P4) Added `pg_advisory_xact_lock` keyed on MD5 hash of `(opportunity_id, company_id)` before FOR UPDATE subquery — serialises concurrent first-uploads even when no rows exist; (P5) Wrapped `_soft_delete_s3_object` call with `asyncio.to_thread()` to avoid blocking event loop; (P6) Changed `ScanResultRequest.status` from `str` to `Literal["clean", "infected"]` for defense-in-depth and OpenAPI docs; (P7) Added `application/x-zip-compressed` to test `_ALLOWED_TYPES` list. All 21 tests pass (5 new + 16 existing); 0 regressions. Ruff clean. Status → review. |
| 2026-04-17 | BMad Code Review (bmad-code-review) | Review 2: Approved. All 7 previous patch findings verified resolved. All 10 ACs pass adversarial audit (Blind Hunter + Edge Case Hunter + Acceptance Auditor). 21 tests cover all required scenarios. No new patch or blocking findings. 7 deferred items unchanged (pre-existing or architectural). Status → done. |
