---
epicNumber: 4
title: AI-Assisted Proposal Drafting
status: draft
dependencies: [1, 2, 3]
frsCovered: [FR-26, FR-27, FR-28, FR-29, FR-30, FR-33, FR-34]
nfrsRelevant: [NFR-1, NFR-2, NFR-3, NFR-7, NFR-15, NFR-18, NFR-19, NFR-20]
uxDrsCovered: [UX-P17, UX-P18, UX-P19]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
---

## Epic 4: AI-Assisted Proposal Drafting

This epic enables users to act on the insights from Epic 3. It covers creating a proposal, generating an AI-assisted first draft, editing in a Tiptap rich-text editor with autosave, viewing/restoring version history, validating against the auto-generated compliance checklist, and exporting the final document to PDF and DOCX. It delivers Elena's "Drafting" arc end-to-end for the MVP single-user flow.

### Story 4.1: Create a New Proposal

As a user,
I want to create a new proposal associated with a specific opportunity,
So that I can begin the drafting process.

**Acceptance Criteria:**

**Given** I am on an opportunity detail page
**When** I click "Create Proposal"
**Then** a `client.proposals` record is created with `(company_id, opportunity_id, created_by_user_id)`
**And** I am routed to the proposal editor with an empty document
**And** the proposal title defaults to the opportunity title with a timestamp suffix.

**Given** a proposal already exists for this `(company_id, opportunity_id)`
**When** I click "Create Proposal"
**Then** I am routed to the existing proposal (no duplicate is created).

### Story 4.2: AI-Assisted First Draft Generation

As a user starting a new proposal,
I want the AI to generate a structured first draft based on the tender requirements and my company profile,
So that I have a strong starting point.

**Acceptance Criteria:**

**Given** I have a new (empty) proposal and the tender has been analysed in Epic 3
**When** I click "Generate AI Draft"
**Then** the AI gateway streams a coherent draft into the editor section by section, with TTFB < 500ms
**And** the draft addresses all major tender sections (executive summary, methodology, team, timeline, financials)
**And** the draft personalises content using my company profile (name, capabilities, references)
**And** my tier's AI generation quota is decremented atomically.

**Given** generation fails mid-stream (network/AI provider error)
**When** the failure is detected
**Then** the partial draft is preserved
**And** I see a "Resume generation" button that continues from the last completed section.

### Story 4.3: Rich Text Editor for Proposals

As a user,
I want a rich text editor to write and format my proposal,
So that I can produce a professional document.

**Acceptance Criteria:**

**Given** I am in the proposal editor (Tiptap)
**When** I use the toolbar
**Then** I can apply bold, italic, underline, headings (H1--H4), bulleted/numbered lists, tables, links, images, and code blocks
**And** content is autosaved every 5 seconds and on blur, with an "All changes saved" indicator
**And** the editor is fully keyboard navigable (NFR-19) and screen-reader accessible (NFR-20).

**Given** my browser loses connection
**When** I continue editing
**Then** changes are queued in local storage and synced on reconnect
**And** I see an "Offline -- changes will sync when reconnected" status.

### Story 4.4: Version History for Proposals

As a user editing a proposal,
I want a version history of my drafts,
So that I can view and restore previous versions.

**Acceptance Criteria:**

**Given** I am in the proposal editor
**When** I open the Version History panel
**Then** I see a chronological list of saved versions with timestamp, author, and a short change summary
**And** I can preview any version in read-only mode.

**Given** I select a previous version
**When** I click "Restore"
**Then** the current draft becomes a new version (the current state is never lost)
**And** the selected version's content becomes the active draft.

**Given** the version count exceeds 100
**When** the next version is created
**Then** versions older than 90 days are pruned (except major-tagged versions and the original AI draft).

### Story 4.5: Export Proposal to PDF and DOCX

As a user with a completed proposal,
I want to export the final document to PDF and DOCX,
So that I can submit it via official channels.

**Acceptance Criteria:**

**Given** I have a completed proposal
**When** I click "Export" and choose PDF or DOCX
**Then** the export job runs server-side and produces a file with consistent typography, page numbers, headers/footers, and the company logo
**And** the export honours the proposal's structure (sections, tables, images)
**And** the exported file is downloaded within 10 seconds for proposals up to 100 pages.

**Given** an export job exceeds 60 seconds
**When** the timeout fires
**Then** the user is notified by email/in-app once the file is ready
**And** the file is available from a "Recent exports" panel for 7 days.

### Story 4.6: Proposal Validation Against Compliance Checklist

As a user,
I want to validate my proposal against the AI-generated compliance checklist,
So that I can ensure all mandatory requirements are addressed before submission.

**Acceptance Criteria:**

**Given** I have a draft proposal and an extracted requirements checklist
**When** I click "Run validation"
**Then** the system cross-references each checklist item against the proposal content and marks each as satisfied / partially satisfied / missing
**And** any missing requirements are highlighted with a deep-link into the editor at the relevant section (or "no relevant section yet" if absent)
**And** the validation report is timestamped and stored for audit.

**Given** I change the proposal after a validation run
**When** I re-run validation
**Then** the previous report is preserved as historical and a new report is generated.

### Story 4.7: Pricing Assistant for Proposals

As a user drafting financials,
I want an AI-assisted Pricing Assistant that suggests pricing based on similar past tenders and market data,
So that my financial section is competitive and well-supported.

**Acceptance Criteria:**

**Given** I open the Pricing Assistant in the proposal editor
**When** market data is available
**Then** I see a confidence indicator reflecting market data quality (UX-P17)
**And** when I adjust pricing parameters (e.g., margin, scope), recalculation is debounced with a non-blocking loading overlay (UX-P18).

**Given** I click "Insert pricing into proposal"
**When** an existing pricing table is detected in the proposal
**Then** the system offers to replace it (with a diff preview) rather than appending duplicates (UX-P19).
