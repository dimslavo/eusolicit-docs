# Story 17.32: OAuth Callback Hosting + Fernet Token Vault + SirmaAI Secret Push

**Status:** backlog
**Epic:** E17 amendment / E27 — CRM via SirmaAI MCP
**Points:** 5
**Type:** backend
**Dependencies:** S17.30b (Dynamics stub registered), S17.31 (HubSpot stub registered)
**Blocks:** S17.33, S17.34, S17.35, S17.36
**Created:** 2026-05-15
**Source:** E17 amendment §Inject

## Story

As **Elena (workspace admin on Pro+ tier)**,
I want **to connect Dynamics 365 + HubSpot to my workspace via OAuth, with tokens stored Fernet-encrypted in EU Solicit and pushed to SirmaAI's MCP-server config**,
so that **my qualifier + quantifier agents can enrich opportunities with my CRM data without me ever seeing or managing API keys directly**.

## Acceptance Criteria

1. **Endpoint** `GET /api/v1/workspaces/{id}/crm/{provider}/connect` returns provider auth URL with state nonce. Auth scopes: minimum-privilege per provider (read accounts/deals, write deals).
2. **Endpoint** `GET /api/v1/crm/oauth/callback` (single shared callback URL registered with both providers) validates state nonce, exchanges code for tokens, persists Fernet-encrypted in `client.crm_connections`. Schema: `id`, `company_id`, `provider`, `access_token_encrypted`, `refresh_token_encrypted`, `expires_at`, `scopes`, `created_at`.
3. **On successful OAuth**: push tokens to SirmaAI MCP-server secrets via SirmaAI admin API `PUT /api/organizations/{orgId}/projects/{projectId}/mcp-servers/{id}/secrets`. Update `client.sirmaai_mcp_servers.status='registered'`.
4. **Tier gating**: Pro+ only. Free/Starter get 402 with upgrade CTA.
5. **Failure path**: SirmaAI secret push fails → keep OAuth tokens in EU Solicit; do NOT advance `client.sirmaai_mcp_servers.status`; alert admin; retry via Beat.
6. **State nonce expiry**: 10-min TTL; expired returns 400 with "OAuth state expired — please retry".
7. **Audit log**: each connect/callback writes audit rows.
8. **Cross-tenant**: tenant A's tokens cannot be used by tenant B. Provider field + company_id scope enforced.
9. **Integration test** with mocked provider OAuth + mocked SirmaAI secret push.

## Dev Notes

### Pattern reuse
- Fernet canonical (Epic 9).
- OAuth flow: similar to Google Calendar OAuth in E09 S09.08 (mirror it).
- State nonce in Redis with TTL.

### Files likely touched
- `services/client-api/alembic/versions/<next>_crm_connections.py` (new — may exist already from original E17; verify)
- `services/client-api/src/client_api/routers/crm.py` (new)
- `services/client-api/src/client_api/services/crm_oauth_service.py` (new)
- `services/sirmaai-gateway/src/sirmaai_gateway/clients/admin_client.py` (extend with `push_mcp_secrets`)
- `services/client-api/tests/integration/test_crm_oauth_flow.py`

### Out of scope
- Token rotation (S17.33).
- Conflict log (S17.34).
- CRM dashboard UI (S17.35).
- Workspace archive cleanup (S17.36, S24.06).

## Risks

- **R1**: Original E17 had `client.crm_connections` table — verify whether the schema is reusable or needs migration adjustment.

## Testing

- Integration: full OAuth round-trip with mock provider.
- Cross-tenant: scope-check.

## See also

- E17 amendment §S17.32
- E09 S09.08 (Google Calendar OAuth — same pattern)
- PRD amendment FR-53
