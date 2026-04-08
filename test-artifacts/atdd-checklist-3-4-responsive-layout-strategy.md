---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-08'
workflowType: atdd
storyId: 3-4-responsive-layout-strategy
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/3-4-responsive-layout-strategy.md
  - eusolicit-docs/test-artifacts/test-design-epic-03.md
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/e2e/specs/shell/app-shell-layout.spec.ts
  - eusolicit-app/e2e/specs/shell/app-shell-layout.admin.spec.ts
  - eusolicit-docs/test-artifacts/atdd-checklist-3-3-app-shell-layout-sidebar-top-bar-content-area.md
---

# ATDD Checklist: Story 3.4 — Responsive Layout Strategy

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated; feature not yet implemented
**Story:** `eusolicit-docs/implementation-artifacts/3-4-responsive-layout-strategy.md`
**Epic:** E03 — Frontend Shell & Design System

---

## Step 1: Preflight & Context

| Item | Status | Notes |
|------|--------|-------|
| Story has clear acceptance criteria | ✅ | 9 ACs; explicit `data-testid` specs in Dev Notes |
| Playwright config found | ✅ | `eusolicit-app/playwright.config.ts` |
| Detected stack | ✅ | `frontend` — Next.js 14 App Router, Playwright E2E |
| Test artifacts directory | ✅ | `eusolicit-docs/test-artifacts/` |
| Epic test design loaded | ✅ | `test-design-epic-03.md` — E03-P1-011, E03-P2-006, E03-P2-007 |
| Prior story ATDD loaded | ✅ | S3.3 checklist; `app-shell-layout.spec.ts` (GREEN) reviewed for patterns |

---

## Step 2: Generation Mode

**Mode:** AI Generation (no browser recording — target UI not implemented yet)
**Reason:** `BottomNav`, `MobileSidebarSheet`, `useBreakpoint`, and the responsive AppShell/TopBar changes
do not exist. Browser automation cannot capture selectors from non-existent components.
All selectors derived from the story's explicit `data-testid` requirements (Dev Notes) and AC specifications.

---

## Step 3: Test Strategy

### AC → Test Mapping

| AC | Requirement Summary | Test Level | Priority | Epic Test ID | Describe Block |
|----|---------------------|-----------|----------|--------------|----------------|
| AC1 | Desktop (≥1280px): full sidebar visible; toggle works; uiStore respected | E2E | P1 | E03-P1-011 | `[P1] AC1 — Desktop Layout (≥1280px)` |
| AC2 | Tablet (768–1279px): auto-collapses on entry; user expands; auto-collapses on route change | E2E | P1 | E03-P1-011, E03-P2-007 | `[P1] AC2 — Tablet Auto-Collapse` |
| AC3 | Mobile (<768px): sidebar hidden; BottomNav shows first 5 items | E2E | P1 | E03-P1-011 | `[P1] AC3, AC4, AC5 — Mobile Bottom Navigation` |
| AC4 | BottomNav: fixed, h-16, safe-area-aware, bg-white, border-t | E2E | P1 | — | `[P1] AC3, AC4, AC5 — Mobile Bottom Navigation` |
| AC5 | BottomNav: active indigo-600, inactive slate-500 | E2E | P1 | — | `[P1] AC3, AC4, AC5 — Mobile Bottom Navigation` |
| AC6 | Hamburger in TopBar opens Sheet overlay; closes on route change | E2E | P2 | E03-P2-006 | `[P2] AC6 — Mobile Sidebar Sheet via Hamburger` |
| AC7 | ml-0 on mobile; no reflow; no hydration errors | E2E | P2 | E03-R-004, E03-R-005 | `[P2] AC7 — Layout Margins & SSR Safety` |
| AC8 | Both client and admin apps implement responsive behaviour | E2E (admin) | P1/P2 | E03-P1-011 | `[P1] AC8 — Admin App Responsive Layout` |
| AC9 | `pnpm build` exits 0 for both apps | CI gate | P0 | E03-P0-001 | _(existing CI gate — no new test needed)_ |

**Note on AC9:** `E03-P0-001` (`pnpm build` gate) is already an active CI test established in Story 3.1.
No new test file is generated for AC9. Story 3.4 must not break the build.

