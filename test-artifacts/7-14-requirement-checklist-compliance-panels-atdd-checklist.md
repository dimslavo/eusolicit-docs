# ATDD Checklist for Story 7.14: Requirement Checklist & Compliance Panels

**Epic Test Design:** `E07-P2-008`, `E07-P2-009`, `E07-P2-015`
**Story:** `7-14-requirement-checklist-compliance-panels`
**Status:** All tests are conceptually **FAILING** as the implementation is not yet complete.

This checklist outlines the acceptance tests to be implemented. These tests are derived from the Acceptance Criteria and will be written as source-inspection tests in `__tests__/requirement-checklist-compliance-s7-14.test.ts`.

---

### AC 1, 7: Panel Integration in Workspace

- [ ] **Test:** `RequirementChecklistPanel.tsx` file exists in the correct directory.
- [ ] **Test:** `CompliancePanel.tsx` file exists in the correct directory.
- [ ] **Test:** `ProposalWorkspacePage.tsx` source code no longer contains the placeholder text `"Checklist — S07.14"`.
- [ ] **Test:** `ProposalWorkspacePage.tsx` source code no longer contains the placeholder text `"Compliance — S07.14"`.
- [ ] **Test:** `ProposalWorkspacePage.tsx` source code contains a dynamic import for `"./RequirementChecklistPanel"`.
- [ ] **Test:** `ProposalWorkspacePage.tsx` source code contains a dynamic import for `"./CompliancePanel"`.
- [ ] **Test:** `ProposalWorkspacePage.tsx` source code renders the `<RequirementChecklistPanel>` component.
- [ ] **Test:** `ProposalWorkspacePage.tsx` source code renders the `<CompliancePanel>` component.

### AC 2-6: Requirement Checklist Panel (`RequirementChecklistPanel.tsx`)

- [ ] **Test (AC2):** The component uses `<QueryGuard>` to handle loading, error, and empty states.
- [ ] **Test (AC2):** In the empty state, a `data-testid="checklist-empty"` element is rendered.
- [ ] **Test (AC3):** The empty state contains a generate button with `data-testid="btn-generate-checklist"`.
- [ ] **Test (AC3):** When a checklist already exists, a re-generate button with `data-testid="btn-regenerate-checklist"` is rendered.
- [ ] **Test (AC4):** A progress bar with `data-testid="checklist-progress"` is rendered when items are present.
- [ ] **Test (AC4):** The component contains logic to group checklist items by their `category` field.
- [ ] **Test (AC5):** The component renders checklist items with a `data-testid="checklist-item-{item.id}"` wrapper.
- [ ] **Test (AC6):** The component checks for `linked_section_key` to determine if an item label is a button or plain text.
- [ ] **Test (AC6):** When an item is clickable, it calls `setActiveSectionKey` from the `useProposalEditorStore`.

### AC 7-12: Compliance Panel (`CompliancePanel.tsx`)

- [ ] **Test (AC8):** The advisory `data-testid="compliance-advisory"` is rendered unconditionally (i.e., it appears outside of any state-based conditional logic).
- [ ] **Test (AC9):** In the idle state, a trigger button `data-testid="btn-check-compliance"` is rendered.
- [ ] **Test (AC9):** In the loading state, a `data-testid="compliance-loading"` element is rendered.
- [ ] **Test (AC9):** In the error state, a `data-testid="compliance-error"` element and a `data-testid="btn-compliance-retry"` button are rendered.
- [ ] **Test (AC10):** Compliance results are rendered in a list with items having a `data-testid="compliance-criterion-{i}"` wrapper.
- [ ] **Test (AC10):** Pass/fail status is indicated by rendering `<CheckCircle2>` and `<XCircle>` icons.
- [ ] **Test (AC10):** Each criterion has an expandable section toggled by `data-testid="compliance-criterion-{i}-toggle"` to show details in `data-testid="compliance-criterion-{i}-details"`.
- [ ] **Test (AC11):** A "Fix" button with `data-testid="compliance-criterion-{i}-fix"` is rendered for failed criteria.
- [ ] **Test (AC11):** The "Fix" button handler calls `setActiveSectionKey` from the `useProposalEditorStore`.
- [ ] **Test (AC12):** When results are displayed, a re-run button with `data-testid="btn-recheck-compliance"` is rendered.

