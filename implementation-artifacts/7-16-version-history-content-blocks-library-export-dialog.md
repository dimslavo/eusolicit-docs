# Story 7.16: Version History, Content Blocks Library & Export Dialog

Status: draft

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager**,
I want a **Version History sidebar**, a **Content Blocks Library modal**, and an **Export dialog** in the proposal workspace,
so that I can review and restore past proposal versions, insert reusable content snippets at any cursor position, and export a publication-ready PDF or DOCX document — completing the full proposal lifecycle.

## Acceptance Criteria

### Version History Sidebar

1. **Sidebar toggle** — Clicking the History button (`toolbar-btn-history`, already wired in S07.11) opens a slide-in overlay drawer from the right (`data-testid="version-history-sidebar"`). Clicking it again closes the drawer. The drawer overlays the right panel without collapsing it.

2. **Version list** — The drawer shows a scrollable list of versions fetched from `GET /api/v1/proposals/:id/versions`. Each row has `data-testid="version-row-{version_number}"` and displays: version number (`v{N}`), creation timestamp (formatted as relative time using `date-fns` `formatDistanceToNow`), author name, and an optional change summary truncated to 80 chars. The list is sorted newest first (server-guaranteed).

3. **Preview mode** — Clicking a version row calls `GET /api/v1/proposals/:id/versions/:vid` and renders the version content in the `ProposalEditor` with `editable={false}` (read-only). A banner (`data-testid="version-preview-banner"`) appears above the editor with "Viewing v{N} — Read only" and a "Return to current version" button (`data-testid="btn-return-to-current"`) that restores the editor to the current version content.

4. **Compare / diff viewer** — Each version row shows a "Compare" button (`data-testid="btn-compare-{version_number}"`). Clicking it opens `GET /api/v1/proposals/:id/versions/diff?from={current_version_id}&to={selected_vid}` and renders a diff viewer (`data-testid="version-diff-viewer"`).
   - **Side-by-side** (default): two columns, before on left and after on right, with insertions highlighted in green (`data-added="true"`) and deletions in red (`data-removed="true"`).
   - A toggle (`data-testid="diff-toggle-inline"` / `data-testid="diff-toggle-side-by-side"`) switches between inline (single column with +/- markers) and side-by-side views.

5. **Rollback** — Each version row shows a "Rollback" button (`data-testid="btn-rollback-{version_number}"`). Clicking it opens a confirmation dialog (`data-testid="rollback-confirm-dialog"`) with text "Create a new version from v{N}?". Confirming calls `POST /api/v1/proposals/:id/versions/:vid/rollback`. On success, invalidate `["proposal", id]` and `["proposal-versions", id]` caches; close the history sidebar; show a toast "Rolled back to v{N}".

6. **Loading and error states** — Version list shows `<SkeletonCard />` while loading. Rollback in progress shows a spinner on the confirm button. API errors show a toast.

### Content Blocks Library Modal

7. **Open / close** — A "Content Library" button (`data-testid="btn-open-content-library"`) is added to the proposal workspace toolbar (or the left panel tab triggers it via `data-testid="left-tab-content-library"` already in S07.11). Clicking it opens a full-screen modal (`data-testid="content-library-modal"`). The modal closes via an X button or `Escape` key.

8. **Real-time search** — The modal contains a search input (`data-testid="content-library-search"`). On each keystroke (debounced 300ms), call `GET /api/v1/content-blocks/search?q={term}`. Below the search, show category filter chips (`data-testid="content-library-category-chip-{category}"`); clicking a chip filters by that category (appends `?category={category}` to the query).

9. **Result list** — Each result is a card (`data-testid="content-block-card-{id}"`) showing: title, a 120-char preview snippet of the body, tag badges, and an approval badge (green "Approved" or grey "Pending"). An **"Insert"** button (`data-testid="btn-insert-block-{id}"`) inserts the block's HTML body at the active Tiptap editor cursor position.

10. **Insert at cursor** — Call `useProposalEditorStore().getActiveEditor()?.commands.insertContent(block.body)`. If no editor is active (no section focused), show a toast "Click a section in the editor first, then insert." and do not close the modal.

11. **Empty and loading states** — Show a loading spinner while fetching; an empty state when no results match.

### Export Dialog

