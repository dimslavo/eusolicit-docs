---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-17'
workflowType: bmad-testarch-atdd
mode: create
storyId: 6-14-upgrade-prompt-modal-tier-gating-ui
storyFile: eusolicit-docs/implementation-artifacts/6-14-upgrade-prompt-modal-tier-gating-ui.md
detectedStack: fullstack
generationMode: ai-generation
tddPhase: RED
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-14-upgrade-prompt-modal-tier-gating-ui.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/frontend/apps/client/__tests__/opportunities-ai-s6-13.test.ts
  - eusolicit-app/frontend/apps/client/vitest.config.ts
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/layout.tsx
  - packages/ui/src/lib/stores/auth-store.ts
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.14 — Upgrade Prompt Modal & Tier Gating UI

**Date:** 2026-04-17
**Author:** BMad TEA Master Test Architect
**TDD Phase:** 🔴 RED — All implementation tests FAIL until S06.14 is implemented
**Story Status:** `ready-for-dev`
**Epic:** E06 | Opportunity Discovery & Intelligence

---

## TDD Red Phase Summary

| Metric | Value |
|--------|-------|
| **Total Tests** | 190 |
| **Failing (RED)** | 181 |
| **Passing (infrastructure exist-checks)** | 9 |
| **Test File** | `eusolicit-app/frontend/apps/client/__tests__/opportunities-upgrade-s6-14.test.ts` |
| **Framework** | Vitest (`environment: 'node'`) |
| **Run Command** | `cd eusolicit-app/frontend/apps/client && npx vitest run __tests__/opportunities-upgrade-s6-14.test.ts` |

### Passing Tests (infrastructure checks — expected to pass even in RED phase)

These 9 tests pass because they verify infrastructure that already exists:

| # | Test | Reason passes |
|---|------|---------------|
| 1 | ATDD test file self-reference | Test file was just written |
| 2 | `en.json` exists | Pre-existing file |
| 3 | `bg.json` exists | Pre-existing file |
| 4 | `app/[locale]/(protected)/layout.tsx` exists | Pre-existing file |
| 5 | `packages/ui/src/lib/stores/auth-store.ts` exists | Pre-existing file |
| 6 | `OpportunityCard.tsx` exists | S06.09 already implemented |
| 7 | `locked-field-budget-{id}` testid in `OpportunityCard.tsx` | S06.09 already implemented |
| 8 | `locked-field-relevance-{id}` testid in `OpportunityCard.tsx` | S06.09 already implemented |
| 9 | `upgrade-prompt-store.ts` does NOT call `apiClient.interceptors` | File doesn't exist → ENOENT caught correctly |

---

## Acceptance Criteria Coverage

| AC | Description | Test Group(s) | Test Count | Priority |
|----|-------------|--------------|-----------|----------|
| **AC1** | `upgrade-prompt-store.ts` — `useUpgradePromptStore`, `UpgradePromptPayload`, `show`/`dismiss`, no `persist` | `AC1 — File structure: upgrade-prompt-store.ts` | 13 | P0 |
| **AC2** | `upgrade-interceptor.ts` — `registerUpgradeInterceptor`, 403/tier_limit, 429/usage_limit_exceeded, `Promise.reject` | `AC2 — File structure: upgrade-interceptor.ts` | 17 | P0 |
| **AC3** | `UpgradeInterceptor.tsx` — `'use client'`, `useEffect`, `[show]` dep, `return null` | `AC3 — File structure: UpgradeInterceptor.tsx` | 12 | P0 |
| **AC4** | `UpgradePromptModal.tsx` — `'use client'`, imports Dialog/useUpgradePromptStore/useAuthStore/useParams | `AC4 — File structure: UpgradePromptModal.tsx` | 9 | P0 |
| **AC5** | Modal renders: current tier, limitation text, tier table, recommended row, CTA, dismiss | `AC5 — dialog`, `AC5 — tier table`, `AC5 — CTA/dismiss` | 25 | P0 |
| **AC6** | Protected layout modified — imports and renders `UpgradeInterceptor` + `UpgradePromptModal` | `AC6 — Protected layout` | 6 | P1 |
| **AC7** | `auth-store.ts` `User` interface has `subscription_tier?: string` | `AC7 — auth-store.ts` | 3 | P1 |
| **AC8** | All 36 `upgradePrompt.*` i18n keys in `en.json` + `bg.json` | `AC8 — i18n en.json`, `AC8 — i18n bg.json` | 82 | P1 |
| **AC10** | All 7 `data-testid` attributes from reference table in `UpgradePromptModal.tsx` | `AC10 — data-testid coverage` | 9 | P0 |
| **AC11** | ATDD test file at correct path | `AC11 — ATDD test file exists` | 1 | P3 |

