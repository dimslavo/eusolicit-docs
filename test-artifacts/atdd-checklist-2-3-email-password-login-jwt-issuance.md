---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-07'
workflowType: 'testarch-atdd'
storyId: '2-3-email-password-login-jwt-issuance'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/2-3-email-password-login-jwt-issuance.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-02.md'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
  - 'eusolicit-app/services/client-api/tests/api/test_register.py'
---

# ATDD Checklist: Story 2.3 — Email/Password Login & JWT Issuance

## Story Summary

**Epic:** E02 — Authentication & Identity
**Story:** 2.3 — Email/Password Login & JWT Issuance
**Sprint:** 1 | **Points:** N/A
**Risk-Driven:** Yes — E02-R-003 (Token security, Score: 6 — HIGH), E02-R-006 (Rate limit Redis state leakage, Score: 4 — MEDIUM)

**As a** registered company user,
**I want** to POST my credentials to `POST /api/v1/auth/login`,
**So that** I receive a signed RS256 JWT access token and opaque refresh token to authenticate subsequent API requests.

---

## TDD Red Phase (Current)

**Status:** RED — All 11 tests are `@pytest.mark.skip`'d. Zero passes until implementation is complete.

| Test Class | File | Test Functions | AC | Priority | Epic Test ID | Status |
|------------|------|---------------|-----|----------|--------------|--------|
| `TestAC1AC2AC3LoginHappyPath` | `tests/api/test_login.py` | 3 | AC1, AC2, AC3 | P0 | E02-P0-002 | 3 SKIP |
| `TestAC4RefreshTokenDB` | `tests/api/test_login.py` | 1 | AC4 | P0 | E02-P0-002 | 1 SKIP |
| `TestAC5InvalidCredentials` | `tests/api/test_login.py` | 3 | AC5 | P1 | E02-P1-003 | 3 SKIP |
| `TestAC6UnverifiedEmail` | `tests/api/test_login.py` | 1 | AC6 | P1 | E02-P1-004 | 1 SKIP |
| `TestAC7RateLimiting` | `tests/api/test_login.py` | 3 | AC7 | P1 | E02-P1-005 | 3 SKIP |
| **Total** | **1 file** | **11 functions** | **AC1–AC7** | | | **11 SKIP / 0 PASS** |

---

## Acceptance Criteria Coverage

| AC | Description | Test Class(es) | Tests | Priority | Epic Test ID | Coverage |
|----|-------------|-----------------|-------|----------|--------------|----------|
| AC1 | `POST /api/v1/auth/login` with valid credentials returns HTTP 200 with `{ access_token, refresh_token, token_type: "bearer", expires_in: 900 }` | `TestAC1AC2AC3LoginHappyPath` | 1 | P0 | E02-P0-002 | Full — 200 with exact response shape; all 4 fields present and typed correctly |
| AC2 | Access token is RS256-signed with 15-minute expiry (900 seconds) | `TestAC1AC2AC3LoginHappyPath` | 1 | P0 | E02-P0-002 | Full — JWT header checked for alg=RS256; signature verifiable with test public key; exp−iat==900 |
| AC3 | Access token payload contains exactly: sub, company_id, role, exp, iat, jti | `TestAC1AC2AC3LoginHappyPath` | 1 | P0 | E02-P0-002 | Full — sub matches user_id from DB; company_id matches membership; role=='admin'; jti is valid UUID; exp−iat==900 |
| AC4 | Refresh token opaque; SHA-256 hash stored in `client.refresh_tokens` with user_id, expires_at (now+7d), is_revoked=False, family_id | `TestAC4RefreshTokenDB` | 1 | P0 | E02-P0-002 | Full — DB row verified; token_hash == SHA256(raw_token); raw token NOT in DB; expires_at ≈ now+7d (±5min); family_id is valid UUID |
| AC5 | Wrong password or non-existent email → 401 with identical generic `{"error":"unauthorized","message":"Invalid credentials"}` body | `TestAC5InvalidCredentials` | 3 | P1 | E02-P1-003 | Full — wrong-password 401; nonexistent-email 401; byte-identical response bodies (anti-enumeration assertion) |
| AC6 | Login with `email_verified=False` (correct credentials) → 403 with message about verification; credential check runs first | `TestAC6UnverifiedEmail` | 1 | P1 | E02-P1-004 | Full — unverified user returns 403; response body contains 'verified' reference |
| AC7 | After 5 consecutive failures per email in 15-min window → 6th attempt returns 429 + Retry-After header; successful login resets counter | `TestAC7RateLimiting` | 3 | P1 | E02-P1-005 | Full — 5 failures then 429; Retry-After header present with positive integer ≤900; counter reset by success confirmed |

**Total coverage: 11 test functions across 7 acceptance criteria — 100% AC coverage.**

---

## Stack & Generation Mode

