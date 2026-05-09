---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-identify-targets
  - step-03-generate-tests
  - step-04-validate-and-summarize
lastStep: step-04-validate-and-summarize
lastSaved: '2026-05-09'
workflowType: bmad-testarch-automate
mode: bmad-integrated
scope: cluster-A-batch-e
batch: P2-e
storyKeys:
  - 3-9-company-profile-setup-wizard
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/e2e/specs/setup/wizard.spec.ts
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/frontend/apps/client/middleware.ts
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/setup/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/setup/components/WizardStepper.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/setup/components/Step1CompanyInfo.tsx
  - eusolicit-app/frontend/apps/client/lib/stores/wizard-store.ts
  - eusolicit-app/frontend/packages/ui/src/lib/stores/auth-store.ts
---

# Automation Summary: P2-e — setup/wizard

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P2-e (cluster A, sub-run 5 of 5 — **closes cluster A**)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Triage

Same shape as P2-d: Story 3.9 was implemented to spec — testids, store keys,
i18n, schema messages all match the spec contract verbatim. Three precision
fixes were enough to flip all 50 tests from RED skip to active.

**Test count:** 50 active, 0 fixme.

---

## Files Modified

### `e2e/specs/setup/wizard.spec.ts`

**Three precision fixes:**

1. **Dashboard redirect URL regex** — `setup/page.tsx:105` redirects to
   `/${locale}/workspace/default/dashboard` after Complete Setup, not bare
   `/${locale}/dashboard`. Updated:

   ```ts
   const DASHBOARD_RE = new RegExp(`/${LOCALE}/(workspace/[^/]+/)?dashboard`);
   ```

2. **Cookie seeding in `seedAuth`** — `middleware.ts:23-32` lists `/setup` as a
   PROTECTED_PATH, so the edge middleware 307-redirects to `/{locale}/login`
   when the `eusolicit-session` cookie is absent. Pure localStorage seeding
   wouldn't pass this gate. Updated `seedAuth` to also drop a stub session
   cookie via `page.context().addCookies(...)` before navigation. The
   middleware accepts any non-empty cookie value (no signature check at the
   edge — auth is JWT-validated server-side per route).

3. **`test.skip(` → `test(` mass conversion** — all 50 declarations converted
   to active definitions. No body changes.

   The unauthenticated-redirect test (line 187) deliberately does NOT call
   `seedAuth`, so my cookie addition there has no side effect — it remains the
   only test that exercises the no-cookie middleware redirect path.

**Header docstring** updated 🔴→🟢 with the actual production paths
(`wizard-store.ts:95` for the persist key, `setup/page.tsx:105` for redirect,
`setup/page.tsx:42` for unauth redirect).

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| Route `/en/setup` exists | `app/[locale]/(protected)/setup/page.tsx` | ✅ |
| `data-testid="wizard-page"` on container | `setup/page.tsx:114` | ✅ |
| `data-testid="wizard-stepper"` on `<nav>` | `WizardStepper.tsx:17` | ✅ |
| `data-testid="wizard-step-1"` | `Step1CompanyInfo.tsx:72` | ✅ |
| Wizard persist key `eusolicit-wizard-store` | `wizard-store.ts:95` | ✅ |
| Auth-store v1→v2 migration on load | `auth-store.ts:38-64` | ✅ |
| Setup completion redirect to dashboard | `setup/page.tsx:105` (`/${locale}/workspace/default/dashboard`) | ✅ regex now matches |
| Unauthenticated → `/login` | `setup/page.tsx:42` (page-level) + `middleware.ts:65-72` (edge) | ✅ |

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for `setup/wizard.spec.ts` | ✅ 0 errors |
| `npx playwright test --list` | ✅ **50 tests collected** in `e2e/specs/setup/wizard.spec.ts` |
| Active vs fixme split | 50 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Pre-existing TS errors (not introduced)

`tsc` reported 7 errors in a separate file: `e2e/specs/onboarding/onboarding-wizard.spec.ts` (different from `e2e/specs/setup/wizard.spec.ts`). These are pre-existing — `onboarding-wizard.spec.ts` has untyped `req` params and `Page.page` access that don't exist. Out of scope for P2-e.

