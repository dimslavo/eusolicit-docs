---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation']
lastStep: 'step-02-generation'
lastSaved: '2026-04-19'
storyId: '8-13-subscription-management-page-trial-banner'
storyKey: '8-13'
outputFile: 'atdd-checklist-8-13-subscription-management-page-trial-banner.md'
tddPhase: 'RED'
testFile: 'eusolicit-app/frontend/apps/client/__tests__/subscription-management-s8-13.test.ts'
testResults: '160 failing | 22 passing (182 total) — RED phase confirmed 2026-04-19'
inputDocuments:
  - eusolicit-docs/implementation-artifacts/8-13-subscription-management-page-trial-banner.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - eusolicit-app/frontend/apps/client/__tests__/pricing-page-s8-12.test.ts
  - eusolicit-app/frontend/apps/client/vitest.config.ts
  - eusolicit-app/_bmad/bmm/config.yaml
detectedStack: fullstack
framework: 'Vitest (environment: node) + static file assertions'
epicTestDesignIds:
  - 'AC-14 (P1 bundled) — Trial banner shows days remaining + upgrade CTA during trialing status'
  - 'AC-15 (P1) — Subscription management page: tier, usage meters, portal link, billing history'
---

# ATDD Checklist — Story 8.13: Subscription Management Page & Trial Banner

## Overview

**Story:** 8.13 — Subscription Management Page & Trial Banner
**TDD Phase:** 🔴 RED — All 160 target tests FAIL until the story is implemented
**Test File:** `eusolicit-app/frontend/apps/client/__tests__/subscription-management-s8-13.test.ts`
**Run Command:** `cd eusolicit-app/frontend/apps/client && ./node_modules/.bin/vitest run __tests__/subscription-management-s8-13.test.ts`
**Confirmed RED:** 2026-04-19 — 160 failing | 22 passing (pre-existing file/content checks pass)

---

## Stack & Framework

- **Stack:** Fullstack (FastAPI backend + Next.js 14 frontend)
- **Frontend test framework:** Vitest 1.6.1, environment: `node`
- **Test strategy:** Static file-system assertions (Vitest + Node `fs`) — no DOM rendering
- **Pattern source:** `__tests__/pricing-page-s8-12.test.ts` (Story 8.12 ATDD template)

---

## Test Coverage Map

| AC | Test Group | Tests | Priority |
|----|------------|-------|----------|
| AC1 | File structure | 8 exists checks | P1 |
| AC1 | `settings/billing/page.tsx` server shell | 4 tests | P1 |
| AC1 | `SubscriptionManagementPage.tsx` page-level testids | 12 tests | P1 |
| AC2 | Usage meters — `subscription-usage-*` testids | 18 tests | P1 |
| AC3 | Billing history table — `subscription-billing-history` | 7 tests | P1 |
| AC4 | `TrialBanner.tsx` component | 19 tests | P1 |
| AC5 | `(protected)/layout.tsx` TrialBanner integration | 9 tests | P1 |
| AC6 | Backend `GET /api/v1/billing/invoices` in `billing.py` | 8 tests | P1 |
| AC7 | `lib/api/billing.ts` new interfaces + functions | 16 tests | P1 |
| AC7 | `lib/queries/use-billing.ts` new hooks | 11 tests | P1 |
| AC8 | `en.json` i18n key completeness | 12 tests | P1 |
| AC8 | `bg.json` i18n key completeness | 12 tests | P1 |
| AC8 | i18n parity (en ↔ bg key counts) | 3 tests | P1 |
| Dev Notes | Critical mistake prevention | 10 tests | P1 |
| **Total** | | **182 tests** | |

---

## Epic 8 Test Design Coverage

- **AC-14** (P1 bundled): Trial banner shows days remaining + upgrade CTA during trialing status
  → Covered by `TrialBanner.tsx` component assertions (testids, sessionStorage, zero-day state)
  → E2E coverage bundled with 8.6-E2E-001 setup per test-design-epic-08.md

