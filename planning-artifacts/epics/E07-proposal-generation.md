# E07: Proposal Generation & Document Intelligence

**Sprint**: 7-8 | **Points**: 55 | **Dependencies**: E04, E06 | **Milestone**: Demo

## Goal

Deliver EU Solicit's flagship capability: an AI-powered proposal workspace where users generate full tender responses from opportunity requirements and company profile data, refine them in a rich text editor, validate compliance, simulate evaluator scoring, and export publication-ready documents. The Proposal Generator Workflow and supporting KraftData agents are invoked exclusively through the AI Gateway (E04) with SSE streaming so that draft sections appear in real time. Combined with a reusable content-blocks library, version history, and PDF/DOCX export, this epic closes the loop from opportunity discovery to submission-ready proposal and completes the demo milestone.

## Acceptance Criteria

- [ ] Proposals can be created from an opportunity, listed, viewed with current version content, updated, and archived
- [ ] Every content edit creates an auditable version; users can list versions, view any past version, and diff two versions at section level
- [ ] Section-based auto-save (PATCH) and full save (PUT) persist content reliably without data loss
- [ ] Triggering the Proposal Generator Workflow streams section-by-section output via SSE into the frontend editor in real time
- [ ] Requirement Checklist Agent produces a structured, checkable list linked to proposal sections
- [ ] Compliance Checker Agent validates a proposal against its assigned framework and returns per-criterion pass/fail with details
- [ ] Clause Risk Analyzer Agent flags high-risk clauses with risk level and explanation
- [ ] Scoring Simulator Agent returns a per-criterion scorecard with scores, maximums, and improvement suggestions
- [ ] Pricing Assistant Agent returns a competitive pricing recommendation with market context
- [ ] Win Theme Extractor Agent identifies key positioning themes for the proposal
- [ ] Content blocks can be created, searched (full-text), filtered by category/tags, approved, and inserted into the editor at cursor position
- [ ] Proposals can be exported to PDF and DOCX with company branding, section headers, page numbers, and table of contents
- [ ] The proposal workspace provides a cohesive full-width layout: Tiptap editor center, collapsible AI/checklist/compliance/scoring panels on the sides
- [ ] Version history sidebar supports viewing past versions, side-by-side or inline diffs, and one-click rollback
- [ ] All AI agent calls go through the AI Gateway with proper error handling, loading states, and timeout management

## Stories

### S07.01: Proposal & Version DB Schema + Migrations
**Points**: 2 | **Type**: backend

Create the `proposals`, `proposal_versions`, and `content_blocks` tables in the client schema. `proposals` holds metadata (company_id, opportunity_id, title, status enum, current_version_id FK, created_by, timestamps). `proposal_versions` stores version_number, JSONB content (`{sections: [{key, title, body}]}`), created_by, and change_summary. `content_blocks` stores reusable library entries with title, category, body, tags (text[]), version, approval fields, and timestamps. Add indexes on company_id, opportunity_id, status, and content_blocks full-text search. Write Alembic migration and seed a sample proposal for dev.

---

### S07.02: Proposal CRUD API
**Points**: 3 | **Type**: backend

Implement REST endpoints: `POST /proposals` (create from opportunity, initialises first empty version), `GET /proposals` (list by company with optional opportunity_id and status filters, paginated), `GET /proposals/:id` (return proposal with current version content joined), `PATCH /proposals/:id` (update title, status), `DELETE /proposals/:id` (soft-archive, set status to `archived`). Enforce company-scoped RLS. Return 404 when proposal belongs to a different company. Write unit and integration tests for each endpoint.

---

### S07.03: Proposal Versioning API
**Points**: 3 | **Type**: backend

Implement versioning endpoints: `POST /proposals/:id/versions` (snapshot current content into a new version row, increment version_number, accept optional change_summary), `GET /proposals/:id/versions` (list all versions, newest first), `GET /proposals/:id/versions/:vid` (get specific version content). Implement `GET /proposals/:id/versions/diff?from=X&to=Y` that returns a section-level diff (added/removed/changed sections with before/after body). Implement rollback via `POST /proposals/:id/versions/:vid/rollback` which creates a new version from the target version's content. Write tests covering version ordering, diff accuracy, and rollback correctness.

---

