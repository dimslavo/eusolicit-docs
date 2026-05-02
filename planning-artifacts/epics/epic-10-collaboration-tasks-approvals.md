# Epic 10: Collaboration, Tasks & Approvals

Users organize work efficiently by declaring task dependencies, instantiating templates, and enforcing approval gates before bid submission.

### Story 10.1: Tasks CRUD with Dependencies
As a Bid Manager,
I want a Gantt-like task structure,
So that I can manage the entire proposal workflow.

**Acceptance Criteria:**
**Given** the tasks interface
**When** I create or update tasks with dependencies
**Then** cycle detection (DFS) runs and returns 422 if a cycle is detected
**And** status transitions enforce dependency gates (e.g., cannot complete if upstream is pending)

### Story 10.2: Task Templates
As a Bid Manager,
I want repeatable bid playbooks,
So that I can quickly set up tasks for new proposals.

**Acceptance Criteria:**
**Given** a new proposal workspace
**When** I apply a task template
**Then** N tasks and M dependencies are instantiated in one DB transaction
**And** an audit row is recorded on template apply

### Story 10.3: Approval Workflow
As a Bid Manager,
I want sign-off before submission,
So that I ensure quality control.

**Acceptance Criteria:**
**Given** a proposal ready for submission
**When** an approval is requested
**Then** the proposal cannot move to submitted until the Reviewer signs off
**And** the Reviewer is notified and all transitions are written to shared.audit_log
