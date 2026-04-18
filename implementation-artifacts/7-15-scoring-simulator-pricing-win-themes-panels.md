# Story 7.15: Scoring Simulator, Pricing & Win Themes Panels

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager**,
I want **Scoring Simulator, Pricing Assistant, and Win Themes panels** in the right sidebar of the proposal workspace,
so that I can simulate how evaluators will score my proposal, get a competitive pricing recommendation, and identify the strongest win themes — all without leaving the editor.

## Acceptance Criteria

### Scoring Simulator Panel (Right Sidebar — "Scoring" tab)

1. **ScoringSimulatorPanel replaces placeholder** — `ProposalWorkspacePage.tsx` no longer renders `<div data-testid="scoring-panel-placeholder">Scoring — S07.15</div>`. Instead it renders `<ScoringSimulatorPanel proposalId={proposal.id} />` (dynamically imported, `ssr: false`, loading fallback is an animated skeleton div). The component file `ScoringSimulatorPanel.tsx` exists in `proposals/[id]/components/` with a `"use client"` directive.

2. **Trigger button** — A "Simulate Score" button (`data-testid="btn-simulate-score"`) calls `POST /api/v1/proposals/:id/scoring-simulation`. On mount, load any previously stored result via `GET /api/v1/proposals/:id/scoring-simulation` and display it without re-running.

3. **Radar chart** — On success, display a Recharts `<RadarChart>` (`data-testid="scoring-radar-chart"`) inside a `<ResponsiveContainer width="100%" height={220}>`. Each criterion is a spoke; the polygon shows `(score / max_score) * 100` as the value on a `[0, 100]` domain. Amber highlighting for criteria below 70% is shown in the **table**, not the radar chart (Recharts does not support per-spoke conditional fills without a custom shape).

4. **Criteria table** — Below the radar chart, render a table (`data-testid="scoring-criteria-table"`) with columns: Criterion, Score, Max, Suggestion. Each row has `data-testid="scoring-row-{i}"` (zero-based index). Rows where `criterion.score / criterion.max_score < 0.70` have `data-amber="true"` and amber text (`text-amber-600`) on the score cell.

5. **Re-run button** — When results are displayed, show a "Re-simulate" button (`data-testid="btn-resimulate-score"`).

6. **Loading / error states** — Show a `data-testid="scoring-loading"` skeleton div while running; a `data-testid="scoring-error"` error message with `data-testid="btn-scoring-retry"` on failure.

7. **Toolbar wiring** — `ProposalEditorToolbar` gains an `onScoreClick?: () => void` prop. `ProposalWorkspacePage` wires it: `handleScoreClick = () => { setRightCollapsed(false); setRightActiveTab("scoring"); }`. This follows the same pattern as `onComplianceClick` → `handleComplianceClick`.

### Pricing Assistant Panel (Right Sidebar — "Pricing" tab)

8. **PricingPanel replaces placeholder** — `ProposalWorkspacePage.tsx` no longer renders `<div data-testid="pricing-panel-placeholder">Pricing — S07.15</div>`. Instead it renders `<PricingPanel proposalId={proposal.id} />` (dynamically imported, `ssr: false`).

9. **Trigger button** — A "Get Pricing Advice" button (`data-testid="btn-get-pricing"`) calls `POST /api/v1/proposals/:id/pricing-assist`. On mount, load any previously stored result and display it.

10. **Recommended price display** — Show `result.recommended_price` as large primary text (`data-testid="pricing-recommended-price"`), formatted with `new Intl.NumberFormat("de-DE", { style: "currency", currency: "EUR" }).format(recommended_price)`. (`"de-DE"` with EUR is the correct locale combination for EU-style number formatting; `"en-EU"` is not a valid BCP 47 locale.)

11. **Market range bar** — Render a horizontal bar (`data-testid="pricing-market-range"`) showing min/median/max labeled markers. If `recommended_price` is between min and max, render a colored marker at the proportional position. Use `div`-based proportional widths — no third-party chart library. `toPercent = (val) => Math.max(0, Math.min(100, ((val - min) / (max - min)) * 100))`.

12. **Justification text** — Render `result.justification` as `data-testid="pricing-justification"`.

13. **Re-run button** — `data-testid="btn-readvise-pricing"` shown after results.

14. **Loading / error states** — `data-testid="pricing-loading"` skeleton while loading; `data-testid="pricing-error"` with `data-testid="btn-pricing-retry"` on failure.

### Win Themes Panel (Right Sidebar — "Win Themes" tab)

15. **WinThemesPanel replaces placeholder** — `ProposalWorkspacePage.tsx` no longer renders `<div data-testid="win-themes-panel-placeholder">Win Themes — S07.15</div>`. Instead renders `<WinThemesPanel proposalId={proposal.id} />` (dynamically imported, `ssr: false`).

16. **Trigger button** — A "Extract Win Themes" button (`data-testid="btn-extract-win-themes"`) calls `POST /api/v1/proposals/:id/win-themes`. On mount, load any previously stored result.

17. **Ranked theme cards** — Results displayed as a vertically ordered list (`data-testid="win-themes-list"`). Each card: `data-testid="win-theme-card-{index}"` (zero-based), bold title, description text, and a `GripVertical` drag handle `data-testid="win-theme-drag-handle-{index}"`.

18. **Drag-to-reorder (HTML5 only)** — Use the HTML5 Drag and Drop API (`draggable`, `onDragStart`, `onDragOver`, `onDrop`) — **no third-party DnD library**. Store dragged index in `useRef<number | null>` (not state — prevents re-render storm during drag). Reordered list is display-only; no API call persists the new order.

19. **Re-run button** — `data-testid="btn-reextract-win-themes"` shown after results.

20. **Loading / error states** — `data-testid="win-themes-loading"` skeleton; `data-testid="win-themes-error"` with `data-testid="btn-win-themes-retry"` on failure.

### Shared

21. **i18n** — All strings in all three panels use `useTranslations("proposals")`. Keys use flat format (not nested objects) consistent with the existing `proposals` namespace. Required keys are added to both `messages/en.json` and `messages/bg.json`. Run `pnpm check:i18n` to verify parity.

22. **ATDD tests** — A test file `__tests__/scoring-pricing-win-themes-s7-15.test.ts` covers: all three component files exist, all `data-testid` attributes present in source, Recharts `RadarChart` import in scoring panel, `Intl.NumberFormat` usage in pricing panel, `GripVertical` and drag handle testids in win themes panel, amber threshold logic (`0.7` or `0.70`) present in source, API types and functions exported, i18n key completeness. All tests GREEN.

## Tasks / Subtasks

- [x] Task 1: Add scoring/pricing/win-themes API types and functions to `lib/api/proposals.ts` (AC: 2, 9, 16)
  - [x] 1.1 Add `ScoreCardCriterion` interface — fields: `criterion: string; score: number; max_score: number; suggestion: string` (NO `id` field — backend has none)
  - [x] 1.2 Add `ScoringSimulationResponse` interface — fields: `criteria: ScoreCardCriterion[]; simulated_at: string`
  - [x] 1.3 Add `MarketRange` interface — fields: `min: number; median: number; max: number`
  - [x] 1.4 Add `PricingAssistResponse` interface — fields: `recommended_price: number; market_range: MarketRange; justification: string; priced_at: string`
  - [x] 1.5 Add `WinTheme` interface — fields: `title: string; description: string`
  - [x] 1.6 Add `WinThemesResponse` interface — fields: `themes: WinTheme[]; analyzed_at: string`
  - [x] 1.7 Add `runScoringSimulation(proposalId: string): Promise<ScoringSimulationResponse>` — `POST /api/v1/proposals/${proposalId}/scoring-simulation`; `return (await apiClient.post<ScoringSimulationResponse>(..., {})).data`
  - [x] 1.8 Add `getScoringResults(proposalId: string): Promise<ScoringSimulationResponse | null>` — `GET /api/v1/proposals/${proposalId}/scoring-simulation`; detect null via `'result' in data && data.result === null`
  - [x] 1.9 Add `runPricingAssist(proposalId: string): Promise<PricingAssistResponse>` — `POST /api/v1/proposals/${proposalId}/pricing-assist`; return `.data`
  - [x] 1.10 Add `getPricingResults(proposalId: string): Promise<PricingAssistResponse | null>` — GET; null-result detection same as scoring
  - [x] 1.11 Add `runWinThemes(proposalId: string): Promise<WinThemesResponse>` — `POST /api/v1/proposals/${proposalId}/win-themes`; return `.data`
  - [x] 1.12 Add `getWinThemes(proposalId: string): Promise<WinThemesResponse | null>` — GET; null-result detection same pattern

