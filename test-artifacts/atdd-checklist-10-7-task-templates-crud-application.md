# ATDD Checklist: Story 10.7 — Task Templates CRUD & Application

**Status:** RED (Not Implemented — tests must fail before dev begins)
**Story:** `eusolicit-docs/implementation-artifacts/10-7-task-templates-crud-application.md`
**Test files:**
- `services/client-api/tests/api/test_task_templates.py`
- `services/client-api/tests/unit/test_task_template_service.py`

> **Coverage note:** No epic-level test design doc exists for Epic 10 (see story Dev Notes). Coverage strategy follows the established patterns from Epic 7 and Epic 9 ATDD checklists: `httpx.AsyncClient` for API tests, `async with AsyncSession` for unit/service tests, `pytest-asyncio` with `asyncio_mode = "auto"`. Cross-tenant isolation returns 404 (not 403) per the existence-leakage guarantee consistent with Stories 7.2, 9.3, 10.1, 10.5, and 10.6.

---

## AC1 — Alembic Migration `041_task_templates`

- [ ] **`test_migration_041_creates_task_templates_table`**: After running `041_task_templates.py` upgrade, `client.task_templates` exists with all required columns (`id`, `company_id`, `name`, `description`, `opportunity_type`, `stages`, `created_by`, `created_at`, `updated_at`, `deleted_at`).
- [ ] **`test_migration_041_column_constraints`**: `opportunity_type` CHECK constraint rejects values outside `('tender', 'grant', 'any')`. `stages` JSONB CHECK constraint rejects an empty array and an array with > 50 elements.
- [ ] **`test_migration_041_partial_unique_index`**: Two active rows with the same `(company_id, name)` raise `IntegrityError`. A row with matching `(company_id, name)` but non-NULL `deleted_at` does NOT trigger the constraint.
- [ ] **`test_migration_041_indexes_exist`**: `ix_task_templates_company_id`, `ix_task_templates_opportunity_type`, and `ix_task_templates_deleted_at` are present in `pg_indexes`.
- [ ] **`test_migration_041_role_grants`**: `orchestrator_client_api` role has `SELECT, INSERT, UPDATE, DELETE` on `client.task_templates`.
- [ ] **`test_migration_041_downgrade_drops_table`**: Running `downgrade()` removes `client.task_templates` and its indexes.

---

## AC2 — `TaskTemplate` ORM Model

- [ ] **`test_orm_task_template_importable`**: `from client_api.models import TaskTemplate` succeeds without import error.
- [ ] **`test_orm_task_template_schema`**: `TaskTemplate.__table__.schema == "client"`.
- [ ] **`test_orm_task_template_stages_jsonb`**: Column type for `stages` is `JSON` / PostgreSQL JSONB (not a relationship or separate table).
- [ ] **`test_orm_task_template_indexes_and_constraints`**: `__table_args__` includes the partial unique index and all three standard indexes.
- [ ] **`test_orm_task_template_no_task_relationship`**: `TaskTemplate` has no back-reference relationship to `Task` (templates are factories, not parents).

---

## AC3 — Pydantic Schemas

### `StageDefinition`

- [ ] **`test_schema_stage_definition_valid`**: A stage with `title="Draft"`, `role="bid_manager"`, `relative_days_before_deadline=7`, `dependency_indices=[]` passes validation.
- [ ] **`test_schema_stage_definition_title_too_short`**: `title=""` raises `ValidationError`.
- [ ] **`test_schema_stage_definition_title_too_long`**: `title` of 501 chars raises `ValidationError`.
- [ ] **`test_schema_stage_definition_description_too_long`**: `description` > 2000 chars raises `ValidationError`.
- [ ] **`test_schema_stage_definition_invalid_role`**: `role="superadmin"` raises `ValidationError`.
- [ ] **`test_schema_stage_definition_relative_days_negative`**: `relative_days_before_deadline=-1` raises `ValidationError`.
- [ ] **`test_schema_stage_definition_relative_days_too_large`**: `relative_days_before_deadline=366` raises `ValidationError`.
- [ ] **`test_schema_stage_definition_forward_reference`**: Stage at index 1 with `dependency_indices=[2]` raises `ValidationError` (forward reference).
- [ ] **`test_schema_stage_definition_self_reference`**: Stage at index 2 with `dependency_indices=[2]` raises `ValidationError` (self-reference).
- [ ] **`test_schema_stage_definition_duplicate_in_indices`**: `dependency_indices=[0, 0]` raises `ValidationError` (duplicate index).

### `TaskTemplateCreateRequest`

