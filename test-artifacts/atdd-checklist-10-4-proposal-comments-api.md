---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
lastStep: step-04c-aggregate
lastSaved: '2026-04-20'
workflowType: bmad-testarch-atdd
mode: create
storyId: 10-4-proposal-comments-api
detectedStack: backend
generationMode: AI Generation (backend — no browser recording)
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/10-4-proposal-comments-api.md
  - eusolicit-docs/test-artifacts/test-design-architecture.md
  - eusolicit-docs/test-artifacts/test-design-qa.md
  - eusolicit-docs/test-artifacts/atdd-checklist-10-3-section-locking-api.md
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/tests/api/conftest.py
  - eusolicit-app/services/client-api/tests/api/test_proposal_section_locks.py
---

# ATDD Checklist: Story 10.4 — Proposal Comments API

**Date:** 2026-04-20
**Author:** BMAD TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated; feature not yet implemented
**Story Status:** ready-for-dev

---

## Step 1: Preflight & Context

### Stack Detection

- **Detected stack:** `backend`
- **Detection evidence:** FastAPI + SQLAlchemy + Alembic stack; pytest + pytest-asyncio + httpx AsyncClient test harness; no browser dependencies.
- **Test framework:** pytest 8.x + pytest-asyncio + httpx AsyncClient

### Prerequisites Check

| Requirement | Status | Notes |
|---|---|---|
| Story has clear acceptance criteria | ✅ | 18 ACs with full task breakdown |
| pytest conftest with async session fixture | ✅ | `client_api_session_factory` + `client_session` in tests/conftest.py |
| `section_lock_ctx` fixture available | ✅ | `tests/api/conftest.py` — 5-role + non-collab + cross-company environment (Story 10.3) |
| `require_proposal_role` + `PROPOSAL_WRITE_ROLES` / `PROPOSAL_ALL_ROLES` | ✅ | `core/rbac.py` established by Story 10.2 |
| `client.proposals` + `create_version` hook point | ✅ | `proposal_service.py:484` from Story 7.3 |
| `_get_version_write_lock` lock registry | ✅ | `proposal_service.py` from Story 7.3 |
| `write_audit_entry` / `_write_denial_audit` | ✅ | `audit_service.py` from Story 2.11 |
| Migration 037 (`proposal_section_locks`) | ✅ | Previous slot; 038 is this story's slot |
| `ProposalComment` ORM model | ❌ | **Does not exist** — blocked on AC1/AC2 Task 1 |
| `client.proposal_comments` table | ❌ | **Does not exist** — blocked on migration 038 |
| `CommentCreateRequest` / `CommentResponse` / `CommentListResponse` | ❌ | **Does not exist** — blocked on AC3 Task 2 |
| `proposal_comment_service.py` | ❌ | **Does not exist** — blocked on AC9/AC10 Task 3 |
| `api/v1/proposal_comments.py` router (5 endpoints) | ❌ | **Does not exist** — blocked on AC4–AC7/AC10/AC11 Task 4 |
| Carry-forward hook in `proposal_service.create_version` | ❌ | **Does not exist** — blocked on AC8 Task 5 |

### Test Design Sources

Per Story 10.4 Dev Notes (§Test Expectations), no `test-design-epic-10.md` exists at story-creation time. Test expectations are drawn from:

- **Epic 10 AC3 (verbatim)** — "Comments can be created on a section+version, listed per section, resolved/unresolved, and unresolved comments carry forward when a new version is created." Closed by this story.
- **Epic 10 AC13 (verbatim)** — Proposal-level RBAC enforcement on all mutation endpoints; `read_only` denied mutation, `PROPOSAL_ALL_ROLES` can read.
- **`test-design-architecture.md` R-006** — "Entity-Level RBAC bypass on per-proposal collaborator permissions" (score 6); mandates 40-assertion matrix across 5 endpoints × 8 caller profiles (AC15); partially closed for the comments axis by this story.
- **`test-design-qa.md` P1-007** — Section-level state locking precedent; the comments carry-forward is the analogous per-section-state carry-over concern.
- **Story 10.3 surrogate pattern** — `section_lock_ctx` fixture, split PATCH-by-action (acquire/release), idempotency no-op audit suppression. Story 10.4 mirrors these conventions on all 5 endpoints.
- **Story 7.3 precedent** — `create_version` transactional discipline; carry-forward hook must be inside the existing `_get_version_write_lock` block. Failure-mode test (monkeypatch raise → 500 + rollback) parallels the Story 7.3 conflict-body pattern.

