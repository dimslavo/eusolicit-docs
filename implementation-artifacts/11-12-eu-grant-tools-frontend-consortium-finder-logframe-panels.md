# Story 11.12: EU Grant Tools Frontend — Consortium Finder & Logframe Panels

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **company user on EU Solicit**,
I want Consortium Finder, Logframe, and Reporting Template panels added to the existing Grant Tools page,
so that I can identify suitable consortium partners, generate a complete logical framework with Gantt chart, and produce a pre-filled downloadable report template — all from one integrated grant-preparation workspace.

## Acceptance Criteria

### Page Tabs Extension

1. **AC1** — The existing Grant Tools page at `apps/client/app/[locale]/(protected)/grants/tools/page.tsx` is extended with three new `<TabsTrigger>` / `<TabsContent>` entries appended after the existing `"budget"` tab:
   - `value="consortium"` — label `t("grantTools.consortium.tabLabel")`; content: `<ConsortiumFinderPanel />`
   - `value="logframe"` — label `t("grantTools.logframe.tabLabel")`; content: `<LogframePanel />`
   - `value="reporting"` — label `t("grantTools.reporting.tabLabel")`; content: `<ReportingTemplatePanel />`

   The two existing tabs (`"eligibility"`, `"budget"`) are **not modified**. `data-testid="grant-tools-page"` on the root element is unchanged.

---

### Consortium Finder Panel

2. **AC2** — `ConsortiumFinderPanel` component (`apps/client/app/[locale]/(protected)/grants/tools/components/ConsortiumFinderPanel.tsx`) renders an input form with these fields:
   - **Project description** — `<Textarea>` (`data-testid="consortium-project-description"`, `name="project_description"`, required, min 10 chars, label `t("grantTools.consortium.projectDescription")`)
   - **Required capabilities** — tag input (`data-testid="consortium-capabilities-input"`): a plain `<input type="text">` where pressing `Enter` or `,` adds the current value as a capability chip to a `useState<string[]>` array. Capability chips render below the input as `<span>` elements each with `data-testid="consortium-capability-tag"` and an `×` remove button (`data-testid="consortium-capability-remove-{index}"`). At least one capability is required (validated on submit).
   - **Target countries** — multi-select implemented as a collapsible dropdown (`data-testid="consortium-countries-dropdown"`): clicking a button (`data-testid="consortium-countries-toggle"`) opens a scrollable checkbox list. Each country row has a `<Checkbox>` from `@eusolicit/ui`. Selected countries are tracked in `useState<string[]>`. A badge on the toggle button shows `{n} selected` when n > 0 (`data-testid="consortium-countries-badge"`). Countries list: `["Austria", "Belgium", "Bulgaria", "Croatia", "Cyprus", "Czech Republic", "Denmark", "Estonia", "Finland", "France", "Germany", "Greece", "Hungary", "Ireland", "Italy", "Latvia", "Lithuania", "Luxembourg", "Malta", "Netherlands", "Poland", "Portugal", "Romania", "Slovakia", "Slovenia", "Spain", "Sweden"]`. A search input inside the dropdown (`data-testid="consortium-countries-search"`) filters the visible list.
   - **Max results** — `<Input type="number" min={1} max={20} defaultValue={5}>` (`data-testid="consortium-max-results"`, `name="max_results"`, label `t("grantTools.consortium.maxResults")`)
   - **Submit button** — `<Button type="submit" data-testid="consortium-submit-btn">` with label `t("grantTools.consortium.submitBtn")`

   Form state for `project_description` and `max_results` is managed via `useZodForm` with `consortiumFinderSchema` (see Dev Notes for schema definition). Capabilities and countries are managed as separate `useState` arrays, passed manually to the mutation payload on submit.

3. **AC3** — While the consortium mutation is in-flight (`mutation.isPending`), the submit button shows `<Loader2 className="animate-spin" />` and is disabled (`data-testid="consortium-loading-state"` on the spinner). All form fields are disabled during the in-flight period.

4. **AC4** — On mutation success with `partners.length > 0`, results render as a CSS grid (`data-testid="consortium-results-grid"`, `className="grid grid-cols-1 md:grid-cols-2 gap-4"`). Each partner renders as a `<PartnerCard>` component (`data-testid="consortium-partner-card"`) containing:
   - **Organisation name** (`data-testid="partner-name"`) in `font-semibold text-slate-900`
   - **Country** (`data-testid="partner-country"`) as `{country_flag_emoji} {country}` text in `text-sm text-slate-600`. Map country names to Unicode flag emoji using a lookup object (see Dev Notes for the EU country → emoji map).
   - **Organisation type** — `<Badge>` (`data-testid="partner-org-type"`) using `className="bg-blue-100 text-blue-800 text-xs"`
   - **Matched capabilities** — rendered as a row of `<Badge>` chips (`data-testid="partner-capability-chip-{index}"`, `className="bg-indigo-100 text-indigo-700 text-xs"`) — only capabilities that appear in the search request's `required_capabilities` list are marked with `ring-2 ring-indigo-400` (highlighted match). Gracefully renders as empty section when `relevant_capabilities` is missing or null.
   - **Collaboration score bar** — a horizontal progress bar (`data-testid="partner-score-bar"`) implemented as `<div className="w-full bg-slate-200 rounded-full h-2"><div className="bg-indigo-600 h-2 rounded-full" style={{ width: \`${collaboration_score}%\` }} /></div>` with score value label `data-testid="partner-score-value"`. Score range 0–100.
   - **Past projects** — collapsible section (`data-testid="partner-past-projects-toggle"`, initially collapsed) showing a `<ul>` of project name strings. Renders "No past projects" text if `past_projects` is null or empty (graceful degradation per E11-R-012). Toggle managed via `useState<boolean>`.
   - **Contact info** — rendered as `data-testid="partner-contact-info"` with contact string. Renders nothing (no empty placeholder) when `contact_info` is null or absent (graceful degradation).
   - **Shortlist toggle** — `<Button variant="outline" size="sm" data-testid="partner-shortlist-btn">`: label `t("grantTools.consortium.addToShortlist")` when not in shortlist; `t("grantTools.consortium.removeFromShortlist")` when in shortlist. Shortlist state managed in `ConsortiumFinderPanel` as `useState<Set<string>>` keyed by `organisation_name`. When n > 0 partners are shortlisted, a banner `data-testid="consortium-shortlist-banner"` shows `t("grantTools.consortium.shortlistCount", { count: n })`.

5. **AC5** — On mutation success with `partners.length === 0`, `<EmptyState>` from `@eusolicit/ui` renders (`data-testid="consortium-empty-state"`) with `title={t("grantTools.consortium.emptyTitle")}` and `description={t("grantTools.consortium.emptyDescription")}`.

6. **AC6** — On mutation failure, an inline error block (`data-testid="consortium-error-state"`) renders with `t("errors.serverError")` and a retry button (`data-testid="consortium-retry-btn"`) that resets and re-invokes the mutation.

---

### Logframe Panel

