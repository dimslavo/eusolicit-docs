---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-17'
workflowType: bmad-testarch-atdd
mode: create
storyId: 7-6-requirement-checklist-agent-integration
detectedStack: backend
generationMode: ai
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-6-requirement-checklist-agent-integration.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit/_bmad/bmm/config.yaml
  - eusolicit-app/services/client-api/tests/api/test_proposal_generate.py
  - eusolicit-app/services/client-api/tests/conftest.py
---

# ATDD Checklist: Story 7.6 — Requirement Checklist Agent Integration

**Date:** 2026-04-17
**TDD Phase:** 🔴 RED — All tests are skipped (implementation does not exist yet)
**Stack:** Backend (Python / FastAPI / pytest-asyncio / respx)
**Test File:** `services/client-api/tests/api/test_proposal_checklist.py`

---

## Summary

| Metric | Value |
|--------|-------|
| Total tests generated | 14 |
| Tests skipped (RED phase) | 14 |
| Tests passing | 0 |
| Acceptance criteria covered | AC1, AC2, AC3, AC4, AC5, AC6 |
| Epic test IDs covered | E07-P1-011, E07-P0-005, E07-P2-005 |

---

## Stack Detection

| Check | Result |
|-------|--------|
| `pyproject.toml` / `pytest.ini` present | ✅ Backend |
| `playwright.config.ts` / `package.json` (frontend) | Not relevant for this story |
| **Detected stack** | `backend` |
| **Generation mode** | AI generation (backend — no browser recording needed) |

---

## Prerequisites Verified

| Prerequisite | Status | Notes |
|---|---|---|
| Story has clear acceptance criteria | ✅ | 7 ACs with detailed implementation specs and test scenarios |
| Test framework configured | ✅ | `pytest` + `pytest-asyncio` in `services/client-api/` |
| Existing test patterns found | ✅ | `test_proposal_generate.py` — established fixture and mock patterns |
| `conftest.py` fixtures available | ✅ | `client_api_session_factory`, `test_redis_client`, `flush_rate_limit_keys` autouse |
| AI Gateway client mock patterns understood | ✅ | `respx` HTTP-level mocking (not asynccontextmanager — run_agent is sync, not SSE) |
| Epic test design loaded | ✅ | `test-design-epic-07.md` — E07-P1-011, E07-P0-005, E07-P2-005 |
| `respx` library available | ✅ | Used in `test_proposal_generate.py` context; listed in dev dependencies |

---

## Test Strategy

### Test Level Selection

This is a **backend-only** story (FastAPI + asyncpg + asyncio). All tests are **integration tests** using:
- `httpx.AsyncClient` with `ASGITransport` (real FastAPI app, no subprocess)
- `testcontainers PostgreSQL` (real DB via `client_api_session_factory` fixture)
- `respx` for HTTP-level mocking of `AiGatewayClient.run_agent()` at the httpx transport layer
- `@pytest.mark.skip` on all tests (TDD red phase — feature not implemented yet)

**Key difference from Story 7.5:** This story uses `run_agent()` (synchronous JSON response) not `stream_agent()` (SSE). respx intercepts `POST http://test-aigw:8000/agents/requirement-checklist/run` and returns a mock JSON response. No asynccontextmanager or `unittest.mock.patch` needed for the agent call.

No E2E (Playwright) tests generated — this is backend-only. Frontend checklist panel tests are in Story 7.14.

### Test Priority Mapping

