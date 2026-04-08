---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-08'
workflowType: 'testarch-atdd'
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-03-frontend-shell-design-system.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-03.md'
  - 'eusolicit-app/frontend/apps/client/vitest.config.ts'
  - 'eusolicit-app/frontend/apps/client/__tests__/loading-states-s3-10.test.ts'
  - 'eusolicit-app/frontend/apps/client/lib/stores/ui-store.ts'
  - 'eusolicit-app/frontend/packages/ui/index.ts'
  - '_bmad/bmm/config.yaml'
---

# ATDD Checklist — Epic 3, Story 3.11: Toast Notification System

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**Primary Test Level:** Static/File-system (Vitest node environment)
**TDD Phase:** 🔴 RED — 60 tests failing as expected (6 AC1 regression guards pass)

---

## Story Summary

Story 3.11 delivers the toast notification system for transient feedback messages (success, error,
warning, info). Toasts appear in the bottom-right corner, stack vertically, auto-dismiss after a
configurable duration per type, and can be manually dismissed. The system builds on the `uiStore`
toast state already added in S3.5 and provides:

- A visual `ToastItem` component with type-specific icons and colours
- A `ToastContainer` portal component that subscribes to `uiStore.toasts[]`
- A `useToast()` hook with shorthand convenience methods
- A `/dev/toasts` demo page
- All symbols exported from `packages/ui`

**As a** frontend developer on the EU Solicit team
**I want** a reusable toast notification system (visual components + `useToast()` hook) built on
the `uiStore.toasts[]` state already established in S3.5
**So that** any page or feature can trigger consistent, accessible, auto-dismissing feedback toasts
by calling `toast.success(title)`, `toast.error(title)`, `toast.warning(title)`, or
`toast.info(title)` without rebuilding the notification pattern per feature

---

## Acceptance Criteria

1. **AC1 — `addToast` from `uiStore`:** `addToast({ type, title, description?, duration? })`
   and `removeToast(id)` available from `uiStore`. `Toast` interface has `id`, `type`, `title`,
   `description?`, `duration?` fields. (Implemented in S3.5 — S3.11 regression-guarded.)

2. **AC2 — Toast types:** `success` (green icon), `error` (red icon), `warning` (amber icon),
   `info` (blue icon). All four handled in `ToastItem` component.
   `data-testid="toast-item"` on root. `"use client"` directive.

3. **AC3 — `ToastContainer`:** Fixed portal in bottom-right corner, max 5 visible, stacked with
   8px gap. Reads `toasts` array from `uiStore`. `data-testid="toast-container"` on root.

4. **AC4 — Default auto-dismiss:** 5s for `success`/`info`, 8s for `warning`, 10s for `error`.
   Timer cleared on unmount. No auto-dismiss when `duration: Infinity`.

5. **AC5 — Close button + hover pause:** Each toast has a close button
   (`data-testid="toast-close"`). `onMouseEnter` pauses auto-dismiss timer;
   `onMouseLeave` resumes it. Close button calls `removeToast` (or `onDismiss/onClose` prop).

6. **AC6 — Entry/exit animations:** Slide in from right (`translate-x-full → translate-x-0`)
   + fade in (`opacity-0 → opacity-100`). Transition duration class present. Exit reverses.

7. **AC7 — `useToast()` hook:** Exported from `packages/ui`. `"use client"` directive.
   Returns `{ success(title, opts?), error(title, opts?), warning(title, opts?), info(title, opts?) }`
   shorthand methods. Wraps `useUIStore.addToast`.

8. **AC8 — `/dev/toasts` demo page:** `"use client"` page at
   `apps/client/app/[locale]/dev/toasts/page.tsx`. Imports `useToast` from `@eusolicit/ui`.
   Has buttons to trigger all four toast types.

9. **AC9 — Exports:** `ToastItem`, `ToastItemProps`, `ToastContainer`, `useToast` exported from
   `packages/ui/index.ts` under `// New in S3.11` block. `ToastItem` and `ToastContainer`
   re-exported from `feedback/index.ts` barrel.

10. **AC10 — Build & type-check:** `pnpm build` and `pnpm type-check` exit 0. (Validated by CI;
    not in Vitest scope.)

---

## Preflight & Context (Step 1)

### Stack Detection

**Detected stack:** `frontend`
**Rationale:** Project has `package.json` with React/Next.js dependencies, `playwright.config.ts`
and `vitest.config.ts`. No backend indicators (`pyproject.toml`, `go.mod`, etc.) at project root.

### Config Flags

- `tea_use_playwright_utils`: not configured (treat as disabled)
- `tea_use_pactjs_utils`: not configured (treat as disabled)
- `tea_browser_automation`: not configured (treat as none)
- `test_stack_type`: not configured (auto-detected as `frontend`)

