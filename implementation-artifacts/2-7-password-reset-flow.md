# Story 2.7: Password Reset Flow

Status: done

## Story

As a registered user who has forgotten my password,
I want to request a password reset link via email and then set a new password,
so that I can regain access to my account without contacting support.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/auth/password-reset/request` accepts an `email` field and **always** returns HTTP 200 with an identical response body regardless of whether the email exists in the database (no user enumeration).
2. **AC2** — When the email corresponds to an existing user, a cryptographically random, URL-safe reset token is generated, SHA-256 hashed before storage, and expires in exactly 1 hour from creation.
3. **AC3** — Each reset token is single-use: `consumed_at` is set on successful confirmation; a subsequent call with the same token returns HTTP 400 with a clear message.
4. **AC4** — `POST /api/v1/auth/password-reset/confirm` validates the token (not expired, not consumed), enforces the same password policy as registration (min 8 chars, 1 uppercase, 1 digit), and updates `users.hashed_password` with a fresh bcrypt hash (cost=12).
5. **AC5** — After a successful password reset, **all** existing `RefreshToken` rows for the user are bulk-revoked (`is_revoked=True`) so no prior session can survive the reset.
6. **AC6** — Presenting an expired or already-consumed token to the confirm endpoint returns HTTP 400 with a message indicating the token is invalid or expired (no distinction between the two failure modes).

## Tasks / Subtasks

- [x] Task 1 — Create `PasswordResetToken` ORM model (AC: 2, 3, 6)
  - [x] 1.1 Create `src/client_api/models/password_reset_token.py` with `PasswordResetToken` class mapping to `client.password_reset_tokens` table: columns `id` (UUID PK, default=uuid4), `token_hash` (VARCHAR 64, unique, not null), `user_id` (UUID FK → `client.users.id` CASCADE), `expires_at` (TIMESTAMPTZ not null), `consumed_at` (TIMESTAMPTZ nullable), `created_at` (TIMESTAMPTZ server_default=now())
  - [x] 1.2 Add `UniqueConstraint("token_hash")` and `Index("ix_password_reset_tokens_user_id", "user_id")` in `__table_args__`; set `{"schema": "client"}` in `__table_args__`
  - [x] 1.3 Export `PasswordResetToken` from `src/client_api/models/__init__.py` (add to import list and `__all__`)

- [x] Task 2 — Add Alembic migration 006 for `password_reset_tokens` table (AC: 2)
  - [x] 2.1 Create `alembic/versions/006_password_reset_tokens.py` with `upgrade()` that executes `CREATE TABLE client.password_reset_tokens (...)` mirroring the ORM model exactly (UUID PK with `gen_random_uuid()` default, token_hash VARCHAR(64) UNIQUE NOT NULL, user_id UUID NOT NULL REFERENCES client.users(id) ON DELETE CASCADE, expires_at TIMESTAMPTZ NOT NULL, consumed_at TIMESTAMPTZ, created_at TIMESTAMPTZ NOT NULL DEFAULT now())
  - [x] 2.2 Add `CREATE UNIQUE INDEX uq_password_reset_tokens_token_hash ON client.password_reset_tokens (token_hash)` and `CREATE INDEX ix_password_reset_tokens_user_id ON client.password_reset_tokens (user_id)` in `upgrade()`
  - [x] 2.3 Add `downgrade()` that drops `client.password_reset_tokens`
  - [x] 2.4 Set `down_revision` to the revision ID of migration 005

- [x] Task 3 — Extend `EmailServiceBase` and `StubEmailService` with password reset email method (AC: 2)
  - [x] 3.1 Add abstract method `send_password_reset_email(self, to_email: str, token: str) -> None` to `EmailServiceBase` in `services/email_service.py`
  - [x] 3.2 Implement `send_password_reset_email` in `StubEmailService` — log via structlog with `log.info("password_reset_token_generated", to_email=to_email, reset_token=token)` (token visible in logs for test capture)

- [x] Task 4 — Add Pydantic schemas to `schemas/auth.py` (AC: 1, 4)
  - [x] 4.1 Add `PasswordResetRequestRequest(BaseModel)` with field `email: EmailStr`
  - [x] 4.2 Add `PasswordResetConfirmRequest(BaseModel)` with fields `token: str = Field(min_length=1, max_length=128)` and `new_password: str`; apply the same `@field_validator("new_password")` password-strength check as `RegisterRequest.validate_password_strength` (min 8 chars, 1 uppercase, 1 digit)

- [x] Task 5 — Implement `request_password_reset()` and `confirm_password_reset()` in `auth_service.py` (AC: 1–6)
  - [x] 5.1 Implement `async def request_password_reset(request: PasswordResetRequestRequest, session: AsyncSession, email_svc: EmailServiceBase) -> None`:
    - Look up `User` by email using `SELECT … WHERE email = :email LIMIT 1`
    - If user is not found → return silently (no error — enumeration prevention, AC1)
    - Generate raw token: `secrets.token_urlsafe(32)`; compute `token_hash = hashlib.sha256(raw_token.encode()).hexdigest()`
    - Create `PasswordResetToken(token_hash=token_hash, user_id=user.id, expires_at=datetime.now(UTC) + timedelta(hours=1))`; `session.add(...)`; `await session.flush()`
    - Call `await email_svc.send_password_reset_email(user.email, raw_token)` after DB flush (non-blocking stub)
  - [x] 5.2 Implement `async def confirm_password_reset(request: PasswordResetConfirmRequest, session: AsyncSession) -> None`:
    - Hash the incoming token: `token_hash = hashlib.sha256(request.token.encode()).hexdigest()`
    - Query `PasswordResetToken` by `token_hash`; if not found → raise `ValidationError("Invalid or expired reset token")` (results in HTTP 400)
    - If `consumed_at is not None` → raise `ValidationError("Invalid or expired reset token")` (AC3, AC6)
    - Check expiry: if `expires_at < datetime.now(UTC)` → raise `ValidationError("Invalid or expired reset token")` (AC6)
    - Load the `User` by `token_record.user_id`
    - Hash new password: `bcrypt.hashpw(request.new_password.encode(), bcrypt.gensalt(rounds=12)).decode()`; set `user.hashed_password = new_hash`; `await session.flush()`
    - Mark token consumed: `token_record.consumed_at = datetime.now(UTC)`; `await session.flush()`
    - Bulk-revoke all refresh tokens: `await session.execute(update(RefreshToken).where(RefreshToken.user_id == user.id).values(is_revoked=True))` (AC5)
    - `await session.flush()`
  - [x] 5.3 Import `PasswordResetToken` from `client_api.models` and `PasswordResetRequestRequest`, `PasswordResetConfirmRequest` from `client_api.schemas.auth` at the top of `auth_service.py`

- [x] Task 6 — Add route handlers to `api/v1/auth.py` (AC: 1, 4, 6)
  - [x] 6.1 Import `PasswordResetRequestRequest`, `PasswordResetConfirmRequest` from `client_api.schemas.auth`
  - [x] 6.2 Add `POST /auth/password-reset/request` route (status_code=200): calls `auth_service.request_password_reset(request, session, email_svc)`; wraps `ValidationError` (from eusolicit_common or a local ValueError) in `HTTPException(400, …)`; always returns `{"message": "If the email is registered, a reset link has been sent"}`
  - [x] 6.3 Add `POST /auth/password-reset/confirm` route (status_code=200): calls `auth_service.confirm_password_reset(request, session)`; wraps `ValidationError` in `HTTPException(400, detail={"error": "invalid_token", "message": "Invalid or expired reset token"})`; returns `{"message": "Password has been reset successfully"}`

- [x] Task 7 — Write API tests `tests/api/test_auth_password_reset.py` (AC: 1–6; E02-P1-008, E02-P1-009, E02-P1-010, E02-P3-003)
  - [x] 7.1 Create `password_reset_client_and_session` fixture following the shared-session pattern (same shape as `refresh_client_and_session`): registers a unique user, verifies email via SQL, yields `(client, session)` with rollback in `finally`
  - [x] 7.2 `TestAC1NoEnumeration` — POST `/auth/password-reset/request` with non-existent email → 200, response body contains `"message"` key; repeat with existing email → 200, identical response shape (E02-P1-008)
  - [x] 7.3 `TestAC2TokenProperties` — After request with existing email, query `password_reset_tokens` table directly via session; assert `token_hash` is 64-char hex string; assert `consumed_at IS NULL`; assert `expires_at` is within ~1 hour of now (±5s); assert raw token does NOT appear in the table (hash-only storage)
  - [x] 7.4 `TestAC3SingleUse` — Capture the raw token via `StubEmailService` monkey-patch (capture the `token` argument passed to `send_password_reset_email`); call confirm once → 200; call confirm again with same token → 400 with `"invalid_token"` error code (E02-P1-009)
  - [x] 7.5 `TestAC4PasswordPolicyEnforced` — Call confirm with a valid token but a weak `new_password` (7-char, no uppercase, no digit) → 422 (Pydantic validation error before service is reached)
  - [x] 7.6 `TestAC5RefreshTokensRevoked` — Log in to obtain a refresh token; call request + confirm; attempt `POST /auth/refresh` with the pre-reset refresh token → 401 (E02-P1-010)
  - [x] 7.7 `TestAC6ExpiredToken` — Insert a `PasswordResetToken` row directly via session with `expires_at = datetime.now(UTC) - timedelta(seconds=1)`; call confirm → 400 with `"invalid_token"` error code (E02-P1-009)
  - [x] 7.8 `TestP3FullFlow` — End-to-end: register → verify email → login → capture refresh token → request password reset → capture reset token from stub → confirm reset → assert old refresh token rejected → login with new password succeeds → assert old login credentials rejected (E02-P3-003)

## Dev Notes

### Architecture Constraints (MUST FOLLOW)

**1. Extend existing files only — do NOT create new modules except the model.**
Files to modify:
- `src/client_api/models/password_reset_token.py` — **NEW** model file (acceptable: new model per existing pattern)
- `src/client_api/models/__init__.py` — add `PasswordResetToken` export
- `alembic/versions/006_password_reset_tokens.py` — **NEW** migration
- `src/client_api/services/email_service.py` — add `send_password_reset_email()` abstract + stub
- `src/client_api/schemas/auth.py` — add `PasswordResetRequestRequest`, `PasswordResetConfirmRequest`
- `src/client_api/services/auth_service.py` — add `request_password_reset()`, `confirm_password_reset()`
- `src/client_api/api/v1/auth.py` — add two new route handlers
- `tests/api/test_auth_password_reset.py` — **NEW** test file

**2. No new Alembic table columns on `users`.** The password reset flow updates `users.hashed_password` (existing column from migration 002). No schema change to `users` is needed.

**3. Enumeration prevention is non-negotiable (AC1, E02-R-005).** The `request_password_reset()` service function must return `None` (not raise) when the email is not found. The route handler always responds with the same 200 body. No timing side-channel mitigation is required for MVP, but the response body MUST be identical for existing and non-existing emails.

**4. Token storage: hash-before-store (AC2, E02-R-005).** Raw token sent to user; SHA-256 hash persisted in DB. A DB breach does not expose valid reset links. Pattern is identical to `VerificationToken` in Story 2.2. Use `hashlib.sha256(raw_token.encode()).hexdigest()` — 64-character hex string fits `VARCHAR(64)`.

**5. Bulk refresh token revocation (AC5).** After successful password reset, issue a bulk UPDATE:
```python
await session.execute(
    update(RefreshToken)
    .where(RefreshToken.user_id == user.id)
    .values(is_revoked=True)
)
```
This follows the same approach as breach detection in Story 2.5 but scoped to a single user rather than a token family. No audit log entry is required for MVP (deferred to Story 2.11 audit trail middleware).

**6. Error type for token validation failures.** The `eusolicit_common` package exposes `ValidationError` (HTTP 400) — use this for token-not-found, token-consumed, and token-expired cases. The route handler wraps it in `HTTPException(400)`. All three failure modes return the same error body (`{"error": "invalid_token", "message": "Invalid or expired reset token"}`) — no distinction (AC6).

**7. Password policy re-use.** The `PasswordResetConfirmRequest.new_password` validator must be a copy of `RegisterRequest.validate_password_strength` (same regex, same error messages). Do NOT import from `RegisterRequest` — Pydantic validators on classmethods cannot be shared via inheritance in this pattern; copy the implementation.

**8. `from __future__ import annotations` at top of every modified file.** Already present in all target files; maintain it in the new model file.

**9. Use `structlog` for all log statements.** No `print()` or `logging.getLogger()`. Import: `import structlog; logger = structlog.get_logger()`.

**10. `get_db_session` handles commit/rollback.** Do NOT call `session.commit()` inside the service functions (unlike the breach detection special case in Story 2.5 — this flow has no need for a mid-exception commit).

### `PasswordResetToken` Model

```python
# src/client_api/models/password_reset_token.py
"""PasswordResetToken ORM model — client.password_reset_tokens table."""
from __future__ import annotations

