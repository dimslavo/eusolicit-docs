# Story 3.4: Responsive Layout Strategy

Status: done

## Story

As a **frontend developer on the EU Solicit team**,
I want **the app shell to adapt its layout automatically across desktop, tablet, and mobile breakpoints — showing a full sidebar on desktop, auto-collapsing it on tablet, and replacing it with a bottom navigation bar on mobile (with an overlay sheet reachable via a hamburger icon)**,
so that **every feature epic built on top of this shell works correctly on all device sizes without per-page responsive code**.

## Acceptance Criteria

1. Desktop (≥1280px): full sidebar visible by default; user can manually collapse/expand via the toggle button; collapsed state from `uiStore` is respected
2. Tablet (768–1279px): sidebar auto-collapses to icon-only mode (`isCollapsed=true`) when breakpoint is first detected; user may expand via toggle but sidebar auto-collapses again on the next route change
3. Mobile (<768px): sidebar is hidden; a fixed bottom navigation bar (`<BottomNav>`) replaces it, showing the first 5 primary nav items with icon + label
4. Bottom nav bar: fixed, 64px height, safe-area-aware (respects `env(safe-area-inset-bottom)` for notched devices), `bg-white` with top border
5. Bottom nav highlights active route with `text-indigo-600` icon and label colour; inactive items use `text-slate-500`
6. On mobile, a hamburger icon (`Menu` from Lucide) in the top-left of the `TopBar` opens the full sidebar as a Sheet overlay (left-side); the sheet closes on route change or native dismiss
7. Layout transitions are smooth and no content reflow or jump occurs at breakpoint crossings; `ml-0` applied to the right column on mobile so no sidebar offset exists
8. Both client and admin apps implement the responsive behaviour
9. `pnpm build` exits 0 for both apps after this story's changes

## Tasks / Subtasks

