# Story 26.08: Opportunity-Detail UI for Analysis Results

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 5
**Type:** frontend + integration
**Dependencies:** S26.04 (schema), S26.05 (user-trigger endpoint), S26.07 (SSE), S25.09 (citation chips), UX amendment §3 J1 + §5.12 (async-run progress)
**Blocks:** Slice 3 DoD
**Created:** 2026-05-15
**Source:** E26 epic §S26.08

## Story

As **Elena reviewing an ingested opportunity**,
I want **Qualification + Quantification panels rendered on the opportunity-detail page with citations, async-progress UX, and a re-run option**,
so that **I see the AI's pursue/monitor/decline call AND its quantitative case in one screen, with the ability to dig into the KB sources behind it**.

## Acceptance Criteria

1. **Qualification panel** at `frontend/apps/client/app/(protected)/workspace/[workspaceId]/opportunities/[id]/components/QualificationPanel.tsx`:
   - States: **No analysis yet** (CTA "Run qualification") · **Submitted** · **Running (async-progress per UX amendment §5.12)** · **Completed (result)** · **Failed (with error reason + retry CTA)** · **Cached (last run)**.
   - Result-state shows: fit-score gauge (0–100, color-banded), recommended_action badge (pursue/monitor/decline), gap_analysis as bullet list with citation chips per S25.09, confidence band.
2. **Quantification panel** similarly:
   - Tier-gated (Free/Starter shows upgrade CTA; Pro+ shows "Run quantification" only when qualification = `pursue`).
   - Result: effort (person-days), win-probability (with confidence band), expected_value (EUR), recommended_bid_threshold, citations strip.
   - "Why this estimate?" affordance expands the citation list.
3. **Re-run** button on both panels respects tier-gate; sets `force=true` on the trigger.
4. **TanStack Query** invalidation via SSE subscription (S26.07); UI updates on completion without page refresh.
5. **WCAG 2.1 AA**:
   - Keyboard navigation across panels.
   - `aria-live="polite"` on status changes.
   - Fit-score gauge: sufficient contrast for color bands; numeric value also rendered as text.
   - Citation chips per S25.09 + UX amendment §6.X.
6. **E2E Playwright test** (`tests/e2e/opportunity-analysis.spec.ts`):
   - Opportunity ingested (mocked) → auto-qualification fires → UI shows result within 60s.
   - User clicks "Run quantification" → progress state → result appears.
   - Click citation chip → side panel with passage opens.
7. **Stuck state** (per UX amendment §5.12): after 90s of no SSE event AND no terminal webhook, UI shifts to "Still working… (this can take up to 3 minutes)" without auto-failing.

## Dev Notes

### Pattern reuse
- Existing opportunity-detail page in `frontend/apps/client/app/(protected)/workspace/[workspaceId]/opportunities/[id]/`.
- TanStack Query + SSE pattern: existing in proposal-editor.
- Tier-gate UI pattern: existing.
- Citation chips from S25.09.
- Async-run progress component (NEW common surface — likely in `frontend/packages/ui/src/components/AsyncRunProgress.tsx`).

### Files likely touched
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/opportunities/[id]/components/QualificationPanel.tsx` (new)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/opportunities/[id]/components/QuantificationPanel.tsx` (new)
- `frontend/packages/ui/src/components/AsyncRunProgress.tsx` (new)
- `frontend/packages/ui/src/components/FitScoreGauge.tsx` (new)
- `frontend/apps/client/lib/hooks/useOpportunityAnalysis.ts` (new — wraps SSE subscription + TanStack Query)
- `frontend/apps/client/messages/en.json` + `bg.json` (i18n keys for `opportunityAnalysis.*`)
- `tests/e2e/opportunity-analysis.spec.ts`

### Out of scope
- Comparison view across multiple opportunities (post-launch).
- Manual override of AI recommendation (post-launch).

## Risks

- **R1**: Component complexity — break into Panel + Result + Async + Empty states early.
- **R2**: i18n parity — `pnpm check:i18n` gate.

## Testing

- Unit: each panel state.
- E2E: full cascade per AC6.
- Accessibility: axe-core + manual keyboard nav.

## See also

- Epic file §S26.08
- UX amendment §3 J1, §5.12, §6.X
- PRD amendment FR-47, FR-48, FR-51
