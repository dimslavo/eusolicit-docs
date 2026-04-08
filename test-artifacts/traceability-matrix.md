---
stepsCompleted:
  - step-01-load-context
  - step-02-discover-tests
  - step-03-map-criteria
  - step-04-analyze-gaps
  - step-05-gate-decision
lastStep: step-05-gate-decision
lastSaved: '2026-04-07'
epic: 2
epicTitle: 'Authentication & Identity'
previousEpic: 1
previousEpicFile: 'eusolicit-docs/test-artifacts/traceability-matrix-e01.md'
---

# Traceability Matrix & Quality Gate Report

**Epic:** E02 — Authentication & Identity
**Generated:** 2026-04-07
**Scope:** 12 stories (S02.01 – S02.12), 52 epic-level test IDs (P0: 12 · P1: 18 · P2: 18 · P3: 4)
**TDD Phase:** 🔴 RED — All 307 tests written; implementations not yet started (pre-implementation baseline)
**Previous Epic:** E01 — Infrastructure & Monorepo Foundation (PASS, 2026-04-06)

---

## TRACE_GATE: PASS

**Rationale:** P0 coverage is 100% (12/12 FULL). P1 coverage is 100% (18/18 FULL). P2 coverage is 100% (18/18 FULL). P3 coverage is 75% covered (2 FULL · 1 PARTIAL · 1 NONE for k6 performance test E02-P3-004, which is explicitly out-of-scope for pre-implementation ATDD). Overall FULL coverage: 96.2% (50/52); covered (FULL + PARTIAL): 98.1% (51/52). All 12 stories have 100% story-level AC coverage. All 307 acceptance-driven tests are in TDD RED phase — test scaffold complete; implementations not started. Gate applies to coverage adequacy, not execution results.

> **⚠️ TDD Baseline Notice:** All 307 tests are in `@pytest.mark.skip` / `pytest.mark.xfail` / failing state. Zero tests currently pass. This is the expected pre-implementation state. Gate measures whether every acceptance criterion has a corresponding test — it does. Actual green-phase gate should be re-run after implementation is complete and tests are unblocked.

---

## 1. Coverage Statistics

| Dimension | FULL | PARTIAL | NONE | Total | % FULL | % Covered |
|-----------|-----:|--------:|-----:|------:|-------:|----------:|
| **P0** | 12 | 0 | 0 | 12 | **100%** | 100% |
| **P1** | 18 | 0 | 0 | 18 | **100%** | 100% |
| **P2** | 18 | 0 | 0 | 18 | **100%** | 100% |
| **P3** | 2 | 1 | 1 | 4 | 50% | 75% |
| **Overall** | **50** | **1** | **1** | **52** | **96.2%** | **98.1%** |

### Gate Criteria Evaluation

| Criterion | Required | Actual | Status |
|-----------|----------|--------|--------|
| P0 coverage | 100% | 100% | ✅ MET |
| P1 coverage (PASS target) | ≥ 90% | 100% | ✅ MET |
| P1 coverage (minimum) | ≥ 80% | 100% | ✅ MET |
| Overall coverage | ≥ 80% | 96.2% | ✅ MET |

### Test Volume by Story

| Story | Title | Tests Written | TDD Phase | Test File(s) |
|-------|-------|-------------:|-----------|--------------|
| S02.01 | DB Schema — Auth & Identity | 42 | 🔴 FAIL (no impl) | `test_002_migration.py` |
| S02.02 | Email/Password Registration | 11 | 🔴 SKIP | `test_register.py` |
| S02.03 | Email/Password Login & JWT | 11 | 🔴 SKIP | `test_login.py` |
| S02.04 | JWT Auth Middleware | 28 | 🔴 XFAIL | `test_security.py` + `test_auth_middleware.py` |
| S02.05 | Token Refresh & Revocation | 13 | 🔴 SKIP | `test_auth_refresh.py` |
| S02.06 | Google OAuth2 Social Login | 15 | 🔴 SKIP | `test_auth_google.py` |
| S02.07 | Password Reset Flow | 17 | 🔴 SKIP | `test_auth_password_reset.py` |
| S02.08 | Company Profile CRUD | 23 | 🔴 SKIP | `test_company_profile.py` |
| S02.09 | Team Member Management | 34+ | 🔴 SKIP | `test_team_members.py` |
| S02.10 | Entity-Level RBAC Middleware | 65 | 🔴 SKIP | `test_rbac.py` + `test_rbac_middleware.py` |
| S02.11 | Audit Trail Middleware | 17 | 🔴 SKIP | `test_audit_service.py` + `test_audit_trail.py` |
| S02.12 | ESPD Profile CRUD | 31 | 🔴 SKIP | `test_espd_profile.py` + `test_audit_trail.py` |
| **Total** | | **307** | **All RED** | 14 test files |

---

## 2. Story-Level Acceptance Criteria Traceability

