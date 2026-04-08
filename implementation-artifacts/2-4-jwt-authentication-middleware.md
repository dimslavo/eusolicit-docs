# Story 2.4: JWT Authentication Middleware

Status: review

## Story

As a backend service,
I want a `get_current_user` FastAPI dependency that validates RS256 JWT Bearer tokens and a `require_role` factory that enforces company-level role minimums,
so that all protected endpoints can be secured without repeating token-validation logic.

## Acceptance Criteria

1. **AC1** — Requests without an `Authorization: Bearer <token>` header return 401.
2. **AC2** — Expired tokens return 401 with `error="unauthorized"` and `details.error_code="token_expired"` in the JSON body.
3. **AC3** — Tampered/invalid tokens return 401 with `error="unauthorized"` and `details.error_code="token_invalid"`.
4. **AC4** — A valid token injects a `CurrentUser(user_id, company_id, role)` dataclass into the route (no database query for Story 2.4; access token claims are the sole source of truth).
5. **AC5** — `require_role("bid_manager")` allows admin and bid_manager; rejects contributor, reviewer, read_only with 403.
6. **AC6** — Role hierarchy is enforced as: admin(5) > bid_manager(4) > contributor(3) > reviewer(2) > read_only(1).

## Tasks / Subtasks

- [x] Task 1 — Extend `core/security.py`: `CurrentUser`, `ROLE_HIERARCHY`, `get_current_user`, `require_role` (AC: 1–6)
  - [x] 1.1 Add `CurrentUser` dataclass at top of `src/client_api/core/security.py`:
    ```python
    from dataclasses import dataclass
    from uuid import UUID

    @dataclass
    class CurrentUser:
        """Authenticated user context injected by get_current_user dependency."""
        user_id: UUID
        company_id: UUID
        role: str  # CompanyRole.value string
    ```
  - [x] 1.2 Add `ROLE_HIERARCHY` mapping immediately after `CurrentUser`:
    ```python
    ROLE_HIERARCHY: dict[str, int] = {
        "admin": 5,
        "bid_manager": 4,
        "contributor": 3,
        "reviewer": 2,
        "read_only": 1,
    }
    ```
  - [x] 1.3 Add `http_bearer` singleton and `get_current_user` dependency function:
    ```python
    from typing import Annotated, Callable
    from fastapi import Depends
    from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
    from jwt.exceptions import ExpiredSignatureError, InvalidTokenError
    from eusolicit_common.exceptions import ForbiddenError, UnauthorizedError

    http_bearer = HTTPBearer(auto_error=False)
    # auto_error=False: HTTPBearer returns None instead of raising 403 when
    # the header is absent; we raise 401 ourselves (correct HTTP semantics).

    async def get_current_user(
        credentials: Annotated[HTTPAuthorizationCredentials | None, Depends(http_bearer)],
    ) -> CurrentUser:
        """Extract and verify RS256 JWT; return CurrentUser from claims.

        Raises UnauthorizedError (401) for missing, expired, or tampered tokens.
        No database query — access token claims are the source of truth for 2.4.
        """
        if credentials is None:
            raise UnauthorizedError("Missing or invalid Authorization header")
        token = credentials.credentials
        try:
            payload = jwt.decode(token, get_rsa_public_key(), algorithms=["RS256"])
        except ExpiredSignatureError:
            raise UnauthorizedError(
                "Access token has expired",
                details={"error_code": "token_expired"},
            )
        except InvalidTokenError:
            raise UnauthorizedError(
                "Access token is invalid",
                details={"error_code": "token_invalid"},
            )
        return CurrentUser(
            user_id=UUID(payload["sub"]),
            company_id=UUID(payload["company_id"]),
            role=payload["role"],
        )
    ```
  - [x] 1.4 Add `require_role` factory function:
    ```python
    def require_role(minimum_role: str) -> Callable[..., CurrentUser]:
        """Return a FastAPI dependency that requires the user to hold minimum_role or higher.

        Usage: ``Depends(require_role("bid_manager"))``

        Raises ForbiddenError (403) if the user's role ranks below minimum_role.
        """
        async def _check_role(
            current_user: Annotated[CurrentUser, Depends(get_current_user)],
        ) -> CurrentUser:
            user_rank = ROLE_HIERARCHY.get(current_user.role, 0)
            required_rank = ROLE_HIERARCHY.get(minimum_role, 0)
            if user_rank < required_rank:
                raise ForbiddenError(
                    f"Role '{current_user.role}' does not meet minimum required role '{minimum_role}'"
                )
            return current_user
        return _check_role
    ```

