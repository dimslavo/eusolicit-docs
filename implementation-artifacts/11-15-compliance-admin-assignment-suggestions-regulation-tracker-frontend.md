# Story 11.15: Compliance Admin — Assignment, Suggestions & Regulation Tracker Frontend

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **platform administrator on EU Solicit**,
I want three new sub-pages within the compliance admin section — a Framework Assignment page, an Auto-Suggestion Queue, and a Regulation Tracker Dashboard — plus a shared sub-navigation bar linking all compliance pages,
so that I can assign compliance frameworks to procurement opportunities, process AI-generated framework suggestions with confidence scores, and monitor and acknowledge regulatory changes that affect the platform's compliance rule sets.

## Acceptance Criteria

### Route Structure

1. **AC1** — Six new files are created under `apps/admin/app/[locale]/(protected)/compliance/`:
   - `apps/admin/app/[locale]/(protected)/compliance/assign/page.tsx` — Framework Assignment page shell (server component, no `"use client"`)
   - `apps/admin/app/[locale]/(protected)/compliance/assign/components/FrameworkAssignmentPage.tsx` — client component
   - `apps/admin/app/[locale]/(protected)/compliance/suggestions/page.tsx` — Suggestion Queue page shell (server component)
   - `apps/admin/app/[locale]/(protected)/compliance/suggestions/components/SuggestionQueue.tsx` — client component
   - `apps/admin/app/[locale]/(protected)/compliance/regulations/page.tsx` — Regulation Tracker page shell (server component)
   - `apps/admin/app/[locale]/(protected)/compliance/regulations/components/RegulationTracker.tsx` — client component

   All page shells are minimal server component wrappers that import and render their respective client component. No `"use client"` on page shells.

---

### Compliance Sub-Navigation

2. **AC2** — A shared `ComplianceSubNav` component is created at `apps/admin/app/[locale]/(protected)/compliance/components/ComplianceSubNav.tsx` with `"use client"` directive. It renders a horizontal tab-style nav bar (`data-testid="compliance-subnav"`, `className="flex gap-1 border-b border-slate-200 mb-6"`) with four tab entries:
   - **Frameworks** → `/${locale}/compliance` — label: `t("compliance.nav.frameworks")`
   - **Assign** → `/${locale}/compliance/assign` — label: `t("compliance.nav.assign")`
   - **Suggestions** → `/${locale}/compliance/suggestions` — label: `t("compliance.nav.suggestions")`
   - **Regulations** → `/${locale}/compliance/regulations` — label: `t("compliance.nav.regulations")`

   Active-tab detection uses `usePathname()` from `next/navigation`:
   ```typescript
   const pathParts = pathname.split('/').filter(Boolean);
   // pathParts: ['bg', 'compliance'] or ['bg', 'compliance', 'assign'] etc.
   const complianceSegment = pathParts[2] ?? ''; // 'assign' | 'suggestions' | 'regulations' | '' | 'new' | id
   const SUB_ROUTES = ['assign', 'suggestions', 'regulations'];

   const isActive = (slug: string) => {
     if (slug === 'frameworks') return !SUB_ROUTES.includes(complianceSegment);
     return complianceSegment === slug;
   };
   ```

   Each tab renders as `<Link href={\`/${locale}/${slug === 'frameworks' ? 'compliance' : \`compliance/${slug}\`}\`} data-testid={isActive(slug) ? \`compliance-subnav-${slug}-active\` : \`compliance-subnav-${slug}\`} className={isActive(slug) ? "pb-2 px-3 text-sm font-semibold text-indigo-700 border-b-2 border-indigo-600" : "pb-2 px-3 text-sm text-slate-600 hover:text-slate-900 border-b-2 border-transparent"}>`. Uses `<Link>` from `next/link`. Locale obtained via `useParams<{ locale: string }>()`.

3. **AC3** — `ComplianceSubNav` is imported and rendered as the **first child** inside the root element of:
   - `FrameworkList.tsx` (existing S11.14 component — **minimal modification**: insert `<ComplianceSubNav />` before the `data-testid="compliance-page-header"` div; do not alter any other existing logic, state, or JSX from S11.14)
   - `FrameworkAssignmentPage.tsx` (new)
   - `SuggestionQueue.tsx` (new)
   - `RegulationTracker.tsx` (new)

---

### Framework Assignment Page

4. **AC4** — `FrameworkAssignmentPage` (`apps/admin/app/[locale]/(protected)/compliance/assign/components/FrameworkAssignmentPage.tsx`) is a `"use client"` component. Root: `<div data-testid="compliance-assignment-page">`. On mount it calls `useOpportunities()`. Page header (`data-testid="assignment-page-header"`):
   - `<h1 data-testid="assignment-page-title" className="text-2xl font-bold text-slate-900">t("compliance.assign.pageTitle")</h1>`
   - `<p data-testid="assignment-page-description" className="text-sm text-slate-500 mt-1">t("compliance.assign.pageDescription")</p>`

   Below the header, the main content: `<div className="grid grid-cols-1 lg:grid-cols-5 gap-6 mt-6">` — left column `className="lg:col-span-2"` (opportunity selector), right column `className="lg:col-span-3"` (assigned frameworks panel).

5. **AC5 — Left column: Opportunity Selector.** Inside `<div data-testid="opportunity-selector-panel">`:
   - `<h2 data-testid="opportunity-selector-heading" className="text-base font-semibold text-slate-800 mb-3">t("compliance.assign.opportunitySelector")</h2>`
   - **Search input** — `<Input data-testid="opportunity-search-input" placeholder={t("compliance.assign.opportunitySearch")} />` with debounced `useState<string>("")` (`debouncedSearch`, 300 ms via `useEffect`) filtering client-side: `opp.name.toLowerCase().includes(debouncedSearch.toLowerCase())`.
   - While `useOpportunities` loading: `<SkeletonTable rows={4} data-testid="opportunity-list-skeleton" />`.
   - On success, `<ul data-testid="opportunity-list" className="border border-slate-200 rounded-lg divide-y divide-slate-100 max-h-[520px] overflow-y-auto">`. Each item:
     ```tsx
     <li
       key={opp.id}
       data-testid={`opportunity-item-${opp.id}`}
       onClick={() => setSelectedOpportunityId(opp.id)}
       className={`px-3 py-2.5 cursor-pointer transition-colors ${
         selectedOpportunityId === opp.id
           ? "bg-indigo-50 border-l-4 border-l-indigo-500"
           : "hover:bg-slate-50 border-l-4 border-l-transparent"
       }`}
     >
       <p data-testid={`opportunity-name-${opp.id}`} className="text-sm font-medium text-slate-900">{opp.name}</p>
       <p className="text-xs text-slate-400 mt-0.5">{opp.source} · {opp.country}</p>
     </li>
     ```
   - `selectedOpportunityId` tracked via `useState<string | null>(null)`.
   - Filtered empty: `<p data-testid="opportunity-list-empty" className="text-sm text-slate-500 py-6 text-center px-3">t("compliance.assign.noOpportunities")</p>`

6. **AC6 — Right column: Assigned Frameworks Panel.** Inside `<div data-testid="assigned-frameworks-panel">`:
   - `<h2 data-testid="assigned-frameworks-heading" className="text-base font-semibold text-slate-800 mb-3">t("compliance.assign.assignedFrameworks")</h2>`
   - When `selectedOpportunityId === null`: `<EmptyState data-testid="assignment-select-opportunity-empty" icon={ShieldCheck} title={t("compliance.assign.selectOpportunity")} description={t("compliance.assign.selectOpportunityHint")} />` (no action prop).
   - When `selectedOpportunityId` is set, calls `useOpportunityFrameworks(selectedOpportunityId)`. While loading: `<SkeletonTable rows={3} data-testid="assigned-frameworks-skeleton" />`. On success:
     - If `assignments.length === 0`: `<p data-testid="assigned-frameworks-empty" className="text-sm text-slate-500 py-4">t("compliance.assign.noAssignedFrameworks")</p>`
     - If assignments exist: `<table data-testid="assigned-frameworks-table" className="w-full text-sm">` with columns: Framework name, Country/Type, Assigned by, Assigned at, Actions. Each row `<tr data-testid={`assignment-row-${a.framework_id}`}>`:
       - **Name** — `<td data-testid={`assigned-fw-name-${a.framework_id}`} className="font-medium text-slate-900 py-2 pr-3">{a.framework_name}</td>`
       - **Type** — `<td><Badge data-testid={`assigned-fw-type-${a.framework_id}`} className={a.regulation_type === "national" ? "bg-blue-100 text-blue-800" : a.regulation_type === "eu" ? "bg-indigo-100 text-indigo-800" : "bg-green-100 text-green-800"}>{t(\`compliance.type.${a.regulation_type}\`)}</Badge></td>` (reuses `compliance.type.*` keys from S11.14)
       - **Assigned by** — `<td data-testid={`assigned-fw-by-${a.framework_id}`} className="text-slate-500 text-xs">{a.assigned_by}</td>`
       - **Assigned at** — `<td data-testid={`assigned-fw-at-${a.framework_id}`} className="text-slate-400 text-xs">{new Date(a.assigned_at).toLocaleDateString(locale)}</td>`
       - **Remove** — `<td><Button variant="ghost" size="sm" data-testid={`remove-framework-btn-${a.framework_id}`} className="text-red-600 hover:text-red-700" disabled={removingFrameworkId === a.framework_id} onClick={() => handleRemove(a.framework_id)}>{removingFrameworkId === a.framework_id ? <Loader2 className="animate-spin w-3 h-3" /> : t("compliance.assign.removeBtn")}</Button></td>`
       - `removingFrameworkId` tracked via `useState<string | null>(null)`; set on click, cleared on success/error via `useRemoveFramework()` mutation callbacks.

