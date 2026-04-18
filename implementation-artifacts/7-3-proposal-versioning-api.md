# Story 7.3: Proposal Versioning API

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager or company admin**,
I want to snapshot proposal content into numbered versions, list and view past versions, compute a section-level diff between any two versions, and roll back to a past version with one click,
so that my company has a complete, auditable history of every change made to a tender response and can recover from unwanted edits without losing work.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/proposals/{proposal_id}/versions` snapshots the current version's content into a new `proposal_versions` row. `version_number` = `MAX(version_number)` for that proposal + 1. The optional `change_summary` field is stored if provided. `proposals.current_version_id` is updated to the new version's id. Returns HTTP 201 with the new `VersionResponse`. Returns HTTP 404 if the proposal does not exist or belongs to a different company (E07-R-003 mitigation). Requires `admin` or `bid_manager` role.

2. **AC2** — `GET /api/v1/proposals/{proposal_id}/versions` returns HTTP 200 with all versions for the proposal, ordered `version_number DESC` (newest first). Each item includes `id`, `proposal_id`, `version_number`, `content`, `created_by`, `change_summary`, `created_at`. Returns HTTP 404 for cross-company or non-existent proposals. Accessible to all authenticated company members.

3. **AC3** — `GET /api/v1/proposals/{proposal_id}/versions/{version_id}` returns HTTP 200 with the full content of the specified version. Returns HTTP 404 if the version does not exist for that proposal, or if the proposal is cross-company. Accessible to all authenticated company members.

4. **AC4** — `GET /api/v1/proposals/{proposal_id}/versions/diff?from={version_id_a}&to={version_id_b}` returns HTTP 200 with a section-level diff. Both `from` and `to` must be valid UUIDs referencing versions of the same proposal. The diff compares `sections[]` arrays by `key`:
   - `added`: key exists in `to` but not in `from` — includes `after` body.
   - `removed`: key exists in `from` but not in `to` — includes `before` body.
   - `changed`: key exists in both but `body` differs — includes `before` and `after`.
   - `unchanged`: key exists in both with identical `body` — body fields omitted from response.
   Returns HTTP 422 if `from` or `to` params are missing or not valid UUIDs. Returns HTTP 404 if either version is not found for that proposal.

5. **AC5** — `POST /api/v1/proposals/{proposal_id}/versions/{version_id}/rollback` creates a new version row with the target version's content. `version_number` = next sequential number. `change_summary` defaults to `"Rolled back to version {target.version_number}"` unless an override is provided in the request body. `proposals.current_version_id` is updated to the new version's id. The target version and all intermediate versions are preserved (no deletion). Returns HTTP 201 with the new `VersionResponse`. Returns HTTP 404 for cross-company or non-existent proposal/version. Requires `admin` or `bid_manager` role.

6. **AC6** — All five endpoints enforce company-scoped ownership via the same `_assert_company_owns_proposal` helper from `proposal_service.py`. All cross-company or non-existent proposal accesses return HTTP 404 (never 403) — E07-P0-005 mitigation. The `GET /versions` and `GET /versions/{vid}` endpoints must also validate the version belongs to the specified proposal (no cross-proposal version access).

7. **AC7** — Version creation (`POST /versions`) and rollback (`POST /versions/:vid/rollback`) both use `SELECT FOR UPDATE` on the `proposals` row to prevent version-number races under concurrent writes — E07-R-006 mitigation. The next `version_number` is computed inside the lock as `SELECT MAX(version_number) FROM client.proposal_versions WHERE proposal_id = :id`.

8. **AC8** — Integration tests pass for all five endpoints covering: nominal version creation, list ordering (newest first), detail retrieval, section-level diff accuracy (added/removed/changed/unchanged), rollback version integrity (new version created, historical preserved, `current_version_id` updated), company-scoped RLS 404 for all endpoint types with a second company JWT, concurrent rollback + PATCH race producing consistent state. Tests located at `services/client-api/tests/api/test_proposal_versions.py`.

## Tasks / Subtasks

- [x] Task 1: Create `src/client_api/schemas/proposal_versions.py` (AC: 1–5)
  - [x] 1.1 Add `from __future__ import annotations` header
  - [x] 1.2 Define `VersionCreateRequest(BaseModel)` with `change_summary: str | None = None`
  - [x] 1.3 Define `RollbackRequest(BaseModel)` with `change_summary: str | None = None` (overrides default rollback summary)
  - [x] 1.4 Define `VersionResponse(BaseModel, model_config=from_attributes)` with `id: UUID`, `proposal_id: UUID`, `version_number: int`, `content: dict`, `created_by: UUID | None`, `change_summary: str | None`, `created_at: datetime`
  - [x] 1.5 Define `VersionListResponse(BaseModel)` with `versions: list[VersionResponse]` and `total: int`
  - [x] 1.6 Define `SectionDiffEntry(BaseModel)` with `key: str`, `status: Literal["added", "removed", "changed", "unchanged"]`, `before: str | None = None`, `after: str | None = None`
  - [x] 1.7 Define `VersionDiffResponse(BaseModel)` with `from_version_id: UUID`, `to_version_id: UUID`, `from_version_number: int`, `to_version_number: int`, `sections: list[SectionDiffEntry]`

- [x] Task 2: Add versioning functions to `src/client_api/services/proposal_service.py` (AC: 1–8)
  - [x] 2.1 Add private helper `_get_proposal_for_update` with SELECT FOR UPDATE (E07-R-006)
  - [x] 2.2 Add private helper `_next_version_number` using COALESCE(MAX)+1
  - [x] 2.3 Implement `create_version` with asyncio write-lock + two-phase flush
  - [x] 2.4 Implement `list_versions` (read-only, DESC order)
  - [x] 2.5 Implement `get_version` with proposal_id filter (prevents cross-proposal access)
  - [x] 2.6 Implement `diff_versions` with `_compute_section_diff` pure helper
  - [x] 2.7 Implement `rollback_version` with asyncio write-lock + two-phase flush

- [x] Task 3: Add versioning endpoints to `src/client_api/api/v1/proposals.py` (AC: 1–7)
  - [x] 3.1 Add imports for new schemas from `proposal_versions`
  - [x] 3.2 Add `POST /{proposal_id}/versions` (201, bid_manager)
  - [x] 3.3 Add `GET /{proposal_id}/versions` (200, any auth)
  - [x] 3.4 CRITICAL route order: diff BEFORE /{version_id}
  - [x] 3.5 Add `GET /{proposal_id}/versions/diff` with `from_: UUID = Query(..., alias="from")`
  - [x] 3.6 Add `GET /{proposal_id}/versions/{version_id}` (200, any auth)
  - [x] 3.7 Add `POST /{proposal_id}/versions/{version_id}/rollback` (201, bid_manager)

- [x] Task 4: Integration tests `tests/api/test_proposal_versions.py` (AC: 1–8) — pre-written ATDD, all 32 tests pass
  - [x] 4.1 `proposal_and_token` fixture
  - [x] 4.2 `_create_version` helper
  - [x] 4.3 `_register_and_login_second_company` helper
  - [x] 4.4 TestAC1CreateVersion (7 tests) — all pass
  - [x] 4.5 TestAC2ListVersions (4 tests) — all pass
  - [x] 4.6 TestAC3GetVersion (5 tests) — all pass
  - [x] 4.7 TestAC4DiffVersions (7 tests) — all pass
  - [x] 4.8 TestAC5Rollback (6 tests) — all pass
  - [x] 4.9 TestVersionNumberIntegrity (3 tests incl. concurrent) — all pass

## Dev Notes

### Context: Built on Stories 7.1 and 7.2

**Story 7.1** (done) created:
- Migration 020: `client.proposals`, `client.proposal_versions`, `client.content_blocks` tables
- ORM models: `Proposal` (updated), `ProposalVersion` (new), `ContentBlock` (new) — all in `services/client-api/src/client_api/models/`
- Factories: `ProposalFactory`, `ProposalVersionFactory`, `ContentBlockFactory` in `eusolicit-test-utils`
- `proposal_versions` UNIQUE constraint: `(proposal_id, version_number)` — this prevents duplicate version numbers at the DB level (additional safety beyond the app-layer `SELECT FOR UPDATE` lock)

**Story 7.2** (done) created:
- `services/client-api/src/client_api/schemas/proposals.py` — `ProposalResponse`, `ProposalDetailResponse`, `ProposalListResponse`, `ProposalCreateRequest`, `ProposalPatchRequest`
- `services/client-api/src/client_api/services/proposal_service.py` — `create_proposal`, `list_proposals`, `get_proposal`, `update_proposal`, `archive_proposal`, and the critical helper `_assert_company_owns_proposal`
- `services/client-api/src/client_api/api/v1/proposals.py` — FastAPI `APIRouter` with 5 CRUD endpoints, registered under `/proposals`
- `services/client-api/tests/api/test_proposals.py` — integration tests including `_register_and_login_second_company` helper

This story adds **5 new endpoints** to the existing proposals router and extends the service file. No new router registration in `main.py` is needed — the versioning endpoints are sub-paths of `/proposals/{id}`.

### Existing Infrastructure — DO NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/alembic/versions/020_proposal_workspace_schema.py` | Migration with `proposal_versions` table + UNIQUE(proposal_id, version_number) | **DO NOT TOUCH** |
| `services/client-api/src/client_api/models/proposal.py` | `Proposal` ORM with `current_version_id`, `opportunity_id`, `status`, `created_by` | **DO NOT TOUCH** — read-only reference |
| `services/client-api/src/client_api/models/proposal_version.py` | `ProposalVersion` ORM: `id, proposal_id, version_number, content (JSONB), created_by, change_summary, created_at` | **DO NOT TOUCH** — read-only reference |
| `services/client-api/src/client_api/services/proposal_service.py` | S07.02 CRUD functions + `_assert_company_owns_proposal` helper | **ADD new functions only** — do not modify or remove existing |
| `services/client-api/src/client_api/api/v1/proposals.py` | S07.02 router with 5 CRUD endpoints | **ADD new endpoints only** |
| `services/client-api/src/client_api/core/security.py` | `require_role`, `get_current_user`, `CurrentUser` | **Reuse unchanged** |
| `services/client-api/src/client_api/dependencies.py` | `get_db_session` | **Reuse unchanged** |
| `services/client-api/tests/api/test_proposals.py` | S07.02 tests + `_register_and_login_second_company` helper | **DO NOT MODIFY** |

