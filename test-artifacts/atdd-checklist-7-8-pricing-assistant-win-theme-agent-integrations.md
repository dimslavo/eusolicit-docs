---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-18'
workflowType: 'testarch-atdd'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/7-8-pricing-assistant-win-theme-agent-integrations.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-07.md'
  - 'eusolicit-app/services/client-api/tests/api/test_proposal_compliance_risk_scoring.py'
  - 'eusolicit/_bmad/bmm/config.yaml'
---

# ATDD Checklist — Epic 7, Story 7.8: Pricing Assistant & Win Theme Agent Integrations

**Date:** 2026-04-18
**Author:** TEA Master Test Architect
**Primary Test Level:** Integration (Backend API)
**TDD Phase:** 🔴 RED — All tests designed to FAIL before implementation

---

## Story Summary

A bid manager needs AI-powered pricing recommendations and win theme analysis for their proposal, enabling competitive pricing and strong differentiator positioning before tender submission. This story adds two new synchronous AI agent integrations (`pricing-assistant` and `win-theme-extractor`) with full CRUD (POST to invoke + GET to retrieve) and JSONB persistence on the `proposals` table. Both agents use `run_agent()` (not `stream_agent()`), share the `_build_agent_payload_base` payload builder from Story 7.7, and follow the same error-handling contract.

**As a** bid manager  
**I want** AI-powered pricing recommendations and win theme analysis  
**So that** I can competitively price my tender and position it with the strongest differentiators before submission

---

## Acceptance Criteria

1. **AC1** — `POST /api/v1/proposals/{proposal_id}/pricing-assist` invokes `pricing-assistant` via `AiGatewayClient.run_agent()` (synchronous, non-SSE). Requires `bid_manager` role; returns 404 for cross-company or non-existent proposal. Payload from `_build_agent_payload_base`. Agent response: `{"recommended_price": float, "market_range": {"min": float, "median": float, "max": float}, "justification": str}`. Result stored in `proposals.pricing_result` JSONB. Returns `HTTP 200` with full pricing result and `priced_at` timestamp.

2. **AC2** — `POST /api/v1/proposals/{proposal_id}/win-themes` invokes `win-theme-extractor` via `AiGatewayClient.run_agent()`. Requires `bid_manager` role; returns 404 for cross-company or non-existent proposal. Payload from `_build_agent_payload_base`. Agent response: `{"themes": [{"title": str, "description": str}, ...]}` (rank-ordered, first = highest priority). Result stored in `proposals.win_themes_result` JSONB. Returns `HTTP 200` with themes list and `analyzed_at` timestamp.

3. **AC3** — `GET /api/v1/proposals/{proposal_id}/pricing-assist` and `GET /api/v1/proposals/{proposal_id}/win-themes` retrieve stored results. Any authenticated company user may call GET (no `bid_manager` requirement). Returns 404 for cross-company or non-existent proposal. If no result generated yet, returns `HTTP 200` with `{"result": null}` (never a 404 for missing result).

4. **AC4** — Error handling: `AiGatewayTimeoutError` → `HTTP 504` with `{"error": "agent_timeout", "retry_after": 30}`. `AiGatewayUnavailableError` → `HTTP 503` with `{"error": "agent_unavailable"}`. Malformed pricing response (missing `recommended_price`) → `HTTP 200` with `{"result": null}` + no DB store. Malformed win-themes response (missing `themes`) → `HTTP 200` with `{"themes": [], "analyzed_at": ...}`. Cross-company or non-existent → `HTTP 404` (never 403).

5. **AC5** — Alembic migration `024_proposal_pricing_win_themes` adds `pricing_result JSONB NULL` and `win_themes_result JSONB NULL` to `client.proposals`. `revision = "024_proposal_pricing_win_themes"`, `down_revision = "023_proposal_agent_results"`. Reversible.

6. **AC6** — Integration tests at `services/client-api/tests/api/test_proposal_pricing_win_themes.py`. All 161 existing tests continue to pass.

---

## Step 1: Preflight & Context

### Stack Detection

