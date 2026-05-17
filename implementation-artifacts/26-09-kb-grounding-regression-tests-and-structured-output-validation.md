# Story 26.09: KB-Grounding Regression Tests + Structured-Output Validation

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 3
**Type:** integration / test
**Dependencies:** S26.01..S26.03 done
**Blocks:** Sprint 5 quality gate
**Created:** 2026-05-15
**Source:** E26 epic §S26.09

## Story

As **Murat (TEA)**,
I want **a regression test suite that asserts qualification + quantification ground in the tenant KB when artefacts are present, surface citations correctly, and reject malformed structured output**,
so that **agent-prompt drift or SirmaAI-side regressions are caught before they reach production**.

## Acceptance Criteria

1. Integration test suite (`services/sirmaai-gateway/tests/integration/test_agent_grounding_regression.py`) with mocked SirmaAI responses covering ≥20 scenarios:
   - Qualification with company-profile KB → output cites profile.
   - Qualification with past-proposal KB → output cites past-proposal.
   - Quantification with thin KB → output `confidence=low`, `caveat="low_kb_context"`.
   - Quantification with rich KB → output `confidence=medium|high`.
   - Quantification with mocked MCP CRM data → output considers it.
   - Quantification with NO MCP data → output graceful-fallback caveat.
   - Malformed agent output → Pydantic raises → `record_failure` path triggered.
   - Mid-stream cancel scenario.
   - SirmaAI 429 retry scenario.
2. **Snapshot tests** for citation format (frozen contract for frontend) at `services/sirmaai-gateway/tests/snapshots/citation_format.json` per S25.09.
3. **Citation correctness**: assertions that `kb_citations[*].file_id` references actual `client.sirmaai_kb_files.id` rows (post-enrichment per S25.09).
4. **Schema validation fail-fast**: tests assert that any required field missing → fail-fast (no silent partial result).
5. **Tests run on every CI build** as part of `make test-integration` in the sirmaai-gateway test suite.
6. **Performance**: test suite completes in < 3 min (mocks only; no real SirmaAI calls).

## Dev Notes

### Pattern reuse
- pytest fixtures for mock SirmaAI responses: extend existing patterns.
- Snapshot test pattern: use `syrupy` or simple JSON diff.

### Files likely touched
- `services/sirmaai-gateway/tests/integration/test_agent_grounding_regression.py` (new)
- `services/sirmaai-gateway/tests/fixtures/mock_agent_responses/` (new directory with sample responses)
- `services/sirmaai-gateway/tests/snapshots/citation_format.json` (new)
- `services/sirmaai-gateway/conftest.py` (extend with shared fixtures)

### Out of scope
- Eval-runs against real SirmaAI (those are owned by S26.01/02 baseline tests).
- E2E end-user testing (S26.08 covers).

## Risks

- **R1**: Brittle snapshot tests — keep them coarse (key set + value type), not fine-grained content.

## Testing

- Self-testing: the test suite IS the deliverable. Coverage threshold: ≥20 scenarios passing.

## See also

- Epic file §S26.09
- S26.01 + S26.02 (agent definitions)
- S25.09 (citation contract)