- [x] Task 1: Create `useBreakpoint` hook in `packages/ui` (AC: 1, 2, 3, 7)
  - [x] 1.1 Create `packages/ui/src/lib/hooks/useBreakpoint.ts` — returns `{ breakpoint: "mobile" | "tablet" | "desktop", isMobile, isTablet, isDesktop }`; SSR-safe default is `"desktop"`; uses `window.matchMedia("(min-width: 768px)")` and `window.matchMedia("(min-width: 1280px)")` with `addEventListener("change", ...)` (fires only at breakpoint crossings, not every resize pixel); runs `update()` synchronously on mount to detect actual viewport; cleans up listeners on unmount; mark `"use client"`
  - [x] 1.2 Export `useBreakpoint` from `packages/ui/src/lib/hooks/index.ts` (create if it doesn't exist) and re-export from `packages/ui/index.ts`

- [x] Task 2: Create `BottomNav` component in `packages/ui` (AC: 3, 4, 5)
  - [x] 2.1 Create `packages/ui/src/components/app-shell/BottomNav.tsx` — accepts `navItems: NavItemConfig[]` (renders first 5 items only); fixed bottom bar: `fixed bottom-0 left-0 right-0 z-30 bg-white border-t border-slate-200`; inner container: `flex items-center justify-around h-16 [padding-bottom:env(safe-area-inset-bottom)]`; add `data-testid="bottom-nav"` on root element
  - [x] 2.2 Each item in BottomNav: renders `<Link href={...}>` with icon + label stacked vertically (`flex flex-col items-center gap-0.5`); uses `usePathname()` for active state; active: `text-indigo-600`; inactive: `text-slate-500 hover:text-slate-700`; icon `h-5 w-5`, label `text-[10px] font-medium`; add `data-testid="bottom-nav-item-{label}"` (lowercase label, e.g. `bottom-nav-item-dashboard`)
  - [x] 2.3 Mark `BottomNav` with `"use client"` (uses `usePathname`)

- [x] Task 3: Create `MobileSidebarSheet` component in `packages/ui` (AC: 6)
  - [x] 3.1 Create `packages/ui/src/components/app-shell/MobileSidebarSheet.tsx` — accepts `open: boolean`, `onOpenChange: (open: boolean) => void`, `navItems: NavItemConfig[]`; wraps shadcn `<Sheet open={open} onOpenChange={onOpenChange}>`; `<SheetContent side="left" className="w-64 p-0">` with `data-testid="mobile-sidebar-sheet"` on `SheetContent`; renders a header area (`h-16 px-4 border-b border-slate-200 flex items-center`) with "EU Solicit" title; renders a `<nav>` with all `navItems` using `<NavItem isCollapsed={false} />` for each
  - [x] 3.2 Mark `MobileSidebarSheet` with `"use client"`

- [x] Task 4: Update `AppShell` to support `isMobile` prop (AC: 3, 7)
  - [x] 4.1 Add `isMobile?: boolean` and `bottomNav?: React.ReactNode` props to `AppShellProps` in `packages/ui/src/components/app-shell/AppShell.tsx`
  - [x] 4.2 Conditionally render the sidebar slot: `{!isMobile && sidebar}` — sidebar is hidden on mobile
  - [x] 4.3 Right column margin: `isMobile ? "ml-0" : isCollapsed ? "ml-16" : "ml-64"` — no sidebar offset on mobile
  - [x] 4.4 Add `pb-16` to `<main>` content area when `isMobile && bottomNav` — prevents bottom nav from obscuring content: `isMobile && bottomNav ? "pb-16" : ""`
  - [x] 4.5 Render `{bottomNav}` after `</div>` (right column close), so it is fixed-positioned at bottom of viewport

- [x] Task 5: Update `TopBar` to support hamburger icon on mobile (AC: 6)
  - [x] 5.1 Add `onOpenMobileSidebar?: () => void` prop to `TopBarProps` in `packages/ui/src/components/app-shell/TopBar.tsx`
  - [x] 5.2 Import `Menu` from `lucide-react`
  - [x] 5.3 When `onOpenMobileSidebar` is provided, render a `<button data-testid="hamburger-menu" onClick={onOpenMobileSidebar}>` with `<Menu className="h-5 w-5" />` as the first child in the left side div (before `leftSlot`); style: `flex items-center justify-center h-8 w-8 rounded-md text-slate-500 hover:bg-slate-100 hover:text-slate-900 transition-colors flex-shrink-0`; `aria-label="Open menu"`

- [x] Task 6: Export new components and hook from `packages/ui` (AC: 1, 2, 3, 6)
  - [x] 6.1 Update `packages/ui/index.ts` — add exports: `BottomNav`, `MobileSidebarSheet`, `useBreakpoint`

- [x] Task 7: Update client `(protected)/layout.tsx` with responsive logic (AC: 1, 2, 3, 6, 7, 8)
  - [x] 7.1 Import `usePathname` from `next/navigation`, `useBreakpoint` and `BottomNav`, `MobileSidebarSheet` from `@eusolicit/ui` in `apps/client/app/(protected)/layout.tsx`
  - [x] 7.2 Add state: `const [isMobileSheetOpen, setIsMobileSheetOpen] = useState(false)`
  - [x] 7.3 Add `const pathname = usePathname()` to detect route changes
  - [x] 7.4 Add `const { isMobile, isTablet } = useBreakpoint()` (SSR-safe: returns `isDesktop=true` until mount, preventing hydration mismatch)
  - [x] 7.5 Add `useEffect` for breakpoint-driven collapse: `if (isTablet && mounted) { setSidebarCollapsed(true); }` — deps: `[isTablet, mounted]`
  - [x] 7.6 Add `useEffect` for route-change auto-collapse + sheet close: `if (isTablet && mounted) { setSidebarCollapsed(true); } setIsMobileSheetOpen(false);` — deps: `[pathname]`
  - [x] 7.7 Update `AppShell` usage: add `isMobile={isMobile && mounted}` and `bottomNav={isMobile && mounted ? <BottomNav navItems={clientNavItems.slice(0, 5)} /> : undefined}` props
  - [x] 7.8 Pass `onOpenMobileSidebar` to `TopBar`: `onOpenMobileSidebar={isMobile && mounted ? () => setIsMobileSheetOpen(true) : undefined}`
  - [x] 7.9 Render `<MobileSidebarSheet open={isMobileSheetOpen} onOpenChange={setIsMobileSheetOpen} navItems={clientNavItems} />` inside the layout JSX (sibling to `<AppShell>` via a Fragment)

- [x] Task 8: Update admin `(protected)/layout.tsx` with responsive logic (AC: 1, 2, 3, 6, 7, 8)
  - [x] 8.1 Apply the same changes as Task 7 to `apps/admin/app/(protected)/layout.tsx`, using `adminNavItems` instead of `clientNavItems`

- [x] Task 9: Verify build and smoke test (AC: 9)
  - [x] 9.1 Run `pnpm build` from `eusolicit-app/frontend/` — both apps must exit 0
  - [x] 9.2 Run `pnpm type-check` — zero TypeScript errors
  - [x] 9.3 Manually verify in Chrome DevTools: set viewport to 375px → bottom nav visible, sidebar hidden, hamburger in top bar → click hamburger → Sheet opens with nav items; set to 900px → sidebar icon-only; set to 1280px → full sidebar visible

## Dev Notes

### Working Directory

All frontend code lives under: `eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

### Critical Learnings from Stories 3.1–3.3 (MUST APPLY)

1. **`next.config.mjs` not `.ts`** — Both apps use `.mjs`. Do NOT create `.ts` next configs.
2. **Turborepo v2 `tasks` not `pipeline`** — `turbo.json` is already correct; don't touch it.
3. **ESLint `require.resolve()` workaround** — already in `packages/config/.eslintrc.js`; don't modify.
4. **`@/*` maps to `./` not `./src/`** — `@/lib/stores/ui-store` resolves to `apps/*/lib/stores/ui-store.ts`.
5. **`presets: [sharedPreset]` not spread** — Already fixed in S3.2. Don't touch tailwind configs.
6. **`suppressHydrationWarning` is already on `<html>` tag** — don't add it anywhere else.
7. **`lucide-react ^0.400.0`** is already in `packages/ui/package.json` and both app `package.json` files (added in S3.3). Do NOT install again.
8. **`mounted` flag pattern** — Both `(protected)/layout.tsx` files already have `const [mounted, setMounted] = useState(false)` + `useEffect(() => setMounted(true), [])`. Story 3.4 builds on this pattern; do NOT add a second `mounted` flag — reuse the existing one.

### `useBreakpoint` Hook Implementation

Create in `packages/ui/src/lib/hooks/useBreakpoint.ts`. This file likely requires creating the `hooks/` directory inside `packages/ui/src/lib/`.

```typescript
// packages/ui/src/lib/hooks/useBreakpoint.ts
"use client";

import { useEffect, useState } from "react";

type Breakpoint = "mobile" | "tablet" | "desktop";

function getBreakpoint(): Breakpoint {
  if (typeof window === "undefined") return "desktop"; // SSR fallback
  const width = window.innerWidth;
  if (width < 768) return "mobile";
  if (width < 1280) return "tablet";
  return "desktop";
}

export function useBreakpoint() {
  // "desktop" as SSR default ensures no SSR/client mismatch flash on desktop (the most common case)
  const [breakpoint, setBreakpoint] = useState<Breakpoint>("desktop");

  useEffect(() => {
    // Synchronously set actual breakpoint after mount
    setBreakpoint(getBreakpoint());

    const update = () => setBreakpoint(getBreakpoint());

    // matchMedia events fire only at breakpoint crossings — no debounce needed
    const mdQuery = window.matchMedia("(min-width: 768px)");
    const lgQuery = window.matchMedia("(min-width: 1280px)");

    mdQuery.addEventListener("change", update);
    lgQuery.addEventListener("change", update);

    return () => {
      mdQuery.removeEventListener("change", update);
      lgQuery.removeEventListener("change", update);
    };
  }, []);

  return {
    breakpoint,
    isMobile: breakpoint === "mobile",
    isTablet: breakpoint === "tablet",
    isDesktop: breakpoint === "desktop",
  };
}
```

**Why `matchMedia` over `resize`:** The `resize` event fires continuously during drag. `matchMedia` fires only when crossing 768px or 1280px thresholds — no debounce needed and no excessive re-renders (E03-R-005 mitigation).

**SSR safety:** The hook returns `"desktop"` until `useEffect` runs on the client. Combined with the existing `mounted` flag in the layouts, this ensures no hydration mismatch.

### `BottomNav` Component

```tsx
// packages/ui/src/components/app-shell/BottomNav.tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import { cn } from "../../lib/utils";
import type { NavItemConfig } from "./NavItem";

interface BottomNavProps {
  navItems: NavItemConfig[]; // Caller passes .slice(0, 5) — only first 5 shown
}

export function BottomNav({ navItems }: BottomNavProps) {
  const pathname = usePathname();
  const items = navItems.slice(0, 5);

  return (
    <div
      data-testid="bottom-nav"
      className="fixed bottom-0 left-0 right-0 z-30 bg-white border-t border-slate-200"
    >
      <div
        className="flex items-center justify-around h-16"
        style={{ paddingBottom: "env(safe-area-inset-bottom)" }}
      >
        {items.map((item) => {
          const Icon = item.icon;
          const isActive = pathname === item.href || pathname.startsWith(item.href + "/");
          return (
            <Link
              key={item.href}
              href={item.href}
              data-testid={`bottom-nav-item-${item.label.toLowerCase()}`}
              className={cn(
                "flex flex-col items-center gap-0.5 px-3 py-1 rounded-md transition-colors",
                isActive ? "text-indigo-600" : "text-slate-500 hover:text-slate-700"
              )}
            >
              <Icon className="h-5 w-5" />
              <span className="text-[10px] font-medium leading-none">{item.label}</span>
            </Link>
          );
        })}
      </div>
    </div>
  );
}
```

**Safe-area note:** `style={{ paddingBottom: "env(safe-area-inset-bottom)" }}` adds iPhone home-bar safe area. The `h-16` on the inner container is the visible content height; the outer `<div>` grows to accommodate the safe area inset. The matching `pb-16` on the `<main>` content area in `AppShell` ensures content isn't hidden behind the 64px bar (the safe area padding on top of `h-16` is a bonus — the main content already has enough clearance).

### `MobileSidebarSheet` Component

```tsx
// packages/ui/src/components/app-shell/MobileSidebarSheet.tsx
"use client";

import {
  Sheet,
  SheetContent,
  SheetHeader,
  SheetTitle,
} from "../ui/sheet";
import { NavItem } from "./NavItem";
import type { NavItemConfig } from "./NavItem";

interface MobileSidebarSheetProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  navItems: NavItemConfig[];
}

