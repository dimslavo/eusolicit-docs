# Story 3.3: App Shell Layout — Sidebar, Top Bar, Content Area

Status: done

## Story

As a **frontend developer on the EU Solicit team**,
I want **a reusable `<AppShell>` component in `packages/ui` with a collapsible sidebar, sticky top bar, and scrollable main content area, wired into both the client and admin Next.js apps**,
so that **every subsequent E03 and feature-epic story drops its UI into a fully functioning, consistently styled shell without rebuilding navigation scaffolding**.

## Acceptance Criteria

1. `<AppShell>` component in `packages/ui` accepts `sidebar`, `topbar`, and `children` slot props; exported from `@eusolicit/ui`
2. Sidebar renders `<NavItem icon={...} label={...} href={...} />` components with active-state highlight: `bg-indigo-50 text-indigo-600` on the active route
3. Sidebar toggles between full mode (~256px, icon + label) and collapsed mode (~64px, icon-only) with a 200ms ease CSS width transition; labels are hidden in collapsed mode
4. Top bar is sticky, 64px height (`h-16`), contains: left-side `<Breadcrumbs />`, right-side action cluster with: notifications bell (with unread-count badge), language selector placeholder, user avatar `<DropdownMenu>`
5. User avatar dropdown menu shows: user display name, email, "Profile" link, "Settings" link, a `<Separator />`, and a "Sign out" button
6. Notifications bell shows an unread count badge (static placeholder value)
7. Main content area scrolls independently; sidebar and top bar remain fixed/sticky during scroll
8. Client app uses the shell for a `(protected)` route group: sidebar nav items are Dashboard, Tenders, Offers, Documents, Team, Settings
9. Admin app uses the shell for a `(protected)` route group: sidebar nav items are Dashboard, Companies, Tenders, Users, Reports, Settings
10. Sidebar collapsed state is stored in a minimal `uiStore` (Zustand) in each app's `lib/stores/ui-store.ts`, persisted to `localStorage`; SSR hydration mismatch is prevented via client-only rendering of the collapsed state
11. `pnpm build` exits 0 for both apps after this story's changes

## Tasks / Subtasks

- [x] Task 1: Add `zustand` dependency to both apps (AC: 10)
  - [x] 1.1 Add `"zustand": "^4.5.0"` to `dependencies` in `apps/client/package.json`
  - [x] 1.2 Add `"zustand": "^4.5.0"` to `dependencies` in `apps/admin/package.json`
  - [x] 1.3 Run `pnpm install` from `frontend/` root

- [x] Task 2: Create minimal `uiStore` in both apps (AC: 10)
  - [x] 2.1 Create `apps/client/lib/stores/ui-store.ts` — Zustand store with `persist` middleware; state: `sidebarCollapsed: boolean`; actions: `toggleSidebar()`, `setSidebarCollapsed(val: boolean)`; persist key: `"eusolicit-ui-store"`
  - [x] 2.2 Create `apps/admin/lib/stores/ui-store.ts` — identical store; persist key: `"eusolicit-admin-ui-store"`

- [x] Task 3: Create `NavItem` component in `packages/ui` (AC: 2)
  - [x] 3.1 Create `packages/ui/src/components/app-shell/NavItem.tsx` — accepts `icon: React.ElementType`, `label: string`, `href: string`, `isCollapsed?: boolean`; renders Next.js `<Link>` with active highlight using `usePathname()`; renders icon always, label only when `!isCollapsed`; active state: `bg-indigo-50 text-indigo-600 font-medium`; inactive: `text-slate-600 hover:bg-slate-100 hover:text-slate-900`
  - [x] 3.2 Mark `NavItem` with `"use client"` (uses `usePathname`)

- [x] Task 4: Create `Sidebar` component in `packages/ui` (AC: 2, 3, 7)
  - [x] 4.1 Create `packages/ui/src/components/app-shell/Sidebar.tsx` — accepts `navItems: NavItemConfig[]`, `isCollapsed: boolean`, `onToggle: () => void`; renders fixed left sidebar `fixed inset-y-0 left-0 z-30 flex flex-col bg-white border-r border-slate-200`; width transition: `transition-[width] duration-200 ease-in-out`; full: `w-64`, collapsed: `w-16`
  - [x] 4.2 Add toggle button inside Sidebar (bottom of sidebar or top section) using Lucide `ChevronLeft`/`ChevronRight` icons; rotates on collapse
  - [x] 4.3 Add overflow hidden on labels: `overflow-hidden whitespace-nowrap` so text clips cleanly during transition
  - [x] 4.4 Mark `Sidebar` with `"use client"`

