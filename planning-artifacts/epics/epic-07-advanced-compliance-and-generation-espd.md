---
epicNumber: 7
title: Advanced Compliance & Generation (ESPD)
status: draft
dependencies: [1, 2]
frsCovered: [FR-36]
nfrsRelevant: [NFR-7, NFR-15, NFR-18, NFR-19, NFR-20]
uxDrsCovered: [UX-P29, UX-P30, UX-P31, UX-P32]
sourceDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
---

## Epic 7: Advanced Compliance & Generation (ESPD)

This epic delivers a high-value, domain-specific feature for EU procurement: pre-filled European Single Procurement Document (ESPD) generation. The system maps fields from the company profile, persists user-completed responses across opportunities for re-use, validates completeness against the official ESPD schema, and exports a legally compliant XML document.

### Story 7.1: Map Company Profile Data to ESPD Fields

As a platform engineer,
I need a deterministic mapping from `client.companies` profile fields to ESPD form fields,
So that the ESPD builder can reliably auto-fill an opportunity-specific form.

**Acceptance Criteria:**

**Given** the official ESPD schema is loaded into `shared.espd_schema`
**When** the mapping function runs for a `(company_id, opportunity_id)`
**Then** every ESPD field that has a profile-driven source is auto-populated with the correct value and provenance metadata
**And** unmapped fields are returned with status `requires_user_input`
**And** the mapping is unit-tested with golden fixtures for at least 5 representative profiles.

### Story 7.2: ESPD Builder UI

As a user responding to an EU tender,
I want a guided ESPD builder UI that pre-fills my company data,
So that I can complete the form quickly with minimal manual work.

**Acceptance Criteria:**

**Given** I am working on an EU opportunity and click "Generate ESPD"
**When** the builder loads
**Then** I see the ESPD form pre-filled from my profile, segmented by Part (I--VI) with a sticky progress indicator (UX-P29)
**And** each pre-filled field shows a "from profile" badge and a tooltip with the source field and last-updated timestamp (UX-P30)
**And** I can override any pre-filled value, and overrides persist for the current ESPD instance only.

**Given** my company profile is incomplete for required ESPD fields
**When** I open the builder
**Then** I see a banner listing missing profile fields with deep-links to the profile page (UX-P31)
**And** I can save my partial ESPD progress and return later.

### Story 7.3: ESPD Validation and XML Export

As a user,
I want the system to validate and export my ESPD as official XML,
So that I can submit it through EU procurement channels.

**Acceptance Criteria:**

**Given** I have completed the ESPD builder
**When** I click "Validate"
**Then** the system runs schema validation and a business-rules check (e.g., declarations consistent with one another)
**And** validation errors are listed with anchor links into the offending fields.

**Given** validation passes
**When** I click "Export XML"
**Then** the system generates an ESPD-compliant XML file
**And** the file passes XSD validation against the official ESPD schema
**And** the export is recorded in the audit log with a hash of the file contents (UX-P32).

### Story 7.4: Cross-Opportunity ESPD Reuse

As a returning user,
I want my previous ESPD responses to be reusable as starting points for new opportunities,
So that I don't re-enter the same data each time.

**Acceptance Criteria:**

**Given** I have submitted an ESPD for opportunity A
**When** I generate an ESPD for opportunity B
**Then** I am offered the option to import my answers from the most recent matching prior ESPD
**And** imported answers are clearly tagged as "from prior ESPD" with an option to re-source from profile.

**Given** my company profile changed since the prior ESPD
**When** I import prior answers
**Then** the system flags fields where the imported answer differs from the current profile value, with both values shown for me to choose.
