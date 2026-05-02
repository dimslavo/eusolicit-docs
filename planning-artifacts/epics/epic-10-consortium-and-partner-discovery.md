---
epicNumber: 10
title: Consortium & Partner Discovery
status: draft
dependencies: [1, 2, 3]
frsCovered: []
nfrsRelevant: [NFR-1, NFR-7, NFR-18, NFR-19, NFR-20]
uxDrsCovered: [UX-P33, UX-P34, UX-P35, UX-P36, UX-P37, UX-P38, UX-P39, UX-P40]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
phase: post-mvp
note: "Driven by the UX Supplement Consortium Finder and the EU-grants vision in PRD Phase 2."
---

## Epic 10: Consortium & Partner Discovery

This epic helps users assemble consortium partners for EU grant applications (especially Horizon Europe), where multi-country consortia are mandatory. It provides a searchable directory of organisations, an AI-powered partner-suggestion engine that complements the user's strengths, and a consortium workspace to coordinate joint application drafting.

### Story 10.1: Searchable Directory of Potential Consortium Partners

As a user preparing a grant application,
I want to search a directory of organisations to find potential consortium partners,
So that I can build a strong, compliant consortium.

**Acceptance Criteria:**

**Given** the consortium directory is populated (from EU CORDIS and user-registered profiles)
**When** I open Consortium Finder for a grant
**Then** I can search by expertise (CPV/keywords), country, organisation type (SME, university, NGO, large enterprise), past EU project experience, and certifications (UX-P33)
**And** results show organisation profile cards with country flag, type, expertise tags, past projects count, and a relevance score for the current grant (UX-P34).

**Given** I select an organisation
**When** I open its profile
**Then** I see full details: legal name, website, key personnel, public references, prior EU projects with role and outcome, contact channels.

### Story 10.2: AI-Powered Partner Suggestions

As a user building a consortium,
I want the AI to suggest partners that complement my team's strengths and fill gaps,
So that I can discover the best possible partners.

**Acceptance Criteria:**

**Given** I have defined the grant scope and my own organisation's role
**When** I click "Suggest Partners"
**Then** the AI returns a ranked list of recommended partners with explanations (e.g., "fills technical gap in X", "satisfies geographic requirement Y") (UX-P35)
**And** each suggestion shows a confidence indicator and a "why this partner" expandable rationale (UX-P36).

**Given** I disagree with a suggestion
**When** I click "Not interested"
**Then** the suggestion is dismissed for this grant context
**And** my feedback is recorded to refine future suggestions.

### Story 10.3: Consortium Workspace for Joint Application Drafting

As a consortium lead,
I want a shared workspace where invited partners can co-author the joint application,
So that we coordinate without exchanging files via email.

**Acceptance Criteria:**

**Given** I am the lead applicant on a grant
**When** I create a Consortium Workspace and invite partner organisations by email
**Then** invited partners receive an invitation with a workspace-scoped guest account (no full EU Solicit subscription required for guests)
**And** the workspace contains: shared proposal draft, partner contributions section, role assignments, budget split, and a shared document repository (UX-P37).

**Given** a partner accepts the invitation
**When** they enter the workspace
**Then** they can view/edit only the sections assigned to their organisation, controlled by a per-section access list (UX-P38).

### Story 10.4: Partner Outreach Tracking

As a consortium lead,
I want to track partner-outreach status (pending/declined/accepted/withdrawn),
So that I can manage the assembly process.

**Acceptance Criteria:**

**Given** I have shortlisted candidate partners
**When** I record an outreach event (email sent, call scheduled, response received)
**Then** the candidate's status updates accordingly
**And** I can see a Kanban-style board (Shortlisted / Outreach Sent / In Conversation / Accepted / Declined) with last-touch timestamps (UX-P39).

**Given** the application deadline is approaching and required partner roles are not yet filled
**When** the daily check runs
**Then** I receive an in-app and email alert listing the unfilled roles and the deadline (UX-P40).
