# Story 10.5: Task CRUD API

Status: done

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-5-task-crud-api
- **Points:** 3
- **Type:** backend
- **Module:** Client API (`client_api.api.v1.tasks`, `client_api.models.task`, `client_api.schemas.tasks`, `client_api.services.task_service`, new Alembic migration `039_tasks`)
- **Priority:** P0 (Epic 10 foundation — unblocks Story 10.6 Task Dependencies & DAG Validation and 10.14 Task Kanban Board)
- **Depends On:** Story 10.2 (Proposal-Level RBAC Middleware), Story 7.2 (Proposals API)

## Story

As a **bid_manager on a proposal (or a company admin)**,
I want **to create, update, and manage tasks with statuses, priorities, and assignees**,
so that **I can track the preparation of a proposal and ensure tasks are completed on time.**

## Description

This story implements full CRUD for the `tasks` table. Tasks can be associated with a `company_id`, and optionally linked to an `opportunity_id` and/or `proposal_id`. They include fields for `title`, `description`, `assigned_to`, `priority`, and `due_date`. The story provides listing with pagination and filtering, as well as status updates. Transitioning a task to `completed` will set `completed_at` to the current time. Note that blocking dependency logic is formally enforced in Story 10.6, but the schema must accommodate `status` changes.

## Acceptance Criteria

1. [x] **AC1 — `client.tasks` Alembic migration.** Create a new migration file. The `tasks` table includes:
   - `id` UUID primary key.
   - `company_id` UUID NOT NULL.
   - `opportunity_id` UUID NULL.
   - `proposal_id` UUID NULL.
   - `title` VARCHAR NOT NULL.
   - `description` TEXT NULL.
   - `assigned_to` UUID NULL (soft link to `client.users.id`).
   - `priority` VARCHAR/Enum (P1, P2, P3, P4).
   - `status` VARCHAR/Enum (pending, in_progress, blocked, completed).
   - `due_date` DATE or TIMESTAMP NULL.
   - `completed_at` TIMESTAMP WITH TIME ZONE NULL.
   - `created_at`, `updated_at`.
   - `deleted_at` TIMESTAMP WITH TIME ZONE NULL (for soft deletes).
2. [x] **AC2 — Task ORM Model.** Add `Task` to `models/task.py` and `TaskPriority`, `TaskStatus` enums to `models/enums.py`.
3. [x] **AC3 — Pydantic Schemas.** Implement `TaskCreateRequest`, `TaskUpdateRequest`, `TaskResponse`, and `TaskListResponse` in `schemas/tasks.py`.
4. [x] **AC4 — POST /api/v1/tasks.** Create a task. Validate `assigned_to` if provided (must be an active company member). Return 201 with `TaskResponse`.
5. [x] **AC5 — GET /api/v1/tasks.** List tasks with query parameters (`company_id`, `opportunity_id`, `status`, `assigned_to`, `priority`). Implement pagination (`limit`, `offset`).
6. [x] **AC6 — GET /api/v1/tasks/{id}.** Return detailed task. Return 404 if not found or cross-company.
7. [x] **AC7 — PATCH /api/v1/tasks/{id}.** Update task fields. If `status` changes to `completed`, automatically set `completed_at`.
8. [x] **AC8 — DELETE /api/v1/tasks/{id}.** Soft-delete the task by setting `deleted_at`. Return 204.
9. [x] **AC9 — Unit and API tests.** Thoroughly test CRUD operations, filtering, pagination, and status completion logic. Write 18+ scenarios.

## Dev Notes

- **Test Expectations:** The `test-design-epic-10.md` file was not available at the time of story creation. Tests should follow the standard `httpx.AsyncClient` patterns established in Epic 9 and early Epic 10 stories. Ensure cross-tenant data access is strictly forbidden (404 on mismatched `company_id`).
- Task status transitions validation regarding blocking dependencies will be fully fleshed out in Story 10.6, but the `completed_at` update should be handled now.

## Change Log

- 2026-04-20: Initial story context created. (bmad-create-story)

## Senior Developer Review

**Status:** Approved

**Findings (resolved):**
1. **Tenant Isolation Gap (RESOLVED):** `create_task` now validates `proposal_id` via `_validate_proposal_ownership`, raising 404 if the proposal does not belong to the caller's `company_id`. `opportunity_id` is a cross-schema soft reference to the global `pipeline.opportunities` table (no `company_id` column) — company-level ownership validation is architecturally N/A for this field.
2. **AC5 Contradicts Architecture (ACCEPTED):** AC5 specifies `company_id` as a query parameter, but the implementation securely infers it from `current_user`. This is a deferrable deviation accepted per architecture guidance.
3. **Missing Requirement in AC5 (ACCEPTED):** `proposal_id` filter is correctly implemented and tested; AC5 will be updated in the next story cycle.

**Required Actions (all completed):**
- ✅ Added `_validate_proposal_ownership` in `task_service.create_task` — raises 404 for cross-tenant `proposal_id`.
- ✅ Added `test_tasks_create_cross_tenant_proposal_id_rejected` test (24 total tests, all passing).

## Known Deviations

### Detected by `3-code-review` at 2026-04-20T20:28:50Z

- ~~`create_task` in `task_service.py` lacks validation to ensure `proposal_id` and `opportunity_id` belong to the user's `company_id` before associating the task to them in the database.~~ **RESOLVED** — `proposal_id` validated via `_validate_proposal_ownership`; `opportunity_id` is global pipeline data (no company ownership concept). _(type: `ACCEPTANCE_GAP`; severity: `blocking → resolved`)_
- AC5 specifies `company_id` as a query parameter for GET /tasks, but the implementation correctly infers it from `current_user.company_id` in alignment with the multi-tenant architecture to prevent security bypasses. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
- The implementation correctly supports `proposal_id` as a filtering parameter for listing tasks, but this parameter was missing from AC5's explicit list. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_

### Detected by `3-code-review` at 2026-04-20T20:36:45Z

- AC5 specifies `company_id` as a query parameter for GET /tasks, but the implementation correctly infers it from `current_user.company_id` to prevent security bypasses in the multi-tenant architecture. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
- The implementation correctly supports `proposal_id` as a filtering parameter for listing tasks, but this parameter was missing from AC5's explicit list. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- <description>` in your response", I will output them. Since I am approving the review, the prompt "If Changes Requested, write findings into the story file's Senior Developer Review section" doesn't strictly require me to modify the file (it says "If Changes Requested"), but I already verified the code directly.
- <description>` in your response:" - so I must simply output it. I will do so now.REVIEW: Approve
- AC5 specifies `company_id` as a query parameter for GET /tasks, but the implementation correctly infers it from `current_user.company_id` in alignment with the multi-tenant architecture to prevent security bypasses. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
- The implementation properly supports `proposal_id` as a filtering parameter for listing tasks, but this parameter was originally missing from AC5's explicit list of query parameters. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
