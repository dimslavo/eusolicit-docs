# Story 6.13: AI Summary Panel with SSE Streaming

Status: complete

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **paid-tier user viewing the AI Analysis tab on the opportunity detail page**,
I want **an AI summary panel that checks for a cached summary on mount, streams a new AI-generated executive summary via SSE when I click "Generate" (with typing animation and live progress), shows my remaining monthly quota, and surfaces an upgrade prompt when my limit is reached**,
so that **I can view AI-powered opportunity intelligence without leaving the detail page, see progress in real time, and understand my plan's usage limits**.

## Acceptance Criteria

1. `AISummaryPanel.tsx` is created at `app/[locale]/(protected)/opportunities/components/AISummaryPanel.tsx`. The file begins with `'use client'` (requires browser APIs: `fetch`, `ReadableStream`, `AbortController`, `crypto.randomUUID`). The component is exported as a named export for use by `AIAnalysisTab.tsx`.

2. `AIAnalysisTab.tsx` is modified (minimally) to import and render `<AISummaryPanel opportunityId={opportunity.id} initialSummary={opportunity.ai_summary} />` in place of all existing TODO-marked stub content (static summary display, usage placeholder, and both buttons with TODO comments). `AIAnalysisTab.tsx` becomes a thin tab wrapper that delegates all panel logic to `AISummaryPanel`.

3. When mounted with `initialSummary != null`:
   - Summary content text is displayed in `data-testid="ai-summary-content"`.
   - Metadata is displayed in `data-testid="ai-summary-metadata"` showing `t("aiTokensUsed", { count: summary.tokens_used })` (generation time omitted on initial load since no elapsed time is available yet).
   - "Regenerate" button (`data-testid="ai-regenerate-btn"`) is visible.
   - Empty state and Generate CTA are hidden.

4. When mounted with `initialSummary == null`:
   - Empty state container (`data-testid="ai-empty-state"`) is shown with `t("aiEmptyDescription")`.
   - "Generate Summary" button (`data-testid="ai-generate-btn"`) is visible.
   - No skeleton is shown until the user clicks Generate.

5. On "Generate" CTA click, the streaming flow begins:
   - Immediately transition to `streaming` state: hide the Generate button and show pulsing skeleton (`data-testid="ai-summary-skeleton"`) while waiting for the first `delta` event.
   - On first `delta` event: hide the skeleton, show `data-testid="ai-streaming-text"` container with accumulated text and a blinking cursor indicator (`data-testid="ai-typing-cursor"`, `aria-hidden="true"`) appended to the end.
   - Each subsequent `delta.text` chunk is appended to the accumulated text; the cursor stays at the end.
   - Generation start time is recorded with `Date.now()` for elapsed-time calculation at completion.

6. "Regenerate" click (when a summary already exists) opens a confirmation dialog before streaming:
   - Dialog container: `data-testid="ai-regenerate-confirm-dialog"`.
   - Shows `t("aiRegenerateConfirmTitle")` as title.
   - Shows `t("aiRegenerateConfirmMessage", { remaining: usageRemaining })` if `usageRemaining != null`; otherwise `t("aiRegenerateConfirmMessageUnknown")`.
   - "Generate" confirm button: `data-testid="ai-regenerate-confirm-action"` — triggers the same streaming flow as Generate.
   - "Cancel" button: `data-testid="ai-regenerate-confirm-cancel"` — closes dialog without streaming.

7. Streaming uses native `fetch` (NOT `EventSource`, NOT `new EventSource(url)`, NOT the `useSSE` hook) for `POST /api/v1/opportunities/${opportunityId}/ai-summary` with headers:
   - `Authorization: Bearer ${token}` (token from `useAuthStore()`)
   - `Content-Type: application/json`
   - `x-request-id: crypto.randomUUID()`
   Streaming is aborted via `AbortController` signal; the controller is aborted in `useEffect` cleanup on unmount.

8. SSE events parsed from the response body `ReadableStream`:
   - `delta`: `{ text: string }` — appended to `streamedText` state; sets `hasFirstDelta: true` on first occurrence.
   - `metadata`: `{ tokens: number; model: string }` — stored in `metadata` state.
   - `done`: `{ summary_id: string }` — transitions to `complete` state; elapsed time since stream start (rounded to 1 decimal second) stored; calls `queryClient.invalidateQueries({ queryKey: ["opportunity-detail", opportunityId] })` to refresh the detail page cache.
   - `error`: `{ message: string }` — transitions to `error` state (unless message is `"usage_limit_exceeded"` — see AC 11).

9. After streaming completes (`done` event):
   - `data-testid="ai-summary-content"` displays `streamedText`.
   - `data-testid="ai-summary-metadata"` shows `t("aiSummaryCompleteMeta", { tokens, seconds })` using metadata tokens and elapsed seconds.
   - `data-testid="ai-regenerate-btn"` is visible.
   - Skeleton and streaming container are hidden.

10. Usage counter (`data-testid="ai-usage-counter"`) shows:
    - Before first generation: `t("aiUsagePlaceholder")` (existing key — do not re-add).
    - After successful generation: `t("aiUsageRemaining", { remaining, limit })` where `remaining` = `parseInt(X-Usage-Remaining response header)` and `limit` = `TIER_SUMMARY_LIMITS[subscriptionTier]`.
    - For Enterprise tier: `t("aiUsageUnlimited")`.
    - `subscriptionTier` read from `useAuthStore()` → `user?.subscription_tier ?? "free"`.
    - `TIER_SUMMARY_LIMITS` constant: `{ free: 0, starter: 10, professional: 50, enterprise: Infinity }`.

11. If POST returns HTTP 429 (usage limit exceeded):
    - Set `usageRemaining` to `0`.
    - Hide Generate/Regenerate CTA.
    - Show upgrade prompt (`data-testid="ai-upgrade-prompt"`) with `t("aiUpgradeForSummaries")` text.
    - Do NOT transition to `error` state — this is a limit state, not a failure.

