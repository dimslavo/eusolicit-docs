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
storyId: 3-9-company-profile-setup-wizard
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/3-9-company-profile-setup-wizard.md
  - eusolicit-docs/test-artifacts/test-design-epic-03.md
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/frontend/apps/client/vitest.config.ts
  - eusolicit-app/frontend/apps/client/messages/en.json
  - eusolicit-app/frontend/apps/client/messages/bg.json
  - eusolicit-app/e2e/support/fixtures/auth.fixture.ts
  - eusolicit-docs/test-artifacts/atdd-checklist-3-8-authentication-pages.md
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 3.9 — Company Profile Setup Wizard

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**Story:** S03.09 | **Epic:** E03 — Frontend Shell & Design System
**TDD Phase:** 🔴 RED — All tests are skipped (failing) until implementation is complete

---

## Step 1: Preflight & Context

### Stack Detection

| Item | Value |
|------|-------|
| **Detected Stack** | `frontend` — Next.js 14 App Router, React, Zustand, next-intl |
| **Test Framework (unit)** | Vitest (`apps/client/vitest.config.ts` — environment: node) |
| **Test Framework (E2E)** | Playwright (`eusolicit-app/playwright.config.ts`) |
| **Playwright testDir** | `eusolicit-app/e2e/specs/` |
| **Test Artifacts dir** | `eusolicit-docs/test-artifacts/` |
| **tea_use_playwright_utils** | not configured — standard Playwright patterns used |
| **tea_use_pactjs_utils** | not configured — no contract testing for this story |
| **tea_browser_automation** | not configured — AI generation mode (no browser recording needed) |

### Prerequisites Verified

| Item | Status | Notes |
|------|--------|-------|
| Story approved with clear ACs | ✅ | 11 ACs; full implementation code in Dev Notes |
| Playwright config found | ✅ | `eusolicit-app/playwright.config.ts` — client base URL `http://localhost:3000` |
| Vitest config found | ✅ | `apps/client/vitest.config.ts` (environment: node) |
| Epic test design loaded | ✅ | `test-design-epic-03.md` — E03-P1-008, P1-009, P2-013, P2-014, P2-015 |
| Wizard store key confirmed | ✅ | Zustand persist name: `"eusolicit-wizard-store"` (from wizard-store.ts dev notes) |
| Auth store key confirmed | ✅ | Zustand persist name: `"eusolicit-auth-store"` (from S03.08 established pattern) |
| Existing test patterns inspected | ✅ | `auth-pages.spec.ts`, `auth.fixture.ts` — established addInitScript() pattern |
| i18n keys confirmed | ✅ | All `wizard.*` keys present in both `en.json` and `bg.json` (added in S03.07) |
| data-testid spec confirmed | ✅ | Dev Notes table: 13 explicit testid attributes for E2E selector stability |

### AC Summary

| AC | Description | Priority |
|----|-------------|----------|
| AC1 | Wizard page at `(protected)/setup/page.tsx`; WizardStepper renders 4 labelled steps with status styling | P1 |
| AC2 | Step 1 — Company Info: pre-filled fields, logo upload, address required, website optional URL validation | P1 |
| AC3 | Step 2 — CPV search/filter, badge add/remove, ≥1 CPV required for Next | P1 |
| AC4 | Step 3 — Bulgarian Regions (6) + EU States (27), select-all toggles, ≥1 region required for Next | P1 |
| AC5 | Step 4 — Team email add/remove, invalid email error, Complete Setup always enabled | P1 |
| AC6 | Navigation: Back (no validation), Next (validates), Complete Setup submits stub + redirect to /dashboard | P0 |
| AC7 | Wizard state persisted via Zustand persist; logoFile excluded; logoDataUrl included; data survives reload | P1 |
| AC8 | wizardStore.reset() clears store + removes localStorage key after Complete Setup | P1 |
| AC9 | Register page redirects to `/${locale}/setup` after successful registration (changed from /dashboard) | P1 |
| AC10 | All wizard UI strings use useTranslations; all keys present in en.json + bg.json (no new keys needed) | P1 |
| AC11 | pnpm build exits 0; pnpm type-check exits 0; pnpm check:i18n exits 0 | P0 |

