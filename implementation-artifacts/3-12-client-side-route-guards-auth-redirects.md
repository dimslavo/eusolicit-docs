# Story 3.12 — Client-Side Route Guards & Auth Redirects

**Epic:** 3 — Frontend Shell & Design System
**Status:** ✅ Done
**Points:** 2 | **Type:** frontend

---

## User Story

**As a** frontend developer on the EU Solicit team
**I want** route guards that automatically redirect unauthenticated users to `/login` and
already-authenticated users away from auth pages, with a full-page loading spinner during
the Zustand hydration window so no protected content ever flashes
**So that** every protected route in both the client and admin apps is secured at the
presentation layer, the `redirect` query param is round-tripped correctly through the login
flow, and the server-side middleware provides a secondary cookie-based guard before React
even renders

---

## Acceptance Criteria

1. **AC1 — Unauthenticated redirect to login:** An unauthenticated user visiting `/dashboard`
   (or any route inside the `(protected)` layout group) is redirected to
   `/login?redirect=/dashboard`. The `redirect` query param encodes the originally requested
   path so the login page can send the user back after successful authentication.

2. **AC2 — Post-login redirect:** After a successful login, the auth pages read the `redirect`
   query param and redirect the user to the stored path (or `/dashboard` if the param is
   absent/invalid).

3. **AC3 — Authenticated redirect away from auth pages:** An authenticated user visiting
   `/login` or `/register` (any route inside the `(auth)` layout group) is redirected to
   `/dashboard`.

4. **AC4 — Token expiry check:** The route guard checks both `authStore.isAuthenticated` and
   whether the stored `token` is present (non-null). If `isAuthenticated` is `true` but
   `token` is `null` (corrupt state), the guard treats the user as unauthenticated and
   redirects to `/login`.

5. **AC5 — Hydration spinner (no flash of protected content):** During the SSR→client
   hydration window (before `useAuthStore.persist.hasHydrated()` resolves), the `<AuthGuard>`
   renders an opaque full-page loading spinner (`data-testid="auth-loading-spinner"`) so
   no protected page markup is visible to unauthenticated users before the redirect fires.

6. **AC6 — Guards work in both apps:** The `<AuthGuard>` component and the updated
   `middleware.ts` are applied in both `apps/client` and `apps/admin`.

7. **AC7 — Server-side middleware check:** The `middleware.ts` in each app is extended
   (beyond the existing next-intl handler) to inspect for a `eusolicit-session` cookie on
   protected paths. If the cookie is absent, the middleware returns a 307 redirect to
   `/login` **before** the Next.js page renders. The next-intl middleware still runs first
   for locale prefix rewriting.

8. **AC8 — Redirect loop protection:** The guard logic detects if the `redirect` param itself
   points to `/login` or another auth route and falls back to `/dashboard`, preventing
   circular redirects. The maximum observed redirect count for any entry path must be ≤ 2.

9. **AC9 — `AuthGuard` exported from `packages/ui`:** The `<AuthGuard>` component is
   implemented in `packages/ui/src/components/auth/AuthGuard.tsx`, exported from
   `packages/ui/index.ts` under a `// New in S3.12` block, and imported from
   `@eusolicit/ui` in both app layouts.

10. **AC10 — Build & type-check:** `pnpm build` and `pnpm type-check` pass with zero errors
    across all packages and apps. (Validated by CI.)

---

## Implementation Notes

### Architecture

This story adds a **two-layer route guard** to both apps:

**Layer 1 — Next.js middleware (server-side, secondary):**
The existing `middleware.ts` in each app only runs the next-intl locale handler. S3.12
chains an auth cookie check **before** the locale handler. Protected path patterns
(`/*/dashboard`, `/*/tenders`, etc.) are checked for the presence of a
`eusolicit-session` cookie (set by the login API in E02). If absent, a 307 redirect to
`/${locale}/login` is returned immediately — no React rendering occurs.

The existing `createMiddleware` from `next-intl` continues to handle locale prefix
rewriting. Use `chain()` or sequential composition to run auth check first, then i18n.

**Layer 2 — `<AuthGuard>` client component (primary):**
A `"use client"` component in `packages/ui/src/components/auth/AuthGuard.tsx` wraps
`app/[locale]/(protected)/layout.tsx` in both apps. It:

1. Reads `useAuthStore.persist.hasHydrated()` via a `useEffect` on mount — this resolves
   once Zustand has rehydrated from `localStorage`.
2. Until hydrated: renders a full-page opaque spinner (`data-testid="auth-loading-spinner"`)
   — slate-100 background, centred indigo-600 `Loader2` icon from Lucide with
   `animate-spin`.
3. After hydration: checks `isAuthenticated && token !== null`.
   - If not authenticated: calls `router.replace(`/${locale}/login?redirect=${pathname}`)`.
   - If authenticated: renders `{children}`.

For the `(auth)` layout group, a lighter `<AuthRedirect>` component (or inline logic in
`(auth)/layout.tsx`) does the reverse: if `isAuthenticated && token !== null` after
hydration, redirect to `/${locale}/dashboard`.

**Redirect loop guard:**
Before building the `redirect` param, check if `pathname` starts with the locale-prefixed
auth segment (e.g. `/bg/login`, `/bg/register`). If so, fall back to
`/${locale}/dashboard` as the target.

### Key Files to Create / Modify

| File | Action |
|------|--------|
| `packages/ui/src/components/auth/AuthGuard.tsx` | **Create** — primary client-side guard |
| `packages/ui/src/components/auth/AuthRedirect.tsx` | **Create** — redirect authenticated users away from auth pages |
| `packages/ui/src/components/auth/index.ts` | **Create** — barrel export |
| `packages/ui/index.ts` | **Update** — add `// New in S3.12` export block |
| `apps/client/app/[locale]/(protected)/layout.tsx` | **Update** — wrap with `<AuthGuard>` |
| `apps/client/app/[locale]/(auth)/layout.tsx` | **Update** — add `<AuthRedirect>` |
| `apps/client/middleware.ts` | **Update** — chain auth cookie check before next-intl |
| `apps/admin/app/[locale]/(protected)/layout.tsx` | **Update** — wrap with `<AuthGuard>` |
| `apps/admin/middleware.ts` | **Update** — chain auth cookie check before next-intl |

### `AuthGuard` Component Sketch

```tsx
// packages/ui/src/components/auth/AuthGuard.tsx
"use client";

import { useEffect, useState } from "react";
import { useRouter, usePathname } from "next/navigation";
import { Loader2 } from "lucide-react";
import { useAuthStore } from "../../lib/stores/auth-store";

const AUTH_ROUTES = ["/login", "/register", "/forgot-password", "/callback"];

function isAuthRoute(pathname: string): boolean {
  return AUTH_ROUTES.some((r) => pathname.includes(r));
}

export function AuthGuard({
  children,
  locale,
}: {
  children: React.ReactNode;
  locale: string;
}) {
  const [hydrated, setHydrated] = useState(false);
  const router = useRouter();
  const pathname = usePathname();
  const { isAuthenticated, token } = useAuthStore();

  useEffect(() => {
    // Wait for Zustand persist rehydration
    const unsub = useAuthStore.persist.onFinishHydration(() => {
      setHydrated(true);
    });
    // In case already hydrated synchronously
    if (useAuthStore.persist.hasHydrated()) {
      setHydrated(true);
    }
    return unsub;
  }, []);

  useEffect(() => {
    if (!hydrated) return;
    if (!isAuthenticated || !token) {
      const safePath = isAuthRoute(pathname) ? `/${locale}/dashboard` : pathname;
      const redirectParam = isAuthRoute(safePath) ? "" : `?redirect=${encodeURIComponent(safePath)}`;
      router.replace(`/${locale}/login${redirectParam}`);
    }
  }, [hydrated, isAuthenticated, token, pathname, locale, router]);

  if (!hydrated || !isAuthenticated || !token) {
    return (
      <div
        data-testid="auth-loading-spinner"
        className="fixed inset-0 z-50 flex items-center justify-center bg-slate-100"
      >
        <Loader2 className="h-10 w-10 animate-spin text-indigo-600" />
      </div>
    );
  }

  return <>{children}</>;
}
```

### Middleware Chaining Pattern

