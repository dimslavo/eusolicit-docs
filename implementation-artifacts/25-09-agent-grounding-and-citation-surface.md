# Story 25.09: Agent Grounding + Citation Surface

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 5
**Type:** backend + integration + frontend
**Dependencies:** S25.04 (search + download endpoints), S26.08 (opportunity detail UI)
**Blocks:** Slice 2 + Slice 3 DoD
**Created:** 2026-05-15
**Source:** E25 epic §S25.09

## Story

As **Elena reviewing the AI's qualification output**,
I want **every claim grounded in my KB to show a clickable citation chip that opens the source passage**,
so that **I can sanity-check the AI before pursuing the bid AND I never have to take an AI assertion on faith**.

## Acceptance Criteria

1. **Agent output post-processing in `sirmaai-gateway`**: when a SirmaAI agent run returns a structured output containing `kb_citations: [{file_id, passage_excerpt, relevance}]`, enrich each citation with EU-Solicit-side metadata:
   - Look up `client.sirmaai_kb_files` by `sirmaai_file_id` → return `{kb_file_id, filename, artefact_category, archived_at}`.
   - If row not found (orphaned citation — file deleted after run): return `{kb_file_id: null, filename: "[Source no longer accessible]", archived_at: <now>}`.
2. **Citation contract** (added to qualifier + quantifier + ESPD + proposal-drafter response schemas):
   ```json
   {
     "kb_citations": [
       {
         "kb_file_id": "uuid",
         "sirmaai_file_id": "...",
         "filename": "past-proposal-2024-12.pdf",
         "artefact_category": "proposal",
         "passage_excerpt": "...environmental impact assessment...",
         "relevance": 0.87,
         "archived_at": null
       }
     ]
   }
   ```
3. **Frontend citation-chip pattern** (per UX amendment §6.X):
   - Inline `[¹]` markers next to grounded assertions.
   - Hover tooltip: filename + category badge + excerpt.
   - Click opens side panel with full passage + "Open in KB" deep-link to KB artefact detail.
   - Archived-source state: chip greyed out, copy "Source archived" + last-known-filename.
   - Keyboard: focusable, Enter opens side panel, Esc closes.
4. **Integration**: opportunity-detail page (S26.08) Qualification + Quantification panels render citations using this pattern.
5. **Integration test** (mocked SirmaAI):
   - Agent returns 3 KB citations → backend enriches → frontend renders 3 chips.
   - One citation references an archived file → "Source archived" state visible.
   - Click-through on a non-archived citation → side panel opens with passage + KB-deep-link.
6. **Snapshot tests** for citation JSON format (frozen contract for frontend) under `services/sirmaai-gateway/tests/snapshots/`.
7. **WCAG 2.1 AA**: citation chips have `aria-label` describing relationship; side panel `role="dialog"` with focus trap; tooltip dismisses on Esc.

## Dev Notes

### Pattern reuse
- Agent response post-processing in `sirmaai-gateway` — extend existing structured-output pipeline.
- Side-panel pattern: see proposal-draft side panel in Epic 7.

### Files likely touched
- `services/sirmaai-gateway/src/sirmaai_gateway/services/citation_enricher.py` (new)
- `services/sirmaai-gateway/src/sirmaai_gateway/routers/agents.py` (post-process response)
- `frontend/packages/ui/src/components/kb/CitationChip.tsx` (new)
- `frontend/packages/ui/src/components/kb/CitationSidePanel.tsx` (new)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/opportunities/[id]/components/QualificationPanel.tsx` (use chips)
- `services/sirmaai-gateway/tests/snapshots/citation_format.json` (new)
- `tests/e2e/kb-citations.spec.ts`

### Out of scope
- Multi-passage citation merging (one chip per file even if multiple passages — defer to v2).
- AI-driven "is this citation strong enough?" relevance threshold filtering (return ALL citations; let user judge).

## Risks

- **R1**: SirmaAI citation format may evolve — snapshot tests catch the drift; pin contract version in citation schema.
- **R2**: Performance: citation enrichment adds 1 DB query per file_id. Batch via `IN` clause.

## Testing

- Unit: enricher with multiple citations + archived state.
- Integration: agent → enrichment → frontend render.
- Snapshot: frozen citation contract.

## See also

- Epic file §S25.09
- UX amendment §6.X (citation chip pattern)
- PRD amendment FR-51
- E26 S26.08 (UI consumer)
