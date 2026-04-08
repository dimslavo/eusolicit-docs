---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-07'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 3
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-03-frontend-shell-design-system.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/test-design-progress.md'
---

# Test Design: Epic 3 — Frontend Shell & Design System

**Date:** 2026-04-07
**Author:** TEA Master Test Architect
**Status:** Draft
**Epic:** E03 | **Sprint:** 1–2 (Weeks 1–4) | **Points:** 37

---

## Executive Summary

**Scope:** Epic-level test design for E03 — Frontend Shell & Design System. Covers the Next.js 14 App Router monorepo scaffold (client + admin apps), Tailwind CSS design-token preset and shadcn/ui theming, app shell layout (collapsible sidebar, top bar, breadcrumbs), responsive breakpoint strategy, Zustand state stores, TanStack Query + apiClient, React Hook Form + Zod pattern, next-intl i18n (BG + EN), authentication pages (Login, Register, Forgot Password, OAuth callback), company profile setup wizard (4-step multi-form), skeleton loaders / error boundaries / empty states, toast notifications, and client-side route guards. 12 stories (S03.01–S03.12), 37 points, Sprints 1–2.

This epic is a **platform-wide dependency**: every subsequent feature epic drops UI into the shell and components established here. Regressions in route guards, locale routing, or the API client have cascading impact across all frontend epics.

**Risk Summary:**

- Total risks identified: 10
- High-priority risks (≥6): 3 (locale middleware routing, route guard flash of protected content, 401 refresh deduplication)
- Critical categories: TECH (locale routing, 401 dedup, SSE leak), SEC (route guard flash), BUS (responsive regression, wizard state loss), DATA (i18n key gaps)

**Coverage Summary:**

