---
stepsCompleted:
  - step-01-load-context
  - step-02-discover-tests
  - step-03-map-criteria
  - step-04-analyze-gaps
  - step-05-gate-decision
lastStep: step-05-gate-decision
lastSaved: '2026-04-09'
epic: 3
epicTitle: 'Frontend Shell & Design System'
previousEpic: 2
previousEpicFile: 'eusolicit-docs/test-artifacts/traceability-matrix.md'
runVersion: 2
---

# Traceability Matrix & Quality Gate Report

**Epic:** E03 — Frontend Shell & Design System
**Generated:** 2026-04-09 (Run 2 — updated after `locale-redirect.spec.ts` added)
**Scope:** 12 stories (S03.01 – S03.12), 52 epic-level test IDs (P0: 10 · P1: 18 · P2: 20 · P3: 4)
**TDD Phase:** 🔴 RED — All 711 tests written; implementations not yet started (pre-implementation baseline)
**Previous Epic:** E02 — Authentication & Identity (PASS, 2026-04-07)

---

## TRACE_GATE: PASS

**Rationale:** All gate criteria are now met. P0 coverage is 100% (10/10 FULL). The single P0 blocker from Run 1 — E03-P0-005 (locale redirect loop E2E assertion) — has been resolved: `e2e/specs/shell/locale-redirect.spec.ts` was added with 4 Playwright tests that use `countRedirects()` to assert ≤1 redirect hop for `/` → `/bg/`, `/dashboard` → `/bg/dashboard`, and verify no prefix duplication for `/bg/dashboard` and `/en/dashboard`. S03.07 AC10 is upgraded from PARTIAL to FULL. P1 coverage remains 94.4% (17/18 FULL; ≥90% PASS target met). Overall FULL coverage is 88.5% (46/52; ≥80% minimum met). Remaining PARTIAL items — E03-P1-014 (toast hover-pause browser E2E) and E03-P2-017 (toast hover-pause E2E) — are deferred to the `automate` workflow phase and do not block the gate at the achieved P1 coverage level. All 4 P3 tests (dark mode, animation timing, performance) are intentionally deferred.

> **⚠️ TDD Baseline Notice:** All 711 tests are in `test.skip()` / `it.skip()` state (except 6 passing AC1 regression guards in S3.11). Zero tests currently pass against the production implementation. This is the expected pre-implementation state. Gate measures whether every acceptance criterion has a corresponding test — all P0 and P1 criteria are now fully or partially covered above their respective gate thresholds.

> **✅ To reach GREEN:** Implement Epic 3 stories; unblock tests sequentially following the story order. Re-run this gate after implementation completes to confirm test results match the TDD assertions.

---

## 1. Coverage Statistics

| Dimension | FULL | PARTIAL | NONE | Total | % FULL | % Covered |
|-----------|-----:|--------:|-----:|------:|-------:|----------:|
| **P0** | 10 | 0 | 0 | 10 | **100%** | 100% |
| **P1** | 17 | 1 | 0 | 18 | **94.4%** | 100% |
| **P2** | 19 | 1 | 0 | 20 | **95%** | 100% |
| **P3** | 0 | 0 | 4 | 4 | **0%** | 0% |
| **Overall** | **46** | **2** | **4** | **52** | **88.5%** | **92.3%** |

### Gate Criteria Evaluation

| Criterion | Required | Actual | Status |
|-----------|----------|--------|--------|
| P0 FULL coverage | 100% | 100% | ✅ MET |
| P1 FULL coverage (PASS target) | ≥ 90% | 94.4% | ✅ MET |
| P1 FULL coverage (minimum) | ≥ 80% | 94.4% | ✅ MET |
| Overall FULL coverage | ≥ 80% | 88.5% | ✅ MET |

### Change Summary vs Run 1

| Metric | Run 1 | Run 2 | Δ |
|--------|-------|-------|---|
| P0 FULL | 9/10 (90%) | **10/10 (100%)** | +1 ✅ |
| Overall FULL | 45/52 (86.5%) | **46/52 (88.5%)** | +1 |
| PARTIAL | 3 | **2** | -1 |
| Gate | ❌ FAIL | **✅ PASS** | CLEARED |
| New test files | — | `locale-redirect.spec.ts` (+4 tests) | +4 |
| Total tests | 707 | **711** | +4 |

### Test Volume by Story

| Story | Title | Tests Written | TDD Phase | Test File(s) |
|-------|-------|-------------:|-----------|--------------|
| S03.01 | Next.js 14 Monorepo Scaffold | 44 | 🔴 SKIP | `frontend-monorepo-scaffold.api.spec.ts` |
| S03.02 | Tailwind Design Token Preset & shadcn/ui Theming | 48 | 🔴 SKIP | `design-system-setup.api.spec.ts`, `design-system-smoke.spec.ts` |
| S03.03 | App Shell Layout — Sidebar, Top Bar, Content Area | 42 | 🔴 SKIP | `app-shell-layout.spec.ts`, `app-shell-layout.admin.spec.ts` |
| S03.04 | Responsive Layout Strategy | 26 | 🔴 SKIP | `responsive-layout.spec.ts`, `responsive-layout.admin.spec.ts` |
| S03.05 | Zustand Stores & TanStack Query Setup | 32 | 🔴 SKIP | `auth-store.test.ts`, `ui-store.test.ts`, `api-client.test.ts`, `useSSE.test.ts`, `api-test-page.spec.ts` |
| S03.06 | React Hook Form + Zod Validation Patterns | 27 | 🔴 SKIP | `useZodForm.test.ts`, `FormField.test.tsx`, `form-test-page.spec.ts` |
| S03.07 | i18n Setup with next-intl (BG + EN) | **65** | 🔴 SKIP | 7 unit files + `locale-redirect.spec.ts` (+4 tests) |
| S03.08 | Authentication Pages | 74 | 🔴 SKIP | `auth-pages.spec.ts`, `auth-pages-s3-8.test.ts` |
| S03.09 | Company Profile Setup Wizard | 94 | 🔴 SKIP | `wizard.spec.ts`, `wizard-s3-9.test.ts` |
| S03.10 | Loading States, Error Boundaries & Empty States | 155 | 🔴 SKIP | `loading-states-s3-10.test.ts` |
| S03.11 | Toast Notification System | 66 | 🔴 60 FAIL / ✅ 6 PASS | `toast-s3-11.test.ts` |
| S03.12 | Client-Side Route Guards & Auth Redirects | 38 | 🔴 SKIP | `route-guards.spec.ts`, `route-guards.admin.spec.ts` |
| **Total** | | **711** | **All RED** | 24 test files |

---

## 2. Story-Level Acceptance Criteria Traceability

