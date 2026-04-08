# Story 2.2: Email/Password Registration

Status: done

## Story

As a new company representative,
I want to register my company and create an admin account via `POST /api/v1/auth/register`,
so that my company and I have a platform identity for all subsequent authenticated operations.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/auth/register` creates both `User` and `Company` rows in one atomic database transaction and returns HTTP 201 with `user` + `company` in the response body.
2. **AC2** — The first user is assigned `admin` role in `company_memberships` (with `accepted_at = datetime.now(UTC)`) within the same transaction.
3. **AC3** — Password is hashed with bcrypt at cost factor 12 before storage; plaintext password is never persisted.
4. **AC4** — If the email already exists, endpoint returns 409 Conflict; existing records are unchanged.
5. **AC5** — Weak passwords are rejected with 422: min 8 chars AND ≥1 uppercase AND ≥1 digit (boundary: 7-char → 422; all-lowercase → 422; no-digit → 422; valid → 201).
6. **AC6** — An email verification token (URL-safe, cryptographically random, 32-byte) is generated, SHA-256 hashed, stored in `client.verification_tokens` with `expires_at = now + 24 h`; raw token passed to stub email service (logs it; no SMTP).
7. **AC7** — Response body excludes `hashed_password`; includes `id`, `email`, `full_name`, `is_active`, `email_verified`, `created_at` for user; `id`, `name`, `created_at` for company.

## Tasks / Subtasks

- [ ] Task 1 — Add missing runtime dependencies (AC: 3, 5)
  - [ ] 1.1 Add `bcrypt>=4.0` to `[project] dependencies` in `services/client-api/pyproject.toml`
  - [ ] 1.2 Add `email-validator>=2.0` to `[project] dependencies` (required for Pydantic `EmailStr`)

- [ ] Task 2 — `VerificationToken` ORM model (AC: 6)
  - [ ] 2.1 Create `src/client_api/models/verification_token.py`:
    - Table `client.verification_tokens`, columns: `id UUID PK default uuid4`, `token_hash VARCHAR(64) NOT NULL UNIQUE`, `user_id UUID FK→client.users.id ON DELETE CASCADE NOT NULL`, `expires_at TIMESTAMPTZ NOT NULL`, `consumed_at TIMESTAMPTZ NULL`, `created_at TIMESTAMPTZ NOT NULL server_default=now()`
    - `__table_args__`: `(UniqueConstraint("token_hash"), Index("ix_verification_tokens_user_id", "user_id"), {"schema": "client"})`
  - [ ] 2.2 Add `VerificationToken` import and export to `src/client_api/models/__init__.py`

- [ ] Task 3 — Alembic migration 003: `verification_tokens` (AC: 6)
  - [ ] 3.1 Create `alembic/versions/003_verification_tokens.py` with `down_revision = "002"`, `revision = "003"`
  - [ ] 3.2 `upgrade()`: `op.create_table("verification_tokens", schema="client", ...)` — columns per Task 2.1; `op.create_index("ix_verification_tokens_user_id", "verification_tokens", ["user_id"], schema="client")`
  - [ ] 3.3 `downgrade()`: drop index then drop table (schema="client")

- [ ] Task 4 — Pydantic schemas (AC: 5, 7)
  - [ ] 4.1 Create `src/client_api/schemas/__init__.py`
  - [ ] 4.2 Create `src/client_api/schemas/auth.py` with:
    - `RegisterRequest`: `email: EmailStr`, `password: str` (with `@field_validator("password")` enforcing ≥8 chars + ≥1 uppercase + ≥1 digit — raise `ValueError` on failure), `full_name: str`, `company_name: str`
    - `UserResponse(BaseModel)`: `id: UUID`, `email: str`, `full_name: str`, `is_active: bool`, `email_verified: bool`, `created_at: datetime` — **NO `hashed_password`**
    - `CompanyResponse(BaseModel)`: `id: UUID`, `name: str`, `created_at: datetime`
    - `RegisterResponse(BaseModel)`: `user: UserResponse`, `company: CompanyResponse`
    - All response models: `model_config = ConfigDict(from_attributes=True)` for ORM → Pydantic

- [ ] Task 5 — Email service stub (AC: 6)
  - [ ] 5.1 Create `src/client_api/services/__init__.py`
  - [ ] 5.2 Create `src/client_api/services/email_service.py`:
    - Abstract base `EmailServiceBase(ABC)` with `@abstractmethod async def send_verification_email(self, to_email: str, token: str) -> None`
    - `StubEmailService(EmailServiceBase)`: logs `verification_token_generated` via structlog (token in log field); does NOT raise; does NOT send SMTP
    - Module-level singleton: `_stub = StubEmailService()` and `def get_email_service() -> EmailServiceBase: return _stub`

- [ ] Task 6 — Auth service (AC: 1–7)
  - [ ] 6.1 Create `src/client_api/services/auth_service.py`
  - [ ] 6.2 Implement `async def register(request: RegisterRequest, session: AsyncSession, email_svc: EmailServiceBase) -> RegisterResponse`:
    - Check email uniqueness: `SELECT id FROM client.users WHERE email = ?` → raise `ConflictError("Email already registered")` from `eusolicit_common.exceptions`
    - Hash: `bcrypt.hashpw(request.password.encode(), bcrypt.gensalt(rounds=12)).decode()`
    - Create `Company(name=request.company_name)` → `session.add(company)` → `await session.flush()` (to get company.id)
    - Create `User(email=request.email, hashed_password=hashed, full_name=request.full_name)` → `session.add(user)` → `await session.flush()` (to get user.id)
    - Create `CompanyMembership(user_id=user.id, company_id=company.id, role=CompanyRole.admin, accepted_at=datetime.now(UTC))` → `session.add(membership)`
    - Generate verification token: `raw_token = secrets.token_urlsafe(32)`, `token_hash = hashlib.sha256(raw_token.encode()).hexdigest()`
    - Create `VerificationToken(token_hash=token_hash, user_id=user.id, expires_at=datetime.now(UTC) + timedelta(hours=24))` → `session.add(vt)`
    - `await session.commit()` (single commit = atomic transaction for all 4 objects)
    - `await session.refresh(user)` and `await session.refresh(company)` to load server-defaults (created_at, etc.)
    - `await email_svc.send_verification_email(user.email, raw_token)` (after commit — non-blocking stub)
    - `return RegisterResponse(user=UserResponse.model_validate(user), company=CompanyResponse.model_validate(company))`

- [ ] Task 7 — Service config & DB dependency (AC: 1)
  - [ ] 7.1 Create `src/client_api/config.py`:
    ```python
    import functools
    from pydantic_settings import SettingsConfigDict
    from eusolicit_common.config import BaseServiceSettings
    class ClientApiSettings(BaseServiceSettings):
        model_config = SettingsConfigDict(env_prefix="CLIENT_API_", env_file=".env", extra="ignore")
        service_name: str = "client-api"
    @functools.lru_cache(maxsize=1)
    def get_settings() -> ClientApiSettings: return ClientApiSettings()
    ```
  - [ ] 7.2 Create `src/client_api/dependencies.py`:
    - Module-level `_engine: AsyncEngine | None = None`
    - `def get_engine() -> AsyncEngine` — lazy-creates engine from `ClientApiSettings().database_url`
    - `async def get_db_session() -> AsyncGenerator[AsyncSession, None]` — yields `AsyncSession` from engine; commits on success, rolls back on exception, disposes session on exit
    - `def get_email_service_dep() -> EmailServiceBase` — delegates to `email_service.get_email_service()`

- [ ] Task 8 — Auth API router (AC: 1–7)
  - [ ] 8.1 Create `src/client_api/api/__init__.py` and `src/client_api/api/v1/__init__.py`
  - [ ] 8.2 Create `src/client_api/api/v1/auth.py`:
    ```python
    router = APIRouter(prefix="/auth", tags=["auth"])
    @router.post("/register", status_code=201, response_model=RegisterResponse)
    async def register(
        request: RegisterRequest,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        email_svc: Annotated[EmailServiceBase, Depends(get_email_service_dep)],
    ) -> RegisterResponse:
        return await auth_service.register(request, session, email_svc)
    ```

- [ ] Task 9 — Wire into `main.py` (AC: 1)
  - [ ] 9.1 Update `src/client_api/main.py`:
    - Import `register_exception_handlers` from `eusolicit_common.exceptions` — call on app creation
    - Create `api_v1_router = APIRouter(prefix="/api/v1")` and include `auth.router`
    - Include `api_v1_router` on `app`
    - Keep existing `/healthz` endpoint

- [ ] Task 10 — Update conftest.py and write API tests (AC: 1–7; E02-P0-001, E02-P1-001, E02-P1-002)
  - [ ] 10.1 Update `tests/conftest.py` `app` fixture — replace `pytest.skip` with real app import + dependency overrides:
    ```python
    @pytest_asyncio.fixture
    async def app(client_api_session_factory):
        from client_api.dependencies import get_db_session, get_email_service_dep
        from client_api.main import app as fastapi_app
        async def override_db():
            async with client_api_session_factory() as s:
                async with s.begin():
                    yield s
                    await s.rollback()
        fastapi_app.dependency_overrides[get_db_session] = override_db
        fastapi_app.dependency_overrides[get_email_service_dep] = lambda: StubEmailService()
        yield fastapi_app
        fastapi_app.dependency_overrides.clear()
    ```
  - [ ] 10.2 Create `tests/api/test_register.py` with test cases:
    - `test_register_happy_path_returns_201_with_user_and_company` (E02-P0-001)
    - `test_register_response_excludes_hashed_password` (E02-P0-001)
    - `test_register_creates_admin_membership_in_db` (E02-P0-001) — verify `company_memberships` row via DB
    - `test_register_stores_verification_token_hash_in_db` (AC6)
    - `test_register_duplicate_email_returns_409` (E02-P1-001)
    - `test_register_weak_password_too_short_returns_422` (E02-P1-002)
    - `test_register_weak_password_no_uppercase_returns_422` (E02-P1-002)
    - `test_register_weak_password_no_digit_returns_422` (E02-P1-002)

## Dev Notes

### Architecture Constraints (MUST FOLLOW)

- **Schema isolation enforced:** All new tables use `schema="client"`. Never create tables in `public` or cross-schema FKs from `client` to `shared`. [Source: eusolicit-docs/project-context.md#Database]
- **Single commit = transaction:** All 4 objects (Company, User, CompanyMembership, VerificationToken) must be inserted and committed in a single `await session.commit()`. Use `await session.flush()` after each `session.add()` to resolve IDs before dependent inserts without committing.
- **No `print()` / `logging.getLogger()`:** Use `structlog.get_logger()` from `structlog` for all logging. [Source: eusolicit-docs/project-context.md#Shared Packages]
- **eusolicit-common exceptions:** Use `ConflictError` (409), `BadRequestError` (400), `UnauthorizedError` (401), `ForbiddenError` (403) from `eusolicit_common.exceptions`. These are already wired to FastAPI via `register_exception_handlers`. [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/exceptions.py]
- **Import from eusolicit-common** for config (`BaseServiceSettings`), logging, and exceptions. [Source: eusolicit-docs/project-context.md#Shared Packages rule 9]
- **Python 3.12+ syntax:** Use `type` statements where appropriate, `datetime.now(UTC)` (not `utcnow()` which is deprecated).

### File Structure (New Files)

```
services/client-api/
  pyproject.toml                          ← add bcrypt>=4.0, email-validator>=2.0
  alembic/versions/
    003_verification_tokens.py            ← NEW migration
  src/client_api/
    main.py                               ← UPDATE: wire router + exception handlers
    config.py                             ← NEW
    dependencies.py                       ← NEW
    schemas/
      __init__.py                         ← NEW
      auth.py                             ← NEW: RegisterRequest, *Response schemas
    services/
      __init__.py                         ← NEW
      auth_service.py                     ← NEW
      email_service.py                    ← NEW
    api/
      __init__.py                         ← NEW
      v1/
        __init__.py                       ← NEW
        auth.py                           ← NEW: /register router
    models/
      verification_token.py               ← NEW ORM model
      __init__.py                         ← UPDATE: add VerificationToken
  tests/
    conftest.py                           ← UPDATE: app fixture
    api/
      test_register.py                    ← NEW
