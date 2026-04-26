# Epic 3: AI Intelligence & Proposal Generation

Empower users to extract requirements from tender docs, generate executive summaries, validate compliance, and draft proposals using AI.

### Story 3.1: Ingest and Extract Requirements from Tender Docs

As a Bid Manager,
I want to upload tender documents up to 100MB and have the AI extract structured requirements,
So that I don't have to read hundreds of pages to find the mandatory items.

**Acceptance Criteria:**

**Given** a PDF or DOCX file (up to 100MB)
**When** I upload it to my active workspace
**Then** the system extracts key structured requirements
**And** stores them in the workspace's vector storage vault.

### Story 3.2: AI-Generated Executive Summary

As a Bid Manager,
I want an AI-generated one-page executive summary highlighting risks and financial thresholds,
So that I can quickly assess if the opportunity is viable.

**Acceptance Criteria:**

**Given** an ingested tender document
**When** I request an executive summary
**Then** the KraftData Executive Summary Agent generates a summary outlining key risks, deadlines, and financial bounds
**And** increments the Redis usage meter for my workspace.

### Story 3.3: Automated Compliance Checklist Generation

As a Bid Manager,
I want an automated compliance checklist mapping mandatory requirements,
So that my team knows exactly what deliverables are required.

**Acceptance Criteria:**

**Given** an ingested tender document
**When** I request a compliance checklist
**Then** the system generates a trackable list of mandatory items
**And** makes it available within the workspace view.

### Story 3.4: Draft Proposal Generation via SSE

As a Technical Writer,
I want the AI to draft an initial proposal utilizing real-time streaming,
So that I can start from a solid foundation tailored to my company's profile.

**Acceptance Criteria:**

**Given** an opportunity and past context in the workspace vector store
**When** I trigger the KraftData Proposal Generator Workflow
**Then** the draft streams to the frontend via Server-Sent Events (SSE)
**And** if the connection drops, the system correctly cleans up the semaphore permit within 120s.

### Story 3.5: Validate Proposal Against Compliance Framework

As a Reviewer,
I want to validate my proposal against a specific regulatory framework,
So that I avoid disqualification due to missing mandatory sections.

**Acceptance Criteria:**

**Given** an admin-assigned regulatory framework (e.g., ZOP, 2014/24/EU) and a draft proposal
**When** I trigger the Compliance Validator Agent
**Then** it evaluates the proposal and flags any non-compliant formatting or missing sections
**And** logs the validation action in the `shared.audit_log`.