---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-07'
workflowType: 'testarch-atdd'
storyId: '2-2-email-password-registration'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/2-2-email-password-registration.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-02.md'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
  - 'eusolicit-app/services/client-api/pyproject.toml'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-2-1-database-schema-auth-identity-tables.md'
---

# ATDD Checklist: Story 2.2 — Email/Password Registration

## Story Summary

**Epic:** E02 — Authentication & Identity
**Story:** 2.2 — Email/Password Registration
**Sprint:** 1 | **Points:** N/A
**Risk-Driven:** Yes — E02-R-005 (Password security gaps, Score: 3 — MEDIUM)

**As a** new company representative,
**I want** to register my company and create an admin account via `POST /api/v1/auth/register`,
**So that** my company and I have a platform identity for all subsequent authenticated operations.

---

## TDD Red Phase (Current)

**Status:** RED — All 11 tests are `@pytest.mark.skip`'d. Zero passes until implementation is complete.

| Test Class | File | Test Functions | AC | Priority | Epic Test ID | Status |
|------------|------|---------------|-----|----------|--------------|--------|
| `TestAC1AC2AC7RegisterHappyPath` | `tests/api/test_register.py` | 3 | AC1, AC2, AC7 | P0 | E02-P0-001 | 3 SKIP |
| `TestAC3PasswordHashing` | `tests/api/test_register.py` | 1 | AC3 | P2 | — | 1 SKIP |
| `TestAC6VerificationToken` | `tests/api/test_register.py` | 2 | AC6 | P2 | — | 2 SKIP |
| `TestAC4DuplicateEmail` | `tests/api/test_register.py` | 1 | AC4 | P1 | E02-P1-001 | 1 SKIP |
| `TestAC5WeakPassword` | `tests/api/test_register.py` | 4 | AC5 | P1 | E02-P1-002 | 4 SKIP |
| **Total** | **1 file** | **11 functions** | **AC1–AC7** | | | **11 SKIP / 0 PASS** |

---

## Acceptance Criteria Coverage

| AC | Description | Test Class(es) | Tests | Priority | Epic Test ID | Coverage |
|----|-------------|-----------------|-------|----------|--------------|----------|
| AC1 | `POST /api/v1/auth/register` creates User + Company in one atomic transaction; returns 201 with `user` + `company` | `TestAC1AC2AC7RegisterHappyPath` | 2 | P0 | E02-P0-001 | Full — 201 with correct body; DB membership confirms single-transaction atomicity |
| AC2 | First user assigned `admin` role in `company_memberships` with `accepted_at = now(UTC)` in same transaction | `TestAC1AC2AC7RegisterHappyPath` | 1 | P0 | E02-P0-001 | Full — DB query verifies role='admin' and accepted_at IS NOT NULL |
| AC3 | Password hashed with bcrypt cost=12; plaintext never persisted | `TestAC3PasswordHashing` | 1 | P2 | E02-R-005 | Full — DB query checks `$2b$12$` prefix; plaintext not in hash |
| AC4 | Duplicate email → 409 Conflict; existing records unchanged | `TestAC4DuplicateEmail` | 1 | P1 | E02-P1-001 | Full — second POST with same email → 409; only one user row in DB |
| AC5 | Weak passwords rejected with 422: <8 chars, all-lowercase, no-digit; valid → 201 | `TestAC5WeakPassword` | 4 | P1 | E02-P1-002 | Full — three 422 boundary failures + one 201 positive boundary |
| AC6 | Verification token: URL-safe 32-byte random; SHA-256 hashed; stored in `client.verification_tokens`; expires_at = now + 24h; raw token to stub email service | `TestAC6VerificationToken` | 2 | P2 | — | Full — DB row with 64-char hex token_hash; expires_at ≈ now+24h (±5 min) |
| AC7 | Response excludes `hashed_password`; includes `id`, `email`, `full_name`, `is_active`, `email_verified`, `created_at` for user; `id`, `name`, `created_at` for company | `TestAC1AC2AC7RegisterHappyPath` | 2 | P0 | E02-P0-001 | Full — response shape verified; hashed_password explicitly absent |

**Total coverage: 11 test functions across 7 acceptance criteria — 100% AC coverage.**

---

## Epic-Level Test ID Mapping

| Epic Test ID | Description | Test Class | Tests |
|--------------|-------------|------------|-------|
| **E02-P0-001** | Registration creates company + admin user in single transaction; `user` + `company` in response; `hashed_password` absent; `company_memberships` row with role=admin in DB | `TestAC1AC2AC7RegisterHappyPath` | 3 |
| **E02-P1-001** | Duplicate email returns 409 Conflict; existing user unchanged | `TestAC4DuplicateEmail` | 1 |
| **E02-P1-002** | Weak password boundary tests → 422 (7-char, all-lowercase, no-digit fail; valid passes) | `TestAC5WeakPassword` | 4 |

