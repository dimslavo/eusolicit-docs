# Story 12.14: Admin Frontend Pages

Status: review

## Story

As a **platform operator (internal admin)**,
I want **dedicated admin frontend pages for tenant management, crawler monitoring & scheduling, per-tenant white-label configuration, audit log inspection, and platform-level analytics**,
so that **I can manage tenants, control data ingestion pipelines, apply custom branding for enterprise clients, review security-relevant activity, and track key business health indicators — all from the secure admin interface without direct database access**.

## Acceptance Criteria

### AC1 — Route Structure

Six new page shells are created under `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/`:

| Route | Page shell file | Root client component |
|---|---|---|
| `/{locale}/tenants` | `tenants/page.tsx` | `TenantListPage` |
| `/{locale}/tenants/[id]/white-label` | `tenants/[id]/white-label/page.tsx` | `WhiteLabelPage` |
| `/{locale}/crawlers` | `crawlers/page.tsx` | `CrawlerManagementPage` |
| `/{locale}/audit-log` | `audit-log/page.tsx` | `AuditLogPage` |
| `/{locale}/analytics` | `analytics/page.tsx` | `PlatformAnalyticsPage` |

Each `page.tsx` is a minimal server component (no `"use client"`) that imports and renders its root client component, forwarding `params.locale`. Example:

```tsx
// apps/admin/app/[locale]/(protected)/tenants/page.tsx
import TenantListPage from "./components/TenantListPage";
export default function Page({ params: { locale } }: { params: { locale: string } }) {
  return <TenantListPage locale={locale} />;
}
```

The white-label page shell additionally forwards `params.id`:
```tsx
// apps/admin/app/[locale]/(protected)/tenants/[id]/white-label/page.tsx
import WhiteLabelPage from "./components/WhiteLabelPage";
export default function Page({ params }: { params: { locale: string; id: string } }) {
  return <WhiteLabelPage locale={params.locale} companyId={params.id} />;
}
```

---

### AC2 — Navigation Update

`apps/admin/app/[locale]/(protected)/layout.tsx` is updated. Four new entries are added to `adminNavItems` after the existing `ShieldCheck` (Compliance) entry:

```typescript
{ icon: Users2,     label: t("tenants"),   href: `/${locale}/tenants`   },
{ icon: Bot,        label: t("crawlers"),  href: `/${locale}/crawlers`  },
{ icon: ScrollText, label: t("auditLog"),  href: `/${locale}/audit-log` },
{ icon: TrendingUp, label: t("analytics"), href: `/${locale}/analytics` },
```

Import `Users2`, `Bot`, `ScrollText`, `TrendingUp` from `"lucide-react"`. The i18n keys `"tenants"`, `"crawlers"`, `"auditLog"`, `"analytics"` are added under the `"nav"` namespace (see AC16).

---

### AC3 — Admin Role Guard

New file: `apps/admin/components/AdminRoleGuard.tsx`

```tsx
"use client";
// Redirects to /{locale}/403 if the authenticated user does not hold the platform_admin role.
```

- Reads `user` from `useAuthStore()` (imported from `@eusolicit/ui`).
- If `user` is null (not yet loaded): render a full-screen spinner centered on page — `<div className="flex items-center justify-center h-screen"><Loader2 className="animate-spin h-8 w-8 text-slate-400" /></div>`.
- If `user` is present but `user.role !== "platform_admin"`: render `null` and call `router.replace(\`/${locale}/403\`)` inside `useEffect`.
- If `user.role === "platform_admin"`: render `<>{children}</>`.
- Props: `{ locale: string; children: React.ReactNode }`.
- The root wrapper div of the child content receives `data-testid="admin-role-guard"` (pass through from the guarded content, or wrap in a fragment — do not add an extra wrapping div that affects layout).

**Dev note on role field**: Check `apps/admin/lib/stores/` or `useAuthStore` from `@eusolicit/ui` for the exact field name on the user object. The backend `AdminUser` model (from `services/admin-api/src/admin_api/core/security.py`) has `role: str` with value `"platform_admin"`. If the auth store exposes this as `user.role`, use `user.role === "platform_admin"`. If the admin app uses a different field, adapt accordingly.

**Layout update**: In `apps/admin/app/[locale]/(protected)/layout.tsx`, wrap the `<AuthGuard>` children with `<AdminRoleGuard locale={locale}>`:

```tsx
return (
  <AuthGuard locale={locale}>
    <AdminRoleGuard locale={locale}>
      <div data-testid="protected-content">
        {/* existing AppShell / sidebar / topbar content unchanged */}
      </div>
    </AdminRoleGuard>
  </AuthGuard>
);
```

**403 page**: If `apps/admin/app/[locale]/403/page.tsx` does not exist, create a minimal page: `<div className="flex flex-col items-center justify-center h-screen gap-4"><h1 className="text-4xl font-bold">403</h1><p className="text-slate-500">Access denied. Platform admin role required.</p></div>`.

---

### AC4 — Tenant Management Page: Table & Filters

New file: `apps/admin/app/[locale]/(protected)/tenants/components/TenantListPage.tsx` (`"use client"`).

Root element: `<div data-testid="tenant-management-page">`.

**Header** (`data-testid="tenant-page-header"`):
- `<h1 data-testid="tenant-page-title" className="text-2xl font-bold text-slate-900">t("tenants.pageTitle")</h1>`
- `<p data-testid="tenant-page-description" className="text-sm text-slate-500 mt-1">t("tenants.pageDescription")</p>`

**Filter bar** (`data-testid="tenant-filter-bar"`, `className="flex flex-wrap gap-3 mt-4 mb-6"`):
- Search input: `<Input data-testid="tenant-search-input" placeholder={t("tenants.list.searchPlaceholder")} value={searchQuery} onChange={(e) => setSearchQuery(e.target.value)} className="w-64" />`. State: `useState<string>("")` for `searchQuery`.
- Tier filter: `<Select data-testid="tenant-tier-filter" value={tierFilter} onValueChange={setTierFilter}>` with `<SelectItem value="">` (All tiers), `value="free"`, `value="starter"`, `value="professional"`, `value="enterprise"`. State: `useState<string>("")` for `tierFilter`.
- Apply button: `<Button data-testid="tenant-filter-apply-btn" onClick={handleApplyFilters}>t("tenants.list.applyBtn")</Button>` — sets `appliedSearch` and `appliedTier` from current states, resets `page` to 1.
- Clear button: `<Button variant="outline" data-testid="tenant-filter-clear-btn" onClick={handleClearFilters}>t("tenants.list.clearBtn")</Button>` — resets `searchQuery`, `tierFilter`, `appliedSearch`, `appliedTier`, and `page` to defaults.

Applied filters are stored in separate state variables (`appliedSearch: string`, `appliedTier: string`) passed to `useAdminTenants({ search: appliedSearch, tier: appliedTier, page, page_size: 20 })`. The query only re-fires when applied values change, not on every keystroke.

**Tenant table** (`data-testid="tenant-table"`):

While `isLoading`: `<SkeletonTable rows={5} data-testid="tenant-table-skeleton" />` from `@eusolicit/ui`.

On success with items: `<table className="w-full text-sm">` with:
- Header: columns Company Name, Tier, Status, Period End, Actions (using `t("tenants.list.col*")` keys).
- Body rows: `<tr data-testid="tenant-row-{company_id}" className="border-b hover:bg-slate-50 cursor-pointer" onClick={() => openDrawer(item.company_id)}>`:
  - `<td data-testid="tenant-name-{company_id}" className="font-medium py-3 px-4">` — `item.name`
  - `<td data-testid="tenant-tier-{company_id}" className="py-3 px-4">` — `<Badge data-testid="tenant-tier-badge-{company_id}" className={tierBadgeClass(item.plan)}>t(\`tenants.tier.${item.plan ?? "free"}\`)</Badge>`. Badge classes: `free` → `"bg-slate-100 text-slate-700"`, `starter` → `"bg-blue-100 text-blue-700"`, `professional` → `"bg-purple-100 text-purple-700"`, `enterprise` → `"bg-green-100 text-green-700"`.
  - `<td data-testid="tenant-status-{company_id}" className="py-3 px-4 text-slate-600">` — `item.sub_status ?? "—"`
  - `<td data-testid="tenant-period-end-{company_id}" className="py-3 px-4 text-slate-500">` — `item.period_end ? new Date(item.period_end).toLocaleDateString(locale) : "—"`
  - `<td data-testid="tenant-actions-{company_id}" className="py-3 px-4">` — `<Button variant="ghost" size="sm" data-testid="tenant-view-btn-{company_id}" onClick={(e) => { e.stopPropagation(); openDrawer(item.company_id); }}>t("tenants.list.viewBtn")</Button>`

On empty: `<EmptyState data-testid="tenant-empty-state" title={t("tenants.list.emptyTitle")} description={t("tenants.list.emptyDescription")} />`.

