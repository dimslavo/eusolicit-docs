# Story 7.2: Proposal CRUD API

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager or company admin**,
I want to create, list, view, update, and archive proposals linked to procurement opportunities,
so that my company can manage a full portfolio of tender responses, each initialised with an empty version ready for content generation.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/proposals` creates a new proposal for the authenticated user's company. Accepts `title` (string, optional, max 500 chars), `opportunity_id` (UUID, optional soft reference) in the request body. Returns HTTP 201 with the created proposal. `company_id` is derived exclusively from the JWT — any `company_id` in the request body is ignored. Simultaneously creates `proposal_versions` row with `version_number = 1`, `content = {"sections": []}`, `created_by = jwt.user_id`, and sets `proposals.current_version_id` to that row's id. Returns the proposal with `current_version_content` (the joined version content) in the response body. Requires `admin` or `bid_manager` role.

2. **AC2** — `GET /api/v1/proposals` returns HTTP 200 with a paginated list of proposals belonging to the authenticated user's company. Supports optional query params: `opportunity_id` (UUID filter), `status` (one of `draft`, `active`, `archived`). Default sort: `created_at` descending. Each item includes `id`, `company_id`, `opportunity_id`, `title`, `status`, `current_version_id`, `created_by`, `created_at`, `updated_at`. Pagination params: `limit` (default 20, max 100), `offset` (default 0). Returns `{"proposals": [], "total": 0}` when no proposals match. Accessible to all authenticated company members.

3. **AC3** — `GET /api/v1/proposals/{proposal_id}` returns HTTP 200 with the proposal record joined with its current version's content. Response includes all proposal fields plus `current_version_content: {"sections": [...]}` from `proposal_versions.content`. Returns HTTP 404 if the proposal does not exist OR belongs to a different company (404 prevents UUID enumeration — E07-P0-005 mitigation; never 403). Accessible to all authenticated company members.

4. **AC4** — `PATCH /api/v1/proposals/{proposal_id}` accepts an optional `title` (string, max 500 chars) and/or `status` (one of `draft`, `active`). Updates the proposal in place; only provided fields are changed. Returns HTTP 200 with the updated proposal (without joined content — PATCH response matches the list item shape). Returns HTTP 404 if not found or cross-company. Returns HTTP 422 if `status` value is invalid. `status = archived` cannot be set via PATCH (use DELETE for archiving). Requires `admin` or `bid_manager` role.

5. **AC5** — `DELETE /api/v1/proposals/{proposal_id}` performs a soft-archive: sets `status = 'archived'` and updates `updated_at`. Returns HTTP 200 with the updated proposal (not 204, to confirm the archive was applied). Returns HTTP 404 if not found or cross-company. A subsequently archived proposal does NOT appear in `GET /proposals` without `status=archived` filter. Requires `admin` or `bid_manager` role.

6. **AC6** — Company-scoped RLS enforcement on all 5 endpoints: `company_id` is always derived from the JWT `organization_id` claim and is never user-configurable. All endpoints that accept `proposal_id` in the path return HTTP 404 — not HTTP 403 — when the proposal UUID does not belong to the JWT company. This prevents cross-company proposal enumeration via UUID guessing (E07-P0-005).

7. **AC7** — `GET /proposals` by default excludes archived proposals (status ≠ `archived`). When `status=archived` query param is provided, only archived proposals are returned. When `status=draft` or `status=active` is provided, only proposals matching that status are returned. All other query combinations are additive filters (AND logic with `opportunity_id`).

8. **AC8** — Unauthenticated requests to all 5 endpoints return HTTP 401. `POST`, `PATCH`, and `DELETE` require `admin` or `bid_manager` role; `contributor`, `reviewer`, and `read_only` roles receive HTTP 403. `GET` (list and detail) is accessible to all authenticated company members regardless of role.

9. **AC9** — Integration tests pass for all 5 endpoints, covering: nominal CRUD flow, company-scoped filtering (list returns only own-company proposals), RLS cross-company 404 check (all 5 endpoint types with a second company JWT), pagination, status filter, 404 on non-existent ID, 422 on invalid input, 401 on unauthenticated, 403 on low-privilege role. Tests located at `services/client-api/tests/api/test_proposals.py`.

## Tasks / Subtasks

- [ ] Task 1: Create `src/client_api/schemas/proposals.py` (AC: 1, 2, 3, 4, 5)
  - [ ] 1.1 Define `ProposalCreateRequest(BaseModel)` with `title: str | None = Field(None, max_length=500)` and `opportunity_id: UUID | None = None`
  - [ ] 1.2 Define `ProposalPatchRequest(BaseModel)` with `title: str | None = Field(None, max_length=500)` and `status: Literal["draft", "active"] | None = None`. Both optional. Status `archived` is excluded — soft-archive is DELETE only.
  - [ ] 1.3 Define `ProposalVersionContentResponse(BaseModel)` with `sections: list[dict]` for embedding current version content in the detail response
  - [ ] 1.4 Define `ProposalResponse(BaseModel, model_config=from_attributes)` with all proposal fields: `id: UUID`, `company_id: UUID`, `opportunity_id: UUID | None`, `title: str | None`, `status: str`, `current_version_id: UUID | None`, `created_by: UUID | None`, `created_at: datetime`, `updated_at: datetime`
  - [ ] 1.5 Define `ProposalDetailResponse(ProposalResponse)` that adds `current_version_content: dict` — the JSONB content from `proposal_versions` (e.g. `{"sections": [...]}`)
  - [ ] 1.6 Define `ProposalListResponse(BaseModel)` with `proposals: list[ProposalResponse]` and `total: int`

- [ ] Task 2: Create `src/client_api/services/proposal_service.py` (AC: 1–9)
  - [ ] 2.1 Add `from __future__ import annotations` header, `structlog` logger, and all required imports: `UUID`, `AsyncSession`, `select`, `func`, `Proposal`, `ProposalVersion`, `CurrentUser`, `HTTPException`
  - [ ] 2.2 Implement helper `def _assert_company_owns_proposal(proposal: Proposal, current_user: CurrentUser) -> None` — raises `HTTPException(status_code=404, detail="Proposal not found")` if `proposal.company_id != current_user.company_id`. Returns 404 not 403 (E07-P0-005 mitigation).
  - [ ] 2.3 Implement `async def create_proposal(request: ProposalCreateRequest, current_user: CurrentUser, session: AsyncSession) -> tuple[Proposal, ProposalVersion]`:
    - Create `Proposal(company_id=current_user.company_id, title=request.title, opportunity_id=request.opportunity_id, status="draft", created_by=current_user.user_id)`
    - `session.add(proposal)` → `await session.flush()` to generate `proposal.id`
    - Create `ProposalVersion(proposal_id=proposal.id, version_number=1, content={"sections": []}, created_by=current_user.user_id, change_summary="Initial version")`
    - `session.add(version)` → `await session.flush()` to generate `version.id`
    - Set `proposal.current_version_id = version.id` → `await session.flush()` → `await session.refresh(proposal)`
    - Log `proposal.created` with `company_id`, `proposal_id`, `opportunity_id`
    - Return `(proposal, version)`
  - [ ] 2.4 Implement `async def list_proposals(current_user: CurrentUser, session: AsyncSession, opportunity_id: UUID | None, status: str | None, limit: int, offset: int) -> tuple[list[Proposal], int]`:
    - Build query filtering by `company_id = current_user.company_id`
    - If `status` param provided: filter `Proposal.status == status`; if not provided: exclude archived (`Proposal.status != "archived"`)
    - If `opportunity_id` provided: filter `Proposal.opportunity_id == opportunity_id`
    - Count total matching rows (separate `SELECT COUNT(*)` for accurate pagination)
    - Apply `ORDER BY created_at DESC`, `LIMIT limit OFFSET offset`
    - Return `(proposals_list, total_count)`
  - [ ] 2.5 Implement `async def get_proposal(proposal_id: UUID, current_user: CurrentUser, session: AsyncSession) -> tuple[Proposal, dict]`:
    - `SELECT * FROM client.proposals WHERE id = :proposal_id` → `scalar_one_or_none()`
    - Raise `HTTPException(404, "Proposal not found")` if None
    - Call `_assert_company_owns_proposal(proposal, current_user)`
    - If `proposal.current_version_id` is not None: query `ProposalVersion` by that id and return its `content`; else return `{"sections": []}`
    - Return `(proposal, content_dict)`
  - [ ] 2.6 Implement `async def update_proposal(proposal_id: UUID, request: ProposalPatchRequest, current_user: CurrentUser, session: AsyncSession) -> Proposal`:
    - Fetch proposal (same fetch-then-ownership pattern as `get_proposal`)
    - If `request.title is not None`: `proposal.title = request.title`
    - If `request.status is not None`: `proposal.status = request.status`
    - `await session.flush()` → `await session.refresh(proposal)`
    - Log `proposal.updated` with `company_id`, `proposal_id`, `changed_keys`
    - Return `proposal`
  - [ ] 2.7 Implement `async def archive_proposal(proposal_id: UUID, current_user: CurrentUser, session: AsyncSession) -> Proposal`:
    - Fetch proposal (same fetch-then-ownership pattern)
    - `proposal.status = "archived"` → `await session.flush()` → `await session.refresh(proposal)`
    - Log `proposal.archived` with `company_id`, `proposal_id`
    - Return `proposal`

- [ ] Task 3: Create `src/client_api/api/v1/proposals.py` (AC: 1–9)
  - [ ] 3.1 Add `from __future__ import annotations` header
  - [ ] 3.2 Define `router = APIRouter(prefix="/proposals", tags=["proposals"])`
  - [ ] 3.3 Implement `POST /` — inject `body: ProposalCreateRequest`, `session`, `current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]`; call `proposal_service.create_proposal(body, current_user, session)`; return `ProposalDetailResponse` with `status_code=201`
  - [ ] 3.4 Implement `GET /` — inject `session`, `current_user: Annotated[CurrentUser, Depends(get_current_user)]`, `opportunity_id: UUID | None = Query(None)`, `status: str | None = Query(None)`, `limit: int = Query(20, ge=1, le=100)`, `offset: int = Query(0, ge=0)`; call `proposal_service.list_proposals(...)`; return `ProposalListResponse`
  - [ ] 3.5 Implement `GET /{proposal_id}` — inject `proposal_id: UUID`, `session`, `current_user: Annotated[CurrentUser, Depends(get_current_user)]`; call `proposal_service.get_proposal(proposal_id, current_user, session)`; return `ProposalDetailResponse`
  - [ ] 3.6 Implement `PATCH /{proposal_id}` — inject `proposal_id: UUID`, `body: ProposalPatchRequest`, `session`, `current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]`; call `proposal_service.update_proposal(proposal_id, body, current_user, session)`; return `ProposalResponse` (list-item shape, no joined content)
  - [ ] 3.7 Implement `DELETE /{proposal_id}` — inject `proposal_id: UUID`, `session`, `current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]`; call `proposal_service.archive_proposal(proposal_id, current_user, session)`; return `ProposalResponse` with HTTP 200 (soft-archive confirmation, not 204)

- [ ] Task 4: Register router in `src/client_api/main.py` (AC: all)
  - [ ] 4.1 Add `from client_api.api.v1 import proposals as proposals_v1` import
  - [ ] 4.2 Add `api_v1_router.include_router(proposals_v1.router)` after the existing router registrations

- [ ] Task 5: Write integration tests `tests/api/test_proposals.py` (AC: 1–9)
  - [ ] 5.1 Define fixture `proposal_client_and_session` — follow `company_client_and_session` pattern from `test_company_profile.py`: register → verify email via SQL UPDATE → login → yield `(client, session, access_token, company_id_str)`
  - [ ] 5.2 Define helper `_register_and_login_second_company(session, client)` → returns `token_b: str` for a second company's JWT (for RLS cross-company tests)
  - [ ] 5.3 Define helper `_register_and_verify_with_role(session, client, token, role)` → creates a user with a specified role in the same company (for role enforcement tests); reuse pattern from `test_company_profile.py`
  - [ ] 5.4 **Test class `TestAC1CreateProposal`** (AC1, AC8, E07-P1-001)
    - `test_post_creates_proposal_with_opportunity_id_returns_201` — POST with `opportunity_id` and `title` → 201, response has `id`, `status = "draft"`, `current_version_id` not None, `current_version_content = {"sections": []}`, `company_id` matches JWT company
    - `test_post_creates_proposal_with_title_only_returns_201` — POST with only `title`, no `opportunity_id` → 201, `opportunity_id == null`
    - `test_post_creates_proposal_minimal_returns_201` — POST with empty body `{}` → 201, `title == null`, `opportunity_id == null`
    - `test_post_initialises_first_version` — POST → verify `current_version_id` not None and GET /{id} returns `current_version_content.sections == []`
    - `test_post_company_id_in_body_is_ignored` — POST with explicit `company_id` of a different company → 201, created proposal `company_id` matches JWT company
    - `test_post_title_over_500_chars_returns_422` — POST with 501-char title → 422
    - `test_unauthenticated_post_returns_401` — POST without token → 401 (AC8)
  - [ ] 5.5 **Test class `TestAC2ListProposals`** (AC2, AC7, AC8, E07-P1-002)
    - `test_get_list_empty_returns_200` — GET before any POST → 200, `proposals == []`, `total == 0`
    - `test_get_list_returns_own_company_proposals` — POST 2 proposals → GET list → `total == 2`
    - `test_get_list_excludes_archived_by_default` — POST proposal, archive it (DELETE), GET list without status filter → archived not returned; GET with `status=archived` → returned
    - `test_get_list_filter_by_opportunity_id` — POST 2 proposals with different `opportunity_id`s → GET `?opportunity_id=X` → only 1 returned
    - `test_get_list_filter_by_status` — POST proposals, PATCH one to `active` → GET `?status=active` → only active returned
    - `test_get_list_pagination` — POST 5 proposals → GET `?limit=2&offset=0` → 2 proposals, `total == 5`; GET `?limit=2&offset=4` → 1 proposal
    - `test_get_list_ordered_by_created_at_desc` — POST proposal A, then B → GET list → B appears first
    - `test_unauthenticated_list_returns_401` — GET without token → 401 (AC8)
  - [ ] 5.6 **Test class `TestAC3GetProposal`** (AC3, AC8, E07-P1-003)
    - `test_get_detail_returns_200_with_current_version_content` — POST proposal → GET /{id} → 200, `current_version_content.sections == []`, all proposal fields present
    - `test_get_nonexistent_proposal_returns_404` — GET with random UUID → 404
    - `test_unauthenticated_get_returns_401` — GET without token → 401 (AC8)
  - [ ] 5.7 **Test class `TestAC4PatchProposal`** (AC4, AC8)
    - `test_patch_title_only_returns_200` — POST proposal → PATCH with only `title` → 200, `title` updated
    - `test_patch_status_to_active_returns_200` — POST proposal → PATCH with `{"status": "active"}` → 200, `status == "active"`
    - `test_patch_both_fields_returns_200` — POST proposal → PATCH with `title` + `status` → 200, both updated
    - `test_patch_status_archived_returns_422` — POST proposal → PATCH with `{"status": "archived"}` → 422 (archived is soft-delete only)
    - `test_patch_invalid_status_returns_422` — PATCH with `{"status": "unknown"}` → 422
    - `test_patch_nonexistent_proposal_returns_404` — PATCH random UUID → 404
    - `test_unauthenticated_patch_returns_401` — PATCH without token → 401 (AC8)
  - [ ] 5.8 **Test class `TestAC5ArchiveProposal`** (AC5, AC7, AC8, E07-P1-004)
    - `test_delete_archives_proposal_returns_200` — POST proposal → DELETE → 200, `status == "archived"`; subsequent GET /{id} → 200 (proposal still exists)
    - `test_delete_archived_proposal_not_in_default_list` — POST proposal → DELETE → GET list (no filter) → archived not in results
    - `test_delete_archived_proposal_visible_with_status_filter` — POST proposal → DELETE → GET `?status=archived` → proposal returned
    - `test_delete_nonexistent_proposal_returns_404` — DELETE random UUID → 404
    - `test_unauthenticated_delete_returns_401` — DELETE without token → 401 (AC8)
  - [ ] 5.9 **Test class `TestAC6CrossCompanyRLS`** (AC6, E07-P0-005)
    - `test_company_b_cannot_get_company_a_proposal_returns_404` — Company A creates proposal; Company B JWT GET /{id} → 404
    - `test_company_b_cannot_patch_company_a_proposal_returns_404` — Company A creates proposal; Company B JWT PATCH /{id} → 404
    - `test_company_b_cannot_delete_company_a_proposal_returns_404` — Company A creates proposal; Company B JWT DELETE /{id} → 404
    - `test_company_b_cannot_view_company_a_versions_endpoint_returns_404` — Company A creates proposal; Company B JWT GET /{id} → 404 (same as detail, confirming no version data leaks)
    - `test_list_only_returns_own_company_proposals` — Company A creates 2 proposals; Company B creates 1; Company A GET list → `total == 2` (E07-P1-002 cross-company variant)
    - `test_post_company_id_body_ignored_uses_jwt_company` — POST with explicit foreign `company_id` → 201, proposal belongs to JWT company
  - [ ] 5.10 **Test class `TestAC8RoleEnforcement`** (AC8)
    - `test_low_privilege_roles_cannot_post` (parametrized: contributor, reviewer, read_only) — 403 on POST
    - `test_low_privilege_roles_cannot_patch` (parametrized: contributor, reviewer, read_only) — 403 on PATCH
    - `test_low_privilege_roles_cannot_delete` (parametrized: contributor, reviewer, read_only) — 403 on DELETE
    - `test_bid_manager_can_post_patch_delete` — 201/200/200 for all write operations
    - `test_all_roles_can_get_list_and_detail` (parametrized: contributor, reviewer, read_only) — 200 on GET list and GET detail

## Dev Notes

### Context: Built on Story 7.1 Schema

Story 7.1 (done) created:
- `client.proposals` — extended with `opportunity_id`, `status`, `current_version_id`, `created_by` (migration 020)
- `client.proposal_versions` — new table for version snapshots (migration 020)
- `client.content_blocks` — new table for reusable content (migration 020; used in S07.09)
- ORM models: `Proposal` (updated), `ProposalVersion` (new), `ContentBlock` (new)
- Factories: `ProposalFactory`, `ProposalVersionFactory`, `ContentBlockFactory` (in `eusolicit-test-utils`)

This story implements the **REST API layer** on top of that schema. No migration is needed.

### Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/alembic/versions/` | Migrations 001–020 | **DO NOT TOUCH** |
| `services/client-api/src/client_api/models/proposal.py` | Updated `Proposal` ORM model (E07 columns) | **DO NOT TOUCH** — read-only reference |
| `services/client-api/src/client_api/models/proposal_version.py` | `ProposalVersion` ORM model | **DO NOT TOUCH** — read-only reference |
| `services/client-api/src/client_api/models/content_block.py` | `ContentBlock` ORM model | **DO NOT TOUCH** — not needed in S07.02 |
| `services/client-api/src/client_api/models/__init__.py` | Exports `Proposal`, `ProposalVersion`, `ContentBlock` | **DO NOT TOUCH** |
| `services/client-api/src/client_api/core/security.py` | `require_role`, `get_current_user`, `CurrentUser` | Reuse, do not modify |
| `services/client-api/src/client_api/dependencies.py` | `get_db_session` | Reuse, do not modify |