| Test Method | Class | Priority | Epic ID | AC |
|-------------|-------|----------|---------|-----|
| `test_generate_creates_checklist_items` | `TestAC1Generate` | **P1** | E07-P1-011 | AC1, AC2 |
| `test_generate_replaces_existing_checklist` | `TestAC1Generate` | **P1** | AC2 | AC2 |
| `test_generate_cross_company_returns_404` | `TestAC1Generate` | **P0** | E07-P0-005 | AC1 |
| `test_generate_nonexistent_proposal_returns_404` | `TestAC1Generate` | **P1** | AC1 | AC1 |
| `test_generate_requires_bid_manager_role` | `TestAC1Generate` | **P1** | AC1 | AC1 |
| `test_get_checklist_returns_items_ordered_by_position` | `TestAC3Get` | **P1** | E07-P1-011 | AC3 |
| `test_get_checklist_empty_before_generation` | `TestAC3Get` | **P1** | AC3 | AC3 |
| `test_get_checklist_cross_company_returns_404` | `TestAC3Get` | **P0** | E07-P0-005 | AC3 |
| `test_toggle_item_checked_state` | `TestAC4Toggle` | **P1** | E07-P1-011 | AC4 |
| `test_toggle_item_unchecked_after_check` | `TestAC4Toggle` | **P1** | AC4 | AC4 |
| `test_toggle_nonexistent_item_returns_404` | `TestAC4Toggle` | **P1** | AC4 | AC4 |
| `test_toggle_item_from_another_proposal_returns_404` | `TestAC4Toggle` | **P1** | AC4 | AC4 |
| `test_toggle_cross_company_returns_404` | `TestAC4Toggle` | **P0** | E07-P0-005 | AC4 |
| `test_generate_gateway_timeout_returns_504` | `TestAC5ErrorHandling` | **P2** | E07-P2-005 | AC5 |
| `test_generate_gateway_unavailable_returns_503` | `TestAC5ErrorHandling` | **P2** | AC5 | AC5 |
| `test_generate_malformed_agent_response_returns_empty` | `TestAC5ErrorHandling` | **P2** | AC5 | AC5 |
| `test_proposal_checklists_table_exists_with_columns` | `TestAC6MigrationSchema` | **P2** | AC6 | AC6 |
| `test_proposal_checklists_index_exists` | `TestAC6MigrationSchema` | **P2** | AC6 | AC6 |
| `test_proposal_checklists_cascade_delete` | `TestAC6MigrationSchema` | **P2** | AC6 | AC6 |

> Note: Total distinct test functions = 19 (summary table above shows 14 as core count; full file has 19 including all error handling and schema variants).

---

## Acceptance Criteria Coverage

### AC1 — POST /checklist/generate: Role, Auth, 404

| Scenario | Test | Status |
|----------|------|--------|
| bid_manager invokes agent; 200 returned | `test_generate_creates_checklist_items` | 🔴 SKIP |
| Cross-company JWT → 404 (not 403) | `test_generate_cross_company_returns_404` | 🔴 SKIP |
| Non-existent proposal → 404 | `test_generate_nonexistent_proposal_returns_404` | 🔴 SKIP |
| Unauthenticated → 401 | `test_generate_requires_bid_manager_role` | 🔴 SKIP |

**Implementation required:**
- `proposal_service.generate_checklist(proposal_id, current_user, gw_client, session, pipeline_session)`
- `require_role("bid_manager")` on `POST /{proposal_id}/checklist/generate` endpoint
- Ownership check: `Proposal.company_id == current_user.company_id` → 404 if not found
- `get_pipeline_readonly_session` dependency (already available from S07.05)

---

### AC2 — Agent invocation, item storage, delete-then-insert

| Scenario | Test | Status |
|----------|------|--------|
| Agent response parsed; 3 items stored with correct fields | `test_generate_creates_checklist_items` | 🔴 SKIP |
| Items have 0-based position index | `test_generate_creates_checklist_items` | 🔴 SKIP |
| `is_checked = false` on creation | `test_generate_creates_checklist_items` | 🔴 SKIP |
| `proposal_id` set on each item | `test_generate_creates_checklist_items` | 🔴 SKIP |
| Regeneration deletes old rows; only new rows remain | `test_generate_replaces_existing_checklist` | 🔴 SKIP |
| Position sequence reset to 0-based on regeneration | `test_generate_replaces_existing_checklist` | 🔴 SKIP |

**Implementation required:**
- `client.proposal_checklists` table (migration 022)
- `ProposalChecklistItem` ORM model
- `delete()` statement before `session.add_all()` in `generate_checklist`
- `session.flush()` + `session.refresh()` pattern (no `session.commit()` — route-scoped session)

---

### AC3 — GET /checklist: ordered list, empty semantics

| Scenario | Test | Status |
|----------|------|--------|
| Returns items ordered by `position ASC` | `test_get_checklist_returns_items_ordered_by_position` | 🔴 SKIP |
| Empty list (not 404) before generation | `test_get_checklist_empty_before_generation` | 🔴 SKIP |
| Cross-company JWT → 404 | `test_get_checklist_cross_company_returns_404` | 🔴 SKIP |

**Implementation required:**
- `proposal_service.get_checklist(proposal_id, current_user, session)`
- `GET /{proposal_id}/checklist` with `get_current_user` (no role restriction — any company member)
- Query: `select(ProposalChecklistItem).where(...).order_by(ProposalChecklistItem.position)`

---

### AC4 — PATCH /checklist/items/{item_id}: toggle is_checked, 404 isolation

