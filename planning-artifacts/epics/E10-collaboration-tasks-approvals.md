# E10: Collaboration, Tasks & Approvals

**Sprint**: 11-12 | **Points**: 55 | **Dependencies**: E02, E07 | **Milestone**: MVP

## Goal

Enable multi-user bid preparation by providing per-proposal role-based collaboration, pessimistic section locking, threaded comments, a full task management system with DAG-based dependencies and templates, company-configurable approval workflows, AI-assisted bid/no-bid decision support, and outcome tracking with preparation cost logging. These capabilities turn EU Solicit from a single-user tool into a team-oriented bid management platform where managers can orchestrate work, reviewers can approve deliverables through structured stages, and the organisation can measure win rates and ROI.

## Acceptance Criteria

- [ ] Collaborators can be assigned to a proposal with one of five roles (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only) and removed; the collaborator list is queryable per proposal
- [ ] A user can acquire a pessimistic lock on a proposal section; a second user receives HTTP 423; locks auto-expire after 15 minutes; lock status is returned in the proposal content endpoint
- [ ] Comments can be created on a section+version, listed per section, resolved/unresolved, and unresolved comments carry forward when a new version is created
- [ ] Tasks can be created, assigned, updated (status, priority, due date), and listed with filters by company, opportunity, status, assignee, and priority
- [ ] Task dependency edges can be added/removed; cycles are rejected (DAG validation); a task cannot transition to completed while upstream finish_to_start dependencies are incomplete
- [ ] Task templates can be CRUDed per company; applying a template to an opportunity creates concrete tasks with deadlines computed from the opportunity deadline
- [ ] Approval workflows with ordered stages can be CRUDed; one workflow can be set as company default
- [ ] Approval decisions (approved/rejected/returned_for_revision) can be submitted per stage; auto_advance moves to the next stage; an `approval.decided` event is emitted
- [ ] Bid/no-bid decision triggers the Bid/No-Bid Decision Agent via AI Gateway, returning a structured scorecard; the user can override with justification; the decision is persisted
- [ ] Bid outcomes (won/lost/withdrawn) can be recorded with optional contract value, feedback, and scores; recording triggers the Lessons Learned Agent
- [ ] Bid preparation time and costs can be logged per user per proposal
- [ ] Frontend: collaborator management panel, section lock indicators, comments sidebar, task kanban board, task detail modal, template manager, approval pipeline page, bid/no-bid decision page, and outcome recording form are all implemented
- [ ] All endpoints enforce proposal-level RBAC via the collaborator role; read_only users cannot mutate
- [ ] All write operations produce audit-trail entries

## Stories

### S10.01: Proposal Collaborator CRUD API

**Points**: 3 | **Type**: backend

Add the `proposal_collaborators` table migration and FastAPI endpoints. `POST /proposals/{id}/collaborators` accepts `{user_id, role}`, enforces that the caller is bid_manager on that proposal or a company admin, and inserts the record. `GET /proposals/{id}/collaborators` returns the list with user display names. `DELETE /proposals/{id}/collaborators/{user_id}` removes access. `PATCH /proposals/{id}/collaborators/{user_id}` changes the role. Add a unique constraint on `(proposal_id, user_id)`. Include a `granted_by` and `granted_at` audit trail. Write unit tests for role enforcement (only bid_manager/admin can manage), duplicate prevention, and self-removal guard (bid_manager cannot remove themselves if they are the last bid_manager).

---

### S10.02: Proposal-Level RBAC Middleware

**Points**: 3 | **Type**: backend

Create a FastAPI dependency `require_proposal_role(*allowed_roles)` that resolves the current user's role on the target proposal from `proposal_collaborators`. If the user has no row or their role is not in the allowed set, raise HTTP 403. Integrate this dependency into all existing proposal endpoints (content read requires any role; content write requires all roles except read_only). Company admins bypass the check. Write tests covering each role boundary, missing collaborator row, and admin bypass.

---

### S10.03: Section Locking API

**Points**: 3 | **Type**: backend

Add the `proposal_section_locks` table with a unique constraint on `(proposal_id, section_key)`. Implement `POST /proposals/{id}/sections/{key}/lock` which attempts an INSERT; if the row already exists and `expires_at > now()` and `locked_by != current_user`, return 423 with the lock holder's name and expiry. If the existing lock is expired, replace it. `DELETE /proposals/{id}/sections/{key}/lock` releases only if the caller holds the lock or is bid_manager. Add a `GET /proposals/{id}/sections` response enrichment that includes lock status per section. Implement a scheduled background task (or DB trigger) that cleans up expired locks. Write tests for contention, expiry, force-release by bid_manager, and concurrent acquisition attempts.

---

### S10.04: Proposal Comments API

**Points**: 3 | **Type**: backend