### New Files to Create

| File | Purpose |
|------|---------|
| `services/client-api/src/client_api/schemas/proposals.py` | Pydantic request/response schemas |
| `services/client-api/src/client_api/services/proposal_service.py` | CRUD service functions |
| `services/client-api/src/client_api/api/v1/proposals.py` | FastAPI router with 5 endpoints |
| `services/client-api/tests/api/test_proposals.py` | Integration tests for all endpoints |

### Files to Modify

| File | Change |
|------|--------|
| `services/client-api/src/client_api/main.py` | Import and register `proposals_v1.router` in `api_v1_router` |

### New API Endpoint URLs

```
POST   /api/v1/proposals              → create proposal + first version (admin/bid_manager)
GET    /api/v1/proposals              → list company proposals with filters (any auth)
GET    /api/v1/proposals/{id}         → get proposal with current version content (any auth)
PATCH  /api/v1/proposals/{id}         → update title/status (admin/bid_manager)
DELETE /api/v1/proposals/{id}         → soft-archive (set status=archived) (admin/bid_manager)
```

### Architecture Constraints (MUST FOLLOW)

**1. `from __future__ import annotations` at the top of every file.** Project-wide rule.

**2. `structlog` for all logging. No `print()` or `logging.getLogger()`:**
```python
import structlog
log = structlog.get_logger()
```