7. **AC7 — Add Framework section.** Below the assigned frameworks list (still inside the right column), an add-framework section renders **only** when `selectedOpportunityId !== null` (`data-testid="add-framework-section"`, `className="mt-4 pt-4 border-t border-slate-200"`):
   - `<Select data-testid="framework-add-select">` populated from `useComplianceFrameworks({ is_active: true })` (imported from `use-compliance-frameworks.ts` — **reuse existing hook from S11.14**). Default: `<SelectItem value="">t("compliance.assign.frameworkPlaceholder")</SelectItem>` + one `<SelectItem value={fw.id}>` per active framework (`{fw.name} ({fw.country})`). Managed via `useState<string>("")` for `selectedFrameworkId`.
   - `<Button data-testid="add-framework-btn" disabled={!selectedFrameworkId || assignMutation.isPending} onClick={handleAddFramework} className="mt-2">t("compliance.assign.addBtn")</Button>`. While `assignMutation.isPending`: `<Loader2 className="animate-spin w-3 h-3" />` shown + button disabled. On success: reset `selectedFrameworkId` to `""`, call `queryClient.invalidateQueries({ queryKey: ["opportunity-frameworks", selectedOpportunityId] })`.
   - On error HTTP 409: `<p data-testid="add-framework-duplicate-error" className="text-sm text-red-600 mt-2">t("compliance.assign.alreadyAssigned")</p>`.
   - On other error: `<p data-testid="add-framework-error" className="text-sm text-red-600 mt-2">t("errors.serverError")</p>`.
   - Error state: `useState<"duplicate" | "server" | null>(null)` for `addError`; cleared when `selectedFrameworkId` changes.

---

### Auto-Suggestion Queue Page

8. **AC8** — `SuggestionQueue` (`apps/admin/app/[locale]/(protected)/compliance/suggestions/components/SuggestionQueue.tsx`) is a `"use client"` component. Root: `<div data-testid="compliance-suggestion-queue-page">`. Calls `useFrameworkSuggestions(filterParams)` where `filterParams = { date_from: dateFrom || undefined, date_to: dateTo || undefined }` (status `"pending"` is always applied server-side via the hook default). Page header:
   - `<h1 data-testid="suggestions-page-title" className="text-2xl font-bold text-slate-900">t("compliance.suggestions.pageTitle")</h1>`
   - `<p data-testid="suggestions-page-description" className="text-sm text-slate-500 mt-1">t("compliance.suggestions.pageDescription")</p>`

9. **AC9 — Suggestion Filters & Batch Controls.** A filter section (`data-testid="suggestions-filters-section"`, `className="flex flex-wrap items-center gap-3 mt-4 mb-4"`) renders:
   - **Min confidence input** — `<Input type="number" min={0} max={100} data-testid="suggestions-filter-confidence" placeholder={t("compliance.suggestions.filterConfidencePlaceholder")} className="w-32" />` — managed via `useState<number>(0)` for `minConfidence`. Client-side filter: `suggestion.confidence >= minConfidence`.
   - **Date from** — `<Input type="date" data-testid="suggestions-filter-date-from" className="w-40" />` — managed via `useState<string>("")` for `dateFrom`.
   - **Date to** — `<Input type="date" data-testid="suggestions-filter-date-to" className="w-40" />` — managed via `useState<string>("")` for `dateTo`.
   - **Batch Accept button** — `<Button data-testid="suggestions-batch-accept-btn" disabled={selectedIds.size === 0 || batchProcessing} onClick={handleBatchAccept} variant="outline" className="ml-auto">{batchProcessing ? <Loader2 className="animate-spin w-3 h-3 mr-1" /> : null}{t("compliance.suggestions.batchAcceptBtn")} ({selectedIds.size})</Button>`. Enabled only when `selectedIds.size > 0`. `batchProcessing` via `useState<boolean>(false)`.
   - Selected rows tracked via `useState<Set<string>>(new Set())` for `selectedIds`.

10. **AC10 — Suggestion Table.** While loading: `<SkeletonTable rows={4} data-testid="suggestions-list-skeleton" />`. Apply client-side confidence filter: `const visibleSuggestions = (data ?? []).filter(s => s.confidence >= minConfidence)`. On query success with 1+ visible results:

    `<table data-testid="suggestions-table" className="w-full text-sm">` with header:
    - `<th>` — `<input type="checkbox" data-testid="suggestions-select-all" checked={selectedIds.size === visibleSuggestions.length && visibleSuggestions.length > 0} onChange={toggleSelectAll} />` (selects/deselects all visible suggestions)
    - `<th>t("compliance.suggestions.colOpportunity")</th>`
    - `<th>t("compliance.suggestions.colFramework")</th>`
    - `<th>t("compliance.suggestions.colConfidence")</th>`
    - `<th>t("compliance.suggestions.colActions")</th>`

    Each suggestion row `<tr data-testid={`suggestion-row-${sug.id}`}>`:
    - **Checkbox** — `<td><input type="checkbox" data-testid={`suggestion-checkbox-${sug.id}`} checked={selectedIds.has(sug.id)} onChange={() => toggleSuggestionSelection(sug.id)} /></td>`
    - **Opportunity** — `<td data-testid={`suggestion-opportunity-${sug.id}`} className="font-medium text-slate-900">{sug.opportunity_name}</td>`
    - **Framework** — `<td data-testid={`suggestion-framework-${sug.id}`} className="text-slate-700">{sug.framework_name}</td>`
    - **Confidence** — `<td data-testid={`suggestion-confidence-cell-${sug.id}`}>`:
      ```tsx
      <div className="flex items-center gap-2 min-w-[120px]">
        <div className="flex-1 h-2 rounded-full bg-slate-100 overflow-hidden">
          <div
            data-testid={`confidence-bar-${sug.id}`}
            style={{ width: `${sug.confidence}%` }}
            className={
              sug.confidence >= 80 ? "h-full bg-green-500" :
              sug.confidence >= 60 ? "h-full bg-amber-500" :
              "h-full bg-red-500"
            }
          />
        </div>
        <span
          data-testid={`confidence-score-${sug.id}`}
          className={`text-xs font-medium ${
            sug.confidence >= 80 ? "text-green-700" :
            sug.confidence >= 60 ? "text-amber-700" :
            "text-red-700"
          }`}
        >{sug.confidence}%</span>
      </div>
      ```
    - **Actions** — `<td data-testid={`suggestion-actions-${sug.id}`}>` (disable all when `processingIds.has(sug.id)`):
      - Accept: `<Button size="sm" variant="outline" data-testid={`suggestion-accept-btn-${sug.id}`} disabled={processingIds.has(sug.id)} onClick={() => handleAccept(sug.id)} className="text-green-700 border-green-300 hover:bg-green-50">{processingIds.has(sug.id) ? <Loader2 className="animate-spin w-3 h-3" /> : t("compliance.suggestions.acceptBtn")}</Button>`
      - Override: (see AC11 for inline picker logic) — `<Button size="sm" variant="ghost" data-testid={`suggestion-override-btn-${sug.id}`} disabled={processingIds.has(sug.id)} onClick={() => setOverridingId(sug.id)}>t("compliance.suggestions.overrideBtn")</Button>`
      - Dismiss: `<Button size="sm" variant="ghost" data-testid={`suggestion-dismiss-btn-${sug.id}`} disabled={processingIds.has(sug.id)} onClick={() => handleDismiss(sug.id)} className="text-red-500 hover:text-red-700">t("compliance.suggestions.dismissBtn")</Button>`
    - `processingIds` tracked via `useState<Set<string>>(new Set())` for per-row in-flight state.

11. **AC11 — Override picker.** When `overridingId === sug.id`, the actions cell additionally renders an inline picker below the buttons:
    ```tsx
    <div data-testid={`override-picker-${sug.id}`} className="mt-2 flex items-center gap-2">
      <Select data-testid={`override-framework-select-${sug.id}`} value={overrideFrameworkId} onValueChange={setOverrideFrameworkId}>
        {/* Populated from useComplianceFrameworks({ is_active: true }) — reuse S11.14 hook */}
        <SelectItem value="">t("compliance.assign.frameworkPlaceholder")</SelectItem>
        {activeFrameworks.map(fw => <SelectItem key={fw.id} value={fw.id}>{fw.name}</SelectItem>)}
      </Select>
      <Button size="sm" data-testid={`override-confirm-btn-${sug.id}`} disabled={!overrideFrameworkId} onClick={() => handleOverride(sug.id, overrideFrameworkId!)}>t("compliance.suggestions.overrideConfirmBtn")</Button>
      <Button size="sm" variant="ghost" data-testid={`override-cancel-btn-${sug.id}`} onClick={() => { setOverridingId(null); setOverrideFrameworkId(""); }}>t("common.cancel")</Button>
    </div>
    ```
    `overridingId` via `useState<string | null>(null)`; `overrideFrameworkId` via `useState<string>("")`. Active frameworks from `useComplianceFrameworks({ is_active: true })` — **reuse existing hook from S11.14**.

12. **AC12 — Suggestion actions.** All three actions call `useUpdateFrameworkSuggestion()` mutation:
    - `handleAccept(id)`: adds `id` to `processingIds` → `mutate({ id, data: { status: "accepted" } })`. On success: `invalidateQueries({ queryKey: ["framework-suggestions"] })`, toast `t("compliance.suggestions.acceptSuccess")`, remove `id` from `processingIds`.
    - `handleOverride(id, overrideFrameworkId)`: adds `id` to `processingIds` → `mutate({ id, data: { status: "accepted", override_framework_id: overrideFrameworkId } })`. On success: same as accept. Reset `overridingId` to `null`.
    - `handleDismiss(id)`: adds `id` to `processingIds` → `mutate({ id, data: { status: "rejected" } })`. On success: `invalidateQueries`, toast `t("compliance.suggestions.dismissSuccess")`, remove `id` from `processingIds`.
    - On any error: `toast({ title: t("errors.serverError"), variant: "destructive" })`, remove `id` from `processingIds`.
    - All use the **same** `updateSuggestionMutation` instance (one `useUpdateFrameworkSuggestion()` call in the component).

13. **AC13 — Batch Accept.** `handleBatchAccept` sets `batchProcessing = true`, adds all `selectedIds` to `processingIds`, then calls `updateSuggestionMutation.mutateAsync({ id, data: { status: "accepted" } })` in `Promise.allSettled` for each selected ID. On all settled: clear `processingIds` for all processed IDs, clear `selectedIds`, set `batchProcessing = false`. Count fulfilled vs rejected. If all fulfilled: toast `t("compliance.suggestions.batchAcceptSuccess")`. If any rejected: toast `t("errors.serverError", { context: "partial" })` (or generic error toast). `invalidateQueries({ queryKey: ["framework-suggestions"] })` after settlement.

14. **AC14 — Empty state.** When `visibleSuggestions.length === 0` (after confidence filter), `<EmptyState data-testid="suggestions-empty-state" icon={ShieldCheck} title={t("compliance.suggestions.emptyTitle")} description={t("compliance.suggestions.emptyDescription")} />`.

---

### Regulation Tracker Dashboard Page

