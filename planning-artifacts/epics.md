---
stepsCompleted: ["step-01-validate-prerequisites", "step-02-design-epics", "step-03-create-stories"]
inputDocuments: ["PRD.md", "architecture.md", "ux-spec.md", "ux-design-specification.md"]
---

# EU Solicit - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for EU Solicit, decomposing the requirements from the PRD, UX Design if it exists, and Architecture requirements into implementable stories.

## Requirements Inventory

### Functional Requirements

FR1: Ingest AOP, TED, and EU grant portals data into a centralized PostgreSQL 16 data store.
FR2: Perform AI relevance scoring against user capabilities, certifications, and financial capacities.
FR3: Send configurable email digests based on CPV, regions, budget, and deadlines.
FR4: Ingest full tender packages (PDF, DOCX) up to 100MB per file, max 500MB per package.
FR5: Generate one-page executive summaries of complex RFPs.
FR6: Extract mandatory requirements into a compliance checklist.
FR7: Flag unusual or high-risk contractual clauses.
FR8: Streamed generation (run-stream) of full technical proposals using KraftData vector stores (RAG) and user profiles.
FR9: Predict evaluator scoring and suggest improvements.
FR10: Automate compliance checks against admin-assigned regulatory frameworks (e.g., ZOP).
FR11: Manage Free, Starter, Professional, Enterprise subscription tiers gated by Stripe.
FR12: Sync calendar with Google and Outlook (Pro/Enterprise) plus an iCal feed.
FR13: Enforce Role-Based Access Control (RBAC) with company role ceiling and entity-level overrides.
FR14: Maintain immutable audit logs capturing user actions, IP, before/after values for POST/PATCH/DELETE endpoints.
FR15: Provide AI recommendation on bid viability and the ability for admin to override it.

### NonFunctional Requirements

NFR1: 99.5% uptime SLA.
NFR2: All AI generation operations must use Server-Sent Events (SSE) with run-stream, no blocking waits.
NFR3: SSE endpoints must implement idle timeout (120s), total timeout (600s), and heartbeat (15s).
NFR4: Passwords bounded at max_length=128 characters.
NFR5: hmac.compare_digest() required for all inbound webhook validations.
NFR6: AES-256 for data at rest; TLS 1.3 for transit.
NFR7: is_active checked for all token endpoints.
NFR8: Outbound calls must use circuit breakers wrapping exponential backoff.
NFR9: Idempotent Redis Streams consumption via DB constraints + SETNX.
NFR10: Atomic Lua scripts for rate/usage limits.
NFR11: Celery background tasks use SELECT FOR UPDATE SKIP LOCKED for DB queues to prevent race conditions.
NFR12: All data stored within the EU (GDPR compliance).

### Additional Requirements

- Starter Template: Next.js 14 App Router, TypeScript, Zustand, TanStack Query v5, shadcn/ui for frontend; Python 3.12+, FastAPI, PostgreSQL 16, Redis 7 for backend.
- Strict PostgreSQL schema isolation per service (client, admin, pipeline, ai, notification, shared).
- Use Redis Streams (via eusolicit-common EventPublisher/Consumer) for cross-service events.
- Protect routes using both a client-side layout `<AuthGuard>` (with 5s hydration timeout) and server-side `middleware.ts` cookie check.
- Wrap all outbound HTTP client calls with an outer circuit breaker and inner exponential retry.
- Use `SELECT ... FOR UPDATE SKIP LOCKED` for DB-driven queues and Redis `SETNX` + DB Constraints for webhook idempotency.
- All data fetching must be wrapped in `<QueryGuard>`.

### UX Design Requirements

UX-DR1: Implement "Split-Pane Proposal Editor" with a left sidebar (document outline), center Tiptap rich-text editor, and right context panel (AI assistants).
UX-DR2: Implement "AI Diff Review Block" with side-by-side or inline text view showing deletions (red strike-through) and additions (green highlight), with "Accept" and "Reject" buttons.
UX-DR3: Implement "Compliance Metric Card" with circular progress ring, fraction display, and expandable accordion for missing mandatory items.
UX-DR4: Implement Dashboard with metrics cards, recent activity feed, and upcoming deadlines calendar snippet.
UX-DR5: Implement Tender Detail View with tabbed interface (Executive Summary, Extracted Checklist, Original Documents, Risk Analysis) and file upload dropzone.
UX-DR6: Utilize shadcn/ui foundation components and Tailwind CSS with slate/indigo color tokens.
UX-DR7: Implement transient Toasts (via Sonner) for success feedback and persistent inline Alert banners (Amber/Red) for critical warnings.
UX-DR8: Implement inline form validation via Zod with error messages appearing onBlur.

### FR Coverage Map