**3. `session.flush()` then `session.refresh()` after mutations.** Same pattern used in `company_service.py` and `espd_service.py`. Do NOT call `session.commit()` — the session is managed by the `get_db_session` dependency.

**4. `company_id` exclusively from JWT.** Never accept `company_id` from request body, path params, or query params.

**5. Return 404 (not 403) for cross-company access.** This prevents UUID enumeration. The pattern is in `espd_service._assert_company_owns_profile()` — mirror exactly for proposals.

**6. DELETE is soft-archive returning HTTP 200**, not hard-delete returning 204. This is intentional — the proposal and all its versions are preserved; only `status` changes to `archived`.

**7. `GET /proposals` excludes archived by default.** When no `status` filter is provided, the service must add `Proposal.status != "archived"` to the WHERE clause.

### Proposal Creation — Two-Phase Insert (Circular FK)

`proposals.current_version_id` has a FK to `proposal_versions.id`. The circular reference (`proposals → proposal_versions → proposals`) requires a two-phase insert:

```python
# Phase 1: Create proposal without current_version_id (NULL initially)
proposal = Proposal(
    company_id=current_user.company_id,
    title=request.title,
    opportunity_id=request.opportunity_id,
    status="draft",
    created_by=current_user.user_id,
    current_version_id=None,  # NULL until version is created
)
session.add(proposal)
await session.flush()  # generates proposal.id

# Phase 2: Create first version (FK → proposal.id is now valid)
version = ProposalVersion(
    proposal_id=proposal.id,
    version_number=1,
    content={"sections": []},
    created_by=current_user.user_id,
    change_summary="Initial version",
)
session.add(version)
await session.flush()  # generates version.id

# Phase 3: Set current_version_id on proposal
proposal.current_version_id = version.id
await session.flush()
await session.refresh(proposal)
```