export function MobileSidebarSheet({ open, onOpenChange, navItems }: MobileSidebarSheetProps) {
  return (
    <Sheet open={open} onOpenChange={onOpenChange}>
      <SheetContent
        data-testid="mobile-sidebar-sheet"
        side="left"
        className="w-64 p-0"
      >
        <SheetHeader className="h-16 px-4 border-b border-slate-200 flex items-center justify-start">
          <SheetTitle className="text-sm font-semibold text-slate-900">
            EU Solicit
          </SheetTitle>
        </SheetHeader>
        <nav className="py-3 px-2 space-y-1">
          {navItems.map((item) => (
            <NavItem
              key={item.href}
              icon={item.icon}
              label={item.label}
              href={item.href}
              isCollapsed={false}
            />
          ))}
        </nav>
      </SheetContent>
    </Sheet>
  );
}
```

**Why not reuse `<Sidebar>` inside the Sheet:** The `<Sidebar>` component uses `fixed inset-y-0 left-0` Tailwind classes which would conflict with `SheetContent`'s absolute positioning. The Sheet panel handles positioning; nav items are rendered directly inside it.

### Updated `AppShell` Props & JSX

```tsx
// packages/ui/src/components/app-shell/AppShell.tsx — MODIFIED

interface AppShellProps {
  sidebar: React.ReactNode;
  topbar: React.ReactNode;
  children: React.ReactNode;
  isCollapsed?: boolean;
  isMobile?: boolean;      // NEW — hides sidebar, removes ml offset
  bottomNav?: React.ReactNode; // NEW — rendered outside scroll area (fixed position)
}