| Property | Value |
|----------|-------|
| **Detected stack** | `backend` (Python/FastAPI; pyproject.toml present, no playwright.config.*) |
| **Generation mode** | AI generation (backend stack; no browser recording required) |
| **Test framework** | pytest + pytest-asyncio + httpx AsyncClient |
| **No E2E tests** | Correct — backend-only project; E2E auth flow deferred to Epic 9 (Next.js frontend) |
| **Pact/contract testing** | Not applicable for this story |

---

## Test Strategy

### Priority Mapping

| Test ID | Description | AC | Level | Priority | Risk Link |
|---------|-------------|-----|-------|----------|-----------|
| `test_login_happy_path_returns_200_with_token_structure` | Response shape: access_token, refresh_token, token_type, expires_in | AC1 | API Integration | **P0** | E02-R-003 |
| `test_login_access_token_is_rs256_signed` | JWT header alg=RS256; signature verifiable with public key | AC2 | API Integration | **P0** | E02-R-003 |
| `test_login_access_token_claims` | JWT claims: sub, company_id, role, exp, iat, jti; exp−iat==900 | AC3 | API Integration | **P0** | E02-R-003 |
| `test_login_refresh_token_stored_in_db` | DB row: SHA-256 hash stored; raw not stored; is_revoked=False; expires_at+7d; family_id UUID | AC4 | API Integration | **P0** | E02-R-003 |
| `test_login_wrong_password_returns_401_generic` | Wrong password → 401 + generic error body | AC5 | API Integration | **P1** | E02-R-005 |
| `test_login_nonexistent_email_returns_401_generic` | Non-existent email → 401 + generic error body | AC5 | API Integration | **P1** | E02-R-005 |
| `test_login_identical_response_prevents_enumeration` | Both paths produce identical JSON bodies (anti-enumeration) | AC5 | API Integration | **P1** | E02-R-005 |
| `test_login_unverified_email_returns_403` | email_verified=False + correct credentials → 403 | AC6 | API Integration | **P1** | — |
| `test_login_rate_limit_5_failures_then_429` | 5 failures → 6th = 429 | AC7 | API Integration | **P1** | E02-R-006 |
| `test_login_rate_limit_retry_after_header_present` | 429 response includes Retry-After header (positive int ≤ 900) | AC7 | API Integration | **P1** | E02-R-006 |
| `test_login_success_resets_rate_limit_counter` | 4 fail → 1 success → 5 fail = all 401 (counter reset confirmed) | AC7 | API Integration | **P1** | E02-R-006 |

### Red Phase Design Rationale

All tests assert **expected behavior** (what the implementation MUST produce), not placeholder assertions. They are `@pytest.mark.skip`'d because the implementation does not exist yet. Removing `@pytest.mark.skip` before implementation will produce explicit assertion failures, not import errors or `pass` silences.

---

## Generated Test File

**Path:** `eusolicit-app/services/client-api/tests/api/test_login.py`

**Fixture architecture:**

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `login_client_and_session` | function | Registered+**verified** user; RSA keys injected; test Redis (DB 1); returns `(client, session, credentials)` |
| `unverified_login_client` | function | Registered user with `email_verified=False`; RSA keys + Redis; returns `(client, credentials)` |
| `rate_limit_login_client` | function | Registered+verified user; pre-flushed rate-limit key; returns `(client, credentials)` |

All fixtures are self-contained in `test_login.py` (no changes to `tests/conftest.py` required for RED phase).

**Why self-contained fixtures (not conftest)?**
Pattern is consistent with `test_register.py` (`reg_client_and_session`). Avoids import errors from not-yet-implemented modules (`client_api.core.security`, `client_api.core.rate_limit`, `client_api.dependencies.get_redis_client`) during the RED phase collection step.

---

## conftest.py Updates Required (Before GREEN Phase)

The following changes to `tests/conftest.py` are specified in Task 10.1 and must be completed before removing `@pytest.mark.skip` from the tests:

```python
# 1. Session-scoped RSA key pair fixture
@pytest.fixture(scope="session")
def rsa_test_key_pair() -> tuple[str, str]:
    """Generate a 2048-bit RSA key pair for test JWT signing/verification."""
    from cryptography.hazmat.primitives.asymmetric import rsa
    from cryptography.hazmat.primitives.serialization import (
        Encoding, NoEncryption, PrivateFormat, PublicFormat,
    )
    private_key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
    private_pem = private_key.private_bytes(
        Encoding.PEM, PrivateFormat.TraditionalOpenSSL, NoEncryption()
    ).decode("utf-8")
    public_pem = private_key.public_key().public_bytes(
        Encoding.PEM, PublicFormat.SubjectPublicKeyInfo
    ).decode("utf-8")
    return private_pem, public_pem

# 2. Autouse session fixture: inject RSA keys + clear key caches
@pytest.fixture(scope="session", autouse=True)
def rsa_env_setup(rsa_test_key_pair):
    private_pem, public_pem = rsa_test_key_pair
    import os, client_api.core.security as sec
    os.environ["CLIENT_API_RSA_PRIVATE_KEY"] = private_pem
    os.environ["CLIENT_API_RSA_PUBLIC_KEY"] = public_pem
    sec._private_key_cache = None
    sec._public_key_cache = None
    yield
    sec._private_key_cache = None
    sec._public_key_cache = None

# 3. Test Redis client (DB index 1)
@pytest_asyncio.fixture(scope="session")
async def test_redis_client():
    import redis.asyncio as aioredis
    r = aioredis.Redis.from_url("redis://localhost:6379/1", decode_responses=True)
    yield r
    await r.aclose()

# 4. Update app fixture to override get_redis_client
# Add inside app fixture dependency_overrides block:
#   from client_api.dependencies import get_redis_client
#   fastapi_app.dependency_overrides[get_redis_client] = lambda: test_redis_client

# 5. Autouse function fixture: flush rate-limit keys between tests
@pytest_asyncio.fixture(autouse=True)
async def flush_rate_limit_keys(test_redis_client):
    yield
    keys = await test_redis_client.keys("rate_limit:login:*")
    if keys:
        await test_redis_client.delete(*keys)
```