### New Files to Create

| File | Purpose |
|------|---------|
| `services/client-api/src/client_api/schemas/proposal_versions.py` | Pydantic schemas for versioning endpoints |
| `services/client-api/tests/api/test_proposal_versions.py` | Integration tests for all versioning endpoints |

### Files to Modify

| File | Change |
|------|--------|
| `services/client-api/src/client_api/services/proposal_service.py` | Add `_get_proposal_for_update`, `_next_version_number`, `create_version`, `list_versions`, `get_version`, `diff_versions`, `rollback_version` |
| `services/client-api/src/client_api/api/v1/proposals.py` | Add 5 new route handlers; import from `proposal_versions` schemas |

### New API Endpoint URLs

```
POST   /api/v1/proposals/{proposal_id}/versions                     → snapshot + new version (admin/bid_manager)
GET    /api/v1/proposals/{proposal_id}/versions                     → list versions newest first (any auth)
GET    /api/v1/proposals/{proposal_id}/versions/diff?from=X&to=Y   → section-level diff (any auth)
GET    /api/v1/proposals/{proposal_id}/versions/{version_id}        → get version detail (any auth)
POST   /api/v1/proposals/{proposal_id}/versions/{version_id}/rollback → rollback to version (admin/bid_manager)
```

### Architecture Constraints (MUST FOLLOW)

