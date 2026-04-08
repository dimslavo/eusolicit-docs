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
storyId: 2-9-team-member-management
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/2-9-team-member-management.md
  - eusolicit-docs/test-artifacts/test-design-epic-02.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_company_profile.py
  - eusolicit-app/services/client-api/pyproject.toml
  - eusolicit-app/services/client-api/tests/api/test_auth_password_reset.py
  - eusolicit-docs/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 2.9 — Team Member Management

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated, implementation pending
**Story:** `eusolicit-docs/implementation-artifacts/2-9-team-member-management.md`

---

## Preflight Summary

| Item | Result |
|------|--------|
| Stack detected | `fullstack` (Python backend + Next.js frontend) |
| Story stack | Backend-only (Python/FastAPI) |
| Test framework | `pytest` + `pytest-asyncio` + `httpx` |
| Story status | `ready-for-dev` — AC clear and complete |
| Epic test design | Loaded (`test-design-epic-02.md`) |
| TEA config flags | All unset — no playwright-utils, no pact.js, no browser automation |
| Generation mode | AI generation (backend story, no browser recording needed) |

---

## Generation Mode

**Selected:** AI generation

**Rationale:** Story 2.9 is pure backend (FastAPI). All acceptance criteria describe
HTTP API contracts, database state changes, and audit log entries — no UI interaction
needed. Backend pattern used: `pytest + pytest-asyncio + httpx` matching project
conventions from `test_company_profile.py` and `test_auth_password_reset.py`.

---

## Test Strategy

### AC → Test Scenario Mapping

| AC | Description | Scenarios | Level | Priority | Epic ID |
|----|-------------|-----------|-------|----------|---------|
| AC1 | Only `admin` role can call invite/role-change/remove | Non-admin roles → 403 on all mutation endpoints | API | P0 | — |
| AC2 | POST invite → 201, `client.invitations` row, hashed token, email stub | Happy path; DB state; token hash; expires_at | API | P1 | E02-P1-015 |
| AC3 | GET members → list with id, email, full_name, role, status | Admin; any member; field presence; pending status | API | P2 | E02-P2-011 |
| AC4 | PATCH role → 200, DB updated, audit log before/after | Role update; DB verification; audit log fields | API | P2 | E02-P2-012 |
| AC5 | DELETE last admin → 409 before deletion | 409 guard; membership persists; non-admin → 204 | API | P1 | E02-P1-017 |
| AC6 | POST accept-invite → creates/links user, activates membership | New user; existing user; missing password → 422 | API | P1 | E02-P1-016 |
| AC7 | Token expiry (7 days); invalid/already-accepted → 400 | Expired; reused; fabricated token | API | P2 | E02-P2-013, E02-R-009 |
| AC8 | All mutations audit-logged with entity_type, action_type, before, after | Invite; role-change; remove | API | P2 | E02-P1-018 |
| AC9 | Unauthenticated → 401; cross-company → 403 | No token; cross-company GET/POST/PATCH/DELETE | API | P0 | E02-R-002 |

### Red Phase Design Decisions

- All tests use `@pytest.mark.skip(reason="ATDD red phase: Story 2.9 not yet implemented")`
- Tests assert **expected** behavior (realistic assertions, not placeholders)
- Tests will fail because:
  1. `members.py` router not yet registered → all endpoints return 404
  2. `client.invitations` table not created → `client.invitations` queries raise DB errors
  3. `member_service.py` not yet implemented → import errors if wired
  4. `send_invite_email` abstract method not yet on `EmailServiceBase` → `AttributeError`
- `CapturingInviteEmailService` captures raw invite tokens for `accept-invite` tests

---

## TDD Red Phase Status

✅ Failing tests generated

| Category | Test Count | Skip Markers |
|----------|-----------|--------------|
| AC2 — Invite Member | 4 | All `@pytest.mark.skip` |
| AC3 — List Members | 5 | All `@pytest.mark.skip` |
| AC4 — Change Role | 4 | All `@pytest.mark.skip` |
| AC5 — Last Admin Guard | 4 | All `@pytest.mark.skip` |
| AC6 — Accept Invite | 4 | All `@pytest.mark.skip` |
| AC7 — Token Expiry | 3 | All `@pytest.mark.skip` |
| AC8 — Audit Log | 3 | All `@pytest.mark.skip` |
| AC1+AC9 — Auth Enforcement | 7 (+2 parametrized) | All `@pytest.mark.skip` |
| **Total** | **34** | **All skipped** |

**Note on parametrized tests:** `test_non_admin_roles_cannot_invite` expands to 3 cases
(reviewer, read_only, bid_manager) and `test_non_admin_roles_cannot_change_role` expands
to 3 cases (contributor, reviewer, read_only) — total ~40 test executions.

---

## Acceptance Criteria Coverage

### AC1 — Admin-only role enforcement

- [x] `test_contributor_cannot_invite_returns_403`
- [x] `test_non_admin_roles_cannot_invite` (parametrized: reviewer, read_only, bid_manager)
- [x] `test_non_admin_roles_cannot_change_role` (parametrized: contributor, reviewer, read_only)

