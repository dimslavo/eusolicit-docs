# Story 3.1: Next.js 14 Monorepo Scaffold

Status: done

## Story

As a **frontend developer on the EU Solicit team**,
I want **a Turborepo monorepo under `frontend/` with two Next.js 14 App Router applications (`client` and `admin`) and shared `packages/ui` and `packages/config` packages**,
so that **every subsequent frontend story drops UI into a properly structured, type-safe monorepo with shared tooling, consistent configuration, and clean `pnpm build` output**.

## Acceptance Criteria

1. `frontend/apps/client` and `frontend/apps/admin` each contain a Next.js 14 App Router project (`app/` directory enabled, not Pages Router)
2. `frontend/packages/ui` exists as a shared component library package with `index.ts` barrel export
3. `frontend/packages/config` exports shared `tailwind.config.ts`, `tsconfig.json`, and `.eslintrc.js`
4. `pnpm dev --filter client` starts the client app on port **3000** without errors
5. `pnpm dev --filter admin` starts the admin app on port **3001** without errors
6. `pnpm build` at `frontend/` root succeeds for **both** apps (exit code 0, no TypeScript errors, no ESLint failures)
7. TypeScript strict mode enabled in all apps and shared packages
8. ESLint + Prettier configured and `pnpm lint` passes across all workspaces

## Tasks / Subtasks

- [x]Task 1: Replace existing `frontend/` placeholder with Turborepo structure (AC: 1, 2, 3)
  - [x]1.1 Remove existing flat `frontend/` structure (package.json, tsconfig.json, next.config.mjs, app/) — migrate content to new structure
  - [x]1.2 Create `frontend/pnpm-workspace.yaml` referencing `apps/*` and `packages/*`
  - [x]1.3 Create `frontend/package.json` (root workspace, no private apps here — just scripts and devDeps for turbo)
  - [x]1.4 Create `frontend/turbo.json` with `build`, `dev`, `lint`, `type-check` pipeline tasks
  - [x]1.5 Create `frontend/.npmrc` with `strict-peer-dependencies=false` and `shamefully-hoist=false`

- [x]Task 2: Create `frontend/packages/config` shared configuration package (AC: 3, 7, 8)
  - [x]2.1 `packages/config/package.json` — name `@eusolicit/config`, version `0.0.0`
  - [x]2.2 `packages/config/tailwind.config.ts` — base preset (slate neutrals, indigo-600 primary, spacing base 4px) — just the scaffold for Story 3.2; keep it minimal here
  - [x]2.3 `packages/config/tsconfig.json` — strict TypeScript base config for sharing across apps
  - [x]2.4 `packages/config/.eslintrc.js` — base ESLint config (Next.js + TypeScript rules)
  - [x]2.5 `packages/config/prettier.config.js` — shared Prettier config (singleQuote, semi, tabWidth: 2)

- [x]Task 3: Create `frontend/packages/ui` shared component library (AC: 2)
  - [x]3.1 `packages/ui/package.json` — name `@eusolicit/ui`, version `0.0.0`, type `module`; peer deps: `react`, `react-dom`, `next`
  - [x]3.2 `packages/ui/tsconfig.json` — extends `@eusolicit/config/tsconfig.json`
  - [x]3.3 `packages/ui/index.ts` — barrel export file (empty export placeholder: `export {};`)
  - [x]3.4 `packages/ui/src/` directory with placeholder `Button.tsx` component (simple `<button>` wrapper)

- [x]Task 4: Create `frontend/apps/client` Next.js 14 app (AC: 1, 4, 6, 7, 8)
  - [x]4.1 `apps/client/package.json` — name `client`, depends on `@eusolicit/ui` and `@eusolicit/config` (workspace:*)
  - [x]4.2 `apps/client/next.config.ts` — App Router, `transpilePackages: ['@eusolicit/ui']`
  - [x]4.3 `apps/client/tsconfig.json` — extends `@eusolicit/config/tsconfig.json`, adds paths `@/*` → `./src/*`
  - [x]4.4 `apps/client/.eslintrc.js` — extends `@eusolicit/config`
  - [x]4.5 `apps/client/app/layout.tsx` — Root layout with `<html lang="bg">`, metadata `title: "EU Solicit Client"`
  - [x]4.6 `apps/client/app/page.tsx` — placeholder home page
  - [x]4.7 `apps/client/app/globals.css` — base CSS with Tailwind directives `@tailwind base/components/utilities`
  - [x]4.8 Add `lib/` and `components/` directories with `.gitkeep` placeholders
  - [x]4.9 Configure dev port 3000 in `next.config.ts` or `package.json` dev script (`next dev -p 3000`)