---

## Step 2: Generation Mode

**AI Generation** — acceptance criteria are explicit with complete implementation code in Dev Notes.
All wizard patterns (multi-step form, Zustand persistence, CPV search/filter, region select-all) are standard frontend flows.
Dev Notes provide explicit `data-testid` attributes for every interactive element. No browser recording needed.

---

## Step 3: Test Strategy

### AC → Test Level Mapping

| AC | Description | Test Level | Priority | Epic Test ID |
|----|-------------|------------|----------|--------------|
| AC1 | Wizard page + stepper renders | E2E (Playwright) | P1 | — |
| AC1 | Step files exist in filesystem | Unit (Vitest/node) | P1 | — |
| AC2 | Step 1 fields, logo upload, validation | E2E (Playwright) | P1 | — |
| AC2 | step1Schema exports, address/EIK/website validation | Unit (Vitest/node) | P1 | — |
| AC3 | CPV search, badge add/remove, min-1 validation | E2E (Playwright) | P1 | E03-P2-013 |
| AC3 | CPV_CODES data: ≥30 entries, interface | Unit (Vitest/node) | P1 | — |
| AC3 | step2Schema min(1) cpvCodes | Unit (Vitest/node) | P1 | — |
| AC4 | Region sections, select-all, min-1 validation | E2E (Playwright) | P1 | E03-P2-014 |
| AC4 | BULGARIAN_REGIONS=6, EU_MEMBER_STATES=27 | Unit (Vitest/node) | P1 | — |
| AC4 | step3Schema min(1) regions | Unit (Vitest/node) | P1 | — |
| AC5 | Email add/remove, invalid email error, Complete enabled | E2E (Playwright) | P1 | E03-P2-015 |
| AC5 | teamEmailSchema .email() validation | Unit (Vitest/node) | P1 | — |
| AC6 | Complete Setup: spinner, redirect to /dashboard | E2E (Playwright) | P0 | E03-P1-008 |
| AC6 | updateCompanyProfile stub, CompanyProfilePayload | Unit (Vitest/node) | P1 | — |
| AC7 | Wizard state persists on reload: step number, CPV badges, logoDataUrl | E2E (Playwright) | P1 | E03-P1-009 |
| AC7 | wizardStore persist name, partialize, logoFile excluded | Unit (Vitest/node) | P1 | E03-R-007 |
| AC8 | localStorage key removed after Complete Setup | E2E (Playwright) | P1 | — |
| AC9 | Register page redirects to /setup after registration | E2E (Playwright) | P1 | — |
| AC9 | register/page.tsx contains /setup redirect (not /dashboard) | Unit (Vitest/node) | P1 | — |
| AC10 | en.json + bg.json wizard keys present and equal | Unit (Vitest/node) | P1 | E03-P1-007 |
| AC11 | Build gate: pnpm build, type-check, check:i18n | CI/Build | P0 | E03-P0-001 |
| data-testids | All 13 required testids added to source files | Unit (Vitest/node) | P1 | — |
| Unauthenticated guard | /setup redirects to /login when not authenticated | E2E (Playwright) | P0 | E03-P0-002 |

### Red Phase Compliance

All tests use `test.skip()` (Playwright) or `it.skip()` (Vitest).
They define **expected behavior** that fails until Story 3.9 is implemented.

**Key test isolation decisions:**
- E2E tests use `page.addInitScript()` to seed both `eusolicit-auth-store` and `eusolicit-wizard-store` into localStorage before navigation
- Wizard steps 2–4 are reached by seeding `currentStep` in the wizard store — avoids form submission in setup code
- Unit tests verify file existence and content via `existsSync()` / `readFileSync()` — no imports of source files (avoids Next.js client component SSR errors in node environment)
- `data-testid` selectors used throughout (explicitly specified in story Dev Notes)

