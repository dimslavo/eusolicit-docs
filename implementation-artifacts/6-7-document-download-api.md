# Story 6.7: Document Download API

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **paid-tier user who needs to retrieve an uploaded opportunity document**,
I want **`GET /api/v1/opportunities/{opportunity_id}/documents/{document_id}/download` to verify my access rights (document owner or same-company paid-tier member), verify the document passed the ClamAV scan (`scan_status = clean`), generate a short-lived S3 presigned GET URL with `Content-Disposition: attachment`, log the download event to the audit trail, and return structured error responses for denied (403), not-found (404), and unclean (422) states**,
so that **only authorized users can retrieve clean documents, every download is auditable, and infected or unscanned files can never be delivered to the browser**.

## Acceptance Criteria

1. `GET /api/v1/opportunities/{opportunity_id}/documents/{document_id}/download` requires a valid Bearer JWT; returns 401 if absent or invalid.

2. Access control check: the requesting user must be either (a) the document owner (`document.user_id == current_user.user_id`) OR (b) a member of the same company (`document.company_id == current_user.company_id`) with a paid subscription tier (`starter`, `professional`, `enterprise`). Returns 403 with `{"error": "access_denied", "detail": "You do not have access to this document."}` when neither condition is met.

3. Returns 404 with `{"error": "not_found", "detail": "Document not found."}` when no non-soft-deleted row exists for the given `document_id` in `client.documents` (i.e. `deleted_at IS NULL`).

4. Returns 422 with `{"error": "scan_pending", "detail": "Document is awaiting virus scan. Please try again shortly."}` when `scan_status = 'pending'`.

5. Returns 422 with `{"error": "scan_failed", "detail": "Document failed virus scan and cannot be downloaded."}` when `scan_status = 'infected'` or `scan_status = 'failed'`.

6. For authorized access to a clean document: generates an S3 presigned GET URL using `s3.generate_presigned_url("get_object", ...)` with `ExpiresIn = settings.document_presigned_get_expiry` (default 600 seconds, 10 minutes) and `ResponseContentDisposition = f'attachment; filename="{document.filename}"'` in the `Params` dict. Returns `{"download_url": "<url>", "expires_in": 600, "filename": "<filename>"}` with HTTP 200.

7. On successful presigned URL generation: writes an audit log entry via `write_audit_entry(session, user_id=current_user.user_id, action_type="document.download", entity_type="document", entity_id=document_id)`.

8. `document_presigned_get_expiry: int` is added to `ClientApiSettings` in `config.py` with `default=600`, `env="DOCUMENT_PRESIGNED_GET_EXPIRY"`, and an appropriate description.

9. The endpoint appears in `/openapi.json` with correct path parameters, response schema, and response descriptions for 200, 401, 403, 404, and 422 status codes.

10. Integration tests in `tests/api/test_document_download.py` cover:
    - 200: document owner downloads a clean document; presigned URL returned with `expires_in = 600` and `filename` present (E06-P1-018)
    - 200: same-company paid-tier member (non-owner) downloads successfully — `Content-Disposition` attachment filename verified (E06-P1-018)
    - 403: different-company user denied (E06-P1-019)
    - 403: free-tier same-company user denied (AC2 company-access requires paid tier)
    - 404: non-existent or soft-deleted document_id
    - 422: `scan_status = 'pending'` returns `scan_pending` error (E06-P0-006 full gate)
    - 422: `scan_status = 'infected'` returns `scan_failed` error (E06-P0-007 full gate)
    - 422: `scan_status = 'failed'` returns `scan_failed` error
    - Audit log entry present in `shared.audit_log` after successful download (E06-P1-020)
    - 401: unauthenticated request denied

## Tasks / Subtasks

- [x] Task 1: Add `DocumentDownloadResponse` schema to `schemas/documents.py` (AC: 6, 9)
  - [x] 1.1 Append to `eusolicit-app/services/client-api/src/client_api/schemas/documents.py` after the existing `ScanResultResponse` class:

    ```python
    # ---------------------------------------------------------------------------
    # Download response — S06.07
    # ---------------------------------------------------------------------------

    class DocumentDownloadResponse(BaseModel):
        """Presigned S3 GET URL returned for document download.

        The browser must follow this URL within `expires_in` seconds.
        The presigned URL encodes Content-Disposition: attachment so the browser
        downloads the file rather than attempting to render it inline.
        """

        download_url: str
        expires_in: int = Field(description="Seconds until presigned URL expires (600 = 10 min).")
        filename: str = Field(description="Original filename for user display.")
    ```

- [x] Task 2: Add `document_presigned_get_expiry` setting to `config.py` (AC: 8)
  - [x] 2.1 Edit `eusolicit-app/services/client-api/src/client_api/config.py` — append after `max_package_size_bytes` field inside `ClientApiSettings`:

    ```python
        document_presigned_get_expiry: int = Field(
            default=600,
            env="DOCUMENT_PRESIGNED_GET_EXPIRY",
            description="Presigned GET URL lifetime in seconds for document download (default 10 minutes).",
        )
    ```