### AC2 — POST /invite creates pending invitation (E02-P1-015)

- [x] `test_admin_invite_returns_201`
- [x] `test_invite_creates_invitation_row_with_null_accepted_at`
- [x] `test_invite_stores_sha256_hash_and_sends_raw_token`
- [x] `test_invite_sets_expires_at_7_days_in_future`

### AC3 — GET /members returns member list (E02-P2-011)

- [x] `test_admin_can_list_members_returns_200`
- [x] `test_member_response_contains_all_required_fields`
- [x] `test_active_admin_has_status_active`
- [x] `test_pending_invite_appears_with_status_pending`
- [x] `test_non_admin_member_can_list_members`

### AC4 — PATCH role + audit log (E02-P2-012)

- [x] `test_admin_can_change_member_role_returns_200`
- [x] `test_role_change_updates_membership_in_db`
- [x] `test_role_change_creates_audit_log_with_before_and_after`
- [x] `test_role_change_non_existent_user_returns_404`

### AC5 — DELETE last admin guard (E02-P1-017)

- [x] `test_delete_only_admin_returns_409`
- [x] `test_delete_only_admin_does_not_remove_membership`
- [x] `test_delete_non_admin_member_returns_204`
- [x] `test_second_admin_allows_first_admin_removal`

### AC6 — POST /auth/accept-invite (E02-P1-016)

- [x] `test_new_user_accept_invite_returns_200_with_is_new_user_true`
- [x] `test_accept_invite_marks_invitation_accepted_at`
- [x] `test_new_user_missing_password_returns_422`
- [x] `test_existing_user_accept_invite_returns_is_new_user_false`

### AC7 — Token expiry and reuse (E02-P2-013, E02-R-009)

- [x] `test_expired_token_returns_400`
- [x] `test_already_accepted_token_returns_400`
- [x] `test_invalid_token_returns_400`

### AC8 — Audit log for all mutations

- [x] `test_invite_creates_audit_log_entry`
- [x] `test_remove_member_creates_audit_log_with_before`
- [x] `test_audit_log_includes_ip_address`
- [x] *(role-change audit log covered in AC4: `test_role_change_creates_audit_log_with_before_and_after`)*

### AC9 — Authentication and cross-company isolation (E02-R-002)

- [x] `test_unauthenticated_get_members_returns_401`
- [x] `test_unauthenticated_post_invite_returns_401`
- [x] `test_cross_company_admin_cannot_list_members`
- [x] `test_cross_company_admin_cannot_invite_to_other_company`

---

## Epic Test ID Traceability

| Epic Test ID | Coverage | Test Method |
|---|---|---|
| E02-P1-015 | Invite → invitations row; token hash; email stub | `TestAC2InviteMember` (4 tests) + `test_pending_invite_appears_with_status_pending` |
| E02-P1-016 | Accept invite: new-user + existing-user paths | `TestAC6AcceptInvite` (4 tests) |
| E02-P1-017 | DELETE last admin → 409; no deletion | `TestAC5LastAdminGuard` (first 2 tests) |
| E02-P2-011 | GET members → all required fields; active + pending | `TestAC3ListMembers` (5 tests) |
| E02-P2-012 | Role change → audit before.role ≠ after.role | `test_role_change_creates_audit_log_with_before_and_after` |
| E02-P2-013 | Expired token (>7 days) → 400 | `test_expired_token_returns_400` |
| E02-R-002 | Cross-company admin → 403 | `test_cross_company_admin_cannot_list_members`, `test_cross_company_admin_cannot_invite_to_other_company` |
| E02-R-009 | Already-accepted token → 400 | `test_already_accepted_token_returns_400` |

---

## Key Design Decisions and Assumptions

### 1. Python test pattern (not TypeScript)

Project uses `pytest + pytest-asyncio + httpx`. The ATDD skill's TypeScript `test.skip()`
convention maps to Python's `@pytest.mark.skip(reason="...")`. All tests follow the
shared-session fixture pattern from `test_company_profile.py`.

### 2. CapturingInviteEmailService

Tests for `accept-invite` (AC6, AC7) need the raw invite token that `invite_member()`
sends via email. The `CapturingInviteEmailService` stub intercepts `send_invite_email()`
calls and stores raw tokens. **Implements all three abstract methods** that
`EmailServiceBase` will declare:
- `send_verification_email` (existing, Story 2.1)
- `send_password_reset_email` (existing, Story 2.7)
- `send_invite_email` (new, Story 2.9 Task 3)

### 3. Audit log column naming

The `AuditLog` model uses `before` and `after` as attribute names (per dev notes).
SQL queries use `"before"` and `"after"` (double-quoted) to avoid potential
PostgreSQL keyword conflicts.

### 4. AC8 entity_type ambiguity

- **AC8** states `entity_type="company_membership"` for **all** mutations including invite.
- **Dev notes (Task 5.3)** specify `entity_type="invitation"` for invite actions (since
  no membership row exists yet at invite time).