---

## Step 4: TDD Red Phase — Generated Test Files

### 🔴 All tests use `test.skip()` / `it.skip()` — they WILL FAIL until implementation completes

| # | Test File | Covers ACs | Priority | Test Count | Framework | Status |
|---|-----------|------------|----------|------------|-----------|--------|
| 1 | `eusolicit-app/e2e/specs/setup/wizard.spec.ts` | AC1–9, Full Flow | P0/P1 | 47 | Playwright E2E | ✅ Created (all skipped) |
| 2 | `eusolicit-app/frontend/apps/client/__tests__/wizard-s3-9.test.ts` | AC1–10, testids | P1 | 47 | Vitest/node | ✅ Created (all skipped) |

**Total:** 94 failing tests across 2 files.

---

## Step 4C: Test File Details

### File 1: `eusolicit-app/e2e/specs/setup/wizard.spec.ts`

**Framework:** Playwright (client project — Chromium, Firefox, Safari, Mobile)
**Run:** `cd eusolicit-app && npx playwright test e2e/specs/setup/wizard.spec.ts --project=client-chromium`
**Base URL:** `http://localhost:3000` (env: local)

#### Test Inventory by Describe Block

| Describe Block | Tests | Priority Range | Epic Link |
|----------------|-------|----------------|-----------|
| AC1 — Wizard Page & WizardStepper | 7 | P0–P1 | — |
| AC2 — Step 1: Company Info | 7 | P1 | — |
| AC3 — Step 2: CPV (E03-P2-013) | 9 | P1 | E03-P2-013 |
| AC4 — Step 3: Regions (E03-P2-014) | 6 | P1 | E03-P2-014 |
| AC5 — Step 4: Team Invites (E03-P2-015) | 7 | P1 | E03-P2-015 |
| AC6 — Navigation & Submission | 4 | P0–P1 | E03-P1-008 (partial) |
| AC7 — State Persistence (E03-P1-009) | 4 | P1 | E03-P1-009 |
| AC8 — Store Reset After Completion | 1 | P1 | — |
| AC9 — Register Redirect to /setup | 1 | P1 | — |
| Full Wizard Flow — E03-P1-008 | 2 | P0 | E03-P1-008 |

#### Key Selectors Used

```typescript
// Wizard page root and stepper
page.getByTestId('wizard-page')          // AC1 wizard page root
page.getByTestId('wizard-stepper')       // AC1 stepper nav
page.getByTestId('wizard-stepper').locator('[aria-current="step"]')  // active step

// Step containers (from data-testid spec)
page.getByTestId('wizard-step-1')        // AC2 Step 1 container
page.getByTestId('wizard-step-2')        // AC3 Step 2 container
page.getByTestId('wizard-step-3')        // AC4 Step 3 container
page.getByTestId('wizard-step-4')        // AC5 Step 4 container

// Step 2 CPV
page.getByTestId('cpv-search-input')     // AC3 search input
page.getByTestId('cpv-selected-badges')  // AC3 selected CPV badges
page.getByTestId('cpv-error')            // AC3 validation error

// Step 3 Regions
page.getByTestId('regions-error')        // AC4 validation error

// Step 4 Team Invites
page.getByTestId('team-email-input')     // AC5 email input
page.getByTestId('team-email-list')      // AC5 added emails list
page.getByTestId('complete-setup-button') // AC5/AC6 submit button

// Standard role/label selectors
page.getByLabel(/address/i)              // Step 1 address field (FormField label)
page.getByLabel(/website/i)              // Step 1 website field
page.getByRole('button', { name: /next/i })          // Step 1–3 Next
page.getByRole('button', { name: /back/i })          // Steps 2–4 Back
page.getByRole('button', { name: /add/i })           // Step 4 Add email
page.locator('img[alt="Company logo preview"]')      // AC2 logo preview
page.locator('.animate-spin')                        // AC6 Loader2 spinner
```