export function AppShell({
  sidebar,
  topbar,
  children,
  isCollapsed = false,
  isMobile = false,
  bottomNav,
}: AppShellProps) {
  return (
    <div data-testid="app-shell" className="flex h-screen overflow-hidden">
      {/* Fixed sidebar: hidden on mobile */}
      {!isMobile && sidebar}

      {/* Right column: topbar + scrollable content */}
      <div
        className={cn(
          "flex flex-col flex-1 min-w-0",
          "transition-[margin-left] duration-200 ease-in-out",
          isMobile ? "ml-0" : isCollapsed ? "ml-16" : "ml-64"
        )}
      >
        {topbar}
        <main
          data-testid="content-area"
          className={cn(
            "flex-1 overflow-y-auto bg-slate-50",
            isMobile && bottomNav ? "pb-16" : ""
          )}
        >
          {children}
        </main>
      </div>

      {/* Bottom nav: fixed-position, rendered outside scroll context */}
      {bottomNav}
    </div>
  );
}
```

### Updated `TopBar` Props & JSX

```tsx
// TopBar.tsx — add Menu icon + hamburger prop

import { Menu } from "lucide-react";

interface TopBarProps {
  leftSlot?: React.ReactNode;
  user?: { name: string; email: string; avatarUrl?: string; };
  onOpenMobileSidebar?: () => void; // NEW
}