```

### bcrypt Usage (Python package, not passlib)

Use the standalone `bcrypt` package (NOT `passlib`). The dependency is `bcrypt>=4.0`.

```python
import bcrypt
# Hash
hashed = bcrypt.hashpw(password.encode("utf-8"), bcrypt.gensalt(rounds=12)).decode("utf-8")
# Verify (for login, Story 2.3)
is_valid = bcrypt.checkpw(password.encode("utf-8"), stored_hash.encode("utf-8"))
```

Cost factor 12 is mandated by the epic. Do not use a lower value even in tests — the test overhead is acceptable.

### Password Validation (Pydantic field_validator)

```python
from pydantic import field_validator
import re

@field_validator("password")
@classmethod
def validate_password_strength(cls, v: str) -> str:
    if len(v) < 8:
        raise ValueError("Password must be at least 8 characters")
    if not re.search(r"[A-Z]", v):
        raise ValueError("Password must contain at least one uppercase letter")
    if not re.search(r"\d", v):
        raise ValueError("Password must contain at least one digit")
    return v
```

Pydantic v2 field validators return `ValueError` → FastAPI converts to 422 Unprocessable Entity automatically.

### Verification Token Pattern

```python
import hashlib, secrets
from datetime import UTC, datetime, timedelta

raw_token = secrets.token_urlsafe(32)          # 43-char URL-safe base64
token_hash = hashlib.sha256(raw_token.encode()).hexdigest()  # 64-char hex → VARCHAR(64)
expires_at = datetime.now(UTC) + timedelta(hours=24)
```

Store only `token_hash` in DB. Pass `raw_token` to the email stub. On verification (Story 2.3/later), recompute the hash and compare. This pattern prevents token leakage from DB compromise.

### Migration 003 Pattern

Follow the exact pattern from `002_auth_identity_tables.py`. Use `down_revision = "002"` (string `"002"`, not a hex hash). The `alembic/env.py` already has `target_metadata = Base.metadata` set in Story 2.1 — autogenerate will work if `VerificationToken` model is correctly mapped and imported.

Example structure:
```python
def upgrade() -> None:
    op.create_table(
        "verification_tokens",
        sa.Column("id", sa.UUID(), primary_key=True, server_default=sa.text("gen_random_uuid()")),
        sa.Column("token_hash", sa.String(64), nullable=False),
        sa.Column("user_id", sa.UUID(), sa.ForeignKey("client.users.id", ondelete="CASCADE"), nullable=False),
        sa.Column("expires_at", sa.DateTime(timezone=True), nullable=False),
        sa.Column("consumed_at", sa.DateTime(timezone=True), nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now(), nullable=False),
        sa.UniqueConstraint("token_hash", name="uq_verification_tokens_token_hash"),
        schema="client",
    )
    op.create_index("ix_verification_tokens_user_id", "verification_tokens", ["user_id"], schema="client")

