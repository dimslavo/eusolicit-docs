# Story 11.14: Compliance Admin — Framework Management Frontend

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **platform administrator on EU Solicit**,
I want a dedicated compliance administration section at `/compliance` in the admin app where I can create, list, search, filter, edit, activate/deactivate, and delete compliance frameworks with structured validation rules,
so that I can define and maintain the regulatory rule sets (national, EU, and programme-level) that govern procurement opportunities across the platform — including a live rules editor with preview validation against a sample proposal.

## Acceptance Criteria

### Route Structure

1. **AC1** — Three new pages are created under `apps/admin/app/[locale]/(protected)/compliance/`:
   - `apps/admin/app/[locale]/(protected)/compliance/page.tsx` — Framework List page (server component shell; renders `<FrameworkList />`)
   - `apps/admin/app/[locale]/(protected)/compliance/new/page.tsx` — Create Framework page (server component shell; renders `<FrameworkEditor mode="create" />`)
   - `apps/admin/app/[locale]/(protected)/compliance/[id]/edit/page.tsx` — Edit Framework page (server component shell; renders `<FrameworkEditor mode="edit" frameworkId={params.id} />`)

   All three page files use `"use client"` only on their inner components (not on the page shells themselves). Each page shell is a minimal server component wrapper that imports and renders its client component.

---

### Navigation Update

2. **AC2** — The admin sidebar navigation in `apps/admin/app/[locale]/(protected)/layout.tsx` gains a **Compliance** entry. The `adminNavItems` array receives a new element (inserted after the existing Dashboard entry): `{ icon: ShieldCheck, label: t("nav.compliance"), href: \`/${locale}/compliance\` }`. Import `ShieldCheck` from `"lucide-react"`. The `"nav.compliance"` key is added to both `apps/admin/messages/en.json` and `apps/admin/messages/bg.json` under the `"nav"` namespace (see AC10 for values).

---

### Framework List Page

3. **AC3** — `FrameworkList` component (`apps/admin/app/[locale]/(protected)/compliance/components/FrameworkList.tsx`) is a `"use client"` component. On mount it calls `useComplianceFrameworks(queryParams)` (a TanStack Query `useQuery` hook). While loading, it renders `<SkeletonTable rows={5} data-testid="compliance-list-skeleton" />` from `@eusolicit/ui`. The root element has `data-testid="compliance-framework-list-page"`.

4. **AC4** — Above the table, a page header section (`data-testid="compliance-page-header"`) renders:
   - `<h1 data-testid="compliance-page-title" className="text-2xl font-bold text-slate-900">t("compliance.pageTitle")</h1>`
   - `<p data-testid="compliance-page-description" className="text-sm text-slate-500 mt-1">t("compliance.pageDescription")</p>`
   - `<Button data-testid="compliance-create-btn" onClick={() => router.push(\`/${locale}/compliance/new\`)}>t("compliance.createBtn")</Button>`

5. **AC5** — Below the page header, a **filter and search bar** section (`data-testid="compliance-filters-section"`) renders:
   - **Search input** — `<Input data-testid="compliance-search-input" placeholder={t("compliance.list.searchPlaceholder")} />` with a debounced `useState<string>("")` for `searchQuery` (300 ms debounce via `useEffect`). The search filters the framework list client-side by matching `framework.name.toLowerCase().includes(searchQuery.toLowerCase())`.
   - **Country filter** — `<Select data-testid="compliance-filter-country">` with a `<SelectItem value="">` for "All countries" plus one `<SelectItem>` per unique country in the loaded dataset. Managed via `useState<string>("")` for `filterCountry`. Triggers re-query when changed.
   - **Regulation type filter** — Three checkboxes (`data-testid="compliance-filter-type-national"`, `data-testid="compliance-filter-type-eu"`, `data-testid="compliance-filter-type-programme"`) for multi-select filtering. Labels: `t("compliance.type.national")`, `t("compliance.type.eu")`, `t("compliance.type.programme")`. Managed via `useState<Set<string>>(new Set())` for `filterTypes`. When no types are checked, all types show.
   - **Status filter** — `<Select data-testid="compliance-filter-status">` with options: `<SelectItem value="all">` `t("compliance.list.statusAll")`, `<SelectItem value="active">` `t("compliance.list.statusActive")`, `<SelectItem value="inactive">` `t("compliance.list.statusInactive")`. Managed via `useState<"all" | "active" | "inactive">("all")` for `filterStatus`. Triggers re-query when changed.

6. **AC6** — On query success with at least one framework (after client-side filtering), a `<table data-testid="compliance-framework-table">` renders with:
   - **Header row** with six columns: `t("compliance.list.colName")`, `t("compliance.list.colCountry")`, `t("compliance.list.colType")`, `t("compliance.list.colStatus")`, `t("compliance.list.colCreated")`, `t("compliance.list.colActions")`
   - **One body row per framework** (`data-testid="framework-row-{id}"`), containing:
     - **Name cell** — `<td data-testid="framework-name-{id}" className="font-medium text-slate-900">` — displays `framework.name`
     - **Country cell** — `<td data-testid="framework-country-{id}" className="text-sm text-slate-600">` — displays `framework.country`
     - **Type cell** — `<td data-testid="framework-type-cell-{id}">` — renders a `<Badge data-testid="framework-type-badge-{id}">` with colour-coded className based on `regulation_type`: `national` → `bg-blue-100 text-blue-800`, `eu` → `bg-indigo-100 text-indigo-800`, `programme` → `bg-green-100 text-green-800`. Badge label: `t(\`compliance.type.${framework.regulation_type}\`)`
     - **Status cell** — `<td data-testid="framework-status-cell-{id}">` — renders a toggle button `<button data-testid="framework-active-toggle-{id}" aria-pressed={framework.is_active} className={framework.is_active ? "bg-green-500 ..." : "bg-slate-300 ..."}>` styled as a pill toggle. Clicking the toggle calls `updateFrameworkMutation.mutate({ id: framework.id, data: { is_active: !framework.is_active } })` via `useUpdateComplianceFramework()`. While the mutation is in-flight for this specific framework (tracked via `useState<string | null>(null)` for `togglingId`), the toggle shows `<Loader2 className="animate-spin w-3 h-3" />` and is disabled. On success, `queryClient.invalidateQueries({ queryKey: ["compliance-frameworks"] })` refreshes the list.
     - **Created date cell** — `<td data-testid="framework-created-{id}" className="text-sm text-slate-500">` formatted with `new Date(framework.created_at).toLocaleDateString(locale)`
     - **Actions cell** — `<td data-testid="framework-actions-{id}">` with two buttons:
       - **Edit** — `<Button variant="ghost" size="sm" data-testid="framework-edit-btn-{id}">t("compliance.list.editBtn")</Button>` — navigates to `/${locale}/compliance/{id}/edit` via `router.push()`
       - **Delete** — `<Button variant="ghost" size="sm" data-testid="framework-delete-btn-{id}" className="text-red-600 hover:text-red-700">t("compliance.list.deleteBtn")</Button>` — opens the delete confirmation dialog for that framework (see AC8)

7. **AC7** — When the query returns zero frameworks after client-side filtering, `<EmptyState data-testid="compliance-empty-state">` from `@eusolicit/ui` renders with `icon={ShieldCheck}` (from `lucide-react`), `title={t("compliance.list.emptyTitle")}`, `description={t("compliance.list.emptyDescription")}`, and `action={{ label: t("compliance.createBtn"), onClick: () => router.push(\`/${locale}/compliance/new\`) }}`.

8. **AC8** — Delete confirmation renders as a `<Dialog data-testid="framework-delete-dialog">` with:
   - Title: `t("compliance.list.deleteDialogTitle")` and description: `t("compliance.list.deleteDialogDescription", { name: selectedFramework.name })`
   - **Cancel button** (`data-testid="framework-delete-cancel-btn"`) — closes dialog
   - **Confirm delete button** (`data-testid="framework-delete-confirm-btn"`, `className="bg-red-600 hover:bg-red-700 text-white"`) — calls `deleteFrameworkMutation.mutate(selectedFrameworkId)` via `useDeleteComplianceFramework()`; shows `<Loader2 className="animate-spin" />` while in-flight (`data-testid="framework-delete-loading"`); on success closes dialog and triggers `queryClient.invalidateQueries({ queryKey: ["compliance-frameworks"] })`; on error (e.g. HTTP 409 — framework assigned to active opportunity), shows inline error message `data-testid="framework-delete-error"` with text `t("compliance.list.deleteBlockedError")` instead of generic server error
   - Dialog state managed via `useState<string | null>(null)` for `selectedFrameworkId` and `useState<boolean>(false)` for `deleteDialogOpen`

