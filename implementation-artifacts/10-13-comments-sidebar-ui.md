# Story 10.13: Comments Sidebar UI

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-13-comments-sidebar-ui
- **Points:** 3
- **Type:** frontend
- **Module:** Client web app (`apps/client`) — new components: `CommentsSidebar`, `CommentsSidebarHeader`, `CommentList`, `CommentItem`, `NewCommentForm`, `CarryForwardBanner`, `SectionCommentsBadge`; extended components: `ProposalWorkspacePage` (toolbar toggle + slide-over mount + URL-sync of `commentsOpen`), `ProposalEditor` (section-click listener surfacing the active `section_key` to the sidebar), `ProposalSection` (renders `SectionCommentsBadge` in the header next to `SectionLockIndicator`); new API client: `apps/client/lib/api/proposal-comments.ts`; new queries: `apps/client/lib/queries/use-proposal-comments.ts`; new store: `apps/client/lib/stores/comments-sidebar-store.ts` (active section + sidebar open/closed + per-comment optimistic overlay); i18n: `apps/client/messages/{en,bg}.json` (new `proposals.comments.*` namespace); ATDD test: `apps/client/__tests__/comments-sidebar-s10-13.test.ts`.
- **Priority:** P1 (Epic 10 AC3 user-visible completion — the Story 10.4 Comments API has been in `done` since Sprint 11; this story delivers the first and only UI surface for it. Without S10.13 the team cannot exercise the comment lifecycle end-to-end, and carry-forward semantics remain invisible to the user. Ships alongside the S10.12 collaborator panel on the same proposal workspace, closing the first half of Epic 10's UI track.)
- **Depends On:** Story 10.4 (Proposal Comments API — `POST/GET/PATCH /api/v1/proposals/{proposal_id}/comments*` returning the `CommentResponse`/`CommentListResponse` shapes: `id`, `proposal_id`, `version_id`, `section_key`, `author_id`, `author_full_name`, `body`, `parent_comment_id`, `resolved`, `resolved_by`, `resolved_by_full_name`, `resolved_at`, `carried_forward_from_id`, `created_at`, `updated_at`; the `unresolved_count` aggregate is consumed by the per-section badge), Story 10.1/10.2 (`require_proposal_role` gate and `ProposalCollaboratorRole` enum — drive the "hide resolve checkbox for read_only" UI rule), Story 10.12 (`useMyProposalRole(proposalId)` hook is reused verbatim — single source of truth for "can I mutate this proposal?"; also reuses the `Sheet`/slide-over primitive + `proposalId` toolbar plumbing), Story 7.11 / 7.12 (`ProposalWorkspacePage`, `ProposalEditor`, `ProposalSection`, `useProposalEditorStore` for `activeSectionKey` — same integration points as S10.12), Story 7.3 (the `POST /api/v1/proposals/{id}/versions` endpoint — the carry-forward banner is surfaced when `current_version_id` changes and the server-side carry-forward hook populates new clones with `carried_forward_from_id IS NOT NULL`), Story 3.5 (Zustand + TanStack Query), Story 3.6 (React Hook Form + Zod — `NewCommentForm`), Story 3.7 (next-intl EN/BG parity), Story 3.10 (loading / empty / error primitives), Story 3.11 (toast store — `useUIStore.addToast` for mutation errors).

## Story

As **a proposal collaborator with write permission (bid_manager, technical_writer, financial_analyst, or legal_reviewer) reviewing a teammate's draft**,
I want **a collapsible sidebar on the proposal workspace that automatically filters to the comments for whichever section I have focused in the Tiptap editor, lets me post a new comment on that section, toggle resolved/unresolved on any comment with a checkbox, see resolved comments dimmed and grouped at the bottom, see an unresolved-count badge on each section header, and see a banner when the team has created a new version so I know which comments were carried forward from the previous one**,
so that **feedback stays anchored to the exact (section, version) tuple my teammates are looking at, I can converge on what is open vs. closed without leaving the editor, and I have clear signals when a new version snapshot changes the baseline of the conversation (thereby closing Epic 10 AC3 end-to-end and giving the §10.12 lock model a partner UX surface so a reviewer who cannot acquire the lock can still leave review notes).**

## Description

Story 10.13 delivers the second user-visible surface of Epic 10's collaboration model (after the §10.12 collaborator panel and lock indicators). The backend Comments API (Story 10.4) has been in `done` since Sprint 11 — this story consumes the existing `POST/GET/PATCH /api/v1/proposals/{proposal_id}/comments*` endpoints from the proposal workspace without backend changes.

It ships **three tightly-coupled UI surfaces plus a thin shared store**:

1. **Comments Sidebar (right-edge slide-over).** Opened from a new toolbar button on `ProposalWorkspacePage` (between the §10.12 "Collaborators" button and the version indicator). Width: `w-96` on desktop, `w-full` on mobile. Reuses the same `Sheet` primitive the `VersionHistorySidebar` (Story 7.16) and `CollaboratorPanel` (Story 10.12) already use — one coherent right-rail idiom across the workspace. The sidebar is **section-aware**: it subscribes to `useProposalEditorStore.activeSectionKey` (already populated by the existing `ProposalSection.onFocus` callback from Story 7.12) and filters its displayed comments to that section + the current version via `GET /api/v1/proposals/{id}/comments?version_id={currentVersion}&section_key={activeSection}`. When no section is focused, the sidebar shows an empty-state prompt asking the user to click into a section. The sidebar is **independent of** the lock state — a reviewer with a `foreign` lock on the section can still post comments (reviewing is the point; blocking writing is what the lock does).

2. **Per-Section Unresolved-Count Badge.** Each `ProposalSection` header gets a `SectionCommentsBadge` (rendered next to the existing `SectionLockIndicator`) showing the `unresolved_count` for that section. Driven by a single `useQuery` keyed on `["proposal-comments-summary", proposalId, currentVersionId]` that issues a **single** `GET /comments?version_id=...` (no `section_key` filter), then groups the result client-side by `section_key` and exposes a `useCommentsCountBySection(sectionKey)` selector. This avoids N queries for N sections. The badge links: clicking it opens the sidebar AND focuses that section (so the sidebar auto-filters).

3. **Carry-Forward Banner.** When the user creates a new version via the existing version-history flow (Story 7.16), OR when the polled current-version-id changes because a teammate created one, the sidebar displays a banner at the top: "3 unresolved comments were carried forward from version 2." The banner is derived from the new version's comments: any row with `carried_forward_from_id !== null` on the current version → counted. The banner has a dismiss button; dismissal persists in `useCommentsSidebarStore` per `(proposalId, version_id)` so it does not reappear on re-open until the next version creation.

Plus the forms:

4. **New Comment Form.** A `<textarea>` + submit button at the top of the sidebar (below the header, above the comment list). Disabled + hidden for `read_only` proposal collaborators; disabled (with a tooltip) when no section is focused. Submitting POSTs `{ version_id, section_key, body }` with optimistic UI insertion — the new row appears immediately with `id` = a temporary UUID, then reconciles with the server response; on error the optimistic row is removed and the failure is surfaced via toast. Max length 10000 chars (matches the backend DB check). Empty/whitespace-only bodies are locally rejected (do not hit the API).

5. **Resolve Checkbox per Comment.** Each `CommentItem` has a `<Checkbox>` labelled "Resolved" that is present and interactable for PROPOSAL_WRITE_ROLES + admin, and hidden for `read_only`. Toggling it calls the resolve/unresolve endpoint with optimistic UI (flip the `resolved` state immediately, roll back on error). Because the backend endpoints are idempotent (Story 10.4 Design Decision c), repeated clicks during round-trips cannot corrupt state.

**Six non-obvious design decisions drive the implementation:**

**(a) Filter mode is read-time, not write-time.** The sidebar always issues `GET /comments?version_id={current}&section_key={active}` and re-fires on `activeSectionKey` changes (via the existing `useProposalEditorStore` subscription). We do NOT keep a client-side cache of "all comments for all sections" and filter locally, for three reasons: (i) the backend AC5 single-query endpoint is already optimised for this shape (composite index `ix_proposal_comments_version_section`), (ii) the badge query (`useCommentsCountBySection`) is separately responsible for the unsegmented read, (iii) keeping the sidebar query section-scoped means it refetches on section change and picks up comments posted by teammates on the same section since the last render. The two queries (`["proposal-comments", proposalId, versionId, sectionKey]` for the list and `["proposal-comments-summary", proposalId, versionId]` for the badges) have independent `staleTime` (30 s for the summary, 0 for the active-section list so focus changes always re-fetch).

**(b) Optimistic insertion uses a temp-UUID prefix that is never sent to the server.** When the user posts a new comment, we build a client-side `CommentResponse`-shaped row with `id = \`temp-${crypto.randomUUID()}\`` and `optimistic = true` (a frontend-only flag added via a `CommentWithOptimisticState` type). The row is prepended to the TanStack-Query cache via `queryClient.setQueryData(...)`. On mutation success, the real server response replaces the temp row by calling `queryClient.invalidateQueries(["proposal-comments", ...])`. On error, we `queryClient.setQueryData(...)` again to remove the temp row. This is safer than `optimisticUpdate` (TanStack Query v5 native) because it gives us full control over failure rollback + badge-count recalculation. The badge summary query is invalidated on both success and failure so the count stays truthful.

**(c) Resolve/unresolve optimism is purely in-place (no row movement) until the server confirms.** When the user checks the "Resolved" box on an open comment, we immediately mark the row `resolved: true` in the TanStack-Query cache (still in the unresolved group — NOT moved to the resolved group) and fire the PATCH. Only after the server's 200 response do we invalidate the summary query, which triggers a re-render that moves the row to the resolved group (dimmed, bottom of the list). Rationale: the resolve click must not make the row disappear out from under the user's cursor mid-click, which would feel jarring; an additional 200 ms to re-group after the server confirm is acceptable. On error, we flip `resolved` back and surface a toast. The idempotency guarantee (Story 10.4 AC6/AC7) means a repeated click during the round-trip cannot break state.

**(d) Polling is opt-in, not default. The sidebar refetches on `activeSectionKey` change and on window focus — not on a timer.** Section focus already fires a refetch via the query-key change. `refetchOnWindowFocus: true` covers the "teammate posted while you were in another tab" case. The summary badge query adds `refetchInterval: 60_000` (NOT 30_000 — locks are 30 s because contention is high-signal; comments are lower-stakes and 60 s keeps polling traffic sane) with `refetchIntervalInBackground: false`. Neither query uses SSE or WebSocket — that's a post-MVP upgrade path (see §Out of Scope). If comment traffic becomes noticeable during QA, the intervals can be tuned without structural change.

**(e) The carry-forward banner is derived from server data, not from a client-side event.** We do NOT listen for a "version-created" client-side signal. Instead, we compute the banner from the current query response: if `data.comments.some(c => c.carried_forward_from_id !== null)` AND the banner for `(proposalId, currentVersionId)` has not been dismissed, show it. The count is `data.comments.filter(c => c.carried_forward_from_id !== null && c.resolved === false).length`. This is correct in all cases: page reload, new-tab open, teammate-creates-version-while-you-are-viewing (the next 60 s refetch surfaces it). Dismissal is per-tuple so the banner appears exactly once per new version per user.

**(f) `read_only` gating hides but does not remove the new-comment form or resolve checkbox — it leaves the visible skeleton intact with an inline note.** When `useMyProposalRole(proposalId).role === "read_only"`, we render the `NewCommentForm` wrapper but replace the textarea with a `readOnlyNote` ("Read-only collaborators cannot post comments."). Similarly, `CommentItem` renders the resolve checkbox as disabled + tooltipped for read_only users. Rationale: suddenly-missing affordances confuse users; explicit-but-disabled affordances communicate the role boundary. This mirrors Story 10.12's collaborator-panel `readOnlyNote` pattern. Server-side, the 403 remains the source of truth (defence-in-depth).

A seventh subtle point worth calling out: **the sidebar and the per-section badge share no cache key, by design.** The sidebar query is keyed on `["proposal-comments", proposalId, versionId, sectionKey]` with a specific `section_key` filter; the summary query is keyed on `["proposal-comments-summary", proposalId, versionId]` with NO `section_key`. Mutations invalidate BOTH keys — a new comment on section "intro" invalidates the summary (so all section badges refresh) and the active-section list (if the user is viewing "intro"). TanStack Query handles the fan-out idempotently; no manual refetch is required. This avoids the N+1 pattern (N queries for N section badges) while keeping the active-section list lazily-scoped.

The sidebar and the badge are independent on the page (the toolbar toggles the sidebar; the badges always render when the summary query has loaded). Either surface can be deployed independently behind a feature flag if rollback is needed. The new-comment form and resolve checkbox are subordinate to the sidebar (they only render inside it) and do not need their own flag.

## Acceptance Criteria

### File-system & Routing

1. [ ] **AC1 — Seven new component files plus one API + one query file + one store file exist under `apps/client/`.**

    Components:
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/CommentsSidebar.tsx` (`"use client"`; default export `CommentsSidebar`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/CommentsSidebarHeader.tsx` (`"use client"`; named export `CommentsSidebarHeader`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/CommentList.tsx` (`"use client"`; named export `CommentList`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/CommentItem.tsx` (`"use client"`; named export `CommentItem`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/NewCommentForm.tsx` (`"use client"`; named export `NewCommentForm`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/CarryForwardBanner.tsx` (`"use client"`; named export `CarryForwardBanner`).
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/SectionCommentsBadge.tsx` (`"use client"`; named export `SectionCommentsBadge`).

    API / queries / store:
    - `apps/client/lib/api/proposal-comments.ts` (`listComments`, `createComment`, `resolveComment`, `unresolveComment`, `listReplies`, plus the six interfaces enumerated in AC2).
    - `apps/client/lib/queries/use-proposal-comments.ts` (exports `useProposalComments`, `useCommentsSummary`, `useCommentsCountBySection`, `useCreateComment`, `useResolveComment`, `useUnresolveComment`).
    - `apps/client/lib/stores/comments-sidebar-store.ts` (exports `useCommentsSidebarStore`; see AC9).

    Modified files:
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalWorkspacePage.tsx` — adds the toolbar "Comments" button + state for `commentsOpen` + mounts `<CommentsSidebar proposalId={proposal.id} currentVersionId={proposal.current_version_id} isOpen={commentsOpen} onClose={() => setCommentsOpen(false)} />` alongside `VersionHistorySidebar`, `CollaboratorPanel`, and `ExportDialog`.
    - `apps/client/app/[locale]/(protected)/proposals/[id]/components/ProposalSection.tsx` — adds `<SectionCommentsBadge proposalId={proposalId} versionId={versionId} sectionKey={section.key} onClick={() => openSidebarOnSection(section.key)} />` inside the existing `<h3 data-testid="section-header-{key}">` block, immediately after `<SectionLockIndicator/>`.

2. [ ] **AC2 — `apps/client/lib/api/proposal-comments.ts` exports.**

    ```ts
    export interface CommentResponse {
      id: string;
      proposal_id: string;
      version_id: string;
      section_key: string;
      author_id: string;
      author_full_name: string | null;
      body: string;
      parent_comment_id: string | null;
      resolved: boolean;
      resolved_by: string | null;
      resolved_by_full_name: string | null;
      resolved_at: string | null;     // ISO-8601
      carried_forward_from_id: string | null;
      created_at: string;             // ISO-8601
      updated_at: string;             // ISO-8601
    }

    /** UI-only extension: optimistic rows carry this flag so the
     *  renderer can dim them and skip rendering the resolve checkbox. */
    export interface CommentWithOptimisticState extends CommentResponse {
      optimistic?: boolean;
    }

    export interface CommentListResponse {
      proposal_id: string;
      version_id: string;
      section_key: string | null;
      comments: CommentResponse[];
      total: number;
      unresolved_count: number;
    }

    export interface CreateCommentRequest {
      version_id: string;
      section_key: string;
      body: string;
      parent_comment_id?: string | null;
    }

    export interface ListCommentsParams {
      version_id: string;
      section_key?: string;
      resolved?: boolean;
    }

    export type CommentsBySectionMap = Record<string, { total: number; unresolved: number }>;
    ```

    Functions (all use the shared `apiClient` pattern from `@eusolicit/ui` / `lib/api/_client.ts`):
    - `listComments(proposalId: string, params: ListCommentsParams): Promise<CommentListResponse>` — `GET /api/v1/proposals/{proposalId}/comments?version_id=...&section_key?=...&resolved?=...`. Throws `AxiosError` on non-2xx.
    - `createComment(proposalId: string, payload: CreateCommentRequest): Promise<CommentResponse>` — `POST /api/v1/proposals/{proposalId}/comments` with `Content-Type: application/json`. Expects `201`.
    - `resolveComment(proposalId: string, commentId: string): Promise<CommentResponse>` — `PATCH /api/v1/proposals/{proposalId}/comments/{commentId}/resolve` with NO body.
    - `unresolveComment(proposalId: string, commentId: string): Promise<CommentResponse>` — `PATCH /api/v1/proposals/{proposalId}/comments/{commentId}/unresolve` with NO body.
    - `listReplies(proposalId: string, commentId: string): Promise<CommentResponse[]>` — `GET /api/v1/proposals/{proposalId}/comments/{commentId}/replies`. Used by an (out-of-scope in this story but pre-wired) thread-expand affordance; exported for forward-compatibility.

    All functions throw the raw `AxiosError` on non-2xx; callers map `response.status` to user-facing error messages via the project's standard `error-utils.ts` helpers. The `listComments` status codes we care about: `200` (happy), `400` (bad `version_id`), `403` (non-collaborator non-admin), `404` (cross-company or missing proposal). The `createComment` status codes: `201` (happy), `400` (bad parent/version), `403` (read_only or non-collaborator), `404` (cross-company), `422` (validation — empty/oversized body). Resolve/unresolve: `200` (happy, including idempotent no-op), `403` (read_only), `404` (missing comment or tenant mismatch).

3. [ ] **AC3 — `apps/client/lib/queries/use-proposal-comments.ts` exports.**

    ```ts
    export function useProposalComments(
      proposalId: string,
      versionId: string | null,
      sectionKey: string | null,
    )
        // useQuery({
        //   queryKey: ["proposal-comments", proposalId, versionId, sectionKey],
        //   queryFn: () => listComments(proposalId, { version_id: versionId!, section_key: sectionKey ?? undefined }),
        //   enabled: Boolean(proposalId && versionId && sectionKey),
        //   staleTime: 0,
        //   refetchOnWindowFocus: true,
        // })

    export function useCommentsSummary(proposalId: string, versionId: string | null)
        // useQuery({
        //   queryKey: ["proposal-comments-summary", proposalId, versionId],
        //   queryFn: () => listComments(proposalId, { version_id: versionId! }),
        //   enabled: Boolean(proposalId && versionId),
        //   staleTime: 30_000,
        //   refetchInterval: 60_000,
        //   refetchIntervalInBackground: false,
        //   refetchOnWindowFocus: true,
        //   select: (data: CommentListResponse): { bySection: CommentsBySectionMap; carriedForwardCount: number } => ({
        //     bySection: buildCommentsBySectionMap(data.comments),
        //     carriedForwardCount: data.comments.filter(c => c.carried_forward_from_id !== null && !c.resolved).length,
        //   }),
        // })

    export function useCommentsCountBySection(proposalId: string, versionId: string | null, sectionKey: string):
        { total: number; unresolved: number; isLoading: boolean }
        // Thin selector over useCommentsSummary; returns zero-map defaults when query is pending.

    export function useCreateComment(proposalId: string, versionId: string, sectionKey: string)
        // useMutation; optimistic:
        //   onMutate: prepend a temp-UUID CommentWithOptimisticState row to
        //     queryClient.getQueryData(["proposal-comments", proposalId, versionId, sectionKey]).
        //   onSuccess: invalidate ["proposal-comments", ...] AND ["proposal-comments-summary", ...].
        //   onError: roll back the optimistic insert; toast.
        // 422 surfaces t("comments.createError.validation");
        // 403 surfaces t("comments.createError.notAuthorised");
        // 404 surfaces t("comments.createError.proposalGone");
        // 5xx surfaces t("forms.serverError") + console.error.

    export function useResolveComment(proposalId: string, versionId: string, sectionKey: string)
        // useMutation; optimistic resolved=true in cache; onSettled invalidates both keys.
        // 403 → t("comments.resolveError.notAuthorised"); 404 → t("comments.resolveError.gone");
        // 5xx → t("forms.serverError").

    export function useUnresolveComment(proposalId: string, versionId: string, sectionKey: string)
        // Mirror of useResolveComment with the inverse optimistic flip.
    ```

    Helper: `buildCommentsBySectionMap(comments: CommentResponse[]): CommentsBySectionMap` — pure function returning `{ [section_key]: { total: number, unresolved: number } }`. Unit-tested.

    Each mutation uses `useUIStore.addToast` (sourced from `@/lib/stores/ui-store` per the Story 10.12 F1 fix) for error UX. On any 5xx the description is `t("forms.serverError")` and the error is logged to `console.error` with `proposalId`, `versionId`, `sectionKey`, `commentId` (if applicable) context.

4. [ ] **AC4 — `CommentsSidebar` (slide-over root).**

    Props: `{ proposalId: string; currentVersionId: string | null; isOpen: boolean; onClose: () => void }`.

    Renders a right-edge `<Sheet side="right" open={isOpen} onOpenChange={(open) => !open && onClose()}>` — the same primitive the `VersionHistorySidebar` (Story 7.16) and `CollaboratorPanel` (Story 10.12) use. Width: `w-96` on desktop, `w-full` on mobile. Root `data-testid="comments-sidebar"`.

    Behaviour:

    4.1 — On mount, subscribe to `useProposalEditorStore(s => s.activeSectionKey)` (already populated by `ProposalSection.onFocus` from Story 7.12). Store it in `useCommentsSidebarStore.activeSectionKey` so the filter persists across sidebar close/open within the same workspace mount.

    4.2 — Render in order:
    - `<CommentsSidebarHeader proposalId={proposalId} activeSectionKey={activeSectionKey} onClose={onClose} />` — shows title, the active section title (or a "Select a section" placeholder), and a close button.
    - `<CarryForwardBanner proposalId={proposalId} versionId={currentVersionId} />` — self-manages dismiss via the store.
    - `<NewCommentForm proposalId={proposalId} versionId={currentVersionId} sectionKey={activeSectionKey} />` — hidden (returns `null`) when `currentVersionId === null` OR `activeSectionKey === null`; renders a disabled skeleton in the latter case with a "Focus a section to add a comment" empty state (per §Description (f) pattern).
    - `<CommentList proposalId={proposalId} versionId={currentVersionId} sectionKey={activeSectionKey} />` — three states below.

    4.3 — `CommentList` states (driven by `useProposalComments(proposalId, currentVersionId, activeSectionKey)`):
    - **Disabled** — when `currentVersionId === null` OR `activeSectionKey === null`: render `<EmptyState data-testid="comments-sidebar-empty-noselection" icon={MousePointerClick} title={t("comments.noSectionSelectedTitle")} description={t("comments.noSectionSelectedDescription")} />`.
    - **Loading** — `<SkeletonCard data-testid="comments-sidebar-skeleton" rows={4} />`.
    - **Error** — `<EmptyState data-testid="comments-sidebar-error" icon={AlertCircle} title={t("comments.errorTitle")} description={t("comments.errorDescription")} />`.
    - **Success, empty** — `data.comments.length === 0` → `<EmptyState data-testid="comments-sidebar-empty-nocomments" icon={MessageSquarePlus} title={t("comments.emptyTitle")} description={t("comments.emptyDescription")} />`.
    - **Success, populated** — render two groups: unresolved first (ordered by `created_at` ASC — matches the backend AC5 ordering), then a visual divider + group heading `t("comments.resolvedGroupTitle", { count })`, then the resolved rows (also ordered `created_at` ASC, dimmed via `opacity-60`). Each row is a `<CommentItem key={c.id} comment={c} canMutate={canMutate} proposalId={proposalId} versionId={currentVersionId} sectionKey={activeSectionKey} />`.

    4.4 — `canMutate = useMyProposalRole(proposalId).role !== "read_only" && useMyProposalRole(proposalId).role !== null` — derived in one place, passed down to `NewCommentForm` and `CommentItem`. Admin bypass is handled inside `useMyProposalRole` (Story 10.12).

    4.5 — Scrollable body: the sidebar's inner wrapper is `flex flex-col overflow-y-auto` with `max-height: calc(100vh - header - footer)` so long comment lists scroll independently from the editor. The `NewCommentForm` is sticky at the top (inside the scrollable area) per Epic 10 S10.13 copy: "a 'New comment' text area at the top posts via the comments API."

5. [ ] **AC5 — `CommentsSidebarHeader`.**

    Props: `{ proposalId: string; activeSectionKey: string | null; onClose: () => void }`.

    Renders:
    - `<h2 data-testid="comments-sidebar-title">` with `t("comments.panelTitle")`.
    - `<p data-testid="comments-sidebar-active-section">` with `t("comments.activeSectionLabel", { section })` where `section` is the section title resolved from `useProposalEditorStore(s => s.content?.sections.find(s => s.key === activeSectionKey)?.title ?? t("comments.unknownSection"))` — falls back to `t("comments.noSectionSelectedShort")` when `activeSectionKey === null`.
    - `<button data-testid="comments-sidebar-close-btn">` with `<X />` icon and `aria-label={t("comments.closeBtn")}`.

    The header is a small `<div>` with `border-b` separating it from the scrollable body.

6. [ ] **AC6 — `NewCommentForm`.**

    Props: `{ proposalId: string; versionId: string | null; sectionKey: string | null }`.

    Renders a `<form data-testid="new-comment-form" onSubmit={handleSubmit}>`. State managed via `useForm` + `zodResolver` (see Story 3.6 pattern) with schema:
    ```ts
    const schema = z.object({
      body: z.string()
        .trim()
        .min(1, { message: "Comment cannot be empty" })
        .max(10000, { message: "Comment is too long (10 000 character limit)" }),
    });
    ```

    Fields:
    - **Textarea** — `<Textarea data-testid="new-comment-body" rows={3} placeholder={t("comments.newCommentPlaceholder")} maxLength={10000} />`.
    - **Character counter** — `<span data-testid="new-comment-char-counter" className="text-xs text-muted-foreground">` showing `{value.length} / 10000` (only visible when the user has typed something).
    - **Submit button** — `<Button type="submit" data-testid="new-comment-submit-btn" disabled={mutation.isPending || !form.formState.isValid || !sectionKey || !versionId}>` with label `t("comments.submitBtn")`. While in-flight: `<Loader2 className="animate-spin h-4 w-4 mr-2" />` + `t("comments.submittingBtn")`.

    Behaviour:
    - When `canMutate === false` (role is `read_only` or null): render the form wrapper with the textarea REPLACED by `<p data-testid="new-comment-readonly-note">{t("comments.readOnlyNote")}</p>` and NO submit button. This keeps the vertical layout stable for screen-readers and sighted users alike.
    - When `sectionKey === null`: render the textarea + button but keep the button disabled; show a `<p data-testid="new-comment-no-section-note" className="text-xs text-muted-foreground">{t("comments.focusSectionNote")}</p>` below the button.
    - On submit success: form is reset (`form.reset({ body: "" })`); optimistic row was already inserted in `onMutate`; success toast `addToast({ type: "success", title: t("comments.createSuccessTitle") })` fires (the description is omitted — the new row appearing IS the confirmation).
    - On submit error: optimistic row rolled back in `onError`; the form is NOT reset (the user can retry); error toast per AC3's status-code-to-message mapping.

7. [ ] **AC7 — `CommentItem`.**

    Props: `{ comment: CommentWithOptimisticState; canMutate: boolean; proposalId: string; versionId: string; sectionKey: string }`.

    Renders a `<li data-testid={\`comment-item-${comment.id}\`} className={cn("flex gap-2 p-3 border-b last:border-b-0", comment.resolved && "opacity-60", comment.optimistic && "opacity-70 italic")}>`:

    - **Avatar** — `<Avatar data-testid={\`comment-avatar-${comment.id}\`}>` showing initials computed from `author_full_name ?? "?"` (first letter of first two whitespace-separated tokens; empty names fall back to "?"). Matches the Story 10.12 `CollaboratorRow` avatar pattern.
    - **Body column** (`<div className="flex-1">`):
      - **Header row** — `<div className="flex items-center justify-between">`:
        - `<span data-testid={\`comment-author-${comment.id}\`} className="text-sm font-medium">{author_full_name ?? t("comments.unknownAuthor")}</span>`.
        - `<time data-testid={\`comment-timestamp-${comment.id}\`} dateTime={created_at} className="text-xs text-muted-foreground" title={new Date(created_at).toLocaleString(locale)}>{formatRelativeTime(created_at, locale)}</time>` — relative-time formatter (e.g. "2 h ago"); tooltip shows the full ISO-8601 timestamp. Use the project's existing `formatRelativeTime` helper if one exists; otherwise implement with `Intl.RelativeTimeFormat` inline.
      - **Body** — `<p data-testid={\`comment-body-${comment.id}\`} className="text-sm whitespace-pre-wrap break-words">{body}</p>`. `whitespace-pre-wrap` preserves multi-line comments; `break-words` handles long-unbroken-tokens.
      - **Carry-forward marker** — when `carried_forward_from_id !== null`, render `<span data-testid={\`comment-carried-forward-${comment.id}\`} className="text-xs text-muted-foreground"><ArrowDownCircle className="h-3 w-3 inline mr-0.5" />{t("comments.carriedForwardFromPriorVersion")}</span>` below the body.
      - **Resolve checkbox row** — `<div data-testid={\`comment-resolve-row-${comment.id}\`} className="mt-1 flex items-center gap-1">`:
        - If `canMutate === true` AND `comment.optimistic !== true`: `<Checkbox data-testid={\`comment-resolve-checkbox-${comment.id}\`} checked={comment.resolved} disabled={mutation.isPending} onCheckedChange={(checked) => checked ? resolveMutation.mutate(comment.id) : unresolveMutation.mutate(comment.id)} aria-label={t("comments.resolveAria", { author: author_full_name })} />` + `<label className="text-xs cursor-pointer">{t("comments.resolvedLabel")}</label>`.
        - If `canMutate === false`: render the same structure with `disabled={true}` and a `<Tooltip>` showing `t("comments.resolveReadOnly")`.
        - If `comment.resolved === true` AND `resolved_by_full_name` is non-null: add a `<span data-testid={\`comment-resolved-by-${comment.id}\`} className="text-xs text-muted-foreground ml-2">{t("comments.resolvedByLabel", { name: resolved_by_full_name, when: formatRelativeTime(resolved_at!, locale) })}</span>`.
      - **Optimistic row badge** — when `comment.optimistic === true`, render a small `<Badge data-testid={\`comment-optimistic-${comment.id}\`} variant="outline" className="text-xs">{t("comments.posting")}</Badge>` in place of the timestamp. Comment is dimmed (via the class above) and cannot be resolved (checkbox hidden per the `comment.optimistic !== true` check).

    The checkbox uses the existing `@eusolicit/ui` `Checkbox` primitive (shadcn-based, Story 3.2). The mutations come from `useResolveComment` and `useUnresolveComment` (AC3). Both are called with `comment.id` as their only argument — the hook closes over `proposalId`, `versionId`, `sectionKey` at construction time.

8. [ ] **AC8 — `CarryForwardBanner`.**

    Props: `{ proposalId: string; versionId: string | null }`.

    Renders `null` when any of:
    - `versionId === null` (no version yet).
    - `useCommentsSummary(proposalId, versionId).data?.carriedForwardCount === 0` (nothing was carried forward).
    - `useCommentsSidebarStore.isBannerDismissed(proposalId, versionId) === true` (user dismissed it for this version).

    Otherwise renders a `<div data-testid="carry-forward-banner" role="status" className="bg-amber-50 border-l-4 border-amber-400 px-3 py-2 flex items-start gap-2">`:
    - `<ArrowDownCircle className="h-4 w-4 text-amber-700 mt-0.5" />`.
    - `<p data-testid="carry-forward-banner-message" className="text-sm text-amber-900 flex-1">{t("comments.carryForwardBannerMessage", { count })}</p>` — where `count = data.carriedForwardCount`. The plural form is handled by the i18n `plural` syntax (see AC10).
    - `<button data-testid="carry-forward-banner-dismiss-btn" onClick={() => dismissBanner(proposalId, versionId)} aria-label={t("comments.carryForwardDismissAria")}>` with a `<X className="h-3 w-3" />`.

    The banner lives between `CommentsSidebarHeader` and `NewCommentForm` in the sidebar. Dismissal is stored in `useCommentsSidebarStore.dismissedBanners` (a `Record<string, boolean>` keyed as `${proposalId}:${versionId}`). This `Record` is persisted via Zustand's `persist` middleware under key `eusolicit-comments-banner-dismissals` so a reload does not resurface a dismissed banner for the same version. On version change the stored key naturally rotates (different `versionId` → fresh banner).

9. [ ] **AC9 — `useCommentsSidebarStore` (Zustand).**

    Lives at `apps/client/lib/stores/comments-sidebar-store.ts`. State shape:
    ```ts
    interface CommentsSidebarStoreState {
      // The section the sidebar is currently filtered to. Mirrors
      // useProposalEditorStore.activeSectionKey but is stored locally
      // so the sidebar can keep a filter pinned while the editor focus
      // shifts (future enhancement). MVP sets this from the editor store on every change.
      activeSectionKey: string | null;
      setActiveSectionKey: (sectionKey: string | null) => void;

      // Per-(proposalId, versionId) dismissal of the carry-forward banner.
      // Keyed as `${proposalId}:${versionId}`. Persisted across page reloads.
      dismissedBanners: Record<string, boolean>;
      dismissBanner: (proposalId: string, versionId: string) => void;
      isBannerDismissed: (proposalId: string, versionId: string) => boolean;

      // Sidebar open/closed mirror of ProposalWorkspacePage's local state.
      // Exposed so SectionCommentsBadge can call openSidebar(sectionKey) to
      // both open the sidebar and pin the filter.
      isOpen: boolean;
      openSidebar: (sectionKey?: string | null) => void;
      closeSidebar: () => void;

      // Per-proposal reset hook — called on unmount of ProposalEditor
      // to avoid leaking activeSectionKey across proposal navigation.
      resetForProposal: () => void;
    }

    export const useCommentsSidebarStore = create<CommentsSidebarStoreState>()(
      persist(
        (set, get) => ({ ... }),
        {
          name: "eusolicit-comments-banner-dismissals",
          partialize: (state) => ({ dismissedBanners: state.dismissedBanners }), // only persist dismissals
        },
      ),
    );
    ```

    Notes:
    - `activeSectionKey` and `isOpen` are NOT persisted — they are transient UI state.
    - `dismissedBanners` IS persisted (banner should stay dismissed across reloads for the same `(proposal, version)` tuple).
    - `openSidebar(sectionKey)` is called by `SectionCommentsBadge.onClick` and sets both `isOpen=true` AND `activeSectionKey=sectionKey` (if provided).
    - `resetForProposal()` clears `activeSectionKey` and closes the sidebar; called on `ProposalEditor` unmount.

10. [ ] **AC10 — `SectionCommentsBadge`.**

    Props: `{ proposalId: string; versionId: string | null; sectionKey: string; onClick: () => void }`.

    Subscribes to `useCommentsCountBySection(proposalId, versionId, sectionKey)` and renders:
    - **Zero unresolved, zero total** — renders `null` (no badge clutter on untouched sections).
    - **Zero unresolved, non-zero total (i.e. all resolved)** — renders `<button data-testid={\`section-comments-badge-${sectionKey}\`} onClick={onClick} className="ml-2 inline-flex items-center gap-1 rounded bg-muted px-1.5 py-0.5 text-xs text-muted-foreground hover:bg-muted/80" aria-label={t("comments.badgeAllResolvedAria", { total })}><MessageSquare className="h-3 w-3" /><span>{total}</span></button>`.
    - **Non-zero unresolved** — renders `<button data-testid={\`section-comments-badge-${sectionKey}\`} onClick={onClick} className="ml-2 inline-flex items-center gap-1 rounded bg-blue-100 px-1.5 py-0.5 text-xs font-medium text-blue-800 hover:bg-blue-200" aria-label={t("comments.badgeUnresolvedAria", { unresolved })}><MessageSquare className="h-3 w-3" /><span>{unresolved}</span></button>`.
    - **Loading** — `null` (no skeleton; badge appearing is a non-jarring late paint).

    The button's `onClick` is bound by the parent (`ProposalSection` — see AC1 modified files) to `() => useCommentsSidebarStore.getState().openSidebar(section.key)`. This both opens the sidebar AND pins the filter to this section. The parent does NOT need to pass `activeSectionKey` explicitly — the store acts as the shared channel.

    Placement: inside `<h3 data-testid="section-header-{key}">`, immediately after `<SectionLockIndicator/>` (Story 10.12). Visual spacing: `ml-2` on both, flex-row alignment from the existing `h3` style.

11. [ ] **AC11 — `ProposalWorkspacePage` integration.**

    Add a new toolbar button between the Story 10.12 "Collaborators" button and the version indicator:
    ```tsx
    <Button
      data-testid="toolbar-btn-comments"
      variant="outline"
      size="sm"
      onClick={() => setCommentsOpen(true)}
      aria-label={t("comments.toolbarBtnAria")}
    >
      <MessageSquare className="h-4 w-4" />
      {summary?.totalUnresolved ? (
        <span data-testid="toolbar-btn-comments-count" className="ml-1 text-xs">{summary.totalUnresolved}</span>
      ) : null}
    </Button>
    ```
    The count badge sources from `useCommentsSummary(proposal.id, proposal.current_version_id)` aggregated to the full-proposal `unresolved_count` (the backend already returns it — we reuse `data.unresolved_count` from the un-filtered summary query). When zero, the count span is omitted.

    Mount the sidebar at the bottom of `ProposalWorkspacePage` (siblings to `VersionHistorySidebar`, `CollaboratorPanel`, `ExportDialog`):
    ```tsx
    <CommentsSidebar
      proposalId={proposal.id}
      currentVersionId={proposal.current_version_id}
      isOpen={commentsOpen}
      onClose={() => setCommentsOpen(false)}
    />
    ```

    `commentsOpen` is a `useState(false)` at `ProposalWorkspacePage` scope; keep parity with `historyOpen`, `collaboratorPanelOpen`, `exportOpen` (Story 7.16 / 10.12 convention — do NOT lift into Zustand; per-page state). The `useCommentsSidebarStore.isOpen` is a separate flag used by `SectionCommentsBadge` to open the sidebar from outside the toolbar; `ProposalWorkspacePage` reads it via `useCommentsSidebarStore(s => s.isOpen)` in a `useEffect` that syncs it back to local `commentsOpen` (both directions: local → store on toolbar toggle, store → local on badge click). This keeps the two buttons' behaviour consistent.

    On `ProposalEditor` unmount (existing cleanup effect), call `useCommentsSidebarStore.getState().resetForProposal()` to avoid state leak when navigating between proposals — mirrors the Story 10.12 `clearLocalLockStates()` pattern.

12. [ ] **AC12 — i18n keys (`apps/client/messages/{en,bg}.json`).**

    New namespace `proposals.comments.*` with 33 keys total. EN is authoritative; BG parity is enforced by the existing `pnpm check:i18n` script (add an explicit subset parity test if the script does not already cover new namespaces).

    `proposals.comments.*` (EN):
    - `panelTitle` — "Comments"
    - `closeBtn` — "Close comments sidebar"
    - `toolbarBtnAria` — "Open comments sidebar"
    - `activeSectionLabel` — "Showing comments on: {section}"
    - `noSectionSelectedShort` — "No section selected"
    - `noSectionSelectedTitle` — "Select a section"
    - `noSectionSelectedDescription` — "Click a section in the editor to see its comments."
    - `unknownSection` — "Unknown section"
    - `errorTitle` — "Could not load comments"
    - `errorDescription` — "Please try again."
    - `emptyTitle` — "No comments yet"
    - `emptyDescription` — "Post the first review note on this section."
    - `resolvedGroupTitle` — "Resolved ({count})"
    - `newCommentPlaceholder` — "Write a review note…"
    - `submitBtn` — "Post"
    - `submittingBtn` — "Posting…"
    - `readOnlyNote` — "Read-only collaborators cannot post comments."
    - `focusSectionNote` — "Focus a section in the editor to post a comment."
    - `createSuccessTitle` — "Comment posted"
    - `createError.validation` — "Please enter a non-empty comment under 10 000 characters."
    - `createError.notAuthorised` — "You are not allowed to post comments on this proposal."
    - `createError.proposalGone` — "This proposal is no longer available."
    - `resolveError.notAuthorised` — "You are not allowed to resolve comments on this proposal."
    - `resolveError.gone` — "This comment no longer exists."
    - `resolvedLabel` — "Resolved"
    - `resolveAria` — "Toggle resolved for {author}'s comment"
    - `resolveReadOnly` — "Only write-role collaborators can resolve comments."
    - `resolvedByLabel` — "resolved by {name} {when}"
    - `carriedForwardFromPriorVersion` — "Carried forward from a prior version"
    - `carryForwardBannerMessage` — "{count, plural, one {# unresolved comment was carried forward from the prior version.} other {# unresolved comments were carried forward from the prior version.}}"
    - `carryForwardDismissAria` — "Dismiss carry-forward banner"
    - `badgeUnresolvedAria` — "{unresolved, plural, one {# unresolved comment on this section} other {# unresolved comments on this section}}"
    - `badgeAllResolvedAria` — "{total, plural, one {# comment on this section, all resolved} other {# comments on this section, all resolved}}"
    - `unknownAuthor` — "Unknown author"
    - `posting` — "Posting…"

    BG translations MUST land in `bg.json` in the same key positions. Suggested seeds (translator may refine — CI parity is the gate):
    - `panelTitle` — "Коментари"
    - `submitBtn` — "Публикувай"
    - `emptyTitle` — "Все още няма коментари"
    - `resolvedLabel` — "Решено"
    - `carriedForwardFromPriorVersion` — "Пренесен от предишна версия"
    - `carryForwardBannerMessage` — "{count, plural, one {# нерешен коментар е пренесен от предишната версия.} other {# нерешени коментара са пренесени от предишната версия.}}"
    - (full set added by the developer; CI parity check enforces 33/33 match)

    `pnpm check:i18n` (or the equivalent script run by CI) MUST exit `0`. If the project's existing parity check does not recurse into new sub-namespaces, add a one-liner test in `apps/client/__tests__/i18n-setup.test.ts`:
    ```ts
    expect(Object.keys(en.proposals.comments).sort()).toEqual(Object.keys(bg.proposals.comments).sort());
    ```

13. [ ] **AC13 — ATDD test file `apps/client/__tests__/comments-sidebar-s10-13.test.ts`.**

    Static / structural Vitest test mirroring the Story 10.12 pattern (structural + selective behavioural RTL smoke). RED-phase before implementation. Total expected assertions: **≥ 36 across 14 ACs**.

    **File-system asserts (AC1):** `existsSync` for each of the seven new component files + one new API file + one new query file + one new store file. Fail with descriptive messages.

    **API client exports (AC2):** parse `lib/api/proposal-comments.ts` for named exports `listComments`, `createComment`, `resolveComment`, `unresolveComment`, `listReplies`; assert each interface name (`CommentResponse`, `CommentWithOptimisticState`, `CommentListResponse`, `CreateCommentRequest`, `ListCommentsParams`, `CommentsBySectionMap`) appears as an `export interface`/`export type`.

    **Hook exports (AC3):** assert each hook name appears as a top-level `export function` in `lib/queries/use-proposal-comments.ts`: `useProposalComments`, `useCommentsSummary`, `useCommentsCountBySection`, `useCreateComment`, `useResolveComment`, `useUnresolveComment`. Also assert the `buildCommentsBySectionMap` helper is exported.

    **Store contract (AC9):** assert `useCommentsSidebarStore` is exported; render a smoke test that calls `getState()` and reads `activeSectionKey`, `dismissedBanners`, `isOpen`, and the five setters (`setActiveSectionKey`, `dismissBanner`, `isBannerDismissed`, `openSidebar`, `closeSidebar`, `resetForProposal`). Assert `persist` middleware wraps the creator (check for `persist` in the source).

    **i18n parity (AC12):** load `en.json` and `bg.json`; assert every `proposals.comments.*` key in en is present in bg AND vice versa; assert the key count is ≥ 33; assert the `carryForwardBannerMessage` key includes the `{count, plural, ...}` ICU plural syntax in both locales.

    **Component data-testid presence (AC4–AC11):** read each component file as text; assert the prescribed `data-testid` strings appear as literal substrings:
    - `CommentsSidebar.tsx` → `comments-sidebar`, `comments-sidebar-empty-noselection`, `comments-sidebar-skeleton`, `comments-sidebar-error`, `comments-sidebar-empty-nocomments`.
    - `CommentsSidebarHeader.tsx` → `comments-sidebar-title`, `comments-sidebar-active-section`, `comments-sidebar-close-btn`.
    - `CommentList.tsx` → (renders `CommentItem`s; assert `comment-item-` template-string substring).
    - `CommentItem.tsx` → `comment-item-`, `comment-avatar-`, `comment-author-`, `comment-timestamp-`, `comment-body-`, `comment-resolve-row-`, `comment-resolve-checkbox-`, `comment-carried-forward-`, `comment-resolved-by-`, `comment-optimistic-`.
    - `NewCommentForm.tsx` → `new-comment-form`, `new-comment-body`, `new-comment-char-counter`, `new-comment-submit-btn`, `new-comment-readonly-note`, `new-comment-no-section-note`.
    - `CarryForwardBanner.tsx` → `carry-forward-banner`, `carry-forward-banner-message`, `carry-forward-banner-dismiss-btn`.
    - `SectionCommentsBadge.tsx` → `section-comments-badge-`.
    - `ProposalWorkspacePage.tsx` → `toolbar-btn-comments`, `toolbar-btn-comments-count`.

    **Behavioural smoke (React Testing Library):** at least six scenarios, each with `it.skip` applied upfront (RED-phase pattern per Story 10-12 / 10-11 precedent) — markers come off per the TDD activation sequence:
    - `test_comments_sidebar_renders_skeleton_when_query_is_loading` — mocks `useProposalComments` to `{ isLoading: true }`; asserts skeleton testid present.
    - `test_comments_sidebar_renders_empty_state_when_no_section_selected` — mocks `useProposalEditorStore` to `{ activeSectionKey: null }`; asserts `comments-sidebar-empty-noselection` present.
    - `test_comments_sidebar_renders_two_groups_when_mixed_resolved_and_open` — mocks query to return 2 unresolved + 1 resolved; asserts open-group heading absent, resolved-group heading present with count=1, resolved row has `opacity-60` class.
    - `test_new_comment_form_hides_textarea_when_role_is_read_only` — mocks `useMyProposalRole` to `{ role: "read_only" }`; asserts `new-comment-body` is NOT in document, `new-comment-readonly-note` IS in document.
    - `test_comment_item_hides_resolve_checkbox_on_optimistic_row` — renders a `CommentItem` with `comment.optimistic=true`; asserts `comment-resolve-checkbox-` testid is absent, `comment-optimistic-` present.
    - `test_carry_forward_banner_returns_null_when_count_is_zero` — mocks `useCommentsSummary` to return `{ carriedForwardCount: 0 }`; asserts `carry-forward-banner` testid is NOT in document.

    **Mutation-surface asserts:**
    - `test_create_comment_schema_rejects_empty_body` — calls the Zod schema from `NewCommentForm` (exported as `createCommentSchema` from the component's own module OR duplicated in a `schemas/` helper — dev's choice); assert empty-string and whitespace-only bodies fail validation.
    - `test_create_comment_schema_rejects_over_10k_chars` — assert a 10001-char body fails; a 10000-char body passes.
    - `test_create_comment_mutation_prepends_optimistic_row` — mocks `queryClient.setQueryData`; calls `useCreateComment.mutate({ body: "hello" })`; asserts `setQueryData` was called with a new array whose first element has `optimistic: true`.
    - `test_create_comment_mutation_rolls_back_on_error` — mocks the API to reject; asserts `setQueryData` is called twice (insert + rollback) AND `addToast` was called with an error type.
    - `test_resolve_mutation_flips_resolved_in_cache_optimistically` — mocks the API; calls `useResolveComment.mutate(commentId)`; asserts the cache entry's `resolved` flipped to `true` BEFORE the API resolves.

    **Polling / cache-key asserts:**
    - `test_use_proposal_comments_query_key_includes_section_key` — inspects `useProposalComments("p1", "v1", "intro")` via a mock `useQuery` spy; asserts the `queryKey` is exactly `["proposal-comments", "p1", "v1", "intro"]`.
    - `test_use_comments_summary_has_60s_refetch_interval` — asserts `refetchInterval: 60_000`.
    - `test_use_comments_summary_refetches_on_window_focus` — asserts `refetchOnWindowFocus: true`.
    - `test_use_proposal_comments_is_disabled_when_section_key_null` — asserts `enabled: false` when `sectionKey === null`.

    **Store asserts:**
    - `test_dismiss_banner_persists_to_local_storage` — calls `dismissBanner("p1", "v1")`; reads `localStorage.getItem("eusolicit-comments-banner-dismissals")`; asserts the key `"p1:v1": true` is present.
    - `test_open_sidebar_with_section_key_sets_active_section` — calls `openSidebar("intro")`; asserts `activeSectionKey === "intro"` AND `isOpen === true`.
    - `test_reset_for_proposal_clears_active_section_and_closes` — sets state to `{ activeSectionKey: "intro", isOpen: true }`; calls `resetForProposal()`; asserts both cleared.

    Use `vi.mock("@/lib/stores/ui-store")` to stub `useUIStore.addToast` for the mutation-error assertions. Use `QueryClientProvider` wrapping for any hook test — the project's existing `apps/client/lib/queries/__tests__/test-utils.ts` (or equivalent) ships a helper; reuse it.

14. [ ] **AC14 — No regressions in Story 10.12 ATDD suite.** After implementing this story, re-run `vitest run __tests__/collaborator-lock-s10-12.test.ts` and confirm **160 / 160 still pass**. The new `SectionCommentsBadge` is added to the same `<h3>` as the lock indicator; the existing `section-header-{key}` and `section-lock-self-{key}` / `section-lock-foreign-{key}` testids MUST remain present and unchanged. The Story 10.12 i18n parity check MUST still pass after the new `proposals.comments.*` namespace lands.

## Design Constraints

- **Section-scoped filter is authoritative — no cross-section cache merging.** The active-section query is the source of truth for the visible list. We do NOT filter a cached full-proposal list locally. This guarantees (a) consistency with the backend `ix_proposal_comments_version_section` optimisation, (b) section-change refetches pick up teammate-posted comments within 0 s of the focus event, (c) no stale-cache bugs on the visible list.
- **Summary query and list query have separate cache keys.** `["proposal-comments-summary", proposalId, versionId]` (no `section_key`) drives the per-section badges + toolbar total + carry-forward banner. `["proposal-comments", proposalId, versionId, sectionKey]` drives the visible list. Mutations invalidate BOTH.
- **Optimistic rows carry a client-only `optimistic: true` flag and a `temp-` prefixed `id`.** These flags are NEVER sent to the backend. The `id` is regenerated server-side and replaces the temp row on invalidation. Rollback on mutation error removes the temp row.
- **Resolve/unresolve is optimistically flipped in place.** The row does NOT move between groups until the server confirms + the cache is invalidated. Avoids the mid-click "row-disappears-from-under-cursor" UX bug.
- **Polling is modest (60 s summary, 0 s on section change).** No SSE / WebSocket push in MVP — see §Out of Scope.
- **`read_only` role is UI-gated AND server-gated.** The UI hides/disables mutation affordances; the server returns 403 if a hostile client bypasses the gate. Both layers exist for belt-and-braces per the Epic 10 AC13 convention.
- **Carry-forward banner is derived from server data, not a client-side event.** Any of: page reload, tab switch, new-version created by a teammate all surface the banner correctly via the next summary-query refetch.
- **Dismissed-banner state is persisted; `activeSectionKey` and `isOpen` are not.** Reload should preserve user intent ("I dismissed that banner") but not UI position ("the sidebar was open").
- **No new package dependencies.** Reuses shadcn primitives, TanStack Query, Zustand, Zod, React Hook Form already in `apps/client/package.json`. `MessageSquare` / `ArrowDownCircle` / `MousePointerClick` / `MessageSquarePlus` / `AlertCircle` icons come from `lucide-react` (already a dependency).
- **Thread expansion (the `GET /comments/{id}/replies` path) is scaffolded but not surfaced in UI.** The `listReplies` API function is exported for forward-compatibility; no `<CommentThread>` component ships. A future story can add threaded replies if the conversation volume warrants it. The Epic file's story summary does not call out threads explicitly for 10.13 — top-level comments only for MVP.
- **No comment editing or deletion affordances.** Backend Story 10.4 explicitly omits both endpoints ("body is immutable after create"; "no delete in MVP"). The UI follows suit — no edit pencil, no delete menu. The `read_only` group owns the entire mutation surface for this story.
- **Author and resolver avatars are computed from initials, not uploaded images.** Matches Story 10.12's convention. The `author_full_name` (falls back to `user_email` never populated here) is the initial source; if null, fall back to `"?"`.
- **`created_at` is displayed as relative time with ISO tooltip.** Matches industry convention; accessibility-friendly via `<time dateTime={...}>` + `title`.
- **The sidebar's empty state for "no section selected" is distinct from the empty state for "no comments on this section".** Two distinct testids (`comments-sidebar-empty-noselection` vs. `comments-sidebar-empty-nocomments`) so QA can assert the correct copy fires.

## Tasks / Subtasks

- [ ] **Task 1: API client + query hooks. (AC2, AC3)**
  - [ ] Create `apps/client/lib/api/proposal-comments.ts` with the six interfaces and five functions per AC2. Mirror the Story 10.12 `section-locks.ts` axios style.
  - [ ] Create `apps/client/lib/queries/use-proposal-comments.ts` with the six hooks + `buildCommentsBySectionMap` helper per AC3.
  - [ ] Unit-test `buildCommentsBySectionMap` in `apps/client/lib/queries/__tests__/use-proposal-comments.test.ts` with three seeds: empty, single section, multi-section-mixed-resolved.

- [ ] **Task 2: Store. (AC9)**
  - [ ] Create `apps/client/lib/stores/comments-sidebar-store.ts` with the seven state fields + six setters per AC9.
  - [ ] Wrap in `persist` middleware with `partialize` restricted to `dismissedBanners` — reload should NOT preserve `activeSectionKey` or `isOpen`.
  - [ ] Unit-test store behaviour: `openSidebar(sectionKey)` sets both `isOpen` and `activeSectionKey`; `resetForProposal` clears both; `dismissBanner` writes to `localStorage`.

- [ ] **Task 3: Components — sidebar skeleton (no mutations yet). (AC4, AC5, AC7)**
  - [ ] Create `CommentsSidebar.tsx` with the three states (disabled / loading / error / empty / populated) per AC4.
  - [ ] Create `CommentsSidebarHeader.tsx` per AC5.
  - [ ] Create `CommentList.tsx` that renders the two-group layout (unresolved + resolved) per AC4.3.
  - [ ] Create `CommentItem.tsx` per AC7 with all the fields (avatar, author, timestamp, body, resolve row, carry-forward marker, optimistic badge).

- [ ] **Task 4: Components — new-comment form + banner + badge. (AC6, AC8, AC10)**
  - [ ] Create `NewCommentForm.tsx` with React Hook Form + Zod per AC6. Export `createCommentSchema` so the ATDD test can import it.
  - [ ] Create `CarryForwardBanner.tsx` per AC8.
  - [ ] Create `SectionCommentsBadge.tsx` per AC10.

- [ ] **Task 5: Workspace + section integration. (AC1 modified files, AC11)**
  - [ ] Modify `ProposalWorkspacePage.tsx` to add the toolbar button + mount the sidebar + wire `commentsOpen` to both local state and `useCommentsSidebarStore.isOpen`.
  - [ ] Modify `ProposalSection.tsx` to render `<SectionCommentsBadge>` inside the existing `<h3>` after the `<SectionLockIndicator>`. Bind the badge's `onClick` to `useCommentsSidebarStore.getState().openSidebar(section.key)`.
  - [ ] Wire the unmount cleanup in `ProposalEditor.tsx` (the existing `useEffect` that clears `useProposalEditorStore` state + `useCollaboratorStore.clearLocalLockStates()`) to also call `useCommentsSidebarStore.getState().resetForProposal()`.

- [ ] **Task 6: i18n. (AC12)**
  - [ ] Add 33 keys to `apps/client/messages/en.json` under `proposals.comments.*` per AC12. Use ICU plural syntax for `carryForwardBannerMessage`, `badgeUnresolvedAria`, `badgeAllResolvedAria`, `resolvedGroupTitle`.
  - [ ] Add the same 33 keys to `apps/client/messages/bg.json` with seeded Bulgarian translations (developer provides initial pass; a translator refines later).
  - [ ] Run `pnpm check:i18n` — confirm 0 exit. If the existing parity check does not catch drift on sub-namespaces, extend `apps/client/__tests__/i18n-setup.test.ts`.

- [ ] **Task 7: Optimistic mutations. (AC3 mutation hooks, AC6 form submission, AC7 resolve checkbox)**
  - [ ] Implement `useCreateComment` with `onMutate` prepending a temp-UUID row, `onSuccess` invalidating both cache keys, `onError` rolling back.
  - [ ] Implement `useResolveComment` with `onMutate` flipping `resolved` in cache, `onSettled` invalidating both keys, `onError` flipping back + toast.
  - [ ] Mirror for `useUnresolveComment`.
  - [ ] Verify the hooks integrate cleanly in the component tree — mount the sidebar, post a comment, confirm the optimistic row appears, confirm it is replaced on success.

- [ ] **Task 8: ATDD test file. (AC13)**
  - [ ] Create `apps/client/__tests__/comments-sidebar-s10-13.test.ts` per AC13 (≥ 36 assertions across the 14 ACs).
  - [ ] Apply `it.skip` to the behavioural + mutation tests (RED phase); remove during TDD activation.
  - [ ] Run `vitest run __tests__/comments-sidebar-s10-13.test.ts` — expect green on the structural assertions, skipped on the behavioural ones.

- [ ] **Task 9: Regression verification. (AC14)**
  - [ ] Run `vitest run __tests__/collaborator-lock-s10-12.test.ts` → expect **160 / 160 passing** (Story 10.12 unchanged).
  - [ ] Run `tsc --noEmit` from `apps/client/` → expect NO NEW errors introduced by this story's files. Pre-existing errors in unrelated files (e.g., `@tiptap/extension-table` default import) are owned by other stories — document any residual noise in Change Log.
  - [ ] Run the full Vitest suite (`pnpm test` or `pnpm test:client`) and confirm no regressions.

- [ ] **Task 10: Lint + format + manual smoke.**
  - [ ] `pnpm lint` clean on all new/modified files in `apps/client/`.
  - [ ] `pnpm format:check` clean.
  - [ ] Manual smoke: open a proposal workspace, focus a section, click the Comments toolbar button, post a comment, observe optimistic insert, observe server-confirmed replacement, check the resolved checkbox, observe in-place flip + delayed re-group, create a new version via Version History, observe the carry-forward banner, dismiss it, reload — confirm it stays dismissed.

## Dev Notes

### Test Expectations (from Epic 10 Test Design surrogate sources)

**NOTE on `test-design-epic-10` absence:** Epic 10 was never epic-test-designed — same caveat as Stories 10.4 / 10.9 / 10.10 / 10.11 / 10.12. Verified at story-creation time: `eusolicit-docs/test-artifacts/` contains per-story ATDD checklists `atdd-checklist-10-1.md` through `atdd-checklist-10-12.md` and `test-design/` contains `test-design-epic-01.md` through `test-design-epic-09.md`, `test-design-epic-11.md`, `test-design-epic-12.md` — but NO `test-design-epic-10.md`. Epic 10 was skipped in the epic-test-design pipeline (same gap noted in Story 10.12 Dev Notes).

Test expectations for this story are therefore drawn from four surrogate sources:

- **Epic 10 AC3 (verbatim)** — "Comments can be created on a section+version, listed per section, resolved/unresolved, and unresolved comments carry forward when a new version is created." This story closes AC3 **end-to-end** on the frontend track: creation UI (AC6), per-section+version filter (AC4.3), resolve/unresolve UI (AC7 + AC3 hooks), and the carry-forward banner (AC8) making the server-side carry-forward visible to the user.
- **Epic 10 AC11 (frontend surface AC, verbatim):** "comments sidebar" is explicitly named in the list of frontend surfaces Epic 10 must ship. This story closes that sub-item.
- **Epic 10 AC13 (RBAC enforcement):** "All endpoints enforce proposal-level RBAC via the collaborator role; read_only users cannot mutate." The UI mirror (AC7 resolve checkbox hidden for read_only; AC6 new-comment form hidden for read_only) is this story's contribution to AC13.
- **Story 10.4 ATDD checklist (RBAC matrix + backend contract):** `atdd-checklist-10-4-proposal-comments-api.md` enumerates the exact `CommentResponse`/`CommentListResponse` shapes and the 5-endpoint × 8-caller-profile × 40-assertion matrix for the server. The frontend takes those shapes as-is (AC2) and relies on the server's RBAC to enforce 403 paths; the UI additionally hides affordances for the `read_only` profile to avoid 403 round-trips.

Requirement + risk rows this story owns:

| Priority | Requirement | Level | Count | Test Harness |
|---|---|---|---|---|
| **P0** | Epic 10 AC3 — comments sidebar filters to the active section + current version | UI | 2 | ATDD structural + RTL smoke (`test_comments_sidebar_renders_empty_state_when_no_section_selected`, query-key assert). |
| **P0** | Epic 10 AC3 — `POST /comments` happy path with optimistic insertion + rollback | UI | 2 | `test_create_comment_mutation_prepends_optimistic_row`, `test_create_comment_mutation_rolls_back_on_error`. |
| **P0** | Epic 10 AC3 — resolve/unresolve toggles with optimistic flip | UI | 1 | `test_resolve_mutation_flips_resolved_in_cache_optimistically`. |
| **P0** | Epic 10 AC3 — carry-forward banner surfaces when `carried_forward_from_id !== null` count > 0 | UI | 1 | `test_carry_forward_banner_returns_null_when_count_is_zero` (+ inverse positive test during TDD activation). |
| **P0** | Epic 10 AC11 — per-section unresolved-count badge on section header | UI | 2 | testid assertion `section-comments-badge-{key}` + `useCommentsCountBySection` selector unit test. |
| **P0** | Epic 10 AC13 — read_only cannot post or resolve | UI | 2 | `test_new_comment_form_hides_textarea_when_role_is_read_only`, disabled-checkbox assertion in `CommentItem`. |
| **P1** | Body validation: empty, whitespace, >10k chars rejected client-side (422 defence-in-depth) | UI | 3 | `test_create_comment_schema_rejects_empty_body`, `test_create_comment_schema_rejects_over_10k_chars`, plus whitespace case. |
| **P1** | Dismissed-banner state persists across reload | UI | 1 | `test_dismiss_banner_persists_to_local_storage`. |
| **P1** | Store `resetForProposal` clears state on proposal navigation | UI | 1 | `test_reset_for_proposal_clears_active_section_and_closes`. |
| **P1** | Optimistic rows render dimmed + hide resolve checkbox | UI | 1 | `test_comment_item_hides_resolve_checkbox_on_optimistic_row`. |
| **P1** | Summary query refetches every 60 s + on window focus | UI | 2 | `test_use_comments_summary_has_60s_refetch_interval`, `test_use_comments_summary_refetches_on_window_focus`. |
| **P1** | Section-scoped query disables when `sectionKey === null` | UI | 1 | `test_use_proposal_comments_is_disabled_when_section_key_null`. |
| **P2** | i18n EN/BG parity on the new `proposals.comments.*` namespace (33 keys) | CI | 1 | i18n parity assertion. |
| **P2** | No regressions in Story 10.12 ATDD suite | CI | 1 | `vitest run __tests__/collaborator-lock-s10-12.test.ts` → 160/160. |

**Coverage strategy:** Vitest with `environment: "jsdom"` for component tests and `environment: "node"` for the structural file-system asserts (mix in the same file via `vitest-environment` directives — Story 7.12 + 10.12 set the precedent). Use React Testing Library + `userEvent` for interactive scenarios, `vi.useFakeTimers()` for the relative-time formatter, `QueryClientProvider` wrapping for any TanStack Query hook test. Mock `useAuthStore` via `vi.mock("@eusolicit/ui", async () => ({ ...await vi.importActual("@eusolicit/ui"), useAuthStore: vi.fn() }))` — matches Story 10.12 convention. Apply `it.skip` upfront to behavioural tests (RED phase) so the suite compiles green at story-creation time; markers come off during TDD activation.

### Backend contract verification — read-before-write

Before authoring `lib/api/proposal-comments.ts`, confirm the live routes in `services/client-api/src/client_api/main.py`:
- The `proposal_comments_v1` router is registered via `api_v1_router.include_router(proposal_comments_v1.router, prefix="/proposals")` (Story 10.4 Task 4). Its own `APIRouter` prefix is `/{proposal_id}/comments`, so the full paths resolve to `/api/v1/proposals/{proposal_id}/comments*`.
- Expected status codes from `services/client-api/src/client_api/api/v1/proposal_comments.py` (Story 10.4 AC4–AC7, AC10):
  - `POST /comments` → 201 happy, 400 bad version/parent, 403 read_only, 404 cross-company/missing proposal, 422 Pydantic validation.
  - `GET /comments` → 200 happy (including empty), 400 mismatched `version_id`, 403 non-collaborator non-admin, 404 cross-company.
  - `PATCH /comments/{id}/resolve` and `.../unresolve` → 200 happy (including idempotent no-op), 403 read_only, 404 missing comment OR tenant mismatch.
  - `GET /comments/{id}/replies` → 200 (including empty list), 404 missing parent.
- The `CommentResponse` shape is frozen by Story 10.4 AC3 and the ATDD matrix — do NOT hallucinate fields. Particular care: `resolved_by_full_name` is nullable (populated via LEFT JOIN, null for unresolved OR for deleted users per ON DELETE SET NULL), `author_full_name` is nullable (never null for active users but defensive against FK restrict-deleted rare cases), `carried_forward_from_id` is the critical banner driver.
- Capture the literal URL strings in the API client function bodies — do NOT abstract behind a path-builder for this story; the five endpoints are well-bounded. This matches the Story 10.12 convention.

### Reuse `useMyProposalRole` from Story 10.12 — do NOT duplicate

The `useMyProposalRole(proposalId)` hook is the single source of truth for "what can I do on this proposal?" across Epic 10 frontend. Story 10.12 defines it in `apps/client/lib/queries/use-collaborators.ts`. This story imports it verbatim:
```ts
import { useMyProposalRole } from "@/lib/queries/use-collaborators";
```
The hook returns `{ role: "bid_manager" | "technical_writer" | "financial_analyst" | "legal_reviewer" | "read_only" | "admin" | null; isLoading: boolean }`. The `canMutate` derivation for this story is: `role !== null && role !== "read_only"`. The admin bypass is handled inside the hook (Story 10.12 Dev Note). No additional auth logic is needed here.

### Reuse `useProposalEditorStore` for active section — do NOT re-wire Tiptap focus

Story 7.12's `ProposalEditor` already wires each `ProposalSection.onFocus` to `useProposalEditorStore.setActiveSectionKey(key)`. This story subscribes to `useProposalEditorStore(s => s.activeSectionKey)` in the sidebar (AC4.1) and pipes it into `useCommentsSidebarStore.activeSectionKey` via a `useEffect`. Do NOT add new focus handlers to `ProposalSection`; the existing ones are sufficient. If a future story needs a "pin filter" button (so the sidebar's filter doesn't follow the focus), the local store already supports it (the setter can be called directly).

### TanStack Query cache semantics

- `useProposalComments` cache key: `["proposal-comments", proposalId, versionId, sectionKey]`. `staleTime: 0` so every mount / key-change refetches; `refetchOnWindowFocus: true` catches teammate-posted updates on tab return.
- `useCommentsSummary` cache key: `["proposal-comments-summary", proposalId, versionId]`. `staleTime: 30_000`, `refetchInterval: 60_000`, `refetchIntervalInBackground: false`, `refetchOnWindowFocus: true`. The `select` option returns `{ bySection: CommentsBySectionMap; carriedForwardCount: number }` so every consumer gets the shape it needs without re-iterating raw comments.
- Mutations invalidate BOTH keys on success AND on error (so badge counts and lists stay consistent even when the user's optimistic action fails).
- Cross-proposal navigation does not require explicit cache cleanup — the per-`proposalId` key naturally segregates caches. The store's `resetForProposal` handles UI state cleanup only.

### Optimistic insertion pattern (TanStack Query v5)

```ts
const mutation = useMutation({
  mutationFn: (body: string) => createComment(proposalId, { version_id: versionId, section_key: sectionKey, body }),
  onMutate: async (body) => {
    await queryClient.cancelQueries({ queryKey: ["proposal-comments", proposalId, versionId, sectionKey] });
    const previous = queryClient.getQueryData<CommentListResponse>(["proposal-comments", proposalId, versionId, sectionKey]);
    const tempRow: CommentWithOptimisticState = {
      id: `temp-${crypto.randomUUID()}`,
      proposal_id: proposalId,
      version_id: versionId,
      section_key: sectionKey,
      author_id: currentUser.id,
      author_full_name: currentUser.fullName,
      body,
      parent_comment_id: null,
      resolved: false,
      resolved_by: null,
      resolved_by_full_name: null,
      resolved_at: null,
      carried_forward_from_id: null,
      created_at: new Date().toISOString(),
      updated_at: new Date().toISOString(),
      optimistic: true,
    };
    queryClient.setQueryData<CommentListResponse>(
      ["proposal-comments", proposalId, versionId, sectionKey],
      (old) => old ? { ...old, comments: [...old.comments, tempRow], total: old.total + 1, unresolved_count: old.unresolved_count + 1 } : old,
    );
    return { previous, tempId: tempRow.id };
  },
  onError: (_err, _body, context) => {
    if (context?.previous) {
      queryClient.setQueryData(["proposal-comments", proposalId, versionId, sectionKey], context.previous);
    }
    addToast({ type: "error", title: t("comments.createError.validation"), description: /* status-code dispatch */ });
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ["proposal-comments", proposalId, versionId, sectionKey] });
    queryClient.invalidateQueries({ queryKey: ["proposal-comments-summary", proposalId, versionId] });
  },
});
```

The `onSettled` (not just `onSuccess`) handler fires on both success and error — this ensures the summary counts stay correct even when the temp row was rolled back. The Story 10.12 F11 precedent on 5xx toast mapping applies: add an explicit `status >= 500` branch in the error handler that uses `t("forms.serverError")` + `console.error`.

### Resolve/unresolve optimistic flip

```ts
onMutate: async (commentId) => {
  await queryClient.cancelQueries({ queryKey: ["proposal-comments", proposalId, versionId, sectionKey] });
  const previous = queryClient.getQueryData<CommentListResponse>(["proposal-comments", proposalId, versionId, sectionKey]);
  queryClient.setQueryData<CommentListResponse>(
    ["proposal-comments", proposalId, versionId, sectionKey],
    (old) => old ? {
      ...old,
      comments: old.comments.map(c => c.id === commentId ? { ...c, resolved: true, resolved_by: currentUser.id, resolved_by_full_name: currentUser.fullName, resolved_at: new Date().toISOString() } : c),
      unresolved_count: Math.max(0, old.unresolved_count - 1),
    } : old,
  );
  return { previous };
},
```

Note the clamp `Math.max(0, old.unresolved_count - 1)` — defensive against race conditions where the count is already stale.

### Relative-time formatter

Use `Intl.RelativeTimeFormat` with the current locale (derived from `next-intl`'s `useLocale()`). Bucket into: `<60s` → "just now", `<60m` → "N min ago", `<24h` → "N h ago", `<7d` → "N d ago", else `new Date(iso).toLocaleDateString(locale)`. Implementation is a pure function exported from `lib/utils/relative-time.ts` (or similar — check for an existing helper in `apps/client/lib/utils/` before creating a new one). Tooltip shows the full ISO via `title={new Date(iso).toLocaleString(locale)}`.

### Character counter + max-length handling

- Textarea has `maxLength={10000}` — the browser enforces the cap before submit.
- Zod schema independently enforces `max(10000)` — defence-in-depth if `maxLength` is bypassed (disabled via devtools).
- Character counter is visible only when `value.length > 0` (avoids visual noise on an empty form). Colour shifts to `text-amber-600` at `>= 9000` and `text-red-600` at `>= 9900` for proactive warning.

### `Sheet` primitive reuse

Check `@eusolicit/ui` for the `Sheet` / `Drawer` primitive Story 10.12 uses for `CollaboratorPanel`. If it's a shadcn-based wrapper, reuse it verbatim — visual + a11y parity (focus-trap, Escape-to-close, overlay click-to-close) is free. The `<Sheet side="right">` API signature matches Story 10.12's usage. If the primitive has per-sheet state collision issues (e.g., two sheets open simultaneously should stack), verify behaviour before shipping — MVP assumes only one sheet is open at a time (the Toolbar toggles between Version History / Collaborators / Comments mutually, but there is no hard constraint; opening multiple at once is defensible as a power-user feature).

### `data-testid` discipline

Every interactive element MUST carry a stable `data-testid`. Per-comment testids include the comment id (e.g. `comment-item-{comment.id}`); per-section testids include the section key (`section-comments-badge-{section_key}`). This matches the Story 7.12 (`section-header-{key}`) and Story 10.12 (`collaborator-row-{user_id}`) convention.

### i18n ICU plural syntax

The three plural-aware keys (`carryForwardBannerMessage`, `badgeUnresolvedAria`, `badgeAllResolvedAria`, `resolvedGroupTitle`) use ICU plural syntax:
```
"carryForwardBannerMessage": "{count, plural, one {# unresolved comment was carried forward from the prior version.} other {# unresolved comments were carried forward from the prior version.}}"
```
`next-intl`'s `t("key", { count: 3 })` correctly resolves plurals in both EN and BG. Bulgarian has the same `one`/`other` split as English for these cases (no special `few`/`many` cases needed; if Bulgarian plural rules require it, the ICU message handles it automatically). Verify with a test asserting `t("carryForwardBannerMessage", { count: 1 })` vs `count: 2` produce different strings.

### Out of scope

- **Real-time push of comment changes (WebSockets / SSE).** Polling at 60 s (summary) + refetch-on-window-focus is the MVP spec. A future story can upgrade to SSE if comments become a high-traffic surface during QA.
- **Threaded replies UI.** The backend supports `parent_comment_id` (Story 10.4) but this story does NOT surface a reply affordance. Top-level comments only. `listReplies` is exported for forward-compat.
- **Comment editing.** Backend omits the endpoint (Story 10.4 Design Decision d). UI follows — no edit pencil.
- **Comment deletion.** Backend omits the endpoint (Story 10.4 Design Constraint). UI follows — no delete menu. If moderation becomes a requirement, a follow-up story adds both.
- **@-mentions with user picker.** The `NewCommentForm` textarea is plain text. @-mentions that notify teammates via the Story 9.x notification stream are a follow-up.
- **Rich-text formatting (bold, italic, markdown) in comments.** Plain text only; body is rendered via `white-space: pre-wrap` for multi-line preservation. A future story can introduce a mini-Tiptap instance for rich-text review notes.
- **Cross-version comment viewing.** The sidebar always filters to the current version. Viewing comments on an earlier version (via Version History) is a follow-up; the backend supports it via the `version_id` query param.
- **Orphan-section badges.** If a reviewer commented on a `section_key` that was later deleted or renamed, the backend does not prevent the orphan comment from existing (Story 10.4 Design Decision e). This story does NOT surface "orphan" badges; it lists comments on the active section only (so orphans become invisible, which is mildly surprising but MVP-acceptable). A future story can add an "Orphan comments" filter to the sidebar header.
- **Avatar uploads / Gravatar integration.** Initials only — matches Story 10.12's scope.
- **Bulgarian translation polish.** Initial BG translations are seeded by the developer; a professional translator pass is a follow-up. CI parity (`pnpm check:i18n`) enforces structure only.
- **Mobile-specific UX for the sidebar.** The panel is `w-full` on mobile per AC4, but the new-comment textarea and relative-time tooltips are desktop-first. Touch-optimised edits are TBD.

### Future-compatibility notes

- **Story 10.14 (Task Kanban Board) and 10.16 (Approval Pipeline)** operate at the company/opportunity scope, not the proposal scope — they do NOT need `useMyProposalRole` or the sidebar primitives directly. However, they share the same `@eusolicit/ui` `Sheet` / `Drawer` + shadcn Card + Badge palette, so the visual rhythm (right-rail slide-over, testid prefix convention, `data-testid={\`xyz-${id}\`}` template literal pattern) stays consistent across Epic 10.
- **SSE upgrade path.** When a future story wires SSE for comment streams, `useProposalComments` can be refactored to consume the SSE stream instead of polling — its external surface (returning a `CommentListResponse` shape) wouldn't need to change; only the query function's implementation swaps.
- **Thread expansion.** `listReplies` is pre-wired; a future `<CommentThread>` component can consume it without an API-client change.
- **Comment presence ("3 reviewers reading this comment").** A future story can add a lightweight presence layer keyed on `(proposal_id, comment_id)`. The sidebar architecture (per-comment `CommentItem`) admits a presence badge slot trivially.
- **The 60 s summary-poll interval is a server-friendly default.** If comment traffic during QA proves noisy, the interval can be bumped to 120 s or moved behind a subscription-tier gate; the structural change is zero (single-line constant).

## Senior Developer Review

**Reviewer:** bmad-code-review (adversarial)
**Date:** 2026-04-21
**Outcome:** Changes Requested

### Summary

Core architecture is sound and spec-faithful. The store/partialize restriction to `dismissedBanners`, the query-key shape (`["proposal-comments", ...]` vs `["proposal-comments-summary", ...]`), the `refetchInterval: 60_000` + `staleTime: 0` split, the optimistic resolve/unresolve `Math.max(0, ...)` clamp, the `read_only` inline-note pattern, the `createCommentSchema` export for test access, i18n parity (35=35 keys, superset of spec's 33; ICU plural in the banner + badge aria strings), the `ProposalEditor` unmount → `resetForProposal()` wiring, and the `SectionCommentsBadge` placement inside `<h3>` right after `<SectionLockIndicator>` are all implemented correctly. The ATDD suite mirrors the 10.12 RED-phase structure and exceeds the ≥36 assertion target.

Two issues block approval; the remainder are deferrable polish.

### Blocking findings

**F1 — `CommentsSidebarHeader` never resolves the active section title (AC5 regression).**
- `CommentsSidebarHeader.tsx:9–25` accepts an optional `activeSectionTitle` prop and falls back to `t("unknownSection")` when it is undefined.
- `CommentsSidebar.tsx:52–56` does **not** pass `activeSectionTitle`.
- Net effect: whenever a section is focused, the header always shows `"Showing comments on: Unknown section"` — the user never sees the real section title, making the filter opaque.
- AC5 mandates the header resolve the title via `useProposalEditorStore(s => s.content?.sections.find(s => s.key === activeSectionKey)?.title ...)`. Either (a) move the lookup inside `CommentsSidebarHeader` per spec, or (b) have `CommentsSidebar` compute the title from `useProposalEditorStore` and pass it as `activeSectionTitle`. Option (a) matches the spec literally.
- `DEVIATION: CommentsSidebarHeader does not resolve the section title for the active section — falls through to "unknown section" in all real-use cases`
- `DEVIATION_TYPE: ACCEPTANCE_GAP`
- `DEVIATION_SEVERITY: blocking`

**F2 — Optimistic row is appended, spec says prepend (AC3 / Description (b) / AC13).**
- `use-proposal-comments.ts:164` — `comments: [...old.comments, tempRow]` appends.
- AC3 onMutate doc-comment, Description (b) ("The row is prepended to the TanStack-Query cache"), and the ATDD test name `test_create_comment_mutation_prepends_optimistic_row` all specify prepend.
- Functionally benign today because the server re-sorts `created_at ASC` on invalidation, so the final rendered order converges. But the skipped ATDD behavioural test will fail when un-skipped (it asserts "first element has `optimistic: true`" per AC13). Fix is one line: `[tempRow, ...old.comments]`.
- `DEVIATION: Optimistic insertion appends instead of prepending; contradicts AC3/AC13 and will fail the RED→GREEN activation of test_create_comment_mutation_prepends_optimistic_row`
- `DEVIATION_TYPE: CONTRADICTORY_SPEC`
- `DEVIATION_SEVERITY: blocking`

### Deferrable findings (post-approval polish)

**F3 — `ProposalWorkspacePage` store↔local sync is one-way.**
- `ProposalWorkspacePage.tsx` syncs store `isOpen` → local `commentsOpen` (and closes on sheet dismissal), but the toolbar button setting local `commentsOpen = true` never flips `storeIsOpen`. AC11 explicitly requires bi-directional parity so the two entry points (toolbar + badge click) stay consistent. Minor since the badge path still works; two-way sync matters for edge cases (e.g. focus-trap handoff, future keyboard shortcut).
- `DEVIATION_TYPE: ACCEPTANCE_GAP` / `DEVIATION_SEVERITY: deferrable`

**F4 — Toolbar unresolved total does not use `data.unresolved_count`.**
- `ProposalWorkspacePage.tsx:418–420` computes the count by reducing over `commentsSummary.bySection`. AC11 asks to "reuse `data.unresolved_count` from the un-filtered summary query." The `select` transform at `use-proposal-comments.ts:77–82` discards the raw `unresolved_count`, forcing this workaround. Numerically equivalent but wastes a reduce and loses the server's canonical aggregate. Fix: extend `select` to carry `totalUnresolved: data.unresolved_count`.
- `DEVIATION_TYPE: ACCEPTANCE_GAP` / `DEVIATION_SEVERITY: deferrable`

**F5 — `resolvedGroupTitle` EN value is interpolation, not ICU plural.**
- Task 6 lists `resolvedGroupTitle` among the ICU-plural keys; EN renders `"Resolved ({count})"` as straight interpolation, which is fine for English but technically misses the ICU syntax the Task 6 checklist calls out. Low impact — the displayed string reads correctly. Tighten to `"Resolved ({count, plural, one {#} other {#}})"` only if strict parity with Task 6 wording is required.
- `DEVIATION_TYPE: ACCEPTANCE_GAP` / `DEVIATION_SEVERITY: deferrable`

**F6 — `err: any` leaks in three mutation `onError` handlers.**
- `use-proposal-comments.ts:174, 261, 343`. Prefer `unknown` + `AxiosError` narrowing or a typed error-utils helper. Matches existing lint convention for the rest of `apps/client`.
- `DEVIATION_TYPE: ARCHITECTURAL_DRIFT` / `DEVIATION_SEVERITY: deferrable`

**F7 — Per-row `useMutation` instantiation in `CommentItem`.**
- `CommentItem.tsx:86–87` builds fresh `resolveMutation` / `unresolveMutation` hooks per row. On a 50-comment section that's 100 hooks per render. TanStack's `useMutation` is cheap but a single list-level mutation (or the `useMutation({ mutationFn: ({id, nextState}) => ... })` shape called from a callback) would be tidier. Not a correctness bug.
- `DEVIATION_TYPE: ARCHITECTURAL_DRIFT` / `DEVIATION_SEVERITY: deferrable`

**F8 — `unresolve` optimistic increment is unclamped.**
- `use-proposal-comments.ts` (unresolve onMutate) increments `unresolved_count` without a corresponding `Math.min(total, unresolved_count + 1)` safety. Spec only mandates the clamp on the decrement path, and `onSettled` invalidation reconciles — symmetric risk is minimal. Defensive addition would be `unresolved_count: Math.min(old.total, old.unresolved_count + 1)`.
- Not a DEVIATION against spec; noted as a minor hardening opportunity.

### Strengths

- `CarryForwardBanner` null-short-circuit order (`versionId === null`, count=0, dismissed) is correct.
- `formatRelativeTime` wraps `Intl.RelativeTimeFormat` with try/catch fallback.
- Zustand `persist` correctly restricts persistence to `dismissedBanners` only; storage key matches spec (`eusolicit-comments-banner-dismissals`).
- `useProposalComments` `enabled` gating on `Boolean(proposalId && versionId && sectionKey)` matches AC3.
- ATDD file has query-key assertions, refetchInterval assertions, persist-to-localStorage assertions, and behavioural scenarios all with `it.skip` per RED-phase convention.
- `ProposalSection` badge placement is correct — inside the existing `<h3>` after `<SectionLockIndicator>`, preserving Story 10.12 testid surface.
- API client uses literal URL strings (no path-builder abstraction) matching Story 10.12 convention.

### Required next steps before approval

1. Fix F1 — move the section-title lookup into `CommentsSidebarHeader` (per AC5) or thread `activeSectionTitle` from `CommentsSidebar` to the header.
2. Fix F2 — change `[...old.comments, tempRow]` to `[tempRow, ...old.comments]` in `useCreateComment.onMutate`.
3. Re-run `vitest run __tests__/comments-sidebar-s10-13.test.ts` and `vitest run __tests__/collaborator-lock-s10-12.test.ts` → confirm no regressions.

The deferrable findings (F3–F8) can be addressed in a small follow-up or left for the next Epic 10 hardening pass; none of them block end-to-end functionality.

### Re-review 2026-04-21 (post-fix)

**Reviewer:** bmad-code-review (adversarial)
**Outcome:** Approve

**F1 verification — RESOLVED.** `CommentsSidebarHeader.tsx:21-23` now resolves the active section title via `useProposalEditorStore((s) => s.sections.find((sec) => sec.key === activeSectionKey)?.title ?? null)` per AC5. The fallback chain (real title → `unknownSection` if focused but missing → `noSectionSelectedShort` if no focus) reads correctly.

**F2 verification — RESOLVED.** `use-proposal-comments.ts:164` now uses `comments: [tempRow, ...old.comments]` (prepend) per AC3, Description (b), and AC13's `test_create_comment_mutation_prepends_optimistic_row`.

**Spot-check on additional surfaces:**
- `CommentsSidebar.tsx` correctly wires editor → sidebar store via `useEffect`, derives `canMutate` once via `useMyProposalRole`, and threads it down. ✓
- `useResolveComment` clamps decrement with `Math.max(0, ...)` (line 253). ✓
- `onSettled` invalidates BOTH cache keys for create/resolve/unresolve. ✓
- 5xx branch with `console.error` + `forms.serverError` toast present in all three mutations. ✓
- ATDD test file exists at `apps/client/__tests__/comments-sidebar-s10-13.test.ts` — 841 lines, exceeds the ≥36 assertion target. ✓
- `ProposalWorkspacePage.tsx:453-458` syncs store → local state (one-way per F3 deferrable). The toolbar button still does not push back into the store, but this remains the agreed deferrable polish.

The deferrable findings F3 (one-way store sync), F4 (toolbar count via reduce instead of `data.unresolved_count`), F5 (resolvedGroupTitle non-ICU), F6 (`err: any`), F7 (per-row `useMutation`), and F8 (unresolve unclamped) are unchanged; none block end-to-end functionality and can roll into an Epic 10 hardening pass.

**REVIEW: Approve**

## Change Log

- 2026-04-21: Initial story context created. (bmad-create-story)
- 2026-04-21: Senior Developer Review completed — Changes Requested (2 blocking, 6 deferrable findings). (bmad-code-review)
- 2026-04-21: Re-review after blocking-fix iteration — Approve (F1 + F2 resolved; F3–F8 remain as deferrable polish). (bmad-code-review)

## Known Deviations

### Detected by `3-code-review` at 2026-04-21T06:46:29Z (session fbafa085-4729-4630-82af-6a2f7c752b44)

- CommentsSidebarHeader does not resolve active section title — always shows "unknown section" once a section is focused _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- Optimistic row is appended rather than prepended; contradicts AC3 / AC13 test name _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- CommentsSidebarHeader does not resolve active section title — always shows "unknown section" once a section is focused _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- Optimistic row is appended rather than prepended; contradicts AC3 / AC13 test name _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
