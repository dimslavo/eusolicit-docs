---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-17'
workflowType: 'testarch-atdd'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/6-13-ai-summary-panel-with-sse-streaming.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-06.md'
  - 'eusolicit-app/frontend/apps/client/__tests__/opportunities-upload-s6-12.test.ts'
  - 'eusolicit/_bmad/bmm/config.yaml'
---

# ATDD Checklist — Epic 6, Story 13: AI Summary Panel with SSE Streaming

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**Primary Test Level:** Static source assertion (Vitest, `environment: 'node'`)
**TDD Phase:** 🔴 RED — All 149 structural tests fail until Story 6.13 is implemented

---

## Story Summary

Paid-tier users viewing the AI Analysis tab need a real-time AI summary panel that checks for
cached summaries on mount, streams a new executive summary via SSE when they click "Generate"
(with typing animation and live progress), shows their remaining monthly quota, and surfaces
an upgrade prompt when the limit is reached.

**As a** paid-tier user viewing the AI Analysis tab on the opportunity detail page
**I want** an AI summary panel with SSE streaming, typing animation, usage quota display, and
upgrade prompts
**So that** I can view AI-powered opportunity intelligence in real time without leaving the detail
page, see progress as it streams, and understand my plan's usage limits

---

## Stack Detection

**Detected stack:** `fullstack` (Next.js frontend + FastAPI backend)
**Story scope:** Pure frontend (backend endpoints already implemented in S06.08)
**Generation mode:** AI generation — static source assertions (same pattern as S06.12)
**Recording:** Not required — no live browser session needed for structural assertions

---

## Acceptance Criteria

1. `AISummaryPanel.tsx` created at correct path with `'use client'`, named export
2. `AIAnalysisTab.tsx` imports/renders `<AISummaryPanel>`, removes all TODO stubs
3. `initialSummary != null` → renders `ai-summary-content`, `ai-summary-metadata`, `ai-regenerate-btn`
4. `initialSummary == null` → renders `ai-empty-state`, `ai-generate-btn`
5. "Generate" click → skeleton → streaming text + typing cursor; `hasFirstDelta` controls transition
6. Regenerate click → confirmation dialog with `ai-regenerate-confirm-dialog` and quota display
7. Uses native `fetch` (NOT `EventSource`); `AbortController` + `useEffect` cleanup; `x-request-id` header
8. SSE events: `delta`, `metadata`, `done`, `error`; `done` → `queryClient.invalidateQueries`
9. After `done` → `ai-summary-content` (streamed text) + `ai-summary-metadata` (tokens + seconds)
10. `ai-usage-counter` with `TIER_SUMMARY_LIMITS` constant; Enterprise shows `aiUsageUnlimited`
11. HTTP 429 → `ai-upgrade-prompt`; `usageRemaining: 0`; NOT error state
12. Network/SSE error → `ai-summary-error` + `ai-summary-retry-btn`; AbortError is non-fatal
13. `opportunities.ts` exports: `SSEHandlers`, `getAISummary`, `streamAISummary`, `parseSSEStream`
14. `use-opportunities.ts` exports `useAISummary` with correct queryKey, staleTime, enabled guard
15. All 13 new i18n keys in both `en.json` and `bg.json` under `opportunities` namespace
16. All 16 `data-testid` attributes from the reference table present in `AISummaryPanel.tsx`
17. ATDD test file `__tests__/opportunities-ai-s6-13.test.ts` covers all of the above

---

## Failing Tests Created (RED Phase)

**File:** `eusolicit-app/frontend/apps/client/__tests__/opportunities-ai-s6-13.test.ts`
**Total tests:** 162 | **Failing (RED):** 149 | **Passing:** 13
**Run:** `cd eusolicit-app/frontend/apps/client && ./node_modules/.bin/vitest run __tests__/opportunities-ai-s6-13.test.ts`

### AC17: ATDD Self-Reference (1 test)

- ✅ **Test:** `opportunities-ai-s6-13.test.ts exists at the correct path`
  - **Status:** GREEN ✅ (file already saved — self-referential)
  - **Verifies:** AC17 — test file created at the correct path

### AC1: File Structure — AISummaryPanel.tsx (4 tests)

- 🔴 **Test:** `AISummaryPanel.tsx exists at app/[locale]/(protected)/opportunities/components/`
  - **Status:** RED — ENOENT, file not yet created
  - **Verifies:** AC1 — correct file path

- 🔴 **Test:** `is a client component — has 'use client' directive at the top`
  - **Status:** RED — ENOENT
  - **Verifies:** AC1 — `'use client'` required for browser APIs

- 🔴 **Test:** `exports AISummaryPanel as a named export`
  - **Status:** RED — ENOENT
  - **Verifies:** AC1 — named export for use by `AIAnalysisTab.tsx`

- 🔴 **Test:** `defines AISummaryPanelProps with opportunityId and initialSummary props`
  - **Status:** RED — ENOENT
  - **Verifies:** AC1 — correct prop interface shape

