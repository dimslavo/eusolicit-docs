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
workflowType: atdd
storyId: 2-10-entity-level-rbac-middleware
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/2-10-entity-level-rbac-middleware.md
  - eusolicit-docs/test-artifacts/test-design-epic-02.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_team_members.py
  - eusolicit-app/services/client-api/tests/api/test_company_profile.py
  - eusolicit-app/services/client-api/tests/unit/test_security.py
  - eusolicit-app/services/client-api/src/client_api/models/entity_permission.py
  - eusolicit-app/services/client-api/src/client_api/models/audit_log.py
  - eusolicit-app/services/client-api/src/client_api/models/enums.py
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit-docs/_bmad/bmm/config.yaml
  - eusolicit-docs/_bmad/tea/config.yaml
---

# ATDD Checklist: Story 2.10 — Entity-Level RBAC Middleware

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated, implementation pending
**Story:** `eusolicit-docs/implementation-artifacts/2-10-entity-level-rbac-middleware.md`

---

## Preflight Summary

| Item | Result |
|------|--------|
| Stack detected | `backend` (Python FastAPI — no frontend indicators in project root) |
| Story stack | Backend-only (Python/FastAPI, new `src/client_api/core/rbac.py`) |
| Test framework | `pytest` + `pytest-asyncio` + `httpx` (project convention) |
| Story status | `ready-for-dev` — all 7 ACs clear and testable |
| Epic test design | Loaded (`test-design-epic-02.md`) — P0 risks E02-R-001 (RBAC ceiling, score 6), E02-R-002 (cross-tenant, score 6) |
| TEA config flags | `tea_use_playwright_utils: true` (API-only profile — no browser automation), `tea_use_pactjs_utils: false`, `tea_execution_mode: auto` |
| Generation mode | AI generation (pure backend, no browser recording needed) |
| Knowledge fragments loaded | `data-factories`, `test-quality`, `test-healing-patterns`, `test-levels-framework`, `test-priorities-matrix`, `error-handling` |
| Existing patterns used | `test_team_members.py` (fixture pattern, `_register_and_verify_with_role`), `test_security.py` (unit mock pattern with deferred imports) |

---

## Generation Mode

**Selected:** AI generation (sequential)

**Rationale:** Story 2.10 is a pure backend FastAPI dependency. All acceptance criteria describe
RBAC logic, HTTP status codes, database state changes (entity_permissions, audit_log), and
in-memory cache behaviour — no UI interaction required. Backend pattern:
`pytest + pytest-asyncio + httpx` matching project conventions from `test_security.py` (unit mocks)
and `test_team_members.py` (API integration with shared-session fixture and SQL seeding).

---

## Test Strategy

### AC → Test Scenario Mapping

| AC | Description | Test Scenarios | Level | Priority | Epic ID |
|----|-------------|----------------|-------|----------|---------|
| AC1 | `check_entity_access(entity_type, required_permission)` returns async callable | Factory returns callable; inner dep is coroutine; optional `entity_id_param` accepted | Unit | P1 | — |
| AC2 | No entity_permissions row → 403 (explicit grant model) | contributor/reviewer/read_only with no row → 403 | Unit + API | P0 | E02-P2-018 |
| AC3 | Role ceiling enforcement: contributor≤write, reviewer/read_only≤read | manage grant clamped to ceiling; reviewer+write grant still fails write req; contributor+manage grant fails manage req but passes write req | Unit + API | P0 | E02-P0-007 |
| AC4 | admin/bid_manager bypass (own company only) | bypass roles get 200 with no row; Company A admin → 403 for Company B entity | Unit + API | P0 | E02-P0-008, E02-P0-009 |
| AC5 | Single optimised query per check (no N+1) | `_resolve_entity_permission` called exactly once per HTTP request | API | P1 | — |
| AC6 | Denial logged to `shared.audit_log` (non-blocking) | 403 → audit_log row: action_type="access_denied", entity_type, entity_id, user_id; response still 403 not 500 | API | P2 | E02-P2-016 |
| AC7 | Per-request cache via `request.state._rbac_cache` | Second dep call in same request hits cache; different entity_ids separate entries; cache initialised when missing | Unit + API | P1 | — |

### Coverage: Epic Test Design Assignments

