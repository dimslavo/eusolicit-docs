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
workflowType: atdd
storyId: 3-6-react-hook-form-zod-validation-patterns
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/3-6-react-hook-form-zod-validation-patterns.md
  - eusolicit-docs/test-artifacts/test-design-epic-03.md
  - eusolicit-app/frontend/packages/ui/vitest.config.ts
  - eusolicit-app/frontend/packages/ui/src/__tests__/lib/hooks/useSSE.test.ts
  - eusolicit-app/frontend/packages/ui/src/__tests__/stores/auth-store.test.ts
  - eusolicit-docs/test-artifacts/atdd-checklist-3-5-zustand-stores-tanstack-query-setup.md
---

# ATDD Checklist: Story 3.6 — React Hook Form + Zod Validation Patterns

**Date:** 2026-04-08
**Author:** TEA Master Test Architect
**TDD Phase:** 🔴 RED — Failing tests generated; feature not yet implemented
**Story:** `eusolicit-docs/implementation-artifacts/3-6-react-hook-form-zod-validation-patterns.md`
**Epic:** E03 — Frontend Shell & Design System

---

## Step 1: Preflight & Context

| Item | Status | Notes |
|------|--------|-------|
| Story has clear acceptance criteria | ✅ | 7 ACs; full implementation specs including code in Dev Notes |
| Vitest config found | ✅ | `eusolicit-app/frontend/packages/ui/vitest.config.ts` |
| Playwright config found | ✅ | `eusolicit-app/playwright.config.ts` (from S3.5 precedent) |
| Detected stack | ✅ | `frontend` — Next.js 14 App Router, Vitest (unit/component), Playwright (E2E) |
| Test artifacts directory | ✅ | `eusolicit-docs/test-artifacts/` |
| Epic test design loaded | ✅ | `test-design-epic-03.md` — E03-P1-013, E03-P2-009, E03-P2-010 |
| Prior story ATDD loaded | ✅ | S3.5 checklist reviewed for conventions; jsdom env pattern followed |
| Existing test patterns inspected | ✅ | `useSSE.test.ts`, `auth-store.test.ts` — patterns adopted for new tests |

### Unit/Component Test Prerequisites (Before Unskipping Vitest Tests)

**Runtime dependencies (Story 3.6 Task 1 — must be installed first):**
```bash
# From eusolicit-app/frontend/
pnpm add react-hook-form@^7.52.0 @hookform/resolvers@^3.9.0 zod@^3.23.0 \
  --filter @eusolicit/ui
pnpm add react-hook-form@^7.52.0 zod@^3.23.0 --filter client
pnpm add react-hook-form@^7.52.0 zod@^3.23.0 --filter admin
pnpm install
```

**Dev dependencies (already installed in S3.5 — no changes needed):**
```bash
# These are already available in packages/ui:
# @testing-library/react, jsdom, @vitejs/plugin-react, vitest
```

**⚠️ vitest.config.ts UPDATE REQUIRED for FormField component tests:**

`FormField.test.tsx` lives in `src/__tests__/components/forms/` — this path is NOT covered by the
current `environmentMatchGlobs`. Add the `components/**` pattern before unskipping File 2:

```typescript
// eusolicit-app/frontend/packages/ui/vitest.config.ts
environmentMatchGlobs: [
  ["src/__tests__/lib/hooks/**", "jsdom"],    // existing (S3.5)
  ["src/__tests__/components/**", "jsdom"],   // ADD — S3.6 FormField component tests
],
```

Without this addition, FormField tests run in `node` environment and DOM APIs (`document.querySelector`,
`screen.getByRole`, React rendering) are unavailable → all 15 component tests fail with
`ReferenceError: document is not defined`.

---

## Step 2: Generation Mode

**Mode:** AI Generation (no browser recording)
**Reason:** All target files (`useZodForm.ts`, `form.tsx`, `FormField.tsx`, `form-test/page.tsx`)
do not exist yet. Browser recording cannot capture selectors from pages that return 404.
All selectors and assertions derived from story AC specifications and Dev Notes implementation
code (full implementations provided inline in story Dev Notes — precise assertion targets available).