- [x]Task 5: Create `frontend/apps/admin` Next.js 14 app (AC: 1, 5, 6, 7, 8)
  - [x]5.1 `apps/admin/package.json` — name `admin`, depends on `@eusolicit/ui` and `@eusolicit/config` (workspace:*)
  - [x]5.2 `apps/admin/next.config.ts` — App Router, `transpilePackages: ['@eusolicit/ui']`
  - [x]5.3 `apps/admin/tsconfig.json` — extends `@eusolicit/config/tsconfig.json`, paths `@/*` → `./src/*`
  - [x]5.4 `apps/admin/.eslintrc.js` — extends `@eusolicit/config`
  - [x]5.5 `apps/admin/app/layout.tsx` — Root layout with `<html lang="bg">`, metadata `title: "EU Solicit Admin"`
  - [x]5.6 `apps/admin/app/page.tsx` — placeholder home page
  - [x]5.7 `apps/admin/app/globals.css` — base CSS with Tailwind directives
  - [x]5.8 Add `lib/` and `components/` directories with `.gitkeep` placeholders
  - [x]5.9 Configure dev port 3001 (`next dev -p 3001`)

- [x]Task 6: Install dependencies and verify build (AC: 4, 5, 6, 8)
  - [x]6.1 Run `pnpm install` from `frontend/` root — verify no errors
  - [x]6.2 Verify `pnpm dev --filter client` starts on port 3000 (at least hot-reload server starts; no browser test needed)
  - [x]6.3 Verify `pnpm dev --filter admin` starts on port 3001
  - [x]6.4 Run `pnpm build` at `frontend/` root — both apps must exit 0
  - [x]6.5 Run `pnpm lint` — zero ESLint failures across all workspaces
  - [x]6.6 Verify TypeScript strict mode: `pnpm exec tsc --noEmit` in both apps exits 0

## Dev Notes

### CRITICAL: Existing `frontend/` Is a Placeholder — Must Be Restructured

**Story 1.1 created a flat Next.js placeholder** at `eusolicit-app/frontend/` with these files:
```
frontend/
├── package.json          # flat app, name: "eusolicit-frontend", next ^14.2.0
├── tsconfig.json         # strict: true already set
├── next.config.mjs       # simple config
└── app/
    ├── layout.tsx        # basic layout
    └── page.tsx          # placeholder page
```

**This story REPLACES the flat structure with a Turborepo monorepo.** The dev agent must:
1. Remove (or move) the existing `frontend/package.json`, `tsconfig.json`, `next.config.mjs`, and `app/` directory
2. Create the new Turborepo structure under `frontend/`
3. Migrate `app/layout.tsx` content to `apps/client/app/layout.tsx` (reuse the metadata pattern)

**Do NOT leave the old flat files alongside the new structure.** The old `frontend/app/` directory must be removed once content is migrated.

### Target Directory Structure

```
eusolicit-app/frontend/
├── apps/
│   ├── client/                     # Next.js 14 App Router, port 3000
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   └── globals.css
│   │   ├── lib/                    # (empty, placeholder for future use)
│   │   ├── components/             # (empty, placeholder for future use)
│   │   ├── package.json
│   │   ├── next.config.ts
│   │   ├── tsconfig.json
│   │   └── .eslintrc.js
│   └── admin/                      # Next.js 14 App Router, port 3001
│       ├── app/
│       │   ├── layout.tsx
│       │   ├── page.tsx
│       │   └── globals.css
│       ├── lib/
│       ├── components/
│       ├── package.json
│       ├── next.config.ts
│       ├── tsconfig.json
│       └── .eslintrc.js
├── packages/
│   ├── config/                     # Shared tooling config
│   │   ├── package.json            # name: "@eusolicit/config"
│   │   ├── tailwind.config.ts      # base preset (extend in apps)
│   │   ├── tsconfig.json           # strict TypeScript base
│   │   ├── .eslintrc.js            # shared ESLint rules
│   │   └── prettier.config.js      # shared Prettier
│   └── ui/                         # Shared component library
│       ├── package.json            # name: "@eusolicit/ui"
│       ├── tsconfig.json
│       ├── index.ts                # REQUIRED barrel export
│       └── src/
│           └── Button.tsx          # placeholder component
├── package.json                    # root workspace (private: true)
├── pnpm-workspace.yaml
├── turbo.json
└── .npmrc
```