| Test ID | Level | Scenario | Covered By |
|---------|-------|----------|------------|
| **E02-P0-007** | API | contributor + `entity_permissions.permission=manage` calling route requiring manage → 403 | `TestRoleCeilingEnforcement::test_contributor_with_manage_grant_denied_on_manage_route` |
| **E02-P0-007 (clarification)** | API | contributor + manage grant + route requiring write → 200 (ceiling=write satisfies) | `TestRoleCeilingEnforcement::test_contributor_with_manage_grant_allowed_on_write_route` |
| **E02-P0-008** | API | admin/bid_manager with no entity_permissions row → 200 | `TestBypassRolesOwnCompany::test_bypass_role_gets_200_with_no_entity_permissions_row` |
| **E02-P0-009** | API | Company A admin targeting Company B entity_id → 403 | `TestCrossCompanyIsolation::test_company_a_admin_denied_on_company_b_entity` |
| **E02-P2-016** | API | access denial → audit_log row with action_type="access_denied" | `TestAuditLogOnDenial::test_access_denial_creates_audit_log_row` |
| **E02-P2-018** | Unit | no entity_permissions row → denied for contributor/reviewer/read_only | `TestExplicitGrantModel::test_no_entity_permission_row_raises_forbidden_for_non_bypass_roles` |

### Red Phase Design Decisions

- All tests use `@pytest.mark.skip(reason="ATDD red phase: Story 2.10 not yet implemented — create src/client_api/core/rbac.py first.")`
- Tests assert **expected** behaviour with realistic assertions (no placeholder `assert True`)
- All `client_api.core.rbac` imports deferred to test/fixture bodies (`# noqa: PLC0415`) — prevents ImportError at collection time
- Tests will fail when unskipped because:
  1. `client_api.core.rbac` module does not exist → `ImportError` on every import
  2. `GET /api/v1/test-rbac/{entity_id}` route not registered → 404 on all API calls
  3. No entity_permissions query logic → no row found → exception path not implemented

---

## Generated Test Files

### File 1: Unit Tests

**Path:** `eusolicit-app/services/client-api/tests/unit/test_rbac.py`

**Test Classes and Methods:**

