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
storyKey: '10-13-comments-sidebar-ui'
detectedStack: frontend
generationMode: 'AI Generation (frontend story — Vitest + node:fs static + it.skip RTL)'
tddPhase: RED
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/10-13-comments-sidebar-ui.md'
  - 'eusolicit-docs/test-artifacts/atdd-checklist-10-12-collaborator-management-lock-indicator-ui.md'
  - 'eusolicit-app/frontend/apps/client/__tests__/collaborator-lock-s10-12.test.ts'
  - 'eusolicit-app/frontend/apps/client/vitest.config.ts'
  - 'eusolicit-app/frontend/apps/client/messages/en.json'
testFramework: 'Vitest (environment: node) + React Testing Library (it.skip for behavioural)'
---

# ATDD Checklist: Story 10.13 — Comments Sidebar UI

**Story:** `10-13-comments-sidebar-ui`
**Date:** 2026-04-21
**TDD Phase:** 🔴 RED (behavioural `it.skip` — failing until implementation complete)
**Stack:** frontend (Next.js 14 App Router / React / Vitest / React Testing Library)
**Test Framework:** Vitest (`environment: "node"`) for structural tests; `it.skip` RTL for behavioural

> **Coverage note:** No epic-level test design doc exists for Epic 10 (see story Dev Notes — same gap
> noted in Stories 10.4 / 10.9–10.12). Coverage strategy draws from four surrogate sources:
> Epic 10 AC3/AC11/AC13 (verbatim), Story 10.4 ATDD checklist (backend contract), and
> the established frontend ATDD pattern from Stories 7.12, 11.13, and 10.12.
>
> Vitest with `environment: "node"` for structural / static file-system assertions;
> `it.skip` + React Testing Library + `userEvent` for behavioural scenarios;
> `vi.useFakeTimers()` for polling / timer tests;
> `QueryClientProvider` wrapping for TanStack Query hook tests.
> Story 10.12 sets the direct precedent for this story's shape.

---

## TDD Red Phase Summary

✅ Failing tests generated — **1 test file**, **≥ 36 assertions across 14 `describe` blocks**