- [x] Task 3: Add `download_document` service function to `document_service.py` (AC: 2, 3, 4, 5, 6, 7)
  - [x] 3.1 Add imports to `eusolicit-app/services/client-api/src/client_api/services/document_service.py` — add to existing import block:

    ```python
    from client_api.schemas.documents import (
        ...  # existing imports
        SCAN_STATUS_FAILED,  # already defined
        DocumentDownloadResponse,  # new
    )
    from client_api.services.audit_service import write_audit_entry
    ```

    Note: `SCAN_STATUS_FAILED` may already be imported (check existing imports in file); `DocumentDownloadResponse` is new from Task 1.

  - [x] 3.2 Append `download_document` function to `document_service.py`:

    ```python
    # ---------------------------------------------------------------------------
    # download_document — S06.07
    # ---------------------------------------------------------------------------

    #: Paid plans that grant company-level (non-owner) document access.
    _DOWNLOAD_PAID_PLANS: frozenset[str] = frozenset({"starter", "professional", "enterprise"})


    async def download_document(
        *,
        db: AsyncSession,
        opportunity_id: UUID,
        document_id: UUID,
        user_id: UUID,
        company_id: UUID,
        subscription_tier: str,
    ) -> DocumentDownloadResponse:
        """Access-check a document and return a presigned S3 GET URL for download.

        Access rules:
          1. Document owner (user_id match) — always allowed if document is clean.
          2. Same-company member with a paid subscription tier — allowed if clean.
          3. Any other caller — 403.

        Scan status gates:
          - 'pending'               → 422 scan_pending
          - 'infected' or 'failed'  → 422 scan_failed
          - 'clean'                 → proceed to presigned URL

        Parameters
        ----------
        db:
            Active AsyncSession — SELECT only; no writes except the audit flush.
        opportunity_id:
            Path parameter; used only to scope the query and validate the document
            belongs to the stated opportunity (no cross-opportunity access).
        document_id:
            Primary key of the ``client.documents`` row.
        user_id, company_id, subscription_tier:
            From the authenticated ``CurrentUser``.

        Returns
        -------
        DocumentDownloadResponse
            download_url: S3 presigned GET URL (10-minute expiry by default).
            expires_in: configured expiry seconds (default 600).
            filename: original filename for client display.

        Raises
        ------
        AppException (404)  Document not found or soft-deleted.
        AppException (403)  Access denied (wrong user, wrong company, or free tier).
        AppException (422)  Document scan is pending, infected, or failed.
        AppException (503)  S3 presigned URL generation failed.
        """
        settings = get_settings()

        # --- Fetch document (non-soft-deleted rows only) ---
        stmt = (
            select(
                doc_t.c.id,
                doc_t.c.opportunity_id,
                doc_t.c.user_id,
                doc_t.c.company_id,
                doc_t.c.filename,
                doc_t.c.s3_key,
                doc_t.c.scan_status,
                doc_t.c.deleted_at,
            )
            .where(doc_t.c.id == document_id)
            .where(doc_t.c.opportunity_id == opportunity_id)
            .where(doc_t.c.deleted_at.is_(None))
        )
        row = (await db.execute(stmt)).mappings().one_or_none()

        if row is None:
            raise AppException(
                "Document not found.",
                error="not_found",
                status_code=404,
            )

        # --- Access control ---
        is_owner = row["user_id"] == user_id
        is_company_member = row["company_id"] == company_id
        is_paid = subscription_tier in _DOWNLOAD_PAID_PLANS

        if not is_owner and not (is_company_member and is_paid):
            log.warning(
                "document_download.access_denied",
                document_id=str(document_id),
                user_id=str(user_id),
                doc_company=str(row["company_id"]),
                user_company=str(company_id),
                tier=subscription_tier,
            )
            raise AppException(
                "You do not have access to this document.",
                error="access_denied",
                status_code=403,
            )

        # --- Scan status gate ---
        scan_status: str = row["scan_status"]
        if scan_status == SCAN_STATUS_PENDING:
            raise AppException(
                "Document is awaiting virus scan. Please try again shortly.",
                error="scan_pending",
                status_code=422,
            )
        if scan_status in (SCAN_STATUS_INFECTED, SCAN_STATUS_FAILED):
            raise AppException(
                "Document failed virus scan and cannot be downloaded.",
                error="scan_failed",
                status_code=422,
            )
        if scan_status != SCAN_STATUS_CLEAN:
            # Guard against unexpected scan_status values
            raise AppException(
                "Document is not available for download.",
                error="scan_failed",
                status_code=422,
            )

        # --- Generate presigned GET URL ---
        filename: str = row["filename"]
        s3_key: str = row["s3_key"]
        expiry: int = settings.document_presigned_get_expiry

        try:
            s3 = _get_s3_client()
            download_url = s3.generate_presigned_url(
                "get_object",
                Params={
                    "Bucket": settings.documents_s3_bucket,
                    "Key": s3_key,
                    "ResponseContentDisposition": f'attachment; filename="{filename}"',
                },
                ExpiresIn=expiry,
            )
        except Exception as exc:  # noqa: BLE001
            log.error(
                "document_download.presigned_url_failed",
                document_id=str(document_id),
                error=str(exc),
            )
            raise AppException(
                "Failed to generate download URL. Please try again.",
                error="s3_error",
                status_code=503,
            ) from exc

        # --- Audit trail (never raises — write_audit_entry swallows errors) ---
        await write_audit_entry(
            db,
            user_id=user_id,
            action_type="document.download",
            entity_type="document",
            entity_id=document_id,
        )

        log.info(
            "document_download.url_generated",
            document_id=str(document_id),
            user_id=str(user_id),
            filename=filename,
        )

        return DocumentDownloadResponse(
            download_url=download_url,
            expires_in=expiry,
            filename=filename,
        )
    ```

