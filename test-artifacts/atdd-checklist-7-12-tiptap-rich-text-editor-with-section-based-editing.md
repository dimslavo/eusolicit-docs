---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-18'
workflowType: bmad-testarch-atdd
mode: story-level
storyKey: 7-12-tiptap-rich-text-editor-with-section-based-editing
storyId: S07.12
inputDocuments:
  - eusolicit-docs/implementation-artifacts/7-12-tiptap-rich-text-editor-with-section-based-editing.md
  - eusolicit-docs/test-artifacts/test-design-epic-07.md
  - eusolicit-app/frontend/apps/client/__tests__/proposals-workspace-s7-11.test.ts
  - eusolicit-app/frontend/apps/client/messages/en.json
  - eusolicit-app/frontend/apps/client/vitest.config.ts
  - eusolicit/_bmad/bmm/config.yaml
tddPhase: RED
detectedStack: fullstack
generationMode: ai-generation
executionMode: sequential
---

# ATDD Checklist: Story 7.12 — Tiptap Rich Text Editor with Section-Based Editing

**Date:** 2026-04-18  
**Author:** TEA Master Test Architect  
**TDD Phase:** 🔴 RED — Tests define the specification; all will FAIL until implementation is complete.  
**Story:** S07.12 | **Epic:** E07 | **Test File:** `__tests__/tiptap-editor-s7-12.test.ts`

---

## Step 1: Preflight & Context

### Stack Detection

- **Detected stack:** `fullstack` (Next.js 14 App Router + FastAPI backend)
- **Story scope:** Pure frontend story — Tiptap editor integration; no backend changes
- **Test framework:** Vitest (`environment: "node"`) — static/structural tests only
- **Browser automation:** Not applicable for this story (ATDD uses filesystem + source inspection)

### Prerequisites Satisfied

- [x] Story approved with 14 clear acceptance criteria (AC1–AC14)
- [x] Vitest configured (`vitest.config.ts` — `environment: "node"`)
- [x] Existing test pattern available (`proposals-workspace-s7-11.test.ts`)
- [x] Epic-level test design loaded (`test-design-epic-07.md`)

### Context Loaded

- **Story file:** `eusolicit-docs/implementation-artifacts/7-12-tiptap-rich-text-editor-with-section-based-editing.md`
- **Test design IDs in scope:** E07-P1-022, E07-P1-023, E07-P1-024, E07-P2-015, E07-P0-011 (structural only)
- **Related risks in scope:** E07-R-002 (optimistic locking race — validates hash implementation exists)
- **Pattern source:** `proposals-workspace-s7-11.test.ts` — Vitest + `node:fs` static assertions

---

## Step 2: Generation Mode

**Mode selected:** AI Generation (sequential)

**Rationale:** This story specifies structural/static tests (Vitest Node `fs` pattern). The acceptance criteria describe file-system artefacts, source code patterns, exports, and JSON key presence — all inspectable without a running browser. No browser recording needed. Story AC14 explicitly specifies the test file format as: "same Vitest + Node `fs` pattern as `proposals-workspace-s7-11.test.ts`."

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Mapping

| AC  | Description | Test Level | Priority | Epic Test ID |
|-----|-------------|------------|----------|--------------|
| AC1 | Tiptap + md5 packages in package.json | Static (JSON) | P0 | — |
| AC2 | Placeholder removed; ProposalEditor referenced | Static (source) | P0 | E07-P1-022 |
| AC3 | ProposalEditor + ProposalSection exist with testids | Static (source) | P0 | E07-P1-022 |
| AC4 | ProposalEditorToolbar with all 8 button testids | Static (source) | P0 | E07-P1-022 |
| AC5 | Zustand store: activeSectionKey, no persist | Static (source) | P0 | E07-P1-022 |
| AC6 | Auto-save: 1_500 debounce, autoSaveSection, saveStatus | Static (source) | P0 | E07-P1-023 |
| AC7 | Full save: metaKey/ctrlKey + "s", preventDefault, cleanup | Static (source) | P0 | E07-P1-023 |
| AC8 | content-hash.ts exports computeContentHash; 4 new API interfaces + 2 functions | Static (source) | P0 | E07-R-002 |
| AC9 | ConflictDialog.tsx with 3 testids + translations | Static (source) | P0 | E07-P1-024 |
| AC10 | Store: getActiveEditor + registerEditor; no data- attribute | Static (source) | P1 | — |
| AC11 | isGenerating prop; data-generating; editable; toolbar disabled | Static (source) | P1 | E07-P2-015 |
| AC12 | ProposalWorkspacePage: useProposalEditorStore; no hardcoded saveStatus | Static (source) | P1 | E07-P1-023 |
| AC13 | 16 new i18n keys in en.json + bg.json; key parity | Static (JSON) | P0 | — |
| AC14 | useSaveSection + useFullSave mutations; no auto-invalidate | Static (source) | P1 | — |

