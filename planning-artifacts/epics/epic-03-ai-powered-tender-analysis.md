---
epicNumber: 3
title: AI-Powered Tender Analysis
status: draft
dependencies: [1, 2]
frsCovered: [FR-21, FR-22, FR-23, FR-24, FR-25]
nfrsRelevant: [NFR-2, NFR-7, NFR-11, NFR-21, NFR-23]
uxDrsCovered: [UX-P20, UX-P21, UX-P22, UX-P23, UX-P24, UX-P25, UX-P26, UX-P27, UX-P28]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
---

## Epic 3: AI-Powered Tender Analysis

This epic introduces the core AI "magic." Users upload tender documents (or use ingested ones) and receive time-saving AI analyses: an executive summary, a structured requirements checklist, risk-clause identification, and a score simulator. The AI gateway streams responses (SSE) for low-latency UX, persists results in `client.analyses`, and meters usage atomically against tier quotas.

### Story 3.1: Upload Tender Documents

As a user,
I want to upload tender documents (PDF, DOCX) to a specific opportunity,
So that the AI can analyze them.

**Acceptance Criteria:**

**Given** I am viewing an opportunity detail page
**When** I drag-and-drop or select files (max 50 MB each, PDF/DOCX only)
**Then** the files are uploaded to MinIO with virus-scanned via ClamAV before storage
**And** infected or invalid files are rejected with a clear error message
**And** clean files are recorded in `client.documents` linked to the opportunity
**And** an `OpportunitiesIngested` enrichment event triggers downstream analysis availability.

### Story 3.2: AI Executive Summary Generation

As a user with uploaded documents,
I want to click a button to generate an AI-powered executive summary of the tender,
So that I can quickly understand the opportunity's scope without reading the full document set.

**Acceptance Criteria:**

**Given** I have uploaded documents to an opportunity
**When** I click "Generate Summary"
**Then** the AI gateway opens an SSE stream and TTFB is < 500ms (NFR-2)
**And** the summary streams into the UI with a typing indicator until completion
**And** the result is persisted in `client.analyses` with `analysis_type='executive_summary'`
**And** my tier's monthly AI generation counter is atomically incremented.

**Given** I navigate away during streaming
**When** I return to the opportunity
**Then** the streaming continues server-side, the partial result is preserved, and on return I see the completed summary.

### Story 3.3: AI Requirements Checklist Extraction

As a user,
I want the AI to automatically extract a structured list of mandatory requirements,
So that I can use it as a compliance checklist.

**Acceptance Criteria:**

**Given** AI analysis has run on the tender documents
**When** I navigate to the Requirements tab
**Then** I see a checklist segmented by category (administrative, technical, financial)
**And** each item shows source document and page reference
**And** a real-time segmented progress indicator shows completion per category (UX-P24).

**Given** I refresh the checklist after the underlying documents change
**When** new requirements are detected
**Then** I see a diff view (added/removed/changed) and my manually entered notes are preserved (UX-P26).

**Given** I want to add an item manually
**When** I click "Add requirement"
**Then** I can author a custom checklist item that persists alongside AI-extracted ones (UX-P27).

### Story 3.4: AI High-Risk Clause Identification

As a user,
I want the AI to flag potentially high-risk or unusual clauses,
So that I can focus my legal review on the most critical parts.

**Acceptance Criteria:**

**Given** AI analysis has run on the tender documents
**When** I navigate to the Risk Analysis tab
**Then** I see a list of flagged clauses each with a severity badge (low/medium/high) using accessible icon-differentiated colour coding (UX-P20)
**And** each clause has an AI explanation in plain business language with adequate display space (UX-P22)
**And** each clause has a workflow status (open/accepted/mitigated/rejected) with a visible history log (UX-P23).

**Given** the underlying documents change
**When** I open the Risk Analysis tab
**Then** I am prompted to re-run the analysis (UX-P21).

### Story 3.5: AI Proposal Score Simulation

As a user,
I want the AI to simulate a potential score for my proposal based on the tender's evaluation criteria,
So that I can gauge win probability and identify improvement areas.

**Acceptance Criteria:**

**Given** I have a draft proposal and the tender's evaluation criteria have been extracted
**When** I click "Simulate Score"
**Then** the AI returns a simulated score (0--100) with a per-criterion breakdown
**And** each criterion includes specific suggestions for improvement
**And** the simulation result is persisted with a timestamp so I can compare against later runs.

**Given** my tier does not include score simulation (Free, Starter)
**When** I click "Simulate Score"
**Then** I see a paywall modal with an upgrade CTA and the simulation is not run.

### Story 3.6: Auto-Link Requirements to Proposal Sections

As a user drafting a proposal,
I want the AI to auto-link extracted requirements to the corresponding proposal sections,
So that I can verify each requirement is addressed.

**Acceptance Criteria:**

**Given** I have an AI-extracted requirements checklist and a proposal with sections
**When** I click "Auto-link"
**Then** each requirement is linked to its best-matching proposal section with a confidence indicator (high/medium/low) (UX-P25)
**And** I can override or remove any AI-suggested link.

**Given** I click on a requirement
**When** the link target exists
**Then** the proposal scrolls to the linked section with a brief highlight and a "back to requirement" affordance (UX-P28).
