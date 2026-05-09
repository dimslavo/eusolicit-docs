---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-identify-targets
  - step-03-generate-tests
  - step-04-validate-and-summarize
lastStep: step-04-validate-and-summarize
lastSaved: '2026-05-09'
workflowType: bmad-testarch-automate
mode: bmad-integrated
scope: cluster-C-approvals
batch: P3-b
storyKeys:
  - 10-8-approval-workflow-stages-crud-api
  - 10-9-approval-decision-engine
  - 10-16-approval-pipeline-bid-decision-ui
detectedStack: fullstack
executionMode: sequential
inputDocuments:
  - eusolicit-docs/test-artifacts/full-suite-rollout-plan.md
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/approvals/page.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/approvals/components/ApprovalPipelinePage.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/approvals/components/ApprovalStepper.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/approvals/components/ApprovalDecisionForm.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/approvals/components/ApprovalEmptyState.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/approvals/components/ApprovalHistoryTable.tsx
  - eusolicit-app/frontend/apps/client/app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/approvals/components/WorkflowPickerBanner.tsx
  - eusolicit-app/frontend/apps/client/lib/api/approvals.ts
  - eusolicit-app/frontend/apps/client/lib/queries/use-collaborators.ts
---

# Automation Summary: P3-b — Approvals UI (cluster C, sub-run 2)

**Date:** 2026-05-09
**Author:** TEA Master Test Architect (bmad-testarch-automate)
**Batch:** P3-b (cluster C, second net-new E2E coverage sub-run)
**Status:** ✅ Static validation passed; runtime execution defers to CI.

---

## Why this surface

Compliance-critical workflow under epic 10 (3 stories: 10-8/10-9/10-16). Multi-stage
pipeline UI with role-gated decision form, multiple empty-states, and live POST
submission. Before this sub-run: 0 Playwright specs, only API + service-layer pytest.

The page is a non-trivial graph of conditional branches:

| Branch | Condition | UI |
|---|---|---|
| Loading | any of 4 hooks loading | `loading-spinner` |
| Page error | proposal/history/workflows query errors | `approval-pipeline-page-error` |
| No workflows | `workflows.items.length === 0` | `approval-empty-state-no-workflows` |
| Pinned workflow deleted | history points at workflow that 404s | `approval-pipeline-workflow-deleted` |
| No default workflow | no pinned, no default | `WorkflowPickerBanner` |
| Happy path | workflow + history resolved | `approval-pipeline-page` + stepper + (form if canDecide) + history |

5 user-journey tests cover the most common (and highest-risk) branches:
empty / error / happy-admin / read-only-gating / decision-submission. Edge cases
like "pinned workflow soft-deleted" and "fully-approved banner after final stage"
are explicitly noted as deferred follow-ups in the summary below.

---

## Files Created

### `e2e/specs/approvals/approvals.spec.ts` (NEW)

5 tests, 5 describe scopes:

| # | Scope | Description |
|---|---|---|
| 1 | Empty workflows | Mock `GET /api/v1/approval-workflows` → `{items: [], total: 0}`. Assert `approval-empty-state-no-workflows` testid renders (component branch at ApprovalPipelinePage.tsx:189-191). |
| 2 | Error state | Mock `GET /api/v1/proposals/{id}/approvals/history` → 500. Assert `approval-pipeline-page-error` testid surfaces (line 178-187). |
| 3 | Happy path (admin, P0) | Seed admin auth-store, mock all 5 GET endpoints. Assert `approval-pipeline-page`, `approval-stepper`, `approval-decision-form`, and `approval-history-table` all visible. Validates `canDecide=true` branch for role="admin" (line 148). |
| 4 | Permission gating (read_only) | Seed read_only auth-store, empty collaborators list. Assert decision form **not** rendered while stepper + history still visible. Validates `useMyProposalRole` returns null for non-admin not in collaborators (use-collaborators.ts:26-39) → `canDecide=false`. |
| 5 | Decision submission (P0) | Admin user, mock workflows + history (empty so first stage is current). Click `approval-decision-radio-approved`, fill `approval-decision-comment`, click `approval-decision-submit`. Capture POST `/api/v1/proposals/{id}/approvals/decide` and assert `{decision: "approved", comment: "...", stage_id: <legal-stage-id>}`. |

### Mock helpers

- `mockApprovalsBackend(page, opts)` — single helper that wires all 5 GET routes (`/api/v1/companies/*`, `/api/v1/companies/*/members`, `/api/v1/approval-workflows`, `/api/v1/proposals/{id}`, `/api/v1/proposals/{id}/approvals/history`, `/api/v1/proposals/{id}/collaborators`). Accepts overrides for workflows / history / collaborators / members so each test specifies only what it needs. Supports `{status, body}` failure shape for error coverage.
- `seedAuthStore(page, user)` — pins a specific role in the v1 `eusolicit-auth-store` key + drops the `eusolicit-session` cookie. Avoids the standard `authenticatedPage` fixture because this spec needs admin / read_only role variants that the fixture (which mints role=member) doesn't expose.
- `WORKFLOW_PAYLOAD` / `EMPTY_WORKFLOWS_PAYLOAD` / `EMPTY_HISTORY_PAYLOAD` / `PROPOSAL_PAYLOAD` / `ADMIN_MEMBER_PAYLOAD` — structured fixtures with stable UUIDs so testid suffixes are deterministic.

---

## Validation

| Check | Result |
|---|---|
| `tsc -p e2e/tsconfig.json` filtered for the new file | ✅ 0 errors |
| `npx playwright test --list` | ✅ **5 tests collected** in `e2e/specs/approvals/approvals.spec.ts` |
| Active vs fixme split | 5 active / 0 fixme |
| Local runtime | ❌ Same chromium-headless-shell blocker (`libnspr4.so`) — defers to CI. |

