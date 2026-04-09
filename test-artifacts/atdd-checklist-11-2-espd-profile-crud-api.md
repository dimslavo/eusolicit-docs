---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-09'
workflowType: 'testarch-atdd'
storyKey: '11-2-espd-profile-crud-api'
detectedStack: 'backend'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/11-2-espd-profile-crud-api.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-11.md'
  - '_bmad/bmm/config.yaml'
  - 'eusolicit-app/services/client-api/tests/api/test_company_profile.py'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
---

# ATDD Checklist — Epic 11, Story 11.2: ESPD Profile CRUD API

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**Primary Test Level:** API / Integration (pytest + pytest-asyncio + httpx)
**TDD Phase:** 🔴 RED — All tests skipped pending Story 11.2 implementation

---

## Story Summary

Story 11.2 replaces the old S2.12 versioned single-profile ESPD API with a new
multi-profile CRUD API at `/api/v1/espd-profiles`. The new design allows companies
to maintain multiple named ESPD profiles mapped to EU ESPD Parts II–V for reuse
across tender submissions.

**As a** company admin or bid manager,
**I want** to create, list, view, update, and delete named ESPD profiles for my company,
**So that** my company can maintain multiple reusable ESPD documents ready to use across different procurement tender submissions.

---

## Acceptance Criteria

1. **AC1** — `POST /api/v1/espd-profiles` creates profile → HTTP 201; `company_id` from JWT only; `admin`/`bid_manager` required; others → 403.
2. **AC2** — `GET /api/v1/espd-profiles` → HTTP 200; all company profiles; ordered by `created_at` DESC; accessible to all authenticated members.
3. **AC3** — `GET /api/v1/espd-profiles/{id}` → HTTP 200 (own company) or HTTP 404 (not found or cross-company); no 403.
4. **AC4** — `PATCH /api/v1/espd-profiles/{id}` → HTTP 200; `profile_name` replaced if provided; `espd_data` merged at Part level; HTTP 404 if not found/cross-company.
5. **AC5** — `DELETE /api/v1/espd-profiles/{id}` → HTTP 204 No Content; HTTP 404 if not found/cross-company.
6. **AC6** — Company-scoped RLS: all operations derive `company_id` from JWT; GET/PATCH/DELETE cross-company → HTTP 404 (not 403) to prevent UUID enumeration.
7. **AC7** — `espd_data` PATCH merge: Part-level units (`part_ii`–`part_v`); absent Parts left unchanged; no recursive merge.
8. **AC8** — `espd_data` validation: must be a dict; each recognised Part key must be dict or absent; non-dict Part → HTTP 422 with field-level error.
9. **AC9** — Unauthenticated → HTTP 401 on all endpoints; `POST`/`PATCH`/`DELETE` require `admin` or `bid_manager`; others → HTTP 403; GET operations accessible to all authenticated members.

---

## Test File

**File:** `eusolicit-app/services/client-api/tests/api/test_espd_profile.py` (1361 lines)

**Note:** This file completely replaces the S2.12 test suite. The old tests covered
the versioned single-profile API at `/api/v1/companies/{id}/espd-profile` which no
longer exists after Story 11.01's schema migration.

---

## Failing Tests Created (RED Phase)

All 33 test methods marked `@pytest.mark.skip(reason="ATDD RED PHASE: Story 11.2 not yet implemented.")`. With parametrization (`@pytest.mark.parametrize` on 3-role suites), this expands to **41 pytest test cases** (verified by `pytest --collect-only`).

### TestAC1CreateEspdProfile — 7 tests (AC1, AC8, AC9, E11-P1-008)

