# Story 7.4: Proposal Content Save API (Auto-save + Full Save)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager or company admin**,
I want section-level auto-save (PATCH) and full-proposal save (PUT) that update the current draft content with optimistic locking,
so that my edits are persisted reliably in real time and concurrent writes from stale clients are detected and rejected with the latest content for reconciliation.

## Acceptance Criteria

1. **AC1** — `PATCH /api/v1/proposals/{proposal_id}/content/sections/{section_key}` updates only the matching section's `body` field inside the current version's JSONB `content`. The client sends `body` (new body text) and `content_hash` (md5 of the entire current `content` JSONB as last received from the server). The server computes the current content hash atomically. If hashes match, the section body is updated in-place within `proposal_versions.content`. `proposals.updated_at` is advanced to `now()`. Returns HTTP 200 with the updated `content` dict. Returns HTTP 404 if section_key does not exist in the current version. Returns HTTP 404 if proposal is cross-company or non-existent. Returns HTTP 409 with `{"error": "version_conflict", "latest_content": <current_content>}` if hashes do not match. Requires `bid_manager` role (permits admin and bid_manager).

2. **AC2** — `PUT /api/v1/proposals/{proposal_id}/content` replaces the entire `sections` array in the current version's JSONB `content`. The client sends `sections` (full list of section objects) and `content_hash` (md5 of entire current content). If hashes match, `proposal_versions.content` is replaced with `{"sections": <new_sections>}`. `proposals.updated_at` is advanced to `now()`. Returns HTTP 200 with the full updated `content` dict. Returns HTTP 404 if proposal is cross-company or non-existent. Returns HTTP 409 with `{"error": "version_conflict", "latest_content": <current_content>}` if hashes do not match. Requires `bid_manager` role.

3. **AC3** — Optimistic locking is implemented as a **single atomic DB operation**: `SELECT...FOR UPDATE` on the `proposal_versions` row (identified via `proposals.current_version_id`) acquires a row-level lock; within that same transaction the server computes `md5(content::text)` and compares with `content_hash`; if matched, applies the update (`jsonb_set` for PATCH, full JSON replacement for PUT) and commits; rows-affected = 0 from the lock means another concurrent request is in progress — return 409. The hash must be computed from the **entire** JSONB content object (all sections), never from a single section body only. No split SELECT + UPDATE (two round-trips).

4. **AC4** — Concurrent PATCH race: if two concurrent PATCH requests arrive with the same `content_hash`, exactly one returns HTTP 200 (the first to acquire the lock) and the second returns HTTP 409 with `latest_content` reflecting the winner's updated content. No silent data loss occurs — the section body from the losing write is not applied. Verified by `asyncio.gather` test with testcontainers PostgreSQL.

5. **AC5** — All cross-company or non-existent proposal accesses return HTTP 404 (never 403). The `_assert_company_owns_proposal` helper from `proposal_service.py` is reused for all access checks. `company_id` is derived exclusively from the JWT `organization_id` claim — never from request body or path.

6. **AC6** — Integration tests pass covering: PATCH nominal (section updated, other sections intact, updated_at advanced), PUT nominal (all sections replaced, updated_at advanced), PATCH 404 for missing section_key, PATCH 409 on stale hash with latest_content in response body, PUT 409 on stale hash, concurrent PATCH race (asyncio.gather), cross-company 404 for both endpoints. Tests located at `services/client-api/tests/api/test_proposal_content_save.py`.

## Tasks / Subtasks

- [x] Task 1: Create `src/client_api/schemas/proposal_content.py` (AC: 1–5)
  - [x] 1.1 Add `from __future__ import annotations` header
  - [x] 1.2 Define `SectionItem(BaseModel)` with `key: str`, `title: str`, `body: str`
  - [x] 1.3 Define `SectionAutoSaveRequest(BaseModel)` with `body: str`, `content_hash: str`
  - [x] 1.4 Define `FullSaveRequest(BaseModel)` with `sections: list[SectionItem]`, `content_hash: str`
  - [x] 1.5 Define `ContentSaveResponse(BaseModel)` with `content: dict` (mirrors `{"sections": [...]}` shape)
  - [x] 1.6 Define `ContentConflictDetail(BaseModel)` with `error: Literal["version_conflict"]`, `latest_content: dict`

