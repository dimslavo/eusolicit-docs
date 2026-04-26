## Epic 18: Trust Center & Compliance Posture

Launch a public Trust Center page to provide prospective customers with immediate access to compliance artefacts, accelerating the procurement process.
**FRs covered:** FR10.1, FR10.2, FR10.3, FR10.4

### Story 18.1: Public Trust Center Page

As a Procurement Committee Member,
I want to view EU Solicit's compliance posture on a public page,
So that I can verify GDPR and ISO 27001 status before purchasing.

**Acceptance Criteria:**

**Given** an unauthenticated user navigates to `/trust`
**Then** they see the current compliance posture (GDPR, ISO 27001 roadmap, Data Residency)
**And** they can download the GDPR DPA and Security Overview PDFs

### Story 18.2: Sub-Processor YAML Pipeline

As a System Admin,
I want to maintain the sub-processor list in a YAML file,
So that it automatically updates the Trust Center and tracks changes in version control.

**Acceptance Criteria:**

**Given** an update to `infra/sub-processors.yaml` is deployed
**When** a user visits the Trust Center
**Then** the generated Sub-Processor List PDF reflects the new changes
**And** a changelog entry is visible

### Story 18.3: Compliance Notifications

As a Tenant Admin with an active DPA,
I want to be notified when sub-processors change,
So that my firm remains compliant with GDPR Article 28.

**Acceptance Criteria:**

**Given** a change to the sub-processor list
**When** the new list is published
**Then** an email notification is automatically queued for all active tenant admins
