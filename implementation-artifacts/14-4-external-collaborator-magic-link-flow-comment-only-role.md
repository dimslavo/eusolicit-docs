# Story 14.4: External Collaborator Magic-Link Flow + Comment-Only Role

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a Bid Manager,
I want to invite my end-client as a comment-only (or read-only) collaborator via a 30-day single-use magic link,
so that they can review the draft proposal in-app without consuming a paid Stripe seat and without seeing other workspace data.

## Acceptance Criteria

1. **AC 1 — Invite endpoint:** `POST /api/v1/workspaces/{workspace_id}/proposals/{proposal_id}/invite-external` accepts `{ email: str, role: "comment_only" | "read_only" }`, requires the caller to hold an `admin` or `bid_manager` membership on the parent workspace (membership row in `client.workspace_memberships`) AND `read` access on the proposal, and returns `201` with `{ id, email, role, expires_at, magic_link_url }`. The `magic_link_url` MUST be of the form `{frontend_url}/{locale}/external/accept?token={raw_jwt}` (locale defaults to `en` when not supplied by the caller). Invalid roles return `422`. Duplicate active invite for the same `(proposal_id, email)` returns `409` with `{"detail": "Active invitation already exists"}` — never 500.

2. **AC 2 — JWT structure:** The magic link embeds an RS256-signed JWT created by `client_api.core.security` with claims:
   - `sub` = `external_collaborator.id` (UUID string — NOT a `users.id`)
   - `email` = invitee email (lowercased)
   - `role` = `"comment_only"` or `"read_only"`
   - `workspace_id` = invited workspace UUID
   - `proposal_id` = invited proposal UUID (the only resource the token unlocks)
   - `external_collaborator` = `true` (boolean discriminator)
   - `iat` / `exp` (30 days) / `jti` = `external_collaborator.magic_link_jti` (unique, persisted in DB)
   - NO `company_id` claim, NO `subscription_tier` claim (deliberately omitted to prevent reuse via `get_current_user`).

3. **AC 3 — Persistence:** Inserts a `client.external_collaborators` row (model `ExternalCollaborator`, migration 046) with `workspace_id`, `proposal_id`, `email` (lowercased), `magic_link_jti` (the JWT `jti`), `expires_at` (now+30d UTC), `role`, `accepted_at=NULL`. The audit log is written with `action_type="create"`, `entity_type="external_collaborator"`, `company_id=current_user.company_id`, `after={"workspace_id", "proposal_id", "email", "role", "expires_at"}` via the canonical `write_audit_entry` helper (in-transaction).

4. **AC 4 — Accept landing endpoint:** `POST /api/v1/external/accept` with body `{ token: str }`:
   - 200: `{ collaborator_id, workspace_id, proposal_id, role, expires_at }` plus `Set-Cookie` is **not** used (token-based; SPA stores the raw JWT in `sessionStorage` and re-sends as `Authorization: Bearer`).
   - 401 for tampered / invalid signature / expired (claim `exp` past) / unknown `jti` / `external_collaborator != true`.
   - 410 (Gone) when `external_collaborators.accepted_at IS NOT NULL` AND the collaborator was created with single-use semantics. **Default = single-use:** `accepted_at` is stamped on first successful accept; subsequent accept calls with the same `jti` return 410.
   - 410 when `external_collaborators` row was deleted (revoked).
   - All four 401/410 paths write a non-blocking `auth.external_accept_denied` audit entry with `entity_type="external_collaborator"`, `entity_id=jti_uuid_or_null`, `after={"reason": "expired" | "revoked" | "consumed" | "invalid_signature"}`, `external_collaborator=true` flag in `after`.
   - Successful accept stamps `accepted_at = NOW()` and writes `auth.external_accept` audit entry.

5. **AC 5 — External collaborator authentication dependency:** New `get_external_collaborator(credentials: HTTPAuthorizationCredentials)` FastAPI dependency in `client_api/core/security.py` returns an `ExternalCollaboratorPrincipal` dataclass `{ collaborator_id, email, role, workspace_id, proposal_id, jti }`. It MUST validate the JWT independently of `get_current_user` (different claim shape), look up the `external_collaborators` row by `jti`, verify `expires_at > now`, verify `accepted_at IS NOT NULL` (i.e. token was accepted) — if any check fails, raise `UnauthorizedError`. Token revocation = row hard-deleted = 401.

6. **AC 6 — Scoped read access:** `GET /api/v1/proposals/{proposal_id}` MUST accept either `get_current_user` OR `get_external_collaborator`. When called with an external collaborator JWT, the response is identical in shape but **only** when `proposal_id == principal.proposal_id`; any other proposal_id returns `404` (existence-leakage protection — same status code an unauthenticated user would receive). `GET /api/v1/proposals/{proposal_id}/comments`, `GET /api/v1/proposals/{proposal_id}/versions`, `GET /api/v1/proposals/{proposal_id}/sections` (and the per-section content endpoints) MUST also accept the external principal subject to the same `proposal_id` match. **No other endpoint** in the codebase may accept an external collaborator token — list endpoints (`GET /api/v1/workspaces`, `GET /api/v1/proposals`, `GET /api/v1/opportunities`, etc.) reject with 401.