---

## Step 2: Generation Mode

**Selected mode:** AI Generation (backend — no browser recording)

**Rationale:** Pure backend API; no browser automation needed. Test patterns follow `test_proposal_section_locks.py`, `test_proposal_collaborators_crud.py`, and `test_audit_trail.py`. The carry-forward integration test uses the `httpx.AsyncClient` end-to-end pattern (POST /versions → assert DB state) established by `test_proposal_versions.py`.

---

## Step 3: Test Strategy

| Priority | Area | Scenarios | Harness |
|---|---|---|---|
| **P0** | Create comment — happy paths (top-level + reply) | 3 | `test_proposal_comments.py` — 201 + CommentResponse + DB + audit + structlog |
| **P0** | Create comment — validation failures (version_id, parent, body) | 8 | `test_proposal_comments.py` — 400/422/403/404 |
| **P0** | List comments — happy paths (with/without filters) | 5 | `test_proposal_comments.py` — 200 + counts + ordering |
| **P0** | Resolve / unresolve — happy + idempotent | 4 | `test_proposal_comments.py` — 200 + state + audit |
| **P0** | Replies endpoint | 3 | `test_proposal_comments.py` — happy + 404 + empty |
| **P0** | Carry-forward — main flow (mixed resolved/unresolved seed) | 2 | `test_proposal_comments_carry_forward.py` — v2 has 4 rows; v1 unchanged |
| **P0** | Carry-forward — failure mode → rollback (no orphan v2) | 1 | `test_proposal_comments_carry_forward.py` — monkeypatch raise |
| **P0** | RBAC matrix — 5 endpoints × 8 profiles | 40 | `test_comments_rbac_matrix.py` |
| **P1** | Carry-forward — edge cases (empty, all-resolved, first version) | 3 | `test_proposal_comments_carry_forward.py` |
| **P1** | Audit log shape (create/resolve/unresolve; idempotency no-op) | 5 | `test_proposal_comments.py` — audit_log row assertions |
| **P1** | Structlog events (PII guard — body absent) | 2 | unit test + API test |
| **P1** | Service-layer unit tests | 12 | `test_proposal_comment_service.py` |
| **P2** | ORM model structure | 4 | `test_proposal_comment_model.py` |
| **P2** | Route-dependency snapshot (5 new routes) | 1 | `test_proposals_router_dependency_spec.py` (extended) |
| **Integration** | Migration 038 up/down + CHECK constraints + FK ON DELETE | 7 | `test_migration_038.py` |

**Total:** ~60+ test assertions across 7 test files; ~36+ test functions.

---

## Step 4: Generated Test Files

### Unit Tests

#### `tests/unit/test_proposal_comment_model.py`

- `test_proposal_comment_tablename` — asserts `ProposalComment.__tablename__ == "proposal_comments"`
- `test_proposal_comment_schema` — asserts `__table_args__` includes `{"schema": "client"}`
- `test_proposal_comment_indexes_present` — asserts all 4 index names (`ix_proposal_comments_proposal_id`, `ix_proposal_comments_version_section`, `ix_proposal_comments_resolved`, `ix_proposal_comments_parent`) are in `__table_args__`
- `test_proposal_comment_check_constraint_present` — asserts `ck_proposal_comments_resolved_consistency` CHECK in `__table_args__`
- `test_proposal_comment_nullable_columns` — `parent_comment_id`, `resolved_by`, `resolved_at`, `carried_forward_from_id` are nullable; `proposal_id`, `version_id`, `section_key`, `author_id`, `body`, `resolved`, `created_at`, `updated_at` are NOT NULL
- `test_proposal_comment_resolved_default_false` — `resolved` column default is `False`

#### `tests/unit/test_proposal_comment_service.py`