**Pagination** (`data-testid="tenant-pagination"`, renders when `total > pageSize`):
- Info: `<span data-testid="tenant-pagination-info">t("tenants.list.paginationInfo", { from: (page-1)*20+1, to: Math.min(page*20, total), total })</span>`
- `<Button data-testid="tenant-prev-btn" disabled={page === 1} onClick={() => setPage(p => p-1)}>t("common.back")</Button>`
- `<Button data-testid="tenant-next-btn" disabled={page * 20 >= total} onClick={() => setPage(p => p+1)}>t("common.next")</Button>`
- State: `useState<number>(1)` for `page` (resets to 1 when applied filters change).

**Drawer state**: `useState<string | null>(null)` for `selectedCompanyId` and `useState<boolean>(false)` for `drawerOpen`. `openDrawer(id)` sets both.

---

### AC5 — Tenant Management Page: Detail Drawer

The `TenantListPage` component also renders a `<Sheet open={drawerOpen} onOpenChange={setDrawerOpen}>`. The `SheetContent` has `data-testid="tenant-detail-drawer"` and `className="w-[480px] sm:w-[560px] overflow-y-auto"`.

While `useAdminTenantDetail(selectedCompanyId)` is loading: `<SkeletonCard data-testid="drawer-skeleton" className="m-4" />`.

On data loaded:

**Sheet header** (`<SheetHeader className="px-6 pt-6 pb-4 border-b">`):
- `<SheetTitle data-testid="drawer-company-name">detail.name</SheetTitle>`
- `<SheetDescription className="text-xs text-slate-400 font-mono">detail.company_id</SheetDescription>`

**Subscription section** (`<section data-testid="drawer-subscription-section" className="px-6 py-4 border-b">`):
- `<h3 className="text-sm font-semibold text-slate-700 mb-3">t("tenants.drawer.subscriptionTitle")</h3>`
- Tier badge (same `tierBadgeClass()` as AC4): `<Badge data-testid="drawer-sub-tier">t(\`tenants.tier.${detail.plan ?? "free"}\`)</Badge>`
- Status: `<span data-testid="drawer-sub-status" className="ml-2 text-sm text-slate-600">detail.sub_status</span>`
- Period: `<p data-testid="drawer-sub-period" className="text-xs text-slate-500 mt-1">t("tenants.drawer.period", { start: detail.period_start ? new Date(detail.period_start).toLocaleDateString(locale) : "—", end: detail.period_end ? new Date(detail.period_end).toLocaleDateString(locale) : "—" })</p>`

**Usage meters section** (`<section data-testid="drawer-usage-section" className="px-6 py-4 border-b">`):
- `<h3 className="text-sm font-semibold text-slate-700 mb-3">t("tenants.drawer.usageTitle")</h3>`
- For each `metric` in `detail.usage`: `<div data-testid={"drawer-usage-meter-" + metric.metric_type} className="mb-3">`:
  - Label: `<p className="text-xs font-medium text-slate-600 mb-1">t(\`tenants.usage.${metric.metric_type}\`)</p>`
  - Progress bar: `<div className="w-full h-2 bg-slate-100 rounded-full"><div className="h-2 rounded-full bg-indigo-500" style={{ width: metric.limit_value ? \`${Math.min((metric.consumed / metric.limit_value) * 100, 100)}%\` : "0%" }} /></div>`
  - Values: `<p className="text-xs text-slate-500 mt-1">{metric.consumed} / {metric.limit_value ?? "∞"}</p>`

**Activity section** (`<section data-testid="drawer-activity-section" className="px-6 py-4 border-b">`):
- `<h3 className="text-sm font-semibold text-slate-700 mb-3">t("tenants.drawer.activityTitle")</h3>`
- `<dl className="grid grid-cols-2 gap-2 text-sm">`:
  - `<dt>` "Last Login" / `<dd data-testid="drawer-last-login">` — formatted date or `t("tenants.drawer.noActivity")`
  - `<dt>` "Bids This Month" / `<dd data-testid="drawer-bids-this-month">` — `detail.activity.bids_submitted_this_month`
  - `<dt>` "Proposals This Month" / `<dd data-testid="drawer-proposals-this-month">` — `detail.activity.proposals_generated_this_month`
  - `<dt>` "Active Users" / `<dd data-testid="drawer-active-users">` — `detail.activity.active_user_count`

**White-label button** at bottom of drawer:
`<Button variant="outline" size="sm" data-testid="drawer-whitelabel-btn" className="mt-4 mx-6" onClick={() => router.push(\`/${locale}/tenants/${selectedCompanyId}/white-label\`)}>t("tenants.drawer.whitelabelBtn")</Button>`

---

### AC6 — Tier Override Form

Rendered as a `<section data-testid="drawer-tier-override-section" className="px-6 py-4">` below the activity section, inside the same Sheet.

- Heading: `<h3 className="text-sm font-semibold text-slate-700 mb-3">t("tenants.drawer.tierOverrideTitle")</h3>`
- New plan select: `<Select data-testid="drawer-override-plan-select" value={overridePlan} onValueChange={setOverridePlan}>` — same options as AC4 tier filter. State: `useState<string>(detail.plan ?? "free")` for `overridePlan` (initialised from drawer data when it loads).
- Reason textarea: `<Textarea data-testid="drawer-override-reason" placeholder={t("tenants.drawer.tierOverrideReasonPlaceholder")} value={overrideReason} onChange={(e) => setOverrideReason(e.target.value)} rows={3} />`. State: `useState<string>("")` for `overrideReason`.
- Inline validation error: `{reasonError && <p data-testid="drawer-override-reason-error" className="text-xs text-red-600 mt-1">{t("tenants.drawer.tierOverrideReasonRequired")}</p>}`. State: `useState<boolean>(false)` for `reasonError`.
- Submit button: `<Button data-testid="drawer-override-submit-btn" disabled={isOverriding} className="mt-3 w-full">`. While mutation in-flight (`isOverriding`): shows `<Loader2 className="animate-spin h-4 w-4 mr-2" />`.

On submit handler:
1. If `overrideReason.trim().length < 10`: set `reasonError = true`; return early.
2. Call `useOverrideTenantTier().mutate({ company_id: selectedCompanyId!, new_plan: overridePlan, reason: overrideReason.trim() })`.
3. `onSuccess`: `queryClient.invalidateQueries({ queryKey: ["admin-tenants"] })`; `queryClient.invalidateQueries({ queryKey: ["admin-tenant-detail", selectedCompanyId] })`; show success toast `t("tenants.drawer.tierOverrideSuccess")`; reset `overrideReason` to `""`.
4. `onError`: show error toast `t("tenants.drawer.tierOverrideError")`.

---

### AC7 — Crawler Management Page: Run History Table

New file: `apps/admin/app/[locale]/(protected)/crawlers/components/CrawlerManagementPage.tsx` (`"use client"`).

Root element: `<div data-testid="crawler-management-page" className="space-y-8">`.

Page header: `<h1 data-testid="crawler-page-title" className="text-2xl font-bold text-slate-900">t("crawlers.pageTitle")</h1>`

**Run History section** (`<section data-testid="crawler-run-history-section">`):

- Heading: `<h2 data-testid="crawler-run-history-title" className="text-lg font-semibold mb-3">t("crawlers.history.title")</h2>`
- Type filter: `<Select data-testid="crawler-type-filter" value={typeFilter} onValueChange={(v) => { setTypeFilter(v); setRunPage(1); }}>` — options: `""` (All), `"ted"`, `"find_a_tender"`, `"merps"`, `"grants_gov"`. Labels use `t(\`crawlers.type.${value}\`)`.
- Table (`<table data-testid="crawler-run-table" className="w-full text-sm mt-3">`), columns: Crawler Type, Status, Triggered By, Started At, Completed At, Results, Actions.
- Each row `<tr data-testid={"crawler-run-row-" + run.id}>`:
  - Type: `<td>t(\`crawlers.type.${run.crawler_type}\`)</td>`
  - Status badge: `<Badge data-testid={"crawler-run-status-" + run.id} className={runStatusClass(run.status)}>t(\`crawlers.status.${run.status}\`)</Badge>`. Classes: `pending` → `"bg-slate-100 text-slate-600"`, `running` → `"bg-blue-100 text-blue-700"`, `completed` → `"bg-green-100 text-green-700"`, `failed` → `"bg-red-100 text-red-700"`.
  - Triggered by: `<td>{run.triggered_by}</td>`
  - Started at: `<td data-testid={"crawler-run-started-" + run.id}>{run.started_at ? new Date(run.started_at).toLocaleString(locale) : "—"}</td>`
  - Completed at: `<td data-testid={"crawler-run-completed-" + run.id}>{run.completed_at ? new Date(run.completed_at).toLocaleString(locale) : "—"}</td>`
  - Results: `<td data-testid={"crawler-run-results-" + run.id}>{run.result_count ?? "—"}</td>`
  - Actions: `<Button variant="ghost" size="sm" data-testid={"crawler-run-view-btn-" + run.id} onClick={() => openRunModal(run.id)}>t("crawlers.history.viewBtn")</Button>`