### pnpm Version & Node.js Requirements

- **pnpm**: 8+ (from E03 entry criteria — `pnpm v8+ and Node.js 20 LTS available in CI runner`)
- **Node.js**: 20 LTS (existing `.nvmrc` at repo root specifies v22, which is also ≥20 LTS)
- **Next.js**: Pin to **14.x** (NOT 15.x — S03.01 explicitly says "Pin Next.js to 14.x")

**Use exact version pins for framework dependencies to match architecture spec:**
```json
"next": "14.2.x"  // NOT ^14, NOT 15.x
```

### Root `frontend/package.json` Pattern

```json
{
  "name": "eusolicit-frontend",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "build": "turbo build",
    "dev": "turbo dev",
    "lint": "turbo lint",
    "type-check": "turbo type-check",
    "format": "prettier --write \"**/*.{ts,tsx,md}\""
  },
  "devDependencies": {
    "turbo": "latest",
    "prettier": "^3.0.0"
  },
  "engines": {
    "node": ">=20",
    "pnpm": ">=8"
  }
}
```

### `pnpm-workspace.yaml` Pattern

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

### `turbo.json` Pattern

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "dist/**"]
    },
    "dev": {
      "dependsOn": ["^build"],
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^lint"],
      "outputs": []
    },
    "type-check": {
      "dependsOn": ["^build"],
      "outputs": []
    }
  }
}
```

### `.npmrc` Pattern

```
strict-peer-dependencies=false
shamefully-hoist=false
```

### `packages/config/tsconfig.json` — Strict TypeScript Base

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "display": "Default",
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": false,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }]
  },
  "exclude": ["node_modules"]
}
```

Each app's `tsconfig.json` extends this:
```json
{
  "extends": "@eusolicit/config/tsconfig.json",
  "compilerOptions": {
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### `packages/config/.eslintrc.js` Pattern

```js
module.exports = {
  extends: [
    "next/core-web-vitals",
    "plugin:@typescript-eslint/recommended"
  ],
  rules: {
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/no-explicit-any": "warn"
  }
};
```

App `.eslintrc.js`:
```js
module.exports = {
  root: true,
  extends: ["@eusolicit/config/.eslintrc.js"]
};
```

### `packages/config/tailwind.config.ts` — Base Preset Scaffold

This is just the scaffold; the full design token system is Story 3.2. Keep minimal:

```typescript
import type { Config } from "tailwindcss";

const config: Config = {
  content: [],
  theme: {
    extend: {}
  },
  plugins: []
};

export default config;
```

Apps extend it:
```typescript
import type { Config } from "tailwindcss";
import baseConfig from "@eusolicit/config/tailwind.config";

const config: Config = {
  ...baseConfig,
  content: [
    "./src/**/*.{js,ts,jsx,tsx,mdx}",
    "./app/**/*.{js,ts,jsx,tsx,mdx}",
    "./components/**/*.{js,ts,jsx,tsx,mdx}",
    "../../packages/ui/src/**/*.{js,ts,jsx,tsx,mdx}"
  ]
};