- `test_create_comment_inserts_row_with_correct_fields` — call `create_comment(...)` on a mocked session; assert `ProposalComment` row constructed with `resolved=False`, `author_id` = caller, `body` = input body, `version_id` = supplied id
- `test_create_comment_returns_author_full_name` — stub `JOIN client.users`; assert `ProposalCommentWithUser.author_full_name` is populated
- `test_create_comment_body_never_in_structlog` — assert structlog `proposal_comment.created` event does NOT contain a `body` key (PII guard)
- `test_list_comments_filter_by_section_key` — seed 5 rows across 2 sections; assert service returns only the 3 matching `section_key='intro'`
- `test_list_comments_filter_by_resolved_true` — seed 2 resolved + 3 open; assert service returns only 2 rows when `resolved=True`
- `test_list_comments_filter_by_resolved_false` — assert 3 rows returned when `resolved=False`
- `test_list_comments_no_filters_returns_all` — assert all 5 rows returned
- `test_list_comments_returns_correct_total_and_unresolved_count` — mixed seed: assert `(total=5, unresolved_count=3)`
- `test_list_comments_ordered_by_created_at_asc` — seed rows with non-monotone timestamps; assert output order matches created_at ASC
- `test_resolve_comment_transitions_to_resolved` — open comment → `resolve_comment(...)` returns `(comment, transitioned=True)`; row has `resolved=True`, `resolved_by` set, `resolved_at` set
- `test_resolve_comment_idempotent_no_op` — already-resolved comment → `resolve_comment(...)` returns `(comment, transitioned=False)`; no state change
- `test_unresolve_comment_transitions_to_unresolved` — resolved comment → `unresolve_comment(...)` returns `(comment, transitioned=True)`; `resolved=False`, `resolved_by=None`, `resolved_at=None`
- `test_unresolve_comment_idempotent_no_op` — already-open comment → `unresolve_comment(...)` returns `(comment, transitioned=False)`
- `test_list_comment_replies_returns_ordered_children` — seed parent + 2 replies with different timestamps; assert replies ordered by `created_at ASC`
- `test_list_comment_replies_empty_for_childless_parent` — parent with no children → returns `[]`
- `test_carry_forward_copies_only_unresolved_rows` — 3 unresolved + 2 resolved on v1; `carry_forward_unresolved_comments(...)` returns 3; only 3 rows written for v2; resolved rows absent
- `test_carry_forward_preserves_author_id_not_trigger_user` — `author_id` on cloned rows matches original authors, NOT the `triggered_by` user
- `test_carry_forward_nulls_parent_comment_id` — original threaded replies have `parent_comment_id` set; clones have `parent_comment_id=NULL`
- `test_carry_forward_populates_carried_forward_from_id` — each clone's `carried_forward_from_id` == the originating v1 row's `id`
- `test_carry_forward_returns_correct_count` — seed 4 unresolved; assert return value == 4
- `test_carry_forward_empty_source_returns_zero` — no comments on v1 → returns 0

#### `tests/unit/test_proposals_router_dependency_spec.py` *(extended — AC17)*

- `test_router_dependency_spec_includes_5_new_comment_routes` — assert snapshot dict now contains all 5 new entries:
  - `POST /api/v1/proposals/{proposal_id}/comments` → `require_proposal_role:PROPOSAL_WRITE_ROLES`
  - `GET /api/v1/proposals/{proposal_id}/comments` → `require_proposal_role:PROPOSAL_ALL_ROLES`
  - `PATCH /api/v1/proposals/{proposal_id}/comments/{comment_id}/resolve` → `require_proposal_role:PROPOSAL_WRITE_ROLES`
  - `PATCH /api/v1/proposals/{proposal_id}/comments/{comment_id}/unresolve` → `require_proposal_role:PROPOSAL_WRITE_ROLES`
  - `GET /api/v1/proposals/{proposal_id}/comments/{comment_id}/replies` → `require_proposal_role:PROPOSAL_ALL_ROLES`
- `test_router_dependency_spec_pre_existing_routes_unchanged` — all routes present in the pre-10.4 snapshot still present with unchanged dependency tags

