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
scope: cluster-A-batch-a
batch: P2-a
storyKeys:
  - 7-13-ai-draft-generation-panel
  - 8-10-eu-vat-handling-via-stripe-tax-vies-validation
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/e2e/proposal_generation.spec.ts
  - eusolicit-app/e2e/specs/billing-vat.spec.ts
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/e2e/global-setup.ts
  - eusolicit-app/e2e/support/fixtures/auth.fixture.ts
  - eusolicit-app/frontend/packages/ui/src/components/VatNumberInput.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/settings/company/page.tsx
  - eusolicit-app/frontend/apps/client/messages/en.json
  - eusolicit-app/frontend/apps/client/messages/bg.json
  - eusolicit-app/frontend/apps/client/middleware.ts
  - eusolicit-app/services/client-api/src/client_api/api/v1/billing.py
  - eusolicit-app/services/client-api/src/client_api/api/v1/auth.py
---

# Automation Summary: P2-a — proposal_generation + billing-vat

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P2-a (cluster A, sub-run 1 of 5)
**Status:** ✅ Static validation passed; runtime execution blocked by environment (no chromium system libs locally — defers to CI).

---

## Files Modified

### `e2e/proposal_generation.spec.ts` — fixme

**Outcome:** marked `test.fixme` with re-author guidance.

**Why fixme, not un-skip:**

| Concern | Spec assumed | Production reality |
|---|---|---|
| URL | `/proposals/some-proposal-id/workspace` | `/[locale]/workspace/[workspaceId]/proposals/[id]` (App Router) |
| Component | `AIDraftGenerationPanel` (no source file exists) | `AiGeneratePanel.tsx` under `app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/components/` |
| testids | `ai-draft-generation-panel`, `generation-progress-indicator` | `ai-generate-btn`, `ai-generate-stop`, `generation-progress-list`, `progress-section-{key}`, `ai-generate-apply-sections`, `ai-generate-accept-all`, `ai-generate-discard`, `ai-generate-error`, `ai-generate-retry` |
| Buttons | named "Generate Draft" / "Stop Generating" / "Accept Draft" / "Discard Draft" | i18n strings via `useTranslations` |
| SSE schema | `{type: "delta", section, content}` / `{type: "done"}` | needs verification against `api/v1/proposals.py:612` |

This is a full re-author, not a repair. fixme preserves the AC trail (E07-P0-012) and points future authors to `AiGeneratePanel.tsx` snapshot + correct URL pattern.

**Tracking:** P3 sub-run "proposal generation re-author" (added to rollout plan).

### `e2e/specs/billing-vat.spec.ts` — un-skipped + repaired

**Outcome:** all 5 tests un-skipped; rewritten against the live UI.

**Repairs:**

1. **Auth wiring** — switched `import { test, expect } from '@playwright/test'` to `import { test, expect } from '../support/fixtures'`; tests use `authenticatedPage` fixture (calls `/api/v1/auth/test-login`, sets `eusolicit-session` cookie matching the middleware).
2. **URL paths** — `/settings/company` → `/en/workspace/default/settings/company` (next-intl `localePrefix: "always"` + protected route under `/workspace/<id>/...` per middleware.ts:84).
3. **BG locale URL** — `/bg/settings/company` → `/bg/workspace/default/settings/company`.
4. **Company profile mock** — `/api/v1/companies/profile` → `**/api/v1/companies/*` (the live page calls `GET /api/v1/companies/{companyId}` per page.tsx:87 — no `/profile` segment).
5. **Helper extracted** — `mockCompanyProfile(page, overrides)` so each test sets only what it needs (initial tax_id / vat_validation_status / country).
6. **Status assertions** — switched from regex matches like `/valid/i` to `getByText('Valid', { exact: true })` aligned to the actual i18n strings (`billing.vatStatusValid` = "Valid" in en.json:1201, "Валиден" in bg.json:1201).
7. **Header docstring** — 🔴 RED PHASE block replaced with 🟢 GREEN PHASE (matches the live impl: `VatNumberInput.tsx`, `/api/v1/billing/vat/validate` route at billing.py:454, EN+BG i18n keys at en.json:1196-1204 / bg.json:1196-1204).

**Tests un-skipped:**