- P0 scenarios: 10 (~20–35 hours)
- P1 scenarios: 18 (~15–25 hours)
- P2 scenarios: 20 (~10–18 hours)
- P3 scenarios: 4 (~4–8 hours)
- **Total effort:** ~50–86 hours (~1.5–2.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **Backend auth API logic** | JWT issuance, RBAC, token refresh logic fully covered in E02 test design | E03 stubs auth API calls with MSW or mock responses; auth endpoint contract tested in E02 |
| **KraftData / AI features** | AI Gateway not involved in frontend shell or design system | Covered in future epics (E04 onwards) when AI features are integrated |
| **Stripe billing integration** | No subscription UI in E03; billing pages deferred to E05 | Deferred to E05 test design |
| **Full WCAG 2.1 accessibility audit** | MVP scope does not require AA certification | Basic a11y smoke only; full audit deferred post-MVP |
| **Microsoft OAuth2** | Deferred post-MVP (Google OAuth2 + email/password only per ADR 1.8) | Google OAuth2 callback page tested; Microsoft flow deferred |
| **Email delivery verification** | SendGrid integration deferred to E07 Notification Service | Auth pages stub email submission; delivery tested in E07 |
| **Full Storybook infrastructure** | /dev/components visual QA page is in scope; Storybook toolchain is not | /dev/components page covers component visual QA for this epic |
| **IE11 / Safari 12 compatibility** | Modern browsers only per project ADRs | Chromium (Playwright default) + Firefox spot-check sufficient |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E03-R-001** | **TECH** | next-intl locale middleware misconfiguration — incorrect `locales` or `defaultLocale` config causes redirect loops (`/dashboard` → `/bg/dashboard` → `/bg/bg/dashboard`), serves wrong locale silently, or fails to persist locale preference; affects every page in both apps | 2 | 3 | **6** | Explicit middleware config (`locales: ['bg', 'en']`, `defaultLocale: 'bg'`); verify ≤1 redirect per entry path; test localStorage `NEXT_LOCALE` read on initial load and middleware cookie sync | Frontend / QA | Sprint 1–2 |
| **E03-R-002** | **SEC** | Route guard flash of protected content — `<AuthGuard>` reads Zustand after client-side hydration; during the SSR→hydration window, protected page markup may briefly render before redirect, exposing shell structure to unauthenticated users | 2 | 3 | **6** | Guard renders full-page loading spinner until `authStore.persist.hasHydrated()` resolves; E2E test with network throttle asserts spinner visible before redirect; server-side middleware cookie check as secondary layer | Frontend / QA | Sprint 1–2 |
| **E03-R-003** | **TECH** | 401 refresh deduplication — multiple concurrent API requests all receive 401; without a refresh-lock singleton, `apiClient` fires multiple `POST /auth/refresh` calls simultaneously; the second consumed refresh token triggers E02's token family revocation, logging the user out unexpectedly | 2 | 3 | **6** | Implement `Promise` singleton refresh-lock in `apiClient`: queue all subsequent 401s until first refresh resolves, then retry all with new token; unit test asserts `POST /auth/refresh` called exactly once for 3 concurrent 401s | Frontend / QA | Sprint 1–2 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E03-R-004 | TECH | Zustand localStorage hydration mismatch — SSR renders with default state; client hydrates with persisted state; mismatch causes React hydration errors or flickers on first paint (sidebarCollapsed, locale, token) | 2 | 2 | 4 | Use Zustand `skipHydration` + manual `rehydrate()` in `useEffect`; suppress hydration warnings only on layout wrappers; unit test verifies persisted state survives reload | Frontend |
| E03-R-005 | BUS | Responsive breakpoint regression — sidebar/bottom nav rendering at wrong breakpoints breaks mobile experience entirely; `useMediaQuery` hook running on every resize without debounce can cause excessive re-renders | 2 | 2 | 4 | E2E tests with Playwright viewport resizing for each breakpoint tier (mobile/tablet/desktop); debounce resize handler; DevTools device toolbar baseline in CI | QA |
| E03-R-006 | DATA | Missing i18n translation keys — BG/EN `messages/*.json` with nested namespaces (`common`, `nav`, `auth`, `forms`, `errors`, `wizard`); any untranslated key renders as the key string (e.g., "nav.dashboard") in production | 2 | 2 | 4 | Build-time check or test that verifies all keys present in both `bg.json` and `en.json` match the same key set; fail CI if key count differs between locale files | Frontend / QA |
| E03-R-007 | BUS | Company wizard state loss mid-session — wizard state persisted in Zustand; localStorage cleared, browser crash, or serialization failure loses multi-step form progress; user must restart CPV selection and region choices | 2 | 2 | 4 | E2E test: fill 2 steps → reload → verify form data restored; wizard store uses Zustand `persist` middleware with serialization error handling | Frontend / QA |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E03-R-008 | OPS | CSS variable conflicts — shared Tailwind preset in `packages/config` conflicts with app-level `globals.css` overrides; components render with wrong colours or spacing in one app but not the other | 2 | 1 | 2 | Monitor — catch in /dev/components visual QA; both apps import from same preset |
| E03-R-009 | TECH | SSE EventSource memory leak — `useSSE(url)` wraps `EventSource` in `useEffect`; missed cleanup on unmount causes memory leak and ghost listeners in long-running sessions | 1 | 2 | 2 | Monitor — unit test verifies EventSource.close() called on unmount |
| E03-R-010 | OPS | Dark mode flash of incorrect theme — stretch requirement; `prefers-color-scheme` detection via `matchMedia` runs client-side only; SSR renders without dark class causing flash | 1 | 1 | 1 | Monitor — optional stretch AC; document known limitation; deferred to post-MVP polish |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

### System-Level Risk Inheritance

- E03-R-001 (locale routing) → extends system R-013 (i18n Bulgarian + English coverage, score 2). E03 is the concrete implementation of i18n; failure here propagates to all UI-rendered strings across every epic.
- E03-R-003 (401 deduplication) → extends E02-R-003 (token security, score 6). E02 defined token family revocation; E03's apiClient is the primary mechanism that can inadvertently trigger it.

---

## Entry Criteria

- [ ] E01 complete: monorepo build pipeline (pnpm workspaces, Turborepo) operational; CI green
- [ ] E02 complete or auth endpoints stubbed: `POST /auth/login`, `POST /auth/refresh`, `POST /auth/register`, `POST /auth/forgot-password` available (stub via MSW or mock delays)
- [ ] pnpm v8+ and Node.js 20 LTS available in CI runner
- [ ] Playwright configured in `frontend/` for E2E tests (baseline from E01 framework setup)
- [ ] BG locale placeholder translations strategy agreed (placeholder text acceptable for all keys at Sprint 1)
- [ ] Google OAuth2 stub strategy agreed (popup mock or fixed redirect) — real Google consent not needed for E03 testing

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95% or failures triaged and accepted)
- [ ] No open high-severity bugs in core shell navigation or route guards
- [ ] Route guard flash test passing — no protected content visible to unauthenticated users during hydration
- [ ] Locale routing: BG/EN switches without redirect loops; locale persists across reload
- [ ] Both apps (`client` and `admin`) build cleanly with `pnpm build` — exit code 0
- [ ] Branch coverage ≥80% on Zustand stores (`authStore`, `uiStore`, `wizardStore`) and `apiClient`

