---
epic: 5
title: AI-Assisted Proposal Drafting
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-26", "FR-27", "FR-28", "FR-29", "FR-30", "FR-33"]
nfrs_relevant: ["NFR-2", "NFR-15", "NFR-3"]
ux_drs_relevant: ["UX-DR2", "UX-DR4"]
---

# Epic 5: AI-Assisted Proposal Drafting

**Phase**: MVP | **Dependencies**: Epic 1 (Identity), Epic 2 (Tier gates), Epic 3 (Opportunities), Epic 4 (Tender Analysis), `ai-gateway`

## Goal

Users can create a proposal tied to an opportunity, generate an AI-assisted first draft from their company profile and the extracted tender requirements, refine it in a Tiptap rich-text editor with a contextual inspector pane (UX-DR2), maintain version history with restore, and export the final proposal to PDF and DOCX.

## User Outcome

After this epic, a user can go from a starred opportunity to a polished, exported proposal: click "Create Proposal", receive a streamed AI draft seeded with their profile + tender requirements, refine in a split-pane editor, restore any prior version, and export as PDF or DOCX for submission.

## Epic Acceptance Criteria

- [ ] Each proposal is uniquely associated with one opportunity and one workspace; cross-tenant access returns 403.
- [ ] AI draft generation streams (TTFB < 500ms, NFR-2) and is cancellable.
- [ ] Autosave guarantees zero data loss for in-flight edits — kill-the-tab tests recover the latest committed state (NFR-15).
- [ ] Version history is immutable and append-only; restoring a version creates a new "head" version, never overwrites the past.
- [ ] PDF/DOCX export preserves headings, tables, lists, page breaks, and embedded company branding.
- [ ] Editor meets WCAG 2.1 AA: full keyboard reachability, ARIA roles for toolbar, screen-reader-friendly toolbar buttons.

## Stories

### Story 5.1: Create New Proposal Workspace

As a user,
I want to create a new proposal linked to a specific opportunity,
So that I can begin drafting my response.

**FR coverage**: FR-26

**Acceptance Criteria:**

**Given** I am on an opportunity detail page
**When** I click "Create Proposal"
**Then** a new `client.proposals` row is created with `status=draft`, linked to `opportunity_id` and `workspace_id`
**And** I am routed to `/proposals/[id]` showing the split-pane Tiptap editor (left) and inspector tabs (Requirements, Compliance, Score) (right) per UX-DR2.
**And** if a draft already exists for me on that opportunity, I am routed to it instead of creating a duplicate.

---

### Story 5.2: AI First Draft Generation

As a user,
I want the AI to generate a first draft of the proposal,
So that I don't have to start from a blank page.

**FR coverage**: FR-27
**NFR coverage**: NFR-2

**Acceptance Criteria:**

**Given** I am in an empty proposal editor with a known opportunity (and ideally a Story-4.3 requirement checklist)
**When** I click "Generate Draft"
**Then** the AI gateway streams a structured draft (sections matching the tender's expected response structure) into the editor
**And** the AI state machine progresses through `requesting → streaming → complete` (UX-DR4) and is reflected in the UI.
**And** I can `accept`, `regenerate`, or `cancel` mid-stream.
**And** the draft is seeded with my company profile (Story 1.5) and the extracted requirements.
**And** generation is tier-gated (Professional+ or per-bid add-on).

---

### Story 5.3: Rich Text Editing

As a user,
I want to edit proposal content in a rich text editor with autosave,
So that I can refine formatting and text without fear of losing work.

**FR coverage**: FR-28
**NFR coverage**: NFR-15

**Acceptance Criteria:**

**Given** I am viewing a proposal draft
**When** I edit content in the Tiptap editor
**Then** changes appear immediately and autosave fires within 2s of idle.
**And** autosave failures display a non-blocking warning and retry with exponential backoff.
**And** the editor supports headings (H1–H4), bold, italic, lists, tables, links, code blocks, and inline citations.
**And** killing the browser tab and reopening loads the most recently saved state.

---

### Story 5.4: Version History and Restoration

As a user,
I want to view my proposal's version history and restore a previous version,
So that I can recover from mistakes or unwanted edits.

**FR coverage**: FR-29, FR-30

**Acceptance Criteria:**

**Given** a proposal with multiple committed versions (committed automatically every N edits or on user-triggered "Save snapshot")
**When** I open the Version History panel
**Then** I see a chronological list of versions with author, timestamp, and a short diff summary.
**And** clicking "Preview" shows the historical content read-only.
**And** clicking "Restore" creates a new head version whose content equals the selected snapshot — the prior history is preserved.
**And** restoration is logged to the audit trail (Epic 8).

---

### Story 5.5: Document Export

As a user,
I want to export my final proposal to PDF or DOCX,
So that I can submit it to the contracting authority.

**FR coverage**: FR-33

**Acceptance Criteria:**

**Given** I have a proposal in any state
**When** I select "Export → PDF" or "Export → DOCX"
**Then** a background export job runs and produces a file with preserved formatting, embedded fonts, and (if configured) workspace-branded header/footer.
**And** I am notified in-app and (optionally) by email when the export is ready.
**And** the file is downloadable via a signed URL valid for 1 hour.
**And** export jobs are isolated per tenant and rate-limited.

## Implementation Notes

- Editor is Tiptap on the frontend; backend persists ProseMirror JSON, not rendered HTML.
- Autosave uses optimistic updates + a per-proposal `version` integer with optimistic concurrency control.
- Versions are stored in `client.proposal_versions` with a content hash; identical-content edits do not create duplicate versions.
- PDF export uses headless Chromium via a worker; DOCX uses `python-docx` mapped from ProseMirror JSON.
- Inspector pane data (Requirements / Compliance / Score) is read from the same canonical sources as Epics 4 and 6 — no duplication.