- [ ] **`test_schema_create_request_valid`**: A valid request with 3 stages, valid dependency chain passes.
- [ ] **`test_schema_create_request_empty_stages`**: `stages=[]` raises `ValidationError`.
- [ ] **`test_schema_create_request_too_many_stages`**: 51 stages raises `ValidationError`.
- [ ] **`test_schema_create_request_invalid_opportunity_type`**: `opportunity_type="construction"` raises `ValidationError`.
- [ ] **`test_schema_create_request_out_of_range_dependency_index`**: Stage with `dependency_indices=[7]` when only 3 stages exist raises `ValidationError`.
- [ ] **`test_schema_create_request_root_validator_cross_stage_forward_ref`**: Root validator catches forward reference not caught by per-stage validator (e.g. last stage references a non-existent future index).

### `TaskTemplateUpdateRequest`

- [ ] **`test_schema_update_request_all_optional`**: An empty `{}` body passes (no required fields).
- [ ] **`test_schema_update_request_stages_full_replacement`**: Providing `stages` replaces the full list; partial-stage merging is not supported.

### `TaskTemplateApplyRequest` / `TaskTemplateApplyResponse`

- [ ] **`test_schema_apply_request_valid`**: `{"opportunity_id": "<uuid>"}` passes.
- [ ] **`test_schema_apply_response_structure`**: Response contains `created_task_ids`, `created_dependency_ids`, and `warnings` fields.

---

## AC4 — `POST /api/v1/task-templates`

- [ ] **`test_create_template_success_201`**: `bid_manager` POSTs a valid payload with 3 stages; response is 201 with a `TaskTemplateResponse` containing the correct `company_id`, `created_by`, and `stages`.
- [ ] **`test_create_template_sets_company_id_from_token`**: `company_id` in the response equals the caller's company, not an arbitrary value in the request body.
- [ ] **`test_create_template_default_opportunity_type_any`**: Omitting `opportunity_type` defaults to `"any"`.
- [ ] **`test_create_template_409_duplicate_name`**: POSTing a second template with the same `name` in the same company returns 409 with detail `"A template with this name already exists"`.
- [ ] **`test_create_template_writes_audit_log`**: After successful create, `audit_log` contains a row with `entity_type="task_template"`, `action="create"`, `entity_id=<new template id>`.
- [ ] **`test_create_template_403_non_bid_manager`**: A `technical_writer` role user attempting to create a template receives 403.
- [ ] **`test_create_template_admin_bypass`**: A company admin (not `bid_manager` role) can create a template and receives 201.
- [ ] **`test_create_template_422_forward_reference_in_stages`**: Stage 1 references index 2 → 422.
- [ ] **`test_create_template_422_self_reference_in_stages`**: Stage 0 references index 0 → 422.
- [ ] **`test_create_template_422_out_of_range_index`**: Stage 1 with `dependency_indices=[7]` (3-stage payload) → 422.
- [ ] **`test_create_template_422_duplicate_dependency_index`**: `dependency_indices=[0, 0]` in any stage → 422.
- [ ] **`test_create_template_422_empty_stages`**: `stages=[]` → 422.
- [ ] **`test_create_template_422_stages_exceeds_50`**: 51 stages → 422.
- [ ] **`test_create_template_422_invalid_opportunity_type`**: `opportunity_type="construction"` → 422.

---

## AC5 — `GET /api/v1/task-templates`

- [ ] **`test_list_templates_any_authenticated_user`**: A `read_only` company member (not bid_manager) can successfully list templates.
- [ ] **`test_list_templates_company_scoped`**: Returns only templates belonging to `current_user.company_id`; templates from other companies are absent.
- [ ] **`test_list_templates_excludes_soft_deleted`**: A soft-deleted template does not appear in the list.
- [ ] **`test_list_templates_filter_tender`**: `?opportunity_type=tender` returns `tender` and `any` templates; `grant`-only templates are absent.
- [ ] **`test_list_templates_filter_grant`**: `?opportunity_type=grant` returns `grant` and `any` templates; `tender`-only templates are absent.
- [ ] **`test_list_templates_no_filter_returns_all`**: Without `opportunity_type` filter, all active company templates are returned.
- [ ] **`test_list_templates_pagination_limit_offset`**: `?limit=2&offset=1` returns the correct page slice; `total` reflects the full count.
- [ ] **`test_list_templates_limit_max_100`**: `?limit=101` returns 422 (exceeds maximum).
- [ ] **`test_list_templates_ordered_by_created_at_desc`**: Newest template appears first; stable secondary sort by `id ASC` produces deterministic pages.
- [ ] **`test_list_templates_returns_task_template_list_response`**: Response shape matches `TaskTemplateListResponse` (`items`, `total`, `limit`, `offset`).

