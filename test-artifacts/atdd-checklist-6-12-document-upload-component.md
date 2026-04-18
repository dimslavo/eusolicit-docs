---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-17'
workflowType: bmad-testarch-atdd
mode: story-level
storyId: 6-12-document-upload-component
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-12-document-upload-component.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/frontend/apps/client/__tests__/opportunities-detail-s6-11.test.ts
  - eusolicit-app/frontend/apps/client/lib/api/opportunities.ts
  - eusolicit-app/frontend/apps/client/lib/queries/use-opportunities.ts
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/DocumentsTab.tsx
  - eusolicit-app/frontend/apps/client/vitest.config.ts
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.12 — Document Upload Component

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**Story:** S06.12 | **Status:** ready-for-dev
**Test File:** `eusolicit-app/frontend/apps/client/__tests__/opportunities-upload-s6-12.test.ts`
**TDD Phase:** 🔴 RED — 112 tests failing / 10 passing (infrastructure checks only)

---

## Step 1: Preflight & Context

### Stack Detection

- **Result:** `fullstack` — both `playwright.config.ts` (E2E) and `vitest.config.ts` (component/unit) present
- **Effective test level for this story:** Vitest static assertions (`environment: "node"`), matching S06.09–S06.11 ATDD pattern
- No browser recording required (pure frontend static-assertion story)

### Prerequisites

- [x] Story approved with clear acceptance criteria (14 ACs)
- [x] Vitest framework configured at `eusolicit-app/frontend/apps/client/vitest.config.ts`
- [x] Existing test pattern established: `opportunities-detail-s6-11.test.ts` (231 tests, static source assertions)
- [x] Development environment available

### TEA Config Flags (from config.yaml defaults)

- `tea_use_playwright_utils`: not configured → N/A for this story
- `tea_use_pactjs_utils`: not configured → N/A
- `tea_pact_mcp`: not configured → N/A
- `tea_browser_automation`: not configured → N/A (Vitest story)
- `test_stack_type`: not configured → auto-detected as `fullstack`

### Inputs Loaded

| Artifact | Status | Key Information |
|----------|--------|-----------------|
| Story 6.12 | ✅ Loaded | 14 ACs, 8 tasks, detailed Dev Notes |
| Epic 06 Test Design | ✅ Loaded | E06-P1-028, E06-P2-005, E06-P1-016 directly map to this story |
| `opportunities-detail-s6-11.test.ts` | ✅ Loaded | Reference pattern for static-assertion style |
| `lib/api/opportunities.ts` | ✅ Loaded | Current exports — upload interfaces ABSENT |
| `lib/queries/use-opportunities.ts` | ✅ Loaded | `useOpportunityDocuments` takes only `id: string` (no options yet) |
| `DocumentsTab.tsx` | ✅ Loaded | Still has static `detail-document-upload-zone` placeholder |
| `vitest.config.ts` | ✅ Loaded | `environment: "node"`, globals: true |

---

## Step 2: Generation Mode

**Selected mode:** AI Generation
**Rationale:** Story is pure frontend static assertions; acceptance criteria are explicit; no browser recording needed for static source inspection tests. All file paths, testid names, API contracts, and i18n keys are fully specified in Dev Notes. Matches established project pattern.

---

## Step 3: Test Strategy

### AC → Test Scenario Mapping

