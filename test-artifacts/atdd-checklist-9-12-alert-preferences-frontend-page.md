---
stepsCompleted: ['step-01-load-story', 'step-02-load-epic-test-design', 'step-03-map-acs', 'step-04-generate-tests', 'step-05-write-output']
lastStep: 'step-05-write-output'
lastSaved: '2026-04-20'
workflowType: 'testarch-atdd'
mode: 'story-level'
storyKey: '9-12-alert-preferences-frontend-page'
epicNumber: 9
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/9-12-alert-preferences-frontend-page.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-09.md'
  - 'eusolicit-app/frontend/apps/client/__tests__/grant-tools-s11-11.test.ts'
  - 'eusolicit-app/frontend/apps/client/__tests__/subscription-management-s8-13.test.ts'
testFile: 'eusolicit-app/frontend/apps/client/__tests__/alert-preferences-s9-12.test.ts'
status: 'RED — test file written; all tests fail until story is implemented'
---

# ATDD Checklist — Story 9.12: Alert Preferences Frontend Page

**Story:** S09.12 — Alert Preferences Frontend Page  
**Epic:** E09 — Notifications, Alerts & Calendar  
**Story Points:** 3  
**Type:** Frontend  
**Date:** 2026-04-20  
**Status:** 🔴 RED — failing acceptance tests written; implementation not yet started  

---

## Coverage Strategy (from Epic Test Design)

Source: `eusolicit-docs/test-artifacts/test-design-epic-09.md` — rows scoped to S09.12.

| Priority | Epic Test Design Row | Level | Count | Owner | Notes |
|---|---|---|---|---|---|
| **P1** | Alert preferences page renders existing preferences as editable cards; 5-preference limit message shown correctly | Component | 3 | Dev | Testing Library (TEA phase); ATDD here validates static artefacts |
| **P1** | Schedule selector; enable/disable toggle calls PATCH with only `is_active` field | Component | 3 | Dev | Deferred to TEA; ATDD here validates source structure |
| **P2** | Delete preference — confirm dialog shown; cancel does not call DELETE API | Component | 2 | Dev | Deferred to TEA; ATDD validates AlertDialog testid presence |
| **P3** | CPV multi-select search renders 10,000 codes without UI freeze (< 2s render) | Component/Perf | 1 | Dev | Deferred to TEA (requires JSDOM + full CPV JSON) |

**ATDD scope for this story (static, `environment: 'node'`):**  
File existence, export surface, `data-testid` string presence in source, i18n key parity across both locales.  
The 9 component-render tests (P1/P2/P3) are deferred to the TEA `*automate` phase — consistent with Stories 6.x, 7.x, 8.x, 11.x, 12.x precedent.

**Coverage gate from epic test design:**  
Frontend components (alert preferences page): ≥ 70% statement coverage — to be verified during TEA.

---

## Acceptance Criteria → Test Mapping

### AC1 — Route + page scaffold

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 1.1 | `settings/alerts/page.tsx` exists | File existence | ✅ exists |
| 1.2 | `page.tsx` imports `AlertPreferencesPage` | Source grep | contains `AlertPreferencesPage` |
| 1.3 | `settings/alerts/components/AlertPreferencesPage.tsx` exists | File existence | ✅ exists |
| 1.4 | `AlertPreferencesPage.tsx` has `"use client"` directive | Source grep | starts with `"use client"` |
| 1.5 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-page"` | Source grep | string present |
| 1.6 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-page-title"` | Source grep | string present |

---

### AC2 — List rendering + empty / loading states

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 2.1 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-loading"` | Source grep | string present |
| 2.2 | Loading state renders `<Skeleton>` components | Source grep | contains `Skeleton` |
| 2.3 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-empty"` | Source grep | string present |
| 2.4 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-empty-create-btn"` | Source grep | string present |
| 2.5 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-error"` | Source grep | string present |
| 2.6 | Error state uses `Alert` with `variant="destructive"` | Source grep | contains `variant="destructive"` |

---

### AC3 — 5-preference limit messaging

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 3.1 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-create-btn"` | Source grep | string present |
| 3.2 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-counter"` | Source grep | string present |
| 3.3 | `AlertPreferencesPage.tsx` has `data-testid="alert-preferences-limit-tooltip"` | Source grep | string present |
| 3.4 | Create button exposes `aria-disabled` attribute when at limit | Source grep | contains `aria-disabled` |

