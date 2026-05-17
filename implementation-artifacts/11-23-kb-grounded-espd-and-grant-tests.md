# Story 11.23: KB-Grounded ESPD + Grant Tests

**Status:** backlog
**Epic:** E11 amendment
**Points:** 3
**Type:** backend + integration
**Dependencies:** S11.20, S11.21, S11.22 done; S25.09 (citation surface)
**Blocks:** Sprint 5 quality gate
**Created:** 2026-05-15
**Source:** E11 epic §S11.23

## Story

As **Murat (TEA)**,
I want **integration tests verifying that ESPD Auto-Fill + Grant Eligibility agents consume the tenant KB when present and surface citations correctly**,
so that **the grant/compliance migration to SirmaAI doesn't silently regress on KB-grounding behavior**.

## Acceptance Criteria

1. Integration test suite at `services/client-api/tests/integration/test_kb_grounded_grant_compliance.py` covering:
   - ESPD Auto-Fill with uploaded ESPD-template artefact in KB → output cites the template.
   - Grant Eligibility with uploaded company-profile artefact in KB → output cites profile sectors.
   - ESPD Auto-Fill with NO ESPD templates in KB → output proceeds but surfaces `caveat="no_kb_template"`.
   - Grant Eligibility with NO profile in KB → output uses the structured profile fields only + surfaces appropriate caveat.
2. **Citation correctness**: each test asserts `kb_citations[*].file_id` references an actual `client.sirmaai_kb_files.id`.
3. **Tests use the citation enrichment** from S25.09.
4. **Test fixtures** for KB artefacts: small synthetic ESPD template + synthetic company profile artefact committed at `services/client-api/tests/data/fixtures/kb_artefacts/`.
5. **Tests run on every CI build**.

## Dev Notes

### Pattern reuse
- E26 S26.09 kb-grounding regression-test pattern.
- Test fixtures: similar to E26 baseline-dataset spec.

### Files likely touched
- `services/client-api/tests/integration/test_kb_grounded_grant_compliance.py` (new)
- `services/client-api/tests/data/fixtures/kb_artefacts/` (new — ESPD + profile samples)

### Out of scope
- Eval-runs against production data (post-launch).

## Testing

- Self-testing.

## See also

- E11 epic §S11.23
- S25.09, S26.09 (parallel patterns)