9. **AC9** — Pagination renders below the table (`data-testid="compliance-pagination"`) when `total > page_size`. It shows:
   - `<span data-testid="compliance-pagination-info">` with text: `t("compliance.list.paginationInfo", { from, to, total })` (e.g. "1–20 of 47")
   - `<Button data-testid="compliance-prev-btn" disabled={page <= 1} onClick={() => setPage(p => p - 1)}>t("common.back")</Button>`
   - `<Button data-testid="compliance-next-btn" disabled={page * pageSize >= total} onClick={() => setPage(p => p + 1)}>t("common.next")</Button>`
   - Page state: `useState<number>(1)` for `page`, constant `pageSize = 20`. `page` resets to `1` whenever any filter or `searchQuery` changes.

---

### Framework Editor Page (Create + Edit)

10. **AC10** — `FrameworkEditor` component (`apps/admin/app/[locale]/(protected)/compliance/components/FrameworkEditor.tsx`) is a `"use client"` component accepting props `mode: "create" | "edit"` and `frameworkId?: string`. In `mode="edit"`, it calls `useComplianceFramework(frameworkId!)` to load existing data; while loading it renders `<SkeletonCard data-testid="framework-editor-skeleton" />`. The root element has `data-testid="framework-editor-page"`.

    Page title renders as:
    - Create mode: `<h1 data-testid="framework-editor-title">t("compliance.editor.createTitle")</h1>`
    - Edit mode: `<h1 data-testid="framework-editor-title">t("compliance.editor.editTitle")</h1>`

11. **AC11 — Main form fields.** The form uses `useZodForm` from `@eusolicit/ui` with `frameworkEditorSchema` (defined in the component file via `z.object({...})`). All fields are pre-populated from `existingFramework` when in edit mode. Fields:
    - **Name** — `<Input data-testid="framework-name-input" name="name" placeholder={t("compliance.editor.namePlaceholder")} />` with label `t("compliance.editor.nameLabel")`. Required, `minLength(1)`, `maxLength(255)`. Shows inline Zod error `data-testid="framework-name-error"` if empty on submit.
    - **Description** — `<Textarea data-testid="framework-description-input" name="description" rows={3} />` with label `t("compliance.editor.descriptionLabel")`. Optional.
    - **Country** — `<Input data-testid="framework-country-input" name="country" placeholder={t("compliance.editor.countryPlaceholder")} />` with label `t("compliance.editor.countryLabel")`. Required; accepts ISO country codes or `"EU"` / `"INT"` (max 10 chars). Shows inline error `data-testid="framework-country-error"` if empty on submit. Dev Notes provide a helper comment listing common values: `"BG"`, `"EU"`, `"INT"`, `"DE"`, `"FR"`, `"PL"`.
    - **Regulation type** — A radio group `data-testid="framework-regulation-type-group"` with three options managed via `useZodForm`'s `watch/setValue`:
      - `<input type="radio" data-testid="framework-type-national" value="national" name="regulation_type" />` with label `t("compliance.type.national")`
      - `<input type="radio" data-testid="framework-type-eu" value="eu" name="regulation_type" />` with label `t("compliance.type.eu")`
      - `<input type="radio" data-testid="framework-type-programme" value="programme" name="regulation_type" />` with label `t("compliance.type.programme")`
      Required; shows error `data-testid="framework-type-error"` if unselected on submit.

12. **AC12 — Rules editor.** Below the main form fields, a rules editor section (`data-testid="framework-rules-editor"`) manages validation rules independently of `useZodForm`. Rules state: `useState<ValidationRuleLocal[]>([])` for `rules`, where `ValidationRuleLocal` adds a client-only `localId: string` (generated via `crypto.randomUUID()`) to each rule for stable React key and targeting. On edit mode load, rules are initialised from `existingFramework.rules` (each rule gets a `localId`).

    - **Section heading** — `<h2 data-testid="framework-rules-heading">t("compliance.editor.rulesTitle")</h2>`
    - **"Add Rule" button** — `<Button type="button" variant="outline" data-testid="framework-add-rule-btn">t("compliance.editor.addRuleBtn")</Button>` — appends a new default rule `{ localId: crypto.randomUUID(), criterion: "", check_type: "contains", threshold: null, description: "" }` to `rules` state.
    - **Rule rows** — Each rule in `rules` renders as a `<div data-testid="framework-rule-{index}" key={rule.localId}>` (index = array position) containing:
      - **Criterion input** — `<Input data-testid="rule-criterion-{index}" placeholder={t("compliance.editor.rulesCriterion")} value={rule.criterion} onChange={...} />`. Required when saving (validated in handleSubmit, not via Zod — see AC13). Inline error `data-testid="rule-criterion-error-{index}"` shown if empty on submit.
      - **Check type select** — `<Select data-testid="rule-check-type-{index}" value={rule.check_type} onValueChange={...}>` with four `<SelectItem>` options:
        - `value="contains"` → label `t("compliance.editor.checkTypeContains")`
        - `value="regex"` → label `t("compliance.editor.checkTypeRegex")`
        - `value="threshold"` → label `t("compliance.editor.checkTypeThreshold")`
        - `value="boolean"` → label `t("compliance.editor.checkTypeBoolean")`
        On change, also resets `threshold` to `null` if new type is not `"threshold"`.
      - **Threshold input** (conditional) — Renders **only** when `rule.check_type === "threshold"`: `<Input type="number" data-testid="rule-threshold-{index}" placeholder={t("compliance.editor.rulesThreshold")} value={rule.threshold ?? ""} onChange={...} />`. Required when visible (validated in handleSubmit). Inline error `data-testid="rule-threshold-error-{index}"` if empty/non-numeric on submit.
      - **Description input** — `<Input data-testid="rule-description-{index}" placeholder={t("compliance.editor.rulesDescription")} value={rule.description ?? ""} onChange={...} />`. Optional.
      - **Remove button** — `<Button type="button" variant="ghost" size="sm" data-testid="rule-remove-btn-{index}" className="text-red-500 hover:text-red-700">` with `<Trash2 />` icon from `lucide-react`. Removes the rule at `index` from the `rules` array (`setRules(prev => prev.filter((_, i) => i !== index))`).
      - **Move-up button** — `<Button type="button" variant="ghost" size="sm" data-testid="rule-move-up-{index}" disabled={index === 0}>` with `<ArrowUp />` icon. Disabled when `index === 0`. Swaps rules at positions `index-1` and `index`.
      - **Move-down button** — `<Button type="button" variant="ghost" size="sm" data-testid="rule-move-down-{index}" disabled={index === rules.length - 1}>` with `<ArrowDown />` icon. Disabled when `index === rules.length - 1`. Swaps rules at positions `index` and `index+1`.

13. **AC13 — Preview panel.** Alongside the rules editor (two-column layout on large screens: `grid grid-cols-1 lg:grid-cols-2 gap-6`), a preview panel (`data-testid="framework-preview-panel"`) shows a live validation simulation. The panel has:
    - **Heading** — `<h2 data-testid="framework-preview-heading">t("compliance.editor.previewTitle")</h2>`
    - **Sample proposal text** (static, defined in Dev Notes) rendered as `<p data-testid="framework-preview-sample" className="text-xs text-slate-500 italic mb-3">` — truncated to first 120 characters with `…`
    - **Per-rule validation results** — for each rule in `rules` state, a `<div data-testid="framework-preview-result-{index}">` showing:
      - Rule criterion label in `text-sm font-medium`
      - Result indicator computed client-side (no API call):
        - `check_type === "contains"` and `criterion` non-empty: check if `SAMPLE_PROPOSAL_TEXT.toLowerCase().includes(criterion.toLowerCase())` → `true` renders `<span data-testid="preview-pass-{index}" className="text-green-600">t("compliance.editor.previewPass")</span>` with `<CheckCircle2 />` icon; `false` → `<span data-testid="preview-fail-{index}" className="text-red-600">t("compliance.editor.previewFail")</span>` with `<XCircle />` icon
        - `check_type === "regex"` and `criterion` non-empty: try `new RegExp(criterion).test(SAMPLE_PROPOSAL_TEXT)` — if test passes → pass indicator; if fails → fail indicator; if `criterion` is invalid regex (caught via `try/catch`) → `<span data-testid="preview-invalid-regex-{index}" className="text-amber-600">` with `<AlertCircle />` icon and `t("compliance.editor.previewInvalidRegex")`
        - `check_type === "boolean"` or `check_type === "threshold"`: → `<span data-testid="preview-na-{index}" className="text-slate-400">t("compliance.editor.previewNA")</span>` with `<Minus />` icon
        - If `criterion` is empty string: → `<span className="text-slate-300">—</span>`
    - Preview updates reactively on every `rules` state change (no debounce needed — purely client-side computation).

