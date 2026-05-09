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
scope: cluster-C-admin-ui
batch: P3-h
storyKeys:
  - 12-1-tenant-list-management
  - 15-1-pricing-tier-admin
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/playwright.config.ts
  - eusolicit-app/frontend/apps/admin/middleware.ts
  - eusolicit-app/frontend/apps/admin/components/AdminRoleGuard.tsx
  - eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/layout.tsx
  - eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/tenants/components/TenantListPage.tsx
  - eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/billing/pricing-tiers/components/PricingTierListPage.tsx
  - eusolicit-app/frontend/apps/admin/lib/api/admin-tenants.ts
  - eusolicit-app/frontend/apps/admin/lib/api/admin-pricing-tiers.ts
  - eusolicit-app/frontend/apps/admin/lib/api/admin-client.ts
  - eusolicit-app/frontend/packages/ui/src/lib/stores/auth-store.ts
  - eusolicit-app/frontend/packages/ui/src/components/auth/AuthGuard.tsx
---

# Automation Summary: P3-h — Admin UI (cluster C, sub-run 8)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P3-h (cluster C, eighth net-new E2E coverage sub-run)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why this surface

The admin app at port `:3001` had **zero** Playwright UI coverage prior to
this run. The only existing admin spec — `admin-access-control.api.spec.ts`
— is API-level and entirely `test.skip()`-ed pending S12.11–S12.13.

Two surfaces were prioritised because they bracket the spectrum of admin
control-plane risk:

- **Tenants list** (`/tenants`) — the workhorse of platform operations.
  Operators rely on this page to find a customer's company, their tier,
  and basic activity. Story 12.1's `listAdminTenants` is currently a stub
  (`lib/api/admin-tenants.ts:193`) that returns 4 hard-coded items;
  swapping in the real backend should not change the UI shape, so the
  test is a forward-looking lock on the contract.
- **Pricing tiers admin** (`/billing/pricing-tiers`) — Story 15.1, the
  *only* admin module already wired to a real backend
  (`adminApiClient → :8002`). Pricing changes are revenue-impacting; an
  admin-only mis-render here flows directly to billing. The page renders
  empty / populated / loading branches conditionally on the API response,
  so deterministic mocks unlock all three.

The "Admin UI" batch was deferred to its own sub-run because it requires a
separate Playwright project (`admin-chromium`) wired against
`adminBaseURL = http://localhost:3001`, plus a distinct auth-store
namespace (`eusolicit-admin-auth-store-v2`).

---

## Files Created

### `e2e/specs/admin/admin-ui.admin.spec.ts` (NEW)

The `.admin.spec.ts` suffix is required — `playwright.config.ts:148`
routes only files matching `/\.admin\.spec\.ts$/` to the `admin-chromium`
project. (The default `.spec.ts` files are filtered to the client project
via `(?<!\.admin)\.spec\.ts$`.)

4 tests across 2 describe scopes:

| # | Surface | Scope | Description |
|---|---|---|---|
| 1 | `/en/tenants` | P0 | Seed admin auth-store + session cookie; navigate; assert `tenant-management-page`, `tenant-page-title`, `tenant-filter-bar`, `tenant-table`, plus `tenant-row-{cmp-001..cmp-004}` (the four hard-coded stub tenants from `admin-tenants.ts:109-149`). Empty/skeleton states absent post-resolve. |
| 2 | `/en/tenants` (non-admin) | P1 | Same seed pattern but `role: 'member'`. AdminRoleGuard (`apps/admin/components/AdminRoleGuard.tsx:21`) detects mismatch and `router.replace('/{locale}/403')`. Asserts URL matches `/en/403` and the tenant page wrapper does NOT render. |
| 3 | `/en/billing/pricing-tiers` | P1 | Mock `GET /api/v1/admin/billing/pricing-tiers` → `[]`. Assert `pricing-tier-management-page`, page title, `pricing-tier-add-btn`, and `pricing-tier-empty-state`. Table + skeleton absent. |
| 4 | `/en/billing/pricing-tiers` | P0 | Mock returns 1 active tier (`name: 'standard'`, EUR 99.00). Assert `pricing-tier-table`, `pricing-tier-row-standard`, name + price cells, edit + delete buttons (delete gated on `tier.active`). Empty + skeleton absent. |

### Mock + auth helpers

- `seedAdminAuth(page, user)` — addCookies (`eusolicit-session`) +
  addInitScript that writes the v1 `eusolicit-admin-auth-store` key. The
  shared store's auto-migration (`packages/ui/src/lib/stores/auth-store.ts:38-64`)
  lifts state to `eusolicit-admin-auth-store-v2` on first paint, so the
  AuthGuard's `onFinishHydration` callback resolves with `isAuthenticated=true`.
- `PLATFORM_ADMIN` / `NON_ADMIN` — fixture user objects with the right
  shape for the `User` interface in `auth-store.ts:6-22`.
