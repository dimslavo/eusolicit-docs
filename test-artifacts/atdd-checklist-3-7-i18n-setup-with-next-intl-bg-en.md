---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-08'
workflowType: 'testarch-atdd'
storyId: '3-7-i18n-setup-with-next-intl-bg-en'
tddPhase: 'RED'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/3-7-i18n-setup-with-next-intl-bg-en.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-03.md'
  - 'eusolicit-app/frontend/apps/client/vitest.config.ts'
  - 'eusolicit-app/frontend/packages/ui/vitest.config.ts'
  - 'eusolicit-app/frontend/packages/ui/src/__tests__/components/forms/FormField.test.tsx'
  - 'eusolicit-app/frontend/apps/client/lib/stores/__tests__/ui-store.test.ts'
  - 'eusolicit-app/frontend/packages/ui/src/components/app-shell/TopBar.tsx'
  - 'eusolicit-app/frontend/packages/ui/src/components/app-shell/UserAvatarMenu.tsx'
  - 'eusolicit-app/frontend/packages/ui/index.ts'
  - '_bmad/bmm/config.yaml'
---

# ATDD Checklist: Story 3.7 — i18n Setup with next-intl (BG + EN)

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**Story:** S03.07 | **Epic:** E03 — Frontend Shell & Design System
**TDD Phase:** 🔴 RED — All tests are skipped (failing) until implementation is complete

---

## Preflight Summary

### Stack Detection
- **Detected Stack:** `frontend` (Next.js 14, React, Zustand, TanStack Query)
- **Test Framework:** Vitest (`apps/client` + `packages/ui` both configured)
- **Component Testing:** Vitest + @testing-library/react (jsdom — `packages/ui`)
- **Playwright:** ⚠️ No `playwright.config.*` found — E2E locale-redirect tests (E03-P0-005, E03-P0-006 full flow) cannot run until Playwright is configured. Unit and component coverage substitutes for the red phase.

### Admin App Note
`apps/admin` does **not** have Vitest configured. Admin test file generated at
`apps/admin/scripts/__tests__/check-i18n-keys.test.ts` — requires:
1. Add `vitest` to `apps/admin/devDependencies`
2. Add `"test": "vitest run"` to `apps/admin/package.json` scripts
3. Create `apps/admin/vitest.config.ts` (copy from `apps/client`)

---

## Generation Mode
**AI Generation** — acceptance criteria are well-defined; story is a configuration and utility story with no complex UI flows requiring live browser recording.

---

## Test Strategy

| AC | Description | Test Level | Priority | Epic Test ID |
|----|-------------|------------|----------|--------------|
| AC1 | next-intl installed; next.config.mjs wraps with createNextIntlPlugin | File-read unit | P1 | E03-P0-001 (build) |
| AC2 | middleware.ts handles locale detection (locales, defaultLocale, localePrefix) | File-read unit | P1 | E03-R-001 mitigation |
| AC3 | app/[locale]/ route structure; NextIntlClientProvider + QueryProvider in locale layout | Build/CI | P1 | E03-P0-005 (indirect) |
| AC4 | messages/bg.json + en.json with all 6 namespaces; identical key sets | Unit (file read) | P1 | E03-P1-007 |
| AC5 | Shell chrome uses useTranslations(); UserAvatarMenu labels prop; TopBar aria labels | Component | P1 | E03-P2-004 |
| AC6 | TopBar: globe icon + locale code; clicking inactive locale calls onLocaleChange | Component | P0 | E03-P0-006 |
| AC7 | Locale persisted: setLocale action writes uiStore.locale to localStorage | Unit | P1 | E03-P1-006 |
| AC8 | formatDate + formatCurrency in packages/ui/src/lib/format.ts; exported from index.ts | Unit | P2 | E03-P2-011 |
| AC9 | check-i18n-keys.mjs exits 0 on match, non-zero on mismatch; pnpm check:i18n | Unit (exec) | P1 | E03-P1-007 |
| AC10 | pnpm build exits 0; pnpm type-check exits 0; / redirects to /bg/ ≤1 redirect | Build/CI | P0 | E03-P0-001, E03-P0-005 |

---

## TDD Red Phase — Generated Test Files

### 🔴 All tests use `it.skip()` — they WILL FAIL until the story is implemented

