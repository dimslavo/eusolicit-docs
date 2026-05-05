---
epic: 1
title: User Onboarding and Identity
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-1", "FR-2", "FR-3", "FR-4", "FR-5"]
nfrs_relevant: ["NFR-5", "NFR-6", "NFR-7", "NFR-18", "NFR-19", "NFR-20"]
ux_drs_relevant: []
---

# Epic 1: User Onboarding and Identity

**Phase**: MVP | **Dependencies**: Infrastructure foundation (E01), Auth service (E02), Frontend shell (E03)

## Goal

Users can securely register, log in, verify their email, and manage their individual and company profiles so they can begin using the platform with a complete, accurate identity footprint that downstream services (matching, ESPD, billing, RBAC) depend on.

## User Outcome

After this epic, a brand-new visitor can self-serve through the entire onboarding funnel: create an account by email/password or Google, verify their address, set up their company profile (CPV codes, sectors, key personnel), and arrive at an authenticated dashboard ready to discover opportunities.

## Epic Acceptance Criteria

- [ ] A new user can complete the entire registration → verification → company-profile flow without manual intervention from EU Solicit staff.
- [ ] All authentication is enforced via RS256 JWT with short-lived access tokens and securely stored refresh tokens (NFR-5).
- [ ] Google OAuth registration creates a linked identity with no separate password prompt and reuses verified email status from Google (FR-2).
- [ ] Email verification is mandatory before access to AI features, opportunity detail data, and billing actions (FR-4).
- [ ] Company profile completeness is visible to the user as a meter that updates as fields are filled (FR-5).
- [ ] All forms meet WCAG 2.1 AA: keyboard-navigable, screen-reader-labelled, contrast-compliant (NFR-18, NFR-19, NFR-20).
- [ ] Cross-tenant data leakage is impossible — every authenticated request resolves to a single tenant context enforced by middleware and RLS-equivalent guards (NFR-7).

## Stories

### Story 1.1: User Registration with Email and Password

As a new user,
I want to register using my email and password,
So that I can create an account on the platform.

**FR coverage**: FR-1

**Acceptance Criteria:**

**Given** I am on the registration page
**When** I enter a valid email, a secure password (meeting strength rules), and submit the form
**Then** an account is created in `client.users` with `is_active=false` until email verification completes
**And** a verification email is dispatched via the notification service.
**And** weak or reused passwords are rejected client-side and server-side with clear error messages.

---

### Story 1.2: User Registration with Google OAuth

As a new user,
I want to register using my Google account,
So that I can quickly sign up without managing a new password.

**FR coverage**: FR-2

**Acceptance Criteria:**

**Given** I am on the registration page
**When** I choose "Sign up with Google" and complete the Google consent screen
**Then** a `client.users` row is created linked to the Google `sub` identifier
**And** the email is auto-marked verified if Google reports it verified
**And** I am logged into the platform with a valid RS256 JWT.

---

### Story 1.3: User Login and Logout

As a registered user,
I want to log in and out of my account securely,
So that my session and data remain protected.

**FR coverage**: FR-3
**NFR coverage**: NFR-5

**Acceptance Criteria:**

**Given** I have an active, verified account
**When** I submit valid credentials on the login page
**Then** the backend issues a short-lived access token and a refresh token via httpOnly secure cookie
**And** I am redirected to the dashboard.
**Given** I am logged in
**When** I click logout
**Then** the refresh token is revoked server-side
**And** subsequent requests with the prior access token are rejected after expiry.

---

### Story 1.4: Email Verification

As a newly registered user,
I want to verify my email address,
So that I can access the core features of the platform.

**FR coverage**: FR-4

**Acceptance Criteria:**

**Given** I have received a verification email containing a single-use token
**When** I click the verification link
**Then** my user record is marked `email_verified_at = now()`
**And** I gain access to gated routes (opportunity details, AI features, billing).
**Given** the token is expired or already consumed
**When** I click the link
**Then** I am shown an error and offered to request a fresh token.

---

### Story 1.5: Company Profile Management

As an admin user,
I want to create and update my company profile,
So that the platform can use my company's capabilities and sectors for matching, ESPD generation, and AI personalisation.

**FR coverage**: FR-5

**Acceptance Criteria:**

**Given** I am an authenticated admin user with a company workspace
**When** I navigate to Company Profile settings
**Then** I can input company legal name, VAT, sectors, CPV codes (multi-select), key personnel, and certifications
**And** I can save partial progress.
**And** a profile completeness meter reflects which fields are populated and weighted by impact.
**And** the profile data is persisted in `client.companies` and is consumable by the AI relevance scoring service.

## Implementation Notes

- Use `auth_service` from `client-api` with bcrypt password hashing and the existing JWT signing infra.
- Email verification tokens stored in `client.email_verifications` with 24h expiry and single-use semantics.
- Google OAuth uses the configured `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` and PKCE flow.
- Frontend uses `useZodForm` + `<FormField>` wrappers; client+server validation must match.
- All endpoints negative-tested: anonymous → 401, wrong tenant → 403, unverified email accessing gated route → 403 with `verification_required` error code.