15. **AC15** — `RegulationTracker` (`apps/admin/app/[locale]/(protected)/compliance/regulations/components/RegulationTracker.tsx`) is a `"use client"` component. Root: `<div data-testid="compliance-regulation-tracker-page">`. Calls `useRegulatoryChanges(filterParams)` where `filterParams` derives from filter state. Default: `filterStatus = "new"` (shows only unreviewed changes on first load). Page header:
    - `<h1 data-testid="regulations-page-title" className="text-2xl font-bold text-slate-900">t("compliance.regulations.pageTitle")</h1>`
    - `<p data-testid="regulations-page-description" className="text-sm text-slate-500 mt-1">t("compliance.regulations.pageDescription")</p>`

16. **AC16 — Regulation Filters.** Filter section (`data-testid="regulations-filters-section"`, `className="flex flex-wrap items-center gap-3 mt-4 mb-6"`):
    - **Status** — `<Select data-testid="regulations-filter-status" value={filterStatus} onValueChange={setFilterStatus}>`:
      - `<SelectItem value="all">t("compliance.regulations.statusAll")</SelectItem>`
      - `<SelectItem value="new">t("compliance.regulations.status.new")</SelectItem>`
      - `<SelectItem value="acknowledged">t("compliance.regulations.status.acknowledged")</SelectItem>`
      - `<SelectItem value="dismissed">t("compliance.regulations.status.dismissed")</SelectItem>`
      Managed via `useState<string>("new")`.
    - **Severity** — `<Select data-testid="regulations-filter-severity" value={filterSeverity} onValueChange={setFilterSeverity}>` with options: `<SelectItem value="">t("compliance.regulations.statusAll")</SelectItem>`, `<SelectItem value="high">t("compliance.regulations.severity.high")</SelectItem>`, `<SelectItem value="medium">t("compliance.regulations.severity.medium")</SelectItem>`, `<SelectItem value="low">t("compliance.regulations.severity.low")</SelectItem>`. Managed via `useState<string>("")`.
    - **Date from** — `<Input type="date" data-testid="regulations-filter-date-from" value={dateFrom} onChange={e => setDateFrom(e.target.value)} className="w-40" />` — `useState<string>("")`.
    - **Date to** — `<Input type="date" data-testid="regulations-filter-date-to" value={dateTo} onChange={e => setDateTo(e.target.value)} className="w-40" />` — `useState<string>("")`.
    All filter state changes rebuild `filterParams = { status: filterStatus !== "all" ? filterStatus : undefined, severity: filterSeverity || undefined, date_from: dateFrom || undefined, date_to: dateTo || undefined }` and trigger re-query via the `useRegulatoryChanges(filterParams)` hook (query key includes params).

17. **AC17 — Change Feed.** While loading: `<div data-testid="regulations-feed-skeleton" className="flex flex-col gap-4">` with 3 `<SkeletonCard data-testid="regulation-card-skeleton" />` placeholders. On success with 1+ results: `<div data-testid="regulation-tracker-feed" className="flex flex-col gap-4">`. Each change renders:

    `<div data-testid={`regulation-card-${change.id}`} className={`border rounded-lg p-4 bg-white ${change.status === "dismissed" ? "opacity-60" : ""}`}>`:

    **Row 1 — Header:** `<div className="flex items-start justify-between mb-2">`:
    - Left: `<span data-testid={`regulation-source-${change.id}`} className="text-xs font-semibold text-slate-500 uppercase tracking-wide">{change.source}</span>`
    - Right: `<Badge data-testid={`regulation-type-badge-${change.id}`} className={change.change_type === "new" ? "bg-blue-100 text-blue-800" : change.change_type === "amended" ? "bg-amber-100 text-amber-800" : "bg-red-100 text-red-800"}>t(\`compliance.regulations.changeType.${change.change_type}\`)</Badge>`

    **Row 2 — Severity:** `<div data-testid={`regulation-severity-${change.id}`} className="flex items-center gap-1 mb-2">`:
    - `high` → `<AlertTriangle className="w-4 h-4 text-red-500" /><span className="text-red-700 text-sm font-medium">t("compliance.regulations.severity.high")</span>`
    - `medium` → `<AlertTriangle className="w-4 h-4 text-amber-500" /><span className="text-amber-700 text-sm font-medium">t("compliance.regulations.severity.medium")</span>`
    - `low` → `<Info className="w-4 h-4 text-slate-400" /><span className="text-slate-500 text-sm">t("compliance.regulations.severity.low")</span>`

    **Row 3 — Summary:** `<p data-testid={`regulation-summary-${change.id}`} className="text-sm text-slate-700 mb-2">{change.summary}</p>`

    **Row 4 — Date:** `<span data-testid={`regulation-detected-${change.id}`} className="text-xs text-slate-400">{new Date(change.detected_at).toLocaleDateString(locale)}</span>`

    **Row 5 — Affected Frameworks** (`data-testid={`regulation-affected-frameworks-${change.id}`}`, `className="flex flex-wrap gap-2 mt-2"`): for each `fw` in `change.affected_frameworks`:
    ```tsx
    <Link
      key={fw.id}
      data-testid={`affected-framework-chip-${change.id}-${fw.id}`}
      href={`/${locale}/compliance/${fw.id}/edit`}
      className="inline-flex items-center gap-1 px-2 py-0.5 rounded-full bg-indigo-50 text-indigo-700 text-xs hover:bg-indigo-100 transition-colors"
    >
      {fw.name}
    </Link>
    ```
    Uses `<Link>` from `next/link`.

    **Row 6 — Actions (status === "new" only):** `<div data-testid={`regulation-actions-${change.id}`} className="flex gap-2 mt-3">`:
    - `<Button size="sm" data-testid={`acknowledge-btn-${change.id}`} onClick={() => setAcknowledgingId(change.id)}>t("compliance.regulations.acknowledgeBtn")</Button>`
    - `<Button size="sm" variant="ghost" data-testid={`dismiss-btn-${change.id}`} disabled={dismissingIds.has(change.id)} onClick={() => handleDismiss(change.id)} className="text-slate-500 hover:text-slate-700">{dismissingIds.has(change.id) ? <Loader2 className="animate-spin w-3 h-3" /> : t("compliance.regulations.dismissBtn")}</Button>`

    **Row 6 — Acknowledged state (status === "acknowledged"):**
    - `<div data-testid={`regulation-acknowledged-info-${change.id}`} className="mt-2 text-xs text-slate-500">{t("compliance.regulations.acknowledgedBy", { by: change.acknowledged_by ?? "—", at: new Date(change.acknowledged_at!).toLocaleDateString(locale) })}</div>`
    - If `change.notes`: `<p data-testid={`regulation-notes-${change.id}`} className="text-xs text-slate-600 italic mt-1">{change.notes}</p>`

    **Row 6 — Dismissed state (status === "dismissed"):**
    - `<span data-testid={`regulation-dismissed-label-${change.id}`} className="text-xs text-slate-400 mt-2 inline-block">t("compliance.regulations.dismissedLabel")</span>`

18. **AC18 — Acknowledge inline panel.** When `acknowledgingId === change.id`, the acknowledge panel renders within the card (`data-testid={`acknowledge-panel-${change.id}`}`, `className="mt-3 p-3 bg-slate-50 rounded-lg"`):
    - `<Textarea data-testid={`acknowledge-notes-${change.id}`} placeholder={t("compliance.regulations.acknowledgeNotesPlaceholder")} rows={2} value={acknowledgeNotes[change.id] ?? ""} onChange={e => setAcknowledgeNotes(prev => ({ ...prev, [change.id]: e.target.value }))} className="mb-2" />`
    - `<div className="flex gap-2">`
      - `<Button size="sm" data-testid={`acknowledge-confirm-btn-${change.id}`} disabled={updateChangeMutation.isPending} onClick={() => handleAcknowledge(change.id)}>{updateChangeMutation.isPending ? <Loader2 className="animate-spin w-3 h-3 mr-1" /> : null}t("compliance.regulations.acknowledgeConfirmBtn")</Button>`
      - `<Button size="sm" variant="ghost" data-testid={`acknowledge-cancel-btn-${change.id}`} onClick={() => setAcknowledgingId(null)}>t("common.cancel")</Button>`

    `acknowledgeNotes` via `useState<Record<string, string>>({})` (keyed by change ID). `acknowledgingId` via `useState<string | null>(null)`.

    `handleAcknowledge(id)`: calls `updateChangeMutation.mutate({ id, data: { status: "acknowledged", notes: acknowledgeNotes[id] ?? undefined } })`. On success: `invalidateQueries({ queryKey: ["regulatory-changes"] })`, toast `t("compliance.regulations.acknowledgeSuccess")`, `setAcknowledgingId(null)`. On error: `toast({ title: t("errors.serverError"), variant: "destructive" })`.

19. **AC19 — Dismiss flow.** `handleDismiss(id)`: adds `id` to `dismissingIds` (`useState<Set<string>>(new Set())`), calls `updateChangeMutation.mutate({ id, data: { status: "dismissed" } })`. On success: `invalidateQueries({ queryKey: ["regulatory-changes"] })`, toast `t("compliance.regulations.dismissSuccess")`, remove `id` from `dismissingIds`. On error: toast error, remove `id` from `dismissingIds`.

    **Note:** `updateChangeMutation` is a single `useUpdateRegulatoryChange()` instance. Both acknowledge and dismiss share it. Because `isPending` on the shared mutation would block unrelated rows during an acknowledge operation, the separate `dismissingIds` Set provides per-row in-flight state for the dismiss button. The acknowledge confirm button uses `updateChangeMutation.isPending` since only one acknowledge panel is open at a time (`acknowledgingId` is scalar).

20. **AC20 — Empty state.** When feed has 0 results: `<EmptyState data-testid="regulations-empty-state" icon={ShieldCheck} title={t("compliance.regulations.emptyTitle")} description={t("compliance.regulations.emptyDescription")} />`.

---

### API Client Modules