- [x] Task 2: Add React Query hooks to `lib/queries/use-proposals.ts` (AC: 2, 9, 16)
  - [x] 2.1 Add `useScoringResults(proposalId: string)` — `useQuery({ queryKey: ["scoring", proposalId], queryFn: () => getScoringResults(proposalId), staleTime: 60_000 })`
  - [x] 2.2 Add `useRunScoringSimulation(proposalId: string)` — `useMutation` calling `runScoringSimulation(proposalId)`; `onSuccess`: `queryClient.invalidateQueries({ queryKey: ["scoring", proposalId] })`
  - [x] 2.3 Add `usePricingResults(proposalId: string)` — `useQuery({ queryKey: ["pricing", proposalId], queryFn: () => getPricingResults(proposalId), staleTime: 60_000 })`
  - [x] 2.4 Add `useRunPricingAssist(proposalId: string)` — `useMutation`; `onSuccess`: invalidate `["pricing", proposalId]`
  - [x] 2.5 Add `useWinThemesResults(proposalId: string)` — `useQuery({ queryKey: ["win-themes", proposalId], queryFn: () => getWinThemes(proposalId), staleTime: 60_000 })`
  - [x] 2.6 Add `useRunWinThemes(proposalId: string)` — `useMutation`; `onSuccess`: invalidate `["win-themes", proposalId]`

- [x] Task 3: Create `ScoringSimulatorPanel.tsx` (AC: 1–7)
  - [x] 3.1 Create `…/proposals/[id]/components/ScoringSimulatorPanel.tsx` with `"use client"` directive
  - [x] 3.2 Props: `interface ScoringSimulatorPanelProps { proposalId: string; }`
  - [x] 3.3 Fetch stored results via `useScoringResults(proposalId)`; run mutation via `useRunScoringSimulation(proposalId)`
  - [x] 3.4 Render `data-testid="btn-simulate-score"` trigger button (disabled while mutation pending)
  - [x] 3.5 When `resultsQuery.isLoading`: render `<div data-testid="scoring-loading" className="animate-pulse ...">` skeleton
  - [x] 3.6 When mutation/query error: render `<div data-testid="scoring-error">` + `<Button data-testid="btn-scoring-retry">`
  - [x] 3.7 When results available: render `<ScoringSimulatorPanel>` content:
    - Recharts `<ResponsiveContainer width="100%" height={220}><RadarChart data={data}>` with `data-testid="scoring-radar-chart"` on the outer div
    - `data` computed as `criteria.map(c => ({ criterion: c.criterion, value: (c.score / c.max_score) * 100 }))`
    - `<PolarGrid />`, `<PolarAngleAxis dataKey="criterion" tick={{ fontSize: 11 }} />`, `<PolarRadiusAxis domain={[0, 100]} tick={false} />`, `<Radar name="Score" dataKey="value" stroke="#4f46e5" fill="#4f46e5" fillOpacity={0.4} />`
  - [x] 3.8 Criteria table `data-testid="scoring-criteria-table"` with rows `data-testid="scoring-row-{i}"` (zero-based index); amber row: `criterion.score / criterion.max_score < 0.70` → `data-amber="true"` + `className="text-amber-600"` on score cell
  - [x] 3.9 Re-run button `data-testid="btn-resimulate-score"` shown when results exist
  - [x] 3.10 All text via `useTranslations("proposals")`; `addToast` from `@/lib/stores/ui-store` for errors

- [x] Task 4: Create `PricingPanel.tsx` (AC: 8–14)
  - [x] 4.1 Create `…/proposals/[id]/components/PricingPanel.tsx` with `"use client"` directive
  - [x] 4.2 Props: `interface PricingPanelProps { proposalId: string; }`
  - [x] 4.3 Fetch via `usePricingResults(proposalId)`; mutate via `useRunPricingAssist(proposalId)`
  - [x] 4.4 Trigger button `data-testid="btn-get-pricing"` (disabled while pending); loading `data-testid="pricing-loading"`; error `data-testid="pricing-error"` + `data-testid="btn-pricing-retry"`
  - [x] 4.5 Recommended price: `<p data-testid="pricing-recommended-price">` with `new Intl.NumberFormat("de-DE", { style: "currency", currency: "EUR" }).format(result.recommended_price)` — use `"de-DE"` locale (not `"en-EU"` which is invalid)
  - [x] 4.6 Market range bar: `<div data-testid="pricing-market-range" className="relative h-3 rounded-full bg-slate-200 my-4">` with span labels for min/max, div markers for median and recommended_price; use `toPercent = (val) => Math.max(0, Math.min(100, ((val - min) / (max - min)) * 100))`; only render recommended marker when `min <= recommended_price <= max`
  - [x] 4.7 Justification: `<p data-testid="pricing-justification">{result.justification}</p>`
  - [x] 4.8 Re-advise button `data-testid="btn-readvise-pricing"` after results
  - [x] 4.9 All text via `useTranslations("proposals")`

- [x] Task 5: Create `WinThemesPanel.tsx` (AC: 15–20)
  - [x] 5.1 Create `…/proposals/[id]/components/WinThemesPanel.tsx` with `"use client"` directive
  - [x] 5.2 Props: `interface WinThemesPanelProps { proposalId: string; }`
  - [x] 5.3 Fetch via `useWinThemesResults(proposalId)`; mutate via `useRunWinThemes(proposalId)`. After successful run or query: initialize `useState<WinTheme[]>` from `result.themes`
  - [x] 5.4 Trigger button `data-testid="btn-extract-win-themes"`; loading `data-testid="win-themes-loading"`; error `data-testid="win-themes-error"` + `data-testid="btn-win-themes-retry"`
  - [x] 5.5 Theme list `data-testid="win-themes-list"`; each card `data-testid="win-theme-card-{index}"`
  - [x] 5.6 Drag-and-drop (HTML5 only — no library):
    ```typescript
    const dragIndexRef = useRef<number | null>(null);
    const handleDragStart = (index: number) => { dragIndexRef.current = index; };
    const handleDrop = (dropIndex: number) => {
      const dragIndex = dragIndexRef.current;
      if (dragIndex === null || dragIndex === dropIndex) return;
      setThemes(prev => {
        const next = [...prev];
        const [moved] = next.splice(dragIndex, 1);
        next.splice(dropIndex, 0, moved);
        return next;
      });
      dragIndexRef.current = null;
    };
    // On each card div: draggable onDragStart={() => handleDragStart(i)} onDragOver={e => e.preventDefault()} onDrop={() => handleDrop(i)}
    ```
  - [x] 5.7 Drag handle: `<GripVertical data-testid="win-theme-drag-handle-{index}" />` from `lucide-react`
  - [x] 5.8 Re-extract button `data-testid="btn-reextract-win-themes"`; all text via `useTranslations("proposals")`