- [x] Task 5: Create `TopBar` component in `packages/ui` (AC: 4, 5, 6)
  - [x] 5.1 Create `packages/ui/src/components/app-shell/TopBar.tsx` — accepts `leftSlot?: React.ReactNode`, `user?: { name: string; email: string }`; renders `sticky top-0 z-20 h-16 bg-white border-b border-slate-200 flex items-center justify-between px-4`
  - [x] 5.2 Left side: renders `leftSlot` (breadcrumbs placeholder) or an empty `<div>`
  - [x] 5.3 Right side action cluster: `NotificationsBell`, language selector placeholder `<Button variant="ghost" size="icon">`, `UserAvatarMenu`
  - [x] 5.4 Mark `TopBar` with `"use client"` (DropdownMenu interaction)

- [x] Task 6: Create `NotificationsBell` component in `packages/ui` (AC: 6)
  - [x] 6.1 Create `packages/ui/src/components/app-shell/NotificationsBell.tsx` — renders `<Button variant="ghost" size="icon">` with Lucide `Bell` icon; overlaid `<Badge>` showing static count `"3"` positioned absolutely; add `data-testid="notifications-bell"` and `data-testid="notifications-badge"` on badge element

- [x] Task 7: Create `UserAvatarMenu` component in `packages/ui` (AC: 5)
  - [x] 7.1 Create `packages/ui/src/components/app-shell/UserAvatarMenu.tsx` — accepts `user?: { name: string; email: string }` prop; renders `<DropdownMenu>` trigger as `<Avatar>` with fallback initials; menu content: `<DropdownMenuLabel>` with name + email (2 lines), `<DropdownMenuItem>Profile</DropdownMenuItem>`, `<DropdownMenuItem>Settings</DropdownMenuItem>`, `<DropdownMenuSeparator />`, `<DropdownMenuItem className="text-red-600">Sign out</DropdownMenuItem>`; add `data-testid="user-avatar-menu-trigger"` on trigger button
  - [x] 7.2 Mark `UserAvatarMenu` with `"use client"`

- [x] Task 8: Create `Breadcrumbs` component in `packages/ui` (AC: 4)
  - [x] 8.1 Create `packages/ui/src/components/app-shell/Breadcrumbs.tsx` — accepts `items: { label: string; href?: string }[]`; renders breadcrumb trail with `/` separators; last item non-linked; styled `text-sm text-slate-500`; active (last) item `text-slate-900 font-medium`
  - [x] 8.2 Mark `Breadcrumbs` with `"use client"`

- [x] Task 9: Create `AppShell` wrapper component in `packages/ui` (AC: 1, 7)
  - [x] 9.1 Create `packages/ui/src/components/app-shell/AppShell.tsx` — accepts `sidebar: React.ReactNode`, `topbar: React.ReactNode`, `children: React.ReactNode`; renders: fixed sidebar slot on left, flex column on right containing topbar + scrollable content area; layout: `<div className="flex h-screen overflow-hidden">`; content area: `<main className="flex-1 overflow-y-auto">`
  - [x] 9.2 Add `data-testid="app-shell"`, `data-testid="sidebar"`, `data-testid="topbar"`, `data-testid="content-area"` on key elements for E2E tests

- [x] Task 10: Export all app-shell components from `packages/ui` (AC: 1)
  - [x] 10.1 Update `packages/ui/index.ts` — add named exports for: `AppShell`, `Sidebar`, `NavItem`, `TopBar`, `NotificationsBell`, `UserAvatarMenu`, `Breadcrumbs`; also export `NavItemConfig` type