---

### AC4 — Card display

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 4.1 | `settings/alerts/components/AlertPreferenceCard.tsx` exists | File existence | ✅ exists |
| 4.2 | Card has `data-testid` with `alert-preference-card-${id}-schedule` pattern | Source grep | template literal present |
| 4.3 | Card has `data-testid` with `alert-preference-card-${id}-toggle` pattern | Source grep | template literal present |
| 4.4 | Card has `data-testid` with `alert-preference-card-${id}-edit-btn` pattern | Source grep | template literal present |
| 4.5 | Card has `data-testid` with `alert-preference-card-${id}-delete-btn` pattern | Source grep | template literal present |
| 4.6 | Card has `data-testid` with `alert-preference-card-${id}-delete-dialog` pattern | Source grep | template literal present |
| 4.7 | Card uses `<Switch>` from `@eusolicit/ui` | Source grep | contains `Switch` |

---

### AC5 — Form drawer for create / edit

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 5.1 | `settings/alerts/components/AlertPreferenceFormDrawer.tsx` exists | File existence | ✅ exists |
| 5.2 | Drawer has `data-testid="alert-preference-form-drawer"` | Source grep | string present |
| 5.3 | Drawer has `data-testid="alert-preference-form-cancel-btn"` | Source grep | string present |
| 5.4 | Drawer has `data-testid="alert-preference-form-submit-btn"` | Source grep | string present |
| 5.5 | Submit button is `disabled` while mutation pending | Source grep | contains `disabled` and `pending` |
| 5.6 | Drawer uses `<Sheet>` from `@eusolicit/ui` | Source grep | contains `Sheet` |

---

### AC6 — CPV multi-select sub-component

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 6.1 | `settings/alerts/components/CpvMultiSelect.tsx` exists | File existence | ✅ exists |
| 6.2 | `CpvMultiSelect.tsx` has `data-testid="alert-preference-cpv-select"` | Source grep | string present |
| 6.3 | `CpvMultiSelect.tsx` has `data-testid="alert-preference-cpv-search"` | Source grep | string present |
| 6.4 | `CpvMultiSelect.tsx` has chip testid pattern `alert-preference-cpv-chip-` | Source grep | template literal present |
| 6.5 | `CpvMultiSelect.tsx` uses `react-window` for virtualization | Source grep | contains `react-window` or `FixedSizeList` |

---

### AC7 — EU region checkbox groups

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 7.1 | `settings/alerts/components/RegionCheckboxGroups.tsx` exists | File existence | ✅ exists |
| 7.2 | `RegionCheckboxGroups.tsx` has `data-testid="alert-preference-regions"` | Source grep | string present |
| 7.3 | Select-all testid pattern `alert-preference-region-group-select-all-` present | Source grep | template literal present |
| 7.4 | Country checkbox testid pattern `alert-preference-region-checkbox-` present | Source grep | template literal present |
| 7.5 | All five geographic group keys present: `western`, `eastern`, `northern`, `southern`, `other` | Source grep | each string present |
| 7.6 | Country code `XI` is NOT in the region checkboxes (excluded per AC7) | Source grep | `XI` absent from region list |

---

### AC8 / AC9 / AC10 — Budget / Deadline / Schedule sub-components

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 8.1 | `settings/alerts/components/BudgetRangeInput.tsx` exists | File existence | ✅ exists |
| 8.2 | `BudgetRangeInput.tsx` has `data-testid="alert-preference-budget"` | Source grep | string present |
| 8.3 | `BudgetRangeInput.tsx` has `data-testid="alert-preference-budget-min"` | Source grep | string present |
| 8.4 | `BudgetRangeInput.tsx` has `data-testid="alert-preference-budget-max"` | Source grep | string present |
| 9.1 | `settings/alerts/components/DeadlineSlider.tsx` exists | File existence | ✅ exists |
| 9.2 | `DeadlineSlider.tsx` has `data-testid="alert-preference-deadline"` | Source grep | string present |
| 9.3 | `DeadlineSlider.tsx` has `data-testid="alert-preference-deadline-value"` | Source grep | string present |
| 9.4 | `DeadlineSlider.tsx` range is 1–90 with default 30 | Source grep | contains `1` and `90` and `30` |
| 10.1 | `settings/alerts/components/ScheduleRadioGroup.tsx` exists | File existence | ✅ exists |
| 10.2 | `ScheduleRadioGroup.tsx` has `data-testid="alert-preference-schedule"` | Source grep | string present |
| 10.3 | Schedule item testid pattern `alert-preference-schedule-` present for all 3 values | Source grep | `immediate`, `daily`, `weekly` present |
| 10.4 | Default schedule is `"daily"` | Source grep | contains `"daily"` as default |