12. On network error or SSE `error` event (non-429):
    - Display `data-testid="ai-summary-error"` with `t("aiSummaryError")`.
    - Display retry button `data-testid="ai-summary-retry-btn"` with `t("aiSummaryRetry")`.
    - Clicking retry re-invokes the streaming flow.

13. `lib/api/opportunities.ts` is extended with:
    - `SSEHandlers` interface: `{ onDelta(text: string): void; onMetadata(tokens: number, model: string): void; onDone(summaryId: string): void; onError(message: string): void; }`
    - `getAISummary(opportunityId: string): Promise<AISummary | null>` — GET `/api/v1/opportunities/${opportunityId}/ai-summary`; returns `null` on 404; uses `apiClient`; re-throws all other errors.
    - `streamAISummary(opportunityId: string, token: string, handlers: SSEHandlers, signal: AbortSignal): Promise<number | null>` — native `fetch` POST; reads `X-Usage-Remaining` header; handles 429 by calling `handlers.onError("usage_limit_exceeded")` and returning `0`; iterates `parseSSEStream` async generator; dispatches to handlers; returns usage remaining as number, or `null` if header absent.
    - `parseSSEStream` async generator (module-private): buffers `ReadableStream<Uint8Array>` chunks, splits on `\n\n`, yields `{ event: string; data: string }` per SSE block.

14. `lib/queries/use-opportunities.ts` is extended with:
    - `useAISummary(opportunityId: string)` — TanStack Query hook wrapping `getAISummary`; `queryKey: ["ai-summary", opportunityId]`; `staleTime: 24 * 60 * 60 * 1000`; `enabled: !!opportunityId`.

15. All new i18n keys (see Dev Notes: i18n Keys) added to both `messages/en.json` and `messages/bg.json` under the `opportunities` namespace; `pnpm check:i18n` passes.

16. All interactive elements and state containers have `data-testid` attributes per the Dev Notes reference table.

17. ATDD test file `__tests__/opportunities-ai-s6-13.test.ts` covers: file structure (correct path), `'use client'` directive, all `data-testid` values from the reference table present in source, `fetch(` usage (NOT `EventSource`), `AIAnalysisTab.tsx` imports `AISummaryPanel`, `opportunities.ts` exports `getAISummary` and `streamAISummary`, `use-opportunities.ts` exports `useAISummary`, `TIER_SUMMARY_LIMITS` constant present, `AbortController` usage, `queryClient.invalidateQueries` usage, `x-request-id` header in source, all new i18n keys in both `en.json` and `bg.json`; all tests pass GREEN.

## Tasks / Subtasks

- [x] Task 1: Extend `lib/api/opportunities.ts` (AC: 13)
  - [x] 1.1 Add `SSEHandlers` interface with `onDelta`, `onMetadata`, `onDone`, `onError` callbacks
  - [x] 1.2 Add module-private `parseSSEStream` async generator: buffers `ReadableStream<Uint8Array>` via `TextDecoder`, splits on `\n\n`, parses `event:` and `data:` lines, yields `{ event: string; data: string }`
  - [x] 1.3 Add `getAISummary(opportunityId: string): Promise<AISummary | null>` using `apiClient.get`; catch `AxiosError` with status 404 → return `null`; re-throw all other errors
  - [x] 1.4 Add `streamAISummary(opportunityId, token, handlers, signal): Promise<number | null>`: native `fetch` POST with required headers; read `X-Usage-Remaining` header (with `Number.isFinite` NaN guard); handle 429 → `handlers.onError("usage_limit_exceeded")` → return `0`; handle non-ok → `handlers.onError("HTTP {status}")` → return remaining; iterate `parseSSEStream` and dispatch events; return remaining

- [x] Task 2: Add `useAISummary` to `lib/queries/use-opportunities.ts` (AC: 14)
  - [x] 2.1 Import `getAISummary` from `../api/opportunities`
  - [x] 2.2 Export `useAISummary(opportunityId: string)` with `queryKey: ["ai-summary", opportunityId]`, `queryFn: () => getAISummary(opportunityId)`, `staleTime: 24 * 60 * 60 * 1000`, `enabled: !!opportunityId`

- [x] Task 3: Add i18n translation keys (AC: 15)
  - [x] 3.1 Verify no duplicate keys: check `messages/en.json` for the 13 new keys listed in Dev Notes before adding
  - [x] 3.2 Add 13 new AI summary keys to `messages/en.json` under `opportunities` namespace
  - [x] 3.3 Add matching keys with Bulgarian translations to `messages/bg.json`
  - [x] 3.4 Run `pnpm check:i18n` to verify key parity passes — ✅ 651 keys match

