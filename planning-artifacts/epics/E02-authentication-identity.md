# E02: Authentication & Identity

**Sprint**: 1--2 (Weeks 1--4) | **Points**: 34 | **Dependencies**: E01 | **Milestone**: Demo

## Goal

Deliver the full authentication and identity layer for the EU Solicit platform, enabling companies to register, authenticate via email/password or Google OAuth2, manage their profiles and team members, and operate under an entity-level RBAC model. All mutations are recorded in a shared audit log. JWT RS256 tokens (15-min access / 7-day refresh) secure every downstream API call. This epic produces the security foundation that every other epic depends on.

## Acceptance Criteria

- [ ] A new company + admin user can be created via the registration endpoint
- [ ] Email/password login returns a signed RS256 access token (15 min) and refresh token (7 days)
- [ ] Google OAuth2 redirect flow creates or links a user account and issues tokens
- [ ] The refresh endpoint rotates tokens and invalidates the consumed refresh token
- [ ] Password reset issues a time-limited email token and allows password change
- [ ] Company profile CRUD works for all specified fields (name, tax_id, address, industry/CPV sectors, regions, description, certifications)
- [ ] Admin users can invite team members by email and assign company-wide roles (admin, bid_manager, contributor, reviewer, read_only)
- [ ] RBAC middleware resolves effective permission by intersecting company role ceiling with entity_permissions grants
- [ ] ESPD profile CRUD stores and retrieves structured JSONB field values per company
- [ ] Every mutation is logged to shared.audit_log with timestamp, user_id, action_type, entity_type, entity_id, before/after JSONB
- [ ] All endpoints reject unauthenticated requests with 401 and unauthorized requests with 403
- [ ] Unit and integration tests cover all auth flows with >90% branch coverage

## Stories

### S02.01: Database Schema -- Auth & Identity Tables

**Points**: 3 | **Type**: backend
**Description**: Create Alembic migrations for the core auth and identity tables: `users` (id, email, hashed_password, full_name, google_sub, is_active, email_verified, created_at, updated_at), `companies` (id, name, tax_id, address JSONB, industry, cpv_sectors JSONB, regions JSONB, description, certifications JSONB, created_at, updated_at), `company_memberships` (user_id, company_id, role enum, invited_at, accepted_at), `subscriptions` (id, company_id, plan, status, current_period_start, current_period_end -- skeleton), `entity_permissions` (id, user_id, company_id, entity_type, entity_id, permission enum, granted_by, created_at), `espd_profiles` (id, company_id, field_values JSONB, version, created_at, updated_at), and `shared.audit_log` (id, timestamp, user_id, action_type, entity_type, entity_id, before JSONB, after JSONB, ip_address).
**Acceptance Criteria**:
- [ ] Alembic migration runs cleanly on a fresh database
- [ ] All tables exist with correct column types, constraints, and indexes
- [ ] Foreign key relationships are correct (users <-> companies via company_memberships)
- [ ] Role enum includes: admin, bid_manager, contributor, reviewer, read_only
- [ ] `shared.audit_log` lives in the `shared` schema
**Implementation Notes**: Use SQLAlchemy models in `app/models/`. Create the `shared` schema in a base migration if not present from E01. Add indexes on `users.email` (unique), `entity_permissions(user_id, entity_type, entity_id)`, and `audit_log.entity_type + entity_id`.

---

### S02.02: Email/Password Registration

**Points**: 3 | **Type**: backend
**Description**: Implement `POST /api/v1/auth/register` which creates a new company and its first admin user in a single transaction. Accepts email, password, full_name, and company_name. Hashes password with bcrypt. Sends an email verification link (stub the email sender behind an interface). Returns the created user (without password) and company.
**Acceptance Criteria**:
- [ ] Endpoint creates both user and company in one transaction
- [ ] User is assigned `admin` role in `company_memberships`
- [ ] Password is hashed with bcrypt (cost factor 12)
- [ ] Duplicate email returns 409 Conflict
- [ ] Validation rejects weak passwords (min 8 chars, 1 uppercase, 1 digit)
- [ ] Email verification token is generated and logged (email sending is stubbed)
- [ ] Response excludes hashed_password
**Implementation Notes**: Use Pydantic schemas for request/response validation. Place auth routes in `app/api/v1/auth.py`. Use a `services/auth_service.py` to keep business logic out of the route handler. Email verification token: URL-safe random token stored in a `verification_tokens` table or Redis with 24h TTL.

