# Story 2.5: Token Refresh & Revocation

Status: done

## Story

As a client application,
I want `POST /api/v1/auth/refresh` that rotates my refresh token and `POST /api/v1/auth/logout` that revokes it,
so that sessions can be extended securely and terminated on demand, with breach detection that invalidates all sessions when a stolen token is replayed.

## Acceptance Criteria

1. **AC1** — `POST /auth/refresh` with a valid refresh token returns a new `{ access_token, refresh_token, token_type, expires_in }` pair; both tokens differ from the original.
2. **AC2** — The consumed refresh token is marked `is_revoked=true` in the DB immediately after rotation and cannot be reused (returns 401).
3. **AC3** — If a revoked (already-consumed) refresh token is presented, ALL tokens sharing the same `family_id` are bulk-revoked and a `shared.audit_log` entry is inserted with `action_type="auth.token_breach"`.
4. **AC4** — A refresh token past its `expires_at` timestamp returns 401.
5. **AC5** — `POST /auth/logout` with a valid refresh token sets `is_revoked=true`; a subsequent `/auth/refresh` call with that token returns 401.
6. **AC6** — The breach detection audit log entry includes `user_id`, `entity_type="refresh_token_family"`, and `entity_id=<family_id>`.

## Tasks / Subtasks

- [x] Task 1 — Add `RefreshRequest` and `LogoutRequest` schemas to `schemas/auth.py` (AC: 1, 5)
  - [x] 1.1 Add `RefreshRequest(BaseModel)` with field `refresh_token: str`
  - [x] 1.2 Add `LogoutRequest(BaseModel)` with field `refresh_token: str`
  - [x] 1.3 `LoginResponse` is reused as the response type for `/refresh` — no new response schema needed

- [x] Task 2 — Implement `refresh_token()` in `auth_service.py` (AC: 1–4, 6)
  - [x] 2.1 Hash the raw token with SHA-256 (`hashlib` already imported)
  - [x] 2.2 Look up `RefreshToken` in DB by `token_hash`; if not found → `UnauthorizedError("Invalid refresh token")`
  - [x] 2.3 If `token_record.is_revoked` → bulk-revoke entire family via `sqlalchemy.update()`, insert `AuditLog` breach entry, then raise `UnauthorizedError`
  - [x] 2.4 If `token_record.expires_at < datetime.now(UTC)` → `UnauthorizedError("Refresh token has expired")`
  - [x] 2.5 Mark consumed token `is_revoked=True`; `await session.flush()`
  - [x] 2.6 Query `User` + `CompanyMembership` via `user_id` JOIN (same pattern as `login()`)
  - [x] 2.7 Issue new access token via `create_access_token()`
  - [x] 2.8 Issue new refresh token via `create_refresh_token()`; store new `RefreshToken` row with **same `family_id`** as consumed token
  - [x] 2.9 Return `LoginResponse(access_token=..., refresh_token=raw_token)`

- [x] Task 3 — Implement `logout()` in `auth_service.py` (AC: 5)
  - [x] 3.1 Hash raw token; look up in DB
  - [x] 3.2 If not found or already `is_revoked` → return silently (idempotent — no error)
  - [x] 3.3 Set `is_revoked=True`; `await session.flush()`

- [x] Task 4 — Add `/refresh` and `/logout` routes to `api/v1/auth.py` (AC: 1–6)
  - [x] 4.1 `POST /auth/refresh` — calls `auth_service.refresh_token(request.refresh_token, session)`, returns `LoginResponse`, status 200
  - [x] 4.2 `POST /auth/logout` — calls `auth_service.logout(request.refresh_token, session)`, returns `None`, status 204

- [x] Task 5 — API tests `tests/api/test_auth_refresh.py` (AC: 1–6; E02-P0-010, E02-P0-011, E02-P1-006, E02-P1-007)
  - [x] 5.1 Define `refresh_client_and_session` fixture (shared session pattern — see Dev Notes)
  - [x] 5.2 `TestAC1ValidRefresh` — valid token returns new pair; new tokens differ from originals (E02-P0-010 partial)
  - [x] 5.3 `TestAC2RotationRevocation` — consumed token rejected with 401 on reuse (E02-P0-010)
  - [x] 5.4 `TestAC3BreachDetection` — revoked token triggers family revocation + `shared.audit_log` entry with `action_type="auth.token_breach"` (E02-P0-011)
  - [x] 5.5 `TestAC4ExpiredToken` — token past `expires_at` returns 401 (E02-P1-007)
  - [x] 5.6 `TestAC5Logout` — logout revokes token; subsequent `/refresh` returns 401 (E02-P1-006)