---

## Test Coverage Plan

**IMPORTANT:** P0/P1/P2/P3 = **priority and risk level** (what to focus on if time-constrained), NOT execution timing. See "Execution Strategy" for when tests run.

### P0 (Critical)

**Criteria:** Blocks core shell functionality + High risk (≥6) + No workaround + All downstream epics depend on it

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E03-P0-001** | `pnpm build` succeeds for both `client` and `admin` apps — no TypeScript errors, no ESLint failures, no import resolution errors | Build/CI | — | Run at root; both apps must exit 0; TypeScript strict mode must not produce type errors |
| **E03-P0-002** | Unauthenticated user visiting `/dashboard` (or any protected route) is redirected to `/login?redirect=/dashboard` | E2E | E03-R-002 | Assert redirect URL includes `redirect` param; assert dashboard content NOT rendered before redirect |
| **E03-P0-003** | Full-page loading spinner shown during auth hydration — no flash of protected page content | E2E | E03-R-002 | Throttle network in Playwright to slow Zustand hydration; assert spinner element visible; assert protected content absent before redirect |
| **E03-P0-004** | Authenticated user visiting `/login` or `/register` is redirected to `/dashboard` | E2E | E03-R-002 | Seed authStore with valid token; navigate to `/login` → assert redirect to `/dashboard` |
| **E03-P0-005** | Default locale routes `/` and `/dashboard` to `/bg/dashboard` without redirect loop (≤1 redirect) | E2E | E03-R-001 | `response.redirectedFrom()` count ≤1; final URL matches `/bg/*`; no infinite loop |
| **E03-P0-006** | Language selector switches BG → EN: URL updates to `/en/*`, all shell chrome strings re-render in English | E2E | E03-R-001 | Click EN selector; assert `window.location.pathname` prefix `/en/`; assert sidebar nav items in English; assert no `next-intl` key strings rendered |
| **E03-P0-007** | 401 refresh deduplication: 3 concurrent API calls all receive 401 → `POST /auth/refresh` called exactly once → all 3 requests retried with new token | Unit | E03-R-003 | Mock `fetch`/`axios`; assert refresh endpoint called once via spy; assert all 3 original requests return 200 after retry |
| **E03-P0-008** | `apiClient` calls `authStore.logout()` when `POST /auth/refresh` itself returns 401 (expired refresh token) | Unit | E03-R-003 | Mock refresh endpoint returning 401; assert `authStore.logout()` called; assert no retry loop |
| **E03-P0-009** | Client app shell renders correctly: collapsible sidebar (Dashboard, Tenders, Offers, Documents, Team, Settings), top bar, breadcrumbs, main content area | E2E | — | Navigate to `/bg/dashboard` (authenticated); assert sidebar nav items present; assert top bar visible with avatar menu trigger |
| **E03-P0-010** | Admin app shell renders with admin-specific nav: Dashboard, Companies, Tenders, Users, Reports, Settings | E2E | — | Navigate to admin app (port 3001) `/bg/dashboard`; assert admin sidebar items |

