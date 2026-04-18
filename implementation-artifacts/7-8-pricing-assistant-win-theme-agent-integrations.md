# Story 7.8: Pricing Assistant & Win Theme Agent Integrations

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager**,
I want to get AI-powered pricing recommendations and win theme analysis for my proposal,
so that I can competitively price my tender response and position it with the strongest differentiators before submission.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/proposals/{proposal_id}/pricing-assist` invokes the `pricing-assistant` agent via `AiGatewayClient.run_agent()` (synchronous, non-SSE). Requires `bid_manager` role; returns 404 for cross-company or non-existent proposal. Payload built from `_build_agent_payload_base` — opportunity details (title, description, submission_deadline, evaluation_criteria) and company profile (name, description) provide market context. Agent response expected as `{"recommended_price": <float>, "market_range": {"min": <float>, "median": <float>, "max": <float>}, "justification": <str>}`. Result stored in `proposals.pricing_result` JSONB column. Returns `HTTP 200` with the full pricing result and a top-level `priced_at` timestamp.

2. **AC2** — `POST /api/v1/proposals/{proposal_id}/win-themes` invokes the `win-theme-extractor` agent via `AiGatewayClient.run_agent()`. Requires `bid_manager` role; returns 404 for cross-company or non-existent proposal. Payload built from `_build_agent_payload_base` — opportunity requirements and company strengths (from company profile description) are the primary inputs. Agent response expected as `{"themes": [{"title": <str>, "description": <str>}, ...]}` (rank-ordered, first theme is highest priority). Result stored in `proposals.win_themes_result` JSONB column. Returns `HTTP 200` with the themes list (rank-ordered) and `analyzed_at` timestamp.

3. **AC3** — `GET /api/v1/proposals/{proposal_id}/pricing-assist` and `GET /api/v1/proposals/{proposal_id}/win-themes` retrieve stored results. Any authenticated company user may call GET (no `bid_manager` requirement). Returns 404 for cross-company or non-existent proposal. If no result has been generated yet, returns `HTTP 200` with `{"result": null}` (never a 404 for missing result).

4. **AC4** — **Error handling**: `AiGatewayTimeoutError` → `HTTP 504` with `{"error": "agent_timeout", "retry_after": 30}`. `AiGatewayUnavailableError` → `HTTP 503` with `{"error": "agent_unavailable"}`. Agent response with missing or malformed top-level key (`recommended_price`/`themes`) → return `HTTP 200` with safe defaults (null pricing result or empty themes list) and log a warning (no crash, no partial store). Cross-company or non-existent proposal → `HTTP 404` (never 403).

5. **AC5** — Alembic migration `024_proposal_pricing_win_themes` adds two nullable JSONB columns to `client.proposals`:
   - `pricing_result JSONB NULL`
   - `win_themes_result JSONB NULL`

   `revision = "024_proposal_pricing_win_themes"`, `down_revision = "023_proposal_agent_results"`. `upgrade()` uses `op.add_column` with `JSONB()` from `sqlalchemy.dialects.postgresql`; `downgrade()` uses `op.drop_column`. Reversible — `alembic upgrade head` then `alembic downgrade -1` must succeed cleanly.

6. **AC6** — Integration tests at `services/client-api/tests/api/test_proposal_pricing_win_themes.py` covering: (a) nominal pricing assist — mock agent response stored and retrievable via GET; (b) nominal win themes — themes rank-ordered, stored, retrievable; (c) malformed agent response returns safe defaults (no crash); (d) cross-company 404 for all 4 endpoints (parameterized or individual); (e) AI Gateway timeout → 504 (one per agent, representative); (f) GET with no result → `{"result": null}`. All 161 existing tests (from Stories 7.1–7.7) must continue to pass.

## Tasks / Subtasks

- [x] Task 1: Alembic migration `024_proposal_pricing_win_themes.py` (AC: 5)
  - [ ] 1.1 Create `services/client-api/alembic/versions/024_proposal_pricing_win_themes.py`:
    ```python
    from __future__ import annotations
    from alembic import op
    from sqlalchemy.dialects.postgresql import JSONB

    revision = "024_proposal_pricing_win_themes"
    down_revision = "023_proposal_agent_results"
    branch_labels = None
    depends_on = None

    def upgrade() -> None:
        op.add_column(
            "proposals",
            JSONB().with_variant(JSONB(), "postgresql"),  # use JSONB not sa.JSON
            schema="client",
        )
        # Correct form:
        op.add_column(
            "proposals",
            op.f("pricing_result"),  # placeholder — use proper column below
            schema="client",
        )
    ```
    **Actual correct migration (use this exact code):**
    ```python
    from __future__ import annotations
    import sqlalchemy as sa
    from alembic import op
    from sqlalchemy.dialects.postgresql import JSONB

    revision = "024_proposal_pricing_win_themes"
    down_revision = "023_proposal_agent_results"
    branch_labels = None
    depends_on = None

    def upgrade() -> None:
        op.add_column(
            "proposals",
            sa.Column("pricing_result", JSONB(), nullable=True),
            schema="client",
        )
        op.add_column(
            "proposals",
            sa.Column("win_themes_result", JSONB(), nullable=True),
            schema="client",
        )

    def downgrade() -> None:
        op.drop_column("proposals", "win_themes_result", schema="client")
        op.drop_column("proposals", "pricing_result", schema="client")
    ```
  - [ ] 1.2 Verify `down_revision = "023_proposal_agent_results"` exactly matches the revision string of the 7.7 migration (NOT `"023_proposal_agent_results_columns"` — it was truncated to fit VARCHAR(32))
  - [ ] 1.3 Run `alembic upgrade head` then `alembic downgrade -1` — both must succeed

- [x] Task 2: Update `Proposal` ORM model with 2 new JSONB columns (AC: 5)
  - [ ] 2.1 Open `services/client-api/src/client_api/models/proposal.py` — verify existing JSONB import before adding (it was added in 7.7). ADD after the 3 existing result columns from Story 7.7:
    ```python
    # After scoring_simulation_result (added in Story 7.7):
    pricing_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    win_themes_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
    ```
  - [ ] 2.2 Do NOT touch `compliance_check_result`, `clause_risk_result`, `scoring_simulation_result`, `generation_status`, `current_version_id`, `company_id`, `opportunity_id`, or any other existing columns

- [x] Task 3: Pydantic schemas in `proposal_pricing_win_themes.py` (AC: 1–4)
  - [ ] 3.1 Create `services/client-api/src/client_api/schemas/proposal_pricing_win_themes.py`:
    ```python
    from __future__ import annotations
    from datetime import datetime
    from pydantic import BaseModel


    # ── Pricing Assistant ─────────────────────────────────────────────────────
    class MarketRange(BaseModel):
        min: float
        median: float
        max: float

    class PricingAssistResponse(BaseModel):
        recommended_price: float
        market_range: MarketRange
        justification: str
        priced_at: datetime

    # ── Win Theme Extractor ───────────────────────────────────────────────────
    class WinTheme(BaseModel):
        title: str
        description: str

    class WinThemesResponse(BaseModel):
        themes: list[WinTheme]
        analyzed_at: datetime
    ```
  - [ ] 3.2 Import `NullResultResponse` from `proposal_agent_results.py` in the router — do NOT redefine it. It is already at `services/client-api/src/client_api/schemas/proposal_agent_results.py`.

- [x] Task 4: Service functions in `proposal_service.py` (AC: 1–4)
  - [ ] 4.1 Add imports at top of `services/client-api/src/client_api/services/proposal_service.py` (ADD only — never remove existing):
    ```python
    from client_api.schemas.proposal_pricing_win_themes import (
        MarketRange,
        PricingAssistResponse,
        WinTheme,
    )
    ```
  - [ ] 4.2 Add `run_pricing_assist(proposal_id, current_user, gw_client, session, pipeline_session) -> tuple[PricingAssistResponse, datetime]`:
    ```python
    async def run_pricing_assist(
        proposal_id: uuid.UUID,
        current_user: CurrentUser,
        gw_client: AiGatewayClient,
        session: AsyncSession,
        pipeline_session: AsyncSession,
    ) -> PricingAssistResponse:
        # 1. Ownership check
        stmt = select(Proposal).where(
            Proposal.id == proposal_id,
            Proposal.company_id == current_user.company_id,
        )
        proposal = (await session.execute(stmt)).scalar_one_or_none()
        if proposal is None:
            raise HTTPException(status_code=404, detail="Proposal not found")

        # 2. Build payload using shared base builder
        payload = await _build_agent_payload_base(proposal, current_user, session, pipeline_session)

        # 3. Call agent (synchronous run_agent — NOT stream_agent)
        try:
            result = await gw_client.run_agent("pricing-assistant", payload)
        except AiGatewayTimeoutError:
            raise HTTPException(status_code=504, detail={"error": "agent_timeout", "retry_after": 30})
        except AiGatewayUnavailableError:
            raise HTTPException(status_code=503, detail={"error": "agent_unavailable"})

        # 4. Parse — malformed response returns None (safe default), no crash
        priced_at = datetime.now(tz=timezone.utc)
        if not isinstance(result, dict) or "recommended_price" not in result:
            log.warning("proposal.pricing_assist.malformed_response", proposal_id=str(proposal_id))
            # Do NOT persist on malformed — pricing_result stays NULL
            raise HTTPException(status_code=200, ...)  # actually return safe response
            # See note: return PricingAssistResponse with None fields OR raise a 200 with empty
            # Best approach: return a sentinel — see router for how to handle
        
        # Parse market_range
        raw_range = result.get("market_range", {})
        market_range = MarketRange(
            min=float(raw_range.get("min", 0.0)),
            median=float(raw_range.get("median", 0.0)),
            max=float(raw_range.get("max", 0.0)),
        )
        pricing = PricingAssistResponse(
            recommended_price=float(result["recommended_price"]),
            market_range=market_range,
            justification=str(result.get("justification", "")),
            priced_at=priced_at,
        )

        # 5. Persist
        proposal.pricing_result = {
            "recommended_price": pricing.recommended_price,
            "market_range": pricing.market_range.model_dump(),
            "justification": pricing.justification,
            "priced_at": priced_at.isoformat(),
        }
        await session.flush()
        log.info("proposal.pricing_assist.complete", proposal_id=str(proposal_id))
        return pricing
    ```
    **IMPORTANT NOTE on malformed response:** Unlike Story 7.7 which returned an empty list, pricing has a complex structure. The cleanest pattern: wrap the entire parse in try/except, on any `KeyError`/`TypeError`/`ValueError` → log warning, do NOT store to DB, return a `NullResultResponse`-compatible response. The router checks if service raises or returns `None`, and the router returns `NullResultResponse`. Alternatively, raise an `HTTPException(status_code=200)` — but better pattern: service returns `None` on malformed, router handles it. **Follow whichever pattern is cleanest** — see existing `get_compliance_check_result` for the `None`-return → `NullResultResponse` pattern as a guide.

  - [ ] 4.3 Add `run_win_themes(proposal_id, current_user, gw_client, session, pipeline_session) -> tuple[list[WinTheme], datetime]`:
    - Ownership check (same pattern as all other agents)
    - Build payload with `_build_agent_payload_base` (opportunity requirements + company description serve as "company strengths" context)
    - Call `gw_client.run_agent("win-theme-extractor", payload)`; handle `AiGatewayTimeoutError` → 504, `AiGatewayUnavailableError` → 503
    - Parse `result.get("themes", [])` — each item must have `title` and `description`; malformed items logged and skipped
    - The list is already rank-ordered (first theme = highest priority); preserve order exactly
    - Persist `proposal.win_themes_result = {"themes": [t.model_dump() for t in themes], "analyzed_at": analyzed_at.isoformat()}`
    - `await session.flush()` — never `session.commit()`
    - Return `(themes, analyzed_at)`

  - [ ] 4.4 Add two GET helpers (no agent call — DB read only):
    ```python
    async def get_pricing_assist_result(
        proposal_id: uuid.UUID,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> dict | None:
        stmt = select(Proposal).where(
            Proposal.id == proposal_id,
            Proposal.company_id == current_user.company_id,
        )
        proposal = (await session.execute(stmt)).scalar_one_or_none()
        if proposal is None:
            raise HTTPException(status_code=404, detail="Proposal not found")
        return proposal.pricing_result  # None if never generated

    async def get_win_themes_result(
        proposal_id: uuid.UUID,
        current_user: CurrentUser,
        session: AsyncSession,
    ) -> dict | None:
        stmt = select(Proposal).where(
            Proposal.id == proposal_id,
            Proposal.company_id == current_user.company_id,
        )
        proposal = (await session.execute(stmt)).scalar_one_or_none()
        if proposal is None:
            raise HTTPException(status_code=404, detail="Proposal not found")
        return proposal.win_themes_result  # None if never generated
    ```

- [x] Task 5: Router endpoints in `proposals.py` (AC: 1–4)
  - [ ] 5.1 Add imports to `services/client-api/src/client_api/api/v1/proposals.py`:
    ```python
    from client_api.schemas.proposal_pricing_win_themes import (
        PricingAssistResponse,
        WinThemesResponse,
    )
    # NullResultResponse is already imported from proposal_agent_results — reuse it
    ```
  - [ ] 5.2 Add `POST /{proposal_id}/pricing-assist`:
    ```python
    @router.post(
        "/{proposal_id}/pricing-assist",
        response_model=PricingAssistResponse,
        responses={
            404: {"description": "Proposal not found or cross-company"},
            503: {"description": "AI Gateway unavailable"},
            504: {"description": "AI Gateway timeout"},
        },
    )
    async def pricing_assist(
        proposal_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        pipeline_session: Annotated[AsyncSession, Depends(get_pipeline_readonly_session)],
        current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
    ) -> PricingAssistResponse | NullResultResponse:
        result = await proposal_service.run_pricing_assist(
            proposal_id, current_user, gw_client, session, pipeline_session
        )
        if result is None:
            return NullResultResponse()
        return result
    ```
  - [ ] 5.3 Add `GET /{proposal_id}/pricing-assist`:
    ```python
    @router.get("/{proposal_id}/pricing-assist")
    async def get_pricing_assist(
        proposal_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> PricingAssistResponse | NullResultResponse:
        result = await proposal_service.get_pricing_assist_result(proposal_id, current_user, session)
        if result is None:
            return NullResultResponse()
        return PricingAssistResponse(**result)
    ```
  - [ ] 5.4 Add `POST /{proposal_id}/win-themes`:
    ```python
    @router.post(
        "/{proposal_id}/win-themes",
        response_model=WinThemesResponse,
        responses={
            404: {"description": "Proposal not found or cross-company"},
            503: {"description": "AI Gateway unavailable"},
            504: {"description": "AI Gateway timeout"},
        },
    )
    async def win_themes(
        proposal_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        pipeline_session: Annotated[AsyncSession, Depends(get_pipeline_readonly_session)],
        current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
    ) -> WinThemesResponse:
        themes, analyzed_at = await proposal_service.run_win_themes(
            proposal_id, current_user, gw_client, session, pipeline_session
        )
        return WinThemesResponse(themes=themes, analyzed_at=analyzed_at)
    ```
  - [ ] 5.5 Add `GET /{proposal_id}/win-themes`:
    ```python
    @router.get("/{proposal_id}/win-themes")
    async def get_win_themes(
        proposal_id: UUID,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> WinThemesResponse | NullResultResponse:
        result = await proposal_service.get_win_themes_result(proposal_id, current_user, session)
        if result is None:
            return NullResultResponse()
        return WinThemesResponse(**result)
    ```

- [x] Task 6: Integration tests `tests/api/test_proposal_pricing_win_themes.py` (AC: 6)
  - [ ] 6.1 Set up `aigw_url_env` autouse session fixture — **EXACT SAME PATTERN** as `test_proposal_compliance_risk_scoring.py` (Story 7.7):
    ```python
    @pytest.fixture(autouse=True, scope="session")
    def aigw_url_env():
        import os
        from client_api.config import get_settings
        os.environ["CLIENT_API_AIGW_BASE_URL"] = "http://test-aigw:8000"
        get_settings.cache_clear()
        yield
        get_settings.cache_clear()
    ```
  - [ ] 6.2 Create `pricing_win_themes_proposal` fixture — yields (client, session, token, proposal_id) with seeded sections (reuse `agent_results_proposal` fixture pattern from 7.7)
  - [ ] 6.3 `respx` mock fixtures:
    ```python
    @pytest.fixture
    def mock_pricing_agent(respx_mock):
        pricing_data = {
            "recommended_price": 485000.0,
            "market_range": {"min": 400000.0, "median": 480000.0, "max": 600000.0},
            "justification": "Based on comparable tenders in the public sector over last 12 months",
        }
        respx_mock.post("http://test-aigw:8000/agents/pricing-assistant/run").mock(
            return_value=httpx.Response(200, json=pricing_data)
        )
        return pricing_data

    @pytest.fixture
    def mock_win_themes_agent(respx_mock):
        themes_data = {
            "themes": [
                {"title": "Digital Transformation Expertise", "description": "10 years delivering public sector digital services"},
                {"title": "Cost Efficiency", "description": "Proven track record of 15% below-budget delivery"},
                {"title": "Social Value Commitment", "description": "50 apprenticeships per £1M contract value"},
            ]
        }
        respx_mock.post("http://test-aigw:8000/agents/win-theme-extractor/run").mock(
            return_value=httpx.Response(200, json=themes_data)
        )
        return themes_data
    ```
  - [ ] 6.4 `TestPricingAssist`:
    - `test_pricing_assist_stores_and_retrieves_result` (E07-P1-015) — POST with mock agent; verify 200 with `recommended_price`, `market_range`, `justification`, `priced_at`; GET endpoint returns same stored result
    - `test_pricing_assist_opportunity_in_payload` — verify agent payload includes `opportunity` dict with `title` and `existing_sections` via `respx.calls[0].request.content`
    - `test_pricing_assist_malformed_response_returns_null` — mock agent returning `{"status": "ok"}` (missing `recommended_price`); POST; verify 200 with `{"result": null}`; no crash; `pricing_result` remains NULL in DB
    - `test_pricing_assist_cross_company_returns_404` — Company B JWT calls Company A proposal; verify 404 (E07-R-003)
    - `test_pricing_assist_timeout_returns_504` — mock `httpx.TimeoutException`; POST; verify 504 `{"error": "agent_timeout", "retry_after": 30}` (E07-R-007/E07-P2-005)
    - `test_get_pricing_assist_no_result_returns_null` — GET on fresh proposal (never triggered pricing); verify `{"result": null}` (not 404)
    - `test_get_pricing_assist_cross_company_returns_404`
  - [ ] 6.5 `TestWinThemes`:
    - `test_win_themes_stores_and_retrieves_result` (E07-P1-016) — POST with mock agent returning 3 themes; verify 200 with `themes` list and `analyzed_at`; GET retrieves in same rank order
    - `test_win_themes_rank_order_preserved` — POST with mock returning themes A, B, C; verify GET returns A, B, C in that exact order (not sorted by any other key)
    - `test_win_themes_company_in_payload` — verify agent payload includes `company` dict and `existing_sections` via payload inspection
    - `test_win_themes_malformed_response_returns_empty_themes` — mock agent returning `{"result": "ok"}` (missing `themes`); POST; verify 200 with `{"themes": [], "analyzed_at": ...}`; no crash
    - `test_win_themes_cross_company_returns_404` — Company B JWT calls Company A proposal; verify 404
    - `test_win_themes_timeout_returns_504` — mock `httpx.TimeoutException`; POST; verify 504
    - `test_get_win_themes_no_result_returns_null` — GET on fresh proposal; verify `{"result": null}`
    - `test_get_win_themes_cross_company_returns_404`

## Dev Notes

### Context: Built on Stories 7.1–7.7

**Migration chain (CRITICAL — do NOT modify prior migrations):**
- `020_proposal_workspace_schema`: proposals / proposal_versions / content_blocks
- `021_proposal_generation_status`: `generation_status` column
- `022_proposal_checklist`: `proposal_checklists` table
- `023_proposal_agent_results` (NOT `023_proposal_agent_results_columns` — truncated for VARCHAR(32)): 3 JSONB result columns
- **THIS STORY: `024_proposal_pricing_win_themes`** — adds `pricing_result` and `win_themes_result` JSONB columns

**CRITICAL — Revision ID length:** PostgreSQL `alembic_version.version_num` is `VARCHAR(32)`. Story 7.7 discovered this when `"023_proposal_agent_results_columns"` (34 chars) was truncated to `"023_proposal_agent_results"` (26 chars). This story's ID `"024_proposal_pricing_win_themes"` is **31 chars** — fits within 32. Use it exactly.

**`down_revision` must be `"023_proposal_agent_results"` — NOT the story spec name.** Verify by inspecting the actual migration file at `services/client-api/alembic/versions/023_proposal_agent_results_columns.py` before writing the migration.

**Current endpoint count: 19** (13 from 7.1–7.5 + 3 from 7.6 + 6 from 7.7 — includes 3 POST + 3 GET from compliance/clause-risk/scoring). This story adds 4 new endpoints. No new router registration in `main.py` needed.

**Test baseline: 161 tests passing** (131 from 7.1–7.6 + 30 from 7.7). All must remain green.

### CRITICAL: Both Agents Use `run_agent()` — NOT `stream_agent()`

This story is entirely synchronous. Do NOT use `stream_agent()`. Both agents return a full JSON response.

```python
# Correct pattern for BOTH agents:
result = await gw_client.run_agent("pricing-assistant", payload)
result = await gw_client.run_agent("win-theme-extractor", payload)

# Mock URLs in tests (must match agents.yaml logical names):
# POST http://test-aigw:8000/agents/pricing-assistant/run
# POST http://test-aigw:8000/agents/win-theme-extractor/run
```

### CRITICAL: Reuse `_build_agent_payload_base` — Do NOT Duplicate

Story 7.7 refactored `_build_checklist_payload` into a shared `_build_agent_payload_base`. This story's two service functions must call `_build_agent_payload_base` directly — no new loading logic for opportunity or company data.

```python
# Both agents use the base payload as-is:
payload = await _build_agent_payload_base(proposal, current_user, session, pipeline_session)
# Then call agent with payload directly (no augmentation needed for either)
result = await gw_client.run_agent("pricing-assistant", payload)
```

Payload structure (from `_build_agent_payload_base`):
```python
{
    "opportunity": {
        "title": "<str>",
        "description": "<str>",
        "evaluation_criteria": [...],
        "submission_deadline": "<str>",
    },
    "company": {
        "name": "<str>",
        "description": "<str>",   # ← serves as "company strengths" for win-theme-extractor
    },
    "existing_sections": [{"key": ..., "title": ..., "body": ...}],
    "proposal_id": "<str(proposal_id)>",
}
```

### Existing Infrastructure — DO NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/alembic/versions/020_proposal_workspace_schema.py` | Base schema | **DO NOT TOUCH** |
| `services/client-api/alembic/versions/021_proposal_generation_status.py` | generation_status | **DO NOT TOUCH** |
| `services/client-api/alembic/versions/022_proposal_checklist.py` | proposal_checklists | **DO NOT TOUCH** |
| `services/client-api/alembic/versions/023_proposal_agent_results_columns.py` | 3 result columns | **DO NOT TOUCH** |
| `services/client-api/src/client_api/services/ai_gateway_client.py` | `run_agent()`, `stream_agent()`, error types | **Reuse unchanged** |
| `services/client-api/src/client_api/dependencies.py` | `get_db_session`, `get_pipeline_readonly_session`, `get_ai_gateway_client` | **Reuse unchanged** |
| `services/client-api/src/client_api/schemas/proposal_agent_results.py` | `NullResultResponse`, `ComplianceCriterion`, etc. | **Import `NullResultResponse` — do not redefine** |
| `services/client-api/tests/api/test_proposals.py` | 66 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_versions.py` | 32 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_content_save.py` | 12 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_generate.py` | 13 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_checklist.py` | 19 tests | **DO NOT MODIFY** |
| `services/client-api/tests/api/test_proposal_compliance_risk_scoring.py` | 30 tests | **DO NOT MODIFY** |

### Modifying `proposal.py` ORM Model — Safe Extension Pattern

Identical to Story 7.7's safe extension. The `JSONB` import already exists (added in 7.7). **Check before adding.** ADD only after the 3 result columns from 7.7:

```python
# In services/client-api/src/client_api/models/proposal.py
# Existing from Story 7.7 (already there — verify, do NOT re-add):
# compliance_check_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
# clause_risk_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
# scoring_simulation_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)

# ADD these NEW columns (Story 7.8):
pricing_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
win_themes_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)
```

### Persisting Results — JSONB Columns on Proposals Table

```python
# Pricing assist — store full structure:
proposal.pricing_result = {
    "recommended_price": pricing.recommended_price,
    "market_range": {"min": ..., "median": ..., "max": ...},
    "justification": pricing.justification,
    "priced_at": datetime.now(tz=timezone.utc).isoformat(),
}
await session.flush()  # NO session.commit() — route-scoped session commits on exit

# Win themes — store ranked list:
proposal.win_themes_result = {
    "themes": [{"title": t.title, "description": t.description} for t in themes],
    "analyzed_at": datetime.now(tz=timezone.utc).isoformat(),
}
await session.flush()
```

### GET Endpoints — Return JSONB Column Directly

Pattern identical to Story 7.7's GET helpers. Service returns `dict | None`; router wraps in response schema or `NullResultResponse`:

```python
# In router for GET /pricing-assist:
result = await proposal_service.get_pricing_assist_result(proposal_id, current_user, session)
if result is None:
    return NullResultResponse()
return PricingAssistResponse(**result)

# In router for GET /win-themes:
result = await proposal_service.get_win_themes_result(proposal_id, current_user, session)
if result is None:
    return NullResultResponse()
return WinThemesResponse(**result)
```

### Malformed Response Handling — Pricing vs Win Themes

**Win Themes** follows the Story 7.7 list pattern: parse `result.get("themes", [])`, skip malformed items, return empty list (and store empty list). Always returns valid `WinThemesResponse`.

**Pricing** has a different structure — `recommended_price` is a required float. If missing/malformed, the service cannot return a `PricingAssistResponse`. Two options:
1. Return `None` from service → router returns `NullResultResponse()` (do NOT store)
2. Raise a logged warning + return default zeros (not recommended — misleads the user)

**Use option 1**: service returns `None` on malformed pricing response, router returns `NullResultResponse`. This matches the "no partial store" principle from Story 7.7 (completion note 4).

### `aigw_url_env` Fixture — LRU Cache Issue (Lesson from Story 7.6/7.7)

**MUST include** in `test_proposal_pricing_win_themes.py`. Pattern from 7.7 (do NOT use `setdefault`):
```python
@pytest.fixture(autouse=True, scope="session")
def aigw_url_env():
    import os
    from client_api.config import get_settings
    os.environ["CLIENT_API_AIGW_BASE_URL"] = "http://test-aigw:8000"
    get_settings.cache_clear()
    yield
    get_settings.cache_clear()
```

Also call `get_settings.cache_clear()` in any per-test fixture that creates a new `async_client`.

### New Files to Create

| File | Purpose |
|------|---------|
| `services/client-api/alembic/versions/024_proposal_pricing_win_themes.py` | Migration: 2 JSONB columns |
| `services/client-api/src/client_api/schemas/proposal_pricing_win_themes.py` | Pydantic schemas for pricing + win themes |
| `services/client-api/tests/api/test_proposal_pricing_win_themes.py` | Integration tests (~15 tests) |

### Files to Modify

| File | Change |
|------|--------|
| `services/client-api/src/client_api/models/proposal.py` | Add 2 JSONB mapped columns after existing 3 from Story 7.7 |
| `services/client-api/src/client_api/services/proposal_service.py` | Add 4 new service functions (run_pricing_assist, get_pricing_assist_result, run_win_themes, get_win_themes_result) |
| `services/client-api/src/client_api/api/v1/proposals.py` | Add 4 new endpoints + schema imports |

### New API Endpoints

```
POST  /api/v1/proposals/{proposal_id}/pricing-assist    → Pricing Assistant Agent (bid_manager)
GET   /api/v1/proposals/{proposal_id}/pricing-assist    → retrieve latest pricing result (any user)
POST  /api/v1/proposals/{proposal_id}/win-themes        → Win Theme Extractor Agent (bid_manager)
GET   /api/v1/proposals/{proposal_id}/win-themes        → retrieve latest win themes result (any user)
```

### CRITICAL Architecture Constraints (Inherited — Must Follow)

1. **`from __future__ import annotations`** at top of every new file
2. **`structlog` for all logging**: `log = structlog.get_logger()` (never `logging`)
3. **`session.flush()` — never `session.commit()`** for route-scoped sessions
4. **`company_id` exclusively from JWT** — `Proposal.company_id == current_user.company_id` in every query; never accept from request body
5. **Return 404 (not 403) for cross-company** — prevents UUID enumeration (E07-R-003)
6. **`require_role("bid_manager")`** for POST endpoints; `get_current_user` for GET endpoints
7. **`run_agent()` not `stream_agent()`** — both agents are synchronous
8. **AI Gateway timeout → 504 with `retry_after: 30`** — same contract as Stories 7.6 and 7.7
9. **Malformed agent response → safe default + warning log** — never crash, never 500

### Running Tests

```bash
cd eusolicit-app/services/client-api

# New story tests only:
pytest tests/api/test_proposal_pricing_win_themes.py -v

# Full regression suite (all 161 existing + new tests):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py \
       tests/api/test_proposal_compliance_risk_scoring.py \
       tests/api/test_proposal_pricing_win_themes.py -v
```

### Test Coverage Map (from Epic 7 Test Design)

| Test Design ID | Priority | Scenario | Implementation |
|----------------|----------|----------|----------------|
| **E07-P1-015** | P1 | Pricing assistant returns recommendation with market range | `TestPricingAssist.test_pricing_assist_stores_and_retrieves_result` |
| **E07-P1-016** | P1 | Win themes returns ranked list | `TestWinThemes.test_win_themes_stores_and_retrieves_result` |
| **E07-P0-005** | P0 (extension) | Cross-company 404 on all new endpoints | `test_*_cross_company_returns_404` (4 tests) |
| **E07-P2-005** | P2 | AI Gateway timeout → 504 | `test_pricing_assist_timeout_returns_504`, `test_win_themes_timeout_returns_504` |
| **E07-R-003** | Risk | RLS bypass via cross-company ID enumeration | All `test_*_cross_company_returns_404` tests |
| **E07-R-007** | Risk | AI agent timeout cascade | `AI_AGENT_TIMEOUT_SECONDS` env var applied to both agent calls |

### Advisory Items from Story 7.7 Review (Carry Forward Awareness)

1. **ComplianceFramework company-scoping**: The 7.7 reviewer noted that `ComplianceFramework` lookup is not company-scoped. This story's agents do not load `ComplianceFramework` so this is not a concern here.

2. **Malformed response DB state**: Story 7.7 reviewer noted tests don't verify DB NULL state after malformed response. For this story, add one assertion in `test_pricing_assist_malformed_response_returns_null` that directly queries `proposals.pricing_result` and confirms it remains `NULL` after the malformed response.

### Project Structure Notes

- All 4 new endpoints added to existing `proposals.py` router (no new router file, no new `main.py` registration)
- Schema file `proposal_pricing_win_themes.py` is separate from `proposal_agent_results.py` (consistent with per-story schema file pattern from 7.5–7.7)
- Migration 024 is purely additive — no data migration; existing proposals will have both new columns as `NULL` until each agent is invoked
- `Proposal` ORM extension is backward compatible — all existing 161 tests work unchanged

### References

- [Source: eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md#S07.08] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P1-015] — pricing assist integration test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P1-016] — win themes integration test
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-005] — cross-company RLS tests (extends to all new endpoints)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P2-005] — AI Gateway timeout → 504 (representative)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-003] — RLS bypass; 404 not 403
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-R-007] — AI agent timeout cascade; `AI_AGENT_TIMEOUT_SECONDS` env var
- [Source: eusolicit-docs/implementation-artifacts/7-7-compliance-risk-scoring-agent-integrations.md] — `run_agent()` pattern, `_build_agent_payload_base`, respx mock fixtures, `aigw_url_env` LRU cache fix, `require_role`/`get_current_user` pattern, `session.flush()` discipline, JSONB column pattern, NullResultResponse usage, revision ID VARCHAR(32) constraint (critical — down_revision = "023_proposal_agent_results")
- [Source: eusolicit-docs/implementation-artifacts/7-6-requirement-checklist-agent-integration.md] — original `_build_checklist_payload` (now `_build_agent_payload_base`), respx mock pattern foundations
- [Source: eusolicit-docs/implementation-artifacts/7-5-ai-draft-generation-backend-sse-integration.md] — `AiGatewayClient`, structlog pattern, `_assert_company_owns_proposal` ownership check pattern
- [Source: eusolicit-app/services/client-api/src/client_api/services/ai_gateway_client.py] — `run_agent()` async method signature (returns `dict`, raises `AiGatewayTimeoutError` / `AiGatewayUnavailableError`)
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py] — established patterns for `require_role`, `get_current_user`, session dependency injection
- [Source: eusolicit-app/services/client-api/src/client_api/schemas/proposal_agent_results.py] — `NullResultResponse` definition (import from here, do NOT redefine)
- [Source: eusolicit-docs/planning-artifacts/epics/E04-ai-gateway-service.md#S04.03] — `agents.yaml` logical name registry; agent names: `pricing-assistant`, `win-theme-extractor`

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

### Completion Notes List

### File List

## Known Deviations

### Detected by `3c-tea-test-review` at 2026-04-17T23:25:29Z

- The tests `test_pricing_assist_requires_bid_manager_role` and `test_win_themes_requires_bid_manager_role` claim to verify the `bid_manager` role requirement. However, they only send an unauthenticated request and expect a 401 Unauthorized response. This only verifies that the endpoint requires authentication, not that it actually enforces the `bid_manager` role. Tests using an authenticated user without the `bid_manager` role (expecting a 403 Forbidden) are missing. Furthermore, there are no tests verifying that non-bid-manager users can successfully access the GET endpoints as stated in AC3.