**Detected stack:** `backend`

**Indicators found:**
- `pyproject.toml` present at `services/client-api/pyproject.toml`
- No `package.json` with React/Vue/Next dependencies
- No `playwright.config.ts` or `cypress.config.ts`
- Test framework: `pytest` + `pytest-asyncio` + `respx`

**TEA Config Flags:**
- `tea_use_playwright_utils`: N/A (backend)
- `tea_use_pactjs_utils`: N/A (backend)
- `test_stack_type`: backend

### Prerequisites

- ✅ Story 7.8 has clear acceptance criteria (AC1–AC6)
- ✅ Test framework exists: `services/client-api/tests/` with `pytest-asyncio`, `respx`
- ✅ Development environment confirmed available
- ✅ Existing test baseline: 161 tests from Stories 7.1–7.7

### Knowledge Fragments Loaded

**Core (always):**
- `data-factories.md` — Fixture patterns for proposal test data
- `test-quality.md` — Given-When-Then, isolation, determinism
- `test-healing-patterns.md` — Resilient assertion patterns

**Backend (fullstack/backend):**
- `test-levels-framework.md` — Integration level selection for API+DB tests
- `test-priorities-matrix.md` — P0–P3 assignment criteria
- `api-testing-patterns.md` — HTTP status code and response contract testing

---

## Step 2: Generation Mode

**Mode selected:** AI Generation (Backend)

**Rationale:** Backend-only story. All 4 new endpoints are standard API integration tests following the identical pattern established in `test_proposal_compliance_risk_scoring.py` (Story 7.7). No browser recording needed. The respx mock pattern is well-established in this codebase.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenarios Mapping

| AC | Scenario | Priority | Test Level | Test ID |
|----|----------|----------|------------|---------|
| AC1 | POST /pricing-assist nominal: stores result, returns 200 with pricing shape | P1 | Integration | `test_pricing_assist_stores_and_retrieves_result` |
| AC1 | POST /pricing-assist payload contains opportunity + company + existing_sections | P1 | Integration | `test_pricing_assist_opportunity_in_payload` |
| AC1 | POST /pricing-assist without auth → 401 | P1 | Integration | `test_pricing_assist_requires_bid_manager_role` |
| AC2 | POST /win-themes nominal: stores themes, returns 200 with rank order | P1 | Integration | `test_win_themes_stores_and_retrieves_result` |
| AC2 | POST /win-themes rank order preserved through store+retrieve cycle | P1 | Integration | `test_win_themes_rank_order_preserved` |
| AC2 | POST /win-themes payload contains company dict and existing_sections | P1 | Integration | `test_win_themes_company_in_payload` |
| AC2 | POST /win-themes without auth → 401 | P1 | Integration | `test_win_themes_requires_bid_manager_role` |
| AC3 | GET /pricing-assist: fresh proposal → 200 {"result": null} | P1 | Integration | `test_get_pricing_assist_no_result_returns_null` |
| AC3 | GET /pricing-assist: cross-company → 404 | P0 | Integration | `test_get_pricing_assist_cross_company_returns_404` |
| AC3 | GET /pricing-assist: non-existent proposal → 404 | P1 | Integration | `test_get_pricing_assist_nonexistent_proposal_returns_404` |
| AC3 | GET /win-themes: fresh proposal → 200 {"result": null} | P1 | Integration | `test_get_win_themes_no_result_returns_null` |
| AC3 | GET /win-themes: cross-company → 404 | P0 | Integration | `test_get_win_themes_cross_company_returns_404` |
| AC3 | GET /win-themes: non-existent proposal → 404 | P1 | Integration | `test_get_win_themes_nonexistent_proposal_returns_404` |
| AC4 | POST /pricing-assist: malformed response → 200 {"result": null}, pricing_result NULL in DB | P1 | Integration | `test_pricing_assist_malformed_response_returns_null` |
| AC4 | POST /pricing-assist: AiGatewayTimeoutError → 504 {"error": "agent_timeout", "retry_after": 30} | P2 | Integration | `test_pricing_assist_timeout_returns_504` |
| AC4 | POST /pricing-assist: AiGatewayUnavailableError → 503 {"error": "agent_unavailable"} | P2 | Integration | `test_pricing_assist_gateway_unavailable_returns_503` |
| AC4 | POST /win-themes: malformed response (no 'themes' key) → 200 with empty themes list | P1 | Integration | `test_win_themes_malformed_response_returns_empty_themes` |
| AC4 | POST /win-themes: AiGatewayTimeoutError → 504 | P2 | Integration | `test_win_themes_timeout_returns_504` |
| AC4 | POST /pricing-assist: cross-company → 404 | P0 | Integration | `test_pricing_assist_cross_company_returns_404` |
| AC4 | POST /win-themes: cross-company → 404 | P0 | Integration | `test_win_themes_cross_company_returns_404` |
| AC5 | Migration 024: pricing_result column exists and is nullable | P2 | Integration | `test_proposals_has_pricing_result_column` |
| AC5 | Migration 024: win_themes_result column exists and is nullable | P2 | Integration | `test_proposals_has_win_themes_result_column` |
| AC5 | Fresh proposal: both new columns default to NULL | P2 | Integration | `test_new_columns_default_to_null` |