from uuid import uuid4

import sqlalchemy as sa
from sqlalchemy.orm import Mapped, mapped_column

from .base import Base


class PasswordResetToken(Base):
    """Password reset token — stores SHA-256 hash (never the raw token)."""

    __tablename__ = "password_reset_tokens"
    __table_args__ = (
        sa.UniqueConstraint("token_hash", name="uq_password_reset_tokens_token_hash"),
        sa.Index("ix_password_reset_tokens_user_id", "user_id"),
        {"schema": "client"},
    )

    id: Mapped[sa.UUID] = mapped_column(
        sa.UUID(as_uuid=True),
        primary_key=True,
        default=uuid4,
    )
    token_hash: Mapped[str] = mapped_column(
        sa.String(64),
        nullable=False,
        unique=True,
    )
    user_id: Mapped[sa.UUID] = mapped_column(
        sa.UUID(as_uuid=True),
        sa.ForeignKey("client.users.id", ondelete="CASCADE"),
        nullable=False,
    )
    expires_at: Mapped[sa.DateTime] = mapped_column(
        sa.DateTime(timezone=True),
        nullable=False,
    )
    consumed_at: Mapped[sa.DateTime | None] = mapped_column(
        sa.DateTime(timezone=True),
        nullable=True,
    )
    created_at: Mapped[sa.DateTime] = mapped_column(
        sa.DateTime(timezone=True),
        server_default=sa.func.now(),
        nullable=False,
    )
