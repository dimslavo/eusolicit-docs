---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-07'
workflowType: bmad-testarch-atdd
storyId: 2-12-espd-profile-crud
storyFile: eusolicit-docs/implementation-artifacts/2-12-espd-profile-crud.md
outputFile: eusolicit-docs/test-artifacts/atdd-checklist-2-12-espd-profile-crud.md
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
inputDocuments:
  - eusolicit-docs/implementation-artifacts/2-12-espd-profile-crud.md
  - eusolicit-docs/test-artifacts/test-design-epic-02.md
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_company_profile.py
  - eusolicit-app/services/client-api/tests/api/test_audit_trail.py
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 2.12 — ESPD Profile CRUD

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**Status:** 🔴 RED PHASE — Failing tests generated, awaiting implementation

---

## Step 1 — Preflight & Context

### Stack Detection

- **Detected stack:** `backend`
- **Evidence:** `pyproject.toml` (FastAPI + pytest-asyncio), no `playwright.config.ts` or `cypress.config.ts`
- **Test framework:** `pytest` + `pytest-asyncio`
- **Test directory:** `eusolicit-app/services/client-api/tests/`

### Prerequisites Verified

- [x] Story 2-12-espd-profile-crud has clear acceptance criteria (8 ACs)
- [x] Test framework configured: `pyproject.toml` with pytest, pytest-asyncio, pytest-cov
- [x] Existing test patterns available: `test_company_profile.py`, `test_audit_trail.py`, `conftest.py`
- [x] `ESPDProfile` ORM model already exists (`src/client_api/models/espd_profile.py`)
- [x] `espd_profiles` table already exists (migration 002)

### TEA Config Flags (Resolved from config.yaml)

| Flag | Value | Effect |
|------|-------|--------|
| `tea_use_playwright_utils` | not set → disabled | No Playwright Utils fragments loaded |
| `tea_use_pactjs_utils` | not set → disabled | No contract testing fragments |
| `tea_pact_mcp` | not set → none | No SmartBear MCP |
| `tea_browser_automation` | not set → none | No browser automation (backend only) |
| `test_stack_type` | not set → auto → `backend` | Backend patterns only |

---

## Step 2 — Generation Mode

**Selected mode:** AI Generation (automatic for `backend` stack)

No browser recording needed. Acceptance criteria are clear API contracts (CRUD endpoints, HTTP status codes, auth/authz rules, audit logging, version semantics). All scenarios are standard REST API patterns suitable for direct AI generation from story spec.

---

## Step 3 — Test Strategy

### Acceptance Criteria → Test Scenario Mapping

| AC | Description | Test Scenarios | Priority |
|----|-------------|----------------|----------|
| **AC1** | GET latest profile → 200; 404 if none; 403 cross-tenant | 3 tests: 404 before PUT; 200 after PUT (version field, field_values); 401 unauth | P0 |
| **AC2** | PUT full profile → 200, version=1 first write; 422 if any section missing | 7 tests: version=1 first; E02-P2-007 second version; 422 ×4 (one per missing section); boundary 200; 401 unauth | P2 |
| **AC3** | PATCH partial merge → 200; 404 if no base profile | 3 tests: 404 before PUT; E02-P2-008 partial merge correctness; 401 unauth | P2 |
| **AC4** | Role enforcement: admin/bid_manager → 200 PUT/PATCH; lower roles → 403 | 6 tests: parametrized 403 contributor/reviewer/read_only on PUT; parametrized 403 on PATCH; admin 200; bid_manager 200 PUT+PATCH; all roles 200 GET | P1 |
| **AC5** | Monotonic version counter; GET returns highest | Covered by AC1/AC2 tests (version=1, version=2, highest version returned by GET) | P2 |
| **AC6** | GET /versions → 200 ascending; 404 if none | 3 tests: 404 before PUT; E02-P2-009 three PUTs→[1,2,3]; 401 unauth | P2 |
| **AC7** | Audit log for every PUT/PATCH | 3 tests in `test_audit_trail.py`: create audit (before=None); update audit (before/after differ); PATCH audit (patched key differs) | P1 |
| **AC8** | Unauthenticated → 401; cross-tenant → 403 all endpoints | 4+4 tests: 401 for GET/PUT/PATCH/versions; E02-P0-012: cross-tenant 403 for all 4 endpoints | P0 |

### Epic Test ID Mapping

