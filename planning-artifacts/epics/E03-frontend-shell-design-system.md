# E03: Frontend Shell & Design System

**Sprint**: 1--2 (Weeks 1--4) | **Points**: 37 | **Dependencies**: None (scaffolds independently; integrates with E02 auth APIs when ready) | **Milestone**: Demo

## Goal

Stand up the complete frontend scaffolding for both EU Solicit applications (client and admin) so that every subsequent feature epic drops UI into a fully functioning shell with consistent design language, internationalisation, state management, data-fetching plumbing, form handling, authentication flows, and company onboarding. By the end of Sprint 2 a reviewer can open both apps in the browser, switch between Bulgarian and English, log in (stubbed or real), walk through the company setup wizard, see skeleton loaders, error states, empty states, and toast notifications -- all rendered in the monochromatic slate + indigo design system.

## Acceptance Criteria

- [ ] `frontend/` monorepo contains two Next.js 14 App Router applications (`client` and `admin`) that build and start without errors
- [ ] Tailwind CSS config exports a shared design-token preset (slate neutrals, indigo-600 primary, semantic status colours, spacing scale, typography scale, shadow scale)
- [ ] shadcn/ui is installed in both apps; Button, Input, Select, Dialog, DropdownMenu, Tooltip, Badge, Card, Tabs, Table, Skeleton components are themed and documented in a local Storybook-like page (`/dev/components`)
- [ ] App shell renders in both apps: collapsible sidebar (full + icon-only), top bar with user avatar menu, language selector, and notifications bell icon; main content area with breadcrumbs
- [ ] Responsive behaviour: sidebar collapses to icon-only on tablet; bottom navigation replaces sidebar on mobile
- [ ] Zustand stores for `authStore` (user, token, logout) and `uiStore` (sidebarCollapsed, theme, locale) are initialised and persisted where appropriate
- [ ] TanStack Query `QueryClientProvider` wraps the app; a shared `apiClient` (Axios or fetch wrapper) attaches JWT from authStore, refreshes on 401, and exposes an SSE subscription helper
- [ ] React Hook Form + Zod validation is wired with a reusable `FormField` component pattern
- [ ] `next-intl` serves BG and EN locales; language selector toggles locale and persists choice to `localStorage`; all shell chrome strings are translated
- [ ] Auth pages exist and function (with stubbed API or real E02 endpoints): Login, Register, Forgot Password, OAuth callback
- [ ] Company profile setup wizard (multi-step form) collects company info, industry/CPV sectors, regions, and team invites
- [ ] Skeleton loaders, error boundaries (global + per-section), empty state components, and toast system are in place and demonstrable
- [ ] Client-side route guards redirect unauthenticated users to `/login` and authenticated users away from auth pages
- [ ] (Stretch) Dark mode toggle with system-preference detection works across both apps

## Stories

### S03.01: Next.js 14 Monorepo Scaffold

**Points**: 3 | **Type**: frontend
**Description**: Create the `frontend/` directory with two Next.js 14 App Router applications (`client` and `admin`). Configure a shared Turborepo (or npm workspaces) setup so both apps share a `packages/ui` component library and a `packages/config` package for common Tailwind/TypeScript/ESLint config. Each app should have the standard App Router folder structure (`app/`, `lib/`, `components/`) and pass `next build` cleanly.
**Acceptance Criteria**:
- [ ] `frontend/apps/client` and `frontend/apps/admin` each contain a Next.js 14 App Router project
- [ ] `frontend/packages/ui` exists as a shared component library package
- [ ] `frontend/packages/config` exports shared `tailwind.config.ts`, `tsconfig.json`, and `.eslintrc.js`
- [ ] `pnpm dev --filter client` and `pnpm dev --filter admin` both start on different ports (3000, 3001)
- [ ] `pnpm build` at root succeeds for both apps
- [ ] TypeScript strict mode enabled; ESLint + Prettier configured and passing
**Implementation Notes**: Use `create-turbo` or manual Turborepo init. Pin Next.js to 14.x. Use pnpm workspaces. App Router (`app/` directory) must be enabled in both apps. Add `packages/ui/index.ts` barrel export.

---

### S03.02: Tailwind Design Token Preset & shadcn/ui Theming

