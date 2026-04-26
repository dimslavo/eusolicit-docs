# Epic 5: Compliance Validation

**Goal:** Users can validate their drafted proposals against administrative regulatory frameworks (e.g., ZOP) to ensure no mandatory requirements are missed.
**FRs covered:** FR10

## Stories

### Story 5.1: Automated Regulatory Framework Validation
As a bid manager,
I want the system to check my proposal against assigned frameworks like ZOP,
So that I am confident my bid is fully compliant.

**Acceptance Criteria:**
**Given** I have completed the proposal draft
**When** I click "Run Compliance Check"
**Then** the system must validate the content against the assigned regulatory framework rules
**And** identify any failures or missing mandatory items

### Story 5.2: Compliance Metric Card Component
As a user,
I want a clear visual indicator of my proposal's compliance status,
So that I instantly know if it's ready to submit.

**Acceptance Criteria:**
**Given** a compliance check has run
**When** I view the dashboard or proposal editor
**Then** I must see a Compliance Metric Card showing a circular progress ring
**And** an expandable accordion detailing any missing mandatory items