def downgrade() -> None:
    op.drop_index("ix_verification_tokens_user_id", table_name="verification_tokens", schema="client")
    op.drop_table("verification_tokens", schema="client")
```

### DB Dependency Pattern (FastAPI)

```python
# dependencies.py
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncSession, async_sessionmaker, create_async_engine
from collections.abc import AsyncGenerator

_engine: AsyncEngine | None = None
_session_factory: async_sessionmaker[AsyncSession] | None = None

def get_engine() -> AsyncEngine:
    global _engine, _session_factory
    if _engine is None:
        settings = get_settings()
        _engine = create_async_engine(settings.database_url, pool_pre_ping=True)
        _session_factory = async_sessionmaker(_engine, expire_on_commit=False)
    return _engine

async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    factory = _session_factory or (get_engine() and _session_factory)
    async with factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

Note: In API tests, `get_db_session` is **overridden** via `app.dependency_overrides` with a session that always rolls back. The auth_service must use `await session.flush()` (not commit) for intermediate ID generation within the service, and let the route/dependency handle the final commit.

**IMPORTANT:** Since the test fixture overrides `get_db_session` with a session that rolls back, the `auth_service.register()` must NOT call `session.commit()` internally. Instead, the dependency wrapper handles commit/rollback. Change Task 6.2 — call `await session.flush()` for all adds (not commit), and let the dependency do the final `await session.commit()`.

