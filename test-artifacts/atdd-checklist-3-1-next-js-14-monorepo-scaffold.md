---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-07'
workflowType: 'testarch-atdd'
storyId: '3-1-next-js-14-monorepo-scaffold'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/3-1-next-js-14-monorepo-scaffold.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-03.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-app/playwright.config.ts'
  - 'eusolicit-app/e2e/specs/smoke/scaffold-structure.api.spec.ts'
---

# ATDD Checklist: Story 3.1 — Next.js 14 Monorepo Scaffold

## Story Summary

**Epic:** E03 — Frontend Shell & Design System
**Story:** 3.1 — Next.js 14 Monorepo Scaffold
**Sprint:** 1 | **Priority:** P0 (foundation for all 11 remaining E03 stories)
**Risk-Driven:** Yes — E03-P0-001 is a hard CI gate; all 51 other E03 tests depend on this scaffold

**Focus Areas:**
- AC1: Turborepo monorepo structure with apps/client and apps/admin using App Router
- AC2: packages/ui shared component library with index.ts barrel export (E03-P2-001)
- AC3: packages/config shared tooling configuration (tailwind, tsconfig, eslint, prettier)
- AC4/AC5: Dev server port assignment — client on 3000, admin on 3001
- AC6: pnpm build succeeds for both apps with zero TypeScript/ESLint errors (E03-P0-001)
- AC7: TypeScript strict mode inherited from packages/config/tsconfig.json (E03-P2-020)
- AC8: ESLint + Prettier configured and pnpm lint passes

**Why RED:** The current `eusolicit-app/frontend/` is a flat Next.js placeholder from Story 1.1
(package.json, tsconfig.json, next.config.mjs, app/layout.tsx, app/page.tsx). The Turborepo
monorepo structure (apps/, packages/, turbo.json, pnpm-workspace.yaml) does not exist yet.

---

## TDD Red Phase (Current)

**Status:** RED — All tests use `test.skip()` and will be skipped until Story 3.1 is implemented.

| Test Group | File | Test Count | AC | Status |
|------------|------|-----------|----|----|
| `[P0] AC1 — App Router Directory Structure` | `e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts` | 8 | AC1 | Skipped (RED) |
| `[P1] AC1 — Turborepo Root Files` | same file | 5 | AC1 | Skipped (RED) |
| `[P0] AC2 — packages/ui Library` | same file | 5 | AC2 | Skipped (RED) |
| `[P2] AC2 — Workspace deps` | same file | 1 | AC2 | Skipped (RED) |
| `[P0] AC3 — packages/config` | same file | 4 | AC3 | Skipped (RED) |
| `[P1] AC3 — Prettier + app ESLint` | same file | 2 | AC3 | Skipped (RED) |
| `[P1] AC4/AC5 — Port Config` | same file | 5 | AC4, AC5 | Skipped (RED) |
| `[P0] AC6 — Build Gate` | same file | 5 | AC6 | Skipped (RED) |
| `[P2] AC7 — TypeScript Strict` | same file | 6 | AC7 | Skipped (RED) |
| `[P1] AC8 — ESLint + Prettier lint` | same file | 3 | AC8 | Skipped (RED) |
| **Total** | **1 file** | **44 tests** | **AC1–AC8** | **All skipped** |

---

## Acceptance Criteria Coverage

### AC1: App Router Directory Structure

