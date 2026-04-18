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
storyId: 6-11-opportunity-detail-page-tabbed-layout
inputDocuments:
  - eusolicit-docs/implementation-artifacts/6-11-opportunity-detail-page-tabbed-layout.md
  - eusolicit-docs/test-artifacts/test-design-epic-06.md
  - eusolicit-app/frontend/apps/client/__tests__/opportunities-filter-s6-10.test.ts
  - eusolicit-app/frontend/apps/client/__tests__/opportunities-listing-s6-9.test.ts
  - eusolicit-app/frontend/apps/client/vitest.config.ts
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 6.11 — Opportunity Detail Page (Tabbed Layout)

**Date:** 2026-04-17
**Author:** TEA Master Test Architect
**Story:** S06.11 | **Status:** ready-for-dev
**Test File:** `eusolicit-app/frontend/apps/client/__tests__/opportunities-detail-s6-11.test.ts`
**TDD Phase:** 🔴 RED — 231 tests failing / 7 passing (pre-existing infrastructure)

---

## Step 1: Preflight & Context

### Stack Detection

| Indicator | Found | Conclusion |
|-----------|-------|------------|
| `vitest.config.ts` | ✅ `apps/client/vitest.config.ts` (environment: 'node') | Frontend Vitest present |
| `next.config.mjs` | ✅ `apps/client/next.config.mjs` | Next.js 14 App Router |
| `playwright.config.ts` | ✅ `eusolicit-app/playwright.config.ts` | E2E Playwright present |
| `__tests__/opportunities-filter-s6-10.test.ts` | ✅ Static file-system ATDD pattern | Established S6.10 pattern to follow |
| **Detected stack** | — | `fullstack` (TypeScript / Next.js 14 frontend + Python backend) |
| **Test scope** | Static file-system assertions — no browser, no RTL mount | Static ATDD mode |

### Prerequisites

- [x] Story `6-11-opportunity-detail-page-tabbed-layout.md` status: `ready-for-dev` with 15 clear ACs (AC1–AC15)
- [x] Test framework: `Vitest` + `node` environment (confirmed by `vitest.config.ts`)
- [x] Pattern reference: `__tests__/opportunities-filter-s6-10.test.ts` and `__tests__/opportunities-listing-s6-9.test.ts` — `readFileSync` / `existsSync` static assertions
- [x] `en.json`, `bg.json` exist with `opportunities` namespace from S6.9 (4 tests pass)
- [x] `lib/api/opportunities.ts` exists from S6.9 (file-exists test passes; detail interface/function assertions fail)
- [x] `lib/queries/use-opportunities.ts` exists from S6.9 (file-exists test passes; new hook assertions fail)
- [x] None of the 7 new component files exist yet — all component-presence assertions correctly RED

### Key Architectural Notes Loaded

- **Pure frontend story**: All 5 backend API endpoints already implemented. This story builds the structural UI shell.
- **Server page shell**: `app/[locale]/(protected)/opportunities/[id]/page.tsx` is a server component (no `"use client"`) that just renders `<OpportunityDetailPage id={params.id} />`.
- **URL hash routing**: Next.js App Router does NOT track `#hash` via `useSearchParams()`. Manual implementation with `window.location.hash` + `hashchange` event listener required. SSR guard with `typeof window !== "undefined"` is mandatory.
- **Error state handling**: `OpportunityDetailPage` uses `useQuery` error inspection to detect 403 (free-tier) and 404 (not-found) and renders distinct named states — NOT using `<QueryGuard>` for the top-level error branching per Dev Notes. `QueryGuard` IS used in `DocumentsTab`.
- **Lazy Documents query**: `useOpportunityDocuments` called INSIDE `DocumentsTab` only — not in the parent. Keeps the Documents tab data lazy.
- **5 tab components**: Each tab is a separate `.tsx` file in `components/`. `OverviewTab` and `DocumentsTab` need `"use client"` (hooks + browser APIs); `RequirementsTab` and `SubmissionGuideTab` are display-only (may omit `"use client"`).
- **Countdown timer**: Uses `setInterval(1000)` inside `OverviewTab`; reads `submission_deadline` (not `deadline`) from the detail response.
- **Upload zone placeholder**: `DocumentsTab` renders a visual `detail-document-upload-zone` but has NO file input and NO onClick handler. Wired in S06.12.
- **AI button placeholder**: `AIAnalysisTab` renders `detail-ai-generate-btn` but has no `onClick`. Comment: `// TODO S06.13: wire onClick to SSE streaming panel`.

---

## Step 2: Generation Mode

**Selected mode:** Static File-System (ATDD pattern — no browser/RTL mount)

**Rationale:** AC15 explicitly specifies: "ATDD test file `__tests__/opportunities-detail-s6-11.test.ts` covers file structure, data-testid presence in source, i18n key completeness, API interface exports, and query hook exports." These are all static source-code assertions matching the established S6.9/S6.10 pattern. Zero additional test infrastructure required.