### Test Dependency Override Pattern

```python
# In conftest.py app fixture
from client_api.dependencies import get_db_session, get_email_service_dep
from client_api.services.email_service import StubEmailService

@pytest_asyncio.fixture
async def app(client_api_session_factory):
    from client_api.main import app as fastapi_app

    async def override_db():
        async with client_api_session_factory() as session:
            async with session.begin():
                yield session
                await session.rollback()  # always rollback in tests

    fastapi_app.dependency_overrides[get_db_session] = override_db
    fastapi_app.dependency_overrides[get_email_service_dep] = lambda: StubEmailService()
    yield fastapi_app
    fastapi_app.dependency_overrides.clear()
```

This means `auth_service.register()` must use `await session.flush()` for intermediate operations and NOT `await session.commit()`. The dependency wrapper (or test override) is responsible for commit/rollback.

### Existing Models — Key Facts

- `User`: `full_name: str` (single field, not first/last), `hashed_password: str | None` (nullable — null for Google-only accounts), `email_verified: bool server_default=false`, `is_active: bool server_default=true` [Source: models/user.py]
- `Company`: Only `name` is required for registration; all other fields (tax_id, address, industry, etc.) are nullable. [Source: models/company.py]
- `CompanyMembership`: Composite PK (user_id, company_id); `accepted_at: datetime | None` — null = pending invite. For registration, set `accepted_at = datetime.now(UTC)` (admin is immediately active). [Source: models/company_membership.py]
- `CompanyRole.admin` = `"admin"` enum value [Source: models/enums.py]

