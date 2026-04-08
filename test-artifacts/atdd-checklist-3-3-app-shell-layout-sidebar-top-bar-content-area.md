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
storyId: 3-3-app-shell-layout-sidebar-top-bar-content-area
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/3-3-app-shell-layout-sidebar-top-bar-content-area.md
  - eusolicit-docs/test-artifacts/test-design-epic-03.md
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/e2e/support/fixtures/auth.fixture.ts
  - eusolicit-app/e2e/support/fixtures/index.ts
  - eusolicit-app/e2e/specs/smoke/design-system-smoke.spec.ts
  - .claude/skills/bmad-testarch-atdd/resources/knowledge/selector-resilience.md
  - .claude/skills/bmad-testarch-atdd/resources/knowledge/fixture-architecture.md
  - .claude/skills/bmad-testarch-atdd/resources/knowledge/network-first.md
---

# ATDD Checklist: Story 3.3 — App Shell Layout (Sidebar, Top Bar, Content Area)

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated; feature not yet implemented
**Story:** `eusolicit-docs/implementation-artifacts/3-3-app-shell-layout-sidebar-top-bar-content-area.md`
**Epic:** E03 — Frontend Shell & Design System

---

## Step 1: Preflight & Context

| Item | Status | Notes |
|------|--------|-------|
| Story has clear acceptance criteria | ✅ | 11 ACs, all well-specified with data-testid requirements |
| Playwright config found | ✅ | `eusolicit-app/playwright.config.ts` |
| Detected stack | ✅ | `frontend` — Next.js 14 App Router, Playwright E2E |
| Test artifacts directory | ✅ | `eusolicit-docs/test-artifacts/` |
| Epic test design loaded | ✅ | `test-design-epic-03.md` (E03-P0-009, P0-010, P1-010, P2-004, P2-005) |

---

## Step 2: Generation Mode

**Mode:** AI Generation (no browser recording — components not implemented yet)
**Reason:** Target UI (`AppShell`, `Sidebar`, `NavItem`, etc.) does not exist. Browser automation cannot capture selectors from non-existent components. All selectors derived from story's explicit `data-testid` requirements and AC specifications.

---

## Step 3: Test Strategy

### AC → Test Mapping

| AC | Requirement Summary | Test Level | Priority | Epic Test ID | Describe Block |
|----|---------------------|-----------|----------|--------------|----------------|
| AC1 | `<AppShell>` slot props, exported from `@eusolicit/ui` | E2E (structure) | P0 | E03-P0-009 | `[P0] AC1, AC7, AC8` |
| AC2 | NavItem active highlight `bg-indigo-50 text-indigo-600` | E2E | P0 | E03-P0-009 | `[P0] AC2, AC8` |
| AC3 | Sidebar 256px↔64px toggle, 200ms transition, labels hidden when collapsed | E2E | P1 | E03-P1-010 | `[P1] AC3, AC10` |
| AC4 | TopBar sticky h-16, Breadcrumbs left, bell+lang+avatar right | E2E | P0 | E03-P0-009 | `[P0] AC4` |
| AC5 | Avatar dropdown: name, email, Profile, Settings, Separator, Sign out | E2E | P2 | E03-P2-004 | `[P2] AC5` |
| AC6 | Notifications bell + badge (static "3") | E2E | P2 | E03-P2-005 | `[P2] AC6` |
| AC7 | Content scrolls independently; sidebar fixed, topbar sticky | E2E | P1 | — | `[P0] AC1, AC7, AC8` |
| AC8 | Client `(protected)` — Dashboard, Tenders, Offers, Documents, Team, Settings | E2E | P0 | E03-P0-009 | `[P0] AC2, AC8` |
| AC9 | Admin `(protected)` — Dashboard, Companies, Tenders, Users, Reports, Settings | E2E (admin) | P0 | E03-P0-010 | `[P0] AC9 (admin spec)` |
| AC10 | Zustand `uiStore`, localStorage persist, SSR hydration safe | E2E | P1 | E03-P1-010 | `[P1] AC3, AC10` |
| AC11 | `pnpm build` exits 0 both apps | Build/CI | P0 | E03-P0-001 | _(CI gate, no new test needed)_ |

