# Epic 6: Document Analysis & Summarisation

Users receive automated executive summaries, extracted requirement checklists, and flagged risk insights from complex uploaded tender documents.

### Story 6.1: Generate Executive Summary
As a Reviewer,
I want a one-page AI summary of a 200-page tender,
So that I can understand the core requirements quickly.

**Acceptance Criteria:**
**Given** an uploaded tender document
**When** the AI parser processes the document
**Then** it extracts deadlines, budget, and generates a one-page summary streamed via SSE
**And** risk analysis flags unusual clauses based on a configurable risk threshold

### Story 6.2: Compliance Checklist Extraction
As a Bid Manager,
I need a structured requirements checklist,
So that I can track what needs to be fulfilled in the bid.

**Acceptance Criteria:**
**Given** a parsed tender document
**When** the requirements checklist is generated
**Then** each checklist item links to its source span in the original document
**And** items can be marked complete per proposal version
