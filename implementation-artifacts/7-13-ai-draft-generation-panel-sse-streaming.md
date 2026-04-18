# Story 7.13: AI Draft Generation Panel with SSE Streaming

Status: draft

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **bid manager**,
I want an **AI Draft Generation panel in the right sidebar** where I can trigger AI-powered draft generation, watch each section stream in progressively, and then choose to accept or discard the generated content,
so that I can capture AI-generated proposal content without losing my editor's current work and retain full control over what gets committed to the proposal.

## Acceptance Criteria

1. **Generate Draft button** — The "AI Generate" tab in the right panel (placed in `data-testid="ai-generate-panel-placeholder"` from S07.11) is replaced with a real `<AiGeneratePanel />` component. When `generation_status` is `idle`, it renders a "Generate Draft" button (`data-testid="btn-generate-draft"`). Clicking it calls `POST /api/v1/proposals/:id/generate` and initiates the SSE stream. The toolbar's existing "Generate" button (`toolbar-btn-generate`) delegates to the same action.

2. **SSE stream consumption** — Use native `fetch` + `ReadableStream` to consume the SSE response (not `EventSource` — the endpoint requires authentication headers that `EventSource` cannot send). Parse the stream using the same `parseSSEStream` utility in `lib/api/opportunities.ts` (or a shared helper). Each parsed event has `event` (string) and `data` (JSON object).

3. **Progressive section rendering** — As each `section` event arrives (`{section_key, title, body}`), the streamed sections are displayed in a scrollable preview list inside the panel (`data-testid="generation-preview-list"`). Each section in the list has `data-testid="preview-section-{section_key}"` and shows the section title and a truncated body preview (first 120 chars). A checkmark icon appears next to each section as it completes.

4. **Progress indicator** — A progress bar or step indicator (`data-testid="generation-progress"`) shows how many sections have been received. The indicator is indeterminate while streaming and fills as sections arrive. An estimated section count derived from the current proposal's section structure is used as the denominator; if zero sections exist yet, display `"Generating…"` text only.

5. **Streaming state — editor lock** — While `generation_status` is `generating` (detected from the SSE stream start and reset on `done`/`error`):
   - The `ProposalEditor` receives `isGenerating={true}` (AC11 of S07.12), disabling all section editors.
   - The "Generate Draft" button is replaced by a "Stop" button (`data-testid="btn-stop-generation"`). Clicking "Stop" aborts the `fetch` stream (via `AbortController`) and resets the panel to idle state. A PATCH is NOT sent to reset `generation_status` on the backend — the cleanup job handles stuck proposals.
   - The panel displays `"Generating draft…"` status text.

6. **Accept All action** — After the `done` SSE event is received (stream complete), show three action buttons:
   - **"Accept All"** (`data-testid="btn-accept-all"`): replaces all sections in the Tiptap editor with the streamed content. Calls `PUT /api/v1/proposals/:id/content` with the complete sections array. On success, invalidates the `["proposal", id]` React Query cache. The `done` event body contains `{version_id, version_number}` — update the version indicator in the toolbar after accept. Show "Saved" in the save status indicator on success.
   - **"Accept Section"** — disabled during streaming (per E07-R-008 mitigation); enabled only after stream completes. Each section in the preview list gains an individual "Accept" button that inserts that section's content into the matching section editor (replacing existing body). Does not trigger a full-save — auto-save debounce handles persistence.
   - **"Discard"** (`data-testid="btn-discard-generation"`): clears the preview list and returns panel to idle state. No API call. Editor content is unchanged.

7. **Error handling** — If the SSE stream emits an `error` event or the `fetch` throws (network failure, 4xx/5xx response), display an error message in the panel (`data-testid="generation-error-message"`) with text from the error event data or a generic fallback. Show a "Retry" button (`data-testid="btn-retry-generation"`). The "Generate Draft" button is restored.

8. **Duplicate trigger guard** — If the server returns HTTP 409 (`generation_in_progress`), show an informational message (`data-testid="generation-in-progress-message"`) with text "A draft is already being generated. Please wait." Do not show an error state — this is a transient status.

