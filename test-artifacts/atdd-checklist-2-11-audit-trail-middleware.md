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
workflowType: bmad-testarch-atdd
mode: create
storyId: 2-11-audit-trail-middleware
detectedStack: backend
generationMode: AI Generation (backend — no browser recording)
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/2-11-audit-trail-middleware.md
  - eusolicit-docs/test-artifacts/test-design-epic-02.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_company_profile.py
  - eusolicit-app/services/client-api/tests/api/test_team_members.py
  - eusolicit-app/services/client-api/tests/api/test_auth_password_reset.py
  - eusolicit-app/services/client-api/tests/unit/test_rbac.py
  - eusolicit-app/services/client-api/src/client_api/models/audit_log.py
  - eusolicit-app/services/client-api/src/client_api/services/member_service.py
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 2.11 — Audit Trail Middleware

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated; feature not yet implemented
**Story Status:** not started

---

## Step 1: Preflight & Context

### Stack Detection

- **Detected stack:** `backend`
- **Detection evidence:** `pyproject.toml` with Python/FastAPI deps; test suite uses pytest + pytest-asyncio + httpx; no `playwright.config.*` or browser dependencies
- **Test framework:** pytest 8.x + pytest-asyncio + httpx AsyncClient

### Prerequisites Check

| Requirement | Status | Notes |
|-------------|--------|-------|
| Story has clear acceptance criteria | ✅ | 6 ACs with full task breakdown |
| pytest conftest with async session fixture | ✅ | `client_api_session_factory` + `client_session` in tests/conftest.py |
| AuditLog ORM model exists | ✅ | `src/client_api/models/audit_log.py` — shared.audit_log table |
| DB append-only enforcement in place | ✅ | Migration 002 revoked UPDATE/DELETE on shared.audit_log |
| Test fixture patterns available | ✅ | `company_client_and_session` in test_company_profile.py |
| Email capturing stub available | ✅ | CapturingEmailService pattern in test_auth_password_reset.py |
| `audit_service.py` exists | ❌ | **Does not exist** — blocked on S2.11 Task 1 |

### TEA Config Flags

| Flag | Value | Effect |
|------|-------|--------|
| `test_stack_type` | not set → `auto` → `backend` | Backend-only generation; no E2E tests needed |
| `tea_use_playwright_utils` | not configured | Disabled; pytest-native patterns used |
| `tea_use_pactjs_utils` | not configured | Disabled |
| `tea_pact_mcp` | not configured | None |
| `tea_browser_automation` | not configured | None |

---

## Step 2: Generation Mode

**Mode selected:** AI Generation

**Rationale:** `detected_stack = backend` — no browser recording needed. Acceptance criteria are clear with explicit test specs (Tasks 5 and 6). Standard pytest async patterns apply throughout.

---

## Step 3: Test Strategy

### AC → Test Level Mapping

| Acceptance Criterion | Test Level | Priority | Test IDs |
|---------------------|------------|----------|----------|
| **AC1** — `write_audit_entry` adds AuditLog to session + flushes; never raises | Unit | P1 | 2-11-UNIT-001 to 004 |
| **AC2** — POST/PUT/PATCH/DELETE mutations produce audit_log rows with all 8 fields | API Integration | P1 | 2-11-API-001 to 005 |
| **AC3** — Auth lifecycle events logged with `auth.` prefix | API Integration | P2 | 2-11-API-006 to 011 |
| **AC4** — `write_audit_entry` is the sole canonical audit helper (refactoring) | API Integration | P1 | Covered by AC2/AC3 tests |
| **AC5** — DB-level append-only: UPDATE raises ProgrammingError/InternalError | API/DB | P2 | 2-11-API-012 |
| **AC6** — `ip_address` extracted from HTTP request and passed to audit entry | API Integration | P1 | Asserted in all mutation/auth tests |

### Test Level Rationale

- **Unit** for AC1: `write_audit_entry` is a pure async helper — testable with `AsyncMock` session without a real DB. Exception suppression and structlog warning are best verified at unit level.
- **API Integration** for AC2, AC3, AC5, AC6: These require real DB round-trips, HTTP request context (for ip_address), and the full service call chain. Middleware behavior (route → service → audit) must be verified end-to-end.
- **No E2E tests**: Backend-only project; all tests use `httpx.AsyncClient` against ASGI app with real DB.