```

### Service Implementation Sketch

```python
# In auth_service.py — add after the google_callback() function

import structlog
logger = structlog.get_logger()  # already defined in auth_service.py

from client_api.models import PasswordResetToken  # add to existing import
from client_api.schemas.auth import (
    PasswordResetRequestRequest,  # add to existing import
    PasswordResetConfirmRequest,
)

async def request_password_reset(
    request: PasswordResetRequestRequest,
    session: AsyncSession,
    email_svc: EmailServiceBase,
) -> None:
    """Issue a password reset token if the email exists — always returns silently."""
    result = await session.execute(
        select(User).where(User.email == str(request.email)).limit(1)
    )
    user = result.scalar_one_or_none()
    if user is None:
        # Enumeration prevention: do not reveal whether the email exists
        logger.info("password_reset.email_not_found", email=str(request.email))
        return

    raw_token = secrets.token_urlsafe(32)
    token_hash = hashlib.sha256(raw_token.encode()).hexdigest()
    reset_token = PasswordResetToken(
        token_hash=token_hash,
        user_id=user.id,
        expires_at=datetime.now(UTC) + timedelta(hours=1),
    )
    session.add(reset_token)
    await session.flush()

    await email_svc.send_password_reset_email(user.email, raw_token)
    logger.info("password_reset.token_issued", user_id=str(user.id))