---

### S02.03: Email/Password Login & JWT Issuance

**Points**: 3 | **Type**: backend
**Description**: Implement `POST /api/v1/auth/login` which validates credentials and returns RS256-signed JWT access and refresh tokens. Access token contains user_id, company_id, and company role. Refresh token is opaque, stored in the database with expiry metadata.
**Acceptance Criteria**:
- [ ] Valid credentials return `{ access_token, refresh_token, token_type, expires_in }`
- [ ] Access token is RS256-signed with 15-minute expiry
- [ ] Access token claims include: sub (user_id), company_id, role, exp, iat, jti
- [ ] Refresh token is stored in DB with user_id, expiry, and is_revoked flag
- [ ] Invalid credentials return 401 with generic error (no user enumeration)
- [ ] Unverified email returns 403 with clear message
- [ ] Rate limiting: max 5 failed attempts per email per 15-minute window
**Implementation Notes**: Use PyJWT for RS256 signing. Load RSA keys from environment (private key for signing, public key for verification). Store refresh tokens in a `refresh_tokens` table. Rate limiting can use Redis or an in-memory sliding window (Redis preferred for multi-worker).

---

### S02.04: JWT Authentication Middleware

**Points**: 2 | **Type**: backend
**Description**: Create a FastAPI dependency (`get_current_user`) that extracts the Bearer token from the Authorization header, verifies the RS256 signature and claims, and returns the authenticated user context. Create a secondary dependency (`require_role`) that checks company-level role minimums.
**Acceptance Criteria**:
- [ ] Requests without Authorization header receive 401
- [ ] Expired tokens receive 401 with `token_expired` error code
- [ ] Tampered tokens receive 401 with `token_invalid` error code
- [ ] Valid token injects `CurrentUser` (user_id, company_id, role) into the request
- [ ] `require_role("bid_manager")` rejects users with lower roles (contributor, reviewer, read_only)
- [ ] Role hierarchy is enforced: admin > bid_manager > contributor > reviewer > read_only
**Implementation Notes**: Place in `app/core/security.py`. Use `fastapi.Depends` for injection. Cache the public key in memory at startup. Consider adding the dependency to an APIRouter-level `dependencies` list for protected route groups.

---

### S02.05: Token Refresh & Revocation

**Points**: 2 | **Type**: backend
**Description**: Implement `POST /api/v1/auth/refresh` which accepts a refresh token, validates it, issues a new access + refresh token pair, and revokes the consumed refresh token (rotation). Implement `POST /api/v1/auth/logout` which revokes the current refresh token.
**Acceptance Criteria**:
- [ ] Valid refresh token returns new access + refresh token pair
- [ ] Consumed refresh token is marked revoked and cannot be reused
- [ ] If a revoked refresh token is presented, revoke all tokens for that user (breach detection)
- [ ] Expired refresh token returns 401
- [ ] Logout revokes the provided refresh token
- [ ] Refresh token reuse detection logs a security warning to audit_log
**Implementation Notes**: Implement refresh token family tracking: each refresh token stores a `family_id`. On reuse of a revoked token, revoke all tokens in the same family. This handles the scenario where an attacker replays a stolen refresh token after the legitimate user has already rotated it.

---

### S02.06: Google OAuth2 Social Login

**Points**: 3 | **Type**: backend
**Description**: Implement the Google OAuth2 redirect flow using authlib. `GET /api/v1/auth/google` redirects to Google's consent screen. `GET /api/v1/auth/google/callback` exchanges the authorization code for user info, creates or links the user account, and issues JWT tokens. Redirect the browser to the frontend with tokens in a short-lived query parameter or fragment.
**Acceptance Criteria**:
- [ ] `/auth/google` redirects to Google with correct scopes (openid, email, profile)
- [ ] Callback validates the `state` parameter to prevent CSRF
- [ ] New Google user creates a user record with `google_sub` populated and no password
- [ ] Existing user with matching email links the Google account (sets `google_sub`)
- [ ] Tokens are issued identically to email/password login
- [ ] If the user has no company, they are prompted to create one (frontend redirect with flag)
- [ ] Invalid or expired authorization codes return a user-friendly error redirect
**Implementation Notes**: Use `authlib.integrations.starlette_client`. Store Google OAuth client_id and client_secret in environment. The callback should set an HTTP-only secure cookie or redirect to `{frontend_url}/auth/callback?token=...` depending on the agreed frontend contract.