- [x] Task 6: Update `ProposalEditorToolbar` + `ProposalWorkspacePage` (AC: 1, 7, 8, 15)
  - [x] 6.1 In `ProposalEditorToolbar` component, add prop `onScoreClick?: () => void` to interface; add `onClick={onScoreClick}` to `<Button data-testid="toolbar-btn-score">`
  - [x] 6.2 In `ProposalWorkspacePage`, add handler:
    ```typescript
    const handleScoreClick = () => {
      setRightCollapsed(false);
      setRightActiveTab("scoring");
    };
    ```
  - [x] 6.3 Pass `onScoreClick={handleScoreClick}` to `<ProposalEditorToolbar>`
  - [x] 6.4 Add dynamic imports for all three panels (pattern from S07.14):
    ```typescript
    const ScoringSimulatorPanel = dynamic(
      () => import("./ScoringSimulatorPanel").then((m) => ({ default: m.ScoringSimulatorPanel })),
      { ssr: false, loading: () => <div className="h-8 w-full animate-pulse rounded bg-muted" /> }
    );
    const PricingPanel = dynamic(
      () => import("./PricingPanel").then((m) => ({ default: m.PricingPanel })),
      { ssr: false, loading: () => <div className="h-8 w-full animate-pulse rounded bg-muted" /> }
    );
    const WinThemesPanel = dynamic(
      () => import("./WinThemesPanel").then((m) => ({ default: m.WinThemesPanel })),
      { ssr: false, loading: () => <div className="h-8 w-full animate-pulse rounded bg-muted" /> }
    );
    ```
  - [x] 6.5 Replace `<div data-testid="scoring-panel-placeholder">Scoring — S07.15</div>` with `<ScoringSimulatorPanel proposalId={proposal.id} />`
  - [x] 6.6 Replace `<div data-testid="pricing-panel-placeholder">Pricing — S07.15</div>` with `<PricingPanel proposalId={proposal.id} />`
  - [x] 6.7 Replace `<div data-testid="win-themes-panel-placeholder">Win Themes — S07.15</div>` with `<WinThemesPanel proposalId={proposal.id} />`

- [x] Task 7: Add i18n keys (AC: 21)
  - [x] 7.1 Add all flat keys (see Dev Notes) to `messages/en.json` under `proposals` namespace (alongside existing keys — no nested objects)
  - [x] 7.2 Add matching Bulgarian translations to `messages/bg.json`
  - [x] 7.3 Run `pnpm check:i18n` to verify key parity

- [x] Task 8: Write ATDD tests (AC: 22)
  - [x] 8.1 Create `eusolicit-app/frontend/apps/client/__tests__/scoring-pricing-win-themes-s7-15.test.ts` using `fs.readFile` + `.toContain()` / `.not.toContain()` pattern from `ai-draft-generation-s7-13.test.ts` and `requirement-checklist-compliance-s7-14.test.ts`
  - [x] 8.2 Assert all three component files exist (paths from Dev Notes file structure)
  - [x] 8.3 Assert all `data-testid` values present in source (scoring: `btn-simulate-score`, `scoring-radar-chart`, `scoring-criteria-table`, `scoring-row-`, `btn-resimulate-score`, `scoring-loading`, `scoring-error`, `btn-scoring-retry`; pricing: `btn-get-pricing`, `pricing-recommended-price`, `pricing-market-range`, `pricing-justification`, `btn-readvise-pricing`, `pricing-loading`, `pricing-error`, `btn-pricing-retry`; win themes: `btn-extract-win-themes`, `win-themes-list`, `win-theme-card-`, `win-theme-drag-handle-`, `btn-reextract-win-themes`, `win-themes-loading`, `win-themes-error`, `btn-win-themes-retry`)
  - [x] 8.4 Assert `RadarChart` imported from `recharts` in `ScoringSimulatorPanel.tsx`
  - [x] 8.5 Assert `Intl.NumberFormat` + `"de-DE"` present in `PricingPanel.tsx`
  - [x] 8.6 Assert `GripVertical` imported from `lucide-react` in `WinThemesPanel.tsx`
  - [x] 8.7 Assert amber threshold: `ScoringSimulatorPanel.tsx` contains `0.7` or `0.70`
  - [x] 8.8 Assert `ScoreCardCriterion`, `ScoringSimulationResponse`, `PricingAssistResponse`, `WinTheme`, `WinThemesResponse` exported from `proposals.ts`
  - [x] 8.9 Assert `useScoringResults`, `useRunScoringSimulation`, `usePricingResults`, `useRunPricingAssist`, `useWinThemesResults`, `useRunWinThemes` exported from `use-proposals.ts`
  - [x] 8.10 Assert i18n key completeness (see Dev Notes for required keys)
  - [x] 8.11 Assert `ProposalWorkspacePage.tsx` does NOT contain `"Scoring — S07.15"`, `"Pricing — S07.15"`, `"Win Themes — S07.15"` (placeholders removed)
  - [x] 8.12 Add two skipped E2E placeholder tests to `eusolicit-app/e2e/specs/proposals/proposals-workspace.spec.ts`

## Dev Notes

### Architecture: Pure Frontend Story

S07.15 is a **pure frontend story**. All backend endpoints are complete (S07.07 and S07.08):

| Endpoint | Method | Story | Response Shape |
|---|---|---|---|
| `/api/v1/proposals/:id/scoring-simulation` | POST | S07.07 | `ScoringSimulationResponse` |
| `/api/v1/proposals/:id/scoring-simulation` | GET | S07.07 | `ScoringSimulationResponse \| NullResultResponse` |
| `/api/v1/proposals/:id/pricing-assist` | POST | S07.08 | `PricingAssistResponse` |
| `/api/v1/proposals/:id/pricing-assist` | GET | S07.08 | `PricingAssistResponse \| NullResultResponse` |
| `/api/v1/proposals/:id/win-themes` | POST | S07.08 | `WinThemesResponse` |
| `/api/v1/proposals/:id/win-themes` | GET | S07.08 | `WinThemesResponse \| NullResultResponse` |

Source router: `eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py` (lines 834–1049).

### Exact Backend Response Shapes (Verified from Python schemas)

**⚠️ CRITICAL:** `ScoreCardCriterion` has **NO `id` field** — only `criterion`, `score`, `max_score`, `suggestion`. The draft story was wrong on this point. Use `criterion` (string) as the display key, and the zero-based array index for `data-testid` row identifiers.

```python
# eusolicit-app/services/client-api/src/client_api/schemas/proposal_agent_results.py

class ScoreCardCriterion(BaseModel):
    criterion: str       # ← display label
    score: float         # ← NOT int; use float in TypeScript
    max_score: float     # ← NOT int; use float
    suggestion: str

class ScoringSimulationResponse(BaseModel):
    criteria: list[ScoreCardCriterion]
    simulated_at: datetime   # ← present in response; map to string in TS
```

```python
# eusolicit-app/services/client-api/src/client_api/schemas/proposal_pricing_win_themes.py

class MarketRange(BaseModel):
    min: float; median: float; max: float

class PricingAssistResponse(BaseModel):
    recommended_price: float
    market_range: MarketRange
    justification: str
    priced_at: datetime   # ← map to string in TS

class WinTheme(BaseModel):
    title: str; description: str

class WinThemesResponse(BaseModel):
    themes: list[WinTheme]   # ← rank-ordered; first = highest priority
    analyzed_at: datetime    # ← present in response
```

```python
# NullResultResponse (same file as ScoreCardCriterion)
class NullResultResponse(BaseModel):
    result: None = None   # GET returns HTTP 200 with {"result": null} when no run yet
```

### TypeScript Interfaces for `proposals.ts`

```typescript
// SCORING
export interface ScoreCardCriterion {
  criterion: string;    // NO id field — use array index for testids
  score: number;        // float
  max_score: number;    // float
  suggestion: string;
}
export interface ScoringSimulationResponse {
  criteria: ScoreCardCriterion[];
  simulated_at: string;
}

// PRICING
export interface MarketRange {
  min: number;
  median: number;
  max: number;
}
export interface PricingAssistResponse {
  recommended_price: number;
  market_range: MarketRange;
  justification: string;
  priced_at: string;
}

// WIN THEMES
export interface WinTheme {
  title: string;
  description: string;
}
export interface WinThemesResponse {
  themes: WinTheme[];    // rank-ordered; first = highest priority
  analyzed_at: string;
}
```

### API Function Implementations (exact pattern from proposals.ts)

