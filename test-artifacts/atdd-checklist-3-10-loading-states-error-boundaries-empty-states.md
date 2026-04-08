---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-08'
workflowType: 'testarch-atdd'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/3-10-loading-states-error-boundaries-empty-states.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-03.md'
  - 'eusolicit-app/frontend/apps/client/vitest.config.ts'
  - 'eusolicit-app/frontend/apps/client/__tests__/wizard-s3-9.test.ts'
  - '_bmad/bmm/config.yaml'
  - 'resources/knowledge/component-tdd.md'
  - 'resources/knowledge/fixture-architecture.md'
  - 'resources/knowledge/test-quality.md'
  - 'resources/knowledge/test-levels-framework.md'
  - 'resources/knowledge/test-priorities-matrix.md'
  - 'resources/knowledge/selector-resilience.md'
---

# ATDD Checklist — Epic 3, Story 3.10: Loading States, Error Boundaries & Empty States

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**Primary Test Level:** Static/File-system (Vitest node environment)
**TDD Phase:** 🔴 RED — All 155 tests failing as expected

---

## Story Summary

Story 3.10 delivers a comprehensive set of reusable UI feedback primitives for the EU Solicit frontend:
five skeleton loader variants (`SkeletonCard`, `SkeletonTable`, `SkeletonList`, `SkeletonText`,
`SkeletonAvatar`), a Next.js App Router global error boundary (`error.tsx`), a per-section
`<ErrorBoundary>` component (wrapping `react-error-boundary`), a base `<EmptyState>` and three
pre-built variants, a `<QueryGuard>` orchestrator component, new i18n keys in both `en.json` and
`bg.json`, and a `/dev/ui-states` visual-QA demo page — all exported from `packages/ui`.

**As a** frontend developer on the EU Solicit team
**I want** reusable skeleton loader variants, a global error boundary, per-section error boundary,
empty state variants, and a `<QueryGuard>` wrapper — all exported from `packages/ui` and wired into
the client app's Next.js App Router
**So that** every feature page that loads data automatically gets consistent, layout-matched loading
skeletons, graceful error recovery, and friendly empty-state messaging without rebuilding these
patterns per feature

---

## Acceptance Criteria

1. **AC1 — Skeleton variants:** Five layout-matched skeleton components exported from `packages/ui`,
   all using `animate-pulse` with `bg-slate-200` override. `SkeletonCard`, `SkeletonTable(rows, cols)`,
   `SkeletonList(items)`, `SkeletonText(lines)`, `SkeletonAvatar(size?)`. All accept `className` prop.

2. **AC2 — Global error boundary:** Next.js App Router `error.tsx` at
   `apps/client/app/[locale]/error.tsx`. `"use client"`, AlertCircle icon, translated heading/desc,
   Try-again button calling `reset()`, collapsible error details. Three `data-testid` attributes.

3. **AC3 — Per-section `<ErrorBoundary>`:** Exported from `packages/ui`. Wraps `react-error-boundary`.
   Accepts optional `fallback` prop. Default fallback: red-50 card + AlertTriangle icon.
   `data-testid="section-error-boundary"` on default fallback root. Exports `ErrorBoundaryProps`.

4. **AC4 — `<EmptyState>` component:** Exported from `packages/ui`. Props: `icon`, `title`,
   `description?`, `action?`, `className?`. Centred flex layout. `data-testid="empty-state"` and
   `data-testid="empty-state-action"`. Exports `EmptyStateProps`.

5. **AC5 — Three pre-built empty state variants:** `EmptyStateNoResults` (SearchX, optional
   `onClearFilters`), `EmptyStateGetStarted` (PlusCircle, fully customisable),
   `EmptyStateNoAccess` (ShieldOff, `mailto:support@eusolicit.com`). All `"use client"` where needed.

6. **AC6 — `<QueryGuard>` component:** Exported from `packages/ui`. Priority render:
   `isLoading → skeleton | isError → error card | isEmpty → emptyState | children`.
   `data-testid` on each conditional wrapper. Exports `QueryGuardProps`.

7. **AC7 — New i18n keys:** `states` namespace (6 keys) added to `en.json` and `bg.json`.
   Two keys added to existing `errors` namespace. Key parity between locales.

8. **AC8 — `/dev/ui-states` demo page:** `"use client"` page at
   `apps/client/app/[locale]/dev/ui-states/page.tsx`. Renders all skeletons, empty state variants,
   and QueryGuard in loading/error/empty states.

9. **AC9 — Exports:** All new symbols in `packages/ui/index.ts` under `// New in S3.10` block.
   Includes types: `SkeletonAvatarSize`, `ErrorBoundaryProps`, `EmptyStateProps`, `QueryGuardProps`.

10. **AC10 — Build & type-check:** `pnpm build` and `pnpm type-check` exit 0 across all packages.
    `pnpm check:i18n --filter client` exits 0. (Verified via CI; not in Vitest scope.)

