---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-07'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 2
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-02-authentication-identity.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
---

# Test Design: Epic 2 — Authentication & Identity

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**Status:** Draft

---

## Executive Summary

**Scope:** Epic-level test design for E02 — Authentication & Identity. Covers email/password registration, JWT RS256 issuance and validation, token refresh and revocation (with family tracking), Google OAuth2 social login, password reset, company profile CRUD, team member management, entity-level RBAC middleware, audit trail middleware, and ESPD profile CRUD. 12 stories, 34 points, Sprints 1–2. This epic is the security foundation that every downstream epic depends on.

**Risk Summary:**

- Total risks identified: 9
- High-priority risks (≥6): 3 (RBAC ceiling enforcement, cross-tenant isolation, token security)
- Critical categories: SEC (authentication, RBAC, token security, data isolation), DATA (audit completeness), TECH (test isolation for stateful rate limiting)

**Coverage Summary:**

- P0 scenarios: 12 (~20–35 hours)
- P1 scenarios: 18 (~15–25 hours)
- P2 scenarios: 18 (~10–18 hours)
- P3 scenarios: 4 (~4–8 hours)
- **Total effort:** ~50–86 hours (~1.5–2.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **Frontend login/register UI** | E02 is backend API only; Next.js frontend covered in E09 | E2E login flow deferred to E09 test design |
| **Microsoft OAuth2** | Deferred post-MVP per ADR 1.8 | Google OAuth2 + email/password covered in this epic |
| **Email delivery validation** | Email sending is stubbed in E02; SendGrid integration in Notification epic | Token generation and validation tested; delivery verified in E07 |
| **Stripe subscription creation on registration** | Subscription skeleton table created in S02.01; billing logic in E05 | ESPD profile + company registration tested; billing deferred |
| **Entity-level permissions for non-auth entities** | `entity_permissions` table created in S02.01; full usage tested when entities exist (E03+) | RBAC middleware dependency injection tested with mocked entity IDs |
| **ESPD XML compliance** | ESPD Pydantic model covers top-level sections only; full 200+ field XML spec out of MVP scope | JSONB schema validation and version history tested |
| **Concurrent token refresh races** | Pessimistic family lock acceptable for MVP scale | Token family revocation logic tested sequentially |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E02-R-001** | **SEC** | RBAC ceiling enforcement failure — 5 company roles × 3 permission levels × multiple entity types; any gap in ceiling logic allows a lower-privileged role to gain `manage` access they should not have | 2 | 3 | **6** | Exhaustive parametrized permission matrix: all 5 roles × {read, write, manage} × own-company/cross-company; verify contributor cannot exceed `write` ceiling; verify `read_only` cannot exceed `read` | Backend / QA | Sprint 1–2 |
| **E02-R-002** | **SEC** | Cross-tenant isolation via missing `company_id` scoping — a single missing filter in auth queries exposes Company B's users, memberships, ESPD profiles, or entity permissions to Company A | 2 | 3 | **6** | Automated cross-tenant sweep: create Company A + Company B; with Company A credentials attempt to GET Company B users, memberships, permissions, ESPD profiles → all return 403 or 404 | Backend / QA | Sprint 1–2 |
| **E02-R-003** | **SEC** | Token security — RS256 private key mishandling, refresh token replay (stolen token reused before legitimate rotation), or family tracking gaps allow session hijacking without revocation | 2 | 3 | **6** | Test tampered signatures (401 + token_invalid), expired tokens (401 + token_expired), refresh token reuse after rotation (family revocation triggered), logout revocation; verify revoked tokens cannot access protected endpoints | Backend / QA | Sprint 1–2 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E02-R-004 | SEC | Google OAuth2 CSRF — authlib `state` parameter not validated correctly in callback allows cross-site request forgery | 2 | 2 | 4 | Test callback with missing/invalid state → verify rejection; test expired authorization code | Backend |
| E02-R-005 | SEC | Password security gaps — weak bcrypt cost factor, plaintext reset token stored in DB, or user enumeration via password reset endpoint | 1 | 3 | 3 | Verify bcrypt cost=12; verify reset token hash before storage (SHA-256); verify request endpoint always returns 200 | Backend |
| E02-R-006 | TECH | Rate limiting Redis state leakage between tests — sliding window counters persist across test runs causing false 429s or masking actual rate limit failures | 2 | 2 | 4 | Flush rate limit keys in test teardown; use unique email addresses per test run; test isolation fixture | QA |
| E02-R-007 | DATA | Audit log completeness — missing before/after snapshots on update/delete operations; append-only not enforced at DB level allowing audit log tampering | 2 | 2 | 4 | Assert before-state captured on every PUT/PATCH/DELETE; assert after-state captured on POST/PUT/PATCH; verify application code cannot UPDATE/DELETE audit_log rows | Backend / DBA |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E02-R-008 | DATA | ESPD JSONB schema gaps — malformed `field_values` structure accepted without validation, stored without key sections | 1 | 2 | 2 | Test invalid JSONB (missing required top-level keys) rejected with 422 |
| E02-R-009 | SEC | Invite token lifecycle — tokens accepted after expiry (>7 days) or reused after acceptance | 1 | 2 | 2 | Test expired invite token returns 400/410; test accepted invite token cannot be used again |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

### System-Level Risk Inheritance

- E02-R-001 (RBAC ceiling) → extends system R-006 (entity-level RBAC enforcement, score 6). E02 implements the two-tier RBAC that R-006 identified as a platform-wide high risk.
- E02-R-002 (cross-tenant isolation) → extends system R-002 (multi-tenant data isolation, score 6). E02's `company_id` scoping is the first concrete implementation of the data boundary established by the DB schema (E01-R-001).

---

## Entry Criteria

- [ ] Epic 2 stories reviewed and accepted by team
- [ ] E01 complete: PostgreSQL schemas + roles in place; shared packages importable
- [ ] RSA key pair fixture generated and available for test suite (RS256 signing/verification)
- [ ] Redis available in test Docker Compose environment (rate limiting)
- [ ] authlib Google OAuth2 mock/stub strategy agreed (unit mock vs. integration stub)
- [ ] Email stub captures verification + reset tokens without delivery (e.g., test SMTP capture or in-memory stub)
- [ ] Test data seeding API (TB-01) available for user/company state creation, or pytest factories provide equivalent

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95% or failures triaged and accepted)
- [ ] No open high-severity auth/RBAC bugs
- [ ] Cross-tenant isolation sweep: Company A cannot access any Company B resource
- [ ] RBAC permission matrix: all 5 roles × all permission levels verified
- [ ] Audit log verified for all mutation types (create/update/delete)
- [ ] Branch coverage ≥90% on auth and RBAC modules (per epic AC)

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on if time-constrained), NOT execution timing. See "Execution Strategy" for when tests run.

