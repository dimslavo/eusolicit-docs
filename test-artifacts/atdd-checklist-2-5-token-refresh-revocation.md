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
storyKey: 2-5-token-refresh-revocation
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/2-5-token-refresh-revocation.md
  - eusolicit-docs/test-artifacts/test-design-epic-02.md
  - eusolicit-app/services/client-api/tests/api/test_login.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit-app/pyproject.toml
---

# ATDD Checklist: Story 2.5 — Token Refresh & Revocation

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — tests written before implementation
**Story Status:** ready-for-dev

---

## Preflight Summary

| Item | Value |
|------|-------|
| Story Key | `2-5-token-refresh-revocation` |
| Detected Stack | `fullstack` (Python backend service under test) |
| Generation Mode | AI Generation (backend story — no browser recording needed) |
| Execution Mode | Sequential |
| Test Framework | pytest + pytest-asyncio (`asyncio_mode = auto`) |
| Test File | `eusolicit-app/services/client-api/tests/api/test_auth_refresh.py` |
| Total Tests Generated | 13 |
| TDD Phase | 🔴 RED (all tests `@pytest.mark.skip`) |

### Prerequisites Verified

- [x] Story has clear acceptance criteria (AC1–AC6)
- [x] `conftest.py` provides `client_api_session_factory`, `test_redis_client`, `rsa_env_setup`
- [x] Shared-session pattern established in `test_login.py:login_client_and_session`
- [x] `client.refresh_tokens` table and `shared.audit_log` table exist (Story 2.1)
- [x] `pytest.ini_options` defines `integration` marker (`pyproject.toml`)

---

## TDD Red Phase Status

🔴 **13 failing tests generated** — all decorated with:

```python
@pytest.mark.skip(reason="RED PHASE: POST /api/v1/auth/refresh not yet implemented")
# or
@pytest.mark.skip(reason="RED PHASE: POST /api/v1/auth/logout not yet implemented")
```

Tests assert the **expected behavior** of the not-yet-implemented endpoints.
They will be **skipped** (not failed) in CI until the developer removes the
`@pytest.mark.skip` decorator once the feature is complete.

---

## Generation Mode

**Mode selected:** AI Generation

**Rationale:** Acceptance criteria are clear and cover standard API/auth patterns
(token rotation, revocation, breach detection). The stack under test is a Python
FastAPI backend service — no browser or UI interaction required. AI generation
from the story specification, epic test design, and existing test patterns
(test_login.py) produces precise, realistic tests.

---

## Test Strategy

### Stack & Test Level

| Dimension | Decision |
|-----------|----------|
| Service | `client-api` (Python / FastAPI / asyncpg) |
| Test level | API integration — full ASGI stack via `httpx.AsyncClient + ASGITransport` |
| DB isolation | Shared-session pattern — all requests share one rolled-back transaction |
| No E2E | Backend-only story; no browser tests needed |

### Fixture Design: `refresh_client_and_session`

```
register user → verify email (SQL) → login → yield (client, session, refresh_token)
                                                         └── rolled back after test
```

**Why shared session:** Tests need to inspect DB state (e.g., `is_revoked`,
`shared.audit_log`) after HTTP requests. Per-request sessions commit and make
cross-step DB visibility impossible. The shared session keeps all flushed writes
visible within the test transaction.

### Priority Assignments

| Priority | ACs | Epic IDs | Rationale |
|----------|-----|----------|-----------|
| **P0** | AC1, AC2, AC3, AC6 | E02-P0-010, E02-P0-011 | Core token security — blocks all downstream auth flows; E02-R-003 (score 6) |
| **P1** | AC4, AC5 | E02-P1-007, E02-P1-006 | Important flows; workaround exists (token expiry / manual DB revocation) |

---

## Acceptance Criteria Coverage