| Test | Status | AC | Verifies |
|------|--------|----|---------|
| `test_post_creates_profile_with_all_parts_returns_201` | 🔴 RED | AC1 | POST full espd_data → 201; all fields in response; company_id from JWT |
| `test_post_with_empty_espd_data_returns_201` | 🔴 RED | AC1 | POST name only → 201; espd_data defaults to {} |
| `test_post_missing_profile_name_returns_422` | 🔴 RED | AC8 | Missing profile_name → 422 |
| `test_post_empty_profile_name_returns_422` | 🔴 RED | AC8 | Empty string profile_name → 422 (min_length=1) |
| `test_post_profile_name_over_255_chars_returns_422` | 🔴 RED | AC8 | 256-char profile_name → 422 (max_length=255) |
| `test_post_invalid_part_type_returns_422` | 🔴 RED | AC8 | part_iii="not_a_dict" → 422 with detail |
| `test_unauthenticated_post_returns_401` | 🔴 RED | AC9 | POST without token → 401 |

### TestAC2ListEspdProfiles — 4 tests (AC2, AC9, E11-P1-008)

| Test | Status | AC | Verifies |
|------|--------|----|---------|
| `test_get_list_empty_returns_200_with_empty_list` | 🔴 RED | AC2 | GET before any POST → 200; profiles=[]; total=0 |
| `test_get_list_returns_profiles_for_company` | 🔴 RED | AC2 | POST 2 → GET list → total=2; both present |
| `test_get_list_ordered_by_created_at_desc` | 🔴 RED | AC2 | POST A then B → list[0] is B (newest first) |
| `test_unauthenticated_list_returns_401` | 🔴 RED | AC9 | GET without token → 401 |

### TestAC3GetEspdProfile — 3 tests (AC3, AC9, E11-P1-008, E11-P0-001)

| Test | Status | AC | Verifies |
|------|--------|----|---------|
| `test_get_detail_returns_200_with_correct_data` | 🔴 RED | AC3 | POST → GET/{id} → 200; all fields match |
| `test_get_nonexistent_profile_returns_404` | 🔴 RED | AC3, AC6 | Random UUID → 404 (not 403) |
| `test_unauthenticated_get_returns_401` | 🔴 RED | AC9 | GET/{id} without token → 401 |

### TestAC4PatchEspdProfile — 6 tests (AC4, AC7, AC9, E11-P1-008, E11-P2-005)

| Test | Status | AC | Verifies |
|------|--------|----|---------|
| `test_patch_profile_name_only_returns_200` | 🔴 RED | AC4 | PATCH name only → 200; espd_data unchanged |
| `test_patch_espd_data_part_level_merge` | 🔴 RED | AC7, E11-P2-005 | PATCH part_iii → part_iii replaced; part_ii/iv/v unchanged |
| `test_patch_both_fields_returns_200` | 🔴 RED | AC4 | PATCH both name+espd_data → 200; both updated |
| `test_patch_empty_body_returns_200_unchanged` | 🔴 RED | AC4 | PATCH {} → 200; no-op; nothing changes |
| `test_patch_nonexistent_profile_returns_404` | 🔴 RED | AC4, AC6 | PATCH random UUID → 404 |
| `test_unauthenticated_patch_returns_401` | 🔴 RED | AC9 | PATCH without token → 401 |

### TestAC5DeleteEspdProfile — 3 tests (AC5, AC9, E11-P1-008)

| Test | Status | AC | Verifies |
|------|--------|----|---------|
| `test_delete_profile_returns_204` | 🔴 RED | AC5 | POST → DELETE → 204; subsequent GET → 404 |
| `test_delete_nonexistent_profile_returns_404` | 🔴 RED | AC5, AC6 | DELETE random UUID → 404 |
| `test_unauthenticated_delete_returns_401` | 🔴 RED | AC9 | DELETE without token → 401 |

### TestAC6CrossCompanyRLS — 5 tests (AC6, E11-P0-001, E11-P2-004)

