---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-09'
workflowType: atdd
tddPhase: RED
storyKey: 3-12-client-side-route-guards-auth-redirects
inputDocuments:
  - eusolicit-docs/implementation-artifacts/3-12-client-side-route-guards-auth-redirects.md
  - eusolicit-docs/test-artifacts/test-design-epic-03.md
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/e2e/specs/auth/auth-pages.spec.ts
  - eusolicit-app/e2e/specs/shell/app-shell-layout.spec.ts
  - eusolicit-app/e2e/support/fixtures/auth.fixture.ts
  - eusolicit-docs/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 3.12 — Client-Side Route Guards & Auth Redirects

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**Story:** S03.12 | **Epic:** E03 — Frontend Shell & Design System
**Status:** 🔴 TDD RED PHASE — All tests skipped until feature implemented
**Stack:** Frontend (Next.js 14, React, Zustand, Playwright)

---

## TDD Red Phase Summary

| Test Suite | File | Tests | Phase |
|---|---|---|---|
| Client App Route Guards (E2E + request) | `e2e/specs/shell/route-guards.spec.ts` | 29 | 🔴 RED |
| Admin App Route Guards (AC6) | `e2e/specs/shell/route-guards.admin.spec.ts` | 9 | 🔴 RED |
| **Total** | | **38** | **🔴 RED** |

All tests use `test.skip()` — they document expected behaviour before implementation.

---

## Acceptance Criteria Coverage

| AC | Description | Test IDs | Priority | Tests |
|---|---|---|---|---|
| **AC1** | Unauthenticated redirect to `/login?redirect=/dashboard` | E03-P0-002 | P0 | AC1-T01, AC1-T02, AC1-T03, AC1-T04 |
| **AC2** | Post-login redirect round-trip via `redirect` param | E03-P1-017 | P1 | AC2-T01, AC2-T02, AC2-T03 |
| **AC3** | Authenticated user redirected away from auth pages | E03-P0-004 | P0 | AC3-T01, AC3-T02, AC3-T03 |
| **AC4** | Corrupt state (`isAuthenticated=true, token=null`) → unauthenticated | E03-P2-019 | P2 | AC4-T01, AC4-T02 |
| **AC5** | Hydration spinner — no flash of protected content | E03-P0-003 | P0 | AC5-T01, AC5-T02, AC5-T03, AC5-T04 |
| **AC6** | Guards work in both client + admin apps | E03-P0-002/003/004 | P1 | AC6-T01 through T04, AC7-AC6-T01/T02, AC9-T01 |
| **AC7** | Server-side middleware 307 when `eusolicit-session` cookie absent | E03-P2-018 | P2 | AC7-T01 through T05 |
| **AC8** | Redirect loop protection (≤2 redirects, no circular) | E03-P2-019 | P2 | AC8-T01, AC8-T02, AC8-T03 |
| **AC9** | `AuthGuard` exported from `packages/ui` | — | P2 | AC9-T01 (admin file) |
| **AC10** | `pnpm build` + `pnpm type-check` pass | — | P0 | CI gate — not in E2E suite |
| **E03-P1-018** | Logout clears state + redirects to `/login` | E03-P1-018 | P1 | P1-018-T01, P1-018-T02, P1-018-T03 |

---

## Test Files Created

### `eusolicit-app/e2e/specs/shell/route-guards.spec.ts`

**Playwright project:** `client-chromium`, `client-firefox`, `client-webkit`, `client-mobile`
**Base URL:** `http://localhost:3000`

