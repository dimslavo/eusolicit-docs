# Epic 7: Proposal Generation

Users can collaboratively draft proposals using an AI-assisted split-pane editor featuring section locks, optimistic concurrency, and rich text export.

### Story 7.1: Stream a Proposal Draft
As a Contributor,
I want streaming AI drafting per content block,
So that I can rapidly author proposal sections using context.

**Acceptance Criteria:**
**Given** I am in the Split-Pane Editor
**When** I request an AI draft for a section
**Then** the draft streams via SSE using native fetch + ReadableStream
**And** an aria-live region announces streaming updates

### Story 7.2: Optimistic Locking on Save
As a collaborator,
I never want my edits silently overwritten,
So that I can work safely with multiple people.

**Acceptance Criteria:**
**Given** multiple users editing the same proposal
**When** a save conflict occurs due to a mismatched content_hash
**Then** the backend returns 409 Conflict
**And** a 3-way diff modal appears to resolve the conflict

### Story 7.3: Section-Level Lock
As a team,
I want clear "who is editing what",
So that we do not overwrite each other.

**Acceptance Criteria:**
**Given** I am editing a content block
**When** I lock the section
**Then** other users see a "Locked by [Name]" banner
**And** the lock auto-releases after 90 seconds of idle time or upon saving

### Story 7.4: Version History & Restore
As a Bid Manager,
I want to see and restore prior drafts,
So that I can revert unwanted changes.

**Acceptance Criteria:**
**Given** a proposal with multiple edits
**When** I view the version history
**Then** I can see a diff between any two versions
**And** restoring a version creates a new version pointing to the chosen content

### Story 7.5: Export PDF / DOCX
As a Bid Manager,
I want a submission-ready document,
So that I can submit the proposal to the portal.

**Acceptance Criteria:**
**Given** a completed proposal in a Professional+ tier account
**When** I click export
**Then** a ThreadPoolExecutor runs WeasyPrint for PDF and python-docx for DOCX
**And** the final file is provided for download
