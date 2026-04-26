# ATDD Checklist: Story 10-5 Task CRUD API

**Status:** RED (Not Implemented)
**Story:** `eusolicit-docs/implementation-artifacts/10-5-task-crud-api.md`

## Setup & Fixtures
- [ ] Task fixture providing standard authenticated client + session (e.g. `task_client_and_session`)
- [ ] Multi-tenant isolation helper (Client B vs Client A data)
- [ ] Seed standard lookup data (opportunities, proposals, company users) if needed for foreign keys

## Creation (POST /api/v1/tasks)
- [ ] `test_tasks_create_success`: Creates a task with minimum required fields (title, priority, status) returns 201 TaskResponse.
- [ ] `test_tasks_create_with_all_fields`: Creates a task with description, assigned_to, opportunity_id, proposal_id, due_date.
- [ ] `test_tasks_create_invalid_assignee_not_in_company`: Fails (400 or 403) when assigned_to belongs to a different company.
- [ ] `test_tasks_create_invalid_assignee_not_found`: Fails (404 or 400) when assigned_to does not exist.

## Detail (GET /api/v1/tasks/{id})
- [ ] `test_tasks_get_detail_success`: Retrieves task and matches expected Pydantic schema.
- [ ] `test_tasks_get_detail_not_found`: Returns 404 for non-existent ID.
- [ ] `test_tasks_get_detail_cross_tenant_returns_404`: Returns 404 when querying an existing task belonging to another company (Tenant Isolation).

## Listing & Pagination (GET /api/v1/tasks)
- [ ] `test_tasks_list_empty`: Returns empty list with standard pagination metadata when no tasks exist.
- [ ] `test_tasks_list_with_pagination`: Correctly respects limit and offset parameters.
- [ ] `test_tasks_list_filter_status`: Filters tasks by status (e.g. 'pending', 'completed').
- [ ] `test_tasks_list_filter_priority`: Filters tasks by priority (e.g. 'P1').
- [ ] `test_tasks_list_filter_assigned_to`: Filters tasks assigned to a specific user.
- [ ] `test_tasks_list_filter_opportunity_id`: Filters tasks linked to a specific opportunity.
- [ ] `test_tasks_list_tenant_isolation`: Ensures only tasks from the JWT's company_id are returned, regardless of filters.

## Update (PATCH /api/v1/tasks/{id})
- [ ] `test_tasks_update_success`: Updates title and priority, returns 200 TaskResponse.
- [ ] `test_tasks_update_status_to_completed_sets_completed_at`: Updates status to 'completed' and verifies `completed_at` is set to current UTC time.
- [ ] `test_tasks_update_status_from_completed_clears_completed_at`: Updates status from 'completed' back to 'in_progress', verifies `completed_at` is null.
- [ ] `test_tasks_update_cross_tenant_returns_404`: Attempting to update another company's task returns 404.

## Deletion (DELETE /api/v1/tasks/{id})
- [ ] `test_tasks_delete_success_soft_delete`: Returns 204. Subsequent GET returns 404. Database record remains with `deleted_at` populated.
- [ ] `test_tasks_delete_cross_tenant_returns_404`: Attempting to delete another company's task returns 404.
- [ ] `test_tasks_delete_not_found`: Deleting a non-existent task returns 404.

## Data integrity / Enums
- [ ] `test_tasks_invalid_priority_enum`: POST/PATCH rejects invalid priority string with 422.
- [ ] `test_tasks_invalid_status_enum`: POST/PATCH rejects invalid status string with 422.
