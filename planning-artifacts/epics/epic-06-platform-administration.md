# Epic 6: Platform Administration & Governance

This epic provides the internal tools needed to manage the platform. It includes the secure admin portal for managing tenants, subscriptions, compliance frameworks, and viewing the system-wide audit trail.

**FRs covered:** FR-35, FR-42, FR-43

## Stories

### Story 6.1: Secure Admin Portal
As a platform administrator,
I need a secure, IP-restricted admin portal,
So that I can manage the platform's core settings and data safely.

**Acceptance Criteria:**
**Given** I am a platform administrator
**When** I navigate to the admin URL from an authorized IP address
**Then** I am prompted to log in with my admin credentials (including MFA) to access the portal.

### Story 6.2: Tenant and Subscription Management
As a platform administrator,
I want to be able to view and manage all customer tenants and their subscriptions,
So that I can provide support and oversee the platform's business operations.

**Acceptance Criteria:**
**Given** I am in the admin portal
**When** I navigate to the "Tenants" section
**Then** I can search for tenants, view their user lists, and see their current subscription status.

### Story 6.3: Compliance Framework Management
As a platform administrator,
I want to create and manage the regulatory compliance frameworks (e.g., ZOP) that the AI uses,
So that the platform's compliance checks remain up-to-date with the latest laws.

**Acceptance Criteria:**
**Given** I am in the admin portal
**When** I navigate to the "Compliance Frameworks" section
**Then** I can add, edit, or remove rules that the AI compliance validator will use.

### Story 6.4: Immutable Audit Trail
As a platform administrator,
I want access to an immutable audit trail of all significant actions taken by users and the system,
So that I can investigate issues and ensure accountability.

**Acceptance Criteria:**
**Given** I am in the admin portal
**When** I access the "Audit Log"
**Then** I can search and view a time-stamped log of events like user logins, proposal creations, and subscription changes.
