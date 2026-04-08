# Story 3.11 — Toast Notification System

**Epic:** 3 — Frontend Shell & Design System
**Status:** ✅ Done
**Implemented:** 2026-04-08

---

## User Story

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
   shorthand methods. Wraps `useToastStore.addToast`.

8. **AC8 — `/dev/toasts` demo page:** `"use client"` page at
   `apps/client/app/[locale]/dev/toasts/page.tsx`. Imports `useToast` from `@eusolicit/ui`.
   Has buttons to trigger all four toast types.

9. **AC9 — Exports:** `ToastItem`, `ToastItemProps`, `ToastContainer`, `useToast` exported from
   `packages/ui/index.ts` under `// New in S3.11` block. `ToastItem` and `ToastContainer`
   re-exported from `feedback/index.ts` barrel.

10. **AC10 — Build & type-check:** `pnpm build` and `pnpm type-check` exit 0. (Validated by CI.)

---

## Implementation Notes

### Architecture

This story implements a **self-contained toast notification system** within `packages/ui`:

- **`packages/ui/src/lib/stores/toast-store.ts`** — Standalone Zustand store that manages
  `toasts[]`, `addToast`, and `removeToast`. Independent of `apps/client/lib/stores/ui-store.ts`
  to keep `packages/ui` free of app-level dependencies. The `apps/client` `uiStore` retains its
  toast interface as a regression guard (AC1).

- **`packages/ui/src/components/feedback/ToastItem.tsx`** — Individual toast component. Uses
  `useEffect` with `setTimeout` for auto-dismiss, `useRef` to track remaining time for
  pause-on-hover. Implements entry (slide+fade in) and exit (reverse) animations via Tailwind
  transition classes toggled on a `visible` state flag.

- **`packages/ui/src/components/feedback/ToastContainer.tsx`** — Portal container that subscribes
  to `useToastStore`, slices to max 5, renders a stack of `ToastItem`s in the bottom-right corner.

- **`packages/ui/src/lib/hooks/useToast.ts`** — Convenience hook exposing `success`, `error`,
  `warning`, `info` shorthand methods backed by `useToastStore.addToast`.

### Default Durations

| Type    | Duration |
|---------|----------|
| success | 5 000 ms |
| info    | 5 000 ms |
| warning | 8 000 ms |
| error   | 10 000 ms |
| any     | Infinity → no auto-dismiss |

### Files Created / Modified

| File | Action |
|------|--------|
| `packages/ui/src/lib/stores/toast-store.ts` | Created |
| `packages/ui/src/components/feedback/ToastItem.tsx` | Created |
| `packages/ui/src/components/feedback/ToastContainer.tsx` | Created |
| `packages/ui/src/lib/hooks/useToast.ts` | Created |
| `apps/client/app/[locale]/dev/toasts/page.tsx` | Created |
| `packages/ui/index.ts` | Updated — added `// New in S3.11` block |
| `packages/ui/src/components/feedback/index.ts` | Updated — added ToastItem/ToastContainer barrel exports |

---

## Test Results

**ATDD test file:** `apps/client/__tests__/toast-s3-11.test.ts`
**Result:** ✅ 66 / 66 PASSING (full suite: 354 / 354 passing — no regressions)

```
Test Files  8 passed (8)
Tests      354 passed (354)
```
