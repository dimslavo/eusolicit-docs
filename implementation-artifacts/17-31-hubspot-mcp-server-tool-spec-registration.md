# Story 17.31: HubSpot MCP Server — Tool Spec + Registration

**Status:** backlog
**Epic:** E17 amendment / E27 — CRM via SirmaAI MCP
**Points:** 5
**Type:** backend + integration
**Dependencies:** S24.04 (registration mechanism)
**Blocks:** S17.32 (OAuth activates this stub); tenant onboarding for HubSpot
**Created:** 2026-05-15
**Source:** E17 amendment §Inject + E27 §Stories

## Story

As an **agent integrator**,
I want **a HubSpot MCP server with the same 5-tool surface as Dynamics, including spec authoring + registration + sandbox tests**,
so that **HubSpot tenants have feature parity with Dynamics 365 from day one**.

## Acceptance Criteria

1. Spec YAML at `services/integrations-api/config/sirmaai_mcp_specs/hubspot.yaml` — same shape as S17.30a, mapped to HubSpot CRM API endpoints.
2. 5 tools authored: `find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`. HubSpot-specific fields (companies vs accounts, deals stage IDs, properties).
3. Registration flow per S17.30b pattern: stub registered at provisioning time, activated via OAuth (S17.32).
4. **Sandbox integration tests** against HubSpot dev account covering all 5 tools end-to-end.
5. Negative paths (auth failure, rate-limit) parallel to S17.30b.
6. Observability via S17.34 invocation log.

## Dev Notes

### Pattern reuse
- Same shape as S17.30a + S17.30b. HubSpot is simpler than Dynamics (fewer custom fields).

### Files likely touched
- `services/integrations-api/config/sirmaai_mcp_specs/hubspot.yaml` (new)
- `services/integrations-api/src/integrations_api/services/hubspot_mcp.py` (new)
- `services/integrations-api/tests/integration/test_hubspot_sandbox.py`

### Out of scope
- OAuth flow (S17.32).
- Conflict resolution (S17.34).

## Risks

- **R1**: HubSpot API rate limits more aggressive than Dynamics for free-tier dev accounts — coordinate with HubSpot.

## Testing

- Sandbox integration.

## See also

- E17 amendment §S17.31
- S17.30a/b (Dynamics parallel pattern)
