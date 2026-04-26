# Story 10.6: Task Dependencies & DAG Validation

Status: complete

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-6-task-dependencies-dag-validation
- **Points:** 5
- **Type:** backend
- **Module:** Client API (`client_api.api.v1.tasks`, `client_api.models.task`, `client_api.schemas.tasks`, `client_api.services.task_service`, new Alembic migration)
- **Priority:** P0 (Core workflow requirement)
- **Depends On:** Story 10.5 (Task CRUD API)

## Story

As a **bid_manager**,
I want **to create dependencies between tasks and prevent cyclic dependencies**,
so that **I can model complex workflows accurately without causing deadlocks.**

## Description

This story implements the `task_dependencies` table to track task prerequisites. Endpoints to add and remove dependencies (`finish_to_start` and `finish_to_finish`) will be implemented. Crucially, before inserting a new dependency, a DAG cycle detection algorithm (e.g., DFS/BFS) must verify that the new dependency will not create a cycle; if it does, a 422 error must be returned. The task status transition logic implemented in S10.05 will be enhanced: a task cannot transition to `completed` if any `finish_to_start` upstream dependency is not `completed`, or if `finish_to_finish` dependencies are not `in_progress` or `completed`. When a task is marked `completed`, downstream tasks in `blocked` status should automatically transition to `pending` if they are unblocked.

## Acceptance Criteria

1. [x] **AC1 — `client.task_dependencies` Alembic migration.** Create a new migration file. The table should have:
   - `id` UUID primary key
   - `task_id` UUID NOT NULL (FK to tasks)
   - `depends_on_task_id` UUID NOT NULL (FK to tasks)
   - `dependency_type` VARCHAR/Enum (finish_to_start, finish_to_finish)
   - Unique constraint on `(task_id, depends_on_task_id)`
2. [x] **AC2 — TaskDependency ORM Model & Schemas.** Add `TaskDependency` model and update `Task` model to have a relationship to its dependencies. Create `TaskDependencyCreateRequest` and update `TaskResponse` to include `dependencies`.
3. [x] **AC3 — POST /api/v1/tasks/{id}/dependencies.** Endpoint to add a dependency. Accepts `{depends_on_task_id, dependency_type}`.
4. [x] **AC4 — DAG Cycle Detection.** Before inserting a dependency, perform a DFS/BFS traversal to ensure that `task_id` is not reachable from `depends_on_task_id`. Return 422 if cycle detected.
5. [x] **AC5 — DELETE /api/v1/tasks/{id}/dependencies/{dep_id}.** Endpoint to remove an edge.
6. [x] **AC6 — Enhance Task Completion Logic.** Update the PATCH task logic:
   - When transitioning to `completed`: Verify all upstream `finish_to_start` dependencies are `completed`.
   - When transitioning to `completed`: Verify all upstream `finish_to_finish` dependencies are `completed` or `in_progress`.
   - If conditions not met, return 422.
7. [x] **AC7 — Downstream Auto-Unblock.** When a task transitions to `completed`, check if any downstream tasks in `blocked` status can now start. If all of a downstream task's `finish_to_start` dependencies are `completed`, transition it to `pending`.
8. [x] **AC8 — Unit and API tests.** Write tests for:
   - Simple chain
   - Diamond dependency
   - Cycle rejection (simple and multi-hop)
   - Status blocking logic
   - Auto-unblock logic

## Dev Notes

- **Test Expectations:** The `test-design-epic-10.md` file is not currently available in `test-artifacts/`. Tests should follow standard `httpx.AsyncClient` patterns and focus heavily on edge cases for the graph traversal (cycle detection) and correct auto-unblocking behavior.

## Change Log

- 2026-04-20: Initial story context created. (bmad-create-story)
- 2026-04-21: Added diamond dependency test (`test_diamond_dependency_accepted_and_multi_parent_unblock`) covering AC8. All 12 tests pass. Story marked complete. (bmad-dev-story)

## Senior Developer Review

**REVIEW: Approve**

### Findings (final review 2026-04-21):

The sole blocker from the prior review — the missing diamond-topology test for AC8 — has been addressed. `test_diamond_dependency_accepted_and_multi_parent_unblock` in `services/client-api/tests/api/test_task_dependencies.py` exercises the full fan-out / fan-in acyclic graph:
- Wires up A→B, A→C, B→D, C→D and asserts all four POST /dependencies return 201 (DFS correctly accepts the acyclic diamond).
- Verifies A's dependency list contains both B and C via GET.
- Completes D and asserts B and C auto-unblock to `pending` while A remains `blocked` (multi-parent FTS fan-in check).
- Completes B; asserts A stays `blocked` while C is still pending (no premature unblock).
- Completes C; asserts A finally auto-unblocks to `pending`.

Full suite: 12/12 tests pass locally (`pytest tests/api/test_task_dependencies.py`), covering happy path, 404, duplicate, direct cycle, 3-node multi-hop cycle, deletion, FTS completion gating (fail + succeed), FTF completion gating (fail + succeed), single-parent auto-unblock, and the new diamond multi-parent topology.

### What looks good:

- **AC1/AC2 (migration + models):** `040_task_dependencies.py` correctly creates the enum, table, unique constraint on `(task_id, depends_on_task_id)`, both FK indexes, FK CASCADE semantics, and role grants. ORM relationships in `models/task.py` cleanly use `foreign_keys=` to disambiguate the two FKs back to `tasks.id`, with `cascade="all, delete-orphan"` on both sides.
- **AC3/AC5 (endpoints):** `POST /tasks/{id}/dependencies` and `DELETE /tasks/{id}/dependencies/{dep_id}` are role-gated to `bid_manager` via `require_role`, consistent with the rest of the tasks router.
- **AC4 (DAG cycle detection):** `_is_reachable(start=depends_on_task_id, target=task_id)` is directionally correct — it walks the upstream chain from the prospective prerequisite and detects whether the dependent task is already reachable, which is exactly the condition for a cycle when the new edge is inserted. Direct (2-node) and multi-hop (3-node) cycles are covered by tests.
- **AC6 (completion gating):** `_verify_dependencies_for_completion` correctly enforces FTS→`completed` and FTF→(`in_progress`|`completed`), raising `ValidationError` (422) otherwise. Tests cover both happy and failure paths for both dependency types.
- **AC7 (auto-unblock):** `_auto_unblock_downstream` re-evaluates every downstream task's full FTS set before transitioning `blocked → pending`, which correctly handles multi-parent downstream tasks (it will not prematurely unblock a task that still has another incomplete FTS upstream).
- **Misc hardening:** Self-dependency (`task_id == depends_on_task_id`) is rejected; duplicate edges are rejected pre-insert in addition to the DB-level unique constraint; cross-company task IDs are rejected via `get_task` tenant check on both sides before the edge is created.

### Minor observations (non-blocking):

- The cycle-detection DFS issues one SQL query per visited node. For deep graphs this is O(V) round-trips; a single recursive CTE would be faster but is not required at current scale.
- `_auto_unblock_downstream` is invoked after `task.status` is set and flushed, so the completed task's own status is correctly visible to the downstream FTS check — good. Worth a comment stating the ordering is load-bearing.

### Change Log entry:

- 2026-04-21: Re-review complete — diamond test verified present and passing (12/12 suite green). Approved. (bmad-code-review)