| Test | Status | AC | Verifies |
|------|--------|----|---------|
| `test_company_a_cannot_get_company_b_profile_returns_404` | 🔴 RED | AC6, E11-P0-001 | Company B JWT → GET Company A profile → **404** (not 403) |
| `test_company_a_cannot_patch_company_b_profile_returns_404` | 🔴 RED | AC6, E11-P2-004 | Company B JWT → PATCH Company A profile → **404** |
| `test_company_a_cannot_delete_company_b_profile_returns_404` | 🔴 RED | AC6, E11-P2-004 | Company B JWT → DELETE Company A profile → **404** |
| `test_post_company_id_in_body_is_ignored` | 🔴 RED | AC1, AC6 | POST with injected company_id → 201; bound to JWT company |
| `test_list_only_returns_own_company_profiles` | 🔴 RED | AC2, AC6 | Company A list → only Company A profiles (total=2, not 3) |

### TestAC9RoleEnforcement — 5 methods × parametrize = 13 pytest cases (AC9)

| Test | Params | Status | Verifies |
|------|--------|--------|---------|
| `test_low_privilege_roles_cannot_post` | contributor, reviewer, read_only | 🔴 RED × 3 | Low-privilege POST → 403 |
| `test_low_privilege_roles_cannot_patch` | contributor, reviewer, read_only | 🔴 RED × 3 | Low-privilege PATCH → 403 |
| `test_low_privilege_roles_cannot_delete` | contributor, reviewer, read_only | 🔴 RED × 3 | Low-privilege DELETE → 403 |
| `test_bid_manager_can_post_and_patch_and_delete` | — | 🔴 RED × 1 | bid_manager → 201/200/204 |
| `test_all_roles_can_get_list_and_detail` | contributor, reviewer, read_only | 🔴 RED × 3 | All roles → 200 on GET list+detail |

---

## Test Infrastructure

### Fixture: `espd_client_and_session`

Follows the `company_client_and_session` pattern from `test_company_profile.py` exactly.

**Pattern:**
- Registers unique admin user + company via `POST /api/v1/auth/register`
- Sets `email_verified = TRUE` via SQL (bypasses email delivery)
- Logs in to obtain real JWT (encodes `company_id` + `role=admin`)
- Yields `(client, session, access_token, company_id_str)`
- Rolls back in `finally` — no persistent state

**Why shared session:** Multi-step state inspection (POST then inspect DB, cross-company assertions) requires all flushed writes to be visible within the same transaction.

### Helper: `_register_and_verify_with_role`

Reused from `test_company_profile.py` pattern.

**Algorithm:**
1. Register user2 (creates TempCo + admin membership)
2. Verify email via SQL UPDATE
3. Null-out TempCo membership (`accepted_at = NULL`) — forces login JOIN to skip it
4. Insert target company membership with desired role
5. Login → JWT encodes `company_id = target_company_id`, `role = role`

**Used by:** `TestAC9RoleEnforcement` (all 5 tests)

### Constants

```python
ESPD_FULL_DATA = {
    "part_ii": {"operator_name": "Test Corp", "registration_number": "BG123456789"},
    "part_iii": {"criminal_convictions": False, "corruption": False, "fraud": False},
    "part_iv": {"economic_financial": {"annual_turnover": 5000000}, "technical_professional": {}},
    "part_v": {"reduction_of_candidates": {}}
}
ESPD_CREATE_PAYLOAD = {"profile_name": "Test ESPD Profile", "espd_data": ESPD_FULL_DATA}
```

---

## Epic Test ID Coverage

| Epic Test ID | Priority | Covered By | Tests |
|-------------|----------|-----------|-------|
| **E11-P0-001** | P0 🔴 | `TestAC6CrossCompanyRLS.test_company_a_cannot_get_company_b_profile_returns_404` | 1 |
| **E11-P0-001** | P0 🔴 | `TestAC6CrossCompanyRLS.test_company_a_cannot_patch_company_b_profile_returns_404` | 1 |
| **E11-P0-001** | P0 🔴 | `TestAC6CrossCompanyRLS.test_company_a_cannot_delete_company_b_profile_returns_404` | 1 |
| **E11-P1-008** | P1 | `TestAC1–5` (smoke: create, list, get, update, delete) | 5+ |
| **E11-P2-004** | P2 | `TestAC6CrossCompanyRLS` (PATCH + DELETE cross-company) | 2 |
| **E11-P2-005** | P2 | `TestAC4PatchEspdProfile.test_patch_espd_data_part_level_merge` | 1 |