This pattern is required because the DB FK constraint would reject a single-statement INSERT that references a version that doesn't exist yet.

### Company-Scoped RLS Pattern

Mirror the ESPD pattern from `espd_service.py`:

```python
def _assert_company_owns_proposal(proposal: Proposal, current_user: CurrentUser) -> None:
    """Raise 404 if the proposal does not belong to the current user's company.

    Returns 404 (not 403) to prevent UUID enumeration by authenticated users.
    See E07-R-003 mitigation in test-design-epic-07.md and E07-P0-005.
    """
    if proposal.company_id != current_user.company_id:
        raise HTTPException(status_code=404, detail="Proposal not found")
```

For list operations, always filter by `company_id` in the SQL WHERE clause — never return all proposals and filter in Python.

### Role Guard Pattern

Same pattern used in `espd.py`:

```python
# POST, PATCH, DELETE — admin or bid_manager only:
current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]

# GET list + detail — any authenticated member:
current_user: Annotated[CurrentUser, Depends(get_current_user)]
```

`require_role("bid_manager")` permits `admin` and `bid_manager` roles (minimum privilege level, not exact match).

### Pydantic Schemas Specification

```python
# schemas/proposals.py
from __future__ import annotations

from datetime import datetime
from uuid import UUID
from typing import Literal

from pydantic import BaseModel, ConfigDict, Field


class ProposalCreateRequest(BaseModel):
    title: str | None = Field(None, max_length=500)
    opportunity_id: UUID | None = None


class ProposalPatchRequest(BaseModel):
    title: str | None = Field(None, max_length=500)
    # `archived` is NOT allowed via PATCH — use DELETE endpoint to soft-archive
    status: Literal["draft", "active"] | None = None


class ProposalResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    company_id: UUID
    opportunity_id: UUID | None
    title: str | None
    status: str
    current_version_id: UUID | None
    created_by: UUID | None
    created_at: datetime
    updated_at: datetime


class ProposalDetailResponse(ProposalResponse):
    """Extends ProposalResponse with joined current version content."""
    current_version_content: dict  # e.g. {"sections": [...]}


class ProposalListResponse(BaseModel):
    proposals: list[ProposalResponse]
    total: int
```