```ts
// apps/client/middleware.ts
import type { NextRequest } from "next/server";
import { NextResponse } from "next/server";
import createIntlMiddleware from "next-intl/middleware";

const LOCALES = ["bg", "en"] as const;
const DEFAULT_LOCALE = "bg";

// Paths (after locale stripping) that require authentication
const PROTECTED_PATHS = ["/dashboard", "/tenders", "/offers", "/documents", "/team", "/settings", "/setup"];

const intlMiddleware = createIntlMiddleware({
  locales: LOCALES,
  defaultLocale: DEFAULT_LOCALE,
  localePrefix: "always",
});

export default function middleware(req: NextRequest) {
  const { pathname } = req.nextUrl;

  // Strip locale prefix to check path type
  const localePrefix = LOCALES.find((l) => pathname.startsWith(`/${l}/`) || pathname === `/${l}`);
  const pathWithoutLocale = localePrefix
    ? pathname.slice(`/${localePrefix}`.length) || "/"
    : pathname;

  const isProtected = PROTECTED_PATHS.some(
    (p) => pathWithoutLocale === p || pathWithoutLocale.startsWith(`${p}/`)
  );

  if (isProtected) {
    const sessionCookie = req.cookies.get("eusolicit-session");
    if (!sessionCookie) {
      const locale = localePrefix ?? DEFAULT_LOCALE;
      const loginUrl = new URL(`/${locale}/login`, req.url);
      loginUrl.searchParams.set("redirect", pathname);
      return NextResponse.redirect(loginUrl, { status: 307 });
    }
  }

  // Pass through to next-intl for locale handling
  return intlMiddleware(req);
}

export const config = {
  matcher: ["/((?!_next|_vercel|api|.*\\..*).*)" ],
};
```

### Cookie Name Convention
The login API (E02) sets a `eusolicit-session` cookie on successful authentication.
The middleware checks for this cookie by name. The cookie value is not validated in
middleware (signature validation happens server-side in E02 endpoints). This is the
"secondary" layer — the client-side `<AuthGuard>` is the primary, authoritative check
using the Zustand `token`.

### Zustand Persist Hydration API
`useAuthStore.persist.hasHydrated()` — synchronous check (returns `true` if already done).
`useAuthStore.persist.onFinishHydration(cb)` — fires callback when hydration completes;
returns an unsubscribe function. Both are standard Zustand `persist` middleware APIs.

### Protected vs Auth Route Groups

```
app/[locale]/
  (protected)/         ← AuthGuard wraps this layout
    layout.tsx         ← update: wrap children with <AuthGuard locale={params.locale}>
    dashboard/
    tenders/
    ...
  (auth)/              ← AuthRedirect wraps this layout
    layout.tsx         ← update: wrap children with <AuthRedirect locale={params.locale}>
    login/
    register/
    forgot-password/
    callback/
```

---

## Test Expectations (from Epic Test Design)

The following tests from `test-design-epic-03.md` apply to this story. The ATDD suite
must cover all P0 and P1 items; P2 items are expected in the automated suite.

### P0 — Critical (must all pass before merge)

| Test ID | Scenario | Assertion |
|---------|----------|-----------|
| **E03-P0-002** | Unauthenticated visit to `/dashboard` → redirect to `/login?redirect=/dashboard` | Assert redirect URL contains `redirect` param; assert dashboard content NOT rendered before redirect |
| **E03-P0-003** | Full-page loading spinner during auth hydration — no flash of protected content | Throttle network in Playwright (4× CPU slowdown); assert `[data-testid=auth-loading-spinner]` visible; assert protected page content absent before redirect |
| **E03-P0-004** | Authenticated user visiting `/login` or `/register` → redirect to `/dashboard` | Seed authStore with valid token; navigate to `/login` → assert redirect to `/${locale}/dashboard` |

### P1 — High

| Test ID | Scenario | Assertion |
|---------|----------|-----------|
| **E03-P1-017** | `redirect` param round-trip: unauthenticated visit to `/tenders` → login → redirect to `/tenders` | Navigate to protected route; intercept redirect; complete login (mocked); assert final URL is `/tenders` |
| **E03-P1-018** | `authStore.logout()` clears all state and redirects to `/login` | Unit: call `logout()`; assert `user`, `token`, `refreshToken` null, `isAuthenticated` false, localStorage cleared. E2E: click "Sign out" in avatar menu → assert redirect to `/login` |

### P2 — Medium

