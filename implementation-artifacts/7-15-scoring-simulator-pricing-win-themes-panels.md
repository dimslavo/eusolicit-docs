# Story 7.15: Scoring Simulator, Pricing & Win Themes Panels

Status: ready-for-dev

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

- [ ] Task 1: Add scoring/pricing/win-themes API types and functions to `lib/api/proposals.ts` (AC: 2, 9, 16)
  - [ ] 1.1 Add `ScoreCardCriterion` interface — fields: `criterion: string; score: number; max_score: number; suggestion: string` (NO `id` field — backend has none)
  - [ ] 1.2 Add `ScoringSimulationResponse` interface — fields: `criteria: ScoreCardCriterion[]; simulated_at: string`
  - [ ] 1.3 Add `MarketRange` interface — fields: `min: number; median: number; max: number`
  - [ ] 1.4 Add `PricingAssistResponse` interface — fields: `recommended_price: number; market_range: MarketRange; justification: string; priced_at: string`
  - [ ] 1.5 Add `WinTheme` interface — fields: `title: string; description: string`
  - [ ] 1.6 Add `WinThemesResponse` interface — fields: `themes: WinTheme[]; analyzed_at: string`
  - [ ] 1.7 Add `runScoringSimulation(proposalId: string): Promise<ScoringSimulationResponse>` — `POST /api/v1/proposals/${proposalId}/scoring-simulation`; `return (await apiClient.post<ScoringSimulationResponse>(..., {})).data`
  - [ ] 1.8 Add `getScoringResults(proposalId: string): Promise<ScoringSimulationResponse | null>` — `GET /api/v1/proposals/${proposalId}/scoring-simulation`; detect null via `'result' in data && data.result === null`
  - [ ] 1.9 Add `runPricingAssist(proposalId: string): Promise<PricingAssistResponse>` — `POST /api/v1/proposals/${proposalId}/pricing-assist`; return `.data`
  - [ ] 1.10 Add `getPricingResults(proposalId: string): Promise<PricingAssistResponse | null>` — GET; null-result detection same as scoring
  - [ ] 1.11 Add `runWinThemes(proposalId: string): Promise<WinThemesResponse>` — `POST /api/v1/proposals/${proposalId}/win-themes`; return `.data`
  - [ ] 1.12 Add `getWinThemes(proposalId: string): Promise<WinThemesResponse | null>` — GET; null-result detection same pattern

- [ ] Task 2: Add React Query hooks to `lib/queries/use-proposals.ts` (AC: 2, 9, 16)
  - [ ] 2.1 Add `useScoringResults(proposalId: string)` — `useQuery({ queryKey: ["scoring", proposalId], queryFn: () => getScoringResults(proposalId), staleTime: 60_000 })`
  - [ ] 2.2 Add `useRunScoringSimulation(proposalId: string)` — `useMutation` calling `runScoringSimulation(proposalId)`; `onSuccess`: `queryClient.invalidateQueries({ queryKey: ["scoring", proposalId] })`
  - [ ] 2.3 Add `usePricingResults(proposalId: string)` — `useQuery({ queryKey: ["pricing", proposalId], queryFn: () => getPricingResults(proposalId), staleTime: 60_000 })`
  - [ ] 2.4 Add `useRunPricingAssist(proposalId: string)` — `useMutation`; `onSuccess`: invalidate `["pricing", proposalId]`
  - [ ] 2.5 Add `useWinThemesResults(proposalId: string)` — `useQuery({ queryKey: ["win-themes", proposalId], queryFn: () => getWinThemes(proposalId), staleTime: 60_000 })`
  - [ ] 2.6 Add `useRunWinThemes(proposalId: string)` — `useMutation`; `onSuccess`: invalidate `["win-themes", proposalId]`

