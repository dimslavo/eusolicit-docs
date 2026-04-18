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
storyId: 7-3-proposal-versioning-api
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-3-proposal-versioning-api.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/services/client-api/tests/api/test_proposals.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 7.3 — Proposal Versioning API

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED (failing tests generated — implementation pending)
**Test File:** `services/client-api/tests/api/test_proposal_versions.py`

---

## Step 1: Preflight & Context

### Stack Detection

| Signal | Found | Result |
|--------|-------|--------|
| `pyproject.toml` (multiple services) | ✅ | Backend indicators |
| `eusolicit-app/frontend/` with React | ✅ | Frontend indicators |
| **Detected stack** | | **`fullstack`** |
| **This story's test level** | | **Backend integration tests (Python pytest)** |

Story 7.3 is a pure backend API story. All tests are Python `pytest-asyncio` integration tests
against the FastAPI service. No E2E or browser tests generated.

### Prerequisites Verified

| Prerequisite | Status | Notes |
|--------------|--------|-------|
| Story approved with clear AC | ✅ | 8 ACs, all acceptance-testable |
| Test framework configured | ✅ | `pyproject.toml` with pytest-asyncio, httpx |
| `conftest.py` fixtures available | ✅ | `client_api_session_factory`, `test_redis_client` |
| Story 7.2 patterns available | ✅ | `test_proposals.py` — fixture and helper patterns |
| Epic test design loaded | ✅ | `test-design-epic-07.md` — E07-P0-005/008/009 coverage |

### Files Loaded

- Story: `7-3-proposal-versioning-api.md`
- Epic test design: `test-design-epic-07.md`
- Pattern reference: `test_proposals.py`, `conftest.py`

---

## Step 2: Generation Mode

**Selected mode:** AI Generation (backend stack, no browser recording)

**Rationale:** All 5 new endpoints are REST API endpoints testable via `httpx`. Section-level
diff logic is pure Python. No UI interaction required for this story.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Mapping

| AC | Endpoint | Test Class | Test Count | Priority |
|----|----------|------------|------------|----------|
| AC1 | `POST /proposals/{id}/versions` | `TestAC1CreateVersion` | 7 | P0/P1 |
| AC2 | `GET /proposals/{id}/versions` | `TestAC2ListVersions` | 4 | P1 |
| AC3 | `GET /proposals/{id}/versions/{vid}` | `TestAC3GetVersion` | 5 | P1 |
| AC4 | `GET /proposals/{id}/versions/diff` | `TestAC4DiffVersions` | 7 | P0/P2 |
| AC5 | `POST /proposals/{id}/versions/{vid}/rollback` | `TestAC5Rollback` | 6 | P0 |
| AC6 | RLS cross-company (all endpoints) | Distributed across classes | 7 | P0 |
| AC7 | SELECT FOR UPDATE race guard | `TestVersionNumberIntegrity` | 3 | P2 |
| AC8 | Integration test completeness | All classes | — | Composite |

**Total: 32 failing tests**

### Test Levels

All tests are **integration** level (`@pytest.mark.integration`):
- Real PostgreSQL via `client_api_session_factory`
- Real FastAPI ASGI via `httpx.AsyncClient` + `ASGITransport`
- Real JWT auth via `require_role` and `get_current_user` dependencies
- Direct SQL for test data seeding (diff content, email verification)

### Red Phase Design Rationale

Tests fail because none of the following exist yet:
1. `schemas/proposal_versions.py` — `VersionResponse`, `VersionListResponse`, etc.
2. `services/proposal_service.py` — versioning functions not added
3. `api/v1/proposals.py` — 5 new route handlers not registered

All 5 endpoints return 404 until implemented. Tests assert 201/200 → guaranteed RED.

---

## Step 4: Generated Tests (TDD RED PHASE)

### Test File

**Location:** `services/client-api/tests/api/test_proposal_versions.py`

**All 32 tests are marked:**
```python
@pytest.mark.skip(reason="🔴 TDD RED PHASE — Story 7.3 not yet implemented")
```
Applied at **class level** so each method inherits the skip without requiring
per-method decoration.

### Fixtures and Helpers

| Name | Type | Purpose |
|------|------|---------|
| `proposal_and_token` | `pytest_asyncio.fixture` | Register + verify + login + create proposal → yields `(client, session, token, proposal_id)` |
| `_create_version()` | helper function | `POST /proposals/{id}/versions` → returns `httpx.Response` |
| `_register_and_login_second_company()` | helper function | Creates Company B and returns its JWT — used by all E07-P0-005 RLS tests |
| `_seed_version_with_content()` | helper function | Direct SQL INSERT of `proposal_versions` row — used by diff tests to avoid Story 7.4 (PATCH) dependency |