### Prerequisites

- ✅ Story found in epic planning artifact with clear acceptance criteria (8 functional ACs)
- ✅ Test framework configured: `apps/client/vitest.config.ts` (Vitest `environment: 'node'`)
- ✅ Development environment available (Node.js v22 via nvm)
- ✅ Epic test design loaded: `test-design-epic-03.md`
- ✅ AC1 prerequisite (uiStore Toast state from S3.5) verified present in `ui-store.ts`
- ⚠️ Story file (`3-11-toast-notification-system.md`) not yet created — story content sourced from
  epic planning artifact. Story is in **backlog** state.

### Coverage Strategy from Epic Test Design

From `test-design-epic-03.md`, Story 3.11 maps to:

| Epic Test ID | Story AC | Priority | Test Type |
|-------------|---------|----------|-----------|
| **E03-P1-014** | AC2+AC3+AC4+AC5 (ToastItem + ToastContainer) | **P1** | E2E + Static |
| **E03-P2-017** | AC5 (hover pause + close button) | **P2** | E2E |
| — | AC1 (uiStore guard) | **P1** | Static/Unit |
| — | AC7 (useToast hook) | **P1** | Static/Unit |
| — | AC8 (demo page) | **P2** | Static |
| — | AC9 (exports) | **P1** | Static |
| **E03-P0-001** | AC10 (build) | **P0** | Build/CI |

---

## Generation Mode (Step 2)

**Mode chosen:** AI Generation (static/file-system)

**Rationale:** All acceptance criteria for S3.11 are structural (file presence, component source
content, hook exports, animation class names, timer constant values). The project's established
ATDD pattern (matching `loading-states-s3-10.test.ts` and `wizard-s3-9.test.ts`) uses Vitest with
`environment: 'node'` to verify file existence and source content directly — no browser recording
needed. AC1 is a regression guard (already implemented). This is the fastest and most
deterministic approach for this story's ACs.

**No recording session required.**

---

## Test Strategy (Step 3)

### AC-to-Test Mapping

| AC | Scenarios | Test Level | Priority | Approach |
|----|-----------|------------|----------|---------|
| AC1 | 6 uiStore regression guards (Toast type shape, addToast/removeToast, duration field) | Static/Unit | P1 | File read + string assertions |
| AC2 | 1 file existence + 12 content assertions (use client, type variants, colour classes, icons, testid, props) | Static/Unit | P1 | File read + string/regex assertions |
| AC3 | 1 file existence + 9 content assertions (export, use client, fixed, bottom, right, max-5, gap, testid, ToastItem render) | Static/Unit | P1 | File read + string/regex assertions |
| AC4 | 6 content assertions (5000, 8000, 10000, Infinity, setTimeout/setInterval, clearTimeout/clearInterval) | Static/Unit | P1 | File read + string/regex assertions |
| AC5 | 5 content assertions (close button, testid, removeToast/onDismiss, onMouseEnter, onMouseLeave) | Static/Unit | P1 | File read + regex assertions |
| AC6 | 4 content assertions (transition/animate, opacity-0/100, translate-x, duration class) | Static/Unit | P2 | File read + regex assertions |
| AC7 | 1 file existence + 7 content assertions (export, use client, 4 type shorthands, addToast reference) | Static/Unit | P1 | File read + string assertions |
| AC8 | 1 file existence + 6 content assertions (use client, useToast import, 4 type triggers) | Static/Unit | P2 | File read + string assertions |
| AC9 | 5 packages/ui/index.ts assertions + 2 feedback/index.ts barrel assertions | Static/Unit | P1 | File read + string assertions |
| AC10 | Not in Vitest scope — validated by CI (`pnpm build`, `pnpm type-check`) | Build/CI | P0 | Run commands |

### Red Phase Requirements

- All tests use `it()` (live test, not skipped)
- Tests fail before implementation due to `ENOENT` (missing files) or assertion failures on
  existing files that lack new content
- Tests fail due to **missing implementation**, not test bugs
- AC1 tests pass immediately (prerequisite already implemented in S3.5) — this is intentional

---

## Failing Tests Created (RED Phase)

### Static/Unit Tests (66 tests)

**File:** `eusolicit-app/frontend/apps/client/__tests__/toast-s3-11.test.ts`

**Run command:**
```bash
cd eusolicit-app/frontend
source ~/.nvm/nvm.sh
./apps/client/node_modules/.bin/vitest run apps/client/__tests__/toast-s3-11.test.ts
```

**Test result before implementation:** 🔴 60 FAILING | ✅ 6 PASSING

#### AC1 — uiStore Toast integration guard (6 tests)

