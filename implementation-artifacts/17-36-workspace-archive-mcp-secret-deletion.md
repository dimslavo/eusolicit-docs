# Story 17.36: Workspace Archive → MCP Secret Deletion

**Status:** backlog
**Epic:** E17 amendment / E27 — CRM via SirmaAI MCP
**Points:** 2
**Type:** backend
**Dependencies:** S17.32 (token vault), S24.06 (workspace archive flow)
**Blocks:** S24.07 (erasure proof)
**Created:** 2026-05-15
**Source:** E17 amendment §Inject

## Story

As a **company admin archiving my workspace**,
I want **all CRM OAuth tokens revoked at the provider, all SirmaAI MCP-server secrets deleted, and audit-logged**,
so that **I have a clean audit trail showing CRM data flow stopped at archive time**.

## Acceptance Criteria

1. Subscriber on existing Redis Stream `company.archived` (or hook into S24.06 archive task): on each event, dispatch `revoke_crm_connections(company_id)`.
2. For each `client.crm_connections` row WHERE `company_id=X AND invalidated_at IS NULL`:
   - (a) Call provider's revoke endpoint (HubSpot, Dynamics) with the access + refresh tokens.
   - (b) Delete MCP-server secrets in SirmaAI via `DELETE /api/organizations/{orgId}/projects/{projectId}/mcp-servers/{id}/secrets`.
   - (c) Update `client.sirmaai_mcp_servers.status='inactive'`.
   - (d) Set `client.crm_connections.invalidated_at=now()`.
3. **Audit log**: each step writes `shared.audit_log` row.
4. **Idempotency**: re-running is safe (no-op on already-invalidated rows).
5. **Failure path**: provider revoke 5xx → retry with backoff; if persistent, set `invalidated_at` anyway and surface for admin review (provider-side cleanup may need manual operator action).
6. **Integration test**: synthetic archive → both providers revoked + SirmaAI secrets gone.

## Dev Notes

### Pattern reuse
- S24.06 archive flow (this story chains into it).
- Provider OAuth revoke: per provider docs.

### Files likely touched
- `services/client-api/src/client_api/tasks/crm_revoke.py` (new)
- `services/client-api/src/client_api/consumers/company_archived.py` (extend or new consumer)
- `services/client-api/tests/integration/test_workspace_archive_crm_revoke.py`

### Out of scope
- Manual operator cleanup of provider-side leftovers (post-launch).

## Risks

- **R1**: Provider revoke endpoint may return 404 if token already revoked — treat as success.

## Testing

- Integration: full archive → revoke → SirmaAI cleanup.

## See also

- E17 amendment §S17.36
- S24.06 (workspace archive)
- S24.07 (erasure proof)