- [x] Task 4: Add `GET /{doc_id}/download` route to `documents.py` router (AC: 1, 9)
  - [x] 4.1 Edit `eusolicit-app/services/client-api/src/client_api/api/v1/documents.py` — add import and new route:

    Add to imports block:
    ```python
    from client_api.schemas.documents import (
        DocumentConfirmResponse,
        DocumentDownloadResponse,   # new
        DocumentUploadRequest,
        DocumentUploadResponse,
        ScanResultRequest,
        ScanResultResponse,
    )
    ```

    Append new route after `confirm_document_upload`:
    ```python
    # ---------------------------------------------------------------------------
    # GET /{doc_id}/download — generate presigned GET URL (S06.07)
    # ---------------------------------------------------------------------------


    @router.get(
        "/{doc_id}/download",
        summary="Generate presigned S3 GET URL for document download",
        description=(
            "Verifies access rights (document owner or same-company paid-tier member), "
            "verifies scan_status = clean, and returns a presigned S3 GET URL with "
            "Content-Disposition: attachment and 10-minute expiry. "
            "Returns 403 if access denied, 404 if not found, "
            "422 if scan is pending/infected/failed."
        ),
        response_model=DocumentDownloadResponse,
        responses={
            403: {"description": "Access denied — not owner or wrong tier"},
            404: {"description": "Document not found"},
            422: {"description": "Document scan pending, infected, or failed"},
        },
    )
    async def download_document(
        opportunity_id: UUID,
        doc_id: UUID,
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        db: Annotated[AsyncSession, Depends(get_db_session)],
    ) -> DocumentDownloadResponse:
        """Return presigned S3 GET URL for a clean, accessible document.

        AC1: Requires Bearer JWT.
        AC2: Access control — owner or paid company member.
        AC3–5: 404/422 for not-found/unclean documents.
        AC6: Presigned URL with attachment Content-Disposition, 10-min expiry.
        AC7: Audit log entry written on success.
        """
        return await document_service.download_document(
            db=db,
            opportunity_id=opportunity_id,
            document_id=doc_id,
            user_id=current_user.user_id,
            company_id=current_user.company_id,
            subscription_tier=current_user.subscription_tier,
        )
    ```

    CRITICAL: Register the `GET /{doc_id}/download` route BEFORE `POST /{doc_id}/confirm` in the file if there are any ordering issues, OR confirm FastAPI handles `GET` vs `POST` on the same path pattern correctly (FastAPI does route by method, so ordering is not an issue here — both `/{doc_id}/confirm` as POST and `/{doc_id}/download` as GET can coexist safely).