**1. `from __future__ import annotations` at the top of every new file.** Project-wide rule.

**2. `structlog` for all logging:**
```python
import structlog
log = structlog.get_logger()
```

**3. `session.flush()` then `session.refresh()` after mutations.** Never `session.commit()` — session managed by `get_db_session` dependency.

**4. `company_id` exclusively from JWT.** The `_assert_company_owns_proposal` helper (already in `proposal_service.py`) handles all cross-company 404 enforcement.

**5. Return 404 (not 403) for cross-company access** — prevents UUID enumeration per E07-R-003 mitigation and E07-P0-005.

**6. The `bid_manager` role guard permits `admin` and `bid_manager`** (minimum privilege, not exact match). `require_role("bid_manager")` is the correct dependency call. GET endpoints use `get_current_user` (any authenticated member).

### CRITICAL: Route Registration Order in `proposals.py`

FastAPI matches routes in the **ORDER they are registered** in the router file. The `/diff` static path segment must come BEFORE the parameterised `/{version_id}` path:

```python
# ✅ CORRECT — /diff route registered first
@router.get("/{proposal_id}/versions/diff", response_model=VersionDiffResponse)
async def get_version_diff(..., from_: UUID = Query(..., alias="from"), ...):
    ...

@router.get("/{proposal_id}/versions/{version_id}", response_model=VersionResponse)
async def get_version_detail(...):
    ...

# ❌ WRONG — if /{version_id} comes first, a request to /versions/diff will try
# to cast the string "diff" as a UUID → FastAPI returns 422 instead of routing to the diff handler
```

