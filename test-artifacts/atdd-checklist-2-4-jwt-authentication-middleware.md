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
workflowType: testarch-atdd
storyId: 2-4-jwt-authentication-middleware
detectedStack: backend
generationMode: ai
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/2-4-jwt-authentication-middleware.md
  - eusolicit-docs/test-artifacts/test-design-epic-02.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/src/client_api/core/security.py
  - resources/knowledge/data-factories.md
  - resources/knowledge/test-quality.md
  - resources/knowledge/test-levels-framework.md
---

# ATDD Checklist — Epic 2, Story 2.4: JWT Authentication Middleware

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**Primary Test Levels:** Unit (AC5/AC6), Integration/API (AC1–AC4)
**TDD Phase:** 🔴 RED — Tests are failing until Story 2.4 is implemented

---

## Story Summary

As a backend service, I want a `get_current_user` FastAPI dependency that validates
RS256 JWT Bearer tokens and a `require_role` factory that enforces company-level role
minimums, so that all protected endpoints can be secured without repeating
token-validation logic.

**As a** backend service
**I want** stateless JWT middleware (`get_current_user`, `require_role`) in `core/security.py`
**So that** every protected endpoint can enforce authentication and role-based access without duplicating validation logic

---

## Acceptance Criteria

1. **AC1** — Requests without an `Authorization: Bearer <token>` header return 401.
2. **AC2** — Expired tokens return 401 with `error="unauthorized"` and `details.error_code="token_expired"`.
3. **AC3** — Tampered/invalid tokens return 401 with `error="unauthorized"` and `details.error_code="token_invalid"`.
4. **AC4** — A valid token injects a `CurrentUser(user_id, company_id, role)` dataclass into the route (no database query for Story 2.4).
5. **AC5** — `require_role("bid_manager")` allows admin and bid_manager; rejects contributor, reviewer, read_only with 403.
6. **AC6** — Role hierarchy is enforced as: admin(5) > bid_manager(4) > contributor(3) > reviewer(2) > read_only(1).

---

## Preflight Verification

| Check | Status | Notes |
|-------|--------|-------|
| Story status | ✅ `ready-for-dev` | |
| Stack detected | ✅ `backend` | Python/FastAPI, `pyproject.toml` only |
| Test framework | ✅ `pytest + pytest-asyncio` | `pyproject.toml` dev deps |
| `conftest.py` | ✅ exists | `test_client`, `rsa_test_key_pair`, `rsa_env_setup` (autouse) available |
| Epic test design | ✅ loaded | `test-design-epic-02.md` — E02-P0-003/004/005/006 |
| Existing `security.py` | ✅ present | Has `create_access_token`, `get_rsa_public_key` from Story 2.3 |
| Missing symbols | 🔴 expected | `CurrentUser`, `ROLE_HIERARCHY`, `get_current_user`, `require_role` not yet in `security.py` |
| Missing endpoint | 🔴 expected | `GET /api/v1/auth/me` not yet in `api/v1/auth.py` |

---

## Generation Mode

**Mode selected:** AI generation (backend stack — no browser recording needed)

---

## Test Strategy

| AC | Epic Test ID | Test Level | Test Class | Failure Reason (RED Phase) |
|----|-------------|------------|------------|---------------------------|
| AC1 | E02-P0-005 | API/Integration | `TestAC1MissingAuthHeader` | `GET /me` returns 404 (endpoint not yet registered) |
| AC2 | E02-P0-003 | API/Integration | `TestAC2ExpiredToken` | `GET /me` returns 404 |
| AC3 | E02-P0-004 | API/Integration | `TestAC3TamperedToken` | `GET /me` returns 404 |
| AC4 | — | API/Integration | `TestAC4ValidToken` | `GET /me` returns 404 |
| AC5/AC6 | E02-P0-006 | Unit | `TestRoleHierarchy` | `ImportError`: `ROLE_HIERARCHY` not in `security.py` |
| AC5/AC6 | E02-P0-006 | Unit | `TestCurrentUser` | `ImportError`: `CurrentUser` not in `security.py` |
| AC5/AC6 | E02-P0-006 | Unit | `TestRequireRole` | `ImportError`: `ROLE_HIERARCHY`, `require_role` not in `security.py` |

**No E2E tests** — pure backend story; no browser interaction required.

---

## Failing Tests Created (🔴 RED Phase)

### Unit Tests — 15 tests

**File:** `eusolicit-app/services/client-api/tests/unit/test_security.py`

