---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-08'
workflowType: testarch-atdd
storyId: 3-8-authentication-pages
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/3-8-authentication-pages.md
  - eusolicit-docs/test-artifacts/test-design-epic-03.md
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/frontend/apps/client/vitest.config.ts
  - eusolicit-app/frontend/packages/ui/src/lib/stores/auth-store.ts
  - eusolicit-app/e2e/specs/auth/login.spec.ts
  - eusolicit-app/e2e/support/page-objects/login.page.ts
  - eusolicit-app/e2e/support/fixtures/auth.fixture.ts
  - eusolicit-app/frontend/apps/client/messages/en.json
  - eusolicit-app/frontend/apps/client/messages/bg.json
  - eusolicit-docs/test-artifacts/atdd-checklist-3-7-i18n-setup-with-next-intl-bg-en.md
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 3.8 — Authentication Pages

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**Story:** S03.08 | **Epic:** E03 — Frontend Shell & Design System
**TDD Phase:** 🔴 RED — All tests are skipped (failing) until implementation is complete

---

## Step 1: Preflight & Context

### Stack Detection

| Item | Value |
|------|-------|
| **Detected Stack** | `frontend` — Next.js 14 App Router, React, Zustand, next-intl |
| **Test Framework (unit)** | Vitest (`apps/client` — environment: node) |
| **Test Framework (E2E)** | Playwright (`eusolicit-app/playwright.config.ts`) |
| **Playwright testDir** | `eusolicit-app/e2e/specs/` |
| **Test Artifacts dir** | `eusolicit-docs/test-artifacts/` |

### Prerequisites Verified

| Item | Status | Notes |
|------|--------|-------|
| Story approved with clear ACs | ✅ | 10 ACs; full implementation code in Dev Notes |
| Playwright config found | ✅ | `eusolicit-app/playwright.config.ts` — client base URL `http://localhost:3000` |
| Vitest config found | ✅ | `apps/client/vitest.config.ts` (environment: node) |
| Epic test design loaded | ✅ | `test-design-epic-03.md` — E03-P0-004, P1-003, P1-004, P1-005, P2-012 |
| Auth store key confirmed | ✅ | Zustand persist name: `"eusolicit-auth-store"` |
| Existing test patterns inspected | ✅ | `login.spec.ts`, `auth.fixture.ts`, `login.page.ts` |
| i18n keys confirmed | ✅ | `en.json` auth namespace read — 17 pre-existing keys; 8 new ones required |

### AC Summary

| AC | Description | Priority |
|----|-------------|----------|
| AC1 | Auth layout: no sidebar/topbar, centred flex, EU Solicit logo, max-w-md white card | P1 |
| AC2 | Login page: all fields, links, Google OAuth button; submits to stub; redirects to dashboard | P0 |
| AC3 | Register page: company, EIK, personal fields, terms; submits to stub; redirects | P1 |
| AC4 | Forgot password: email + submit; on success → "Check your email" state, form hidden | P1 |
| AC5 | OAuth callback: code + state from useSearchParams; loading/error states; Suspense boundary | P2 |
| AC6 | Zod validation on blur + submit via useZodForm for all pages; errors in red-500 | P1 |
| AC7 | Loading states: Loader2 spinner + disabled button + disabled inputs during submission | P1 |
| AC8 | Authenticated redirect: !mounted guard + isAuthenticated redirect to dashboard | P0 |
| AC9 | 8 new auth i18n keys in both en.json and bg.json; all visible text uses useTranslations | P1 |
| AC10 | pnpm build exits 0; pnpm type-check exits 0; pnpm check:i18n exits 0 | P0 |

---

## Step 2: Generation Mode

**AI Generation** — Acceptance criteria are explicit with full implementation code in Dev Notes.
Standard auth UI patterns (form validation, loading state, redirect on submit) are well-defined.
No live browser recording needed; acceptance criteria unambiguously describe expected behavior.

---

## Step 3: Test Strategy

### AC → Test Level Mapping

