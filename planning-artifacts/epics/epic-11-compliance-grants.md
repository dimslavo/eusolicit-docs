# Epic 11: Compliance & Grants

Users can validate proposals against strict regulatory frameworks, generate ESPD forms, and evaluate EU grant budget co-financing rules automatically.

### Story 11.1: Run Compliance Check (ZOP)
As a Bid Manager,
I want to validate against my regulatory framework,
So that I ensure the proposal meets legal standards.

**Acceptance Criteria:**
**Given** an assigned framework (e.g., ZOP)
**When** I run a compliance check
**Then** the system returns pass/fail/warning rules with severity
**And** the UI renders interactive progress rings and an accordion list with suggested remediations

### Story 11.2: ESPD Generator
As a bidder,
I want my ESPD auto-filled,
So that I save time on mandatory documentation.

**Acceptance Criteria:**
**Given** a company profile and opportunity
**When** I generate an ESPD
**Then** an XML and PDF version is produced
**And** a wizard stepper handles fields requiring confirmation

### Story 11.3: EU Grant Budget Calculator
As a grant applicant,
I want my budget validated against EU rules,
So that my budget is accurate and eligible.

**Acceptance Criteria:**
**Given** an EU grant opportunity
**When** I input my budget
**Then** total caps, co-financing rules, and eligibility windows are validated
**And** float arithmetic uses _ARITHMETIC_TOLERANCE = 0.01 for comparisons
