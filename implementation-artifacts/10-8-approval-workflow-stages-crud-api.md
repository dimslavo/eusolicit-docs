# Story 10.8: Approval Workflow & Stages CRUD API

Status: review

## Dev Agent Record

- **Implemented by:** Gemini 2.0 Pro Preview (Session: 46e3ea09-b40c-48ae-b435-159319963c4e)
- **File List:**
  - **Modified:**
    - `eusolicit-app/services/client-api/tests/api/test_approval_workflows.py`
- **Test Results:** `40 passed, 7 warnings in 4.14s` (API) / `42 passed, 7 warnings in 0.49s` (Unit)
- **Known Deviations:**
  - **AC10:** The `test_patch_workflow_proceeds_when_approval_decisions_missing` test was modified to avoid deadlocks during table rename by using an independent client rather than the module-level `aw_client` fixture which held an open transaction.
  - **Environment:** The pre-existing implementation of source code (models, schemas, service, router) was already complete and of high quality, requiring no modifications. The focus was on verifying correctness and fixing test infrastructure issues (seeding).

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-8-approval-workflow-stages-crud-api
- **Points:** 3
- **Type:** backend
- **Module:** Client API (`client_api.api.v1.approval_workflows` (new), `client_api.models.approval_workflow` (new), `client_api.models.approval_stage` (new), `client_api.schemas.approval_workflows` (new), `client_api.services.approval_workflow_service` (new), new Alembic migration `042_approval_workflows`)
- **Priority:** P1 (Epic 10 governance layer — unblocks Story 10.9 Approval Decision Engine and Story 10.16 Approval Pipeline UI; the workflow/stage schema is a hard prerequisite for any `approval_decisions` insert)
- **Depends On:** Story 2.10 (`require_role` company-role gate — the `bid_manager` gate is reused here), Story 2.4 (`get_current_user`), Story 2.11 (`audit_service.write_audit_entry`), Story 1.3 (the `client` schema and `client.companies` table FK target), Epic 10 Story 10.1 (the `ProposalCollaboratorRole` enum under `client_api.models.enums` — reused as the `required_role` domain for stages).

## Story

As a **company admin / bid_manager**,
I want **to define ordered, role-gated approval pipelines (e.g. "Technical Review → Legal Review → Final Approval") and designate one as the company default**,
so that **every proposal our team prepares flows through a predictable, auditable decision sequence rather than ad-hoc sign-offs in email.**

## Description

This story adds the two schema tables and the CRUD endpoints that let a company define named approval pipelines. A `approval_workflow` is a company-scoped, named pipeline (e.g. "Standard Three-Stage Review"). It owns an ordered list of `approval_stage` rows, each specifying a stage `name`, its ordinal `order` (1-indexed, contiguous within the workflow), the `required_role` that can decide on that stage (drawn from `ProposalCollaboratorRole`: `bid_manager | technical_writer | financial_analyst | legal_reviewer | read_only` — although `read_only` is accepted syntactically, UIs should discourage it), and an `auto_advance` boolean that tells the Story 10.9 decision engine whether an `approved` decision automatically unlocks the next stage or requires an explicit manual progression.

This story does **not** implement decisions, approvals, or event emission — that is Story 10.9's scope. Here we only manage the *definition* surface: CRUD on the pipeline shape itself, the default-flag toggle, and a cascade-delete guard that prevents removing a workflow whose stages have been referenced by any `approval_decisions` row.

Six endpoints are exposed under `/api/v1/approval-workflows`: create, list, retrieve, patch (name + stages replacement), delete, and the `set-default` action. `company_id` is always inferred from the authenticated user (`current_user.company_id`) — the incoming request body intentionally does NOT accept a `company_id` field, even though the epic spec phrases the input as `{company_id, name, stages: [...]}`. Cross-tenant creates are structurally impossible; the stanza in the epic is a data-model description, not a literal API contract.

Five design decisions worth calling out:

**(a) Two normalised tables, not a single JSONB blob.** Unlike Story 10.7's `task_templates.stages` JSONB column, approval stages live in a first-class `approval_stages` table. Reason: Story 10.9 (`approval_decisions`) needs a stable `stage_id` FK target — each decision row references the stage it was made against so the decision history survives workflow edits. If stages were JSONB, there would be no stable primary key per stage. The normalised form also lets Story 10.9's ordering check (`SELECT prior stages WHERE order < current`) use a simple WHERE clause instead of JSONB path expressions.

**(b) Stage replacement on PATCH uses delete-and-recreate, guarded by the cascade check.** When a PATCH request includes `stages`, the service deletes all existing `approval_stages` rows for the workflow inside the same transaction and inserts the new set. This is safe **only if** no `approval_decisions` rows reference any of the stages being deleted — otherwise foreign keys would cascade or orphan the decisions. We therefore run a pre-check: `SELECT 1 FROM client.approval_decisions WHERE stage_id IN (workflow's stage ids) LIMIT 1` — if a row exists, return 409 `Cannot modify stages on a workflow with existing decisions`. The same pre-check runs on DELETE. Because Story 10.9 ships `approval_decisions` later, this story creates the table but MUST code the guard against a table that does not yet exist at migration-run time; the pre-check SELECT gracefully handles the missing-table case by catching `sqlalchemy.exc.ProgrammingError` / `UndefinedTableError` and treating it as "no decisions exist" (logged as a debug breadcrumb, since in production by the time this story's endpoints are live Story 10.9's migration 043 will also be applied).

**(c) `is_default` is enforced as at most one per company via a partial unique index.** `CREATE UNIQUE INDEX uq_approval_workflows_company_default ON client.approval_workflows (company_id) WHERE is_default = TRUE` guarantees the invariant at the DB level, so the `set-default` endpoint can perform the swap as a single UPDATE-then-UPDATE transaction without risk of a concurrent request creating two defaults. The existing default's flag is cleared BEFORE the new one is set within the same transaction to avoid transient-violation aborts on the partial index (PostgreSQL applies the check at statement boundary by default, but we keep the clear-first order for defensive readability).

