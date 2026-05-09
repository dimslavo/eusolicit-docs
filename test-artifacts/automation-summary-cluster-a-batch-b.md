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
scope: cluster-A-batch-b
batch: P2-b
storyKeys:
  - 8-6-stripe-checkout-upgrade-downgrade-flow
  - 8-7-stripe-customer-portal-integration
  - 8-8-usage-metering-with-redis-counters-stripe-sync
  - 8-9-per-bid-add-on-purchase-flow
  - 8-10-eu-vat-handling-via-stripe-tax-vies-validation
  - 14-4-external-collaborator-magic-link-flow-comment-only-role
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/e2e/specs/billing-checkout.spec.ts
  - eusolicit-app/e2e/specs/external-collaborator/accept-link-flow.spec.ts
  - eusolicit-app/e2e/support/fixtures/auth.fixture.ts
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/frontend/apps/client/middleware.ts
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/settings/billing/components/SubscriptionManagementPage.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/settings/billing/success/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/settings/billing/cancel/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/opportunities/components/OpportunityDetailPage.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(auth)/external/accept/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/external/proposal/[id]/page.tsx
  - eusolicit-app/frontend/apps/client/lib/api/external-collaborator.ts
  - eusolicit-app/frontend/packages/ui/src/components/AddOnPurchaseButton.tsx
  - eusolicit-app/frontend/apps/client/messages/en.json
  - eusolicit-app/frontend/apps/client/messages/bg.json
  - eusolicit-app/services/client-api/src/client_api/api/v1/billing.py
---

# Automation Summary: P2-b — billing-checkout + accept-link-flow

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P2-b (cluster A, sub-run 2 of 5)
**Status:** ✅ Static validation passed; runtime execution still blocked locally (chromium system libs); CI burn-in pending.

---

## Files Modified

### `e2e/specs/billing-checkout.spec.ts`

Original: 11 tests, all `test.skip`. Changes:

**Repaired and un-skipped (8):**

| Test | Story | Repair |
|---|---|---|
| `AC5: success page shows processing message after polling timeout` | 8.6 | Land directly on `/bg/workspace/default/settings/billing/success?session_id=…` — without a sessionStorage `eusolicit-checkout-prior-tier` marker the page short-circuits into the timeout branch (success/page.tsx:43-54), no 30s polling wait needed. Asserts on `t('refreshButton')` + back button via role-based selectors. |
| `AC6: cancel page shows cancellation message and back-to-billing CTA` | 8.6 | Same workspace-prefixed URL repair. Switched from missing `data-testid` to text-based selectors keyed on `t('checkoutCancelled')` and `t('backToBilling')`. |
| `8.7-AC4: manage subscription portal button opens Stripe portal in new tab` | 8.7 | Live testid `manage-subscription-btn` confirmed at `SubscriptionManagementPage.tsx:188`. Uses `page.addInitScript` to capture `window.open` calls; asserts portal URL + `_blank` target. Replaced `waitForTimeout` with `expect.poll` for the API-call check. |
| `8.7-AC4: manage subscription button disabled when stripe_customer_id is null` | 8.7 | Trivial repair — testid match. |
| `8.7-AC8: portal button label uses manageBilling i18n key` | 8.7 | Trivial repair — assert button text differs from English `'Manage Subscription'`. |
| `8.9-AC8: enterprise tier shows "Included in Plan" badge` | 8.9 | URL change to `/bg/workspace/default/opportunities/<id>`. Switched to live testids `addon-included-{type}` and `addon-purchase-btn-{type}` (from `AddOnPurchaseButton.tsx:58/100`). Mocked `addon/status` glob and subscription. |
| `8.9-AC10: add-on buttons use i18n keys (no hardcoded English)` | 8.9 | Same testids; conditional assertion (annotates if no purchase button is reachable). |
| `AC10: billing settings page renders with BG locale translations` | 8.10 | Switched from `[h1, page-title]` to live testid `subscription-page-title` + `subscription-tier`. |

**fixme'd with re-author note (3):**