| Epic Test ID | Priority | Covered By | Status |
|-------------|----------|------------|--------|
| **E02-P0-012** (ESPD portion) | P0 | `TestCrossTenantEspd` (4 tests) | 🔴 RED |
| **E02-P1-018** (ESPD extension) | P1 | `TestAuditTrailEspd` in `test_audit_trail.py` (3 tests) | 🔴 RED |
| **E02-P2-007** | P2 | `TestAC2PutEspdProfile.test_second_put_increments_version_to_2` | 🔴 RED |
| **E02-P2-008** | P2 | `TestAC3PatchEspdProfile.test_patch_exclusion_grounds_only_leaves_other_sections_unchanged` | 🔴 RED |
| **E02-P2-009** | P2 | `TestAC6VersionHistory.test_three_puts_produce_three_versions_ascending` | 🔴 RED |
| **E02-P2-010** | P2 | `TestAC2PutEspdProfile` (4 × 422 tests + boundary 200) | 🔴 RED |

### Risk Coverage

| Risk ID | Description | Tests Covering |
|---------|-------------|----------------|
| E02-R-002 (score 6) | Cross-tenant isolation | `TestCrossTenantEspd` (all 4 ESPD endpoints) |
| E02-R-007 (score 4) | Audit log completeness | `TestAuditTrailEspd` (create/update/patch audit rows) |
| E02-R-008 (score 2) | ESPD JSONB schema gaps | `TestAC2PutEspdProfile` (422 for each missing required section) |

---

## Step 4 — Generated Test Files (TDD RED PHASE)

### 🔴 TDD Red Phase: All tests decorated with `@pytest.mark.skip`

Tests assert **expected** behavior. They will **fail** until Story 2.12 is implemented.

---

### File 1: `tests/api/test_espd_profile.py` (NEW)

**Path:** `eusolicit-app/services/client-api/tests/api/test_espd_profile.py`
**Lines:** ~1,266
**Status:** 🔴 RED — all 28 tests skip-decorated

#### Test Classes

**`TestAC1GetEspdProfile`** (4 tests) — AC1, AC5, AC8
| Test Method | AC | Epic ID | Expected |
|-------------|----|---------|----|
| `test_get_before_any_put_returns_404` | AC1 | — | 404 before first PUT |
| `test_get_after_put_returns_200_with_full_profile` | AC1, AC5 | E02-P2-007 basis | 200, version=1, field_values match |
| `test_get_returns_highest_version_after_multiple_puts` | AC1, AC5 | — | GET returns latest version row |
| `test_unauthenticated_get_returns_401` | AC8 | — | 401 no auth header |

**`TestAC2PutEspdProfile`** (8 tests) — AC2, AC5, AC8, E02-P2-007, E02-P2-010
| Test Method | AC | Epic ID | Expected |
|-------------|----|---------|----|
| `test_first_put_returns_200_with_version_1` | AC2, AC5 | — | 200, version=1, all 4 sections stored |
| `test_second_put_increments_version_to_2` | AC2, AC5 | E02-P2-007 | 200, version=2 |
| `test_put_missing_exclusion_grounds_returns_422` | AC2 | E02-P2-010 | 422 |
| `test_put_missing_economic_standing_returns_422` | AC2 | E02-P2-010 | 422 |
| `test_put_missing_technical_ability_returns_422` | AC2 | E02-P2-010 | 422 |
| `test_put_missing_quality_assurance_returns_422` | AC2 | E02-P2-010 | 422 |
| `test_put_with_all_four_sections_returns_200` | AC2 | E02-P2-010 boundary | 200 |
| `test_unauthenticated_put_returns_401` | AC8 | — | 401 no auth header |

**`TestAC3PatchEspdProfile`** (3 tests) — AC3, AC8, E02-P2-008
| Test Method | AC | Epic ID | Expected |
|-------------|----|---------|----|
| `test_patch_before_any_put_returns_404` | AC3 | — | 404 no base profile |
| `test_patch_exclusion_grounds_only_leaves_other_sections_unchanged` | AC3 | E02-P2-008 | 200, exclusion_grounds updated, 3 others unchanged, version=2 |
| `test_unauthenticated_patch_returns_401` | AC8 | — | 401 no auth header |

**`TestAC4RoleEnforcement`** (6 tests) — AC4
| Test Method | AC | Epic ID | Expected |
|-------------|----|---------|----|
| `test_low_privilege_role_cannot_put_espd_profile` (parametrized ×3) | AC4 | E02-P1-014 pattern | 403 for contributor/reviewer/read_only |
| `test_low_privilege_role_cannot_patch_espd_profile` (parametrized ×3) | AC4 | E02-P1-014 pattern | 403 for contributor/reviewer/read_only |
| `test_admin_can_put_espd_profile` | AC4 | — | 200 for admin |
| `test_bid_manager_can_put_espd_profile` | AC4 | — | 200 for bid_manager |
| `test_bid_manager_can_patch_espd_profile` | AC4 | — | 200 for bid_manager |
| `test_all_authenticated_roles_can_get_espd_profile` (parametrized ×3) | AC4 | E02-P2-004 pattern | 200 for contributor/reviewer/read_only |

