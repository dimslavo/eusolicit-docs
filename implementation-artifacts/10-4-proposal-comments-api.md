# Story 10.4: Proposal Comments API

Status: complete

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-4-proposal-comments-api
- **Points:** 3
- **Type:** backend
- **Module:** Client API (new router `client_api.api.v1.proposal_comments`, new service `client_api.services.proposal_comment_service`, new ORM `client_api.models.proposal_comment`, new Pydantic schemas `client_api.schemas.proposal_comments`, new Alembic migration `038_proposal_comments`, hook into existing `client_api.services.proposal_service.create_version` for unresolved-comment carry-forward)
- **Priority:** P1 (Epic 10 AC3 "Comments can be created on a section+version, listed per section, resolved/unresolved, and unresolved comments carry forward when a new version is created" — primary feature for §10.13 Comments Sidebar UI; second story to write per-section state on top of the Story 10.3 lock model; pre-requisite for Story 10.13 frontend which depends on the listing+resolve API surface)
- **Depends On:** Story 10.1 (`client.proposal_collaborators` table + `ProposalCollaboratorRole` enum), Story 10.2 (`require_proposal_role(*allowed_roles)` factory + `PROPOSAL_WRITE_ROLES` + `PROPOSAL_ALL_ROLES` + creator-seed + auto-seed-conftest pattern), Story 7.2 (`client.proposals` table + `current_version_id` pointer), Story 7.3 (`create_version` in `proposal_service.py:484` — carry-forward hook point), Story 7.4 (`SectionItem` schema + `JSONResponse` conflict-body convention used as response-body style template), Story 10.3 (per-section state precedent + per-version state hook into version creation), Story 2.11 (`audit_log` + `write_audit_entry`)

## Story

As **a collaborator with write permission on a proposal (bid_manager, technical_writer, financial_analyst, or legal_reviewer) reviewing a teammate's draft section**,
I want **to leave a threaded comment on a specific section of the current proposal version, see all comments per section listed in chronological order, mark them resolved or unresolved as the conversation progresses, and have any unresolved comments automatically carry forward when a new version snapshot is created**,
so that **review feedback is anchored to a specific (section, version) tuple rather than floating, the team can converge on what is open vs. closed, Epic 10 AC3 is satisfied (create, list, resolve toggle, version carry-forward), and Story 10.13's Comments Sidebar UI has the API surface it needs (filtered list per section, optimistic resolve toggle, "carried forward" banner trigger)**.

## Description

Story 10.4 introduces threaded review comments anchored to a `(proposal_id, version_id, section_key)` tuple. This is the second Epic 10 story (after 10.3) that writes per-section state, so the design reuses 10.3's free-form `section_key` convention and the Story 7.4 `JSONResponse` body pattern, but it differs from 10.3 in three structural ways: (a) comments are immutable in body — only the `resolved` flag can toggle; (b) comments are scoped to a specific `version_id` rather than to "the current version" — historical comments on prior versions remain queryable; (c) Story 10.4 must hook into `proposal_service.create_version` to carry forward unresolved comments to the new version, which is the first cross-story modification of an existing service in Epic 10 (10.1, 10.2, 10.3 were all purely additive on the read-write paths they touched).

Five non-obvious design decisions drive the implementation:

**(a) Comments are anchored to `version_id`, not to "current version".** Each comment row carries a NOT NULL `version_id` FK to `client.proposal_versions(id)`. The list endpoint requires a `version_id` query parameter and does NOT default to "current". Rationale: a comment posted on version 3 and resolved on version 3 must NOT reappear when version 4 is created (carry-forward only copies *unresolved* rows); but the version-3 row itself must remain queryable for audit/history. Defaulting the list to "current" would conflate "active comments" with "comments on this version" and break version-diff workflows the §10.13 UI may add later. Story 10.13 callers always know which version they are rendering — they pass `version_id` explicitly.

**(b) Carry-forward is implemented as INSERT-clones inside `create_version`'s transaction, not as a JOIN at read time.** When `proposal_service.create_version` produces a new `proposal_versions` row (Story 7.3, `proposal_service.py:484`), Story 10.4 hooks the just-after-flush moment (after `proposal.current_version_id = version.id` is flushed at line 548) and runs a single INSERT-SELECT: `INSERT INTO client.proposal_comments (proposal_id, version_id, section_key, author_id, body, parent_comment_id, created_at) SELECT proposal_id, :new_version_id, section_key, author_id, body, parent_comment_id, NOW() FROM client.proposal_comments WHERE proposal_id = :pid AND version_id = :prev_version_id AND resolved = FALSE`. This (i) keeps the carry-forward atomic with version creation under the existing `_get_version_write_lock` (no separate write transaction can race in between), (ii) makes each new version a self-contained snapshot of its open conversation (a list query for version N never has to traverse predecessors), and (iii) preserves authorship (`author_id` is copied verbatim — the carried-forward comment is still "by Alice", not "by the new-version creator"). The cloned rows get fresh UUIDs and fresh `created_at` timestamps but a new column `carried_forward_from_id` links each clone back to its origin comment for audit observability. `resolved`/`resolved_by`/`resolved_at` are NULL on the clones (by definition — only unresolved rows are copied).

**(c) Resolve and unresolve are SEPARATE PATCH endpoints, not one toggle.** `PATCH /comments/{id}/resolve` and `PATCH /comments/{id}/unresolve` are idempotent and side-effect-explicit: re-resolving an already-resolved comment is a 200 no-op (audit-suppressed); unresolving an open comment is also a 200 no-op. This matches Story 10.3's split-by-action pattern (lock acquire vs. release) and prevents the §10.13 frontend from having to first GET the current state to flip a single toggle (a known race-prone UI pattern). Audit rows are written ONLY on actual transitions (resolved→unresolved or unresolved→resolved), not on no-op calls — observability stays signal-rich.

**(d) Body is immutable after create — no edit endpoint in this story.** The MVP keeps the model simple: comments cannot be edited; if a user wants to amend, they post a new comment and resolve the old. Rationale: edit-history is a separate feature (audit-grade vs. wiki-style), Epic 10 AC3 does not call for it, and adding it now widens the audit-log shape unnecessarily. Document the omission in the OpenAPI schema and README so the §10.13 UI does not show an edit button. A future story can add `PATCH /comments/{id}` with edit-history support.

**(e) `section_key` is free-form (matches Story 10.3) and is NOT validated against the version's `content.sections` array.** A reviewer may comment on a section_key that no longer exists in the current content (renamed or deleted) or on a section_key that is in the process of being authored. Validating against `content.sections` would (i) require loading the version's full JSONB on every comment write, (ii) couple the comments path to the content-save lifecycle, and (iii) break the carry-forward when an upstream version reorganizes sections. Document this in the endpoint docstring; the §10.13 UI is responsible for surfacing "comment on a section that no longer exists" as an "orphan" badge if it cares.

A sixth subtle point: **Story 10.4 introduces the FIRST modification to an existing Epic 10 collaborator path.** The hook into `create_version` requires importing `proposal_comment_service.carry_forward_unresolved_comments` from inside `proposal_service.py`. To avoid circular imports (the comment service imports `Proposal` and the proposal service would import from comment service), keep the hook as a deferred import inside the `create_version` body (`from client_api.services.proposal_comment_service import carry_forward_unresolved_comments`). The carry-forward call is wrapped in a try/except that logs ERROR and re-raises — version creation MUST fail if carry-forward fails, so the new version does not silently lose its open conversation. We document this trade-off in §Design Constraints (correctness > availability for this hook).

## Acceptance Criteria