---

## Implementation Checklist (Tasks Developer Must Complete)

The following tasks from the story spec must be implemented before tests can go GREEN:

- [ ] **Task 1** — Add `redis[asyncio]>=5.0` and `cryptography>=42.0` to `pyproject.toml`
- [ ] **Task 2** — Create `RefreshToken` ORM model (`src/client_api/models/refresh_token.py`)
- [ ] **Task 3** — Create Alembic migration `004_refresh_tokens.py`
- [ ] **Task 4** — Create `src/client_api/core/security.py` with RSA key loaders, `create_access_token()`, `create_refresh_token()`
- [ ] **Task 5** — Add `rsa_private_key` / `rsa_public_key` fields to `ClientApiSettings`
- [ ] **Task 6** — Create `src/client_api/core/rate_limit.py` (`LoginRateLimiter`, `RateLimitExceededError`); update `dependencies.py` with `get_redis_client()` + `get_login_rate_limiter()`
- [ ] **Task 7** — Add `LoginRequest` / `LoginResponse` schemas to `schemas/auth.py`
- [ ] **Task 8** — Implement `auth_service.login()` (rate limit pre-check → credential verify → email check → reset counter → create tokens → store refresh token)
- [ ] **Task 9** — Add `POST /login` route to `api/v1/auth.py` with `RateLimitExceededError` → 429 + Retry-After handler
- [ ] **Task 10.1** — Update `tests/conftest.py` with RSA + Redis fixtures (see above)
- [ ] **Task 10.2** — Remove `@pytest.mark.skip` from all 11 tests in `test_login.py`

---

## Next Steps (TDD Green Phase)

After all implementation tasks are complete:

1. Remove all `@pytest.mark.skip(...)` decorators from `tests/api/test_login.py`
2. Apply migration: `make reset-db && make up` (or equivalent)
3. Run tests:
   ```bash
   cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/api/test_login.py -v
   ```
4. Verify all 11 tests **PASS** (green phase)
5. Run full auth suite to verify no regressions:
   ```bash
   cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/api/ -v
   ```
6. If any test fails: fix the implementation (feature bug) OR fix the test (test bug — verify against AC)
7. Commit passing tests

---

## Risks & Assumptions

| Risk | Impact | Mitigation |
|------|--------|------------|
| **E02-R-003** (token security): RS256 key cache not cleared between test sessions → test using stale/wrong key | Signature verification fails | Fixture teardown clears `security_module._private_key_cache` and `_public_key_cache`; confirmed in all three local fixtures |
| **E02-R-006** (Redis state leakage): rate-limit counter persists from prior tests → false 429 or masked failures | AC7 tests flaky | `rate_limit_login_client` pre-flushes key before yield AND post-flushes on teardown; unique email per fixture invocation |
| `client.refresh_tokens` table missing | `TestAC4RefreshTokenDB` fails with DB error | Covered by Task 3 (migration 004); GREEN phase requires migration applied |
| `session.flush()` vs `session.commit()` in `auth_service.login()` | Refresh token row not visible in test session | Story spec correctly mandates `flush()` not `commit()`; test session shares the same transaction so flushed data is visible |
| Multiple active memberships (edge case in `test_login_access_token_claims`) | JWT company_id assertion flaky | Query uses `LIMIT 1` with `accepted_at IS NOT NULL` — consistent with production behavior; test uses freshly registered user with exactly one membership |

---

## Quality Gate

Before Story 2.3 is marked **Done**:

- [ ] All 11 tests in `test_login.py` pass (GREEN phase)
- [ ] No regressions in `test_register.py` (11 existing tests still pass)
- [ ] Migration 004 applied cleanly; `003→004` upgrade + downgrade verified
- [ ] `client.refresh_tokens` table exists with correct schema (via `test_002_migration.py` or equivalent)
- [ ] Branch coverage ≥90% on `core/security.py`, `core/rate_limit.py`, `services/auth_service.py` (login path)