### TDD Red Phase Requirements

All tests are designed to **fail before implementation** because:

- **New files don't exist yet:** `ProposalEditor.tsx`, `ProposalEditorToolbar.tsx`, `ProposalSection.tsx`, `ConflictDialog.tsx`, `proposal-editor-store.ts`, `content-hash.ts` → `existsSync()` returns `false`
- **Tiptap packages not installed:** `package.json` doesn't yet contain `@tiptap/react` etc. → JSON assertion fails
- **Placeholder still present:** `ProposalWorkspacePage.tsx` contains `"Editor area — S07.12"` → `not.toContain()` fails
- **New i18n keys missing:** `en.json` and `bg.json` don't have 16 new editor keys → key presence assertions fail
- **New exports missing:** `lib/api/proposals.ts` and `use-proposals.ts` don't yet export new interfaces/functions
- **Store import missing:** `ProposalWorkspacePage.tsx` doesn't import `useProposalEditorStore`

---

## Step 4: Test Generation

### Test File Created

**File:** `eusolicit-app/frontend/apps/client/__tests__/tiptap-editor-s7-12.test.ts`  
**Framework:** Vitest (environment: `node`)  
**Pattern:** Same as `proposals-workspace-s7-11.test.ts` — `readFileSync` + `existsSync` + `readJSON`

### Test Counts

| Category | Test Count | Status |
|----------|-----------|--------|
| AC1 — Package installation | 12 | 🔴 WILL FAIL |
| AC2 — Placeholder removed | 5 | 🔴 WILL FAIL |
| AC3 — Section-based rendering | 12 | 🔴 WILL FAIL |
| AC4 — Formatting toolbar | 12 | 🔴 WILL FAIL |
| AC5 — Active section tracking | 8 | 🔴 WILL FAIL |
| AC6 — Auto-save debounce | 9 | 🔴 WILL FAIL |
| AC7 — Cmd+S full save | 6 | 🔴 WILL FAIL |
| AC8 — Optimistic locking hash | 13 | 🔴 WILL FAIL |
| AC9 — Conflict dialog | 8 | 🔴 WILL FAIL |
| AC10 — Cursor tracking | 5 | 🔴 WILL FAIL |
| AC11 — Disabled during generation | 5 | 🔴 WILL FAIL |
| AC12 — saveStatus wired | 3 | 🔴 WILL FAIL |
| AC13 — i18n en.json (16 keys) | 18 | 🔴 WILL FAIL |
| AC13 — i18n bg.json (16 keys + parity) | 19 | 🔴 WILL FAIL |
| AC14 — Mutation hooks | 5 | 🔴 WILL FAIL |
| File structure | 6 | 🔴 WILL FAIL |
| **Total** | **146** | **🔴 137 FAIL / 9 PASS** |

### Actual Test Run (RED Confirmation)

Run: `node node_modules/vitest/vitest.mjs run __tests__/tiptap-editor-s7-12.test.ts`  
Result: **137 FAIL / 9 PASS (146 total)** ✅ Confirmed RED phase.

The 9 passing tests check pre-existing baseline conditions (files already present: `ProposalWorkspacePage.tsx`, `en.json`, `bg.json`, `package.json`; their `"use client"` / namespace presence). These are correct — they establish baseline without gating the new implementation.

### Key Design Decisions

1. **No `test.skip()` used** — this story uses the S7-11 pattern where tests run but fail on missing files/content. The failure occurs at the `existsSync()` / `readFileSync()` / `toContain()` level, not at the Vitest skip level. This matches the established project pattern.

2. **Structural assertions only** — no Tiptap rendering, no DOM simulation. Tests inspect source code text and JSON structure. This keeps tests stable and independent of Next.js/Tiptap internals (per Epic 7 design: "Treat Tiptap as a black box").

3. **Key sorting verification** — AC8 tests that `content-hash.ts` calls `.sort()` on object keys, verifying the critical algorithmic requirement that makes the frontend hash match the Python backend.

4. **AC2 is intentionally RED against S7-11** — The S7-11 test `expect(src).toContain('Editor area — S07.12')` will break when S7-12 is done. The S7-12 ATDD test `expect(src).not.toContain('Editor area — S07.12')` is currently RED and will turn GREEN at the same time.

5. **AC12 hardcoded check** — `not.toMatch(/saveStatus\s*[=:]\s*["']saved["']/)` catches the specific S7-11 pattern `saveStatus="saved"` and ensures it's replaced by a dynamic store-derived value.