- [x] Task 2 — Add `CurrentUserResponse` schema and `GET /auth/me` endpoint (AC: 4)
  - [x] 2.1 Add to `src/client_api/schemas/auth.py`:
    ```python
    from client_api.core.security import CurrentUser  # avoid circular: import at schema level only

    class CurrentUserResponse(BaseModel):
        """Response body for GET /api/v1/auth/me."""
        model_config = ConfigDict(from_attributes=True)

        user_id: UUID
        company_id: UUID
        role: str
    ```
    Note: Import `CurrentUser` lazily (inside function) or use forward-ref to avoid circular imports if needed. Alternatively, just use `UUID` and `str` directly — `CurrentUserResponse` does not depend on `CurrentUser` type directly.
  - [x] 2.2 Update `src/client_api/api/v1/auth.py` — add `/me` route:
    ```python
    from client_api.core.security import CurrentUser, get_current_user
    from client_api.schemas.auth import CurrentUserResponse

    @router.get("/me", status_code=200, response_model=CurrentUserResponse)
    async def get_me(
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
    ) -> CurrentUserResponse:
        """Return the authenticated user's identity from the JWT claims.

        Requires a valid Bearer token. Returns 401 for missing/invalid/expired tokens.
        No database query; reflects access token claims directly.
        """
        return CurrentUserResponse(
            user_id=current_user.user_id,
            company_id=current_user.company_id,
            role=current_user.role,
        )
    ```

- [x] Task 3 — Unit tests for role hierarchy and dependency logic (AC: 5, 6; E02-P0-006)
  - [x] 3.1 Create `tests/unit/test_security.py`:
    ```python
    """Unit tests: Story 2.4 — JWT Authentication Middleware

    Covers:
      E02-P0-006 — require_role enforces company role hierarchy
    """
    import pytest
    from client_api.core.security import ROLE_HIERARCHY, require_role

    ROLES_IN_ORDER = ["read_only", "reviewer", "contributor", "bid_manager", "admin"]

    class TestRoleHierarchy:
        def test_all_five_roles_present(self):
            assert set(ROLE_HIERARCHY) == {"admin", "bid_manager", "contributor", "reviewer", "read_only"}

        def test_admin_has_highest_rank(self):
            assert ROLE_HIERARCHY["admin"] > ROLE_HIERARCHY["bid_manager"]

        def test_order_is_strictly_ascending(self):
            ranks = [ROLE_HIERARCHY[r] for r in ROLES_IN_ORDER]
            assert ranks == sorted(ranks), "ROLE_HIERARCHY must be strictly ascending"

    class TestRequireRole:
        """E02-P0-006: Parametrized role hierarchy enforcement."""

        @pytest.mark.parametrize("user_role", ["admin", "bid_manager"])
        async def test_require_bid_manager_allows_admin_and_bid_manager(self, user_role: str):
            from dataclasses import dataclass
            from uuid import uuid4
            from client_api.core.security import CurrentUser, ROLE_HIERARCHY
            from eusolicit_common.exceptions import ForbiddenError

            user = CurrentUser(user_id=uuid4(), company_id=uuid4(), role=user_role)
            user_rank = ROLE_HIERARCHY.get(user.role, 0)
            required_rank = ROLE_HIERARCHY.get("bid_manager", 0)
            assert user_rank >= required_rank  # should pass

        @pytest.mark.parametrize("user_role", ["contributor", "reviewer", "read_only"])
        async def test_require_bid_manager_rejects_lower_roles(self, user_role: str):
            from client_api.core.security import ROLE_HIERARCHY

            user_rank = ROLE_HIERARCHY.get(user_role, 0)
            required_rank = ROLE_HIERARCHY.get("bid_manager", 0)
            assert user_rank < required_rank  # confirms rejection condition
    ```