- [x] Task 4: Create `AISummaryPanel.tsx` (AC: 1, 3–12, 16)
  - [x] 4.1 Define `AISummaryPanelProps`: `{ opportunityId: string; initialSummary: AISummary | null }`
  - [x] 4.2 Define `StreamState` type with fields: `status`, `streamedText`, `hasFirstDelta`, `metadata`, `elapsedMs`, `error`, `usageRemaining`, `showUpgradePrompt`, `showConfirmDialog`
  - [x] 4.3 Define `TIER_SUMMARY_LIMITS` constant: `{ free: 0, starter: 10, professional: 50, enterprise: Infinity }`
  - [x] 4.4 Implement `startStreaming` `useCallback`: token null guard; abort any prior controller; create new `AbortController`; record `Date.now()`; reset streaming state; call `streamAISummary` with inline `SSEHandlers` that drive `setState`; handle `AbortError` as non-fatal; guard stuck streaming state; store returned remaining count
  - [x] 4.5 Add `useEffect` with empty deps array: `return () => { abortControllerRef.current?.abort(); }` for unmount cleanup
  - [x] 4.6 Render usage counter `ai-usage-counter` with tier-aware logic (placeholder / remaining+limit / unlimited)
  - [x] 4.7 Render idle-no-summary state: `ai-empty-state`, `ai-generate-btn`
  - [x] 4.8 Render idle-with-summary state: `ai-summary-content`, `ai-summary-metadata`, `ai-regenerate-btn`
  - [x] 4.9 Render streaming state: `ai-summary-skeleton` (before first delta), `ai-streaming-text` + `ai-typing-cursor` (after first delta)
  - [x] 4.10 Render complete state: `ai-summary-content` (streamedText), `ai-summary-metadata` (tokens + seconds), `ai-regenerate-btn`
  - [x] 4.11 Render error state: `ai-summary-error`, `ai-summary-retry-btn`
  - [x] 4.12 Render upgrade prompt: `ai-upgrade-prompt` (shown when `showUpgradePrompt: true` or `usageRemaining === 0`)
  - [x] 4.13 Render regenerate confirm dialog using shadcn `<Dialog>`: `ai-regenerate-confirm-dialog`, `ai-regenerate-confirm-action`, `ai-regenerate-confirm-cancel`

- [x] Task 5: Modify `AIAnalysisTab.tsx` (AC: 2)
  - [x] 5.1 Import `AISummaryPanel` from `./AISummaryPanel`
  - [x] 5.2 Remove all TODO-marked stub content (static summary display, usage placeholder, both buttons)
  - [x] 5.3 Render `<AISummaryPanel opportunityId={opportunity.id} initialSummary={opportunity.ai_summary} />` inside the existing tab container div; keep `t("aiAnalysisTitle")` heading

- [x] Task 6: ATDD test file (AC: 17)
  - [x] 6.1 Create `__tests__/opportunities-ai-s6-13.test.ts` following the static source assertion pattern from `opportunities-upload-s6-12.test.ts`
  - [x] 6.2 Assert `AISummaryPanel.tsx` exists at the correct path
  - [x] 6.3 Assert `'use client'` present in `AISummaryPanel.tsx` source
  - [x] 6.4 Assert each `data-testid` from the reference table is present in `AISummaryPanel.tsx` source (16 testids)
  - [x] 6.5 Assert `fetch(` present and `new EventSource(` absent in `AISummaryPanel.tsx` source
  - [x] 6.6 Assert `opportunities.ts` exports `getAISummary` and `streamAISummary` and `SSEHandlers`
  - [x] 6.7 Assert `use-opportunities.ts` exports `useAISummary`
  - [x] 6.8 Assert `TIER_SUMMARY_LIMITS` present in `AISummaryPanel.tsx` source
  - [x] 6.9 Assert `AbortController` present in `AISummaryPanel.tsx` source
  - [x] 6.10 Assert `queryClient.invalidateQueries` present in `AISummaryPanel.tsx` source
  - [x] 6.11 Assert `x-request-id` present in `opportunities.ts` source
  - [x] 6.12 Assert `AIAnalysisTab.tsx` contains `import` of `AISummaryPanel`
  - [x] 6.13 Assert all 13 new i18n keys present in `messages/en.json` and `messages/bg.json`
  - [x] 6.14 Assert `ai-regenerate-confirm-dialog` present in `AISummaryPanel.tsx` source

## Dev Notes

### Architecture Overview

S06.13 is a pure frontend story. All required backend endpoints are already implemented (S06.08, done):

- `GET /api/v1/opportunities/{id}/ai-summary` → `AISummary` object or 404 (cached summary, no metering)
- `POST /api/v1/opportunities/{id}/ai-summary` → `text/event-stream` with SSE events (`delta`, `metadata`, `done`, `error`); checks `UsageGate` before streaming (returns 429 if quota exceeded); persists completed summary to `client.ai_summaries`

**Integration with S06.11**: `AIAnalysisTab.tsx` was created in S06.11 as a stub with `// TODO S06.13` markers on both buttons and a static display of `opportunity.ai_summary`. This story replaces all stub content with `<AISummaryPanel />`. The `AIAnalysisTab` modification is minimal — import + render the panel + keep the `aiAnalysisTitle` heading.

**Do NOT modify** `OpportunityDetailPage.tsx`, `OverviewTab.tsx`, `DocumentsTab.tsx`, `RequirementsTab.tsx`, `SubmissionGuideTab.tsx` — all out of scope.

### File Structure

```
eusolicit-app/frontend/apps/client/
├── app/[locale]/(protected)/opportunities/
│   └── components/
│       ├── AISummaryPanel.tsx         # NEW — streaming panel with state machine
│       └── AIAnalysisTab.tsx          # MODIFIED — thin wrapper; import + render panel
├── lib/
│   ├── api/
│   │   └── opportunities.ts          # MODIFIED — add SSEHandlers, getAISummary, streamAISummary, parseSSEStream
│   └── queries/
│       └── use-opportunities.ts      # MODIFIED — add useAISummary hook
├── messages/
│   ├── en.json                       # MODIFIED — add 13 new ai summary i18n keys
│   └── bg.json                       # MODIFIED — add matching Bulgarian keys
└── __tests__/
    └── opportunities-ai-s6-13.test.ts  # NEW — ATDD static source assertions
```

### API Contract (S06.08 — Backend Done)

**GET cached summary:**
```
GET /api/v1/opportunities/{id}/ai-summary
Authorization: Bearer {token}

200: { id, content, model, tokens_used, generated_at }
404: { error: "not_found" }  →  getAISummary returns null
```