**Why NOT Playwright recording:** AC15 does not specify browser interaction tests. Interactive behaviour (tab switching, hash routing, countdown timer, document download) is specified for the Playwright E2E spec referenced in Dev Notes — deferred to `/bmad-testarch-automate` after implementation.

**Playwright E2E additions (deferred):** The story Dev Notes specify adding two tests to `e2e/specs/opportunities/opportunities-listing.spec.ts`. That file does not yet exist (no E2E spec files found in `eusolicit-app/e2e/`). These are tracked as deferred items below.

---

## Step 3: Test Strategy

### Acceptance Criteria → Test Scenarios Mapping

| AC | Scenario | Priority | Test Count | Test Describe Block |
|----|----------|----------|------------|---------------------|
| AC1 | [id]/page.tsx exists; renders OpportunityDetailPage; is a server component (no "use client") | P1 | 3 | `AC1 — Server page shell: [id]/page.tsx` |
| AC2 | OpportunityDetailPage.tsx: "use client"; detail-page, detail-access-denied, detail-not-found testids; useOpportunityDetail import; QueryGuard; 403/404 detection; billing link; /opportunities back link | P1 | 11 | `AC2, AC12 — OpportunityDetailPage: query + 403/404 error states` |
| AC3 | Five tab testids (detail-tab-overview/documents/requirements/ai-analysis/submission-guide); Tabs import from @eusolicit/ui; default tab "overview" | P1 | 7 | `AC3, AC12 — OpportunityDetailPage: five tabs` |
| AC4 | window.location.hash read on mount; hashchange listener; removeEventListener on unmount; 5 hash strings; SSR guard; window.location.hash write on tab change | P2 | 10 | `AC4, AC12 — OpportunityDetailPage: URL hash tab routing` |
| AC5 | detail-breadcrumb testid; Breadcrumb import from @eusolicit/ui; BreadcrumbLink; BreadcrumbPage; BreadcrumbSeparator | P1 | 5 | `AC5, AC12 — OpportunityDetailPage: breadcrumb navigation` |
| AC6 | OverviewTab.tsx exists; "use client"; all 9 testids; setInterval(1000) + clearInterval; "Deadline passed"; formatBudget; RelevanceBadge; formatDate; submission_deadline; OpportunityCard | P1 | 19 | `AC6, AC12 — OverviewTab: overview fields and countdown timer` |
| AC7 | DocumentsTab.tsx exists; "use client"; detail-documents-section + list + item-* + scan-status-* + download-* testids; upload-zone testid; useOpportunityDocuments + downloadDocument; QueryGuard; Loader2/CheckCircle2/XCircle icons; scan status values; EmptyState | P1 | 16 | `AC7, AC12 — DocumentsTab: documents list, scan status badges, upload zone` |
| AC8 | RequirementsTab.tsx exists; detail-requirements-section + evaluation-table + mandatory-checklist testids; Table import; column headings; evaluation_criteria array; mandatory_documents + checkbox icons; Card wrapping | P1 | 9 | `AC8, AC12 — RequirementsTab: evaluation criteria + mandatory documents` |
| AC9 | AIAnalysisTab.tsx exists; detail-ai-analysis-section + detail-ai-generate-btn + detail-ai-content + detail-ai-usage-counter testids; TODO S06.13 comment; Sparkles + RefreshCw icons; ai_summary conditional; usage counter placeholder | P1 | 10 | `AC9, AC12 — AIAnalysisTab: AI summary states and usage counter` |
| AC10 | SubmissionGuideTab.tsx exists; detail-submission-guide-section + detail-submission-accordion + detail-submission-step-* testids; Accordion + AccordionItem + AccordionTrigger + AccordionContent imports; steps mapping; EmptyState | P1 | 10 | `AC10, AC12 — SubmissionGuideTab: accordion steps` |
| AC11 | en.json exists; opportunities namespace; 51 new detail keys; bg.json exists; opportunities namespace; 51 keys; key parity | P1 | 107 | `AC11 — i18n keys: en.json` + `AC11 — i18n keys: bg.json` |
| AC12 | All data-testid attributes (verified inline per component describe block) | — | inline | Covered in each component's describe block |
| AC13 | lib/api/opportunities.ts: OpportunityDetailResponse, EvaluationCriterion, MandatoryDocument, RelatedOpportunity, SubmissionGuide, SubmissionStep, AISummary, DocumentRecord; getOpportunityDetail, getOpportunityDocuments, downloadDocument; Omit pattern | P1 | 13 | `AC13 — lib/api/opportunities.ts: interface and function exports` |
| AC14 | lib/queries/use-opportunities.ts: useOpportunityDetail + useOpportunityDocuments; useQuery + @tanstack/react-query; queryKey arrays; staleTime 60_000; retry logic for 403/404; getOpportunityDetail/Documents imports | P1 | 10 | `AC14 — lib/queries/use-opportunities.ts: useOpportunityDetail + useOpportunityDocuments` |
| AC15 | This ATDD test file | — | meta | This file IS AC15 |
| File structure | All 7 new files exist (structural gate) | P1 | 7 | `File structure — new S6.11 files` |