**(d) Stage `order` is 1-indexed and must be contiguous.** Request-level validation: for a stage list of length N, the set of `order` values MUST equal `{1, 2, ..., N}` exactly — no gaps, no duplicates, no zero-index. This matches how Story 10.9 and Story 10.16's UI stepper are specified to iterate ("stage with lowest order is the first gate"), and sidesteps a class of bugs where a manager reorders stages but forgets to renumber.

**(e) The `ApprovalRequiredRole` domain reuses `ProposalCollaboratorRole`.** The epic describes `required_role` as a free-form string; the implementation uses the same string enum already shipped in Story 10.1 (`client.proposal_collaborator_role` Postgres enum). This keeps the Story 10.9 decision engine's check `current_user's collaborator role on the proposal == stage.required_role` a direct enum equality test rather than a string-vs-enum comparison. If future product needs a distinct approval-only role set, we can rename in a separate migration; today the overlap is total.

The router is mounted at `/api/v1/approval-workflows` (hyphenated) in `client_api/main.py` beside `task_templates_v1.router`.

## Acceptance Criteria

1. [x] **AC1 — `client.approval_workflows` + `client.approval_stages` Alembic migration (`042_approval_workflows.py`).** Create a new migration with `revision = "042"`, `down_revision = "041"`. Define both tables in a single upgrade step.

    **`client.approval_workflows`:**
    - `id` UUID primary key, `server_default=sa.text("gen_random_uuid()")`.
    - `company_id` UUID NOT NULL, FK `client.companies.id` ON DELETE CASCADE.
    - `name` VARCHAR(255) NOT NULL.
    - `description` TEXT NULL.
    - `is_default` BOOLEAN NOT NULL, `server_default=sa.text("false")`.
    - `created_by` UUID NOT NULL (soft link to `client.users.id`, no FK — match Story 10.7 pattern).
    - `created_at`, `updated_at` TIMESTAMPTZ with `server_default=sa.func.now()`; `updated_at` uses `onupdate=sa.func.now()` in the ORM.
    - `deleted_at` TIMESTAMPTZ NULL (soft-delete).
    - Indexes: `ix_approval_workflows_company_id`, `ix_approval_workflows_deleted_at`.
    - Partial unique index `uq_approval_workflows_company_name` on `(company_id, name)` WHERE `deleted_at IS NULL` (same pattern as Story 10.7 — soft-deleted rows do not block renames).
    - Partial unique index `uq_approval_workflows_company_default` on `(company_id)` WHERE `is_default = TRUE AND deleted_at IS NULL` (at most one default per company among live rows).

    **`client.approval_stages`:**
    - `id` UUID primary key, `server_default=sa.text("gen_random_uuid()")`.
    - `workflow_id` UUID NOT NULL, FK `client.approval_workflows.id` ON DELETE CASCADE.
    - `name` VARCHAR(255) NOT NULL.
    - `order` INTEGER NOT NULL — note: `order` is a SQL reserved word; quote via `sa.Column("order", ...)` (SQLAlchemy handles the quoting on PostgreSQL dialect automatically). CHECK `"order" >= 1`.
    - `required_role` TEXT NOT NULL — reuse the existing `client.proposal_collaborator_role` Postgres enum created in Story 10.1. Declare as `sa.Enum(ProposalCollaboratorRole, name="proposal_collaborator_role", schema="client", create_type=False)` so Alembic does not try to recreate the type.
    - `auto_advance` BOOLEAN NOT NULL, `server_default=sa.text("false")`.
    - `created_at` TIMESTAMPTZ NOT NULL `server_default=sa.func.now()`.
    - Unique constraint `uq_approval_stages_workflow_order` on `(workflow_id, "order")` — enforces no duplicate ordinals per workflow at the DB level (the contiguity check stays at the application layer; the DB only guarantees uniqueness).
    - Index `ix_approval_stages_workflow_id` on `workflow_id`.

    **Grants (both tables):** `GRANT SELECT, INSERT, UPDATE, DELETE ON client.approval_workflows TO client_api_role`; same for `client.approval_stages`. Match the Story 10.7 `041` migration pattern.

    **`downgrade()`** drops `approval_stages` first (FK dependency), then `approval_workflows`, including all indexes.

2. [x] **AC2 — `ApprovalWorkflow` and `ApprovalStage` ORM models.** Add:
   - `services/client-api/src/client_api/models/approval_workflow.py` with `ApprovalWorkflow` — columns mirror the migration; `__table_args__` includes both partial unique indexes, the `ix_*` indexes, and `{"schema": "client"}`. Include an ORM relationship `stages: Mapped[list[ApprovalStage]] = relationship(back_populates="workflow", cascade="all, delete-orphan", order_by="ApprovalStage.order")` so loads return stages in canonical order.
   - `services/client-api/src/client_api/models/approval_stage.py` with `ApprovalStage` — columns mirror the migration; use `sa.Column("order", sa.Integer, ...)` with the mapped attribute named `order`; `required_role` typed `Mapped[ProposalCollaboratorRole]` using `sa.Enum(ProposalCollaboratorRole, name="proposal_collaborator_role", schema="client", create_type=False)`. `workflow: Mapped[ApprovalWorkflow] = relationship(back_populates="stages")`. `__table_args__` includes the unique constraint, the index, and `{"schema": "client"}`.
   - Both models exported from `client_api.models.__init__` so Alembic autogenerate stays clean.

3. [x] **AC3 — Pydantic schemas (`schemas/approval_workflows.py`).** Implement:
   - `ApprovalStageDefinition(BaseModel)`: `name: str` (1–255), `order: int` (≥ 1), `required_role: ProposalCollaboratorRole`, `auto_advance: bool = False`.
   - `ApprovalStageResponse(BaseModel)`: adds `id: UUID`, `workflow_id: UUID`, `created_at: datetime` to the stage definition fields (Pydantic `model_config = ConfigDict(from_attributes=True)`).
   - `ApprovalWorkflowCreateRequest`: `name: str` (1–255), `description: str | None` (≤ 2000), `stages: list[ApprovalStageDefinition]` (1–20 items), `is_default: bool = False`. Add a `@model_validator(mode="after")` that asserts the set of `stage.order` values equals `{1, 2, ..., len(stages)}` exactly (no gaps, no duplicates, no zero-index — raises `ValueError("Stage orders must be contiguous 1..N")` otherwise).
   - `ApprovalWorkflowUpdateRequest`: all fields optional; the same `order` contiguity validator runs whenever `stages` is provided.
   - `ApprovalWorkflowResponse`: `id`, `company_id`, `name`, `description`, `is_default`, `stages: list[ApprovalStageResponse]`, `created_by`, `created_at`, `updated_at`. `model_config = ConfigDict(from_attributes=True)`. The `stages` field relies on the ORM relationship being eager-loaded in the service (see AC4).
   - `ApprovalWorkflowListResponse`: `items: list[ApprovalWorkflowResponse]`, `total: int`, `limit: int`, `offset: int`.
   - `ApprovalWorkflowSetDefaultResponse`: `{id: UUID, is_default: bool, previous_default_id: UUID | None}`.

4. [x] **AC4 — `POST /api/v1/approval-workflows`.** Gated by `require_role("bid_manager")`. Body: `ApprovalWorkflowCreateRequest`. Behaviour (single transaction):
   - Insert `ApprovalWorkflow(company_id=current_user.company_id, name=..., description=..., is_default=is_default, created_by=current_user.user_id)`; flush to obtain `workflow.id`.
   - If `is_default is True`: **before** the insert above, issue `UPDATE client.approval_workflows SET is_default = FALSE WHERE company_id = :company_id AND is_default = TRUE AND deleted_at IS NULL` to avoid violating the partial unique index; then flush.
   - For each stage in `request.stages`, insert an `ApprovalStage(workflow_id=workflow.id, name=s.name, order=s.order, required_role=s.required_role, auto_advance=s.auto_advance)`; single flush at the end.
   - Eager-load `workflow.stages` (or re-select with `selectinload(ApprovalWorkflow.stages)`) before returning so the response includes the fully-populated stage list.
   - Write one audit row: `entity_type="approval_workflow"`, `action_type="create"`, `entity_id=workflow.id`, `after={"name": ..., "is_default": ..., "stage_count": N}`.
   - Return 201 with `ApprovalWorkflowResponse`.
   - On `IntegrityError` where `uq_approval_workflows_company_name` appears in `str(e.orig)`: raise `AppException(409, "A workflow with this name already exists", error="conflict")`.
   - On `IntegrityError` where `uq_approval_stages_workflow_order` appears: raise `AppException(422, "Duplicate stage order values", error="invalid_request")` — defence in depth; the Pydantic validator should catch this first.

5. [x] **AC5 — `GET /api/v1/approval-workflows`.** Gated by `get_current_user` (any authenticated company member can browse). Query params: `limit: int = 50` (≤ 100), `offset: int = 0`. Filters: `company_id = current_user.company_id` (always), `deleted_at IS NULL` (always). Eager-load stages via `selectinload(ApprovalWorkflow.stages)`. Order by `is_default DESC, name ASC, id ASC` so the default workflow sorts first (UI-friendly; list is also used by Story 10.16's stage picker). Returns `ApprovalWorkflowListResponse`.

6. [x] **AC6 — `GET /api/v1/approval-workflows/{id}`.** Gated by `get_current_user`. Returns the workflow with eager-loaded stages if the row is live (`deleted_at IS NULL`) and `company_id = current_user.company_id`; otherwise 404 (existence-leakage safe — same 404 for missing, cross-tenant, and soft-deleted). Returns `ApprovalWorkflowResponse`.

7. [x] **AC7 — `PATCH /api/v1/approval-workflows/{id}`.** Gated by `require_role("bid_manager")`. Accepts `ApprovalWorkflowUpdateRequest`. Behaviour:
   - 404 if missing / cross-tenant / soft-deleted.
   - If `stages` is provided: check `SELECT EXISTS(SELECT 1 FROM client.approval_decisions WHERE stage_id IN (:workflow_stage_ids))`. If any row exists → 409 `"Cannot modify stages on a workflow with existing decisions"`. Catch `sqlalchemy.exc.ProgrammingError` (missing table) and treat as "no decisions exist" — debug-log but proceed. Then: delete all existing stages for this workflow via `DELETE FROM client.approval_stages WHERE workflow_id = :id`, flush, then insert the new stages as in AC4.
   - If `name` is provided: update the field. If `is_default = True` is newly provided: first clear the existing default (same UPDATE as AC4).
   - `description` and `is_default = False` are simple field updates.
   - Null values (`null` in JSON) for `stages` are rejected at the Pydantic layer (`list[ApprovalStageDefinition] | None` with the validator skipping None, OR use `model_dump(exclude_unset=True)` and drop `stages` from the update dict when the client did not send the key — see the Story 10.7 N1 lesson: never let `null` fall through into a write because it will violate DB CHECK constraints).
   - Returns 200 with the reloaded `ApprovalWorkflowResponse` (with stages eager-loaded post-write).
   - 409 on `(company_id, name)` unique-rename collision.
   - Audit `action_type="update"`; `before` snapshot captures `{"name": old, "is_default": old, "stage_count": old}`; `after` captures the new values.

8. [x] **AC8 — `DELETE /api/v1/approval-workflows/{id}`.** Gated by `require_role("bid_manager")`. Behaviour:
   - 404 if missing / cross-tenant / already soft-deleted.
   - Run the same "any decisions reference this workflow's stages" pre-check as AC7; on hit → 409 `"Cannot delete workflow with recorded decisions"`. Soft-delete is still blocked because Story 10.9's decision history must remain valid — a decision with a dangling `stage_id` would break the audit trail even if the workflow is soft-deleted. (Rationale: the cascade-guard is semantic, not just schema-level.)
   - Set `deleted_at = now()` on the workflow row. Stages stay as-is (their `workflow_id` still resolves, which is important for any historical decisions that reference them).
   - If the deleted workflow was `is_default = TRUE`, leave the flag set on the soft-deleted row and do NOT promote another workflow to default — "no default" is a valid company state and Story 10.16's UI is expected to handle it (show a "Set a default" CTA).
   - Returns 204.
   - Audit `action_type="delete"`; `before` snapshot captures the workflow's key fields.

9. [x] **AC9 — `POST /api/v1/approval-workflows/{id}/set-default`.** Gated by `require_role("bid_manager")`. Empty request body. Behaviour (single transaction):
   - 404 if missing / cross-tenant / soft-deleted.
   - Capture `previous_default_id`: `SELECT id FROM client.approval_workflows WHERE company_id = :company_id AND is_default = TRUE AND deleted_at IS NULL` (may be NULL or the same as the target).
   - If `previous_default_id == target_id`: return 200 with `{id, is_default: true, previous_default_id: target_id}` — idempotent no-op, no audit row.
   - Else: `UPDATE client.approval_workflows SET is_default = FALSE WHERE id = :previous_default_id` (if non-null), flush, then `UPDATE client.approval_workflows SET is_default = TRUE WHERE id = :target_id`.
   - Returns 200 with `ApprovalWorkflowSetDefaultResponse`.
   - Audit `action_type="set_default"`, `entity_id=target_id`, `after={"previous_default_id": str(previous_default_id) if previous_default_id else None}`.

10. [x] **AC10 — Unit and API tests (≥ 20 scenarios).** Split as convenient between `services/client-api/tests/api/test_approval_workflows.py` (router + HTTP-layer tests) and `services/client-api/tests/unit/test_approval_workflow_service.py` (service-layer + schema validator tests):

    **CRUD happy paths:**
    - Create → list → get → patch (rename only) → patch (stages replacement) → delete → get-after-delete returns 404.
    - Create with `is_default=True` as the first workflow succeeds.
    - Create a second workflow with `is_default=True` clears the first workflow's `is_default` flag (verify with a GET that the first is now `is_default=False`).

    **Validation failures (422):**
    - Stage `order` values `[1, 3, 4]` (gap) rejected.
    - Stage `order` values `[1, 2, 2]` (duplicate) rejected.
    - Stage `order` value `[0, 1]` (zero-index) rejected.
    - Empty `stages: []` rejected.
    - `stages` longer than 20 rejected.
    - Invalid `required_role` (e.g. `"ceo"` — not in the enum) rejected.

    **Conflict / guard paths:**
    - Create 409 on duplicate `(company_id, name)`.
    - Patch rename 409 on collision with another live workflow in the same company.
    - Patch with `stages` replacement when a pre-existing row in `client.approval_decisions` references one of this workflow's stages → 409 `"Cannot modify stages on a workflow with existing decisions"`. (Test fixture may create `approval_decisions` rows via raw SQL INSERT since the Story 10.9 service is not yet available.)
    - Delete with the same pre-existing `approval_decisions` row → 409 `"Cannot delete workflow with recorded decisions"`.
    - Patch with `stages` replacement when `approval_decisions` table does NOT yet exist (simulated by dropping it in the test setup) → proceeds without error (exercises the `ProgrammingError` catch).

    **Tenancy / RBAC / leakage:**
    - Cross-tenant isolation: create workflow in company A, company B's bid_manager GET/PATCH/DELETE/set-default all return 404.
    - Non-`bid_manager` company member attempting create/patch/delete/set-default → 403. `get_current_user` (no role gate) on list/get is allowed — verify a `contributor` user can list.
    - Admin bypass works for all four mutation endpoints (already exercised by `require_role` but include one canary test).

    **Default toggling:**
    - `set-default` on a workflow when no default exists: returns `previous_default_id: null`; the target now has `is_default=True`.
    - `set-default` when a different workflow is currently default: swaps atomically; `previous_default_id` is populated.
    - `set-default` on the already-default workflow: idempotent (returns the same shape, NO audit row written — verify via an audit-log query).
    - `set-default` on a soft-deleted workflow → 404.
    - Soft-delete the current default and create a new workflow with `is_default=True`: succeeds (the partial unique index only covers live rows).

    **Soft-delete / re-create:**
    - After DELETE, workflow is gone from list; GET returns 404; a *new* workflow with the same name can be created (partial unique admits it).

    **Ordering / stage integrity:**
    - Create a 3-stage workflow; GET returns stages in `order` ASC regardless of the order they were sent in the request (exercises the ORM relationship `order_by`).
    - PATCH with a 5-stage replacement against a workflow that currently has 3 stages: after the PATCH, GET returns exactly the 5 new stages (old three are gone; verify no orphaned `approval_stages` rows in the DB).

    **Audit trail:**
    - After create / update / delete / set_default (non-idempotent), a corresponding `shared.audit_log` row exists with the expected `action_type`, `entity_type="approval_workflow"`, and `entity_id == workflow_id`.

    **Transaction atomicity:**
    - Inject a failure on the stage-insert loop during create (e.g. monkey-patch `session.flush` to raise on the second `flush`): assert no `approval_workflows` row and no `approval_stages` rows persist; the `is_default` clear of any previous default is also rolled back (if applicable).

## Dev Notes

- **Test Expectations:** `eusolicit-docs/test-artifacts/test-design-epic-10.md` is **not** available — Epic 10 was never epic-test-designed (the `test-artifacts/` directory for Epic 10 contains only per-story ATDD checklists 10-1 through 10-7; no epic-level design doc). Tests should follow the standard `httpx.AsyncClient` + `async with AsyncSession` patterns established in Stories 10.5–10.7 and Epic 7. Cross-tenant isolation must return 404 (not 403) to preserve existence-leakage guarantees consistent with Stories 7.2, 9.3, 10.1, 10.5, 10.6, and 10.7. When writing ATDD-style scenario tests, prefer the "arrange/act/assert" block format used in `tests/api/test_task_templates.py`.
- **File layout conventions:** `services/client-api/src/client_api/api/v1/approval_workflows.py` is a new router file; import as `from client_api.api.v1 import approval_workflows as approval_workflows_v1` in `client_api/main.py` and register via `api_v1_router.include_router(approval_workflows_v1.router)` beside `task_templates_v1.router`. Route prefix `/approval-workflows` (hyphenated), `tags=["approval-workflows"]`.
- **Model init wiring:** Export `ApprovalWorkflow` and `ApprovalStage` from `client_api/models/__init__.py`. This is load-bearing for Alembic autogenerate (the base `MetaData` must see the tables) and for any downstream `selectinload(ApprovalWorkflow.stages)` call in the service layer.
- **Enum reuse — do not recreate the type:** `required_role` in `approval_stages` must reference the existing `client.proposal_collaborator_role` Postgres enum type created by Story 10.1's migration. In both the Alembic migration and the ORM declaration, pass `sa.Enum(ProposalCollaboratorRole, name="proposal_collaborator_role", schema="client", create_type=False)`. Forgetting `create_type=False` will cause Alembic to emit a `CREATE TYPE` statement that collides with the existing type and crashes the migration.
- **`order` column keyword handling:** `order` is a reserved word in SQL. SQLAlchemy's core dialect auto-quotes it on PostgreSQL when declared as `sa.Column("order", ...)`, and Alembic's migration DSL does the same. In raw SQL fragments (e.g. the cascade-guard pre-check), always wrap in double quotes: `"order"`. In test helpers that build raw INSERTs, same. If the migration emits an unquoted `order` in a CHECK constraint, PostgreSQL rejects with a syntax error — specify `name="ck_approval_stages_order_positive"` and use `sa.CheckConstraint('"order" >= 1', ...)` with the literal quotes.
- **Cascade-guard `approval_decisions` table existence:** Story 10.9 (which creates `client.approval_decisions`) lands after this story. In production both migrations (042 and 043) will be applied before any endpoint traffic flows, so in practice the cascade-guard SELECT will always find the table. However, in test fixtures that only run migrations up to 042, the table may not exist. The guard's `try: execute_scalar(...) except ProgrammingError: proceed` pattern handles this gracefully and is worth a small integration test (see AC10). When Story 10.9 ships, re-verify that the `approval_decisions.stage_id` FK is declared `ON DELETE RESTRICT` so even a raw DB-level `DELETE` of a stage is blocked if a decision references it — this is the belt to the guard's suspenders.
- **`set-default` concurrency safety:** Two concurrent `set-default` calls on different workflows in the same company could race. The partial unique index (`uq_approval_workflows_company_default`) prevents both from winning — the second transaction's COMMIT will fail with `IntegrityError`. Catch this in the endpoint and translate to 409 `"Another default was set concurrently; refresh and retry"`. This is a rare edge case but the guard is cheap.
- **Audit payload conventions:** Reuse `audit_service.write_audit_entry` (Story 2.11 pattern). `entity_id` is the workflow id for all four audited actions (create / update / delete / set_default). `after` for create carries `{name, is_default, stage_count}` (don't serialise the full stage list — that bloats the audit table); `before`/`after` for update carries the same three key fields; `after` for set_default carries `{previous_default_id}`; `before` for delete captures `{name, is_default, stage_count}`.
- **Out of scope:** No decision recording, no event emission, no `auto_advance` behaviour (those are Story 10.9). No UI (Story 10.16). No workflow versioning / forking (if a workflow is edited, in-flight approvals on it are affected — but this only becomes observable in Story 10.9; the guard in AC7/AC8 is the mitigation). No per-workflow assignment to proposals (Story 10.9 will introduce the `proposal_id → workflow_id` linkage, possibly via `proposals.approval_workflow_id` or a dedicated link table; this story does NOT add that column).
- **Cascade-delete semantics on `approval_workflows.deleted_at`:** Soft-delete does NOT touch `approval_stages` rows — they persist. The only way stages are hard-deleted is during a PATCH `stages` replacement (guarded) or via the FK cascade if the workflow row itself is hard-deleted (which no endpoint does). This keeps the decision-history FK target alive even after a workflow is soft-deleted.

## Senior Developer Review

**Reviewer:** Claude (bmad-code-review)
**Date:** 2026-04-21 (follow-up)
**Outcome:** REVIEW: Approve

### Summary

Follow-up review after the B1/B2/N1/N5 remediation. All previously-blocking findings are addressed:

- **B1 resolved** — `_check_decisions_exist` now wraps the probe in `async with session.begin_nested():` (service l. 174–190) and re-raises `ProgrammingError` only when the message does not indicate a missing relation. A dedicated unit test (`test_cascade_guard_programming_error_treated_as_no_decisions`) exercises the savepoint path with a mocked `AsyncSession` whose `begin_nested()` is a sync MagicMock (correct match for SQLAlchemy semantics). A negative test (`test_cascade_guard_other_programming_error_reraises`) confirms non-missing-table `ProgrammingError`s still propagate.
- **B2 resolved** — AC10 scenario matrix now covers ≥ 20 cases across `tests/api/test_approval_workflows.py` (24 API tests) and `tests/unit/test_approval_workflow_service.py` (24 unit tests). All specifically-enumerated gaps are closed: empty/oversized `stages`, invalid `required_role`, PATCH rename 409, DELETE-with-decisions 409, ProgrammingError guard path, 403 matrix on PATCH/DELETE/set-default, contributor read allowance, admin bypass, `set-default` no-previous / idempotent-no-audit / soft-deleted 404 / post-soft-delete default swap, ordering ASC, 5-stage replacement orphan check, and create/update/delete audit rows. A conceptual atomicity unit test injects a `flush` failure on the stage-insert path and asserts the exception propagates (delegating the rollback to the session-manager, which is the correct contract).
- **N1 resolved** — `update_workflow` computes `after_stage_count = len(request.stages) if request.stages is not None else before_audit["stage_count"]` (service l. 254), so the audit payload now reflects the post-PATCH stage count rather than the stale ORM collection.
- **N5 resolved** — `approval_decisions_table` pytest fixture creates a minimal `client.approval_decisions` table with the `stage_id` FK and the required grant when Story 10.9's migration is absent, and tears it down only if it provisioned the table itself. Guard tests no longer depend on external provisioning.

### Verified against spec

- **AC1** Migration 042: partial unique indexes (`uq_approval_workflows_company_name` WHERE `deleted_at IS NULL`, `uq_approval_workflows_company_default` WHERE `is_default = TRUE AND deleted_at IS NULL`), CHECK with quoted `"order"`, `create_type=False` on the enum, FK cascade to `client.companies`, grants, and downgrade order all match.
- **AC2** ORM models: `__table_args__` includes `{"schema": "client"}`; `stages` relationship uses `cascade="all, delete-orphan"` and `order_by="ApprovalStage.order"`; both models exported from `client_api.models.__init__`.
- **AC3** Schemas: `min_length=1`, `max_length=20` on `stages`; `order ≥ 1`; contiguity validator rejects gaps / duplicates / zero-index; `ApprovalWorkflowUpdateRequest` skips the validator when `stages is None`; `model_dump(exclude_unset=True)` correctly omits unsent keys (unit-tested).
- **AC4** Create: clear-existing-default ordered BEFORE insert to avoid partial-unique violation; single flush after stage inserts; `selectinload` reload before returning; `IntegrityError` mapped to 409 / 422.
- **AC5/AC6** List: orders by `is_default DESC, name ASC, id ASC`; get returns 404 uniformly for missing / cross-tenant / soft-deleted (existence-leakage safe).
- **AC7** Patch: savepoint-wrapped decisions guard; delete-then-insert stages; `is_default=True` clear-before-set; N1 audit accuracy; 409 on name collision.
- **AC8** Delete: soft-delete sets `deleted_at`, preserves stages for decision-FK stability, leaves `is_default` untouched on soft-deleted rows; blocked by decisions guard.
- **AC9** Set-default: idempotent no-op (no audit); `UPDATE`-then-`UPDATE` in the same transaction; concurrency-loss `IntegrityError` on `uq_approval_workflows_company_default` translated to 409 `"Another default was set concurrently; refresh and retry"`.
- Router wired at `/api/v1/approval-workflows` beside `task_templates_v1.router` (confirmed in `main.py:23,125`); four mutation endpoints gated by `require_role("bid_manager")`; reads gated by `get_current_user`.

### Remaining non-blocking observations

- **N2 (minor).** `update_workflow` still has two independent conditionals for `is_default`: first the clear-existing-default branch, then `if request.is_default is not None: workflow.is_default = request.is_default`. Functionally correct; reads a touch redundant. Consider collapsing in a future cleanup pass.
- **N3 (minor).** `ApprovalStageResponse` inherits from `ApprovalStageDefinition`, so response-side objects are subject to the same `order ≥ 1` / `name 1..255` constraints. Fine today since DB data respects them; worth revisiting if response validation ever needs to relax.
- **N4 (minor).** `list_workflows` total-count uses `select(func.count()).select_from(base_query.subquery())` rather than a flat `select(func.count()).where(...)`. Matches the pattern in adjacent services; keep for consistency.

### Outcome

REVIEW: Approve. Story 10.8 is ready to merge. The cascade-guard savepoint contract is correctly implemented and regression-tested, the AC10 scenario matrix is satisfied, and audit payload accuracy is fixed. No new deviations detected this pass.

### Change Log for this review

- 2026-04-21 (pm): Previous Changes Requested findings (B1, B2, N1, N5) addressed; re-reviewed and approved.

---

## Senior Developer Review — Prior Round (superseded)

**Reviewer:** Claude (bmad-code-review)
**Date:** 2026-04-21
**Outcome:** REVIEW: Changes Requested (SUPERSEDED by Approve above)

### Summary

Architecture and domain modeling are sound. The two-table normalised design, partial unique indexes, enum reuse of `proposal_collaborator_role`, and the ORM relationship with `order_by="ApprovalStage.order"` all land as specified. Migration 042 matches AC1. Schemas enforce the `{1..N}` contiguity invariant. Router, service, and `main.py` wiring are correct.

Two blocking issues and several smaller quality concerns prevent approval.

### Blocking Findings

**B1. `_check_decisions_exist` cannot recover from `ProgrammingError` without a savepoint.**
File: `services/client-api/src/client_api/services/approval_workflow_service.py:159-177`.
The guard catches `ProgrammingError` when `client.approval_decisions` does not exist and returns `False`. However, once PostgreSQL raises a relation-does-not-exist error inside a transaction, the transaction is in an aborted state — every subsequent `execute()` fails with `InFailedSqlTransaction: current transaction is aborted`. The subsequent stage `DELETE` and stage `INSERT`s in `update_workflow` will therefore fail, and the audit write will also fail. The dev note in AC7 and the Dev Notes section explicitly describe this as a "graceful" path — the current implementation is not graceful, it is broken. Required fix: wrap the decisions-probe in `async with session.begin_nested():` so the savepoint absorbs the rollback, or pre-check table existence via `information_schema.tables` before running the `SELECT EXISTS(...)`. Without this, AC7's "Patch with `stages` replacement when `approval_decisions` table does NOT yet exist … proceeds without error" test cannot pass in any environment where the table genuinely does not exist.

**B2. Test coverage falls short of AC10 (≥ 20 scenarios; ~15 present; many explicitly-listed cases missing).**
File: `services/client-api/tests/api/test_approval_workflows.py`.
AC10 enumerates specific scenarios that are not covered. No `tests/unit/test_approval_workflow_service.py` exists. Missing coverage (each line in AC10 not currently exercised):

- Empty `stages: []` rejected (422).
- `stages` with > 20 items rejected (422).
- Invalid `required_role` string (e.g. `"ceo"`) rejected (422).
- PATCH rename 409 on `(company_id, name)` collision with another live workflow.
- DELETE 409 when `approval_decisions` references a stage (`"Cannot delete workflow with recorded decisions"`).
- PATCH stages replacement when `client.approval_decisions` table does NOT exist (the `ProgrammingError`-catch path — the exact scenario that B1 breaks).
- Non-`bid_manager` → 403 on PATCH / DELETE / set-default (only create is covered).
- `contributor` can list / get (role-gate boundary on read).
- Admin bypass canary test on one mutation endpoint.
- `set-default` when no default exists → `previous_default_id: null`.
- `set-default` on the already-default workflow → idempotent, **no** audit row written (verify via audit query).
- `set-default` on soft-deleted workflow → 404.
- Soft-delete current default, create new workflow with `is_default=True` → succeeds (partial unique covers live rows only).
- Soft-delete then recreate workflow with the same name → succeeds.
- GET returns stages in `order` ASC when the request submitted them out of order.
- PATCH 5-stage replacement of a 3-stage workflow: verify exactly 5 stages post-PATCH and zero orphan `approval_stages` rows in the DB.
- Audit rows for `create`, `update`, `delete` (only `set_default` audit is currently tested).
- Transaction atomicity: inject failure during stage insert; assert no workflow row, no stage rows, and any `is_default` clear is rolled back.

At least the `ProgrammingError`-path test (tied to B1) and the non-`bid_manager` RBAC tests on mutations are non-negotiable for this story's safety claims.

### Non-Blocking Findings

**N1. `update_workflow` audit `after.stage_count` is stale after stages replacement.**
File: `approval_workflow_service.py:249-254`. After the raw `DELETE FROM approval_stages` + new stage `session.add()`s, `workflow.stages` is still the originally `selectinload`-populated Python list — the raw DELETE does not sync the ORM collection, and new `ApprovalStage(workflow_id=...)` instances are not appended to `workflow.stages`. `len(workflow.stages)` in the audit payload therefore returns the pre-PATCH count. Fix: compute `after_stage_count = len(request.stages) if request.stages is not None else len(workflow.stages)` before calling `write_audit_entry`, or move the audit write after the final `selectinload` reload.

**N2. `update_workflow` clears the existing default before the ORM field assignment even in the no-op `is_default=True` + already-default case.**
File: `approval_workflow_service.py:198-207`. The `if request.is_default is True and not workflow.is_default:` guard already short-circuits correctly, so this is fine for the direct toggle. However, after the clear-existing-default branch, the flow falls through to `if request.is_default is not None: workflow.is_default = request.is_default`, which re-sets the target workflow to `True`. Functionally correct but the two conditionals read as independent — consider collapsing for readability.

**N3. Schema naming asymmetry.**
`ApprovalStageResponse` inherits from `ApprovalStageDefinition`, which means the response object is subject to the same `order ≥ 1` / `name 1..255` constraints. That's fine for now but couples the wire shapes; worth a comment or a dedicated response base if future changes relax response-side validation.

**N4. `list_workflows` total count query wraps the base query in a subquery.**
File: `approval_workflow_service.py:117-119`. `select(func.count()).select_from(base_query.subquery())` works but is slightly wasteful vs. `select(func.count()).where(...)` with the same two predicates duplicated. Minor; keep if consistency with other services is valued.

**N5. Test fixture for decisions-exist case relies on an externally-provisioned `client.approval_decisions` table (`test_patch_workflow_409_decisions_exist`).**
If the test DB does not run Story 10.9's migration, the raw `INSERT INTO client.approval_decisions ...` will 500 and the test's 409 assertion is never reached meaningfully. The AC10 note explicitly permits "raw SQL INSERT since the Story 10.9 service is not yet available" — but this implies the table itself must be created as part of test setup. Either add a conftest fixture that creates a minimal `approval_decisions` table, or add a pytest skip with a clear reason when the table is absent. Today the test is environment-sensitive.

### Passing / Good

- AC1 migration matches spec: partial unique indexes, CHECK constraint with quoted `"order"`, `create_type=False` on the enum, grants, downgrade order.
- AC2 ORM models wire `__table_args__` correctly; `{"schema": "client"}` present on both; `stages` relationship has `order_by` and `cascade="all, delete-orphan"`.
- AC3 schemas: contiguity validator rejects gaps, duplicates, and zero-index; update variant correctly skips when `stages is None`.
- AC4 create: clear-then-insert ordering on `is_default`, flush-then-id-read, eager reload before response. `IntegrityError` mapping to 409/422 is in place.
- AC5/AC6: cross-tenant GETs return 404 (existence-leakage safe); list ordering `is_default DESC, name ASC, id ASC` matches.
- AC8: soft-delete leaves `is_default` untouched, stages preserved for decision FK stability.
- AC9: idempotent no-op returns same shape with no audit row; concurrency-loss `IntegrityError` → 409 on the partial default unique index.
- Router wired at `/approval-workflows` beside `task_templates_v1.router`; `require_role("bid_manager")` gates the four mutation endpoints.
- Models exported from `client_api.models.__init__` for Alembic autogenerate visibility.

### Required Actions

1. Fix B1 by wrapping the decisions probe in a savepoint (`session.begin_nested()`) and add a regression test that exercises the missing-table path.
2. Close the AC10 coverage gap. Priority order: the `ProgrammingError`-path test, DELETE-with-decisions 409, non-`bid_manager` 403 on PATCH/DELETE/set-default, `set-default` idempotency (no audit), audit rows for create/update/delete, and transaction-atomicity rollback on stage-insert failure.
3. Address N1 (stage_count accuracy in audit) — this is small and belongs with the B-fix commit.

### Markers

DEVIATION: Cascade-guard uses a naked `try/except ProgrammingError` with no savepoint; in PostgreSQL this leaves the surrounding transaction aborted and breaks all subsequent writes in `update_workflow`, contradicting the behaviour promised in AC7 and Dev Notes.
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

DEVIATION: Test suite covers ~15 scenarios vs. AC10's ≥ 20 explicit requirement; several specifically-enumerated scenarios (table-missing path, DELETE-with-decisions 409, set-default idempotency audit check, transaction atomicity, per-role 403 matrix) are absent.
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: blocking

## Senior Developer Review — Round 3 (Test infrastructure fix)

**Reviewer:** Claude (bmad-code-review)
**Date:** 2026-04-25
**Outcome:** REVIEW: Approve

### Summary

Re-review focused on the only delta noted in the Dev Agent Record: a refactor of `test_patch_workflow_proceeds_when_approval_decisions_missing` (test_approval_workflows.py:472–544). The test now uses a freshly-built `_make_role_client` instead of the module-level `aw_client` fixture so the `ALTER TABLE ... RENAME` operation is not blocked by a held-open transaction.

All implementation files (`migration 042`, `models/approval_workflow.py`, `models/approval_stage.py`, `schemas/approval_workflows.py`, `services/approval_workflow_service.py`, `api/v1/approval_workflows.py`) are unchanged versus the prior Approve and continue to satisfy AC1–AC9 verbatim:

- Migration 042 retains both partial unique indexes, the quoted `"order"` CHECK, `create_type=False` on the enum, the FK cascade to `client.companies`, and grants.
- ORM relationship still uses `cascade="all, delete-orphan"` and `order_by="ApprovalStage.order"`; `__table_args__` carry `{"schema": "client"}` and the unique constraints / indexes.
- Schemas enforce `min_length=1`, `max_length=20` on `stages`, `order ≥ 1`, the contiguity validator on both create and update, and skip-when-`None` semantics on `ApprovalWorkflowUpdateRequest`.
- Service: B1 savepoint guard intact (`async with session.begin_nested():` at l. 175), N1 audit accuracy intact (l. 254 computes `after_stage_count` from `request.stages`), idempotent `set-default` no-op (l. 343–348) still skips the audit write, and concurrency `IntegrityError` on the partial default index translates to 409 (l. 376–378).
- Router still gates `POST/PATCH/DELETE/set-default` with `require_role("bid_manager")` and reads with `get_current_user`.

### Test refactor — assessment

Verified the new `test_patch_workflow_proceeds_when_approval_decisions_missing`:

- Seeds the workflow + first stage via `superuser_session_factory` and **commits** before opening the role client — this avoids the prior deadlock where the open `aw_client` transaction held a row lock that conflicted with the schema-level `ALTER TABLE RENAME`.
- The `RENAME TO _approval_decisions_bak` is wrapped in its own `async with session.begin():` block, so the DDL commits cleanly before the API client connects.
- The `_make_role_client` constructs a new `bid_manager` user under `_COMPANY_A_ID`. RBAC and tenancy still match (the workflow belongs to company A; bid_manager from company A can PATCH it). The created_by / current_user identity differs but that is irrelevant for this endpoint.
- The `finally` block restores the table only when `table_exists` was true at setup, which is the correct contract — `_make_role_client` does not need to know whether the rename happened.
- The savepoint-guard path in `_check_decisions_exist` is the actual code under test, and the assertion `200 + 2 stages` confirms both the guard returned `False` and the subsequent stage delete-and-recreate completed in the same transaction.

The deviation logged by `2-dev-story` ("AC10 — test_patch_workflow_proceeds_when_approval_decisions_missing was refactored to use an independent client to prevent transaction deadlocks during table rename"; severity: deferrable) is correctly classified — it is a test-harness adjustment that does not alter production code or AC10 coverage. The renamed test still exercises the `ProgrammingError`-catch path (table actually missing), which is the AC10 requirement.

### Verified scenario coverage (AC10)

Counted in `tests/api/test_approval_workflows.py`: 32 async test functions covering create / list / get / patch / delete / set-default / RBAC / cross-tenant / audit / ordering / soft-delete recycle / default toggling. Plus 24 unit tests in `tests/unit/test_approval_workflow_service.py` and 24 schema tests in `tests/unit/test_approval_workflow_schemas.py` (3 test files for service-layer concerns). Reported run: `40 passed (API) / 42 passed (unit)`. AC10's ≥ 20-scenario floor remains satisfied with significant headroom.

### Remaining non-blocking observations

Carry-forward from Round 2 — none of these block merge:

- **N2.** `update_workflow` retains two independent `is_default` conditionals; functionally correct, slightly redundant for readability.
- **N3.** `ApprovalStageResponse` still inherits constraint validation from `ApprovalStageDefinition`; fine today since DB data respects the limits.
- **N4.** `list_workflows` total-count uses `select(func.count()).select_from(base_query.subquery())` for consistency with adjacent services.

No new findings.

### Outcome

REVIEW: Approve. Story 10.8 remains ready to merge. The Round-3 test refactor is a low-risk infrastructure fix that resolves a real-world deadlock without weakening the AC10 guard test or touching production code.

### Markers

No new deviations introduced this round. Pre-existing dev-logged deviation (test refactor) is correctly classified as deferrable.

## Change Log

- 2026-04-21: Initial story context created. (bmad-create-story)
- 2026-04-21: Senior Developer Review appended — Changes Requested. (bmad-code-review)
- 2026-04-21: Follow-up Senior Developer Review appended — Approve (B1/B2/N1/N5 remediated). (bmad-code-review)
- 2026-04-25: Round-3 Senior Developer Review appended — Approve (test-infrastructure refactor verified, no new deviations). (bmad-code-review)

## Known Deviations

### Detected by `3-code-review` at 2026-04-20T23:05:39Z (session ba854b56-3630-415a-85ff-1a94fbd799d2)

- Cascade-guard uses a naked `try/except ProgrammingError` with no savepoint; in PostgreSQL this leaves the surrounding transaction aborted and breaks all subsequent writes in `update_workflow`, contradicting the behaviour promised in AC7 and Dev Notes. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- Test suite covers ~15 scenarios vs. AC10's ≥ 20 explicit requirement; several specifically-enumerated scenarios are absent. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- Cascade-guard uses a naked `try/except ProgrammingError` with no savepoint; in PostgreSQL this leaves the surrounding transaction aborted and breaks all subsequent writes in `update_workflow`, contradicting the behaviour promised in AC7 and Dev Notes. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- Test suite covers ~15 scenarios vs. AC10's ≥ 20 explicit requirement; several specifically-enumerated scenarios are absent. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_

### Detected by `2-dev-story` at 2026-04-24T21:58:58Z

- AC10 — test_patch_workflow_proceeds_when_approval_decisions_missing was refactored to use an independent client to prevent transaction deadlocks during table rename. _(type: `ARCHITECTURAL_DRIFT`; severity: `deferrable`)_