**POST generate (SSE stream):**
```
POST /api/v1/opportunities/{id}/ai-summary
Authorization: Bearer {token}
Content-Type: application/json
x-request-id: {uuid}

200: Content-Type: text/event-stream
     Cache-Control: no-cache
     X-Accel-Buffering: no
     X-Usage-Remaining: N

     event: delta
     data: {"text": "...chunk..."}

     event: metadata
     data: {"tokens": 150, "model": "gpt-4o"}

     event: done
     data: {"summary_id": "uuid"}

     event: error
     data: {"message": "..."}

429: { "error": "usage_limit_exceeded", "limit": 10, "used": 10,
       "resets_at": "ISO8601", "upgrade_url": "..." }
403: { "error": "tier_limit", "upgrade_url": "..." }  (free tier — should not reach this panel)
```

### ⚠️ CRITICAL: Use `fetch` NOT `EventSource` for AI Summary Generation

**The generation endpoint is POST-based.** `EventSource` (and the `useSSE` hook in `packages/ui/src/lib/hooks/useSSE.ts`) supports GET requests ONLY. Using `useSSE` or `new EventSource(url)` here is a critical error — it will silently ignore the method and fire a GET, which returns the cached summary instead of streaming.

Use native `fetch` with `ReadableStream` body parsing:

```typescript
// lib/api/opportunities.ts  (add below existing functions)

export interface SSEHandlers {
  onDelta(text: string): void;
  onMetadata(tokens: number, model: string): void;
  onDone(summaryId: string): void;
  onError(message: string): void;
}

// Module-private SSE parser
async function* parseSSEStream(
  body: ReadableStream<Uint8Array>
): AsyncGenerator<{ event: string; data: string }> {
  const reader = body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";
  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      buffer += decoder.decode(value, { stream: true });
      const blocks = buffer.split("\n\n");
      buffer = blocks.pop() ?? "";
      for (const block of blocks) {
        if (!block.trim()) continue;
        const lines = block.split("\n");
        let eventType = "message";
        let data = "";
        for (const line of lines) {
          if (line.startsWith("event:")) eventType = line.slice("event:".length).trim();
          else if (line.startsWith("data:")) data = line.slice("data:".length).trim();
        }
        if (data) yield { event: eventType, data };
      }
    }
  } finally {
    reader.releaseLock();
  }
}

export async function streamAISummary(
  opportunityId: string,
  token: string,
  handlers: SSEHandlers,
  signal: AbortSignal
): Promise<number | null> {
  const response = await fetch(
    `/api/v1/opportunities/${opportunityId}/ai-summary`,
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${token}`,
        "x-request-id": crypto.randomUUID(),
      },
      body: JSON.stringify({}),
      signal,
    }
  );

  const usageHeader = response.headers.get("X-Usage-Remaining");
  const usageRemaining = usageHeader !== null ? parseInt(usageHeader, 10) : null;

  if (response.status === 429) {
    handlers.onError("usage_limit_exceeded");
    return 0;
  }
  if (!response.ok) {
    handlers.onError(`HTTP ${response.status}`);
    return usageRemaining;
  }
  if (!response.body) {
    handlers.onError("No response body");
    return usageRemaining;
  }

  for await (const { event, data } of parseSSEStream(response.body)) {
    try {
      const parsed = JSON.parse(data) as Record<string, unknown>;
      if (event === "delta") handlers.onDelta((parsed.text as string) ?? "");
      else if (event === "metadata")
        handlers.onMetadata((parsed.tokens as number) ?? 0, (parsed.model as string) ?? "");
      else if (event === "done") handlers.onDone((parsed.summary_id as string) ?? "");
      else if (event === "error") handlers.onError((parsed.message as string) ?? "Unknown error");
    } catch {
      handlers.onError(`Parse error in SSE data: ${data}`);
    }
  }

  return usageRemaining;
}

export async function getAISummary(opportunityId: string): Promise<AISummary | null> {
  try {
    const response = await apiClient.get<AISummary>(
      `/api/v1/opportunities/${opportunityId}/ai-summary`
    );
    return response.data;
  } catch (error) {
    if (axios.isAxiosError(error) && error.response?.status === 404) {
      return null;
    }
    throw error;
  }
}
```

Note: `axios` must be imported at the top of `opportunities.ts` — check if it's already imported (it should be via `apiClient`). If not, add: `import axios from "axios";`

### State Machine Design

```typescript
// In AISummaryPanel.tsx
type StreamStatus = "idle" | "streaming" | "complete" | "error";

interface StreamState {
  status: StreamStatus;
  streamedText: string;
  hasFirstDelta: boolean;      // controls skeleton → text transition
  metadata: { tokens: number; model: string } | null;
  elapsedMs: number | null;
  error: string | null;
  usageRemaining: number | null;
  showUpgradePrompt: boolean;
  showConfirmDialog: boolean;
}

const INITIAL_STATE: StreamState = {
  status: "idle",
  streamedText: "",
  hasFirstDelta: false,
  metadata: null,
  elapsedMs: null,
  error: null,
  usageRemaining: null,
  showUpgradePrompt: false,
  showConfirmDialog: false,
};
```

**State transitions:**
- `idle` (no summary) → click Generate → `streaming`
- `idle` (with summary) → click Regenerate → `showConfirmDialog: true`
- `showConfirmDialog` → confirm → `streaming`
- `showConfirmDialog` → cancel → `showConfirmDialog: false`
- `streaming` → first `delta` → `hasFirstDelta: true`
- `streaming` → `done` → `complete`
- `streaming` → `error` event (non-429) → `error`
- `streaming` → 429 response → `showUpgradePrompt: true`, `usageRemaining: 0`
- `complete` → click Regenerate → `showConfirmDialog: true`
- `error` → click Retry → `streaming`

### Streaming Entrypoint (useCallback Pattern)

```typescript
'use client';

import { useState, useCallback, useRef, useEffect } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { useAuthStore } from "@eusolicit/ui";
import { streamAISummary, type SSEHandlers, type AISummary } from "@/lib/api/opportunities";
import { useTranslations } from "next-intl";