async def confirm_password_reset(
    request: PasswordResetConfirmRequest,
    session: AsyncSession,
) -> None:
    """Validate the reset token and update the user's password."""
    from eusolicit_common.exceptions import ValidationError  # adjust import path if needed

    token_hash = hashlib.sha256(request.token.encode()).hexdigest()
    result = await session.execute(
        select(PasswordResetToken).where(PasswordResetToken.token_hash == token_hash)
    )
    token_record = result.scalar_one_or_none()

    if token_record is None or token_record.consumed_at is not None:
        raise ValidationError("Invalid or expired reset token")

    expires_at = token_record.expires_at
    if expires_at.tzinfo is None:
        expires_at = expires_at.replace(tzinfo=UTC)
    if expires_at < datetime.now(UTC):
        raise ValidationError("Invalid or expired reset token")

    # Load user
    result = await session.execute(
        select(User).where(User.id == token_record.user_id)
    )
    user = result.scalar_one_or_none()
    if user is None:
        raise ValidationError("Invalid or expired reset token")

    # Update password hash
    new_hash = bcrypt.hashpw(
        request.new_password.encode("utf-8"),
        bcrypt.gensalt(rounds=12),
    ).decode("utf-8")
    user.hashed_password = new_hash
    await session.flush()

    # Mark token consumed
    token_record.consumed_at = datetime.now(UTC)
    await session.flush()

    # Bulk-revoke all active refresh tokens for this user
    await session.execute(
        update(RefreshToken)
        .where(RefreshToken.user_id == user.id)
        .values(is_revoked=True)
    )
    await session.flush()
    logger.info("password_reset.confirmed", user_id=str(user.id))