**Note on AC7 (safe-area-inset):** `env(safe-area-inset-bottom)` evaluates to `0px` in Playwright's default
Chrome viewport (no physical notch simulation). The test for AC4 verifies the `paddingBottom` CSS style
attribute presence via bottom edge proximity rather than exact safe-area value.

---

## Step 4: Generated Test Files (TDD RED PHASE)

### 🔴 File 1: `eusolicit-app/e2e/specs/shell/responsive-layout.spec.ts`

**Playwright projects:** `client-chromium`, `client-firefox`, `client-webkit`, `client-mobile`
**baseURL:** `http://localhost:3000`
**Note:** Each test calls `page.setViewportSize()` explicitly, overriding the project default viewport.
This ensures breakpoint assertions are deterministic across all four projects.

| Test ID | Describe Block | Priority | AC | Epic ID | Failure Reason |
|---------|----------------|----------|-----|---------|----------------|
| T01 | `[P1] AC1 — Desktop Layout` | P1 | AC1 | E03-P1-011 | `useBreakpoint` missing → layout never receives `isMobile`/`isTablet` → responsive logic absent |
| T02 | `[P1] AC1 — Desktop Layout` | P1 | AC1 | — | `sidebarCollapsed` from localStorage not fed into AppShell at desktop (layout not updated) |
| T03 | `[P1] AC1 — Desktop Layout` | P1 | AC6 | — | `hamburger-menu` testid doesn't exist; TopBar has no `onOpenMobileSidebar` prop yet |
| T04 | `[P1] AC2 — Tablet Auto-Collapse` | P1 | AC2 | E03-P1-011 | `useEffect([isTablet, mounted])` not in layout → sidebar never auto-collapses at 900px |
| T05 | `[P1] AC2 — Tablet Auto-Collapse` | P1 | AC2 | — | Auto-collapse precondition fails (T04 fails) → expand test unreachable |
| T06 | `[P1] AC2 — Tablet Auto-Collapse` | P1 | AC2 | E03-P2-007 | `useEffect([pathname])` not in layout → sidebar doesn't re-collapse on route change |
| T07 | `[P1] AC3, AC4, AC5 — Mobile BottomNav` | P1 | AC3 | E03-P1-011 | `bottom-nav` testid doesn't exist; `AppShell` has no `isMobile`/`bottomNav` props → sidebar still visible |
| T08 | `[P1] AC3, AC4, AC5 — Mobile BottomNav` | P1 | AC3 | E03-P1-011 | `bottom-nav-item-{label}` testids don't exist (BottomNav component missing) |
| T09 | `[P1] AC3, AC4, AC5 — Mobile BottomNav` | P1 | AC4 | — | `bottom-nav` testid doesn't exist → CSS position/bg-color checks fail |
| T10 | `[P1] AC3, AC4, AC5 — Mobile BottomNav` | P1 | AC4 | — | `bottom-nav` testid doesn't exist → inner height assertion fails |
| T11 | `[P1] AC3, AC4, AC5 — Mobile BottomNav` | P1 | AC4 | — | `bottom-nav` testid doesn't exist → bottom-of-viewport anchoring check fails |
| T12 | `[P1] AC3, AC4, AC5 — Mobile BottomNav` | P1 | AC5 | — | `bottom-nav-item-dashboard` testid doesn't exist → active colour check fails |
| T13 | `[P1] AC3, AC4, AC5 — Mobile BottomNav` | P1 | AC5 | — | `bottom-nav-item-tenders` testid doesn't exist → inactive colour check fails |
| T14 | `[P1] AC3, AC4, AC5 — Mobile BottomNav` | P1 | AC4, AC7 | — | `AppShell` has no `pb-16` on `<main>` when `isMobile && bottomNav` → padding check fails |
| T15 | `[P2] AC6 — Mobile Sidebar Sheet` | P2 | AC6 | E03-P2-006 | `hamburger-menu` testid doesn't exist (TopBar missing `onOpenMobileSidebar` prop + button) |
| T16 | `[P2] AC6 — Mobile Sidebar Sheet` | P2 | AC6 | E03-P2-006 | `mobile-sidebar-sheet` testid doesn't exist (MobileSidebarSheet component missing) |
| T17 | `[P2] AC6 — Mobile Sidebar Sheet` | P2 | AC6 | E03-P2-006 | `mobile-sidebar-sheet` doesn't exist → `nav-item-*` inside sheet unreachable |
| T18 | `[P2] AC6 — Mobile Sidebar Sheet` | P2 | AC6 | — | `mobile-sidebar-sheet` missing → route-change close assertion fails |
| T19 | `[P2] AC7 — Layout Margins` | P2 | AC7 | — | `AppShell` has no `isMobile` support → right column still has `ml-64` or `ml-16` on mobile |
| T20 | `[P2] AC7 — Layout Margins` | P2 | AC7 | E03-R-004 | Responsive rendering without `mounted` gate may produce hydration errors |

