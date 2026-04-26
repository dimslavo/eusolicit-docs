# Epic 6: Workflows & Approvals

**Goal:** Users can manage the bid lifecycle, sync critical deadlines to their calendars, and make AI-assisted bid/no-bid decisions.
**FRs covered:** FR12, FR15

## Stories

### Story 6.1: Bid/No-Bid AI Recommendation
As a bid manager,
I want the AI to recommend whether we should pursue a bid,
So that I can make data-driven decisions quickly.

**Acceptance Criteria:**
**Given** an analyzed tender
**When** I view the decision matrix
**Then** the AI must provide a recommendation payload (win probability, risks)
**And** I must have the ability to override this decision manually

### Story 6.2: Two-Way Calendar Sync
As a professional or enterprise user,
I want my proposal deadlines synced to my Google or Outlook calendar,
So that I never miss a critical submission date.

**Acceptance Criteria:**
**Given** I am a Pro/Enterprise user
**When** I configure my calendar integration
**Then** the system must sync tender deadlines and milestones to my calendar automatically
**And** provide an iCal feed option

### Story 6.3: Upcoming Deadlines Calendar View
As a bid manager,
I want a calendar view of all upcoming deadlines in the platform,
So that I can coordinate my team's workload effectively.

**Acceptance Criteria:**
**Given** I have multiple active bids
**When** I view the deadlines calendar snippet on the dashboard
**Then** I must see all upcoming milestones and submission dates clearly marked