**Total P0:** 10 tests, ~20–35 hours

---

### P1 (High)

**Criteria:** Important shell/auth flows + Medium risk (3–5) + Common user paths + Workaround exists but painful

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E03-P1-001** | `authStore` persists `user`, `token`, `refreshToken` to localStorage; rehydrates correctly on page reload — `isAuthenticated` true after reload with valid stored token | Unit | E03-R-004 | Use `zustand/middleware` persist; mock localStorage; assert store values after `rehydrate()` |
| **E03-P1-002** | `uiStore.sidebarCollapsed` persists to localStorage; restored on reload | Unit | E03-R-004 | Toggle sidebar; reload; assert collapsed state matches persisted value |
| **E03-P1-003** | Login form: invalid email and short password show inline Zod errors on blur and on submit; valid credentials show loading spinner | E2E | — | Submit with bad email → assert error below email field; submit with valid creds (mocked) → assert button spinner + inputs disabled |
| **E03-P1-004** | Register form: validates all fields — invalid EIK format, password confirm mismatch, unchecked terms all show inline errors | E2E | — | Submit with each invalid variant; assert specific error messages per field; valid submit shows loading state |
| **E03-P1-005** | Forgot password page: valid email submission shows "Check your email" success state; form hidden after submit | E2E | — | Submit with email; assert success message visible; assert form hidden |
| **E03-P1-006** | Locale preference persists to localStorage (`NEXT_LOCALE`) — switching to EN and reloading preserves EN locale | E2E | E03-R-001 | Switch to EN; reload; assert `/en/` prefix in URL; assert sidebar labels in English |
| **E03-P1-007** | All shell chrome i18n keys present in both `messages/bg.json` and `messages/en.json` — key sets match exactly (no missing or extra keys) | Unit/Build | E03-R-006 | Script or test that loads both JSON files, flattens keys, asserts sets are equal; fail if count differs |
| **E03-P1-008** | Company setup wizard: Step 1 → Step 4 full completion — each "Next" validates current step; "Complete Setup" submits and redirects to /dashboard | E2E | E03-R-007 | Fill all 4 steps with valid data; assert stepper advances; assert completion POST call made; assert redirect |
| **E03-P1-009** | Wizard state survives page reload mid-wizard (after Step 2 filled): Step 2 data present after reload | E2E | E03-R-007 | Fill Steps 1–2; reload page; assert Step 2 pre-populated from persisted Zustand store |
| **E03-P1-010** | Sidebar toggle: collapses to icon-only mode (labels hidden, width ~64px); expand restores labels; state persists in `uiStore` | E2E | — | Click toggle; assert no label text visible; assert CSS width transition; reload → assert collapsed state preserved |
| **E03-P1-011** | Desktop (≥1280px) sidebar visible by default; tablet (768–1279px) sidebar auto-collapses to icon-only; mobile (<768px) bottom nav renders instead of sidebar | E2E | E03-R-005 | Playwright `page.setViewportSize()` for each breakpoint; assert correct layout component rendered |
| **E03-P1-012** | TanStack Query health check: `useHealthCheck` hits `GET /health` (mocked) and renders result on `/dev/api-test` | E2E | — | Navigate to /dev/api-test; mock health endpoint 200; assert result text rendered |
| **E03-P1-013** | `FormField` renders label, input, error message (red-500), and description text; error appears with animation on blur with invalid data | Component | — | Mount FormField with Zod schema; trigger blur with invalid value; assert error element visible with red class |
| **E03-P1-014** | Toast: each type (`success`, `error`, `warning`, `info`) renders with correct icon and colour; auto-dismisses after configured duration; close button works | E2E | — | Navigate to /dev/toasts; trigger each type; assert toast visible with correct colour class; assert disappears after timeout |
| **E03-P1-015** | Skeleton variants (`SkeletonCard`, `SkeletonTable`, `SkeletonList`, `SkeletonText`, `SkeletonAvatar`) render with `animate-pulse` class | Component | — | Mount each variant; assert animate-pulse class present; assert slate-200 background |
| **E03-P1-016** | Global error boundary (`error.tsx`) catches render error — shows friendly message + "Try again" button; per-section `<ErrorBoundary>` prevents full-page crash | E2E | — | Trigger React render error via a test component; assert fallback UI rendered; assert rest of page still functional |
| **E03-P1-017** | `redirect` query param preserved through auth flow: unauthenticated visit to `/tenders` → login → redirected to `/tenders` | E2E | E03-R-002 | Navigate to protected route; intercept redirect; complete login (mocked); assert final URL matches original destination |
| **E03-P1-018** | `authStore.logout()` clears `user`, `token`, `refreshToken` from state and localStorage; redirects to `/login` | Unit + E2E | — | Call logout(); assert store values null; assert localStorage keys cleared; E2E: click Sign out in avatar menu → assert redirect to /login |