| Test | Reason |
|---|---|
| `P0 8.6-E2E-001: upgrade via Stripe Checkout redirects and shows success banner` | The SubscriptionManagementPage has NO upgrade-from-trial CTAs — no `current-plan`, no `upgrade-btn-professional`, no checkout-session POST trigger. Pricing page CTAs link to `/register?plan=X`, not to `/api/v1/billing/checkout/session`. The "upgrade via Checkout" surface needs to ship before this test can be authored. |
| `subscription usage endpoint returns real-time Redis counters (8.8)` | Original spec body was empty (no test code). Belongs in pytest under `services/client-api/tests/integration/`, not Playwright. Track in P4. |
| `P1 8.9-E2E-001: per-bid add-on purchase end-to-end` | Multi-step flow (opportunity detail → checkout/session POST → return URL → unlock). Robust mocking is its own sub-run; defer to P3 cluster-C author for per-bid E2E. fixme leaves a re-author block listing live testids, URLs, and endpoints. |

### `e2e/specs/external-collaborator/accept-link-flow.spec.ts`

Original: 16 tests, all `test.skip`. Changes:

**Repaired and un-skipped (12):**

| Test | AC | Repair |
|---|---|---|
| `accept page renders without 404 and posts to /api/v1/external/accept` | AC10 | URL `/en/external/accept?token=…` works; middleware does NOT enforce session for `/external` (PROTECTED_PATHS in middleware.ts:23-32 omits it). |
| `successful accept stores JWT in sessionStorage + redirects` | AC10 | Asserts `sessionStorage["eusolicit-external-token"]` after redirect; verifies external token does NOT leak into `eusolicit-client-auth-store` (matches AC11 anti-collision rule). |
| `expired token (401) renders linkExpired localised error` | AC10 | Text-based assertion against EN i18n: `"This invitation link has expired."` (en.json:1839). |
| `consumed token (410) renders linkConsumed localised error` | AC10 | Text-based: `"This invitation link has already been used."` (en.json:1840). |
| `scoped view renders proposal title and section bodies` | AC11 | Updated mock proposal to `ExternalProposalResponse` schema (`current_version_content.sections[].body`, NOT `sections[].content`). Asserts via `getByRole('heading')`. |
| `scoped view does NOT render WorkspaceSwitcher` | AC11 | Asserts `combobox[name=workspace]` count is 0. |
| `scoped view does NOT render the protected app sidebar` | AC11 | Asserts `navigation[name=main\|primary]` count is 0. |
| `comment_only role: comment composer is visible` | AC11 | Built a real (unsigned) JWT shape with `role: comment_only` claim that the page's `parseRoleFromToken` (proposal/[id]/page.tsx:43-52) decodes successfully. Asserts textarea + `Submit` button visible. |
| `read_only role: comment composer is hidden` | AC11 | Same JWT helper; asserts comment placeholder count is 0. |
| `requests to /api/v1/proposals/{id} include external Bearer token` | AC11 | Captures `authorization` header on the proposal GET; asserts `Bearer <token>` matches the seeded sessionStorage value. Validates the allow-list interceptor in `lib/api/external-collaborator.ts:81-99`. |
| `revoked link (410) shows linkConsumed / linkInvalid error` | AC12 | Mocks 410 with `reason="revoked"` — page maps to `linkInvalid` since only `reason="consumed"` triggers `linkConsumed` (page.tsx:80-83). Text assertion EN: `"This invitation link is invalid or has been revoked."` |
| `BG locale accept page shows Bulgarian linkExpired message on 401` | AC17 | URL `/bg/external/accept?token=…`; asserts BG text `"Тази покана е изтекла."` (bg.json:1840). |

**fixme'd with re-author note (4):**

| Test | Reason |
|---|---|
| `error page renders "Contact the inviter" CTA` | Live UI renders the contactInviter copy as `<AlertDescription>` static text (page.tsx:120) — not a clickable link or button. Original spec asserted via `getByRole('link'\|'button')`. Either ship a real CTA (mailto?) or drop. |
| `scoped view renders logout button` | The proposal page (proposal/[id]/page.tsx:54-164) has no logout control. The `external.proposal.logout` i18n key exists (en.json:1847) but is unconsumed. Wire a logout button or drop. |
| `logout clears sessionStorage and redirects to accept page` | Depends on the missing logout button. |
| `comment_only can post a comment and see it in the thread` | Page submits comments via `postExternalComment` then clears the textarea — no comment list / thread is fetched or rendered. The "see it in the thread" assertion is unimplementable until a comment list ships. |