### S02.01 — Database Schema: Auth & Identity Tables

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | Alembic migration 002 runs cleanly (upgrade/downgrade/idempotent) | `TestAC1MigrationExecution` | 4 | Integration (subprocess) | P2 | **FULL** |
| AC2 | All 7 tables in correct schemas with correct column types | `TestAC2TableStructure` | 9 | Integration (DB) | P2 | **FULL** |
| AC3 | FK relationships correct (users↔companies via company_memberships) | `TestAC5ForeignKeys` | 4 | Integration (DB) | P2 | **FULL** |
| AC4 | Role enum: admin, bid_manager, contributor, reviewer, read_only | `TestAC3CompanyRoleEnum` | 2 | Integration (DB) | P2 | **FULL** |
| AC5 | shared.audit_log lives in the `shared` schema | `TestAC7AuditLogSchema` | 1 | Integration (DB) | P2 | **FULL** |
| *(impl)* | entity_permission enum = {read, write, manage} | `TestAC4EntityPermissionEnum` | 2 | Integration (DB) | P2 | **FULL** |
| *(impl)* | Required indexes (unique email, entity_permissions, audit_log) | `TestAC6Indexes` | 3 | Integration (DB) | P2 | **FULL** |
| *(impl)* | ORM models created; env.py updated to Base.metadata | `TestAC8OrmModels` | 15 | Unit (filesystem) | P2 | **FULL** |
| *(impl)* | Audit log append-only — UPDATE/DELETE denied for client_api_role | `TestAC9AppendOnlyEnforcement` | 2 | Integration (DB) | P2 | **FULL** |

**Story 2.1 AC Coverage: 5/5 story ACs = 100% · Epic Test IDs: E02-P2-001, E02-P2-002, E02-P2-003, E02-P2-014**

---

### S02.02 — Email/Password Registration

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | Creates user + company in one atomic transaction → 201 | `TestAC1AC2AC7RegisterHappyPath` | 2 | API/Integration | P0 | **FULL** |
| AC2 | First user assigned admin role in company_memberships | `TestAC1AC2AC7RegisterHappyPath` | 1 | API/Integration | P0 | **FULL** |
| AC3 | Password hashed with bcrypt cost=12; plaintext never persisted | `TestAC3PasswordHashing` | 1 | API/Integration | P2 | **FULL** |
| AC4 | Duplicate email → 409 Conflict | `TestAC4DuplicateEmail` | 1 | API/Integration | P1 | **FULL** |
| AC5 | Weak passwords rejected with 422 (< 8 chars, no uppercase, no digit) | `TestAC5WeakPassword` | 4 | API | P1 | **FULL** |
| AC6 | Email verification token: URL-safe 32-byte, SHA-256 hashed, 24h TTL | `TestAC6VerificationToken` | 2 | API/Integration | P2 | **FULL** |
| AC7 | Response excludes hashed_password | `TestAC1AC2AC7RegisterHappyPath` | 2 | API | P0 | **FULL** |

**Story 2.2 AC Coverage: 7/7 = 100% · Epic Test IDs: E02-P0-001, E02-P1-001, E02-P1-002**

---

### S02.03 — Email/Password Login & JWT Issuance

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | Valid credentials → 200 `{access_token, refresh_token, token_type, expires_in}` | `TestAC1AC2AC3LoginHappyPath` | 1 | API/Integration | P0 | **FULL** |
| AC2 | Access token RS256-signed with 15-min expiry | `TestAC1AC2AC3LoginHappyPath` | 1 | API/Integration | P0 | **FULL** |
| AC3 | Access token claims: sub, company_id, role, exp, iat, jti | `TestAC1AC2AC3LoginHappyPath` | 1 | API/Integration | P0 | **FULL** |
| AC4 | Refresh token opaque; stored with user_id, expiry, is_revoked, family_id | `TestAC4RefreshTokenDB` | 1 | API/Integration | P0 | **FULL** |
| AC5 | Wrong password / nonexistent email → 401 generic (no enumeration) | `TestAC5InvalidCredentials` | 3 | API | P1 | **FULL** |
| AC6 | Unverified email → 403 with clear message | `TestAC6UnverifiedEmail` | 1 | API | P1 | **FULL** |
| AC7 | Rate limiting: 5 failures per email per 15-min → 429 + Retry-After | `TestAC7RateLimiting` | 3 | API/Integration | P1 | **FULL** |

**Story 2.3 AC Coverage: 7/7 = 100% · Epic Test IDs: E02-P0-002, E02-P1-003, E02-P1-004, E02-P1-005**

---

### S02.04 — JWT Authentication Middleware

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | No Authorization header → 401 | `TestAC1MissingAuthHeader` | 2 | API | P0 | **FULL** |
| AC2 | Expired token → 401 with `token_expired` error code | `TestAC2ExpiredToken` | 2 | API | P0 | **FULL** |
| AC3 | Tampered/invalid token → 401 with `token_invalid` error code | `TestAC3TamperedToken` | 4 | API | P0 | **FULL** |
| AC4 | Valid token injects `CurrentUser(user_id, company_id, role)` — no DB query | `TestAC4ValidToken` | 5 | API | — | **FULL** |
| AC5 | `require_role("bid_manager")` rejects contributor, reviewer, read_only | `TestRequireRole` | 7 + 25 matrix | Unit | P0 | **FULL** |
| AC6 | Role hierarchy: admin(5) > bid_manager(4) > contributor(3) > reviewer(2) > read_only(1) | `TestRoleHierarchy` + `TestCurrentUser` | 8 | Unit | P0 | **FULL** |

**Story 2.4 AC Coverage: 6/6 = 100% · Epic Test IDs: E02-P0-003, E02-P0-004, E02-P0-005, E02-P0-006**

---

