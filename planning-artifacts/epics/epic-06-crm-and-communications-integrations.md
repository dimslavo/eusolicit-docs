# Epic 6: CRM & Communications Integrations

Sync opportunities and outcomes with CRM systems and send intelligent alerts to Slack/Teams.

### Story 6.1: OAuth2 Configuration for CRM

As a Bid Manager,
I want to authenticate and connect my workspace to HubSpot, Salesforce, or Pipedrive via OAuth2,
So that I can sync opportunities without manually exporting data.

**Acceptance Criteria:**

**Given** I am a Bid Manager on the Pro+ tier
**When** I configure a CRM integration in my workspace settings
**Then** the system completes the OAuth2 flow
**And** stores the CRM tokens securely using Fernet encryption at rest.

### Story 6.2: Bidirectional Sync of Opportunities with CRM

As a Bid Manager,
I want my EU Solicit opportunities and statuses to sync bidirectionally with my CRM Deals,
So that my sales pipeline is always up to date.

**Acceptance Criteria:**

**Given** a connected CRM integration
**When** an opportunity status changes in EU Solicit or in the external CRM
**Then** the changes sync across systems within 5 minutes via Celery background tasks
**And** conflict resolution applies a last-write-wins policy backed by an audit log entry.

### Story 6.3: Slack and Teams Webhook Notifications

As a Bid Team Lead,
I want to configure Slack and Teams webhooks for critical opportunity events,
So that my team responds to deadlines and alerts quickly.

**Acceptance Criteria:**

**Given** I am on the Professional tier or above
**When** I configure a webhook in my workspace settings
**Then** the system sends localized notifications for new matching opportunities, upcoming deadlines, and generation completions
**And** throttles notifications to a maximum of 1 per opportunity per event-type to prevent flooding.