**Quality gate:** 100% pass rate for E02-P0-001, E02-P1-001, E02-P1-002.

---

## Risk Mitigation Verification

### E02-R-005: Password Security Gaps (Score: 3 — MEDIUM)

| Mitigation | Test Coverage | Status |
|------------|--------------|--------|
| bcrypt cost=12 (not lower) | `TestAC3PasswordHashing::test_register_stores_bcrypt_hashed_password_with_cost_12` | RED |
| Plaintext password not stored in DB | `TestAC3PasswordHashing::test_register_stores_bcrypt_hashed_password_with_cost_12` | RED |
| Weak password (< 8 chars) rejected | `TestAC5WeakPassword::test_register_weak_password_too_short_returns_422` | RED |
| Weak password (no uppercase) rejected | `TestAC5WeakPassword::test_register_weak_password_no_uppercase_returns_422` | RED |
| Weak password (no digit) rejected | `TestAC5WeakPassword::test_register_weak_password_no_digit_returns_422` | RED |
| Verification token hash (not plaintext) stored in DB | `TestAC6VerificationToken::test_register_stores_verification_token_hash_in_db` | RED |

---

## Test Strategy

### Stack Detection

- **Detected stack:** `backend` (Python/FastAPI, pytest/pytest-asyncio — no frontend indicators)
- **Story scope:** API endpoint (`POST /api/v1/auth/register`) + DB state verification
- **Test framework:** pytest + pytest-asyncio + httpx (ASGITransport) + sqlalchemy[asyncio] + asyncpg
- **No E2E/browser tests:** Pure backend story

### Generation Mode

- **Mode:** AI generation (backend project; no browser recording needed)

### Test Level Selection

| Level | Justification | Count |
|-------|--------------|-------|
| API/Integration (httpx + DB) | Full endpoint contract + transaction verification: status codes, response shape, DB state, bcrypt hash, token hash, duplicate handling | 7 |
| API (httpx only) | Password validation rejection: 422/201 status codes, no DB verification needed | 4 |

### Priority Distribution

| Priority | Tests | Criteria |
|----------|-------|----------|
| **P0** | 3 | Core registration transaction and response shape — blocks all downstream Epic 2 stories |
| **P1** | 5 | Conflict + boundary tests — important auth flows with medium risk |
| **P2** | 3 | bcrypt security + token storage — technical correctness, no workaround |

---

## Failing Tests Created (RED Phase)

### API + Integration Tests (11 test functions in 5 classes)

**File:** `eusolicit-app/services/client-api/tests/api/test_register.py`

---

#### AC1 + AC2 + AC7: Register Happy Path (3 tests)

- **`TestAC1AC2AC7RegisterHappyPath::test_register_happy_path_returns_201_with_user_and_company`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Epic: E02-P0-001 | Priority: P0 | ACs: AC1, AC7
  - Verifies: POST → 201; `body.user` has `{id, email, full_name, is_active=True, email_verified=False, created_at}`; `body.company` has `{id, name, created_at}`

- **`TestAC1AC2AC7RegisterHappyPath::test_register_response_excludes_hashed_password`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Epic: E02-P0-001 | Priority: P0 | AC: AC7
  - Verifies: `hashed_password` absent from response root AND from `body.user`; plaintext `password` also absent from `body.user`

- **`TestAC1AC2AC7RegisterHappyPath::test_register_creates_admin_membership_in_db`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Epic: E02-P0-001 | Priority: P0 | ACs: AC1, AC2
  - Verifies: `client.company_memberships` row exists for (user_id, company_id); `role = 'admin'`; `accepted_at IS NOT NULL`

---

#### AC3: Password Hashing (1 test)

- **`TestAC3PasswordHashing::test_register_stores_bcrypt_hashed_password_with_cost_12`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Priority: P2 | Risk: E02-R-005
  - Verifies: `client.users.hashed_password` is NOT NULL; starts with `$2b$12$`; plaintext password is NOT present in the hash string

---

#### AC6: Verification Token (2 tests)

- **`TestAC6VerificationToken::test_register_stores_verification_token_hash_in_db`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Priority: P2 | AC: AC6
  - Verifies: `client.verification_tokens` row exists for user; `token_hash` is exactly 64 characters; all hex characters (SHA-256 hexdigest); `consumed_at IS NULL`

- **`TestAC6VerificationToken::test_register_verification_token_expires_in_24_hours`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Priority: P2 | AC: AC6
  - Verifies: `expires_at AT TIME ZONE 'UTC'` is within range `[now+24h-5min, now+24h+5min]`