```

**Note on `ValidationError`:** Check what exception type from `eusolicit_common` maps to HTTP 400. If `ValidationError` does not exist, use a local `ValueError` and convert to `HTTPException(400)` in the route handler. The pattern for raising HTTP 400 from service code should match whatever convention Stories 2.2–2.6 use for client errors.

### Route Handler Pattern

```python
# In api/v1/auth.py — add after the /google/callback route

from client_api.schemas.auth import (
    PasswordResetRequestRequest,   # add to existing import
    PasswordResetConfirmRequest,
)

@router.post("/password-reset/request", status_code=200)
async def password_reset_request(
    request: PasswordResetRequestRequest,
    session: Annotated[AsyncSession, Depends(get_db_session)],
    email_svc: Annotated[EmailServiceBase, Depends(get_email_service_dep)],
) -> dict[str, str]:
    """Request a password reset link.

    Always returns 200 regardless of whether the email exists.
    If the email is registered, a reset token is generated and emailed (stubbed).
    """
    await auth_service.request_password_reset(request, session, email_svc)
    return {"message": "If the email is registered, a reset link has been sent"}


@router.post("/password-reset/confirm", status_code=200)
async def password_reset_confirm(
    request: PasswordResetConfirmRequest,
    session: Annotated[AsyncSession, Depends(get_db_session)],
) -> dict[str, str]:
    """Confirm a password reset using the token from the reset email.

    Validates token (not expired, not consumed), enforces password policy,
    updates the password hash, and revokes all existing refresh tokens.

    Returns 400 for invalid/expired/consumed tokens.
    """
    try:
        await auth_service.confirm_password_reset(request, session)
    except SomeValidationError as exc:  # replace with actual exception type
        raise HTTPException(
            status_code=400,
            detail={"error": "invalid_token", "message": str(exc)},
        )
    return {"message": "Password has been reset successfully"}
```

### Token Capture Pattern for Tests

To intercept the raw reset token in tests (which is only logged, not returned by the API), monkey-patch `StubEmailService.send_password_reset_email`:

```python
captured_tokens: list[str] = []

async def capturing_send(to_email: str, token: str) -> None:
    captured_tokens.append(token)

monkeypatch.setattr(
    StubEmailService,
    "send_password_reset_email",
    capturing_send,
)
# ... call POST /auth/password-reset/request ...
raw_token = captured_tokens[0]
```

Alternatively, override the `email_svc` dependency with a custom `CapturingEmailService` subclass that stores tokens in a list attribute. This is the cleaner approach and avoids the need for `monkeypatch` on a non-local attribute.

### Alembic Migration Skeleton

```python
# alembic/versions/006_password_reset_tokens.py
"""Add client.password_reset_tokens table.

Revision ID: 006
Revises: 005
Create Date: 2026-04-07
"""
from __future__ import annotations

import sqlalchemy as sa
from alembic import op

