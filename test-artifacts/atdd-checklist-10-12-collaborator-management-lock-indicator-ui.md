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
storyKey: '10-12-collaborator-management-lock-indicator-ui'
detectedStack: frontend
generationMode: 'AI Generation (frontend story — Vitest + node:fs static + it.skip RTL)'
tddPhase: RED
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/10-12-collaborator-management-lock-indicator-ui.md'
  - 'eusolicit-app/frontend/apps/client/__tests__/espd-s11-13.test.ts'
  - 'eusolicit-app/frontend/apps/client/__tests__/tiptap-editor-s7-12.test.ts'
  - 'eusolicit-app/frontend/apps/client/vitest.config.ts'
  - 'eusolicit-app/frontend/apps/client/messages/en.json'
testFramework: 'Vitest (environment: node) + React Testing Library (it.skip for behavioural)'
---

# ATDD Checklist: Story 10.12 — Collaborator Management & Lock Indicator UI

**Story:** `10-12-collaborator-management-lock-indicator-ui`
**Date:** 2026-04-21
**TDD Phase:** 🔴 RED (behavioural `it.skip` — failing until implementation complete)
**Stack:** frontend (Next.js 14 App Router / React / Vitest / React Testing Library)
**Test Framework:** Vitest (`environment: "node"`) for structural tests; `it.skip` RTL for behavioural

> **Coverage note:** No epic-level test design doc exists for Epic 10 (see story Dev Notes).
> Coverage strategy follows established frontend story patterns:
> Vitest with `environment: "node"` for structural/static file-system assertions;
> `it.skip` + React Testing Library + `userEvent` for behavioural scenarios;
> `vi.useFakeTimers()` for blur-debounce and countdown tests;
> `vi.spyOn(navigator, "sendBeacon")` for pagehide tests;
> `QueryClientProvider` wrapping for TanStack Query hook tests.
> Stories 7.12 and 11.13 set the pattern for this story.

---

## TDD Red Phase Summary

✅ Failing tests generated — **1 test file**, **≥ 28 tests total** (14 structural + 14 behavioural/it.skip)