**Points**: 3 | **Type**: frontend
**Description**: Define the EU Solicit visual design language as a Tailwind CSS preset in `packages/config`. Install shadcn/ui in both apps, configure its theme to use the custom tokens, and generate the initial component set. Create a `/dev/components` route in the client app that renders every installed component for visual QA.
**Acceptance Criteria**:
- [ ] Tailwind preset defines: slate-based neutral scale (50--950), indigo-600 primary + 50--950 shades, semantic colours (success/green, warning/amber, error/red, info/blue), 4px spacing base, Inter + JetBrains Mono font stacks, shadow-sm through shadow-2xl custom values
- [ ] CSS variables for `--primary`, `--secondary`, `--destructive`, `--muted`, `--accent`, `--background`, `--foreground`, `--border`, `--ring` are set in `globals.css` for light (and optionally dark) mode
- [ ] shadcn/ui components installed: Button, Input, Textarea, Select, Checkbox, RadioGroup, Switch, Dialog, Sheet, DropdownMenu, Tooltip, Badge, Card, Tabs, Table, Skeleton, Avatar, Separator, ScrollArea
- [ ] `/dev/components` page renders all components with labels and variant previews
- [ ] Both apps import and render a shared `<Button>` from `packages/ui` without errors
**Implementation Notes**: Run `npx shadcn-ui@latest init` in each app pointing at the shared CSS variables. Copy generated components into `packages/ui/components/ui/`. Use `tailwind.config.ts` `presets` array to inherit the shared preset. Inter from `next/font/google`.

---

### S03.03: App Shell Layout -- Sidebar, Top Bar, Content Area

**Points**: 5 | **Type**: frontend
**Description**: Build the authenticated app shell used by both client and admin apps. The shell consists of a collapsible left sidebar, a top bar, and a scrollable main content area. The sidebar supports full mode (icon + label, ~256px) and collapsed mode (icon-only, ~64px) with a toggle button. The top bar contains breadcrumbs on the left and a cluster of actions on the right: notification bell, language selector, and user avatar dropdown menu.
**Acceptance Criteria**:
- [ ] `<AppShell>` component in `packages/ui` accepts `sidebar`, `topbar`, and `children` slots
- [ ] Sidebar renders navigation items as `<NavItem icon={...} label={...} href={...} />` with active-state highlight (indigo-50 bg, indigo-600 text)
- [ ] Sidebar toggle animates width transition (200ms ease) and hides labels in collapsed mode
- [ ] Top bar is sticky, 64px height, contains: left breadcrumbs, right action cluster
- [ ] User avatar dropdown shows: user name, email, "Profile", "Settings", divider, "Sign out"
- [ ] Notifications bell shows unread count badge (static placeholder)
- [ ] Main content area scrolls independently; sidebar and top bar remain fixed
- [ ] Client app shell sidebar items: Dashboard, Tenders, Offers, Documents, Team, Settings
- [ ] Admin app shell sidebar items: Dashboard, Companies, Tenders, Users, Reports, Settings
**Implementation Notes**: Use shadcn `Sheet` for mobile sidebar overlay. Store sidebar collapsed state in `uiStore` (Zustand) and persist to `localStorage`. Use `next/link` for nav items with `usePathname()` for active detection. Sidebar and top bar are server-component-compatible wrappers around client interactive pieces.

---

### S03.04: Responsive Layout Strategy

**Points**: 3 | **Type**: frontend
**Description**: Implement the responsive breakpoint behaviour for the app shell. Desktop (>=1280px) shows the full sidebar. Tablet (768--1279px) auto-collapses the sidebar to icon-only mode. Mobile (<768px) hides the sidebar entirely and renders a fixed bottom navigation bar with the five primary nav items.
**Acceptance Criteria**:
- [ ] Desktop: full sidebar visible by default; user can manually collapse/expand
- [ ] Tablet: sidebar renders in icon-only (collapsed) mode; expand is possible via toggle but auto-collapses on route change
- [ ] Mobile: sidebar hidden; bottom nav bar (fixed, 64px, safe-area-aware) shows 5 icons + labels
- [ ] Bottom nav highlights active route with indigo-600 icon colour
- [ ] Layout transitions are smooth, no content reflow or jump on breakpoint crossing
- [ ] Mobile sidebar can still be opened as an overlay sheet via hamburger icon in the top bar
**Implementation Notes**: Use Tailwind `md:` and `lg:` prefixes. A `useMediaQuery` hook (or `useWindowSize`) drives the `uiStore` sidebar state at mount and on resize. Bottom nav is a separate `<BottomNav>` component rendered conditionally. Test with Chrome DevTools device toolbar.

