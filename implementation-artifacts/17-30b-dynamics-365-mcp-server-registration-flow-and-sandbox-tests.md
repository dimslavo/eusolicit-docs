# Story 17.30b: Dynamics 365 MCP Server — Registration Flow + Sandbox Tests

**Status:** backlog
**Epic:** E17 amendment / E27 — CRM via SirmaAI MCP
**Points:** 4
**Type:** backend + integration
**Dependencies:** S17.30a (spec); S24.04 (MCP stub registration mechanism)
**Blocks:** S17.32 (OAuth activates this stub); tenant onboarding for D365
**Created:** 2026-05-15 (SPLIT from former 8pt S17.30 per IR remediation Issue #7)
**Source:** E17 amendment §Inject + E27 §Stories

## Story

As a **platform engineer**,
I want **the Dynamics 365 MCP stub registration flow + integration tests against a real Dynamics 365 sandbox covering all 5 tools end-to-end**,
so that **the OAuth-connect flow (S17.32) can activate the stub with confidence and all 5 tools work in production**.

## Acceptance Criteria

1. Implement registration flow consuming the S17.30a spec: at tenant-provisioning time (E24 S24.04 invokes this), register an inactive Dynamics 365 MCP-server in the SirmaAI Project via `POST /api/organizations/{orgId}/projects/{projectId}/mcp-servers` with the spec body. `client.sirmaai_mcp_servers.status='inactive'` until OAuth completion.
2. Tool routing via MCP-tool-invocation infrastructure (server-side) so calls to `find_account` map to the right Dataverse endpoint with auth-token injection.
3. **Sandbox integration tests** against a Dynamics 365 dev/sandbox tenant covering each of the 5 tools end-to-end:
   - `find_account`: query by name → returns matching accounts.
   - `create_deal`: create an opportunity → returns the new ID + URL.
   - `update_deal_stage`: move opportunity stage → returns updated state.
   - `enrich_contact`: lookup contact by email → returns enriched profile.
   - `attach_note`: attach a structured note → returns note ID.
4. **Negative paths**:
   - Malformed spec → registration 4xx with structured error.
   - Sandbox 401 → MCP returns auth error; caller receives clear message.
   - Sandbox rate-limit → MCP propagates 429; agent retries via existing pattern.
5. **Test fixtures**: sandbox-tenant credentials in `.env.test` (NOT in repo); CI conditionally runs the sandbox tests only when `DYNAMICS365_SANDBOX_TOKEN` is set.
6. **Idempotency** of registration: same tenant + provider → no duplicate (per UNIQUE constraint in S24.01).
7. **Observability**: each tool invocation logged with `tool_name`, `tenant_id`, `outcome`, `duration_ms` (via S17.34 invocation log).

## Dev Notes

### Pattern reuse
- S24.04 stub-registration code path; this story makes the registration RICH (full spec, not stub).

### Files likely touched
- `services/integrations-api/src/integrations_api/services/dynamics365_mcp.py` (new — registration adapter consuming the spec)
- `services/integrations-api/tests/integration/test_dynamics365_sandbox.py` (new — gated by `DYNAMICS365_SANDBOX_TOKEN`)
- `.env.test.example` (extend with Dynamics sandbox slots)

### Out of scope
- OAuth flow (S17.32).
- Token rotation (S17.33).
- Conflict resolution (S17.34).
- HubSpot equivalent (S17.31).

## Risks

- **R1**: Dynamics sandbox API quirks — surface in PR review.
- **R2**: Sandbox tenant cost — coordinate with finance.

## Testing

- Sandbox integration: all 5 tools.
- Negative paths.

## See also

- E17 amendment §S17.30b
- S17.30a (spec authoring)
- S24.04 (registration mechanism)