### P0 (Critical)

**Criteria:** Blocks core auth/security functionality + High risk (≥6) + No workaround + All downstream epics depend on it

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E02-P0-001** | Registration creates company + admin user in single transaction, returns correct structure (no password field) | API | E02-R-002 | POST /auth/register → 201; verify `user`, `company` in response; `hashed_password` absent; DB state: company_memberships row with role=admin |
| **E02-P0-002** | Login returns RS256-signed access token (15 min) with correct claims + opaque refresh token | API | E02-R-003 | POST /auth/login with valid creds → verify JWT header alg=RS256; decode + assert sub, company_id, role, exp (now+15min), iat, jti present; refresh_token non-null |
| **E02-P0-003** | Expired JWT rejected with 401 + `token_expired` error code | API | E02-R-003 | Generate token with exp=now-1min → GET /opportunities → 401; body.error_code = "token_expired" |
| **E02-P0-004** | Tampered JWT rejected with 401 + `token_invalid` error code | API | E02-R-003 | Modify JWT payload (e.g., change role) without re-signing → 401; body.error_code = "token_invalid" |
| **E02-P0-005** | Requests without Authorization header rejected with 401 | API | E02-R-003 | GET any protected endpoint with no header → 401 |
| **E02-P0-006** | `require_role` enforces company role hierarchy — lower roles rejected | Unit | E02-R-001 | Parametrized: require_role("bid_manager") rejects contributor, reviewer, read_only; passes admin, bid_manager |
| **E02-P0-007** | RBAC ceiling — contributor cannot receive `manage` permission via entity_permissions | API | E02-R-001 | Grant entity_permissions(user=contributor, permission=manage) → check_entity_access returns 403; verify role ceiling map |
| **E02-P0-008** | Admin and bid_manager bypass entity-level checks for own-company entities | API | E02-R-001 | Grant no entity_permissions for admin/bid_manager → check_entity_access passes for all permission levels on own-company entity |
| **E02-P0-009** | Cross-company RBAC — Company A user denied access to Company B entity | API | E02-R-001, E02-R-002 | User from Company A calls check_entity_access on Company B entity_id → 403 regardless of entity_permissions state |
| **E02-P0-010** | Refresh token rotation — consumed token cannot be reused | API | E02-R-003 | POST /auth/refresh with valid token → new pair issued + old token is_revoked=true; repeat POST with old token → 401 |
| **E02-P0-011** | Refresh token replay triggers family revocation (breach detection) | API | E02-R-003 | Simulate replay: rotate token legitimately → attempt replay with original token → verify all tokens in family revoked; audit_log entry with action_type="auth.token_breach" |
| **E02-P0-012** | Cross-tenant isolation — Company A cannot access Company B's users, memberships, or ESPD profile | API | E02-R-002 | Create Company A + Company B; with Company A token attempt GET/PATCH on Company B's /members, /espd-profile → 403 or 404 for all endpoints |