7. **AC7** — `LogframePanel` component (`apps/client/app/[locale]/(protected)/grants/tools/components/LogframePanel.tsx`) renders an input form with:
   - **Project narrative** — `<Textarea>` (`data-testid="logframe-project-narrative"`, `name="project_narrative"`, required, min 20 chars, label `t("grantTools.logframe.narrative")`)
   - **Target programme** — `<Select>` (`data-testid="logframe-target-programme"`, `name="target_programme"`, required) with options: `Horizon Europe`, `Digital Europe`, `INTERREG`, `Structural Funds`, `CAP`.
   - **Submit button** — `<Button type="submit" data-testid="logframe-submit-btn">` with label `t("grantTools.logframe.submitBtn")`

   Form managed via `useZodForm` with `logframeSchema` (see Dev Notes).

8. **AC8** — While the logframe mutation is in-flight, submit button shows spinner (`data-testid="logframe-loading-state"`) and is disabled; form fields are disabled.

9. **AC9** — On mutation success, four result sections render inside `data-testid="logframe-result-panel"`:

   **A — Logical Framework Table** (`data-testid="logframe-framework-table"`):  
   A `<table>` with four columns: `t("grantTools.logframe.objective")`, `t("grantTools.logframe.activity")`, `t("grantTools.logframe.indicator")`, `t("grantTools.logframe.verification")`. Each row maps to one item in `logical_framework` array. Rows use `data-testid="logframe-framework-row-{index}"`. When `logical_framework` is null or empty, renders a `<p className="text-slate-500 text-sm">` with placeholder text.

   **B — Work Packages** (`data-testid="logframe-work-packages"`):  
   Expandable cards, one per item in `work_packages`. Each card header (`data-testid="logframe-wp-card-{index}"`) shows `WP{wp_number}: {title}` and a `<ChevronDown>/<ChevronUp>` toggle. Expanded body shows: lead (`data-testid="logframe-wp-lead"`), description, deliverables list, person-months badge. Expansion managed via `useState<Set<number>>`. When `work_packages` is null or empty, renders a "No work packages" text.

   **C — Gantt Chart** (`data-testid="logframe-gantt-chart"`):  
   Horizontal bar chart using Recharts (`BarChart` with `layout="vertical"`). Data is derived from `gantt_data` array — each task becomes a bar entry. The X-axis (`XAxis type="number"`) represents months (`domain={[0, maxMonth]}`). Each bar renders as two `<Bar>` segments: an invisible spacer bar (transparent, representing `start_month` offset) and a visible bar (representing duration `end_month - start_month`, `fill="#4f46e5"`). The chart is wrapped in `<ResponsiveContainer width="100%" height={gantt_data.length * 40 + 60}>`. Task names (`name`) appear on the Y-axis (`YAxis dataKey="name" type="category" width={120}`). **When `gantt_data` is null or absent in the API response, the entire Gantt section is replaced with a `<p data-testid="logframe-gantt-unavailable" className="text-slate-500 text-sm">` message — no error thrown (graceful degradation per E11-R-008/E11-P1-005).**

   **D — Deliverable Table** (`data-testid="logframe-deliverable-table"`):  
   A `<table>` with sortable columns: `#`, `t("grantTools.logframe.deliverableTitle")`, `WP`, `t("grantTools.logframe.deliverableType")`, `t("grantTools.logframe.disseminationLevel")`, `t("grantTools.logframe.dueMonth")`. Sort state managed via `useState<{ column: keyof DeliverableRow; direction: "asc" | "desc" }>` (default: `{ column: "deliverable_number", direction: "asc" }`). Clicking any column header button toggles direction and sets that column. Sorted rows are computed using `[...deliverable_table].sort(...)`. Column header buttons have `data-testid="logframe-deliverable-sort-{column}"`. Each row has `data-testid="logframe-deliverable-row-{index}"`.

10. **AC10** — On logframe mutation failure, inline error block (`data-testid="logframe-error-state"`) renders with `t("errors.serverError")` and a retry button.

---

### Reporting Template Panel

11. **AC11** — `ReportingTemplatePanel` component (`apps/client/app/[locale]/(protected)/grants/tools/components/ReportingTemplatePanel.tsx`) renders:
    - **Project selector** — `<Select>` (`data-testid="reporting-project-select"`, label `t("grantTools.reporting.selectProject")`). Options are populated from a stub projects list (see Dev Notes for `STUB_REPORTING_PROJECTS`). The selected `project_id` is tracked via `useState<string>("")`.
    - **Generate button** — `<Button data-testid="reporting-generate-btn">` with label `t("grantTools.reporting.submitBtn")`. Disabled when no project is selected.

    When no project is selected and no result exists, `<EmptyState>` renders (`data-testid="reporting-empty-state"`) with `title={t("grantTools.reporting.emptyTitle")}` and `description={t("grantTools.reporting.emptyDescription")}`.

12. **AC12** — While the reporting mutation is in-flight, the generate button shows a `<Loader2 className="animate-spin" />` spinner (`data-testid="reporting-loading-state"`) and is disabled.

13. **AC13** — On mutation success, the result panel (`data-testid="reporting-result-panel"`) renders:
    - **Project name** and **reporting period** displayed in a card header (`data-testid="reporting-header"`).
    - **Sections list**: each section in `sections` array renders as a `<Card>` (`data-testid="reporting-section-{index}"`) with:
      - `<CardHeader>` showing `section_title`.
      - `<CardContent>`: when `is_editable` is `true`, a `<Textarea>` pre-filled with `content` (`data-testid="reporting-section-textarea-{index}"`), managed via `useState<string[]>` tracking editable contents. When `is_editable` is `false`, a `<p>` with the content (read-only).
    - **Download DOCX button** — `<Button data-testid="reporting-download-btn">` with label `t("grantTools.reporting.downloadDocx")`. While the export mutation is in-flight, button shows `t("grantTools.reporting.downloadingDocx")` and is disabled (`data-testid="reporting-download-loading"`). On click, calls `exportReportingTemplate(project_id)` and triggers a browser file download by creating a temporary `<a>` element with `URL.createObjectURL(blob)` and programmatically clicking it. The downloaded filename is `reporting-template.docx`. During stub phase, the export function returns a stub Blob (see Dev Notes).

14. **AC14** — On reporting mutation failure, inline error block (`data-testid="reporting-error-state"`) renders with `t("errors.serverError")` and a retry button.

---

### API Client Extensions

15. **AC15** — `apps/client/lib/api/grants.ts` is **extended** (not replaced) with new TypeScript interfaces and functions:

    **New interfaces:**
    - `ConsortiumFinderParams`, `Partner`, `ConsortiumFinderResult` (see Dev Notes for full definitions)
    - `LogframeGenerateParams`, `LogicalFrameworkRow`, `WorkPackage`, `GanttTask`, `DeliverableRow`, `LogframeResult` (see Dev Notes)
    - `ReportingTemplateParams`, `ReportingSection`, `ReportingTemplateResult` (see Dev Notes)

    **New API functions:**
    - `findConsortium(params: ConsortiumFinderParams): Promise<ConsortiumFinderResult>` — calls `apiClient.post<ConsortiumFinderResult>("/api/v1/grants/consortium-finder", params)` then returns `response.data`. During stub phase, returns `STUB_CONSORTIUM` after 800 ms delay.
    - `generateLogframe(params: LogframeGenerateParams): Promise<LogframeResult>` — calls `apiClient.post<LogframeResult>("/api/v1/grants/logframe-generate", params)` then returns `response.data`. During stub phase, returns `STUB_LOGFRAME` after 800 ms.
    - `generateReportingTemplate(params: ReportingTemplateParams): Promise<ReportingTemplateResult>` — calls `apiClient.post<ReportingTemplateResult>("/api/v1/grants/reporting-template", params)` then returns `response.data`. During stub phase, returns `STUB_REPORTING_TEMPLATE` after 800 ms.
    - `exportReportingTemplate(project_id: string): Promise<Blob>` — calls `apiClient.post("/api/v1/grants/reporting-template/export", { project_id, format: "docx" }, { responseType: "blob" })` and returns the blob. During stub phase, returns a stub Blob of type `application/vnd.openxmlformats-officedocument.wordprocessingml.document` immediately (see Dev Notes).