export default config;
```

### `packages/ui/package.json` Pattern

```json
{
  "name": "@eusolicit/ui",
  "version": "0.0.0",
  "private": true,
  "main": "./index.ts",
  "types": "./index.ts",
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "type-check": "tsc --noEmit"
  },
  "devDependencies": {
    "@eusolicit/config": "workspace:*",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "typescript": "^5.4.0"
  },
  "peerDependencies": {
    "next": ">=14.0.0",
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  }
}
```

**IMPORTANT: `packages/ui/index.ts` must exist.** S03.01 implementation note explicitly requires `packages/ui/index.ts` barrel export. Even if empty, the file must exist:

```typescript
// packages/ui/index.ts
export { Button } from "./src/Button";
```

### `apps/client/package.json` and `apps/admin/package.json` Pattern

```json
{
  "name": "client",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev -p 3000",
    "build": "next build",
    "start": "next start -p 3000",
    "lint": "next lint",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "@eusolicit/config": "workspace:*",
    "@eusolicit/ui": "workspace:*",
    "next": "14.2.29",
    "react": "^18.3.0",
    "react-dom": "^18.3.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "autoprefixer": "^10.4.0",
    "eslint": "^8.0.0",
    "eslint-config-next": "14.2.29",
    "postcss": "^8.4.0",
    "tailwindcss": "^3.4.0",
    "typescript": "^5.4.0"
  }
}
```

Admin: same but `"name": "admin"` and port 3001 in dev/start scripts.

### `apps/client/next.config.ts` Pattern

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  transpilePackages: ["@eusolicit/ui"],
};

export default nextConfig;
```

**NOTE:** Using `next.config.ts` (TypeScript) not `next.config.js` or `next.config.mjs`. Next.js 14.2+ supports `next.config.ts`.

### Port Assignments

Per epic S03.01 implementation notes:
- **client app**: port **3000**
- **admin app**: port **3001**

These must be in the `dev` and `start` scripts (`-p 3000` / `-p 3001`).

### Placeholder `Button.tsx` Component

```typescript
// packages/ui/src/Button.tsx
import type { ButtonHTMLAttributes } from "react";

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: "primary" | "secondary";
}

export function Button({ variant = "primary", children, ...props }: ButtonProps) {
  return (
    <button
      {...props}
      className={`inline-flex items-center justify-center rounded-md px-4 py-2 text-sm font-medium transition-colors ${
        variant === "primary"
          ? "bg-indigo-600 text-white hover:bg-indigo-700"
          : "bg-slate-100 text-slate-900 hover:bg-slate-200"
      } ${props.className ?? ""}`}
    >
      {children}
    </button>
  );
}
```

This placeholder is intentionally minimal. Story 3.2 will replace it with the full shadcn/ui theming.

### app/layout.tsx Pattern (both apps)

```typescript
// apps/client/app/layout.tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "EU Solicit Client",
  description: "EU Solicit Client Portal",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="bg">
      <body>{children}</body>
    </html>
  );
}
```

**Admin** uses `title: "EU Solicit Admin"`, `description: "EU Solicit Administration Panel"`.

Note: `lang="bg"` (Bulgarian) per the default locale for EU Solicit (next-intl default locale = `bg`). Story 3.7 will convert this to a dynamic `[locale]` layout.

### globals.css Pattern

```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### ESLint + Prettier Configuration

`packages/config/prettier.config.js`:
```js
module.exports = {
  semi: true,
  singleQuote: false,
  tabWidth: 2,
  trailingComma: "es5",
  printWidth: 100,
};
```

Root `frontend/package.json` should also add:
```json
"prettier": "@eusolicit/config/prettier.config.js"
```

### TypeScript Strict Mode Requirements

Both apps and `packages/ui` must have `"strict": true` in their effective TypeScript config (inherited from `packages/config/tsconfig.json`). This means:
- `strictNullChecks: true` (implicit with `strict: true`)
- `noImplicitAny: true` (implicit with `strict: true`)
- `strictFunctionTypes: true` (implicit)

Run `pnpm exec tsc --noEmit` in each app and `packages/ui` — must exit 0 with zero errors.

### Turborepo Filter Commands

```bash
# Start client only
pnpm dev --filter client

# Start admin only
pnpm dev --filter admin

# Build everything
pnpm build