---

## Preflight & Context (Step 1)

### Stack Detection

**Detected stack:** `frontend`
**Rationale:** Project has `package.json` with React/Next.js dependencies, `playwright.config.ts`
(via framework setup), and `vitest.config.ts`. No backend indicators (`pyproject.toml`, `go.mod`, etc.)
found at project root.

### Prerequisites

- ✅ Story approved with clear acceptance criteria (10 ACs, 10 tasks/subtasks)
- ✅ Test framework configured: `apps/client/vitest.config.ts` (Vitest `environment: 'node'`)
- ✅ Development environment available (Node.js v22.22.1 via nvm)
- ✅ Epic test design loaded: `test-design-epic-03.md` (relevant P1 scenarios E03-P1-015, E03-P1-016, E03-P2-016)

### Coverage Strategy from Epic Test Design

From `test-design-epic-03.md`, Story 3.10 maps to:

| Epic Test ID | Story AC | Priority | Test Type |
|-------------|---------|----------|-----------|
| **E03-P1-015** | AC1 (Skeleton variants) | **P1** | Component |
| **E03-P1-016** | AC2+AC3 (Error boundaries) | **P1** | E2E + Static |
| **E03-P2-016** | AC4+AC5 (Empty state variants) | **P2** | Component |
| — | AC6 (QueryGuard) | **P1** | Static/Component |
| — | AC7 (i18n keys) | **P1** | Static/Build |
| — | AC8 (dev page) | **P2** | Static |
| — | AC9 (exports) | **P1** | Static |
| **E03-P0-001** | AC10 (build) | **P0** | Build/CI |

---

## Generation Mode (Step 2)

**Mode chosen:** AI Generation (static/file-system)

**Rationale:** All acceptance criteria for S3.10 are structural (file presence, source code content,
i18n key values, export declarations). The project's established ATDD pattern (matching
`wizard-s3-9.test.ts`) uses Vitest with `environment: 'node'` to check files and source content
directly — no browser recording needed. This is the fastest and most deterministic approach for this
story's ACs.

**No recording session required.**

---

## Test Strategy (Step 3)

### AC-to-Test Mapping

| AC | Scenarios | Test Level | Priority | Approach |
|----|-----------|------------|----------|---------|
| AC1 | 5 file existence + 35 content assertions (animate-pulse, bg-slate-200, className, per-variant props) | Static/Unit | P1 | File read + string/regex assertions |
| AC2 | 1 file existence + 12 content assertions (use client, testids, icons, i18n keys, reset prop) | Static/Unit | P1 | File read + string assertions |
| AC3 | 1 file existence + 8 content assertions (react-error-boundary, testid, AlertTriangle, red-50) | Static/Unit | P1 | File read + string assertions |
| AC4 | 1 file existence + 9 content assertions (icon/title/action props, testids, Tailwind classes) | Static/Unit | P1 | File read + string assertions |
| AC5 | 3 file existence + 3×5 content assertions (icons, i18n, onClearFilters, mailto) | Static/Unit | P2 | File read + string assertions |
| AC6 | 1 file existence + 12 content assertions (props, testids, priority ordering, default fallbacks) | Static/Unit | P1 | File read + string/ordering assertions |
| AC7 | 2 JSON file existence + 18 key/value assertions + 2 parity assertions | Static/Unit | P1 | JSON parse + property checks |
| AC8 | 1 file existence + 7 content assertions (use client, imports, QueryGuard states) | Static/Unit | P2 | File read + string assertions |
| AC9 | 1 file existence + 17 symbol export assertions + 8 barrel assertions | Static/Unit | P1 | File read + string assertions |
| AC10 | Not in Vitest scope — validated by CI (`pnpm build`, `pnpm type-check`, `pnpm check:i18n`) | Build/CI | P0 | Run commands |

### Red Phase Requirements

- All tests use `it()` (live test, not skipped)
- Tests fail before implementation due to ENOENT (missing files) or missing content
- Tests fail due to **missing implementation**, not test bugs
- Failure messages are clear: ENOENT paths identify exactly which task is missing

---

## Failing Tests Created (RED Phase)

### Static/Unit Tests (159 tests)

**File:** `eusolicit-app/frontend/apps/client/__tests__/loading-states-s3-10.test.ts`

**Run command:**
```bash
cd eusolicit-app/frontend
source ~/.nvm/nvm.sh
./apps/client/node_modules/.bin/vitest run apps/client/__tests__/loading-states-s3-10.test.ts
```

#### AC1 — Skeleton Variants (6 describe blocks, ~47 tests)