**RED Phase Mechanism:** `pytestmark = pytest.mark.xfail(strict=False, ...)` + lazy imports inside each test method. Tests will produce `XFAIL` (ImportError) when run before implementation.

#### TestRoleHierarchy (5 tests — AC6)

- ✅ **`test_all_five_roles_present`**
  - **Status:** 🔴 RED — `ImportError: cannot import name 'ROLE_HIERARCHY' from 'client_api.core.security'`
  - **Verifies:** AC6 — `ROLE_HIERARCHY` dict contains exactly the 5 CompanyRole values

- ✅ **`test_rank_values_match_specification`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC6 — Each role maps to the exact rank specified: admin=5, bid_manager=4, contributor=3, reviewer=2, read_only=1

- ✅ **`test_admin_has_highest_rank`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC6 — `admin` outranks all other roles

- ✅ **`test_order_is_strictly_ascending`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC6 — No ties or reversals in the hierarchy ordering

- ✅ **`test_all_ranks_are_positive_integers`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC6 — All rank values are positive integers

#### TestCurrentUser (3 tests — AC4)

- ✅ **`test_current_user_is_a_dataclass`**
  - **Status:** 🔴 RED — `ImportError: cannot import name 'CurrentUser'`
  - **Verifies:** AC4 — `CurrentUser` is a `@dataclass` (not Pydantic)

- ✅ **`test_current_user_has_required_fields`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC4 — `user_id`, `company_id`, `role` fields are present and correct after instantiation

- ✅ **`test_current_user_role_is_plain_string`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC4 — `role` is a plain `str` (JWT claims are strings; no enum coercion)

#### TestRequireRole (7 tests — AC5/AC6 / E02-P0-006)

- ✅ **`test_roles_at_or_above_bid_manager_satisfy_require_bid_manager[admin]`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC5 — `admin` satisfies `require_role("bid_manager")`

- ✅ **`test_roles_at_or_above_bid_manager_satisfy_require_bid_manager[bid_manager]`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC5 — `bid_manager` satisfies `require_role("bid_manager")`

- ✅ **`test_roles_below_bid_manager_are_rejected_by_require_bid_manager[contributor]`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC5 — `contributor` is rejected by `require_role("bid_manager")`

- ✅ **`test_roles_below_bid_manager_are_rejected_by_require_bid_manager[reviewer]`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC5 — `reviewer` is rejected by `require_role("bid_manager")`

- ✅ **`test_roles_below_bid_manager_are_rejected_by_require_bid_manager[read_only]`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC5 — `read_only` is rejected by `require_role("bid_manager")`

- ✅ **`test_role_hierarchy_full_matrix[...]`** (25 parametrized combinations)
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC5/AC6 — Full 5×5 role × minimum_role matrix (exhaustive E02-P0-006 coverage)

- ✅ **`test_require_role_factory_returns_callable`**
  - **Status:** 🔴 RED — `ImportError: cannot import name 'require_role'`
  - **Verifies:** AC5 — `require_role("bid_manager")` returns a FastAPI-compatible callable dependency

- ✅ **`test_require_role_with_each_valid_minimum`**
  - **Status:** 🔴 RED — `ImportError`
  - **Verifies:** AC5 — `require_role()` accepts every role name without raising at factory time

---

### API / Integration Tests — 13 tests

**File:** `eusolicit-app/services/client-api/tests/api/test_auth_middleware.py`

**RED Phase Mechanism:** `pytestmark = pytest.mark.xfail(strict=False, ...)`. Tests fail with `AssertionError` (expected status differs from actual 404) because `GET /api/v1/auth/me` is not yet registered.

#### TestAC1MissingAuthHeader (2 tests — E02-P0-005)

- ✅ **`test_me_without_any_auth_header_returns_401`**
  - **Status:** 🔴 RED — `AssertionError: Expected 401; got 404`
  - **Verifies:** AC1 — No `Authorization` header → 401 (not 403 or 404)

- ✅ **`test_me_with_non_bearer_scheme_returns_401`**
  - **Status:** 🔴 RED — `AssertionError: Expected 401; got 404`
  - **Verifies:** AC1 — Non-Bearer scheme (e.g., `Basic ...`) → 401

#### TestAC2ExpiredToken (2 tests — E02-P0-003)

- ✅ **`test_me_with_expired_token_returns_401`**
  - **Status:** 🔴 RED — `AssertionError: Expected 401; got 404`
  - **Verifies:** AC2 — Expired JWT → 401 with `error="unauthorized"` and `details.error_code="token_expired"`