### Production-state cross-check

| Test asserts | Live source | Match? |
|---|---|---|
| Route `/en/workspace/default/proposals/{id}/approvals` exists | `app/[locale]/(protected)/workspace/[workspaceId]/proposals/[id]/approvals/page.tsx` | ✅ |
| `data-testid="approval-pipeline-page"` outer wrapper | `ApprovalPipelinePage.tsx:226` | ✅ |
| `data-testid="approval-pipeline-page-error"` (error branch) | `ApprovalPipelinePage.tsx:179` | ✅ |
| `data-testid="approval-empty-state-no-workflows"` (empty branch) | `ApprovalEmptyState.tsx:27` | ✅ |
| `data-testid="approval-stepper"` | `ApprovalStepper.tsx:49` | ✅ |
| `data-testid="approval-decision-form"` | `ApprovalDecisionForm.tsx:110` | ✅ |
| `data-testid="approval-decision-radio-approved"` etc. | `ApprovalDecisionForm.tsx:140-162` | ✅ |
| `data-testid="approval-decision-comment"` | `ApprovalDecisionForm.tsx:197` | ✅ |
| `data-testid="approval-decision-submit"` | `ApprovalDecisionForm.tsx:214` | ✅ |
| `data-testid="approval-history-table"` | `ApprovalHistoryTable.tsx:70` | ✅ |
| Admin role short-circuits to `canDecide=true` | `use-collaborators.ts:31-32` + `ApprovalPipelinePage.tsx:148` | ✅ |
| GET `/api/v1/approval-workflows` | `lib/api/approvals.ts:79` | ✅ |
| GET `/api/v1/proposals/{id}/approvals/history` | `lib/api/approvals.ts:115` | ✅ |
| POST `/api/v1/proposals/{id}/approvals/decide` | `lib/api/approvals.ts:101` | ✅ |
| GET `/api/v1/proposals/{id}/collaborators` | `lib/api/collaborators.ts:41` | ✅ |
| Admin user-id appears in members list under `user_id` field | `lib/api/members.ts` resp shape (also `id` for legacy fixtures) | ✅ |

### Runtime validation runbook

```bash
# From eusolicit-app/
make up
sudo apt-get install -y libnspr4 libnss3 libxkbcommon0 \
    libatk1.0-0 libatk-bridge2.0-0 libcups2 libxcomposite1 \
    libxdamage1 libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 \
    libcairo2 libasound2  # or: npx playwright install-deps
npx playwright test --config=playwright.config.ts \
    e2e/specs/approvals/approvals.spec.ts \
    --project=client-chromium --reporter=list
```

### Risks / open questions for the CI pass

1. **Stage ID propagation in decision submission**: test #5 asserts the POST body
   contains `stage_id: <legal-stage-id>`. `computeCurrentStage` (in
   `approval-stepper-utils`) determines this from history vs workflow stages.
   With empty history + a workflow whose first stage is `STAGE_LEGAL_ID`, the
   first stage should be selected. If the util's behaviour differs (e.g. it
   uses `sequence` order vs index), the assertion may fail. Watch first CI run.

2. **Decision-form submit-button enable conditions**: `ApprovalDecisionForm`
   may require both a radio selection AND a non-empty comment before the submit
   button enables. The test fills both before clicking, so this should be safe.
   If submit stays disabled due to other validation (e.g. "comment is too short"
   for the rejected/returned variants), the click would no-op silently. Adding
   an explicit `await expect(submitBtn).toBeEnabled()` before click would
   harden this — leaving as-is for now since the live assertions are wrapped
   in `expect.poll`.

3. **Auth-store role hydration**: same as P3-a — relies on the v1→v2 migration
   path lifting the seeded role. If a future auth-store change strips unknown
   `role` values, the read_only gating test may fail. Saved as a fallback note.

4. **`mockApprovalsBackend` route order**: `/api/v1/companies/*` routes catch
   all GETs to that path. The handler explicitly falls through to `/members`
   when the URL contains `/members` so the `mockMembers` companion routes can
   handle it. If the order of `page.route` registration matters, the spec may
   need to register the more-specific `/members` route FIRST. Tested via
   `--list` collection only — first runtime should confirm.

5. **Deferred branches** (worth covering in a follow-up sub-run):
   - **Pinned-workflow-deleted state** (`approval-pipeline-workflow-deleted` testid).
     Needs a 404 response from the orphan-fetch path.
   - **Fully-approved banner** (`approval-fully-approved-banner` + pill). Needs
     a history payload with all stages decided, calling `isProposalFullyApproved`
     truthy.
   - **WorkflowPickerBanner** (`approval-workflow-picker` + select + apply).
     Needs a workflows list with no `is_default: true` entry and empty history.
   - **Reject + return-for-revision flows** for decision form.

---

## Coverage Delta

- Before P3-b: **0** Playwright tests for the Approvals page.
- After P3-b: **5 active** tests covering empty / error / happy-admin /
  read-only-gating / decision-submission scopes.

---

## What's Next

Per `full-suite-rollout-plan.md`:

- **P3-c** — Analytics dashboards: Pipeline Forecasting + ROI (Professional-tier
  gated). Tier-gate visual coverage is the big gap for cluster C.
- **Deferred follow-up** for P3-b:
  - Pinned-workflow-deleted state (story 10-8 edge case).
  - Fully-approved banner + pill (final-stage decision flow).
  - WorkflowPickerBanner manual-selection flow.
  - Reject + return-for-revision decision variants.
- After P3-b runs green in CI, update the rollout-plan checklist line.
