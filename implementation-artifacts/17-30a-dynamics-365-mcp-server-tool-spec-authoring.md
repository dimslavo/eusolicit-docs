# Story 17.30a: Dynamics 365 MCP Server — Tool Spec Authoring

**Status:** backlog
**Epic:** E17 amendment / E27 — CRM via SirmaAI MCP
**Points:** 4
**Type:** backend
**Dependencies:** S04.20 done; planning-only (can start in parallel with E24)
**Blocks:** S17.30b (registration consumes this); S24.04 (MCP stub registration reads the spec)
**Created:** 2026-05-15 (SPLIT from former 8pt S17.30 per IR remediation Issue #7)
**Source:** E17 amendment §Inject + E27 §Stories (canonical S17.* keys)

## Story

As an **EU Solicit agent author**,
I want **a versioned, PR-reviewed MCP tool-spec for Microsoft Dynamics 365 covering 5 tools (`find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`)**,
so that **the SirmaAI Project registration flow (S17.30b) has a stable artefact to register, and agents have a typed contract to call against**.

## Acceptance Criteria

1. Author tool spec YAML at `services/integrations-api/config/sirmaai_mcp_specs/dynamics365.yaml`. Structure:
   ```yaml
   version: "v1"
   provider: dynamics365
   description: "Microsoft Dynamics 365 (Dataverse Web API)"
   tools:
     - name: find_account
       input_schema: { ... }
       output_schema: { ... }
       error_mapping:
         "400": "INVALID_QUERY"
         "401": "AUTH_FAILED"
         ...
       dataverse_api:
         method: GET
         path: /accounts
         ...
     - name: create_deal
       ...
     # ...3 more
   ```
2. **5 tools authored**: `find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`. Each tool's input + output schema is JSON Schema (draft-07); error-mapping table maps Dataverse error codes → MCP error responses.
3. **Dynamics-specific fields** captured in input schemas: entity types (`account`, `opportunity`, `contact`), custom field IDs (parameterised per tenant if needed), OptionSet enums.
4. **Reference**: Microsoft Dataverse Web API documented in spec header comment block (link + version).
5. **Pydantic validator** loads + validates the YAML at boot — invalid spec fails CI.
6. **PR review**: spec must include reviewer sign-off from someone with Dynamics knowledge.

## Dev Notes

### Pattern reuse
- YAML-as-spec pattern (similar to `services/client-api/config/sirmaai_project_template.yaml`).
- JSON Schema draft-07 for input/output validation.

### Files likely touched
- `services/integrations-api/config/sirmaai_mcp_specs/dynamics365.yaml` (new)
- `services/integrations-api/src/integrations_api/services/mcp_spec_loader.py` (new — shared with S17.31 HubSpot)
- `services/integrations-api/tests/unit/test_dynamics365_spec.py`

### Out of scope
- Registration flow (S17.30b).
- OAuth (S17.32).
- Agent-side consumption patterns (S27.08).

## Risks

- **R1**: Dynamics 365 API access for testing — coordinate procurement of a Dynamics dev tenant (PM R7 in build-sequence).

## Testing

- Unit: spec loader validates the YAML.
- Schema validation: each tool's input/output passes JSON Schema validation.

## See also

- E17 amendment §S17.30a (this story)
- E27 §S17.30a in canonical-keys table
- Architecture amendment ADR-020 (CRM via MCP)