FR1: Epic 2 - Ingest public procurement data
FR2: Epic 2 - Perform AI relevance scoring
FR3: Epic 2 - Send configurable email digests
FR4: Epic 3 - Ingest full tender packages
FR5: Epic 3 - Generate one-page executive summaries
FR6: Epic 3 - Extract mandatory requirements into a checklist
FR7: Epic 3 - Flag unusual or high-risk contractual clauses
FR8: Epic 4 - Streamed generation of technical proposals
FR9: Epic 4 - Predict evaluator scoring and suggest improvements
FR10: Epic 5 - Automate compliance checks
FR11: Epic 1 - Manage subscription tiers
FR12: Epic 6 - Sync calendar with Google/Outlook
FR13: Epic 1 - Enforce Role-Based Access Control (RBAC)
FR14: Epic 1 - Maintain immutable audit logs
FR15: Epic 6 - Provide AI recommendation on bid viability

## Epic List

### Epic 1: Onboarding & Account Management
Users can register, select subscription tiers, and manage company roles and audit logs securely.
**FRs covered:** FR11, FR13, FR14

### Epic 2: Intelligence & Discovery
Users can discover matching public tenders through automated ingestion, AI scoring, and email digests.
**FRs covered:** FR1, FR2, FR3

### Epic 3: Document Analysis & Summary
Users can upload massive tender packages to receive instant executive summaries, compliance checklists, and risk analyses.
**FRs covered:** FR4, FR5, FR6, FR7

### Epic 4: AI Proposal Generation
Users can collaboratively draft proposals with real-time AI assistance, predictive scoring, and streamed text generation.
**FRs covered:** FR8, FR9

### Epic 5: Compliance Validation
Users can validate their drafted proposals against administrative regulatory frameworks (e.g., ZOP) to ensure no mandatory requirements are missed.
**FRs covered:** FR10

### Epic 6: Workflows & Approvals
Users can manage the bid lifecycle, sync critical deadlines to their calendars, and make AI-assisted bid/no-bid decisions.
**FRs covered:** FR12, FR15


## Epic 1: Onboarding & Account Management

Users can register, select subscription tiers, and manage company roles and audit logs securely.

### Story 1.1: Project Bootstrap and Architecture Setup

As a developer,
I want to set up the initial project from the starter template,
So that I have the foundational architecture for all future stories.

**Acceptance Criteria:**

**Given** I am initializing the application
**When** I configure the monorepo
**Then** I must set up Next.js 14 App Router, TypeScript, Zustand, TanStack Query v5, and shadcn/ui for the frontend
**And** Python 3.12+, FastAPI, PostgreSQL 16, and Redis 7 for the backend
**And** establish the base schemas (client, admin, pipeline, ai, notification, shared)

### Story 1.2: User Registration and Identity

As a new user,
I want to securely register for an account using my email and password,
So that I can access the platform and configure my company profile.

**Acceptance Criteria:**

**Given** I am on the registration page
**When** I submit my email and a password
**Then** the password must be bounded at max_length=128 and hashed asynchronously
**And** inline validation (Zod) must provide errors onBlur
**And** my session must be protected by `<AuthGuard>` and `middleware.ts` cookie checks

### Story 1.2: Subscription Tier Selection with Stripe

As a registered user,
I want to select or upgrade my subscription tier (Free, Starter, Pro, Enterprise),
So that I can access features like AI proposal generation.

**Acceptance Criteria:**

**Given** I am on the billing page
**When** I choose to upgrade to Professional and complete the Stripe checkout
**Then** the webhook processor must update my tier using idempotent Redis DB constraints (`SETNX`)
**And** the API must instantly apply access rights

### Story 1.3: Role-Based Access Control (RBAC)

As a company admin,
I want to manage roles for my team members,
So that I can control access to bid creation and account settings.

**Acceptance Criteria:**

**Given** I am a company admin
**When** I navigate to the team settings
**Then** I can assign roles (admin, bid_manager, contributor, reviewer, read_only)
**And** the system must enforce company role ceilings and entity-level overrides

### Story 1.4: Immutable Audit Logging

As a platform admin,
I want the system to automatically log critical actions,
So that I have an immutable trail of security and access events.

**Acceptance Criteria:**

**Given** a user performs a POST, PATCH, or DELETE action
**When** the request is processed
**Then** the system must append an entry to the `shared.audit_log` with user actions, IP, and before/after values


## Epic 2: Intelligence & Discovery

Users can discover matching public tenders through automated ingestion, AI scoring, and email digests.

### Story 2.1: Automated Data Ingestion Pipeline

As a bid manager,
I want the platform to automatically ingest data from AOP and TED,
So that I can search a centralized database of opportunities.

**Acceptance Criteria:**

**Given** the background Celery workers are running
**When** new data is published on AOP or TED
**Then** the pipeline must ingest it into the PostgreSQL 16 `pipeline` schema using `SELECT FOR UPDATE SKIP LOCKED`
**And** circuit breakers must protect outbound calls to these portals

### Story 2.2: AI Relevance Scoring Engine

As a bid manager,
I want opportunities scored against my company capabilities,
So that I can quickly identify the most relevant tenders to pursue.

**Acceptance Criteria:**

**Given** an ingested tender
**When** the scoring engine evaluates it
**Then** it must calculate a relevance score based on my company profile, certifications, and financials
**And** store this mapping in the `ai` schema

### Story 2.3: Configurable Email Digests