| AC | Description | Test Class | Epic Test ID | Priority | Status |
|----|-------------|-----------|-------------|----------|--------|
| **AC1** | Valid refresh → 200 with new `{access_token, refresh_token, token_type, expires_in}`; both tokens differ from originals | `TestAC1ValidRefresh` | E02-P0-010 (partial) | P0 | 🔴 RED |
| **AC2** | Consumed refresh token `is_revoked=True` immediately; reuse → 401 | `TestAC2RotationRevocation` | E02-P0-010 | P0 | 🔴 RED |
| **AC3** | Revoked token replayed → ALL family tokens bulk-revoked; `shared.audit_log` entry with `action_type='auth.token_breach'` | `TestAC3BreachDetection` | E02-P0-011 | P0 | 🔴 RED |
| **AC4** | Token past `expires_at` → 401 | `TestAC4ExpiredToken` | E02-P1-007 | P1 | 🔴 RED |
| **AC5** | `/logout` sets `is_revoked=True`; subsequent `/refresh` → 401; idempotent with unknown/already-revoked tokens | `TestAC5Logout` | E02-P1-006 | P1 | 🔴 RED |
| **AC6** | Breach audit entry includes `user_id`, `entity_type='refresh_token_family'`, `entity_id=<family_id>` | `TestAC3BreachDetection` | E02-P0-011 | P0 | 🔴 RED |

All 6 acceptance criteria covered. ✅

---

## Test Methods Inventory

| # | Class | Method | AC(s) | Epic ID | Priority |
|---|-------|--------|-------|---------|----------|
| 1 | `TestAC1ValidRefresh` | `test_valid_refresh_returns_200_with_token_structure` | AC1 | E02-P0-010 | P0 |
| 2 | `TestAC1ValidRefresh` | `test_new_tokens_differ_from_originals` | AC1 | E02-P0-010 | P0 |
| 3 | `TestAC2RotationRevocation` | `test_consumed_token_is_marked_revoked_in_db` | AC2 | E02-P0-010 | P0 |
| 4 | `TestAC2RotationRevocation` | `test_reuse_of_consumed_token_returns_401` | AC2 | E02-P0-010 | P0 |
| 5 | `TestAC3BreachDetection` | `test_breach_revokes_family_and_logs_audit` | AC3 + AC6 | E02-P0-011 | P0 |
| 6 | `TestAC3BreachDetection` | `test_breach_audit_entity_id_matches_family_id` | AC6 | E02-P0-011 | P0 |
| 7 | `TestAC4ExpiredToken` | `test_expired_refresh_token_returns_401` | AC4 | E02-P1-007 | P1 |
| 8 | `TestAC4ExpiredToken` | `test_expired_token_does_not_trigger_breach_detection` | AC4 / AC3 guard | E02-P1-007 | P1 |
| 9 | `TestAC5Logout` | `test_logout_returns_204` | AC5 | E02-P1-006 | P1 |
| 10 | `TestAC5Logout` | `test_logout_sets_token_revoked_in_db` | AC5 | E02-P1-006 | P1 |
| 11 | `TestAC5Logout` | `test_logout_prevents_subsequent_refresh` | AC5 | E02-P1-006 | P1 |
| 12 | `TestAC5Logout` | `test_logout_idempotent_with_already_revoked_token` | AC5 | E02-P1-006 | P1 |
| 13 | `TestAC5Logout` | `test_logout_idempotent_with_unknown_token` | AC5 | E02-P1-006 | P1 |

**P0 tests:** 6 | **P1 tests:** 7 | **Total:** 13

---

## TDD Validation

### Red Phase Compliance ✅

All 13 tests:
- [x] Decorated with `@pytest.mark.skip(reason="RED PHASE: ...")` — documented intent
- [x] Assert **expected behavior** (not placeholder assertions like `assert True`)
- [x] Use realistic test data (unique emails, real tokens from login flow)
- [x] Will be **skipped** in CI until `@pytest.mark.skip` is removed
- [x] Marked `@pytest.mark.integration` (registered marker in `pyproject.toml`)
- [x] Marked `@pytest.mark.asyncio` (consistent with existing test_login.py pattern)

### No Placeholder Assertions ✅

All assertions check specific expected values:
- HTTP status codes (`200`, `204`, `401`)
- JSON response fields (`access_token`, `refresh_token`, `token_type`, `expires_in`)
- DB column values (`is_revoked is True`)
- Audit log fields (`action_type`, `entity_type`, `user_id`, `entity_id`)
- Token identity (`new_token != original_token`)

---

## Key Implementation Constraints Encoded in Tests