**Why this matters:** Even though FastAPI's UUID type would reject "diff" and might skip to the next route in theory, Starlette's path matching evaluates routes in order — placing the static `/diff` path before `/{version_id}` guarantees correct routing behaviour regardless of FastAPI version.

### SELECT FOR UPDATE — Preventing Version Number Race (E07-R-006)

Version creation and rollback MUST acquire a row-level lock on the `proposals` row before computing the next `version_number`. This prevents two concurrent requests from reading the same MAX(version_number) and creating duplicate version numbers:

```python
# _get_proposal_for_update — use in create_version and rollback_version
from sqlalchemy import select

async def _get_proposal_for_update(
    proposal_id: UUID,
    current_user: CurrentUser,
    session: AsyncSession,
) -> Proposal:
    stmt = (
        select(Proposal)
        .where(Proposal.id == proposal_id)
        .with_for_update()  # ← acquires ROW-LEVEL LOCK
    )
    result = await session.execute(stmt)
    proposal = result.scalar_one_or_none()
    if proposal is None:
        raise HTTPException(status_code=404, detail="Proposal not found")
    _assert_company_owns_proposal(proposal, current_user)
    return proposal
```

**Note:** `_get_proposal_for_update` is for write operations (create, rollback) only. Read operations (list, detail, diff) use a standard `select()` without `with_for_update()` — locking for reads would unnecessarily serialize concurrent readers.

### Next Version Number Query

```python
async def _next_version_number(proposal_id: UUID, session: AsyncSession) -> int:
    """Compute next version number inside an active SELECT FOR UPDATE lock on the proposal row."""
    from sqlalchemy import func, select
    stmt = select(
        func.coalesce(func.max(ProposalVersion.version_number), 0) + 1
    ).where(ProposalVersion.proposal_id == proposal_id)
    result = await session.execute(stmt)
    return result.scalar_one()
```

**Note:** The `UNIQUE(proposal_id, version_number)` DB constraint from migration 020 provides a safety net — if two concurrent writes somehow produce the same version_number despite the lock, the DB will raise an `IntegrityError` rather than silently create a duplicate.

