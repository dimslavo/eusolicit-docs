# Epic 1: Workspace & Access Management

Users can register securely, manage profiles, roles, and navigate between multiple isolated tenant workspaces as a consulting firm.

### Story 1.1: User Registration
As a prospective customer,
I want to register with email and password or Google OAuth,
So that I can evaluate EU Solicit.

**Acceptance Criteria:**
**Given** I am on the registration page
**When** I register with email and password or Google OAuth
**Then** the API returns 201 within 300 ms and a Stripe customer provisioning runs as BackgroundTask
**And** an email verification message is dispatched and cross-tenant access returns 404

### Story 1.2: Login, Refresh, Logout
As a user,
I want secure session management,
So that my account remains safe while I use the application.

**Acceptance Criteria:**
**Given** I am a registered user
**When** I log in with my credentials
**Then** I receive an RS256 JWT access token and a rotating refresh token
**And** User.is_active is checked on every protected endpoint and token refresh, and logout revokes the active refresh family

### Story 1.3: Company Roles & Memberships
As an admin,
I want to invite teammates and assign roles,
So that I can control access to the platform.

**Acceptance Criteria:**
**Given** I am an admin of a company
**When** I assign roles or entity-level overrides
**Then** the roles (admin, bid_manager, contributor, reviewer, read_only) are enforced
**And** removing a user revokes all entity overrides and an audit row is written

### Story 1.4: Multi-Client Workspace
As a consulting firm,
I want to manage multiple isolated client workspaces under one parent company,
So that I can keep client data separate while managing billing centrally.

**Acceptance Criteria:**
**Given** I am a parent company
**When** I create multiple child workspaces
**Then** branding is per workspace and cross-workspace queries are filtered by workspace_id
**And** cross-workspace access returns 404, billing rolls up to the parent, and usage is recorded per workspace
