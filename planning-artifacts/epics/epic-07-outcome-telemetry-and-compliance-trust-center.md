# Epic 7: Outcome Telemetry & Compliance Trust Center

Provide workspaces with Outcome Dashboards and automated briefs, and host a public Trust Center for compliance artefacts.

### Story 7.1: Workspace Outcome Dashboard

As a Bid Director,
I want a dashboard showing my bids tracked, win rate, AI-summary usage, and hours saved,
So that I can measure the ROI of the platform.

**Acceptance Criteria:**

**Given** I am in a workspace with historical activity
**When** I navigate to the Outcome Dashboard
**Then** I see metrics calculated via daily-refreshed materialized views
**And** the "First Bid in 14 Days" onboarding milestone tracker is visible.

### Story 7.2: Generate Monthly Outcome Brief PDF

As a Bid Director,
I want an auto-generated Outcome Brief PDF covering the prior month,
So that I can share structured ROI reports with my leadership team.

**Acceptance Criteria:**

**Given** a workspace with generated outcome telemetry
**When** the first of the month arrives
**Then** the system auto-generates an Outcome Brief PDF with key metrics, charts, and ROI calculations
**And** makes it available for download.

### Story 7.3: Public Trust Center Page

As a Procurement-Committee Member,
I want to visit a public `/trust` page to review compliance posture,
So that I can complete my vendor due diligence.

**Acceptance Criteria:**

**Given** I navigate to `/trust` without logging in
**When** the page loads
**Then** I can see the current compliance posture (GDPR, ISO 27001 status, Data Residency)
**And** download PDFs for the GDPR DPA, Security Overview, and BCP Summary.

### Story 7.4: Versioned Sub-Processor List

As a Compliance Officer,
I want the Trust Center's Sub-Processor List to auto-update from a structured file,
So that changes are tracked in version control and customers are notified correctly.

**Acceptance Criteria:**

**Given** the `infra/sub-processors.yaml` file is updated and deployed
**When** a user visits the `/trust` page
**Then** the Sub-Processor List reflects the updated source
**And** includes versioned change-log entries.