- [x] Task 4 — API tests for middleware (AC: 1–5; E02-P0-003, E02-P0-004, E02-P0-005)
  - [x] 4.1 Create `tests/api/test_auth_middleware.py`:

    ```python
    """
    API tests: Story 2.4 — JWT Authentication Middleware

    Tests GET /api/v1/auth/me as the protected endpoint under test.

    Epic test coverage:
      E02-P0-003 — Expired JWT rejected with 401 + token_expired
      E02-P0-004 — Tampered JWT rejected with 401 + token_invalid
      E02-P0-005 — Requests without Authorization header rejected with 401
    """
    import pytest
    import pytest_asyncio
    from datetime import UTC, datetime, timedelta
    from uuid import uuid4
    import jwt  # PyJWT

    ME_URL = "/api/v1/auth/me"

    def make_valid_token(private_key: bytes, role: str = "admin") -> str:
        """Create a valid RS256 JWT with specified role (15-min expiry)."""
        from client_api.core.security import create_access_token
        return create_access_token(
            user_id=uuid4(),
            company_id=uuid4(),
            role=role,
        )

    def make_expired_token(private_key: bytes) -> str:
        """Create an RS256 JWT with exp in the past."""
        from datetime import UTC, datetime, timedelta
        from uuid import uuid4
        now = datetime.now(UTC)
        payload = {
            "sub": str(uuid4()),
            "company_id": str(uuid4()),
            "role": "admin",
            "iat": int((now - timedelta(hours=1)).timestamp()),
            "exp": int((now - timedelta(minutes=1)).timestamp()),
            "jti": str(uuid4()),
        }
        return jwt.encode(payload, private_key, algorithm="RS256")

    def make_tampered_token(original_token: str) -> str:
        """Corrupt the signature of a valid JWT."""
        header, payload, sig = original_token.split(".")
        return f"{header}.{payload}.invalidsignatureXXX"

    class TestAC1MissingAuthHeader:
        """E02-P0-005: No Authorization header → 401."""

        async def test_me_without_auth_header_returns_401(self, test_client):
            resp = await test_client.get(ME_URL)
            assert resp.status_code == 401

    class TestAC2ExpiredToken:
        """E02-P0-003: Expired token → 401 with error_code=token_expired."""

        async def test_me_with_expired_token_returns_401(self, test_client, rsa_test_key_pair):
            private_pem, _ = rsa_test_key_pair
            token = make_expired_token(private_pem.encode())
            resp = await test_client.get(ME_URL, headers={"Authorization": f"Bearer {token}"})
            assert resp.status_code == 401
            body = resp.json()
            assert body["error"] == "unauthorized"
            assert body["details"]["error_code"] == "token_expired"

    class TestAC3TamperedToken:
        """E02-P0-004: Tampered token → 401 with error_code=token_invalid."""

        async def test_me_with_tampered_token_returns_401(self, test_client, rsa_test_key_pair):
            private_pem, _ = rsa_test_key_pair
            valid_token = make_valid_token(private_pem.encode())
            tampered = make_tampered_token(valid_token)
            resp = await test_client.get(ME_URL, headers={"Authorization": f"Bearer {tampered}"})
            assert resp.status_code == 401
            body = resp.json()
            assert body["error"] == "unauthorized"
            assert body["details"]["error_code"] == "token_invalid"

    class TestAC4ValidToken:
        """Valid token injects CurrentUser; GET /me returns correct claims."""

        async def test_me_with_valid_token_returns_200_with_user_context(
            self, test_client, rsa_test_key_pair
        ):
            private_pem, _ = rsa_test_key_pair
            token = make_valid_token(private_pem.encode(), role="admin")
            resp = await test_client.get(ME_URL, headers={"Authorization": f"Bearer {token}"})
            assert resp.status_code == 200
            body = resp.json()
            assert "user_id" in body
            assert "company_id" in body
            assert body["role"] == "admin"
    ```

### Review Follow-ups (AI)