---

## Step 3: Test Strategy

### AC → Test Mapping

| AC | Requirement Summary | Test Level | Priority | Epic Test ID | File |
|----|---------------------|-----------|----------|--------------|------|
| AC1 | `useZodForm<TSchema>(schema, options?)` returns typed `UseFormReturn<z.infer<TSchema>>` with zodResolver | Unit (Vitest) | P2 | E03-P2-009 | `useZodForm.test.ts` |
| AC1 | `zodResolver` wired — invalid submit populates `formState.errors` | Unit (Vitest) | P2 | E03-P2-009 | `useZodForm.test.ts` |
| AC1 | Valid data passes resolver — no errors after submit | Unit (Vitest) | P2 | E03-P2-009 | `useZodForm.test.ts` |
| AC1 | Default mode is "onBlur" — trigger() surfaces errors without submit | Unit (Vitest) | P2 | E03-P2-009 | `useZodForm.test.ts` |
| AC1 | Options forwarded — `defaultValues` applied to form state | Unit (Vitest) | P2 | — | `useZodForm.test.ts` |
| AC1 | Options forwarded — `mode: "onSubmit"` override respected | Unit (Vitest) | P2 | — | `useZodForm.test.ts` |
| AC2 | shadcn form primitives exist at `packages/ui/src/components/ui/form.tsx` | Component (Vitest) | P1 | — | `FormField.test.tsx` (indirect: imports fail if form.tsx missing) |
| AC2 | `FormFieldInternal` NOT barrel-exported from `packages/ui/index.ts` | CI gate | P0 | E03-P0-001 | _(TypeScript strict build gate — existing CI)_ |
| AC3 | `<FormField>` renders label text above input (non-checkbox) | Component (Vitest) | P1 | E03-P1-013 | `FormField.test.tsx` |
| AC3 | `<FormField>` renders input slot (text input, type="text" default) | Component (Vitest) | P1 | E03-P1-013 | `FormField.test.tsx` |
| AC3 | `<FormField>` renders description text (FormDescription) | Component (Vitest) | P1 | E03-P1-013 | `FormField.test.tsx` |
| AC3 | `<FormField>` renders error message with `text-destructive` class on blur | Component (Vitest) | P1 | E03-P1-013 | `FormField.test.tsx` |
| AC3 | Render-prop children take precedence over type variant | Component (Vitest) | P1 | E03-P1-013 | `FormField.test.tsx` |
| AC3 | `disabled` prop passes through to underlying input | Component (Vitest) | P1 | — | `FormField.test.tsx` |
| AC4 | type="email" → `<input type="email">` | Component (Vitest) | P2 | — | `FormField.test.tsx` |
| AC4 | type="password" → `<input type="password">` | Component (Vitest) | P2 | — | `FormField.test.tsx` |
| AC4 | type="textarea" → `<textarea>` | Component (Vitest) | P2 | — | `FormField.test.tsx` |
| AC4 | type="select" → shadcn Select with combobox role | Component (Vitest) | P2 | — | `FormField.test.tsx` |
| AC4 | type="checkbox" → checkbox + inline label | Component (Vitest) | P2 | — | `FormField.test.tsx` |
| AC4 | type="radio" → RadioGroup with N options | Component (Vitest) | P2 | — | `FormField.test.tsx` |
| AC4 | type="date" → `<input type="date">` | Component (Vitest) | P2 | — | `FormField.test.tsx` |
| AC4 | type="file" → `<input type="file">` | Component (Vitest) | P2 | — | `FormField.test.tsx` |
| AC5 | FormMessage has `transition-all duration-200 ease-in-out` classes | Component (Vitest) | P1 | E03-P1-013 | `FormField.test.tsx` |
| AC6 | `/dev/form-test` page renders with expected heading | E2E (Playwright) | P2 | E03-P2-010 | `form-test-page.spec.ts` |
| AC6 | Empty submit shows Zod errors for all 4 required fields | E2E (Playwright) | P2 | E03-P2-010 | `form-test-page.spec.ts` |
| AC6 | Inline errors appear on blur (onBlur mode active) | E2E (Playwright) | P2 | E03-P2-010 | `form-test-page.spec.ts` |
| AC6 | Valid submit shows local success banner + calls addToast | E2E (Playwright) | P1 | E03-P2-010 | `form-test-page.spec.ts` |
| AC5 | Error animation classes present in E2E DOM | E2E (Playwright) | P2 | E03-P2-010 | `form-test-page.spec.ts` |
| AC7 | All exports from `packages/ui/index.ts` resolve | CI gate | P0 | E03-P0-001 | _(existing CI gate — no new test file)_ |
| AC7 | `pnpm build` exits 0 both apps | CI gate | P0 | E03-P0-001 | _(existing CI gate — no new test file)_ |
| AC7 | `pnpm type-check` exits 0 all packages | CI gate | P0 | E03-P0-001 | _(existing CI gate — no new test file)_ |