- **AC-15** (P1): Subscription management page: tier, usage meters, portal link, billing history
  → Covered by `SubscriptionManagementPage.tsx` structural assertions, usage meter testids,
     billing history table, backend endpoint, API module, and hook exports

---

## Acceptance Criteria → Test Mapping

### AC1 — Refactored Subscription Dashboard

**Failing tests (implementation required):**
- [ ] `settings/billing/page.tsx` is a server component — no `"use client"` directive
- [ ] `settings/billing/page.tsx` imports/renders `<SubscriptionManagementPage />`
- [ ] `settings/billing/page.tsx` does NOT import `useQuery` directly
- [ ] `settings/billing/components/SubscriptionManagementPage.tsx` exists
- [ ] `SubscriptionManagementPage.tsx` has `"use client"` directive
- [ ] `SubscriptionManagementPage.tsx` exports `SubscriptionManagementPage` function
- [ ] `data-testid="subscription-page-title"` present
- [ ] `data-testid="subscription-tier"` present
- [ ] `data-testid="subscription-status-badge"` present
- [ ] `data-testid="subscription-billing-period"` present
- [ ] `data-testid="subscription-trial-end"` present (conditional on `is_trial`)
- [ ] `data-testid="manage-subscription-btn"` present (Stripe portal CTA preserved)
- [ ] Uses `useTranslations("billing")`
- [ ] Preserves portal mutation (`/api/v1/billing/portal/session`)
- [ ] Imports `useSubscription, useUsage, useInvoices` from `use-billing`
- [ ] Does NOT include checkout mutation (removed — upgrade goes via pricing page)

### AC2 — Usage Meters (Progress Bars)

**Failing tests (implementation required):**
- [ ] `data-testid="subscription-usage-meters"` container present
- [ ] `data-testid="subscription-usage-ai_summary"` wrapper present
- [ ] `data-testid="subscription-usage-proposal_draft"` wrapper present
- [ ] `data-testid="subscription-usage-compliance_check"` wrapper present
- [ ] `data-testid="subscription-usage-{metric}-text"` (3 testids: consumed/limit text)
- [ ] `data-testid="subscription-usage-{metric}-pct"` (3 testids: percentage display)
- [ ] `data-testid="subscription-usage-{metric}-bar"` (3 testids: progress bar fill)
- [ ] `data-testid="subscription-usage-{metric}-unlimited"` (3 testids: Enterprise unlimited)
- [ ] `data-testid="subscription-usage-loading"` skeleton state
- [ ] Bar color `bg-indigo-500` for < 80%
- [ ] Bar color `bg-amber-500` for 80–99%
- [ ] Bar color `bg-red-500` for ≥ 100%
- [ ] `Math.min` used to cap bar fill at 100%
- [ ] `null` limit handled as unlimited (Enterprise tier)

### AC3 — Billing History Table

**Failing tests (implementation required):**
- [ ] `data-testid="subscription-billing-history"` container present
- [ ] `data-testid="subscription-billing-empty"` EmptyState for no invoices
- [ ] `noInvoices` i18n key used in empty state
- [ ] `Table`, `TableBody`, `TableRow` components used
- [ ] Maps over `invoices` from `useInvoices()`
- [ ] Invoice `created_at` formatted as locale date string
- [ ] Invoice `status` and `payment_terms` columns present

### AC4 — Trial Banner Component