- `mockPricingTiers(page, tiers)` — single-glob `**/api/v1/admin/billing/pricing-tiers`
  GET intercept. The admin-client uses native `fetch` (not axios) and
  targets `process.env.NEXT_PUBLIC_ADMIN_API_URL` (default `http://localhost:8002`);
  Playwright matches on the full URL so `**` covers any host prefix.

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for new file | ✅ 0 errors |
| `npx playwright test --list … --project=admin-chromium` | ✅ **4 tests collected** in 1 spec file |
| Active vs fixme split | 4 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| Admin app at `:3001` (test runner project `admin-chromium`) | `playwright.config.ts:142-149` (testMatch `\.admin\.spec\.ts$`) | ✅ |
| Auth-store key `eusolicit-admin-auth-store-v2` (admin app) | `auth-store.ts:87` (`NEXT_PUBLIC_APP_NAME` = "admin") | ✅ |
| v1 → v2 migration on first paint | `auth-store.ts:38-64` (synchronous `if (!localStorage.getItem(v2Key))`) | ✅ |
| `eusolicit-session` cookie required for `/tenants`, `/billing/*` | `apps/admin/middleware.ts` PROTECTED_PATHS list | ✅ |
| Non-admin role → `/{locale}/403` redirect | `apps/admin/components/AdminRoleGuard.tsx:21-23` | ✅ |
| `tenant-management-page` + `tenant-page-title` | `TenantListPage.tsx:163, 166` | ✅ |
| `tenant-filter-bar`, `tenant-table`, `tenant-row-{id}` | `TenantListPage.tsx:176, 223, 237` | ✅ |
| Stub returns 4 tenants `cmp-001..cmp-004` | `admin-tenants.ts:109-149` | ✅ |
| `pricing-tier-management-page` + `pricing-tier-add-btn` | `PricingTierListPage.tsx:222, 232` | ✅ |
| `pricing-tier-empty-state` (no tiers) | `PricingTierListPage.tsx:247` | ✅ |
| `pricing-tier-row-{name}` + per-row name/price/edit/delete | `PricingTierListPage.tsx:269, 273, 285, 309, 319` | ✅ |
| Delete button gated on `tier.active` | `PricingTierListPage.tsx:315` (`tier.active && (`) | ✅ |
| GET `/api/v1/admin/billing/pricing-tiers` (real) | `admin-pricing-tiers.ts:64` | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up                              # client + admin Next.js apps (3000, 3001)
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/admin/admin-ui.admin.spec.ts \
    --project=admin-chromium --reporter=list
```

### Risks / open questions for the CI pass

1. **Admin app must be running on `:3001`**: prior client batches assumed a
   single Next.js app on `:3000`. The `admin-chromium` project's `baseURL`
   is `http://localhost:3001` — `make up` already starts both apps, so this
   is no extra work in CI, but a CI matrix that historically only spun up
   the client app would need updating.

2. **Auth-store migration timing**: the v1→v2 migration runs synchronously
   at module load. `addInitScript` injects the v1 key BEFORE the page's
   first paint, so the migration sees it. If a future change makes the
   migration async (e.g. Suspense boundary), the seed strategy will need
   to write directly to `eusolicit-admin-auth-store-v2` instead.

3. **Tenants stub vs real backend**: when Story 12.1 swaps the stub for
   the real `/api/v1/admin/tenants` call, this test starts to depend on
   that backend's response shape. The 4-row assertion is the canary —
   a live-backend swap that returns a different count or different
   `company_id` values will fail the test, prompting a follow-up to add
   a `mockAdminTenants(page, items)` helper modeled on `mockPricingTiers`.

4. **Delete button visibility logic**: test #4 uses an active tier
   (`active: true`) so the delete button renders. If the stub default ever
   flips to inactive, `pricing-tier-delete-btn-{name}` would become absent
   and the assertion would tighten.

5. **`/403` redirect race**: test #2 asserts on URL `/en/403`. The redirect
   fires from `useEffect` after hydration, so a 15s timeout window covers
   the worst-case React-strict-mode double-render plus the route swap.
   Hardware-slow CI agents may need this bumped.

6. **Pricing tier name uniqueness**: testids embed `tier.name`, which
   is one of the three enum values (`standard`/`grant_module`/`enterprise_stack`).
   If the backend ever serves multiple tiers with the same name (e.g.
   versioning), the testid becomes ambiguous. Static check until that
   changes.

7. **Deferred follow-ups for P3-h**:
   - Pricing tier create/edit/delete mutations (POST/PATCH/DELETE captures).
   - Tenant detail drawer + tier-override form (`drawer-tier-override-section`,
     `drawer-override-plan-select`, `drawer-override-submit`).
   - Other admin surfaces with their own queries: crawlers (`/crawlers`),
     audit log (`/audit-log`), analytics (`/analytics`), compliance
     workflows (`/compliance/*`), white-label (`/tenants/[id]/white-label`).
   - Login page (`/[locale]/login`) — public, no auth seed needed.

---

## Coverage Delta

- Before P3-h: **0** Playwright UI tests for the admin app (only one
  admin-API access-control spec, fully `test.skip()`).
- After P3-h: **4 active** user-level E2E tests covering tenants-populated /
  tenants-rbac-redirect / pricing-empty / pricing-populated scopes.

---

## Cluster C running totals (through P3-h)

| Sub-run | Spec file(s) | Active | Fixme |
|---|---|---:|---:|
| P3-a | `tasks-kanban.spec.ts` | 5 | 0 |
| P3-b | `approvals.spec.ts` | 5 | 0 |
| P3-c | `analytics-pipeline-roi.spec.ts` | 5 | 0 |
| P3-d | `analytics-rest.spec.ts` | 8 | 0 |
| P3-e | `reports/reports.spec.ts` | 5 | 0 |
| P3-f | `calendar/calendar-settings.spec.ts` + `bid-outcomes/bid-outcomes.spec.ts` | 6 | 0 |
| P3-g | `proposals/comments-content-blocks.spec.ts` | 4 | 0 |
| P3-h | `admin/admin-ui.admin.spec.ts` | 4 | 0 |
| **Total** | **9 spec files** | **42** | **0** |

P3 (cluster C) — net-new E2E coverage — is now complete. **42 active tests
across 9 spec files**, all passing static validation, all CI runtime pending.

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P4-a** — `nps_feedback` API (cluster D, gap fill).
- **P4-b** — billing API (cluster D, gap fill).
- **P4-c** — Cross-service notification flow (cluster D, gap fill).

After P3-h runs green in CI, update the rollout-plan checklist line and
flip the cluster-C aggregate row to ✅ done.
