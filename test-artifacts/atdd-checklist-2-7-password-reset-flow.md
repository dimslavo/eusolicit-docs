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
storyId: 2-7-password-reset-flow
workflowType: atdd
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/2-7-password-reset-flow.md
  - eusolicit-docs/test-artifacts/test-design-epic-02.md
  - eusolicit-app/services/client-api/tests/api/test_auth_refresh.py
  - eusolicit-app/services/client-api/tests/api/test_auth_google.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/src/client_api/services/email_service.py
  - eusolicit-app/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 2.7 — Password Reset Flow

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED (tests are failing — feature not implemented yet)
**Story:** `eusolicit-docs/implementation-artifacts/2-7-password-reset-flow.md`
**Epic Test Design:** `eusolicit-docs/test-artifacts/test-design-epic-02.md`

---

## Step 1: Preflight & Context

### Stack Detection

| Indicator | Found | Location |
|-----------|-------|----------|
| `package.json` (frontend) | ✅ | `eusolicit-app/frontend/package.json` |
| `playwright.config.ts` | ✅ | `eusolicit-app/playwright.config.ts` |
| `pyproject.toml` (backend) | ✅ | `eusolicit-app/services/client-api/pyproject.toml` |
| `conftest.py` (pytest) | ✅ | `eusolicit-app/services/client-api/tests/conftest.py` |

**Detected stack:** `fullstack`
**Story target layer:** `backend` (Python/pytest API tests — no UI component in Story 2.7)

### TEA Config Flags

| Flag | Value | Source |
|------|-------|--------|
| `test_stack_type` | `auto` → detected `fullstack` | config.yaml (not set, auto-detected) |
| `tea_use_playwright_utils` | not set → `disabled` | config.yaml |
| `tea_use_pactjs_utils` | not set → `disabled` | config.yaml |
| `tea_pact_mcp` | not set → `none` | config.yaml |
| `tea_browser_automation` | not set → `none` | config.yaml |

### Prerequisites Satisfied

- [x] Story 2.7 has clear acceptance criteria (AC1–AC6)
- [x] pytest + pytest-asyncio configured (`pyproject.toml` in client-api)
- [x] `conftest.py` provides `client_api_session_factory`, `test_redis_client`, `rsa_env_setup`
- [x] Shared-session pattern established in `test_auth_refresh.py`
- [x] `CapturingEmailService` pattern designed (covers `send_password_reset_email` new method)
- [x] 165 existing tests passing (Story 2.6 baseline — must not regress)
- [x] Epic test design loaded (E02-P1-008, E02-P1-009, E02-P1-010, E02-P3-003)

### Knowledge Fragments Applied

| Fragment | Tier | Applied |
|----------|------|---------|
| `data-factories.md` | core | ✅ `_register_and_verify()` helper |
| `test-quality.md` | core | ✅ Isolation via rollback, unique UUIDs per test |
| `test-healing-patterns.md` | core | ✅ Pre-condition asserts with diagnostic messages |
| `test-levels-framework.md` | backend | ✅ Integration-level API tests (no unit mocks) |
| `test-priorities-matrix.md` | backend | ✅ P1 for AC1–AC6, P3 for full E2E flow |

---

## Step 2: Generation Mode

**Mode selected:** AI Generation (backend story, standard auth pattern, clear ACs)

**Rationale:** Story 2.7 is a backend-only story with well-specified acceptance criteria.
All endpoints are standard REST API calls (POST). No browser recording required.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenarios

| AC | Description | Test Level | Priority | Epic Test ID |
|----|-------------|------------|----------|--------------|
| **AC1** | Request always returns 200 regardless of email existence (no enumeration) | API/Integration | P1 | E02-P1-008 |
| **AC2** | Token is 64-char SHA-256 hex; expires_at ≈ now+1h; raw token not in DB | API/Integration | P1 | — (part of AC3 validation) |
| **AC3** | Token is single-use; second confirm → 400 `invalid_token` | API/Integration | P1 | E02-P1-009 |
| **AC4** | Password policy enforced via Pydantic (min 8, 1 upper, 1 digit) → 422 | API/Integration | P1 | — |
| **AC5** | All refresh tokens bulk-revoked after reset; pre-reset token → 401 | API/Integration | P1 | E02-P1-010 |
| **AC6** | Expired token → 400 `invalid_token` (same error as consumed — no distinction) | API/Integration | P1 | E02-P1-009 |
| **P3** | Full E2E: register → verify → login → request → confirm → re-login | API/Integration | P3 | E02-P3-003 |