## Dev Notes

### Architecture Constraints (MUST FOLLOW)

**1. Extend existing files only.** No new modules:
- `src/client_api/schemas/auth.py` — add `RefreshRequest`, `LogoutRequest`
- `src/client_api/services/auth_service.py` — add `refresh_token()`, `logout()`
- `src/client_api/api/v1/auth.py` — add routes

**2. Check `is_revoked` BEFORE `expires_at`.** A token can be both revoked AND expired; breach detection must fire regardless of expiry:
```python
if token_record.is_revoked:
    # breach path — bulk revoke + audit
if token_record.expires_at.replace(tzinfo=UTC) < datetime.now(UTC):
    # expired path
```
The `.replace(tzinfo=UTC)` guard handles the rare case of a timezone-naive datetime from a test fixture; in production asyncpg returns timezone-aware datetimes.

**3. Hash-first DB lookup.** `hashlib` is already imported in `auth_service.py`:
```python
token_hash = hashlib.sha256(refresh_token.encode()).hexdigest()
stmt = select(RefreshToken).where(RefreshToken.token_hash == token_hash)
result = await session.execute(stmt)
token_record = result.scalar_one_or_none()
```

**4. Bulk family revocation via `sqlalchemy.update()`.** Do NOT load all family tokens into memory:
```python
from sqlalchemy import select, update   # add `update` to existing import

await session.execute(
    update(RefreshToken)
    .where(RefreshToken.family_id == token_record.family_id)
    .values(is_revoked=True)
)
```

**5. Audit log for breach detection — synchronous, within session.** Story 2.11 implements the full audit middleware; for Story 2.5 ONLY the breach detection case writes to `shared.audit_log`. Write synchronously (not BackgroundTasks) — it is a critical security event that must not be lost:
```python
from client_api.models import AuditLog  # already in models/__init__.py

audit_entry = AuditLog(
    user_id=token_record.user_id,
    action_type="auth.token_breach",
    entity_type="refresh_token_family",
    entity_id=token_record.family_id,
    after={"family_id": str(token_record.family_id)},
)
session.add(audit_entry)
await session.flush()   # flush before raising UnauthorizedError
```

**6. Refresh token rotation preserves `family_id`.** New token issued on rotation MUST use the same `family_id`:
```python
raw_token, new_hash = create_refresh_token()
new_rt = RefreshToken(
    user_id=token_record.user_id,
    token_hash=new_hash,
    family_id=token_record.family_id,   # ← same family
    expires_at=datetime.now(UTC) + timedelta(days=7),
)
session.add(new_rt)
await session.flush()
```

**7. `LoginResponse` is reused for `/refresh` response.** No new Pydantic schema needed:
```python
return LoginResponse(access_token=access_token, refresh_token=raw_token)
```

**8. Logout is idempotent.** Unknown or already-revoked token → silent return (no error):
```python
async def logout(refresh_token: str, session: AsyncSession) -> None:
    token_hash = hashlib.sha256(refresh_token.encode()).hexdigest()
    result = await session.execute(
        select(RefreshToken).where(RefreshToken.token_hash == token_hash)
    )
    token_record = result.scalar_one_or_none()
    if token_record is None or token_record.is_revoked:
        return   # idempotent
    token_record.is_revoked = True
    await session.flush()
```

**9. No new Alembic migration.** The `client.refresh_tokens` table with columns `id`, `user_id`, `token_hash`, `family_id`, `expires_at`, `is_revoked`, `created_at` was created in Story 2.1. `shared.audit_log` also exists. No DDL needed.

**10. `from __future__ import annotations` at top of every modified file.** Already present in all three target files — do not remove.

**11. Use `structlog` for any log statements.** No `print()` or `logging.getLogger()`:
```python
import structlog
logger = structlog.get_logger()
logger.warning("auth.token_breach_detected", user_id=str(token_record.user_id), ...)
```

### Existing Patterns (DO NOT REINVENT)

**`create_refresh_token()` in `core/security.py` already exists:**
```python
from client_api.core.security import create_access_token, create_refresh_token

raw_token, token_hash = create_refresh_token()
# raw_token → returned to client
# token_hash → stored in DB (SHA-256 hex, 64 chars)
```