| AC | Test Scenarios | Level | Priority | Currently Failing Trigger |
|----|---------------|-------|----------|--------------------------|
| **AC1** | `DocumentUploadComponent.tsx` exists at correct path | Static | P1 | File doesn't exist |
| **AC1** | `"use client"` directive present | Static | P1 | File doesn't exist |
| **AC1** | Component exported (named or default) | Static | P1 | File doesn't exist |
| **AC2** | `data-testid="upload-zone"` in source | Static | P1 | File doesn't exist |
| **AC2** | `data-testid="upload-file-input"` in source | Static | P1 | File doesn't exist |
| **AC2** | `type="file"`, `multiple`, `accept=".pdf,.docx,.xlsx,.zip"` on hidden input | Static | P1 | File doesn't exist |
| **AC2** | `fileInputRef` + `.click()` for browse-files trigger | Static | P1 | File doesn't exist |
| **AC2** | `onDragOver` calls `preventDefault()`, sets `isDragging` | Static | P1 | File doesn't exist |
| **AC2** | `onDragLeave` resets `isDragging` | Static | P1 | File doesn't exist |
| **AC2** | `onDrop` processes `dataTransfer.files` | Static | P1 | File doesn't exist |
| **AC2** | `isDragging` drives conditional dashed-border class | Static | P1 | File doesn't exist |
| **AC3** | `application/pdf` in MIME allowset | Static | P1 (E06-P1-016) | File doesn't exist |
| **AC3** | DOCX MIME type in allowset | Static | P1 | File doesn't exist |
| **AC3** | XLSX MIME type in allowset | Static | P1 | File doesn't exist |
| **AC3** | `application/zip` in allowset | Static | P1 | File doesn't exist |
| **AC3** | `application/x-zip-compressed` fallback in allowset | Static | P1 | File doesn't exist |
| **AC3** | `MAX_FILE_BYTES` = 104857600 in source | Static | P1 (E06-P2-005) | File doesn't exist |
| **AC3** | `MAX_PACKAGE_BYTES` = 524288000 in source | Static | P1 | File doesn't exist |
| **AC3** | `inferMimeType` helper defined | Static | P1 | File doesn't exist |
| **AC3** | Extension map `.pdf`, `.docx`, `.xlsx`, `.zip` present | Static | P1 | File doesn't exist |
| **AC3** | `validateFile` function defined | Static | P1 | File doesn't exist |
| **AC3** | `upload-error-` testid pattern in source | Static | P2 | File doesn't exist |
| **AC4** | `data-testid="upload-file-queue"` in source | Static | P1 | File doesn't exist |
| **AC4** | `upload-file-item-` dynamic pattern in source | Static | P1 | File doesn't exist |
| **AC4** | `upload-file-status-` dynamic pattern | Static | P1 | File doesn't exist |
| **AC4** | `upload-file-progress-` dynamic pattern | Static | P1 | File doesn't exist |
| **AC4** | `upload-file-cancel-` dynamic pattern | Static | P1 | File doesn't exist |
| **AC4** | `queue` state + `useState` | Static | P1 | File doesn't exist |
| **AC4** | `formatFileSize` called for display | Static | P2 | File doesn't exist |
| **AC4** | `crypto.randomUUID()` for unique file IDs | Static | P1 | File doesn't exist |
| **AC5 (P0)** | `XMLHttpRequest` in source (NOT fetch) | Static | **P0** | File doesn't exist |
| **AC5 (P0)** | XHR opened with `"PUT"` method | Static | **P0** | File doesn't exist |
| **AC5 (P0)** | `xhr.upload.addEventListener` + `"progress"` event | Static | P1 (E06-P1-028) | File doesn't exist |
| **AC5 (P0)** | `Content-Type` + `setRequestHeader` present | Static | P1 | File doesn't exist |
| **AC5 (P0)** | No `Authorization` header set on S3 XHR | Static | **P0** | File doesn't exist |
| **AC5** | `requestPresignedUrl` called with `filename`, `mime_type` | Static | P1 | File doesn't exist |
| **AC5** | `confirmDocumentUpload` called on 2xx | Static | P1 | File doesn't exist |
| **AC5** | Status → `"uploading"` with progress | Static | P1 | File doesn't exist |
| **AC5** | Status → `"scanning"` after confirm | Static | P1 | File doesn't exist |
| **AC5** | Status → `"failed"` on XHR error | Static | P1 | File doesn't exist |
| **AC5** | `uploadToS3` helper or `Promise` wrapper | Static | P1 | File doesn't exist |
| **AC5** | `onUploadComplete()` called after confirm | Static | P1 | File doesn't exist |
| **AC6** | `AbortController` used | Static | P1 | File doesn't exist |
| **AC6** | `signal.addEventListener("abort", () => xhr.abort())` | Static | P1 | File doesn't exist |
| **AC6** | `handleCancel` / `removeFromQueue` removes entry | Static | P1 | File doesn't exist |
| **AC6** | abort stored on `FileUploadState` entry | Static | P1 | File doesn't exist |
| **AC7** | `refetchInterval` in `DocumentsTab.tsx` | Static | P1 | Tab not yet modified |
| **AC7** | `3000` ms + `pending` condition for polling | Static | P1 | Tab not yet modified |
| **AC7** | `false` when no pending docs (stops polling) | Static | P1 | Tab not yet modified |
| **AC8** | `"queued"` state in source | Static | P1 | File doesn't exist |
| **AC8** | `"uploading"` state in source | Static | P1 | File doesn't exist |
| **AC8** | `"scanning"` state in source | Static | P1 | File doesn't exist |
| **AC8** | `"clean"` state in source | Static | P1 | File doesn't exist |
| **AC8** | `"infected"` state in source | Static | P1 | File doesn't exist |
| **AC8** | `"failed"` state in source | Static | P1 | File doesn't exist |
| **AC8** | `UploadStatus` type defined | Static | P1 | File doesn't exist |
| **AC9** | `data-testid="upload-package-size-indicator"` in source | Static | P2 | File doesn't exist |
| **AC9** | `existingTotalBytes` + active bytes computation | Static | P2 | File doesn't exist |
| **AC9** | `formatFileSize` used for size display | Static | P2 | File doesn't exist |
| **AC9** | `existingTotalBytes` prop received | Static | P2 | File doesn't exist |
| **AC10** | `DocumentsTab.tsx` imports `DocumentUploadComponent` | Static | P1 | Tab not yet modified |
| **AC10** | `<DocumentUploadComponent>` rendered | Static | P1 | Tab not yet modified |
| **AC10** | `opportunityId` prop passed | Static | P1 | Tab not yet modified |
| **AC10** | `existingTotalBytes` computed from `reduce()` | Static | P1 | Tab not yet modified |
| **AC10** | `existingTotalBytes` prop passed | Static | P1 | Tab not yet modified |
| **AC10** | `onUploadComplete` + `refetch` wired | Static | P1 | Tab not yet modified |
| **AC10** | `refetchInterval` added to query call | Static | P1 | Tab not yet modified |
| **AC10** | `3000` + `pending` in polling condition | Static | P1 | Tab not yet modified |
| **AC10** | `false` stop condition present | Static | P1 | Tab not yet modified |
| **AC11** | `PresignedUploadResponse` exported from `opportunities.ts` | Static | P1 | Interface absent |
| **AC11** | `doc_id`, `presigned_url`, `expires_in` fields | Static | P1 | Interface absent |
| **AC11** | `ConfirmUploadResponse` exported | Static | P1 | Interface absent |
| **AC11** | `message` field in `ConfirmUploadResponse` | Static | P1 | Interface absent |
| **AC11** | `requestPresignedUrl` exported function | Static | P1 | Function absent |
| **AC11** | POSTs to `/documents/upload` | Static | P1 | Function absent |
| **AC11** | Accepts `filename`, `mime_type` metadata | Static | P1 | Function absent |
| **AC11** | `confirmDocumentUpload` exported function | Static | P1 | Function absent |
| **AC11** | POSTs to `/confirm` endpoint | Static | P1 | Function absent |
| **AC11b** | `formatFileSize` exported from `opportunity-utils.tsx` | Static | P2 | Not added yet |
| **AC11b** | `bytes` parameter present | Static | P2 | Not added yet |
| **AC11b** | `1024`, `KB`, `MB` thresholds in logic | Static | P2 | Not added yet |
| **AC11c** | `useOpportunityDocuments` accepts optional `options` param | Static | P1 | Signature not updated |
| **AC11c** | `...options` spread into `useQuery` | Static | P1 | Not added yet |
| **AC11c** | `UseQueryOptions` type referenced | Static | P1 | Not added yet |
| **AC12** | 15 upload keys in `en.json` (×15 parameterized) | Static | P1 | Keys absent |
| **AC12** | 15 upload keys in `bg.json` (×15 parameterized) | Static | P1 | Keys absent |
| **AC12** | `bg.json` key parity ≥ `en.json` | Static | P1 | *(passes: both absent)* |
| **AC13** | All 9 testid patterns present in one assertion | Static | P2 | File doesn't exist |