**Priority Breakdown:**
- P0: 4 tests (cross-company RLS — E07-R-003)
- P1: 13 tests (nominal paths, GET null, malformed, auth)
- P2: 6 tests (timeouts, 503, migration schema checks)
- **Total: 23 tests**

### Red Phase Requirements

All 23 tests are designed to FAIL before implementation because:
1. `POST /api/v1/proposals/{id}/pricing-assist` endpoint does not exist → 404/405
2. `POST /api/v1/proposals/{id}/win-themes` endpoint does not exist → 404/405
3. `GET /api/v1/proposals/{id}/pricing-assist` endpoint does not exist → 404/405
4. `GET /api/v1/proposals/{id}/win-themes` endpoint does not exist → 404/405
5. `pricing_result` column does not exist in `client.proposals` → DB query errors
6. `win_themes_result` column does not exist in `client.proposals` → DB query errors

---

## Step 4: Failing Tests Generated (RED Phase)

### Integration Tests — 23 tests

**File:** `services/client-api/tests/api/test_proposal_pricing_win_themes.py`

**Shared fixtures:**
- `aigw_url_env` — session-scoped autouse; sets `CLIENT_API_AIGW_BASE_URL=http://test-aigw:8000`; clears `get_settings()` and `get_ai_gateway_client()` LRU caches
- `pricing_win_themes_proposal` — per-test; registers + logs in bid_manager user; creates proposal with 3 seeded sections; yields `(client, session, token, proposal_id)`
- `_make_pipeline_session_override()` — mocks pipeline read-only session (no real pipeline DB in CI)

---

#### Class: `TestPricingAssist` (AC1 — POST /pricing-assist)

- 🔴 **`test_pricing_assist_stores_and_retrieves_result`**
  - **Status:** RED — `POST /api/v1/proposals/{id}/pricing-assist` endpoint not implemented
  - **Verifies:** HTTP 200 with `recommended_price`, `market_range` (min/median/max), `justification`, `priced_at`; DB `pricing_result` JSONB populated; GET retrieves same data
  - **Coverage:** E07-P1-015, AC1, AC3
  - **Mock:** `respx.post("http://test-aigw:8000/agents/pricing-assistant/run")`

- 🔴 **`test_pricing_assist_opportunity_in_payload`**
  - **Status:** RED — endpoint not implemented; respx call never happens
  - **Verifies:** Agent payload includes `opportunity`, `company`, `existing_sections` (3 seeded sections), `proposal_id` from `_build_agent_payload_base`
  - **Coverage:** AC1 (payload structure)

- 🔴 **`test_pricing_assist_malformed_response_returns_null`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Agent returns `{"status": "ok", "data": "no price"}` (missing `recommended_price`); response is `{"result": null}`; DB `pricing_result` remains NULL (no partial store)
  - **Coverage:** AC4 (malformed pricing → null, no DB store)
  - **Advisory:** Includes direct DB assertion for NULL state (Story 7.7 review lesson)

