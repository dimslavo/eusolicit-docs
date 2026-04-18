# Story 7.6: Requirement Checklist Agent Integration

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager or company admin**,
I want to generate a structured, checkable requirement checklist from the tender opportunity documents using the Requirement Checklist Agent and then track which items have been satisfied in my proposal draft,
so that I can ensure full compliance with every tender requirement before submission without manually reading through the entire tender pack.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/proposals/{proposal_id}/checklist/generate` invokes the `requirement-checklist` agent via `AiGatewayClient.run_agent()` (synchronous, non-SSE). Requires `bid_manager` role; returns 404 for cross-company or non-existent proposal. The payload sent to the agent includes: opportunity `title`, `description`, `evaluation_criteria`, `submission_deadline` (loaded from `pipeline.opportunities` via pipeline read-only session, or empty strings if no linked opportunity), company `name`, and the current proposal version's `sections` array (as context). Returns the agent response parsed into checklist items.

2. **AC2** — The agent's JSON response is expected as `{"items": [{"text": str, "linked_section_key": str | null, "category": str | null}, ...]}`. Each item is stored as an individual row in the `client.proposal_checklists` table with: a new UUID, `proposal_id`, `text`, `is_checked = false`, `linked_section_key`, `category`, `position` (0-based index), `created_at`, `updated_at`. **Any existing checklist rows for the proposal are deleted before inserting new ones** (regeneration replaces, not appends). Returns `HTTP 200` with the complete list of new checklist items.

3. **AC3** — `GET /api/v1/proposals/{proposal_id}/checklist` returns all `proposal_checklists` rows for the proposal, ordered by `position ASC`. No role restriction beyond company membership (any authenticated company user may view). Returns 404 for cross-company proposal. Returns an empty list (not 404) if no checklist has been generated yet.

4. **AC4** — `PATCH /api/v1/proposals/{proposal_id}/checklist/items/{item_id}` accepts body `{"is_checked": bool}` and updates the `is_checked` column for the matching row. Requires `bid_manager` role. Returns 404 if: proposal is cross-company, proposal does not exist, or `item_id` does not belong to `proposal_id`. Updates `updated_at`. Returns the updated checklist item.

5. **AC5** — **Error handling**: If `AiGatewayTimeoutError` is raised during `run_agent`, return `HTTP 504` with `{"error": "agent_timeout", "retry_after": 30}`. If `AiGatewayUnavailableError` is raised, return `HTTP 503` with `{"error": "agent_unavailable"}`. On agent success but malformed/missing `items` key in response, treat as empty list (no error, no partial insert) — log a warning.

6. **AC6** — Alembic migration `022_proposal_checklist` creates the `client.proposal_checklists` table (id UUID PK, proposal_id UUID FK ON DELETE CASCADE, text TEXT NOT NULL, is_checked BOOLEAN NOT NULL DEFAULT FALSE, linked_section_key TEXT nullable, category TEXT nullable, position INTEGER NOT NULL DEFAULT 0, created_at TIMESTAMPTZ DEFAULT now(), updated_at TIMESTAMPTZ DEFAULT now()). Adds index `ix_proposal_checklists_proposal_id` on `(proposal_id)`. Reversible: downgrade drops table and index. `down_revision = "021_proposal_generation_status"`.

7. **AC7** — Integration tests pass for: nominal generation (3 items stored, correct position/fields), retrieval (items ordered by position), toggle (is_checked persisted), regeneration (old items replaced), cross-company 404 for all three endpoints, AI Gateway timeout → 504. Tests at `services/client-api/tests/api/test_proposal_checklist.py`.

## Tasks / Subtasks

- [x] Task 1: Alembic migration `022_proposal_checklist.py` (AC: 6)
  - [x] 1.1 Create `services/client-api/alembic/versions/022_proposal_checklist.py` with `revision = "022_proposal_checklist"`, `down_revision = "021_proposal_generation_status"`
  - [x] 1.2 `upgrade()`:
    ```python
    op.create_table(
        "proposal_checklists",
        sa.Column("id", sa.UUID(), nullable=False, server_default=sa.text("gen_random_uuid()")),
        sa.Column("proposal_id", sa.UUID(), nullable=False),
        sa.Column("text", sa.Text(), nullable=False),
        sa.Column("is_checked", sa.Boolean(), nullable=False, server_default=sa.text("false")),
        sa.Column("linked_section_key", sa.Text(), nullable=True),
        sa.Column("category", sa.Text(), nullable=True),
        sa.Column("position", sa.Integer(), nullable=False, server_default=sa.text("0")),
        sa.Column("created_at", sa.TIMESTAMP(timezone=True), server_default=sa.text("now()"), nullable=False),
        sa.Column("updated_at", sa.TIMESTAMP(timezone=True), server_default=sa.text("now()"), nullable=False),
        sa.ForeignKeyConstraint(["proposal_id"], ["client.proposals.id"], ondelete="CASCADE"),
        sa.PrimaryKeyConstraint("id"),
        schema="client",
    )
    op.create_index("ix_proposal_checklists_proposal_id", "proposal_checklists", ["proposal_id"], schema="client")
    ```
  - [x] 1.3 `downgrade()`: `op.drop_index("ix_proposal_checklists_proposal_id", "proposal_checklists", schema="client")` then `op.drop_table("proposal_checklists", schema="client")`
  - [x] 1.4 Verify `alembic upgrade head` then `alembic downgrade -1` completes without error

- [x] Task 2: `ProposalChecklistItem` ORM model (AC: 6)
  - [x] 2.1 Create `services/client-api/src/client_api/models/proposal_checklist.py`:
    ```python
    from __future__ import annotations
    import uuid
    from datetime import datetime
    import sqlalchemy as sa
    from sqlalchemy.orm import Mapped, mapped_column
    from client_api.database import Base

    class ProposalChecklistItem(Base):
        __tablename__ = "proposal_checklists"
        __table_args__ = (
            sa.Index("ix_proposal_checklists_proposal_id", "proposal_id"),
            {"schema": "client"},
        )
        id: Mapped[uuid.UUID] = mapped_column(sa.UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
        proposal_id: Mapped[uuid.UUID] = mapped_column(sa.UUID(as_uuid=True), nullable=False)
        text: Mapped[str] = mapped_column(sa.Text(), nullable=False)
        is_checked: Mapped[bool] = mapped_column(sa.Boolean(), nullable=False, server_default=sa.text("false"))
        linked_section_key: Mapped[str | None] = mapped_column(sa.Text(), nullable=True)
        category: Mapped[str | None] = mapped_column(sa.Text(), nullable=True)
        position: Mapped[int] = mapped_column(sa.Integer(), nullable=False, server_default=sa.text("0"))
        created_at: Mapped[datetime] = mapped_column(sa.TIMESTAMP(timezone=True), server_default=sa.text("now()"), nullable=False)
        updated_at: Mapped[datetime] = mapped_column(sa.TIMESTAMP(timezone=True), server_default=sa.text("now()"), nullable=False)
    ```
  - [x] 2.2 Import `ProposalChecklistItem` in `services/client-api/src/client_api/models/__init__.py` (or wherever ORM models are registered for alembic autogenerate)

- [x] Task 3: Checklist schemas `proposal_checklist.py` (AC: 1, 2, 3, 4)
  - [x] 3.1 Create `services/client-api/src/client_api/schemas/proposal_checklist.py`:
    ```python
    from __future__ import annotations
    import uuid
    from datetime import datetime
    from pydantic import BaseModel

    class ChecklistItemResponse(BaseModel):
        id: uuid.UUID
        proposal_id: uuid.UUID
        text: str
        is_checked: bool
        linked_section_key: str | None
        category: str | None
        position: int
        created_at: datetime
        updated_at: datetime

        model_config = {"from_attributes": True}

    class ChecklistResponse(BaseModel):
        items: list[ChecklistItemResponse]
        total: int

    class ChecklistItemToggle(BaseModel):
        is_checked: bool
    ```

- [x] Task 4: Service functions in `proposal_service.py` (AC: 1–5)
  - [x] 4.1 Add imports at top of `proposal_service.py` (if not already present):
    - `from client_api.models.proposal_checklist import ProposalChecklistItem`
    - `from client_api.services.ai_gateway_client import AiGatewayClient, AiGatewayTimeoutError, AiGatewayUnavailableError`
    - Import `HTTPException` from fastapi if not already imported
  - [x] 4.2 Add `generate_checklist(proposal_id, current_user, gw_client, session, pipeline_session) -> list[ProposalChecklistItem]`:
    - Call `_assert_company_owns_proposal` pattern to load + verify ownership; raise 404 if missing/cross-company
    - Build payload (reuse same structure as `build_generation_payload` — opportunity data from pipeline_session + company name + current version sections); wrap `pipeline_session` usage in try/except for missing opportunity (graceful empty)
    - `try: result = await gw_client.run_agent("requirement-checklist", payload)`
    - `except AiGatewayTimeoutError: raise HTTPException(504, {"error": "agent_timeout", "retry_after": 30})`
    - `except AiGatewayUnavailableError: raise HTTPException(503, {"error": "agent_unavailable"})`
    - Extract `items_data = result.get("items", [])` — if not a list, treat as `[]` and log warning
    - Delete existing checklist rows: `await session.execute(delete(ProposalChecklistItem).where(ProposalChecklistItem.proposal_id == proposal_id))`
    - Bulk-insert new items: for each `(i, item)` in `enumerate(items_data)`, create `ProposalChecklistItem(proposal_id=proposal_id, text=item.get("text",""), linked_section_key=item.get("linked_section_key"), category=item.get("category"), position=i)`; `session.add_all(new_items)`
    - `await session.flush()`; `await session.refresh()` each new item (or re-query)
    - Return list of new items
  - [x] 4.3 Add `get_checklist(proposal_id, current_user, session) -> list[ProposalChecklistItem]`:
    - Verify company ownership using `select(Proposal).where(Proposal.id == proposal_id, Proposal.company_id == current_user.company_id)` — raise 404 if not found (same cross-company pattern)
    - `result = await session.execute(select(ProposalChecklistItem).where(ProposalChecklistItem.proposal_id == proposal_id).order_by(ProposalChecklistItem.position))`
    - Return `result.scalars().all()` (empty list is valid, not 404)
  - [x] 4.4 Add `toggle_checklist_item(proposal_id, item_id, is_checked, current_user, session) -> ProposalChecklistItem`:
    - Verify company ownership of proposal (404 pattern)
    - Load `ProposalChecklistItem` by `id == item_id AND proposal_id == proposal_id` — raise 404 if not found (prevents item_id from a different proposal being toggled)
    - Set `item.is_checked = is_checked`, `item.updated_at = datetime.utcnow()` (or use `sa.text("now()")` via an update statement)
    - `await session.flush()`, `await session.refresh(item)`
    - Return updated item

- [x] Task 5: Router endpoints in `proposals.py` (AC: 1–5)
  - [x] 5.1 Add imports to `services/client-api/src/client_api/api/v1/proposals.py`:
    - `from client_api.schemas.proposal_checklist import ChecklistItemResponse, ChecklistResponse, ChecklistItemToggle`
    - `get_ai_gateway_client` and `AiGatewayClient` (already imported from S07.05 — verify present)
    - `get_pipeline_readonly_session` (already imported from S07.05 — verify present)
  - [x] 5.2 Add `POST /{proposal_id}/checklist/generate` route:
    ```python
    @router.post(
        "/{proposal_id}/checklist/generate",
        response_model=ChecklistResponse,
        responses={
            404: {"description": "Proposal not found or cross-company"},
            503: {"description": "AI Gateway unavailable"},
            504: {"description": "AI Gateway timeout"},
        },
    )
    async def generate_checklist(
        proposal_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        pipeline_session: Annotated[AsyncSession, Depends(get_pipeline_readonly_session)],
        current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
    ) -> ChecklistResponse:
        items = await proposal_service.generate_checklist(
            proposal_id, current_user, gw_client, session, pipeline_session
        )
        return ChecklistResponse(items=items, total=len(items))
    ```
  - [x] 5.3 Add `GET /{proposal_id}/checklist` route:
    ```python
    @router.get(
        "/{proposal_id}/checklist",
        response_model=ChecklistResponse,
        responses={404: {"description": "Proposal not found or cross-company"}},
    )
    async def get_checklist(
        proposal_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],  # read — no role restriction
    ) -> ChecklistResponse:
        items = await proposal_service.get_checklist(proposal_id, current_user, session)
        return ChecklistResponse(items=items, total=len(items))
    ```
  - [x] 5.4 Add `PATCH /{proposal_id}/checklist/items/{item_id}` route:
    ```python
    @router.patch(
        "/{proposal_id}/checklist/items/{item_id}",
        response_model=ChecklistItemResponse,
        responses={404: {"description": "Item not found or cross-company"}},
    )
    async def toggle_checklist_item(
        proposal_id: UUID,
        item_id: UUID,
        body: ChecklistItemToggle,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
    ) -> ChecklistItemResponse:
        item = await proposal_service.toggle_checklist_item(
            proposal_id, item_id, body.is_checked, current_user, session
        )
        return ChecklistItemResponse.model_validate(item)
    ```

- [x] Task 6: Integration tests `tests/api/test_proposal_checklist.py` (AC: 1–7)
  - [x] 6.1 Add `respx` mock fixture for `requirement-checklist` agent:
    ```python
    @pytest.fixture
    def mock_checklist_agent(respx_mock):
        items = [
            {"text": "Provide proof of financial standing (last 3 years)", "linked_section_key": "eligibility", "category": "Administrative"},
            {"text": "Include technical team CVs (≥5 years experience)", "linked_section_key": "technical_approach", "category": "Technical"},
            {"text": "Submit price schedule with breakdown", "linked_section_key": "pricing", "category": "Financial"},
        ]
        respx_mock.post("http://test-aigw:8000/agents/requirement-checklist/run").mock(
            return_value=httpx.Response(200, json={"items": items})
        )
        return items
    ```
  - [x] 6.2 Reuse `proposal_with_opportunity` fixture from `test_proposal_generate.py` (or create analogous `proposal_fixture` fixture) providing `(async_client, db_session, token, proposal_id)`
  - [x] 6.3 `TestAC1Generate`:
    - `test_generate_creates_checklist_items` — POST checklist/generate with mock agent; verify 200; verify response has 3 items with correct text/category/position (0-based); GET checklist; verify same 3 items returned from DB (AC: E07-P1-011)
    - `test_generate_replaces_existing_checklist` — generate once (3 items); generate again with different mock response (2 items); verify DB has exactly 2 items (no accumulation)
    - `test_generate_cross_company_returns_404` — use Company B JWT to call Company A proposal; verify 404 (AC: E07-P0-005 extension)
    - `test_generate_requires_bid_manager_role` — call with viewer/read-only JWT; verify 403 or 401
  - [x] 6.4 `TestAC3Get`:
    - `test_get_checklist_returns_items_ordered_by_position` — generate 3 items; GET checklist; verify items in position order 0,1,2
    - `test_get_checklist_empty_before_generation` — GET on fresh proposal with no checklist; verify 200 with empty items list (not 404)
    - `test_get_checklist_cross_company_returns_404` (AC: E07-P0-005 extension)
  - [x] 6.5 `TestAC4Toggle`:
    - `test_toggle_item_checked_state` — generate checklist; PATCH item 0 `{is_checked: true}`; verify 200 with `is_checked=true`; GET checklist; verify DB state matches (AC: E07-P1-011)
    - `test_toggle_item_unchecked_after_check` — toggle to true then false; verify state transitions
    - `test_toggle_nonexistent_item_returns_404` — PATCH with random UUID item_id; verify 404
    - `test_toggle_item_from_another_proposal_returns_404` — create 2 proposals; try to toggle item from proposal A using proposal B's path param; verify 404
    - `test_toggle_cross_company_returns_404` (AC: E07-P0-005 extension)
  - [x] 6.6 `TestAC5ErrorHandling`:
    - `test_generate_gateway_timeout_returns_504` — mock AI Gateway with `httpx.TimeoutException`; POST generate; verify 504 with `{"error": "agent_timeout", "retry_after": 30}`
    - `test_generate_gateway_unavailable_returns_503` — mock AI Gateway with 503 response; POST generate; verify 503 with `{"error": "agent_unavailable"}`
    - `test_generate_malformed_agent_response_returns_empty` — mock agent returning `{"result": "ok"}` (missing `items` key); POST generate; verify 200 with empty checklist; no crash

## Dev Notes

### Context: Built on Stories 7.1–7.5

**Story 7.1** created `proposals`, `proposal_versions`, `content_blocks` tables (migration 020).
**Story 7.5** added migration 021 for `generation_status` column.
**This story adds migration 022** for `proposal_checklists` table — do NOT modify migrations 020 or 021.

**Story 7.2** (done) created `proposal_service.py` with:
- `_assert_company_owns_proposal(proposal, current_user)` — raises 404 for cross-company
- Router at `services/client-api/src/client_api/api/v1/proposals.py` (prefix `/proposals`)

**Story 7.5** (done) added to `proposal_service.py`:
- `claim_generation`, `build_generation_payload`, `set_generation_status`, `persist_generated_draft`, `reset_stuck_generating_proposals`, `_run_generation_task`
- `POST /{proposal_id}/generate` endpoint (SSE streaming)
- `AiGatewayClient`, `get_ai_gateway_client`, `get_pipeline_readonly_session` already imported in `proposals.py`

**Current router total: 10 endpoints**. This story adds 3 new endpoints (generate checklist, get checklist, toggle item). **No new router registration in `main.py` needed.**

**Test baseline: 106 tests currently passing — must all remain green.**

### CRITICAL: `run_agent` vs `stream_agent` — This Story Is Synchronous

Unlike S07.05 which used `stream_agent()` (SSE), this story uses **`run_agent()`** — a standard async call that returns a `dict` of the full agent response.

```python
# Correct pattern for this story:
result = await gw_client.run_agent("requirement-checklist", payload)
# Returns: {"items": [...]} (or raises AiGatewayTimeoutError / AiGatewayUnavailableError)