---

### S02.07: Password Reset Flow

**Points**: 2 | **Type**: backend
**Description**: Implement `POST /api/v1/auth/password-reset/request` (accepts email, always returns 200 to prevent enumeration, sends reset link if user exists) and `POST /api/v1/auth/password-reset/confirm` (accepts token + new password, resets the password).
**Acceptance Criteria**:
- [ ] Request endpoint always returns 200 regardless of whether the email exists
- [ ] Reset token is cryptographically random, URL-safe, and expires in 1 hour
- [ ] Reset token is single-use (marked consumed after successful reset)
- [ ] Confirm endpoint validates token, enforces password policy, and updates the hash
- [ ] All existing refresh tokens for the user are revoked after password reset
- [ ] Invalid or expired token returns 400 with clear message
**Implementation Notes**: Store reset tokens in a `password_reset_tokens` table (token_hash, user_id, expires_at, consumed_at). Hash the token before storage (SHA-256) so a DB leak does not expose valid reset links. The email body is stubbed but the token generation and validation logic must be production-ready.

---

### S02.08: Company Profile CRUD

**Points**: 3 | **Type**: backend
**Description**: Implement endpoints for managing company profiles: `GET /api/v1/companies/{id}`, `PUT /api/v1/companies/{id}`, `PATCH /api/v1/companies/{id}`. Only admin and bid_manager roles may update. Fields: name, tax_id, address (structured JSONB: street, city, postal_code, country), industry, cpv_sectors (array), regions (array), description, certifications (array of objects: name, issuer, valid_until).
**Acceptance Criteria**:
- [ ] GET returns the full company profile for any authenticated member
- [ ] PUT replaces the entire profile (all required fields must be present)
- [ ] PATCH merges partial updates into the existing profile
- [ ] Only admin and bid_manager roles can update (others receive 403)
- [ ] CPV sector codes are validated against a known set (or at minimum against format `^\d{8}-\d$`)
- [ ] Address validation ensures required sub-fields are present
- [ ] All mutations are audit-logged with before/after snapshots
**Implementation Notes**: Use `app/api/v1/companies.py`. Pydantic models with nested schemas for address and certifications. The audit log middleware (S02.11) handles logging automatically, but ensure the before-state is captured before the update.

---

### S02.09: Team Member Management

**Points**: 3 | **Type**: backend
**Description**: Implement endpoints for managing team members within a company: `POST /api/v1/companies/{id}/members/invite` (send invitation email with role assignment), `GET /api/v1/companies/{id}/members` (list members), `PATCH /api/v1/companies/{id}/members/{user_id}` (change role), `DELETE /api/v1/companies/{id}/members/{user_id}` (remove member). `POST /api/v1/auth/accept-invite` accepts an invitation token and creates/links the user.
**Acceptance Criteria**:
- [ ] Only admin role can invite, change roles, or remove members
- [ ] Invitation creates a pending `company_memberships` row and generates a secure invite token
- [ ] Invite acceptance creates a new user (if needed) and activates the membership
- [ ] List endpoint returns members with id, email, full_name, role, status (pending/active)
- [ ] Role change is audit-logged with old and new role
- [ ] Cannot remove the last admin from a company (returns 409)
- [ ] Invite tokens expire after 7 days
**Implementation Notes**: Store invite tokens in an `invitations` table (token_hash, company_id, email, role, expires_at, accepted_at). The invitation email is stubbed. When accepting, if a user with that email already exists, add them to the company; otherwise create a new user record and prompt for password setup.

---

### S02.10: Entity-Level RBAC Middleware