---

### S03.05: Zustand Stores & TanStack Query Setup

**Points**: 3 | **Type**: frontend
**Description**: Wire up client-side state management (Zustand) and server-state data-fetching (TanStack Query). Create the foundational stores and the API client layer that all subsequent features will use.
**Acceptance Criteria**:
- [ ] `authStore` (Zustand): `user`, `token`, `refreshToken`, `isAuthenticated`, `login()`, `logout()`, `setUser()`, `setTokens()`; persisted to `localStorage` via `zustand/middleware`
- [ ] `uiStore` (Zustand): `sidebarCollapsed`, `theme` ("light" | "dark" | "system"), `locale` ("bg" | "en"), `toasts[]`, `addToast()`, `removeToast()`; `sidebarCollapsed` persisted
- [ ] TanStack Query `QueryClient` configured with sensible defaults: `staleTime: 30s`, `gcTime: 5min`, `retry: 1`, `refetchOnWindowFocus: true`
- [ ] `apiClient` (Axios instance or fetch wrapper) in `packages/ui/lib/api-client.ts` attaches `Authorization: Bearer <token>` header from `authStore`, intercepts 401 responses to attempt token refresh, and on refresh failure calls `authStore.logout()`
- [ ] `useSSE(url)` hook subscribes to a Server-Sent Events endpoint and returns a reactive stream value; cleans up on unmount
- [ ] A smoke-test query (`useHealthCheck`) hits `GET /health` and displays result on `/dev/api-test` page
**Implementation Notes**: Use `zustand` with `persist` and `devtools` middleware. TanStack Query v5. Axios interceptors for JWT attach and 401 refresh. SSE via `EventSource` wrapped in `useEffect`.

---

### S03.06: React Hook Form + Zod Validation Patterns

**Points**: 2 | **Type**: frontend
**Description**: Establish the reusable form architecture that all application forms will follow. Create a `<FormField>` wrapper that connects React Hook Form controllers to shadcn/ui inputs, displays validation errors inline, and supports Zod schema validation.
**Acceptance Criteria**:
- [ ] `useZodForm(schema)` hook returns a configured `useForm` instance with `zodResolver`
- [ ] `<FormField name={...} label={...} description={...}>` component renders label, input slot (children or render prop), error message, and optional description text
- [ ] `<FormField>` variants exist for: text input, textarea, select, checkbox, radio group, date picker, file upload
- [ ] Error messages appear below the field in red-500 text with a smooth height animation
- [ ] A demo form on `/dev/form-test` validates a sample schema (text, email, select, checkbox) and shows toast on submit
- [ ] All form labels and error messages use `next-intl` translation keys
**Implementation Notes**: Use `@hookform/resolvers/zod`. The `<FormField>` wraps shadcn `FormItem`/`FormLabel`/`FormControl`/`FormMessage`. Export from `packages/ui`. Zod schemas live alongside the forms that use them in `lib/schemas/`.

---

### S03.07: i18n Setup with next-intl (BG + EN)

**Points**: 3 | **Type**: frontend
**Description**: Configure `next-intl` for both applications with Bulgarian and English locale support. All static UI strings (navigation labels, button text, form labels, error messages, page titles) must be translatable. The language selector in the top bar toggles locale and persists the choice.
**Acceptance Criteria**:
- [ ] `next-intl` configured with `middleware.ts` locale detection (Accept-Language header, cookie, default "bg")
- [ ] Message files: `messages/bg.json` and `messages/en.json` with nested namespace structure (`common`, `nav`, `auth`, `forms`, `errors`, `wizard`)
- [ ] All shell chrome strings (sidebar labels, top bar, breadcrumbs, user menu items) use `useTranslations()` / `getTranslations()`
- [ ] Language selector in top bar shows flag icon + language code (BG / EN); click toggles locale
- [ ] Locale preference persisted to `localStorage` and used on next visit
- [ ] URL structure uses locale prefix: `/bg/dashboard`, `/en/dashboard` (default locale prefix optional)
- [ ] Date and number formatting helpers (`formatDate`, `formatCurrency`) respect active locale
**Implementation Notes**: Use `next-intl` v3 with App Router integration (`next-intl/plugin`). Locale is a route segment: `app/[locale]/`. Middleware rewrites to default locale. Bulgarian translations can be placeholder text initially but must cover all keys. Store locale in `uiStore` and sync with `next-intl`.