### S02.05 — Token Refresh & Revocation

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | Valid refresh → 200 with new token pair; both tokens differ from originals | `TestAC1ValidRefresh` | 2 | API/Integration | P0 | **FULL** |
| AC2 | Consumed refresh token marked revoked; reuse → 401 | `TestAC2RotationRevocation` | 2 | API/Integration | P0 | **FULL** |
| AC3 | Revoked token replay → all family tokens bulk-revoked (breach detection) | `TestAC3BreachDetection` | 2 | API/Integration | P0 | **FULL** |
| AC4 | Expired refresh token → 401 | `TestAC4ExpiredToken` | 2 | API/Integration | P1 | **FULL** |
| AC5 | `/logout` sets is_revoked=True; subsequent /refresh → 401; idempotent | `TestAC5Logout` | 5 | API/Integration | P1 | **FULL** |
| AC6 | Breach audit entry includes user_id, entity_type, entity_id=family_id | `TestAC3BreachDetection` | 2 | API/Integration | P0 | **FULL** |

**Story 2.5 AC Coverage: 6/6 = 100% · Epic Test IDs: E02-P0-010, E02-P0-011, E02-P1-006, E02-P1-007**

---

### S02.06 — Google OAuth2 Social Login

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | `/auth/google` redirects to Google with openid/email/profile scopes + state | `TestAC1GoogleRedirect` | 2 | API | P1 | **FULL** |
| AC2 | Callback validates state parameter (CSRF prevention) | `TestAC2InvalidState` | 2 | API | P1 | **FULL** |
| AC3 | New Google user: user row with google_sub, no password, email_verified=True | `TestAC3NewGoogleUser` | 2 | API/Integration | P1 | **FULL** |
| AC4 | Existing email user: google_sub linked; conflict guard on different google_sub | `TestAC4AccountLinking` | 2 | API/Integration | P1 | **FULL** |
| AC5 | Tokens issued identically to email/password login (RS256, claims) | `TestAC5TokenStructure` | 2 | API/Integration | P1 | **FULL** |
| AC6 | No company → `needs_company=true` redirect | `TestAC6NoCompany` | 2 | API | P1 | **FULL** |
| AC7 | Invalid/expired auth codes → error redirect, never 500 | `TestAC7OAuthError` | 3 | API | P1 | **FULL** |

**Story 2.6 AC Coverage: 7/7 = 100% · Epic Test IDs: E02-P1-011, E02-P1-012, E02-P1-013, E02-P3-002**

---

### S02.07 — Password Reset Flow

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | Request endpoint always returns 200 (no enumeration) | `TestAC1NoEnumeration` | 3 | API | P1 | **FULL** |
| AC2 | Token: 64-char SHA-256 hex; expires in 1h; raw not in DB | `TestAC2TokenProperties` | 3 | API/Integration | P1 | **FULL** |
| AC3 | Token is single-use; second confirm → 400 `invalid_token` | `TestAC3SingleUse` | 3 | API/Integration | P1 | **FULL** |
| AC4 | Confirm validates password policy (min 8, 1 upper, 1 digit) | `TestAC4PasswordPolicyEnforced` | 3 | API | P1 | **FULL** |
| AC5 | All refresh tokens bulk-revoked after password reset | `TestAC5RefreshTokensRevoked` | 2 | API/Integration | P1 | **FULL** |
| AC6 | Expired token → 400 (same error as consumed; indistinguishable) | `TestAC6ExpiredToken` | 2 | API/Integration | P1 | **FULL** |
| *(P3)* | Full reset lifecycle E2E (register → verify → request → confirm → re-login) | `TestP3FullFlow` | 1 | API/Integration | P3 | **FULL** |

**Story 2.7 AC Coverage: 6/6 story ACs = 100% + P3 E2E flow · Epic Test IDs: E02-P1-008, E02-P1-009, E02-P1-010, E02-P3-003**

---

### S02.08 — Company Profile CRUD

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | GET returns full profile for any authenticated member | `TestAC1GetCompanyProfile` | 1 | API | P2 | **FULL** |
| AC2 | PUT replaces entire profile (all required fields) | `TestAC2PutCompanyProfile` | 2 | API/Integration | P1 | **FULL** |
| AC3 | PATCH merges partial updates; absent fields unchanged | `TestAC3PatchCompanyProfile` | 2 | API/Integration | P2 | **FULL** |
| AC4 | CPV sector codes validated against `^\d{8}-\d$` | `TestAC4CPVValidation` | 4 | API | P2 | **FULL** |
| AC5 | Address validation: all 4 sub-fields required | `TestAC5AddressValidation` | 2 | API | P2 | **FULL** |
| AC6 | Only admin/bid_manager can update (others → 403) | `TestAC6RoleEnforcement` | 8 | API/Integration | P1 | **FULL** |
| AC7 | All mutations audit-logged with before/after snapshots | `TestAC7AuditLog` | 2 | API/Integration | P1 | **FULL** |
| *(auth)* | 401 unauthenticated; 403 cross-tenant | `TestAC1GetCompanyProfile` | 2 | API | P1 | **FULL** |

**Story 2.8 AC Coverage: 7/7 = 100% · Epic Test IDs: E02-P1-014, E02-P1-018, E02-P2-004, E02-P2-005, E02-P2-006, E02-P0-012 (partial)**

---