**Note on AC2 (shadcn form primitives not barrel-exported):** `form.tsx` exports `FormFieldInternal`
for internal use only. The TypeScript build gate (`pnpm type-check` / `E03-P0-001`) validates that
the barrel `packages/ui/index.ts` does NOT export `FormFieldInternal`. No separate test file generated.

**Note on AC7 (exports + build gate):** `E03-P0-001` is the active CI gate established in Story 3.1.
All new exports from `packages/ui/index.ts` are validated by the TypeScript compiler.
No new test file generated for AC7.

---

## Step 4: Generated Test Files (TDD RED PHASE)

### 🔴 File 1: `eusolicit-app/frontend/packages/ui/src/__tests__/lib/hooks/useZodForm.test.ts`

**Framework:** Vitest + @testing-library/react (renderHook + act)
**Environment:** jsdom (covered by existing `src/__tests__/lib/hooks/**` glob — no config change)
**Covers:** AC1 / E03-P2-009

| Test ID | it.skip() Description | Priority | AC | Epic ID | Failure Reason |
|---------|----------------------|----------|-----|---------|----------------|
| T01 | Returns core RHF methods: handleSubmit, register, reset, trigger, control, formState | P2 | AC1 | E03-P2-009 | `useZodForm.ts` does not exist → import error |
| T02 | E03-P2-009: invalid data (empty strings) populates `formState.errors` via zodResolver | P2 | AC1 | E03-P2-009 | `useZodForm.ts` missing; if exists without zodResolver → `handleSubmit` calls `onValid` anyway |
| T03 | E03-P2-009: valid data passes resolver — `onValid` called, no errors in formState | P2 | AC1 | E03-P2-009 | `useZodForm.ts` does not exist → import error |
| T04 | Default mode is "onBlur" — `trigger('name')` surfaces error without submit | P2 | AC1 | E03-P2-009 | `useZodForm.ts` missing; if mode defaults to "onSubmit" → trigger silently passes |
| T05 | `defaultValues` option applied to form state — getValues() returns preset values | P2 | AC1 | — | `useZodForm.ts` does not exist → import error; if options spread missing → defaults ignored |
| T06 | `mode: "onSubmit"` override respected — no errors before submit triggered | P2 | AC1 | — | `useZodForm.ts` missing; if options spread missing → mode override ignored |

**Total:** 6 tests (all `it.skip()`)

---

### 🔴 File 2: `eusolicit-app/frontend/packages/ui/src/__tests__/components/forms/FormField.test.tsx`

**Framework:** Vitest + @testing-library/react (render + screen + fireEvent + waitFor)
**Environment:** jsdom (requires `["src/__tests__/components/**", "jsdom"]` added to `vitest.config.ts`)
**Covers:** AC3, AC4, AC5 / E03-P1-013