- ✅ **`test_expired_token_body_contains_correlation_id`**
  - **Status:** 🔴 RED — `AssertionError`
  - **Verifies:** AC2 — Error response includes `correlation_id` (AppException envelope)

#### TestAC3TamperedToken (4 tests — E02-P0-004)

- ✅ **`test_me_with_corrupted_signature_returns_401`**
  - **Status:** 🔴 RED — `AssertionError: Expected 401; got 404`
  - **Verifies:** AC3 — Corrupted JWT signature → 401 + `error_code="token_invalid"`

- ✅ **`test_me_with_tampered_payload_returns_401`**
  - **Status:** 🔴 RED — `AssertionError`
  - **Verifies:** AC3 — Modified payload without re-signing → 401 + `token_invalid`

- ✅ **`test_me_with_random_string_as_token_returns_401`**
  - **Status:** 🔴 RED — `AssertionError`
  - **Verifies:** AC3 — Non-JWT Bearer value → 401 + `token_invalid`

- ✅ **`test_expired_error_code_takes_priority_over_invalid`**
  - **Status:** 🔴 RED — `AssertionError`
  - **Verifies:** AC3 (ordering) — Expired token → `token_expired` not `token_invalid`; validates `ExpiredSignatureError` caught before `InvalidTokenError`

#### TestAC4ValidToken (5 tests)

- ✅ **`test_me_with_valid_admin_token_returns_200`**
  - **Status:** 🔴 RED — `AssertionError: Expected 200; got 404`
  - **Verifies:** AC4 — Valid admin JWT → 200 with `user_id`, `company_id`, `role="admin"` in body

- ✅ **`test_me_reflects_correct_role_for_each_company_role[admin/bid_manager/contributor/reviewer/read_only]`** (5 parametrized)
  - **Status:** 🔴 RED — `AssertionError`
  - **Verifies:** AC4 — Each CompanyRole round-trips correctly through JWT claims → response

- ✅ **`test_me_user_id_and_company_id_are_valid_uuids`**
  - **Status:** 🔴 RED — `AssertionError`
  - **Verifies:** AC4 — `user_id` and `company_id` in response are valid UUID strings

- ✅ **`test_me_response_does_not_expose_sensitive_jwt_internals`**
  - **Status:** 🔴 RED — `AssertionError`
  - **Verifies:** AC4 — `hashed_password`, `jti`, `exp`, `iat` absent from response body

- ✅ **`test_me_does_not_make_database_queries`**
  - **Status:** 🔴 RED — `AssertionError`
  - **Verifies:** AC4 — `get_current_user` is stateless; no DB query needed (Story 2.4 scope)

---

## Test Summary

| Category | Count | TDD Phase |
|----------|-------|-----------|
| Unit tests (`test_security.py`) | 15 | 🔴 RED |
| API/integration tests (`test_auth_middleware.py`) | 13 (+ 5 parametrized = 18 total runs) | 🔴 RED |
| **Total test definitions** | **28** | **🔴 RED** |

**All tests will produce `XFAIL` when run before implementation.**

---

## Fixtures Used (from `conftest.py`)

All fixtures are pre-existing from Story 2.3. No new fixtures required for Story 2.4.

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `rsa_test_key_pair` | session | Generates 2048-bit RSA key pair; returns `(private_pem, public_pem)` |
| `rsa_env_setup` | session, **autouse** | Injects test keys into `security._private_key_cache` / `_public_key_cache`; sets env vars |
| `test_redis_client` | session | Redis client (DB 1) for rate-limit isolation |
| `flush_rate_limit_keys` | function, **autouse** | Cleans `rate_limit:login:*` keys after each test |
| `app` | function | FastAPI app with DB rollback + Redis + StubEmailService overrides |
| `test_client` | function | `httpx.AsyncClient` backed by ASGI transport |

---

## Mock Requirements

None. Story 2.4 uses no external services. RSA key loading is handled by
`rsa_env_setup` (autouse fixture). No email, no payment gateway, no external OAuth.

---

## Implementation Checklist

### Task 1 — Extend `core/security.py`

**File:** `eusolicit-app/services/client-api/src/client_api/core/security.py`

- [ ] **1.1** Add `CurrentUser` dataclass (fields: `user_id: UUID`, `company_id: UUID`, `role: str`)
  - **Unblocks:** `TestCurrentUser.*` (3 unit tests)
