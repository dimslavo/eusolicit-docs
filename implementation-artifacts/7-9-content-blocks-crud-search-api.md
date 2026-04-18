# Story 7.9: Content Blocks CRUD & Search API

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager**,
I want a reusable content library API where I can create, search, and manage pre-approved content blocks,
so that I can efficiently compose high-quality proposals by inserting relevant pre-approved content at cursor position in the Tiptap editor.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/content-blocks` creates a new content block with `title`, `category`, `body`, and `tags` (text[]). `company_id` derived exclusively from JWT — never from request body. `version` initialised to `1`. `approved_at` and `approved_by` default to `null`. Returns HTTP 201 with full block object (`id`, `company_id`, `title`, `category`, `body`, `tags`, `version`, `approved_at`, `approved_by`, `created_at`, `updated_at`). Title and body are required; missing either returns HTTP 422.

2. **AC2** — `GET /api/v1/content-blocks` lists all company blocks. Supports: `limit` (default 20, max 100) and `offset` (default 0) for pagination; `category` (exact match filter); `tags` (comma-separated; blocks must contain ALL specified tags using PostgreSQL `@>` array containment). Default sort `created_at DESC`. Returns empty list (not 404) when no blocks match. All results scoped to authenticated company via RLS.

3. **AC3** — `GET /api/v1/content-blocks/search?q=<query>` performs PostgreSQL full-text search across `title` and `body` using the `search_vector` tsvector column with `plainto_tsquery('english', q)`. Results sorted by `ts_rank` relevance descending. Returns empty list if no matches. Cross-company search returns zero results (never exposes other companies' block titles or IDs). **Route `/search` MUST be registered before `/{block_id}` in the router to prevent FastAPI path collision.**

4. **AC4** — `GET /api/v1/content-blocks/{block_id}` returns the full block object. Returns 404 if block does not exist or belongs to a different company. Never returns 403 — prevents UUID enumeration.

5. **AC5** — `PATCH /api/v1/content-blocks/{block_id}` accepts partial update of `title`, `category`, `body`, `tags`. On any update: increments `version` by 1; updates `updated_at`. **Application-level tsvector refresh (E07-R-009 mitigation):** if `title` or `body` is in the patch payload, recalculate `search_vector = to_tsvector('english', coalesce(new_title,'') || ' ' || coalesce(new_body,''))` and persist it explicitly in the service — do NOT rely solely on any DB trigger for freshness. Returns 404 for cross-company or non-existent block.

6. **AC6** — `POST /api/v1/content-blocks/{block_id}/approve` sets `approved_at = datetime.now(tz=UTC)` and `approved_by = current_user.user_id`. `POST /api/v1/content-blocks/{block_id}/unapprove` clears both (`approved_at = null`, `approved_by = null`). Both return HTTP 200 with the updated block. Return 404 for cross-company or non-existent block.

7. **AC7** — `DELETE /api/v1/content-blocks/{block_id}` hard-deletes the block (NOT soft-archive — unlike proposals which are soft-archived). Returns HTTP 204 No Content on success. Returns 404 for cross-company or non-existent block.

8. **AC8** — Integration tests at `services/client-api/tests/api/test_content_blocks.py` covering: full CRUD lifecycle (E07-P1-017); full-text search correctness and relevance ordering (E07-P1-018, E07-P3-003); tsvector freshness after PATCH body and title changes (E07-P2-006, E07-R-009); cross-company 404 for all 5 endpoint types (E07-P0-006); pagination, category, and tag filtering (E07-P2-007). All 184 existing tests (Stories 7.1–7.8) must continue to pass.

## Tasks / Subtasks

- [x] Task 1: Verify `ContentBlock` ORM model and `search_vector` tsvector in migration 020 (AC: 1–7)
  - [ ] 1.1 Open `services/client-api/alembic/versions/020_proposal_workspace_schema.py` — verify `content_blocks` table definition includes:
    - `search_vector TSVECTOR` column with GIN index
    - `version INTEGER NOT NULL DEFAULT 1`
    - `approved_at TIMESTAMPTZ NULL`, `approved_by UUID NULL`
    - `tags TEXT[] NOT NULL DEFAULT '{}'`
    - `company_id UUID NOT NULL` (FK or raw, with index)
  - [ ] 1.2 Check for a PostgreSQL trigger on `content_blocks` in migration 020 that updates `search_vector` on INSERT OR UPDATE of `title`/`body`. If trigger is MISSING: create `025_content_blocks_search_trigger.py` (revision `"025_content_blocks_search_trigger"` = 33 chars → OK if limit is 50, but VERIFY VARCHAR limit against alembic_version table before using)
  - [ ] 1.3 Locate the `ContentBlock` ORM model — most likely co-located in `services/client-api/src/client_api/models/proposal.py` (Story 7.1 created all three models together). If found: import it; DO NOT duplicate. If not found: create `services/client-api/src/client_api/models/content_block.py`.
  - [ ] 1.4 Verify the ORM model has `search_vector` mapped as `TSVECTOR`:
    ```python
    from sqlalchemy.dialects.postgresql import TSVECTOR, ARRAY
    import sqlalchemy as sa
    search_vector: Mapped[str | None] = mapped_column(TSVECTOR, nullable=True)
    tags: Mapped[list[str]] = mapped_column(ARRAY(sa.String), nullable=False, server_default="{}")
    ```
    If missing, add these mappings to the existing model.

- [x] Task 2: Pydantic schemas at `services/client-api/src/client_api/schemas/content_block.py` (AC: 1–7)
  - [ ] 2.1 Create `services/client-api/src/client_api/schemas/content_block.py`:
    ```python
    from __future__ import annotations
    from datetime import datetime
    from uuid import UUID
    from pydantic import BaseModel, Field


    class ContentBlockCreate(BaseModel):
        title: str = Field(..., min_length=1, max_length=500)
        category: str = Field(..., min_length=1, max_length=100)
        body: str = Field(..., min_length=1)
        tags: list[str] = Field(default_factory=list)


    class ContentBlockPatch(BaseModel):
        title: str | None = Field(None, min_length=1, max_length=500)
        category: str | None = Field(None, min_length=1, max_length=100)
        body: str | None = None
        tags: list[str] | None = None


    class ContentBlockResponse(BaseModel):
        id: UUID
        company_id: UUID
        title: str
        category: str
        body: str
        tags: list[str]
        version: int
        approved_at: datetime | None
        approved_by: UUID | None
        created_at: datetime
        updated_at: datetime

        class Config:
            from_attributes = True
    ```
  - [ ] 2.2 Note: `search_vector` is intentionally excluded from `ContentBlockResponse` — it is internal to PostgreSQL FTS and not exposed to API consumers.

- [x] Task 3: Service functions at `services/client-api/src/client_api/services/content_block_service.py` (AC: 1–7)
  - [ ] 3.1 Create new service file — header:
    ```python
    from __future__ import annotations
    from datetime import datetime, timezone
    from uuid import UUID
    import structlog
    from fastapi import HTTPException
    from sqlalchemy import cast, func, select
    from sqlalchemy.dialects.postgresql import ARRAY
    import sqlalchemy as sa
    from sqlalchemy.ext.asyncio import AsyncSession
    from client_api.auth import CurrentUser
    from client_api.models.proposal import ContentBlock  # adjust import path if model is elsewhere
    from client_api.schemas.content_block import ContentBlockCreate, ContentBlockPatch

    log = structlog.get_logger()
    ```
  - [ ] 3.2 Implement `create_content_block(data, current_user, session)`:
    ```python
    async def create_content_block(
        data: ContentBlockCreate,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> ContentBlock:
        block = ContentBlock(
            company_id=current_user.company_id,  # ALWAYS from JWT — never from request body
            title=data.title,
            category=data.category,
            body=data.body,
            tags=data.tags,
            version=1,
            search_vector=func.to_tsvector("english", data.title + " " + data.body),
        )
        session.add(block)
        await session.flush()
        await session.refresh(block)
        log.info("content_block.created", block_id=str(block.id), company_id=str(block.company_id))
        return block
    ```
  - [ ] 3.3 Implement `list_content_blocks(current_user, session, limit, offset, category, tags_csv)`:
    ```python
    async def list_content_blocks(
        current_user: CurrentUser,
        session: AsyncSession,
        limit: int = 20,
        offset: int = 0,
        category: str | None = None,
        tags_csv: str | None = None,
    ) -> list[ContentBlock]:
        stmt = (
            select(ContentBlock)
            .where(ContentBlock.company_id == current_user.company_id)
            .order_by(ContentBlock.created_at.desc())
            .offset(offset)
            .limit(min(limit, 100))
        )
        if category:
            stmt = stmt.where(ContentBlock.category == category)
        if tags_csv:
            tag_list = [t.strip() for t in tags_csv.split(",") if t.strip()]
            # PostgreSQL @> operator: left array contains ALL elements of right array
            stmt = stmt.where(
                ContentBlock.tags.op("@>")(cast(tag_list, ARRAY(sa.String)))
            )
        result = await session.execute(stmt)
        return list(result.scalars().all())
    ```
  - [ ] 3.4 Implement `search_content_blocks(query, current_user, session, limit, offset)`:
    ```python
    async def search_content_blocks(
        query: str,
        current_user: CurrentUser,
        session: AsyncSession,
        limit: int = 20,
        offset: int = 0,
    ) -> list[ContentBlock]:
        ts_query = func.plainto_tsquery("english", query)
        stmt = (
            select(ContentBlock)
            .where(
                ContentBlock.company_id == current_user.company_id,
                ContentBlock.search_vector.op("@@")(ts_query),
            )
            .order_by(func.ts_rank(ContentBlock.search_vector, ts_query).desc())
            .offset(offset)
            .limit(min(limit, 100))
        )
        result = await session.execute(stmt)
        return list(result.scalars().all())
    ```
    Note: `company_id` scoping BEFORE the `@@` filter ensures zero cross-company leakage (E07-P0-006).
  - [ ] 3.5 Implement helper `_get_block_or_404(block_id, current_user, session)`:
    ```python
    async def _get_block_or_404(
        block_id: UUID,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> ContentBlock:
        stmt = select(ContentBlock).where(
            ContentBlock.id == block_id,
            ContentBlock.company_id == current_user.company_id,  # RLS enforcement
        )
        block = (await session.execute(stmt)).scalar_one_or_none()
        if block is None:
            raise HTTPException(status_code=404, detail="Content block not found")
        return block
    ```
    Returns 404 for both "not found" AND "cross-company" cases — never 403 (UUID enumeration prevention).
  - [ ] 3.6 Implement `get_content_block(block_id, current_user, session)` — thin wrapper:
    ```python
    async def get_content_block(
        block_id: UUID,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> ContentBlock:
        return await _get_block_or_404(block_id, current_user, session)
    ```
  - [ ] 3.7 Implement `update_content_block(block_id, data, current_user, session)` — E07-R-009 mitigation:
    ```python
    async def update_content_block(
        block_id: UUID,
        data: ContentBlockPatch,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> ContentBlock:
        block = await _get_block_or_404(block_id, current_user, session)
        tsvector_dirty = False
        if data.title is not None:
            block.title = data.title
            tsvector_dirty = True
        if data.body is not None:
            block.body = data.body
            tsvector_dirty = True
        if data.category is not None:
            block.category = data.category
        if data.tags is not None:
            block.tags = data.tags
        block.version = block.version + 1
        block.updated_at = datetime.now(tz=timezone.utc)
        # E07-R-009 MITIGATION: application-level tsvector refresh
        # MUST update explicitly even if a DB trigger exists — ensures freshness
        # regardless of trigger presence in migration 020
        if tsvector_dirty:
            new_title = block.title  # already updated above
            new_body = block.body    # already updated above
            block.search_vector = func.to_tsvector(
                "english",
                func.coalesce(new_title, "") + " " + func.coalesce(new_body, ""),
            )
        await session.flush()
        await session.refresh(block)
        log.info("content_block.updated", block_id=str(block_id), version=block.version)
        return block
    ```
  - [ ] 3.8 Implement `approve_content_block(block_id, current_user, session)`:
    ```python
    async def approve_content_block(
        block_id: UUID,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> ContentBlock:
        block = await _get_block_or_404(block_id, current_user, session)
        block.approved_at = datetime.now(tz=timezone.utc)
        block.approved_by = current_user.user_id  # UUID of approving user
        await session.flush()
        await session.refresh(block)
        log.info("content_block.approved", block_id=str(block_id), approved_by=str(current_user.user_id))
        return block
    ```
  - [ ] 3.9 Implement `unapprove_content_block(block_id, current_user, session)`:
    ```python
    async def unapprove_content_block(
        block_id: UUID,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> ContentBlock:
        block = await _get_block_or_404(block_id, current_user, session)
        block.approved_at = None
        block.approved_by = None
        await session.flush()
        await session.refresh(block)
        log.info("content_block.unapproved", block_id=str(block_id))
        return block
    ```
  - [ ] 3.10 Implement `delete_content_block(block_id, current_user, session)`:
    ```python
    async def delete_content_block(
        block_id: UUID,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> None:
        block = await _get_block_or_404(block_id, current_user, session)
        await session.delete(block)
        await session.flush()
        log.info("content_block.deleted", block_id=str(block_id))
    ```
    Hard delete — no `status` field on content blocks; physically removed from DB.

- [x] Task 4: Router at `services/client-api/src/client_api/api/v1/content_blocks.py` (AC: 1–7)
  - [ ] 4.1 Create router file — header:
    ```python
    from __future__ import annotations
    from typing import Annotated
    from uuid import UUID
    from fastapi import APIRouter, Depends, Query, Response, status
    from sqlalchemy.ext.asyncio import AsyncSession
    from client_api.auth import CurrentUser
    from client_api.dependencies import get_current_user, get_db_session
    from client_api.schemas.content_block import ContentBlockCreate, ContentBlockPatch, ContentBlockResponse
    from client_api.services import content_block_service

    router = APIRouter()
    ```
  - [ ] 4.2 Define endpoints in THIS EXACT ORDER (search BEFORE block_id — see CRITICAL note below):
    ```python
    @router.post(
        "/",
        response_model=ContentBlockResponse,
        status_code=status.HTTP_201_CREATED,
    )
    async def create_content_block(
        data: ContentBlockCreate,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> ContentBlockResponse:
        block = await content_block_service.create_content_block(data, current_user, session)
        return ContentBlockResponse.model_validate(block)


    @router.get("/search", response_model=list[ContentBlockResponse])  # ← BEFORE /{block_id}
    async def search_content_blocks(
        q: Annotated[str, Query(min_length=1)],
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        limit: int = Query(default=20, ge=1, le=100),
        offset: int = Query(default=0, ge=0),
    ) -> list[ContentBlockResponse]:
        blocks = await content_block_service.search_content_blocks(q, current_user, session, limit, offset)
        return [ContentBlockResponse.model_validate(b) for b in blocks]


    @router.get("/", response_model=list[ContentBlockResponse])
    async def list_content_blocks(
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        limit: int = Query(default=20, ge=1, le=100),
        offset: int = Query(default=0, ge=0),
        category: str | None = Query(default=None),
        tags: str | None = Query(default=None, description="Comma-separated; block must contain ALL"),
    ) -> list[ContentBlockResponse]:
        blocks = await content_block_service.list_content_blocks(
            current_user, session, limit, offset, category, tags
        )
        return [ContentBlockResponse.model_validate(b) for b in blocks]


    @router.get("/{block_id}", response_model=ContentBlockResponse)  # ← AFTER /search
    async def get_content_block(
        block_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> ContentBlockResponse:
        block = await content_block_service.get_content_block(block_id, current_user, session)
        return ContentBlockResponse.model_validate(block)


    @router.patch("/{block_id}", response_model=ContentBlockResponse)
    async def update_content_block(
        block_id: UUID,
        data: ContentBlockPatch,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> ContentBlockResponse:
        block = await content_block_service.update_content_block(block_id, data, current_user, session)
        return ContentBlockResponse.model_validate(block)


    @router.post("/{block_id}/approve", response_model=ContentBlockResponse)
    async def approve_content_block(
        block_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> ContentBlockResponse:
        block = await content_block_service.approve_content_block(block_id, current_user, session)
        return ContentBlockResponse.model_validate(block)


    @router.post("/{block_id}/unapprove", response_model=ContentBlockResponse)
    async def unapprove_content_block(
        block_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> ContentBlockResponse:
        block = await content_block_service.unapprove_content_block(block_id, current_user, session)
        return ContentBlockResponse.model_validate(block)


    @router.delete("/{block_id}", status_code=status.HTTP_204_NO_CONTENT)
    async def delete_content_block(
        block_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> Response:
        await content_block_service.delete_content_block(block_id, current_user, session)
        return Response(status_code=status.HTTP_204_NO_CONTENT)
    ```

- [x] Task 5: Register content_blocks router in `main.py` (AC: 1–7)
  - [ ] 5.1 Open `services/client-api/src/client_api/main.py`
  - [ ] 5.2 Add import (follow existing router import pattern):
    ```python
    from client_api.api.v1.content_blocks import router as content_blocks_router
    ```
  - [ ] 5.3 Add `include_router` call after the proposals router:
    ```python
    app.include_router(
        content_blocks_router,
        prefix="/api/v1/content-blocks",
        tags=["content-blocks"],
    )
    ```
  - [ ] 5.4 Verify all existing routers (proposals, auth, etc.) remain unchanged — run `pytest tests/api/test_proposals.py -v` to confirm no regressions.

- [x] Task 6: Integration tests at `services/client-api/tests/api/test_content_blocks.py` (AC: 8)
  - [ ] 6.1 Set up fixtures:
    ```python
    @pytest.fixture
    async def cb_user(async_client, db_session):
        """Register + log in a standard company user; yield (client, session, token)."""
        # Register via POST /api/v1/auth/register
        # Verify email via direct SQL: UPDATE client.users SET email_verified = TRUE WHERE email = ...
        # Log in via POST /api/v1/auth/login → extract access_token
        # Yield (async_client, db_session, access_token)
        # Finally: await db_session.rollback()

    @pytest.fixture
    async def another_company(async_client, db_session):
        """Register a second independent company; yield Company B's access_token."""
        # Separate company + user; return only the JWT token for cross-company assertions
    ```
  - [ ] 6.2 `TestContentBlockCRUD` (E07-P1-017 — CRUD lifecycle):
    - `test_create_content_block` — POST with all fields; verify 201, `version=1`, `approved_at=null`, `approved_by=null`, `company_id` matches JWT org; `id` is UUID
    - `test_create_sets_company_from_jwt_not_body` — POST with explicit `company_id` in body (unexpected field should be ignored); verify response `company_id` equals JWT `organization_id`
    - `test_create_rejects_missing_title` — POST without `title`; verify 422
    - `test_create_rejects_missing_body` — POST without `body`; verify 422
    - `test_get_content_block_by_id` — create, GET by ID; verify all fields match
    - `test_get_nonexistent_returns_404` — GET with random UUID; verify 404
    - `test_update_increments_version` — create (version=1), PATCH `title`; verify version=2 in response
    - `test_update_partial_patch_preserves_other_fields` — PATCH only `category`; verify `title`, `body`, `tags` unchanged; `version` incremented
    - `test_approve_sets_approved_fields` — POST /approve; verify `approved_at` not null, `approved_by` = current user UUID
    - `test_unapprove_clears_approved_fields` — approve, then POST /unapprove; verify `approved_at=null`, `approved_by=null`
    - `test_delete_removes_block` — create, DELETE; verify 204; GET by ID returns 404 (physical deletion confirmed)
  - [ ] 6.3 `TestContentBlockSearch` (E07-P1-018, E07-P3-003):
    - `test_search_returns_matching_blocks` — create block with unique keyword "xyzprocurement2026"; search for it; verify block returned; `total >= 1`
    - `test_search_returns_empty_for_no_match` — search for "xyznonexistent999"; verify empty list (not 404, not 500)
    - `test_search_relevance_ordering` (E07-P3-003) — create block A with "tender" 3× in title+body; create block B with "tender" 1× in body; search "tender"; verify block A is first in results (ts_rank higher for more frequent match)
    - `test_search_cross_company_returns_zero_results` (E07-P0-006) — create block in Company A with keyword "secretproposal2026"; search with Company B JWT; verify empty list; "secretproposal2026" NOT in any response field
  - [ ] 6.4 `TestContentBlockTsvectorFreshness` (E07-P2-006, E07-R-009):
    - `test_search_updated_after_patch_body`:
      1. Create block with body containing "procurement"
      2. PATCH body replacing "procurement" with "sourcing"
      3. Search "procurement" → empty list (old keyword no longer matches)
      4. Search "sourcing" → block returned (new keyword matches)
    - `test_search_updated_after_patch_title`:
      1. Create block with title containing "eligibility"
      2. PATCH title replacing "eligibility" with "compliance"
      3. Search "eligibility" → empty list
      4. Search "compliance" → block returned
  - [ ] 6.5 `TestContentBlockListFilters` (E07-P2-007):
    - `test_list_pagination` — seed 5 blocks; `?limit=2&offset=0` returns 2; `?limit=2&offset=2` returns 2; `?limit=2&offset=4` returns 1; `?limit=2&offset=5` returns 0
    - `test_list_filter_by_category` — seed 3 blocks (2 "legal", 1 "technical"); `?category=legal` returns 2
    - `test_list_filter_by_tags_all_match` — seed blocks with tags `["EU","grants"]`, `["EU"]`, `["compliance"]`; `?tags=EU,grants` returns only first block (must contain BOTH tags)
    - `test_list_returns_only_company_blocks` — Company A creates 3 blocks; Company B authenticates; `GET /content-blocks` returns empty list
    - `test_list_default_sort_newest_first` — create blocks B1, B2, B3 in sequence; verify list returns B3, B2, B1 order
  - [ ] 6.6 `TestContentBlockRLS` (E07-P0-006 — P0 priority!):
    - `test_get_cross_company_returns_404` — Company A creates block; Company B JWT `GET /{block_id}` → 404 (not 403)
    - `test_patch_cross_company_returns_404` — Company B JWT PATCH Company A's block → 404
    - `test_approve_cross_company_returns_404` — Company B JWT POST `/approve` on Company A's block → 404
    - `test_unapprove_cross_company_returns_404` — Company B JWT POST `/unapprove` on Company A's block → 404
    - `test_delete_cross_company_returns_404` — Company B JWT DELETE Company A's block → 404; verify block still exists via Company A's JWT

## Dev Notes

### Context: Built on Stories 7.1–7.8

**Migration chain (CRITICAL — understand before implementing):**
- `020_proposal_workspace_schema`: created `proposals`, `proposal_versions`, AND `content_blocks` tables. **This story adds NO new columns** to existing tables — the content_blocks table schema was established in Story 7.1.
- `021_proposal_generation_status` → `022_proposal_checklist` → `023_proposal_agent_results_columns.py` (actual revision: `"023_proposal_agent_results"`) → `024_proposal_pricing_win_themes`
- **This story does NOT require a new migration** IF migration 020 includes the `search_vector` tsvector column with GIN index and trigger. VERIFY migration 020 before adding any new migration.
- If tsvector trigger is absent from 020: add `025_content_blocks_search_trigger.py`. Verify VARCHAR limit on `alembic_version.version_num` — Story 7.7 discovered it is VARCHAR(32). `"025_content_blocks_search_trigger"` = 34 chars — **EXCEEDS 32**. Use `"025_cb_search_trigger"` (21 chars) instead.

**ContentBlock ORM model — LOCATE BEFORE WRITING:**
- Story 7.1 created all three tables together. The `ContentBlock` ORM model is most likely in `services/client-api/src/client_api/models/proposal.py` alongside `Proposal` and `ProposalVersion`.
- DO NOT create a duplicate model. Check `models/proposal.py` first.
- If found: import `ContentBlock` into `content_block_service.py` from `client_api.models.proposal`.

**Current endpoint count: 23** (from Stories 7.1–7.8). This story adds 8 new endpoints on a NEW router prefix `/api/v1/content-blocks`. Router registration in `main.py` IS required (unlike Stories 7.2–7.8 which all added to the existing proposals router).

**Test baseline: 184 tests** (per ATDD checklist for Story 7.8: 161 from 7.1–7.7 + 23 from 7.8). All must remain green.

### CRITICAL: Route Order — `/search` MUST Be Registered Before `/{block_id}`

FastAPI matches routes in registration order. If `/{block_id}` is registered before `/search`, then:
```
GET /api/v1/content-blocks/search?q=procurement
```
matches `/{block_id}` with `block_id="search"` → FastAPI UUID validation fails → HTTP 422 Unprocessable Entity (not search results).

```python
# CORRECT ORDER — define in this sequence:
@router.get("/search", ...)           # 1. Exact path first
async def search_content_blocks(): ...

@router.get("/", ...)                 # 2. List endpoint
async def list_content_blocks(): ...

@router.get("/{block_id}", ...)       # 3. Parameterized AFTER exact paths
async def get_content_block(): ...
```

This is the same pattern used in E06 (`/opportunities/search` before `/opportunities/{id}`).

### E07-R-009: Application-Level tsvector Refresh (MANDATORY)

The test design documents that if `search_vector` is only populated on INSERT (not on UPDATE), searching after a PATCH returns stale results. The `update_content_block` service MUST explicitly recalculate and set `search_vector` when `title` or `body` changes:

```python
# In update_content_block service — ALWAYS execute this when title or body changes:
if tsvector_dirty:
    block.search_vector = func.to_tsvector(
        "english",
        func.coalesce(block.title, "") + " " + func.coalesce(block.body, ""),
    )
```

This is the E07-R-009 mitigation. It MUST be implemented regardless of whether a DB trigger exists. Tests `test_search_updated_after_patch_body` and `test_search_updated_after_patch_title` directly verify this behaviour.

### Full-Text Search — SQLAlchemy + PostgreSQL Pattern

```python
from sqlalchemy import func
from sqlalchemy.dialects.postgresql import TSVECTOR

# ORM mapping (verify in existing model):
search_vector: Mapped[str | None] = mapped_column(TSVECTOR, nullable=True)

# Search query implementation:
ts_query = func.plainto_tsquery("english", query)
stmt = (
    select(ContentBlock)
    .where(
        ContentBlock.company_id == current_user.company_id,  # RLS — MUST come before @@ filter
        ContentBlock.search_vector.op("@@")(ts_query),
    )
    .order_by(func.ts_rank(ContentBlock.search_vector, ts_query).desc())
)
```

Use `plainto_tsquery` (not `to_tsquery`) — handles plain text queries without requiring special syntax. `to_tsquery` would fail on multi-word queries without `&` operator.

### Tags Filtering — PostgreSQL Array `@>` Containment Operator

```python
from sqlalchemy import cast
from sqlalchemy.dialects.postgresql import ARRAY
import sqlalchemy as sa

# Filter: blocks that contain ALL specified tags:
tag_list = [t.strip() for t in tags_csv.split(",") if t.strip()]
stmt = stmt.where(
    ContentBlock.tags.op("@>")(cast(tag_list, ARRAY(sa.String)))
)

# ORM mapping for tags column (verify in existing model):
from sqlalchemy.dialects.postgresql import ARRAY
tags: Mapped[list[str]] = mapped_column(ARRAY(sa.String), nullable=False, server_default="{}")
```

### RLS — Identical Pattern to E07-P0-005/E07-P0-006

E07-P0-006 is a P0 test that MUST pass. Pattern is identical to proposals:
- Every query includes `ContentBlock.company_id == current_user.company_id`
- Returns 404 (not 403) for cross-company access — prevents UUID enumeration
- `company_id` on creation ALWAYS from `current_user.company_id` — never accepted from request body
- Search endpoint: `company_id` filter applied BEFORE `@@` FTS filter so cross-company block content never enters the result set

### Approval Workflow — `approved_by` Stores User UUID

```python
# approve:
block.approved_by = current_user.user_id  # UUID of the approving user, NOT a boolean

# unapprove:
block.approved_at = None
block.approved_by = None
```

The frontend content blocks library modal (S07.16) shows an "approval badge" — it reads `approved_at` (not null = approved). `approved_by` enables audit trail of who approved the block.

### Hard Delete vs. Soft Archive

Content blocks use HARD DELETE (physical row removal). Unlike proposals (which have `status = archived` for soft-delete), content blocks have no status field. Deletion is final. Return `HTTP 204 No Content` on success.

### Existing Infrastructure — DO NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/alembic/versions/020_proposal_workspace_schema.py` | `content_blocks` table | **VERIFY ONLY — DO NOT MODIFY** |
| `services/client-api/src/client_api/models/proposal.py` | Likely contains `ContentBlock` ORM model | **LOCATE + IMPORT — DO NOT DUPLICATE** |
| `services/client-api/src/client_api/services/proposal_service.py` | All proposal/version/agent service functions | **NO CHANGES** |
| `services/client-api/src/client_api/api/v1/proposals.py` | All 23 proposal endpoints | **NO CHANGES** |
| `services/client-api/tests/api/test_proposals.py` | 66 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_versions.py` | 32 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_content_save.py` | 12 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_generate.py` | 13 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_checklist.py` | 19 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_compliance_risk_scoring.py` | 30 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_pricing_win_themes.py` | 23 tests | **DO NOT MODIFY** |

### New Files to Create

| File | Purpose |
|------|---------|
| `services/client-api/src/client_api/schemas/content_block.py` | Pydantic: `ContentBlockCreate`, `ContentBlockPatch`, `ContentBlockResponse` |
| `services/client-api/src/client_api/services/content_block_service.py` | Service functions for all CRUD + search |
| `services/client-api/src/client_api/api/v1/content_blocks.py` | FastAPI router with 8 endpoints |
| `services/client-api/tests/api/test_content_blocks.py` | Integration tests (~20 tests) |
| `services/client-api/alembic/versions/025_cb_search_trigger.py` | *(conditional)* Only if tsvector trigger is absent from migration 020 |

### Files to Modify

| File | Change |
|------|--------|
| `services/client-api/src/client_api/main.py` | Register `content_blocks_router` at `/api/v1/content-blocks` |
| `services/client-api/src/client_api/models/proposal.py` | *(conditional)* Add TSVECTOR mapping to `ContentBlock` if missing |

### New API Endpoints

```
POST   /api/v1/content-blocks                        → Create (any auth user, 201)
GET    /api/v1/content-blocks/search?q=<query>       → Full-text search (any auth user, 200)
GET    /api/v1/content-blocks                        → List + paginate + filter (any auth user, 200)
GET    /api/v1/content-blocks/{block_id}             → Get by ID (any auth user, 200)
PATCH  /api/v1/content-blocks/{block_id}             → Update + version++ + tsvector refresh (200)
POST   /api/v1/content-blocks/{block_id}/approve     → Set approval fields (200)
POST   /api/v1/content-blocks/{block_id}/unapprove   → Clear approval fields (200)
DELETE /api/v1/content-blocks/{block_id}             → Hard delete (204)
```

### CRITICAL Architecture Constraints (Inherited from Stories 7.1–7.8)

1. **`from __future__ import annotations`** at top of every new file
2. **`structlog` for all logging**: `log = structlog.get_logger()` (never `logging`)
3. **`session.flush()` — never `session.commit()`** for route-scoped sessions
4. **`company_id` exclusively from JWT** — never accept `company_id` from request body, path params, or query params
5. **Return 404 (not 403) for cross-company** — prevents UUID enumeration (E07-R-003, E07-P0-006)
6. **`get_current_user` for all endpoints** — content blocks do not require `bid_manager` role (no role specified in S07.09)
7. **Application-level tsvector update on PATCH** — E07-R-009 mitigation; mandatory even if DB trigger exists
8. **`/search` before `/{block_id}` in router** — path collision prevention; FastAPI matches in registration order
9. **Hard delete** — `DELETE /content-blocks/:id` physically removes the row; no `status = archived` pattern
10. **Version increment on every PATCH** — any field update increments `version`; frontend uses this for staleness detection
11. **`await session.refresh(block)` after flush** — required to hydrate the ORM object after SQLAlchemy expression columns (e.g., `func.to_tsvector(...)`) are evaluated server-side

### Running Tests

```bash
cd eusolicit-app/services/client-api

# New story tests only:
pytest tests/api/test_content_blocks.py -v

# Specific test classes:
pytest tests/api/test_content_blocks.py::TestContentBlockCRUD -v
pytest tests/api/test_content_blocks.py::TestContentBlockSearch -v
pytest tests/api/test_content_blocks.py::TestContentBlockTsvectorFreshness -v
pytest tests/api/test_content_blocks.py::TestContentBlockListFilters -v
pytest tests/api/test_content_blocks.py::TestContentBlockRLS -v

# Full regression suite (all 184 existing + new tests):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py \
       tests/api/test_proposal_compliance_risk_scoring.py \
       tests/api/test_proposal_pricing_win_themes.py \
       tests/api/test_content_blocks.py -v
```

### Test Coverage Map (from Epic 7 Test Design)

| Test Design ID | Priority | Scenario | Implementation |
|----------------|----------|----------|----------------|
| **E07-P0-006** | P0 | Content blocks RLS cross-company isolation | `TestContentBlockRLS` (5 tests: GET, PATCH, approve, unapprove, DELETE) |
| **E07-P1-017** | P1 | Content block CRUD lifecycle | `TestContentBlockCRUD` (create → GET → PATCH → approve → unapprove → DELETE) |
| **E07-P1-018** | P1 | Full-text search correctness | `TestContentBlockSearch.test_search_returns_matching_blocks` |
| **E07-P2-006** | P2 | tsvector freshness after PATCH (E07-R-009) | `TestContentBlockTsvectorFreshness.test_search_updated_after_patch_{body,title}` |
| **E07-P2-007** | P2 | Pagination, category, and tag filtering | `TestContentBlockListFilters` (5 tests) |
| **E07-P3-003** | P3 | Search relevance ordering | `TestContentBlockSearch.test_search_relevance_ordering` |
| **E07-R-003** | Risk | RLS bypass — 404 not 403 | All `TestContentBlockRLS.test_*_cross_company_returns_404` tests |
| **E07-R-009** | Risk | tsvector stale after PATCH | Both `test_search_updated_after_patch_*` tests |

### Advisory: Story 7.7 Review Lessons — Apply Here

1. **Verify DB NULL state in tests** — after `DELETE`, verify via GET that block returns 404 (physical deletion). After `unapprove`, verify `approved_at` IS NULL directly (not just that the field is absent from response).
2. **`test_create_sets_company_from_jwt_not_body`** — explicitly verify that any attempt to set `company_id` in the POST body is ignored; response `company_id` always equals JWT `organization_id`. This prevents the common mistake of reading `company_id` from request data.
3. **Cross-company search returns zero results, not error** — search endpoint with Company B JWT should return `[]` for Company A content, never raise 404 or expose any block metadata. Separate test from ID-based 404 tests.

### Project Structure Notes

- New router registered at `/api/v1/content-blocks` (separate resource from `/api/v1/proposals`) — requires `main.py` modification, unlike all previous Stories 7.2–7.8
- Schema file `content_block.py` follows per-story schema file pattern (separate from `proposal_agent_results.py`, `proposal_pricing_win_themes.py`)
- Service file `content_block_service.py` is a new file — content blocks are a distinct domain entity from proposals
- `ContentBlock` ORM model was created in Story 7.1 — locate before writing any model code
- No AI Gateway calls in this story — pure CRUD + PostgreSQL FTS, no `respx` mocks needed
- No frontend work in this story (backend only); content blocks library UI is implemented in S07.16

### References

- [Source: eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md#S07.09] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-006] — content blocks RLS cross-company isolation (P0 test)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P1-017] — CRUD lifecycle test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P1-018] — full-text search correctness test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P2-006] — tsvector freshness after PATCH (E07-R-009 mitigation)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P2-007] — pagination and filter tests
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P3-003] — search relevance ordering
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-009] — tsvector stale search vector risk; mitigation: DB trigger or application-level update
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-003] — RLS bypass; 404 not 403 for cross-company
- [Source: eusolicit-docs/implementation-artifacts/7-8-pricing-assistant-win-theme-agent-integrations.md] — architecture constraints: from __future__, structlog, session.flush, company_id from JWT, 404 not 403, revision ID VARCHAR(32)
- [Source: eusolicit-docs/implementation-artifacts/7-7-compliance-risk-scoring-agent-integrations.md] — service/schema/router file pattern, NullResultResponse, test fixture patterns
- [Source: eusolicit-app/services/client-api/alembic/versions/020_proposal_workspace_schema.py] — content_blocks table definition (VERIFY BEFORE WRITING ANY MIGRATION)
- [Source: eusolicit-app/services/client-api/src/client_api/models/proposal.py] — ContentBlock ORM model location (VERIFY BEFORE WRITING)
- [Source: eusolicit-app/services/client-api/src/client_api/main.py] — existing router registration pattern
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md] — full epic test design with all risks and mitigations

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

None — implementation completed first-run with one route-path fix (used `""` not `"/"` for root routes, consistent with proposals pattern to avoid 307 redirects) and one timestamp fix (explicit Python `datetime.now(tz=utc)` on create to avoid transaction-scoped `NOW()` collapsing all timestamps within the same test session).

### Completion Notes List

1. Migration 020 already contained the full `content_blocks` table, GIN index on `search_vector`, and a `BEFORE INSERT OR UPDATE` trigger — no new migration needed.
2. `ContentBlock` ORM model already existed in `models/content_block.py` with `search_vector` as `deferred=True` (TSVECTOR, DB-managed). Imported from there.
3. Router root routes must use `""` (empty string), not `"/"` — FastAPI issues 307 redirects for trailing-slash mismatches.
4. Python-side `created_at=datetime.now(tz=utc)` set explicitly in `create_content_block` to avoid all blocks sharing the same transaction-scoped `NOW()` value in shared test sessions.
5. E07-R-009 mitigation: `update_content_block` explicitly recalculates `search_vector = func.to_tsvector("english", title + " " + body)` when title or body changes — DB trigger also fires (BEFORE UPDATE), both produce same result.
6. All 219 tests pass: 27 new (Story 7.9) + all prior Stories 7.1–7.8 tests green.

### File List

- `services/client-api/src/client_api/schemas/content_block.py` (created)
- `services/client-api/src/client_api/services/content_block_service.py` (created)
- `services/client-api/src/client_api/api/v1/content_blocks.py` (created)
- `services/client-api/src/client_api/main.py` (modified — added content_blocks_v1 import and include_router)
- `services/client-api/tests/api/test_content_blocks.py` (modified — removed all @pytest.mark.skip decorators)
