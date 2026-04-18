# Story 7.10: Document Export API (PDF & DOCX)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager**,
I want to export a finished proposal to a downloadable PDF or DOCX file with company branding, section headers, page numbers, and a table of contents,
so that I can submit a professionally formatted tender response document to a procurement body directly from EU Solicit.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/proposals/{proposal_id}/export` accepts a JSON body with `format` param (`"pdf"` or `"docx"`). Requires `bid_manager` role. `company_id` derived exclusively from JWT — never accepted from any request field. Returns 422 on missing or invalid `format` value (e.g., `"xml"` → 422 with Pydantic validation error).

2. **AC2** — For `format: "pdf"`: generates a PDF document using `render_pdf` from `eusolicit_common.document_generation` (via `client_api.utils.document_export`). The PDF contains:
   - A title page: company name (Heading1), proposal title (Heading2), generation timestamp.
   - A "Table of Contents" page: `HeaderSection(level=1, text="Table of Contents")` followed by a `TextSection` listing all section titles (one per line, numbered).
   - One `HeaderSection(level=1, text=section["title"])` + `TextSection(body=section["body"])` pair per proposal section, in order.
   - Page numbers at the bottom of each page (provided by the shared renderer's `_page_number_footer` callback — no additional implementation required).
   - Valid PDF bytes (magic bytes `b"%PDF"`) returned as a streaming download.

3. **AC3** — For `format: "docx"`: generates a DOCX document using `render_docx` from `eusolicit_common.document_generation` (via `client_api.utils.document_export`). The DOCX contains the same structure as AC2 (title page, TOC section, section headers + body). Valid DOCX bytes returned as a streaming download (ZIP magic bytes `b"PK"`).

4. **AC4** — Response headers:
   - `Content-Type: application/pdf` for PDF; `Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document` for DOCX.
   - `Content-Disposition: attachment; filename="proposal-{proposal_id}.pdf"` (or `.docx`).
   - Endpoint returns HTTP 200 with the file bytes as the response body.

5. **AC5** — Returns HTTP 404 if the proposal does not exist or belongs to a different company (404 — not 403; UUID enumeration prevention, E07-R-003 pattern).

6. **AC6** — Size limits (E07-R-010 mitigation): if the proposal has more than `settings.export_max_sections` sections OR the total content size (sum of all `body` field lengths) exceeds `settings.export_max_content_bytes`, return HTTP 400 with `{"error": "proposal_too_large", "detail": "<reason>"}`. Default limits: `EXPORT_MAX_SECTIONS=50`, `EXPORT_MAX_CONTENT_BYTES=524288` (512 KB). Both are configurable via environment variables.

7. **AC7** — Empty proposal handling: if `current_version_id` is `None` or the version `content.sections` list is empty, the export proceeds gracefully — produce a valid minimal PDF/DOCX (title page + empty body text "No content available") rather than raising a 500.

8. **AC8** — CPU-bound rendering offloaded to a thread pool (E07-R-005 mitigation): the synchronous `render_pdf` / `render_docx` call MUST be wrapped in `await asyncio.get_running_loop().run_in_executor(None, render_fn, doc)` so that the uvicorn async event loop is not blocked during file generation.

9. **AC9** — Integration tests at `services/client-api/tests/api/test_proposal_export.py` covering: PDF valid file (E07-P0-010); DOCX valid file (E07-P1-019); company name and section headers present in export (E07-P1-020); invalid format 422 (E07-P2-008); empty proposal graceful export (E07-P2-009); cross-company 404 (E07-P0-005 partial). All prior tests (Stories 7.1–7.9) must continue to pass.

## Tasks / Subtasks

- [ ] Task 1: Add export size-limit settings to `config.py` (AC6)
  - [ ] 1.1 Open `services/client-api/src/client_api/config.py`
  - [ ] 1.2 Locate `ClientApiSettings` class. Add two new fields after `max_concurrent_sse_streams`:
    ```python
    export_max_sections: int = Field(
        default=50,
        description=(
            "Maximum number of sections allowed in a single export request. "
            "Requests exceeding this limit receive HTTP 400. "
            "Set via EXPORT_MAX_SECTIONS env var."
        ),
    )
    export_max_content_bytes: int = Field(
        default=524288,  # 512 KB
        description=(
            "Maximum total content size (bytes) allowed in a single export. "
            "Sum of all section body lengths. Returns HTTP 400 if exceeded. "
            "Set via EXPORT_MAX_CONTENT_BYTES env var."
        ),
    )
    ```
  - [ ] 1.3 `get_settings()` is `@lru_cache` — new fields are picked up automatically from env vars.

- [ ] Task 2: Add `pypdf` to dev dependencies (AC9 — needed for E07-P0-010 test assertion)
  - [ ] 2.1 Open `services/client-api/pyproject.toml`
  - [ ] 2.2 Under `[project.optional-dependencies]` → `dev = [...]`, add `"pypdf>=4.0"` after `"respx>=0.21"`.
  - [ ] 2.3 Run `pip install -e ".[dev]"` inside the client-api venv to make `pypdf` available during test execution.

- [ ] Task 3: Create Pydantic schema at `services/client-api/src/client_api/schemas/proposal_export.py` (AC1)
  - [ ] 3.1 Create file:
    ```python
    from __future__ import annotations

    from typing import Literal

    from pydantic import BaseModel


    class ExportRequest(BaseModel):
        """Request body for POST /proposals/{id}/export (AC1)."""

        format: Literal["pdf", "docx"]
    ```
  - [ ] 3.2 No response model needed — endpoint returns a raw `Response` with file bytes.

- [ ] Task 4: Create export service at `services/client-api/src/client_api/services/export_service.py` (AC2–AC8)
  - [ ] 4.1 Create file — header:
    ```python
    from __future__ import annotations

    import asyncio
    from datetime import UTC, datetime
    from uuid import UUID

    import structlog
    from fastapi import HTTPException
    from sqlalchemy import select
    from sqlalchemy.ext.asyncio import AsyncSession

    from client_api.config import get_settings
    from client_api.core.security import CurrentUser
    from client_api.models.company import Company
    from client_api.models.proposal import Proposal
    from client_api.models.proposal_version import ProposalVersion
    from client_api.utils.document_export import (
        HeaderSection,
        ReportDocument,
        ReportMetadata,
        TextSection,
        render_docx,
        render_pdf,
    )

    log = structlog.get_logger()
    ```
  - [ ] 4.2 Implement `_load_proposal_with_content(proposal_id, current_user, session)`:
    ```python
    async def _load_proposal_with_content(
        proposal_id: UUID,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> tuple[Proposal, list[dict]]:
        """Fetch proposal and enforce RLS; return (proposal, sections_list).

        Raises HTTP 404 if proposal not found or belongs to a different company.
        Returns empty sections list if proposal has no current version.
        """
        stmt = select(Proposal).where(Proposal.id == proposal_id)
        result = await session.execute(stmt)
        proposal = result.scalar_one_or_none()

        if proposal is None:
            raise HTTPException(status_code=404, detail="Proposal not found")

        if proposal.company_id != current_user.company_id:
            raise HTTPException(status_code=404, detail="Proposal not found")

        sections: list[dict] = []
        if proposal.current_version_id is not None:
            ver_stmt = select(ProposalVersion).where(
                ProposalVersion.id == proposal.current_version_id
            )
            ver_result = await session.execute(ver_stmt)
            version = ver_result.scalar_one_or_none()
            if version is not None:
                sections = version.content.get("sections", [])

        return proposal, sections
    ```
  - [ ] 4.3 Implement `_load_company(company_id, session)`:
    ```python
    async def _load_company(
        company_id: UUID,
        session: AsyncSession,
    ) -> Company | None:
        """Load Company row for branding; returns None if not found."""
        stmt = select(Company).where(Company.id == company_id)
        result = await session.execute(stmt)
        return result.scalar_one_or_none()
    ```
  - [ ] 4.4 Implement `build_proposal_report_document(proposal, company, sections)` — **pure synchronous function** (safe to run in executor):
    ```python
    def build_proposal_report_document(
        proposal: Proposal,
        company: Company | None,
        sections: list[dict],
    ) -> ReportDocument:
        """Build a ReportDocument from proposal data for rendering.

        Structure:
          - Metadata: proposal title, company name, generated_at timestamp
          - TOC section: HeaderSection(level=1, "Table of Contents") +
                         TextSection listing all section titles
          - Per section: HeaderSection(level=1, title) + TextSection(body)
          - Empty proposal: single TextSection("No content available")

        This function is synchronous and safe to run in run_in_executor.
        """
        company_name = company.name if company else "EU Solicit"
        proposal_title = proposal.title or "Untitled Proposal"

        meta = ReportMetadata(
            title=proposal_title,
            company_name=company_name,
            generated_at=datetime.now(UTC),
        )

        doc_sections: list = []

        if not sections:
            # AC7: empty proposal — graceful fallback
            doc_sections.append(TextSection(body="No content available."))
        else:
            # Table of Contents (AC2)
            toc_lines = "\n".join(
                f"{i + 1}. {s.get('title', f'Section {i + 1}')}"
                for i, s in enumerate(sections)
            )
            doc_sections.append(HeaderSection(level=1, text="Table of Contents"))
            doc_sections.append(TextSection(body=toc_lines))

            # Section content
            for section in sections:
                title = section.get("title", "Untitled Section")
                body = section.get("body", "")
                doc_sections.append(HeaderSection(level=1, text=title))
                doc_sections.append(TextSection(body=body if body else "(empty)"))

        return ReportDocument(metadata=meta, sections=doc_sections)
    ```
  - [ ] 4.5 Implement `async generate_export(proposal_id, format, current_user, session) -> bytes`:
    ```python
    async def generate_export(
        proposal_id: UUID,
        export_format: str,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> bytes:
        """Fetch proposal data, validate size limits, build ReportDocument,
        render PDF or DOCX in a thread pool executor (E07-R-005 mitigation).

        Args:
            proposal_id: UUID of the proposal to export.
            export_format: "pdf" or "docx" (validated by ExportRequest schema).
            current_user: Authenticated user with company_id and role.
            session: Async DB session for data fetching.

        Returns:
            File bytes (PDF or DOCX).

        Raises:
            HTTP 404: Proposal not found or cross-company.
            HTTP 400: Proposal exceeds export_max_sections or export_max_content_bytes limits.
        """
        settings = get_settings()

        # Fetch proposal + content (enforces RLS — raises 404 if cross-company)
        proposal, sections = await _load_proposal_with_content(
            proposal_id, current_user, session
        )

        # Fetch company for branding
        company = await _load_company(current_user.company_id, session)

        # AC6: size limit validation (E07-R-010 mitigation)
        if len(sections) > settings.export_max_sections:
            raise HTTPException(
                status_code=400,
                detail={
                    "error": "proposal_too_large",
                    "detail": (
                        f"Proposal has {len(sections)} sections, "
                        f"maximum allowed is {settings.export_max_sections}."
                    ),
                },
            )

        total_content_bytes = sum(len(s.get("body", "").encode("utf-8")) for s in sections)
        if total_content_bytes > settings.export_max_content_bytes:
            raise HTTPException(
                status_code=400,
                detail={
                    "error": "proposal_too_large",
                    "detail": (
                        f"Total proposal content is {total_content_bytes} bytes, "
                        f"maximum allowed is {settings.export_max_content_bytes}."
                    ),
                },
            )

        # Build ReportDocument (pure data transform — no DB I/O)
        doc = build_proposal_report_document(proposal, company, sections)

        # AC8: offload CPU-bound rendering to thread pool (E07-R-005 mitigation)
        render_fn = render_pdf if export_format == "pdf" else render_docx
        loop = asyncio.get_running_loop()
        file_bytes: bytes = await loop.run_in_executor(None, render_fn, doc)

        log.info(
            "proposal.export.generated",
            proposal_id=str(proposal_id),
            format=export_format,
            size_bytes=len(file_bytes),
            section_count=len(sections),
        )
        return file_bytes
    ```

- [ ] Task 5: Add export endpoint to proposals router (AC1–AC8)
  - [ ] 5.1 Open `services/client-api/src/client_api/api/v1/proposals.py`
  - [ ] 5.2 Add import at the top of the file (after existing imports):
    ```python
    from client_api.schemas.proposal_export import ExportRequest
    from client_api.services import export_service
    ```
  - [ ] 5.3 Add the following `# Story 7.10` section at the END of the proposals router (after the `/{proposal_id}/win-themes` GET endpoint):
    ```python
    # ---------------------------------------------------------------------------
    # Story 7.10 — Document Export (PDF & DOCX)
    # ---------------------------------------------------------------------------


    @router.post(
        "/{proposal_id}/export",
        status_code=200,
        responses={
            200: {
                "content": {
                    "application/pdf": {},
                    "application/vnd.openxmlformats-officedocument.wordprocessingml.document": {},
                },
                "description": "Generated document file download.",
            },
            400: {"description": "Proposal too large for export"},
            404: {"description": "Proposal not found or cross-company"},
            422: {"description": "Invalid format parameter"},
        },
    )
    async def export_proposal(
        proposal_id: UUID,
        body: ExportRequest,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
    ) -> Response:
        """Generate and download a proposal as PDF or DOCX (AC1–AC8).

        Requires bid_manager role.
        company_id always from JWT (E07-R-003 mitigation).

        Returns HTTP 200 with the generated file bytes and appropriate
        Content-Type / Content-Disposition headers.
        Returns HTTP 404 for cross-company or non-existent proposal.
        Returns HTTP 400 if the proposal exceeds EXPORT_MAX_SECTIONS or
        EXPORT_MAX_CONTENT_BYTES limits.
        Returns HTTP 422 for invalid format parameter (handled by Pydantic
        ExportRequest validation before this handler is invoked).

        CPU-bound rendering is offloaded to a thread pool executor to avoid
        blocking the uvicorn async event loop (E07-R-005 mitigation).
        """
        file_bytes = await export_service.generate_export(
            proposal_id, body.format, current_user, session
        )

        if body.format == "pdf":
            content_type = "application/pdf"
            filename = f"proposal-{proposal_id}.pdf"
        else:
            content_type = (
                "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
            )
            filename = f"proposal-{proposal_id}.docx"

        return Response(
            content=file_bytes,
            media_type=content_type,
            headers={"Content-Disposition": f'attachment; filename="{filename}"'},
        )
    ```
  - [ ] 5.4 Verify `Response` is already imported from `fastapi` — it IS (used by `delete_content_block` pattern). No additional import needed.

- [ ] Task 6: Integration tests at `services/client-api/tests/api/test_proposal_export.py` (AC9)
  - [ ] 6.1 File header and imports:
    ```python
    """Integration tests: Story 7.10 — Proposal Export API (PDF & DOCX).

    Coverage (from test-design-epic-07.md):
      E07-P0-010  — PDF export produces valid parsable file with section headers + TOC
      E07-P1-019  — DOCX export produces valid parsable file
      E07-P1-020  — Export contains company name and section titles (branding)
      E07-P2-008  — Export rejects invalid format parameter (422)
      E07-P2-009  — Export handles empty proposal gracefully (200, valid file)
      E07-P0-005  — Cross-company proposal returns 404 (RLS enforcement)
    """
    from __future__ import annotations

    import io
    import uuid
    from collections.abc import AsyncGenerator

    import httpx
    import pytest
    import pytest_asyncio
    from sqlalchemy import text
    from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker
    ```
  - [ ] 6.2 Use the same fixture pattern as `test_proposals.py` — register user+company, verify email via SQL, login for JWT:
    ```python
    @pytest_asyncio.fixture
    async def export_client_and_session(
        client_api_session_factory: async_sessionmaker[AsyncSession],
        test_redis_client,
    ) -> AsyncGenerator[tuple[httpx.AsyncClient, AsyncSession, str, str], None]:
        """Yield (client, session, access_token, company_id_str).

        Same shared-session pattern as test_proposals.py fixture.
        """
        from client_api.main import app as fastapi_app
        from client_api.dependencies import get_db_session, get_email_service_dep, get_redis_client
        from client_api.services.email_service import StubEmailService
        from httpx import ASGITransport

        uid = uuid.uuid4().hex[:8]
        email = f"export-{uid}@eusolicit-test.com"
        password = "SecurePass1"
        company_name = f"Export Co {uid} Ltd"

        session = client_api_session_factory()
        stub_email = StubEmailService()

        async def override_db():
            yield session

        async def override_email():
            return stub_email

        async def override_redis():
            return test_redis_client

        fastapi_app.dependency_overrides[get_db_session] = override_db
        fastapi_app.dependency_overrides[get_email_service_dep] = override_email
        fastapi_app.dependency_overrides[get_redis_client] = override_redis

        try:
            async with httpx.AsyncClient(
                transport=ASGITransport(app=fastapi_app),
                base_url="http://test",
            ) as client:
                reg = await client.post(
                    "/api/v1/auth/register",
                    json={
                        "email": email,
                        "password": password,
                        "company_name": company_name,
                        "role": "bid_manager",
                    },
                )
                assert reg.status_code == 201, reg.text

                await session.execute(
                    text("UPDATE client.users SET email_verified = TRUE WHERE email = :email"),
                    {"email": email},
                )
                await session.flush()

                login = await client.post(
                    "/api/v1/auth/login",
                    json={"email": email, "password": password},
                )
                assert login.status_code == 200, login.text
                access_token = login.json()["access_token"]
                company_id_str = reg.json()["company_id"]

                yield client, session, access_token, company_id_str
        finally:
            await session.rollback()
            await session.close()
            fastapi_app.dependency_overrides.clear()


    @pytest_asyncio.fixture
    async def another_company_token(
        client_api_session_factory: async_sessionmaker[AsyncSession],
        test_redis_client,
    ) -> AsyncGenerator[str, None]:
        """Yield Company B's access_token for RLS cross-company tests."""
        from client_api.main import app as fastapi_app
        from client_api.dependencies import get_db_session, get_email_service_dep, get_redis_client
        from client_api.services.email_service import StubEmailService
        from httpx import ASGITransport

        uid = uuid.uuid4().hex[:8]
        email = f"export-b-{uid}@eusolicit-test.com"
        password = "SecurePass1"
        company_name = f"Company B {uid} Ltd"

        session = client_api_session_factory()
        stub_email = StubEmailService()

        async def override_db():
            yield session

        async def override_email():
            return stub_email

        async def override_redis():
            return test_redis_client

        fastapi_app.dependency_overrides[get_db_session] = override_db
        fastapi_app.dependency_overrides[get_email_service_dep] = override_email
        fastapi_app.dependency_overrides[get_redis_client] = override_redis

        try:
            async with httpx.AsyncClient(
                transport=ASGITransport(app=fastapi_app),
                base_url="http://test",
            ) as client:
                reg = await client.post(
                    "/api/v1/auth/register",
                    json={
                        "email": email,
                        "password": password,
                        "company_name": company_name,
                        "role": "bid_manager",
                    },
                )
                assert reg.status_code == 201, reg.text

                await session.execute(
                    text("UPDATE client.users SET email_verified = TRUE WHERE email = :email"),
                    {"email": email},
                )
                await session.flush()

                login = await client.post(
                    "/api/v1/auth/login",
                    json={"email": email, "password": password},
                )
                assert login.status_code == 200, login.text
                yield login.json()["access_token"]
        finally:
            await session.rollback()
            await session.close()
            fastapi_app.dependency_overrides.clear()
    ```
  - [ ] 6.3 `TestProposalExportPDF` (E07-P0-010 — P0 priority!):
    - `test_pdf_export_returns_valid_pdf_bytes`:
      1. Create a proposal via `POST /api/v1/proposals` (requires an opportunity or just title)
      2. Set current version content with 2 sections: `[{"key": "intro", "title": "Introduction", "body": "We are a leading firm."}, {"key": "approach", "title": "Technical Approach", "body": "Our approach is..."}]` via `PUT /api/v1/proposals/{id}/content` (content hash trick: first PATCH then PUT, or seed directly via session)
      3. `POST /api/v1/proposals/{id}/export` with `{"format": "pdf"}` → HTTP 200
      4. Verify `Content-Type: application/pdf`
      5. Verify `Content-Disposition` header contains `attachment` and `.pdf`
      6. Parse response bytes with `pypdf.PdfReader(io.BytesIO(response.content))`
      7. Verify `len(reader.pages) >= 1`
      8. Extract text from all pages; verify `"Introduction"` and `"Technical Approach"` present
      9. Verify `"Table of Contents"` text present in extracted PDF text (TOC page)
    - `test_pdf_export_magic_bytes`:
      - Verify `response.content[:4] == b"%PDF"` (PDF magic bytes)
  - [ ] 6.4 `TestProposalExportDOCX` (E07-P1-019):
    - `test_docx_export_returns_valid_docx_bytes`:
      1. Same setup as above (proposal with 2 sections)
      2. `POST /api/v1/proposals/{id}/export` with `{"format": "docx"}` → HTTP 200
      3. Verify `Content-Type` header contains `wordprocessingml`
      4. Verify `Content-Disposition` header contains `attachment` and `.docx`
      5. Parse with `from docx import Document; doc = Document(io.BytesIO(response.content))`
      6. Verify document has paragraphs
      7. Extract all heading text; verify `"Introduction"` and `"Technical Approach"` appear as headings
    - `test_docx_magic_bytes`:
      - Verify `response.content[:2] == b"PK"` (ZIP/DOCX magic bytes)
  - [ ] 6.5 `TestProposalExportBranding` (E07-P1-020):
    - `test_pdf_contains_company_name`:
      1. Create proposal; export as PDF
      2. Extract all text from PDF via pypdf
      3. Verify `company_name` (from fixture registration) appears in the PDF text
    - `test_docx_contains_company_name`:
      1. Create proposal; export as DOCX
      2. Parse with python-docx; collect all paragraph text
      3. Verify `company_name` appears in the document
  - [ ] 6.6 `TestProposalExportValidation` (E07-P2-008):
    - `test_invalid_format_returns_422`:
      1. Create proposal; `POST /export` with `{"format": "xml"}` → HTTP 422
      2. Verify error body contains validation detail about `format` field
    - `test_missing_format_returns_422`:
      1. `POST /export` with `{}` (no format field) → HTTP 422
  - [ ] 6.7 `TestProposalExportEmptyProposal` (E07-P2-009):
    - `test_empty_proposal_exports_gracefully`:
      1. Create proposal (POST /proposals) — has a current_version_id with empty sections by default
      2. `POST /export` with `{"format": "pdf"}` → HTTP 200 (NOT 500)
      3. Verify response.content is valid PDF bytes (magic bytes `b"%PDF"`)
    - `test_empty_proposal_docx_exports_gracefully`:
      1. Same but `{"format": "docx"}` → HTTP 200, valid DOCX
  - [ ] 6.8 `TestProposalExportRLS` (E07-P0-005 partial — export endpoint):
    - `test_cross_company_export_returns_404`:
      1. Company A creates a proposal
      2. Company B JWT: `POST /proposals/{company_a_proposal_id}/export` with `{"format": "pdf"}` → HTTP 404
      3. Verify response is 404, not 403 (no ID confirmation)
    - `test_nonexistent_proposal_returns_404`:
      1. `POST /proposals/{random_uuid}/export` → HTTP 404
  - [ ] 6.9 Helper: set proposal content via `PUT /api/v1/proposals/{id}/content`:
    ```python
    async def _set_proposal_content(client, token, proposal_id, sections):
        """Set proposal content directly via PUT endpoint.
        Uses 'all-zeros' hash that the PUT endpoint accepts for first-write
        OR seed via proposal_service.compute_content_hash.
        """
        # First GET the current content to obtain the current hash
        get_resp = await client.get(
            f"/api/v1/proposals/{proposal_id}",
            headers={"Authorization": f"Bearer {token}"},
        )
        assert get_resp.status_code == 200
        # Compute hash of current content (empty sections)
        import hashlib, json
        current_content = get_resp.json().get("current_version_content", {"sections": []})
        current_hash = hashlib.md5(json.dumps(current_content, sort_keys=True).encode()).hexdigest()

        put_resp = await client.put(
            f"/api/v1/proposals/{proposal_id}/content",
            json={"sections": sections, "content_hash": current_hash},
            headers={"Authorization": f"Bearer {token}"},
        )
        assert put_resp.status_code in (200, 409), f"PUT content failed: {put_resp.text}"
        return put_resp
    ```
    **CRITICAL NOTE:** The exact hash computation must match what `proposal_service._compute_content_hash` uses. Check that function before implementing the helper — if it uses `json.dumps(content, sort_keys=True)`, use the same. Do NOT hardcode a hash format.

## Dev Notes

### Context: Built on Stories 7.1–7.9

**No new migration needed.** This story adds no new DB columns. All data (proposal, version, company) is fetched from existing tables using already-established query patterns.

**Test baseline: 351+ tests** across Stories 7.1–7.9 (including extended ATDD tests added by TEA). All must continue to pass when this story is implemented.

**Current endpoint count: 31** (proposals router: 23 endpoints from 7.2–7.8 + 8 content_blocks endpoints from 7.9, registered on a separate router). This story adds 1 new endpoint to the proposals router.

### CRITICAL: Executor Pattern for CPU-Bound Rendering (E07-R-005)

`render_pdf` and `render_docx` are **synchronous** functions that use reportlab/python-docx. Calling them directly inside an `async` FastAPI handler blocks the uvicorn event loop for all concurrent requests during rendering.

**Always wrap in executor:**
```python
loop = asyncio.get_running_loop()
file_bytes = await loop.run_in_executor(None, render_fn, doc)
```

`None` uses the default `ThreadPoolExecutor`. For truly CPU-bound work a `ProcessPoolExecutor` is cleaner, but for the export sizes expected (50 sections max, 512 KB) a thread pool avoids event loop blocking without the `multiprocessing` complexity. The test design (E07-R-005) specifically calls out this mitigation — it MUST be implemented, not skipped.

### Import Chain for Rendering (AC4 of Story 12.9)

The rendering functions MUST be imported via `client_api.utils.document_export`, not directly from `eusolicit_common.document_generation`, to preserve the import chain established in Story 12.9 AC4:

```python
# CORRECT — preserves Story 12.9 AC4 contract:
from client_api.utils.document_export import render_pdf, render_docx, ReportDocument, ...

# WRONG — bypasses the stub module:
from eusolicit_common.document_generation import render_pdf
```

The `document_export.py` stub at `services/client-api/src/client_api/utils/document_export.py` already re-exports all needed symbols from `eusolicit_common.document_generation`. It does NOT need modification.

### ReportDocument Structure for Proposals

The shared `ReportDocument` model (from Story 12.9) was designed for analytics reports but maps cleanly to proposals:

```
ReportMetadata:
  title       = proposal.title or "Untitled Proposal"
  company_name = company.name   (branding — company name on title page)
  generated_at = datetime.now(UTC)
  subtitle    = None  (not needed for proposals)
  date_from   = None  (proposals don't have a date range)

Sections (in order):
  1. HeaderSection(level=1, text="Table of Contents")
  2. TextSection(body="1. Introduction\n2. Technical Approach\n...")
  3. HeaderSection(level=1, text=section["title"])
  4. TextSection(body=section["body"])
  5. ... (repeat 3-4 for each section)
```

**The PDF renderer** (`render_pdf`) automatically adds:
- Title page: `company_name` as Heading1, `title` as Heading2
- Page numbers at bottom of each page (via `_page_number_footer` callback)

**You do NOT need to implement page numbers or title page** — the shared renderer handles those.

**The TOC** is implemented as a "Table of Contents" header + numbered text list. This is intentionally simple — it produces a parsable TOC page that satisfies the E07-P0-010 test assertion (`"Table of Contents"` present in PDF text layer).

### Company Branding — Available Fields

The `Company` model (`models/company.py`) does NOT have `logo_url` or `brand_color` fields. Available branding fields:
- `name` (required, String(255)) — used as Heading1 on the title page by the renderer
- `description` (optional, Text) — not included in export (too long for header use)
- `registration_number` (optional) — not included

The "branding" in this story consists of the **company name** rendered as a Heading1 on the PDF/DOCX title page by the shared renderer's `meta.company_name`. Test E07-P1-020 verifies company name is present in the exported document. There is no logo or color brand requirement implementable from the current Company schema.

### RLS Pattern — Same as All Prior E07 Stories

```python
# Always fetch proposal first, then check company ownership:
stmt = select(Proposal).where(Proposal.id == proposal_id)
result = await session.execute(stmt)
proposal = result.scalar_one_or_none()

if proposal is None or proposal.company_id != current_user.company_id:
    raise HTTPException(status_code=404, detail="Proposal not found")
```

Return **404** (not 403) — avoids confirming the proposal UUID exists to a cross-company attacker (E07-R-003 mitigation, E07-P0-005).

### Company Query Pattern (Established in proposal_service.py lines 944, 1319)

The `Company` model is already imported via a local import in `proposal_service.py` with `# noqa: PLC0415`. For the export service, import at module level is acceptable since there's no circular import risk (export_service.py does not depend on proposal_service.py).

```python
# Top-level import is fine in export_service.py:
from client_api.models.company import Company
```

### Export Response — Use `Response` Not `StreamingResponse`

The export bytes are fully buffered in memory (in-memory `io.BytesIO` in both renderers). Return them as a single `Response` with `content=file_bytes` — this is simpler and correct for files up to the 512 KB limit. `StreamingResponse` is appropriate for streaming generation (e.g., SSE), not for fully-buffered files.

```python
return Response(
    content=file_bytes,
    media_type=content_type,
    headers={"Content-Disposition": f'attachment; filename="{filename}"'},
)
```

### Test Helpers — Setting Proposal Content

The `PUT /api/v1/proposals/{id}/content` endpoint requires the current content hash for optimistic locking (E07-R-002 mitigation). For tests:

1. **Recommended approach**: Seed the proposal version content directly via `session.execute()` or by using an `UPDATE` on the `proposal_versions` table. This avoids the hash computation entirely.

2. **Alternative**: Use `proposal_service._compute_content_hash` (if exported) — but check the actual implementation in `proposal_service.py` before using it.

3. **Verify `_compute_content_hash` implementation**:
   ```bash
   grep -n "_compute_content_hash\|compute_content_hash" \
     services/client-api/src/client_api/services/proposal_service.py | head -10
   ```
   The hash may be `hashlib.md5(json.dumps(content, sort_keys=True).encode()).hexdigest()` or a SHA hash — verify before implementing the test helper.

4. **Simplest test approach**: Create the proposal (which initialises an empty version), then directly update the version content via SQL in the test session:
   ```python
   await session.execute(
       text("""
           UPDATE client.proposal_versions
           SET content = :content
           WHERE id = (
               SELECT current_version_id FROM client.proposals WHERE id = :proposal_id
           )
       """),
       {
           "content": json.dumps({"sections": sections}),
           "proposal_id": str(proposal_id),
       },
   )
   await session.flush()
   ```

### pypdf API for PDF Validation

`pypdf>=4.0` (new package name; successor to `PyPDF2`) for PDF structural validation in E07-P0-010:

```python
import pypdf
import io

reader = pypdf.PdfReader(io.BytesIO(response.content))
assert len(reader.pages) >= 1

# Extract text from all pages:
all_text = ""
for page in reader.pages:
    all_text += page.extract_text() or ""

assert "Introduction" in all_text
assert "Table of Contents" in all_text
```

### python-docx API for DOCX Validation

`python-docx` is already in production dependencies (not just dev):

```python
import io
from docx import Document

doc = Document(io.BytesIO(response.content))
# Get all paragraph text:
all_text = "\n".join(p.text for p in doc.paragraphs)
assert "Introduction" in all_text
```

### No New Migration — Verify

```bash
ls services/client-api/alembic/versions/
```

Existing migrations: 010–024. This story adds NO new migration. The export is a pure read + render operation.

### CRITICAL Architecture Constraints (Inherited from Stories 7.1–7.9)

1. **`from __future__ import annotations`** at top of every new file
2. **`structlog` for all logging**: `log = structlog.get_logger()` (never `logging`)
3. **`session.flush()` — never `session.commit()`** for route-scoped sessions
4. **`company_id` exclusively from JWT** — never accept from request body, path params, or query params
5. **Return 404 (not 403) for cross-company** — prevents UUID enumeration (E07-R-003, E07-P0-005)
6. **`require_role("bid_manager")`** for export endpoint (write/generate operations require bid_manager)
7. **`asyncio.run_in_executor`** for CPU-bound rendering — MUST NOT call render_pdf/render_docx synchronously in async handler (E07-R-005)
8. **Import via `client_api.utils.document_export`** — not directly from `eusolicit_common.document_generation` (Story 12.9 AC4 contract)

### Existing Infrastructure — DO NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/src/client_api/utils/document_export.py` | Stub: re-exports render_pdf, render_docx etc. from eusolicit_common | **DO NOT MODIFY** — import from it |
| `packages/eusolicit-common/src/eusolicit_common/document_generation/` | render_pdf, render_docx, ReportDocument, all section models | **DO NOT MODIFY** — shared with E12 |
| `services/client-api/src/client_api/services/proposal_service.py` | All proposal/version/agent service functions | **NO CHANGES** |
| `services/client-api/src/client_api/api/v1/proposals.py` | All 31 proposal endpoints (7.2–7.8) | **APPEND Story 7.10 section only** |
| `services/client-api/alembic/versions/024_proposal_pricing_win_themes.py` | Latest migration | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposals.py` | 41+ tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_content_blocks.py` | 27+ tests | **DO NOT MODIFY** |

### New Files to Create

| File | Purpose |
|------|---------|
| `services/client-api/src/client_api/schemas/proposal_export.py` | `ExportRequest` Pydantic model: `format: Literal["pdf", "docx"]` |
| `services/client-api/src/client_api/services/export_service.py` | Service: fetch + validate + build ReportDocument + async render |
| `services/client-api/tests/api/test_proposal_export.py` | Integration tests (~12 tests) |

### Files to Modify

| File | Change |
|------|--------|
| `services/client-api/src/client_api/config.py` | Add `export_max_sections` and `export_max_content_bytes` settings |
| `services/client-api/src/client_api/api/v1/proposals.py` | Append Story 7.10 export endpoint section |
| `services/client-api/pyproject.toml` | Add `pypdf>=4.0` to dev dependencies |

### New API Endpoint

```
POST   /api/v1/proposals/{proposal_id}/export    → Generate PDF or DOCX download (bid_manager)
```

### Running Tests

```bash
cd eusolicit-app/services/client-api

# New story tests only:
pytest tests/api/test_proposal_export.py -v

# Full regression suite (all E07 tests):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py \
       tests/api/test_proposal_compliance_risk_scoring.py \
       tests/api/test_proposal_pricing_win_themes.py \
       tests/api/test_content_blocks.py \
       tests/api/test_proposal_export.py -v
```

### Test Coverage Map (from Epic 7 Test Design)

| Test Design ID | Priority | Scenario | Implementation |
|----------------|----------|----------|----------------|
| **E07-P0-010** | P0 | PDF export produces valid parsable file with section headers and TOC | `TestProposalExportPDF.test_pdf_export_returns_valid_pdf_bytes` + `test_pdf_magic_bytes` |
| **E07-P1-019** | P1 | DOCX export produces valid parsable file | `TestProposalExportDOCX.test_docx_export_returns_valid_docx_bytes` + `test_docx_magic_bytes` |
| **E07-P1-020** | P1 | Export includes company branding and section headers | `TestProposalExportBranding.test_pdf_contains_company_name` + `test_docx_contains_company_name` |
| **E07-P2-008** | P2 | Export rejects invalid format parameter (422) | `TestProposalExportValidation.test_invalid_format_returns_422` + `test_missing_format_returns_422` |
| **E07-P2-009** | P2 | Export handles empty proposal gracefully | `TestProposalExportEmptyProposal.test_empty_proposal_exports_gracefully` (PDF + DOCX) |
| **E07-P0-005** | P0 | Cross-company 404 (export endpoint) | `TestProposalExportRLS.test_cross_company_export_returns_404` |
| **E07-R-003** | Risk | RLS bypass — 404 not 403 | `TestProposalExportRLS.test_*_returns_404` (not 403) |
| **E07-R-005** | Risk | CPU-bound export blocks event loop | `run_in_executor` in `export_service.generate_export` (code-level mitigation; no separate test) |
| **E07-R-010** | Risk | Export size denial-of-service | `export_max_sections` + `export_max_content_bytes` settings; validated in `generate_export` |

### References

- [Source: eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md#S07.10] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-010] — PDF export P0 test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P1-019] — DOCX export P1 test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P1-020] — branding test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P2-008] — invalid format 422
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P2-009] — empty proposal graceful export
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-005] — CPU-bound rendering block; mitigation: run_in_executor
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-010] — export size DOS; mitigation: EXPORT_MAX_SECTIONS + EXPORT_MAX_CONTENT_BYTES
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-003] — RLS bypass; 404 not 403
- [Source: eusolicit-docs/implementation-artifacts/12-9-report-generation-engine-pdf-docx.md#AC3] — eusolicit_common.document_generation package; render_pdf, render_docx, ReportDocument
- [Source: eusolicit-docs/implementation-artifacts/12-9-report-generation-engine-pdf-docx.md#AC4] — document_export.py stub import chain contract
- [Source: eusolicit-docs/implementation-artifacts/7-9-content-blocks-crud-search-api.md] — architecture constraints: from __future__, structlog, session.flush, company_id from JWT, 404 not 403
- [Source: eusolicit-app/services/client-api/src/client_api/utils/document_export.py] — stub module re-exporting from eusolicit_common; IMPORT FROM HERE
- [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/document_generation/pdf_renderer.py] — render_pdf implementation; page numbers + title page handled by renderer
- [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/document_generation/models.py] — ReportDocument, ReportMetadata, HeaderSection, TextSection
- [Source: eusolicit-app/services/client-api/src/client_api/models/company.py] — Company model (name, description, no logo_url/brand_color)
- [Source: eusolicit-app/services/client-api/src/client_api/models/proposal.py] — Proposal model
- [Source: eusolicit-app/services/client-api/src/client_api/models/proposal_version.py] — ProposalVersion model; content JSONB with sections array
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py] — existing proposals router; APPEND export endpoint at end
- [Source: eusolicit-app/services/client-api/src/client_api/config.py] — ClientApiSettings; ADD export_max_sections + export_max_content_bytes
- [Source: eusolicit-app/services/client-api/pyproject.toml] — ADD pypdf>=4.0 to dev dependencies