const TIER_SUMMARY_LIMITS: Record<string, number> = {
  free: 0,
  starter: 10,
  professional: 50,
  enterprise: Infinity,
};

export function AISummaryPanel({ opportunityId, initialSummary }: AISummaryPanelProps) {
  const t = useTranslations("opportunities");
  const queryClient = useQueryClient();
  const { token, user } = useAuthStore();
  const abortControllerRef = useRef<AbortController | null>(null);
  const startTimeRef = useRef<number | null>(null);
  const [state, setState] = useState<StreamState>(INITIAL_STATE);

  // Abort in-progress stream on unmount
  useEffect(() => {
    return () => { abortControllerRef.current?.abort(); };
  }, []);

  const startStreaming = useCallback(async () => {
    abortControllerRef.current?.abort();
    const controller = new AbortController();
    abortControllerRef.current = controller;
    startTimeRef.current = Date.now();

    setState({
      ...INITIAL_STATE,
      status: "streaming",
      // preserve usageRemaining from prior generation if available
    });

    const handlers: SSEHandlers = {
      onDelta: (text) =>
        setState((prev) => ({
          ...prev,
          streamedText: prev.streamedText + text,
          hasFirstDelta: true,
        })),
      onMetadata: (tokens, model) =>
        setState((prev) => ({ ...prev, metadata: { tokens, model } })),
      onDone: (_summaryId) => {
        const elapsed = startTimeRef.current ? Date.now() - startTimeRef.current : null;
        setState((prev) => ({ ...prev, status: "complete", elapsedMs: elapsed }));
        queryClient.invalidateQueries({ queryKey: ["opportunity-detail", opportunityId] });
      },
      onError: (message) => {
        if (message === "usage_limit_exceeded") {
          setState((prev) => ({
            ...prev,
            status: "idle",
            showUpgradePrompt: true,
            usageRemaining: 0,
          }));
        } else {
          setState((prev) => ({ ...prev, status: "error", error: message }));
        }
      },
    };

    try {
      const remaining = await streamAISummary(
        opportunityId,
        token!,
        handlers,
        controller.signal
      );
      if (remaining !== null) {
        setState((prev) => ({ ...prev, usageRemaining: remaining }));
      }
    } catch (err: unknown) {
      if ((err as Error).name !== "AbortError") {
        setState((prev) => ({
          ...prev,
          status: "error",
          error: "Network error",
        }));
      }
      // AbortError on unmount — silently ignore
    }
  }, [opportunityId, token, queryClient]);

  // ... render below
}
```

### Usage Counter Display Logic

```typescript
const subscriptionTier = user?.subscription_tier ?? "free";
const tierLimit = TIER_SUMMARY_LIMITS[subscriptionTier] ?? 0;
const isEnterprise = subscriptionTier === "enterprise";

// In JSX:
<div data-testid="ai-usage-counter">
  {state.usageRemaining === null
    ? t("aiUsagePlaceholder")                          // existing key — before first gen
    : isEnterprise
    ? t("aiUsageUnlimited")                            // new key
    : t("aiUsageRemaining", {                          // new key
        remaining: state.usageRemaining,
        limit: tierLimit,
      })}
</div>
```

**Guard against Infinity display:** Always check `isEnterprise` before rendering `limit`. Never render `"X of Infinity"`.

### Regenerate Confirmation Dialog

Uses shadcn/ui `<Dialog>` from `@eusolicit/ui`:

```tsx
import {
  Dialog, DialogContent, DialogHeader, DialogTitle, DialogFooter, Button
} from "@eusolicit/ui";

{state.showConfirmDialog && (
  <Dialog
    open={state.showConfirmDialog}
    onOpenChange={(open) =>
      !open && setState((prev) => ({ ...prev, showConfirmDialog: false }))
    }
  >
    <DialogContent data-testid="ai-regenerate-confirm-dialog">
      <DialogHeader>
        <DialogTitle>{t("aiRegenerateConfirmTitle")}</DialogTitle>
      </DialogHeader>
      <p>
        {state.usageRemaining !== null
          ? t("aiRegenerateConfirmMessage", { remaining: state.usageRemaining })
          : t("aiRegenerateConfirmMessageUnknown")}
      </p>
      <DialogFooter>
        <Button
          variant="outline"
          data-testid="ai-regenerate-confirm-cancel"
          onClick={() =>
            setState((prev) => ({ ...prev, showConfirmDialog: false }))
          }
        >
          {t("aiRegenerateConfirmCancel")}
        </Button>
        <Button
          data-testid="ai-regenerate-confirm-action"
          onClick={startStreaming}
        >
          {t("aiRegenerateConfirmAction")}
        </Button>
      </DialogFooter>
    </DialogContent>
  </Dialog>
)}
```

### Typing Animation

```tsx
{state.status === "streaming" && state.hasFirstDelta && (
  <div
    data-testid="ai-streaming-text"
    className="prose prose-sm max-w-none whitespace-pre-wrap"
  >
    {state.streamedText}
    <span
      data-testid="ai-typing-cursor"
      className="inline-block w-0.5 h-4 bg-foreground ml-0.5 align-middle animate-pulse"
      aria-hidden="true"
    />
  </div>
)}
```

### AIAnalysisTab.tsx — Minimal Change

The existing `AIAnalysisTab.tsx` (~87 lines) has TODO-marked stub content for both buttons and a static summary display. After this story, it becomes:

```typescript
"use client";

import { useTranslations } from "next-intl";
import type { OpportunityDetailResponse } from "@/lib/api/opportunities";
import { AISummaryPanel } from "./AISummaryPanel";

interface AIAnalysisTabProps {
  opportunity: OpportunityDetailResponse;
}