- [ ] **1.2** Add `ROLE_HIERARCHY: dict[str, int]` with values `{admin:5, bid_manager:4, contributor:3, reviewer:2, read_only:1}`
  - **Unblocks:** `TestRoleHierarchy.*` (5 unit tests), `TestRequireRole.*` rank-comparison tests
- [ ] **1.3** Add `http_bearer = HTTPBearer(auto_error=False)` singleton
- [ ] **1.4** Add `get_current_user` async dependency function (validates RS256 JWT, raises `UnauthorizedError` with `details={"error_code": "token_expired"|"token_invalid"}`)
  - **Unblocks:** `TestAC1*`, `TestAC2*`, `TestAC3*`, `TestAC4*` (all API tests — pending endpoint)
- [ ] **1.5** Add `require_role(minimum_role: str) -> Callable` factory function
  - **Unblocks:** `TestRequireRole.test_require_role_factory_returns_callable`
- [ ] **1.6** Add `# TODO: S02.05 — check token revocation` comment in `get_current_user`

**Verify unit tests pass:**
```bash
cd eusolicit-app/services/client-api
python -m pytest tests/unit/test_security.py -v
```

---

### Task 2 — Add `CurrentUserResponse` schema and `GET /auth/me` endpoint

**Files:**
- `eusolicit-app/services/client-api/src/client_api/schemas/auth.py`
- `eusolicit-app/services/client-api/src/client_api/api/v1/auth.py`

- [ ] **2.1** Add `CurrentUserResponse(BaseModel)` to `schemas/auth.py` (fields: `user_id: UUID`, `company_id: UUID`, `role: str`)
- [ ] **2.2** Add `GET /me` route to `api/v1/auth.py` with `Depends(get_current_user)`
  - Response: `CurrentUserResponse` (200)
  - **Unblocks:** ALL API integration tests (13 tests)

**Verify API tests pass:**
```bash
cd eusolicit-app/services/client-api
python -m pytest tests/api/test_auth_middleware.py -v
```

---

### Task 3 — Remove xfail markers (GREEN Phase confirmation)

Once Tasks 1 & 2 are complete and all tests pass:

- [ ] Remove `pytestmark = pytest.mark.xfail(...)` from `tests/unit/test_security.py`
- [ ] Remove `pytestmark = pytest.mark.xfail(...)` from `tests/api/test_auth_middleware.py`
- [ ] Run full test suite and confirm all pass (no XPASS, no XFAIL):

```bash
cd eusolicit-app/services/client-api
python -m pytest tests/unit/test_security.py tests/api/test_auth_middleware.py -v
```

---

## Running Tests

```bash
# Run all Story 2.4 tests (RED phase — expect XFAIL)
cd eusolicit-app/services/client-api
python -m pytest tests/unit/test_security.py tests/api/test_auth_middleware.py -v

# Run unit tests only (role hierarchy + CurrentUser + require_role)
python -m pytest tests/unit/test_security.py -v -k "unit"

# Run API middleware tests only
python -m pytest tests/api/test_auth_middleware.py -v

# Run with coverage (after green phase)
python -m pytest tests/unit/test_security.py tests/api/test_auth_middleware.py \
  --cov=client_api.core.security --cov-report=term-missing

# Run by epic test ID tags
python -m pytest tests/ -m "integration" -v
python -m pytest tests/ -m "unit" -v

# Run full test suite (regression safety net)
python -m pytest tests/ -v
```

---

## Red-Green-Refactor Workflow

### 🔴 RED Phase (Complete)

**TEA Agent Responsibilities:**

- ✅ 28 failing tests written across 2 test files
- ✅ Unit tests use lazy imports + `pytestmark xfail` (no collection failure)
- ✅ API tests fail deterministically with `AssertionError` (404 ≠ expected status)
- ✅ All tests assert EXPECTED behavior, not placeholders
- ✅ All tests annotated with epic test IDs (E02-P0-003/004/005/006)
- ✅ Implementation checklist created (tasks mapped to tests)
- ✅ Existing fixtures reused (`rsa_test_key_pair`, `test_client`, `rsa_env_setup`)

**Verification command:**
```bash
python -m pytest tests/unit/test_security.py tests/api/test_auth_middleware.py -v
# Expected: all tests marked XFAIL (no FAILED, no ERROR)
```

---

### 🟢 GREEN Phase (DEV Team — Next Steps)

**DEV Agent Responsibilities:**

