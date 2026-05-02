# Epic 5: Advanced Proposal Collaboration & Workflow

This post-MVP epic enhances the proposal drafting process with features for teams, including section locking, commenting, task management with dependencies, and multi-stage approval workflows.

**FRs covered:** FR-31, FR-32, FR-37, FR-38, FR-39, FR-41

## Stories

### Story 5.1: Section Locking in Proposal Editor
As a user collaborating on a proposal,
I want to be able to lock a section that I am actively editing,
So that I can prevent concurrent edits and merge conflicts with my teammates.

**Acceptance Criteria:**
**Given** multiple users are in the same proposal editor
**When** I start editing a section
**Then** the section is automatically locked, and other users see a visual indicator that I am editing it.

### Story 5.2: Comments on Proposal Sections
As a team member reviewing a proposal,
I want to be able to add comments to specific sections or text selections,
So that I can provide feedback and ask questions without directly editing the content.

**Acceptance Criteria:**
**Given** I am viewing a proposal
**When** I select text and click "Comment"
**Then** a comment thread is created, and other users can view and reply to my comment.

### Story 5.3: Task Management for Proposals
As a bid manager,
I want to create and assign tasks related to a proposal,
So that I can organize the work and track my team's progress.

**Acceptance Criteria:**
**Given** I am managing a proposal
**When** I navigate to the "Tasks" tab
**Then** I can create tasks, assign them to team members, set due dates, and view their status on a Kanban board.

### Story 5.4: Task Dependencies
As a bid manager,
I want to define dependencies between tasks,
So that the workflow is clear and tasks are completed in the correct order.

**Acceptance Criteria:**
**Given** I am creating or editing a task
**When** I set a dependency (e.g., "Task B is blocked by Task A")
**Then** Task B cannot be marked as "Done" until Task A is complete.

### Story 5.5: Multi-stage Approval Workflow
As a bid manager,
I want to enforce a multi-stage approval workflow before a proposal can be submitted,
So that I can ensure quality and compliance with internal policies.

**Acceptance Criteria:**
**Given** I have configured an approval workflow (e.g., Technical Review -> Legal Review -> Final Approval)
**When** a draft is ready for review
**Then** the designated reviewers are notified, and they can approve or request changes for their stage.

### Story 5.6: Calendar Sync for Deadlines
As a user,
I want to sync opportunity deadlines and task due dates with my external calendar (Google/Outlook),
So that I can manage my schedule effectively.

**Acceptance Criteria:**
**Given** I have connected my external calendar account
**When** I track an opportunity or am assigned a task
**Then** the relevant deadlines appear in my Google or Outlook calendar.