Structural tests fail immediately (files don't exist yet).
Behavioural tests use `it.skip` and compile green — markers come off per TDD activation sequence below.

---

## Generated Test Files

| File | Tests | Priority | ACs Covered |
|------|-------|----------|-------------|
| `apps/client/__tests__/collaborator-lock-s10-12.test.ts` | 28+ | P0/P1 | AC1–AC14 |

---

## Acceptance Criteria Coverage

### ✅ AC1 — File-system & Routing (six new components + seven API/query/store files)

**Structural (existsSync — all fail RED):**
- [ ] **`test_collaborator_panel_file_exists`** — `CollaboratorPanel.tsx` exists under `proposals/[id]/components/`.
- [ ] **`test_collaborator_row_file_exists`** — `CollaboratorRow.tsx` exists.
- [ ] **`test_add_collaborator_form_file_exists`** — `AddCollaboratorForm.tsx` exists.
- [ ] **`test_section_lock_indicator_file_exists`** — `SectionLockIndicator.tsx` exists.
- [ ] **`test_locked_section_toast_body_file_exists`** — `LockedSectionToastBody.tsx` exists.
- [ ] **`test_api_collaborators_file_exists`** — `lib/api/collaborators.ts` exists.
- [ ] **`test_api_section_locks_file_exists`** — `lib/api/section-locks.ts` exists.
- [ ] **`test_api_members_file_exists`** — `lib/api/members.ts` exists.
- [ ] **`test_queries_use_collaborators_file_exists`** — `lib/queries/use-collaborators.ts` exists.
- [ ] **`test_queries_use_section_locks_file_exists`** — `lib/queries/use-section-locks.ts` exists.
- [ ] **`test_queries_use_members_file_exists`** — `lib/queries/use-members.ts` exists.
- [ ] **`test_store_collaborator_store_file_exists`** — `lib/stores/collaborator-store.ts` exists.

**Modified files contain new additions (source text assertions):**
- [ ] **`test_proposal_workspace_page_has_toolbar_collaborators_btn`** — `ProposalWorkspacePage.tsx` contains `toolbar-btn-collaborators`.
- [ ] **`test_proposal_workspace_page_mounts_collaborator_panel`** — `ProposalWorkspacePage.tsx` contains `CollaboratorPanel`.
- [ ] **`test_proposal_workspace_page_has_collaborator_panel_open_state`** — `ProposalWorkspacePage.tsx` contains `collaboratorPanelOpen`.
- [ ] **`test_proposal_section_has_section_lock_indicator`** — `ProposalSection.tsx` contains `SectionLockIndicator`.
- [ ] **`test_proposal_editor_has_lock_state_gate_for_autosave`** — `ProposalEditor.tsx` contains `localLockState` and `"held"`.

### ✅ AC2 — `lib/api/collaborators.ts` exports

**Source text assertions:**
- [ ] **`test_collaborators_api_exports_collaborator_response_interface`** — source contains `export interface CollaboratorResponse`.
- [ ] **`test_collaborators_api_exports_collaborator_list_response`** — source contains `export interface CollaboratorListResponse`.
- [ ] **`test_collaborators_api_exports_add_collaborator_request`** — source contains `export interface AddCollaboratorRequest`.
- [ ] **`test_collaborators_api_exports_update_collaborator_role_request`** — source contains `export interface UpdateCollaboratorRoleRequest`.
- [ ] **`test_collaborators_api_exports_proposal_collaborator_role_value_type`** — source contains `export type ProposalCollaboratorRoleValue`.
- [ ] **`test_collaborators_api_has_all_five_role_values`** — source contains all of `bid_manager`, `technical_writer`, `financial_analyst`, `legal_reviewer`, `read_only`.
- [ ] **`test_collaborators_api_exports_list_collaborators_fn`** — source contains `export function listCollaborators` or `export const listCollaborators`.
- [ ] **`test_collaborators_api_exports_add_collaborator_fn`** — source contains `export function addCollaborator` or `export const addCollaborator`.
- [ ] **`test_collaborators_api_exports_update_collaborator_role_fn`** — source contains `export function updateCollaboratorRole` or `export const updateCollaboratorRole`.
- [ ] **`test_collaborators_api_exports_remove_collaborator_fn`** — source contains `export function removeCollaborator` or `export const removeCollaborator`.
- [ ] **`test_collaborators_api_list_collaborators_uses_correct_path`** — source contains `/api/v1/proposals/`.
- [ ] **`test_collaborators_api_remove_collaborator_204`** — source contains `204` or `DELETE`.

### ✅ AC3 — `lib/api/section-locks.ts` exports

**Source text assertions:**
- [ ] **`test_section_locks_api_exports_section_lock_info_interface`** — source contains `export interface SectionLockInfo`.
- [ ] **`test_section_locks_api_exports_section_with_lock_interface`** — source contains `export interface SectionWithLock`.
- [ ] **`test_section_locks_api_exports_sections_list_response`** — source contains `export interface SectionsListResponse`.
- [ ] **`test_section_locks_api_exports_section_lock_acquire_response`** — source contains `export interface SectionLockAcquireResponse`.
- [ ] **`test_section_locks_api_exports_section_lock_conflict_detail`** — source contains `export interface SectionLockConflictDetail`.
- [ ] **`test_section_locks_api_exports_list_sections_with_locks_fn`** — source contains `listSectionsWithLocks`.
- [ ] **`test_section_locks_api_exports_acquire_section_lock_fn`** — source contains `acquireSectionLock`.
- [ ] **`test_section_locks_api_exports_release_section_lock_fn`** — source contains `releaseSectionLock`.
- [ ] **`test_section_locks_api_exports_release_section_lock_beacon_fn`** — source contains `releaseSectionLockBeacon`.
- [ ] **`test_section_locks_api_acquire_returns_conflict_on_423`** — source contains `conflict` shape handling (does NOT throw on 423).
- [ ] **`test_section_locks_api_beacon_uses_send_beacon`** — source contains `navigator.sendBeacon`.
- [ ] **`test_section_locks_api_beacon_fallback_uses_fetch_keepalive`** — source contains `keepalive: true`.

### ✅ AC4 — `lib/queries/use-collaborators.ts` exports

**Source text assertions:**
- [ ] **`test_use_collaborators_hook_exported`** — source contains `export function useCollaborators`.
- [ ] **`test_use_my_proposal_role_hook_exported`** — source contains `export function useMyProposalRole`.
- [ ] **`test_use_add_collaborator_hook_exported`** — source contains `export function useAddCollaborator`.
- [ ] **`test_use_update_collaborator_role_hook_exported`** — source contains `export function useUpdateCollaboratorRole`.
- [ ] **`test_use_remove_collaborator_hook_exported`** — source contains `export function useRemoveCollaborator`.
- [ ] **`test_use_collaborators_query_key_plural`** — source contains `"collaborators"` (plural cache key).
- [ ] **`test_use_collaborators_stale_time_30000`** — source contains `staleTime: 30_000` or `staleTime: 30000`.
- [ ] **`test_use_my_proposal_role_handles_admin_bypass`** — source contains `"admin"` role bypass check.
- [ ] **`test_use_add_collaborator_invalidates_collaborators_cache`** — source contains `invalidateQueries` and `"collaborators"`.
- [ ] **`test_use_add_collaborator_409_toast`** — source contains `409` or `duplicateCollaborator` error handling.
- [ ] **`test_use_remove_collaborator_409_toast_last_bid_manager`** — source contains `lastBidManager` toast key.

### ✅ AC5 — `lib/queries/use-section-locks.ts` exports

**Source text assertions:**
- [ ] **`test_use_section_locks_hook_exported`** — source contains `export function useSectionLocks`.
- [ ] **`test_use_acquire_section_lock_hook_exported`** — source contains `export function useAcquireSectionLock`.
- [ ] **`test_use_release_section_lock_hook_exported`** — source contains `export function useReleaseSectionLock`.
- [ ] **`test_use_section_locks_refetch_interval_30000`** — source contains `refetchInterval: 30_000` or `refetchInterval: 30000`.
- [ ] **`test_use_section_locks_refetch_interval_in_background_false`** — source contains `refetchIntervalInBackground: false`.
- [ ] **`test_use_section_locks_refetch_on_window_focus_true`** — source contains `refetchOnWindowFocus: true`.
- [ ] **`test_use_section_locks_stale_time_0`** — source contains `staleTime: 0`.
- [ ] **`test_use_acquire_section_lock_writes_foreign_lock_conflict`** — source contains `setForeignLockConflict`.

### ✅ AC6 — `CollaboratorPanel` slide-over structure

**Component source text assertions:**
- [ ] **`test_collaborator_panel_has_root_test_id`** — `CollaboratorPanel.tsx` contains `data-testid="collaborator-panel"`.
- [ ] **`test_collaborator_panel_has_header_test_id`** — source contains `data-testid="collaborator-panel-header"`.
- [ ] **`test_collaborator_panel_has_title_test_id`** — source contains `data-testid="collaborator-panel-title"`.
- [ ] **`test_collaborator_panel_has_close_btn_test_id`** — source contains `data-testid="collaborator-panel-close-btn"`.
- [ ] **`test_collaborator_panel_has_skeleton_test_id`** — source contains `data-testid="collaborator-panel-skeleton"`.
- [ ] **`test_collaborator_panel_has_error_test_id`** — source contains `data-testid="collaborator-panel-error"`.
- [ ] **`test_collaborator_panel_has_empty_test_id`** — source contains `data-testid="collaborator-panel-empty"`.
- [ ] **`test_collaborator_panel_has_collaborator_list_test_id`** — source contains `data-testid="collaborator-list"`.
- [ ] **`test_collaborator_panel_has_add_section_title_test_id`** — source contains `data-testid="add-collaborator-section-title"`.
- [ ] **`test_collaborator_panel_has_readonly_note_test_id`** — source contains `data-testid="collaborator-panel-readonly-note"`.
- [ ] **`test_collaborator_panel_uses_use_collaborators_hook`** — source contains `useCollaborators`.
- [ ] **`test_collaborator_panel_uses_use_my_proposal_role_hook`** — source contains `useMyProposalRole`.
- [ ] **`test_collaborator_panel_has_use_client_directive`** — source contains `"use client"`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_collaborator_panel_renders_skeleton_when_loading`** (it.skip) — RTL: renders `collaborator-panel-skeleton` when `useCollaborators` is loading.
- [ ] **`test_collaborator_panel_renders_empty_state_when_no_collaborators`** (it.skip) — RTL: renders `collaborator-panel-empty` when `data.items.length === 0`.
- [ ] **`test_collaborator_panel_renders_readonly_note_when_cannot_manage`** (it.skip) — RTL: `collaborator-panel-readonly-note` visible when `myRole === "technical_writer"`.
- [ ] **`test_collaborator_panel_renders_add_form_when_can_manage`** (it.skip) — RTL: `add-collaborator-form` visible when `myRole === "bid_manager"`.

### ✅ AC7 — `CollaboratorRow` structure

**Component source text assertions:**
- [ ] **`test_collaborator_row_has_per_row_test_id`** — `CollaboratorRow.tsx` contains `` `collaborator-row-${ ``.
- [ ] **`test_collaborator_row_has_avatar_test_id`** — source contains `` `collaborator-avatar-${ ``.
- [ ] **`test_collaborator_row_has_name_test_id`** — source contains `` `collaborator-name-${ ``.
- [ ] **`test_collaborator_row_has_email_test_id`** — source contains `` `collaborator-email-${ ``.
- [ ] **`test_collaborator_row_has_role_badge_test_id`** — source contains `` `collaborator-role-badge-${ ``.
- [ ] **`test_collaborator_row_has_role_select_test_id`** — source contains `` `collaborator-role-select-${ ``.
- [ ] **`test_collaborator_row_has_remove_btn_test_id`** — source contains `` `collaborator-remove-btn-${ ``.
- [ ] **`test_collaborator_row_has_granted_tooltip_test_id`** — source contains `` `collaborator-granted-tooltip-${ ``.
- [ ] **`test_collaborator_row_has_remove_dialog_test_id`** — source contains `data-testid="collaborator-remove-dialog"`.
- [ ] **`test_collaborator_row_has_use_client_directive`** — source contains `"use client"`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_collaborator_row_hides_remove_btn_when_cannot_manage`** (it.skip) — RTL: `collaborator-remove-btn-{userId}` absent when `canManage === false`.
- [ ] **`test_collaborator_row_hides_role_select_when_cannot_manage`** (it.skip) — RTL: `collaborator-role-select-{userId}` absent when `canManage === false`.
- [ ] **`test_collaborator_row_shows_confirmation_dialog_on_remove_click`** (it.skip) — RTL: clicking remove btn opens `collaborator-remove-dialog`.

### ✅ AC8 — `AddCollaboratorForm` structure

**Component source text assertions:**
- [ ] **`test_add_collaborator_form_has_form_test_id`** — `AddCollaboratorForm.tsx` contains `data-testid="add-collaborator-form"`.
- [ ] **`test_add_collaborator_form_has_user_search_test_id`** — source contains `data-testid="add-collaborator-user-search"`.
- [ ] **`test_add_collaborator_form_has_role_select_test_id`** — source contains `data-testid="add-collaborator-role-select"`.
- [ ] **`test_add_collaborator_form_has_submit_btn_test_id`** — source contains `data-testid="add-collaborator-submit-btn"`.
- [ ] **`test_add_collaborator_form_default_role_technical_writer`** — source contains `"technical_writer"` as default role value.
- [ ] **`test_add_collaborator_form_uses_zod_schema`** — source contains `z.object` or `z.enum`.
- [ ] **`test_add_collaborator_form_uses_use_company_members`** — source contains `useCompanyMembers`.
- [ ] **`test_add_collaborator_form_has_use_client_directive`** — source contains `"use client"`.
- [ ] **`test_proposal_editor_auto_save_gated_by_lock_state`** — `ProposalEditor.tsx` source contains `lockState !== "held"` or `localLockState` check before `doAutoSave`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_add_collaborator_form_disabled_when_user_id_empty`** (it.skip) — RTL: submit button disabled when no user selected.
- [ ] **`test_add_collaborator_form_excludes_existing_users_from_search`** (it.skip) — RTL: already-added user IDs absent from dropdown options.
- [ ] **`test_remove_collaborator_409_keeps_dialog_open_with_inline_error`** (it.skip) — RTL: 409 response keeps `collaborator-remove-dialog` open with `lastBidManagerError` text.

### ✅ AC9 — `SectionLockIndicator` + `LockCountdown`

**Component source text assertions:**
- [ ] **`test_section_lock_indicator_has_self_pill_test_id`** — `SectionLockIndicator.tsx` contains `` `section-lock-self-${ ``.
- [ ] **`test_section_lock_indicator_has_foreign_pill_test_id`** — source contains `` `section-lock-foreign-${ ``.
- [ ] **`test_section_lock_indicator_has_foreign_tooltip_test_id`** — source contains `` `section-lock-foreign-tooltip-${ ``.
- [ ] **`test_section_lock_indicator_exports_lock_countdown`** — source contains `export` and `LockCountdown`.
- [ ] **`test_lock_countdown_uses_set_interval`** — source contains `setInterval`.
- [ ] **`test_lock_countdown_stops_on_zero`** — source contains countdown expiry stop logic (`remaining <= 0` or `clearInterval`).
- [ ] **`test_lock_countdown_is_memoised`** — source contains `React.memo` or `memo(`.
- [ ] **`test_toolbar_lock_summary_badge_test_id_in_workspace`** — `ProposalWorkspacePage.tsx` or `SectionLockIndicator.tsx` contains `data-testid="toolbar-lock-summary-badge"`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_section_lock_indicator_renders_nothing_when_lock_is_null`** (it.skip) — RTL: `SectionLockIndicator` returns null when `lock === null`.
- [ ] **`test_section_lock_indicator_renders_self_pill_when_lock_is_current_user`** (it.skip) — RTL: `section-lock-self-{key}` visible when `isCurrentUser === true`.
- [ ] **`test_section_lock_indicator_renders_foreign_pill_when_lock_is_other_user`** (it.skip) — RTL: `section-lock-foreign-{key}` visible when `isCurrentUser === false`.

### ✅ AC10 — `useCollaboratorStore` (Zustand)

**Source text assertions:**
- [ ] **`test_collaborator_store_exports_use_collaborator_store`** — `lib/stores/collaborator-store.ts` contains `export const useCollaboratorStore`.
- [ ] **`test_collaborator_store_has_local_lock_state`** — source contains `localLockState`.
- [ ] **`test_collaborator_store_has_set_local_lock_state`** — source contains `setLocalLockState`.
- [ ] **`test_collaborator_store_has_clear_local_lock_states`** — source contains `clearLocalLockStates`.
- [ ] **`test_collaborator_store_has_foreign_lock_conflicts`** — source contains `foreignLockConflicts`.
- [ ] **`test_collaborator_store_has_set_foreign_lock_conflict`** — source contains `setForeignLockConflict`.
- [ ] **`test_collaborator_store_has_pending_release_timers`** — source contains `pendingReleaseTimers`.
- [ ] **`test_collaborator_store_has_set_pending_release_timer`** — source contains `setPendingReleaseTimer`.
- [ ] **`test_collaborator_store_has_no_persist_middleware`** — source does NOT contain `persist(` middleware.
- [ ] **`test_collaborator_store_local_lock_state_type_values`** — source contains all of `"idle"`, `"acquiring"`, `"held"`, `"foreign"`, `"releasing"`.

### ✅ AC11 — `ProposalSection` modifications

**Source text assertions:**
- [ ] **`test_proposal_section_has_section_lock_indicator_render`** — `ProposalSection.tsx` contains `SectionLockIndicator`.
- [ ] **`test_proposal_section_uses_use_section_locks_hook`** — source contains `useSectionLocks`.
- [ ] **`test_proposal_section_uses_use_acquire_section_lock_hook`** — source contains `useAcquireSectionLock`.
- [ ] **`test_proposal_section_uses_use_release_section_lock_hook`** — source contains `useReleaseSectionLock`.
- [ ] **`test_proposal_section_blur_handler_has_800ms_debounce`** — source contains `800` debounce constant.
- [ ] **`test_proposal_section_focus_handler_cancels_pending_release`** — source contains `clearTimeout` in focus context.
- [ ] **`test_proposal_section_editor_editable_gated_by_lock_state`** — source contains `setEditable` and lock state condition.
- [ ] **`test_proposal_editor_has_pagehide_handler`** — `ProposalEditor.tsx` contains `pagehide`.
- [ ] **`test_proposal_editor_has_visibility_change_handler`** — `ProposalEditor.tsx` contains `visibilitychange`.
- [ ] **`test_proposal_editor_unmount_clears_lock_states`** — `ProposalEditor.tsx` contains `clearLocalLockStates`.
- [ ] **`test_proposal_section_optimistic_acquiring_state`** — source contains `setLocalLockState` and `"acquiring"`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_acquire_section_lock_optimistic_state_transitions_to_held`** (it.skip) — mocks `acquireSectionLock` → 200; `localLockState["key"]` transitions to `"held"`.
- [ ] **`test_acquire_section_lock_423_writes_foreign_conflict_and_blurs_editor`** (it.skip) — mocks API → `{ conflict: ... }`; `setForeignLockConflict` called; `editor.setEditable(false)` called.
- [ ] **`test_blur_release_is_cancelled_if_focus_returns_within_800ms`** (it.skip) — `vi.useFakeTimers()`; blur fires; focus returns within 800 ms; `releaseSectionLock` NOT called.
- [ ] **`test_pagehide_handler_fires_beacon_for_held_locks`** (it.skip) — `vi.spyOn(navigator, "sendBeacon")`; `localLockState` has `"held"` entries; `pagehide` event fires; beacon called with correct path.
- [ ] **`test_section_editor_set_editable_false_when_foreign_lock_active`** (it.skip) — mocks `useSectionLocks` returning foreign lock; asserts editor `editable === false`.
- [ ] **`test_auto_save_suppressed_when_lock_state_not_held`** (it.skip) — mocks `apiClient.patch`; `localLockState["key"] === "acquiring"`; auto-save fires; `apiClient.patch` NOT called; warning toast emitted.

### ✅ AC12 — Toolbar integration (`ProposalWorkspacePage`)

**Source text assertions:**
- [ ] **`test_proposal_workspace_page_has_toolbar_collaborators_btn`** — `ProposalWorkspacePage.tsx` contains `data-testid="toolbar-btn-collaborators"`.
- [ ] **`test_proposal_workspace_page_mounts_collaborator_panel_component`** — source contains `<CollaboratorPanel`.
- [ ] **`test_proposal_workspace_page_has_collaborator_panel_open_state`** — source contains `collaboratorPanelOpen`.
- [ ] **`test_proposal_workspace_page_uses_use_collaborators_hook`** — source contains `useCollaborators`.
- [ ] **`test_toolbar_lock_summary_badge_present`** — `ProposalWorkspacePage.tsx` or `ProposalToolbar`-related source contains `toolbar-lock-summary-badge`.

### ✅ AC13 — i18n keys (`messages/en.json` + `messages/bg.json`)

**i18n parity assertions (structural):**
- [ ] **`test_i18n_en_has_proposals_collaborators_namespace`** — `en.json` contains `proposals.collaborators` nested object.
- [ ] **`test_i18n_bg_has_proposals_collaborators_namespace`** — `bg.json` contains `proposals.collaborators` nested object.
- [ ] **`test_i18n_en_has_proposals_locks_namespace`** — `en.json` contains `proposals.locks` nested object.
- [ ] **`test_i18n_bg_has_proposals_locks_namespace`** — `bg.json` contains `proposals.locks` nested object.
- [ ] **`test_i18n_collaborators_en_bg_key_parity`** — every key in `en.proposals.collaborators` is present in `bg.proposals.collaborators` and vice versa.
- [ ] **`test_i18n_locks_en_bg_key_parity`** — every key in `en.proposals.locks` is present in `bg.proposals.locks` and vice versa.
- [ ] **`test_i18n_collaborators_roles_cover_all_five_values`** — `en.proposals.collaborators.roles` has exactly the five keys: `bid_manager`, `technical_writer`, `financial_analyst`, `legal_reviewer`, `read_only`.
- [ ] **`test_i18n_en_collaborators_panel_title_key`** — `en.proposals.collaborators.panelTitle` is `"Collaborators"`.
- [ ] **`test_i18n_en_locks_editing_pill_key`** — `en.proposals.locks.editingPill` is `"Editing"`.
- [ ] **`test_i18n_en_locks_expired_label_key`** — `en.proposals.locks.expiredLabel` is `"expired"`.

**Polling / hook asserts (structural source assertions):**
- [ ] **`test_use_section_locks_uses_30s_refetch_interval`** — `lib/queries/use-section-locks.ts` contains `refetchInterval: 30_000` or `refetchInterval: 30000`.
- [ ] **`test_use_section_locks_refetches_on_window_focus`** — source contains `refetchOnWindowFocus: true`.

---

## Not-yet-covered (defer to Implementation / bmad-testarch-automate)

| AC | Gap | Reason |
|----|-----|--------|
| AC9 | Visual countdown tick accuracy (MM:SS format at 1s vs 5s) | Requires real timer execution; deferred to GREEN with fake timers |
| AC11 | `doFullSave` suppression when multiple sections have foreign locks | Full-save path is complex; covered by unit test concept; deferred to GREEN |
| AC11 | `pagehide` + `visibilitychange > 5s` distinction | Browser event subtleties; covered by `pagehide` beacon test; `visibilitychange` deferred |
| AC13 | `pnpm check:i18n` script exit code | External tool; structural key-parity test covers same guarantee |
| AC14 | `ProposalSection.tsx` section-header integration rendering end-to-end | Full editor mount complex; structural testid presence assertion substitutes |

---

## Risk Traceability

| Risk | Test | Status |
|------|------|--------|
| Collaborator panel missing root testid | `test_collaborator_panel_has_root_test_id` | 🔴 RED |
| `read_only` role user sees manage controls | `test_collaborator_panel_renders_readonly_note_when_cannot_manage` | 🔴 RED (it.skip) |
| Foreign lock 423 doesn't surface conflict toast | `test_acquire_section_lock_423_writes_foreign_conflict_and_blurs_editor` | 🔴 RED (it.skip) |
| Blur-release fires before refocus (flap) | `test_blur_release_is_cancelled_if_focus_returns_within_800ms` | 🔴 RED (it.skip) |
| Auto-save fires without held lock | `test_auto_save_suppressed_when_lock_state_not_held` | 🔴 RED (it.skip) |
| pagehide doesn't beacon held locks | `test_pagehide_handler_fires_beacon_for_held_locks` | 🔴 RED (it.skip) |
| Editor stays editable during foreign lock | `test_section_editor_set_editable_false_when_foreign_lock_active` | 🔴 RED (it.skip) |
| Collaborator store persists across proposals | `test_collaborator_store_has_no_persist_middleware` | 🔴 RED |
| i18n EN/BG key drift | `test_i18n_collaborators_en_bg_key_parity` | 🔴 RED |
| i18n locks namespace drift | `test_i18n_locks_en_bg_key_parity` | 🔴 RED |
| Polling continues in background | `test_use_section_locks_refetch_interval_in_background_false` | 🔴 RED |
| 409 last bid_manager removes user anyway | `test_remove_collaborator_409_keeps_dialog_open_with_inline_error` | 🔴 RED (it.skip) |
| acquireSectionLock throws on 423 instead of returning conflict | `test_section_locks_api_acquire_returns_conflict_on_423` | 🔴 RED |
| Collaborator cache key singular (breaks invalidation) | `test_use_collaborators_query_key_plural` | 🔴 RED |

---

## TDD Red → Green Activation Sequence

Remove `it.skip` markers in this order:

### Phase 1: File-system structure (AC1) — pure existsSync
```bash
cd eusolicit-app/frontend
pnpm test --filter client -- --reporter=verbose collaborator-lock-s10-12
```
All 12 `existsSync` tests must pass (files now created).

### Phase 2: API client signatures (AC2, AC3)
```bash
pnpm test --filter client -- --reporter=verbose collaborator-lock-s10-12 \
  -t "collaborators_api|section_locks_api"
```

### Phase 3: Query hook exports (AC4, AC5)
```bash
pnpm test --filter client -- --reporter=verbose collaborator-lock-s10-12 \
  -t "use_collaborators|use_section_locks|use_members"
```

### Phase 4: Zustand store contract (AC10)
```bash
pnpm test --filter client -- --reporter=verbose collaborator-lock-s10-12 \
  -t "collaborator_store"
```

### Phase 5: Component testid presence (AC6, AC7, AC8, AC9, AC12)
```bash
pnpm test --filter client -- --reporter=verbose collaborator-lock-s10-12 \
  -t "collaborator_panel|collaborator_row|add_collaborator_form|section_lock_indicator|toolbar"
```

### Phase 6: i18n parity (AC13)
```bash
pnpm test --filter client -- --reporter=verbose collaborator-lock-s10-12 \
  -t "i18n"
```

### Phase 7: Modified file integrations (AC11, AC12)
```bash
pnpm test --filter client -- --reporter=verbose collaborator-lock-s10-12 \
  -t "proposal_section|proposal_editor|proposal_workspace"
```

### Phase 8: Activate behavioural RTL tests (remove it.skip)
In `collaborator-lock-s10-12.test.ts`, remove `it.skip` prefixes one block at a time:
1. Panel skeleton / empty state / readonly note / add form
2. Row hide controls / confirmation dialog
3. Form disabled / excludes existing users / 409 keeps dialog
4. Lock indicator null / self / foreign
5. Lock acquire optimistic → held
6. Lock acquire 423 → conflict + blur
7. Blur-release cancelled within 800ms
8. pagehide beacon
9. editor setEditable(false) on foreign lock
10. auto-save suppressed when not held

### Phase 9: Full suite
```bash
pnpm test --filter client -- --reporter=verbose collaborator-lock-s10-12
```

---

## Quality Gate

- [ ] All 28+ tests GREEN (it.skip removed from behavioural tests)
- [ ] `pnpm check:i18n` exits `0` (EN/BG parity, all new keys present)
- [ ] TypeScript type-check passes: `pnpm tsc --noEmit --filter client`
- [ ] Lint passes: `pnpm lint --filter client`
- [ ] `test_i18n_collaborators_en_bg_key_parity` GREEN — zero drift
- [ ] `test_i18n_locks_en_bg_key_parity` GREEN — zero drift
- [ ] `test_collaborator_store_has_no_persist_middleware` GREEN — no stale cross-proposal state
- [ ] `test_acquire_section_lock_423_writes_foreign_conflict_and_blurs_editor` GREEN — 423 path safe
- [ ] `test_auto_save_suppressed_when_lock_state_not_held` GREEN — no phantom saves
- [ ] `test_blur_release_is_cancelled_if_focus_returns_within_800ms` GREEN — no flap
- [ ] No regression on `tiptap-editor-s7-12.test.ts` (Story 7.12 — all existing tests still green)
- [ ] No regression on `proposals-workspace-s7-11.test.ts` (Story 7.11 — all existing tests still green)

---

## Implementation Files Expected (New)

| File | Required By |
|------|-------------|
| `apps/client/app/[locale]/(protected)/proposals/[id]/components/CollaboratorPanel.tsx` | AC1, AC6 |
| `apps/client/app/[locale]/(protected)/proposals/[id]/components/CollaboratorRow.tsx` | AC1, AC7 |
| `apps/client/app/[locale]/(protected)/proposals/[id]/components/AddCollaboratorForm.tsx` | AC1, AC8 |
| `apps/client/app/[locale]/(protected)/proposals/[id]/components/SectionLockIndicator.tsx` | AC1, AC9 |
| `apps/client/app/[locale]/(protected)/proposals/[id]/components/LockedSectionToastBody.tsx` | AC1, AC9 |
| `apps/client/lib/api/collaborators.ts` | AC2 |
| `apps/client/lib/api/section-locks.ts` | AC3 |
| `apps/client/lib/api/members.ts` | AC1 |
| `apps/client/lib/queries/use-collaborators.ts` | AC4 |
| `apps/client/lib/queries/use-section-locks.ts` | AC5 |
| `apps/client/lib/queries/use-members.ts` | AC1 |
| `apps/client/lib/stores/collaborator-store.ts` | AC10 |

## Implementation Files Expected (Modified)

| File | Change |
|------|--------|
| `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalWorkspacePage.tsx` | Toolbar collaborators btn + panel mount + lock badge |
| `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalEditor.tsx` | Lock state gate on auto-save + pagehide/visibilitychange handler + unmount cleanup |
| `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalSection.tsx` | SectionLockIndicator render + focus/blur handlers + editor editable gating |
| `apps/client/messages/en.json` | `proposals.collaborators.*` + `proposals.locks.*` namespaces |
| `apps/client/messages/bg.json` | Bulgarian parity for all new keys |

---

**Generated by:** BMad TEA Agent — ATDD Workflow (bmad-testarch-atdd)
**Story:** 10-12-collaborator-management-lock-indicator-ui
**Date:** 2026-04-21