1. **Implement Task 1** — Extend `core/security.py` with `CurrentUser`, `ROLE_HIERARCHY`, `get_current_user`, `require_role`
2. **Run unit tests** → `python -m pytest tests/unit/test_security.py -v` → should pass
3. **Implement Task 2** — Add `CurrentUserResponse` schema + `GET /auth/me` route
4. **Run API tests** → `python -m pytest tests/api/test_auth_middleware.py -v` → should pass
5. **Remove xfail markers** from both test files (Task 3 above)
6. **Run full suite** → confirm all pass without xfail

**Key Principles:**

- One failing test at a time (start with `test_all_five_roles_present`)
- Minimal implementation (don't over-engineer)
- Follow the story's code samples in Tasks 1.1–1.5 exactly

---

### ♻️ REFACTOR Phase (After All Tests Pass)

1. Verify `ROLE_HIERARCHY` and `CurrentUser` are placed before `http_bearer` in `security.py`
2. Ensure `from __future__ import annotations` is at the top of all modified files
3. Check `structlog.get_logger()` is used for any added log statements (no `print()`)
4. Verify `# TODO: S02.05 — check token revocation` comment is present in `get_current_user`
5. Run `ruff check` and `mypy` passes
6. Ensure no circular imports: `api → schemas + core`, `core → config` only

---

## Epic Coverage Mapping

| Epic Test ID | AC | Story 2.4 Test Coverage |
|-------------|-----|-------------------------|
| E02-P0-003 | AC2 | `TestAC2ExpiredToken` (2 tests) |
| E02-P0-004 | AC3 | `TestAC3TamperedToken` (4 tests) |
| E02-P0-005 | AC1 | `TestAC1MissingAuthHeader` (2 tests) |
| E02-P0-006 | AC5/AC6 | `TestRoleHierarchy` (5) + `TestCurrentUser` (3) + `TestRequireRole` (7+25 matrix) |
| — | AC4 | `TestAC4ValidToken` (5 tests) |

**Remaining E02-P0 items not in Story 2.4 scope:**

| Epic Test ID | Story | Notes |
|-------------|-------|-------|
| E02-P0-001 | 2.2 | Registration — covered in `test_register.py` |
| E02-P0-002 | 2.3 | Login JWT issuance — covered in `test_login.py` |
| E02-P0-007/008/009 | 2.x | Entity-level RBAC — post-2.4 scope |
| E02-P0-010/011 | 2.5 | Token refresh/revocation — next story |
| E02-P0-012 | 2.x | Cross-tenant isolation — future scope |

---

## Key Risks and Assumptions

| Risk | Mitigation |
|------|-----------|
| `ExpiredSignatureError` caught after `InvalidTokenError` → wrong `error_code` | `test_expired_error_code_takes_priority_over_invalid` explicitly verifies catch order |
| `HTTPBearer(auto_error=False)` not used → 403 instead of 401 on missing header | `TestAC1MissingAuthHeader` asserts 401 specifically; failure message points to `auto_error=False` |
| `role` stored as enum type (not `str`) in `CurrentUser` → JWT deserialization fails | `test_current_user_role_is_plain_string` validates `isinstance(user.role, str)` |
| `GET /me` makes DB query (violates stateless contract) | `test_me_does_not_make_database_queries` uses rolled-back session to surface accidental DB calls |
| Circular imports from `schemas/auth.py` importing `CurrentUser` | Dev notes explicitly warn: use raw `UUID` and `str` in `CurrentUserResponse`, do NOT import `CurrentUser` |

---

## Notes

- **No new Alembic migration required** for Story 2.4 (pure code change, no schema changes).
- **`create_access_token` already works** (Story 2.3); used in `make_valid_token()` helper.
- **`rsa_env_setup` is autouse** — the test key pair is automatically injected into `security._public_key_cache` / `_private_key_cache` before any test runs.
- **`test_auth_middleware.py` uses `test_client` from `conftest.py`** — no local fixtures needed for Story 2.4 API tests.
- The `require_role` HTTP 403 behavior (AC5) is tested at the unit level via rank comparison. A dedicated integration test for 403 responses requires an endpoint that uses `Depends(require_role(...))` — this will be covered in stories that implement role-gated routes (Epic 2 post-S02.04).

---

## Contact

**Questions or Issues?** Refer to:

- Story specification: `eusolicit-docs/implementation-artifacts/2-4-jwt-authentication-middleware.md`
- Epic test design: `eusolicit-docs/test-artifacts/test-design-epic-02.md`
- Project rules: `eusolicit-docs/project-context.md`

---

**Generated by BMad TEA Agent** — 2026-04-07
