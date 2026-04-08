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
storyKey: 2-6-google-oauth2-social-login
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/2-6-google-oauth2-social-login.md
  - eusolicit-docs/test-artifacts/test-design-epic-02.md
  - eusolicit-app/services/client-api/tests/api/test_auth_refresh.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit-app/pyproject.toml
---

# ATDD Checklist: Story 2.6 — Google OAuth2 Social Login

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — tests written before implementation
**Story Status:** ready-for-dev

---

## Preflight Summary

| Item | Value |
|------|-------|
| Story Key | `2-6-google-oauth2-social-login` |
| Detected Stack | `backend` (Python/FastAPI — `client-api` service) |
| Generation Mode | AI Generation (backend story — no browser recording needed) |
| Execution Mode | Sequential |
| Test Framework | pytest + pytest-asyncio (`asyncio_mode = "auto"`) |
| Test File | `eusolicit-app/services/client-api/tests/api/test_auth_google.py` |
| Total Tests Generated | 15 |
| TDD Phase | 🔴 RED (all tests `@pytest.mark.skip`) |

### Prerequisites Verified

- [x] Story has clear acceptance criteria (AC1–AC7)
- [x] `conftest.py` provides `client_api_session_factory`, `test_redis_client`, `rsa_env_setup`
- [x] Shared-session pattern established in `test_auth_refresh.py:refresh_client_and_session`
- [x] `client.users.google_sub` column exists (VARCHAR 255, unique, nullable) — created in Story 2.1
- [x] `client.company_memberships.accepted_at` column exists — created in Story 2.1
- [x] `pytest.ini_options` defines `integration` marker (root `pyproject.toml`)
- [ ] `authlib>=1.3` added to `services/client-api/pyproject.toml` — **pending Task 1.1**
- [ ] `itsdangerous>=2.1` added to `services/client-api/pyproject.toml` — **pending Task 1.3**
- [ ] `core/oauth.py` created with `OAuth` singleton — **pending Task 2**
- [ ] `SessionMiddleware` registered in `main.py` — **pending Task 2**
- [ ] `google_client_id`, `google_client_secret`, `frontend_url`, `oauth_secret_key` added to `config.py` — **pending Task 1.2**

---

## TDD Red Phase Status

🔴 **15 failing tests generated** — all decorated with:

```python
@pytest.mark.skip(reason="RED PHASE: <feature> not yet implemented")
```

Tests assert **expected behavior** before implementation. Remove `@pytest.mark.skip`
after each AC is implemented and verified passing.

---

## Test Strategy

### Test Level Selection

All tests are **API integration tests** (`pytest.mark.integration`):

- Backend service under test — no browser interaction
- Use `httpx.AsyncClient` with `ASGITransport` (in-process, no network)
- All Google OAuth methods mocked via `unittest.mock.AsyncMock` + `patch`
- Shared-session pattern ensures DB state visible across requests in same test

### Priority Mapping

| AC | Epic ID | Priority | Tests | Rationale |
|----|---------|----------|-------|-----------|
| AC1 | E02-P3-002 | P1 | 2 | OAuth redirect entry point; security baseline |
| AC2 | E02-P1-011 | P1 | 2 | CSRF state validation; E02-R-004 mitigation |
| AC3 | E02-P1-012 | P1 | 2 | New user creation path; account integrity |
| AC4 | E02-P1-013 | P1 | 2 | Account linking + conflict guard; data integrity |
| AC5 | (within AC3/4) | P1 | 2 | Token structure parity with email/password flow |
| AC6 | — | P1 | 2 | No-company redirect; UX continuity |
| AC7 | — | P1 | 3 | Error resilience; never 500 philosophy |

### Mock Strategy

Google OAuth is an external dependency that cannot be used in tests directly.
Two authlib methods are patched per fixture instance:

```python
# Default success mock (in fixture)
patch("client_api.core.oauth.oauth.google.authorize_redirect", ...)
patch("client_api.core.oauth.oauth.google.authorize_access_token", ...)

# Per-test error overrides (for AC2, AC7)
patch("client_api.core.oauth.oauth.google.authorize_access_token",
      new=AsyncMock(side_effect=OAuthError(...)))
```

The mock target path `client_api.core.oauth.oauth.google.*` assumes
the `oauth` singleton lives in `client_api/core/oauth.py` (recommended
approach from story Dev Notes to avoid circular imports).

---

## Test Coverage by Acceptance Criterion

### AC1 — `GET /api/v1/auth/google` redirects to Google

| Test | Priority | Skip Reason |
|------|----------|-------------|
| `TestAC1GoogleRedirect::test_google_login_returns_302_to_google` | P1 | `GET /api/v1/auth/google not yet implemented` |
| `TestAC1GoogleRedirect::test_google_login_redirect_includes_state_param` | P1 | `GET /api/v1/auth/google not yet implemented` |

