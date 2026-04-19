---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-19'
workflowType: 'atdd'
story: '8-12-pricing-page-tier-comparison-ui'
tddPhase: 'RED'
inputDocuments:
  - eusolicit-docs/implementation-artifacts/8-12-pricing-page-tier-comparison-ui.md
  - eusolicit-docs/test-artifacts/test-design-epic-08.md
  - _bmad/bmm/config.yaml
  - eusolicit-app/frontend/apps/client/__tests__/opportunities-listing-s6-9.test.ts
---

# ATDD Checklist: Story 8.12 — Pricing Page & Tier Comparison UI

**Date:** 2026-04-19
**Author:** BMad TEA Agent (ATDD Workflow)
**Story:** 8-12-pricing-page-tier-comparison-ui
**Epic:** 08 — Subscription & Billing
**TDD Phase:** 🔴 RED — All tests will FAIL until Story 8.12 is implemented
**Epic Test Design ID:** 8.12-COMP-001 (P2)

---

## Executive Summary

Story 8.12 is primarily a **frontend story** with a minor backend addition. The ATDD approach
uses **structural file-system tests** (Vitest + Node `fs`, no DOM rendering required) matching
the existing pattern established in `__tests__/opportunities-listing-s6-9.test.ts`.

All tests are in **TDD RED phase**: they assert expected file structure, testid presence, i18n
key completeness, and API module exports — none of which exist until the developer implements
the story. Tests will naturally fail on `existsSync` calls for non-existent files and `toContain`
calls on unread content.

---

## Stack Detection

- **Detected stack:** `fullstack`
  - Frontend indicators: `playwright.config.ts`, `vitest.config.ts`, Next.js in `package.json`
  - Backend indicators: `pyproject.toml` in `services/client-api/`
- **Execution mode:** `sequential` (AI generation, no subagent dispatch needed)
- **Test framework:** Vitest (node environment) — matches existing ATDD test pattern

---

## TDD Red Phase Status

✅ **Failing tests generated**

- **Static/Structural Tests:** 75 assertions across 12 `describe` blocks + file structure checks
- All tests will **FAIL** before implementation (files do not exist yet)
- All tests will **PASS** after implementation (when all story files are created/modified)

---

## Generated Test Files

| File | Status | Purpose |
|------|--------|---------|
| `eusolicit-app/frontend/apps/client/__tests__/pricing-page-s8-12.test.ts` | ✅ Created | Primary ATDD test file — all ACs |

---

## Acceptance Criteria Coverage

| AC | Description | Test Group | Test Count | Coverage |
|----|-------------|-----------|------------|---------|
| AC1 | Page shell at `app/[locale]/pricing/page.tsx` — server component, imports PricingPage, not in (protected)/(auth) | `AC1 — Page route shell` | 6 | ✅ Full |
| AC2 | `GET /api/v1/billing/tiers` public endpoint in `billing.py` — TierPolicyResponse, get_tier_policies, no auth | `AC2 — Backend endpoint` | 6 | ✅ Full |
| AC3 | `data-testid="pricing-grid"` + four tier card testids + responsive grid classes | `AC3 — Pricing comparison grid` | 8 | ✅ Full |
| AC4 | `data-testid="pricing-price-{tier}"` + TIER_PRICES constant with 4 tiers + labels | `AC4 — Monthly price display` | 5 | ✅ Full |
| AC5 | Feature matrix testids `pricing-feature-{tier}-{feature}` for 9 features + -1 sentinel | `AC5 — Feature matrix testids` | 5 | ✅ Full |
| AC6 | `data-testid="pricing-badge-popular"` + `ring-2 ring-indigo` on Professional | `AC6 — Most Popular badge` | 5 | ✅ Full |
| AC7 | CTA buttons for all 4 tiers + mailto + target="_blank" + locale links + variant="default" | `AC7 — CTA buttons per tier` | 9 | ✅ Full |
| AC8 | `data-testid="pricing-loading"` + SkeletonCard placeholders + isLoading | `AC8 — Loading skeleton state` | 4 | ✅ Full |
| AC9 | `data-testid="pricing-error"` + EmptyState + refetch() + isError | `AC9 — Error/failure state` | 5 | ✅ Full |
| AC10 | Hero: `pricing-hero`, `pricing-heading`, `pricing-subheading` + i18n keys, no hardcoded strings | `AC10 — Hero section` | 5 | ✅ Full |
| AC11 | 23 `billing.pricing.*` keys in en.json and bg.json; identical key count; no existing key deletion | `AC11 — i18n keys (en.json)` + `AC11 — i18n keys (bg.json)` | 11 | ✅ Full |
| AC12 | `lib/api/billing.ts`: exports getTiers, TIER_PRICES, TIER_ORDER, TierPolicy interface, no "use client"; `lib/queries/use-billing.ts`: exports useTiers, queryKey, staleTime 300_000 | `AC12 — API module` + `AC12 — React Query hook` | 16 | ✅ Full |
| AC13 | ATDD test file covers all ACs (meta-AC) | This checklist + the test file itself | — | ✅ Satisfied |

**Total assertions:** ~75 (across 12 `describe` blocks + file structure + mistake prevention)

---

## Test Strategy

### Test Level Selection

This story uses **static structural tests** (not E2E browser tests) because:

1. The primary risk is **missing files, wrong testids, or missing i18n keys** — detectable without a browser
2. The component and API module structure is fully specified in the story — structural verification is sufficient
3. Browser-level rendering of the pricing grid (8.12-COMP-001) is covered by a **separate component test** described in the epic test design (`8.12-COMP-001: Render page with mocked getTiers() response; assert tier names and feature limits visible via testids`) — that test belongs in green phase

