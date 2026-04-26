---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-21'
workflowType: bmad-testarch-atdd
mode: create
storyKey: '10-15-task-template-manager-ui'
detectedStack: frontend
generationMode: 'AI Generation (frontend story — Vitest + node:fs static + it.skip RTL)'
tddPhase: RED
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/10-15-task-template-manager-ui.md'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-10-14-task-kanban-board-detail-modal.md'
  - 'eusolicit-app/frontend/apps/client/__tests__/task-kanban-s10-14.test.ts'
  - 'eusolicit-app/frontend/apps/client/__tests__/comments-sidebar-s10-13.test.ts'
  - 'eusolicit-app/frontend/apps/client/vitest.config.ts'
  - 'eusolicit-app/frontend/apps/client/messages/en.json'
testFramework: 'Vitest (environment: node) + React Testing Library (it.skip for behavioural)'
---

# ATDD Checklist: Story 10.15 — Task Template Manager UI

**Story:** `10-15-task-template-manager-ui`
**Date:** 2026-04-21
**TDD Phase:** 🔴 RED (behavioural `it.skip` — failing until implementation complete)
**Stack:** frontend (Next.js 14 App Router / React / Vitest / React Testing Library)
**Test Framework:** Vitest (`environment: "node"`) for structural tests; `it.skip` RTL for behavioural