- Loading: `<SkeletonTable rows={5} data-testid="crawler-run-skeleton" />`
- Empty: `<EmptyState data-testid="crawler-run-empty" title={t("crawlers.history.emptyTitle")} />`
- Pagination: same pattern as AC4 — testids `crawler-run-pagination`, `crawler-run-prev-btn`, `crawler-run-next-btn`, `crawler-run-page-info`. State: `useState<number>(1)` for `runPage`, `pageSize = 20`.

**Run Detail Modal** (`<Dialog open={runModalOpen} onOpenChange={setRunModalOpen}>`):

`data-testid="crawler-run-detail-modal"` on the `<DialogContent>`.

While loading `useAdminCrawlerRunDetail(selectedRunId)`: `<SkeletonCard data-testid="run-detail-skeleton" />`.

On data loaded (`<dl data-testid="run-detail-content" className="grid grid-cols-2 gap-y-3 text-sm">`):
- `data-testid="run-detail-id"` — full run UUID
- `data-testid="run-detail-type"` — `t(\`crawlers.type.${run.crawler_type}\`)`
- `data-testid="run-detail-status"` — status badge (same classes as table)
- `data-testid="run-detail-started"` — formatted timestamp or `"—"`
- `data-testid="run-detail-completed"` — formatted timestamp or `"—"`
- `data-testid="run-detail-result-count"` — count or `"—"`
- `data-testid="run-detail-task-id"` — `run.celery_task_id ?? "—"`
- Error detail section: shown only when `run.status === "failed" && run.error_detail`: `<pre data-testid="run-detail-error" className="col-span-2 text-red-600 text-xs whitespace-pre-wrap bg-red-50 rounded p-3">{run.error_detail}</pre>`

Close button: `<Button variant="outline" data-testid="run-detail-close-btn" onClick={() => setRunModalOpen(false)}>t("crawlers.runDetail.closeBtn")</Button>`

State: `useState<string | null>(null)` for `selectedRunId` and `useState<boolean>(false)` for `runModalOpen`.

---

### AC8 — Crawler Management Page: Schedule Configuration

**Schedule section** (`<section data-testid="crawler-schedule-section">`):

- Heading: `<h2 data-testid="crawler-schedule-title">t("crawlers.schedule.title")</h2>`
- For each type in `["ted", "find_a_tender", "merps", "grants_gov"]`: render a `<div data-testid={"crawler-schedule-card-" + type} className="p-4 border rounded-lg mb-3">`.
  - `useAdminCrawlerSchedule(type)` fetches the schedule. While loading: `<SkeletonCard data-testid={"crawler-schedule-skeleton-" + type} />`.
  - On data: render inside a local form state (separate `useState` per card, initialised from the fetched schedule):
    - Type label: `<h3 className="font-medium mb-3">t(\`crawlers.type.${type}\`)</h3>`
    - Enabled toggle: `<Switch data-testid={"crawler-schedule-enabled-" + type} checked={localEnabled} onCheckedChange={setLocalEnabled} />` with label `t("crawlers.schedule.enabledLabel")`.
    - Hour input: `<Input type="number" min="0" max="23" data-testid={"crawler-schedule-hour-" + type} value={localHour} onChange={(e) => setLocalHour(e.target.value)} className="w-20" />` with label `t("crawlers.schedule.hourLabel")`.
    - Minute input: `<Input type="number" min="0" max="59" data-testid={"crawler-schedule-minute-" + type} value={localMinute} onChange={(e) => setLocalMinute(e.target.value)} className="w-20" />` with label `t("crawlers.schedule.minuteLabel")`.
    - Save button: `<Button size="sm" data-testid={"crawler-schedule-save-" + type} disabled={isSaving} onClick={() => handleSaveSchedule(type)}>`. Shows `<Loader2 />` while `useUpdateCrawlerSchedule()` mutation is in-flight for this type. On success: `queryClient.invalidateQueries({ queryKey: ["admin-crawler-schedules", type] })` + success toast.

Each card manages its own local state independently. The `isSaving` state per card is tracked via `useState<string | null>(null)` for `savingType` at the parent component level (set to `type` during mutation, null on complete).

---

### AC9 — Crawler Management Page: Manual Trigger

**Manual trigger section** (`<section data-testid="crawler-trigger-section">`):

- Heading: `<h2 data-testid="crawler-trigger-title">t("crawlers.trigger.title")</h2>`
- Four buttons in a `<div className="grid grid-cols-2 gap-3 mt-3">`:
  ```
  <Button data-testid={"crawler-trigger-btn-" + type} onClick={() => openTriggerDialog(type)}>
    t(`crawlers.type.${type}`) — t("crawlers.trigger.triggerBtn")
  </Button>
  ```
- State: `useState<string | null>(null)` for `pendingTriggerType` and `useState<boolean>(false)` for `triggerDialogOpen`.

**AlertDialog** (`<AlertDialog open={triggerDialogOpen} onOpenChange={setTriggerDialogOpen}>`):
- `<AlertDialogContent data-testid="crawler-trigger-confirm-dialog">`
- `<AlertDialogTitle>t("crawlers.trigger.confirmTitle")</AlertDialogTitle>`
- `<AlertDialogDescription>t("crawlers.trigger.confirmDescription", { type: pendingTriggerType ? t(\`crawlers.type.${pendingTriggerType}\`) : "" })</AlertDialogDescription>`
- `<AlertDialogCancel data-testid="crawler-trigger-cancel-btn">t("common.cancel")</AlertDialogCancel>`
- `<AlertDialogAction data-testid="crawler-trigger-confirm-btn" onClick={handleConfirmTrigger}>t("crawlers.trigger.triggerBtn")</AlertDialogAction>`

`handleConfirmTrigger()`:
1. Calls `useManualTriggerCrawl().mutate(pendingTriggerType!)`.
2. On success: show toast `t("crawlers.trigger.successToast", { runId: result.run_id })`; `queryClient.invalidateQueries({ queryKey: ["admin-crawler-runs"] })`.
3. Closes dialog (`setTriggerDialogOpen(false)`).

---

### AC10 — White-Label Configuration Page

New file: `apps/admin/app/[locale]/(protected)/tenants/[id]/white-label/components/WhiteLabelPage.tsx` (`"use client"`).

Root element: `<div data-testid="whitelabel-page" className="max-w-5xl mx-auto p-6">`.

**Header** (`data-testid="whitelabel-page-header"`):
- Back button: `<Button variant="ghost" data-testid="whitelabel-back-btn" onClick={() => router.back()}>← t("common.back")</Button>`
- `<h1 data-testid="whitelabel-page-title" className="text-2xl font-bold mt-4">t("whiteLabelSettings.pageTitle")</h1>`

`useAdminWhiteLabel(companyId)` fetches existing settings. While loading: `<SkeletonCard data-testid="whitelabel-skeleton" />`.

**Two-column layout** (`<div className="grid grid-cols-1 lg:grid-cols-2 gap-8 mt-6">`):

**Left — Settings form** (`<form data-testid="whitelabel-form" onSubmit={handleSave} className="space-y-5">`):
- Form state: `useState<string>` for each field — `subdomain`, `logoUrl`, `primaryColor` (default `"#3b82f6"`), `accentColor` (default `"#10b981"`), `emailDomain`. All initialised from `useAdminWhiteLabel` data via `useEffect` when data loads.
- Subdomain: `<Input data-testid="whitelabel-subdomain-input" placeholder="mycompany" value={subdomain} onChange={(e) => setSubdomain(e.target.value)} />` with label `t("whiteLabelSettings.subdomainLabel")` and helper `<p className="text-xs text-slate-400">{t("whiteLabelSettings.subdomainHelper", { subdomain: subdomain || "mycompany" })}</p>`. On 409 error from save: show `<p data-testid="whitelabel-subdomain-error" className="text-xs text-red-600">{t("whiteLabelSettings.subdomainConflict")}</p>`. State: `useState<boolean>(false)` for `subdomainConflict`.
- Logo URL: `<Input data-testid="whitelabel-logo-url-input" type="url" placeholder="https://..." value={logoUrl} onChange={(e) => setLogoUrl(e.target.value)} />` with label `t("whiteLabelSettings.logoUrlLabel")`.
- Primary colour: label `t("whiteLabelSettings.primaryColorLabel")`. Row with `<input type="color" data-testid="whitelabel-primary-color-input" value={primaryColor} onChange={(e) => setPrimaryColor(e.target.value)} className="h-9 w-12 rounded border cursor-pointer" />` and `<Input data-testid="whitelabel-primary-color-hex" value={primaryColor} onChange={(e) => setPrimaryColor(e.target.value)} className="w-32 font-mono" placeholder="#3b82f6" />` — both keep `primaryColor` state in sync.
- Accent colour: same pattern — `data-testid="whitelabel-accent-color-input"` and `data-testid="whitelabel-accent-color-hex"`.
- Email domain: `<Input data-testid="whitelabel-email-domain-input" placeholder="mail.example.com" value={emailDomain} onChange={(e) => setEmailDomain(e.target.value)} />` with label `t("whiteLabelSettings.emailDomainLabel")`.
- DNS status: `<Badge data-testid="whitelabel-dns-badge" className={settings?.dns_verified ? "bg-green-100 text-green-700" : "bg-amber-100 text-amber-700"}>{settings?.dns_verified ? t("whiteLabelSettings.dnsVerified") : t("whiteLabelSettings.dnsPending")}</Badge>` (rendered from fetched data, not form state).
- Save button: `<Button type="submit" data-testid="whitelabel-save-btn" disabled={isSaving}>`. While `useUpdateAdminWhiteLabel().mutate(...)` in-flight: shows `<Loader2 className="animate-spin h-4 w-4 mr-2" />` + label. On success: `queryClient.invalidateQueries({ queryKey: ["admin-whitelabel", companyId] })` + success toast. On 409: set `subdomainConflict = true`.