14. **AC14 — Save and Cancel.** Below the rules editor + preview layout:
    - **Cancel button** — `<Button type="button" variant="outline" data-testid="framework-cancel-btn" onClick={() => router.push(\`/${locale}/compliance\`)}>t("compliance.editor.cancelBtn")</Button>`
    - **Save button** — `<Button type="submit" data-testid="framework-save-btn">t("compliance.editor.saveBtn")</Button>`

    On form submit (`handleSubmit` from `useZodForm`):
    1. Validate main form fields via Zod schema.
    2. Validate rules: any rule with empty `criterion` → set `rulesHaveErrors = true` and show per-rule criterion errors; any rule with `check_type === "threshold"` and empty/non-numeric `threshold` → set `rulesHaveErrors = true`. If `rulesHaveErrors`, abort submission and show errors inline.
    3. Build final payload: strip `localId` from each rule; if `rule_id` is present in the original rule (from edit mode), preserve it; for new rules, omit `rule_id` (backend auto-generates).
    4. In **create mode**: call `createFrameworkMutation.mutate(payload)`. On success: show toast `t("compliance.editor.createSuccess")` via `useToast()`, then `router.push(\`/${locale}/compliance\`)`.
    5. In **edit mode**: call `updateFrameworkMutation.mutate({ id: frameworkId!, data: payload })`. On success: show toast `t("compliance.editor.updateSuccess")` via `useToast()`, then `router.push(\`/${locale}/compliance\`)`.
    6. While mutation is in-flight (`mutation.isPending`): Save button shows `<Loader2 className="animate-spin" />` and is `disabled`; all form fields and rule inputs are `disabled`.
    7. On mutation error: inline error block `data-testid="framework-save-error"` renders below the save button with `t("errors.serverError")`. Form re-enables.

---

### API Client & React Query Hooks

15. **AC15** — File `apps/admin/lib/api/compliance-frameworks.ts` is created and exports:
    - TypeScript interfaces: `ValidationRule`, `ValidationRuleLocal`, `ComplianceFramework`, `ComplianceFrameworkListResponse`, `ListFrameworksParams`, `CreateFrameworkRequest`, `UpdateFrameworkRequest` (see Dev Notes for full definitions).
    - `listComplianceFrameworks(params?: ListFrameworksParams): Promise<ComplianceFrameworkListResponse>` — calls `apiClient.get<ComplianceFrameworkListResponse>("/api/v1/admin/compliance-frameworks", { params })` then returns `response.data`. During stub phase: returns `STUB_FRAMEWORKS_LIST` after 600 ms delay (see Dev Notes for fixture).
    - `getComplianceFramework(id: string): Promise<ComplianceFramework>` — calls `apiClient.get<ComplianceFramework>(\`/api/v1/admin/compliance-frameworks/${id}\`)` then returns `response.data`. During stub phase: returns matching stub entry or rejects after 400 ms.
    - `createComplianceFramework(data: CreateFrameworkRequest): Promise<ComplianceFramework>` — calls `apiClient.post<ComplianceFramework>("/api/v1/admin/compliance-frameworks", data)` then returns `response.data`. During stub phase: returns a synthetic framework after 800 ms.
    - `updateComplianceFramework(id: string, data: UpdateFrameworkRequest): Promise<ComplianceFramework>` — calls `apiClient.patch<ComplianceFramework>(\`/api/v1/admin/compliance-frameworks/${id}\`, data)` then returns `response.data`. During stub phase: merges data into the matching stub after 800 ms.
    - `deleteComplianceFramework(id: string): Promise<void>` — calls `apiClient.delete(\`/api/v1/admin/compliance-frameworks/${id}\`)`. During stub phase: resolves after 400 ms.
    - Import: `import { apiClient } from "@eusolicit/ui"` (commented out in stub phase — uncomment when backend is deployed).

16. **AC16** — File `apps/admin/lib/queries/use-compliance-frameworks.ts` is created with `"use client"` directive and exports:
    - `useComplianceFrameworks(params?: ListFrameworksParams)` — returns `useQuery({ queryKey: ["compliance-frameworks", params], queryFn: () => listComplianceFrameworks(params), staleTime: 30_000 })` from `@tanstack/react-query`. Server-side filters (`country`, `regulation_type`, `is_active`) are passed as `params`; `page` and `page_size` are included. Client-side `searchQuery` filtering is applied in the component (not as an API param) to avoid re-fetching on every keystroke.
    - `useComplianceFramework(id: string)` — returns `useQuery({ queryKey: ["compliance-frameworks", id], queryFn: () => getComplianceFramework(id), enabled: !!id, staleTime: 30_000 })`.
    - `useCreateComplianceFramework()` — returns `useMutation({ mutationFn: createComplianceFramework, onSuccess: () => queryClient.invalidateQueries({ queryKey: ["compliance-frameworks"] }) })`.
    - `useUpdateComplianceFramework()` — returns `useMutation({ mutationFn: ({ id, data }: { id: string; data: UpdateFrameworkRequest }) => updateComplianceFramework(id, data), onSuccess: () => queryClient.invalidateQueries({ queryKey: ["compliance-frameworks"] }) })`.
    - `useDeleteComplianceFramework()` — returns `useMutation({ mutationFn: deleteComplianceFramework, onSuccess: () => queryClient.invalidateQueries({ queryKey: ["compliance-frameworks"] }) })`.
    - `queryClient` is obtained via `useQueryClient()` from `@tanstack/react-query`.

---

### i18n

17. **AC17** — All user-visible strings in the new components use `useTranslations("compliance")` (plus `useTranslations("common")` and `useTranslations("errors")` for pre-existing keys). New translation keys are added to `apps/admin/messages/en.json` and `apps/admin/messages/bg.json`:

    **Under `"nav"` namespace (add to existing object):**
    ```
    nav.compliance
    ```

    **New `"compliance"` namespace:**
    ```
    compliance.pageTitle
    compliance.pageDescription
    compliance.createBtn
    compliance.list.colName
    compliance.list.colCountry
    compliance.list.colType
    compliance.list.colStatus
    compliance.list.colCreated
    compliance.list.colActions
    compliance.list.editBtn
    compliance.list.deleteBtn
    compliance.list.emptyTitle
    compliance.list.emptyDescription
    compliance.list.deleteDialogTitle
    compliance.list.deleteDialogDescription
    compliance.list.deleteConfirmBtn
    compliance.list.deleteCancelBtn
    compliance.list.deleteBlockedError
    compliance.list.searchPlaceholder
    compliance.list.filterCountry
    compliance.list.filterType
    compliance.list.filterStatus
    compliance.list.statusAll
    compliance.list.statusActive
    compliance.list.statusInactive
    compliance.list.paginationInfo
    compliance.type.national
    compliance.type.eu
    compliance.type.programme
    compliance.editor.createTitle
    compliance.editor.editTitle
    compliance.editor.nameLabel
    compliance.editor.namePlaceholder
    compliance.editor.descriptionLabel
    compliance.editor.countryLabel
    compliance.editor.countryPlaceholder
    compliance.editor.typeLabel
    compliance.editor.rulesTitle
    compliance.editor.addRuleBtn
    compliance.editor.rulesCriterion
    compliance.editor.rulesCheckType
    compliance.editor.rulesThreshold
    compliance.editor.rulesDescription
    compliance.editor.checkTypeContains
    compliance.editor.checkTypeRegex
    compliance.editor.checkTypeThreshold
    compliance.editor.checkTypeBoolean
    compliance.editor.previewTitle
    compliance.editor.previewPass
    compliance.editor.previewFail
    compliance.editor.previewNA
    compliance.editor.previewInvalidRegex
    compliance.editor.saveBtn
    compliance.editor.cancelBtn
    compliance.editor.createSuccess
    compliance.editor.updateSuccess
    ```

    `pnpm check:i18n --filter admin` must exit 0 after additions.

---

### Build & Type-Check

18. **AC18** — `pnpm build` exits 0 for both `client` and `admin` apps. `pnpm type-check` exits 0 across all packages. No TypeScript errors in any new or modified file.

---

## Tasks / Subtasks