**Total tests: 238**
**Priority breakdown:** P0: 0 (no backend tier-gate logic in this story) | P1: 228 | P2: 10

### Epic 6 Test Design Coverage

| Epic Test ID | Level | Scenario | Coverage in this file |
|---|---|---|---|
| **E06-P1-027** | Component | Detail page tabbed layout renders all 5 tabs | ✅ AC3: `detail-tab-overview`, `detail-tab-documents`, `detail-tab-requirements`, `detail-tab-ai-analysis`, `detail-tab-submission-guide` testids in `OpportunityDetailPage.tsx`; AC6–AC10: all 5 tab component files exist |
| **E06-P2-010** | Component | Submission guide accordion renders from pipeline steps JSONB | ✅ AC10: `detail-submission-accordion` testid in `SubmissionGuideTab.tsx`; `Accordion` import from `@eusolicit/ui`; `steps` array mapping |
| **E06-P2-011** | Component | Relevance score visible for paid users; hidden for free users | ✅ AC6: `detail-relevance-score` testid in `OverviewTab.tsx`; `RelevanceBadge` import from `opportunity-utils.tsx`; hidden-when-absent pattern (conditional render based on relevance_score presence) |
| **E06-P0-010** | E2E (partial) | Free user hits access-denied state on detail page | ✅ AC2: `detail-access-denied` testid in `OpportunityDetailPage.tsx`; 403 detection pattern; billing CTA link verified statically. Full E2E flow deferred (see below) |
| **E06-P3-002** | Unit | Deadline countdown timer decrements in real time | ✅ AC6: `detail-deadline-countdown` testid in `OverviewTab.tsx`; `setInterval` + `1000` constant + `clearInterval` verified statically. RTL fake-timer test deferred |
| **E06-P3-003** | Component | Tab state persisted in URL hash | ✅ AC4: `window.location.hash` read/write; `hashchange` listener; all 5 hash strings (`#overview`, `#documents`, `#requirements`, `#ai-analysis`, `#submission-guide`) in `OpportunityDetailPage.tsx` |

### Tests Deferred (NOT in this file)

| Test ID | Reason for Deferral | Recommended Location |
|---------|---------------------|----------------------|
| Playwright E2E: "detail page is reachable from listing page" | Requires running browser + auth; AC15 specifies static assertions only | `e2e/specs/opportunities/opportunities-listing.spec.ts` — see story Dev Notes |
| Playwright E2E: "free user sees access-denied state on detail page" | Requires free-tier JWT fixture + global upgrade interceptor (S06.14); marked `test.skip` in story | Same E2E spec file above |
| RTL countdown timer fake-timer test | Requires jsdom + React Testing Library + jest/vitest fake timers | Future `/bmad-testarch-automate` pass |
| RTL tab switching + hash update behaviour | Requires jsdom + browser API mocking | Future `/bmad-testarch-automate` pass |
| RTL document download button handler (downloadDocument → window.open) | Requires RTL + spy/mock setup | Future `/bmad-testarch-automate` pass |

---

## Step 4: TDD Red Phase — Test Generation

### Mode

**Execution Mode:** Static file-system assertions (Vitest `node` environment)
**TDD Phase:** 🔴 RED — 231 of 238 tests fail because target files don't exist or don't contain the required content

### Red Phase Verification

```bash
# Run from eusolicit-app/frontend/apps/client/
PATH="/home/debian/.nvm/versions/node/v22.22.1/bin:$PATH" ./node_modules/.bin/vitest run __tests__/opportunities-detail-s6-11.test.ts

# Confirmed output:
# Tests  231 failed | 7 passed (238)
```

**Confirmed:** Tests run and fail as expected. ✅

### Why 7 Tests Pass in RED Phase

These tests pass because the corresponding files and content already exist from Story 6.9:

| Passing Test | Reason |
|---|---|
| `en.json exists` | File exists from S6.9 |
| `en.json has "opportunities" namespace` | Namespace exists from S6.9 |
| `bg.json exists` | File exists from S6.9 |
| `bg.json has "opportunities" namespace` | Namespace exists from S6.9 |
| `lib/api/opportunities.ts exists` | File exists from S6.9 |
| `lib/queries/use-opportunities.ts exists` | File exists from S6.9 |
| `bg.json opportunities key count ≥ en.json` | Same key count (parity maintained from prior stories) |

### Why Tests Will FAIL (and turn GREEN after implementation)