9. **`generation_status` from proposal data** — On workspace load, if `proposal.generation_status` is `generating` (backend reports stuck state), show the panel in a warning state with message "A previous generation may be in progress. Wait a moment and refresh if this persists." This uses the `generation_status` field that must be added to the `ProposalResponse` interface (deferred from S07.11).

10. **i18n** — All strings in `<AiGeneratePanel />` use `useTranslations("proposals")`. Required keys added to both `messages/en.json` and `messages/bg.json` under the `proposals` namespace (see Dev Notes for key list).

11. **ATDD tests** — A test file `__tests__/ai-generate-panel-s7-13.test.ts` covers: component file existence, all `data-testid` attributes present in source, SSE fetch call (not `new EventSource`), `AbortController` usage, `Accept All` API call, i18n key completeness. All tests must be GREEN after implementation.

## Tasks / Subtasks

- [ ] Task 1: Update `ProposalResponse` interface — add `generation_status` field (AC: 9)
  - [ ] 1.1 In `lib/api/proposals.ts`, add `generation_status: "idle" | "generating" | "completed" | "failed"` to `ProposalResponse` interface
  - [ ] 1.2 Verify against `services/client-api/src/client_api/schemas/proposal.py` — confirm backend returns `generation_status` in `GET /proposals/:id` response
  - [ ] 1.3 Update the `useProposal` hook if `staleTime` needs adjustment for generation polling (consider `refetchInterval` when `generation_status === "generating"`)

- [ ] Task 2: Create `AiGeneratePanel` component (AC: 1–9)
  - [ ] 2.1 Create `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/proposals/[id]/components/AiGeneratePanel.tsx`
  - [ ] 2.2 Implement idle state: "Generate Draft" button; call `POST /api/v1/proposals/:id/generate` using native `fetch` with `Authorization` header from `apiClient`'s token
  - [ ] 2.3 Parse SSE stream using `ReadableStream` + `TextDecoder`; accumulate sections in `useState<SectionPreview[]>`
  - [ ] 2.4 Implement `AbortController` for "Stop" button; store ref in `useRef<AbortController | null>`
  - [ ] 2.5 Implement progress indicator — `sections.length / estimatedCount * 100` or indeterminate if `estimatedCount === 0`
  - [ ] 2.6 Implement Accept All: `PUT /api/v1/proposals/:id/content` with all streamed sections; invalidate React Query cache; update save status
  - [ ] 2.7 Implement Accept Section: individual section insert via `useProposalEditorStore().getActiveEditor()` or section-key lookup via a new `insertSectionContent(key, body)` store method
  - [ ] 2.8 Implement Discard: clear `streamedSections` state, reset to idle
  - [ ] 2.9 Implement error state with retry; 409 in-progress state
  - [ ] 2.10 Implement `generation_status === "generating"` warning on workspace load

- [ ] Task 3: Wire `isGenerating` prop from `ProposalWorkspacePage` to `ProposalEditor` (AC: 5)
  - [ ] 3.1 In `ProposalWorkspacePage.tsx`, lift `isGenerating` state — set to `true` when SSE stream starts, `false` on `done`/`error`/`abort`
  - [ ] 3.2 Pass `isGenerating` prop through from workspace to `ProposalEditor`
  - [ ] 3.3 Wire toolbar `toolbar-btn-generate` click to `AiGeneratePanel`'s trigger function (via callback prop or shared state)

- [ ] Task 4: Add `insertSectionContent` to `useProposalEditorStore` (AC: 6 Accept Section)
  - [ ] 4.1 In the Zustand store defined in S07.12, add `sectionEditors: Map<string, Editor>` (keyed by `section_key`) and `insertSectionContent(key: string, body: string): void`
  - [ ] 4.2 Each `ProposalSection` component registers its editor instance in the store on mount and deregisters on unmount
  - [ ] 4.3 `insertSectionContent` calls `editor.commands.setContent(html)` on the matched section editor