# DO NOT use stream_agent — there is no SSE stream for checklist generation
```

`run_agent` sends `POST {base_url}/agents/{agent_name}/run` with `X-Caller-Service: client-api` header.
Mock URL in tests: `http://test-aigw:8000/agents/requirement-checklist/run` (POST, not run-stream).

### Existing Infrastructure — DO NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/alembic/versions/020_proposal_workspace_schema.py` | Migration for proposals/versions/content_blocks | **DO NOT TOUCH** |
| `services/client-api/alembic/versions/021_proposal_generation_status.py` | Migration for generation_status column | **DO NOT TOUCH** |
| `services/client-api/src/client_api/models/proposal.py` | `Proposal` ORM with generation_status | **DO NOT MODIFY** |
| `services/client-api/src/client_api/services/proposal_service.py` | S07.02–S07.05 functions | **ADD new functions only** — do not rename/remove existing |
| `services/client-api/src/client_api/api/v1/proposals.py` | 10 existing endpoints | **ADD 3 new endpoints + imports only** |
| `services/client-api/src/client_api/services/ai_gateway_client.py` | `run_agent()` + `stream_agent()` + error types | **Reuse unchanged** |
| `services/client-api/src/client_api/dependencies.py` | `get_session_factory`, `get_pipeline_readonly_session`, `get_db_session`, `get_ai_gateway_client` | **Reuse unchanged** |
| `services/client-api/tests/api/test_proposals.py` | 66 tests — **must not regress** | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_versions.py` | 32 tests — **must not regress** | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_content_save.py` | 12 tests — **must not regress** | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_generate.py` | 13 tests — **must not regress** | **DO NOT MODIFY** |

### New Files to Create

| File | Purpose |
|------|---------|
| `services/client-api/alembic/versions/022_proposal_checklist.py` | Migration: `proposal_checklists` table + index |
| `services/client-api/src/client_api/models/proposal_checklist.py` | `ProposalChecklistItem` ORM model |
| `services/client-api/src/client_api/schemas/proposal_checklist.py` | `ChecklistItemResponse`, `ChecklistResponse`, `ChecklistItemToggle` schemas |
| `services/client-api/tests/api/test_proposal_checklist.py` | Integration tests (all scenarios) |

### Files to Modify

| File | Change |
|------|--------|
| `services/client-api/src/client_api/models/__init__.py` | Import `ProposalChecklistItem` for alembic autogenerate |
| `services/client-api/src/client_api/services/proposal_service.py` | Add `generate_checklist`, `get_checklist`, `toggle_checklist_item` |
| `services/client-api/src/client_api/api/v1/proposals.py` | Add 3 new endpoints + schema imports |

### New API Endpoints

```
POST  /api/v1/proposals/{proposal_id}/checklist/generate       → invoke Requirement Checklist Agent (bid_manager)
GET   /api/v1/proposals/{proposal_id}/checklist                 → retrieve checklist items (any company user)
PATCH /api/v1/proposals/{proposal_id}/checklist/items/{item_id} → toggle is_checked (bid_manager)
```

### CRITICAL: Checklist Generation — Delete-Then-Insert Pattern

Regenerating the checklist replaces all existing items. Use SQLAlchemy `delete()` in the same session transaction before inserting:

```python
from sqlalchemy import delete, select
from client_api.models.proposal_checklist import ProposalChecklistItem

