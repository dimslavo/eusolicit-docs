# Story 26.02: Opportunity Quantifier SirmaAI Agent Definition + Template

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 3
**Type:** backend + prompt-engineering
**Dependencies:** E26 baseline dataset spec curation; S24.03 (template seed mechanism); optionally S17.34 (MCP-tool log for CRM context — graceful-skip if not landed)
**Blocks:** S26.03 (workflow), S26.05 (user trigger)
**Created:** 2026-05-15
**Source:** E26 epic §S26.02

## Story

As **Elena who has qualified an opportunity as `pursue`**,
I want **the platform to quantify effort, win probability, and expected value with confidence indicators**,
so that **I can defend a bid/no-bid decision to my partners with concrete numbers grounded in our historical data**.

## Acceptance Criteria

1. **Prerequisite**: `services/client-api/tests/data/sirmaai-baselines/v1/quantifier-baseline-v1.json` must exist (15 opportunities + 2 synthetic adversarial + historical actuals per E26 baseline spec).
2. Quantifier agent definition committed to `services/client-api/config/sirmaai_project_template.yaml`:
   - Tools: `kb_search`, optionally `mcp_dynamics365.find_account`, `mcp_hubspot.find_account` (for past-deal-history; graceful-skip if MCP servers inactive or not registered).
   - Structured output schema with required fields: `estimated_effort_person_days`, `estimated_win_probability`, `expected_value_eur`, `recommended_bid_threshold_eur`, `confidence`, `caveat`, `kb_citations`.
3. **Pydantic schema** at `services/sirmaai-gateway/src/sirmaai_gateway/schemas/quantifier_output.py`; validates response shape.
4. **Eval-runs** against quantifier baseline:
   - Effort estimate mean error ≤ ±25% across 15 production opportunities.
   - Expected-value mean error ≤ ±25%.
   - Brier-score reported but not gated (calibration needs N ≥ 50; baseline has only 15).
5. **Fallback-confidence test** for the 2 synthetic thin-KB opportunities:
   - Agent MUST return `confidence ∈ {low, medium}` AND `caveat: "low_kb_context"`.
   - Overconfidence (confidence=high with no KB context) is a fail signal — eval-run regression catches it.
6. **Graceful MCP-absent path**: when no CRM MCP server is registered (status=inactive), agent runs with KB only and surfaces `caveat: "no_crm_data"` in output. Verified via integration test mocking 0 MCP servers.
7. **Tier-aware caveat**: quantifier is Pro+ tier-gated (S26.05); agent definition has no tier knowledge — gating is at endpoint layer.
8. Agent propagation: applied via S24.03 template seed; verified on staging.

## Dev Notes

### Pattern reuse
- Same shape as S26.01.
- SirmaAI MCP-tool call binding pattern: per E27 conventions.

### Files likely touched
- `services/client-api/config/sirmaai_project_template.yaml` (extend with `opportunity_quantifier`)
- `services/sirmaai-gateway/src/sirmaai_gateway/schemas/quantifier_output.py` (new — Pydantic)
- `services/sirmaai-gateway/tests/integration/test_quantifier_eval_run.py`

### Out of scope
- The workflow + user trigger + auto-trigger (S26.03, S26.05, S26.06).
- Frontend display (S26.08).
- Brier-score calibration gate (N too small for v1).

## Risks

- **R1**: Effort estimation is hard without `bid_preparation_logs` data — quantifier may need to surface `confidence=low` until E19 telemetry table populates.
- **R2**: MCP-tool binding may have SirmaAI-side coupling — verify with `27-08-agent-side-mcp-tool-consumption-patterns` story.

## Testing

- Eval-run: ±25% effort + EV; thin-KB caveat surfaces.
- Integration: quantifier with mocked CRM tools returning data, no-CRM, partial-CRM.

## See also

- Epic file §S26.02
- E26 baseline dataset spec
- E27 §S27.08 (MCP consumption patterns)
- PRD amendment FR-48