**`login()` already establishes `family_id` at login time:**
```python
# From auth_service.login():
refresh_token_record = RefreshToken(
    user_id=user.id,
    token_hash=token_hash,
    family_id=uuid4(),          # initial family_id assigned at login
    expires_at=datetime.now(UTC) + timedelta(days=7),
)
```
Story 2.5 rotation inherits this `family_id` and uses it for breach detection.

**User + membership JOIN query from `login()`** — reuse exactly:
```python
stmt = (
    select(User, CompanyMembership)
    .join(CompanyMembership, CompanyMembership.user_id == User.id)
    .where(User.id == token_record.user_id)
    .where(CompanyMembership.accepted_at.is_not(None))
    .limit(1)
)
result = await session.execute(stmt)
row = result.first()
if row is None:
    raise UnauthorizedError("User account or membership not found")
user, membership = row._tuple()
```

**Error conventions** — already in `auth_service.py`:
```python
from eusolicit_common.exceptions import UnauthorizedError
raise UnauthorizedError("Refresh token has been revoked. All active sessions terminated.")
```

**`uuid4` already imported** in `auth_service.py` — no extra import needed for `family_id`.

### Route Handler Implementation

```python
# In api/v1/auth.py — add after the /me route:

from client_api.schemas.auth import (
    ...,          # existing imports
    LogoutRequest,
    RefreshRequest,
)

@router.post("/refresh", status_code=200, response_model=LoginResponse)
async def refresh(
    request: RefreshRequest,
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> LoginResponse:
    """Rotate a refresh token and issue a new access + refresh token pair.

    Accepts the raw refresh token in the request body.
    Revokes the consumed token (rotation). Triggers family revocation if the
    token was already revoked (breach detection).
    """
    return await auth_service.refresh_token(request.refresh_token, session)


@router.post("/logout", status_code=204)
async def logout(
    request: LogoutRequest,
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> None:
    """Revoke the provided refresh token (logout).

    Idempotent: calling with an already-revoked or unknown token is not an error.
    """
    await auth_service.logout(request.refresh_token, session)
```

### File Structure (Modified Files)

```
services/client-api/
  src/client_api/
    schemas/
      auth.py              ← UPDATE: add RefreshRequest, LogoutRequest
    services/
      auth_service.py      ← UPDATE: add refresh_token(), logout(); add `update` import
    api/
      v1/
        auth.py            ← UPDATE: add POST /auth/refresh, POST /auth/logout
  tests/
    api/
      test_auth_refresh.py ← NEW: AC1–AC6 tests (E02-P0-010, E02-P0-011, E02-P1-006, E02-P1-007)
```

No new files in `core/`, no `models/`, no `migrations/`.

### Test Fixture Pattern (CRITICAL — shared session)

Story 2.5 tests need multi-step DB state (register → verify → login → get refresh token → make assertions). Use the **shared-session pattern** from `test_login.py:login_client_and_session`, NOT the `test_client` fixture from `conftest.py`.

**Why:** `conftest.py:test_client` uses a per-request session that commits data. The shared-session pattern overrides `get_db_session` to yield the SAME session for all requests in a test, keeping everything in one transaction that rolls back after the test.