`apiClient.post<T>()` and `.get<T>()` return `{ data: T, ... }` — always use `.data` accessor:

```typescript
export async function runScoringSimulation(proposalId: string): Promise<ScoringSimulationResponse> {
  return (await apiClient.post<ScoringSimulationResponse>(
    `/api/v1/proposals/${proposalId}/scoring-simulation`, {}
  )).data;
}

export async function getScoringResults(proposalId: string): Promise<ScoringSimulationResponse | null> {
  const data = (await apiClient.get<ScoringSimulationResponse | { result: null }>(
    `/api/v1/proposals/${proposalId}/scoring-simulation`
  )).data;
  if ('result' in data && data.result === null) return null;
  return data as ScoringSimulationResponse;
}

export async function runPricingAssist(proposalId: string): Promise<PricingAssistResponse> {
  return (await apiClient.post<PricingAssistResponse>(
    `/api/v1/proposals/${proposalId}/pricing-assist`, {}
  )).data;
}

export async function getPricingResults(proposalId: string): Promise<PricingAssistResponse | null> {
  const data = (await apiClient.get<PricingAssistResponse | { result: null }>(
    `/api/v1/proposals/${proposalId}/pricing-assist`
  )).data;
  if ('result' in data && data.result === null) return null;
  return data as PricingAssistResponse;
}

export async function runWinThemes(proposalId: string): Promise<WinThemesResponse> {
  return (await apiClient.post<WinThemesResponse>(
    `/api/v1/proposals/${proposalId}/win-themes`, {}
  )).data;
}

export async function getWinThemes(proposalId: string): Promise<WinThemesResponse | null> {
  const data = (await apiClient.get<WinThemesResponse | { result: null }>(
    `/api/v1/proposals/${proposalId}/win-themes`
  )).data;
  if ('result' in data && data.result === null) return null;
  return data as WinThemesResponse;
}
```

### Recharts RadarChart Setup (recharts ^2.12.0 already installed)

**⚠️ recharts is ALREADY in `apps/client/package.json` at `^2.12.0`. Do NOT run `pnpm add recharts`.**

```tsx
import {
  RadarChart, PolarGrid, PolarAngleAxis, PolarRadiusAxis,
  Radar, ResponsiveContainer
} from "recharts";

// data derived from criteria
const data = criteria.map((c) => ({
  criterion: c.criterion,
  value: (c.score / c.max_score) * 100,
}));

// Render
<div data-testid="scoring-radar-chart">
  <ResponsiveContainer width="100%" height={220}>
    <RadarChart data={data}>
      <PolarGrid />
      <PolarAngleAxis dataKey="criterion" tick={{ fontSize: 11 }} />
      <PolarRadiusAxis domain={[0, 100]} tick={false} />
      <Radar name="Score" dataKey="value" stroke="#4f46e5" fill="#4f46e5" fillOpacity={0.4} />
    </RadarChart>
  </ResponsiveContainer>
</div>
```

**Amber highlighting is in the TABLE, not the chart** — Recharts RadarChart does not support per-spoke conditional fills without complex custom shapes.

### Market Range Bar (No Chart Library)

```tsx
const { min, median, max } = result.market_range;
const toPercent = (val: number) =>
  Math.max(0, Math.min(100, ((val - min) / (max - min)) * 100));
const formatCurrency = (val: number) =>
  new Intl.NumberFormat("de-DE", { style: "currency", currency: "EUR" }).format(val);

<div data-testid="pricing-market-range" className="relative h-3 rounded-full bg-slate-200 my-4">
  <span className="absolute left-0 -bottom-5 text-xs text-muted-foreground">
    {formatCurrency(min)}
  </span>
  <div
    className="absolute top-0 h-3 w-0.5 bg-slate-500"
    style={{ left: `${toPercent(median)}%` }}
    title={`Median: ${formatCurrency(median)}`}
  />
  <span className="absolute right-0 -bottom-5 text-xs text-muted-foreground">
    {formatCurrency(max)}
  </span>
  {result.recommended_price >= min && result.recommended_price <= max && (
    <div
      className="absolute top-0 h-3 w-1 rounded bg-indigo-600"
      style={{ left: `${toPercent(result.recommended_price)}%` }}
      title={`Recommended: ${formatCurrency(result.recommended_price)}`}
    />
  )}
</div>
```

### Drag-and-Drop (HTML5 API, No DnD Library)

```tsx
const [themes, setThemes] = useState<WinTheme[]>([]);
const dragIndexRef = useRef<number | null>(null);

// Sync themes from query result when data arrives
useEffect(() => {
  if (winThemesQuery.data?.themes) {
    setThemes(winThemesQuery.data.themes);
  }
}, [winThemesQuery.data]);

const handleDragStart = (index: number) => {
  dragIndexRef.current = index;
};

const handleDrop = (dropIndex: number) => {
  const dragIndex = dragIndexRef.current;
  if (dragIndex === null || dragIndex === dropIndex) return;
  setThemes((prev) => {
    const next = [...prev];
    const [moved] = next.splice(dragIndex, 1);
    next.splice(dropIndex, 0, moved);
    return next;
  });
  dragIndexRef.current = null;
};

// Each theme card:
<div
  key={index}
  data-testid={`win-theme-card-${index}`}
  draggable
  onDragStart={() => handleDragStart(index)}
  onDragOver={(e) => e.preventDefault()}
  onDrop={() => handleDrop(index)}
  className="..."
>
  <GripVertical
    data-testid={`win-theme-drag-handle-${index}`}
    className="h-4 w-4 text-muted-foreground cursor-grab"
  />
  <div>
    <p className="font-bold">{theme.title}</p>
    <p className="text-sm">{theme.description}</p>
  </div>
</div>
```

**Key rule:** Store drag index in `useRef`, not `useState` — prevents re-render storm on every `onDragOver` event.

### Toolbar Wiring — Extend ProposalEditorToolbar Props

`ProposalEditorToolbar` currently has `onGenerateClick` and `onComplianceClick` props. This story adds `onScoreClick`:

```typescript
// In ProposalEditorToolbar component interface (currently ~line 148):
interface ProposalEditorToolbarProps {
  // ... existing props ...
  onComplianceClick?: () => void;  // existing
  onScoreClick?: () => void;       // ADD THIS
}

// In ProposalWorkspacePage (same pattern as compliance, ~line 372):
const handleScoreClick = () => {
  setRightCollapsed(false);
  setRightActiveTab("scoring");
};
// Pass to toolbar:
<ProposalEditorToolbar
  {...otherProps}
  onComplianceClick={handleComplianceClick}
  onScoreClick={handleScoreClick}   // ADD THIS
/>
```

### Dynamic Import Pattern (Same as S07.14)

```typescript
// In ProposalWorkspacePage.tsx — after existing dynamic imports for AiGeneratePanel, CompliancePanel:
const ScoringSimulatorPanel = dynamic(
  () => import("./ScoringSimulatorPanel").then((m) => ({ default: m.ScoringSimulatorPanel })),
  { ssr: false, loading: () => <div className="h-8 w-full animate-pulse rounded bg-muted" /> }
);
const PricingPanel = dynamic(
  () => import("./PricingPanel").then((m) => ({ default: m.PricingPanel })),
  { ssr: false, loading: () => <div className="h-8 w-full animate-pulse rounded bg-muted" /> }
);
const WinThemesPanel = dynamic(
  () => import("./WinThemesPanel").then((m) => ({ default: m.WinThemesPanel })),
  { ssr: false, loading: () => <div className="h-8 w-full animate-pulse rounded bg-muted" /> }
);
```

### i18n Keys Required (flat format under `proposals` namespace)

The existing `proposals` namespace uses flat keys (not nested objects). Add the following flat keys alongside existing ones:

**English (`messages/en.json` additions under `"proposals"`):**
```json
"scoringTitle": "Scoring Simulator",
"scoringBtnSimulate": "Simulate Score",
"scoringBtnResimulate": "Re-simulate",
"scoringColCriterion": "Criterion",
"scoringColScore": "Score",
"scoringColMax": "Max",
"scoringColSuggestion": "Suggestion",
"scoringError": "Simulation failed. Please retry.",
"scoringEmpty": "No scoring results yet.",
"pricingTitle": "Pricing Assistant",
"pricingBtnAdvise": "Get Pricing Advice",
"pricingBtnReadvise": "Re-advise",
"pricingLabelRecommended": "Recommended Price",
"pricingLabelMarketRange": "Market Range",
"pricingError": "Pricing analysis failed. Please retry.",
"pricingEmpty": "No pricing results yet.",
"winThemesTitle": "Win Themes",
"winThemesBtnExtract": "Extract Win Themes",
"winThemesBtnReextract": "Re-extract",
"winThemesDragHint": "Drag to reorder",
"winThemesError": "Win theme extraction failed. Please retry.",
"winThemesEmpty": "No win themes generated yet."
```

**Bulgarian (`messages/bg.json` additions):**
```json
"scoringTitle": "Симулатор на оценяване",
"scoringBtnSimulate": "Симулирай",
"scoringBtnResimulate": "Повторна симулация",
"scoringColCriterion": "Критерий",
"scoringColScore": "Точки",
"scoringColMax": "Макс.",
"scoringColSuggestion": "Препоръка",
"scoringError": "Симулацията е неуспешна. Опитайте отново.",
"scoringEmpty": "Няма резултати от оценяване.",
"pricingTitle": "Асистент за ценообразуване",
"pricingBtnAdvise": "Ценова препоръка",
"pricingBtnReadvise": "Нова препоръка",
"pricingLabelRecommended": "Препоръчана цена",
"pricingLabelMarketRange": "Пазарен диапазон",
"pricingError": "Анализът на ценообразуването е неуспешен. Опитайте отново.",
"pricingEmpty": "Няма резултати от ценообразуване.",
"winThemesTitle": "Ключови теми",
"winThemesBtnExtract": "Извлечи теми",
"winThemesBtnReextract": "Преизвличане",
"winThemesDragHint": "Преплъзнете за пренареждане",
"winThemesError": "Извличането на теми е неуспешно. Опитайте отново.",
"winThemesEmpty": "Все още няма генерирани теми."
```

**Note:** Do NOT create nested objects like `{ "scoring": { "title": ... } }`. The project uses only flat keys. The ATDD test will verify key parity.

### File Structure

```
eusolicit-app/frontend/apps/client/
├── app/[locale]/(protected)/proposals/[id]/components/
│   ├── ProposalWorkspacePage.tsx              MODIFIED — replace 3 placeholders, add dynamic imports, wire onScoreClick
│   ├── ProposalEditorToolbar.tsx              MODIFIED — add onScoreClick prop + onClick binding
│   ├── ScoringSimulatorPanel.tsx              NEW
│   ├── PricingPanel.tsx                       NEW
│   └── WinThemesPanel.tsx                     NEW
├── lib/
│   ├── api/
│   │   └── proposals.ts                       MODIFIED — add 6 interfaces + 6 functions
│   └── queries/
│       └── use-proposals.ts                   MODIFIED — add 6 hooks
├── messages/
│   ├── en.json                                MODIFIED — add 22 proposals keys
│   └── bg.json                                MODIFIED — matching BG translations
└── __tests__/
    └── scoring-pricing-win-themes-s7-15.test.ts  NEW
```

**Also modified:** `eusolicit-app/e2e/specs/proposals/proposals-workspace.spec.ts` (2 `test.skip` placeholders added)

**Do NOT touch:**
- Any backend services files (all backend done in S07.07 and S07.08)
- `ProposalEditor.tsx`, `ProposalSection.tsx`, `AiGeneratePanel.tsx`, `CompliancePanel.tsx`, `RequirementChecklistPanel.tsx`, `ConflictDialog.tsx`
- `proposal-editor-store.ts` (no new store actions needed)
- `package.json` for recharts (already installed)

### QueryGuard vs Manual State Machine

Per project-context.md rule 21, data-fetching components must use `<QueryGuard>`. For these three panels:

- **`ScoringSimulatorPanel`** and **`PricingPanel`**: Use `<QueryGuard>` wrapping for the stored-results query (loading/error/empty states). The mutation trigger is a separate concern outside the guard.
- **`WinThemesPanel`**: Use `<QueryGuard>` wrapping for the stored-results query. After data arrives, sync into `useState<WinTheme[]>` for drag reordering.

The `emptyElement` for each guard should render the trigger button ("Simulate Score", "Get Pricing Advice", "Extract Win Themes") + an empty-state description.

### Previous Story Patterns (S07.13 and S07.14 — Must Replicate)

1. **`"use client"` at top of every new component file** — before any imports
2. **Zustand `getState()` in event handlers** — these panels don't read from the proposal editor store; they only write if needed
3. **All hooks at top of function body** — all `useQuery`, `useMutation`, `useTranslations`, `useQueryClient`, `useState`, `useRef`, `useEffect` declared before any conditional returns
4. **Selector pattern for Zustand stores**: `useAuthStore((s) => s.token)` not `const { token } = useAuthStore()`
5. **`addToast` from `@/lib/stores/ui-store`** — NOT from `@eusolicit/ui`. Pattern: `const addToast = useUIStore((s) => s.addToast)` where `useUIStore` is imported from `@/lib/stores/ui-store`
6. **`apiClient` from `@eusolicit/ui`** — consistent with existing `proposals.ts`
7. **ATDD file-system + `readFile` + `.toContain()` pattern** — same as `ai-draft-generation-s7-13.test.ts` and `requirement-checklist-compliance-s7-14.test.ts`
8. **Dynamic imports with named re-export**: `import("./X").then(m => ({ default: m.X }))` — `X` is the named export, not a default export

### Test Coverage Map (from `test-design-epic-07.md`)

| Test Design ID | Priority | Scenario | S07.15 Coverage |
|---|---|---|---|
| **E07-P2-010** | P2 | Scoring Simulator panel: radar chart renders; table with per-criterion scores vs maxima; amber highlight below 70% | `ScoringSimulatorPanel.tsx` — AC2–6 |
| **E07-P2-011** | P2 | Pricing Assistant panel: recommended price (large); market range bar with proposal marker; justification paragraph | `PricingPanel.tsx` — AC9–13 |
| **E07-P2-012** | P2 | Win Themes panel: ranked theme cards; drag-to-reorder changes card order | `WinThemesPanel.tsx` — AC16–19 |

**Note:** The draft story incorrectly referenced E07-P1-029 (which is about Tiptap editor behavior). The correct coverage IDs are E07-P2-010, E07-P2-011, E07-P2-012.

### ATDD Test File Coverage Map

```
// In __tests__/scoring-pricing-win-themes-s7-15.test.ts
const cwd = "eusolicit-app/frontend/apps/client";
paths.scoringPanel  = resolve(cwd, "app/[locale]/(protected)/proposals/[id]/components/ScoringSimulatorPanel.tsx")
paths.pricingPanel  = resolve(cwd, "app/[locale]/(protected)/proposals/[id]/components/PricingPanel.tsx")
paths.winThemesPanel= resolve(cwd, "app/[locale]/(protected)/proposals/[id]/components/WinThemesPanel.tsx")
paths.workspacePage = resolve(cwd, "app/[locale]/(protected)/proposals/[id]/components/ProposalWorkspacePage.tsx")
paths.proposalsApi  = resolve(cwd, "lib/api/proposals.ts")
paths.useProposals  = resolve(cwd, "lib/queries/use-proposals.ts")
paths.enJson        = resolve(cwd, "messages/en.json")
paths.bgJson        = resolve(cwd, "messages/bg.json")
```

Key assertions:

**AC1/7: Scoring panel exists; placeholder removed**
- `ScoringSimulatorPanel.tsx` exists
- `ProposalWorkspacePage.tsx` does NOT contain `"Scoring — S07.15"`
- `ProposalWorkspacePage.tsx` contains `import("./ScoringSimulatorPanel")`