**Risk E11-R-005 mitigation (SEC, score 6):** `TestAC6CrossCompanyRLS` directly verifies the 404-not-403 enumeration prevention pattern. These are **P0 severity** per QA gate.

---

## Implementation Checklist (for dev-story agent)

### To make `TestAC1CreateEspdProfile` pass:
- [ ] Task 1.3–1.4: Define `ESPDData` + `ESPDProfileCreateRequest` Pydantic models
- [ ] Task 2.4: Implement `create_espd_profile()` service function
- [ ] Task 3.3: Implement `POST /` route with `require_role("bid_manager")` guard
- [ ] Task 4.1: Confirm router registered in `main.py`

### To make `TestAC2ListEspdProfiles` pass:
- [ ] Task 1.7: Define `ESPDProfileListResponse` schema
- [ ] Task 2.5: Implement `list_espd_profiles()` with `ORDER BY created_at DESC`
- [ ] Task 3.4: Implement `GET /` route with `get_current_user` (any auth)

### To make `TestAC3GetEspdProfile` pass:
- [ ] Task 1.6: Define `ESPDProfileResponse` schema
- [ ] Task 2.6: Implement `get_espd_profile()` with `_assert_company_owns_profile()` → 404
- [ ] Task 3.5: Implement `GET /{profile_id}` route

### To make `TestAC4PatchEspdProfile` pass:
- [ ] Task 1.5: Define `ESPDProfilePatchRequest` schema (both optional)
- [ ] Task 2.7: Implement `update_espd_profile()` with Part-level merge semantics
- [ ] Task 3.6: Implement `PATCH /{profile_id}` route

### To make `TestAC5DeleteEspdProfile` pass:
- [ ] Task 2.8: Implement `delete_espd_profile()` → `session.delete()` + flush
- [ ] Task 3.7: Implement `DELETE /{profile_id}` route → `Response(status_code=204)`

### To make `TestAC6CrossCompanyRLS` pass:
- [ ] Task 2.3: Implement `_assert_company_owns_profile()` raising HTTP 404 (not 403)
- [ ] Task 2.4: Verify `company_id` in POST comes from `current_user.company_id`, never request body
- [ ] Task 2.5: Verify `list_espd_profiles()` filters by `company_id` in WHERE clause (not Python-level)

### To make `TestAC9RoleEnforcement` pass:
- [ ] Task 3.3, 3.6, 3.7: Use `require_role("bid_manager")` on POST/PATCH/DELETE
- [ ] Task 3.4, 3.5: Use `get_current_user` (no role check) on GET list + GET detail

---

## Running Tests