**Note:** `ProposalDetailResponse` inherits from `ProposalResponse` but cannot be built directly from the ORM object (since `current_version_content` is not a column). Build it manually in the router:

```python
# In router POST and GET /{id}:
proposal, version_content = await proposal_service.create_proposal(body, current_user, session)
return ProposalDetailResponse(
    **ProposalResponse.model_validate(proposal).model_dump(),
    current_version_content=version_content,
)
```

### Service Layer — List Proposals with Pagination

```python
async def list_proposals(
    current_user: CurrentUser,
    session: AsyncSession,
    opportunity_id: UUID | None = None,
    status: str | None = None,
    limit: int = 20,
    offset: int = 0,
) -> tuple[list[Proposal], int]:
    # Base filter: always scoped to JWT company
    filters = [Proposal.company_id == current_user.company_id]

    # Status filtering: default excludes archived
    if status is not None:
        filters.append(Proposal.status == status)
    else:
        filters.append(Proposal.status != "archived")

    # Optional opportunity filter
    if opportunity_id is not None:
        filters.append(Proposal.opportunity_id == opportunity_id)

    # Count query
    count_stmt = select(func.count()).select_from(Proposal).where(*filters)
    total = (await session.execute(count_stmt)).scalar_one()

    # Data query with ordering and pagination
    stmt = (
        select(Proposal)
        .where(*filters)
        .order_by(Proposal.created_at.desc())
        .limit(limit)
        .offset(offset)
    )
    result = await session.execute(stmt)
    proposals = list(result.scalars().all())

    return proposals, total
```