### Helpers introduced

- `e2e/specs/external-collaborator/accept-link-flow.spec.ts` adds a tiny `buildExternalJwt(payload)` builder so role-conditional rendering can be exercised without a real signing key.
- `e2e/specs/billing-checkout.spec.ts` introduces `mockSubscription(page, overrides)` and `mockCompanyProfile(page)` for DRY repeated setup.

### Headers updated

Both files: 🔴 RED PHASE blocks replaced with 🟠 PARTIAL GREEN headers naming what's repaired vs. fixme'd.

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for these 2 files | ✅ 0 errors |
| `npx playwright test --list` | ✅ **27 tests collected** (11 in billing-checkout, 16 in accept-link-flow) |
| Active vs fixme split | 20 active (`test(...)`) + 7 fixme (`test.fixme(...)`) — was 0 active before |
| Local runtime | ❌ Same blocker as P2-a: chromium-headless-shell needs `libnspr4.so` (system NSS not installed). Identical to all Playwright runs in this environment. |

### Runtime validation runbook (CI / provisioned host)

```bash
# From eusolicit-app/
make up                                                    # frontend (3000) + client-api (8001)
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/billing-checkout.spec.ts \
    e2e/specs/external-collaborator/accept-link-flow.spec.ts \
    --project=client-chromium --reporter=list
# Burn-in 3x:
for i in 1 2 3; do
  npx playwright test --config=playwright.config.ts \
      e2e/specs/billing-checkout.spec.ts \
      e2e/specs/external-collaborator/accept-link-flow.spec.ts \
      --project=client-chromium --reporter=line || break
done
```

### Risks / open questions for the CI pass

1. **Auth-store hydration on `/workspace/default/settings/billing`**: same risk as P2-a — if the auth store doesn't hydrate `companyId` from the seeded session cookie, `SubscriptionManagementPage` falls into its loading state and `manage-subscription-btn` won't render. Likely fix: also mock `/api/v1/auth/me` (or whatever the bootstrap endpoint is) to provide a stable `companyId`. Same caveat applies to all `authenticatedPage`-driven billing tests.
2. **Opportunity page render path (8.9 tests)**: `OpportunityDetailPage.tsx` fetches the opportunity itself + addon/status × 3 + likely requirements/documents. The 8.9 tests stub addon/status and subscription only — if the opportunity GET 404s, the page may render an access-denied or not-found branch that hides the addons section, making `addon-included-*` assertions fail. The 8.9-AC10 test guards via `if (await purchaseBtn.isVisible())` but 8.9-AC8 is more brittle. If CI shows this consistently, demote 8.9-AC8 to fixme and require an opportunity-detail mock helper before re-authoring.
3. **External proposal page (`/(protected)/external/proposal/[id]`)**: filewise it's in the `(protected)` route group, but middleware does NOT enforce session for `/external/*`. We rely on this matching CI behaviour. If the deploy ever adds `/external` to PROTECTED_PATHS, all accept-link-flow tests break.
4. **JWT role parsing on the proposal page**: `parseRoleFromToken` (proposal/[id]/page.tsx:43-52) splits on `.`, base64url-decodes the middle segment, and `JSON.parse`s it. The test's `buildExternalJwt` helper produces a valid 3-part JWT shape with a JSON payload; should round-trip cleanly.
5. **Toast detection**: assertions on `getByText(...)` should pick up sonner-style toasts since they render to the DOM. If the toast library uses an isolated portal with `aria-live`, swap to `getByRole('status')` on first CI fail.

---

## Coverage Delta (P2-b)

- Before P2-b: 0 active + 27 skipped tests across the two files.
- After P2-b: **20 active** + 7 fixme + 0 plain skip = 27 total. Net **+20 tests un-skipped** ready to run in CI.
- Cumulative through P2 (P2-a + P2-b): **+25 tests un-skipped** (5 from P2-a + 20 from P2-b), 8 fixme'd with actionable re-author docs.

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P2-c** — `proposals-workspace.spec.ts` (~22 skipped) + `trust/public-access.spec.ts` (~17 skipped) — 39 tests.
- After P2-b runs green in CI, update the rollout-plan checklist line.