---

### API Tests

#### `tests/api/test_proposal_comments.py` *(AC12)*

**Setup fixture note:** Requires `make_comment(proposal_id, version_id, section_key, author_id, *, body="Test comment.", resolved=False, parent_comment_id=None)` helper in `tests/api/conftest.py` (creates a row directly via ORM). Also requires `comments_ctx` (or reuse `section_lock_ctx`) seeding company + admin + 5 collaborator roles + non-collab + cross-company.

**Happy-path create:**

- `test_happy_create_top_level_comment_bid_manager` — bid_manager POSTs `{version_id, section_key="intro", body="Nice section"}` → 201 + `CommentResponse` with `id`, `proposal_id`, `version_id`, `section_key`, `author_id`, `body`, `resolved=False`, `author_full_name` populated
- `test_happy_create_top_level_comment_db_row_present` — after 201, query DB; assert `proposal_comments` row exists with correct fields
- `test_happy_create_audit_row_present` — assert one `audit_log` row with `action_type="create"`, `entity_type="proposal_comment"`, `entity_id=<comment_id>`; assert `after.body_length` present but `after.body` absent (PII guard)
- `test_happy_create_structlog_event_captured` — capfd / structlog capture; assert `proposal_comment.created` event has `is_reply=False`, no `body` key
- `test_happy_create_reply_parent_linkage_correct` — create parent comment; then POST with `parent_comment_id=<parent_id>` → 201; assert `parent_comment_id` in response matches; structlog `is_reply=True`

**Parent validation failures:**

- `test_create_reply_cross_section_parent_returns_400` — parent is on `section_key="intro"`, reply attempts `section_key="scope"` → 400 `detail="parent_comment_id does not belong to this proposal+version+section"`
- `test_create_reply_cross_version_parent_returns_400` — parent is on `version_id=v1`, reply body has `version_id=v2` → 400
- `test_create_reply_cross_proposal_parent_returns_400` — parent exists on proposal X; attempt to thread on proposal Y → 400

**Version validation:**

- `test_create_with_version_id_from_different_proposal_returns_400` — `version_id` belongs to proposal B but `proposal_id` is A → 400 `detail="version_id does not belong to this proposal"`
- `test_list_with_version_id_from_different_proposal_returns_400` — GET with crafted `version_id` → 400 (tenant guard on read)

**Body validation (422):**

- `test_create_empty_body_returns_422` — body="" → 422 Pydantic validation error
- `test_create_whitespace_only_body_returns_422` — body=" " (whitespace, min_length=1) → 422
- `test_create_oversized_body_returns_422` — body of 10001 chars → 422

**RBAC on create:**

- `test_create_as_read_only_returns_403` — read_only collaborator → 403
- `test_create_as_non_collaborator_returns_403` — authenticated company member without collaborator row → 403
- `test_create_cross_company_returns_404` — company-B caller → 404
- `test_create_as_company_admin_no_collaborator_row_returns_201` — admin bypass → 201

**List happy paths:**

- `test_list_happy_path_section_filter_ordered_asc` — 5 comments seeded: 3 on `section_key="intro"`, 2 on `section_key="scope"` (all unresolved); GET `?version_id=<v1>&section_key=intro` → 200 + `total=3`, `unresolved_count=3`, `comments` in `created_at` ASC order
- `test_list_happy_path_no_section_filter_returns_all` — GET `?version_id=<v1>` (no `section_key`) → `total=5`, `comments` length 5
- `test_list_filter_resolved_true` — 2 resolved comments; GET `?version_id=<v1>&resolved=true` → returns only resolved rows
- `test_list_filter_resolved_false` — GET `?version_id=<v1>&resolved=false` → returns only open rows; count matches `unresolved_count`
- `test_list_empty_proposal_returns_200_not_404` — proposal with no comments; GET `?version_id=<v1>` → 200 + `total=0`, `unresolved_count=0`, `comments=[]`
- `test_list_as_read_only_returns_200` — read_only collaborator → 200 (read-only CAN read comments)

**Resolve:**