- [x] Task 2: Add content-save functions to `src/client_api/services/proposal_service.py` (AC: 1–5)
  - [x] 2.1 Implement `_get_current_version_for_update(proposal_id, current_user, session) -> ProposalVersion` — SELECT proposal via `_assert_company_owns_proposal`, then SELECT proposal_version WHERE id = proposal.current_version_id FOR UPDATE (row-level lock)
  - [x] 2.2 Implement `_compute_content_hash(content: dict) -> str` — returns `hashlib.md5(json.dumps(content, sort_keys=True, separators=(',', ':')).encode()).hexdigest()` (deterministic serialization); alternative to raw SQL `md5(content::text)` — **use this Python implementation throughout**; document the exact serialization so frontend can reproduce the hash
  - [x] 2.3 Implement `auto_save_section(proposal_id, section_key, body, content_hash, current_user, session) -> dict` — lock version, check hash, find section index, apply jsonb_set update via Python dict mutation + flush, update `proposal.updated_at`, return updated content
  - [x] 2.4 Implement `full_save_content(proposal_id, sections, content_hash, current_user, session) -> dict` — lock version, check hash, replace entire content dict, flush, update `proposal.updated_at`, return updated content
  - [x] 2.5 Raise `HTTPException(status_code=409, detail={"error": "version_conflict", "latest_content": current_content})` on hash mismatch in both functions
  - [x] 2.6 Raise `HTTPException(status_code=404, detail="Section not found")` if `section_key` not in current version sections (PATCH only)

- [x] Task 3: Add content-save endpoints to `src/client_api/api/v1/proposals.py` (AC: 1–5)
  - [x] 3.1 Import schemas from `proposal_content`: `SectionAutoSaveRequest`, `FullSaveRequest`, `ContentSaveResponse`
  - [x] 3.2 Add `PATCH /{proposal_id}/content/sections/{section_key}` → `auto_save_section` (200, bid_manager)
  - [x] 3.3 Add `PUT /{proposal_id}/content` → `full_save_content` (200, bid_manager)
  - [x] 3.4 Both endpoints set `responses={409: {"model": ContentConflictDetail}}` in decorator for OpenAPI docs
  - [x] 3.5 Verify route ordering: new `/content` routes do not conflict with existing `/versions` and CRUD routes

- [x] Task 4: Integration tests `tests/api/test_proposal_content_save.py` (AC: 1–6)
  - [x] 4.1 Reuse `proposal_and_token` fixture pattern from `test_proposal_versions.py` — register, verify email, login, create proposal
  - [x] 4.2 Add `_get_initial_content_hash(client, token, proposal_id)` helper — GET proposal detail, compute hash from `current_version.content`
  - [x] 4.3 `TestAC1SectionAutoSave` — test_patch_updates_only_target_section, test_patch_advances_updated_at, test_patch_returns_full_content, test_patch_missing_section_key_returns_404, test_patch_stale_hash_returns_409_with_latest_content, test_patch_cross_company_returns_404 (6 tests)
  - [x] 4.4 `TestAC2FullSave` — test_put_replaces_all_sections, test_put_advances_updated_at, test_put_stale_hash_returns_409_with_latest_content, test_put_cross_company_returns_404 (4 tests)
  - [x] 4.5 `TestConcurrentWrites` — test_concurrent_patch_race_exactly_one_200_one_409 (E07-P0-003), test_stale_hash_full_save_after_concurrent_patch (E07-P0-004) (2 tests)
  - [x] 4.6 All tests use `asyncio.gather` (NOT `asyncio.sleep`) for concurrency tests; testcontainers PostgreSQL only (NOT SQLite)

## Dev Notes

### Context: Built on Stories 7.1, 7.2, 7.3

**Story 7.1** (done) created:
- Migration 020: `client.proposals`, `client.proposal_versions`, `client.content_blocks` tables
- `ProposalVersion` ORM: `id`, `proposal_id`, `version_number`, `content` (JSONB, shape `{"sections": [{"key": str, "title": str, "body": str}]}`), `created_by`, `change_summary`, `created_at`
- **No `updated_at` on `proposal_versions`** — versions are immutable snapshots. `proposals.updated_at` is the field to advance.
- `ProposalFactory`, `ProposalVersionFactory` factories in `eusolicit-test-utils`

