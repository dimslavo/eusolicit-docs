## Epic 14: Multi-Client Workspace Data Model & RBAC

Establish a multi-client workspace data model with per-client isolation, RBAC, and audit-trail scoping so consulting firms can manage multiple client engagements concurrently.
**FRs covered:** FR8.1, FR8.2, FR8.3, FR8.4, FR8.5, FR8.6

### Story 14.1: Workspace CRUD & Data Model

As a Tenant Admin,
I want to create, read, update, and archive client workspaces,
So that I can manage my firm's active client engagements separately.

**Acceptance Criteria:**

**Given** a user is logged in with `tenant_admin` role
**When** they submit a request to create a new workspace
**Then** the workspace is created under their tenant
**And** the workspace appears in the workspace switcher

### Story 14.2: RBAC Extension for Workspaces

As a Tenant Admin,
I want to assign roles to users per workspace,
So that team members only access the client engagements they are assigned to.

**Acceptance Criteria:**

**Given** a user is assigned a `contributor` role in workspace A and no role in workspace B
**When** the user attempts to access an opportunity in workspace B
**Then** the system returns a 403 Forbidden error
**And** the access attempt is logged in the tenant's audit trail

### Story 14.3: External Collaborator Magic-Link Auth

As a Bid Manager,
I want to invite my end-client as a comment-only collaborator via a magic link,
So that they can review the draft without consuming a paid seat.

**Acceptance Criteria:**

**Given** a Bid Manager creates an external invite for a specific proposal
**When** the end-client clicks the magic link
**Then** they are authenticated into the workspace with `comment_only` role
**And** the link expires after 30 days or upon its first use (if configured for one-time use)

### Story 14.4: Cross-Workspace Search for Admin

As a Tenant Admin,
I want to search across all workspaces,
So that I can reuse content from past successful bids regardless of the client engagement.

**Acceptance Criteria:**

**Given** multiple workspaces exist with proposals
**When** a `tenant_admin` performs a cross-workspace search
**Then** results from all accessible workspaces are returned
**And** results indicate which workspace they originate from
