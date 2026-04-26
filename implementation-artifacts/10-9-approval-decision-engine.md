# Story 10.9: Approval Decision Engine

Status: complete

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-9-approval-decision-engine
- **Points:** 3
- **Type:** backend
- **Module:** Client API (`client_api.api.v1.approvals` (new), `client_api.models.approval_decision` (new), `client_api.schemas.approvals` (new), `client_api.services.approval_decision_service` (new), new Alembic migration `043_approval_decisions`)
- **Priority:** P1 (Epic 10 final governance gate — unblocks Story 10.16 Approval Pipeline UI, enables the Story 9.10 `ApprovalDecided` notification consumer to actually receive events, and closes out the approvals sub-epic)
- **Depends On:** Story 10.1 (`ProposalCollaborator` table + `ProposalCollaboratorRole` enum — the per-proposal role lookup used in stage-role enforcement), Story 10.2 (`require_proposal_role` FastAPI dependency factory + 404-on-cross-tenant existence-leakage pattern), Story 10.8 (`approval_workflows` + `approval_stages` tables, ORM models, `proposal_collaborator_role` enum reuse — the cascade-guard in 10.8 specifically targets THIS story's `approval_decisions` table and the partial unique index on `is_default` gates workflow selection), Story 2.11 (`audit_service.write_audit_entry`), Story 1.5 (`EventPublisher` / Redis Streams — the event emission substrate), Story 9.10 (`ApprovalDecided` event schema + `eu-solicit:approvals` consumer — this story is the first producer).

## Story

As a **designated reviewer (bid_manager, technical_writer, financial_analyst, legal_reviewer) on a proposal**,
I want **to cast my decision (approve / reject / return for revision) on the current approval stage and have the system automatically advance to the next stage when I approve**,
so that **proposals move through our company-defined review pipeline without hand-off emails or shared spreadsheets, and my teammates receive notifications when a decision changes the proposal's state.**

## Description

This story is the runtime counterpart to Story 10.8's workflow-definition surface. It adds the `approval_decisions` table that Story 10.8's cascade-guard was designed against, wires up a `POST /proposals/{proposal_id}/approvals/decide` endpoint, emits the `ApprovalDecided` event on Redis Stream `eu-solicit:approvals` so Story 9.10's notification consumer can fire, and — when the **final** stage of a workflow is approved — marks the proposal as fully approved by setting `proposals.approved_at` (a new column added in this story's migration).

The decision engine enforces four gates in order before writing:

1. **Proposal tenancy gate** — `require_proposal_role(*ALL_ROLES_EXCEPT_READ_ONLY)` resolves the caller's role on the target proposal; 404 on cross-tenant or missing proposal (existence-leakage safe per Stories 10.1 / 10.2 / 10.5 / 10.6 / 10.7 / 10.8).
2. **Stage belongs to a live workflow in the caller's company** — `stage_id` is validated to (a) exist, (b) belong to a workflow whose `deleted_at IS NULL`, (c) belong to a workflow with `company_id == current_user.company_id`. Any failure → 404 (NOT 422 or 403 — a stage in another company simply does not exist from the caller's viewpoint).
3. **Stage is gated by the caller's `required_role` on the PROPOSAL** — `stage.required_role` is compared against the caller's `proposal_collaborators.role` on this proposal. The caller's **company-level** role (admin / member) is NOT what gates this check — the comparison is exclusively against the proposal-level collaborator role from Story 10.1. Company admins DO bypass this check (consistent with the admin-bypass in Story 10.2's `require_proposal_role`). Mismatch → 403 `"Your role on this proposal is not authorised to decide on this stage"`.
4. **Prior stages all have an `approved` decision on this proposal** — for the stage's workflow, every stage with `order < stage.order` must have at least one `approval_decisions` row with `decision = 'approved'` for this `(proposal_id, stage_id)` pair. If any prior stage is unapproved (no decision OR a `rejected` / `returned_for_revision` decision), return 409 `"Prior stage(s) not yet approved"`. Stages MAY be decided multiple times (a `rejected` decision can be overridden by a later `approved` on the same stage), so the check is "latest decision per prior stage is `approved`", ordered by `created_at DESC` with a single-per-stage tiebreak.

Once the gates pass, the service inserts the `approval_decisions` row inside a single transaction that also performs two conditional side-effects:

- **Auto-advance marker.** If the decision is `approved` and `stage.auto_advance = True`, the response body includes the next pending stage's metadata (`next_stage: {id, name, order, required_role, auto_advance}`) so the UI can immediately render the next step without a refresh. If no next stage exists, `next_stage` is `null`. If `auto_advance = False`, `next_stage` is `null` even if a next stage exists (the UI must render a "Progress to next stage" CTA). The underlying data does not require an explicit "advancement" write — stages are purely ordered definitions, and "what stage is current" is derived from the latest decision set. The `auto_advance` flag is purely a UI hint.
- **Final-stage approval.** If the decision is `approved` AND the stage is the last in its workflow (`stage.order == max(workflow.stages.order)`) AND no higher-order stage exists, set `proposals.approved_at = now()` on the target proposal. This is a nullable timestamp (added in migration 043) rather than a new `proposals.status` value because the existing `ck_proposals_status` CHECK constraint only permits `{'draft', 'active', 'archived'}` and mutating the check + adding `'approved'` would conflict with E11 / E07 existing state transitions (see Dev Notes for the rejected alternative).

After the transaction commits, the service emits `ApprovalDecided` to Redis Stream `eu-solicit:approvals` via `EventPublisher` (pattern from `webhook_service._publish_subscription_changed` — **must be called AFTER `session.commit()`**; emission failure is logged but never bubbles up to the caller since the decision is already durable). The event schema (`eusolicit_models.events.ApprovalDecided`) constrains `decision` to `Literal["approved", "rejected"]`, so `returned_for_revision` decisions do NOT emit an event — they are a soft signal captured only in the decision row + audit trail. If a future Epic 10 or Epic 12 iteration wants to notify on revision requests, the `ApprovalDecided` schema will need to be extended and Story 9.10's consumer re-certified. This story keeps the event surface aligned with what 9.10 currently handles.