### Diff Algorithm Implementation

The section-level diff compares `sections[]` by `key`. Sections are ordered arrays; keys must be unique within a version:

```python
def _compute_section_diff(
    from_content: dict, to_content: dict
) -> list[SectionDiffEntry]:
    """Compare two version content dicts at the section level."""
    from_sections = {s["key"]: s["body"] for s in from_content.get("sections", [])}
    to_sections = {s["key"]: s["body"] for s in to_content.get("sections", [])}

    # Maintain order: from-sections first, then new sections in to
    result = []
    seen_keys = set()

    for key, from_body in from_sections.items():
        seen_keys.add(key)
        if key not in to_sections:
            result.append(SectionDiffEntry(key=key, status="removed", before=from_body))
        elif to_sections[key] != from_body:
            result.append(SectionDiffEntry(key=key, status="changed", before=from_body, after=to_sections[key]))
        else:
            result.append(SectionDiffEntry(key=key, status="unchanged"))

    for key, to_body in to_sections.items():
        if key not in seen_keys:
            result.append(SectionDiffEntry(key=key, status="added", after=to_body))

    return result
```

**Note:** The diff is implemented as a pure function — no DB calls — so it can be unit-tested independently from the service layer.

### Schemas Specification

```python
# schemas/proposal_versions.py
from __future__ import annotations

from datetime import datetime
from typing import Literal
from uuid import UUID

from pydantic import BaseModel, ConfigDict


class VersionCreateRequest(BaseModel):
    change_summary: str | None = None


class RollbackRequest(BaseModel):
    change_summary: str | None = None


class VersionResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: UUID
    proposal_id: UUID
    version_number: int
    content: dict          # {"sections": [{"key": str, "title": str, "body": str}]}
    created_by: UUID | None
    change_summary: str | None
    created_at: datetime


class VersionListResponse(BaseModel):
    versions: list[VersionResponse]
    total: int


class SectionDiffEntry(BaseModel):
    key: str
    status: Literal["added", "removed", "changed", "unchanged"]
    before: str | None = None   # present for "removed" and "changed"
    after: str | None = None    # present for "added" and "changed"


class VersionDiffResponse(BaseModel):
    from_version_id: UUID
    to_version_id: UUID
    from_version_number: int
    to_version_number: int
    sections: list[SectionDiffEntry]
```

### Diff Endpoint — `from` is a Python Reserved Keyword

The query parameter `from` conflicts with Python's `from` keyword. Use FastAPI's `alias` to handle it:

```python
from fastapi import Query
from uuid import UUID

@router.get("/{proposal_id}/versions/diff", response_model=VersionDiffResponse)
async def get_version_diff(
    proposal_id: UUID,
    from_: UUID = Query(..., alias="from"),   # ← alias maps URL "from" → Python "from_"
    to: UUID = Query(...),
    session: Annotated[AsyncSession, Depends(get_db_session)] = ...,
    current_user: Annotated[CurrentUser, Depends(get_current_user)] = ...,
) -> VersionDiffResponse:
    return await proposal_service.diff_versions(proposal_id, from_, to, current_user, session)
```

### Test Fixture Pattern

Reuse the `proposal_client_and_session` fixture pattern from `test_proposals.py`:

```python
@pytest_asyncio.fixture
async def proposal_and_token(async_client, db_session):
    """Register → verify → login → create proposal. Yields (client, session, token, proposal_id)."""
    uid = uuid.uuid4().hex[:8]
    email = f"test-{uid}@example.com"
    await async_client.post("/api/v1/auth/register", json={
        "email": email, "password": "SecurePass1",
        "full_name": "Test User", "company_name": f"Test Co {uid}",
    })
    await db_session.execute(
        text("UPDATE client.users SET email_verified = TRUE WHERE email = :email"),
        {"email": email},
    )
    await db_session.flush()
    login = await async_client.post("/api/v1/auth/login", json={"email": email, "password": "SecurePass1"})
    token = login.json()["access_token"]
    proposal = await async_client.post(
        "/api/v1/proposals", json={"title": "Test Proposal"},
        headers={"Authorization": f"Bearer {token}"},
    )
    proposal_id = proposal.json()["id"]
    yield async_client, db_session, token, proposal_id
```