- [x] **Task 1: Create compliance framework API client module** (AC: 15)
  - [x] 1.1 Create `apps/admin/lib/api/compliance-frameworks.ts`:
    - Define all TypeScript interfaces: `ValidationRule`, `ValidationRuleLocal`, `ComplianceFramework`, `ComplianceFrameworkListResponse`, `ListFrameworksParams`, `CreateFrameworkRequest`, `UpdateFrameworkRequest` (see Dev Notes for full definitions)
    - Implement `STUB_FRAMEWORKS_LIST` and `STUB_SINGLE_FRAMEWORK` fixture constants (see Dev Notes for fixture data)
    - Implement `listComplianceFrameworks()` stub (600 ms delay + `STUB_FRAMEWORKS_LIST`)
    - Implement `getComplianceFramework()` stub (400 ms delay + find from stub list)
    - Implement `createComplianceFramework()` stub (800 ms delay + synthetic framework)
    - Implement `updateComplianceFramework()` stub (800 ms delay + merged object)
    - Implement `deleteComplianceFramework()` stub (400 ms delay + void)
    - Wire real API calls with `apiClient` behind comments (for when S11.08 backend is integrated)

- [x] **Task 2: Create React Query hooks** (AC: 16)
  - [x] 2.1 Create `apps/admin/lib/queries/use-compliance-frameworks.ts` with `"use client"` directive
  - [x] 2.2 Implement `useComplianceFrameworks(params?)` — useQuery hook
  - [x] 2.3 Implement `useComplianceFramework(id)` — useQuery hook with `enabled: !!id`
  - [x] 2.4 Implement `useCreateComplianceFramework()` — useMutation with invalidation
  - [x] 2.5 Implement `useUpdateComplianceFramework()` — useMutation with invalidation
  - [x] 2.6 Implement `useDeleteComplianceFramework()` — useMutation with invalidation

- [x] **Task 3: Add i18n translation keys** (AC: 17)
  - [x] 3.1 Add `"nav": { "compliance": "Compliance" }` entry (merge into existing `nav` object) and new `"compliance": {...}` namespace to `apps/admin/messages/en.json` with all required keys (see Dev Notes for English values)
  - [x] 3.2 Add corresponding Bulgarian translations to `apps/admin/messages/bg.json` (see Dev Notes for Bulgarian values)
  - [x] 3.3 Run `pnpm check:i18n --filter admin` — must exit 0

- [x] **Task 4: Update admin navigation** (AC: 2)
  - [x] 4.1 Open `apps/admin/app/[locale]/(protected)/layout.tsx`
  - [x] 4.2 Add `ShieldCheck` to lucide-react import
  - [x] 4.3 Add compliance nav item to `adminNavItems` array: `{ icon: ShieldCheck, label: t("nav.compliance"), href: \`/${locale}/compliance\` }`

- [x] **Task 5: Build Framework List page** (AC: 3, 4, 5, 6, 7, 8, 9)
  - [x] 5.1 Create `apps/admin/app/[locale]/(protected)/compliance/page.tsx` — minimal server component shell that imports and renders `<FrameworkList />`
  - [x] 5.2 Create `apps/admin/app/[locale]/(protected)/compliance/components/` directory
  - [x] 5.3 Create `apps/admin/app/[locale]/(protected)/compliance/components/FrameworkList.tsx`:
    - `"use client"` directive; imports from `@eusolicit/ui`, `lucide-react`, `next/navigation`, `next-intl`
    - Page header with title, description, Create button
    - Filter section: search input (debounced 300ms), country dropdown, regulation type checkboxes, status dropdown
    - Table with 6 columns; skeleton while loading; rows with name, country, type badge, active toggle (with Loader2 while toggling), created date, actions
    - Active toggle: click calls `useUpdateComplianceFramework()` with `{ is_active: !current }`, tracks `togglingId` state
    - Edit button: `router.push(\`/${locale}/compliance/${id}/edit\`)`
    - Delete button: opens `deleteDialogOpen` dialog with selected framework
    - Delete dialog: `<Dialog>` with confirm (calls `useDeleteComplianceFramework()`) and cancel buttons; shows error on 409 conflict
    - Empty state with `<EmptyState>` component
    - Pagination: prev/next buttons, page info span; page resets to 1 on filter change

- [x] **Task 6: Build Framework Editor page** (AC: 10, 11, 12, 13, 14)
  - [x] 6.1 Create `apps/admin/app/[locale]/(protected)/compliance/new/page.tsx` — shell rendering `<FrameworkEditor mode="create" />`
  - [x] 6.2 Create `apps/admin/app/[locale]/(protected)/compliance/[id]/edit/page.tsx` — shell rendering `<FrameworkEditor mode="edit" frameworkId={params.id} />`
  - [x] 6.3 Create `apps/admin/app/[locale]/(protected)/compliance/components/FrameworkEditor.tsx`:
    - `"use client"` directive; imports from `@eusolicit/ui`, `lucide-react`, `next/navigation`, `next-intl`
    - In edit mode: load existing framework via `useComplianceFramework(frameworkId)`; show skeleton while loading
    - Main form using `useZodForm` with `frameworkEditorSchema` (z.object): name (required), description (optional), country (required, max 10), regulation_type (enum, required)
    - Regulation type as three radio inputs, managed via `register("regulation_type")` and `watch("regulation_type")`
    - Rules editor as separate `useState<ValidationRuleLocal[]>` state; add/remove/reorder (swap adjacent items) with buttons; per-rule: criterion input, check_type select, conditional threshold input, description input
    - `SAMPLE_PROPOSAL_TEXT` constant (see Dev Notes); preview panel computed from `rules` state; regex try/catch for invalid patterns
    - `handleSubmit`: validate Zod schema, validate rules (criterion required, threshold required for threshold type), build payload (strip localId, preserve rule_id from existing rules), call create/update mutation
    - Success: toast + navigate to list; error: inline error block; loading: disabled fields + spinner on save button
    - Cancel button navigates to `/${locale}/compliance`
    - Two-column grid layout on large screens for rules editor + preview panel
  - [x] 6.4 Define `SAMPLE_PROPOSAL_TEXT` constant in `FrameworkEditor.tsx` (see Dev Notes for value)
  - [x] 6.5 Define `frameworkEditorSchema` as a `z.object` in `FrameworkEditor.tsx` using `z` imported from `@eusolicit/ui`

- [x] **Task 7: Build and type-check verification** (AC: 18)
  - [x] 7.1 Run `pnpm build` from `/home/debian/Projects/eusolicit/eusolicit-app/frontend/` — both `client` and `admin` apps must exit 0
  - [x] 7.2 Run `pnpm type-check` from `/home/debian/Projects/eusolicit/eusolicit-app/frontend/` — zero TypeScript errors

### Review Findings

> **Code review complete (2026-04-10).** 0 `decision-needed`, 10 `patch`, 8 `defer`, 11 dismissed as noise.
> Review layers: Blind Hunter ✅, Edge Case Hunter ✅, Acceptance Auditor ✅.

#### Patch Findings

