# Story 26.01: Opportunity Qualifier SirmaAI Agent Definition + Template

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 3
**Type:** backend + prompt-engineering
**Dependencies:** E26 baseline dataset spec curation (`test-artifacts/e26-baseline-datasets-spec.md`); S24.03 (template seed mechanism)
**Blocks:** S26.03 (workflow), S26.05 (user trigger)
**Created:** 2026-05-15
**Source:** E26 epic §S26.01

## Story

As **Elena seeing a new tender ingested into her tenant**,
I want **an AI qualifier to auto-produce a structured assessment (fit / gap / recommend) within 30 seconds**,
so that **I can triage 30+ daily ingested opportunities in minutes rather than hours of reading**.

## Acceptance Criteria

1. **Prerequisite** (added 2026-05-15): `services/client-api/tests/data/sirmaai-baselines/v1/qualifier-baseline-v1.json` must exist (20 opportunities + SME labels per `test-artifacts/e26-baseline-datasets-spec.md`).
2. Qualifier agent definition authored as a SirmaAI Project template entry committed to `services/client-api/config/sirmaai_project_template.yaml`:
   ```yaml
   agents:
     - logical_name: opportunity_qualifier
       sirmaai_definition:
         model: <SirmaAI-recommended foundation model>
         system_prompt: |
           You are an EU procurement qualification agent. Given a normalised opportunity
           and a company profile, produce a structured assessment:
           ...
         tools:
           - kb_search
         structured_output_schema:
           type: object
           required: [fit_score, gap_analysis, recommended_action, confidence, kb_citations]
           properties:
             fit_score: { type: integer, minimum: 0, maximum: 100 }
             gap_analysis: { type: array, items: { type: string } }
             recommended_action: { type: string, enum: [pursue, monitor, decline] }
             confidence: { type: string, enum: [low, medium, high] }
             kb_citations: ...
   ```
3. Tool binding: `kb_search` (allows the agent to retrieve company profile + past proposals + qualification rubrics from the tenant KB per S25.04).
4. Eval-runs harness in SirmaAI evaluates the agent against `qualifier-baseline-v1.json`:
   - Alignment metric per E26 baseline spec: 0.6·action-match + 0.3·fit-band-match + 0.1·gap-keyword-overlap.
   - Aggregate alignment ≥85% across 20 opportunities for AC to pass.
5. **Pydantic schema** on EU Solicit side at `services/sirmaai-gateway/src/sirmaai_gateway/schemas/qualifier_output.py` mirrors the structured_output_schema; validates every agent run on response — invalid output → `failed` status (no silent partial result).
6. **Regression protocol**: every change to the agent prompt OR template version triggers re-eval; if alignment drops below 85%, PR cannot merge.
7. **Pre-launch checkpoint**: at least one SME-paired evaluation has confirmed the eval-run baseline aligns with human judgment (this is the SME work in the baseline dataset spec).
8. Agent definition propagation: when S24.03 applies the template, qualifier agent is seeded into every tenant's Project; verified via integration test on staging.

## Dev Notes

### Pattern reuse
- YAML schema validated by Pydantic at boot.
- SME-paired eval harness pattern: see if any existing pattern in `test_artifacts/` for similar.

### Files likely touched
- `services/client-api/config/sirmaai_project_template.yaml` (extend with `opportunity_qualifier`)
- `services/sirmaai-gateway/src/sirmaai_gateway/schemas/qualifier_output.py` (new — Pydantic)
- `services/client-api/tests/data/sirmaai-baselines/v1/qualifier-baseline-v1.json` (curated by separate effort — referenced)
- `services/sirmaai-gateway/tests/integration/test_qualifier_eval_run.py`

### Out of scope
- The N8N workflow that orchestrates the run (S26.03).
- User-trigger endpoint (S26.05).
- Auto-trigger on ingestion (S26.06).
- Frontend display (S26.08).

## Risks

- **R1**: SME unavailable → baseline curation falls back to single-rater (v1); document the lower-confidence posture in `project-context.md`.
- **R2**: Foundation model deprecation / version-pinning at SirmaAI side — coordinate with SirmaAI on model lifecycle.

## Testing

- Eval-run: ≥85% baseline alignment.
- Unit: Pydantic schema rejects malformed agent output.
- Integration: agent seeded into staging tenant Project; one-shot test query returns valid output.

## See also

- Epic file §S26.01
- E26 baseline dataset spec (`test-artifacts/e26-baseline-datasets-spec.md`)
- PRD amendment FR-47