| Test ID | Scenario | Assertion |
|---------|----------|-----------|
| **E03-P2-018** | Server-side middleware: request without `eusolicit-session` cookie to protected path returns 307 with `Location: /login` | Playwright request without cookie; assert response `status: 307` and `Location` header points to `/login` |
| **E03-P2-019** | Redirect loop protection: inconsistent auth state does not cause infinite chain | Simulate partial auth state (isAuthenticated: true, token: null); assert max 2 redirects; no circular redirect observed |

### Risk Mitigations to Verify

- **E03-R-002 (SEC — Route guard flash, Score 6):** The spinner (`data-testid="auth-loading-spinner"`) must be rendered and `[data-testid=protected-content]` must NOT be visible in Playwright before the redirect completes. Use `page.emulateNetworkConditions` or CPU throttle.
- **E03-R-002 (middleware secondary check):** `E03-P2-018` verifies the server-side cookie guard returns 307 without any React rendering.

---

## Dependencies

| Dependency | Status | Notes |
|-----------|--------|-------|
| `useAuthStore` (S3.5) | ✅ Done | `isAuthenticated`, `token`, `logout()`, `persist.hasHydrated()`, `persist.onFinishHydration()` all available in `packages/ui` |
| `(protected)/layout.tsx` in client and admin apps | ✅ Done | Shell renders; no auth guard yet |
| `(auth)/layout.tsx` in client app | ✅ Done | Centred card layout exists; no auth redirect yet |
| `next-intl` middleware (S3.7) | ✅ Done | `createMiddleware` in place; needs chaining |
| `eusolicit-session` cookie from login API (E02 S2.3) | ✅ Done | Cookie set on `POST /auth/login`; name confirmed |

---

## Out of Scope

- JWT signature validation in middleware (handled by E02 backend)
- Persistent session management (covered by S3.5 `authStore` persist)
- Role-based access control within authenticated routes (covered by E02 S2.10 RBAC middleware)
- Microsoft OAuth2 callback guard (deferred post-MVP per ADR 1.8)

---

## Dev Agent Record

### Implementation Plan

Implemented a two-layer route guard for both `apps/client` and `apps/admin`:

**Layer 1 — Server-side middleware** (`middleware.ts`):
Chained an `eusolicit-session` cookie check before the `next-intl` locale handler in each app. Protected path lists match the `(protected)/` route groups. Auth routes (`/login`, `/register`, etc.) are excluded. Returns 307 with `Location: /${locale}/login?redirect={pathname}` when cookie is absent.

**Layer 2 — Client-side `<AuthGuard>`** (`packages/ui/src/components/auth/AuthGuard.tsx`):
- Subscribes to `useAuthStore.persist.onFinishHydration` via `useEffect` to detect Zustand rehydration.
- Renders an opaque full-page spinner (`data-testid="auth-loading-spinner"`, `fixed inset-0 z-50 bg-slate-100`) while not hydrated or not authenticated.
- After hydration: checks `isAuthenticated && token !== null`. If false, `router.replace` to `/login?redirect={pathname}` with circular-redirect guard (auth-route paths fall back to `/dashboard`).
- If authenticated: renders `{children}`.

**`<AuthRedirect>`** (`packages/ui/src/components/auth/AuthRedirect.tsx`):
- Mirrors `AuthGuard` but for the `(auth)` layout group.
- After hydration: if `isAuthenticated && token !== null`, redirects to the `redirect` query param (validated) or `/dashboard`. Unauthenticated users pass through.

**Login page update** (`apps/client/app/[locale]/(auth)/login/page.tsx`):
- Reads `redirect` query param via `useSearchParams`.
- `onSubmit`: after successful login, navigates to `redirect` param (with loop guard) instead of always `/dashboard`.
- Same guard in the "already authenticated" `useEffect`.

**Sign-out wiring** (both protected layouts):
- `handleSignOut` calls `useAuthStore().logout()` then `router.replace(/${locale}/login)`.
- Passed via `TopBar` → `UserAvatarMenu` (which already had `data-testid="user-avatar-menu-trigger"` and `onSignOut` prop).

**Admin auth pages** (new):
- Created `apps/admin/app/[locale]/(auth)/layout.tsx` wrapping `<AuthRedirect>`.
- Created `apps/admin/app/[locale]/(auth)/login/page.tsx` (minimal login form with redirect-param handling).