21. **AC21** — File `apps/admin/lib/api/framework-assignments.ts` is created and exports:
    - TypeScript interfaces: `OpportunityItem`, `ListOpportunitiesResponse`, `FrameworkAssignment`, `AssignFrameworkRequest` (see Dev Notes for definitions)
    - `listOpportunities(params?: { search?: string; page?: number; page_size?: number }): Promise<ListOpportunitiesResponse>` — `apiClient.get<ListOpportunitiesResponse>("/api/v1/admin/opportunities", { params })`. Stub: returns `STUB_OPPORTUNITIES_LIST` after 500 ms.
    - `getOpportunityFrameworks(opportunityId: string): Promise<FrameworkAssignment[]>` — `apiClient.get<FrameworkAssignment[]>(\`/api/v1/admin/opportunities/${opportunityId}/compliance-frameworks\`)`. Stub: returns filtered `STUB_ASSIGNMENTS` matching `opportunityId` after 400 ms.
    - `assignFrameworkToOpportunity(opportunityId: string, data: AssignFrameworkRequest): Promise<FrameworkAssignment>` — `apiClient.post`. Stub: returns synthetic assignment after 600 ms (throws synthetic 409 if framework already in stub list for that opportunity).
    - `removeFrameworkFromOpportunity(opportunityId: string, frameworkId: string): Promise<void>` — `apiClient.delete`. Stub: resolves after 400 ms.
    - `import { apiClient } from "@eusolicit/ui"` (commented out in stub phase — uncomment when S11.09 backend is wired).

22. **AC22** — File `apps/admin/lib/api/framework-suggestions.ts` is created and exports:
    - TypeScript interfaces: `FrameworkSuggestion`, `ListSuggestionsParams`, `UpdateSuggestionRequest`, `UpdateSuggestionResponse` (see Dev Notes)
    - `listFrameworkSuggestions(params?: ListSuggestionsParams): Promise<FrameworkSuggestion[]>` — `apiClient.get<FrameworkSuggestion[]>("/api/v1/admin/framework-suggestions", { params })`. Stub: returns `STUB_SUGGESTIONS` (status always `"pending"`) filtered by `params.date_from`/`date_to` if set, after 500 ms.
    - `updateFrameworkSuggestion(id: string, data: UpdateSuggestionRequest): Promise<FrameworkSuggestion>` — `apiClient.patch`. Stub: returns updated suggestion after 600 ms.

23. **AC23** — File `apps/admin/lib/api/regulatory-changes.ts` is created and exports:
    - TypeScript interfaces: `RegulatoryChange`, `AffectedFramework`, `ListRegulatoryChangesParams`, `UpdateRegulatoryChangeRequest` (see Dev Notes)
    - `listRegulatoryChanges(params?: ListRegulatoryChangesParams): Promise<RegulatoryChange[]>` — `apiClient.get<RegulatoryChange[]>("/api/v1/admin/regulatory-changes", { params })`. Stub: returns `STUB_REGULATORY_CHANGES` filtered by `params.status` (default filter `"new"` if no status param) after 500 ms.
    - `updateRegulatoryChange(id: string, data: UpdateRegulatoryChangeRequest): Promise<RegulatoryChange>` — `apiClient.patch`. Stub: returns updated change after 600 ms.

---

### React Query Hooks

24. **AC24** — File `apps/admin/lib/queries/use-framework-assignments.ts` (`"use client"`) exports:
    - `useOpportunities(params?: Parameters<typeof listOpportunities>[0])` — `useQuery({ queryKey: ["opportunities", params], queryFn: () => listOpportunities(params), staleTime: 60_000 })`
    - `useOpportunityFrameworks(opportunityId: string | null)` — `useQuery({ queryKey: ["opportunity-frameworks", opportunityId], queryFn: () => getOpportunityFrameworks(opportunityId!), enabled: !!opportunityId, staleTime: 30_000 })`
    - `useAssignFramework()` — `useMutation({ mutationFn: ({ opportunityId, data }: { opportunityId: string; data: AssignFrameworkRequest }) => assignFrameworkToOpportunity(opportunityId, data) })` — onSuccess callback invalidates `["opportunity-frameworks", opportunityId]` via vars.
    - `useRemoveFramework()` — `useMutation({ mutationFn: ({ opportunityId, frameworkId }: { opportunityId: string; frameworkId: string }) => removeFrameworkFromOpportunity(opportunityId, frameworkId) })` — onSuccess invalidates `["opportunity-frameworks", opportunityId]` via vars.
    - `queryClient` obtained via `useQueryClient()`.

25. **AC25** — File `apps/admin/lib/queries/use-framework-suggestions.ts` (`"use client"`) exports:
    - `useFrameworkSuggestions(params?: ListSuggestionsParams)` — `useQuery({ queryKey: ["framework-suggestions", params], queryFn: () => listFrameworkSuggestions(params), staleTime: 30_000 })`
    - `useUpdateFrameworkSuggestion()` — `useMutation({ mutationFn: ({ id, data }: { id: string; data: UpdateSuggestionRequest }) => updateFrameworkSuggestion(id, data), onSuccess: () => queryClient.invalidateQueries({ queryKey: ["framework-suggestions"] }) })`

26. **AC26** — File `apps/admin/lib/queries/use-regulatory-changes.ts` (`"use client"`) exports:
    - `useRegulatoryChanges(params?: ListRegulatoryChangesParams)` — `useQuery({ queryKey: ["regulatory-changes", params], queryFn: () => listRegulatoryChanges(params), staleTime: 30_000 })`
    - `useUpdateRegulatoryChange()` — `useMutation({ mutationFn: ({ id, data }: { id: string; data: UpdateRegulatoryChangeRequest }) => updateRegulatoryChange(id, data), onSuccess: () => queryClient.invalidateQueries({ queryKey: ["regulatory-changes"] }) })`

---

### i18n

27. **AC27** — All user-visible strings use `useTranslations("compliance")`, `useTranslations("common")`, and `useTranslations("errors")`. New translation keys are added to `apps/admin/messages/en.json` and `apps/admin/messages/bg.json` **under the existing `"compliance"` namespace** (appended alongside the keys from S11.14 — do not overwrite existing keys). New keys listed in Dev Notes (see i18n Values section). `pnpm check:i18n --filter admin` must exit 0.

---

### Build & Type-Check

28. **AC28** — `pnpm build` exits 0 for both `client` and `admin` apps. `pnpm type-check` exits 0 across all packages. No TypeScript errors in any new or modified file.

---

## Tasks / Subtasks

- [x] **Task 1: Create API client modules** (AC: 21, 22, 23)
  - [x] 1.1 Create `apps/admin/lib/api/framework-assignments.ts`:
    - Define `OpportunityItem`, `ListOpportunitiesResponse`, `FrameworkAssignment`, `AssignFrameworkRequest` interfaces (see Dev Notes)
    - Implement `STUB_OPPORTUNITIES_LIST` and `STUB_ASSIGNMENTS` constants (see Dev Notes for fixture data)
    - Implement all 4 API functions with stub delays; wire real `apiClient` calls behind comments
  - [x] 1.2 Create `apps/admin/lib/api/framework-suggestions.ts`:
    - Define `FrameworkSuggestion`, `ListSuggestionsParams`, `UpdateSuggestionRequest`, `UpdateSuggestionResponse` interfaces
    - Implement `STUB_SUGGESTIONS` constant (see Dev Notes)
    - Implement `listFrameworkSuggestions()` and `updateFrameworkSuggestion()` stubs
  - [x] 1.3 Create `apps/admin/lib/api/regulatory-changes.ts`:
    - Define `AffectedFramework`, `RegulatoryChange`, `ListRegulatoryChangesParams`, `UpdateRegulatoryChangeRequest` interfaces
    - Implement `STUB_REGULATORY_CHANGES` constant (see Dev Notes)
    - Implement `listRegulatoryChanges()` and `updateRegulatoryChange()` stubs

- [x] **Task 2: Create React Query hooks** (AC: 24, 25, 26)
  - [x] 2.1 Create `apps/admin/lib/queries/use-framework-assignments.ts` — 4 hooks
  - [x] 2.2 Create `apps/admin/lib/queries/use-framework-suggestions.ts` — 2 hooks
  - [x] 2.3 Create `apps/admin/lib/queries/use-regulatory-changes.ts` — 2 hooks

- [x] **Task 3: Add i18n translation keys** (AC: 27)
  - [x] 3.1 Add new keys under existing `"compliance"` namespace in `apps/admin/messages/en.json` — `nav.*`, `assign.*`, `suggestions.*`, `regulations.*` (see Dev Notes for exact English values)
  - [x] 3.2 Add corresponding Bulgarian translations to `apps/admin/messages/bg.json` (see Dev Notes for Bulgarian values)
  - [x] 3.3 Run `pnpm check:i18n --filter admin` — must exit 0 ✅ (196 keys matched)

- [x] **Task 4: Create ComplianceSubNav component** (AC: 2)
  - [x] 4.1 Create `apps/admin/app/[locale]/(protected)/compliance/components/ComplianceSubNav.tsx`
  - [x] 4.2 Implement `"use client"` directive; import `usePathname`, `useParams` from `next/navigation`, `Link` from `next/link`, `useTranslations` from `next-intl`
  - [x] 4.3 Implement active-tab detection logic (see AC2 for path-splitting algorithm)
  - [x] 4.4 Render 4 tab links with correct `data-testid` attributes

- [x] **Task 5: Modify FrameworkList to add ComplianceSubNav** (AC: 3)
  - [x] 5.1 Open `apps/admin/app/[locale]/(protected)/compliance/components/FrameworkList.tsx`
  - [x] 5.2 Import `ComplianceSubNav` from `"../components/ComplianceSubNav"` (relative import within components directory — actually: `import { ComplianceSubNav } from "./ComplianceSubNav"`)
  - [x] 5.3 Insert `<ComplianceSubNav />` as the first child inside `<div data-testid="compliance-framework-list-page">`, before `<div data-testid="compliance-page-header">`
  - [x] 5.4 **Do not change any other code** in FrameworkList.tsx

- [x] **Task 6: Build Framework Assignment Page** (AC: 4, 5, 6, 7)
  - [x] 6.1 Create `apps/admin/app/[locale]/(protected)/compliance/assign/page.tsx` — server shell rendering `<FrameworkAssignmentPage />`
  - [x] 6.2 Create `apps/admin/app/[locale]/(protected)/compliance/assign/components/` directory
  - [x] 6.3 Create `apps/admin/app/[locale]/(protected)/compliance/assign/components/FrameworkAssignmentPage.tsx`:
    - `"use client"` directive; imports from `@eusolicit/ui`, `lucide-react`, `next/navigation`, `next-intl`, `next/link`
    - Import `ComplianceSubNav` from `"../../components/ComplianceSubNav"`
    - Import `useComplianceFrameworks` from `"@/lib/queries/use-compliance-frameworks"` (S11.14 — reuse existing hook)
    - Import `useOpportunities`, `useOpportunityFrameworks`, `useAssignFramework`, `useRemoveFramework` from `"@/lib/queries/use-framework-assignments"`
    - Implement two-column grid layout (`lg:grid-cols-5`): left `lg:col-span-2`, right `lg:col-span-3`
    - Opportunity list with debounced search, selection state, skeleton
    - Assigned frameworks table with regulation_type badge (same colour mapping as S11.14 AC6), remove button with per-row loading
    - Add-framework Select + Assign button; duplicate error detection via `error.response?.status === 409`

