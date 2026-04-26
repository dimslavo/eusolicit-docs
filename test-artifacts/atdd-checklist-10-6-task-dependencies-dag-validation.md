# ATDD Checklist: Story 10.6 - Task Dependencies & DAG Validation

**Story:** 10-6-task-dependencies-dag-validation
**Status:** In-Progress

## AC1: Alembic Migration `client.task_dependencies`
- [ ] Write Alembic revision for `task_dependencies` table.
- [ ] Verify `id` is UUID primary key.
- [ ] Verify `task_id` is UUID (FK to tasks).
- [ ] Verify `depends_on_task_id` is UUID (FK to tasks).
- [ ] Verify `dependency_type` is VARCHAR/Enum (e.g., `finish_to_start`, `finish_to_finish`).
- [ ] Verify unique constraint on `(task_id, depends_on_task_id)`.
- [ ] Verify cascade deletes (if task is deleted, its dependencies should also be removed).

## AC2: ORM Model & Schemas
- [ ] Create `TaskDependency` SQLAlchemy model.
- [ ] Update `Task` model to have relationships for upstream (`dependencies`) and downstream dependencies.
- [ ] Create `TaskDependencyCreateRequest` schema (fields: `depends_on_task_id`, `dependency_type`).
- [ ] Update `TaskResponse` schema to include `dependencies`.

## AC3: POST `/api/v1/tasks/{id}/dependencies`
- [ ] Endpoint successfully adds a dependency with valid inputs.
- [ ] Endpoint returns 201 Created and the new dependency record.
- [ ] Endpoint returns 404 Not Found if `task_id` or `depends_on_task_id` do not exist.
- [ ] Endpoint returns 400/409/422 if the dependency already exists (violating unique constraint).

## AC4: DAG Cycle Detection (422)
- [ ] Adding a dependency that forms a direct cycle (A -> B, B -> A) returns 422 Unprocessable Entity.
- [ ] Adding a dependency that forms a multi-hop cycle (A -> B -> C -> A) returns 422 Unprocessable Entity.
- [ ] Error message clearly indicates a cycle was detected.

## AC5: DELETE `/api/v1/tasks/{id}/dependencies/{dep_id}`
- [ ] Endpoint successfully removes an existing dependency.
- [ ] Endpoint returns 204 No Content.
- [ ] Endpoint returns 404 Not Found if the dependency does not exist.

## AC6: Enhance Task Completion Logic
- [ ] Task CANNOT transition to `completed` if any upstream `finish_to_start` dependency is NOT `completed` (returns 422).
- [ ] Task CAN transition to `completed` if all upstream `finish_to_start` dependencies ARE `completed`.
- [ ] Task CANNOT transition to `completed` if any upstream `finish_to_finish` dependency is `pending` or `blocked` (returns 422).
- [ ] Task CAN transition to `completed` if all upstream `finish_to_finish` dependencies ARE `in_progress` or `completed`.

## AC7: Downstream Auto-Unblock
- [ ] When a task is marked `completed`, its downstream tasks in `blocked` state are automatically evaluated.
- [ ] If all upstream `finish_to_start` dependencies of a downstream task are `completed`, transition the downstream task to `pending`.
- [ ] If downstream tasks still have incomplete upstream `finish_to_start` dependencies, they remain `blocked`.

## AC8: Unit and API Tests
- [ ] Write simple chain dependency tests.
- [ ] Write diamond dependency tests.
- [ ] Write cycle rejection tests (simple & multi-hop).
- [ ] Write status blocking logic tests.
- [ ] Write auto-unblock logic tests.