**Story 7.2** (done) created:
- `services/client-api/src/client_api/schemas/proposals.py` — `ProposalResponse`, `ProposalDetailResponse`, `ProposalCreateRequest`, `ProposalPatchRequest`
- `services/client-api/src/client_api/services/proposal_service.py` — `_assert_company_owns_proposal`, `create_proposal`, `list_proposals`, `get_proposal`, `update_proposal`, `archive_proposal`
- `services/client-api/src/client_api/api/v1/proposals.py` — 5 CRUD endpoints registered under `/proposals`
- `services/client-api/tests/api/test_proposals.py` + `test_proposals_extended.py` (66 tests total, must stay green)

**Story 7.3** (done) added to `proposal_service.py`:
- `_get_proposal_for_update` (SELECT FOR UPDATE on proposal row)
- `_next_version_number`, `create_version`, `list_versions`, `get_version`, `diff_versions`, `rollback_version`
- `_version_write_locks: dict[UUID, asyncio.Lock]` — per-proposal asyncio lock registry for version number sequencing
- Added 5 versioning endpoints to `proposals.py` router (107 cumulative proposal tests passing)
- Schema file: `services/client-api/src/client_api/schemas/proposal_versions.py`

This story adds **2 new endpoints** to the existing proposals router. No new router registration in `main.py` needed.

### Existing Infrastructure — DO NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/alembic/versions/020_proposal_workspace_schema.py` | Migration with all 3 tables + indexes | **DO NOT TOUCH** |
| `services/client-api/src/client_api/models/proposal.py` | `Proposal` ORM with `updated_at`, `current_version_id` | **DO NOT TOUCH** — read-only reference |
| `services/client-api/src/client_api/models/proposal_version.py` | `ProposalVersion` ORM (no `updated_at`) | **DO NOT TOUCH** — read-only reference |
| `services/client-api/src/client_api/services/proposal_service.py` | S07.02+S07.03 functions + `_assert_company_owns_proposal` + `_version_write_locks` | **ADD new functions only** — do not remove or rename existing |
| `services/client-api/src/client_api/api/v1/proposals.py` | S07.02+S07.03 router (7 endpoints) | **ADD 2 new endpoints only** |
| `services/client-api/src/client_api/core/security.py` | `require_role`, `get_current_user`, `CurrentUser` | **Reuse unchanged** |
| `services/client-api/src/client_api/dependencies.py` | `get_db_session` | **Reuse unchanged** |
| `services/client-api/tests/api/test_proposals.py` | 66 tests — **must not regress** | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_versions.py` | 32 tests — **must not regress** | **DO NOT MODIFY** |

### New Files to Create

| File | Purpose |
|------|---------|
| `services/client-api/src/client_api/schemas/proposal_content.py` | Pydantic schemas for content-save endpoints |
| `services/client-api/tests/api/test_proposal_content_save.py` | Integration tests (all 12 tests) |

### Files to Modify

| File | Change |
|------|--------|
| `services/client-api/src/client_api/services/proposal_service.py` | Add `_get_current_version_for_update`, `_compute_content_hash`, `auto_save_section`, `full_save_content` |
| `services/client-api/src/client_api/api/v1/proposals.py` | Add 2 new route handlers; import from `proposal_content` schemas |

### New API Endpoint URLs

```
PATCH  /api/v1/proposals/{proposal_id}/content/sections/{section_key}   → single-section auto-save (bid_manager)
PUT    /api/v1/proposals/{proposal_id}/content                           → full content replace (bid_manager)
```

### CRITICAL: Optimistic Locking Implementation (E07-R-002)

**The core risk (Score 6):** A split SELECT-then-UPDATE (two round-trips) allows two concurrent PATCHes reading the same hash to both pass the check and both write — last-write-wins with silent data loss.

**Required implementation — SELECT FOR UPDATE pattern:**

```python
async def _get_current_version_for_update(
    proposal_id: UUID,
    current_user: CurrentUser,
    session: AsyncSession,
) -> tuple[Proposal, ProposalVersion]:
    """Load proposal + current version with a row-level WRITE LOCK on the version row."""
    from sqlalchemy import select

    # 1. Load and assert company ownership on the proposal
    stmt_proposal = select(Proposal).where(Proposal.id == proposal_id)
    result = await session.execute(stmt_proposal)
    proposal = result.scalar_one_or_none()
    if proposal is None:
        raise HTTPException(status_code=404, detail="Proposal not found")
    _assert_company_owns_proposal(proposal, current_user)  # raises 404 if cross-company

    # 2. Acquire row-level lock on the current version row
    stmt_version = (
        select(ProposalVersion)
        .where(ProposalVersion.id == proposal.current_version_id)
        .with_for_update()  # ← ROW-LEVEL LOCK prevents concurrent hash-check races
    )
    result = await session.execute(stmt_version)
    version = result.scalar_one_or_none()
    if version is None:
        raise HTTPException(status_code=404, detail="Current version not found")

    return proposal, version