- [x] Task 5: Create `tests/api/test_document_download.py` (AC: 10)
  - [x] 5.1 Create `eusolicit-app/services/client-api/tests/api/test_document_download.py`:

    ```python
    """Integration tests for Document Download API — Story S06.07.

    Test IDs covered:
        E06-P0-006  scan_status=pending blocks download → 422 scan_pending (full gate)
        E06-P0-007  scan_status=infected blocks download → 422 scan_failed (full gate)
        E06-P1-018  Clean document: presigned GET URL with expires_in=600 and filename returned;
                    Content-Disposition attachment filename encoded in presigned URL params
        E06-P1-019  Non-owner without tier access denied → 403 access_denied
        E06-P1-020  Audit log entry written after successful download
        (Implicit)  scan_status=failed → 422 scan_failed
        (Implicit)  Free-tier same-company user → 403 (no company-level access without paid tier)
        (Implicit)  Non-existent / soft-deleted document_id → 404
        (Implicit)  Unauthenticated request → 401

    Fixtures:
        download_app  — overrides get_db_session + mocks boto3 S3 (MagicMock)
        seeded_document — inserts a client.documents row in the requested scan_status

    Stack: backend / FastAPI / pytest-asyncio / httpx AsyncClient

    Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#P0-006, P0-007, P1-018,
            P1-019, P1-020
    """
    from __future__ import annotations

    import uuid
    from unittest.mock import MagicMock, patch

    import httpx
    import pytest
    import pytest_asyncio
    from sqlalchemy import insert, select, text

    from client_api.models.audit_log import AuditLog
    from client_api.models.client_document import documents_table as doc_t

    # ---------------------------------------------------------------------------
    # Constants
    # ---------------------------------------------------------------------------

    _TEST_OPP_ID = str(uuid.uuid4())
    _OWNER_USER_ID = "aaaaaaaa-0000-0000-0000-000000000001"
    _OWNER_COMPANY_ID = "bbbbbbbb-0000-0000-0000-000000000001"
    _OTHER_USER_ID = "cccccccc-0000-0000-0000-000000000002"
    _OTHER_COMPANY_ID = "dddddddd-0000-0000-0000-000000000002"
    _SAME_COMPANY_USER_ID = "eeeeeeee-0000-0000-0000-000000000003"

    _FAKE_PRESIGNED_URL = "https://s3.amazonaws.com/eusolicit-documents-dev/fake-key?X-Amz-Signature=abc"


    # ---------------------------------------------------------------------------
    # Fixtures
    # ---------------------------------------------------------------------------

    @pytest_asyncio.fixture
    async def download_app(app, client_api_session_factory):
        """Override get_db_session + mock boto3 S3 for download tests.

        Uses MagicMock to patch boto3 so no real AWS credentials are needed.
        Each HTTP request gets its own session with a real transaction.
        Per-test isolation provided by cleanup_documents autouse fixture.
        """
        import os
        from client_api.dependencies import get_db_session

        os.environ.setdefault("AWS_DEFAULT_REGION", "eu-central-1")
        os.environ.setdefault("DOCUMENTS_S3_BUCKET", "eusolicit-documents-dev")
        os.environ.setdefault("DOCUMENT_PRESIGNED_GET_EXPIRY", "600")

        async def _session_override():
            async with client_api_session_factory() as session:
                async with session.begin():
                    yield session

        app.dependency_overrides[get_db_session] = _session_override
        yield app
        app.dependency_overrides.pop(get_db_session, None)


    @pytest_asyncio.fixture(autouse=True)
    async def cleanup_documents(client_api_session_factory):
        """Delete all test documents before each test for isolation."""
        yield
        async with client_api_session_factory() as session:
            async with session.begin():
                await session.execute(
                    doc_t.delete().where(
                        doc_t.c.opportunity_id == uuid.UUID(_TEST_OPP_ID)
                    )
                )


    async def _seed_document(
        session_factory,
        *,
        doc_id: str | None = None,
        opportunity_id: str = _TEST_OPP_ID,
        user_id: str = _OWNER_USER_ID,
        company_id: str = _OWNER_COMPANY_ID,
        scan_status: str = "clean",
        deleted: bool = False,
        filename: str = "proposal.pdf",
    ) -> str:
        """Insert a client.documents row and return the doc_id string."""
        _doc_id = doc_id or str(uuid.uuid4())
        async with session_factory() as session:
            async with session.begin():
                from datetime import UTC, datetime
                await session.execute(
                    doc_t.insert().values(
                        id=uuid.UUID(_doc_id),
                        opportunity_id=uuid.UUID(opportunity_id),
                        user_id=uuid.UUID(user_id),
                        company_id=uuid.UUID(company_id),
                        filename=filename,
                        size=1024,
                        mime_type="application/pdf",
                        s3_key=f"documents/{company_id}/{opportunity_id}/{_doc_id}/{filename}",
                        scan_status=scan_status,
                        deleted_at=datetime.now(UTC) if deleted else None,
                        created_at=datetime.now(UTC),
                        uploaded_at=datetime.now(UTC),
                    )
                )
        return _doc_id


    def _make_jwt(app, user_id: str, company_id: str, tier: str = "starter") -> str:
        """Issue a signed test JWT for the given user/company/tier."""
        from client_api.core.security import create_access_token
        return create_access_token(
            user_id=uuid.UUID(user_id),
            company_id=uuid.UUID(company_id),
            role="contributor",
            subscription_tier=tier,
        )


    # ---------------------------------------------------------------------------
    # E06-P1-018: Owner downloads clean document — presigned URL returned
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_owner_downloads_clean_document(download_app, client_api_session_factory):
        """E06-P1-018: Document owner receives presigned GET URL with expires_in=600."""
        doc_id = await _seed_document(client_api_session_factory, scan_status="clean")
        token = _make_jwt(download_app, _OWNER_USER_ID, _OWNER_COMPANY_ID, tier="starter")

        mock_s3 = MagicMock()
        mock_s3.generate_presigned_url.return_value = _FAKE_PRESIGNED_URL

        with patch("client_api.services.document_service._get_s3_client", return_value=mock_s3):
            async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
                resp = await ac.get(
                    f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                    headers={"Authorization": f"Bearer {token}"},
                )

        assert resp.status_code == 200
        data = resp.json()
        assert data["download_url"] == _FAKE_PRESIGNED_URL
        assert data["expires_in"] == 600
        assert data["filename"] == "proposal.pdf"

        # Verify Content-Disposition: attachment was requested in presigned URL params
        call_kwargs = mock_s3.generate_presigned_url.call_args
        params = call_kwargs[1]["Params"] if call_kwargs[1] else call_kwargs[0][1]["Params"]
        assert "ResponseContentDisposition" in params
        assert "attachment" in params["ResponseContentDisposition"]
        assert "proposal.pdf" in params["ResponseContentDisposition"]
        assert call_kwargs[1].get("ExpiresIn") == 600 or call_kwargs[0][-1] == 600


    # ---------------------------------------------------------------------------
    # E06-P1-018 (part 2): Same-company paid-tier member downloads successfully
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_same_company_paid_member_downloads(download_app, client_api_session_factory):
        """E06-P1-018: Same-company paid-tier non-owner can download clean document."""
        doc_id = await _seed_document(
            client_api_session_factory,
            scan_status="clean",
            user_id=_OWNER_USER_ID,
            company_id=_OWNER_COMPANY_ID,
        )
        # Different user, same company, paid tier
        token = _make_jwt(download_app, _SAME_COMPANY_USER_ID, _OWNER_COMPANY_ID, tier="professional")

        mock_s3 = MagicMock()
        mock_s3.generate_presigned_url.return_value = _FAKE_PRESIGNED_URL

        with patch("client_api.services.document_service._get_s3_client", return_value=mock_s3):
            async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
                resp = await ac.get(
                    f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                    headers={"Authorization": f"Bearer {token}"},
                )

        assert resp.status_code == 200
        assert resp.json()["download_url"] == _FAKE_PRESIGNED_URL


    # ---------------------------------------------------------------------------
    # E06-P1-019: Different-company user denied
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_different_company_denied(download_app, client_api_session_factory):
        """E06-P1-019: User from a different company receives 403 access_denied."""
        doc_id = await _seed_document(client_api_session_factory, scan_status="clean")
        token = _make_jwt(download_app, _OTHER_USER_ID, _OTHER_COMPANY_ID, tier="professional")

        mock_s3 = MagicMock()
        mock_s3.generate_presigned_url.return_value = _FAKE_PRESIGNED_URL

        with patch("client_api.services.document_service._get_s3_client", return_value=mock_s3):
            async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
                resp = await ac.get(
                    f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                    headers={"Authorization": f"Bearer {token}"},
                )

        assert resp.status_code == 403
        assert resp.json()["error"] == "access_denied"


    # ---------------------------------------------------------------------------
    # AC2 (implicit): Same-company free-tier user denied
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_same_company_free_tier_denied(download_app, client_api_session_factory):
        """Free-tier same-company user (non-owner) cannot download — 403."""
        doc_id = await _seed_document(
            client_api_session_factory,
            scan_status="clean",
            user_id=_OWNER_USER_ID,
            company_id=_OWNER_COMPANY_ID,
        )
        token = _make_jwt(download_app, _SAME_COMPANY_USER_ID, _OWNER_COMPANY_ID, tier="free")

        mock_s3 = MagicMock()
        mock_s3.generate_presigned_url.return_value = _FAKE_PRESIGNED_URL

        with patch("client_api.services.document_service._get_s3_client", return_value=mock_s3):
            async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
                resp = await ac.get(
                    f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                    headers={"Authorization": f"Bearer {token}"},
                )

        assert resp.status_code == 403


    # ---------------------------------------------------------------------------
    # E06-P0-006 (full gate): scan_status=pending → 422 scan_pending
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_pending_scan_returns_422(download_app, client_api_session_factory):
        """E06-P0-006: Document with scan_status=pending cannot be downloaded (422)."""
        doc_id = await _seed_document(client_api_session_factory, scan_status="pending")
        token = _make_jwt(download_app, _OWNER_USER_ID, _OWNER_COMPANY_ID)

        async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
            resp = await ac.get(
                f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                headers={"Authorization": f"Bearer {token}"},
            )

        assert resp.status_code == 422
        assert resp.json()["error"] == "scan_pending"


    # ---------------------------------------------------------------------------
    # E06-P0-007 (full gate): scan_status=infected → 422 scan_failed
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_infected_scan_returns_422(download_app, client_api_session_factory):
        """E06-P0-007: Document with scan_status=infected cannot be downloaded (422)."""
        doc_id = await _seed_document(client_api_session_factory, scan_status="infected")
        token = _make_jwt(download_app, _OWNER_USER_ID, _OWNER_COMPANY_ID)

        async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
            resp = await ac.get(
                f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                headers={"Authorization": f"Bearer {token}"},
            )

        assert resp.status_code == 422
        assert resp.json()["error"] == "scan_failed"


    # ---------------------------------------------------------------------------
    # scan_status=failed → 422 scan_failed
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_failed_scan_returns_422(download_app, client_api_session_factory):
        """scan_status=failed (timeout) cannot be downloaded (422 scan_failed)."""
        doc_id = await _seed_document(client_api_session_factory, scan_status="failed")
        token = _make_jwt(download_app, _OWNER_USER_ID, _OWNER_COMPANY_ID)

        async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
            resp = await ac.get(
                f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                headers={"Authorization": f"Bearer {token}"},
            )

        assert resp.status_code == 422
        assert resp.json()["error"] == "scan_failed"


    # ---------------------------------------------------------------------------
    # 404: non-existent document
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_nonexistent_document_returns_404(download_app, client_api_session_factory):
        """Non-existent document_id returns 404 not_found."""
        token = _make_jwt(download_app, _OWNER_USER_ID, _OWNER_COMPANY_ID)
        random_doc_id = str(uuid.uuid4())

        async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
            resp = await ac.get(
                f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{random_doc_id}/download",
                headers={"Authorization": f"Bearer {token}"},
            )

        assert resp.status_code == 404
        assert resp.json()["error"] == "not_found"


    # ---------------------------------------------------------------------------
    # 404: soft-deleted document
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_soft_deleted_document_returns_404(download_app, client_api_session_factory):
        """Soft-deleted (infected) document returns 404 — deleted_at IS NOT NULL."""
        doc_id = await _seed_document(
            client_api_session_factory, scan_status="infected", deleted=True
        )
        token = _make_jwt(download_app, _OWNER_USER_ID, _OWNER_COMPANY_ID)

        async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
            resp = await ac.get(
                f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                headers={"Authorization": f"Bearer {token}"},
            )

        assert resp.status_code == 404


    # ---------------------------------------------------------------------------
    # E06-P1-020: Audit log written on successful download
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_audit_log_written_on_download(download_app, client_api_session_factory):
        """E06-P1-020: Successful download writes audit entry to shared.audit_log."""
        doc_id = await _seed_document(client_api_session_factory, scan_status="clean")
        token = _make_jwt(download_app, _OWNER_USER_ID, _OWNER_COMPANY_ID)

        mock_s3 = MagicMock()
        mock_s3.generate_presigned_url.return_value = _FAKE_PRESIGNED_URL

        with patch("client_api.services.document_service._get_s3_client", return_value=mock_s3):
            async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
                resp = await ac.get(
                    f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{doc_id}/download",
                    headers={"Authorization": f"Bearer {token}"},
                )
        assert resp.status_code == 200

        # Verify audit log entry was written
        async with client_api_session_factory() as session:
            result = await session.execute(
                select(AuditLog).where(
                    AuditLog.entity_type == "document",
                    AuditLog.entity_id == uuid.UUID(doc_id),
                    AuditLog.action_type == "document.download",
                    AuditLog.user_id == uuid.UUID(_OWNER_USER_ID),
                )
            )
            entry = result.scalar_one_or_none()

        assert entry is not None, "Audit log entry for document.download not found"


    # ---------------------------------------------------------------------------
    # 401: unauthenticated
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_unauthenticated_returns_401(download_app):
        """Unauthenticated request returns 401."""
        random_doc_id = str(uuid.uuid4())

        async with httpx.AsyncClient(app=download_app, base_url="http://test") as ac:
            resp = await ac.get(
                f"/api/v1/opportunities/{_TEST_OPP_ID}/documents/{random_doc_id}/download",
            )

        assert resp.status_code == 401
    ```

    Note: The `insert` import from sqlalchemy is included above but `_seed_document` uses `doc_t.insert()` directly. `text` is available if needed for raw queries. Remove unused imports if the linter flags them.

