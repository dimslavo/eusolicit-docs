# Story 10.7: Task Templates CRUD & Application

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-7-task-templates-crud-application
- **Points:** 3
- **Type:** backend
- **Module:** Client API (`client_api.api.v1.task_templates` (new), `client_api.models.task_template` (new), `client_api.schemas.task_templates` (new), `client_api.services.task_template_service` (new), reuse of `client_api.services.task_service`, new Alembic migration `041_task_templates`)
- **Priority:** P1 (Epic 10 productivity layer — unblocks Story 10.15 Task Template Manager UI; reuses the Task CRUD (10.5) and DAG validation (10.6) primitives)
- **Depends On:** Story 10.5 (Task CRUD API — `client.tasks` table, `Task` ORM, `TaskStatus`/`TaskPriority` enums, `task_service.create_task`), Story 10.6 (`client.task_dependencies` table, `DependencyType` enum, `task_service` dependency insert + DAG validation helpers), Story 6.5 (`pipeline.opportunities` schema — `deadline` DateTime column, `opportunity_type` column), Story 2.10 (`require_role` company-role gate), Story 2.4 (`get_current_user`).

## Story

As a **bid_manager**,
I want **to define reusable task templates per company and apply them to an opportunity with one action**,
so that **my team does not have to hand-build the same 8–15-step preparation checklist for every new bid, and deadlines and prerequisite links are computed automatically from the opportunity's submission deadline.**

## Description

This story adds a reusable template layer on top of the task/dependency schema shipped in Stories 10.5 and 10.6. A `task_template` is a company-scoped, optionally opportunity-type-scoped (tender/grant/any) blueprint that stores an ordered list of stage definitions in a single JSONB column. Each stage definition carries the fields needed to materialise a concrete `client.tasks` row — `title`, `description`, `role`, `relative_days_before_deadline`, `dependency_indices` — where `role` is one of the five `ProposalCollaboratorRole` values (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only) used purely as a documentation hint on the resulting task (the task assignee itself remains unassigned at apply-time and is filled in later by the manager), and `dependency_indices` is a list of 0-based indices referencing earlier stages in the same template (forward references forbidden — must be strictly less than the current index — which guarantees by construction that the emitted `task_dependencies` rows form a DAG).

Five endpoints are exposed under `/api/v1/task-templates`: create, list, retrieve, patch, delete. A sixth endpoint, `POST /api/v1/task-templates/{id}/apply`, is the productivity payoff — it reads the target opportunity's `deadline`, iterates the template's stages once to insert `client.tasks` rows (capturing the new task UUIDs in an index-aligned array), then iterates a second time to insert `client.task_dependencies` rows with `dependency_type = 'finish_to_start'` using the index array to resolve `depends_on_task_id`. `due_date` for each task is computed as `deadline - interval 'N days'` where `N = relative_days_before_deadline`; if a stage's `relative_days_before_deadline` is larger than the number of days between `now()` and `deadline`, the resulting `due_date` lands in the past — this is **allowed** (the manager can surface overdue tasks in the kanban) but the response includes a `warnings` array listing the indices of such stages so the UI in Story 10.15 can show an "N tasks are already overdue" banner.

Four design decisions worth calling out:

**(a) `stages` is a single `JSONB NOT NULL` column, not a normalised `task_template_stages` table.** Stages are always read and written as a group — there is no query pattern that needs to filter or join on individual stages. Storing them as JSONB keeps the apply path a single two-statement transaction per template instead of an N+1 fan-out, and dependency_indices naturally survive serialisation without needing a surrogate key. A CHECK constraint (`jsonb_typeof(stages) = 'array' AND jsonb_array_length(stages) > 0 AND jsonb_array_length(stages) <= 50`) enforces shape at the DB level. Application-level Pydantic validation enforces the richer per-stage shape (title length, dependency_indices < current index, role ∈ enum).