| # | Test File | Covers ACs | Priority | Test Count | Framework | Verified |
|---|-----------|------------|----------|------------|-----------|----------|
| 1 | `apps/client/__tests__/i18n-setup.test.ts` | AC1, AC2 | P1 | 11 | Vitest/node | ✅ 11 skipped |
| 2 | `apps/client/scripts/__tests__/check-i18n-keys.test.ts` | AC4, AC9 | P1 | 14 | Vitest/node | ✅ 14 skipped |
| 3 | `apps/admin/scripts/__tests__/check-i18n-keys.test.ts` | AC4, AC9 | P1 | 10 | Vitest/node | ⚠️ needs Vitest in admin |
| 4 | `apps/client/lib/stores/__tests__/ui-store-s3-7.test.ts` | AC7 | P1 | 5 | Vitest/node | ✅ 5 skipped |
| 5 | `packages/ui/src/__tests__/components/app-shell/TopBar-i18n.test.tsx` | AC5, AC6 | P0 | 7 | Vitest/jsdom | ✅ 7 skipped |
| 6 | `packages/ui/src/__tests__/components/app-shell/UserAvatarMenu-i18n.test.tsx` | AC5 | P1 | 4 | Vitest/jsdom | ✅ 4 skipped |
| 7 | `packages/ui/src/__tests__/lib/format.test.ts` | AC8 | P2 | 10 | Vitest/node | ✅ 10 skipped |

⚠️ = requires Vitest setup in apps/admin before tests can run

**Total: 61 tests — 51 verified skipped (TDD red), 10 pending admin Vitest setup**
**Existing tests: 49 passing (0 regressions introduced)**

---

## Acceptance Criteria Coverage

### AC1: next-intl installed; next.config.mjs wraps with createNextIntlPlugin('./i18n.ts')
- ✅ `[P1-T01]` package.json lists next-intl in dependencies (not devDependencies)
- ✅ `[P1-T02]` next-intl version is ^3.26.0 or higher
- ✅ `[P1-T03]` i18n.ts exists at apps/client/i18n.ts
- ✅ `[P1-T04]` i18n.ts contains getRequestConfig from next-intl/server
- ✅ `[P1-T05]` next.config.mjs contains createNextIntlPlugin import
- ✅ `[P1-T06]` next.config.mjs wraps config with withNextIntl("./i18n.ts")
- 📝 *Admin equivalent: verify same in apps/admin (manual check or extend test)*

### AC2: middleware.ts handles locale detection
- ✅ `[P1-T07]` middleware.ts exists at apps/client/middleware.ts
- ✅ `[P1-T08]` middleware.ts configures locales ["bg", "en"]
- ✅ `[P1-T09]` middleware.ts sets defaultLocale to "bg"
- ✅ `[P1-T10]` middleware.ts sets localePrefix to "always"
- ✅ `[P1-T11]` middleware.ts matcher excludes _next, api, and static asset paths
- 📝 *Admin equivalent: verify same in apps/admin (manual check or extend test)*

### AC3: app/[locale]/ route structure; layouts correct
- 📝 *Build-level verification: `pnpm build` + route structure confirmed by file-system review*
- 📝 *No unit test generated — verified via AC10 build gate*

### AC4: Message files with all namespaces and identical key sets
- ✅ `[P1-T01..T04]` Client bg.json: existence, parsing, namespaces, nav keys
- ✅ `[P1-T05..T08]` Client en.json: existence, parsing, namespaces, nav keys
- ✅ `[P1-T09..T11]` Client BG/EN parity: identical flattened key sets
- ✅ `[P1-T01..T03]` Admin bg.json: existence, namespaces, admin nav keys ⚠️
- ✅ `[P1-T04..T06]` Admin en.json: existence, namespaces, admin nav keys ⚠️
- ✅ `[P1-T07]` Admin BG/EN parity ⚠️

### AC5: Shell chrome strings translated; UserAvatarMenu labels prop; TopBar aria labels
- ✅ `[P0-T01]` TopBar accepts locale and onLocaleChange props
- ✅ `[P1-T07]` TopBar accepts languageSelectorAriaLabel prop (translated aria label)
- ✅ `[P1-T01]` UserAvatarMenu accepts optional labels prop
- ✅ `[P1-T02]` UserAvatarMenu shows BG labels when labels prop provided
- ✅ `[P1-T03]` UserAvatarMenu falls back to English defaults without labels prop
- ✅ `[P1-T04]` UserAvatarMenu accepts onSignOut prop and calls it on click
- 📝 *Nav labels: useTranslations('nav') verified via AC4 key completeness + build gate*

