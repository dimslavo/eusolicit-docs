# Story 10.1: Proposal Collaborator CRUD API

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-1-proposal-collaborator-crud-api
- **Points:** 3
- **Type:** backend
- **Module:** Client API (`client_api.api.v1.proposal_collaborators`, `client_api.models.proposal_collaborator`, `client_api.schemas.proposal_collaborators`, `client_api.services.proposal_collaborator_service`, new Alembic migration `035_proposal_collaborators`)
- **Priority:** P0 (Epic 10 foundation — unblocks Stories 10.2 proposal-level RBAC middleware, 10.3 section locking, 10.4 comments, and 10.12 collaborator management UI; pre-E10-exit gate for risk R-006 "Entity-Level RBAC Bypass on Per-Proposal Collaborator Permissions" per `test-design-qa.md`)
- **Depends On:** Story 7.2 (proposal CRUD API, `client.proposals` table + router prefix `/api/v1/proposals`), Story 2.9 (`client.company_memberships` + admin role), Story 2.11 (`client.audit_log` + audit middleware — reused for `granted_by`/`granted_at` trail and `access_denied` denial logs)

## Story

As a **bid_manager on a proposal (or a company admin)**,
I want **to manage which team-members can access a specific proposal and what role each holds (bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only)**,
so that **I can delegate sections of the bid to the right specialists, enforce proposal-level RBAC downstream (Stories 10.2–10.11), and satisfy Epic 10 AC1 "Collaborators can be assigned to a proposal with one of five roles and removed; the collaborator list is queryable per proposal" while preserving the "last bid_manager" safety invariant (a proposal must always have at least one active bid_manager, or be cleanly removed from circulation)**.

## Description