## Dev Notes

### What This Story Adds

Story 6.7 adds the download leg of the document management flow started in S06.06. Story 6.6 implemented upload (presigned PUT), confirm, scan-result callback, and the `client.documents` table. This story adds the `GET .../download` route and service function.

**No new Alembic migration is required.** The `client.documents` table (migration `019_documents.py`) with all needed columns (`scan_status`, `s3_key`, `filename`, `deleted_at`) was created in S06.06.

### Files to Touch

| File | Action | Notes |
|------|--------|-------|
| `src/client_api/schemas/documents.py` | Append `DocumentDownloadResponse` class | End of file; after `ScanResultResponse` |
| `src/client_api/config.py` | Append `document_presigned_get_expiry` field | Inside `ClientApiSettings`; after `max_package_size_bytes` |
| `src/client_api/services/document_service.py` | Add import + `download_document` function | Re-use existing `_get_s3_client()` helper |
| `src/client_api/api/v1/documents.py` | Add `DocumentDownloadResponse` import + `GET /{doc_id}/download` route | After `confirm_document_upload` handler |
| `tests/api/test_document_download.py` | Create new test file | Mirror patterns from `test_document_upload.py` |

**Do NOT** create a new Alembic migration, a new model file, or a new service file — all infrastructure is reused from S06.06.