### Test Level Justification

- **Integration (API)**: All tests hit the FastAPI app via HTTPX AsyncClient against a real PostgreSQL session. No unit mocks for service logic — integration coverage provides stronger confidence for security-critical flows.
- **No E2E browser tests**: Story 2.7 is backend-only; frontend flows are deferred to Epic 9.
- **No Pact contract tests**: `tea_use_pactjs_utils` disabled; no consumer–provider contract needed at this stage.

### Risk Coverage

| Risk ID | Category | Score | Test Coverage |
|---------|----------|-------|---------------|
| E02-R-005 | SEC | 3 | AC1 (enumeration prevention), AC2 (hash-before-store), AC3/AC6 (token lifecycle) |
| E02-R-003 | SEC | 6 | AC5 (refresh token bulk revocation after reset) |

---

## Step 4: Test Generation Results

### 🔴 TDD Red Phase

All tests are decorated with `@pytest.mark.skip` — they document the **expected behavior** that the implementation must satisfy. They will fail (or be skipped) until the feature is implemented.

**Remove `@pytest.mark.skip` after implementing all Tasks in Story 2.7.**

### Generated Test File

**Path:** `eusolicit-app/services/client-api/tests/api/test_auth_password_reset.py`
**Status:** 🔴 RED — all tests skipped (pre-implementation)

### Test Inventory

| Test Class | Test Method | AC | Epic ID | Priority | Red Reason |
|------------|-------------|-----|---------|----------|------------|
| `TestAC1NoEnumeration` | `test_nonexistent_email_returns_200` | AC1 | E02-P1-008 | P1 | Endpoint not implemented |
| `TestAC1NoEnumeration` | `test_existing_email_returns_200` | AC1 | E02-P1-008 | P1 | Endpoint not implemented |
| `TestAC1NoEnumeration` | `test_response_body_identical_for_known_and_unknown_email` | AC1 | E02-P1-008 | P1 | Endpoint not implemented |
| `TestAC2TokenProperties` | `test_token_hash_is_64_char_hex_and_consumed_at_is_null` | AC2 | — | P1 | Model + endpoint not implemented |
| `TestAC2TokenProperties` | `test_expires_at_is_approximately_one_hour_from_now` | AC2 | — | P1 | Model + endpoint not implemented |
| `TestAC2TokenProperties` | `test_raw_token_not_stored_in_db` | AC2 | — | P1 | Model + endpoint not implemented |
| `TestAC3SingleUse` | `test_first_confirm_returns_200` | AC3 | E02-P1-009 | P1 | Confirm endpoint not implemented |
| `TestAC3SingleUse` | `test_confirm_sets_consumed_at_in_db` | AC3 | E02-P1-009 | P1 | Confirm endpoint not implemented |
| `TestAC3SingleUse` | `test_second_confirm_with_same_token_returns_400_invalid_token` | AC3 | E02-P1-009 | P1 | Confirm endpoint not implemented |
| `TestAC4PasswordPolicyEnforced` | `test_password_too_short_returns_422` | AC4 | — | P1 | Schema not implemented |
| `TestAC4PasswordPolicyEnforced` | `test_password_no_uppercase_returns_422` | AC4 | — | P1 | Schema not implemented |
| `TestAC4PasswordPolicyEnforced` | `test_password_no_digit_returns_422` | AC4 | — | P1 | Schema not implemented |
| `TestAC5RefreshTokensRevoked` | `test_pre_reset_refresh_token_rejected_after_reset` | AC5 | E02-P1-010 | P1 | Confirm + revocation not implemented |
| `TestAC5RefreshTokensRevoked` | `test_all_refresh_token_rows_marked_revoked_in_db` | AC5 | E02-P1-010 | P1 | Confirm + revocation not implemented |
| `TestAC6ExpiredToken` | `test_expired_token_returns_400_invalid_token` | AC6 | E02-P1-009 | P1 | Confirm + model not implemented |
| `TestAC6ExpiredToken` | `test_expired_and_consumed_errors_are_indistinguishable` | AC6 | E02-P1-009 | P1 | Confirm + model not implemented |
| `TestP3FullFlow` | `test_full_password_reset_lifecycle` | ALL | E02-P3-003 | P3 | Full flow not implemented |