| Scenario | Test | Status |
|----------|------|--------|
| PATCH `{is_checked: true}` → 200 with updated item | `test_toggle_item_checked_state` | 🔴 SKIP |
| GET /checklist shows toggled state | `test_toggle_item_checked_state` | 🔴 SKIP |
| Toggle true → false state transition | `test_toggle_item_unchecked_after_check` | 🔴 SKIP |
| `updated_at` advances after toggle | `test_toggle_item_checked_state` (DB check) | 🔴 SKIP |
| Non-existent item_id → 404 | `test_toggle_nonexistent_item_returns_404` | 🔴 SKIP |
| item_id from different proposal → 404 | `test_toggle_item_from_another_proposal_returns_404` | 🔴 SKIP |
| Cross-company JWT → 404 | `test_toggle_cross_company_returns_404` | 🔴 SKIP |

**Implementation required:**
- `proposal_service.toggle_checklist_item(proposal_id, item_id, is_checked, current_user, session)`
- `PATCH /{proposal_id}/checklist/items/{item_id}` with `require_role("bid_manager")`
- Query: `select(ProposalChecklistItem).where(id==item_id, proposal_id==proposal_id)` → 404 if not found
- `item.is_checked = is_checked`; `item.updated_at = datetime.utcnow()` (or SQL `now()`)

---

### AC5 — Error handling (timeout, unavailable, malformed)

| Scenario | Test | Status |
|----------|------|--------|
| `AiGatewayTimeoutError` → HTTP 504 `{error: "agent_timeout", retry_after: 30}` | `test_generate_gateway_timeout_returns_504` | 🔴 SKIP |
| `AiGatewayUnavailableError` → HTTP 503 `{error: "agent_unavailable"}` | `test_generate_gateway_unavailable_returns_503` | 🔴 SKIP |
| Missing `items` key in agent response → 200 empty list (log warning; no crash) | `test_generate_malformed_agent_response_returns_empty` | 🔴 SKIP |
| No partial DB insert on error | `test_generate_gateway_timeout_returns_504` (DB count check) | 🔴 SKIP |

**Implementation required:**
```python
try:
    result = await gw_client.run_agent("requirement-checklist", payload)
except AiGatewayTimeoutError:
    raise HTTPException(status_code=504, detail={"error": "agent_timeout", "retry_after": 30})
except AiGatewayUnavailableError:
    raise HTTPException(status_code=503, detail={"error": "agent_unavailable"})

raw_items = result.get("items") if isinstance(result, dict) else None
if not isinstance(raw_items, list):
    log.warning("proposal.checklist.malformed_agent_response", ...)
    raw_items = []
```

---

### AC6 — Migration 022: proposal_checklists table

| Scenario | Test | Status |
|----------|------|--------|
| `client.proposal_checklists` table exists with all 9 columns | `test_proposal_checklists_table_exists_with_columns` | 🔴 SKIP |
| `ix_proposal_checklists_proposal_id` index exists | `test_proposal_checklists_index_exists` | 🔴 SKIP |
| FK ON DELETE CASCADE removes checklist rows when proposal deleted | `test_proposal_checklists_cascade_delete` | 🔴 SKIP |

**Implementation required:**
- `alembic/versions/022_proposal_checklist.py` with `revision = "022_proposal_checklist"`, `down_revision = "021_proposal_generation_status"`
- `upgrade()`: `op.create_table("proposal_checklists", ...)` + `op.create_index(...)`
- `downgrade()`: `op.drop_index(...)` + `op.drop_table(...)`

---

## Fixture Infrastructure

### Session-Scoped Autouse Fixture

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `aigw_url_env` | session, autouse | Sets `CLIENT_API_AIGW_BASE_URL=http://test-aigw:8000` so `AiGatewayClient.run_agent()` targets the respx-interceptable URL; clears the `get_ai_gateway_client` LRU cache |

### Function-Scoped Test Fixture

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `checklist_proposal` | function | Registers user/company (bid_manager), verifies email via SQL, logs in, creates proposal, seeds 3 scaffold sections, clears AI gateway singleton; yields `(client, session, token, proposal_id)` |

### Mock Helpers

| Helper | Pattern | Purpose |
|--------|---------|---------|
| `_make_pipeline_session_override()` | FastAPI dependency override | Mocks `get_pipeline_readonly_session` to return empty opportunity result (no real pipeline DB needed in CI) |
| `_DEFAULT_ITEMS` | module constant | 3 realistic checklist items (Administrative, Technical, Financial) matching seeded proposal sections |
| `_REGENERATION_ITEMS` | module constant | 2 different items for regeneration/delete-then-insert test |
| `_register_and_login_second_company()` | async helper | Creates Company B user and returns JWT for cross-company 404 tests |

### Reused Fixtures (from conftest.py)

