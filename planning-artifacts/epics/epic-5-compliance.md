---
epic: 5
title: "Compliance Validation"
---

# Epic 5: Compliance Validation

Ensure generated proposals meet all administrative and regulatory frameworks through automated checks and visual metrics.

## Stories

### Story 5.1: Validate Proposal against Frameworks
As a Bid Manager,
I want the system to check my drafted proposal against the assigned compliance framework (e.g., ZOP),
So that I can ensure the bid will not be rejected for administrative errors.

**Acceptance Criteria:**
**Given** a drafted proposal and assigned regulatory framework
**When** the compliance check runs
**Then** the compliance checker agent analyzes the document
**And** the frontend displays pass/fail criteria and specific improvement suggestions

### Story 5.2: Compliance Metric Card (UX)
As a Bid Manager,
I want an instant visual summary of proposal compliance,
So that I know the readiness status at a glance.

**Acceptance Criteria:**
**Given** a proposal with validation results
**When** viewing the dashboard or editor header
**Then** a circular progress ring displays the compliance percentage
**And** a fraction shows the number of requirements met
**And** an expandable accordion details any missing mandatory items