### S07.04: Proposal Content Save API (Auto-save + Full Save)
**Points**: 2 | **Type**: backend

Implement `PATCH /proposals/:id/content/sections/:key` for single-section auto-save (update only the matching section's body in the current version's JSONB content). Implement `PUT /proposals/:id/content` for full-save (replace entire sections array in current version). Both endpoints update `updated_at`. Add optimistic locking via a content hash or version check to prevent overwrites from stale clients. Return 409 on conflict with the latest content so the frontend can reconcile. Write tests for partial update, full replace, and conflict detection.

---

### S07.05: AI Draft Generation - Backend SSE Integration
**Points**: 3 | **Type**: backend

Create `POST /proposals/:id/generate` that invokes the Proposal Generator Workflow through the AI Gateway. Send opportunity requirements and company profile as input. Proxy the AI Gateway SSE stream back to the client: each SSE event carries a section key, title, and body chunk. On stream completion, persist the full generated content as a new proposal version. Handle AI Gateway errors, timeouts, and cancellation (client disconnect). Add a `generation_status` field to proposals (idle/generating/completed/failed) to prevent duplicate triggers. Write integration tests with a mocked AI Gateway SSE stream.

---

### S07.06: Requirement Checklist Agent Integration
**Points**: 2 | **Type**: backend

Create `POST /proposals/:id/checklist/generate` that sends tender documents to the Requirement Checklist Agent via AI Gateway. Parse the structured response into checklist items (id, text, is_checked, linked_section_key, category). Store as JSONB metadata on the proposal (or a dedicated `proposal_checklists` table if volume warrants it). Implement `GET /proposals/:id/checklist` to retrieve the checklist and `PATCH /proposals/:id/checklist/items/:itemId` to toggle checked state. Write tests for generation, retrieval, and toggle.

---

### S07.07: Compliance, Risk & Scoring Agent Integrations
**Points**: 3 | **Type**: backend

Implement three endpoints that invoke KraftData agents through the AI Gateway:
1. `POST /proposals/:id/compliance-check` - sends proposal content + assigned compliance framework to Compliance Checker Agent, returns per-criterion pass/fail with details, stores results on proposal.
2. `POST /proposals/:id/clause-risk` - sends uploaded contract documents to Clause Risk Analyzer Agent, returns flagged clauses with risk level (low/medium/high) and explanation.
3. `POST /proposals/:id/scoring-simulation` - sends proposal content to Scoring Simulator Agent, returns per-criterion scorecard (criterion, score, max_score, suggestion).

All results are persisted as proposal metadata. Add `GET` counterparts to retrieve latest results. Write tests with mocked agent responses.

---

### S07.08: Pricing Assistant & Win Theme Agent Integrations
**Points**: 2 | **Type**: backend

Implement two endpoints:
1. `POST /proposals/:id/pricing-assist` - sends opportunity details and historical data context to Pricing Assistant Agent via AI Gateway, returns recommended price, market range (min/median/max), and justification text. Store result on proposal.
2. `POST /proposals/:id/win-themes` - sends opportunity requirements and company strengths to Win Theme Extractor Agent, returns ranked list of key themes with descriptions. Store result on proposal.

Add `GET` endpoints to retrieve latest results. Write tests with mocked agent responses.

---

### S07.09: Content Blocks CRUD & Search API
**Points**: 2 | **Type**: backend

Implement REST endpoints for the reusable content library: `POST /content-blocks` (create with title, category, body, tags), `GET /content-blocks` (list with pagination, filter by category and tags query params), `GET /content-blocks/search?q=` (full-text search across title and body using PostgreSQL tsvector), `GET /content-blocks/:id`, `PATCH /content-blocks/:id` (update fields, increment version), `POST /content-blocks/:id/approve` and `/unapprove` (set approval fields), `DELETE /content-blocks/:id`. All scoped to company via RLS. Write tests covering CRUD, search relevance, and approval workflow.

---

### S07.10: Document Export API (PDF & DOCX)
**Points**: 3 | **Type**: backend