```

**Why this is atomic:** `SELECT...FOR UPDATE` within the same SQLAlchemy async transaction holds the lock until the transaction commits (via `get_db_session` dependency). Any concurrent request calling `_get_current_version_for_update` for the same version_id will block at the `with_for_update()` until the first transaction completes.

### Content Hash — Deterministic Python Implementation

Do NOT rely on `md5(content::text)` from PostgreSQL — the JSON serialization ordering in PostgreSQL JSONB may differ from what the client expects. Use a deterministic Python hash:

```python
import hashlib
import json

def _compute_content_hash(content: dict) -> str:
    """Deterministic hash of proposal content dict. Must match frontend implementation."""
    serialized = json.dumps(content, sort_keys=True, separators=(',', ':'), ensure_ascii=False)
    return hashlib.md5(serialized.encode('utf-8')).hexdigest()
```

**Frontend contract:** The frontend must compute the hash identically — `JSON.stringify` with sorted keys + md5. Document the exact serialization in the API response (e.g., include `content_hash` field in `ContentSaveResponse`) so the frontend always has the latest hash without recomputing.

### Section Auto-Save Implementation

```python
async def auto_save_section(
    proposal_id: UUID,
    section_key: str,
    body: str,
    content_hash: str,
    current_user: CurrentUser,
    session: AsyncSession,
) -> dict:
    """PATCH: update one section body in the current version. Returns updated content."""
    from datetime import UTC, datetime

    proposal, version = await _get_current_version_for_update(proposal_id, current_user, session)

    # Hash check (within the lock)
    server_hash = _compute_content_hash(version.content)
    if server_hash != content_hash:
        raise HTTPException(
            status_code=409,
            detail={"error": "version_conflict", "latest_content": version.content},
        )

    # Find section index
    sections = version.content.get("sections", [])
    section_idx = next((i for i, s in enumerate(sections) if s["key"] == section_key), None)
    if section_idx is None:
        raise HTTPException(status_code=404, detail=f"Section '{section_key}' not found")

    # Mutate a copy (important: SQLAlchemy JSONB tracks object identity, not deep equality)
    import copy
    new_content = copy.deepcopy(version.content)
    new_content["sections"][section_idx]["body"] = body

    # Persist
    version.content = new_content
    proposal.updated_at = datetime.now(UTC)
    await session.flush()
    await session.refresh(version)
    await session.refresh(proposal)

    log.info("auto_save_section.success", proposal_id=str(proposal_id), section_key=section_key)
    return version.content
```

**CRITICAL — JSONB mutation gotcha (from S07.03 learnings):** SQLAlchemy tracks JSONB column changes by object identity. If you mutate the dict in-place (`version.content["sections"][0]["body"] = body`), SQLAlchemy may NOT detect the change (dirty tracking not triggered for nested dict mutations). **Always reassign `version.content` to a new dict object** — use `copy.deepcopy` or construct a fresh dict, then assign:
```python
version.content = new_content  # ← must reassign the column attribute entirely
```

### Full Save Implementation

```python
async def full_save_content(
    proposal_id: UUID,
    sections: list[dict],
    content_hash: str,
    current_user: CurrentUser,
    session: AsyncSession,
) -> dict:
    """PUT: replace all sections in the current version. Returns updated content."""
    from datetime import UTC, datetime

    proposal, version = await _get_current_version_for_update(proposal_id, current_user, session)

    server_hash = _compute_content_hash(version.content)
    if server_hash != content_hash:
        raise HTTPException(
            status_code=409,
            detail={"error": "version_conflict", "latest_content": version.content},
        )

    new_content = {"sections": sections}
    version.content = new_content
    proposal.updated_at = datetime.now(UTC)
    await session.flush()
    await session.refresh(version)
    await session.refresh(proposal)

    log.info("full_save_content.success", proposal_id=str(proposal_id), section_count=len(sections))
    return version.content