**Right — Live preview panel** (`<div data-testid="whitelabel-preview-panel" className="sticky top-6">`):
- `<p className="text-sm font-semibold text-slate-700 mb-3">t("whiteLabelSettings.previewTitle")</p>`
- Preview box (`<div data-testid="whitelabel-preview-box" className="rounded-lg border overflow-hidden" style={{ backgroundColor: primaryColor }}`):
  - Mock nav bar: `<div className="h-12 flex items-center px-4 gap-3">`:
    - Logo: when `logoUrl` is non-empty, `<img src={logoUrl} alt="logo" data-testid="whitelabel-preview-logo" className="h-7 object-contain" onError={(e) => { (e.target as HTMLImageElement).style.display = "none"; }} />` (hidden on broken URL)
    - Brand name: `<span className="font-bold text-white" style={{ color: accentColor }}>EU Solicit</span>`
  - Subdomain chip: `<div className="bg-white bg-opacity-20 px-3 py-2 text-xs text-white">{subdomain || "mycompany"}.eusolicit.com</div>`
  - Sample content block: `<div className="bg-white m-3 rounded p-3 text-xs text-slate-500">Sample page content preview</div>`
- The preview updates in real time because all preview values are read directly from the controlled form state (`primaryColor`, `accentColor`, `logoUrl`, `subdomain`).

---

### AC11 — Audit Log Page: Filter Bar & Table

New file: `apps/admin/app/[locale]/(protected)/audit-log/components/AuditLogPage.tsx` (`"use client"`).

Root element: `<div data-testid="audit-log-page">`.

Header: `<h1 data-testid="audit-log-page-title" className="text-2xl font-bold mb-6">t("auditLog.pageTitle")</h1>`

**Filter bar** (`<div data-testid="audit-log-filter-bar" className="flex flex-wrap gap-3 mb-4 p-4 bg-slate-50 rounded-lg">`):
- User ID: `<Input data-testid="audit-log-filter-user" placeholder={t("auditLog.filter.userPlaceholder")} value={userFilter} onChange={(e) => setUserFilter(e.target.value)} className="w-64" />`
- Action type: `<Select data-testid="audit-log-filter-action" value={actionFilter} onValueChange={setActionFilter}>` — options: `""` (All actions), `"tier_override"`, `"create"`, `"update"`, `"delete"`, `"login"`, `"logout"`.
- Entity type: `<Select data-testid="audit-log-filter-entity" value={entityTypeFilter} onValueChange={setEntityTypeFilter}>` — options: `""` (All entities), `"subscription"`, `"company"`, `"user"`, `"opportunity"`, `"proposal"`.
- Date from: `<Input type="date" data-testid="audit-log-filter-date-from" value={dateFrom} onChange={(e) => setDateFrom(e.target.value)} />`
- Date to: `<Input type="date" data-testid="audit-log-filter-date-to" value={dateTo} onChange={(e) => setDateTo(e.target.value)} />`
- Apply: `<Button data-testid="audit-log-filter-apply-btn" onClick={handleApplyFilters}>t("auditLog.filter.applyBtn")</Button>` — sets all `applied*` states from current inputs; resets `page` to 1.
- Clear: `<Button variant="outline" data-testid="audit-log-filter-clear-btn" onClick={handleClearFilters}>t("auditLog.filter.clearBtn")</Button>` — resets all filter inputs and applied values.
- Export CSV: `<Button variant="outline" data-testid="audit-log-export-btn" onClick={handleExport} disabled={isExporting}>` — see AC12.

Applied filters stored separately: `appliedUser`, `appliedAction`, `appliedEntityType`, `appliedDateFrom`, `appliedDateTo` (all `useState<string>("")`). Passed to `useAdminAuditLogs({ user_id: appliedUser || undefined, action_type: appliedAction || undefined, entity_type: appliedEntityType || undefined, date_from: appliedDateFrom || undefined, date_to: appliedDateTo || undefined, page, page_size: 25 })`.

**Audit log table** (`<table data-testid="audit-log-table" className="w-full text-sm">`):

Columns: ID, User, Action, Entity, Entity ID, IP, Timestamp.

While loading: `<SkeletonTable rows={8} data-testid="audit-log-skeleton" />`.

Each row `<tr data-testid={"audit-log-row-" + entry.id}>`:
- ID: `<td data-testid={"audit-log-id-" + entry.id} className="font-mono text-xs">{entry.id.slice(0, 8)}…</td>`
- User: `<td data-testid={"audit-log-user-" + entry.id}>{entry.user_id ? entry.user_id.slice(0, 8) + "…" : "—"}</td>`
- Action: `<td data-testid={"audit-log-action-" + entry.id}><Badge className={actionBadgeClass(entry.action_type)}>{entry.action_type}</Badge></td>`. Badge classes: `tier_override` → `"bg-amber-100 text-amber-700"`, `delete` → `"bg-red-100 text-red-700"`, `create` → `"bg-green-100 text-green-700"`, default → `"bg-slate-100 text-slate-700"`.
- Entity type: `<td data-testid={"audit-log-entity-type-" + entry.id}>{entry.entity_type ?? "—"}</td>`
- Entity ID: `<td data-testid={"audit-log-entity-id-" + entry.id} className="font-mono text-xs">{entry.entity_id ? entry.entity_id.slice(0, 8) + "…" : "—"}</td>`
- IP: `<td data-testid={"audit-log-ip-" + entry.id}>{entry.ip_address ?? "—"}</td>`
- Timestamp: `<td data-testid={"audit-log-ts-" + entry.id} className="text-slate-500">{new Date(entry.created_at).toLocaleString(locale)}</td>`

Empty: `<EmptyState data-testid="audit-log-empty" title={t("auditLog.emptyTitle")} description={t("auditLog.emptyDescription")} />`.

Pagination: testids `audit-log-pagination`, `audit-log-prev-btn`, `audit-log-next-btn`, `audit-log-page-info`. `pageSize = 25`.

---

### AC12 — Audit Log CSV Export

`handleExport()` function called from the Export CSV button:
1. Set `isExporting = true`.
2. Call `fetchAdminAuditLogExport({ user_id: appliedUser || undefined, action_type: appliedAction || undefined, entity_type: appliedEntityType || undefined, date_from: appliedDateFrom || undefined, date_to: appliedDateTo || undefined })` — makes `GET /api/v1/admin/audit-logs/export` and returns the CSV text.
3. Create download: `const blob = new Blob([csvText], { type: "text/csv;charset=utf-8;" }); const url = URL.createObjectURL(blob); const a = document.createElement("a"); a.href = url; a.download = \`audit-log-export-${new Date().toISOString().slice(0,10)}.csv\`; a.click(); URL.revokeObjectURL(url);`
4. Set `isExporting = false`.
5. On error: show error toast `t("auditLog.exportError")`; set `isExporting = false`.

State: `useState<boolean>(false)` for `isExporting`.

---

### AC13 — Platform Analytics Page

New file: `apps/admin/app/[locale]/(protected)/analytics/components/PlatformAnalyticsPage.tsx` (`"use client"`).

Root element: `<div data-testid="platform-analytics-page" className="space-y-8">`.

Header: `<h1 data-testid="platform-analytics-title" className="text-2xl font-bold">t("platformAnalytics.pageTitle")</h1>`

**Section 1 — Signup Funnel** (`<section data-testid="platform-analytics-funnel-section">`):
- Heading: `<h2 className="text-lg font-semibold mb-4">t("platformAnalytics.funnelTitle")</h2>`
- While `useAdminFunnel()` loading: `<SkeletonCard data-testid="funnel-skeleton" />`
- On data: horizontal funnel bars. The `registered` count is used as the 100% baseline. For each `stage` in `funnel.stages`:
  ```
  <div data-testid={"funnel-stage-" + stage.stage} className="mb-2">
    <div className="flex items-center gap-3">
      <span className="text-sm w-36 text-slate-600">{t(`platformAnalytics.funnel.${stage.stage}`)}</span>
      <div className="flex-1 h-7 bg-slate-100 rounded relative overflow-hidden">
        <div className="h-full bg-indigo-500 rounded" style={{ width: `${widthPct}%` }} />
        <span className="absolute inset-0 flex items-center px-3 text-xs font-medium text-white">{stage.count}</span>
      </div>
      <span className="text-xs text-slate-500 w-24">{conversionLabel}</span>
    </div>
  </div>
  ```
  `widthPct`: `registered` = 100, others = `Math.round((stage.count / registeredCount) * 100)`. `conversionLabel`: for non-registered stages = `t("platformAnalytics.funnel.conversionRate", { rate: widthPct })`.