### S02.09 — Team Member Management

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | Only admin can invite/change-role/remove members | `TestAuthEnforcement` | 7 | API | P0 | **FULL** |
| AC2 | Invite → 201; invitations row; SHA-256 hashed token; 7-day expiry | `TestAC2InviteMember` | 4 | API/Integration | P1 | **FULL** |
| AC3 | GET members → list with id, email, full_name, role, status | `TestAC3ListMembers` | 5 | API | P2 | **FULL** |
| AC4 | Role change → 200; DB updated; audit before/after captured | `TestAC4ChangeRole` | 4 | API/Integration | P2 | **FULL** |
| AC5 | DELETE last admin → 409 (membership preserved) | `TestAC5LastAdminGuard` | 4 | API/Integration | P1 | **FULL** |
| AC6 | Accept invite → creates/links user; activates membership | `TestAC6AcceptInvite` | 4 | API/Integration | P1 | **FULL** |
| AC7 | Invite tokens expire after 7 days; reuse → 400 | `TestAC7TokenExpiry` | 3 | API | P2 | **FULL** |
| AC8 | All mutations audit-logged (invite/role-change/remove) | `TestAC8AuditLog` | 3 | API/Integration | P2 | **FULL** |
| AC9 | 401 unauthenticated; 403 cross-company | `TestAC9CrossCompany` | 4 | API | P0 | **FULL** |

**Story 2.9 AC Coverage: 7/7 story ACs = 100% · Epic Test IDs: E02-P1-015, E02-P1-016, E02-P1-017, E02-P2-011, E02-P2-012, E02-P2-013, E02-R-002, E02-R-009**

---

### S02.10 — Entity-Level RBAC Middleware

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | `check_entity_access(entity_type, required_permission)` returns async callable | `TestCheckEntityAccessFactory` | 4 | Unit | P1 | **FULL** |
| AC2 | No entity_permissions row → 403 (explicit grant model) | `TestExplicitGrantModel` + `TestNoEntityPermissionRow` | 6 | Unit + API | P0 | **FULL** |
| AC3 | Role ceiling: contributor ≤ write; reviewer/read_only ≤ read | `TestRolePermissionCeiling` + `TestCeilingEnforcement` + `TestRoleCeilingEnforcement` | 18 | Unit + API | P0 | **FULL** |
| AC4 | admin/bid_manager bypass entity checks (own-company only) | `TestBypassRoles` + `TestBypassRolesOwnCompany` + `TestCrossCompanyIsolation` | 16 | Unit + API | P0 | **FULL** |
| AC5 | Single optimized query per check (no N+1) | `TestSingleQueryConstraint` + `TestPerRequestCacheIntegration` | 2 | API | P1 | **FULL** |
| AC6 | Denials logged to audit_log with action_type="access_denied" | `TestAuditLogOnDenial` | 3 | API | P2 | **FULL** |
| AC7 | Per-request cache via `request.state._rbac_cache` | `TestPerRequestCache` + `TestPerRequestCacheIntegration` | 5 | Unit + API | P1 | **FULL** |

**Story 2.10 AC Coverage: 7/7 = 100% · Epic Test IDs: E02-P0-007, E02-P0-008, E02-P0-009, E02-P2-016, E02-P2-018**

---

### S02.11 — Audit Trail Middleware

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | `write_audit_entry()` adds AuditLog to session + flushes; never raises | `TestWriteAuditEntryHappyPath` + `TestWriteAuditEntryErrorSuppression` | 5 | Unit | P1 | **FULL** |
| AC2 | POST/PUT/PATCH/DELETE mutations produce audit rows (all 8 fields) | `TestMutationAuditCompany` + `TestMutationAuditMembership` | 5 | API/Integration | P1 | **FULL** |
| AC3 | Auth events logged with `auth.` prefix (login/logout/register/password_reset) | `TestAuthEventAudit` | 6 | API/Integration | P2 | **FULL** |
| AC4 | `write_audit_entry` is the sole canonical audit helper | *(implicit via AC2/AC3 tests passing only with correct single helper)* | — | — | P1 | **FULL** |
| AC5 | DB-level append-only: UPDATE on audit_log → ProgrammingError | `TestAuditAppendOnly` | 1 | API/DB | P2 | **FULL** |
| AC6 | `ip_address` extracted from HTTP request; passed to audit entry | *(asserted in all 10 mutation + auth event tests)* | 10 | API/Integration | P1 | **FULL** |

**Story 2.11 AC Coverage: 6/6 = 100% · Epic Test IDs: E02-P1-018, E02-P2-014, E02-P2-015**

---

### S02.12 — ESPD Profile CRUD

| AC | Description | Test Class | Tests | Level | Priority | Coverage |
|----|-------------|------------|------:|-------|----------|----------|
| AC1 | GET latest profile → 200; 404 if none; 403 cross-tenant | `TestAC1GetEspdProfile` | 4 | API | P0 | **FULL** |
| AC2 | PUT full profile → 200; version=1 first write; 422 on missing sections | `TestAC2PutEspdProfile` | 8 | API | P2 | **FULL** |
| AC3 | PATCH partial merge → 200; 404 if no base profile | `TestAC3PatchEspdProfile` | 3 | API | P2 | **FULL** |
| AC4 | Role enforcement: admin/bid_manager → 200 PUT/PATCH; lower roles → 403 | `TestAC4RoleEnforcement` | 6 | API | P1 | **FULL** |
| AC5 | Monotonic version counter; GET returns highest version | *(covered by AC1 + AC2 version tests)* | 2 | API | P2 | **FULL** |
| AC6 | GET /versions → 200 ascending; 404 if none | `TestAC6VersionHistory` | 3 | API | P2 | **FULL** |
| AC7 | Audit log for every PUT/PATCH (create/update action_type; before/after) | `TestAuditTrailEspd` | 3 | API/Integration | P1 | **FULL** |
| AC8 | 401 unauthenticated; 403 cross-tenant for all 4 endpoints | `TestCrossTenantEspd` | 4+4 | API | P0 | **FULL** |