| AC Detail | Test | Priority |
|-----------|------|----------|
| `frontend/apps/client/app/` exists | `[P0] frontend/apps/client/ directory exists with App Router app/ subdirectory` | P0 |
| `frontend/apps/admin/app/` exists | `[P0] frontend/apps/admin/ directory exists with App Router app/ subdirectory` | P0 |
| `apps/client/app/` has layout.tsx, page.tsx, globals.css | `[P0] apps/client/app/ contains layout.tsx, page.tsx, and globals.css` | P0 |
| `apps/admin/app/` has layout.tsx, page.tsx, globals.css | `[P0] apps/admin/app/ contains layout.tsx, page.tsx, and globals.css` | P0 |
| `apps/client/app/layout.tsx` uses metadata export and `lang="bg"` | `[P0] apps/client/app/layout.tsx uses App Router metadata export and lang="bg"` | P0 |
| `apps/admin/app/layout.tsx` uses metadata export and `lang="bg"` | `[P0] apps/admin/app/layout.tsx uses App Router metadata export and lang="bg"` | P0 |
| Both apps use `next.config.ts` (not .js or .mjs) | `[P0] Both apps use next.config.ts (TypeScript format — not .js or .mjs)` | P0 |
| Both apps `next.config.ts` declare `transpilePackages: ['@eusolicit/ui']` | `[P0] Both apps next.config.ts declare transpilePackages: ["@eusolicit/ui"]` | P0 |
| Old flat `frontend/app/` directory removed | `[P1] Old flat frontend/app/ directory removed (Story 1.1 placeholder cleaned up)` | P1 |
| `pnpm-workspace.yaml`, `turbo.json`, `.npmrc` all present | `[P1] Turborepo root files present: pnpm-workspace.yaml, turbo.json, .npmrc` | P1 |
| `pnpm-workspace.yaml` references `apps/*` and `packages/*` | `[P1] pnpm-workspace.yaml references apps/* and packages/*` | P1 |
| `turbo.json` pipeline has build, dev, lint, type-check | `[P1] turbo.json pipeline declares build, dev, lint, type-check tasks` | P1 |
| `.npmrc` has strict-peer-dependencies=false, shamefully-hoist=false | `[P1] .npmrc declares strict-peer-dependencies=false and shamefully-hoist=false` | P1 |

### AC2: packages/ui Shared Component Library

| AC Detail | Test | Priority |
|-----------|------|----------|
| `frontend/packages/ui/` directory exists | `[P0] frontend/packages/ui/ directory exists` | P0 |
| `packages/ui/index.ts` barrel export file exists | `[P0] packages/ui/index.ts barrel export file exists (REQUIRED per story spec)` | P0 |
| `packages/ui/index.ts` exports Button | `[P0] packages/ui/index.ts exports Button component` | P0 |
| `packages/ui/src/Button.tsx` is valid React component | `[P0] packages/ui/src/Button.tsx placeholder component exists and is a valid React component` | P0 |
| `packages/ui/package.json` declares `@eusolicit/ui`, version 0.0.0, peer deps | `[P0] packages/ui/package.json declares correct name, version and peer deps` | P0 |
| Both apps depend on `@eusolicit/ui` via `workspace:*` | `[P2] apps/client and apps/admin both declare @eusolicit/ui workspace dependency (E03-P2-001)` | P2 |

### AC3: packages/config Shared Configuration

| AC Detail | Test | Priority |
|-----------|------|----------|
| `frontend/packages/config/` exists, package.json names `@eusolicit/config` | `[P0] frontend/packages/config/ directory exists with correct package name` | P0 |
| `packages/config/tailwind.config.ts` is TypeScript format | `[P0] packages/config/tailwind.config.ts exists and is TypeScript (not .js)` | P0 |
| `packages/config/tsconfig.json` is valid JSON | `[P0] packages/config/tsconfig.json is valid JSON with compilerOptions` | P0 |
| `packages/config/.eslintrc.js` extends next/core-web-vitals + @typescript-eslint | `[P0] packages/config/.eslintrc.js extends next/core-web-vitals and @typescript-eslint` | P0 |
| `packages/config/prettier.config.js` declares tabWidth, printWidth, trailingComma | `[P1] packages/config/prettier.config.js declares tabWidth: 2 and printWidth: 100` | P1 |
| Both apps `.eslintrc.js` extend `@eusolicit/config` | `[P1] apps/client and apps/admin .eslintrc.js files extend @eusolicit/config` | P1 |

### AC4 & AC5: Dev Server Port Assignment

| AC Detail | Test | Priority |
|-----------|------|----------|
| `apps/client/package.json` dev script has `-p 3000` | `[P1] apps/client/package.json dev script runs on port 3000 (AC4)` | P1 |
| `apps/admin/package.json` dev script has `-p 3001` | `[P1] apps/admin/package.json dev script runs on port 3001 (AC5)` | P1 |
| Both apps pin Next.js to `14.x` (not 15.x) | `[P1] Both apps declare Next.js 14.x (NOT 15.x — pinned per epic specification)` | P1 |
| Root `frontend/package.json` scripts delegate to turbo | `[P1] root frontend/package.json has turbo-delegating build, dev, lint, type-check scripts` | P1 |
| Root `frontend/package.json` declares `engines: node >=20, pnpm >=8` | `[P1] root frontend/package.json declares engines: node >=20 and pnpm >=8` | P1 |

### AC6: pnpm build Build Gate