| Test ID | Priority | Acceptance Criteria | Description |
|---|---|---|---|
| AC1-T01 | P0 | AC1 / E03-P0-002 | Unauthenticated visit to /dashboard → `/login?redirect=/dashboard` |
| AC1-T02 | P0 | AC1 / E03-P0-002 | Protected content NOT rendered before redirect fires |
| AC1-T03 | P1 | AC1 | Unauthenticated /tenders → `/login?redirect=/tenders` |
| AC1-T04 | P1 | AC1 | Unauthenticated /offers → `/login?redirect=/offers` |
| AC5-T01 | P0 | AC5 / E03-P0-003 | `[data-testid=auth-loading-spinner]` visible before redirect |
| AC5-T02 | P0 | AC5 / E03-R-002 | Protected content NOT visible under CPU throttle (4×) |
| AC5-T03 | P1 | AC5 | Spinner has `fixed inset-0 z-50 bg-slate-100` classes |
| AC5-T04 | P1 | AC5 | Spinner contains Lucide `animate-spin` SVG icon |
| AC3-T01 | P0 | AC3 / E03-P0-004 | Authenticated user on /login → redirect to /dashboard |
| AC3-T02 | P0 | AC3 / E03-P0-004 | Authenticated user on /register → redirect to /dashboard |
| AC3-T03 | P1 | AC3 | Unauthenticated user CAN access /login (not blocked) |
| AC2-T01 | P1 | AC2 / E03-P1-017 | Redirect param in login URL encodes original path |
| AC2-T02 | P1 | AC2 / E03-P1-017 | After login, user redirected to originally requested path |
| AC2-T03 | P1 | AC2 | Missing redirect param → post-login defaults to /dashboard |
| AC4-T01 | P2 | AC4 / E03-P2-019 | `isAuthenticated=true, token=null` → redirect to /login |
| AC4-T02 | P2 | AC4 | Corrupt state does not render protected content |
| AC8-T01 | P2 | AC8 / E03-P2-019 | Redirect param → /login sanitized to /dashboard |
| AC8-T02 | P2 | AC8 | Redirect param → /register sanitized to /dashboard |
| AC8-T03 | P2 | AC8 | Maximum 2 redirects for any entry path |
| AC7-T01 | P2 | AC7 / E03-P2-018 | `/en/dashboard` without cookie → 307 to /login |
| AC7-T02 | P2 | AC7 | 307 Location header includes `redirect=` param |
| AC7-T03 | P2 | AC7 | `/en/tenders` without cookie → 307 |
| AC7-T04 | P2 | AC7 | `/en/login` without cookie → NOT 307 (auth route exempt) |
| AC7-T05 | P2 | AC7 | With `eusolicit-session` cookie → NOT 307 (passes through) |
| P1-018-T01 | P1 | E03-P1-018 | Click Sign Out → redirect to /login |
| P1-018-T02 | P1 | E03-P1-018 | After logout, /dashboard redirects to /login again |
| P1-018-T03 | P1 | E03-P1-018 | Auth store in localStorage cleared after logout |

**Subtotal: 29 tests**

---

### `eusolicit-app/e2e/specs/shell/route-guards.admin.spec.ts`

**Playwright project:** `admin-chromium`
**Base URL:** `http://localhost:3001`

| Test ID | Priority | Acceptance Criteria | Description |
|---|---|---|---|
| AC6-T01 | P1 | AC6 / E03-P0-002 | Admin: unauthenticated /dashboard → redirect to /login?redirect= |
| AC6-T02 | P1 | AC6 / E03-R-002 | Admin: protected content NOT visible before redirect |
| AC6-T03 | P1 | AC5 / AC6 / E03-P0-003 | Admin: `[data-testid=auth-loading-spinner]` visible during hydration |
| AC6-T04 | P1 | AC3 / AC6 / E03-P0-004 | Admin: authenticated user on /login → redirect to /dashboard |
| AC7-AC6-T01 | P2 | AC7 / AC6 / E03-P2-018 | Admin: /en/dashboard without cookie → 307 |
| AC7-AC6-T02 | P2 | AC7 / AC6 | Admin: with session cookie → NOT 307 |
| AC9-T01 | P2 | AC9 | Admin app starts without import errors (AuthGuard resolvable) |

**Subtotal: 9 tests**

---

## Priority Distribution

| Priority | Tests | Criteria |
|---|---|---|
| P0 | 7 | Critical — blocks merge; all must pass |
| P1 | 16 | High — common paths; ≥95% required |
| P2 | 15 | Medium — edge cases; cover before sprint close |
| P3 | 0 | — |
| **Total** | **38** | |

---

## Why Tests Will Fail (Red Phase Failure Modes)

When `test.skip()` is removed before implementation:

| Failure Scenario | Root Cause |
|---|---|
| `/en/dashboard` renders without redirect | `AuthGuard` not created or not applied in `(protected)/layout.tsx` |
| No `[data-testid=auth-loading-spinner]` found | `AuthGuard` spinner not implemented |
| Protected content visible under CPU throttle | `AuthGuard` not rendering opaque spinner covering children |
| Authenticated user can access `/login` | `AuthRedirect` not created or not applied in `(auth)/layout.tsx` |
| No `redirect` param in login URL | `AuthGuard` not calling `router.replace(`...?redirect=${pathname}`)` |
| After login, user lands on `/dashboard` instead of `/tenders` | Login page not reading `redirect` param after auth success |
| Corrupt state (`token=null`) grants access | `AuthGuard` only checking `isAuthenticated`, not `&& token !== null` |
| 307 not returned from middleware | `middleware.ts` not updated to check `eusolicit-session` cookie |
| Location header missing `redirect=` | `loginUrl.searchParams.set('redirect', pathname)` not implemented |
| Circular redirect to `/login` from redirect param | `isAuthRoute()` check not applied before building redirect param |
| Sign Out does not redirect to `/login` | `authStore.logout()` not called / router not navigating post-logout |
| Admin app: all guards absent | `AuthGuard` / middleware not applied in `apps/admin` |
| Admin app returns 500 | `AuthGuard` not exported from `packages/ui/index.ts` |

---

## Implementation Guidance

### Files to Create (RED phase → GREEN phase)

