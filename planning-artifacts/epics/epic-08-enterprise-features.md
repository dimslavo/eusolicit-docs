# Epic 8: Enterprise & API Features

This epic focuses on features for large organizations, including the ability to manage multiple client workspaces, an external REST API for data integration, and advanced user management.

**FRs covered:** FR-8, FR-9, FR-44

## Stories

### Story 8.1: Multiple Client Workspaces
As an enterprise admin,
I want to manage multiple, isolated client workspaces under a single parent company account,
So that my consulting firm can manage bids on behalf of different clients within one platform.

**Acceptance Criteria:**
**Given** I am an admin on an Enterprise plan
**When** I navigate to my organization settings
**Then** I can create new client workspaces, each with its own isolated data and user permissions.

### Story 8.2: Workspace Switching
As a user belonging to multiple workspaces,
I want to be able to easily switch between them,
So that I can work on proposals for different companies or clients seamlessly.

**Acceptance Criteria:**
**Given** my user account is associated with more than one workspace
**When** I click on my user profile or a workspace switcher UI element
**Then** I can see a list of my workspaces and select one to switch to, updating my entire session context.

### Story 8.3: External REST API
As a developer at an enterprise customer,
I want to access my company's data (like opportunities and proposals) via a secure REST API,
So that I can integrate EU Solicit with our internal systems, like our CRM.

**Acceptance Criteria:**
**Given** I am on an Enterprise plan
**When** I generate an API key from my account settings
**Then** I can use this key to authenticate with the EU Solicit API and retrieve data according to the provided documentation.