---

## AC6 — `GET /api/v1/task-templates/{id}`

- [ ] **`test_get_template_success`**: Returns 200 `TaskTemplateResponse` for a valid id owned by the caller's company.
- [ ] **`test_get_template_404_not_found`**: Non-existent UUID returns 404.
- [ ] **`test_get_template_404_cross_tenant`**: Valid UUID belonging to another company returns 404 (existence-leakage-safe).
- [ ] **`test_get_template_404_soft_deleted`**: Soft-deleted template returns 404.
- [ ] **`test_get_template_any_authenticated_user`**: A `read_only` user can retrieve a template (no bid_manager gate on reads).

---

## AC7 — `PATCH /api/v1/task-templates/{id}`

- [ ] **`test_patch_template_success_200`**: Updates `name` and `description`; returns 200 with updated `TaskTemplateResponse`.
- [ ] **`test_patch_template_stages_full_replacement`**: Providing a new `stages` array replaces the entire list (not merged); updated `stages` are reflected in the response.
- [ ] **`test_patch_template_updated_at_refreshed`**: `updated_at` timestamp is greater than `created_at` after patch.
- [ ] **`test_patch_template_404_not_found`**: Returns 404 for non-existent id.
- [ ] **`test_patch_template_404_cross_tenant`**: Returns 404 when patching a template belonging to another company.
- [ ] **`test_patch_template_404_soft_deleted`**: Returns 404 for a soft-deleted template.
- [ ] **`test_patch_template_409_rename_collision`**: Renaming a template to an existing active name in the same company returns 409.
- [ ] **`test_patch_template_rename_to_deleted_name_allowed`**: Renaming to the name of a soft-deleted template succeeds (partial unique constraint ignores `deleted_at IS NOT NULL` rows).
- [ ] **`test_patch_template_403_non_bid_manager`**: A non-bid_manager returns 403.
- [ ] **`test_patch_template_admin_bypass`**: Company admin can patch successfully.
- [ ] **`test_patch_template_writes_audit_log`**: `audit_log` row with `entity_type="task_template"`, `action="update"` exists after patch.
- [ ] **`test_patch_template_422_invalid_stages`**: Patching with a stages array containing a forward reference returns 422.
- [ ] **`test_patch_template_stages_dag_revalidated`**: Service layer re-runs DAG assertion on replacement `stages`; a stages array with a cycle-equivalent forward index returns 422.

---

## AC8 — `DELETE /api/v1/task-templates/{id}`

- [ ] **`test_delete_template_success_204`**: Returns 204; subsequent GET returns 404.
- [ ] **`test_delete_template_soft_delete_not_hard_delete`**: After DELETE, the DB row still exists with `deleted_at IS NOT NULL`.
- [ ] **`test_delete_template_404_not_found`**: Non-existent id returns 404.
- [ ] **`test_delete_template_404_cross_tenant`**: Another company's template returns 404.
- [ ] **`test_delete_template_404_already_deleted`**: Attempting to delete an already-soft-deleted template returns 404.
- [ ] **`test_delete_template_allows_same_name_after_soft_delete`**: After DELETE, a new template with the same `(company_id, name)` can be created (partial unique index does not block).
- [ ] **`test_delete_template_removed_from_list`**: After DELETE, the template no longer appears in `GET /api/v1/task-templates`.
- [ ] **`test_delete_template_403_non_bid_manager`**: Non-bid_manager returns 403.
- [ ] **`test_delete_template_admin_bypass`**: Company admin can delete successfully.
- [ ] **`test_delete_template_writes_audit_log`**: `audit_log` row with `entity_type="task_template"`, `action="delete"` exists after delete.

---

## AC9 — `POST /api/v1/task-templates/{id}/apply`

### Happy Path — Linear Chain (0 → 1 → 2)

- [ ] **`test_apply_linear_chain_returns_201`**: 3-stage template with chain `0→1→2` applied to an opportunity with a deadline in the future; response is 201 `TaskTemplateApplyResponse` with `created_task_ids` (3 items) and `created_dependency_ids` (2 items).
- [ ] **`test_apply_linear_chain_task_due_dates`**: Each created task's `due_date = opportunity.deadline - timedelta(days=relative_days_before_deadline)` matches the expected value.
- [ ] **`test_apply_linear_chain_opportunity_id_set`**: All created tasks have `opportunity_id = <applied opportunity>`.
- [ ] **`test_apply_linear_chain_proposal_id_null`**: All created tasks have `proposal_id IS NULL`.
- [ ] **`test_apply_linear_chain_assigned_to_null`**: All created tasks have `assigned_to IS NULL`.
- [ ] **`test_apply_linear_chain_root_task_pending`**: The root task (stage 0, no dependencies) has `status = pending`.
- [ ] **`test_apply_linear_chain_dependents_blocked`**: Stage 1 and stage 2 (which have upstream FTS dependencies) have `status = blocked`.
- [ ] **`test_apply_linear_chain_dependency_type_finish_to_start`**: All emitted `task_dependencies` rows have `dependency_type = 'finish_to_start'`.