| Test ID | it.skip() Description | Priority | AC | Epic ID | Failure Reason |
|---------|----------------------|----------|-----|---------|----------------|
| T01 | E03-P1-013: renders label text when `label` prop provided | P1 | AC3 | E03-P1-013 | `FormField.tsx` does not exist → import error |
| T02 | Renders `<input type="text">` as default type variant | P1 | AC3/AC4 | E03-P1-013 | `FormField.tsx` does not exist → import error |
| T03 | E03-P1-013: renders description text in FormDescription | P1 | AC3 | E03-P1-013 | `FormField.tsx` / `form.tsx` missing → import error |
| T04 | E03-P1-013: error message visible with `text-destructive` class after blur with invalid data | P1 | AC3 | E03-P1-013 | `FormField.tsx` / `form.tsx` missing; or FormMessage uses wrong class |
| T05 | E03-P1-013: FormMessage `<p>` has `transition-all duration-200 ease-in-out overflow-hidden` classes (AC5) | P1 | AC5 | E03-P1-013 | `FormField.tsx` / `form.tsx` missing; or FormMessage missing animation classes |
| T06 | `type="email"` renders `<input type="email">` | P2 | AC4 | — | `FormField.tsx` does not exist → import error |
| T07 | `type="password"` renders `<input type="password">` | P2 | AC4 | — | `FormField.tsx` does not exist → import error |
| T08 | `type="textarea"` renders `<textarea>` | P2 | AC4 | — | `FormField.tsx` does not exist → import error |
| T09 | `type="select"` renders shadcn Select with combobox role | P2 | AC4 | — | `FormField.tsx` does not exist → import error |
| T10 | `type="checkbox"` renders checkbox + inline label | P2 | AC4 | — | `FormField.tsx` does not exist → import error |
| T11 | `type="radio"` renders RadioGroup with 2 radio options | P2 | AC4 | — | `FormField.tsx` does not exist → import error |
| T12 | `type="date"` renders `<input type="date">` | P2 | AC4 | — | `FormField.tsx` does not exist → import error |
| T13 | `type="file"` renders `<input type="file">` | P2 | AC4 | — | `FormField.tsx` does not exist → import error |
| T14 | Render-prop `children` takes precedence over `type` variant (AC3) | P1 | AC3 | E03-P1-013 | `FormField.tsx` missing; or children prop ignored → type variant renders instead |
| T15 | `disabled` prop passes through to underlying input | P1 | AC3 | — | `FormField.tsx` missing; or disabled not forwarded → input remains enabled |

**Total:** 15 tests (all `it.skip()`)

---

### 🔴 File 3: `eusolicit-app/e2e/specs/smoke/form-test-page.spec.ts`

**Framework:** Playwright
**Playwright projects:** `client-chromium` (baseURL: `http://localhost:3000`)
**Covers:** AC5, AC6 / E03-P2-010

| Test ID | test.skip() Description | Priority | AC | Epic ID | Failure Reason |
|---------|------------------------|----------|-----|---------|----------------|
| T01 | Page renders at `/dev/form-test` with heading and RHF + Zod subtitle | P2 | AC6 | E03-P2-010 | `apps/client/app/dev/form-test/page.tsx` missing → 404 |
| T02 | E03-P2-010: empty submit shows Zod errors for all 4 fields (name, email, role, agreeToTerms) | P2 | AC6 | E03-P2-010 | Page missing → 404; or zodResolver not configured → no errors shown |
| T03 | E03-P2-010: name field shows "Name must be at least 2 characters" on blur with 1-char value | P2 | AC6 | E03-P2-010 | Page missing → 404; or mode is "onSubmit" → blur does not trigger validation |
| T04 | E03-P2-010: email field shows "Please enter a valid email address" on blur with invalid format | P2 | AC6 | E03-P2-010 | Page missing → 404; or zodResolver not wired → no blur validation |
| T05 | E03-P2-010: valid form (all fields filled) submission shows success banner | P1 | AC6 | E03-P2-010 | Page missing → 404; or `handleSubmit` not called; or `agreeToTerms` unchecked validation fails |
| T06 | E03-P2-010: error paragraphs carry `transition-all duration-200` CSS classes (AC5) | P2 | AC5 | E03-P2-010 | Page missing → 404; or FormMessage missing animation classes |

