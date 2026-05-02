# Epic 3: Document Analysis & Insights

Users can upload large tender packages and receive AI-generated executive summaries, compliance checklists, and risk analyses.
**FRs covered:** FR4, FR5, FR6, FR7

## Story 3.1: Tender Package Upload

As a user,
I want to upload a tender package (up to 500MB total),
So that the system can analyze it.

**Acceptance Criteria:**

**Given** I have PDF/DOCX files
**When** I upload them
**Then** the system accepts files up to 100MB each
**And** saves them to the data store with an audit log entry

## Story 3.2: Executive Summary Generation

As a user,
I want an AI-generated executive summary of the tender,
So that I can quickly decide if it's worth pursuing.

**Acceptance Criteria:**

**Given** an uploaded tender package
**When** I request a summary
**Then** the AI Gateway generates a one-page summary
**And** the text is presented clearly with citations

## Story 3.3: Compliance Checklist Extraction

As a Bid Manager,
I want the system to extract mandatory requirements,
So that I have a clear checklist for compliance.

**Acceptance Criteria:**

**Given** an analyzed tender
**When** I view the requirements tab
**Then** a checklist of mandatory items is displayed
**And** I can mark them as complete

## Story 3.4: Risk Clause Flagging

As a Reviewer,
I want the system to flag high-risk clauses,
So that I can mitigate contractual risks early.

**Acceptance Criteria:**

**Given** the analyzed tender
**When** the risk analysis runs
**Then** unusual/high-risk clauses are highlighted in the UI
**And** the UI uses persistent inline Warning banners for critical risks