---

#### AC4: Duplicate Email (1 test)

- **`TestAC4DuplicateEmail::test_register_duplicate_email_returns_409`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Epic: E02-P1-001 | Priority: P1 | AC: AC4
  - Verifies: First POST → 201; second POST (same email) → 409; DB shows exactly one user row with that email; user.id = original id

---

#### AC5: Weak Password Boundaries (4 tests)

- **`TestAC5WeakPassword::test_register_weak_password_too_short_returns_422`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Epic: E02-P1-002 | Priority: P1 | AC: AC5
  - Input: `password="Pass1Ab"` (exactly 7 chars — one below minimum)
  - Verifies: 422 Unprocessable Entity

- **`TestAC5WeakPassword::test_register_weak_password_no_uppercase_returns_422`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Epic: E02-P1-002 | Priority: P1 | AC: AC5
  - Input: `password="secure1abc"` (≥8 chars, has digit, all lowercase)
  - Verifies: 422 Unprocessable Entity

- **`TestAC5WeakPassword::test_register_weak_password_no_digit_returns_422`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Epic: E02-P1-002 | Priority: P1 | AC: AC5
  - Input: `password="SecurePass"` (≥8 chars, has uppercase, no digit)
  - Verifies: 422 Unprocessable Entity

- **`TestAC5WeakPassword::test_register_valid_boundary_password_returns_201`**
  - Status: RED — `@pytest.mark.skip` | endpoint not implemented
  - Epic: E02-P1-002 | Priority: P1 | AC: AC5 (positive boundary)
  - Input: `password="Secure1!"` (exactly 8 chars, has `S` uppercase, has `1` digit)
  - Verifies: 201 Created — boundary password meeting all three rules accepted

---

## Fixture Architecture

### `reg_client_and_session` (local to `test_register.py`)

| Attribute | Value |
|-----------|-------|
| Scope | `function` (default) |
| Yields | `(httpx.AsyncClient, AsyncSession)` |
| Session sharing | App's `get_db_session` override reuses the same `AsyncSession` |
| Isolation | `session.begin()` context rolls back on exit — no persistent state |
| Purpose | Enables in-transaction DB assertions on flushed (not committed) data |

**Requires Task 10.1:** The fixture imports `client_api.main` and `client_api.dependencies` — these modules must exist before the fixture can run (skip marks prevent ImportError in RED phase).

### `client_api_session_factory` (from `conftest.py`)

Pre-existing session-scoped fixture — `async_sessionmaker` for the `client_api_role` database connection. Used by `reg_client_and_session`.

---

## Data Factories

**`make_register_payload(**overrides)`** (local to `test_register.py`)

Generates a valid registration payload with a unique `uuid.uuid4().hex[:8]`-suffixed email to prevent collisions in parallel test runs. Default password `"SecurePass1"` satisfies all AC5 rules (≥8 chars, ≥1 uppercase, ≥1 digit).

Overrides used in tests:
- `password=` — for AC5 boundary and AC3 tests
- `email=` — for AC4 duplicate email test (intentionally reused)
- `company_name=`, `full_name=` — for AC4 second request (intentionally different)

---

## Mock Requirements

| Component | Approach | Details |
|-----------|----------|---------|
| Database | Real PostgreSQL via `client_api_role` | `eusolicit_test` DB; Docker Compose required |
| Email service | `StubEmailService` injected via `dependency_overrides` | Logs `verification_token_generated` via structlog; does NOT send SMTP |
| No other mocks | — | No Redis, no external APIs in Story 2.2 |

---

## Required data-testid Attributes

N/A — No UI components in this story.

---

## Dependencies Before Running Tests

| Requirement | Source | Status |
|-------------|--------|--------|
| `bcrypt>=4.0` in `[project] dependencies` | Task 1.1 | Must add to `pyproject.toml` |
| `email-validator>=2.0` in `[project] dependencies` | Task 1.2 | Must add to `pyproject.toml` |
| `VerificationToken` ORM model (`verification_token.py`) | Task 2 | Must create |
| Alembic migration `003_verification_tokens.py` applied | Task 3 | Must create + `make reset-db && make up` |
| Pydantic schemas (`auth.py`) | Task 4 | Must create |
| `StubEmailService` in `services/email_service.py` | Task 5 | Must create |
| `auth_service.py` with `register()` using `flush()` not `commit()` | Task 6 | Must create |
| `config.py` + `dependencies.py` | Task 7 | Must create |
| Auth router `api/v1/auth.py` wired to `POST /register` | Task 8 | Must create |
| `main.py` updated with `api_v1_router` + `register_exception_handlers` | Task 9 | Must update |
| `conftest.py` `app` fixture updated (Task 10.1) | Task 10.1 | Must update before removing skip marks |
| PostgreSQL running with `eusolicit_test` DB + migration 003 applied | `docker-compose.yml` | `make reset-db && make up` |
| Migration 002 (`002_auth_identity_tables.py`) applied (Story 2.1) | Story 2.1 | Pre-requisite: must be GREEN |