The invite audit log test (`test_invite_creates_audit_log_entry`) accepts either
`entity_type='invitation'` OR `entity_type='company_membership'` to handle both
interpretations. A comment in the test flags the discrepancy for the developer to resolve.

**Recommendation:** Follow dev notes — use `entity_type="invitation"` for invite actions.
Update AC8 to clarify.

### 5. `list_members` dual-query requirement

`GET /members` must query BOTH `client.company_memberships JOIN client.users` (active
members) AND `client.invitations WHERE accepted_at IS NULL` (pending invitations). The
`test_pending_invite_appears_with_status_pending` test validates this UNION behaviour.

### 6. No `company_memberships` row for pending invites

Per dev note #2, pending invitations store data only in `client.invitations`. No
`company_memberships` row with `user_id=NULL` is created at invite time (PK constraint
violation). Tests do **not** assert a membership row after inviting — only after
`accept-invite`.

---

## Fixture Architecture

```
conftest.py (session-scoped, autouse)
  ├── rsa_env_setup          — RSA key injection (JWT signing/verification)
  ├── test_redis_client      — Redis DB 1 for rate-limit isolation
  └── flush_rate_limit_keys  — Per-test cleanup of rate_limit:login:* keys

test_team_members.py (function-scoped fixtures)
  ├── CapturingInviteEmailService   — Captures raw invite tokens from send_invite_email()
  ├── admin_client_and_session      — Admin user + company + httpx client + shared session
  │     Yields: (client, session, access_token, company_id_str, invite_email_svc)
  └── _register_and_verify_with_role()  — Helper to create users with specific roles
```

---

## Implementation Guidance for Developer

### Files to Create (in order)

1. **`alembic/versions/007_invitations.py`** — Creates `client.invitations` table
   (required before any DB-touching tests can pass)
2. **`src/client_api/models/invitation.py`** — `Invitation` ORM model
3. **`src/client_api/schemas/members.py`** — Pydantic schemas (InviteMemberRequest, etc.)
4. **`src/client_api/services/member_service.py`** — Business logic
5. **`src/client_api/api/v1/members.py`** — APIRouter
6. **Files to modify:** `models/__init__.py`, `email_service.py`, `companies.py`, `auth.py`

### Critical implementation rules (must pass tests)

| Test | Implementation requirement |
|------|---------------------------|
| `test_invite_stores_sha256_hash` | Store `hashlib.sha256(raw_token.encode()).hexdigest()`; never store raw token |
| `test_invite_creates_invitation_row_with_null_accepted_at` | Insert `Invitation` row only; no `CompanyMembership` for pending |
| `test_pending_invite_appears_with_status_pending` | `list_members` must UNION active members + pending invitations |
| `test_delete_only_admin_returns_409` | Count admins BEFORE delete; 409 if count == 1 and target is that admin |
| `test_new_user_accept_invite_returns_200_with_is_new_user_true` | Create `User` + `CompanyMembership` atomically on acceptance |
| `test_new_user_missing_password_returns_422` | Service-level validation: password required when no existing user |
| `test_role_change_creates_audit_log_with_before_and_after` | Capture `before = {"role": membership.role.value}` BEFORE update |
| `test_cross_company_admin_cannot_list_members` | `_guard_company_access`: compare `current_user.company_id != company_id` |

---

## Green Phase Checklist (Post-Implementation)

After Story 2.9 is implemented:

1. Run migration: `alembic upgrade head` (includes `007_invitations.py`)
2. Remove `@pytest.mark.skip` from **all** test methods in `test_team_members.py`
3. Run: `pytest tests/api/test_team_members.py -v`
4. All tests should **PASS** (green phase)
5. If any test fails:
   - Failure = implementation bug (fix the service/router/model)
   - Or clarify the test assertion against the AC if the AC was ambiguous
6. Run full suite to verify no regressions:
   `pytest tests/ -v --cov=client_api --cov-report=term-missing`

---

## Generated Files

| File | Status | Notes |
|------|--------|-------|
| `eusolicit-app/services/client-api/tests/api/test_team_members.py` | ✅ Created | 34 tests, all `@pytest.mark.skip` (RED phase) |
| `eusolicit-docs/test-artifacts/atdd-checklist-2-9-team-member-management.md` | ✅ Created | This file |

---

## Risks and Notes

| Risk | Mitigation |
|------|-----------|
| `CapturingInviteEmailService` must implement new `send_invite_email` abstract method | Fixture will fail with `AttributeError` if Task 3.1 (add abstract method) not done — intentional RED phase failure |
| AC8 entity_type ambiguity (invitation vs company_membership) | Test accepts both values; developer should clarify with PO |
| `list_members` UNION pattern is complex | `test_pending_invite_appears_with_status_pending` directly validates this dual-query requirement |
| `before`/`after` column names may need quoting in PostgreSQL | Tests use `"before"` / `"after"` with double quotes in SQL |
| Parametrized async tests require `pytest-asyncio >= 0.23` | Already in pyproject.toml dev dependencies |