```

### Schema File — `proposal_content.py`

```python
from __future__ import annotations

from typing import Literal
from uuid import UUID

from pydantic import BaseModel


class SectionItem(BaseModel):
    key: str
    title: str
    body: str


class SectionAutoSaveRequest(BaseModel):
    body: str
    content_hash: str  # md5 of entire current content (deterministic JSON, sort_keys=True)


class FullSaveRequest(BaseModel):
    sections: list[SectionItem]
    content_hash: str  # md5 of entire current content


class ContentSaveResponse(BaseModel):
    content: dict        # updated content dict: {"sections": [...]}
    content_hash: str    # NEW hash of the updated content — client must store this for next save


class ContentConflictDetail(BaseModel):
    error: Literal["version_conflict"]
    latest_content: dict  # latest server content for client reconciliation
```

**Note:** Include `content_hash` in `ContentSaveResponse` so the frontend always has the latest server-side hash without needing to recompute it. This eliminates hash drift from encoding differences.

### Router Endpoints — `proposals.py`

```python
from client_api.schemas.proposal_content import (
    SectionAutoSaveRequest, FullSaveRequest, ContentSaveResponse, ContentConflictDetail
)

@router.patch(
    "/{proposal_id}/content/sections/{section_key}",
    response_model=ContentSaveResponse,
    responses={409: {"model": ContentConflictDetail}},
    status_code=200,
)
async def patch_section(
    proposal_id: UUID,
    section_key: str,
    payload: SectionAutoSaveRequest,
    session: Annotated[AsyncSession, Depends(get_db_session)],
    current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
) -> ContentSaveResponse:
    content = await proposal_service.auto_save_section(
        proposal_id, section_key, payload.body, payload.content_hash, current_user, session
    )
    return ContentSaveResponse(content=content, content_hash=_compute_content_hash(content))


@router.put(
    "/{proposal_id}/content",
    response_model=ContentSaveResponse,
    responses={409: {"model": ContentConflictDetail}},
    status_code=200,
)
async def put_content(
    proposal_id: UUID,
    payload: FullSaveRequest,
    session: Annotated[AsyncSession, Depends(get_db_session)],
    current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
) -> ContentSaveResponse:
    content = await proposal_service.full_save_content(
        proposal_id, [s.model_dump() for s in payload.sections], payload.content_hash, current_user, session
    )
    return ContentSaveResponse(content=content, content_hash=_compute_content_hash(content))
```

**Import `_compute_content_hash` into the router** so it can compute the response hash, OR expose it from `proposal_service.py` as a public helper.

### Test File Structure

```python
# tests/api/test_proposal_content_save.py
from __future__ import annotations
import asyncio, copy, hashlib, json, uuid
from sqlalchemy import text
import pytest, pytest_asyncio

def _hash(content: dict) -> str:
    return hashlib.md5(
        json.dumps(content, sort_keys=True, separators=(',', ':')).encode()
    ).hexdigest()

async def _seed_proposal_with_sections(client, db_session, token, proposal_id, sections):
    """Helper: directly UPDATE proposal_versions.content for test setup."""
    content = {"sections": sections}
    await db_session.execute(
        text("""
            UPDATE client.proposal_versions pv
            SET content = CAST(:content AS jsonb)
            FROM client.proposals p
            WHERE p.id = CAST(:proposal_id AS uuid)
              AND pv.id = p.current_version_id
        """),
        {"content": json.dumps(content), "proposal_id": str(proposal_id)},
    )
    await db_session.flush()
    return content, _hash(content)