---

### S03.08: Authentication Pages

**Points**: 3 | **Type**: frontend
**Description**: Build the four authentication pages: Login, Register, Forgot Password, and OAuth Callback. These pages use a centred card layout (no sidebar) with the EU Solicit logo. Forms validate client-side with Zod and submit to E02 auth endpoints (stubbed with mock responses until E02 is ready).
**Acceptance Criteria**:
- [ ] `/login` page: email + password fields, "Remember me" checkbox, "Forgot password?" link, "Sign in" button, Google OAuth button, link to register
- [ ] `/register` page: company name, EIK (Bulgarian company ID), first name, last name, email, password, confirm password, terms checkbox; submits to `POST /auth/register`
- [ ] `/forgot-password` page: email field, submit button; success state shows "Check your email" message
- [ ] `/auth/callback` page: handles OAuth redirect, extracts code/state params, exchanges for tokens, redirects to dashboard
- [ ] All forms show inline Zod validation errors on blur and on submit
- [ ] Loading states: buttons show spinner on submit, inputs disabled during submission
- [ ] Auth pages redirect to `/dashboard` if user is already authenticated
- [ ] All text translated (BG + EN)
- [ ] Auth layout: centred vertically and horizontally, max-w-md card, EU Solicit logo above card
**Implementation Notes**: Create `app/[locale]/(auth)/layout.tsx` with the centred card layout. Use `useZodForm` from S03.06. Google OAuth button opens a popup or redirects to Google consent URL. Stub API calls with `msw` or simple `setTimeout` mock until E02 is deployed.

---

### S03.09: Company Profile Setup Wizard

**Points**: 5 | **Type**: frontend
**Description**: After first registration, users land on a multi-step company setup wizard. The wizard collects: (1) company details (name, EIK, address, phone, website, logo upload), (2) industry and CPV sectors (searchable multi-select from CPV taxonomy), (3) regions of interest (map or list of Bulgarian planning regions + EU countries), (4) team invites (add email addresses for colleagues). A progress stepper shows the current step. Users can navigate back and forth; data persists between steps.
**Acceptance Criteria**:
- [ ] Wizard renders four steps with a horizontal stepper component showing step labels, icons, and completion state
- [ ] Step 1 -- Company Info: name (prefilled from registration), EIK (prefilled), address, phone, website, logo upload with preview
- [ ] Step 2 -- Industry & CPV: searchable input that filters a CPV code tree; selected items shown as dismissible badges; minimum 1 selection required
- [ ] Step 3 -- Regions: checkable list of Bulgarian planning regions (6) and EU member states (27); select-all toggles; minimum 1 required
- [ ] Step 4 -- Team Invites: email input with "Add" button; list of added emails with remove; optional step (can be skipped)
- [ ] "Back" and "Next" buttons; "Next" validates current step before advancing; "Complete Setup" on final step
- [ ] Wizard state persisted in Zustand (survives page refresh mid-wizard)
- [ ] On completion, POST to `/companies/:id/profile` and redirect to `/dashboard`
- [ ] Fully translated (BG + EN)
**Implementation Notes**: Use React Hook Form in `useForm` mode with per-step schemas. CPV codes loaded from a static JSON file (or API when available). Stepper component built on top of shadcn primitives. Logo upload uses a file input with `URL.createObjectURL` preview. Persist wizard state in a `wizardStore` (Zustand, non-persisted to localStorage is fine since it's session-scoped -- actually persist it so refresh doesn't lose data).

---

### S03.10: Loading States, Error Boundaries & Empty States