- [x] **Task 7: Build Auto-Suggestion Queue Page** (AC: 8, 9, 10, 11, 12, 13, 14)
  - [x] 7.1 Create `apps/admin/app/[locale]/(protected)/compliance/suggestions/page.tsx` — server shell
  - [x] 7.2 Create `apps/admin/app/[locale]/(protected)/compliance/suggestions/components/` directory
  - [x] 7.3 Create `apps/admin/app/[locale]/(protected)/compliance/suggestions/components/SuggestionQueue.tsx`:
    - `"use client"` directive; imports from `@eusolicit/ui`, `lucide-react`, `next/navigation`, `next-intl`
    - Import `ComplianceSubNav` from `"../../components/ComplianceSubNav"`
    - Import `useComplianceFrameworks` for override picker framework list (S11.14 reuse)
    - Filters: min confidence input, date range inputs, batch accept button with selectedIds count
    - Table with select-all checkbox; per-row: checkbox, opportunity, framework, confidence bar (colour-coded via inline style + className), three action buttons
    - Override picker: inline Select + confirm/cancel (renders only when `overridingId === sug.id`)
    - Per-row processing state via `processingIds` Set
    - Batch accept via `Promise.allSettled`

- [x] **Task 8: Build Regulation Tracker Dashboard Page** (AC: 15, 16, 17, 18, 19, 20)
  - [x] 8.1 Create `apps/admin/app/[locale]/(protected)/compliance/regulations/page.tsx` — server shell
  - [x] 8.2 Create `apps/admin/app/[locale]/(protected)/compliance/regulations/components/` directory
  - [x] 8.3 Create `apps/admin/app/[locale]/(protected)/compliance/regulations/components/RegulationTracker.tsx`:
    - `"use client"` directive; imports including `Link` from `next/link`, `AlertTriangle`, `Info` from `lucide-react`
    - Import `ComplianceSubNav` from `"../../components/ComplianceSubNav"`
    - Default filter: `filterStatus = "new"` (shows only unreviewed on load)
    - Card feed: change_type badge (blue/amber/red), severity icon+text (AlertTriangle/Info), summary, detected date, affected framework chips (as `<Link>` to editor page)
    - `status === "new"`: show acknowledge + dismiss buttons
    - `status === "acknowledged"`: show acknowledgedBy text + notes
    - `status === "dismissed"`: muted card + dismissed label
    - Acknowledge inline panel: Textarea + confirm/cancel
    - `dismissingIds` Set for per-row dismiss in-flight state
    - `acknowledgeNotes` Record keyed by change ID

- [x] **Task 9: Build and type-check verification** (AC: 28)
  - [x] 9.1 Run `pnpm build` from `/home/debian/Projects/eusolicit/eusolicit-app/frontend/` — both apps exit 0 ✅
  - [x] 9.2 Run `pnpm type-check` from `/home/debian/Projects/eusolicit/eusolicit-app/frontend/` — zero TypeScript errors ✅

- **Review Follow-ups (AI)**
  - [x] [AI-Review][HIGH] F4: Fixed status filter "all" bug in `RegulationTracker.tsx` — changed `status: filterStatus !== "all" ? filterStatus : undefined` to `status: filterStatus` so the API correctly receives `"all"` and returns all records (not just `status=new`)
  - [x] [AI-Review][MEDIUM] F5: Added `useEffect` + `useEffect` import in `SuggestionQueue.tsx` to sync `selectedIds` with visible suggestions when `minConfidence` or `data` changes — prevents batch-accepting hidden (below-threshold) suggestions
  - [x] [AI-Review][MEDIUM] F6: Extended `addError` clear effect in `FrameworkAssignmentPage.tsx` to depend on both `selectedFrameworkId` and `selectedOpportunityId` — prevents stale error messages persisting across opportunity switches

---

## Dev Notes

### Working Directory

All frontend code lives under:
`eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

Run all `pnpm` commands from: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`

---

### Critical Carry-Over Learnings from S11.14 (MUST APPLY)