7. **AC 7 — `comment_only` write path:** `POST /api/v1/proposals/{proposal_id}/comments` (existing endpoint) MUST permit external collaborators with `role == "comment_only"`. The current `require_proposal_role(*PROPOSAL_WRITE_ROLES)` dependency does NOT cover external principals; introduce a new combined dependency `require_proposal_comment_access` that authorises (a) any `PROPOSAL_WRITE_ROLES` collaborator OR (b) an external collaborator whose `proposal_id` matches the path AND `role == "comment_only"`. The audit row written for the comment uses `user_id=NULL`, sets `entity_type="proposal_comment"`, and includes `external_collaborator=true` and `external_collaborator_id=<uuid>` in the audit `after` payload. `read_only` external collaborators receive **403**, not 401, on this endpoint.

8. **AC 8 — All other write/export endpoints rejected:** `PATCH`/`PUT`/`DELETE` on proposals, content blocks, sections, exports, downloads (`GET /api/v1/proposals/{id}/export`, `GET /api/v1/proposals/{id}/sections/{key}/export`), tasks, approvals, calendar, ESPD, etc. MUST return **403** when the caller presents a valid external collaborator JWT — never 200, never 500. Add a single guard inside `get_external_collaborator`-protected routes is not sufficient: the integration test matrix (AC 14) walks every router currently using `Depends(get_current_user)` and asserts 401 or 403 for an external token. The default behaviour is 401 (token shape rejected by `get_current_user`); only explicitly opt-in endpoints (the four enumerated in AC 6 + AC 7) accept external tokens.

9. **AC 9 — Stripe seat-count regression test:** Add an integration test that (a) creates a Pro+ subscription with seat_count=N, (b) invites M external collaborators, (c) asserts `count_active_seats(company_id)` (the helper used by `billing_service.report_usage` / Stripe `set_quantity`) still returns `N`, NOT `N+M`. The seat-count function MUST be defined as `SELECT COUNT(*) FROM client.company_memberships WHERE company_id = :cid AND accepted_at IS NOT NULL` (or equivalent existing helper); external_collaborators is a different table and must not be joined. If a seat-count helper does not yet exist in `billing_service`, add it, and use it from the test. **This AC is the literal contract guaranteeing FR8.5: external collaborators do not consume a paid Stripe seat.**

10. **AC 10 — Frontend landing page:** New route `app/[locale]/(auth)/external/accept/page.tsx` (or equivalent — a sibling to `callback`) reads `?token=...` from the URL, POSTs to `/api/v1/external/accept`, on success stores the raw JWT in `sessionStorage` under key `eusolicit-external-token`, sets `activeWorkspaceId` and `externalCollaboratorPrincipal` in the existing Zustand v2 store, and redirects to `/[locale]/external/proposal/{proposal_id}` (a **new, scoped** view at `app/[locale]/(protected)/external/proposal/[id]/page.tsx` — NOT under `/workspace/[workspaceId]/`). On 401/410 from the backend, render a localised error explaining "Link expired or already used" with a CTA to "Contact the inviter".

11. **AC 11 — Frontend scoped view:** The external proposal view renders the proposal title, sections, content (read-only TipTap render), and the comments thread. **No workspace switcher renders.** **No left nav renders** — only a minimal AppShell with logout (clears `sessionStorage`). For `comment_only` role, comment-creation UI is enabled (TipTap editor inline on each section); for `read_only`, the comment-creation UI is hidden. The Axios interceptor MUST include `Authorization: Bearer ${sessionStorage.getItem('eusolicit-external-token')}` for `/api/v1/external/*` and the four allow-listed proposal endpoints — NOT the regular auth-store JWT (avoid claim collision). Include source-inspection ATDD assertion (project-context Epic 11): the page imports `<Select>` from `@eusolicit/ui` if any select control is required (none in this scope, but assert no native `<select>` in the comment composer).

12. **AC 12 — Revocation endpoint:** `DELETE /api/v1/workspaces/{workspace_id}/proposals/{proposal_id}/external-invites/{collaborator_id}` (auth: `admin` or `bid_manager` workspace member) hard-deletes the `external_collaborators` row. Subsequent attempts to use the magic-link return 401 (AC 5 — JTI not found). Audit `action_type="delete"`, `entity_type="external_collaborator"`, `before={"email", "role", "expires_at"}`, `after=NULL`. List endpoint `GET /api/v1/workspaces/{workspace_id}/proposals/{proposal_id}/external-invites` returns active (non-deleted) invites with `accepted_at` and computed `expires_in_seconds`.

