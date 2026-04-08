---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-08'
workflowType: 'testarch-atdd'
storyId: '3-2-tailwind-design-token-preset-shadcn-ui-theming'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/3-2-tailwind-design-token-preset-shadcn-ui-theming.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-03.md'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-3-1-next-js-14-monorepo-scaffold.md'
  - 'eusolicit-app/playwright.config.ts'
  - 'eusolicit-app/e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts'
  - 'eusolicit-app/frontend/packages/config/tailwind.config.ts'
  - 'eusolicit-app/frontend/apps/client/app/globals.css'
---

# ATDD Checklist: Story 3.2 — Tailwind Design Token Preset & shadcn/ui Theming

## Story Summary

**Epic:** E03 — Frontend Shell & Design System
**Story:** 3.2 — Tailwind Design Token Preset & shadcn/ui Theming
**Sprint:** 1 | **Priority:** P0 (design system foundation — all 10 remaining E03 stories depend on it)
**Risk-Driven:** Yes — E03-R-008 (CSS variable conflicts), E03-P0-001 (build gate), E03-P2-002 (/dev/components)

**Focus Areas:**
- AC1: packages/config/tailwind.config.ts upgraded with full EU Solicit design token preset
- AC2: CSS variables for light/dark mode in both apps' globals.css
- AC3: 19 shadcn/ui components installed in packages/ui/src/components/ui/
- AC4: /dev/components showcase page in client app only (E03-P2-002)
- AC5: Both apps import `<Button>` from `@eusolicit/ui`; `pnpm build` exits 0 (E03-P0-001)

**Why RED:** The current state of `eusolicit-app/frontend/` after Story 3.1:
- `packages/config/tailwind.config.ts` is a minimal placeholder (empty `content[]`, `theme: {extend:{}}`, `plugins: []`)
- Both apps' `globals.css` contain only `@tailwind` directives — no CSS variable block
- `packages/ui/src/components/` directory does NOT exist — all 19 components missing
- Old placeholder `packages/ui/src/Button.tsx` still present (must be deleted and replaced)
- `tsconfig.json` path alias still uses `"./src/*"` (deferred 3.1 finding, must be fixed in 3.2)

---

## TDD Red Phase (Current)

**Status:** RED — All tests use `test.skip()` and will be skipped until Story 3.2 is implemented.

| Test Group | File | Test Count | AC | Status |
|------------|------|-----------|----|----|
| `[P0] AC1 — Tailwind Design Token Preset` | `design-system-setup.api.spec.ts` | 4 | AC1 | Skipped (RED) |
| `[P1] AC1 — Semantic Colors, Fonts, Shadows, Plugin` | same file | 5 | AC1 | Skipped (RED) |
| `[P1] AC1 — App tailwind.config.ts: presets Array Fix` | same file | 3 | AC1 | Skipped (RED) |
| `[P0] AC2 — CSS Variables in globals.css` | same file | 4 | AC2 | Skipped (RED) |
| `[P1] AC2 — CSS Variables Scoped in @layer base` | same file | 4 | AC2 | Skipped (RED) |
| `[P0] AC3 — shadcn/ui Component Files Exist` | same file | 5 | AC3 | Skipped (RED) |
| `[P1] AC3 — packages/ui Barrel Export and Dependencies` | same file | 4 | AC3 | Skipped (RED) |
| `[P2] AC3 — App-Level shadcn/ui Configuration` | same file | 3 | AC3 | Skipped (RED) |
| `[P2] AC4 — /dev/components Page File (Filesystem)` | same file | 3 | AC4 | Skipped (RED) |
| `[P0] AC5 — pnpm Build Gate (E03-P0-001)` | same file | 3 | AC5 | Skipped (RED) |
| `[P2] AC5 — TypeScript Strict Mode (E03-P2-020)` | same file | 2 | AC5 | Skipped (RED) |
| `[P2] AC4 — /dev/components Component Gallery` | `design-system-smoke.spec.ts` | 5 | AC4 | Skipped (RED) |
| `[P2] AC2 — CSS Design Tokens Applied in Browser` | `design-system-smoke.spec.ts` | 3 | AC2 | Skipped (RED) |
| **Total** | **2 files** | **48 tests** | **AC1–AC5** | **All skipped** |

