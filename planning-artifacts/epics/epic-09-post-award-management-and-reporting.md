---
epicNumber: 9
title: Post-Award Management & Reporting
status: draft
dependencies: [1, 2, 4]
frsCovered: []
nfrsRelevant: [NFR-7, NFR-15, NFR-18, NFR-19, NFR-20]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
phase: post-mvp
note: "FR-44 etc. are covered elsewhere; this epic is driven by the UX Supplement and the EU-grants vision in PRD Phase 2/3."
---

## Epic 9: Post-Award Management & Reporting

This epic extends platform value beyond bid submission. Once a tender or grant is awarded, users can manage its lifecycle inside EU Solicit: tracking deliverables, recording budget expenditure against approved cost categories, and generating periodic technical and financial reports required by the funding agency. This addresses the long-term grant-management vision in PRD Phase 2.

### Story 9.1: Mark Opportunity as Awarded and Open Active-Grant Workspace

As a grant manager,
I want to mark a won opportunity as Awarded,
So that I can manage its lifecycle post-award.

**Acceptance Criteria:**

**Given** I have submitted a proposal and received a contract
**When** I mark the opportunity as "Awarded" with the contract value, start date, and end date
**Then** an `client.active_grants` record is created linked to the opportunity and proposal
**And** the grant appears on a new "Active Grants" dashboard
**And** the original proposal becomes read-only (locked as the awarded version) with a clear "Submitted" state.

### Story 9.2: Track Budget Expenditure by Cost Category

As a grant manager,
I want to track expenditure against the grant's approved cost categories,
So that I can ensure financial compliance and monitor burn rate.

**Acceptance Criteria:**

**Given** I am viewing an active grant
**When** I open the Budget tab
**Then** I see budget categories from the proposal with approved amount, committed amount, actual amount, and remaining
**And** I can log expenses against any category with date, vendor, amount, currency, description, and supporting attachments
**And** an over-budget category is highlighted with a warning indicator
**And** a real-time burn-rate chart shows monthly spend vs. plan.

**Given** an expense is logged in a foreign currency
**When** the entry is saved
**Then** the system records both the original amount and the EUR equivalent at the prevailing daily ECB rate.

### Story 9.3: Track Deliverables and Work Package Progress

As a grant manager,
I want to track deliverables and work-package (WP) progress,
So that I can show the funding agency we're on plan.

**Acceptance Criteria:**

**Given** the awarded proposal includes work packages and deliverables
**When** the grant workspace is created
**Then** WPs and deliverables are imported automatically with planned start/end dates
**And** I can update each WP's actual progress (%) and each deliverable's status (planned / in progress / submitted / accepted / rejected).

**Given** a deliverable due date is approaching (within 14 days)
**When** the daily reminder job runs
**Then** the assigned owner receives an in-app and email reminder
**And** overdue deliverables are visible on the dashboard.

### Story 9.4: Generate Periodic Technical and Financial Reports

As a grant manager,
I want the system to help me generate periodic reports required by the funding agency,
So that I can simplify reporting and reduce overhead.

**Acceptance Criteria:**

**Given** a reporting period is ending for an active grant
**When** I click "Generate Report"
**Then** the system creates a report draft pre-filled with: WP progress narrative summary, deliverables list, expenditure summary by category, variance analysis, and supporting attachments index
**And** the draft is editable in a TipTap rich-text editor for me to add narrative
**And** the final report is exportable to PDF and DOCX following the funding-agency template (Horizon Europe by default).

**Given** I submit a generated report
**When** the submission is recorded
**Then** the version is locked and added to the grant's report history with timestamp and submitter.