**Total:** 6 tests (all `test.skip()`)

---

## Step 4C: Aggregate Summary

| Metric | Value |
|--------|-------|
| TDD Phase | 🔴 RED |
| Total tests generated | 27 |
| Vitest unit tests (File 1) | 6 (useZodForm) |
| Vitest component tests (File 2) | 15 (FormField) |
| Playwright E2E tests (File 3) | 6 (form-test page) |
| All tests use skip() | ✅ `it.skip()` (Vitest) / `test.skip()` (Playwright) |
| All tests assert expected behavior | ✅ |
| Placeholder assertions (`expect(true).toBe(true)`) | 0 |
| Execution mode | SEQUENTIAL |

### Priority Distribution

| Priority | Vitest Unit | Vitest Component | E2E | Total |
|----------|------------|------------------|-----|-------|
| P0 | 0 | 0 | 0 | 0 (covered by existing E03-P0-001 CI gate) |
| P1 | 0 | 7 (T01–T05, T14, T15) | 1 (T05) | 8 |
| P2 | 6 | 8 (T06–T13) | 5 | 19 |
| P3 | 0 | 0 | 0 | 0 |
| **Total** | **6** | **15** | **6** | **27** |

### AC Coverage

| AC | Tests Covering | Files |
|----|----------------|-------|
| AC1 | T01–T06 | `useZodForm.test.ts` |
| AC2 | _(indirect — FormField/form.tsx imports fail if primitives missing)_ | `FormField.test.tsx` |
| AC3 | T01, T02, T03, T04, T14, T15 | `FormField.test.tsx` |
| AC4 | T02 (text), T06–T13 (all 8 variants) | `FormField.test.tsx` |
| AC5 | T05 (Vitest animation classes), T06 (E2E animation classes) | `FormField.test.tsx`, `form-test-page.spec.ts` |
| AC6 | T01–T06 | `form-test-page.spec.ts` |
| AC7 | _(existing E03-P0-001 CI build + type-check gate)_ | CI |

**All 7 ACs have test coverage. AC2 and AC7 covered by existing CI build gate.**

### Epic Test ID Coverage

| Epic Test ID | Priority | Story AC | Test File(s) | Tests |
|-------------|---------|----------|--------------|-------|
| E03-P1-013 | P1 | AC3, AC5 | `FormField.test.tsx` | T01, T02, T03, T04, T05, T14 |
| E03-P2-009 | P2 | AC1 | `useZodForm.test.ts` | T01, T02, T03, T04 |
| E03-P2-010 | P1/P2 | AC6 | `form-test-page.spec.ts` | T01–T06 |
| E03-P0-001 | P0 | AC7 | CI gate (existing) | — |

---

## Step 5: Validation

### Validation Checklist

- [x] Story has approved acceptance criteria (7 ACs, all specified with implementation code in Dev Notes)
- [x] All 7 ACs have test coverage (AC2/AC7 via existing CI gate; AC1/AC3/AC4/AC5/AC6 via new tests)
- [x] All Vitest tests use `it.skip()` (TDD red phase)
- [x] All Playwright tests use `test.skip()` (TDD red phase)
- [x] No placeholder assertions (`expect(true).toBe(true)`) — zero instances
- [x] Tests assert EXPECTED behavior (what will work once implemented)
- [x] No orphaned browser sessions (no Playwright CLI recording; not applicable)
- [x] Vitest unit tests saved to `src/__tests__/lib/hooks/` (existing jsdom glob path)
- [x] Vitest component tests saved to `src/__tests__/components/forms/` (requires vitest.config.ts update)
- [x] E2E test file saved to `eusolicit-app/e2e/specs/smoke/` (consistent with S3.5 smoke tests)
- [x] Checklist saved to `eusolicit-docs/test-artifacts/`
- [x] E2E spec uses `.spec.ts` suffix → runs in client-chromium project
- [x] Failure reasons are accurate (modules/pages do not exist before implementation)
- [x] Epic test IDs cross-referenced correctly (E03-P1-013, E03-P2-009, E03-P2-010)
- [x] All test imports are dynamic (`await import(...)`) consistent with S3.5 patterns
- [x] `renderHook` + `act` from `@testing-library/react` used for hook tests (consistent with S3.5 useSSE pattern)
- [x] React wrapper component pattern used for FormField component tests (Form + useZodForm)