| AC | Description | Test Level | Priority | Epic Test ID |
|----|-------------|------------|----------|--------------|
| AC1 | Auth layout: no sidebar, logo, card | E2E (Playwright) | P1 | — |
| AC2 | Login page elements + happy path redirect | E2E (Playwright) | P0/P1 | E03-P0-004 (partial) |
| AC2 | Login page route file existence | Unit (Vitest/node) | P1 | E03-P0-001 (build) |
| AC3 | Register page elements + validation + redirect | E2E (Playwright) | P1 | E03-P1-004 |
| AC3 | Register page route file existence | Unit (Vitest/node) | P1 | — |
| AC4 | Forgot password success state | E2E (Playwright) | P1 | E03-P1-005 |
| AC5 | OAuth callback: loading/error/Suspense | E2E (Playwright) | P2 | E03-P2-012 |
| AC5 | Callback file: Suspense wrapper + testids | Unit (Vitest/node) | P1 | — |
| AC6 | Inline Zod errors on blur + submit: all pages | E2E (Playwright) | P1 | E03-P1-003, E03-P1-004 |
| AC6 | Schema file exports + EIK regex + API stub file | Unit (Vitest/node) | P1 | — |
| AC7 | Loading spinner + disabled state on all pages | E2E (Playwright) | P1 | E03-P1-003 |
| AC8 | Authenticated redirect: login + register | E2E (Playwright) | P0 | E03-P0-004 |
| AC9 | 8 new keys in en.json (exact values) | Unit (Vitest/node) | P1 | E03-P1-007 (extension) |
| AC9 | 8 new keys in bg.json (non-empty BG translations) | Unit (Vitest/node) | P1 | E03-P1-007 (extension) |
| AC9 | Key parity en.json ↔ bg.json | Unit (Vitest/node) | P1 | E03-P1-007 |
| AC9 | i18n keys render correctly in UI (smoke) | E2E (Playwright) | P1 | — |
| AC10 | Build gate: pnpm build, type-check, check:i18n | CI/Build | P0 | E03-P0-001 |

### Red Phase Compliance

All tests use `test.skip()` (Playwright) or `it.skip()` (Vitest).
They define **expected behavior** that fails until implementation completes.

**Key test isolation decisions:**
- E2E tests use `page.addInitScript()` to seed Zustand localStorage before navigation (no server required for auth state)
- E2E tests use label-based selectors (`page.getByLabel()`) aligned with FormField's `<FormLabel>` rendering
- `data-testid` selectors only for explicitly specified testids (`auth-callback-loading`, `auth-callback-error`)
- Vitest unit tests verify file existence and content via `existsSync()` / `readFileSync()` — no imports of source files (avoids SSR boundary errors in node environment)

---

## Step 4: TDD Red Phase — Generated Test Files

### 🔴 All tests use `test.skip()` / `it.skip()` — they WILL FAIL until implementation completes

| # | Test File | Covers ACs | Priority | Test Count | Framework | Status |
|---|-----------|------------|----------|------------|-----------|--------|
| 1 | `eusolicit-app/e2e/specs/auth/auth-pages.spec.ts` | AC1, AC2, AC3, AC4, AC5, AC6, AC7, AC8, AC9 | P0/P1/P2 | 38 | Playwright E2E | ✅ Created (all skipped) |
| 2 | `eusolicit-app/frontend/apps/client/__tests__/auth-pages-s3-8.test.ts` | AC1–6, AC9 | P1 | 36 | Vitest/node | ✅ Created (all skipped) |

**Total:** 74 failing tests across 2 files.

---

## Step 4C: Test File Details

### File 1: `eusolicit-app/e2e/specs/auth/auth-pages.spec.ts`

**Framework:** Playwright (client project — Chromium, Firefox, Safari, Mobile)
**Run:** `cd eusolicit-app && npx playwright test e2e/specs/auth/auth-pages.spec.ts`
**Base URL:** `http://localhost:3000` (env: local)

#### Test Inventory by Describe Block

| Describe Block | Tests | Priority Range |
|----------------|-------|----------------|
| Auth Layout (AC1) | 5 | P1 |
| Login Page (AC2, AC6, AC7, AC8) | 10 | P0–P1 |
| Register Page (AC3, AC6, AC7, AC8) | 10 | P0–P1 |
| Forgot Password Page (AC4, AC6, AC7) | 8 | P1 |
| OAuth Callback Page (AC5) | 7 | P1–P2 |
| i18n Translation Keys — UI Smoke (AC9) | 7 | P1 |

#### Key Selectors Used

```typescript
// Auth Layout
page.locator('.min-h-screen.bg-slate-50')         // AC1 outer container
page.locator('.max-w-md.bg-white.rounded-xl.shadow-md')  // AC1 card
page.getByText('EU Solicit')                       // AC1 logo

// Login / Register / Forgot Password
page.getByLabel('Email address')                   // FormField email
page.getByLabel('Password')                        // FormField password
page.getByLabel('Remember me')                     // FormField checkbox (login)
page.getByLabel('EIK (Company ID)')                // FormField EIK (register)
page.getByLabel(/I agree to the terms/i)           // FormField agreeToTerms
page.getByRole('button', { name: /sign in/i })     // submit (login)
page.getByRole('button', { name: /register/i })    // submit (register)
page.getByRole('button', { name: /send reset link/i }) // submit (forgot-pwd)
page.getByRole('button', { name: /continue with google/i }) // Google OAuth

// Callback Page (explicit testids from AC5)
page.getByTestId('auth-callback-loading')          // AC5 loading state
page.getByTestId('auth-callback-error')            // AC5 error state
```