As a bid manager,
I want to receive daily email digests of matched opportunities,
So that I don't have to manually log in to check for new tenders.

**Acceptance Criteria:**

**Given** I have configured my digest preferences (CPV, regions, budget)
**When** new matching tenders are ingested
**Then** the Notification service must generate and send a formatted email digest
**And** update the dispatch logs in the `notification` schema

### Story 2.4: Dashboard Metrics & Discovery

As a user,
I want a dashboard displaying key metrics and recent activity,
So that I have a high-level view of my opportunity pipeline.

**Acceptance Criteria:**

**Given** I log into the platform
**When** the dashboard loads
**Then** I must see metrics cards, a recent activity feed, and an upcoming deadlines calendar snippet
**And** the data fetching must be wrapped in `<QueryGuard>`


## Epic 3: Document Analysis & Summary

Users can upload massive tender packages to receive instant executive summaries, compliance checklists, and risk analyses.

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


## Epic 4: AI Proposal Generation

Users can collaboratively draft proposals with real-time AI assistance, predictive scoring, and streamed text generation.

### Story 4.1: Split-Pane Proposal Editor Interface

As a contributor,
I want a specialized editor with my document structure and AI tools side-by-side,
So that I can draft efficiently without switching contexts.

**Acceptance Criteria:**

**Given** I open a proposal draft
**When** the editor loads
**Then** I must see a left sidebar with the outline, a center Tiptap rich-text editor, and a right context panel for AI assistants
**And** the layout must be responsive

### Story 4.2: Streamed Proposal Text Generation

As a contributor,
I want the AI to draft sections of the proposal in real-time,
So that I can see the content as it is generated without waiting.

**Acceptance Criteria:**

**Given** I select a section and click "Generate Draft"
**When** the AI Gateway begins generation using KraftData RAG
**Then** the backend must stream responses using SSE (`run-stream`)
**And** the frontend must render the text immediately using native `fetch` + `ReadableStream`
**And** idle timeouts (120s) and total timeouts (600s) must be respected

### Story 4.3: AI Diff Review Block

As a contributor,
I want to see exactly what the AI has changed or suggested,
So that I can accept or reject its edits safely.

**Acceptance Criteria:**

**Given** the AI modifies existing text
**When** the generation completes
**Then** the UI must present an "AI Diff Review Block" showing deletions in red strike-through and additions in green highlight
**And** I must explicitly click "Accept" or "Reject" before changes are saved

### Story 4.4: Predictive Evaluator Scoring

As a bid manager,
I want the system to predict how evaluators will score my proposal,
So that I can make improvements before submission.

**Acceptance Criteria:**

**Given** a drafted proposal
**When** I run the predictive scorer
**Then** the system must predict a score based on the tender's evaluation criteria
**And** provide specific improvement suggestions


## Epic 5: Compliance Validation

Users can validate their drafted proposals against administrative regulatory frameworks (e.g., ZOP) to ensure no mandatory requirements are missed.

### Story 5.1: Automated Regulatory Framework Validation

As a bid manager,
I want the system to check my proposal against assigned frameworks like ZOP,
So that I am confident my bid is fully compliant.

**Acceptance Criteria:**

**Given** I have completed the proposal draft
**When** I click "Run Compliance Check"
**Then** the system must validate the content against the assigned regulatory framework rules
**And** identify any failures or missing mandatory items

### Story 5.2: Compliance Metric Card Component

As a user,
I want a clear visual indicator of my proposal's compliance status,
So that I instantly know if it's ready to submit.

**Acceptance Criteria:**

**Given** a compliance check has run
**When** I view the dashboard or proposal editor
**Then** I must see a Compliance Metric Card showing a circular progress ring
**And** an expandable accordion detailing any missing mandatory items


## Epic 6: Workflows & Approvals

Users can manage the bid lifecycle, sync critical deadlines to their calendars, and make AI-assisted bid/no-bid decisions.

### Story 6.1: Bid/No-Bid AI Recommendation

As a bid manager,
I want the AI to recommend whether we should pursue a bid,
So that I can make data-driven decisions quickly.

**Acceptance Criteria:**

**Given** an analyzed tender
**When** I view the decision matrix
**Then** the AI must provide a recommendation payload (win probability, risks)
**And** I must have the ability to override this decision manually

### Story 6.2: Two-Way Calendar Sync

As a professional or enterprise user,
I want my proposal deadlines synced to my Google or Outlook calendar,
So that I never miss a critical submission date.

**Acceptance Criteria:**

**Given** I am a Pro/Enterprise user
**When** I configure my calendar integration
**Then** the system must sync tender deadlines and milestones to my calendar automatically
**And** provide an iCal feed option

### Story 6.3: Upcoming Deadlines Calendar View

As a bid manager,
I want a calendar view of all upcoming deadlines in the platform,
So that I can coordinate my team's workload effectively.

**Acceptance Criteria:**

**Given** I have multiple active bids
**When** I view the deadlines calendar snippet on the dashboard
**Then** I must see all upcoming milestones and submission dates clearly marked