### Happy Path — Diamond Topology (0→1, 0→2, 1→3, 2→3)

- [ ] **`test_apply_diamond_topology_counts`**: 4-stage diamond returns 4 `created_task_ids` and 4 `created_dependency_ids`.
- [ ] **`test_apply_diamond_topology_blocked_status`**: Stages 1, 2, and 3 are `blocked`; stage 0 is `pending`.
- [ ] **`test_apply_diamond_topology_stage3_has_two_dependencies`**: Stage 3 has two `TaskDependency` rows (one from stage 1, one from stage 2).
- [ ] **`test_apply_diamond_topology_unblock_path`**: After marking stage 0 `completed` via PATCH, stage 1 and stage 2 transition to `pending` (Story 10.6 auto-unblock logic exercised indirectly).

### Warnings

- [ ] **`test_apply_past_due_date_warning_emitted`**: When `relative_days_before_deadline > (deadline - now()).days`, the response `warnings` contains an entry for that stage index, and the task is still created with the past `due_date`.
- [ ] **`test_apply_past_due_date_task_still_created`**: Even with a past `due_date`, the task row exists in the database with the computed past date.
- [ ] **`test_apply_type_mismatch_warning`**: Template with `opportunity_type='tender'` applied to a `grant` opportunity returns a type-mismatch warning in `warnings` but still returns 201 and creates all tasks.
- [ ] **`test_apply_any_template_no_type_mismatch_warning`**: Template with `opportunity_type='any'` applied to any opportunity type produces no type-mismatch warning.

### Error Cases

- [ ] **`test_apply_404_template_not_found`**: Applying a non-existent template id → 404 `"Template not found"`.
- [ ] **`test_apply_404_soft_deleted_template`**: Applying a soft-deleted template → 404 `"Template not found"`.
- [ ] **`test_apply_404_cross_tenant_template`**: Template from another company → 404 `"Template not found"`.
- [ ] **`test_apply_404_opportunity_not_found`**: Non-existent `opportunity_id` → 404 `"Opportunity not found"`.
- [ ] **`test_apply_404_cross_tenant_opportunity`**: Opportunity not visible to `current_user.company_id` (per `ensure_opportunity_visible`) → 404 `"Opportunity not found"`.
- [ ] **`test_apply_422_opportunity_no_deadline`**: Opportunity with `deadline IS NULL` → 422 `"Cannot apply template: opportunity has no deadline"`.
- [ ] **`test_apply_403_non_bid_manager`**: Non-bid_manager company member → 403.
- [ ] **`test_apply_admin_bypass`**: Company admin can apply successfully.

### Audit

- [ ] **`test_apply_writes_audit_log`**: After successful apply, `audit_log` contains a row with `entity_type="task_template_application"`, `action="apply"`, `entity_id=<template id>`, and `metadata` contains `opportunity_id` and `created_task_count`.

---

## AC10 — Additional Test Coverage

### Cross-Tenant Isolation

- [ ] **`test_cross_tenant_get_returns_404`**: Company B's `bid_manager` GETs company A's template → 404.
- [ ] **`test_cross_tenant_patch_returns_404`**: Company B's `bid_manager` PATCHes company A's template → 404.
- [ ] **`test_cross_tenant_delete_returns_404`**: Company B's `bid_manager` DELETEs company A's template → 404.
- [ ] **`test_cross_tenant_apply_returns_404`**: Company B's `bid_manager` APPLYs company A's template → 404 `"Template not found"`.

### Soft-Delete / Name Recycle

- [ ] **`test_soft_delete_then_create_same_name`**: Delete a template, then create a new one with the identical `(company_id, name)` — returns 201 (partial unique index admits it).
- [ ] **`test_soft_delete_not_in_list`**: Soft-deleted template is absent from `GET /api/v1/task-templates`.
- [ ] **`test_soft_delete_get_returns_404`**: GET of soft-deleted id returns 404.

### Transaction Atomicity (Apply)