revision = "006"
down_revision = "005"
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("""
        CREATE TABLE client.password_reset_tokens (
            id UUID NOT NULL DEFAULT gen_random_uuid() PRIMARY KEY,
            token_hash VARCHAR(64) NOT NULL,
            user_id UUID NOT NULL REFERENCES client.users(id) ON DELETE CASCADE,
            expires_at TIMESTAMPTZ NOT NULL,
            consumed_at TIMESTAMPTZ,
            created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
            CONSTRAINT uq_password_reset_tokens_token_hash UNIQUE (token_hash)
        )
    """)
    op.execute("""
        CREATE INDEX ix_password_reset_tokens_user_id
            ON client.password_reset_tokens (user_id)
    """)


def downgrade() -> None:
    op.execute("DROP TABLE IF EXISTS client.password_reset_tokens")
```

**Important:** Verify the `down_revision` value matches the actual revision string in `005_uuid_defaults_and_company_reg_number.py` (the `revision` variable at the top of that file).

### File Structure (Modified / New Files)

```
services/client-api/
  alembic/versions/
    006_password_reset_tokens.py               ← NEW: migration
  src/client_api/
    models/
      password_reset_token.py                  ← NEW: PasswordResetToken ORM model
      __init__.py                              ← UPDATE: export PasswordResetToken
    schemas/
      auth.py                                  ← UPDATE: add PasswordResetRequestRequest, PasswordResetConfirmRequest
    services/
      email_service.py                         ← UPDATE: add send_password_reset_email()
      auth_service.py                          ← UPDATE: add request_password_reset(), confirm_password_reset()
    api/
      v1/
        auth.py                                ← UPDATE: add POST /auth/password-reset/request, /confirm
  tests/
    api/
      test_auth_password_reset.py              ← NEW: AC1–AC6 tests + P3 E2E flow