**Failing tests (implementation required):**
- [ ] `(protected)/components/TrialBanner.tsx` exists
- [ ] `"use client"` directive present
- [ ] Exports `TrialBanner` function
- [ ] `data-testid="trial-banner"` on root element
- [ ] `role="banner"` for accessibility
- [ ] `data-testid="trial-banner-days"` for countdown text
- [ ] `data-testid="trial-banner-upgrade-cta"` link to `/{locale}/settings/billing`
- [ ] Upgrade CTA uses `next/link`
- [ ] `data-testid="trial-banner-close"` dismiss button
- [ ] `sessionStorage` key `eusolicit-trial-banner-dismissed` used
- [ ] `DISMISS_KEY` constant defined
- [ ] `Math.ceil` and `Math.max(0, ...)` for days calculation
- [ ] Zero-day state (`daysRemaining === 0`) renders `zeroDay` i18n key
- [ ] Countdown uses `trialBanner.daysRemaining` with `{days}` interpolation
- [ ] Uses `useTranslations("billing")`
- [ ] `useEffect` for SSR-safe sessionStorage access
- [ ] `useState` for `dismissed` flag
- [ ] Returns `null` when dismissed
- [ ] Amber styling (`bg-amber-50` or similar)
- [ ] Accepts `trialEnd` and `locale` props
- [ ] No hardcoded English strings (all text via `t()`)

### AC5 — Trial Banner Integration in Protected Layout

**Failing tests (implementation required):**
- [ ] `(protected)/layout.tsx` imports `TrialBanner`
- [ ] Import path contains `./components/TrialBanner`
- [ ] Shared `queryKey: ["billing", "subscription"]`
- [ ] `staleTime: 60_000` on subscription query
- [ ] Conditional render on `trialing` status
- [ ] Passes `trial_end` to `TrialBanner`
- [ ] React Fragment `<>...</>` wraps `TopBar` + `TrialBanner` in `topbar` slot
- [ ] NO `suspense: true` on subscription query (must not block layout render)
- [ ] Imports `getSubscription` from `lib/api/billing`

### AC6 — Backend Billing Invoices Endpoint

**Failing tests (implementation required):**
- [ ] `billing.py` declares `"/invoices"` route string
- [ ] `get_invoices` function defined
- [ ] `InvoiceDTO` Pydantic model defined
- [ ] `InvoiceDTO` has `stripe_invoice_id`, `payment_terms`, `line_items`, `created_at`, `paid_at`
- [ ] `EnterpriseInvoice` imported in `billing.py`
- [ ] `get_invoices` uses `require_auth` (authenticated)
- [ ] `get_invoices` uses `_billing_session()` context manager
- [ ] Queries with `company_id` filter, `created_at` order, `LIMIT 12`
- [ ] Module docstring updated with `/billing/invoices` (Story 8-13)

### AC7 — API Module & React Query Hooks

**Failing tests (implementation required):**
- [ ] `billing.ts` exports `SubscriptionStatus` interface
- [ ] `SubscriptionStatus` has `tier`, `is_trial`, `trial_end`, `current_period_end`
- [ ] `billing.ts` exports `UsageResponse` interface
- [ ] `UsageResponse` includes metric shape for `ai_summary`, `proposal_draft`, `compliance_check`
- [ ] `billing.ts` exports `InvoiceRecord` interface
- [ ] `InvoiceRecord` has `stripe_invoice_id`, `payment_terms`, `line_items`, `paid_at`
- [ ] `billing.ts` exports `getSubscription` async function → `GET /api/v1/billing/subscription`
- [ ] `billing.ts` exports `getUsage` async function → `GET /api/v1/subscription/usage`
- [ ] `billing.ts` exports `getInvoices` async function → `GET /api/v1/billing/invoices`
- [ ] `billing.ts` preserves `getTiers` (regression guard)
- [ ] `billing.ts` preserves `TIER_PRICES` (regression guard)
- [ ] `use-billing.ts` exports `useSubscription` with `queryKey: ["billing", "subscription"]`
- [ ] `useSubscription` has `staleTime: 60_000`
- [ ] `useSubscription` has `refetchOnWindowFocus: true`
- [ ] `use-billing.ts` exports `useUsage` with `queryKey: ["billing", "usage"]`, `staleTime: 120_000`
- [ ] `use-billing.ts` exports `useInvoices` with `queryKey: ["billing", "invoices"]`, `staleTime: 300_000`
- [ ] `use-billing.ts` preserves `useTiers` (regression guard for pricing page)
- [ ] `use-billing.ts` imports `getSubscription, getUsage, getInvoices` from billing

