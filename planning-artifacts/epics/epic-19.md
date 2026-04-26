## Epic 19: Outcome Telemetry & Renewal Proof

Implement telemetry and reporting tools that quantify platform ROI (win rate, hours saved) to facilitate renewals and prove value to consulting firms.
**FRs covered:** FR9.1, FR9.2, FR9.3, FR9.4, FR9.5

### Story 19.1: Opportunity Attribution and Outcome Capture

As a Bid Manager,
I want to log the outcome of an opportunity (Won/Lost) and whether it was platform-attributed,
So that I can build accurate historical data.

**Acceptance Criteria:**

**Given** an opportunity in the "submitted" stage
**When** the user marks it as "Won"
**Then** the outcome, evaluator score, and contract value are saved
**And** the system retains the `platform_attributed` boolean flag

### Story 19.2: Outcome Dashboard

As a Bid Director,
I want to view an aggregated dashboard of my workspace's performance,
So that I can track win rates, content reuse, and hours saved.

**Acceptance Criteria:**

**Given** a workspace with historical bid data
**When** the Bid Director accesses the Outcome Dashboard
**Then** they see widgets for rolling 12-month win rate, total bids submitted, and estimated hours saved
**And** the metrics load in under 500ms using materialized views

### Story 19.3: Monthly Outcome Brief PDF

As a Bid Director,
I want to download a monthly PDF report of our platform usage and ROI,
So that I can present it in internal review meetings.

**Acceptance Criteria:**

**Given** the 1st day of a new month
**When** the user requests the prior month's Outcome Brief
**Then** a PDF is generated summarizing key metrics, trends, and ROI calculations based on workspace-configurable hourly rates