- 🔴 **`test_pricing_assist_cross_company_returns_404`**
  - **Status:** RED — endpoint not implemented (would get 404/405, not cross-company 404)
  - **Verifies:** Company B JWT → 404 (not 403) for Company A's proposal; UUID enumeration prevention
  - **Coverage:** E07-P0-005, AC1, E07-R-003

- 🔴 **`test_pricing_assist_timeout_returns_504`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** `httpx.TimeoutException` → 504 with `{"error": "agent_timeout", "retry_after": 30}`; DB remains NULL
  - **Coverage:** E07-P2-005, AC4, E07-R-007
  - **Mock:** `side_effect=httpx.TimeoutException(...)`

- 🔴 **`test_pricing_assist_gateway_unavailable_returns_503`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** AI Gateway 503 → HTTP 503 with `{"error": "agent_unavailable"}`
  - **Coverage:** AC4

- 🔴 **`test_pricing_assist_requires_bid_manager_role`**
  - **Status:** RED — endpoint not implemented (returns 404, not 401)
  - **Verifies:** No Authorization header → 401
  - **Coverage:** AC1 (bid_manager role requirement)

---

#### Class: `TestGetPricingAssist` (AC3 — GET /pricing-assist)

- 🔴 **`test_get_pricing_assist_no_result_returns_null`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Fresh proposal (no POST called) → 200 with `{"result": null}`; never 404 for missing result
  - **Coverage:** AC3

- 🔴 **`test_get_pricing_assist_cross_company_returns_404`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Company B JWT → 404 for Company A's proposal (E07-R-003)
  - **Coverage:** E07-P0-005, AC3, E07-R-003

- 🔴 **`test_get_pricing_assist_nonexistent_proposal_returns_404`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Random UUID → 404
  - **Coverage:** AC3

---

#### Class: `TestWinThemes` (AC2 — POST /win-themes)

- 🔴 **`test_win_themes_stores_and_retrieves_result`**
  - **Status:** RED — `POST /api/v1/proposals/{id}/win-themes` endpoint not implemented
  - **Verifies:** HTTP 200 with `themes` list (3 items) and `analyzed_at`; each theme has `title` and `description`; first theme is highest priority; DB `win_themes_result` JSONB populated; GET retrieves 3 themes
  - **Coverage:** E07-P1-016, AC2, AC3
  - **Mock:** `respx.post("http://test-aigw:8000/agents/win-theme-extractor/run")`

- 🔴 **`test_win_themes_rank_order_preserved`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Themes A, B, C in POST response and GET response are in exact same order; no alphabetical or other sorting applied
  - **Coverage:** AC2 (rank-ordered, first = highest priority)

- 🔴 **`test_win_themes_company_in_payload`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Agent payload includes `company` dict (with `description` as company strengths), `opportunity`, `existing_sections`, `proposal_id`
  - **Coverage:** AC2 (payload built from `_build_agent_payload_base`)

- 🔴 **`test_win_themes_malformed_response_returns_empty_themes`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Agent returns `{"result": "ok", "status": "no themes"}` (missing `themes` key); response is `{"themes": [], "analyzed_at": ...}`; no crash
  - **Coverage:** AC4 (malformed win-themes → empty list, not null)

- 🔴 **`test_win_themes_cross_company_returns_404`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Company B JWT → 404 (not 403) for Company A's proposal
  - **Coverage:** E07-P0-005, AC2, E07-R-003

- 🔴 **`test_win_themes_timeout_returns_504`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** `httpx.TimeoutException` → 504 with `{"error": "agent_timeout", "retry_after": 30}`
  - **Coverage:** E07-P2-005, AC4, E07-R-007

- 🔴 **`test_win_themes_requires_bid_manager_role`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** No Authorization header → 401
  - **Coverage:** AC2 (bid_manager role requirement)

---

#### Class: `TestGetWinThemes` (AC3 — GET /win-themes)