The following dev notes from the story are encoded as test assertions or comments:

| Constraint | Where Encoded |
|-----------|---------------|
| `is_revoked` checked BEFORE `expires_at` (breach must fire regardless of expiry) | `test_expired_token_does_not_trigger_breach_detection` — guards against breach on expiry path |
| Bulk family revocation via `UPDATE` — new token also invalidated | Step 3 in `test_breach_revokes_family_and_logs_audit` asserts `new_token → 401` |
| Audit entry includes `user_id`, `entity_type='refresh_token_family'`, `entity_id=<family_id>` | Both breach tests assert all 4 audit fields |
| Logout is idempotent — unknown/already-revoked token returns 204 silently | `test_logout_idempotent_with_already_revoked_token`, `test_logout_idempotent_with_unknown_token` |
| Token rotation preserves `family_id` (new token shares same family) | `test_breach_audit_entity_id_matches_family_id` fetches `family_id` before rotation, verifies it appears in audit entry |
| `expires_in=900` (15 min access token) | AC1 response structure test |

---

## Generated Files

| File | Type | Status |
|------|------|--------|
| `eusolicit-app/services/client-api/tests/api/test_auth_refresh.py` | Test file (Python) | ✅ Created |
| `eusolicit-docs/test-artifacts/atdd-checklist-2-5-token-refresh-revocation.md` | ATDD checklist | ✅ Created |

---

## Next Steps: GREEN PHASE

After implementing Story 2.5 (routes, service methods, schemas):

1. **Remove `@pytest.mark.skip`** from all 13 test methods in `test_auth_refresh.py`
2. **Run tests:**
   ```bash
   cd eusolicit-app/services/client-api
   pytest tests/api/test_auth_refresh.py -v -m integration
   ```
3. **Verify all 13 tests PASS** (green phase)
4. **Regression check:** `pytest tests/ -v` — all 137 + 13 = 150 tests must pass
5. If any tests fail:
   - **Implementation bug** → fix the service/route code
   - **Test bug** → fix the test assertion (re-open this story)
6. Commit passing tests with story completion

### Routes to Implement

```
POST /api/v1/auth/refresh    → 200 LoginResponse (RefreshRequest body)
POST /api/v1/auth/logout     → 204 No Content    (LogoutRequest body)
```

### Files to Modify

```
src/client_api/schemas/auth.py          ← add RefreshRequest, LogoutRequest
src/client_api/services/auth_service.py ← add refresh_token(), logout()
src/client_api/api/v1/auth.py           ← add POST /refresh, POST /logout routes
```

---

## Risk Notes

| Risk | Mitigation in Tests |
|------|---------------------|
| `is_revoked` check order (breach vs expiry path) | `test_expired_token_does_not_trigger_breach_detection` explicitly guards this boundary |
| Family sweep missing new token | Step 3 in breach test asserts the freshly-rotated token is also revoked |
| Audit log missing fields | Both breach tests assert all 4 required fields (`action_type`, `entity_type`, `user_id`, `entity_id`) |
| Logout raises error on unknown token | `test_logout_idempotent_with_unknown_token` asserts 204 for completely unknown token |
| Shared session visibility gap | Fixture uses `await session.flush()` indirectly via service calls — SQLAlchemy flush before queries ensures visibility within the same transaction |

---

## Summary Statistics

```
🔴 TDD RED PHASE: Failing Tests Generated

📊 Summary:
- Total Tests: 13 (all @pytest.mark.skip)
  - P0 API Tests: 6 (RED)
  - P1 API Tests: 7 (RED)
  - E2E Tests: 0 (N/A — backend story)
- Fixtures Created: 1 (refresh_client_and_session)
- All tests will be SKIPPED until feature implemented

✅ Acceptance Criteria Coverage: 6/6 (100%)
🚀 Execution Mode: SEQUENTIAL (backend-only, AI generation)

📂 Generated Files:
- eusolicit-app/services/client-api/tests/api/test_auth_refresh.py (13 skipped tests)
- eusolicit-docs/test-artifacts/atdd-checklist-2-5-token-refresh-revocation.md

📝 Next: Implement Story 2.5 → remove @pytest.mark.skip → verify GREEN PHASE
```