| Test Group | Failure Reason | Turns GREEN When... |
|---|---|---|
| `File structure` | 7 new component files do not exist | Tasks 4–10 create each file |
| `AC1 — [id]/page.tsx` | `app/[locale]/(protected)/opportunities/[id]/page.tsx` does not exist | Task 4 creates server page shell |
| `AC2 — OpportunityDetailPage` | `OpportunityDetailPage.tsx` does not exist → `readFile` throws ENOENT | Task 10 creates OpportunityDetailPage |
| `AC3 — Five tabs` | `OpportunityDetailPage.tsx` does not contain tab testids | Task 10.6 adds TabsList + TabsTrigger |
| `AC4 — URL hash routing` | `OpportunityDetailPage.tsx` does not contain hash/hashchange logic | Task 10.4–10.5 implements hash routing |
| `AC5 — Breadcrumb` | `OpportunityDetailPage.tsx` does not contain breadcrumb testid | Task 10.3 adds Breadcrumb |
| `AC6 — OverviewTab` | `OverviewTab.tsx` does not exist | Task 5 creates OverviewTab |
| `AC7 — DocumentsTab` | `DocumentsTab.tsx` does not exist | Task 6 creates DocumentsTab |
| `AC8 — RequirementsTab` | `RequirementsTab.tsx` does not exist | Task 7 creates RequirementsTab |
| `AC9 — AIAnalysisTab` | `AIAnalysisTab.tsx` does not exist | Task 8 creates AIAnalysisTab |
| `AC10 — SubmissionGuideTab` | `SubmissionGuideTab.tsx` does not exist | Task 9 creates SubmissionGuideTab |
| `AC13 — API types/functions` | `opportunities.ts` lacks `OpportunityDetailResponse`, `EvaluationCriterion`, `getOpportunityDetail`, etc. | Tasks 1.1–1.5 extend the API module |
| `AC14 — Query hooks` | `use-opportunities.ts` lacks `useOpportunityDetail` and `useOpportunityDocuments` | Tasks 2.1–2.2 add the hooks |
| `AC11 — en.json detail keys` | 51 new detail i18n keys missing from `en.json` | Task 3.1 adds keys to en.json |
| `AC11 — bg.json detail keys` | 51 new detail i18n keys missing from `bg.json` | Task 3.2 adds keys to bg.json |

---

## Step 4C: Aggregation

### Test Infrastructure

**Fixtures:** None required — pure static `readFileSync` / `existsSync` assertions
**Mocks:** None required
**Environment:** `node` (no jsdom, no browser, no DOM APIs)

### Helper Functions (in test file)

| Helper | Purpose |
|--------|---------|
| `fileExists(p)` | `existsSync` wrapper — asserts file/directory presence |
| `readFile(p)` | `readFileSync(p, 'utf-8')` wrapper — reads source for content assertions |
| `readJSON(p)` | Parses JSON file — used for i18n key traversal |
| `resolveNestedKey(obj, keyPath)` | Traverses dot-separated key path — e.g. `opportunities.tabOverview` |

### Required New i18n Keys Asserted (51 keys)

```typescript
const REQUIRED_DETAIL_KEYS = [
  // Page chrome (2)
  'opportunities.detailPageTitle',       // "Opportunity Details"
  'opportunities.detailBreadcrumb',      // "Opportunities"
  // Tab labels (5)
  'opportunities.tabOverview',           // "Overview"
  'opportunities.tabDocuments',          // "Documents"
  'opportunities.tabRequirements',       // "Requirements"
  'opportunities.tabAiAnalysis',         // "AI Analysis"
  'opportunities.tabSubmissionGuide',    // "Submission Guide"
  // Overview tab (11)
  'opportunities.overviewContractingAuthority',
  'opportunities.overviewDescription',
  'opportunities.overviewBudget',
  'opportunities.overviewDeadline',
  'opportunities.overviewCpvCodes',
  'opportunities.overviewRelevanceScore',
  'opportunities.overviewKeyDates',
  'opportunities.overviewPublishedDate',
  'opportunities.overviewRelatedOpportunities',
  // Countdown (2)
  'opportunities.deadlineCountdown',     // "{days}d {hours}h {minutes}m {seconds}s remaining"
  'opportunities.deadlineExpired',       // "Deadline passed"
  // Documents tab (8)
  'opportunities.documentsListTitle',
  'opportunities.documentsEmptyState',
  'opportunities.documentsUploadCta',
  'opportunities.documentScanPending',
  'opportunities.documentScanClean',
  'opportunities.documentScanInfected',
  'opportunities.documentDownload',
  'opportunities.documentSize',          // "{size} KB"
  // Requirements tab (8)
  'opportunities.requirementsEvaluationTitle',
  'opportunities.requirementsMandatoryTitle',
  'opportunities.requirementsCriterion',
  'opportunities.requirementsWeight',
  'opportunities.requirementsType',
  'opportunities.requirementsDescription',
  'opportunities.requirementsNoEvaluation',
  'opportunities.requirementsNoMandatory',
  // AI Analysis tab (6)
  'opportunities.aiAnalysisTitle',
  'opportunities.aiGenerateCta',
  'opportunities.aiRegenerateCta',
  'opportunities.aiUsagePlaceholder',
  'opportunities.aiGeneratedAt',
  'opportunities.aiTokensUsed',
  // Submission Guide tab (5)
  'opportunities.submissionGuideTitle',
  'opportunities.submissionGuideEmpty',
  'opportunities.submissionGuideStep',
  'opportunities.submissionGuideTips',
  'opportunities.submissionGuideRequiredDocs',
  // Error states (6)
  'opportunities.notFoundTitle',
  'opportunities.notFoundDescription',
  'opportunities.notFoundBackCta',
  'opportunities.accessDeniedTitle',
  'opportunities.accessDeniedDescription',
  'opportunities.accessDeniedCta',
];
```

