---
epic: 6
title: Automated Compliance & Validation
status: draft
source: eusolicit-docs/planning-artifacts/epics.md
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
frs_covered: ["FR-34", "FR-35", "FR-36"]
nfrs_relevant: ["NFR-7", "NFR-8", "NFR-15", "NFR-23"]
ux_drs_relevant: []
---

# Epic 6: Automated Compliance & Validation

**Phase**: MVP (FR-34, FR-36) + Phase 2 enrichment (FR-35) | **Dependencies**: Epic 1 (Identity), Epic 4 (Requirements checklist), Epic 5 (Proposal drafting), Admin portal scaffold (Epic 8)

## Goal

Users can validate their proposal drafts against the AI-extracted requirements checklist (FR-22 from Epic 4) and against platform-managed regulatory frameworks (e.g., ZOP, GDPR, Horizon Europe). The platform also auto-generates pre-filled European Single Procurement Documents (ESPD) from the company profile.

## User Outcome

After this epic, a user can run a one-click compliance check that surfaces gaps inline in the proposal, an internal admin can publish updated regulatory rules that take effect for all customers immediately, and any user can generate a pre-filled ESPD XML+PDF for any opportunity without manually retyping company data.

## Epic Acceptance Criteria

- [ ] Compliance checks compare the proposal's content (and metadata) against the requirements checklist plus all active regulatory frameworks attached to the opportunity's domain.
- [ ] Compliance results render inline in the editor's inspector pane (per Epic 5's UX-DR2 split-pane pattern).
- [ ] Regulatory framework changes by an admin propagate to running checks within 60s without app restart.
- [ ] Generated ESPD documents are valid XML conforming to the EU ESPD schema and produce a faithful PDF rendering.
- [ ] All compliance and ESPD operations are tenant-isolated and audit-logged (NFR-7, FR-43).

## Stories

### Story 6.1: Proposal Compliance Validation

As a user,
I want to validate my proposal draft against the extracted requirements checklist,
So that I can ensure no mandatory elements are missing before submission.

**FR coverage**: FR-34

**Acceptance Criteria:**

**Given** I am editing a proposal that has an associated requirements checklist (Story 4.3)
**When** I click "Run Compliance Check"
**Then** the AI compares each checklist item against the draft content
**And** each item resolves to a state: `pass`, `fail`, `partial`, or `not_applicable`, with a one-sentence justification.
**And** failed/partial items are surfaced inline (anchored to the relevant section) and aggregated in the inspector "Compliance" tab.
**And** the most recent compliance run timestamp is visible; older runs remain in history but only the latest is "active".

---

### Story 6.2: Regulatory Framework Management

As a platform admin,
I want to create, version, and manage regulatory compliance frameworks (e.g., ZOP, GDPR, Horizon Europe),
So that the compliance engine always uses current legal constraints.

**FR coverage**: FR-35

**Acceptance Criteria:**

**Given** I am authenticated to the admin portal (with MFA + IP allowlist per NFR-8)
**When** I navigate to "Compliance Frameworks"
**Then** I can create a new framework or a new version of an existing framework with `effective_from` / `effective_to` dates.
**And** each framework version contains rules with id, description, severity, and a machine-readable check spec.
**And** publishing a new version makes it active for new compliance runs within 60s, while in-flight runs continue with the version they started under.
**And** frameworks are scoped by jurisdiction/domain and auto-attached to opportunities matching that domain.
**And** all framework mutations are append-only audited.

---

### Story 6.3: ESPD Generation

As a user,
I want to automatically generate a pre-filled European Single Procurement Document (ESPD) from my company profile,
So that I don't have to manually enter company details into complex forms.

**FR coverage**: FR-36

**Acceptance Criteria:**

**Given** my company profile is sufficiently complete (Story 1.5 completeness ≥ a defined threshold)
**When** I request an ESPD for a specific opportunity
**Then** the system maps profile fields to the EU ESPD XML schema
**And** any missing required fields are surfaced as a "ready-to-fix" list before download.
**And** I receive both the canonical ESPD XML and a human-readable PDF rendering.
**And** the artefact is persisted in `client.espd_documents` and downloadable via a signed URL.

## Implementation Notes

- Compliance evaluation runs through `ai-gateway` for natural-language rules and through a deterministic rules engine for structured checks (page count, mandatory attachment presence, date validity).
- Framework rules use a small DSL (or JSON Logic) so non-AI deterministic checks are auditable and testable.
- ESPD XML schema is sourced from the EU's official artefacts; vendor mappings live in `shared.espd_mappings` and are versioned alongside frameworks.
- All compliance runs emit `ComplianceRunCompleted` events to the bus for downstream notifications and analytics.
- Admin actions on frameworks are gated behind the admin portal (Epic 8) and never exposed to tenant users.