### Epic Test Design IDs Covered

| Test Design ID | Priority | Scenario | ATDD Coverage |
|---------------|----------|----------|---------------|
| **E06-P1-030** | P1 | Upgrade prompt modal triggered by 403 `tier_limit` | Interceptor checks 403+tier_limit; modal testids; modal dismissible |
| **E06-P2-013** | P2 | Lock icons visible for free-tier users | `OpportunityCard.tsx` lock testids preserved (S06.09 non-regression) |
| **E06-P2-015** | P2 | Upgrade prompt triggered by 429 `usage_limit_exceeded` — recommended tier highlighted | Interceptor checks 429+usage_limit_exceeded; `upgrade-prompt-recommended-tier` testid |
| **E06-P0-010** | P0 | E2E tier enforcement free→locked fields→upgrade prompt | Modal component structure asserted (full E2E flow requires Playwright) |

---

## Test Groups Detail

### Group 1: `upgrade-prompt-store.ts` (AC1) — 13 tests

| Test | What it asserts | Fails because |
|------|-----------------|---------------|
| File exists at `lib/stores/upgrade-prompt-store.ts` | `existsSync` → true | File doesn't exist |
| Exports `useUpgradePromptStore` | Named export pattern | File doesn't exist |
| Exports `UpgradePromptPayload` interface | Export pattern | File doesn't exist |
| `UpgradePromptPayload.errorCode` union type | `"tier_limit"` \| `"usage_limit_exceeded"` | File doesn't exist |
| `UpgradePromptPayload` optional fields | `upgradeUrl`, `used`, `limit` | File doesn't exist |
| Store state has `isOpen: boolean` | Property name | File doesn't exist |
| Store state has `payload: UpgradePromptPayload \| null` | Property name + type | File doesn't exist |
| `show(payload)` action | Action signature | File doesn't exist |
| `show(payload)` sets `isOpen: true` | State update | File doesn't exist |
| `dismiss()` action | Action signature | File doesn't exist |
| `dismiss()` sets `isOpen: false` (preserves payload) | No `payload: null` in dismiss | File doesn't exist |
| No `persist` middleware | Absence check | File doesn't exist |
| Uses Zustand `create()` | `from 'zustand'` | File doesn't exist |

### Group 2: `upgrade-interceptor.ts` (AC2) — 17 tests

| Test | What it asserts | Fails because |
|------|-----------------|---------------|
| File exists at `lib/api/upgrade-interceptor.ts` | `existsSync` → true | File doesn't exist |
| Exports `registerUpgradeInterceptor` | Named export | File doesn't exist |
| Accepts `show` callback | Param signature | File doesn't exist |
| Returns eject cleanup function | `eject` in source | File doesn't exist |
| Imports `apiClient` from `@eusolicit/ui` | Import statement | File doesn't exist |
| Imports `UpgradePromptPayload` from store | Import statement | File doesn't exist |
| Registers via `apiClient.interceptors.response.use` | API call | File doesn't exist |
| Handles HTTP 403 | `403` in source | File doesn't exist |
| Handles `"tier_limit"` error code | String literal | File doesn't exist |
| Calls `show()` with `errorCode: "tier_limit"` | `show({` pattern | File doesn't exist |
| Handles HTTP 429 | `429` in source | File doesn't exist |
| Handles `"usage_limit_exceeded"` error code | String literal | File doesn't exist |
| Calls `show()` with `usage_limit_exceeded` + `used`/`limit` | Payload construction | File doesn't exist |
| Extracts `upgrade_url` from response data | `data.upgrade_url` | File doesn't exist |
| Extracts `used`/`limit` from 429 data | `data.used`, `data.limit` | File doesn't exist |
| Always calls `Promise.reject(error)` | Final return | File doesn't exist |
| `return Promise.reject(error)` present | Exact pattern | File doesn't exist |

