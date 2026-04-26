# Epic 3: Document Analysis & Summary

**Goal:** Users can upload massive tender packages to receive instant executive summaries, compliance checklists, and risk analyses.
**FRs covered:** FR4, FR5, FR6, FR7

## Stories

### Story 3.1: Large File Upload and Storage
As a reviewer,
I want to upload massive tender packages (PDF, DOCX),
So that the platform can analyze the documents.

**Acceptance Criteria:**
**Given** I am on the tender detail view
**When** I upload a file up to 100MB (max 500MB per package)
**Then** the system must successfully store the document
**And** provide a progress bar and scanning status indicator

### Story 3.2: AI Executive Summary Generation
As a reviewer,
I want an AI-generated executive summary of the tender,
So that I can quickly understand the requirements without reading 200+ pages.

**Acceptance Criteria:**
**Given** a successfully uploaded tender document
**When** I request a summary
**Then** the AI Gateway service must stream the generated summary
**And** the UI must present it in a tabbed interface alongside the original documents

### Story 3.3: Compliance Checklist Extraction
As a bid manager,
I want the system to extract all mandatory requirements from the tender,
So that I have a clear checklist of what is needed to comply.

**Acceptance Criteria:**
**Given** an analyzed tender document
**When** the extraction is complete
**Then** the system must generate a checklist of mandatory items
**And** display it in the Extracted Checklist tab

### Story 3.4: High-Risk Clause Analysis
As a reviewer,
I want the system to highlight high-risk contractual clauses,
So that I am aware of critical risks like unlimited liability before bidding.

**Acceptance Criteria:**
**Given** an analyzed tender document
**When** the risk analysis is complete
**Then** the system must flag unusual or high-risk clauses
**And** display them in the Risk Analysis tab with explanations