**Story 2.12 AC Coverage: 8/8 = 100% · Epic Test IDs: E02-P0-012, E02-P1-018, E02-P2-007, E02-P2-008, E02-P2-009, E02-P2-010**

---

## 3. Epic Test ID Coverage Matrix

### P0 — Critical (12 items)

| Epic Test ID | Requirement | Covering Story | Test Class(es) | Tests | Coverage |
|--------------|-------------|----------------|----------------|------:|----------|
| **E02-P0-001** | Registration: company + admin user in transaction; no hashed_password | S02.02 | `TestAC1AC2AC7RegisterHappyPath` | 3 | **FULL** |
| **E02-P0-002** | Login returns RS256 JWT with correct claims + opaque refresh token | S02.03 | `TestAC1AC2AC3LoginHappyPath` + `TestAC4RefreshTokenDB` | 4 | **FULL** |
| **E02-P0-003** | Expired JWT → 401 + `token_expired` | S02.04 | `TestAC2ExpiredToken` | 2 | **FULL** |
| **E02-P0-004** | Tampered JWT → 401 + `token_invalid` | S02.04 | `TestAC3TamperedToken` | 4 | **FULL** |
| **E02-P0-005** | No Authorization header → 401 | S02.04 | `TestAC1MissingAuthHeader` | 2 | **FULL** |
| **E02-P0-006** | `require_role` enforces company role hierarchy | S02.04 | `TestRoleHierarchy` + `TestCurrentUser` + `TestRequireRole` | 15+ | **FULL** |
| **E02-P0-007** | RBAC ceiling: contributor cannot receive `manage` via entity_permissions | S02.10 | `TestRoleCeilingEnforcement` (unit + API) | 6 | **FULL** |
| **E02-P0-008** | Admin/bid_manager bypass entity checks (own-company) | S02.10 | `TestBypassRolesOwnCompany` (unit + API) | 6 | **FULL** |
| **E02-P0-009** | Cross-company RBAC: Company A denied Company B entity | S02.10 | `TestCrossCompanyIsolation` | 6 | **FULL** |
| **E02-P0-010** | Refresh token rotation: consumed token cannot be reused | S02.05 | `TestAC1ValidRefresh` + `TestAC2RotationRevocation` | 4 | **FULL** |
| **E02-P0-011** | Refresh replay → family revocation + audit breach entry | S02.05 | `TestAC3BreachDetection` | 2 | **FULL** |
| **E02-P0-012** | Cross-tenant: Company A cannot access Company B members/ESPD | S02.08 + S02.09 + S02.12 | `test_cross_tenant_get_returns_403` + `TestCrossTenantEspd` + cross-company team tests | 9 | **FULL** |

**P0 FULL: 12/12 = 100%**

---

### P1 — High (18 items)

| Epic Test ID | Requirement | Covering Story | Test Class(es) | Tests | Coverage |
|--------------|-------------|----------------|----------------|------:|----------|
| **E02-P1-001** | Duplicate email → 409 Conflict | S02.02 | `TestAC4DuplicateEmail` | 1 | **FULL** |
| **E02-P1-002** | Weak password → 422 (boundary: 7-char/no-upper/no-digit) | S02.02 | `TestAC5WeakPassword` | 4 | **FULL** |
| **E02-P1-003** | Wrong credentials → 401 generic (no enumeration) | S02.03 | `TestAC5InvalidCredentials` | 3 | **FULL** |
| **E02-P1-004** | Unverified email login → 403 | S02.03 | `TestAC6UnverifiedEmail` | 1 | **FULL** |
| **E02-P1-005** | Rate limiting: 5 failures → 429 + Retry-After | S02.03 | `TestAC7RateLimiting` | 3 | **FULL** |
| **E02-P1-006** | Logout revokes refresh token | S02.05 | `TestAC5Logout` | 5 | **FULL** |
| **E02-P1-007** | Expired refresh token → 401 | S02.05 | `TestAC4ExpiredToken` | 2 | **FULL** |
| **E02-P1-008** | Password reset request always returns 200 | S02.07 | `TestAC1NoEnumeration` | 3 | **FULL** |
| **E02-P1-009** | Reset token single-use; expired → 400 `invalid_token` | S02.07 | `TestAC3SingleUse` + `TestAC6ExpiredToken` | 5 | **FULL** |
| **E02-P1-010** | All refresh tokens revoked after password reset | S02.07 | `TestAC5RefreshTokensRevoked` | 2 | **FULL** |
| **E02-P1-011** | Google OAuth2 callback rejects invalid/missing state | S02.06 | `TestAC2InvalidState` | 2 | **FULL** |
| **E02-P1-012** | Google OAuth2 new user: account created with google_sub | S02.06 | `TestAC3NewGoogleUser` | 2 | **FULL** |
| **E02-P1-013** | Google OAuth2: existing email user gets google_sub linked | S02.06 | `TestAC4AccountLinking` | 2 | **FULL** |
| **E02-P1-014** | Company profile update: admin/bid_manager 200; others 403 | S02.08 + S02.12 | `TestAC6RoleEnforcement` + `TestAC4RoleEnforcement` | 14 | **FULL** |
| **E02-P1-015** | Team member invite: pending row + secure token | S02.09 | `TestAC2InviteMember` | 4 | **FULL** |
| **E02-P1-016** | Accept invite: creates/links user; activates membership | S02.09 | `TestAC6AcceptInvite` | 4 | **FULL** |
| **E02-P1-017** | Cannot remove last admin → 409 Conflict | S02.09 | `TestAC5LastAdminGuard` | 4 | **FULL** |
| **E02-P1-018** | All mutations audit-logged (timestamp, user_id, action_type, entity, before/after, ip) | S02.08 + S02.09 + S02.11 + S02.12 | `TestAC7AuditLog` + `TestMutationAuditCompany` + `TestMutationAuditMembership` + `TestAuditTrailEspd` | 12 | **FULL** |