---

## Step 5: Validation & Completion

### Validation Checklist

- [x] Prerequisites satisfied (Vitest configured, story has clear ACs)
- [x] Test file written to correct path per story spec (`__tests__/tiptap-editor-s7-12.test.ts`)
- [x] All 14 ACs have test coverage
- [x] Tests are designed to fail before implementation (RED phase)
- [x] Follows established project test pattern (S7-11 Vitest Node fs)
- [x] No orphaned browser sessions (no browser automation used)
- [x] Temp artifacts not used (direct file writes only)
- [x] ATDD checklist saved to `test_artifacts/`

### Epic Test Design Coverage

| Epic Test ID | Priority | Covered By |
|-------------|----------|------------|
| E07-P1-022 | P1 | AC2, AC3, AC4 — section blocks render, toolbar visible |
| E07-P1-023 | P1 | AC6, AC12 — auto-save debounce, save indicator wired |
| E07-P1-024 | P1 | AC9 — conflict dialog testids |
| E07-P2-015 | P2 | AC11 — editor disabled during generation |
| E07-P0-011 | P0 | AC3 (structural) — editor renders; full E2E gated on S07.13 |
| E07-R-002 | Risk | AC8 — optimistic locking hash implementation verified |

### Known Assumptions & Risks

1. **`readFileSync` throws if file missing** — tests checking file existence use `existsSync()` first in the "file structure" group; AC-specific tests that `readFile()` directly will throw if the file doesn't exist. The stack trace will clearly identify which file is missing, aiding developer diagnosis.

2. **S7-11 test regression** — The S7-11 test `'AC2 — centre area contains placeholder text "Editor area — S07.12"'` will become RED when S7-12 removes the placeholder. Developer must update that test to reflect the new state (or delete it, since AC2 in S7-12 supersedes it).

3. **AC14 no-invalidation test** — The regex-based check for no `invalidateQueries` in mutation blocks is a best-effort heuristic. A dev could structure the code to defeat it. Runtime verification during green phase is recommended.

4. **Tiptap SSR** — The story specifies `dynamic(..., { ssr: false })` for `ProposalEditor`. The structural test for AC2 checks `ProposalWorkspacePage.tsx` contains `ProposalEditor` but does not verify `dynamic` usage specifically. This is adequate for ATDD; the SSR verification is best done via E2E (E07-P0-011) once S07.13 is ready.

---

## Next Steps (TDD Green Phase)

After implementing all Story 7.12 tasks:

1. Run tests: `cd eusolicit-app/frontend && pnpm test --filter client`
2. Verify all 146 tests in `tiptap-editor-s7-12.test.ts` PASS (green phase)
3. If any tests fail:
   - Either fix the implementation to match the spec
   - Or fix the test if the spec itself changed (document the change)
4. Update (or remove) the now-failing S7-11 test: `'AC2 — centre area contains placeholder text "Editor area — S07.12"'`
5. Run `pnpm check:i18n` to verify i18n parity
6. Commit passing tests + implementation together

### Implementation Guidance from Tests

**Files to create (tests fail at `existsSync()`):**

| File | Tests That Unblock |
|------|--------------------|
| `components/ProposalEditor.tsx` | AC2, AC3, AC6, AC7, AC8, AC9, AC11, AC12 |
| `components/ProposalEditorToolbar.tsx` | AC4, AC5, AC11 |
| `components/ProposalSection.tsx` | AC3, AC10, AC11 |
| `components/ConflictDialog.tsx` | AC9 |
| `lib/stores/proposal-editor-store.ts` | AC5, AC6, AC10 |
| `lib/utils/content-hash.ts` | AC8 |

**Files to modify (tests fail at `toContain()` / `toMatch()`):**

| File | What to Add |
|------|-------------|
| `apps/client/package.json` | 9 `@tiptap/*` packages + `md5` in deps; `@types/md5` in devDeps |
| `components/ProposalWorkspacePage.tsx` | Remove placeholder; add `ProposalEditor`; import `useProposalEditorStore`; pass `saveStatus` from store |
| `lib/api/proposals.ts` | 4 new interfaces; `autoSaveSection`; `fullSaveProposal`; `generation_status` on `ProposalResponse` |
| `lib/queries/use-proposals.ts` | `useSaveSection`; `useFullSave` mutations (no `invalidateQueries`) |
| `messages/en.json` | 16 new keys under `proposals` namespace |
| `messages/bg.json` | 16 matching Bulgarian keys under `proposals` namespace |

### Recommended Next Workflow

After green phase: run `bmad-testarch-automate` to expand coverage with integration and E2E tests for the rich-text editing interactions (typing, auto-save triggering, conflict resolution flow).