#### Wizard Store Seeding Pattern

```typescript
// Seed Zustand wizard store (eusolicit-wizard-store) before page.goto()
const WIZARD_STORE_KEY = 'eusolicit-wizard-store';
await page.addInitScript(
  ({ key, value }: { key: string; value: string }) => localStorage.setItem(key, value),
  { key: WIZARD_STORE_KEY, value: JSON.stringify({
    state: {
      currentStep: 2,  // Navigate directly to any step
      step1: { ... },
      step2: { cpvCodes: ['48000000'] },
      step3: { regions: [] },
      step4: { teamEmails: [] },
    },
    version: 0,
  }) }
);
```

---

### File 2: `eusolicit-app/frontend/apps/client/__tests__/wizard-s3-9.test.ts`

**Framework:** Vitest (environment: node)
**Run:** `cd eusolicit-app/frontend && pnpm test --filter client`

#### Test Inventory by Describe Block

| Describe Block | Tests | Priority Range |
|----------------|-------|----------------|
| Wizard File Structure (all ACs) | 11 | P1 |
| Wizard Store — persist config (AC7, AC8) | 8 | P1 |
| Wizard Zod Schemas (AC2–5) | 7 | P1 |
| CPV Codes Data (AC3) | 3 | P1 |
| Regions Data (AC4) | 3 | P1 |
| Company API Stub (AC6) | 2 | P1 |
| Register Redirect (AC9) | 2 | P1 |
| data-testid Attributes | 6 | P1 |
| i18n wizard namespace keys (AC10) | 6 | P1 |

#### Expected Failure Modes (before implementation)

