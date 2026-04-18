---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-18'
workflowType: bmad-testarch-atdd
storyId: 7-9-content-blocks-crud-search-api
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-9-content-blocks-crud-search-api.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/services/client-api/tests/api/test_proposal_pricing_win_themes.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 7.9 — Content Blocks CRUD & Search API

**Date:** 2026-04-18
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED (failing tests generated — implementation pending)
**Test File:** `services/client-api/tests/api/test_content_blocks.py`

---

## Step 1: Preflight & Context

### Stack Detection

| Signal | Found | Result |
|--------|-------|--------|
| `pyproject.toml` (services/client-api) | ✅ | Backend indicators |
| `eusolicit-app/frontend/` with React + Vitest | ✅ | Frontend indicators |
| **Detected stack** | | **`fullstack`** |
| **This story's test level** | | **Backend integration tests (Python pytest-asyncio)** |

Story 7.9 is a pure backend API story. All tests are Python `pytest-asyncio` integration
tests against the FastAPI service using ASGI transport. No E2E or browser tests generated.

### Prerequisites Verified

| Prerequisite | Status | Notes |
|--------------|--------|-------|
| Story approved with clear AC | ✅ | 8 ACs, all acceptance-testable |
| Test framework configured | ✅ | `pyproject.toml` with pytest-asyncio, httpx |
| `conftest.py` fixtures available | ✅ | `client_api_session_factory`, `test_redis_client` |
| Story 7.8 patterns available | ✅ | `test_proposal_pricing_win_themes.py` — fixture and helper patterns |
| Epic test design loaded | ✅ | `test-design-epic-07.md` — E07-P0-006, P1-017/018, P2-006/007, P3-003 |
| Stories 7.1–7.8 complete | ✅ | 184 existing tests, all passing |

### Files Loaded

- Story: `7-9-content-blocks-crud-search-api.md`
- Epic test design: `test-design-epic-07.md`
- Pattern reference: `test_proposal_pricing_win_themes.py`, `conftest.py`

### TEA Config Flags

- `tea_use_playwright_utils`: N/A (backend-only story)
- `tea_use_pactjs_utils`: N/A
- `test_stack_type`: backend (this story)
- No `aigw_url_env` fixture needed (no AI Gateway calls in Story 7.9)

---

## Step 2: Generation Mode

**Selected mode:** AI Generation (Backend stack, no browser recording)

**Rationale:** All 8 new endpoints are pure REST API endpoints testable via `httpx.AsyncClient`
with `ASGITransport`. No AI Gateway calls, no SSE streams, no `respx` mocks needed.
No UI interaction required for this story. Pattern follows Story 7.8 exactly for fixture
structure; simpler because no agent mocking required.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Mapping

| AC | Endpoint(s) | Test Class | Test Count | Priority |
|----|-------------|------------|------------|----------|
| AC1 | `POST /api/v1/content-blocks` | `TestContentBlockCRUD` | 4 | P1 |
| AC2 | `GET /api/v1/content-blocks` | `TestContentBlockListFilters` | 5 | P1/P2 |
| AC3 | `GET /api/v1/content-blocks/search?q=` | `TestContentBlockSearch` | 4 | P0/P1/P3 |
| AC4 | `GET /api/v1/content-blocks/{block_id}` | `TestContentBlockCRUD` | 2 | P1 |
| AC5 | `PATCH /api/v1/content-blocks/{block_id}` | `TestContentBlockCRUD` + `TestContentBlockTsvectorFreshness` | 4 | P1/P2 |
| AC6 | `POST /{block_id}/approve` + `/{block_id}/unapprove` | `TestContentBlockCRUD` | 2 | P1 |
| AC7 | `DELETE /api/v1/content-blocks/{block_id}` | `TestContentBlockCRUD` | 1 | P1 |
| AC8 | Cross-company for all 5 endpoint types | `TestContentBlockRLS` + `TestContentBlockSearch` | 5 + 1 | P0 |

**Total: 27 failing tests**