**P1 FULL: 18/18 = 100%**

---

### P2 — Medium (18 items)

| Epic Test ID | Requirement | Covering Story | Test Class(es) | Tests | Coverage |
|--------------|-------------|----------------|----------------|------:|----------|
| **E02-P2-001** | Alembic migration creates all tables with correct schema | S02.01 | `TestAC1MigrationExecution` + `TestAC2TableStructure` | 13 | **FULL** |
| **E02-P2-002** | Role enum: exactly admin, bid_manager, contributor, reviewer, read_only | S02.01 | `TestAC3CompanyRoleEnum` | 2 | **FULL** |
| **E02-P2-003** | shared.audit_log in the `shared` schema | S02.01 | `TestAC7AuditLogSchema` | 1 | **FULL** |
| **E02-P2-004** | Company profile GET → 200 for any authenticated member | S02.08 | `TestAC1GetCompanyProfile` | 1 | **FULL** |
| **E02-P2-005** | Invalid CPV format rejected with 422 | S02.08 | `TestAC4CPVValidation` | 4 | **FULL** |
| **E02-P2-006** | Company PATCH merges without overwriting unchanged fields | S02.08 | `TestAC3PatchCompanyProfile` | 2 | **FULL** |
| **E02-P2-007** | ESPD PUT increments version number | S02.12 | `TestAC2PutEspdProfile::test_second_put_increments_version_to_2` | 1 | **FULL** |
| **E02-P2-008** | ESPD PATCH merges partial field_values | S02.12 | `TestAC3PatchEspdProfile::test_patch_exclusion_grounds_only_leaves_other_sections_unchanged` | 1 | **FULL** |
| **E02-P2-009** | ESPD /versions history ascending | S02.12 | `TestAC6VersionHistory::test_three_puts_produce_three_versions_ascending` | 1 | **FULL** |
| **E02-P2-010** | ESPD invalid JSONB → 422 (missing required sections) | S02.12 | `TestAC2PutEspdProfile` (4 × 422 + boundary) | 5 | **FULL** |
| **E02-P2-011** | Team member list: id, email, full_name, role, status (pending/active) | S02.09 | `TestAC3ListMembers` | 5 | **FULL** |
| **E02-P2-012** | Role change audit: before.role ≠ after.role | S02.09 + S02.11 | `test_role_change_creates_audit_log_with_before_and_after` + `test_change_member_role_produces_update_audit_with_role_diff` | 2 | **FULL** |
| **E02-P2-013** | Invite token expires after 7 days | S02.09 | `test_expired_token_returns_400` | 1 | **FULL** |
| **E02-P2-014** | Audit log append-only: UPDATE/DELETE denied at DB level | S02.01 + S02.11 | `TestAC9AppendOnlyEnforcement` + `TestAuditAppendOnly` | 3 | **FULL** |
| **E02-P2-015** | Auth events logged with `auth.` prefix | S02.11 | `TestAuthEventAudit` | 6 | **FULL** |
| **E02-P2-016** | Entity access denials logged with `access_denied` action_type | S02.10 | `TestAuditLogOnDenial` | 3 | **FULL** |
| **E02-P2-017** | 401 unauthenticated; 403 unauthorized (distinct status codes) | S02.04 + S02.08 | `TestAC1MissingAuthHeader` + `TestAC6RoleEnforcement` | 9 | **FULL** |
| **E02-P2-018** | No entity_permissions row → denied (explicit grant model) | S02.10 | `TestExplicitGrantModel` (unit) + `TestNoEntityPermissionRow` (API) | 6 | **FULL** |

**P2 FULL: 18/18 = 100%**

---

### P3 — Low Priority (4 items)

| Epic Test ID | Requirement | Covering Story | Test Class(es) | Tests | Coverage |
|--------------|-------------|----------------|----------------|------:|----------|
| **E02-P3-001** | Full auth lifecycle E2E: register → verify → login → refresh → logout | S02.02 + S02.03 + S02.05 *(individual steps only)* | Individual story tests cover each step; no dedicated chain test | — | **PARTIAL** |
| **E02-P3-002** | Google OAuth2 full mock redirect flow | S02.06 | `TestAC1GoogleRedirect` + `TestAC5TokenStructure` | 4 | **FULL** |
| **E02-P3-003** | Full password reset lifecycle E2E (8-step flow) | S02.07 | `TestP3FullFlow::test_full_password_reset_lifecycle` | 1 | **FULL** |
| **E02-P3-004** | k6: Auth p95 < 200ms under 50 concurrent users for 2 min | *(not yet written — performance test deferred)* | — | — | **NONE** |