### Red Phase Design

All tests use `@pytest.mark.skip(reason="TDD RED PHASE: ...")`. This is intentional:
- Tests import `write_audit_entry` inside test methods to defer the `ImportError` until test execution (project convention — avoids collection failure)
- API tests are skipped to prevent false failures from missing service implementations
- Removing skip decorators is the signal that the story implementation is complete and tests should be green

---

## Step 4: Generated Test Files

### File 1: `tests/unit/test_audit_service.py`

**Purpose:** Unit tests for `write_audit_entry` function (AC1)

**Test classes and methods:**

| Test ID | Class | Method | AC | Epic ID | Priority |
|---------|-------|--------|----|---------|----------|
| 2-11-UNIT-001 | `TestWriteAuditEntryHappyPath` | `test_adds_audit_log_instance_with_all_fields_set` | AC1 | — | P1 |
| 2-11-UNIT-002 | `TestWriteAuditEntryHappyPath` | `test_all_optional_fields_as_none` | AC1 | — | P1 |
| 2-11-UNIT-003 | `TestWriteAuditEntryErrorSuppression` | `test_flush_exception_does_not_propagate` | AC1 | — | P0 |
| 2-11-UNIT-004 | `TestWriteAuditEntryErrorSuppression` | `test_flush_exception_triggers_warning_log` | AC1 | — | P1 |
| 2-11-UNIT-005 | `TestWriteAuditEntryErrorSuppression` | `test_session_add_exception_does_not_propagate` | AC1 | — | P0 |

**Fixtures required:** None (uses `AsyncMock`/`MagicMock` — no DB or HTTP infrastructure)

**Knowledge fragments used:** `test-quality.md`, `test-levels-framework.md`, `data-factories.md`

---

### File 2: `tests/api/test_audit_trail.py`

**Purpose:** API integration tests for mutation audit (AC2), auth event audit (AC3), ip_address (AC6), append-only enforcement (AC5)

**Test classes and methods:**

| Test ID | Class | Method | AC | Epic ID | Priority |
|---------|-------|--------|----|---------|----------|
| 2-11-API-001 | `TestMutationAuditCompany` | `test_put_company_produces_update_audit_row` | AC2, AC6 | E02-P1-018 | P1 |
| 2-11-API-002 | `TestMutationAuditCompany` | `test_patch_company_produces_update_audit_row` | AC2, AC6 | E02-P1-018 | P1 |
| 2-11-API-003 | `TestMutationAuditMembership` | `test_invite_member_produces_create_audit_row` | AC2, AC6 | E02-P1-018 | P1 |
| 2-11-API-004 | `TestMutationAuditMembership` | `test_change_member_role_produces_update_audit_with_role_diff` | AC2, AC6 | E02-P1-018, E02-P2-012 | P1 |
| 2-11-API-005 | `TestMutationAuditMembership` | `test_remove_member_produces_delete_audit_row` | AC2, AC6 | E02-P1-018 | P1 |
| 2-11-API-006 | `TestAuthEventAudit` | `test_login_produces_auth_login_audit_row` | AC3, AC6 | E02-P2-015 | P2 |
| 2-11-API-007 | `TestAuthEventAudit` | `test_logout_produces_auth_logout_audit_row` | AC3, AC6 | E02-P2-015 | P2 |
| 2-11-API-008 | `TestAuthEventAudit` | `test_register_produces_auth_register_audit_row` | AC3, AC6 | E02-P2-015 | P2 |
| 2-11-API-009 | `TestAuthEventAudit` | `test_password_reset_request_for_existing_user_produces_audit_row` | AC3, AC6 | E02-P2-015 | P2 |
| 2-11-API-010 | `TestAuthEventAudit` | `test_password_reset_request_for_nonexistent_user_produces_no_audit_row` | AC3 | E02-P2-015 | P2 |
| 2-11-API-011 | `TestAuthEventAudit` | `test_password_reset_confirm_produces_auth_confirm_audit_row` | AC3, AC6 | E02-P2-015 | P2 |
| 2-11-API-012 | `TestAuditAppendOnly` | `test_update_audit_log_raises_permission_error` | AC5 | E02-P2-014 | P2 |

**Fixtures defined in test file:**