### eusolicit-common Available (Already Installed)

| Export | Use in This Story |
|--------|-------------------|
| `eusolicit_common.exceptions.ConflictError` | Duplicate email → 409 |
| `eusolicit_common.exceptions.register_exception_handlers` | Wire to FastAPI app in main.py |
| `eusolicit_common.config.BaseServiceSettings` | Extend for `ClientApiSettings` |
| `eusolicit_common.logging` | structlog setup (already available) |

### Test Expectations from Epic Test Design

This story satisfies the following epic-level test IDs (from `test-design-epic-02.md`):

| Test ID | Description | AC |
|---------|-------------|-----|
| **E02-P0-001** | Registration creates company + admin user in single transaction; `user` + `company` in response; `hashed_password` absent; `company_memberships` row with `role=admin` in DB | AC1, AC2, AC7 |
| **E02-P1-001** | Duplicate email returns 409 Conflict; existing user unchanged | AC4 |
| **E02-P1-002** | Weak password boundary tests → 422 (7-char, all-lowercase, no-digit fail; valid passes) | AC5 |

Story 2.2 does **not** own E02-P0-002 through E02-P0-012 (JWT/RBAC/refresh — those are Stories 2.3–2.10). TEA ATDD checklist for Story 2.2 will be generated separately.

### Running Tests

```bash
# Run Story 2.2 API tests
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/api/test_register.py -v

# Run with migration applied (requires Docker Compose)
make reset-db && make up
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/api/test_register.py -v

# Verify no regressions on Story 2.1 tests
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v

# Run full client-api test suite
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/ -v
```

### Project Structure Notes

- All new source files go under `src/client_api/` (packages.find with `where=["src"]` in pyproject.toml)
- New Alembic migration must live in `alembic/versions/003_verification_tokens.py` (sequential naming)
- Auth routes use path prefix `/auth` on router + `/api/v1` on app → full path: `POST /api/v1/auth/register`
- No `python-jose` or `passlib` — use standalone `bcrypt` and `pyjwt` (already in deps for Story 2.3)
- `pydantic-settings` is a transitive dep via `eusolicit-common` — no explicit add needed

### References