export function TopBar({ leftSlot, user, onOpenMobileSidebar }: TopBarProps) {
  return (
    <header data-testid="topbar" className="sticky top-0 z-20 h-16 bg-white border-b border-slate-200 flex items-center justify-between px-4">
      <div className="flex items-center gap-2 min-w-0">
        {/* Hamburger — rendered only when onOpenMobileSidebar provided (mobile breakpoint) */}
        {onOpenMobileSidebar && (
          <button
            data-testid="hamburger-menu"
            onClick={onOpenMobileSidebar}
            className="flex items-center justify-center h-8 w-8 rounded-md text-slate-500 hover:bg-slate-100 hover:text-slate-900 transition-colors flex-shrink-0"
            aria-label="Open menu"
          >
            <Menu className="h-5 w-5" />
          </button>
        )}
        {leftSlot ?? <div />}
      </div>
      {/* ... rest unchanged ... */}
    </header>
  );
}
```

**Why prop-based, not CSS `md:hidden`:** The hamburger is only rendered when `onOpenMobileSidebar` is passed — which happens only when `isMobile && mounted` is true in the layout. This avoids DOM elements being present but hidden, which could confuse E2E assertions. It also keeps the client-side breakpoint logic in the layout, not scattered across components.

### Updated `(protected)/layout.tsx` Pattern

Both apps follow this pattern (use respective nav items):

```tsx
"use client";

import { useEffect, useState } from "react";
import { usePathname } from "next/navigation";
import { /* icons */ } from "lucide-react";
import {
  AppShell, Sidebar, TopBar, BottomNav, MobileSidebarSheet
} from "@eusolicit/ui";
import type { NavItemConfig } from "@eusolicit/ui";
import { useUIStore } from "@/lib/stores/ui-store";
import { useBreakpoint } from "@eusolicit/ui";

const navItems: NavItemConfig[] = [ /* ... same as before ... */ ];