**Total P1:** 18 tests, ~15–25 hours

---

### P2 (Medium)

**Criteria:** Secondary flows + Low risk (1–2) + Edge cases + Component variants

| Test ID | Requirement | Test Level | Risk Link | Notes |
|---------|-------------|------------|-----------|-------|
| **E03-P2-001** | Shared `<Button>` from `packages/ui` imports and renders without errors in both client and admin apps | Component | — | Import Button from packages/ui in each app; assert renders; assert no console errors |
| **E03-P2-002** | `/dev/components` page renders all 19 shadcn/ui components without JS errors | E2E | — | Navigate to /dev/components; assert each component section heading visible; assert no console errors |
| **E03-P2-003** | Tailwind preset design tokens visible: slate neutrals, indigo-600 primary, semantic status colours, custom shadow scale | E2E | E03-R-008 | Inspect computed styles on /dev/components; assert CSS variables `--primary`, `--destructive`, `--muted` set; assert indigo-600 swatch visible |
| **E03-P2-004** | User avatar dropdown shows name, email, "Profile", "Settings", divider, "Sign out" | E2E | — | Open avatar dropdown; assert all 5 items + divider present |
| **E03-P2-005** | Notifications bell renders unread count badge (static placeholder) | E2E | — | Assert bell icon visible; assert badge element present with non-empty count |
| **E03-P2-006** | Mobile sidebar opens as Sheet overlay via hamburger icon in top bar | E2E | — | Set viewport to 375px; click hamburger; assert Sheet element visible and contains nav items |
| **E03-P2-007** | Tablet sidebar auto-collapses after route change | E2E | E03-R-005 | Set viewport to 900px; expand sidebar; navigate to different route; assert sidebar reverts to collapsed |
| **E03-P2-008** | `useSSE(url)` subscribes to EventSource; `EventSource.close()` called on component unmount | Unit | E03-R-009 | Mock EventSource constructor; mount/unmount useSSE hook; assert close() called on unmount |
| **E03-P2-009** | `useZodForm(schema)` returns configured form with `zodResolver`; submitting invalid data returns field errors | Unit | — | Call useZodForm with test schema; trigger submit with invalid data; assert `formState.errors` populated |
| **E03-P2-010** | `/dev/form-test` validates sample schema (text, email, select, checkbox) and shows toast on valid submit | E2E | — | Navigate to /dev/form-test; submit invalid → errors visible; submit valid → toast visible |
| **E03-P2-011** | `formatDate` and `formatCurrency` helpers render correct locale-specific output for BG vs EN locale | Unit | E03-R-006 | Assert `formatDate` with BG locale formats dates in Bulgarian style; EN formats in English style; `formatCurrency` uses correct decimal separator |
| **E03-P2-012** | OAuth callback page (`/auth/callback`) extracts `code` and `state` query params and shows processing/loading state | E2E | — | Navigate to /auth/callback?code=abc&state=xyz; assert loading indicator visible; mock token exchange |
| **E03-P2-013** | Wizard Step 2: CPV code search input filters list; at least 1 selection required — validation fires on Next without selection | E2E | — | Enter CPV search term; assert filtered results; click Next without selection → assert validation error |
| **E03-P2-014** | Wizard Step 3: region list select-all toggle selects/deselects all; minimum 1 required fires on Next without selection | E2E | — | Click select-all; assert all regions checked; deselect all; click Next → assert error |
| **E03-P2-015** | Wizard Step 4: email add/remove works; step can be skipped (Submit without adding any email) | E2E | — | Add 2 emails; remove 1; assert 1 remains; click Skip/Complete without any email → no error |
| **E03-P2-016** | Empty state variants render with correct icon, title, description, and CTA: "No results found", "Get started", "No access" | Component | — | Mount each EmptyState variant; assert expected text content and action button |
| **E03-P2-017** | Hover on toast pauses auto-dismiss timer; close button dismisses immediately | E2E | — | Trigger toast; hover over it; assert still visible after default duration; move mouse away → assert dismisses; trigger another → click close → immediate dismiss |
| **E03-P2-018** | Next.js middleware server-side redirect: request without session cookie to protected route returns 307 redirect to `/login` | E2E | E03-R-002 | Playwright request without cookie; assert response status 307 and Location header points to /login |
| **E03-P2-019** | Auth redirect loop protection: inconsistent auth state does not cause infinite redirect chain | E2E | E03-R-002 | Simulate partial auth state (token present, no user object); assert max 2 redirects; assert no circular redirect |
| **E03-P2-020** | TypeScript strict mode: `pnpm build` produces zero type errors across all packages and apps | Build | — | Run tsc --noEmit; assert exit code 0; no implicit any, no unused variables under strict rules |