### AC6: TopBar language selector — globe icon, locale codes, onLocaleChange
- ✅ `[P0-T01]` TopBar renders with locale + onLocaleChange props
- ✅ `[P0-T02]` locale="bg" → selector shows "BG"
- ✅ `[P0-T03]` locale="en" → selector shows "EN"
- ✅ `[P0-T04]` Globe icon rendered (data-testid="locale-globe-icon")
- ✅ `[P0-T05]` Clicking inactive EN button calls onLocaleChange("en")
- ✅ `[P0-T06]` Clicking inactive BG button calls onLocaleChange("bg")

### AC7: Locale preference persisted via uiStore + localStorage
- ✅ `[P1-T01]` setLocale action exists on uiStore
- ✅ `[P1-T02]` setLocale("en") updates locale to "en"
- ✅ `[P1-T03]` setLocale("bg") restores default locale
- ✅ `[P1-T04]` setLocale persists new locale to localStorage
- ✅ `[P1-T05]` Locale persists across simulated page reload (rehydration from localStorage)

### AC8: formatDate and formatCurrency utilities in packages/ui
- ✅ `[P2-T01]` formatDate(date, "bg") returns non-empty string
- ✅ `[P2-T02]` formatDate BG result contains year 2026
- ✅ `[P2-T03]` formatDate EN result contains year 2026
- ✅ `[P2-T04]` formatDate BG ≠ EN for same date (locale-aware)
- ✅ `[P2-T05]` formatDate exported from packages/ui/index.ts
- ✅ `[P2-T06]` formatCurrency(1234.56, "BGN", "bg") returns non-empty string
- ✅ `[P2-T07]` formatCurrency BG uses comma decimal separator
- ✅ `[P2-T08]` formatCurrency EN uses period decimal separator
- ✅ `[P2-T09]` formatCurrency BG ≠ EN for same amount
- ✅ `[P2-T10]` formatCurrency exported from packages/ui/index.ts

### AC9: check-i18n-keys.mjs script exits 0 on match, non-zero on mismatch
- ✅ `[P1-T12]` scripts/check-i18n-keys.mjs exists (client)
- ✅ `[P1-T13]` package.json has "check:i18n" script entry (client)
- ✅ `[P1-T14]` pnpm check:i18n exits 0 when keys match (client)
- ✅ `[P1-T08]` scripts/check-i18n-keys.mjs exists (admin) ⚠️
- ✅ `[P1-T09]` package.json has "check:i18n" script entry (admin) ⚠️
- ✅ `[P1-T10]` pnpm check:i18n exits 0 when admin keys match ⚠️

### AC10: pnpm build exits 0; / redirects to /bg/ (≤1 redirect)
- 📝 *CI/Build gate: `pnpm build` both apps — E03-P0-001 (hard gate)*
- 📝 *`pnpm type-check` across all packages — E03-P0-001*
- 📝 *Redirect test: requires Playwright E2E (E03-P0-005) — add after Playwright config set up*
- 📝 *No unit test for redirect: Next.js middleware redirect behavior requires HTTP-level testing*

---

## Fixture Needs

| Fixture | Needed By | Status |
|---------|-----------|--------|
| `localStorageMock` | ui-store-s3-7.test.ts | ✅ Inline (pattern from S3.5 tests) |
| Test user object `{ name, email }` | UserAvatarMenu-i18n.test.tsx | ✅ Inline |
| BG/EN label objects | UserAvatarMenu-i18n.test.tsx | ✅ Inline |
| `vi.fn()` for onLocaleChange | TopBar-i18n.test.tsx | ✅ Vitest built-in |

---

## Implementation Guidance

### Components to modify (Task 9):
```
packages/ui/src/components/app-shell/TopBar.tsx
  + locale: 'bg' | 'en'                          (new required prop)
  + onLocaleChange: (locale: 'bg' | 'en') => void (new required prop)
  + languageSelectorAriaLabel?: string            (optional, for translations)
  + Globe icon from lucide-react with data-testid="locale-globe-icon"
  + Two locale buttons: data-testid="locale-switch-bg" / "locale-switch-en"
  + Active locale displayed as text; clicking inactive calls onLocaleChange

packages/ui/src/components/app-shell/UserAvatarMenu.tsx
  + labels?: { profile: string; settings: string; signOut: string }  (optional)
  + onSignOut?: () => void                        (optional)
  + Use labels.profile / labels.settings / labels.signOut in menu items
  + Fallback: English defaults when labels not provided
```