- [x] [Review][Patch] **Delete error handler shows 409-specific message for ALL errors** — `FrameworkList.tsx:155-162` — Both `if` and `else` branches call `t("list.deleteBlockedError")`. The `else` branch must use `tErrors("serverError")` for generic errors. (AC8 violation, HIGH)
- [x] [Review][Patch] **Double cache invalidation on toggle and delete mutations** — `FrameworkList.tsx:128,151` — Component callbacks call `queryClient.invalidateQueries()` but `useUpdateComplianceFramework` and `useDeleteComplianceFramework` hooks already invalidate in their `onSuccess`. Remove inline `queryClient.invalidateQueries()` calls from the component. (MEDIUM)
- [x] [Review][Patch] **Pagination displays server totals instead of filtered totals** — `FrameworkList.tsx:117,198-199` — `filterTypes` (regulation type checkboxes) filters client-side, but pagination info/controls read `total` and `pageSize` from the server response. Pagination shows "1–20 of 20" when only 5 match the type filter. Use `filteredFrameworks.length` for display counts and next-button logic. (AC5/AC9, MEDIUM)
- [x] [Review][Patch] **Country filter uses hardcoded STUB_COUNTRIES instead of dataset-derived values** — `FrameworkList.tsx:166-168,557` — AC5 says "one SelectItem per unique country in the loaded dataset". Replace `STUB_COUNTRIES` with `Array.from(new Set((data?.frameworks ?? []).map(f => f.country)))`. (AC5 violation, MEDIUM)
- [x] [Review][Patch] **removeRule leaves stale error indices in ruleErrors** — `FrameworkEditor.tsx:158-165` — `ruleErrors` keyed by array position; deleting rule at index `i` doesn't re-key `i+1, i+2, …` so validation errors display on wrong rows after deletion. Rebuild ruleErrors with shifted indices or key by `localId`. (MEDIUM)
- [x] [Review][Patch] **frameworkId non-null assertion unsafe in edit mode** — `FrameworkEditor.tsx:237` — `frameworkId!` assertion silences TS but if `id` param is missing, `updateComplianceFramework(undefined, ...)` is called. Add early guard: `if (mode === "edit" && !frameworkId) return <ErrorState />;` (MEDIUM)
- [x] [Review][Patch] **Delete error state persists across dialog reopen** — `FrameworkList.tsx:138-142` — `openDeleteDialog()` clears `deleteError`, but if the user closes the dialog via `onOpenChange` (backdrop click, Escape key) after an error, then reopens it for a different framework, the error is correctly cleared. Verified: `openDeleteDialog` does call `setDeleteError(null)`. However, ensure `onOpenChange` handler also clears deleteError for the escape/backdrop path: add `onOpenChange={(open) => { setDeleteDialogOpen(open); if (!open) setDeleteError(null); }}`. (LOW)
- [x] [Review][Patch] **i18n key `rulesCheckType` defined but never rendered — check-type Select lacks an accessible label** — `FrameworkEditor.tsx:492-517` — The `<Select>` for check_type has no visible `<label>` element. The key `compliance.editor.rulesCheckType` exists in both JSON files but is never used. Add a label above the Select. (AC17 gap, LOW)
- [x] [Review][Patch] **useComplianceFramework receives empty string instead of undefined** — `FrameworkEditor.tsx:106` — `frameworkId ?? ""` puts `""` into cache key `["compliance-frameworks", ""]`. Cleaner: update hook signature to accept `string | undefined`, pass `frameworkId` directly. (LOW)
- [x] [Review][Patch] **existingFramework.rules may be undefined on malformed response** — `FrameworkEditor.tsx:128` — `existingFramework.rules.map(...)` will throw if `rules` is `undefined`. Add guard: `(existingFramework.rules ?? []).map(...)`. (LOW)

#### Deferred Findings

- [x] [Review][Defer] **crypto.randomUUID() without fallback for older browsers** — `FrameworkEditor.tsx:75,128` — All modern browsers support it; admin targets modern browsers only. — deferred, pre-existing environment assumption
- [x] [Review][Defer] **ReDoS vulnerability in regex preview** — `FrameworkEditor.tsx:291` — `new RegExp(criterion).test()` with user input can hang on catastrophic backtracking. Admin-only UI limits attack surface. — deferred, future hardening
- [x] [Review][Defer] **Next.js 15 params async breaking change** — `[id]/edit/page.tsx:5-10` — `params.id` accessed synchronously; Next.js 15 makes `params` a Promise. — deferred, pre-existing (current app is Next.js 14)
- [x] [Review][Defer] **Query key structural collision between list and detail queries** — `use-compliance-frameworks.ts:31,43` — Both use `["compliance-frameworks", ...]`. Prefix invalidation works correctly but keys share namespace. — deferred, pre-existing pattern
- [x] [Review][Defer] **Mutation callbacks may fire after component unmount on navigation** — `FrameworkEditor.tsx:226-248` — React Query handles gracefully in modern versions. — deferred, pre-existing
- [x] [Review][Defer] **Concurrent toggle + delete mutations race condition** — `FrameworkList.tsx:121-163` — Low probability admin scenario; mutations serialized by React Query. — deferred, pre-existing
- [x] [Review][Defer] **Null created_at could crash toLocaleDateString** — `FrameworkList.tsx:432` — TypeScript interface guarantees string. Backend contract enforces non-null. — deferred, pre-existing
- [x] [Review][Defer] **Form reset clobbers unsaved edits if stale query refetches** — `FrameworkEditor.tsx:118-135` — Standard react-hook-form pattern; admin concurrent editing is rare. — deferred, pre-existing

### Senior Developer Review (2026-04-10)

> **Re-review complete.** All 10 previous patch findings verified fixed. 1 new `patch` finding, 2 new `defer`, 7 dismissed as noise.
> Review layers: Blind Hunter ✅, Edge Case Hunter ✅, Acceptance Auditor ✅.
> Type-check: ✅ (`pnpm type-check` exits 0, all 4 packages pass)

#### Previous Patch Findings — Verification

All 10 patch findings from the prior review have been correctly addressed:

1. ✅ Delete error handler — `else` branch now uses `tErrors("serverError")` (line 157)
2. ✅ Double cache invalidation — removed inline `queryClient.invalidateQueries()` from component callbacks
3. ✅ Pagination totals — uses `filteredTotal = filteredFrameworks.length` (lines 118, 466, 472, 488)
4. ✅ Country filter — derived from dataset via `Array.from(new Set(...))` (line 164)
5. ✅ ruleErrors keyed by `localId` — `Record<string, { criterion?; threshold? }>` (lines 141-143)
6. ✅ frameworkId guard — early return with error state (lines 257-265)
7. ✅ Dialog `onOpenChange` clears error — `if (!open) setDeleteError(null)` (lines 500-503)
8. ✅ Check-type Select label — `<label>` element with `t("editor.rulesCheckType")` (lines 509-510)
9. ✅ Hook accepts `string | undefined` — `useComplianceFramework(id: string | undefined)` (hook line 41)
10. ✅ Rules null guard — `(existingFramework.rules ?? []).map(...)` (line 129)

#### New Patch Findings

- [x] [Review][Patch] **Hardcoded English strings bypass i18n — AC17 violation** — `FrameworkEditor.tsx:476,636,647` — Three user-visible strings are hardcoded in English instead of using `useTranslations()`. Bulgarian users would see English text mixed with Bulgarian UI. (1) Line 476: `No rules yet. Click "Add Rule" to create one.` (2) Line 636: `Add rules to see live validation results.` (3) Line 647: `` `Rule ${index + 1}` `` fallback label. Fix: add three new i18n keys (e.g. `editor.rulesEmpty`, `editor.previewEmpty`, `editor.ruleDefaultLabel`) to both `en.json` and `bg.json`, then use `t(...)` for all three strings. (AC17, HIGH — all three layers flagged this independently)

#### New Deferred Findings

- [x] [Review][Defer] **Toggle active error handler provides no user feedback** — `FrameworkList.tsx:129-131` — `onError` only clears `togglingId`; no toast or inline error shown. Toggle snaps back to original value silently. AC6 does not specify toggle error handling. — deferred, UX enhancement
- [x] [Review][Defer] **Client-side type/search filters over server-paginated data produce incomplete results at scale** — `FrameworkList.tsx:101-118,164` — When dataset exceeds one page, client-side type/search filters only operate on the current server page, missing matching items on other pages. Country dropdown also only shows current-page countries. By AC design (AC5 specifies client-side filtering, API supports single regulation_type only). — deferred, design limitation for future sprint

### Final Review (2026-04-10)

> **REVIEW: Approve** — 0 `decision-needed`, 0 `patch`, 7 `defer` (5 re-confirmed, 2 new), 3 dismissed as noise.
> Review layers: Blind Hunter ✅, Edge Case Hunter ✅, Acceptance Auditor ✅.
> All 18 acceptance criteria verified. All 11 prior patch findings confirmed fixed.

#### Prior Findings — All Verified Fixed

All 10 first-review patch findings and the 1 second-review patch finding (hardcoded i18n strings) are confirmed correctly resolved. Code matches spec for every AC.

#### New Deferred Findings

- [x] [Review][Defer] **Missing `isError` handling on framework list query** — `FrameworkList.tsx:82` — Query failure is indistinguishable from empty dataset (empty state renders). Not required by AC3; stub never errors. — deferred, API integration hardening for S11.08 wiring
- [x] [Review][Defer] **Missing `isError` handling on framework editor query (edit mode)** — `FrameworkEditor.tsx:105-106` — Failed fetch in edit mode renders blank form instead of error. Not required by AC10; stub never errors. — deferred, API integration hardening for S11.08 wiring

#### Dismissed Findings (3)

1. `updateComplianceFramework` stub fabricates framework on unknown ID — stub-only code, real API returns 404
2. Stale closure in `moveDown` guard — button disabled state uses same render value; synchronous React cycle prevents mismatch
3. `frameworkId!` in `onSubmit` — guard at line 257 prevents form from rendering when frameworkId is missing; assertion is unreachable