- 🔴 **`test_get_win_themes_no_result_returns_null`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Fresh proposal → 200 with `{"result": null}`; never 404 for missing result
  - **Coverage:** AC3

- 🔴 **`test_get_win_themes_cross_company_returns_404`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Company B JWT → 404 for Company A's proposal
  - **Coverage:** E07-P0-005, AC3, E07-R-003

- 🔴 **`test_get_win_themes_nonexistent_proposal_returns_404`**
  - **Status:** RED — endpoint not implemented
  - **Verifies:** Random UUID → 404
  - **Coverage:** AC3

---

#### Class: `TestMigrationSchema` (AC5 — Migration 024)

- 🔴 **`test_proposals_has_pricing_result_column`**
  - **Status:** RED — migration 024 not applied
  - **Verifies:** `information_schema.columns` shows `pricing_result` column in `client.proposals`; `is_nullable = 'YES'`
  - **Coverage:** AC5

- 🔴 **`test_proposals_has_win_themes_result_column`**
  - **Status:** RED — migration 024 not applied
  - **Verifies:** `information_schema.columns` shows `win_themes_result` column in `client.proposals`; `is_nullable = 'YES'`
  - **Coverage:** AC5

- 🔴 **`test_new_columns_default_to_null`**
  - **Status:** RED — migration 024 not applied
  - **Verifies:** Fresh proposal has `pricing_result IS NULL` and `win_themes_result IS NULL`; migration is purely additive
  - **Coverage:** AC5

---

## Mock Requirements

### AI Gateway — Pricing Assistant

**Endpoint:** `POST http://test-aigw:8000/agents/pricing-assistant/run`

**Nominal response:**
```json
{
  "recommended_price": 485000.0,
  "market_range": {
    "min": 400000.0,
    "median": 480000.0,
    "max": 600000.0
  },
  "justification": "Based on comparable public sector tenders over the last 12 months..."
}
```

**Malformed response (triggers safe default):**
```json
{"status": "ok", "data": "no price"}
```

**Timeout:** `side_effect=httpx.TimeoutException("Connection timed out after 30s")`

**Unavailable:** `return_value=httpx.Response(503, json={"error": "Service temporarily unavailable"})`

---

### AI Gateway — Win Theme Extractor

**Endpoint:** `POST http://test-aigw:8000/agents/win-theme-extractor/run`

**Nominal response:**
```json
{
  "themes": [
    {
      "title": "Digital Transformation Expertise",
      "description": "10 years delivering public sector digital services..."
    },
    {
      "title": "Cost Efficiency",
      "description": "Proven track record of 15% below-budget delivery..."
    },
    {
      "title": "Social Value Commitment",
      "description": "Committed to 50 apprenticeships per £1M contract value..."
    }
  ]
}
```

**Malformed response (triggers empty themes):**
```json
{"result": "ok", "status": "no themes"}
```

**Timeout:** `side_effect=httpx.TimeoutException("Connection timed out after 30s")`

---

## Fixtures

### `pricing_win_themes_proposal` Fixture

**File:** `services/client-api/tests/api/test_proposal_pricing_win_themes.py`

**Provides:** `(httpx.AsyncClient, AsyncSession, access_token: str, proposal_id: str)`

**Setup:**
1. Clears `get_settings()` and `get_ai_gateway_client()` LRU caches
2. Registers a bid_manager user + unique company
3. Verifies email via direct SQL (`UPDATE client.users SET email_verified = TRUE`)
4. Logs in and obtains JWT access token
5. Creates proposal via `POST /api/v1/proposals`
6. Seeds current version with 3 sections via direct SQL

**Cleanup:** Calls `session.rollback()` + clears LRU caches in `finally` block

### `aigw_url_env` Fixture (session-scoped, autouse)

Sets `CLIENT_API_AIGW_BASE_URL=http://test-aigw:8000` so all `run_agent()` calls target the respx mock. Clears `get_settings()` and `get_ai_gateway_client()` LRU caches. Pattern matches `test_proposal_compliance_risk_scoring.py` exactly.

---

## Implementation Checklist