---

## Acceptance Criteria Coverage

### AC1: Tailwind Design Token Preset

| AC Detail | Test | Priority |
|-----------|------|----------|
| `tailwind.config.ts` declares `darkMode: ["class"]` | `[P0] packages/config/tailwind.config.ts declares darkMode: ["class"]` | P0 |
| CSS-variable colors: primary, background, foreground, border, ring mapped | `[P0] tailwind.config.ts maps primary, background, foreground, border, ring CSS variables` | P0 |
| CSS-variable colors: destructive, muted, accent, secondary, popover, card mapped | `[P0] tailwind.config.ts maps destructive, muted, accent, secondary, popover, card CSS variables` | P0 |
| Colors use `hsl(var(--...))` wrapper in Tailwind token map | `[P0] tailwind.config.ts uses hsl(var(--...)) wrapper for CSS variable colors` | P0 |
| Semantic direct colors: success, warning, error, info | `[P1] tailwind.config.ts includes semantic direct colors: success, warning, error, info` | P1 |
| fontFamily.sans uses `--font-inter`; fontFamily.mono uses `--font-jetbrains-mono` | `[P1] tailwind.config.ts declares fontFamily with --font-inter and --font-jetbrains-mono` | P1 |
| Custom boxShadow scale: sm through 2xl | `[P1] tailwind.config.ts declares custom boxShadow scale (sm through 2xl)` | P1 |
| `tailwindcss-animate` plugin registered | `[P1] tailwind.config.ts registers tailwindcss-animate plugin` | P1 |
| `packages/config/package.json` declares `tailwindcss-animate ^1.0.7` | `[P1] packages/config/package.json declares tailwindcss-animate ^1.0.7` | P1 |
| Both apps use `presets: [sharedPreset]` array (not `...baseConfig` spread) | `[P1] apps/client/tailwind.config.ts uses presets: [sharedPreset]` | P1 |
| Both apps use `presets: [sharedPreset]` array (admin) | `[P1] apps/admin/tailwind.config.ts uses presets: [sharedPreset]` | P1 |
| Both apps content array includes `packages/ui/src` for class scanning | `[P1] Both apps tailwind.config.ts content includes packages/ui/src` | P1 |

### AC2: CSS Variables in globals.css

| AC Detail | Test | Priority |
|-----------|------|----------|
| Both apps `:root` block with all required CSS vars (--primary through --ring) | `[P0] Both apps globals.css contain :root CSS variable block with all required tokens` | P0 |
| Both apps `.dark` class CSS variable block | `[P0] Both apps globals.css contain .dark class CSS variable block` | P0 |
| CSS vars in HSL component format (no `hsl()` wrapper in CSS source) | `[P0] Both apps globals.css CSS variables use HSL component values` | P0 |
| Both apps define `--radius` CSS variable | `[P0] Both apps globals.css define --radius CSS variable` | P0 |
| CSS var blocks wrapped in `@layer base` | `[P1] Both apps globals.css wrap CSS variable blocks in @layer base` | P1 |
| `apps/client/app/layout.tsx` loads Inter + JetBrains_Mono from `next/font/google` | `[P1] apps/client/app/layout.tsx loads Inter and JetBrains_Mono fonts` | P1 |
| `apps/admin/app/layout.tsx` loads Inter + JetBrains_Mono from `next/font/google` | `[P1] apps/admin/app/layout.tsx loads Inter and JetBrains_Mono fonts` | P1 |
| Both layouts apply `font-sans antialiased` to `<body>` | `[P1] Both app layouts apply font-sans and antialiased classes to <body>` | P1 |
| `--primary` CSS variable applied in browser (HSL components) | `[P2] --primary CSS variable is defined and in HSL component format (E03-P2-003)` | P2 |
| `--background` and `--foreground` defined in browser | `[P2] --background and --foreground CSS variables are defined (E03-P2-003)` | P2 |
| `--destructive` defined in browser | `[P2] --destructive CSS variable is defined (E03-P2-003)` | P2 |

