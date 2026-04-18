---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-17'
workflowType: bmad-testarch-atdd
storyId: 7-4-proposal-content-save-api-auto-save-full-save
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-4-proposal-content-save-api-auto-save-full-save.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/services/client-api/tests/api/test_proposal_versions.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 7.4 — Proposal Content Save API (Auto-save + Full Save)

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED (failing tests generated — implementation pending)
**Test File:** `services/client-api/tests/api/test_proposal_content_save.py`

---

## Step 1: Preflight & Context

### Stack Detection

| Signal | Found | Result |
|--------|-------|--------|
| `pyproject.toml` (multiple services) | ✅ | Backend indicators |
| `eusolicit-app/frontend/` with React | ✅ | Frontend indicators |
| **Detected stack** | | **`fullstack`** |
| **This story's test level** | | **Backend integration tests (Python pytest)** |

Story 7.4 is a pure backend API story. All tests are Python `pytest-asyncio` integration tests
against the FastAPI service. No E2E or browser tests generated.

### Prerequisites Verified

| Prerequisite | Status | Notes |
|--------------|--------|-------|
| Story approved with clear AC | ✅ | 6 ACs, all acceptance-testable |
| Test framework configured | ✅ | `pyproject.toml` with pytest-asyncio, httpx |
| `conftest.py` fixtures available | ✅ | `client_api_session_factory`, `test_redis_client` |
| Story 7.3 patterns available | ✅ | `test_proposal_versions.py` — fixture and helper patterns |
| Epic test design loaded | ✅ | `test-design-epic-07.md` — E07-P0-003/004/005, P1-007/008 |
| Stories 7.1–7.3 complete | ✅ | `proposals`, `proposal_versions` tables + CRUD endpoints exist |

### Files Loaded

- Story: `7-4-proposal-content-save-api-auto-save-full-save.md`
- Epic test design: `test-design-epic-07.md`
- Pattern reference: `test_proposal_versions.py`, `conftest.py`

---

## Step 2: Generation Mode

**Selected mode:** AI Generation (backend stack, no browser recording)

**Rationale:** Both new endpoints are REST API endpoints testable via `httpx.AsyncClient` with
`ASGITransport`. Optimistic locking semantics are tested via `asyncio.gather` concurrency. No UI
interaction required for this story.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Mapping

| AC | Endpoint | Test Class | Test Count | Priority |
|----|----------|------------|------------|----------|
| AC1 | `PATCH /proposals/{id}/content/sections/{key}` | `TestAC1SectionAutoSave` | 6 | P0/P1 |
| AC2 | `PUT /proposals/{id}/content` | `TestAC2FullSave` | 4 | P0/P1 |
| AC3 | Atomic hash check (SELECT FOR UPDATE) | `TestConcurrentWrites` | 2 | P0 |
| AC4 | Concurrent race: 1×200, 1×409 | `TestConcurrentWrites` | (within AC3 tests) | P0 |
| AC5 | Cross-company 404 for both endpoints | Distributed across AC1/AC2 | 2 | P0 |
| AC6 | Integration test completeness | All classes | — | Composite |

**Total: 12 failing tests**

### Test Levels

All tests are **integration** level (`@pytest.mark.integration`):
- Real PostgreSQL via `client_api_session_factory` (testcontainers-backed)
- Real FastAPI ASGI via `httpx.AsyncClient` + `ASGITransport`
- Real JWT auth via `require_role("bid_manager")` and `get_current_user` dependencies
- Direct SQL for test data seeding and DB state verification
- `asyncio.gather` for concurrent write tests (E07-P0-003, E07-P0-004)

### Red Phase Design Rationale

Tests fail because none of the following exist yet:
1. `schemas/proposal_content.py` — `SectionAutoSaveRequest`, `FullSaveRequest`, `ContentSaveResponse`, `ContentConflictDetail`
2. `services/proposal_service.py` — `_get_current_version_for_update`, `_compute_content_hash`, `auto_save_section`, `full_save_content` not added
3. `api/v1/proposals.py` — 2 new route handlers not registered

Both new endpoints return 404 until registered. All AC1/AC2 tests assert 200 → guaranteed RED.
Concurrent tests (AC3/AC4) additionally require the SELECT FOR UPDATE implementation.

---

## Step 4: Generated Tests (TDD RED PHASE)

### Test File

**Location:** `services/client-api/tests/api/test_proposal_content_save.py`

**All 12 tests are marked:**
```python
@pytest.mark.skip(reason="🔴 TDD RED PHASE — Story 7.4 not yet implemented")
```
Applied at **class level** so each method inherits the skip without requiring
per-method decoration.

### Helpers and Fixtures

| Name | Type | Purpose |
|------|------|---------|
| `_hash(content)` | module function | Deterministic MD5 of content dict — matches `_compute_content_hash()` in service |
| `_seed_proposal_with_sections(session, proposal_id, sections)` | async function | Direct SQL UPDATE of `proposal_versions.content` for test setup — bypasses Story 7.4 API |
| `proposal_and_token` | `pytest_asyncio.fixture` | Register + verify + login + create proposal + seed 2 sections → yields `(client, session, token, proposal_id, content, content_hash)` |
| `_register_and_login_second_company()` | async helper | Creates Company B user and returns its JWT — used for all E07-P0-005 RLS tests |