**Total P0:** 12 tests, ~20–35 hours

---

### P1 (High)

**Criteria:** Important auth flows + Medium risk (3–5) + Common user paths + Workaround exists but painful

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E02-P1-001** | Duplicate email registration returns 409 Conflict | API | — | POST /auth/register with existing email → 409; existing user unchanged |
| **E02-P1-002** | Weak password rejected on registration (min 8 chars, 1 uppercase, 1 digit) | API | E02-R-005 | Boundary tests: 7-char, all-lowercase, no-digit passwords → 422; valid password → 201 |
| **E02-P1-003** | Invalid credentials return 401 with generic error (no user enumeration) | API | E02-R-005 | Wrong password → 401; non-existent email → 401; same generic error body for both |
| **E02-P1-004** | Unverified email login returns 403 with clear message | API | — | Login before email verification → 403; body indicates email verification required |
| **E02-P1-005** | Login rate limiting — 5 failed attempts per email per 15-min window → 429 | API | E02-R-006 | Send 5 failed logins → 6th → 429 with Retry-After header; successful login resets counter |
| **E02-P1-006** | Logout revokes the current refresh token | API | E02-R-003 | POST /auth/logout with valid refresh_token → is_revoked=true in DB; subsequent /auth/refresh with that token → 401 |
| **E02-P1-007** | Expired refresh token returns 401 | API | E02-R-003 | Present refresh token with expired expiry → 401 |
| **E02-P1-008** | Password reset request always returns 200 regardless of email existence | API | E02-R-005 | POST /auth/password-reset/request with non-existent email → 200; with existing email → 200; response body identical |
| **E02-P1-009** | Password reset token is single-use; expired token rejected | API | E02-R-005 | Use reset token once → 200; use same token again → 400; advance time past 1h expiry → 400 |
| **E02-P1-010** | All refresh tokens revoked after successful password reset | API | E02-R-003 | Obtain refresh token → reset password → attempt /auth/refresh with original refresh token → 401 |
| **E02-P1-011** | Google OAuth2 callback rejects invalid or missing state parameter | API | E02-R-004 | GET /auth/google/callback with no state → error redirect; with wrong state → error redirect |
| **E02-P1-012** | Google OAuth2 — new user creates account with google_sub populated, no password | API | E02-R-004 | Mock Google userinfo response → callback creates user with google_sub set; hashed_password null; tokens issued |
| **E02-P1-013** | Google OAuth2 — existing email user gets google_sub linked | API | E02-R-004 | Existing email/password user → Google OAuth with same email → google_sub linked; tokens issued; no duplicate user |
| **E02-P1-014** | Company profile update — admin/bid_manager succeeds; contributor/reviewer/read_only receive 403 | API | E02-R-001 | Parametrized by role: PUT /companies/{id} with each of 5 roles; verify 200 for admin+bid_manager, 403 for others |
| **E02-P1-015** | Team member invite — creates pending membership row, generates secure invite token | API | — | POST /companies/{id}/members/invite (admin role) → 201; company_memberships row with accepted_at=null; invite token generated |
| **E02-P1-016** | Accept invite — creates or links user, activates membership | API | E02-R-009 | POST /auth/accept-invite with valid token → membership accepted_at set; user created or linked; role assigned |
| **E02-P1-017** | Cannot remove last admin from a company → 409 Conflict | API | — | DELETE /companies/{id}/members/{admin_user_id} when only one admin → 409 |
| **E02-P1-018** | Audit trail — all POST/PUT/PATCH/DELETE mutations logged with timestamp, user_id, action_type, entity_type, entity_id, before, after, ip_address | API | E02-R-007 | Create/update/delete company profile, ESPD profile, membership → verify audit_log row for each; before captured on update/delete; after captured on create/update |