- [x] [AI-Review][Patch] Fix malformed JWT claims → 500: wrap `CurrentUser` construction in `try/except (KeyError, ValueError, TypeError)` in `security.py:get_current_user` to raise `UnauthorizedError` with `error_code="token_invalid"` instead of 500 (security.py:176-180)
- [x] [AI-Review][Patch] Fix `require_role` unknown role silent bypass: add `if minimum_role not in ROLE_HIERARCHY: raise ValueError(...)` validation at the top of `require_role` factory (security.py:188)
- [x] [AI-Review][Patch] Add unit tests that actually invoke `require_role` async `_check_role` function to verify `ForbiddenError` is raised for insufficient roles and authorized roles pass through (test_security.py)
- [x] [AI-Review][Patch] Add API-level test for `require_role` 403 rejection: add a test-only route using `Depends(require_role("bid_manager"))` in conftest, verify 403 for contributor/reviewer/read_only and 200 for admin/bid_manager (test_auth_middleware.py)

## Dev Notes

### Architecture Constraints (MUST FOLLOW)

- **Extend `core/security.py`, NOT a new file.** `CurrentUser`, `ROLE_HIERARCHY`, `get_current_user`, and `require_role` all go into the **existing** `src/client_api/core/security.py`. Story 2.3 already created this module with RSA key loaders and `create_access_token`. Do NOT create a separate `auth.py` or `middleware.py`.
- **No DB query in `get_current_user`.** Story 2.4 is stateless JWT verification only. Token revocation (checking `refresh_tokens.is_revoked`) is Story 2.5's scope. Add a `# TODO: S02.05 — check token revocation` comment for clarity.
- **PyJWT only.** Use `import jwt` (pyjwt>=2.8). Never use python-jose. Verify with `algorithms=["RS256"]` exactly (list, not string).
- **`jwt.decode` raises specific exceptions.** Catch `jwt.exceptions.ExpiredSignatureError` before `jwt.exceptions.InvalidTokenError` (the latter is the base class that covers all invalid-token scenarios including expired; but catching expired first produces the right `error_code`):
  ```python
  from jwt.exceptions import ExpiredSignatureError, InvalidTokenError
  try:
      payload = jwt.decode(token, get_rsa_public_key(), algorithms=["RS256"])
  except ExpiredSignatureError:
      raise UnauthorizedError("...", details={"error_code": "token_expired"})
  except InvalidTokenError:
      raise UnauthorizedError("...", details={"error_code": "token_invalid"})
  ```