**Note on AC11:** E03-P0-001 (`pnpm build` gate) is already covered by the CI pipeline established in Story 3.1. No new test file is generated — the existing `frontend-monorepo-scaffold.api.spec.ts` and CI build gate cover this. Story 3.3 must pass `pnpm build` after implementation and this is validated by the CI gate.

**Note on API tests:** Story 3.3 creates no backend endpoints. All test coverage is E2E (browser). No API test sub-worker is applicable.

---

## Step 4: Generated Test Files (TDD RED PHASE)

### 🔴 File 1: `eusolicit-app/e2e/specs/shell/app-shell-layout.spec.ts`

**Playwright project:** `client-chromium`, `client-firefox`, `client-webkit`, `client-mobile`
**baseURL:** `http://localhost:3000`

| Test ID | Describe Block | Priority | AC | Epic ID | Failure Reason |
|---------|---------------|----------|-----|---------|----------------|
| T01 | `[P0] AC1, AC7, AC8 — AppShell Layout Structure` | P0 | AC1 | E03-P0-009 | `(protected)/layout.tsx` missing → no AppShell |
| T02 | `[P0] AC1, AC7, AC8 — AppShell Layout Structure` | P0 | AC1 | E03-P0-009 | sidebar/topbar/content-area testids absent |
| T03 | `[P0] AC1, AC7, AC8 — AppShell Layout Structure` | P0 | AC8 | E03-P0-009 | dashboard-page testid not rendered |
| T04 | `[P0] AC1, AC7, AC8 — AppShell Layout Structure` | P1 | AC7 | — | sidebar position:fixed, topbar sticky not implemented |
| T05 | `[P0] AC2, AC8 — Client Sidebar Navigation Items` | P0 | AC8 | E03-P0-009 | 6 nav items not rendered (NavItem + Sidebar missing) |
| T06 | `[P0] AC2, AC8 — Client Sidebar Navigation Items` | P0 | AC2 | E03-P0-009 | Active indigo-50 highlight not implemented |
| T07 | `[P0] AC2, AC8 — Client Sidebar Navigation Items` | P1 | AC2 | E03-P0-009 | Inactive state check (fails if shell not rendered) |
| T08 | `[P0] AC2, AC8 — Client Sidebar Navigation Items` | P0 | AC2 | E03-P0-009 | Nav item href attributes not rendered |
| T09 | `[P0] AC2, AC8 — Client Sidebar Navigation Items` | P1 | AC3 | — | Labels not visible (shell not rendered) |
| T10 | `[P1] AC3, AC10 — Sidebar Toggle & State Persistence` | P1 | AC3 | E03-P1-010 | sidebar-toggle button doesn't exist |
| T11 | `[P1] AC3, AC10 — Sidebar Toggle & State Persistence` | P1 | AC3 | E03-P1-010 | Full width (~256px) assert fails (sidebar absent) |
| T12 | `[P1] AC3, AC10 — Sidebar Toggle & State Persistence` | P1 | AC3 | E03-P1-010 | Collapse to ~64px fails (sidebar-toggle absent) |
| T13 | `[P1] AC3, AC10 — Sidebar Toggle & State Persistence` | P1 | AC3 | E03-P1-010 | Labels not hidden (collapsed state not implemented) |
| T14 | `[P1] AC3, AC10 — Sidebar Toggle & State Persistence` | P1 | AC3 | E03-P1-010 | Expand back to full width fails |
| T15 | `[P1] AC3, AC10 — Sidebar Toggle & State Persistence` | P1 | AC10 | E03-P1-010 | localStorage key absent (uiStore not created) |
| T16 | `[P1] AC3, AC10 — Sidebar Toggle & State Persistence` | P1 | AC10 | E03-P1-010 | State not restored after reload (no localStorage) |
| T17 | `[P1] AC3, AC10 — Sidebar Toggle & State Persistence` | P1 | AC10 | E03-P1-010 | Hydration errors may occur (mounted flag pattern not implemented) |
| T18 | `[P0] AC4 — Top Bar Structure` | P0 | AC4 | E03-P0-009 | topbar testid absent |
| T19 | `[P0] AC4 — Top Bar Structure` | P0 | AC4 | E03-P0-009 | TopBar height/position not implemented |
| T20 | `[P0] AC4 — Top Bar Structure` | P0 | AC4 | E03-P0-009 | user-avatar-menu-trigger absent |
| T21 | `[P0] AC4 — Top Bar Structure` | P0 | AC4 | E03-P0-009 | notifications-bell absent |
| T22 | `[P0] AC4 — Top Bar Structure` | P1 | AC4 | — | language-selector placeholder absent |
| T23 | `[P2] AC5 — User Avatar Dropdown Menu` | P2 | AC5 | E03-P2-004 | DropdownMenu not rendered (UserAvatarMenu missing) |
| T24 | `[P2] AC5 — User Avatar Dropdown Menu` | P2 | AC5 | E03-P2-004 | Menu label text absent |
| T25 | `[P2] AC5 — User Avatar Dropdown Menu` | P2 | AC5 | — | Dismiss behavior (menu doesn't exist) |
| T26 | `[P2] AC6 — Notifications Bell` | P2 | AC6 | E03-P2-005 | notifications-bell testid absent |
| T27 | `[P2] AC6 — Notifications Bell` | P2 | AC6 | E03-P2-005 | notifications-badge testid absent |
| T28 | `[P2] AC6 — Notifications Bell` | P2 | AC6 | E03-P2-005 | Badge count not "3" (component missing) |
| T29 | `[P2] AC6 — Notifications Bell` | P2 | AC6 | — | Badge overlay position (component missing) |

**Total client tests:** 29 (all with `test.skip()` — TDD RED phase)

---

### 🔴 File 2: `eusolicit-app/e2e/specs/shell/app-shell-layout.admin.spec.ts`

**Playwright project:** `admin-chromium`
**baseURL:** `http://localhost:3001`

| Test ID | Describe Block | Priority | AC | Epic ID | Failure Reason |
|---------|---------------|----------|-----|---------|----------------|
| A01 | `[P0] AC9 — Admin App Shell Navigation` | P0 | AC9 | E03-P0-010 | `apps/admin/(protected)/layout.tsx` missing |
| A02 | `[P0] AC9 — Admin App Shell Navigation` | P0 | AC9 | E03-P0-010 | 6 admin nav items absent (Companies, Users, Reports) |
| A03 | `[P0] AC9 — Admin App Shell Navigation` | P0 | AC9 | E03-P0-010 | Client nav items (Offers, Documents, Team) isolation |
| A04 | `[P0] AC9 — Admin App Shell Navigation` | P0 | AC9 | E03-P0-010 | Admin-specific nav items absent |
| A05 | `[P0] AC9 — Admin App Shell Navigation` | P0 | AC9 | E03-P0-010 | user-avatar-menu-trigger absent in admin |
| A06 | `[P0] AC9 — Admin App Shell Navigation` | P1 | AC9 | — | admin dashboard-page testid absent |
| A07 | `[P0] AC9 — Admin App Shell Navigation` | P1 | AC1,AC9 | — | sidebar/topbar/content-area absent in admin |
| A08 | `[P1] AC10 — Admin uiStore State Persistence` | P1 | AC10 | — | sidebar-toggle absent in admin |
| A09 | `[P1] AC10 — Admin uiStore State Persistence` | P1 | AC10 | — | `eusolicit-admin-ui-store` key absent in localStorage |
| A10 | `[P1] AC10 — Admin uiStore State Persistence` | P1 | AC10 | — | Separate admin store key isolation |
| A11 | `[P1] AC10 — Admin uiStore State Persistence` | P1 | AC10 | — | Admin state not preserved after reload |
| A12 | `[P1] AC9, AC2 — Admin Navigation Active State` | P1 | AC2,AC9 | E03-P0-010 | Active indigo-50 highlight not implemented |
| A13 | `[P1] AC9, AC2 — Admin Navigation Active State` | P1 | AC9 | E03-P0-010 | Admin nav item hrefs not rendered |

**Total admin tests:** 13 (all with `test.skip()` — TDD RED phase)

---

## Step 4C: Aggregate Summary

| Metric | Value |
|--------|-------|
| TDD Phase | 🔴 RED |
| Total tests generated | 42 |
| Client E2E tests | 29 (in `app-shell-layout.spec.ts`) |
| Admin E2E tests | 13 (in `app-shell-layout.admin.spec.ts`) |
| API tests | 0 (not applicable — no backend endpoints) |
| All tests use `test.skip()` | ✅ |
| All tests assert expected behavior (not placeholders) | ✅ |
| Placeholder assertions (`expect(true).toBe(true)`) | 0 |

### Priority Distribution

| Priority | Client | Admin | Total |
|----------|--------|-------|-------|
| P0 | 9 | 5 | 14 |
| P1 | 11 | 8 | 19 |
| P2 | 9 | 0 | 9 |
| P3 | 0 | 0 | 0 |
| **Total** | **29** | **13** | **42** |

### AC Coverage

| AC | Tests Covering | Files |
|----|----------------|-------|
| AC1 | T01, T02, A07 | both |
| AC2 | T06, T07, T08, T09, A12, A13 | both |
| AC3 | T10, T11, T12, T13, T14, A08 | client, admin |
| AC4 | T18, T19, T20, T21, T22 | client |
| AC5 | T23, T24, T25 | client |
| AC6 | T26, T27, T28, T29 | client |
| AC7 | T04 | client |
| AC8 | T03, T05 | client |
| AC9 | A01, A02, A03, A04, A05, A06, A12, A13 | admin |
| AC10 | T15, T16, T17, A09, A10, A11 | both |
| AC11 | _(CI gate — existing `pnpm build` test)_ | CI |

**All 11 ACs have at least 1 test.** AC11 is covered by the existing CI build gate (`E03-P0-001`).

---

## Step 5: Validation

### Validation Checklist

- [x] Story has approved acceptance criteria
- [x] All 11 ACs have test coverage
- [x] All tests use `test.skip()` (TDD red phase — documented intentional failure)
- [x] No placeholder assertions (`expect(true).toBe(true)`)
- [x] Tests assert EXPECTED behavior (behavior that will work once implemented)
- [x] Selectors follow hierarchy: `data-testid` primary, `getByRole` secondary, `getByText` tertiary
- [x] No CSS class selectors (brittle), no arbitrary `nth()` indexes
- [x] Network-first patterns not needed (shell is static rendering — no API calls intercepted)
- [x] No orphaned browser sessions (browser automation not used — AI generation mode)
- [x] Test files saved to `eusolicit-app/e2e/specs/shell/` (not random locations)
- [x] Checklist saved to `eusolicit-docs/test-artifacts/` (correct path)
- [x] Client spec uses non-admin matcher (no `.admin.` in filename) → runs in client-chromium, Firefox, WebKit, Mobile
- [x] Admin spec uses `.admin.spec.ts` suffix → runs in admin-chromium only (correct project isolation)
- [x] Failure reasons are accurate (match current codebase state — components not implemented)
- [x] Epic test IDs cross-referenced correctly (E03-P0-009, E03-P0-010, E03-P1-010, E03-P2-004, E03-P2-005)

### Selector Strategy Applied

All selectors use `data-testid` attributes explicitly defined in the story dev notes:
- `data-testid="app-shell"` ← `AppShell.tsx` root div (Task 9.2)
- `data-testid="sidebar"` ← `Sidebar.tsx` `<aside>` (Task 4.1)
- `data-testid="topbar"` ← `TopBar.tsx` `<header>` (Task 5.1)
- `data-testid="content-area"` ← `AppShell.tsx` `<main>` (Task 9.2)
- `data-testid="sidebar-toggle"` ← toggle button in `Sidebar.tsx` (Task 4.2)
- `data-testid="notifications-bell"` ← `NotificationsBell.tsx` (Task 6.1)
- `data-testid="notifications-badge"` ← badge in `NotificationsBell.tsx` (Task 6.1)
- `data-testid="user-avatar-menu-trigger"` ← avatar trigger in `UserAvatarMenu.tsx` (Task 7.1)
- `data-testid="nav-item-{label}"` ← each `NavItem` (lowercase), e.g., `nav-item-dashboard` (Task 3.1)
- `data-testid="dashboard-page"` ← dashboard placeholder page (Tasks 11.2, 12.2)
- `data-testid="language-selector"` ← language selector placeholder (Dev Notes)

---

## Next Steps (TDD Green Phase)

After implementing Story 3.3:

1. **Remove `test.skip()`** from each describe block in:
   - `eusolicit-app/e2e/specs/shell/app-shell-layout.spec.ts`
   - `eusolicit-app/e2e/specs/shell/app-shell-layout.admin.spec.ts`

2. **Start dev servers:**
   ```bash
   cd eusolicit-app/frontend
   pnpm dev --filter client   # http://localhost:3000
   pnpm dev --filter admin    # http://localhost:3001
   ```

3. **Run tests (green phase verification):**
   ```bash
   # From eusolicit-app/
   npx playwright test e2e/specs/shell/ --project=client-chromium
   npx playwright test e2e/specs/shell/ --project=admin-chromium
   ```

4. **Verify ALL 42 tests pass** — if any fail:
   - **Implementation bug:** Fix the component/layout
   - **Test bug:** Fix the assertion (update selector or expected value)

5. **Commit passing tests** alongside the implementation

6. **Proceed to Story 3.4** (responsive breakpoint strategy — mobile/tablet sidebar)
   - Story 3.4 will add `Sheet` overlay for mobile sidebar — tests in a new spec file
   - `AppShell` and `Sidebar` accept `isCollapsed` as a prop (Story 3.3 ensures this) so Story 3.4 can drive breakpoint logic without modifying Story 3.3 code

---

## Implementation Guidance

### UI Components to Implement (Story 3.3)

```
packages/ui/src/components/app-shell/
├── AppShell.tsx          ← AC1, AC7 (data-testid: app-shell, content-area)
├── Sidebar.tsx           ← AC2, AC3, AC7 (data-testid: sidebar, sidebar-toggle)
├── NavItem.tsx           ← AC2, AC8, AC9 (data-testid: nav-item-{label})
├── TopBar.tsx            ← AC4 (data-testid: topbar)
├── NotificationsBell.tsx ← AC6 (data-testid: notifications-bell, notifications-badge)
├── UserAvatarMenu.tsx    ← AC5 (data-testid: user-avatar-menu-trigger)
└── Breadcrumbs.tsx       ← AC4 (no testid required)

apps/client/
├── app/(protected)/layout.tsx  ← AC8 (client nav wiring)
├── app/(protected)/dashboard/page.tsx  ← AC8 (data-testid: dashboard-page)
└── lib/stores/ui-store.ts      ← AC10 (persist key: "eusolicit-ui-store")

apps/admin/
├── app/(protected)/layout.tsx  ← AC9 (admin nav wiring)
├── app/(protected)/dashboard/page.tsx  ← AC9 (data-testid: dashboard-page)
└── lib/stores/ui-store.ts      ← AC10 (persist key: "eusolicit-admin-ui-store")
```

### Critical Risk: SSR Hydration (AC10 — E03-R-004)

Test T17 specifically validates the hydration mismatch prevention. The `mounted` flag pattern **must** be implemented in both `(protected)/layout.tsx` files:
- Render `sidebarCollapsed = false` until `mounted = true` (prevents SSR mismatch)
- Do NOT use `suppressHydrationWarning` on content elements
- See Story 3.3 Dev Notes "SSR Hydration Mismatch Prevention" for the required pattern

### Responsive Compatibility (AC10 — E03-R-005)

Tests verify `isCollapsed` is a **prop** on `<Sidebar>`, not internal state. Story 3.4 depends on this for breakpoint-driven collapsing. Do NOT hardcode sidebar visibility inside `Sidebar.tsx` — the prop must propagate from the layout.

---

## Notes & Assumptions

1. **No authentication required for S3.3 shell tests.** The `(protected)` route group in S3.3 sets up the layout but does not implement auth guards (that is a separate story). Tests navigate directly to `/dashboard` without auth tokens.

2. **No locale prefix in S3.3.** `next-intl` (S3.7) is not implemented yet. URLs are `/dashboard` not `/bg/dashboard`. Epic test design references `/bg/dashboard` for E03-P0-009/010 — those tests will need a URL update when S3.7 lands.

3. **Static placeholder user data.** The `UserAvatarMenu` and `TopBar` in S3.3 accept `user?: { name: string; email: string }` props with placeholder data. Test T24 verifies non-empty label content without asserting specific values.

4. **Sidebar CSS width transition (200ms).** Tests wait 350ms after toggle (200ms + 150ms buffer). This avoids flakiness from animation timing on slower CI runners.

5. **Badge overlay positioning test (T29).** Uses bounding box comparison with ±10px tolerance for absolute positioning. This is a best-effort spatial assertion for the RED phase.

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Story:** 3.3 — App Shell Layout — Sidebar, Top Bar, Content Area
**Version:** BMad v6 (sequential execution mode)