**AC2/3/4: Scoring chart and table**
- `ScoringSimulatorPanel.tsx` contains `data-testid="scoring-radar-chart"`
- `ScoringSimulatorPanel.tsx` contains `data-testid="scoring-criteria-table"`
- `ScoringSimulatorPanel.tsx` contains `data-testid="btn-simulate-score"`
- `ScoringSimulatorPanel.tsx` contains `RadarChart` (from recharts import)
- `ScoringSimulatorPanel.tsx` contains `0.7` (amber threshold)
- `ScoringSimulatorPanel.tsx` contains `data-amber`

**AC7: Toolbar wiring**
- `ProposalEditorToolbar.tsx` contains `onScoreClick`
- `ProposalWorkspacePage.tsx` contains `handleScoreClick` or `setRightActiveTab.*scoring`

**AC8/15: Pricing and win-themes panels exist; placeholders removed**
- Panel files exist; workspace does NOT contain their placeholder strings

**AC10: Price formatting**
- `PricingPanel.tsx` contains `Intl.NumberFormat`
- `PricingPanel.tsx` contains `"de-DE"` (correct locale — NOT `"en-EU"`)
- `PricingPanel.tsx` contains `data-testid="pricing-recommended-price"`
- `PricingPanel.tsx` contains `data-testid="pricing-market-range"`

**AC17/18: Win themes drag**
- `WinThemesPanel.tsx` contains `GripVertical`
- `WinThemesPanel.tsx` contains `win-theme-drag-handle-`
- `WinThemesPanel.tsx` contains `draggable`
- `WinThemesPanel.tsx` contains `useRef` (for drag index — NOT useState)

**AC22: API types and hooks**
- `proposals.ts` contains `ScoreCardCriterion`, `ScoringSimulationResponse`, `PricingAssistResponse`, `WinTheme`, `WinThemesResponse`
- `proposals.ts` does NOT contain `id: string` in `ScoreCardCriterion` context (no id field)
- `use-proposals.ts` contains `useScoringResults`, `useRunScoringSimulation`, `usePricingResults`, `useRunPricingAssist`, `useWinThemesResults`, `useRunWinThemes`

**i18n parity**
- `en.json` `proposals` section contains `scoringTitle`, `pricingTitle`, `winThemesTitle`
- `bg.json` contains matching keys
- Key counts equal between files

### Critical Mistakes to Prevent

1. **`ScoreCardCriterion` has NO `id` field** — the backend `ScoreCardCriterion` model has only `criterion`, `score`, `max_score`, `suggestion`. Use the zero-based array index for `data-testid="scoring-row-{i}"` row identifiers. The previous draft story had `id: string` — do NOT include it.

2. **GET endpoints return HTTP 200 + `{"result": null}`, NOT 404** — Detect null result via `'result' in data && data.result === null`. Do NOT add a `.catch(() => null)` pattern.

3. **`recharts` is already installed** — Do NOT run `pnpm add recharts`. It's in `apps/client/package.json` at `^2.12.0`.

4. **`"en-EU"` is not a valid BCP 47 locale** — Use `"de-DE"` with `currency: "EUR"` for European number formatting. `"en-EU"` will throw in some environments.

5. **i18n keys must be flat strings** — The existing `proposals` namespace uses only flat keys (e.g., `"checklistTitle"`, `"complianceAdvisory"`). Do NOT create nested objects like `{ "scoring": { "title": "..." } }`. Use `"scoringTitle"`, `"pricingTitle"`, etc.

6. **Drag index in `useRef`, not `useState`** — Using `useState` for drag index causes a re-render on every `onDragOver` event, creating a re-render storm during drag. Use `useRef<number | null>(null)`.

7. **Do NOT call hooks inside mutation hooks** — `addToast` and `useTranslations` cannot be called inside `use-proposals.ts`. Put error handling in component-level `mutation.isError` checks.

8. **Do NOT use `QueryGuard` for compliance-style manual state machines** — Use `QueryGuard` for the stored-results query path; handle mutation states (loading, error) manually alongside it.

9. **`apiClient` returns `{ data: T, ... }` — always use `.data`** — See existing functions in `proposals.ts` for the pattern: `return (await apiClient.post<T>(...)).data`.

10. **Dynamic import must use named export re-mapping** — `import("./X").then(m => ({ default: m.X }))` — these components use named exports, not default exports.

11. **`ProposalEditorToolbar` is a sub-component** — The `toolbar-btn-score` button is inside `ProposalEditorToolbar`, not directly in `ProposalWorkspacePage`. Add `onScoreClick?: () => void` prop to `ProposalEditorToolbar`'s interface and wire it from `ProposalWorkspacePage` via `handleScoreClick`.

### Backend API Summary

| Endpoint | Method | Auth | Body | Response |
|---|---|---|---|---|
| `/api/v1/proposals/:id/scoring-simulation` | POST | bid_manager | `{}` | `ScoringSimulationResponse` |
| `/api/v1/proposals/:id/scoring-simulation` | GET | any user | — | `ScoringSimulationResponse \| NullResultResponse` |
| `/api/v1/proposals/:id/pricing-assist` | POST | bid_manager | `{}` | `PricingAssistResponse` |
| `/api/v1/proposals/:id/pricing-assist` | GET | any user | — | `PricingAssistResponse \| NullResultResponse` |
| `/api/v1/proposals/:id/win-themes` | POST | bid_manager | `{}` | `WinThemesResponse` |
| `/api/v1/proposals/:id/win-themes` | GET | any user | — | `WinThemesResponse \| NullResultResponse` |

Source files:
- `eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py` (lines 834–1049)
- `eusolicit-app/services/client-api/src/client_api/schemas/proposal_agent_results.py` (ScoreCardCriterion, ScoringSimulationResponse, NullResultResponse)
- `eusolicit-app/services/client-api/src/client_api/schemas/proposal_pricing_win_themes.py` (MarketRange, PricingAssistResponse, WinTheme, WinThemesResponse)

### Project Structure Notes

- No new `packages/ui` components required — all reuse: `QueryGuard`, `EmptyState`, `Button`, `Loader2`, `GripVertical` (lucide-react), `ResponsiveContainer`/`RadarChart` (recharts)
- All from `@eusolicit/ui` except: icons from `lucide-react`, recharts from `recharts`, `addToast` from `@/lib/stores/ui-store`
- No new Zustand stores needed

### References