---

### AC11 — Form validation (zod)

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 11.1 | `lib/schemas/alert-preference.ts` exists | File existence | ✅ exists |
| 11.2 | Exports `alertPreferenceFormSchema` | Source grep | string present |
| 11.3 | Exports `EU_COUNTRY_CODES` | Source grep | string present |
| 11.4 | Exports `AlertPreferenceFormValues` type | Source grep | string present |
| 11.5 | CPV regex `/^\d{8}(-\d)?$/` is used | Source grep | regex pattern present |
| 11.6 | `deadline_days_ahead` constrained to 1–90 | Source grep | `.min(1).max(90)` present |
| 11.7 | Budget cross-field check: `budget_max > budget_min` | Source grep | contains refinement / superRefine logic |
| 11.8 | Form error testid pattern `alert-preference-form-error-` present in drawer | Source grep | template literal present |

---

### AC13 — Sidebar navigation entry

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 13.1 | `app/[locale]/(protected)/layout.tsx` imports `Bell` from `lucide-react` | Source grep | contains `Bell` and `lucide-react` |
| 13.2 | Layout has `testId: "sidebar-nav-alerts"` in nav items | Source grep | contains `sidebar-nav-alerts` |
| 13.3 | Layout nav item href contains `settings/alerts` | Source grep | contains `settings/alerts` |

---

### AC14 — i18n keys

All keys below must be present in both `messages/en.json` and `messages/bg.json` with non-empty string values.

**Required keys (47 total leaf keys):**

```
alerts.preferences.pageTitle
alerts.preferences.pageDescription
alerts.preferences.emptyTitle
alerts.preferences.emptyDescription
alerts.preferences.createBtn
alerts.preferences.counter
alerts.preferences.limitReached
alerts.preferences.limitReachedToast
alerts.preferences.scheduleLabel.immediate
alerts.preferences.scheduleLabel.daily
alerts.preferences.scheduleLabel.weekly
alerts.preferences.scheduleDescription.immediate
alerts.preferences.scheduleDescription.daily
alerts.preferences.scheduleDescription.weekly
alerts.preferences.cpvLabel
alerts.preferences.cpvSearchPlaceholder
alerts.preferences.cpvNoResults
alerts.preferences.cpvSelectedCount
alerts.preferences.regionsLabel
alerts.preferences.regionGroupLabel.western
alerts.preferences.regionGroupLabel.eastern
alerts.preferences.regionGroupLabel.northern
alerts.preferences.regionGroupLabel.southern
alerts.preferences.regionGroupLabel.other
alerts.preferences.regionSelectAll
alerts.preferences.budgetLabel
alerts.preferences.budgetMinLabel
alerts.preferences.budgetMaxLabel
alerts.preferences.budgetUnit
alerts.preferences.deadlineLabel
alerts.preferences.deadlineDays
alerts.preferences.activeLabel
alerts.preferences.inactiveLabel
alerts.preferences.editBtn
alerts.preferences.deleteBtn
alerts.preferences.cancelBtn
alerts.preferences.saveBtn
alerts.preferences.confirmDeleteTitle
alerts.preferences.confirmDeleteDescription
alerts.preferences.confirmDeleteBtn
alerts.preferences.created
alerts.preferences.updated
alerts.preferences.deleted
alerts.preferences.toggleFailed
alerts.preferences.genericError
alerts.preferences.validation.budgetMaxGtMin
alerts.preferences.validation.cpvFormat
alerts.preferences.validation.regionInvalid
alerts.preferences.validation.deadlineRange
nav.alerts
```

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 14.1 | `messages/en.json` exists | File existence | ✅ exists |
| 14.2 | `messages/bg.json` exists | File existence | ✅ exists |
| 14.3 | en.json has `alerts.preferences` namespace | JSON key check | defined |
| 14.4 | bg.json has `alerts.preferences` namespace | JSON key check | defined |
| 14.5 | en.json has all 49 required leaf keys | JSON key check (it.each) | all defined and non-empty strings |
| 14.6 | bg.json has all 49 required leaf keys | JSON key check (it.each) | all defined and non-empty strings |
| 14.7 | en.json and bg.json have the same key count under `alerts.preferences` | Structural parity | counts match |