### GET Detail — Joining Current Version Content

```python
async def get_proposal(
    proposal_id: UUID,
    current_user: CurrentUser,
    session: AsyncSession,
) -> tuple[Proposal, dict]:
    stmt = select(Proposal).where(Proposal.id == proposal_id)
    result = await session.execute(stmt)
    proposal = result.scalar_one_or_none()

    if proposal is None:
        raise HTTPException(status_code=404, detail="Proposal not found")

    _assert_company_owns_proposal(proposal, current_user)

    # Fetch current version content (empty sections if no version set)
    content: dict = {"sections": []}
    if proposal.current_version_id is not None:
        version_stmt = select(ProposalVersion).where(
            ProposalVersion.id == proposal.current_version_id
        )
        version_result = await session.execute(version_stmt)
        version = version_result.scalar_one_or_none()
        if version is not None:
            content = version.content

    return proposal, content
```

### DELETE Router — Returns ProposalResponse (200, not 204)

```python
@router.delete("/{proposal_id}", status_code=200, response_model=ProposalResponse)
async def archive_proposal(
    proposal_id: UUID,
    session: Annotated[AsyncSession, Depends(get_db_session)],
    current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
) -> ProposalResponse:
    """Soft-archive a proposal (set status=archived). Returns 200 with updated proposal."""
    proposal = await proposal_service.archive_proposal(proposal_id, current_user, session)
    return ProposalResponse.model_validate(proposal)
```