- [ ] Task 3: Create `ScoringSimulatorPanel.tsx` (AC: 1–7)
  - [ ] 3.1 Create `…/proposals/[id]/components/ScoringSimulatorPanel.tsx` with `"use client"` directive
  - [ ] 3.2 Props: `interface ScoringSimulatorPanelProps { proposalId: string; }`
  - [ ] 3.3 Fetch stored results via `useScoringResults(proposalId)`; run mutation via `useRunScoringSimulation(proposalId)`
  - [ ] 3.4 Render `data-testid="btn-simulate-score"` trigger button (disabled while mutation pending)
  - [ ] 3.5 When `resultsQuery.isLoading`: render `<div data-testid="scoring-loading" className="animate-pulse ...">` skeleton
  - [ ] 3.6 When mutation/query error: render `<div data-testid="scoring-error">` + `<Button data-testid="btn-scoring-retry">`
  - [ ] 3.7 When results available: render `<ScoringSimulatorPanel>` content:
    - Recharts `<ResponsiveContainer width="100%" height={220}><RadarChart data={data}>` with `data-testid="scoring-radar-chart"` on the outer div
    - `data` computed as `criteria.map(c => ({ criterion: c.criterion, value: (c.score / c.max_score) * 100 }))`
    - `<PolarGrid />`, `<PolarAngleAxis dataKey="criterion" tick={{ fontSize: 11 }} />`, `<PolarRadiusAxis domain={[0, 100]} tick={false} />`, `<Radar name="Score" dataKey="value" stroke="#4f46e5" fill="#4f46e5" fillOpacity={0.4} />`
  - [ ] 3.8 Criteria table `data-testid="scoring-criteria-table"` with rows `data-testid="scoring-row-{i}"` (zero-based index); amber row: `criterion.score / criterion.max_score < 0.70` → `data-amber="true"` + `className="text-amber-600"` on score cell
  - [ ] 3.9 Re-run button `data-testid="btn-resimulate-score"` shown when results exist
  - [ ] 3.10 All text via `useTranslations("proposals")`; `addToast` from `@/lib/stores/ui-store` for errors

- [ ] Task 4: Create `PricingPanel.tsx` (AC: 8–14)
  - [ ] 4.1 Create `…/proposals/[id]/components/PricingPanel.tsx` with `"use client"` directive
  - [ ] 4.2 Props: `interface PricingPanelProps { proposalId: string; }`
  - [ ] 4.3 Fetch via `usePricingResults(proposalId)`; mutate via `useRunPricingAssist(proposalId)`
  - [ ] 4.4 Trigger button `data-testid="btn-get-pricing"` (disabled while pending); loading `data-testid="pricing-loading"`; error `data-testid="pricing-error"` + `data-testid="btn-pricing-retry"`
  - [ ] 4.5 Recommended price: `<p data-testid="pricing-recommended-price">` with `new Intl.NumberFormat("de-DE", { style: "currency", currency: "EUR" }).format(result.recommended_price)` — use `"de-DE"` locale (not `"en-EU"` which is invalid)
  - [ ] 4.6 Market range bar: `<div data-testid="pricing-market-range" className="relative h-3 rounded-full bg-slate-200 my-4">` with span labels for min/max, div markers for median and recommended_price; use `toPercent = (val) => Math.max(0, Math.min(100, ((val - min) / (max - min)) * 100))`; only render recommended marker when `min <= recommended_price <= max`
  - [ ] 4.7 Justification: `<p data-testid="pricing-justification">{result.justification}</p>`
  - [ ] 4.8 Re-advise button `data-testid="btn-readvise-pricing"` after results
  - [ ] 4.9 All text via `useTranslations("proposals")`

- [ ] Task 5: Create `WinThemesPanel.tsx` (AC: 15–20)
  - [ ] 5.1 Create `…/proposals/[id]/components/WinThemesPanel.tsx` with `"use client"` directive
  - [ ] 5.2 Props: `interface WinThemesPanelProps { proposalId: string; }`
  - [ ] 5.3 Fetch via `useWinThemesResults(proposalId)`; mutate via `useRunWinThemes(proposalId)`. After successful run or query: initialize `useState<WinTheme[]>` from `result.themes`
  - [ ] 5.4 Trigger button `data-testid="btn-extract-win-themes"`; loading `data-testid="win-themes-loading"`; error `data-testid="win-themes-error"` + `data-testid="btn-win-themes-retry"`
  - [ ] 5.5 Theme list `data-testid="win-themes-list"`; each card `data-testid="win-theme-card-{index}"`
  - [ ] 5.6 Drag-and-drop (HTML5 only — no library):
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
  - [ ] 5.7 Drag handle: `<GripVertical data-testid="win-theme-drag-handle-{index}" />` from `lucide-react`
  - [ ] 5.8 Re-extract button `data-testid="btn-reextract-win-themes"`; all text via `useTranslations("proposals")`