---

## Implementation Checklist

Tasks that unlock each test class (remove `@pytest.mark.skip` when all listed tasks are done):

### TestAC5WeakPassword (4 tests) — Unlock after Tasks 1, 4, 8, 9, 10.1

Tests that will pass when complete:
- [ ] `test_register_weak_password_too_short_returns_422` — Pydantic `@field_validator` raises ValueError → FastAPI 422
- [ ] `test_register_weak_password_no_uppercase_returns_422` — same validator
- [ ] `test_register_weak_password_no_digit_returns_422` — same validator
- [ ] `test_register_valid_boundary_password_returns_201` — 8-char password accepted

### TestAC4DuplicateEmail (1 test) — Unlock after Tasks 1, 4, 5, 6, 7, 8, 9, 10.1 + Migration 003

Tests that will pass when complete:
- [ ] `test_register_duplicate_email_returns_409` — `auth_service.register()` queries email before insert; raises `ConflictError` → 409

### TestAC1AC2AC7RegisterHappyPath (3 tests) — Unlock after ALL Tasks 1–10.1 + Migration 003

Tests that will pass when complete:
- [ ] `test_register_happy_path_returns_201_with_user_and_company` — full happy path: 201 + correct response shape
- [ ] `test_register_response_excludes_hashed_password` — `UserResponse` model has no `hashed_password` field
- [ ] `test_register_creates_admin_membership_in_db` — `CompanyMembership(role=admin, accepted_at=now())` flushed in same transaction

### TestAC3PasswordHashing (1 test) — Unlock after ALL Tasks 1–10.1 + Migration 003

Tests that will pass when complete:
- [ ] `test_register_stores_bcrypt_hashed_password_with_cost_12` — `bcrypt.hashpw(..., gensalt(rounds=12))` produces `$2b$12$` prefix

### TestAC6VerificationToken (2 tests) — Unlock after ALL Tasks 1–10.1 + Migration 003

Tests that will pass when complete:
- [ ] `test_register_stores_verification_token_hash_in_db` — `secrets.token_urlsafe(32)` → SHA-256 hex → `verification_tokens.token_hash`
- [ ] `test_register_verification_token_expires_in_24_hours` — `expires_at = datetime.now(UTC) + timedelta(hours=24)`

---

## Next Steps (TDD Green Phase)

After completing implementation Tasks 1–10:

```bash
# 1. Apply migration 003
cd eusolicit-app && make reset-db && make up

# 2. Run just the new tests to verify RED → GREEN transition
.venv/bin/python -m pytest services/client-api/tests/api/test_register.py -v

# 3. Verify no regressions on Story 2.1 tests
.venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v

# 4. Run full client-api test suite
.venv/bin/python -m pytest services/client-api/tests/ -v
```

**Green phase checklist:**
- [ ] Remove `@pytest.mark.skip` from all 11 test functions in `test_register.py`
- [ ] All 11 tests PASS (0 failures, 0 errors)
- [ ] Story 2.1 migration tests still PASS (no regression)
- [ ] Branch coverage ≥ 90% on `auth_service.py` and `api/v1/auth.py`

---

## Notes and Assumptions

1. **`auth_service.register()` uses `flush()` not `commit()`**: As noted in Story 2.2 dev notes (DB Dependency Pattern section), the service must use `await session.flush()` for intermediate operations and NOT `await session.commit()`. The dependency wrapper (and test override) handles the transaction commit/rollback. The `reg_client_and_session` fixture depends on this.

2. **Shared session pattern**: The `reg_client_and_session` fixture overrides `get_db_session` to yield the test's own session. This allows the test to see flushed (uncommitted) rows from `auth_service.register()` in the same transaction — enabling in-transaction DB verification without committing to the database.

3. **`consumed_at` column**: Tests verify `consumed_at IS NULL` for newly created tokens. The `VerificationToken` model must include `consumed_at: datetime | None` (nullable).

4. **Token hash hex format**: The test checks `all(c in "0123456789abcdef" for c in token_hash)` — `hashlib.sha256(...).hexdigest()` always produces lowercase hex, so this assertion is correct.

5. **Verification token not in response**: AC6 specifies the raw token is passed to the stub email service (which logs it). The test does NOT verify what was logged — only that the hash is stored correctly in the DB. Verifying the logged token is out of scope for this story.
