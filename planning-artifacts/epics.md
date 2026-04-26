---
stepsCompleted:
  - step-01-validate-prerequisites
  - step-02-design-epics
  - step-03-create-stories
  - step-04-final-validation
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/architecture.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
  - eusolicit-docs/planning-artifacts/gap-coverage-matrix.md
epicFiles:
  - epics/E01-infrastructure-foundation.md
  - epics/E02-authentication-identity.md
  - epics/E03-frontend-shell-design-system.md
  - epics/E04-ai-gateway-service.md
  - epics/E05-data-pipeline-ingestion.md
  - epics/E06-opportunity-discovery.md
  - epics/E07-proposal-generation.md
  - epics/E08-subscription-billing.md
  - epics/E09-notifications-alerts-calendar.md
  - epics/E10-collaboration-tasks-approvals.md
  - epics/E11-grants-compliance.md
  - epics/E12-analytics-admin-platform.md
  - epics/E14-multi-client-workspace.md
  - epics/E15-per-bid-sku-pro-plus-tier.md
  - epics/E16-slack-teams-notifications.md
  - epics/E17-crm-integrations.md
  - epics/E18-trust-center-compliance.md
  - epics/E19-outcome-telemetry-renewal-proof.md
  - epics/E20-nps-reviews-onboarding.md
  - epics/E21-platform-reliability-99-9-sla.md
amendmentSource:
  research: planning-artifacts/research/market-eu-govwin-tender-intelligence-research-2026-04-25.md
  prdAmendment: planning-artifacts/prd-amendment-2026-04-25.md
  architectureEvaluation: planning-artifacts/architecture-evaluation-2026-04-25.md
  sprintChangeProposal: planning-artifacts/sprint-change-proposal-2026-04-25-v4.md
project_name: EU Solicit
date: '2026-04-25'
status: canonical
---

# eusolicit - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for eusolicit, decomposing the requirements from the PRD, UX Design if it exists, and Architecture requirements into implementable stories.

## Requirements Inventory

### Functional Requirements

FR1: Multi-source automated crawling (AOP, TED, EU grant portals).
FR2: Searchable and filterable opportunity listings.
FR3: Tier-based visibility (Free tier restricted to metadata; Paid tiers access full data).
FR4: AI summary generation (executive summaries of tender docs).
FR5: Smart relevance matching and configurable email alerts.
FR6: Competitor tracking and pipeline forecasting (Professional+).
FR7: AI document parser extracting requirements, criteria, deadlines.
FR8: Auto-generated requirement checklists mapping mandatory items.
FR9: Clause risk flagging for legal review.
FR10: Multi-language processing (BG, EN, DE, FR, RO).
FR11: AI draft generator utilizing template libraries and institutional memory.
FR12: Scoring simulator to predict evaluator scores and suggest improvements.
FR13: Compliance validator for pre-submission checks.
FR14: Pricing assistant based on historical award data.
FR15: Collaborative rich-text editor (Tiptap) with version control and pessimistic locking.
FR16: Document export (PDF, DOCX) and reusable content blocks.
FR17: Entity-level RBAC (roles: bid manager, technical writer, etc.).
FR18: Task orchestration with DAG dependencies.
FR19: Approval workflows and bid/no-bid decision scoring.
FR20: ROI and preparation time tracking.
FR21: Grant eligibility matcher and AI-assisted budget builder.
FR22: Consortium finder and logframe generator.
FR23: Reporting template generator for post-award requirements.
FR24: Admin-configurable compliance frameworks (ZOP, EU directives).
FR25: AI framework auto-suggestion and regulation tracking.
FR26: ESPD auto-fill from company profiles.
FR27: Immutable audit trails for all mutations.
FR28: Freemium model with Free, Starter, Professional, and Enterprise tiers.
FR29: Stripe integration (Checkout, Customer Portal, Usage Metering, VAT).
FR30: 14-day free trial and per-bid add-ons.

### NonFunctional Requirements

