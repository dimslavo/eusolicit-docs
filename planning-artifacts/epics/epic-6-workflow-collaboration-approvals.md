# Epic 6: Workflow, Collaboration & Approvals

Admin users can make bid/no-bid decisions, sync calendars, and the system securely logs all actions for auditability.
**FRs covered:** FR12, FR14, FR15

## Story 6.1: Bid/No-Bid Decision

As an Admin,
I want to record the bid or no-bid decision with a manual override,
So that the team knows whether to proceed.

**Acceptance Criteria:**

**Given** the AI recommendation
**When** I select a decision
**Then** it is recorded independently of the AI recommendation
**And** written to audit logs asynchronously

## Story 6.2: Calendar Synchronization

As a Pro user,
I want my proposal deadlines synced to Google/Outlook,
So that I don't miss submission dates.

**Acceptance Criteria:**

**Given** I connect my calendar
**When** a deadline is set in the workspace
**Then** it is automatically synced to my external calendar
**And** it triggers Transient Success Toasts upon successful sync