```

### Test Coverage Mapping

| AC | Epic Test ID | Test Class | Description |
|----|-------------|-----------|-------------|
| AC1 | E02-P1-008 | `TestAC1NoEnumeration` | Request with unknown + known email → always 200, identical body |
| AC2 | (part of AC3) | `TestAC2TokenProperties` | DB row: 64-char hash, consumed_at=null, expires_at ≈ now+1h |
| AC3 | E02-P1-009 | `TestAC3SingleUse` | Confirm once → 200; confirm again → 400 invalid_token |
| AC4 | — | `TestAC4PasswordPolicyEnforced` | Weak new_password → 422 from Pydantic |
| AC5 | E02-P1-010 | `TestAC5RefreshTokensRevoked` | Pre-reset refresh token → 401 after reset |
| AC6 | E02-P1-009 | `TestAC6ExpiredToken` | Expired token inserted directly → 400 invalid_token |
| P3 | E02-P3-003 | `TestP3FullFlow` | Full E2E: register → verify → login → request → confirm → re-login |

### Previous Story Intelligence (Story 2.6)

- `auth_service.py` now contains: `register()`, `login()`, `refresh_token()`, `logout()`, `google_login()`, `google_callback()`
- `core/security.py` contains: `create_access_token()`, `create_refresh_token()`, `get_current_user`, `require_role`, `ROLE_HIERARCHY`, `CurrentUser`
- `models/__init__.py` exports: `AuditLog`, `Company`, `CompanyMembership`, `EntityPermission`, `EntityPermissionEnum`, `ESPDProfile`, `RefreshToken`, `Subscription`, `User`, `VerificationToken`
- `schemas/auth.py` contains: `RegisterRequest`, `UserResponse`, `CompanyResponse`, `RegisterResponse`, `LoginRequest`, `LoginResponse`, `CurrentUserResponse`, `RefreshRequest`, `LogoutRequest`
- `dependencies.py` provides: `get_db_session`, `get_email_service_dep`, `get_redis_client`, `get_login_rate_limiter`
- `email_service.py` provides: `EmailServiceBase`, `StubEmailService`, `get_email_service()` with `send_verification_email()` abstract method
- **165 tests pass.** Story 2.7 must not regress any of these.
- Latest Alembic migration revision is `005` (UUID defaults + company registration_number). New migration `006` must set `down_revision = "005"` (verify the exact string in the 005 file).
- The `VerificationToken` model is the closest structural parallel to `PasswordResetToken` — follow its pattern for `__tablename__`, `__table_args__`, `id`, `token_hash`, `user_id`, `expires_at`, `consumed_at`, `created_at`.
- The `update(RefreshToken).where(...).values(is_revoked=True)` bulk-revoke pattern already exists in `auth_service.refresh_token()` (breach detection) — reuse the same SQLAlchemy import and idiom.
- The shared-session test fixture pattern (used in Stories 2.5 and 2.6) is the correct approach for `password_reset_client_and_session` — copy and adapt from `refresh_client_and_session` in `test_auth_refresh.py`.

### Security Notes (from Epic Test Design)

- **E02-R-005** (password security gaps): This story directly mitigates the enumeration risk (`request` always returns 200) and plaintext token storage risk (SHA-256 hash-before-store). The ATDD checklist (E02-P1-008, E02-P1-009) must confirm both.
- **E02-R-003** (token security): All refresh tokens are revoked on reset (AC5). Tests must verify a pre-reset refresh token is rejected post-reset (E02-P1-010).
- The reset token has no CSRF protection requirement (it is consumed server-side; the attacker would need the token from the email). No additional state parameter is needed beyond the opaque URL-safe token itself.

### References

- Epic story definition: [Source: eusolicit-docs/planning-artifacts/epic-02-authentication-identity.md#S02.07]
- Epic test design: [Source: eusolicit-docs/test-artifacts/test-design-epic-02.md] (E02-P1-008, E02-P1-009, E02-P1-010, E02-P3-003)
- VerificationToken model (structural parallel): [Source: eusolicit-app/services/client-api/src/client_api/models/verification_token.py]
- Refresh token bulk-revoke pattern: [Source: eusolicit-app/services/client-api/src/client_api/services/auth_service.py#refresh_token]
- Shared-session test pattern: [Source: eusolicit-app/services/client-api/tests/api/test_auth_refresh.py#refresh_client_and_session]
- Email service stub: [Source: eusolicit-app/services/client-api/src/client_api/services/email_service.py]
- Token hash pattern (SHA-256): [Source: eusolicit-app/services/client-api/src/client_api/services/auth_service.py#register]
- Config settings: [Source: eusolicit-app/services/client-api/src/client_api/config.py]
- Project rules: [Source: eusolicit-docs/project-context.md]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (claude-code)

### Debug Log References

None — implementation was clean on first pass. Migration ran successfully on both dev and test databases.

### Completion Notes List

- Used `ValueError` (not `eusolicit_common.ValidationError`) in the service layer since `ValidationError` maps to HTTP 422, not 400. The route handler catches `ValueError` and raises `HTTPException(400, detail={"error": "invalid_token", "message": "..."})`.
- Pre-written ATDD test file (`test_auth_password_reset.py`) existed in RED PHASE with 17 `@pytest.mark.skip` decorators. All decorators removed; all 17 tests pass green.
- Migration 006 applied to both `eusolicit` (dev) and `eusolicit_test` databases.
- Final test run: **182/182 passing** (17 new + 165 pre-existing). Zero regressions.

### File List

- `src/client_api/models/password_reset_token.py` — NEW: PasswordResetToken ORM model
- `src/client_api/models/__init__.py` — UPDATED: added PasswordResetToken import + `__all__`
- `alembic/versions/006_password_reset_tokens.py` — NEW: migration creating `client.password_reset_tokens`
- `src/client_api/services/email_service.py` — UPDATED: added `send_password_reset_email()` abstract + stub
- `src/client_api/schemas/auth.py` — UPDATED: added `PasswordResetRequestRequest`, `PasswordResetConfirmRequest`
- `src/client_api/services/auth_service.py` — UPDATED: added `request_password_reset()`, `confirm_password_reset()`
- `src/client_api/api/v1/auth.py` — UPDATED: added `POST /password-reset/request`, `POST /password-reset/confirm`
- `tests/api/test_auth_password_reset.py` — UPDATED: removed 17 RED PHASE skip decorators; all tests now pass