NFR1: 99.5% uptime.
NFR2: API latency (p95) < 200ms.
NFR3: Streaming TTFB < 500ms.
NFR4: Resilience via two-layer resilience (circuit breaker + exponential backoff) for all external HTTP calls.
NFR5: Security: HMAC/ECDSA signature validation for webhooks.
NFR6: Security: Passwords bounded to max_length=128 (bcrypt in executor).
NFR7: Security: JWT auth with User.is_active enforcement.
NFR8: Security: Dual-layer frontend auth guard.
NFR9: Data Isolation: PostgreSQL schema isolation per service.
NFR10: Data Isolation: API endpoints scoped to company_id.
NFR11: Integrations: KraftData Agentic AI via REST/SSE.
NFR12: Integrations: Stripe billing.
NFR13: Observability: Prometheus metrics (/metrics), structlog.
NFR14: Observability: Audit trail table for all POST/PUT/PATCH/DELETE mutations.
NFR15: System Architecture: Python 3.12+ (FastAPI, asyncpg), Next.js 14 App Router, Zustand, TanStack Query, shadcn/ui.

### Additional Requirements

- [Starter Template Requirement] Architecture specifies Next.js 14 App Router with strict TypeScript for Frontend, Python 3.12+ FastAPI for Backend.
- [Database Requirement] PostgreSQL 16 with schema-level isolation per microservice.
- [Event Bus Requirement] Redis 7 via Redis Streams for inter-service communication and Celery for async execution.
- [Security Implementation Requirement] Optimistic Locking for Collaboration using content_hash column (SHA-256) to prevent mid-air collisions.
- [Integration Requirement] KraftData / Sirma AI requires strictly enforced timeout, idle timeout (120s), and heartbeat (15s).
- [Infrastructure Setup Requirement] Docker Compose for local development; Helm 3 + Kubernetes / Terraform for production.
- [API Design Requirement] Server-Sent Events (SSE) used for AI Gateway streaming.

### UX Design Requirements