### Test Pattern — Cross-Company RLS (E07-P0-005)

Mirror the pattern from `tests/api/test_espd_profile.py > TestAC6CrossCompanyRLS`:

```python
async def _register_and_login_second_company(session, client) -> str:
    """Register a second company and return its access token."""
    uid = uuid.uuid4().hex[:8]
    await client.post("/api/v1/auth/register", json={
        "email": f"company-b-{uid}@test.com",
        "password": "SecurePass1",
        "full_name": "Company B Admin",
        "company_name": f"Company B {uid}",
    })
    await session.execute(
        text("UPDATE client.users SET email_verified = TRUE WHERE email = :email"),
        {"email": f"company-b-{uid}@test.com"},
    )
    await session.flush()
    login_resp = await client.post("/api/v1/auth/login", json={
        "email": f"company-b-{uid}@test.com", "password": "SecurePass1"
    })
    return login_resp.json()["access_token"]


class TestAC6CrossCompanyRLS:
    async def test_company_b_cannot_get_company_a_proposal_returns_404(
        self, proposal_client_and_session
    ):
        client, session, token_a, _ = proposal_client_and_session
        # Company A creates a proposal
        resp_a = await client.post(
            "/api/v1/proposals",
            json={"title": "Company A Proposal"},
            headers={"Authorization": f"Bearer {token_a}"},
        )
        proposal_id = resp_a.json()["id"]

        # Company B attempts to access it
        token_b = await _register_and_login_second_company(session, client)
        resp_b = await client.get(
            f"/api/v1/proposals/{proposal_id}",
            headers={"Authorization": f"Bearer {token_b}"},
        )
        assert resp_b.status_code == 404  # NOT 403 — E07-P0-005
```

The test covers all 5 endpoint types (E07-P0-005 requires GET detail, PATCH, DELETE, GET versions, POST generate — the last two are S07.03/S07.05; S07.02 tests cover GET detail + PATCH + DELETE and list isolation).

### Test Coverage Alignment (from test-design-epic-07.md)