**`data-testid="protected-content"`**: Added wrapping `<div data-testid="protected-content">` inside `<AuthGuard>` in both protected layouts. When spinner is shown, this div is not in the DOM → `not.toBeVisible()` passes.

### Completion Notes

- All 10 Acceptance Criteria satisfied (AC1–AC10).
- All 7 Senior Developer Review patch findings resolved (2026-04-09 — second pass).
- `pnpm type-check` — 4/4 packages pass, zero TypeScript errors.
- `pnpm build` — client and admin Next.js builds succeed with zero errors.
- Unit tests: 60 @eusolicit/ui + 354 client + 10 admin = **424 tests pass**, zero regressions.
- ATDD E2E test files: `test.skip()` removed from all 38 tests; `seedAuthState` Playwright API bug fixed.

---

## File List

| File | Action |
|------|--------|
| `eusolicit-app/frontend/packages/ui/src/components/auth/AuthGuard.tsx` | Created / Modified — uses shared auth-routes utility |
| `eusolicit-app/frontend/packages/ui/src/components/auth/AuthRedirect.tsx` | Created / Modified — uses shared auth-routes utility |
| `eusolicit-app/frontend/packages/ui/src/components/auth/index.ts` | Created |
| `eusolicit-app/frontend/packages/ui/src/lib/auth-routes.ts` | **Created** — shared `isAuthRoute`, `isRelativePath`, `getSafeRedirect` |
| `eusolicit-app/frontend/packages/ui/index.ts` | Modified — added `// New in S3.12` export block + auth-routes exports |
| `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx` | Modified — `<AuthGuard>`, `data-testid="protected-content"`, `handleSignOut`, real user from store |
| `eusolicit-app/frontend/apps/client/app/[locale]/(auth)/layout.tsx` | Modified — `<AuthRedirect>` wrapper |
| `eusolicit-app/frontend/apps/client/app/[locale]/(auth)/login/page.tsx` | Modified — redirect round-trip + open-redirect guard via `getSafeRedirect` |
| `eusolicit-app/frontend/apps/client/middleware.ts` | Modified — cookie check before next-intl |
| `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/layout.tsx` | Modified — `<AuthGuard>`, `data-testid="protected-content"`, `handleSignOut`, real user from store |
| `eusolicit-app/frontend/apps/admin/app/[locale]/(auth)/layout.tsx` | Created |
| `eusolicit-app/frontend/apps/admin/app/[locale]/(auth)/login/page.tsx` | Created / Modified — `getSafeRedirect`, i18n error via `tErrors("general")` |
| `eusolicit-app/frontend/apps/admin/middleware.ts` | Modified — cookie check before next-intl |
| `eusolicit-app/e2e/specs/shell/route-guards.spec.ts` | Modified — `test.skip()` removed; `seedAuthState` Playwright arg fixed |
| `eusolicit-app/e2e/specs/shell/route-guards.admin.spec.ts` | Modified — `test.skip()` removed; `seedAuthState` Playwright arg fixed |
| `eusolicit-app/e2e/support/fixtures/auth.fixture.ts` | Modified — cookie name `auth-token` → `eusolicit-session` |
| `eusolicit-app/e2e/support/helpers/auth.helper.ts` | Modified — cookie name `auth-token` → `eusolicit-session` |
| `eusolicit-docs/implementation-artifacts/sprint-status.yaml` | Modified — status: in-progress → review |

---

## Senior Developer Review

### Review Pass 1 (2026-04-09)

**Verdict:** Changes Requested
**Review layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor
**Stats:** 7 patch, 5 defer, 10 dismissed

#### Patch (must fix before merge) — all resolved ✅

- [x] [Review][Patch][HIGH][SEC] **Open redirect via unvalidated `redirect` query param** — `AuthRedirect.tsx:56`, `AuthGuard.tsx:70`, `client login/page.tsx:49+69`, `admin login/page.tsx:53`. The `redirect` query parameter is read from the URL and passed directly to `router.replace(safeRedirect)`. The only sanitisation (`isAuthRoute()`) blocks auth-route substrings but does NOT validate that the value is a relative path. An attacker can craft `?redirect=https://evil.com` and after login the user navigates to the external malicious site. **Fix:** validate that `redirectParam` starts with `/` and does not start with `//` before using it. Example: `const isRelative = (p: string) => p.startsWith("/") && !p.startsWith("//"); const safeRedirect = !redirectParam || !isRelative(redirectParam) || isAuthRoute(redirectParam) ? \`/${locale}/dashboard\` : redirectParam;`. Apply in all four files (or once after extracting the shared utility per F2).