**Points**: 3 | **Type**: backend
**Description**: Implement a FastAPI dependency `check_entity_access(entity_type, entity_id, required_permission)` that resolves effective permissions by checking the `entity_permissions` table. The company role sets the ceiling (e.g., a `read_only` user cannot gain `write` via entity_permissions). Permissions: read, write, manage. Admin and bid_manager roles bypass entity checks for their own company's entities.
**Acceptance Criteria**:
- [ ] Dependency can be composed into any route: `Depends(check_entity_access("opportunity", opp_id, "write"))`
- [ ] If no entity_permissions row exists, access is denied (explicit grant model)
- [ ] Company role ceiling is enforced: contributor can receive read/write but not manage
- [ ] Admin and bid_manager bypass entity-level checks for own-company entities
- [ ] Permission check runs in a single optimized query (JOIN company_memberships and entity_permissions)
- [ ] Access denials are logged to audit_log with action_type `access_denied`
- [ ] Caching: entity permissions are cached per-request (not across requests) to avoid repeated DB hits within a single API call
**Implementation Notes**: Place in `app/core/rbac.py`. Define a role-to-max-permission mapping: `{ admin: manage, bid_manager: manage, contributor: write, reviewer: read, read_only: read }`. The middleware takes `entity_type` and `entity_id` as parameters and queries for a matching grant where the permission level is >= the required level and <= the role ceiling.

---

### S02.11: Audit Trail Middleware

**Points**: 3 | **Type**: backend
**Description**: Implement a FastAPI middleware or dependency that automatically logs all state-changing requests (POST, PUT, PATCH, DELETE) to `shared.audit_log`. Captures the before-state (for updates/deletes) and after-state (for creates/updates) as JSONB, along with user_id, action_type, entity_type, entity_id, timestamp, and client IP.
**Acceptance Criteria**:
- [ ] All POST/PUT/PATCH/DELETE requests on tracked resources are logged
- [ ] Before-state is captured for update and delete operations
- [ ] After-state is captured for create and update operations
- [ ] Log entry includes: timestamp (UTC), user_id, action_type (create/update/delete), entity_type, entity_id, before, after, ip_address
- [ ] Audit writes are non-blocking (async insert or background task) and do not fail the parent request
- [ ] Audit log is append-only (no UPDATE or DELETE allowed on the table via application code)
- [ ] Auth events (login, logout, password_reset, access_denied) are also logged with action_type prefixed by `auth.`
**Implementation Notes**: Implement as a composable service (`app/services/audit_service.py`) that route handlers or other services call explicitly, rather than a global middleware (which cannot easily access entity-level before/after state). Use `BackgroundTasks` to write audit entries asynchronously. For auth events, call the audit service directly from auth_service.py. Enforce append-only at the DB level with a trigger or by granting only INSERT on the audit_log table to the application role.

---

### S02.12: ESPD Profile CRUD

**Points**: 3 | **Type**: backend
**Description**: Implement endpoints for managing a company's ESPD (European Single Procurement Document) profile: `GET /api/v1/companies/{id}/espd-profile`, `PUT /api/v1/companies/{id}/espd-profile`, `PATCH /api/v1/companies/{id}/espd-profile`. The ESPD profile stores pre-mapped field values as structured JSONB, allowing companies to maintain a reusable set of answers for procurement exclusion/selection criteria.
**Acceptance Criteria**:
- [ ] GET returns the current ESPD profile with all field values
- [ ] PUT replaces the entire ESPD profile (version is incremented)
- [ ] PATCH merges partial updates into existing field_values JSONB
- [ ] Only admin and bid_manager roles can update (others receive 403)
- [ ] Profile is versioned: each update increments the version number
- [ ] JSONB field_values conform to a defined schema (validated on write) covering at minimum: exclusion_grounds, economic_standing, technical_ability, quality_assurance
- [ ] Previous versions are queryable via `GET /api/v1/companies/{id}/espd-profile/versions`
- [ ] All mutations are audit-logged
**Implementation Notes**: Use `app/api/v1/espd.py`. The ESPD field schema should be defined as a Pydantic model that mirrors the ESPD XML structure at a high level (not the full 200+ field spec -- focus on the top-level sections with freeform JSONB within each). Store versions by inserting a new row on each update rather than overwriting, using the `version` column as a monotonic counter. The latest version is the one with the highest version number for that company_id.