- Epic story spec: [Source: eusolicit-docs/planning-artifacts/epics/E02-authentication-identity.md#S02.02]
- Test coverage plan: [Source: eusolicit-docs/test-artifacts/test-design-epic-02.md#P0, #P1]
- Existing models: [Source: eusolicit-app/services/client-api/src/client_api/models/]
- eusolicit-common exceptions: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/exceptions.py]
- eusolicit-common config: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/config.py]
- Story 2.1 (previous): [Source: eusolicit-docs/implementation-artifacts/2-1-database-schema-auth-identity-tables.md]
- Project context rules: [Source: eusolicit-docs/project-context.md]
- Migration 002 pattern: [Source: eusolicit-docs/test-artifacts/atdd-checklist-2-1-database-schema-auth-identity-tables.md#Migration Pattern]

## Senior Developer Review

**Review Date:** 2026-04-07
**Review Model:** claude-opus-4-20250514
**Review Layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor (3/3 completed)
**Verdict:** REVIEW: Changes Requested

### Acceptance Criteria Audit

All 7 ACs verified as implemented. All 10 tasks completed. All 3 epic test expectations (E02-P0-001, E02-P1-001, E02-P1-002) covered by 11 passing tests. No AC violations found.

### Review Findings

**Patch Items (4) — actionable, unambiguous fixes:**

- [x] [Review][Patch] **Input length limits missing on full_name and company_name** — Both fields accept arbitrarily long strings. DB columns are `String(255)` so oversized values cause a 500 instead of 422. Add `Field(min_length=1, max_length=255)` to both fields. [schemas/auth.py:16-17]
- [x] [Review][Patch] **Empty/whitespace-only strings accepted for full_name and company_name** — `full_name=""` or `company_name="   "` pass validation and create meaningless records. Add `min_length=1` and a `@field_validator` to reject whitespace-only values (or combine with the above via `Field(min_length=1, max_length=255, strip_whitespace=True)`). [schemas/auth.py:16-17]
- [x] [Review][Patch] **TOCTOU race on email uniqueness — concurrent requests get 500 instead of 409** — The SELECT-then-INSERT pattern for email uniqueness is not atomic. Two concurrent registrations with the same email both pass the check; the second flush raises an unhandled `IntegrityError` (DB UNIQUE constraint), surfacing as a 500 instead of 409. Wrap the user flush in a try/except that catches `IntegrityError` and raises `ConflictError`. [auth_service.py:54]
- [x] [Review][Patch] **Stale TDD RED PHASE docstring in test_register.py** — Module docstring references `@pytest.mark.skip` decorators that no longer exist and says tests are in RED phase, but all tests are active. Update docstring to reflect GREEN/complete status. [test_register.py:1-11]

**Deferred Items (8) — pre-existing or out-of-scope, tracked for future:**

- [x] [Review][Defer] **bcrypt silently truncates passwords at 72 bytes** — No max password length enforced. User could set a 200-char password where only first 72 bytes matter. Enhancement: add `max_length=128` on password field and/or pre-hash with SHA-256. — deferred, security hardening for future story
- [x] [Review][Defer] **Email sent before DB commit** — `send_verification_email` fires inside `register()` before the dependency wrapper commits. With the stub (log-only), this is harmless. With a real email service, a failed commit means an email was sent for a non-existent user. — deferred, address when implementing real email service
- [x] [Review][Defer] **Blocking bcrypt.hashpw on async event loop** — bcrypt cost=12 takes ~200-400ms CPU. Blocks the event loop for all concurrent requests. Wrap in `asyncio.loop.run_in_executor()`. — deferred, performance optimization
- [x] [Review][Defer] **Email case sensitivity** — `User@Example.com` and `user@example.com` are treated as different emails. Pydantic EmailStr normalizes the domain but not the local part. Add `LOWER(email)` index or normalize on input. — deferred, design decision needed
- [x] [Review][Defer] **get_engine() module-level singleton not thread-safe** — Concurrent first access can create duplicate engines and leak connection pools. Add a lock or use `asyncio.Lock`. — deferred, low probability in single-worker deployment
- [x] [Review][Defer] **database_url=None causes cryptic error** — If `CLIENT_API_DATABASE_URL` is not set, `create_async_engine(None)` raises an opaque `ArgumentError`. Add an explicit check with a clear error message. — deferred, pre-existing config weakness
- [x] [Review][Defer] **ORM Mapped[] type annotations use SA types instead of Python types** — `Mapped[sa.DateTime]` and `Mapped[sa.UUID]` should be `Mapped[datetime]` and `Mapped[uuid.UUID]`. Pre-existing pattern across all models (user.py, company.py, etc.). — deferred, codebase-wide fix
- [x] [Review][Defer] **No structlog logging in auth_service.py** — Service layer has no logging of registration events. Observability improvement. — deferred, enhancement

**Dismissed (9):** Token logged in stub (spec-mandated), no rate limiting (infra concern), email enumeration via 409 (spec-mandated), test fixture transaction model (intentional), assert for control flow (safety check with noqa), orphaned company (rollback handles it), password policy in error messages (standard practice), no special char requirement (spec-compliant), token_urlsafe output length (32 bytes entropy, spec-compliant).

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-7

### Debug Log References

None — implementation completed cleanly.

### Completion Notes List

- All 10 tasks completed; 11/11 Story 2.2 tests pass (54/54 total client-api tests pass).
- `auth_service.register()` uses `session.flush()` (not `session.commit()`) so the dependency wrapper controls the transaction lifecycle; test fixture rollback works correctly.
- `test_002_migration.py` `migrated_db` fixture teardown updated to re-upgrade to head after downgrade, preventing DB state pollution across test runs.
- `test_upgrade_head_is_idempotent` assertion updated from `"002"` to `"(head)"` to be migration-version-agnostic.
- `bcrypt>=4.0` and `email-validator>=2.0` added to `pyproject.toml` and installed.
- Migration 003 applied to both `eusolicit` and `eusolicit_test` databases.
- **Review patches applied (2026-04-07):** All 4 [Review][Patch] items resolved; 54/54 tests still pass.
  - `schemas/auth.py`: `full_name` and `company_name` now have `Field(min_length=1, max_length=255)` and a `@field_validator("full_name", "company_name")` that strips and rejects whitespace-only values.
  - `auth_service.py`: User flush now wrapped in `try/except IntegrityError → ConflictError` to handle TOCTOU concurrent-registration race; returns 409 instead of 500.
  - `test_register.py`: Module docstring updated from RED PHASE to GREEN/complete status.

### File List

- `services/client-api/pyproject.toml` — added `bcrypt>=4.0`, `email-validator>=2.0`
- `services/client-api/alembic/versions/003_verification_tokens.py` — NEW migration
- `services/client-api/src/client_api/models/verification_token.py` — NEW ORM model
- `services/client-api/src/client_api/models/__init__.py` — updated: added VerificationToken
- `services/client-api/src/client_api/schemas/__init__.py` — NEW
- `services/client-api/src/client_api/schemas/auth.py` — NEW: RegisterRequest, *Response schemas
- `services/client-api/src/client_api/services/__init__.py` — NEW
- `services/client-api/src/client_api/services/email_service.py` — NEW: StubEmailService
- `services/client-api/src/client_api/services/auth_service.py` — NEW: register()
- `services/client-api/src/client_api/config.py` — NEW: ClientApiSettings
- `services/client-api/src/client_api/dependencies.py` — NEW: get_db_session, get_email_service_dep
- `services/client-api/src/client_api/api/__init__.py` — NEW
- `services/client-api/src/client_api/api/v1/__init__.py` — NEW
- `services/client-api/src/client_api/api/v1/auth.py` — NEW: /register router
- `services/client-api/src/client_api/main.py` — updated: wired router + exception handlers
- `services/client-api/tests/conftest.py` — updated: app fixture (replaced pytest.skip)
- `services/client-api/tests/api/test_register.py` — updated: removed all @pytest.mark.skip
- `services/client-api/tests/integration/test_002_migration.py` — updated: migrated_db teardown restores to head; idempotent test assertion made version-agnostic