### AC 13: Toolbar Integration

- [ ] **Test:** In `ProposalWorkspacePage.tsx`, the `onComplianceClick` prop is passed to `ProposalToolbar`.
- [ ] **Test:** The `onComplianceClick` handler contains logic to call `setRightActiveTab("compliance")`.

### AC 14: API Additions (`proposals.ts`)

- [ ] **Test:** `lib/api/proposals.ts` exports the `ChecklistItem` interface.
- [ ] **Test:** `lib/api/proposals.ts` exports the `ChecklistResponse` interface.
- [ ] **Test:** `lib/api/proposals.ts` exports the `ComplianceCriterion` interface.
- [ ] **Test:** `lib/api/proposals.ts` exports the `ComplianceCheckResponse` interface.
- [ ] **Test:** `lib/api/proposals.ts` exports the `generateChecklist` async function.
- [ ] **Test:** `lib/api/proposals.ts` implementation for `generateChecklist` calls `POST /api/v1/proposals/.../checklist/generate`.
- [ ] **Test:** `lib/api/proposals.ts` exports the `getChecklist` async function.
- [ ] **Test:** `lib/api/proposals.ts` exports the `toggleChecklistItem` async function.
- [ ] **Test:** `lib/api/proposals.ts` implementation for `toggleChecklistItem` calls `PATCH /api/v1/proposals/.../checklist/items/...`.
- [ ] **Test:** `lib/api/proposals.ts` exports the `runComplianceCheck` async function.
- [ ] **Test:** `lib/api/proposals.ts` implementation for `runComplianceCheck` calls `POST /api/v1/proposals/.../compliance-check`.
- [ ] **Test:** `lib/api/proposals.ts` exports the `getComplianceCheck` async function.
- [ ] **Test:** `lib/api/proposals.ts` implementation for `getComplianceCheck` correctly handles the `NullResultResponse` shape.

### AC 15: React Query Hooks (`use-proposals.ts`)

- [ ] **Test:** `lib/queries/use-proposals.ts` exports the `useChecklist` hook.
- [ ] **Test:** `lib/queries/use-proposals.ts` `useChecklist` uses the query key `["checklist", proposalId]`.
- [ ] **Test:** `lib/queries/use-proposals.ts` exports the `useGenerateChecklist` hook.
- [ ] **Test:** `lib/queries/use-proposals.ts` `useGenerateChecklist` invalidates the `["checklist", proposalId]` query on success.
- [ ] **Test:** `lib/queries/use-proposals.ts` exports the `useToggleChecklistItem` hook.
- [ ] **Test:** `lib/queries/use-proposals.ts` `useToggleChecklistItem` implements an optimistic update pattern using `onMutate`.
- [ ] **Test:** `lib/queries/use-proposals.ts` exports the `useComplianceCheckResult` hook.
- [ ] **Test:** `lib/queries/use-proposals.ts` `useComplianceCheckResult` uses the query key `["compliance", proposalId]`.
- [ ] **Test:** `lib/queries/use-proposals.ts` exports the `useRunComplianceCheck` hook.
- [ ] **Test:** `lib/queries/use-proposals.ts` `useRunComplianceCheck` invalidates the `["compliance", proposalId]` query on success.

### AC 16: i18n Keys

- [ ] **Test:** `messages/en.json` contains the `complianceAdvisory` key under the `proposals` namespace.
- [ ] **Test:** `messages/en.json` contains the `checklistGenerateBtn` key under the `proposals` namespace.
- [ ] **Test:** `messages/bg.json` contains the `complianceAdvisory` key under the `proposals` namespace.
- [ ] **Test:** `messages/bg.json` contains the `checklistGenerateBtn` key under the `proposals` namespace.
- [ ] **Test:** The number of keys in the `proposals` namespace is identical between `en.json` and `bg.json`.

---
Generated by: BMAD Test Architect Agent
Date: 2026-04-18