### Initial Proposal State (from fixture)

All tests start with:
```python
content = {
    "sections": [
        {"key": "intro", "title": "Introduction", "body": "Hello World"},
        {"key": "body",  "title": "Main Body",    "body": "Original body text"},
    ]
}
content_hash = _hash(content)  # deterministic MD5
```

### Test Classes and Methods

#### TestAC1SectionAutoSave (6 tests) — P1/P0
```
test_patch_updates_only_target_section            [P1, E07-P1-007]
test_patch_advances_updated_at                    [P1, E07-P1-008]
test_patch_returns_full_content_with_hash         [P1, AC1]
test_patch_missing_section_key_returns_404        [P2, AC1, E07-P2-001]
test_patch_stale_hash_returns_409_with_latest_content  [P0, AC1, AC3]
test_patch_cross_company_returns_404              [P0, E07-P0-005]
```

#### TestAC2FullSave (4 tests) — P1/P0
```
test_put_replaces_all_sections                    [P1, AC2]
test_put_advances_updated_at                      [P1, AC2]
test_put_stale_hash_returns_409_with_latest_content    [P0, AC2, AC3]
test_put_cross_company_returns_404                [P0, E07-P0-005]
```

#### TestConcurrentWrites (2 tests) — P0 (Critical)
```
test_concurrent_patch_race_exactly_one_200_one_409     [P0, E07-P0-003, AC3, AC4]
test_stale_hash_full_save_after_concurrent_patch       [P0, E07-P0-004, AC3, AC4]
```

---

## Step 4C: Aggregation

### TDD Red Phase Validation

- ✅ All 12 tests use `@pytest.mark.skip` (class-level) — TDD red phase compliant
- ✅ All tests assert **expected behavior** (real HTTP status codes and response shapes)
- ✅ No placeholder assertions (`assert True`, `pass`, empty bodies)
- ✅ Tests fail because endpoints/schemas/service functions don't exist — guaranteed RED
- ✅ `asyncio.gather` used for concurrency tests (E07-P0-003, E07-P0-004) — NOT `asyncio.sleep`
- ✅ `CAST(:param AS uuid)` used in all raw SQL — asyncpg compatibility pattern from S07.03

### Epic Test Coverage Matrix

| Epic Test ID | Priority | This Story | Test Method |
|-------------|----------|------------|-------------|
| **E07-P0-003** | P0 | ✅ | `TestConcurrentWrites.test_concurrent_patch_race_exactly_one_200_one_409` |
| **E07-P0-004** | P0 | ✅ | `TestConcurrentWrites.test_stale_hash_full_save_after_concurrent_patch` |
| **E07-P0-005** | P0 | ✅ (×2) | `TestAC1SectionAutoSave.test_patch_cross_company_returns_404`, `TestAC2FullSave.test_put_cross_company_returns_404` |
| **E07-P1-007** | P1 | ✅ | `TestAC1SectionAutoSave.test_patch_updates_only_target_section` |
| **E07-P1-008** | P1 | ✅ | `TestAC1SectionAutoSave.test_patch_advances_updated_at` |
| **E07-P2-001** | P2 | ✅ | `TestAC1SectionAutoSave.test_patch_missing_section_key_returns_404` |

### Acceptance Criteria Coverage

| AC | Covered | Tests |
|----|---------|-------|
| AC1 — PATCH updates section, returns 200 + content + hash; 404 for missing key; 409 on stale hash; 404 cross-company | ✅ | 6 tests |
| AC2 — PUT replaces sections, returns 200 + content + hash; 404 cross-company; 409 on stale hash | ✅ | 4 tests |
| AC3 — Atomic SELECT FOR UPDATE; hash from entire content; no split round-trips | ✅ | 2 tests (concurrent race conditions) |
| AC4 — Concurrent PATCH: exactly one 200, one 409; no silent data loss | ✅ | `test_concurrent_patch_race_exactly_one_200_one_409` |
| AC5 — Cross-company 404 (not 403) for both endpoints | ✅ | 2 tests (one per endpoint) |
| AC6 — Integration tests at `tests/api/test_proposal_content_save.py` | ✅ | 12 total tests |

### Design Decisions

**`_hash()` helper matches server-side `_compute_content_hash()`:**
Both use `json.dumps(content, sort_keys=True, separators=(',', ':'))`.
This is the frontend contract — documented in the story so the client can reproduce
the hash without an extra API round-trip. Tests verify the round-trip by asserting
`data["content_hash"] == _hash(data["content"])`.

**`_seed_proposal_with_sections` direct SQL helper:**
Tests need a known two-section state before exercising the PATCH/PUT endpoints.
Using the PATCH endpoint itself for seeding would create a circular dependency
(the endpoint under test). Direct SQL UPDATE on `proposal_versions.content` isolates
the test from the implementation — consistent with the `_seed_version_with_content`
pattern from Story 7.3.