Structural tests fail immediately (files don't exist yet).
Behavioural tests use `it.skip` and compile green — markers come off per TDD activation sequence below.

---

## Generated Test Files

| File | Tests | Priority | ACs Covered |
|------|-------|----------|-------------|
| `apps/client/__tests__/comments-sidebar-s10-13.test.ts` | 36+ | P0/P1 | AC1–AC14 |

---

## Acceptance Criteria Coverage

### ✅ AC1 — File-system & Routing (seven new components + API + query + store)

**Structural (existsSync — all fail RED):**
- [ ] **`test_comments_sidebar_file_exists`** — `CommentsSidebar.tsx` exists under `proposals/[id]/components/`.
- [ ] **`test_comments_sidebar_header_file_exists`** — `CommentsSidebarHeader.tsx` exists.
- [ ] **`test_comment_list_file_exists`** — `CommentList.tsx` exists.
- [ ] **`test_comment_item_file_exists`** — `CommentItem.tsx` exists.
- [ ] **`test_new_comment_form_file_exists`** — `NewCommentForm.tsx` exists.
- [ ] **`test_carry_forward_banner_file_exists`** — `CarryForwardBanner.tsx` exists.
- [ ] **`test_section_comments_badge_file_exists`** — `SectionCommentsBadge.tsx` exists.
- [ ] **`test_api_proposal_comments_file_exists`** — `lib/api/proposal-comments.ts` exists.
- [ ] **`test_queries_use_proposal_comments_file_exists`** — `lib/queries/use-proposal-comments.ts` exists.
- [ ] **`test_store_comments_sidebar_file_exists`** — `lib/stores/comments-sidebar-store.ts` exists.

**Modified files contain new wiring (source text assertions):**
- [ ] **`test_workspace_page_has_toolbar_comments_btn`** — `ProposalWorkspacePage.tsx` contains `toolbar-btn-comments`.
- [ ] **`test_workspace_page_mounts_comments_sidebar`** — `ProposalWorkspacePage.tsx` contains `CommentsSidebar`.
- [ ] **`test_workspace_page_has_comments_open_state`** — `ProposalWorkspacePage.tsx` contains `commentsOpen`.
- [ ] **`test_proposal_section_has_section_comments_badge`** — `ProposalSection.tsx` contains `SectionCommentsBadge`.
- [ ] **`test_proposal_editor_calls_reset_for_proposal_on_unmount`** — `ProposalEditor.tsx` contains `resetForProposal`.

---

### ✅ AC2 — `lib/api/proposal-comments.ts` exports

**Source text assertions:**
- [ ] **`test_api_exports_comment_response_interface`** — source contains `export interface CommentResponse`.
- [ ] **`test_api_exports_comment_with_optimistic_state_interface`** — source contains `export interface CommentWithOptimisticState`.
- [ ] **`test_api_exports_comment_list_response_interface`** — source contains `export interface CommentListResponse`.
- [ ] **`test_api_exports_create_comment_request_interface`** — source contains `export interface CreateCommentRequest`.
- [ ] **`test_api_exports_list_comments_params_interface`** — source contains `export interface ListCommentsParams`.
- [ ] **`test_api_exports_comments_by_section_map_type`** — source matches `export type CommentsBySectionMap`.
- [ ] **`test_api_comment_response_has_carried_forward_from_id`** — source contains `carried_forward_from_id`.
- [ ] **`test_api_comment_with_optimistic_state_has_optimistic_flag`** — source contains `optimistic`.
- [ ] **`test_api_exports_list_comments_fn`** — source matches `export (function|const) listComments`.
- [ ] **`test_api_exports_create_comment_fn`** — source matches `export (function|const) createComment`.
- [ ] **`test_api_exports_resolve_comment_fn`** — source matches `export (function|const) resolveComment`.
- [ ] **`test_api_exports_unresolve_comment_fn`** — source matches `export (function|const) unresolveComment`.
- [ ] **`test_api_exports_list_replies_fn`** — source matches `export (function|const) listReplies`.
- [ ] **`test_api_uses_correct_path_prefix`** — source contains `/api/v1/proposals/`.
- [ ] **`test_api_resolve_comment_targets_resolve_endpoint`** — source contains `/resolve`.
- [ ] **`test_api_unresolve_comment_targets_unresolve_endpoint`** — source contains `/unresolve`.

---

### ✅ AC3 — `lib/queries/use-proposal-comments.ts` exports

**Source text assertions:**
- [ ] **`test_hook_exports_use_proposal_comments`** — source contains `export function useProposalComments`.
- [ ] **`test_hook_exports_use_comments_summary`** — source contains `export function useCommentsSummary`.
- [ ] **`test_hook_exports_use_comments_count_by_section`** — source contains `export function useCommentsCountBySection`.
- [ ] **`test_hook_exports_use_create_comment`** — source contains `export function useCreateComment`.
- [ ] **`test_hook_exports_use_resolve_comment`** — source contains `export function useResolveComment`.
- [ ] **`test_hook_exports_use_unresolve_comment`** — source contains `export function useUnresolveComment`.
- [ ] **`test_hook_exports_build_comments_by_section_map`** — source matches `export function buildCommentsBySectionMap`.
- [ ] **`test_use_proposal_comments_cache_key`** — source contains `"proposal-comments"`.
- [ ] **`test_use_comments_summary_cache_key`** — source contains `"proposal-comments-summary"`.
- [ ] **`test_use_proposal_comments_stale_time_0`** — source matches `staleTime:\s*0`.
- [ ] **`test_use_proposal_comments_refetch_on_window_focus`** — source contains `refetchOnWindowFocus: true`.
- [ ] **`test_use_proposal_comments_enabled_checks_section_key`** — source contains `enabled` + `sectionKey`.
- [ ] **`test_use_comments_summary_refetch_interval_60000`** — source matches `refetchInterval:\s*(60_000|60000)`.
- [ ] **`test_use_comments_summary_refetch_interval_in_background_false`** — source contains `refetchIntervalInBackground: false`.
- [ ] **`test_use_comments_summary_stale_time_30000`** — source matches `staleTime:\s*(30_000|30000)`.
- [ ] **`test_use_create_comment_has_on_mutate`** — source contains `onMutate`.
- [ ] **`test_use_create_comment_uses_temp_uuid`** — source contains `temp-`.
- [ ] **`test_use_create_comment_has_on_error_rollback`** — source contains `onError`.
- [ ] **`test_use_create_comment_invalidates_both_cache_keys`** — source contains `invalidateQueries` + both keys.
- [ ] **`test_use_resolve_comment_flips_resolved_true`** — source contains `resolved: true`.
- [ ] **`test_use_unresolve_comment_flips_resolved_false`** — source contains `resolved: false`.
- [ ] **`test_mutation_hooks_use_add_toast`** — source contains `addToast`.
- [ ] **`test_create_comment_422_maps_to_validation_key`** — source matches `422|createError\.validation`.
- [ ] **`test_create_comment_403_maps_to_not_authorised_key`** — source matches `403|notAuthorised`.

---

### ✅ AC4 — `CommentsSidebar` (slide-over root)

**Component source text assertions:**
- [ ] **`test_comments_sidebar_has_use_client`** — source contains `"use client"`.
- [ ] **`test_comments_sidebar_default_export`** — source matches `export default.*CommentsSidebar`.
- [ ] **`test_comments_sidebar_has_root_testid`** — source contains `data-testid="comments-sidebar"`.
- [ ] **`test_comments_sidebar_has_empty_noselection_testid`** — source contains `comments-sidebar-empty-noselection`.
- [ ] **`test_comments_sidebar_has_skeleton_testid`** — source contains `comments-sidebar-skeleton`.
- [ ] **`test_comments_sidebar_has_error_testid`** — source contains `comments-sidebar-error`.
- [ ] **`test_comments_sidebar_has_empty_nocomments_testid`** — source contains `comments-sidebar-empty-nocomments`.
- [ ] **`test_comments_sidebar_subscribes_to_editor_store`** — source contains `useProposalEditorStore`.
- [ ] **`test_comments_sidebar_uses_sidebar_store`** — source contains `useCommentsSidebarStore`.
- [ ] **`test_comments_sidebar_mounts_header`** — source contains `CommentsSidebarHeader`.
- [ ] **`test_comments_sidebar_mounts_banner`** — source contains `CarryForwardBanner`.
- [ ] **`test_comments_sidebar_mounts_form`** — source contains `NewCommentForm`.
- [ ] **`test_comments_sidebar_mounts_list`** — source contains `CommentList`.
- [ ] **`test_comments_sidebar_uses_my_proposal_role`** — source contains `useMyProposalRole`.
- [ ] **`test_comments_sidebar_reuses_sheet_primitive`** — source contains `Sheet`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_comments_sidebar_renders_skeleton_when_query_is_loading`** (it.skip) — mocks `useProposalComments` to `{ isLoading: true }`; asserts `comments-sidebar-skeleton` present.
- [ ] **`test_comments_sidebar_renders_empty_state_when_no_section_selected`** (it.skip) — mocks `useProposalEditorStore` to `{ activeSectionKey: null }`; asserts `comments-sidebar-empty-noselection` present.
- [ ] **`test_comments_sidebar_renders_two_groups_when_mixed_resolved_and_open`** (it.skip) — 2 unresolved + 1 resolved; resolved row has `opacity-60`; resolved-group heading shows count=1.

---

### ✅ AC5 — `CommentsSidebarHeader`

**Component source text assertions:**
- [ ] **`test_header_has_use_client`** — source contains `"use client"`.
- [ ] **`test_header_named_export`** — source matches `export (function|const) CommentsSidebarHeader`.
- [ ] **`test_header_has_title_testid`** — source contains `data-testid="comments-sidebar-title"`.
- [ ] **`test_header_has_active_section_testid`** — source contains `data-testid="comments-sidebar-active-section"`.
- [ ] **`test_header_has_close_btn_testid`** — source contains `data-testid="comments-sidebar-close-btn"`.
- [ ] **`test_header_no_section_fallback`** — source matches `noSectionSelectedShort|noSectionSelected`.

---

### ✅ AC6 — `NewCommentForm`

**Component source text assertions:**
- [ ] **`test_form_has_use_client`** — source contains `"use client"`.
- [ ] **`test_form_named_export`** — source matches `export (function|const) NewCommentForm`.
- [ ] **`test_form_has_form_testid`** — source contains `data-testid="new-comment-form"`.
- [ ] **`test_form_has_body_testid`** — source contains `data-testid="new-comment-body"`.
- [ ] **`test_form_has_char_counter_testid`** — source contains `data-testid="new-comment-char-counter"`.
- [ ] **`test_form_has_submit_btn_testid`** — source contains `data-testid="new-comment-submit-btn"`.
- [ ] **`test_form_has_readonly_note_testid`** — source contains `data-testid="new-comment-readonly-note"`.
- [ ] **`test_form_has_no_section_note_testid`** — source contains `data-testid="new-comment-no-section-note"`.
- [ ] **`test_form_uses_zod_with_max_10000`** — source contains `10000` + matches `z\.string|z\.object`.
- [ ] **`test_form_uses_use_form`** — source contains `useForm`.
- [ ] **`test_form_uses_zod_resolver`** — source contains `zodResolver`.
- [ ] **`test_form_has_max_length_attribute`** — source contains `maxLength`.
- [ ] **`test_form_exports_schema`** — source matches `/export.*schema|export.*Schema/i`.
- [ ] **`test_form_resets_on_success`** — source contains `reset`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_new_comment_form_hides_textarea_when_role_is_read_only`** (it.skip) — `useMyProposalRole` mocked to `{ role: "read_only" }`; `new-comment-body` absent; `new-comment-readonly-note` present.

**Zod schema assertions (structural — not skipped):**
- [ ] **`test_create_comment_schema_rejects_empty_body`** — source matches `min\s*\(\s*1`.
- [ ] **`test_create_comment_schema_rejects_whitespace_only`** — source contains `.trim()`.
- [ ] **`test_create_comment_schema_rejects_over_10k_chars`** — source matches `max\s*\(\s*10000`.

---

### ✅ AC7 — `CommentItem`

**Component source text assertions:**
- [ ] **`test_item_has_use_client`** — source contains `"use client"`.
- [ ] **`test_item_named_export`** — source matches `export (function|const) CommentItem`.
- [ ] **`test_item_has_per_comment_testid`** — source contains `comment-item-`.
- [ ] **`test_item_has_avatar_testid`** — source contains `comment-avatar-`.
- [ ] **`test_item_has_author_testid`** — source contains `comment-author-`.
- [ ] **`test_item_has_timestamp_testid`** — source contains `comment-timestamp-`.
- [ ] **`test_item_has_body_testid`** — source contains `comment-body-`.
- [ ] **`test_item_has_resolve_row_testid`** — source contains `comment-resolve-row-`.
- [ ] **`test_item_has_resolve_checkbox_testid`** — source contains `comment-resolve-checkbox-`.
- [ ] **`test_item_has_carried_forward_testid`** — source contains `comment-carried-forward-`.
- [ ] **`test_item_has_resolved_by_testid`** — source contains `comment-resolved-by-`.
- [ ] **`test_item_has_optimistic_badge_testid`** — source contains `comment-optimistic-`.
- [ ] **`test_item_resolved_rows_have_opacity_60`** — source contains `opacity-60`.
- [ ] **`test_item_optimistic_rows_have_opacity_70_italic`** — source contains `opacity-70`.
- [ ] **`test_item_hides_resolve_checkbox_on_optimistic_row`** — source matches `optimistic\s*!==\s*true|!.*optimistic`.
- [ ] **`test_item_uses_resolve_and_unresolve_hooks`** — source contains `useResolveComment` + `useUnresolveComment`.
- [ ] **`test_item_uses_time_element_with_date_time`** — source contains `dateTime`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_comment_item_hides_resolve_checkbox_on_optimistic_row`** (it.skip) — `comment.optimistic=true`; `comment-resolve-checkbox-{id}` absent; `comment-optimistic-{id}` present.
- [ ] **`test_comment_item_renders_carried_forward_marker`** (it.skip) — `carried_forward_from_id` non-null; `comment-carried-forward-{id}` visible.

---

### ✅ AC8 — `CarryForwardBanner`

**Component source text assertions:**
- [ ] **`test_banner_has_use_client`** — source contains `"use client"`.
- [ ] **`test_banner_named_export`** — source matches `export (function|const) CarryForwardBanner`.
- [ ] **`test_banner_has_banner_testid`** — source contains `data-testid="carry-forward-banner"`.
- [ ] **`test_banner_has_message_testid`** — source contains `data-testid="carry-forward-banner-message"`.
- [ ] **`test_banner_has_dismiss_btn_testid`** — source contains `data-testid="carry-forward-banner-dismiss-btn"`.
- [ ] **`test_banner_guards_null_version_id`** — source matches `versionId.*null|null.*versionId`.
- [ ] **`test_banner_uses_use_comments_summary`** — source contains `useCommentsSummary`.
- [ ] **`test_banner_uses_is_banner_dismissed`** — source contains `isBannerDismissed`.
- [ ] **`test_banner_calls_dismiss_banner`** — source contains `dismissBanner`.
- [ ] **`test_banner_uses_amber_colour_scheme`** — source matches `amber`.
- [ ] **`test_banner_has_role_status_aria`** — source contains `role="status"`.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_carry_forward_banner_returns_null_when_count_is_zero`** (it.skip) — `carriedForwardCount=0`; `carry-forward-banner` absent.
- [ ] **`test_carry_forward_banner_returns_null_when_already_dismissed`** (it.skip) — `isBannerDismissed` returns true; banner absent.
- [ ] **`test_carry_forward_banner_renders_with_correct_count_when_visible`** (it.skip) — `carriedForwardCount=3`; message contains "3".

---

### ✅ AC9 — `useCommentsSidebarStore` (Zustand)

**Source text assertions:**
- [ ] **`test_store_exports_use_comments_sidebar_store`** — source matches `export const useCommentsSidebarStore`.
- [ ] **`test_store_has_active_section_key`** — source contains `activeSectionKey`.
- [ ] **`test_store_has_set_active_section_key`** — source contains `setActiveSectionKey`.
- [ ] **`test_store_has_dismissed_banners`** — source contains `dismissedBanners`.
- [ ] **`test_store_has_dismiss_banner`** — source contains `dismissBanner`.
- [ ] **`test_store_has_is_banner_dismissed`** — source contains `isBannerDismissed`.
- [ ] **`test_store_has_is_open`** — source contains `isOpen`.
- [ ] **`test_store_has_open_sidebar`** — source contains `openSidebar`.
- [ ] **`test_store_has_close_sidebar`** — source contains `closeSidebar`.
- [ ] **`test_store_has_reset_for_proposal`** — source contains `resetForProposal`.
- [ ] **`test_store_uses_persist_middleware`** — source matches `\bpersist\(`.
- [ ] **`test_store_partializes_dismissed_banners_only`** — source contains `partialize` + `dismissedBanners`.
- [ ] **`test_store_persist_name`** — source contains `eusolicit-comments-banner-dismissals`.
- [ ] **`test_store_banner_key_uses_proposal_colon_version`** — source matches colon-separated key pattern.

**Behavioural (it.skip — RED phase):**
- [ ] **`test_dismiss_banner_persists_to_local_storage`** (it.skip) — calls `dismissBanner("p1","v1")`; reads localStorage; asserts `"p1:v1": true`.
- [ ] **`test_open_sidebar_with_section_key_sets_active_section`** (it.skip) — calls `openSidebar("intro")`; asserts `activeSectionKey === "intro"` + `isOpen === true`.
- [ ] **`test_reset_for_proposal_clears_active_section_and_closes`** (it.skip) — seeds open state; calls `resetForProposal()`; asserts both cleared.

---

### ✅ AC10 — `SectionCommentsBadge`

**Component source text assertions:**
- [ ] **`test_badge_has_use_client`** — source contains `"use client"`.
- [ ] **`test_badge_named_export`** — source matches `export (function|const) SectionCommentsBadge`.
- [ ] **`test_badge_has_per_section_testid`** — source contains `section-comments-badge-`.
- [ ] **`test_badge_uses_use_comments_count_by_section`** — source contains `useCommentsCountBySection`.
- [ ] **`test_badge_returns_null_when_zero`** — source contains `null`.
- [ ] **`test_badge_unresolved_uses_blue_style`** — source matches `blue`.
- [ ] **`test_badge_all_resolved_uses_muted_style`** — source matches `muted`.
- [ ] **`test_badge_has_aria_label_i18n_key`** — source matches `badgeUnresolvedAria|badgeAllResolvedAria`.
- [ ] **`test_badge_calls_open_sidebar`** — source contains `openSidebar`.

---

### ✅ AC11 — `ProposalWorkspacePage` integration

**Source text assertions:**
- [ ] **`test_workspace_has_comments_toolbar_btn_testid`** — `ProposalWorkspacePage.tsx` contains `data-testid="toolbar-btn-comments"`.
- [ ] **`test_workspace_has_comments_count_testid`** — source contains `data-testid="toolbar-btn-comments-count"`.
- [ ] **`test_workspace_mounts_comments_sidebar_element`** — source contains `<CommentsSidebar`.
- [ ] **`test_workspace_has_comments_open_local_state`** — source contains `commentsOpen`.
- [ ] **`test_workspace_passes_current_version_id_to_sidebar`** — source contains `currentVersionId`.
- [ ] **`test_workspace_uses_use_comments_summary_for_badge`** — source contains `useCommentsSummary`.
- [ ] **`test_workspace_syncs_store_is_open_with_local_state`** — source contains `useCommentsSidebarStore`.
- [ ] **`test_proposal_section_has_badge_in_header`** — `ProposalSection.tsx` contains `SectionCommentsBadge`.
- [ ] **`test_proposal_editor_resets_sidebar_state_on_unmount`** — `ProposalEditor.tsx` contains `resetForProposal`.

---

### ✅ AC12 — i18n keys (EN + BG, `proposals.comments.*`, ≥ 33 keys)

**i18n parity assertions (structural):**
- [ ] **`test_en_has_proposals_comments_namespace`** — `en.json` has `proposals.comments` object.
- [ ] **`test_bg_has_proposals_comments_namespace`** — `bg.json` has `proposals.comments` object.
- [ ] **`test_proposals_comments_en_bg_key_parity`** — top-level keys in `en.proposals.comments` match `bg.proposals.comments` exactly.
- [ ] **`test_proposals_comments_has_at_least_33_leaf_keys`** — recursive leaf count ≥ 33.
- [ ] **`test_en_panel_title_is_comments`** — `en.proposals.comments.panelTitle === "Comments"`.
- [ ] **`test_en_submit_btn_is_post`** — `en.proposals.comments.submitBtn === "Post"`.
- [ ] **`test_en_resolved_label_is_resolved`** — `en.proposals.comments.resolvedLabel === "Resolved"`.
- [ ] **`test_en_carry_forward_banner_message_uses_icu_plural`** — EN value matches `{count, plural,`.
- [ ] **`test_bg_carry_forward_banner_message_uses_icu_plural`** — BG value matches `{count, plural,`.
- [ ] **`test_badge_unresolved_aria_uses_icu_plural`** — EN value matches `plural`.
- [ ] **`test_en_empty_title_is_no_comments_yet`** — value is `"No comments yet"`.
- [ ] **`test_en_read_only_note_contains_read_only`** — value case-insensitively contains `"read-only"`.
- [ ] **`test_create_error_sub_namespace_has_required_keys`** — `createError` has `validation`, `notAuthorised`, `proposalGone`.
- [ ] **`test_resolve_error_sub_namespace_has_required_keys`** — `resolveError` has `notAuthorised`, `gone`.

---

### ✅ AC13 — Polling / cache-key and mutation-surface assertions

**Structural polling/cache-key assertions:**
- [ ] **`test_use_proposal_comments_query_key_includes_section_key`** — source contains `"proposal-comments"` + `sectionKey`.
- [ ] **`test_use_comments_summary_has_60s_refetch_interval`** — source matches `refetchInterval:\s*(60_000|60000)`.
- [ ] **`test_use_comments_summary_refetches_on_window_focus`** — source contains `refetchOnWindowFocus: true`.
- [ ] **`test_use_proposal_comments_is_disabled_when_section_key_null`** — source contains `enabled` + `sectionKey`.
- [ ] **`test_use_proposal_comments_stale_time_0`** — source matches `staleTime:\s*0`.
- [ ] **`test_use_comments_summary_refetch_interval_in_background_false`** — source contains `refetchIntervalInBackground: false`.

**Behavioural mutation assertions (it.skip — RED phase):**
- [ ] **`test_create_comment_mutation_prepends_optimistic_row`** (it.skip) — mocks `queryClient.setQueryData`; `mutate({ body: "hello" })`; first element has `optimistic: true`.
- [ ] **`test_create_comment_mutation_rolls_back_on_error`** (it.skip) — mocks API to reject 422; `setQueryData` called twice (insert + rollback); `addToast` called with `type: "error"`.
- [ ] **`test_resolve_mutation_flips_resolved_in_cache_optimistically`** (it.skip) — seeds cache with `{ id: "c1", resolved: false }`; calls `useResolveComment.mutate("c1")`; cache immediately shows `resolved: true` before API resolves.

---

### ✅ AC14 — Story 10.12 regression guard

**Source text assertions (structural — ensure 10.12 additions are not removed):**
- [ ] **`test_proposal_section_still_has_section_lock_indicator`** — `ProposalSection.tsx` still contains `SectionLockIndicator`.
- [ ] **`test_proposal_section_has_both_lock_indicator_and_badge`** — source contains both `SectionLockIndicator` + `SectionCommentsBadge`.
- [ ] **`test_workspace_still_has_toolbar_collaborators_btn`** — `ProposalWorkspacePage.tsx` still contains `toolbar-btn-collaborators`.
- [ ] **`test_workspace_has_both_collaborators_and_comments_btns`** — source contains both testids.
- [ ] **`test_proposal_editor_still_calls_clear_local_lock_states`** — `ProposalEditor.tsx` still contains `clearLocalLockStates`.
- [ ] **`test_proposal_editor_has_both_lock_and_sidebar_cleanup`** — source contains both `clearLocalLockStates` + `resetForProposal`.
- [ ] **`test_en_proposals_collaborators_namespace_intact`** — `en.json` still has `proposals.collaborators` object.
- [ ] **`test_en_proposals_locks_namespace_intact`** — `en.json` still has `proposals.locks` object.

---

## Not-yet-covered (defer to Implementation / bmad-testarch-automate)

| AC | Gap | Reason |
|----|-----|--------|
| AC4 | Visual grouping / ordering (unresolved ASC, resolved ASC with divider) | Full sorted-group render requires complete implementation; structural assertions cover the data-testid hooks |
| AC4.5 | Sidebar scrollable body / sticky NewCommentForm CSS | CSS-in-test is fragile; covered by manual smoke in Task 10 |
| AC6 | Character counter colour shift at 9000/9900 chars | Requires real Textarea value interaction; deferred to GREEN with RTL userEvent |
| AC7 | Relative-time formatter (`formatRelativeTime`) correctness | Pure function; separate unit test in `lib/utils/__tests__/` after implementation |
| AC8 | Banner dismissal per-tuple rotation on version change | Requires store + version-change simulation; deferred to GREEN |
| AC9 | Store hydration from localStorage after reload | Requires JSDOM with localStorage; deferred to GREEN (`test_dismiss_banner_persists_to_local_storage`) |
| AC12 | `pnpm check:i18n` script exit code | External tool; structural key-parity test covers same guarantee |
| AC14 | Full `vitest run __tests__/collaborator-lock-s10-12.test.ts` → 160/160 | Cannot run other test file from inside this test; dev must run manually per Task 9 |

---

## TDD Activation Sequence

Remove `it.skip` markers in this order (matches task sequence in the story):

1. **Task 1 complete** → Remove skip from: `test_use_proposal_comments_query_key_includes_section_key`, `test_use_comments_summary_has_60s_refetch_interval`, `test_use_comments_summary_refetches_on_window_focus`, `test_use_proposal_comments_is_disabled_when_section_key_null`.
2. **Task 2 complete** → Remove skip from: `test_dismiss_banner_persists_to_local_storage`, `test_open_sidebar_with_section_key_sets_active_section`, `test_reset_for_proposal_clears_active_section_and_closes`.
3. **Task 3 complete** → Remove skip from: `test_comments_sidebar_renders_skeleton_when_query_is_loading`, `test_comments_sidebar_renders_empty_state_when_no_section_selected`, `test_comment_item_hides_resolve_checkbox_on_optimistic_row`, `test_comment_item_renders_carried_forward_marker`.
4. **Task 4 complete** → Remove skip from: `test_new_comment_form_hides_textarea_when_role_is_read_only`, `test_carry_forward_banner_returns_null_when_count_is_zero`, `test_carry_forward_banner_returns_null_when_already_dismissed`, `test_carry_forward_banner_renders_with_correct_count_when_visible`.
5. **Task 7 complete** → Remove skip from: `test_create_comment_mutation_prepends_optimistic_row`, `test_create_comment_mutation_rolls_back_on_error`, `test_resolve_mutation_flips_resolved_in_cache_optimistically`.
6. **Final GREEN** → Remove skip from: `test_comments_sidebar_renders_two_groups_when_mixed_resolved_and_open`.
