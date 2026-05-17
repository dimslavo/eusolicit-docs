# Story 17.34: MCP-Tool Invocation Log + Conflict Resolution

**Status:** backlog
**Epic:** E17 amendment / E27 — CRM via SirmaAI MCP
**Points:** 3
**Type:** backend
**Dependencies:** S17.32
**Blocks:** qual+quant CRM enrichment correctness; S17.35 (dashboard shows conflicts)
**Created:** 2026-05-15
**Source:** E17 amendment §Inject

## Story

As a **platform engineer**,
I want **every MCP-tool invocation logged with outcome AND a Last-Write-Wins conflict-resolution mechanism when CRM webhook reflection diverges from agent-intended state**,
so that **agents enriching opportunities don't fight with provider-side CRM edits AND tenant admins can investigate divergences**.

## Acceptance Criteria

1. Repurpose existing `integrations.conflict_log` table (from original E17). Schema fields: `id`, `company_id`, `provider`, `entity_type`, `entity_id`, `tool_name`, `intended_state` JSONB, `observed_state` JSONB, `resolution` TEXT, `resolved_at`, `created_at`.
2. **MCP-tool invocation log**: SirmaAI returns tool-call outcomes to `sirmaai-gateway`; gateway logs each invocation to `integrations.mcp_invocations` table (NEW or extension of conflict_log): tool_name, tenant_id, outcome, duration_ms, error_message.
3. **Conflict detection**: when a CRM webhook arrives (provider-side change) reflecting an entity that an agent recently modified (within last 60s), compare states. Divergence → log conflict + apply LWW resolution (latest write wins; provider-side wins if newer).
4. **Tenant-visible conflict surface**: existing CRM-dashboard widget (S17.35) renders conflict-log entries; tenant can see "AI created opp X but provider-side merged with manual edit Y".
5. **Resolution outcomes**: `resolution ∈ {kept_intended, kept_observed, merged, manual_review}`.
6. **Idempotency**: conflict log entries deduped on `(provider, entity_id, intended_at)`.

## Dev Notes

### Pattern reuse
- Existing `integrations.conflict_log` from original E17 (verify schema; extend if needed).
- LWW pattern: timestamp-based.

### Files likely touched
- `services/integrations-api/alembic/versions/<next>_mcp_invocation_log.py`
- `services/integrations-api/src/integrations_api/services/conflict_resolver.py` (new or extend existing)
- `services/integrations-api/tests/integration/test_mcp_conflict_resolution.py`

### Out of scope
- Manual-review UI (post-launch).
- Cross-provider entity merging.

## Risks

- **R1**: Clock skew between EU Solicit and provider — LWW could pick the wrong write. Mitigate with conservative window (60s grace).

## Testing

- Integration: agent-write → provider-webhook within 60s → conflict logged + LWW applied.

## See also

- E17 amendment §S17.34
- ADR-020 (CRM via MCP)