- [x] [Review][Patch][HIGH] **E2E `seedAuthState` helper incompatible with Playwright `addInitScript` API** — `e2e/specs/shell/route-guards.spec.ts`, `e2e/specs/shell/route-guards.admin.spec.ts`. `page.addInitScript(seedAuthState, AUTH_STORE_KEY, AUTH_STORE_AUTHENTICATED)` passes three positional arguments. Playwright's `addInitScript(script, arg?)` accepts only two — the third argument (`AUTH_STORE_AUTHENTICATED`) is silently dropped. `seedAuthState` receives only the key with `value=undefined`, storing `"undefined"` in localStorage. Zustand cannot parse this, so the store never hydrates as authenticated. Every test relying on seeded auth state (AC3-T01, AC3-T02, AC6-T04, P1-018-T01/T02/T03) is silently broken. **Fix:** restructure to use a single object arg: `await page.addInitScript(({key, value}) => localStorage.setItem(key, value), { key: AUTH_STORE_KEY, value: AUTH_STORE_AUTHENTICATED });` — and update `seedAuthState` to destructure accordingly.

- [x] [Review][Patch][MEDIUM] **Hardcoded placeholder user in TopBar** — `client (protected)/layout.tsx:103-104`, `admin (protected)/layout.tsx:92`. Both layouts pass `user={{ name: "Demo User", email: "demo@eusolicit.eu" }}` (or `"Admin User"`) to `<TopBar>` instead of reading the authenticated user from `useAuthStore().user`. Every logged-in user sees "Demo User" in the header. **Fix:** destructure `user` from `useAuthStore()` alongside `logout`, and pass `user={{ name: user?.name ?? "", email: user?.email ?? "" }}` to TopBar.

- [x] [Review][Patch][MEDIUM] **Auth fixture cookie name mismatch** — `e2e/support/fixtures/auth.fixture.ts:36`, `e2e/support/helpers/auth.helper.ts:59`. Both inject a cookie named `auth-token`, but the middleware introduced in S3.12 checks for `eusolicit-session`. Any future test using `authenticatedPage` or `createAuthenticatedContext` will not pass the middleware auth check. **Fix:** update both files to use `name: 'eusolicit-session'`.

- [x] [Review][Patch][LOW] **`AUTH_ROUTES` / `isAuthRoute()` duplicated in 4 files** — `AuthGuard.tsx:8-12`, `AuthRedirect.tsx:8-12`, `client login/page.tsx:19-22`, `admin login/page.tsx:14-16`. DRY violation. A future route addition (e.g. `/reset-password`) must be added in all four copies or the redirect-loop guard becomes inconsistent. **Fix:** extract to `packages/ui/src/lib/utils/auth-routes.ts` and import everywhere.

- [x] [Review][Patch][LOW] **`isAuthRoute()` uses `String.includes()` — fragile substring matching** — Same 4 files. `pathname.includes("/login")` matches any path containing the substring (e.g. `/settings/login-history`). No such paths exist today, but this is a latent bug as the app grows. **Fix:** use path-segment-aware matching: `AUTH_ROUTE_SEGMENTS.some(seg => pathname.split('/').includes(seg.replace('/', '')))` or similar.

- [x] [Review][Patch][LOW] **Admin login error message hardcoded in English** — `admin login/page.tsx:59`. `setError("Invalid credentials. Please try again.")` bypasses i18n. The client login page uses `tErrors("general")`. **Fix:** add `const tErrors = useTranslations("errors");` and call `setError(tErrors("general"))`.

#### Defer (pre-existing or cross-cutting — not caused by S3.12)