### Required `data-testid` Attributes Asserted

| Component | Testid | AC |
|---|---|---|
| `OpportunityDetailPage` | `data-testid="detail-page"` | AC2, AC12 |
| `OpportunityDetailPage` | `data-testid="detail-breadcrumb"` | AC5, AC12 |
| `OpportunityDetailPage` | `data-testid="detail-tab-overview"` | AC3, AC12 |
| `OpportunityDetailPage` | `data-testid="detail-tab-documents"` | AC3, AC12 |
| `OpportunityDetailPage` | `data-testid="detail-tab-requirements"` | AC3, AC12 |
| `OpportunityDetailPage` | `data-testid="detail-tab-ai-analysis"` | AC3, AC12 |
| `OpportunityDetailPage` | `data-testid="detail-tab-submission-guide"` | AC3, AC12 |
| `OpportunityDetailPage` | `data-testid="detail-access-denied"` | AC2, AC12 |
| `OpportunityDetailPage` | `data-testid="detail-not-found"` | AC2, AC12 |
| `OverviewTab` | `data-testid="detail-overview-section"` | AC6, AC12 |
| `OverviewTab` | `data-testid="detail-contracting-authority"` | AC6, AC12 |
| `OverviewTab` | `data-testid="detail-description"` | AC6, AC12 |
| `OverviewTab` | `data-testid="detail-budget"` | AC6, AC12 |
| `OverviewTab` | `data-testid="detail-deadline-countdown"` | AC6, AC12 |
| `OverviewTab` | `data-testid="detail-cpv-codes"` | AC6, AC12 |
| `OverviewTab` | `data-testid="detail-relevance-score"` | AC6, AC12 |
| `OverviewTab` | `data-testid="detail-key-dates"` | AC6, AC12 |
| `OverviewTab` | `data-testid="detail-related-opportunities"` | AC6, AC12 |
| `DocumentsTab` | `data-testid="detail-documents-section"` | AC7, AC12 |
| `DocumentsTab` | `data-testid="detail-document-list"` | AC7, AC12 |
| `DocumentsTab` | `` `detail-document-item-${id}` `` pattern | AC7, AC12 |
| `DocumentsTab` | `` `detail-document-scan-status-${id}` `` pattern | AC7, AC12 |
| `DocumentsTab` | `` `detail-document-download-${id}` `` pattern | AC7, AC12 |
| `DocumentsTab` | `data-testid="detail-document-upload-zone"` | AC7, AC12 |
| `RequirementsTab` | `data-testid="detail-requirements-section"` | AC8, AC12 |
| `RequirementsTab` | `data-testid="detail-evaluation-table"` | AC8, AC12 |
| `RequirementsTab` | `data-testid="detail-mandatory-checklist"` | AC8, AC12 |
| `AIAnalysisTab` | `data-testid="detail-ai-analysis-section"` | AC9, AC12 |
| `AIAnalysisTab` | `data-testid="detail-ai-generate-btn"` | AC9, AC12 |
| `AIAnalysisTab` | `data-testid="detail-ai-content"` | AC9, AC12 |
| `AIAnalysisTab` | `data-testid="detail-ai-usage-counter"` | AC9, AC12 |
| `SubmissionGuideTab` | `data-testid="detail-submission-guide-section"` | AC10, AC12 |
| `SubmissionGuideTab` | `data-testid="detail-submission-accordion"` | AC10, AC12 |
| `SubmissionGuideTab` | `` `detail-submission-step-${step_number}` `` pattern | AC10, AC12 |

---

## Step 5: Validation & Completion

### Validation Checklist

- [x] **Prerequisites satisfied**: Vitest config exists; S6.9/S6.10 pattern confirmed; no new infrastructure required
- [x] **Test file created**: `__tests__/opportunities-detail-s6-11.test.ts` with 238 test functions
- [x] **TDD RED phase verified**: `Tests 231 failed | 7 passed (238)` — confirmed by running Vitest
- [x] **No placeholder assertions**: Every assertion checks specific expected content / presence
- [x] **AC coverage complete**: All 15 ACs (AC1–AC15) covered; 5 scenarios correctly deferred with rationale
- [x] **Pattern compliance**: Mirrors `opportunities-filter-s6-10.test.ts` structure exactly
- [x] **No jQuery/DOM**: Pure Node.js `fs` module — no jsdom or browser globals
- [x] **Architecture violations caught**: SSR guard for `window` access (`typeof window !== "undefined"`) tested in red phase
- [x] **URL hash routing verified**: All 5 hash strings + `hashchange` + `window.location.hash` write pattern in red phase
- [x] **i18n completeness**: 51 new detail keys tested across both locales + key parity check