### Group 3: `UpgradeInterceptor.tsx` (AC3) — 12 tests

| Test | What it asserts | Fails because |
|------|-----------------|---------------|
| File exists | `existsSync` → true | File doesn't exist |
| `'use client'` directive | Directive present | File doesn't exist |
| `'use client'` before imports | Index comparison | File doesn't exist |
| Named export `UpgradeInterceptor` | Export pattern | File doesn't exist |
| Imports `useEffect` from react | Import | File doesn't exist |
| Imports `registerUpgradeInterceptor` | Import | File doesn't exist |
| Imports `useUpgradePromptStore` | Import | File doesn't exist |
| Reads `show` from store | Usage pattern | File doesn't exist |
| `useEffect` with `[show]` dep and `registerUpgradeInterceptor` | Effect structure | File doesn't exist |
| `useEffect` returns eject directly | `return eject` | File doesn't exist |
| `return null` | JSX return | File doesn't exist |
| No visible JSX elements | No `<div>`/`<span>` | File doesn't exist |

### Group 4: `UpgradePromptModal.tsx` structure (AC4) — 9 tests

| Test | What it asserts | Fails because |
|------|-----------------|---------------|
| File exists | `existsSync` → true | File doesn't exist |
| `'use client'` directive | Directive present | File doesn't exist |
| `'use client'` before imports | Index comparison | File doesn't exist |
| Named export `UpgradePromptModal` | Export pattern | File doesn't exist |
| Imports `Dialog` from `@eusolicit/ui` | Import | File doesn't exist |
| Imports `useUpgradePromptStore` from store | Import | File doesn't exist |
| Imports `useAuthStore` from `@eusolicit/ui` | Import | File doesn't exist |
| Uses `useParams` for locale | Hook usage | File doesn't exist |
| Imports `Link` from `next/link` | Import | File doesn't exist |

### Group 5: Modal dialog behavior (AC5) — 12 tests

Tests covering `isOpen`/`dismiss` binding, limitation text, `tier_limit` vs `usage_limit_exceeded` rendering, fallback `–` for missing `used`/`limit` values.

### Group 6: Tier comparison table (AC5) — 10 tests

Tests covering `TIER_COMPARISON` constant, `RECOMMENDED_TIER` map, 4 tiers (free/starter/professional/enterprise), `isRecommended` flag, indigo highlight styling, column header i18n keys.

### Group 7: CTA and dismiss buttons (AC5) — 6 tests

Tests covering `<Button asChild>` + `<Link>` pattern, `upgradeUrl` fallback to `/${locale}/settings/billing`, `onClick={dismiss}`.

### Group 8: Protected layout (AC6) — 6 tests

Tests covering both imports, both renders, alongside `OnboardingWizard`.

### Group 9: `auth-store.ts` extension (AC7) — 3 tests

Tests covering file existence, `subscription_tier` property, optional string type annotation.

### Group 10: i18n keys (AC8) — 82 tests

- **en.json:** `upgradePrompt` namespace exists, 36 × key-by-key assertions (each key non-empty string), namespace count ≥ 36, `limitationUsageLimitDetail` has `{used}` + `{limit}` params
- **bg.json:** Same structure, plus key parity check vs en.json, `{used}` + `{limit}` preserved in BG translation

### Group 11: data-testid coverage (AC10) — 9 tests

All 7 testids in one bulk assertion + 7 individual `it.each` tests + 1 parent `describe` count check.

