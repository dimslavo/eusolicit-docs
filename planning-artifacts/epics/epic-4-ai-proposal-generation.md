# Epic 4: AI Proposal Generation

**Goal:** Users can collaboratively draft proposals with real-time AI assistance, predictive scoring, and streamed text generation.
**FRs covered:** FR8, FR9

## Stories

### Story 4.1: Split-Pane Proposal Editor Interface
As a contributor,
I want a specialized editor with my document structure and AI tools side-by-side,
So that I can draft efficiently without switching contexts.

**Acceptance Criteria:**
**Given** I open a proposal draft
**When** the editor loads
**Then** I must see a left sidebar with the outline, a center Tiptap rich-text editor, and a right context panel for AI assistants
**And** the layout must be responsive

### Story 4.2: Streamed Proposal Text Generation
As a contributor,
I want the AI to draft sections of the proposal in real-time,
So that I can see the content as it is generated without waiting.

**Acceptance Criteria:**
**Given** I select a section and click "Generate Draft"
**When** the AI Gateway begins generation using KraftData RAG
**Then** the backend must stream responses using SSE (`run-stream`)
**And** the frontend must render the text immediately using native `fetch` + `ReadableStream`
**And** idle timeouts (120s) and total timeouts (600s) must be respected

### Story 4.3: AI Diff Review Block
As a contributor,
I want to see exactly what the AI has changed or suggested,
So that I can accept or reject its edits safely.

**Acceptance Criteria:**
**Given** the AI modifies existing text
**When** the generation completes
**Then** the UI must present an "AI Diff Review Block" showing deletions in red strike-through and additions in green highlight
**And** I must explicitly click "Accept" or "Reject" before changes are saved

### Story 4.4: Predictive Evaluator Scoring
As a bid manager,
I want the system to predict how evaluators will score my proposal,
So that I can make improvements before submission.

**Acceptance Criteria:**
**Given** a drafted proposal
**When** I run the predictive scorer
**Then** the system must predict a score based on the tender's evaluation criteria
**And** provide specific improvement suggestions