#### Auth State Seeding Pattern

```typescript
// Seed Zustand authStore in localStorage before page.goto()
const AUTH_STORE_KEY = 'eusolicit-auth-store'; // persist name from auth-store.ts
await page.addInitScript(
  ({ key, value }) => { localStorage.setItem(key, value); },
  { key: AUTH_STORE_KEY, value: JSON.stringify(AUTH_STORE_AUTHENTICATED) }
);
```

---

### File 2: `eusolicit-app/frontend/apps/client/__tests__/auth-pages-s3-8.test.ts`

**Framework:** Vitest (environment: node)
**Run:** `cd eusolicit-app/frontend && pnpm test --filter client`

#### Test Inventory by Describe Block

| Describe Block | Tests | Priority Range |
|----------------|-------|----------------|
| Auth Route Group Files (AC1–5) | 12 | P1 |
| Auth Zod Schemas and Stub API (AC6) | 10 | P1 |
| messages/en.json new auth keys (AC9) | 9 | P1 |
| messages/bg.json new auth keys (AC9) | 6 | P1 |
| Key parity en.json ↔ bg.json (AC9 / E03-P1-007) | 3 | P1 |

#### Test Failure Modes

```
// Before implementation — each test fails with:
[AC1-U01] ENOENT: no such file: app/[locale]/(auth)/layout.tsx
[AC2-U01] ENOENT: no such file: app/[locale]/(auth)/login/page.tsx
[AC6-U01] ENOENT: no such file: lib/schemas/auth.ts
[AC9-U01] expect(['profile', 'settings', ...]).toContain('googleSignIn') → FAIL
[AC9-U16] missingInBg: ['googleSignIn', 'noAccount', ...] length 8, expected 0 → FAIL
```

---

## Step 5: Validate & Complete

### TDD Red Phase Validation

| Check | Status |
|-------|--------|
| All E2E tests use `test.skip()` | ✅ |
| All unit tests use `it.skip()` | ✅ |
| Tests assert **expected** behavior (not current missing behavior) | ✅ |
| Tests would fail if `skip()` removed before implementation | ✅ |
| Test selectors are stable (label-based for FormField, role-based for buttons) | ✅ |
| No temp files in random locations — tests are in project directories | ✅ |
| Story AC10 (build gate) noted — covered by CI, not new test files | ✅ |

### Coverage Matrix

| AC | E2E Tests | Unit Tests | Total | Epic Link |
|----|-----------|------------|-------|-----------|
| AC1 (layout) | 5 | 3 | 8 | — |
| AC2 (login) | 7 | 4 | 11 | E03-P0-004 |
| AC3 (register) | 7 | 3 | 10 | E03-P1-004 |
| AC4 (forgot-pwd) | 5 | 2 | 7 | E03-P1-005 |
| AC5 (callback) | 7 | 5 | 12 | E03-P2-012 |
| AC6 (validation) | 10 | 7 | 17 | E03-P1-003, E03-P1-004 |
| AC7 (loading) | 4 | 0 | 4 | E03-P1-003 |
| AC8 (auth redirect) | 4 | 0 | 4 | E03-P0-004 |
| AC9 (i18n) | 7 | 18 | 25 | E03-P1-007 |
| AC10 (build) | CI only | CI only | CI gate | E03-P0-001 |

> Note: Some tests cover multiple ACs — counts are approximate.

### Epic Test ID Coverage

| Epic Test ID | Coverage | Test Location |
|-------------|----------|---------------|
| **E03-P0-004** (partial) | Authenticated user redirected from login + register | `auth-pages.spec.ts`: AC8-T01, AC8-T03 |
| **E03-P1-003** | Login validation + loading spinner | `auth-pages.spec.ts`: AC6-T01, AC6-T02, AC7-T01, AC7-T02 |
| **E03-P1-004** | Register validation (EIK, mismatch, terms) | `auth-pages.spec.ts`: AC6-T04, AC6-T05, AC6-T06, AC6-T07 |
| **E03-P1-005** | Forgot password success state | `auth-pages.spec.ts`: AC4-T03, AC4-T04, AC4-T05 |
| **E03-P2-012** | OAuth callback loading state | `auth-pages.spec.ts`: AC5-T01 |
| **E03-P1-007** (extension) | en.json ↔ bg.json key parity for new auth keys | `auth-pages-s3-8.test.ts`: AC9-U16 |
| **E03-P0-001** | Build gate (pnpm build) | CI command (Task 9.1) |