### Group 12: Interceptor behavior (E06-P1-030/E06-P2-015) — 5 tests

Combined test group asserting both 403+tier_limit and 429+usage_limit_exceeded interceptor paths, no error swallowing.

### Group 13: Lock icon non-regression (E06-P2-013) — 3 tests

`OpportunityCard.tsx` still exists and still has `locked-field-budget-` and `locked-field-relevance-` testids (S06.09 must not be broken by S06.14).

### Group 14: Architectural constraints — 5 tests

| Test | What it asserts | Purpose |
|------|-----------------|---------|
| Store does NOT call `apiClient.interceptors` | No HTTP in store | Separation of concerns |
| Interceptor does NOT call `useUpgradePromptStore.getState()` | No circular dep | Package boundary |
| `UpgradeInterceptor` only in layout (not per-page) | Single mount | Prevents duplicate interceptors |
| Modal does NOT reference `AISummaryPanel` | Separate upgrade paths | No scope creep |
| Store does NOT import from `packages/ui/src` directly | Use `@eusolicit/ui` alias | Module boundary |

---

## New Files Required by S06.14

| File | Type | Status |
|------|------|--------|
| `apps/client/lib/stores/upgrade-prompt-store.ts` | NEW | ❌ Not created |
| `apps/client/lib/api/upgrade-interceptor.ts` | NEW | ❌ Not created |
| `apps/client/app/[locale]/(protected)/components/UpgradeInterceptor.tsx` | NEW | ❌ Not created |
| `apps/client/app/[locale]/(protected)/components/UpgradePromptModal.tsx` | NEW | ❌ Not created |

## Modified Files Required by S06.14

| File | Change | Status |
|------|--------|--------|
| `packages/ui/src/lib/stores/auth-store.ts` | Add `subscription_tier?: string` to `User` | ❌ Not added |
| `apps/client/app/[locale]/(protected)/layout.tsx` | Import + render `UpgradeInterceptor` + `UpgradePromptModal` | ❌ Not modified |
| `apps/client/messages/en.json` | Add `upgradePrompt` namespace (36 keys) | ❌ Not added |
| `apps/client/messages/bg.json` | Add `upgradePrompt` namespace (36 keys, Bulgarian) | ❌ Not added |

---

## i18n Keys Required (36 keys in `upgradePrompt` namespace)

```json
"upgradePrompt": {
  "modalTitle": "Upgrade Your Plan",
  "currentTierLabel": "Current plan:",
  "tierFree": "Free",
  "tierStarter": "Starter",
  "tierProfessional": "Professional",
  "tierEnterprise": "Enterprise",
  "limitationTierLimit": "You've reached the limit of your current plan",
  "limitationUsageLimitDetail": "{used}/{limit} AI summaries used this month",
  "tierTableTitle": "Compare Plans",
  "colTier": "Plan",
  "colFields": "Fields",
  "colRegions": "Regions",
  "colCpv": "CPV Sectors",
  "colBudget": "Budget",
  "colAiSummaries": "AI Summaries / mo",
  "recommended": "Recommended",
  "ctaButton": "Upgrade Now",
  "dismissButton": "Maybe later",
  "tierDataFieldsFree": "Basic only",
  "tierDataFieldsPaid": "All fields",
  "tierDataRegionsFree": "—",
  "tierDataRegionsStarter": "Bulgaria only",
  "tierDataRegionsProfessional": "BG + 3 EU countries",
  "tierDataRegionsEnterprise": "All EU",
  "tierDataCpvFree": "—",
  "tierDataCpvStarter": "1 sector",
  "tierDataCpvProfessional": "Up to 3 sectors",
  "tierDataCpvEnterprise": "Unlimited",
  "tierDataBudgetFree": "—",
  "tierDataBudgetStarter": "Up to €500K",
  "tierDataBudgetProfessional": "Up to €5M",
  "tierDataBudgetEnterprise": "Unlimited",
  "tierDataAiFree": "0",
  "tierDataAiStarter": "10 / month",
  "tierDataAiProfessional": "50 / month",
  "tierDataAiEnterprise": "Unlimited"
}
```