### Risks / open questions for the CI pass

1. **Stub session cookie strength**: `seedAuth` drops `eusolicit-session=stub-session-cookie` to satisfy the edge middleware. If the wizard page or any of its API calls (Step 1 submit, Step 4 invites POST) further validate the JWT signature server-side, the stub will fail those calls. Tests that only verify the wizard surface (rendering, stepper, validation, store reset) will still work. Tests that exercise full API submission (Full Wizard Flow E03-P1-008) may need the `authenticatedPage` fixture from `support/fixtures` to mint a real JWT via `/api/v1/auth/test-login` instead. If CI shows submission tests failing with 401, swap `page` → `authenticatedPage` for those tests.

2. **Auth-store migration timing**: same concern as P2-d — `addInitScript` runs before module IIFEs, so the v1→v2 migration should run after the script seeds localStorage. Confirmed flow:
   - `addInitScript` runs first → writes `eusolicit-auth-store` to localStorage.
   - Page bundle loads → `auth-store.ts` IIFE runs → migration sees v1 entry, writes to v2.
   - React mounts → `useAuthStore` reads from v2 → `isAuthenticated: true`.

3. **Wizard reload-persistence test (E03-P1-009)**: the spec at line 605-ish reloads the page mid-wizard and expects state to restore. Since the persist middleware writes to `eusolicit-wizard-store`, this should round-trip correctly. The test seeds an empty wizard state at first, then types into Step 1, reloads, and asserts the typed values are there — assumes Zustand persist debounce isn't longer than the test's interaction window. If flaky, add a small `await page.waitForTimeout(100)` before `page.reload()`.

4. **Register redirect to /setup (AC9)**: line 670 asserts that successful registration takes the new user to `/setup` instead of `/dashboard`. This is a Story 3.9 Task 12 contract. If the registration page in production redirects to `/dashboard` (or the workspace-prefixed dashboard) for new users without checking onboarding status, this test fails — and the fix lives in the register page, not the test.

---

## Coverage Delta (P2-e)

- Before P2-e: 0 active + 50 truly-skipped tests in `setup/wizard.spec.ts`.
- After P2-e: **50 active** + 0 fixme.

---

## 🎉 Cluster A Closed — Cumulative Totals

P2-a (proposal_generation + billing-vat) — 5 active + 1 fixme
P2-b (billing-checkout + accept-link-flow) — 20 active + 7 fixme
P2-c (proposals-workspace + trust/public-access) — 27 active + 6 fixme
P2-d (auth-pages) — 48 active + 0 fixme
P2-e (setup/wizard) — 50 active + 0 fixme

**Total cluster A: 150 newly active tests across 7 spec files; 14 fixme entries with concrete re-author docs.**

Files touched:
- `e2e/proposal_generation.spec.ts` (1 fixme — superseded scaffolding)
- `e2e/specs/billing-vat.spec.ts` (5 active)
- `e2e/specs/billing-checkout.spec.ts` (8 active + 3 fixme)
- `e2e/specs/external-collaborator/accept-link-flow.spec.ts` (12 active + 4 fixme)
- `e2e/specs/proposals/proposals-workspace.spec.ts` (8 active + 3 fixme)
- `e2e/specs/trust/public-access.spec.ts` (19 active + 3 fixme)
- `e2e/specs/auth/auth-pages.spec.ts` (48 active + 0 fixme)
- `e2e/specs/setup/wizard.spec.ts` (50 active + 0 fixme)

Per-batch summaries: `automation-summary-cluster-a-batch-{a..e}.md` under
`eusolicit-docs/test-artifacts/`.

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **Cluster A complete.** Recommended next step before starting cluster C / P3
  is a CI dry-run of all 7 P2 spec files against a properly-provisioned host
  (chromium system libs installed) so the un-skipped tests can burn-in 3× green
  before being considered locked.
- **P3-a** (cluster C) — new E2E specs for Tasks/Kanban surface (highest DAU
  surface with no E2E coverage). Smallest first sub-run.

After P2-e CI runs, update the rollout-plan checklist line and mark cluster A
fully shipped.
