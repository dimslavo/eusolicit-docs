# Story 3.10: Loading States, Error Boundaries & Empty States

Status: review

## Story

As a **frontend developer on the EU Solicit team**,
I want **reusable skeleton loader variants, a global error boundary, per-section error boundary component, empty state variants, and a `<QueryGuard>` wrapper — all exported from `packages/ui` and wired into the client app's Next.js App Router**,
so that **every feature page that loads data automatically gets consistent, layout-matched loading skeletons, graceful error recovery, and friendly empty-state messaging without rebuilding these patterns per feature**.

## Acceptance Criteria

1. **Skeleton variants** — five layout-matched skeleton components exported from `packages/ui`, all using `animate-pulse` with `bg-slate-200` background (overriding shadcn's `bg-muted` default via `className`):
   - `<SkeletonCard />` — card-shaped placeholder: slate-200 header strip (~40% width, h-5), body area (h-4 lines, 100%/80%/60% widths), optional footer strip; matches `<Card>` dimensions
   - `<SkeletonTable rows={number} columns={number} />` — configurable grid of rows × columns cells, each `h-4 rounded bg-slate-200 animate-pulse`; header row cells slightly taller (`h-5`)
   - `<SkeletonList items={number} />` — vertical list of `items` rows; each row: circular avatar placeholder (h-10 w-10, `rounded-full`) + two text lines (h-4 full-width and h-3 60%-width) side by side
   - `<SkeletonText lines={number} />` — configurable paragraph placeholder; each line `h-4 bg-slate-200 rounded animate-pulse`; last line 60% width; 8px gap between lines
   - `<SkeletonAvatar size?="sm"|"md"|"lg" />` — circular avatar placeholder; sm=h-8 w-8, md=h-10 w-10 (default), lg=h-12 w-12; `rounded-full bg-slate-200 animate-pulse`
   - All five components accept `className` prop for additional Tailwind overrides

2. **Global error boundary** — Next.js App Router `error.tsx` at `apps/client/app/[locale]/error.tsx`:
   - Must be a `"use client"` component (required by Next.js for `error.tsx`)
   - Receives `error: Error & { digest?: string }` and `reset: () => void` props
   - Renders: indigo-600 alert icon (Lucide `AlertCircle`, h-12 w-12), translated heading (`t("errors.somethingWentWrong")`), translated description (`t("errors.tryAgainDescription")`), primary "Try again" `<Button>` that calls `reset()`, optional collapsible error details section (Lucide `ChevronDown` toggle, shows `error.message` and `error.digest` in `<code>` styled text when expanded)
   - Entire error UI centred vertically and horizontally in the viewport (`flex min-h-[60vh] flex-col items-center justify-center gap-4`)
   - Add `data-testid="global-error-boundary"` on the root `<div>`, `data-testid="error-try-again"` on the "Try again" button, `data-testid="error-details-toggle"` on the details toggle

3. **Per-section `<ErrorBoundary>` component** — exported from `packages/ui`:
   - React class component (or wrapping `react-error-boundary` `ErrorBoundary`) that accepts `fallback?: React.ReactNode` and `children`
   - If `fallback` not provided, renders a default inline error card: red-50 background, red-200 border, Lucide `AlertTriangle` icon (h-5 w-5 text-red-500), short "Something went wrong" message, no "Try again" button (section-level only shows passive error)
   - One section crashing must NOT unmount siblings or the shell; each `<ErrorBoundary>` isolates its subtree
   - Add `data-testid="section-error-boundary"` on the default fallback root element

4. **`<EmptyState>` component** — exported from `packages/ui`:
   - Props: `icon: React.ElementType` (Lucide icon component), `title: string`, `description?: string`, `action?: { label: string; onClick: () => void }`, `className?: string`
   - Layout: centred flex column; icon rendered at h-12 w-12 `text-slate-400`; title `text-lg font-semibold text-slate-900`; description `text-sm text-slate-500`; action rendered as `<Button variant="default">` if provided
   - Add `data-testid="empty-state"` on root, `data-testid="empty-state-action"` on the action button

5. **Three pre-built empty state variant components** — exported from `packages/ui`:
   - `<EmptyStateNoResults onClearFilters?: () => void />` — Lucide `SearchX` icon, title from `t("states.noResultsTitle")` ("No results found"), description `t("states.noResultsDescription")` ("Try adjusting your search or filters"), action button "Clear filters" visible only when `onClearFilters` is provided
   - `<EmptyStateGetStarted title: string, description?: string, action: { label: string; onClick: () => void } />` — Lucide `PlusCircle` icon; fully customisable title/description/action via props (no hardcoded strings in this variant)
   - `<EmptyStateNoAccess />` — Lucide `ShieldOff` icon, title `t("states.noAccessTitle")` ("Access restricted"), description `t("states.noAccessDescription")` ("You don't have permission to view this content"), action button "Request access" calls `t("states.requestAccess")` label and links to `mailto:support@eusolicit.com` (opens email client via `window.location.href`)

6. **`<QueryGuard>` component** — exported from `packages/ui`:
   - Props: `isLoading: boolean`, `isError: boolean`, `isEmpty: boolean`, `error?: unknown`, `skeleton?: React.ReactNode`, `emptyState?: React.ReactNode`, `children: React.ReactNode`
   - Render logic (in order): if `isLoading` → render `skeleton` (or a default `<SkeletonList items={3} />` fallback); else if `isError` → render a default inline error card similar to the section ErrorBoundary default fallback (NOT the per-section `<ErrorBoundary>`, since TanStack Query errors are not React render errors); else if `isEmpty` → render `emptyState` (or `<EmptyStateNoResults />` fallback); else → render `children`
   - Add `data-testid="query-guard-loading"` on the loading wrapper, `data-testid="query-guard-error"` on the error wrapper, `data-testid="query-guard-empty"` on the empty wrapper

7. **New i18n keys** — add a `states` namespace to both `apps/client/messages/en.json` and `apps/client/messages/bg.json` alongside the existing namespaces; also add two keys to the existing `errors` namespace:
   - New `states` namespace (EN):
     - `"noResultsTitle"`: `"No results found"`
     - `"noResultsDescription"`: `"Try adjusting your search or filters"`
     - `"clearFilters"`: `"Clear filters"`
     - `"noAccessTitle"`: `"Access restricted"`
     - `"noAccessDescription"`: `"You don't have permission to view this content"`
     - `"requestAccess"`: `"Request access"`
   - New `errors` namespace keys (EN):
     - `"somethingWentWrong"`: `"Something went wrong"`
     - `"tryAgainDescription"`: `"An unexpected error occurred. You can try reloading this section or go back to the previous page."`
   - BG translations must cover all the same keys with Bulgarian text
   - Run `pnpm check:i18n --filter client` — must exit 0 after changes

8. **`/dev/ui-states` demo page** — `apps/client/app/[locale]/dev/ui-states/page.tsx`:
   - `"use client"` page accessible at `/{locale}/dev/ui-states`
   - Renders five labelled sections, each with a heading:
     1. "Skeleton Variants" — renders all five skeletons (`SkeletonCard`, `SkeletonTable rows={3} columns={4}`, `SkeletonList items={3}`, `SkeletonText lines={4}`, `SkeletonAvatar` in all three sizes) side by side with labels
     2. "Empty States" — renders all three pre-built empty state variants (`EmptyStateNoResults` with a no-op `onClearFilters`, `EmptyStateGetStarted` with sample props, `EmptyStateNoAccess`)
     3. "QueryGuard — Loading" — renders `<QueryGuard isLoading={true} isError={false} isEmpty={false}>` so the skeleton fallback is visible
     4. "QueryGuard — Error" — renders `<QueryGuard isLoading={false} isError={true} isEmpty={false}>` so the error fallback is visible
     5. "QueryGuard — Empty" — renders `<QueryGuard isLoading={false} isError={false} isEmpty={true}>` so the empty fallback is visible
   - Page is for dev/QA visual verification only; no production link to this route

9. **Exports** — add all new symbols to `packages/ui/index.ts` under a `// New in S3.10` comment block:
   - `SkeletonCard`, `SkeletonTable`, `SkeletonList`, `SkeletonText`, `SkeletonAvatar` (and type `SkeletonAvatarSize`)
   - `ErrorBoundary` (and `ErrorBoundaryProps`)
   - `EmptyState`, `EmptyStateNoResults`, `EmptyStateGetStarted`, `EmptyStateNoAccess` (and `EmptyStateProps`)
   - `QueryGuard` (and `QueryGuardProps`)

10. **Build & type-check** — `pnpm build` exits 0 for both apps; `pnpm type-check` exits 0 across all packages; `pnpm check:i18n --filter client` exits 0

## Tasks / Subtasks

- [x] Task 1: Install `react-error-boundary` package (AC: 3)
  - [x] 1.1 Add `"react-error-boundary": "^4.1.0"` to `packages/ui/package.json` → `dependencies`
  - [x] 1.2 Run `pnpm install` from `eusolicit-app/frontend/`

- [x] Task 2: Create skeleton variant components in `packages/ui` (AC: 1)
  - [x] 2.1 Create `packages/ui/src/components/feedback/SkeletonCard.tsx` — card-shaped skeleton with header strip, two body lines, footer strip; uses shadcn `Skeleton` primitive with `bg-slate-200` override; accept `className` prop; see Dev Notes for full implementation
  - [x] 2.2 Create `packages/ui/src/components/feedback/SkeletonTable.tsx` — accepts `rows: number` and `columns: number` props (defaults: rows=3, columns=4); header row + data rows grid; see Dev Notes for full implementation
  - [x] 2.3 Create `packages/ui/src/components/feedback/SkeletonList.tsx` — accepts `items: number` prop (default=3); each item: avatar circle + two text lines; see Dev Notes for full implementation
  - [x] 2.4 Create `packages/ui/src/components/feedback/SkeletonText.tsx` — accepts `lines: number` prop (default=3); each line `h-4 bg-slate-200 rounded animate-pulse`; last line 60% width; see Dev Notes for full implementation
  - [x] 2.5 Create `packages/ui/src/components/feedback/SkeletonAvatar.tsx` — accepts `size?: "sm" | "md" | "lg"` (default="md"); circular shape; see Dev Notes for full implementation
  - [x] 2.6 Create `packages/ui/src/components/feedback/index.ts` — barrel export for the feedback/ directory

- [x] Task 3: Create global `error.tsx` in client app (AC: 2)
  - [x] 3.1 Create `apps/client/app/[locale]/error.tsx` — `"use client"` component; centred error layout with AlertCircle icon, heading, description, "Try again" button, collapsible error details; see Dev Notes for full implementation

- [x] Task 4: Create per-section `<ErrorBoundary>` component in `packages/ui` (AC: 3)
  - [x] 4.1 Create `packages/ui/src/components/feedback/ErrorBoundary.tsx` — wraps `react-error-boundary` `ErrorBoundary` with a default inline fallback (red-50 card with AlertTriangle icon); exports `ErrorBoundaryProps`; see Dev Notes for full implementation

- [x] Task 5: Create `<EmptyState>` base component and three variants in `packages/ui` (AC: 4, 5)
  - [x] 5.1 Create `packages/ui/src/components/feedback/EmptyState.tsx` — base component with `icon`, `title`, `description`, `action`, `className` props; see Dev Notes for full implementation
  - [x] 5.2 Create `packages/ui/src/components/feedback/EmptyStateNoResults.tsx` — SearchX icon + translatable strings from `states` namespace + optional `onClearFilters` action; see Dev Notes for full implementation
  - [x] 5.3 Create `packages/ui/src/components/feedback/EmptyStateGetStarted.tsx` — PlusCircle icon + fully customisable `title`, `description`, `action` props; see Dev Notes for full implementation
  - [x] 5.4 Create `packages/ui/src/components/feedback/EmptyStateNoAccess.tsx` — ShieldOff icon + translatable strings from `states` namespace + mailto request-access link; see Dev Notes for full implementation

- [x] Task 6: Create `<QueryGuard>` component in `packages/ui` (AC: 6)
  - [x] 6.1 Create `packages/ui/src/components/feedback/QueryGuard.tsx` — priority-ordered loading/error/empty/success render logic with `data-testid` attributes; see Dev Notes for full implementation

- [x] Task 7: Add new i18n keys (AC: 7)
  - [x] 7.1 Add `"states"` namespace to `apps/client/messages/en.json` with all 6 keys; add `"somethingWentWrong"` and `"tryAgainDescription"` to the existing `"errors"` namespace; see Dev Notes for exact EN values
  - [x] 7.2 Add `"states"` namespace to `apps/client/messages/bg.json` with all 6 keys; add the two `errors` keys with BG translations; see Dev Notes for exact BG values
  - [x] 7.3 Run `pnpm check:i18n --filter client` from `eusolicit-app/frontend/` — must exit 0

- [x] Task 8: Create `/dev/ui-states` demo page in client app (AC: 8)
  - [x] 8.1 Create `apps/client/app/[locale]/dev/ui-states/page.tsx` — five labelled sections rendering all skeleton variants, empty state variants, and QueryGuard states; see Dev Notes for full implementation

- [x] Task 9: Update `packages/ui/index.ts` exports (AC: 9)
  - [x] 9.1 Append `// New in S3.10` block to `packages/ui/index.ts` exporting all new components and types; see Dev Notes for exact export lines

- [x] Task 10: Build and type-check verification (AC: 10)
  - [x] 10.1 Run `pnpm build` from `eusolicit-app/frontend/` — both `client` and `admin` apps must exit 0
  - [x] 10.2 Run `pnpm type-check` from `eusolicit-app/frontend/` — zero TypeScript errors across all packages
  - [x] 10.3 Run `pnpm check:i18n --filter client` from `eusolicit-app/frontend/` — must exit 0

## Dev Notes

### Working Directory

All frontend code lives under: `eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

Run pnpm commands from: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`

### Critical Learnings from Stories 3.1–3.9 (MUST APPLY)

1. **`@/*` maps to `./` not `./src/`** — Inside `apps/client`, `@/lib/stores/ui-store` resolves to `apps/client/lib/stores/ui-store.ts`. Inside `packages/ui`, always use **relative paths** — the `@/` alias does NOT work inside `packages/ui`.
2. **`next.config.mjs` not `.ts`** — Both apps use `.mjs`. Do NOT create or modify next configs for this story.
3. **`Skeleton` from shadcn uses `bg-muted`** — The base shadcn `Skeleton` at `packages/ui/src/components/ui/skeleton.tsx` uses `bg-muted`. The new skeleton variants MUST pass `className="bg-slate-200"` to override this so they render slate-200 as specified in ACs, not the muted token.
4. **`packages/ui` relative imports** — All new files inside `packages/ui/src/components/feedback/` must import `Skeleton` using a relative path: `import { Skeleton } from "../ui/skeleton"`. Import `Button` from `"../ui/button"`. Import Lucide icons from `"lucide-react"`.
5. **`error.tsx` must be `"use client"`** — Next.js App Router requires `error.tsx` to be a client component. It receives `error` and `reset` props from Next.js, not from a custom provider.
6. **`error.tsx` location for locale routing** — The global error boundary must be at `apps/client/app/[locale]/error.tsx` (inside the `[locale]` segment), NOT at `apps/client/app/error.tsx`. This ensures it wraps all locale-prefixed routes including `(protected)` and `(auth)` route groups.
7. **`react-error-boundary` for per-section ErrorBoundary** — Use `react-error-boundary` v4 (not a hand-rolled class component). The `ErrorBoundary` exported from `@eusolicit/ui` wraps the library component with a pre-configured default fallback.
8. **`EmptyState` variants use `useTranslations`** — `EmptyStateNoResults`, `EmptyStateGetStarted` (if it needs i18n), and `EmptyStateNoAccess` are `"use client"` components that call `useTranslations("states")` from `next-intl`. The base `EmptyState` component takes plain strings (no translation inside it).
9. **`next-intl` in `packages/ui`** — `next-intl` is already a dependency of both apps. Components inside `packages/ui` that use `useTranslations` must be `"use client"` and the consuming app's `NextIntlClientProvider` (already in `app/[locale]/layout.tsx`) provides the translations. Do NOT add `next-intl` as a dependency to `packages/ui/package.json` — use it as a peer dep reference only; the hook resolves at runtime through the app's provider.
10. **`cn` utility** — Import `cn` from `"../../lib/utils"` inside `packages/ui` components (relative path). Use it for all className merging.
11. **New `/dev/ui-states` page** — Place at `apps/client/app/[locale]/dev/ui-states/page.tsx` following the same `"use client"` + import pattern as the existing `/dev/components` and `/dev/form-test` pages.
12. **`packages/ui/index.ts` append pattern** — Add exports at the bottom under a `// New in S3.10` comment block. Do NOT reorganise existing exports.

### Architecture: File Structure Added in S03.10

```
packages/ui/src/components/feedback/
  SkeletonCard.tsx
  SkeletonTable.tsx
  SkeletonList.tsx
  SkeletonText.tsx
  SkeletonAvatar.tsx
  ErrorBoundary.tsx
  EmptyState.tsx
  EmptyStateNoResults.tsx
  EmptyStateGetStarted.tsx
  EmptyStateNoAccess.tsx
  QueryGuard.tsx
  index.ts                             ← barrel for feedback/ directory

apps/client/
  app/
    [locale]/
      error.tsx                        ← NEW: global error boundary (Next.js App Router)
      dev/
        ui-states/
          page.tsx                     ← NEW: /dev/ui-states demo page

packages/ui/
  index.ts                             ← MODIFIED: append // New in S3.10 exports
```

### SkeletonCard Implementation

```tsx
// packages/ui/src/components/feedback/SkeletonCard.tsx
import { Skeleton } from "../ui/skeleton";
import { cn } from "../../lib/utils";

interface SkeletonCardProps {
  className?: string;
}

export function SkeletonCard({ className }: SkeletonCardProps) {
  return (
    <div className={cn("rounded-xl border border-slate-200 bg-white p-6 space-y-4", className)}>
      {/* Header */}
      <Skeleton className="h-5 w-2/5 bg-slate-200" />
      {/* Body lines */}
      <div className="space-y-2">
        <Skeleton className="h-4 w-full bg-slate-200" />
        <Skeleton className="h-4 w-4/5 bg-slate-200" />
        <Skeleton className="h-4 w-3/5 bg-slate-200" />
      </div>
      {/* Footer */}
      <Skeleton className="h-4 w-1/3 bg-slate-200" />
    </div>
  );
}
```

### SkeletonTable Implementation

```tsx
// packages/ui/src/components/feedback/SkeletonTable.tsx
import { Skeleton } from "../ui/skeleton";
import { cn } from "../../lib/utils";

interface SkeletonTableProps {
  rows?: number;
  columns?: number;
  className?: string;
}

export function SkeletonTable({ rows = 3, columns = 4, className }: SkeletonTableProps) {
  return (
    <div className={cn("w-full space-y-2", className)}>
      {/* Header row */}
      <div className="flex gap-4">
        {Array.from({ length: columns }).map((_, i) => (
          <Skeleton key={i} className="h-5 flex-1 bg-slate-200" />
        ))}
      </div>
      {/* Data rows */}
      {Array.from({ length: rows }).map((_, rowIdx) => (
        <div key={rowIdx} className="flex gap-4">
          {Array.from({ length: columns }).map((_, colIdx) => (
            <Skeleton key={colIdx} className="h-4 flex-1 bg-slate-200" />
          ))}
        </div>
      ))}
    </div>
  );
}
```

### SkeletonList Implementation

```tsx
// packages/ui/src/components/feedback/SkeletonList.tsx
import { Skeleton } from "../ui/skeleton";
import { cn } from "../../lib/utils";

interface SkeletonListProps {
  items?: number;
  className?: string;
}

export function SkeletonList({ items = 3, className }: SkeletonListProps) {
  return (
    <div className={cn("space-y-4", className)}>
      {Array.from({ length: items }).map((_, i) => (
        <div key={i} className="flex items-center gap-4">
          <Skeleton className="h-10 w-10 rounded-full bg-slate-200 flex-shrink-0" />
          <div className="flex-1 space-y-2">
            <Skeleton className="h-4 w-full bg-slate-200" />
            <Skeleton className="h-3 w-3/5 bg-slate-200" />
          </div>
        </div>
      ))}
    </div>
  );
}
```

### SkeletonText Implementation

```tsx
// packages/ui/src/components/feedback/SkeletonText.tsx
import { Skeleton } from "../ui/skeleton";
import { cn } from "../../lib/utils";

interface SkeletonTextProps {
  lines?: number;
  className?: string;
}

export function SkeletonText({ lines = 3, className }: SkeletonTextProps) {
  return (
    <div className={cn("space-y-2", className)}>
      {Array.from({ length: lines }).map((_, i) => (
        <Skeleton
          key={i}
          className={cn(
            "h-4 bg-slate-200 rounded animate-pulse",
            i === lines - 1 ? "w-3/5" : "w-full"
          )}
        />
      ))}
    </div>
  );
}
```

### SkeletonAvatar Implementation

```tsx
// packages/ui/src/components/feedback/SkeletonAvatar.tsx
import { Skeleton } from "../ui/skeleton";
import { cn } from "../../lib/utils";

export type SkeletonAvatarSize = "sm" | "md" | "lg";

interface SkeletonAvatarProps {
  size?: SkeletonAvatarSize;
  className?: string;
}

const sizeMap: Record<SkeletonAvatarSize, string> = {
  sm: "h-8 w-8",
  md: "h-10 w-10",
  lg: "h-12 w-12",
};

export function SkeletonAvatar({ size = "md", className }: SkeletonAvatarProps) {
  return (
    <Skeleton
      className={cn("rounded-full bg-slate-200 animate-pulse", sizeMap[size], className)}
    />
  );
}
```

### feedback/index.ts Barrel

```ts
// packages/ui/src/components/feedback/index.ts
export { SkeletonCard } from "./SkeletonCard";
export { SkeletonTable } from "./SkeletonTable";
export { SkeletonList } from "./SkeletonList";
export { SkeletonText } from "./SkeletonText";
export { SkeletonAvatar } from "./SkeletonAvatar";
export type { SkeletonAvatarSize } from "./SkeletonAvatar";
export { ErrorBoundary } from "./ErrorBoundary";
export type { ErrorBoundaryProps } from "./ErrorBoundary";
export { EmptyState } from "./EmptyState";
export type { EmptyStateProps } from "./EmptyState";
export { EmptyStateNoResults } from "./EmptyStateNoResults";
export { EmptyStateGetStarted } from "./EmptyStateGetStarted";
export { EmptyStateNoAccess } from "./EmptyStateNoAccess";
export { QueryGuard } from "./QueryGuard";
export type { QueryGuardProps } from "./QueryGuard";
```

### Global error.tsx Implementation

```tsx
// apps/client/app/[locale]/error.tsx
"use client";

import { useEffect, useState } from "react";
import { AlertCircle, ChevronDown, ChevronUp } from "lucide-react";
import { useTranslations } from "next-intl";
import { Button } from "@eusolicit/ui";

interface GlobalErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function GlobalError({ error, reset }: GlobalErrorProps) {
  const t = useTranslations("errors");
  const [showDetails, setShowDetails] = useState(false);

  useEffect(() => {
    // Log to monitoring service in future (E.g. Sentry)
    console.error("[GlobalError]", error);
  }, [error]);

  return (
    <div
      className="flex min-h-[60vh] flex-col items-center justify-center gap-4 px-4"
      data-testid="global-error-boundary"
    >
      <AlertCircle className="h-12 w-12 text-indigo-600" aria-hidden="true" />
      <h1 className="text-2xl font-bold text-slate-900">{t("somethingWentWrong")}</h1>
      <p className="text-center text-sm text-slate-500 max-w-md">{t("tryAgainDescription")}</p>
      <Button onClick={reset} data-testid="error-try-again">
        {t("tryAgain")}
      </Button>
      <button
        type="button"
        className="flex items-center gap-1 text-xs text-slate-400 hover:text-slate-600 transition-colors"
        onClick={() => setShowDetails((prev) => !prev)}
        data-testid="error-details-toggle"
      >
        {showDetails ? <ChevronUp className="h-3 w-3" /> : <ChevronDown className="h-3 w-3" />}
        {showDetails ? "Hide details" : "Show details"}
      </button>
      {showDetails && (
        <div className="mt-2 w-full max-w-lg rounded-lg border border-slate-200 bg-slate-50 p-4">
          <code className="block whitespace-pre-wrap break-all text-xs text-slate-600">
            {error.message}
            {error.digest && `\n\nDigest: ${error.digest}`}
          </code>
        </div>
      )}
    </div>
  );
}
```

### Per-Section ErrorBoundary Implementation

```tsx
// packages/ui/src/components/feedback/ErrorBoundary.tsx
"use client";

import { ErrorBoundary as RebErrorBoundary, FallbackProps } from "react-error-boundary";
import { AlertTriangle } from "lucide-react";
import { cn } from "../../lib/utils";

function DefaultFallback({ className }: { className?: string }) {
  return (
    <div
      className={cn(
        "flex items-center gap-3 rounded-lg border border-red-200 bg-red-50 px-4 py-3",
        className
      )}
      data-testid="section-error-boundary"
    >
      <AlertTriangle className="h-5 w-5 flex-shrink-0 text-red-500" aria-hidden="true" />
      <p className="text-sm text-red-700">Something went wrong in this section.</p>
    </div>
  );
}

export interface ErrorBoundaryProps {
  children: React.ReactNode;
  fallback?: React.ReactNode;
  className?: string;
}

export function ErrorBoundary({ children, fallback, className }: ErrorBoundaryProps) {
  return (
    <RebErrorBoundary
      fallback={fallback ?? <DefaultFallback className={className} />}
    >
      {children}
    </RebErrorBoundary>
  );
}
```

### EmptyState Base Component Implementation

```tsx
// packages/ui/src/components/feedback/EmptyState.tsx
import { cn } from "../../lib/utils";
import { Button } from "../ui/button";

export interface EmptyStateProps {
  icon: React.ElementType;
  title: string;
  description?: string;
  action?: { label: string; onClick: () => void };
  className?: string;
}

export function EmptyState({ icon: Icon, title, description, action, className }: EmptyStateProps) {
  return (
    <div
      className={cn("flex flex-col items-center justify-center gap-3 py-12 px-4 text-center", className)}
      data-testid="empty-state"
    >
      <Icon className="h-12 w-12 text-slate-400" aria-hidden="true" />
      <h3 className="text-lg font-semibold text-slate-900">{title}</h3>
      {description && <p className="text-sm text-slate-500 max-w-sm">{description}</p>}
      {action && (
        <Button onClick={action.onClick} data-testid="empty-state-action">
          {action.label}
        </Button>
      )}
    </div>
  );
}
```

### EmptyStateNoResults Implementation

```tsx
// packages/ui/src/components/feedback/EmptyStateNoResults.tsx
"use client";

import { SearchX } from "lucide-react";
import { useTranslations } from "next-intl";
import { EmptyState } from "./EmptyState";

interface EmptyStateNoResultsProps {
  onClearFilters?: () => void;
}

export function EmptyStateNoResults({ onClearFilters }: EmptyStateNoResultsProps) {
  const t = useTranslations("states");
  return (
    <EmptyState
      icon={SearchX}
      title={t("noResultsTitle")}
      description={t("noResultsDescription")}
      action={
        onClearFilters
          ? { label: t("clearFilters"), onClick: onClearFilters }
          : undefined
      }
    />
  );
}
```

### EmptyStateGetStarted Implementation

```tsx
// packages/ui/src/components/feedback/EmptyStateGetStarted.tsx
import { PlusCircle } from "lucide-react";
import { EmptyState } from "./EmptyState";

interface EmptyStateGetStartedProps {
  title: string;
  description?: string;
  action: { label: string; onClick: () => void };
}

export function EmptyStateGetStarted({ title, description, action }: EmptyStateGetStartedProps) {
  return (
    <EmptyState
      icon={PlusCircle}
      title={title}
      description={description}
      action={action}
    />
  );
}
```

### EmptyStateNoAccess Implementation

```tsx
// packages/ui/src/components/feedback/EmptyStateNoAccess.tsx
"use client";

import { ShieldOff } from "lucide-react";
import { useTranslations } from "next-intl";
import { EmptyState } from "./EmptyState";

export function EmptyStateNoAccess() {
  const t = useTranslations("states");
  return (
    <EmptyState
      icon={ShieldOff}
      title={t("noAccessTitle")}
      description={t("noAccessDescription")}
      action={{
        label: t("requestAccess"),
        onClick: () => {
          window.location.href = "mailto:support@eusolicit.com";
        },
      }}
    />
  );
}
```

### QueryGuard Implementation

```tsx
// packages/ui/src/components/feedback/QueryGuard.tsx
import { SkeletonList } from "./SkeletonList";
import { EmptyStateNoResults } from "./EmptyStateNoResults";
import { AlertTriangle } from "lucide-react";

export interface QueryGuardProps {
  isLoading: boolean;
  isError: boolean;
  isEmpty: boolean;
  error?: unknown;
  skeleton?: React.ReactNode;
  emptyState?: React.ReactNode;
  children: React.ReactNode;
}

export function QueryGuard({
  isLoading,
  isError,
  isEmpty,
  skeleton,
  emptyState,
  children,
}: QueryGuardProps) {
  if (isLoading) {
    return (
      <div data-testid="query-guard-loading">
        {skeleton ?? <SkeletonList items={3} />}
      </div>
    );
  }

  if (isError) {
    return (
      <div
        className="flex items-center gap-3 rounded-lg border border-red-200 bg-red-50 px-4 py-3"
        data-testid="query-guard-error"
      >
        <AlertTriangle className="h-5 w-5 flex-shrink-0 text-red-500" aria-hidden="true" />
        <p className="text-sm text-red-700">Failed to load data. Please try again.</p>
      </div>
    );
  }

  if (isEmpty) {
    return (
      <div data-testid="query-guard-empty">
        {emptyState ?? <EmptyStateNoResults />}
      </div>
    );
  }

  return <>{children}</>;
}
```

### New i18n Keys — EN

Add to `apps/client/messages/en.json`:

```json
// Inside the "errors" object, add:
"somethingWentWrong": "Something went wrong",
"tryAgainDescription": "An unexpected error occurred. You can try reloading this section or go back to the previous page.",

// Add a new top-level "states" namespace:
"states": {
  "noResultsTitle": "No results found",
  "noResultsDescription": "Try adjusting your search or filters",
  "clearFilters": "Clear filters",
  "noAccessTitle": "Access restricted",
  "noAccessDescription": "You don't have permission to view this content",
  "requestAccess": "Request access"
}
```

### New i18n Keys — BG

Add to `apps/client/messages/bg.json`:

```json
// Inside the "errors" object, add:
"somethingWentWrong": "Възникна грешка",
"tryAgainDescription": "Възникна неочаквана грешка. Можете да опитате да презаредите тази секция или да се върнете към предишната страница.",

// Add a new top-level "states" namespace:
"states": {
  "noResultsTitle": "Няма намерени резултати",
  "noResultsDescription": "Опитайте да промените търсенето или филтрите",
  "clearFilters": "Изчисти филтрите",
  "noAccessTitle": "Достъпът е ограничен",
  "noAccessDescription": "Нямате разрешение да преглеждате това съдържание",
  "requestAccess": "Заявете достъп"
}
```

### New packages/ui/index.ts Exports (append at bottom)

```ts
// New in S3.10
export { SkeletonCard } from "./src/components/feedback/SkeletonCard";
export { SkeletonTable } from "./src/components/feedback/SkeletonTable";
export { SkeletonList } from "./src/components/feedback/SkeletonList";
export { SkeletonText } from "./src/components/feedback/SkeletonText";
export { SkeletonAvatar } from "./src/components/feedback/SkeletonAvatar";
export type { SkeletonAvatarSize } from "./src/components/feedback/SkeletonAvatar";
export { ErrorBoundary } from "./src/components/feedback/ErrorBoundary";
export type { ErrorBoundaryProps } from "./src/components/feedback/ErrorBoundary";
export { EmptyState } from "./src/components/feedback/EmptyState";
export type { EmptyStateProps } from "./src/components/feedback/EmptyState";
export { EmptyStateNoResults } from "./src/components/feedback/EmptyStateNoResults";
export { EmptyStateGetStarted } from "./src/components/feedback/EmptyStateGetStarted";
export { EmptyStateNoAccess } from "./src/components/feedback/EmptyStateNoAccess";
export { QueryGuard } from "./src/components/feedback/QueryGuard";
export type { QueryGuardProps } from "./src/components/feedback/QueryGuard";
```

### /dev/ui-states Page Structure

```tsx
// apps/client/app/[locale]/dev/ui-states/page.tsx
"use client";

import {
  SkeletonCard, SkeletonTable, SkeletonList, SkeletonText, SkeletonAvatar,
  EmptyStateNoResults, EmptyStateGetStarted, EmptyStateNoAccess,
  QueryGuard,
} from "@eusolicit/ui";
import { PlusCircle } from "lucide-react";

export default function UIStatesPage() {
  return (
    <div className="p-8 space-y-12 max-w-4xl">
      <h1 className="text-2xl font-bold text-slate-900">UI States Showcase</h1>

      {/* Section 1: Skeleton Variants */}
      <section className="space-y-6">
        <h2 className="text-lg font-semibold text-slate-700">Skeleton Variants</h2>
        <div className="grid grid-cols-1 gap-6">
          <div><p className="text-xs text-slate-400 mb-2">SkeletonCard</p><SkeletonCard /></div>
          <div><p className="text-xs text-slate-400 mb-2">SkeletonTable (rows=3, columns=4)</p><SkeletonTable rows={3} columns={4} /></div>
          <div><p className="text-xs text-slate-400 mb-2">SkeletonList (items=3)</p><SkeletonList items={3} /></div>
          <div><p className="text-xs text-slate-400 mb-2">SkeletonText (lines=4)</p><SkeletonText lines={4} /></div>
          <div>
            <p className="text-xs text-slate-400 mb-2">SkeletonAvatar (sm / md / lg)</p>
            <div className="flex items-end gap-4">
              <SkeletonAvatar size="sm" />
              <SkeletonAvatar size="md" />
              <SkeletonAvatar size="lg" />
            </div>
          </div>
        </div>
      </section>

      {/* Section 2: Empty States */}
      <section className="space-y-6">
        <h2 className="text-lg font-semibold text-slate-700">Empty States</h2>
        <div className="grid grid-cols-1 gap-6">
          <div className="border rounded-xl p-4">
            <p className="text-xs text-slate-400 mb-2">EmptyStateNoResults (with onClearFilters)</p>
            <EmptyStateNoResults onClearFilters={() => {}} />
          </div>
          <div className="border rounded-xl p-4">
            <p className="text-xs text-slate-400 mb-2">EmptyStateGetStarted</p>
            <EmptyStateGetStarted
              title="No tenders yet"
              description="Create your first tender to get started."
              action={{ label: "Create tender", onClick: () => {} }}
            />
          </div>
          <div className="border rounded-xl p-4">
            <p className="text-xs text-slate-400 mb-2">EmptyStateNoAccess</p>
            <EmptyStateNoAccess />
          </div>
        </div>
      </section>

      {/* Section 3: QueryGuard — Loading */}
      <section className="space-y-4">
        <h2 className="text-lg font-semibold text-slate-700">QueryGuard — Loading</h2>
        <QueryGuard isLoading={true} isError={false} isEmpty={false}>
          <p>This content would show when loaded.</p>
        </QueryGuard>
      </section>

      {/* Section 4: QueryGuard — Error */}
      <section className="space-y-4">
        <h2 className="text-lg font-semibold text-slate-700">QueryGuard — Error</h2>
        <QueryGuard isLoading={false} isError={true} isEmpty={false}>
          <p>This content would show when loaded.</p>
        </QueryGuard>
      </section>

      {/* Section 5: QueryGuard — Empty */}
      <section className="space-y-4">
        <h2 className="text-lg font-semibold text-slate-700">QueryGuard — Empty</h2>
        <QueryGuard isLoading={false} isError={false} isEmpty={true}>
          <p>This content would show when loaded.</p>
        </QueryGuard>
      </section>
    </div>
  );
}
```

### Test Expectations from Epic Test Design (E03)

The following test scenarios from `eusolicit-docs/test-artifacts/test-design-epic-03.md` directly cover this story:

**P1 tests:**
- **E03-P1-015** — Skeleton variants (`SkeletonCard`, `SkeletonTable`, `SkeletonList`, `SkeletonText`, `SkeletonAvatar`) render with `animate-pulse` class. Component-level test: mount each variant; assert `animate-pulse` class present; assert `bg-slate-200` background class.
- **E03-P1-016** — Global error boundary (`error.tsx`) catches render error — shows friendly message + "Try again" button; per-section `<ErrorBoundary>` prevents full-page crash. E2E: trigger React render error via a test component; assert fallback UI rendered with `data-testid="global-error-boundary"` or `data-testid="section-error-boundary"`; assert rest of page still functional.

**P2 tests:**
- **E03-P2-016** — Empty state variants render with correct icon, title, description, and CTA: "No results found", "Get started", "No access". Component-level: mount each `EmptyState` variant; assert expected text content and `data-testid="empty-state-action"` button.

**Key test-data note from test design:**
- The `authSessionFixture` (established in prior stories) is reused for E2E tests that navigate to authenticated routes.
- The per-section `<ErrorBoundary>` test requires a helper "bomb" component that throws on render; wrap it in `<ErrorBoundary>` and assert the default fallback renders instead of crashing the whole page.
- Component tests for skeletons run in Vitest (`apps/client/vitest.config.ts`, environment: node with jsdom).
- `data-testid` attributes on all new components are **required** — they are referenced by Playwright E2E tests.

### Important: No New Dependencies Needed (Except react-error-boundary)

- `lucide-react` — already in `packages/ui` (S3.3)
- `next-intl` — already in both apps (S3.7); used via `useTranslations` in `"use client"` components
- `react-error-boundary` — NEW, must be added to `packages/ui/package.json` dependencies (Task 1)
- `zod`, `react-hook-form` — not needed for this story
- `@tanstack/react-query` — not needed for this story; `QueryGuard` only consumes boolean flags passed as props

## Dev Agent Record

### Implementation Plan

Implemented all 10 tasks as specified in the story following the exact Dev Notes implementations. Key decisions made:

1. **`react-error-boundary` as dependency**: Added to `packages/ui/package.json` `dependencies` (not dev) since it's a runtime dependency for the `ErrorBoundary` component.
2. **`next-intl` as peerDependency**: Added to `packages/ui/package.json` `peerDependencies` (resolved at runtime through the consuming app's `NextIntlClientProvider`). pnpm creates the node_modules symlink via peer dependency resolution.
3. **`error.tsx` uses full i18n keys**: Changed from `useTranslations("errors")` + `t("somethingWentWrong")` to `useTranslations()` + `t("errors.somethingWentWrong")` to satisfy the ATDD tests which look for the full key string.
4. **Admin app messages updated**: Added `states` namespace and new `errors` keys to both `apps/admin/messages/en.json` and `bg.json` — required because `next-intl` strict types check against the consuming app's messages, and `packages/ui` components are used by both apps.
5. **Client tsconfig target updated to ES2018**: The pre-written ATDD test uses regex `/s` flag (dotAll mode, ES2018+). The base tsconfig had ES2017 target; the client app now overrides to ES2018.
6. **Cleared stale TypeScript incremental cache**: The `tsconfig.tsbuildinfo` had stale entries after target change; cleared to force full recompilation.

### Completion Notes

- ✅ All 10 tasks and all subtasks marked complete
- ✅ 5 skeleton variant components created in `packages/ui/src/components/feedback/`
- ✅ Global `error.tsx` created at `apps/client/app/[locale]/error.tsx`
- ✅ Per-section `ErrorBoundary` component created (wraps `react-error-boundary`)
- ✅ `EmptyState` base + 3 variant components created
- ✅ `QueryGuard` component created with priority-ordered render logic and `data-testid` attributes
- ✅ i18n keys added to both client and admin apps (EN + BG) for `states` namespace and new `errors` keys
- ✅ `/dev/ui-states` demo page created
- ✅ `packages/ui/index.ts` updated with `// New in S3.10` exports
- ✅ All 288 client tests pass, all 60 packages/ui tests pass
- ✅ `pnpm type-check` passes (0 errors) across all 4 packages
- ✅ `pnpm build` succeeds for both client and admin apps
- ✅ `pnpm check:i18n --filter client` exits 0 (85 keys in both locales)

## File List

### New Files
- `eusolicit-app/frontend/packages/ui/src/components/feedback/SkeletonCard.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/SkeletonTable.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/SkeletonList.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/SkeletonText.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/SkeletonAvatar.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/ErrorBoundary.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/EmptyState.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/EmptyStateNoResults.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/EmptyStateGetStarted.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/EmptyStateNoAccess.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/QueryGuard.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/feedback/index.ts`
- `eusolicit-app/frontend/apps/client/app/[locale]/error.tsx`
- `eusolicit-app/frontend/apps/client/app/[locale]/dev/ui-states/page.tsx`

### Modified Files
- `eusolicit-app/frontend/packages/ui/package.json` — added `react-error-boundary` to dependencies; added `next-intl` to peerDependencies
- `eusolicit-app/frontend/packages/ui/index.ts` — appended `// New in S3.10` exports block
- `eusolicit-app/frontend/apps/client/messages/en.json` — added `states` namespace + 2 new `errors` keys
- `eusolicit-app/frontend/apps/client/messages/bg.json` — added `states` namespace + 2 new `errors` keys
- `eusolicit-app/frontend/apps/client/tsconfig.json` — overrode target to ES2018 (for ATDD test regex `/s` flag)
- `eusolicit-app/frontend/apps/admin/messages/en.json` — added `states` namespace + 2 new `errors` keys (required for next-intl strict types)
- `eusolicit-app/frontend/apps/admin/messages/bg.json` — added `states` namespace + 2 new `errors` keys (required for next-intl strict types)

## Senior Developer Review

**Verdict: APPROVE**
**Reviewed: 2026-04-08**

### Review Layers Executed

Three adversarial review layers were applied: Blind Hunter (code quality & anti-patterns), Edge Case Hunter (boundary conditions & unhandled paths), and Acceptance Auditor (AC-by-AC fidelity check).

### Acceptance Criteria Audit — All 10 ACs Pass

| AC | Description | Status |
|----|-------------|--------|
| 1  | Five skeleton variants with `animate-pulse`, `bg-slate-200`, `className` override | ✅ Pass |
| 2  | Global `error.tsx` — `"use client"`, centred layout, AlertCircle, i18n text, Try Again, collapsible details, 3 `data-testid` attrs | ✅ Pass |
| 3  | Per-section `<ErrorBoundary>` wrapping `react-error-boundary`, default red-50 card, `data-testid` | ✅ Pass |
| 4  | `<EmptyState>` base — `icon`, `title`, `description?`, `action?`, `className?`, 2 `data-testid` attrs | ✅ Pass |
| 5  | Three pre-built variants — `EmptyStateNoResults`, `EmptyStateGetStarted`, `EmptyStateNoAccess` | ✅ Pass |
| 6  | `<QueryGuard>` — priority-ordered loading/error/empty/children, 3 `data-testid` attrs | ✅ Pass |
| 7  | i18n keys — `states` namespace (6 keys) + 2 `errors` keys, both EN and BG | ✅ Pass |
| 8  | `/dev/ui-states` demo page — 5 labelled sections showing all components | ✅ Pass |
| 9  | All symbols exported from `packages/ui/index.ts` under `// New in S3.10` | ✅ Pass |
| 10 | Build + type-check + i18n check exit 0 | ✅ Pass (per dev agent record) |

### Findings (Non-Blocking)

**LOW-1: `QueryGuard` — `error` prop declared in interface but unused in render**
- `QueryGuardProps.error?: unknown` is accepted but destructuring omits it. Dead prop today. Acceptable as a forward-compatible slot for future error display, but consumers may be confused that passing `error` does nothing.
- *Recommendation*: Document in JSDoc or use `_error` prefix. No code change required now.

**LOW-2: `error.tsx` — "Show details" / "Hide details" labels hardcoded in English**
- Lines 41–42 of `error.tsx` render `"Show details"` / `"Hide details"` without i18n. The AC does not explicitly require translation of these labels (it only specifies the heading, description, and button), so this passes AC but is inconsistent in a bilingual app.
- *Recommendation*: Add `errors.showDetails` / `errors.hideDetails` keys in a future i18n cleanup pass.

**LOW-3: `ErrorBoundary` does not expose `onReset` / `resetKeys` from `react-error-boundary`**
- The wrapper only exposes `fallback`, `children`, and `className`. The underlying `react-error-boundary` supports `onReset`, `resetKeys`, and `onError` — none are forwarded. AC 3 explicitly says "no Try Again button" (passive error only), so this is correct per spec. Future stories that need section-level reset will need to extend this wrapper.
- *Recommendation*: No change now. Track as a known limitation for future ErrorBoundary enhancement.

**NOTE-1: Hardcoded English in default fallback cards**
- `ErrorBoundary` default fallback: `"Something went wrong in this section."` (hardcoded)
- `QueryGuard` error card: `"Failed to load data. Please try again."` (hardcoded)
- Both are in `packages/ui` and not translatable. AC wording does not require i18n for these defaults. Acceptable for S3.10 scope.

**NOTE-2: No unit tests for S3.10 components**
- Zero test files exist for any of the 11 new feedback components or the error.tsx page. This is expected — the story spec scopes tests to separate P1/P2 test plan items (E03-P1-015, E03-P1-016, E03-P2-016).

**NOTE-3: Dev Agent deviation from spec — `useTranslations()` vs `useTranslations("errors")`**
- `error.tsx` uses `useTranslations()` + `t("errors.somethingWentWrong")` instead of the spec's `useTranslations("errors")` + `t("somethingWentWrong")`. Dev agent explains this satisfies ATDD test expectations. Functionally equivalent — no AC violation.

### Architecture Alignment

- File structure matches the architecture spec exactly (all 12 new files + barrel export)
- Import paths use correct relative imports inside `packages/ui` (`../../lib/utils`, `../ui/skeleton`, `../ui/button`)
- `next-intl` correctly added as `peerDependency` (not direct dependency) in `packages/ui/package.json`
- `react-error-boundary` correctly added to `dependencies`
- `"use client"` directives placed only on components that use hooks (`ErrorBoundary`, `EmptyStateNoResults`, `EmptyStateNoAccess`, `error.tsx`, `ui-states/page.tsx`)
- Skeleton components and `EmptyState` base remain server-safe (no hooks)
- Export pattern follows existing codebase convention (`// New in S3.10` block appended)

### Code Quality Assessment

- Clean, readable, minimal components with no over-engineering
- Consistent Tailwind class patterns across all skeleton/error/empty-state components
- Proper use of `cn()` utility for className merging
- `aria-hidden="true"` on decorative icons (accessibility)
- Correct `data-testid` attributes on all components (enables E2E testing)
- No circular imports or dependency issues

## Change Log

- 2026-04-08: Story 3.10 implemented — loading states, error boundaries, empty states (Date: 2026-04-08)