**Total client tests:** 20 (all with `test.skip()` — TDD RED phase)

---

### 🔴 File 2: `eusolicit-app/e2e/specs/shell/responsive-layout.admin.spec.ts`

**Playwright project:** `admin-chromium` ONLY
**baseURL:** `http://localhost:3001`

| Test ID | Describe Block | Priority | AC | Epic ID | Failure Reason |
|---------|----------------|----------|-----|---------|----------------|
| A01 | `[P1] AC8 — Admin Responsive Layout` | P1 | AC8 | E03-P1-011 | Admin `(protected)/layout.tsx` not updated → no responsive logic in admin |
| A02 | `[P1] AC8 — Admin Responsive Layout` | P1 | AC8 | E03-P1-011 | Admin sidebar never auto-collapses at tablet (layout not updated) |
| A03 | `[P1] AC8 — Admin Responsive Layout` | P1 | AC8 | E03-P1-011 | Admin `bottom-nav` testid missing; admin sidebar visible on mobile |
| A04 | `[P1] AC8 — Admin Responsive Layout` | P1 | AC8 | — | Admin `bottom-nav-item-{label}` testids don't exist (BottomNav not wired in admin) |
| A05 | `[P1] AC8 — Admin Responsive Layout` | P2 | AC8 | — | Admin `hamburger-menu` and `mobile-sidebar-sheet` don't exist |
| A06 | `[P1] AC8 — Admin Responsive Layout` | P1 | AC8 | E03-P2-007 | Admin sidebar doesn't auto-collapse on route change (layout not updated) |

**Total admin tests:** 6 (all with `test.skip()` — TDD RED phase)

---

## Step 4C: Aggregate Summary

| Metric | Value |
|--------|-------|
| TDD Phase | 🔴 RED |
| Total tests generated | 26 |
| Client E2E tests | 20 (in `responsive-layout.spec.ts`) |
| Admin E2E tests | 6 (in `responsive-layout.admin.spec.ts`) |
| API tests | 0 (not applicable — no backend endpoints in this story) |
| All tests use `test.skip()` | ✅ |
| All tests assert expected behavior (not placeholders) | ✅ |
| Placeholder assertions (`expect(true).toBe(true)`) | 0 |

### Priority Distribution

| Priority | Client | Admin | Total |
|----------|--------|-------|-------|
| P0 | 0 | 0 | 0 |
| P1 | 14 | 5 | 19 |
| P2 | 6 | 1 | 7 |
| P3 | 0 | 0 | 0 |
| **Total** | **20** | **6** | **26** |

**Note on P0:** AC9 (`pnpm build`) is P0 and covered by the existing CI gate `E03-P0-001`. No new P0
test file is generated for Story 3.4. The build gate runs automatically for every PR.

### AC Coverage

| AC | Tests Covering | Files |
|----|----------------|-------|
| AC1 | T01, T02, T03 | client |
| AC2 | T04, T05, T06 | client |
| AC3 | T07, T08, A03, A04 | client, admin |
| AC4 | T09, T10, T11, T14 | client |
| AC5 | T12, T13 | client |
| AC6 | T15, T16, T17, T18, A05 | client, admin |
| AC7 | T19, T20 | client |
| AC8 | A01, A02, A03, A04, A05, A06 | admin |
| AC9 | _(existing E03-P0-001 CI gate)_ | CI |

**All 9 ACs have at least 1 test. AC9 is covered by the existing CI build gate.**

---

## Step 5: Validation

### Validation Checklist