- [x] [Review][Defer] **Middleware cookie value not validated** — `client middleware.ts:56`, `admin middleware.ts:54`. Any non-empty cookie value passes the check. Documented as intentional per spec ("signature validation happens server-side in E02 endpoints"). Deferred — middleware is a secondary guard.
- [x] [Review][Defer] **`logout()` doesn't invalidate server-side session or clear cookie** — `client (protected)/layout.tsx:80`, `admin (protected)/layout.tsx:74`. Only clears Zustand state. The `eusolicit-session` cookie (httpOnly, set by E02 backend) persists. Needs a `/auth/logout` API endpoint — E02 scope.
- [x] [Review][Defer] **`loginUser()` response shape not validated** — `client login/page.tsx:63-64`. API response consumed without schema validation. Pre-existing pattern from S3.8.
- [x] [Review][Defer] **`setTokens()` creates partial auth state (user=null, isAuthenticated=true)** — `auth-store.ts:41-42`. AuthGuard would grant access but TopBar/profile components would NPE on null user. Pre-existing S3.5 store design.
- [x] [Review][Defer] **`handleLocaleChange` uses naive `String.replace()`** — Both protected layouts. Replaces only first occurrence — paths where the locale token appears in a non-prefix segment could be corrupted. Pre-existing from S3.3/S3.7.

---

### Review Pass 2 (2026-04-09) — Post-patch verification

**Verdict:** Approve ✅
**Review layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor
**Stats:** 0 patch, 9 defer, 20 dismissed

#### Patch verification (7/7 confirmed resolved)

All 7 patch findings from Review Pass 1 verified fixed:
1. ✅ Open redirect → `getSafeRedirect()` + `isRelativePath()` in shared `auth-routes.ts`. All four consumers import from the shared utility. Additionally, `router.replace()` (Next.js App Router) does not support external URL navigation — even a bypass of `isRelativePath` would 404, not redirect externally.
2. ✅ `seedAuthState` → Uses single-object arg `{ key, value }` in both spec files. Destructured correctly in the function signature.
3. ✅ Hardcoded TopBar user → Both layouts destructure `user` from `useAuthStore()` and pass `user?.name ?? ""`, `user?.email ?? ""`.
4. ✅ Cookie name → Both `auth.fixture.ts` and `auth.helper.ts` use `name: 'eusolicit-session'`.
5. ✅ DRY `isAuthRoute()` → Single source of truth in `packages/ui/src/lib/auth-routes.ts`, imported by AuthGuard, AuthRedirect, client login, admin login.
6. ✅ Substring matching → Segment-based: `pathname.split("/").filter(Boolean)` + `segments.includes(seg)`. No false positives on `/settings/login-history`.
7. ✅ Admin i18n → `const tErrors = useTranslations("errors")` + `setError(tErrors("general"))`.

#### New findings — none blocking

**0 patch findings.** No HIGH or MEDIUM issues. Three review layers analysed 17 files across both apps, middleware, shared utilities, E2E specs, and test helpers. No new actionable issues found.

#### Defer (pre-existing or cross-cutting — not caused by S3.12)

- [x] [Review][Defer] **No hydration timeout for corrupt localStorage** — `AuthGuard.tsx:33-49`. If localStorage contains malformed JSON under key `eusolicit-auth-store`, Zustand persist may fail to hydrate and never fire `onFinishHydration`. User sees permanent spinner. Recovery: clear browser data. Defensive improvement — not blocking.
- [x] [Review][Defer] **AuthRedirect renders children during redirect (brief flash)** — `AuthRedirect.tsx:63`. Authenticated users visiting `/login` briefly see the login form before redirect fires. Spec describes AuthRedirect as a "lighter" component. Cosmetic UX issue, not a security concern.
- [x] [Review][Defer] **Cross-app localStorage key collision** — `auth-store.ts:47`. Both apps use key `eusolicit-auth-store`. Not exploitable with separate origins (different ports), but if apps share an origin (reverse proxy), a regular user's client auth state could leak into admin. Pre-existing S3.5 design.
- [x] [Review][Defer] **Token expiry not tracked client-side** — `AuthGuard.tsx:54`. Guard checks `isAuthenticated && token` but not JWT expiry. Stale token is treated as valid until API call fails. Architectural decision — API endpoints validate JWTs server-side.
- [x] [Review][Defer] Carried forward from Pass 1: Middleware cookie not validated (intentional), `logout()` doesn't clear cookie (E02 scope), `loginUser()` response unvalidated (S3.8), `setTokens()` partial state (S3.5), `handleLocaleChange` naive replace (S3.3/S3.7).