### AC3: 19 shadcn/ui Components in packages/ui

| AC Detail | Test | Priority |
|-----------|------|----------|
| All 19 component files exist in `packages/ui/src/components/ui/` | `[P0] All 19 shadcn/ui component files exist in packages/ui/src/components/ui/` | P0 |
| `label.tsx` exists (required by Radix form primitives) | `[P0] label.tsx exists in packages/ui/src/components/ui/` | P0 |
| `packages/ui/src/lib/utils.ts` with `cn()` using `clsx` + `twMerge` | `[P0] packages/ui/src/lib/utils.ts exists with cn() helper using clsx + twMerge` | P0 |
| Old `packages/ui/src/Button.tsx` removed | `[P0] Old packages/ui/src/Button.tsx removed (replaced by components/ui/button.tsx)` | P0 |
| `button.tsx` uses CVA with all required variants | `[P0] button.tsx implements CVA variants (default, destructive, outline, secondary, ghost, link)` | P0 |
| `packages/ui/index.ts` exports all 19 components | `[P1] packages/ui/index.ts exports all 19 shadcn/ui components (E03-P2-001)` | P1 |
| `packages/ui/index.ts` exports `cn` from `./src/lib/utils` | `[P1] packages/ui/index.ts exports cn utility from ./src/lib/utils` | P1 |
| `packages/ui/package.json` has all 13 Radix UI primitive deps | `[P1] packages/ui/package.json declares all required Radix UI primitive dependencies` | P1 |
| `packages/ui/package.json` has cva, clsx, tailwind-merge, lucide-react | `[P1] packages/ui/package.json declares utility dependencies: cva, clsx, tailwind-merge, lucide-react` | P1 |
| Both apps `tsconfig.json` path alias `@/*` → `["./*"]` | `[P2] Both apps tsconfig.json @/* path alias maps to ["./*"] — Story 3.1 fix` | P2 |
| Both apps `lib/utils.ts` re-exports `cn` from `@eusolicit/ui` | `[P2] Both apps have lib/utils.ts that re-exports cn from @eusolicit/ui` | P2 |
| Both apps have `components.json` (shadcn/ui CLI config) | `[P2] Both apps have components.json (shadcn/ui CLI configuration)` | P2 |

### AC4: /dev/components Page (client app only)

| AC Detail | Test | Priority |
|-----------|------|----------|
| `apps/client/app/dev/components/page.tsx` exists | `[P2] apps/client/app/dev/components/page.tsx exists (client app only per AC4)` | P2 |
| admin app does NOT have /dev/components page | `[P2] apps/admin does NOT have /dev/components page (client app only per AC4)` | P2 |
| `page.tsx` imports from `@eusolicit/ui` and references all 19 components | `[P2] page.tsx imports from @eusolicit/ui and renders all 19 components` | P2 |
| Page loads HTTP 200 with no JS errors (E03-P2-002) | `[P2] /dev/components page loads with HTTP 200 and no JavaScript errors (E03-P2-002)` | P2 |
| All 19 component section headings visible (E03-P2-002) | `[P2] /dev/components page renders section headings for all 19 components (E03-P2-002)` | P2 |
| Button renders at least 6 variant instances (E03-P2-002) | `[P2] Button component renders at least 6 variant instances (E03-P2-002)` | P2 |
| Badge section visible (E03-P2-002) | `[P2] Badge component renders all 4 variants (E03-P2-002)` | P2 |
| Card section visible (E03-P2-002) | `[P2] Card component section renders without errors (E03-P2-002)` | P2 |

### AC5: Build Gate + Button Import