**Total P2:** 20 tests, ~10–18 hours

---

### P3 (Low)

**Criteria:** Nice-to-have + Stretch requirements + Performance benchmarks

| Test ID | Requirement | Test Level | Notes |
|---------|-------------|------------|-------|
| **E03-P3-001** | Dark mode toggle switches theme across both apps; persists selection to localStorage | E2E | Stretch AC from S03.02; skip if dark mode deferred |
| **E03-P3-002** | System `prefers-color-scheme: dark` sets initial theme to dark on first load | E2E | Playwright `colorScheme: 'dark'` emulation; assert dark theme class applied before JS hydration |
| **E03-P3-003** | Sidebar collapse animation runs at 200ms ease — no visible jank or reflow on toggle | E2E | Visual/timing test; use Playwright trace to verify transition; manual QA baseline |
| **E03-P3-004** | `/dashboard` page load p95 < 2s under 100 concurrent users (client app, Next.js SSR) | Performance | k6 load test; nightly run |

**Total P3:** 4 tests, ~4–8 hours

---

## Execution Strategy

**PR (every pull request):** All Playwright E2E, unit, and build tests — expected ~10–15 min with Playwright parallelization. This includes all P0, P1, and P2 tests plus E03-P0-001 build gate. Frontend-only tests run quickly; no reason to defer to nightly.

**Nightly:** k6 performance test (E03-P3-004) — `/dashboard` load test under 100 concurrent users.

**Weekly / Manual:** Dark mode E2E (P3-001, P3-002) if stretch AC is implemented; physical device responsive QA for tablet/mobile (supplement to E2E breakpoint tests); sidebar animation visual QA (P3-003).

**Philosophy:** Run everything in PRs if <15 min. Only defer k6 performance and manual visual checks.

---

## Resource Estimates

| Priority | Count | Total Hours | Notes |
|----------|-------|-------------|-------|
| P0 | 10 | ~20–35 hours | Includes E2E setup for route guard hydration timing; unit test mock scaffolding |
| P1 | 18 | ~15–25 hours | Mix of E2E flows + unit store tests; i18n key check script counts as ~2 hours |
| P2 | 20 | ~10–18 hours | Component mount tests + secondary E2E; many are straightforward assertions |
| P3 | 4 | ~4–8 hours | k6 script + dark mode E2E; skip if stretch ACs not implemented |
| **Total** | **52** | **~50–86 hours** | **~1.5–2.5 weeks, 1 QA** |