### Selector Strategy (E2E Tests)

E2E tests use role-based and text-based selectors derivable from the story implementation:

| Selector | Component | Source in Story Dev Notes |
|----------|-----------|--------------------------|
| `getByRole('heading', { name: /Form Test/i })` | `<h1>` in `form-test/page.tsx` | AC6 Dev Notes heading text |
| `getByRole('textbox', { name: /Full Name/i })` | `<FormField name="name" label="Full Name">` | AC6 form structure |
| `getByRole('textbox', { name: /Email Address/i })` | `<FormField name="email" label="Email Address">` | AC6 form structure |
| `getByRole('combobox')` | shadcn Select trigger | AC4 — Select renders combobox role |
| `getByRole('option', { name: /Standard User/i })` | `<SelectItem value="user">` | AC6 roleOptions const |
| `getByRole('checkbox')` | `<FormField name="agreeToTerms" type="checkbox">` | AC6 form structure |
| `getByRole('button', { name: /Submit Demo Form/i })` | `<Button type="submit">` | AC6 button text |
| `getByText(/Form submitted successfully/i)` | Success banner `<div>` | AC6 success banner text |
| `getByText('Name must be at least 2 characters')` | `<FormMessage>` | Demo schema error message |
| `getByText('Please enter a valid email address')` | `<FormMessage>` | Demo schema error message |
| `getByText('Please select a role')` | `<FormMessage>` | Demo schema `required_error` |
| `getByText('You must agree to the terms to continue')` | `<FormMessage>` | Demo schema `errorMap` |

### Risk Coverage

No new high-risk items (no scores ≥6) are introduced by Story 3.6. The tests cover:

| Area | Test | Mitigation |
|------|------|------------|
| zodResolver not wired (form never validates) | T02 in useZodForm.test.ts | Import fails; then: handleSubmit calls onValid without errors → T02 fails |
| onBlur mode wrong (errors only on submit) | T04 in useZodForm.test.ts; T03 in form-test-page.spec.ts | trigger() silently passes; blur doesn't show error in E2E |
| FormMessage animation classes missing (AC5) | T05 in FormField.test.tsx; T06 in form-test-page.spec.ts | `transition-all` absent from className → assertion fails |
| agreeToTerms using z.boolean() instead of z.literal(true) | T05 in form-test-page.spec.ts | Unchecked checkbox (false) would pass boolean validation but fail literal(true) |

---

## Next Steps (TDD Green Phase)

After implementing Story 3.6 (all 7 tasks):

### 1. Install runtime dependencies (Task 1)

```bash
cd eusolicit-app/frontend
pnpm add react-hook-form@^7.52.0 @hookform/resolvers@^3.9.0 zod@^3.23.0 \
  --filter @eusolicit/ui
pnpm add react-hook-form@^7.52.0 zod@^3.23.0 --filter client
pnpm add react-hook-form@^7.52.0 zod@^3.23.0 --filter admin
pnpm install
```

### 2. Update vitest.config.ts (for FormField component tests)

```typescript
// eusolicit-app/frontend/packages/ui/vitest.config.ts
environmentMatchGlobs: [
  ["src/__tests__/lib/hooks/**",    "jsdom"],  // existing
  ["src/__tests__/components/**",   "jsdom"],  // ADD — FormField.test.tsx
],
```

### 3. Remove `it.skip()` / `test.skip()` from each test file