12. **Open dialog** — Clicking `toolbar-btn-export` opens a dialog (`data-testid="export-dialog"`).

13. **Format selector** — The dialog shows two format options with icons:
    - **PDF** (`data-testid="export-format-pdf"`) — FileText icon.
    - **DOCX** (`data-testid="export-format-docx"`) — FileCode icon.
    The selected format is visually highlighted with a border/ring. PDF is selected by default.

14. **Branding preview** — Below the selector, show a small preview thumbnail (`data-testid="export-branding-preview"`) depicting the company name and logo (from `proposal.company_id` via a `useCompany(companyId)` hook). This is a static placeholder display — not a real rendered page preview.

15. **Download** — A "Download" button (`data-testid="btn-export-download"`) calls `POST /api/v1/proposals/:id/export` with `{ format: "pdf" | "docx" }`. The response is a file blob. Use `window.URL.createObjectURL` + a transient `<a>` element click to trigger the browser download. Set the filename to `proposal-{proposal.id}.{ext}`.

16. **Progress indicator** — While the export request is in flight, disable the Download button and show a spinner with text "Generating {FORMAT}…" (`data-testid="export-progress"`). Large proposals may take 5–10 seconds.

17. **Error handling** — On export failure, show an error toast with "Export failed. Please retry." The dialog remains open.

### Shared

18. **i18n** — All strings use `useTranslations("proposals")`. Required keys added to both locale files.

19. **ATDD tests** — A test file `__tests__/version-history-content-blocks-export-s7-16.test.ts` covers: all component files exist, all `data-testid` attributes present, rollback confirmation dialog present, diff toggle buttons present, insert-at-cursor `insertContent` call present, `window.URL.createObjectURL` usage for download, i18n key completeness. All tests GREEN after implementation.

## Tasks / Subtasks

- [ ] Task 1: Install `date-fns` (if not already installed) (AC: 2)
  - [ ] 1.1 Check `pnpm list date-fns` — it may already be installed as a TanStack Query / shadcn dependency
  - [ ] 1.2 If absent: `pnpm --filter client add date-fns`

- [ ] Task 2: Add version history and export API functions to `lib/api/proposals.ts` (AC: 2, 4, 5, 15)
  - [ ] 2.1 Add `ProposalVersion` interface: `{ id: string; version_number: number; change_summary: string | null; created_by: string; created_at: string; content: ProposalVersionContent }`
  - [ ] 2.2 Add `SectionDiff` interface: `{ key: string; status: "added" | "removed" | "changed" | "unchanged"; before: string | null; after: string | null }`
  - [ ] 2.3 Add `getProposalVersions(proposalId: string): Promise<ProposalVersion[]>`
  - [ ] 2.4 Add `getProposalVersion(proposalId: string, versionId: string): Promise<ProposalVersion>`
  - [ ] 2.5 Add `diffProposalVersions(proposalId: string, fromId: string, toId: string): Promise<SectionDiff[]>`
  - [ ] 2.6 Add `rollbackToVersion(proposalId: string, versionId: string): Promise<ProposalVersion>`
  - [ ] 2.7 Add `exportProposal(proposalId: string, format: "pdf" | "docx"): Promise<Blob>` — use `apiClient.post(..., { responseType: "blob" })`

- [ ] Task 3: Add content blocks API functions to `lib/api/proposals.ts` (AC: 8, 9)
  - [ ] 3.1 Add `ContentBlock` interface: `{ id: string; title: string; body: string; category: string; tags: string[]; approved_at: string | null }`
  - [ ] 3.2 Add `searchContentBlocks(query: string, category?: string): Promise<ContentBlock[]>`

- [ ] Task 4: Create `VersionHistorySidebar` component (AC: 1–6)
  - [ ] 4.1 Create `…/proposals/[id]/components/VersionHistorySidebar.tsx`
  - [ ] 4.2 `useQuery` for `getProposalVersions` with key `["proposal-versions", id]`; `isOpen` prop drives render
  - [ ] 4.3 Implement version list with `formatDistanceToNow` timestamps
  - [ ] 4.4 Implement preview mode: local state `previewVersionId`; when set, pass `previewContent` to `ProposalEditor` and set `editable={false}`; preview banner with return button
  - [ ] 4.5 Implement diff viewer with side-by-side/inline toggle; use `dangerouslySetInnerHTML` only for diff content rendered inside `data-testid="version-diff-viewer"` — escape content first
  - [ ] 4.6 Implement rollback with confirmation dialog using `AlertDialog` from `@eusolicit/ui`; `useMutation` for `rollbackToVersion`