```bash
# Run all S11.02 failing tests for this story (RED phase — all skip)
cd eusolicit-app/services/client-api
pytest tests/api/test_espd_profile.py -v

# Run with skip report (shows all skipped tests and their reasons)
pytest tests/api/test_espd_profile.py -v --tb=short -rs

# Run specific test class
pytest tests/api/test_espd_profile.py::TestAC6CrossCompanyRLS -v

# Run specific test
pytest tests/api/test_espd_profile.py::TestAC4PatchEspdProfile::test_patch_espd_data_part_level_merge -v

# After removing skip decorators — verify green phase:
pytest tests/api/test_espd_profile.py -v --tb=short

# Run with coverage
pytest tests/api/test_espd_profile.py --cov=client_api --cov-report=term-missing -v
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

**TEA Agent Responsibilities:**

- ✅ All 33 test methods (42 pytest cases with parametrization) written
- ✅ All tests marked `@pytest.mark.skip(reason=_SKIP_REASON)` — TDD red phase
- ✅ All tests assert expected behavior (not placeholders)
- ✅ `espd_client_and_session` fixture follows established pattern
- ✅ `_register_and_verify_with_role` helper included
- ✅ Constants `ESPD_FULL_DATA` + `ESPD_CREATE_PAYLOAD` defined
- ✅ Cross-company pattern documented and implemented in `TestAC6CrossCompanyRLS`
- ✅ ATDD checklist saved
- ✅ Old S2.12 test file fully replaced

**Expected RED phase behavior:**
All 41 test cases will appear as `SKIPPED` when run. This is intentional — they define the contract.

---

### GREEN Phase (DEV Agent — Next Steps)

1. **Implement Story 11.2** per tasks in `11-2-espd-profile-crud-api.md`:
   - Task 1: Rewrite `src/client_api/schemas/espd.py`
   - Task 2: Rewrite `src/client_api/services/espd_service.py`
   - Task 3: Rewrite `src/client_api/api/v1/espd.py`
   - Task 4: Verify `main.py` router registration

2. **Remove skip markers** from `test_espd_profile.py`:
   ```bash
   # Remove all @pytest.mark.skip lines from the test file
   sed -i '/@pytest.mark.skip(reason=_SKIP_REASON)/d' tests/api/test_espd_profile.py
   ```
   Or remove manually one class at a time as each AC is implemented.

3. **Run tests to verify green**:
   ```bash
   cd eusolicit-app/services/client-api
   pytest tests/api/test_espd_profile.py -v --tb=short
   ```

4. **Expected green phase:** All 41 test cases PASS.

5. **QA gate — P0 tests must pass at 100%** (E11-P0-001 cross-company RLS):
   - `TestAC6CrossCompanyRLS::test_company_a_cannot_get_company_b_profile_returns_404`
   - `TestAC6CrossCompanyRLS::test_company_a_cannot_patch_company_b_profile_returns_404`
   - `TestAC6CrossCompanyRLS::test_company_a_cannot_delete_company_b_profile_returns_404`

---

### REFACTOR Phase (After All Tests Pass)

- Review `espd_service.py` for potential optimisation (e.g., single DB query for ownership check)
- Verify structured logging events: `espd_profile.created`, `espd_profile.updated`, `espd_profile.deleted`
- Ensure no `session.commit()` calls — only `session.flush()` + `session.refresh()`
- Check `model_dump(exclude_none=True, mode="json")` serialisation for JSONB correctness

---

## Test Coverage Summary

| Metric | Value |
|--------|-------|
| Test methods | 33 |
| Pytest cases (with parametrize) | 41 |
| Acceptance criteria covered | AC1–AC9 (all 9) |
| Epic test IDs covered | E11-P0-001, E11-P1-008, E11-P2-004, E11-P2-005 |
| P0 security test assertions | 3 (E11-P0-001 RLS, 404-not-403) |
| Test levels | API / Integration only (pure backend, no E2E needed) |
| TDD Phase | 🔴 RED (all skipped, awaiting implementation) |

---

## Notes

- **E11-P1-009 (espd_data validation, missing Part III → 422):** Not in scope for S11.02 CRUD. Part III completeness is enforced at export time in S11.03. The AC8 validator covers invalid Part type (non-dict → 422) which is the relevant S11.02 validation.
- **E11-R-005 (ESPD RLS, score 6):** The 404-not-403 pattern is critical for security. Tests explicitly assert 404 and document the reason in assertion messages to guide developers.
- **`require_role("bid_manager")`** permits both `admin` and `bid_manager` roles (minimum-privilege gate, not exact match). This is consistent with all other write endpoints in the project.
- **Cross-company tests** use inline company registration within the test body, not a separate fixture, to keep the setup explicit and traceable.

---

## Knowledge Base References Applied

- **test-levels-framework.md** — API/Integration level selected (pure backend; no E2E)
- **test-priorities-matrix.md** — P0 for RLS/security (E11-R-005 score 6); P1 for CRUD smoke
- **data-factories.md** — Constants (`ESPD_FULL_DATA`, `ESPD_CREATE_PAYLOAD`) as inline factories
- **test-quality.md** — Given-When-Then structure; one assertion type per test; no placeholders
- **test-healing-patterns.md** — Explicit pre-condition assertions with descriptive failure messages

---

**Generated by BMad TEA Agent** — 2026-04-09
