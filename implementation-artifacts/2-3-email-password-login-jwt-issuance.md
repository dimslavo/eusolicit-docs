# Story 2.3: Email/Password Login & JWT Issuance

Status: done

## Story

As a registered company user,
I want to POST my credentials to `POST /api/v1/auth/login`,
so that I receive a signed RS256 JWT access token and opaque refresh token to authenticate subsequent API requests.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/auth/login` with valid, verified credentials returns HTTP 200 with `{ access_token, refresh_token, token_type: "bearer", expires_in: 900 }`.
2. **AC2** — Access token is RS256-signed with 15-minute expiry (900 seconds from issuance).
3. **AC3** — Access token payload includes exactly: `sub` (user_id as string UUID), `company_id` (string UUID), `role` (company role string, e.g. `"admin"`), `exp` (epoch int), `iat` (epoch int), `jti` (unique UUID string per token).
4. **AC4** — Refresh token is an opaque URL-safe random string (never returned as-is in DB); its SHA-256 hash is stored in `client.refresh_tokens` with `user_id`, `expires_at` (now + 7 days), `is_revoked = false`, and `family_id` (UUID assigned at creation time for breach detection in Story 2.5).
5. **AC5** — Invalid credentials (wrong password or non-existent email) return 401 with generic `{"error": "unauthorized", "message": "Invalid credentials"}` body; both cases produce **identical** responses (no user enumeration).
6. **AC6** — Login attempt for a user where `email_verified = false` returns 403 with message `"Email address has not been verified. Please check your inbox."` Only triggered after credential check passes (prevents oracle on unverified vs non-existent).
7. **AC7** — Rate limiting: after 5 consecutive failed login attempts for the same email within a 15-minute sliding window, the 6th and subsequent attempts return 429 with a `Retry-After: <seconds>` response header. A successful login resets the failure counter for that email.

## Tasks / Subtasks

- [ ] Task 1 — Add runtime dependencies (AC: 2, 7)
  - [ ] 1.1 Add `redis[asyncio]>=5.0` to `[project] dependencies` in `services/client-api/pyproject.toml` (async Redis client for login rate limiting)
  - [ ] 1.2 Add `cryptography>=42.0` to `[project] dependencies` (required by PyJWT for RS256 PEM key loading)
  Note: `pyjwt>=2.8` is already present in `pyproject.toml` from Story 2.1 planning.

- [ ] Task 2 — `RefreshToken` ORM model (AC: 4)
  - [ ] 2.1 Create `src/client_api/models/refresh_token.py`:
    - Table `client.refresh_tokens`; columns:
      - `id: UUID PK default=uuid4`
      - `user_id: UUID FK→client.users.id ON DELETE CASCADE NOT NULL`
      - `token_hash: VARCHAR(64) NOT NULL UNIQUE` (SHA-256 hex digest of the raw token)
      - `family_id: UUID NOT NULL` (rotation family for breach detection — used by Story 2.5)
      - `expires_at: TIMESTAMPTZ NOT NULL`
      - `is_revoked: BOOL NOT NULL server_default=false`
      - `created_at: TIMESTAMPTZ NOT NULL server_default=now()`
    - `__table_args__`: `(UniqueConstraint("token_hash", name="uq_refresh_tokens_token_hash"), Index("ix_refresh_tokens_user_id", "user_id"), {"schema": "client"})`
  - [ ] 2.2 Add `RefreshToken` import and export to `src/client_api/models/__init__.py`

- [ ] Task 3 — Alembic migration 004: `refresh_tokens` (AC: 4)
  - [ ] 3.1 Create `alembic/versions/004_refresh_tokens.py` with `down_revision = "003"`, `revision = "004"`
  - [ ] 3.2 `upgrade()`: create table and index per Task 2.1 schema:
    ```python
    op.create_table(
        "refresh_tokens",
        sa.Column("id", sa.UUID(), primary_key=True, server_default=sa.text("gen_random_uuid()")),
        sa.Column("user_id", sa.UUID(), sa.ForeignKey("client.users.id", ondelete="CASCADE"), nullable=False),
        sa.Column("token_hash", sa.String(64), nullable=False),
        sa.Column("family_id", sa.UUID(), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("is_revoked", sa.Boolean(), nullable=False, server_default=sa.text("false")),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.UniqueConstraint("token_hash", name="uq_refresh_tokens_token_hash"),
        schema="client",
    )
    op.create_index("ix_refresh_tokens_user_id", "refresh_tokens", ["user_id"], schema="client")
    ```
  - [ ] 3.3 `downgrade()`: drop index then drop table (schema="client")

- [ ] Task 4 — JWT utilities in `core/security.py` (AC: 2, 3)
  - [ ] 4.1 Create `src/client_api/core/__init__.py` (empty)
  - [ ] 4.2 Create `src/client_api/core/security.py`:
    - Module-level caches: `_private_key_cache: bytes | None = None`, `_public_key_cache: bytes | None = None`
    - `def get_rsa_private_key() -> bytes` — reads `get_settings().rsa_private_key`; raises `RuntimeError` if not set; encodes to bytes; caches in `_private_key_cache`
    - `def get_rsa_public_key() -> bytes` — reads `get_settings().rsa_public_key`; raises `RuntimeError` if not set; encodes to bytes; caches in `_public_key_cache`
    - `def create_access_token(user_id: UUID, company_id: UUID, role: str) -> str`:
      ```python
      now = datetime.now(UTC)
      payload = {
          "sub": str(user_id),
          "company_id": str(company_id),
          "role": role,
          "iat": int(now.timestamp()),
          "exp": int((now + timedelta(minutes=15)).timestamp()),
          "jti": str(uuid4()),
      }
      return jwt.encode(payload, get_rsa_private_key(), algorithm="RS256")
      ```
    - `def create_refresh_token() -> tuple[str, str]`:
      ```python
      raw_token = secrets.token_urlsafe(32)
      token_hash = hashlib.sha256(raw_token.encode()).hexdigest()
      return raw_token, token_hash
      ```

- [ ] Task 5 — Add RSA key settings to `ClientApiSettings` (AC: 2)
  - [ ] 5.1 Update `src/client_api/config.py` — add fields to `ClientApiSettings`:
    ```python
    rsa_private_key: str | None = None   # PEM string; loaded from CLIENT_API_RSA_PRIVATE_KEY
    rsa_public_key: str | None = None    # PEM string; loaded from CLIENT_API_RSA_PUBLIC_KEY
    ```
  Note: `redis_url` is inherited from `BaseServiceSettings.redis_url` — no new field needed; `CLIENT_API_REDIS_URL` resolves via `env_prefix`.

- [ ] Task 6 — Redis login rate limiter in `core/rate_limit.py` (AC: 7)
  - [ ] 6.1 Create `src/client_api/core/rate_limit.py`:
    ```python
    import time
    from redis.asyncio import Redis

    LOGIN_LIMIT = 5
    LOGIN_WINDOW_SECONDS = 900  # 15 minutes

    class RateLimitExceededError(Exception):
        """Raised when a login rate limit is exceeded."""
        def __init__(self, retry_after: int) -> None:
            self.retry_after = retry_after
            super().__init__(f"Rate limit exceeded. Retry after {retry_after}s.")

    class LoginRateLimiter:
        def __init__(self, redis: Redis) -> None:
            self._redis = redis

        def _key(self, email: str) -> str:
            return f"rate_limit:login:{email.lower()}"

        async def check(self, email: str) -> None:
            """Raise RateLimitExceededError if the email has >= LOGIN_LIMIT failures in the window."""
            key = self._key(email)
            count_raw = await self._redis.get(key)
            count = int(count_raw) if count_raw else 0
            if count >= LOGIN_LIMIT:
                ttl = await self._redis.ttl(key)
                retry_after = max(ttl, 1)
                raise RateLimitExceededError(retry_after=retry_after)

        async def record_failure(self, email: str) -> None:
            """Increment the failure counter; set TTL on first failure."""
            key = self._key(email)
            count = await self._redis.incr(key)
            if count == 1:
                # Set window expiry only when counter is created (first failure)
                await self._redis.expire(key, LOGIN_WINDOW_SECONDS)

        async def reset(self, email: str) -> None:
            """Delete the failure counter on successful login."""
            await self._redis.delete(self._key(email))
    ```
  - [ ] 6.2 Update `src/client_api/dependencies.py` to add Redis and rate limiter providers:
    ```python
    import redis.asyncio as aioredis
    from client_api.core.rate_limit import LoginRateLimiter

    _redis_client: aioredis.Redis | None = None

    def get_redis_client() -> aioredis.Redis:
        global _redis_client
        if _redis_client is None:
            url = get_settings().redis_url or "redis://localhost:6379/0"
            _redis_client = aioredis.Redis.from_url(url, decode_responses=True)
        return _redis_client

    def get_login_rate_limiter(
        r: Annotated[aioredis.Redis, Depends(get_redis_client)],
    ) -> LoginRateLimiter:
        return LoginRateLimiter(r)
    ```

- [ ] Task 7 — Pydantic schemas: `LoginRequest`, `LoginResponse` (AC: 1)
  - [ ] 7.1 Add to `src/client_api/schemas/auth.py`:
    ```python
    class LoginRequest(BaseModel):
        """Request body for POST /api/v1/auth/login."""
        email: EmailStr
        password: str

    class LoginResponse(BaseModel):
        """Response body for POST /api/v1/auth/login (HTTP 200)."""
        access_token: str
        refresh_token: str
        token_type: str = "bearer"
        expires_in: int = 900  # seconds (matches 15-min access token lifetime)
    ```

- [ ] Task 8 — Auth service: `login()` (AC: 1–7)
  - [ ] 8.1 Update `src/client_api/services/auth_service.py` — add the following at module scope (new imports: `jwt`, `bcrypt`, `CompanyMembership`, `RefreshToken`, `create_access_token`, `create_refresh_token`, `LoginRateLimiter`, `LoginRequest`, `LoginResponse`, `UnauthorizedError`, `ForbiddenError`).
  - [ ] 8.2 Implement `async def login(request: LoginRequest, session: AsyncSession, rate_limiter: LoginRateLimiter) -> LoginResponse`:
    1. **Rate limit pre-check**: `await rate_limiter.check(request.email)` (before any DB query; raises `RateLimitExceededError` if exceeded — handled at route level)
    2. **Fetch user + active membership** in a single JOIN query:
       ```python
       stmt = (
           select(User, CompanyMembership)
           .join(CompanyMembership, CompanyMembership.user_id == User.id)
           .where(User.email == str(request.email))
           .where(CompanyMembership.accepted_at.is_not(None))
           .limit(1)
       )
       result = await session.execute(stmt)
       row = result.first()
       ```
    3. **Credential verification**: if `row is None` or `user.hashed_password is None` or `not bcrypt.checkpw(request.password.encode("utf-8"), user.hashed_password.encode("utf-8"))`:
       - `await rate_limiter.record_failure(request.email)`
       - `raise UnauthorizedError("Invalid credentials")`
    4. **Email verified check**: if `not user.email_verified`:
       - Do NOT record a rate limit failure here (credential was correct)
       - `raise ForbiddenError("Email address has not been verified. Please check your inbox.")`
    5. **Reset rate limit counter**: `await rate_limiter.reset(request.email)`
    6. **Create access token**: `access_token = create_access_token(user.id, membership.company_id, membership.role.value)`
    7. **Create and store refresh token**:
       ```python
       raw_token, token_hash = create_refresh_token()
       refresh_token_record = RefreshToken(
           user_id=user.id,
           token_hash=token_hash,
           family_id=uuid4(),
           expires_at=datetime.now(UTC) + timedelta(days=7),
       )
       session.add(refresh_token_record)
       await session.flush()
       ```
    8. **Return**: `return LoginResponse(access_token=access_token, refresh_token=raw_token)`

- [ ] Task 9 — Auth router: `POST /login` (AC: 1, 7)
  - [ ] 9.1 Update `src/client_api/api/v1/auth.py` — add login route:
    ```python
    from fastapi import HTTPException
    from client_api.core.rate_limit import RateLimitExceededError
    from client_api.dependencies import get_login_rate_limiter
    from client_api.core.rate_limit import LoginRateLimiter
    from client_api.schemas.auth import LoginRequest, LoginResponse

    @router.post("/login", status_code=200, response_model=LoginResponse)
    async def login(
        request: LoginRequest,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        rate_limiter: Annotated[LoginRateLimiter, Depends(get_login_rate_limiter)],
    ) -> LoginResponse:
        """Validate email/password credentials and issue RS256 JWT + refresh token."""
        try:
            return await auth_service.login(request, session, rate_limiter)
        except RateLimitExceededError as exc:
            raise HTTPException(
                status_code=429,
                detail={"error": "rate_limited", "message": "Too many login attempts. Try again later."},
                headers={"Retry-After": str(exc.retry_after)},
            )
    ```

- [ ] Task 10 — Update conftest.py and write API tests (AC: 1–7; E02-P0-002, E02-P1-003, E02-P1-004, E02-P1-005)
  - [ ] 10.1 Update `tests/conftest.py`:
    - Add `rsa_test_key_pair` session-scoped fixture (generates a 2048-bit RSA key pair using `cryptography`; returns `(private_pem: str, public_pem: str)`)
    - Add `rsa_env_setup` autouse session fixture that sets `CLIENT_API_RSA_PRIVATE_KEY` and `CLIENT_API_RSA_PUBLIC_KEY` env vars and clears the key caches in `core.security` (so tests use the test keys)
    - Add `test_redis_client` async session fixture connecting to test Redis (same URL as `CLIENT_API_REDIS_URL` or default `redis://localhost:6379/1` to avoid collision with app)
    - Override `get_redis_client` in the `app` fixture to return the test Redis client (via `dependency_overrides`)
    - Add `flush_rate_limit_keys` autouse function fixture that calls `await redis_client.delete` on `rate_limit:login:*` pattern after each test
  - [ ] 10.2 Create `tests/api/test_login.py` with the following test classes and functions:

    **`TestAC1AC2AC3LoginHappyPath`** — E02-P0-002
    - `test_login_happy_path_returns_200_with_token_structure` — POST /auth/login with valid creds → 200; body has `access_token`, `refresh_token`, `token_type="bearer"`, `expires_in=900`
    - `test_login_access_token_is_rs256_signed` — decode header of access_token with PyJWT (no verify) → `alg == "RS256"`; then verify with public key → no exception
    - `test_login_access_token_claims` — decode access_token with public key → assert `sub` == user_id, `company_id` == company.id, `role` == "admin", `jti` is a valid UUID string, `exp - iat == 900`

    **`TestAC4RefreshTokenDB`**
    - `test_login_refresh_token_stored_in_db` — POST /login → query `client.refresh_tokens WHERE user_id=?` → row exists; `token_hash` is 64-char hex; `is_revoked=false`; `expires_at ≈ now + 7 days ± 5 min`; `family_id` is a valid UUID; raw refresh_token from response is NOT stored (hash comparison confirms)

    **`TestAC5InvalidCredentials`** — E02-P1-003
    - `test_login_wrong_password_returns_401_generic` — valid email, wrong password → 401; body `error="unauthorized"`, `message="Invalid credentials"`
    - `test_login_nonexistent_email_returns_401_generic` — non-existent email → 401; body `error="unauthorized"`, `message="Invalid credentials"`
    - `test_login_identical_response_prevents_enumeration` — both wrong-password and nonexistent-email → assert `response.json()` dicts are **identical** (no information leakage)

    **`TestAC6UnverifiedEmail`** — E02-P1-004
    - `test_login_unverified_email_returns_403` — register user (email_verified defaults to false) → POST /login → 403; body contains "verified" in message

    **`TestAC7RateLimiting`** — E02-P1-005
    - `test_login_rate_limit_5_failures_then_429` — 5× failed POST /login → 6th attempt → 429
    - `test_login_rate_limit_retry_after_header_present` — after 5 failures → 429 → response has `Retry-After` header with positive integer value
    - `test_login_success_resets_rate_limit_counter` — 4× fail → 1× success → 5× more fail → all return 401 (not 429) confirming counter was reset

## Dev Notes

### Architecture Constraints (MUST FOLLOW)

- **Schema isolation enforced:** All new tables use `schema="client"`. [Source: eusolicit-docs/project-context.md#Database]
- **No `print()` / `logging.getLogger()`:** Use `structlog.get_logger()` for all logging. [Source: eusolicit-docs/project-context.md#Shared Packages rule 9]
- **eusolicit-common exceptions:** `UnauthorizedError` (401), `ForbiddenError` (403) from `eusolicit_common.exceptions`. `RateLimitExceededError` is a local exception (not from eusolicit-common) caught at the route level to add `Retry-After` header.
- **PyJWT, not python-jose:** Use `pyjwt>=2.8` (already in pyproject.toml). Import as `import jwt`.
- **RS256 only:** Never sign tokens with HS256 or RS512. Algorithm is hardcoded to `"RS256"` in `create_access_token`.
- **Rate limit key is per-email, not per-IP:** Key pattern `rate_limit:login:{email.lower()}`. IP-based limiting is handled by the existing `AbstractRateLimitMiddleware`; login needs per-account limiting.
- **auth_service.login() uses flush, not commit:** The dependency wrapper in `get_db_session` handles the final `session.commit()`. Service functions must use `await session.flush()` only.
- **Python 3.12+ syntax:** `datetime.now(UTC)` (not deprecated `utcnow()`), `type` statements where appropriate.

### File Structure (New and Updated Files)

```
services/client-api/
  pyproject.toml                           ← UPDATE: add redis[asyncio]>=5.0, cryptography>=42.0
  alembic/versions/
    004_refresh_tokens.py                  ← NEW migration
  src/client_api/
    core/
      __init__.py                          ← NEW (empty)
      security.py                          ← NEW: RSA key loaders, create_access_token, create_refresh_token
      rate_limit.py                        ← NEW: RateLimitExceededError, LoginRateLimiter
    models/
      refresh_token.py                     ← NEW ORM model
      __init__.py                          ← UPDATE: add RefreshToken
    schemas/
      auth.py                              ← UPDATE: add LoginRequest, LoginResponse
    services/
      auth_service.py                      ← UPDATE: add login()
    api/
      v1/
        auth.py                            ← UPDATE: add /login route
    config.py                              ← UPDATE: add rsa_private_key, rsa_public_key fields
    dependencies.py                        ← UPDATE: add get_redis_client, get_login_rate_limiter
  tests/
    conftest.py                            ← UPDATE: RSA key fixtures, Redis fixture, overrides
    api/
      test_login.py                        ← NEW
```

### PyJWT RS256 Pattern

```python
import jwt
from datetime import UTC, datetime, timedelta
from uuid import UUID, uuid4

def create_access_token(user_id: UUID, company_id: UUID, role: str) -> str:
    now = datetime.now(UTC)
    payload = {
        "sub": str(user_id),
        "company_id": str(company_id),
        "role": role,
        "iat": int(now.timestamp()),
        "exp": int((now + timedelta(minutes=15)).timestamp()),
        "jti": str(uuid4()),
    }
    return jwt.encode(payload, get_rsa_private_key(), algorithm="RS256")

# Decoding (for Story 2.4 middleware and tests):
decoded = jwt.decode(token, get_rsa_public_key(), algorithms=["RS256"])
```

`jwt.encode()` returns a `str` in PyJWT >= 2.0. No `.decode()` call needed.

### RSA Key Loading Pattern

```python
# core/security.py
from client_api.config import get_settings

_private_key_cache: bytes | None = None
_public_key_cache: bytes | None = None

def get_rsa_private_key() -> bytes:
    global _private_key_cache
    if _private_key_cache is None:
        key_str = get_settings().rsa_private_key
        if not key_str:
            raise RuntimeError(
                "CLIENT_API_RSA_PRIVATE_KEY is not set. "
                "Generate an RSA key pair and set the PEM string in the environment."
            )
        _private_key_cache = key_str.encode("utf-8")
    return _private_key_cache
```

**For test fixtures:** The test conftest must invalidate these caches before each test session by setting `core.security._private_key_cache = None` and `core.security._public_key_cache = None` after patching the environment variables. Alternatively, override `core.security.get_rsa_private_key` and `core.security.get_rsa_public_key` directly via `monkeypatch`.

### Test RSA Key Pair Generation

```python
# tests/conftest.py
import pytest
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives.serialization import (
    Encoding, NoEncryption, PrivateFormat, PublicFormat,
)

@pytest.fixture(scope="session")
def rsa_test_key_pair() -> tuple[str, str]:
    """Generate a 2048-bit RSA key pair for test JWT signing/verification."""
    private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
    private_pem = private_key.private_bytes(
        Encoding.PEM, PrivateFormat.TraditionalOpenSSL, NoEncryption()
    ).decode("utf-8")
    public_pem = private_key.public_key().public_bytes(
        Encoding.PEM, PublicFormat.SubjectPublicKeyInfo
    ).decode("utf-8")
    return private_pem, public_pem
```

The `app` fixture must be updated to also override `get_rsa_private_key` / `get_rsa_public_key` in `core.security` (or patch `get_settings()` to return a settings instance with the test keys). The cleanest approach: `monkeypatch.setenv("CLIENT_API_RSA_PRIVATE_KEY", private_pem)` and clear the key caches.

### bcrypt Password Verification Pattern

```python
import bcrypt

# In login() — standalone bcrypt package (NOT passlib)
if not bcrypt.checkpw(
    request.password.encode("utf-8"),
    user.hashed_password.encode("utf-8"),
):
    await rate_limiter.record_failure(request.email)
    raise UnauthorizedError("Invalid credentials")
```

Guard against `hashed_password is None` (Google-only accounts) **before** calling `checkpw`:
```python
if user.hashed_password is None or not bcrypt.checkpw(...):
    await rate_limiter.record_failure(request.email)
    raise UnauthorizedError("Invalid credentials")
```

### User + Membership JOIN Query

The `client.users` table does not carry `company_id` or `role` — those live in `client.company_memberships`. A user may have multiple memberships (one per company). Login uses the **active** membership (where `accepted_at IS NOT NULL`). If multiple active memberships exist (edge case), `LIMIT 1` returns one deterministically — this is acceptable for MVP as multi-company support is not yet surfaced in login. Future stories may add a `company_id` request parameter.

```python
from sqlalchemy import select
from client_api.models import User, CompanyMembership

stmt = (
    select(User, CompanyMembership)
    .join(CompanyMembership, CompanyMembership.user_id == User.id)
    .where(User.email == str(request.email))
    .where(CompanyMembership.accepted_at.is_not(None))
    .limit(1)
)
result = await session.execute(stmt)
row = result.first()
if row is None:
    await rate_limiter.record_failure(request.email)
    raise UnauthorizedError("Invalid credentials")
user, membership = row.tuple()
```

### Rate Limiting Redis Key Pattern

Key: `rate_limit:login:{email.lower()}` — always lowercase the email to treat `User@Example.com` and `user@example.com` as the same account.

**INCR + EXPIRE atomicity note:** The pattern `INCR key` then `EXPIRE key 900` (only if count == 1) is not atomic. Two concurrent first-failures could both receive count=1 and both attempt EXPIRE — this is safe (second EXPIRE is a no-op, same TTL). The race condition where count increments past the window without a TTL is mitigated by always setting EXPIRE when count == 1.

**Test isolation:** The `tests/conftest.py` must flush rate limit keys between tests (or use unique email addresses per test). Recommended approach: use unique email suffixes in test payloads (UUID hex) AND add a teardown that deletes `rate_limit:login:*` keys from the test Redis database. Use Redis database index 1 (`redis://localhost:6379/1`) in tests to avoid collisions with the app's index 0.

### Retry-After Header Strategy

`RateLimitExceededError` is raised inside `auth_service.login()` and propagated up to the route handler, where it is caught and converted to an `HTTPException` with the `Retry-After` header. This approach:
- Keeps header-setting logic in the transport layer (route handler)
- Avoids modifying `eusolicit_common.exceptions.AppException` (which doesn't support response headers)
- Allows `auth_service.login()` to remain transport-agnostic

### Migration 004 Pattern

Follow the exact pattern of `003_verification_tokens.py`. Use `down_revision = "003"` (string, not hex hash). The `family_id` column uses `sa.UUID()` (same as `refresh_tokens.id`). Do **not** add a default UUID in the migration — `family_id` is always supplied by the application at insert time.

### Existing Models — Key Facts

- `User.email_verified: bool server_default=false` — False after registration until email is clicked; Story 2.3 enforces this at login. [Source: models/user.py]
- `User.hashed_password: str | None` — Nullable; null for Google-only accounts (AC5 guard required). [Source: models/user.py]
- `CompanyMembership.accepted_at: datetime | None` — Null = pending invite; not-null = active. Login requires an active membership. [Source: models/company_membership.py]
- `CompanyRole` enum values: `admin`, `bid_manager`, `contributor`, `reviewer`, `read_only`. Access token `role` field uses `.value` (string). [Source: models/enums.py]

### eusolicit-common Available (Already Installed)

| Export | Use in This Story |
|--------|-------------------|
| `eusolicit_common.exceptions.UnauthorizedError` | Invalid credentials → 401 |
| `eusolicit_common.exceptions.ForbiddenError` | Unverified email → 403 |
| `eusolicit_common.exceptions.register_exception_handlers` | Already wired in main.py |
| `eusolicit_common.config.BaseServiceSettings` | `redis_url` already in base (inherited by `ClientApiSettings`) |

### Test Expectations from Epic Test Design

This story satisfies the following epic-level test IDs (from `test-design-epic-02.md`):

| Test ID | Description | AC |
|---------|-------------|-----|
| **E02-P0-002** | Login returns RS256-signed access token (15 min) with correct claims + opaque refresh token | AC1, AC2, AC3, AC4 |
| **E02-P1-003** | Invalid credentials return 401 with generic error (no user enumeration) | AC5 |
| **E02-P1-004** | Unverified email login returns 403 with clear message | AC6 |
| **E02-P1-005** | Login rate limiting — 5 failed attempts per email per 15-min window → 429 with Retry-After header | AC7 |

Story 2.3 does **not** own E02-P0-003 / E02-P0-004 (expired/tampered JWT — those are Story 2.4 middleware tests) or E02-P1-005's "successful login resets counter" which is an extension tested here (AC7, last test case in `TestAC7RateLimiting`). Story 2.3 also does not implement E02-P0-010/E02-P0-011 (token refresh rotation/family revocation — those are Story 2.5).

### Running Tests

```bash
# Run Story 2.3 API tests only
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/api/test_login.py -v

# Run full auth test suite (includes Story 2.2 register tests)
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/api/ -v

# Apply migration and run all client-api tests
make reset-db && make up
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/ -v

# Run integration migration test to verify 004 migration
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v
```

### References

- Epic story spec: [Source: eusolicit-docs/planning-artifacts/epic-02-authentication-identity.md#S02.03]
- Test coverage plan: [Source: eusolicit-docs/test-artifacts/test-design-epic-02.md#P0, #P1]
- Existing auth service: [Source: eusolicit-app/services/client-api/src/client_api/services/auth_service.py]
- Existing auth schemas: [Source: eusolicit-app/services/client-api/src/client_api/schemas/auth.py]
- Existing models: [Source: eusolicit-app/services/client-api/src/client_api/models/]
- eusolicit-common exceptions: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/exceptions.py]
- eusolicit-common rate limit middleware: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/middleware/rate_limit.py]
- Story 2.2 (registration): [Source: eusolicit-docs/implementation-artifacts/2-2-email-password-registration.md]
- Migration 003 pattern: [Source: eusolicit-app/services/client-api/alembic/versions/003_verification_tokens.py]
- Project context rules: [Source: eusolicit-docs/project-context.md]

## Senior Developer Review

**Date:** 2026-04-07
**Verdict:** APPROVED

All 7 acceptance criteria verified against implementation. Architecture constraints satisfied (schema isolation, eusolicit-common exceptions, PyJWT RS256, flush-not-commit, Python 3.12+ syntax). 11 tests cover all ACs and epic test IDs (E02-P0-002, E02-P1-003, E02-P1-004, E02-P1-005). 65/65 full suite passes — zero regressions.

Three-layer adversarial review (Blind Hunter, Edge Case Hunter, Acceptance Auditor) produced 17 raw findings. After deduplication and triage: **0 patch, 0 decision-needed, 11 deferred (pre-existing/out-of-scope), 5 dismissed (noise/by-design).**

Key deferred items (not blocking — pre-existing or out-of-scope for this story):
- [x] [Review][Defer] Unbounded password length on LoginRequest/RegisterRequest (bcrypt HashDoS vector) — pre-existing from Story 2.2; fix both schemas together
- [x] [Review][Defer] 429 response envelope uses HTTPException `{"detail": {...}}` vs standard error envelope `{"error": ...}` — AC7 doesn't specify body format; Retry-After header correct; no test relies on body
- [x] [Review][Defer] Rate limit INCR/EXPIRE non-atomic under concurrency — documented as acceptable in story notes; Lua script upgrade planned
- [x] [Review][Defer] Multi-membership non-deterministic (LIMIT 1, no ORDER BY) — documented MVP limitation; future company-selector planned
- [x] [Review][Defer] Refresh token row growth (no cleanup) — Story 2.5 scope (rotation/revocation)
- [x] [Review][Defer] `Mapped[sa.UUID]` type annotations (should be `Mapped[uuid.UUID]`) — project-wide convention across all 8 models
- [x] [Review][Defer] Redis unavailability causes hard 500 on all logins — infrastructure concern; fail-closed is safer than fail-open for rate limiting
- [x] [Review][Defer] Redis singleton lifecycle across event loop restarts — pre-existing pattern (DB engine identical)
- [x] [Review][Defer] TTL=-1 edge case could permanently lock out an email — requires key eviction between INCR and EXPIRE; extremely unlikely
- [x] [Review][Defer] Anti-enumeration test fragile if correlation-ID middleware added later — works correctly now
- [x] [Review][Defer] Cross-store inconsistency if DB flush fails after Redis rate-limit reset — benign (counter resets to 0)

## Dev Agent Record

### Completion Notes

**Review Follow-up Session — 2026-04-07**

All 3 code review patches resolved. 65/65 tests pass (11 login + 54 existing — zero regressions).

- ✅ Resolved review finding [Patch]: Duplicate unique constraint — removed `unique=True` from `token_hash` `mapped_column`; named `UniqueConstraint` in `__table_args__` is now the sole constraint.
- ✅ Resolved review finding [Patch]: Stale RED PHASE docstring — updated `test_login.py` module docstring to reflect GREEN PHASE / completed implementation.
- ✅ Resolved review finding [Patch]: Shared test fixtures — extracted `rsa_test_key_pair` (session), `rsa_env_setup` (session, autouse), `test_redis_client` (session), `flush_rate_limit_keys` (function, autouse) to `tests/conftest.py`; updated `app` fixture to override `get_redis_client`; simplified all three local fixtures in `test_login.py` to use shared conftest fixtures (removing ~120 lines of duplicated setup/teardown code).

### File List

- `services/client-api/src/client_api/models/refresh_token.py` — removed `unique=True` from `token_hash` mapped_column
- `services/client-api/tests/conftest.py` — added `rsa_test_key_pair`, `rsa_env_setup`, `test_redis_client`, `flush_rate_limit_keys` fixtures; updated `app` fixture to inject test Redis
- `services/client-api/tests/api/test_login.py` — updated module docstring; simplified `login_client_and_session`, `unverified_login_client`, `rate_limit_login_client` fixtures to use conftest shared fixtures

### Change Log

- 2026-04-07: Addressed code review findings — 3 patch items resolved

---

## Senior Developer Review

**Date:** 2026-04-07
**Verdict:** Changes Requested → All Patches Resolved
**Review Layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor (all completed)
**Stats:** 3 patch (all resolved), 5 deferred, 1 dismissed

---

### Acceptance Criteria Compliance

All 7 ACs verified against implementation:

| AC | Status | Evidence |
|----|--------|----------|
| AC1 | ✅ Pass | `POST /login` → 200 with `{access_token, refresh_token, token_type:"bearer", expires_in:900}` via `LoginResponse` defaults |
| AC2 | ✅ Pass | `jwt.encode(..., algorithm="RS256")` + `timedelta(minutes=15)` in `core/security.py:84-87` |
| AC3 | ✅ Pass | Payload includes exactly `sub`, `company_id`, `role`, `exp`, `iat`, `jti` — `core/security.py:79-86` |
| AC4 | ✅ Pass | SHA-256 hash stored in `client.refresh_tokens` with `user_id`, `expires_at` (now+7d), `is_revoked=false`, `family_id` — `auth_service.py:215-223` |
| AC5 | ✅ Pass | Both wrong-password and non-existent email → `UnauthorizedError("Invalid credentials")` with identical response bodies — `auth_service.py:189-200` |
| AC6 | ✅ Pass | `email_verified=false` → `ForbiddenError(...)` only after credential check passes — `auth_service.py:202-206` |
| AC7 | ✅ Pass | 5 failures/15-min window → `RateLimitExceededError` → 429 + `Retry-After` header; success resets counter — `rate_limit.py` + `auth.py:52-57` |

### Architecture Alignment

| Rule | Status | Notes |
|------|--------|-------|
| Schema isolation (`schema="client"`) | ✅ | `RefreshToken.__table_args__`, migration 004 |
| No `print()`/`logging.getLogger()` | ✅ | No violations; `email_service.py` uses `structlog` correctly |
| `eusolicit_common` exceptions | ✅ | `UnauthorizedError` (401), `ForbiddenError` (403) imported from `eusolicit_common.exceptions` |
| `pyjwt` not `python-jose` | ✅ | `import jwt` in `core/security.py` |
| RS256 only | ✅ | Hardcoded `algorithm="RS256"` |
| Rate limit key per-email (`.lower()`) | ✅ | `rate_limit.py:37` |
| `session.flush()` not `session.commit()` | ✅ | `auth_service.py:223` |
| Python 3.12+ syntax | ✅ | `datetime.now(UTC)`, `bytes | None` union syntax |

### Test Coverage

11 tests across 5 classes covering all 4 epic test IDs:

| Epic Test ID | AC Coverage | Test Class | Tests |
|-------------|-------------|------------|-------|
| E02-P0-002 | AC1,AC2,AC3,AC4 | `TestAC1AC2AC3LoginHappyPath` + `TestAC4RefreshTokenDB` | 4 |
| E02-P1-003 | AC5 | `TestAC5InvalidCredentials` | 3 |
| E02-P1-004 | AC6 | `TestAC6UnverifiedEmail` | 1 |
| E02-P1-005 | AC7 | `TestAC7RateLimiting` | 3 |

---

### Review Findings

#### Patch (fix before closing)

- [x] [Review][Patch] **Duplicate unique constraint on `token_hash`** [`models/refresh_token.py:15,33`] — `mapped_column(..., unique=True)` on line 33 creates a column-level constraint AND `UniqueConstraint("token_hash", name="uq_refresh_tokens_token_hash")` in `__table_args__` on line 15 creates a table-level named constraint. This produces two identical constraints on the same column when using `Base.metadata.create_all()` (test environments). Remove `unique=True` from the `mapped_column` call; the named `UniqueConstraint` in `__table_args__` is the authoritative one (matches migration 004).

- [x] [Review][Patch] **Stale RED PHASE docstring in test file** [`tests/api/test_login.py:4`] — Module docstring states "All 11 tests are @pytest.mark.skip'd. Zero passes until implementation is complete." but no tests carry `@pytest.mark.skip` and the story status is `done`. Update docstring to reflect GREEN/DONE phase (e.g. "GREEN PHASE: All 11 tests passing against the completed implementation.").

- [x] [Review][Patch] **Shared test fixtures not extracted to conftest.py per Task 10.1** [`tests/api/test_login.py`, `tests/conftest.py`] — Task 10.1 explicitly requires `rsa_test_key_pair`, `rsa_env_setup`, `test_redis_client`, `get_redis_client` override, and `flush_rate_limit_keys` fixtures in `tests/conftest.py`. Instead, ~200 lines of near-identical setup (RSA key generation, Redis DB-1 init, dependency overrides, cache teardown) are duplicated verbatim across three local fixtures (`login_client_and_session`, `unverified_login_client`, `rate_limit_login_client`). Extract shared infrastructure to conftest.py per the spec. This is **critical** for Stories 2.4 (JWT middleware) and 2.5 (token refresh) which will need the same RSA/Redis fixtures, and compounds DRY violations with each new story. Also violates project-context rules on shared test utilities.

#### Dismissed

- [x] [Review][Dismiss] **Prior finding: `row._tuple()` → `row.tuple()`** [`auth_service.py:193`] — Dismissed after verification. SQLAlchemy 2.0.19 **deprecated** `Row.tuple()` in favor of `Row._tuple()` (underscore avoids column name conflicts). The installed SA version is 2.0.49. `Row._tuple()` is the correct public API. No change needed.

#### Deferred (pre-existing or out-of-scope — tracked separately)

- [x] [Review][Defer] **Missing `User.is_active` check in login query** [`auth_service.py:178-186`] — deferred, not specified in AC. The login JOIN query does not filter on `User.is_active`, so deactivated users can still authenticate. Real security gap but not in scope for Story 2.3 acceptance criteria. Track as future security hardening story.

- [x] [Review][Defer] **Redis outage blocks all logins (no graceful degradation)** [`auth_service.py:174-175`] — deferred, architectural concern. If Redis is unreachable, `rate_limiter.check()` throws a `ConnectionError` which propagates as 500. All logins fail. Consider wrapping rate limiter calls with try/except to allow login to proceed when Redis is down (fail-open for rate limiting).

- [x] [Review][Defer] **Nondeterministic membership selection for multi-company users** [`auth_service.py:184`] — deferred, acknowledged in dev notes as acceptable for MVP. `LIMIT 1` without `ORDER BY` returns an arbitrary membership. Future multi-company login should add `company_id` to `LoginRequest` or use `ORDER BY accepted_at DESC`.

- [x] [Review][Defer] **Rate limiter INCR+EXPIRE non-atomicity** [`core/rate_limit.py:52-55`] — deferred, acknowledged in dev notes. Under extreme concurrency near key expiry, the counter could theoretically become immortal (no TTL). Fix: use Redis pipeline (`INCR` + `EXPIRE` in one roundtrip) or a Lua script.

- [x] [Review][Defer] **ORM `Mapped[sa.UUID]` type annotations** [`models/refresh_token.py`] — deferred, pre-existing pattern across all models. Should be `Mapped[uuid.UUID]` and `Mapped[datetime]` per SQLAlchemy 2.0 best practices. Affects type checking. Applies to User, CompanyMembership, and all other models equally.
