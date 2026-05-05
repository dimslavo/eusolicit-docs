---
epic: 7
title: Team Collaboration and Workflow
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-6", "FR-7", "FR-31", "FR-32", "FR-37", "FR-38", "FR-39"]
nfrs_relevant: ["NFR-7", "NFR-15", "NFR-18", "NFR-19"]
ux_drs_relevant: ["UX-DR1", "UX-DR5"]
---

# Epic 7: Team Collaboration and Workflow

**Phase**: MVP (FR-6, FR-7) + Post-MVP (FR-31, FR-32, FR-37, FR-38, FR-39) | **Dependencies**: Epic 1 (Identity), Epic 5 (Proposals), Epic 9 (Notifications)

## Goal

Teams can collaborate on proposals: an admin invites teammates and assigns granular roles (Admin, Bid Manager, Contributor, Reviewer, Read-Only); contributors lock proposal sections to avoid concurrent overwrite; reviewers leave inline comments and @mentions; bid managers create tasks with dependencies and run multi-stage approval workflows before submission.

## User Outcome

After this epic, Elena (the bid manager from PRD Journey 1) can run her entire team's bid pipeline inside the platform: invite her 5 teammates, assign roles, parallel-author sections with locking, resolve feedback via inline comments, track task dependencies, and require sign-off before a proposal moves to "submitted".

## Epic Acceptance Criteria

- [ ] Workspace invitations with role selection are auditable and revocable.
- [ ] Role changes propagate platform-wide within one request cycle; unauthorised actions return 403/404 (NFR-7) — never leak existence of cross-tenant data.
- [ ] Section locks are enforced server-side in addition to UI affordances; expired locks are auto-released after a configurable timeout.
- [ ] Inline comments are anchored to a stable text range (ProseMirror mark) and survive non-conflicting edits.
- [ ] Task dependencies prevent dependent tasks from being marked "in progress" until predecessors are complete.
- [ ] Approval workflows are multi-stage, configurable per workspace, and block the proposal `status=submitted` transition until all stages pass.

## Stories

### Story 7.1: Workspace Member Invitations

As an admin user,
I want to invite other users to my workspace,
So that my team can collaborate on bids together.

**FR coverage**: FR-6

**Acceptance Criteria:**

**Given** I am the workspace admin
**When** I enter one or more emails and select a role per invitee in workspace settings
**Then** invitation records are created in `client.workspace_invitations` with single-use tokens
**And** invited users receive a branded email with an accept link.
**And** accepting the invitation creates a `client.workspace_members` row with the assigned role and lands the user in the workspace.
**And** I can revoke a pending invitation; revocation invalidates the token immediately.

---

### Story 7.2: Role-Based Access Control (RBAC) Matrix

As an admin user,
I want to assign granular roles (Admin, Bid Manager, Contributor, Reviewer, Read-Only) to my team members,
So that I can control who can view, edit, or approve proposals.

**FR coverage**: FR-7
**NFR coverage**: NFR-7

**Acceptance Criteria:**

**Given** I am managing workspace members
**When** I change a user's role
**Then** their effective permissions update at the next request cycle (no app restart needed).
**And** unauthorised actions are blocked with HTTP 403 (or 404 for cross-tenant resource probes per NFR-7).
**And** the RBAC matrix supports per-entity overrides via `client.entity_permissions` (e.g., a Contributor on workspace can be a Reviewer on one specific proposal).
**And** every role mutation is audit-logged.

---

### Story 7.3: Proposal Section Locking

As a contributor,
I want to lock a specific section of a proposal while I am editing it,
So that I prevent concurrent overwrite conflicts with my teammates.

**FR coverage**: FR-31 (Post-MVP)

**Acceptance Criteria:**

**Given** a multi-user proposal (Story 5.1)
**When** I focus into a section
**Then** a soft lock is requested via API; if available, the section is locked to my user for N minutes (configurable, default 5)
**And** other users see a read-only view with my name + avatar as lock owner.
**And** locks auto-renew while I am actively editing (heartbeat) and auto-expire when I leave or close the tab.
**And** an admin can force-release any stuck lock from the proposal toolbar; force-release is audit-logged.

---

### Story 7.4: Inline Comments and Mentions

As a team member,
I want to add comments and @mention colleagues on specific parts of the proposal,
So that we can discuss and resolve feedback directly in context.

**FR coverage**: FR-32 (Post-MVP)

**Acceptance Criteria:**

**Given** I am reviewing a proposal
**When** I select text and add a comment with an @mention
**Then** the comment is anchored to the text range via a ProseMirror mark and persists across non-conflicting edits.
**And** the mentioned user receives a notification (email + in-app) via Epic 9.
**And** comment threads support replies and a "Resolved" state which collapses but never deletes the thread.
**And** Reviewer-role users can comment but not edit body content.

---

### Story 7.5: Task Management and Dependencies

As a Bid Manager,
I want to create tasks, assign them, and define dependencies,
So that I can track the progress of the entire proposal effort.

**FR coverage**: FR-37, FR-38 (Post-MVP)

**Acceptance Criteria:**

**Given** I am viewing a proposal's "Workflow" tab
**When** I create a task with title, assignee, due date, and optional predecessor task(s)
**Then** the task appears on the team's task list and on the assignee's "My tasks" view.
**And** dependent tasks cannot transition to `in_progress` until all predecessors are `done`.
**And** task changes (status, assignee, due date) are notified via Epic 9.
**And** circular dependencies are rejected at creation time.

---

### Story 7.6: Multi-Stage Approval Workflow

As a Bid Manager,
I want to enforce an approval workflow before a proposal is marked submitted,
So that quality is guaranteed by designated reviewers.

**FR coverage**: FR-39 (Post-MVP)

**Acceptance Criteria:**

**Given** a workspace has configured a multi-stage approval workflow (e.g., Reviewer → Admin)
**When** the proposal author clicks "Send for Approval" on a draft
**Then** the proposal transitions to `status=in_review` with the first stage's approvers notified.
**And** each stage's approver can "Approve" (advances to next stage) or "Request Changes" (returns to author with comments).
**And** the proposal cannot transition to `status=submitted` until all stages approve.
**And** every approval/rejection is audit-logged with the actor, stage, and timestamp.

## Implementation Notes

- The RBAC dependency factory `check_entity_access()` in `client_api/core/rbac.py` is the single canonical enforcement point — never re-implement in route handlers.
- All cross-tenant endpoints get explicit negative tests: company A → company B resource → must return 403 or 404 (never 500 or leaking data).
- Section locks are kept in Redis with TTL + Postgres mirror for durability; the heartbeat path is hot and must be ≤ 50ms server time.
- Comments use ProseMirror marks with stable IDs so editor diff-merge does not orphan threads.
- Approval workflows are configured per-workspace, not per-proposal, for MVP simplicity; per-opportunity overrides are out of scope for this epic.