async def generate_checklist(
    proposal_id: UUID,
    current_user: CurrentUser,
    gw_client: AiGatewayClient,
    session: AsyncSession,
    pipeline_session: AsyncSession,
) -> list[ProposalChecklistItem]:
    # 1. Verify ownership (404 if cross-company)
    stmt = select(Proposal).where(
        Proposal.id == proposal_id,
        Proposal.company_id == current_user.company_id,
    )
    result = await session.execute(stmt)
    proposal = result.scalar_one_or_none()
    if proposal is None:
        raise HTTPException(status_code=404, detail="Proposal not found")

    # 2. Build payload (reuse same structure as build_generation_payload)
    payload = await _build_checklist_payload(proposal, current_user, session, pipeline_session)

    # 3. Call agent (synchronous — raises HTTPException on timeout/unavailable)
    try:
        agent_result = await gw_client.run_agent("requirement-checklist", payload)
    except AiGatewayTimeoutError:
        raise HTTPException(status_code=504, detail={"error": "agent_timeout", "retry_after": 30})
    except AiGatewayUnavailableError:
        raise HTTPException(status_code=503, detail={"error": "agent_unavailable"})

    # 4. Parse — gracefully handle missing/malformed items key
    raw_items = agent_result.get("items") if isinstance(agent_result, dict) else None
    if not isinstance(raw_items, list):
        log.warning(
            "proposal.checklist.malformed_agent_response",
            proposal_id=str(proposal_id),
            agent_result_keys=list(agent_result.keys()) if isinstance(agent_result, dict) else None,
        )
        raw_items = []

    # 5. Delete existing items (same session — route-scoped, OK here)
    await session.execute(
        delete(ProposalChecklistItem).where(
            ProposalChecklistItem.proposal_id == proposal_id
        )
    )

    # 6. Insert new items
    new_items = [
        ProposalChecklistItem(
            proposal_id=proposal_id,
            text=item.get("text", ""),
            linked_section_key=item.get("linked_section_key"),
            category=item.get("category"),
            position=idx,
        )
        for idx, item in enumerate(raw_items)
    ]
    session.add_all(new_items)
    await session.flush()
    for item in new_items:
        await session.refresh(item)

    log.info(
        "proposal.checklist.generated",
        proposal_id=str(proposal_id),
        item_count=len(new_items),
    )
    return new_items