1. **`@/*` alias maps to `apps/admin/`** — Inside `apps/admin`, `@/lib/queries/use-compliance-frameworks` resolves to `apps/admin/lib/queries/use-compliance-frameworks.ts`. Do NOT use `@/` inside `packages/ui` — relative imports only there.
2. **Route structure** — `apps/admin/app/[locale]/(protected)/compliance/assign/page.tsx` maps to URL `/{locale}/compliance/assign`. The `(protected)` route group adds no URL segment. Sub-route directories (`assign/`, `suggestions/`, `regulations/`) are created under the existing `compliance/` directory.
3. **`"use client"` on all interactive components** — `useState`, `useMutation`, `useQuery`, event handlers → requires `"use client"`. Page shells (page.tsx) are server components.
4. **`AuthGuard` already applied in layout.tsx** — Individual page components do NOT add `<AuthGuard>` again.
5. **`apiClient` from `@eusolicit/ui`** — Import: `import { apiClient } from "@eusolicit/ui"`. Comment out in stub phase. Admin API endpoints require `platform_admin` JWT — stub bypasses this.
6. **`useZodForm` and `z` from `@eusolicit/ui`** — NOT needed for S11.15 (no zod-validated forms). Use plain React state (`useState`) for form inputs on these pages.
7. **`useToast` from `@eusolicit/ui`** — Import: `import { useToast } from "@eusolicit/ui"`. Destructure `const { toast } = useToast()`.
8. **UI components from `@eusolicit/ui`** — `Button`, `Input`, `Textarea`, `Select`, `SelectItem`, `Badge`, `Dialog` from `"@eusolicit/ui"`. All verified available from S11.14.
9. **`EmptyState`, `SkeletonTable`, `SkeletonCard` from `@eusolicit/ui`** — Use `<EmptyState icon={...} title={...} description={...} action={{ label: ..., onClick: ... }} />`. `action` prop is optional (omit when no CTA needed).
10. **`useTranslations` from `next-intl`** — Import: `import { useTranslations } from "next-intl"`. New pages use `useTranslations("compliance")`, `useTranslations("common")`, `useTranslations("errors")`.
11. **`useRouter`, `useParams`, `usePathname` from `next/navigation`** — `const { locale } = useParams<{ locale: string }>()`. `usePathname()` needed in ComplianceSubNav.
12. **`lucide-react` icons** — New icons needed for S11.15: `AlertTriangle`, `Info` (severity indicators in Regulation Tracker). Existing: `ShieldCheck`, `Loader2`, `CheckCircle2`. All available in `lucide-react ^0.400.0`.
13. **`useQueryClient` from `@tanstack/react-query`** — `import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"`.
14. **Locale prefix in navigation** — `router.push(\`/${locale}/compliance/assign\`)`. Links use `<Link href={\`/${locale}/compliance/...\`}>`.
15. **Regulation type badge colours** — Established pattern (from S11.14 AC6, verified in S11.15 AC6): `national` → `bg-blue-100 text-blue-800`, `eu` → `bg-indigo-100 text-indigo-800`, `programme` → `bg-green-100 text-green-800`. Use the existing `compliance.type.*` i18n keys.
16. **Remove/dismiss 409 error pattern** — Check `error.response?.status === 409` to detect framework-already-assigned errors (same as S11.14's delete 409 guard).
17. **`useComplianceFrameworks` REUSE** — Both `FrameworkAssignmentPage` and `SuggestionQueue` need the active frameworks list for dropdowns. Import `useComplianceFrameworks` from `@/lib/queries/use-compliance-frameworks` (created in S11.14) and call `useComplianceFrameworks({ is_active: true })`. **Do not create a duplicate hook.**
18. **Debounce pattern** — Use the same `useEffect` debounce pattern from S11.14 (300ms timeout, cleanup function) for the opportunity search input in `FrameworkAssignmentPage`.
19. **`Promise.allSettled` for batch accept** — Do NOT use `Promise.all` (fails fast on first error). Use `Promise.allSettled` to process all selected suggestions regardless of individual failures.
20. **Modifying FrameworkList.tsx** — This file was fully reviewed and approved in S11.14 with 0 open issues. The only change for S11.15 is inserting `<ComplianceSubNav />` as the first child of `<div data-testid="compliance-framework-list-page">`. Touch nothing else in this file.

---

### Architecture: File Structure Added in S11.15

```
eusolicit-app/frontend/
  apps/admin/
    lib/
      api/
        framework-assignments.ts              ← CREATED
        framework-suggestions.ts              ← CREATED
        regulatory-changes.ts                 ← CREATED
        compliance-frameworks.ts              ← UNCHANGED (S11.14)
      queries/
        use-framework-assignments.ts          ← CREATED
        use-framework-suggestions.ts          ← CREATED
        use-regulatory-changes.ts             ← CREATED
        use-compliance-frameworks.ts          ← UNCHANGED (S11.14) — REUSED
    app/
      [locale]/
        (protected)/
          compliance/
            page.tsx                          ← UNCHANGED (S11.14)
            components/
              ComplianceSubNav.tsx            ← CREATED
              FrameworkList.tsx               ← MODIFIED (add ComplianceSubNav only)
              FrameworkEditor.tsx             ← UNCHANGED (S11.14)
            assign/                           ← NEW DIRECTORY
              page.tsx                        ← CREATED
              components/                     ← NEW DIRECTORY
                FrameworkAssignmentPage.tsx   ← CREATED
            suggestions/                      ← NEW DIRECTORY
              page.tsx                        ← CREATED
              components/                     ← NEW DIRECTORY
                SuggestionQueue.tsx           ← CREATED
            regulations/                      ← NEW DIRECTORY
              page.tsx                        ← CREATED
              components/                     ← NEW DIRECTORY
                RegulationTracker.tsx         ← CREATED
    messages/
      en.json                                 ← MODIFIED (add assign/suggestions/regulations/nav keys)
      bg.json                                 ← MODIFIED (add assign/suggestions/regulations/nav keys)
  packages/
    (no changes to packages/ui for this story)
```

---

### TypeScript Interface Definitions

#### `apps/admin/lib/api/framework-assignments.ts`

```typescript
export interface OpportunityItem {
  id: string;
  name: string;
  source: string;  // e.g. "TED", "ZOP", "EU", "INTERREG"
  country: string; // e.g. "BG", "EU"
}

export interface ListOpportunitiesResponse {
  items: OpportunityItem[];
  total: number;
  page: number;
  page_size: number;
}

export interface FrameworkAssignment {
  id: string;
  opportunity_id: string;
  framework_id: string;
  framework_name: string;
  framework_country: string;
  regulation_type: "national" | "eu" | "programme";
  assigned_by: string;
  assigned_at: string; // ISO 8601
}

export interface AssignFrameworkRequest {
  framework_id: string;
}
```

#### `apps/admin/lib/api/framework-suggestions.ts`

```typescript
export interface FrameworkSuggestion {
  id: string;
  opportunity_id: string;
  opportunity_name: string;
  framework_id: string;
  framework_name: string;
  confidence: number; // 0–100
  status: "pending" | "accepted" | "rejected";
  created_at: string;
}

export interface ListSuggestionsParams {
  status?: "pending" | "accepted" | "rejected";
  date_from?: string;
  date_to?: string;
}

export interface UpdateSuggestionRequest {
  status: "accepted" | "rejected";
  override_framework_id?: string; // If set, overrides the suggested framework_id on accept
}

export type UpdateSuggestionResponse = FrameworkSuggestion;
```

#### `apps/admin/lib/api/regulatory-changes.ts`

```typescript
export interface AffectedFramework {
  id: string;
  name: string;
}

export interface RegulatoryChange {
  id: string;
  source: string;          // e.g. "EUR-Lex", "ZOP Portal", "Horizon Europe Portal"
  change_type: "new" | "amended" | "repealed";
  summary: string;
  severity: "low" | "medium" | "high";
  status: "new" | "acknowledged" | "dismissed";
  detected_at: string;     // ISO 8601
  affected_frameworks: AffectedFramework[];
  acknowledged_by?: string | null;
  acknowledged_at?: string | null;
  notes?: string | null;
}

export interface ListRegulatoryChangesParams {
  status?: string;
  severity?: string;
  date_from?: string;
  date_to?: string;
}

export interface UpdateRegulatoryChangeRequest {
  status: "acknowledged" | "dismissed";
  notes?: string;
}
```

---

### Stub Fixture Data

#### `STUB_OPPORTUNITIES_LIST` (for `framework-assignments.ts`)

```typescript
const STUB_OPPORTUNITIES_LIST: ListOpportunitiesResponse = {
  items: [
    { id: "opp-001", name: "IT System Integration for National Registry", source: "TED", country: "BG" },
    { id: "opp-002", name: "Digital Transformation of Public Services", source: "ZOP", country: "BG" },
    { id: "opp-003", name: "Horizon Europe Research Infrastructure Grant", source: "EU", country: "EU" },
    { id: "opp-004", name: "INTERREG CBC Bulgaria–Romania Project", source: "INTERREG", country: "BG" },
  ],
  total: 4,
  page: 1,
  page_size: 20,
};
```

#### `STUB_ASSIGNMENTS` (for `framework-assignments.ts`)

```typescript
// Keyed by opportunity_id for stub filtering
const STUB_ASSIGNMENTS: FrameworkAssignment[] = [
  {
    id: "assign-001",
    opportunity_id: "opp-001",
    framework_id: "fw-001",
    framework_name: "ZOP — Bulgarian Public Procurement Law 2024",
    framework_country: "BG",
    regulation_type: "national",
    assigned_by: "admin-001",
    assigned_at: "2026-04-01T10:00:00Z",
  },
  {
    id: "assign-002",
    opportunity_id: "opp-001",
    framework_id: "fw-002",
    framework_name: "Horizon Europe — General Conditions",
    framework_country: "EU",
    regulation_type: "eu",
    assigned_by: "admin-001",
    assigned_at: "2026-04-01T10:05:00Z",
  },
  {
    id: "assign-003",
    opportunity_id: "opp-003",
    framework_id: "fw-002",
    framework_name: "Horizon Europe — General Conditions",
    framework_country: "EU",
    regulation_type: "eu",
    assigned_by: "admin-002",
    assigned_at: "2026-04-02T09:30:00Z",
  },
];

// In getOpportunityFrameworks stub: return STUB_ASSIGNMENTS.filter(a => a.opportunity_id === opportunityId)
```

#### `STUB_SUGGESTIONS` (for `framework-suggestions.ts`)

```typescript
const STUB_SUGGESTIONS: FrameworkSuggestion[] = [
  {
    id: "sug-001",
    opportunity_id: "opp-002",
    opportunity_name: "Digital Transformation of Public Services",
    framework_id: "fw-001",
    framework_name: "ZOP — Bulgarian Public Procurement Law 2024",
    confidence: 94,
    status: "pending",
    created_at: "2026-04-08T14:00:00Z",
  },
  {
    id: "sug-002",
    opportunity_id: "opp-002",
    opportunity_name: "Digital Transformation of Public Services",
    framework_id: "fw-003",
    framework_name: "Digital Europe Programme — SME Requirements",
    confidence: 71,
    status: "pending",
    created_at: "2026-04-08T14:01:00Z",
  },
  {
    id: "sug-003",
    opportunity_id: "opp-003",
    opportunity_name: "Horizon Europe Research Infrastructure Grant",
    framework_id: "fw-002",
    framework_name: "Horizon Europe — General Conditions",
    confidence: 88,
    status: "pending",
    created_at: "2026-04-09T09:00:00Z",
  },
  {
    id: "sug-004",
    opportunity_id: "opp-004",
    opportunity_name: "INTERREG CBC Bulgaria–Romania Project",
    framework_id: "fw-004",
    framework_name: "INTERREG Bulgaria-Romania — Cross-border Rules",
    confidence: 42,
    status: "pending",
    created_at: "2026-04-09T09:30:00Z",
  },
];
```

#### `STUB_REGULATORY_CHANGES` (for `regulatory-changes.ts`)

```typescript
const STUB_REGULATORY_CHANGES: RegulatoryChange[] = [
  {
    id: "rc-001",
    source: "EUR-Lex",
    change_type: "amended",
    summary:
      "EU Directive 2014/24/EU Article 18 updated — new transparency requirements for above-threshold procurements effective 2026-06-01. Contracting authorities must publish award criteria rationale.",
    severity: "high",
    status: "new",
    detected_at: "2026-04-07T08:00:00Z",
    affected_frameworks: [
      { id: "fw-001", name: "ZOP — Bulgarian Public Procurement Law 2024" },
      { id: "fw-002", name: "Horizon Europe — General Conditions" },
    ],
  },
  {
    id: "rc-002",
    source: "ZOP Portal",
    change_type: "new",
    summary:
      "Statutory instrument ZOP-2026-003 introduces mandatory AI usage disclosure for IT procurement contracts above BGN 50,000.",
    severity: "medium",
    status: "new",
    detected_at: "2026-04-06T10:30:00Z",
    affected_frameworks: [{ id: "fw-001", name: "ZOP — Bulgarian Public Procurement Law 2024" }],
  },
  {
    id: "rc-003",
    source: "Horizon Europe Portal",
    change_type: "new",
    summary:
      "New open science mandate for Horizon Europe grants: all research outputs must be freely accessible under CC-BY licence from 2027.",
    severity: "low",
    status: "acknowledged",
    detected_at: "2026-04-03T14:00:00Z",
    affected_frameworks: [{ id: "fw-002", name: "Horizon Europe — General Conditions" }],
    acknowledged_by: "admin-001",
    acknowledged_at: "2026-04-05T09:00:00Z",
    notes: "Reviewed — framework guidelines updated to reference open science requirements.",
  },
];

// In listRegulatoryChanges stub: filter by params.status if provided (default: "new" if status absent)
// i.e. if (!params?.status || params.status === "new") return changes with status === "new"
// if (params?.status === "all" || params?.status === undefined) return all
// if (params?.status === "acknowledged") return acknowledged only
// etc.
// Actual behaviour: pass params.status directly as filter, return matching items
```

---

### i18n Values — English (`en.json`)

Add under **existing `"compliance"` namespace** (merge; do not overwrite S11.14 keys):

```json
{
  "nav": {
    "frameworks": "Frameworks",
    "assign": "Assign",
    "suggestions": "Suggestions",
    "regulations": "Regulations"
  },
  "assign": {
    "pageTitle": "Framework Assignment",
    "pageDescription": "Assign compliance frameworks to procurement opportunities.",
    "opportunitySelector": "Opportunities",
    "opportunitySearch": "Search opportunities...",
    "noOpportunities": "No matching opportunities.",
    "assignedFrameworks": "Assigned Frameworks",
    "selectOpportunity": "Select an opportunity",
    "selectOpportunityHint": "Choose an opportunity from the list to view and manage its assigned compliance frameworks.",
    "noAssignedFrameworks": "No frameworks assigned to this opportunity yet.",
    "removeBtn": "Remove",
    "addBtn": "Assign",
    "frameworkPlaceholder": "Select a framework...",
    "alreadyAssigned": "This framework is already assigned to this opportunity."
  },
  "suggestions": {
    "pageTitle": "Suggestion Queue",
    "pageDescription": "Review and process AI-generated framework suggestions for incoming opportunities.",
    "colOpportunity": "Opportunity",
    "colFramework": "Suggested Framework",
    "colConfidence": "Confidence",
    "colActions": "Actions",
    "filterConfidencePlaceholder": "Min confidence %",
    "acceptBtn": "Accept",
    "dismissBtn": "Dismiss",
    "overrideBtn": "Override",
    "overrideConfirmBtn": "Confirm",
    "batchAcceptBtn": "Accept Selected",
    "batchAcceptSuccess": "Selected suggestions accepted and frameworks assigned.",
    "acceptSuccess": "Suggestion accepted and framework assigned.",
    "dismissSuccess": "Suggestion dismissed.",
    "emptyTitle": "No pending suggestions",
    "emptyDescription": "All framework suggestions have been processed. New suggestions appear automatically when opportunities are ingested."
  },
  "regulations": {
    "pageTitle": "Regulation Tracker",
    "pageDescription": "Monitor and acknowledge regulatory changes that may affect compliance frameworks.",
    "filterStatus": "Status",
    "filterSeverity": "Severity",
    "filterDateFrom": "From",
    "filterDateTo": "To",
    "statusAll": "All statuses",
    "acknowledgeBtn": "Acknowledge",
    "dismissBtn": "Dismiss",
    "acknowledgeNotesPlaceholder": "Add notes (optional)...",
    "acknowledgeConfirmBtn": "Confirm Acknowledgement",
    "acknowledgeSuccess": "Regulatory change acknowledged.",
    "acknowledgedBy": "Acknowledged by {by} on {at}",
    "dismissSuccess": "Regulatory change dismissed.",
    "dismissedLabel": "Dismissed",
    "affectedFrameworks": "Affected frameworks",
    "emptyTitle": "No regulatory changes",
    "emptyDescription": "No regulatory changes match the current filters. The Regulation Tracker runs weekly and surfaces changes automatically.",
    "changeType": {
      "new": "New",
      "amended": "Amended",
      "repealed": "Repealed"
    },
    "severity": {
      "high": "High",
      "medium": "Medium",
      "low": "Low"
    },
    "status": {
      "new": "New",
      "acknowledged": "Acknowledged",
      "dismissed": "Dismissed"
    }
  }
}
```

**Note on merge:** The `"nav"` sub-object under `"compliance"` is new (S11.14 added `"compliance"` to the top-level `"nav"` object, not inside the `"compliance"` namespace). The new `compliance.nav.*` keys live **inside the `"compliance"` namespace** (i.e. `en.json["compliance"]["nav"]`), accessed via `useTranslations("compliance")` → `t("nav.frameworks")`.

---

### i18n Values — Bulgarian (`bg.json`)

Add under **existing `"compliance"` namespace** (merge):

```json
{
  "nav": {
    "frameworks": "Рамки",
    "assign": "Присвояване",
    "suggestions": "Предложения",
    "regulations": "Регулации"
  },
  "assign": {
    "pageTitle": "Присвояване на рамки",
    "pageDescription": "Присвоявайте рамки за съответствие към обществени поръчки.",
    "opportunitySelector": "Обществени поръчки",
    "opportunitySearch": "Търсене на поръчки...",
    "noOpportunities": "Няма съответстващи поръчки.",
    "assignedFrameworks": "Присвоени рамки",
    "selectOpportunity": "Изберете поръчка",
    "selectOpportunityHint": "Изберете поръчка от списъка, за да прегледате и управлявате присвоените рамки за съответствие.",
    "noAssignedFrameworks": "Към тази поръчка все още не са присвоени рамки.",
    "removeBtn": "Премахни",
    "addBtn": "Присвои",
    "frameworkPlaceholder": "Изберете рамка...",
    "alreadyAssigned": "Тази рамка вече е присвоена към тази поръчка."
  },
  "suggestions": {
    "pageTitle": "Опашка с предложения",
    "pageDescription": "Преглеждайте и обработвайте предложения за рамки, генерирани от изкуствен интелект.",
    "colOpportunity": "Поръчка",
    "colFramework": "Предложена рамка",
    "colConfidence": "Увереност",
    "colActions": "Действия",
    "filterConfidencePlaceholder": "Мин. увереност %",
    "acceptBtn": "Приеми",
    "dismissBtn": "Отхвърли",
    "overrideBtn": "Замести",
    "overrideConfirmBtn": "Потвърди",
    "batchAcceptBtn": "Приеми избраните",
    "batchAcceptSuccess": "Избраните предложения са приети и рамките са присвоени.",
    "acceptSuccess": "Предложението е прието и рамката е присвоена.",
    "dismissSuccess": "Предложението е отхвърлено.",
    "emptyTitle": "Няма чакащи предложения",
    "emptyDescription": "Всички предложения за рамки са обработени. Нови предложения ще се появят автоматично при приемане на нови поръчки."
  },
  "regulations": {
    "pageTitle": "Тракер за регулации",
    "pageDescription": "Наблюдавайте и потвърждавайте регулаторни промени, засягащи рамките за съответствие.",
    "filterStatus": "Статус",
    "filterSeverity": "Сериозност",
    "filterDateFrom": "От",
    "filterDateTo": "До",
    "statusAll": "Всички статуси",
    "acknowledgeBtn": "Потвърди",
    "dismissBtn": "Пренебрегни",
    "acknowledgeNotesPlaceholder": "Добавете бележки (по желание)...",
    "acknowledgeConfirmBtn": "Потвърди прегледа",
    "acknowledgeSuccess": "Регулаторната промяна е потвърдена.",
    "acknowledgedBy": "Потвърдено от {by} на {at}",
    "dismissSuccess": "Регулаторната промяна е пренебрегната.",
    "dismissedLabel": "Пренебрегнато",
    "affectedFrameworks": "Засегнати рамки",
    "emptyTitle": "Няма регулаторни промени",
    "emptyDescription": "Няма регулаторни промени, отговарящи на текущите филтри. Тракерът работи седмично и открива промени автоматично.",
    "changeType": {
      "new": "Нова",
      "amended": "Изменена",
      "repealed": "Отменена"
    },
    "severity": {
      "high": "Висока",
      "medium": "Средна",
      "low": "Ниска"
    },
    "status": {
      "new": "Нова",
      "acknowledged": "Потвърдена",
      "dismissed": "Пренебрегната"
    }
  }
}
```

---

### Test Design Expectations (from `test-design-epic-11.md`)

The following E11 test IDs map to this story's components. Dev agent must ensure `data-testid` attributes are correct for these tests to pass:

| Test ID | Description | Key data-testid attributes |
|---------|-------------|---------------------------|
| **E11-P2-019** | Framework Assignment Page: add 2 frameworks to opportunity; remove 1; list shows remaining | `compliance-assignment-page`, `opportunity-item-{id}`, `assigned-frameworks-table`, `assignment-row-{frameworkId}`, `add-framework-btn`, `remove-framework-btn-{frameworkId}` |
| **E11-P2-020** | Auto-Suggestion Queue: batch accept; confidence bar colour-coded; filter by confidence threshold | `compliance-suggestion-queue-page`, `suggestions-filter-confidence`, `suggestions-batch-accept-btn`, `suggestion-row-{id}`, `confidence-bar-{id}`, `suggestion-accept-btn-{id}` |
| **E11-P3-007** | Regulation Tracker: feed renders change cards; acknowledge + dismiss buttons functional; acknowledged change links to framework | `regulation-tracker-feed`, `regulation-card-{id}`, `acknowledge-btn-{id}`, `dismiss-btn-{id}`, `acknowledge-confirm-btn-{id}`, `affected-framework-chip-{changeId}-{fwId}` |
| **E11-P3-005** | E2E Journey 5: regulation tracker fires → admin views change → acknowledges → reviews affected framework | Full journey: filter `status=new`, click `acknowledge-btn`, fill notes, click `acknowledge-confirm-btn`, click `affected-framework-chip` to navigate to editor |

**Confidence bar colour thresholds** (aligned with E11 test design):
- `confidence >= 80` → `bg-green-500` (high confidence, green)
- `60 <= confidence < 80` → `bg-amber-500` (medium confidence, amber)
- `confidence < 60` → `bg-red-500` (low confidence, red)

**Note on S11.09 backend APIs:** All three pages wire to admin-only endpoints (`/api/v1/admin/...`). The stub phase uses in-memory data. When S11.09 and S11.10 backends are integrated, un-comment `apiClient` imports and remove stub delays. Admin JWT is enforced by the backend; the frontend passes the stored JWT automatically via `apiClient`'s request interceptor.

### Project Structure Notes

- This story operates **exclusively within `apps/admin/`**. The `apps/client/` frontend is not touched.
- All new components follow the established `apps/admin/app/[locale]/(protected)/` route pattern.
- `packages/ui` requires no changes — all needed primitives (`Button`, `Input`, `Textarea`, `Select`, `Badge`, `EmptyState`, `SkeletonTable`, `SkeletonCard`, `Loader2`) were verified available in S11.14.
- The `compliance/` directory and the `compliance/components/` subdirectory already exist from S11.14. `ComplianceSubNav.tsx` is added alongside `FrameworkList.tsx` and `FrameworkEditor.tsx` in that directory.

### References

- [Source: eusolicit-docs/planning-artifacts/epic-11-grants-compliance.md#S11.15] — Story requirements (three admin sub-pages)
- [Source: eusolicit-docs/planning-artifacts/epic-11-grants-compliance.md#S11.09] — Backend APIs for assignment and suggestions
- [Source: eusolicit-docs/planning-artifacts/epic-11-grants-compliance.md#S11.10] — Backend APIs for regulation tracker
- [Source: eusolicit-docs/test-artifacts/test-design-epic-11.md#P2-P3] — E11-P2-019, E11-P2-020, E11-P3-005, E11-P3-007 test expectations
- [Source: eusolicit-docs/implementation-artifacts/11-14-compliance-admin-framework-management-frontend.md#Dev Notes] — Critical carry-over learnings, established patterns, `ComplianceFramework` type definitions (reused), `useComplianceFrameworks` hook (reused)
- [Source: eusolicit-docs/project-context.md#Frontend Architecture] — Rules 19–31 (component structure, `QueryGuard`, `useZodForm`, auth guard, Zustand, apiClient patterns)

---

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5)

### Debug Log References

- Fixed: `useToast` API in this project returns `{ success, error, warning, info }` methods directly, NOT a `{ toast }` destructure. Updated all three client components accordingly.
- Fixed: ESLint `no-unused-vars` on `params` argument in `listOpportunities` stub — renamed to `_params` per project convention.

### Completion Notes List

- **Task 1** — Created 3 API client modules with TypeScript interfaces, stub fixture data, and real `apiClient` calls commented out pending S11.09/S11.10 backend wiring. Stub `listOpportunities` ignores params (server-side pagination out of scope for stub). `assignFrameworkToOpportunity` throws a synthetic 409 error if the framework is already in the stub list for that opportunity.
- **Task 2** — Created 3 TanStack Query hook files. `useAssignFramework` and `useRemoveFramework` accept `opportunityId` in the mutation vars for targeted `invalidateQueries` calls.
- **Task 3** — Added 53 new keys across `nav.*`, `assign.*`, `suggestions.*`, `regulations.*` sub-namespaces inside the existing `"compliance"` namespace. `pnpm check:i18n` confirmed 196 keys matched in both locales.
- **Task 4** — Created `ComplianceSubNav` using the AC2-specified path-splitting algorithm. Active state uses `pathParts[2]` (index 2 = segment after locale + "compliance") to detect sub-route. Frameworks tab is active for all non-sub-route paths (empty string, UUIDs, "new").
- **Task 5** — Inserted `<ComplianceSubNav />` as first child of `<div data-testid="compliance-framework-list-page">` with a single import addition. No other changes to FrameworkList.tsx.
- **Task 6** — `FrameworkAssignmentPage` implements debounced opportunity search (300ms), per-row skeleton/loading/error states, 409 duplicate detection, and add-framework + remove-framework flows per AC4–AC7.
- **Task 7** — `SuggestionQueue` implements confidence bar with colour thresholds (≥80 green, ≥60 amber, <60 red), client-side confidence filter, select-all toggle, override picker per-row inline, per-row `processingIds` Set, and `Promise.allSettled` batch accept per AC8–AC14.
- **Task 8** — `RegulationTracker` implements status/severity/date filters, card feed with change_type badge, severity icon (AlertTriangle/Info), affected framework chips as `<Link>` to editor, acknowledge panel with textarea, `dismissingIds` Set for per-row dismiss in-flight state, and shared `updateChangeMutation` per AC15–AC20.
- **Task 9** — `pnpm build` and `pnpm type-check` both exit 0 across admin and client apps. All 8 new/modified routes compile cleanly.
- ✅ Resolved review finding [HIGH] F4: Fixed "all" status filter pass-through in `RegulationTracker.tsx` — `filterParams.status` now passes `filterStatus` directly; API stub correctly handles `"all"` returning all records.
- ✅ Resolved review finding [MEDIUM] F5: Added `useEffect` sync in `SuggestionQueue.tsx` (also added `useEffect` import) — `selectedIds` is intersected with visible suggestion IDs when `minConfidence` or `data` changes; batch accept only operates on visible selections.
- ✅ Resolved review finding [MEDIUM] F6: Extended `addError` clear effect in `FrameworkAssignmentPage.tsx` to depend on `[selectedFrameworkId, selectedOpportunityId]` — stale error messages are cleared on opportunity switch.

### File List

**Created:**
- `eusolicit-app/frontend/apps/admin/lib/api/framework-assignments.ts`
- `eusolicit-app/frontend/apps/admin/lib/api/framework-suggestions.ts`
- `eusolicit-app/frontend/apps/admin/lib/api/regulatory-changes.ts`
- `eusolicit-app/frontend/apps/admin/lib/queries/use-framework-assignments.ts`
- `eusolicit-app/frontend/apps/admin/lib/queries/use-framework-suggestions.ts`
- `eusolicit-app/frontend/apps/admin/lib/queries/use-regulatory-changes.ts`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/components/ComplianceSubNav.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/assign/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/assign/components/FrameworkAssignmentPage.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/suggestions/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/suggestions/components/SuggestionQueue.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/regulations/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/regulations/components/RegulationTracker.tsx`

**Modified:**
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/compliance/components/FrameworkList.tsx` — added `ComplianceSubNav` import + render
- `eusolicit-app/frontend/apps/admin/messages/en.json` — added `compliance.nav.*`, `compliance.assign.*`, `compliance.suggestions.*`, `compliance.regulations.*` keys
- `eusolicit-app/frontend/apps/admin/messages/bg.json` — added same keys in Bulgarian

### Change Log

- **2026-04-10** — Implemented Story 11.15: Framework Assignment page, Auto-Suggestion Queue page, Regulation Tracker Dashboard page; shared ComplianceSubNav component; 3 API client modules; 3 React Query hook files; 53 new i18n keys (en + bg). `pnpm build` and `pnpm type-check` exit 0.
- **2026-04-10** — Resolved 3 review findings (pass 1): (1) Fixed severity filter `SelectItem value=""` bug in `RegulationTracker.tsx` — changed to `"__all__"` sentinel per Radix UI constraint; (2) Replaced hardcoded `"Assigned by"` / `"Assigned at"` table headers in `FrameworkAssignmentPage.tsx` with `t("assign.colAssignedBy")` / `t("assign.colAssignedAt")` + added keys to both `en.json` and `bg.json`; (3) Replaced hardcoded partial batch error template literal in `SuggestionQueue.tsx` with `t("suggestions.batchPartialError", { fulfilled, rejected })` + added key to both message files. `pnpm build` and `pnpm type-check` both exit 0.
- **2026-04-10** — Resolved 3 review findings (pass 2): (F4) Fixed `RegulationTracker.tsx` status filter to pass `filterStatus` directly to API — "all" now correctly returns all records; (F5) Added `useEffect` sync in `SuggestionQueue.tsx` to intersect `selectedIds` with visible suggestions on confidence filter/data change — batch accept now operates only on visible selections; (F6) Extended `addError` clear effect in `FrameworkAssignmentPage.tsx` to also clear on `selectedOpportunityId` change — prevents stale error messages across opportunity switches. `pnpm build` and `pnpm type-check` both exit 0.

---

## Senior Developer Review

**Reviewer:** Claude Opus 4.6 (code-review agent)
**Date:** 2026-04-10 (pass 3 — final)
**Verdict:** ✅ APPROVED — All acceptance criteria met. All prior findings resolved. No blocking issues.

**Review scope:** All 13 created files + 3 modified files. Adversarial review with Blind Hunter, Edge Case Hunter, and Acceptance Auditor layers. Prior reviews (pass 1: 3 findings, pass 2: 3 findings) — all 6 resolved and verified in code.

### Acceptance Criteria Audit — All 28 ACs Passed

| AC Range | Area | Status |
|----------|------|--------|
| AC1 | Route structure — 6 new files | ✅ All files created, page shells are server components, client components have `"use client"` |
| AC2–AC3 | ComplianceSubNav | ✅ Correct location, 4 tabs, active-tab detection, integrated into all 4 pages |
| AC4–AC7 | Framework Assignment Page | ✅ Two-column grid, debounced search, opportunity selection, assigned frameworks table, add/remove, 409 handling |
| AC8–AC14 | Auto-Suggestion Queue | ✅ Filters, confidence bar colour-coding, select-all, override picker, per-row processing, `Promise.allSettled` batch |
| AC15–AC20 | Regulation Tracker Dashboard | ✅ Status/severity/date filters, card feed, severity icons, affected framework chips, acknowledge panel, dismiss flow |
| AC21–AC23 | API client modules | ✅ Interfaces match spec, stub delays, 409 simulation, mutable stub state |
| AC24–AC26 | React Query hooks | ✅ Correct query keys, stale times, mutation invalidation |
| AC27 | i18n | ✅ 123 compliance keys in both en.json and bg.json — parity verified |
| AC28 | Build & type-check | ✅ `pnpm build` and `pnpm type-check` exit 0 |

### Architecture Alignment

- **S11.14 hook reuse:** `useComplianceFrameworks` imported from existing `@/lib/queries/use-compliance-frameworks` in both `FrameworkAssignmentPage` and `SuggestionQueue` — no duplication ✅
- **Server/client boundary:** All 3 `page.tsx` shells are server components; all client components have `"use client"` ✅
- **`useToast` API:** Verified correct — `@eusolicit/ui` returns `{ success, error, warning, info }` methods directly ✅
- **Import paths:** `@/lib/...` alias used correctly within `apps/admin/` scope ✅
- **UI components:** All from `@eusolicit/ui` barrel export — no app-local duplicates ✅

### Prior Findings — All 6 Resolved ✅

| Pass | # | Finding | Status |
|------|---|---------|--------|
| 1 | F1 | Severity filter `SelectItem value=""` bug | ✅ Fixed — `"__all__"` sentinel |
| 1 | F2 | Hardcoded "Assigned by"/"Assigned at" headers | ✅ Fixed — i18n keys added |
| 1 | F3 | Hardcoded batch error template literal | ✅ Fixed — i18n key added |
| 2 | F4 | Status filter "all" broken in RegulationTracker | ✅ Fixed — passes `filterStatus` directly |
| 2 | F5 | `selectedIds` not synced with visible suggestions | ✅ Fixed — `useEffect` intersects with `visibleIds` |
| 2 | F6 | `addError` persists across opportunity switch | ✅ Fixed — effect depends on `[selectedFrameworkId, selectedOpportunityId]` |

### Pass 3 — New Findings: None blocking

No HIGH or MEDIUM issues found. Code quality is clean and consistent across all 3 new pages.

### Noted but not blocking

| # | Observation | Severity | Rationale for not blocking |
|---|-------------|----------|---------------------------|
| N1 | `useAssignFramework` hook and `handleAddFramework` both invalidate `["opportunity-frameworks", ...]` on success — double refetch | Low | Redundant network call, no functional harm. Can clean up post-merge. |
| N2 | `removingFrameworkId` is a scalar `useState<string \| null>` — rapid multi-row removes would lose loading state on the first row | Low | Spec explicitly defines it as scalar (AC6). Matches spec intent; UX edge case only on rapid sequential clicks. |
| N3 | Shared `updateChangeMutation` in RegulationTracker: `isPending` cross-contaminates acknowledge + dismiss buttons | Low | Spec explicitly requires this design (AC19 note). The spec acknowledges the trade-off and mitigates dismiss via `dismissingIds` Set. |
| N4 | `selectedIds` in SuggestionQueue retains IDs of individually accepted/dismissed suggestions until data refetch clears them from `visibleSuggestions` | Low | Batch button count may briefly show +1. Functionally harmless — batch accept on a stale ID triggers a server error handled by `Promise.allSettled`. |
| N5 | Module-level mutable `let` for stub data (`STUB_ASSIGNMENTS`, etc.) — would leak state across SSR requests or shared test module caches | Info | Intentional stub design, marked for replacement when backend wires up. Client-only execution path. |
| N6 | `QueryGuard` not used — project-context rule 21 overridden by spec which mandates ad-hoc skeleton patterns | Low | Spec-level design choice. All three components handle loading/empty states per AC. |
| N7 | Query error states (`isError`) not destructured from data-fetching hooks | Low | Spec omits query-level error handling. Mutation errors handled per AC. Spec gap, not implementation defect. |
| N8 | `getTypeBadgeClass()` duplicated in `FrameworkList.tsx` and `FrameworkAssignmentPage.tsx` | Low | Exact same function. Extract to shared utility post-merge. |
| N9 | No dedicated unit tests for S11.15 components | Info | AC28 requires build/type-check only. Test design (E11-P2-019, E11-P2-020, E11-P3-007) covers E2E; those tests are written in a separate test story. |