### To make `TestMigrationSchema` pass (Task 1 + Task 2)

- [ ] Create `services/client-api/alembic/versions/024_proposal_pricing_win_themes.py`
  - `revision = "024_proposal_pricing_win_themes"` (31 chars — fits VARCHAR(32))
  - `down_revision = "023_proposal_agent_results"` (verify against actual file)
  - `op.add_column("proposals", sa.Column("pricing_result", JSONB(), nullable=True), schema="client")`
  - `op.add_column("proposals", sa.Column("win_themes_result", JSONB(), nullable=True), schema="client")`
  - `downgrade()` drops both columns
- [ ] Add to `services/client-api/src/client_api/models/proposal.py`:
  - `pricing_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)`
  - `win_themes_result: Mapped[dict | None] = mapped_column(JSONB, nullable=True)`
- [ ] Run: `alembic upgrade head && alembic downgrade -1 && alembic upgrade head`
- [ ] Verify: `pytest tests/api/test_proposal_pricing_win_themes.py::TestMigrationSchema -v`

### To make `TestPricingAssist` and `TestGetPricingAssist` pass (Tasks 3, 4, 5)

- [ ] Create `services/client-api/src/client_api/schemas/proposal_pricing_win_themes.py`:
  - `MarketRange(BaseModel)`: `min`, `median`, `max` (floats)
  - `PricingAssistResponse(BaseModel)`: `recommended_price`, `market_range`, `justification`, `priced_at`
  - `WinTheme(BaseModel)`: `title`, `description`
  - `WinThemesResponse(BaseModel)`: `themes: list[WinTheme]`, `analyzed_at`
- [ ] Add to `services/client-api/src/client_api/services/proposal_service.py`:
  - `run_pricing_assist(proposal_id, current_user, gw_client, session, pipeline_session)`:
    - Ownership check (404 for cross-company/non-existent)
    - `payload = await _build_agent_payload_base(...)`
    - `result = await gw_client.run_agent("pricing-assistant", payload)`
    - `AiGatewayTimeoutError` → raise HTTPException 504
    - `AiGatewayUnavailableError` → raise HTTPException 503
    - Missing `recommended_price` → log warning, return `None` (no DB store)
    - Nominal: persist `proposal.pricing_result = {...}`, `await session.flush()`
    - Return `PricingAssistResponse` or `None`
  - `get_pricing_assist_result(proposal_id, current_user, session)`:
    - Ownership check, return `proposal.pricing_result` (may be `None`)
- [ ] Add to `services/client-api/src/client_api/api/v1/proposals.py`:
  - `POST /{proposal_id}/pricing-assist` (requires `bid_manager` role):
    - Calls `run_pricing_assist(...)`; returns `NullResultResponse()` if service returns `None`
  - `GET /{proposal_id}/pricing-assist` (requires `get_current_user`):
    - Calls `get_pricing_assist_result(...)`; returns `NullResultResponse()` if `None`
- [ ] Verify: `pytest tests/api/test_proposal_pricing_win_themes.py::TestPricingAssist tests/api/test_proposal_pricing_win_themes.py::TestGetPricingAssist -v`

### To make `TestWinThemes` and `TestGetWinThemes` pass (Tasks 3, 4, 5)

- [ ] Add to `services/client-api/src/client_api/services/proposal_service.py`:
  - `run_win_themes(proposal_id, current_user, gw_client, session, pipeline_session)`:
    - Ownership check (404 for cross-company/non-existent)
    - `payload = await _build_agent_payload_base(...)`
    - `result = await gw_client.run_agent("win-theme-extractor", payload)`
    - `AiGatewayTimeoutError` → raise HTTPException 504
    - `AiGatewayUnavailableError` → raise HTTPException 503
    - Missing/malformed `themes` → log warning, return `([], analyzed_at)` (empty list safe default)
    - Nominal: parse `result.get("themes", [])`, skip malformed items, preserve order
    - Persist `proposal.win_themes_result = {"themes": [...], "analyzed_at": ...}`
    - `await session.flush()`; return `(themes, analyzed_at)`
  - `get_win_themes_result(proposal_id, current_user, session)`:
    - Ownership check, return `proposal.win_themes_result` (may be `None`)