**Assertions:**
- Response status = 302
- `Location` header contains `accounts.google.com`
- `Location` header contains `state=` (CSRF parameter)

---

### AC2 — State validation rejects invalid/missing state

| Test | Priority | Skip Reason |
|------|----------|-------------|
| `TestAC2InvalidState::test_missing_state_redirects_to_error` | P1 | `GET /api/v1/auth/google/callback not yet implemented` |
| `TestAC2InvalidState::test_mismatched_state_redirects_to_error_not_500` | P1 | `GET /api/v1/auth/google/callback not yet implemented` |

**Assertions:**
- Status = 302 (not 500)
- `Location` contains `/auth/error`
- `Location` contains `invalid_state` or `oauth_error` as reason

**Mock:** `authorize_access_token` raises `OAuthError("mismatched_state", ...)`

---

### AC3 — New Google user: account created correctly

| Test | Priority | Skip Reason |
|------|----------|-------------|
| `TestAC3NewGoogleUser::test_new_google_user_row_created_with_correct_fields` | P1 | `Google OAuth new-user creation not yet implemented` |
| `TestAC3NewGoogleUser::test_new_google_user_redirected_with_needs_company` | P1 | `Google OAuth new-user creation not yet implemented` |

**Assertions:**
- `client.users` row created with `google_sub = mock_userinfo["sub"]`
- `hashed_password = NULL`
- `email_verified = TRUE`
- Redirect `Location` → `/auth/callback?needs_company=true`

---

### AC4 — Existing email user: google_sub linked, no duplicate

| Test | Priority | Skip Reason |
|------|----------|-------------|
| `TestAC4AccountLinking::test_existing_email_user_gets_google_sub_linked` | P1 | `Google OAuth account linking not yet implemented` |
| `TestAC4AccountLinking::test_account_conflict_different_google_sub_redirects_to_error` | P1 | `Google OAuth account conflict guard not yet implemented` |

**Assertions (linking):**
- Pre-seeded user count = 1 before callback
- Post-callback user count still = 1 (no duplicate)
- `google_sub` updated to `mock_userinfo["sub"]`
- `hashed_password` preserved (not nulled)

**Assertions (conflict):**
- Pre-seeded user has `google_sub = existing_sub` (different from mock)
- Post-callback redirect → `/auth/error?reason=account_conflict`

---

### AC5 — Token structure matches email/password tokens

| Test | Priority | Skip Reason |
|------|----------|-------------|
| `TestAC5TokenStructure::test_google_oauth_tokens_present_in_redirect_url` | P1 | `Google OAuth token issuance for company-linked user not yet implemented` |
| `TestAC5TokenStructure::test_google_oauth_access_token_is_rs256_jwt` | P1 | `Google OAuth RS256-signed access token not yet implemented` |

**Setup:** `google_user_with_company_and_client` fixture pre-seeds user + accepted company membership.

**Assertions:**
- Redirect → `/auth/callback` (not error, not needs_company)
- `access_token` present in URL params
- `refresh_token` present in URL params
- `expires_in = 900`
- JWT header: `alg = RS256`, `typ = JWT`

---

### AC6 — No company membership → needs_company redirect

| Test | Priority | Skip Reason |
|------|----------|-------------|
| `TestAC6NoCompany::test_new_google_user_no_company_redirect_has_needs_company_flag` | P1 | `Google OAuth no-company redirect not yet implemented` |
| `TestAC6NoCompany::test_no_company_redirect_is_not_error_path` | P1 | `Google OAuth no-company redirect not yet implemented` |

**Assertions:**
- Redirect → `/auth/callback` (not `/auth/error`)
- `needs_company=true` in URL params

---

### AC7 — OAuth errors redirect, never 500

| Test | Priority | Skip Reason |
|------|----------|-------------|
| `TestAC7OAuthError::test_invalid_code_redirects_to_oauth_error` | P1 | `Google OAuth error handling not yet implemented` |
| `TestAC7OAuthError::test_oauth_error_access_denied_redirects_not_500` | P1 | `Google OAuth error handling not yet implemented` |
| `TestAC7OAuthError::test_missing_userinfo_in_token_response_redirects_to_error` | P1 | `Google OAuth error handling not yet implemented` |

**Mocks:**
- `OAuthError("invalid_grant", "Code expired")` → `reason=oauth_error`
- `OAuthError("access_denied", "User denied")` → redirect, not 500
- `authorize_access_token` returns `{}` (no userinfo key) → `reason=missing_userinfo` or `oauth_error`

---

## Fixture Infrastructure

### `google_mock_client_and_session` (module-level, function-scoped)

Follows exact shared-session pattern from `refresh_client_and_session`:

```
client_api_session_factory → AsyncSession (shared)
    ├── fastapi_app.dependency_overrides[get_db_session] → yields shared session
    ├── fastapi_app.dependency_overrides[get_email_service_dep] → StubEmailService
    ├── fastapi_app.dependency_overrides[get_redis_client] → test_redis_client
    └── httpx.AsyncClient(follow_redirects=False)
        ├── patch("...oauth.google.authorize_redirect") → 302 to fake Google URL
        └── patch("...oauth.google.authorize_access_token") → {"userinfo": mock_userinfo}
```

Teardown: `session.rollback()` in `finally` block.

### `google_user_with_company_and_client` (module-level, function-scoped)

Extends `google_mock_client_and_session`:
- Pre-seeds `client.users` row with `google_sub`, `email_verified=True`, `hashed_password=NULL`
- Pre-seeds `client.companies` row
- Pre-seeds `client.company_memberships` row with `accepted_at=NOW()`
- Used only by `TestAC5TokenStructure` tests

---

## Implementation Guard Checklist

Before removing `@pytest.mark.skip` decorators, verify:

### Task 1 (Config & Dependencies)
- [ ] `authlib>=1.3` in `services/client-api/pyproject.toml`
- [ ] `itsdangerous>=2.1` in `services/client-api/pyproject.toml`
- [ ] `google_client_id`, `google_client_secret`, `frontend_url`, `oauth_secret_key` in `config.py`

### Task 2 (Middleware & OAuth Singleton)
- [ ] `SessionMiddleware` registered in `main.py`
- [ ] `client_api/core/oauth.py` created with module-level `oauth = create_oauth_client()`
- [ ] OAuth singleton registered with `server_metadata_url=".../openid-configuration"` and scope `openid email profile`

### Task 4 (Service Functions)
- [ ] `google_login(request)` implemented in `auth_service.py`
- [ ] `google_callback(request, session)` implemented with:
  - [ ] `authorize_access_token` call wrapped in try/except → error redirect
  - [ ] `userinfo` extraction + null check → `missing_userinfo` redirect
  - [ ] User lookup by `google_sub` → by email → new user creation (AC3)
  - [ ] Account linking guard for conflicting `google_sub` (AC4 conflict)
  - [ ] Company membership check → `needs_company=true` redirect (AC6)
  - [ ] Full token issuance via `create_access_token` + `create_refresh_token` (AC5)

### Task 5 (Routes)
- [ ] `GET /auth/google` route added to `api/v1/auth.py`
- [ ] `GET /auth/google/callback` route added to `api/v1/auth.py`

### Mock Target Alignment
- [ ] OAuth singleton in `client_api/core/oauth.py` (tests patch `client_api.core.oauth.oauth.google.*`)
- [ ] If placed in `main.py` instead, update patch targets in all test classes

---

## TDD Green Phase Instructions

After implementing each task:

1. Remove `@pytest.mark.skip` from the relevant test class
2. Run tests: `cd eusolicit-app && python -m pytest services/client-api/tests/api/test_auth_google.py -v --tb=short`
3. Verify tests PASS (green phase)
4. If a test fails:
   - **Implementation bug** → fix `auth_service.py` / `api/v1/auth.py`
   - **Test assumption wrong** → update the test to match the agreed contract
5. Run full suite to prevent regressions: `python -m pytest services/client-api/tests/ -v`
6. Commit passing tests

### Recommended Remove-Skip Order (by task completion)

| When | Remove Skip From |
|------|-----------------|
| Task 5 complete (routes exist) | `TestAC1GoogleRedirect` |
| Task 4.2.1 complete (OAuthError catch) | `TestAC2InvalidState`, `TestAC7OAuthError` |
| Task 4.2.3–4.2.4 complete (new user creation) | `TestAC3NewGoogleUser` |
| Task 4.2.5 complete (account linking) | `TestAC4AccountLinking` |
| Task 4.2.6–4.2.7 complete (company check + tokens) | `TestAC5TokenStructure`, `TestAC6NoCompany` |

---

## Risk Notes

| Risk | Mitigation |
|------|-----------|
| E02-R-004: CSRF state not validated | AC2 tests verify `OAuthError` is caught and redirected correctly |
| Circular import (`oauth` in `main.py`) | Tests patch `client_api.core.oauth.*` — requires `core/oauth.py` approach |
| Token redirect exposes tokens in URL | MVP acceptable; logged as tech debt (see story Dev Notes §6) |
| `SessionMiddleware` secret_key in tests | Tests use mock at the authlib layer — session cookie not exercised in unit tests |
| `authlib` not yet installed | Tests will fail to import until `authlib>=1.3` added to `pyproject.toml` |

---

## Generated Files

| File | Type | Status |
|------|------|--------|
| `eusolicit-app/services/client-api/tests/api/test_auth_google.py` | Test file | ✅ Created (RED phase) |
| `eusolicit-docs/test-artifacts/atdd-checklist-2-6-google-oauth2-social-login.md` | This document | ✅ Created |