@pytest_asyncio.fixture
async def proposal_and_token(async_client, db_session):
    """Register → verify → login → create proposal with 2 sections."""
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
    login = await async_client.post("/api/v1/auth/login",
                                    json={"email": email, "password": "SecurePass1"})
    token = login.json()["access_token"]
    proposal = await async_client.post(
        "/api/v1/proposals", json={"title": "Test Proposal"},
        headers={"Authorization": f"Bearer {token}"},
    )
    proposal_id = proposal.json()["id"]

    # Seed with 2 sections
    initial_sections = [
        {"key": "intro", "title": "Introduction", "body": "Hello World"},
        {"key": "body",  "title": "Main Body",    "body": "Original body text"},
    ]
    content, content_hash = await _seed_proposal_with_sections(
        async_client, db_session, token, proposal_id, initial_sections
    )
    yield async_client, db_session, token, proposal_id, content, content_hash
```

### Concurrency Test Pattern (E07-P0-003)

```python
async def test_concurrent_patch_race_exactly_one_200_one_409(proposal_and_token):
    client, db_session, token, proposal_id, content, content_hash = proposal_and_token
    headers = {"Authorization": f"Bearer {token}"}
    url = f"/api/v1/proposals/{proposal_id}/content/sections/intro"

    async def do_patch(body_text: str):
        return await client.patch(url, json={"body": body_text, "content_hash": content_hash},
                                  headers=headers)

    responses = await asyncio.gather(do_patch("Edit A"), do_patch("Edit B"))
    statuses = sorted(r.status_code for r in responses)
    assert statuses == [200, 409], f"Expected [200, 409], got {statuses}"

    # 409 response must contain latest_content for reconciliation
    conflict_response = next(r for r in responses if r.status_code == 409)
    detail = conflict_response.json()
    assert detail["error"] == "version_conflict"
    assert "latest_content" in detail
    assert detail["latest_content"]["sections"][0]["key"] == "intro"
```

### Route Ordering — No Conflict with Existing Routes

The two new routes (`/content/sections/{section_key}` and `/content`) are sub-paths of `/{proposal_id}` that start with `/content`. Existing routes are:
- `/` (list/create)
- `/{proposal_id}` (detail/patch/delete)
- `/{proposal_id}/versions` and sub-routes

The `/content` path segment is new and does not conflict. No special ordering is required.

### Architecture Constraints (MUST FOLLOW — from S07.03 learnings)

1. **`from __future__ import annotations`** at top of every new file (project-wide rule)
2. **`structlog` for all logging:**
   ```python
   import structlog
   log = structlog.get_logger()
   ```
3. **`session.flush()` then `session.refresh()` — never `session.commit()`** — the session is managed by `get_db_session` dependency
4. **`company_id` exclusively from JWT** — `_assert_company_owns_proposal` handles all ownership checks
5. **Return 404 (not 403) for cross-company access** — prevents UUID enumeration (E07-R-003)
6. **`require_role("bid_manager")`** for write operations (permits admin + bid_manager)
7. **JSONB mutation requires full column reassignment** — `version.content = new_dict` (never mutate in-place without reassigning the attribute; SQLAlchemy dirty tracking on JSONB requires identity change)
8. **Use `CAST(:param AS uuid)` in `text()` SQL** — never `::uuid` cast syntax (asyncpg incompatibility, discovered in S07.03)

### Test Coverage Alignment (from test-design-epic-07.md)

| Epic Test ID | Priority | Scenario | Implemented By |
|-------------|----------|----------|----------------|
| **E07-P0-003** | P0 | Concurrent PATCH race → exactly one 200, one 409; no silent data loss; 409 has `latest_content` | `TestConcurrentWrites.test_concurrent_patch_race_exactly_one_200_one_409` |
| **E07-P0-004** | P0 | Stale hash full-save: externally PATCH advances content; PUT with old hash → 409; DB not overwritten | `TestConcurrentWrites.test_stale_hash_full_save_after_concurrent_patch` |
| **E07-P1-007** | P1 | PATCH updates only `intro` section; `body` section unchanged in DB | `TestAC1SectionAutoSave.test_patch_updates_only_target_section` |
| **E07-P1-008** | P1 | Successful PATCH advances `proposals.updated_at` timestamp | `TestAC1SectionAutoSave.test_patch_advances_updated_at` |
| **E07-P0-005** | P0 | Company B JWT → both endpoints return 404 | `TestAC1SectionAutoSave.test_patch_cross_company_returns_404`, `TestAC2FullSave.test_put_cross_company_returns_404` |
| **E07-P2-001** | P2 | Non-existent proposal UUID → 404 | `TestAC1SectionAutoSave.test_patch_missing_section_key_returns_404`, `TestAC2FullSave.test_put_cross_company_returns_404` |

**E07-P0-003/004 must use testcontainers PostgreSQL** (already in `conftest.py` as `db_session` + `async_client`) — SQLite or SQLAlchemy mock cannot reproduce real MVCC locking behaviour.

### Test for Updated_At Advancement

```python
async def test_patch_advances_updated_at(proposal_and_token, db_session):
    client, _, token, proposal_id, content, content_hash = proposal_and_token

    # Record updated_at before PATCH
    row = await db_session.execute(
        text("SELECT updated_at FROM client.proposals WHERE id = CAST(:id AS uuid)"),
        {"id": str(proposal_id)},
    )
    before = row.scalar_one()

    resp = await client.patch(
        f"/api/v1/proposals/{proposal_id}/content/sections/intro",
        json={"body": "Updated body", "content_hash": content_hash},
        headers={"Authorization": f"Bearer {token}"},
    )
    assert resp.status_code == 200

    row = await db_session.execute(
        text("SELECT updated_at FROM client.proposals WHERE id = CAST(:id AS uuid)"),
        {"id": str(proposal_id)},
    )
    after = row.scalar_one()
    assert after > before