- [ ] Add to `services/client-api/src/client_api/api/v1/proposals.py`:
  - `POST /{proposal_id}/win-themes` (requires `bid_manager` role):
    - Calls `run_win_themes(...)`; returns `WinThemesResponse(themes=themes, analyzed_at=analyzed_at)`
  - `GET /{proposal_id}/win-themes` (requires `get_current_user`):
    - Calls `get_win_themes_result(...)`; returns `NullResultResponse()` if `None`
- [ ] Verify: `pytest tests/api/test_proposal_pricing_win_themes.py::TestWinThemes tests/api/test_proposal_pricing_win_themes.py::TestGetWinThemes -v`

---

## Running Tests

```bash
cd eusolicit-app/services/client-api

# New story tests only (23 tests):
pytest tests/api/test_proposal_pricing_win_themes.py -v

# Specific test classes:
pytest tests/api/test_proposal_pricing_win_themes.py::TestPricingAssist -v
pytest tests/api/test_proposal_pricing_win_themes.py::TestWinThemes -v
pytest tests/api/test_proposal_pricing_win_themes.py::TestGetPricingAssist -v
pytest tests/api/test_proposal_pricing_win_themes.py::TestGetWinThemes -v
pytest tests/api/test_proposal_pricing_win_themes.py::TestMigrationSchema -v

# Full regression suite (all 161 existing + 23 new = 184 tests):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py \
       tests/api/test_proposal_compliance_risk_scoring.py \
       tests/api/test_proposal_pricing_win_themes.py -v
```

---

## Red-Green-Refactor Workflow

### 🔴 RED Phase (Complete) ✅

**TEA Agent Responsibilities:**

- ✅ All 23 tests written and failing (RED phase)
- ✅ Fixture `pricing_win_themes_proposal` designed (pattern from Story 7.7)
- ✅ Mock requirements documented (`respx` for both AI agents)
- ✅ No data-testid requirements (backend-only story)
- ✅ Implementation checklist created with ordered tasks

**Expected Failure Reasons Before Implementation:**
- Migration tests fail → `pricing_result`/`win_themes_result` columns absent from schema query
- POST/GET endpoint tests fail → 404/405 responses (endpoints not registered)
- Cross-company tests fail → same 404/405 (but for wrong reason — route missing)
- Timeout/503 tests fail → 404/405 (endpoints not registered)

---

### 🟢 GREEN Phase (DEV Team — Next Steps)

1. **Start with migration** (`TestMigrationSchema` — easiest to make green first)
   - Creates `024_proposal_pricing_win_themes.py` migration
   - Adds ORM columns to `Proposal` model
   - Run `pytest tests/api/test_proposal_pricing_win_themes.py::TestMigrationSchema -v`

2. **Add schemas** (`proposal_pricing_win_themes.py` — no dependencies)
   - `MarketRange`, `PricingAssistResponse`, `WinTheme`, `WinThemesResponse`

3. **Add service functions** (`proposal_service.py`)
   - `run_pricing_assist` + `get_pricing_assist_result`
   - `run_win_themes` + `get_win_themes_result`

4. **Add router endpoints** (`proposals.py`)
   - All 4 endpoints with correct role guards and dependency injection
   - Run: `pytest tests/api/test_proposal_pricing_win_themes.py -v`

5. **Run full regression**
   - Ensure all 161 existing tests remain green
   - `pytest tests/api/test_proposal*.py -v`

---

### 🔵 REFACTOR Phase (After All Tests Pass)

- Verify no duplication with Story 7.7 service code (particularly payload building)
- Confirm `_build_agent_payload_base` is reused, not duplicated
- Ensure `from __future__ import annotations` at top of all new files
- Verify `structlog` used for all logging (never `logging`)
- Confirm `session.flush()` (never `session.commit()`) in service functions
- Run `pytest tests/api/test_proposal_pricing_win_themes.py -v` — all 23 must pass

---