| AC Detail | Test | Priority |
|-----------|------|----------|
| `pnpm build` exits code 0 for both apps (E03-P0-001) | `[P0] pnpm build at frontend/ root exits with code 0 (E03-P0-001)` ⚠️ SLOW | P0 |
| Build output has no TypeScript errors (`error TS\d+`) | `[P0] pnpm build output contains no TypeScript error diagnostics` ⚠️ SLOW | P0 |
| Build output has no ESLint failures | `[P0] pnpm build output contains no ESLint failure lines` ⚠️ SLOW | P0 |
| `tsc --noEmit` exits 0 in apps/client (E03-P2-020) | `[P2] tsc --noEmit exits 0 in apps/client (E03-P2-020)` ⚠️ SLOW | P2 |
| `tsc --noEmit` exits 0 in apps/admin (E03-P2-020) | `[P2] tsc --noEmit exits 0 in apps/admin (E03-P2-020)` ⚠️ SLOW | P2 |

---

## Epic Test Design Traceability

| Epic Test Design Requirement | AC | Test(s) | Priority |
|------------------------------|-----|---------|----------|
| **E03-P0-001**: `pnpm build` succeeds for both apps — hard CI gate | AC5 | `[P0] pnpm build exits code 0` | **P0** |
| **E03-P2-001**: Shared `<Button>` from `packages/ui` importable in both apps | AC3, AC5 | `[P1] packages/ui/index.ts exports all 19 components`, build gate | P1 / P0 |
| **E03-P2-002**: `/dev/components` page renders all 19 shadcn/ui components without JS errors | AC4 | `[P2] /dev/components page renders 19 section headings` (smoke) | P2 |
| **E03-P2-003**: Tailwind preset design tokens visible — CSS vars `--primary`, `--destructive`, `--muted` set | AC1, AC2 | `[P2] --primary CSS variable defined in browser` (smoke) | P2 |
| **E03-P2-020**: TypeScript strict mode — zero type errors across all packages and apps | AC5 | `[P2] tsc --noEmit exits 0 in apps/client and apps/admin` | P2 |
| **E03-R-008**: CSS variable conflicts between shared preset and app-level globals.css | AC1, AC2 | `/dev/components visual QA + [P0] Both apps globals.css define all required CSS vars` | P0 / P2 |

---

## Anti-Pattern Guards (from Story Dev Notes)

These tests directly guard against the "Critical Story 3.1 Learnings" and "What NOT to Do" in the story spec:

| Anti-Pattern | Guard Test |
|--------------|-----------|
| `...baseConfig` spread in apps' tailwind.config.ts (silently drops base entries) | `[P1] apps/*.tailwind.config.ts must use presets: [sharedPreset] — NOT ...baseConfig spread` |
| `"@/*": ["./src/*"]` in tsconfig.json (apps have no `src/` dir) | `[P2] Both apps tsconfig.json @/* must map to ["./*"]` |
| Old `packages/ui/src/Button.tsx` left alongside new button.tsx | `[P0] Old packages/ui/src/Button.tsx must be deleted` |
| CSS variables in globals.css using `hsl()` wrapper (breaks shadcn/ui convention) | `[P0] Both apps globals.css CSS variables use HSL component values — no hsl() wrapper` |
| `/dev/components` page created in admin app (must be client-only per AC4) | `[P2] admin app must NOT have /dev/components page` |
| tailwind.config.ts content array includes `./src/**/*` (no src/ dir in apps) | Implicit in `[P1] Both apps tailwind.config.ts content includes packages/ui/src` (not ./src) |
| Creating `next.config.ts` (not supported in Next.js 14.x) | Inherited from Story 3.1 ATDD — no new files of this type created in Story 3.2 |

---

## Fixture Needs

_None required for GREEN phase._ The API-level tests are pure filesystem and subprocess assertions.
The browser E2E tests (design-system-smoke.spec.ts) require only a running dev server — no auth, no fixtures, no database.