```python
@pytest_asyncio.fixture
async def refresh_client_and_session(
    client_api_session_factory: async_sessionmaker[AsyncSession],
    test_redis_client,
) -> AsyncGenerator[tuple[httpx.AsyncClient, AsyncSession, str], None]:
    """Yield (client, session, refresh_token_str) sharing one DB transaction.

    Sets up:
    - Registered user with email_verified=True
    - Logged-in session → extracts refresh_token from /login response
    - All DB state visible within the shared transaction
    - Rolls back after test — no persistent state
    """
    from client_api.main import app as fastapi_app
    from client_api.dependencies import get_db_session, get_email_service_dep, get_redis_client
    from client_api.services.email_service import StubEmailService
    from httpx import ASGITransport

    uid = uuid.uuid4().hex[:8]
    email = f"refresh-{uid}@eusolicit-test.com"
    password = "SecurePass1"
    company_name = f"Refresh Corp {uid}"

    async with client_api_session_factory() as session:
        async with session.begin():
            async def override_db() -> AsyncGenerator[AsyncSession, None]:
                yield session  # all requests share this transaction

            fastapi_app.dependency_overrides[get_db_session] = override_db
            fastapi_app.dependency_overrides[get_email_service_dep] = lambda: StubEmailService()
            fastapi_app.dependency_overrides[get_redis_client] = lambda: test_redis_client

            async with httpx.AsyncClient(
                transport=ASGITransport(app=fastapi_app),
                base_url="http://test",
                headers={"Content-Type": "application/json"},
            ) as client:
                # Register
                reg = await client.post("/api/v1/auth/register", json={
                    "email": email, "password": password,
                    "full_name": "Refresh Test", "company_name": company_name,
                })
                assert reg.status_code == 201
                user_id = reg.json()["user"]["id"]

                # Verify email directly via SQL
                await session.execute(
                    text("UPDATE client.users SET email_verified = TRUE WHERE id = :id"),
                    {"id": user_id},
                )

                # Login to get refresh token
                login_resp = await client.post("/api/v1/auth/login", json={
                    "email": email, "password": password,
                })
                assert login_resp.status_code == 200
                refresh_token = login_resp.json()["refresh_token"]

                yield client, session, refresh_token

            fastapi_app.dependency_overrides.clear()
        # session.begin() __aexit__ rolls back — no persistent state
```

### Test Design Mapping

| AC | Epic Test ID | Test Class | Description |
|----|-------------|-----------|-------------|
| AC1 | E02-P0-010 (partial) | `TestAC1ValidRefresh` | Valid token → new pair returned |
| AC2 | E02-P0-010 | `TestAC2RotationRevocation` | Consumed token → 401 on reuse |
| AC3+AC6 | E02-P0-011 | `TestAC3BreachDetection` | Revoked token → family revoked + audit log |
| AC4 | E02-P1-007 | `TestAC4ExpiredToken` | Expired token → 401 |
| AC5 | E02-P1-006 | `TestAC5Logout` | Logout revokes token → refresh → 401 |

### Breach Detection Test (E02-P0-011) — Step-by-Step

```python
class TestAC3BreachDetection:
    """E02-P0-011: Replay of revoked token revokes entire family + audit log."""

    async def test_breach_revokes_family_and_logs_audit(
        self, refresh_client_and_session
    ):
        client, session, original_token = refresh_client_and_session

        # Step 1: Rotate token (original becomes revoked, new token issued)
        resp1 = await client.post(
            "/api/v1/auth/refresh", json={"refresh_token": original_token}
        )
        assert resp1.status_code == 200
        new_token = resp1.json()["refresh_token"]

        # Step 2: Replay the original (now-revoked) token — triggers breach
        resp2 = await client.post(
            "/api/v1/auth/refresh", json={"refresh_token": original_token}
        )
        assert resp2.status_code == 401

        # Step 3: New token (from Step 1) also revoked by family sweep
        resp3 = await client.post(
            "/api/v1/auth/refresh", json={"refresh_token": new_token}
        )
        assert resp3.status_code == 401  # new token also invalidated

        # Step 4: Verify audit_log entry exists
        result = await session.execute(
            text(
                "SELECT action_type, entity_type FROM shared.audit_log "
                "WHERE action_type = 'auth.token_breach' "
                "ORDER BY timestamp DESC LIMIT 1"
            )
        )
        row = result.first()
        assert row is not None, "Expected auth.token_breach audit log entry"
        assert row.action_type == "auth.token_breach"
        assert row.entity_type == "refresh_token_family"
```

**Why Step 3 works:** The family revocation in step 2 sets `is_revoked=True` for ALL family tokens (including `new_token`). This is visible within the same shared session transaction.

### Expired Token Test (E02-P1-007) — Direct DB Manipulation

To test AC4 without waiting 7 days, set `expires_at` to the past via SQL:

```python
class TestAC4ExpiredToken:
    async def test_expired_refresh_token_returns_401(
        self, refresh_client_and_session
    ):
        client, session, refresh_token = refresh_client_and_session

        # Hash token to find its DB record
        import hashlib
        token_hash = hashlib.sha256(refresh_token.encode()).hexdigest()

        # Set expires_at to 1 minute ago
        await session.execute(
            text(
                "UPDATE client.refresh_tokens "
                "SET expires_at = NOW() - INTERVAL '1 minute' "
                "WHERE token_hash = :hash"
            ),
            {"hash": token_hash},
        )

        resp = await client.post(
            "/api/v1/auth/refresh", json={"refresh_token": refresh_token}
        )
        assert resp.status_code == 401
```

