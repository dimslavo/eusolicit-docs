# E14: Multi-Client Workspace Data Model & RBAC

**Sprint**: 14–16 | **Points**: 34 | **Dependencies**: E01, E02, E06 | **Milestone**: Consulting-firm ICP foundation
**Source:** PRD v1.1 §6 FR8 + §8 US7/US8; architecture-evaluation §2 Change-1, §3 CC-1/CC-4, §4 data model

## Goal

Introduce the multi-client workspace tenancy model — the foundational change that enables EU Solicit to credibly serve consulting firms managing 10–50 client engagements concurrently. A `company` row remains the tenant boundary (no rename — preserves Epics 1–13 RBAC and billing semantics); a new `client_workspaces` child entity scopes opportunities, proposals, content, audit-trail entries, AI-summary metering, and outcome telemetry. Existing single-tenant accounts auto-backfill with one default workspace named "Default" via an atomic migration in S14.00. RBAC extends to a third tier (`tenant_admin` for cross-workspace bypass) by extending the existing `check_entity_access()` factory rather than rewriting it. The frontend gains a top-level workspace switcher with `/[locale]/workspace/[workspaceId]/...` routing and a Zustand persist namespace migration. External end-clients (the consulting firm's customers) can be invited as comment-only collaborators via magic-link without consuming a paid Stripe seat.

## Acceptance Criteria

- [ ] `client.client_workspaces` table created with `(company_id, name)` unique constraint (where `archived_at IS NULL`); `client.workspace_memberships`, `client.external_collaborators` tables created
- [ ] All workspace-scoped tables (`opportunities`, `proposals`, `documents`, `content_blocks`) gain a nullable `workspace_id` column; existing rows backfilled to a single default workspace per company in one atomic migration; **regression test inserts ≥2 rows per company under same grouping key before testing migration downgrade safety** (project-context Epic 11 pattern)
- [ ] `shared.audit_log` gains nullable `workspace_id` column; existing entries remain valid; new entries record `workspace_id` for all workspace-scoped mutations
- [ ] No transition shim required (no existing paying customers per locked decision §11.1) — migration is one-shot atomic with rollback safety
- [ ] Tenant-admin user can create / rename / archive / delete workspaces via REST API; archive is soft-delete with retention; delete enforces 30-day grace
- [ ] `WorkspaceScope` FastAPI Depends() factory enforces workspace-scoped access; `tenant_admin` role bypasses for cross-workspace search and analytics
- [ ] Cross-workspace negative test passes: user assigned only to W1 cannot access W2 (403) **even within the same tenant** — extension of existing cross-tenant pattern
- [ ] External collaborators authenticate via magic-link (30-day expiry, single-use) with `comment_only` or `read_only` role; do not consume a paid Stripe seat; activity audit-logged with `external_collaborator` flag
- [ ] Frontend workspace switcher renders in topbar; URL routing under `/[locale]/workspace/[workspaceId]/...` for all protected pages
- [ ] Zustand persist namespace migrated from `eusolicit-client-auth-store` to `eusolicit-client-auth-store-v2` with read-old-write-new fallback (no data loss on first load)
- [ ] Workspace-scoped negative tests cover all new endpoints (mandatory pattern extension)
- [ ] Frontend ATDD includes source-inspection assertions for design-system component compliance (no native `<select>` where `<Select>` from `@eusolicit/ui` is required — project-context Epic 11 anti-pattern)

## Stories

### S14.00: Schema Migration + Atomic Default-Workspace Backfill
**Points**: 5 | **Type**: backend | **Sequence**: First — gates all of Epic 14

Create Alembic migration adding the three new tables (`client_workspaces`, `workspace_memberships`, `external_collaborators`) and the nullable `workspace_id` column on `opportunities`, `proposals`, `documents`, `content_blocks`. In the same migration, atomically create one default workspace per existing `company` row (named `"Default"`) and backfill all rows' `workspace_id` to point at it. Add nullable `workspace_id` to `shared.audit_log`. Migration must be reversible — downgrade drops the column and tables but **must include a regression test inserting ≥2 opportunity rows per company under the same workspace before exercising downgrade** to validate multi-row safety (project-context Epic 11 pattern: "Migration Downgrade Multi-Row Safety"). Cutover runs in a single transaction during a planned 15-minute maintenance window. No transition shim — backend supports only the new request shape from cutover.

---

### S14.01: Workspace CRUD API + Audit Trail Extension
**Points**: 5 | **Type**: backend

Implement REST endpoints for workspace management at the tenant scope:
- `POST /api/v1/workspaces` — create (admin role required at tenant level)
- `GET /api/v1/workspaces` — list active workspaces
- `GET /api/v1/workspaces/:id` — detail
- `PATCH /api/v1/workspaces/:id` — rename
- `POST /api/v1/workspaces/:id/archive` — soft-delete, 30-day grace
- `DELETE /api/v1/workspaces/:id` — hard delete after grace

Every mutation writes to `shared.audit_log` with the new `workspace_id` column populated and `entity_type='workspace'`. Audit writes use the canonical fire-and-forget pattern (project-context Epic 13 Rule 45). Workspace name uniqueness enforced at DB level per the `(company_id, name) WHERE archived_at IS NULL` constraint. Cross-tenant negative test mandatory.

---

### S14.02: RBAC Extension — WorkspaceScope Depends + tenant_admin Cross-Workspace Bypass
**Points**: 8 | **Type**: backend | **Security-critical**

Extend the existing `check_entity_access()` RBAC factory in `client_api/core/rbac.py` to take an additional `workspace_id` scope parameter. Introduce `WorkspaceScope` FastAPI Depends() injection that:
1. Reads `workspace_id` from URL path (`/workspace/{workspace_id}/...`)
2. Verifies user has membership row in `workspace_memberships` for that workspace_id, OR is `tenant_admin` for the parent company (cross-workspace bypass)
3. Loads workspace context into request state for downstream handlers

Add tenant-level `tenant_admin` role to the existing role enum. Refactor every existing workspace-scoped endpoint (opportunities, proposals, documents, content_blocks, analytics) to require `WorkspaceScope` injection. **Cross-workspace negative test mandatory: user with only W1 membership receives 403 on W2 endpoints within the same tenant.** Extension of existing cross-tenant pattern. Audit log entries scope `workspace_id` consistently.

---

### S14.03: Frontend Workspace Switcher + URL Routing + Zustand Migration
**Points**: 13 | **Type**: frontend | **2 ATDD-heavy sprints**

Refactor `apps/client/app/[locale]/(protected)/` to nest under `/workspace/[workspaceId]/`. Add a top-level workspace switcher component in the topbar (rendered server-side per AppShell pattern; interactive leaves `'use client'`). Switcher lists active workspaces, indicates current, allows quick-switch (URL navigation), and surfaces "Create Workspace" CTA for tenant-admins.

Migrate Zustand persist namespace from `eusolicit-client-auth-store` to `eusolicit-client-auth-store-v2` adding `activeWorkspaceId` to persisted state. Migration logic on first load: read v1 → write v2 → delete v1 (no data loss). All TanStack Query keys gain `workspace_id` as a key segment for proper cache isolation between workspaces.

ATDD source-inspection assertions required:
- Switcher uses `<Select>` from `@eusolicit/ui` (NOT native `<select>` — project-context Epic 11 anti-pattern)
- All workspace-scoped pages wrap data fetching in `<QueryGuard>` (existing pattern)
- Forms use `useZodForm(schema)` + `<FormField>` (existing pattern)
- All UI strings use `useTranslations()` from next-intl; BG/EN parity check passes

E2E Playwright tests cover: workspace switch retains auth, URL persists correctly, deep-link to `/workspace/W2/opportunities/X` while logged-in to W1 redirects appropriately.

---

### S14.04: External Collaborator Magic-Link Flow + Comment-Only Role
**Points**: 3 | **Type**: fullstack

Implement external collaborator invitation:
- `POST /api/v1/workspaces/:id/proposals/:pid/invite-external` — accepts email, role (`comment_only` or `read_only`), generates magic-link with 30-day expiry, single-use, `external_collaborator` JWT claim
- Magic-link landing route on frontend authenticates the external user, sets a scoped session that grants access only to the invited proposal in the invited workspace
- External user sees only the invited proposal — cannot navigate to other workspace data
- `comment_only` role can add comments to proposal sections but cannot edit content or download exports
- `read_only` role can view but not comment
- External collaborator activity audit-logged with `external_collaborator=true` flag and `workspace_id` set
- External collaborators do NOT consume a paid Stripe seat (verified by integration test against Pro+ seat-count metering)
- Magic-link revocable by inviting user; expires automatically after 30 days

JWT structure reuses existing RS256 pattern (Epic 2) with new `external_collaborator` claim and `scoped_resource` claim binding to a single proposal_id.