```

### Running Tests

```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_proposal_content_save.py -v
# Full regression check (must stay green — 98 existing tests):
pytest tests/api/test_proposals.py tests/api/test_proposal_versions.py tests/api/test_proposal_content_save.py -v
```

### Project Structure Notes

- New schemas in `proposal_content.py` (separate domain file, consistent with `proposal_versions.py` from S07.03)
- Service functions added to `proposal_service.py` (single service file for all proposal concerns — consistent with ESPD pattern)
- Tests in new `test_proposal_content_save.py` (one test file per router concern)
- No changes to `main.py` — new endpoints are sub-paths of the existing proposals router already registered

### References

- [Source: eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md#S07.04] — story definition and endpoint spec
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-003] — concurrent PATCH race test requirement
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-004] — stale hash full-save test requirement
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-002] — optimistic locking race risk + atomic SQL mitigation
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-003] — RLS bypass risk; 404 not 403 requirement
- [Source: eusolicit-docs/implementation-artifacts/7-3-proposal-versioning-api.md] — JSONB, asyncio lock, flush/refresh, `CAST(... AS uuid)` patterns
- [Source: eusolicit-docs/implementation-artifacts/7-2-proposal-crud-api.md] — `_assert_company_owns_proposal`, role guard patterns

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

None required — all tests passed after two minor implementation corrections.

### Completion Notes List

1. **ensure_ascii alignment**: Story spec showed `_compute_content_hash` with `ensure_ascii=False`, but the pre-written ATDD test's `_hash()` helper uses default (`ensure_ascii=True`). Removed `ensure_ascii=False` from the server implementation to align with the test contract. This means non-ASCII characters (e.g. `€`) are encoded as `\uXXXX` in the hash computation. Frontend must match this behavior.

2. **409 response body**: FastAPI wraps `HTTPException.detail` in `{"detail": ...}` automatically. Tests expect the conflict body directly (`{"error": "version_conflict", "latest_content": ...}`). Router endpoints now catch the 409 `HTTPException` from the service and return a `JSONResponse` with the raw detail dict (no wrapper).

3. **Concurrent race test (E07-P0-003)**: Passes with `asyncio.gather` + real PostgreSQL `SELECT FOR UPDATE` — exactly one 200, one 409, no silent data loss confirmed.

### File List

- `eusolicit-app/services/client-api/src/client_api/schemas/proposal_content.py` ← NEW
- `eusolicit-app/services/client-api/src/client_api/services/proposal_service.py` ← MODIFIED (added Story 7.4 functions)
- `eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py` ← MODIFIED (added 2 endpoints)
- `eusolicit-app/services/client-api/tests/api/test_proposal_content_save.py` ← MODIFIED (removed @pytest.mark.skip)