```

**Key**: Unlike S07.05 `generate_draft`, this is a regular route-scoped session — **no background task, no independent session_factory**. The agent call is synchronous (awaited inline). No `session.commit()` needed — the route handler's `get_db_session` dependency commits on exit.

### CRITICAL: Architecture Constraints (MUST FOLLOW — from S07.03/S07.04/S07.05 learnings)

1. **`from __future__ import annotations`** at top of every new file
2. **`structlog` for all logging**: `log = structlog.get_logger()` (never `logging`)
3. **`session.flush()` then `session.refresh()` — never `session.commit()`** for route-scoped sessions
4. **`company_id` exclusively from JWT** — `Proposal.company_id == current_user.company_id` in every query
5. **Return 404 (not 403) for cross-company access** — prevents UUID enumeration (E07-R-003)
6. **`require_role("bid_manager")`** for write operations (generate, toggle); `get_current_user` for read (GET checklist)
7. **`CAST(:param AS uuid)` in `text()` SQL** — if you need raw SQL; never `::uuid` syntax (asyncpg incompatibility). For ORM-based queries in this story, use `Mapped[uuid.UUID]` columns directly — no raw SQL needed.
8. **`delete()` import** from `sqlalchemy` for bulk-delete (not `session.execute(delete(...))` via `text()`)

### Checklist Payload Structure

Payload sent to the `requirement-checklist` agent (reusing the same approach as `build_generation_payload`):
```python
{
    "opportunity": {
        "title": "<from pipeline.opportunities>",
        "description": "<from pipeline.opportunities>",
        "evaluation_criteria": [...],
        "submission_deadline": "<str>",
    },  # All fields → "" if no linked opportunity
    "company": {
        "name": "<from client.companies>",
        "description": "<from client.companies>",
    },
    "existing_sections": [{"key": ..., "title": ..., "body": ...}],  # Current proposal version
    "proposal_id": "<str(proposal_id)>",
}
```

Extract a private helper `_build_checklist_payload(proposal, current_user, session, pipeline_session) -> dict` — do NOT duplicate the opportunity/company loading logic from `build_generation_payload`; refactor shared payload-building logic into a common helper if both functions need it.

### respx Mock Pattern for Sync Agent Calls

Unlike S07.05 SSE mocks, the checklist agent mock is a standard JSON response:

```python
import respx
import httpx