- [x] Story has approved acceptance criteria (9 ACs, all well-specified)
- [x] All 9 ACs have test coverage (AC9 via existing CI gate; AC1–AC8 via new tests)
- [x] All tests use `test.skip()` (TDD red phase — documented intentional failure)
- [x] No placeholder assertions (`expect(true).toBe(true)`)
- [x] Tests assert EXPECTED behavior (behavior that will work once implemented)
- [x] Selectors follow hierarchy: `data-testid` primary, `getByRole` secondary, `getByText` tertiary
- [x] No CSS class selectors (brittle), no arbitrary `nth()` indexes
- [x] No orphaned browser sessions (AI generation mode — no Playwright CLI recording needed)
- [x] Test files saved to `eusolicit-app/e2e/specs/shell/` (correct location, consistent with S3.3)
- [x] Checklist saved to `eusolicit-docs/test-artifacts/` (correct path)
- [x] Client spec uses `responsive-layout.spec.ts` (no `.admin.` suffix) → runs in client-chromium, Firefox, WebKit, Mobile
- [x] Admin spec uses `responsive-layout.admin.spec.ts` → runs in admin-chromium only (correct project isolation)
- [x] Failure reasons are accurate (components/logic not yet implemented)
- [x] Epic test IDs cross-referenced correctly (E03-P1-011, E03-P2-006, E03-P2-007)
- [x] Viewport set via `page.setViewportSize()` in each test (deterministic across projects)
- [x] `waitForTimeout(400)` buffer after navigation for `useBreakpoint` + `mounted` flag to settle

### Selector Strategy Applied

All selectors use `data-testid` attributes explicitly defined in the story dev notes:

| Selector | Component | Task |
|----------|-----------|------|
| `data-testid="bottom-nav"` | `BottomNav.tsx` root `<div>` | Task 2.1 |
| `data-testid="bottom-nav-item-{label}"` | each `<Link>` in `BottomNav.tsx` | Task 2.2 |
| `data-testid="hamburger-menu"` | hamburger `<button>` in `TopBar.tsx` | Task 5.3 |
| `data-testid="mobile-sidebar-sheet"` | `SheetContent` in `MobileSidebarSheet.tsx` | Task 3.1 |
| `data-testid="sidebar"` | existing — `Sidebar.tsx` `<aside>` | Story 3.3 Task 4.1 |
| `data-testid="sidebar-toggle"` | existing — toggle button in `Sidebar.tsx` | Story 3.3 Task 4.2 |
| `data-testid="content-area"` | existing — `AppShell.tsx` `<main>` | Story 3.3 Task 9.2 |
| `data-testid="nav-item-{label}"` | existing — each `NavItem` | Story 3.3 Task 3.1 |

### Risk Coverage

| Risk ID | Score | How Mitigated in Tests |
|---------|-------|------------------------|
| E03-R-005 (responsive regression) | 4 | T01, T04, T07, A01–A03: `page.setViewportSize()` for each breakpoint tier; assert correct component rendered |
| E03-R-004 (Zustand SSR hydration) | 4 | T02: pre-seed localStorage, verify collapsed state respected; T19: captures console hydration errors |

---

## Next Steps (TDD Green Phase)

After implementing Story 3.4 (all 9 tasks):

1. **Remove `test.skip()`** from each test in:
   - `eusolicit-app/e2e/specs/shell/responsive-layout.spec.ts` (20 tests)
   - `eusolicit-app/e2e/specs/shell/responsive-layout.admin.spec.ts` (6 tests)

2. **Start dev servers:**
   ```bash
   cd eusolicit-app/frontend
   pnpm dev --filter client   # http://localhost:3000
   pnpm dev --filter admin    # http://localhost:3001
   ```

3. **Run tests (green phase verification):**
   ```bash
   # From eusolicit-app/
   npx playwright test e2e/specs/shell/responsive-layout.spec.ts --project=client-chromium
   npx playwright test e2e/specs/shell/responsive-layout.admin.spec.ts --project=admin-chromium
   ```

4. **Verify ALL 26 tests pass** — if any fail:
   - **Implementation bug:** Fix the component / layout logic
   - **Test bug:** Fix the assertion (update selector or expected value)

5. **Common GREEN-phase adjustments to expect:**
   - T10 (bg-white): verify `rgb(255, 255, 255)` matches actual computed value
   - T11/T12 (colour checks): verify indigo-600 = `rgb(79, 70, 229)`, slate-500 = `rgb(100, 116, 139)` — these are standard Tailwind v3 values; update if project uses a custom colour palette
   - T09/T10 (BottomNav height): inner div measurement via `.querySelector('div')` — adjust selector if DOM structure differs from spec