- [x] Task 11: Wire client app shell — `(protected)` route group (AC: 8)
  - [x] 11.1 Create `apps/client/app/(protected)/layout.tsx` — `"use client"` layout; reads `sidebarCollapsed` from `useUIStore`; handles hydration via mounted state (render `sidebarCollapsed` default `false` until mounted); renders `<AppShell>` with client nav items; client nav items with Lucide icons: Dashboard (`LayoutDashboard`), Tenders (`FileText`), Offers (`Briefcase`), Documents (`FolderOpen`), Team (`Users`), Settings (`Settings`)
  - [x] 11.2 Create `apps/client/app/(protected)/dashboard/page.tsx` — simple placeholder: `<h1 className="text-2xl font-semibold text-slate-900">Dashboard</h1><p className="text-slate-500 mt-2">Welcome to EU Solicit.</p>`; add `data-testid="dashboard-page"`

- [x] Task 12: Wire admin app shell — `(protected)` route group (AC: 9)
  - [x] 12.1 Create `apps/admin/app/(protected)/layout.tsx` — same pattern as client but with admin nav items; admin nav items with Lucide icons: Dashboard (`LayoutDashboard`), Companies (`Building2`), Tenders (`FileText`), Users (`Users`), Reports (`BarChart2`), Settings (`Settings`)
  - [x] 12.2 Create `apps/admin/app/(protected)/dashboard/page.tsx` — placeholder page with `data-testid="dashboard-page"`

- [x] Task 13: Add Lucide icons dependency to both apps (AC: 8, 9)
  - [x] 13.1 Add `"lucide-react": "^0.400.0"` to `dependencies` in `apps/client/package.json` (already in `packages/ui` but apps need it for icon imports in layouts)
  - [x] 13.2 Add `"lucide-react": "^0.400.0"` to `dependencies` in `apps/admin/package.json`
  - [x] 13.3 Run `pnpm install` from `frontend/` root

- [x] Task 14: Verify build (AC: 11)
  - [x] 14.1 Run `pnpm build` from `frontend/` — both apps must exit 0
  - [x] 14.2 Run `pnpm type-check` — zero TypeScript errors
  - [x] 14.3 Manually verify: `pnpm dev --filter client` → navigate to `http://localhost:3000/dashboard` → shell renders with sidebar and top bar

## Dev Notes

### Working Directory

All frontend code: `eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

### Critical Learnings from Stories 3.1 and 3.2 (MUST APPLY)

1. **`next.config.mjs` not `.ts`** — Both apps already use `.mjs`. Do NOT create `.ts` next configs.
2. **Turborepo v2 `tasks` not `pipeline`** — `turbo.json` already correct; don't touch it.
3. **ESLint `require.resolve()` workaround** — already in `packages/config/.eslintrc.js`; don't modify.
4. **`@/*` maps to `./` not `./src/`** — Fixed in 3.2. `@/lib/stores/ui-store` resolves to `apps/*/lib/stores/ui-store.ts`.
5. **`presets: [sharedPreset]` not spread** — Already fixed in 3.2. Don't touch tailwind configs.
6. **`suppressHydrationWarning` is already on `<html>` tag** in both layouts — don't add it anywhere else.
7. **`lucide-react ^0.400.0`** is already in `packages/ui/package.json`. If importing icons in `packages/ui` components, they're already available. For app-level layouts importing icons, add the dep to each app.

### Existing Packages Already Available (no new installs needed for packages/ui)

All of these are already exported from `@eusolicit/ui` (from `packages/ui/index.ts`):
- `Sheet`, `SheetContent`, `SheetHeader`, `SheetTitle`, `SheetTrigger`, `SheetClose` — for mobile sidebar overlay (Story 3.4)
- `Avatar`, `AvatarImage`, `AvatarFallback`
- `DropdownMenu`, `DropdownMenuContent`, `DropdownMenuItem`, `DropdownMenuLabel`, `DropdownMenuSeparator`, `DropdownMenuTrigger`
- `Tooltip`, `TooltipProvider`, `TooltipTrigger`, `TooltipContent`
- `Badge`, `badgeVariants`
- `Button`, `buttonVariants`
- `Separator`
- `cn` (utility)

### Route Group `(protected)` Pattern

In Next.js 14 App Router, `(protected)` is a **route group** — the parentheses mean the folder does NOT appear in the URL. So `app/(protected)/dashboard/page.tsx` maps to `/dashboard`.

Both apps currently have only:
- `app/layout.tsx` (root layout — keeps fonts, `<html>`, `<body>`)
- `app/page.tsx` (root placeholder)

The `(protected)/layout.tsx` wraps only protected routes, not auth pages. This is the correct Next.js layout nesting pattern for this use case.

### Zustand `uiStore` — Minimal Version for S3.3

S3.5 will define the **full** `uiStore` with `theme`, `locale`, `toasts[]`, etc. S3.3 creates only the sidebar slice. **S3.5 MUST expand this store, not replace it** — it should add properties to the same store file/key.

```typescript
// apps/client/lib/stores/ui-store.ts
"use client";