@pytest.fixture
def mock_checklist_agent(respx_mock):
    items = [
        {"text": "Provide proof of financial standing", "linked_section_key": "eligibility", "category": "Administrative"},
        {"text": "Include technical team CVs", "linked_section_key": "technical_approach", "category": "Technical"},
        {"text": "Submit price schedule", "linked_section_key": "pricing", "category": "Financial"},
    ]
    respx_mock.post(
        "http://test-aigw:8000/agents/requirement-checklist/run"
    ).mock(
        return_value=httpx.Response(200, json={"items": items})
    )
    return items

# For timeout test:
respx_mock.post(
    "http://test-aigw:8000/agents/requirement-checklist/run"
).mock(side_effect=httpx.TimeoutException("timeout"))
```

**IMPORTANT**: The `aigw_url_env` session-scoped autouse fixture from `test_proposal_generate.py` sets `CLIENT_API_AIGW_BASE_URL=http://test-aigw:8000` — import and reuse it (or replicate with `conftest.py` scope) so the test AI Gateway client targets the respx mock URL.

### `get_current_user` vs `require_role` Pattern

The GET checklist endpoint uses `get_current_user` (no role check) while generate and toggle use `require_role("bid_manager")`. Verify the correct dependency import name from `proposals.py`:
- Existing usage: `require_role("bid_manager")` wraps writes throughout the file
- For read-only endpoint: look for `get_current_user` or `get_authenticated_user` — use whichever is established in the existing proposals router for read-only endpoints like `GET /proposals` (list)