### Quality Gate Alignment

Per `test-design-epic-03.md`:
- **E03-P0-004** must pass before Demo milestone — covered by AC8 tests (P0 priority)
- **E03-P1-007** (i18n key parity) — extended to cover new S3.8 keys in `auth-pages-s3-8.test.ts`
- **Build gate** (E03-P0-001) — AC10 is verified by `pnpm build` in CI (Tasks 9.1, 9.2, 8.3)

---

## Implementation Notes for Developer

### Run Command (after implementation)

```bash
# Remove test.skip() from both test files, then:

# E2E tests (Playwright)
cd eusolicit-app
npx playwright test e2e/specs/auth/auth-pages.spec.ts --project=client-chromium

# Unit tests (Vitest)
cd eusolicit-app/frontend
pnpm test --filter client

# Build gate (AC10)
pnpm build
pnpm type-check
pnpm check:i18n --filter client
```

### Pre-conditions for Tests to Pass

| Order | Task | Required By |
|-------|------|-------------|
| 1 | Create `apps/client/lib/schemas/auth.ts` | AC6 unit tests; AC2–4 E2E validation |
| 2 | Create `apps/client/lib/api/auth.ts` | AC2–5 E2E submission tests |
| 3 | Create `apps/client/app/[locale]/(auth)/layout.tsx` | AC1 E2E tests |
| 4 | Create `apps/client/app/[locale]/(auth)/login/page.tsx` | AC2, AC6, AC7, AC8 E2E tests |
| 5 | Create `apps/client/app/[locale]/(auth)/register/page.tsx` | AC3, AC6, AC7, AC8 E2E tests |
| 6 | Create `apps/client/app/[locale]/(auth)/forgot-password/page.tsx` | AC4, AC6, AC7 E2E tests |
| 7 | Create `apps/client/app/[locale]/(auth)/callback/page.tsx` | AC5 E2E tests |
| 8 | Add 8 keys to `messages/en.json` auth namespace | AC9 unit + E2E smoke tests |
| 9 | Add 8 keys to `messages/bg.json` auth namespace | AC9 unit tests; key parity gate |

### Known Assumptions

1. **Locale prefix**: Tests use `/en/` locale prefix — default Next.js behavior routes to `/bg/` but EN provides predictable English text for assertions. Add locale-parameterized tests post-implementation if needed.
2. **Stub delay**: Auth stub functions have 800ms delay — loading state tests check spinner during that window. Playwright default actionTimeout (15s) covers this.
3. **Zustand persist**: Auth store uses `{ name: "eusolicit-auth-store" }` — confirmed from `auth-store.ts`. If persist config changes, update `AUTH_STORE_KEY` in E2E test.
4. **FormField label rendering**: Tests use `page.getByLabel()` assuming FormField's `<FormLabel>` renders an associated `<label>` with the `label` prop text. If FormField doesn't render accessible labels, switch to role-based or testid selectors.
5. **No hardcoded test IDs**: The implementation code (Dev Notes) doesn't add `data-testid` to form inputs. Tests use `getByLabel()` and `getByRole()` which work if FormField renders proper HTML semantics. The callback page explicitly adds `data-testid="auth-callback-loading"` and `data-testid="auth-callback-error"`.

---

## Completion Summary

**Test files created:**
1. `eusolicit-app/e2e/specs/auth/auth-pages.spec.ts` — 38 Playwright E2E tests (all skipped)
2. `eusolicit-app/frontend/apps/client/__tests__/auth-pages-s3-8.test.ts` — 36 Vitest unit tests (all skipped)

**Checklist saved to:**
`eusolicit-docs/test-artifacts/atdd-checklist-3-8-authentication-pages.md`

**Next recommended workflow:**
1. Implement Story 3.8 tasks (Tasks 1–9)
2. Remove `test.skip()` / `it.skip()` from both test files
3. Run Vitest unit tests first (fast, no browser required): `pnpm test --filter client`
4. Run Playwright E2E tests (requires Next.js dev server): `npx playwright test e2e/specs/auth/auth-pages.spec.ts`
5. Run build gate: `pnpm build && pnpm type-check && pnpm check:i18n --filter client`
6. Once all tests green → run `bmad-testarch-automate` to expand coverage for Story 3.8

---

**Generated by:** BMad TEA Agent — Test Architect Module (ATDD workflow)
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
