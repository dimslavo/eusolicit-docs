---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
lastStep: step-04c-aggregate
lastSaved: '2026-04-18'
story: 7-15-scoring-simulator-pricing-win-themes-panels
storyFile: eusolicit-docs/implementation-artifacts/7-15-scoring-simulator-pricing-win-themes-panels.md
testFile: eusolicit-app/frontend/apps/client/__tests__/scoring-pricing-win-themes-s7-15.test.ts
epicTestDesignIds:
  - E07-P2-010
  - E07-P2-011
  - E07-P2-012
tddPhase: GREEN   # All 86 tests pass — implementation complete
totalTests: 86
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-15-scoring-simulator-pricing-win-themes-panels.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/frontend/apps/client/__tests__/requirement-checklist-compliance-s7-14.test.ts
  - eusolicit-app/frontend/apps/client/__tests__/ai-draft-generation-s7-13.test.ts
  - eusolicit-docs/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 7.15 — Scoring Simulator, Pricing & Win Themes Panels

**Epic:** 07 — Proposal Generation & Workspace  
**Epic Test Design IDs:** E07-P2-010, E07-P2-011, E07-P2-012  
**Test File:** `eusolicit-app/frontend/apps/client/__tests__/scoring-pricing-win-themes-s7-15.test.ts`  
**Test Count:** 86 tests (source-inspection pattern — same as S07.13, S07.14)  
**TDD Status:** ✅ GREEN — All 86 tests pass; implementation is complete.

---

## Stack & Generation Mode

- **Stack Detected:** `fullstack` (Python FastAPI backend + Next.js 14 frontend)
- **Story Type:** Pure-frontend — all backend endpoints delivered in S07.07 and S07.08
- **Generation Mode:** AI generation — source-inspection tests (no browser recording needed)
- **Test Framework:** Vitest (node environment, `globals: true`)

---

## Acceptance Criteria Coverage

### AC 1, 8, 15 — Component Files Exist + Placeholders Removed

- [x] `ScoringSimulatorPanel.tsx` exists in `app/[locale]/(protected)/proposals/[id]/components/`
- [x] `ScoringSimulatorPanel.tsx` has `"use client"` directive at top
- [x] `PricingPanel.tsx` exists with `"use client"` directive
- [x] `WinThemesPanel.tsx` exists with `"use client"` directive
- [x] `ProposalWorkspacePage.tsx` does NOT contain `"Scoring — S07.15"` placeholder
- [x] `ProposalWorkspacePage.tsx` does NOT contain `"Pricing — S07.15"` placeholder
- [x] `ProposalWorkspacePage.tsx` does NOT contain `"Win Themes — S07.15"` placeholder
- [x] `ProposalWorkspacePage.tsx` contains dynamic imports for all three panels (`ssr: false`)
- [x] `ProposalWorkspacePage.tsx` renders `<ScoringSimulatorPanel>`, `<PricingPanel>`, `<WinThemesPanel>`

### AC 2, 9, 16 — API Types & Functions (`lib/api/proposals.ts`)

- [x] Exports `interface ScoreCardCriterion` (fields: `criterion`, `score`, `max_score`, `suggestion` — **no `id`**)
- [x] Exports `interface ScoringSimulationResponse` (fields: `criteria`, `simulated_at`)
- [x] Exports `interface MarketRange` (fields: `min`, `median`, `max`)
- [x] Exports `interface PricingAssistResponse` (fields: `recommended_price`, `market_range`, `justification`, `priced_at`)
- [x] Exports `interface WinTheme` (fields: `title`, `description`)
- [x] Exports `interface WinThemesResponse` (fields: `themes`, `analyzed_at`)
- [x] Exports `async function runScoringSimulation(proposalId)`
- [x] Exports `async function getScoringResults(proposalId)` — null-result detection uses `"result" in data && data.result === null`
- [x] Exports `async function runPricingAssist(proposalId)`
- [x] Exports `async function getPricingResults(proposalId)` — same null-result pattern
- [x] Exports `async function runWinThemes(proposalId)`
- [x] Exports `async function getWinThemes(proposalId)` — same null-result pattern
- [x] `ScoreCardCriterion` has **no `id:` field** (backend schema omits it)

### AC 2, 9, 16 — React Query Hooks (`lib/queries/use-proposals.ts`)

