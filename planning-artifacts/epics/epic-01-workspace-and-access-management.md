# Epic 1: Workspace & Access Management

Enable consulting firms to create, manage, and isolate client workspaces with appropriate RBAC.

### Story 1.1: Create and Manage Client Workspaces

As a consulting-firm tenant admin,
I want to create, rename, archive, and delete client workspaces,
So that I can segregate data and access for my different clients.

**Acceptance Criteria:**

**Given** I am logged in with the `tenant_admin` role
**When** I navigate to the workspace management section and create a new workspace
**Then** a new workspace entity is created and linked to my tenant
**And** I can subsequently rename, archive, or delete this workspace.

### Story 1.2: Workspace Switcher and Context Persistence

As a user belonging to multiple workspaces,
I want to use a top-level workspace switcher to navigate between my client engagements,
So that my active context is preserved and my actions are scoped to the right workspace.

**Acceptance Criteria:**

**Given** I am a user with access to multiple workspaces
**When** I use the workspace switcher component in the Next.js frontend
**Then** the active workspace state is persisted in the URL and Zustand store
**And** all subsequent API calls and dashboard views automatically scope to the selected workspace.

### Story 1.3: Invite External Collaborator via Magic Link

As a Bid Manager,
I want to invite my end-client to review a draft proposal as a comment-only collaborator via a magic link,
So that they can review drafts securely without consuming a paid seat.

**Acceptance Criteria:**

**Given** I am a Bid Manager inside a specific workspace
**When** I generate an invite for an external collaborator for a specific proposal
**Then** the system generates a 30-day expiring magic link with a `comment_only` role
**And** upon access, the external collaborator can only view and comment on the shared proposal without seeing other workspace data.

### Story 1.4: Enforce Workspace Data Isolation

As a system operator,
I want robust row-level scoping and middleware validation on all workspace API endpoints,
So that cross-workspace and cross-tenant data leaks are impossible.

**Acceptance Criteria:**

**Given** the backend API serves multiple workspaces
**When** a user requests data for a workspace
**Then** the system validates `verify_workspace_access` dependency to ensure the user has appropriate permissions
**And** cross-tenant and cross-workspace negative tests return 403 Forbidden for unauthorized access attempts.