**Section 2 — Tier Distribution** (`<section data-testid="platform-analytics-tiers-section">`):
- Heading: `<h2>t("platformAnalytics.tiersTitle")</h2>`
- While `useAdminTierDistribution()` loading: `<SkeletonCard data-testid="tiers-skeleton" />`
- On data: `<PieChart data-testid="platform-tiers-chart" width={360} height={280}><Pie data={tiers} dataKey="count" nameKey="plan" cx="50%" cy="50%" outerRadius={100}>{tiers.map((entry) => <Cell key={entry.plan} fill={tierPieColor(entry.plan)} />)}</Pie><Tooltip /><Legend /></PieChart>`. Pie colours: `free` → `"#94a3b8"`, `starter` → `"#60a5fa"`, `professional` → `"#a78bfa"`, `enterprise` → `"#34d399"`.
- Total companies badge: `<p className="text-sm text-slate-500 mt-2">Total: {data.total_companies}</p>`

**Section 3 — Revenue & Churn** (`<section data-testid="platform-analytics-revenue-section">`):
- Heading: `<h2>t("platformAnalytics.revenueTitle")</h2>`
- While `useAdminRevenue()` loading: `<SkeletonCard data-testid="revenue-skeleton" />`
- On data: `<div data-testid="revenue-metrics-grid" className="grid grid-cols-2 md:grid-cols-4 gap-4">`:
  - MRR: `<div data-testid="revenue-mrr-card" className="p-4 border rounded-lg"><p className="text-xs text-slate-500">{t("platformAnalytics.revenue.mrr")}</p><p className="text-2xl font-bold mt-1">${(m.mrr_usd / 1000).toFixed(1)}K</p></div>`
  - Growth: `<div data-testid="revenue-growth-card">` — value `{m.growth_rate_pct >= 0 ? "+" : ""}{m.growth_rate_pct.toFixed(1)}%` with `className={m.growth_rate_pct >= 0 ? "text-green-600" : "text-red-600"}` on the value element.
  - Churn: `<div data-testid="revenue-churn-card">` — value `{m.churn_rate_pct.toFixed(1)}%`
  - Active Paid: `<div data-testid="revenue-active-card">` — value `{m.active_paid_companies}`

**Section 4 — Platform Usage** (`<section data-testid="platform-analytics-usage-section">`):
- Heading: `<h2>t("platformAnalytics.usageTitle")</h2>`
- While `useAdminPlatformUsage()` loading: `<SkeletonCard data-testid="usage-skeleton" />`
- On data: period label `<p data-testid="platform-usage-period" className="text-sm text-slate-500 mb-3">{t("platformAnalytics.usage.period", { month: data.period_month })}</p>` + `<BarChart data-testid="platform-usage-chart" width={480} height={240} data={data.metrics}><XAxis dataKey="metric_type" /><YAxis /><Tooltip /><Legend /><Bar dataKey="total_consumed" fill="#6366f1" /></BarChart>`.

---

### AC14 — API Functions

**`apps/admin/lib/api/admin-tenants.ts`** — stub phase, following `compliance-frameworks.ts` pattern:

TypeScript interfaces (matching backend Pydantic schemas from S12.11–S12.12):
```typescript
export interface TenantListParams { search?: string; tier?: string; page?: number; page_size?: number; }
export interface TenantListItem { company_id: string; name: string; registration_number: string|null; industry: string|null; plan: string|null; sub_status: string|null; period_end: string|null; created_at: string|null; }
export interface TenantListResponse { items: TenantListItem[]; total: number; page: number; page_size: number; }
export interface UsageMetric { metric_type: string; consumed: number; limit_value: number|null; remaining: number|null; period_start: string|null; period_end: string|null; }
export interface TenantActivitySummary { last_login: string|null; bids_submitted_this_month: number; proposals_generated_this_month: number; active_user_count: number; }
export interface TenantDetailResponse { company_id: string; name: string; registration_number: string|null; industry: string|null; plan: string|null; sub_status: string|null; period_start: string|null; period_end: string|null; usage: UsageMetric[]; activity: TenantActivitySummary; }
export interface TierOverrideRequest { new_plan: "free"|"starter"|"professional"|"enterprise"; reason: string; }
export interface TierOverrideResponse { company_id: string; previous_plan: string|null; new_plan: string; reason: string; overridden_by_admin_id: string; overridden_at: string; }
export interface WhiteLabelSettings { company_id: string; custom_subdomain: string|null; logo_url: string|null; brand_primary_color: string|null; brand_accent_color: string|null; email_sender_domain: string|null; dns_verified: boolean; }
export interface WhiteLabelUpdateRequest { custom_subdomain?: string|null; logo_url?: string|null; brand_primary_color?: string|null; brand_accent_color?: string|null; email_sender_domain?: string|null; }
```

Exported functions with stubs and real API comments:
- `listAdminTenants(params?)` → Real: `GET /api/v1/admin/tenants` with query params
- `getAdminTenantDetail(companyId)` → Real: `GET /api/v1/admin/tenants/{company_id}`
- `overrideAdminTenantTier(companyId, body)` → Real: `POST /api/v1/admin/tenants/{company_id}/tier-override`
- `getAdminWhiteLabel(companyId)` → Real: `GET /api/v1/admin/tenants/{company_id}/white-label`
- `updateAdminWhiteLabel(companyId, body)` → Real: `PUT /api/v1/admin/tenants/{company_id}/white-label`

Provide realistic stub data: 3–4 `TenantListItem` entries (varied tiers/statuses), a detail response with 3 usage metrics, white-label settings.

**`apps/admin/lib/api/admin-crawlers.ts`** — interfaces and stubs for:
```typescript
export interface CrawlerRun { id: string; crawler_type: string; status: "pending"|"running"|"completed"|"failed"; triggered_by: string; triggered_at: string; started_at: string|null; completed_at: string|null; result_count: number|null; error_detail: string|null; celery_task_id: string|null; }
export interface CrawlerRunListParams { crawler_type?: string; page?: number; page_size?: number; }
export interface CrawlerRunListResponse { items: CrawlerRun[]; total: number; page: number; page_size: number; }
export interface CrawlerSchedule { crawler_type: string; cron_minute: string; cron_hour: string; cron_day_of_week: string; cron_day_of_month: string; cron_month_of_year: string; enabled: boolean; updated_at: string; }
export interface CrawlerScheduleUpdateRequest { cron_minute?: string; cron_hour?: string; enabled?: boolean; }
export interface TriggerResponse { run_id: string; crawler_type: string; status: string; }
```

Functions:
- `listCrawlerRuns(params?)` → Real: `GET /api/v1/admin/crawlers/runs`
- `getCrawlerSchedule(crawlerType)` → Real: `GET /api/v1/admin/crawlers/schedule/{crawler_type}`
- `updateCrawlerSchedule(crawlerType, body)` → Real: `PUT /api/v1/admin/crawlers/schedule/{crawler_type}`
- `triggerCrawl(crawlerType)` → Real: `POST /api/v1/admin/crawlers/trigger/{crawler_type}`
- `getCrawlerRunDetail(runId)` → Real: `GET /api/v1/admin/crawlers/runs/{run_id}`

**`apps/admin/lib/api/admin-audit-log.ts`** — interfaces and stubs for:
```typescript
export interface AuditLogEntry { id: string; user_id: string|null; action_type: string; entity_type: string|null; entity_id: string|null; before: Record<string, unknown>|null; after: Record<string, unknown>|null; ip_address: string|null; created_at: string; }
export interface AuditLogListParams { user_id?: string; action_type?: string; entity_type?: string; date_from?: string; date_to?: string; page?: number; page_size?: number; }
export interface AuditLogListResponse { items: AuditLogEntry[]; total: number; page: number; page_size: number; }
export type AuditLogExportParams = Omit<AuditLogListParams, "page" | "page_size">;
```

Functions:
- `listAuditLogs(params?)` → Real: `GET /api/v1/admin/audit-logs`
- `fetchAdminAuditLogExport(params?)` → Real: `GET /api/v1/admin/audit-logs/export` — returns CSV text string

**`apps/admin/lib/api/admin-analytics.ts`** — interfaces and stubs for:
```typescript
export interface FunnelStage { stage: "registered"|"trial_started"|"paid_conversion"; count: number; }
export interface FunnelResponse { stages: FunnelStage[]; }
export interface TierDistributionItem { plan: string; count: number; }
export interface TierDistributionResponse { tiers: TierDistributionItem[]; total_companies: number; }
export interface UsageAggregate { metric_type: string; total_consumed: number; total_limit: number|null; company_count: number; }
export interface PlatformUsageResponse { period_month: string; metrics: UsageAggregate[]; }
export interface RevenueMetrics { mrr_usd: number; growth_rate_pct: number; churn_rate_pct: number; active_paid_companies: number; new_paid_this_month: number; churned_this_month: number; }
export interface RevenueResponse { metrics: RevenueMetrics; }
```