- [x] Exports `useScoringResults(proposalId)` — `queryKey: ["scoring", proposalId]`
- [x] Exports `useRunScoringSimulation(proposalId)` — `useMutation`; invalidates `["scoring"]` on success
- [x] Exports `usePricingResults(proposalId)` — `queryKey: ["pricing", proposalId]`
- [x] Exports `useRunPricingAssist(proposalId)` — `useMutation`; invalidates `["pricing"]` on success
- [x] Exports `useWinThemesResults(proposalId)` — `queryKey: ["win-themes", proposalId]`
- [x] Exports `useRunWinThemes(proposalId)` — `useMutation`; invalidates `["win-themes"]` on success
- [x] Mutation hooks call `invalidateQueries` on success

### AC 1–7 — `ScoringSimulatorPanel.tsx`

- [x] `data-testid="btn-simulate-score"` present (trigger button)
- [x] `data-testid="scoring-radar-chart"` present (wrapper div around RadarChart)
- [x] `data-testid="scoring-criteria-table"` present
- [x] Template-literal row testid: `data-testid={\`scoring-row-${i}\`}` (zero-based index)
- [x] `data-testid="btn-resimulate-score"` present
- [x] `data-testid="scoring-loading"` present (skeleton while running)
- [x] `data-testid="scoring-error"` present (error state)
- [x] `data-testid="btn-scoring-retry"` present (retry on failure)
- [x] `RadarChart` imported from `recharts` (`import { RadarChart, ... } from "recharts"`)
- [x] `ResponsiveContainer` with `height={220}` and `domain={[0, 100]}`
- [x] Radar data computed as `(criterion.score / criterion.max_score) * 100`
- [x] Amber threshold: `criterion.score / criterion.max_score < 0.70` (or `0.7`)
- [x] `data-amber=` attribute on amber rows (any assignment form)
- [x] `text-amber-600` class on amber score cell
- [x] `useTranslations("proposals")` used for all text strings

### AC 8–14 — `PricingPanel.tsx`

- [x] `data-testid="btn-get-pricing"` present
- [x] `data-testid="pricing-recommended-price"` present
- [x] `data-testid="pricing-market-range"` present (div-based, no chart library)
- [x] `data-testid="pricing-justification"` present
- [x] `data-testid="btn-readvise-pricing"` present
- [x] `data-testid="pricing-loading"` present
- [x] `data-testid="pricing-error"` present
- [x] `data-testid="btn-pricing-retry"` present
- [x] `Intl.NumberFormat` with `"de-DE"` locale and `"EUR"` currency
- [x] **NOT** using `"en-EU"` (invalid BCP 47 locale)
- [x] `toPercent` helper using `Math.max` / `Math.min` clamp
- [x] Market range bar uses `div` elements — recharts NOT imported in `PricingPanel.tsx`
- [x] `useTranslations("proposals")` used for all text strings

### AC 15–20 — `WinThemesPanel.tsx`

- [x] `data-testid="btn-extract-win-themes"` present
- [x] `data-testid="win-themes-list"` present
- [x] Template-literal card testid: `data-testid={\`win-theme-card-${index}\`}` (zero-based)
- [x] Template-literal drag handle: `data-testid={\`win-theme-drag-handle-${index}\`}`
- [x] `data-testid="btn-reextract-win-themes"` present
- [x] `data-testid="win-themes-loading"` present
- [x] `data-testid="win-themes-error"` present
- [x] `data-testid="btn-win-themes-retry"` present
- [x] `GripVertical` imported from `lucide-react`
- [x] HTML5 Drag and Drop API used: `draggable`, `onDragStart`, `onDragOver`, `onDrop`
- [x] Drag index stored in `useRef<number | null>` (NOT `useState` — prevents re-render storm)
- [x] `dragIndexRef` variable name used for the ref
- [x] `useTranslations("proposals")` used for all text strings

### AC 7 — Toolbar Wiring

- [x] `ProposalEditorToolbar.tsx` exposes optional `onScoreClick?: () => void` prop
- [x] `ProposalWorkspacePage.tsx` defines `handleScoreClick` handler
- [x] `handleScoreClick` calls `setRightActiveTab("scoring")`
- [x] `handleScoreClick` calls `setRightCollapsed(false)`
- [x] `onScoreClick={handleScoreClick}` passed to toolbar component
- [x] Workspace renders toolbar component (local `<ProposalToolbar>` or imported `<ProposalEditorToolbar>`) with the wiring

### AC 21 — i18n Keys (22 flat keys, EN + BG parity)

All keys present in both `messages/en.json` and `messages/bg.json` under the flat `proposals` namespace:

| Key | EN | BG |
|---|---|---|
| `scoringTitle` | ✅ | ✅ |
| `scoringBtnSimulate` | ✅ | ✅ |
| `scoringBtnResimulate` | ✅ | ✅ |
| `scoringColCriterion` | ✅ | ✅ |
| `scoringColScore` | ✅ | ✅ |
| `scoringColMax` | ✅ | ✅ |
| `scoringColSuggestion` | ✅ | ✅ |
| `scoringError` | ✅ | ✅ |
| `scoringEmpty` | ✅ | ✅ |
| `pricingTitle` | ✅ | ✅ |
| `pricingBtnAdvise` | ✅ | ✅ |
| `pricingBtnReadvise` | ✅ | ✅ |
| `pricingLabelRecommended` | ✅ | ✅ |
| `pricingLabelMarketRange` | ✅ | ✅ |
| `pricingError` | ✅ | ✅ |
| `pricingEmpty` | ✅ | ✅ |
| `winThemesTitle` | ✅ | ✅ |
| `winThemesBtnExtract` | ✅ | ✅ |
| `winThemesBtnReextract` | ✅ | ✅ |
| `winThemesDragHint` | ✅ | ✅ |
| `winThemesError` | ✅ | ✅ |
| `winThemesEmpty` | ✅ | ✅ |

- [x] EN and BG key counts equal (full parity)
- [x] All 22 new keys are flat strings (not nested objects)

### AC 22 — E2E Spec Placeholders

- [x] `e2e/specs/proposals/proposals-workspace.spec.ts` contains `S07.15` comment block
- [x] `test.skip(` stub for E07-P2-010 (Scoring Simulator — radar chart + criteria table)
- [x] `test.skip(` stub for E07-P2-012 (Win Themes — draggable cards)

---

## Test Issues Fixed During ATDD Generation

The pre-existing test file had these bugs that were corrected:

| # | Bug | Fix Applied |
|---|---|---|
| 1 | Paths used bare `resolve("app/...")` — fragile when vitest CWD differs | Changed to `resolve(__dirname, "..", ...)` anchored to test file location |
| 2 | Amber regex used unescaped `.` chars: `/criterion.score/` | Fixed to `/\w+\.score\s*\/\s*\w+\.max_score\s*<\s*0\.7/` |
| 3 | Radar formula regex didn't allow `)` before `* 100` | Added `\)?`: `/\w+\.score\s*\/\s*\w+\.max_score\s*\)?\s*\*\s*100/` |
| 4 | `data-amber={isAmber}` hardcoded variable name | Changed to `data-amber=` (any assignment form accepted) |
| 5 | `<ProposalEditorToolbar` in workspace — actual component is local `<ProposalToolbar` | Accepts either name with `||` check |
| 6 | Null-result pattern used single quotes `'result' in data` | Fixed to double quotes matching source: `"result" in data` |
| 7 | Flat-strings check caught pre-existing `status` object key (unrelated to S07.15) | Scoped check to the 22 new S07.15 keys only |
| 8 | Missing explicit `import { describe, it, expect, beforeAll } from "vitest"` | Added (consistent with S07.14 pattern) |
| 9 | Missing `MarketRange` interface check | Added to proposals.ts describe block |
| 10 | Missing `"use client"` checks on new panels | Added to each panel describe block |

---

## Critical Implementation Constraints Verified by Tests

| Constraint (from story Dev Notes) | Test Covering It |
|---|---|
| `ScoreCardCriterion` has NO `id` field | `ScoreCardCriterion has NO id field` |
| GET endpoints return 200 + `{"result":null}` — NOT 404 | `null-result detection uses "result" in data pattern` |
| `recharts` already installed — do not `pnpm add` | Confirmed by `RadarChart` import test passing |
| `"en-EU"` is not a valid BCP 47 locale — use `"de-DE"` | `NOT using "en-EU"` assertion in PricingPanel tests |
| i18n keys must be flat strings — no nesting | `all new S07.15 i18n keys are flat strings` |
| Drag index in `useRef`, not `useState` | `stores drag index in useRef (not useState)` |
| Dynamic import uses named re-export mapping | `contains dynamic imports for all three panels` |

---

## Next Steps

All 86 ATDD tests pass. Story 7.15 implementation is confirmed complete.

For the dev agent, to do a final quality gate:

```bash
# From eusolicit-app/frontend/apps/client/
node_modules/.bin/vitest run __tests__/scoring-pricing-win-themes-s7-15.test.ts

# i18n parity check
pnpm check:i18n

# Full quality gate from eusolicit-app/
make quality-check
```
