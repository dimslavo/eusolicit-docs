---
epicNumber: 5
title: Advanced Proposal Collaboration & Workflow
status: draft
dependencies: [1, 2, 3, 4]
frsCovered: [FR-31, FR-32, FR-37, FR-38, FR-39, FR-41]
nfrsRelevant: [NFR-1, NFR-7, NFR-11, NFR-18, NFR-19, NFR-20, NFR-23]
uxDrsCovered: [UX-P09, UX-P10, UX-P11, UX-P12, UX-P13, UX-P14, UX-P15, UX-P16]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
phase: post-mvp
---

## Epic 5: Advanced Proposal Collaboration & Workflow

This post-MVP epic extends the proposal drafting experience for teams. It adds section locking to prevent concurrent-edit conflicts, threaded comments and @mentions, task management with dependencies and a Kanban board, multi-stage approval pipelines with reviewer-required comments, and external calendar synchronisation for opportunity deadlines and task due dates.

### Story 5.1: Section Locking in the Proposal Editor

As a user collaborating on a proposal,
I want to lock a section while I edit it,
So that I prevent concurrent-edit conflicts.

**Acceptance Criteria:**

**Given** multiple users are viewing the same proposal
**When** I begin editing a section
**Then** the section is locked to my user for 5 minutes (auto-renewed while I am active)
**And** other users see a lock indicator with my name and avatar
**And** other users see the section in read-only mode until the lock is released or expires.

**Given** the lock-holder has been inactive for 5 minutes
**When** the lock TTL expires
**Then** the lock is auto-released
**And** other users can claim the section.

### Story 5.2: Comments on Proposal Sections

As a team member reviewing a proposal,
I want to add comments on sections or text selections,
So that I can give feedback without editing the content.

**Acceptance Criteria:**

**Given** I am viewing a proposal
**When** I select text and click "Comment"
**Then** an inline margin-anchored comment thread is created (UX-P14)
**And** I can @mention teammates, who are notified via in-app and email
**And** other users can reply, like, and resolve threads.

**Given** a comment thread is resolved
**When** I view the proposal
**Then** the thread is collapsed by default with an option to show resolved threads.

### Story 5.3: Task Management for Proposals (Kanban Board)

As a bid manager,
I want to create and assign tasks for a proposal,
So that I can organise the work and track progress.

**Acceptance Criteria:**

**Given** I am managing a proposal
**When** I open the Tasks tab
**Then** I see a Kanban board with columns (To Do / In Progress / Blocked / Done)
**And** I can create tasks, assign assignees, set due dates and priorities
**And** the board supports real-time optimistic updates with error rollback (UX-P09)
**And** a persistent progress summary bar shows totals per column (UX-P12).

**Given** a task is auto-generated from a tender deadline
**When** the task is created
**Then** its due date is calculated relative to the opportunity submission deadline, accounting for compressed timelines (UX-P10).

### Story 5.4: Task Dependencies and Blocking

As a bid manager,
I want to define dependencies between tasks,
So that the workflow runs in the correct order.

**Acceptance Criteria:**

**Given** I am editing a task
**When** I add a dependency ("Blocked by Task A")
**Then** the dependency is persisted and visualised on the board with a link line
**And** the dependent task is marked Blocked while its blocker is incomplete
**And** blocked tasks are visually distinct and guarded from completion until unblocked (UX-P11).

**Given** I attempt to create a circular dependency
**When** I save the task
**Then** the operation is rejected with a clear error.

### Story 5.5: Multi-Stage Approval Workflow

As a bid manager,
I want to enforce a multi-stage approval workflow before submission,
So that I can ensure quality and policy compliance.

**Acceptance Criteria:**

**Given** I configure an approval pipeline (e.g., Technical Review -> Legal Review -> Final Approval)
**When** the configuration would result in an invalid state (no reviewer for a stage)
**Then** the system rejects the configuration (UX-P15).

**Given** a proposal is submitted to the pipeline
**When** it reaches a review stage
**Then** the designated reviewers are notified via in-app, email, and (if configured) Slack/Teams with a one-click deep link to the review page (UX-P16)
**And** the proposal's editor shows a sticky Pipeline Status Bar that pulses on pending reviews (UX-P13).

**Given** a reviewer rejects a stage
**When** they submit the rejection
**Then** a comment is required, displayed as inline margin annotations in the editor (UX-P14)
**And** the proposal is returned to the previous stage with all reviewer feedback preserved.

### Story 5.6: Calendar Sync for Opportunity Deadlines and Task Due Dates

As a user,
I want to sync opportunity deadlines and task due dates with my Google or Outlook calendar,
So that I can manage my schedule from a single place.

**Acceptance Criteria:**

**Given** I am on the Calendar Integrations settings
**When** I click "Connect Google" or "Connect Outlook"
**Then** I complete OAuth and grant write access to a dedicated EU Solicit calendar
**And** my external OAuth tokens are encrypted at the application layer in addition to at-rest encryption (NFR-6).

**Given** I have connected a calendar
**When** I track an opportunity or am assigned a task
**Then** the relevant deadlines appear in the connected calendar within 60 seconds
**And** updates to deadlines/tasks are synchronised within 60 seconds
**And** disconnecting the integration removes future-dated EU Solicit events from the user's calendar.