| AC Detail | Test | Priority |
|-----------|------|----------|
| Both apps have `postcss.config.js` | `[P0] Both apps postcss.config.js exists (required for Tailwind CSS — build prerequisite)` | P0 |
| Both apps `globals.css` has `@tailwind` directives | `[P0] Both apps globals.css contains @tailwind base, components, utilities directives` | P0 |
| `pnpm build` exits code 0 (E03-P0-001 hard gate) | `[P0] pnpm build at frontend/ root exits with code 0 for both client and admin (E03-P0-001)` ⚠️ SLOW | **P0** |
| Build output has no TypeScript errors (`error TS\d+`) | `[P0] pnpm build output contains no TypeScript error diagnostics (error TS\d+)` ⚠️ SLOW | P0 |
| Build output has no ESLint failures | `[P0] pnpm build output contains no ESLint failure lines` ⚠️ SLOW | P0 |

### AC7: TypeScript Strict Mode

| AC Detail | Test | Priority |
|-----------|------|----------|
| `packages/config/tsconfig.json` has `strict: true` | `[P2] packages/config/tsconfig.json base config declares strict: true` | P2 |
| `apps/client/tsconfig.json` extends `@eusolicit/config` | `[P2] apps/client/tsconfig.json extends @eusolicit/config/tsconfig.json` | P2 |
| `apps/admin/tsconfig.json` extends `@eusolicit/config` | `[P2] apps/admin/tsconfig.json extends @eusolicit/config/tsconfig.json` | P2 |
| `packages/ui/tsconfig.json` extends `@eusolicit/config` | `[P2] packages/ui/tsconfig.json extends @eusolicit/config/tsconfig.json` | P2 |
| `tsc --noEmit` exits 0 in apps/client (E03-P2-020) | `[P2] tsc --noEmit exits 0 with zero type errors in apps/client (E03-P2-020)` ⚠️ SLOW | P2 |
| `tsc --noEmit` exits 0 in apps/admin (E03-P2-020) | `[P2] tsc --noEmit exits 0 with zero type errors in apps/admin (E03-P2-020)` ⚠️ SLOW | P2 |

### AC8: ESLint + Prettier

| AC Detail | Test | Priority |
|-----------|------|----------|
| Root `frontend/package.json` references `@eusolicit/config` prettier config | `[P1] root frontend/package.json references shared Prettier config from @eusolicit/config` | P1 |
| Both apps `postcss.config.js` register tailwindcss + autoprefixer | `[P1] Both apps have postcss.config.js that registers tailwindcss and autoprefixer plugins` | P1 |
| `pnpm lint` exits 0 across all workspaces | `[P1] pnpm lint at frontend/ root exits 0 with zero ESLint failures across all workspaces` ⚠️ SLOW | P1 |

---

## Epic Test Design Traceability

| Epic Test Design Requirement | AC | Test(s) | Priority |
|------------------------------|-----|---------|----------|
| **E03-P0-001**: `pnpm build` succeeds for both apps — hard CI gate | AC6 | `[P0] pnpm build at frontend/ root exits with code 0...` | **P0** |
| **E03-P2-001**: Shared `<Button>` from `packages/ui` importable in both apps | AC2 | `[P0] packages/ui/index.ts exports Button component`, `[P2] apps/* depend on @eusolicit/ui` | P0 / P2 |
| **E03-P2-020**: TypeScript strict mode — zero type errors across all packages | AC7 | `[P2] packages/config/tsconfig.json declares strict: true`, `[P2] tsc --noEmit passes in apps/*` | P2 |
| E03 entry criteria: pnpm v8+ and Node.js 20 LTS | AC4/AC5 | `[P1] root frontend/package.json declares engines: node >=20 and pnpm >=8` | P1 |
| App Router (not Pages Router) — mandatory per epic spec | AC1 | `[P0] frontend/apps/client/ exists with App Router app/ subdirectory` | P0 |
| `transpilePackages: ['@eusolicit/ui']` critical for module resolution | AC1 | `[P0] Both apps next.config.ts declare transpilePackages` | P0 |

---

## Anti-Pattern Guards (from Story Dev Notes)

These tests directly guard against the "What NOT to Do" list in the story spec:

| Anti-Pattern | Guard Test |
|--------------|-----------|
| Using npm or yarn (must be pnpm) | Root package.json `engines.pnpm >=8` test (implicit) |
| Pinning Next.js to 15.x | `[P1] Both apps declare Next.js 14.x (NOT 15.x)` |
| Using Pages Router instead of App Router | No `pages/` directory check (implicit: `app/` must exist) |
| Using `next.config.js` or `next.config.mjs` | `[P0] Both apps use next.config.ts` + absence of .js/.mjs |
| Leaving old flat `frontend/app/` alongside new structure | `[P1] Old flat frontend/app/ directory removed` |
| Missing `postcss.config.js` (breaks Tailwind CSS processing) | `[P0] Both apps postcss.config.js exists` |
| `shamefully-hoist=true` in `.npmrc` | `[P1] .npmrc declares shamefully-hoist=false` |

---

## Fixture Needs

_None required for GREEN phase._ These tests are pure filesystem and subprocess assertions — no database, browser fixtures, or service mocks needed.

| Resource | Purpose | Status |
|----------|---------|--------|
| `pnpm` in PATH | Build command execution (AC6, AC8 SLOW tests) | Required at GREEN phase |
| `eusolicit-app/frontend/` directory | Target of all filesystem assertions | Must exist after Story 3.1 |
| Node.js `fs` module | File existence and content checks | Available (stdlib) |
| Node.js `child_process.spawnSync` | Build command execution | Available (stdlib) |

---

## Mock Requirements

_None._ Story 3.1 tests are scaffold and build verification — no HTTP requests, no service dependencies, no database access.

---

## Required data-testid Attributes

_Not applicable_ — all tests are filesystem, JSON content, and build command verification. No UI interactions.

---

## Slow Test Strategy

Three test groups execute real shell commands and take significant time:

| Test | Command | Expected Duration | Run Mode |
|------|---------|-------------------|----------|
| `[P0] pnpm build exits code 0` | `pnpm build` at `frontend/` | ~2–4 minutes | CI on every PR |
| `[P0] pnpm build — no TS errors` | Same `pnpm build` output | same run | CI on every PR |
| `[P0] pnpm build — no ESLint failures` | Same `pnpm build` output | same run | CI on every PR |
| `[P2] tsc --noEmit in apps/client` | `pnpm exec tsc --noEmit` | ~30 seconds | CI on every PR |
| `[P2] tsc --noEmit in apps/admin` | `pnpm exec tsc --noEmit` | ~30 seconds | CI on every PR |
| `[P1] pnpm lint exits 0` | `pnpm lint` at `frontend/` | ~1–2 minutes | CI on every PR |

**Playwright project:** `api` (no browser — uses `.api.spec.ts` file match pattern). Run slow tests in isolation:

```bash
# All Story 3.1 tests (slow tests included)
cd eusolicit-app
pnpm playwright test --project=api e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts

# Filesystem-only tests (fast, no build commands)
pnpm playwright test --project=api -g "AC1|AC2|AC3|AC4|AC5"

# Build gate only (E03-P0-001)
pnpm playwright test --project=api -g "E03-P0-001"
```

---

## Implementation Guidance

### What Must Be Created (to Turn Tests GREEN)

#### Turborepo Root (`eusolicit-app/frontend/`)
- `pnpm-workspace.yaml` — packages: `["apps/*", "packages/*"]`
- `turbo.json` — pipeline: build, dev, lint, type-check tasks
- `.npmrc` — `strict-peer-dependencies=false`, `shamefully-hoist=false`
- `package.json` — root workspace (private: true), turbo-delegating scripts, engines: node >=20 / pnpm >=8
- **REMOVE:** `next.config.mjs`, old flat `app/` directory

#### `packages/config/` — Shared Tooling Config
- `package.json` — `@eusolicit/config`, version `0.0.0`
- `tsconfig.json` — base TypeScript config with `strict: true`, target ES2017, moduleResolution bundler
- `tailwind.config.ts` — minimal scaffold: empty content array, extend stub
- `.eslintrc.js` — extends `next/core-web-vitals` + `@typescript-eslint/recommended`
- `prettier.config.js` — `singleQuote: false, semi: true, tabWidth: 2, trailingComma: "es5", printWidth: 100`

#### `packages/ui/` — Shared Component Library
- `package.json` — `@eusolicit/ui`, version `0.0.0`, peer deps: react, react-dom, next
- `tsconfig.json` — extends `@eusolicit/config/tsconfig.json`
- `index.ts` — `export { Button } from "./src/Button";`
- `src/Button.tsx` — minimal Button component with `variant` prop (primary | secondary)