| Class | Method | AC | Priority | Epic ID |
|-------|--------|----|----------|---------|
| `TestPermissionLevel` | `test_all_three_permissions_present` | — | P1 | — |
| `TestPermissionLevel` | `test_read_is_lower_than_write` | — | P1 | — |
| `TestPermissionLevel` | `test_write_is_lower_than_manage` | — | P1 | — |
| `TestPermissionLevel` | `test_ordering_is_strictly_ascending` | — | P1 | — |
| `TestPermissionLevel` | `test_all_values_are_positive_integers` | — | P1 | — |
| `TestPermissionLevel` | `test_exact_specification_values` | — | P1 | — |
| `TestRolePermissionCeiling` | `test_all_five_roles_have_ceilings` | AC3 | P0 | — |
| `TestRolePermissionCeiling` | `test_admin_ceiling_is_manage` | AC3 | P0 | — |
| `TestRolePermissionCeiling` | `test_bid_manager_ceiling_is_manage` | AC3 | P0 | — |
| `TestRolePermissionCeiling` | `test_contributor_ceiling_is_write` | AC3 | P0 | — |
| `TestRolePermissionCeiling` | `test_reviewer_ceiling_is_read` | AC3 | P0 | — |
| `TestRolePermissionCeiling` | `test_read_only_ceiling_is_read` | AC3 | P0 | — |
| `TestRolePermissionCeiling` | `test_all_ceiling_values_are_valid_permission_strings` | AC3 | P1 | — |
| `TestBypassRoles` | `test_bypass_roles_is_frozenset` | AC4 | P1 | — |
| `TestBypassRoles` | `test_admin_is_bypass_role` | AC4 | P0 | — |
| `TestBypassRoles` | `test_bid_manager_is_bypass_role` | AC4 | P0 | — |
| `TestBypassRoles` | `test_non_bypass_roles_are_excluded[contributor]` | AC4 | P0 | — |
| `TestBypassRoles` | `test_non_bypass_roles_are_excluded[reviewer]` | AC4 | P0 | — |
| `TestBypassRoles` | `test_non_bypass_roles_are_excluded[read_only]` | AC4 | P0 | — |
| `TestBypassRoles` | `test_exactly_two_bypass_roles` | AC4 | P1 | — |
| `TestCheckEntityAccessFactory` | `test_factory_returns_callable` | AC1 | P1 | — |
| `TestCheckEntityAccessFactory` | `test_factory_returns_callable_for_all_valid_combinations` | AC1 | P1 | — |
| `TestCheckEntityAccessFactory` | `test_factory_accepts_entity_id_param_override` | AC1 | P1 | — |
| `TestCheckEntityAccessFactory` | `test_inner_dependency_is_async_coroutine` | AC1 | P1 | — |
| `TestCeilingEnforcement` | `test_manage_grant_is_clamped_to_role_ceiling[contributor-write]` | AC3 | P0 | E02-P0-007 unit |
| `TestCeilingEnforcement` | `test_manage_grant_is_clamped_to_role_ceiling[reviewer-read]` | AC3 | P0 | — |
| `TestCeilingEnforcement` | `test_manage_grant_is_clamped_to_role_ceiling[read_only-read]` | AC3 | P0 | — |
| `TestCeilingEnforcement` | `test_write_grant_is_clamped_to_read_for_restricted_roles[reviewer-read]` | AC3 | P0 | — |
| `TestCeilingEnforcement` | `test_write_grant_is_clamped_to_read_for_restricted_roles[read_only-read]` | AC3 | P0 | — |
| `TestCeilingEnforcement` | `test_contributor_write_grant_satisfies_read_requirement` | AC3 | P1 | — |
| `TestCeilingEnforcement` | `test_contributor_write_grant_does_not_satisfy_manage_requirement` | AC3 | P0 | — |
| `TestCeilingEnforcement` | `test_reviewer_read_grant_satisfies_read_requirement` | AC3 | P1 | — |
| `TestCeilingEnforcement` | `test_reviewer_read_grant_does_not_satisfy_write_requirement` | AC3 | P0 | — |
| `TestExplicitGrantModel` | `test_no_entity_permission_row_raises_forbidden_for_non_bypass_roles[contributor]` | AC2 | P0 | E02-P2-018 |
| `TestExplicitGrantModel` | `test_no_entity_permission_row_raises_forbidden_for_non_bypass_roles[reviewer]` | AC2 | P0 | E02-P2-018 |
| `TestExplicitGrantModel` | `test_no_entity_permission_row_raises_forbidden_for_non_bypass_roles[read_only]` | AC2 | P0 | E02-P2-018 |
| `TestExplicitGrantModel` | `test_bypass_roles_do_not_call_resolve_entity_permission[admin]` | AC4 | P0 | — |
| `TestExplicitGrantModel` | `test_bypass_roles_do_not_call_resolve_entity_permission[bid_manager]` | AC4 | P0 | — |
| `TestExplicitGrantModel` | `test_explicit_deny_contributor_read_grant_insufficient_for_write` | AC3 | P0 | — |
| `TestExplicitGrantModel` | `test_explicit_allow_contributor_write_grant_sufficient_for_read` | AC3 | P1 | — |
| `TestPerRequestCache` | `test_second_dep_call_uses_cache_not_db` | AC7 | P1 | — |
| `TestPerRequestCache` | `test_cache_initialised_when_state_has_no_rbac_cache_attribute` | AC7 | P1 | — |
| `TestPerRequestCache` | `test_different_entity_ids_have_separate_cache_entries` | AC7 | P1 | — |

**Total unit tests: 42** (including parametrized variants)

---

### File 2: API Integration Tests

**Path:** `eusolicit-app/services/client-api/tests/api/test_rbac_middleware.py`

**Test Router Mounted:**
- `GET /api/v1/test-rbac/{entity_id}` → `Depends(check_entity_access("test_entity", "write"))`
- `GET /api/v1/test-rbac-manage/{entity_id}` → `Depends(check_entity_access("test_entity", "manage"))`

**Test Classes and Methods:**