import { create } from "zustand";
import { persist } from "zustand/middleware";

interface UIState {
  sidebarCollapsed: boolean;
  toggleSidebar: () => void;
  setSidebarCollapsed: (collapsed: boolean) => void;
}

export const useUIStore = create<UIState>()(
  persist(
    (set) => ({
      sidebarCollapsed: false,
      toggleSidebar: () =>
        set((state) => ({ sidebarCollapsed: !state.sidebarCollapsed })),
      setSidebarCollapsed: (collapsed) => set({ sidebarCollapsed: collapsed }),
    }),
    {
      name: "eusolicit-ui-store", // admin uses "eusolicit-admin-ui-store"
    }
  )
);
```

### SSR Hydration Mismatch Prevention (E03-R-004)

Zustand `persist` middleware stores state in `localStorage` (browser-only). During SSR, the store returns the default value (`sidebarCollapsed: false`). After client hydration, it reads from `localStorage` — if persisted value differs from default, React throws a hydration error.

**Required pattern in `(protected)/layout.tsx`:**

```tsx
"use client";

import { useEffect, useState } from "react";
import { useUIStore } from "@/lib/stores/ui-store";

export default function ProtectedLayout({ children }: { children: React.ReactNode }) {
  const { sidebarCollapsed, toggleSidebar } = useUIStore();
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    setMounted(true);
  }, []);

  // Use false (default) until mounted to prevent hydration mismatch
  const collapsed = mounted ? sidebarCollapsed : false;

  return (
    <AppShell
      sidebar={<Sidebar ... isCollapsed={collapsed} onToggle={toggleSidebar} />}
      topbar={<TopBar ... />}
    >
      {children}
    </AppShell>
  );
}
```

This avoids `suppressHydrationWarning` on content areas and prevents any flash.

### `NavItemConfig` Type Definition

Define in `packages/ui/src/components/app-shell/NavItem.tsx` and re-export:

```typescript
export interface NavItemConfig {
  icon: React.ElementType;
  label: string;
  href: string;
}
```

### Sidebar CSS Pattern — Width Transition

```tsx
// Sidebar.tsx
<aside
  data-testid="sidebar"
  className={cn(
    "fixed inset-y-0 left-0 z-30 flex flex-col bg-white border-r border-slate-200",
    "transition-[width] duration-200 ease-in-out",
    isCollapsed ? "w-16" : "w-64"
  )}
>
```

The `transition-[width]` Tailwind class uses CSS `transition` property for width changes. The `duration-200 ease-in-out` controls the 200ms ease animation per the epic spec.

### Main Content Area — Sidebar Offset

The main content must be offset to account for the sidebar width. Apply margin-left that matches sidebar width and also transitions:

```tsx
// AppShell.tsx or ProtectedLayout
<div
  className={cn(
    "transition-[margin-left] duration-200 ease-in-out",
    isCollapsed ? "ml-16" : "ml-64"
  )}
>
  {/* topbar + content */}
</div>
```

### Active Nav Detection with `usePathname()`

```tsx
// NavItem.tsx
"use client";
import { usePathname } from "next/navigation";
import Link from "next/link";