---

### React Query Hooks Extension

16. **AC16** — `apps/client/lib/queries/use-grant-tools.ts` is **extended** with three new exports:
    - `useConsortiumFinder()` — `useMutation({ mutationFn: findConsortium, mutationKey: ["consortiumFinder"] })`
    - `useLogframeGenerator()` — `useMutation({ mutationFn: generateLogframe, mutationKey: ["logframeGenerator"] })`
    - `useReportingTemplate()` — `useMutation({ mutationFn: generateReportingTemplate, mutationKey: ["reportingTemplate"] })`
    - `useExportReportingTemplate()` — `useMutation({ mutationFn: (project_id: string) => exportReportingTemplate(project_id), mutationKey: ["exportReportingTemplate"] })`

---

### i18n

17. **AC17** — New i18n keys are added under the `"grantTools"` namespace in both `apps/client/messages/en.json` and `apps/client/messages/bg.json`. Required keys (Bulgarian values in Dev Notes):

    ```
    grantTools.consortium.tabLabel
    grantTools.consortium.projectDescription
    grantTools.consortium.capabilities
    grantTools.consortium.capabilitiesPlaceholder
    grantTools.consortium.countries
    grantTools.consortium.maxResults
    grantTools.consortium.submitBtn
    grantTools.consortium.emptyTitle
    grantTools.consortium.emptyDescription
    grantTools.consortium.addToShortlist
    grantTools.consortium.removeFromShortlist
    grantTools.consortium.shortlistCount
    grantTools.consortium.score
    grantTools.consortium.pastProjects
    grantTools.consortium.contact
    grantTools.logframe.tabLabel
    grantTools.logframe.narrative
    grantTools.logframe.targetProgramme
    grantTools.logframe.submitBtn
    grantTools.logframe.frameworkTitle
    grantTools.logframe.objective
    grantTools.logframe.activity
    grantTools.logframe.indicator
    grantTools.logframe.verification
    grantTools.logframe.workPackages
    grantTools.logframe.ganttTitle
    grantTools.logframe.ganttUnavailable
    grantTools.logframe.deliverables
    grantTools.logframe.deliverableTitle
    grantTools.logframe.deliverableType
    grantTools.logframe.disseminationLevel
    grantTools.logframe.dueMonth
    grantTools.reporting.tabLabel
    grantTools.reporting.selectProject
    grantTools.reporting.submitBtn
    grantTools.reporting.emptyTitle
    grantTools.reporting.emptyDescription
    grantTools.reporting.downloadDocx
    grantTools.reporting.downloadingDocx
    ```

    `pnpm check:i18n --filter client` must exit 0 after additions.

---

### Build & Type-Check

18. **AC18** — `pnpm build` exits 0 for both `client` and `admin` apps. `pnpm type-check` exits 0 across all packages. No TypeScript errors in any new or modified file.

---

## Tasks / Subtasks

- [x] Task 1: Extend grants API client (AC: 15)
  - [x] 1.1 Add new TypeScript interfaces to `apps/client/lib/api/grants.ts`: `ConsortiumFinderParams`, `Partner`, `ConsortiumFinderResult`, `LogframeGenerateParams`, `LogicalFrameworkRow`, `WorkPackage`, `GanttTask`, `DeliverableRow`, `LogframeResult`, `ReportingTemplateParams`, `ReportingSection`, `ReportingTemplateResult`
  - [x] 1.2 Add stub fixtures: `STUB_CONSORTIUM`, `STUB_LOGFRAME`, `STUB_REPORTING_TEMPLATE` (see Dev Notes for fixture data)
  - [x] 1.3 Implement `findConsortium()` stub (800 ms + STUB_CONSORTIUM)
  - [x] 1.4 Implement `generateLogframe()` stub (800 ms + STUB_LOGFRAME)
  - [x] 1.5 Implement `generateReportingTemplate()` stub (800 ms + STUB_REPORTING_TEMPLATE)
  - [x] 1.6 Implement `exportReportingTemplate()` stub (returns stub Blob immediately)
  - [x] 1.7 Wire real API calls with `apiClient.post()` behind stubs (commented out)

- [x] Task 2: Extend React Query hooks (AC: 16)
  - [x] 2.1 Add `useConsortiumFinder()`, `useLogframeGenerator()`, `useReportingTemplate()`, `useExportReportingTemplate()` to `apps/client/lib/queries/use-grant-tools.ts`

- [x] Task 3: Add i18n translation keys (AC: 17)
  - [x] 3.1 Add all 38 new keys to `apps/client/messages/en.json` under `grantTools.consortium`, `grantTools.logframe`, `grantTools.reporting`
  - [x] 3.2 Add all 38 new keys to `apps/client/messages/bg.json` with Bulgarian translations (from Dev Notes)
  - [x] 3.3 Run `pnpm check:i18n --filter client` — must exit 0

- [x] Task 4: Build Consortium Finder Panel (AC: 2, 3, 4, 5, 6)
  - [x] 4.1 Create `apps/client/app/[locale]/(protected)/grants/tools/components/ConsortiumFinderPanel.tsx` — `"use client"` directive
  - [x] 4.2 Implement project description textarea + max results input using `useZodForm` with `consortiumFinderSchema`
  - [x] 4.3 Implement tag input for capabilities — keyboard event handler on `Enter`/`,` to add chip, remove on `×` click
  - [x] 4.4 Implement countries multi-select dropdown — collapsible panel with checkbox list + search filter; selected count badge
  - [x] 4.5 Implement `PartnerCard` sub-component (same file or colocated) — all fields including flag emoji, capabilities highlighting, score bar, past projects toggle, contact info, shortlist button
  - [x] 4.6 Implement shortlist state (`useState<Set<string>>`) in panel; banner showing shortlisted count
  - [x] 4.7 Implement loading, empty, and error states

- [x] Task 5: Build Logframe Panel (AC: 7, 8, 9, 10)
  - [x] 5.1 Create `apps/client/app/[locale]/(protected)/grants/tools/components/LogframePanel.tsx` — `"use client"` directive
  - [x] 5.2 Implement narrative textarea + programme select using `useZodForm` with `logframeSchema`
  - [x] 5.3 Implement logical framework table (4 columns, one row per `logical_framework` item)
  - [x] 5.4 Implement work packages expandable cards using `useState<Set<number>>`
  - [x] 5.5 Implement Gantt chart using Recharts `BarChart layout="vertical"` with spacer + duration bars; graceful null/absent handling for `gantt_data`
  - [x] 5.6 Implement deliverable table with sortable columns using `useState<{ column, direction }>` and client-side sort
  - [x] 5.7 Implement loading, empty, and error states