> ⚠️ The story AC8 says "18 keys" but Dev Notes clarifies the actual count is 36 (the tier table cell data keys were not counted in the initial estimate). The ATDD tests assert all 36 keys.

---

## data-testid Reference Table (7 IDs)

| `data-testid` | Component | Element | AC |
|---|---|---|---|
| `upgrade-prompt-modal` | `UpgradePromptModal` | `DialogContent` root | AC5 |
| `upgrade-prompt-current-tier` | `UpgradePromptModal` | Current plan badge | AC5 |
| `upgrade-prompt-limitation` | `UpgradePromptModal` | Limitation text `<p>` | AC5 |
| `upgrade-prompt-tier-table` | `UpgradePromptModal` | `<table>` element | AC5 |
| `upgrade-prompt-recommended-tier` | `UpgradePromptModal` | Recommended tier `<tr>` | AC5 |
| `upgrade-prompt-cta` | `UpgradePromptModal` | Primary upgrade CTA `<Button>` | AC5 |
| `upgrade-prompt-dismiss` | `UpgradePromptModal` | "Maybe later" `<Button>` | AC5 |

---

## Critical Implementation Notes for Developer

1. **`dismiss()` must NOT set `payload: null`** — the payload must be preserved during the Dialog close animation to prevent a flash where the modal briefly renders with empty content before unmounting.

2. **`registerUpgradeInterceptor` must always `return Promise.reject(error)`** — even after calling `show()`. Swallowing 403/429 errors breaks TanStack Query's error state across the entire app.

3. **`UpgradeInterceptor` must be mounted exactly once** (in protected layout only) — duplicate mounts stack interceptors and cause `show()` to be called multiple times per error response.

4. **`upgrade-interceptor.ts` must receive `show` as a parameter** — it must never call `useUpgradePromptStore.getState()` directly, which would create a circular dependency between `packages/ui` and `apps/client`.

5. **`enterprise` tier maps to `null` in `RECOMMENDED_TIER`** — no row gets the `upgrade-prompt-recommended-tier` testid for enterprise users; the modal renders gracefully without a highlighted row.

6. **Do NOT use `persist` middleware** in `upgrade-prompt-store.ts` — the modal is ephemeral UI state and must not reopen after page reload.

---

## TDD Green Phase Checklist

After implementing Story 6.14, developer should:

- [ ] Create `lib/stores/upgrade-prompt-store.ts` per AC1 spec
- [ ] Create `lib/api/upgrade-interceptor.ts` per AC2 spec
- [ ] Create `app/[locale]/(protected)/components/UpgradeInterceptor.tsx` per AC3 spec
- [ ] Create `app/[locale]/(protected)/components/UpgradePromptModal.tsx` per AC4/5/10 spec
- [ ] Modify `app/[locale]/(protected)/layout.tsx` per AC6 spec
- [ ] Add `subscription_tier?: string` to `User` in `packages/ui/src/lib/stores/auth-store.ts` (AC7)
- [ ] Add 36 `upgradePrompt.*` keys to `messages/en.json` (AC8)
- [ ] Add matching Bulgarian keys to `messages/bg.json` (AC8)
- [ ] Run `pnpm check:i18n` to verify key parity
- [ ] Run ATDD test and verify **0 failing, 190 passing**
- [ ] Run command: `cd eusolicit-app/frontend/apps/client && npx vitest run __tests__/opportunities-upgrade-s6-14.test.ts`

---

## Next Recommended Workflows

- `/bmad-dev-story story-file: eusolicit-docs/implementation-artifacts/6-14-upgrade-prompt-modal-tier-gating-ui.md` — implement the story
- `/bmad-testarch-automate` — expand test coverage with component-level tests after implementation
- `/bmad-sprint-status` — update sprint tracking after story completion

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd` (Create mode)
**Story:** S06.14 — Upgrade Prompt Modal & Tier Gating UI
**Version:** BMad v6