### S03.01 — Next.js 14 Monorepo Scaffold

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `apps/client` and `apps/admin` contain Next.js 14 App Router projects (`app/` dir, metadata, `lang="bg"`, `next.config.ts`, `transpilePackages`) | `[P0] AC1 — App Router Directory Structure` (8 tests); `[P1] AC1 — Turborepo Root Files` (5 tests) | Build/CI | P0/P1 | **FULL** |
| AC2 | `packages/ui` shared component library with `index.ts` barrel + Button export | `[P0] AC2 — packages/ui Library` (5 tests); `[P2] AC2 — Workspace deps` (1 test) | Build/CI | P0/P2 | **FULL** |
| AC3 | `packages/config` exports shared `tailwind.config.ts`, `tsconfig.json`, `.eslintrc.js` | `[P0] AC3 — packages/config` (4 tests); `[P1] AC3 — Prettier + ESLint` (2 tests) | Build/CI | P0/P1 | **FULL** |
| AC4 | `pnpm dev --filter client` starts on port 3000 | `[P1] AC4/AC5 — Port Config` (5 tests) | Build/CI | P1 | **FULL** |
| AC5 | `pnpm dev --filter admin` starts on port 3001 | `[P1] AC4/AC5 — Port Config` (5 tests) | Build/CI | P1 | **FULL** |
| AC6 | `pnpm build` at root succeeds — both apps exit 0 (E03-P0-001) | `[P0] AC6 — Build Gate` (5 tests including TS + ESLint checks) | Build/CI | P0 | **FULL** |
| AC7 (AC8 in story) | TypeScript strict mode + ESLint + Prettier configured and passing | `[P2] AC7 — TypeScript Strict` (6 tests); `[P1] AC8 — ESLint + Prettier lint` (3 tests) | Build/CI | P1/P2 | **FULL** |

**Story 3.1 AC Coverage: 7/7 ACs = 100% · Epic Test IDs: E03-P0-001 (AC6), E03-P2-001 (AC2), E03-P2-020 (AC7)**

---

### S03.02 — Tailwind Design Token Preset & shadcn/ui Theming

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Tailwind preset: slate neutrals, indigo-600, semantic colours, fonts, shadows, `darkMode: ["class"]`, CSS-var colour mapping | `[P0] AC1 — Tailwind Design Token Preset` (4 tests); `[P1] AC1 — Semantic Colours, Fonts, Shadows, Plugin` (8 tests) | Build/CI | P0/P1 | **FULL** |
| AC2 | CSS variables for `--primary` through `--ring` in `:root` and `.dark`; HSL component format; `--radius`; fonts in layouts | `[P0] AC2 — CSS Variables in globals.css` (4 tests); `[P1] AC2 — CSS Variables Scoped in @layer base` (4 tests); `[P2] AC2 — CSS Design Tokens Applied in Browser` (3 smoke tests) | Build/CI + E2E | P0/P1/P2 | **FULL** |
| AC3 | 19 shadcn/ui components in `packages/ui/src/components/ui/`; barrel exports; old `Button.tsx` removed; `cn()` util | `[P0] AC3 — shadcn/ui Component Files Exist` (5 tests); `[P1] AC3 — Barrel Export and Dependencies` (4 tests); `[P2] AC3 — App-Level shadcn/ui Configuration` (3 tests) | Build/CI | P0/P1/P2 | **FULL** |
| AC4 | `/dev/components` page renders all components with labels and variant previews | `[P2] AC4 — /dev/components Page File` (3 filesystem tests); `[P2] AC4 — /dev/components Component Gallery` (5 browser smoke tests) | E2E | P2 | **FULL** |
| AC5 | Both apps import `<Button>` from `packages/ui`; `pnpm build` exits 0 (E03-P0-001) | `[P0] AC5 — pnpm Build Gate` (3 tests); `[P2] AC5 — TypeScript Strict Mode` (2 tests) | Build/CI | P0/P2 | **FULL** |

**Story 3.2 AC Coverage: 5/5 ACs = 100% · Epic Test IDs: E03-P0-001 (AC5), E03-P2-001 (AC3+AC5), E03-P2-002 (AC4), E03-P2-003 (AC1+AC2), E03-P2-020 (AC5)**

---