export function NavItem({ icon: Icon, label, href, isCollapsed }: NavItemProps) {
  const pathname = usePathname();
  const isActive = pathname === href || pathname.startsWith(href + "/");

  return (
    <Link
      href={href}
      className={cn(
        "flex items-center gap-3 px-3 py-2 rounded-md text-sm transition-colors",
        isActive
          ? "bg-indigo-50 text-indigo-600 font-medium"
          : "text-slate-600 hover:bg-slate-100 hover:text-slate-900"
      )}
    >
      <Icon className="h-5 w-5 flex-shrink-0" />
      {!isCollapsed && <span className="truncate">{label}</span>}
    </Link>
  );
}
```

### User Avatar Menu — Fallback Initials

When no `avatarUrl` is provided (placeholder for S3.3), use initials extracted from `user.name`:

```tsx
const initials = user?.name
  ?.split(" ")
  .map((n) => n[0])
  .join("")
  .toUpperCase()
  .slice(0, 2) ?? "U";
```

### `packages/ui` — Client Components Pattern

`packages/ui` components that use hooks (`usePathname`, `useState`, `useEffect`) or event handlers **must** have `"use client"` as the first line. Server components importing them will use the client boundary. This is the correct Next.js 14 App Router pattern.

**Do NOT** add `"use client"` to `packages/ui/index.ts` (the barrel export) — only to individual component files that need it.

### `next/navigation` in `packages/ui`

`usePathname()` from `next/navigation` works in `packages/ui` when imported by a Next.js app because `next` is a peer dependency. The `transpilePackages: ['@eusolicit/ui']` config in each app's `next.config.mjs` ensures this resolves correctly (already set up in S3.1).

### File Structure Reference

```
eusolicit-app/frontend/
├── apps/
│   ├── client/
│   │   ├── app/
│   │   │   ├── (protected)/              ← NEW route group
│   │   │   │   ├── layout.tsx            ← NEW app shell layout
│   │   │   │   └── dashboard/
│   │   │   │       └── page.tsx          ← NEW placeholder
│   │   │   ├── layout.tsx                ← EXISTING (keep unchanged)
│   │   │   ├── page.tsx                  ← EXISTING (keep unchanged)
│   │   │   └── dev/components/page.tsx   ← EXISTING (keep unchanged)
│   │   ├── lib/
│   │   │   ├── stores/
│   │   │   │   └── ui-store.ts           ← NEW
│   │   │   └── utils.ts                  ← EXISTING (keep unchanged)
│   │   └── package.json                  ← MODIFY (add zustand, lucide-react)
│   └── admin/
│       ├── app/
│       │   ├── (protected)/              ← NEW route group
│       │   │   ├── layout.tsx            ← NEW app shell layout
│       │   │   └── dashboard/
│       │   │       └── page.tsx          ← NEW placeholder
│       │   ├── layout.tsx                ← EXISTING (keep unchanged)
│       │   └── page.tsx                  ← EXISTING (keep unchanged)
│       ├── lib/
│       │   ├── stores/
│       │   │   └── ui-store.ts           ← NEW
│       │   └── utils.ts                  ← EXISTING (keep unchanged)
│       └── package.json                  ← MODIFY (add zustand, lucide-react)
└── packages/
    └── ui/
        ├── src/
        │   └── components/
        │       ├── app-shell/            ← NEW directory
        │       │   ├── AppShell.tsx      ← NEW
        │       │   ├── Sidebar.tsx       ← NEW
        │       │   ├── NavItem.tsx       ← NEW (+ NavItemConfig type)
        │       │   ├── TopBar.tsx        ← NEW
        │       │   ├── Breadcrumbs.tsx   ← NEW
        │       │   ├── NotificationsBell.tsx ← NEW
        │       │   └── UserAvatarMenu.tsx ← NEW
        │       └── ui/                   ← EXISTING (keep unchanged)
        └── index.ts                      ← MODIFY (add app-shell exports)