### Prerequisites

**Test Data:**

- `authSessionFixture` — seeds Zustand authStore with valid token + user; cleans up after test
- `wizardStateFactory` — creates partial/complete wizard state objects for each step
- MSW service worker (or Playwright `route()` intercept) for auth endpoint stubs

**Tooling:**

- Playwright (already configured) — E2E + network intercept for API mocks
- Vitest or Jest for unit tests on Zustand stores and apiClient hooks
- i18n key diff script (Node.js) — compares `bg.json` and `en.json` key sets at build time

**Environment:**

- Both `client` (port 3000) and `admin` (port 3001) apps running in test mode
- Next.js `NEXT_PUBLIC_API_URL` set to mock server or test base URL
- MSW configured in both apps for API stub in E2E context

---

## Quality Gate Criteria

- **P0 pass rate:** 100% (no exceptions; build gate E03-P0-001 is hard-fail)
- **P1 pass rate:** ≥95% (failures require triage and PM sign-off)
- **Route guard tests (E03-R-002):** 100% — P0-002, P0-003, P0-004 must all pass before Demo milestone
- **Locale routing (E03-R-001):** 100% — P0-005, P0-006 must pass; redirect loop test must pass
- **Both apps build:** Hard gate — CI fails the PR if either app fails `pnpm build`
- **Branch coverage:** ≥80% on `authStore.ts`, `uiStore.ts`, `wizardStore.ts`, `api-client.ts`

---

## Mitigation Plans

### E03-R-001: next-intl Locale Middleware Routing Failure (Score: 6)

**Mitigation Strategy:**
1. Explicit middleware config: `locales: ['bg', 'en']`, `defaultLocale: 'bg'`, with `localeDetection: true` (Accept-Language) and cookie-based override (`NEXT_LOCALE`)
2. Verify ≤1 redirect for all entry paths: `/`, `/dashboard`, `/en/dashboard` — no redirect chains
3. Test that `NEXT_LOCALE` cookie set by middleware is read correctly on subsequent requests
4. Build-time validation: i18n key diff script (P1-007) catches missing BG/EN keys before deploy

**Owner:** Frontend Lead
**Timeline:** Sprint 1 (configuration), Sprint 2 (full validation)
**Status:** Planned
**Verification:** E2E E03-P0-005, E03-P0-006, E03-P1-006; redirect count assertion via `response.redirectedFrom()` in Playwright

---

### E03-R-002: Route Guard Flash of Protected Content (Score: 6)

**Mitigation Strategy:**
1. `<AuthGuard>` renders opaque full-page loading spinner until `useAuthStore.persist.hasHydrated()` returns `true` — no partial page render during hydration
2. Server-side middleware (`middleware.ts`) checks session cookie as secondary guard (returns 307 before React renders)
3. `suppressHydrationWarning` used only on outermost `<html>` tag — never on content areas
4. Playwright E2E test with network throttle (CPU slowdown 4×) verifies spinner visible before redirect completes

**Owner:** Frontend Lead
**Timeline:** Sprint 1 (implementation), Sprint 2 (E2E verification)
**Status:** Planned
**Verification:** E2E E03-P0-002, E03-P0-003, E03-P0-004, E03-P2-018; assert protected content absent via `expect(page.locator('[data-testid=protected-content]')).not.toBeVisible()` before redirect

---

### E03-R-003: 401 Refresh Deduplication (Score: 6)

**Mitigation Strategy:**
1. Implement refresh-lock singleton in `apiClient`: `let refreshPromise: Promise<Token> | null = null`; on 401, if `refreshPromise` is null start refresh; otherwise await existing promise
2. All queued requests retry with new token after single refresh resolves
3. On refresh 401 (expired refresh token): call `authStore.logout()` and reject all queued requests with `SessionExpiredError`
4. Unit test: mock 3 concurrent requests returning 401; assert refresh called once; assert all 3 retried successfully