**Total P1:** 18 tests, ~15–25 hours

---

### P2 (Medium)

**Criteria:** Secondary flows + Low risk (1–2) + Edge cases + Schema/migration validation

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E02-P2-001** | Alembic migration creates all E02 tables with correct column types, constraints, and indexes | Unit | — | Check `users`, `companies`, `company_memberships`, `entity_permissions`, `espd_profiles`, `shared.audit_log`, `refresh_tokens`, token tables exist with correct schemas |
| **E02-P2-002** | Role enum includes exactly: admin, bid_manager, contributor, reviewer, read_only | Unit | — | Assert enum values against DB or SQLAlchemy model |
| **E02-P2-003** | `shared.audit_log` lives in the `shared` schema | Unit | — | Verify table schema via `information_schema.tables` |
| **E02-P2-004** | Company profile GET returns full profile for any authenticated company member | API | — | Contributor, reviewer, read_only can GET /companies/{id} → 200 |
| **E02-P2-005** | Company profile — invalid CPV code format rejected with 422 | API | — | PUT /companies/{id} with malformed CPV code (not `\d{8}-\d`) → 422 |
| **E02-P2-006** | Company profile PATCH merges partial updates without overwriting unchanged fields | API | — | PATCH with only `description` → other fields unchanged in DB; returns merged profile |
| **E02-P2-007** | ESPD profile PUT replaces and increments version number | API | — | PUT /companies/{id}/espd-profile twice → second version = first version + 1 |
| **E02-P2-008** | ESPD profile PATCH merges partial field_values updates | API | — | PATCH with only `exclusion_grounds` → other sections unchanged |
| **E02-P2-009** | ESPD profile version history queryable via /espd-profile/versions | API | — | After 3 PUTs → GET /versions returns 3 entries with ascending version numbers |
| **E02-P2-010** | ESPD profile — invalid JSONB schema rejected with 422 | API | E02-R-008 | PUT with missing required top-level keys (exclusion_grounds, economic_standing, technical_ability, quality_assurance) → 422 |
| **E02-P2-011** | Team member list returns id, email, full_name, role, status (pending/active) for all members | API | — | GET /companies/{id}/members → each member has all required fields; status reflects accepted_at state |
| **E02-P2-012** | Role change audit-logged with old and new role values | API | E02-R-007 | PATCH /companies/{id}/members/{user_id} with new role → audit_log.before.role ≠ audit_log.after.role |
| **E02-P2-013** | Invite token expires after 7 days | API | E02-R-009 | Present invite token with created_at > 7 days ago → 400 or 410 |
| **E02-P2-014** | Audit log append-only — application code cannot UPDATE or DELETE audit_log rows | Unit | E02-R-007 | Attempt UPDATE/DELETE on shared.audit_log via app DB role → permission denied (DB-level enforcement) |
| **E02-P2-015** | Auth events logged with `auth.` prefix (login, logout, password_reset, access_denied) | API | E02-R-007 | Trigger each auth event → audit_log.action_type starts with "auth." |
| **E02-P2-016** | Entity access denials logged with action_type `access_denied` | API | E02-R-001 | Trigger check_entity_access failure → audit_log row with action_type="access_denied", entity_type, entity_id present |
| **E02-P2-017** | Unauthenticated requests return 401; unauthorized requests return 403 | API | — | No token → 401; valid token + insufficient role → 403; verify status codes are distinct |
| **E02-P2-018** | No entity_permissions row → access denied (explicit grant model) | Unit | E02-R-001 | check_entity_access with no entity_permissions row for user → denied for contributor/reviewer/read_only |