| Resource | Purpose | Status |
|----------|---------|--------|
| `pnpm` in PATH | Build command execution (AC5 SLOW tests) | Required at GREEN phase |
| `eusolicit-app/frontend/` directory | Target of all filesystem assertions | Exists from Story 3.1 |
| `pnpm dev --filter client` | Client app running at port 3000 | Required for smoke tests |
| Node.js `fs` module | File existence and content checks | Available (stdlib) |
| Node.js `child_process.spawnSync` | Build command execution | Available (stdlib) |

---

## Mock Requirements

_None._ Story 3.2 tests are scaffold, build, and visual verification — no HTTP API calls, no service dependencies, no database access, no authentication.

---

## Required data-testid Attributes

_Not applicable_ for filesystem and build tests (design-system-setup.api.spec.ts).

For browser E2E tests (design-system-smoke.spec.ts):
- No `data-testid` attributes required — tests use text-based selectors (`getByText`) and structural assertions
- This matches the story requirement: components "with section labels" (text visible)

---

## Slow Test Strategy

Tests that execute real shell commands (~2-5 minutes total for all slow tests):

| Test | Command | Expected Duration | Run Mode |
|------|---------|-------------------|----------|
| `[P0] pnpm build exits code 0` | `pnpm build` at `frontend/` | ~2–4 minutes | CI on every PR |
| `[P0] pnpm build — no TS errors` | Same `pnpm build` output | same run | CI on every PR |
| `[P0] pnpm build — no ESLint failures` | Same `pnpm build` output | same run | CI on every PR |
| `[P2] tsc --noEmit in apps/client` | `pnpm exec tsc --noEmit` | ~30 seconds | CI on every PR |
| `[P2] tsc --noEmit in apps/admin` | `pnpm exec tsc --noEmit` | ~30 seconds | CI on every PR |

**Playwright project:** `api` (no browser — uses `.api.spec.ts` file match pattern). Run slow tests in isolation:

```bash
# All Story 3.2 API-level tests (filesystem + build)
cd eusolicit-app
pnpm playwright test --project=api e2e/specs/smoke/design-system-setup.api.spec.ts

# Filesystem-only tests (fast, no build commands)
pnpm playwright test --project=api -g "AC1|AC2|AC3|AC4" --grep-invert "E03-P0-001|E03-P2-020"

# Build gate only (E03-P0-001 — SLOW)
pnpm playwright test --project=api -g "E03-P0-001"

# TypeScript strict checks (E03-P2-020 — SLOW)
pnpm playwright test --project=api -g "E03-P2-020"

# Browser smoke tests (requires running dev server)
pnpm playwright test --project=client-chromium e2e/specs/smoke/design-system-smoke.spec.ts
```

---

## Implementation Guidance

### What Must Be Created/Modified (to Turn Tests GREEN)

#### Task 1: Upgrade Tailwind Config Preset
- `packages/config/package.json` — add `tailwindcss-animate ^1.0.7` to devDependencies
- `packages/config/tailwind.config.ts` — **replace entire file** with full design token preset:
  - `darkMode: ["class"]`
  - CSS-variable colors (primary through card + foreground variants)
  - Semantic direct colors (success/green, warning/amber, error/red, info/blue)
  - fontFamily.sans: `["var(--font-inter)", "system-ui", ...]`
  - fontFamily.mono: `["var(--font-jetbrains-mono)", "ui-monospace", ...]`
  - boxShadow: sm through 2xl custom values
  - keyframes: accordion-down, accordion-up
  - plugins: `[require("tailwindcss-animate")]`
- `apps/client/tailwind.config.ts` — replace `...baseConfig` spread with `presets: [sharedPreset]`
- `apps/admin/tailwind.config.ts` — same fix

#### Task 2: Add CSS Variables to Both Apps
- `apps/client/app/globals.css` — replace with `@tailwind` directives + `@layer base { :root { ... } .dark { ... } }` blocks
- `apps/admin/app/globals.css` — identical content