Story 10.1 is the **foundational backend deliverable** of Epic 10. It stands up the `proposal_collaborators` table plus four REST endpoints under `/api/v1/proposals/{proposal_id}/collaborators` and wires in the audit trail, role enforcement, and uniqueness/self-removal invariants. Downstream stories in Epic 10 (Section Locking 10.3, Comments 10.4, Task CRUD 10.5, Approval Decisions 10.9, Bid/No-Bid 10.10, and the UI slice 10.12) all read from this table via the Story 10.2 middleware `require_proposal_role(*allowed_roles)`. This story therefore owns only the **data model + CRUD surface** — it deliberately does NOT ship the `require_proposal_role` dependency (that's 10.2) nor modify any existing proposal endpoint's authorisation (that's also 10.2).

Two non-obvious constraints drive the design:

1. **Proposal-level collaborator roles are a new axis, separate from `CompanyRole` and from `entity_permissions`.** The five Epic 10 roles (`bid_manager`, `technical_writer`, `financial_analyst`, `legal_reviewer`, `read_only`) overlap lexically with `CompanyRole` (which also has `bid_manager`, `read_only`) but they mean different things: `CompanyRole` is a user's ceiling within a company (driven by Story 2.10 RBAC ceilings); `ProposalCollaboratorRole` is a per-proposal contract that a company admin or proposal bid_manager grants. A single user can be a `contributor` at the company level and hold `technical_writer` on Proposal A and `read_only` on Proposal B simultaneously. The existing `entity_permissions` table (Story 2.10) uses a different grant vocabulary (`read`/`write`/`manage`) and is retained unchanged — Story 10.1 introduces a purpose-built `proposal_collaborators` table rather than overloading `entity_permissions`. Story 10.2's middleware will consume `proposal_collaborators`; the generic `check_entity_access` dependency is left alone.

2. **"Last bid_manager" safety invariant.** Epic 10's S10.01 description mandates: "self-removal guard (bid_manager cannot remove themselves if they are the last bid_manager)." This applies to BOTH `DELETE` (remove) and `PATCH` (role change that would demote the last bid_manager). The guard uses a transactional `SELECT ... FOR UPDATE` on the collaborator rows within the same transaction as the write to avoid the classic TOCTOU race where two bid_managers simultaneously try to demote each other. Company admins are NOT subject to this guard — an admin may remove/demote the last bid_manager (administrative intervention). The invariant is: "Any proposal with at least one collaborator row must have at least one row with `role='bid_manager'` UNLESS the mutation is performed by a company admin." Implementation uses a `CHECK` constraint at the app layer (not DB — Postgres cannot express "at least one row where X" on a separate table without a trigger; we use a service-layer transactional check instead, since Story 10.2 will enforce the write-path gate upstream anyway).

A third subtle point: Epic 10 AC1 calls for queryable collaborator lists "with user display names." That requires a JOIN to `client.users` (full_name, email, avatar url). To keep the listing query cheap, we join on `user_id` and project a flat `CollaboratorResponse` shape (`{user_id, user_full_name, user_email, role, granted_by, granted_by_full_name, granted_at}`). Pagination is explicitly NOT required in this story — a proposal is expected to have at most ~20 collaborators in practice, so a single unpaginated list suffices. If a future performance story reveals a hot spot, pagination can be added backward-compatibly via optional `limit`/`offset` query parameters.

Finally, this story **creates the ORM model, routes, and service layer — but does NOT yet modify `services/client-api/src/client_api/api/v1/proposals.py`**. Story 10.2 will insert `require_proposal_role` into the existing proposal endpoints. Story 10.1 only exposes a new sibling router.

## Acceptance Criteria

1. [x] **AC1 — `client.proposal_collaborators` Alembic migration (035).** New migration `035_proposal_collaborators.py` (revision `"035"`, down_revision `"034"`, `branch_labels = None`, `depends_on = None`) creates `client.proposal_collaborators` with columns:
   - `id` UUID primary key, `server_default=sa.text("gen_random_uuid()")`
   - `proposal_id` UUID NOT NULL, `FOREIGN KEY (proposal_id) REFERENCES client.proposals(id) ON DELETE CASCADE`
   - `user_id` UUID NOT NULL, `FOREIGN KEY (user_id) REFERENCES client.users(id) ON DELETE CASCADE`
   - `role` enum `proposal_collaborator_role` (`bid_manager`, `technical_writer`, `financial_analyst`, `legal_reviewer`, `read_only`), NOT NULL
   - `granted_by` UUID NULL (soft reference to `client.users.id`; NULL when the row was seeded by a system actor such as proposal creation; application-layer FK only — no DB FK to avoid CASCADE interaction with deleted granters)
   - `granted_at` TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT `NOW()`
   - `updated_at` TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT `NOW()`, `onupdate=NOW()`
   - UNIQUE constraint `uq_proposal_collaborators_proposal_user` on `(proposal_id, user_id)`
   - INDEX `ix_proposal_collaborators_proposal_id` on `(proposal_id)` (covers list-by-proposal; unique constraint already covers lookups by `(proposal_id, user_id)`)
   - INDEX `ix_proposal_collaborators_user_id` on `(user_id)` (covers Story 10.2's future "what proposals can user X access" query)

   The enum `proposal_collaborator_role` is created via the `DO $$ BEGIN IF NOT EXISTS ... END $$` pattern used by migration 032 (see `032_alert_preferences.py` lines 26–30) so re-running on a partial-apply test DB is idempotent. Schema is `client` on every `op.*` call (project-context rule #3). `downgrade()` drops the table, the two indexes, and the enum type in reverse order.

2. [x] **AC2 — `ProposalCollaboratorRole` enum + `ProposalCollaborator` ORM model.**
   - Add `class ProposalCollaboratorRole(str, Enum)` to `client_api/models/enums.py` with the five members matching AC1. Ordering in the source matches Epic 10's description.
   - Add a new `client_api/models/proposal_collaborator.py` with `class ProposalCollaborator(Base)`:
     - `__tablename__ = "proposal_collaborators"`
     - `__table_args__ = (sa.UniqueConstraint("proposal_id", "user_id", name="uq_proposal_collaborators_proposal_user"), sa.Index("ix_proposal_collaborators_proposal_id", "proposal_id"), sa.Index("ix_proposal_collaborators_user_id", "user_id"), {"schema": "client"})`
     - Mapped columns mirroring AC1. `role: Mapped[ProposalCollaboratorRole] = mapped_column(sa.Enum(ProposalCollaboratorRole, name="proposal_collaborator_role", schema="client"), nullable=False)`.
     - `__from __future__ import annotations__` at top (project-wide pattern).
     - Export from `client_api/models/__init__.py` via `from .proposal_collaborator import ProposalCollaborator` and add `"ProposalCollaborator"` to `__all__`.

3. [x] **AC3 — Pydantic schemas in `client_api/schemas/proposal_collaborators.py`.** Define:
   - `CollaboratorCreateRequest(BaseModel)`: `user_id: UUID`, `role: ProposalCollaboratorRole`.
   - `CollaboratorUpdateRequest(BaseModel)`: `role: ProposalCollaboratorRole` (only mutable field).
   - `CollaboratorResponse(BaseModel)`: `id: UUID`, `proposal_id: UUID`, `user_id: UUID`, `user_full_name: str | None`, `user_email: EmailStr`, `role: ProposalCollaboratorRole`, `granted_by: UUID | None`, `granted_by_full_name: str | None`, `granted_at: datetime`, `updated_at: datetime`. Configured with `model_config = ConfigDict(from_attributes=True)` so `from_attributes` allows conversion from SQLAlchemy result tuples via explicit field projection (since the response requires joined columns not present on `ProposalCollaborator`).
   - `CollaboratorListResponse(BaseModel)`: `items: list[CollaboratorResponse]`, `total: int`. (`total` is the length of `items` for this unpaginated release; future pagination keeps the field stable.)
   - No `CollaboratorDeleteResponse` — `DELETE` returns HTTP 204 with empty body.

4. [x] **AC4 — `POST /api/v1/proposals/{proposal_id}/collaborators` endpoint.** Path: `POST /api/v1/proposals/{proposal_id}/collaborators`. Auth: `Depends(get_current_user)`. Body: `CollaboratorCreateRequest`. Behaviour:
   - Resolve the proposal row (`SELECT ... WHERE id = :proposal_id AND company_id = current_user.company_id`). If not found → HTTP 404 with `detail="Proposal not found"`. NOTE: this cross-company guard returns 404 (NOT 403) to avoid existence leakage (same pattern as Story 9.3 AC8 + Story 7.2).
   - Enforce caller authorisation: caller must be either (a) a company admin (`current_user.role == CompanyRole.admin`) OR (b) already hold `ProposalCollaboratorRole.bid_manager` on THIS proposal (lookup `client.proposal_collaborators` with `(proposal_id, user_id) = (target_proposal.id, current_user.user_id)`). If neither → HTTP 403 with `detail="Only company admins or proposal bid_managers can manage collaborators"`. A denial audit log entry is written via the Story 2.11 audit middleware pattern (`action_type="access_denied"`, `entity_type="proposal_collaborator"`, `entity_id=target_proposal.id`). The `await` on the audit write MUST NOT block the 403 response on a transient failure (reuse `_write_denial_audit` semantics from `core/rbac.py:105`).
   - Validate target user: the `user_id` in the body must (a) exist in `client.users` (`404` with `detail="Target user not found"` if not) AND (b) be an ACTIVE member of the SAME company via `client.company_memberships` (`role IS NOT NULL AND accepted_at IS NOT NULL AND users.is_active = TRUE`). If not a company member → HTTP 422 with `detail="Target user is not an active member of this company"` to preserve tenant isolation (R-006 mitigation: a removed user cannot re-enter via collaborator backdoor).
   - Uniqueness: INSERT with catch on `IntegrityError` → HTTP 409 `detail="User is already a collaborator on this proposal"`. DO NOT silently upsert (this is an explicit "add" endpoint; role changes go via PATCH).
   - On success: insert row with `granted_by = current_user.user_id`, `granted_at = NOW()`. Return HTTP 201 + `CollaboratorResponse` with joined user display fields.
   - Write an audit log row: `action_type="create"`, `entity_type="proposal_collaborator"`, `entity_id=new_collaborator.id`, `after={"proposal_id":..., "user_id":..., "role": role.value}`.
   - Structlog event: `proposal_collaborator.created` at `info` with `proposal_id`, `user_id`, `role`, `granted_by`, `company_id`.

5. [x] **AC5 — `GET /api/v1/proposals/{proposal_id}/collaborators` endpoint.** Path: `GET /api/v1/proposals/{proposal_id}/collaborators`. Auth: `Depends(get_current_user)`. Behaviour:
   - Same proposal resolution + 404-on-cross-company guard as AC4.
   - **Authorisation for list:** any company member who (a) is a company admin, OR (b) already holds ANY `ProposalCollaboratorRole` row on this proposal may list. Non-collaborator non-admin callers receive HTTP 403 `detail="Not authorised to view collaborators"` (denial audit written). Rationale: proposal collaborator identity is sensitive — exposing it to non-collaborators leaks "which bid specialists are engaged on what pursuit."
   - Query: `SELECT proposal_collaborators.*, users.full_name, users.email, granters.full_name AS granted_by_full_name FROM client.proposal_collaborators JOIN client.users ON users.id = proposal_collaborators.user_id LEFT JOIN client.users AS granters ON granters.id = proposal_collaborators.granted_by WHERE proposal_id = :proposal_id ORDER BY proposal_collaborators.granted_at ASC`. Map rows to `CollaboratorResponse` objects via explicit field assignment (NOT `from_attributes=True` shortcut — the joined fields are positional, not ORM-attributed).
   - Return `CollaboratorListResponse(items=..., total=len(items))` with HTTP 200.
   - Structlog event: `proposal_collaborator.listed` at `debug` level only (to avoid log noise on every workspace page load) with `proposal_id`, `count`, `caller_id`.

6. [x] **AC6 — `PATCH /api/v1/proposals/{proposal_id}/collaborators/{user_id}` endpoint.** Path: `PATCH /api/v1/proposals/{proposal_id}/collaborators/{user_id}`. Auth: `Depends(get_current_user)`. Body: `CollaboratorUpdateRequest`. Behaviour:
   - Same proposal-resolution guard + admin-or-bid_manager authorisation as AC4.
   - Load the target row `(proposal_id, user_id)` with `SELECT ... FOR UPDATE` inside the request transaction (prevents TOCTOU for the last-bid_manager guard). If missing → HTTP 404 `detail="Collaborator not found on this proposal"`.
   - **Last-bid_manager guard** (self-demotion): if `(user_id == current_user.user_id)` AND the target row's current `role == bid_manager` AND the incoming `role != bid_manager` AND `current_user.role != CompanyRole.admin` AND the count of `proposal_collaborators WHERE proposal_id = :pid AND role = 'bid_manager'` (taken within the same transaction, AFTER the `FOR UPDATE` on the target row) equals 1 → HTTP 409 `detail="Cannot demote the last bid_manager on this proposal; assign another bid_manager first or have a company admin perform the change"`.
   - Same-role short-circuit: if the incoming `role == existing.role` → HTTP 200 `CollaboratorResponse` with no-op (do NOT bump `updated_at`; do NOT write audit). Return the current row with its joined user fields.
   - On successful role change: UPDATE `role` and `updated_at = NOW()`. Return HTTP 200 with the refreshed `CollaboratorResponse`. Write audit `action_type="update"`, `entity_type="proposal_collaborator"`, `entity_id=row.id`, `before={"role": old_role.value}`, `after={"role": new_role.value}`. Structlog `proposal_collaborator.role_changed` with `proposal_id`, `user_id`, `old_role`, `new_role`, `changed_by`.

7. [x] **AC7 — `DELETE /api/v1/proposals/{proposal_id}/collaborators/{user_id}` endpoint.** Path: `DELETE /api/v1/proposals/{proposal_id}/collaborators/{user_id}`. Auth: `Depends(get_current_user)`. Behaviour:
   - Same proposal-resolution guard + admin-or-bid_manager authorisation as AC4.
   - Load target row with `SELECT ... FOR UPDATE` inside transaction. If missing → HTTP 404.
   - **Last-bid_manager guard** (symmetric to AC6): if target's role is `bid_manager` AND `current_user.role != CompanyRole.admin` AND count of other bid_managers on this proposal (transactional count excluding the target row) is zero → HTTP 409 `detail="Cannot remove the last bid_manager on this proposal; assign another bid_manager first or have a company admin perform the removal"`. The guard applies whether or not the caller is removing *themselves* — removing ANY sole bid_manager without admin privileges is blocked. (Self-removal by a non-sole bid_manager is permitted.)
   - On success: DELETE the row. Return HTTP 204 with empty body. Write audit `action_type="delete"`, `entity_type="proposal_collaborator"`, `entity_id=row.id`, `before={"user_id": user_id, "role": old_role.value}`. Structlog `proposal_collaborator.removed` with `proposal_id`, `user_id`, `old_role`, `removed_by`.

8. [x] **AC8 — Router registration + module structure.**
   - Add a NEW file `services/client-api/src/client_api/api/v1/proposal_collaborators.py` that owns the new sibling router `router = APIRouter(prefix="/proposals/{proposal_id}/collaborators", tags=["proposal-collaborators"])`. The collaborator endpoints live here, NOT in `proposals.py`, to keep Story 7.2's routing file stable and to make Story 10.2's later `require_proposal_role` rewiring a surgical change.
   - Register the router in `services/client-api/src/client_api/api/v1/__init__.py` (or the equivalent central registrar — follow the pattern used by `alert_preferences.py`). Verify via `client.get("/api/v1/proposals/00000000-0000-0000-0000-000000000000/collaborators")` returning 401 (unauth) → 404 (after auth) in the integration suite.
   - Add a small service module `services/client-api/src/client_api/services/proposal_collaborator_service.py` that hosts the query helpers (`load_proposal_or_404`, `assert_caller_is_admin_or_proposal_bid_manager`, `count_bid_managers`, `list_collaborators_with_user`) consumed by the four endpoints. This isolates the transactional guard logic for direct unit-testability without a live router.
   - `__init__.py` exports the service module symbol so Story 10.2 can import the same helpers for `require_proposal_role`.

9. [x] **AC9 — Tenant isolation invariant (R-006 primary mitigation).** A dedicated cross-tenant negative-path test file `tests/api/test_proposal_collaborators_tenant_isolation.py` covers:
   - Company A admin attempts to POST a collaborator on Company B's proposal → HTTP 404 (not 403).
   - Company A bid_manager attempts GET/PATCH/DELETE on Company B's proposal → HTTP 404.
   - `user_id` in POST body belongs to a different company than the proposal → HTTP 422 `detail="Target user is not an active member of this company"`. This closes the "elevation via foreign user" vector.
   - Removing a user from a company's memberships (via the Story 2.9 endpoint) does NOT by itself cascade-delete their `proposal_collaborators` rows — but because `company_memberships` membership is checked on every Story 10.2 `require_proposal_role` call (NOT in this story directly), the removed user is effectively locked out. Story 10.1 documents this in `README.md` under "Lifecycle interaction" but does not implement the cleanup (it's a 10.2/10.11 concern). The test asserts that deleting `company_memberships` row leaves `proposal_collaborators` unchanged (contract, not a bug).

10. [x] **AC10 — Unit + API test coverage.** At minimum:
    - `tests/unit/test_proposal_collaborator_service.py` — 6 unit tests: `load_proposal_or_404_cross_company_returns_none`, `assert_caller_is_admin_or_proposal_bid_manager_admin_ok`, `..._bid_manager_ok`, `..._contributor_denied`, `count_bid_managers_excluding_target`, `list_collaborators_joins_user_and_granter_names`.
    - `tests/api/test_proposal_collaborators_crud.py` — 18+ scenarios covering happy paths (POST 201, GET 200, PATCH 200, DELETE 204), role-enforcement denial (contributor caller 403), uniqueness (duplicate POST 409), target-user-validation (non-company-member 422, missing user 404), last-bid_manager self-demotion (409), last-bid_manager non-self removal (409), admin bypass of last-bid_manager guard (200/204), same-role PATCH no-op (200 without audit), joined user display fields present in responses, cross-company proposal lookup (404), unauthenticated access (401).
    - `tests/api/test_proposal_collaborators_audit.py` — asserts each mutating endpoint writes exactly one `audit_log` row with the correct `action_type`, `entity_type="proposal_collaborator"`, `before`/`after` fields, and denial paths write `action_type="access_denied"`. Reuses the Story 2.11 audit test harness.
    - `tests/api/test_proposal_collaborators_tenant_isolation.py` — per AC9.
    - Target coverage: ≥85% branch coverage on `proposal_collaborator_service.py`, ≥80% on the router module. Assert via `pytest --cov` in CI.

11. [x] **AC11 — Audit trail (Epic 10 AC14 partial).** Every mutation writes exactly one `audit_log` row via the existing Story 2.11 pattern: `action_type ∈ {"create","update","delete","access_denied"}`, `entity_type="proposal_collaborator"`, `entity_id` = row id for mutations / proposal id for denials, `user_id` = caller, `ip_address` from `Request.client.host`, `before`/`after` JSON per-change. NO audit row is written on a listing (GET) or a same-role PATCH no-op. No `audit_log` write is skipped silently on transient DB failures — log at ERROR with `audit.write_failed` and swallow (Story 2.11 convention) so the user-visible operation still succeeds. Document this trade-off in the dev notes.

12. [x] **AC12 — Observability + structured logging.** All six business events emit structlog records with stable event names: `proposal_collaborator.created`, `proposal_collaborator.listed` (debug only), `proposal_collaborator.role_changed`, `proposal_collaborator.removed`, `proposal_collaborator.access_denied`, `proposal_collaborator.last_bid_manager_guarded`. Fields always include `proposal_id`, `user_id` (target), `caller_id`, `company_id` (resolved from the loaded proposal, NOT from the JWT — defence in depth against JWT tampering claims). Correlation id is read from `structlog.contextvars` if bound by the request middleware; fall back to `None`.

13. [x] **AC13 — OpenAPI schema + docs.** All four endpoints have non-empty `summary` and `description` fields matching the AC wording, request/response models declared via Pydantic, explicit `responses={401, 403, 404, 409, 422}` annotations on the POST and PATCH routes. A new entry is added to `services/client-api/README.md` under "Proposal Collaboration (Story 10.1)" summarising the endpoints, the authorisation matrix (admin | proposal-bid_manager | other), the last-bid_manager invariant, and a pointer to Story 10.2 for the `require_proposal_role` downstream consumer.

14. [x] **AC14 — No modification to existing proposal endpoints.** `services/client-api/src/client_api/api/v1/proposals.py` is NOT edited in this story. The router is unchanged. Story 10.2 will later inject `require_proposal_role` into each existing proposal endpoint. A test `tests/unit/test_proposals_router_unchanged_by_s10_1.py` asserts the set of route methods+paths registered under the `/api/v1/proposals` prefix as of Story 7.17 is a strict subset of what's registered after this story (i.e. Story 10.1 only ADDS `/collaborators` sub-routes; does not remove or rename anything). This guards against accidental regressions during the Story 10.2 rewire.

## Design Constraints

- **No modification to `entity_permissions` semantics.** Story 2.10's `check_entity_access` / `entity_permissions` table is unchanged. The new `proposal_collaborators` table is a parallel, proposal-specific authorisation axis. A future story MAY unify them, but this story explicitly keeps them separate to minimise blast radius.
- **No proposal-level RBAC middleware in this story.** `require_proposal_role` is Story 10.2 scope. Story 10.1 enforces authorisation inline in its four endpoints via a single service helper (`assert_caller_is_admin_or_proposal_bid_manager`). Do NOT prematurely extract that helper as a FastAPI `Depends` — that coupling is what Story 10.2 will build.
- **No schema FK on `granted_by`.** `granted_by` is a soft reference to `client.users.id`. Not enforced via DB FK because CASCADE/SET NULL interactions with user deletion (Story 2.9 offers soft-delete) could disrupt the audit trail. Application layer validates the UUID exists at write time only.
- **`proposal_collaborators` is NOT automatically seeded on proposal creation.** Story 7.2's `POST /api/v1/proposals` is not modified. A proposal created before Story 10.2 lands is accessible to admins/bid_managers via the existing ceiling-based guard in `proposals.py`. Once Story 10.2 lands, newly-created proposals will need an initial `bid_manager` row (seeded by Story 10.2's modified `POST /proposals` or by a one-off migration). Document this migration path in the README but do NOT implement it here.
- **Uniqueness via DB constraint + app-level catch.** The unique constraint on `(proposal_id, user_id)` guarantees consistency under concurrent inserts. The router catches `IntegrityError` specifically on `uq_proposal_collaborators_proposal_user` (inspect `orig.diag.constraint_name`) and returns 409. Other `IntegrityError`s bubble to the global 500 handler unchanged.
- **Last-bid_manager guard is transactional, not DB-declarative.** We do NOT add a Postgres trigger or deferrable constraint to enforce "at least one bid_manager." Triggers are hard to observe, hard to test, and the admin-bypass exception complicates the rule. Instead, the service layer runs `SELECT ... FOR UPDATE` on the target row + `SELECT count(*) WHERE role = 'bid_manager'` inside the same transaction before accepting the write. Both queries use the row-level lock so concurrent demotions on the same proposal serialize. Acknowledged cost: two reads per mutation. Benefit: all logic lives in Python, easy to test, and admin bypass is one-line.
- **Pagination deliberately deferred.** Proposals are expected to have ≤20 collaborators. If a production telemetry spike shows collaborator lists >100, add optional `limit`/`offset` query params and an `ORDER BY granted_at ASC` stable sort — the current query already does that.
- **No direct cross-schema reads.** `client.users` is in the same schema as `client.proposal_collaborators` — no cross-schema grants needed. Validate via `make lint` that the ORM's foreign key target matches.
- **`PATCH`-only for role change.** We deliberately do NOT accept `PUT` (would imply replace-semantics including immutable fields). `PATCH` with a single `role` field keeps the API surface minimal.

## Tasks / Subtasks

- [x] **Task 1: Create Alembic migration 035 and ORM model. (AC1, AC2)**
  - [x] Create `services/client-api/alembic/versions/035_proposal_collaborators.py`. Revision `"035"`, down_revision `"034"`. Implement `upgrade()` per AC1: create enum type (idempotent `DO $$ ... END $$` pattern), create table with all columns + constraints + indexes. Implement `downgrade()` in reverse.
  - [x] Verify migration runs cleanly on a fresh Postgres testcontainer: `alembic upgrade head` then `alembic downgrade 034` then `alembic upgrade head`. No errors.
  - [x] Add `ProposalCollaboratorRole` to `client_api/models/enums.py` (alphabetical position after `EntityPermission`).
  - [x] Create `client_api/models/proposal_collaborator.py` with the `ProposalCollaborator` ORM model.
  - [x] Wire export in `client_api/models/__init__.py` (add import + `__all__` entry alphabetised).
  - [x] Unit test `tests/unit/test_proposal_collaborator_model.py`: asserts `ProposalCollaborator.__tablename__`, schema, constraints (by name), and enum round-trip (insert `bid_manager` → read back matches).

- [x] **Task 2: Pydantic schemas. (AC3)**
  - [x] Create `client_api/schemas/proposal_collaborators.py` with `CollaboratorCreateRequest`, `CollaboratorUpdateRequest`, `CollaboratorResponse`, `CollaboratorListResponse`. Import `ProposalCollaboratorRole` from `client_api.models.enums` to keep the wire enum literally identical to the DB enum.
  - [x] `CollaboratorResponse.model_config = ConfigDict(from_attributes=True, extra="forbid")`.
  - [x] Unit test `tests/unit/test_proposal_collaborator_schemas.py`: round-trip a `CollaboratorResponse` from a dict; reject unknown fields (`extra="forbid"`); reject invalid role strings on create.

- [x] **Task 3: Service layer. (AC4–AC7 logic extraction, AC8 helper hosting)**
  - [x] Create `client_api/services/proposal_collaborator_service.py`. Implement async helpers:
    - `async def load_proposal_or_404(session, proposal_id: UUID, company_id: UUID) -> Proposal` — raises `HTTPException(404)` with `detail="Proposal not found"` if missing or cross-company.
    - `async def assert_caller_is_admin_or_proposal_bid_manager(session, current_user: CurrentUser, proposal_id: UUID) -> None` — admin bypass OR lookup `proposal_collaborators` for `(proposal_id, current_user.user_id, role='bid_manager')`; raises `HTTPException(403)` with `detail="Only company admins or proposal bid_managers can manage collaborators"` otherwise.
    - `async def assert_caller_is_admin_or_collaborator(session, current_user: CurrentUser, proposal_id: UUID) -> None` — admin bypass OR lookup any `proposal_collaborators` row for `(proposal_id, current_user.user_id)`; raises 403 `detail="Not authorised to view collaborators"` otherwise.
    - `async def validate_target_user_is_company_member(session, user_id: UUID, company_id: UUID) -> User` — returns the `User` or raises 404 `detail="Target user not found"` / 422 `detail="Target user is not an active member of this company"`.
    - `async def count_bid_managers(session, proposal_id: UUID, *, excluding_user_id: UUID | None = None) -> int` — transactional count (relies on caller's enclosing `SELECT ... FOR UPDATE` for serialisation).
    - `async def list_collaborators_with_user(session, proposal_id: UUID) -> list[CollaboratorResponse]` — performs the JOIN-to-users query per AC5.
  - [x] Unit tests (Task 10's file) cover each helper independently with mocked `AsyncSession`.

- [x] **Task 4: POST endpoint. (AC4, AC11, AC12)**
  - [x] Create `client_api/api/v1/proposal_collaborators.py`. Declare `router = APIRouter(prefix="/proposals/{proposal_id}/collaborators", tags=["proposal-collaborators"])`.
  - [x] `@router.post("", status_code=201, response_model=CollaboratorResponse, responses={401: ..., 403: ..., 404: ..., 409: ..., 422: ...})` — wire sequentially: `load_proposal_or_404` → `assert_caller_is_admin_or_proposal_bid_manager` → `validate_target_user_is_company_member` → INSERT with `IntegrityError` catch → audit write → structlog emit → return response.
  - [x] Wrap the INSERT + audit in the request-scoped session (single transaction). If the INSERT raises, rollback covers the audit write (which was only staged, not committed). The denial-path audit uses `_write_denial_audit` pattern (separate session, independent commit).
  - [x] Structlog event `proposal_collaborator.created` with `proposal_id`, `user_id`, `role`, `granted_by`, `company_id`.

- [x] **Task 5: GET endpoint. (AC5, AC12)**
  - [x] `@router.get("", response_model=CollaboratorListResponse, responses={401, 403, 404})` — `load_proposal_or_404` → `assert_caller_is_admin_or_collaborator` → `list_collaborators_with_user` → wrap in `CollaboratorListResponse(items=..., total=len(items))`.
  - [x] Structlog `proposal_collaborator.listed` at debug.

- [x] **Task 6: PATCH endpoint with last-bid_manager guard. (AC6, AC11, AC12)**
  - [x] `@router.patch("/{user_id}", response_model=CollaboratorResponse, responses={401, 403, 404, 409})` — `load_proposal_or_404` → `assert_caller_is_admin_or_proposal_bid_manager` → open explicit `async with session.begin_nested()` → `SELECT ... FOR UPDATE` on the target row → same-role short-circuit OR last-bid_manager check OR UPDATE + audit → commit → return refreshed response.
  - [x] Same-role short-circuit returns 200 without audit write, without `updated_at` bump.
  - [x] Structlog `proposal_collaborator.role_changed` with `old_role`, `new_role`.
  - [x] Structlog `proposal_collaborator.last_bid_manager_guarded` on 409 with `caller_id`, `proposal_id`.

- [x] **Task 7: DELETE endpoint with last-bid_manager guard. (AC7, AC11, AC12)**
  - [x] `@router.delete("/{user_id}", status_code=204, responses={401, 403, 404, 409})` — same scaffolding as Task 6. Last-bid_manager guard counts `role='bid_manager' AND user_id != target_user_id`; if zero AND target's role is `bid_manager` AND caller is not admin → 409.
  - [x] Structlog `proposal_collaborator.removed` on success. Return `Response(status_code=204)` (FastAPI).

- [x] **Task 8: Router registration + OpenAPI metadata. (AC8, AC13)**
  - [x] Open `services/client-api/src/client_api/api/v1/__init__.py` (or wherever the router registration lives — follow the `alert_preferences` pattern). Append `router.include_router(proposal_collaborators_router, prefix="/api/v1")`. Verify no prefix duplication (the sub-router already declares its own `prefix="/proposals/{proposal_id}/collaborators"`).
  - [x] Add each endpoint's `summary`, `description`, and `responses={...}` per AC13.
  - [x] Extend `services/client-api/README.md` with the new "Proposal Collaboration (Story 10.1)" section covering endpoint matrix, authorisation table, last-bid_manager invariant, and forward-pointer to Story 10.2.

- [x] **Task 9: Unit + API + tenant-isolation tests. (AC10, AC9, AC14)**
  - [x] Create `tests/unit/test_proposal_collaborator_service.py`: helper-by-helper unit tests per AC10.
  - [x] Create `tests/api/test_proposal_collaborators_crud.py`: 18+ scenarios per AC10 using httpx `AsyncClient`.
  - [x] Create `tests/api/test_proposal_collaborators_audit.py`: audit-row assertions per AC11 reusing the Story 2.11 fixtures.
  - [x] Create `tests/api/test_proposal_collaborators_tenant_isolation.py`: cross-company 404s + cross-company-user 422 per AC9.
  - [x] Create `tests/unit/test_proposals_router_unchanged_by_s10_1.py`: route-stability guard per AC14 (snapshot the pre-10.1 set of `(method, path)` tuples registered under the `/api/v1/proposals` prefix and assert `post-10.1 ⊇ pre-10.1` strictly).
  - [x] Coverage target ≥85% on service module, ≥80% on router.

- [x] **Task 10: Integration test — end-to-end collaborator lifecycle. (AC10)**
  - [x] Create `tests/integration/test_proposal_collaborator_lifecycle.py` with `@pytest.mark.integration`. Spin up Postgres testcontainer; run migrations up to 035. Seed two companies (A, B) each with two users (A1 admin, A2 contributor; B1 admin, B2 contributor) and one proposal per company.
  - [x] Scenario 1: A1 (admin) POSTs A2 as `bid_manager` on proposal-A → expect 201. A2 POSTs their own self again → 409.
  - [x] Scenario 2: A2 (now bid_manager on proposal-A) POSTs B1 as collaborator on proposal-A → expect 422 (cross-company user). POST B1 on proposal-B while authed as A1 → expect 404 (cross-company proposal).
  - [x] Scenario 3: A1 creates A2 as `bid_manager`; A2 PATCHes self from `bid_manager` → `read_only` → expect 409 (last-bid_manager). A1 (admin) PATCHes A2 down → expect 200. A1 PATCHes A2 back up to `bid_manager`. A2 adds a third user as `bid_manager`; then A2 demotes self → expect 200.
  - [x] Scenario 4: A1 DELETEs the last bid_manager (non-admin context: retry the same op under A2 after A1 resigns) → 409. A1 DELETEs themselves but is admin → 204 (admin bypass).
  - [x] Scenario 5: verify `audit_log` contains one row per mutation + one per denial with correct `action_type` and `entity_type="proposal_collaborator"`.

- [x] **Task 11: Lint + type-check + CI alignment.**
  - [x] `ruff check services/client-api/src/client_api/api/v1/proposal_collaborators.py services/client-api/src/client_api/models/proposal_collaborator.py services/client-api/src/client_api/services/proposal_collaborator_service.py services/client-api/src/client_api/schemas/proposal_collaborators.py services/client-api/alembic/versions/035_proposal_collaborators.py services/client-api/tests/unit/test_proposal_collaborator_*.py services/client-api/tests/api/test_proposal_collaborators_*.py` — clean.
  - [x] `ruff format --check` — clean.
  - [x] `mypy` (project default config) — clean on all new files.
  - [x] Verify the `make test-client-api` target (or equivalent from Story 9.3's CI) includes the new test files.

### Review Follow-ups (AI)

_Source: Senior Developer Review on 2026-04-24 (outcome: Changes Requested). Items below address the required and recommended fixes from that review._

- [x] **[AI-Review][High] #1** Replace tautological `"key" in before or "key" in after` disjunctions in `tests/api/test_proposal_collaborators_audit.py` (POST/PATCH/DELETE) with strict equality on the mutation payload — assert `after`/`before` dicts include the expected keys and enum `.value` strings literally.
- [x] **[AI-Review][High] #2** Add the AC14-specified test `tests/unit/test_proposals_router_unchanged_by_s10_1.py` that snapshots the pre-10.1 `(method, path)` set registered under `/api/v1/proposals` and asserts `post-10.1 ⊇ pre-10.1`.
- [x] **[AI-Review][High] #3** Wrap `tests/integration/test_proposal_collaborator_lifecycle.py` `lifecycle_ctx` in explicit cleanup (DELETE seeded rows in `finally:`) so the integration DB is not polluted across runs.
- [x] **[AI-Review][Med] #4** Add the AC6 self-demotion precondition (`user_id == current_user.user_id`) to `update_collaborator_role` to match AC6 wording literally.
- [x] **[AI-Review][Med] #5** Remove the dead commented-out `# select(ProposalCollaborator) …` block and stray blank line in `update_collaborator_role`.
- [x] **[AI-Review][Med] #6** Scope the 403-denial audit count in `test_403_denial_writes_access_denied_audit_row` to `entity_id = proposal_id` so prior tests' 403 rows cannot satisfy the delta.
- [x] **[AI-Review][Med] #7** Seed at least one collaborator row before calling GET in `test_get_collaborators_writes_no_audit_row`, so the assertion covers the populated-list path, not only empty.
- [x] **[AI-Review][Low] #8** Replace POST/PATCH post-mutation `list_collaborators_with_user` + linear scan with a targeted single-row joined hydrator.
- [x] **[AI-Review][Low] #9** Replace brittle `"uq_proposal_collaborators_proposal_user" in str(exc.orig)` with structured constraint-name detection via `getattr(exc.orig, "diag", None)` / `exc.orig.constraint_name`.
- [x] **[AI-Review][Low] #10** Add a docstring on migration 035 explaining the deliberate use of `op.execute()` over SQLAlchemy autogen for enum + table creation.
- [x] **[AI-Review][Low] #11** Replace `if excluding_user_id:` with `if excluding_user_id is not None:` in `count_bid_managers`.

## Dev Agent Record

- **Implemented by:** gemini-2.0-flash-exp + session-cbac67c2-30c7-473e-9614-757bb7375a98 + 15m
- **Review follow-ups addressed by:** claude-sonnet-4-5 + dev-story session on 2026-04-24 + ~25m
- **File List:**
  - **New:**
    - `services/client-api/tests/unit/test_proposals_router_unchanged_by_s10_1.py` (AC14 route-stability snapshot)
  - **Modified:**
    - `services/client-api/alembic/versions/025_audit_hardening.py` (Shortened revision ID)
    - `services/client-api/alembic/versions/026_stripe_customer_id.py` (Updated down_revision)
    - `services/client-api/alembic/versions/027_usage_metering_middleware.py` (Updated down_revision)
    - `services/client-api/alembic/versions/028_opportunity_listing_filters.py` (Updated down_revision)
    - `services/client-api/alembic/versions/029_document_upload_api.py` (Updated down_revision)
    - `services/client-api/alembic/versions/030_proposal_version_db_schema.py` (Updated down_revision)
    - `services/client-api/alembic/versions/031_proposal_crud_api.py` (Updated down_revision)
    - `services/client-api/alembic/versions/032_alert_preferences.py` (Updated down_revision)
    - `services/client-api/alembic/versions/033_ical_feed_generation.py` (Updated down_revision)
    - `services/client-api/alembic/versions/034_scheduled_report_generation.py` (Updated down_revision)
    - `services/client-api/alembic/versions/035_proposal_collaborators.py` (Fix #10 rationale docstring + lint fixes)
    - `services/client-api/src/client_api/api/v1/proposal_collaborators.py` (Fix #4 self-demotion precondition, Fix #5 dead-code removal, Fix #8 single-row response hydrator, Fix #9 structured IntegrityError constraint detection)
    - `services/client-api/src/client_api/services/proposal_collaborator_service.py` (Fix #8 `get_collaborator_response` hydrator, Fix #11 `is not None` check, Fix #13 docstring note on `seed_creator_as_bid_manager`)
    - `services/client-api/tests/api/test_proposal_collaborators_crud.py` (Made seeding idempotent — earlier pass)
    - `services/client-api/tests/api/test_proposal_collaborators_audit.py` (Fix #1 strict `before`/`after` equality on POST/PATCH/DELETE; Fix #6 denial scoped to proposal entity_id; Fix #7 GET no-audit seeded collaborator)
    - `services/client-api/tests/integration/test_proposal_collaborator_lifecycle.py` (Fix #3 track + CASCADE-clean seeded rows in `finally:`; scenario 3 pre-delete auto-seeded creator row to match spec)
- **Test Results:** `================== 76 passed, 7 warnings in 65.88s (0:01:05) ===================`

  The 76-test suite covers: 8 unit (model/schemas/service + route-stability snapshot), 34 API (CRUD/audit/tenant-isolation, including the 11 new strict-equality assertions), 5 integration lifecycle scenarios (now all passing end-to-end with cleanup), and all adjacent Story 10.1 tests.

- **Known Deviations:**
  - **Bug Fix (Alembic):** The revision ID `025_audit_log_company_correlation` exceeded the 32-character limit of the `alembic_version` table. Shortened to `025_audit_hardening` and updated all downstream migrations. This was a blocker for running tests on a fresh DB. _(pre-existing)_
  - **Bug Fix (API):** Fixed a `NameError` in `remove_collaborator` where `bid_manager_count` was used instead of `other_bid_manager_count`. _(pre-existing)_
  - **Environment Remediation:** The `eusolicit_test` database was in a broken/partial state — reset, recreated schemas, and granted proper ownership/permissions to `migration_role` and `client_api_role`. _(pre-existing)_
  - **Test scenario adjustment (Scenario 3):** The integration lifecycle test's Scenario 3 was relying on an implicit "only one bid_manager on the proposal" premise, but `POST /api/v1/proposals` already invokes `seed_creator_as_bid_manager` (a Story 10.2 hook that is pre-landed in this service module). Scenario 3 now explicitly clears the proposal's collaborator rows before Step A, matching the story-spec wording and keeping the guard exercised correctly. Not a production-code change — a test-fixture alignment.
  - **Pre-existing test failure outside this story's scope:** `tests/unit/test_proposals_router_dependency_spec.py` (a Story 10.2 artifact) has a stale `_EXPECTED_DEPENDENCIES` snapshot that does not include the approval/preparation-log routes added by Stories 10.9/10.11. That failure existed before this review-follow-up session, was not touched, and is unrelated to Story 10.1's acceptance surface. To be addressed under Story 10.2.

### Review Follow-up Resolutions (2026-04-24)

Every action item from the Senior Developer Review has a matching test, code diff, or documented rationale:

- ✅ Resolved review finding [High] #1: audit-row assertions now assert exact dicts with enum `.value` strings on POST/PATCH/DELETE; `before`/`after` emptiness is also asserted where applicable.
- ✅ Resolved review finding [High] #2: `tests/unit/test_proposals_router_unchanged_by_s10_1.py` implements the AC14 superset invariant against a frozen pre-10.1 snapshot.
- ✅ Resolved review finding [High] #3: lifecycle fixture now tracks every seeded user/company/proposal and tears them down in `finally:` via CASCADE; audit rows tagged to those users are deleted in the same block.
- ✅ Resolved review finding [Med] #4: PATCH guard now includes the `user_id == current_user.user_id` precondition verbatim from AC6.
- ✅ Resolved review finding [Med] #5: dead commented-out `select(ProposalCollaborator)…` block and stray blank line in DELETE handler removed.
- ✅ Resolved review finding [Med] #6: denial-audit assertion now counts only rows with `entity_id = <this proposal_id>`, making the delta meaningful across concurrently-seeded test state.
- ✅ Resolved review finding [Med] #7: `test_get_collaborators_writes_no_audit_row` now seeds a collaborator first so the populated-list GET path is exercised.
- ✅ Resolved review finding [Low] #8: `get_collaborator_response(session, proposal_id, user_id)` performs a single-row targeted JOIN; POST and PATCH now call it instead of `list_collaborators_with_user(…)` + linear scan.
- ✅ Resolved review finding [Low] #9: POST's `IntegrityError` handler prefers `exc.orig.diag.constraint_name` / `exc.orig.constraint_name` before falling back to `str()` matching.
- ✅ Resolved review finding [Low] #10: migration 035 now carries an explicit module-level rationale for using raw `op.execute` over `op.create_table`.
- ✅ Resolved review finding [Low] #11: `count_bid_managers` uses `if excluding_user_id is not None:`.
- ✅ Resolved review finding [Low] #13: `seed_creator_as_bid_manager` docstring clarifies it's a pre-landed Story 10.2 helper, not part of Story 10.1's endpoints.

## Change Log

- 2026-04-20: Initial story context created. (bmad-create-story)
- 2026-04-24: Verification and environmental fixes completed. Story marked for review.
- 2026-04-24: Senior Developer Review appended — outcome Changes Requested. (bmad-code-review)
- 2026-04-24: Addressed code review findings — 12 items resolved (3 High, 4 Med, 5 Low); AC14 route-stability test added; integration lifecycle fixture now cleans up seeded state via CASCADE; all 76 Story 10.1 tests pass. (bmad-dev-story)
- 2026-04-24: Follow-up Senior Developer Review — outcome **Approve**. All 12 findings from the prior review verified as resolved in code (strict audit-row equality, AC14 route-stability snapshot, integration fixture CASCADE cleanup, self-demotion precondition, dead-code removal, scoped denial-audit count, populated-list GET no-audit path, single-row response hydrator, structured IntegrityError constraint detection, migration rationale docstring, `is not None` guard). 76 tests passing. No new blocking or non-blocking issues found. Ready for QA. (bmad-code-review)

## Senior Developer Review

**Reviewer:** Senior Dev (bmad-code-review, adversarial parallel layers)
**Date:** 2026-04-24
**Outcome:** **Changes Requested**

The implementation satisfies the main functional shape of ACs 1–13 and 71 tests pass. However, several defects around test *fidelity*, an AC14 gap, and a small set of code-quality issues warrant fixes before QA. Nothing here blocks the Story 10.2 downstream work from starting, but these should close before the epic exits.

### High

1. **Tautological audit-row assertions — `tests/api/test_proposal_collaborators_audit.py`.**
   Lines 252–253 (POST), ~304 (PATCH), ~386 (DELETE) use `assert "role" in after or "proposal_id" in after` / equivalent disjunctions. The disjunction passes even when `after` is missing the expected mutation payload. Consequence: the audit path could silently stop writing `role` (or write `before`/`after` with the wrong keys) and these tests would still pass. AC11 explicitly requires per-mutation `before`/`after` keys — the tests must assert the *exact* keys and the changed values.
   *Fix:* Replace each disjunctive assertion with strict equality on `after` (and `before` on PATCH/DELETE) against a literal dict including the enum `.value` strings.

2. **AC14 route-stability test is missing as specified.**
   AC14 and Task 9 call for `tests/unit/test_proposals_router_unchanged_by_s10_1.py` that snapshots the pre-10.1 `(method, path)` set under `/api/v1/proposals` and asserts post-10.1 ⊇ pre-10.1. What exists is `test_proposals_router_dependency_spec.py` (a Story 10.2 artifact) which *explicitly skips* `/collaborators` routes (line 138) and tests dependency tags instead of route-set stability. No test currently guards against a regression that silently renames/removes an existing proposal endpoint during Story 10.2's rewire.
   *Fix:* Add the AC14-named test with a hardcoded pre-10.1 snapshot (can be extracted from the current `_EXPECTED_DEPENDENCIES.keys()`) and a superset assertion.

3. **Integration fixture deliberately skips DB rollback — violates CLAUDE.md gold-standard isolation.**
   `tests/integration/test_proposal_collaborator_lifecycle.py` `lifecycle_ctx` (lines ~40–170) states in-source "does NOT roll back" and the teardown only clears dependency overrides. Users/proposals/collaborators created during the suite persist in the integration DB. This is a direct deviation from the project rule "never commit in tests" / per-test transaction rollback. UUID randomness covers single-run uniqueness but leaves state for subsequent runs and leaks audit rows that can pollute other suites' `COUNT(*)` assertions (see Medium #6).
   *Fix:* Wrap the fixture in the `db_session` transaction-rollback pattern used across the other integration tests, or explicitly clean up inserted rows in `finally:`.

### Medium

4. **Last-bid_manager guard wording vs. AC6 scope.**
   AC6 scopes the PATCH guard to *self-demotion* (`user_id == current_user.user_id`). The implementation in `update_collaborator_role` applies the guard to any demotion by a non-admin caller. In practice the valid caller set (admin OR a proposal bid_manager) means a non-self demotion path will have `count >= 2` and won't trip the guard, so functionally the outcome matches AC6. Still a spec-wording drift that should either be made faithful or the AC reworded.
   *Fix:* Either add the `user_id == current_user.user_id` guard precondition to match AC6, or update AC6 to reflect the broader but logically equivalent check.

5. **Dead commented-out alternate implementation in `update_collaborator_role`** (`api/v1/proposal_collaborators.py` lines 200–205, stray blank line at 363–364). Code-quality noise; remove.

6. **403-denial audit test is pollution-sensitive** (`test_proposal_collaborators_audit.py` ~line 412): `count_after >= count_before + 1` will pass even if the assertion was meant to validate *this* endpoint's denial, because any prior test's 403 also bumps the count. Filter by `entity_type="proposal_collaborator" AND entity_id=<proposal_id>`.

7. **Same-role no-op and GET audit tests could be stronger.**
   - `TestSameRolePatch` (CRUD file) correctly checks delta, good.
   - `test_get_collaborators_writes_no_audit_row` runs GET on an empty proposal; it never exercises the "GET over populated collaborators does NOT audit" case. AC11 requires no audit on list — strengthen by seeding rows first, then asserting zero audits.

8. **Inefficient post-mutation response construction.** POST / PATCH / DELETE (for response consistency) call `list_collaborators_with_user(session, proposal_id)` and linear-scan for the target user — i.e. N+1 JOIN per write for a single row. Swap to a single-row targeted variant of the join query, or hydrate the response directly from the ORM row + one `User` lookup.

9. **Brittle IntegrityError constraint detection.** `"uq_proposal_collaborators_proposal_user" in str(exc.orig)` (POST handler, line ~85) is string-match fragile across DB driver versions. Prefer `getattr(exc.orig, "diag", None)` / `exc.orig.constraint_name` (psycopg3) or `isinstance(exc.orig, asyncpg.UniqueViolationError)`.

10. **Migration deviates from AC1 `op.*` idiom.** AC1 specifies `op.create_table(...)` with `schema="client"` on every call. The implementation uses `op.execute()` with raw SQL "to avoid SQLAlchemy visit_enum magic." Works, but loses Alembic's autogenerate diffing and forfeits type/column introspection. Acceptable workaround — flag in the migration docstring as a deliberate pattern choice and document the motivation.

11. **`count_bid_managers` uses truthiness check** (`if excluding_user_id:`) where `is not None` is clearer and safer against nil-UUID edge cases.

### Low

12. **Out-of-scope chain repair in migrations 025–034.** Dev record documents shortening `025_audit_log_company_correlation` → `025_audit_hardening` and re-parenting migrations 026–034. Valid infra bug fix, but it was rolled into a story that AC-wise only touches migration 035. Document in the story's "Known Deviations" (already done) and, if possible, land as a pre-commit under a separate housekeeping story for traceability.
    - **DEVIATION:** Migration chain repair (025 rename + 026–034 down_revision rewrites) performed inside a story scoped to migration 035.
    - **DEVIATION_TYPE:** SCOPE_CREEP
    - **DEVIATION_SEVERITY:** deferrable

13. **`seed_creator_as_bid_manager` belongs to Story 10.2.** Helper added in this story but unused by the four S10.1 endpoints. Fine to pre-land for 10.2, but call that out explicitly in a docstring linking to S10.2.

### What's Good

- Proposal resolution returns 404 cross-company (no existence leak) — R-006 mitigation present.
- `SELECT ... FOR UPDATE` + same-transaction count for the last-bid_manager guard is the right choice given the admin-bypass exception.
- Uniqueness constraint + IntegrityError catch is correct; no silent upsert.
- Denial-path audit writes use `_write_denial_audit` (dedicated session) correctly — rollback on 403 does not destroy the denial trail.
- Pydantic response model uses `extra="forbid"` — schema-drift safety belt.
- Router registered in `main.py` line 134, verified.

### Required Actions Before QA

- [x] **Must:** Tighten audit-row assertions (fix #1). _(resolved 2026-04-24)_
- [x] **Must:** Add AC14 route-stability test (fix #2). _(resolved 2026-04-24)_
- [x] **Must:** Rollback state in the integration lifecycle fixture (fix #3). _(resolved 2026-04-24 via tracked-cleanup finally block)_
- [x] **Should:** Address #4 (AC6 wording vs. implementation) — chose "add the precondition". _(resolved 2026-04-24)_
- [x] **Should:** Remove dead comments (#5), strengthen 403-denial filter (#6), strengthen GET no-audit test (#7). _(all resolved 2026-04-24)_
- [x] **Nice-to-have:** #8 (efficiency), #9 (IntegrityError parsing), #10 (migration docstring), #11 (`is not None`). _(all resolved 2026-04-24)_

DEVIATION: Migration chain repair (025 rename + 026–034 down_revision rewrites) performed inside a story scoped to migration 035.
DEVIATION_TYPE: SCOPE_CREEP
DEVIATION_SEVERITY: deferrable

DEVIATION: AC14 explicitly called for `tests/unit/test_proposals_router_unchanged_by_s10_1.py`; the existing `test_proposals_router_dependency_spec.py` (10.2 artifact) skips collaborator routes and tests dependency tags instead of the superset invariant.
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable

DEVIATION: AC11 audit-row shape assertions in `test_proposal_collaborators_audit.py` use tautological disjunctions that pass regardless of `before`/`after` correctness.
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable

FAILURE_REASON: N/A — review completed, outcome Changes Requested (not a phase failure).

## Known Deviations

### Detected by `3-code-review` at 2026-04-24T15:53:01Z (session fb02ceea-5be6-4b6a-ae7d-e40c3f785f54)

- Migration chain repair (025 rename + 026–034 down_revision rewrites) performed inside a story scoped to migration 035. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- AC14 route-stability test not implemented as specified; the existing dependency-spec test skips collaborator routes. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- AC11 audit-row assertions tautological; would pass with wrong audit payloads. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- Migration chain repair (025 rename + 026–034 down_revision rewrites) performed inside a story scoped to migration 035. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- AC14 route-stability test not implemented as specified; the existing dependency-spec test skips collaborator routes. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- AC11 audit-row assertions tautological; would pass with wrong audit payloads. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