- [x] Task 6: Build Reporting Template Panel (AC: 11, 12, 13, 14)
  - [x] 6.1 Create `apps/client/app/[locale]/(protected)/grants/tools/components/ReportingTemplatePanel.tsx` — `"use client"` directive
  - [x] 6.2 Implement project selector using stub projects list; disabled generate button until project selected
  - [x] 6.3 Implement result panel: header (project name + period), editable/read-only sections via `useState<string[]>` for editable content
  - [x] 6.4 Implement "Download DOCX" button: trigger `useExportReportingTemplate()` mutation; on success, create temporary `<a>` element with `URL.createObjectURL(blob)` and click to trigger download; filename `reporting-template.docx`; revoke URL after click
  - [x] 6.5 Implement loading, empty, and error states

- [x] Task 7: Extend Grant Tools page with new tabs (AC: 1)
  - [x] 7.1 Import `ConsortiumFinderPanel`, `LogframePanel`, `ReportingTemplatePanel` in `page.tsx`
  - [x] 7.2 Add three new `<TabsTrigger>` and `<TabsContent>` entries to the existing `<Tabs>` component

- [x] Task 8: Build and type-check verification (AC: 18)
  - [x] 8.1 Run `pnpm build` from `/home/debian/Projects/eusolicit/eusolicit-app/frontend/` — both apps must exit 0
  - [x] 8.2 Run `pnpm type-check` from `/home/debian/Projects/eusolicit/eusolicit-app/frontend/` — zero TypeScript errors

---

## Dev Notes

### Working Directory