| Epic Test ID | Priority | Story | Scenario | Implemented In |
|-------------|----------|-------|----------|----------------|
| **E07-P0-005** | P0 | S07.02, S07.03, S07.05 | Company B JWT → all proposal endpoints for Company A proposal → 404 | `TestAC6CrossCompanyRLS` (this story covers GET detail, PATCH, DELETE, list isolation) |
| **E07-P1-001** | P1 | S07.02 | Proposal creation from opportunity — `company_id` from JWT, not body | `TestAC1CreateProposal.test_post_creates_proposal_with_opportunity_id_returns_201` |
| **E07-P1-002** | P1 | S07.02 | Proposal list filtering — `opportunity_id` filter + status filter + pagination | `TestAC2ListProposals` |
| **E07-P1-003** | P1 | S07.02 | Proposal detail includes current version content joined | `TestAC3GetProposal.test_get_detail_returns_200_with_current_version_content` |
| **E07-P1-004** | P1 | S07.02 | Proposal soft-archive — `status = archived`; excluded from default list | `TestAC5ArchiveProposal` |
| **E07-P2-001** | P2 | S07.02 | Proposal not found returns 404 | `TestAC3GetProposal.test_get_nonexistent_proposal_returns_404`, `TestAC4PatchProposal.test_patch_nonexistent_proposal_returns_404`, `TestAC5ArchiveProposal.test_delete_nonexistent_proposal_returns_404` |
| **E07-P3-001** | P3 | All S07.02–S07.10 | OpenAPI schema coverage — all endpoints appear in `/openapi.json` | Verified by FastAPI router registration + response_model declarations |

**Note on E07-P0-005 scope:** The full E07-P0-005 test covers 5 endpoint types including `GET /proposals/:id/versions` (S07.03) and `POST /proposals/:id/generate` (S07.05). This story (S07.02) implements and tests 3 of those 5 types (GET detail, PATCH, DELETE) plus list isolation. The remaining 2 types are tested in S07.03 and S07.05 respectively.

### Relevant Existing Files (for Pattern Reference)

| File | Why Relevant |
|------|-------------|
| `services/client-api/src/client_api/services/espd_service.py` | CRUD service pattern (`session.flush`, `session.refresh`, `_assert_company_owns_*`, structlog) |
| `services/client-api/src/client_api/api/v1/espd.py` | Router pattern (`APIRouter`, `require_role`, `get_current_user`, `Depends`, return schemas) |
| `services/client-api/tests/api/test_espd_profile.py` | Test fixture and cross-company RLS pattern |
| `services/client-api/tests/api/test_company_profile.py` | `company_client_and_session` fixture and `_register_and_verify_with_role` helper |
| `services/client-api/src/client_api/core/security.py` | `require_role`, `get_current_user`, `CurrentUser` |
| `services/client-api/src/client_api/dependencies.py` | `get_db_session` |
| `services/client-api/src/client_api/models/proposal.py` | `Proposal` ORM model (E07 columns) |
| `services/client-api/src/client_api/models/proposal_version.py` | `ProposalVersion` ORM model |
| `eusolicit-docs/implementation-artifacts/7-1-proposal-version-db-schema-migrations.md` | S07.01 context: schema details, factory specs |
| `eusolicit-docs/test-artifacts/test-design-epic-07.md` | Epic test scenarios, risk IDs, E07-P0-005, E07-P1-001 through P1-004, E07-P2-001 |

### Running Tests

```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_proposals.py -v
```

Same `pytest-asyncio`, `httpx`, `pytest_asyncio` stack as existing tests.

### Security Notes (E07-R-003)

- `proposals.company_id` FK to `client.companies(id)` enforces referential integrity at DB level (from migration 020)
- Row-level access control is application-enforced via JWT `organization_id` claim in all proposal endpoints — consistent with E06 and E11 patterns
- `opportunity_id` on proposals is a soft reference (no DB FK) — consistent with cross-schema reference pattern
- All cross-company access must return 404 (not 403) per E07-P0-005 spec to avoid ID enumeration
- `company_id` from request body must be silently ignored (never override JWT-derived company_id)

### `CurrentUser` Fields Reference

The `CurrentUser` object injected by `get_current_user`/`require_role` provides:
- `current_user.company_id: UUID` — from JWT `organization_id` claim
- `current_user.user_id: UUID` — from JWT `sub` claim
- `current_user.role: str` — from JWT `role` claim

Use `current_user.user_id` for `created_by` fields in proposal and version rows.