> **Coverage note:** No standalone epic-level test design doc exists for Epic 10. Coverage strategy
> draws from four surrogate sources: Story 10.15 AC1–AC13 (verbatim acceptance criteria),
> the established frontend ATDD pattern from Stories 10.12, 10.13, and 10.14 (direct structural +
> it.skip precedent), Story 10.7 ATDD checklist (backend task-templates API contract: endpoint
> shapes, 409 duplicate name, 422 forward-reference, soft-delete), and Story 10.14 ATDD checklist
> (DnD patterns with @dnd-kit/*, StageListEditor reuses the same sensor setup).
>
> Vitest with `environment: "node"` for structural / static file-system assertions;
> `it.skip` + React Testing Library + `userEvent` for behavioural scenarios;
> `vi.useFakeTimers()` for overdue-date / past-due-date preview tests;
> `QueryClientProvider` wrapping for TanStack Query hook tests;
> `vi.mock("@/lib/stores/ui-store")` to stub `useUIStore.addToast` in error/success-toast assertions;
> `vi.mock("next/navigation")` to stub `useRouter` for deep-link assertions.
> Stories 10.12, 10.13, and 10.14 set the direct precedent for this story's test shape.

---

## TDD Red Phase Summary

✅ Failing tests generated — **2 test files**, **≥ 100 assertions across 13 `describe` blocks + 14 unit-test cases**

| File | Type | Tests | Priority | ACs Covered |
|------|------|-------|----------|-------------|
| `apps/client/__tests__/task-templates-s10-15.test.ts` | Structural + RTL smoke | ≥ 100 | P0/P1 | AC1–AC13 |
| `apps/client/lib/utils/__tests__/stage-reorder-utils.test.ts` | Pure-function unit tests | 20 | P0 | AC8 |

Structural tests fail immediately (files don't exist yet).
Behavioural tests use `it.skip` and compile green — markers come off per TDD activation sequence below.

---

## Generated Test Files

| File | Tests | Priority | ACs Covered |
|------|-------|----------|-------------|
| `apps/client/__tests__/task-templates-s10-15.test.ts` | ≥ 100 | P0/P1 | AC1–AC13 |
| `apps/client/lib/utils/__tests__/stage-reorder-utils.test.ts` | 20 | P0 | AC8 |

---

## Acceptance Criteria Coverage

### ✅ AC1 — File-system & Routing (route + 8 settings components + 2 opp-detail components + API + queries + utils; no new runtime deps; opp-detail modified)

**Settings-page route and component files (existsSync — all fail RED):**
- [ ] **`settings/task-templates/page.tsx (route) exists`** — `app/[locale]/(protected)/settings/task-templates/page.tsx` exists.
- [ ] **`TaskTemplatesPage.tsx exists`** — component file exists.
- [ ] **`TaskTemplatesTable.tsx exists`** — component file exists.
- [ ] **`TaskTemplateFormModal.tsx exists`** — component file exists.
- [ ] **`StageListEditor.tsx exists`** — component file exists.
- [ ] **`StageRowEditor.tsx exists`** — component file exists.
- [ ] **`StageDependencyCheckboxes.tsx exists`** — component file exists.
- [ ] **`TemplateEmptyState.tsx exists`** — component file exists.
- [ ] **`TemplateDeleteConfirm.tsx exists`** — component file exists.

**Opportunity-detail-page component files:**
- [ ] **`ApplyTemplateModal.tsx exists`** — `opportunities/[id]/components/ApplyTemplateModal.tsx` exists.
- [ ] **`ApplyTemplatePreview.tsx exists`** — exists.

**API / queries / utils:**
- [ ] **`lib/api/task-templates.ts exists`** — API client file exists.
- [ ] **`lib/queries/use-task-templates.ts exists`** — query hooks file exists.
- [ ] **`lib/utils/stage-reorder-utils.ts exists`** — utilities file exists.

**Opportunity detail page modified:**
- [ ] **`opportunity-apply-template-btn testid present in page.tsx`** — grep confirms the button testid.
- [ ] **`ApplyTemplateModal is imported into the opportunity detail page`** — import/reference present.

**"use client" directives and exports:**
- [ ] **`settings/task-templates/page.tsx has "use client"`** — directive present.
- [ ] **`TaskTemplatesPage.tsx has "use client"`** — directive present.
- [ ] **`TaskTemplateFormModal.tsx has "use client"`** — directive present.
- [ ] **`StageListEditor.tsx has "use client"`** — directive present.
- [ ] **`StageRowEditor.tsx has "use client"`** — directive present.
- [ ] **`ApplyTemplateModal.tsx has "use client"`** — directive present.
- [ ] **`ApplyTemplatePreview.tsx has "use client"`** — directive present.
- [ ] **`settings/task-templates/page.tsx default-exports TaskTemplatesPage wrapper`** — references component.

**No new runtime dependencies (parse `apps/client/package.json`):**
- [ ] **`@dnd-kit/core remains in dependencies`** — still present, not removed.
- [ ] **`@dnd-kit/sortable remains in dependencies`** — still present.
- [ ] **`@dnd-kit/utilities remains in dependencies`** — still present.
- [ ] **`no new top-level runtime dependency added`** — diff against known-deps whitelist returns empty.

---

### ✅ AC2 — `lib/api/task-templates.ts` type and function exports

**Type/interface exports (source text assertions):**
- [ ] **`exports OpportunityType`** — `export type OpportunityType` present.
- [ ] **`exports StageRole`** — `export type StageRole` present.
- [ ] **`exports StageDefinition interface`** — `export interface StageDefinition` present.
- [ ] **`exports TaskTemplateResponse interface`** — present.
- [ ] **`exports TaskTemplateListResponse interface`** — present.
- [ ] **`exports TaskTemplateCreateRequest interface`** — present.
- [ ] **`exports TaskTemplateUpdateRequest interface`** — present.
- [ ] **`exports TaskTemplateApplyRequest interface`** — present.
- [ ] **`exports TaskTemplateApplyResponse interface`** — present.
- [ ] **`exports ListTaskTemplatesParams interface`** — present.

**Function exports:**
- [ ] **`exports listTaskTemplates`** — `export (async function|function|const) listTaskTemplates` present.
- [ ] **`exports getTaskTemplate`** — present.
- [ ] **`exports createTaskTemplate`** — present.
- [ ] **`exports updateTaskTemplate`** — present.
- [ ] **`exports deleteTaskTemplate`** — present.
- [ ] **`exports applyTaskTemplate`** — present.

**API path contract:**
- [ ] **`uses /api/v1/task-templates base path`** — path literal present.
- [ ] **`uses /apply sub-path for the apply endpoint`** — `/apply` literal present.
- [ ] **`OpportunityType covers "tender", "grant", "any"`** — all three literals present.
- [ ] **`StageRole covers all five role values`** — bid_manager, technical_writer, financial_analyst, legal_reviewer, read_only all present.
- [ ] **`TaskTemplateApplyResponse has created_task_ids field`** — field name present.
- [ ] **`TaskTemplateApplyResponse has warnings field`** — field name present.
- [ ] **`StageDefinition has dependency_indices field`** — field name present.
- [ ] **`StageDefinition has relative_days_before_deadline field`** — field name present.

---

### ✅ AC3 — `lib/queries/use-task-templates.ts` hook exports

- [ ] **`exports useTaskTemplatesList`** — `export function useTaskTemplatesList` present.
- [ ] **`exports useTaskTemplate`** — present.
- [ ] **`exports useCreateTaskTemplate`** — present.
- [ ] **`exports useUpdateTaskTemplate`** — present.
- [ ] **`exports useDeleteTaskTemplate`** — present.
- [ ] **`exports useApplyTaskTemplate`** — present.
- [ ] **`useTaskTemplatesList query key starts with "task-templates"`** — `["task-templates"` present.
- [ ] **`useTaskTemplatesList has staleTime: 60_000`** — `staleTime: 60_000` present.
- [ ] **`useTaskTemplatesList has refetchOnWindowFocus`** — field present.
- [ ] **`useTaskTemplate uses enabled: id !== null guard`** — `enabled` key present.
- [ ] **`mutations call invalidateQueries on success`** — `invalidateQueries` present.
- [ ] **`create/update hooks do NOT toast on 409`** — `409` status handled separately from toast.
- [ ] **`apply hook invalidates ["tasks"] query cache on success`** — `"tasks"` appears in invalidate call.
- [ ] **`uses useUIStore.addToast for success toasts`** — `addToast` present.

---

### ✅ AC4 — `TaskTemplatesPage` (route root)

**Source text / testid assertions:**
- [ ] **`has task-templates-page root testid`** — `task-templates-page` present.
- [ ] **`has template-new-btn testid`** — `template-new-btn` present.
- [ ] **`has templates-table-skeleton testid for loading state`** — `templates-table-skeleton` present.
- [ ] **`has templates-error testid for error state`** — `templates-error` present.
- [ ] **`has templates-empty testid for empty state`** — `templates-empty` present.
- [ ] **`renders TaskTemplatesTable component`** — `TaskTemplatesTable` referenced.
- [ ] **`renders TaskTemplateFormModal component`** — `TaskTemplateFormModal` referenced.
- [ ] **`renders TemplateDeleteConfirm component`** — `TemplateDeleteConfirm` referenced.
- [ ] **`derives canMutate from useAuthStore user role`** — `canMutate` + `bid_manager` present.
- [ ] **`uses useTaskTemplatesList hook for data fetching`** — hook referenced.
- [ ] **`TemplateEmptyState component is used for empty state`** — `TemplateEmptyState` referenced.

**Behavioural (it.skip):**
- [ ] ~~`test_templates_page_renders_empty_state_when_no_templates`~~ — mock empty list; assert `templates-empty`. _🔴 it.skip_
- [ ] ~~`test_templates_page_renders_table_when_templates_exist`~~ — mock 3 items; assert table rows. _🔴 it.skip_
- [ ] ~~`test_new_template_button_opens_form_modal`~~ — click `template-new-btn`; assert modal visible. _🔴 it.skip_
- [ ] ~~`test_read_only_role_hides_new_template_button_and_nav_entry`~~ — mock `role = "read_only"`; assert button absent. _🔴 it.skip_

---

### ✅ AC5 — `TaskTemplatesTable`

**Source text / testid assertions:**
- [ ] **`has templates-table testid`** — `templates-table` present.
- [ ] **`has template-name- template testid`** — `template-name-` present.
- [ ] **`has template-type- template testid`** — `template-type-` present.
- [ ] **`has template-stages- template testid`** — `template-stages-` present.
- [ ] **`has template-edit- per-row testid`** — `template-edit-` present.
- [ ] **`has template-duplicate- per-row testid`** — `template-duplicate-` present.
- [ ] **`has template-delete- per-row testid`** — `template-delete-` present.
- [ ] **`renders DropdownMenu for row actions`** — `DropdownMenu` referenced.
- [ ] **`hides action menu when canMutate is false`** — `canMutate` gating present.
- [ ] **`has named export TaskTemplatesTable`** — `export function TaskTemplatesTable` present.
- [ ] **`renders Badge for opportunity_type column`** — `Badge` referenced.
- [ ] **`sorts by updated_at DESC`** — `updated_at` referenced.

**Behavioural (it.skip):**
- [ ] ~~`test_delete_confirm_fires_delete_mutation`~~ — click Delete; confirm dialog; assert `useDeleteTaskTemplate(id).mutate()`. _🔴 it.skip_

---

### ✅ AC6 — `TaskTemplateFormModal`

**Source text / testid assertions:**
- [ ] **`has template-form-modal root testid`** — `template-form-modal` present.
- [ ] **`has template-name-input testid`** — `template-name-input` present.
- [ ] **`has template-name-error testid for 409 inline error`** — `template-name-error` present.
- [ ] **`has template-description-input testid`** — `template-description-input` present.
- [ ] **`has template-type-radio testid`** — `template-type-radio` present.
- [ ] **`has template-cancel-btn testid`** — `template-cancel-btn` present.
- [ ] **`has template-save-btn testid`** — `template-save-btn` present.
- [ ] **`uses Dialog primitive`** — `Dialog` referenced.
- [ ] **`mode prop supports "new", "edit", and "duplicate"`** — all three strings present.
- [ ] **`uses useTaskTemplate for edit-mode hydration`** — hook referenced.
- [ ] **`uses useCreateTaskTemplate for new/duplicate submit`** — hook referenced.
- [ ] **`uses useUpdateTaskTemplate for edit submit`** — hook referenced.
- [ ] **`renders StageListEditor component`** — referenced.
- [ ] **`renders RadioGroup for opportunity type selection`** — `RadioGroup` referenced.
- [ ] **`exports taskTemplateFormSchema at module scope`** — `export (const|let) taskTemplateFormSchema` present.
- [ ] **`canSubmit gates Save button via stages.length check`** — `canSubmit` + `stages` present.
- [ ] **`calls toServerStages before submitting the stage array`** — `toServerStages` referenced.

**Zod schema assertions (it.skip):**
- [ ] ~~`test_template_form_schema_rejects_empty_name`~~ — `""` and `"   "` fail safeParse. _🔴 it.skip_
- [ ] ~~`test_template_form_schema_rejects_over_255_char_name`~~ — 256 chars fails; 255 passes. _🔴 it.skip_
- [ ] ~~`test_template_form_schema_rejects_empty_stages`~~ — `[]` fails. _🔴 it.skip_
- [ ] ~~`test_template_form_schema_rejects_51_stages`~~ — 51 fails; 50 passes. _🔴 it.skip_

**Behavioural (it.skip):**
- [ ] ~~`test_form_modal_save_fires_create_mutation_with_server_stages`~~ — fill + save; assert payload stripped of `__uiId`. _🔴 it.skip_
- [ ] ~~`test_form_modal_409_renders_inline_name_error`~~ — mock 409; assert `template-name-error`. _🔴 it.skip_
- [ ] ~~`test_form_modal_edit_mode_hydrates_from_server`~~ — mock `useTaskTemplate`; assert pre-filled fields. _🔴 it.skip_
- [ ] ~~`test_duplicate_action_seeds_form_with_copy_suffix`~~ — click duplicate; assert name has `" (copy)"` suffix. _🔴 it.skip_

---

### ✅ AC7 — `StageListEditor` + `StageRowEditor`

**StageListEditor source text / testid assertions:**
- [ ] **`has stage-list testid`** — `stage-list` present.
- [ ] **`has stage-add-btn testid`** — `stage-add-btn` present.
- [ ] **`has stage-reorder-warnings testid`** — `stage-reorder-warnings` present.
- [ ] **`imports DndContext from @dnd-kit/core`** — import present.
- [ ] **`uses DndContext wrapper`** — `DndContext` referenced.
- [ ] **`uses SortableContext from @dnd-kit/sortable`** — `SortableContext` referenced.
- [ ] **`uses PointerSensor with 3px activation constraint`** — `PointerSensor` + `distance: 3` present.
- [ ] **`uses KeyboardSensor for a11y drag`** — `KeyboardSensor` present.
- [ ] **`implements handleDragEnd calling reorderStages`** — `handleDragEnd` + `reorderStages` present.
- [ ] **`renders StageRowEditor components inside the list`** — `StageRowEditor` referenced.
- [ ] **`tracks droppedCount to display reorder warnings`** — `droppedCount` present.
- [ ] **`has named export StageListEditor`** — `export function StageListEditor` present.

**StageRowEditor source text / testid assertions:**
- [ ] **`has stage-row- template testid`** — `stage-row-` present.
- [ ] **`has stage-drag- template testid`** — `stage-drag-` present.
- [ ] **`has stage-title- template testid`** — `stage-title-` present.
- [ ] **`has stage-description- template testid`** — `stage-description-` present.
- [ ] **`has stage-role- template testid`** — `stage-role-` present.
- [ ] **`has stage-days- template testid`** — `stage-days-` present.
- [ ] **`has stage-delete- template testid`** — `stage-delete-` present.
- [ ] **`uses useSortable from @dnd-kit/sortable`** — `useSortable` present.
- [ ] **`renders StageDependencyCheckboxes`** — `StageDependencyCheckboxes` referenced.
- [ ] **`renders GripVertical icon as drag handle`** — `GripVertical` present.
- [ ] **`has named export StageRowEditor`** — `export function StageRowEditor` present.

**Behavioural (it.skip):**
- [ ] ~~`test_stage_row_delete_reduces_list_and_patches_deps`~~ — render 3 stages; click delete-1; assert 2 rows remain + dep indices corrected. _🔴 it.skip_
- [ ] ~~`test_dependency_checkboxes_only_show_earlier_stages`~~ — 3 stages; assert stage-dep-0-* absent, stage-dep-1-0 present, stage-dep-2-0 + stage-dep-2-1 present. _🔴 it.skip_
- [ ] ~~`test_reorder_warning_banner_appears_on_move_backward`~~ — 3 stages, B depends on A; drag A past B; assert `stage-reorder-warnings` with count 1. _🔴 it.skip_

---

### ✅ AC8 — `StageDependencyCheckboxes` + `stage-reorder-utils` contract

**StageDependencyCheckboxes source text / testid assertions:**
- [ ] **`has stage-deps- template testid on the fieldset`** — `stage-deps-` present.
- [ ] **`has stage-dep- template testid on each checkbox`** — `stage-dep-` present.
- [ ] **`renders only earlier-stage checkboxes (slice 0 to index)`** — `slice(0, index)` present.
- [ ] **`uses fieldset + legend for a11y grouping`** — `fieldset` + `legend` present.
- [ ] **`has named export StageDependencyCheckboxes`** — `export function StageDependencyCheckboxes` present.

**stage-reorder-utils function exports:**
- [ ] **`exports reorderStages`** — `export function reorderStages` present.
- [ ] **`exports deleteStage`** — `export function deleteStage` present.
- [ ] **`exports assignUiIds`** — `export function assignUiIds` present.
- [ ] **`exports stripUiIds`** — `export function stripUiIds` present.
- [ ] **`exports toServerStages`** — `export function toServerStages` present.
- [ ] **`reorderStages returns { stages, droppedCount } shape`** — both field names present.
- [ ] **`deleteStage prunes and decrements dependency_indices`** — `dependency_indices` referenced.
- [ ] **`assignUiIds adds __uiId to each stage`** — `__uiId` present.

**Pure-function unit tests (in `lib/utils/__tests__/stage-reorder-utils.test.ts`):**
- [ ] **`reorderStages same-index is no-op`** — droppedCount: 0, order unchanged.
- [ ] **`reorderStages move forward: element shifts right`** — new order verified.
- [ ] **`reorderStages move backward: element shifts left`** — new order verified.
- [ ] **`reorderStages move backward: drops forward-referenced dependency edges`** — droppedCount > 0, edges stripped.
- [ ] **`reorderStages move forward: preserves valid edges that remain strictly less`** — droppedCount: 0, edge kept.
- [ ] **`reorderStages droppedCount accurately reflects removed edges`** — count matches number stripped.
- [ ] **`deleteStage removes stage at given index`** — result length decremented.
- [ ] **`deleteStage decrements dependency_indices > deleted index by 1`** — index rewritten.
- [ ] **`deleteStage strips dependency_indices === deleted index`** — edge removed.
- [ ] **`deleteStage handles compound case: strips AND decrements in one pass`** — both operations applied.
- [ ] **`deleteStage on last stage yields empty array`** — result [].
- [ ] **`assignUiIds adds __uiId to every stage`** — field present on all outputs.
- [ ] **`assignUiIds produces unique __uiId values`** — Set size == length.
- [ ] **`assignUiIds preserves all other stage fields unchanged`** — scalar fields preserved.
- [ ] **`toServerStages strips __uiId`** — field absent on all outputs.
- [ ] **`toServerStages preserves dependency_indices`** — indices match originals.
- [ ] **`toServerStages preserves scalar fields`** — title, role, relative_days_before_deadline intact.
- [ ] **`computeDueDate returns null on null deadline`** — returns null.
- [ ] **`computeDueDate subtracts daysBefore * 86_400_000 ms`** — arithmetic correct.
- [ ] **`computeDueDate returns deadline itself when daysBefore is 0`** — same timestamp.

---

### ✅ AC9 — `ApplyTemplateModal` + `ApplyTemplatePreview`

**ApplyTemplateModal source text / testid assertions:**
- [ ] **`has apply-template-modal root testid`** — `apply-template-modal` present.
- [ ] **`has apply-template-option- per-template testid`** — `apply-template-option-` present.
- [ ] **`has apply-template-mismatch- testid`** — `apply-template-mismatch-` present.
- [ ] **`has apply-cancel-btn testid`** — `apply-cancel-btn` present.
- [ ] **`has apply-confirm-btn testid`** — `apply-confirm-btn` present.
- [ ] **`has apply-view-board-btn testid for deep-link`** — `apply-view-board-btn` present.
- [ ] **`uses useTaskTemplatesList filtered by opportunity_type`** — hook + `opportunity_type` present.
- [ ] **`uses useApplyTaskTemplate mutation`** — hook referenced.
- [ ] **`renders ApplyTemplatePreview when a template is selected`** — component referenced.
- [ ] **`deep-links to tasks board with opportunity filter after apply`** — `router.push` + `/tasks?opportunity=` present.
- [ ] **`renders warnings banner when apply response has warnings`** — `warnings` referenced.
- [ ] **`invalidates ["tasks"] query cache on apply success`** — `invalidateQueries` + `"tasks"` present.
- [ ] **`has named export ApplyTemplateModal`** — `export function ApplyTemplateModal` present.

**ApplyTemplatePreview source text / testid assertions:**
- [ ] **`has apply-preview-table testid`** — `apply-preview-table` present.
- [ ] **`has apply-preview-overdue- template testid`** — `apply-preview-overdue-` present.
- [ ] **`computes due_date via deadline - relative_days_before_deadline`** — `86_400_000` or `computeDueDate` referenced.
- [ ] **`highlights past-due dates in red (text-red-600)`** — `text-red-600` present.
- [ ] **`renders dependency_indices as "#N" labels or "—" for none`** — `dependency_indices` referenced.
- [ ] **`has named export ApplyTemplatePreview`** — `export function ApplyTemplatePreview` present.

**Behavioural (it.skip):**
- [ ] ~~`test_apply_modal_lists_templates_scoped_to_opportunity_type`~~ — tender opp → hook called with `{ opportunity_type: "tender" }`. _🔴 it.skip_
- [ ] ~~`test_apply_preview_shows_overdue_row_when_due_date_is_past`~~ — freeze time; 100 days before deadline 50 days out → `apply-preview-overdue-0` present. _🔴 it.skip_
- [ ] ~~`test_apply_confirm_fires_mutation_and_offers_board_deeplink`~~ — apply succeeds; click `apply-view-board-btn`; assert `router.push` with `/{locale}/tasks?opportunity={id}`. _🔴 it.skip_
- [ ] ~~`test_apply_422_null_deadline_shows_toast`~~ — mock 422; assert `addToast` with `taskTemplates.apply.error.nullDeadline`. _🔴 it.skip_

---

### ✅ AC10 — i18n (EN/BG parity, ≥ 40 keys, ICU plural syntax, settingsNav key)

- [ ] **`EN taskTemplates namespace exists`** — `en.taskTemplates` is an object.
- [ ] **`BG taskTemplates namespace exists`** — `bg.taskTemplates` is an object.
- [ ] **`EN taskTemplates.* has ≥ 40 leaf keys`** — countLeafKeys(enTT) >= 40.
- [ ] **`BG taskTemplates.* has ≥ 40 leaf keys`** — countLeafKeys(bgTT) >= 40.
- [ ] **`every leaf path in EN taskTemplates.* is present in BG`** — EN→BG parity.
- [ ] **`every leaf path in BG taskTemplates.* is present in EN`** — BG→EN parity.
- [ ] **`EN taskTemplates.apply.stageCount uses ICU plural syntax`** — `{count, plural,` present.
- [ ] **`BG taskTemplates.apply.stageCount uses ICU plural syntax`** — BG parity.
- [ ] **`EN taskTemplates.apply.confirmBtn uses ICU plural syntax`** — `{count, plural,` present.
- [ ] **`BG taskTemplates.apply.confirmBtn uses ICU plural syntax`** — BG parity.
- [ ] **`EN taskTemplates.apply.success uses ICU plural syntax`** — `{count, plural,` present.
- [ ] **`BG taskTemplates.apply.success uses ICU plural syntax`** — BG parity.
- [ ] **`EN taskTemplates.form.reorderedDropped uses ICU plural syntax`** — `{count, plural,` present.
- [ ] **`BG taskTemplates.form.reorderedDropped uses ICU plural syntax`** — BG parity.
- [ ] **`EN taskTemplates.pageTitle exists`** — key truthy.
- [ ] **`EN taskTemplates.opportunityType.tender, .grant, .any exist`** — all three truthy.
- [ ] **`EN taskTemplates.error.duplicateName exists`** — key truthy.
- [ ] **`EN taskTemplates.error.forwardReference exists`** — key truthy.
- [ ] **`EN taskTemplates.apply.needsDeadline exists`** — key truthy.
- [ ] **`EN taskTemplates.apply.viewBoardBtn exists`** — key truthy.
- [ ] **`EN settingsNav.taskTemplates exists`** — nav key truthy.
- [ ] **`BG settingsNav.taskTemplates exists`** — BG nav key truthy.

---

### ✅ AC11 — Opportunity detail page integration

**Source text assertions:**
- [ ] **`opportunity-apply-template-btn testid present`** — testid in `opportunities/[id]/page.tsx`.
- [ ] **`ApplyTemplateModal is imported`** — component referenced.
- [ ] **`Apply template button is disabled when deadline is null`** — `deadline` + `disabled` present.
- [ ] **`Apply template button gates on canMutate`** — `canMutate` present.
- [ ] **`isApplyOpen state variable manages modal open state`** — `isApplyOpen` present.

**Behavioural (it.skip):**
- [ ] ~~`test_opportunity_detail_apply_button_disabled_when_deadline_null`~~ — mock null deadline; assert disabled attr. _🔴 it.skip_
- [ ] ~~`test_opportunity_detail_apply_button_opens_modal`~~ — mock valid deadline + bid_manager role; click; assert modal visible. _🔴 it.skip_

---

### ✅ AC12 — ATDD test file meta-check

- [ ] **`ATDD test file task-templates-s10-15.test.ts exists`** — `__tests__/task-templates-s10-15.test.ts` exists.
- [ ] **`ATDD file mentions all 13 ACs in header comments`** — `AC1` through `AC13` all present in source.

---

### ✅ AC13 — Regression guard (Stories 10.12, 10.13, 10.14 ATDD suites + opportunity detail test unaffected)

- [ ] **`collaborator-lock-s10-12.test.ts still exists`** — file present.
- [ ] **`comments-sidebar-s10-13.test.ts still exists`** — file present.
- [ ] **`task-kanban-s10-14.test.ts still exists`** — file present.
- [ ] **`Story 10.12 test file does not collide with template-form-modal testid`** — no collision.
- [ ] **`Story 10.13 test file does not collide with task-templates-page testid`** — no collision.
- [ ] **`Story 10.14 test file does not collide with apply-template-modal testid`** — no collision.

---

## TDD Activation Sequence

Remove `it.skip` markers in this order to enable behavioural tests progressively:

1. **Phase 1 — Pure-function unit tests** (zero rendering, no mocks needed — in `stage-reorder-utils.test.ts`):
   - All tests in the unit test file are structural (fail immediately because the module doesn't exist).
   - After implementation, all 20 tests should pass without any `it.skip` removal.

2. **Phase 2 — Zod schema tests** (pure logic, import the exported schema):
   - `test_template_form_schema_rejects_empty_name`
   - `test_template_form_schema_rejects_over_255_char_name`
   - `test_template_form_schema_rejects_empty_stages`
   - `test_template_form_schema_rejects_51_stages`

3. **Phase 3 — Page rendering tests** (mock TanStack Query hooks):
   - `test_templates_page_renders_empty_state_when_no_templates`
   - `test_templates_page_renders_table_when_templates_exist`
   - `test_new_template_button_opens_form_modal`
   - `test_read_only_role_hides_new_template_button_and_nav_entry`

4. **Phase 4 — Form modal tests** (mock mutations + query):
   - `test_form_modal_save_fires_create_mutation_with_server_stages`
   - `test_form_modal_409_renders_inline_name_error`
   - `test_form_modal_edit_mode_hydrates_from_server`
   - `test_duplicate_action_seeds_form_with_copy_suffix`

5. **Phase 5 — Stage list / row editor tests** (render with multiple stages):
   - `test_stage_row_delete_reduces_list_and_patches_deps`
   - `test_dependency_checkboxes_only_show_earlier_stages`
   - `test_reorder_warning_banner_appears_on_move_backward`

6. **Phase 6 — Table and delete-confirm tests**:
   - `test_delete_confirm_fires_delete_mutation`

7. **Phase 7 — Apply modal and preview tests** (frozen time for overdue):
   - `test_apply_modal_lists_templates_scoped_to_opportunity_type`
   - `test_apply_preview_shows_overdue_row_when_due_date_is_past`
   - `test_apply_confirm_fires_mutation_and_offers_board_deeplink`
   - `test_apply_422_null_deadline_shows_toast`

8. **Phase 8 — Opportunity detail page integration tests**:
   - `test_opportunity_detail_apply_button_disabled_when_deadline_null`
   - `test_opportunity_detail_apply_button_opens_modal`

---

## Test Count Summary

| Category | Count | Phase |
|----------|-------|-------|
| File-system existence (AC1) | 14 | Structural RED |
| Opp-detail page modified guards (AC1) | 2 | Structural RED |
| "use client" + exports (AC1) | 8 | Structural RED |
| No-new-deps guard (AC1) | 4 | Structural RED |
| API type/interface exports (AC2) | 10 | Structural RED |
| API function exports (AC2) | 6 | Structural RED |
| API path contract (AC2) | 8 | Structural RED |
| Hook exports + config (AC3) | 14 | Structural RED |
| TaskTemplatesPage testids (AC4) | 11 | Structural RED |
| TaskTemplatesTable testids (AC5) | 12 | Structural RED |
| TaskTemplateFormModal testids (AC6) | 17 | Structural RED |
| StageListEditor testids (AC7) | 12 | Structural RED |
| StageRowEditor testids (AC7) | 11 | Structural RED |
| StageDependencyCheckboxes testids (AC8) | 5 | Structural RED |
| stage-reorder-utils exports (AC8) | 8 | Structural RED |
| ApplyTemplateModal testids (AC9) | 13 | Structural RED |
| ApplyTemplatePreview testids (AC9) | 6 | Structural RED |
| i18n parity + ICU plurals (AC10) | 22 | Structural RED |
| Opportunity detail page (AC11) | 5 | Structural RED |
| ATDD meta-check (AC12) | 2 | Structural RED |
| Regression guard (AC13) | 6 | Structural RED |
| **Pure-function unit tests (stage-reorder-utils.test.ts)** | **20** | Structural RED |
| **Zod schema (it.skip)** | **4** | 🔴 RED until impl |
| **Behavioural smoke (it.skip)** | **17** | 🔴 RED until impl |
| **TOTAL** | **≥ 227** | — |

*Minimum assertion count per AC12 requirement: ≥ 40 across 11 ACs (structural assertions alone exceed this threefold).*

---