- [ ] **`test_apply_rollback_on_second_task_insert_failure`**: Inject an error during the second task insert (e.g. mock `session.flush` to raise on the second call); assert no `client.tasks` rows, no `client.task_dependencies` rows, and no `audit_log` row were persisted.
- [ ] **`test_apply_rollback_on_dependency_insert_failure`**: Inject an error during dependency insertion (pass 2); assert all task rows and dependency rows are rolled back atomically.

### RBAC Completeness

- [ ] **`test_rbac_create_403_for_technical_writer`**: `technical_writer` → 403 on POST.
- [ ] **`test_rbac_create_403_for_read_only`**: `read_only` → 403 on POST.
- [ ] **`test_rbac_patch_403_for_financial_analyst`**: `financial_analyst` → 403 on PATCH.
- [ ] **`test_rbac_delete_403_for_legal_reviewer`**: `legal_reviewer` → 403 on DELETE.
- [ ] **`test_rbac_apply_403_for_technical_writer`**: `technical_writer` → 403 on apply.
- [ ] **`test_rbac_list_200_for_read_only`**: `read_only` → 200 on GET list (reads open to all authenticated company members).
- [ ] **`test_rbac_get_200_for_read_only`**: `read_only` → 200 on GET detail.

### Audit Coverage

- [ ] **`test_audit_create_row_exists`**: After create, `audit_log` has `action="create"`, `entity_type="task_template"`.
- [ ] **`test_audit_update_row_exists`**: After patch, `audit_log` has `action="update"`, `entity_type="task_template"`.
- [ ] **`test_audit_delete_row_exists`**: After delete, `audit_log` has `action="delete"`, `entity_type="task_template"`.
- [ ] **`test_audit_apply_row_exists`**: After apply, `audit_log` has `action="apply"`, `entity_type="task_template_application"`, and `metadata.created_task_count` matches the number of stages.

---

## Unit Tests (`test_task_template_service.py`)

### Pure-function DAG validation

- [ ] **`test_validate_stages_valid_chain`**: `[0→[], 1→[0], 2→[1]]` passes without error.
- [ ] **`test_validate_stages_valid_diamond`**: `[0→[], 1→[0], 2→[0], 3→[1,2]]` passes without error.
- [ ] **`test_validate_stages_forward_reference`**: Stage 0 referencing index 1 raises `ValueError`.
- [ ] **`test_validate_stages_self_reference`**: Stage 2 with `dependency_indices=[2]` raises `ValueError`.
- [ ] **`test_validate_stages_out_of_range_index`**: Any stage referencing index ≥ len(stages) raises `ValueError`.
- [ ] **`test_validate_stages_duplicate_within_stage`**: `[0, 0]` in a single stage's `dependency_indices` raises `ValueError`.
- [ ] **`test_validate_stages_empty_list`**: Empty `stages` raises `ValueError`.
- [ ] **`test_validate_stages_51_stages`**: List of 51 stages raises `ValueError`.

### Apply service logic (unit, no DB)

- [ ] **`test_apply_service_computes_due_date_correctly`**: Given `deadline=date(2026,6,1)` and `relative_days_before_deadline=10`, computed `due_date == date(2026,5,22)`.
- [ ] **`test_apply_service_past_due_date_emits_warning`**: When `due_date < now()`, the service appends a warning string containing the stage index and computed date.
- [ ] **`test_apply_service_type_mismatch_emits_warning`**: `template.opportunity_type='tender'` vs `opportunity.opportunity_type='grant'` → warning string contains both types.
- [ ] **`test_apply_service_any_template_no_warning`**: `template.opportunity_type='any'` → no type-mismatch warning regardless of opportunity type.
- [ ] **`test_apply_service_dependent_tasks_blocked`**: Service sets `status=blocked` on tasks that have at least one FTS dependency; tasks with no dependencies remain `pending`.

---

## Setup & Fixtures Required

- [ ] `bid_manager_client` fixture: authenticated `httpx.AsyncClient` + JWT for a `bid_manager` company member.
- [ ] `read_only_client` fixture: authenticated client for a `read_only` company member.
- [ ] `company_b_bid_manager_client` fixture: client for a different company (cross-tenant isolation).
- [ ] `admin_client` fixture: company admin client.
- [ ] `opportunity_with_deadline` fixture: seeded `pipeline.opportunities` row with a future `deadline` and `opportunity_type`.
- [ ] `opportunity_without_deadline` fixture: seeded opportunity with `deadline=NULL`.
- [ ] `template_payload` factory: helper returning a valid `TaskTemplateCreateRequest`-shaped dict with a customisable number of stages.
- [ ] `audit_log_query` helper: test-only async function to fetch the latest `audit_log` row by `entity_type` + `action`.
- [ ] DB session accessible from tests to assert raw row state (soft-delete, rollback verification).