#### Task 3: Font Loading in Both App Layouts
- `apps/client/app/layout.tsx` — add Inter + JetBrains_Mono from `next/font/google`; apply `--font-inter`, `--font-jetbrains-mono` variables; body gets `font-sans antialiased`
- `apps/admin/app/layout.tsx` — same pattern

#### Task 4: tsconfig Path Alias + lib/utils
- `apps/client/tsconfig.json` — change `"@/*": ["./src/*"]` → `"@/*": ["./*"]`
- `apps/admin/tsconfig.json` — same change
- `apps/client/lib/utils.ts` — `export { cn } from '@eusolicit/ui'`
- `apps/admin/lib/utils.ts` — same

#### Task 5: packages/ui Dependencies
- `packages/ui/package.json` — add `dependencies` field with 13 Radix UI primitives + 4 utilities
- Run `pnpm install` from `frontend/` root

#### Task 6: Create 19 shadcn/ui Components
- Delete `packages/ui/src/Button.tsx` (placeholder)
- Create `packages/ui/src/lib/utils.ts` — `cn()` with `clsx` + `twMerge`
- Create all 19 + label in `packages/ui/src/components/ui/`:
  `button.tsx`, `input.tsx`, `textarea.tsx`, `label.tsx`, `select.tsx`, `checkbox.tsx`, `radio-group.tsx`, `switch.tsx`, `dialog.tsx`, `sheet.tsx`, `dropdown-menu.tsx`, `tooltip.tsx`, `badge.tsx`, `card.tsx`, `tabs.tsx`, `table.tsx`, `skeleton.tsx`, `avatar.tsx`, `separator.tsx`, `scroll-area.tsx`

#### Task 7: Update packages/ui Barrel Export
- `packages/ui/index.ts` — rewrite to export `cn` + all 20 component files (19 + label)

#### Task 8: Add components.json to Both Apps
- `apps/client/components.json` — shadcn/ui config with `style: "default"`, `rsc: true`, aliases
- `apps/admin/components.json` — same config

#### Task 9: Create /dev/components Showcase Page
- `apps/client/app/dev/components/page.tsx` — renders all 19 components with section headings and variant previews; imports from `@eusolicit/ui`

#### Task 10: Verify Build
- Run `pnpm build` at `frontend/` — both apps must exit 0
- Run `pnpm type-check` — no TypeScript errors

---

## Red-Green-Refactor Workflow

### RED Phase (Complete — TEA)

- [x] 48 acceptance test cases generated across 2 test files
- [x] All tests use `test.skip()` to document intentional RED phase
- [x] Tests assert EXPECTED behavior against unimplemented Story 3.2
- [x] Tests organized by acceptance criterion (AC1–AC5) in named describe blocks
- [x] P0 build gate test (E03-P0-001) and TypeScript strict test (E03-P2-020) explicitly linked
- [x] Anti-pattern guards from story spec included (spread, src alias, old Button.tsx)
- [x] Test files:
  - `eusolicit-app/e2e/specs/smoke/design-system-setup.api.spec.ts` (API-level, no browser)
  - `eusolicit-app/e2e/specs/smoke/design-system-smoke.spec.ts` (browser E2E, client-chromium)

### GREEN Phase (Next — Dev Team)

After implementing each task group, enable tests one AC group at a time:

1. **Start with AC1 (tailwind preset) and AC2 (CSS variables) — fast tests:**
   ```bash
   # Remove test.skip() from AC1 and AC2 describe blocks, run:
   pnpm playwright test --project=api -g "AC1|AC2" --grep-invert "SLOW"
   ```

2. **Enable AC3 (components) — all filesystem tests:**
   ```bash
   pnpm playwright test --project=api -g "AC3"
   ```

3. **Enable AC4 filesystem check:**
   ```bash
   pnpm playwright test --project=api -g "AC4 — /dev/components Page File"
   ```

4. **Run build gate (AC5 — SLOW, ~3-5 minutes):**
   ```bash
   pnpm playwright test --project=api -g "E03-P0-001"
   ```

