# Story DW-03: UI Breakpoint Hook Fix (1280px → 1024px)

Status: draft

**Origin:** Deferred work from code review of S07.11 (2026-04-18)
**Priority:** Medium — panels collapse on 1024–1279px viewports that spec requires expanded
**Epic:** Post-E07 cleanup (no assigned epic)
**Points:** 1 | **Type:** frontend (shared UI library)

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager using a 1080p monitor**,
I want the **proposal workspace panels to expand by default at 1024px+ viewports** (not 1280px+),
so that the workspace is usable without manual panel expansion on typical laptop screens.

## Acceptance Criteria

1. **`isDesktop` threshold changed from 1280px to 1024px** — In `eusolicit-app/frontend/packages/ui/src/lib/hooks/useBreakpoint.ts` (line 11 per deferred-work.md), change the `isDesktop` breakpoint from Tailwind `xl` (1280px) to Tailwind `lg` (1024px). Specifically, change `window.innerWidth >= 1280` (or the equivalent Tailwind breakpoint constant) to `window.innerWidth >= 1024`.

2. **All consumers of `useBreakpoint` still function correctly** — `isDesktop` is a shared hook. Before changing the threshold, grep for all usages of `useBreakpoint` across `eusolicit-app/frontend/`:
   - If any consumer depends on the 1280px threshold for layout purposes (e.g., a sidebar that genuinely needs 1280px+), document the dependency and do NOT change the shared hook — instead, accept the `lg` threshold as a local override in `ProposalWorkspacePage.tsx` only.
   - If all consumers want `lg` (1024px), change the shared hook.

3. **`ProposalWorkspacePage` panels expand at 1024px** — After the fix, panels in the proposal workspace start expanded when `window.innerWidth >= 1024`. The ATDD test in `__tests__/proposals-workspace-s7-11.test.ts` already asserts `useBreakpoint` usage — run it to confirm still green after the change.

4. **No existing tests regress** — Run `pnpm --filter client test` and confirm the existing 159 ATDD tests in `proposals-workspace-s7-11.test.ts` and all other test files pass.

## Tasks / Subtasks

- [ ] Task 1: Audit `useBreakpoint` consumers (AC: 2)
  - [ ] 1.1 `grep -r "useBreakpoint" eusolicit-app/frontend/ --include="*.tsx" --include="*.ts" -l`
  - [ ] 1.2 For each consumer file, read the component and check whether its layout depends on 1280px specifically

- [ ] Task 2: Apply threshold fix (AC: 1, 2)
  - [ ] 2.1 **If all consumers can use 1024px**: Open `packages/ui/src/lib/hooks/useBreakpoint.ts`; change the threshold from `1280` to `1024`
  - [ ] 2.2 **If only the proposal workspace needs 1024px**: In `ProposalWorkspacePage.tsx`, replace `const { isDesktop } = useBreakpoint()` with `const [isDesktop, setIsDesktop] = useState(false); useEffect(() => { const handler = () => setIsDesktop(window.innerWidth >= 1024); handler(); window.addEventListener("resize", handler); return () => window.removeEventListener("resize", handler); }, [])` as a local override
  - [ ] 2.3 Run `pnpm --filter client test` to verify 159 ATDD tests still green

## Dev Notes

### File Location

From deferred-work.md: `packages/ui/src/lib/hooks/useBreakpoint.ts:11`

The hook likely uses a `window.innerWidth` comparison or `matchMedia` with a Tailwind breakpoint value. The current code:
```typescript
// approximate — verify actual implementation before changing
const isDesktop = window.innerWidth >= 1280; // xl breakpoint
```

Should become:
```typescript
const isDesktop = window.innerWidth >= 1024; // lg breakpoint
```

### Impact Assessment

`useBreakpoint` was introduced in E03. It is used in at least `ProposalWorkspacePage.tsx` (S07.11). Check whether `AppShell`, sidebar, or other layout components also use it — those are more likely to have breakpoint-sensitive layout logic.

### References

- [Source: eusolicit-docs/implementation-artifacts/deferred-work.md] — DW-03 origin item
- [Source: eusolicit-docs/implementation-artifacts/7-11-proposal-workspace-page-layout-navigation.md#Known Deviations] — breakpoint deviation
- [Source: eusolicit-app/frontend/packages/ui/src/lib/hooks/useBreakpoint.ts:11] — target line
- [Source: eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalWorkspacePage.tsx] — primary consumer