### S3 Presigned GET URL — Critical Parameter

For the download endpoint, the `Content-Disposition: attachment` header must be encoded into the presigned URL using the `ResponseContentDisposition` parameter (not a response header set by the API):

```python
s3.generate_presigned_url(
    "get_object",
    Params={
        "Bucket": settings.documents_s3_bucket,
        "Key": s3_key,
        "ResponseContentDisposition": f'attachment; filename="{filename}"',
    },
    ExpiresIn=600,
)
```

This is different from the PUT URL in S06.06 which used `ContentType`. The `ResponseContentDisposition` tells S3 to inject the header when the browser follows the presigned URL, so the browser downloads instead of rendering inline.

For the report service comparison: `report_service._refresh_presigned_url` uses `"get_object"` but without `ResponseContentDisposition` (reports have their own download URL field). Follow the same `_get_s3_client()` pattern from `document_service.py` — it already exists in that file, so no duplication needed.

### Existing `_get_s3_client()` in `document_service.py`

The `_get_s3_client()` helper is **already defined in `document_service.py`** from S06.06. The `download_document` function calls it directly — no import needed, no duplication.

### Access Control Logic

The epic says "document owner or have appropriate tier access to the opportunity." Implementation:

```
is_owner = row["user_id"] == user_id  # exact user match
is_company_member = row["company_id"] == company_id  # same company
is_paid = subscription_tier in {"starter", "professional", "enterprise"}

allow = is_owner OR (is_company_member AND is_paid)
```

Free-tier users who are document owners can download their own documents — they could not have uploaded (upload requires paid tier per S06.06 AC2), so in practice this path is never reached. The implementation is still correct — it does not gate owner access on tier, only company-member access.

Test E06-P1-019 tests a **different-company** user: both `is_owner` and `is_company_member` are False → 403.

### Audit Trail Pattern

Use `write_audit_entry` from `client_api.services.audit_service` (already used in `report_service.py`):

```python
await write_audit_entry(
    db,
    user_id=user_id,
    action_type="document.download",
    entity_type="document",
    entity_id=document_id,
)
```

The `write_audit_entry` function **never raises** (swallows exceptions, logs warnings) — the download will succeed even if the audit write fails. This matches the pattern in `report_service.py`.

Action type naming convention (from S02.11): `"{entity}.{verb}"` — `"document.download"` follows this pattern.

### `scan_status` Constant Imports

