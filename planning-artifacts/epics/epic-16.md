## Epic 16: CRM Integrations (HubSpot, Salesforce, Pipedrive)

Provide bi-directional synchronization between EU Solicit opportunities and external CRM deals so that pipeline data remains consistent.
**FRs covered:** FR11.1, FR11.2, FR11.3, FR11.5

### Story 16.1: OAuth2 Connection & Token Storage

As a Bid Manager,
I want to securely connect my workspace to my CRM,
So that the platform can sync opportunities automatically.

**Acceptance Criteria:**

**Given** a workspace on a Pro+ tier
**When** the user initiates OAuth2 flow with HubSpot
**Then** the connection is established
**And** tokens are stored securely using Fernet encryption

### Story 16.2: Opportunity Sync (EU Solicit to CRM)

As a Bid Manager,
I want new opportunities added to my workspace to create deals in my CRM,
So that my sales team has visibility into the bid pipeline.

**Acceptance Criteria:**

**Given** an active CRM connection
**When** an opportunity status changes to "qualified" or "bid"
**Then** a corresponding Deal is created or updated in the CRM within 60 seconds
**And** the CRM deal link is saved in the opportunity record

### Story 16.3: CRM Deal Update Sync (CRM to EU Solicit)

As a Sales Director,
I want changes made in the CRM to reflect in EU Solicit,
So that the bid team works with the latest deal values and stages.

**Acceptance Criteria:**

**Given** a linked opportunity and CRM Deal
**When** the Deal stage is updated in the CRM
**Then** the opportunity status in EU Solicit is updated within 5 minutes via a webhook or polling mechanism