13. **AC 13 — Cross-tenant negative test (MANDATORY per project-context):** A `bid_manager` in tenant A invoking `POST /api/v1/workspaces/{tenant_b_workspace_id}/proposals/{tenant_b_proposal_id}/invite-external` returns **404** (existence-leakage protection — same status code an admin from a different tenant receives for unrelated workspace IDs). Must also pass for `DELETE` and `GET` external-invite endpoints. **Use the canonical `register_and_verify_with_role` and `create_company_pair` fixtures from root `conftest.py` — do not rebuild bespoke session/transport plumbing (Epic 14.2 anti-pattern: blocking finding #3, project-context Epic 14 carry-forward).**

14. **AC 14 — Parametrised endpoint matrix (MANDATORY per project-context Epic 14.2 Dev-Notes pattern):** Add `@pytest.mark.parametrize` matrix covering at minimum:
    - `endpoints` × `external_role` × `expected_status`
    - 4 allow-listed read endpoints (proposal detail, comments list, versions list, sections content) × {comment_only, read_only} × {match_proposal_id=200, mismatch_proposal_id=404}
    - 1 allow-listed write endpoint (POST comments) × {comment_only=201, read_only=403}
    - 6 representative reject-path endpoints (PATCH proposal, POST proposal, DELETE proposal, GET workspaces, GET opportunities, GET export) × any role × 403
    No partial coverage of the matrix is acceptable (Epic 14.2 review finding #2).

15. **AC 15 — Activate Epic 14.2 deferred RBAC scaffolds:** The three `@pytest.mark.skip` tests in `eusolicit-app/services/client-api/tests/integration/test_workspace_rbac.py` (`test_workspace_cross_access_forbidden`, `test_tenant_admin_and_bid_manager_bypass_own_company`, `test_audit_log_granted_and_denied_access`) were left as RED scaffolds explicitly tagged "tracked by E14.4". Either un-skip them and make them GREEN under the new `WorkspaceScope` enforcement, OR explicitly remove them and replace with the AC 13/AC 14 coverage above. Document which path was chosen in Dev Agent Record. Do NOT leave them skipped.

16. **AC 16 — Email send via existing `EmailServiceBase`:** Extend `EmailServiceBase` with `async def send_external_invite_email(self, to_email: str, magic_link_url: str, inviter_name: str, proposal_title: str, role: str) -> None` (and matching `StubEmailService` impl that logs at INFO with `event="external_invite_sent"`). Send the email AFTER the `external_collaborators` row + audit log are flushed (mirrors the `register()` pattern in `auth_service.py` line 145). Email failures MUST NOT roll back the invite (`try/except Exception`, log at WARNING, return 201 anyway).

17. **AC 17 — i18n parity (project-context Epic 11 anti-pattern guard):** All new EN strings (frontend landing page, error states, empty-comment placeholder, "Link expired" message, "Logout" CTA) added to `apps/client/messages/en.json` MUST have BG counterparts in `apps/client/messages/bg.json`. Run `pnpm check:i18n` and paste the output line into Dev Agent Record. ESLint `no-literal-text` rule (where configured) must not regress.

## Tasks / Subtasks

- [ ] Task 1 — Backend: Invite + JWT issue (AC 1, 2, 3, 13)
  - [ ] Subtask 1.1: Add `client_api/schemas/external_collaborator.py` with `ExternalInviteRequest`, `ExternalInviteResponse`, `ExternalAcceptRequest`, `ExternalAcceptResponse`, `ExternalCollaboratorPrincipal` dataclass.
  - [ ] Subtask 1.2: Add `services/external_collaborator_service.py` — `create_invite(workspace_id, proposal_id, email, role, current_user, session, ip)`. Validate workspace membership via `client.workspace_memberships`; validate proposal belongs to workspace; lowercase email; generate JTI (UUID4); insert row; build JWT with `create_external_magic_link_token()`; write audit; return response with `magic_link_url`. Use `from datetime import UTC` (project-context Epic 13 standard).
  - [ ] Subtask 1.3: Add `create_external_magic_link_token(collaborator_id, email, role, workspace_id, proposal_id, jti, ttl_days=30) -> str` to `client_api/core/security.py` — RS256, claims per AC 2.
  - [ ] Subtask 1.4: Add router `client_api/api/v1/external_invites.py` mounted under `/api/v1/workspaces/{workspace_id}/proposals/{proposal_id}/invite-external` (POST), `/external-invites` (GET list), `/external-invites/{collaborator_id}` (DELETE). Wire into `main.py` `api_v1_router.include_router(external_invites.router)`.
  - [ ] Subtask 1.5: Catch `IntegrityError` on unique `magic_link_jti` and on `(proposal_id, email)` active duplicate (add a partial unique index in a NEW alembic migration `048_external_invites_active_unique.py`: `UNIQUE (proposal_id, email) WHERE accepted_at IS NULL`). Return 409 — never 500 (project-context Epic 14.1 review pattern).

- [ ] Task 2 — Backend: Accept + auth dependency (AC 2, 4, 5)
  - [ ] Subtask 2.1: Add `client_api/api/v1/external.py` with `POST /external/accept` route. Wire into `main.py`.
  - [ ] Subtask 2.2: Add `services/external_collaborator_service.accept(token, session, ip) -> ExternalAcceptResponse`. Decode JWT; assert `external_collaborator==true`; SELECT row by `jti` FOR UPDATE; check single-use and expiry; stamp `accepted_at`; write audit. All denial paths return 401 or 410 with reason classifiers, NOT 500.
  - [ ] Subtask 2.3: Add `get_external_collaborator()` dependency to `client_api/core/security.py`. Returns `ExternalCollaboratorPrincipal` or raises 401. Mirror `get_current_user` shape but with separate claim validation (do not share code paths — claim collision risk).
  - [ ] Subtask 2.4: Add helper `get_principal_or_external(...)` returning `Union[CurrentUser, ExternalCollaboratorPrincipal]` for the four AC 6 read endpoints.

- [ ] Task 3 — Backend: Scoped read + comment write (AC 6, 7, 8)
  - [ ] Subtask 3.1: Refactor `GET /api/v1/proposals/{proposal_id}` (in `api/v1/proposals.py`) and the three other AC 6 read endpoints to use the union dependency. For external principal, enforce `proposal_id == principal.proposal_id` ELSE return 404 (existence-leakage; same as cross-tenant 14.2 pattern).
  - [ ] Subtask 3.2: Add new `require_proposal_comment_access` dependency in `client_api/core/rbac.py` covering both authenticated user roles and `comment_only` external principals. Refactor `POST /api/v1/proposals/{proposal_id}/comments` to use it. Audit row sets `user_id=NULL`, `after.external_collaborator=true`, `after.external_collaborator_id=<uuid>`.
  - [ ] Subtask 3.3: Verify (via integration test, AC 14 matrix) that NO other endpoint accepts the external token. Default behaviour: `get_current_user` rejects (claim shape mismatch → 401). The matrix is the contract.

- [ ] Task 4 — Backend: Revocation + list (AC 12)
  - [ ] Subtask 4.1: `DELETE /api/v1/workspaces/{ws}/proposals/{p}/external-invites/{collab_id}` — auth `bid_manager+ workspace member`, hard-delete row, audit `delete`.
  - [ ] Subtask 4.2: `GET /api/v1/workspaces/{ws}/proposals/{p}/external-invites` — list active rows, compute `expires_in_seconds = max(0, expires_at - now)`.

- [ ] Task 5 — Backend: Email service extension (AC 16)
  - [ ] Subtask 5.1: Add `send_external_invite_email` abstract method to `EmailServiceBase` and impl on `StubEmailService` (`structlog.info(event="external_invite_sent", ...)`).
  - [ ] Subtask 5.2: Wire into invite service AFTER flush + audit (mirror `register()` pattern). `try/except Exception: log.warning(...)` — never roll back.

- [ ] Task 6 — Backend: Stripe seat regression (AC 9)
  - [ ] Subtask 6.1: Locate or add `count_active_seats(company_id, session) -> int` in `services/billing_service.py`. Definition: `COUNT(*) FROM client.company_memberships WHERE company_id=:cid AND accepted_at IS NOT NULL`. Must NOT join `external_collaborators`.
  - [ ] Subtask 6.2: Integration test `test_external_invite_does_not_consume_seat` — create company with 3 active members, invite 5 external collaborators (mix of comment_only / read_only / accepted / unaccepted), assert `count_active_seats == 3`. Add a second test asserting Stripe `report_usage` payload (mocked Stripe SDK) carries `quantity=3` not 8.

- [ ] Task 7 — Backend: Test matrix + RBAC scaffold cleanup (AC 13, 14, 15)
  - [ ] Subtask 7.1: New file `tests/integration/test_external_collaborator_flow.py`. Use root `conftest.py` fixtures (`client_api`, `db_session`, `register_and_verify_with_role`, `create_company_pair`, `UserFactory`, `CompanyFactory`). DO NOT rebuild bespoke ASGI transport (Epic 14.2 blocking #3).
  - [ ] Subtask 7.2: `@pytest.mark.parametrize` matrix per AC 14 — at least 14 distinct cases. Run with `pytest tests/integration/test_external_collaborator_flow.py -v` and paste full pass count into Dev Agent Record (project-context Epic 13 anti-pattern: "Review approval without quoted test execution output" — completion claim invalid without quoted pytest output line).
  - [ ] Subtask 7.3: Activate / refactor / remove the three `@pytest.mark.skip` tests in `test_workspace_rbac.py`. Document choice in Dev Agent Record.
  - [ ] Subtask 7.4: Cross-tenant negative for invite POST/GET/DELETE returning 404 (AC 13).

- [ ] Task 8 — Frontend: Magic-link landing + scoped view (AC 10, 11, 17)
  - [ ] Subtask 8.1: New page `apps/client/app/[locale]/(auth)/external/accept/page.tsx`. Reads `?token=` from `useSearchParams()`, POSTs `/api/v1/external/accept`, stores in `sessionStorage.eusolicit-external-token`, redirects to scoped view. Use `useZodForm` for any input (project-context anti-pattern guard).
  - [ ] Subtask 8.2: New route group `apps/client/app/[locale]/(protected)/external/proposal/[id]/page.tsx` — sibling to `/workspace/`. Layout WITHOUT workspace switcher / sidebar; minimal AppShell + logout.
  - [ ] Subtask 8.3: New `lib/api/external-collaborator.ts` API client with `acceptExternalToken`, `fetchScopedProposal`, `fetchScopedProposalComments`, `createScopedComment`. Axios interceptor branch keyed on URL path: requests under `/api/v1/external/*` and `/api/v1/proposals/{id}` (when `sessionStorage.eusolicit-external-token` is set AND user not authenticated) attach the external token; otherwise fall through to existing auth-store JWT. **Do NOT reuse `eusolicit-client-auth-store-v2` for external tokens — claim shape collision risk.**
  - [ ] Subtask 8.4: TipTap render for sections (read-only). Comment composer enabled iff `principal.role === "comment_only"`.
  - [ ] Subtask 8.5: Source-inspection ATDD assertion (project-context Epic 11 pattern): unit test `__tests__/external-proposal-page.test.ts` asserts the page does NOT import `WorkspaceSwitcher`, does NOT call `useWorkspaceSync`, and does NOT use native `<select>`.
  - [ ] Subtask 8.6: i18n keys in `messages/en.json` AND `messages/bg.json` for: `external.accept.loading`, `external.accept.linkExpired`, `external.accept.linkConsumed`, `external.accept.linkInvalid`, `external.accept.contactInviter`, `external.proposal.viewOnly`, `external.proposal.commentPlaceholder`, `external.proposal.logout`, `external.proposal.expiresIn`. Run `pnpm check:i18n` and paste output to Dev Agent Record.

- [ ] Task 9 — Frontend: Inviter UI on proposal page (AC 1, 12)
  - [ ] Subtask 9.1: Add "Invite external reviewer" button to proposal toolbar (visible only to `admin` / `bid_manager` workspace members). Opens dialog with email input (Zod validation: valid email; not already invited active for this proposal) and role select (`<Select>` from `@eusolicit/ui` — NOT native `<select>`; project-context Epic 11). On success, displays the magic-link URL with copy-to-clipboard button.
  - [ ] Subtask 9.2: List active invites on the proposal sidebar with revoke button.

- [ ] Task 10 — Validation gate (project-context Epic 13 pattern: Review approval requires quoted test output)
  - [ ] Subtask 10.1: `make test-service SVC=client-api` — paste pytest summary line into Dev Agent Record.
  - [ ] Subtask 10.2: `python -c "from client_api.main import app"` — paste `OK` into Dev Agent Record (Epic 14.3 import-smoke pattern).
  - [ ] Subtask 10.3: Frontend `pnpm test --filter=@eusolicit/client` — paste summary into Dev Agent Record.
  - [ ] Subtask 10.4: `pnpm check:i18n` — paste line into Dev Agent Record.
  - [ ] Subtask 10.5: `alembic check` — confirm no pending autogenerate diff after migration 048 (Epic 14.0 pattern).

## Dev Notes

### Architecture Patterns & Constraints

- **Schema isolation (CLAUDE.md):** All new endpoints live in `client-api`. No cross-schema joins; `audit_log` lives in `shared` schema and is written via `write_audit_entry(session, ...)` (which uses the same client-api session — no cross-schema FK).
- **JWT contract (Story 2 Epic 2):** RS256, `client_api.core.security.get_rsa_private_key()` / `get_rsa_public_key()`. `jwt.encode(payload, key, algorithm="RS256")` / `jwt.decode(token, key, algorithms=["RS256"])`. **DO NOT** introduce a new signing algorithm or key — reuse the existing keys.
- **Claim shape isolation:** External collaborator JWTs MUST be rejected by `get_current_user`. Current `get_current_user` requires `sub`, `company_id`, `role`. External tokens deliberately omit `company_id` → claim parse raises `KeyError` → 401. This is the load-bearing isolation primitive.
- **Audit canonical form (project-context Epic 13 Rule 45):** In-transaction `write_audit_entry()`. For denials inside `get_external_collaborator`, use the `_write_denial_audit` pattern from `rbac.py` (separate session). Do NOT add `BackgroundTasks` for audit on denial paths.
- **External principal vs CurrentUser:** Define `ExternalCollaboratorPrincipal` as a SEPARATE dataclass (not a CurrentUser subclass) to make the type system enforce non-substitutability. Endpoints that accept both use `Union[CurrentUser, ExternalCollaboratorPrincipal]` explicitly.
- **No Stripe seat consumption (FR8.5 contract):** `external_collaborators` is a SEPARATE table from `company_memberships`. Stripe `set_quantity` calls in `billing_service.py` MUST count only `company_memberships` rows. Adding any join to `external_collaborators` is a contract violation. AC 9 is the regression guard.
- **Single-use semantics:** `accepted_at` stamped on FIRST accept; subsequent uses 410. This is project-context Epic 8 idempotency-via-DB-constraint pattern (see `webhook_events.event_id` unique constraint).
- **Existence-leakage protection (project-context Epic 14.2 Round 2):** Cross-tenant accesses return 404, not 403, to avoid leaking which workspace/proposal IDs exist in other tenants. Same protection applied to mismatched proposal_id under an external token.
- **PATCH semantics for nullable fields (project-context Epic 14.1 finding #3 / Epic 11 anti-pattern):** Not directly applicable here, but the invite list response uses `expires_in_seconds` (computed from `expires_at`), not raw `expires_at`, to avoid timezone bugs.
- **`Literal[...]` for enumerable status fields (project-context Epic 13 anti-pattern):** `ExternalInviteResponse.role: Literal["comment_only", "read_only"]`, `ExternalAcceptResponse.role: Literal[...]`. NEVER bare `str` for known enums.

### Hot-Fix / Carry-Forward Context (Pre-Applied — Verify, Don't Re-Implement)

- **Migration 046 already created the `external_collaborators` table.** Schema: `id`, `workspace_id`, `proposal_id`, `email`, `magic_link_jti` (unique), `expires_at`, `accepted_at` (nullable), `role` (CHECK `IN ('read_only', 'comment_only')`), `created_at`. **Do not redefine the table.** A new migration 048 ONLY adds the partial-unique index `UNIQUE (proposal_id, email) WHERE accepted_at IS NULL` to support AC 1 idempotent invite (409 on active duplicate).
- **`ExternalCollaborator` ORM model exists** at `client_api/models/external_collaborator.py` and is exported from `client_api/models/__init__.py`. **Do not redefine.** Add new relationship from `Proposal.external_collaborators` if needed (back_populates) — verify it doesn't already exist before adding.
- **Migration 046 already added a `workspace_id` column to `proposals` (NOT NULL)** — every proposal has a workspace. AC 1 validation `proposal.workspace_id == path workspace_id` is a single-column check, no JOIN.
- **Epic 14.2's RBAC code was reverted** (Epic 14.3 review BLOCKING-1 fix). The current `core/rbac.py` does NOT have `WorkspaceScope`, does NOT have workspace-aware bypass branches. This story does NOT need to extend `check_entity_access` for the workspace dimension — the new `require_proposal_comment_access` is a fresh, narrow dependency. If a broader `WorkspaceScope` is needed by future stories, that is out of scope here.
- **Three skipped tests in `test_workspace_rbac.py`** were tagged "tracked by E14.4" (Epic 14.3 round 2 follow-up). AC 15 requires un-skip OR remove with documented rationale.

### Previous Story Learnings — Critical Anti-Patterns to Avoid

From Epic 14.0–14.3 retrospectives and project-context Epics 11–13:

1. **Bespoke test fixtures (Epic 14.1, 14.2 BLOCKING #3):** `test_workspace_rbac.py` rebuilt `_register_and_verify_with_role`, used `AsyncClient(transport=ASGITransport(app=fastapi_app))` directly, used `db_session` (no rollback) instead of the canonical `client_session`, mutated state via raw SQL `text("UPDATE ... CAST(:role AS client.company_role)")`. **DO NOT REPEAT.** Use root `conftest.py` `client_api`, `db_session` (rollback semantics), `UserFactory`, `CompanyFactory`, `register_and_verify_with_role`, `create_company_pair` — these exist in `eusolicit-test-utils`.
2. **Parametrised matrix mandatory (Epic 14.2 BLOCKING #2):** "Cover all N roles × M permissions × own/cross-company combinations with `@pytest.mark.parametrize`. No partial coverage acceptable." AC 14 codifies the matrix dimensions for this story.
3. **Breaking schema change without backfill (Epic 14.2 BLOCKING #1):** S14.02 made `ProposalCreateRequest.workspace_id` required without migrating callers; broke `test_proposal_collaborators_audit.py`. **For this story:** if you change `Proposal` or comment endpoint signatures in ANY way, run the full client-api pytest suite and update all callers in the same PR. **DO NOT** ship a "5 passed" result for a feature that touches shared schemas without proving the rest of the suite still passes.
4. **Quoted test output required (project-context Epic 13 anti-pattern):** Completion notes claiming "all tests pass" without a quoted pytest summary line are invalid. Every Test Results section MUST contain the verbatim `N passed, M warnings in Xs` line from the actual run. The 2nd-pass reviewer trusting an agent's "all tests passing" summary is a critical anti-pattern (Story 7-17 round 3 review).
5. **Dual code paths (Epic 14.3 MEDIUM-2):** The fix collapsed Zustand migration into a single source of truth. **For this story:** authentication MUST have a single source of truth — either (a) Axios interceptor branches on URL pattern (preferred), or (b) two distinct Axios instances. NOT both. Document which.
6. **Don't re-implement merged inline fixes (project-context Epic 13 anti-pattern):** Migration 046 already created `external_collaborators`; the model already exists. New migration 048 ONLY adds the partial-unique index. Do not duplicate table-creation DDL.
7. **i18n parity (project-context Epic 11 anti-pattern):** Add EN+BG together in the SAME commit; run `pnpm check:i18n` BEFORE marking review-ready. Don't add EN keys "for now" and defer BG.
8. **Design-system component compliance (project-context Epic 11):** No native `<select>` in any new `WorkspaceSwitcher`-adjacent UI. Use `<Select>` from `@eusolicit/ui`. Add source-inspection ATDD test asserting this for the inviter dialog (Subtask 9.1).
9. **AC numeric constants in implementation (project-context Epic 7 anti-pattern):** AC 2 specifies 30-day TTL; AC 1 specifies 30-day expiry. Implementation MUST use a single named constant `EXTERNAL_INVITE_TTL = timedelta(days=30)` in `client_api/core/security.py` — not magic numbers in two places. Same for the TTL applied in audit `expires_at`.
10. **Pre-existing test failure (`test_publish_trial_expiring_sends_correct_payload`)** still fails at HEAD per Epic 14.3 deviation. NOT introduced by this story; do not "fix" it here.

### Technical Requirements

- **Language/framework:** Python 3.12+, FastAPI, async SQLAlchemy, PyJWT (existing deps).
- **`from __future__ import annotations`** at the top of every new module (project-wide pattern).
- **`from datetime import UTC`** — NEVER `timezone.utc` (project-context Epic 14.2 pattern).
- **structlog** for all logging (`log = structlog.get_logger()`).
- **DB constraint:** Migration 048 adds `CREATE UNIQUE INDEX ix_external_collaborators_active_email ON client.external_collaborators (proposal_id, email) WHERE accepted_at IS NULL`. Symmetric downgrade.
- **HMAC / signature comparison:** N/A here (JWT verification is library-managed). If any HMAC is added, use `hmac.compare_digest` (CLAUDE.md rule).
- **External HTTP timeout:** N/A (no outbound HTTP added). Stripe SDK call from billing_service is existing; not modified.
- **Email send is non-blocking and fail-open:** AC 16. Mirror `auth_service.register()` line 145 — email send happens AFTER all DB work, in the same async function but inside its own `try/except Exception`.
- **Negative tests for every new endpoint** (CLAUDE.md rule). Six new endpoints → minimum six 401/403/404 negative tests.
- **No `from module import *`**, no bare `except:` (CLAUDE.md rules).

### File Structure Requirements

```
eusolicit-app/services/client-api/
├── alembic/versions/
│   └── 048_external_invites_active_unique.py             # NEW
├── src/client_api/
│   ├── core/
│   │   └── security.py                                   # MODIFY — add create_external_magic_link_token, get_external_collaborator, ExternalCollaboratorPrincipal
│   ├── core/
│   │   └── rbac.py                                       # MODIFY — add require_proposal_comment_access dep
│   ├── api/v1/
│   │   ├── external_invites.py                           # NEW — POST/GET/DELETE under workspace+proposal scope
│   │   ├── external.py                                   # NEW — POST /external/accept
│   │   └── proposals.py                                  # MODIFY — accept Union[CurrentUser, ExternalCollaboratorPrincipal] on the four AC 6 read endpoints
│   ├── api/v1/
│   │   └── proposal_comments.py                          # MODIFY — accept external comment_only role on POST
│   ├── schemas/
│   │   └── external_collaborator.py                      # NEW
│   ├── services/
│   │   ├── external_collaborator_service.py              # NEW — invite, accept, list, revoke
│   │   ├── billing_service.py                            # MODIFY (or verify) — count_active_seats helper does NOT include external_collaborators
│   │   └── email_service.py                              # MODIFY — add send_external_invite_email
│   ├── main.py                                           # MODIFY — include_router for external + external_invites
│   └── models/external_collaborator.py                   # READ-ONLY (already exists from migration 046; only add Proposal.external_collaborators relationship if needed)
└── tests/integration/
    ├── test_external_collaborator_flow.py                # NEW — primary AC 14 matrix + AC 9 seat regression + AC 16 email + AC 13 cross-tenant
    └── test_workspace_rbac.py                            # MODIFY — un-skip / refactor / remove the three E14.4 scaffolds (AC 15)

eusolicit-app/frontend/apps/client/
├── app/[locale]/(auth)/external/accept/
│   └── page.tsx                                          # NEW — landing
├── app/[locale]/(protected)/external/proposal/[id]/
│   ├── layout.tsx                                        # NEW — minimal AppShell, no switcher
│   └── page.tsx                                          # NEW — scoped read view + comment composer
├── lib/api/
│   └── external-collaborator.ts                          # NEW
├── components/proposal/
│   └── InviteExternalDialog.tsx                          # NEW — invite + revoke UI for bid_manager+
├── messages/
│   ├── en.json                                           # MODIFY — external.* keys per Subtask 8.6
│   └── bg.json                                           # MODIFY — BG parity
└── __tests__/
    └── external-proposal-page.test.ts                    # NEW — source-inspection ATDD per Subtask 8.5
```

### Testing Requirements

- **ATDD-first (project-context Epic 5 pattern):** Write the AC 14 parametrised matrix as RED tests FIRST. Then implement. AC 13 cross-tenant negative is non-negotiable (project-context backend-conventions).
- **Test isolation:** `db_session` (rollback) — never `commit()` in tests. `clean_redis` if any Redis interaction. Override `get_db_session` and clear `dependency_overrides` in finally (root `conftest.py` pattern).
- **Pytest markers:** `@pytest.mark.integration` on flow tests; `@pytest.mark.unit` on JWT helper unit tests if any.
- **Test data:** `register_and_verify_with_role`, `create_company_pair`, `UserFactory`, `CompanyFactory` — DO NOT rebuild bespoke `_register_and_verify_with_role` (Epic 14.2 anti-pattern repetition will block merge).
- **Frontend:** Vitest source-inspection ATDD tests for component compliance (project-context Epic 11 standard); Playwright E2E test for accept-link → comment-create flow under `e2e/specs/external-collaborator/`.
- **Coverage:** ≥ 80% (`make coverage` minimum).

### References

- [Source: eusolicit-docs/planning-artifacts/epics/E14-multi-client-workspace.md#S14.04]
- [Source: eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md#FR8.5] (no Stripe seat consumption contract)
- [Source: eusolicit-docs/implementation-artifacts/14-0-schema-migration-atomic-default-workspace-backfill.md] (migration 046 created `external_collaborators` table — DO NOT redefine)
- [Source: eusolicit-docs/implementation-artifacts/14-1-workspace-crud-api-audit-trail-extension.md#Senior-Developer-Review] (workspace CRUD pattern + 409 / IntegrityError translation)
- [Source: eusolicit-docs/implementation-artifacts/14-2-rbac-extension-workspacescope-depends-tenant-admin-cross-workspace-bypass.md#Senior-Developer-Review] (parametrised matrix requirement, canonical fixture rule)
- [Source: eusolicit-docs/implementation-artifacts/14-3-frontend-workspace-switcher-url-routing-zustand-migration.md#Round-2-3-4] (Zustand persist v2, sessionStorage isolation, design-system compliance)
- [Source: eusolicit-app/services/client-api/src/client_api/core/security.py#L96-L122] (RS256 token creation pattern — reuse keys)
- [Source: eusolicit-app/services/client-api/src/client_api/services/auth_service.py#L37-L150] (register() — canonical email send AFTER flush + audit pattern)
- [Source: eusolicit-app/services/client-api/src/client_api/services/audit_service.py#L30-L96] (write_audit_entry — non-blocking in-transaction pattern)
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/proposal_comments.py#L31-L130] (comments POST — extend to accept external comment_only)
- [Source: eusolicit-app/services/client-api/src/client_api/models/external_collaborator.py] (model — read-only reference)
- [Source: eusolicit-app/services/client-api/alembic/versions/046_schema_migration_atomic_default_workspace_backfill.py#L90-L124] (existing external_collaborators DDL)
- [Source: eusolicit-app/services/client-api/tests/integration/test_workspace_rbac.py#L85-L297] (three skipped scaffolds — AC 15)
- [Source: test_artifacts/atdd-checklist-14-3-*.md] (frontend ATDD pattern reference; no epic-level test design exists for E14)
- [Source: project-context.md#Patterns] (Epic 11–13 patterns and anti-patterns referenced inline above)
- [Source: CLAUDE.md] (cross-tenant negative-test rule, schema isolation, HMAC, audit, no bare except)

### Project Context Reference

**Note:** No epic-level `test-design-epic-14.md` exists in `test_artifacts/`. Test expectations were derived from:
1. The four story-level ATDD checklists `atdd-checklist-14-{0,1,2,3}-*.md` (only 14-3 includes a structured test strategy table — frontend-only).
2. The Epic 14.2 review's mandated parametrised matrix (BLOCKING #2).
3. Epic 14.3's E2E test pattern at `e2e/specs/shell/workspace-switcher.spec.ts` (UUID mock IDs, middleware regex, `toHaveURL` assertions) — extend with external-collaborator E2E spec under `e2e/specs/external-collaborator/`.
4. Project-context Epics 11–13 anti-patterns (i18n parity, design-system compliance, parametrised matrix, quoted pytest output, single source of truth).

This story explicitly fills the test-design gap by codifying AC 14 as the canonical contract matrix.

## Dev Agent Record

### Agent Model Used

{{agent_model_name_version}}

### Debug Log References

### Completion Notes List

### File List

### Test Results

<!-- Required (project-context Epic 13 anti-pattern guard): paste verbatim quoted pytest / vitest / playwright / i18n summary lines here. Without these, review approval is invalid. -->

- client-api pytest: `<paste `N passed, M warnings in Xs` here>`
- client-api import smoke (`python -c "from client_api.main import app"`): `<paste OK here>`
- frontend client vitest: `<paste `Test Files X passed Tests Y passed` here>`
- frontend `@eusolicit/ui` vitest: `<paste here>`
- i18n parity (`pnpm check:i18n`): `<paste `✅ i18n keys match: N keys ...` here>`
- alembic check: `<paste `No new upgrade operations detected.` here>`
- Playwright (if executed): `<paste summary or document infra deviation per Epic 14.3 pattern>`