**Total P2:** 18 tests, ~10–18 hours

---

### P3 (Low)

**Criteria:** Nice-to-have + E2E journeys + Performance benchmarks

| Test ID | Requirement | Test Level | Notes |
|---------|-------------|------------|-------|
| **E02-P3-001** | Full registration → email verification → login → token refresh → logout E2E flow | API | Happy path chain; verifies all auth lifecycle steps connect correctly |
| **E02-P3-002** | Google OAuth2 full redirect flow with mocked Google IDP | API | authlib mock server; redirect → callback → tokens issued → frontend redirect verified |
| **E02-P3-003** | Full password reset flow: request → stub-captured token → confirm → old refresh tokens revoked | API | Verifies end-to-end reset lifecycle without email delivery |
| **E02-P3-004** | k6: Auth endpoint latency — p95 < 200ms for login, register, token refresh under load | Perf | 50 concurrent users sustained 2 min; alerts if p95 breached |

**Total P3:** 4 tests, ~4–8 hours

---

## Execution Strategy

**Philosophy:** Run everything in PRs if <15 min; defer only if expensive or long-running.

### Every PR: pytest + Playwright API Tests (~10–15 min)

All functional tests from P0–P3 (excluding k6) run on every PR:
- Unit tests (RBAC logic, middleware, schema validation): pytest with pytest-asyncio
- API integration tests (auth flows, token lifecycle, CRUD, audit log): pytest against Docker Compose client-api service
- Parametrized RBAC matrix tests: pytest-parametrize across all 5 roles × 3 permissions
- Parallelized via pytest-xdist workers

### Nightly: k6 Performance Tests

- E02-P3-004: Auth endpoint latency under load (~10–20 min)
- Requires staging environment + k6 runner

### Weekly: Security Regression

- Token tampering matrix (all JWT manipulation vectors)
- Rate limit stress test (Redis state reset + sustained failure sequence)
- Cross-tenant sweep automation across all auth endpoints

---

## Resource Estimates

### Test Development Effort

| Priority | Count | Effort Range | Notes |
|----------|-------|-------------|-------|
| P0 | 12 | ~20–35 hours | Complex setup: RS256 key fixtures, token manipulation, RBAC parametrized matrix, cross-tenant seeding |
| P1 | 18 | ~15–25 hours | Auth flows, OAuth mock, rate limiting with Redis teardown, invite lifecycle |
| P2 | 18 | ~10–18 hours | Migration validation, CRUD edge cases, audit log assertions |
| P3 | 4 | ~4–8 hours | E2E auth chains + k6 script |
| **Total** | **52** | **~49–86 hours** | **~1.5–2.5 weeks (1 QA)** |

### Prerequisites

**Test Fixtures Required:**
- RSA key pair for RS256 signing (private key for token generation, public key for service config)
- Test SMTP capture or in-memory email stub (intercepts verification tokens + reset tokens)
- authlib Google OAuth2 mock (stub `GET_TOKEN` + `USERINFO` responses)
- Redis flush fixture for rate limiting test isolation
- User/company factory: `UserFactory`, `CompanyFactory`, `MembershipFactory` (from eusolicit-test-utils or seeding API)

**Tooling:**
- pytest + pytest-asyncio + pytest-parametrize (backend unit/API tests)
- httpx AsyncClient (API assertions against Docker Compose client-api)
- PyJWT (token construction and inspection in tests)
- redis-py (rate limit key flush in teardown)
- k6 (performance baseline — P3-004)

**Environment:**
- Docker Compose with client-api + PostgreSQL + Redis running
- E01 migrations applied before E02 tests run
- Test-specific environment variables: RSA_PRIVATE_KEY, RSA_PUBLIC_KEY, GOOGLE_CLIENT_ID, GOOGLE_CLIENT_SECRET (stubbed values)

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions — security foundation)
- **P1 pass rate:** ≥95% (waivers require tech lead sign-off)
- **P2/P3 pass rate:** ≥90% (informational)
- **SEC tests (E02-R-001, E02-R-002, E02-R-003):** 100% — these are non-negotiable before E03 begins