### AC2: AIAnalysisTab.tsx Wiring (7 tests)

- ✅ **Test:** `AIAnalysisTab.tsx exists`
  - **Status:** GREEN ✅ (file exists from S06.11)
  - **Verifies:** prerequisite file present

- 🔴 **Test:** `imports AISummaryPanel from ./AISummaryPanel`
  - **Status:** RED — import not yet present
  - **Verifies:** AC2 — thin wrapper pattern

- 🔴 **Test:** `renders <AISummaryPanel> component`
  - **Status:** RED — not yet rendered
  - **Verifies:** AC2 — panel delegation

- 🔴 **Test:** `passes opportunityId={opportunity.id} to AISummaryPanel`
  - **Status:** RED — prop not yet passed
  - **Verifies:** AC2 — correct prop wiring

- 🔴 **Test:** `passes initialSummary={opportunity.ai_summary} to AISummaryPanel`
  - **Status:** RED — prop not yet passed
  - **Verifies:** AC2 — initial summary hydration

- 🔴 **Test:** `still renders aiAnalysisTitle heading`
  - **Status:** RED — modification not yet made (actually may pass if key already there)
  - **Verifies:** AC2 — heading preserved during modification

- 🔴 **Test:** `does NOT contain TODO-marked stub content`
  - **Status:** RED — TODO stubs still present from S06.11
  - **Verifies:** AC2 — all stub content removed

### AC3: Initial Summary State (4 tests)

- 🔴 **Test:** `has data-testid="ai-summary-content" for summary text display`
  - **Status:** RED — AISummaryPanel.tsx doesn't exist
  - **Verifies:** AC3, AC16 — testid present in source

- 🔴 **Test:** `has data-testid="ai-summary-metadata" for token count and generation time`
  - **Status:** RED — ENOENT
  - **Verifies:** AC3, AC16 — metadata display

- 🔴 **Test:** `has data-testid="ai-regenerate-btn" for the Regenerate button`
  - **Status:** RED — ENOENT
  - **Verifies:** AC3, AC16 — regenerate button

- 🔴 **Test:** `references t("aiTokensUsed") for initial metadata display`
  - **Status:** RED — ENOENT
  - **Verifies:** AC3 — existing i18n key usage

### AC4: Empty State (3 tests)

- 🔴 **Test:** `has data-testid="ai-empty-state" container for the no-summary state`
  - **Status:** RED — ENOENT
  - **Verifies:** AC4, AC16

- 🔴 **Test:** `has data-testid="ai-generate-btn" for the Generate Summary CTA`
  - **Status:** RED — ENOENT
  - **Verifies:** AC4, AC16

- 🔴 **Test:** `references t("aiEmptyDescription") for empty state description`
  - **Status:** RED — ENOENT
  - **Verifies:** AC4 — existing i18n key usage

### AC5: Streaming State (8 tests)

- 🔴 **Test:** `has data-testid="ai-summary-skeleton" for the pulsing skeleton before first delta`
  - **Status:** RED — ENOENT
  - **Verifies:** AC5, AC16

- 🔴 **Test:** `has data-testid="ai-streaming-text" container for accumulating delta text`
  - **Status:** RED — ENOENT
  - **Verifies:** AC5, AC16

- 🔴 **Test:** `has data-testid="ai-typing-cursor" for the blinking cursor indicator`
  - **Status:** RED — ENOENT
  - **Verifies:** AC5, AC16

- 🔴 **Test:** `typing cursor has aria-hidden="true"`
  - **Status:** RED — ENOENT
  - **Verifies:** AC5 — accessibility (decorative element)

- 🔴 **Test:** `manages hasFirstDelta state to control skeleton → text transition`
  - **Status:** RED — ENOENT
  - **Verifies:** AC5 — state machine fidelity

- 🔴 **Test:** `records generation start time with Date.now()`
  - **Status:** RED — ENOENT
  - **Verifies:** AC5 — elapsed time for metadata display

- 🔴 **Test:** `defines StreamStatus type with "idle" | "streaming" | "complete" | "error" values`
  - **Status:** RED — ENOENT
  - **Verifies:** AC5 — state machine type safety

- 🔴 **Test:** `manages streamedText state that accumulates delta chunks`
  - **Status:** RED — ENOENT
  - **Verifies:** AC5 — streaming accumulation

### AC6: Regenerate Confirmation Dialog (8 tests)

- 🔴 **Test:** `has data-testid="ai-regenerate-confirm-dialog" on the Dialog container`
  - **Status:** RED — ENOENT
  - **Verifies:** AC6, AC16

- 🔴 **Test:** `has data-testid="ai-regenerate-confirm-action" on the confirm Generate button`
  - **Status:** RED — ENOENT
  - **Verifies:** AC6, AC16