- [ ] Task 5: Create `ContentLibraryModal` component (AC: 7–11)
  - [ ] 5.1 Create `…/proposals/[id]/components/ContentLibraryModal.tsx`
  - [ ] 5.2 `isOpen` / `onClose` props; render as `Dialog` from `@eusolicit/ui`
  - [ ] 5.3 Debounced search (300ms) using `useDebounce` from `@eusolicit/ui` or a custom `useRef`-based debounce
  - [ ] 5.4 `useQuery` with `["content-blocks", query, category]` key; `enabled: query.length >= 2`
  - [ ] 5.5 Insert action: `useProposalEditorStore().getActiveEditor()?.commands.insertContent(block.body)`; guard for null editor with toast fallback

- [ ] Task 6: Create `ExportDialog` component (AC: 12–17)
  - [ ] 6.1 Create `…/proposals/[id]/components/ExportDialog.tsx`
  - [ ] 6.2 `isOpen` / `onClose` props; render as `Dialog` from `@eusolicit/ui`
  - [ ] 6.3 Format selector with visual highlight; state `format: "pdf" | "docx"` defaults to `"pdf"`
  - [ ] 6.4 `useMutation` for `exportProposal`; on success: create `objectURL`, create transient `<a>` element, set `href` and `download` attribute, click and revoke URL
  - [ ] 6.5 Branding preview: fetch company name via `useCompany(proposal.company_id)` hook (if available) or use `proposal.company_id` as fallback label

- [ ] Task 7: Wire all components into `ProposalWorkspacePage` (AC: 1, 7, 12)
  - [ ] 7.1 `VersionHistorySidebar` — wire `toolbar-btn-history` toggle state (already in S07.11) to `isOpen` prop
  - [ ] 7.2 `ContentLibraryModal` — wire `left-tab-content-library` or a new toolbar button to open state; replace `content-library-panel-placeholder`
  - [ ] 7.3 `ExportDialog` — wire `toolbar-btn-export` click to `isExportOpen` state

- [ ] Task 8: Unskip E2E Playwright spec (AC: per E07-P0-011)
  - [ ] 8.1 Unskip `test.skip("S07.16 required — version history and export")` in `proposals-workspace.spec.ts`
  - [ ] 8.2 Implement version history navigation test; mock `GET /proposals/:id/versions`
  - [ ] 8.3 Implement export download test using `page.waitForEvent("download")`

- [ ] Task 9: Add i18n keys (AC: 18) + Write ATDD tests (AC: 19)

## Dev Notes

### Diff Viewer — Content Safety

The section diff data comes from the backend (`SectionDiff.before` and `SectionDiff.after` are HTML strings). Do NOT use `dangerouslySetInnerHTML` directly on raw backend HTML. Sanitize with DOMPurify before rendering:
```typescript
import DOMPurify from "dompurify";
// ...
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(diff.after ?? "") }} />
```
Install: `pnpm --filter client add dompurify && pnpm --filter client add -D @types/dompurify`

Check if DOMPurify is already installed (it may have been added in S06.13 for AI summary rendering).

### Export Blob Download Pattern

```typescript
const exportMutation = useMutation({
  mutationFn: () => exportProposal(proposalId, selectedFormat),
  onSuccess: (blob) => {
    const url = window.URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = `proposal-${proposalId}.${selectedFormat}`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    window.URL.revokeObjectURL(url);
  },
  onError: () => {
    useUIStore.getState().addToast({ type: "error", message: t("export.error") });
  },
});
```

`apiClient.post` must be called with `responseType: "blob"` (Axios config) — this returns the raw binary as a `Blob` instead of attempting JSON parse.

### Version Preview — Editor Integration

Preview mode requires passing content to `ProposalEditor` from outside. Extend the `useProposalEditorStore` with `setPreviewContent(content: ProposalVersionContent | null)`: when set, the editor renders the preview content (read-only); when null, renders the current version content.

Alternatively, pass `previewContent` as a prop from `ProposalWorkspacePage` — cleaner for one-way data flow.

