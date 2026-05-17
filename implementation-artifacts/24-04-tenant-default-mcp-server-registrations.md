# Story 24.04: Tenant-Default MCP Server Registrations (Inactive Stubs)

**Status:** backlog
**Epic:** E24 — SirmaAI Tenant Provisioning
**Points:** 3
**Type:** backend
**Dependencies:** S24.03 (template seed), S17.30a + S17.30b (Dynamics 365 MCP spec ready), S17.31 (HubSpot MCP spec ready)
**Blocks:** S17.32 (OAuth flow activates the stubs); S24.07 (erasure flow needs MCP secret refs)
**Created:** 2026-05-15
**Source:** `eusolicit-docs/planning-artifacts/epics/E24-sirmaai-tenant-provisioning.md` §S24.04

## Story

As a **platform engineer**,
I want **every newly-provisioned tenant SirmaAI Project to have inactive MCP-server stubs for Dynamics 365 + HubSpot pre-registered**,
so that **when a Pro+ admin clicks "Connect CRM" later, the OAuth flow can activate an existing stub rather than register a fresh MCP server mid-OAuth (which would race against the agent that wants to use it)**.

## Acceptance Criteria

1. After S24.03 seed succeeds, dispatch a follow-up Celery task `register_default_mcp_servers(company_id)` that registers two inactive MCP-server stubs per Project.
2. For each provider (`dynamics365`, `hubspot`):
   - Call `POST /api/organizations/{orgId}/projects/{projectId}/mcp-servers` with the tool-spec body sourced from `services/integrations-api/config/sirmaai_mcp_specs/<provider>.yaml` (authored under S17.30a / S17.31). Body includes the 5 tool definitions: `find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`.
   - Set `status='inactive'` (no secrets pushed — OAuth flow per E17/E27 S17.32 activates).
   - Persist `client.sirmaai_mcp_servers` row with the SirmaAI-side `sirmaai_mcp_server_id` and `status='inactive'`.
3. Both stubs are queryable via an admin-API endpoint (added in S24.05): `GET /api/v1/admin/companies/{id}/mcp-servers` returns a list with provider + status.
4. **Idempotency**: re-running the task on a tenant with existing stubs is a no-op (UNIQUE `(company_id, provider)` enforces).
5. **Failure-mode**: if S17.30a or S17.31 has not landed (the spec YAML doesn't exist), the task logs a warning and skips that provider gracefully — does NOT fail provisioning. This allows S24.04 to ship before all CRM stories complete.
6. **Cross-tenant**: tenant A's MCP-server stubs are scoped to tenant A's Project; verified by inspecting that `sirmaai_mcp_server_id` cannot be cross-referenced across tenants (SirmaAI-side enforces).
7. No OAuth tokens, refresh tokens, or any provider secrets are written by this story. Status `registered` only happens later via S17.32.
8. Audit log: each MCP-server registration writes `shared.audit_log` row with `action_type='mcp_server_stub_registered'`.

## Dev Notes

### Pattern reuse
- Spec YAML format authored in S17.30a (Dynamics) + S17.31 (HubSpot). This story READS those specs; if they're absent at deploy time, fall back gracefully (warn + skip).
- SirmaAI admin client: extend with `create_mcp_server(org_id, project_id, spec)` — see `sirmaai-gateway/clients/admin_client.py`.

### Files likely touched
- `services/client-api/src/client_api/tasks/sirmaai_provisioning.py` (extend)
- `services/sirmaai-gateway/src/sirmaai_gateway/clients/admin_client.py` (extend with `create_mcp_server`)
- `services/client-api/src/client_api/services/mcp_spec_loader.py` (new — YAML reader for spec files)
- `services/client-api/tests/unit/tasks/test_register_default_mcp_servers.py`
- `services/client-api/tests/integration/test_mcp_stub_registration.py`

### Why "inactive" matters
- An "inactive" stub still has the tool spec registered server-side. When OAuth activates it (S17.32 pushes secrets), SirmaAI flips status server-side and the stub becomes callable without re-registration. This avoids a race condition where qualification agent could try to call `find_account` against a not-yet-registered MCP server.

### Out of scope
- OAuth callback handling + token push (S17.32)
- Token rotation (S17.33)
- CRM dashboard UI (S17.35)
- Pipedrive + Salesforce providers (post-launch — same pattern, separate stories)

## Risks

- **R1**: Specs land later than S24.04 — the graceful-skip behavior mitigates. If both providers are unregistered, OAuth-connect flow simply returns "Provider not yet available — try again after platform upgrade".
- **R2**: SirmaAI MCP-server quota per Project — unknown limit; flag with operator before high-volume tenant rollout.

## Testing

- Unit: spec-YAML loader, graceful skip on missing file, idempotency check.
- Integration: provision a Project + run seed → both stubs present via admin-API list.
- Cross-tenant: 2 companies provisioned; each has 2 stubs; no cross-references.

## See also

- Epic file §S24.04
- E17 amendment §S17.30a, S17.31 (spec authoring)
- E27 §S27 numbering note (canonical keys = S17.*)
- PRD amendment FR-53