All frontend code lives under:  
`eusolicit-app/frontend/` (absolute: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`)

Run all `pnpm` commands from: `/home/debian/Projects/eusolicit/eusolicit-app/frontend/`

---

### Critical Carry-Over Learnings (MUST APPLY)

1. **`@/*` alias maps to `apps/client/`** — Inside `apps/client`, `@/lib/api/grants` resolves to `apps/client/lib/api/grants.ts`. Use `@/` imports throughout client components.
2. **`"use client"` on all interactive components** — Any component using `useState`, `useMutation`, or event handlers requires `"use client"` at the top.
3. **`recharts` is already installed** — `recharts: ^2.12.0` is in `apps/client/package.json`. No reinstall needed. Import from `"recharts"`: `import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, Cell } from "recharts"`.
4. **No `Accordion` in `@eusolicit/ui`** — Expandable work packages and past projects toggles must use `useState<Set<number>>` / `useState<boolean>` with conditional rendering.
5. **No `TagInput` or `MultiSelect` in `@eusolicit/ui`** — Tag input for capabilities and country multi-select must be built manually. See AC2 and Tasks 4.3/4.4.
6. **No `Slider` in `@eusolicit/ui`** — Not used in this story (overhead rate slider was in S11.11).
7. **`Checkbox` IS available** — `import { Checkbox } from "@eusolicit/ui"` — available in `packages/ui/src/components/ui/checkbox.tsx`.
8. **`Card`, `CardContent`, `CardHeader`, `CardTitle` from `@eusolicit/ui`** — Use for partner cards and reporting section cards.
9. **`EmptyState` from `@eusolicit/ui`** — Available for empty results states.
10. **`apiClient` from `@eusolicit/ui`** — Import: `import { apiClient } from "@eusolicit/ui"` (commented out during stub phase).
11. **`useZodForm` and `z` from `@eusolicit/ui`** — Import `z` from `@eusolicit/ui` for schema definitions.
12. **`grants.ts` already exists** — This story **extends** the existing file; do NOT overwrite. Append new interfaces, fixtures, and functions at the end of the file.
13. **`use-grant-tools.ts` already exists** — This story **extends** the existing file. Add new hooks at the bottom.
14. **`page.tsx` already exists** — This story **modifies** the existing page to add three tabs. The file currently imports only `EligibilityPanel` and `BudgetBuilderPanel`; add three more imports.
15. **Gantt chart spacer pattern** — Recharts has no native Gantt support. Use two stacked `<Bar>` per entry: the first bar's value = `start_month` with `fill="transparent"` and `stackId="gantt"`, the second bar's value = `end_month - start_month` with `fill="#4f46e5"` and `stackId="gantt"`. The data object per task: `{ name: task.title, offset: task.start_month, duration: task.end_month - task.start_month }`.
16. **DOCX download pattern** — `exportReportingTemplate()` returns a `Blob`. On `useExportReportingTemplate()` mutation success, trigger download in `onSuccess` callback:
    ```typescript
    const url = URL.createObjectURL(data);
    const a = document.createElement("a");
    a.href = url;
    a.download = "reporting-template.docx";
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
    ```
17. **Country emoji map** — Define a `const EU_COUNTRY_FLAGS: Record<string, string>` inside `ConsortiumFinderPanel.tsx`. Key sample entries: `{ "Austria": "🇦🇹", "Belgium": "🇧🇪", "Bulgaria": "🇧🇬", "France": "🇫🇷", "Germany": "🇩🇪", "Italy": "🇮🇹", "Netherlands": "🇳🇱", "Poland": "🇵🇱", "Romania": "🇷🇴", "Spain": "🇪🇸", "Sweden": "🇸🇪" }` — fill all 27 EU members.
18. **`select` placeholder in `<Select>`** — Use `<SelectItem value="" disabled>` as first item with placeholder text.

---

### Architecture: File Structure Added/Modified in S11.12

```
eusolicit-app/frontend/
  apps/client/
    lib/
      api/
        grants.ts                                              ← MODIFIED (new interfaces + functions appended)
      queries/
        use-grant-tools.ts                                     ← MODIFIED (new hooks appended)
    app/
      [locale]/
        (protected)/
          grants/
            tools/
              page.tsx                                         ← MODIFIED (3 new tabs added)
              components/
                EligibilityPanel.tsx                           ← UNCHANGED
                BudgetBuilderPanel.tsx                         ← UNCHANGED
                ConsortiumFinderPanel.tsx                      ← CREATED
                LogframePanel.tsx                              ← CREATED
                ReportingTemplatePanel.tsx                     ← CREATED
    messages/
      en.json                                                  ← MODIFIED (consortium, logframe, reporting keys)
      bg.json                                                  ← MODIFIED (consortium, logframe, reporting keys)
```

---

### New TypeScript Interface Definitions

Add to end of `apps/client/lib/api/grants.ts`:

```typescript
// ─── Consortium Finder ─────────────────────────────────────────────────────

export interface ConsortiumFinderParams {
  project_description: string;
  required_capabilities: string[];
  target_countries?: string[];
  max_results: number;
}

export interface Partner {
  organisation_name: string;
  country: string;
  organisation_type: string;
  relevant_capabilities: string[] | null;
  past_projects: string[] | null;
  collaboration_score: number;         // 0-100
  contact_info: string | null;
}

export interface ConsortiumFinderResult {
  partners: Partner[];
  total_count: number;
}

// ─── Logframe Generator ────────────────────────────────────────────────────

export interface LogframeGenerateParams {
  project_narrative: string;
  target_programme: string;
}

export interface LogicalFrameworkRow {
  objective: string;
  activity: string;
  indicator: string;
  means_of_verification: string;
}

export interface WorkPackage {
  wp_number: number;
  title: string;
  lead: string;
  description: string;
  deliverables: string[];
  person_months: number;
}

export interface GanttTask {
  task_id: string;
  title: string;
  start_month: number;
  end_month: number;
  dependencies: string[];
}

export interface DeliverableRow {
  deliverable_number: string;
  title: string;
  wp: string;
  type: string;
  dissemination_level: string;
  due_month: number;
}

export interface LogframeResult {
  logical_framework: LogicalFrameworkRow[];
  work_packages: WorkPackage[];
  gantt_data: GanttTask[] | null;
  deliverable_table: DeliverableRow[];
}

// ─── Reporting Template Generator ─────────────────────────────────────────

export interface ReportingTemplateParams {
  project_id: string;
}

export interface ReportingSection {
  section_title: string;
  content: string;
  is_editable: boolean;
}

export interface ReportingTemplateResult {
  project_name: string;
  reporting_period: string;
  sections: ReportingSection[];
}
```

---

### Zod Schemas for New Forms

```typescript
// consortiumFinderSchema — for project_description and max_results fields
// Place inside ConsortiumFinderPanel.tsx
import { z } from "@eusolicit/ui";

const consortiumFinderSchema = z.object({
  project_description: z.string().min(10, "Minimum 10 characters required"),
  max_results: z.coerce.number().int().min(1).max(20),
});
type ConsortiumFinderFormData = z.infer<typeof consortiumFinderSchema>;

// logframeSchema — for project_narrative and target_programme
// Place inside LogframePanel.tsx
const TARGET_PROGRAMMES = ["Horizon Europe", "Digital Europe", "INTERREG", "Structural Funds", "CAP"] as const;

const logframeSchema = z.object({
  project_narrative: z.string().min(20, "Minimum 20 characters required"),
  target_programme: z.enum(TARGET_PROGRAMMES, { errorMap: () => ({ message: "Select a programme" }) }),
});
type LogframeFormData = z.infer<typeof logframeSchema>;
```

---

### Stub Fixture Data

Add to `apps/client/lib/api/grants.ts` after existing `STUB_BUDGET`:

```typescript
const STUB_CONSORTIUM: ConsortiumFinderResult = {
  total_count: 3,
  partners: [
    {
      organisation_name: "Fraunhofer Institute for Digital Medicine",
      country: "Germany",
      organisation_type: "Research Institute",
      relevant_capabilities: ["AI/ML", "Healthcare Data", "Clinical Trials"],
      past_projects: [
        "HORIZON-HEALTH-2023-AI",
        "Digital Europe — Health Data Space",
        "Interreg — MED Health Innovation",
      ],
      collaboration_score: 91,
      contact_info: "partnership@fraunhofer-idm.de",
    },
    {
      organisation_name: "Sofia Tech Park EAD",
      country: "Bulgaria",
      organisation_type: "Innovation Hub",
      relevant_capabilities: ["Deep Tech", "AI/ML", "SME Ecosystem"],
      past_projects: ["Horizon Europe — EIC Accelerator", "INTERREG CENTRAL EUROPE"],
      collaboration_score: 74,
      contact_info: "international@sofiatechpark.eu",
    },
    {
      organisation_name: "Centre for Advanced Studies Barcelona",
      country: "Spain",
      organisation_type: "University Research Centre",
      relevant_capabilities: ["Healthcare Data", "Biomedical Engineering"],
      past_projects: null,
      collaboration_score: 62,
      contact_info: null,
    },
  ],
};

const STUB_LOGFRAME: LogframeResult = {
  logical_framework: [
    {
      objective: "Develop AI-driven early diagnosis system",
      activity: "Design ML model architecture",
      indicator: "Model accuracy ≥ 90% on validation dataset",
      means_of_verification: "Quarterly technical review report",
    },
    {
      objective: "Validate system in clinical environment",
      activity: "Pilot deployment in 3 hospitals",
      indicator: "500 patient records processed; clinician satisfaction ≥ 80%",
      means_of_verification: "Hospital sign-off report; survey results",
    },
    {
      objective: "Disseminate results and open-source components",
      activity: "Publish 3 peer-reviewed papers; GitHub release",
      indicator: "Papers accepted; 100+ GitHub stars within 6 months",
      means_of_verification: "Journal DOI links; GitHub analytics",
    },
  ],
  work_packages: [
    {
      wp_number: 1,
      title: "Project Management & Coordination",
      lead: "Sofia Tech Park EAD",
      description: "Coordination of consortium activities, reporting, and quality assurance.",
      deliverables: ["D1.1 Project Management Plan", "D1.2 Periodic Progress Reports"],
      person_months: 12,
    },
    {
      wp_number: 2,
      title: "AI Model Development",
      lead: "Fraunhofer Institute for Digital Medicine",
      description: "Design, training, and validation of the core ML diagnostic model.",
      deliverables: ["D2.1 Model Architecture Specification", "D2.2 Trained Model v1.0", "D2.3 Validation Report"],
      person_months: 36,
    },
    {
      wp_number: 3,
      title: "Clinical Pilot",
      lead: "Centre for Advanced Studies Barcelona",
      description: "Pilot deployment and clinical validation across three hospital partners.",
      deliverables: ["D3.1 Pilot Protocol", "D3.2 Clinical Validation Report"],
      person_months: 18,
    },
  ],
  gantt_data: [
    { task_id: "T1", title: "WP1: Management", start_month: 1, end_month: 24, dependencies: [] },
    { task_id: "T2", title: "WP2: Model Development", start_month: 1, end_month: 18, dependencies: [] },
    { task_id: "T3", title: "WP2: Model Validation", start_month: 16, end_month: 22, dependencies: ["T2"] },
    { task_id: "T4", title: "WP3: Clinical Pilot", start_month: 18, end_month: 24, dependencies: ["T3"] },
  ],
  deliverable_table: [
    { deliverable_number: "D1.1", title: "Project Management Plan", wp: "WP1", type: "Report", dissemination_level: "Public", due_month: 2 },
    { deliverable_number: "D1.2", title: "Periodic Progress Reports (x3)", wp: "WP1", type: "Report", dissemination_level: "Confidential", due_month: 24 },
    { deliverable_number: "D2.1", title: "Model Architecture Specification", wp: "WP2", type: "Technical", dissemination_level: "Public", due_month: 6 },
    { deliverable_number: "D2.2", title: "Trained Model v1.0", wp: "WP2", type: "Software", dissemination_level: "Confidential", due_month: 18 },
    { deliverable_number: "D2.3", title: "Validation Report", wp: "WP2", type: "Report", dissemination_level: "Public", due_month: 22 },
    { deliverable_number: "D3.1", title: "Pilot Protocol", wp: "WP3", type: "Report", dissemination_level: "Confidential", due_month: 19 },
    { deliverable_number: "D3.2", title: "Clinical Validation Report", wp: "WP3", type: "Report", dissemination_level: "Public", due_month: 24 },
  ],
};

const STUB_REPORTING_TEMPLATE: ReportingTemplateResult = {
  project_name: "AI-Driven Early Diagnosis — Horizon Europe Grant",
  reporting_period: "Month 1 – Month 12 (Periodic Report 1)",
  sections: [
    {
      section_title: "Executive Summary",
      content: "This periodic report covers the first 12 months of project execution. The consortium successfully completed all planned deliverables for WP1 and WP2, achieving the primary milestone of a validated AI model architecture.",
      is_editable: true,
    },
    {
      section_title: "Work Packages Progress",
      content: "WP1: Project Management — on schedule. WP2: AI Model Development — 85% complete. WP3: Clinical Pilot — preparation phase initiated.",
      is_editable: true,
    },
    {
      section_title: "Grant Agreement Reference",
      content: "Grant Agreement No: 101234567 | Project Acronym: AIEDIAG | Duration: 24 months",
      is_editable: false,
    },
    {
      section_title: "Financial Report Summary",
      content: "Total EU contribution: €1,200,000. Expenditure to date: €487,350 (40.6% of total). Funds on track with budget plan.",
      is_editable: true,
    },
    {
      section_title: "Deviations & Corrective Actions",
      content: "Minor delay in hospital partner onboarding (WP3). Mitigation: agreements signed by Month 14.",
      is_editable: true,
    },
  ],
};

export const STUB_REPORTING_PROJECTS = [
  { id: "proj-001", name: "AI-Driven Early Diagnosis — Horizon Europe" },
  { id: "proj-002", name: "Digital Health Infrastructure — Digital Europe" },
];
```

---

### API Functions to Add to `grants.ts`

```typescript
export async function findConsortium(
  params: ConsortiumFinderParams
): Promise<ConsortiumFinderResult> {
  // TODO: Replace with real call when S11.06 backend is deployed:
  // const response = await apiClient.post<ConsortiumFinderResult>("/api/v1/grants/consortium-finder", params);
  // return response.data;
  void params;
  await delay(STUB_DELAY_MS);
  return STUB_CONSORTIUM;
}

export async function generateLogframe(
  params: LogframeGenerateParams
): Promise<LogframeResult> {
  // TODO: Replace with real call when S11.07 backend is deployed:
  // const response = await apiClient.post<LogframeResult>("/api/v1/grants/logframe-generate", params);
  // return response.data;
  void params;
  await delay(STUB_DELAY_MS);
  return STUB_LOGFRAME;
}

export async function generateReportingTemplate(
  params: ReportingTemplateParams
): Promise<ReportingTemplateResult> {
  // TODO: Replace with real call when S11.07 backend is deployed:
  // const response = await apiClient.post<ReportingTemplateResult>("/api/v1/grants/reporting-template", params);
  // return response.data;
  void params;
  await delay(STUB_DELAY_MS);
  return STUB_REPORTING_TEMPLATE;
}

export async function exportReportingTemplate(project_id: string): Promise<Blob> {
  // TODO: Replace with real call when S11.07 backend is deployed:
  // const response = await apiClient.post("/api/v1/grants/reporting-template/export", { project_id, format: "docx" }, { responseType: "blob" });
  // return response.data as Blob;
  void project_id;
  // Stub: return an empty DOCX-type blob
  return new Blob(["stub-docx-content"], {
    type: "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  });
}
```

---

### i18n Keys — Bulgarian Translations

Add to `apps/client/messages/bg.json` under `"grantTools"`:

```json
"consortium": {
  "tabLabel": "Намиране на консорциум",
  "projectDescription": "Описание на проекта",
  "capabilities": "Необходими компетенции",
  "capabilitiesPlaceholder": "Въведете компетенция и натиснете Enter",
  "countries": "Целеви държави",
  "maxResults": "Максимален брой резултати",
  "submitBtn": "Намери партньори",
  "emptyTitle": "Няма намерени партньори",
  "emptyDescription": "Няма организации, отговарящи на зададените критерии. Разширете критериите за търсене.",
  "addToShortlist": "Добави в списъка",
  "removeFromShortlist": "Премахни от списъка",
  "shortlistCount": "{count} избрани",
  "score": "Оценка",
  "pastProjects": "Минали проекти",
  "contact": "Контакт"
},
"logframe": {
  "tabLabel": "Логическа рамка",
  "narrative": "Разказ за проекта",
  "targetProgramme": "Целева програма",
  "submitBtn": "Генерирай",
  "frameworkTitle": "Логическа рамка",
  "objective": "Цел",
  "activity": "Дейност",
  "indicator": "Индикатор",
  "verification": "Начин на проверка",
  "workPackages": "Работни пакети",
  "ganttTitle": "Диаграма на Гант",
  "ganttUnavailable": "Диаграмата на Гант не е налична за тази генерация.",
  "deliverables": "Резултати",
  "deliverableTitle": "Заглавие",
  "deliverableType": "Тип",
  "disseminationLevel": "Ниво на разпространение",
  "dueMonth": "Месец"
},
"reporting": {
  "tabLabel": "Шаблон за отчет",
  "selectProject": "Изберете спечелен проект",
  "submitBtn": "Генерирай шаблон",
  "emptyTitle": "Изберете проект",
  "emptyDescription": "Изберете проект от списъка, за да генерирате предварително попълнен шаблон за периодичен отчет.",
  "downloadDocx": "Изтегли DOCX",
  "downloadingDocx": "Изтегляне..."
}
```

---

### Test Expectations from Epic-Level Test Design

The following test IDs from `eusolicit-docs/test-artifacts/test-design-epic-11.md` are relevant to this story's frontend implementation. Dev should ensure the component structure supports these assertions:

| Test ID | What it validates | Implementation requirement |
|---------|------------------|---------------------------|
| **E11-P2-013** | Consortium Finder Panel: tag input + multi-select country render; partner card grid displays all fields | `data-testid` attributes on tag input, country dropdown, capability chips, score bar, past projects toggle, contact info section |
| **E11-P2-014** | Logframe Panel: Gantt chart renders with tasks; deliverable table is sortable | `data-testid="logframe-gantt-chart"` wrapping Recharts; `data-testid="logframe-deliverable-sort-{column}"` on sortable headers |
| **E11-P1-003** | Consortium Finder API: paginated results; capability overlap ranking; single-country filter; max_results honoured | Front-end passes `required_capabilities`, `target_countries`, `max_results` from form state to mutation params |
| **E11-P1-004** | Logframe: all 4 output fields present | Result panel renders all 4 sections; each has `data-testid` |
| **E11-P1-005** | Logframe: `gantt_data` absent → partial result, no 500 | `gantt_data` null check before rendering chart; `data-testid="logframe-gantt-unavailable"` renders instead |
| **E11-P1-007** | Reporting Template DOCX export: Content-Type `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | `exportReportingTemplate()` returns correctly typed Blob; download triggered on mutation success |
| **E11-R-012** | Consortium Finder: missing `contact_info` / `past_projects` → graceful null, not crash | `partner.contact_info && <...>` guard; `past_projects?.length ? ... : "No past projects"` |

**Security notes (backend — not this story's scope):** E11-P0-001 (ESPD RLS), E11-P0-006 (admin 403), and E11-P0-007 are backend tests. The frontend simply handles 403/404 error responses via the mutation's `onError` path, which is covered by AC6/AC10/AC14.

---

### Key Component Data Flow

```
ConsortiumFinderPanel
  useZodForm (project_description, max_results)
  useState<string[]> → capabilities (tag input)
  useState<string[]> → selectedCountries (multi-select)
  useState<Set<string>> → shortlisted (partner names)
  useConsortiumFinder() mutation
    → findConsortium({ project_description, required_capabilities: capabilities, target_countries: selectedCountries, max_results })
    → ConsortiumFinderResult { partners[], total_count }
  PartnerCard[] → renders each partner

LogframePanel
  useZodForm (project_narrative, target_programme)
  useLogframeGenerator() mutation
    → generateLogframe({ project_narrative, target_programme })
    → LogframeResult { logical_framework[], work_packages[], gantt_data: GanttTask[] | null, deliverable_table[] }
  useState<Set<number>> → openWPs (expanded work packages)
  useState<{ column, direction }> → sortState (deliverable table)

ReportingTemplatePanel
  useState<string> → selectedProjectId
  useState<string[]> → editableSectionContents (mirrors sections[].content for editable sections)
  useReportingTemplate() mutation
    → generateReportingTemplate({ project_id: selectedProjectId })
    → ReportingTemplateResult { project_name, reporting_period, sections[] }
  useExportReportingTemplate() mutation
    → exportReportingTemplate(selectedProjectId)
    → Blob → browser download
```

---

## Dev Agent Record

### Implementation Plan

All tasks executed in sequence per the story spec. The existing `grants.ts` and `use-grant-tools.ts` files were extended (not replaced). Three new panel components were created following the exact same patterns established in S11.11 (`EligibilityPanel`, `BudgetBuilderPanel`). Key implementation decisions:

1. **Tag input for capabilities** — built as a controlled `<input>` with `onKeyDown` handling for `Enter` and `,` keys; chips stored in `useState<string[]>`.
2. **Country multi-select** — built as a collapsible dropdown with `useState<boolean>` toggle, `Checkbox` from `@eusolicit/ui`, and a search filter `<input>` inside the dropdown.
3. **Gantt chart** — Recharts `BarChart layout="vertical"` with two stacked `<Bar>` per task: transparent spacer (`offset = start_month - 1`) + indigo duration bar (`duration = end_month - start_month + 1`). Null `gantt_data` gracefully renders `data-testid="logframe-gantt-unavailable"` paragraph.
4. **Deliverable sort** — client-side `[...array].sort(...)` on each render; `SortState` typed as `{ column: keyof DeliverableRow; direction: "asc" | "desc" }`.
5. **DOCX download** — `URL.createObjectURL(blob)` → temp `<a>` element → `click()` → `removeChild` → `revokeObjectURL`.
6. **ESLint fix** — removed unused `CardHeader`/`CardTitle` imports from `LogframePanel.tsx` on first build attempt.
7. **EmptyState icon** — `EmptyState` requires a mandatory `icon` prop; `SearchX` from lucide-react used for consortium and reporting empty states.
8. **i18n** — `check-i18n-keys.mjs` confirmed 149 keys in both `en.json` and `bg.json` after additions.

### Completion Notes

- **AC1** ✅ — Three new tabs (`consortium`, `logframe`, `reporting`) appended to `page.tsx`; existing tabs untouched.
- **AC2–AC6** ✅ — `ConsortiumFinderPanel` with all required `data-testid` attributes, form validation, partner cards, shortlist, empty & error states.
- **AC7–AC10** ✅ — `LogframePanel` with framework table, expandable WP cards, Gantt chart (graceful null), sortable deliverable table, error state.
- **AC11–AC14** ✅ — `ReportingTemplatePanel` with project select, editable sections, DOCX download, empty & error states.
- **AC15** ✅ — All interfaces, stubs, and API functions appended to `grants.ts`.
- **AC16** ✅ — Four new mutation hooks added to `use-grant-tools.ts`.
- **AC17** ✅ — 38 new i18n keys added to both `en.json` and `bg.json`; `check:i18n` exits 0.
- **AC18** ✅ — `pnpm build` exits 0 (both `client` and `admin`); `pnpm type-check` exits 0 across all 4 packages.

---

## File List

- `eusolicit-app/frontend/apps/client/lib/api/grants.ts` — MODIFIED (new interfaces, stub fixtures, API functions appended)
- `eusolicit-app/frontend/apps/client/lib/queries/use-grant-tools.ts` — MODIFIED (4 new mutation hooks appended)
- `eusolicit-app/frontend/apps/client/messages/en.json` — MODIFIED (38 new i18n keys under grantTools.consortium, logframe, reporting)
- `eusolicit-app/frontend/apps/client/messages/bg.json` — MODIFIED (38 new i18n keys with Bulgarian translations)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/grants/tools/page.tsx` — MODIFIED (3 new tab triggers + content added)
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/grants/tools/components/ConsortiumFinderPanel.tsx` — CREATED
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/grants/tools/components/LogframePanel.tsx` — CREATED
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/grants/tools/components/ReportingTemplatePanel.tsx` — CREATED

---

## Senior Developer Review

### Review Findings

> **Code review complete.** 0 `decision-needed`, 10 `patch`, 6 `defer`, 8 dismissed as noise.
> Review layers: Blind Hunter ✅, Edge Case Hunter ✅, Acceptance Auditor ✅. All layers returned findings.
> Findings written below.

**Patch — required fixes (AC violations and bugs):**

- [x] [Review][Patch] P1: **LogframePanel uses native `<select>` instead of `<Select>` from `@eusolicit/ui`** [LogframePanel.tsx:172] — AC7 specifies `<Select>` component from the design system. Dev Note 18 references `<SelectItem>` syntax, confirming Radix-style Select was intended. Native `<select>` is used with `register()`. Fix: switch to `<Select>` from `@eusolicit/ui` with controlled value via `form.setValue()`/`form.watch()`. *(Source: auditor — AC7 violation)*
- [x] [Review][Patch] P2: **ReportingTemplatePanel uses native `<select>` instead of `<Select>` from `@eusolicit/ui`** [ReportingTemplatePanel.tsx:83] — AC11 specifies `<Select>` component. Same root cause and fix pattern as LogframePanel. *(Source: auditor — AC11 violation)*
- [x] [Review][Patch] P3: **Retry handlers bypass form validation in all three panels** [ConsortiumFinderPanel.tsx:496, LogframePanel.tsx:224, ReportingTemplatePanel.tsx:130] — Retry buttons call `mutation.mutate()` directly using `form.getValues()` without re-running Zod validation. Consortium retry can send empty `required_capabilities: []` (bypassing the `capabilities.length === 0` guard on normal submit). Logframe retry can send `target_programme: undefined` (bypassing Zod enum validation). ReportingTemplatePanel retry silently does nothing if user clears the dropdown after error — error state is cleared by `mutation.reset()` but no request fires, leaving user stranded. Fix: re-route retry through `form.handleSubmit()` or add explicit validation guards before `mutate()` calls. *(Source: blind+edge — AC6/AC10/AC14 defect)*
- [x] [Review][Patch] P4: **Country dropdown has no click-outside handler** [ConsortiumFinderPanel.tsx:400-429] — The absolute-positioned country dropdown stays open when clicking anywhere outside it. It overlaps form elements below (capability input, submit button), potentially blocking interaction. No blur handler, no Escape key handler, no focus-trap. Fix: add `useRef` + `useEffect` with a document `mousedown` listener to close the dropdown on outside click. *(Source: blind+auditor — AC2 UX defect)*
- [x] [Review][Patch] P5: **DOCX download uses `selectedProjectId` instead of the generated result's project ID** [ReportingTemplatePanel.tsx:49] — If the user changes the project selector after generating but before downloading, `exportMutation.mutate(selectedProjectId)` exports the wrong project. The displayed template is still for the original project. Fix: track `generatedProjectId` in state, set it in `onSuccess`, and pass it to the export mutation. *(Source: blind+edge — AC13 data mismatch bug)*
- [x] [Review][Patch] P6: **No error state rendered for export mutation failure** [ReportingTemplatePanel.tsx:48-61, 190-208] — The component renders error state for the generate mutation (`mutation.isError`) but not for the export mutation (`exportMutation.isError`). If the DOCX export API call fails (network error, 500), the spinner stops but the user sees no error message, no retry option, and no indication that the download failed. Fix: add `exportMutation.isError` check and inline error block near the download button. *(Source: blind)*
- [x] [Review][Patch] P7: **Editable content state mismatch when re-generating for a different project** [ReportingTemplatePanel.tsx:27, 40-43] — `editableContents` is set in `onSuccess` but is keyed by array index against `sections`. If the user edits content for project A, then generates for project B (which may have a different number of sections), the `editableContents[index]` array can be misaligned with the new sections during the transition. Fix: reset `editableContents` to `[]` when `selectedProjectId` changes (via `useEffect`), or clear on mutation start. *(Source: blind)*
- [x] [Review][Patch] P8: **Shortlist state not cleared between searches** [ConsortiumFinderPanel.tsx:225] — `shortlisted` (Set<string>) is never reset when a new search is submitted. Ghost entries from previous results inflate the shortlist count banner and may incorrectly pre-check the shortlist button on name collisions. Fix: call `setShortlisted(new Set())` in `onSubmit` or in mutation `onSuccess`. *(Source: edge)*
- [x] [Review][Patch] P9: **Null element in `relevant_capabilities` array crashes PartnerCard** [ConsortiumFinderPanel.tsx:117-119] — The `partner.relevant_capabilities` array guard handles `null` at the array level, but individual elements inside the array are not null-checked. If the API returns `["AI", null, "ML"]`, `cap.toLowerCase()` on the null element throws `TypeError: Cannot read properties of null`. Fix: add `.filter(Boolean)` before `.map()` on `relevant_capabilities`, or null-check inside the map callback. *(Source: edge)*
- [x] [Review][Patch] P10: **Empty deliverable table renders headers with no body or message** [LogframePanel.tsx:417-519] — When `deliverable_table` is an empty array, the table renders column headers and an empty `<tbody>`. Other sections (logical framework, work packages) have graceful empty-state fallbacks but the deliverable table does not. Fix: add an empty-state row/message when `sortedDeliverables.length === 0`. *(Source: edge+auditor)*

**Deferred — pre-existing or not caused by this change:**

- [x] [Review][Defer] W1: **Tab switch destroys all component state (shortlist, editable content, sort, form inputs)** [page.tsx:40-58] — Radix `TabsContent` unmounts inactive tabs by default. Switching to another tab and back resets all `useState` in panels. This is the established pattern from S11.11 (`EligibilityPanel`, `BudgetBuilderPanel`). Fix when addressed: add `forceMount` to `TabsContent` or lift state into a context/zustand store. *(Source: blind — pre-existing architectural pattern)*
- [x] [Review][Defer] W2: **Partner `organisation_name` used as React key and shortlist identifier — not guaranteed unique** [ConsortiumFinderPanel.tsx:529] — If the real API returns two partners with the same name, React key collision causes rendering bugs and shortlist misbehaves. Defer: only relevant when stub is replaced with real API (S11.06 integration). *(Source: blind+edge — pre-existing stub limitation)*
- [x] [Review][Defer] W3: **Score bar overflows if `collaboration_score` is outside 0–100 range** [ConsortiumFinderPanel.tsx:142] — No runtime clamp on `collaboration_score`; values >100 or <0 break the progress bar layout. Defer: only relevant with real API data; add `Math.min(100, Math.max(0, score))` at integration time. *(Source: edge — pre-existing stub limitation)*
- [x] [Review][Defer] W4: **Negative Gantt bar duration when `start_month > end_month`** [LogframePanel.tsx:129] — Corrupted API data would produce a negative duration bar in Recharts. Defer: only relevant with real API; add `Math.max(1, duration)` clamp at integration time. *(Source: edge — pre-existing stub limitation)*
- [x] [Review][Defer] W5: **Country toggle button text overflow with many selections** [ConsortiumFinderPanel.tsx:385] — Selecting many countries causes the joined label to overflow the fixed-height trigger button. Defer: UX polish, not a functional bug; truncate or summarize in a future UX pass. *(Source: edge — cosmetic)*
- [x] [Review][Defer] W6: **Stub API functions return same mutable object reference on every call** [grants.ts:419-460] — All stub functions return the same `STUB_*` constant. If any consumer mutates the returned data in place, it corrupts the stub for subsequent calls. Currently safe because `[...rawDeliverables].sort(...)` copies before sorting, but fragile for future consumers. Defer: replace with `structuredClone()` at backend integration time. *(Source: blind — pre-existing pattern from S11.11)*

---

## Change Log

- 2026-04-10: Story 11-12 implemented — Consortium Finder, Logframe, and Reporting Template panels added to Grant Tools page. All 18 ACs satisfied. `pnpm build` and `pnpm type-check` exit 0.
- 2026-04-10: Code review round 1 (Blind Hunter + Edge Case Hunter + Acceptance Auditor) — 7 patch, 4 deferred. Key issues: native `<select>` violates AC7/AC11, retry bypass, country dropdown UX, shortlist state leak, DOCX export stale project ID.
- 2026-04-10: Code review round 2 (independent 3-layer re-review) — found 3 additional patch items (P6: missing export error state, P7: editable content mismatch, P9: null capability crash) and 2 additional defers (W1: tab unmount, W6: mutable stubs). Total: 10 patch, 6 deferred, 8 dismissed. Status remains `in-progress` pending fixes.
- 2026-04-10: Addressed all 10 code review patch items — P1 (LogframePanel Radix Select), P2 (ReportingTemplatePanel Radix Select), P3 (retry handlers re-routed through handleSubmit in all panels), P4 (click-outside handler for country dropdown via useRef+useEffect), P5 (generatedProjectId state to prevent stale DOCX export), P6 (export mutation error state + retry in ReportingTemplatePanel), P7 (editableContents reset via useEffect on project change), P8 (shortlist cleared in onSubmit), P9 (.filter(Boolean) on relevant_capabilities), P10 (empty-state row in deliverable table). `pnpm build` and `pnpm type-check` exit 0. Status updated to `review`.
- 2026-04-10: **Final code review — APPROVED.** Independent 3-layer re-review (Blind Hunter, Edge Case Hunter, Acceptance Auditor) verified all 10 patch fixes correctly applied. All 18 ACs satisfied. `pnpm type-check` exits 0 (4/4 packages). 6 deferred items remain documented (pre-existing patterns / integration-time). 0 new findings. Status updated to `done`.