---

## Dev Notes

### Working Directory

All frontend code lives under:
`eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

Run all `pnpm` commands from: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`

---

### Critical Carry-Over Learnings (MUST APPLY)

1. **`@/*` alias maps to `apps/admin/`** — Inside `apps/admin`, `@/lib/api/compliance-frameworks` resolves to `apps/admin/lib/api/compliance-frameworks.ts`. Inside `packages/ui`, always use **relative imports** — `@/` does NOT work there.
2. **Route structure** — The file `apps/admin/app/[locale]/(protected)/compliance/page.tsx` maps to URL `/{locale}/compliance` on the admin app (running on port 3001). The `(protected)` route group adds no URL segment. Create `compliance/` as a new directory under `(protected)/`.
3. **`"use client"` on all interactive components** — Any component using `useState`, `useMutation`, `useQuery`, or event handlers requires `"use client"` at the top of the file. Page shells (page.tsx) are server components — no `"use client"` there.
4. **`AuthGuard` from `@eusolicit/ui`** — Already applied in `apps/admin/app/[locale]/(protected)/layout.tsx`. Individual page components do NOT need to add `<AuthGuard>` again.
5. **`apiClient` from `@eusolicit/ui`** — Established in S3.05. Import: `import { apiClient } from "@eusolicit/ui"`. For the stub phase, the real `apiClient` calls are commented out. The admin API endpoints require a `platform_admin` JWT — in the stub phase, this is bypassed since the stub returns synthetic data. Uncomment real calls when S11.08 backend is integrated.
6. **`useZodForm` and `z` from `@eusolicit/ui`** — Established in S3.6. Import: `import { useZodForm, z } from "@eusolicit/ui"`. Define `frameworkEditorSchema` directly in `FrameworkEditor.tsx` using `z.object({ name: z.string().min(1).max(255), description: z.string().nullable().optional(), country: z.string().min(1).max(10), regulation_type: z.enum(["national", "eu", "programme"]) })`.
7. **`useToast` from `@eusolicit/ui`** — Established in S3.11. Import: `import { useToast } from "@eusolicit/ui"`. Call `toast({ title: t("compliance.editor.createSuccess") })` on successful create/update.
8. **`Button`, `Input`, `Textarea`, `Select`, `SelectItem`, `Badge`, `Dialog` from `@eusolicit/ui`** — All verified available. Import: `import { Button, Input, Textarea, Select, SelectItem, Badge, Dialog, DialogContent, DialogHeader, DialogTitle, DialogDescription } from "@eusolicit/ui"`.
9. **`EmptyState`, `SkeletonTable`, `SkeletonCard` from `@eusolicit/ui`** — Established in S3.10 and prior stories. Use `<EmptyState icon={ShieldCheck} title={...} description={...} action={{ label: ..., onClick: ... }} />`.
10. **`useTranslations` from `next-intl`** — Import: `import { useTranslations } from "next-intl"`. Use `useTranslations("compliance")` and `useTranslations("common")`.
11. **`useRouter` and `useParams` from `next/navigation`** — Import: `import { useRouter, useParams } from "next/navigation"`. Use `const { locale } = useParams<{ locale: string }>()` in components to get the current locale for navigation.
12. **`lucide-react` icons** — Import: `import { ShieldCheck, Loader2, Trash2, ArrowUp, ArrowDown, CheckCircle2, XCircle, AlertCircle, Minus } from "lucide-react"`. All available in `lucide-react ^0.400.0`.
13. **No Drag-and-drop library in admin** — Rules reordering uses up/down arrow buttons with array swap logic: `const swap = (arr, i, j) => { const next = [...arr]; [next[i], next[j]] = [next[j], next[i]]; return next; }`.
14. **No Accordion component in `@eusolicit/ui`** — The rules editor uses a flat list with toggle state managed via `useState`. No accordion needed.
15. **`useQueryClient` from `@tanstack/react-query`** — Import: `import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"`.
16. **Locale prefix in navigation** — Any `router.push()` must use `/${locale}/` prefix. Get locale via `useParams<{ locale: string }>()` in the component.
17. **Active toggle loading state** — Track the in-flight toggle via `useState<string | null>(null)` for `togglingId`. Set `togglingId` to the framework ID when the toggle mutation starts; clear on success/error. Use `togglingId === framework.id` to show spinner on the correct row.
18. **Delete 409 error** — When the delete mutation returns a 409 error (framework assigned to active opportunity), display `t("compliance.list.deleteBlockedError")` (not the generic server error) in `data-testid="framework-delete-error"`. Check error response: `error.response?.status === 409`.
19. **Debounced search** — Implement search debounce with `useEffect`:
    ```typescript
    const [searchQuery, setSearchQuery] = useState("");
    const [debouncedSearch, setDebouncedSearch] = useState("");
    useEffect(() => {
      const timer = setTimeout(() => setDebouncedSearch(searchQuery), 300);
      return () => clearTimeout(timer);
    }, [searchQuery]);
    ```
    Use `debouncedSearch` for filtering, `searchQuery` for the input's value.
20. **Radio inputs for regulation_type** — Since `@eusolicit/ui` does not export a `RadioGroup` component, use native `<input type="radio">` elements with RHF's `register("regulation_type")`. Apply `{...register("regulation_type")}` to each radio input. The `watch("regulation_type")` call provides the current selection for visual highlighting (e.g. adding `font-semibold` to the selected option's label).

---

### Architecture: File Structure Added in S11.14

```
eusolicit-app/frontend/
  apps/admin/
    lib/
      api/
        compliance-frameworks.ts                                   ← CREATED
      queries/
        use-compliance-frameworks.ts                               ← CREATED
    app/
      [locale]/
        (protected)/
          layout.tsx                                               ← MODIFIED (ShieldCheck nav item)
          compliance/                                              ← NEW DIRECTORY
            page.tsx                                               ← CREATED
            new/                                                   ← NEW DIRECTORY
              page.tsx                                             ← CREATED
            [id]/                                                  ← NEW DIRECTORY
              edit/                                                ← NEW DIRECTORY
                page.tsx                                           ← CREATED
            components/                                            ← NEW DIRECTORY
              FrameworkList.tsx                                     ← CREATED
              FrameworkEditor.tsx                                   ← CREATED
    messages/
      en.json                                                      ← MODIFIED (nav.compliance + compliance namespace)
      bg.json                                                      ← MODIFIED (nav.compliance + compliance namespace)
  packages/
    (no changes to packages/ui for this story)
```

---

### TypeScript Interface Definitions (for `apps/admin/lib/api/compliance-frameworks.ts`)

```typescript
export interface ValidationRule {
  rule_id?: string;
  criterion: string;
  check_type: "contains" | "regex" | "threshold" | "boolean";
  threshold?: number | null;
  description?: string | null;
}

// Client-only extension — localId is stripped before sending to API
export interface ValidationRuleLocal extends ValidationRule {
  localId: string; // crypto.randomUUID() — used as React key and for targeting specific rules
}

export interface ComplianceFramework {
  id: string;
  name: string;
  description: string | null;
  country: string;
  regulation_type: "national" | "eu" | "programme";
  rules: ValidationRule[];
  is_active: boolean;
  created_by: string;
  created_at: string;
  updated_at: string;
}

export interface ComplianceFrameworkListResponse {
  frameworks: ComplianceFramework[];
  total: number;
  page: number;
  page_size: number;
}

export interface ListFrameworksParams {
  country?: string;
  regulation_type?: string;
  is_active?: boolean;
  page?: number;
  page_size?: number;
}

export interface CreateFrameworkRequest {
  name: string;
  description?: string | null;
  country: string;
  regulation_type: "national" | "eu" | "programme";
  rules?: ValidationRule[];
}

export interface UpdateFrameworkRequest {
  name?: string;
  description?: string | null;
  country?: string;
  regulation_type?: "national" | "eu" | "programme";
  rules?: ValidationRule[];
  is_active?: boolean;
}
```

---

### Stub Fixture Data (for `apps/admin/lib/api/compliance-frameworks.ts`)