| Fixture | Scope | Yields | Used by |
|---------|-------|--------|---------|
| `audit_company_client_and_session` | function | `(client, session, access_token, company_id_str, user_id_str, capturing_email_svc)` | Mutation audit tests |
| `audit_auth_bare_client_and_session` | function | `(client, session, capturing_email_svc)` | Auth event audit tests |

**Fixture from conftest.py used:**

| Fixture | Used by |
|---------|---------|
| `client_session` | `TestAuditAppendOnly` |

**Helpers defined:**

| Helper | Purpose |
|--------|---------|
| `_CapturingEmailServiceForAudit` | Captures verification, reset, and invite tokens |
| `_register_and_verify()` | Register + verify email via SQL; returns (user_id, email, company_id) |
| `_add_second_member()` | Register second user + insert membership via SQL for role/delete tests |

---

## Step 5: Acceptance Criteria Coverage

### AC1 — `write_audit_entry` function

- [x] **2-11-UNIT-001** — Adds AuditLog instance with all 7 keyword fields correctly mapped
- [x] **2-11-UNIT-002** — All optional fields (before, after, entity_type, entity_id, ip_address) passed as None → stored as None in AuditLog
- [x] **2-11-UNIT-003** — `session.flush()` raises → no exception propagates
- [x] **2-11-UNIT-004** — `session.flush()` raises → `log.warning("audit.write_failed", ...)` called
- [x] **2-11-UNIT-005** — `session.add()` raises → no exception propagates

### AC2 — All POST/PUT/PATCH/DELETE mutations produce audit_log rows

- [x] **2-11-API-001** — PUT /companies/{id} → `action_type="update"`, `entity_type="company"`, `before` non-null, `after` non-null
- [x] **2-11-API-002** — PATCH /companies/{id} → same structure
- [x] **2-11-API-003** — POST /members/invite → `action_type="create"`, `entity_type="invitation"`, `after` contains email + role
- [x] **2-11-API-004** — PATCH /members/{user_id} → `action_type="update"`, `before["role"] != after["role"]`
- [x] **2-11-API-005** — DELETE /members/{user_id} → `action_type="delete"`, `before` non-null, `after` is None

### AC3 — Auth lifecycle events with `auth.` prefix

- [x] **2-11-API-006** — POST /auth/login → `action_type="auth.login"`, `user_id` set, `entity_type="user"`, `after` contains company_id + role
- [x] **2-11-API-007** — POST /auth/logout → `action_type="auth.logout"`, `user_id` set, `entity_type="refresh_token"`
- [x] **2-11-API-008** — POST /auth/register → `action_type="auth.register"`, `user_id` set, `after["role"]="admin"`
- [x] **2-11-API-009** — POST /auth/password-reset/request (existing email) → `action_type="auth.password_reset_request"`, `user_id` set
- [x] **2-11-API-010** — POST /auth/password-reset/request (non-existent email) → **no audit row** (enumeration protection)
- [x] **2-11-API-011** — POST /auth/password-reset/confirm → `action_type="auth.password_reset_confirm"`, `user_id` set, `entity_type="user"`

### AC4 — Single canonical audit helper (covered via AC2/AC3 tests)

- [x] AC2 mutation tests pass implicitly only when company_service + member_service use `write_audit_entry` (not inline audit)
- [x] AC3 auth event tests pass implicitly only when auth_service uses `write_audit_entry`
- [ ] **Not explicitly tested with a separate test:** The canonical helper requirement is architectural. Consider adding a grep-based static check (e.g., in CI: `grep -r "session.add(AuditLog" src/` should return only `audit_service.py` and `rbac.py`).

### AC5 — Append-only at DB level

- [x] **2-11-API-012** — `UPDATE shared.audit_log … WHERE FALSE` via `client_api_role` session → `ProgrammingError` or `InternalError`

### AC6 — `ip_address` captured and passed

- [x] All 10 mutation + auth event tests assert `row.ip_address is not None`

---

## TDD Red Phase Validation

### Compliance Checklist