### Red Phase Requirements

All tests assert expected **post-implementation** behavior. They fail because:

1. `DocumentUploadComponent.tsx` does not exist → ~70 tests fail immediately on `readFileSync`
2. `PresignedUploadResponse`, `ConfirmUploadResponse`, `requestPresignedUrl`, `confirmDocumentUpload` absent from `opportunities.ts` → 9 tests fail
3. `formatFileSize` absent from `opportunity-utils.tsx` → 3 tests fail
4. `useOpportunityDocuments` does not accept `options` → 3 tests fail
5. `DocumentsTab.tsx` not yet modified (no `DocumentUploadComponent` import, no `refetchInterval`) → 9 tests fail
6. Upload i18n keys absent from `en.json` and `bg.json` → 30 tests fail (15 × 2 languages)

**Verified run (2026-04-17):** `112 failed | 10 passed (122 total)` ✅ RED phase confirmed

---

## Step 4: Test Generation

### Execution Mode

**Resolved mode:** `sequential` (config does not specify `tea_execution_mode`; no subagents dispatched; tests generated directly in main context)

### Generated Test File

**Path:** `eusolicit-app/frontend/apps/client/__tests__/opportunities-upload-s6-12.test.ts`
**Framework:** Vitest (`environment: "node"`)
**Test count:** 122 tests across 16 describe blocks