- [ ] Task 6: Update `ProposalEditorToolbar` + `ProposalWorkspacePage` (AC: 1, 7, 8, 15)
  - [ ] 6.1 In `ProposalEditorToolbar` component, add prop `onScoreClick?: () => void` to interface; add `onClick={onScoreClick}` to `<Button data-testid="toolbar-btn-score">`
  - [ ] 6.2 In `ProposalWorkspacePage`, add handler:
    ```typescript
    const handleScoreClick = () => {
      setRightCollapsed(false);
      setRightActiveTab("scoring");
    };
    ```
  - [ ] 6.3 Pass `onScoreClick={handleScoreClick}` to `<ProposalEditorToolbar>`
  - [ ] 6.4 Add dynamic imports for all three panels (pattern from S07.14):
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
  - [ ] 6.5 Replace `<div data-testid="scoring-panel-placeholder">Scoring — S07.15</div>` with `<ScoringSimulatorPanel proposalId={proposal.id} />`
  - [ ] 6.6 Replace `<div data-testid="pricing-panel-placeholder">Pricing — S07.15</div>` with `<PricingPanel proposalId={proposal.id} />`
  - [ ] 6.7 Replace `<div data-testid="win-themes-panel-placeholder">Win Themes — S07.15</div>` with `<WinThemesPanel proposalId={proposal.id} />`

- [ ] Task 7: Add i18n keys (AC: 21)
  - [ ] 7.1 Add all flat keys (see Dev Notes) to `messages/en.json` under `proposals` namespace (alongside existing keys — no nested objects)
  - [ ] 7.2 Add matching Bulgarian translations to `messages/bg.json`
  - [ ] 7.3 Run `pnpm check:i18n` to verify key parity

- [ ] Task 8: Write ATDD tests (AC: 22)
  - [ ] 8.1 Create `eusolicit-app/frontend/apps/client/__tests__/scoring-pricing-win-themes-s7-15.test.ts` using `fs.readFile` + `.toContain()` / `.not.toContain()` pattern from `ai-draft-generation-s7-13.test.ts` and `requirement-checklist-compliance-s7-14.test.ts`
  - [ ] 8.2 Assert all three component files exist (paths from Dev Notes file structure)
  - [ ] 8.3 Assert all `data-testid` values present in source (scoring: `btn-simulate-score`, `scoring-radar-chart`, `scoring-criteria-table`, `scoring-row-`, `btn-resimulate-score`, `scoring-loading`, `scoring-error`, `btn-scoring-retry`; pricing: `btn-get-pricing`, `pricing-recommended-price`, `pricing-market-range`, `pricing-justification`, `btn-readvise-pricing`, `pricing-loading`, `pricing-error`, `btn-pricing-retry`; win themes: `btn-extract-win-themes`, `win-themes-list`, `win-theme-card-`, `win-theme-drag-handle-`, `btn-reextract-win-themes`, `win-themes-loading`, `win-themes-error`, `btn-win-themes-retry`)
  - [ ] 8.4 Assert `RadarChart` imported from `recharts` in `ScoringSimulatorPanel.tsx`
  - [ ] 8.5 Assert `Intl.NumberFormat` + `"de-DE"` present in `PricingPanel.tsx`
  - [ ] 8.6 Assert `GripVertical` imported from `lucide-react` in `WinThemesPanel.tsx`
  - [ ] 8.7 Assert amber threshold: `ScoringSimulatorPanel.tsx` contains `0.7` or `0.70`
  - [ ] 8.8 Assert `ScoreCardCriterion`, `ScoringSimulationResponse`, `PricingAssistResponse`, `WinTheme`, `WinThemesResponse` exported from `proposals.ts`
  - [ ] 8.9 Assert `useScoringResults`, `useRunScoringSimulation`, `usePricingResults`, `useRunPricingAssist`, `useWinThemesResults`, `useRunWinThemes` exported from `use-proposals.ts`
  - [ ] 8.10 Assert i18n key completeness (see Dev Notes for required keys)
  - [ ] 8.11 Assert `ProposalWorkspacePage.tsx` does NOT contain `"Scoring — S07.15"`, `"Pricing — S07.15"`, `"Win Themes — S07.15"` (placeholders removed)
  - [ ] 8.12 Add two skipped E2E placeholder tests to `eusolicit-app/e2e/specs/proposals/proposals-workspace.spec.ts`

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

### Debug Log References

### Completion Notes List

### File List