### AC8 — i18n Completeness

**Failing tests (implementation required):**

**en.json (4 trial banner keys):**
- [ ] `billing.trialBanner.daysRemaining` (with `{days}` interpolation)
- [ ] `billing.trialBanner.zeroDay`
- [ ] `billing.trialBanner.upgradeNow`
- [ ] `billing.trialBanner.dismiss`

**en.json (12 subscription keys):**
- [ ] `billing.subscription.pageTitle`
- [ ] `billing.subscription.usageMetersTitle`
- [ ] `billing.subscription.aiSummaryLabel`
- [ ] `billing.subscription.proposalDraftLabel`
- [ ] `billing.subscription.complianceCheckLabel`
- [ ] `billing.subscription.unlimited`
- [ ] `billing.subscription.billingPeriodLabel`
- [ ] `billing.subscription.billingHistoryTitle`
- [ ] `billing.subscription.noInvoices`
- [ ] `billing.subscription.invoiceDate`, `invoiceStatus`, `invoiceTerms`
- [ ] `billing.subscription.statusBadge.trialing|active|pastDue|canceled` (4 values)

**bg.json:** Matching Bulgarian translations for all 16+ keys above

**Parity:**
- [ ] `billing.trialBanner` key counts match between `en.json` and `bg.json`
- [ ] `billing.subscription` leaf key counts match between `en.json` and `bg.json`
- [ ] Existing `billing.*` keys NOT removed (regression guard)

---

## Critical Mistakes to Prevent (enforced by tests)

| Mistake | Enforced By |
|---------|-------------|
| Adding `"use client"` to `page.tsx` | `page.tsx is server component` test |
| Adding `"use client"` to `lib/api/billing.ts` | `billing.ts no use client` test |
| Dropping portal mutation from `SubscriptionManagementPage` | `preserves portal mutation` test |
| Re-defining `SubscriptionStatus` inline in component | `does NOT re-inline SubscriptionStatus` test |
| Removing `useTiers` from `use-billing.ts` | `useTiers regression guard` test |
| Using `suspense: true` on layout subscription query | `no suspense in layout` test |
| Missing React Fragment in `topbar` slot | `Fragment wraps TopBar + TrialBanner` test |
| Accessing `sessionStorage` outside `useEffect` in `TrialBanner` | SSR safety test |
| Hardcoding English strings in `TrialBanner` | `no hardcoded English strings` test |

---

## Files Modified / Created by This Story

| Path | Action | AC |
|------|--------|----|
| `services/client-api/src/client_api/api/v1/billing.py` | MODIFIED | AC6 |
| `frontend/apps/client/lib/api/billing.ts` | MODIFIED | AC7 |
| `frontend/apps/client/lib/queries/use-billing.ts` | MODIFIED | AC7 |
| `frontend/apps/client/app/[locale]/(protected)/layout.tsx` | MODIFIED | AC5 |
| `frontend/apps/client/app/[locale]/(protected)/settings/billing/page.tsx` | MODIFIED | AC1 |
| `frontend/apps/client/app/[locale]/(protected)/settings/billing/components/SubscriptionManagementPage.tsx` | NEW | AC1/2/3 |
| `frontend/apps/client/app/[locale]/(protected)/components/TrialBanner.tsx` | NEW | AC4 |
| `frontend/apps/client/messages/en.json` | MODIFIED | AC8 |
| `frontend/apps/client/messages/bg.json` | MODIFIED | AC8 |
| `frontend/apps/client/__tests__/subscription-management-s8-13.test.ts` | NEW | AC9 |

---

## GREEN Phase Completion Criteria

All 182 tests must pass. Target breakdown:
- 160 currently RED (new feature assertions)
- 22 currently GREEN (pre-existing file/content checks)

Run to verify GREEN: `cd eusolicit-app/frontend/apps/client && ./node_modules/.bin/vitest run __tests__/subscription-management-s8-13.test.ts`

Expected GREEN output: `Tests  182 passed (182)`