5. **Run TypeScript strict checks (AC5 — SLOW):**
   ```bash
   pnpm playwright test --project=api -g "E03-P2-020"
   ```

6. **Run browser smoke tests (AC4 + AC2 browser — requires dev server):**
   ```bash
   # Start dev server first:
   cd frontend && pnpm dev --filter client &
   # Then run smoke tests:
   cd /eusolicit-app
   pnpm playwright test --project=client-chromium e2e/specs/smoke/design-system-smoke.spec.ts
   ```

7. **Full suite:**
   ```bash
   pnpm playwright test --project=api e2e/specs/smoke/design-system-setup.api.spec.ts
   pnpm playwright test --project=client-chromium e2e/specs/smoke/design-system-smoke.spec.ts
   ```

If any tests fail:
- **Filesystem test fails:** Check that the file was created at the expected path (verify FRONTEND_ROOT path)
- **CSS variable test fails:** Verify globals.css format — HSL components only, no `hsl()` wrapper
- **Component file test fails:** Verify all 19 files are in `src/components/ui/` (not `src/components/`)
- **Old Button.tsx test fails:** Delete `packages/ui/src/Button.tsx` explicitly
- **tsconfig path test fails:** Check `"@/*": ["./*"]` (not `["./src/*"]`)
- **Build test fails:** Run `pnpm build` manually; check for TypeScript errors, missing deps, wrong content scan paths

### REFACTOR Phase (Dev Team)

- Consider splitting slow build tests into a `*.build.spec.ts` suite with Playwright tags (`@slow`)
- Add per-component visual regression snapshots once design system is stable (P3 enhancement)
- Add `pnpm type-check` as separate turbo task gate once E03-P0-001 is consistently green
- Consider Storybook integration for component visual QA (deferred to post-MVP per Epic scope)

---

## Execution Commands

```bash
# Navigate to project root
cd eusolicit-app

# Run ALL Story 3.2 API-level tests (currently all skipped — RED phase)
pnpm playwright test --project=api e2e/specs/smoke/design-system-setup.api.spec.ts

# Run by AC group (fast — filesystem only)
pnpm playwright test --project=api -g "AC1 — Tailwind"
pnpm playwright test --project=api -g "AC2 — CSS Variables"
pnpm playwright test --project=api -g "AC3 — shadcn"
pnpm playwright test --project=api -g "AC4 — /dev/components Page File"

# Run build gate only (E03-P0-001 — SLOW, ~3-5 min)
pnpm playwright test --project=api -g "E03-P0-001"

# Run TypeScript strict checks (E03-P2-020 — SLOW)
pnpm playwright test --project=api -g "E03-P2-020"

# Run browser smoke tests for /dev/components (E03-P2-002, E03-P2-003)
pnpm playwright test --project=client-chromium e2e/specs/smoke/design-system-smoke.spec.ts

# Run as part of the full smoke suite
pnpm playwright test --project=api e2e/specs/smoke/ --reporter=list

# Run cross-browser for smoke (once GREEN)
pnpm playwright test e2e/specs/smoke/design-system-smoke.spec.ts --reporter=list
```

---

## Completion Summary

| Metric | Value |
|--------|-------|
| **Story ID** | 3-2-tailwind-design-token-preset-shadcn-ui-theming |
| **Primary test level** | Build/CI + Scaffold (filesystem + subprocess) + Browser E2E (component gallery) |
| **Test functions** | 48 |
| **Test files** | 2 (`design-system-setup.api.spec.ts`, `design-system-smoke.spec.ts`) |
| **Playwright projects** | `api` (no browser) + `client-chromium/firefox/webkit/mobile` (browser smoke) |
| **Fixtures needed** | 0 (pure filesystem/subprocess for API tests; dev server for smoke tests) |
| **Mock requirements** | 0 |
| **data-testid requirements** | 0 (text-based selectors for smoke tests) |
| **External dependencies** | `pnpm` in PATH (for slow build tests); dev server on port 3000 (for smoke tests) |
| **Execution mode** | Parallel-safe for API tests; smoke tests require sequential dev server startup |
| **TDD phase** | RED (all 48 tests skipped) |