`document_service.py` already imports `SCAN_STATUS_PENDING`, `SCAN_STATUS_CLEAN`, `SCAN_STATUS_INFECTED` from `schemas/documents.py`. The `SCAN_STATUS_FAILED` constant is defined in `schemas/documents.py` but may not be in the existing import list — check and add if absent.

### `CurrentUser.subscription_tier` Field

The `CurrentUser` dataclass in `core/security.py` includes `subscription_tier: str = "free"` (defaults to free). This is populated from the JWT `subscription_tier` claim by `get_current_user`. The `download_document` service function receives this value directly — no additional DB query needed for the tier check.

This differs from `get_opportunity_tier_gate` which queries `client.subscriptions` for a DB-backed tier. For document access control, the JWT claim is authoritative (same as upload tier check in `documents.py` router).

### Router Pattern

The `documents.py` router prefix is `/{opportunity_id}/documents`. It is mounted on the opportunities router. The download route will be `GET /{doc_id}/download` — this creates the full path `GET /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/download`.

FastAPI handles method-based routing cleanly: `POST /{doc_id}/confirm` and `GET /{doc_id}/download` on the same path template coexist without ordering issues.

### Testing — Mock Pattern for S3

Follow the same mock pattern as `test_document_upload.py`:

```python
mock_s3 = MagicMock()
mock_s3.generate_presigned_url.return_value = _FAKE_PRESIGNED_URL

with patch("client_api.services.document_service._get_s3_client", return_value=mock_s3):
    # ... make HTTP request
```

For audit log verification: query `shared.audit_log` via the test session factory using SQLAlchemy ORM (`AuditLog` model from `client_api.models.audit_log`).

### Scan Timeout Edge Case (E06-P2-006)

The scan timeout transition (`pending → failed` after CLAMAV_SCAN_TIMEOUT) is implemented in a background job (specified in test design E06-P2-006, S06.06 AC-related). The download gate for `scan_status = 'failed'` is part of this story — the service function already returns 422 for `failed`. The background timeout job itself is out of scope for this story.

### Project Structure Notes

- All files follow the `eusolicit-app/services/client-api/src/client_api/` package structure
- New test file goes in `tests/api/` (not `tests/integration/`) — API-level test with mocked S3
- No `__init__.py` changes needed (the directory already has `__init__.py`)
- `from __future__ import annotations` at top of all Python files (project standard)

### References

- Story 6.6 (immediate predecessor): `eusolicit-docs/implementation-artifacts/6-6-document-upload-api-s3-clamav.md`
- Epic AC: `eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#S06.07`
- Test design: `eusolicit-docs/test-artifacts/test-design-epic-06.md#P0-006, P0-007, P1-018, P1-019, P1-020`
- `document_service.py`: `eusolicit-app/services/client-api/src/client_api/services/document_service.py`
- `documents.py` router: `eusolicit-app/services/client-api/src/client_api/api/v1/documents.py`
- `schemas/documents.py`: `eusolicit-app/services/client-api/src/client_api/schemas/documents.py`
- `config.py`: `eusolicit-app/services/client-api/src/client_api/config.py`
- `report_service.py` (presigned GET URL pattern): `eusolicit-app/services/client-api/src/client_api/services/report_service.py#_refresh_presigned_url`
- `audit_service.py`: `eusolicit-app/services/client-api/src/client_api/services/audit_service.py`
- `test_document_upload.py` (testing patterns): `eusolicit-app/services/client-api/tests/api/test_document_upload.py`

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

N/A — all tests pass cleanly.

### Completion Notes List

1. **`eusolicit_common/exceptions.py`** (backward-compatible): Added `"detail": exc.message` to the `app_exception_handler` response envelope. The pre-written ATDD tests assert `"detail" in body` and `body["detail"]`, but the existing handler only exported `"message"`. Adding `"detail"` as an alias resolves this without breaking any existing tests. (`DEVIATION_TYPE: CONTRADICTORY_SPEC`, `DEVIATION_SEVERITY: deferrable`).

2. **Test file FK constraint fix**: The pre-written test had hardcoded fake user/company UUIDs (`aaaaaaaa-…`, `bbbbbbbb-…`) that violate the FK constraints of `client.documents → client.users` and `client.documents → client.companies`. Updated `_OWNER_USER_ID` to the real test-DB seeded value `190d12d0-e227-463e-9bd8-289b3276a274` and `_OWNER_COMPANY_ID` to `12345678-0000-0000-0000-000000000001`. Fake IDs for the JWT-only actors (`_OTHER_USER_ID`, `_OTHER_COMPANY_ID`, `_SAME_COMPANY_USER_ID`) remain unchanged since they never hit the DB.

3. **`_make_jwt` helper fix**: The pre-written test's `_make_jwt` called `create_access_token(…, subscription_tier=tier)`, but `create_access_token` does not accept that parameter. Updated `_make_jwt` to use `jwt.encode()` directly (the same pattern as the conftest's `_make_rs256_token`), allowing arbitrary `subscription_tier` values in test JWTs.

4. **OpenAPI 401 response**: Added `401: {"description": "Unauthorized — missing or invalid Bearer token"}` to the route's `responses` dict. The `test_download_endpoint_in_openapi` test checks for all of 200, 401, 403, 404, 422.

5. **`document_presigned_get_expiry` env var**: The story spec says `env="DOCUMENT_PRESIGNED_GET_EXPIRY"` but the test fixture sets `CLIENT_API_DOCUMENT_PRESIGNED_GET_EXPIRY`. Implemented WITHOUT the `env=` override so the `CLIENT_API_` prefix applies — consistent with the `document_presigned_put_expiry` pattern. The default of 600 satisfies the test expectation regardless of which env var is checked.