### Coverage Targets

- **Auth and RBAC modules:** ≥90% branch coverage (per epic AC: "Unit and integration tests cover all auth flows with >90% branch coverage")
- **Security scenarios (cross-tenant, RBAC matrix, token integrity):** 100%

### Non-Negotiable Requirements

- [ ] All P0 tests pass
- [ ] RBAC permission matrix verified for all 5 roles × all 3 permissions × own/cross-company
- [ ] No open SEC-category bugs before E03 epic begins
- [ ] Audit log append-only enforcement verified at DB level

---

## Mitigation Plans

### E02-R-001: RBAC Ceiling Enforcement Failure (Score: 6)

**Mitigation Strategy:**
1. Define role-to-max-permission mapping in `app/core/rbac.py`: `{ admin: manage, bid_manager: manage, contributor: write, reviewer: read, read_only: read }`
2. Parametrized test matrix: `@pytest.mark.parametrize("role,permission,own_company,expected")` covering all 15 combinations (5 roles × 3 permissions)
3. Separate test for cross-company: user with any role cannot access any other company's entities regardless of entity_permissions grants
4. Admin/bid_manager bypass: verify both roles can perform `manage` on own-company entities without an entity_permissions row

**Owner:** Backend / QA
**Timeline:** Sprint 1–2
**Status:** Planned
**Verification:** P0-006, P0-007, P0-008, P0-009; P2-016, P2-018

### E02-R-002: Cross-Tenant Isolation via Missing company_id Scope (Score: 6)

**Mitigation Strategy:**
1. All auth service queries include `WHERE company_id = :current_company_id` — reviewed in PR
2. Automated cross-tenant sweep at epic completion: Company A token + Company B resource IDs → verify all protected endpoints return 403 or 404
3. Test covers: GET /companies/{b_id}, GET /companies/{b_id}/members, GET /companies/{b_id}/espd-profile, PATCH /companies/{b_id}/members/{b_user_id}

**Owner:** Backend / QA
**Timeline:** Sprint 1–2
**Status:** Planned
**Verification:** P0-012; P0-009 (entity RBAC cross-company)

### E02-R-003: Token Security — RS256 + Refresh Token Family Tracking (Score: 6)

**Mitigation Strategy:**
1. RSA key management: private key loaded from environment at startup; never logged or returned in responses
2. Token tampering tests: modify `.` segments without re-signing → assert 401 token_invalid
3. Refresh token family: each token assigned `family_id` at login; rotation creates new token in same family; reuse of revoked token → revoke entire family + audit_log breach entry
4. Expiry tests: use PyJWT to generate tokens with `exp=now-1` in test fixtures; assert 401 token_expired

**Owner:** Backend / QA
**Timeline:** Sprint 1–2
**Status:** Planned
**Verification:** P0-003, P0-004, P0-010, P0-011; P1-006, P1-007, P1-010

---

## Assumptions and Dependencies

### Assumptions

1. E01 is complete: `shared` schema exists, PostgreSQL DB roles created, Redis available
2. RSA key pair generated before implementation begins (not generated at runtime in tests)
3. authlib state parameter is stored server-side (e.g., Redis with short TTL) for CSRF protection — test assumes this design
4. Email sending stubbed via interface: `EmailService.send(to, subject, body)` — tests capture the token from the service call, not email delivery
5. `entity_permissions` rows can be inserted directly in test fixtures (seeding API or factory) before RBAC checks

### Dependencies

| Dependency | Required By | Blocks |
|-----------|-------------|--------|
| E01 complete (DB schemas, shared package, Redis) | Sprint 1 | All E02 tests |
| TB-01 test data seeding API (or pytest factories for users/companies) | Sprint 1 | P0, P1 auth flow tests |
| RSA key pair test fixture available | Sprint 1 | P0-002, P0-003, P0-004 |
| authlib Google OAuth2 mock strategy agreed | Sprint 2 | P1-011, P1-012, P1-013; P3-002 |
| Redis available in Docker Compose test environment | Sprint 1 | P1-005 (rate limiting) |