### Test File Structure

| Describe Block | Test Count | AC Covered |
|----------------|-----------|------------|
| File structure — S6.12 DocumentUploadComponent | 3 | AC1 |
| AC2 — Upload zone and file input | 10 | AC2 |
| AC3 — Validation constants and helpers | 11 | AC3 |
| AC4 — Upload queue and per-file elements | 8 | AC4 |
| AC5 (P0) — XHR-based S3 PUT with progress | 11 | AC5 |
| AC6 — Cancel upload flow | 4 | AC6 |
| AC8 — All six status badge states | 7 | AC8 |
| AC9 — Package size indicator | 4 | AC9 |
| AC13 — Complete data-testid reference coverage | 1 | AC13 |
| AC10 — DocumentsTab.tsx wiring | 9 | AC10 |
| AC11 — API upload interfaces and functions | 9 | AC11 |
| AC11b — opportunity-utils formatFileSize | 4 | AC11b |
| AC11c — useOpportunityDocuments options param | 3 | AC11c |
| AC12 — i18n en.json (15 upload keys) | 17 | AC12 |
| AC12 — i18n bg.json (15 upload keys) | 17 | AC12 |
| **Total** | **122** | **AC1–AC13** |

---

## Step 5: Validation & Completion

### Checklist

- [x] Prerequisites satisfied (Vitest framework configured, story approved)
- [x] Test file created at correct path (`__tests__/opportunities-upload-s6-12.test.ts`)
- [x] All acceptance criteria from story covered (AC1–AC13)
- [x] **Tests designed to fail before implementation — RED phase verified**
  - Verified run output: `112 failed | 10 passed (122 total)`
  - 10 passing: infrastructure existence checks (files that already exist)
  - 112 failing: all implementation-specific assertions
- [x] No browser CLI sessions required or opened
- [x] Temp artifacts: none (sequential mode, no temp files)
- [x] Epic test design scenarios mapped:
  - E06-P1-028 → AC5 XHR + AC2 drag-drop + AC7 polling assertions
  - E06-P2-005 → AC3 MAX_FILE_BYTES constant assertion
  - E06-P1-016 → AC3 MIME allowset assertions
- [x] Critical security check included: AC5 test verifies `Authorization` header NOT sent to S3

### S06.11 Impact Note

The S06.11 ATDD test (`opportunities-detail-s6-11.test.ts`) has one test that will be broken by S06.12 implementation:

```
AC7: 'has data-testid="detail-document-upload-zone" upload placeholder (no upload logic)'
```

Once `DocumentsTab.tsx` is modified to import `DocumentUploadComponent` (replacing the static placeholder), the testid `detail-document-upload-zone` will no longer exist in that source file. The developer must update that S06.11 assertion to check for the `DocumentUploadComponent` import instead — as called out in the Dev Notes, Section "Critical Mistakes to Prevent", item 4.

### Coverage Strategy (from Epic 06 Test Design)

- **Component-level static assertions cover:** 100% of S06.12 ACs
- **No new Playwright E2E tests required for this story** — as confirmed in Dev Notes "Test Design Coverage" section: "The component Vitest tests provide sufficient coverage at this tier"
- **Full E2E tier-gate flow** (E06-P0-010) will be covered when S06.14 global interceptor is implemented

### TDD Red→Green Path

When developer implements Story 6.12:

| Implementation step | Tests that turn GREEN |
|---------------------|----------------------|
| Create `DocumentUploadComponent.tsx` | File structure (3), all component source assertions (~70) |
| Add interfaces + functions to `opportunities.ts` | API export assertions (9) |
| Add `formatFileSize` to `opportunity-utils.tsx` | Utility assertions (3) |
| Update `useOpportunityDocuments` with options param | Query hook assertions (3) |
| Modify `DocumentsTab.tsx` | DocumentsTab assertions (9) |
| Add upload keys to `en.json` | en.json i18n assertions (17) |
| Add upload keys to `bg.json` | bg.json i18n assertions (17) |
| **All done** | **122 / 122 GREEN ✅** |

---

## Run Command

```bash
cd eusolicit-app/frontend/apps/client
./node_modules/.bin/vitest run __tests__/opportunities-upload-s6-12.test.ts
```

*(Requires Node.js in PATH — use `nvm use` if needed)*