**Why 12 tests only (not more):**
Story 7.4 has a narrow surface area: 2 endpoints, 1 shared optimistic locking mechanism.
The 12 tests cover all 6 ACs, both P0 concurrency scenarios (E07-P0-003/004), and all
cross-company 404 paths (E07-P0-005). Additional edge cases (401 unauthenticated,
non-existent proposal UUID) are covered by the existing 107 proposal tests
(`test_proposals.py` + `test_proposal_versions.py`) via the same auth middleware.

**`asyncio.gather` for concurrent writes (not `asyncio.sleep`):**
`asyncio.gather` is the only way to fire two requests truly concurrently within a
single async test. `asyncio.sleep` would serialize the requests and defeat the purpose.
The story spec and epic test design both mandate `asyncio.gather`.

**`CAST(:param AS uuid)` in all raw SQL:**
asyncpg does not support PostgreSQL `::` cast syntax in parameterized queries.
Uses `CAST(... AS uuid)` consistently — pattern established in Story 7.3.

---

## Step 5: Validation & Completion

### Validation Checklist

- [x] All 12 tests marked `@pytest.mark.skip` (TDD red phase)
- [x] Test file written to `services/client-api/tests/api/test_proposal_content_save.py`
- [x] Checklist saved to `eusolicit-docs/test-artifacts/atdd-checklist-7-4-proposal-content-save-api-auto-save-full-save.md`
- [x] All 6 ACs covered
- [x] All 6 epic test IDs from story's test coverage alignment table covered
- [x] `_register_and_login_second_company` helper defined (E07-P0-005 RLS tests)
- [x] `_seed_proposal_with_sections` helper defined (bypasses Story 7.4 API for fixture setup)
- [x] `_hash()` helper defined and documented as frontend contract
- [x] `asyncio.gather` used for concurrent write tests (E07-P0-003, E07-P0-004)
- [x] `CAST(... AS uuid)` pattern used throughout (asyncpg compatibility)
- [x] No orphaned browser sessions (backend tests only — no browser)
- [x] No temp artifacts outside `test_artifacts/`
- [x] Existing 107 proposal tests not modified (`test_proposals.py`, `test_proposal_versions.py`)

### Run Command

```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_proposal_content_save.py -v
```

**Expected output (RED phase):** All 12 tests skipped with reason
`"🔴 TDD RED PHASE — Story 7.4 not yet implemented"`.

**Full regression check (must stay green — 107 existing tests):**
```bash
pytest tests/api/test_proposals.py tests/api/test_proposal_versions.py tests/api/test_proposal_content_save.py -v
```

---

## Next Steps (TDD Green Phase)

After Story 7.4 implementation is complete:

1. **Remove** `@pytest.mark.skip` from all 3 test classes in `test_proposal_content_save.py`
2. **Run tests:** `pytest tests/api/test_proposal_content_save.py -v`
3. **Verify all 12 tests PASS** (green phase)
4. If any tests fail:
   - Fix implementation (feature bug) OR fix test assertion (misalignment with spec)
5. **Run full regression:** `pytest tests/api/test_proposals.py tests/api/test_proposal_versions.py tests/api/test_proposal_content_save.py -v` (must stay at 119 total)
6. Commit passing test file

### Implementation Files Required

| File | Action | Key Items |
|------|--------|-----------|
| `src/client_api/schemas/proposal_content.py` | **CREATE** | `SectionItem`, `SectionAutoSaveRequest`, `FullSaveRequest`, `ContentSaveResponse` (with `content_hash`), `ContentConflictDetail` |
| `src/client_api/services/proposal_service.py` | **MODIFY** (add only) | `_get_current_version_for_update` (SELECT FOR UPDATE), `_compute_content_hash`, `auto_save_section`, `full_save_content` |
| `src/client_api/api/v1/proposals.py` | **MODIFY** (add only) | `PATCH /{id}/content/sections/{key}` + `PUT /{id}/content`; import from `proposal_content` schemas; expose `_compute_content_hash` for response hash |

### Critical Implementation Notes

1. **SELECT FOR UPDATE is mandatory** (E07-R-002): lock `proposal_versions` row via `with_for_update()` before computing hash — never split SELECT + UPDATE
2. **Hash entire JSONB content** (not just target section): prevents hash collisions between different section body combinations
3. **JSONB mutation requires full column reassignment**: `version.content = new_dict` — SQLAlchemy dirty tracking does not detect in-place mutations
4. **`session.flush()` + `session.refresh()` — never `session.commit()`**: session lifecycle owned by `get_db_session` dependency
5. **`_compute_content_hash` must be importable from router**: router needs to compute the response hash after the service call

### Follow-on Workflows

- Run `/bmad-agent-dev` with this story file to execute the implementation
- Run `/bmad-testarch-automate` to expand coverage beyond the 12 AC tests
- Run `/bmad-testarch-trace` to update the traceability matrix with E07-P0-003/004/005

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Story:** `7-4-proposal-content-save-api-auto-save-full-save`
**Total Tests:** 12 (all 🔴 RED — skipped until implementation)