### Test Coverage Alignment (from test-design-epic-07.md)

| Epic Test ID | Priority | Scenario | Test Method |
|-------------|----------|----------|-------------|
| **E07-P0-005** | P0 | Company B JWT → `GET /proposals/:a_id/versions` + rollback endpoint → 404 | `TestAC2ListVersions.test_get_list_cross_company_returns_404`, `TestAC5Rollback.test_rollback_cross_company_returns_404` |
| **E07-P0-008** | P0 | Rollback to version 1 → new version 4 created; versions 2+3 preserved; `current_version_id` = version 4 | `TestAC5Rollback.test_rollback_creates_new_version_from_target_content` |
| **E07-P0-009** | P0 | Diff accuracy: intro=changed, body=unchanged, conclusion=added | `TestAC4DiffVersions.test_diff_section_added_changed_unchanged` |
| **E07-P1-005** | P1 | Version snapshot: version_number = previous + 1, content copied, change_summary stored | `TestAC1CreateVersion.test_post_creates_version_with_change_summary_returns_201`, `.test_post_increments_version_number`, `.test_post_snapshots_current_content` |
| **E07-P1-006** | P1 | Version list newest first | `TestAC2ListVersions.test_get_list_returns_all_versions_newest_first` |
| **E07-P2-002** | P2 | Sequential non-repeating version numbers; rollback continues sequence | `TestVersionNumberIntegrity.test_version_numbers_are_sequential_no_gaps`, `.test_rollback_continues_sequence` |
| **E07-P2-003** | P2 | Diff on identical versions → all-unchanged | `TestAC4DiffVersions.test_diff_identical_versions_all_unchanged` |
| **E07-P2-004** | P2 | Concurrent rollback + auto-save → consistent state (E07-R-006) | `TestVersionNumberIntegrity.test_concurrent_version_creation_no_duplicate_numbers` |

### Note on E07-P0-005 Scope

The full E07-P0-005 test requires Company B JWT to return 404 for **5 endpoint types**: GET detail, PATCH, DELETE (covered in S07.02 `test_proposals.py`), **GET versions** (this story), and POST generate (S07.05). This story adds the RLS test for the `GET /versions` and rollback endpoints.

### Running Tests

```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_proposal_versions.py -v
```

Same `pytest-asyncio`, `httpx`, `pytest-asyncio` stack as `test_proposals.py`.

### Relevant Existing Files (Pattern Reference)

| File | Why Relevant |
|------|-------------|
| `services/client-api/src/client_api/services/proposal_service.py` | Existing service: `_assert_company_owns_proposal`, session flush/refresh pattern, structlog |
| `services/client-api/src/client_api/api/v1/proposals.py` | Existing router: `APIRouter`, `require_role`, `get_current_user`, `Depends` pattern |
| `services/client-api/tests/api/test_proposals.py` | `_register_and_login_second_company` helper, `proposal_client_and_session` fixture |
| `services/client-api/src/client_api/schemas/proposals.py` | Schema patterns: `ConfigDict(from_attributes=True)`, `ProposalResponse` inheritance |
| `services/client-api/src/client_api/models/proposal_version.py` | `ProposalVersion` ORM fields reference |
| `eusolicit-docs/test-artifacts/test-design-epic-07.md` | E07-P0-005, E07-P0-008, E07-P0-009, E07-R-002, E07-R-003, E07-R-006 risk mitigations |

### ProposalVersion ORM Fields Reference