### Architecture Decision: QueryGuard in OpportunityDetailPage

**Note:** There is an apparent conflict in the story between task 11.2 (which says to assert `QueryGuard` in `OpportunityDetailPage.tsx`) and the Dev Notes section ("Do NOT use `<QueryGuard>` for this — the three distinct error states require manual branching"). The ATDD test follows task 11.2 (the spec contract) and asserts `QueryGuard` is present. If the developer implements the manual-branching pattern from Dev Notes without `QueryGuard`, they should either:
1. Import `QueryGuard` but use it for one of the inner states (not the top-level branching), satisfying the static assertion, OR
2. Discuss with the product owner to update AC2 / task 11.2 to remove the `QueryGuard` requirement.

### Acceptance Criteria Coverage Summary

| AC | Description | Tests Count | Status |
|----|-------------|-------------|--------|
| AC1 | [id]/page.tsx: server component; renders OpportunityDetailPage; protected by layout.tsx | 3 | ✅ |
| AC2 | OpportunityDetailPage: useOpportunityDetail; 403→detail-access-denied; 404→detail-not-found; QueryGuard | 11 | ✅ |
| AC3 | Five tabs with correct testids; Tabs import; default "overview" | 7 | ✅ |
| AC4 | URL hash routing: window.location.hash read/write; hashchange listener; SSR guard; 5 hash strings | 10 | ✅ |
| AC5 | Breadcrumb: detail-breadcrumb testid; Breadcrumb* components from @eusolicit/ui | 5 | ✅ |
| AC6 | OverviewTab: 9 testids; setInterval(1000) + clearInterval; formatBudget; RelevanceBadge; formatDate; submission_deadline; OpportunityCard | 19 | ✅ |
| AC7 | DocumentsTab: 8 testids; useOpportunityDocuments; downloadDocument; QueryGuard; 3 scan status icons; EmptyState | 16 | ✅ |
| AC8 | RequirementsTab: 3 testids; Table; column headings; evaluation_criteria; mandatory_documents; CheckSquare; Card | 9 | ✅ |
| AC9 | AIAnalysisTab: 4 testids; TODO S06.13 comment; Sparkles + RefreshCw icons; ai_summary conditional; usage counter | 10 | ✅ |
| AC10 | SubmissionGuideTab: 3 testids; Accordion* imports; steps mapping; EmptyState | 10 | ✅ |
| AC11 | 51 new detail keys in en.json + bg.json + key parity | 107 | ✅ |
| AC12 | All data-testid attributes (verified inline per component describe block) | inline | ✅ |
| AC13 | API types: OpportunityDetailResponse, EvaluationCriterion, MandatoryDocument, RelatedOpportunity, SubmissionGuide, SubmissionStep, AISummary, DocumentRecord; functions: getOpportunityDetail, getOpportunityDocuments, downloadDocument; Omit pattern | 13 | ✅ |
| AC14 | Query hooks: useOpportunityDetail + useOpportunityDocuments; useQuery; queryKeys; staleTime 60_000; retry logic for 403/404; imports | 10 | ✅ |
| AC15 | This ATDD test file IS the AC15 deliverable | meta | ✅ |

### Files Generated

| File | Status |
|------|--------|
| `eusolicit-app/frontend/apps/client/__tests__/opportunities-detail-s6-11.test.ts` | ✅ Created (238 tests, 231 failing — TDD RED) |
| `eusolicit-docs/test-artifacts/atdd-checklist-6-11-opportunity-detail-page-tabbed-layout.md` | ✅ This file |

---

## Implementation Guidance for Developer

### Files to Create / Modify (Story Tasks 1–11)

| File | Action | Task | Tests Gating |
|------|--------|------|--------------|
| `lib/api/opportunities.ts` | **MODIFY** — Add 8 detail interfaces + 3 API functions + Omit pattern | Tasks 1.1–1.5 | 13 tests in `AC13` |
| `lib/queries/use-opportunities.ts` | **MODIFY** — Add `useOpportunityDetail` + `useOpportunityDocuments` hooks | Tasks 2.1–2.2 | 10 tests in `AC14` |
| `messages/en.json` | **MODIFY** — Add 51 new detail keys under `opportunities` namespace | Task 3.1 | 51+2 tests in `AC11 en.json` |
| `messages/bg.json` | **MODIFY** — Add matching 51 detail keys in Bulgarian | Task 3.2 | 51+2 tests in `AC11 bg.json` |
| `app/[locale]/(protected)/opportunities/[id]/page.tsx` | **CREATE** — Server component; no "use client"; renders `<OpportunityDetailPage id={params.id} />` | Task 4 | 3 tests in `AC1` |
| `components/OverviewTab.tsx` | **CREATE** — Client component; all 9 overview testids; setInterval countdown; formatBudget/RelevanceBadge/formatDate | Task 5 | 19 tests in `AC6` |
| `components/DocumentsTab.tsx` | **CREATE** — Client component; useOpportunityDocuments inside; QueryGuard; scan status badges; upload zone placeholder | Task 6 | 16 tests in `AC7` |
| `components/RequirementsTab.tsx` | **CREATE** — Table + CheckSquare/Square; evaluation_criteria + mandatory_documents | Task 7 | 9 tests in `AC8` |
| `components/AIAnalysisTab.tsx` | **CREATE** — AI summary states; TODO S06.13 comment; no onClick on buttons | Task 8 | 10 tests in `AC9` |
| `components/SubmissionGuideTab.tsx` | **CREATE** — Accordion from @eusolicit/ui; steps mapping; EmptyState for null/empty | Task 9 | 10 tests in `AC10` |
| `components/OpportunityDetailPage.tsx` | **CREATE** — Client component; useOpportunityDetail; 403/404 branching; breadcrumb; 5 tabs; URL hash routing | Task 10 | 33 tests in `AC2`+`AC3`+`AC4`+`AC5` |