### Test Classes and Methods

#### TestAC1CreateVersion (7 tests) — P1/P0
```
test_post_creates_version_with_change_summary_returns_201    [P1, E07-P1-005]
test_post_creates_version_without_summary_returns_201        [P1]
test_post_increments_version_number                          [P1, E07-P1-005]
test_post_updates_current_version_id_on_proposal             [P1]
test_post_snapshots_current_content                          [P1, E07-P1-005]
test_post_unauthenticated_returns_401                        [P1]
test_post_cross_company_returns_404                          [P0, E07-P0-005]
```

#### TestAC2ListVersions (4 tests) — P1/P0
```
test_get_list_returns_all_versions_newest_first              [P1, E07-P1-006]
test_get_list_returns_initial_version                        [P1]
test_get_list_cross_company_returns_404                      [P0, E07-P0-005]
test_get_list_unauthenticated_returns_401                    [P1]
```

#### TestAC3GetVersion (5 tests) — P1/P0
```
test_get_version_returns_200_with_content                    [P1]
test_get_version_wrong_proposal_returns_404                  [P1, AC6]
test_get_version_nonexistent_returns_404                     [P2]
test_get_version_cross_company_returns_404                   [P0, E07-P0-005]
test_get_version_unauthenticated_returns_401                 [P1]
```

#### TestAC4DiffVersions (7 tests) — P0/P2
```
test_diff_section_added_changed_unchanged                    [P0, E07-P0-009]
test_diff_section_removed                                    [P1]
test_diff_identical_versions_all_unchanged                   [P2, E07-P2-003]
test_diff_missing_from_param_returns_422                     [P1]
test_diff_invalid_uuid_param_returns_422                     [P1]
test_diff_version_not_in_proposal_returns_404                [P1, AC6]
test_diff_cross_company_returns_404                          [P0, E07-P0-005]
```

#### TestAC5Rollback (6 tests) — P0
```
test_rollback_creates_new_version_from_target_content        [P0, E07-P0-008]
test_rollback_default_change_summary                         [P1]
test_rollback_custom_change_summary                          [P1]
test_rollback_nonexistent_version_returns_404                [P2]
test_rollback_cross_company_returns_404                      [P0, E07-P0-005]
test_rollback_unauthenticated_returns_401                    [P1]
```

#### TestVersionNumberIntegrity (3 tests) — P2
```
test_version_numbers_are_sequential_no_gaps                  [P2, E07-P2-002]
test_rollback_continues_sequence                             [P2, E07-P2-002]
test_concurrent_version_creation_no_duplicate_numbers        [P2, E07-P2-004, E07-R-006]
```

---

## Step 4C: Aggregation

### TDD Red Phase Validation

- ✅ All 32 tests use `@pytest.mark.skip` (class-level) — TDD red phase compliant
- ✅ All tests assert **expected behavior** (real HTTP status codes and response shapes)
- ✅ No placeholder assertions (`assert True`, `expect(true).toBe(true)`)
- ✅ Tests fail because endpoints don't exist — guaranteed RED until implementation

### Epic Test Coverage Matrix

| Epic Test ID | Priority | This Story | Test Method |
|-------------|----------|------------|-------------|
| E07-P0-005 | P0 | ✅ | 5× cross-company 404 (one per endpoint type) across AC1–AC5 |
| E07-P0-008 | P0 | ✅ | `TestAC5Rollback.test_rollback_creates_new_version_from_target_content` |
| E07-P0-009 | P0 | ✅ | `TestAC4DiffVersions.test_diff_section_added_changed_unchanged` |
| E07-P1-005 | P1 | ✅ | `TestAC1CreateVersion` — snapshot, increment, content copy |
| E07-P1-006 | P1 | ✅ | `TestAC2ListVersions.test_get_list_returns_all_versions_newest_first` |
| E07-P2-002 | P2 | ✅ | `TestVersionNumberIntegrity` — sequential + rollback continues |
| E07-P2-003 | P2 | ✅ | `TestAC4DiffVersions.test_diff_identical_versions_all_unchanged` |
| E07-P2-004 | P2 | ✅ | `TestVersionNumberIntegrity.test_concurrent_version_creation_no_duplicate_numbers` |