```

### What NOT to Touch

- `apps/client/app/layout.tsx` — root layout with fonts; do NOT modify
- `apps/admin/app/layout.tsx` — root layout; do NOT modify
- `apps/*/app/globals.css` — CSS variables set in 3.2; do NOT modify
- `packages/config/tailwind.config.ts` — design tokens from 3.2; do NOT modify
- `apps/*/tailwind.config.ts` — preset setup from 3.2; do NOT modify
- `packages/ui/src/components/ui/**` — shadcn components from 3.2; do NOT modify
- `turbo.json` — Turborepo v2 config; do NOT modify

### Test Expectations from Epic-Level Test Design

These specific test scenarios from `test-design-epic-03.md` apply to this story:

**Must Pass (P0 — blocks demo):**
- **E03-P0-009**: Client app shell renders correctly — navigate to `/dashboard` (authenticated); assert sidebar nav items present (`data-testid` on nav items); assert top bar visible with avatar menu trigger; assert `data-testid="app-shell"` present
- **E03-P0-010**: Admin app shell renders with admin-specific nav: Dashboard, Companies, Tenders, Users, Reports, Settings — navigate to admin app (port 3001) `/dashboard`

**High (P1):**
- **E03-P1-010**: Sidebar toggle — collapses to icon-only (labels hidden, width ~64px); expand restores labels; state persists in `uiStore`; on reload collapsed state is preserved (from localStorage)

**Medium (P2):**
- **E03-P2-004**: User avatar dropdown shows: user name, email, "Profile", "Settings", divider, "Sign out" — all 5 items + separator present
- **E03-P2-005**: Notifications bell renders with unread count badge; `data-testid="notifications-bell"` and `data-testid="notifications-badge"` present with non-empty count

**Required data-testid attributes for E2E testing:**
- `data-testid="app-shell"` — AppShell root div
- `data-testid="sidebar"` — Sidebar `<aside>`
- `data-testid="topbar"` — TopBar `<header>`
- `data-testid="content-area"` — Main content `<main>`
- `data-testid="sidebar-toggle"` — Toggle button in sidebar
- `data-testid="notifications-bell"` — Bell button
- `data-testid="notifications-badge"` — Unread count badge
- `data-testid="user-avatar-menu-trigger"` — Avatar trigger button
- `data-testid="nav-item-{label}"` — Each NavItem (e.g., `data-testid="nav-item-dashboard"`, lowercase)
- `data-testid="dashboard-page"` — Dashboard placeholder page

### Risk Mitigation Required

**E03-R-004 (Zustand SSR hydration mismatch, score 4):** Use the `mounted` flag pattern in `(protected)/layout.tsx` (see Zustand SSR section above). Do NOT use `suppressHydrationWarning` on content elements — only the root `<html>` tag has it (already set in root layout).

**E03-R-005 (Responsive breakpoint regression, score 4):** This story does NOT implement responsive breakpoints — that is Story 3.4. However, do NOT hardcode sidebar visibility in a way that breaks responsive implementation. The `AppShell` and `Sidebar` must accept `isCollapsed` as a prop (not hard-code it internally) so S3.4 can drive it with breakpoint logic.

### Language Selector Placeholder

S3.7 implements `next-intl`. For S3.3, render a simple `<Button variant="ghost" size="sm">BG</Button>` as a placeholder in the TopBar action cluster. S3.7 will replace this with the real language selector. Add `data-testid="language-selector"` to it.

### What S3.5 Will Add to `uiStore`

S3.5 will expand `ui-store.ts` in both apps to add: `theme: "light" | "dark" | "system"`, `locale: "bg" | "en"`, `toasts: Toast[]`, `addToast()`, `removeToast()`. Do NOT pre-add these fields in S3.3. S3.5 is responsible for the full store definition.

### Project Structure Notes

- Alignment: App Router route groups (`(protected)`) are the standard Next.js 14 pattern for layout nesting without URL segments
- `packages/ui` exports server-safe by default; individual client components use `"use client"` directive
- `lib/stores/` is consistent with the established `lib/` convention in each app (see `lib/utils.ts` from S3.2)

### References

- [Source: eusolicit-docs/planning-artifacts/epic-03-frontend-shell-design-system.md#S03.03]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#P0 E03-P0-009, E03-P0-010]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#P1 E03-P1-010]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#Risk E03-R-004, E03-R-005]
- [Source: eusolicit-docs/implementation-artifacts/3-2-tailwind-design-token-preset-shadcn-ui-theming.md#Dev Notes - Critical Story 3.1 Learnings]
- [Source: eusolicit-docs/implementation-artifacts/3-1-next-js-14-monorepo-scaffold.md#Dev Notes]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

None — build succeeded on first attempt.

### Completion Notes List

- All 14 tasks completed; `pnpm build` exits 0 for both client and admin apps.
- `pnpm type-check` exits 0 — zero TypeScript errors.
- `NotificationsBell` uses an inline `<span>` (rather than shadcn `<Badge>`) for the overlay badge, positioned absolutely via Tailwind, to satisfy the E2E overlay-positioning assertions.
- Language selector placeholder uses `<Button variant="ghost" size="sm">BG</Button>` per dev notes; `data-testid="language-selector"` attached.
- SSR hydration mismatch prevention implemented in both `(protected)/layout.tsx` files via `mounted` flag pattern (default `false` until `useEffect` fires on client).
- `AppShell` accepts `isCollapsed` prop and passes it as `ml-16`/`ml-64` margin to the right column, enabling the CSS width transition to animate content offset in sync with sidebar.
- ATDD tests transitioned from RED to GREEN phase — all `test.skip()` calls removed from both spec files.

### File List

**New files created:**
- `eusolicit-app/frontend/apps/client/lib/stores/ui-store.ts`
- `eusolicit-app/frontend/apps/admin/lib/stores/ui-store.ts`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/AppShell.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/Sidebar.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/NavItem.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/TopBar.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/Breadcrumbs.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/NotificationsBell.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/UserAvatarMenu.tsx`
- `eusolicit-app/frontend/apps/client/app/(protected)/layout.tsx`
- `eusolicit-app/frontend/apps/client/app/(protected)/dashboard/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/(protected)/layout.tsx`
- `eusolicit-app/frontend/apps/admin/app/(protected)/dashboard/page.tsx`

**Modified files:**
- `eusolicit-app/frontend/packages/ui/index.ts` — added app-shell exports
- `eusolicit-app/frontend/apps/client/package.json` — added zustand, lucide-react
- `eusolicit-app/frontend/apps/admin/package.json` — added zustand, lucide-react
- `eusolicit-app/e2e/specs/shell/app-shell-layout.spec.ts` — removed test.skip(), GREEN phase
- `eusolicit-app/e2e/specs/shell/app-shell-layout.admin.spec.ts` — removed test.skip(), GREEN phase

## Senior Developer Review

**Verdict**: APPROVE

**Review Date**: 2026-04-08
**Reviewer**: claude-opus-4-5 (adversarial code review — Blind Hunter, Edge Case Hunter, Acceptance Auditor)

### Acceptance Criteria Audit

| AC | Status | Notes |
|----|--------|-------|
| 1 | PASS | `AppShell` accepts `sidebar`, `topbar`, `children` slots; exported from `@eusolicit/ui` via `index.ts` |
| 2 | PASS | `NavItem` renders with `icon`, `label`, `href`; active state uses `bg-indigo-50 text-indigo-600 font-medium` matching spec exactly |
| 3 | PASS | Sidebar uses `w-64`/`w-16` (256px/64px) with `transition-[width] duration-200 ease-in-out`; labels hidden via conditional render `{!isCollapsed && ...}` |
| 4 | PASS | TopBar is `sticky top-0 z-20 h-16`; left-side has `leftSlot` for Breadcrumbs (Task 5.2: "or an empty div"); right-side has NotificationsBell, language selector "BG" placeholder, UserAvatarMenu; `Breadcrumbs` component created and exported per Task 8 |
| 5 | PASS | Dropdown shows: name, email in `DropdownMenuLabel`; `DropdownMenuSeparator`; Profile, Settings as `DropdownMenuItem`; `DropdownMenuSeparator`; Sign out with `text-red-600` — matches Task 7.1 spec |
| 6 | PASS | `NotificationsBell` shows unread badge with static "3" per Task 6.1; `data-testid="notifications-badge"` present |
| 7 | PASS | Content `<main>` has `overflow-y-auto`; Sidebar is `fixed`; TopBar is `sticky top-0`; root shell is `h-screen overflow-hidden` |
| 8 | PASS | Client `(protected)/layout.tsx` renders: Dashboard, Tenders, Offers, Documents, Team, Settings — 6 items matching spec exactly |
| 9 | PASS | Admin `(protected)/layout.tsx` renders: Dashboard, Companies, Tenders, Users, Reports, Settings — 6 items matching spec exactly |
| 10 | PASS | Zustand stores with `persist` middleware; client key `"eusolicit-ui-store"`, admin key `"eusolicit-admin-ui-store"`; `mounted` flag pattern prevents SSR hydration mismatch (E03-R-004) |
| 11 | PASS | `pnpm build` exits 0; both apps compile successfully; Turbo reports 2/2 tasks successful |

### data-testid Verification

All 10 required `data-testid` attributes confirmed present: `app-shell`, `sidebar`, `topbar`, `content-area`, `sidebar-toggle`, `notifications-bell`, `notifications-badge`, `user-avatar-menu-trigger`, `nav-item-{label}` (lowercase), `dashboard-page`. Additionally `language-selector` is present per dev notes.

### E2E Test Coverage

ATDD tests transitioned from RED to GREEN. Both spec files (`app-shell-layout.spec.ts`, `app-shell-layout.admin.spec.ts`) are active and cover:
- E03-P0-009: Client shell renders (sidebar, topbar, content, nav items, active state, hrefs)
- E03-P0-010: Admin shell renders with admin-specific nav items; negative test confirms client-only items absent
- E03-P1-010: Sidebar toggle collapse/expand, localStorage persistence, reload persistence, SSR hydration error detection
- E03-P2-004: Avatar dropdown items (Profile, Settings, separator, Sign out), open/close behavior
- E03-P2-005: Notifications bell, badge visibility, static count "3", overlay positioning

### Risk Mitigations Verified

- **E03-R-004** (Zustand SSR hydration): `mounted` flag pattern correctly implemented in both layouts — server and first client render use `collapsed = false`, only reading persisted value post-hydration.
- **E03-R-005** (Responsive breakpoint): `isCollapsed` is a prop (not internal state), enabling S3.4 to drive it with breakpoint logic. `Sidebar` and `AppShell` are fully controllable from outside.

### Advisory Notes (Non-Blocking)

These are observations for future stories — none block this story's approval:

1. **Unused `Badge` import** (NotificationsBell.tsx L5): `Badge` from `../ui/badge` is imported but not used — a raw `<span>` is used for the overlay badge instead. Dead import; should be cleaned up in a future pass.

2. **`"use client"` on store files** (both `ui-store.ts`): The `"use client"` directive on Zustand store modules is semantically unnecessary — stores are not React components. They will be pulled into the client bundle automatically via client component imports. Harmless but non-standard.

3. **Hydration layout shift** (both protected layouts): The `mounted` flag pattern correctly prevents React hydration errors but produces a visible layout shift when a user reloads with `sidebarCollapsed: true` persisted. The sidebar briefly shows expanded then snaps to collapsed. This is an inherent trade-off of the pattern specified by the story (E03-R-004). A future enhancement could suppress the CSS transition during initial hydration (e.g., `transition-none` class removed after mount) or use `opacity-0` until mounted.

4. **`isCollapsed` passed independently to `AppShell` and `Sidebar`**: The collapsed state drives `ml-16`/`ml-64` on the content wrapper (in `AppShell`) and `w-16`/`w-64` on the sidebar (in `Sidebar`) via two separate props from the parent layout. A prop mismatch would cause layout desync. The current layouts correctly pass the same `collapsed` value to both. Consider consolidating into a single component or React context in a future refactor if complexity grows.

5. **Accessibility enhancements for future stories**: (a) Add `aria-current="page"` to the active `NavItem` link; (b) Add a "Skip to main content" link as first focusable element in `AppShell` (WCAG 2.4.1); (c) Add `aria-label="Main navigation"` to the sidebar `<nav>` element. These can be addressed in the accessibility story if one exists in the backlog.

6. **NavItem `data-testid` uses `label.toLowerCase()`**: Works well for English labels but will produce non-ASCII test IDs when labels are localized (S3.7 i18n). Consider deriving test IDs from `href` instead (e.g., `nav-item-${href.replace(/^\//, '')}`) when i18n is added.

### Build Verification

```
pnpm build → EXIT_CODE=0
Tasks: 2 successful, 2 total
Both apps: Compiled successfully, static pages generated, zero TypeScript errors
```