### Previous Story Intelligence (Story 2.4)

- `core/security.py` now contains: `CurrentUser`, `ROLE_HIERARCHY`, `http_bearer`, `get_current_user`, `require_role`, `create_access_token`, `create_refresh_token`, `get_rsa_private_key`, `get_rsa_public_key`
- `get_current_user` has a `# TODO: S02.05 — check token revocation` comment. Story 2.5 does NOT implement access-token revocation (only refresh token revocation). The TODO comment can remain — revoked refresh tokens still allow the associated access token to live out its 15-minute lifetime (acceptable for MVP).
- Review finding from 2.4: `from __future__ import annotations` + FastAPI route annotation resolution — module-level imports for types used in route signatures are required. When adding routes in `auth.py`, ensure `RefreshRequest`, `LogoutRequest`, `LoginResponse` are at module-level imports.
- 137 tests pass. Story 2.5 must not regress any of these.

### Git Context

The last committed work is Story 2.4 (JWT Authentication Middleware). Files last modified:
- `core/security.py` — latest version with all 2.4 additions
- `api/v1/auth.py` — has `/register`, `/login`, `/me`
- `services/auth_service.py` — has `register()`, `login()`
- `schemas/auth.py` — has `RegisterRequest`, `UserResponse`, `CompanyResponse`, `RegisterResponse`, `LoginRequest`, `LoginResponse`, `CurrentUserResponse`

### Project Structure Notes

- All code in `services/client-api/` under `eusolicit-app/`
- Service import path: `from client_api.services import auth_service`
- Module path for schemas: `from client_api.schemas.auth import RefreshRequest, LogoutRequest, LoginResponse`
- Test files: `eusolicit-app/services/client-api/tests/api/test_auth_refresh.py`
- Markers: `@pytest.mark.integration` for all tests in `test_auth_refresh.py` (they hit the full ASGI stack)

### References