### Risks to Plan

- **Risk:** authlib integration with Google requires live HTTP to Google's discovery endpoint on import
  - **Impact:** Unit tests fail unless discovery URL is mocked or `requests_mock` fixture applied
  - **Contingency:** Use `responses` or `pytest-httpx` to stub `accounts.google.com/.well-known/openid-configuration` in test setup

- **Risk:** RS256 key rotation mid-sprint invalidates all issued tokens
  - **Impact:** Running test suite fails if keys change between CI runs
  - **Contingency:** Pin test RSA keys in fixtures (test-only keys, not production); separate CI secret management for test vs. production keys

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|-------------------|--------|-----------------|
| **client-api auth routes** | Primary test target — all E02 endpoints | Full P0 + P1 + P2 suite on every PR |
| **RBAC middleware** (`app/core/rbac.py`) | Used by every downstream epic's protected routes | Permission matrix parametrized tests; re-run when new entity types added |
| **Audit trail service** (`app/services/audit_service.py`) | Every mutation in every epic triggers audit log | P1-018, P2-012, P2-014, P2-015 form regression suite for audit correctness |
| **JWT middleware** (`app/core/security.py`) | Consumed by all 5 services for token validation | P0-003, P0-004, P0-005 are platform-wide regression tests |
| **company_memberships + entity_permissions tables** | E03+ use these tables for access control | Schema and constraint tests (P2-001, P2-002) prevent drift |
| **shared.audit_log** | Cross-epic compliance requirement | Append-only enforcement (P2-014) verified at DB level; regression on any schema change |

**Regression strategy:** Every PR runs the full pytest suite for client-api (~10–15 min). RBAC matrix tests are parametrized — adding a new entity type or permission level automatically extends coverage. Auth tests tagged `@pytest.mark.p0` and `@pytest.mark.auth` run first in CI for fast failure detection.

---

## Appendix A: Code Examples

### pytest — RS256 test fixtures

```python
# tests/conftest.py or services/client-api/tests/conftest.py
import pytest
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.backends import default_backend
import jwt
from datetime import datetime, timedelta, timezone

@pytest.fixture(scope="session")
def rsa_private_key():
    return rsa.generate_private_key(
        public_exponent=65537,
        key_size=2048,
        backend=default_backend(),
    )

@pytest.fixture(scope="session")
def rsa_public_key(rsa_private_key):
    return rsa_private_key.public_key()

@pytest.fixture
def valid_access_token(rsa_private_key):
    payload = {
        "sub": "user-uuid-123",
        "company_id": "company-uuid-abc",
        "role": "bid_manager",
        "exp": datetime.now(timezone.utc) + timedelta(minutes=15),
        "iat": datetime.now(timezone.utc),
        "jti": "token-uuid-xyz",
    }
    return jwt.encode(payload, rsa_private_key, algorithm="RS256")

@pytest.fixture
def expired_access_token(rsa_private_key):
    payload = {
        "sub": "user-uuid-123",
        "company_id": "company-uuid-abc",
        "role": "bid_manager",
        "exp": datetime.now(timezone.utc) - timedelta(minutes=1),
        "iat": datetime.now(timezone.utc) - timedelta(minutes=16),
        "jti": "expired-token-xyz",
    }
    return jwt.encode(payload, rsa_private_key, algorithm="RS256")
```

### pytest — RBAC permission matrix