- ✅ **Describe: AC1: uiStore.ts — Toast type and addToast regression guard** (6 tests)
  - **Status:** GREEN (✅ — AC1 was implemented in S3.5; these are regression guards)
  - **Tests:** File exists, `export interface Toast`, all 4 type variants, `addToast`, `removeToast`, `duration` field

#### AC2 — ToastItem component (13 tests, 1 file existence + 12 content)

- ✅ **Describe: AC2: ToastItem — file exists** (1 test)
  - **Status:** RED — `expected false to be true` (file doesn't exist)
  - **Verifies:** Task: create `packages/ui/src/components/feedback/ToastItem.tsx`

- ✅ **Describe: AC2: ToastItem — structure and type variants** (12 tests)
  - **Status:** RED — `ENOENT: ToastItem.tsx`
  - **Verifies:** `export function ToastItem`, `"use client"`, type/title/description props,
    `success`+green class, `error`+red class, `warning`+amber class, `info`+blue class,
    lucide-react import, `data-testid="toast-item"`, `ToastItemProps` type export

#### AC3 — ToastContainer component (10 tests, 1 file existence + 9 content)

- ✅ **Describe: AC3: ToastContainer — file exists** (1 test)
  - **Status:** RED — `expected false to be true` (file doesn't exist)
  - **Verifies:** Task: create `packages/ui/src/components/feedback/ToastContainer.tsx`

- ✅ **Describe: AC3: ToastContainer — fixed portal, bottom-right, max-5, gap** (9 tests)
  - **Status:** RED — `ENOENT: ToastContainer.tsx`
  - **Verifies:** `export function ToastContainer`, `"use client"`, `fixed`, `bottom-*` class,
    `right-*` class, max-5 limit (slice/constant), `gap-2`/`space-y-2`, `data-testid="toast-container"`,
    renders `ToastItem`

#### AC4 — Auto-dismiss timings (6 tests)

- ✅ **Describe: AC4: ToastItem — auto-dismiss duration constants** (6 tests)
  - **Status:** RED — `ENOENT: ToastItem.tsx`
  - **Verifies:** `5000` (success/info), `8000` (warning), `10000` (error), `Infinity` handling,
    `setTimeout`/`setInterval` usage, `clearTimeout`/`clearInterval` on unmount

#### AC5 — Close button + hover pause (5 tests)

- ✅ **Describe: AC5: ToastItem — close button and hover-pause behaviour** (5 tests)
  - **Status:** RED — `ENOENT: ToastItem.tsx`
  - **Verifies:** Close button element, `data-testid="toast-close"`, `removeToast`/`onDismiss`/`onClose` call,
    `onMouseEnter` handler (pause), `onMouseLeave` handler (resume)

#### AC6 — Entry/exit animations (4 tests)

- ✅ **Describe: AC6: ToastItem — slide-in/out and fade animations** (4 tests)
  - **Status:** RED — `ENOENT: ToastItem.tsx`
  - **Verifies:** `transition`/`animate-` class, `opacity-0`/`opacity-100`, `translate-x-full`/`translate-x-0`,
    `duration-*` timing class

#### AC7 — useToast hook (8 tests, 1 file existence + 7 content)

- ✅ **Describe: AC7: useToast hook — file exists** (1 test)
  - **Status:** RED — `expected false to be true` (file doesn't exist)
  - **Verifies:** Task: create `packages/ui/src/lib/hooks/useToast.ts`

- ✅ **Describe: AC7: useToast — shorthand methods and export** (7 tests)
  - **Status:** RED — `ENOENT: useToast.ts`
  - **Verifies:** `export function useToast`, `success` shorthand, `error` shorthand,
    `warning` shorthand, `info` shorthand, `addToast` reference (wraps uiStore), `"use client"`

#### AC8 — /dev/toasts demo page (7 tests, 1 file existence + 6 content)

- ✅ **Describe: AC8: /dev/toasts demo page — file exists** (1 test)
  - **Status:** RED — `expected false to be true` (file doesn't exist)
  - **Verifies:** Task: create `apps/client/app/[locale]/dev/toasts/page.tsx`

- ✅ **Describe: AC8: /dev/toasts demo page — structure and toast triggers** (6 tests)
  - **Status:** RED — `ENOENT: page.tsx`
  - **Verifies:** `"use client"`, `useToast` import from `@eusolicit/ui`, `success` trigger,
    `error` trigger, `warning` trigger, `info` trigger

#### AC9 — Exports (7 tests)

- ✅ **Describe: AC9: packages/ui/index.ts — exports all new S3.11 symbols** (5 tests)
  - **Status:** RED — missing `S3.11`, `ToastItem`, `ToastItemProps`, `ToastContainer`, `useToast`
  - **Verifies:** `// New in S3.11` comment, `ToastItem`, `ToastItemProps`, `ToastContainer`, `useToast`

- ✅ **Describe: AC9: packages/ui/src/components/feedback/index.ts — barrel re-exports** (2 tests)
  - **Status:** RED — content assertions fail (`ToastItem`, `ToastContainer` not yet in barrel)
  - **Verifies:** feedback barrel re-exports `ToastItem` and `ToastContainer`

---

## Test Result Summary

| Phase | Status | Tests | Notes |
|-------|--------|-------|-------|
| **RED (current)** | 🔴 FAILING | 60 failing, 6 passing | Expected — implementation does not exist |
| **GREEN (after implementation)** | Expected ✅ PASSING | All 66 | Remove nothing — tests stay as-is |

**TDD Red Phase Compliance:**
- ✅ Tests use `it()` (not `it.skip()`)
- ✅ Tests fail due to missing implementation (ENOENT or assertion failures), not test bugs
- ✅ AC1 guard tests pass intentionally (S3.5 implementation present)
- ✅ All assertions verify expected behaviour, not placeholders

---

## Files to Create (Implementation Guide)

| File | AC | Notes |
|------|----|-------|
| `packages/ui/src/components/feedback/ToastItem.tsx` | AC2, AC4, AC5, AC6 | `"use client"`, type variants, colour classes, timer, close, hover, animations |
| `packages/ui/src/components/feedback/ToastContainer.tsx` | AC3 | `"use client"`, fixed bottom-right portal, max 5, gap-2, reads uiStore.toasts |
| `packages/ui/src/lib/hooks/useToast.ts` | AC7 | `"use client"`, wraps `useUIStore.addToast`, 4 shorthand methods |
| `apps/client/app/[locale]/dev/toasts/page.tsx` | AC8 | `"use client"`, 4 type trigger buttons |
| `packages/ui/index.ts` | AC9 | Add `// New in S3.11` block with ToastItem, ToastItemProps, ToastContainer, useToast |
| `packages/ui/src/components/feedback/index.ts` | AC9 | Add ToastItem, ToastContainer barrel re-exports |

**Key implementation constraints from ACs:**
- `ToastItem`: default durations: `success`/`info` → 5000ms, `warning` → 8000ms, `error` → 10000ms
- `ToastItem`: `Infinity` duration = no auto-dismiss (skip timer setup)
- `ToastItem`: hover (`onMouseEnter`) pauses timer; `onMouseLeave` resumes
- `ToastItem`: animation must include `translate-x-full`/`translate-x-0` AND `opacity-0`/`opacity-100`
- `ToastContainer`: `.slice(-5)` or equivalent to show max 5 toasts
- `useToast`: exports `{ success, error, warning, info }` methods from a `useToast()` call
- All new components exported from `packages/ui/index.ts` under `// New in S3.11`

---

## Next Steps (TDD Green Phase)

After implementing Story 3.11:

1. Run tests: `cd eusolicit-app/frontend && ./apps/client/node_modules/.bin/vitest run apps/client/__tests__/toast-s3-11.test.ts`
2. Verify all 66 tests **PASS** (green phase)
3. If any tests fail, fix the implementation (not the tests)
4. Run full test suite to confirm no regressions: `pnpm test --filter client`
5. Verify build: `pnpm build`
6. Commit passing tests with implementation

---

## Appendix

### Related Test Scenarios from Epic Test Design

The epic test design (`test-design-epic-03.md`) contains two E2E-level scenarios for this story
that are NOT covered by the static tests and require browser-based Playwright tests. These are
out of scope for S3.11 ATDD (static phase) but should be implemented during the `automate`
workflow phase:

- **E03-P1-014**: Each type (success/error/warning/info) renders with correct icon and colour;
  auto-dismisses after configured duration; close button works
  → Playwright: navigate to `/dev/toasts`; trigger each; assert toast visible; assert disappears
- **E03-P2-017**: Hover on toast pauses auto-dismiss timer; close button dismisses immediately
  → Playwright: trigger toast; hover; assert still visible after timeout; move away → assert dismisses

### Implementation Notes (from Epic Planning Artifact)

> Build on top of shadcn `Toast` / `Sonner` integration (shadcn ships a `sonner` toast by
> default). Customise appearance to match the design system. Wire `uiStore.toasts[]` as the
> backing state or use Sonner's internal state. Export `useToast` from `packages/ui`.

**Dependency note**: Neither `sonner` nor the shadcn `toast` component (`@radix-ui/react-toast`)
is currently installed in `packages/ui`. If Sonner is chosen, add `sonner` to
`packages/ui/package.json` before implementing.

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