- 🔴 **Test:** `has data-testid="ai-regenerate-confirm-cancel" on the Cancel button`
  - **Status:** RED — ENOENT
  - **Verifies:** AC6, AC16

- 🔴 **Test:** `references t("aiRegenerateConfirmTitle") as the dialog title`
  - **Status:** RED — ENOENT
  - **Verifies:** AC6 — i18n key usage

- 🔴 **Test:** `references t("aiRegenerateConfirmMessage") with {remaining} param`
  - **Status:** RED — ENOENT
  - **Verifies:** AC6 — quota-aware confirm message

- 🔴 **Test:** `references t("aiRegenerateConfirmMessageUnknown") as fallback`
  - **Status:** RED — ENOENT
  - **Verifies:** AC6 — null usageRemaining fallback

- 🔴 **Test:** `manages showConfirmDialog state for dialog open/close`
  - **Status:** RED — ENOENT
  - **Verifies:** AC6 — dialog state management

- 🔴 **Test:** `uses shadcn Dialog component from @eusolicit/ui`
  - **Status:** RED — ENOENT
  - **Verifies:** AC6 — correct UI library

### AC7 (P0): fetch NOT EventSource; AbortController; x-request-id (11 tests)

- 🔴 **Test:** `AISummaryPanel.tsx uses fetch() for streaming (not EventSource)`
  - **Status:** RED — ENOENT
  - **Verifies:** AC7 — critical: POST-based SSE requires native fetch

- 🔴 **Test:** `AISummaryPanel.tsx does NOT use new EventSource`
  - **Status:** RED — ENOENT (negative assertion — would fail if wrongly present)
  - **Verifies:** AC7 — critical mistake prevention

- 🔴 **Test:** `AISummaryPanel.tsx does NOT import useSSE hook`
  - **Status:** RED — ENOENT
  - **Verifies:** AC7 — GET-only hook must not be used for POST SSE

- 🔴 **Test:** `opportunities.ts uses native fetch() for the POST streaming call`
  - **Status:** RED — function not yet added to opportunities.ts
  - **Verifies:** AC7, AC13 — native fetch in API layer

- 🔴 **Test:** `opportunities.ts does NOT use new EventSource for the POST streaming call`
  - **Status:** RED — negative assertion
  - **Verifies:** AC7, AC13 — critical mistake prevention

- 🔴 **Test:** `AISummaryPanel.tsx uses AbortController for stream cancellation`
  - **Status:** RED — ENOENT
  - **Verifies:** AC7 — memory leak prevention

- 🔴 **Test:** `AISummaryPanel.tsx aborts controller in useEffect cleanup on unmount`
  - **Status:** RED — ENOENT
  - **Verifies:** AC7 — unmount cleanup prevents state updates on unmounted component

- 🔴 **Test:** `AISummaryPanel.tsx stores AbortController in a ref (abortControllerRef)`
  - **Status:** RED — ENOENT
  - **Verifies:** AC7 — ref-based controller for stable reference across renders

- 🔴 **Test:** `opportunities.ts sets x-request-id header with crypto.randomUUID()`
  - **Status:** RED — function not yet added
  - **Verifies:** AC7 — request correlation header

