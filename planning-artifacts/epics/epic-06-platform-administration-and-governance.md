---
epicNumber: 6
title: Platform Administration & Governance
status: draft
dependencies: [1]
frsCovered: [FR-35, FR-42, FR-43]
nfrsRelevant: [NFR-5, NFR-6, NFR-7, NFR-8, NFR-23]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
---

## Epic 6: Platform Administration & Governance

This epic provides the internal tools needed to operate the platform safely. It includes the secure, IP-restricted, MFA-gated admin portal; tenant and subscription oversight; compliance-framework management (e.g., ZOP rules used by the AI validator); and a queryable, immutable audit trail of all significant user and system actions. After this epic, the platform admin persona (Journey 3) can update compliance frameworks and monitor crawler health from a single console.

### Story 6.1: Secure Admin Portal Access

As a platform administrator,
I need a secure, IP-restricted admin portal,
So that I can manage the platform safely.

**Acceptance Criteria:**

**Given** the admin portal is deployed on a separate hostname
**When** a request arrives from an IP outside the configured allow-list
**Then** the request is rejected at the edge before reaching the application
**And** the rejection is logged with source IP and timestamp.

**Given** I am an admin from an allow-listed IP
**When** I authenticate with email/password
**Then** I am required to complete TOTP MFA before a session is established (NFR-8)
**And** the session JWT carries an `is_platform_admin` claim with a 30-minute idle timeout.

**Given** I am authenticated as a platform admin
**When** I perform any mutating action
**Then** the action is recorded in the audit trail with my admin user ID, source IP, action type, before/after values, and correlation ID.

### Story 6.2: Tenant and Subscription Management

As a platform administrator,
I want to view and manage all customer tenants and their subscriptions,
So that I can support customers and oversee operations.

**Acceptance Criteria:**

**Given** I am in the admin portal
**When** I navigate to "Tenants"
**Then** I can search by company name, slug, contact email, or `customer_id`
**And** for each tenant I can view: user list, current subscription tier, billing status, usage counters, recent activity, and key metrics.

**Given** I select a tenant
**When** I open the subscription panel
**Then** I can apply admin actions: extend trial, grant credits, suspend account, force-cancel subscription
**And** every action requires a justification comment that is persisted in the audit log.

### Story 6.3: Compliance Framework Management

As a platform administrator,
I want to create and manage regulatory compliance frameworks (e.g., ZOP, Horizon Europe),
So that the AI compliance validator stays current with the latest law.

**Acceptance Criteria:**

**Given** I am in the admin portal
**When** I navigate to "Compliance Frameworks"
**Then** I can create a framework with a name, jurisdiction, and version
**And** within a framework I can author rules with title, description, severity (low/med/high/critical), category, and the natural-language criteria the AI evaluates.

**Given** I update a published framework
**When** I save the changes
**Then** the new version is published immediately to the AI validator
**And** all running and queued validations use the new version
**And** previous versions are retained for historical audit.

**Given** a compliance framework rule is referenced by tenant proposals
**When** I attempt to delete the rule
**Then** the rule is soft-deleted (deactivated) rather than hard-deleted, preserving historical references.

### Story 6.4: Immutable Audit Trail

As a platform administrator,
I want a queryable, immutable audit trail of significant user and system actions,
So that I can investigate issues and meet compliance obligations.

**Acceptance Criteria:**

**Given** any tenant or admin user performs a significant action (login, role change, proposal create/submit, subscription change, file upload, document export, admin mutation)
**When** the action completes
**Then** an `admin.audit_events` row is written with `event_id`, `timestamp`, `actor_user_id`, `actor_role`, `tenant_id`, `action_type`, `resource_type`, `resource_id`, `before_value`, `after_value`, `source_ip`, `user_agent`, `correlation_id`
**And** the row is append-only at the database layer (no UPDATE or DELETE permitted on the role used by services).

**Given** I query the audit log in the admin portal
**When** I filter by actor, tenant, action type, or time range
**Then** results return within 2 seconds for windows up to 30 days
**And** results can be exported as CSV with full lineage.

### Story 6.5: Crawler Health & Operational Dashboards

As a platform administrator,
I want operational dashboards for the data pipeline crawlers and core services,
So that I can detect and resolve issues proactively (Journey 3 climax).

**Acceptance Criteria:**

**Given** the data-pipeline service runs scheduled crawlers
**When** I open the "Crawler Status" dashboard
**Then** I see per-source metrics: last run timestamp, records ingested, error rate, average duration, and 24-hour trend
**And** I can drill into a per-source error log with stack traces and source-portal HTML snippets
**And** anomalies (error rate >5% over 1h) trigger an automatic alert to the on-call channel.

**Given** I am on the dashboard
**When** I click "File Engineering Ticket"
**Then** a pre-filled GitHub/Linear issue is created with the failing source, recent error log, and crawler version.