**Points**: 3 | **Type**: frontend
**Description**: Create the reusable UX patterns for loading, error, and empty data states that every feature page will use. Skeleton loaders should match the actual content layout to prevent reflow. Error boundaries should catch rendering errors gracefully. Empty state components should be friendly and actionable.
**Acceptance Criteria**:
- [ ] `<Skeleton>` variants: `SkeletonCard`, `SkeletonTable` (configurable rows/columns), `SkeletonList`, `SkeletonText` (configurable lines), `SkeletonAvatar`
- [ ] Skeletons use `animate-pulse` with slate-200 background, matching dimensions of real content
- [ ] Global `error.tsx` at `app/[locale]/error.tsx` catches unhandled errors, shows friendly message + "Try again" button + optional error details in collapsible
- [ ] Per-section `<ErrorBoundary fallback={...}>` component wraps individual page sections; one section crashing does not take down the whole page
- [ ] `<EmptyState icon={...} title={...} description={...} action={...} />` component with illustration slot, used when lists/tables have no data
- [ ] Three pre-built empty state variants: "No results found" (with search suggestion), "Get started" (with create CTA), "No access" (with request-access CTA)
- [ ] All strings translated (BG + EN)
**Implementation Notes**: Skeleton components in `packages/ui`. Use React `ErrorBoundary` class component (or `react-error-boundary` library). Empty state illustrations can be simple SVG icons from Lucide at large size. Wire TanStack Query `isLoading` to skeleton display in a `<QueryGuard>` wrapper that handles loading/error/empty/success states.

---

### S03.11: Toast Notification System

**Points**: 2 | **Type**: frontend
**Description**: Implement a toast notification system for transient feedback messages (success, error, warning, info). Toasts appear in the bottom-right corner, stack vertically, auto-dismiss after a configurable duration, and can be manually dismissed.
**Acceptance Criteria**:
- [ ] `addToast({ type, title, description?, duration? })` function available from `uiStore`
- [ ] Toast types: `success` (green icon), `error` (red icon), `warning` (amber icon), `info` (blue icon)
- [ ] Toasts render in a fixed portal in the bottom-right corner, max 5 visible, stacked with 8px gap
- [ ] Default auto-dismiss: 5s for success/info, 8s for warning, 10s for error (no auto-dismiss if `duration: Infinity`)
- [ ] Each toast has a close button; hover pauses the auto-dismiss timer
- [ ] Entry animation: slide in from right + fade; exit animation: slide out to right + fade
- [ ] `useToast()` hook provides `toast.success(title)`, `toast.error(title)`, `toast.info(title)` shorthand methods
- [ ] Demo on `/dev/toasts` page with buttons to trigger each type
**Implementation Notes**: Build on top of shadcn `Toast` / `Sonner` integration (shadcn ships a `sonner` toast by default). Customise appearance to match the design system. Wire `uiStore.toasts[]` as the backing state or use Sonner's internal state. Export `useToast` from `packages/ui`.

---

### S03.12: Client-Side Route Guards & Auth Redirects

**Points**: 2 | **Type**: frontend
**Description**: Protect authenticated routes so unauthenticated users are redirected to `/login`, and already-authenticated users accessing auth pages are redirected to `/dashboard`. Implement as a Next.js middleware and/or a client-side layout wrapper.
**Acceptance Criteria**:
- [ ] Unauthenticated user visiting `/dashboard` (or any protected route) is redirected to `/login?redirect=/dashboard`
- [ ] After login, user is redirected to the original `redirect` URL (or `/dashboard` if none)
- [ ] Authenticated user visiting `/login` or `/register` is redirected to `/dashboard`
- [ ] Route guard checks `authStore.isAuthenticated` and `authStore.token` expiry
- [ ] During the auth check (hydration), a full-page loading spinner is shown (no flash of protected content)
- [ ] Guards work in both client and admin apps
- [ ] Next.js middleware (`middleware.ts`) handles server-side redirect for cookie-based sessions as a secondary check
**Implementation Notes**: Primary guard is a `<AuthGuard>` client component wrapping `app/[locale]/(protected)/layout.tsx`. It reads `authStore` after hydration. The middleware provides server-side protection by checking for a session cookie (set on login). Use `useRouter().replace()` for client redirects. Protected layout group: `(protected)` contains dashboard, tenders, etc. Auth layout group: `(auth)` contains login, register, forgot-password.
