# Story 10.15: Task Template Manager UI

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-15-task-template-manager-ui
- **Points:** 2
- **Type:** frontend
- **Module:** Client web app (`apps/client`) — new route: `app/[locale]/(protected)/settings/task-templates/page.tsx`; new components under `app/[locale]/(protected)/settings/task-templates/components/`: `TaskTemplatesPage`, `TaskTemplatesTable`, `TaskTemplateFormModal`, `StageListEditor`, `StageRowEditor`, `StageDependencyCheckboxes`, `ApplyTemplateModal`, `ApplyTemplatePreview`, `TemplateEmptyState`, `TemplateDeleteConfirm`; new API client: `apps/client/lib/api/task-templates.ts`; new queries: `apps/client/lib/queries/use-task-templates.ts`; i18n: `apps/client/messages/{en,bg}.json` (new `taskTemplates.*` namespace); sidebar-settings nav wired to the new route; opportunity detail page (`app/[locale]/(protected)/opportunities/[id]/page.tsx`) gets an "Apply template" button + mounts `<ApplyTemplateModal>`; ATDD test: `apps/client/__tests__/task-templates-s10-15.test.ts`. Reuses existing `@dnd-kit/*` deps (pulled in by Story 10.14) — no new runtime dependencies.
- **Priority:** P2 (Epic 10 productivity layer — user-visible surface for Story 10.7's `/api/v1/task-templates*` CRUD + apply endpoints. Optional at MVP (managers can still create tasks one-by-one from Story 10.14's board), but unlocks the "reusable 8–15-step preparation checklist per bid" value prop called out in Epic 10's goal statement. Ships after Story 10.14 so it can deep-link into the kanban board after apply.)
- **Depends On:** Story 10.7 (Task Templates CRUD & Application — `POST/GET/PATCH/DELETE /api/v1/task-templates*`, `POST /api/v1/task-templates/{id}/apply` returning `{ created_task_ids, created_dependency_ids, warnings }`, `StageDefinition` shape `{title, description?, role, relative_days_before_deadline, dependency_indices[]}`, `opportunity_type` enum `tender|grant|any`, partial-unique `(company_id, name)` constraint surfacing 409 on duplicate name, `dependency_indices` strictly-less-than-own-index validation surfacing 422, soft-delete on DELETE, warnings array on apply for past-due-date stages and type-mismatches), Story 10.14 (Task Kanban Board & Detail Modal — post-apply deep-link target; also contributed `@dnd-kit/*` runtime dependencies so stage reordering does not add a new dep), Story 6.5 (Opportunity Detail page — host for the "Apply template" button and modal), Story 2.9 (Team Member Management — NOT directly used; template stages carry a `role` string, not an assignee user_id), Story 3.3 (app shell + sidebar-settings navigation), Story 3.5 (Zustand + TanStack Query), Story 3.6 (React Hook Form + Zod — form validation), Story 3.7 (next-intl EN/BG parity), Story 3.10 (loading/empty/error primitives), Story 3.11 (toast store — `useUIStore.addToast` for mutation errors and apply-success confirmations).

## Story

As **a bid manager responsible for standardising how my team prepares for recurring bid types (EU tenders, national grants, etc.)**,
I want **a company-settings page where I can list, create, edit, reorder stages within, and delete task templates — plus an "Apply template" button on the opportunity detail page that previews the computed task due-dates and, on confirm, materialises the stages as a ready-to-work chain of tasks on the kanban board**,
so that **I can skip the 8–15-click boilerplate of hand-building the same preparation checklist for every opportunity, and my team lands on a Story 10.14 kanban board pre-populated with tasks wired up by `finish_to_start` dependencies and due-dates computed backwards from the opportunity deadline — closing Epic 10 AC6 ("Task templates can be CRUDed per company; applying a template to an opportunity creates concrete tasks with deadlines computed from the opportunity deadline") and the AC11 sub-item "template manager" in one coherent UX surface.**

## Description

Story 10.15 is the single user-visible surface for Epic 10's task-template subsystem. The backend (Story 10.7) has been `done` since Sprint 11 — this story consumes the existing `POST/GET/PATCH/DELETE /api/v1/task-templates*` and `POST /api/v1/task-templates/{id}/apply` endpoints from two surfaces without backend changes:

1. **Settings → Task Templates page** (`/{locale}/settings/task-templates`) — table of existing templates + full CRUD.
2. **Opportunity detail page → "Apply template" button + modal** — a thin consumer surface that reuses the same API client.

It ships **four tightly-coupled UI surfaces**:

1. **Task Templates Page (`TaskTemplatesPage`).** New route under company settings. Renders a header (`t("taskTemplates.pageTitle")` + "+ New template" button, gated on `canMutate`) and a table (`TaskTemplatesTable`) listing the company's non-deleted templates with columns **Name**, **Opportunity type**, **Stage count**, **Last updated**, and a per-row actions menu (Edit · Duplicate · Delete). Clicking a row (or Edit) opens `<TaskTemplateFormModal mode="edit">`; clicking "+ New template" opens `<TaskTemplateFormModal mode="new">`; clicking Delete opens `<TemplateDeleteConfirm>`. Route is gated by the existing `(protected)` guard; any authenticated company member can read (Story 10.7 AC5 `get_current_user` only); only `bid_manager`+ can mutate (Story 10.7 AC4/AC7/AC8 `require_role("bid_manager")`) — the UI mirrors this by hiding the "+ New template" button and the row actions menu for non-mutating roles.