### Test Coverage Alignment (from test-design-epic-07.md)

| Epic Test ID | Priority | Scenario | Implemented By |
|-------------|----------|----------|----------------|
| **E07-P1-011** | P1 | Generate → store → GET retrieve → PATCH toggle | `TestAC1Generate.test_generate_creates_checklist_items` + `TestAC4Toggle.test_toggle_item_checked_state` |
| **E07-P0-005** | P0 | Company B JWT → 404 on checklist endpoints | `test_generate_cross_company_returns_404`, `test_get_checklist_cross_company_returns_404`, `test_toggle_cross_company_returns_404` |
| **E07-P2-005** | P2 | AI Gateway timeout → 504 | `TestAC5ErrorHandling.test_generate_gateway_timeout_returns_504` |

### Running Tests

```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_proposal_checklist.py -v

# Full regression check (must stay green — 106 existing tests):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py -v
```

### Project Structure Notes

- `ProposalChecklistItem` ORM model in its own file `models/proposal_checklist.py` (consistent with per-model file pattern)
- All three service functions added to `proposal_service.py` (service layer owns business logic — no new service file)
- All three endpoints added to `proposals.py` router (consistent with S07.05 pattern — no sub-router)
- `proposal_checklist.py` schema file separate from `proposals.py` (consistent with `proposal_generate.py` pattern from S07.05)
- Migration 022 is purely additive — no data migration needed; existing proposals simply have no checklist rows until `generate` is called