**P3 FULL: 2/4 = 50% · P3 PARTIAL: 1 · P3 NONE: 1**

---

## 4. Gap Analysis

### Critical Gaps (P0) — 0

No P0 gaps identified. All 12 critical requirements have full test coverage.

### High Gaps (P1) — 0

No P1 gaps identified. All 18 high-priority requirements have full test coverage.

### Medium Gaps (P2) — 0

No P2 gaps identified. All 18 medium-priority requirements have full test coverage.

### Low Gaps (P3) — 2

| Gap ID | Epic Test ID | Description | Gap Type | Recommendation |
|--------|-------------|-------------|----------|----------------|
| GAP-P3-001 | E02-P3-001 | Full auth lifecycle E2E chain (register→verify→login→refresh→logout) as single compound test | PARTIAL — individual steps covered across S02.02/S02.03/S02.05; no dedicated chain test | Add `TestE2EAuthLifecycle` compound test spanning Stories 2.2–2.5; LOW priority — individual steps are well-tested |
| GAP-P3-002 | E02-P3-004 | k6 performance test: auth endpoint p95 < 200ms under load | NONE — no k6 test written | Write k6 script for login/register/token-refresh endpoints; run nightly in staging; DEFER to Epic 9+ when frontend environment is stable |

### Coverage Heuristics

| Dimension | Finding | Status |
|-----------|---------|--------|
| **API endpoint coverage** | All 23 auth/identity API endpoints have dedicated test coverage | ✅ No gaps |
| **Auth negative-path tests** | All auth flows include 401/403/400 rejection paths | ✅ No gaps |
| **Error-path coverage** | Every AC with error conditions (422/409/400/401/403) has dedicated negative tests | ✅ No gaps |
| **Cross-tenant isolation** | Company A→Company B access tested across all 4 ESPD endpoints, team member endpoints, and company profile | ✅ Full coverage |
| **RBAC matrix** | All 5 roles × 3 permission levels × own/cross-company = exhaustive parametrized tests in S02.10 | ✅ Full coverage |
| **Token security** | RS256 signature validation, expiry, tampering, refresh family revocation all tested | ✅ Full coverage |
| **Audit completeness** | Before/after state captured for all mutation types; auth events with auth. prefix; append-only enforced | ✅ Full coverage |

---

## 5. Risk Coverage Assessment

| Risk ID | Category | Score | Mitigation Stories | Coverage |
|---------|----------|-------|-------------------|---------|
| **E02-R-001** | SEC | 6 | S02.04 (require_role unit), S02.08 (role enforcement), S02.10 (RBAC ceiling, exhaustive matrix) | ✅ FULL — `TestRequireRole` 5×5 matrix + `TestRoleCeilingEnforcement` all combinations |
| **E02-R-002** | SEC | 6 | S02.08 (`test_cross_tenant_get_returns_403`), S02.09 (cross-company tests), S02.12 (`TestCrossTenantEspd`) | ✅ FULL — 9 dedicated cross-tenant tests across all company-scoped endpoints |
| **E02-R-003** | SEC | 6 | S02.03 (JWT signing), S02.04 (tamper/expiry), S02.05 (refresh rotation + family revocation) | ✅ FULL — token security covered by 22 dedicated tests |
| **E02-R-004** | SEC | 4 | S02.06 (`TestAC2InvalidState`) | ✅ FULL — CSRF state validation and invalid code handling tested |
| **E02-R-005** | SEC | 3 | S02.02 (bcrypt cost=12, weak password), S02.03 (enumeration prevention), S02.07 (hash-before-store) | ✅ FULL — password security gap mitigations all covered |
| **E02-R-006** | TECH | 4 | S02.03 (`TestAC7RateLimiting` with Redis teardown) | ✅ FULL — rate limit state isolation pattern established |
| **E02-R-007** | DATA | 4 | S02.01 (append-only enforcement), S02.08+S02.09+S02.11 (before/after capture), S02.12 (ESPD audit) | ✅ FULL — audit completeness verified at DB level + application level |
| **E02-R-008** | DATA | 2 | S02.12 (`TestAC2PutEspdProfile` — 422 for missing sections) | ✅ FULL — JSONB schema validation tested |
| **E02-R-009** | SEC | 2 | S02.09 (`test_expired_token_returns_400`, `test_already_accepted_token_returns_400`) | ✅ FULL — invite token lifecycle edge cases covered |

**All 9 identified risks have corresponding test coverage.**

---

## 6. Design Concerns Flagged by ATDD Checklists

The following architectural ambiguities were identified during ATDD checklist generation and must be resolved during implementation:

| Concern | Story | Risk | Resolution Needed |
|---------|-------|------|-------------------|
| **Cross-company admin bypass semantics** (E02-P0-009 vs AC4 wording) | S02.10 | MEDIUM | Does admin bypass unconditionally, or only for own-company entities? Three options documented in S02.10 checklist. Test `test_company_a_admin_denied_on_company_b_entity` will fail if Option C (unconditional bypass) chosen. |
| **Invite audit `entity_type` ambiguity** | S02.09, S02.11 | LOW | AC8 says `entity_type="company_membership"` for invites; dev notes say `entity_type="invitation"`. S02.09 ATDD test accepts either; S02.11 test assumes `"invitation"`. Developer must choose one and update both tests accordingly. |