### Priority Breakdown

| Priority | Test Count | AC Coverage |
|----------|------------|-------------|
| **P0** | 16 | AC1 (4 tests), AC2 (4 tests), AC3 (5 tests), AC5 (3 tests — SLOW build gate) |
| **P1** | 16 | AC1 (8 tests: semantic, fonts, shadows, plugin, presets), AC2 (4 tests: @layer, fonts, body), AC3 (4 tests: barrel export + deps) |
| **P2** | 16 | AC3 (3 tests: app config), AC4 (8 tests: 3 filesystem + 5 browser), AC2 (3 browser: CSS token validation), AC5 (2 tests: tsc) |
| **P3** | 0 | — (dark mode deferred to stretch AC per Epic 3 test design) |
| **Total** | **48** | **AC1–AC5 (100%)** |

### Epic Test Design Coverage

| Epic Test ID | Priority | AC | Covered By | Gate |
|-------------|---------|-----|-----------|------|
| E03-P0-001 | **P0** | AC5 | `pnpm build exits code 0` | Hard CI fail |
| E03-P2-001 | P1 | AC3 | `packages/ui/index.ts exports all 19 components` + build gate | — |
| E03-P2-002 | P2 | AC4 | `/dev/components renders 19 section headings` (smoke) | — |
| E03-P2-003 | P2 | AC1, AC2 | `--primary CSS variable defined in browser` (smoke) | — |
| E03-P2-020 | P2 | AC5 | `tsc --noEmit exits 0 in both apps` | — |

### Risk Mitigation Coverage

| Risk | Score | Mitigation | Test Coverage |
|------|-------|------------|---------------|
| E03-R-008: CSS variable conflicts | 2 | Both apps CSS vars from same preset; caught in /dev/components visual QA | `[P0]` All 19 CSS vars tested in both apps; smoke E03-P2-003 browser validation |
| Tailwind `...baseConfig` spread (AC1) | High | Anti-pattern guard from story spec | `[P1]` Both apps presets array checked |
| Old `Button.tsx` left behind (AC3) | Medium | Explicit deletion check | `[P0]` Old file absence asserted |
| Wrong tsconfig @/* alias (AC3 fix) | Medium | Explicit path value assertion | `[P2]` Exact `["./*"]` value checked |
| Build failure from missing deps (AC5) | High | Build gate + TypeScript checks | `[P0]` 3 SLOW build gate tests |

### Knowledge Base References Applied

- `test-levels-framework.md` — Build/CI level for scaffold + config stories; Browser E2E for visual gallery
- `test-priorities-matrix.md` — P0 for build gate; P1 for token/CSS/component setup; P2 for visual QA
- `test-quality.md` — Deterministic assertions, parallel-safe API tests, zero external service dependencies
- `selector-resilience.md` — Text-based selectors (`getByText`) for smoke tests (no brittle CSS class selectors)
- Story 3.1 ATDD pattern — filesystem + subprocess test structure carried forward

### Next Steps

1. **Implement Story 3.2** — Follow Tasks 1–10 in the story spec (see Implementation Guidance above)
2. **Remove `test.skip()`** — One AC group at a time; verify GREEN per group
3. **Run API suite** — `pnpm playwright test --project=api e2e/specs/smoke/design-system-setup.api.spec.ts`
4. **Start dev server and run smoke** — `pnpm playwright test --project=client-chromium e2e/specs/smoke/design-system-smoke.spec.ts`
5. **Verify CI gate** — Ensure `pnpm build` gate (E03-P0-001) is part of the PR check workflow
6. **Proceed to Story 3.3** — App Shell Layout (depends on design system from this story)

---

**Generated by:** BMad TEA Agent — ATDD Module
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