---

### AC15 — Optimistic toggle UX

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 15.1 | `lib/api/alerts.ts` exists | File existence | ✅ exists |
| 15.2 | `lib/api/alerts.ts` exports `AlertPreferenceRecord` type | Source grep | string present |
| 15.3 | `lib/api/alerts.ts` exports `AlertSchedule` type | Source grep | string present |
| 15.4 | `lib/api/alerts.ts` exports `AlertPreferencePayload` type | Source grep | string present |
| 15.5 | `lib/api/alerts.ts` exports `listAlertPreferences` function | Source grep | contains `listAlertPreferences` |
| 15.6 | `lib/api/alerts.ts` exports `toggleAlertPreference` function | Source grep | contains `toggleAlertPreference` |
| 15.7 | `lib/queries/use-alerts.ts` exists | File existence | ✅ exists |
| 15.8 | `lib/queries/use-alerts.ts` has `"use client"` directive | Source grep | starts with `"use client"` |
| 15.9 | `lib/queries/use-alerts.ts` exports `useAlertPreferences` hook | Source grep | contains `useAlertPreferences` |
| 15.10 | `lib/queries/use-alerts.ts` exports `useToggleAlertPreference` hook | Source grep | contains `useToggleAlertPreference` |
| 15.11 | `useToggleAlertPreference` uses `onMutate` for optimistic update | Source grep | contains `onMutate` |
| 15.12 | `useToggleAlertPreference` uses `onError` for rollback | Source grep | contains `onError` |
| 15.13 | All mutations use `["alerts", "preferences"]` query key | Source grep | contains `"alerts"` and `"preferences"` |

---

### AC16 — ATDD coverage (static)

| # | Check | Type | Expected outcome |
|---|---|---|---|
| 16.1 | `__tests__/alert-preferences-s9-12.test.ts` exists | File existence | ✅ exists |
| 16.2 | Test file has ≥ 18 `it(...)` blocks | Structural | count ≥ 18 |

---

## Test File Location

```
eusolicit-app/frontend/apps/client/__tests__/alert-preferences-s9-12.test.ts
```

**Run command:**
```bash
cd eusolicit-app/frontend && pnpm test --filter client -- alert-preferences-s9-12
```

---

## Coverage Summary

| AC | Describe Block | Static Checks | Component (TEA) |
|---|---|---|---|
| AC1 | Route + page scaffold | 6 | — |
| AC2 | List rendering states | 6 | 3 (P1) |
| AC3 | 5-preference limit messaging | 4 | — |
| AC4 | Card display | 7 | 2 (P2) |
| AC5 | Form drawer | 6 | — |
| AC6 | CPV multi-select | 5 | 1 (P3 perf) |
| AC7 | Region checkbox groups | 6 | — |
| AC8/9/10 | Budget/deadline/schedule | 12 | 3 (P1) |
| AC11 | zod schema | 8 | — |
| AC13 | Sidebar nav entry | 3 | — |
| AC14 | i18n parity | 7 + it.each | — |
| AC15 | API module + optimistic toggle | 13 | — |
| AC16 | Test file structure | 2 | — |
| **Total** | | **≥ 48 `it(...)` blocks** | **9 deferred to TEA** |

---

## Pass Criteria

- 🔴 **RED** (now): All static `it(...)` blocks fail because none of the target files exist.
- 🟢 **GREEN** (after implementation): All static `it(...)` blocks pass. Developer may not mark story `done` until this file is fully green.
- TEA gate: ≥ 70% component coverage achieved via the 9 deferred component tests in the TEA `*automate` phase.

---

## Related Artefacts

- Story file: `eusolicit-docs/implementation-artifacts/9-12-alert-preferences-frontend-page.md`
- Epic test design: `eusolicit-docs/test-artifacts/test-design-epic-09.md`
- ATDD precedent (S11.11): `eusolicit-app/frontend/apps/client/__tests__/grant-tools-s11-11.test.ts`
- ATDD precedent (S8.13): `eusolicit-app/frontend/apps/client/__tests__/subscription-management-s8-13.test.ts`
- i18n parity test: `eusolicit-app/frontend/apps/client/__tests__/i18n-setup.test.ts`
