# Epic 7: Advanced Compliance & Generation (ESPD)

This epic delivers a high-value, domain-specific feature: the ability to generate a pre-filled European Single Procurement Document (ESPD) by mapping data from a company's profile.

**FRs covered:** FR-36

## Stories

### Story 7.1: ESPD Generation from Company Profile
As a user responding to an EU tender,
I want the system to generate a pre-filled ESPD using the data from my company profile,
So that I can save significant time and reduce errors on this mandatory administrative document.

**Acceptance Criteria:**
**Given** my company profile is complete and I am working on an EU opportunity
**When** I click "Generate ESPD"
**Then** a new ESPD form is created with all possible fields (e.g., company name, VAT, legal declarations) automatically populated from my profile.
**And** I can review, complete any remaining fields, and export the final document in the official XML format.