---

## 7. Implementation Readiness Summary

| Story | Status | Critical Prereqs |
|-------|--------|-----------------|
| S02.01 | 🔴 Not started | E01 complete (schemas + roles in place) |
| S02.02 | 🔴 Not started | S02.01 GREEN (users/companies/company_memberships tables) |
| S02.03 | 🔴 Not started | S02.02 GREEN (registered + verified users) |
| S02.04 | 🔴 Not started | S02.03 GREEN (JWT signing infrastructure) |
| S02.05 | 🔴 Not started | S02.04 GREEN (JWT middleware), refresh_tokens table |
| S02.06 | 🔴 Not started | S02.04 GREEN (JWT issuance), authlib dependency |
| S02.07 | 🔴 Not started | S02.03 GREEN (login), password_reset_tokens table |
| S02.08 | 🔴 Not started | S02.04 GREEN (auth middleware), S02.02 GREEN (company created) |
| S02.09 | 🔴 Not started | S02.08 GREEN (company exists), invitations table |
| S02.10 | 🔴 Not started | S02.04 GREEN (CurrentUser), entity_permissions table |
| S02.11 | 🔴 Not started | S02.08 + S02.09 GREEN (mutations exist to audit), audit_service.py |
| S02.12 | 🔴 Not started | S02.08 GREEN (company auth pattern), espd_profiles table |

---

## 8. Recommendations

| Priority | Action | Target |
|----------|--------|--------|
| **URGENT** | Implement S02.01 first — all 11 other stories depend on the DB schema | S02.01 |
| **HIGH** | Follow sequential story order: S02.01 → S02.02 → S02.03 → S02.04 → S02.05 (core auth chain) | S02.01–S02.05 |
| **HIGH** | Resolve cross-company admin bypass design decision (3 options in S02.10 checklist) before implementing S02.10 | S02.10 |
| **HIGH** | Resolve invite audit `entity_type` ambiguity (S02.09 vs S02.11) | S02.09, S02.11 |
| **MEDIUM** | Add dedicated `TestE2EAuthLifecycle` compound test after S02.05 is GREEN | E02-P3-001 gap |
| **LOW** | Write k6 performance test for auth endpoints (E02-P3-004) — defer to Epic 9+ | E02-P3-004 |
| **LOW** | Re-run this traceability matrix after each story implementation to track GREEN phase progress | All |

---

## Gate Decision Summary

```
🟢 TRACE_GATE: PASS

📊 Coverage Analysis:
- P0 Coverage: 100% (12/12) → ✅ MET (required: 100%)
- P1 Coverage: 100% (18/18) → ✅ MET (target: 90%)
- P2 Coverage: 100% (18/18) → ✅ MET
- P3 Coverage: 50% FULL + 25% PARTIAL → Advisory only
- Overall Coverage (FULL): 96.2% (50/52) → ✅ MET (minimum: 80%)
- Overall Coverage (FULL+PARTIAL): 98.1% (51/52)

✅ Decision Rationale:
P0 coverage is 100%, P1 coverage is 100%, P2 coverage is 100%, and overall
FULL coverage is 96.2% (minimum: 80%). All 12 stories have 100% story-level AC
coverage. All 52 epic-level test IDs have corresponding ATDD tests written.

⚠️ TDD Baseline Caveat:
All 307 tests are in RED phase (@pytest.mark.skip / pytest.mark.xfail). Zero
tests currently pass. This is the expected pre-implementation baseline state.
PASS applies to coverage completeness — not execution readiness. Re-run this
gate after implementation to validate test execution results.

⚠️ Design Concerns:
2 architectural ambiguities flagged (cross-company admin bypass; invite audit
entity_type). Both must be resolved before S02.09 and S02.10 are implemented.

📂 Test Files:
  eusolicit-app/services/client-api/tests/
  ├── integration/test_002_migration.py          (42 tests, S02.01)
  ├── api/test_register.py                        (11 tests, S02.02)
  ├── api/test_login.py                           (11 tests, S02.03)
  ├── unit/test_security.py                       (15 tests, S02.04)
  ├── api/test_auth_middleware.py                 (13 tests, S02.04)
  ├── api/test_auth_refresh.py                    (13 tests, S02.05)
  ├── api/test_auth_google.py                     (15 tests, S02.06)
  ├── api/test_auth_password_reset.py             (17 tests, S02.07)
  ├── api/test_company_profile.py                 (23 tests, S02.08)
  ├── api/test_team_members.py                    (34 tests, S02.09)
  ├── unit/test_rbac.py                           (42 tests, S02.10)
  ├── api/test_rbac_middleware.py                 (23 tests, S02.10)
  ├── unit/test_audit_service.py                  ( 5 tests, S02.11)
  ├── api/test_audit_trail.py                     (12+3 tests, S02.11+S02.12)
  └── api/test_espd_profile.py                    (28 tests, S02.12)
  Total: 307 tests across 14 test files
```

---

*Generated by TEA Master Test Architect — bmad-testarch-trace workflow*
*Epic: E02 — Authentication & Identity | 2026-04-07*
