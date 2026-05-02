---
epicNumber: 8
title: Enterprise & API Features
status: draft
dependencies: [1, 2, 4]
frsCovered: [FR-8, FR-9, FR-44]
nfrsRelevant: [NFR-1, NFR-5, NFR-6, NFR-7, NFR-9, NFR-11, NFR-13]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
phase: post-mvp
---

## Epic 8: Enterprise & API Features

This epic enables Enterprise customers to manage multiple isolated client workspaces under a single parent organisation, switch between workspaces seamlessly, and access platform data programmatically via a documented REST API with API key management. After this epic, Journey 4 (the Enterprise developer integrating with Salesforce) is fully unblocked.

### Story 8.1: Multiple Client Workspaces under a Parent Organisation

As an enterprise admin,
I want to manage multiple, isolated client workspaces under a single parent organisation,
So that my consulting firm can manage bids on behalf of different clients within one platform.

**Acceptance Criteria:**

**Given** my organisation is on the Enterprise plan
**When** I navigate to "Organisation Settings" -> "Workspaces"
**Then** I can create a new workspace with its own name, slug, branding, and isolated data
**And** each workspace has its own `company_id` linked by `parent_organisation_id`
**And** data isolation is enforced at the database layer (no cross-workspace queries possible from application roles).

**Given** I am an enterprise admin
**When** I assign users to specific workspaces with workspace-scoped roles
**Then** users see only the workspaces they have been granted access to
**And** removing a user from a workspace immediately revokes their access on next request.

### Story 8.2: Workspace Switching for Multi-Workspace Users

As a user belonging to multiple workspaces,
I want to switch between them,
So that I can work on proposals for different companies/clients seamlessly.

**Acceptance Criteria:**

**Given** my user account is associated with more than one workspace
**When** I open the workspace switcher in the global header
**Then** I see a list of my workspaces with last-active timestamps and quick-search
**And** selecting a workspace swaps my entire session context (active `company_id`, role, branding, recent items) within 500ms
**And** my JWT is reissued with the new active workspace claim.

**Given** I switch workspaces
**When** I navigate around the app
**Then** all in-flight requests are cancelled and re-issued under the new workspace
**And** browser-tab-scoped state (open editors, modals) is cleared to prevent cross-workspace data leakage.

### Story 8.3: Enterprise REST API with API Key Management

As a developer at an Enterprise customer,
I want to access my workspace's data via a secure REST API,
So that I can integrate EU Solicit with our internal systems (e.g., CRM, BI).

**Acceptance Criteria:**

**Given** my organisation is on the Enterprise plan
**When** I open Organisation Settings -> API Keys
**Then** I can create a new API key with a label, scopes (read/write per resource), and an expiry date
**And** the key value is shown ONCE on creation (with copy-to-clipboard) and stored as a hash
**And** I can revoke any active key, which takes effect within 60 seconds.

**Given** I have a valid API key
**When** I call any documented endpoint (`/api/v1/opportunities`, `/api/v1/proposals`, `/api/v1/companies/me`, etc.)
**Then** authentication succeeds via `Authorization: Bearer <key>`
**And** every response includes rate-limit headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`)
**And** rate limits are enforced per key (default 1000 req/min, configurable).

**Given** I exceed the rate limit
**When** I make another request
**Then** I get `429 Too Many Requests` with a `Retry-After` header.

### Story 8.4: OpenAPI Documentation Portal

As a developer,
I want clear OpenAPI/Swagger documentation,
So that I can integrate against the API confidently.

**Acceptance Criteria:**

**Given** the Enterprise API is deployed
**When** I visit `/api/v1/docs`
**Then** I see an OpenAPI 3.1 documentation portal listing all endpoints, parameters, request/response schemas, and example payloads
**And** I can authenticate with my API key and "try it" against my own data
**And** the OpenAPI spec is downloadable as JSON/YAML.

**Given** any endpoint changes
**When** the service starts
**Then** the OpenAPI spec is regenerated automatically from FastAPI route definitions
**And** breaking changes are detected in CI by comparing against a checked-in spec snapshot.