- ✅ **Describe: AC1: Skeleton Variant Files — All files exist** (6 tests)
  - **Status:** RED — `expected false to be true` (files don't exist)
  - **Verifies:** Tasks 2.1–2.6 file creation

- ✅ **Describe: AC1: SkeletonCard — structure and styling** (7 tests)
  - **Status:** RED — `ENOENT: SkeletonCard.tsx`
  - **Verifies:** `export function SkeletonCard`, `className` prop, `bg-slate-200`, relative Skeleton import, header (h-5 w-2/5), body lines (w-full, w-4/5, w-3/5)

- ✅ **Describe: AC1: SkeletonTable — configurable rows×columns** (7 tests)
  - **Status:** RED — `ENOENT: SkeletonTable.tsx`
  - **Verifies:** Export, `rows`/`columns` props, `className`, h-5 header, h-4 cells, `bg-slate-200`, `Array.from` iteration

- ✅ **Describe: AC1: SkeletonList — avatar + text rows** (6 tests)
  - **Status:** RED — `ENOENT: SkeletonList.tsx`
  - **Verifies:** Export, `items` prop, `className`, `rounded-full h-10 w-10`, h-4/h-3 text lines, `bg-slate-200`

- ✅ **Describe: AC1: SkeletonText — configurable paragraph placeholder** (7 tests)
  - **Status:** RED — `ENOENT: SkeletonText.tsx`
  - **Verifies:** Export, `lines` prop, `className`, w-3/5 last line, h-4 lines, `bg-slate-200`, `space-y-2`/`gap-2`

- ✅ **Describe: AC1: SkeletonAvatar — three sizes** (9 tests)
  - **Status:** RED — `ENOENT: SkeletonAvatar.tsx`
  - **Verifies:** Export, size prop (sm/md/lg), h-8 w-8 (sm), h-10 w-10 (md), h-12 w-12 (lg), `rounded-full`, `bg-slate-200`, `className`, `SkeletonAvatarSize` type export

#### AC2 — Global Error Boundary (1 describe block, 12 tests)

- ✅ **Describe: AC2: Global error boundary — error.tsx exists and has correct structure** (12 tests)
  - **Status:** RED — `expected false to be true` (file missing) / `ENOENT: error.tsx`
  - **Verifies:** File exists, `"use client"`, error+reset props, `AlertCircle` from lucide-react, `errors.somethingWentWrong`, `errors.tryAgainDescription`, `data-testid="global-error-boundary"`, `data-testid="error-try-again"`, `data-testid="error-details-toggle"`, `ChevronDown`, centred layout (`items-center justify-center min-h-`), `reset()` on onClick

#### AC3 — Per-section ErrorBoundary (1 describe block, 9 tests)

- ✅ **Describe: AC3: Per-section ErrorBoundary — file exists and has correct structure** (9 tests)
  - **Status:** RED — `expected false to be true` / `ENOENT: ErrorBoundary.tsx`
  - **Verifies:** File exists, `react-error-boundary` import, `ErrorBoundary` export, `ErrorBoundaryProps` export, `fallback` prop, `data-testid="section-error-boundary"`, `AlertTriangle` icon, `red-50` bg, `red-200` border

#### AC4 — EmptyState base component (1 describe block, 10 tests)

- ✅ **Describe: AC4: EmptyState — file exists and has correct structure** (10 tests)
  - **Status:** RED — `expected false to be true` / `ENOENT: EmptyState.tsx`
  - **Verifies:** File exists, `export function EmptyState`, `EmptyStateProps`, `icon` prop, title/description/action/className props, `data-testid="empty-state"`, `data-testid="empty-state-action"`, h-12 w-12 text-slate-400, text-lg/font-semibold/text-slate-900, text-sm/text-slate-500

#### AC5 — Empty State Variants (3 describe blocks, ~19 tests)

- ✅ **Describe: AC5: EmptyStateNoResults** (8 tests)
  - **Status:** RED — `expected false to be true` / `ENOENT: EmptyStateNoResults.tsx`
  - **Verifies:** File, export, `SearchX` icon, `"use client"`, `noResultsTitle`, `noResultsDescription`, `onClearFilters` prop, conditional render of Clear Filters button

- ✅ **Describe: AC5: EmptyStateGetStarted** (5 tests)
  - **Status:** RED — `expected false to be true` / `ENOENT: EmptyStateGetStarted.tsx`
  - **Verifies:** File, export, `PlusCircle` icon, title/description/action props, no `useTranslations` (customisable variant)

- ✅ **Describe: AC5: EmptyStateNoAccess** (7 tests)
  - **Status:** RED — `expected false to be true` / `ENOENT: EmptyStateNoAccess.tsx`
  - **Verifies:** File, export, `ShieldOff` icon, `"use client"`, `noAccessTitle`, `noAccessDescription`, `mailto:support@eusolicit.com`

#### AC6 — QueryGuard (1 describe block, 11 tests)

- ✅ **Describe: AC6: QueryGuard — file exists and has correct structure** (11 tests)
  - **Status:** RED — `expected false to be true` / `ENOENT: QueryGuard.tsx`
  - **Verifies:** File, `export function QueryGuard`, `QueryGuardProps`, `isLoading`/`isError`/`isEmpty` props, `skeleton`/`emptyState`/`children`/`error` props, `data-testid="query-guard-loading"`, `data-testid="query-guard-error"`, `data-testid="query-guard-empty"`, priority ordering (loading before error before empty), `SkeletonList items={3}` default, `EmptyStateNoResults` default

#### AC7 — i18n Keys (3 describe blocks, ~20 tests)

- ✅ **Describe: AC7: i18n keys — en.json** (9 tests)
  - **Status:** RED — JSON key assertion failures (`states` namespace missing, error keys missing)
  - **Verifies:** `states` namespace exists, `noResultsTitle="No results found"`, `noResultsDescription`, `clearFilters`, `noAccessTitle="Access restricted"`, `noAccessDescription`, `requestAccess`, `errors.somethingWentWrong`, `errors.tryAgainDescription`

- ✅ **Describe: AC7: i18n keys — bg.json** (9 tests)
  - **Status:** RED — JSON key assertion failures
  - **Verifies:** `states` namespace, all 6 BG translations (must be non-English), `errors.somethingWentWrong`, `errors.tryAgainDescription`

- ✅ **Describe: AC7: i18n key parity** (2 tests)
  - **Status:** RED — key comparison fails
  - **Verifies:** `bg.json states` keys === `en.json states` keys (sorted), both error keys present in bg

#### AC8 — /dev/ui-states demo page (1 describe block, 8 tests)

- ✅ **Describe: AC8: /dev/ui-states demo page** (8 tests)
  - **Status:** RED — `expected false to be true` / `ENOENT: page.tsx`
  - **Verifies:** File exists, `"use client"`, all 5 skeleton imports, all 3 empty-state imports, `QueryGuard` import, `isLoading={true}`, `isError={true}`, `isEmpty={true}` demo sections

#### AC9 — Exports (2 describe blocks, ~26 tests)

- ✅ **Describe: AC9: packages/ui/index.ts — exports all new S3.10 symbols** (18 tests)
  - **Status:** RED — `S3.10` comment missing and symbols absent
  - **Verifies:** index.ts exists, `S3.10` comment block, all 5 skeleton exports + `SkeletonAvatarSize`, `ErrorBoundary` + `ErrorBoundaryProps`, `EmptyState` + all 3 variants + `EmptyStateProps`, `QueryGuard` + `QueryGuardProps`

- ✅ **Describe: AC9: packages/ui/src/components/feedback/index.ts — barrel exports** (8 tests)
  - **Status:** RED — `ENOENT: feedback/index.ts`
  - **Verifies:** All 8 component groups re-exported from barrel

---

## Test Execution Evidence — RED Phase ✅

### Run Command

```bash
cd /home/debian/Projects/eusolicit/eusolicit-app/frontend
source ~/.nvm/nvm.sh
./apps/client/node_modules/.bin/vitest run apps/client/__tests__/loading-states-s3-10.test.ts
```

### Result Summary

```
Test Files  1 failed (1)
      Tests  155 failed | 4 passed (159)
   Start at  08:42:50
   Duration  428ms
```

- **Total tests:** 159
- **Failing:** 155 ✅ (expected — RED phase)
- **Passing:** 4 (pre-existing en.json/bg.json files exist; `errors` namespace key assertions
  that resolve against existing keys)
- **Status:** ✅ RED phase verified — tests fail due to missing implementation, not test bugs

### Primary Failure Messages

```
expected false to be true // Object.is equality
→ File does not exist (ENOENT assertion)

ENOENT: no such file or directory, open '.../feedback/SkeletonCard.tsx'
ENOENT: no such file or directory, open '.../feedback/SkeletonTable.tsx'
ENOENT: no such file or directory, open '.../feedback/SkeletonList.tsx'
ENOENT: no such file or directory, open '.../feedback/SkeletonText.tsx'
ENOENT: no such file or directory, open '.../feedback/SkeletonAvatar.tsx'
ENOENT: no such file or directory, open '.../feedback/ErrorBoundary.tsx'
ENOENT: no such file or directory, open '.../feedback/EmptyState.tsx'
ENOENT: no such file or directory, open '.../feedback/EmptyStateNoResults.tsx'
ENOENT: no such file or directory, open '.../feedback/EmptyStateGetStarted.tsx'
ENOENT: no such file or directory, open '.../feedback/EmptyStateNoAccess.tsx'
ENOENT: no such file or directory, open '.../feedback/QueryGuard.tsx'
ENOENT: no such file or directory, open '.../feedback/index.ts'
ENOENT: no such file or directory, open '.../app/[locale]/error.tsx'
ENOENT: no such file or directory, open '.../dev/ui-states/page.tsx'
```

---

## Required data-testid Attributes

### Global Error Boundary (`apps/client/app/[locale]/error.tsx`)

- `data-testid="global-error-boundary"` — root `<div>` wrapping entire error UI
- `data-testid="error-try-again"` — the "Try again" `<Button>` that calls `reset()`
- `data-testid="error-details-toggle"` — the collapsible details toggle (ChevronDown button)

### Per-section ErrorBoundary (`packages/ui/src/components/feedback/ErrorBoundary.tsx`)

- `data-testid="section-error-boundary"` — root element of the default fallback inline error card

### EmptyState (`packages/ui/src/components/feedback/EmptyState.tsx`)

- `data-testid="empty-state"` — root element of EmptyState
- `data-testid="empty-state-action"` — action `<Button>` (rendered only when `action` prop provided)

### QueryGuard (`packages/ui/src/components/feedback/QueryGuard.tsx`)

- `data-testid="query-guard-loading"` — wrapper rendered when `isLoading={true}`
- `data-testid="query-guard-error"` — wrapper rendered when `isError={true}`
- `data-testid="query-guard-empty"` — wrapper rendered when `isEmpty={true}`

---

## Mock Requirements

**None required for static tests.**

The test suite uses Vitest with `environment: 'node'` to check file existence and source code
content. No network calls, no browser, no mock services.

For future runtime E2E tests (expanding on E03-P1-016 from the Epic 3 test design):
- Mock `POST /auth/token-refresh` for QueryGuard error state
- No external service mocks needed for skeleton/empty state components

---

## Data Factories Created

**None required for static tests.**

These tests assert file structure and source content only. No factory data needed.
Story 3.10 components are presentational and UI-structural; no data seeding is required.

---

## Implementation Checklist

### Task 1: Install react-error-boundary (AC3)

- [ ] 1.1 Add `"react-error-boundary": "^4.1.0"` to `packages/ui/package.json` → `dependencies`
- [ ] 1.2 Run `pnpm install` from `eusolicit-app/frontend/`
- [ ] Run test to confirm: `AC3: Per-section ErrorBoundary > ErrorBoundary.tsx imports from react-error-boundary`
- [ ] Estimated effort: 0.25h

---

### Task 2: Create skeleton variant components in `packages/ui` (AC1)

**Make pass (43 tests in 6 describe blocks):**

- [ ] 2.1 Create `packages/ui/src/components/feedback/SkeletonCard.tsx`
  - `export function SkeletonCard({ className }: SkeletonCardProps)`
  - Imports: `Skeleton` from `"../ui/skeleton"`, `cn` from `"../../lib/utils"`
  - Header: `<Skeleton className="h-5 w-2/5 bg-slate-200" />`
  - Body: 3 lines (w-full, w-4/5, w-3/5) all `h-4 bg-slate-200`
  - Footer: `<Skeleton className="h-4 w-1/3 bg-slate-200" />`

- [ ] 2.2 Create `packages/ui/src/components/feedback/SkeletonTable.tsx`
  - Props: `rows?: number` (default=3), `columns?: number` (default=4), `className?`
  - Uses `Array.from({ length: columns })` and `Array.from({ length: rows })`
  - Header row: `h-5 bg-slate-200`; data rows: `h-4 bg-slate-200`

- [ ] 2.3 Create `packages/ui/src/components/feedback/SkeletonList.tsx`
  - Props: `items?: number` (default=3), `className?`
  - Each item: circular avatar `h-10 w-10 rounded-full bg-slate-200` + text line `h-4` (full) + `h-3 w-3/5`

- [ ] 2.4 Create `packages/ui/src/components/feedback/SkeletonText.tsx`
  - Props: `lines?: number` (default=3), `className?`
  - Lines use `h-4 bg-slate-200`; last line `w-3/5`; gap uses `space-y-2`

- [ ] 2.5 Create `packages/ui/src/components/feedback/SkeletonAvatar.tsx`
  - Props: `size?: "sm" | "md" | "lg"` (default="md"), `className?`
  - `export type SkeletonAvatarSize = "sm" | "md" | "lg"`
  - sm=`h-8 w-8`, md=`h-10 w-10`, lg=`h-12 w-12`; all `rounded-full bg-slate-200`

- [ ] 2.6 Create `packages/ui/src/components/feedback/index.ts` barrel
  - Re-exports all 5 skeletons + ErrorBoundary + EmptyState variants + QueryGuard

- [ ] Run tests: 43 skeleton tests should turn green
- [ ] **Estimated effort:** 2h

---

### Task 3: Create global error.tsx in client app (AC2)

**Make pass (12 tests in 1 describe block):**

- [ ] 3.1 Create `apps/client/app/[locale]/error.tsx`
  - `"use client"` directive at top
  - Props: `{ error: Error & { digest?: string }; reset: () => void }`
  - Layout: `<div data-testid="global-error-boundary" className="flex min-h-[60vh] flex-col items-center justify-center gap-4">`
  - AlertCircle icon from `lucide-react` (h-12 w-12)
  - `useTranslations("errors")` → `t("errors.somethingWentWrong")`, `t("errors.tryAgainDescription")`
  - `<Button data-testid="error-try-again" onClick={reset}>Try again</Button>`
  - Collapsible details: ChevronDown toggle `data-testid="error-details-toggle"`, shows `error.message` + `error.digest`

- [ ] Run tests: 12 global error boundary tests should turn green
- [ ] **Estimated effort:** 1h

---

### Task 4: Create per-section `<ErrorBoundary>` in `packages/ui` (AC3)

**Make pass (9 tests in 1 describe block):**

- [ ] 4.1 Create `packages/ui/src/components/feedback/ErrorBoundary.tsx`
  - Wraps `react-error-boundary` `ErrorBoundary`
  - `export interface ErrorBoundaryProps { fallback?: React.ReactNode; children: React.ReactNode }`
  - Default fallback: `<div data-testid="section-error-boundary" className="... bg-red-50 border border-red-200 ...">`
    with `AlertTriangle` from lucide-react (h-5 w-5 text-red-500) + "Something went wrong" message
  - When `fallback` prop provided: use it instead of default

- [ ] Run tests: 9 ErrorBoundary tests should turn green
- [ ] **Estimated effort:** 1h

---

### Task 5: Create `<EmptyState>` and three variants in `packages/ui` (AC4, AC5)

**Make pass (10 + 8 + 5 + 7 = 30 tests in 4 describe blocks):**

- [ ] 5.1 Create `packages/ui/src/components/feedback/EmptyState.tsx`
  - `export interface EmptyStateProps { icon: React.ElementType; title: string; description?: string; action?: { label: string; onClick: () => void }; className?: string }`
  - Root: `<div data-testid="empty-state" className="flex flex-col items-center ...">`
  - Icon: rendered at `h-12 w-12 text-slate-400`
  - Title: `text-lg font-semibold text-slate-900`
  - Description: `text-sm text-slate-500`
  - Action: `<Button data-testid="empty-state-action" variant="default">`

- [ ] 5.2 Create `packages/ui/src/components/feedback/EmptyStateNoResults.tsx`
  - `"use client"` directive
  - Uses `SearchX` icon from `lucide-react`
  - `useTranslations("states")` → `t("noResultsTitle")`, `t("noResultsDescription")`, `t("clearFilters")`
  - Props: `{ onClearFilters?: () => void }`
  - Clear Filters button only rendered when `onClearFilters` is provided

- [ ] 5.3 Create `packages/ui/src/components/feedback/EmptyStateGetStarted.tsx`
  - Uses `PlusCircle` icon from `lucide-react`
  - Props: `{ title: string; description?: string; action: { label: string; onClick: () => void } }`
  - No `useTranslations` — all strings passed as props

- [ ] 5.4 Create `packages/ui/src/components/feedback/EmptyStateNoAccess.tsx`
  - `"use client"` directive
  - Uses `ShieldOff` icon from `lucide-react`
  - `useTranslations("states")` → `t("noAccessTitle")`, `t("noAccessDescription")`, `t("requestAccess")`
  - "Request access" action calls `window.location.href = "mailto:support@eusolicit.com"`

- [ ] Run tests: 30 EmptyState tests should turn green
- [ ] **Estimated effort:** 2h

---

### Task 6: Create `<QueryGuard>` in `packages/ui` (AC6)

**Make pass (11 tests in 1 describe block):**

- [ ] 6.1 Create `packages/ui/src/components/feedback/QueryGuard.tsx`
  - `export interface QueryGuardProps { isLoading: boolean; isError: boolean; isEmpty: boolean; error?: unknown; skeleton?: React.ReactNode; emptyState?: React.ReactNode; children: React.ReactNode }`
  - Render priority: 1→loading, 2→error, 3→empty, 4→children
  - Loading wrapper: `<div data-testid="query-guard-loading">{skeleton ?? <SkeletonList items={3} />}</div>`
  - Error wrapper: `<div data-testid="query-guard-error">` inline error card
  - Empty wrapper: `<div data-testid="query-guard-empty">{emptyState ?? <EmptyStateNoResults />}</div>`
  - Import `SkeletonList` and `EmptyStateNoResults` from relative paths within feedback/

- [ ] Run tests: 11 QueryGuard tests should turn green
- [ ] **Estimated effort:** 1h

---

### Task 7: Add new i18n keys (AC7)

**Make pass (20 tests in 3 describe blocks):**

- [ ] 7.1 Update `apps/client/messages/en.json`:
  - Add `"states"` namespace: `noResultsTitle`, `noResultsDescription`, `clearFilters`, `noAccessTitle`, `noAccessDescription`, `requestAccess`
  - Add to existing `"errors"` namespace: `"somethingWentWrong": "Something went wrong"`, `"tryAgainDescription": "An unexpected error occurred. You can try reloading this section or go back to the previous page."`

- [ ] 7.2 Update `apps/client/messages/bg.json`:
  - Add `"states"` namespace with Bulgarian translations for all 6 keys
  - Add to `"errors"` namespace: BG translations for `somethingWentWrong` and `tryAgainDescription`
  - Key set must exactly match `en.json states` keys

- [ ] 7.3 Run `pnpm check:i18n --filter client` from `eusolicit-app/frontend/` — must exit 0

- [ ] Run tests: 20 i18n tests should turn green
- [ ] **Estimated effort:** 1h

---

### Task 8: Create `/dev/ui-states` demo page (AC8)

**Make pass (8 tests in 1 describe block):**

- [ ] 8.1 Create `apps/client/app/[locale]/dev/ui-states/page.tsx`
  - `"use client"` directive
  - Import all 5 skeletons, all 3 empty state variants, `QueryGuard`
  - Section "Skeleton Variants": renders all 5 skeletons
  - Section "Empty States": renders 3 empty state variants
  - Section "QueryGuard — Loading": `<QueryGuard isLoading={true} isError={false} isEmpty={false}>`
  - Section "QueryGuard — Error": `<QueryGuard isLoading={false} isError={true} isEmpty={false}>`
  - Section "QueryGuard — Empty": `<QueryGuard isLoading={false} isError={false} isEmpty={true}>`

- [ ] Run tests: 8 demo page tests should turn green
- [ ] **Estimated effort:** 0.75h

---

### Task 9: Update `packages/ui/index.ts` exports (AC9)

**Make pass (26 tests in 2 describe blocks):**

- [ ] 9.1 Append to `packages/ui/index.ts`:
  ```ts
  // New in S3.10
  export { SkeletonCard } from './src/components/feedback/SkeletonCard';
  export { SkeletonTable } from './src/components/feedback/SkeletonTable';
  export { SkeletonList } from './src/components/feedback/SkeletonList';
  export { SkeletonText } from './src/components/feedback/SkeletonText';
  export { SkeletonAvatar } from './src/components/feedback/SkeletonAvatar';
  export type { SkeletonAvatarSize } from './src/components/feedback/SkeletonAvatar';
  export { ErrorBoundary } from './src/components/feedback/ErrorBoundary';
  export type { ErrorBoundaryProps } from './src/components/feedback/ErrorBoundary';
  export { EmptyState } from './src/components/feedback/EmptyState';
  export type { EmptyStateProps } from './src/components/feedback/EmptyState';
  export { EmptyStateNoResults } from './src/components/feedback/EmptyStateNoResults';
  export { EmptyStateGetStarted } from './src/components/feedback/EmptyStateGetStarted';
  export { EmptyStateNoAccess } from './src/components/feedback/EmptyStateNoAccess';
  export { QueryGuard } from './src/components/feedback/QueryGuard';
  export type { QueryGuardProps } from './src/components/feedback/QueryGuard';
  ```

- [ ] Update `packages/ui/src/components/feedback/index.ts` barrel to re-export all new components

- [ ] Run tests: 26 export tests should turn green
- [ ] **Estimated effort:** 0.5h

---

### Task 10: Build & type-check verification (AC10)

*Not in Vitest scope — run manually in CI or local terminal.*

- [ ] 10.1 `pnpm build` from `eusolicit-app/frontend/` — both `client` and `admin` apps exit 0
- [ ] 10.2 `pnpm type-check` — zero TypeScript errors across all packages
- [ ] 10.3 `pnpm check:i18n --filter client` — exits 0
- [ ] **Estimated effort:** 0.5h (verification only; build issues would be type errors)

---

## Running Tests

```bash
# Run all failing tests for this story
cd eusolicit-app/frontend
source ~/.nvm/nvm.sh
./apps/client/node_modules/.bin/vitest run apps/client/__tests__/loading-states-s3-10.test.ts

# Run in watch mode during development
./apps/client/node_modules/.bin/vitest watch apps/client/__tests__/loading-states-s3-10.test.ts

# Run all client tests (includes S3.10 + previous stories)
./apps/client/node_modules/.bin/vitest run --reporter=verbose

# Run with coverage
./apps/client/node_modules/.bin/vitest run --coverage apps/client/__tests__/loading-states-s3-10.test.ts

# Run build verification (AC10)
cd eusolicit-app/frontend
pnpm build
pnpm type-check
pnpm check:i18n --filter client
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

**TEA Agent Responsibilities:**

- ✅ All 159 tests written
- ✅ 155 tests failing (RED) — confirmed by test run
- ✅ Failure messages are clear: ENOENT paths identify missing files by task number
- ✅ 4 tests passing (pre-existing files: en.json, bg.json)
- ✅ Tests fail due to missing implementation, not test bugs
- ✅ Mock requirements documented (none needed for static phase)
- ✅ data-testid requirements listed
- ✅ Implementation checklist created with per-task test counts

**Verification:**

```
Test Files  1 failed (1)
      Tests  155 failed | 4 passed (159)
✅ RED phase verified
```

---

### GREEN Phase (DEV Team — Next Steps)

**DEV Agent Responsibilities:**

1. **Pick one task** from the implementation checklist (start with Task 1: install react-error-boundary)
2. **Read the failing tests** for that task to understand expected behavior
3. **Implement the code** per Dev Notes in the story spec
4. **Run the tests** — verify they turn green
5. **Move to next task** in sequence (Tasks 1→2→3→4→5→6→7→8→9→10)

**Recommended task order:**

1. Task 1 — install react-error-boundary (unblocks Task 4)
2. Task 2 — skeleton variants (most test coverage, no dependencies)
3. Task 4 — ErrorBoundary (depends on Task 1)
4. Task 5 — EmptyState + variants (depends on nothing)
5. Task 6 — QueryGuard (depends on Tasks 2, 5)
6. Task 7 — i18n keys (depends on Tasks 3, 5)
7. Task 3 — global error.tsx (depends on Task 7 for i18n keys)
8. Task 8 — /dev/ui-states demo page (depends on Tasks 2, 5, 6)
9. Task 9 — update index.ts exports (depends on all Tasks 2–5)
10. Task 10 — build verification (depends on all previous tasks)

---

### REFACTOR Phase (DEV Team — After All Tests Pass)

1. Verify all 159 tests pass (green)
2. Check TypeScript types are strict (no implicit any)
3. Verify `pnpm build` and `pnpm type-check` exit 0
4. Run `pnpm check:i18n --filter client` — exit 0
5. Verify no duplicate logic across EmptyState variants
6. Confirm `packages/ui` barrel export does NOT add `"use client"` to non-client components
7. Mark story status as `done` in sprint-status.yaml

---

## Next Steps

1. **Share this checklist** with the dev workflow
2. **Start with Task 1** (install react-error-boundary) — 15 mins
3. **Run failing tests** after each task to confirm RED→GREEN transition
4. **Work one task at a time** — 155 tests total, ~8.5h estimated effort
5. **When all tests pass**, run `pnpm build` + `pnpm type-check` for AC10
6. **When refactoring complete**, update story status to 'done'
7. **Run `*automate`** workflow to expand runtime E2E coverage for E03-P1-015/P1-016/P2-016

---

## Assumptions and Risks

### Assumptions

1. `react-error-boundary` v4 (`^4.1.0`) is compatible with the React 18 version in `packages/ui/package.json`
2. `next-intl` is available as a peer dependency in `packages/ui` via the consuming app's provider — components using `useTranslations` will work at runtime without adding `next-intl` to `packages/ui/package.json`
3. `cn` utility at `packages/ui/src/lib/utils.ts` is already established (from S3.1–3.9)
4. The `Skeleton` component at `packages/ui/src/components/ui/skeleton.tsx` exists (from S3.2 shadcn setup)
5. `Button` component at `packages/ui/src/components/ui/button.tsx` exists (from S3.2)
6. BG translations for `states` namespace can be placeholder Bulgarian text at this stage (accuracy tested by QA, not unit tests)

### Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| `react-error-boundary` v4 API differs from assumed | Medium | Check actual v4 API (`ErrorBoundary` from `react-error-boundary` accepts `FallbackComponent` or `fallbackRender`) |
| `cn` import path in `packages/ui` uses `../../lib/utils` (not `@/`) | Low | Enforced by AC1 test: `SkeletonCard imports Skeleton from relative path` |
| Next.js App Router `error.tsx` must be at `[locale]` level, not app root | Low | Test explicitly checks `apps/client/app/[locale]/error.tsx` — wrong placement caught by ENOENT |
| BG translation strings missing or placeholder empty | Low | AC7 test checks `.toBeTruthy()` for BG values — empty string would fail |

---

## Knowledge Base References Applied

- **component-tdd.md** — Red-Green-Refactor loop, provider isolation, component test patterns
- **fixture-architecture.md** — File-based test pattern (pure assertion, no browser)
- **test-quality.md** — One assertion per concern, deterministic, isolated tests
- **test-levels-framework.md** — Static/unit for structural checks; E2E deferred to `*automate`
- **test-priorities-matrix.md** — P1 for core exports and i18n; P2 for dev page
- **selector-resilience.md** — data-testid requirements documented for future E2E expansion
- **test-design-epic-03.md** — Coverage strategy aligned with E03-P1-015, P1-016, P2-016

---

## Contact

**Questions or Issues?**

- Refer to story spec: `eusolicit-docs/implementation-artifacts/3-10-loading-states-error-boundaries-empty-states.md`
- Epic test design: `eusolicit-docs/test-artifacts/test-design-epic-03.md`
- Consult `.claude/skills/bmad-testarch-atdd/resources/knowledge/` for testing best practices

---

**Generated by BMad TEA Agent** — 2026-04-08