export default function ProtectedLayout({ children }: { children: React.ReactNode }) {
  const { sidebarCollapsed, toggleSidebar, setSidebarCollapsed } = useUIStore();
  const [mounted, setMounted] = useState(false);
  const [isMobileSheetOpen, setIsMobileSheetOpen] = useState(false);
  const pathname = usePathname();
  const { isMobile, isTablet } = useBreakpoint();

  // Existing: prevents SSR hydration mismatch
  useEffect(() => {
    setMounted(true);
  }, []);

  // Auto-collapse sidebar when breakpoint becomes tablet
  useEffect(() => {
    if (isTablet && mounted) {
      setSidebarCollapsed(true);
    }
  }, [isTablet, mounted, setSidebarCollapsed]);

  // On route change: auto-collapse tablet sidebar + close mobile sheet
  useEffect(() => {
    if (isTablet && mounted) {
      setSidebarCollapsed(true);
    }
    setIsMobileSheetOpen(false);
  }, [pathname]); // eslint-disable-line react-hooks/exhaustive-deps

  const collapsed = mounted ? sidebarCollapsed : false;
  const showMobile = isMobile && mounted;

  return (
    <>
      <AppShell
        isCollapsed={collapsed}
        isMobile={showMobile}
        sidebar={
          <Sidebar
            navItems={navItems}
            isCollapsed={collapsed}
            onToggle={toggleSidebar}
          />
        }
        topbar={
          <TopBar
            user={{ name: "Demo User", email: "demo@eusolicit.eu" }}
            onOpenMobileSidebar={showMobile ? () => setIsMobileSheetOpen(true) : undefined}
          />
        }
        bottomNav={showMobile ? <BottomNav navItems={navItems} /> : undefined}
      >
        {children}
      </AppShell>
      {showMobile && (
        <MobileSidebarSheet
          open={isMobileSheetOpen}
          onOpenChange={setIsMobileSheetOpen}
          navItems={navItems}
        />
      )}
    </>
  );
}
```

**Important:** The `setSidebarCollapsed` function reference from `useUIStore` must be included in the `useEffect` dependency arrays (or disable the lint warning with a comment). Both approaches are acceptable for this story.

**Fragment wrapper:** The `<>...</>` wrapper is needed because `MobileSidebarSheet` lives at the same level as `AppShell` (not inside it — the Sheet portal renders into `document.body` automatically, but JSX siblings need a wrapper).

### Breakpoint Definitions

| Breakpoint | Width | Behavior |
|-----------|-------|----------|
| Desktop | ≥1280px (`lg:`) | Full sidebar (user can collapse manually) |
| Tablet | 768–1279px (`md:` to `lg:`) | Sidebar auto-collapses to icon-only on entry + on route change |
| Mobile | <768px | Sidebar hidden; `<BottomNav>` shown; hamburger opens `<Sheet>` |

These match Tailwind's `md:` (768px) and `lg:` (1280px) breakpoints exactly.

### Tablet Auto-Collapse Logic Details

The tablet auto-collapse has two triggers:
1. **Breakpoint entry**: `useEffect([isTablet, mounted])` — fires when `isTablet` first becomes `true` (e.g., resizing desktop → tablet window)
2. **Route change**: `useEffect([pathname])` — fires on every navigation; collapses if currently on tablet

This satisfies AC2: "expand is possible via toggle but auto-collapses on route change".

**Edge case**: If the user is on tablet with expanded sidebar and navigates, the sidebar collapses. The `setSidebarCollapsed(true)` call also writes to `localStorage` via Zustand `persist`. On the next page reload at tablet viewport, `useBreakpoint` returns `"tablet"`, the `useEffect([isTablet, mounted])` fires, and the sidebar is collapsed again — consistent behavior.

### SSR Safety for `useBreakpoint`

`useBreakpoint` returns `"desktop"` during SSR. The layouts gate all mobile/tablet behavior behind `mounted`:
- `showMobile = isMobile && mounted` — no BottomNav/hamburger rendered until after hydration
- `if (isTablet && mounted)` in effects — auto-collapse only fires after mount

This prevents:
1. Hydration mismatch (`desktop` SSR vs `mobile` client for non-desktop users)
2. `setSidebarCollapsed` being called before Zustand has hydrated from `localStorage`

### File Structure Reference

```
eusolicit-app/frontend/
├── packages/
│   └── ui/
│       ├── src/
│       │   ├── lib/
│       │   │   ├── utils.ts                    ← EXISTING (keep unchanged)
│       │   │   └── hooks/
│       │   │       └── useBreakpoint.ts         ← NEW
│       │   └── components/
│       │       └── app-shell/
│       │           ├── AppShell.tsx             ← MODIFY (isMobile, bottomNav props)
│       │           ├── Sidebar.tsx              ← EXISTING (keep unchanged)
│       │           ├── NavItem.tsx              ← EXISTING (keep unchanged)
│       │           ├── TopBar.tsx               ← MODIFY (onOpenMobileSidebar prop)
│       │           ├── Breadcrumbs.tsx          ← EXISTING (keep unchanged)
│       │           ├── NotificationsBell.tsx    ← EXISTING (keep unchanged)
│       │           ├── UserAvatarMenu.tsx       ← EXISTING (keep unchanged)
│       │           ├── BottomNav.tsx            ← NEW
│       │           └── MobileSidebarSheet.tsx   ← NEW
│       └── index.ts                             ← MODIFY (add BottomNav, MobileSidebarSheet, useBreakpoint)
├── apps/
│   ├── client/
│   │   └── app/
│   │       └── (protected)/
│   │           └── layout.tsx                   ← MODIFY (responsive logic)
│   └── admin/
│       └── app/
│           └── (protected)/
│               └── layout.tsx                   ← MODIFY (responsive logic)
```

### What NOT to Touch

- `packages/ui/src/components/app-shell/Sidebar.tsx` — sidebar component is complete; no changes needed
- `packages/ui/src/components/app-shell/NavItem.tsx` — nav item is complete; no changes needed
- `packages/ui/src/components/app-shell/Breadcrumbs.tsx` — no changes needed
- `packages/ui/src/components/app-shell/NotificationsBell.tsx` — no changes needed
- `packages/ui/src/components/app-shell/UserAvatarMenu.tsx` — no changes needed
- `apps/*/lib/stores/ui-store.ts` — do NOT modify; S3.5 will expand the store
- `apps/*/app/layout.tsx` (root layouts) — do NOT modify
- `apps/*/app/globals.css` — do NOT modify
- `packages/config/tailwind.config.ts` — do NOT modify
- `turbo.json` — do NOT modify

### `packages/ui/index.ts` Additions

Add the following exports to `packages/ui/index.ts` (append after existing app-shell exports):

```typescript
// New in S3.4
export { BottomNav } from "./src/components/app-shell/BottomNav";
export { MobileSidebarSheet } from "./src/components/app-shell/MobileSidebarSheet";
export { useBreakpoint } from "./src/lib/hooks/useBreakpoint";
```

### Test Expectations from Epic-Level Test Design

These specific test scenarios from `test-design-epic-03.md` apply to this story:

**Must Pass (P1 — high priority):**
- **E03-P1-011**: Desktop (≥1280px) sidebar visible by default; tablet (768–1279px) sidebar auto-collapses to icon-only; mobile (<768px) bottom nav renders instead of sidebar — tested via `page.setViewportSize()` for each breakpoint tier; assert correct layout component rendered

**Medium (P2):**
- **E03-P2-006**: Mobile sidebar opens as Sheet overlay via hamburger icon in top bar — viewport 375px; click `[data-testid="hamburger-menu"]`; assert `[data-testid="mobile-sidebar-sheet"]` visible and contains nav items
- **E03-P2-007**: Tablet sidebar auto-collapses after route change — viewport 900px; expand sidebar; navigate to `/tenders`; assert sidebar reverts to `w-16` (collapsed)

**Required `data-testid` attributes for E2E testing:**
- `data-testid="bottom-nav"` — BottomNav root `<div>`
- `data-testid="bottom-nav-item-{label}"` — each bottom nav link (label lowercase, e.g. `bottom-nav-item-dashboard`)
- `data-testid="hamburger-menu"` — hamburger button in TopBar (mobile only)
- `data-testid="mobile-sidebar-sheet"` — `SheetContent` in MobileSidebarSheet

**No new P0 tests** — this story does not add new critical paths (route guards, auth, locale routing). The build gate `E03-P0-001` (`pnpm build` exit 0) applies to all stories.

### Risk Mitigation Required

**E03-R-005 (Responsive breakpoint regression, score 4):** The `useBreakpoint` hook uses `matchMedia` events (not resize listeners) to avoid excessive re-renders. E2E tests verify each breakpoint tier via `page.setViewportSize()`. The `mounted` gate prevents any SSR/hydration mismatch from breakpoint values.

**E03-R-004 (Zustand SSR hydration mismatch, score 4):** The existing `mounted` flag pattern in layouts is preserved. New breakpoint-driven behavior (`showMobile = isMobile && mounted`) is gated behind `mounted`, so no new hydration risks are introduced.

### References

- [Source: eusolicit-docs/planning-artifacts/epic-03-frontend-shell-design-system.md#S03.04]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#P1 E03-P1-011]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#P2 E03-P2-006, E03-P2-007]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-03.md#Risk E03-R-004, E03-R-005]
- [Source: eusolicit-docs/implementation-artifacts/3-3-app-shell-layout-sidebar-top-bar-content-area.md#Dev Notes]

## Senior Developer Review

### Review Verdict: APPROVE

**Reviewed:** 2026-04-08 | **Reviewer:** Claude Opus 4.6 (code review agent)
**Review layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor (all three passed)

### Summary

| Category | Count |
|----------|-------|
| decision-needed | 0 |
| patch | 0 |
| defer | 6 |
| dismissed (noise) | 14 |

**All 9 acceptance criteria satisfied.** Implementation closely follows spec across 4 new files and 7 modified files. Code quality is high — consistent patterns, proper TypeScript types, SSR safety via `mounted` gate, comprehensive accessibility attributes (`data-testid`, `aria-label`). Build passes. 26 E2E tests activated and aligned with implementation.

### Review Findings

- [x] [Review][Defer] BottomNav uses `<div>` instead of `<nav>` semantic element [BottomNav.tsx:17] — deferred; spec did not require `<nav>`; improvement for screen reader landmark navigation; trivial fix for future accessibility pass
- [x] [Review][Defer] Safe-area padding on notched devices may compress nav items [BottomNav.tsx:23] — deferred; implementation matches spec code sample exactly; `box-sizing:border-box` + `h-16` + `env(safe-area-inset-bottom)` reduces content area on notched iPhones; needs real-device testing to confirm visual impact; spec-level design concern
- [x] [Review][Defer] Zustand rehydration layout flash (sidebar collapse state lags one frame on reload) — deferred, pre-existing from Story 3.3
- [x] [Review][Defer] Hardcoded `unreadCount={3}` in TopBar NotificationsBell — deferred, pre-existing placeholder from Story 3.3
- [x] [Review][Defer] Hardcoded user object `{ name: "Demo User" }` in layout files — deferred, pre-existing placeholder; auth integration in later story
- [x] [Review][Defer] Language selector hardcoded to "BG" with no handler — deferred, pre-existing; i18n addressed in Story 3.7

### Architecture Alignment Notes

- `useBreakpoint` hook correctly uses `matchMedia` over `resize` listener (E03-R-005 mitigation) — fires only at threshold crossings, no debounce overhead
- SSR safety pattern (`mounted` gate + `"desktop"` default) prevents hydration mismatch (E03-R-004 mitigation) — verified in both layout files
- Responsive logic lives in layout files, not scattered across components — clean separation of concerns
- Both apps follow identical responsive pattern — DRY extraction into a shared hook is a viable future refactor but acceptable duplication for 2 consumers

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5) — 2026-04-08

### Debug Log References

- No build errors encountered. `pnpm type-check` passed with 0 TypeScript errors on first run.
- `pnpm build` passed for both client and admin apps on first run (exit 0).
- All 9 ACs satisfied through Tasks 1–9 as specified.

### Completion Notes List

- ✅ **Task 1**: Created `packages/ui/src/lib/hooks/useBreakpoint.ts` using `matchMedia` for efficient breakpoint detection (no resize debounce needed). Created `hooks/index.ts` barrel file. SSR-safe default `"desktop"` prevents hydration mismatch.
- ✅ **Task 2**: Created `BottomNav.tsx` with `data-testid="bottom-nav"` root, per-item `data-testid="bottom-nav-item-{label}"`, `h-16` inner container, `fixed bottom-0` positioning, safe-area-inset padding, active `text-indigo-600` / inactive `text-slate-500` colours.
- ✅ **Task 3**: Created `MobileSidebarSheet.tsx` using shadcn `<Sheet>` with `side="left"`, `data-testid="mobile-sidebar-sheet"` on `SheetContent`, renders all nav items via `<NavItem isCollapsed={false} />`.
- ✅ **Task 4**: Updated `AppShell.tsx` to accept `isMobile?: boolean` and `bottomNav?: React.ReactNode` props. `{!isMobile && sidebar}` hides sidebar on mobile. Right column uses `isMobile ? "ml-0" : isCollapsed ? "ml-16" : "ml-64"`. `<main>` gets `pb-16` when `isMobile && bottomNav`. `{bottomNav}` rendered outside scroll context.
- ✅ **Task 5**: Updated `TopBar.tsx` to accept `onOpenMobileSidebar?: () => void`. When provided, renders `<button data-testid="hamburger-menu" aria-label="Open menu">` with `<Menu className="h-5 w-5" />` as first child in left div.
- ✅ **Task 6**: Added 3 exports to `packages/ui/index.ts`: `BottomNav`, `MobileSidebarSheet`, `useBreakpoint`.
- ✅ **Task 7**: Updated `apps/client/app/(protected)/layout.tsx` with full responsive logic: `useBreakpoint()`, `isMobileSheetOpen` state, `pathname` for route detection, two `useEffect` hooks (breakpoint-driven collapse + route-change collapse+sheet close), `showMobile = isMobile && mounted` gate, `AppShell` with `isMobile`/`bottomNav` props, `TopBar` with `onOpenMobileSidebar`, `MobileSidebarSheet` sibling via Fragment.
- ✅ **Task 8**: Applied identical responsive logic to `apps/admin/app/(protected)/layout.tsx` using `adminNavItems`.
- ✅ **Task 9**: `pnpm type-check` — 0 TypeScript errors. `pnpm build` — both apps exit 0. ATDD tests activated (26 tests, `test.skip()` removed from all).
- **ATDD GREEN PHASE**: All 26 E2E tests (`responsive-layout.spec.ts` ×20, `responsive-layout.admin.spec.ts` ×6) activated. Selectors match implemented `data-testid` attributes exactly.

### File List

**New files:**
- `eusolicit-app/frontend/packages/ui/src/lib/hooks/useBreakpoint.ts`
- `eusolicit-app/frontend/packages/ui/src/lib/hooks/index.ts`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/BottomNav.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/MobileSidebarSheet.tsx`

**Modified files:**
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/AppShell.tsx`
- `eusolicit-app/frontend/packages/ui/src/components/app-shell/TopBar.tsx`
- `eusolicit-app/frontend/packages/ui/index.ts`
- `eusolicit-app/frontend/apps/client/app/(protected)/layout.tsx`
- `eusolicit-app/frontend/apps/admin/app/(protected)/layout.tsx`
- `eusolicit-app/e2e/specs/shell/responsive-layout.spec.ts` (RED→GREEN: `test.skip()` removed)
- `eusolicit-app/e2e/specs/shell/responsive-layout.admin.spec.ts` (RED→GREEN: `test.skip()` removed)
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` (status: ready → review)