# Lint all workspaces
pnpm lint
```

The `--filter` flag is Turborepo's workspace selector. For pnpm + Turborepo, the filter works on the `name` field in `package.json`, so `client` (not `apps/client`).

### What NOT to Do (Anti-Patterns)

- **Do NOT use npm or yarn** — this project uses pnpm exclusively (E03 test design entry criteria specifies `pnpm v8+`)
- **Do NOT pin Next.js to 15.x** — must be 14.x per epic specification
- **Do NOT use the Pages Router** — App Router (`app/` directory) is mandatory
- **Do NOT create `next.config.js` or `next.config.mjs`** — use `next.config.ts` (TypeScript config)
- **Do NOT leave the old flat `frontend/` structure** alongside the new structure — clean up placeholder files
- **Do NOT add `"type": "module"` to app package.json** — Next.js doesn't require this and it can cause CommonJS interop issues
- **Do NOT use `shamefully-hoist=true` in `.npmrc`** — pnpm workspaces should use strict hoisting
- **Do NOT create a `postcss.config.js` without ensuring Tailwind works** — each app needs `postcss.config.js` or `postcss.config.ts` alongside `tailwind.config.ts` for CSS processing

### PostCSS Configuration (Required for Tailwind)

Each app needs `postcss.config.js`:
```js
// apps/client/postcss.config.js (and apps/admin/postcss.config.js)
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

Without this, `globals.css` will fail to process Tailwind directives and `pnpm build` will emit warnings or fail.

### Project Structure Notes

**Working directory context:**
- All frontend code lives under `eusolicit-app/frontend/` (not project root `frontend/`)
- The full path is `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`
- This story operates ONLY within `eusolicit-app/frontend/`
- Do NOT modify any backend service files, shared Python packages, or CI config

**Alignment with unified project structure:**
- `eusolicit-app/frontend/` → Turborepo monorepo root (this story)
- `eusolicit-app/frontend/apps/client/` → Client portal Next.js app
- `eusolicit-app/frontend/apps/admin/` → Admin Next.js app
- `eusolicit-app/frontend/packages/ui/` → Shared component library (used by all future stories in E03)
- `eusolicit-app/frontend/packages/config/` → Shared tooling config

**No changes to:**
- `eusolicit-app/pyproject.toml` (Python workspace config — backend only)
- `eusolicit-app/services/` (Python services)
- `eusolicit-app/packages/` (Python packages)
- `eusolicit-app/e2e/` (Playwright E2E framework — already configured)
- `eusolicit-app/.github/` (CI workflows)

**The existing `playwright.config.ts` at repo root** — the E2E framework uses Playwright. This story's apps will eventually be tested with Playwright (E03-P0-009, P0-010, etc.) but no changes to playwright.config.ts are needed for this story. The build gate (E03-P0-001) is what matters here.

### Test Expectations (from E03 Test Design)

This story directly enables the following tests:

| Priority | Test ID | Requirement | How to Verify |
|----------|---------|-------------|---------------|
| **P0** | E03-P0-001 | `pnpm build` succeeds for both `client` and `admin` — no TypeScript errors, no ESLint failures, no import resolution errors | Run `pnpm build` at `frontend/` root; both apps must exit 0 |
| **P2** | E03-P2-020 | TypeScript strict mode: zero type errors across all packages and apps | `tsc --noEmit` passes in both apps and `packages/ui` |
| **P2** | E03-P2-001 | Shared `<Button>` from `packages/ui` imports and renders without errors in both apps | Import `Button` from `@eusolicit/ui` in each app's page.tsx; `pnpm build` exits 0 |

**Test risk context from E03 test design:**
- E03-P0-001 is a **hard gate** — CI fails the PR if either app fails `pnpm build`
- This is the foundation test; all 51 other E03 tests depend on this story's scaffold being correct
- The `transpilePackages: ['@eusolicit/ui']` setting in `next.config.ts` is critical — without it, the `@eusolicit/ui` import will fail with a module resolution error

### Deferred Work Awareness

From `deferred-work.md` (Story 1.1 review):
> Frontend declares tailwindcss devDependency but missing config files [frontend/] — `package.json` lists tailwindcss, postcss, autoprefixer but no `tailwind.config.js`, `postcss.config.js`, or `app/globals.css` with `@tailwind` directives exist.

This story resolves that deferred item: the new structure must include proper `tailwind.config.ts` (in `packages/config/`), `postcss.config.js` (in each app), and `globals.css` with `@tailwind` directives.

### References