**Owner:** Frontend Lead
**Timeline:** Sprint 1 (apiClient implementation), Sprint 2 (unit test)
**Status:** Planned
**Verification:** Unit tests E03-P0-007, E03-P0-008

---

## Assumptions and Dependencies

### Assumptions

1. E01 complete and monorepo builds cleanly before E03 testing begins
2. E02 auth endpoints available as stubs (MSW or test server mock) — real JWT generation not required for E03 frontend testing
3. BG locale placeholder translations cover all key namespaces (`common`, `nav`, `auth`, `forms`, `errors`, `wizard`) at Sprint 1 start; accuracy of Bulgarian text not tested in this epic
4. Playwright version compatible with Next.js 14 App Router (no known incompatibilities in Playwright ≥1.40)
5. Both `client` and `admin` apps start reliably in CI without port conflicts

### Dependencies

| Dependency | Required By | Blocks |
|-----------|-------------|--------|
| E01 monorepo scaffold + pnpm workspace stable | Sprint 1 Week 1 | All E03 tests |
| MSW or Playwright `route()` auth stubs for `/auth/login`, `/auth/refresh`, `/auth/register` | Sprint 1 Week 2 | E03-P1-003, P1-004, P1-005, P0-003, P0-004 |
| `packages/ui` barrel exports stable (Button, FormField, AppShell) | Sprint 1 Week 2 | E03-P2-001, E03-P1-013, E03-P1-015 |
| CPV codes static JSON available (for wizard Step 2 search) | Sprint 2 Week 3 | E03-P2-013 |
| Both apps start in CI (Docker or process-based) | Sprint 1 Week 2 | All E2E tests |

### Risks to Plan

- **E02 auth APIs delayed** → E03 auth page and route guard E2E tests depend on stubs; if MSW setup is complex, defer to integration phase when E02 APIs are available. Contingency: use Playwright `route()` intercept for all auth endpoints — no MSW required.
- **pnpm workspace resolution instability** (inherited from E01-R-004) → shared packages import failures block all component tests. Contingency: pin package versions and use `pnpm dedupe` in CI.

---

## Interworking & Regression

| Service/Component | Impact | Regression Scope |
|-------------------|--------|------------------|
| **E01 (monorepo scaffold)** | All frontend builds and packages/ui imports depend on workspace resolution | `pnpm build` at root must pass; shared package imports must resolve in both apps |
| **E02 (auth backend)** | Login, register, token refresh, logout flows consumed by E03 frontend | Auth page E2E tests and route guard tests; any E02 API contract change breaks E03-P0-007/P0-008 |
| **E04–E12 (all future frontend epics)** | All future epics drop UI into the shell; AppShell, design tokens, Zustand stores, apiClient, FormField are shared dependencies | Regression: packages/ui component imports; authStore/uiStore shape; apiClient interface; next-intl namespace structure |

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk scoring (P×I), gate decision rules
- `probability-impact.md` — Probability and impact scale definitions
- `test-levels-framework.md` — E2E vs Component vs Unit selection criteria
- `test-priorities-matrix.md` — P0–P3 criteria and coverage targets

### Related Documents

- **Epic:** `eusolicit-docs/planning-artifacts/epic-03-frontend-shell-design-system.md`
- **System-level architecture test design:** `eusolicit-docs/test-artifacts/test-design-architecture.md`
- **System-level QA test design:** `eusolicit-docs/test-artifacts/test-design-qa.md`
- **E01 test design (infra dependency):** `eusolicit-docs/test-artifacts/test-design-epic-01.md`
- **E02 test design (auth dependency):** `eusolicit-docs/test-artifacts/test-design-epic-02.md`

---

## Follow-on Workflows

- Run `*atdd` to generate failing P0 tests (separate workflow; not auto-run).
- Run `*automate` for broader coverage once E03 stories are implemented.

---

**Generated by:** BMad TEA Agent — Test Architect Module
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