```
eusolicit-app/frontend/packages/ui/src/__tests__/lib/hooks/useZodForm.test.ts      — 6 tests
eusolicit-app/frontend/packages/ui/src/__tests__/components/forms/FormField.test.tsx — 15 tests
eusolicit-app/e2e/specs/smoke/form-test-page.spec.ts                                — 6 tests
```

### 4. Start dev server (for E2E tests)

```bash
cd eusolicit-app/frontend
pnpm dev --filter client    # http://localhost:3000
```

### 5. Run Vitest unit/component tests

```bash
# From eusolicit-app/frontend/
pnpm vitest run --filter @eusolicit/ui
```

### 6. Run Playwright E2E tests

```bash
# From eusolicit-app/
npx playwright test e2e/specs/smoke/form-test-page.spec.ts --project=client-chromium
```

### 7. Verify ALL 27 tests pass

If any tests fail after implementation:

- **T01 (useZodForm — methods)** — check that `useZodForm.ts` exports a named `useZodForm` function
- **T02 (zodResolver integration)** — ensure `resolver: zodResolver(schema)` is passed to `useForm()`; also ensure `defaultValues: { name: '', email: '' }` produces errors not "Required"
- **T04 (onBlur mode)** — ensure `mode: options?.mode ?? "onBlur"` is in the hook (default applied correctly)
- **T04 (FormField — text-destructive)** — check FormMessage has `text-destructive` in its className (not `text-red-500` raw — use CSS var approach)
- **T05 (FormField — animation classes)** — FormMessage must always render the `<p>` element (even when no error) with `transition-all duration-200 ease-in-out overflow-hidden`
- **T09 (select variant)** — shadcn Select uses `onValueChange` + `defaultValue` pattern; ensure RHF `field.onChange` is wired correctly
- **T05 (E2E — valid submit)** — ensure `agreeToTerms` uses `z.literal(true)` not `z.boolean()`; check that `form.reset()` clears the form and success banner appears before reset

### 8. Common GREEN-phase adjustments to expect

- **T04 useZodForm (onBlur mode via trigger):** `trigger('name')` is the programmatic equivalent of blurring a field. If the field has no value registered (just `trigger()` without `register()`), it may not surface errors. The test wraps `register('name')` in the same `act()` block before `trigger()`.
- **T09 shadcn Select in jsdom:** Radix UI Portals may not attach to `document.body` in jsdom by default. If `getByRole('combobox')` fails, confirm the `@radix-ui/react-select` package is available and that the test wrapper has a proper `document.body`.
- **T05 E2E animation classes:** If `errorEl?.getAttribute('class')` returns null, the `<p>` may be wrapped in another element. Use `await page.locator('p:has-text("Name must be at least 2 characters")').getAttribute('class')` as an alternative locator strategy.
- **T02 form-test-page E2E (select role):** The shadcn Select in Playwright may require `page.waitForSelector('[role="option"]')` after clicking the trigger if the option portal has an animation delay. Add a short wait if needed.

### 9. Commit passing tests

Commit all 3 test files together with the Story 3.6 implementation. Update vitest.config.ts as part of the same commit.

---

## Implementation Guidance

### Files to Create (RED → GREEN triggers)

```
eusolicit-app/frontend/
├── packages/ui/
│   ├── src/
│   │   ├── components/
│   │   │   ├── ui/
│   │   │   │   └── form.tsx               ← T01–T15 (FormField.test.tsx — indirect via import chain)
│   │   │   └── forms/
│   │   │       ├── FormField.tsx          ← T01–T15 (FormField.test.tsx)
│   │   │       └── index.ts              ← barrel export
│   │   └── lib/
│   │       └── hooks/
│   │           ├── useZodForm.ts          ← T01–T06 (useZodForm.test.ts)
│   │           └── index.ts              ← MODIFY: add useZodForm export
│   └── index.ts                           ← MODIFY: S3.6 exports section
├── apps/
│   ├── client/
│   │   ├── app/dev/form-test/
│   │   │   └── page.tsx                  ← T01–T06 (form-test-page.spec.ts)
│   │   └── package.json                  ← MODIFY: add react-hook-form, zod
│   └── admin/
│       └── package.json                  ← MODIFY: add react-hook-form, zod
```