### `date-fns` Relative Timestamps

```typescript
import { formatDistanceToNow } from "date-fns";
// ...
formatDistanceToNow(new Date(version.created_at), { addSuffix: true })
// → "about 2 hours ago"
```

### i18n Keys Required (abbreviated)

```json
{
  "proposals": {
    "versionHistory": {
      "title": "Version History",
      "labelVersion": "v{number}",
      "btnCompare": "Compare",
      "btnRollback": "Rollback",
      "btnReturnCurrent": "Return to current version",
      "previewBanner": "Viewing v{number} — Read only",
      "rollbackConfirm": "Create a new version from v{number}?",
      "rollbackSuccess": "Rolled back to v{number}",
      "diffSideBySide": "Side by side",
      "diffInline": "Inline"
    },
    "contentLibrary": {
      "title": "Content Library",
      "searchPlaceholder": "Search blocks…",
      "btnInsert": "Insert",
      "noActiveEditor": "Click a section in the editor first, then insert.",
      "empty": "No content blocks found.",
      "badgeApproved": "Approved",
      "badgePending": "Pending"
    },
    "export": {
      "title": "Export Proposal",
      "labelPdf": "PDF Document",
      "labelDocx": "Word Document",
      "btnDownload": "Download",
      "progressPdf": "Generating PDF…",
      "progressDocx": "Generating DOCX…",
      "error": "Export failed. Please retry."
    }
  }
}
```

### File Structure

```
eusolicit-app/frontend/apps/client/
├── app/[locale]/(protected)/proposals/[id]/components/
│   ├── VersionHistorySidebar.tsx     # NEW
│   ├── ContentLibraryModal.tsx       # NEW
│   └── ExportDialog.tsx              # NEW
├── lib/api/
│   └── proposals.ts                  # MODIFIED — versions, diff, rollback, content-blocks, export
├── messages/
│   ├── en.json                       # MODIFIED
│   └── bg.json                       # MODIFIED
└── __tests__/
    └── version-history-content-blocks-export-s7-16.test.ts  # NEW
```

Also modify:
- `ProposalWorkspacePage.tsx` — wire history toggle, content library open state, export dialog open state
- Possible: `useProposalEditorStore` (from S07.12) — add `setPreviewContent` or `previewContent` prop pass-through

### Test Coverage Map (from Epic 7 Test Design)

| Test Design ID | Priority | Scenario | Coverage |
|---|---|---|---|
| **E07-P0-011** | P0 | E2E: Generate → Accept All → Version History → Export PDF | Playwright |
| **E07-P1-030** | P1 | Version history lists versions; side-by-side diff; insertions green, deletions red | Component test |
| **E07-P1-031** | P1 | Content blocks: search, filter, insert at cursor | Component test |
| **E07-P2-012** | P2 | Export dialog: format selector, Download triggers API | Component test |
| **E07-P2-013** | P2 | Rollback confirmation dialog prevents accidental rollback | Component test |
| **E07-P3-005** | P3 | Diff toggle: side-by-side ↔ inline | Component test |

### References

- [Source: eusolicit-docs/planning-artifacts/epics/E07-proposal-generation.md#S07.16] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P0-011, E07-P1-030, E07-P1-031, E07-P2-012, E07-P2-013] — test scenarios
- [Source: eusolicit-docs/implementation-artifacts/7-3-proposal-versioning-api.md] — versions, diff, rollback backend endpoints
- [Source: eusolicit-docs/implementation-artifacts/7-9-content-blocks-crud-search-api.md] — content blocks search endpoint
- [Source: eusolicit-docs/implementation-artifacts/7-10-document-export-api-pdf-docx.md] — export endpoint (blob response)
- [Source: eusolicit-docs/implementation-artifacts/7-12-tiptap-rich-text-editor-with-section-based-editing.md] — `useProposalEditorStore`, `getActiveEditor()`
- [Source: eusolicit-docs/implementation-artifacts/7-11-proposal-workspace-page-layout-navigation.md] — toolbar buttons wired; history toggle state
- [Source: eusolicit-app/frontend/packages/ui/index.ts] — `AlertDialog`, `Dialog`, `Badge` components
- [Source: eusolicit-app/frontend/apps/client/lib/api/proposals.ts] — extend with new API functions