Functions:
- `getPlatformFunnel()` → Real: `GET /api/v1/admin/analytics/funnel`
- `getPlatformTiers()` → Real: `GET /api/v1/admin/analytics/tiers`
- `getPlatformUsage()` → Real: `GET /api/v1/admin/analytics/usage`
- `getPlatformRevenue()` → Real: `GET /api/v1/admin/analytics/revenue`

---

### AC15 — TanStack Query Hooks

All files begin with `"use client";`.

**`apps/admin/lib/queries/use-admin-tenants.ts`**:
```typescript
export function useAdminTenants(params?: TenantListParams) {
  return useQuery({ queryKey: ["admin-tenants", params], queryFn: () => listAdminTenants(params), staleTime: 30_000 });
}
export function useAdminTenantDetail(companyId: string | null) {
  return useQuery({ queryKey: ["admin-tenant-detail", companyId], queryFn: () => getAdminTenantDetail(companyId!), enabled: !!companyId, staleTime: 30_000 });
}
export function useOverrideTenantTier() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ company_id, ...body }: { company_id: string } & TierOverrideRequest) =>
      overrideAdminTenantTier(company_id, body),
    onSuccess: (_, vars) => {
      queryClient.invalidateQueries({ queryKey: ["admin-tenants"] });
      queryClient.invalidateQueries({ queryKey: ["admin-tenant-detail", vars.company_id] });
    },
  });
}
export function useAdminWhiteLabel(companyId: string | null) {
  return useQuery({ queryKey: ["admin-whitelabel", companyId], queryFn: () => getAdminWhiteLabel(companyId!), enabled: !!companyId, staleTime: 60_000 });
}
export function useUpdateAdminWhiteLabel() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: ({ companyId, body }: { companyId: string; body: WhiteLabelUpdateRequest }) =>
      updateAdminWhiteLabel(companyId, body),
    onSuccess: (_, vars) => {
      queryClient.invalidateQueries({ queryKey: ["admin-whitelabel", vars.companyId] });
    },
  });
}
```

**`apps/admin/lib/queries/use-admin-crawlers.ts`**:
- `useAdminCrawlerRuns(params?)` — `queryKey: ["admin-crawler-runs", params]`, `staleTime: 15_000`
- `useAdminCrawlerSchedule(crawlerType: string)` — `queryKey: ["admin-crawler-schedules", crawlerType]`, `enabled: !!crawlerType`, `staleTime: 60_000`
- `useUpdateCrawlerSchedule()` — `useMutation` wrapping `({ crawlerType, body }) => updateCrawlerSchedule(crawlerType, body)`; `onSuccess` invalidates `["admin-crawler-schedules", crawlerType]`
- `useManualTriggerCrawl()` — `useMutation` wrapping `triggerCrawl`; `onSuccess` invalidates `["admin-crawler-runs"]`
- `useAdminCrawlerRunDetail(runId: string | null)` — `queryKey: ["admin-crawler-run-detail", runId]`, `enabled: !!runId`, `staleTime: 10_000`

**`apps/admin/lib/queries/use-admin-audit-log.ts`**:
- `useAdminAuditLogs(params?: AuditLogListParams)` — `queryKey: ["admin-audit-logs", params]`, `staleTime: 10_000`

**`apps/admin/lib/queries/use-admin-analytics.ts`**:
- `useAdminFunnel()` — `queryKey: ["admin-analytics-funnel"]`, `staleTime: 120_000`
- `useAdminTierDistribution()` — `queryKey: ["admin-analytics-tiers"]`, `staleTime: 120_000`
- `useAdminPlatformUsage()` — `queryKey: ["admin-analytics-usage"]`, `staleTime: 120_000`
- `useAdminRevenue()` — `queryKey: ["admin-analytics-revenue"]`, `staleTime: 120_000`

---

### AC16 — i18n Keys

Add to **both** `apps/admin/messages/en.json` and `apps/admin/messages/bg.json`. English values shown; Bulgarian values should be reasonable translations (use the placeholder `[BG: <english>]` pattern used elsewhere in the codebase if professional translations are unavailable at implementation time).

```json
{
  "nav": {
    "tenants":   "Tenants",
    "crawlers":  "Crawlers",
    "auditLog":  "Audit Log",
    "analytics": "Analytics"
  },
  "tenants": {
    "pageTitle": "Tenant Management",
    "pageDescription": "Search, inspect, and manage company subscriptions and tier overrides.",
    "list": {
      "searchPlaceholder": "Search by company name…",
      "applyBtn": "Apply",
      "clearBtn": "Clear",
      "colName": "Company Name",
      "colTier": "Tier",
      "colStatus": "Status",
      "colPeriodEnd": "Period End",
      "colActions": "Actions",
      "viewBtn": "View",
      "emptyTitle": "No tenants found",
      "emptyDescription": "Try adjusting your search or filters.",
      "paginationInfo": "{from}–{to} of {total}"
    },
    "tier": {
      "free": "Free",
      "starter": "Starter",
      "professional": "Professional",
      "enterprise": "Enterprise"
    },
    "drawer": {
      "subscriptionTitle": "Subscription",
      "period": "{start} – {end}",
      "usageTitle": "Current Usage",
      "activityTitle": "Activity",
      "noActivity": "No activity recorded",
      "tierOverrideTitle": "Override Tier",
      "tierOverrideReasonPlaceholder": "Reason for override (min. 10 characters)…",
      "tierOverrideSubmitBtn": "Apply Override",
      "tierOverrideReasonRequired": "Please provide a reason (at least 10 characters).",
      "tierOverrideSuccess": "Tier updated successfully.",
      "tierOverrideError": "Failed to update tier. Please try again.",
      "whitelabelBtn": "Configure White-Label"
    },
    "usage": {
      "ai_summaries": "AI Summaries",
      "proposal_drafts": "Proposal Drafts",
      "compliance_checks": "Compliance Checks"
    }
  },
  "crawlers": {
    "pageTitle": "Crawler Management",
    "history": {
      "title": "Crawler Run History",
      "viewBtn": "Details",
      "emptyTitle": "No crawler runs found",
      "paginationInfo": "{from}–{to} of {total}"
    },
    "schedule": {
      "title": "Crawler Schedules",
      "saveBtn": "Save",
      "hourLabel": "Hour (0–23)",
      "minuteLabel": "Minute (0–59)",
      "enabledLabel": "Enabled"
    },
    "trigger": {
      "title": "Manual Trigger",
      "triggerBtn": "Trigger",
      "confirmTitle": "Trigger crawl?",
      "confirmDescription": "Start a manual {type} crawl. This may take several minutes.",
      "successToast": "Crawl triggered. Run ID: {runId}"
    },
    "type": {
      "ted": "TED",
      "find_a_tender": "Find a Tender",
      "merps": "MERPS",
      "grants_gov": "Grants.gov"
    },
    "status": {
      "pending": "Pending",
      "running": "Running",
      "completed": "Completed",
      "failed": "Failed"
    },
    "runDetail": {
      "title": "Run Details",
      "closeBtn": "Close",
      "runId": "Run ID",
      "type": "Crawler Type",
      "status": "Status",
      "startedAt": "Started At",
      "completedAt": "Completed At",
      "resultCount": "Records Processed",
      "taskId": "Celery Task ID",
      "errorDetail": "Error Details"
    }
  },
  "auditLog": {
    "pageTitle": "Audit Log",
    "filter": {
      "userPlaceholder": "Filter by user ID…",
      "applyBtn": "Apply",
      "clearBtn": "Clear"
    },
    "table": {
      "colId": "ID",
      "colUser": "User",
      "colAction": "Action",
      "colEntity": "Entity",
      "colEntityId": "Entity ID",
      "colIp": "IP Address",
      "colTimestamp": "Timestamp"
    },
    "exportBtn": "Export CSV",
    "exportError": "Failed to export audit log.",
    "emptyTitle": "No audit log entries",
    "emptyDescription": "Adjust your filters to see results.",
    "paginationInfo": "{from}–{to} of {total}"
  },
  "whiteLabelSettings": {
    "pageTitle": "White-Label Configuration",
    "subdomainLabel": "Custom Subdomain",
    "subdomainHelper": "Will be served at {subdomain}.eusolicit.com",
    "logoUrlLabel": "Logo URL",
    "primaryColorLabel": "Primary Colour",
    "accentColorLabel": "Accent Colour",
    "emailDomainLabel": "Email Sender Domain",
    "dnsVerified": "DNS Verified ✓",
    "dnsPending": "DNS Verification Pending",
    "previewTitle": "Live Preview",
    "saveSuccess": "White-label settings saved.",
    "saveError": "Failed to save settings.",
    "subdomainConflict": "This subdomain is already in use by another tenant."
  },
  "platformAnalytics": {
    "pageTitle": "Platform Analytics",
    "funnelTitle": "Signup Funnel",
    "tiersTitle": "Tier Distribution",
    "revenueTitle": "Revenue & Churn",
    "usageTitle": "Platform Usage",
    "funnel": {
      "registered": "Registered",
      "trial_started": "Trial Started",
      "paid_conversion": "Paid",
      "conversionRate": "{rate}% conversion"
    },
    "revenue": {
      "mrr": "Monthly Recurring Revenue",
      "growth": "MoM Growth",
      "churn": "Monthly Churn",
      "activePaid": "Active Paid Companies"
    },
    "usage": {
      "period": "Period: {month}"
    }
  }
}
```