Implement `POST /proposals/:id/export` with `format` body param (`pdf` or `docx`). For PDF, use reportlab to render proposal content with company branding (logo, colors from company profile), section headers, page numbers, and auto-generated table of contents. For DOCX, use python-docx with styled templates for the same formatting. Return the generated file as a streaming download with appropriate Content-Type and Content-Disposition headers. Handle large proposals gracefully. Write tests that verify generated files are valid PDF/DOCX and contain expected section titles.

---

### S07.11: Proposal Workspace Page Layout & Navigation
**Points**: 3 | **Type**: frontend

Build the proposal workspace page at `/proposals/:id`. Implement the full-width layout: collapsible left panel (requirement checklist, content blocks library), center Tiptap editor area, collapsible right panel (AI generation, compliance, scoring, pricing, win themes). Add a top toolbar with proposal title (editable inline), status badge, version indicator, save status, and action buttons (Generate, Check Compliance, Simulate Score, Export). Implement panel toggle buttons and responsive collapse behavior. Wire up React Query for proposal data fetching and cache management. Handle loading and error states.

---

### S07.12: Tiptap Rich Text Editor with Section-Based Editing
**Points**: 3 | **Type**: frontend

Integrate Tiptap editor configured for section-based proposal editing. Each proposal section renders as a distinct block with a section header. Implement formatting toolbar (bold, italic, headings, lists, tables, links). Add auto-save: debounce content changes (1.5s), call PATCH endpoint for the active section, show save indicator (saving/saved/error). Implement full-save on Cmd+S. Handle optimistic locking conflicts by showing a reconciliation dialog. Support cursor position tracking for content block insertion.

---

### S07.13: AI Draft Generation Panel with SSE Streaming
**Points**: 3 | **Type**: frontend

Build the AI draft generation panel in the right sidebar. "Generate Draft" button triggers the backend SSE endpoint. Use EventSource or fetch with ReadableStream to consume SSE events. As each section streams in, render it progressively in the editor (typewriter effect or section-by-section reveal). Show a progress indicator listing sections with checkmarks as they complete. Provide "Accept All" (keep entire draft), "Accept Section" (keep individual sections), and "Discard" actions. Disable editor modifications during active generation. Handle stream errors and cancellation (user clicks "Stop"). Update proposal version after acceptance.

---

### S07.14: Requirement Checklist & Compliance Panels
**Points**: 2 | **Type**: frontend

Build the requirement checklist panel in the left sidebar. Render checklist items as checkboxes grouped by category. Each item is clickable to scroll the editor to the linked section. Show a completion progress bar (checked / total). Implement check/uncheck with optimistic updates.

Build the compliance check results panel in the right sidebar. Trigger button invokes compliance check. Display per-criterion results as pass (green) / fail (red) indicators with expandable detail text. "Fix" action on failed criteria scrolls the editor to the relevant section. Show loading spinner during check execution.

---

### S07.15: Scoring Simulator, Pricing & Win Themes Panels
**Points**: 2 | **Type**: frontend

Build three right-sidebar panels:
1. **Scoring Simulator**: trigger button, results displayed as a radar chart (using Recharts) showing per-criterion scores against maximums, plus a table with criterion, score, max, and improvement suggestion. Highlight criteria below 70% in amber.
2. **Pricing Assistant**: trigger button, results displayed as recommended price (large text), market range bar visualization (min/median/max with proposal price marker), and justification text paragraph.
3. **Win Themes**: trigger button, results displayed as a ranked list of theme cards with title and description, draggable to reorder priority.

All panels show loading states and handle errors gracefully.

---

### S07.16: Version History, Content Blocks Library & Export Dialog
**Points**: 3 | **Type**: frontend

Build three components:
1. **Version history sidebar**: toggled from toolbar. Lists versions with version number, date, author, and change summary. Click to preview a past version (read-only in editor). "Compare" button opens a diff viewer showing section-level changes (side-by-side or inline toggle, insertions in green, deletions in red). "Rollback" button with confirmation dialog creates a new version from the selected version.
2. **Content blocks library modal**: opened from toolbar or keyboard shortcut. Search bar with real-time full-text search, category filter chips, tag filter. Results show title, preview snippet, approval badge. "Insert" button inserts the block body at the current cursor position in the editor.
3. **Export dialog**: format selector (PDF/DOCX) with icons, company branding preview thumbnail, "Download" button that triggers the export API and downloads the file. Show progress indicator for large exports.
