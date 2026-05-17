# Story 27.08: Agent-Side MCP-Tool Consumption Patterns

**Status:** backlog
**Epic:** E27 (E27-unique — not aliased to E17 amendment)
**Points:** 3
**Type:** backend + integration
**Dependencies:** S17.30b + S17.31 (specs registered), S26.02 (quantifier definition)
**Blocks:** quality gate for CRM-enriched quantification
**Created:** 2026-05-15
**Source:** E27 epic §S27.08

## Story

As a **prompt engineer authoring agent definitions**,
I want **documented patterns + eval-runs for how qualification, quantification, and on-demand enrichment agents call MCP CRM tools mid-run**,
so that **the agents correctly use CRM data when present AND degrade gracefully when not, with a frozen contract for the prompt + tool-binding format**.

## Acceptance Criteria

1. Documentation at `eusolicit-docs/agent-patterns/sirmaai-mcp-tool-consumption.md` covering:
   - **Discovery pattern**: how an agent identifies which MCP servers are available (via SirmaAI Project metadata).
   - **Invocation pattern**: how to call `mcp_dynamics365.find_account(name="...")` vs `mcp_hubspot.find_account(name="...")`.
   - **Fallback pattern**: when no MCP server is registered, agent must surface `caveat="no_crm_data"` in output.
   - **Multi-provider scenario**: if both providers are active, prefer the one with a connected_since older than 30 days (assume more populated history).
2. **Eval-runs** mocking each scenario:
   - Both providers active + data found → agent uses CRM data; quantification reflects.
   - Both providers active + no match → agent surfaces `caveat="no_crm_match"`.
   - One provider active + match → uses it.
   - No provider active → graceful fallback.
3. **Frozen contract**: the prompt + tool-binding pattern in agent definitions follows the documented form. Lint: a script asserts every agent definition mentioning MCP tools follows the contract.
4. **Integration tests** at `services/sirmaai-gateway/tests/integration/test_agent_mcp_consumption.py`.

## Dev Notes

### Pattern reuse
- Existing prompt-engineering patterns in `services/client-api/config/sirmaai_project_template.yaml`.

### Files likely touched
- `eusolicit-docs/agent-patterns/sirmaai-mcp-tool-consumption.md` (new)
- `services/sirmaai-gateway/tests/integration/test_agent_mcp_consumption.py` (new)
- `scripts/lint_agent_definitions.py` (extend with MCP contract check)

### Out of scope
- The 5 specific tool implementations (S17.30a, S17.31).
- Token vault / rotation (S17.32, S17.33).

## Risks

- **R1**: Pattern documentation drift — lint script enforces.

## Testing

- Eval-runs per AC2.

## See also

- E27 §S27.08 (E27-unique)
- E26 §S26.02 (quantifier consumes this pattern)