## Key Architecture Constraints

| Constraint | Implementation Rule |
|-----------|---------------------|
| `run_agent()` only | Both agents synchronous — never use `stream_agent()` |
| `_build_agent_payload_base` | Reuse from Story 7.7 — do NOT duplicate payload building |
| `NullResultResponse` | Import from `proposal_agent_results.py` — do NOT redefine |
| `session.flush()` | Never `session.commit()` — route-scoped session |
| 404 not 403 | Cross-company → 404 prevents UUID enumeration (E07-R-003) |
| `require_role("bid_manager")` | POST endpoints only; GET uses `get_current_user` |
| `company_id` from JWT only | Never accept from request body |
| Malformed pricing → `None` return | Service returns `None`; router returns `NullResultResponse()` |
| Malformed win-themes → empty list | Service returns `([], analyzed_at)`; always valid `WinThemesResponse` |
| Revision ID: 31 chars | `"024_proposal_pricing_win_themes"` fits VARCHAR(32) ✅ |
| `down_revision` | Must be `"023_proposal_agent_results"` (not `"023_proposal_agent_results_columns"`) |

---

## Test Coverage Map (from Epic 7 Test Design)

| Test Design ID | Priority | Story 7.8 Test |
|----------------|----------|----------------|
| E07-P1-015 | P1 | `TestPricingAssist.test_pricing_assist_stores_and_retrieves_result` |
| E07-P1-016 | P1 | `TestWinThemes.test_win_themes_stores_and_retrieves_result` |
| E07-P0-005 | P0 | `test_pricing_assist_cross_company_returns_404`, `test_win_themes_cross_company_returns_404`, `test_get_pricing_assist_cross_company_returns_404`, `test_get_win_themes_cross_company_returns_404` |
| E07-P2-005 | P2 | `test_pricing_assist_timeout_returns_504`, `test_win_themes_timeout_returns_504` |
| E07-R-003 | Risk | All 4 cross-company 404 tests |
| E07-R-007 | Risk | Both timeout 504 tests |

---

## Step 5: Validation

### Checklist Validation

- ✅ Prerequisites satisfied (story approved, test framework exists)
- ✅ Test file created at correct path
- ✅ No temp artifacts in random locations (no CLI sessions needed — backend only)
- ✅ 23 integration tests covering all 6 ACs
- ✅ Tests designed to fail: endpoints do not exist → 404/405 failures
- ✅ Migration schema tests fail: columns absent → `information_schema` returns no rows
- ✅ Checklist matches acceptance criteria (AC1–AC6 all covered)
- ✅ RED phase: no `pytest.mark.skip` needed — tests fail organically from missing implementation

### Advisory Items Carried Forward (from Story 7.7 Review)

1. **DB NULL assertion for malformed pricing** — `test_pricing_assist_malformed_response_returns_null` includes direct SQL assertion that `pricing_result IS NULL` after malformed response. This was missing from Story 7.7.

2. **`aigw_url_env` LRU cache pattern** — Uses `os.environ[key] = value` (not `setdefault`) with `get_settings.cache_clear()` AND `get_ai_gateway_client.cache_clear()`. Matches Story 7.7 pattern exactly.

3. **Pricing vs win-themes safe default divergence** — Pricing returns `None` (router returns `NullResultResponse`) because `recommended_price` is a required float; win-themes returns `[]` because it's a list that can be empty. Tests assert these distinct behaviors.

---

## Knowledge Base References Applied

- **`api-testing-patterns.md`** — HTTP status code contract testing, respx mock patterns
- **`data-factories.md`** — Fixture patterns for test isolation and cleanup
- **`test-quality.md`** — Given-When-Then, single assertion focus, determinism
- **`test-levels-framework.md`** — Integration level selection (API+DB, no E2E for backend)
- **`test-priorities-matrix.md`** — P0 for RLS/security; P1 for nominal paths; P2 for edge cases
- **`test-healing-patterns.md`** — Resilient assertions (not brittle exact-match on IDs)

---

**Generated by BMad TEA Agent** — 2026-04-18