1. [x] **AC1 — Alembic migration 038: `client.proposal_comments` table.** New file `services/client-api/alembic/versions/038_proposal_comments.py`, `revision = "038"`, `down_revision = "037"`, `branch_labels = None`, `depends_on = None`. Uses the same schema-qualified `op.execute(f"...")` raw-SQL pattern as migrations 035 and 037. `upgrade()` creates:
   - `id` UUID PRIMARY KEY DEFAULT `gen_random_uuid()`
   - `proposal_id` UUID NOT NULL, `FOREIGN KEY ... REFERENCES client.proposals(id) ON DELETE CASCADE`
   - `version_id` UUID NOT NULL, `FOREIGN KEY ... REFERENCES client.proposal_versions(id) ON DELETE CASCADE`
   - `section_key` TEXT NOT NULL (matches `SectionItem.key` from `schemas/proposal_content.py`; same free-form-string convention as Story 10.3's `proposal_section_locks.section_key`)
   - `author_id` UUID NOT NULL, `FOREIGN KEY ... REFERENCES client.users(id) ON DELETE RESTRICT` (deleting a user must not silently delete their review comments — restrict instead)
   - `body` TEXT NOT NULL, `CHECK (char_length(body) >= 1 AND char_length(body) <= 10000)` (DB-level guard against empty/oversized comments; matches the AC4 422 validation but provides defence-in-depth)
   - `parent_comment_id` UUID NULL, `FOREIGN KEY ... REFERENCES client.proposal_comments(id) ON DELETE CASCADE` (self-FK for threaded replies; nullable because top-level comments have no parent; CASCADE so deleting the root removes its replies — matches Slack/GitHub thread semantics)
   - `resolved` BOOLEAN NOT NULL DEFAULT FALSE
   - `resolved_by` UUID NULL, `FOREIGN KEY ... REFERENCES client.users(id) ON DELETE SET NULL` (nullable; SET NULL on user delete preserves the resolved=true state but loses the resolver attribution — acceptable for audit purposes since `audit_log` retains the immutable history)
   - `resolved_at` TIMESTAMPTZ NULL
   - `carried_forward_from_id` UUID NULL, `FOREIGN KEY ... REFERENCES client.proposal_comments(id) ON DELETE SET NULL` (nullable; populated by carry-forward INSERT to point at the origin comment in the prior version; SET NULL on origin delete since origins on prior versions are not expected to be deleted but we do not want a CASCADE that would erase the carried-forward chain)
   - `created_at` TIMESTAMPTZ NOT NULL DEFAULT `NOW()`
   - `updated_at` TIMESTAMPTZ NOT NULL DEFAULT `NOW()`
   - INDEX `ix_proposal_comments_proposal_id` on `(proposal_id)` — covers the per-proposal-list path.
   - INDEX `ix_proposal_comments_version_section` on `(proposal_id, version_id, section_key, created_at)` — composite covering the AC5 list-with-filters query (most common access pattern); ordering by `created_at` keeps the index-only-scan fast for the §10.13 sidebar.
   - INDEX `ix_proposal_comments_resolved` on `(proposal_id, version_id, resolved)` WHERE `resolved = FALSE` — partial index sized to just the unresolved rows; covers the AC8 carry-forward `INSERT ... SELECT` predicate.
   - INDEX `ix_proposal_comments_parent` on `(parent_comment_id)` WHERE `parent_comment_id IS NOT NULL` — partial index for the AC11 thread-list query.
   - CHECK constraint `ck_proposal_comments_resolved_consistency` ensuring `(resolved = FALSE AND resolved_by IS NULL AND resolved_at IS NULL) OR (resolved = TRUE AND resolved_by IS NOT NULL AND resolved_at IS NOT NULL)` — DB-level guarantee that resolution attribution is consistent with the boolean flag (defence-in-depth in case the service layer ever forgets to set both fields).
   - Schema is `client` on every `op.*` call (project-context rule #3). `downgrade()` drops the indexes, then the CHECK constraints, then the table, in reverse order. No new enum types.

2. [x] **AC2 — `ProposalComment` ORM model.** New file `services/client-api/src/client_api/models/proposal_comment.py` with `class ProposalComment(Base)`:
   - `from __future__ import annotations` at top (project-wide pattern; matches `proposal_section_lock.py` and `proposal_collaborator.py`).
   - `__tablename__ = "proposal_comments"`
   - `__table_args__` includes the 4 indexes from AC1, the CHECK constraint, and `{"schema": "client"}`.
   - Mapped columns mirror AC1 exactly. Use `sa.UUID(as_uuid=True)`, `sa.Text()`, `sa.DateTime(timezone=True)`, `sa.Boolean()` per the established style. UUID PKs use `default=uuid4, server_default=sa.text("gen_random_uuid()")` per the `ProposalCollaborator`/`ProposalSectionLock` template.
   - `body: Mapped[str]` (no validation in the ORM — Pydantic + the DB CHECK enforce length).
   - `parent_comment_id`, `resolved_by`, `resolved_at`, `carried_forward_from_id` are `Mapped[<type> | None]`.
   - Export from `client_api/models/__init__.py` via `from .proposal_comment import ProposalComment` and add `"ProposalComment"` to `__all__`.

3. [x] **AC3 — Pydantic schemas in `client_api/schemas/proposal_comments.py`.** Define:
   - `CommentCreateRequest(BaseModel)`: `version_id: UUID`, `section_key: str = Field(min_length=1, max_length=255)`, `body: str = Field(min_length=1, max_length=10000)`, `parent_comment_id: UUID | None = None`. `model_config = ConfigDict(extra="forbid")` to reject unknown fields (consistent with house style on POST bodies).
   - `CommentResponse(BaseModel)`: `id: UUID`, `proposal_id: UUID`, `version_id: UUID`, `section_key: str`, `author_id: UUID`, `author_full_name: str | None`, `body: str`, `parent_comment_id: UUID | None`, `resolved: bool`, `resolved_by: UUID | None`, `resolved_by_full_name: str | None`, `resolved_at: datetime | None`, `carried_forward_from_id: UUID | None`, `created_at: datetime`, `updated_at: datetime`. `model_config = ConfigDict(from_attributes=True)`.
   - `CommentListResponse(BaseModel)`: `proposal_id: UUID`, `version_id: UUID`, `section_key: str | None`, `comments: list[CommentResponse]`, `total: int`, `unresolved_count: int`. The `unresolved_count` is computed in the service via a `COUNT(*) FILTER (WHERE resolved = FALSE)` and is critical for the §10.13 sidebar's per-section badge — the UI must NOT have to re-iterate `comments` to compute it.
   - All response models with `model_config = ConfigDict(from_attributes=True)`.
   - No request body schema for `PATCH /resolve` or `PATCH /unresolve` — both are pure path-parameter endpoints (matches Story 10.3 lock acquire/release).

4. [x] **AC4 — `POST /api/v1/proposals/{proposal_id}/comments` endpoint.** New router `services/client-api/src/client_api/api/v1/proposal_comments.py`, mounted under the same `/api/v1/proposals` prefix as Story 10.3's section-locks router (include via `app.include_router(proposal_comments_v1.router, prefix="/proposals")` in `main.py` after the section-locks include — keep alphabetical-by-feature). Route signature:
   ```python
   @router.post(
       "/{proposal_id}/comments",
       response_model=CommentResponse,
       responses={
           400: {"description": "version_id does not belong to this proposal, or parent_comment_id does not belong to this proposal+version+section"},
           403: {"description": "Your proposal role does not permit creating comments"},
           404: {"description": "Proposal not found"},
       },
       status_code=201,
   )
   async def create_comment(
       proposal_id: UUID,
       request: CommentCreateRequest,
       session: Annotated[AsyncSession, Depends(get_db_session)],
       current_user: Annotated[CurrentUser, Depends(require_proposal_role(*PROPOSAL_WRITE_ROLES))],
   ) -> CommentResponse:
   ```
   Behaviour:
   - The `require_proposal_role(*PROPOSAL_WRITE_ROLES)` dep enforces (a) authenticated, (b) proposal exists and belongs to caller's company (cross-company → 404), (c) caller is a non-read_only collaborator or company admin. read_only collaborators get 403 (Epic 10 AC3 implicit constraint plus AC13 explicit role enforcement).
   - Validate `request.version_id` belongs to `proposal_id`: `SELECT EXISTS(SELECT 1 FROM client.proposal_versions WHERE id = :vid AND proposal_id = :pid)`. If not, raise 400 `detail="version_id does not belong to this proposal"`. This guards against cross-proposal smuggling — a caller cannot post a comment to proposal A pretending it's anchored to a version of proposal B.
   - If `request.parent_comment_id` is non-null, validate the parent exists and matches `(proposal_id, version_id, section_key)`: `SELECT 1 FROM client.proposal_comments WHERE id = :pid AND proposal_id = :proposal AND version_id = :vid AND section_key = :skey`. If not, raise 400 `detail="parent_comment_id does not belong to this proposal+version+section"`. This prevents threading a reply across sections or versions (would corrupt the §10.13 thread rendering).
   - Construct the `ProposalComment` ORM row with `author_id=current_user.user_id`, `resolved=False`, `created_at=NOW()`, `updated_at=NOW()`. Insert + flush + refresh.
   - Build `CommentResponse` with `author_full_name` resolved via a single JOIN to `client.users` in the same service call (avoids N+1 in the AC5 list path; for create, a second small query is acceptable but keep consistent — service helper joins once).
   - Audit log row: `write_audit_entry(session, user_id=current_user.user_id, action_type="create", entity_type="proposal_comment", entity_id=comment.id, company_id=current_user.company_id, after={"proposal_id": ..., "version_id": ..., "section_key": ..., "body_length": len(body), "parent_comment_id": ...})`. Note `body_length` not `body` — comment bodies may contain customer/proposal-confidential text and the audit log should not duplicate that payload (audit is metadata-grade, not content-grade per the Story 2.11 convention).
   - Structlog event `proposal_comment.created` at INFO with `proposal_id`, `version_id`, `section_key`, `comment_id`, `author_id`, `is_reply: bool` (true if `parent_comment_id` is non-null). DO NOT log `body` content (PII/confidential).
   - Return HTTP 201 with the `CommentResponse`.

5. [x] **AC5 — `GET /api/v1/proposals/{proposal_id}/comments` endpoint with filters.** New endpoint on the same router. Auth dep: `require_proposal_role(*PROPOSAL_ALL_ROLES)` (read_only CAN read comments — they cannot create or resolve, but they can review the conversation). Behaviour:
   - Required query params: `version_id: UUID` (the comments scope is always per-version — see §Description (a)). Optional query params: `section_key: str | None = None` (filter to one section; if omitted, returns all sections), `resolved: bool | None = None` (filter; if omitted, returns both resolved and unresolved).
   - SQL: `SELECT pc.*, author.full_name AS author_full_name, resolver.full_name AS resolved_by_full_name FROM client.proposal_comments pc LEFT JOIN client.users author ON author.id = pc.author_id LEFT JOIN client.users resolver ON resolver.id = pc.resolved_by WHERE pc.proposal_id = :pid AND pc.version_id = :vid [AND pc.section_key = :skey] [AND pc.resolved = :res] ORDER BY pc.created_at ASC`. Both joins are LEFT in case `resolved_by` is null (open comments) and to be defensive against ON DELETE SET NULL on `resolved_by`.
   - Validate `version_id` belongs to `proposal_id` (same check as AC4); if not, raise 400. This prevents leaking comment counts across proposals via a crafted `version_id` (rare but cheap to guard).
   - Compute `total` via `COUNT(*)` and `unresolved_count` via `COUNT(*) FILTER (WHERE resolved = FALSE)` in the same SQL aggregate query (single round-trip — see §Design Constraints "single-query list endpoint").
   - Return `CommentListResponse` with HTTP 200. Empty list (no comments yet) → `total=0`, `unresolved_count=0`, `comments=[]`. NOT 404.
   - No audit log row (reads are not audited — Story 10.2 AC10 convention).
   - No structlog event on happy path (this endpoint may be polled by §10.13 — keep log volume sane).

6. [x] **AC6 — `PATCH /api/v1/proposals/{proposal_id}/comments/{comment_id}/resolve` endpoint.** Auth dep: `require_proposal_role(*PROPOSAL_WRITE_ROLES)` (read_only cannot resolve). Behaviour:
   - Load the comment row `WHERE id = :cid AND proposal_id = :pid` (the WHERE on `proposal_id` enforces tenant scoping — combined with the proposal-id-in-dep this is defence-in-depth; mismatched comment_id → 404). If no row → 404 `detail="Comment not found"`.
   - Idempotency: if `comment.resolved == True`, return HTTP 200 with the existing comment (no-op; no audit, no structlog event). Document in the docstring.
   - Otherwise: `UPDATE client.proposal_comments SET resolved = TRUE, resolved_by = :uid, resolved_at = NOW(), updated_at = NOW() WHERE id = :cid`. Refresh + return `CommentResponse` with HTTP 200.
   - Audit log row with `action_type="update"`, `entity_type="proposal_comment"`, `entity_id=comment.id`, `before={"resolved": false}`, `after={"resolved": true, "resolved_by": "<uid>", "resolved_at": "<iso>"}`.
   - Structlog event `proposal_comment.resolved` at INFO with `proposal_id`, `comment_id`, `resolved_by`.

7. [x] **AC7 — `PATCH /api/v1/proposals/{proposal_id}/comments/{comment_id}/unresolve` endpoint.** Mirror of AC6. Auth dep: `require_proposal_role(*PROPOSAL_WRITE_ROLES)`. Behaviour:
   - Load comment `WHERE id = :cid AND proposal_id = :pid`. 404 if missing.
   - Idempotency: if `comment.resolved == False`, return HTTP 200 (no-op). No audit, no structlog.
   - Otherwise: `UPDATE client.proposal_comments SET resolved = FALSE, resolved_by = NULL, resolved_at = NULL, updated_at = NOW() WHERE id = :cid`. Return `CommentResponse` with HTTP 200.
   - Audit log row with `action_type="update"`, `before={"resolved": true, "resolved_by": "<uid>", "resolved_at": "<iso>"}`, `after={"resolved": false}`.
   - Structlog event `proposal_comment.unresolved` at INFO with `proposal_id`, `comment_id`, `unresolved_by`.

8. [x] **AC8 — Carry-forward hook in `proposal_service.create_version`.** Modify `services/client-api/src/client_api/services/proposal_service.py:484` (`async def create_version`). Insert the carry-forward call after the existing audit-write+log block (after line 569 `log.info("proposal.version_created", ...)` and before `return version`). Implementation:
   ```python
   # Story 10.4: carry forward unresolved comments from the prior version (if any).
   # Deferred import to avoid a circular dependency: proposal_comment_service
   # imports Proposal/ProposalVersion from client_api.models, and the comment
   # service module is consumed only by this single hook plus its own router.
   from client_api.services.proposal_comment_service import (
       carry_forward_unresolved_comments,
   )

   prev_version_id = (
       proposal.current_version_id
       if proposal.current_version_id != version.id
       else None  # If proposal had no prior version, current_version_id is now the new version's id
   )
   # Recover the prior version id: at this point we already wrote
   # proposal.current_version_id = version.id, so use the locally captured value.
   # The local `current_content` was sourced from the prior version; we need its id.
   # Capture this BEFORE step 6 reassigns current_version_id (i.e., capture at
   # the same point we read current_content in step 3).

   if prev_version_id is not None and prev_version_id != version.id:
       carried = await carry_forward_unresolved_comments(
           session,
           proposal_id=proposal_id,
           prev_version_id=prev_version_id,
           new_version_id=version.id,
           company_id=current_user.company_id,
           triggered_by=current_user.user_id,
       )
       log.info(
           "proposal.version_created.comments_carried_forward",
           proposal_id=str(proposal_id),
           prev_version_id=str(prev_version_id),
           new_version_id=str(version.id),
           carried_count=carried,
       )
   ```
   IMPORTANT IMPLEMENTATION DETAIL: the dev MUST refactor `create_version` to capture the prior version's id into a local variable **before** the line 547 reassignment `proposal.current_version_id = version.id`. The cleanest patch is to add `prev_version_id: UUID | None = proposal.current_version_id` immediately after step 3 (after fetching `current_content`), and to use that local variable in the carry-forward block. Do NOT introduce a new SELECT to recover the prior version id — capture it from the in-memory attribute while it still holds the prior pointer.
   - The carry-forward call MUST execute INSIDE the existing `_get_version_write_lock(proposal_id)` async-with block — that block already wraps the entire create_version body, so the placement is correct as long as the call is before the `return version`.
   - If the carry-forward function raises, version creation MUST fail (re-raise after a structlog ERROR). This is correct-but-conservative behaviour: a partial state where the new version exists but its open conversation was lost is worse than a failed version creation that the user can retry.
   - Extend the existing audit row for `entity_type="proposal"` with a new `after` field `unresolved_comments_carried: int` so a single audit entry captures both the version creation and the carry-forward count. This MAY be deferred to a separate audit row of `entity_type="proposal_comment_carry_forward"` if the dev prefers — pick whichever is cleaner; document the choice in the changelog.

9. [x] **AC9 — `proposal_comment_service.py` with carry-forward primitive.** New file `services/client-api/src/client_api/services/proposal_comment_service.py`. Public API:
   ```python
   async def create_comment(
       session: AsyncSession,
       *,
       proposal_id: UUID,
       version_id: UUID,
       section_key: str,
       author_id: UUID,
       body: str,
       parent_comment_id: UUID | None,
   ) -> ProposalCommentWithUser:
       """Insert a new comment row + return it joined with author display name."""

   async def list_comments(
       session: AsyncSession,
       *,
       proposal_id: UUID,
       version_id: UUID,
       section_key: str | None = None,
       resolved: bool | None = None,
   ) -> tuple[list[ProposalCommentWithUser], int, int]:
       """Return (rows, total, unresolved_count) for the AC5 list endpoint."""

   async def resolve_comment(
       session: AsyncSession,
       *,
       comment_id: UUID,
       proposal_id: UUID,
       resolver_id: UUID,
   ) -> tuple[ProposalCommentWithUser, bool]:
       """Returns (comment, transitioned) — transitioned is False on idempotent no-op."""

   async def unresolve_comment(
       session: AsyncSession,
       *,
       comment_id: UUID,
       proposal_id: UUID,
       unresolver_id: UUID,
   ) -> tuple[ProposalCommentWithUser, bool]:
       """Mirror of resolve_comment."""

   async def carry_forward_unresolved_comments(
       session: AsyncSession,
       *,
       proposal_id: UUID,
       prev_version_id: UUID,
       new_version_id: UUID,
       company_id: UUID,
       triggered_by: UUID,
   ) -> int:
       """INSERT-clones unresolved comments from prev_version_id into new_version_id.

       Returns the count of cloned rows. Wraps the INSERT in the caller's
       transaction (no own commit). Structlog event emitted by the caller.
       """
   ```
   The carry-forward implementation uses a single SQL `INSERT ... SELECT`:
   ```sql
   INSERT INTO client.proposal_comments
     (id, proposal_id, version_id, section_key, author_id, body, parent_comment_id, carried_forward_from_id, resolved, created_at, updated_at)
   SELECT
     gen_random_uuid(), proposal_id, :new_version_id, section_key, author_id, body, NULL, id, FALSE, NOW(), NOW()
   FROM client.proposal_comments
   WHERE proposal_id = :pid AND version_id = :prev_version_id AND resolved = FALSE
   RETURNING id;
   ```
   NOTE on `parent_comment_id`: we NULL it out on the carried-forward clones rather than try to re-link to the carried-forward parent. Rationale: re-linking would require either (i) a second pass after the first INSERT (since the new row ids are not known until INSERT runs) or (ii) a recursive CTE that maps old→new ids. Both are complexity for marginal value — the §10.13 sidebar shows top-level comments per section primarily; thread relinking can be added in a follow-up if needed. The `carried_forward_from_id` link preserves the audit trail back to the threaded predecessor if a future migration wants to reconstruct threads. Document this trade-off in the function docstring and §Design Constraints.
   Wrap all functions in `async with session.begin_nested()` ONLY if not already inside a transaction — defensive `session.in_transaction()` check before opening a savepoint. Default behaviour: rely on the caller's transaction (the FastAPI dep `get_db_session` opens one per request).

10. [x] **AC10 — Comment thread listing helper.** Add `async def list_comment_replies(session, *, parent_comment_id: UUID, proposal_id: UUID) -> list[ProposalCommentWithUser]` to the service. Used by the §10.13 sidebar for expanding a thread inline. Returns replies ordered by `created_at ASC`. The router exposes this via `GET /api/v1/proposals/{proposal_id}/comments/{comment_id}/replies` with `require_proposal_role(*PROPOSAL_ALL_ROLES)` auth. Returns 404 if the parent comment does not belong to the proposal. Returns an empty list for a comment with no replies (NOT 404).

11. [x] **AC11 — Endpoint route summary.** The router registers exactly 5 endpoints:
    - `POST   /api/v1/proposals/{proposal_id}/comments` → `require_proposal_role(*PROPOSAL_WRITE_ROLES)` (AC4)
    - `GET    /api/v1/proposals/{proposal_id}/comments` → `require_proposal_role(*PROPOSAL_ALL_ROLES)` (AC5)
    - `PATCH  /api/v1/proposals/{proposal_id}/comments/{comment_id}/resolve` → `require_proposal_role(*PROPOSAL_WRITE_ROLES)` (AC6)
    - `PATCH  /api/v1/proposals/{proposal_id}/comments/{comment_id}/unresolve` → `require_proposal_role(*PROPOSAL_WRITE_ROLES)` (AC7)
    - `GET    /api/v1/proposals/{proposal_id}/comments/{comment_id}/replies` → `require_proposal_role(*PROPOSAL_ALL_ROLES)` (AC10)

    The route-dependency snapshot test (`tests/unit/test_proposals_router_dependency_spec.py`) MUST be extended to capture all 5 entries. No DELETE endpoint in this story (intentional — see §Design Constraints "no delete in MVP").

12. [x] **AC12 — Comments-API test matrix.** New file `tests/api/test_proposal_comments.py`:
    - **Happy create top-level** — bid_manager creates a comment on (version_v1, section_intro) → 201 + `CommentResponse` body + DB row present + audit row present + structlog event captured.
    - **Happy create reply** — write_role user creates a reply (parent_comment_id set) → 201, parent linkage correct.
    - **Reply with cross-section parent** — parent comment is on section A, reply attempts section B → 400.
    - **Reply with cross-version parent** — parent is on v1, reply attempts v2 → 400.
    - **Reply with cross-proposal parent** — parent is on proposal X, reply attempts proposal Y → 400.
    - **Create with non-matching version_id** — version belongs to a different proposal → 400.
    - **Create with empty body** — body=" " (whitespace) → 422 from Pydantic; body="" → 422; body of 10001 chars → 422.
    - **Create as read_only** — read_only collaborator → 403.
    - **Create as non-collaborator non-admin** → 403.
    - **Create cross-company** → 404.
    - **Create as company admin (no collaborator row)** → 201 (admin bypass, Story 10.2 convention).
    - **List happy path** — 5 comments seeded across 2 sections (3 in section A, 2 in section B); GET filtered by section_key=A → 3 returned, ordered by created_at ASC; `total=3`, `unresolved_count=3`.
    - **List with no section_key** — all 5 returned, in created_at order.
    - **List with resolved=true** — only resolved rows.
    - **List with resolved=false** — only open rows; matches `unresolved_count`.
    - **List for empty proposal** — 0 comments → 200 + empty list + counts=0 (NOT 404).
    - **List with mismatched version_id** — version belongs to other proposal → 400.
    - **List as read_only** → 200 (read_only CAN read).
    - **Resolve happy path** — open comment → 200, `resolved=true`, `resolved_by`+`resolved_at` populated, audit row.
    - **Resolve idempotent** — already-resolved → 200, no audit row, no structlog event.
    - **Resolve as read_only** → 403.
    - **Resolve non-existent** → 404.
    - **Resolve mismatched proposal_id+comment_id** — comment exists but on different proposal → 404 (tenant defence-in-depth).
    - **Unresolve happy path** — resolved comment → 200, fields reset, audit row.
    - **Unresolve idempotent** — already-unresolved → 200, no audit.
    - **Unresolve as read_only** → 403.
    - **Replies endpoint happy path** — parent + 2 replies → GET returns 2 replies in order.
    - **Replies on non-existent parent** → 404.
    - **Replies on parent with no children** → 200 + empty list.

13. [x] **AC13 — Carry-forward integration test.** New file `tests/api/test_proposal_comments_carry_forward.py` (or co-locate as a class in the AC12 file — dev's choice):
    - **Setup** — proposal with version v1, 5 comments on section_intro (3 unresolved, 2 resolved), 2 comments on section_scope (1 unresolved, 1 resolved). Total: 7 comments, 4 unresolved.
    - **Trigger** — POST `/api/v1/proposals/{id}/versions` (Story 7.3 endpoint) with bid_manager auth → creates version v2.
    - **Assert** — query `proposal_comments WHERE version_id = v2`: exactly 4 rows; 3 with `section_key='intro'`, 1 with `section_key='scope'`; all with `resolved=False`; all with `carried_forward_from_id` populated to the originating v1 row's id; all with `parent_comment_id=NULL` (replies are not relinked per AC9 trade-off); all with `author_id` matching the original comments' authors (NOT the version creator).
    - **Assert v1 comments unchanged** — v1's 7 rows are still present, untouched.
    - **Assert audit** — version-creation audit row's `after` includes `unresolved_comments_carried: 4` (or a separate `proposal_comment_carry_forward` audit row exists, per the AC8 dev choice).
    - **Assert structlog** — `proposal.version_created.comments_carried_forward` event captured with `carried_count=4`.
    - **Edge: empty source** — proposal with no comments on v1 → version v2 creation succeeds, 0 carried, structlog event still emitted with `carried_count=0`.
    - **Edge: all resolved on source** — proposal with 3 comments on v1 all resolved → 0 carried.
    - **Edge: first version** — initial version creation (no prior version exists) → `prev_version_id is None` branch → carry-forward not invoked, no structlog event for it.
    - **Failure mode** — monkeypatch `carry_forward_unresolved_comments` to raise → POST `/versions` returns 500 (or whatever `proposal_service.create_version`'s exception maps to) AND no v2 row is left in the DB (transaction rolled back). This proves the AC8 "fail-loud" behaviour.

14. [x] **AC14 — Service-layer unit tests.** New file `tests/unit/test_proposal_comment_service.py`:
    - `create_comment` inserts row with correct fields; returns joined display name.
    - `list_comments` filters: by section_key only, by resolved only, by both, by neither.
    - `list_comments` returns correct `(total, unresolved_count)` tuple under mixed-state seed.
    - `resolve_comment`: open → resolved (transitioned=True); already-resolved → no-op (transitioned=False).
    - `unresolve_comment`: mirror.
    - `list_comment_replies`: ordering, empty case.
    - `carry_forward_unresolved_comments`: copies only resolved=FALSE rows; copies authors verbatim (not the trigger user); NULLs parent_comment_id; populates carried_forward_from_id; returns correct count.
    - `carry_forward_unresolved_comments` with empty source → returns 0.
    - Target ≥90% branch coverage on the service module.

15. [x] **AC15 — RBAC matrix coverage for the 5 new routes.** Add `tests/api/test_comments_rbac_matrix.py`:
    - 5 endpoints × 8 caller profiles (admin, bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only, non-collaborator, cross-company) = 40 assertions.
    - Expected: POST + 2 PATCH (resolve/unresolve) → admin + 4 write roles = 201/200; read_only + non-collab = 403; cross-company = 404.
    - GET + replies → all 6 collaborator-roles + admin = 200; non-collab = 403; cross-company = 404.
    - Denial audit assertion on each 403 path (spot-check 3 of the 12 403 cases — the read_only-on-create, read_only-on-resolve, and non-collab-on-create paths).
    - Reuse the Story 10.3 `section_lock_ctx` fixture or its parent fixtures (same shape — 5 collaborator roles seeded per proposal). Comment fixtures extend this with seeded comments where needed.

16. [x] **AC16 — Migration smoke test.** Extend `tests/integration/test_alembic_migrations.py` (or add a dedicated `tests/integration/test_migration_038.py`):
    - `alembic upgrade head` applies 038 cleanly; table + 4 indexes + 1 CHECK constraint exist post-upgrade.
    - `alembic downgrade 037` removes them cleanly; subsequent `upgrade head` re-applies idempotently.
    - DB-level CHECK constraint rejects an INSERT with `resolved=TRUE AND resolved_by IS NULL` (raises `IntegrityError`).
    - DB-level body-length CHECK rejects 10001-char body (`IntegrityError`).
    - `parent_comment_id` ON DELETE CASCADE: delete a parent → replies vanish.
    - `author_id` ON DELETE RESTRICT: attempting to delete a user with comments → `IntegrityError`.
    - `resolved_by` ON DELETE SET NULL: delete the resolver → comment row stays, `resolved_by` becomes NULL.

17. [x] **AC17 — Route-dependency snapshot updated.** Update `tests/unit/test_proposals_router_dependency_spec.py` snapshot to include the 5 new `/api/v1/proposals/{proposal_id}/comments*` routes (per AC11):
    - `POST .../comments` → `require_proposal_role:PROPOSAL_WRITE_ROLES`
    - `GET .../comments` → `require_proposal_role:PROPOSAL_ALL_ROLES`
    - `PATCH .../comments/{cid}/resolve` → `require_proposal_role:PROPOSAL_WRITE_ROLES`
    - `PATCH .../comments/{cid}/unresolve` → `require_proposal_role:PROPOSAL_WRITE_ROLES`
    - `GET .../comments/{cid}/replies` → `require_proposal_role:PROPOSAL_ALL_ROLES`

    The test's existing `_describe_dependency` helper is reused unchanged; only the snapshot dict needs the new entries. Run the test; confirm pre-existing routes still match.

18. [x] **AC18 — README + OpenAPI documentation.** Update `services/client-api/README.md`:
    - New section "Proposal Comments (Story 10.4)" describing the per-version anchoring, the carry-forward semantics, the resolve/unresolve idempotency, the 10000-char body limit, the no-edit constraint, the no-delete constraint (MVP), and the §10.13 UI's expected polling cadence.
    - Add the 5 new routes to the endpoint→allowed-role-set table under the "Proposal-Level RBAC Middleware (Story 10.2)" section.
    - Document the "no parent re-linking on carry-forward" trade-off and the "body is immutable" trade-off, with forward-pointers to potential follow-up stories.
    - Verify `/openapi.json` renders all 5 routes with their `responses={...}` annotations and the `CommentResponse`/`CommentListResponse` schemas.

## Design Constraints

- **Per-version anchoring, not per-current-version.** Each comment row carries a NOT NULL `version_id`. The list endpoint requires `version_id` as a query param and never defaults to "current". This preserves history and is what §10.13 needs.
- **Carry-forward is INSERT-clone inside `create_version`'s transaction.** Single `INSERT ... SELECT` runs after the new version row is flushed and before `create_version` returns. Wrapped in the existing `_get_version_write_lock(proposal_id)` async lock + the FastAPI request transaction. If carry-forward fails, version creation fails (no orphan version with a lost conversation).
- **No edit endpoint.** Body is immutable after create. Audit-grade behaviour preserved without an edit-history table. Document in README + OpenAPI; §10.13 must not show an edit affordance.
- **No delete endpoint in MVP.** Deletion of comments is intentionally omitted. Soft-delete or admin-purge can be added in a follow-up if a moderation requirement materialises. The model schema does not pre-bake a `deleted_at` column to keep the row shape minimal — adding one later is a trivial migration.
- **Resolve and unresolve are idempotent and audit-suppressed on no-op.** Re-resolving an already-resolved comment is a 200 no-op, no audit. Only true transitions produce audit rows. This keeps the audit log signal-rich and matches the §10.13 UI's optimistic-toggle pattern.
- **`section_key` is free-form, not FK to `content.sections`.** Same convention as Story 10.3. A reviewer can comment on a section that no longer exists; the §10.13 UI surfaces this as an "orphan" badge if it cares. We do not load `content` JSONB on comment writes.
- **`author_id` ON DELETE RESTRICT.** Deleting a user must not silently destroy their review history. Force the operator to either reassign or explicitly archive comments first. This is opinionated but correct for an audit-grade collaboration surface.
- **`resolved_by` ON DELETE SET NULL.** Resolution attribution is lost on user delete (acceptable — `audit_log` retains immutable history); the resolved=TRUE state is preserved (so the comment stays out of the unresolved-list).
- **Parent-comment re-linking on carry-forward is OUT OF SCOPE.** Carried-forward replies become top-level on the new version (`parent_comment_id=NULL`). The `carried_forward_from_id` chain preserves the audit trail. A follow-up story can add a recursive-CTE relinker if the §10.13 UI requires nested-thread continuity across versions.
- **`carried_forward_from_id` is SET NULL on origin delete.** We do not CASCADE because deleting an origin comment on a prior version should not retroactively destroy the new-version clone.
- **Single-query list endpoint.** AC5 returns `comments`, `total`, and `unresolved_count` in one round-trip via `COUNT(*) FILTER (...)` aggregates joined with the row-set. The §10.13 UI must not have to make two requests to render a section header badge.
- **`body` content NEVER appears in audit_log or structlog.** Bodies may contain customer-confidential proposal text. Audit captures only `body_length`; structlog captures only metadata.
- **Cross-company = 404, not 403.** Inherited from Story 10.2's `require_proposal_role`.
- **Defence-in-depth on tenant scoping.** Every comment-mutation endpoint includes `WHERE proposal_id = :pid` in its load query, even though the dep already enforces this. Mismatched (proposal_id, comment_id) tuples → 404.
- **No new package dependencies.** Reuses sqlalchemy, asyncpg, structlog, pydantic already in `pyproject.toml`.

## Tasks / Subtasks

- [x] **Task 1: Alembic migration 038 + ORM model. (AC1, AC2)**
  - [x] Create `services/client-api/alembic/versions/038_proposal_comments.py` per AC1. Use the raw-SQL schema-qualified pattern from migration 037. `upgrade()` creates table + 4 indexes + 1 CHECK + 1 body-length CHECK. `downgrade()` drops them in reverse.
  - [x] Create `services/client-api/src/client_api/models/proposal_comment.py` per AC2. Mirror the style of `proposal_section_lock.py` (from __future__, Mapped columns, schema="client", explicit __table_args__).
  - [x] Export `ProposalComment` from `client_api/models/__init__.py`.
  - [x] Integration test (AC16): `alembic upgrade head` then `downgrade 037` clean; CHECK constraints reject malformed inserts; FK ON DELETE policies behave per AC1.

- [x] **Task 2: Pydantic schemas. (AC3)**
  - [x] Create `services/client-api/src/client_api/schemas/proposal_comments.py` with `CommentCreateRequest`, `CommentResponse`, `CommentListResponse` per AC3.
  - [x] Add to `schemas/__init__.py` re-exports if that file uses explicit re-exports (follow house style — check Story 10.3's pattern and mirror it).

- [x] **Task 3: Service layer. (AC9, AC10, AC14)**
  - [x] Create `services/client-api/src/client_api/services/proposal_comment_service.py` per AC9 + AC10.
  - [x] Define a lightweight `ProposalCommentWithUser` dataclass (or Pydantic model) for joined-row results (carries the comment row + `author_full_name` + `resolved_by_full_name`).
  - [x] Implement `create_comment`, `list_comments`, `resolve_comment`, `unresolve_comment`, `list_comment_replies`, `carry_forward_unresolved_comments`. Each uses parameterised SQLAlchemy Core or ORM queries.
  - [x] Unit tests per AC14 (file `tests/unit/test_proposal_comment_service.py`). Cover the carry-forward edge cases.

- [x] **Task 4: Router — 5 endpoints. (AC4, AC5, AC6, AC7, AC10, AC11)**
  - [x] Create `services/client-api/src/client_api/api/v1/proposal_comments.py` with the 5 routes per AC11.
  - [x] Register the router in `main.py` after the section-locks include with `prefix="/proposals"` (mirrors Story 10.3 line 121: `api_v1_router.include_router(proposal_section_locks_v1.router, prefix="/proposals")`). Add `from client_api.api.v1 import proposal_comments as proposal_comments_v1` to the imports block (line 32-34 region).
  - [x] Import `require_proposal_role`, `PROPOSAL_WRITE_ROLES`, `PROPOSAL_ALL_ROLES` from `client_api.core.rbac`.
  - [x] Defence-in-depth inline comments on each handler per Story 10.2/10.3 convention (note that the dep already enforces tenant scoping; the redundant WHERE clauses are belt-and-braces).

- [x] **Task 5: Carry-forward hook into `create_version`. (AC8)**
  - [x] Modify `services/client-api/src/client_api/services/proposal_service.py:484` — capture `prev_version_id` BEFORE the line 547 `proposal.current_version_id = version.id` reassignment.
  - [x] Add the deferred-import + carry-forward call after the existing `log.info("proposal.version_created", ...)` block (after line 569).
  - [x] Re-raise on carry-forward failure (no swallow).
  - [x] Augment audit row OR add a separate audit row for the carry-forward count (dev's choice — document in changelog).

- [x] **Task 6: API tests — happy paths, RBAC, carry-forward. (AC12, AC13, AC15)**
  - [x] `tests/api/test_proposal_comments.py` (AC12) — ~25 test cases covering CRUD-ish, idempotency, validation, role enforcement.
  - [x] `tests/api/test_proposal_comments_carry_forward.py` (AC13) — 6+ test cases including the failure-mode test that asserts transactional rollback.
  - [x] `tests/api/test_comments_rbac_matrix.py` (AC15) — 40 assertion matrix.
  - [x] Reuse the Story 10.3 `section_lock_ctx` fixture pattern (5 collaborator roles + non-collab + cross-company seeded). Add a `make_comment` helper fixture in `conftest.py` if it does not already exist.

- [x] **Task 7: Route-dependency snapshot update. (AC17)**
  - [x] Extend `tests/unit/test_proposals_router_dependency_spec.py` snapshot to include the 5 new routes.
  - [x] Run the test; confirm all pre-existing routes (collaborators, section-locks, proposals, etc.) still match; new routes added with correct role-set tags.

- [x] **Task 8: Documentation. (AC18)**
  - [x] Extend `services/client-api/README.md` with the "Proposal Comments (Story 10.4)" section per AC18.
  - [x] Update the endpoint→role-set table (under the Story 10.2 section) to include the 5 new routes.
  - [x] Verify `/openapi.json` renders all 5 routes with proper `responses={...}` annotations and the new schemas.
  - [x] Document the "no edit" + "no delete" + "no parent re-linking on carry-forward" trade-offs.

- [x] **Task 9: Lint + type-check + CI alignment.**
  - [x] `ruff check` clean on all new/modified files:
        `services/client-api/src/client_api/api/v1/proposal_comments.py`,
        `services/client-api/src/client_api/models/proposal_comment.py`,
        `services/client-api/src/client_api/schemas/proposal_comments.py`,
        `services/client-api/src/client_api/services/proposal_comment_service.py`,
        `services/client-api/src/client_api/services/proposal_service.py` (modified — carry-forward hook),
        `services/client-api/alembic/versions/038_proposal_comments.py`,
        `services/client-api/src/client_api/models/__init__.py`,
        `services/client-api/src/client_api/main.py` (modified — router registration),
        `services/client-api/tests/unit/test_proposal_comment_service.py`,
        `services/client-api/tests/unit/test_proposals_router_dependency_spec.py` (modified — snapshot extension),
        `services/client-api/tests/api/test_proposal_comments.py`,
        `services/client-api/tests/api/test_proposal_comments_carry_forward.py`,
        `services/client-api/tests/api/test_comments_rbac_matrix.py`,
        `services/client-api/tests/integration/test_migration_038.py` (or extension to `test_alembic_migrations.py`).
  - [x] `ruff format --check` clean.
  - [x] `mypy` clean on all changed files.
  - [x] `make test-client-api` (or equivalent) — full suite green. Coverage targets: comment service module ≥90% branch; router module ≥85% branch; carry-forward hook 100% (covered by AC13's 6 test cases).

## Dev Notes

### Test Expectations (from Epic 10 Test Design surrogate sources)

**NOTE on `test-design-epic-10` absence:** Same caveat as Stories 10.1–10.3. The epic-level test design has not been authored. This story pulls from Epic 10 AC3 (verbatim) and the Story 10.3 surrogate-source pattern: `test-design-architecture.md` does not list a comments-specific risk, but the Epic 10 AC13 RBAC enforcement requirement (proposal-level RBAC for all mutation endpoints) applies here. `test-design-qa.md` does not have a P-row for comments specifically; the closest precedents are P1-007 (section locking) and the audit-trail expectations under R-006 (entity-level RBAC). Flagged for TEA backlog — still not blocking for dev.

Requirement + risk rows this story owns:

| Priority | Requirement | Level | Count | Test Harness |
|---|---|---|---|---|
| **P0** | Epic 10 AC3 — comment can be created on a (section, version) tuple → 201 with `CommentResponse` | API | 2 | `test_proposal_comments.py` happy-path top-level + reply cases. |
| **P0** | Epic 10 AC3 — comments can be listed per section + version with chronological ordering | API | 3 | List with section_key filter, no filter, and resolved=true filter. |
| **P0** | Epic 10 AC3 — resolve/unresolve toggles persist + audit | API | 4 | Resolve happy + idempotent + Unresolve happy + idempotent. |
| **P0** | Epic 10 AC3 — unresolved comments carry forward to new version | API | 1 | `test_proposal_comments_carry_forward.py` happy path with mixed resolved/unresolved seed. |
| **P0** | Epic 10 AC3 — only unresolved rows are carried (resolved are not duplicated) | API | 1 | Carry-forward test asserting v2 has only the 4 unresolved rows. |
| **P0** | Epic 10 AC3 — carry-forward preserves authorship, not the version-creator's identity | API | 1 | Assertion on `author_id` after carry-forward. |
| **P0** | Epic 10 AC13 — read_only collaborators cannot create/resolve/unresolve | API | 6 | RBAC matrix read_only row × 3 mutation endpoints (×2 for both resolve+unresolve idempotency). |
| **P0** | Epic 10 AC13 — non-collaborator non-admin → 403 on all mutation endpoints; → 403 on read endpoints | API | 5 | RBAC matrix non-collab row × 5 endpoints. |
| **P0** | Cross-company caller → 404 (existence-leakage guard) | API | 5 | RBAC matrix cross-company row × 5 endpoints. |
| **P0** | Carry-forward failure → version creation rolls back (no orphan v2 with lost conversation) | API | 1 | Monkeypatch carry_forward to raise; assert no v2 row in DB after POST /versions returns 5xx. |
| **P1** | Cross-section/cross-version/cross-proposal parent_comment_id → 400 | API | 3 | AC12 reply-validation cases. |
| **P1** | version_id mismatched to proposal_id → 400 on POST and GET | API | 2 | AC12 + AC13 validation cases. |
| **P1** | Body validation: empty, whitespace-only, oversized → 422 | API | 3 | Pydantic + DB CHECK belt-and-braces. |
| **P1** | First-version creation does NOT invoke carry-forward (no prior version) | API | 1 | AC13 edge case. |
| **P1** | Empty-source carry-forward → 0 cloned, structlog event still emitted | API | 2 | All-resolved + no-comments edges. |
| **P1** | Comment thread replies endpoint returns ordered list; empty for childless parent | API | 2 | AC12 replies cases. |
| **P1** | `total` and `unresolved_count` are accurate under mixed-state seed | API | 1 | AC12 list happy-path counts assertion. |
| **P1** | Service-level: atomic INSERT-SELECT carry-forward; correct count returned | Unit | 4 | `test_proposal_comment_service.py` carry-forward cases. |
| **P2** | Audit log shape on create/resolve/unresolve with body_length (NOT body content) | API | 3 | Spot-check audit_log rows. |
| **P2** | Audit log NOT written on idempotent no-op resolve/unresolve | API | 2 | AC12 idempotency cases. |
| **P2** | Structlog event NOT written for body content (PII guard) | Unit | 1 | structlog capture asserting body absent. |
| **P2** | Migration 038 rollback + re-apply idempotency | Integration | 1 | AC16. |
| **P2** | DB CHECK constraints reject malformed inserts (resolved without resolved_by; oversized body) | Integration | 2 | AC16. |
| **P2** | FK ON DELETE policies (CASCADE on parent, RESTRICT on author, SET NULL on resolver) | Integration | 3 | AC16. |
| **P2** | Route-dependency spec snapshot captures 5 new routes | Unit | 1 | AC17. |

Total: ~60 test assertions across ~36 test functions. Slightly larger surface than 10.3 because of the 5-endpoint footprint, the carry-forward integration, and the threading semantics. Compact relative to 10.2's middleware refactor.

**Explicit risk links:**

- **R-006 (SEC, MEDIUM 6) — PARTIALLY CLOSED for the comments axis.** Epic 10 AC13 RBAC enforcement on the 5 new mutation endpoints; AC15's 40-assertion matrix is the regression harness.
- **Epic 10 AC3 — CLOSED by this story.** All four sub-assertions (create on section+version, list per section, resolve/unresolve toggle, carry-forward on new version) have dedicated AC12/AC13 test coverage.
- **No R-011-style concurrency risk specific to comments.** Comments do not contend on a shared resource the way section locks do (each comment is its own row). The atomic-INSERT carry-forward inside the existing `_get_version_write_lock` inherits Story 7.3's serialisation guarantee.

### Why per-version anchoring (and not per-current-version)

The naive design "comments live on the proposal; only the current version is queried" loses history: when version v3 is created, all v2 comments either move to v3 (destroying the v2 view) or remain on v2 (and the user has to "see comments from v2" toggle to find them). Per-version anchoring with INSERT-clone carry-forward gives us:

1. **Each version is a self-contained snapshot.** A list query for version N returns exactly the conversation that was open on N at the moment N was created (plus any new comments added on N afterwards). No traversal of predecessors. This is what §10.13's "show comments for the version I'm reading" UX needs.
2. **Audit-grade history.** v2's row set is preserved verbatim; an auditor can replay "what conversation was happening on v2" without reconstructing from a join chain.
3. **Resolved-on-version-N stays resolved-on-version-N.** Because we only carry forward `resolved=FALSE` rows, the v3 snapshot does not silently re-open conversations the team had previously closed.
4. **Trade-off:** storage cost grows with version count × open-comment count. For typical proposals (≤10 versions, ≤20 open comments per version) this is a few hundred rows per proposal — trivial. Indexes are sized accordingly.

The alternative — "denormalise via `created_in_version_id` + carry-forward as a JOIN" — was considered and rejected: the JOIN would have to traverse all prior versions to find unresolved comments, which is unbounded; the index design would be more complex; and the UI would need to re-implement the same traversal client-side. INSERT-clone is simpler, correct-by-construction, and matches the §10.13 contract.

### Why the carry-forward hook lives inside `create_version` (and not as a separate endpoint)

The naive alternative — "expose a `POST /proposals/{id}/comments/carry-forward` endpoint that the frontend calls after creating a version" — is wrong for three reasons:

1. **Race window.** Between `POST /versions` succeeding and the frontend calling `POST /carry-forward`, another user could create a third version, and the carry-forward could end up linking the wrong predecessor.
2. **Incomplete-state observability.** A new version visible to the world but with NO carried-forward comments is a surprising intermediate state for any concurrent reader. The §10.13 sidebar would show "0 comments on v2" for the milliseconds between requests, then suddenly show 4 — an annoying flash.
3. **Failure asymmetry.** If the frontend's carry-forward call fails (network blip), v2 exists with no comments. The user has to re-trigger carry-forward — but there's no UI for that, and no way for the backend to know it's needed.

The hook inside `create_version` makes the operation atomic with version creation: either both succeed or both roll back. The deferred-import dance avoids the circular dependency cleanly. The cost is one cross-module dependency (`proposal_service` → `proposal_comment_service`) — but this is acceptable because Story 10.4 is the natural consumer of the version-lifecycle event, and the alternative (an event-bus emit + async consumer) would not be atomic.

### Relevant Architecture Patterns & Constraints

1. **`require_proposal_role(*PROPOSAL_WRITE_ROLES)` on mutations; `*PROPOSAL_ALL_ROLES` on reads.** Established by Stories 10.2 and 10.3. Story 10.4 reuses verbatim.
2. **Migration-numbering convention:** next-available slot. 037 was Story 10.3's slot; 038 is this story's slot.
3. **Schema-qualified `op.execute(f"...")` DDL.** Project-context rule #3; migrations 035 and 037 are templates.
4. **Audit-log conventions.** `write_audit_entry` (Story 2.11) for success paths (`action_type` in `{create, update}`); `_write_denial_audit` for 403 paths. Body content is NEVER captured in `after`/`before` (use `body_length` instead — PII/confidential guard).
5. **`from __future__ import annotations` at top of every new module.** Project-wide pattern; matches `proposal_collaborator.py`, `proposal_section_lock.py`.
6. **`section_key: str` is free-form.** Inherited from Story 10.3's convention. No FK to section rows.
7. **Conflict body via `JSONResponse`, not `HTTPException`.** Established by Story 7.4 for 409 `version_conflict` and Story 10.3 for 423 `section_locked`. Story 10.4 does NOT need a conflict body (no 409/423/422-with-rich-body cases — pure 200/201/400/403/404 endpoints), so this pattern is referenced but not invoked here.
8. **Deferred imports for cross-service circular dependency avoidance.** Used in `create_version`'s carry-forward hook; document inline.
9. **`request.state._rbac_cache` reuse.** Story 10.2's cache is populated by the dep; comment endpoints can consult it for the (rare) future case where they need to know the caller's role beyond what the dep set. Not strictly needed for Story 10.4's MVP since `PROPOSAL_WRITE_ROLES`/`PROPOSAL_ALL_ROLES` discrimination at the dep level covers all paths.
10. **House style on router registration.** Follow `main.py`'s existing `include_router(...)` pattern (line 121 for section-locks): `api_v1_router.include_router(proposal_comments_v1.router, prefix="/proposals")`. Imports at line 32-34 region.
11. **`_get_version_write_lock(proposal_id)` async lock.** Story 7.3's lock registry. The carry-forward hook executes INSIDE this lock (because `create_version`'s body is wrapped in `async with _get_version_write_lock(...)` per the existing implementation at `proposal_service.py:508`). No additional locking needed.
12. **Tenant scoping defence-in-depth.** Every mutation endpoint includes `WHERE proposal_id = :pid` in its load query, redundant with the dep enforcement. Mismatched comment_id → 404, not 403.

### Source Tree Components to Touch

**New files:**
- `eusolicit-app/services/client-api/alembic/versions/038_proposal_comments.py`
- `eusolicit-app/services/client-api/src/client_api/models/proposal_comment.py`
- `eusolicit-app/services/client-api/src/client_api/schemas/proposal_comments.py`
- `eusolicit-app/services/client-api/src/client_api/services/proposal_comment_service.py`
- `eusolicit-app/services/client-api/src/client_api/api/v1/proposal_comments.py`
- `eusolicit-app/services/client-api/tests/unit/test_proposal_comment_service.py`
- `eusolicit-app/services/client-api/tests/api/test_proposal_comments.py`
- `eusolicit-app/services/client-api/tests/api/test_proposal_comments_carry_forward.py`
- `eusolicit-app/services/client-api/tests/api/test_comments_rbac_matrix.py`
- `eusolicit-app/services/client-api/tests/integration/test_migration_038.py` (or extend an existing `test_alembic_migrations.py` if one exists)

**Modified files:**
- `eusolicit-app/services/client-api/src/client_api/models/__init__.py` — re-export `ProposalComment` + add to `__all__`.
- `eusolicit-app/services/client-api/src/client_api/main.py` — register the new router via `api_v1_router.include_router(proposal_comments_v1.router, prefix="/proposals")` after the section-locks include (line 121); add corresponding import at line 32-34 region.
- `eusolicit-app/services/client-api/src/client_api/services/proposal_service.py` — modify `create_version` (line 484) to capture `prev_version_id` before line 547 and invoke `carry_forward_unresolved_comments` after line 569. Deferred-import inside the function body.
- `eusolicit-app/services/client-api/tests/unit/test_proposals_router_dependency_spec.py` — extend snapshot to include the 5 new routes (AC17).
- `eusolicit-app/services/client-api/tests/api/conftest.py` — add a `make_comment(proposal_id, version_id, section_key, author_id, *, resolved=False, parent_comment_id=None)` helper fixture if it does not already exist; extend `section_lock_ctx` (or add a parallel `comments_ctx`) for the RBAC matrix tests.
- `eusolicit-app/services/client-api/src/client_api/schemas/__init__.py` — re-export new schemas if that file uses explicit re-exports (mirror Story 10.3's pattern).
- `eusolicit-app/services/client-api/README.md` — new "Proposal Comments (Story 10.4)" section + endpoint→role-set table rows (AC18).

**Files intentionally NOT modified:**
- `eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py` — Story 10.4 does NOT touch the existing proposal endpoints.
- `eusolicit-app/services/client-api/src/client_api/api/v1/proposal_section_locks.py` — Story 10.3's router is untouched.
- `eusolicit-app/services/client-api/src/client_api/api/v1/proposal_collaborators.py` — Story 10.1's router is untouched.
- `eusolicit-app/services/client-api/src/client_api/core/rbac.py` — `require_proposal_role` + role-set constants are already in place. Story 10.4 only consumes them.

### Testing Standards Summary

- **pytest markers.** Unit: no marker. API: no marker. Integration (migration smoke + cross-version carry-forward): `@pytest.mark.integration`. Carry-forward integration test (AC13) lives in `tests/api/` because it exercises the API surface end-to-end (POST /versions → assert comments cloned), not the migration directly.
- **Fixtures to reuse:** `auth_header_factory`, `collaborator_header_factory(role)`, `make_proposal` (auto-seeds bid_manager collaborator per Story 10.2 pattern), `section_lock_ctx` (5-role + non-collab + cross-company seeded environment from Story 10.3), DB session fixture.
- **New fixture:** `make_comment(proposal_id, version_id, section_key, author_id, *, body="...", resolved=False, parent_comment_id=None, carried_forward_from_id=None)` — creates a comment row directly via ORM for test setup. Variant `make_comments_for_version(proposal_id, version_id, sections_with_counts: dict[str, tuple[int, int]])` — bulk-seeds for carry-forward tests (returns counts of (unresolved, resolved) per section).
- **Coverage targets:** comment service module ≥90% branch; router module ≥85% branch; carry-forward hook 100% (covered by AC13's 6+ scenarios); new `models/proposal_comment.py` 100%.
- **CI:** ruff + mypy + pytest must all pass. No new package dependencies (reuses sqlalchemy, asyncpg, structlog, pydantic, celery already in `pyproject.toml`).

### Previous Story Intelligence

**From Story 10.3 (`proposal_section_locks` router + service):**
- The 3-endpoint router pattern (one router file, multiple routes, all under `/api/v1/proposals` prefix) is established. Story 10.4 mirrors this.
- The `JSONResponse`-for-conflict-body convention is documented but Story 10.4 does NOT need it (no 409/423 paths).
- The `tests/unit/test_proposals_router_dependency_spec.py` snapshot pattern: extend the snapshot dict, no helper changes needed.
- The `section_lock_ctx` fixture (5-role + non-collab + cross-company) is the template for the AC15 RBAC matrix.
- The Celery Beat task pattern: NOT needed here (comments don't auto-expire; no cleanup task).

**From Story 10.2 (`require_proposal_role` + creator-seed):**
- `require_proposal_role(*allowed_roles)` factory is live in `core/rbac.py` with three role-set constants: `PROPOSAL_ALL_ROLES`, `PROPOSAL_WRITE_ROLES`, `PROPOSAL_BID_MANAGER_ONLY`. Story 10.4 consumes `PROPOSAL_WRITE_ROLES` (POST + 2 PATCH) and `PROPOSAL_ALL_ROLES` (GET + replies).
- `request.state._rbac_cache` populated by the dep — available for force-action distinctions if needed (not used in Story 10.4's MVP).
- Admin bypass: company admins without a `proposal_collaborators` row are admitted by the dep. Story 10.4 inherits.

**From Story 10.1 (`proposal_collaborators` CRUD):**
- `ProposalCollaboratorRole` enum values are stable: `bid_manager`, `technical_writer`, `financial_analyst`, `legal_reviewer`, `read_only`. Imported from `client_api.models.enums`.
- `client.proposal_collaborators` table exists; carrier-of-truth for the dep's role lookup.

**From Story 7.3 (proposal versioning API):**
- `proposal_service.py:484::create_version` is the carry-forward hook point. The function uses `_get_version_write_lock(proposal_id)` async lock + `_get_proposal_for_update` (SELECT FOR UPDATE on proposal row) for serialisation. The hook MUST execute inside both layers, which it does naturally if placed before `return version`.
- Two `session.flush()` calls per version creation: first for the new `proposal_versions` INSERT, second for the `proposals.current_version_id` UPDATE. Story 10.4's hook runs after both flushes complete.
- `proposal.current_version_id` is the prior version pointer; it is reassigned to the new version's id at line 547. Story 10.4's hook MUST capture it BEFORE this reassignment.

**From Story 7.4 (content-save API):**
- `schemas/proposal_content.py::SectionItem` defines the `{key, title, body}` section shape. Story 10.4's `section_key` field aligns with `SectionItem.key`.
- The 409 `version_conflict` body via `JSONResponse` is referenced but not used in Story 10.4 (no conflict semantics on comments).

**From Story 7.2 (proposal CRUD):**
- `client.proposals` table; tenant scoping via `company_id`. Cross-company existence-leakage guard returns 404. Story 10.4 inherits via the dep.
- `client.proposal_versions` table with FK to proposals; `version_number` unique per proposal. Story 10.4's `version_id` FK references this.

**From Story 2.11 (audit-log middleware):**
- `write_audit_entry` signature: `(session, *, user_id, action_type, entity_type, entity_id, before, after, ip_address, company_id, correlation_id)`. All optional except `action_type`. Body content NEVER goes in `before`/`after` — use `body_length` for PII guard.
- `_write_denial_audit` for 403 paths.

### Git Intelligence — Recent Relevant Commits

- `(Story 10.3 merge)` — lands `proposal_section_locks` table + router + service + Celery cleanup task. Story 10.4 mirrors the per-section-state shape and the `_get_version_write_lock` integration pattern (though for a different lock).
- `(Story 10.2 merge)` — lands `require_proposal_role` + creator-seed + auto-seed-conftest pattern + the renamed route-dependency snapshot test. Story 10.4 consumes all four.
- `(Story 10.1 merge)` — lands `proposal_collaborators` table + `ProposalCollaboratorRole` enum + sibling router + service helpers.
- `(Story 7.3 merge)` — lands `create_version` (the carry-forward hook point) + `_get_version_write_lock` registry.
- `(Story 7.4 merge)` — lands the section-level PATCH and 409-via-`JSONResponse` conflict-body pattern (referenced but not used here).

Implementation of 10.4 is **mostly additive with one targeted modification**: one new migration, one new ORM model, one new service, one new router, four new test files, plus one targeted edit to `proposal_service.create_version` for the carry-forward hook (with a deferred-import to avoid circular dependency) and the standard router-registration edit to `main.py`. The `proposal_service` modification is the FIRST cross-story service touch in Epic 10 — handle the diff carefully and verify Story 7.3's existing version-creation tests still pass.

### Project Structure Notes

- **Alignment:** All new files follow the existing client-api layout (`models/`, `schemas/`, `services/`, `api/v1/`, `tests/unit/`, `tests/api/`, `tests/integration/`). No new top-level directories.
- **Migration numbering:** 038 is the next slot after Story 10.3's 037.
- **BMAD alignment:** Story 10.4 is the FOURTH story of Epic 10. After merge, `sprint-status.yaml` will update `10-4-proposal-comments-api: done`; `current_story` advances to `10-5-task-crud-api`. `epic-10` remains `in-progress`.
- **Variances and rationale:**
  - **`test-design-epic-10.md` absence (again):** Documented in Dev Notes §Test Expectations; surrogate sources cited (Epic 10 AC3 + AC13 verbatim; R-006 RBAC tie-in). Still flagged for TEA backlog.
  - **First Epic 10 story to modify an existing service path (`proposal_service.create_version`):** Documented in §Description (sixth subtle point) and §Tasks Task 5. Deferred-import dance + fail-loud on carry-forward error are explicit decisions.
  - **No DELETE endpoint for comments:** Documented in §Design Constraints. MVP scope; can be added later without schema migration (just an endpoint + service function).
  - **No edit endpoint for comments:** Documented in §Design Constraints. MVP scope; would require an edit-history table to be added without losing audit-grade behaviour.
  - **Parent-comment re-linking on carry-forward is OUT OF SCOPE:** Documented in AC9 and §Design Constraints. Trade-off: `carried_forward_from_id` chain preserves the audit trail; recursive-CTE relinker can be added in a follow-up.
- **No conflicts** with Stories 7.2, 7.3, 7.4, 10.1, 10.2, 10.3, 9.3, or 2.11 conventions.

### References

- Epic: [Source: eusolicit-docs/planning-artifacts/epic-10-collaboration-tasks-approvals.md#S10.04]
- Epic AC closed: AC3 (comments create + list per section + resolve toggle + version carry-forward); partial AC13 (comments axis of proposal-level RBAC).
- Test Design (architecture — R-006 RBAC tie-in): [Source: eusolicit-docs/test-artifacts/test-design-architecture.md#L112 (R-006 entity-level RBAC bypass)]
- Test Design (epic-10 absence — TEA backlog flag): [Source: eusolicit-docs/test-artifacts/ — no `test-design-epic-10.md` present at story-creation time]
- Previous story 10.1 (collaborator CRUD): [Source: eusolicit-docs/implementation-artifacts/10-1-proposal-collaborator-crud-api.md]
- Previous story 10.2 (`require_proposal_role` + role-set constants): [Source: eusolicit-docs/implementation-artifacts/10-2-proposal-level-rbac-middleware.md]
- Previous story 10.3 (per-section state precedent + section-lock router pattern): [Source: eusolicit-docs/implementation-artifacts/10-3-section-locking-api.md]
- Previous story 7.3 (versioning API + `create_version` hook point): [Source: eusolicit-docs/implementation-artifacts/7-3-proposal-versioning-api.md]
- Previous story 7.4 (section auto-save + `JSONResponse` conflict-body convention): [Source: eusolicit-docs/implementation-artifacts/7-4-proposal-content-save-api-auto-save-full-save.md]
- Existing `require_proposal_role` + role-set constants: [Source: eusolicit-app/services/client-api/src/client_api/core/rbac.py]
- Existing `create_version` hook point: [Source: eusolicit-app/services/client-api/src/client_api/services/proposal_service.py:484-571]
- Existing `_get_version_write_lock` lock registry: [Source: eusolicit-app/services/client-api/src/client_api/services/proposal_service.py (search for `_get_version_write_lock`)]
- Existing `SectionItem` schema: [Source: eusolicit-app/services/client-api/src/client_api/schemas/proposal_content.py]
- Existing `ContentConflictDetail` 409 pattern (referenced, not used): [Source: eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py]
- Existing `write_audit_entry` signature: [Source: eusolicit-app/services/client-api/src/client_api/services/audit_service.py]
- Existing router-registration pattern: [Source: eusolicit-app/services/client-api/src/client_api/main.py:121 (section-locks include with `prefix="/proposals"`)]
- Migration 035 (schema-qualified DDL template): [Source: eusolicit-app/services/client-api/alembic/versions/035_proposal_collaborators.py]
- Migration 037 (immediately preceding 038 — section-locks template): [Source: eusolicit-app/services/client-api/alembic/versions/037_proposal_section_locks.py]
- Existing `ProposalCollaborator` ORM (style reference): [Source: eusolicit-app/services/client-api/src/client_api/models/proposal_collaborator.py]
- Existing `ProposalSectionLock` ORM (style reference, same per-section convention): [Source: eusolicit-app/services/client-api/src/client_api/models/proposal_section_lock.py]
- `ProposalCollaboratorRole` enum: [Source: eusolicit-app/services/client-api/src/client_api/models/enums.py]
- Project conventions: [Source: CLAUDE.md].
- Forward-reference: Epic 10 Story 10.13 "Comments Sidebar UI" — consumes `GET /comments?version_id=...&section_key=...` for the per-section sidebar, posts via `POST /comments`, toggles via `PATCH /resolve` and `PATCH /unresolve`, displays the "carried-forward" banner when a new version is created and the carry-forward INSERT count is non-zero.

### Open Questions for PM / TEA (for follow-up, not blocking)

1. **Should comments support edits (with edit history)?** Current decision: NO (MVP). Body is immutable after create. If the product wants edit-history support, a follow-up story adds `comment_edits` table + `PATCH /comments/{id}` endpoint + edit-grade audit. Flag for PM.
2. **Should comments support deletion (soft-delete or admin-purge)?** Current decision: NO (MVP). If a moderation requirement materialises, add `deleted_at` column + `DELETE /comments/{id}` (bid_manager-or-admin-only). Trivial migration; no data loss because `audit_log` retains history.
3. **Should reply threading carry forward across versions?** Current decision: NO (top-level on new version). The `carried_forward_from_id` chain preserves the audit trail. If the §10.13 UI requires nested-thread continuity across versions, a follow-up story adds a recursive-CTE relinker that maps old parent IDs → new clone IDs in a second pass after the bulk INSERT. Flag for §10.13 dev.
4. **Should the API expose a "mention" mechanism (`@user_id` in body)?** Not in scope. The body is plain text. If mentions are added later, they become a separate `comment_mentions` table linking comment_id → user_id, populated by parsing the body on create. The body remains plain text storage-wise.
5. **Should the GET endpoint support pagination?** Current decision: NO. We assume per-section comment counts stay small (typically <20 per section per version). If a heavy-use proposal hits hundreds, add `limit`/`offset` query params in a follow-up. The composite index `(proposal_id, version_id, section_key, created_at)` is already laid out to support `LIMIT N OFFSET M` efficiently.
6. **Should the carry-forward emit an event on the bus (e.g., `proposal.version.comments_carried`)?** Current decision: NO (structlog only). If §10.13 needs real-time cross-tab notification ("this version was created with N carried-forward comments"), add a Redis Streams emit in a follow-up. The hook point in `create_version` is the natural producer.
7. **Should the 422 body-length validation produce a custom error body?** Current: standard Pydantic 422 detail array. If §10.13 wants a friendlier message, add a custom validator with `field_validator` in `CommentCreateRequest`. Defer to UX feedback.
8. **Should `body` allow Markdown / HTML / sanitised rich text?** Current: plain text storage; rendering responsibility on the client. If §10.13 wants Markdown rendering, the client parses on display; the API is content-agnostic. If the product wants server-side sanitisation (XSS guard), add a sanitisation pass in the service before INSERT in a follow-up.

## Change Log

- 2026-04-20: Initial story context created. (bmad-create-story)
- 2026-04-20: Test coverage gap resolved. All blocking findings from Senior Developer Review addressed:
  - AC12: `tests/api/test_proposal_comments.py` expanded to ~25 test cases (happy paths, validation, RBAC, idempotency, reply threading).
  - AC13: `tests/api/test_proposal_comments_carry_forward.py` expanded to 8 cases including failure-mode rollback test (monkeypatched carry-forward + no-v2-row assertion) and all edge cases.
  - AC14: `tests/unit/test_proposal_comment_service.py` expanded to 15 tests covering all filter combinations, `list_comment_replies` ordering/empty, carry-forward authorship/parent-nulling/carried_forward_from_id/count.
  - AC15: `tests/api/test_comments_rbac_matrix.py` rewritten to 8 profiles × 5 endpoints = 40 assertions (added `financial_analyst`/`legal_reviewer` profiles, added 4 parametrised endpoint tests).
  - AC16: `tests/integration/test_migration_038.py` created with 10 test IDs (upgrade clean, columns, 4 indexes, 2 CHECK constraints, 3 FK ON DELETE policies, downgrade+re-upgrade idempotency).
  - AC17: `tests/unit/test_proposals_router_dependency_spec.py` extended with 5 new `/comments*` route entries.
  - `tests/api/conftest.py` extended with `financial_analyst`/`legal_reviewer` collaborators in `section_lock_ctx` and `make_comment` helper fixture.
  - All ruff check + ruff format violations resolved. Status: **complete**. (bmad-dev-story)

## Senior Developer Review

**REVIEW: Changes Requested**

The functional implementation of the Proposal Comments API mostly aligns with the architecture and requirements, and test coverage has been significantly improved. However, adversarial review revealed several critical issues that required remediation:

**Findings:**
1. **Contradictory Spec (AC1):** The acceptance criteria defined a `ck_proposal_comments_resolved_consistency` CHECK constraint that conflicted with the `ON DELETE SET NULL` behavior for `resolved_by`. This caused the `test_038_resolved_by_set_null_on_delete` test to fail with a CheckViolationError when a user who resolved a comment was deleted. The constraint was modified to `(resolved = TRUE AND resolved_at IS NOT NULL)` to resolve this.
2. **Missing Rollback Handling:** The `carry_forward_unresolved_comments` call in `create_version` was not wrapped in a try/except block to properly log the failure and re-raise, causing the endpoint to bubble up a `RuntimeError` directly instead of handling the failure gracefully and rolling back the transaction.
3. **Audit Log Schema Mismatch:** Multiple tests queried `client.audit_log` instead of `shared.audit_log`, and incorrectly referenced `after_data` and `created_at` instead of `after` and `timestamp`.
4. **Ambiguous Column Reference:** The migration test `test_038_body_length_check_constraint` contained an ambiguous `id` reference in a JOIN query.

These issues have been fixed in the codebase, but the contradictory spec must be officially flagged for the product/architecture team.

## Known Deviations

### Detected by `bmad-code-review` at 2026-04-20

- Contradictory spec in AC1: The `ck_proposal_comments_resolved_consistency` CHECK constraint conflicts with the `ON DELETE SET NULL` policy for `resolved_by`. When a user who resolved a comment is deleted, `resolved_by` becomes NULL, violating the specified constraint `(resolved = TRUE AND resolved_by IS NOT NULL AND resolved_at IS NOT NULL)`.
  _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-20T17:46:47Z

- Contradictory spec in AC1: The `ck_proposal_comments_resolved_consistency` CHECK constraint conflicts with the `ON DELETE SET NULL` policy for `resolved_by`. When a user who resolved a comment is deleted, `resolved_by` becomes NULL, violating the specified constraint `(resolved = TRUE AND resolved_by IS NOT NULL AND resolved_at IS NOT NULL)`. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-20T18:06:55Z

- The `ck_proposal_comments_resolved_consistency` CHECK constraint specified in AC1 conflicts with the `ON DELETE SET NULL` policy for `resolved_by`. When a user who resolved a comment is deleted, `resolved_by` becomes NULL, which would violate the originally specified constraint `(resolved = TRUE AND resolved_by IS NOT NULL AND resolved_at IS NOT NULL)`. The implementation pragmatically resolved this by removing the `resolved_by IS NOT NULL` check. _(type: `CONTRADICTORY_SPEC`)_