**Total tests: 17**
- API/Integration: 17
- E2E browser: 0 (N/A — backend-only story)
- All tests: `@pytest.mark.skip` (🔴 RED PHASE)

### Priority Coverage Summary

| Priority | Count | Epic Test IDs |
|----------|-------|---------------|
| P1 | 16 | E02-P1-008 (3), E02-P1-009 (5), E02-P1-010 (2), no-ID (6) |
| P3 | 1 | E02-P3-003 |

### Fixture Infrastructure

**Shared fixture:** `password_reset_client_and_session`
- Pattern: matches `refresh_client_and_session` from `test_auth_refresh.py`
- Yields: `(httpx.AsyncClient, AsyncSession, CapturingEmailService)`
- Setup: registers user, verifies email via SQL
- Teardown: `session.rollback()`

**CapturingEmailService:**
- Replaces `StubEmailService` via dependency override
- Implements `send_verification_email` + `send_password_reset_email`
- Stores raw tokens in `reset_tokens: list[str]` for test inspection
- Eliminates need for `monkeypatch` on global class methods

**Helper:** `_register_and_verify(client, session, uid, prefix, password)`
- Registers fresh user + verifies email via SQL in one call
- Reduces boilerplate across per-test user creation

---

## Step 5: Validation

### Checklist

#### Prerequisites
- [x] Story 2.7 approved with clear ACs (6 ACs explicitly stated)
- [x] Test framework: `pytest` + `pytest-asyncio` (existing, from Stories 2.2–2.6)
- [x] No new test infrastructure required beyond `CapturingEmailService`
- [x] All tests isolated: unique UUID emails, session rollback teardown

#### TDD Red Phase Compliance
- [x] All 17 tests decorated with `@pytest.mark.skip` (documented failing tests)
- [x] All tests assert **expected behavior** (not placeholder `assert True`)
- [x] All test failure messages include diagnostic context (f-strings with status codes + response bodies)
- [x] No tests will erroneously pass before implementation (skip ensures clean red phase)

#### Acceptance Criteria Coverage
- [x] AC1 → 3 tests in `TestAC1NoEnumeration` (unknown email, known email, identical response bodies)
- [x] AC2 → 3 tests in `TestAC2TokenProperties` (64-char hash, expiry window, raw-not-stored)
- [x] AC3 → 3 tests in `TestAC3SingleUse` (first confirm 200, consumed_at set, second 400)
- [x] AC4 → 3 tests in `TestAC4PasswordPolicyEnforced` (too short, no uppercase, no digit)
- [x] AC5 → 2 tests in `TestAC5RefreshTokensRevoked` (API 401, DB bulk revocation)
- [x] AC6 → 2 tests in `TestAC6ExpiredToken` (expired 400, identical error to consumed)
- [x] P3 → 1 test in `TestP3FullFlow` (8-step lifecycle E2E)

#### Pattern Consistency
- [x] Shared-session fixture matches `refresh_client_and_session` pattern exactly
- [x] Module docstring matches existing test file conventions (Story 2.6 template)
- [x] `from __future__ import annotations` at top
- [x] `from client_api.main import app as fastapi_app  # noqa: PLC0415` inside fixture
- [x] `@pytest.mark.asyncio` and `@pytest.mark.integration` on each test method
- [x] No `print()` statements (uses assert messages with f-strings)

#### Security Assertions
- [x] E02-R-005 (enumeration): AC1 tests verify **both** status code AND response body identity
- [x] E02-R-005 (hash storage): AC2 `test_raw_token_not_stored_in_db` verifies SHA-256 relationship
- [x] E02-R-003 (session security): AC5 verifies refresh token rejection at API level + DB level

#### No Regression Risk
- [x] Test file is additive — no changes to existing test files
- [x] `CapturingEmailService` does not modify `StubEmailService` class
- [x] Dependency overrides cleared in `finally` block
- [x] All test emails use unique UUIDs to avoid cross-test interference

---

## Next Steps (TDD Green Phase)

After implementing all Tasks in Story 2.7:

### 1. Remove `@pytest.mark.skip` decorators

```bash
# Remove all skip decorators from the test file
# Then run tests:
cd eusolicit-app/services/client-api
pytest tests/api/test_auth_password_reset.py -v --asyncio-mode=auto
```

### 2. Expected Green Phase: All 17 tests pass