**`TestAC6VersionHistory`** (3 tests) — AC6, AC8, E02-P2-009
| Test Method | AC | Epic ID | Expected |
|-------------|----|---------|----|
| `test_get_versions_before_any_put_returns_404` | AC6 | — | 404 no versions |
| `test_three_puts_produce_three_versions_ascending` | AC6 | E02-P2-009 | 200, versions=[1,2,3], total=3 |
| `test_unauthenticated_get_versions_returns_401` | AC8 | — | 401 no auth header |

**`TestCrossTenantEspd`** (4 tests) — AC8, E02-P0-012
| Test Method | AC | Epic ID | Expected |
|-------------|----|---------|----|
| `test_company_a_token_returns_403_on_company_b_get` | AC8 | E02-P0-012 | 403 cross-tenant GET |
| `test_company_a_token_returns_403_on_company_b_put` | AC8 | E02-P0-012 | 403 cross-tenant PUT |
| `test_company_a_token_returns_403_on_company_b_patch` | AC8 | E02-P0-012 | 403 cross-tenant PATCH |
| `test_company_a_token_returns_403_on_company_b_get_versions` | AC8 | E02-P0-012 | 403 cross-tenant GET /versions |

---

### File 2: `tests/api/test_audit_trail.py` (EXTENDED)

**Path:** `eusolicit-app/services/client-api/tests/api/test_audit_trail.py`
**New lines added:** ~329 (1,239 → 1,568)
**Status:** 🔴 RED — 3 new tests in new `TestAuditTrailEspd` class, all skip-decorated

#### New Class: `TestAuditTrailEspd` (3 tests) — AC7, E02-P1-018

| Test Method | AC | Epic ID | Expected |
|-------------|----|---------|----|
| `test_first_put_espd_produces_create_audit_row` | AC7 | E02-P1-018 (Task 7.1) | audit_log: entity_type="espd_profile", action_type="create", before=None, after≠None, user_id set, ip_address≠None |
| `test_second_put_espd_produces_update_audit_row` | AC7 | E02-P1-018 (Task 7.2) | audit_log: action_type="update", before≠None, after≠None, before≠after |
| `test_patch_espd_produces_update_audit_row_with_differing_before_after` | AC7 | E02-P1-018 (Task 7.3) | audit_log: action_type="update", before[exclusion_grounds]≠after[exclusion_grounds], unpatched sections equal |

---

## Step 4C — Aggregation Summary

| Metric | Value |
|--------|-------|
| **TDD Phase** | 🔴 RED |
| **Total tests generated** | 31 (28 + 3) |
| **API tests (test_espd_profile.py)** | 28 |
| **Audit trail tests (test_audit_trail.py)** | 3 |
| **E2E tests** | 0 (backend stack, no browser tests needed) |
| **All tests skip-decorated** | ✅ Yes (`@pytest.mark.skip`) |
| **Tests assert expected behavior** | ✅ Yes (not placeholder assertions) |
| **Fixture pattern** | `espd_client_and_session` (mirrors `company_client_and_session`) |
| **Helper** | `_register_and_verify_with_role` (copied from `test_company_profile.py`) |
| **Execution mode** | Sequential (backend, no subagents needed) |

### Fixture Needs

| Fixture | Source | Status |
|---------|--------|--------|
| `espd_client_and_session` | Defined in `test_espd_profile.py` | ✅ Created |
| `audit_company_client_and_session` | Already exists in `test_audit_trail.py` | ✅ Reused |
| `client_api_session_factory` | `conftest.py` | ✅ Existing |
| `test_redis_client` | `conftest.py` | ✅ Existing |
| `_register_and_verify_with_role` | Defined in `test_espd_profile.py` | ✅ Created |

---

## Step 5 — Validation

### Prerequisites ✅

- [x] Story 2-12-espd-profile-crud has 8 acceptance criteria, all covered
- [x] `pyproject.toml` confirms pytest + pytest-asyncio available
- [x] No orphaned browser sessions (backend-only, no playwright)
- [x] All test artifacts stored in `test-artifacts/` (checklist) and `tests/api/` (test files)

### TDD Red Phase Compliance ✅

- [x] All 31 tests decorated with `@pytest.mark.skip`
- [x] `_SKIP_REASON` constants explain which story to implement to unlock tests
- [x] No placeholder assertions (`expect(true).toBe(true)` pattern absent)
- [x] Tests assert specific expected behavior (status codes, response body fields, version numbers, audit log fields)
- [x] Both test files pass Python AST parse (`python3 -c "import ast; ast.parse(...)"`)

### Acceptance Criteria Coverage ✅