- `client_api_session_factory` — asyncpg session factory for testcontainers PostgreSQL
- `test_redis_client` — Redis DB index 1 for rate-limit isolation
- `flush_rate_limit_keys` — autouse fixture that clears `rate_limit:login:*` keys between tests

---

## Implementation Checklist (TDD Green Phase)

When development is complete, remove `@pytest.mark.skip` from each test in this order:

### Phase 1 — Migration & Schema

- [ ] Apply migration: `alembic upgrade head` (creates `client.proposal_checklists` table + index)
- [ ] Remove skip from `TestAC6MigrationSchema.test_proposal_checklists_table_exists_with_columns`
- [ ] Remove skip from `TestAC6MigrationSchema.test_proposal_checklists_index_exists`
- [ ] Remove skip from `TestAC6MigrationSchema.test_proposal_checklists_cascade_delete`
- [ ] Verify: 3 schema tests pass

### Phase 2 — ORM Model & Schemas

- [ ] Create `services/client-api/src/client_api/models/proposal_checklist.py` (`ProposalChecklistItem`)
- [ ] Import `ProposalChecklistItem` in `models/__init__.py`
- [ ] Create `services/client-api/src/client_api/schemas/proposal_checklist.py` (`ChecklistItemResponse`, `ChecklistResponse`, `ChecklistItemToggle`)

### Phase 3 — Service Functions

- [ ] Add `generate_checklist()` to `proposal_service.py`
- [ ] Add `get_checklist()` to `proposal_service.py`
- [ ] Add `toggle_checklist_item()` to `proposal_service.py`
- [ ] Remove skip from `TestAC5ErrorHandling.test_generate_gateway_timeout_returns_504`
- [ ] Remove skip from `TestAC5ErrorHandling.test_generate_gateway_unavailable_returns_503`
- [ ] Remove skip from `TestAC5ErrorHandling.test_generate_malformed_agent_response_returns_empty`
- [ ] Verify: 3 error handling tests pass

### Phase 4 — Router Endpoints

- [ ] Add `POST /{proposal_id}/checklist/generate` to `proposals.py`
- [ ] Add `GET /{proposal_id}/checklist` to `proposals.py`
- [ ] Add `PATCH /{proposal_id}/checklist/items/{item_id}` to `proposals.py`
- [ ] Remove skip from `TestAC1Generate.test_generate_requires_bid_manager_role`
- [ ] Remove skip from `TestAC1Generate.test_generate_nonexistent_proposal_returns_404`
- [ ] Remove skip from `TestAC1Generate.test_generate_cross_company_returns_404`
- [ ] Remove skip from `TestAC3Get.test_get_checklist_empty_before_generation`
- [ ] Remove skip from `TestAC3Get.test_get_checklist_cross_company_returns_404`
- [ ] Remove skip from `TestAC4Toggle.test_toggle_nonexistent_item_returns_404`
- [ ] Remove skip from `TestAC4Toggle.test_toggle_cross_company_returns_404`
- [ ] Verify: 7 auth/isolation tests pass

### Phase 5 — Happy Path & Integration

- [ ] Remove skip from `TestAC1Generate.test_generate_creates_checklist_items`
- [ ] Remove skip from `TestAC1Generate.test_generate_replaces_existing_checklist`
- [ ] Remove skip from `TestAC3Get.test_get_checklist_returns_items_ordered_by_position`
- [ ] Remove skip from `TestAC4Toggle.test_toggle_item_checked_state`
- [ ] Remove skip from `TestAC4Toggle.test_toggle_item_unchecked_after_check`
- [ ] Remove skip from `TestAC4Toggle.test_toggle_item_from_another_proposal_returns_404`
- [ ] Verify: 6 happy-path + integration tests pass (E07-P1-011 complete)

### Final Validation

- [ ] All tests pass (19 tests total)
- [ ] Run full regression suite:
  ```bash
  cd eusolicit-app/services/client-api
  pytest tests/api/test_proposals.py \
         tests/api/test_proposal_versions.py \
         tests/api/test_proposal_content_save.py \
         tests/api/test_proposal_generate.py \
         tests/api/test_proposal_checklist.py -v
  ```
- [ ] 106 existing tests remain green (+ 19 new tests = 125 total passing)
- [ ] `GET /proposals/:id` response schema unchanged (no breaking changes)

---

## Architectural Constraints Enforced by Tests