- `test_resolve_happy_path` — open comment → PATCH `.../resolve` → 200; `resolved=True`, `resolved_by` and `resolved_at` populated in response
- `test_resolve_writes_audit_row` — assert `audit_log` row: `action_type="update"`, `before={"resolved": false}`, `after={"resolved": true, "resolved_by": "<uid>", "resolved_at": "<iso>"}`
- `test_resolve_idempotent_already_resolved_returns_200_no_audit` — already-resolved → 200 no-op; assert NO new audit_log row created; assert NO new structlog event emitted
- `test_resolve_as_read_only_returns_403` — read_only collaborator → 403
- `test_resolve_non_existent_comment_returns_404` — random UUID → 404 `detail="Comment not found"`
- `test_resolve_mismatched_proposal_id_returns_404` — comment exists on proposal X; attempt PATCH on proposal Y → 404 (tenant defence-in-depth)

**Unresolve:**

- `test_unresolve_happy_path` — resolved comment → PATCH `.../unresolve` → 200; `resolved=False`, `resolved_by=None`, `resolved_at=None`
- `test_unresolve_writes_audit_row` — assert audit row with `before={"resolved": true, ...}`, `after={"resolved": false}`
- `test_unresolve_idempotent_already_open_returns_200_no_audit` — already-unresolved → 200 no-op; no new audit row
- `test_unresolve_as_read_only_returns_403` — read_only → 403

**Replies endpoint:**

- `test_replies_happy_path_returns_ordered_list` — parent + 2 replies seeded; GET `.../replies` → 200 + 2 replies in `created_at` ASC order
- `test_replies_on_non_existent_parent_returns_404` — unknown `comment_id` → 404
- `test_replies_on_childless_parent_returns_200_empty_list` — parent with no children → 200 + `[]`

---

#### `tests/api/test_proposal_comments_carry_forward.py` *(AC13)*

**Setup:** Uses `make_comment` and `make_comments_for_version` fixtures; proposal with `v1` version.

- `test_carry_forward_happy_path_mixed_seed` — seed v1: 3 unresolved + 2 resolved on `section_intro`, 1 unresolved + 1 resolved on `section_scope`; POST `/api/v1/proposals/{id}/versions` → 201 (v2 created); query `proposal_comments WHERE version_id=v2`: assert exactly 4 rows; 3 with `section_key='intro'`, 1 with `section_key='scope'`; all `resolved=False`; all `carried_forward_from_id` populated to originating v1 row ids
- `test_carry_forward_preserves_author_id_not_version_creator` — assert v2 comment `author_id` matches original comment `author_id`, NOT the version-creator's user id
- `test_carry_forward_parent_comment_id_nulled_on_clones` — original replies have `parent_comment_id` set; v2 clones have `parent_comment_id=NULL`
- `test_carry_forward_v1_rows_unchanged` — after carry-forward, v1 still has all 7 original rows; none modified
- `test_carry_forward_audit_row_includes_carried_count` — version-creation audit row's `after.unresolved_comments_carried == 4` (or separate `proposal_comment_carry_forward` audit row present — either is acceptable per AC8)
- `test_carry_forward_structlog_event_emitted` — `proposal.version_created.comments_carried_forward` event captured with `carried_count=4`, `prev_version_id`, `new_version_id`, `proposal_id`
- `test_carry_forward_edge_empty_source_zero_carried` — proposal with no comments on v1 → version v2 creation succeeds; query `proposal_comments WHERE version_id=v2` → 0 rows; structlog event emitted with `carried_count=0`
- `test_carry_forward_edge_all_resolved_zero_carried` — all 3 v1 comments are resolved → 0 rows cloned to v2; `carried_count=0`
- `test_carry_forward_edge_first_version_no_hook_invoked` — initial version creation (no prior version) → carry-forward NOT invoked; no `proposal.version_created.comments_carried_forward` structlog event; no comments cloned
- `test_carry_forward_failure_mode_version_creation_rolls_back` — monkeypatch `carry_forward_unresolved_comments` to raise `RuntimeError`; POST `/api/v1/proposals/{id}/versions` → 500 (or 5xx per exception mapping); assert NO `proposal_versions` row for the attempted v2 exists in DB (transaction rolled back — atomicity guarantee)