### Store action to add (Task 12):
```
apps/client/lib/stores/ui-store.ts
  + setLocale: (locale: 'bg' | 'en') => void  → set({ locale })

apps/admin/lib/stores/ui-store.ts (if not already identical)
  + Same setLocale action
```

### Files to create (Tasks 1–13):
```
apps/client/i18n.ts                         (Task 2.1)
apps/client/middleware.ts                   (Task 3.1)
apps/client/messages/bg.json               (Task 5.1)
apps/client/messages/en.json               (Task 5.2)
apps/client/scripts/check-i18n-keys.mjs    (Task 11.1)
apps/client/components/LocaleSyncer.tsx    (Task 12.1)
apps/admin/i18n.ts                          (Task 2.2)
apps/admin/middleware.ts                    (Task 3.2)
apps/admin/messages/bg.json                (Task 6.1)
apps/admin/messages/en.json                (Task 6.2)
apps/admin/scripts/check-i18n-keys.mjs     (Task 11.3)
apps/admin/components/LocaleSyncer.tsx     (Task 12.3)
packages/ui/src/lib/format.ts              (Task 10.1)
```

### Files to modify (Tasks 1, 4, 7–10):
```
apps/client/package.json        — add next-intl to dependencies; add check:i18n script
apps/admin/package.json         — same
apps/client/next.config.mjs     — wrap with createNextIntlPlugin
apps/admin/next.config.mjs      — same
packages/ui/index.ts            — export formatDate, formatCurrency
```

### data-testid requirements for TopBar (test contracts):
Dev must add these `data-testid` attributes so tests can find elements:
```tsx
// Globe icon wrapper:
<span data-testid="locale-globe-icon"><Globe .../></span>

// Inactive locale buttons (only the INACTIVE locale gets this testid):
<button data-testid="locale-switch-en" ...>EN</button>  // when locale="bg"
<button data-testid="locale-switch-bg" ...>BG</button>  // when locale="en"
```
*Alternative: use both buttons always, style active one differently (no pointer-events)*

---

## Next Steps — TDD Green Phase

After implementing Story 3.7:

1. **Remove `it.skip()`** from all 7 test files
2. **Run the tests:**
   ```bash
   # packages/ui unit + component tests (format, TopBar, UserAvatarMenu)
   pnpm test --filter @eusolicit/ui

   # client unit tests (config files, message files, key diff, uiStore)
   pnpm test --filter client

   # admin unit tests (after adding Vitest to admin)
   pnpm test --filter admin
   ```
3. **Verify ALL tests pass** (green phase)
4. **Run build gate:**
   ```bash
   cd eusolicit-app/frontend
   pnpm check:i18n --filter client
   pnpm check:i18n --filter admin
   pnpm type-check
   pnpm build
   ```
5. **If any tests fail:** fix the implementation (not the test) unless the test has a bug
6. **Commit** all passing tests alongside the implementation

---

## Risks and Assumptions

| Risk | Description | Mitigation |
|------|-------------|------------|
| **E03-R-001** (score 6) | Locale middleware redirect loops | localePrefix: 'always' enforced; AC10 redirect test requires Playwright (deferred) |
| **E03-R-006** (score 4) | Missing i18n translation keys | AC4/AC9 tests verify key parity at unit level; fails CI before deploy |
| **Admin Vitest gap** | apps/admin has no test runner | Admin key-diff tests generated; dev must add Vitest to admin before they run |
| **Playwright gap** | No playwright.config.ts | AC10 redirect and AC6 full flow E2E tests deferred until Playwright added |
| **setLocale regression** | setLocale might be confused with S3.5 locale field | Tests clearly doc this is a NEW action (not just the field) |

---

## Quality Gate

Per epic test design (`test-design-epic-03.md`):
- **P0 pass rate:** 100% required (TopBar locale selector: T01–T06)
- **P1 pass rate:** ≥95% (message files, key diff, setLocale, UserAvatarMenu labels, config files)
- **P2 pass rate:** best-effort (format utilities)
- **Build gate:** `pnpm build` both apps — hard fail
- **Key diff gate:** `pnpm check:i18n` both apps — hard fail

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Story:** 3.7 i18n Setup with next-intl (BG + EN)
**TDD Phase:** RED (all tests skipped — implement story to reach GREEN)