```typescript
const STUB_DELAY_MS = 600;

function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

export const STUB_FRAMEWORKS_LIST: ComplianceFrameworkListResponse = {
  frameworks: [
    {
      id: "fw-001",
      name: "ZOP — Bulgarian Public Procurement Law 2024",
      description: "National procurement compliance framework based on the Bulgarian Public Procurement Act (ZOP), transposing EU Directive 2014/24/EU.",
      country: "BG",
      regulation_type: "national",
      rules: [
        {
          rule_id: "rule-zop-001",
          criterion: "обществена поръчка",
          check_type: "contains",
          threshold: null,
          description: "Proposal must reference the Bulgarian public procurement procedure",
        },
        {
          rule_id: "rule-zop-002",
          criterion: "ЕИК",
          check_type: "contains",
          threshold: null,
          description: "Company EIK (registration number) must be included",
        },
      ],
      is_active: true,
      created_by: "admin-001",
      created_at: "2026-01-15T10:00:00Z",
      updated_at: "2026-03-20T09:00:00Z",
    },
    {
      id: "fw-002",
      name: "Horizon Europe — General Conditions",
      description: "EU-level compliance framework for Horizon Europe grant applications, covering financial eligibility, open science requirements, and consortium rules.",
      country: "EU",
      regulation_type: "eu",
      rules: [
        {
          rule_id: "rule-he-001",
          criterion: "Horizon Europe",
          check_type: "contains",
          threshold: null,
          description: "Application must reference Horizon Europe programme",
        },
        {
          rule_id: "rule-he-002",
          criterion: "consortium",
          check_type: "contains",
          threshold: null,
          description: "Multi-beneficiary consortium requirement",
        },
        {
          rule_id: "rule-he-003",
          criterion: "co-financing",
          check_type: "threshold",
          threshold: 70,
          description: "EU co-financing must not exceed 70% for standard beneficiaries",
        },
      ],
      is_active: true,
      created_by: "admin-001",
      created_at: "2026-01-20T11:00:00Z",
      updated_at: "2026-04-01T14:00:00Z",
    },
    {
      id: "fw-003",
      name: "Digital Europe Programme — SME Requirements",
      description: "Programme-level compliance rules for Digital Europe applications, with specific constraints for SME participants.",
      country: "EU",
      regulation_type: "programme",
      rules: [
        {
          rule_id: "rule-dep-001",
          criterion: "SME",
          check_type: "contains",
          threshold: null,
          description: "SME status must be declared",
        },
        {
          rule_id: "rule-dep-002",
          criterion: "\\bdigital\\b",
          check_type: "regex",
          threshold: null,
          description: "Proposal must contain the word 'digital' as a whole word",
        },
      ],
      is_active: true,
      created_by: "admin-002",
      created_at: "2026-02-05T08:30:00Z",
      updated_at: "2026-02-05T08:30:00Z",
    },
    {
      id: "fw-004",
      name: "INTERREG Bulgaria-Romania — Cross-border Rules",
      description: "Programme-specific compliance framework for INTERREG CBC Bulgaria-Romania 2021–2027 applications.",
      country: "BG",
      regulation_type: "programme",
      rules: [],
      is_active: false,
      created_by: "admin-001",
      created_at: "2026-03-01T12:00:00Z",
      updated_at: "2026-04-05T16:00:00Z",
    },
  ],
  total: 4,
  page: 1,
  page_size: 20,
};
```

---

### Sample Proposal Text (for `FrameworkEditor.tsx` preview panel)

```typescript
const SAMPLE_PROPOSAL_TEXT =
  "This proposal for digital transformation services in the public sector aims to " +
  "establish a consortium of SME partners across Bulgaria and Romania. The project " +
  "references Horizon Europe eligibility criteria and applies for EU co-financing. " +
  "The applicant EIK is BG202456789 and the обществена поръчка procedure follows ZOP. " +
  "The digital platform will deliver measurable impact through data-driven governance.";
```

---

### i18n Values — English (`en.json`)

Add to existing `"nav"` object:
```json
"compliance": "Compliance"
```

Add new `"compliance"` top-level namespace:
```json
"compliance": {
  "pageTitle": "Compliance Frameworks",
  "pageDescription": "Manage regulatory compliance frameworks assigned to procurement opportunities.",
  "createBtn": "Create Framework",
  "list": {
    "colName": "Name",
    "colCountry": "Country",
    "colType": "Type",
    "colStatus": "Active",
    "colCreated": "Created",
    "colActions": "Actions",
    "editBtn": "Edit",
    "deleteBtn": "Delete",
    "emptyTitle": "No compliance frameworks",
    "emptyDescription": "Create your first compliance framework to assign regulatory rules to procurement opportunities.",
    "deleteDialogTitle": "Delete framework",
    "deleteDialogDescription": "Are you sure you want to delete \"{name}\"? This action cannot be undone.",
    "deleteConfirmBtn": "Delete",
    "deleteCancelBtn": "Cancel",
    "deleteBlockedError": "This framework is assigned to active opportunities and cannot be deleted. Remove all assignments first.",
    "searchPlaceholder": "Search by framework name...",
    "filterCountry": "Country",
    "filterType": "Regulation type",
    "filterStatus": "Status",
    "statusAll": "All statuses",
    "statusActive": "Active",
    "statusInactive": "Inactive",
    "paginationInfo": "{from}–{to} of {total}"
  },
  "type": {
    "national": "National",
    "eu": "EU",
    "programme": "Programme"
  },
  "editor": {
    "createTitle": "Create Compliance Framework",
    "editTitle": "Edit Compliance Framework",
    "nameLabel": "Framework name",
    "namePlaceholder": "e.g. ZOP — Bulgarian Public Procurement Law 2024",
    "descriptionLabel": "Description",
    "countryLabel": "Country / Scope",
    "countryPlaceholder": "e.g. BG, EU, INT",
    "typeLabel": "Regulation type",
    "rulesTitle": "Validation Rules",
    "addRuleBtn": "Add Rule",
    "rulesCriterion": "Criterion",
    "rulesCheckType": "Check type",
    "rulesThreshold": "Threshold value",
    "rulesDescription": "Rule description (optional)",
    "checkTypeContains": "Contains (text match)",
    "checkTypeRegex": "Regex (pattern match)",
    "checkTypeThreshold": "Threshold (numeric)",
    "checkTypeBoolean": "Boolean (manual check)",
    "previewTitle": "Validation Preview",
    "previewPass": "Pass",
    "previewFail": "Fail",
    "previewNA": "N/A",
    "previewInvalidRegex": "Invalid regex",
    "saveBtn": "Save framework",
    "cancelBtn": "Cancel",
    "createSuccess": "Compliance framework created successfully.",
    "updateSuccess": "Compliance framework updated successfully."
  }
}
```

---

### i18n Values — Bulgarian (`bg.json`)

Add to existing `"nav"` object:
```json
"compliance": "Съответствие"
```

Add new `"compliance"` top-level namespace:
```json
"compliance": {
  "pageTitle": "Рамки за съответствие",
  "pageDescription": "Управлявайте регулаторните рамки за съответствие, присвоени на обществени поръчки.",
  "createBtn": "Създай рамка",
  "list": {
    "colName": "Наименование",
    "colCountry": "Държава",
    "colType": "Тип",
    "colStatus": "Активна",
    "colCreated": "Създадена",
    "colActions": "Действия",
    "editBtn": "Редактирай",
    "deleteBtn": "Изтрий",
    "emptyTitle": "Няма рамки за съответствие",
    "emptyDescription": "Създайте първата рамка за съответствие, за да присвоите регулаторни правила към обществени поръчки.",
    "deleteDialogTitle": "Изтриване на рамка",
    "deleteDialogDescription": "Сигурни ли сте, че искате да изтриете „{name}"? Това действие е необратимо.",
    "deleteConfirmBtn": "Изтрий",
    "deleteCancelBtn": "Отказ",
    "deleteBlockedError": "Тази рамка е присвоена към активни обществени поръчки и не може да бъде изтрита. Премахнете всички присвоявания първо.",
    "searchPlaceholder": "Търсене по наименование...",
    "filterCountry": "Държава",
    "filterType": "Тип регулация",
    "filterStatus": "Статус",
    "statusAll": "Всички статуси",
    "statusActive": "Активни",
    "statusInactive": "Неактивни",
    "paginationInfo": "{from}–{to} от {total}"
  },
  "type": {
    "national": "Национален",
    "eu": "ЕС",
    "programme": "Програмен"
  },
  "editor": {
    "createTitle": "Създаване на рамка за съответствие",
    "editTitle": "Редактиране на рамка за съответствие",
    "nameLabel": "Наименование на рамката",
    "namePlaceholder": "напр. ЗОП — Закон за обществените поръчки 2024",
    "descriptionLabel": "Описание",
    "countryLabel": "Държава / Обхват",
    "countryPlaceholder": "напр. BG, EU, INT",
    "typeLabel": "Тип регулация",
    "rulesTitle": "Правила за валидиране",
    "addRuleBtn": "Добави правило",
    "rulesCriterion": "Критерий",
    "rulesCheckType": "Тип проверка",
    "rulesThreshold": "Прагова стойност",
    "rulesDescription": "Описание на правилото (незадължително)",
    "checkTypeContains": "Съдържа (текстово съвпадение)",
    "checkTypeRegex": "Регулярен израз (шаблон)",
    "checkTypeThreshold": "Праг (числова стойност)",
    "checkTypeBoolean": "Булево (ръчна проверка)",
    "previewTitle": "Предварителен преглед",
    "previewPass": "Преминава",
    "previewFail": "Не преминава",
    "previewNA": "Не е приложимо",
    "previewInvalidRegex": "Невалиден регулярен израз",
    "saveBtn": "Запази рамката",
    "cancelBtn": "Отказ",
    "createSuccess": "Рамката за съответствие е създадена успешно.",
    "updateSuccess": "Рамката за съответствие е актуализирана успешно."
  }
}
```