| Class | Method | AC | Priority | Epic ID |
|-------|--------|----|----------|---------|
| `TestNoEntityPermissionRow` | `test_no_row_returns_403_for_non_bypass_role[contributor]` | AC2 | P0 | E02-P2-018 |
| `TestNoEntityPermissionRow` | `test_no_row_returns_403_for_non_bypass_role[reviewer]` | AC2 | P0 | E02-P2-018 |
| `TestNoEntityPermissionRow` | `test_no_row_returns_403_for_non_bypass_role[read_only]` | AC2 | P0 | E02-P2-018 |
| `TestRoleCeilingEnforcement` | `test_contributor_with_manage_grant_denied_on_manage_route` | AC3 | P0 | E02-P0-007 |
| `TestRoleCeilingEnforcement` | `test_contributor_with_manage_grant_allowed_on_write_route` | AC3 | P0 | E02-P0-007 clarification |
| `TestRoleCeilingEnforcement` | `test_reviewer_and_read_only_denied_on_write_route_even_with_write_grant[reviewer]` | AC3 | P0 | — |
| `TestRoleCeilingEnforcement` | `test_reviewer_and_read_only_denied_on_write_route_even_with_write_grant[read_only]` | AC3 | P0 | — |
| `TestBypassRolesOwnCompany` | `test_bypass_role_gets_200_with_no_entity_permissions_row[admin]` | AC4 | P0 | E02-P0-008 |
| `TestBypassRolesOwnCompany` | `test_bypass_role_gets_200_with_no_entity_permissions_row[bid_manager]` | AC4 | P0 | E02-P0-008 |
| `TestBypassRolesOwnCompany` | `test_bypass_role_gets_200_even_for_manage_requirement[admin]` | AC4 | P1 | — |
| `TestBypassRolesOwnCompany` | `test_bypass_role_gets_200_even_for_manage_requirement[bid_manager]` | AC4 | P1 | — |
| `TestCrossCompanyIsolation` | `test_non_bypass_role_cannot_access_other_company_entity[contributor]` | AC4 | P0 | E02-P0-009 |
| `TestCrossCompanyIsolation` | `test_non_bypass_role_cannot_access_other_company_entity[reviewer]` | AC4 | P0 | E02-P0-009 |
| `TestCrossCompanyIsolation` | `test_non_bypass_role_cannot_access_other_company_entity[read_only]` | AC4 | P0 | E02-P0-009 |
| `TestCrossCompanyIsolation` | `test_company_a_contributor_without_entity_permissions_denied_company_b_entity` | AC2+AC4 | P0 | E02-P0-009 |
| `TestCrossCompanyIsolation` | `test_company_a_admin_denied_on_company_b_entity` | AC4 | P0 | E02-P0-009 |
| `TestSingleQueryConstraint` | `test_single_resolve_call_per_permission_check` | AC5 | P1 | — |
| `TestAuditLogOnDenial` | `test_access_denial_creates_audit_log_row` | AC6 | P2 | E02-P2-016 |
| `TestAuditLogOnDenial` | `test_audit_log_contains_entity_id_and_ip_address` | AC6 | P2 | E02-P2-016 |
| `TestAuditLogOnDenial` | `test_denial_audit_write_does_not_block_403_response` | AC6 | P2 | — |
| `TestPerRequestCacheIntegration` | `test_resolve_called_once_per_http_request_with_mock` | AC7 | P1 | — |
| `TestPerRequestCacheIntegration` | `test_second_http_request_does_not_use_previous_request_cache` | AC7 | P1 | — |
| `TestUnauthenticatedAccess` | `test_no_authorization_header_returns_401` | — | P1 | — |

**Total API integration tests: 23** (including parametrized variants)

---

## Acceptance Criteria Coverage Summary

| AC | Description | Status | Test Count |
|----|-------------|--------|-----------|
| AC1 | Factory returns async callable | ✅ Covered | 4 unit |
| AC2 | No row → deny | ✅ Covered | 3 unit + 3 API |
| AC3 | Role ceiling enforcement | ✅ Covered | 11 unit + 4 API |
| AC4 | admin/bid_manager bypass (own company) | ✅ Covered | 5 unit + 9 API |
| AC5 | Single query per check | ✅ Covered | 1 API |
| AC6 | Audit log on denial | ✅ Covered | 3 API |
| AC7 | Per-request cache | ✅ Covered | 3 unit + 2 API |

---

## TDD Red Phase Status

✅ Failing tests generated

**Total tests:** 65 (42 unit + 23 API integration)

**All tests are `@pytest.mark.skip`** — RED PHASE.

**Why tests fail when unskipped:**
1. `from client_api.core.rbac import check_entity_access` → `ModuleNotFoundError` (file does not exist)
2. `GET /api/v1/test-rbac/{entity_id}` → `404 Not Found` (route not registered)
3. No PERMISSION_LEVEL, ROLE_PERMISSION_CEILING, _BYPASS_ROLES → `NameError` / `AttributeError`
4. No audit_log write logic → `shared.audit_log` row not created → assert fails
5. No per-request cache → `_resolve_entity_permission` called N times instead of 1