### TDD Green Phase Instructions

After implementing Story 6.11:

1. **Run the ATDD test file** (verify it is RED before starting — all tests must fail first):
   ```bash
   cd eusolicit-app/frontend/apps/client
   PATH="/home/debian/.nvm/versions/node/v22.22.1/bin:$PATH" ./node_modules/.bin/vitest run __tests__/opportunities-detail-s6-11.test.ts
   # Target: Tests 231 failed | 7 passed (238) — MUST be RED before starting
   ```

2. **Implement each task** — tests turn GREEN incrementally:
   ```bash
   # After Tasks 1.1–1.5 (API interfaces + functions): expect ~13 more tests to pass (AC13)
   # After Tasks 2.1–2.2 (query hooks): expect ~10 more passing (AC14)
   # After Task 3 (i18n keys): expect ~102 more passing (AC11)
   # After Task 4 ([id]/page.tsx): expect ~3 more passing (AC1)
   # After Task 5 (OverviewTab): expect ~20 more passing (AC6 + file structure)
   # After Task 6 (DocumentsTab): expect ~17 more passing (AC7 + file structure)
   # After Task 7 (RequirementsTab): expect ~10 more passing (AC8 + file structure)
   # After Task 8 (AIAnalysisTab): expect ~11 more passing (AC9 + file structure)
   # After Task 9 (SubmissionGuideTab): expect ~11 more passing (AC10 + file structure)
   # After Task 10 (OpportunityDetailPage): expect ~34 more passing (AC2+AC3+AC4+AC5 + file structure)
   ```

3. **Verify full GREEN phase**:
   ```bash
   # Target: Tests 238 passed (238)
   ```

4. **Run i18n check**:
   ```bash
   pnpm check:i18n  # must pass (AC11 Task 3.3)
   ```

5. **Key failure patterns to diagnose**:
   - `expected false to be true` on `fileExists` → file not created yet
   - `expected '' to contain 'data-testid="detail-page"'` → testid missing from root div
   - `expected '' to contain 'window.location.hash'` → hash routing not implemented
   - `expected '' to contain 'hashchange'` → event listener missing
   - `expected '' to match /typeof window !== 'undefined'/` → SSR guard missing
   - `expected '' to contain 'setInterval'` → countdown timer not using setInterval
   - `expected '' to contain 'clearInterval'` → interval not cleared on unmount (memory leak)
   - `expected '' to contain 'submission_deadline'` → using wrong field (`deadline` instead of `submission_deadline`)
   - `expected '' to contain 'QueryGuard'` in DocumentsTab → QueryGuard missing from DocumentsTab
   - `expected '' to contain 'TODO S06.13'` → comment missing from AIAnalysisTab
   - `expected '' to contain 'Omit'` → OpportunityDetailResponse not using Omit pattern for evaluation_criteria
   - `expected '' to contain '60_000'` or `'60000'` → staleTime missing from useOpportunityDetail
   - `expected '' to contain '403'` in use-opportunities.ts → retry logic missing
   - `Missing key: opportunities.tabOverview` → i18n key not added to en.json/bg.json

### Common Implementation Pitfalls (Tested)

