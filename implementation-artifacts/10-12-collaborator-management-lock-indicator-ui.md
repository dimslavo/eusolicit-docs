# Story 10.12: Collaborator Management & Lock Indicator UI

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-12-collaborator-management-lock-indicator-ui
- **Points:** 3
- **Type:** frontend
- **Module:** Client web app (`apps/client`) — new components: `CollaboratorPanel`, `AddCollaboratorForm`, `CollaboratorRow`, `SectionLockIndicator`, `LockedSectionToast`; extended components: `ProposalWorkspacePage` (toolbar + slide-over mount), `ProposalEditor` (lock-aware editor wiring), `ProposalSection` (acquire/release on focus/blur, lock-state rendering); new API client: `apps/client/lib/api/collaborators.ts`, `apps/client/lib/api/section-locks.ts`, `apps/client/lib/api/members.ts`; new queries: `apps/client/lib/queries/use-collaborators.ts`, `apps/client/lib/queries/use-section-locks.ts`, `apps/client/lib/queries/use-members.ts`; new state: `apps/client/lib/stores/collaborator-store.ts` (current user's proposal role + lock map); i18n: `apps/client/messages/{en,bg}.json` (new `proposals.collaborators.*` and `proposals.locks.*` namespaces); ATDD test: `apps/client/__tests__/collaborator-lock-s10-12.test.ts`.
- **Priority:** P1 (Epic 10 frontend surface — first user-visible payoff for the backend stories 10.1–10.3 already in `done` and unlocks Story 10.13 Comments Sidebar UI which depends on the same role-aware proposal editor scaffolding).
- **Depends On:** Story 10.1 (Proposal Collaborator CRUD API — `POST/GET/PATCH/DELETE /api/v1/proposals/{proposal_id}/collaborators` and the `CollaboratorResponse` shape returning `user_full_name`, `user_email`, `role`, `granted_by`, `granted_at`), Story 10.2 (proposal-level RBAC — the server-side gate that returns `403` when a non-collaborator attempts a section-lock POST; the frontend mirrors the same role-set for client-side gating to suppress UI affordances), Story 10.3 (Section Locking API — `POST /api/v1/proposals/{proposal_id}/sections/{section_key}/lock` returning `200` / `423`, `DELETE /api/v1/proposals/{proposal_id}/sections/{section_key}/lock` returning `204`, and `GET /api/v1/proposals/{proposal_id}/sections` enriched with `lock` field per section — the `15`-minute TTL is server-enforced via `expires_at`), Story 2.9 (Team member management — `GET /api/v1/companies/{company_id}/members` returning the candidate user list driving the add-collaborator search/picker), Story 7.11 / 7.12 (Proposal Workspace + Tiptap editor — this story wires into the existing `ProposalWorkspacePage`, `ProposalEditor`, and `ProposalSection` components and the `useProposalEditorStore` singleton), Story 3.5 (Zustand stores + TanStack Query setup — pattern reuse), Story 3.6 (React Hook Form + Zod patterns — reused for the AddCollaborator form), Story 3.7 (next-intl bg/en setup — both new i18n namespaces ship in parity), Story 3.10 (loading / empty / error states), Story 3.11 (toast notification system — `useUIStore.addToast` for the lock-conflict and 403 / 5xx toasts).

## Story

As a **bid_manager or contributor on a proposal in the EU Solicit workspace**,
I want **to see, add, and remove proposal collaborators from a slide-over panel on the proposal workspace page, and to see live section-lock indicators (with the editor's name and an expiry countdown) on every section header in the Tiptap editor — with locks acquired automatically when I focus a section, released on blur or page navigation, and refreshed every 30 seconds via polling**,
so that **my team can coordinate concurrent work on a single proposal without overwriting each other; non-bid_manager teammates can collaborate within their roles without seeing manage-controls they cannot exercise; and contention is communicated visually (lock icon + holder name + remaining TTL) and conversationally (toast on attempted edit of a foreign-locked section) instead of failing silently with a `423` no one understands.**

## Description

This story delivers the first user-visible surface of Epic 10's backend pessimistic-collaboration model. The backend section-lock and collaborator-CRUD APIs (Stories 10.1, 10.2, 10.3) are already in `done` — this story consumes them from the existing proposal workspace.

It ships **two related-but-decoupled UI surfaces** plus a thin shared state layer:

1. **Collaborator Management Slide-over** — opened from a new toolbar button on `ProposalWorkspacePage`. Lists every collaborator (avatar initials, full name, role badge, granted-by attribution) and offers an "Add collaborator" form (email/user search against `/api/v1/companies/{company_id}/members` + role dropdown) plus a per-row remove button. Add / remove / role-change controls render only when the current user is a `bid_manager` on this proposal OR a company `admin`; for any other proposal collaborator role (`technical_writer`, `financial_analyst`, `legal_reviewer`, `read_only`) the panel is read-only — the list is still visible (transparency feature) but the controls disappear.

2. **Section Lock Indicators in the Tiptap Editor** — every `ProposalSection` header gains a lock-state stripe. When `lock` is `null` (no active lock), the header shows nothing extra. When `lock.locked_by === current_user.user_id`, the header shows a green "Editing" pill with the remaining TTL countdown ("Editing — 14:32 left"). When `lock.locked_by !== current_user.user_id` AND `lock.expires_at > now`, the header shows a red lock icon, the holder's name, and the remaining TTL ("Locked by Maria Petrova — 12:08 left"); the section editor is set to non-editable (`editor.setEditable(false)`) and any click into the section emits a toast (`useUIStore.addToast({ type: "warning", title: ..., description: ... })`) instead of stealing focus.

3. **Auto acquire / release wiring** — the `ProposalSection` component gets a focus handler that POSTs the lock-acquire endpoint (debounced to coalesce burst focus events) and a blur handler that DELETEs it (also debounced; cancelled if focus returns to the same section within 800 ms to avoid release-then-reacquire flap). On `pagehide` / route change / tab-close (`visibilitychange === "hidden"` for `> 5 s`), every lock the current user owns on this proposal is released best-effort via `navigator.sendBeacon()`-fallback `fetch({ keepalive: true })`. This matches the 15-minute server TTL safety net — the beacon is best-effort, the TTL is the durable cleanup.

4. **30-second polling** — `useSectionLocks(proposalId)` is a TanStack Query `useQuery` with `refetchInterval: 30_000` and `refetchOnWindowFocus: true`. It hits `GET /api/v1/proposals/{proposal_id}/sections` (the AC3-enriched listing), extracts the `lock` field per section, normalises it into a `Map<sectionKey, SectionLockInfo>` exposed via a Zustand selector, and is consumed by every `ProposalSection` and the toolbar's lock summary. The polling stops automatically when the workspace tab is in the background (TanStack Query's default `refetchIntervalInBackground: false` semantics — explicit for clarity).

Five design decisions worth calling out:

**(a) Slide-over, not modal — chosen for "always visible while you work" semantics.** The collaborator panel does NOT block the editor; the user can drag a section while the panel is open to see who else is in it. We use the existing `Sheet` / `Drawer` primitive from `@eusolicit/ui` (already shipped via shadcn/ui in Story 3.2 and used by `VersionHistorySidebar` in Story 7.16) for visual + behavioural parity. The right-edge slide-in matches the version-history convention; the toggle button lives in the existing `ProposalToolbar` between "History" and any future buttons.

**(b) The current user's proposal role is sourced from the collaborator list, NOT from a separate endpoint.** The `useCollaborators(proposalId)` query returns the full list; the frontend looks up `current_user.user_id` in `items[]` to derive `myRole`. If the current user is a company `admin` and not present in the list, `myRole` is synthesised as `"admin"` (a non-collaborator-role marker) and ALL controls remain enabled (admins can manage any proposal in their own company per the backend contract). If the current user is neither a collaborator nor admin, `myRole === null` — the panel is opened but shows an empty state ("You do not have access to this proposal" — should not happen because the proposal page itself would have 404'd, but defensive). This avoids a second round-trip and keeps the role state in sync with the same query that drives the list.

**(c) Lock acquisition is fire-and-forget; the editor stays editable optimistically until the response arrives.** When the user focuses a section, we (i) optimistically mark the local `localLockState[sectionKey] = "acquiring"`, (ii) POST the lock, (iii) on `200` resolve to `"held"` with the returned `expires_at` and the editor stays editable, (iv) on `423` resolve to `"foreign"`, set the editor to non-editable, surface the conflict toast, and force a refetch of `useSectionLocks` so the UI converges. This avoids the perceived input lag of "wait for lock before letting me type" — typing during the round-trip is captured by the editor's debounced auto-save (Story 7.12) and saves only fire after the lock is confirmed. The Story 7.12 `doAutoSave` path is gated by an additional precondition: `localLockState[sectionKey] === "held"` — if not held, the auto-save is suppressed and the user sees a "Cannot save — section is locked by …" toast.

**(d) The `15`-minute TTL is displayed as a live countdown that ticks every `1`s on the SAME section the current user holds; for foreign locks it ticks every `5`s to keep render churn low.** The countdown component is its own memoised child (`<LockCountdown expiresAt={...} interval={1000 | 5000} />`) so a `1`s tick doesn't re-render the whole `ProposalSection`. When the countdown reaches `0`, the component re-fires `refetchSectionLocks` once and self-stops (no infinite re-render loop). Polling refresh (every `30`s) is the source of truth — the countdown is derived display only.

**(e) `myRole`-driven UI gating mirrors the backend RBAC contract exactly.** The five `ProposalCollaboratorRole` values (`bid_manager | technical_writer | financial_analyst | legal_reviewer | read_only`) plus the company-level `admin` bypass map to UI affordances as follows:
| Action | bid_manager | tech_writer | fin_analyst | legal_reviewer | read_only | admin |
|--------|:-:|:-:|:-:|:-:|:-:|:-:|
| View collaborator list | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Add collaborator | ✓ | — | — | — | — | ✓ |
| Remove collaborator | ✓ | — | — | — | — | ✓ |
| Change role | ✓ | — | — | — | — | ✓ |
| Acquire section lock | ✓ | ✓ | ✓ | ✓ | — | ✓ |
| Force-release foreign lock | ✓ | — | — | — | — | ✓ |
| Edit section content | ✓ | ✓ | ✓ | ✓ | — | ✓ |

The backend remains authoritative — the UI hides controls for clean UX, but every action would 403 anyway if a hostile client bypassed the gate.

The slide-over and lock indicators are independent on the page (the toolbar button toggles the panel; the indicators always render) and share only the `useCollaborators` query for the `myRole` lookup. Either feature can be deployed independently behind a feature flag if rollback is needed.

## Acceptance Criteria

### File-system & Routing