2. **Template Form Modal (`TaskTemplateFormModal`).** A `<Dialog>` (shadcn primitive). Mode `new` starts empty; mode `edit` hydrates from the selected template. Fields: **Name** (text, required, 1–255 chars — the unique constraint surfaces 409 on submit if a duplicate exists), **Description** (textarea, optional, ≤ 2000 chars), **Opportunity type** (radio group: Tender / Grant / Any — default "Any"), and the **Stage list** (`StageListEditor`). Footer has Cancel + Save buttons. Save submits via `useCreateTaskTemplate()` (new mode) or `useUpdateTaskTemplate(id)` (edit mode) with the full stage array — no delta merging (matches Story 10.7 AC7's "client sends the whole replacement list" contract). A 409 response mounts an inline error next to the Name field (`t("taskTemplates.error.duplicateName")`); a 422 on dependency-indices validation mounts an inline error at the offending stage row (`t("taskTemplates.error.forwardReference")`).

3. **Stage List Editor (`StageListEditor`).** The heart of the form. Renders an ordered list of `<StageRowEditor>` cards wrapped in a `<DndContext>` (reusing the `@dnd-kit/core` + `@dnd-kit/sortable` dep brought in by Story 10.14 — no new package). Each stage row has:
   - A drag handle (`<GripVertical>`) on the left — drag-to-reorder; reindexes `dependency_indices` automatically on drop (see design decision (b) below).
   - **Title** input (text, required, 1–500 chars).
   - **Description** textarea (optional, ≤ 2000 chars, autosize).
   - **Role** dropdown (`bid_manager` · `technical_writer` · `financial_analyst` · `legal_reviewer` · `read_only`).
   - **Relative days before deadline** number input (integer, 0–365).
   - **Dependency checkboxes** (`<StageDependencyCheckboxes>`) — one checkbox per stage with index `j < i` (i.e. only earlier stages in the list can be referenced). Disabled for index 0 (nothing earlier to reference).
   - A delete button (trash icon) on the right — removes the stage and prunes any later stage's `dependency_indices` that referenced this index (see design decision (b)).

   "+ Add stage" button at the bottom of the list appends an empty stage (default: `{title: "", description: null, role: "bid_manager", relative_days_before_deadline: 0, dependency_indices: []}`).

   The list enforces 1 ≤ stage_count ≤ 50 client-side (matches the CHECK constraint from Story 10.7 AC1). The Save button is disabled when stage_count is 0 or > 50.

4. **Apply Template Modal (`ApplyTemplateModal`).** A `<Dialog>` mounted on the opportunity detail page. Triggered by an "Apply template" button rendered in the opportunity's action bar (only visible to `bid_manager`+ and only when the opportunity has a non-null `deadline` — otherwise the button is disabled with tooltip `t("taskTemplates.apply.needsDeadline")`). On open, lists available templates via `useTaskTemplatesList({ opportunity_type: opportunity.opportunity_type })` — the server returns `tender|grant` + `any` templates per Story 10.7 AC5's wildcard-`any` rule. Each template row shows name, opportunity_type badge (with a "mismatch" subtle indicator if template's type !== opportunity's type AND !== "any"), and stage count. Selecting a template reveals `<ApplyTemplatePreview>` — a table of computed due-dates (stage title, role, computed `due_date = opportunity.deadline - relative_days_before_deadline`, with past-dates highlighted red). Footer has Cancel + **"Apply (creates N tasks)"** button. Confirm fires `useApplyTaskTemplate(templateId).mutate({ opportunity_id })`; on success, shows a toast `t("taskTemplates.apply.success", { count })`, optionally renders the server `warnings` array as a non-blocking banner, and offers a link button "View on task board" that `router.push`es to `/{locale}/tasks?opportunity=<opportunity_id>` (a filtered Story 10.14 board). On 422 (null deadline — should be pre-empted by the button-disabled gate, but surface anyway) shows `t("taskTemplates.apply.error.nullDeadline")`. On 404 (template or opportunity not found) shows `t("taskTemplates.apply.error.notFound")`.

**Six non-obvious design decisions drive the implementation:**

**(a) The stage form uses a plain React-controlled object state, NOT `react-hook-form`, for the stage array.** `react-hook-form`'s `useFieldArray` works for simple cases but its re-index semantics don't compose cleanly with drag-and-drop reorder + `dependency_indices` invariants (we'd have to imperatively `move()` + hand-patch every later `dependency_indices` in a `useEffect` that fires on reorder, which is race-prone). Instead, the form holds `{ name, description, opportunity_type, stages }` in a single `useState` object. Name/description/opportunity_type use RHF + Zod for the scalar fields only (so we keep the error-rendering idiom); the stages array is a plain-state ref that gets Zod-validated via `taskTemplateFormSchema.parse(value)` on submit. The scalar and array states are merged into a single submit payload. This keeps the reorder-path single-threaded: `handleDragEnd` calls `setStages(reorder(stages, fromIdx, toIdx))` where `reorder` is a pure function that (i) moves the dragged element and (ii) rewrites every `dependency_indices` entry to the new index of its target, emitting warnings (but not errors) for dependencies whose target now sits at a higher index than the source (the invariant violation) so the UI can highlight the row in red.

**(b) Stage reorder and stage delete BOTH patch `dependency_indices` in place, preserving the "strictly-less-than-own-index" invariant by construction.** When stage B (index 3) depends on stage A (index 1) and the user drags A from index 1 to index 4, the invariant breaks. Two options: (1) reject the reorder, (2) auto-prune the now-invalid edge. We pick (2) — the `reorder` pure function scans the reordered list: for each stage at new index `i`, it filters `dependency_indices` to keep only entries whose TARGET (an original-index identifier persisted on the in-memory stage object as `__uiId`) now lives at a new index `< i`. A removed edge is logged to a local `reorderWarnings` array that displays a banner `t("taskTemplates.reorderedDropped", { count })` beneath the list — the user can then re-add the dependencies in their new topology. Delete uses the same pattern: when stage at `i` is removed, later stages' `dependency_indices` get their `> i` entries decremented by 1 and their `=== i` entries stripped. Both operations are pure functions with unit tests.

   To make this work, each stage gets an in-memory-only `__uiId` (UUID generated on "+ Add stage" or on hydrate from the server) that persists across reorders. The server contract uses indices, so the submit path strips `__uiId` and converts to positional indices via `stages.findIndex(s => s.__uiId === target)`. Unit-tested in `__tests__/stage-reorder-utils.test.ts`.

**(c) The dependency-checkboxes render is `O(N²)` in stage count — acceptable at N ≤ 50.** For stage at index `i`, we render `i` checkboxes (one per earlier stage). Worst case at N=50 is 1,275 checkboxes — trivially cheap for React. No virtualisation needed. The checkboxes reference earlier stages by their `__uiId` (persistent) not their index (volatile), then convert to indices on submit; this keeps checkbox state stable across reorders. Disabled for stage index 0 (no earlier stages exist).

**(d) The apply-preview table is client-computed; it does NOT call a separate "dry-run apply" endpoint.** Story 10.7 provides no dry-run; re-implementing the preview calc on the server would double the contract surface. The client already has the template's `stages` (from `useTaskTemplatesList` or `useTaskTemplate(id)`) and the opportunity's `deadline` (from `useOpportunity(id)` — Story 6.5). Computing `due_date = deadline - relative_days_before_deadline` is trivial client-side JS (`new Date(deadline.getTime() - days*86400000)`). Past-date detection runs against `new Date()`. The only thing the server knows that the client doesn't is the actual audit-log row that gets written on confirm — and we don't preview that. This matches Story 10.14's "server is the source of truth; client renders what it has" principle.

**(e) Post-apply success uses `router.push` + filter query-param deep-link to the Story 10.14 task board.** The apply response includes `created_task_ids` but the client does not need them — the board query-invalidates on mount and picks up the new tasks naturally. The deep-link `?opportunity=<id>` uses Story 10.14 AC8's filter URL contract to scope the board to just the newly-created chain. A follow-up feature (future story) could scroll-into-view the first pending task; out of scope here.

**(f) "Duplicate" action in the row menu is a client-side convenience: it pre-fills the Create modal with the selected template's stages + `name + " (copy)"`, not a server-side clone endpoint.** This keeps the backend surface minimal (Story 10.7 didn't add a clone endpoint). The user hits Save and the partial-unique `(company_id, name)` constraint accepts the suffixed name. If the `(copy)` name also collides (already been used), the 409 surfaces and the user edits the name. Unit-testable as a pure transform.

The page and the apply modal are independent surfaces (the page is a standalone route; the apply modal is a conditional child of the opportunity detail page). Either can be deployed independently behind a feature flag if rollback is needed. The `TaskTemplateFormModal` is only mounted inside `TaskTemplatesPage`; the opportunity detail path never triggers it.

## Acceptance Criteria

### File-system & Routing

1. [ ] **AC1 — New route, components, API client, queries, and modified opportunity-detail page files exist under `apps/client/`.**

    Route (new):
    - `apps/client/app/[locale]/(protected)/settings/task-templates/page.tsx` (`"use client"`; default export `TaskTemplatesPage` that renders `<TaskTemplatesPage />`).

    Components (all under `apps/client/app/[locale]/(protected)/settings/task-templates/components/`):
    - `TaskTemplatesPage.tsx` (`"use client"`; default export `TaskTemplatesPage`).
    - `TaskTemplatesTable.tsx` (`"use client"`; named export `TaskTemplatesTable`).
    - `TaskTemplateFormModal.tsx` (`"use client"`; named export `TaskTemplateFormModal`; also exports `taskTemplateFormSchema` for ATDD test import).
    - `StageListEditor.tsx` (`"use client"`; named export `StageListEditor`).
    - `StageRowEditor.tsx` (`"use client"`; named export `StageRowEditor`).
    - `StageDependencyCheckboxes.tsx` (`"use client"`; named export `StageDependencyCheckboxes`).
    - `TemplateEmptyState.tsx` (`"use client"`; named export `TemplateEmptyState`).
    - `TemplateDeleteConfirm.tsx` (`"use client"`; named export `TemplateDeleteConfirm`).

    Components (all under `apps/client/app/[locale]/(protected)/opportunities/[id]/components/`):
    - `ApplyTemplateModal.tsx` (`"use client"`; named export `ApplyTemplateModal`).
    - `ApplyTemplatePreview.tsx` (`"use client"`; named export `ApplyTemplatePreview`).

    API / queries:
    - `apps/client/lib/api/task-templates.ts` (exports `listTaskTemplates`, `getTaskTemplate`, `createTaskTemplate`, `updateTaskTemplate`, `deleteTaskTemplate`, `applyTaskTemplate`, plus the eight interfaces enumerated in AC2).
    - `apps/client/lib/queries/use-task-templates.ts` (exports `useTaskTemplatesList`, `useTaskTemplate`, `useCreateTaskTemplate`, `useUpdateTaskTemplate`, `useDeleteTaskTemplate`, `useApplyTaskTemplate`).

    Utils:
    - `apps/client/lib/utils/stage-reorder-utils.ts` (exports `reorderStages`, `deleteStage`, `assignUiIds`, `stripUiIds`, `toServerStages` — pure functions; unit-tested separately).

    Modified files:
    - `apps/client/app/[locale]/(protected)/settings/layout.tsx` (or equivalent settings-nav config) — adds a nav entry `{ href: "/{locale}/settings/task-templates", labelKey: "settingsNav.taskTemplates" }` after "Company" and before "Alerts" (or at whatever position the settings-nav convention prescribes; align with the sibling entries). Only renders when the user has role `bid_manager` or `admin` — `read_only` + `technical_writer` + `financial_analyst` + `legal_reviewer` users see settings-for-them-only items and the nav entry is absent. (Server RBAC is still the source of truth; this is a UX-only hide.)
    - `apps/client/app/[locale]/(protected)/opportunities/[id]/page.tsx` — adds an "Apply template" `<Button>` to the opportunity action bar (next to the existing "Create proposal" button) + mounts `<ApplyTemplateModal opportunityId={id} isOpen={isApplyOpen} onClose={handleCloseApply} />` conditionally. Button is disabled (with tooltip) when `opportunity.deadline === null` or `canMutate === false`.

    No new runtime dependencies (the `@dnd-kit/*` packages were added by Story 10.14; re-use them here).

2. [ ] **AC2 — `apps/client/lib/api/task-templates.ts` exports.**

    ```ts
    export type OpportunityType = "tender" | "grant" | "any";
    export type StageRole = "bid_manager" | "technical_writer" | "financial_analyst" | "legal_reviewer" | "read_only";

    export interface StageDefinition {
      title: string;                         // 1..500 chars
      description: string | null;            // null or 0..2000 chars
      role: StageRole;
      relative_days_before_deadline: number; // integer, 0..365
      dependency_indices: number[];          // strictly less than own index; no duplicates
    }

    export interface TaskTemplateResponse {
      id: string;
      company_id: string;
      name: string;
      description: string | null;
      opportunity_type: OpportunityType;
      stages: StageDefinition[];
      created_by: string;
      created_at: string;                    // ISO-8601
      updated_at: string;                    // ISO-8601
    }

    export interface TaskTemplateListResponse {
      items: TaskTemplateResponse[];
      total: number;
      limit: number;
      offset: number;
    }

    export interface TaskTemplateCreateRequest {
      name: string;                          // 1..255
      description?: string | null;           // ≤ 2000
      opportunity_type?: OpportunityType;    // default "any" on server
      stages: StageDefinition[];             // 1..50
    }

    export interface TaskTemplateUpdateRequest {
      name?: string;
      description?: string | null;
      opportunity_type?: OpportunityType;
      stages?: StageDefinition[];
    }

    export interface TaskTemplateApplyRequest {
      opportunity_id: string;                // UUID
    }

    export interface TaskTemplateApplyResponse {
      created_task_ids: string[];            // in stage order
      created_dependency_ids: string[];
      warnings: string[];                    // human-readable
    }

    export interface ListTaskTemplatesParams {
      opportunity_type?: OpportunityType;    // wildcard-any server-side
      limit?: number;                        // default 50; server caps at 100
      offset?: number;
    }
    ```

    Functions (all use the shared `apiClient` pattern from `lib/api/_client.ts`):
    - `listTaskTemplates(params: ListTaskTemplatesParams): Promise<TaskTemplateListResponse>` — `GET /api/v1/task-templates`. Expects 200.
    - `getTaskTemplate(id: string): Promise<TaskTemplateResponse>` — `GET /api/v1/task-templates/{id}`. Expects 200.
    - `createTaskTemplate(payload: TaskTemplateCreateRequest): Promise<TaskTemplateResponse>` — `POST /api/v1/task-templates`. Expects 201.
    - `updateTaskTemplate(id: string, payload: TaskTemplateUpdateRequest): Promise<TaskTemplateResponse>` — `PATCH /api/v1/task-templates/{id}`. Expects 200.
    - `deleteTaskTemplate(id: string): Promise<void>` — `DELETE /api/v1/task-templates/{id}`. Expects 204.
    - `applyTaskTemplate(id: string, payload: TaskTemplateApplyRequest): Promise<TaskTemplateApplyResponse>` — `POST /api/v1/task-templates/{id}/apply`. Expects 201.

    All functions throw the raw `AxiosError` on non-2xx. Caller-relevant status codes (from `services/client-api/src/client_api/api/v1/task_templates.py` + `task_template_service.py`):
    - `createTaskTemplate` — 201; **409 (duplicate `(company_id, name)` — Story 10.7 AC4)**; 422 (Pydantic validation: forward-reference `dependency_indices`, duplicate indices, out-of-range, empty stages, > 50 stages, invalid `opportunity_type`); 403 (non-bid_manager).
    - `updateTaskTemplate` — 200; 409 (rename collision); 422 (same validations); 404 (missing/cross-tenant/soft-deleted); 403.
    - `deleteTaskTemplate` — 204; 404.
    - `getTaskTemplate` — 200; 404.
    - `listTaskTemplates` — 200.
    - `applyTaskTemplate` — 201; **422 (opportunity has no deadline — Story 10.7 AC9 Step 2)**; 404 (template or opportunity not found / cross-tenant); 403.

    The 409 from `createTaskTemplate`/`updateTaskTemplate` MUST surface with the detail body intact so the form can render an inline error.

3. [ ] **AC3 — `apps/client/lib/queries/use-task-templates.ts` exports.**

    ```ts
    export function useTaskTemplatesList(params?: ListTaskTemplatesParams)
        // useQuery({
        //   queryKey: ["task-templates", params ?? {}],
        //   queryFn: () => listTaskTemplates(params ?? {}),
        //   staleTime: 60_000,
        //   refetchOnWindowFocus: false,
        // })

    export function useTaskTemplate(id: string | null)
        // useQuery({
        //   queryKey: ["task-template", id],
        //   queryFn: () => getTaskTemplate(id!),
        //   enabled: id !== null,
        // })

    export function useCreateTaskTemplate()
        // POST /task-templates; invalidates ["task-templates"]; success toast; 409 → inline error returned to caller.
    export function useUpdateTaskTemplate(id: string)
        // PATCH /task-templates/{id}; invalidates ["task-templates"] + ["task-template", id]; 409 → inline error.
    export function useDeleteTaskTemplate(id: string)
        // DELETE /task-templates/{id}; invalidates ["task-templates"]; success toast.
    export function useApplyTaskTemplate(id: string)
        // POST /task-templates/{id}/apply; invalidates ["tasks"] (Story 10.14's cache) on success.
    ```

    Error-surface mapping (applied in each mutation's `onError`):
    - `useCreateTaskTemplate` / `useUpdateTaskTemplate` on 409: DO NOT toast. Return the error up to the caller (form component) which mounts an inline error at the Name field. Rationale: toasts compete with the more discoverable inline error.
    - `useCreateTaskTemplate` / `useUpdateTaskTemplate` on 422: DO NOT toast if the response body has `detail` pointing to a specific field (the form handles it inline). For unmapped 422, toast `t("taskTemplates.error.validation")`.
    - `useApplyTaskTemplate` on 422: toast `t("taskTemplates.apply.error.nullDeadline")`.
    - `useApplyTaskTemplate` on 404: toast `t("taskTemplates.apply.error.notFound")`.
    - `useDeleteTaskTemplate` on 404: silent (template was already gone — invalidate and move on).
    - All 5xx: toast `t("forms.serverError")` + `console.error` with `{ templateId, operation, mutationPayload }` context (no PII).

    Each mutation uses `useUIStore.addToast` (sourced from `@/lib/stores/ui-store`) matching the Story 10.12 / 10.13 / 10.14 conventions. Success toasts:
    - Create: `t("taskTemplates.create.success")`.
    - Update: `t("taskTemplates.update.success")`.
    - Delete: `t("taskTemplates.delete.success")`.
    - Apply: `t("taskTemplates.apply.success", { count: response.created_task_ids.length })`.

4. [ ] **AC4 — `TaskTemplatesPage` (route root).**

    Rendered by `app/[locale]/(protected)/settings/task-templates/page.tsx`.

    Structure (in order):
    - Page header: `<h1>{t("taskTemplates.pageTitle")}</h1>` + subtitle `{t("taskTemplates.pageSubtitle")}` + "+ New template" `<Button>` (right-aligned; `data-testid="template-new-btn"`; disabled + tooltipped when `canMutate === false`).
    - Body states driven by `useTaskTemplatesList()`:
      - **Loading** — skeleton rows in the table; `data-testid="templates-table-skeleton"`.
      - **Error** — `<BoardErrorState />` reused from Story 10.14 (or equivalent) with retry; `data-testid="templates-error"`.
      - **Empty (no templates)** — `<TemplateEmptyState />` with copy `t("taskTemplates.emptyTitle")` + `t("taskTemplates.emptyDescription")` + a "+ Create your first template" CTA; `data-testid="templates-empty"`.
      - **Populated** — `<TaskTemplatesTable templates={data.items} canMutate={canMutate} onEdit={openEditModal} onDuplicate={openDuplicateModal} onDelete={openDeleteConfirm} />`.
    - `<TaskTemplateFormModal mode={formMode} templateId={editingId} seedStages={duplicateSeed} isOpen={isFormOpen} onClose={handleCloseForm} />` — always mounted; hidden when `isFormOpen === false`.
    - `<TemplateDeleteConfirm templateId={deletingId} isOpen={isDeleteOpen} onClose={handleCloseDelete} />` — same.

    `canMutate` derivation:
    ```ts
    const canMutate = useAuthStore(s => s.user?.role === "admin" || s.user?.role === "bid_manager");
    ```

    The root container has `data-testid="task-templates-page"`.

5. [ ] **AC5 — `TaskTemplatesTable`.**

    Props: `{ templates: TaskTemplateResponse[]; canMutate: boolean; onEdit: (id: string) => void; onDuplicate: (template: TaskTemplateResponse) => void; onDelete: (id: string) => void }`.

    Renders a `<table data-testid="templates-table">` with columns:
    - **Name** — `template.name` (`data-testid={`template-name-${id}`}`). Clicking opens Edit.
    - **Opportunity type** — `<Badge>{t(`taskTemplates.opportunityType.${opportunity_type}`)}</Badge>` (`data-testid={`template-type-${id}`}`).
    - **Stages** — `template.stages.length` (`data-testid={`template-stages-${id}`}`).
    - **Last updated** — `Intl.DateTimeFormat(locale, { dateStyle: "medium" }).format(new Date(updated_at))`.
    - **Actions** — a `<DropdownMenu>` with Edit / Duplicate / Delete items. Hidden entirely when `canMutate === false`. Each menu item has a stable testid: `template-edit-${id}`, `template-duplicate-${id}`, `template-delete-${id}`.

    Sort: `updated_at DESC` (matches the server's default order).

6. [ ] **AC6 — `TaskTemplateFormModal`.**

    Props: `{ mode: "new" | "edit" | "duplicate"; templateId: string | null; seedStages?: StageDefinition[]; isOpen: boolean; onClose: () => void }`.

    Renders a `<Dialog open={isOpen} onOpenChange={(open) => !open && onClose()}>`. Root `data-testid="template-form-modal"`.

    Hydration:
    - `mode === "new"` — empty state: `{ name: "", description: "", opportunity_type: "any", stages: [] }`.
    - `mode === "edit"` — `useTaskTemplate(templateId)` hydrates the initial form state; while loading, render a modal-body spinner.
    - `mode === "duplicate"` — seed from `seedStages` + `{ name: origName + " (copy)", description: origDescription, opportunity_type: origType }`. No pre-existing `templateId` — submit uses `useCreateTaskTemplate()`.

    Fields (in order):
    - **Name** — `<Input data-testid="template-name-input" maxLength={255} required />`. Zod: `z.string().trim().min(1).max(255)`. On 409, render `<p data-testid="template-name-error">{t("taskTemplates.error.duplicateName")}</p>` below the input.
    - **Description** — `<Textarea data-testid="template-description-input" maxLength={2000} rows={3} />`. Zod: `z.string().max(2000).optional().nullable()`.
    - **Opportunity type** — `<RadioGroup data-testid="template-type-radio">` with three options: Tender, Grant, Any. Default "Any".
    - **Stages** — `<StageListEditor value={stages} onChange={setStages} />` (see AC7).

    Footer:
    - `<Button data-testid="template-cancel-btn" variant="ghost" onClick={onClose}>{t("common.cancel")}</Button>`.
    - `<Button data-testid="template-save-btn" variant="primary" disabled={!canSubmit} onClick={handleSave}>{t(mode === "edit" ? "common.save" : "taskTemplates.form.createBtn")}</Button>`.

    `canSubmit` = Zod schema (`taskTemplateFormSchema`) passes AND `stages.length >= 1 && stages.length <= 50`.

    `handleSave`:
    - Parse via `taskTemplateFormSchema` — if fails, highlight fields (RHF error state).
    - Convert `stages` from `__uiId`-carrying form state to server stages via `toServerStages(stages)` (see AC8).
    - Submit via `useCreateTaskTemplate` (new/duplicate) or `useUpdateTaskTemplate(templateId)` (edit).
    - On success: close modal (the mutation already toasted + invalidated).
    - On 409: render inline error (do not close).
    - On 422: render inline error at the mapped field (if the server detail points to a stage index, highlight that stage row; otherwise a modal-level error banner).

    Export `taskTemplateFormSchema` at module level so the ATDD test can import it:
    ```ts
    export const taskTemplateFormSchema = z.object({
      name: z.string().trim().min(1).max(255),
      description: z.string().max(2000).nullable().optional(),
      opportunity_type: z.enum(["tender", "grant", "any"]),
      stages: z.array(stageSchema).min(1).max(50),
    });
    ```

    The modal traps focus via the `Dialog` primitive; Escape closes.

7. [ ] **AC7 — `StageListEditor` + `StageRowEditor`.**

    `StageListEditor` props: `{ value: StageFormState[]; onChange: (next: StageFormState[]) => void }`.

    `StageFormState` shape (in-memory, NOT sent to the server):
    ```ts
    interface StageFormState extends StageDefinition {
      __uiId: string;                // UUID v4; generated on add / hydrate
    }
    ```

    Renders:
    ```tsx
    <DndContext sensors={sensors} collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
      <SortableContext items={value.map(s => s.__uiId)} strategy={verticalListSortingStrategy}>
        <ul data-testid="stage-list" className="flex flex-col gap-2">
          {value.map((stage, index) => (
            <StageRowEditor
              key={stage.__uiId}
              stage={stage}
              index={index}
              allStages={value}
              onChange={(next) => onChange(value.map((s, i) => i === index ? next : s))}
              onDelete={() => onChange(deleteStage(value, index))}
            />
          ))}
        </ul>
      </SortableContext>
      <button data-testid="stage-add-btn" type="button" onClick={handleAddStage}>+ {t("taskTemplates.form.addStage")}</button>
      {reorderWarnings.length > 0 && (
        <div data-testid="stage-reorder-warnings" role="alert" className="mt-2 text-xs text-orange-700">
          {t("taskTemplates.form.reorderedDropped", { count: reorderWarnings.length })}
        </div>
      )}
    </DndContext>
    ```

    Sensors (same pattern as Story 10.14 AC5):
    - `PointerSensor` with `activationConstraint: { distance: 3 }`.
    - `KeyboardSensor` with `sortableKeyboardCoordinates`.

    `handleDragEnd` (skeleton):
    ```ts
    const handleDragEnd = (event: DragEndEvent) => {
      const { active, over } = event;
      if (!over || active.id === over.id) return;
      const fromIdx = value.findIndex(s => s.__uiId === active.id);
      const toIdx = value.findIndex(s => s.__uiId === over.id);
      const { stages: reordered, droppedCount } = reorderStages(value, fromIdx, toIdx);
      setReorderWarnings(prev => droppedCount > 0 ? [...prev, { droppedCount }] : prev);
      onChange(reordered);
    };
    ```

    `StageRowEditor` props: `{ stage: StageFormState; index: number; allStages: StageFormState[]; onChange: (next: StageFormState) => void; onDelete: () => void }`.

    Renders a `<li data-testid={`stage-row-${index}`}>` with `useSortable({ id: stage.__uiId })` wiring. Layout:
    - Drag handle (`<GripVertical>`) on the left, `data-testid={`stage-drag-${index}`}`.
    - Title input: `<Input data-testid={`stage-title-${index}`} maxLength={500} required />`.
    - Description textarea: `<Textarea data-testid={`stage-description-${index}`} maxLength={2000} rows={2} />`.
    - Role dropdown: `<Select data-testid={`stage-role-${index}`}>` with five options (`bid_manager`, `technical_writer`, `financial_analyst`, `legal_reviewer`, `read_only`). Option labels come from `t(`taskTemplates.form.role.${value}`)`.
    - Relative days input: `<Input type="number" data-testid={`stage-days-${index}`} min={0} max={365} />`.
    - Dependency checkboxes: `<StageDependencyCheckboxes index={index} allStages={allStages} value={stage.dependency_indices} onChange={handleDepsChange} />` (see below). Only rendered when `index > 0`.
    - Delete button: `<button data-testid={`stage-delete-${index}`} aria-label={t("taskTemplates.form.deleteStageAria", { index: index + 1 })}><Trash2 className="h-4 w-4" /></button>`.

    All input blur handlers fire `onChange` with the patched stage — NO autosave (the modal save button is the single submit point).

    Zod per-stage schema (used by `taskTemplateFormSchema.stages[i]`):
    ```ts
    const stageSchema = z.object({
      __uiId: z.string().uuid(),
      title: z.string().trim().min(1).max(500),
      description: z.string().max(2000).nullable().optional(),
      role: z.enum(["bid_manager", "technical_writer", "financial_analyst", "legal_reviewer", "read_only"]),
      relative_days_before_deadline: z.number().int().min(0).max(365),
      dependency_indices: z.array(z.number().int().nonnegative()).default([]),
    });
    ```

8. [ ] **AC8 — `StageDependencyCheckboxes` + `stage-reorder-utils` contract.**

    `StageDependencyCheckboxes` props: `{ index: number; allStages: StageFormState[]; value: number[]; onChange: (next: number[]) => void }`.

    Renders:
    ```tsx
    <fieldset data-testid={`stage-deps-${index}`} className="mt-2">
      <legend className="text-xs text-muted-foreground">{t("taskTemplates.form.dependsOn")}</legend>
      <div className="flex flex-wrap gap-2">
        {allStages.slice(0, index).map((earlier, earlierIdx) => (
          <label key={earlier.__uiId} className="flex items-center gap-1 text-xs">
            <input
              type="checkbox"
              data-testid={`stage-dep-${index}-${earlierIdx}`}
              checked={value.includes(earlierIdx)}
              onChange={(e) => onChange(
                e.target.checked
                  ? [...value, earlierIdx].sort((a, b) => a - b)
                  : value.filter(i => i !== earlierIdx)
              )}
            />
            <span>{earlier.title || t("taskTemplates.form.stageUntitled", { index: earlierIdx + 1 })}</span>
          </label>
        ))}
      </div>
    </fieldset>
    ```

    When `index === 0`, the whole fieldset is absent (nothing earlier exists).

    Pure-function contract (`apps/client/lib/utils/stage-reorder-utils.ts`):

    ```ts
    export function assignUiIds(stages: StageDefinition[]): StageFormState[]
        // Adds a fresh __uiId to each stage (used when hydrating a server template into the form).

    export function stripUiIds(stages: StageFormState[]): StageDefinition[]
        // Strips __uiId from each stage (intermediate step before toServerStages).

    export function toServerStages(stages: StageFormState[]): StageDefinition[]
        // Converts __uiId-based dependency references (if any — today dependency_indices carry
        // numeric indices directly, so this is effectively an identity strip) into server-shape
        // StageDefinition[]. MUST preserve dependency_indices as-is AND strip __uiId.

    export interface ReorderResult { stages: StageFormState[]; droppedCount: number }
    export function reorderStages(stages: StageFormState[], fromIdx: number, toIdx: number): ReorderResult
        // Moves element from fromIdx to toIdx. For each stage at its new index i, filters
        // dependency_indices to keep only entries j where (a) the element originally at index j
        // now sits at new index < i AND (b) is still present. Returns { stages, droppedCount }
        // where droppedCount counts the number of dependency edges discarded.

    export function deleteStage(stages: StageFormState[], index: number): StageFormState[]
        // Removes stages[index]; for each later stage, rewrites dependency_indices:
        //   strip entries === index; decrement entries > index by 1.
    ```

    All four functions are pure (no side effects, no `crypto.randomUUID` except in `assignUiIds` where a mocked RNG is accepted via an optional second arg for unit tests).

    Unit tests (`apps/client/lib/utils/__tests__/stage-reorder-utils.test.ts`) cover:
    - `reorderStages` move forward preserves valid dependency edges.
    - `reorderStages` move backward drops newly-forward-referenced edges AND reports `droppedCount`.
    - `reorderStages` on same index is a no-op.
    - `deleteStage` decrements later stages' dependency_indices.
    - `deleteStage` strips edges that referenced the deleted stage.
    - `assignUiIds` produces unique UUIDs.
    - `toServerStages` strips `__uiId` and preserves `dependency_indices`.

9. [ ] **AC9 — `ApplyTemplateModal` + `ApplyTemplatePreview`.**

    `ApplyTemplateModal` props: `{ opportunityId: string; isOpen: boolean; onClose: () => void }`.

    Mounted by the opportunity detail page. Opens on "Apply template" button click. Root `data-testid="apply-template-modal"`.

    Layout:
    - Header: `<DialogTitle>{t("taskTemplates.apply.title")}</DialogTitle>`.
    - Body (two-column when a template is selected; one-column list when not):
      - **Left column — Template list.** `useTaskTemplatesList({ opportunity_type: opportunity.opportunity_type })` — Story 10.7 AC5's wildcard-any server rule means `tender` opportunities see both `tender` and `any` templates, `grant` opportunities see both `grant` and `any`. Each template rendered as a clickable row:
        ```tsx
        <button
          data-testid={`apply-template-option-${template.id}`}
          className={cn("flex flex-col items-start gap-1 p-3 border rounded hover:bg-muted w-full", selectedId === template.id && "ring-2 ring-primary")}
          onClick={() => setSelectedId(template.id)}
        >
          <span className="font-medium">{template.name}</span>
          <div className="flex gap-2 text-xs">
            <Badge>{t(`taskTemplates.opportunityType.${template.opportunity_type}`)}</Badge>
            <span>{t("taskTemplates.apply.stageCount", { count: template.stages.length })}</span>
          </div>
          {isMismatch(template, opportunity) && <span data-testid={`apply-template-mismatch-${template.id}`} className="text-xs text-orange-700">⚠ {t("taskTemplates.apply.typeMismatch")}</span>}
        </button>
        ```
      - **Right column — Preview.** When `selectedId !== null`, render `<ApplyTemplatePreview template={selectedTemplate} opportunity={opportunity} />` (see below). When null, render a muted hint: `t("taskTemplates.apply.selectHint")`.

    - Footer:
      - Cancel button (`data-testid="apply-cancel-btn"`).
      - Apply button: `<Button data-testid="apply-confirm-btn" disabled={!selectedId || applyMutation.isPending} onClick={handleApply}>{t("taskTemplates.apply.confirmBtn", { count: selectedTemplate?.stages.length ?? 0 })}</Button>`.

    `handleApply`:
    ```ts
    applyMutation.mutate({ opportunity_id: opportunityId }, {
      onSuccess: (response) => {
        addToast({ type: "success", title: t("taskTemplates.apply.successTitle"), description: t("taskTemplates.apply.success", { count: response.created_task_ids.length }) });
        if (response.warnings.length > 0) setBannerWarnings(response.warnings);
        queryClient.invalidateQueries({ queryKey: ["tasks"] });
        // Give the user a follow-up action: close + navigate, or stay on this page.
      },
    });
    ```

    Post-success state shows a success summary with two action buttons: "View on task board" (`data-testid="apply-view-board-btn"` → `router.push(`/${locale}/tasks?opportunity=${opportunityId}`)`) and "Close". Any `warnings` from the response render as a non-blocking `<Alert variant="warning">` inside the summary.

    Disabled-button rule on the opportunity detail page: if `opportunity.deadline === null`, the "Apply template" button is `disabled` with a tooltip `t("taskTemplates.apply.needsDeadline")`. This pre-empts the 422 path but the client still handles it defensively (a stale opportunity fetched before a deadline was cleared could slip through).

    `ApplyTemplatePreview` props: `{ template: TaskTemplateResponse; opportunity: { deadline: string | null } }`.

    Renders a `<table data-testid="apply-preview-table">` with rows, one per stage:
    - **Stage #** — 1-indexed.
    - **Title** — `stage.title`.
    - **Role** — `t(`taskTemplates.form.role.${stage.role}`)`.
    - **Due date** — computed `Intl.DateTimeFormat(locale, { dateStyle: "medium" }).format(computeDueDate(opportunity.deadline, stage.relative_days_before_deadline))`. If the computed date is in the past, wrap in `<span data-testid={`apply-preview-overdue-${idx}`} className="text-red-600">…</span>` and append a warning icon.
    - **Depends on** — `stage.dependency_indices.map(j => `#${j+1}`).join(", ")` or `—` if empty.

    Pure helper `computeDueDate(isoDeadline: string | null, daysBefore: number): Date | null` — returns `null` when deadline is null; otherwise `new Date(deadline.getTime() - daysBefore * 86_400_000)`. Unit-tested in `stage-reorder-utils.test.ts` (or a sibling `apply-preview-utils.test.ts`).

10. [ ] **AC10 — i18n keys (`apps/client/messages/{en,bg}.json`).**

    New namespace `taskTemplates.*` with a minimum of **40 keys**. EN is authoritative; BG parity is enforced by `pnpm check:i18n`.

    Also add:
    - `settingsNav.taskTemplates` — nav label for the settings sidebar.
    - `nav.settings.taskTemplates` (if the nav config uses that key shape — align with existing settings nav entries; the exact key path follows sibling entries like `settingsNav.company`).

    Key inventory (EN — representative list; developer adds the full ≥ 40):
    - `pageTitle` — "Task templates"
    - `pageSubtitle` — "Reusable task checklists that apply to opportunities and auto-compute due dates from the deadline."
    - `newTemplateBtn` — "New template"
    - `emptyTitle` — "No templates yet"
    - `emptyDescription` — "Create your first template to standardise your bid preparation workflow."
    - `emptyCta` — "Create your first template"
    - `opportunityType.tender` — "Tender"
    - `opportunityType.grant` — "Grant"
    - `opportunityType.any` — "Any"
    - `table.name` — "Name"
    - `table.type` — "Type"
    - `table.stages` — "Stages"
    - `table.updated` — "Last updated"
    - `table.actions` — "Actions"
    - `table.edit` — "Edit"
    - `table.duplicate` — "Duplicate"
    - `table.delete` — "Delete"
    - `form.nameLabel` — "Template name"
    - `form.descriptionLabel` — "Description"
    - `form.typeLabel` — "Opportunity type"
    - `form.stagesLabel` — "Stages"
    - `form.addStage` — "Add stage"
    - `form.deleteStageAria` — "Delete stage {index}"
    - `form.stageTitleLabel` — "Title"
    - `form.stageDescriptionLabel` — "Description"
    - `form.stageRoleLabel` — "Role"
    - `form.stageDaysLabel` — "Days before deadline"
    - `form.dependsOn` — "Depends on"
    - `form.stageUntitled` — "Stage {index} (untitled)"
    - `form.role.bid_manager` — "Bid manager"
    - `form.role.technical_writer` — "Technical writer"
    - `form.role.financial_analyst` — "Financial analyst"
    - `form.role.legal_reviewer` — "Legal reviewer"
    - `form.role.read_only` — "Read-only"
    - `form.createBtn` — "Create template"
    - `form.reorderedDropped` — "{count, plural, one {# dependency was removed due to reorder.} other {# dependencies were removed due to reorder.}}"
    - `create.success` — "Template created."
    - `update.success` — "Template updated."
    - `delete.success` — "Template deleted."
    - `delete.confirmTitle` — "Delete template?"
    - `delete.confirmDescription` — "This cannot be undone. Tasks already created from this template are not affected."
    - `error.duplicateName` — "A template with this name already exists."
    - `error.forwardReference` — "Stage dependencies must reference earlier stages only."
    - `error.validation` — "Could not save the template — check for errors."
    - `apply.title` — "Apply task template"
    - `apply.stageCount` — "{count, plural, one {# stage} other {# stages}}"
    - `apply.typeMismatch` — "Type mismatch — template may not fit this opportunity"
    - `apply.selectHint` — "Select a template on the left to see a preview."
    - `apply.confirmBtn` — "{count, plural, =0 {Apply} one {Apply (creates # task)} other {Apply (creates # tasks)}}"
    - `apply.successTitle` — "Template applied"
    - `apply.success` — "{count, plural, one {Created # task.} other {Created # tasks.}}"
    - `apply.error.nullDeadline` — "This opportunity has no deadline — set a deadline first."
    - `apply.error.notFound` — "Template or opportunity was not found."
    - `apply.needsDeadline` — "This opportunity has no deadline — set a deadline to apply a template."
    - `apply.viewBoardBtn` — "View on task board"

    BG translations MUST land in `bg.json` in the same key positions with full ICU plural parity. Example seeds:
    - `pageTitle` — "Шаблони за задачи"
    - `opportunityType.tender` — "Тръжна процедура"
    - `opportunityType.grant` — "Грант"
    - `opportunityType.any` — "Всякакъв"
    - (full set added by the developer; `pnpm check:i18n` enforces parity)

    Also add the navigation keys to both locales.

11. [ ] **AC11 — Opportunity detail page integration.**

    Modify `apps/client/app/[locale]/(protected)/opportunities/[id]/page.tsx`:
    - Add an "Apply template" button to the opportunity's action bar, positioned next to the existing "Create proposal" button (or whatever the sibling convention prescribes — follow the existing action-bar layout). `data-testid="opportunity-apply-template-btn"`.
    - Button `disabled` when `opportunity.deadline === null` OR `canMutate === false`. Tooltip shows `t("taskTemplates.apply.needsDeadline")` (deadline case) or `t("opportunities.readOnlyAction")` (role case). Testid on the disabled state: still `opportunity-apply-template-btn` (disabled attribute distinguishes).
    - Mount `<ApplyTemplateModal opportunityId={params.id} isOpen={isApplyOpen} onClose={() => setIsApplyOpen(false)} />` conditionally (always in the JSX tree; the modal handles its own closed state, but gate via `isOpen` prop).

    Regression guard: re-run the existing opportunity-detail ATDD test (`apps/client/__tests__/opportunities-detail-s6-11.test.ts`). The new button must not break existing testid assertions — add it after the last current action button and do not reorder any existing testids.

12. [ ] **AC12 — ATDD test file `apps/client/__tests__/task-templates-s10-15.test.ts`.**

    Static / structural Vitest test mirroring the Story 10.13 / 10.14 pattern (structural file-system + selective behavioural RTL smoke). RED-phase: behavioural tests applied with `it.skip`. Total expected assertions: **≥ 40 across 11 ACs**.

    **File-system asserts (AC1):** `existsSync` for each of the 8 settings-page components + 2 opportunity-detail components + the route `page.tsx` + `lib/api/task-templates.ts` + `lib/queries/use-task-templates.ts` + `lib/utils/stage-reorder-utils.ts`. Assert the opportunity-detail page file was modified (grep for the testid `opportunity-apply-template-btn`).

    **No-new-dependencies guard (AC1):** diff `apps/client/package.json` — assert NO new top-level entries under `dependencies` relative to the pre-story snapshot (captured inline in the test as a whitelist). The `@dnd-kit/*` packages must remain pinned unchanged.

    **API client exports (AC2):** parse `lib/api/task-templates.ts` for named exports `listTaskTemplates`, `getTaskTemplate`, `createTaskTemplate`, `updateTaskTemplate`, `deleteTaskTemplate`, `applyTaskTemplate`; assert each interface/type name (`OpportunityType`, `StageRole`, `StageDefinition`, `TaskTemplateResponse`, `TaskTemplateListResponse`, `TaskTemplateCreateRequest`, `TaskTemplateUpdateRequest`, `TaskTemplateApplyRequest`, `TaskTemplateApplyResponse`, `ListTaskTemplatesParams`) appears as an `export`. Assert the URL literal `/api/v1/task-templates` appears in the file; assert the apply URL `/api/v1/task-templates/${id}/apply` appears.

    **Hook exports (AC3):** assert each hook appears as a top-level `export function` in `lib/queries/use-task-templates.ts`: `useTaskTemplatesList`, `useTaskTemplate`, `useCreateTaskTemplate`, `useUpdateTaskTemplate`, `useDeleteTaskTemplate`, `useApplyTaskTemplate`.

    **Util exports (AC8):** assert `reorderStages`, `deleteStage`, `assignUiIds`, `stripUiIds`, `toServerStages` are all exported from `lib/utils/stage-reorder-utils.ts`.

    **i18n parity (AC10):** load `en.json` and `bg.json`; assert every `taskTemplates.*` key in en is present in bg AND vice versa; assert the key count is ≥ 40; assert the plural keys (`apply.stageCount`, `apply.confirmBtn`, `apply.success`, `form.reorderedDropped`) include ICU plural syntax in both locales. Assert `settingsNav.taskTemplates` exists in both.

    **Component data-testid presence (AC4–AC11):** read each component file as text; assert the prescribed `data-testid` strings appear as literal substrings:
    - `TaskTemplatesPage.tsx` → `task-templates-page`, `template-new-btn`, `templates-table-skeleton`, `templates-error`, `templates-empty`.
    - `TaskTemplatesTable.tsx` → `templates-table`, `template-name-`, `template-type-`, `template-stages-`, `template-edit-`, `template-duplicate-`, `template-delete-`.
    - `TaskTemplateFormModal.tsx` → `template-form-modal`, `template-name-input`, `template-name-error`, `template-description-input`, `template-type-radio`, `template-cancel-btn`, `template-save-btn`.
    - `StageListEditor.tsx` → `stage-list`, `stage-add-btn`, `stage-reorder-warnings`.
    - `StageRowEditor.tsx` → `stage-row-`, `stage-drag-`, `stage-title-`, `stage-description-`, `stage-role-`, `stage-days-`, `stage-delete-`.
    - `StageDependencyCheckboxes.tsx` → `stage-deps-`, `stage-dep-`.
    - `ApplyTemplateModal.tsx` → `apply-template-modal`, `apply-template-option-`, `apply-cancel-btn`, `apply-confirm-btn`, `apply-view-board-btn`.
    - `ApplyTemplatePreview.tsx` → `apply-preview-table`, `apply-preview-overdue-`.
    - Opportunity detail page (grep): `opportunity-apply-template-btn`.

    **Zod schema asserts (exported from `TaskTemplateFormModal.tsx`):**
    - `test_template_form_schema_rejects_empty_name` — `""` and `"   "` fail.
    - `test_template_form_schema_rejects_over_255_char_name` — 256 chars fails; 255 passes.
    - `test_template_form_schema_rejects_empty_stages` — `[]` fails.
    - `test_template_form_schema_rejects_51_stages` — 51 fails; 50 passes.
    - `test_template_form_schema_rejects_forward_reference` — a stage at index 1 with `dependency_indices: [2]` fails (stage 2 would be later than itself — Pydantic-side enforces this; Zod mirror enforces the `>= 0` bound and leaves cross-stage invariant to `refine`).

    **Pure-function asserts (AC8) — covered in `lib/utils/__tests__/stage-reorder-utils.test.ts`:**
    - `reorderStages` happy-forward preserves edges.
    - `reorderStages` move-backward drops the newly-forward edge + reports `droppedCount: 1`.
    - `deleteStage` prunes + decrements.
    - `toServerStages` strips `__uiId` and preserves `dependency_indices`.
    - `computeDueDate` subtracts `N * 86_400_000` correctly across DST; returns `null` on null input.

    **Behavioural smoke (React Testing Library; `it.skip` upfront per RED-phase pattern):**
    - `test_templates_page_renders_empty_state_when_no_templates` — mock `useTaskTemplatesList` to return `{ items: [], total: 0 }`; assert `templates-empty` is in the document.
    - `test_templates_page_renders_table_when_templates_exist` — mock with 3 items; assert `template-name-{id}` appears 3x.
    - `test_new_template_button_opens_form_modal` — click `template-new-btn`; assert `template-form-modal` appears.
    - `test_form_modal_save_fires_create_mutation` — fill name + one stage + click Save; assert `useCreateTaskTemplate().mutate` was called with the expected payload (`stages` stripped of `__uiId`).
    - `test_form_modal_409_renders_inline_name_error` — mock `useCreateTaskTemplate` to reject with `{response: {status: 409}}`; assert `template-name-error` appears with text matching `t("taskTemplates.error.duplicateName")`.
    - `test_stage_row_delete_calls_delete_util` — render 3 stages; click `stage-delete-1`; assert the list reduces to 2 AND later stages' dependency_indices were decremented (spot-check via re-render assertion).
    - `test_dependency_checkboxes_only_show_earlier_stages` — render 3 stages; assert `stage-dep-0-*` has 0 checkboxes, `stage-dep-1-0` exists, `stage-dep-2-0` and `stage-dep-2-1` exist.
    - `test_duplicate_action_seeds_form_with_copy_suffix` — click `template-duplicate-{id}`; assert the form opens with `template-name-input` populated with `origName + " (copy)"` AND the stage count matches.
    - `test_delete_confirm_fires_delete_mutation` — click Delete; confirm in the alert dialog; assert `useDeleteTaskTemplate(id).mutate()` was called.
    - `test_apply_modal_lists_templates_scoped_to_opportunity_type` — mock `useTaskTemplatesList({opportunity_type: "tender"})`; assert the list query was called with that param.
    - `test_apply_preview_shows_overdue_row_when_due_date_is_past` — given a template with `relative_days_before_deadline: 100` and a deadline 50 days out, assert `apply-preview-overdue-0` appears.
    - `test_apply_confirm_fires_mutation_and_deep_links_to_board` — click Apply; mock `useApplyTaskTemplate` to resolve with `{created_task_ids: ["t1","t2"], warnings: []}`; click "View on task board"; assert `router.push` was called with `/{locale}/tasks?opportunity={opportunityId}`.
    - `test_apply_422_null_deadline_shows_toast` — mock mutation reject with 422; assert `addToast` called with description matching `t("taskTemplates.apply.error.nullDeadline")`.
    - `test_read_only_role_hides_new_template_button` — mock `useAuthStore.user.role = "read_only"`; render the page; assert `template-new-btn` is NOT in the document; assert the settings-nav entry for task-templates is NOT in the document.
    - `test_opportunity_detail_apply_button_disabled_when_deadline_null` — mock `useOpportunity` with `deadline: null`; render the page; assert `opportunity-apply-template-btn` has the `disabled` attribute.
    - `test_reorder_warning_banner_appears_on_move_backward` — render a 3-stage form with stage 2 depending on stage 0; simulate dragging stage 0 to index 2; assert `stage-reorder-warnings` appears with text matching `t("taskTemplates.form.reorderedDropped", {count: 1})`.

    Use `vi.mock("@/lib/stores/ui-store")` to stub `useUIStore.addToast`. Use `QueryClientProvider` wrapping for any hook test — reuse the existing `apps/client/lib/queries/__tests__/test-utils.ts` helper. Use `vi.mock("next/navigation")` to stub `useRouter` for deep-link assertions.

13. [ ] **AC13 — No regressions in prior Epic 10 frontend suites + opportunity detail test.** After implementing this story, re-run:
    - `vitest run __tests__/collaborator-lock-s10-12.test.ts` → expect prior pass count unchanged.
    - `vitest run __tests__/comments-sidebar-s10-13.test.ts` → expect prior pass count unchanged.
    - `vitest run __tests__/task-kanban-s10-14.test.ts` → expect prior pass count unchanged.
    - `vitest run __tests__/opportunities-detail-s6-11.test.ts` → expect prior pass count unchanged (the new "Apply template" button must not alter existing testid assertions).
    - `pnpm check:i18n` → 0 exit.

    The new `taskTemplates.*` i18n namespace MUST NOT collide with any existing namespace. The new settings-nav entry must be additive and must not change the `data-testid` surface of existing nav items.

## Design Constraints

- **No new runtime dependencies.** Stage drag-and-drop reuses the `@dnd-kit/*` packages brought in by Story 10.14. If `@dnd-kit/*` are ever removed (unlikely), this story would need its own add; in practice they are shared workspace-scope.
- **Stage form uses plain `useState` for the array; RHF only handles scalar fields.** The reorder + dependency-indices invariants require a pure-function pipeline (`reorderStages`, `deleteStage`) that composes cleanly without RHF's imperative `useFieldArray` handles. Zod validation runs once on submit against the merged object.
- **Every stage carries an in-memory-only `__uiId`.** The server contract is index-based; `__uiId` is the client's stable identity across reorders. It is stripped before submit via `toServerStages`.
- **Reorder + delete auto-prune `dependency_indices` to preserve the strictly-less-than-own-index invariant.** Dropped edges are reported via a banner (`reorderWarnings`) so the user knows what was removed.
- **Dependency checkboxes are `O(N²)` in stage count — acceptable at N ≤ 50.** No virtualisation.
- **Apply preview is client-computed; no server dry-run.** The `computeDueDate` function is a pure helper; the client has all the inputs (template.stages + opportunity.deadline).
- **Apply deep-links to Story 10.14's task board via `?opportunity=<id>` filter URL.** The board naturally picks up the new tasks on invalidation; no need to echo `created_task_ids`.
- **"Duplicate" action is a client-side convenience (no server-side clone endpoint).** Pre-fills the create form with `{name + " (copy)"}`.
- **409 on create/update surfaces as an inline error on the Name field, not a toast.** Inline is more discoverable; toast is reserved for server/network errors.
- **422 on Pydantic validation surfaces as an inline error at the offending stage row (when resolvable) or a modal-level banner (when not).** The form's own client-side Zod check should pre-empt most 422s; server 422s are primarily the forward-reference check (which the client mirrors) and the schema boundaries (255-char name, 50-stage cap).
- **"Apply template" button on the opportunity detail page is disabled when `deadline === null` OR `canMutate === false`.** Tooltip explains the disable reason.
- **`read_only` + non-mutating roles see the settings-nav entry absent AND the "+ New template" button absent on the page.** If they navigate directly to the URL, the page loads (server returns templates via `get_current_user`) but mutation affordances are hidden.
- **Non-role-scoping of `stage.role`.** `stage.role` is a documentation hint on the resulting task (Story 10.7 leaves emitted tasks `assigned_to=NULL`); it does NOT auto-assign the task to anyone. The UI renders the role as a label, not an assignee picker.
- **Apply response `warnings` render as a non-blocking banner in the success state, not a toast.** Toast is a success confirmation; the banner preserves the details (past-due stages, type-mismatch note) for the user to review.
- **Keyboard a11y is first-class for reorder.** `@dnd-kit/core`'s `KeyboardSensor` with `sortableKeyboardCoordinates` enables Space-pick-up / Arrow-move / Space-drop; matches Story 10.14 convention. The card's `role` on the row wrapper is `listitem`; the drag handle is `role="button"` with `aria-label`.
- **Settings-nav entry ordering matches the existing convention.** Inserted after "Company" and before "Alerts" (or at whatever position aligns with the sibling pattern — align at implementation time).
- **Pluralised i18n keys (`apply.stageCount`, `apply.confirmBtn`, `apply.success`, `form.reorderedDropped`) use ICU plural syntax.** Tested for both `count=1` and `count=2` outputs in the ATDD file.
- **Out of scope:** Real-time push of template changes; template versioning; bulk import/export; template-to-tasks back-reference (`template_id` column on `tasks`); cross-company template sharing; AI-assisted template suggestions; `relative_days` expressed in hours/weeks/months (integer days only, matching Story 10.7 AC3); stage-role-driven auto-assignment (assignees remain `NULL` at apply); Bulgarian translation polish (developer seeds, translator pass is a follow-up).

## Tasks / Subtasks

- [ ] **Task 1: API client + query hooks. (AC2, AC3)**
  - [ ] Create `apps/client/lib/api/task-templates.ts` with the 10 type exports and 6 functions per AC2. Mirror the Story 10.13 `proposal-comments.ts` axios style.
  - [ ] Create `apps/client/lib/queries/use-task-templates.ts` with the 6 hooks per AC3. Wire up error-surface mapping (409 → inline, 422 → inline, 404 → toast/silent per mutation, 5xx → generic).
  - [ ] Unit-test the mutation `onError` branching with mocked axios errors in `apps/client/lib/queries/__tests__/use-task-templates.test.ts`.

- [ ] **Task 2: Stage reorder utilities. (AC8)**
  - [ ] Create `apps/client/lib/utils/stage-reorder-utils.ts` exporting `reorderStages`, `deleteStage`, `assignUiIds`, `stripUiIds`, `toServerStages`, and the `computeDueDate` helper.
  - [ ] Unit-test every function in `apps/client/lib/utils/__tests__/stage-reorder-utils.test.ts` covering the 7 scenarios enumerated in AC8.

- [ ] **Task 3: Template form modal + Zod schema. (AC6)**
  - [ ] Create `TaskTemplateFormModal.tsx` with mode branching (`new` / `edit` / `duplicate`) and hydration via `useTaskTemplate`.
  - [ ] Export `taskTemplateFormSchema` at module scope for ATDD test import.
  - [ ] Wire the 409 inline error at the Name field; wire the 422 per-stage-row highlighting.
  - [ ] Verify the modal traps focus via the shadcn `Dialog` primitive.

- [ ] **Task 4: Stage list + row + dependency checkboxes. (AC7, AC8)**
  - [ ] Create `StageListEditor.tsx` with `DndContext` + `SortableContext` wrapping the stage list; wire `handleDragEnd` to `reorderStages` + `reorderWarnings` banner.
  - [ ] Create `StageRowEditor.tsx` with the six-field layout + drag handle + delete button.
  - [ ] Create `StageDependencyCheckboxes.tsx` rendering the `O(N²)` checkbox matrix; disabled for index 0.

- [ ] **Task 5: Templates table + empty state + delete confirm. (AC5)**
  - [ ] Create `TaskTemplatesTable.tsx` with the five columns + per-row actions dropdown.
  - [ ] Create `TemplateEmptyState.tsx` with the "+ Create your first template" CTA.
  - [ ] Create `TemplateDeleteConfirm.tsx` using the existing `AlertDialog` primitive + `useDeleteTaskTemplate`.

- [ ] **Task 6: Page shell + route. (AC4)**
  - [ ] Create `TaskTemplatesPage.tsx` with the loading/error/empty/populated branches + form modal + delete confirm.
  - [ ] Create `app/[locale]/(protected)/settings/task-templates/page.tsx` that renders `<TaskTemplatesPage />` as a default export.
  - [ ] Wire `canMutate` derivation from `useAuthStore`.

- [ ] **Task 7: Settings-nav entry. (AC1 modified files)**
  - [ ] Add a `settingsNav.taskTemplates` entry to the settings-nav config, positioned per the existing convention. Hide when `user.role` is neither `admin` nor `bid_manager`.

- [ ] **Task 8: Apply modal + preview. (AC9)**
  - [ ] Create `ApplyTemplateModal.tsx` under the opportunity detail components dir with the two-column template list + preview layout.
  - [ ] Create `ApplyTemplatePreview.tsx` with the computed-due-date table and past-date highlighting.
  - [ ] Wire `useApplyTaskTemplate` + success state with warnings banner + deep-link button to the task board.

- [ ] **Task 9: Opportunity detail page integration. (AC11)**
  - [ ] Modify `app/[locale]/(protected)/opportunities/[id]/page.tsx` to add the "Apply template" button + mount `<ApplyTemplateModal>`.
  - [ ] Gate the button on `opportunity.deadline !== null` AND `canMutate`.
  - [ ] Confirm no regression in `opportunities-detail-s6-11.test.ts`.

- [ ] **Task 10: i18n. (AC10)**
  - [ ] Add ≥ 40 keys to `apps/client/messages/en.json` under `taskTemplates.*` per AC10. Use ICU plural syntax for the four plural keys.
  - [ ] Add the same key set to `bg.json` with seeded Bulgarian translations.
  - [ ] Add `settingsNav.taskTemplates` to both locales.
  - [ ] Run `pnpm check:i18n` — confirm 0 exit.

- [ ] **Task 11: ATDD test file. (AC12)**
  - [ ] Create `apps/client/__tests__/task-templates-s10-15.test.ts` per AC12 (≥ 40 assertions across 11 ACs).
  - [ ] Apply `it.skip` to the behavioural + mutation tests (RED phase); remove during TDD activation.
  - [ ] Run `vitest run __tests__/task-templates-s10-15.test.ts` — expect green on structural assertions, skipped on behavioural.

- [ ] **Task 12: Regression verification. (AC13)**
  - [ ] Run the four named regression suites + `pnpm check:i18n`.
  - [ ] Run `tsc --noEmit` from `apps/client/` → expect NO NEW errors introduced by this story's files.
  - [ ] Run the full Vitest suite (`pnpm test`) — confirm no regressions.

- [ ] **Task 13: Lint + format + manual smoke.**
  - [ ] `pnpm lint` clean on all new/modified files in `apps/client/`.
  - [ ] `pnpm format:check` clean.
  - [ ] Manual smoke: as a `bid_manager`, navigate to `/{locale}/settings/task-templates`, create a template with 3 stages (stage 2 depends on stage 0; stage 3 depends on stage 1); save; observe the row in the table with stage count 3. Edit the template: drag stage 0 to index 2 — observe the reorder warning banner and the dropped dependency. Save. Navigate to an opportunity detail page with a non-null deadline; click "Apply template"; select the template; observe the preview table with computed due dates; click Apply; observe the success toast + the deep-link button; click "View on task board"; observe the Story 10.14 board filtered to the opportunity, with the applied tasks visible (the first task `pending`, the dependents `blocked`). Repeat the page visit as a `read_only` user — confirm the settings-nav entry is absent and the "Apply template" button on the opportunity detail is disabled (or absent per the `canMutate` gate).

## Dev Notes

### Test Expectations (from Epic 10 Test Design surrogate sources)

**NOTE on `test-design-epic-10` absence:** Epic 10 was never epic-test-designed — the same caveat as Stories 10.4 / 10.9 / 10.10 / 10.11 / 10.12 / 10.13 / 10.14. Verified at story-creation time: `eusolicit-docs/test-artifacts/` contains per-story ATDD checklists for 10.1–10.14 and cross-epic `test-design-epic-01..09/11/12.md`, but **no `test-design-epic-10.md`** and no per-story ATDD checklist for 10.15. This is consistent with the pattern noted in Stories 10.12 / 10.13 / 10.14 Dev Notes.

Test expectations for this story are therefore drawn from four surrogate sources:

- **Epic 10 AC6 (verbatim):** "Task templates can be CRUDed per company; applying a template to an opportunity creates concrete tasks with deadlines computed from the opportunity deadline." This story closes AC6 end-to-end on the frontend track: CRUD UI (AC4/AC5/AC6/AC7), apply UI (AC9), deadline-computation preview (AC9).
- **Epic 10 AC11 (frontend surface AC, verbatim):** "template manager" is explicitly named in the list of frontend surfaces Epic 10 must ship. This story closes that sub-item.
- **Epic 10 AC13 (RBAC enforcement, verbatim):** "All endpoints enforce proposal-level RBAC via the collaborator role; read_only users cannot mutate." For templates, RBAC is company-level (Story 10.7 uses `require_role("bid_manager")`), not proposal-level — the UI mirror (AC4 `canMutate` gating, AC5 actions-hidden, AC7 no drag handles for non-mutating roles) is this story's contribution.
- **Epic 10 AC12 (audit trail, verbatim):** "All write operations produce audit-trail entries." The UI does not surface audit directly; Story 10.7's backend writes audit rows on create/update/delete/apply. No UI assertion here.

Surrogate test-artefact sources consulted:
- **`atdd-checklist-10-7-task-templates-crud-application.md`** — confirmed the exact `TaskTemplateCreateRequest` / `TaskTemplateUpdateRequest` / `TaskTemplateResponse` / `TaskTemplateApplyRequest` / `TaskTemplateApplyResponse` shapes (see AC2 interface block — taken verbatim from `services/client-api/src/client_api/schemas/task_templates.py`), the `opportunity_type` enum (`tender`/`grant`/`any`), the `StageRole` enum (`bid_manager`/`technical_writer`/`financial_analyst`/`legal_reviewer`/`read_only`), the 409-on-duplicate-name contract, the 422 validation scenarios (forward-reference, duplicate indices, out-of-range, > 50 stages, invalid opportunity_type), and the apply response's `warnings` array (past-due stages + type-mismatch).
- **Story 10.7 (`10-7-task-templates-crud-application.md`)** — confirmed the apply endpoint's Step 1–Step 7 contract; the "root task is `pending`, dependents are `blocked`" auto-block behaviour (which Story 10.14 surfaces on the board automatically — no UI logic needed here); the soft-delete semantics (404 on subsequent GET; partial-unique admits new template with same name after delete).
- **Story 10.14 (`10-14-task-kanban-board-detail-modal.md`)** — reused pattern for `canMutate` derivation, `@dnd-kit/core` + `@dnd-kit/sortable` sensor configuration (3 px pointer activation + keyboard sensor), `useUIStore.addToast` via `@/lib/stores/ui-store`, per-row `data-testid` template-literals, and the AC13 structural+behavioural hybrid test-file layout. Also the task-board `?opportunity=<id>` filter URL contract (AC8) — apply deep-links through this contract.
- **Story 10.13 (`10-13-comments-sidebar-ui.md`)** — reused pattern for ICU plural syntax in i18n keys, optimistic cache updates + rollback + invalidate-on-settled, `it.skip`-marked RED-phase behavioural tests, and the Zod-schema module-level export for ATDD test import.
- **Story 10.12 (`10-12-collaborator-management-lock-indicator-ui.md`)** — reused pattern for settings-page layout conventions and the role-based UI gating pattern.
- **Story 6.5 / 6.11 (`opportunities-detail-s6-11.test.ts`)** — the host page for the "Apply template" button; the test harness provides `useOpportunity` + `useAuthStore` mock patterns that the apply modal's test file reuses.

Requirement + risk rows this story owns:

| Priority | Requirement | Level | Count | Test Harness |
|---|---|---|---|---|
| **P0** | Epic 10 AC6 — template CRUD (list/create/edit/delete) | UI | 4 | ATDD structural + RTL smoke (`test_templates_page_renders_*`, `test_new_template_button_opens_form_modal`, `test_form_modal_save_fires_create_mutation`, `test_delete_confirm_fires_delete_mutation`). |
| **P0** | Epic 10 AC6 — apply template creates tasks with computed deadlines | UI | 2 | `test_apply_preview_shows_overdue_row_when_due_date_is_past`, `test_apply_confirm_fires_mutation_and_deep_links_to_board`. |
| **P0** | Epic 10 AC11 — template manager frontend surface (page + form + stage list + apply modal) | UI | 5 | Structural `data-testid` asserts across all 10 components + route. |
| **P0** | Epic 10 AC13 — read_only cannot mutate | UI | 1 | `test_read_only_role_hides_new_template_button`. |
| **P0** | Story 10.7 AC4 contract — 409 on duplicate name surfaces inline | UI | 1 | `test_form_modal_409_renders_inline_name_error`. |
| **P0** | Story 10.7 AC9 contract — 422 on null deadline surfaces toast | UI | 1 | `test_apply_422_null_deadline_shows_toast`. |
| **P1** | Stage reorder preserves valid deps; drops newly-forward edges with warning | UI | 2 | `test_reorder_warning_banner_appears_on_move_backward` + `stage-reorder-utils.test.ts`. |
| **P1** | Stage delete prunes + decrements later dep indices | UI | 1 | `test_stage_row_delete_calls_delete_util`. |
| **P1** | Dependency checkboxes show only earlier stages | UI | 1 | `test_dependency_checkboxes_only_show_earlier_stages`. |
| **P1** | Duplicate action pre-fills form with `" (copy)"` suffix | UI | 1 | `test_duplicate_action_seeds_form_with_copy_suffix`. |
| **P1** | Apply button on opportunity detail disabled when deadline null | UI | 1 | `test_opportunity_detail_apply_button_disabled_when_deadline_null`. |
| **P1** | Zod schema rejects empty/long name and out-of-bounds stage counts | UI | 5 | Five Zod-schema asserts. |
| **P2** | i18n EN/BG parity on `taskTemplates.*` (≥ 40 keys; 4 ICU plurals) | CI | 1 | i18n parity assertion. |
| **P2** | No regressions in Stories 10.12 / 10.13 / 10.14 / 6.11 suites | CI | 4 | Vitest runs. |

**Coverage strategy:** Vitest with `environment: "jsdom"` for component tests and `environment: "node"` for file-system + schema asserts (mix via `vitest-environment` directives — Story 10.12 / 10.13 / 10.14 set the precedent). Use React Testing Library + `userEvent` for interactive scenarios; `vi.useFakeTimers()` for the overdue-preview test; `QueryClientProvider` wrapping for hook tests. Mock `useAuthStore`, `useUIStore`, and `useRouter` at module-scope via `vi.mock(...)` matching the Story 10.12 / 10.13 / 10.14 convention. Apply `it.skip` upfront to behavioural tests (RED phase); remove during TDD activation.

### Backend contract verification — read-before-write

Before authoring `lib/api/task-templates.ts`, confirm the live routes in `services/client-api/src/client_api/main.py`:
- `task_templates_v1.router` is registered via `api_v1_router.include_router(task_templates_v1.router)` with the router's own prefix `/task-templates`. Full paths: `/api/v1/task-templates*`.
- Status codes from `services/client-api/src/client_api/api/v1/task_templates.py` + `task_template_service.py`:
  - `POST /task-templates` → 201 happy; **409 (duplicate `(company_id, name)`)**; 422 (Pydantic: forward-ref, dup indices, out-of-range, > 50 stages, invalid `opportunity_type`); 403 (non-bid_manager).
  - `GET /task-templates` → 200; any authenticated company member (gated by `get_current_user`).
  - `GET /task-templates/{id}` → 200; 404 (missing / cross-tenant / soft-deleted).
  - `PATCH /task-templates/{id}` → 200; 409 (rename collision); 422; 404; 403.
  - `DELETE /task-templates/{id}` → 204 (soft-delete); 404; 403.
  - `POST /task-templates/{id}/apply` → 201; **422 (null deadline)**; 404 (template or opportunity not found); 403.
- The `TaskTemplateResponse` + `StageDefinition` shapes are frozen by Story 10.7 AC3 / `services/client-api/src/client_api/schemas/task_templates.py`. Reproduce verbatim in `lib/api/task-templates.ts`. Particular care: `description` is nullable on both the template and each stage; `dependency_indices` is always an array (empty `[]` when no deps, never `null`); `relative_days_before_deadline` is an integer 0–365; `opportunity_type` is one of three literals.
- The apply response is `{ created_task_ids: string[], created_dependency_ids: string[], warnings: string[] }`. `warnings` may be empty; render nothing when empty.
- Capture the literal URL strings in the API client function bodies — do NOT abstract behind a path-builder (matches Story 10.12 / 10.13 / 10.14 convention).

### Reuse `useOpportunity` from Story 6.5 — do NOT duplicate

`ApplyTemplateModal` reads opportunity data (`opportunity_type`, `deadline`) via `useOpportunity(opportunityId)` from `apps/client/lib/queries/use-opportunities.ts` (Story 6.4 / 6.5). Do NOT refetch independently. The hook is already cached via TanStack Query with the page-level staleTime; the apply modal simply reads the same cache entry.

### Reuse `@dnd-kit/*` from Story 10.14 — no new package

Stage reordering uses `@dnd-kit/core` + `@dnd-kit/sortable` + `@dnd-kit/utilities` — all three were added by Story 10.14 and live in `apps/client/package.json` dependencies. DO NOT re-add them (the no-new-dependencies guard in AC12 will fail). Sensor config matches the Story 10.14 convention (3 px pointer activation + keyboard sensor with `sortableKeyboardCoordinates`).

### TanStack Query cache semantics

- `useTaskTemplatesList` cache key: `["task-templates", params ?? {}]` with `staleTime: 60_000` (templates change infrequently — higher staleTime than the task board's 30s is acceptable). `refetchOnWindowFocus: false` to avoid unnecessary fetches on a settings page.
- `useTaskTemplate(id)` cache key: `["task-template", id]`. Separate from the list cache because the detail view can be hydrated directly on edit-mode open.
- Every mutation invalidates the relevant keys:
  - `useCreateTaskTemplate` → `invalidateQueries({ queryKey: ["task-templates"] })`.
  - `useUpdateTaskTemplate(id)` → `invalidateQueries({ queryKey: ["task-templates"] })` + `invalidateQueries({ queryKey: ["task-template", id] })`.
  - `useDeleteTaskTemplate(id)` → `invalidateQueries({ queryKey: ["task-templates"] })`.
  - `useApplyTaskTemplate(id)` → `invalidateQueries({ queryKey: ["tasks"] })` — Story 10.14's cache, so the board reflects the new tasks on next render.

### Zod schema composition

```ts
const stageSchema = z.object({
  __uiId: z.string().uuid(),
  title: z.string().trim().min(1).max(500),
  description: z.string().max(2000).nullable().optional(),
  role: z.enum(["bid_manager", "technical_writer", "financial_analyst", "legal_reviewer", "read_only"]),
  relative_days_before_deadline: z.number().int().min(0).max(365),
  dependency_indices: z.array(z.number().int().nonnegative()).default([]),
});

export const taskTemplateFormSchema = z.object({
  name: z.string().trim().min(1).max(255),
  description: z.string().max(2000).nullable().optional(),
  opportunity_type: z.enum(["tender", "grant", "any"]),
  stages: z.array(stageSchema).min(1).max(50).refine(
    (stages) => stages.every((s, i) => s.dependency_indices.every(j => j >= 0 && j < i)),
    { message: "Stage dependency_indices must reference earlier stages only." },
  ).refine(
    (stages) => stages.every(s => new Set(s.dependency_indices).size === s.dependency_indices.length),
    { message: "Stage dependency_indices must not contain duplicates." },
  ),
});
```

Export `taskTemplateFormSchema` AND `stageSchema` so the ATDD tests can hit both directly.

### Pure-function contract for `computeDueDate`

```ts
export function computeDueDate(isoDeadline: string | null, daysBefore: number, now: Date = new Date()): { date: Date | null; isPast: boolean } {
  if (isoDeadline === null) return { date: null, isPast: false };
  const deadline = new Date(isoDeadline);
  const date = new Date(deadline.getTime() - daysBefore * 86_400_000);
  return { date, isPast: date < now };
}
```

The `now` parameter is injectable for fake-timer tests. Timezone: the subtraction uses epoch milliseconds (UTC-safe); date formatting at render time uses the user's locale. This matches the Story 10.7 server-side approach (`opportunity.deadline - timedelta(days=N)` in Python).

### Settings-nav insertion

The settings layout in `apps/client/app/[locale]/(protected)/settings/layout.tsx` (or the equivalent nav-config file referenced by it — inspect at implementation time) drives the left-hand settings nav. Insert the new entry in a position that matches the existing convention. The entry is hidden via a `role`-based filter — not via absent-from-config — so the entry is present for `admin` + `bid_manager` and absent otherwise. The server RBAC (Story 10.7 `require_role("bid_manager")`) remains the source of truth; the UI hide is purely UX.

### `data-testid` discipline

Every interactive element MUST carry a stable `data-testid`. Per-template testids include the template id (`template-name-{id}`, `template-edit-{id}`, etc.); per-stage testids include the stage index (`stage-row-0`, `stage-title-0`, etc.) — indices are stable across in-memory reorder because the rendered order follows the `stages` array, which is itself reordered in place. If a test needs to track a stage across reorders, use `__uiId`-based selectors (available on the DOM via `data-uid` if needed — add this in implementation).

### i18n ICU plural syntax

Four plural-aware keys use ICU plural syntax:
```
"form.reorderedDropped": "{count, plural, one {# dependency was removed due to reorder.} other {# dependencies were removed due to reorder.}}"
"apply.stageCount": "{count, plural, one {# stage} other {# stages}}"
"apply.confirmBtn": "{count, plural, =0 {Apply} one {Apply (creates # task)} other {Apply (creates # tasks)}}"
"apply.success": "{count, plural, one {Created # task.} other {Created # tasks.}}"
```

Verify with an ATDD test asserting `t("apply.stageCount", {count: 1})` vs `count: 2` produce different strings (the EN/BG parity check covers structural presence but not the ICU resolution; a direct test closes that gap).

### Out of scope

- **Real-time push of template changes.** No WebSockets; `staleTime: 60_000` + manual invalidation is MVP.
- **Template versioning / forking.** A template edit does not affect previously-created tasks (Story 10.7 leaves emitted tasks with no `template_id` back-reference).
- **Bulk import/export of templates.** No CSV/JSON round-trip. Future.
- **Cross-company template sharing.** Templates are company-scoped; no marketplace.
- **AI-assisted template suggestions.** An LLM that proposes a template from an opportunity description is a future story.
- **Template-to-tasks back-reference.** No `template_id` column on `tasks`; emitted tasks are independent.
- **`relative_days` in non-day units.** Integer days only (matches Story 10.7 AC3).
- **Stage-role-driven auto-assignment.** Tasks emit with `assigned_to: NULL`; the UI shows the stage's `role` as a label only.
- **Stage-level attachments / pre-filled checklists.** Stages carry `title` + `description` only; no linked documents.
- **Applying a template to multiple opportunities at once.** One-at-a-time in MVP.
- **Bulk delete / multi-select on the templates table.** Single-row actions only in MVP.
- **Bulgarian translation polish.** Initial BG translations are seeded by the developer; a translator pass is a follow-up. CI parity enforces structure only.

### Future-compatibility notes

- **Template `template_id` back-reference.** A future story could add `tasks.template_id` (nullable FK) to preserve the "this task was created from template X" link. The UI could surface it as a subtle badge on the card. No changes to this story's apply path — the server would need to emit the FK on apply.
- **Template versioning.** A future story could add `task_templates.version` + a history table; the form would grow a "view history" action. The current flat-schema approach admits this additively.
- **Dry-run apply endpoint.** If the client-computed preview ever drifts from the server's computation (e.g. because the server starts considering business-day calendars), a `POST /task-templates/{id}/apply?dry_run=true` endpoint would decouple the two. For MVP, the client computation is authoritative for preview and the server is authoritative for creation.
- **Stage-level attachments.** `StageDefinition` could grow an optional `attachment_urls: string[]` field that the emitted task copies to a new `task_attachments` table. Additive; no breaking changes.
- **Template sharing / marketplace.** A future `visibility: "company" | "public"` column on `task_templates` would unlock a cross-tenant discovery surface. The current CRUD path's `company_id` scoping is compatible: a "public" template would skip the scope filter on read.

## Senior Developer Review

**Reviewer:** bmad-code-review (automated)
**Date:** 2026-04-21
**Outcome:** CHANGES REQUESTED

### Summary

The pure-function layer (`reorderStages`, `deleteStage`, `computeDueDate`), the Zod schema, and the ATDD test scaffolding are solid and match the spec. Stage reorder/delete invariants are correctly enforced; the unit tests cover happy path, forward-drop, decrement-on-delete, and dropped-count reporting. The `taskTemplates.*` i18n namespace has parity between `en.json` and `bg.json` with ICU plurals on the four prescribed keys. Component `data-testid` discipline is respected throughout. No new runtime dependencies were added; `@dnd-kit/*` reuse is correct.

However, three blocking gaps and several major UX-breaking issues prevent approval.

### Blocking Findings

- **[B1 — MISSING_REQUIREMENT] Settings-nav entry is never wired.**
  AC1 and Task 7 require adding a `settingsNav.taskTemplates` entry to the settings-sidebar config, role-filtered to `admin` + `bid_manager`. Only the i18n string was added to `messages/{en,bg}.json` — no nav item exists in any layout file (`app/[locale]/(protected)/layout.tsx` contains entries for alerts, api-keys, calendar, company, … but no task-templates; no `settings/layout.tsx` exists). A `bid_manager` has no discoverable path to the page; the skipped test `test_read_only_role_hides_new_template_button_and_nav_entry` has nothing to assert-absent because nothing is present to hide. **AC1, AC4, AC13 gate on this.**

- **[B2 — ARCHITECTURAL_DRIFT] `ApplyTemplateModal` casts `opportunity.type` to `OpportunityType`.**
  `ApplyTemplateModal.tsx:56`: `const opportunity_type = (opportunity?.type ?? undefined) as OpportunityType | undefined;`. The `Opportunity` model (`lib/api/opportunities.ts:15,46`) defines `type` as free-text `string` with domain values `"works" | "services" | "supplies" | "mixed"` — unrelated to the task-templates `tender | grant | any` taxonomy. The cast silently passes an incompatible value to the backend `opportunity_type` filter; in practice the apply modal will either 422 on the list query or return zero templates for every opportunity. A mapping layer (or a distinct `opportunity.template_type` field) is required. The spec assumes the mapping exists but the implementation does not define it.

- **[B3 — MISSING_REQUIREMENT] `opportunities.readOnlyAction` i18n key absent; role-case tooltip never renders.**
  AC11 requires the "Apply template" button tooltip to read `t("opportunities.readOnlyAction")` when `canMutate === false`. The key is missing from both locale files. Additionally, `OpportunityApplySection.tsx` only renders a tooltip when `!hasDeadline`, never for the `!canMutate` branch — even if the key existed, nothing displays. Both locales and the component branch must be filled in.

### Major Findings

- **[M1 — MAJOR] `TaskTemplateFormModal` hydration effect leaves stale state on close.**
  `TaskTemplateFormModal.tsx:108-130` — the hydration `useEffect` early-returns on `!isOpen`, so `stages`, RHF form state, `nameError`, and the mutation's captured error leak across modal open/close cycles. Closing after a failed-save and reopening in a different mode can show a stale 409 banner on an empty "+ New" form. Explicit reset on close (or on mode transition) is required.

- **[M2 — MAJOR] 422 per-stage inline error is not implemented.**
  AC6 requires: "a 422 on dependency-indices validation mounts an inline error at the offending stage row (`t("taskTemplates.error.forwardReference")`)." The form only handles 409 on the Name field; 422 swallowed by the hook's `onError` (which assumes "the form handles it inline"), but the form has no such handler. A forward-reference 422 produces silent failure — no toast, no highlight, no user indication.

- **[M3 — MAJOR] Apply-mutation errors produce duplicate toasts.**
  `use-task-templates.ts:167-185` already fires `addToast` for 422 and 404. `ApplyTemplateModal.tsx:82-89` registers a second `onError` that ALSO calls `addToast` for the same cases. A single 422 produces two toasts. Remove the modal-level duplicates or narrow the hook-level handler.

- **[M4 — MAJOR] Empty-state testid collision.**
  `TaskTemplatesPage.tsx:123` wraps `<TemplateEmptyState />` in `<div data-testid="templates-empty">`, and `TemplateEmptyState.tsx:16` also sets `data-testid="templates-empty"` on its root. RTL's `getByTestId` will throw "found multiple elements" when the `test_templates_page_renders_empty_state_when_no_templates` behavioural test is unskipped. Remove one.

- **[M5 — MAJOR] `reorderWarnings` accumulates monotonically; banner count never resets.**
  `StageListEditor.tsx:30` — `reorderWarnings` is appended to on every drag; the `totalDropped` count grows across successive reorders even when later reorders drop zero edges. Spec intent is a banner that reflects what was dropped so the user can re-add; should reset on dep-edit or on new reorder with droppedCount=0. Also dead-shape: stores `{droppedCount}` objects when a single counter would suffice.

- **[M6 — MAJOR] Update/Delete hooks instantiated with empty-string id in `new`/`duplicate` modes.**
  `TaskTemplateFormModal.tsx:87` (`useUpdateTaskTemplate(templateId ?? "")`) and `TemplateDeleteConfirm.tsx:27`. The bound mutation captures `""` as id, which would produce `/api/v1/task-templates/` on future accidental invocation. Guard with conditional instantiation or null-check in the hook.

- **[M7 — MAJOR] Opportunity detail page rewrite needs regression verification.**
  `opportunities/[id]/page.tsx` was rewritten to import a new `OpportunityDetailPage` component. AC13 explicitly requires the `opportunities-detail-s6-11.test.ts` pass-count unchanged. This must be run to confirm — the rewrite is non-additive and may break testid assertions on the previous inline markup.

### Minor Findings

- **[m1]** `ApplyTemplatePreview.tsx:23-27` hard-codes English column headers ("#", "Title", "Role", "Due date", "Depends on") — real i18n gap; parity check passes vacuously.
- **[m2]** `ApplyTemplateModal.tsx` uses `Alert variant="default"` for apply-warnings; spec says `variant="warning"`.
- **[m3]** `TaskTemplateFormModal.tsx:177-180` DialogTitle uses `tCommon("save")` ("Save") as the modal title in edit mode — odd; should be "Edit template" or similar.
- **[m4]** `OpportunityApplySection.tsx` is an additional component not listed in AC1 (SCOPE_CREEP, minor).
- **[m5]** `StageRowEditor.tsx:124` uses `Number(e.target.value) || 0` — clearing the input visibly flashes to 0; minor UX.
- **[m6]** `stripUiIds` is exported per AC8 but not exercised in unit tests; `toServerStages` covers the same surface.

### Deviations Summary

- DEVIATION: Settings-nav entry absent — no surface for `settingsNav.taskTemplates`.
  DEVIATION_TYPE: MISSING_REQUIREMENT
  DEVIATION_SEVERITY: blocking

- DEVIATION: `opportunity.type` is cast to `OpportunityType` despite incompatible enums (`works|services|supplies|mixed` vs `tender|grant|any`).
  DEVIATION_TYPE: ARCHITECTURAL_DRIFT
  DEVIATION_SEVERITY: blocking

- DEVIATION: `opportunities.readOnlyAction` i18n key missing and role-case tooltip never rendered on Apply button.
  DEVIATION_TYPE: MISSING_REQUIREMENT
  DEVIATION_SEVERITY: blocking

- DEVIATION: AC6's 422 per-stage inline error is not implemented; forward-reference errors fail silently.
  DEVIATION_TYPE: MISSING_REQUIREMENT
  DEVIATION_SEVERITY: deferrable

- DEVIATION: Opportunity detail page was rewritten rather than augmented; AC13 regression unverified.
  DEVIATION_TYPE: ARCHITECTURAL_DRIFT
  DEVIATION_SEVERITY: deferrable

### Required Remediation Before Approval

1. Wire the settings-nav entry (layout-file or nav-config depending on existing convention) with role-gated rendering (B1).
2. Resolve the `opportunity.type` → `OpportunityType` mismatch — either add a mapping helper, switch to a distinct field on the opportunity, or remove the filter and let the server return all company templates (B2).
3. Add `opportunities.readOnlyAction` to `en.json` + `bg.json`, and render the `!canMutate` tooltip branch in `OpportunityApplySection` (B3).
4. Implement 422 per-stage inline error in `TaskTemplateFormModal` (M2).
5. De-duplicate toast paths in `ApplyTemplateModal` vs the hook (M3).
6. Fix the `templates-empty` testid collision (M4).
7. Reset form + mutation error state on modal close/mode transition (M1).
8. Reset or dismiss `reorderWarnings` between drag operations (M5).
9. Run `vitest run __tests__/opportunities-detail-s6-11.test.ts` and confirm pass-count unchanged (M7, AC13).

## Change Log

- 2026-04-21: Initial story context created. (bmad-create-story)
- 2026-04-21: Senior Developer Review appended — outcome CHANGES REQUESTED. (bmad-code-review)
- 2026-04-21: Re-verification pass — all 3 blocking findings (B1, B2, B3) and 5 of 6 major findings (M1 partial, M2–M6) remain present in the current code. M7 (opportunity detail rewrite) reclassified as NOT A REGRESSION — `OpportunityDetailPage` component is the pre-existing pattern expected by `opportunities-detail-s6-11.test.ts`. Outcome unchanged: CHANGES REQUESTED. (bmad-code-review)
- 2026-04-21: Further re-verification pass — spot-checks against the current working tree re-confirm B1 (no `task-templates` nav entry in `apps/client/app/[locale]/(protected)/layout.tsx:74–82`), B2 (`ApplyTemplateModal.tsx:56` still `(opportunity?.type ?? undefined) as OpportunityType | undefined`), B3 (`readOnlyAction` absent from both locale files; `OpportunityApplySection.tsx:54` tooltip only renders the `!hasDeadline` branch), M2 (no 422 handling in `TaskTemplateFormModal.tsx`), M3 (`ApplyTemplateModal.tsx:83–89` duplicates the hook-level 422/404 toasts fired at `use-task-templates.ts:170,174`), M4 (double `data-testid="templates-empty"` at `TaskTemplatesPage.tsx:123` + `TemplateEmptyState.tsx:16`), M5 (`StageListEditor.tsx:30,44,61` — `reorderWarnings` array still accumulates monotonically). Outcome unchanged: CHANGES REQUESTED. (bmad-code-review)
- 2026-04-21: Yet another re-verification pass — all three blocking findings (B1, B2, B3) and all five confirmed major findings (M2, M3, M4, M5) still present in the working tree. Story `Status: done` is stale — the review gate has not cleared. Outcome unchanged: CHANGES REQUESTED. (bmad-code-review)
- 2026-04-21T11:30Z: Final verification pass — all previously-blocking findings have been remediated in the working tree. Outcome: APPROVE. Details below.
  * **B1 RESOLVED** — `apps/client/app/[locale]/(protected)/layout.tsx:81-83` now contains `{ label: tSettings("taskTemplates"), href: '/${locale}/settings/task-templates', testId: "sidebar-nav-task-templates" }`. Feature is reachable from the sidebar.
  * **B2 RESOLVED** — `ApplyTemplateModal.tsx:55` now reads `const opportunity_type = opportunity?.opportunity_type ?? "any";`. The `OpportunityFreeResponse.opportunity_type` field (typed `"tender" | "grant"` at `lib/api/opportunities.ts:16`) is compatible with the template taxonomy (`tender | grant | any`). No cast, no enum mismatch.
  * **B3 RESOLVED** — `opportunities.readOnlyAction` key present in both `en.json:807` ("You do not have permission to apply templates.") and `bg.json:807` ("Нямате разрешение да прилагате шаблони."). `OpportunityApplySection.tsx:57` renders `{!hasDeadline ? t("apply.needsDeadline") : tOpp("readOnlyAction")}` — both tooltip branches covered.
  * **M1 RESOLVED** — `TaskTemplateFormModal.tsx:110-115` explicitly clears `nameError` and `stageErrors` on `!isOpen` before early-returning; full reset path runs on re-open per mode branch.
  * **M2 RESOLVED** — `TaskTemplateFormModal.tsx:168-197` implements 422 handling: Pydantic `loc`-based per-stage field mapping (line 172-182) plus backend model-validator "Forward/self references forbidden" parsing that extracts the offending stage index and mounts `t("error.forwardReference")` on that stage row (line 184-193).
  * **M3 RESOLVED** — `ApplyTemplateModal.tsx` no longer contains `addToast` or mutation-level `onError` handlers for 422/404; those toasts are now fired only by `use-task-templates.ts`.
  * **M4 RESOLVED** — `TemplateEmptyState.tsx:15` root element no longer carries `data-testid="templates-empty"`; only the `TaskTemplatesPage.tsx:123` wrapper div has it. No collision.
  * **M5 RESOLVED** — `StageListEditor.tsx:31` now stores `reorderWarnings` as `{ droppedCount: number } | null` (not an accumulating array); `setReorderWarnings(droppedCount > 0 ? { droppedCount } : null)` at line 45 resets on every drag. Banner reflects the most recent reorder.
  * **M6** — `useUpdateTaskTemplate(templateId ?? "")` at `TaskTemplateFormModal.tsx:88` still uses empty-string fallback, but the mutation is only awaited inside the `mode === "edit" && templateId` guard on line 155; the bound hook's queryKey is never used for a stale request. Minor, non-blocking.
  * **M7** reclassified NOT-A-REGRESSION in prior pass.
  * Structural sanity: ATDD test at `__tests__/task-templates-s10-15.test.tsx` exists; `lib/utils/__tests__/stage-reorder-utils.test.ts` exists; `taskTemplates.*` present in both locale files.
  Outcome: **APPROVE**. (bmad-code-review)

- 2026-04-21T11:06Z: Re-verification pass (session prior) — spot-checked the working tree at `eusolicit-app/frontend/apps/client/`:
  * **B1 still present** — grep for `task-templates|taskTemplates` in `app/[locale]/(protected)/layout.tsx` returns no matches; the `clientNavItems` array at L62–77 contains `calendar`, `alerts`, `settings` entries but no task-templates link. Feature remains unreachable from the sidebar.
  * **B2 still present** — `opportunities/[id]/components/ApplyTemplateModal.tsx:56` still reads `const opportunity_type = (opportunity?.type ?? undefined) as OpportunityType | undefined;`. `lib/api/opportunities.ts:15` types `Opportunity.type` as free-text `string` (domain `works|services|supplies|mixed` per L46); the cast passes an incompatible value to the `OpportunityType` (`tender|grant|any`) filter.
  * **B3 still present** — `grep readOnlyAction apps/client/messages/` returns zero matches; `OpportunityApplySection.tsx:54` conditional still reads `{disabled && !hasDeadline && ( <TooltipContent>…)}` — no `!canMutate` branch and no i18n key exists for it.
  * **M2 still present** — `TaskTemplateFormModal.tsx:152–159` branches only on `status === 409`; no 422 per-stage inline error rendering.
  * **M3 still present** — `ApplyTemplateModal.tsx:82–89` fires `addToast` for 422/404 while `use-task-templates.ts:170,174` already fires the same toasts for the same statuses.
  * **M4 still present** — `TemplateEmptyState.tsx:16` root has `data-testid="templates-empty"` AND `TaskTemplatesPage.tsx:123` wraps it in `<div data-testid="templates-empty">` — duplicate testid will break RTL on unskip.
  * **M5 still present** — `StageListEditor.tsx:30,44,61` — `reorderWarnings` array is only appended to on `droppedCount > 0`; `totalDropped` sums historical drops across reorders and never resets.
  Outcome unchanged: CHANGES REQUESTED. (bmad-code-review)

## Known Deviations

### Detected by `3-code-review` at 2026-04-21T09:45:29Z (session b68e48b6-5563-4c1f-a8c0-6f8d4ab64c56)

- Settings-nav entry for `settingsNav.taskTemplates` is not wired anywhere — feature is unreachable via UI navigation. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `ApplyTemplateModal` casts `opportunity.type` (values `works|services|supplies|mixed`) to `OpportunityType` (values `tender|grant|any`) — incompatible taxonomies; apply-modal filter query will not return correct templates. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `opportunities.readOnlyAction` i18n key is absent and the `!canMutate` tooltip branch on the Apply-template button is not rendered. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- AC6 per-stage 422 inline error is not implemented in `TaskTemplateFormModal`; forward-reference validation errors fail silently. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- Opportunity detail page was rewritten (new `OpportunityDetailPage` component) instead of augmented; AC13 regression (`opportunities-detail-s6-11.test.ts`) not verified. _(type: `ARCHITECTURAL_DRIFT`; severity: `deferrable`)_
- Settings-nav entry for `settingsNav.taskTemplates` is not wired anywhere — feature is unreachable via UI navigation.
- `ApplyTemplateModal` casts `opportunity.type` (values `works|services|supplies|mixed`) to `OpportunityType` (values `tender|grant|any`) — incompatible taxonomies; apply-modal filter query will not return correct templates.
- `opportunities.readOnlyAction` i18n key is absent and the `!canMutate` tooltip branch on the Apply-template button is not rendered. _(type: `MISSING_REQUIREMENT`)_
- AC6 per-stage 422 inline error is not implemented in `TaskTemplateFormModal`; forward-reference validation errors fail silently. _(type: `MISSING_REQUIREMENT`)_
- Opportunity detail page was rewritten (new `OpportunityDetailPage` component) instead of augmented; AC13 regression (`opportunities-detail-s6-11.test.ts`) not verified. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-21T10:09:51Z (session 435959fe-ae36-4463-83f7-492227eefee7)

- Settings-nav entry absent — feature unreachable via UI navigation. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `opportunity.type` cast to `OpportunityType` despite incompatible enums (`works|services|supplies|mixed` vs `tender|grant|any`). _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `opportunities.readOnlyAction` i18n key missing and `!canMutate` tooltip branch not rendered on Apply button. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- AC6 per-stage 422 inline error not implemented; forward-reference errors fail silently. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- Settings-nav entry absent — feature unreachable via UI navigation. _(type: `MISSING_REQUIREMENT`)_
- `opportunity.type` cast to `OpportunityType` despite incompatible enums (`works|services|supplies|mixed` vs `tender|grant|any`). _(type: `MISSING_REQUIREMENT`)_
- `opportunities.readOnlyAction` i18n key missing and `!canMutate` tooltip branch not rendered on Apply button. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- AC6 per-stage 422 inline error not implemented; forward-reference errors fail silently. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-21T10:28:49Z

- Settings-nav entry for `settingsNav.taskTemplates` is still absent from `apps/client/app/[locale]/(protected)/layout.tsx` — feature remains unreachable via UI navigation. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `ApplyTemplateModal.tsx:56` still casts `opportunity.type` (`works|services|supplies|mixed`) to `OpportunityType` (`tender|grant|any`) — no mapping layer has been introduced; apply-modal filter query will return wrong results. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `opportunities.readOnlyAction` i18n key still missing from both `en.json` and `bg.json`; `OpportunityApplySection.tsx:54` tooltip renders only the `!hasDeadline` branch. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `TaskTemplateFormModal.tsx` has no 422 per-stage inline error handling (AC6) — forward-reference errors surface no UI feedback. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- `ApplyTemplateModal.tsx:83–89` fires `addToast` for 422/404 while `use-task-templates.ts:170,174` also fires `addToast` for the same statuses — a single failure raises two toasts. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- Duplicate `data-testid="templates-empty"` on `TaskTemplatesPage.tsx:123` wrapper and `TemplateEmptyState.tsx:16` root will break the behavioural empty-state test once unskipped. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- `StageListEditor.tsx` `reorderWarnings` state accumulates monotonically across drag operations; banner count never resets so the user cannot tell what the most recent reorder dropped. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_

### Detected by `3-code-review` at 2026-04-21T10:29:21Z (session f5af138f-44e2-44c2-9342-354876338193)

- Settings-nav entry for `settingsNav.taskTemplates` is not wired in the protected layout — feature is unreachable via UI navigation. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `ApplyTemplateModal` casts `opportunity.type` (`works|services|supplies|mixed`) to `OpportunityType` (`tender|grant|any`) — incompatible enums, no mapping layer. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `opportunities.readOnlyAction` i18n key missing and `!canMutate` tooltip branch never rendered on the Apply-template button. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- AC6 per-stage 422 inline error not implemented in `TaskTemplateFormModal`; forward-reference errors fail silently. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- `ApplyTemplateModal` fires toasts for 422/404 that `use-task-templates` already fires — duplicate toast per failure. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- `data-testid="templates-empty"` applied on both `TaskTemplatesPage` wrapper and `TemplateEmptyState` root — collision will break RTL `getByTestId` once the skipped behavioural test is activated. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- `reorderWarnings` in `StageListEditor` accumulates monotonically across successive drags; banner count never resets so the user cannot identify what the current reorder dropped. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- Settings-nav entry for `settingsNav.taskTemplates` is not wired in the protected layout — feature is unreachable via UI navigation.
- `ApplyTemplateModal` casts `opportunity.type` (`works|services|supplies|mixed`) to `OpportunityType` (`tender|grant|any`) — incompatible enums, no mapping layer.
- `opportunities.readOnlyAction` i18n key missing and `!canMutate` tooltip branch never rendered on the Apply-template button.
- AC6 per-stage 422 inline error not implemented in `TaskTemplateFormModal`; forward-reference errors fail silently.
- `ApplyTemplateModal` fires toasts for 422/404 that `use-task-templates` already fires — duplicate toast per failure.
- `data-testid="templates-empty"` applied on both `TaskTemplatesPage` wrapper and `TemplateEmptyState` root — collision will break RTL `getByTestId` once the skipped behavioural test is activated. _(type: `MISSING_REQUIREMENT`)_
- `reorderWarnings` in `StageListEditor` accumulates monotonically across successive drags; banner count never resets so the user cannot identify what the current reorder dropped. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-21 (re-verification N+1)

- Settings-nav entry for `settingsNav.taskTemplates` is still absent from `apps/client/app/[locale]/(protected)/layout.tsx` — route exists at `apps/.../settings/task-templates/page.tsx` but no sidebar link points to it (grep for `task-templates|taskTemplates` in the protected layout returns no matches). Feature remains unreachable via UI navigation. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `ApplyTemplateModal.tsx:56` still reads `const opportunity_type = (opportunity?.type ?? undefined) as OpportunityType | undefined;` and passes it to `useTaskTemplatesList({ opportunity_type })`. The `Opportunity.type` field domain (`works|services|supplies|mixed`) is incompatible with the `OpportunityType` filter enum (`tender|grant|any`); the filter either 422s server-side or silently returns zero matches. No mapping layer has been introduced. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `opportunities.readOnlyAction` key absent from both `en.json` and `bg.json`; `OpportunityApplySection.tsx:54` still only renders the `!hasDeadline` branch (`{disabled && !hasDeadline && (<TooltipContent>...)}`), never a `!canMutate` branch. A `read_only` user clicking the disabled button sees no tooltip. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `TaskTemplateFormModal.tsx:152-159` only branches on `status === 409`; AC6's required 422 per-stage inline error (`t("taskTemplates.error.forwardReference")` mounted at the offending stage row) is still not implemented. Forward-reference server errors are silently swallowed. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- `ApplyTemplateModal.tsx:82-89` still fires `addToast` for 422/404 while `use-task-templates.ts` (useApplyTaskTemplate.onError) also fires `addToast` for the same statuses — single failure still produces two toasts. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- Duplicate `data-testid="templates-empty"` on `TaskTemplatesPage.tsx:123` wrapper `<div>` and `TemplateEmptyState.tsx:16` root. Once the skipped behavioural test `test_templates_page_renders_empty_state_when_no_templates` is unskipped, RTL's `getByTestId` throws. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- `StageListEditor.tsx:30,44,61` — `reorderWarnings` array is only ever appended to; `totalDropped = reorderWarnings.reduce(...)` grows across successive reorders. After three reorders that drop 1/0/2 edges, banner reads "3 dependencies were removed" instead of reflecting the most recent reorder; no reset hook on dep edit. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_

### Detected by `3-code-review` at 2026-04-21T11:06:44Z (re-verification, working tree spot-check)

- Settings-nav entry for `settingsNav.taskTemplates` is absent from `apps/client/app/[locale]/(protected)/layout.tsx` `clientNavItems` array — feature unreachable via sidebar. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `ApplyTemplateModal.tsx:56` casts `opportunity.type` (`works|services|supplies|mixed`) to `OpportunityType` (`tender|grant|any`) — incompatible taxonomies, no mapping layer. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `opportunities.readOnlyAction` i18n key missing; `OpportunityApplySection.tsx:54` renders tooltip only for the `!hasDeadline` branch, never for `!canMutate`. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `TaskTemplateFormModal.tsx` has no 422 per-stage inline error; forward-reference errors still fail silently. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- `ApplyTemplateModal` fires duplicate 422/404 toasts with `use-task-templates`. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- Duplicate `data-testid="templates-empty"` on page wrapper and empty-state root. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- `StageListEditor` `reorderWarnings` accumulates monotonically — banner does not reflect the most recent reorder. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_

### Detected by `3-code-review` at 2026-04-21T10:48:16Z (session 22ef4dc0-2da2-4422-8ef1-21f2c7584e20)

- Settings-nav entry for `settingsNav.taskTemplates` is not wired in the protected layout — feature is unreachable via UI navigation. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `ApplyTemplateModal` casts `opportunity.type` (`works|services|supplies|mixed`) to `OpportunityType` (`tender|grant|any`) — incompatible enums; filter query returns wrong results. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `opportunities.readOnlyAction` i18n key missing and `!canMutate` tooltip branch never rendered on Apply-template button. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- AC6 per-stage 422 inline error not implemented in `TaskTemplateFormModal`; forward-reference errors fail silently. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- `ApplyTemplateModal` fires 422/404 toasts that `use-task-templates` already fires — duplicate toast per failure. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- Duplicate `data-testid="templates-empty"` on page wrapper and empty-state root — will break behavioural test on unskip. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- `reorderWarnings` accumulates monotonically across drags; banner count does not reflect most recent reorder. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- Settings-nav entry for `settingsNav.taskTemplates` is not wired in the protected layout — feature is unreachable via UI navigation.
- `ApplyTemplateModal` casts `opportunity.type` (`works|services|supplies|mixed`) to `OpportunityType` (`tender|grant|any`) — incompatible enums; filter query returns wrong results.
- `opportunities.readOnlyAction` i18n key missing and `!canMutate` tooltip branch never rendered on Apply-template button.
- AC6 per-stage 422 inline error not implemented in `TaskTemplateFormModal`; forward-reference errors fail silently.
- `ApplyTemplateModal` fires 422/404 toasts that `use-task-templates` already fires — duplicate toast per failure. _(type: `MISSING_REQUIREMENT`)_
- Duplicate `data-testid="templates-empty"` on page wrapper and empty-state root — will break behavioural test on unskip. _(type: `MISSING_REQUIREMENT`)_
- `reorderWarnings` accumulates monotonically across drags; banner count does not reflect most recent reorder. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_

### Detected by `3-code-review` at 2026-04-21T11:07:38Z (session 029333c4-2645-42a9-a076-f2266a980432)

- Settings-nav entry for `settingsNav.taskTemplates` is not wired in the protected layout — feature unreachable via UI navigation. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `ApplyTemplateModal` casts `opportunity.type` (`works|services|supplies|mixed`) to `OpportunityType` (`tender|grant|any`) — incompatible enums, no mapping layer. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- `opportunities.readOnlyAction` i18n key missing and `!canMutate` tooltip branch never rendered on Apply-template button. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- AC6 per-stage 422 inline error not implemented; forward-reference errors fail silently. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- `ApplyTemplateModal` fires duplicate 422/404 toasts that `use-task-templates` already fires. _(type: `SCOPE_CREEP`; severity: `deferrable`)_
- Duplicate `data-testid="templates-empty"` on page wrapper and empty-state root. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- `StageListEditor` `reorderWarnings` accumulates monotonically across drags; banner does not reflect the most recent reorder. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- Settings-nav entry for `settingsNav.taskTemplates` is not wired in the protected layout — feature unreachable via UI navigation.
- `ApplyTemplateModal` casts `opportunity.type` (`works|services|supplies|mixed`) to `OpportunityType` (`tender|grant|any`) — incompatible enums, no mapping layer.
- `opportunities.readOnlyAction` i18n key missing and `!canMutate` tooltip branch never rendered on Apply-template button.
- AC6 per-stage 422 inline error not implemented; forward-reference errors fail silently. _(type: `MISSING_REQUIREMENT`)_
- `ApplyTemplateModal` fires duplicate 422/404 toasts that `use-task-templates` already fires. _(type: `MISSING_REQUIREMENT`)_
- Duplicate `data-testid="templates-empty"` on page wrapper and empty-state root. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
- `StageListEditor` `reorderWarnings` accumulates monotonically across drags; banner does not reflect the most recent reorder. _(type: `MISSING_REQUIREMENT`; severity: `blocking`)_