| File | Action | Required For |
|---|---|---|
| `packages/ui/src/components/auth/AuthGuard.tsx` | **Create** | AC1, AC4, AC5, AC6 |
| `packages/ui/src/components/auth/AuthRedirect.tsx` | **Create** | AC3, AC6 |
| `packages/ui/src/components/auth/index.ts` | **Create** (barrel) | AC9 |
| `packages/ui/index.ts` | **Update** — add `// New in S3.12` export block | AC9 |
| `apps/client/app/[locale]/(protected)/layout.tsx` | **Update** — wrap children with `<AuthGuard locale={params.locale}>` | AC1, AC5 |
| `apps/client/app/[locale]/(auth)/layout.tsx` | **Update** — wrap children with `<AuthRedirect locale={params.locale}>` | AC3 |
| `apps/client/middleware.ts` | **Update** — chain auth cookie check before next-intl | AC7 |
| `apps/admin/app/[locale]/(protected)/layout.tsx` | **Update** — wrap with `<AuthGuard locale={params.locale}>` | AC6 |
| `apps/admin/middleware.ts` | **Update** — same pattern as client middleware | AC6, AC7 |

### Key Test Data Requirements

| Fixture | Used In | Description |
|---|---|---|
| `AUTH_STORE_AUTHENTICATED` (inline constant) | AC3, AC6, P1-018 | Zustand state with valid token — injected via `page.addInitScript()` |
| `AUTH_STORE_CORRUPT` (inline constant) | AC4 | `isAuthenticated=true, token=null` — inject to test corrupt state guard |
| `page.route('**/api/v1/auth/login', ...)` | AC2, AC8 | Playwright `route()` mock — no MSW required |
| CDP `Emulation.setCPUThrottlingRate` | AC5 | 4× CPU throttle to expose Zustand hydration window |
| `request` fixture with `maxRedirects: 0` | AC7 | Raw HTTP requests without browser for middleware 307 assertions |

### `data-testid` Values Required in Implementation

| `data-testid` | Component | Used In AC |
|---|---|---|
| `auth-loading-spinner` | `<AuthGuard>` spinner div | AC5, AC6 |
| `protected-content` | Top-level content in `(protected)/layout.tsx` | AC1, AC4, AC5, AC6 |
| `user-avatar-menu-trigger` | Avatar button in TopBar | E03-P1-018 |

---

## Next Steps (TDD Green Phase)

After implementing the feature (all files listed above):

1. **Remove `test.skip()`** from all tests in both files
2. **Ensure apps are running:** `pnpm dev` in `eusolicit-app/frontend` (client on port 3000, admin on port 3001)
3. **Run P0 tests first:**
   ```bash
   cd eusolicit-app
   npx playwright test e2e/specs/shell/route-guards.spec.ts --grep="P0" --project=client-chromium
   ```
4. **Run full suite:**
   ```bash
   npx playwright test e2e/specs/shell/ --project=client-chromium
   npx playwright test e2e/specs/shell/route-guards.admin.spec.ts --project=admin-chromium
   ```
5. **Verify all 38 tests PASS** (green phase)
6. **If any tests fail:** Fix implementation bug (not the test) — tests define the expected contract
7. **Commit** test files alongside implementation

### Run Commands Reference

```bash
# All route guard tests (client + admin)
npx playwright test e2e/specs/shell/route-guards.spec.ts e2e/specs/shell/route-guards.admin.spec.ts

# P0 tests only (critical gate)
npx playwright test e2e/specs/shell/route-guards.spec.ts --grep="P0"

# Middleware API tests only
npx playwright test e2e/specs/shell/route-guards.spec.ts --grep="AC7"

# Full epic shell tests
npx playwright test e2e/specs/shell/
```

---

## Risk Mitigations Verified by This Suite

| Risk ID | Description | Tests That Verify Mitigation |
|---|---|---|
| **E03-R-002** (SEC, Score 6) | Route guard flash of protected content | AC5-T01, AC5-T02, AC1-T02, AC6-T02 |
| **E03-R-002** (middleware layer) | Server-side cookie check as secondary guard | AC7-T01 through AC7-T05 |

---

## Assumptions and Open Questions

1. **`data-testid="protected-content"`** — The implementation must add this testid to the wrapper div inside `(protected)/layout.tsx` (or the first element of the protected page content). Tests rely on this to assert no content is visible before auth resolves.

2. **`data-testid="user-avatar-menu-trigger"`** — The TopBar avatar button must have this testid (or the tests must be updated to use the actual selector). If the avatar trigger uses a different selector, update P1-018 tests accordingly.

3. **Login form labels** — Tests use `page.getByLabel(/email/i)` and `page.getByLabel(/password/i)`. If the login form uses different label text, update AC2/AC8 tests to match.

4. **`POST /api/v1/auth/login` route** — Tests mock this endpoint pattern. If the actual API path differs, update `mockLoginSuccess()` route pattern.

5. **Middleware test baseURL** — AC7 tests use the `request` fixture which inherits `baseURL` from the Playwright project config (`http://localhost:3000` for client projects). Confirm this matches the running Next.js client app URL.

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