1. [ ] **AC1 — Six new component files plus three new API/query/store files exist under `apps/client/`.**

    Components:
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/CollaboratorPanel.tsx` (`"use client"`; default export `CollaboratorPanel`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/CollaboratorRow.tsx` (`"use client"`; named export `CollaboratorRow`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/AddCollaboratorForm.tsx` (`"use client"`; named export `AddCollaboratorForm`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/SectionLockIndicator.tsx` (`"use client"`; named exports `SectionLockIndicator`, `LockCountdown`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/LockedSectionToastBody.tsx` (`"use client"`; named export `LockedSectionToastBody`).

    API / queries / store:
    - `apps/client/lib/api/collaborators.ts` (`listCollaborators`, `addCollaborator`, `updateCollaboratorRole`, `removeCollaborator`, plus the four interfaces enumerated in AC2).
    - `apps/client/lib/api/section-locks.ts` (`listSectionsWithLocks`, `acquireSectionLock`, `releaseSectionLock`, plus the three interfaces enumerated in AC3).
    - `apps/client/lib/api/members.ts` (`listCompanyMembers`, returning `{ id, email, full_name, role, status }[]` mirroring the Story 2.9 `MemberResponse` schema; reused by `AddCollaboratorForm` for the user picker).
    - `apps/client/lib/queries/use-collaborators.ts` (exports `useCollaborators`, `useMyProposalRole`, `useAddCollaborator`, `useUpdateCollaboratorRole`, `useRemoveCollaborator`).
    - `apps/client/lib/queries/use-section-locks.ts` (exports `useSectionLocks`, `useAcquireSectionLock`, `useReleaseSectionLock`).
    - `apps/client/lib/queries/use-members.ts` (exports `useCompanyMembers`).
    - `apps/client/lib/stores/collaborator-store.ts` (exports `useCollaboratorStore`; see AC10).

    Modified files:
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalWorkspacePage.tsx` — adds the toolbar "Collaborators" button + state for `collaboratorPanelOpen` + mounts `<CollaboratorPanel proposalId={proposal.id} companyId={proposal.company_id} isOpen={collaboratorPanelOpen} onClose={() => setCollaboratorPanelOpen(false)} />`.
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalEditor.tsx` — passes `proposalId` through to `ProposalSection` (already passed) and gates `doAutoSave` on `localLockState[key] === "held"` per AC8.
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalSection.tsx` — adds the `<SectionLockIndicator />` render in the header, registers focus / blur handlers that call the acquire / release mutations, and switches the editor's `editable` flag based on the lock state per AC7.

2. [ ] **AC2 — `apps/client/lib/api/collaborators.ts` exports.**

    ```ts
    export type ProposalCollaboratorRoleValue =
      | "bid_manager"
      | "technical_writer"
      | "financial_analyst"
      | "legal_reviewer"
      | "read_only";

    export interface CollaboratorResponse {
      id: string;
      proposal_id: string;
      user_id: string;
      user_full_name: string | null;
      user_email: string;
      role: ProposalCollaboratorRoleValue;
      granted_by: string | null;
      granted_by_full_name: string | null;
      granted_at: string;       // ISO-8601
      updated_at: string;       // ISO-8601
    }

    export interface CollaboratorListResponse {
      items: CollaboratorResponse[];
      total: number;
    }

    export interface AddCollaboratorRequest {
      user_id: string;
      role: ProposalCollaboratorRoleValue;
    }

    export interface UpdateCollaboratorRoleRequest {
      role: ProposalCollaboratorRoleValue;
    }
    ```

    Functions (all use the shared `apiClient` from `@eusolicit/ui`):
    - `listCollaborators(proposalId): Promise<CollaboratorListResponse>` — `GET /api/v1/proposals/{proposalId}/collaborators`.
    - `addCollaborator(proposalId, payload: AddCollaboratorRequest): Promise<CollaboratorResponse>` — `POST /api/v1/proposals/{proposalId}/collaborators`.
    - `updateCollaboratorRole(proposalId, userId, payload: UpdateCollaboratorRoleRequest): Promise<CollaboratorResponse>` — `PATCH /api/v1/proposals/{proposalId}/collaborators/{userId}`.
    - `removeCollaborator(proposalId, userId): Promise<void>` — `DELETE /api/v1/proposals/{proposalId}/collaborators/{userId}` (returns `204`).

    All functions throw the raw `AxiosError` on non-2xx; callers map `response.status` to user-facing error messages via the standard `error-utils.ts` helpers.

3. [ ] **AC3 — `apps/client/lib/api/section-locks.ts` exports.**

    ```ts
    export interface SectionLockInfo {
      locked_by: string;                  // UUID
      locked_by_full_name: string | null;
      acquired_at: string;                // ISO-8601
      expires_at: string;                 // ISO-8601
    }

    export interface SectionWithLock {
      key: string;
      title: string;
      body: string;
      lock: SectionLockInfo | null;
    }

    export interface SectionsListResponse {
      proposal_id: string;
      version_id: string | null;
      sections: SectionWithLock[];
    }

    export interface SectionLockAcquireResponse {
      proposal_id: string;
      section_key: string;
      locked_by: string;
      locked_by_full_name: string | null;
      acquired_at: string;
      expires_at: string;
    }

    export interface SectionLockConflictDetail {
      error: "section_locked";
      proposal_id: string;
      section_key: string;
      locked_by: string;
      locked_by_full_name: string | null;
      expires_at: string;
    }
    ```

    Functions:
    - `listSectionsWithLocks(proposalId): Promise<SectionsListResponse>` — `GET /api/v1/proposals/{proposalId}/sections`.
    - `acquireSectionLock(proposalId, sectionKey): Promise<SectionLockAcquireResponse | { conflict: SectionLockConflictDetail }>` — `POST /api/v1/proposals/{proposalId}/sections/{sectionKey}/lock`. On `423` the function does NOT throw — it returns `{ conflict: <body> }`; on every other non-2xx it throws.
    - `releaseSectionLock(proposalId, sectionKey): Promise<void>` — `DELETE /api/v1/proposals/{proposalId}/sections/{sectionKey}/lock`. On `204` resolves; on `403` (caller does not hold the lock and is not a bid_manager / admin) throws — the calling mutation surfaces a toast.
    - `releaseSectionLockBeacon(proposalId, sectionKey): boolean` — synchronous, uses `navigator.sendBeacon(url, "")` if available, else falls back to `fetch(url, { method: "DELETE", keepalive: true, credentials: "include" })`. Used inside the `pagehide` / `visibilitychange` handler. Returns `true` if a beacon was queued; the result is informational only.

4. [ ] **AC4 — `apps/client/lib/queries/use-collaborators.ts` exports.**

    ```ts
    export function useCollaborators(proposalId: string)
        // useQuery({ queryKey: ["collaborators", proposalId], queryFn: () => listCollaborators(proposalId), staleTime: 30_000, enabled: Boolean(proposalId) })

    export function useMyProposalRole(proposalId: string):
        { role: ProposalCollaboratorRoleValue | "admin" | null; isLoading: boolean }
        // Subscribes to useAuthStore.user, useCollaborators(proposalId).
        // - If user.role === "admin" → { role: "admin", isLoading: <collaborators-loading> }
        // - Else if collaborators.items has a row with user_id === user.id → { role: <that row.role>, isLoading: false }
        // - Else → { role: null, isLoading: <collaborators-loading> }

    export function useAddCollaborator(proposalId: string)
        // useMutation; onSuccess invalidates ["collaborators", proposalId].
        // 409 surfaces toast "duplicateCollaborator"; 422 surfaces "memberInactive";
        // 403 surfaces "notAuthorisedToAdd"; 5xx surfaces "serverError".

    export function useUpdateCollaboratorRole(proposalId: string)
        // useMutation; onSuccess invalidates ["collaborators", proposalId].
        // 409 → "lastBidManager" toast; 403 → "notAuthorised"; 404 → "collaboratorGone" + invalidate.

    export function useRemoveCollaborator(proposalId: string)
        // useMutation; onSuccess invalidates ["collaborators", proposalId].
        // 409 → "lastBidManager"; 403 → "notAuthorised"; 404 → "collaboratorGone" + invalidate.
    ```

    Each mutation uses `useUIStore.addToast` for error UX (titles + descriptions are i18n keys per AC11). On 5xx the error toast description is `t("forms.serverError")` and the error is logged to `console.error` with `proposalId` + `userId` context.

5. [ ] **AC5 — `apps/client/lib/queries/use-section-locks.ts` exports.**

    ```ts
    export function useSectionLocks(proposalId: string)
        // useQuery({
        //   queryKey: ["section-locks", proposalId],
        //   queryFn: () => listSectionsWithLocks(proposalId),
        //   refetchInterval: 30_000,                 // 30 s polling per spec
        //   refetchIntervalInBackground: false,
        //   refetchOnWindowFocus: true,
        //   staleTime: 0,                            // always treat as stale → ensures focus refetch fires
        //   enabled: Boolean(proposalId),
        // })

    export function useAcquireSectionLock(proposalId: string)
        // useMutation; on 200 invalidates ["section-locks", proposalId];
        // on conflict-result (the { conflict } shape from acquireSectionLock) does NOT invalidate
        // but writes the conflict into useCollaboratorStore.setForeignLockConflict(sectionKey, conflict)
        // so the indicator + toast can surface immediately, then invalidates ["section-locks", proposalId].

    export function useReleaseSectionLock(proposalId: string)
        // useMutation; onSettled invalidates ["section-locks", proposalId].
    ```

    The query's `select` option normalises the response into a `Map<string, SectionLockInfo | null>` keyed by `section.key` and exposes a `byKey(key)` helper via the returned data. (Implementation may store the map alongside `data.sections` to keep the raw list available for the toolbar's "N collaborators editing" badge — see AC9.)

6. [ ] **AC6 — `CollaboratorPanel` (slide-over).**

    Props: `{ proposalId: string; companyId: string; isOpen: boolean; onClose: () => void }`.

    Renders a right-edge `<Sheet side="right" open={isOpen} onOpenChange={(open) => !open && onClose()}>` (or the project's existing slide-over primitive — match `VersionHistorySidebar`'s implementation pattern). Width: `w-96` on desktop, `w-full` on mobile. Root `data-testid="collaborator-panel"`.

    Header (`data-testid="collaborator-panel-header"`):
    - `<h2 data-testid="collaborator-panel-title">` with `t("collaborators.panelTitle")`.
    - `<button data-testid="collaborator-panel-close-btn">` with `<X />` icon and `aria-label={t("collaborators.closeBtn")}`.

    Body — three states driven by `useCollaborators(proposalId)`:
    - **Loading** — `<SkeletonCard data-testid="collaborator-panel-skeleton" rows={3} />` (or the project's existing skeleton primitive).
    - **Error** — `<EmptyState data-testid="collaborator-panel-error" icon={AlertCircle} title={t("collaborators.errorTitle")} description={t("collaborators.errorDescription")} />`.
    - **Success** — render the list section + the add form (the latter conditionally per AC6.4).

    6.1 — When `data.items.length === 0`, render `<EmptyState data-testid="collaborator-panel-empty" icon={Users} title={t("collaborators.emptyTitle")} description={t("collaborators.emptyDescription")} />`.

    6.2 — When `data.items.length > 0`, render a `<ul data-testid="collaborator-list">` of `<CollaboratorRow key={c.id} collaborator={c} canManage={canManage} onRoleChange={...} onRemove={...} />` for each `c` in `data.items` (sorted by role priority: `bid_manager` first, then alphabetical by `user_full_name ?? user_email`).

    6.3 — `canManage = useMyProposalRole(proposalId).role === "bid_manager" || useMyProposalRole(proposalId).role === "admin"`.

    6.4 — When `canManage === true`, render `<AddCollaboratorForm proposalId={proposalId} companyId={companyId} existingCollaboratorUserIds={data.items.map(c => c.user_id)} />` below the list (separated by a `<hr />` and an `<h3 data-testid="add-collaborator-section-title">` with `t("collaborators.addSectionTitle")`).

    6.5 — When `canManage === false`, render an inline note (`data-testid="collaborator-panel-readonly-note"`) with text `t("collaborators.readOnlyNote")` ("Only proposal bid managers can add or remove collaborators.").

7. [ ] **AC7 — `CollaboratorRow`.**

    Props: `{ collaborator: CollaboratorResponse; canManage: boolean; onRoleChange: (userId: string, role: ProposalCollaboratorRoleValue) => void; onRemove: (userId: string) => void }`.

    Renders a `<li data-testid={\`collaborator-row-${collaborator.user_id}\`}>` with:
    - **Avatar** — `<Avatar data-testid={\`collaborator-avatar-${userId}\`}>` showing the user's initials computed from `user_full_name ?? user_email` (first letter of first two whitespace-separated tokens; fall back to first two letters of `user_email`).
    - **Name + email** — `<div>` with `<span data-testid={\`collaborator-name-${userId}\`}>{user_full_name ?? user_email}</span>` and a `<span data-testid={\`collaborator-email-${userId}\`} className="text-xs text-muted-foreground">{user_email}</span>` (only render if `user_full_name` is not null AND not equal to `user_email`).
    - **Role badge** — `<Badge data-testid={\`collaborator-role-badge-${userId}\`}>` displaying `t(\`collaborators.roles.${role}\`)`. Variant by role: `bid_manager` → `default` (filled primary), `technical_writer` / `financial_analyst` / `legal_reviewer` → `secondary`, `read_only` → outlined neutral.
    - **Role select** (only when `canManage === true`) — `<Select data-testid={\`collaborator-role-select-${userId}\`} value={role} onValueChange={(newRole) => onRoleChange(userId, newRole)}>` with `<SelectItem>` entries for the five `ProposalCollaboratorRole` values, each labelled by `t(\`collaborators.roles.${value}\`)`. The role badge above is hidden in this case (the select replaces it).
    - **Remove button** (only when `canManage === true`) — `<Button variant="ghost" size="sm" data-testid={\`collaborator-remove-btn-${userId}\`} onClick={() => onRemove(userId)} aria-label={t("collaborators.removeBtnAria", { name })}>` with a `<Trash2 />` icon.
    - **Granted-by tooltip** — `<Tooltip data-testid={\`collaborator-granted-tooltip-${userId}\`}>` on the role badge explaining "Added by {granted_by_full_name ?? \"system\"} on {granted_at}" (formatted via `new Date(granted_at).toLocaleDateString(locale)`). When `granted_by` is null (seeded by system), tooltip shows `t("collaborators.grantedBySystem")`.

    The remove button calls `onRemove(userId)` which opens a confirmation `<Dialog data-testid="collaborator-remove-dialog">` with title `t("collaborators.removeDialogTitle")` and description `t("collaborators.removeDialogDescription", { name })`. Confirmation triggers `useRemoveCollaborator.mutate(userId)`. The dialog closes on success; on `409` ("last bid_manager") the dialog stays open and surfaces an inline error from `t("collaborators.lastBidManagerError")`.

8. [ ] **AC8 — `AddCollaboratorForm`.**

    Props: `{ proposalId: string; companyId: string; existingCollaboratorUserIds: string[] }`.

    Renders a `<form data-testid="add-collaborator-form" onSubmit={handleSubmit}>`. State managed via `useZodForm` (see Story 3.6 pattern) with schema:
    ```ts
    const schema = z.object({
      user_id: z.string().uuid({ message: "Select a user from the list" }),
      role: z.enum([
        "bid_manager",
        "technical_writer",
        "financial_analyst",
        "legal_reviewer",
        "read_only",
      ]),
    });
    ```

    Fields:
    - **User search input** — `<Combobox data-testid="add-collaborator-user-search">` (or the project's existing user-picker primitive — if none exists, implement as a `<Popover>` + `<CommandList>` shadcn pattern). The dropdown is populated by `useCompanyMembers(companyId)` filtered client-side to (i) `status === "active"` AND (ii) `id NOT IN existingCollaboratorUserIds` — already-added users are excluded so the user can't double-add. The list shows `{full_name} ({email})` per row; selecting a row sets the form's `user_id` field. Placeholder: `t("collaborators.userSearchPlaceholder")`. Empty-results state shows `t("collaborators.userSearchNoResults")`.
    - **Role select** — `<Select data-testid="add-collaborator-role-select" name="role" defaultValue="technical_writer">` with `<SelectItem>` entries for the five `ProposalCollaboratorRole` values, labelled by `t(\`collaborators.roles.${value}\`)`. Default value: `technical_writer` (least-privileged write role; bid_manager is reserved for explicit promotion).
    - **Submit button** — `<Button type="submit" data-testid="add-collaborator-submit-btn" disabled={mutation.isPending || !form.formState.isValid}>` with label `t("collaborators.addBtn")`. While in-flight: `<Loader2 className="animate-spin h-4 w-4 mr-2" />` + `t("collaborators.addingBtn")`.

    On submit success: form is reset (`form.reset({ user_id: "", role: "technical_writer" })`); a success toast fires `addToast({ type: "success", title: t("collaborators.addSuccessTitle"), description: t("collaborators.addSuccessDescription", { name }) })`. On submit error the form is NOT reset (the user can correct + retry); the error toast title is `t("collaborators.addErrorTitle")` and description is one of: `t("collaborators.duplicateCollaborator")` (409), `t("collaborators.memberInactive")` (422), `t("collaborators.notAuthorisedToAdd")` (403), `t("forms.serverError")` (5xx).

    The `ProposalEditor` auto-save preconditions get a NEW gate (see AC1 modified `ProposalEditor.tsx`): inside `doAutoSave`, before issuing the PATCH:
    ```ts
    const lockState = useCollaboratorStore.getState().localLockState[key];
    if (lockState !== "held") {
      // Suppress the auto-save — we don't hold the lock.
      addToast({
        type: "warning",
        title: t("locks.cannotSaveTitle"),
        description: t("locks.cannotSaveDescription", { sectionTitle }),
      });
      return;
    }
    ```
    Symmetric guard in `doFullSave` — full-save is only attempted if EVERY section the user edited has `lockState === "held"`. If any section is foreign-locked, the full-save is skipped and a single toast lists the affected section titles.

9. [ ] **AC9 — `SectionLockIndicator` + `LockCountdown`.**

    `SectionLockIndicator` props: `{ lock: SectionLockInfo | null; isCurrentUser: boolean; sectionKey: string; sectionTitle: string }`.

    Render rules — the indicator lives inside the section header, to the right of the title:

    9.1 — `lock === null` → render nothing (returns `null`).

    9.2 — `lock !== null && isCurrentUser === true` → render `<span data-testid={\`section-lock-self-${sectionKey}\`} className="ml-2 inline-flex items-center gap-1 rounded bg-emerald-100 px-2 py-0.5 text-xs font-medium text-emerald-800">`, content: `<Pencil className="h-3 w-3" />` + `t("locks.editingPill")` ("Editing") + ` · ` + `<LockCountdown expiresAt={lock.expires_at} interval={1000} />`. ARIA: `aria-label={t("locks.editingPillAria", { remaining })}`.

    9.3 — `lock !== null && isCurrentUser === false` → render `<span data-testid={\`section-lock-foreign-${sectionKey}\`} className="ml-2 inline-flex items-center gap-1 rounded bg-rose-100 px-2 py-0.5 text-xs font-medium text-rose-800">`, content: `<Lock className="h-3 w-3" />` + `t("locks.lockedByPill", { name: lock.locked_by_full_name ?? t("locks.unknownUser") })` + ` · ` + `<LockCountdown expiresAt={lock.expires_at} interval={5000} />`. ARIA: `aria-label={t("locks.foreignPillAria", { name, remaining })}`. The pill is wrapped in a `<Tooltip>` (`data-testid={\`section-lock-foreign-tooltip-${sectionKey}\`}`) showing `t("locks.foreignTooltip", { name, expiresAt })` formatted as `${full_name} — locks until ${time}`.

    `LockCountdown` props: `{ expiresAt: string; interval: 1000 | 5000 }`. Renders the remaining TTL formatted as `MM:SS` (e.g. "12:08"). Uses `setInterval(tick, interval)` registered in a `useEffect` cleanup-pair. When `remaining <= 0`, renders `t("locks.expiredLabel")` ("expired") and stops the interval. Memoised with `React.memo` so re-renders are scoped.

    `LockedSectionToastBody` is a tiny presentational component used as the `description` of the conflict toast: shows `<Lock className="h-4 w-4" />` + `{name}` + countdown derived from `expiresAt`. Used by the section-click handler in `ProposalSection` (AC10).

    Toolbar lock badge — extension to `ProposalToolbar`: when `useSectionLocks(proposalId).data.sections` contains ≥ 1 lock not held by the current user, the toolbar shows a `<Badge data-testid="toolbar-lock-summary-badge" variant="outline">` with `<Lock className="h-3 w-3 mr-1" />` + `t("locks.toolbarSummary", { count })`. Clicking the badge opens the `CollaboratorPanel`.

10. [ ] **AC10 — `useCollaboratorStore` (Zustand).**

    Lives at `apps/client/lib/stores/collaborator-store.ts`. State shape:
    ```ts
    type LocalLockState = "idle" | "acquiring" | "held" | "foreign" | "releasing";

    interface CollaboratorStoreState {
      // Per-section local lock state — mirrors the server but tracks the
      // optimistic / acquiring / releasing transitions that are not visible in
      // the polled snapshot.
      localLockState: Record<string, LocalLockState>;
      setLocalLockState: (sectionKey: string, state: LocalLockState) => void;
      clearLocalLockStates: () => void;

      // Cached foreign-lock conflict from the most recent acquireSectionLock 423.
      // Surfaced in the section-click toast; cleared on next successful acquire
      // or 30-s poll refresh.
      foreignLockConflicts: Record<string, SectionLockConflictDetail | null>;
      setForeignLockConflict: (sectionKey: string, conflict: SectionLockConflictDetail | null) => void;

      // Pending blur-release timers — keyed by section_key. Used by the
      // 800 ms blur-debounce in ProposalSection (AC11) so a release can be
      // cancelled if focus returns to the same section before the timer fires.
      pendingReleaseTimers: Record<string, ReturnType<typeof setTimeout> | null>;
      setPendingReleaseTimer: (sectionKey: string, timer: ReturnType<typeof setTimeout> | null) => void;
    }

    export const useCollaboratorStore = create<CollaboratorStoreState>()(/* no persist */);
    ```

    No `persist` middleware — lock state should NOT survive page reload (the server is the source of truth on next mount). Reset cleanly on `ProposalEditor` unmount via `clearLocalLockStates()` to avoid stale state when navigating between proposals.

11. [ ] **AC11 — `ProposalSection` modifications.**

    Inputs (existing): `section`, `isGenerating`, `onUpdate`, `onFocus`. New deps used internally: `useSectionLocks(proposalId)`, `useAcquireSectionLock(proposalId)`, `useReleaseSectionLock(proposalId)`, `useAuthStore` (for `current_user.user_id`), `useCollaboratorStore` (for `localLockState` + `pendingReleaseTimers` + `foreignLockConflicts`), `useUIStore` (for `addToast`), `useTranslations("proposals")`, plus `proposalId` (now passed down from `ProposalEditor` — already done in the existing prop chain).

    Behaviour:

    11.1 — **Render the lock indicator inside the existing header** to the right of the title:
    ```tsx
    <h3 data-testid={`section-header-${section.key}`} className="...">
      {section.title}
      <SectionLockIndicator
        lock={lockByKey(section.key)}
        isCurrentUser={lockByKey(section.key)?.locked_by === currentUserId}
        sectionKey={section.key}
        sectionTitle={section.title}
      />
    </h3>
    ```
    `lockByKey` is derived from `useSectionLocks(proposalId).data?.sections` via a `useMemo` that builds a `Map<key, SectionLockInfo | null>`.

    11.2 — **Editor `editable` flag** — instead of just `editable: !isGenerating`, the section is editable iff: `(!isGenerating) && (lockByKey(section.key) === null || lockByKey(section.key)?.locked_by === currentUserId)`. The existing `useEffect` that syncs `editor.setEditable(!isGenerating)` is replaced with one that also depends on `lockByKey(section.key)?.locked_by`.

    11.3 — **Focus handler — acquire the lock** (replaces the existing simple `onFocus`):
    ```ts
    onFocus: () => {
      onFocus(section.key);                              // existing — drives toolbar
      setActiveSectionKey(section.key);                  // existing — store
      // Cancel any pending release for this same section (debounce coalesce)
      const pending = useCollaboratorStore.getState().pendingReleaseTimers[section.key];
      if (pending) {
        clearTimeout(pending);
        useCollaboratorStore.getState().setPendingReleaseTimer(section.key, null);
      }
      // Skip acquire if we already hold it (poll told us so) or if a foreign
      // lock is active (the editor is non-editable anyway, so onFocus shouldn't
      // even fire — but defensive)
      const cur = lockByKey(section.key);
      if (cur && cur.locked_by !== currentUserId) {
        // Show the locked-by toast + clear focus on this section
        addToast({
          type: "warning",
          title: t("locks.cannotEditTitle"),
          description: t("locks.cannotEditDescription", { name: cur.locked_by_full_name ?? t("locks.unknownUser"), sectionTitle: section.title }),
        });
        editor?.commands.blur();
        return;
      }
      if (cur && cur.locked_by === currentUserId) return; // already held
      // Optimistic acquiring
      useCollaboratorStore.getState().setLocalLockState(section.key, "acquiring");
      acquireMutation.mutate(section.key, {
        onSuccess: (resp) => {
          if ("conflict" in resp) {
            useCollaboratorStore.getState().setLocalLockState(section.key, "foreign");
            useCollaboratorStore.getState().setForeignLockConflict(section.key, resp.conflict);
            addToast({
              type: "warning",
              title: t("locks.conflictTitle"),
              description: t("locks.conflictDescription", { name: resp.conflict.locked_by_full_name ?? t("locks.unknownUser") }),
            });
            editor?.setEditable(false);
            editor?.commands.blur();
          } else {
            useCollaboratorStore.getState().setLocalLockState(section.key, "held");
          }
        },
        onError: () => {
          useCollaboratorStore.getState().setLocalLockState(section.key, "idle");
          addToast({ type: "error", title: t("locks.acquireFailedTitle"), description: t("forms.serverError") });
        },
      });
    },
    ```

    11.4 — **Blur handler — release the lock with 800 ms debounce** (newly added):
    ```ts
    onBlur: () => {
      // Schedule the release; cancellable by a quick re-focus.
      const timer = setTimeout(() => {
        useCollaboratorStore.getState().setLocalLockState(section.key, "releasing");
        releaseMutation.mutate(section.key, {
          onSettled: () => {
            useCollaboratorStore.getState().setLocalLockState(section.key, "idle");
            useCollaboratorStore.getState().setPendingReleaseTimer(section.key, null);
          },
        });
      }, 800);
      useCollaboratorStore.getState().setPendingReleaseTimer(section.key, timer);
    },
    ```
    Both `onFocus` and `onBlur` are wired into the Tiptap `useEditor` config (the existing `onFocus` callback in `ProposalSection` is already present; `onBlur` is added in parallel).

    11.5 — **`pagehide` / `visibilitychange` global cleanup** — registered ONCE per `ProposalEditor` mount (not per section) inside a `useEffect`:
    ```ts
    useEffect(() => {
      const handler = (event: Event) => {
        if (event.type === "pagehide" || (event as Event).type === "visibilitychange" && document.visibilityState === "hidden") {
          // For each section we hold, fire the beacon release.
          const states = useCollaboratorStore.getState().localLockState;
          Object.entries(states).forEach(([key, state]) => {
            if (state === "held") releaseSectionLockBeacon(proposalId, key);
          });
        }
      };
      window.addEventListener("pagehide", handler);
      document.addEventListener("visibilitychange", handler);
      return () => {
        window.removeEventListener("pagehide", handler);
        document.removeEventListener("visibilitychange", handler);
      };
    }, [proposalId]);
    ```
    Lives in `ProposalEditor.tsx` (one effect at editor scope, not per section, to avoid N beacons per event).

    11.6 — **Unmount cleanup** — `ProposalEditor`'s existing unmount cleanup (the `useEffect` returning `useProposalEditorStore.setState({...})`) is augmented to also call `useCollaboratorStore.getState().clearLocalLockStates()`. This prevents lock state from one proposal leaking into the next on client-side navigation between proposal pages.

12. [ ] **AC12 — Toolbar integration (`ProposalWorkspacePage`).**

    Add a new toolbar button between "History" and the right-edge spacer:
    ```tsx
    <Button
      data-testid="toolbar-btn-collaborators"
      variant="outline"
      size="sm"
      onClick={() => setCollaboratorPanelOpen(true)}
      aria-label={t("collaborators.toolbarBtnAria")}
    >
      <Users className="h-4 w-4" />
      {data?.items?.length ? <span className="ml-1 text-xs">{data.items.length}</span> : null}
    </Button>
    ```
    `data` here is the result of `useCollaborators(proposal.id)`. The collaborator count badge is non-clickable, just informational.

    Mount the panel at the bottom of `ProposalWorkspacePage` (siblings to `VersionHistorySidebar` and `ExportDialog`):
    ```tsx
    <CollaboratorPanel
      proposalId={proposal.id}
      companyId={proposal.company_id}
      isOpen={collaboratorPanelOpen}
      onClose={() => setCollaboratorPanelOpen(false)}
    />
    ```

    `collaboratorPanelOpen` lives in `useState(false)` at `ProposalWorkspacePage` scope (matches `historyOpen` / `exportOpen` precedent).

    The toolbar's lock-summary badge (AC9 last paragraph) is rendered separately, between the version indicator and the save-status indicator. Clicking it opens the same panel.

13. [ ] **AC13 — i18n keys (`apps/client/messages/{en,bg}.json`).**

    Two new namespaces under the existing `proposals` root:

    `proposals.collaborators.*` (en):
    - `panelTitle` — "Collaborators"
    - `closeBtn` — "Close panel"
    - `errorTitle` — "Could not load collaborators"
    - `errorDescription` — "Please try again."
    - `emptyTitle` — "No collaborators yet"
    - `emptyDescription` — "Add team members to collaborate on this proposal."
    - `addSectionTitle` — "Add a collaborator"
    - `readOnlyNote` — "Only proposal bid managers can add or remove collaborators."
    - `userSearchPlaceholder` — "Search team members…"
    - `userSearchNoResults` — "No matching team members."
    - `addBtn` — "Add"
    - `addingBtn` — "Adding…"
    - `addSuccessTitle` — "Collaborator added"
    - `addSuccessDescription` — "{name} can now collaborate on this proposal."
    - `addErrorTitle` — "Could not add collaborator"
    - `duplicateCollaborator` — "This user is already a collaborator on this proposal."
    - `memberInactive` — "This user is not an active member of your company."
    - `notAuthorisedToAdd` — "Only proposal bid managers can add collaborators."
    - `notAuthorised` — "You are not allowed to perform this action."
    - `lastBidManagerError` — "Cannot remove or demote the only bid manager. Promote another collaborator first."
    - `collaboratorGone` — "This collaborator no longer exists. The list has been refreshed."
    - `removeBtnAria` — "Remove {name}"
    - `removeDialogTitle` — "Remove collaborator?"
    - `removeDialogDescription` — "{name} will lose access to this proposal."
    - `removeConfirmBtn` — "Remove"
    - `removeCancelBtn` — "Cancel"
    - `removeSuccessTitle` — "Collaborator removed"
    - `removeErrorTitle` — "Could not remove collaborator"
    - `roles.bid_manager` — "Bid manager"
    - `roles.technical_writer` — "Technical writer"
    - `roles.financial_analyst` — "Financial analyst"
    - `roles.legal_reviewer` — "Legal reviewer"
    - `roles.read_only` — "Read-only"
    - `grantedBySystem` — "Added by system"
    - `grantedByLabel` — "Added by {name} on {date}"
    - `toolbarBtnAria` — "Open collaborators panel"

    `proposals.locks.*` (en):
    - `editingPill` — "Editing"
    - `editingPillAria` — "You are editing this section. {remaining} remaining."
    - `lockedByPill` — "Locked by {name}"
    - `foreignPillAria` — "Locked by {name}. {remaining} remaining."
    - `foreignTooltip` — "{name} is editing — locked until {expiresAt}"
    - `unknownUser` — "another user"
    - `expiredLabel` — "expired"
    - `cannotEditTitle` — "Section is locked"
    - `cannotEditDescription` — "{name} is currently editing the {sectionTitle} section."
    - `conflictTitle` — "Could not acquire lock"
    - `conflictDescription` — "{name} is already editing this section."
    - `acquireFailedTitle` — "Could not acquire lock"
    - `cannotSaveTitle` — "Cannot save"
    - `cannotSaveDescription` — "You do not hold the lock for {sectionTitle}."
    - `toolbarSummary` — "{count} {count, plural, one {section locked} other {sections locked}}"

    Bulgarian translations for ALL of the above MUST land in `bg.json` in the same key positions. The existing `apps/client/scripts/check-i18n.ts` (or equivalent — see the script run by `pnpm check:i18n`) MUST exit `0`. Add a parity test: `apps/client/__tests__/i18n-setup.test.ts` already enumerates key counts — extend it (or rely on the parity check) so any drift fails CI.

    Suggested Bulgarian translations (the developer / translator may refine):
    - `panelTitle` — "Сътрудници"
    - `addBtn` — "Добави"
    - `editingPill` — "Редактирате"
    - `lockedByPill` — "Заключено от {name}"
    - `cannotEditTitle` — "Секцията е заключена"
    - (and so on — full list to be added; CI parity check is the gate.)

14. [ ] **AC14 — ATDD test file `apps/client/__tests__/collaborator-lock-s10-12.test.ts`.**

    Static / structural Vitest test mirroring the Story 11.13 / 7.12 pattern. Asserts file existence + key exports + i18n key presence + structural invariants. RED-phase before implementation.

    **File-system asserts (AC1):** `existsSync` for each of the six new component files + three new API/query/store files. Fail with descriptive messages.

    **API client exports (AC2, AC3):** parse the source files for the named export signatures listed above; assert each function name + interface name appears.

    **Hook exports (AC4, AC5):** assert each hook name appears as a top-level `export function`.

    **Store contract (AC10):** assert `useCollaboratorStore` is exported; assert state shape via a smoke render that calls `getState()` and reads `localLockState`, `foreignLockConflicts`, `pendingReleaseTimers`, and the three setters.

    **i18n parity (AC13):** load `en.json` and `bg.json`; assert every `proposals.collaborators.*` key in en is present in bg AND vice versa; assert the five `proposals.collaborators.roles.*` entries cover the five `ProposalCollaboratorRole` values exactly.

    **Component data-testid presence (AC6, AC7, AC8, AC9, AC12):** read each component file as text; assert the prescribed `data-testid` strings appear (e.g. `collaborator-panel`, `collaborator-row-`, `add-collaborator-form`, `add-collaborator-submit-btn`, `section-lock-self-`, `section-lock-foreign-`, `toolbar-btn-collaborators`, `toolbar-lock-summary-badge`).

    **Behavioural smoke (rendering)** — at least four React Testing Library scenarios:
    - `test_collaborator_panel_renders_skeleton_when_loading`.
    - `test_collaborator_panel_renders_empty_state_when_no_collaborators`.
    - `test_collaborator_row_hides_remove_btn_when_canManage_false`.
    - `test_section_lock_indicator_renders_self_pill_when_lock_is_current_user` (mocks `useAuthStore` + props).

    **Mutation-surface asserts:**
    - `test_add_collaborator_form_disabled_when_user_id_empty`.
    - `test_add_collaborator_form_excludes_existing_users_from_search`.
    - `test_remove_collaborator_409_keeps_dialog_open_with_inline_error`.

    **Lock-state asserts:**
    - `test_acquire_section_lock_optimistic_state_transitions_to_held` (mocks the API to `200`).
    - `test_acquire_section_lock_423_writes_foreign_conflict_and_blurs_editor` (mocks API to return `{ conflict: ... }`).
    - `test_blur_release_is_cancelled_if_focus_returns_within_800ms` (uses `vi.useFakeTimers`).
    - `test_pagehide_handler_fires_beacon_for_held_locks` (asserts `navigator.sendBeacon` called with the right path).
    - `test_section_editor_setEditable_false_when_foreign_lock_active`.
    - `test_auto_save_suppressed_when_lockState_not_held` (mocks `apiClient.patch` and asserts NOT called).

    **Polling asserts:**
    - `test_use_section_locks_uses_30s_refetch_interval` (inspect the QueryClient config or hook return).
    - `test_use_section_locks_refetches_on_window_focus`.

    Use `@pytest.mark.skip`-equivalent — Vitest's `it.skip` — on the behavioural / mutation tests upfront so the suite compiles green at story-creation time; markers come off per the TDD activation sequence (matches the Story 10-11 "RED PHASE" precedent for backend tests).

    Total expected scenarios: **≥ 24**. Coverage areas mirror the 14 ACs.

## Dev Notes

- **Test Expectations (from Epic-level test design):** `eusolicit-docs/test-artifacts/test-design-epic-10.md` is **not** available — Epic 10 was never epic-test-designed (the `test-artifacts/` directory for Epic 10 contains only per-story ATDD checklists 10-1 through 10-11 plus an `e12-p0` summary; no epic-level design doc exists; same gap noted in Stories 10.9 / 10.10 / 10.11). Coverage strategy therefore follows the established frontend story patterns: Vitest with `environment: "jsdom"` for component tests and `environment: "node"` for the structural file-system asserts (mix in the same file using `vitest-environment` directives where needed; Story 7.12 + 11.13 set the precedent). Use React Testing Library + `userEvent` for interactive scenarios, `vi.useFakeTimers()` for the blur-debounce + countdown tests, `vi.spyOn(navigator, "sendBeacon")` for the pagehide test, and `MockedProvider` / `QueryClientProvider` wrapping for any TanStack Query hook test (the project already ships a test helper for this — see `apps/client/lib/queries/__tests__/` in any earlier frontend story for the pattern). Mock `useAuthStore` via `vi.mock("@eusolicit/ui", async () => ({ ...await vi.importActual("@eusolicit/ui"), useAuthStore: vi.fn() }))` since the store is exported from the shared UI package. Apply `it.skip` markers upfront to the behavioural tests (RED phase) so the suite compiles green at story-creation time.

- **Backend contract verification — read-before-write.** Before authoring the API client functions, double-check the live route prefixes in `services/client-api/src/client_api/main.py`: collaborators are at `/api/v1/proposals/{proposal_id}/collaborators` (the `proposal_collaborators_v1.router` already carries the full prefix in its `APIRouter(prefix=...)`); section locks are at `/api/v1/proposals/{proposal_id}/sections/{section_key}/lock` (the section-locks router has prefix `/{proposal_id}/sections` and is included with `prefix="/proposals"` in `main.py`); members are at `/api/v1/companies/{company_id}/members` (the members router has prefix `/{company_id}/members` and is included with `prefix="/companies"`). Capture this in the API client function bodies as the literal URL strings — do NOT abstract behind a path-builder for this story; the four / three / one endpoints are well-bounded.

- **Auth store reuse — no new code.** Import `useAuthStore` from `@eusolicit/ui` (already shipped via `frontend/packages/ui/src/lib/stores/auth-store.ts`). The `User.id`, `User.companyId`, and `User.role` fields are already populated on login (see `apps/client/lib/api/auth.ts::loginUser` which calls `/auth/me`). The `useMyProposalRole` hook reads `useAuthStore.user` to (a) check `role === "admin"` for the admin bypass and (b) match `user.id` against the collaborators list. The `useAuthStore` is `persist`-ed under `eusolicit-auth-store` so it's available immediately on workspace mount.

- **Slide-over primitive — verify existence.** Check whether `@eusolicit/ui` already exports a `Sheet` / `Drawer` / `SidePanel` primitive (likely yes, given Story 7.16's `VersionHistorySidebar`). If yes, reuse it for visual + accessibility parity (focus-trap, Escape-to-close, overlay click-to-close all come for free). If no, fall back to a fixed-positioned `<div>` with manual focus-trap + Escape handler + the existing `<dialog>` polyfill. The acceptance test for this is structural only (`existsSync` + `data-testid="collaborator-panel"`) — the visual polish is at the developer's discretion.

- **TanStack Query polling semantics.** `refetchInterval: 30_000` polls every 30s when the tab is foregrounded; `refetchIntervalInBackground: false` (the default — set explicitly for clarity) stops polling when the tab is backgrounded; `refetchOnWindowFocus: true` re-fires the query when the user returns to the tab — combined this gives "poll while you're looking, freeze when you leave, refresh when you come back" semantics. No external `setInterval` or polling store is needed; React Query owns it. On unmount the query is automatically garbage-collected per the standard TanStack Query lifecycle.

- **Beacon vs. fetch with `keepalive` — the modern fallback chain.** `navigator.sendBeacon(url, data)` returns a `boolean` indicating whether the browser has queued the request; it cannot send custom headers. For a `DELETE` we need `Authorization: Bearer ...` plus the JWT, which `sendBeacon` does NOT support. So the implementation should be: `if (navigator.sendBeacon && supportsBeaconAuth) { sendBeacon(...) } else { fetch(url, { method: "DELETE", credentials: "include", keepalive: true, headers: { Authorization: \`Bearer ${token}\` } }) }`. Modern browsers (Chrome 89+, Firefox 105+, Safari 15.4+) support `fetch` with `keepalive: true` for unload-safe requests up to 64 KB per origin — the DELETE body is empty, so the limit is irrelevant. The auth header gets pulled from `useAuthStore.getState().token` at call time. Wrap in a try/catch and ignore failures — the 15-minute server TTL is the durable safety net.

- **Lock-acquire optimism vs. pessimism — the chosen middle ground.** Tiptap's `setEditable(false)` blocks input but does NOT cancel an in-flight focus animation; if we waited for the acquire response before allowing typing the user would see a noticeable lag (typical RTT 80–200 ms locally, more on poor networks). The chosen optimistic path lets the user type immediately and only blocks save (auto-save + full-save gated by `localLockState === "held"`). On the rare 423 (foreign lock raced us), the user sees a toast + the editor switches to non-editable, and any text they typed during the round-trip is discarded by the next polling refresh + re-render (the ProposalSection's `useEditor` content prop is the source of truth for the section body). A future refinement could capture local edits and offer "save these changes anyway" once the lock frees, but that's out of scope here.

- **Why not a `Sheet` from a third-party drawer library?** The project already standardises on shadcn primitives via `@eusolicit/ui` (Story 3.2). Pulling in a new drawer library (vaul, react-aria-overlays) would add bundle weight and design-system drift. Reuse the existing `Sheet` / `VersionHistorySidebar` pattern — it solves the same problem.

- **`useCollaborators` cache key shape.** Use `["collaborators", proposalId]` (NOT `["collaborator", proposalId]` — match the plural form for consistency with the API and the `["section-locks", proposalId]` plural form used by `useSectionLocks`). Mutations invalidate by exact key. Cross-page navigation (e.g. user opens proposal A, then proposal B) doesn't need explicit cleanup — the per-proposalId key naturally segregates caches. The `staleTime: 30_000` on `useCollaborators` keeps the panel snappy on repeat opens within 30 s.

- **Avatars / initials — local computation, no upload.** The Story 2.8 / 2.9 user model does not include `avatarUrl` for proposal collaborators; we compute initials client-side from `user_full_name ?? user_email`. The `<Avatar>` shadcn primitive accepts a `<AvatarFallback>` for this purpose. If `useAuthStore.user.avatarUrl` is later populated from an opportunity-detail join, the row can be upgraded — out of scope here.

- **Role badge colour palette.** Match the project's existing badge palette (Story 3.2 design tokens): `default` for the highest-trust role (bid_manager — primary brand colour), `secondary` for write roles (technical_writer / financial_analyst / legal_reviewer — neutral surface), and outline for `read_only` (no fill, neutral border). This mirrors the convention already used by `proposal-status-badge` (status="active" is `default`, "draft" is `secondary`, "archived" is the outlined variant).

- **The `/auth/me` endpoint is NOT re-fetched here.** The `useAuthStore` is already populated at login + persisted; this story does not need to call `/auth/me` separately. If a stale role concern arises (e.g. company admin demoted mid-session), that's a session-management concern handled by Story 2.4 / 2.5 token refresh — out of scope.

- **Beacon release — security & privacy note.** The DELETE call carries the user's JWT; on `pagehide` this is sent as a final request before the page unloads. Browsers permit this under `keepalive: true`; the request is identical to a normal in-page DELETE. No secrets are leaked beyond what was already in the page's request headers.

- **i18n key drift detection.** The existing `pnpm check:i18n` (or equivalent script in `apps/client/scripts/`) runs in CI and asserts EN/BG parity. If the script doesn't exist, add a one-liner test: `expect(Object.keys(en.proposals.collaborators).sort()).toEqual(Object.keys(bg.proposals.collaborators).sort())`. Same for `proposals.locks`. The Story 11.13 review history (Round 2 F9 / F10 findings) shows that hardcoded English strings are the most common bug class — the structural i18n test is the cheapest defence.

- **`data-testid` discipline.** Every interactive element MUST carry a stable `data-testid`. Per-row testids include the user_id (e.g. `collaborator-row-{user_id}`); per-section testids include the section key (`section-lock-self-{section_key}`). This matches the convention in Story 7.12 (`section-header-{key}`, `section-editor-{key}`) and Story 11.13 (`espd-profile-row-{id}`).

- **Out of scope:**
  - Real-time push of lock changes (WebSockets / SSE). Polling at 30 s is the spec; a future story can upgrade to SSE if 30 s feels stale during high contention.
  - Lock force-release UI for bid_managers (the API supports it; the UI in this story does not surface a "Steal lock" button — opening that affordance is a Story 10.13+ concern once the comments + presence layer ships).
  - Mobile-specific UX for the slide-over (the panel is `w-full` on mobile per AC6 — the affordances render but the focus / blur lock-acquire flow is desktop-first; mobile users can still browse but typing-on-touch flows are TBD).
  - Avatar uploads / Gravatar integration — initials only.
  - Per-section presence ("3 viewers reading this section") — out of scope; locks-only.
  - Email invitations to non-members (the user must already be a company member via Story 2.9 invite flow before they can be added as a proposal collaborator; this matches the backend's 422 `member-inactive` guard).
  - The `LockedSectionToastBody` component is intentionally minimal — a richer "request lock takeover" UX is a follow-up.
  - Bulgarian translations are scaffolded in AC13 with sample strings; the final translation pass is owned by a translator (or a follow-up i18n story); for MVP the developer commits placeholder Bulgarian text and CI parity passes.

- **Future-compatibility notes:**
  - Story 10.13 (Comments Sidebar UI) will reuse `useMyProposalRole` to gate the "create comment" affordance for non-`read_only` roles. The hook is designed to be the single source of truth for "what can I do on this proposal?" across all Epic 10 frontend surfaces.
  - Story 10.14–10.16 (kanban board, template manager, approval pipeline) operate at the company / opportunity scope, not the proposal scope — they don't need `useMyProposalRole` directly, but they share the same role-badge palette + i18n role labels (`collaborators.roles.*`) so colours and copy stay consistent.
  - The 30-s polling cadence is a server-friendly default; a future Story (e.g. 10.13's comments stream) can upgrade to SSE or WebSocket presence and `useSectionLocks` can be refactored to consume the same stream — the hook's external surface (returning a `SectionsListResponse` shape) wouldn't change.
  - The `CollaboratorPanel`'s slide-over primitive can be promoted to a shared Epic-10-wide "right rail" (e.g. shared with Comments and Tasks) by extracting it into `@eusolicit/ui` as a `RightRail` primitive — out of scope here, but the file layout admits that refactor.

## Change Log

- 2026-04-21: Initial story context created. (bmad-create-story)
- 2026-04-21: Code-review findings resolved. (bmad-dev-story)
  - F1 (blocking): Fixed `useUIStore` import in `use-collaborators.ts`, `use-section-locks.ts`, `ProposalSection.tsx` — sourced from `@/lib/stores/ui-store` instead of `@eusolicit/ui`.
  - F2 (blocking): Fixed `ProposalSection` editable gate to follow AC11.2 (`lock === null || locked_by === currentUserId`) — removed the `localLockState === "held"` clause that blocked optimistic acquire.
  - F3 (blocking): Coerced `isLockedByMe`/`isLockedByOthers` to `boolean` via `!!` to fix TypeScript `boolean | null` type mismatch on `SectionLockIndicator.isCurrentUser` prop.
  - F4: Changed `CollaboratorRow.onRemove` to accept `onSuccess`/`on409Error` callbacks; dialog stays open with inline `lastBidManagerError` on 409.
  - F5: Fixed role-select label in `AddCollaboratorForm` to use `t("roleLabel")`; added `roleLabel` key to `en.json` + `bg.json` (parity maintained).
  - F6: Removed dead `navigator.sendBeacon` commented-out branch from `releaseSectionLockBeacon`; implementation uses `fetch` with `keepalive: true`; updated ATDD assertion to match.
  - F7: `doAutoSave` in `ProposalEditor` now looks up section title from `content.sections` for the cannot-save toast instead of using the raw key.
  - F8: Added `editor.isFocused` check in `acquireLock.onSuccess` to release immediately if editor blurred during round-trip; blur handler reads fresh state via `useCollaboratorStore.getState()`.
  - F9: Added `useEffect` in `ProposalSection` to clear stale `foreignLockConflicts` when polling shows foreign lock released.
  - F10: `useAcquireSectionLock.onError` now surfaces an error toast (`acquireFailedTitle` + `forms.serverError`).
  - F11: `useAddCollaborator.onError` adds `5xx` branch routing to `tForms("serverError")` + `console.error`.
  - F15: Moved `visibilitychange` listener from `window` to `document` (MDN canonical target per AC11.5).
  - Added `forms.serverError` to `en.json` + `bg.json` (used by F10, F11 server-error toasts).
- 2026-04-21: Round 2 blocking findings resolved. (bmad-dev-story)
  - F16 (blocking): Added `"axios": "^1.7.0"` to `apps/client/package.json` dependencies; ran `pnpm install` to link `axios@1.14.0` from the virtual store into `apps/client/node_modules` — resolves `TS2307: Cannot find module 'axios'`.
  - F17 (blocking): Resolved by F16 fix — `axios.isAxiosError(error)` type guard in `section-locks.ts` now correctly narrows `unknown` catch binding via `AxiosError<unknown,unknown>` once axios types are resolvable; `tsc --noEmit` confirms no TS18046 errors remain in `section-locks.ts`.
  - F18 (blocking): Replaced JSX `<LockedSectionToastBody/>` as `addToast({ description })` value in `ProposalSection.tsx` conflict handler with the spec-compliant translated string `t("locks.conflictDescription", { name: ... })` — resolves `TS2322: Type 'Element' is not assignable to type 'string'`. Also completed the full AC11.3 conflict flow: `setLocalLockState(section.key, "foreign")`, `setForeignLockConflict(section.key, resp.conflict)`, `editor?.setEditable(false)`, `editor?.commands.blur()`, correct `type: "warning"` / `title: t("locks.conflictTitle")`. Removed the redundant per-mutation `onError` callback (hook-level `useAcquireSectionLock.onError` already handles state reset + toast — avoids `TS2345` caused by `t("forms.serverError")` being called on the `proposals` namespace translator).
  - ATDD suite: 160 / 160 passing (confirmed).

## Known Deviations

_(none — all code-review findings F1–F18 resolved; F12/F13/F14 remain open as non-blocking style follow-ups per Round 1 classification)_

## Senior Developer Review — Round 3

**Reviewer:** bmad-code-review
**Date:** 2026-04-21
**Outcome:** 🟢 **REVIEW: Approve**

All Round 1 blocking findings (F1–F3) and all Round 2 blocking findings (F16–F18) are resolved. Verification performed against the on-disk implementation at `eusolicit-app/frontend/apps/client/`:

### Verification performed

- **F16 — `axios` dependency.** `apps/client/package.json:18` now declares `"axios": "^1.7.0"`. `pnpm install` has been run and `tsc --noEmit` no longer emits `TS2307: Cannot find module 'axios'` for `lib/api/section-locks.ts`.
- **F17 — `axios.isAxiosError` narrowing.** With F16 in place, `section-locks.ts:61-62` `catch (error)` binding is correctly narrowed by `axios.isAxiosError(error)`. No `TS18046` errors remain.
- **F18 — Toast description is a string.** `ProposalSection.tsx:85-91` now passes `description: t("locks.conflictDescription", { name: ... })` — matches AC11.3 verbatim. The JSX-as-description regression is gone. Full AC11.3 conflict flow present: `setLocalLockState(section.key, "foreign")`, `setForeignLockConflict(section.key, resp.conflict)`, `editor?.setEditable(false)`, `editor?.commands.blur()`, `type: "warning"`, `title: t("locks.conflictTitle")`.
- **TypeScript.** `./node_modules/.bin/tsc --noEmit` from `apps/client/` emits exactly one error touching files this story owns: `ProposalSection.tsx:7` `@tiptap/extension-table` default-import error — explicitly flagged in Round 2 as a pre-existing issue owned by another story, not this one.
- **ATDD suite.** `vitest run __tests__/collaborator-lock-s10-12.test.ts` → **160 / 160 passing** (rerun confirmed, ~547 ms).
- **Round 1 carryovers re-verified on disk:** F1 imports from `@/lib/stores/ui-store` (three files), F2 editable gate matches AC11.2, F3 `!!` boolean coercion, F4 `onRemove`/`on409Error` callbacks keep dialog open on 409, F5 `collaborators.roleLabel` in both locales, F6 no dead `sendBeacon` branch, F7 title lookup via `content.sections.find(...)?.title`, F8 `editor.isFocused` check, F9 stale-conflict cleanup `useEffect`, F10 acquire-error toast at hook level, F11 5xx → `forms.serverError`, F15 `visibilitychange` on `document`.

### Non-blocking items (unchanged from Round 1 triage)

- **F12** — redundant `TooltipProvider` wrappers. Style follow-up.
- **F13** — `t(...) as any` casts for role keys. Would benefit from a typed-key helper; pattern exists elsewhere in codebase.
- **F14** — `useMyProposalRole` returns `isLoading: false` for admin bypass. Round 1 itself noted "arguably more correct" — leaving as-is is defensible.

None of F12/F13/F14 block shipping. They should become tracked tech-debt tickets rather than blocking this story further.

### What landed correctly (recap)

- 14 ACs' file-system, export, testid, and i18n contracts all satisfied (ATDD 160/160).
- 32 / 32 `collaborators.*` keys + 15 / 15 `locks.*` keys EN/BG parity; `forms.serverError` + `collaborators.roleLabel` also parity-clean.
- Store (`localLockState`, `foreignLockConflicts`, `pendingReleaseTimers` + setters + `clearLocalLockStates`) matches AC10; no `persist` middleware.
- Polling config (`refetchInterval: 30_000`, `refetchIntervalInBackground: false`, `refetchOnWindowFocus: true`, `staleTime: 0`) matches AC5 verbatim.
- `acquireSectionLock` returns `{ conflict }` on 423 instead of throwing (AC3).
- `releaseSectionLockBeacon` uses `fetch` with `keepalive: true` + Authorization header (spec-compliant primary path per dev note).
- Toolbar wires both the "Collaborators" button + count badge + lock-summary badge (clickable), and mounts `<CollaboratorPanel/>` alongside `VersionHistorySidebar` / `ExportDialog` (AC12).
- `LockCountdown` is `React.memo`-wrapped, auto-stops at TTL 0 (AC9).
- `ProposalEditor` beacon-release effect registers `pagehide` on `window` and `visibilitychange` on `document`, and clears local lock states on unmount (AC11.5, 11.6).
- Auto-save lock gate present in `doAutoSave` with human-readable section title (AC8 + F7).

### Recommended follow-ups (not blocking)

1. Resolve the pre-existing `@tiptap/extension-table` default-import `tsc` error in its owning story.
2. Widen `Toast.description` to `ReactNode` at the `ui-store` level in a separate refactor if richer toast bodies become needed; defer `LockedSectionToastBody` wiring until then (currently defined per AC1 but intentionally unused as a toast description).
3. Clean up `as any` role-key casts (F13) with a typed i18n-key helper once the codebase-wide pattern is chosen.
4. Consider promoting the tooltip provider to the app shell to remove per-row `TooltipProvider` instantiation (F12).

### Manual smoke recommended before shipping

Although all automated gates pass, the ATDD suite is structural — it does not exercise live editor focus/blur, lock acquire round-trips, or the countdown tick. A developer smoke covering:

1. Focus a section → self-pill appears with countdown ticking every 1s.
2. Second tab (different user) focuses the same section → first tab shows foreign-lock pill with 5s tick, toast description is a string (not `[object Object]`).
3. Close tab → `fetch({ keepalive: true })` release fires in Network tab.
4. Role-gating — log in as `technical_writer` → confirm Add/Remove controls are hidden and `readOnlyNote` is visible.

— would confirm end-to-end behaviour before release.

---

## Senior Developer Review — Round 2

**Reviewer:** bmad-code-review
**Date:** 2026-04-21
**Outcome:** 🔴 **REVIEW: Changes Requested**

F1–F11 and F15 from Round 1 are resolved (verified by code inspection). F12–F14 remain as minor / non-blocking follow-ups per the original review's own classification. However, landing the Round 1 fixes introduced **three new blocking TypeScript defects** that `./node_modules/.bin/tsc --noEmit` surfaces immediately for the files this story touches. The 160-test ATDD suite still passes — a reminder that the suite is structural (file / export / testid / i18n-parity) and does not exercise type safety or runtime behaviour.

### 🔴 Blocking findings (Round 2)

**F16 — `axios` module not resolvable from `apps/client/lib/api/section-locks.ts`.** `tsc --noEmit` on the client app emits:

```
lib/api/section-locks.ts(2,19): error TS2307: Cannot find module 'axios' or its corresponding type declarations.
```

`apps/client/package.json` has no `axios` entry (`grep -c "axios" apps/client/package.json` → 0). The existing `lib/api/opportunities.ts` has the same latent problem but is not in this story's scope; the new `section-locks.ts` re-introduces it. Two defensible fixes:

- (a) Add `"axios": "^<version>"` to `apps/client/package.json` dependencies (preferred — matches what `opportunities.ts` silently relies on today via the `pnpm` hoist).
- (b) Avoid the direct `import axios from "axios"` — the existing `apiClient` is already an axios instance exported from `@eusolicit/ui`; use `apiClient.isAxiosError` if available, or narrow via `error instanceof Error && "response" in error && …`.

Without one of these, `tsc` blocks CI the moment strict type-checking is enforced on this app.

**F17 — `catch (error)` does not narrow in `section-locks.ts:61-62`.**

```
lib/api/section-locks.ts(61,38): error TS18046: 'error' is of type 'unknown'.
lib/api/section-locks.ts(62,26): error TS18046: 'error' is of type 'unknown'.
```

Under `strict: true` (TS 4.4+) the bare `catch (error)` binding is `unknown`, and `axios.isAxiosError` is declared as a type guard that narrows to `AxiosError<any, any>` — so once F16 is resolved the narrowing should fire. If after F16 these errors persist, add an explicit annotation / cast (`if (axios.isAxiosError<SectionLockConflictDetail>(error))`). Verify with `tsc --noEmit` after the fix.

**F18 — `addToast({ description })` receives a `ReactElement`, but `Toast.description` is typed `string | undefined`.** `ProposalSection.tsx:86-90`:

```tsx
addToast({
  type: "error",
  title: t("locks.cannotEditTitle"),
  description: (
    <LockedSectionToastBody
      name={resp.conflict.locked_by_full_name ?? t("locks.unknownUser")}
      expiresAt={resp.conflict.expires_at}
    />
  ),
});
```

`lib/stores/ui-store.ts:9-15` declares:

```ts
export interface Toast {
  id: string;
  type: "success" | "error" | "warning" | "info";
  title: string;
  description?: string;   // ← string only, not ReactNode
  duration?: number;
}
```

`tsc` reports: `ProposalSection.tsx(86,17): error TS2322: Type 'Element' is not assignable to type 'string'.` At runtime the toast component will receive a JSX element where it expects a string; depending on the renderer this either throws or renders as `[object Object]`. This regression was introduced while resolving Round 1 findings (F8 / spec) — the spec itself (AC11.3) specifies a **string** description: `t("locks.conflictDescription", { name: resp.conflict.locked_by_full_name ?? t("locks.unknownUser") })`. Revert the JSX body and use the translated string per AC11.3. The `LockedSectionToastBody` component remains defined per AC1 (the ATDD test asserts its existence) but is not needed as a toast `description`; if a richer body is required, widen the `Toast.description` type to `ReactNode` in a separate follow-up that updates every existing caller.

### 🟠 Still open from Round 1 (non-blocking)

- **F12** (redundant `TooltipProvider` wrappers in `CollaboratorRow` + `SectionLockIndicator`) — not addressed; style follow-up.
- **F13** (`t(\`roles.${role}\` as any)` casts in `CollaboratorRow` + `AddCollaboratorForm`) — not addressed; typed-key helper would resolve.
- **F14** (`useMyProposalRole` returns `isLoading: false` on admin bypass instead of `<collaborators-loading>`) — not addressed; Round 1 note said "arguably more correct" so leaving as-is is defensible.

### ✅ What landed correctly in Round 2

- F1 — `useUIStore` now imported from `@/lib/stores/ui-store` in all three files (verified).
- F2 — `ProposalSection.tsx:66, 155` editable gate matches AC11.2: `!isGenerating && (sectionLock === null || sectionLock?.locked_by === currentUserId)`.
- F3 — `isLockedByMe` / `isLockedByOthers` coerced via `!!` (lines 53-54).
- F4 — `CollaboratorRow.onRemove` now accepts `onSuccess`/`on409Error` callbacks; dialog stays open with `removeError` inline on 409 (`CollaboratorPanel.tsx:114-122`, `CollaboratorRow.tsx:163-198`).
- F5 — `collaborators.roleLabel` key added to `en.json:1092` + `bg.json:1092`; `AddCollaboratorForm.tsx:115` references it.
- F6 — `releaseSectionLockBeacon` uses plain `fetch({ keepalive: true })`; no dead `sendBeacon` branch.
- F7 — `doAutoSave` looks up title via `content.sections.find((s) => s.key === key)?.title ?? key` (`ProposalEditor.tsx:164`).
- F8 — `acquireLock.onSuccess` checks `editor.isFocused`; if false, immediately releases (`ProposalSection.tsx:94-104`).
- F9 — `ProposalSection.tsx:143-150` `useEffect` clears stale `foreignLockConflicts` when polling returns `lock === null` or own lock.
- F10 — `useAcquireSectionLock.onError` surfaces `acquireFailedTitle` + `forms.serverError` toast (`use-section-locks.ts:43-50`).
- F11 — `useAddCollaborator.onError` now routes `5xx` to `tForms("serverError")` + `console.error` (`use-collaborators.ts:56-59`).
- F15 — `visibilitychange` listener registered on `document`, canonical target (`ProposalEditor.tsx:97`).
- ATDD suite: 160 / 160 passing (rerun confirmed).
- i18n parity: `forms.serverError` + `collaborators.roleLabel` present in both locales.

### Suggested fix sequence

1. **F18** — swap the JSX `description` back to `t("locks.conflictDescription", {...})` per AC11.3. Single-line change in `ProposalSection.tsx:86`. Unblocks `tsc` and restores spec-compliant UX.
2. **F16** — add `axios` to `apps/client/package.json` dependencies (or refactor to use `apiClient` / type-only axios import). Run `pnpm install` to update the lockfile.
3. **F17** — re-run `tsc` after F16; if the narrowing errors persist, add explicit generic on `axios.isAxiosError<SectionLockConflictDetail>(error)` or `catch (error: unknown)` + manual narrowing.
4. Re-run `./node_modules/.bin/tsc --noEmit` from `apps/client/` — the three new errors for this story's files should drop to zero. Leave the pre-existing errors in unrelated files (`ProposalSection.tsx:7` extension-table default import, `auth.ts` AxiosResponse fields, etc.) for their owning stories to clean up.
5. Re-run `./node_modules/.bin/vitest run __tests__/collaborator-lock-s10-12.test.ts` — expect 160 / 160 unchanged.

After F16 / F17 / F18 land, a manual smoke in the workspace remains recommended: focus a section → confirm the acquire fires → confirm the self-pill appears → open a second browser tab with a different user → verify the foreign lock toast renders a **string** description (not `[object Object]`).

---

## Senior Developer Review — Round 1 (historical)

**Reviewer:** bmad-code-review
**Date:** 2026-04-21
**Outcome:** 🔴 **REVIEW: Changes Requested**

The ATDD structural suite passes (160/160) and the file-system / i18n / testid scaffolding is all in place. However, the implementation has three **blocking defects** that would prevent the code from compiling under `tsc --noEmit` and would make the lock-acquire flow non-functional at runtime. The structural-only nature of the ATDD suite concealed them.

### 🔴 Blocking findings

**F1 — `useUIStore` imported from the wrong package (three files).** The symbol is defined at `apps/client/lib/stores/ui-store.ts`, **not** in `@eusolicit/ui`. Current imports:

- `apps/client/lib/queries/use-collaborators.ts:2` — `import { useAuthStore, useUIStore } from "@eusolicit/ui";`
- `apps/client/lib/queries/use-section-locks.ts:2` — `import { useUIStore } from "@eusolicit/ui";`
- `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalSection.tsx:12` — `import { useAuthStore, useUIStore } from "@eusolicit/ui";`

`tsc --noEmit` emits: `'"@eusolicit/ui"' has no exported member named 'useUIStore'`. At runtime the import resolves to `undefined`, so `useUIStore((s) => s.addToast)` throws `TypeError: useUIStore is not a function` the moment any collaborator mutation or section lock interaction happens.

**Fix:** change the import to `import { useUIStore } from "@/lib/stores/ui-store"` (the pattern `ProposalEditor.tsx` already uses) in all three files. Keep `useAuthStore` sourced from `@eusolicit/ui` — only `useUIStore` needs to move.

**F2 — `ProposalSection` editable gate contradicts AC11.2 and breaks optimistic acquire.** Current `ProposalSection.tsx:65` and `:128`:

```ts
editable: !isGenerating && !isLockedByOthers && localLockState === "held"
```

AC11.2 specifies: *"the section is editable iff: `(!isGenerating) && (lockByKey(section.key) === null || lockByKey(section.key)?.locked_by === currentUserId)`"*. The implementation adds an extra `localLockState === "held"` requirement. `localLockState` defaults to `"idle"` (the store returns `"idle"` when the key is missing), which means:

1. On mount with no active lock, `editable === false`.
2. Tiptap does not fire `onFocus` on a non-editable editor when the user clicks into it.
3. The acquire mutation is therefore never dispatched.
4. `localLockState` stays `"idle"` forever — the editor never becomes editable.

This is a dead-end state. The whole optimistic-acquire story (Design Decision c, AC11.3) is defeated. Remove the `localLockState === "held"` clause from both `editable:` on line 65 and the sync `useEffect` on line 128 so the flag follows AC11.2 verbatim. Keep the `localLockState === "held"` gate **only** in `doAutoSave` / `doFullSave` (AC8) — that's the correct single place to hard-gate writes.

**F3 — `isCurrentUser` prop type mismatch.** `ProposalSection.tsx:143` passes `isCurrentUser={isLockedByMe}`, where `isLockedByMe = sectionLock && sectionLock.locked_by === currentUserId` has type `boolean | null`. `SectionLockIndicator` declares `isCurrentUser: boolean`. `tsc` fails:

```
ProposalSection.tsx(143,13): error TS2322: Type 'boolean | null' is not assignable to type 'boolean'.
```

**Fix:** `const isLockedByMe = !!sectionLock && sectionLock.locked_by === currentUserId;` (or `Boolean(...)`). Same coercion for `isLockedByOthers`.

### 🟠 Non-blocking bugs

**F4 — `CollaboratorRow` 409 path never surfaces the inline error (AC7 violation).** `CollaboratorRow.tsx:175-178` closes the remove dialog synchronously on confirm, regardless of mutation outcome:

```tsx
onClick={() => {
  onRemove(collaborator.user_id);
  setIsDeleteDialogOpen(false);
}}
```

AC7 requires the dialog to stay open on `409` with `t("collaborators.lastBidManagerError")` rendered inline. The row has no awareness of the mutation state since the parent (`CollaboratorPanel`) owns the mutation. Either (a) lift mutation state so the row reads `mutation.error?.response?.status === 409` and suppresses the close + renders the inline error, or (b) move the dialog into `CollaboratorPanel` where the mutation lives.

**F5 — `AddCollaboratorForm` role-select label misuses a role translation as a field label.** Line 115:

```tsx
<label className="text-sm font-medium">{t("roles.technical_writer")}</label>
```

This renders "Technical writer" as the label for the **role select field**, regardless of the currently selected role. AC8 implies a field-level label (AC13 has no `collaborators.roleLabel` key but needs one). Add `collaborators.roleLabel` ("Role" / "Роля") to `en.json` and `bg.json` and reference it here.

**F6 — `releaseSectionLockBeacon` never actually calls `navigator.sendBeacon`.** `section-locks.ts:89-94` has the `sendBeacon` call commented out; only `fetch({ method: "DELETE", keepalive: true, headers: { Authorization })` fires. The ATDD grep passes (the literal string `navigator.sendBeacon` appears in a comment), but behaviourally the beacon branch is dead code. The AC3 spec path ("uses `navigator.sendBeacon(url, "")` if available, else falls back to `fetch`") is not implemented. Because `sendBeacon` can't carry auth headers, keeping only the fetch+keepalive branch is defensible, but please (a) delete the misleading comment + dead `if (navigator.sendBeacon)` branch, and (b) revise the ATDD assertion to test for the actual observable behaviour (presence of `keepalive: true` alone). Also `tsc` warns on line 90: `This condition will always return true since this function is always defined` — the `if (navigator.sendBeacon)` check evaluates a function reference as truthy.

**F7 — `doAutoSave` warning toast exposes the raw section key instead of the title.** `ProposalEditor.tsx:167`:

```ts
description: t("locks.cannotSaveDescription", { sectionTitle: key })
```

End-users see something like `"You do not hold the lock for executive_summary"` — the dev left a `// Better would be actual title but we only have key here` comment. The title IS available via `content.sections.find(s => s.key === key)?.title`. Pass that into `doAutoSave` (or look it up inline) so the toast is human-readable.

**F8 — Blur during `"acquiring"` leaves an orphan lock until TTL.** `ProposalSection.tsx:96-108`: the blur handler short-circuits unless `localLockState === "held"`. If the user focuses a section (state goes `idle` → `acquiring`), blurs within the ~80–200 ms RTT before the 200 arrives, and never refocuses, the acquire resolves to `"held"` but no release is ever scheduled. Server TTL (15 min) is the only cleanup. Either (a) also handle blur during `"acquiring"` by scheduling a release for when the acquire resolves, or (b) chain off the acquire's `onSuccess` to check if the editor is still focused and release immediately otherwise.

**F9 — `foreignLockConflicts` entries leak across polling refreshes.** `use-section-locks.ts` `useAcquireSectionLock.onSuccess` writes `setForeignLockConflict(sectionKey, resp.conflict)` but nothing clears the entry when the next 30-s poll shows the foreign lock released (either by the holder or by TTL). The conflict sits in the store until the next successful acquire. Trivial fix: in the `useSectionLocks` query's `onSuccess` (or a `useEffect` over `lockData`), iterate entries in `foreignLockConflicts` and clear any whose `sectionKey` now has `lock === null` or `lock.locked_by === currentUserId`.

**F10 — `useAcquireSectionLock` `onError` does not surface a toast.** `use-section-locks.ts:40-42` only sets `localLockState` back to `"idle"` on error. The story (AC11.3) shows the focus handler adding a toast on error; the mutation itself is silent. Given the focus handler ALSO uses `acquireMutation.mutate(section.key, {...})` with its own `onSuccess` but has no `onError`, network failures vanish silently. Add an error toast in one place (ideally the hook).

**F11 — `useAddCollaborator` does not route 5xx to `t("forms.serverError")`.** AC4 specifies `5xx surfaces "serverError"`. Current implementation (`use-collaborators.ts:49-61`) only branches on 409/422/403 and falls through to `description = t("addErrorTitle")` — which happens to be the error title, used as the description. Both a correctness bug and a UX bug. Add an explicit `else if (status && status >= 500)` branch that uses `t("forms.serverError")`, with `console.error` per AC4.

### 🟡 Minor / style

**F12 — Redundant `TooltipProvider` wrappers.** `CollaboratorRow` and `SectionLockIndicator` each wrap their own `TooltipProvider`. The project should have a top-level provider at the app shell level to avoid N providers mounting per render. Non-blocking; follow the existing convention elsewhere in the codebase if there's a single root provider.

**F13 — `CollaboratorRow.tsx:122, 132` cast `t()` keys with `as any`.** Use the proper typed key narrowing or a translation-key helper. The existing codebase may tolerate this; worth a follow-up.

**F14 — `useMyProposalRole` returns `isLoading: false` on admin bypass even while the collaborator query is in flight.** The story spec says `isLoading: <collaborators-loading>` for the admin case. Minor — the admin path doesn't need the collaborator list to function, so `false` is arguably more correct.

**F15 — `ProposalEditor.tsx:80-105` registers both `pagehide` (on `window`) and `visibilitychange` (on `window`, not `document`).** AC11.5 specifies `document.addEventListener("visibilitychange", ...)`. `window` also works in practice due to event bubbling, but `document` is the canonical target. Aligning with the spec makes the effect easier to verify and matches MDN's guidance.

### ✅ What landed correctly

- All 14 ACs' file-system and testid contracts are satisfied; ATDD suite (160 tests) is green.
- i18n parity passes: 32 / 32 collaborators keys and 15 / 15 locks keys across EN and BG.
- Store shape (`localLockState`, `foreignLockConflicts`, `pendingReleaseTimers` + setters + `clearLocalLockStates`) matches AC10 exactly and correctly avoids `persist`.
- `useSectionLocks` polling config (`refetchInterval: 30_000`, `refetchIntervalInBackground: false`, `refetchOnWindowFocus: true`, `staleTime: 0`) matches AC5 verbatim.
- `acquireSectionLock` returns the `{ conflict }` shape on 423 instead of throwing, as AC3 requires.
- `ProposalWorkspacePage` wires the toolbar button, count badge, lock-summary badge (clickable), and panel mount per AC12.
- `LockCountdown` is `React.memo`-wrapped and stops its interval when TTL hits zero (AC9).
- `ProposalEditor` correctly adds `pagehide` + `visibilitychange` beacon release and `clearLocalLockStates` on unmount (AC11.5, 11.6).
- Auto-save lock gate is present in `doAutoSave` (AC8), though F7 above notes the title-display bug.

### Suggested fix sequence

1. F1 (one-line import fix in three files) — unblocks compile + runtime.
2. F2 (remove `localLockState === "held"` from `editable:`) — unblocks actual lock acquisition.
3. F3 (coerce `isLockedByMe` to boolean) — unblocks `tsc`.
4. F4 (dialog 409 inline error) — restores AC7 behaviour.
5. F7, F10, F11, F5 — user-facing correctness.
6. F6, F8, F9, F15 — behavioural robustness + spec alignment.
7. F12, F13, F14 — style follow-ups.

After F1–F3 land, a manual smoke in the workspace (click a section → confirm the editor becomes editable → confirm the acquire fires → confirm the self-pill appears) is recommended before re-requesting review.