From migration 020 and `models/proposal_version.py`:
- `id: UUID` (PK, gen_random_uuid)
- `proposal_id: UUID` (FK → proposals.id ON DELETE CASCADE)
- `version_number: int` (NOT NULL; UNIQUE together with proposal_id)
- `content: dict` (JSONB NOT NULL DEFAULT '{}'; shape: `{"sections": [{"key": str, "title": str, "body": str}]}`)
- `created_by: UUID | None` (soft ref to user, no FK)
- `change_summary: str | None`
- `created_at: datetime` (NOT NULL DEFAULT now())

**No `updated_at` column on `proposal_versions`** — versions are immutable once created. Only `proposals` has an `updated_at`.

### Project Structure Notes

- Service functions go in `proposal_service.py` (consistent with ESPD pattern — all related operations in one service file)
- Versioning schemas in a separate file `proposal_versions.py` (schemas split by domain to keep files manageable)
- Tests in a new `test_proposal_versions.py` (one test file per API router concern)
- No changes to `main.py` — the versioning routes are sub-paths of the existing proposals router which is already registered

### References

- [Source: eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md#S07.03] — story definition and endpoint spec
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-005] — RLS cross-company test requirements
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-008] — rollback correctness test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-009] — diff accuracy test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-003] — RLS bypass risk and mitigation
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-006] — rollback concurrent race risk and SELECT FOR UPDATE mitigation
- [Source: eusolicit-docs/implementation-artifacts/7-2-proposal-crud-api.md] — patterns: `_assert_company_owns_proposal`, session flush, role guards, test fixtures

## Dev Agent Record

### Agent Model Used

claude-opus-4-5

### Debug Log References

None — all issues resolved iteratively during implementation.

### Completion Notes List

1. **Pre-written ATDD test file existed** (`test_proposal_versions.py`) with two categories of bugs that required fixing before tests could pass:
   - **SQL `::uuid` cast syntax** — asyncpg/SQLAlchemy `text()` doesn't support PostgreSQL `::uuid` cast on named parameters (`:param::uuid`). Fixed throughout `_seed_version_with_content` helper and `test_post_snapshots_current_content` by using `CAST(:param AS uuid)` instead.
   - **Concurrent session access** — `asyncio.gather` with shared SQLAlchemy async session causes `InterfaceError: cannot perform operation: another operation is in progress`. Fixed by adding per-proposal `asyncio.Lock` registry (`_version_write_locks`) to `proposal_service.py`, serialising `create_version` and `rollback_version` calls for the same proposal.

2. **SQLAlchemy uuid4 default timing** — The `default=uuid4` column default is applied at flush time, NOT at object construction or `session.add()` time. `create_version` and `rollback_version` use the proven two-phase flush pattern from `create_proposal`: first flush INSERTs the version row (confirming the DB-generated id), then sets `proposal.current_version_id = version.id` and second flush UPDATEs the proposal. The asyncio lock prevents concurrent "Session is already flushing" errors between the two flushes.

3. **FastAPI route registration order** — `/versions/diff` is registered BEFORE `/{version_id}` in the proposals router, as documented in the story spec. Confirmed working by diff-route tests passing.

4. **No regressions** — All 66 existing Story 7.2 tests (`test_proposals.py` + `test_proposals_extended.py`) continue to pass alongside the 32 new versioning tests (98 total).

### File List

- **Created**: `services/client-api/src/client_api/schemas/proposal_versions.py`
- **Modified**: `services/client-api/src/client_api/services/proposal_service.py` (added versioning functions + asyncio lock registry)
- **Modified**: `services/client-api/src/client_api/api/v1/proposals.py` (added 5 versioning endpoints)
- **Modified**: `services/client-api/tests/api/test_proposal_versions.py` (removed skip decorators; fixed SQL `::uuid` → `CAST(... AS uuid)` in 3 SQL text calls)