Five design decisions worth calling out:

**(a) `approval_decisions.stage_id` uses `ON DELETE RESTRICT`, the belt to Story 10.8's suspenders.** Story 10.8's Dev Notes flag this: when 10.8's `_check_decisions_exist` cascade-guard was designed, the contract was that a DB-level `DELETE` on a stage would ALSO fail if a decision referenced it. Migration 043 therefore declares `sa.ForeignKey("client.approval_stages.id", ondelete="RESTRICT")`. A stage cannot be hard-deleted while any decision references it — the PATCH / DELETE workflow endpoints in 10.8 already pre-check for decisions and return 409, but the FK restricts defensively should the guard ever be bypassed (e.g. a raw SQL migration script).

**(b) No "current stage pointer" column is materialised on `proposals`.** The current stage is a derived value: `MIN(stage.order) WHERE no approved decision exists for stage` — i.e. the first stage that is not yet approved. Every additional write surface (a denormalised pointer) introduces a synchronisation bug; treating "current stage" as a read-model query eliminates drift. The endpoint response includes the derived `next_stage` for UI convenience but the DB is not asked to track it.

**(c) Decisions are append-only; no soft-delete.** A `rejected` decision can be superseded by a later `approved` decision on the same stage (the prior-stage-approval check uses the latest decision per stage), but decisions themselves are never updated or deleted. This preserves the full audit history — Story 10.16's UI is specified to show ALL decisions in a history table, not just the latest. Any "undo" is accomplished by submitting a new decision with the opposite verdict and a comment; the older row stays.

**(d) Proposal ↔ workflow linkage is implicit, not stored.** Story 10.8's Dev Notes explicitly state: "No per-workflow assignment to proposals (Story 10.9 will introduce the `proposal_id → workflow_id` linkage, possibly via `proposals.approval_workflow_id` or a dedicated link table; this story does NOT add that column)." After analysis we reject both forms in favour of letting the first decision's `stage.workflow_id` define the linkage. Reasoning: (i) at most one workflow is "in flight" per proposal at a time in the MVP; (ii) a caller submitting a decision against a stage of workflow W on proposal P effectively selects W as the pipeline for P from that point forward; (iii) adding a `proposals.approval_workflow_id` column creates a writeable surface ("assign workflow" endpoint) that the epic does not require — Story 10.16's UI pre-selects the company default workflow client-side when opening the approval panel. The derived-linkage rule is: any decision on proposal P references a `stage_id`, and that stage's `workflow_id` IS the workflow for P. A guard (AC6) rejects mixing workflows — if `proposal_decisions_history(P)` contains at least one decision, a new decision's `stage.workflow_id` MUST equal the earliest decision's `stage.workflow_id`; otherwise 409 `"Proposal already in a different approval workflow"`. This keeps the schema lean while preserving the single-workflow-per-proposal invariant.

**(e) `ApprovalDecided` event publishes `proposal_owner_id` from `proposals.created_by` (with a fallback).** Story 9.10's consumer expects `proposal_owner_id` to be a valid active user id and sends the decision notification to that user's email. The event emitter therefore reads `proposals.created_by` for the notification target. If `created_by IS NULL` (legacy E11 grant proposals migrated without it — see `proposals.created_by: Mapped[uuid.UUID | None]`), the emitter falls back to the first bid_manager on the proposal (SELECT user_id FROM client.proposal_collaborators WHERE proposal_id = :id AND role = 'bid_manager' ORDER BY granted_at ASC LIMIT 1). If neither exists, the event is skipped with a WARN log — no owner means no notifiable target. The decision write still succeeds.

The router is mounted at `/api/v1/proposals/{proposal_id}/approvals` (matches the `proposal_comments` and `proposal_section_locks` routing convention established in Stories 10.3 / 10.4).

## Acceptance Criteria

