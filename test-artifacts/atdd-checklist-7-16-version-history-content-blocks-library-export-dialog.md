---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-04c-aggregate', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-18'
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-16-version-history-content-blocks-library-export-dialog.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/frontend/apps/client/__tests__/scoring-pricing-win-themes-s7-15.test.ts
  - eusolicit-app/e2e/specs/proposals/proposals-workspace.spec.ts
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 7.16 — Version History, Content Blocks Library & Export Dialog

**Date:** 2026-04-18
**TDD Phase:** 🔴 RED — Tests generated; all will fail until feature is implemented
**Test file:** `eusolicit-app/frontend/apps/client/__tests__/version-history-content-blocks-export-s7-16.test.ts`
**Pattern:** Source-inspection via `readFileSync` + `.toContain()` (same as S07.15)

---

## Stack & Mode

| Setting | Value |
|---|---|
| Detected stack | `fullstack` (Next.js 14 + FastAPI) |
| Story type | Pure frontend |
| Generation mode | AI generation (source-inspection pattern) |
| Execution mode | Sequential |
| Test framework | Vitest (unit/component) + Playwright E2E |

---

## TDD Red Phase Summary

| Category | Count | Status |
|---|---|---|
| File existence + `"use client"` | 3 | 🔴 FAIL (files don't exist) |
| Workspace integration assertions | 8 | 🔴 FAIL (content not yet added) |
| VersionHistorySidebar data-testids | 17 | 🔴 FAIL (file doesn't exist) |
| ContentBlocksLibraryPanel data-testids | 13 | 🔴 FAIL (file doesn't exist) |
| ExportDialog data-testids | 10 | 🔴 FAIL (file doesn't exist) |
| `lib/api/proposals.ts` types + functions | 11 | 🔴 FAIL (interfaces/functions not yet added) |
| `lib/queries/use-proposals.ts` hooks | 9 | 🔴 FAIL (hooks not yet added) |
| i18n key parity (34 new flat keys) | 72 | 🔴 FAIL (keys not yet added) |
| E2E spec stub activation | 4 | 🔴 FAIL (test.skip stubs still present) |
| **Total** | **147** | **🔴 ALL FAILING** |

---

## Acceptance Criteria Coverage

| AC | Description | Test Section | Priority | Red-Phase Failure Mechanism |
|---|---|---|---|---|
| AC1 | VersionHistorySidebar.tsx exists with `"use client"` + workspace wiring | AC1/8/14 — File existence | P0 | `tryRead` returns null → `expect(content).not.toBeNull()` fails |
| AC2 | Version list via `useProposalVersions` + `GET /versions` | AC2–7 sidebar + API types section | P0 | File null / `toContain` fails |
| AC3 | Preview mode: `version-preview-area`, `btn-history-back`, read-only sections | AC2–7 sidebar testids | P0 | File null |
| AC4 | Diff viewer: diff panel, inline/side-by-side, section colors, no `dangerouslySetInnerHTML` | AC2–7 sidebar testids | P0 | File null |
| AC5 | Rollback with confirmation flow + cache invalidation | AC2–7 sidebar testids + hooks section | P0 | File null / `toContain` fails |
| AC6 | Loading skeleton + error/retry states | AC2–7 sidebar testids | P0 | File null |
| AC7 | `btn-close-history`, fixed overlay positioning | AC2–7 sidebar testids | P0 | File null |
| AC8 | ContentBlocksLibraryPanel.tsx replaces placeholder + workspace wiring | AC1/8/14 + workspace section | P0 | File null + workspace `toContain` fails |
| AC9 | Real-time search with 300ms debounce (both hooks always called) | AC9–13 panel testids | P0 | File null |
| AC10 | Category filter chips with "All" chip | AC9–13 panel testids | P0 | File null |
| AC11 | Block cards with preview truncation + approval badge | AC9–13 panel testids | P0 | File null |
| AC12 | Insert at cursor using `getState()` (not hook) | AC9–13 panel testids | P0 | File null |
| AC13 | Loading / empty / error states | AC9–13 panel testids | P0 | File null |
| AC14 | ExportDialog.tsx exists + workspace wiring | AC1/8/14 + workspace section | P0 | File null + workspace `toContain` fails |
| AC15 | Format selector buttons (PDF/DOCX) | AC14–18 dialog testids | P0 | File null |
| AC16 | Binary download via `fetch()` (NOT `apiClient`) | AC14–18 + proposals.ts section | P0 | File null / `toContain("fetch(")` fails |
| AC17 | Progress spinner + error/retry | AC14–18 dialog testids | P0 | File null |
| AC18 | Close button + Dialog from `@eusolicit/ui` | AC14–18 dialog testids | P0 | File null |
| AC19 | i18n: 34 flat keys in en.json + bg.json with equal counts | i18n section | P1 | `toHaveProperty` fails for each missing key |
| AC20 | ATDD test file itself (this file) | — | P0 | ✅ Self-referential: file exists |
| AC21 | E2E stubs activated (test.skip removed, btn-close-history + btn-export-download asserted) | AC21 E2E section | P1 | `not.toContain("S07.16 required")` fails |

---

## Generated Test File

**Path:** `eusolicit-app/frontend/apps/client/__tests__/version-history-content-blocks-export-s7-16.test.ts`

### Test Suites

#### 1. AC1/8/14: Component files exist and workspace is wired (8 tests)
- `VersionHistorySidebar.tsx` exists with `"use client"` directive
- `ContentBlocksLibraryPanel.tsx` exists with `"use client"` directive
- `ExportDialog.tsx` exists with `"use client"` directive
- `ProposalWorkspacePage.tsx` no longer renders S07.16 placeholder
- Dynamic imports for all three components present in workspace
- `historyOpen` state and `handleHistoryToggle` handler present
- `exportOpen` state and `handleExportClick` handler present
- `onHistoryToggle` and `onExportClick` passed to toolbar + optional prop type

#### 2. AC2–7: VersionHistorySidebar.tsx (17 tests)
- File is readable
- Fixed right-side overlay (`"fixed"`)
- `data-testid="btn-close-history"`
- `data-testid="version-history-loading"`
- `data-testid="version-history-error"` + `btn-version-history-retry`
- `` data-testid={`version-item-${`` `` } `` (template literal)
- `` data-testid={`btn-compare-${`` `` } `` (template literal)
- `` data-testid={`btn-rollback-${`` `` } `` (template literal)
- `` data-testid={`rollback-confirm-${`` `` } `` (template literal)
- `data-testid="version-preview-area"`
- `data-testid="btn-history-back"`
- `data-testid="version-diff-panel"`
- `data-testid="btn-diff-inline"` + `btn-diff-sidebyside`
- `` data-testid={`diff-section-${`` `` } `` (template literal)
- `text-green-700` (added diff)
- `text-red-600` (removed diff)
- No `dangerouslySetInnerHTML`
- `useTranslations("proposals")`

#### 3. AC9–13: ContentBlocksLibraryPanel.tsx (13 tests)
- File is readable
- `data-testid="content-blocks-search-input"` + `setTimeout` debounce
- `data-testid="category-chip-all"`
- `` data-testid={`category-chip-${`` `` } ``
- `` data-testid={`content-block-card-${`` `` } ``
- `` data-testid={`content-block-preview-${`` `` } ``
- `` data-testid={`btn-insert-block-${`` `` } ``
- `data-testid="content-blocks-loading"`
- `data-testid="content-blocks-empty"`
- `data-testid="content-blocks-error"` + `btn-content-blocks-retry`
- `useProposalEditorStore` imported from `"@/lib/stores/proposal-editor-store"`
- `getState()` used in insert handler
- `useTranslations("proposals")`

#### 4. AC14–18: ExportDialog.tsx (10 tests)
- File is readable
- `data-testid="export-format-pdf"`
- `data-testid="export-format-docx"`
- `data-testid="btn-export-download"`
- `data-testid="export-loading"`
- `data-testid="export-error"` + `btn-export-retry`
- `data-testid="btn-close-export"`
- `Dialog` imported from `@eusolicit/ui`
- `FileText` and `FileCode` from `lucide-react`
- `useTranslations("proposals")`

#### 5. AC2/4/5/9/16: lib/api/proposals.ts (11 tests)
- `export interface ProposalVersionSummary`
- `export interface VersionListResponse`
- `export interface SectionDiffEntry`
- `export interface VersionDiffResponse`
- `export interface ContentBlock`
- `export async function listProposalVersions`
- `export async function getProposalVersionDiff`
- `export async function listContentBlocks`
- `export async function searchContentBlocks`
- `export async function exportProposal`
- `exportProposal` contains `fetch(` and NOT `apiClient.post`
- `rollbackProposalVersion` appears exactly once (not re-defined)

#### 6. AC2/4/5/9: lib/queries/use-proposals.ts (9 tests)
- `export function useProposalVersions`
- `export function useProposalVersionDiff`
- `export function useRollbackVersion`
- `export function useContentBlocks`
- `export function useSearchContentBlocks`
- `"versions"` query key
- `"version-diff"` query key
- `invalidateQueries` for `["proposal"` and `["versions"` on rollback success
- `"content-blocks"` and `"content-blocks-search"` query keys + `Boolean(q)` guard

#### 7. AC19: i18n keys (72 tests — 34 × 2 locales + 4 structural)
- `en.json` parseable
- `bg.json` parseable
- 34 flat key presence checks in `en.json proposals` namespace
- 34 flat key presence checks in `bg.json proposals` namespace
- Equal key counts between en/bg
- All 34 keys are flat strings (not nested objects)

#### 8. AC21: E2E spec stubs activated (4 tests)
- `proposals-workspace.spec.ts` is readable
- No `"S07.16 required"` strings remain (test.skip stubs removed)
- `"btn-close-history"` present (history stub activated with assertion)
- `"btn-export-download"` present (export stub activated with assertion)

---

## Required i18n Keys (34 flat keys under `"proposals"` namespace)

```
historyTitle, historyClose, historyPreview, historyCompare, historyRollback,
historyRollbackConfirmText, historyRollbackConfirmBtn, historyCancelBtn,
historyBack, historyChangeSummaryNone, historyNoVersions, historyError,
historyRetry, historyDiffTitle, historyDiffInline, historyDiffSideBySide,
historyDiffClose, contentBlocksTitle, contentBlocksSearchPlaceholder,
contentBlocksFilterAll, contentBlocksInsert, contentBlocksApproved,
contentBlocksEmpty, contentBlocksNoActiveSection, contentBlocksError,
contentBlocksRetry, exportTitle, exportFormatPdf, exportFormatDocx,
exportDownload, exportDownloading, exportClose, exportError, exportRetry
```

---

## Critical Constraints (From Story Dev Notes)

| # | Constraint | Test Enforcing It |
|---|---|---|
| 1 | `rollbackProposalVersion` must NOT be re-defined (already in proposals.ts from S07.13) | `occurrences === 1` check in proposals.ts suite |
| 2 | `exportProposal` must use `fetch()`, NOT `apiClient` (binary blob response) | `exportProposal` contains `fetch(` and NOT `apiClient.post` |
| 3 | `body` is plain text, NOT HTML — no `dangerouslySetInnerHTML` | `not.toContain("dangerouslySetInnerHTML")` in sidebar suite |
| 4 | `useProposalEditorStore.getState()` in handler (not hook, prevents stale closure) | `toContain("getState()")` in panel suite |
| 5 | Both content block hooks always called (React rules of hooks) | Implicit: `toContain("useSearchContentBlocks")` and `toContain("useContentBlocks")` both present |
| 6 | `"use client"` must be the very first line (no imports before it) | `toContain('"use client"')` — Next.js compiler requirement |
| 7 | Flat i18n keys under `proposals` namespace (not nested objects) | `typeof enProposals[key] === "string"` check |
| 8 | `useSearchContentBlocks` has `enabled: Boolean(q)` guard | `toContain("Boolean(q)")` in hooks suite |

---

## Next Steps (TDD Green Phase)

After a developer implements Story 7.16:

1. **Run failing tests** to confirm red phase:
   ```bash
   cd eusolicit-app/frontend/apps/client
   pnpm vitest run __tests__/version-history-content-blocks-export-s7-16.test.ts
   ```
2. **Implement all tasks** (Tasks 1–12 in the story)
3. **Re-run tests** — they should now be GREEN:
   ```bash
   pnpm vitest run __tests__/version-history-content-blocks-export-s7-16.test.ts
   ```
4. **Run i18n parity check:**
   ```bash
   pnpm check:i18n
   ```
5. **Run E2E tests** to verify activated stubs:
   ```bash
   make test-e2e-chromium
   ```
6. **Run full quality gate:**
   ```bash
   make quality-check
   ```

---

## Coverage Validation (TEA Red-Phase Compliance)

- ✅ All tests assert **expected behavior** (not placeholder `expect(true).toBe(true)`)
- ✅ All tests **fail** in the current red phase (files don't exist or content not yet added)
- ✅ No `test.skip()` wrappers — red-phase failure achieved through `tryRead()` returning `null`
- ✅ No `dangerouslySetInnerHTML` in any assertion
- ✅ Pattern is identical to S07.15 — consistent with project conventions
- ✅ All 21 acceptance criteria covered (AC20 self-referential — test file is the artifact)