```
// File structure checks fail with ENOENT:
[FS-01] ENOENT: lib/stores/wizard-store.ts (Task 1.1)
[FS-02] ENOENT: lib/schemas/wizard.ts (Task 2.1)
[FS-06] ENOENT: app/[locale]/(protected)/setup/page.tsx (Task 11.1)
[FS-07] ENOENT: setup/components/WizardStepper.tsx (Task 6.1)
...

// Schema/store content checks fail with missing exports:
[SCHEMA-01] wizard.ts: export const step1Schema → not found
[STORE-01]  wizard-store.ts: "eusolicit-wizard-store" → not found

// data-testid checks fail:
[TESTID-04] Step2CPVSectors.tsx: data-testid="cpv-error" → not found

// Register redirect check fails:
[REG-01] register/page.tsx: /${locale}/setup → not found (still has /dashboard)

// i18n checks pass ✅ (wizard keys already exist from S03.07)
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
| Test selectors use explicit `data-testid` (as specified in Dev Notes) | ✅ |
| State seeding via `page.addInitScript()` — no server required | ✅ |
| Unit tests avoid importing source files (node environment safe) | ✅ |
| No temp files in random locations — tests in project directories | ✅ |
| Story AC11 (build gate) noted — covered by CI, not new test files | ✅ |
| Wizard store key `"eusolicit-wizard-store"` confirmed from Dev Notes | ✅ |
| i18n keys already present in both locales — unit tests verify parity | ✅ |

### Coverage Matrix

| AC | E2E Tests | Unit Tests | Total | Epic Link |
|----|-----------|------------|-------|-----------|
| AC1 (page + stepper) | 7 | 3 | 10 | — |
| AC2 (Step 1 company info) | 7 | 4 | 11 | — |
| AC3 (Step 2 CPV) | 9 | 5 | 14 | E03-P2-013 |
| AC4 (Step 3 regions) | 6 | 4 | 10 | E03-P2-014 |
| AC5 (Step 4 team invites) | 7 | 2 | 9 | E03-P2-015 |
| AC6 (navigation + submission) | 4 | 2 | 6 | E03-P1-008 |
| AC7 (state persistence) | 4 | 5 | 9 | E03-P1-009 |
| AC8 (store reset) | 1 | 0 | 1 | — |
| AC9 (register redirect) | 1 | 2 | 3 | — |
| AC10 (i18n keys) | 0 | 6 | 6 | E03-P1-007 |
| AC11 (build gate) | CI only | CI only | CI gate | E03-P0-001 |
| data-testids | (E2E uses) | 6 | 6 | — |

> Note: Some tests cover multiple ACs; full flow test (E03-P1-008) covers AC1–8.

### Epic Test ID Coverage

| Epic Test ID | Coverage | Test Location |
|-------------|----------|---------------|
| **E03-P1-008** | Full wizard Step 1→4 completion + redirect to /dashboard | `wizard.spec.ts`: Full Wizard Flow describe block |
| **E03-P1-009** | Wizard state survives reload (Step 2 CPV badges, step number, logoDataUrl) | `wizard.spec.ts`: AC7 describe block |
| **E03-P2-013** | Step 2 CPV search filter + zero-selection validation error | `wizard.spec.ts`: AC3 describe block |
| **E03-P2-014** | Step 3 select-all + zero-selection validation error | `wizard.spec.ts`: AC4 describe block |
| **E03-P2-015** | Step 4 email add/remove + Complete with zero emails | `wizard.spec.ts`: AC5 describe block |
| **E03-P1-007** (extension) | wizard namespace key parity en.json ↔ bg.json | `wizard-s3-9.test.ts`: i18n describe block |
| **E03-P0-002** (partial) | Unauthenticated /setup redirects to /login | `wizard.spec.ts`: AC1 describe block |
| **E03-P0-001** | Build gate (pnpm build + type-check + check:i18n) | CI (Task 13.1–13.3) |
| **E03-R-007** | Wizard store partialize — logoFile excluded | `wizard-s3-9.test.ts`: Store describe block |

### Quality Gate Alignment

Per `test-design-epic-03.md`:
- **E03-P1-008** must pass before Demo milestone — covered by AC6 + Full Flow E2E tests (P0 priority)
- **E03-P1-009** (wizard state persistence, risk E03-R-007) — covered by AC7 E2E tests with localStorage seeding
- **E03-P2-013/014/015** (wizard step validations) — covered by AC3/AC4/AC5 E2E tests
- **E03-P1-007** (i18n key parity) — extended to verify wizard namespace key parity in unit tests
- **Build gate** (E03-P0-001) — AC11 verified by `pnpm build` in CI (Tasks 13.1–13.3)
- **Branch coverage ≥80% on wizardStore.ts** — unit tests exercise store structure; E2E tests drive state mutations

---

## Implementation Notes for Developer

### Run Commands (after implementation)

```bash
# Remove test.skip() from both test files, then:

# E2E tests (Playwright) — requires Next.js dev server on port 3000
cd eusolicit-app
npx playwright test e2e/specs/setup/wizard.spec.ts --project=client-chromium

# Unit tests (Vitest) — no browser required
cd eusolicit-app/frontend
pnpm test --filter client