- **`HTTPBearer(auto_error=False)`.** Use `auto_error=False` so a missing header returns `None` credentials (we raise 401) rather than FastAPI's default 403. This ensures AC1 produces 401, not 403.
- **`UnauthorizedError` / `ForbiddenError` from `eusolicit_common.exceptions`.** Already imported patterns exist in `auth_service.py`. Same import path for security module.
- **Error code in `details`.** The test design (E02-P0-003/004) asserts `body.details.error_code`. The `AppException` error response envelope is `{"error": ..., "message": ..., "details": ..., "correlation_id": ...}`. Pass `details={"error_code": "token_expired"}` to `UnauthorizedError()`.
- **`CompanyRole` enum.** Already defined in `src/client_api/models/enums.py` as `CompanyRole(str, Enum)` with values: admin, bid_manager, contributor, reviewer, read_only. The `role` field in `CurrentUser` is a plain `str` (JWT claims are strings). `require_role` also receives a string. No enum coercion needed.
- **Logging.** Use `structlog.get_logger()` if adding log statements. No `print()` or `logging.getLogger()`. [Source: eusolicit-docs/project-context.md#rule 9]
- **Python 3.12+ syntax.** `from __future__ import annotations` at top of all new/modified files.

### PyJWT Decode Pattern (EXACT)

```python
# Signing (already in security.py, Story 2.3):
jwt.encode(payload, get_rsa_private_key(), algorithm="RS256")  # returns str in PyJWT >= 2.0

# Decoding (Story 2.4 new function):
payload = jwt.decode(token, get_rsa_public_key(), algorithms=["RS256"])
# payload["sub"]        → user UUID string
# payload["company_id"] → company UUID string
# payload["role"]       → role string (e.g. "admin")
# payload["exp"], payload["iat"], payload["jti"] also present
```

`jwt.decode()` automatically validates `exp`. No need to manually check expiry.

### `get_rsa_public_key()` is already cached

`core/security.py` already has `_public_key_cache: bytes | None = None` and `get_rsa_public_key()` which caches in module-level variable on first call. Story 2.4 reuses this without change. The epic note "Cache the public key in memory at startup" is already satisfied by Story 2.3.

### File Structure (New and Updated Files)

```
services/client-api/
  src/client_api/
    core/
      security.py        ← UPDATE: add CurrentUser, ROLE_HIERARCHY, http_bearer,
                                   get_current_user, require_role
                                   (RSA key loaders already present — DO NOT remove)
    schemas/
      auth.py            ← UPDATE: add CurrentUserResponse
    api/
      v1/
        auth.py          ← UPDATE: add GET /auth/me route using get_current_user
  tests/
    unit/
      test_security.py   ← NEW: unit tests for ROLE_HIERARCHY and require_role logic
    api/
      test_auth_middleware.py  ← NEW: API tests for get_current_user (E02-P0-003/004/005)
```

**No new Alembic migration needed.** This story adds no new database tables.

### Circular Import Prevention

`core/security.py` imports from `client_api.config`. `schemas/auth.py` will import `UUID` from stdlib. Avoid importing `CurrentUser` (a core type) into schemas — instead use raw `UUID` and `str` fields in `CurrentUserResponse`. This keeps the dependency direction: `api → schemas + core`, `core → config`. Do not import from `api` inside `core`.

### `GET /auth/me` Response Format

```json
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "company_id": "660e8400-e29b-41d4-a716-446655440001",
  "role": "admin"
}
```

### Role Hierarchy for `require_role`

```
admin(5) > bid_manager(4) > contributor(3) > reviewer(2) > read_only(1)
```

`require_role("bid_manager")` means rank ≥ 4:
- ✅ admin (5), bid_manager (4)
- ❌ contributor (3), reviewer (2), read_only (1)

### Test Fixture: `rsa_test_key_pair` and `rsa_env_setup`

These fixtures already exist in `tests/conftest.py` from Story 2.3:
- `rsa_test_key_pair` — session-scoped, generates 2048-bit RSA pair; returns `(private_pem, public_pem)`
- `rsa_env_setup` — session-scoped autouse; sets env vars and injects into `_private_key_cache` / `_public_key_cache`
- `test_client` — httpx AsyncClient backed by FastAPI ASGI transport
- `app` — FastAPI app with DB rollback + Redis + email overrides

New tests in `test_auth_middleware.py` can request `test_client` and `rsa_test_key_pair` directly. `rsa_env_setup` is `autouse=True` so key caches are pre-populated.

### Previous Story Intelligence (Story 2.3)

- `core/security.py` already contains: `_private_key_cache`, `_public_key_cache`, `get_rsa_private_key()`, `get_rsa_public_key()`, `create_access_token()`, `create_refresh_token()`.
- `dependencies.py` already has: `get_db_session`, `get_redis_client`, `get_login_rate_limiter`.
- `main.py` mounts `APIRouter(prefix="/api/v1")` with `auth_v1.router` (prefix="/auth").
- JWT claims in access tokens: `sub` (user UUID str), `company_id` (str), `role` (str), `iat`, `exp`, `jti`.
- `UnauthorizedError("...", details={...})` and `ForbiddenError("...")` pattern established in `auth_service.py`.
- `conftest.py`: `rsa_env_setup` autouse populates key caches; `test_client` fixture is async httpx client.
- Test pattern: `make_register_payload()`, `make_login_payload()` helpers for unique data per test.

### Epic Test IDs Addressed

| AC | Test ID | Description |
|----|---------|-------------|
| AC1 | E02-P0-005 | No auth header → 401 |
| AC2 | E02-P0-003 | Expired token → 401 + token_expired |
| AC3 | E02-P0-004 | Tampered token → 401 + token_invalid |
| AC4 | — | Valid token → 200 on /me with correct claims |
| AC5/AC6 | E02-P0-006 | require_role hierarchy enforcement (unit tests) |

### Project Structure Notes

- All code lives in `services/client-api/` under `eusolicit-app/`.
- `core/security.py` path: `eusolicit-app/services/client-api/src/client_api/core/security.py`
- Module import: `from client_api.core.security import CurrentUser, get_current_user, require_role`
- Test files location: `eusolicit-app/services/client-api/tests/`
- Markers: `@pytest.mark.unit` for `test_security.py`, `@pytest.mark.integration` for `test_auth_middleware.py`

### References

- Epic 2 story definition: [Source: eusolicit-docs/planning-artifacts/epic-02-authentication-identity.md#S02.04]
- Test design: [Source: eusolicit-docs/test-artifacts/test-design-epic-02.md#P0] (E02-P0-003, 004, 005, 006)
- Existing security module: [Source: eusolicit-app/services/client-api/src/client_api/core/security.py]
- Exception format: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/exceptions.py]
- Enums: [Source: eusolicit-app/services/client-api/src/client_api/models/enums.py]
- conftest fixtures: [Source: eusolicit-app/services/client-api/tests/conftest.py]
- Project rules: [Source: eusolicit-docs/project-context.md]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (Claude Code)

### Debug Log References

No issues encountered. All tests passed green on first implementation pass.

**Review follow-up debug note:** PEP 563 (`from __future__ import annotations`) requires module-level imports for types used in FastAPI route annotations defined inside pytest async fixtures. `FastAPI.get_type_hints()` resolves annotations from `func.__globals__` (the module namespace), not the local fixture scope. Adding `CurrentUser`, `require_role`, `Depends`, `FastAPI` to module-level imports in `test_auth_middleware.py` resolved a 422 UnprocessableEntity that would otherwise appear for all requests to the test route.

### Completion Notes List

- Implemented all AC1–AC6 as specified without deviations.
- `CurrentUser`, `ROLE_HIERARCHY`, `http_bearer`, `get_current_user`, `require_role` added to existing `core/security.py` (no new files).
- `CurrentUserResponse` added to `schemas/auth.py`; imports raw `UUID`/`str` (no circular import).
- `GET /auth/me` route added to `api/v1/auth.py` using `Depends(get_current_user)`.
- ATDD xfail markers removed from both test files (GREEN phase confirmed).
- ✅ Resolved review finding [Patch]: Wrapped `CurrentUser` construction in `try/except (KeyError, ValueError, TypeError)` in `get_current_user` — malformed claims now return 401 with `error_code="token_invalid"` instead of 500.
- ✅ Resolved review finding [Patch]: Added `minimum_role not in ROLE_HIERARCHY` guard to `require_role` factory — unknown role strings raise `ValueError` at startup time, preventing silent privilege escalation.
- ✅ Resolved review finding [Patch]: Added `TestRequireRoleActualBehavior` class to `test_security.py` with 8 tests that actually invoke the async `_check_role` function and assert `ForbiddenError` is raised for insufficient roles.
- ✅ Resolved review finding [Patch]: Added `TestAC5RequireRole403` class and `require_role_client` fixture to `test_auth_middleware.py` with 6 API-level tests verifying 403/200 responses via a standalone test app using `Depends(require_role("bid_manager"))`.
- 137 tests passed total (49 unit + 23 API middleware + 65 pre-existing) — zero regressions.

### File List

- `eusolicit-app/services/client-api/src/client_api/core/security.py` — UPDATED (review: malformed claims fix + require_role validation)
- `eusolicit-app/services/client-api/src/client_api/schemas/auth.py` — UPDATED
- `eusolicit-app/services/client-api/src/client_api/api/v1/auth.py` — UPDATED
- `eusolicit-app/services/client-api/tests/unit/test_security.py` — UPDATED (review: added TestRequireRoleActualBehavior + test_require_role_raises_value_error_for_unknown_minimum_role)
- `eusolicit-app/services/client-api/tests/api/test_auth_middleware.py` — UPDATED (review: added TestAC5RequireRole403 + require_role_client fixture)

## Change Log

- **2026-04-07 — Initial implementation (Story 2.4 green phase):** Added `CurrentUser`, `ROLE_HIERARCHY`, `http_bearer`, `get_current_user`, `require_role` to `core/security.py`; added `CurrentUserResponse` to `schemas/auth.py`; added `GET /auth/me` route; created unit tests and API middleware tests. 79 tests green.
- **2026-04-07 — Addressed code review findings — 4 items resolved:** (1) Wrapped `CurrentUser` construction to catch malformed claims (→ 401 not 500). (2) Added `require_role` factory-time validation for unknown roles (→ `ValueError` at startup). (3) Added `TestRequireRoleActualBehavior` unit tests that actually invoke `_check_role`. (4) Added `TestAC5RequireRole403` API tests with standalone test route. 137 tests green.

## Senior Developer Review

**Review Date:** 2026-04-07
**Reviewer:** Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** Changes Requested

### Review Findings

- [x] [Review][Patch] **Unhandled exceptions from malformed JWT claims → 500 instead of 401** — `get_current_user` at `security.py:176-180` uses direct dict access (`payload["sub"]`, `payload["company_id"]`, `payload["role"]`) and `UUID()` construction after `jwt.decode()` succeeds. A validly-signed JWT missing any of these keys raises `KeyError`; non-UUID strings raise `ValueError`; `null`/non-string types raise `TypeError`/`AttributeError`. None of these are subclasses of `InvalidTokenError`, so they escape both `except` clauses and surface as unhandled 500 Internal Server Errors. **Fix:** Wrap the `CurrentUser` construction in `try/except (KeyError, ValueError, TypeError) as exc: raise UnauthorizedError("Access token is missing required claims", details={"error_code": "token_invalid"}) from exc`. [security.py:176-180]

- [x] [Review][Patch] **`require_role` silently disables gate for unknown `minimum_role` strings** — `ROLE_HIERARCHY.get(minimum_role, 0)` at `security.py:200` defaults to rank `0` for any unrecognized role string. A developer typo like `Depends(require_role("bidmanager"))` would set required_rank=0, granting access to every authenticated user including `read_only`. This is a silent privilege escalation. **Fix:** Add validation at the top of `require_role`: `if minimum_role not in ROLE_HIERARCHY: raise ValueError(f"Unknown minimum_role {minimum_role!r}; must be one of {set(ROLE_HIERARCHY)}")`. This fails fast at import/startup time. [security.py:188-200]

- [x] [Review][Patch] **Unit tests verify `ROLE_HIERARCHY` data, not `require_role` behavior (AC5)** — All `TestRequireRole` tests in `test_security.py` manually compare `ROLE_HIERARCHY.get()` values. None invoke the actual `_check_role` async function returned by `require_role()`. If the comparison operator were changed to `>` instead of `<`, or `ForbiddenError` were never raised, all tests would still pass. **Fix:** Add unit tests that call `await require_role("admin")(current_user=CurrentUser(..., role="read_only"))` and assert `ForbiddenError` is raised, plus a pass case. [test_security.py:144-253]

- [x] [Review][Patch] **No API-level test for `require_role` 403 rejection (AC5 / E02-P0-006)** — `test_auth_middleware.py` only tests `/auth/me` which uses `get_current_user` alone — no `require_role` dependency. The 403 Forbidden path is completely untested at the HTTP level. While no route currently uses `require_role`, AC5 explicitly requires that `require_role("bid_manager")` rejects lower roles with 403. **Fix:** Add a test-only route in conftest (or test file) that uses `Depends(require_role("bid_manager"))` and verify 403 for `contributor`/`reviewer`/`read_only` tokens and 200 for `admin`/`bid_manager`. [test_auth_middleware.py]

- [x] [Review][Defer] **No `aud`/`iss` claim verification on JWT decode** — `jwt.decode()` at `security.py:165` validates only signature and expiration. No `audience` or `issuer` check. A JWT from another service sharing the same RSA key pair would be silently accepted. Deferred: pre-existing architectural gap; Story 2.4 spec does not mention audience/issuer. Track for hardening sprint.

- [x] [Review][Defer] **Token revocation not implemented (Story 2.5 scope)** — Revoked/logged-out tokens remain valid for their full 15-minute lifetime. Noted TODO at `security.py:159`. Deferred: explicitly scoped to Story 2.5.

- [x] [Review][Defer] **Empty-string role accepted through middleware chain** — A JWT with `role=""` constructs a valid `CurrentUser`. The empty role gets rank 0 via `ROLE_HIERARCHY.get("", 0)` and is denied by all `require_role` checks (fails closed), but `GET /me` would echo back the empty role. Low risk. Deferred: will be addressed when claim validation is hardened (Finding 1 fix).

- [x] [Review][Defer] **No structlog logger in `security.py` for auth events** — Security-sensitive operations (token validation failures, role rejections) produce no structured log entries. The exceptions module imports structlog; security.py does not. Deferred: observability improvement, not a functional bug.