- [Source: eusolicit-docs/planning-artifacts/epic-03-frontend-shell-design-system.md#S03.01]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#P0, P2]
- [Source: eusolicit-docs/test-artifacts/test-design-architecture.md#testability-concerns]
- [Source: eusolicit-docs/project-context.md#project-structure-reference]
- [Source: eusolicit-docs/implementation-artifacts/1-1-monorepo-scaffold-project-structure.md#frontend-placeholder]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (claude-code)

### Debug Log References

None — all implementation issues resolved inline.

### Completion Notes List

1. **next.config.ts not supported in Next.js 14.2.x** — TypeScript config files (`next.config.ts`) require Next.js 15+. Used `next.config.mjs` (ESM format) instead. ATDD tests updated to reflect this reality.

2. **Turborepo v2 renamed `pipeline` to `tasks`** — The installed `turbo@2.9.5` requires `tasks` in `turbo.json` instead of `pipeline`. Story spec pattern updated; ATDD test updated to check `turbo['tasks'] ?? turbo['pipeline']` for compatibility.

3. **ESLint scoped-package path resolution quirk** — ESLint v8 normalises `@eusolicit/config/.eslintrc.js` to `@eusolicit/eslint-config-config/.eslintrc.js` (per its scoped-package convention), which doesn't exist. Fixed by using `require.resolve("@eusolicit/config/.eslintrc.js")` in app and package `.eslintrc.js` files — this resolves at Node module load time to an absolute path that ESLint can load directly.

4. **`packageManager` field required by Turborepo v2** — Added `"packageManager": "pnpm@10.33.0"` to root `package.json`.

5. **ATDD build tests run serially** — The three `pnpm build` tests in AC6 were made `test.describe.serial` to prevent concurrent builds colliding on the `.next/export` directory.

6. **ESLint added to packages/ui devDependencies** — The `packages/ui` lint script requires `eslint` binary to be available directly in its devDependencies (pnpm strict hoisting doesn't hoist from `@eusolicit/config`).

### File List

- `eusolicit-app/frontend/package.json` (root workspace)
- `eusolicit-app/frontend/pnpm-workspace.yaml`
- `eusolicit-app/frontend/turbo.json`
- `eusolicit-app/frontend/.npmrc`
- `eusolicit-app/frontend/packages/config/package.json`
- `eusolicit-app/frontend/packages/config/tsconfig.json`
- `eusolicit-app/frontend/packages/config/tailwind.config.ts`
- `eusolicit-app/frontend/packages/config/.eslintrc.js`
- `eusolicit-app/frontend/packages/config/prettier.config.js`
- `eusolicit-app/frontend/packages/ui/package.json`
- `eusolicit-app/frontend/packages/ui/tsconfig.json`
- `eusolicit-app/frontend/packages/ui/.eslintrc.js`
- `eusolicit-app/frontend/packages/ui/index.ts`
- `eusolicit-app/frontend/packages/ui/src/Button.tsx`
- `eusolicit-app/frontend/apps/client/package.json`
- `eusolicit-app/frontend/apps/client/next.config.mjs`
- `eusolicit-app/frontend/apps/client/tsconfig.json`
- `eusolicit-app/frontend/apps/client/.eslintrc.js`
- `eusolicit-app/frontend/apps/client/postcss.config.js`
- `eusolicit-app/frontend/apps/client/tailwind.config.ts`
- `eusolicit-app/frontend/apps/client/app/layout.tsx`
- `eusolicit-app/frontend/apps/client/app/page.tsx`
- `eusolicit-app/frontend/apps/client/app/globals.css`
- `eusolicit-app/frontend/apps/client/lib/.gitkeep`
- `eusolicit-app/frontend/apps/client/components/.gitkeep`
- `eusolicit-app/frontend/apps/admin/package.json`
- `eusolicit-app/frontend/apps/admin/next.config.mjs`
- `eusolicit-app/frontend/apps/admin/tsconfig.json`
- `eusolicit-app/frontend/apps/admin/.eslintrc.js`
- `eusolicit-app/frontend/apps/admin/postcss.config.js`
- `eusolicit-app/frontend/apps/admin/tailwind.config.ts`
- `eusolicit-app/frontend/apps/admin/app/layout.tsx`
- `eusolicit-app/frontend/apps/admin/app/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/globals.css`
- `eusolicit-app/frontend/apps/admin/lib/.gitkeep`
- `eusolicit-app/frontend/apps/admin/components/.gitkeep`
- `eusolicit-app/e2e/specs/smoke/frontend-monorepo-scaffold.api.spec.ts` (ATDD GREEN — 44/44 tests pass)

## Senior Developer Review

**Date:** 2026-04-08
**Reviewer:** claude-opus-4 (bmad-code-review)
**Review Mode:** Full (3-layer adversarial: Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** APPROVED

### Acceptance Criteria Verification

| AC | Status | Notes |
|----|--------|-------|
| AC1 — App Router dirs | PASS | Both `apps/client/app/` and `apps/admin/app/` exist with layout.tsx, page.tsx, globals.css |
| AC2 — packages/ui barrel | PASS | `index.ts` exports Button from `./src/Button` |
| AC3 — packages/config exports | PASS | tailwind.config.ts, tsconfig.json, .eslintrc.js, prettier.config.js present |
| AC4 — client port 3000 | PASS | `"dev": "next dev -p 3000"` |
| AC5 — admin port 3001 | PASS | `"dev": "next dev -p 3001"` |
| AC6 — pnpm build exits 0 | PASS | Verified by Acceptance Auditor (live) — both apps build successfully |
| AC7 — TypeScript strict | PASS | `strict: true` in base tsconfig; all apps/packages extend it |
| AC8 — ESLint + Prettier | PASS | Verified by Acceptance Auditor (live) — `pnpm lint` zero errors |

### Review Findings

- [x] [Review][Defer] `turbo: "latest"` non-reproducible builds — spec-compliant but every fresh `pnpm install` may pull a different major version [frontend/package.json] — deferred, spec design choice
- [x] [Review][Defer] `@eusolicit/config` missing `exports` field — subpath imports (tailwind.config, prettier.config.js, tsconfig.json) rely on pnpm symlink resolution; will break under strict ESM [packages/config/package.json] — deferred, pre-existing
- [x] [Review][Defer] `@/*` path alias maps to `./src/*` but no `src/` directory exists in either app — unused scaffold alias; developers using `@/` imports will get resolution errors [apps/*/tsconfig.json] — deferred, spec design choice
- [x] [Review][Defer] UI package raw TypeScript entrypoint with no build step — works via `transpilePackages` but limits consumers to Next.js only [packages/ui/package.json] — deferred, spec design choice
- [x] [Review][Defer] `@typescript-eslint` v6 is end-of-life — no longer receives patches; blocks ESLint 9 flat config migration [packages/config/package.json, packages/ui/package.json] — deferred, pre-existing
- [x] [Review][Defer] `engines.pnpm >=8` inconsistent with `packageManager: pnpm@10.33.0` — engines implies pnpm 8/9 compatibility but lockfile v9 format requires pnpm 9+ [frontend/package.json] — deferred, minor inconsistency
- [x] [Review][Defer] Tailwind config shallow spread discards base config additions — `...baseConfig` followed by local `content` override silently drops any future base `content` or `theme.extend` entries [apps/*/tailwind.config.ts] — deferred, Story 3.2 will establish full theme system
- [x] [Review][Defer] No `.gitignore` in `frontend/` directory — `.next/`, `node_modules/`, `.turbo/` should be ignored; may rely on parent-level gitignore [frontend/] — deferred, not in story scope

### Review Statistics

- **Layers completed:** 3/3 (Blind Hunter, Edge Case Hunter, Acceptance Auditor)
- **Total findings raised:** 25
- **Decision needed:** 0
- **Patch:** 0
- **Deferred:** 8
- **Dismissed:** 17 (noise, false positives, or spec-compliant by design)

### Documented Deviations Acknowledged

All 6 dev agent completion notes are valid and well-documented:
1. `next.config.mjs` instead of `next.config.ts` — correct, Next.js 14.2.x doesn't support TS config
2. `tasks` instead of `pipeline` in turbo.json — correct for Turborepo v2
3. `require.resolve()` ESLint workaround — correct for pnpm scoped-package resolution
4. `packageManager` field — required by Turborepo v2
5. ATDD serial build tests — correct to avoid `.next/` directory collisions
6. ESLint in packages/ui devDependencies — correct for pnpm strict hoisting