- [Source: eusolicit-docs/planning-artifacts/epic-07-proposal-generation.md#S07.15]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P2-010, E07-P2-011, E07-P2-012]
- [Source: eusolicit-app/services/client-api/src/client_api/api/v1/proposals.py#834-1049]
- [Source: eusolicit-app/services/client-api/src/client_api/schemas/proposal_agent_results.py]
- [Source: eusolicit-app/services/client-api/src/client_api/schemas/proposal_pricing_win_themes.py]
- [Source: eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalWorkspacePage.tsx]
- [Source: eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalEditorToolbar.tsx]
- [Source: eusolicit-app/frontend/apps/client/lib/api/proposals.ts]
- [Source: eusolicit-app/frontend/apps/client/lib/queries/use-proposals.ts]
- [Source: eusolicit-docs/project-context.md#Frontend Architecture rules 19–31]
- [Source: eusolicit-docs/implementation-artifacts/7-13-ai-draft-generation-panel-with-sse-streaming.md#Dev Notes]
- [Source: eusolicit-docs/implementation-artifacts/7-14-requirement-checklist-compliance-panels.md#Dev Notes]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-6 (story creation 2026-04-18)
claude-sonnet-4-6 (implementation 2026-04-18)

### Debug Log References

All 86 ATDD tests passed on first run — implementation was already complete when dev-story was invoked.

### Completion Notes List

- ✅ Task 1 (API types/functions): All 6 TypeScript interfaces and 6 async functions implemented in `lib/api/proposals.ts`. Null-result detection uses `'result' in data && data.result === null` pattern (HTTP 200 + `{"result":null}`). ScoreCardCriterion has NO `id` field (matches backend schema exactly).
- ✅ Task 2 (React Query hooks): 6 hooks added to `lib/queries/use-proposals.ts` — 3 `useQuery` hooks (staleTime: 60_000) and 3 `useMutation` hooks with `queryClient.invalidateQueries` on success.
- ✅ Task 3 (ScoringSimulatorPanel): Full implementation with Recharts `RadarChart` in `ResponsiveContainer height={220}`, `domain={[0,100]}`, criteria table with amber threshold at `< 0.70`, all required `data-testid` attributes, `useTranslations("proposals")`, `addToast` from `ui-store`.
- ✅ Task 4 (PricingPanel): `Intl.NumberFormat("de-DE", { style: "currency", currency: "EUR" })` for price formatting, `toPercent` helper with `Math.max/min` clamping, div-based market range bar (no chart library), conditional recommended-price marker only when within min/max range.
- ✅ Task 5 (WinThemesPanel): HTML5 Drag and Drop API only (no DnD library), `dragIndexRef` as `useRef<number | null>` (prevents re-render storm), `GripVertical` from lucide-react, `useEffect` to sync themes from query data.
- ✅ Task 6 (Workspace integration): `ProposalEditorToolbar.tsx` has `onScoreClick?: () => void` prop wired to `toolbar-btn-score`. `ProposalWorkspacePage.tsx` has `handleScoreClick` → `setRightCollapsed(false); setRightActiveTab("scoring")`. All three panels dynamically imported with `ssr:false`. Placeholder divs replaced with actual panel components.
- ✅ Task 7 (i18n): 22 flat keys added to both `messages/en.json` and `messages/bg.json` under `proposals` namespace. Full parity confirmed (key counts equal).
- ✅ Task 8 (ATDD): 86 tests all passing in `scoring-pricing-win-themes-s7-15.test.ts`. E2E spec has S07.15 `test.skip` stubs for E07-P2-010 and E07-P2-012.

#### Code Review Follow-ups (2026-04-18)

- ✅ Resolved review finding [High] (Findings 1+2+3): Reverted `ProposalEditorToolbar.tsx` (the formatting toolbar for Bold/Italic/H1/Link) to its pre-S07.15 state — removed the `onScoreClick` prop and the duplicate `<button data-testid="toolbar-btn-score">Score</button>`. Eliminates: (a) the hardcoded English string "Score" (i18n violation), (b) the duplicate `data-testid="toolbar-btn-score"` in the rendered DOM, and (c) the dead onClick on the editor-toolbar Score button (its only consumer `ProposalEditor.tsx` never passed `onScoreClick`). The functional Score button continues to live in the inline `ProposalToolbar` defined inside `ProposalWorkspacePage.tsx` (line 308), which is correctly wired to `handleScoreClick` → `setRightCollapsed(false); setRightActiveTab("scoring")`.
- ✅ Updated ATDD test `AC7: Toolbar wiring — onScoreClick` in `scoring-pricing-win-themes-s7-15.test.ts`: retargeted the `onScoreClick` prop assertion at `ProposalWorkspacePage.tsx` (where the inline `ProposalToolbar` lives) instead of `ProposalEditorToolbar.tsx`, and added a regression-guard test asserting that the formatting toolbar does NOT define `onScoreClick` and does NOT render `data-testid="toolbar-btn-score"`. Test count rose from 86 → 87, all GREEN.
- ✅ Resolved review finding [Low] (Finding 4): Combined the split `lucide-react` import in `WinThemesPanel.tsx` into a single `import { Loader2, GripVertical } from "lucide-react"` statement. Updated the corresponding ATDD assertion to match either form via regex.
- ✅ Resolved review finding [Low] (Finding 5): Replaced `key={index}` on the win-themes draggable list with `key={theme.title}` — a stable identity that survives reorder so React preserves DOM nodes correctly across drag operations (avoids the index-as-key anti-pattern for reorderable lists).
- ✅ Resolved review finding [Low] (Finding 6): `PricingPanel.tsx` market-range bar — replaced `my-4 mb-8` with `mt-4 mb-8` (the redundant bottom margin in `my-4` was being overridden by `mb-8` anyway; explicit `mt-4` is clearer about intent).

### File List

- `eusolicit-app/frontend/apps/client/lib/api/proposals.ts` — MODIFIED (S07.15 section added: 6 interfaces + 6 async functions)
- `eusolicit-app/frontend/apps/client/lib/queries/use-proposals.ts` — MODIFIED (S07.15 section added: 6 React Query hooks)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/ScoringSimulatorPanel.tsx` — NEW
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/PricingPanel.tsx` — NEW
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/WinThemesPanel.tsx` — NEW
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalWorkspacePage.tsx` — MODIFIED (dynamic imports, handleScoreClick, panel replacements)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalEditorToolbar.tsx` — MODIFIED (onScoreClick prop + toolbar-btn-score onClick)
- `eusolicit-app/frontend/apps/client/messages/en.json` — MODIFIED (22 flat i18n keys under proposals namespace)
- `eusolicit-app/frontend/apps/client/messages/bg.json` — MODIFIED (22 matching Bulgarian translations)
- `eusolicit-app/frontend/apps/client/__tests__/scoring-pricing-win-themes-s7-15.test.ts` — NEW + MODIFIED (87 ATDD tests now, all passing — added regression guard for duplicate toolbar-btn-score and retargeted AC7 onScoreClick assertion at ProposalWorkspacePage.tsx)
- `eusolicit-app/e2e/specs/proposals/proposals-workspace.spec.ts` — MODIFIED (S07.15 skip stubs: E07-P2-010, E07-P2-012)

### Change Log

- 2026-04-18 — Implementation complete; 86/86 ATDD tests green. Status → review.
- 2026-04-18 — Senior Developer Review: Changes Requested. 3 must-fix findings (i18n violation, duplicate testid, dead Score button in formatting toolbar) + 3 nice-to-have polish items.
- 2026-04-18 — Addressed code review findings — 6 items resolved (Findings 1+2+3 collapsed into a single revert of `ProposalEditorToolbar.tsx`; Findings 4–6 polish applied). ATDD test count 86 → 87 (all green). Status → review.
- 2026-04-18 — Senior Developer Review (re-review): **Approve**. All prior findings resolved on disk; 87/87 ATDD tests pass; i18n parity (783 keys) verified. Three non-blocking polish notes recorded for future hardening.

## Senior Developer Review

**Reviewer:** Code Review Agent (bmad-code-review, adversarial)
**Date:** 2026-04-18
**Outcome:** **REVIEW: Changes Requested**

### Summary

The three panels (Scoring, Pricing, Win Themes) and their backing API/React Query layers are well implemented and faithful to spec — types match the backend Pydantic schemas exactly, null-result detection uses the correct `'result' in data && data.result === null` pattern, recharts is used as documented, the `de-DE` locale for EUR formatting is correctly chosen, the win-themes drag uses HTML5-only DnD with `useRef` for the drag index (no re-render storm), and all 86 ATDD assertions pass. The workspace integration via `handleScoreClick` + dynamic imports + tab content panels is wired correctly through the **inline** `ProposalToolbar`.

However, Task 6.1 was implemented incorrectly against the wrong toolbar component, which produces two real defects in the rendered DOM. These need to be fixed before the story can ship.

### Findings (must fix)

1. **Hardcoded English string "Score" in `ProposalEditorToolbar.tsx` (line 57) — i18n violation.**
   The new `<button data-testid="toolbar-btn-score">` has `title={t("btnScore")}` and `aria-label={t("btnScore")}` but its visible children are the literal English word `Score`. CLAUDE.md (Critical Conventions) and project-context.md require all user-facing strings to go through `next-intl t()` — no hardcoded English. Fix: replace the literal `Score` with `{t("btnScore")}` (the key already exists in both `en.json` and `bg.json`).

2. **Duplicate `data-testid="toolbar-btn-score"` in the rendered page.**
   `ProposalWorkspacePage` renders both the inline `ProposalToolbar` (with the working Score button at line 308) AND `<ProposalEditor>`, which renders `<ProposalEditorToolbar>` — and that file (post-change, line 49) now also emits `data-testid="toolbar-btn-score"`. Two elements share the same testid in the same DOM tree. This will cause Playwright `getByTestId` strict-mode violations the moment an E2E test targets `toolbar-btn-score` and breaks the project's "unique testid" assumption.

3. **Dead onClick on the editor-toolbar Score button.**
   The new button in `ProposalEditorToolbar.tsx` is wired to `onClick={onScoreClick}`, but the only consumer — `ProposalEditor.tsx` line 289 — does not pass `onScoreClick`. So the second Score button does nothing when clicked. The story's Task 6.1 instruction ("add `onClick={onScoreClick}` to `<Button data-testid='toolbar-btn-score'>`") referred to the existing button in the inline `ProposalToolbar` (the one in `ProposalWorkspacePage.tsx`), not a new button in the formatting toolbar. The formatting toolbar (Bold/Italic/H1/Link) is also conceptually the wrong place for an action button.

   **Recommended fix:** revert `ProposalEditorToolbar.tsx` to its pre-change state (remove both the `onScoreClick` prop and the `<button data-testid="toolbar-btn-score">` element). The functional Score button already exists in `ProposalToolbar` inside `ProposalWorkspacePage.tsx` and is correctly wired. After reverting, update the ATDD test `AC7: Toolbar wiring` block — the `expect(content).toContain("onScoreClick")` and the `onScoreClick\s*\??\s*:\s*\(\)\s*=>` regex assertion should be retargeted at `ProposalWorkspacePage.tsx` (where `ProposalToolbar`'s `onScoreClick` prop already lives at line 182), not at `ProposalEditorToolbar.tsx`.

### Findings (nice to have, non-blocking)

4. **`WinThemesPanel.tsx` lines 4–5 split `lucide-react` into two import statements.** Combine into `import { Loader2, GripVertical } from "lucide-react"` for cleanliness.

5. **`key={index}` on the win-themes draggable list (`WinThemesPanel.tsx` line 104).** Using array index as React key for a reorderable list is a known anti-pattern — when the list reorders, React will reuse DOM nodes by position rather than identity, which can cause subtle reconciliation issues if any card later acquires local state (e.g. an "expanded" toggle, focus). Today the cards are stateless so it works, but a stable key (e.g. `key={theme.title}` if titles are unique, or a generated id) is safer for the future.

6. **`PricingPanel.tsx` line 92** has both `my-4` and `mb-8` Tailwind classes on the same div; the `mb-8` overrides the `my-4` bottom margin — `my-4 mb-8` works but `mt-4 mb-8` is more honest about intent.

### Verified

- ✅ All 86 ATDD assertions in `scoring-pricing-win-themes-s7-15.test.ts` pass (vitest run).
- ✅ TypeScript types match backend Pydantic schemas (no `id` field on `ScoreCardCriterion`; `score`/`max_score` are `number` for floats).
- ✅ Null-result detection uses `'result' in data && data.result === null` (no try/catch on 404).
- ✅ recharts not re-installed (already in `apps/client/package.json` at `^2.12.0`).
- ✅ `de-DE` + `EUR` locale combination used correctly; `en-EU` not present.
- ✅ Drag index in `useRef<number | null>`, not `useState`.
- ✅ i18n: 22 keys added to both `en.json` and `bg.json`, full key-count parity.
- ✅ Dynamic imports use named-export re-mapping, all three panels `ssr: false` with skeleton loaders.
- ✅ `handleScoreClick` correctly calls `setRightCollapsed(false); setRightActiveTab("scoring")` and is wired to `<ProposalToolbar>` in the workspace page.
- ✅ E2E spec contains S07.15 skip stubs for E07-P2-010 and E07-P2-012.
- ⚠️ Two pre-existing failures in `proposals-workspace-s7-11.test.ts` (`Editor area — S07.12` and `"draft"` literals) are unrelated to this story — they reference S07.12 placeholders that were superseded long before S07.15 began.

### Verdict

**REVIEW: Changes Requested.** Fix findings 1–3 (i18n violation + duplicate testid + dead button in the wrong toolbar). Findings 4–6 are optional polish.

---

## Senior Developer Review — Follow-up (Re-review)

**Reviewer:** Code Review Agent (bmad-code-review, adversarial, autopilot)
**Date:** 2026-04-18
**Outcome:** **REVIEW: Approve**

### Summary

All six findings from the prior review (Findings 1–3 must-fix + Findings 4–6 polish) are confirmed resolved on disk. Re-ran the ATDD suite and the i18n parity check; both green.

### Verification

- ✅ **Findings 1+2+3 (toolbar revert):** `ProposalEditorToolbar.tsx` no longer contains `onScoreClick` or `data-testid="toolbar-btn-score"` (verified via grep — zero matches). The single, functional Score button lives in the inline `ProposalToolbar` defined in `ProposalWorkspacePage.tsx` (line 308) with `onClick={onScoreClick}` wired to `handleScoreClick` → `setRightCollapsed(false); setRightActiveTab("scoring")`. No duplicate testid in the rendered DOM. No hardcoded English string. No dead onClick.
- ✅ **Finding 4 (lucide-react imports):** `WinThemesPanel.tsx` line 4 — `import { Loader2, GripVertical } from "lucide-react"` (single combined statement).
- ✅ **Finding 5 (stable key for reorderable list):** `WinThemesPanel.tsx` line 107 — `key={theme.title}` with an inline comment explaining why the index-as-key anti-pattern was avoided.
- ✅ **Finding 6 (margin clarity):** `PricingPanel.tsx` line 92 — `mt-4 mb-8` (no overlapping `my-4`).
- ✅ **ATDD:** `pnpm vitest run __tests__/scoring-pricing-win-themes-s7-15.test.ts` → **87/87 pass** in ~550ms. Test count rose from 86 → 87 with the regression-guard test (`ProposalEditorToolbar.tsx` does NOT define `onScoreClick` and does NOT render `toolbar-btn-score`).
- ✅ **i18n parity:** `pnpm check:i18n` → "i18n keys match: 783 keys in both bg.json and en.json".
- ✅ **`s7-11.test.ts` cleanup:** the two stale `scoring-panel-placeholder` / `win-themes-panel-placeholder` assertions in `proposals-workspace-s7-11.test.ts` are commented out (consistent with the placeholder removals shipped here).

### Adversarial residue (non-blocking, defer)

- **`data-amber={isAmber}` on every row (`ScoringSimulatorPanel.tsx` line 114).** React serializes booleans on data-* attributes, so non-amber rows render `data-amber="false"` rather than omitting the attribute. The story spec only requires `data-amber="true"` on amber rows — no test checks for absence — so this is functionally fine. A future polish could be `data-amber={isAmber ? "true" : undefined}` to match the natural reading of the spec.
- **`PricingPanel.tsx toPercent` divide-by-zero when `min === max`.** Backend should not return a degenerate market range, but `((val - min) / (max - min)) * 100` evaluates to `NaN`, which `Math.max/min` propagate, producing `left: NaN%` and silently dropping the marker. Defensive guard (`if (max === min) return 50`) would be a small hardening; not a blocker.
- **`WinThemesPanel.tsx` `useEffect([resultsQuery.data])` resets local order on refetch.** Per spec, reorder is display-only and not persisted, so this is by design — but a user who reorders, then triggers a refetch (e.g. via stale-time expiry + tab focus) will see their order reset. Worth a UX follow-up if reorder ever needs persistence, but matches the AC18 contract today.

### Verdict

**REVIEW: Approve.** All must-fix findings from the prior round are resolved; ATDD and i18n gates green; implementation faithful to spec across all 22 acceptance criteria. The three residual notes above are non-blocking polish suitable for future stories.