Add the `proposal_comments` table migration. Implement `POST /proposals/{id}/comments` accepting `{version_id, section_key, body}`. `GET /proposals/{id}/comments?section_key=...&version_id=...` returns comments ordered by `created_at`. `PATCH /proposals/{id}/comments/{comment_id}/resolve` sets `resolved=true` and `resolved_by`. `PATCH /proposals/{id}/comments/{comment_id}/unresolve` reverts. On new version creation (hook into E07 version endpoint), carry forward all unresolved comments by cloning them with the new `version_id`. Enforce that only non-read_only collaborators can create/resolve. Write tests for create, list filtering, resolve toggle, and version carry-forward logic.

---

### S10.05: Task CRUD API

**Points**: 3 | **Type**: backend

Add the `tasks` table migration. Implement full CRUD: `POST /tasks` with fields `{company_id, opportunity_id?, proposal_id?, title, description, assigned_to, priority, due_date}`. `GET /tasks?company_id=...&opportunity_id=...&status=...&assigned_to=...&priority=...` with pagination. `GET /tasks/{id}` returns detail including dependency list. `PATCH /tasks/{id}` for status transitions, reassignment, priority, and due_date changes. `DELETE /tasks/{id}` soft-deletes. Status transitions validate: cannot move to `completed` if blocking dependencies exist (checked in S10.06). `completed_at` is auto-set. Write tests for CRUD, filter combinations, and pagination.

---

### S10.06: Task Dependencies & DAG Validation

**Points**: 5 | **Type**: backend

Add the `task_dependencies` table with unique constraint on `(task_id, depends_on_task_id)`. Implement `POST /tasks/{id}/dependencies` accepting `{depends_on_task_id, dependency_type}`. Before inserting, run a cycle detection algorithm (DFS/BFS from `depends_on_task_id` following existing edges to check if `task_id` is reachable); reject with 422 if a cycle would be created. `DELETE /tasks/{id}/dependencies/{dep_id}` removes the edge. Enhance the task status transition logic: when a task attempts to move to `completed`, verify all `finish_to_start` upstream dependencies are `completed`; for `finish_to_finish`, verify they are `completed` or `in_progress`. If any dependency of type `finish_to_start` becomes `completed`, check if downstream tasks in `blocked` status can be unblocked and transition them to `pending`. Write thorough tests: simple chain, diamond dependency, cycle rejection, multi-hop cycle, status blocking, and auto-unblock.

---

### S10.07: Task Templates CRUD & Application

**Points**: 3 | **Type**: backend

Add the `task_templates` table. The `stages` JSONB column stores an ordered array of task definitions, each with `{title, description, role, relative_days_before_deadline, dependency_indices[]}`. Implement `POST /task-templates`, `GET /task-templates?company_id=...&opportunity_type=...`, `GET /task-templates/{id}`, `PATCH /task-templates/{id}`, `DELETE /task-templates/{id}`. Implement `POST /task-templates/{id}/apply` accepting `{opportunity_id}`: reads the opportunity's deadline, iterates the template stages, creates concrete `tasks` rows with `due_date = deadline - relative_days`, and wires up `task_dependencies` based on `dependency_indices`. Return the created task IDs. Write tests for template CRUD, deadline computation, dependency wiring, and missing opportunity deadline error.

---

### S10.08: Approval Workflow & Stages CRUD API

**Points**: 3 | **Type**: backend

Add `approval_workflows` and `approval_stages` table migrations. Implement `POST /approval-workflows` accepting `{company_id, name, stages: [{name, order, required_role, auto_advance}]}` which creates the workflow and its stages in a transaction. `GET /approval-workflows?company_id=...` lists workflows. `PATCH /approval-workflows/{id}` updates name and stages (delete-and-recreate stages in transaction). `DELETE /approval-workflows/{id}` cascades to stages (only if no decisions reference it). `POST /approval-workflows/{id}/set-default` marks it as company default and unmarks any previous default. Write tests for CRUD, stage ordering, default toggling, and cascade guard.

---

### S10.09: Approval Decision Engine

**Points**: 3 | **Type**: backend

Add the `approval_decisions` table. Implement `POST /proposals/{id}/approvals/decide` accepting `{stage_id, decision, comment}`. Validate that the current user holds the `required_role` for that stage (from `approval_stages`). Validate that all prior stages (lower `order`) have an `approved` decision. Insert the decision. If `decision == approved` and the stage has `auto_advance == true` and a next stage exists, the response indicates the next pending stage. Emit an `approval.decided` event via the event bus (or outbox table) containing `{proposal_id, stage_id, decision, decided_by}`. If the final stage is approved, update the proposal status to `approved`. Write tests for role enforcement, stage ordering validation, auto-advance, final-stage status update, and event emission.

---

### S10.10: Bid/No-Bid Decision API & AI Integration

**Points**: 5 | **Type**: backend