### S03.03 — App Shell Layout (Sidebar, Top Bar, Content Area)

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `<AppShell>` with sidebar/topbar/children slots exported from `@eusolicit/ui` | T01, T02, A07 | E2E | P0 | **FULL** |
| AC2 | NavItem active highlight (indigo-50 bg, indigo-600 text); nav item hrefs; inactive state | T06, T07, T08, T09, A12, A13 | E2E | P0/P1 | **FULL** |
| AC3 | Sidebar toggle: 256px ↔ 64px with 200ms transition; labels hidden when collapsed | T10–T14, A08 | E2E | P1 | **FULL** |
| AC4 | TopBar sticky 64px; breadcrumbs left; bell + language selector + avatar right | T18–T22 | E2E | P0/P1 | **FULL** |
| AC5 | Avatar dropdown: name, email, Profile, Settings, Separator, Sign out | T23, T24, T25 | E2E | P2 | **FULL** |
| AC6 | Notifications bell + unread count badge (static placeholder "3") | T26, T27, T28, T29 | E2E | P2 | **FULL** |
| AC7 | Content area scrolls independently; sidebar and topbar fixed | T04 | E2E | P1 | **FULL** |
| AC8 | Client sidebar items: Dashboard, Tenders, Offers, Documents, Team, Settings | T03, T05 | E2E | P0 | **FULL** |
| AC9 | Admin sidebar items: Dashboard, Companies, Tenders, Users, Reports, Settings | A01–A06, A12, A13 | E2E | P0 | **FULL** |
| AC10 | Zustand `uiStore` wired; `sidebarCollapsed` persisted to localStorage; hydration-safe | T15–T17, A09–A11 | E2E | P1 | **FULL** |
| AC11 | `pnpm build` exits 0 (CI gate — E03-P0-001) | _(existing CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.3 AC Coverage: 11/11 ACs = 100% · Epic Test IDs: E03-P0-009 (AC1/AC2/AC4/AC8), E03-P0-010 (AC9), E03-P1-010 (AC3/AC10), E03-P2-004 (AC5), E03-P2-005 (AC6)**

---

### S03.04 — Responsive Layout Strategy

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Desktop (≥1280px): full sidebar visible; user toggle respected; hamburger hidden | T01, T02, T03 | E2E | P1 | **FULL** |
| AC2 | Tablet (768–1279px): auto-collapses on entry; expand possible; auto-collapses on route change | T04, T05, T06 | E2E | P1 | **FULL** |
| AC3 | Mobile (<768px): sidebar hidden; BottomNav shows first 5 items | T07, T08, A03, A04 | E2E | P1 | **FULL** |
| AC4 | BottomNav: fixed, 64px, safe-area-aware, bg-white, border-t; main gets `pb-16` | T09, T10, T11, T14 | E2E | P1 | **FULL** |
| AC5 | BottomNav: active indigo-600, inactive slate-500 | T12, T13 | E2E | P1 | **FULL** |
| AC6 | Hamburger in TopBar opens Sheet overlay; Sheet closes on route change | T15–T18, A05 | E2E | P2 | **FULL** |
| AC7 | No content reflow; no hydration errors; `ml-0` on mobile | T19, T20 | E2E | P2 | **FULL** |
| AC8 | Admin app implements responsive behaviour (tablet collapse, mobile BottomNav, Sheet) | A01–A06 | E2E | P1/P2 | **FULL** |
| AC9 | `pnpm build` exits 0 (CI gate — E03-P0-001) | _(existing CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.4 AC Coverage: 9/9 ACs = 100% · Epic Test IDs: E03-P1-011 (AC1/AC2/AC3/AC8), E03-P2-006 (AC6), E03-P2-007 (AC2/AC8)**

---

### S03.05 — Zustand Stores & TanStack Query Setup

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `authStore`: user, token, refreshToken, isAuthenticated, login(), logout(), setUser(), setTokens(); persisted to localStorage as `"eusolicit-auth-store"` | `auth-store.test.ts` T01–T07 | Unit | P1 | **FULL** |
| AC2 | `uiStore`: sidebarCollapsed, theme, locale, toasts[]; addToast(), removeToast(); partialize persists only `sidebarCollapsed` + `locale` | `ui-store.test.ts` T01–T10 | Unit | P1 | **FULL** |
| AC3 | TanStack Query `QueryClient` with `staleTime: 30s`, `gcTime: 5min`, `retry: 1` wrapped in app root | `api-test-page.spec.ts` T02 (indirect validation) | E2E | P1 | **FULL** |
| AC4 | `apiClient`: Bearer token attached; 401 dedup (3 concurrent → 1 refresh); refresh failure → logout; `_retry` infinite-loop guard | `api-client.test.ts` T01–T06 | Unit | P0/P1 | **FULL** |
| AC5 | `useSSE(url)`: EventSource.close() on unmount; status transitions; JSON parse; error event; null guard | `useSSE.test.ts` T01–T05 | Unit | P2 | **FULL** |
| AC6 | `useHealthCheck` hits `GET /health`; `/dev/api-test` shows loading/success/error states | `api-test-page.spec.ts` T01–T04 | E2E | P1 | **FULL** |
| AC7 | `pnpm build` + `pnpm type-check` exit 0 (CI gate) | _(existing CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.5 AC Coverage: 7/7 ACs = 100% · Epic Test IDs: E03-P0-007 (AC4/T03), E03-P0-008 (AC4/T04-T05), E03-P1-001 (AC1/T06-T07), E03-P1-002 (AC2/T09-T10), E03-P1-012 (AC6), E03-P2-008 (AC5/T01)**

---

### S03.06 — React Hook Form + Zod Validation Patterns

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `useZodForm(schema)`: returns typed UseFormReturn with zodResolver; `mode: "onBlur"` default; options forwarded | `useZodForm.test.ts` T01–T06 | Unit | P2 | **FULL** |
| AC2 | shadcn form primitives in `packages/ui/src/components/ui/form.tsx`; `FormFieldInternal` NOT barrel-exported | _(CI gate: TypeScript build validates non-export)_ | Build/CI | P0 | **FULL** |
| AC3 | `<FormField>` renders: label, input slot, error message (text-destructive), description; render-prop children override; disabled forwarded | `FormField.test.tsx` T01–T05, T14, T15 | Component | P1 | **FULL** |
| AC4 | `<FormField>` type variants: text, email, password, textarea, select, checkbox, radio, date, file | `FormField.test.tsx` T02, T06–T13 | Component | P2 | **FULL** |
| AC5 | Error animation: `transition-all duration-200 ease-in-out overflow-hidden` always present on FormMessage | `FormField.test.tsx` T05; `form-test-page.spec.ts` T06 | Component + E2E | P1/P2 | **FULL** |
| AC6 | `/dev/form-test`: demo form with Zod validation; inline errors on blur + submit; success banner + toast on valid submit | `form-test-page.spec.ts` T01–T06 | E2E | P1/P2 | **FULL** |
| AC7 | All exports resolve; `pnpm build` + `pnpm type-check` exit 0 (CI gate) | _(existing CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.6 AC Coverage: 7/7 ACs = 100% · Epic Test IDs: E03-P1-013 (AC3/AC5), E03-P2-009 (AC1), E03-P2-010 (AC6)**

---

### S03.07 — i18n Setup with next-intl (BG + EN)

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `next-intl` installed; `next.config.mjs` wraps with `createNextIntlPlugin('./i18n.ts')`; `i18n.ts` configures `getRequestConfig` | `i18n-setup.test.ts` T01–T06 | Unit/File | P1 | **FULL** |
| AC2 | `middleware.ts`: `locales: ['bg', 'en']`, `defaultLocale: 'bg'`, `localePrefix: 'always'`; correct matcher pattern | `i18n-setup.test.ts` T07–T11 | Unit/File | P1 | **FULL** |
| AC3 | `app/[locale]/` route structure; `NextIntlClientProvider` + `QueryProvider` in locale layout | _(build-level — verified by pnpm build CI gate)_ | Build/CI | P1 | **FULL** |
| AC4 | `messages/bg.json` + `messages/en.json`: all 6 namespaces present (`common`, `nav`, `auth`, `forms`, `errors`, `wizard`); identical key sets | `check-i18n-keys.test.ts` T01–T11 (client); T01–T07 (admin) | Unit/File | P1 | **FULL** |
| AC5 | Shell chrome uses `useTranslations()`; `UserAvatarMenu` accepts `labels` prop; TopBar aria labels | `UserAvatarMenu-i18n.test.tsx` T01–T04; `TopBar-i18n.test.tsx` T07 | Component | P1 | **FULL** |
| AC6 | TopBar: globe icon + locale codes (BG/EN); clicking inactive locale calls `onLocaleChange` | `TopBar-i18n.test.tsx` T01–T06 | Component | P0 | **FULL** |
| AC7 | `setLocale()` action on `uiStore`; locale written to localStorage; persists across reload | `ui-store-s3-7.test.ts` T01–T05 | Unit | P1 | **FULL** |
| AC8 | `formatDate` + `formatCurrency` in `packages/ui/src/lib/format.ts`; locale-aware output; exported | `format.test.ts` T01–T10 | Unit | P2 | **FULL** |
| AC9 | `check-i18n-keys.mjs` exits 0 on key match, non-zero on mismatch; `pnpm check:i18n` script | `check-i18n-keys.test.ts` T12–T14 (client); T08–T10 (admin) | Unit/Exec | P1 | **FULL** |
| AC10 | `pnpm build` exits 0; `pnpm type-check` exits 0; `/` redirects to `/bg/` with ≤1 redirect | _(CI gate: build + type-check)_; **`locale-redirect.spec.ts` T01–T04: runtime redirect-count E2E** | Build/CI + E2E | P0 | ✅ **FULL** _(was PARTIAL in Run 1)_ |

**Story 3.7 AC Coverage: 10/10 ACs = 100% · Epic Test IDs: E03-P0-005 (AC10 — FULL, locale-redirect.spec.ts), E03-P0-006 (AC6), E03-P1-006 (AC7), E03-P1-007 (AC4/AC9), E03-P2-011 (AC8)**

---

### S03.08 — Authentication Pages

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Auth layout: no sidebar/topbar; centred flex; EU Solicit logo; max-w-md white card | `auth-pages.spec.ts` AC1 block (5 E2E); `auth-pages-s3-8.test.ts` AC1 block (3 unit) | E2E + Unit | P1 | **FULL** |
| AC2 | `/login`: email + password + "Remember me" + "Forgot password?" + "Sign in" + Google OAuth + register link | `auth-pages.spec.ts` Login block (7 E2E); `auth-pages-s3-8.test.ts` AC2 block (4 unit) | E2E + Unit | P0/P1 | **FULL** |
| AC3 | `/register`: company name, EIK, first name, last name, email, password, confirm, terms; POST to `POST /auth/register` | `auth-pages.spec.ts` Register block (7 E2E); `auth-pages-s3-8.test.ts` AC3 block (3 unit) | E2E + Unit | P1 | **FULL** |
| AC4 | `/forgot-password`: email + submit; success state shows "Check your email"; form hidden | `auth-pages.spec.ts` Forgot-pwd block (5 E2E); `auth-pages-s3-8.test.ts` AC4 block (2 unit) | E2E + Unit | P1 | **FULL** |
| AC5 | `/auth/callback`: Suspense boundary; extracts code/state params; loading + error states with testids | `auth-pages.spec.ts` OAuth callback block (7 E2E); `auth-pages-s3-8.test.ts` AC5 block (5 unit) | E2E + Unit | P2 | **FULL** |
| AC6 | Inline Zod errors on blur + submit; all pages; loading state: Loader2 spinner + disabled inputs | `auth-pages.spec.ts` Validation block (10 E2E); `auth-pages-s3-8.test.ts` AC6 block (7 unit) | E2E + Unit | P1 | **FULL** |
| AC7 | Loading states: button spinner + disabled inputs during submission (all auth pages) | `auth-pages.spec.ts` AC7 block (4 E2E) | E2E | P1 | **FULL** |
| AC8 | Authenticated users redirect to `/dashboard` from `/login` and `/register` | `auth-pages.spec.ts` AC8 block (4 E2E) | E2E | P0 | **FULL** |
| AC9 | 8 new auth i18n keys in both `en.json` and `bg.json`; key parity; UI smoke renders English text | `auth-pages-s3-8.test.ts` AC9 block (18 unit); `auth-pages.spec.ts` i18n smoke (7 E2E) | Unit + E2E | P1 | **FULL** |
| AC10 | `pnpm build` + `pnpm type-check` + `pnpm check:i18n` exit 0 | _(CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.8 AC Coverage: 10/10 ACs = 100% · Epic Test IDs: E03-P0-004 (AC8), E03-P1-003 (AC6/AC7), E03-P1-004 (AC6), E03-P1-005 (AC4), E03-P1-007 extension (AC9), E03-P2-012 (AC5)**

---

### S03.09 — Company Profile Setup Wizard

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Wizard page at `(protected)/setup`; `WizardStepper` renders 4 labelled steps with status | `wizard.spec.ts` AC1 block (7 E2E); `wizard-s3-9.test.ts` FS block (3 unit) | E2E + Unit | P1 | **FULL** |
| AC2 | Step 1 — Company Info: pre-filled name/EIK, address, phone, website (optional URL), logo upload with preview | `wizard.spec.ts` AC2 block (7 E2E); `wizard-s3-9.test.ts` schemas block | E2E + Unit | P1 | **FULL** |
| AC3 | Step 2 — CPV: searchable input filters list; dismissible badges; ≥1 required for Next | `wizard.spec.ts` AC3 block (9 E2E); `wizard-s3-9.test.ts` CPV block (5 unit) | E2E + Unit | P1 | **FULL** |
| AC4 | Step 3 — Regions: 6 Bulgarian + 27 EU states; select-all toggles; ≥1 required for Next | `wizard.spec.ts` AC4 block (6 E2E); `wizard-s3-9.test.ts` regions block (4 unit) | E2E + Unit | P1 | **FULL** |
| AC5 | Step 4 — Team Invites: email add/remove; invalid email error; Complete Setup always enabled (optional step) | `wizard.spec.ts` AC5 block (7 E2E); `wizard-s3-9.test.ts` schema block | E2E + Unit | P1 | **FULL** |
| AC6 | Back (no validation), Next (validates current step), Complete Setup (stub POST + redirect to /dashboard) | `wizard.spec.ts` AC6 + Full Flow blocks (6 E2E); `wizard-s3-9.test.ts` API stub block | E2E + Unit | P0/P1 | **FULL** |
| AC7 | Wizard state persisted via Zustand persist (`"eusolicit-wizard-store"`); `logoFile` excluded; `logoDataUrl` included; data survives reload | `wizard.spec.ts` AC7 block (4 E2E); `wizard-s3-9.test.ts` store block (5 unit) | E2E + Unit | P1 | **FULL** |
| AC8 | `wizardStore.reset()` clears store + removes localStorage key after Complete Setup | `wizard.spec.ts` AC8 block (1 E2E) | E2E | P1 | **FULL** |
| AC9 | Register page redirects to `/${locale}/setup` (not /dashboard) after successful registration | `wizard.spec.ts` AC9 block (1 E2E); `wizard-s3-9.test.ts` register block (2 unit) | E2E + Unit | P1 | **FULL** |
| AC10 | All wizard UI strings use `useTranslations()`; all `wizard.*` keys present in en.json + bg.json | `wizard-s3-9.test.ts` i18n block (6 unit) | Unit | P1 | **FULL** |
| AC11 | `pnpm build` + `pnpm type-check` + `pnpm check:i18n` exit 0 | _(CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.9 AC Coverage: 11/11 ACs = 100% · Epic Test IDs: E03-P1-008 (AC6/Full Flow), E03-P1-009 (AC7), E03-P2-013 (AC3), E03-P2-014 (AC4), E03-P2-015 (AC5), E03-P0-002 partial (AC1: unauthenticated /setup guard), E03-R-007 (AC7)**

---

### S03.10 — Loading States, Error Boundaries & Empty States

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | 5 Skeleton variants (`SkeletonCard`, `SkeletonTable`, `SkeletonList`, `SkeletonText`, `SkeletonAvatar`): `animate-pulse` + `bg-slate-200`; `className` prop; configurable rows/cols/lines/size | `loading-states-s3-10.test.ts` AC1 block (~30 tests) | Static/File | P1 | **FULL** |
| AC2 | Global `error.tsx` at `app/[locale]/error.tsx`; `"use client"`; AlertCircle icon; translated heading + Try-again; collapsible details; 3 `data-testid` attrs | `loading-states-s3-10.test.ts` AC2 block | Static/File | P1 | **FULL** |
| AC3 | Per-section `<ErrorBoundary>`: wraps `react-error-boundary`; optional `fallback`; default red-50 card with AlertTriangle; `data-testid="section-error-boundary"` | `loading-states-s3-10.test.ts` AC3 block | Static/File | P1 | **FULL** |
| AC4 | `<EmptyState>`: icon, title, description?, action? props; centred flex; `data-testid="empty-state"` + `data-testid="empty-state-action"` | `loading-states-s3-10.test.ts` AC4 block | Static/File | P2 | **FULL** |
| AC5 | 3 pre-built empty state variants: `EmptyStateNoResults`, `EmptyStateGetStarted`, `EmptyStateNoAccess` | `loading-states-s3-10.test.ts` AC5 block | Static/File | P2 | **FULL** |
| AC6 | `<QueryGuard>`: priority render isLoading→skeleton | isError→error | isEmpty→empty | children; `data-testid` on each wrapper; exports `QueryGuardProps` | `loading-states-s3-10.test.ts` AC6 block | Static/File | P1 | **FULL** |
| AC7 | `states` namespace (6 keys) + 2 `errors` namespace keys added to both `en.json` and `bg.json`; key parity | `loading-states-s3-10.test.ts` AC7 block | Static/File | P1 | **FULL** |
| AC8 | `/dev/ui-states` demo page: `"use client"`; renders all skeleton variants + empty states + QueryGuard states | `loading-states-s3-10.test.ts` AC8 block | Static/File | P2 | **FULL** |
| AC9 | All new symbols in `packages/ui/index.ts` under `// New in S3.10`; types exported | `loading-states-s3-10.test.ts` AC9 block | Static/File | P1 | **FULL** |
| AC10 | `pnpm build` + `pnpm type-check` + `pnpm check:i18n` exit 0 | _(CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.10 AC Coverage: 10/10 ACs = 100% · Epic Test IDs: E03-P1-015 (AC1), E03-P1-016 (AC2/AC3), E03-P2-016 (AC4/AC5)**

---

### S03.11 — Toast Notification System

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | `addToast()` and `removeToast()` from `uiStore`; `Toast` interface shape | `toast-s3-11.test.ts` AC1 block (6 tests — ✅ PASSING regression guards) | Static/Unit | P1 | **FULL** |
| AC2 | `ToastItem`: success/error/warning/info types with colour classes + lucide icons; `data-testid="toast-item"`; props | `toast-s3-11.test.ts` AC2 block (13 tests) | Static/File | P1 | **FULL** |
| AC3 | `ToastContainer`: fixed bottom-right portal; max 5 visible; 8px gap; `data-testid="toast-container"`; reads `uiStore.toasts` | `toast-s3-11.test.ts` AC3 block (10 tests) | Static/File | P1 | **FULL** |
| AC4 | Auto-dismiss: 5s (success/info), 8s (warning), 10s (error); `Infinity` → no dismiss; `clearTimeout` on unmount | `toast-s3-11.test.ts` AC4 block (6 tests) | Static/File | P1 | **FULL** |
| AC5 | Close button (`data-testid="toast-close"`); `onMouseEnter` pauses timer; `onMouseLeave` resumes | `toast-s3-11.test.ts` AC5 block (5 tests); **E2E hover-pause behavior: NOT WRITTEN** | Static/File | P1/P2 | **PARTIAL** |
| AC6 | Entry: slide in from right + fade in; exit: reverse; `translate-x-full`/`translate-x-0` + `opacity-0`/`opacity-100` | `toast-s3-11.test.ts` AC6 block (4 tests) | Static/File | P2 | **FULL** |
| AC7 | `useToast()` hook: `"use client"`; `{ success, error, warning, info }` shorthands wrapping `uiStore.addToast` | `toast-s3-11.test.ts` AC7 block (8 tests) | Static/File | P1 | **FULL** |
| AC8 | `/dev/toasts` demo page: `"use client"`; imports `useToast`; 4 type trigger buttons | `toast-s3-11.test.ts` AC8 block (7 tests) | Static/File | P2 | **FULL** |
| AC9 | `ToastItem`, `ToastItemProps`, `ToastContainer`, `useToast` in `packages/ui/index.ts` under `// New in S3.11`; feedback barrel | `toast-s3-11.test.ts` AC9 block (7 tests) | Static/File | P1 | **FULL** |
| AC10 | `pnpm build` + `pnpm type-check` exit 0 | _(CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.11 AC Coverage: 9/10 ACs FULL, 1/10 PARTIAL (AC5 hover-pause E2E behavior) · Epic Test IDs: E03-P1-014 (AC2/AC3/AC4/AC5 — PARTIAL: static coverage, no browser E2E), E03-P2-017 (AC5 — PARTIAL: handler code verified, runtime behavior not tested)**

---

### S03.12 — Client-Side Route Guards & Auth Redirects

| AC | Description | Test(s) | Level | Priority | Coverage |
|----|-------------|---------|-------|----------|----------|
| AC1 | Unauthenticated user → `/login?redirect=/dashboard`; dashboard content NOT rendered before redirect | AC1-T01, AC1-T02, AC1-T03, AC1-T04 | E2E | P0/P1 | **FULL** |
| AC2 | Post-login redirect round-trip: `redirect` param preserved; defaults to /dashboard if missing | AC2-T01, AC2-T02, AC2-T03 | E2E | P1 | **FULL** |
| AC3 | Authenticated user on `/login` or `/register` → redirect to `/dashboard` | AC3-T01, AC3-T02, AC3-T03 | E2E | P0/P1 | **FULL** |
| AC4 | Corrupt auth state (`isAuthenticated=true, token=null`) → treated as unauthenticated | AC4-T01, AC4-T02 | E2E | P2 | **FULL** |
| AC5 | Full-page loading spinner (`data-testid="auth-loading-spinner"`) visible during hydration; no flash of protected content | AC5-T01, AC5-T02, AC5-T03, AC5-T04 | E2E | P0/P1 | **FULL** |
| AC6 | Guards work in both client (port 3000) and admin (port 3001) apps | AC6-T01–T04, AC7-AC6-T01/T02, AC9-T01 (admin suite) | E2E | P1 | **FULL** |
| AC7 | Next.js middleware server-side redirect (307) when `eusolicit-session` cookie absent | AC7-T01–T05 | E2E/Request | P2 | **FULL** |
| AC8 | Redirect loop protection: `/login?redirect=/login` sanitized; ≤2 redirects for any path | AC8-T01, AC8-T02, AC8-T03 | E2E | P2 | **FULL** |
| AC9 | `AuthGuard` exported from `packages/ui`; admin app starts without import errors | AC9-T01 | E2E | P2 | **FULL** |
| AC10 | `pnpm build` + `pnpm type-check` exit 0 | _(CI gate)_ | Build/CI | P0 | **FULL** |

**Story 3.12 AC Coverage: 10/10 ACs = 100% · Epic Test IDs: E03-P0-002 (AC1), E03-P0-003 (AC5), E03-P0-004 (AC3), E03-P1-017 (AC2), E03-P1-018 (P1-018-T01–T03), E03-P2-018 (AC7), E03-P2-019 (AC4/AC8), E03-R-002 (AC5)**

---

## 3. Epic Test Design Traceability

### P0 Requirements

| Epic Test ID | Requirement | Covering Test(s) | Story | Coverage |
|-------------|-------------|-----------------|-------|----------|
| **E03-P0-001** | `pnpm build` succeeds for both `client` and `admin` — hard CI gate | CI gate (all stories); build gate tests in S3.1 AC6, S3.2 AC5 | S03.01–S03.12 | **FULL** |
| **E03-P0-002** | Unauthenticated user → `/login?redirect=/dashboard`; protected content NOT rendered | `route-guards.spec.ts` AC1-T01, AC1-T02; `wizard.spec.ts` AC1 guard | S03.12, S03.09 | **FULL** |
| **E03-P0-003** | Full-page loading spinner visible during auth hydration; no flash of protected content | `route-guards.spec.ts` AC5-T01, AC5-T02, AC5-T03, AC5-T04 | S03.12 | **FULL** |
| **E03-P0-004** | Authenticated user on `/login` or `/register` → redirect to `/dashboard` | `route-guards.spec.ts` AC3-T01, AC3-T02; `auth-pages.spec.ts` AC8-T01, AC8-T03 | S03.12, S03.08 | **FULL** |
| **E03-P0-005** | Default locale `/` and `/dashboard` → `/bg/dashboard`; ≤1 redirect (no loop) | `locale-redirect.spec.ts` T01 (`/` → `/bg/` ≤1 hop); T02 (`/dashboard` → `/bg/dashboard` ≤1 hop); T03 (no prefix duplication); T04 (`/en/dashboard` no forced redirect) | S03.07 | ✅ **FULL** _(upgraded from PARTIAL in Run 1)_ |
| **E03-P0-006** | Language selector BG→EN: URL updates to `/en/*`; shell chrome re-renders in English | `TopBar-i18n.test.tsx` T01–T06 (component interaction) | S03.07 | **FULL** |
| **E03-P0-007** | 3 concurrent 401s → `POST /auth/refresh` called exactly once → all 3 succeed | `api-client.test.ts` T03 | S03.05 | **FULL** |
| **E03-P0-008** | `apiClient` calls `authStore.logout()` when `POST /auth/refresh` returns 401 | `api-client.test.ts` T04, T05 | S03.05 | **FULL** |
| **E03-P0-009** | Client app shell: collapsible sidebar (6 items), top bar, breadcrumbs, content area | `app-shell-layout.spec.ts` T01–T22 | S03.03 | **FULL** |
| **E03-P0-010** | Admin app shell with admin-specific nav: Dashboard, Companies, Tenders, Users, Reports, Settings | `app-shell-layout.admin.spec.ts` A01–A07 | S03.03 | **FULL** |

### P1 Requirements

| Epic Test ID | Requirement | Covering Test(s) | Story | Coverage |
|-------------|-------------|-----------------|-------|----------|
| **E03-P1-001** | `authStore` persists to localStorage; rehydrates `isAuthenticated=true` | `auth-store.test.ts` T06–T07 | S03.05 | **FULL** |
| **E03-P1-002** | `uiStore.sidebarCollapsed` persists; restored on reload | `ui-store.test.ts` T09–T10 | S03.05 | **FULL** |
| **E03-P1-003** | Login form: invalid email/password show inline Zod errors on blur + submit; loading spinner | `auth-pages.spec.ts` AC6-T01, AC6-T02, AC7-T01, AC7-T02 | S03.08 | **FULL** |
| **E03-P1-004** | Register form: invalid EIK, password mismatch, unchecked terms all show inline errors | `auth-pages.spec.ts` AC6-T04–T07 | S03.08 | **FULL** |
| **E03-P1-005** | Forgot password: valid email → "Check your email" success; form hidden | `auth-pages.spec.ts` AC4-T03–T05 | S03.08 | **FULL** |
| **E03-P1-006** | Locale preference persisted to localStorage; EN locale survives reload | `ui-store-s3-7.test.ts` T01–T05 | S03.07 | **FULL** |
| **E03-P1-007** | All shell chrome i18n keys present in both `bg.json` and `en.json` — sets match exactly | `check-i18n-keys.test.ts` T09–T14 (client); + S3.8 key-parity test; + S3.9 wizard keys | S03.07, S03.08, S03.09, S03.10 | **FULL** |
| **E03-P1-008** | Company setup wizard: Step 1→4 completion; Next validates; "Complete Setup" → POST + redirect | `wizard.spec.ts` Full Wizard Flow block (2 E2E) | S03.09 | **FULL** |
| **E03-P1-009** | Wizard state survives page reload mid-wizard (Step 2 CPV data present after reload) | `wizard.spec.ts` AC7 block (4 E2E) | S03.09 | **FULL** |
| **E03-P1-010** | Sidebar toggle: collapses to ~64px (labels hidden); expand restores; `uiStore` persists | `app-shell-layout.spec.ts` T10–T17; `app-shell-layout.admin.spec.ts` A08–A11 | S03.03 | **FULL** |
| **E03-P1-011** | Desktop (≥1280px) sidebar visible; tablet (768–1279px) auto-collapses; mobile (<768px) bottom nav | `responsive-layout.spec.ts` T01–T13; `responsive-layout.admin.spec.ts` A01–A03 | S03.04 | **FULL** |
| **E03-P1-012** | `useHealthCheck` hits `GET /health`; `/dev/api-test` shows loading/success/error | `api-test-page.spec.ts` T01–T04 | S03.05 | **FULL** |
| **E03-P1-013** | `<FormField>` renders label, input, error message (text-destructive, animated), description | `FormField.test.tsx` T01–T05, T14, T15 | S03.06 | **FULL** |
| **E03-P1-014** | Toast: each type (success/error/warning/info) correct icon + colour; auto-dismisses; close works | `toast-s3-11.test.ts` AC2–AC5 static structure (60 tests); **MISSING: browser-level E2E** | S03.11 | ⚠️ **PARTIAL** |
| **E03-P1-015** | Skeleton variants render with `animate-pulse` + slate-200 background | `loading-states-s3-10.test.ts` AC1 block | S03.10 | **FULL** |
| **E03-P1-016** | Global `error.tsx` catches unhandled errors + "Try again"; per-section `<ErrorBoundary>` | `loading-states-s3-10.test.ts` AC2 + AC3 blocks | S03.10 | **FULL** |
| **E03-P1-017** | `redirect` param preserved through auth flow: `/tenders` → login → redirected to `/tenders` | `route-guards.spec.ts` AC2-T01, AC2-T02, AC2-T03 | S03.12 | **FULL** |
| **E03-P1-018** | `authStore.logout()` clears state + localStorage; redirects to `/login` | `route-guards.spec.ts` P1-018-T01–T03; `auth-store.test.ts` T03 | S03.12, S03.05 | **FULL** |

### P2 Requirements

| Epic Test ID | Requirement | Covering Test(s) | Story | Coverage |
|-------------|-------------|-----------------|-------|----------|
| **E03-P2-001** | Shared `<Button>` from `packages/ui` imports and renders in both apps | S3.1 AC2 tests; S3.2 AC3 barrel export tests | S03.01, S03.02 | **FULL** |
| **E03-P2-002** | `/dev/components` renders all 19 shadcn/ui components without JS errors | `design-system-smoke.spec.ts` AC4 browser tests | S03.02 | **FULL** |
| **E03-P2-003** | Tailwind preset tokens visible: `--primary`, `--destructive`, `--muted` CSS vars set | `design-system-smoke.spec.ts` AC2 browser tests | S03.02 | **FULL** |
| **E03-P2-004** | Avatar dropdown shows name, email, Profile, Settings, divider, Sign out | `app-shell-layout.spec.ts` T23–T25 | S03.03 | **FULL** |
| **E03-P2-005** | Notifications bell renders unread count badge (static placeholder) | `app-shell-layout.spec.ts` T26–T29 | S03.03 | **FULL** |
| **E03-P2-006** | Mobile sidebar opens as Sheet overlay via hamburger | `responsive-layout.spec.ts` T15–T18 | S03.04 | **FULL** |
| **E03-P2-007** | Tablet sidebar auto-collapses after route change | `responsive-layout.spec.ts` T06; `responsive-layout.admin.spec.ts` A06 | S03.04 | **FULL** |
| **E03-P2-008** | `useSSE(url)`: `EventSource.close()` called on unmount | `useSSE.test.ts` T01 | S03.05 | **FULL** |
| **E03-P2-009** | `useZodForm(schema)`: returns form with zodResolver; invalid data → `formState.errors` | `useZodForm.test.ts` T01–T04 | S03.06 | **FULL** |
| **E03-P2-010** | `/dev/form-test`: validates sample schema; inline errors on blur + submit; toast on valid | `form-test-page.spec.ts` T01–T06 | S03.06 | **FULL** |
| **E03-P2-011** | `formatDate` + `formatCurrency` produce locale-specific output for BG vs EN | `format.test.ts` T01–T10 | S03.07 | **FULL** |
| **E03-P2-012** | OAuth callback page: extracts code/state params; shows loading state | `auth-pages.spec.ts` AC5 block (7 E2E) | S03.08 | **FULL** |
| **E03-P2-013** | Wizard Step 2: CPV search filters; zero-selection → validation error | `wizard.spec.ts` AC3 block (9 E2E) | S03.09 | **FULL** |
| **E03-P2-014** | Wizard Step 3: select-all toggle; zero-selection → validation error | `wizard.spec.ts` AC4 block (6 E2E) | S03.09 | **FULL** |
| **E03-P2-015** | Wizard Step 4: email add/remove; Complete without any email → no error | `wizard.spec.ts` AC5 block (7 E2E) | S03.09 | **FULL** |
| **E03-P2-016** | Empty state variants: "No results", "Get started", "No access" with icons and CTAs | `loading-states-s3-10.test.ts` AC4 + AC5 blocks | S03.10 | **FULL** |
| **E03-P2-017** | Hover pauses toast auto-dismiss; close button dismisses immediately | `toast-s3-11.test.ts` AC5 block (`onMouseEnter`/`onMouseLeave` code verified); **MISSING: E2E hover interaction** | S03.11 | ⚠️ **PARTIAL** |
| **E03-P2-018** | Next.js middleware 307 for unprotected cookie on protected route | `route-guards.spec.ts` AC7-T01–T05 | S03.12 | **FULL** |
| **E03-P2-019** | Redirect loop protection: corrupt state + sanitized `redirect` param; ≤2 hops | `route-guards.spec.ts` AC4-T01, AC4-T02, AC8-T01–T03 | S03.12 | **FULL** |
| **E03-P2-020** | TypeScript strict mode: `pnpm build` — zero type errors across all packages | S3.1 AC7 `tsc --noEmit` tests; S3.2 AC5 tsc tests | S03.01, S03.02 | **FULL** |

### P3 Requirements (Deferred)

| Epic Test ID | Requirement | Status | Rationale |
|-------------|-------------|--------|-----------|
| **E03-P3-001** | Dark mode toggle with system-preference detection (stretch AC) | **NONE** | Explicitly deferred — stretch requirement; not implemented in Sprint 1–2 scope |
| **E03-P3-002** | `prefers-color-scheme: dark` → initial dark theme before JS hydration | **NONE** | Depends on E03-P3-001; deferred |
| **E03-P3-003** | Sidebar collapse animation 200ms ease — no visible jank | **NONE** | Manual QA baseline only; visual/timing test deferred post-MVP |
| **E03-P3-004** | `/dashboard` p95 < 2s under 100 concurrent users | **NONE** | k6 load test; nightly run; not in sprint ATDD scope |

---

## 4. Gap Analysis

### Resolved Gap (P0 — Previously Blocked Gate, Now CLOSED)

| ID | Gap | Priority | Resolution |
|----|-----|----------|------------|
| ~~**G-001**~~ | ~~E03-P0-005: Locale redirect-count E2E test missing~~ | ~~P0~~ | ✅ **RESOLVED** — `locale-redirect.spec.ts` added (4 tests using `countRedirects()` to assert ≤1 hop for `/`, `/dashboard`, verify no prefix duplication for `/bg/dashboard` and `/en/dashboard`) |

### Significant Gap (P1 — Accepted at Current P1 = 94.4%)

| ID | Gap | Priority | Risk | Recommended Action |
|----|-----|----------|------|--------------------|
| **G-002** | **E03-P1-014: Toast E2E browser tests missing.** Static code-structure tests verify component files, CSS class names, and timer constant values, but no Playwright E2E test navigates to `/dev/toasts`, triggers each toast type, and asserts visual rendering + auto-dismiss timing. | **P1** | E03-R-002 (secondary — toast visibility) | Run `bmad-testarch-automate` for S03.11 to generate: navigate to `/dev/toasts`; trigger each type; assert toast visible with correct colour; assert auto-dismisses after ~5s/8s/10s; assert close button works |

### Minor Gap (P2)

| ID | Gap | Priority | Risk | Recommended Action |
|----|-----|----------|------|--------------------|
| **G-003** | **E03-P2-017: Toast hover-pause E2E behavior missing.** Static tests verify `onMouseEnter`/`onMouseLeave` handler code, but no E2E test simulates hover to confirm the timer actually pauses and resumes. | **P2** | E03-R-002 (minor) | Add to `automate` phase: trigger toast → `page.hover('[data-testid=toast-item]')` → assert still visible after default duration → `page.mouse.move(0,0)` → assert dismisses |

### Deferred Gaps (P3 — Accepted)

| ID | Gap | Status | Justification |
|----|-----|--------|---------------|
| **G-004** | Dark mode (E03-P3-001/002) | Intentionally deferred | Stretch requirement per epic spec; post-MVP |
| **G-005** | Sidebar animation visual/timing (E03-P3-003) | Intentionally deferred | Manual QA only; no automated mechanism in sprint scope |
| **G-006** | k6 performance test (E03-P3-004) | Intentionally deferred | Nightly run; separate from ATDD sprint scope |

---

## 5. Coverage Heuristics

### API Endpoint Coverage

| Endpoint | E03 Consumer | Tests | Gap? |
|----------|-------------|-------|------|
| `POST /auth/login` | S03.08 login page | `auth-pages.spec.ts` (mocked via `page.route()`) | None |
| `POST /auth/register` | S03.08 register page | `auth-pages.spec.ts` (mocked) | None |
| `POST /auth/refresh` | S03.05 apiClient | `api-client.test.ts` T03–T05 (mocked) | None |
| `POST /auth/forgot-password` | S03.08 forgot-password | `auth-pages.spec.ts` (mocked) | None |
| `GET /auth/callback` | S03.08 OAuth callback | `auth-pages.spec.ts` (mocked) | None |
| `GET /health` | S03.05 useHealthCheck | `api-test-page.spec.ts` (mocked via `page.route()`) | None |
| `POST /companies/:id/profile` | S03.09 wizard Complete Setup | `wizard.spec.ts` (stub — no real HTTP) | None |

No uncovered endpoints within E03 scope.

### Auth/AuthZ Coverage

| Path | Tests | Positive | Negative |
|------|-------|----------|---------|
| Unauthenticated → protected route | AC1-T01–T04 (S3.12) | N/A | ✅ Redirect tested |
| Authenticated → auth page | AC3-T01–T02 (S3.12) | N/A | ✅ Redirect tested |
| Corrupt state (token=null) | AC4-T01–T02 (S3.12) | N/A | ✅ Tested |
| Server-side cookie check | AC7-T01–T05 (S3.12) | AC7-T05 | ✅ AC7-T04 (auth route exempt) |
| Logout → cleared state | P1-018-T01–T03 (S3.12) | N/A | ✅ Tested |

Auth/AuthZ negative paths fully covered for E03 frontend scope.

### Error-Path Coverage

| Scenario | Tests | Happy-Path Only? |
|----------|-------|-----------------|
| 401 refresh dedup | `api-client.test.ts` T03 | No — concurrent 401 scenario |
| Refresh token 401 → logout | `api-client.test.ts` T04–T05 | No — failure path tested |
| SSE EventSource cleanup | `useSSE.test.ts` T01 | No — unmount cleanup |
| Form validation errors | `FormField.test.tsx`, `auth-pages.spec.ts`, `wizard.spec.ts` | No — all show errors |
| Error boundary rendering | `loading-states-s3-10.test.ts` AC2/AC3 | No — error state tested |
| OAuth callback error state | `auth-pages.spec.ts` AC5 block | No — error state tested |
| Wizard zero-selection validation | `wizard.spec.ts` AC3/AC4 | No — error state tested |
| Redirect loop protection | `route-guards.spec.ts` AC8 | No — loop prevention tested |
| Locale redirect hop count | `locale-redirect.spec.ts` T01–T04 | No — loop prevention + hop count tested |

No happy-path-only gaps identified in E03 scope.

---

## 6. Recommendations

| Priority | Action | Requirement(s) | Effort |
|----------|--------|---------------|--------|
| **HIGH** | Run `bmad-testarch-automate` for S03.11 (Toast) to generate E2E browser tests (G-002) | E03-P1-014 | ~4 hours |
| **HIGH** | Set up Vitest in `apps/admin` (identified in S3.7 checklist) to enable 10 pending i18n key tests | E03-P1-007 admin verification | ~1 hour |
| **MEDIUM** | Add toast hover-pause E2E test (G-003) during automate phase | E03-P2-017 | ~2 hours |
| **MEDIUM** | Add `/bg/dashboard` locale prefix to S03.03 shell tests — noted in S3.3 checklist: "those tests will need a URL update when S3.7 lands" | E03-P0-009, E03-P0-010 | ~1 hour |
| **LOW** | Run `bmad-testarch-test-review` to assess test quality across 24 test files (711 tests) | All | ~8 hours |
| **LOW** | Add E03-P3-001/002 dark mode tests when stretch AC is implemented | E03-P3-001, E03-P3-002 | ~4 hours |
| **LOW** | Add E03-P3-004 k6 performance test to nightly CI pipeline | E03-P3-004 | ~4 hours |

---

## 7. Gate Decision Summary

```
✅ GATE DECISION: PASS

📊 Coverage Analysis:
- P0 Coverage: 100% (10/10 FULL; Required: 100%) → ✅ MET
- P1 Coverage: 94.4% (17/18 FULL; PASS target: ≥90%) → ✅ MET
- P1 Coverage: 94.4% (minimum: ≥80%) → ✅ MET
- Overall FULL Coverage: 88.5% (46/52; Minimum: ≥80%) → ✅ MET

✅ Decision Rationale:
All gate criteria are now met. The single P0 blocker from Run 1 (E03-P0-005 — locale
redirect-count E2E) has been resolved. `locale-redirect.spec.ts` was added with 4 Playwright
tests using `countRedirects()` to assert that:
  - `/` redirects to `/bg/` with ≤1 hop and no loops (T01)
  - `/dashboard` redirects to `/bg/dashboard` with ≤1 hop (T02)
  - `/bg/dashboard` does NOT double-prefix to `/bg/bg/dashboard` (T03)
  - `/en/dashboard` is not forcibly redirected to `/bg/` (T04)

P0 is now 100% FULL. P1 remains 94.4% FULL (E03-P1-014 toast hover-pause E2E is
PARTIAL and deferred to automate phase — this does not block PASS at 94.4% ≥ 90%).

⚠️ Open Gaps (Non-Blocking):
- P1 PARTIAL: E03-P1-014 (toast browser E2E — deferred to automate phase)
- P2 PARTIAL: E03-P2-017 (toast hover-pause E2E — deferred to automate phase)
- Deferred P3: dark mode, animation timing, k6 performance (all intentional)

📝 Top Recommended Actions:
1. [HIGH]   Run bmad-testarch-automate for S03.11 to add toast browser E2E tests (~4 hours)
2. [HIGH]   Configure Vitest in apps/admin for 10 pending i18n admin key tests (~1 hour)
3. [MEDIUM] Add toast hover-pause E2E test in automate phase (~2 hours)

📂 Full Report: eusolicit-docs/test-artifacts/traceability-matrix.md

✅ GATE: PASS — Coverage thresholds met. Proceed to implementation.
      Total tests: 711 (all in TDD RED phase).
      Re-run gate after implementation to confirm test results reach GREEN.
```

---

**Generated by:** BMad TEA Agent — Traceability Matrix Workflow
**Workflow:** `bmad-testarch-trace`
**Version:** 4.0 (BMad v6)
**Run:** 2 (updated 2026-04-09 after `locale-redirect.spec.ts` added)
**TDD Phase:** 🔴 RED — Pre-implementation baseline. Re-run after implementation completes.