- Epic story definition: [Source: eusolicit-docs/planning-artifacts/epic-02-authentication-identity.md#S02.05]
- Epic test design: [Source: eusolicit-docs/test-artifacts/test-design-epic-02.md#P0] (E02-P0-010, E02-P0-011, E02-P1-006, E02-P1-007)
- RefreshToken model: [Source: eusolicit-app/services/client-api/src/client_api/models/refresh_token.py]
- AuditLog model: [Source: eusolicit-app/services/client-api/src/client_api/models/audit_log.py]
- Existing auth service (login pattern): [Source: eusolicit-app/services/client-api/src/client_api/services/auth_service.py]
- Security module: [Source: eusolicit-app/services/client-api/src/client_api/core/security.py]
- Shared session test pattern: [Source: eusolicit-app/services/client-api/tests/api/test_login.py#login_client_and_session]
- Dependencies (get_db_session commit/rollback): [Source: eusolicit-app/services/client-api/src/client_api/dependencies.py]
- Project rules: [Source: eusolicit-docs/project-context.md]

## Senior Developer Review

**Reviewer:** Claude Code (bmad-code-review)
**Date:** 2026-04-07 (initial) / 2026-04-07 (re-review)
**Test Suite:** 150 passed, 0 failed, 0 errors (20.08s)
**Review Layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor
**Verdict:** REVIEW: Approve

### Review History

**Round 1 (Changes Requested):** Found 3 patch items (1 HIGH, 2 LOW) and 5 deferrals. The HIGH finding — breach detection data rolled back by `get_db_session` exception handler — was a blocking security defect. All 3 patches were applied by the dev agent.

**Round 2 (Approve):** Fresh adversarial re-review with all three parallel layers. All prior patches verified as correctly applied. All 6 ACs fully implemented and meaningfully tested. No new blocking findings.

### Round 1 Findings (all resolved)

- [x] [Review][Patch] **Breach detection family revocation + audit log rolled back by transaction (HIGH)** — When a revoked refresh token is replayed, `refresh_token()` bulk-revokes the family and inserts an `auth.token_breach` audit log via `session.flush()`, then raises `UnauthorizedError`. The `get_db_session` dependency catches all exceptions and calls `session.rollback()`, which **undoes the flush**. The family revocation and audit log entry are never committed to the database. Breach detection is security-theater: the attacker gets 401, but their other valid tokens remain active and no audit trail is recorded. **Fix:** Added `await session.commit()` before `raise UnauthorizedError(...)` in the breach detection block (auth_service.py). Also updated `refresh_client_and_session` fixture from `async with session.begin()` to `try/finally` with explicit `session.rollback()` — the `begin()` context manager prohibits further operations after an internal explicit commit, which caused `InvalidRequestError` in tests. [services/auth_service.py, tests/api/test_auth_refresh.py]

- [x] [Review][Patch] **No input length validation on refresh_token field (LOW)** — `RefreshRequest.refresh_token` and `LogoutRequest.refresh_token` accept empty strings and arbitrarily long inputs. An empty string hashes to the well-known SHA-256 of `""` (`e3b0c44...`), wasting a DB round-trip. An attacker can send multi-MB payloads to force SHA-256 computation over large inputs. **Fix:** Added `Field(min_length=1, max_length=128)` to both schema fields. [schemas/auth.py]

- [x] [Review][Patch] **Stale RED PHASE docstring in test file (LOW)** — Module docstring at lines 4-6 still says "All test methods are decorated with `@pytest.mark.skip`" and references removing them for GREEN PHASE. No skip decorators exist. **Fix:** Updated docstring to reflect GREEN PHASE status. [tests/api/test_auth_refresh.py]

- [x] [Review][Defer] **Race condition in concurrent token rotation** — No `SELECT ... FOR UPDATE` or DB-level constraint prevents two concurrent requests from both successfully rotating the same valid token, forking the family and defeating breach detection. Pre-existing architectural gap; not addressed in story scope. Requires pessimistic locking or a unique constraint on `(family_id, is_revoked=false)`. — deferred, architectural

- [x] [Review][Defer] **Missing is_active check during token refresh** — `refresh_token()` step 6 queries `User + CompanyMembership` but never checks `User.is_active`. Deactivated users can refresh tokens for up to 7 days. Same gap exists in `login()` query pattern. Story spec prescribes reusing the exact login query. — deferred, pre-existing

- [x] [Review][Defer] **Non-deterministic membership selection (LIMIT 1, no ORDER BY)** — Users with multiple accepted `CompanyMembership` rows may receive an arbitrary company_id/role in the refreshed access token. Same pattern used in `login()`. — deferred, pre-existing

- [x] [Review][Defer] **Missing index on family_id column** — `UPDATE ... WHERE family_id = ?` in breach detection has no supporting index, causing sequential scans. Story spec prohibits new Alembic migrations. — deferred, requires migration

- [x] [Review][Defer] **Logout revocation indistinguishable from rotation revocation** — Both `logout()` and `refresh_token()` set `is_revoked=True`. A replayed logged-out token triggers false-positive breach detection. Distinguishing would require a `revocation_reason` column. — deferred, design limitation

### Round 2 Findings (re-review)

All Round 1 patches verified. Fresh adversarial review surfaced the following new observations — all dismissed as non-blocking.

- [x] [Review][Dismissed] **Breach error message distinguishes revoked from invalid tokens (LOW)** — The breach path returns `"Refresh token has been revoked. All active sessions terminated."` while other paths return generic `"Invalid refresh token"`. An attacker could infer breach detection fired. Dismissed: HTTP 401 is returned in all cases; the detailed message aids legitimate client UX (session-terminated notification); internal breach details are logged via structlog, not exposed.

- [x] [Review][Dismissed] **No `email_verified` re-check during refresh (MEDIUM)** — `refresh_token()` does not re-verify `email_verified` status. If email verification is revoked post-login, the user can keep refreshing. Dismissed: story spec mandates reusing the exact login query pattern; `login()` gates on `email_verified` as a separate post-query step, not part of the query itself; revoking email verification post-login is an edge case not addressed by any current story.

- [x] [Review][Dismissed] **Expired tokens not marked as revoked in DB (LOW)** — When an expired token is rejected, it remains `is_revoked=false`. Dismissed: expired tokens are already dead (expiry check prevents use); marking them revoked would add unnecessary write load; the DB state accurately reflects the token was never explicitly revoked, just expired.

- [x] [Review][Dismissed] **No token accumulation cleanup / unbounded table growth (MEDIUM)** — `client.refresh_tokens` grows monotonically (never purged). Dismissed: this is an infrastructure/DevOps concern (background purge job) outside story scope; all production auth systems require periodic token cleanup as a separate operational task.

- [x] [Review][Dismissed] **Stale TODO comment in security.py:159 (LOW)** — `# TODO: S02.05 — check token revocation` remains. Story 2.5 implements refresh-token revocation, not access-token revocation. The TODO refers to access-token-level revocation checking (blocking revoked access tokens during their 15-min lifetime), which is a separate feature. Comment is accurate and should remain.

### Acceptance Criteria Audit

| AC | Implemented | Tested | Tests Authentic | Verdict |
|----|:-----------:|:------:|:---------------:|:-------:|
| AC1 — Valid refresh returns new token pair | YES | YES (2 tests) | YES — full HTTP integration | PASS |
| AC2 — Consumed token revoked + 401 on reuse | YES | YES (2 tests) | YES — DB state + HTTP reuse verified | PASS |
| AC3 — Revoked replay → family revocation + audit | YES | YES (2 tests) | YES — family sweep + audit log verified | PASS |
| AC4 — Expired token → 401 | YES | YES (2 tests) | YES — SQL backdating + negative breach check | PASS |
| AC5 — Logout revokes + 401 on refresh | YES | YES (5 tests) | YES — DB state, cross-endpoint, idempotency | PASS |
| AC6 — Audit log has user_id, entity_type, entity_id | YES | YES (covered by AC3 tests) | YES — field-level assertions | PASS |

### Review Summary

| Category | Round 1 | Round 2 | Total |
|----------|:-------:|:-------:|:-----:|
| Patch (applied) | 3 | 0 | 3 |
| Defer (tracked in deferred-work.md) | 5 | 0 | 5 |
| Dismissed (by-design / out of scope) | 6 | 5 | 11 |

**All blocking findings from Round 1 have been resolved.** The breach detection `session.commit()` fix is verified in code (`auth_service.py:294`) and the test fixture correctly accommodates it (`try/finally` pattern). All 5 deferred items are tracked in `deferred-work.md`. No new actionable findings.

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (Claude Code) / claude-sonnet-4-7 (Claude Code — review fixes)

### Debug Log References

None.

### Completion Notes List

1. All 6 acceptance criteria implemented and verified by 150 passing tests (0 regressions).
2. `structlog` added for breach detection warning log — follows project logging convention.
3. `test_expired_token_does_not_trigger_breach_detection` assertion scoped to `user_id` to be robust against pre-existing audit_log rows in the test DB from prior runs (the shared-session fixture commits rather than rolls back; query now filters by the current test user's ID rather than counting globally).
4. **Review fix (HIGH):** Added `await session.commit()` before `raise UnauthorizedError` in breach detection block so family revocation and audit log survive the `get_db_session` rollback-on-exception lifecycle. Updated `refresh_client_and_session` fixture from `async with session.begin()` to `try/finally` + `session.rollback()` — the `begin()` context manager raises `InvalidRequestError` when an explicit `commit()` is issued inside it and further operations are attempted.
5. **Review fix (LOW):** Added `Field(min_length=1, max_length=128)` to `RefreshRequest.refresh_token` and `LogoutRequest.refresh_token` to prevent empty-string hashing and large-payload SHA-256 abuse.
6. **Review fix (LOW):** Updated module docstring in `test_auth_refresh.py` from RED PHASE to GREEN PHASE language.

### File List

- `eusolicit-app/services/client-api/src/client_api/schemas/auth.py` — added `RefreshRequest`, `LogoutRequest`; added `Field(min_length=1, max_length=128)` validation (review fix)
- `eusolicit-app/services/client-api/src/client_api/services/auth_service.py` — added `refresh_token()`, `logout()`; added `update` import from sqlalchemy; added `structlog` import; added `AuditLog` model import; added `await session.commit()` in breach detection block (review fix)
- `eusolicit-app/services/client-api/src/client_api/api/v1/auth.py` — added `POST /auth/refresh` and `POST /auth/logout` routes; added `RefreshRequest`, `LogoutRequest` imports
- `eusolicit-app/services/client-api/tests/api/test_auth_refresh.py` — GREEN PHASE docstring; `refresh_client_and_session` fixture changed from `session.begin()` context manager to `try/finally` + `session.rollback()` (review fix); fixed `test_expired_token_does_not_trigger_breach_detection` to filter by `user_id`