### Critical Implementation Requirements

1. **`useZodForm` resolver:** `resolver: zodResolver(schema)` MUST be inside the `useForm()` call — T02 asserts errors are populated by zodResolver.
2. **`useZodForm` mode default:** `mode: options?.mode ?? "onBlur"` — T04 (unit) asserts trigger() surfaces errors without submit; T03 (E2E) asserts blur validation works.
3. **FormMessage `text-destructive` class:** T04 in FormField.test.tsx asserts `expect(errorMessage).toHaveClass('text-destructive')`. The class must be `text-destructive` (shadcn CSS var), NOT `text-red-500` directly.
4. **FormMessage animation classes always present:** T05 in FormField.test.tsx uses `document.querySelectorAll('p[class*="transition-all"]')` — the `<p>` must always render (even without an error) with `transition-all duration-200 ease-in-out overflow-hidden`. FormMessage uses conditional `max-h-0 opacity-0` / `max-h-10 opacity-100` for collapse/expand.
5. **agreeToTerms as `z.literal(true)`:** T05 in E2E asserts the success banner appears only when checkbox is checked. Using `z.boolean()` would allow `false` to pass — use `z.literal(true)` as specified.
6. **`FormFieldInternal` NOT in barrel:** TypeScript build gate (E03-P0-001) validates `FormFieldInternal` is absent from `packages/ui/index.ts`. Only the EU Solicit `FormField` is exported.
7. **Select controlled pattern:** shadcn Select uses `onValueChange={field.onChange}` + `defaultValue={field.value}` (NOT `value={field.value}`). T09 in E2E opens the select and clicks an option — the `onValueChange` handler must fire to update the form state.

---

## Notes & Assumptions

1. **Story 3.5 is fully implemented.** `useSSE`, `auth-store`, `api-client`, `uiStore` (with `addToast()`) are in GREEN phase. Story 3.6 depends on `addToast()` from `uiStore` for the demo page — this method is already in place.

2. **No authentication required for E2E tests.** The `/dev/form-test` page is a developer utility under the `/dev/` route — not protected by `<AuthGuard>`. Tests navigate directly without seeding auth state.

3. **shadcn Select in Playwright:** The Radix UI Select opens a portal rendered at `document.body`. In Playwright, the option list appears after clicking the trigger. Tests use `getByRole('option', { name: ... })` which correctly locates the portalled option.

4. **`vitest.config.ts` glob order:** The new `["src/__tests__/components/**", "jsdom"]` entry must be added to `environmentMatchGlobs` BEFORE running FormField tests. Without it, `document` is undefined in node environment → 15 tests fail with `ReferenceError`.

5. **S03.11 Toast dependency:** `addToast()` in the demo page queues a toast into `uiStore`, but the visual `<Toaster>` component doesn't render until S03.11. The local success banner (`setSubmitted(true)`) renders immediately and is what T05 (E2E) asserts against. This is correct per AC6.

6. **Zod `z.literal(true)` vs `z.boolean()`:** The story explicitly specifies `z.literal(true)` for `agreeToTerms` — more precise than `z.boolean()` as it rejects `false` with the custom error map message. T05 E2E depends on the checkbox being checked to pass validation.

7. **FormField component tests require React 18 concurrent mode:** `@testing-library/react` v14+ uses `createRoot` by default (React 18). If tests fail with "act() warning", wrap all state updates in `act()`. The `waitFor()` utility handles async DOM updates correctly.

---

**Generated by:** BMad TEA Agent — ATDD Workflow
**Workflow:** `bmad-testarch-atdd`
**Story:** 3.6 — React Hook Form + Zod Validation Patterns
**Version:** BMad v6 (sequential execution mode)