- [x] **All tests marked with `@pytest.mark.skip`** — will not run until skip is removed
- [x] **All tests assert EXPECTED behavior** — not placeholder `assert True`
- [x] **Imports deferred to method body** — `from client_api.services.audit_service import write_audit_entry  # noqa: PLC0415` inside each test, consistent with project convention
- [x] **Error messages are actionable** — each `assert` has a failure message pointing to the specific implementation task
- [x] **No test > 300 lines** — both test files are focused, well within limit
- [x] **Tests are isolated** — fixtures use `session.rollback()` in `finally`; unique UUIDs prevent cross-test collision
- [x] **No hard waits** — all async operations use `await`; no `asyncio.sleep` or time-based waits

### Known Design Decisions

| Decision | Rationale |
|----------|-----------|
| `entity_type="invitation"` for invite test (2-11-API-003) | Story task 6.4 explicitly specifies this. Current `member_service.py` uses `"company_membership"`. Developer must update `invite_member()` entity_type during Task 4.3 refactoring. |
| `_add_second_member` uses direct SQL insert for role/delete tests | Avoids the full invite-acceptance flow for tests that only need a target member to exist. Invite acceptance is tested separately in test_team_members.py. |
| `audit_auth_bare_client_and_session` has no pre-registered user | Each auth event test controls its own state, preventing cross-test audit row pollution when filtering by `user_id`. |
| `test_update_audit_log_raises_permission_error` uses `client_session` from conftest | Reuses the existing per-test rollback session which connects as `client_api_role` — exactly the role whose UPDATE permission was revoked. |

---

## Next Steps (TDD Green Phase)

After implementing Story 2.11:

### Implementation Order (recommended)

1. **Task 1** — Create `src/client_api/services/audit_service.py`
   → Remove `@pytest.mark.skip` from `tests/unit/test_audit_service.py`
   → Run: `pytest tests/unit/test_audit_service.py -v`
   → Expected: 5 tests GREEN ✅

2. **Task 2 + Task 3** — Add auth event logging to `auth_service.py` + update `auth.py` route handlers
   → Remove `@pytest.mark.skip` from `TestAuthEventAudit` in `test_audit_trail.py`
   → Run: `pytest tests/api/test_audit_trail.py::TestAuthEventAudit -v`
   → Expected: 6 tests GREEN ✅

3. **Task 4** — Refactor company/member inline audit writes
   → Remove `@pytest.mark.skip` from `TestMutationAuditCompany` and `TestMutationAuditMembership`
   → Run: `pytest tests/api/test_audit_trail.py::TestMutationAuditCompany tests/api/test_audit_trail.py::TestMutationAuditMembership -v`
   → Expected: 5 tests GREEN ✅
   → **Note:** `test_invite_member_produces_create_audit_row` requires `entity_type="invitation"` — update invite_member() entity_type during this task.

4. **Verify append-only** (should already be GREEN since DB constraint is pre-existing)
   → Remove `@pytest.mark.skip` from `TestAuditAppendOnly`
   → Run: `pytest tests/api/test_audit_trail.py::TestAuditAppendOnly -v`
   → Expected: 1 test GREEN ✅

5. **Full run**
   → `pytest tests/unit/test_audit_service.py tests/api/test_audit_trail.py -v`
   → Expected: 17 tests GREEN ✅

### Regression Check

Run the full E02 suite to verify S2.11 changes do not break existing audit rows:

```bash
pytest tests/api/test_company_profile.py tests/api/test_team_members.py -v
```

These tests already assert on audit_log rows (E02-P1-014, E02-P2-012) — they must remain green.

---

## Summary Statistics

| Category | Count |
|----------|-------|
| **Unit tests** (AC1) | 5 |
| **API integration tests** (AC2, AC3, AC5, AC6) | 12 |
| **Total tests generated** | **17** |
| Tests currently passing | 0 (all skipped — TDD RED) |
| Acceptance criteria covered | 6 / 6 |
| Epic test IDs covered | E02-P1-018, E02-P2-012, E02-P2-014, E02-P2-015 |

---

## Generated Files

| File | Type | Tests |
|------|------|-------|
| `eusolicit-app/services/client-api/tests/unit/test_audit_service.py` | Unit (pytest) | 5 |
| `eusolicit-app/services/client-api/tests/api/test_audit_trail.py` | API Integration (pytest + httpx) | 12 |
| `eusolicit-docs/test-artifacts/atdd-checklist-2-11-audit-trail-middleware.md` | ATDD Checklist | — |

---

*Generated by TEA Master Test Architect via bmad-testarch-atdd workflow — 2026-04-07*