**(b) Apply uses a single transaction with two passes over stages.** Pass 1: `INSERT INTO client.tasks ... RETURNING id` once per stage, appending each new id to a Python list `created_ids` in stage order. Pass 2: for each stage with non-empty `dependency_indices`, `INSERT INTO client.task_dependencies (task_id, depends_on_task_id, dependency_type) VALUES (created_ids[i], created_ids[j], 'finish_to_start')` for each `j` in the stage's `dependency_indices`. Both passes commit atomically; any failure rolls back the whole apply. We do NOT invoke the DAG cycle-detection from Story 10.6 on each insert because validation at create/patch time (AC4) already guarantees the index graph is a DAG — running DFS per edge at apply time would be O(E·V) wasted work.

**(c) `opportunity_type` filter on the template is advisory, not enforced.** The `GET /task-templates?opportunity_type=tender` query filters by `tt.opportunity_type IN ('tender', 'any')` (i.e. `any` templates always show up). Apply itself does NOT reject when the template's `opportunity_type` mismatches the target opportunity — the UI in Story 10.15 will surface a non-blocking warning. Rationale: managers sometimes legitimately re-use a tender template for a grant-like solicitation; hard blocking creates friction and forces duplicate templates.

**(d) Apply requires the caller to be a proposal collaborator (`bid_manager`) OR a company admin — not just any bid_manager in the company.** Crucially, the tasks emitted by apply carry `proposal_id = NULL` by default (templates are opportunity-scoped, not proposal-scoped), so we cannot reuse Story 10.2's `require_proposal_role`. Instead, apply uses the existing `require_role("bid_manager")` company-role gate from Story 2.10, with an additional runtime check that the target opportunity lives within `current_user.company_id` scope via `opportunity_scope_service.ensure_opportunity_visible` (the same helper used by Story 6.5's opportunity detail endpoint). Template CRUD uses `require_role("bid_manager")` for mutations and `get_current_user` for reads (any authenticated company member can view the company's templates).

A subtle scoping note: there is no `proposal_id` on the template or on the emitted tasks. Story 10.14's task kanban board lists tasks scoped by `opportunity_id`; Story 10.5's task CRUD has supported nullable `proposal_id` since day one precisely to accommodate the opportunity-only workflow that templates produce.

## Acceptance Criteria

1. [x] **AC1 — `client.task_templates` Alembic migration (`041_task_templates.py`).** Create a new migration with `revision = "041"`, `down_revision = "040"`. The table includes:
   - `id` UUID primary key, `server_default=sa.text("gen_random_uuid()")`.
   - `company_id` UUID NOT NULL, FK `client.companies.id` ON DELETE CASCADE.
   - `name` VARCHAR(255) NOT NULL.
   - `description` TEXT NULL.
   - `opportunity_type` VARCHAR(20) NOT NULL, CHECK `opportunity_type IN ('tender', 'grant', 'any')`, default `'any'`.
   - `stages` JSONB NOT NULL, CHECK `jsonb_typeof(stages) = 'array' AND jsonb_array_length(stages) BETWEEN 1 AND 50`.
   - `created_by` UUID NOT NULL (soft link to `client.users.id`).
   - `created_at`, `updated_at` TIMESTAMPTZ with `server_default=sa.func.now()`; `updated_at` has `onupdate=sa.func.now()`.
   - `deleted_at` TIMESTAMPTZ NULL (soft-delete).
   - Indexes: `ix_task_templates_company_id`, `ix_task_templates_opportunity_type`, `ix_task_templates_deleted_at`.
   - Unique constraint on `(company_id, name)` WHERE `deleted_at IS NULL` (partial unique index — one active template per (company, name) pair; soft-deleted rows do not block renames).
   - `downgrade()` drops the table and indexes.
   - Role grants: `GRANT SELECT, INSERT, UPDATE, DELETE ON client.task_templates TO orchestrator_client_api` (match the pattern in `040_task_dependencies.py`).

2. [x] **AC2 — `TaskTemplate` ORM model.** Add `services/client-api/src/client_api/models/task_template.py` with a `TaskTemplate` class mirroring the migration. Stages stored as `sa.JSON` (Postgres dialect → JSONB). Add `__table_args__` with all indexes, the partial unique constraint, and `{"schema": "client"}`. Export from `client_api.models.__init__`. No relationship back from `Task` (the template is a factory, not a parent — emitted tasks carry no template_id).

3. [x] **AC3 — Pydantic schemas (`schemas/task_templates.py`).** Implement:
   - `StageDefinition(BaseModel)`: `title: str` (1–500 chars), `description: str | None` (≤ 2000 chars), `role: ProposalCollaboratorRole`, `relative_days_before_deadline: int` (≥ 0 and ≤ 365), `dependency_indices: list[int] = []` (each value ≥ 0, strict `<` the stage's own index — enforced by `model_validator(mode="after")` on the *list* context).
   - `TaskTemplateCreateRequest`: `name: str` (1–255), `description: str | None` (≤ 2000), `opportunity_type: Literal["tender", "grant", "any"] = "any"`, `stages: list[StageDefinition]` (1–50, non-empty) with a root validator that re-runs the "dependency_indices strictly less than own index" check across the whole array and rejects duplicate indices within a single stage's `dependency_indices`.
   - `TaskTemplateUpdateRequest`: all optional; same validators.
   - `TaskTemplateResponse`: `id`, `company_id`, `name`, `description`, `opportunity_type`, `stages`, `created_by`, `created_at`, `updated_at`.
   - `TaskTemplateListResponse`: `items: list[TaskTemplateResponse]`, `total: int`, `limit: int`, `offset: int`.
   - `TaskTemplateApplyRequest`: `opportunity_id: UUID`.
   - `TaskTemplateApplyResponse`: `created_task_ids: list[UUID]` (in stage order), `created_dependency_ids: list[UUID]`, `warnings: list[str]` (e.g. `"Stage 2 'Draft technical section' due_date computed to 2025-12-01 is in the past"`, `"Template opportunity_type 'tender' does not match opportunity_type 'grant'"`).

4. [x] **AC4 — `POST /api/v1/task-templates`.** Gated by `require_role("bid_manager")`. Validates the request (Pydantic handles shape; service layer re-runs the "indices form a DAG" check as a defense-in-depth assertion before INSERT). Inserts with `company_id = current_user.company_id`, `created_by = current_user.user_id`. Returns 201 with `TaskTemplateResponse`. Rejects with 409 on the partial-unique `(company_id, name)` collision (catch `IntegrityError`, translate to `HTTPException(409, detail="A template with this name already exists")`). Writes an audit-trail entry with `entity_type="task_template"`, `action="create"`.

5. [x] **AC5 — `GET /api/v1/task-templates`.** Gated by `get_current_user` only (any company member can browse). Query params: `opportunity_type: Literal["tender", "grant", "any"] | None = None`, `limit: int = 50` (≤ 100), `offset: int = 0`. Filters: `company_id = current_user.company_id` (always), `deleted_at IS NULL` (always), and when `opportunity_type` is provided, `tt.opportunity_type IN (requested, 'any')` (wildcard match — `any` templates are universally visible). Returns `TaskTemplateListResponse` ordered by `created_at DESC, id ASC` for stable pagination.

6. [x] **AC6 — `GET /api/v1/task-templates/{id}`.** Gated by `get_current_user`. Returns the template if `id` resolves to a row where `company_id = current_user.company_id` and `deleted_at IS NULL`, else 404 (existence-leakage-safe — same 404 for missing, cross-tenant, and soft-deleted). Returns `TaskTemplateResponse`.

7. [x] **AC7 — `PATCH /api/v1/task-templates/{id}`.** Gated by `require_role("bid_manager")`. Accepts `TaskTemplateUpdateRequest`. Applies non-null fields via `model_dump(exclude_unset=True)`. If `stages` is provided, the full Pydantic + service-layer DAG validation re-runs on the new array (no merge — the client sends the whole replacement list). `updated_at` is refreshed by ORM `onupdate`. Returns 200 with the updated `TaskTemplateResponse`. 404 for missing/cross-tenant/soft-deleted. 409 for `(company_id, name)` collisions on rename. Audit `action="update"`.

8. [x] **AC8 — `DELETE /api/v1/task-templates/{id}`.** Gated by `require_role("bid_manager")`. Sets `deleted_at = now()` (soft-delete — never hard-deletes, so historical "this task was created from template X" attribution survives even after X is removed, though no FK enforces this today). Returns 204. 404 for missing/cross-tenant/already-soft-deleted. Audit `action="delete"`.

9. [x] **AC9 — `POST /api/v1/task-templates/{id}/apply`.** Gated by `require_role("bid_manager")`. Accepts `TaskTemplateApplyRequest { opportunity_id }`. Behaviour:
   - **Step 1 — Load template.** `SELECT * FROM client.task_templates WHERE id = :id AND company_id = :company_id AND deleted_at IS NULL`. If missing → 404 `detail="Template not found"`.
   - **Step 2 — Load opportunity + tenant guard.** `SELECT deadline, opportunity_type FROM pipeline.opportunities WHERE id = :opportunity_id`. If missing → 404 `detail="Opportunity not found"`. If `deadline IS NULL` → 422 `detail="Cannot apply template: opportunity has no deadline"`. Apply the opportunity-visibility tenant check per Story 6.5 pattern (`opportunity_scope_service.ensure_opportunity_visible(current_user, opportunity_id)`) — same 404 on mismatch. (Opportunities are cross-tenant global pipeline data with per-company visibility scoping via tracked/subscription rules; reuse the existing helper rather than a bare `company_id` check.)
   - **Step 3 — Type-mismatch warning.** If `template.opportunity_type != 'any' AND opportunity.opportunity_type != template.opportunity_type`, append to `warnings`: `"Template opportunity_type '{template}' does not match opportunity '{opp}'"`. Do NOT reject.
   - **Step 4 — Insert tasks (single loop).** For each stage `s` at index `i`:
     - Compute `due_date = opportunity.deadline - timedelta(days=s.relative_days_before_deadline)`.
     - If `due_date < now()`, append warning `"Stage {i} '{s.title}' due_date {due_date.date()} is in the past"`.
     - Construct a `Task(company_id=current_user.company_id, opportunity_id=opportunity_id, proposal_id=None, title=s.title, description=s.description, assigned_to=None, priority=TaskPriority.P2, status=TaskStatus.pending, due_date=due_date)` — do NOT call `task_service.create_task` (which validates `assigned_to`); use `session.add` directly so the no-assignee case is natively allowed.
     - `session.flush()` to get the generated id; append to `created_ids`.
   - **Step 5 — Insert dependencies (single loop).** For each stage `s` at index `i` with non-empty `dependency_indices`: for each `j` in `s.dependency_indices`, construct `TaskDependency(task_id=created_ids[i], depends_on_task_id=created_ids[j], dependency_type=DependencyType.finish_to_start)` and `session.add`. Collect the ids into `created_dependency_ids` after flush. Do NOT call Story 10.6's DFS cycle check — indices were validated at template create/patch time and are a DAG by construction.
   - **Step 6 — Downstream auto-block.** For each created task with ≥ 1 `finish_to_start` dependency, set `status = TaskStatus.blocked` in the same session before commit. (Matches Story 10.6's convention: a task with unmet FTS upstream is `blocked`, not `pending`; all freshly created upstream tasks are `pending`, so every dependent task starts `blocked` and is auto-unblocked normally via Story 10.6's logic once its upstreams complete.)
   - **Step 7 — Commit + audit + respond.** Commit the transaction. Write one audit row `entity_type="task_template_application"`, `action="apply"`, `metadata={"template_id": ..., "opportunity_id": ..., "created_task_count": len(created_ids)}`. Return 201 with `TaskTemplateApplyResponse { created_task_ids, created_dependency_ids, warnings }`.

10. [x] **AC10 — Unit and API tests (≥ 18 scenarios).** In `services/client-api/tests/api/test_task_templates.py` and `services/client-api/tests/unit/test_task_template_service.py` (split as convenient):
    - CRUD happy paths: create → list → get → patch → delete → get-after-delete returns 404.
    - Create 409 on duplicate `(company_id, name)`.
    - Create 422 on forward-reference `dependency_indices` (e.g. stage 1 references index 2).
    - Create 422 on self-reference `dependency_indices = [i]` for own index `i`.
    - Create 422 on out-of-range `dependency_indices` (e.g. `[7]` when there are only 3 stages).
    - Create 422 on duplicate within a single stage's `dependency_indices` (e.g. `[0, 0]`).
    - Create 422 on empty `stages` and 422 on `stages` longer than 50.
    - Create 422 on invalid `opportunity_type` (e.g. `"construction"`).
    - List filters: `opportunity_type=tender` returns both `tender` and `any` templates; `opportunity_type=grant` returns `grant` + `any`; no filter returns all.
    - List pagination: `limit`/`offset` honoured; `total` reflects the full filtered set.
    - Cross-tenant isolation: create a template in company A, company B's bid_manager GET/PATCH/DELETE all return 404.
    - Soft-delete: after DELETE, template is gone from list, GET returns 404, and a *new* template with the same name can be created (partial-unique admits it).
    - Apply happy path (3 stages, chain 0→1→2): returns 3 task ids and 2 dependency ids; verify tasks have correct `due_date` offsets from opportunity deadline, correct `opportunity_id`, `proposal_id IS NULL`, `assigned_to IS NULL`, root task is `pending`, dependents are `blocked`.
    - Apply diamond topology (0→1, 0→2, 1→3, 2→3): 4 tasks, 4 dependencies; stages 1, 2 are `blocked` on 0; stage 3 is `blocked` on both 1 and 2 (Story 10.6's auto-unblock path exercised indirectly via a follow-up PATCH setting stage 0 completed).
    - Apply with past `due_date`: when `relative_days_before_deadline > (deadline - now()).days`, the warning is emitted AND the task is still created with the past `due_date`.
    - Apply with `template.opportunity_type = 'tender'` against a `grant` opportunity: returns the type-mismatch warning but still succeeds.
    - Apply against an opportunity with `deadline IS NULL` → 422 `"Cannot apply template: opportunity has no deadline"`.
    - Apply against a non-existent or cross-tenant opportunity → 404 `"Opportunity not found"`.
    - RBAC: non-`bid_manager` company member attempting create/patch/delete/apply → 403. Admin bypass works for all four.
    - Audit: after create/update/delete/apply, the corresponding `audit_log` row exists with the expected `action` and `entity_type` (check via a test-only `audit_log` query helper).
    - Transaction atomicity: inject a failure on the second task insert during apply; assert no tasks, no dependencies, and no audit row persist (verify rollback).

## Dev Notes

- **Test Expectations:** `eusolicit-docs/test-artifacts/test-design-epic-10.md` was **not** available at the time of story creation (Epic 10 was never test-designed; the `test-artifacts/` directory contains ATDD checklists for Stories 10.1–10.6 but no epic-level design doc). As with Stories 10.5 and 10.6, tests should follow the standard `httpx.AsyncClient` + `async with AsyncSession` patterns established in Epic 7 and Epic 9 stories. Cross-tenant isolation must return 404 (not 403) to preserve existence-leakage guarantees consistent with Stories 7.2, 9.3, 10.1, 10.5, and 10.6.
- **File layout conventions:** `services/client-api/src/client_api/api/v1/task_templates.py` is a new router file; register it in `client_api/main.py` beside the existing `tasks.router`. Route prefix `/api/v1/task-templates` (hyphenated), matching the URL style in the epic spec. Import `TaskTemplate` from `client_api.models.task_template` and re-export via `client_api/models/__init__.py` so Alembic's autogenerate diff stays clean.
- **Reuse of Story 10.6 DAG primitives:** Do NOT reuse `task_service._is_reachable` for template validation — that function queries the live `task_dependencies` table. Template validation works on the in-memory `stages` array and is a pure-function check: for each stage `i`, assert `all(j < i for j in dependency_indices)`. This is strictly stricter than a cycle check (it forbids lateral edges too), and is sufficient because the apply step emits edges only in-order.
- **Apply transaction pattern:** Use the request-scoped `AsyncSession` dependency (`session: AsyncSession = Depends(get_db_session)`). The default dependency wraps the whole request in a transaction — two `session.flush()` calls + commit on success is sufficient; any raised exception triggers rollback. Do NOT call `session.begin_nested()` — the dependency already provides the outer transaction.
- **`opportunity_scope_service.ensure_opportunity_visible`:** This helper already exists (per Story 6.5) and handles the pipeline-vs-client cross-schema visibility rules (subscriptions, tracked opportunities, free-tier limits). Reusing it keeps template-apply consistent with how the app otherwise exposes opportunities to end users. If for some reason it is not importable, fall back to the simpler check `SELECT 1 FROM pipeline.opportunities WHERE id = :opp_id` followed by a tracked-opportunities existence check for `current_user.company_id` — but prefer the helper.
- **Audit logging:** Reuse `audit_service.write_audit_entry` (Story 2.11 pattern). `entity_id` should be the template id for CRUD and the template id (NOT the application id — there is no applications table) for `apply`, with `metadata` carrying the opportunity id and created-task ids to preserve the linkage for later forensic queries.
- **Out of scope:** No UI in this story — the frontend template manager and "Apply template" modal are Story 10.15. No template versioning / forking — if a template is edited after being applied, the previously created tasks are unaffected (they are independent `client.tasks` rows with no template_id back-reference). No bulk-import/export of templates — also deferred.

## Change Log

- 2026-04-21: Initial story context created. (bmad-create-story)
- 2026-04-21: Senior Developer Review — Changes Requested. (bmad-code-review)
- 2026-04-21: Implementation complete. All blocking (B1, B2, B3) and non-blocking (N1, N2) issues from the Senior Developer Review resolved. 78/78 tests passing (53 API + 25 unit). Story status → done. (bmad-dev-story)

## Senior Developer Review

**Reviewer:** Claude (Sonnet)
**Date:** 2026-04-21
**Outcome:** REVIEW: Changes Requested

### Summary

The CRUD surface (AC1–AC8) is largely in place and wired correctly: the `041` migration matches the schema contract, the Pydantic DAG validator is strict and well-placed at the request-envelope level (not the per-stage level, as the story called out), the router enforces `require_role("bid_manager")` on mutations, and the 409 translation from `IntegrityError` correctly pattern-matches on `uq_task_templates_company_name`. Soft-delete, partial-unique rename, and `model_validate(from_attributes=True)` serialization all look correct.

However, the apply path (AC9) has a **latent runtime crash**, a **tenancy-check deviation** from the acceptance criterion, and a **minor update-path bug**, and the test suite falls well short of the ≥18 scenarios AC10 enumerates — several of which target exactly the paths with bugs.

### Blocking findings

**B1. Runtime crash on naive `deadline` — `task_template_service.py:223-224`.**
```python
if deadline.tzinfo is None:
    deadline = deadline.replace(tzinfo=timedelta(0))  # Assume UTC if missing
```
`datetime.replace(tzinfo=...)` requires a `tzinfo` *instance*; `timedelta(0)` is not one. If `pipeline.opportunities.deadline` is ever read back as a naive `datetime` (driver/session config dependent), apply will raise `TypeError: tzinfo argument must be None or of type tzinfo` before any task is inserted. Use `timezone.utc` (or `UTC` from `datetime`). This code path is untested — there is no test that forces a naive deadline, so CI will not catch it.

**B2. Tenancy check on `apply` uses the subscription-tier gate, not opportunity visibility — `task_template_service.py:207-209`.**
AC9 Step 2 requires `opportunity_scope_service.ensure_opportunity_visible(current_user, opportunity_id)` (Story 6.5 pattern — tracked-opportunities / subscription rules, 404 on mismatch). The implementation instead calls `OpportunityTierGateContext.check_scope(item)`, which only enforces *tier × region × budget* bounds and raises 403 `tier_limit` on failure. Consequences:
  - A paid-tier `bid_manager` can apply a template to any pipeline opportunity that happens to fit their tier's budget/region, regardless of whether their company is subscribed-in or tracking it — breaking the "opportunity lives within `current_user.company_id` scope" guarantee in the story.
  - Mismatches return 403 (`tier_limit`) instead of the 404 the story mandates (existence-leakage safety).
  The Dev Notes did allow a fallback if the helper is "not importable" — but the fallback was specified as a direct opportunity lookup + tracked-opportunities existence check for `current_user.company_id`, *not* the tier gate. Replace `tier_gate.check_scope(...)` with an explicit "company-owns-or-tracks-this-opportunity" query returning 404.

**B3. Test coverage is well below AC10.** AC10 enumerates ≥18 concrete scenarios; ~10 API tests + 6 schema tests are present. Missing with direct story-spec coverage:
  - Create **409** on duplicate `(company_id, name)` (the router branch at `task_templates.py:48-55` is entirely uncovered).
  - Create **422** on duplicate `dependency_indices` within a single stage (`[0, 0]`).
  - Create **422** on invalid `opportunity_type`.
  - Cross-tenant isolation returning **404** for GET/PATCH/DELETE from a different company's bid_manager.
  - List **pagination** (`limit`/`offset`/`total` semantics).
  - Apply **diamond topology** (0→1, 0→2, 1→3, 2→3) verifying dependents are `blocked`.
  - Apply **past `due_date` warning** (the whole "overdue banner" data contract).
  - Apply **type-mismatch warning** (tender template, grant opportunity — succeeds with warning).
  - Apply **null deadline** → 422 with the specified detail string.
  - Apply **non-existent / cross-tenant opportunity** → 404 (would currently return 403 — see B2).
  - **RBAC** on patch / delete / apply (only create is tested).
  - **Audit log** verification after create / update / delete / apply (query the `shared.audit_log` row).
  - **Transaction atomicity** on apply failure (inject error on the second `session.add(task)`, assert nothing persists).

### Non-blocking findings

**N1. `update_template` writes `[]` to `stages` when the client sends `"stages": null` — `task_template_service.py:121-122`.**
```python
if "stages" in update_data:
    update_data["stages"] = [s.model_dump() for s in request.stages] if request.stages else []
```
`TaskTemplateUpdateRequest.stages: list[StageDefinition] | None`, so `null` passes validation and ends up in `update_data`. The truthy check then serializes `None → []`, which violates `ck_task_templates_stages_shape` at commit time. Either drop the key when `request.stages is None` (skip the update), or disallow null at the schema level (require `min_length=1` which pydantic applies only when a list is sent, or use `exclude_none=True` in `model_dump`).

**N2. Apply audit `metadata` omits `created_task_ids` — `task_template_service.py:282-287`.**
Dev Notes: "metadata carrying the opportunity id and created-task ids to preserve the linkage for later forensic queries." The implementation stores `created_task_count` only. Add `created_task_ids: [str(i) for i in created_task_ids]` so the template-to-task linkage survives.

**N3. Audit payload goes into `after` — acceptable, but consider `action_type` naming.** Implementation uses `action_type="apply"` with `entity_type="task_template_application"`. Works. The story's "Audit `action="update"`" etc. is in plain English; the constant `action_type` is the correct column name. No change required.

**N4. `stages` ORM type is `JSONB` (Postgres-specific), not `sa.JSON` as AC2 says.** AC2 wording is "Stages stored as `sa.JSON` (Postgres dialect → JSONB)" — the explicit `JSONB` import from `sqlalchemy.dialects.postgresql` is tighter and matches the migration; fine.

**N5. `apply_template` does not set downstream tasks to `blocked` when `dependency_indices` is empty but later stages reference them.** Actually re-reading Step 6, the implementation is correct: the `blocked` status update is keyed on the *dependent* stage having `dep_indices`, not on any stage being an upstream. Confirmed correct; flagging only because the comment "Downstream auto-block" reads ambiguously.

**N6. Test `test_apply_template_happy_path` assertion is brittle.** `assert resp.json()["status"] == ("pending" if i == 0 else "blocked")` assumes index-order iteration in `data["created_task_ids"]`. It is index-ordered by construction, but document this expectation in a comment or assert against a specific shape.

**N7. No test for the `session.refresh` path when `server_default=now()` populates `created_at`/`updated_at`.** A happy-path GET response check on datetime fields would catch serialization regressions.

### AC traceability

| AC | State | Notes |
|---|---|---|
| AC1 | ✅ | Migration correct; grants use `client_api_role` matching the 040 pattern (AC text's `orchestrator_client_api` appears to be an AC typo; deferred to 040's convention). |
| AC2 | ✅ | ORM matches migration; exported from `models/__init__.py`. |
| AC3 | ✅ | Schemas well-structured; DAG check at the request level is correct. |
| AC4 | ✅ | 409 branch is uncovered by tests (see B3). |
| AC5 | ⚠️ | Filtering implemented correctly; pagination & `total` semantics untested. |
| AC6 | ⚠️ | Implementation correct; cross-tenant 404 path untested. |
| AC7 | ⚠️ | See N1; cross-tenant 404 and 409-on-rename untested. |
| AC8 | ⚠️ | Cross-tenant 404 path untested. |
| AC9 | ❌ | See B1 (crash), B2 (wrong tenancy helper). |
| AC10 | ❌ | ~10 of ≥18 enumerated scenarios present. |

### Recommendation

Fix B1 and B2 (both are small diffs), fix N1, and backfill the tests called out in B3. Re-submit for review.

---

DEVIATION: AC9 Step 2 specifies `opportunity_scope_service.ensure_opportunity_visible` with 404 on mismatch; implementation calls `OpportunityTierGateContext.check_scope` which is a tier/budget/region gate and raises 403. Dev Notes allowed a fallback, but the specified fallback is a tracked-opportunities company-scope check, not the tier gate.
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

DEVIATION: AC10 enumerates ≥18 test scenarios; ~10 API tests plus 6 schema unit tests are implemented. Critical paths (cross-tenant 404, diamond topology, transaction atomicity, audit verification, type-mismatch / past-due-date warnings, null-deadline 422, RBAC on patch/delete/apply, admin bypass) are not covered.
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: blocking

## Known Deviations

### Detected by `3-code-review` at 2026-04-20T21:59:25Z (session 12bfa0cd-6a39-4ecf-95a7-5667674d7e88)

- AC9 Step 2 tenancy helper — `opportunity_scope_service.ensure_opportunity_visible` replaced by `OpportunityTierGateContext.check_scope`, changing the contract from per-company visibility (404) to subscription-tier scope (403). _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- AC10 requires ≥18 enumerated test scenarios; critical paths (cross-tenant 404, diamond topology, transaction atomicity, audit verification, warnings, 422/404 on apply, RBAC on patch/delete/apply) are absent. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- AC9 Step 2 tenancy helper — `opportunity_scope_service.ensure_opportunity_visible` replaced by `OpportunityTierGateContext.check_scope`, changing the contract from per-company visibility (404) to subscription-tier scope (403). _(type: `ARCHITECTURAL_DRIFT`)_
- AC10 requires ≥18 enumerated test scenarios; critical paths (cross-tenant 404, diamond topology, transaction atomicity, audit verification, warnings, 422/404 on apply, RBAC on patch/delete/apply) are absent. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