---

## Tasks / Subtasks

- [x] **Task 1 — Admin Role Guard** (AC3)
  - [x] 1.1 Create `apps/admin/components/AdminRoleGuard.tsx`
  - [x] 1.2 Create `apps/admin/app/[locale]/403/page.tsx` (minimal 403 page if missing)
  - [x] 1.3 Update `apps/admin/app/[locale]/(protected)/layout.tsx` to nest `AdminRoleGuard` inside `AuthGuard`

- [x] **Task 2 — Navigation Update** (AC2)
  - [x] 2.1 Add 4 new nav items (`tenants`, `crawlers`, `audit-log`, `analytics`) to `adminNavItems` in layout.tsx
  - [x] 2.2 Import `Users2`, `Bot`, `ScrollText`, `TrendingUp` from `lucide-react`

- [x] **Task 3 — API Functions** (AC14)
  - [x] 3.1 Create `apps/admin/lib/api/admin-tenants.ts` (TS interfaces + 5 stub functions)
  - [x] 3.2 Create `apps/admin/lib/api/admin-crawlers.ts` (TS interfaces + 5 stub functions)
  - [x] 3.3 Create `apps/admin/lib/api/admin-audit-log.ts` (TS interfaces + 2 stub functions)
  - [x] 3.4 Create `apps/admin/lib/api/admin-analytics.ts` (TS interfaces + 4 stub functions)

- [x] **Task 4 — TanStack Query Hooks** (AC15)
  - [x] 4.1 Create `apps/admin/lib/queries/use-admin-tenants.ts`
  - [x] 4.2 Create `apps/admin/lib/queries/use-admin-crawlers.ts`
  - [x] 4.3 Create `apps/admin/lib/queries/use-admin-audit-log.ts`
  - [x] 4.4 Create `apps/admin/lib/queries/use-admin-analytics.ts`

- [x] **Task 5 — Tenant Management Pages** (AC4, AC5, AC6, AC10)
  - [x] 5.1 Create `tenants/page.tsx` server shell
  - [x] 5.2 Create `tenants/components/TenantListPage.tsx` (table + filter bar + drawer + tier override form)
  - [x] 5.3 Create `tenants/[id]/white-label/page.tsx` server shell
  - [x] 5.4 Create `tenants/[id]/white-label/components/WhiteLabelPage.tsx`

- [x] **Task 6 — Crawler Management Page** (AC7, AC8, AC9)
  - [x] 6.1 Create `crawlers/page.tsx` server shell
  - [x] 6.2 Create `crawlers/components/CrawlerManagementPage.tsx` (all three sections + modals)

- [x] **Task 7 — Audit Log Page** (AC11, AC12)
  - [x] 7.1 Create `audit-log/page.tsx` server shell
  - [x] 7.2 Create `audit-log/components/AuditLogPage.tsx` (filter bar + table + pagination + CSV export)

- [x] **Task 8 — Platform Analytics Page** (AC13)
  - [x] 8.1 Create `analytics/page.tsx` server shell
  - [x] 8.2 Create `analytics/components/PlatformAnalyticsPage.tsx` (funnel + pie + revenue cards + usage bar chart)
  - [x] 8.3 Verify `recharts` is available in `apps/admin/package.json`; add if missing

- [x] **Task 9 — i18n Keys** (AC16)
  - [x] 9.1 Add 5 new namespaces to `apps/admin/messages/en.json`
  - [x] 9.2 Add 5 new namespaces to `apps/admin/messages/bg.json`

---

## Dev Notes

### Service Architecture

- **Admin API service:** `services/admin-api` (port 8002 in docker-compose). All `/api/v1/admin/*` endpoints are implemented and `done` (S12.11–S12.13).
- **Auth model:** The admin JWT `platform_admin` role claim is stored in the `AdminUser` model as `role: str`. Check `services/admin-api/src/admin_api/core/security.py` for the exact claim field. On the frontend, the `useAuthStore()` user object should expose this — verify the field name against the auth store implementation before writing `AdminRoleGuard`.
- **IP allowlist:** All admin API endpoints reject non-VPN IPs in staging/prod. In local dev (`ADMIN_API_IP_ALLOWLIST=""`) all IPs pass through. No special frontend handling needed.

### Verifying the Auth Store User Role Field

Before implementing `AdminRoleGuard`, inspect:
1. `packages/ui/src/stores/auth-store.ts` (or wherever `useAuthStore` is defined in `@eusolicit/ui`)
2. Look for the shape of the `user` object — it should have a `role` field with value `"platform_admin"` for admin users
3. If the admin app uses a different auth store, check `apps/admin/lib/stores/`

### Admin API Client Pattern

The existing stub functions in `apps/admin/lib/api/` use comments like:
```typescript
// Real: apiClient.get<T>("/api/v1/admin/...", { params })
```
The `apiClient` from `@eusolicit/ui` is configured against the client-api base URL. For admin API calls, a separate admin client targeting `NEXT_PUBLIC_ADMIN_API_URL` (default: `http://admin-api:8002`) may be needed. If no `adminApiClient` exists yet:

1. Create `apps/admin/lib/api/admin-client.ts`:
```typescript
import axios from "axios";
const adminApiClient = axios.create({ baseURL: process.env.NEXT_PUBLIC_ADMIN_API_URL ?? "http://localhost:8002" });
// Add auth interceptor: attach Bearer token from auth store
export default adminApiClient;
```
2. All stub functions' real-call comments should reference `adminApiClient` rather than `apiClient`.

Keep all functions in stub phase for this story. Real wiring will be validated during E2E testing setup.

### Admin API Endpoint Reference

All endpoints are confirmed implemented in S12.11–S12.13 (status: done):

| Endpoint | Method | Story |
|---|---|---|
| `/api/v1/admin/tenants` | GET | S12.11 |
| `/api/v1/admin/tenants/{id}` | GET | S12.11 |
| `/api/v1/admin/tenants/{id}/tier-override` | POST | S12.11 |
| `/api/v1/admin/crawlers/runs` | GET | S12.12 |
| `/api/v1/admin/crawlers/schedule/{type}` | GET, PUT | S12.12 |
| `/api/v1/admin/crawlers/trigger/{type}` | POST | S12.12 |
| `/api/v1/admin/crawlers/runs/{run_id}` | GET | S12.12 |
| `/api/v1/admin/tenants/{id}/white-label` | GET, PUT | S12.12 |
| `/api/v1/admin/audit-logs` | GET | S12.13 |
| `/api/v1/admin/audit-logs/export` | GET (CSV stream) | S12.13 |
| `/api/v1/admin/analytics/funnel` | GET | S12.13 |
| `/api/v1/admin/analytics/tiers` | GET | S12.13 |
| `/api/v1/admin/analytics/usage` | GET | S12.13 |
| `/api/v1/admin/analytics/revenue` | GET | S12.13 |

### Backend Schema Deviations to Note (from S12.11 Dev Agent Record)

From 12-11 story completion notes:
- `client.users` links to companies via `client.company_memberships` (not direct `company_id` column) — this affects how activity data is computed, but the API response shape is unchanged
- `shared.audit_log` uses `timestamp` column (not `created_at`) in the DB, but the Pydantic schema serialises it as `created_at` in the JSON response — use `entry.created_at` in the frontend
- `mv_usage_consumption` consumed/limit_value are `Integer` (not float) in the DB, but come through as numbers in JSON — handle gracefully

### Recharts in Admin App

Recharts is used in `apps/client`. Verify it is a workspace-level dependency:
```
cat eusolicit-app/frontend/apps/admin/package.json | grep recharts
```
If not present, add `"recharts": "*"` to `apps/admin/package.json` and run `pnpm install` from the workspace root.

Import pattern:
```typescript
import { PieChart, Pie, Cell, BarChart, Bar, XAxis, YAxis, Tooltip, Legend } from "recharts";
```

### shadcn/ui Components Required

Verify these are available in `apps/admin` (check `apps/admin/components/ui/`):
- `Sheet`, `SheetContent`, `SheetHeader`, `SheetTitle`, `SheetDescription` — for tenant detail drawer
- `AlertDialog`, `AlertDialogContent`, `AlertDialogTitle`, `AlertDialogDescription`, `AlertDialogCancel`, `AlertDialogAction` — for crawler trigger confirmation
- `Dialog`, `DialogContent` — for run detail modal
- `Switch` — for crawler schedule enabled toggle
- `Select`, `SelectItem`, `SelectContent`, `SelectTrigger`, `SelectValue` — for filter dropdowns

If any are missing, add via `pnpm dlx shadcn-ui@latest add <component>` from `apps/admin/`.

### Project File Structure Reference