---

## ⚠️ Design Clarification Required

### Cross-Company Admin Bypass (E02-P0-009 vs AC4)

**Potential ambiguity between AC4 and Dev Note 7:**

- **AC4** states: admin/bid_manager bypass entity-level checks "for entities that belong to the current user's own company"
- **Task 1.10** states: "if current_user.role is in _BYPASS_ROLES → cache result as CurrentUser and return immediately (bypass)" — no company_id check
- **Dev Note 7** states: "bypass path still relies on `current_user.company_id` for the query scope"
- **E02-P0-009** expects: Company A admin → 403 for Company B entity

**Reconciliation guidance for dev:**
For Company A admin to receive 403 on Company B entity, the bypass cannot be completely unconditional. One of these implementations satisfies both AC4 and E02-P0-009:

1. **Option A (recommended):** Even for bypass roles, query `entity_permissions` filtered by `company_id = current_user.company_id`. If a row exists for ANY user in the company on this entity → grant. If no row → deny (entity not in this company's scope). This preserves company_id scoping without requiring the admin to have their own row.

2. **Option B:** Extract `company_id` from path params (e.g., `/companies/{company_id}/...`) and compare to `current_user.company_id`. If mismatch → 403. This requires routes to include `company_id` in path.

3. **Option C:** The bypass is unconditional (all admins access all entities), and E02-P0-009 applies only to non-bypass roles — document this if intentional.

**Test `test_company_a_admin_denied_on_company_b_entity` will fail if Option C is chosen** — discuss with team before implementation.

---

## Next Steps (TDD Green Phase)

After implementing `src/client_api/core/rbac.py`:

1. Remove `@pytest.mark.skip` from all test classes in:
   - `tests/unit/test_rbac.py`
   - `tests/api/test_rbac_middleware.py`

2. Run unit tests:
   ```bash
   cd eusolicit-app/services/client-api
   python -m pytest tests/unit/test_rbac.py -v --tb=short
   ```

3. Run API integration tests (requires Docker Compose services):
   ```bash
   python -m pytest tests/api/test_rbac_middleware.py -v --tb=short
   ```

4. Verify all tests **PASS** (green phase)

5. If any tests fail:
   - `ModuleNotFoundError` → `rbac.py` not created or wrong path
   - `ImportError` on `PERMISSION_LEVEL` → constant not defined in `rbac.py`
   - `403 when 200 expected` → bypass logic not implemented for admin/bid_manager
   - `200 when 403 expected` → ceiling enforcement not applied
   - `assert mock_resolve.call_count == 1` fails → cache not implemented
   - `audit_log row not found` → `session.add(AuditLog(...)); await session.flush()` missing

6. Commit passing tests with story reference.

---

## Implementation Checklist (from Story Dev Notes)

Before removing skip markers, verify:

- [ ] `src/client_api/core/rbac.py` created (NOT `security.py`)
- [ ] `from __future__ import annotations` at top
- [ ] `PERMISSION_LEVEL = {"read": 1, "write": 2, "manage": 3}`
- [ ] `ROLE_PERMISSION_CEILING = {"admin": "manage", "bid_manager": "manage", "contributor": "write", "reviewer": "read", "read_only": "read"}`
- [ ] `_BYPASS_ROLES: frozenset[str] = frozenset({"admin", "bid_manager"})`
- [ ] `_resolve_entity_permission` uses single SELECT with composite index
- [ ] `check_entity_access` factory returns async coroutine
- [ ] Ceiling clamped: `effective = min(PERMISSION_LEVEL[granted], PERMISSION_LEVEL[ceiling])`
- [ ] Bypass logic implemented (reconcile with E02-P0-009 design decision above)
- [ ] Per-request cache via `request.state._rbac_cache` initialised
- [ ] `session.add(AuditLog(...)); await session.flush()` on every denial
- [ ] `from eusolicit_common.exceptions import ForbiddenError` for 403
- [ ] No `session.commit()` in dependency function
- [ ] No imports of `rbac.py` into `security.py` (circular import risk)
- [ ] `check_entity_access` exported from `rbac.py` (importable as `from client_api.core.rbac import check_entity_access`)