#### `apps/client/` — Client Portal
- `package.json` — name: `client`, dev script: `next dev -p 3000`, next: `14.2.x`, deps: `@eusolicit/ui workspace:*`, `@eusolicit/config workspace:*`
- `next.config.ts` — `transpilePackages: ['@eusolicit/ui']`
- `tsconfig.json` — extends `@eusolicit/config/tsconfig.json`, paths `@/*` → `./src/*`
- `.eslintrc.js` — `root: true, extends: ["@eusolicit/config/.eslintrc.js"]`
- `app/layout.tsx` — metadata `title: "EU Solicit Client"`, `lang="bg"`
- `app/page.tsx` — placeholder page
- `app/globals.css` — `@tailwind base/components/utilities`
- `postcss.config.js` — tailwindcss + autoprefixer
- `lib/.gitkeep`, `components/.gitkeep` — placeholder directories

#### `apps/admin/` — Admin Portal
- Same structure as client, but name: `admin`, port 3001, title: `"EU Solicit Admin"`

### Critical Implementation Details

1. **`transpilePackages`**: Without this in `next.config.ts`, `@eusolicit/ui` imports fail with a module resolution error at build time — this is the #1 failure mode.

2. **`postcss.config.js`**: Each app needs this alongside `tailwind.config.ts` — without it, the `@tailwind` directives in `globals.css` fail to process and `pnpm build` emits warnings or fails.

3. **Old `frontend/app/` removal**: The flat Next.js 1.1 placeholder must be removed. Leaving `frontend/app/` alongside `frontend/apps/` breaks the workspace resolution.

4. **Next.js version pinning**: Use `"next": "14.2.x"` (or specific `14.2.29`) — NOT `^14`, NOT `15.x`. The epic spec is explicit.

5. **`packages/ui/index.ts` must exist**: Story spec states "Even if empty, the file must exist." However, the barrel export should export Button for E03-P2-001 coverage.

---

## Red-Green-Refactor Workflow

### RED Phase (Complete — TEA)

- [x] 44 acceptance test cases generated (44 test functions in 1 file)
- [x] All tests use `test.skip()` to document intentional RED phase
- [x] Tests assert EXPECTED behavior against unimplemented scaffold
- [x] Tests organized by acceptance criterion (AC1–AC8) in named describe blocks
- [x] P0 build gate test (E03-P0-001) and TypeScript strict test (E03-P2-020) explicitly linked
- [x] Anti-pattern guards from story spec included
- [x] Test file: `eusolicit-app/e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts`

### GREEN Phase (Next — Dev Team)

After implementing the scaffold, enable tests one AC group at a time:

1. Start with AC1 (structure), AC2 (packages/ui), AC3 (packages/config):
   ```bash
   # Remove test.skip() from AC1 describe block, run:
   pnpm playwright test --project=api -g "AC1"
   ```

2. Enable AC4/AC5 (port config) and AC7 (TypeScript strict — JSON checks):
   ```bash
   pnpm playwright test --project=api -g "AC4|AC5|AC7" --grep-invert "SLOW"
   ```

3. Run build gate (AC6 — slow, ~3-5 minutes for full build):
   ```bash
   pnpm playwright test --project=api -g "AC6"
   ```

4. Run lint gate (AC8 — slow):
   ```bash
   pnpm playwright test --project=api -g "AC8"
   ```

5. Full suite:
   ```bash
   pnpm playwright test --project=api e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts
   ```

If any tests fail:
- **Filesystem test fails:** Check that the required file/directory was created at the correct path
- **Content test fails:** Verify the file content matches the story spec (e.g., `lang="bg"`, `transpilePackages`, `strict: true`)
- **Build test fails:** Check for TypeScript errors, ESLint failures, or missing `postcss.config.js`

### REFACTOR Phase (Dev Team)

- Consider splitting slow build tests into a separate `*.build.spec.ts` suite and use Playwright tags to control CI scheduling
- Add workspace filter-specific build tests once E03-P0-001 is consistently green: `pnpm build --filter client` and `pnpm build --filter admin` separately
- Consider adding `pnpm exec tsc --noEmit` in `packages/ui/` as an additional AC7 coverage point

---

## Execution Commands