Add the `bid_decisions` table. Implement `POST /opportunities/{id}/bid-decision/evaluate` which gathers opportunity data, company profile, and historical win/loss data, then calls the Bid/No-Bid Decision Agent via the AI Gateway. The agent returns a structured JSONB scorecard (dimensions: strategic_fit, technical_capability, financial_viability, competitive_position, resource_availability, each scored 1-10 with rationale). Store the scorecard in `ai_recommendation`. Implement `POST /opportunities/{id}/bid-decision` accepting `{decision, user_override?, override_justification?}` to record the final human decision. `GET /opportunities/{id}/bid-decision` returns the current decision with scorecard. Write tests using a mocked AI Gateway response, override logic, and idempotency (re-evaluation replaces previous scorecard).

---

### S10.11: Bid Outcome & Lessons Learned Integration

**Points**: 3 | **Type**: backend

Add the `bid_outcomes` table. Implement `POST /opportunities/{id}/outcome` accepting `{proposal_id, outcome, contract_value?, evaluator_feedback?, evaluator_scores?}`. On successful insert, emit a `bid.outcome.recorded` event. If the outcome is `won` or `lost`, asynchronously trigger the Lessons Learned Agent via the AI Gateway, passing the outcome, proposal summary, and evaluator feedback. The agent response (key takeaways, improvement areas) is stored or linked for future retrieval. `GET /opportunities/{id}/outcome` returns the recorded outcome. Add the `bid_preparation_logs` table and implement `POST /proposals/{id}/preparation-logs` for logging time/cost entries and `GET /proposals/{id}/preparation-logs` for retrieval with aggregation (total hours, total cost). Write tests for outcome recording, event emission, preparation log aggregation, and duplicate outcome guard.

---

### S10.12: Collaborator Management & Lock Indicator UI

**Points**: 3 | **Type**: frontend

Build the collaborator management panel as a slide-over on the proposal workspace. Show current collaborators in a table with avatar, name, role badge, and a remove button. An "Add collaborator" form has an email/user search input and a role dropdown (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only). Only bid_managers see the add/remove controls. Integrate section lock indicators into the Tiptap proposal editor: each section header shows a lock icon with the editor's name when `locked_by` is populated and `expires_at` is in the future. Clicking into a locked section shows a toast with the lock holder and expiry countdown. Acquiring a lock happens automatically when the user focuses a section; releasing happens on blur or navigation. Poll lock status every 30 seconds.

---

### S10.13: Comments Sidebar UI

**Points**: 3 | **Type**: frontend

Build a collapsible comments sidebar in the proposal editor. When a user selects a section, the sidebar filters to comments for that section and the current version. Each comment shows author avatar, timestamp, body, and a resolve checkbox (visible to non-read_only users). A "New comment" text area at the top posts via the comments API. Resolved comments are visually dimmed and grouped at the bottom. A badge count of unresolved comments appears on the section header. When a new version is created, display a banner noting that unresolved comments have been carried forward. Implement optimistic UI updates for comment creation and resolution toggling.

---

### S10.14: Task Kanban Board & Detail Modal

**Points**: 5 | **Type**: frontend

Build the task board page with four kanban columns: Pending, In Progress, Completed, Blocked. Each card shows title, assignee avatar, priority badge (P1 red, P2 orange, P3 yellow, P4 grey), and due date (red if overdue). Implement drag-and-drop between columns using a library (e.g., dnd-kit), which triggers a `PATCH /tasks/{id}` status update. If the backend rejects the transition (dependency block), revert the card position and show an error toast. Filter bar at the top allows filtering by assignee, priority, and linked opportunity. Clicking a card opens a detail modal with editable title, description, assignee picker, priority selector, due date picker, a dependency list (with add/remove), and a status dropdown. The modal saves on field blur or explicit save button.

---

### S10.15: Task Template Manager UI

**Points**: 2 | **Type**: frontend

Build a task template management page under company settings. List existing templates in a table with name, opportunity type, and stage count. A "Create template" button opens a form with name, opportunity type selector, and a dynamic stage list. Each stage row has: title, description, role dropdown, relative-days-before-deadline number input, and dependency checkboxes referencing earlier stages by index. Stages can be reordered via drag-and-drop and deleted. On the opportunity detail page, add an "Apply template" button that opens a modal listing available templates, previews the computed task dates based on the opportunity deadline, and confirms creation.

---

### S10.16: Approval Pipeline & Bid Decision UI

**Points**: 5 | **Type**: frontend

Build the approval pipeline page for a proposal. Display a horizontal stepper/progress bar with each stage name; completed stages show a green check and the decider's name, the current stage is highlighted with an action button, and future stages are greyed out. The action area for the current stage shows the required role, a decision dropdown (Approve / Reject / Return for Revision), a comment text area, and a submit button. Below the stepper, show a decision history table with stage, decision, decider, comment, and timestamp. Build the bid/no-bid decision page for an opportunity: display the AI scorecard as a radar chart (using a charting library) overlaid on a summary table of dimension scores and rationales. Below, show an override form with a decision selector (Bid / No Bid / Conditional), a justification text area (required if overriding AI recommendation), and a submit button. Build the bid outcome recording form with outcome selector (Won / Lost / Withdrawn), contract value currency input, evaluator feedback text area, and a dynamic score input table. All three views include loading states, error handling, and success confirmations.