### References

- [Source: eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md#S07.06] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P1-011] — generate/retrieve/toggle integration test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-005] — cross-company RLS test (5 endpoint types + checklist endpoints)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P2-005] — agent timeout → 504 test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-003] — RLS bypass risk; 404 not 403
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-007] — AI agent timeout cascade risk; 504 with retry_after
- [Source: eusolicit-docs/implementation-artifacts/7-5-ai-draft-generation-backend-sse-integration.md] — `build_generation_payload` pattern, `AiGatewayClient` usage, `_assert_company_owns_proposal`, `CAST(:id AS uuid)` constraint, `structlog` pattern, respx mock fixtures
- [Source: eusolicit-app/services/client-api/src/client_api/services/ai_gateway_client.py] — `run_agent()` async method signature (returns `dict`, raises `AiGatewayTimeoutError` / `AiGatewayUnavailableError`)
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py] — established patterns for `require_role`, `get_current_user`, session dependency injection, response model annotations

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

- Combined test suite failure (tests/api/test_proposal_checklist.py): `@respx.mock` tests failed when run after `test_proposal_generate.py` because `get_settings()` LRU cache was stale — it held `aigw_base_url=http://ai-gateway:8000` (production default) from before `CLIENT_API_AIGW_BASE_URL` env var was set. Fix: clear `get_settings.cache_clear()` in both `aigw_url_env` (session) and `checklist_proposal` (per-test) fixtures, and use `os.environ[key] = value` (not `setdefault`) in `aigw_url_env`.