```bash
# Navigate to project root
cd eusolicit-app

# Run ALL Story 3.1 tests (currently all skipped — RED phase)
pnpm playwright test --project=api e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts

# Run by AC group (fast filesystem tests)
pnpm playwright test --project=api -g "AC1 — App Router"
pnpm playwright test --project=api -g "AC2 — packages/ui"
pnpm playwright test --project=api -g "AC3 — packages/config"
pnpm playwright test --project=api -g "AC4 & AC5"

# Run build gate only (E03-P0-001 — slow, ~3-5 min)
pnpm playwright test --project=api -g "E03-P0-001"

# Run TypeScript strict mode checks (E03-P2-020)
pnpm playwright test --project=api -g "E03-P2-020"

# Run lint checks (AC8)
pnpm playwright test --project=api -g "AC8 — ESLint"

# Run as part of the full smoke suite
pnpm playwright test --project=api e2e/specs/smoke/ --reporter=list
```

---

## Completion Summary

| Metric | Value |
|--------|-------|
| **Story ID** | 3-1-next-js-14-monorepo-scaffold |
| **Primary test level** | Build/CI + Scaffold (filesystem + subprocess assertions) |
| **Test functions** | 44 |
| **Test files** | 1 (`e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts`) |
| **Playwright project** | `api` (no browser — `.api.spec.ts` pattern) |
| **Fixtures needed** | 0 (pure filesystem/subprocess) |
| **Mock requirements** | 0 |
| **data-testid requirements** | 0 |
| **External dependencies** | `pnpm` in PATH (for slow build/lint tests) |
| **Execution mode** | Parallel-safe (no shared state) |
| **TDD phase** | RED (all 44 tests skipped) |

### Priority Breakdown

| Priority | Test Count | AC Coverage |
|----------|------------|-------------|
| **P0** | 22 | AC1 (8 tests), AC2 (5 tests), AC3 (4 tests), AC6 (5 tests) |
| **P1** | 15 | AC1 (5 tests), AC3 (2 tests), AC4/AC5 (5 tests), AC8 (3 tests) |
| **P2** | 7 | AC2 (1 test — E03-P2-001), AC7 (6 tests — E03-P2-020) |
| **P3** | 0 | — |
| **Total** | **44** | **AC1–AC8 (100%)** |

### Epic Test Design Coverage

| Epic Test ID | Priority | AC | Covered By | Gate |
|-------------|---------|-----|-----------|------|
| E03-P0-001 | **P0** | AC6 | `pnpm build exits code 0` | Hard CI fail |
| E03-P2-001 | P2 | AC2 | `@eusolicit/ui workspace dep` + Button export | — |
| E03-P2-020 | P2 | AC7 | `strict: true` + `tsc --noEmit` | — |

### Risk Mitigation Coverage

| Risk | Score | Mitigation | Test Coverage |
|------|-------|------------|---------------|
| E03-P0-001 build gate | — | Hard gate; all downstream E03 tests depend on it | 3 slow build tests + file prerequisite checks |
| `transpilePackages` omission | Critical | Explicitly tested: `next.config.ts` must contain `transpilePackages` + `@eusolicit/ui` | `[P0] Both apps next.config.ts declare transpilePackages` |
| `postcss.config.js` missing | High | Anti-pattern guard from story spec | `[P0] Both apps postcss.config.js exists` |
| Next.js 15.x pin (wrong version) | High | Anti-pattern guard from story spec | `[P1] Both apps declare Next.js 14.x` |
| Old flat `frontend/app/` left behind | Medium | Explicit cleanup check | `[P1] Old flat frontend/app/ directory removed` |

### Knowledge Base References Applied

- `test-levels-framework.md` — Build/CI level for scaffold stories; no browser or API needed
- `test-priorities-matrix.md` — P0 for build gate; P1 for structure; P2 for strict mode/shared imports
- `test-quality.md` — Deterministic assertions, parallel-safe, zero external dependencies
- Existing `scaffold-structure.api.spec.ts` pattern — followed for filesystem + subprocess test structure

### Next Steps

1. **Implement Story 3.1** — Create all scaffold files per the story spec (see Implementation Guidance)
2. **Remove `test.skip()`** — One AC group at a time; verify GREEN phase per group
3. **Run full suite** — `pnpm playwright test --project=api e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts`
4. **Verify CI** — Ensure `pnpm build` gate (E03-P0-001) is part of the PR check workflow
5. **Proceed to Story 3.2** — Tailwind CSS design-token preset (depends on packages/config from this story)

---

**Generated by:** BMad TEA Agent — ATDD Module
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