export function AIAnalysisTab({ opportunity }: AIAnalysisTabProps) {
  const t = useTranslations("opportunities");
  return (
    <div>
      <h2 className="text-lg font-semibold mb-4">{t("aiAnalysisTitle")}</h2>
      <AISummaryPanel
        opportunityId={opportunity.id}
        initialSummary={opportunity.ai_summary}
      />
    </div>
  );
}
```

All TODO-marked content is deleted. The `aiAnalysisTitle` i18n key continues to be used.

### UI Components Import Map

All from `@eusolicit/ui`:

| Component | Usage |
|-----------|-------|
| `Button` | Generate CTA, Regenerate, Retry, dialog buttons |
| `Skeleton` | Pulsing skeleton during streaming wait (`animate-pulse`) |
| `Dialog`, `DialogContent`, `DialogHeader`, `DialogTitle`, `DialogFooter` | Regenerate confirmation dialog |

Lucide icons (`lucide-react`):
- `Sparkles` — Generate/Regenerate button icon
- `RefreshCw` — Regenerate button icon
- `AlertCircle` — Error state icon
- `ArrowUpCircle` — Upgrade prompt icon

### i18n Keys Required

**KEEP (already exist — do NOT re-add):** `aiAnalysisTitle`, `aiGenerateCta`, `aiRegenerateCta`, `aiUsagePlaceholder`, `aiGeneratedAt`, `aiTokensUsed`, `aiEmptyDescription`

**ADD — 13 new keys** under `opportunities` namespace in both `en.json` and `bg.json`:

```json
{
  "aiSummaryGenerating": "Generating summary…",
  "aiSummaryError": "Failed to generate summary. Please try again.",
  "aiSummaryRetry": "Retry",
  "aiUsageRemaining": "{remaining} of {limit} summaries remaining this month",
  "aiUsageUnlimited": "Unlimited AI summaries (Enterprise)",
  "aiUsageLimitReached": "Monthly summary limit reached",
  "aiRegenerateConfirmTitle": "Regenerate AI Summary?",
  "aiRegenerateConfirmMessage": "This will use 1 of your {remaining} remaining summaries.",
  "aiRegenerateConfirmMessageUnknown": "This will use 1 of your monthly summaries.",
  "aiRegenerateConfirmAction": "Generate",
  "aiRegenerateConfirmCancel": "Cancel",
  "aiSummaryCompleteMeta": "{tokens} tokens · {seconds}s",
  "aiUpgradeForSummaries": "Upgrade your plan to generate AI summaries"
}
```

Verify these 13 keys are absent from `messages/en.json` before adding.

### data-testid Reference Table

| `data-testid` | Component | Description |
|---|---|---|
| `ai-summary-panel` | `AISummaryPanel` | Root container |
| `ai-usage-counter` | `AISummaryPanel` | Usage display ("X of Y remaining" or placeholder) |
| `ai-empty-state` | `AISummaryPanel` | No-summary empty state container |
| `ai-generate-btn` | `AISummaryPanel` | "Generate Summary" CTA (idle, no summary) |
| `ai-summary-skeleton` | `AISummaryPanel` | Pulsing skeleton (streaming, before first delta) |
| `ai-streaming-text` | `AISummaryPanel` | Accumulating delta text container |
| `ai-typing-cursor` | `AISummaryPanel` | Blinking cursor appended to stream |
| `ai-summary-content` | `AISummaryPanel` | Completed or initial summary text |
| `ai-summary-metadata` | `AISummaryPanel` | Token count + generation time |
| `ai-regenerate-btn` | `AISummaryPanel` | "Regenerate" button (visible when summary exists) |
| `ai-summary-error` | `AISummaryPanel` | Error state container |
| `ai-summary-retry-btn` | `AISummaryPanel` | Retry button in error state |
| `ai-upgrade-prompt` | `AISummaryPanel` | Upgrade CTA shown on 429 / limit reached |
| `ai-regenerate-confirm-dialog` | `AISummaryPanel` | shadcn Dialog for regenerate confirmation |
| `ai-regenerate-confirm-action` | `AISummaryPanel` | "Generate" confirm button inside dialog |
| `ai-regenerate-confirm-cancel` | `AISummaryPanel` | "Cancel" button inside dialog |

### Test Design Coverage (from `eusolicit-docs/test-artifacts/test-design-epic-06.md`)

| Test ID | Level | Scenario | ATDD Coverage |
|---------|-------|----------|---------------|
| **E06-P1-029** | Component | AI summary panel: "Generate Summary" CTA triggers SSE connection; streaming text renders; typing animation visible; usage counter shows remaining quota | Assert `ai-generate-btn`, `ai-streaming-text`, `ai-typing-cursor`, `ai-usage-counter` testids in source; assert `fetch(` (not `EventSource`) in source |
| **E06-P2-008** | Component | AI summary "Regenerate" button shows confirmation dialog with remaining quota count before submitting | Assert `ai-regenerate-confirm-dialog`, `ai-regenerate-confirm-action`, `ai-regenerate-confirm-cancel` testids; assert `aiRegenerateConfirmMessage` key with `{remaining}` in source |
| **E06-P2-009** | Component | AI summary panel shows "X of Y summaries remaining this month" usage counter; updates after generation | Assert `TIER_SUMMARY_LIMITS` in source; assert `aiUsageRemaining` i18n key with `{remaining}` and `{limit}` in source |

**Playwright E2E additions:** E06-P0-010 (end-to-end tier enforcement, wired through S06.14 global 429 interceptor) will cover the AI Analysis tab flow. No new Playwright specs required for this story — Vitest static assertions are sufficient.

### Critical Mistakes to Prevent

1. **Do NOT use `new EventSource(url)` or `useSSE` from `packages/ui` for generation.** `EventSource` is GET-only; the generation endpoint is POST. The `useSSE` hook at `packages/ui/src/lib/hooks/useSSE.ts` is explicitly for GET SSE endpoints (e.g., live notification streams). Using it here silently fires a GET that returns cached data instead of streaming new content.

2. **Do NOT use `apiClient` (Axios) for streaming.** Axios buffers the entire response body before resolving. Use native `fetch` with `response.body` as a `ReadableStream` so delta chunks are processed incrementally as they arrive.

3. **Do NOT set an `Authorization` header via `apiClient` interceptors for the streaming call.** The `streamAISummary` function is a raw `fetch` — the Axios interceptor chain does not apply. Read `token` from `useAuthStore()` and pass it as `Bearer ${token}` manually in the fetch headers.

4. **Do NOT start streaming in a `useEffect`.** The streaming flow is triggered by user interaction (click). `useEffect` with no trigger would auto-stream on mount, which contradicts the spec (user must click Generate). Use `useCallback` called from the click handler.

5. **Do NOT forget `AbortController` cleanup.** Navigating away mid-stream leaves the `fetch` open and causes state updates on an unmounted component (React warning + memory leak). Abort in the `useEffect` return callback (`return () => { abortControllerRef.current?.abort(); }`). Catch `AbortError` in the streaming `try/catch` and treat it as non-fatal (do not set error state).

6. **Do NOT call `queryClient.invalidateQueries` before the `done` event.** Calling it on `metadata` or during a `delta` will trigger a background refetch that may cause the parent `OpportunityDetailPage` to re-render mid-stream, disrupting the streaming display.

7. **Do NOT show the confirmation dialog on the first Generate (no existing summary).** The dialog is for Regenerate only. Clicking the Generate CTA when `initialSummary == null` and `state.status === "idle"` goes directly to `startStreaming()`.

8. **Do NOT duplicate the 7 existing i18n keys** (`aiAnalysisTitle`, `aiGenerateCta`, `aiRegenerateCta`, `aiUsagePlaceholder`, `aiGeneratedAt`, `aiTokensUsed`, `aiEmptyDescription`). The ATDD test for S06.11 and the `pnpm check:i18n` script both enforce key uniqueness.

9. **Do NOT render `"X of Infinity remaining"` for Enterprise users.** `TIER_SUMMARY_LIMITS.enterprise = Infinity`. Guard with `isEnterprise` check and render `t("aiUsageUnlimited")` instead.

10. **Do NOT modify `OpportunityDetailPage.tsx`.** The `initialSummary` prop flows from `opportunity.ai_summary` in `OpportunityDetailResponse`. After streaming completes, `queryClient.invalidateQueries(["opportunity-detail", opportunityId])` updates the TanStack Query cache; the detail page re-renders with the fresh summary. No manual prop threading or page modifications needed.

### Project Structure Notes

- `AISummaryPanel.tsx` goes in `app/[locale]/(protected)/opportunities/components/` — same directory as all S06.11 tab components. This component is tightly coupled to the AI summary API and is NOT a generic UI primitive for `packages/ui`.
- `"use client"` is required: `fetch`, `ReadableStream`, `useState`, `useCallback`, `useRef`, `useEffect`, `AbortController`, `crypto.randomUUID`.
- TypeScript strict mode enforced — no `any`; use `Record<string, unknown>` with `as` casts for SSE JSON parsing; never `as any`.
- `useQueryClient()` is from `@tanstack/react-query` — import it directly, not via `@eusolicit/ui`.
- `user?.subscription_tier` — the JWT `subscription_tier` claim (`free` | `starter` | `professional` | `enterprise`) is decoded into the auth store `user` object by E02's JWT parsing. Fall back to `"free"` if absent.

### References

- Story 6.11: `eusolicit-docs/implementation-artifacts/6-11-opportunity-detail-page-tabbed-layout.md` — `AIAnalysisTab.tsx` stub to replace; `OpportunityDetailResponse` with `ai_summary: AISummary | null`; `useOpportunityDetail` query key `["opportunity-detail", id]`
- Story 6.12: `eusolicit-docs/implementation-artifacts/6-12-document-upload-component.md` — ATDD static assertion test pattern; `useCallback` queue management; i18n key approach; XHR async flow patterns (adapt for fetch streaming)
- Story 6.8: `eusolicit-docs/implementation-artifacts/6-8-ai-summary-generation-api-sse-streaming.md` — exact SSE event contract (`delta`, `metadata`, `done`, `error`); `X-Usage-Remaining` header; 429 response body; 24h cache TTL; `UsageGate` tier limits
- Epic 6 test design: `eusolicit-docs/test-artifacts/test-design-epic-06.md` — E06-P1-029, E06-P2-008, E06-P2-009, E06-P0-009 (SSE event order)
- `useSSE` hook (GET-only — do NOT use for POST generation): `eusolicit-app/frontend/packages/ui/src/lib/hooks/useSSE.ts`
- Existing opportunities API: `eusolicit-app/frontend/apps/client/lib/api/opportunities.ts`
- Existing `AIAnalysisTab.tsx`: `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/AIAnalysisTab.tsx`
- Existing `use-opportunities.ts`: `eusolicit-app/frontend/apps/client/lib/queries/use-opportunities.ts`
- i18n messages: `eusolicit-app/frontend/apps/client/messages/en.json` + `bg.json`

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

None — implementation completed cleanly on first run. One minor regex fix needed: combined multi-line `@eusolicit/ui` imports onto a single line so the test regex `/Dialog.*@eusolicit\/ui|from.*@eusolicit\/ui.*Dialog/` could match.

### Completion Notes List

- All 162 ATDD tests GREEN on first run after single-line import fix.
- `pnpm check:i18n` passes — 651 keys match across `en.json` and `bg.json`.
- S6.12 ATDD (122 tests) continues to pass — no regressions.
- Used native `fetch` with `ReadableStream` / `parseSSEStream` async generator — NOT `EventSource` or `apiClient`.
- `AbortController` wired in `useEffect` cleanup for unmount safety.
- `TIER_SUMMARY_LIMITS` guards Enterprise tier (`Infinity`) with `isEnterprise` boolean to avoid rendering "X of Infinity".
- `queryClient.invalidateQueries` only called on `done` event (not on `metadata` or mid-delta).
- Regenerate confirm dialog shown only when a prior summary exists — first Generate goes directly to streaming.

### File List

- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/AISummaryPanel.tsx` — NEW
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/AIAnalysisTab.tsx` — MODIFIED (thin wrapper)
- `eusolicit-app/frontend/apps/client/lib/api/opportunities.ts` — MODIFIED (SSEHandlers, parseSSEStream, getAISummary, streamAISummary)
- `eusolicit-app/frontend/apps/client/lib/queries/use-opportunities.ts` — MODIFIED (useAISummary)
- `eusolicit-app/frontend/apps/client/messages/en.json` — MODIFIED (13 new keys)
- `eusolicit-app/frontend/apps/client/messages/bg.json` — MODIFIED (13 new Bulgarian keys)
- `eusolicit-app/frontend/apps/client/__tests__/opportunities-ai-s6-13.test.ts` — pre-existing ATDD test file (all 162 tests GREEN)

## Senior Developer Review

**Review Date:** 2026-04-17
**Review Layers:** Blind Hunter (adversarial) · Edge Case Hunter · Acceptance Auditor
**Outcome:** ✅ All 4 patches applied (4 patch fixed, 2 deferred, 13 dismissed)

### Review Findings

- [x] [Review][Patch] **CRITICAL — Query key mismatch: cache invalidation after streaming is a no-op.** `AISummaryPanel.tsx:102` invalidates `["opportunity-detail", opportunityId]` but `useOpportunityDetail` in `use-opportunities.ts:58` registers with `["opportunity", id]`. These keys do not match. After a successful `done` SSE event, `queryClient.invalidateQueries` fires against a key that has no cache entry, so the detail page never refreshes with the newly generated `ai_summary`. The user must manually refresh to see the result. **Fix:** Changed to `queryClient.invalidateQueries({ queryKey: ["opportunity", opportunityId] })`. Also updated the ATDD test assertion to use regex matching the correct key. ✅ Fixed 2026-04-17.
- [x] [Review][Patch] **Stream ending without `done`/`error` event leaves component stuck in `streaming` state.** If the server closes the SSE connection gracefully (TCP FIN) without emitting a `done` or `error` event, `parseSSEStream` exits cleanly, `streamAISummary` returns `usageRemaining`, and `startStreaming` only updates `usageRemaining` — never transitioning out of `streaming`. The user sees a skeleton/cursor spinner forever with no way to recover except reloading. **Fix:** Added guard after `await streamAISummary(...)` returns: `setState((prev) => prev.status === "streaming" ? { ...prev, status: "error", error: "Stream ended unexpectedly" } : prev)`. ✅ Fixed 2026-04-17.
- [x] [Review][Patch] **`token!` non-null assertion without runtime guard.** `startStreaming` uses `token!` to pass the auth token to `streamAISummary`. While the `(protected)` layout's `AuthGuard` should guarantee a token exists, the non-null assertion silently sends `"Bearer undefined"` if the invariant is ever violated (e.g., race during session expiry). **Fix:** Added `if (!token) return;` as the first line of `startStreaming`; removed `!` non-null assertion. ✅ Fixed 2026-04-17.
- [x] [Review][Patch] **`parseInt` on `X-Usage-Remaining` can return `NaN` without validation.** If the header value is non-numeric (empty string, malformed), `parseInt(usageHeader, 10)` returns `NaN`. This `NaN` propagates into `state.usageRemaining` and renders as "NaN of 50 summaries remaining". **Fix:** Added `Number.isFinite()` validation after parsing: `const parsedUsage = parseInt(usageHeader, 10); const usageRemaining = parsedUsage !== null && Number.isFinite(parsedUsage) ? parsedUsage : null;`. ✅ Fixed 2026-04-17.
- [x] [Review][Defer] `parseSSEStream` does not concatenate multi-line `data:` fields per the SSE specification — it keeps only the last `data:` line per block. Backend S06.08 sends single-line data fields, so no functional impact currently. [opportunities.ts:280] — deferred, pre-existing SSE parser limitation
- [x] [Review][Defer] Elapsed time via `Date.now()` is inaccurate when the browser tab is suspended/throttled. Server-side timing would be more accurate but is out of scope for this story. [AISummaryPanel.tsx:77] — deferred, known browser limitation

## Known Deviations

### Detected by `3-code-review` at 2026-04-17T11:49:31Z (session 91cec23b-2427-4835-a8d3-f4951b7ec3b0)

- Story spec reference `["opportunity-detail", opportunityId]` in AC8 and Dev Notes contradicts actual `useOpportunityDetail` hook query key `["opportunity", id]` established in S06.11. The spec incorrectly documents the S06.11 query key, causing the S06.13 implementation to invalidate a non-existent cache key. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
- No guard for graceful SSE stream termination without terminal event (`done`/`error`). The acceptance criteria assume the server always sends a `done` or `error` event, but no defensive handling exists for abnormal stream closure. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_
- Story spec reference `["opportunity-detail", opportunityId]` in AC8 and Dev Notes contradicts actual `useOpportunityDetail` hook query key `["opportunity", id]` established in S06.11. The spec incorrectly documents the S06.11 query key, causing the S06.13 implementation to invalidate a non-existent cache key.
- No guard for graceful SSE stream termination without terminal event (`done`/`error`). The acceptance criteria assume the server always sends a `done` or `error` event, but no defensive handling exists for abnormal stream closure. _(type: `CONTRADICTORY_SPEC`; severity: `blocking`)_