- [ ] Task 5: Add i18n keys (AC: 10)
  - [ ] 5.1 Add all `aiGenerate.*` keys to `messages/en.json` and `messages/bg.json` (see Dev Notes)

- [ ] Task 6: Write ATDD tests (AC: 11)
  - [ ] 6.1 Create `__tests__/ai-generate-panel-s7-13.test.ts`
  - [ ] 6.2 Assert `AiGeneratePanel.tsx` file exists; all `data-testid` attributes present
  - [ ] 6.3 Assert `fetch` used for SSE (not `new EventSource`) — verify string `new EventSource` is NOT in component source
  - [ ] 6.4 Assert `AbortController` used for cancellation
  - [ ] 6.5 Assert `PUT /api/v1/proposals/${id}/content` call present in Accept All handler
  - [ ] 6.6 Assert all `aiGenerate.*` i18n keys present in both locale files

- [ ] Task 7: Update Playwright E2E spec (AC: per E07-P0-011)
  - [ ] 7.1 Unskip `test.skip("S07.13 required — AI generate SSE")` in `e2e/specs/proposals/proposals-workspace.spec.ts`
  - [ ] 7.2 Implement using Playwright `page.route()` to intercept `/api/v1/proposals/:id/generate` and return a mock SSE stream with 2 section events + done
  - [ ] 7.3 Assert progress panel shows section checkmarks; Accept All replaces editor content

## Dev Notes

### SSE Stream Pattern (Native Fetch, Not EventSource)

From E06 pattern (project-context.md): `EventSource` is GET-only and cannot send auth headers. Use native `fetch` with `ReadableStream`:

```typescript
const abortController = new AbortController();
const response = await fetch(`/api/v1/proposals/${proposalId}/generate`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${token}`,
    Accept: "text/event-stream",
  },
  signal: abortController.signal,
});

if (!response.ok) {
  if (response.status === 409) { /* in-progress */ return; }
  throw new Error(`HTTP ${response.status}`);
}

const reader = response.body!.getReader();
const decoder = new TextDecoder();
let buffer = "";

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buffer += decoder.decode(value, { stream: true });
  // parse SSE events from buffer
  const events = parseSSEBuffer(buffer);
  for (const event of events.parsed) {
    if (event.type === "section") { /* accumulate */ }
    if (event.type === "done") { /* finalize */ }
    if (event.type === "error") { /* handle */ }
  }
  buffer = events.remainder;
}
```

Get `token` from `useAuthStore().accessToken` (Zustand store from E03).

### `parseSSEBuffer` Helper

Reuse or adapt the existing SSE parser from `lib/api/opportunities.ts` (used by AI summary panel in S06.13). The function splits the buffer on `\n\n` event boundaries, parses `event:` and `data:` lines, and returns `{ parsed: SSEEvent[], remainder: string }`.

Check `lib/api/opportunities.ts` for the existing implementation before writing a new one. Extract to `lib/utils/sse.ts` if not already shared.

### Accept All — Content Shape

The `PUT /api/v1/proposals/:id/content` endpoint (S07.04) expects:
```json
{
  "sections": [{ "key": "string", "title": "string", "body": "string" }],
  "content_hash": "md5_of_current_content"
}
```

For Accept All, compute `content_hash` from the **current** proposal content using the same MD5 algorithm as S07.12 (from `useProposalEditorStore().contentHash`). This prevents overwriting concurrent changes.

### ProposalResponse — `generation_status` Field

Backend `GET /api/v1/proposals/:id` now returns `generation_status` (added in S07.05 migration). Verify by checking:
- `services/client-api/src/client_api/schemas/proposal.py` — `ProposalDetailResponse` class
- `services/client-api/src/client_api/api/v1/proposals.py` — the detail endpoint serializer

If the field is absent from the response schema, add it before implementing the panel.

### File Structure

```
eusolicit-app/frontend/apps/client/
├── app/[locale]/(protected)/proposals/[id]/components/
│   └── AiGeneratePanel.tsx          # NEW
├── lib/
│   └── utils/
│       └── sse.ts                   # NEW (or already exists from S06.13 — check first)
├── messages/
│   ├── en.json                      # MODIFIED — add aiGenerate.* keys
│   └── bg.json                      # MODIFIED — add aiGenerate.* keys
└── __tests__/
    └── ai-generate-panel-s7-13.test.ts  # NEW
