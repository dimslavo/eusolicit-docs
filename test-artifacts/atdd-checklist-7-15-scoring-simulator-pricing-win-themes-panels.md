# ATDD Checklist: Story 7.15 - Scoring Simulator, Pricing & Win Themes Panels

**Story:** [7.15: Scoring Simulator, Pricing & Win Themes Panels](/home/debian/Projects/eusolicit/eusolicit-docs/implementation-artifacts/7-15-scoring-simulator-pricing-win-themes-panels.md)

This checklist is derived from the acceptance criteria for the story and will be verified by the ATDD test file `__tests__/scoring-pricing-win-themes-s7-15.test.ts`. These tests are expected to fail until the story is implemented.

---

### UI Component Creation & Placement (AC 1, 8, 15)

- [ ] **AC 1:** The placeholder `<div data-testid="scoring-panel-placeholder">` is removed from `ProposalWorkspacePage.tsx`.
- [ ] **AC 1:** `ProposalWorkspacePage.tsx` dynamically imports `ScoringSimulatorPanel` from `./ScoringSimulatorPanel`.
- [ ] **AC 1:** `ScoringSimulatorPanel` is rendered as `<ScoringSimulatorPanel proposalId={proposal.id} />`.
- [ ] **AC 1:** `frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/ScoringSimulatorPanel.tsx` exists and contains a `"use client"` directive.
- [ ] **AC 8:** The placeholder `<div data-testid="pricing-panel-placeholder">` is removed from `ProposalWorkspacePage.tsx`.
- [ ] **AC 8:** `ProposalWorkspacePage.tsx` dynamically imports `PricingPanel` from `./PricingPanel`.
- [ ] **AC 8:** `PricingPanel` is rendered as `<PricingPanel proposalId={proposal.id} />`.
- [ ] **AC 8:** `frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/PricingPanel.tsx` exists and contains a `"use client"` directive.
- [ ] **AC 15:** The placeholder `<div data-testid="win-themes-panel-placeholder">` is removed from `ProposalWorkspacePage.tsx`.
- [ ] **AC 15:** `ProposalWorkspacePage.tsx` dynamically imports `WinThemesPanel` from `./WinThemesPanel`.
- [ ] **AC 15:** `WinThemesPanel` is rendered as `<WinThemesPanel proposalId={proposal.id} />`.
- [ ] **AC 15:** `frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/WinThemesPanel.tsx` exists and contains a `"use client"` directive.

### Scoring Simulator Panel (AC 2-7)

- [ ] **AC 2:** `ScoringSimulatorPanel.tsx` contains a button with `data-testid="btn-simulate-score"`.
- [ ] **AC 2:** The component logic calls `GET /api/v1/proposals/:id/scoring-simulation` on mount.
- [ ] **AC 3:** `ScoringSimulatorPanel.tsx` imports `RadarChart` from `recharts`.
- [ ] **AC 3:** The `RadarChart` is rendered inside a `ResponsiveContainer` and the outer div has `data-testid="scoring-radar-chart"`.
- [ ] **AC 4:** A table with `data-testid="scoring-criteria-table"` is rendered.
- [ ] **AC 4:** Table rows have `data-testid="scoring-row-{i}"`.
- [ ] **AC 4:** The component source contains the logic `criterion.score / criterion.max_score < 0.70` (or `0.7`).
- [ ] **AC 4:** Rows matching the low-score condition have `data-amber="true"`.
- [ ] **AC 5:** A "Re-simulate" button with `data-testid="btn-resimulate-score"` is present.
- [ ] **AC 6:** A skeleton loading element with `data-testid="scoring-loading"` is present.
- [ ] **AC 6:** An error message element with `data-testid="scoring-error"` is present.
- [ ] **AC 6:** A retry button with `data-testid="btn-scoring-retry"` is present.
- [ ] **AC 7:** `ProposalEditorToolbar.tsx` has an `onScoreClick?: () => void` prop.
- [ ] **AC 7:** `ProposalWorkspacePage.tsx` contains a handler `handleScoreClick` that calls `setRightActiveTab("scoring")`.

### Pricing Assistant Panel (AC 8-14)

- [ ] **AC 9:** `PricingPanel.tsx` contains a button with `data-testid="btn-get-pricing"`.
- [ ] **AC 9:** The component logic calls the pricing `GET` endpoint on mount.
- [ ] **AC 10:** An element with `data-testid="pricing-recommended-price"` is rendered.
- [ ] **AC 10:** `PricingPanel.tsx` uses `Intl.NumberFormat` with the `"de-DE"` locale for currency formatting.
- [ ] **AC 11:** A `div`-based market range bar with `data-testid="pricing-market-range"` is rendered.
- [ ] **AC 11:** The component source contains the `toPercent` calculation logic.
- [ ] **AC 12:** An element with `data-testid="pricing-justification"` is rendered to display the justification text.
- [ ] **AC 13:** A "Re-advise" button with `data-testid="btn-readvise-pricing"` is present.
- [ ] **AC 14:** A skeleton loading element with `data-testid="pricing-loading"` is present.
- [ ] **AC 14:** An error message element with `data-testid="pricing-error"` is present.
- [ ] **AC 14:** A retry button with `data-testid="btn-pricing-retry"` is present.

### Win Themes Panel (AC 15-20)

- [ ] **AC 16:** `WinThemesPanel.tsx` contains a button with `data-testid="btn-extract-win-themes"`.
- [ ] **AC 16:** The component logic calls the win themes `GET` endpoint on mount.
- [ ] **AC 17:** A list with `data-testid="win-themes-list"` is rendered.
- [ ] **AC 17:** Theme cards have `data-testid="win-theme-card-{index}"`.
- [ ] **AC 17:** Drag handles with `data-testid="win-theme-drag-handle-{index}"` are present.
- [ ] **AC 17:** `WinThemesPanel.tsx` imports `GripVertical` from `lucide-react`.
- [ ] **AC 18:** The component uses the `draggable` attribute for HTML5 drag-and-drop.
- [ ] **AC 18:** The component uses `useRef` to store the dragged index (NOT `useState`).
- [ ] **AC 19:** A "Re-extract" button with `data-testid="btn-reextract-win-themes"` is present.
- [ ] **AC 20:** A skeleton loading element with `data-testid="win-themes-loading"` is present.
- [ ] **AC 20:** An error message element with `data-testid="win-themes-error"` is present.
- [ ] **AC 20:** A retry button with `data-testid="btn-win-themes-retry"` is present.

### Shared Concerns (AC 21-22)

- [ ] **AC 21:** All required i18n keys (e.g., `scoringTitle`, `pricingBtnAdvise`, `winThemesEmpty`) are present in `messages/en.json` under the `proposals` namespace.
- [ ] **AC 21:** All required i18n keys are also present in `messages/bg.json`.
- [ ] **AC 22:** The ATDD test file `__tests__/scoring-pricing-win-themes-s7-15.test.ts` exists.
- [ ] **AC 22:** `lib/api/proposals.ts` exports the required interfaces (`ScoreCardCriterion`, `ScoringSimulationResponse`, `PricingAssistResponse`, `WinTheme`, `WinThemesResponse`).
- [ ] **AC 22:** `lib/queries/use-proposals.ts` exports the required React Query hooks (`useScoringResults`, `useRunScoringSimulation`, etc.).