---

#### `tests/api/test_comments_rbac_matrix.py` *(AC15)*

**Matrix:** 5 endpoints × 8 caller profiles = 40 assertions. Reuses `section_lock_ctx` fixture shape (admin, bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only, non-collaborator, cross-company). Requires a seeded comment in the proposal (use `make_comment` fixture) for PATCH and replies endpoints.

**POST `/proposals/{id}/comments` (requires `PROPOSAL_WRITE_ROLES`):**

- `test_rbac_create_admin_returns_201`
- `test_rbac_create_bid_manager_returns_201`
- `test_rbac_create_technical_writer_returns_201`
- `test_rbac_create_financial_analyst_returns_201`
- `test_rbac_create_legal_reviewer_returns_201`
- `test_rbac_create_read_only_returns_403` *(denial audit spot-check)*
- `test_rbac_create_non_collaborator_returns_403` *(denial audit spot-check)*
- `test_rbac_create_cross_company_returns_404`

**GET `/proposals/{id}/comments` (requires `PROPOSAL_ALL_ROLES`):**

- `test_rbac_list_admin_returns_200`
- `test_rbac_list_bid_manager_returns_200`
- `test_rbac_list_technical_writer_returns_200`
- `test_rbac_list_financial_analyst_returns_200`
- `test_rbac_list_legal_reviewer_returns_200`
- `test_rbac_list_read_only_returns_200`
- `test_rbac_list_non_collaborator_returns_403`
- `test_rbac_list_cross_company_returns_404`

**PATCH `.../resolve` (requires `PROPOSAL_WRITE_ROLES`):**

- `test_rbac_resolve_admin_returns_200`
- `test_rbac_resolve_bid_manager_returns_200`
- `test_rbac_resolve_technical_writer_returns_200`
- `test_rbac_resolve_financial_analyst_returns_200`
- `test_rbac_resolve_legal_reviewer_returns_200`
- `test_rbac_resolve_read_only_returns_403` *(denial audit spot-check)*
- `test_rbac_resolve_non_collaborator_returns_403`
- `test_rbac_resolve_cross_company_returns_404`

**PATCH `.../unresolve` (requires `PROPOSAL_WRITE_ROLES`):**

- `test_rbac_unresolve_admin_returns_200`
- `test_rbac_unresolve_bid_manager_returns_200`
- `test_rbac_unresolve_technical_writer_returns_200`
- `test_rbac_unresolve_financial_analyst_returns_200`
- `test_rbac_unresolve_legal_reviewer_returns_200`
- `test_rbac_unresolve_read_only_returns_403`
- `test_rbac_unresolve_non_collaborator_returns_403`
- `test_rbac_unresolve_cross_company_returns_404`

**GET `.../replies` (requires `PROPOSAL_ALL_ROLES`):**

- `test_rbac_replies_admin_returns_200`
- `test_rbac_replies_bid_manager_returns_200`
- `test_rbac_replies_technical_writer_returns_200`
- `test_rbac_replies_financial_analyst_returns_200`
- `test_rbac_replies_legal_reviewer_returns_200`
- `test_rbac_replies_read_only_returns_200`
- `test_rbac_replies_non_collaborator_returns_403`
- `test_rbac_replies_cross_company_returns_404`

**Denial audit spot-checks (3 of the 12 × 403 cases):**

- `test_rbac_denial_audit_read_only_create` — assert `_write_denial_audit` audit row present after read_only → 403 on POST
- `test_rbac_denial_audit_read_only_resolve` — assert denial audit row after read_only → 403 on PATCH resolve
- `test_rbac_denial_audit_non_collab_create` — assert denial audit row after non-collaborator → 403 on POST

---

### Integration Tests

#### `tests/integration/test_migration_038.py` *(AC16)*

All tests decorated `@pytest.mark.integration`.