```

Also modify:
- `ProposalWorkspacePage.tsx` — replace `ai-generate-panel-placeholder` with `<AiGeneratePanel />`; add `isGenerating` state; wire `toolbar-btn-generate`
- `ProposalEditor.tsx` (from S07.12) — `isGenerating` prop already exists per S07.12 AC11

### i18n Keys Required

```json
{
  "proposals": {
    "aiGenerate": {
      "title": "AI Draft Generation",
      "btnGenerate": "Generate Draft",
      "btnStop": "Stop",
      "btnAcceptAll": "Accept All",
      "btnAcceptSection": "Accept",
      "btnDiscard": "Discard",
      "btnRetry": "Retry",
      "statusGenerating": "Generating draft…",
      "statusComplete": "Generation complete",
      "statusError": "Generation failed",
      "statusInProgress": "A draft is already being generated. Please wait.",
      "statusStuck": "A previous generation may be in progress. Wait a moment and refresh if this persists.",
      "progressLabel": "Generating section {current} of {total}",
      "progressIndeterminate": "Generating…"
    }
  }
}
```

### Critical Mistakes to Prevent

1. **Do NOT use `new EventSource`** — cannot send auth headers; backend returns 401 immediately.
2. **Do NOT call a PATCH on "Stop"** — the cleanup job (`reset_stuck_proposals_task`) handles backend state reset.
3. **Do NOT enable "Accept Section" during streaming** — E07-R-008 mitigation: mixed-state acceptance creates desync between editor and backend version.
4. **Do NOT omit `content_hash` from Accept All PUT** — optimistic locking required; compute from `useProposalEditorStore().contentHash`.
5. **AbortController must be stored in `useRef`** — storing in `useState` causes re-render on creation.

### Test Coverage Map (from Epic 7 Test Design)

| Test Design ID | Priority | Scenario | Coverage |
|---|---|---|---|
| **E07-P0-011** | P0 | End-to-end: Generate Draft → SSE progress → Accept All | `AiGeneratePanel.tsx` — Playwright E2E |
| **E07-P1-025** | P1 | SSE panel renders sections progressively | Component test — Vitest + RTL |
| **E07-P1-026** | P1 | Accept All fires PUT; Discard clears without API call | Component test |
| **E07-P2-015** | P2 | Generation in-progress disables editor; Stop button visible | Component test |

### References

- [Source: eusolicit-docs/planning-artifacts/epics/E07-proposal-generation.md#S07.13] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-07.md#E07-P1-025, E07-P1-026, E07-P2-015] — test scenarios
- [Source: eusolicit-docs/implementation-artifacts/6-13-ai-summary-panel-with-sse-streaming.md] — SSE panel pattern (E06 reference)
- [Source: eusolicit-docs/implementation-artifacts/7-5-ai-draft-generation-backend-sse-integration.md] — backend SSE endpoint spec
- [Source: eusolicit-docs/implementation-artifacts/7-12-tiptap-rich-text-editor-with-section-based-editing.md] — `useProposalEditorStore`, `isGenerating` prop, `contentHash`
- [Source: eusolicit-docs/implementation-artifacts/7-11-proposal-workspace-page-layout-navigation.md] — `AiGeneratePanel` placeholder location, toolbar btn wiring
- [Source: eusolicit-app/frontend/apps/client/lib/api/opportunities.ts] — SSE parseSSEStream helper (check for reuse)
- [Source: eusolicit-app/frontend/packages/ui/src/stores/authStore.ts] — `accessToken` for fetch Authorization header
- [Source: eusolicit-app/services/client-api/src/client_api/schemas/proposal.py] — verify `generation_status` in response