### File List

- `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/exceptions.py` (modified — add `"detail"` key)
- `eusolicit-app/services/client-api/src/client_api/schemas/documents.py` (modified — add `DocumentDownloadResponse`)
- `eusolicit-app/services/client-api/src/client_api/config.py` (modified — add `document_presigned_get_expiry`)
- `eusolicit-app/services/client-api/src/client_api/services/document_service.py` (modified — imports + `download_document` function)
- `eusolicit-app/services/client-api/src/client_api/api/v1/documents.py` (modified — import + `GET /{doc_id}/download` route)
- `eusolicit-app/services/client-api/tests/api/test_document_download.py` (modified — removed skip markers, fixed user/company IDs and `_make_jwt`)

## Senior Developer Review

### Review Findings

Reviewed by `bmad-code-review` at 2026-04-17 — 3 adversarial layers (Blind Hunter, Edge Case Hunter, Acceptance Auditor).

**Verdict: APPROVE** — All 10 acceptance criteria pass. 0 patch, 4 deferred, 9 dismissed.

- [x] [Review][Defer] Content-Disposition filename sanitization — filenames with double quotes could break `attachment; filename="..."` in presigned URL. Upload validator (S06.06) blocks `/`, `\`, `..`, null, control chars but not `"`. Defense-in-depth: escape quotes or use RFC 5987 `filename*` encoding. Pre-existing cross-cutting pattern. [`document_service.py:625`] — deferred, pre-existing
- [x] [Review][Defer] Missing test coverage for special-character filenames — all 12 tests use `proposal.pdf`. No coverage for filenames with `"`, `;`, spaces, or unicode in Content-Disposition path. Not in AC10 test list. [`test_document_download.py`] — deferred, not in AC scope
- [x] [Review][Defer] No bounds on `document_presigned_get_expiry` config field — Field accepts 0 or negative values, which would produce instantly-expiring URLs. Consistent with `document_presigned_put_expiry` pattern. [`config.py:105`] — deferred, pre-existing
- [x] [Review][Defer] Case-sensitive tier matching in `_DOWNLOAD_PAID_PLANS` — no `.lower()` normalization on `subscription_tier` from JWT. If upstream ever produces `Starter` instead of `starter`, paid users would be denied. Consistent with upload tier check in `documents.py` router. [`document_service.py:574`] — deferred, pre-existing

**AC Coverage Matrix:**
| AC | Status | Evidence |
|----|--------|----------|
| AC1 (401 auth) | PASS | `Depends(get_current_user)` + test `test_unauthenticated_returns_401` |
| AC2 (403 access) | PASS | `is_owner OR (is_company_member AND is_paid)` + 3 tests |
| AC3 (404 not found) | PASS | `WHERE deleted_at IS NULL` + 2 tests (nonexistent + soft-deleted) |
| AC4 (422 pending) | PASS | `scan_status == SCAN_STATUS_PENDING` + test |
| AC5 (422 infected/failed) | PASS | `scan_status in (INFECTED, FAILED)` + 2 tests |
| AC6 (200 presigned URL) | PASS | `ResponseContentDisposition`, `ExpiresIn=expiry` + 2 tests |
| AC7 (audit log) | PASS | `write_audit_entry(...)` + test verifies DB row |
| AC8 (config field) | PASS | `document_presigned_get_expiry: int = Field(default=600)` |
| AC9 (OpenAPI) | PASS | `responses={401,403,404,422}` + test `test_download_endpoint_in_openapi` |
| AC10 (tests) | PASS | 12 tests covering all specified cases + OpenAPI smoke |

**Dismissed findings (9):** JWT company_id trust (by design), rate limiting (not in scope), presigned URL log leak (false positive — URL not logged), s3_key traversal (not user-controlled), TOCTOU race on scan_status (false positive — scan transitions are one-way from pending), audit write error propagation (swallowed by design), owner bypasses tier (intentional per AC2), extra error envelope keys (cross-cutting convention), path param `doc_id` vs `document_id` (consistent router pattern).

## Known Deviations

### Detected by `2-dev-story` at 2026-04-17T06:08:42Z (session bec44272-8b1e-4ec3-a595-e0606bd0ac92)

- Pre-written tests assert "detail" key but AppException handler uses "message"` — `DEVIATION_TYPE: CONTRADICTORY_SPEC`, `DEVIATION_SEVERITY: deferrable` — resolved by adding `"detail"` as alias in error response _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
- Test hardcoded fake user/company UUIDs violated FK constraints` — `DEVIATION_TYPE: ACCEPTANCE_GAP`, `DEVIATION_SEVERITY: deferrable` — resolved by updating to real seeded test DB IDs _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- Pre-written tests assert "detail" key but AppException handler uses "message"` — `DEVIATION_TYPE: CONTRADICTORY_SPEC`, `DEVIATION_SEVERITY: deferrable` — resolved by adding `"detail"` as alias in error response _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
- Test hardcoded fake user/company UUIDs violated FK constraints` — `DEVIATION_TYPE: ACCEPTANCE_GAP`, `DEVIATION_SEVERITY: deferrable` — resolved by updating to real seeded test DB IDs _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
