---
epic: 4
title: AI Tender Analysis
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-21", "FR-22", "FR-23", "FR-24", "FR-25"]
nfrs_relevant: ["NFR-2", "NFR-6", "NFR-7", "NFR-23"]
ux_drs_relevant: ["UX-DR4", "UX-DR6"]
---

# Epic 4: AI Tender Analysis

**Phase**: MVP | **Dependencies**: Epic 1 (Identity), Epic 2 (Tier gates), Epic 3 (Opportunities), `ai-gateway`, ClamAV, MinIO

## Goal

Users can upload tender documents (PDF, DOCX) and instantly receive AI-generated executive summaries, structured requirements checklists, high-risk-clause flags, and proposal score simulations — all delivered via streaming (SSE) so the user sees output begin within 500ms of request.

## User Outcome

After this epic, a user can drag a 150-page tender PDF onto an opportunity, watch a one-page executive summary stream in within 90 seconds (UX-DR6), review an extracted compliance checklist linked back to source clauses, see flagged risky clauses, and simulate a 0–100 proposal score with improvement suggestions.

## Epic Acceptance Criteria

- [ ] Document uploads are virus-scanned by ClamAV before any AI processing; infected files are quarantined and the user is notified (NFR-6).
- [ ] All AI streaming endpoints achieve TTFB < 500ms (NFR-2) measured from request receipt to first SSE chunk.
- [ ] AI generations pass through `ai-gateway` with a circuit breaker + retry wrapper and per-tenant quota check before opening the streaming response.
- [ ] Each AI output (summary, checklist item, risk flag) is linked back to source page/section for explainability.
- [ ] AI agent state is exposed via the documented `idle | requesting | streaming | complete | error | canceled` state machine (UX-DR4) and visible in the UI.
- [ ] AI analysis features are tier-gated (Starter+ for summary, Professional+ for risk flags & score simulation).

## Stories

### Story 4.1: Tender Document Upload

As a user,
I want to upload tender documents (PDF, DOCX) to the platform,
So that they can be analysed by the AI agents.

**FR coverage**: FR-24
**NFR coverage**: NFR-6, NFR-7

**Acceptance Criteria:**

**Given** I am on the opportunity detail page
**When** I drop a PDF or DOCX file (≤ 50MB) onto the upload zone
**Then** the file is uploaded to MinIO under a tenant-scoped prefix
**And** is scanned by ClamAV before any further processing.
**And** infected files are deleted, the upload is rejected, and I see an explanatory error.
**And** clean files are registered in `client.documents` with `status=ready` and become selectable for AI analysis.
**And** files are encrypted at rest (AES-256) and access requires a signed, short-lived URL.

---

### Story 4.2: AI Executive Summary Generation

As a user,
I want the AI to generate a one-page executive summary of an uploaded tender document,
So that I can grasp the core scope without reading the entire file.

**FR coverage**: FR-21
**NFR coverage**: NFR-2

**Acceptance Criteria:**

**Given** a clean tender document is registered to an opportunity
**When** I request "Generate Summary"
**Then** the `ai-gateway` checks my tier and quota, opens an SSE stream, and the first chunk arrives at the client in < 500ms.
**And** the full summary completes within 90 seconds (UX-DR6) for a 150-page document.
**And** the summary is persisted in `client.tender_analyses` and re-displayed on subsequent visits without re-running the agent.
**And** stream cancellation by the user immediately stops upstream agent execution and frees the quota.

---

### Story 4.3: Requirements Extraction and Checklist

As a user,
I want the AI to extract mandatory requirements into a structured checklist,
So that I know exactly what documents and conditions must be met.

**FR coverage**: FR-22

**Acceptance Criteria:**

**Given** a tender document is uploaded
**When** I run the requirements-extraction agent
**Then** a structured list of requirements is produced and stored in `client.requirement_checklists`
**And** each item carries `category` (eligibility / technical / financial / administrative), `severity` (mandatory / recommended), and a `source_anchor` pointing to the originating page+span.
**And** clicking an item highlights the source clause in the document viewer.
**And** the checklist becomes the input contract for Epic 6's compliance validation.

---

### Story 4.4: High-Risk Clause Flagging

As a user,
I want the AI to identify and flag high-risk clauses in the tender,
So that I can mitigate potential legal or delivery risks before bidding.

**FR coverage**: FR-23

**Acceptance Criteria:**

**Given** a tender document is uploaded
**When** the risk agent analyses it
**Then** clauses are flagged with severity (low / medium / high / critical) and a one-sentence explanation
**And** flags are displayed inline in the document viewer and aggregated in a "Risks" tab.
**And** the user can dismiss or annotate a flag without removing it from the audit record.

---

### Story 4.5: Proposal Score Simulation

As a user,
I want the system to simulate an evaluation score based on the tender's evaluation criteria,
So that I can estimate my chances of winning before submission.

**FR coverage**: FR-25

**Acceptance Criteria:**

**Given** an opportunity has machine-readable evaluation criteria (extracted in Story 4.3)
**When** I run the score simulator against my proposal draft (or a placeholder if no draft yet)
**Then** the simulator outputs an estimated score (0–100) plus 3–5 specific suggestions to improve the bid
**And** the score is stamped with the simulator version and inputs hash for reproducibility.
**And** running the simulator again uses cached output if inputs are unchanged.

## Implementation Notes

- Use SSE (`text/event-stream`) — not WebSockets — for AI streaming; quota is checked synchronously _before_ opening the StreamingResponse.
- Document parsing happens in `data-pipeline` workers; AI agents run in `ai-gateway` and consume parsed text+layout via the shared event bus.
- Tenant-scoped MinIO prefixes are enforced both by app code and IAM policy on the bucket.
- The `requirement_checklist` schema is the canonical contract consumed by Epic 6 and Epic 5 (proposal generation) — do not let it drift between epics.
- All AI calls emit traces and per-tenant token usage to the metering system for billing (Epic 2).
