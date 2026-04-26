---
epic: 4
title: "Proposal Generation (SSE Streaming)"
---

# Epic 4: Proposal Generation (SSE Streaming)

Autonomously draft technical proposals using AI, providing a real-time streaming experience within a split-pane collaborative editor.

## Stories

### Story 4.1: Stream AI Proposal Draft
As a Contributor,
I want the AI to draft my proposal in real-time,
So that I don't have to wait for the full document to finish generating.

**Acceptance Criteria:**
**Given** a selected tender and user profile
**When** proposal generation is initiated and the frontend uses native `fetch` + `ReadableStream`
**Then** the backend returns a `StreamingResponse` with `Cache-Control: no-cache` and `X-Accel-Buffering: no`
**And** usage limits (UsageGate Lua script) are decremented before the generator yields HTTP 200
**And** the connection terminates gracefully with terminal events

### Story 4.2: Split-Pane Proposal Editor (UX)
As a Contributor,
I want a split-pane editor,
So that I can see the source requirements while drafting my proposal.

**Acceptance Criteria:**
**Given** the proposal editor interface
**When** viewing a draft
**Then** I see a document structure sidebar, a Tiptap center canvas, and a contextual right panel
**And** visual state indicators for section locking and real-time presence are displayed

### Story 4.3: AI Diff Review Block
As a Contributor,
I want to review AI suggestions explicitly before they are applied,
So that I maintain authorial control over the proposal.

**Acceptance Criteria:**
**Given** an AI suggestion or requirement update
**When** presented in the editor
**Then** it is shown as a Git-style diff (red deletions, green additions)
**And** it requires explicit Accept/Reject interactions (no silent overwrites)