#### Acceptance Criteria audit — all 10 ACs pass

| AC | Status | Evidence |
|----|--------|----------|
| AC1 | ✅ | AuthGuard redirects to `/${locale}/login?redirect={pathname}` with `encodeURIComponent`. Middleware provides secondary 307. |
| AC2 | ✅ | Login pages read `searchParams.get("redirect")`, pass through `getSafeRedirect()`, navigate after successful login. Falls back to `/dashboard`. |
| AC3 | ✅ | `<AuthRedirect>` in `(auth)/layout.tsx` redirects authenticated users to `/dashboard` (or safe redirect param). |
| AC4 | ✅ | Guard checks `isAuthenticated && token`. `isAuthenticated=true, token=null` falls through to redirect logic. |
| AC5 | ✅ | `data-testid="auth-loading-spinner"` with `fixed inset-0 z-50 bg-slate-100`. Children gated behind `!hydrated \|\| !isAuthenticated \|\| !token`. |
| AC6 | ✅ | AuthGuard in both `apps/client/(protected)/layout.tsx` and `apps/admin/(protected)/layout.tsx`. AuthRedirect in both auth layouts. Middleware in both apps. |
| AC7 | ✅ | Both middleware files check `eusolicit-session` cookie on PROTECTED_PATHS. Return 307 with `redirect` param. Auth check runs before intlMiddleware (correct per Implementation Notes). |
| AC8 | ✅ | `isAuthRoute()` rejects auth-route redirect targets → fallback to `/dashboard`. `getSafeRedirect()` rejects non-relative paths. E2E `AC8-T03` asserts `≤ 2` redirects. |
| AC9 | ✅ | Exported under `// New in S3.12` block in `packages/ui/index.ts` (lines 154–165). Imported as `@eusolicit/ui` in both apps. |
| AC10 | ✅ | Dev record confirms `pnpm type-check` (4 packages, 0 errors) and `pnpm build` (client + admin, 0 errors). 424 unit tests pass. |

---

## Change Log

| Date | Change |
|------|--------|
| 2026-04-09 | Implemented Story 3.12 — Client-Side Route Guards & Auth Redirects. Created `AuthGuard` and `AuthRedirect` components in `packages/ui`; exported from `packages/ui/index.ts`. Updated `middleware.ts` in both apps to chain `eusolicit-session` cookie check before next-intl. Updated protected and auth layouts in both apps. Updated login page with redirect param round-trip and loop guard. Created admin auth layout and login page. Removed `test.skip()` from 38 ATDD E2E tests. All builds and type-checks pass; zero regressions in 414 unit tests. |
| 2026-04-09 | Senior Developer Review — Changes Requested. 7 patch findings (2 HIGH, 2 MEDIUM, 3 LOW), 5 deferred. Key issues: open redirect vulnerability in redirect param handling; E2E test seedAuthState helper incompatible with Playwright addInitScript API; hardcoded user info in TopBar. |
| 2026-04-09 | All 7 patch findings resolved. Created `packages/ui/src/lib/auth-routes.ts` (shared `isAuthRoute`, `getSafeRedirect`, `isRelativePath` — fixes DRY + fragile substring matching + open-redirect vulnerability). Updated `AuthGuard`, `AuthRedirect`, client login, admin login to import from shared utility. Fixed `seedAuthState` in both E2E spec files to use single-object Playwright arg. Fixed hardcoded TopBar user to read from `useAuthStore().user`. Fixed E2E fixture/helper cookie name to `eusolicit-session`. Fixed admin login to use `tErrors("general")` i18n. All 424 unit tests pass; type-check clean across 4 packages. Status → Done. |
| 2026-04-09 | Senior Developer Review Pass 2 — **Approved**. 3 parallel adversarial layers (Blind Hunter, Edge Case Hunter, Acceptance Auditor) analyzed 17 files. All 7 Pass 1 patches verified resolved. 0 new patch findings. 4 new deferred items (hydration timeout, AuthRedirect flash, cross-app localStorage key, client-side token expiry). 20 dismissed as noise/false positives. All 10 ACs pass. Status remains Done. |
