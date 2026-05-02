# Epic 4: AI Proposal Generation

Users can stream real-time AI-drafted proposals based on tender requirements, user profiles, and predict evaluator scores.
**FRs covered:** FR8, FR9

## Story 4.1: Proposal Draft Initialization

As a Contributor,
I want to start a new proposal draft using a template,
So that I don't have to start from scratch.

**Acceptance Criteria:**

**Given** I am in a Project Workspace
**When** I click "Start Proposal"
**Then** a Split-Pane Proposal Editor is opened
**And** it includes a left sidebar, center Tiptap canvas, and right context panel

## Story 4.2: SSE Streamed Proposal Drafting

As a Contributor,
I want the AI to stream the draft proposal text into the editor,
So that I can see the content as it's generated.

**Acceptance Criteria:**

**Given** I request AI drafting for a section
**When** the backend processes it
**Then** the text is streamed via SSE using ReadableStream
**And** AI Diff Review Block allows me to accept/reject changes

## Story 4.3: Evaluator Score Prediction

As a Bid Manager,
I want to see a predicted evaluator score for my draft,
So that I can improve weak sections.

**Acceptance Criteria:**

**Given** a drafted proposal
**When** I run the scoring simulator
**Then** it returns a predicted score
**And** provides specific improvement suggestions in the right inspector panel