- `test_migration_038_applies_cleanly` — run `alembic upgrade head` from 037; assert `client.proposal_comments` table exists; assert all 4 indexes present (`ix_proposal_comments_proposal_id`, `ix_proposal_comments_version_section`, `ix_proposal_comments_resolved`, `ix_proposal_comments_parent`)
- `test_migration_038_check_constraint_exists` — assert `ck_proposal_comments_resolved_consistency` CHECK constraint is registered in DB catalog
- `test_migration_038_downgrade_to_037_removes_table` — run `alembic downgrade 037`; assert `client.proposal_comments` table does NOT exist; assert all 4 indexes absent
- `test_migration_038_reapply_idempotent` — after downgrade, `alembic upgrade head` again; table + indexes present (idempotency check)
- `test_migration_038_check_constraint_rejects_resolved_without_resolved_by` — attempt `INSERT INTO client.proposal_comments (..., resolved=TRUE, resolved_by=NULL, resolved_at=NULL, ...)`; assert `IntegrityError` raised (DB-level CHECK)
- `test_migration_038_body_length_check_rejects_oversized` — attempt INSERT with 10001-char body; assert `IntegrityError` from `char_length(body) <= 10000` CHECK
- `test_migration_038_parent_comment_cascade_on_delete` — insert parent comment + reply; DELETE parent row; assert reply row is also gone (ON DELETE CASCADE on `parent_comment_id`)
- `test_migration_038_author_restrict_on_user_delete` — insert comment with `author_id` referencing a user; attempt DELETE on that user → `IntegrityError` (ON DELETE RESTRICT)
- `test_migration_038_resolved_by_set_null_on_user_delete` — insert resolved comment with `resolved_by` referencing a user; DELETE that user; assert comment row still exists with `resolved_by=NULL` (ON DELETE SET NULL)

---

## Step 4c: Aggregate

All test files live under `eusolicit-app/services/client-api/tests/`. All tests are in 🔴 **RED** phase — they import from modules that do not yet exist (`client_api.models.proposal_comment`, `client_api.schemas.proposal_comments`, `client_api.services.proposal_comment_service`) and call endpoints that are not yet registered (`/api/v1/proposals/{proposal_id}/comments*`). They WILL FAIL until implementation tasks 1–9 are complete.

### File List

| File | Phase | Approx Count |
|---|---|---|
| `tests/unit/test_proposal_comment_model.py` | 🔴 RED | 6 |
| `tests/unit/test_proposal_comment_service.py` | 🔴 RED | 21 |
| `tests/unit/test_proposals_router_dependency_spec.py` *(extended)* | 🔴 RED | 2 new assertions |
| `tests/api/test_proposal_comments.py` | 🔴 RED | 29 |
| `tests/api/test_proposal_comments_carry_forward.py` | 🔴 RED | 10 |
| `tests/api/test_comments_rbac_matrix.py` | 🔴 RED | 40 + 3 audit spot-checks |
| `tests/integration/test_migration_038.py` | 🔴 RED | 9 |
| **Total** | | **~120 assertions / ~67 test functions** |

### New Fixtures Required in `tests/api/conftest.py`

| Fixture | Signature | Purpose |
|---|---|---|
| `make_comment` | `(proposal_id, version_id, section_key, author_id, *, body="Test comment.", resolved=False, parent_comment_id=None, carried_forward_from_id=None) -> ProposalComment` | Creates one comment row via ORM for test setup |
| `make_comments_for_version` | `(proposal_id, version_id, sections_with_counts: dict[str, tuple[int, int]]) -> dict[str, list[ProposalComment]]` | Bulk-seeds comments per section; each tuple is `(unresolved_count, resolved_count)`; returns dict keyed by `section_key` |
| `comments_ctx` | `async_generator[dict[str, Any], None]` | Extends `section_lock_ctx` shape with a seeded comment per section for RBAC matrix tests; provides `proposal_id`, `version_id`, `comment_id` |

### AC → Test Coverage Mapping