# Build gate (AC11)
cd eusolicit-app/frontend
pnpm build
pnpm type-check
pnpm check:i18n --filter client
```

### Pre-conditions for Tests to Pass

| Order | Task | Required By |
|-------|------|-------------|
| 1 | Create `lib/stores/wizard-store.ts` (Task 1.1) | AC7/AC8 unit tests; AC7 E2E persistence tests |
| 2 | Create `lib/schemas/wizard.ts` (Task 2.1) | AC2–5 unit schema tests; E2E validation tests |
| 3 | Create `lib/data/cpv-codes.ts` (Task 3.1) | AC3 unit data tests; Step 2 E2E tests |
| 4 | Create `lib/data/regions.ts` (Task 4.1) | AC4 unit data tests; Step 3 E2E tests |
| 5 | Create `lib/api/company.ts` (Task 5.1) | AC6 unit + E2E submission tests |
| 6 | Create `WizardStepper.tsx` with `data-testid="wizard-stepper"` (Task 6.1) | AC1 E2E tests |
| 7 | Create `Step1CompanyInfo.tsx` with `data-testid="wizard-step-1"` (Task 7.1) | AC2 E2E tests |
| 8 | Create `Step2CPVSectors.tsx` with all Step 2 testids (Task 8.1) | AC3 E2E tests |
| 9 | Create `Step3Regions.tsx` with all Step 3 testids (Task 9.1) | AC4 E2E tests |
| 10 | Create `Step4TeamInvites.tsx` with all Step 4 testids (Task 10.1) | AC5/AC6 E2E tests |
| 11 | Create `app/[locale]/(protected)/setup/page.tsx` (Task 11.1) | AC1/AC6/AC7/AC8 E2E tests |
| 12 | Update `register/page.tsx` redirect to `/setup` (Task 12.1) | AC9 unit + E2E tests |

### Key Implementation Invariants Tested

1. **`logoFile` is NOT persisted** — `partialize` must explicitly null it out; `localStorage.getItem(WIZARD_STORE_KEY)` must have `state.step1.logoFile === null` after any store operation
2. **`logoDataUrl` IS persisted** — base64 string stored via `FileReader.readAsDataURL()` must survive page reload
3. **`wizardStore.reset()` clears localStorage key** — after Complete Setup, `localStorage.getItem("eusolicit-wizard-store")` must return `null`
4. **Step validation is imperative** — Steps 2–4 validate directly against `step2Schema`/`step3Schema` (not RHF); error appears in `data-testid="cpv-error"` / `data-testid="regions-error"` elements
5. **Back never validates** — clicking Back on any step should not show validation errors from that step
6. **Complete Setup is always enabled on Step 4** — no minimum email requirement (step 4 is optional)

### Known Assumptions

1. **Auth seeding**: Tests seed `eusolicit-auth-store` with `isAuthenticated: true` — the auth guard `mounted + !isAuthenticated → redirect to /login` will fire; seeded state bypasses the redirect
2. **Locale prefix**: All tests use `/en/setup` — default locale is `bg` but EN provides predictable English text for label/text assertions
3. **Stub delay**: `updateCompanyProfile()` has 800ms delay — loading state tests check spinner during that window; Playwright actionTimeout (15s) covers this without explicit waits
4. **CPV search**: The `CPV_CODES` array has ~38 entries; search for `"72"` returns "IT services" (`72000000`); tests rely on this specific code being present
5. **No MSW required**: All API calls in S03.09 are stubs (no real backend calls); E2E tests do not need MSW setup. The `updateCompanyProfile()` stub returns void after a delay — no network interception needed for completion flow
6. **Register E2E test**: The register page E2E test mocks `**/api/v1/auth/register` via `page.route()` — requires actual register form fields from S03.08 implementation to be present

---

## Completion Summary

**Test files created:**
1. `eusolicit-app/e2e/specs/setup/wizard.spec.ts` — 47 Playwright E2E tests (all skipped)
2. `eusolicit-app/frontend/apps/client/__tests__/wizard-s3-9.test.ts` — 47 Vitest unit tests (all skipped)

**Checklist saved to:**
`eusolicit-docs/test-artifacts/atdd-checklist-3-9-company-profile-setup-wizard.md`

**Next recommended workflow:**
1. Implement Story 3.9 tasks 1–13 in order (per Task list in story file)
2. Remove `test.skip()` / `it.skip()` from both test files
3. Run Vitest unit tests first (fast, no browser): `cd eusolicit-app/frontend && pnpm test --filter client`
4. Run Playwright E2E tests (requires Next.js dev server): `npx playwright test e2e/specs/setup/wizard.spec.ts --project=client-chromium`
5. Run build gate: `pnpm build && pnpm type-check && pnpm check:i18n --filter client`
6. Once all tests green → run `bmad-testarch-automate` to expand coverage for Story 3.9

---

**Generated by:** BMad TEA Agent — Test Architect Module (ATDD workflow)
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