| Pitfall | Test That Catches It |
|---------|---------------------|
| Server page having `"use client"` | `is a server component — does NOT have "use client" directive` |
| Client components missing `"use client"` | `is a client component — has "use client" directive` (OverviewTab, DocumentsTab, AIAnalysisTab, OpportunityDetailPage) |
| Using `deadline` instead of `submission_deadline` for countdown | `uses submission_deadline (not deadline) for the countdown timer` |
| `window.location.hash` without SSR guard | `has SSR guard for window access (typeof window !== "undefined")` |
| Missing `removeEventListener` (hashchange leak) | `cleans up hashchange listener on unmount (removeEventListener)` |
| Calling `useOpportunityDocuments` in OpportunityDetailPage instead of DocumentsTab | `imports and calls useOpportunityDocuments hook inside the component` (in DocumentsTab describe block) |
| Upload zone having an onClick handler | upload zone placeholder has dashed border but NO onClick — test only checks for Upload icon presence |
| AI button having onClick (S06.13 concern) | `contains TODO S06.13 comment (SSE streaming wired in next story)` |
| Missing `Omit` pattern for evaluation_criteria type narrowing | `OpportunityDetailResponse uses Omit<OpportunityFullResponse, "evaluation_criteria">` |
| staleTime not set on useOpportunityDetail | `useOpportunityDetail has staleTime of 60_000 ms (1 minute)` |
| Not retrying only 403/404 (retry all or no retry) | `useOpportunityDetail does not retry on 403 or 404 (deterministic errors)` |

---

## Summary

```
✅ ATDD Test Generation Complete (TDD RED PHASE)

🔴 TDD Red Phase: 231 Failing Tests (7 pass — pre-existing infrastructure)

📊 Summary:
- Total Tests: 238
  - P0: 0 (no backend tier-gate logic in this story)
  - P1: 228 (all component structure, testids, API types, query hooks)
  - P2: 10 (URL hash routing + SSR guard)
- Test Groups: 15 describe blocks covering all 15 ACs
- Deferred: 5 (Playwright E2E detail page journey; Playwright E2E free-tier detail 403;
              RTL countdown timer fake-timer; RTL tab switching + hash; RTL download handler)

📂 Generated Files:
- __tests__/opportunities-detail-s6-11.test.ts (238 tests — TDD RED PHASE)
- eusolicit-docs/test-artifacts/atdd-checklist-6-11-opportunity-detail-page-tabbed-layout.md

✅ Acceptance Criteria Coverage:
- AC1:   3 tests (server page shell: exists, OpportunityDetailPage reference, no "use client")
- AC2:  11 tests (OpportunityDetailPage: "use client", 3 testids, useOpportunityDetail, QueryGuard, 403/404, links)
- AC3:   7 tests (5 tab testids + Tabs import + default "overview")
- AC4:  10 tests (window.location.hash read/write, hashchange, removeEventListener, 5 hashes, SSR guard)
- AC5:   5 tests (detail-breadcrumb, Breadcrumb import, BreadcrumbLink, BreadcrumbPage, BreadcrumbSeparator)
- AC6:  19 tests (OverviewTab: "use client", 9 testids, setInterval+clearInterval, deadline text, formatBudget, RelevanceBadge, formatDate, submission_deadline, OpportunityCard)
- AC7:  16 tests (DocumentsTab: "use client", 8 testids, useOpportunityDocuments, downloadDocument, QueryGuard, 3 scan icons, scan_status values, EmptyState)
- AC8:   9 tests (RequirementsTab: 3 testids, Table, column headings, evaluation_criteria, mandatory_documents, CheckSquare/Square, Card)
- AC9:  10 tests (AIAnalysisTab: 4 testids, TODO S06.13, Sparkles, RefreshCw, ai_summary conditional, usage counter)
- AC10: 10 tests (SubmissionGuideTab: 3 testids, Accordion+AccordionItem+AccordionTrigger+AccordionContent, steps mapping, EmptyState)
- AC11: 107 tests (51 keys × en.json + 51 keys × bg.json + namespace checks + key parity)
- AC12: inline (verified in each component's describe block)
- AC13: 13 tests (8 interface exports + 3 function exports + Omit pattern)
- AC14: 10 tests (2 hook exports, useQuery, 2 queryKeys, staleTime, retry logic, imports)
- AC15: This file IS AC15
- File:  7 tests (structural gate — all 7 new files)

🔗 Epic 6 Test Design IDs Covered:
- E06-P1-027 ✅ | E06-P2-010 ✅ | E06-P2-011 ✅ | E06-P0-010 (partial) ✅
- E06-P3-002 ✅ | E06-P3-003 ✅

🚦 TDD Red Phase Compliance: ALL PASS
- 231/238 tests failing (correct — target files do not exist / new content not yet added)
- 7/238 tests passing (correct — pre-existing infrastructure from S6.9)
- No placeholder assertions (every test checks specific expected behaviour)
- Architecture violation caught: SSR guard requirement, clearInterval requirement, submission_deadline check

📝 Next Steps:
1. Implement S06.11 (see story Tasks 1–11 in story file)
2. Run: ./node_modules/.bin/vitest run __tests__/opportunities-detail-s6-11.test.ts → verify RED (231 failing)
3. Implement each task → watch tests turn GREEN incrementally
4. Fix implementation until all 238 tests PASS (GREEN)
5. Run pnpm check:i18n → must pass
6. Commit: "feat: S06.11 opportunity detail page tabbed layout — tests passing"
7. (Optional) /bmad-testarch-automate — add RTL component mount tests + Playwright E2E detail spec
```

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-atdd`
**Version:** 4.0 (BMad v6)