```python
# tests/api/test_rbac_middleware.py
import pytest
from httpx import AsyncClient

ROLE_PERMISSION_MATRIX = [
    # (role, permission, own_company, expected_status)
    ("admin",       "manage", True,  200),
    ("admin",       "write",  True,  200),
    ("admin",       "read",   True,  200),
    ("bid_manager", "manage", True,  200),
    ("bid_manager", "write",  True,  200),
    ("bid_manager", "read",   True,  200),
    ("contributor", "write",  True,  200),  # ceiling: write
    ("contributor", "read",   True,  200),
    ("contributor", "manage", True,  403),  # above ceiling
    ("reviewer",    "read",   True,  200),  # ceiling: read
    ("reviewer",    "write",  True,  403),  # above ceiling
    ("reviewer",    "manage", True,  403),
    ("read_only",   "read",   True,  200),  # ceiling: read
    ("read_only",   "write",  True,  403),  # above ceiling
    ("read_only",   "manage", True,  403),
    # Cross-company: all roles denied regardless of permission
    ("admin",       "manage", False, 403),
    ("bid_manager", "read",   False, 403),
]

@pytest.mark.p0
@pytest.mark.parametrize("role,permission,own_company,expected", ROLE_PERMISSION_MATRIX)
async def test_rbac_ceiling_enforcement(
    client: AsyncClient,
    role: str,
    permission: str,
    own_company: bool,
    expected: int,
    seed_entity_with_permission,
):
    token, entity_id = await seed_entity_with_permission(role=role, permission=permission, own_company=own_company)
    response = await client.get(
        f"/api/v1/entities/{entity_id}/check?permission={permission}",
        headers={"Authorization": f"Bearer {token}"},
    )
    assert response.status_code == expected
```

### pytest — Token replay attack (family revocation)

```python
# tests/api/test_token_refresh.py
import pytest
from httpx import AsyncClient

@pytest.mark.p0
@pytest.mark.auth
async def test_refresh_token_replay_revokes_family(
    client: AsyncClient,
    registered_user: dict,
):
    # Step 1: Login — get original refresh token
    login_resp = await client.post("/api/v1/auth/login", json={
        "email": registered_user["email"],
        "password": registered_user["password"],
    })
    original_refresh_token = login_resp.json()["refresh_token"]

    # Step 2: Legitimate rotation — get new refresh token
    rotate_resp = await client.post("/api/v1/auth/refresh", json={"refresh_token": original_refresh_token})
    assert rotate_resp.status_code == 200
    new_refresh_token = rotate_resp.json()["refresh_token"]
    assert new_refresh_token != original_refresh_token

    # Step 3: Replay attack — reuse the original (now revoked) token
    replay_resp = await client.post("/api/v1/auth/refresh", json={"refresh_token": original_refresh_token})
    assert replay_resp.status_code == 401

    # Step 4: Verify family revocation — new token also unusable
    new_token_resp = await client.post("/api/v1/auth/refresh", json={"refresh_token": new_refresh_token})
    assert new_token_resp.status_code == 401

    # Step 5: Verify breach logged
    # (Assert audit_log row with action_type="auth.token_breach" for this user)
```

### pytest tag usage

```bash
# Run only P0 auth tests
pytest -m "p0 and auth" services/client-api/tests/

# Run RBAC matrix
pytest -m "p0 and rbac" services/client-api/tests/ -v

# Run full E02 suite
pytest -m "auth or rbac or audit" services/client-api/tests/

# Run security tests only
pytest -m "security" services/client-api/tests/
```

---

## Appendix B: Knowledge Base References

- `risk-governance.md` — Risk classification framework (P × I scoring, gate decisions)
- `probability-impact.md` — Risk scoring methodology (1–3 scales)
- `test-levels-framework.md` — Test level selection (E2E vs API vs Unit)
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

---

## Related Documents

- Epic: `eusolicit-docs/planning-artifacts/epic-02-authentication-identity.md`
- System-Level Test Design (Architecture): `eusolicit-docs/test-artifacts/test-design-architecture.md`
- System-Level Test Design (QA): `eusolicit-docs/test-artifacts/test-design-qa.md`
- Architecture: `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md`
- Prior Epic Test Design: `eusolicit-docs/test-artifacts/test-design-epic-01.md`

### Existing Test Coverage (Pre-E02)

| Area | Files | Status |
|------|-------|--------|
| Unit test infrastructure (E01) | eusolicit-app/tests/unit/ — 28 unit test files | E01 infra tests passing; no auth coverage |
| Service test scaffolds | 5 service conftest.py + empty unit/integration/api directories | Fixtures built; test directories empty for all services |
| eusolicit-test-utils | Factories, ServiceClient, DB helpers, Redis helpers, JWT generation | Available for E02 test development |
| Auth-specific tests | None found | **Gap** — all E02 tests are new |

---

**Generated by:** BMad TEA Agent — Test Architect Module
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.1 (BMad v6)