```
eusolicit-app/frontend/apps/admin/
├── app/[locale]/
│   ├── 403/
│   │   └── page.tsx                        ← new (if missing)
│   └── (protected)/
│       ├── layout.tsx                      ← modify: AdminRoleGuard + 4 nav items
│       ├── tenants/
│       │   ├── page.tsx                    ← new
│       │   ├── components/
│       │   │   └── TenantListPage.tsx      ← new
│       │   └── [id]/white-label/
│       │       ├── page.tsx                ← new
│       │       └── components/
│       │           └── WhiteLabelPage.tsx  ← new
│       ├── crawlers/
│       │   ├── page.tsx                    ← new
│       │   └── components/
│       │       └── CrawlerManagementPage.tsx ← new
│       ├── audit-log/
│       │   ├── page.tsx                    ← new
│       │   └── components/
│       │       └── AuditLogPage.tsx        ← new
│       └── analytics/
│           ├── page.tsx                    ← new
│           └── components/
│               └── PlatformAnalyticsPage.tsx ← new
├── components/
│   └── AdminRoleGuard.tsx                  ← new
├── lib/
│   ├── api/
│   │   ├── admin-tenants.ts                ← new
│   │   ├── admin-crawlers.ts               ← new
│   │   ├── admin-audit-log.ts              ← new
│   │   └── admin-analytics.ts              ← new
│   └── queries/
│       ├── use-admin-tenants.ts            ← new
│       ├── use-admin-crawlers.ts           ← new
│       ├── use-admin-audit-log.ts          ← new
│       └── use-admin-analytics.ts          ← new
└── messages/
    ├── en.json                             ← modify: 5 new namespaces
    └── bg.json                             ← modify: 5 new namespaces
```

### test_design Reference (from `test-design-epic-12.md`)

**P0 tests** (must pass before PR review):

| Test | Expected |
|------|----------|
| Non-admin authenticated user navigates directly to `/tenants` | Redirected to `/{locale}/403` |
| Non-admin user navigates to `/crawlers`, `/audit-log`, `/analytics` | All redirect to `/{locale}/403` |

**P1 tests** (core functionality):

| Test | Expected |
|------|----------|
| Tenant table: search "acme" → only matching rows shown; click row → drawer opens with subscription/usage/activity | Filtered table + populated drawer |
| Tier override: select new plan + reason ≥ 10 chars → submit | Tier badge in table updates; success toast shown |
| Tier override: submit with reason < 10 chars | Inline `data-testid="drawer-override-reason-error"` shown; API not called |
| Crawler run history table: status badges displayed; click trigger button → confirmation dialog fires; confirm → new run row appears | Trigger flow end-to-end |
| Audit log: apply date range filter → table re-renders filtered; click "Export CSV" → file download triggered | Filter + CSV export |
| Platform analytics: all 4 sections render with non-zero data values | Funnel bars, pie chart, revenue cards, usage bar chart visible |

**P2 tests**:
- White-label preview updates in real time: change `primaryColor` input → `data-testid="whitelabel-preview-box"` background changes without saving
- Platform analytics tier distribution pie chart renders with correct slice colours per plan

### References

- Admin app protected layout: `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/layout.tsx`
- API functions pattern: `eusolicit-app/frontend/apps/admin/lib/api/compliance-frameworks.ts`
- Query hooks pattern: `eusolicit-app/frontend/apps/admin/lib/queries/use-compliance-frameworks.ts`
- Tenant management backend (schemas + endpoints): `eusolicit-docs/implementation-artifacts/12-11-admin-api-tenant-management.md`
- Crawler + white-label backend: `eusolicit-docs/implementation-artifacts/12-12-admin-api-crawler-white-label-management.md`
- Audit log + analytics backend: `eusolicit-docs/implementation-artifacts/12-13-admin-api-audit-log-platform-analytics.md`
- Epic-12 test design (S12.14 P0/P1): `eusolicit-docs/test-artifacts/test-design-epic-12.md`

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5)

### Debug Log References

- TypeScript type check: dynamic i18n keys (template literals) incompatible with next-intl strict typing → resolved by static lookup objects per component
- `EmptyState` component from `@eusolicit/ui` requires `icon: React.ElementType` prop → added `Building2`, `Bot`, `ScrollText` icons to each usage
- `AlertDialog` not available in `@eusolicit/ui` (only `Dialog` exists, no `@radix-ui/react-alert-dialog` in package.json) → created local `apps/admin/components/ui/alert-dialog.tsx` wrapping Dialog primitives
- `recharts` not in admin's `package.json` → added `"recharts": "*"` and manually symlinked from pnpm virtual store (v2.15.4 already in workspace)
- `pnpm` not on PATH → used `node` directly to run tsc and vitest via module path
- ScheduleCard component originally passed `t` function as prop (type mismatch) → moved `useTranslations()` call into the component itself

### Completion Notes List

- ✅ AC1 (Route Structure): All 5 route page shells created as minimal server components with correct param forwarding
- ✅ AC2 (Navigation): 4 new nav items added to `adminNavItems` in protected layout with correct lucide-react icons
- ✅ AC3 (Admin Role Guard): `AdminRoleGuard.tsx` created with spinner/redirect/render logic; 403 page created; layout updated to wrap content in `<AdminRoleGuard>`
- ✅ AC4 (Tenant Table & Filters): `TenantListPage.tsx` with search, tier filter, apply/clear, table with all required data-testids, skeleton loading, empty state, pagination
- ✅ AC5 (Tenant Detail Drawer): Sheet-based drawer with subscription, usage meters, activity sections; all data-testids present
- ✅ AC6 (Tier Override Form): Select, textarea with reason validation (min 10 chars), error display, submit with loading state, success/error toasts
- ✅ AC7 (Crawler Run History): Run table with type filter, status badges, started/completed timestamps, results, pagination
- ✅ AC8 (Schedule Configuration): Per-crawler schedule cards with enabled toggle, hour/minute inputs, save button per card; independent state
- ✅ AC9 (Manual Trigger): 4 trigger buttons, AlertDialog confirmation (built on Dialog primitives), trigger mutation with toast
- ✅ AC10 (White-Label Page): Two-column form + live preview; all form fields with color pickers, DNS badge, save with 409 conflict detection
- ✅ AC11 (Audit Log Filter Bar): All 5 filter inputs, apply/clear, export CSV button with downloading logic
- ✅ AC12 (CSV Export): Blob download via `URL.createObjectURL`, auto filename with date
- ✅ AC13 (Platform Analytics): All 4 sections — funnel bars, PieChart tier distribution, revenue cards grid, BarChart usage
- ✅ AC14 (API Functions): 4 API files with TypeScript interfaces matching backend schemas; realistic stub data; real call comments
- ✅ AC15 (Query Hooks): 4 query hook files with correct queryKeys, staleTime, mutations with cache invalidation
- ✅ AC16 (i18n Keys): 5 new namespaces in both `en.json` and `bg.json`; nav keys added to existing `nav` namespace
- All 10 existing admin app tests pass (check-i18n-keys.test.ts)
- TypeScript: 0 new type errors in new files (21 pre-existing errors in unrelated files unchanged)

### File List

**New files:**
- `eusolicit-app/frontend/apps/admin/components/AdminRoleGuard.tsx`
- `eusolicit-app/frontend/apps/admin/components/ui/alert-dialog.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/403/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/tenants/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/tenants/components/TenantListPage.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/tenants/[id]/white-label/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/tenants/[id]/white-label/components/WhiteLabelPage.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/crawlers/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/crawlers/components/CrawlerManagementPage.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/audit-log/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/audit-log/components/AuditLogPage.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/analytics/page.tsx`
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/analytics/components/PlatformAnalyticsPage.tsx`
- `eusolicit-app/frontend/apps/admin/lib/api/admin-tenants.ts`
- `eusolicit-app/frontend/apps/admin/lib/api/admin-crawlers.ts`
- `eusolicit-app/frontend/apps/admin/lib/api/admin-audit-log.ts`
- `eusolicit-app/frontend/apps/admin/lib/api/admin-analytics.ts`
- `eusolicit-app/frontend/apps/admin/lib/queries/use-admin-tenants.ts`
- `eusolicit-app/frontend/apps/admin/lib/queries/use-admin-crawlers.ts`
- `eusolicit-app/frontend/apps/admin/lib/queries/use-admin-audit-log.ts`
- `eusolicit-app/frontend/apps/admin/lib/queries/use-admin-analytics.ts`

**Modified files:**
- `eusolicit-app/frontend/apps/admin/app/[locale]/(protected)/layout.tsx`
- `eusolicit-app/frontend/apps/admin/messages/en.json`
- `eusolicit-app/frontend/apps/admin/messages/bg.json`
- `eusolicit-app/frontend/apps/admin/package.json`

### Change Log

- 2026-04-13: Implemented Story 12.14 — Admin Frontend Pages. Created 21 new files and modified 4 existing files. Implemented all 9 tasks covering admin role guard, navigation, API stubs, query hooks, all 5 page components (tenant management with drawer, white-label, crawler management, audit log, platform analytics), and i18n keys for both EN and BG locales. Added recharts to package.json for analytics charts.