### Completion Notes List

- ✅ Alembic migration 022 created `client.proposal_checklists` table with all required columns, FK ON DELETE CASCADE, and `ix_proposal_checklists_proposal_id` index. Downgrade/upgrade cycle verified.
- ✅ `ProposalChecklistItem` ORM model created following established per-model file pattern. Registered in `models/__init__.py` for alembic autogenerate.
- ✅ Pydantic schemas `ChecklistItemResponse`, `ChecklistResponse`, `ChecklistItemToggle` created in `schemas/proposal_checklist.py`.
- ✅ Three service functions added to `proposal_service.py`: `generate_checklist` (delete-then-insert, synchronous agent call, error handling), `get_checklist` (position-ordered, empty list if no checklist), `toggle_checklist_item` (cross-proposal protection, updated_at update). Private helper `_build_checklist_payload` extracted to share opportunity/company loading logic.
- ✅ Three endpoints added to `proposals.py` router: `POST /{id}/checklist/generate` (bid_manager), `GET /{id}/checklist` (any company user), `PATCH /{id}/checklist/items/{item_id}` (bid_manager). Error responses for 503/504 return raw bodies (no FastAPI detail wrapper) matching AC5 contract.
- ✅ All 19 ATDD integration tests pass: AC1-AC7 coverage including nominal generation, regeneration (delete-then-insert), ordered retrieval, toggle state transitions, cross-company 404 isolation, AI Gateway timeout 504, AI Gateway unavailable 503, malformed response empty list, migration schema validation.
- ✅ All 112 existing regression tests continue to pass (total: 131 tests, 0 failures).

### File List

- `services/client-api/alembic/versions/022_proposal_checklist.py` (new)
- `services/client-api/src/client_api/models/proposal_checklist.py` (new)
- `services/client-api/src/client_api/schemas/proposal_checklist.py` (new)
- `services/client-api/tests/api/test_proposal_checklist.py` (modified — removed skip decorators, fixed aigw_url_env fixture for combined test run)
- `services/client-api/src/client_api/models/__init__.py` (modified — added ProposalChecklistItem import)
- `services/client-api/src/client_api/services/proposal_service.py` (modified — added delete import, _build_checklist_payload, generate_checklist, get_checklist, toggle_checklist_item)
- `services/client-api/src/client_api/api/v1/proposals.py` (modified — added ChecklistItemResponse/ChecklistResponse/ChecklistItemToggle imports, 3 new endpoints)

## Change Log

- Story 7.6 implementation complete (Date: 2026-04-17): Added requirement checklist agent integration — Alembic migration 022 (client.proposal_checklists), ProposalChecklistItem ORM model, Pydantic schemas, three service functions (generate_checklist, get_checklist, toggle_checklist_item), three API endpoints (POST/GET/PATCH), 19 integration tests all passing, 131 total tests (0 regressions).