| # | Test | AC | Approach |
|---|---|---|---|
| 1 | `P1 8.10-API-001: valid EU VAT shows "Valid" badge and success toast` | AC7 + 8.10-API-001 | Fill VAT, blur, intercept POST, assert badge + toast text |
| 2 | `P1 8.10-API-002: invalid EU VAT shows "Invalid" badge and error toast` | AC7 + 8.10-API-002 | Fill VAT, mock 422, assert "Invalid" badge + "Invalid VAT number" toast |
| 3 | `P1 8.10-API-003: VIES downtime shows "Pending" badge and info toast` | AC7 + 8.10-API-003 | Mock 200 `{status: "pending"}`, assert "Pending" badge + info toast, no "Invalid" surfaces |
| 4 | `AC7: VatNumberInput shows pre-existing validation status from company profile on page load` | AC7 | tax_id pre-filled, status=valid; assert badge visible without input + no POST fired (component returns early when value === defaultValue per VatNumberInput.tsx:56) |
| 5 | `AC9: VAT input component uses i18n — no hardcoded English in BG locale` | AC9 | Navigate `/bg/workspace/...`; assert label "ДДС номер", no "VAT Number" label visible, success toast "ДДС номерът е потвърден" |

---

## Validation

| Check | Result |
|---|---|
| TypeScript (e2e tsconfig, filtered for these 2 files) | ✅ 0 errors |
| `npx playwright test --list` (collection) | ✅ proposal_generation 1 fixme test, billing-vat 5 tests collected |
| Local runtime execution | ❌ **blocked**: chromium-headless-shell fails with `libnspr4.so: cannot open shared object file`. System NSS libs not installed in this environment. Not a test issue — affects every Playwright spec in this repo when run from this machine. |
| Coverage of i18n keys | ✅ EN + BG keys verified present at en.json:1196-1204 and bg.json:1196-1204 (`vatNumber`, `vatValidated`, `vatInvalid`, `vatPending`, `vatStatusValid`, `vatStatusInvalid`, `vatStatusPending`) |

### Runtime validation deferred

The browser launch error is environment-level:

```
chrome-headless-shell: error while loading shared libraries:
libnspr4.so: cannot open shared object file: No such file or directory
```

**To validate runtime in CI / a properly provisioned environment:**

```bash
# From eusolicit-app/
make up                      # frontend (3000) + client-api (8001) up
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/proposal_generation.spec.ts e2e/specs/billing-vat.spec.ts \
    --project=client-chromium --reporter=list
# Burn-in 3x:
for i in 1 2 3; do
  npx playwright test --config=playwright.config.ts \
      e2e/specs/billing-vat.spec.ts --project=client-chromium --reporter=line || break
done
```

### Risks / open questions for the CI pass

1. **Auth-store hydration**: `authenticatedPage` sets a `eusolicit-session` cookie from `/api/v1/auth/test-login`. The settings page reads `companyId` from `useAuthStore`. The store should hydrate from the session cookie via the standard auth-bootstrap path. **If the store stays empty, the page renders its `!companyId || isLoading` loading state and `getByLabel('VAT Number')` will time out.** Fix would be to also mock `/api/v1/auth/me` (or the equivalent bootstrap endpoint).
2. **Default workspace existence**: URL uses `/workspace/default/...`. The middleware allows `default` as a special-case workspaceId (middleware.ts:84). New users seeded by `test-login` need a `workspaces.default` row — confirmed via the standard registration flow at `services/client-api/.../auth.py`.
3. **Toast library**: assertions use `getByText(...)` against the toast text. If the toast component renders inside an `aria-live` region with non-default semantics, the locator may need `getByRole('status')` instead. Adjust on first CI run if needed.

---

## Coverage Delta

- Before: `proposal_generation.spec.ts` → 2 skips counted (1 actual `test.skip` + 1 in a comment); `billing-vat.spec.ts` → 5 of 5 tests skipped; total deferred = 5 P1 + 1 P0 = **6 dormant tests**.
- After: 5 tests un-skipped and re-authored (billing-vat); 1 test marked fixme with re-author guidance and current testid map (proposal_generation). **Active count: +5 tests; fixme count: 1 with actionable docs.**

---

## What's Next

Per `full-suite-rollout-plan.md` sequence:

- **P2-b** — `billing-checkout.spec.ts` (~9 skips) + `external-collaborator/accept-link-flow.spec.ts` (18 skips) — 27 tests.
- After P2-a runs green in CI, update `full-suite-rollout-plan.md` checklist line for P2-a.