---

### Test Expectations from Epic Test Design (E11)

The following test scenarios from `eusolicit-docs/test-artifacts/test-design-epic-11.md` inform QA expectations for this story's frontend. Developer must ensure component structure supports these tests.

**E11-P2-018 (P2 — Playwright E2E):** Compliance Framework List — filter by country + regulation_type; search by name; pagination.
- `data-testid="compliance-framework-list-page"` must be present on load.
- `data-testid="compliance-search-input"` must accept text and trigger client-side filtering (300 ms debounce).
- `data-testid="compliance-filter-type-national"`, `...-eu"`, `...-programme"` checkboxes must filter the table.
- `data-testid="compliance-pagination"` must render when `total > page_size`; `data-testid="compliance-next-btn"` must advance page.
- Table row count must match filtered result count.

**E11-P3-003 (P3 — full-stack E2E Journey 3):** Admin creates compliance framework → assigns to opportunity → user's proposal is validated against it (reuses E07 compliance checker).
- This story covers **step 1** (create compliance framework) of the journey. `data-testid="framework-editor-page"` must render the creation form; submitting must call the create API and navigate back to the list.
- Steps 2–3 (assignment and validation) are covered by S11.15 and S11.16.

**Component `data-testid` registry** — all `data-testid` values listed in AC3–AC14 must be present exactly as specified. Playwright locators use these IDs directly. Do NOT rename or omit them.

**Admin-only access** — This frontend is in the `apps/admin` app which is protected by `AuthGuard` in the layout (AC4 already enforced by parent layout). The `useComplianceFrameworks` hook uses stub data in the stub phase. When wired to the real `admin-api`, the API client must send the admin JWT; the backend enforces 403 for non-admin tokens (E11-P0-006 — covered in S11.08 backend tests, not re-tested here at the frontend E2E layer).

**Delete 409 conflict** — `data-testid="framework-delete-error"` must show `t("compliance.list.deleteBlockedError")` (not the generic error) when the stub/real API returns HTTP 409. QA will test this by selecting a framework configured to return 409 in the mock.

---

## Dev Agent Record

### Implementation Plan

Story 11.14 was implemented in two phases:

1. **Initial implementation** (pre-review): Created all files listed in the architecture section — API client module, React Query hooks, i18n translations (EN + BG), navigation update, Framework List page, Framework Editor page, route shells. Build and type-check passed.

2. **Review follow-up** (2026-04-10): Applied all 10 patch findings from the code review (0 HIGH blocked, resolved 1 HIGH + 5 MEDIUM + 4 LOW).

### Completion Notes

✅ **All 10 review patch findings resolved (2026-04-10):**

- ✅ Resolved review finding [HIGH]: Delete error else-branch now uses `tErrors("serverError")` for generic errors; 409 path retains `t("list.deleteBlockedError")`.
- ✅ Resolved review finding [MEDIUM]: Removed duplicate `queryClient.invalidateQueries()` calls from `handleToggleActive` and `handleConfirmDelete` — hooks already invalidate in `onSuccess`.
- ✅ Resolved review finding [MEDIUM]: Pagination now uses `filteredTotal = filteredFrameworks.length` for display counts (`from`, `to`, `total`) and next-button disabled logic, correctly reflecting client-side type/search filtering.
- ✅ Resolved review finding [MEDIUM]: Country filter dropdown now derives unique countries from the loaded dataset via `Array.from(new Set((data?.frameworks ?? []).map(f => f.country)))`. Removed `STUB_COUNTRIES` constant.
- ✅ Resolved review finding [MEDIUM]: `ruleErrors` now keyed by `localId` (string) instead of array index (number). `removeRule`, `onSubmit` validation loop, and JSX error lookups all updated. Errors survive rule reorder/remove correctly.
- ✅ Resolved review finding [MEDIUM]: Added early guard `if (mode === "edit" && !frameworkId)` that returns an error state before any hook calls or JSX render, preventing the unsafe `frameworkId!` assertion from reaching `updateComplianceFramework`.
- ✅ Resolved review finding [LOW]: `onOpenChange` handler on the delete Dialog now also calls `setDeleteError(null)` when dialog closes via backdrop/Escape key, preventing stale error display on reopen.
- ✅ Resolved review finding [LOW]: Added `<label>` element above the check-type `<Select>` using `t("editor.rulesCheckType")`, making the field accessible and consuming the previously unused i18n key.
- ✅ Resolved review finding [LOW]: `useComplianceFramework` hook signature updated from `(id: string)` to `(id: string | undefined)`; `FrameworkEditor` now passes `frameworkId` directly (no `?? ""`), avoiding the `["compliance-frameworks", ""]` cache key.
- ✅ Resolved review finding [LOW]: Added `?? []` guard to `existingFramework.rules` in the `useEffect` that initialises rules in edit mode, preventing a throw if `rules` is absent in a malformed response.

**Verification:** `pnpm type-check` — 4/4 packages pass. `pnpm build` — 2/2 apps (client + admin) compile successfully with all compliance routes.

✅ **Second review patch resolved (2026-04-10):**

- ✅ Resolved review finding [HIGH]: Added `editor.rulesEmpty`, `editor.previewEmpty`, and `editor.ruleDefaultLabel` i18n keys to both `en.json` and `bg.json`. Replaced the three hardcoded English strings in `FrameworkEditor.tsx` (lines 476, 636, 647) with `t("editor.rulesEmpty")`, `t("editor.previewEmpty")`, and `t("editor.ruleDefaultLabel", { index: index + 1 })` respectively. `pnpm check:i18n --filter admin` exits 0 (135 keys matched). `pnpm type-check` exits 0 (4/4 packages). `pnpm build` exits 0 (2/2 apps).

## File List

### New files
- `eusolicit-app/frontend/apps/admin/lib/api/compliance-frameworks.ts`
- `eusolicit-app/frontend/apps/admin/lib/queries/use-compliance-frameworks.ts`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/new/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/[id]/edit/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/components/FrameworkList.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/components/FrameworkEditor.tsx`

### Modified files
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/layout.tsx` (ShieldCheck nav item)
- `eusolicit-app/frontend/apps/admin/messages/en.json` (nav.compliance + compliance namespace)
- `eusolicit-app/frontend/apps/admin/messages/bg.json` (nav.compliance + compliance namespace)

## Change Log

- **2026-04-10** — Applied second-review patch finding: added 3 i18n keys (`editor.rulesEmpty`, `editor.previewEmpty`, `editor.ruleDefaultLabel`) to `en.json` and `bg.json`; replaced 3 hardcoded English strings in `FrameworkEditor.tsx` with `t(...)` calls (AC17 HIGH). `check:i18n` exits 0, `type-check` exits 0 (4/4 packages), `build` exits 0 (2/2 apps). Status → review.
- **2026-04-10** — Applied 10 code review patch findings: fixed delete error handler (HIGH), removed double cache invalidation (MEDIUM×2), fixed pagination filtered totals (MEDIUM), replaced STUB_COUNTRIES with dataset-derived values (MEDIUM), fixed ruleErrors stale index by keying on localId (MEDIUM), added frameworkId edit-mode guard (MEDIUM), fixed dialog onOpenChange deleteError clear (LOW), added rulesCheckType label (LOW), fixed useComplianceFramework string|undefined signature (LOW), added rules ?? [] guard (LOW). Status → review.
- **2026-04-09** — Initial implementation of all Tasks 1–7: API client module, React Query hooks, i18n (EN + BG), navigation update, FrameworkList component, FrameworkEditor component, route page shells. Build + type-check verified.