### Priority Mapping

| Test Group | Priority | Rationale |
|-----------|----------|-----------|
| AC1 (route shell) | P2 | Public route placement — critical for unauthenticated access |
| AC2 (backend endpoint) | P1 | Public API without auth must be explicitly verified |
| AC3–AC7 (grid, prices, features, badge, CTAs) | P2 | Core UX — tier comparison is the page's entire purpose |
| AC8–AC9 (loading, error) | P2 | User-facing states; covered by existing SkeletonCard/EmptyState patterns |
| AC10 (hero) | P3 | Decorative; low risk |
| AC11 (i18n) | P1 | Missing i18n keys cause runtime crashes in production |
| AC12 (API module) | P2 | API module shape governs all frontend data fetching |

### Why No E2E Tests in This Red Phase

The story explicitly specifies that `__tests__/pricing-page-s8-12.test.ts` (Vitest structural tests)
is the ATDD deliverable (AC13). The epic test design lists `8.12-COMP-001` as a **component test**
(rendered with mocked `getTiers()` response) — this belongs in the **green phase** integration test
pass, not the red phase structural ATDD.

---

## Implementation Guidance for Developer

### Files to Create

```
eusolicit-app/
├── services/client-api/src/client_api/api/v1/
│   └── billing.py                              MODIFY — add GET /tiers endpoint + TierPolicyResponse
├── frontend/apps/client/
│   ├── app/[locale]/pricing/
│   │   ├── page.tsx                            NEW — server component shell (no "use client")
│   │   └── components/
│   │       └── PricingPage.tsx                 NEW — "use client" pricing component
│   ├── lib/
│   │   ├── api/
│   │   │   └── billing.ts                      NEW — TierPolicy, getTiers(), TIER_PRICES, TIER_ORDER
│   │   └── queries/
│   │       └── use-billing.ts                  NEW — useTiers() with queryKey + staleTime
│   └── messages/
│       ├── en.json                             MODIFY — merge billing.pricing.* keys (23 keys)
│       └── bg.json                             MODIFY — merge billing.pricing.* keys in Bulgarian
```

### Critical Implementation Rules (from story dev notes)

1. **Route placement:** `app/[locale]/pricing/` — NOT inside `(protected)/` or `(auth)/`
2. **Backend endpoint:** `GET /api/v1/billing/tiers` — NO `require_auth()`, NO JWT required
3. **TIER_PRICES:** Hardcoded in frontend, NOT from DB (Free=€0, Starter=€49, Professional=€149, Enterprise="Custom")
4. **-1 sentinel:** Display as `billing.pricing.unlimited` i18n string — never render raw `-1`
5. **Enterprise CTA:** Use `<a href="mailto:...">`, NOT `<Link>` (mailto is not a Next.js route)
6. **`lib/api/billing.ts`:** No `"use client"` — must work in server and client contexts
7. **i18n merging:** Add `pricing` sub-object INTO existing `billing` object — do NOT replace billing keys

---

## Next Steps (TDD Green Phase)

After implementing all story files:

1. Run tests: `cd eusolicit-app/frontend && pnpm test --filter client`
2. Verify all 75 assertions **PASS** (green phase)
3. If any tests fail:
   - **File existence failures** → file not created at correct path
   - **Content failures** → testid, i18n key, or export missing from implementation
   - **i18n count mismatch** → en.json and bg.json have different key counts under `billing.pricing`
4. After all tests green, proceed to component integration test (`8.12-COMP-001`):
   - Render `<PricingPage />` with mocked `getTiers()` via `msw` or `vi.mock`
   - Assert tier names and feature limits visible via testids
5. Commit passing tests with implementation

---

## Epic Test Design Reference

| Test Design ID | Priority | Level | Scenario |
|---------------|----------|-------|---------|
| **8.12-COMP-001** | P2 | Component | Pricing page renders all four tier comparison rows with correct feature limits from API |

The structural ATDD tests in `__tests__/pricing-page-s8-12.test.ts` provide **static verification**
of the component's existence and testid presence. The `8.12-COMP-001` component test (green phase)
provides **runtime verification** by rendering the component with mocked data.

---

## Summary Statistics

```
🔴 TDD RED PHASE: Failing Tests Generated

📊 Test Summary:
  - Structural Tests (Vitest/node):  ~75 assertions in 12 describe blocks
  - E2E Tests:                        0 (deferred to green phase — browser not needed for structural verification)
  - Total Tests:                      ~75
  - TDD Phase:                        RED (all will fail before implementation)

✅ Acceptance Criteria Covered:
  AC1   Page route shell + placement
  AC2   Backend public endpoint
  AC3   Pricing grid + tier card testids
  AC4   Monthly price testids + TIER_PRICES constant
  AC5   Feature matrix testids + -1 sentinel
  AC6   Most Popular badge + ring highlight
  AC7   CTA buttons per tier + mailto + locale links
  AC8   Loading skeleton state
  AC9   Error/retry state
  AC10  Hero section + i18n keys
  AC11  i18n completeness (en + bg, 23 keys each)
  AC12  API module exports + React Query hook
  AC13  This ATDD checklist satisfies the meta-AC

📂 Generated Files:
  eusolicit-app/frontend/apps/client/__tests__/pricing-page-s8-12.test.ts
  eusolicit-docs/test-artifacts/atdd-checklist-8-12-pricing-page-tier-comparison-ui.md

📝 Next Steps:
  1. Developer implements all files listed in "Files to Create" above
  2. Run: cd eusolicit-app/frontend && pnpm test --filter client
  3. Verify all tests PASS (green phase)
  4. Add component integration test for 8.12-COMP-001
  5. Commit passing tests
```