1. [x] **AC1 — `client.approval_decisions` Alembic migration (`043_approval_decisions.py`) + `proposals.approved_at` column.** Create a new migration with `revision = "043"`, `down_revision = "042"`. Two upgrade steps:

    **`client.approval_decisions`:**
    - `id` UUID primary key, `server_default=sa.text("gen_random_uuid()")`.
    - `proposal_id` UUID NOT NULL, FK `client.proposals.id` ON DELETE CASCADE.
    - `stage_id` UUID NOT NULL, FK `client.approval_stages.id` ON DELETE **RESTRICT** (the belt to Story 10.8's cascade-guard suspenders — see design decision (a)).
    - `decision` TEXT NOT NULL. Add a CHECK constraint `decision IN ('approved', 'rejected', 'returned_for_revision')` named `ck_approval_decisions_decision`.
    - `comment` TEXT NULL.
    - `decided_by` UUID NOT NULL (soft link to `client.users.id`, no FK — match Story 10.7 / 10.8 pattern).
    - `created_at` TIMESTAMPTZ NOT NULL `server_default=sa.func.now()`.
    - Indexes: `ix_approval_decisions_proposal_id`, `ix_approval_decisions_stage_id`, `ix_approval_decisions_proposal_stage` (composite on `(proposal_id, stage_id)` — hot path for the prior-stage-approval check and for the workflow-linkage guard).

    **`client.proposals.approved_at`:**
    - `op.add_column("proposals", sa.Column("approved_at", sa.DateTime(timezone=True), nullable=True), schema="client")`.
    - Index `ix_proposals_approved_at` on `approved_at`. (Supports the Story 12.4 ROI dashboard's "approved proposals per period" query planned later — cheap to index upfront.)

    **Grants:** `GRANT SELECT, INSERT, UPDATE, DELETE ON client.approval_decisions TO client_api_role`. `proposals` grants are unchanged — the column add inherits existing table grants.

    **`downgrade()`** drops `ix_proposals_approved_at`, removes `proposals.approved_at`, then drops `approval_decisions` (and its three indexes). No reverse data migration — approved_at is nullable and decision rows cannot be faithfully rehydrated from nothing.

2. [x] **AC2 — `ApprovalDecision` ORM model.** Add `services/client-api/src/client_api/models/approval_decision.py` with `ApprovalDecision` — columns mirror the migration; `__table_args__` includes the three indexes, the CHECK constraint, and `{"schema": "client"}`. No relationship to `ApprovalStage` (one-directional: decisions reference stages, but stages do not need to list their decisions — keeps the Story 10.8 ORM surface untouched). Export from `client_api/models/__init__.py`.

   Additionally: update `client_api/models/proposal.py` to add `approved_at: Mapped[datetime | None]` column declaration (nullable TIMESTAMPTZ, no server_default) and update `__table_args__` to include `sa.Index("ix_proposals_approved_at", "approved_at")`. Do NOT touch the existing `ck_proposals_status` CHECK constraint — the approval state is tracked via the new column, not via `status`.

3. [x] **AC3 — Pydantic schemas (`schemas/approvals.py`).** Implement:
   - `ApprovalDecisionType(str, Enum)`: `approved | rejected | returned_for_revision`. Match the Python enum to the DB CHECK constraint verbatim.
   - `ApprovalDecisionRequest(BaseModel)`: `stage_id: UUID`, `decision: ApprovalDecisionType`, `comment: str | None` (max 5000 chars). A `@model_validator(mode="after")` requires `comment` to be non-empty (after `.strip()`) when `decision in {rejected, returned_for_revision}` — rejections and revision requests without a rationale are a product-UX anti-pattern; raise `ValueError("Comment is required when rejecting or returning for revision")`.
   - `ApprovalDecisionResponse(BaseModel)`: `id`, `proposal_id`, `stage_id`, `decision`, `comment`, `decided_by`, `created_at`. `model_config = ConfigDict(from_attributes=True)`.
   - `NextStageInfo(BaseModel)`: `id`, `name`, `order`, `required_role: ProposalCollaboratorRole`, `auto_advance: bool`. (Subset of `ApprovalStageResponse` — specifically does NOT include `workflow_id` to avoid leaking the internal linkage.)
   - `ApprovalDecideResponse(BaseModel)`: `decision: ApprovalDecisionResponse`, `next_stage: NextStageInfo | None`, `proposal_approved: bool` (True iff the decision was `approved` on the final stage; drives the UI's "Proposal fully approved" banner).
   - `ApprovalDecisionHistoryResponse(BaseModel)`: `items: list[ApprovalDecisionResponse]`, `total: int`. Used by the GET history endpoint (AC8).

4. [x] **AC4 — `POST /api/v1/proposals/{proposal_id}/approvals/decide`.** Gated by `Depends(require_proposal_role(ProposalCollaboratorRole.bid_manager, ProposalCollaboratorRole.technical_writer, ProposalCollaboratorRole.financial_analyst, ProposalCollaboratorRole.legal_reviewer))` — i.e. all roles except `read_only`. The Story 10.2 dependency handles the 404-on-cross-tenant guard and admin bypass.

   Body: `ApprovalDecisionRequest`. Service behaviour (single DB transaction):
   - Load the stage by `stage_id` with an `INNER JOIN` to `approval_workflows` filtered by `deleted_at IS NULL` and `company_id = current_user.company_id`. If the row is missing → 404 `"Stage not found"` (existence-leakage safe for cross-tenant stages).
   - **Workflow-linkage guard (design decision (d)):** Query `SELECT stage.workflow_id FROM approval_decisions ad JOIN approval_stages ON ad.stage_id = stage.id WHERE ad.proposal_id = :proposal_id ORDER BY ad.created_at ASC LIMIT 1`. If a row exists AND its workflow_id ≠ the target stage's workflow_id → 409 `"Proposal already in a different approval workflow"`.
   - **Role-gate on stage:** Skip when `current_user` is a company admin (admin bypass); otherwise compare the caller's proposal-collaborator role (available on `current_user` after the `require_proposal_role` dep resolves — the dep attaches the role to the `CurrentUser` object or the service re-queries `client.proposal_collaborators` given `(proposal_id, user_id)`) against `stage.required_role`. Mismatch → 403 `"Your role on this proposal is not authorised to decide on this stage"`. The `_write_denial_audit` helper from `rbac.py` is invoked on denial so the access_denied audit row is written in a dedicated session that survives the rollback.
   - **Prior-stage gate:** For each `prior_stage` in the same workflow with `order < stage.order`, fetch the **latest** decision per prior_stage (`SELECT DISTINCT ON (stage_id) stage_id, decision FROM approval_decisions WHERE proposal_id = :proposal_id AND stage_id IN (:prior_stage_ids) ORDER BY stage_id, created_at DESC`). If any prior stage has no decision or its latest decision ≠ `'approved'` → 409 `"Prior stage(s) not yet approved"` with a structured detail listing the first unapproved prior stage's name + order.
   - Insert the `ApprovalDecision` row with `decided_by=current_user.user_id`. Flush to obtain `decision.id`.
   - **Final-stage marker:** Fetch `SELECT MAX("order") FROM approval_stages WHERE workflow_id = :workflow_id`. If `stage.order == max_order` AND `request.decision == ApprovalDecisionType.approved`, execute `UPDATE proposals SET approved_at = now() WHERE id = :proposal_id AND approved_at IS NULL` (the `IS NULL` guard makes a second final-stage approval idempotent — we do NOT reset a pre-existing `approved_at` if a later decision supersedes the original approval).
   - Write one audit row: `entity_type="approval_decision"`, `action_type="create"`, `entity_id=decision.id`, `after={"proposal_id": str(proposal_id), "stage_id": str(stage_id), "decision": decision.value, "final_stage": bool, "proposal_approved": bool}`. Use `audit_service.write_audit_entry` in-session.
   - Compute `next_stage`: `SELECT * FROM approval_stages WHERE workflow_id = :workflow_id AND "order" > :current_order ORDER BY "order" ASC LIMIT 1`. If `request.decision == approved` AND `stage.auto_advance` AND such a row exists, return it in `NextStageInfo`; otherwise `next_stage = None`.
   - Compute `proposal_approved`: `True` iff the final-stage marker branch fired (regardless of whether `approved_at` was already set).
   - Commit (the dependency wrapper handles this).
   - **After commit**: emit `ApprovalDecided` to Redis Stream `eu-solicit:approvals` via `EventPublisher.publish(stream="eu-solicit:approvals", event_type="ApprovalDecided", payload={...}, source_service="client-api", tenant_id=str(company_id), correlation_id=<request id>)`. Emission is wrapped in `try/except Exception` that logs at WARN and does NOT bubble up — the decision is already committed. Emission is skipped when `request.decision == returned_for_revision` (the event schema does not model this verdict; see design decision in the Description). Emission is also skipped when the owner cannot be resolved (see design decision (e)) with a structured WARN log.
   - Return 201 with `ApprovalDecideResponse`.

5. [x] **AC5 — Event payload shape matches `eusolicit_models.events.ApprovalDecided`.** The published payload (post-EventPublisher envelope) must deserialise cleanly via `TypeAdapter(ServiceEvent).validate_python(full_event_dict)` — i.e. the exact fields Story 9.10's `approval_consumer.py` expects:
   - `event_type: "ApprovalDecided"` (set automatically by `EventPublisher` — do not set it in payload).
   - `proposal_id: str(UUID)`
   - `approval_id: str(UUID)` — map to the newly-inserted `approval_decisions.id`.
   - `proposal_owner_id: str(UUID)` — resolved from `proposals.created_by`, with fallback to the first bid_manager on the proposal (design decision (e)). When neither can be resolved, the emission is skipped entirely.
   - `decision: Literal["approved", "rejected"]` — map from `ApprovalDecisionType`. `returned_for_revision` decisions do NOT emit (see AC4). The DB CHECK constraint carries all three; the event schema intentionally does not.
   - `decider_id: str(UUID)` — `current_user.user_id`.
   - `decision_note: str | None` — the request's `comment` field.
   - `source_service: "client-api"`.
   - `tenant_id: str(company_id)`.
   - `correlation_id: <request correlation id or uuid4 fallback>`.

   Add a unit test that constructs an `ApprovalDecided` model from the emitted envelope and asserts successful Pydantic validation — this is a contract test against Story 9.10's consumer.

6. [x] **AC6 — Workflow-linkage guard (mixed-workflow rejection).** When a proposal already has at least one decision, a new decision whose `stage.workflow_id` differs from the proposal's established workflow is rejected with 409 `"Proposal already in a different approval workflow"`. The "established workflow" is the workflow_id of the earliest decision row for that proposal (ordered by `created_at ASC`). Test cases:
   - Proposal has no decisions → any workflow is accepted (the first decision establishes the workflow).
   - Proposal has decisions on workflow A → decision on a stage in workflow A is accepted.
   - Proposal has decisions on workflow A → decision on a stage in workflow B (same company) is rejected with 409.
   - Proposal has decisions on workflow A → workflow A is soft-deleted by the bid_manager (Story 10.8 soft-delete path) → decisions already recorded remain valid, but a new decision attempt on a stage of workflow A now fails at the AC4 stage-load step with 404 (the stage-load JOIN filters `deleted_at IS NULL`).

7. [x] **AC7 — Multi-decision-per-stage semantics.** A stage can be decided multiple times on the same proposal; the DB does not enforce uniqueness on `(proposal_id, stage_id)`. The "latest decision per prior stage" gate uses `DISTINCT ON (stage_id) ... ORDER BY stage_id, created_at DESC`. Test cases:
   - Stage 1 decided `rejected` → stage 1 decided again `approved` → stage 2 attempt succeeds (latest on stage 1 is approved).
   - Stage 1 decided `approved` → stage 1 decided again `rejected` → stage 2 attempt fails with 409 (latest on stage 1 is rejected). This is the "unapprove" path; Story 10.16's UI should discourage but not prevent it.
   - Stage 1 decided `returned_for_revision` → stage 2 attempt fails with 409 (returned_for_revision is not `approved`). An `approved` decision on stage 1 after the revision succeeds.
   - Stage 1 decided `approved` → stage 1 decided `returned_for_revision` → stage 2 attempt fails with 409.

8. [x] **AC8 — `GET /api/v1/proposals/{proposal_id}/approvals/history`.** Gated by `Depends(require_proposal_role(*ALL_ROLES))` — all five collaborator roles including `read_only` can view decision history, consistent with comments-listing (Story 10.4). Returns `ApprovalDecisionHistoryResponse` with `items` ordered by `created_at ASC` (chronological). Each item is an `ApprovalDecisionResponse`. No pagination (proposal decision volumes are inherently bounded by stage count × retry count; a proposal with > 100 decisions is operator-remediated out-of-band). 404 on cross-tenant or missing proposal.

9. [x] **AC9 — Router wiring + main.py include.** Add `services/client-api/src/client_api/api/v1/approvals.py` with router `prefix="/proposals/{proposal_id}/approvals"`, `tags=["approvals"]`. Register two routes: `POST /decide` → `ApprovalDecideResponse`, `GET /history` → `ApprovalDecisionHistoryResponse`. Import as `from client_api.api.v1 import approvals as approvals_v1` in `client_api/main.py` and include via `api_v1_router.include_router(approvals_v1.router)` — place the include BESIDE `proposal_comments_v1` and `proposal_section_locks_v1` for locality.

   **Router include pattern note:** `proposal_comments_v1.router` and `proposal_section_locks_v1.router` are currently included with `prefix="/proposals"` because their internal router prefixes are `/{proposal_id}/...`. This story's router embeds the full prefix `/proposals/{proposal_id}/approvals` directly in `APIRouter(prefix=...)`, so the `include_router` call takes NO additional prefix. Verify the resulting path is `/api/v1/proposals/{proposal_id}/approvals/decide` (not doubled) before merging.

10. [x] **AC10 — Unit and API tests (≥ 22 scenarios).** Split between `services/client-api/tests/api/test_approvals.py` (router + HTTP-layer tests) and `services/client-api/tests/unit/test_approval_decision_service.py` (service-layer + schema validator tests).

    **Happy paths:**
    - Decide `approved` on the first stage of a 3-stage workflow → 201; response `decision.decision == "approved"`; `next_stage.id == stage2.id`; `proposal_approved is False`; one `approval_decisions` row persisted.
    - Decide `approved` on the final stage after prior approvals → 201; `next_stage is None`; `proposal_approved is True`; `proposals.approved_at` is non-null (verify via DB query).
    - Decide `approved` on a stage whose `auto_advance=False` → `next_stage is None` even though a next stage exists.
    - Decide `rejected` on stage 1 → 201; no `next_stage`; no event emitted with decision=rejected is valid (decision=rejected IS in the event schema; see AC5 — emit but decision note required).
    - Decide `returned_for_revision` on stage 2 → 201; event NOT emitted (ApprovalDecided does not support this verdict); audit row written.

    **Gate failures:**
    - Non-collaborator user (no row in `proposal_collaborators`) → **403** `"You are not a collaborator on this proposal"` (handled by `require_proposal_role` dep — the dep returns 403 for non-collaborators, NOT 404; see Story 10.2 precedent and `rbac.py` line 313).
    - `read_only` collaborator → 403 via dep's allowed-roles check.
    - Stage belongs to a soft-deleted workflow → 404 `"Stage not found"`.
    - Stage belongs to another company's workflow (same user, cross-tenant stage UUID) → 404.
    - Stage does not exist → 404.
    - Caller's proposal role ≠ stage's `required_role` (e.g. caller is `technical_writer`, stage requires `legal_reviewer`) → 403; access_denied audit row written.
    - Company admin bypass: admin is not a collaborator, stage requires `legal_reviewer` → 201 (admin bypasses both `require_proposal_role` and the stage-role check).
    - Prior stage 1 has no decision → attempt on stage 2 → 409 with detail naming stage 1.
    - Prior stage 1 latest decision is `rejected` → attempt on stage 2 → 409.
    - Prior stage 1 latest decision is `returned_for_revision` → attempt on stage 2 → 409.
    - Prior stage 1 has `rejected` followed by `approved` (override) → attempt on stage 2 → 201.

    **Workflow-linkage guard:**
    - Proposal with an existing decision on workflow A → attempt on a stage of workflow B → 409 `"Proposal already in a different approval workflow"`.
    - Proposal with no decisions → attempt on any live workflow's stage → 201 (workflow established).

    **Validation (422):**
    - `decision="rejected"` with empty/whitespace-only comment → 422 `"Comment is required when rejecting or returning for revision"`.
    - `decision="returned_for_revision"` with `comment=None` → 422.
    - `decision="approved"` with `comment=None` → 201 (comment optional for approvals).
    - Invalid `decision` value → 422 (Pydantic enum validation).
    - `stage_id` is not a valid UUID string → 422.

    **Event emission:**
    - After an `approved` decision commits, one message exists on `eu-solicit:approvals` with `event_type="ApprovalDecided"`, correct `approval_id`, `proposal_id`, `decider_id`, `decision="approved"`, `proposal_owner_id == proposal.created_by`. Validate the envelope via `TypeAdapter(ServiceEvent).validate_python(...)` to prove round-trip compatibility with Story 9.10's consumer.
    - When `proposals.created_by IS NULL`, the emitter falls back to the first bid_manager (ordered by `granted_at ASC`); event emitted with that user_id.
    - When neither `created_by` nor any bid_manager exists, NO event is emitted (the decision write still succeeds); a WARN log is emitted.
    - When `decision="returned_for_revision"`, no event is emitted.
    - When Redis is unavailable (monkey-patch `EventPublisher.publish` to raise), the decision write still succeeds (event failure is non-fatal); a WARN log is captured.

    **Final-stage + approved_at:**
    - Final stage approved → `proposals.approved_at` set to a non-null timestamp.
    - Final stage approved → stage 1 is re-decided `rejected` → `proposals.approved_at` stays set (the AC4 `WHERE approved_at IS NULL` idempotency guard does not reset, and we do not clear the field on a later `rejected`).
    - Final stage approved twice (edge case: workflow only has one stage OR a re-vote) → second call does not change `approved_at` (idempotency guard).

    **History endpoint (AC8):**
    - GET history on a proposal with 4 decisions returns all 4 ordered by `created_at ASC`.
    - GET history on a proposal with 0 decisions returns `items: []`, `total: 0`.
    - GET history by a `read_only` collaborator → 200 (read is allowed for all roles).
    - GET history cross-tenant → 404.

    **Migration tests:**
    - `test_migration_043_creates_approval_decisions_table` — table exists with all required columns + CHECK constraint.
    - `test_migration_043_adds_approved_at_to_proposals` — column exists, nullable, indexed.
    - `test_migration_043_stage_id_fk_restrict` — attempting a raw DB-level `DELETE FROM client.approval_stages WHERE id = :stage_id_with_decision` raises `IntegrityError` (ON DELETE RESTRICT).
    - `test_migration_043_proposal_id_fk_cascade` — deleting a proposal cascades to delete all its `approval_decisions` rows.
    - `test_migration_043_decision_check_constraint` — inserting `decision = 'maybe'` raises `IntegrityError`.
    - `test_migration_043_revision_chain` — `revision == "043"`, `down_revision == "042"`.
    - `test_migration_043_downgrade_reverses` — downgrade drops the table and removes `approved_at` + its index.

    **Audit trail:**
    - After each `create` decision (approved / rejected / returned_for_revision), one `shared.audit_log` row exists with `action_type="create"`, `entity_type="approval_decision"`, `entity_id == decision.id`, `after` carrying `proposal_id`, `stage_id`, `decision`, `final_stage`, `proposal_approved`.
    - After a 403 role-mismatch denial, one `shared.audit_log` row exists with `action_type="access_denied"` (written via `_write_denial_audit`).

## Dev Notes

- **Test Expectations (from Epic-level test design):** `eusolicit-docs/test-artifacts/test-design-epic-10.md` is **not** available — Epic 10 was never epic-test-designed (the `test-artifacts/` directory for Epic 10 contains only per-story ATDD checklists 10-1 through 10-8; no epic-level design doc exists). Coverage strategy therefore follows the established patterns: `httpx.AsyncClient` + `async with AsyncSession` for API tests; `pytest-asyncio` with `asyncio_mode = "auto"`; cross-tenant isolation returns 404 (NOT 403) for existence-leakage safety, consistent with Stories 7.2, 9.3, 10.1, 10.2, 10.4–10.8. Use the arrange/act/assert block format from `tests/api/test_task_templates.py` and `tests/api/test_approval_workflows.py`. For event-emission tests, use the Redis Streams pattern already in use in `tests/integration/test_redis_event_bus.py` — `XREAD`-poll the `eu-solicit:approvals` stream after the HTTP call commits and assert the envelope parses as a `ServiceEvent`. The `ApprovalDecided` schema from `eusolicit_models.events` is the canonical target.
- **File layout conventions:** `services/client-api/src/client_api/api/v1/approvals.py` is a new router file. Import as `from client_api.api.v1 import approvals as approvals_v1` in `client_api/main.py` and include via `api_v1_router.include_router(approvals_v1.router)` BESIDE `proposal_comments_v1` and `proposal_section_locks_v1`. Route prefix `/proposals/{proposal_id}/approvals` (fully embedded, unlike comments/locks which use the split-prefix pattern — see AC9 rationale). `tags=["approvals"]`.
- **Model init wiring:** Export `ApprovalDecision` from `client_api/models/__init__.py` (load-bearing for Alembic autogenerate; the base `MetaData` must see the table). The `approved_at` column addition to `Proposal` does NOT require a new import but DOES require the column declaration in `client_api/models/proposal.py` so ORM queries and eager-loads see it.
- **Proposal status vs. `approved_at` — why not extend the CHECK constraint?** The existing `proposals.ck_proposals_status` CHECK constraint permits `{draft, active, archived}` and is exercised by E07 / E11 state transitions across multiple services. Adding `approved` to the enum would require coordinated changes in E07's proposal_service, E11's reporting-template proposals, and the Story 12.4 ROI dashboard queries that currently group by status. The nullable `approved_at` timestamp approach is strictly additive: it unambiguously represents "fully approved at time T" without disturbing any existing status-based logic. The Story 12.4 dashboard can later filter `WHERE approved_at IS NOT NULL AND approved_at BETWEEN :from AND :to` for its "approved bids" KPI. If a future story needs an explicit status value (e.g. for serialisation in a report), compute it from `approved_at` (`status = 'approved' if approved_at IS NOT NULL else proposal.status`) at the response layer rather than at the DB layer.
- **Event publishing is post-commit, non-fatal.** Follow the pattern from `webhook_service._publish_subscription_changed` exactly: the `EventPublisher.publish` call lives OUTSIDE the DB transaction, is wrapped in a `try/except Exception` that logs at WARN, and never re-raises. Do NOT emit inside the service function that holds the session — route emission to the endpoint layer (after `return` from the service but before FastAPI closes the response) OR to a post-commit hook. The simplest pattern: the endpoint calls `await approval_decision_service.submit_decision(...)` which returns the constructed response dict PLUS enough metadata to build the event; after `session.commit()` completes (via the `get_db_session` dependency), the endpoint emits the event. Since this services uses `get_db_session` to auto-commit at response time, the recommended pattern here is to queue the event details on `request.state.post_commit_events` and emit in a FastAPI after-response hook, OR inline the emission at the end of the endpoint function AFTER an explicit `await session.commit()` inside the service. Study `webhook_service._handle_trial_will_end` and its caller for the exact pattern.
- **`require_proposal_role` caveat:** the dep validates the caller's role on the proposal but does NOT explicitly expose that role back to the service. Two implementation options: (a) the service re-queries `client.proposal_collaborators.role` for the `(proposal_id, user_id)` pair (one extra SELECT, ~300µs, acceptable); (b) the dep is extended to stash the resolved role on `current_user.proposal_role` (requires a change to `CurrentUser` — out of scope for this story). Pick option (a) for the MVP.
- **Admin bypass on stage role-gate:** `current_user.role == CompanyRole.admin.value` short-circuits the stage-role comparison. Rationale: company admins already bypass the `require_proposal_role` dep entirely (admin bypass step 4 in `rbac.py`); treating them as permitted for any stage is consistent. If the product later requires stage-gated admins (e.g. separating "finance-admin" from "legal-admin"), add a new `ApprovalRequiredRole` enum distinct from `ProposalCollaboratorRole` (rejected today — see Story 10.8 design decision (e)) and plumb it through.
- **Prior-stage query uses `DISTINCT ON`:** PostgreSQL-specific but the codebase is PostgreSQL-only (see `alembic/env.py`, `psycopg3` driver). The equivalent ANSI form (`ROW_NUMBER()` window + `HAVING`) is more portable but slower on hot paths; stick with `DISTINCT ON`. Story 10.6's task-dependency cycle check uses similar PostgreSQL-specific recursive CTEs, so the precedent is established.
- **Correlation ID plumbing:** extract `request.headers.get("X-Request-ID")` at the endpoint layer and pass into the service as `correlation_id`. `EventPublisher.publish` accepts `correlation_id` as a keyword; when absent it generates a UUID4. Keep the inbound header as the source of truth when present for cross-service trace correlation.
- **Final-stage detection must tolerate a single-stage workflow.** A workflow with only one stage means stage 1 is both the first and the last; approving it triggers the final-stage marker immediately. AC4's `MAX("order")` query returns 1 and the comparison `stage.order == max_order` (1 == 1) holds. Unit-test this edge.
- **Out of scope:** No workflow re-assignment (once a proposal is linked to workflow A via its first decision, the linkage is sticky — changing it requires hard-deleting all decisions, which no endpoint allows; this is acceptable for MVP). No "withdraw decision" endpoint (decisions are append-only — the workaround is to submit a counter-decision). No email rendering (Story 9.6 owns the template; Story 9.10 consumes the event). No UI (Story 10.16). No batch approval (one stage per request). No `ApprovalRequested` event emission for THIS story — the epic lists "approval requested" conceptually but the Story 9.10 consumer already handles the schema; producing that event is a latent need that future stories may cover if a "request approval" endpoint is added.
- **Future-compatibility note (Story 10.16):** the UI will render a stepper showing (for each stage) the latest decision + decider name. The `GET /history` endpoint is sufficient — the UI will group by `stage_id` client-side. If a future iteration wants a server-computed "current state" view (stages × latest-decision), add a derived read endpoint at that time; keep the API surface minimal today.

## Change Log

- 2026-04-21: Initial story context created. (bmad-create-story)
- 2026-04-21: Senior Developer Review — Changes Requested. (bmad-code-review)
- 2026-04-21: All blocking and non-blocking review items resolved. (bmad-dev-story)
  - **Must-fix 1 (AC10 migration-test gap):** Created `tests/integration/test_migration_043.py` with 10 tests covering table structure, CHECK constraint, nullable `approved_at`, `ix_proposals_approved_at` index, `ON DELETE RESTRICT` on `stage_id`, `ON DELETE CASCADE` on `proposal_id`, and downgrade reversal (drops table + column + index).
  - **Must-fix 2 (raw SQL injection):** Replaced f-string interpolation in `_get_latest_decisions_per_stage` with `text(...).bindparams(bindparam("sids", expanding=True))` — UUIDs are now always bound, never concatenated. Updated the DISTINCT-ON unit test to also assert that UUID values do NOT appear in the SQL string.
  - **Must-fix 3 (AC4 non-collaborator contract):** Updated AC4 text to correctly state 403 (not 404) for non-collaborators — aligned with `rbac.py:313` behaviour and the existing `test_decide_non_collaborator_returns_403` test.
  - **Should-fix 4 (router type annotations):** Changed `Annotated[get_db_session, Depends()]` → `Annotated[AsyncSession, Depends(get_db_session)]` for both endpoints; added `AsyncSession` import.
  - **Should-fix 5 (private helper across modules):** Renamed `_maybe_emit_approval_decided` → `emit_approval_decided` (public); updated all callers (router + unit tests).
  - **Should-fix 6 (double MAX query):** Hoisted `max_order` computation before the final-stage branch; removed the duplicate `select(func.max(...))` in the audit step.
  - **Should-fix 7 (unused import):** Removed `desc` from `from sqlalchemy import ...`.
  - **Should-fix 8 (enum/string comparison):** Prior-stage check now compares `latest != "approved"` (string literal) instead of against the `ApprovalDecisionType` enum; matches DB return type.
  - **Should-fix 9 (AC5 envelope test):** Added `test_service_event_payload_deserialises_as_approval_decided` — constructs `ApprovalDecided` from publisher `call_args["payload"]` + `source_service` and asserts all AC5 fields are present.
  - All 39 unit tests and 43 API tests pass (82 total). Linter clean.

## Senior Developer Review

**Verdict:** Changes Requested
**Reviewed:** 2026-04-21
**Scope reviewed:**
- `services/client-api/src/client_api/api/v1/approvals.py`
- `services/client-api/src/client_api/services/approval_decision_service.py`
- `services/client-api/src/client_api/models/approval_decision.py` and `proposal.py` delta
- `services/client-api/src/client_api/schemas/approvals.py`
- `services/client-api/alembic/versions/043_approval_decisions.py`
- `services/client-api/src/client_api/main.py` wiring
- `services/client-api/tests/unit/test_approval_decision_service.py` (38 tests, all green)
- `services/client-api/tests/api/test_approvals.py` (43 tests, all green)

All 81 existing tests pass. The four gates (tenancy, stage-belongs-to-live-workflow, role-on-proposal, prior-stage-approved) are implemented, the workflow-linkage guard is in place, the final-stage marker and idempotent `approved_at` update work, and the post-commit Redis emission correctly skips `returned_for_revision`. However, several specific AC obligations are unmet and there are code-quality issues that should not ship as-is.

### Must-fix (blocking)

1. **AC10 migration-test gap.** The AC explicitly lists seven migration tests; only two exist in `tests/unit/test_approval_decision_service.py`:
    - Present: `test_migration_043_file_exists`, `test_migration_043_revision_chain`.
    - Missing: `test_migration_043_creates_approval_decisions_table`, `test_migration_043_adds_approved_at_to_proposals`, `test_migration_043_stage_id_fk_restrict`, `test_migration_043_proposal_id_fk_cascade`, `test_migration_043_decision_check_constraint`, `test_migration_043_downgrade_reverses`.
    The FK-restrict test in particular is load-bearing — it's the "belt" half of the belt-and-suspenders guard design decision (a). Without a live-DB test that proves a raw `DELETE FROM approval_stages` with decisions present raises `IntegrityError`, the design intent is undocumented in test code. Add these as integration tests (mirror `tests/integration/` patterns, or colocate alongside `test_approval_workflows.py`'s `approval_decisions_table` fixture already in use).

2. **Raw SQL string interpolation in `_get_latest_decisions_per_stage` (`approval_decision_service.py:272-279`).** The query is built by f-string concatenation of a `proposal_id` and stage-id list directly into `text(...)`:
    ```python
    # Use raw string so "DISTINCT ON" appears in mock call_args (AC7)
    ids_str = ", ".join(f"'{i}'" for i in prior_stage_ids)
    sql = text(
        "SELECT DISTINCT ON (stage_id) ... "
        f"WHERE proposal_id = '{proposal_id}' AND stage_id IN ({ids_str}) "
        ...
    )
    ```
    Inputs are currently server-sourced UUIDs (safe at the moment), but:
    - The comment makes explicit that the raw string form exists to satisfy a mock-test assertion (`call_args` substring match). That's a test-driving-implementation anti-pattern — the test should assert behaviour (rows returned, ordering) via a DB round-trip, not substrings of the SQL text.
    - The pattern is dangerous to copy-paste. First future caller that forwards a non-UUID value creates a SQLi. Fix: use bind params (`:pid`, `:sids` with `.bindparams(bindparam("sids", expanding=True))`) and rewrite the unit test that inspects the rendered SQL to inspect the compiled statement's `text` attribute, or better, replace the mock-based test with a DB-backed one.

3. **AC4 non-collaborator contract drift.** AC4's test matrix line says "Non-collaborator user (no row in `proposal_collaborators`) → 404 (handled by `require_proposal_role` dep, not the decision service)." The actual `require_proposal_role` implementation returns **403** in that path (`rbac.py:313-316`), and `tests/api/test_approvals.py::test_decide_non_collaborator_returns_403` asserts 403. Either:
    - Update AC4 to reflect the 403 contract (Story 10.2 precedent), **or**
    - Change `require_proposal_role` to return 404 for this case and update the test.
    Leaving the story AC and the code contradictory is a traceability problem for future stories that reference 10.9's gate semantics.

### Should-fix (non-blocking but recommended before merge)

4. **Router type annotation style (`approvals.py:43-44`).** `session: Annotated[get_db_session, Depends()]` and `redis_client: Annotated[get_redis_client, Depends()]` use a *function* as the type annotation. This works only because FastAPI's `Depends()` with no argument falls back to the annotation as the callable. The idiomatic form used everywhere else in the codebase (`approval_workflows.py`, `proposal_comments.py`, etc.) is `session: AsyncSession = Depends(get_db_session)` or `Annotated[AsyncSession, Depends(get_db_session)]`. Change to match.

5. **Private helper imported across module boundary.** `approvals.py:71` calls `approval_decision_service._maybe_emit_approval_decided(...)`. The leading underscore signals module-private; the router should either invoke a public `emit_approval_decided(...)` function or the service's `submit_decision` should handle emission via the post-commit hook pattern called out in Dev Notes ("queue the event details on `request.state.post_commit_events`"). Current layering is a leak.

6. **Double `MAX(order)` query per decision.** `submit_decision` calls `_is_final_stage()` (which runs `SELECT MAX("order") …`) and then re-issues the identical `select(func.max(ApprovalStage.order))` a few lines later to populate the audit `final_stage` flag (`approval_decision_service.py:135` vs. `145-149`). Hoist `max_order` once, derive both booleans from it.

7. **Unused import.** `from sqlalchemy import desc, func, select, update, text` — `desc` is never used (`approval_decision_service.py:15`). `ruff` will flag this on the next lint pass.

8. **Mixed enum/string comparison.** `latest != ApprovalDecisionType.approved` works because `ApprovalDecisionType` is a `StrEnum`, but the helper's return type annotation is `dict[uuid.UUID, str]`. Either type-annotate the return as `dict[uuid.UUID, ApprovalDecisionType]` and convert inside the helper, or compare against the string literal `"approved"`. Pick one and be explicit.

9. **No `tests/integration/test_redis_event_bus.py`-style contract test for the AC5 envelope round-trip.** `tests/api/test_approvals.py::test_decide_approved_emits_approval_decided_event` does perform `TypeAdapter(ServiceEvent).validate_python(data)`, which covers most of the AC5 contract test obligation. However, the unit-level AC5 test explicitly required in the AC ("Add a unit test that constructs an `ApprovalDecided` model from the emitted envelope") is effectively present as `test_approval_decided_schema_approved_round_trip` / `_rejected_round_trip` at the schema level but does not exercise the actual `_maybe_emit_approval_decided` envelope construction. Minor — could be closed by asserting the publisher `AsyncMock`'s `call_args["payload"]` deserialises via `ApprovalDecided.model_validate(call_args["payload"] | {"event_type": "ApprovalDecided"})`.

### Nits

- `approval_decision_service.py:15` — imports on a single long line; `ruff format` will probably split it.
- `approvals.py:52` uses `correlation_id = req.headers.get("X-Request-ID")`; no fallback to `uuid4()` at this layer. `_maybe_emit_approval_decided` passes `correlation_id` through; the fallback lives inside `EventPublisher`. Fine, but worth a one-line comment so the contract is obvious.
- `ApprovalDecision.__tablename__ = "approval_decisions"` does not set `__table_args__`'s schema via the column-level FK string — fine, but note the FK strings are `"client.proposals.id"` and `"client.approval_stages.id"` literals, not derived from `Proposal.__table__.schema`. Consistent with the rest of the codebase; no action.

### What's good

- Single-stage workflow edge case is tested (`test_decide_single_stage_workflow_immediately_approves_proposal`).
- Idempotency guard on `approved_at` is tested (`test_decide_final_stage_approved_twice_does_not_reset_approved_at`).
- Workflow-linkage guard including the soft-delete interaction (`test_decide_soft_deleted_workflow_blocks_new_decisions_on_its_stages`) matches AC6 exactly.
- Admin-bypass path is exercised end-to-end (`test_decide_admin_bypasses_all_role_checks`).
- Event schema round-trip via `TypeAdapter(ServiceEvent)` proves Story 9.10 compatibility.
- Owner-resolution fallback chain (`created_by` → first `bid_manager` → skip with WARN) has three dedicated tests covering all three branches.
- Router is mounted correctly (`test_router_no_doubled_prefix`) — the AC9 "verify the resulting path is `/api/v1/proposals/{proposal_id}/approvals/decide` (not doubled)" concern is covered.

### Decision

**Changes Requested.** The must-fix items (migration test gap, raw-SQL interpolation, AC-vs-code contract drift on non-collaborator 403/404) are each individually blocking on the stated ACs; none require a re-architecture. Address items 1–3 and either adjust or fix the style items 4–8 at the author's discretion, then resubmit.