| AC | Status | Test File | Tests |
|----|--------|-----------|-------|
| AC1 — GET latest, 404 if none, 403 cross-tenant | ✅ Covered | test_espd_profile.py | 4 |
| AC2 — PUT full profile, version=1 first, 422 missing sections | ✅ Covered | test_espd_profile.py | 8 |
| AC3 — PATCH partial merge, 404 if no base | ✅ Covered | test_espd_profile.py | 3 |
| AC4 — Role enforcement (5 roles × 3 endpoints) | ✅ Covered | test_espd_profile.py | 6 |
| AC5 — Monotonic version counter | ✅ Covered | test_espd_profile.py | 2 (via AC2 tests) |
| AC6 — GET /versions ascending, 404 if none | ✅ Covered | test_espd_profile.py | 3 |
| AC7 — Audit log for PUT/PATCH | ✅ Covered | test_audit_trail.py | 3 |
| AC8 — 401 unauth, 403 cross-tenant all 4 endpoints | ✅ Covered | test_espd_profile.py | 4+4 |

### Epic Test ID Coverage ✅

| Epic Test ID | Status |
|-------------|--------|
| E02-P0-012 (ESPD portion) | ✅ 4 tests in TestCrossTenantEspd |
| E02-P1-018 (ESPD extension) | ✅ 3 tests in TestAuditTrailEspd |
| E02-P2-007 | ✅ test_second_put_increments_version_to_2 |
| E02-P2-008 | ✅ test_patch_exclusion_grounds_only_leaves_other_sections_unchanged |
| E02-P2-009 | ✅ test_three_puts_produce_three_versions_ascending |
| E02-P2-010 | ✅ 5 tests (4 × 422 + boundary 200) |

---

## Implementation Guidance (GREEN PHASE)

### Endpoints to implement

```
GET  /api/v1/companies/{company_id}/espd-profile
PUT  /api/v1/companies/{company_id}/espd-profile
PATCH /api/v1/companies/{company_id}/espd-profile
GET  /api/v1/companies/{company_id}/espd-profile/versions
```

### Files to create

```
eusolicit-app/services/client-api/
  src/client_api/
    schemas/espd.py                          ← Task 1 (ESPDFieldValues, ESPDProfileResponse, etc.)
    services/espd_service.py                 ← Task 2 (get, upsert full/partial, audit)
    api/v1/espd.py                           ← Task 3 (router with 4 endpoints)
  alembic/versions/008_espd_profile_unique_version.py  ← Task 5
```

### Files to modify

```
src/client_api/main.py                       ← Task 4 (register espd_v1.router)
```

### Key implementation rules (from Dev Notes)

1. `from __future__ import annotations` at top of every new file
2. Every service function calls `_assert_own_company()` before any DB query
3. Versioning is append-only (INSERT, never UPDATE)
4. `PUT` auth guard: `require_role("bid_manager")` (permits admin + bid_manager)
5. `PATCH` auth guard: `require_role("bid_manager")` (same)
6. `GET` auth guard: `get_current_user` (all authenticated members)
7. Audit: `action_type="create"` for first write; `"update"` for subsequent writes
8. `_espd_to_snapshot()` returns `{"field_values": profile.field_values, "version": profile.version}`
9. PATCH merge: `merged = {**existing.field_values, **patch.model_dump(exclude_none=True, mode="json")}` (top-level only, no deep merge)
10. GET /versions returns 404 (not empty list) when no profile exists

### GREEN PHASE Steps

After implementing Story 2.12:

1. Remove all `@pytest.mark.skip` decorators from `test_espd_profile.py`
2. Remove all `@pytest.mark.skip` decorators from the `TestAuditTrailEspd` class in `test_audit_trail.py`
3. Run: `pytest tests/api/test_espd_profile.py tests/api/test_audit_trail.py -v`
4. All 31 tests should **PASS** (green phase)
5. If any fail: either fix the implementation (feature bug) or fix the test (test bug — document the finding)
6. Commit passing tests with reference to Story 2.12

---

## Key Risks & Assumptions

| Item | Risk | Mitigation |
|------|------|-----------|
| Cross-tenant tests create Company B inline | Medium — two sessions sharing one app instance; may have dependency override conflicts | Each cross-tenant test sets up + tears down its own session with rollback |
| `require_role("bid_manager")` — covers admin too | Confirmed from companies.py pattern | AC4 role tests include admin (same fixture user) and bid_manager (registered via helper) |
| PATCH merge is top-level only (not deep) | E02-R-008 gap possible for deep-nested payloads | Tests only verify top-level section replacement, matching Dev Note 8 |
| Audit log `before` uses `_espd_to_snapshot()` format | `before` dict contains `field_values` and `version` keys | Tests assert `before.get("field_values")` and `after.get("field_values")` per Dev Note 5 |

---

*Generated by TEA Master Test Architect via bmad-testarch-atdd workflow.*
*Story: 2-12-espd-profile-crud | Epic: E02 — Authentication & Identity*