UX-DR1: [Design token work] Neutral Scale using Tailwind slate (slate-50 to slate-900), Primary Accent Indigo (#4f46e5).
UX-DR2: [Component proposal] Split-Pane Proposal Editor (Left Sidebar, Center Tiptap canvas, Right Sidebar for context panels).
UX-DR3: [Component proposal] AI Diff Review Block showing deletions in red strike-through and additions in green highlight with Accept/Reject buttons.
UX-DR4: [Component proposal] Compliance Metric Card with prominent circular progress ring and expandable accordion for missing items.
UX-DR5: [Interaction pattern] Contextual Command Menus (Cmd+K) and Slash Commands (Notion-style) in Tiptap editor.
UX-DR6: [Visual standardization] Semantic Status Badging (Green = Success, Amber = Warning, Red = Critical, Blue = Informational).
UX-DR7: [Accessibility requirement] Focus management between panes and clear ARIA announcements for AI panels opening/closing.
UX-DR8: [Accessibility requirement] Support prefers-reduced-motion to disable sliding sheet animations and transition instantly.
UX-DR9: [Responsive design requirement] Desktop-first strategy with breakpoints: sm (640px) for tablet review, md (768px) multi-column, lg (1024px) desktop sidebar, xl (1280px) 3-pane split-editor view, 2xl (1536px) max width constraint.
UX-DR10: [Loading states pattern] Skeleton Loading matching structure instead of generic spinners to prevent layout shift.

### FR Coverage Map

{{requirements_coverage_map}}

## Epic List

{{epics_list}}

## Epic 1: Opportunity Discovery & Intelligence

Users can discover, filter, and track public procurement and grant opportunities across multiple sources with AI-assisted summaries and relevance matching.

### Story 1.1: Automated Multi-Source Crawling Pipeline

As a Platform Admin,
I want to configure automated crawlers for AOP, TED, and EU grant portals,
So that opportunities are continuously ingested into the system.

**Acceptance Criteria:**

**Given** the crawling pipeline is active
**When** a new tender is published on AOP
**Then** the system ingests the tender metadata
**And** saves it to the central database.

### Story 1.2: Searchable Opportunity Listings (Free Tier)

As a Free Explorer,
I want to search and filter opportunity listings by CPV, region, and deadline,
So that I can find relevant tenders.

**Acceptance Criteria:**

**Given** I am an authenticated Free Explorer
**When** I navigate to the opportunities page
**Then** I see a paginated list of tenders
**And** I can filter the results by specific criteria.

### Story 1.3: Tier-Based Visibility Enforcement

As a System,
I want to restrict full document access to paid tiers,
So that free users are incentivized to upgrade.

**Acceptance Criteria:**

**Given** a Free Explorer views a tender detail page
**When** they attempt to download the full documentation
**Then** the system blocks the download
**And** displays a prompt to upgrade to a paid tier.

### Story 1.4: AI Executive Summary Generation

As a Starter User,
I want to generate an AI summary of a tender,
So that I can quickly qualify the opportunity.

**Acceptance Criteria:**

**Given** a tender with full documentation exists
**When** I click "Generate Summary"
**Then** the system invokes the KraftData Agent
**And** streams the summary back to the UI in real-time.

### Story 1.5: Smart Relevance Matching & Alerts

As a Professional User,
I want to set up relevance profiles and email alerts,
So that I am notified of matching opportunities automatically.

**Acceptance Criteria:**

**Given** I have saved a relevance profile
**When** a new matching opportunity is ingested
**Then** the system flags it as a match
**And** sends me an email notification.

### Story 1.6: Pipeline Forecasting & Competitor Tracking

As a Professional User,
I want to track competitors and forecast my bid pipeline,
So that I can manage my resources effectively.

**Acceptance Criteria:**

**Given** I have added competitors to my profile
**When** I view the dashboard
**Then** I see a pipeline forecast visualization
**And** an analysis of competitor win rates on similar tenders.

## Epic 2: AI Document Analysis & Extraction

Users can upload complex tender documents and receive instant AI-extracted requirements, mandatory checklists, and risk flags across multiple languages.

### Story 2.1: AI Document Parsing & Extraction

As a Professional User,
I want the AI to parse tender documents across multiple languages,
So that deadlines and criteria are automatically extracted.

**Acceptance Criteria:**

**Given** a multi-language tender document is uploaded
**When** the document analysis is triggered
**Then** the system extracts key criteria and deadlines
**And** translates them to the user's preferred language.

### Story 2.2: Auto-Generated Requirement Checklists

As a Professional User,
I want the system to generate a compliance checklist from the tender,
So that I don't miss mandatory items.

**Acceptance Criteria:**

**Given** the AI document parser has run
**When** I view the tender requirements
**Then** I see a structured checklist of mandatory items
**And** I can mark them as complete.

### Story 2.3: Clause Risk Flagging

As a Bid Manager,
I want the AI to flag risky legal clauses,
So that I can review them before committing to a bid.

**Acceptance Criteria:**

**Given** the AI document parser has run
**When** I view the risk assessment section
**Then** high-risk clauses (e.g., unlimited liability) are highlighted
**And** the system provides a rationale for the risk flag.

## Epic 3: Proposal Generation & Optimization

Users can collaboratively draft, optimize, and export highly competitive proposals using AI assistance, scoring simulations, and rich-text editing.

### Story 3.1: Split-Pane Collaborative Tiptap Editor

As a Technical Writer,
I want to draft proposals in a split-pane editor with version control,
So that I can see requirements while writing.

**Acceptance Criteria:**

**Given** I am drafting a proposal
**When** I open the editor
**Then** I see the requirements on the side
**And** I can use slash commands in the Tiptap canvas.

### Story 3.2: AI Draft Generation

As a Technical Writer,
I want the AI to generate a first draft using template libraries,
So that I have a starting point for my proposal.

**Acceptance Criteria:**

**Given** a set of requirements and company profile
**When** I click "Generate Draft"
**Then** the AI creates a structured draft
**And** inserts it into the editor.

### Story 3.3: Pessimistic Locking & Optimistic Concurrency

As a Bid Manager,
I want to lock sections while editing,
So that I don't overwrite my colleague's work.

**Acceptance Criteria:**

**Given** another user is editing a section
**When** I attempt to edit the same section
**Then** the section is visually locked
**And** I cannot make changes until they unlock it.

### Story 3.4: Scoring Simulator & Pricing Assistant

As a Bid Manager,
I want to simulate my proposal score and get pricing suggestions,
So that I can optimize my bid.

**Acceptance Criteria:**

**Given** a completed draft
**When** I run the scoring simulator
**Then** the system predicts the evaluator score
**And** suggests pricing adjustments based on historical data.

### Story 3.5: Pre-submission Compliance Validator

As a Reviewer,
I want to run a compliance check against the draft,
So that I ensure all mandatory items are met.

**Acceptance Criteria:**

**Given** a finalized draft
**When** I run the compliance validator
**Then** the system generates a Compliance Metric Card
**And** highlights any missing mandatory items.

### Story 3.6: Document Export & Reusable Blocks

As a Bid Manager,
I want to export the proposal to PDF/DOCX and save reusable blocks,
So that I can submit the bid and reuse content.

**Acceptance Criteria:**

**Given** a validated proposal
**When** I click Export
**Then** the system generates a formatted PDF or DOCX
**And** I can save specific sections to my content library.

## Epic 4: Collaboration & Workflow Orchestration

Teams can collaborate efficiently with role-based access control, task dependencies, approval workflows, and time tracking.

### Story 4.1: Entity-Level RBAC & Team Management

As an Enterprise Admin,
I want to assign roles like bid manager and technical writer,
So that access is securely managed.

**Acceptance Criteria:**

**Given** a team of users
**When** I assign a user the Technical Writer role
**Then** they can edit proposals
**And** cannot submit final bids or view billing.

### Story 4.2: Task Orchestration & DAG Dependencies

As a Bid Manager,
I want to create tasks with dependencies,
So that the team knows what to work on and in what order.

**Acceptance Criteria:**

**Given** a complex bid
**When** I create tasks and set dependencies
**Then** users cannot start a blocked task
**And** they are notified when dependencies are resolved.

### Story 4.3: Approval Workflows & Bid/No-Bid Scoring

As a Reviewer,
I want to use approval workflows to formalize bid decisions,
So that we only pursue viable opportunities.

**Acceptance Criteria:**

**Given** an opportunity
**When** the team submits a Bid/No-Bid assessment
**Then** the approval workflow routes to the designated manager
**And** the decision is recorded with a rationale.

### Story 4.4: ROI & Preparation Time Tracking

As a Bid Manager,
I want to track time spent on proposals versus win rates,
So that I can calculate ROI.

**Acceptance Criteria:**

**Given** a submitted proposal
**When** the award decision is recorded
**Then** the system calculates the ROI based on tracked preparation time
**And** updates the team's performance metrics.

## Epic 5: EU Grant Specialization

Users can identify eligible EU grants, build required budgets, find consortium partners, and generate post-award reporting templates.

### Story 5.1: Grant Eligibility Matcher & Budget Builder

As a Grant Specialist,
I want to match with eligible EU grants and use an AI budget builder,
So that I ensure financial compliance.

**Acceptance Criteria:**

**Given** an EU grant opportunity
**When** I input project parameters
**Then** the system verifies eligibility
**And** generates a compliant budget template.

### Story 5.2: Consortium Finder & Logframe Generator

As a Grant Specialist,
I want to find partners and generate a logical framework,
So that I can build a strong consortium.

**Acceptance Criteria:**

**Given** a grant requiring partners
**When** I use the consortium finder
**Then** the system suggests matching companies
**And** helps generate the project logframe.

### Story 5.3: Post-Award Reporting Template Generator

As a Grant Manager,
I want to generate reporting templates after winning,
So that I fulfill post-award requirements easily.

**Acceptance Criteria:**

**Given** an awarded grant
**When** a reporting milestone approaches
**Then** the system generates the required template
**And** pre-fills it with project data.

## Epic 6: Compliance & Regulatory Frameworks

Users can ensure all proposals meet strict regulatory standards with configurable frameworks, AI auto-suggestions, and automated ESPD forms.

### Story 6.1: Admin-Configurable Compliance Frameworks

As a Platform Admin,
I want to configure compliance frameworks like ZOP and EU Directives,
So that the AI evaluates against the correct rules.

**Acceptance Criteria:**

**Given** a new regulatory framework
**When** I enter the framework rules
**Then** the system makes it available for compliance validation
**And** the AI incorporates it into its knowledge base.

### Story 6.2: AI Framework Auto-Suggestion

As a Technical Writer,
I want the system to auto-suggest the relevant framework for a tender,
So that I don't have to guess.

**Acceptance Criteria:**

**Given** an uploaded tender document
**When** the analysis is complete
**Then** the system suggests the applicable compliance framework
**And** applies its specific rules to the checklist.

### Story 6.3: ESPD Auto-Fill from Company Profile

As a Bid Manager,
I want the system to auto-fill the ESPD form using my company profile,
So that I save time on repetitive paperwork.

**Acceptance Criteria:**

**Given** a completed company profile
**When** I generate an ESPD form
**Then** the system auto-fills the standard fields
**And** highlights any missing data.

### Story 6.4: Immutable Audit Trails

As a Platform Admin,
I want all mutations to be logged immutably,
So that the system is fully auditable.

**Acceptance Criteria:**

**Given** any user modifies a proposal or setting
**When** the change is saved
**Then** a record is written to the audit_log schema
**And** it cannot be altered or deleted.

## Epic 7: Subscription & Billing Management

Users can subscribe to various tiers, manage billing, and trial the platform seamlessly.

### Story 7.1: Freemium Tier Enforcement

As a System,
I want to enforce access limits based on Free, Starter, Professional, and Enterprise tiers,
So that users only access what they pay for.

**Acceptance Criteria:**

**Given** a user is on the Starter tier
**When** they attempt to use a Professional feature
**Then** the system denies access
**And** prompts them to upgrade.

### Story 7.2: Stripe Integration & Usage Metering

As a Professional User,
I want to manage my subscription and see usage via Stripe,
So that my billing is transparent.

**Acceptance Criteria:**

**Given** I am a paying customer
**When** I access the billing portal
**Then** I am redirected to Stripe Customer Portal
**And** my feature usage is accurately metered.

### Story 7.3: 14-Day Free Trial & Per-Bid Add-ons

As a Free Explorer,
I want to start a 14-day trial or buy a one-off bid add-on,
So that I can test premium features.

**Acceptance Criteria:**

**Given** I want to upgrade temporarily
**When** I select the 14-day trial or per-bid purchase
**Then** the system grants temporary access
**And** revokes it when the period or limit expires.
ecklists mapping mandatory items
FR9: Epic 2 - Clause risk flagging for legal review
FR10: Epic 2 - Multi-language processing (BG, EN, DE, FR, RO)
FR11: Epic 3 - AI draft generator utilizing template libraries and institutional memory
FR12: Epic 3 - Scoring simulator to predict evaluator scores and suggest improvements
FR13: Epic 3 - Compliance validator for pre-submission checks
FR14: Epic 3 - Pricing assistant based on historical award data
FR15: Epic 3 - Collaborative rich-text editor (Tiptap) with version control and pessimistic locking
FR16: Epic 3 - Document export (PDF, DOCX) and reusable content blocks
FR17: Epic 4 - Entity-level RBAC (roles: bid manager, technical writer, etc.)
FR18: Epic 4 - Task orchestration with DAG dependencies
FR19: Epic 4 - Approval workflows and bid/no-bid decision scoring
FR20: Epic 4 - ROI and preparation time tracking
FR21: Epic 5 - Grant eligibility matcher and AI-assisted budget builder
FR22: Epic 5 - Consortium finder and logframe generator
FR23: Epic 5 - Reporting template generator for post-award requirements
FR24: Epic 6 - Admin-configurable compliance frameworks (ZOP, EU directives)
FR25: Epic 6 - AI framework auto-suggestion and regulation tracking
FR26: Epic 6 - ESPD auto-fill from company profiles
FR27: Epic 6 - Immutable audit trails for all mutations
FR28: Epic 7 - Freemium model with Free, Starter, Professional, and Enterprise tiers
FR29: Epic 7 - Stripe integration (Checkout, Customer Portal, Usage Metering, VAT)
FR30: Epic 7 - 14-day free trial and per-bid add-ons

## Epic List

{{epics_list}}

## Epic 1: Opportunity Discovery & Intelligence

Users can discover, filter, and track public procurement and grant opportunities across multiple sources with AI-assisted summaries and relevance matching.

### Story 1.1: Automated Multi-Source Crawling Pipeline

As a Platform Admin,
I want to configure automated crawlers for AOP, TED, and EU grant portals,
So that opportunities are continuously ingested into the system.

**Acceptance Criteria:**

**Given** the crawling pipeline is active
**When** a new tender is published on AOP
**Then** the system ingests the tender metadata
**And** saves it to the central database.

### Story 1.2: Searchable Opportunity Listings (Free Tier)

As a Free Explorer,
I want to search and filter opportunity listings by CPV, region, and deadline,
So that I can find relevant tenders.

**Acceptance Criteria:**

**Given** I am an authenticated Free Explorer
**When** I navigate to the opportunities page
**Then** I see a paginated list of tenders
**And** I can filter the results by specific criteria.

### Story 1.3: Tier-Based Visibility Enforcement

As a System,
I want to restrict full document access to paid tiers,
So that free users are incentivized to upgrade.

**Acceptance Criteria:**

**Given** a Free Explorer views a tender detail page
**When** they attempt to download the full documentation
**Then** the system blocks the download
**And** displays a prompt to upgrade to a paid tier.

### Story 1.4: AI Executive Summary Generation

As a Starter User,
I want to generate an AI summary of a tender,
So that I can quickly qualify the opportunity.

**Acceptance Criteria:**

**Given** a tender with full documentation exists
**When** I click "Generate Summary"
**Then** the system invokes the KraftData Agent
**And** streams the summary back to the UI in real-time.

### Story 1.5: Smart Relevance Matching & Alerts

As a Professional User,
I want to set up relevance profiles and email alerts,
So that I am notified of matching opportunities automatically.

**Acceptance Criteria:**

**Given** I have saved a relevance profile
**When** a new matching opportunity is ingested
**Then** the system flags it as a match
**And** sends me an email notification.

### Story 1.6: Pipeline Forecasting & Competitor Tracking

As a Professional User,
I want to track competitors and forecast my bid pipeline,
So that I can manage my resources effectively.

**Acceptance Criteria:**

**Given** I have added competitors to my profile
**When** I view the dashboard
**Then** I see a pipeline forecast visualization
**And** an analysis of competitor win rates on similar tenders.

## Epic 2: AI Document Analysis & Extraction

Users can upload complex tender documents and receive instant AI-extracted requirements, mandatory checklists, and risk flags across multiple languages.

### Story 2.1: AI Document Parsing & Extraction

As a Professional User,
I want the AI to parse tender documents across multiple languages,
So that deadlines and criteria are automatically extracted.

**Acceptance Criteria:**

**Given** a multi-language tender document is uploaded
**When** the document analysis is triggered
**Then** the system extracts key criteria and deadlines
**And** translates them to the user's preferred language.

### Story 2.2: Auto-Generated Requirement Checklists

As a Professional User,
I want the system to generate a compliance checklist from the tender,
So that I don't miss mandatory items.

**Acceptance Criteria:**

**Given** the AI document parser has run
**When** I view the tender requirements
**Then** I see a structured checklist of mandatory items
**And** I can mark them as complete.

### Story 2.3: Clause Risk Flagging

As a Bid Manager,
I want the AI to flag risky legal clauses,
So that I can review them before committing to a bid.

**Acceptance Criteria:**

**Given** the AI document parser has run
**When** I view the risk assessment section
**Then** high-risk clauses (e.g., unlimited liability) are highlighted
**And** the system provides a rationale for the risk flag.

## Epic 3: Proposal Generation & Optimization

Users can collaboratively draft, optimize, and export highly competitive proposals using AI assistance, scoring simulations, and rich-text editing.

### Story 3.1: Split-Pane Collaborative Tiptap Editor

As a Technical Writer,
I want to draft proposals in a split-pane editor with version control,
So that I can see requirements while writing.

**Acceptance Criteria:**

**Given** I am drafting a proposal
**When** I open the editor
**Then** I see the requirements on the side
**And** I can use slash commands in the Tiptap canvas.

### Story 3.2: AI Draft Generation

As a Technical Writer,
I want the AI to generate a first draft using template libraries,
So that I have a starting point for my proposal.

**Acceptance Criteria:**

**Given** a set of requirements and company profile
**When** I click "Generate Draft"
**Then** the AI creates a structured draft
**And** inserts it into the editor.

### Story 3.3: Pessimistic Locking & Optimistic Concurrency

As a Bid Manager,
I want to lock sections while editing,
So that I don't overwrite my colleague's work.

**Acceptance Criteria:**

**Given** another user is editing a section
**When** I attempt to edit the same section
**Then** the section is visually locked
**And** I cannot make changes until they unlock it.

### Story 3.4: Scoring Simulator & Pricing Assistant

As a Bid Manager,
I want to simulate my proposal score and get pricing suggestions,
So that I can optimize my bid.

**Acceptance Criteria:**

**Given** a completed draft
**When** I run the scoring simulator
**Then** the system predicts the evaluator score
**And** suggests pricing adjustments based on historical data.

### Story 3.5: Pre-submission Compliance Validator

As a Reviewer,
I want to run a compliance check against the draft,
So that I ensure all mandatory items are met.

**Acceptance Criteria:**

**Given** a finalized draft
**When** I run the compliance validator
**Then** the system generates a Compliance Metric Card
**And** highlights any missing mandatory items.

### Story 3.6: Document Export & Reusable Blocks

As a Bid Manager,
I want to export the proposal to PDF/DOCX and save reusable blocks,
So that I can submit the bid and reuse content.

**Acceptance Criteria:**

**Given** a validated proposal
**When** I click Export
**Then** the system generates a formatted PDF or DOCX
**And** I can save specific sections to my content library.

## Epic 4: Collaboration & Workflow Orchestration

Teams can collaborate efficiently with role-based access control, task dependencies, approval workflows, and time tracking.

### Story 4.1: Entity-Level RBAC & Team Management

As an Enterprise Admin,
I want to assign roles like bid manager and technical writer,
So that access is securely managed.

**Acceptance Criteria:**

**Given** a team of users
**When** I assign a user the Technical Writer role
**Then** they can edit proposals
**And** cannot submit final bids or view billing.

### Story 4.2: Task Orchestration & DAG Dependencies

As a Bid Manager,
I want to create tasks with dependencies,
So that the team knows what to work on and in what order.

**Acceptance Criteria:**

**Given** a complex bid
**When** I create tasks and set dependencies
**Then** users cannot start a blocked task
**And** they are notified when dependencies are resolved.

### Story 4.3: Approval Workflows & Bid/No-Bid Scoring

As a Reviewer,
I want to use approval workflows to formalize bid decisions,
So that we only pursue viable opportunities.

**Acceptance Criteria:**

**Given** an opportunity
**When** the team submits a Bid/No-Bid assessment
**Then** the approval workflow routes to the designated manager
**And** the decision is recorded with a rationale.

### Story 4.4: ROI & Preparation Time Tracking

As a Bid Manager,
I want to track time spent on proposals versus win rates,
So that I can calculate ROI.

**Acceptance Criteria:**

**Given** a submitted proposal
**When** the award decision is recorded
**Then** the system calculates the ROI based on tracked preparation time
**And** updates the team's performance metrics.

## Epic 5: EU Grant Specialization

Users can identify eligible EU grants, build required budgets, find consortium partners, and generate post-award reporting templates.

### Story 5.1: Grant Eligibility Matcher & Budget Builder

As a Grant Specialist,
I want to match with eligible EU grants and use an AI budget builder,
So that I ensure financial compliance.

**Acceptance Criteria:**

**Given** an EU grant opportunity
**When** I input project parameters
**Then** the system verifies eligibility
**And** generates a compliant budget template.

### Story 5.2: Consortium Finder & Logframe Generator

As a Grant Specialist,
I want to find partners and generate a logical framework,
So that I can build a strong consortium.

**Acceptance Criteria:**

**Given** a grant requiring partners
**When** I use the consortium finder
**Then** the system suggests matching companies
**And** helps generate the project logframe.

### Story 5.3: Post-Award Reporting Template Generator

As a Grant Manager,
I want to generate reporting templates after winning,
So that I fulfill post-award requirements easily.

**Acceptance Criteria:**

**Given** an awarded grant
**When** a reporting milestone approaches
**Then** the system generates the required template
**And** pre-fills it with project data.

## Epic 6: Compliance & Regulatory Frameworks

Users can ensure all proposals meet strict regulatory standards with configurable frameworks, AI auto-suggestions, and automated ESPD forms.

### Story 6.1: Admin-Configurable Compliance Frameworks

As a Platform Admin,
I want to configure compliance frameworks like ZOP and EU Directives,
So that the AI evaluates against the correct rules.

**Acceptance Criteria:**

**Given** a new regulatory framework
**When** I enter the framework rules
**Then** the system makes it available for compliance validation
**And** the AI incorporates it into its knowledge base.

### Story 6.2: AI Framework Auto-Suggestion

As a Technical Writer,
I want the system to auto-suggest the relevant framework for a tender,
So that I don't have to guess.

**Acceptance Criteria:**

**Given** an uploaded tender document
**When** the analysis is complete
**Then** the system suggests the applicable compliance framework
**And** applies its specific rules to the checklist.

### Story 6.3: ESPD Auto-Fill from Company Profile

As a Bid Manager,
I want the system to auto-fill the ESPD form using my company profile,
So that I save time on repetitive paperwork.

**Acceptance Criteria:**

**Given** a completed company profile
**When** I generate an ESPD form
**Then** the system auto-fills the standard fields
**And** highlights any missing data.

### Story 6.4: Immutable Audit Trails

As a Platform Admin,
I want all mutations to be logged immutably,
So that the system is fully auditable.

**Acceptance Criteria:**

**Given** any user modifies a proposal or setting
**When** the change is saved
**Then** a record is written to the audit_log schema
**And** it cannot be altered or deleted.

## Epic 7: Subscription & Billing Management

Users can subscribe to various tiers, manage billing, and trial the platform seamlessly.

### Story 7.1: Freemium Tier Enforcement

As a System,
I want to enforce access limits based on Free, Starter, Professional, and Enterprise tiers,
So that users only access what they pay for.

**Acceptance Criteria:**

**Given** a user is on the Starter tier
**When** they attempt to use a Professional feature
**Then** the system denies access
**And** prompts them to upgrade.

### Story 7.2: Stripe Integration & Usage Metering

As a Professional User,
I want to manage my subscription and see usage via Stripe,
So that my billing is transparent.

**Acceptance Criteria:**

**Given** I am a paying customer
**When** I access the billing portal
**Then** I am redirected to Stripe Customer Portal
**And** my feature usage is accurately metered.

### Story 7.3: 14-Day Free Trial & Per-Bid Add-ons

As a Free Explorer,
I want to start a 14-day trial or buy a one-off bid add-on,
So that I can test premium features.

**Acceptance Criteria:**

**Given** I want to upgrade temporarily
**When** I select the 14-day trial or per-bid purchase
**Then** the system grants temporary access
**And** revokes it when the period or limit expires.


---

## Epics 14–21 — Amendment 2026-04-25 ("Tenders + Grants for EU Consulting Firms")

The following epics were injected via the 2026-04-25 PRD amendment driven by market-research findings (Altura €8M Series A in June 2025, ISO 27001 already shipped, closing the EU dual-segment whitespace within 12–18 months). Detailed sub-stories live in the per-epic files referenced in `epicFiles` frontmatter; rationale, locked decisions, and architectural verdicts live in the four amendment-source documents in `amendmentSource` frontmatter.

| Epic | Title | Sprint | Points | Dependencies | Milestone |
|---|---|---|---|---|---|
| E14 | Multi-Client Workspace Data Model & RBAC | 14–16 | 34 | E01, E02, E06 | Consulting-firm ICP foundation |
| E15 | Per-Bid SKU + Pro+ Tier Launch | 16–17 | 21 | E08, E14 | Consulting-firm monetisation |
| E16 | Slack & Teams Notifications | 17–18 | 8 | E14, E15 | Pro-tier integration foothold |
| E17 | CRM Integrations (HubSpot → Pipedrive → Salesforce) | 18–22 | 55 | E14, E15, E16 | Pro+ tier deal-blocker resolution |
| E18 | Trust Center & Compliance Posture | 14–15 | 13 | E03, E07, E09 | M2 hard deadline |
| E19 | Outcome Telemetry & Renewal Proof Brief | 17–18 | 21 | E12, E14, E07 | Renewal engine |
| E20 | NPS, Reviews & Onboarding (residual) | 18 | 5 | E14, E19 | Review-pipeline + retention |
| E21 | Platform Reliability for 99.9% SLA | 14–17 (parallel) | 34 | Epic 13 carry-forwards | 99.9% SLA externally publishable |

**Plus parallel programme (NOT decomposed as stories — sprint-spanning workstream):** ISO 27001 audit-prep M2–M12, €50K budget, ~8–10 sprint-weeks distributed engineering effort across the calendar.

### Locked Decisions (for downstream skills)

| # | Decision | Locked value |
|---|---|---|
| 1 | Existing paying customers from v1.0 | None — atomic migration in S14.00, no transition shim |
| 2 | Pro+ tier price | €199/user/month |
| 3 | Per-bid SKU pricing tiers | €99 standard / €199 with grant module / €299 full Enterprise feature stack |
| 4 | Trust Center launch deadline | Month 2 |
| 5 | ISO 27001 audit budget | €50K planning assumption |
| 6 | CRM provider order | HubSpot → Pipedrive → Salesforce |

### Dependency Map (Per Architecture Evaluation §7)

```
M0─────M1─────M2─────M3─────M4─────M5─────M6─────M7─────────M12
E14    ████████████░                                          (foundational)
E18    ██████░       (M2 deadline)
E15           █████░
E16                  ███░
E19                  █████░
E17                       ████████████░
E20                            ██░
E21    ████████████░  (parallel platform; gates 99.9% SLA)
ISO       ████████████████████████████████████████░
```

### Carry-Forward Re-Homing

- Epic 13's **k6 baseline** (deferred 6+ epics) → re-homed as **PE.01** in Epic 21 (CRITICAL PATH; gates 99.9% SLA publication)
- Epic 13's **Prometheus /metrics bootstrap** → absorbed into **PE.05** (SLO dashboards) in Epic 21
- Epic 13's **Dependabot config** → remains in Epic 13's set; not absorbed into v4 epics

### Next Skill in Chain

`bmad-sprint-planning` to regenerate `sprint-status.yaml` from the updated `epics.md`. Then story-by-story execution via `bmad-create-story` and `bmad-dev-story`. ISO 27001 audit-prep programme runs as a parallel workstream — not story-decomposed.