6. **Commit passing tests** alongside the Story 3.4 implementation

---

## Implementation Guidance

### Components to Create (Red Phase → Green Phase triggers)

```
packages/ui/src/
├── lib/hooks/
│   └── useBreakpoint.ts           ← T01, T04, T07, A01–A03 (all breakpoint tests)
└── components/app-shell/
    ├── BottomNav.tsx               ← T07–T13, A03–A04 (bottom nav tests)
    ├── MobileSidebarSheet.tsx      ← T14–T17, A05 (sheet tests)
    ├── AppShell.tsx                ← T07, T13, T18 (isMobile, bottomNav, pb-16 props)
    └── TopBar.tsx                  ← T03, T14 (onOpenMobileSidebar prop + hamburger button)

apps/client/app/(protected)/
└── layout.tsx                      ← T01–T19 (all client responsive tests)

apps/admin/app/(protected)/
└── layout.tsx                      ← A01–A06 (all admin responsive tests)

packages/ui/index.ts                ← Required for imports in both layout files
```

### Critical Implementation Requirements for Tests to Pass

1. **`useBreakpoint` hook:** Must use `window.matchMedia` (not `resize`); SSR default must be `"desktop"`;
   `useEffect` must call `update()` synchronously on mount. (T01, T04, T07)

2. **`BottomNav` component:** `data-testid="bottom-nav"` on root `<div>`; `data-testid="bottom-nav-item-{label.toLowerCase()}"` on each `<Link>`. (T07–T12)

3. **`AppShell` props:** `isMobile?: boolean` hides sidebar when true (`{!isMobile && sidebar}`);
   `bottomNav?: ReactNode` renders after right column; `pb-16` on `<main>` when `isMobile && bottomNav`. (T07, T13, T18)

4. **`TopBar` prop:** `onOpenMobileSidebar?: () => void` renders `<button data-testid="hamburger-menu" aria-label="Open menu">` as first child when provided. (T14)

5. **`MobileSidebarSheet`:** `data-testid="mobile-sidebar-sheet"` on `SheetContent`. (T15–T17)

6. **Layout `mounted` gate:** `showMobile = isMobile && mounted` prevents hydration mismatch. (T19, T02)

7. **Tablet auto-collapse effects:** `useEffect([isTablet, mounted])` and `useEffect([pathname])` in both layouts. (T04, T06, A02, A06)

---

## Notes & Assumptions

1. **Story 3.3 is fully implemented.** The `app-shell-layout.spec.ts` is in GREEN phase. Story 3.4 builds on the existing `AppShell`, `Sidebar`, `NavItem`, `TopBar`, `uiStore` and `mounted` flag pattern without modifying them beyond the story's specified changes.

2. **No authentication required for S3.4 shell tests.** Tests navigate directly to `/dashboard` (same as S3.3). Auth guards are not implemented until a separate story.

3. **Viewport via `page.setViewportSize()` in each test.** This overrides the Playwright project's device preset for the duration of that test, ensuring breakpoint assertions are identical across `client-chromium`, `client-firefox`, `client-webkit`, and `client-mobile` projects.

4. **`waitForTimeout(400)`** is used after navigation to allow `useBreakpoint`'s `useEffect` and the layout's `useEffect([isTablet, mounted])` to settle. The 400ms covers: `useEffect` scheduling (~16ms), React re-render (~16ms), CSS transition (200ms), and a 150ms CI buffer. This mirrors the 350ms used in S3.3 for the sidebar toggle animation.

5. **Admin nav items assumed:** Dashboard, Companies, Tenders, Users, Reports, Settings (in that order). Tests A04 assert the first 5. If the actual admin nav order differs from the story spec, update the `expectedAdminItems` array in test A04.

6. **BottomNav inner height measurement (T09):** Uses `.querySelector('div')` to select the inner flex container `<div className="flex items-center justify-around h-16 ...">`. If the implementation wraps the inner div differently, update the evaluate() selector.

7. **Colour assertions (T11, T12):** Use Tailwind v3 computed RGB values: `indigo-600 = rgb(79, 70, 229)`, `slate-500 = rgb(100, 116, 139)`. These assume the project uses standard Tailwind v3. Verify against actual computed styles if a custom theme is applied.

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Story:** 3.4 — Responsive Layout Strategy
**Version:** BMad v6 (sequential execution mode)