| AC | Test File(s) | Coverage |
|---|---|---|
| AC1 — Migration 038 `client.proposal_comments` | `test_migration_038.py` | Up, down, re-apply idempotent; all indexes + CHECK constraints |
| AC2 — `ProposalComment` ORM model | `test_proposal_comment_model.py` | tablename, schema, indexes, CHECK, nullable columns, defaults |
| AC3 — Pydantic schemas | `test_proposal_comment_service.py` (via service call returns), `test_proposal_comments.py` (422 body shape) | Implicit via API responses; field presence + `extra="forbid"` via 422 on unknown fields |
| AC4 — `POST /comments` endpoint | `test_proposal_comments.py` (happy create + validation), `test_comments_rbac_matrix.py` | 201 + CommentResponse + audit + structlog + 400/422/403/404 paths |
| AC5 — `GET /comments` endpoint + filters | `test_proposal_comments.py` (list scenarios), `test_comments_rbac_matrix.py` | All 4 filter combinations + counts + ordering + empty case + read_only access |
| AC6 — `PATCH .../resolve` | `test_proposal_comments.py`, `test_comments_rbac_matrix.py` | Happy + idempotent + audit + 403/404 |
| AC7 — `PATCH .../unresolve` | `test_proposal_comments.py`, `test_comments_rbac_matrix.py` | Happy + idempotent + audit + 403 |
| AC8 — Carry-forward hook in `create_version` | `test_proposal_comments_carry_forward.py` | Happy path + atomicity (failure mode) + structlog + audit |
| AC9 — `proposal_comment_service.py` public API | `test_proposal_comment_service.py` | All 5 service functions; carry-forward edge cases |
| AC10 — `list_comment_replies` + replies endpoint | `test_proposal_comment_service.py`, `test_proposal_comments.py`, `test_comments_rbac_matrix.py` | Ordering + empty + 404 |
| AC11 — 5 route registrations | `test_proposals_router_dependency_spec.py` (extended), 401-implicit via tests | All 5 routes present with correct role-set tags |
| AC12 — Comments API test matrix | `test_proposal_comments.py` | ~29 scenarios covering CRUD-ish, idempotency, validation, roles |
| AC13 — Carry-forward integration test | `test_proposal_comments_carry_forward.py` | 10 scenarios including failure-mode rollback + 3 edge cases |
| AC14 — Service-layer unit tests ≥90% branch | `test_proposal_comment_service.py` | 21 scenarios covering all service functions + carry-forward branches |
| AC15 — RBAC matrix 5×8=40 | `test_comments_rbac_matrix.py` | 40 role assertions + 3 denial audit spot-checks |
| AC16 — Migration smoke test | `test_migration_038.py` | 9 scenarios: up/down/re-apply + 2 CHECK + 3 FK ON DELETE policies |
| AC17 — Route-dependency snapshot updated | `test_proposals_router_dependency_spec.py` (extended) | 5 new routes captured; pre-existing routes stable |
| AC18 — README + OpenAPI docs | Not test-covered (documentation); verified at runtime via `/openapi.json` endpoint sanity check | N/A (manual + CI lint) |

### Risk Coverage

| Risk | Coverage |
|---|---|
| **R-006 SEC/MEDIUM-6** — Entity-level RBAC bypass | AC15 40-assertion matrix + AC12 role enforcement cases; partially closed for comments axis |
| **Epic 10 AC3** — Create + list + resolve/unresolve + carry-forward | AC12 + AC13 + AC14 — all 4 sub-assertions closed |
| **Carry-forward atomicity** — orphan version with lost conversation | `test_carry_forward_failure_mode_version_creation_rolls_back` — monkeypatch raise + DB assertion |
| **PII guard** — body content absent from audit_log + structlog | `test_create_comment_body_never_in_structlog` (unit) + `test_happy_create_audit_row_present` body_length assertion |
| **Tenant scoping / cross-proposal smuggling** — mismatched version_id or comment_id | `test_create_with_version_id_from_different_proposal_returns_400`, `test_resolve_mismatched_proposal_id_returns_404`, cross-company matrix entries |
| **Idempotency safety** — audit log not polluted by no-op resolve/unresolve | `test_resolve_idempotent_already_resolved_returns_200_no_audit`, `test_unresolve_idempotent_already_open_returns_200_no_audit` |

---

*All tests are intentionally failing (🔴 RED phase). They define what dev must implement. Do not green any test by modifying the test itself — only by completing the implementation tasks in Story 10.4.*