### Priority Breakdown

- **P0:** 6 tests (RLS cross-company isolation — E07-P0-006)
  - `test_search_cross_company_returns_zero_results` (in `TestContentBlockSearch`)
  - All 5 tests in `TestContentBlockRLS`
- **P1:** 18 tests (nominal CRUD lifecycle, FTS correctness, list, filters)
- **P2:** 2 tests (tsvector freshness — E07-P2-006, E07-R-009)
- **P3:** 1 test (relevance ordering — E07-P3-003)

### Test Levels

All tests are **integration** level:
- Real PostgreSQL via `client_api_session_factory` (testcontainers-backed in CI)
- Real FastAPI ASGI via `httpx.AsyncClient` + `ASGITransport`
- Real JWT auth via `get_current_user` dependency (no `require_role` — S07.09 doesn't specify a role)
- Direct SQL for email verification (`UPDATE client.users SET email_verified = TRUE`)
- No `respx` needed (no AI Gateway calls in this story)
- No `asyncio.gather` needed (no race condition tests — that's AC3/AC4 in Story 7.4)

### Red Phase Design Rationale

Tests fail because none of the following exist yet:
1. `src/client_api/schemas/content_block.py` — `ContentBlockCreate`, `ContentBlockPatch`, `ContentBlockResponse` not defined
2. `src/client_api/services/content_block_service.py` — all 8 service functions not implemented
3. `src/client_api/api/v1/content_blocks.py` — FastAPI router with 8 endpoints not created
4. Router not registered in `main.py` — all `POST/GET/PATCH/DELETE /api/v1/content-blocks` return 404/405

---

## Step 4: Generated Tests (TDD RED PHASE)

### Test File

**Location:** `services/client-api/tests/api/test_content_blocks.py`

**Red Phase Strategy:** `@pytest.mark.skip(reason="🔴 TDD RED PHASE — Story 7.9 not yet implemented")`
applied at **class level** — each method inherits the skip without per-method decoration.

### Fixture Architecture

| Name | Type | Purpose |
|------|------|---------|
| `cb_user` | `pytest_asyncio.fixture` (per-test) | Register + verify + login a user; yield `(client, session, token)`; rollback in finally |
| `_register_and_login_second_company(client, session)` | async helper function | Creates Company B user; returns Company B JWT; used in all RLS + cross-company search tests |
| `_DEFAULT_BLOCK` | module-level dict | Standard block payload for create operations in CRUD and RLS tests |

**Note:** No `aigw_url_env` fixture needed (no AI Gateway calls in this story).

### Helper: `_register_and_login_second_company`

Follows the exact pattern from `test_proposal_pricing_win_themes.py`:
- `POST /api/v1/auth/register` with unique email/company
- Direct SQL `UPDATE client.users SET email_verified = TRUE WHERE id = :id`
- `POST /api/v1/auth/login` → return `access_token`

### Test Classes and Methods

#### TestContentBlockCRUD (11 tests) — P1 — E07-P1-017

```
test_create_content_block                    [P1, AC1, E07-P1-017]
test_create_sets_company_from_jwt_not_body   [P1, AC1, E07-R-003]
test_create_rejects_missing_title            [P1, AC1]
test_create_rejects_missing_body             [P1, AC1]
test_get_content_block_by_id                 [P1, AC4, E07-P1-017]
test_get_nonexistent_returns_404             [P1, AC4]
test_update_increments_version               [P1, AC5, E07-P1-017]
test_update_partial_patch_preserves_other_fields [P1, AC5]
test_approve_sets_approved_fields            [P1, AC6, E07-P1-017]
test_unapprove_clears_approved_fields        [P1, AC6, E07-P1-017]
test_delete_removes_block                    [P1, AC7, E07-P1-017]
```

#### TestContentBlockSearch (4 tests) — P0/P1/P3 — E07-P1-018, E07-P0-006, E07-P3-003

```
test_search_returns_matching_blocks          [P1, AC3, E07-P1-018]
test_search_returns_empty_for_no_match       [P1, AC3]
test_search_relevance_ordering               [P3, AC3, E07-P3-003]
test_search_cross_company_returns_zero_results [P0, AC3, E07-P0-006, E07-R-003]
```

#### TestContentBlockTsvectorFreshness (2 tests) — P2 — E07-P2-006, E07-R-009

```
test_search_updated_after_patch_body         [P2, AC5, E07-P2-006, E07-R-009]
test_search_updated_after_patch_title        [P2, AC5, E07-P2-006, E07-R-009]
```

#### TestContentBlockListFilters (5 tests) — P1/P2 — E07-P2-007

```
test_list_pagination                         [P2, AC2, E07-P2-007]
test_list_filter_by_category                 [P2, AC2, E07-P2-007]
test_list_filter_by_tags_all_match           [P2, AC2, E07-P2-007]
test_list_returns_only_company_blocks        [P0, AC2, E07-P0-006]
test_list_default_sort_newest_first          [P1, AC2]
```

#### TestContentBlockRLS (5 tests) — P0 — E07-P0-006, E07-R-003

```
test_get_cross_company_returns_404           [P0, AC4, E07-P0-006, E07-R-003]
test_patch_cross_company_returns_404         [P0, AC5, E07-P0-006, E07-R-003]
test_approve_cross_company_returns_404       [P0, AC6, E07-P0-006, E07-R-003]
test_unapprove_cross_company_returns_404     [P0, AC6, E07-P0-006, E07-R-003]
test_delete_cross_company_returns_404        [P0, AC7, E07-P0-006, E07-R-003]
```

**Note:** `test_delete_cross_company_returns_404` additionally verifies the block still
exists via Company A's JWT — direct DB state verification following the Story 7.7 review
lesson on DB NULL state verification.

---

## Step 4C: Aggregation

### TDD Red Phase Validation

- ✅ All 27 tests use `@pytest.mark.skip` (class-level) — TDD red phase compliant
- ✅ All tests assert **expected behavior** (real HTTP status codes and response shapes)
- ✅ No placeholder assertions (`assert True`, `pass`, empty bodies)
- ✅ Tests fail because endpoints/schemas/service functions don't exist → guaranteed RED
- ✅ No `asyncio.gather` needed (no race condition tests in this story)
- ✅ `CAST(... AS uuid)` pattern not needed for this story (no raw SQL for content blocks in tests)
- ✅ `_register_and_login_second_company` helper defined (E07-P0-006 / E07-R-003)
- ✅ No `aigw_url_env` fixture (no AI Gateway calls — pure CRUD + FTS story)

### Epic Test Coverage Matrix

| Epic Test ID | Priority | This Story | Test Method |
|-------------|----------|------------|-------------|
| **E07-P0-006** | P0 | ✅ (×6) | `TestContentBlockSearch.test_search_cross_company_returns_zero_results` + all 5 `TestContentBlockRLS.*` |
| **E07-P1-017** | P1 | ✅ | `TestContentBlockCRUD` (create → GET → PATCH → approve → unapprove → DELETE) |
| **E07-P1-018** | P1 | ✅ | `TestContentBlockSearch.test_search_returns_matching_blocks` |
| **E07-P2-006** | P2 | ✅ (×2) | `TestContentBlockTsvectorFreshness.test_search_updated_after_patch_{body,title}` |
| **E07-P2-007** | P2 | ✅ | `TestContentBlockListFilters` (pagination, category, tags, company isolation, sort) |
| **E07-P3-003** | P3 | ✅ | `TestContentBlockSearch.test_search_relevance_ordering` |
| **E07-R-003** | Risk | ✅ | All 5 `TestContentBlockRLS.test_*_cross_company_returns_404` + company_id JWT test |
| **E07-R-009** | Risk | ✅ | `test_search_updated_after_patch_body` + `test_search_updated_after_patch_title` |

### Acceptance Criteria Coverage

| AC | Covered | Tests |
|----|---------|-------|
| AC1 — POST creates block; company_id from JWT; version=1; 201; 422 on missing title/body | ✅ | 4 tests |
| AC2 — GET list; limit/offset pagination; category filter; tags @> containment; created_at DESC | ✅ | 5 tests |
| AC3 — GET /search; plainto_tsquery FTS; ts_rank ordering; empty list for no match; cross-company zero | ✅ | 4 tests |
| AC4 — GET by ID; 404 for not-found or cross-company; never 403 | ✅ | 2 tests |
| AC5 — PATCH partial update; version++; tsvector refresh on title/body change (E07-R-009) | ✅ | 4 tests |
| AC6 — /approve sets approved_at + approved_by; /unapprove clears both; 404 cross-company | ✅ | 3 tests |
| AC7 — DELETE hard delete; 204; 404 cross-company; block still exists via Company A | ✅ | 2 tests |
| AC8 — Integration tests at test_content_blocks.py; 184 existing tests unchanged | ✅ | 27 total |

### Design Decisions

**`_DEFAULT_BLOCK` module constant:**
Provides a shared valid block payload for CRUD and RLS tests. Tests that need specific
keywords for FTS use inline payloads to ensure keyword uniqueness and test isolation.

**`cb_user` fixture (per-test, not session):**
Each test gets a fresh user + company with no pre-existing content blocks.
Rollback in `finally` block ensures test isolation — no cross-test block pollution.
No shared state across tests. Pattern matches Story 7.8 `pricing_win_themes_proposal` fixture.

**No `pytest_asyncio.fixture` for `another_company_token`:**
Following Story 7.8 pattern: cross-company tests use `_register_and_login_second_company()`
as an async helper called within the test body. This keeps the dependency chain simple
and avoids fixture parameter explosion.

**Globally-unique keywords for FTS tests:**
FTS tests use keywords like `xyzprocurement2026`, `tenderfts2026`, `secretproposal2026`,
`procurement2026abc`, `sourcing2026abc`, `eligibility2026xyz`, `compliance2026xyz`.
These are unlikely to appear in any other test data — ensures search results are deterministic
even when run alongside other test modules.

**`test_search_cross_company_returns_zero_results` additionally checks response text:**
Per Story 7.7 review lesson: cross-company search must not leak even partial block titles
or IDs. Test explicitly asserts `secret_keyword not in search_resp.text` — defense-in-depth.

**`test_delete_cross_company_returns_404` verifies block still exists:**
Per Story 7.7 review lesson on DB NULL state verification. After Company B's failed DELETE
attempt returns 404, the test immediately GETs the block via Company A's token to confirm
the physical row was not deleted.

**`test_list_returns_only_company_blocks` placed in `TestContentBlockListFilters`:**
Although this is a P0 RLS test, it specifically tests the LIST endpoint behavior.
The 5 `TestContentBlockRLS` tests cover the direct-access endpoints (GET by ID, PATCH,
approve, unapprove, DELETE). Keeping list isolation here makes the class more complete.

**Why 27 tests (not 20 from story estimate):**
The story AC8 says "~20 tests" but the detailed task breakdown in AC8 tasks 6.2–6.6
specifies 27 individual test methods when counted. The additional tests cover:
- `test_create_sets_company_from_jwt_not_body` (security-critical AC1 edge case)
- `test_create_rejects_missing_title` and `test_create_rejects_missing_body` (422 validation)
- `test_get_nonexistent_returns_404` (AC4 edge case)
- `test_list_returns_only_company_blocks` (list-level RLS, separate from direct-access RLS)

---

## Step 5: Validation & Completion

### Validation Checklist

- [x] All 27 tests marked `@pytest.mark.skip` (TDD red phase — class level)
- [x] Test file written to `services/client-api/tests/api/test_content_blocks.py`
- [x] Checklist saved to `eusolicit-docs/test-artifacts/atdd-checklist-7-9-content-blocks-crud-search-api.md`
- [x] All 8 ACs covered (AC1 → 4 tests, AC2 → 5, AC3 → 4, AC4 → 2, AC5 → 4, AC6 → 3, AC7 → 2, AC8 → composite)
- [x] All epic test IDs from story coverage map covered (E07-P0-006, P1-017/018, P2-006/007, P3-003, R-003, R-009)
- [x] `_register_and_login_second_company` helper defined
- [x] `_DEFAULT_BLOCK` module constant defined for test payload reuse
- [x] No `aigw_url_env` or `respx` (not needed — no AI Gateway calls)
- [x] No `asyncio.gather` (no race condition tests — content blocks have no optimistic locking)
- [x] Cross-company search verifies both empty list AND keyword not in response text
- [x] Delete cross-company test verifies block still exists via Company A's JWT
- [x] No orphaned browser sessions (backend tests only — no browser)
- [x] No temp artifacts outside `test_artifacts/`
- [x] Existing 184 proposal tests not modified

### Run Commands

```bash
cd eusolicit-app/services/client-api

# New story tests only (27 tests — all skipped in RED phase):
pytest tests/api/test_content_blocks.py -v

# Specific test classes:
pytest tests/api/test_content_blocks.py::TestContentBlockCRUD -v
pytest tests/api/test_content_blocks.py::TestContentBlockSearch -v
pytest tests/api/test_content_blocks.py::TestContentBlockTsvectorFreshness -v
pytest tests/api/test_content_blocks.py::TestContentBlockListFilters -v
pytest tests/api/test_content_blocks.py::TestContentBlockRLS -v

# Full regression suite (all 184 existing + 27 new = 211 tests):
pytest tests/api/test_proposals.py \
       tests/api/test_proposal_versions.py \
       tests/api/test_proposal_content_save.py \
       tests/api/test_proposal_generate.py \
       tests/api/test_proposal_checklist.py \
       tests/api/test_proposal_compliance_risk_scoring.py \
       tests/api/test_proposal_pricing_win_themes.py \
       tests/api/test_content_blocks.py -v
```

**Expected output (RED phase):** All 27 tests skipped with reason
`"🔴 TDD RED PHASE — Story 7.9 not yet implemented"`.

---

## Implementation Checklist (TDD Green Phase)

### To make `TestContentBlockCRUD` pass (Tasks 1–5)

- [ ] Verify `ContentBlock` ORM model in `services/client-api/src/client_api/models/proposal.py`
  - Must have: `search_vector: Mapped[str | None] = mapped_column(TSVECTOR, nullable=True)`
  - Must have: `tags: Mapped[list[str]] = mapped_column(ARRAY(sa.String), nullable=False, server_default="{}")`
  - Must have: `version`, `approved_at`, `approved_by` columns
- [ ] Create `services/client-api/src/client_api/schemas/content_block.py`
  - `ContentBlockCreate(BaseModel)`: `title`, `category`, `body`, `tags`
  - `ContentBlockPatch(BaseModel)`: all fields optional
  - `ContentBlockResponse(BaseModel)`: all 11 fields including `version`, `approved_at`, `approved_by`
- [ ] Create `services/client-api/src/client_api/services/content_block_service.py`
  - `create_content_block`: `company_id` from JWT only; `version=1`; `search_vector` from `to_tsvector`
  - `_get_block_or_404`: returns 404 for both non-existent AND cross-company (never 403)
  - `get_content_block`, `update_content_block`, `approve_content_block`, `unapprove_content_block`, `delete_content_block`
  - **E07-R-009:** `update_content_block` MUST explicitly recalculate `search_vector` when `title` or `body` changes
- [ ] Create `services/client-api/src/client_api/api/v1/content_blocks.py`
  - **CRITICAL:** `/search` MUST be registered BEFORE `/{block_id}` (FastAPI path collision prevention)
  - Endpoint order: `POST /`, `GET /search`, `GET /`, `GET /{block_id}`, `PATCH /{block_id}`, `POST /{block_id}/approve`, `POST /{block_id}/unapprove`, `DELETE /{block_id}`
- [ ] Register router in `services/client-api/src/client_api/main.py`
  - `app.include_router(content_blocks_router, prefix="/api/v1/content-blocks", tags=["content-blocks"])`

### Optionally: Migration 025 if tsvector trigger absent from 020

- [ ] Check `services/client-api/alembic/versions/020_proposal_workspace_schema.py` for `search_vector` trigger
- [ ] If trigger is MISSING: create `025_cb_search_trigger.py`
  - Use revision `"025_cb_search_trigger"` (21 chars — fits VARCHAR(32); **NOT** `"025_content_blocks_search_trigger"` which is 34 chars — EXCEEDS VARCHAR(32))

### Verification Order

1. `pytest tests/api/test_content_blocks.py::TestContentBlockCRUD -v` → make green first
2. `pytest tests/api/test_content_blocks.py::TestContentBlockSearch -v` → needs FTS working
3. `pytest tests/api/test_content_blocks.py::TestContentBlockTsvectorFreshness -v` → needs E07-R-009 mitigation
4. `pytest tests/api/test_content_blocks.py::TestContentBlockListFilters -v` → needs pagination + filters
5. `pytest tests/api/test_content_blocks.py::TestContentBlockRLS -v` → needs RLS enforcement (404 not 403)
6. Full regression: all 211 tests pass

---

## Next Steps (TDD Green Phase)

After Story 7.9 implementation is complete:

1. **Remove** `@pytest.mark.skip` from all 5 test classes in `test_content_blocks.py`
2. **Run tests:** `pytest tests/api/test_content_blocks.py -v`
3. **Verify all 27 tests PASS** (green phase)
4. If any tests fail:
   - Fix implementation (feature bug) OR fix test assertion (misalignment with spec)
5. **Run full regression:** confirm all 184 existing tests remain green
6. Commit passing test file

### Key Architecture Constraints to Verify

| Constraint | Verification |
|-----------|--------------|
| `company_id` from JWT only | `test_create_sets_company_from_jwt_not_body` |
| 404 not 403 for cross-company | All 5 `TestContentBlockRLS` tests |
| `/search` before `/{block_id}` in router | `test_search_returns_matching_blocks` (would get 422 if wrong order) |
| Hard delete (not soft-archive) | `test_delete_removes_block` — GET returns 404 after DELETE |
| Version++ on every PATCH | `test_update_increments_version` + `test_update_partial_patch_preserves_other_fields` |
| E07-R-009: tsvector refresh on PATCH | Both `TestContentBlockTsvectorFreshness` tests |
| `session.flush()` not `session.commit()` | Architecture — tested indirectly by rollback isolation |
| `approved_by` stores UUID (not boolean) | `test_approve_sets_approved_fields` — `uuid.UUID(data["approved_by"])` |
| Cross-company search returns empty (not 404) | `test_search_cross_company_returns_zero_results` |

### Follow-on Workflows

- Run `/bmad-agent-dev` with Story 7.9 file to execute the implementation
- Run `/bmad-testarch-automate` to expand coverage beyond the 27 AC tests
- Run `/bmad-testarch-trace` to update the traceability matrix with E07-P0-006/P1-017/018

---

## Knowledge Base References Applied

- **`test-levels-framework.md`** — Integration level selection (API+DB, no E2E for backend story)
- **`test-priorities-matrix.md`** — P0 for RLS/security; P1 for nominal paths; P2 for edge cases; P3 for nice-to-have
- **`data-factories.md`** — Fixture patterns for test isolation (`cb_user` fixture design)
- **`test-quality.md`** — Given-When-Then, single assertion focus, determinism (unique FTS keywords)
- **`test-healing-patterns.md`** — Resilient assertions; not brittle on exact timestamps or ordering IDs
- **`api-testing-patterns.md`** — HTTP status code contract testing; cross-company 404 not 403

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Story:** `7-9-content-blocks-crud-search-api`
**Total Tests:** 27 (all 🔴 RED — skipped until implementation)