```
tests/api/test_auth_password_reset.py::TestAC1NoEnumeration::test_nonexistent_email_returns_200 PASSED
tests/api/test_auth_password_reset.py::TestAC1NoEnumeration::test_existing_email_returns_200 PASSED
tests/api/test_auth_password_reset.py::TestAC1NoEnumeration::test_response_body_identical_for_known_and_unknown_email PASSED
tests/api/test_auth_password_reset.py::TestAC2TokenProperties::test_token_hash_is_64_char_hex_and_consumed_at_is_null PASSED
tests/api/test_auth_password_reset.py::TestAC2TokenProperties::test_expires_at_is_approximately_one_hour_from_now PASSED
tests/api/test_auth_password_reset.py::TestAC2TokenProperties::test_raw_token_not_stored_in_db PASSED
tests/api/test_auth_password_reset.py::TestAC3SingleUse::test_first_confirm_returns_200 PASSED
tests/api/test_auth_password_reset.py::TestAC3SingleUse::test_confirm_sets_consumed_at_in_db PASSED
tests/api/test_auth_password_reset.py::TestAC3SingleUse::test_second_confirm_with_same_token_returns_400_invalid_token PASSED
tests/api/test_auth_password_reset.py::TestAC4PasswordPolicyEnforced::test_password_too_short_returns_422 PASSED
tests/api/test_auth_password_reset.py::TestAC4PasswordPolicyEnforced::test_password_no_uppercase_returns_422 PASSED
tests/api/test_auth_password_reset.py::TestAC4PasswordPolicyEnforced::test_password_no_digit_returns_422 PASSED
tests/api/test_auth_password_reset.py::TestAC5RefreshTokensRevoked::test_pre_reset_refresh_token_rejected_after_reset PASSED
tests/api/test_auth_password_reset.py::TestAC5RefreshTokensRevoked::test_all_refresh_token_rows_marked_revoked_in_db PASSED
tests/api/test_auth_password_reset.py::TestAC6ExpiredToken::test_expired_token_returns_400_invalid_token PASSED
tests/api/test_auth_password_reset.py::TestAC6ExpiredToken::test_expired_and_consumed_errors_are_indistinguishable PASSED
tests/api/test_auth_password_reset.py::TestP3FullFlow::test_full_password_reset_lifecycle PASSED
```

### 3. Verify no regressions

```bash
pytest tests/ -v --asyncio-mode=auto
# Expect: 165 (existing) + 17 (new) = 182 tests passing
```

### 4. Implementation Checklist (Story Tasks)

Tests will pass once each task below is implemented:

| Task | What it enables |
|------|-----------------|
| Task 1: `PasswordResetToken` ORM model | `TestAC2TokenProperties` (DB row inspection) |
| Task 2: Alembic migration 006 | Table exists for all tests |
| Task 3: `send_password_reset_email()` in email service | `CapturingEmailService` contract satisfied |
| Task 4: Pydantic schemas | `TestAC4PasswordPolicyEnforced` (422 on weak password) |
| Task 5: `request_password_reset()` + `confirm_password_reset()` | All AC1–AC6 |
| Task 6: Route handlers | All AC1–AC6 (endpoints exist) |

### 5. Next recommended workflow

After green phase verified: Run `bmad-code-review` or `bmad-qa-generate-e2e-tests` for
frontend flows when the UI is implemented in Epic 9.

---

## Summary Statistics

| Metric | Value |
|--------|-------|
| Total tests generated | 17 |
| TDD phase | 🔴 RED |
| All tests skipped | ✅ (`@pytest.mark.skip`) |
| AC coverage | 6/6 (AC1–AC6) + P3 E2E |
| Epic test IDs covered | E02-P1-008, E02-P1-009, E02-P1-010, E02-P3-003 |
| Risks mitigated | E02-R-005 (SEC), E02-R-003 (SEC) |
| Test file path | `eusolicit-app/services/client-api/tests/api/test_auth_password_reset.py` |
| Execution mode | Sequential (AI generation, backend story) |
| Parallel gain | N/A (sequential) |

---

## Generated Files

| File | Type | Status |
|------|------|--------|
| `eusolicit-app/services/client-api/tests/api/test_auth_password_reset.py` | Test file | ✅ Created (🔴 RED) |
| `eusolicit-docs/test-artifacts/atdd-checklist-2-7-password-reset-flow.md` | Checklist | ✅ This document |