- 🔴 **Test:** `opportunities.ts sets Authorization: Bearer header from token param`
  - **Status:** RED — function not yet added
  - **Verifies:** AC7 — auth header (note: Axios interceptors don't apply to raw fetch)

- 🔴 **Test:** `AISummaryPanel.tsx reads token from useAuthStore()`
  - **Status:** RED — ENOENT
  - **Verifies:** AC7 — token source

- 🔴 **Test:** `AISummaryPanel.tsx uses useCallback for startStreaming`
  - **Status:** RED — ENOENT
  - **Verifies:** AC7 — click-triggered, not effect-triggered streaming

### AC8: SSE Event Dispatch and Cache Invalidation (11 tests)

- 🔴 **Test:** `opportunities.ts SSEHandlers interface has onDelta/onMetadata/onDone/onError callbacks`
  - **Status:** RED — SSEHandlers not yet exported from opportunities.ts
  - **Verifies:** AC8, AC13 — complete handler interface

- 🔴 **Test:** `AISummaryPanel.tsx calls queryClient.invalidateQueries on done event`
  - **Status:** RED — ENOENT
  - **Verifies:** AC8 — cache refresh after successful generation

- 🔴 **Test:** `queryClient.invalidateQueries uses ["opportunity-detail", opportunityId] queryKey`
  - **Status:** RED — ENOENT
  - **Verifies:** AC8 — correct query key (from S06.11)

- 🔴 **Test:** `AISummaryPanel.tsx imports useQueryClient from @tanstack/react-query`
  - **Status:** RED — ENOENT
  - **Verifies:** AC8 — correct import (not via @eusolicit/ui)

- 🔴 **Test:** `parseSSEStream async generator present in opportunities.ts`
  - **Status:** RED — not yet implemented
  - **Verifies:** AC8, AC13 — SSE stream parser

- 🔴 **Test:** `parseSSEStream splits on "\\n\\n" double-newline SSE block separator`
  - **Status:** RED — not yet implemented
  - **Verifies:** AC8, AC13 — correct SSE framing

- 🔴 **Test:** `delta event handler appends text to streamedText and sets hasFirstDelta`
  - **Status:** RED — ENOENT
  - **Verifies:** AC8 — delta accumulation

- 🔴 **Test:** `done event handler transitions status to "complete" and records elapsed time`
  - **Status:** RED — ENOENT
  - **Verifies:** AC8, AC9 — completion state

- 🔴 **Test:** `error event handler (non-429) transitions status to "error"`
  - **Status:** RED — ENOENT
  - **Verifies:** AC8, AC12 — error state transition

### AC9: Complete State (3 tests)

- 🔴 **Test:** `references t("aiSummaryCompleteMeta") with tokens and seconds params`
  - **Status:** RED — ENOENT
  - **Verifies:** AC9 — post-stream metadata display

- 🔴 **Test:** `stores metadata state with tokens and model fields`
  - **Status:** RED — ENOENT
  - **Verifies:** AC9 — metadata state shape

- 🔴 **Test:** `calculates elapsed seconds from startTimeRef using Date.now()`
  - **Status:** RED — ENOENT
  - **Verifies:** AC9 — elapsed time calculation

### AC10: Usage Counter and TIER_SUMMARY_LIMITS (11 tests)

- 🔴 **Test:** `has data-testid="ai-usage-counter" on the usage display element`
  - **Status:** RED — ENOENT
  - **Verifies:** AC10, AC16

- 🔴 **Test:** `defines TIER_SUMMARY_LIMITS constant with all four tier keys`
  - **Status:** RED — ENOENT
  - **Verifies:** AC10 — tier limit lookup table

- 🔴 **Test:** `TIER_SUMMARY_LIMITS.free = 0 / .starter = 10 / .professional = 50 / .enterprise = Infinity`
  - **Status:** RED — ENOENT
  - **Verifies:** AC10 — correct tier values

- 🔴 **Test:** `reads subscriptionTier from useAuthStore() → user?.subscription_tier`
  - **Status:** RED — ENOENT
  - **Verifies:** AC10 — auth store integration

- 🔴 **Test:** `defaults subscriptionTier to "free" when absent`
  - **Status:** RED — ENOENT
  - **Verifies:** AC10 — safe fallback

- 🔴 **Test:** `reads X-Usage-Remaining response header after streaming`
  - **Status:** RED — streamAISummary not yet in opportunities.ts
  - **Verifies:** AC10 — header-based quota tracking

- 🔴 **Test:** `references t("aiUsageRemaining"), t("aiUsageUnlimited"), t("aiUsagePlaceholder")`
  - **Status:** RED — ENOENT
  - **Verifies:** AC10 — all three usage display states

- 🔴 **Test:** `guards against rendering Infinity for enterprise`
  - **Status:** RED — ENOENT
  - **Verifies:** AC10 — critical: `isEnterprise` check prevents "X of Infinity" display

### AC11: HTTP 429 Upgrade Prompt (7 tests)

- 🔴 **Test:** `has data-testid="ai-upgrade-prompt"`
  - **Status:** RED — ENOENT
  - **Verifies:** AC11, AC16

- 🔴 **Test:** `references t("aiUpgradeForSummaries")`
  - **Status:** RED — ENOENT
  - **Verifies:** AC11 — upgrade CTA text

- 🔴 **Test:** `manages showUpgradePrompt state`
  - **Status:** RED — ENOENT
  - **Verifies:** AC11 — upgrade state tracking

- 🔴 **Test:** `manages usageRemaining state — set to 0 on 429`
  - **Status:** RED — ENOENT
  - **Verifies:** AC11 — quota depletion signal

- 🔴 **Test:** `opportunities.ts handles HTTP 429 by calling handlers.onError("usage_limit_exceeded")`
  - **Status:** RED — streamAISummary not yet implemented
  - **Verifies:** AC11, AC13 — 429 routing in API layer

- 🔴 **Test:** `opportunities.ts returns 0 as usageRemaining on 429 response`
  - **Status:** RED — not yet implemented
  - **Verifies:** AC11, AC13 — zero remaining on 429

- 🔴 **Test:** `429 does NOT transition to "error" state — transitions to upgrade state`
  - **Status:** RED — ENOENT
  - **Verifies:** AC11 — limit state ≠ error state (different UX treatment)

### AC12: Error State (6 tests)

- 🔴 **Test:** `has data-testid="ai-summary-error"`
  - **Status:** RED — ENOENT
  - **Verifies:** AC12, AC16

- 🔴 **Test:** `has data-testid="ai-summary-retry-btn"`
  - **Status:** RED — ENOENT
  - **Verifies:** AC12, AC16

- 🔴 **Test:** `references t("aiSummaryError"), t("aiSummaryRetry")`
  - **Status:** RED — ENOENT
  - **Verifies:** AC12 — error i18n keys

- 🔴 **Test:** `handles AbortError as non-fatal`
  - **Status:** RED — ENOENT
  - **Verifies:** AC12 — unmount abort must not show error UI

- 🔴 **Test:** `retry button re-invokes startStreaming`
  - **Status:** RED — ENOENT
  - **Verifies:** AC12 — retry flow

### AC13: opportunities.ts API Functions (15 tests)

- ✅ **Test:** `opportunities.ts exists`
  - **Status:** GREEN ✅ — file pre-exists
  - **Verifies:** prerequisite

- 🔴 **Test:** `exports SSEHandlers interface` + callback members
  - **Status:** RED — not yet added
  - **Verifies:** AC13

- 🔴 **Test:** `exports getAISummary` / GETs correct endpoint / returns null on 404
  - **Status:** RED — not yet added
  - **Verifies:** AC13

- 🔴 **Test:** `exports streamAISummary` / POSTs correct endpoint / returns `number | null`
  - **Status:** RED — not yet added
  - **Verifies:** AC13

- 🔴 **Test:** `parseSSEStream defined` / uses `TextDecoder`
  - **Status:** RED — not yet added
  - **Verifies:** AC13

### AC14: useAISummary Hook (6 tests)

- ✅ **Test:** `use-opportunities.ts exists`
  - **Status:** GREEN ✅ — pre-exists
  - **Verifies:** prerequisite

- 🔴 **Test:** `exports useAISummary` / `queryKey: ["ai-summary", opportunityId]` / `staleTime: 86400000` / `enabled: !!opportunityId`
  - **Status:** RED — not yet added
  - **Verifies:** AC14

### AC15: i18n Keys (17 tests per locale)

**en.json (13 key tests + 2 structural + 1 no-duplicate check = ~16 tests):**

- ✅ **Test:** `en.json exists` / `has "opportunities" namespace`
  - **Status:** GREEN ✅ — pre-exists
  - **Verifies:** prerequisite

- 🔴 **Test:** `en.json has AI summary key: opportunities.aiSummaryGenerating` (× 13)
  - **Status:** RED — keys not yet added to en.json
  - **Verifies:** AC15

- ✅ **Test:** `en.json does NOT duplicate existing keys`
  - **Status:** GREEN ✅ — keys not yet added so no duplicates
  - **Verifies:** AC15 — key uniqueness

**bg.json (13 key tests + 2 structural + 1 parity check = ~16 tests):**

- ✅ **Test:** `bg.json exists` / `has "opportunities" namespace`
  - **Status:** GREEN ✅ — pre-exists
  - **Verifies:** prerequisite

- 🔴 **Test:** `bg.json has AI summary key: opportunities.aiSummaryGenerating` (× 13)
  - **Status:** RED — keys not yet added to bg.json
  - **Verifies:** AC15

- 🔴 **Test:** `bg.json key parity ≥ en.json`
  - **Status:** RED — new keys added to en but not bg
  - **Verifies:** AC15 — key parity

### AC16: All 16 data-testid Attributes (17 tests)

- 🔴 **Test:** `has all 16 required data-testid values from the reference table` (bulk)
  - **Status:** RED — ENOENT
  - **Verifies:** AC16

- 🔴 **Test:** `has data-testid="{testId}" in AISummaryPanel.tsx` (× 16, parameterized)
  - **Status:** RED — ENOENT
  - **Verifies:** AC16 — individual testid coverage

### State Machine Integrity (3 tests)

- 🔴 **Test:** `defines StreamState interface/type with all required fields`
  - **Status:** RED — ENOENT
  - **Verifies:** State machine completeness

- 🔴 **Test:** `defines INITIAL_STATE constant with status: "idle"`
  - **Status:** RED — ENOENT
  - **Verifies:** Initial state correctness

- 🔴 **Test:** `uses useState for StreamState management`
  - **Status:** RED — ENOENT
  - **Verifies:** State management pattern

---

## Required data-testid Attributes

### AISummaryPanel.tsx (16 testids)

| `data-testid` | State | Description |
|---|---|---|
| `ai-summary-panel` | Always | Root container |
| `ai-usage-counter` | Always | Usage display (placeholder / remaining / unlimited) |
| `ai-empty-state` | Idle, no summary | No-summary empty state container |
| `ai-generate-btn` | Idle, no summary | "Generate Summary" CTA |
| `ai-summary-skeleton` | Streaming (before first delta) | Pulsing skeleton loading indicator |
| `ai-streaming-text` | Streaming (after first delta) | Accumulating delta text container |
| `ai-typing-cursor` | Streaming (after first delta) | Blinking cursor, `aria-hidden="true"` |
| `ai-summary-content` | Idle (with summary) + Complete | Summary text display |
| `ai-summary-metadata` | Idle (with summary) + Complete | Token count + generation time |
| `ai-regenerate-btn` | Idle (with summary) + Complete | "Regenerate" trigger button |
| `ai-summary-error` | Error | Error state container |
| `ai-summary-retry-btn` | Error | Retry button |
| `ai-upgrade-prompt` | Limit reached | Upgrade CTA (429 / usageRemaining === 0) |
| `ai-regenerate-confirm-dialog` | Confirm dialog open | shadcn Dialog wrapper |
| `ai-regenerate-confirm-action` | Confirm dialog open | "Generate" confirm action |
| `ai-regenerate-confirm-cancel` | Confirm dialog open | "Cancel" dismiss action |

---

## New i18n Keys Required

**ADD to both `en.json` and `bg.json` under `opportunities` namespace (13 new keys):**

| Key | English Value | Notes |
|-----|---------------|-------|
| `aiSummaryGenerating` | `"Generating summary…"` | Shown during streaming |
| `aiSummaryError` | `"Failed to generate summary. Please try again."` | Error state |
| `aiSummaryRetry` | `"Retry"` | Retry button label |
| `aiUsageRemaining` | `"{remaining} of {limit} summaries remaining this month"` | With params |
| `aiUsageUnlimited` | `"Unlimited AI summaries (Enterprise)"` | Enterprise tier |
| `aiUsageLimitReached` | `"Monthly summary limit reached"` | Limit state |
| `aiRegenerateConfirmTitle` | `"Regenerate AI Summary?"` | Dialog title |
| `aiRegenerateConfirmMessage` | `"This will use 1 of your {remaining} remaining summaries."` | With param |
| `aiRegenerateConfirmMessageUnknown` | `"This will use 1 of your monthly summaries."` | Fallback |
| `aiRegenerateConfirmAction` | `"Generate"` | Dialog confirm button |
| `aiRegenerateConfirmCancel` | `"Cancel"` | Dialog cancel button |
| `aiSummaryCompleteMeta` | `"{tokens} tokens · {seconds}s"` | Complete metadata |
| `aiUpgradeForSummaries` | `"Upgrade your plan to generate AI summaries"` | Upgrade CTA |

**DO NOT re-add existing keys:** `aiAnalysisTitle`, `aiGenerateCta`, `aiRegenerateCta`,
`aiUsagePlaceholder`, `aiGeneratedAt`, `aiTokensUsed`, `aiEmptyDescription`

---

## Mock Requirements

This story uses static source assertions (no runtime mocking needed for ATDD tests).

The Vitest test file reads source files as strings — no network calls, no component mounting.

**For future Playwright E2E coverage (S06.14 scope):**

### SSE Stream Mock

**Endpoint:** `POST /api/v1/opportunities/{id}/ai-summary`

**Success Stream Response:**
```
HTTP 200 Content-Type: text/event-stream X-Usage-Remaining: 9

event: delta
data: {"text": "This is "}

event: delta
data: {"text": "a test summary."}

event: metadata
data: {"tokens": 42, "model": "gpt-4o"}

event: done
data: {"summary_id": "test-uuid"}
```

**429 Response:**
```json
{ "error": "usage_limit_exceeded", "limit": 10, "used": 10, "resets_at": "2026-05-01T00:00:00Z" }
```

---

## Implementation Checklist

### Task 1: Extend `lib/api/opportunities.ts` (AC13)

- [ ] 1.1 Add `SSEHandlers` interface export with `onDelta`, `onMetadata`, `onDone`, `onError`
- [ ] 1.2 Add private `parseSSEStream` async generator: `TextDecoder` + buffer + `\n\n` split
- [ ] 1.3 Add `getAISummary(opportunityId: string): Promise<AISummary | null>` using `apiClient.get`; catch 404 → return `null`
- [ ] 1.4 Add `streamAISummary(opportunityId, token, handlers, signal): Promise<number | null>`:
  - Native `fetch` POST with `Authorization`, `Content-Type`, `x-request-id: crypto.randomUUID()`
  - Read `X-Usage-Remaining` response header
  - Handle 429 → `handlers.onError("usage_limit_exceeded")` → return `0`
  - Iterate `parseSSEStream` → dispatch events to handlers
  - Return usageRemaining as number or null
- [ ] Run tests: `vitest run __tests__/opportunities-ai-s6-13.test.ts`
- [ ] ✅ AC13 tests green (~15 tests pass)

**Effort:** ~3 hours

---

### Task 2: Add `useAISummary` to `lib/queries/use-opportunities.ts` (AC14)

- [ ] 2.1 Import `getAISummary` from `../api/opportunities`
- [ ] 2.2 Export `useAISummary(opportunityId: string)` with `queryKey: ["ai-summary", opportunityId]`, `staleTime: 24 * 60 * 60 * 1000`, `enabled: !!opportunityId`
- [ ] Run tests
- [ ] ✅ AC14 tests green (~6 tests pass)

**Effort:** ~30 minutes

---

### Task 3: Add i18n Translation Keys (AC15)

- [ ] 3.1 Verify 13 new keys absent from `messages/en.json` before adding
- [ ] 3.2 Add 13 new AI summary keys to `messages/en.json` under `opportunities` namespace
- [ ] 3.3 Add matching keys with Bulgarian translations to `messages/bg.json`
- [ ] 3.4 Run `pnpm check:i18n` to verify key parity
- [ ] Run tests
- [ ] ✅ AC15 tests green (~30 tests pass)

**Effort:** ~45 minutes

---

### Task 4: Create `AISummaryPanel.tsx` (AC1, 3–12, 16)

- [ ] 4.1 Create file at `app/[locale]/(protected)/opportunities/components/AISummaryPanel.tsx`
- [ ] 4.2 Add `'use client'` directive at top
- [ ] 4.3 Define `AISummaryPanelProps` interface
- [ ] 4.4 Define `StreamStatus` type and `StreamState` interface
- [ ] 4.5 Define `TIER_SUMMARY_LIMITS` constant with correct values (enterprise: Infinity)
- [ ] 4.6 Implement state machine: `INITIAL_STATE`, `useState<StreamState>`, `useCallback` for `startStreaming`
- [ ] 4.7 Implement `useEffect` cleanup: `return () => { abortControllerRef.current?.abort(); }`
- [ ] 4.8 Render usage counter `ai-usage-counter` with Enterprise guard (no `"X of Infinity"`)
- [ ] 4.9 Render idle-no-summary: `ai-empty-state`, `ai-generate-btn`
- [ ] 4.10 Render idle-with-summary: `ai-summary-content`, `ai-summary-metadata` (aiTokensUsed), `ai-regenerate-btn`
- [ ] 4.11 Render streaming-before-delta: `ai-summary-skeleton`
- [ ] 4.12 Render streaming-after-delta: `ai-streaming-text` + `ai-typing-cursor` (aria-hidden="true")
- [ ] 4.13 Render complete: `ai-summary-content` (streamedText), `ai-summary-metadata` (aiSummaryCompleteMeta), `ai-regenerate-btn`
- [ ] 4.14 Render error: `ai-summary-error`, `ai-summary-retry-btn`
- [ ] 4.15 Render upgrade prompt: `ai-upgrade-prompt` (showUpgradePrompt || usageRemaining === 0)
- [ ] 4.16 Render confirm dialog using shadcn `<Dialog>`: all three testids, title, message, buttons
- [ ] 4.17 Handle `onError("usage_limit_exceeded")` → set `showUpgradePrompt: true`, `status: "idle"` (NOT error)
- [ ] Run tests
- [ ] ✅ AC1, AC3–AC12, AC16 tests green (~90 tests pass)

**Effort:** ~6 hours

---

### Task 5: Modify `AIAnalysisTab.tsx` (AC2)

- [ ] 5.1 Import `AISummaryPanel` from `./AISummaryPanel`
- [ ] 5.2 Remove all TODO-marked stub content (static summary display, usage placeholder, both buttons)
- [ ] 5.3 Render `<AISummaryPanel opportunityId={opportunity.id} initialSummary={opportunity.ai_summary} />`
- [ ] Keep `t("aiAnalysisTitle")` heading
- [ ] Run tests
- [ ] ✅ AC2 tests green (~7 tests pass)

**Effort:** ~30 minutes

---

## Running Tests

```bash
# Run all failing tests for this story
cd eusolicit-app/frontend/apps/client && \
  PATH="/home/debian/.nvm/versions/node/v22.22.1/bin:$PATH" \
  ./node_modules/.bin/vitest run __tests__/opportunities-ai-s6-13.test.ts

# Run with verbose output
cd eusolicit-app/frontend/apps/client && \
  PATH="/home/debian/.nvm/versions/node/v22.22.1/bin:$PATH" \
  ./node_modules/.bin/vitest run __tests__/opportunities-ai-s6-13.test.ts --reporter=verbose

# Run just one describe block while implementing
cd eusolicit-app/frontend/apps/client && \
  PATH="/home/debian/.nvm/versions/node/v22.22.1/bin:$PATH" \
  ./node_modules/.bin/vitest run __tests__/opportunities-ai-s6-13.test.ts -t "AC13"

# Run i18n key check
cd eusolicit-app/frontend && pnpm check:i18n
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete) ✅

**TEA Agent Responsibilities:**

- ✅ All 162 tests written; 149 fail, 13 pass (correct RED phase)
- ✅ Failure type: `ENOENT` (file not found) or assertion failures — clear actionable messages
- ✅ Tests fail due to missing implementation, NOT test bugs
- ✅ All 16 `data-testid` values documented and asserted individually
- ✅ All 13 i18n keys documented and asserted per locale
- ✅ Critical mistake prevention: `new EventSource` negative assertions present
- ✅ Implementation checklist created

**Verification:**

```
Tests  149 failed | 13 passed (162)
Status: ✅ RED phase verified — 2026-04-17 14:28:52
```

---

### GREEN Phase (DEV Team — Next Steps)

**DEV Agent Responsibilities:**

1. **Start with Task 1** (extend `opportunities.ts`) — unblocks AC13, partial AC10
2. **Task 2** (`useAISummary`) — quick win, ~30 min, unblocks AC14
3. **Task 3** (i18n keys) — quick win, unblocks AC15
4. **Task 4** (create `AISummaryPanel.tsx`) — main body of work
5. **Task 5** (modify `AIAnalysisTab.tsx`) — final integration step
6. Run full test suite after each task to see progress

**Key Implementation Warnings:**

| ⚠️ Trap | ✅ Correct Approach |
|---------|-------------------|
| Use `new EventSource(url)` | Use native `fetch` with `ReadableStream` — POST endpoint |
| Use `useSSE` hook from `@eusolicit/ui` | GET-only hook — cannot POST |
| Use `apiClient` (Axios) for streaming | Axios buffers body — use `response.body` as ReadableStream |
| Start streaming in `useEffect` | Use `useCallback` triggered from click handler only |
| Call `queryClient.invalidateQueries` on `metadata` event | Call only on `done` event |
| Show dialog on first "Generate" (no summary) | Dialog only for "Regenerate" (summary exists) |
| Render `"X of Infinity remaining"` for Enterprise | Check `isEnterprise` → render `t("aiUsageUnlimited")` |
| Re-add existing i18n keys | Only add the 13 NEW keys; keep the 7 existing |
| Set `status: "error"` on `usage_limit_exceeded` | Set `showUpgradePrompt: true`, `status: "idle"` |
| Forget `AbortController` cleanup | `return () => { abortControllerRef.current?.abort(); }` in `useEffect` |

---

### REFACTOR Phase (After All Tests Pass)

1. Verify all 162 tests pass GREEN
2. Check TypeScript strict mode — no `any` (use `Record<string, unknown>` + `as` casts)
3. Run `pnpm check:i18n` for key parity
4. Verify `AbortError` is caught silently (no console noise on navigation)
5. Code review: state machine transitions, useCallback dependencies

---

## Epic Test Design Coverage

| Test ID | Priority | Scenario | Tests in S06.13 |
|---------|----------|----------|-----------------|
| **E06-P1-029** | P1 | AI summary panel Generate CTA → SSE streaming text → typing animation → usage counter | `AC5`, `AC7 (P0)`, `AC10`, `AC16` testids |
| **E06-P2-008** | P2 | "Regenerate" button shows confirmation dialog with remaining quota count | `AC6` dialog testids, `aiRegenerateConfirmMessage` |
| **E06-P2-009** | P2 | "X of Y summaries remaining this month" counter; updates after generation | `AC10` TIER_SUMMARY_LIMITS, `aiUsageRemaining` |

**Playwright E2E Note:** E06-P0-010 (end-to-end tier enforcement via S06.14 global 429 interceptor)
will cover the full AI Analysis tab flow. No new Playwright specs required for this story.

---

## Notes

- **Backend ready:** All SSE endpoints (`GET`/`POST /api/v1/opportunities/{id}/ai-summary`) are
  implemented in S06.08. The backend is not being tested here — only the frontend wires it.
- **Test pattern:** Static source assertions match the established S06.12 pattern exactly.
  No component mounting, no network calls, no browser sessions.
- **P0 marker on AC7 tests:** The `fetch` vs `EventSource` tests are marked as critical (P0)
  because using `EventSource` silently fires a GET, returning cached data instead of streaming —
  a failure mode that would be invisible without this assertion.
- **Self-referential AC17 test:** The test file existence test passes immediately upon file save,
  making AC17 permanently green even before any implementation.
- **13 passing tests on initial run:** The 13 passing tests cover pre-existing infrastructure
  (opportunities.ts, use-opportunities.ts, en.json, bg.json, AIAnalysisTab.tsx,
  and the test file itself). This is expected and correct.

---

## Test Execution Evidence

### Initial Run — RED Phase Verification

**Command:** `cd eusolicit-app/frontend/apps/client && ./node_modules/.bin/vitest run __tests__/opportunities-ai-s6-13.test.ts`

**Results:**
```
Test Files  1 failed (1)
     Tests  149 failed | 13 passed (162)
  Start at  14:28:52
  Duration  615ms
```

**Status:** ✅ RED phase verified — 2026-04-17

**Primary failure reason:** `ENOENT: no such file or directory, open '.../AISummaryPanel.tsx'`
All structural assertions for the new file fail because `AISummaryPanel.tsx` does not yet exist.

---

**Generated by BMad TEA Agent** — 2026-04-17