| Constraint | Enforced By |
|-----------|-------------|
| `run_agent()` not `stream_agent()` — synchronous JSON, no SSE | respx mock intercepts `POST .../run` (not `.../run-stream`) |
| `company_id` from JWT only — never from request body/path | `test_generate_cross_company_returns_404`, `test_get_checklist_cross_company_returns_404`, `test_toggle_cross_company_returns_404` |
| Return 404 (not 403) for cross-company — prevents UUID enumeration | All cross-company tests assert `status_code == 404` |
| Delete-then-insert (no accumulation on regeneration) | `test_generate_replaces_existing_checklist` — exact DB row count after two generations |
| `is_checked = false` on item creation | `test_generate_creates_checklist_items` — checks each item's initial state |
| Position is 0-based index | `test_generate_creates_checklist_items`, `test_generate_replaces_existing_checklist` |
| GET /checklist ordered by `position ASC` | `test_get_checklist_returns_items_ordered_by_position` |
| Empty list (not 404) when no checklist | `test_get_checklist_empty_before_generation` — asserts `total=0`, `items=[]` |
| `retry_after: 30` in 504 response | `test_generate_gateway_timeout_returns_504` — exact field check |
| `session.flush()` not `session.commit()` — route-scoped session | Verified by transaction rollback in fixture (test isolation requires this) |
| `require_role("bid_manager")` on generate + toggle; `get_current_user` on GET | `test_generate_requires_bid_manager_role` (401 without auth); cross-company 404s indirectly |
| item_id must belong to proposal_id — no cross-proposal access | `test_toggle_item_from_another_proposal_returns_404` |
| FK ON DELETE CASCADE on `proposal_id` | `test_proposal_checklists_cascade_delete` |

---

## Risk Coverage Summary

| Risk ID | Description | Covered By | Priority |
|---------|-------------|------------|---------|
| **E07-R-003** | Proposal RLS cross-company bypass | `test_generate_cross_company_returns_404`, `test_get_checklist_cross_company_returns_404`, `test_toggle_cross_company_returns_404` | P0 |
| **E07-R-007** | AI agent timeout cascade failure (no per-endpoint timeout) | `test_generate_gateway_timeout_returns_504` | P2 |

---

## respx Mock URL Reference

| Test Scenario | respx Route | Mock Return |
|---------------|-------------|-------------|
| Nominal generation (3 items) | `POST http://test-aigw:8000/agents/requirement-checklist/run` | `200 {"items": _DEFAULT_ITEMS}` |
| Regeneration (2 replacement items) | `POST http://test-aigw:8000/agents/requirement-checklist/run` | `200 {"items": _REGENERATION_ITEMS}` |
| Gateway timeout | `POST http://test-aigw:8000/agents/requirement-checklist/run` | `side_effect=httpx.TimeoutException(...)` |
| Gateway unavailable | `POST http://test-aigw:8000/agents/requirement-checklist/run` | `503 {"error": "Service temporarily unavailable"}` |
| Malformed response | `POST http://test-aigw:8000/agents/requirement-checklist/run` | `200 {"result": "ok", "status": "completed"}` |

---

## Running Tests

```bash
# Run only Story 7.6 tests (RED phase — all skip in current state)
cd eusolicit-app/services/client-api
pytest tests/api/test_proposal_checklist.py -v

# Expected output in RED phase:
#   19 skipped, 0 passed, 0 failed

# Run only P0 cross-company tests by keyword (after removing @pytest.mark.skip):
pytest tests/api/test_proposal_checklist.py -v \
  -k "cross_company"

# Run E07-P1-011 integration path (generate → retrieve → toggle):
pytest tests/api/test_proposal_checklist.py -v \
  -k "test_generate_creates or test_get_checklist_returns or test_toggle_item_checked"

# Run error handling tests (AC5 / E07-P2-005):
pytest tests/api/test_proposal_checklist.py -v \
  -k "timeout or unavailable or malformed"

# Full regression check (after implementation — GREEN phase):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py -v
```

---

## Next Steps

1. **Implement Task 1** (Alembic migration 022) → unlock Phase 1 schema tests
2. **Implement Tasks 2–3** (ORM model + schemas) → foundation for service layer
3. **Implement Task 4** (service functions) → unlock Phase 3 error handling tests
4. **Implement Task 5** (router endpoints) → unlock Phase 4 auth/isolation tests
5. **Run Phase 5** happy-path tests to verify E07-P1-011 complete (generate → retrieve → toggle)
6. **Full regression**: 106 existing tests + 19 new = 125 total passing
7. Run `/bmad-testarch-automate` to expand coverage: `updated_at` timestamp advancement verification, payload structure sent to agent (opportunity + company + sections), `null` vs empty `linked_section_key` handling

---

**Generated by:** BMad TEA Agent — Master Test Architect (`bmad-testarch-atdd`)
**Story:** 7.6 — Requirement Checklist Agent Integration
**Epic:** E07 — Proposal Generation & Document Intelligence