### Acceptance Criteria Coverage

| AC | Covered | Tests |
|----|---------|-------|
| AC1 — POST /versions → 201, version_number increments, content copied | ✅ | 7 tests |
| AC2 — GET /versions → 200, DESC order | ✅ | 4 tests |
| AC3 — GET /versions/{id} → 200, full content | ✅ | 5 tests |
| AC4 — GET /versions/diff → section diff accuracy, 422/404 errors | ✅ | 7 tests |
| AC5 — POST /versions/{id}/rollback → new version, history preserved | ✅ | 6 tests |
| AC6 — Company-scoped 404 for all 5 endpoint types | ✅ | 5 tests (one per class) |
| AC7 — SELECT FOR UPDATE race guard | ✅ | 1 concurrent test |
| AC8 — Integration test completeness | ✅ | 32 total tests |

### Design Decisions

**`_seed_version_with_content` helper:**
The diff tests require versions with specific section content (e.g., `intro='Hello'`, `body='World'`).
Story 7.4 (section PATCH) is not yet implemented. Direct SQL insertion isolates Story 7.3
acceptance criteria from Story 7.4 dependency — this is intentional and correct for ATDD.

**Route registration order (critical implementation note):**
`GET /versions/diff` **must** be registered BEFORE `GET /versions/{version_id}` in `proposals.py`.
FastAPI matches routes in registration order. If `/{version_id}` comes first, requests to
`/versions/diff` will attempt to cast the string `"diff"` as a UUID → 422 instead of routing to
the diff handler. The diff test `test_diff_missing_from_param_returns_422` validates the diff
route resolves correctly before checking the missing-param 422.

**`proposal_and_token` vs `proposal_client_and_session`:**
`proposal_and_token` yields `proposal_id` (needed for versioning tests) instead of `company_id_str`
(yielded by `proposal_client_and_session` in `test_proposals.py`). These are distinct fixtures
serving different test contexts.

---

## Step 5: Validation & Completion

### Validation Checklist

- [x] All 32 tests marked `@pytest.mark.skip` (TDD red phase)
- [x] Test file written to `services/client-api/tests/api/test_proposal_versions.py`
- [x] Checklist saved to `eusolicit-docs/test-artifacts/atdd-checklist-7-3-proposal-versioning-api.md`
- [x] All 8 ACs covered
- [x] All 8 epic test IDs from story's test coverage alignment table covered
- [x] `_register_and_login_second_company` helper defined (E07-P0-005 RLS tests)
- [x] `_seed_version_with_content` helper defined (AC4 diff tests isolated from Story 7.4)
- [x] Route order warning documented for implementer
- [x] No orphaned browser sessions (backend tests only — no browser)
- [x] No temp artifacts outside `test_artifacts/`

### Run Command

```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_proposal_versions.py -v
```

**Expected output (RED phase):** All 32 tests skipped with reason
`"🔴 TDD RED PHASE — Story 7.3 not yet implemented"`.

---

## Next Steps (TDD Green Phase)

After Story 7.3 implementation is complete:

1. **Remove** `@pytest.mark.skip` from all 6 test classes in `test_proposal_versions.py`
2. **Run tests:** `pytest tests/api/test_proposal_versions.py -v`
3. **Verify all 32 tests PASS** (green phase)
4. If any tests fail:
   - Fix implementation (feature bug) OR fix test assertion (misalignment with spec)
5. **Check route order** in `api/v1/proposals.py` — `/diff` must come before `/{version_id}`
6. Commit passing test file

### Implementation Files Required

| File | Action | Key Items |
|------|--------|-----------|
| `src/client_api/schemas/proposal_versions.py` | CREATE | `VersionCreateRequest`, `RollbackRequest`, `VersionResponse`, `VersionListResponse`, `SectionDiffEntry`, `VersionDiffResponse` |
| `src/client_api/services/proposal_service.py` | MODIFY (add only) | `_get_proposal_for_update`, `_next_version_number`, `create_version`, `list_versions`, `get_version`, `diff_versions`, `rollback_version` |
| `src/client_api/api/v1/proposals.py` | MODIFY (add only) | 5 route handlers; `/diff` route BEFORE `/{version_id}` |

### Follow-on Workflows

- Run `/bmad-testarch-automate` to expand coverage beyond the 32 AC tests
- Run `/bmad-agent-dev` with this story file to execute the implementation

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Story:** `7-3-proposal-versioning-api`
**Total Tests:** 32 (all 🔴 RED — failing until implementation)
