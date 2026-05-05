---
stepsCompleted: ["step-01-validate-prerequisites", "step-02-design-epics", "step-03-create-stories", "step-04-final-validation"]
inputDocuments: ["eusolicit-docs/planning-artifacts/PRD.md", "eusolicit-docs/planning-artifacts/architecture.md", "eusolicit-docs/planning-artifacts/ux-spec.md"]
---

# eusolicit - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for eusolicit, decomposing the requirements from the PRD, UX Design if it exists, and Architecture requirements into implementable stories.

## Requirements Inventory

### Functional Requirements

FR-1: A new user can register for an account using an email and password.
FR-2: A new user can register for an account using their Google identity.
FR-3: A registered user can log in and out of the system.
FR-4: A user must verify their email address before accessing core features.
FR-5: An admin user can create and manage their company profile.
FR-6: An admin user can invite other users to their company workspace.
FR-7: An admin user can assign and modify roles for users within their workspace (Admin, Bid Manager, Contributor, Reviewer, Read-Only).
FR-8: An enterprise admin can manage multiple, isolated client workspaces under a single parent company account.
FR-9: A user can belong to multiple company workspaces and switch between them.
FR-10: A user can subscribe to a paid tier (Starter, Professional, Enterprise) using a credit card via Stripe.
FR-11: The system automatically gates features based on the user's active subscription tier and usage limits.
FR-12: A new user is automatically enrolled in a 14-day free trial of the Professional tier.
FR-13: A user can manage their subscription (upgrade, downgrade, cancel) through a self-service customer portal (Stripe).
FR-14: The system can process one-time add-on purchases for specific premium features.
FR-15: The system can automatically ingest new procurement opportunities from configured public sources (AOP, TED).
FR-16: A user can search for opportunities using full-text search and faceted filters (e.g., CPV code, region, budget).
FR-17: The system can calculate and display an AI-generated relevance score for each opportunity based on a company's profile.
FR-18: A user can view a list of opportunities, sorted by relevance or other criteria (e.g., deadline).
FR-19: A user can view the detailed information for a single opportunity.
FR-20: A user can save or "star" opportunities for future reference.
FR-21: The system can generate a one-page executive summary for a given tender document.
FR-22: The system can extract a structured list of mandatory requirements from a tender document to form a compliance checklist.
FR-23: The system can identify and flag potentially high-risk clauses within a tender document.
FR-24: A user can upload tender documents (PDF, DOCX) for the system to analyze.
FR-25: The system can simulate a potential score for a proposal based on the tender's evaluation criteria.
FR-26: A user can create a new proposal associated with a specific opportunity.
FR-27: The system can generate an AI-assisted first draft of a proposal.
FR-28: A user can edit proposal content in a rich text editor.
FR-29: The system can maintain a version history of proposal drafts.
FR-30: A user can restore a previous version of a proposal.
FR-31: A user can lock a specific section of a proposal to prevent concurrent edits. (Post-MVP)
FR-32: Users can add and resolve comments on proposal sections. (Post-MVP)
FR-33: A user can export a final proposal to PDF and DOCX formats.
FR-34: The system can validate a proposal against its auto-generated compliance checklist.
FR-35: A platform admin can create and manage regulatory compliance frameworks (e.g., ZOP).
FR-36: The system can generate a pre-filled European Single Procurement Document (ESPD) from a company's profile.
FR-37: A user can create and manage tasks associated with a proposal. (Post-MVP)
FR-38: A user can define dependencies between tasks. (Post-MVP)
FR-39: The system can enforce a multi-stage approval workflow for a proposal before submission. (Post-MVP)
FR-40: A user can configure and receive email digests for new, relevant opportunities.
FR-41: The system can sync opportunity deadlines with a user's external calendar (Google/Outlook). (Post-MVP)
FR-42: A platform admin can manage tenants, subscriptions, and system-wide settings through a secure admin portal.
FR-43: The system maintains an immutable audit trail of all significant user and system actions.
FR-44: An enterprise user can access platform data via a secure REST API.

### NonFunctional Requirements

NFR-1 (API Response Time): All user-facing API endpoints must have a 95th percentile (p95) response time of less than 200ms.
NFR-2 (AI Generation): The Time to First Byte (TTFB) for streaming AI-generated content must be less than 500ms.
NFR-3 (Page Load): Core application pages must achieve a Lighthouse Performance score of 90 or higher.
NFR-4 (Concurrency): The system must support 100 concurrent active users during the MVP phase without performance degradation.
NFR-5 (Authentication): User authentication must be handled via JWTs with short-lived access tokens and securely stored refresh tokens.
NFR-6 (Encryption): All data must be encrypted in transit using TLS 1.3+ and at rest using AES-256.
NFR-7 (Data Isolation): Strict data isolation between tenants. No data access should be possible across workspace boundaries.
NFR-8 (Admin Security): Access to the admin portal must be restricted by IP allow-listing and require multi-factor authentication.
NFR-9 (Dependencies): The system must not have any known critical or high-severity vulnerabilities.
NFR-10 (Vertical Scalability): The system's services must be designed to scale vertically.
NFR-11 (Horizontal Scalability): The application services and AI gateways must be stateless.
NFR-12 (Database Scalability): The database architecture must support read replicas.
NFR-13 (Growth Target): Scale to support 10,000 active companies and 1,000,000 total opportunities.
NFR-14 (Uptime): The platform must achieve >= 99.5% uptime.
NFR-15 (Data Integrity): Zero data loss for user-generated content.
NFR-16 (Idempotency): Critical financial operations must be idempotent.
NFR-17 (Disaster Recovery): Service restoration within 4 hours in case of a full regional outage.
NFR-18 (WCAG Compliance): WCAG 2.1 Level AA conformance.
NFR-19 (Keyboard Navigation): Fully navigable and operable using only a keyboard.
NFR-20 (Screen Reader Support): Compatible with modern screen readers with appropriate ARIA attributes.
NFR-21 (Code Quality): Pass ruff/mypy checks for backend and eslint/tsc for frontend.
NFR-22 (Test Coverage): Minimum 80% unit test coverage and 90% E2E test coverage.
NFR-23 (Logging & Monitoring): Structured logs and centralized monitoring (Prometheus/Grafana).

### Additional Requirements

- Strict Data Isolation: Schema-per-service logical isolation in a single PostgreSQL instance. No cross-schema FKs.
- Dual-layer Authentication: Next.js middleware and Backend RS256 JWT with RBAC matrix.
- Circuit Breaker: Two-layer outbound resilience `circuit_breaker(retry(http_factory))`.
- AI Streaming constraints: SSE with TTFB < 500ms, quota check before StreamingResponse.
- Materialized Views: Used for Outcome Telemetry, refresh concurrently.
- Test Architecture: testcontainers with Postgres and Redis mandatory for integration testing.

### UX Design Requirements

UX-DR1: Multi-proposal pipeline, parallel collaboration, deadline-first surfacing.
UX-DR2: Split-pane editor combining Tiptap rich-text editor and contextual inspector panels (Requirements, Compliance, Score).
UX-DR3: Top bar with global command menu Cmd+K for scoped navigation and actions.
UX-DR4: Real-time AI operation state machine (idle, requesting, streaming, complete, error, canceled).
UX-DR5: Explicit per-section locking for proposal drafting.
UX-DR6: One-page AI executive summary generated within 90 seconds.

### FR Coverage Map

{{requirements_coverage_map}}

## Epic List

### Epic 1: User Onboarding and Identity
Goal: Users can securely register, log in, and manage their individual and company profiles.
**FRs covered:** FR-1, FR-2, FR-3, FR-4, FR-5

### Epic 2: Subscription & Billing Management
Goal: Users can subscribe to paid tiers, experience free trials, and manage billing.
**FRs covered:** FR-10, FR-11, FR-12, FR-13, FR-14

### Epic 3: Tender Discovery and Data Ingestion
Goal: Users can find relevant public procurement opportunities via automated ingestion and powerful search.
**FRs covered:** FR-15, FR-16, FR-17, FR-18, FR-19, FR-20

### Epic 4: AI Tender Analysis
Goal: Users can upload documents and instantly receive AI-generated summaries, checklists, and risk flags.
**FRs covered:** FR-21, FR-22, FR-23, FR-24, FR-25

### Epic 5: AI-Assisted Proposal Drafting
Goal: Users can generate AI drafts, edit them in a rich-text environment, and export finished proposals.
**FRs covered:** FR-26, FR-27, FR-28, FR-29, FR-30, FR-33

### Epic 6: Automated Compliance & Validation
Goal: Users can validate their proposals against auto-generated checklists and regulatory frameworks.
**FRs covered:** FR-34, FR-35, FR-36

### Epic 7: Team Collaboration and Workflow
Goal: Teams can collaborate on proposals with roles, section locking, comments, and approvals.
**FRs covered:** FR-6, FR-7, FR-31, FR-32, FR-37, FR-38, FR-39

### Epic 8: Enterprise Admin and Integrations
Goal: Enterprise admins can manage sub-tenants, access the API, and monitor the audit trail.
**FRs covered:** FR-8, FR-9, FR-42, FR-43, FR-44

### Epic 9: Notifications and Scheduling
Goal: Users receive timely alerts and calendar syncs for deadlines and opportunity updates.
**FRs covered:** FR-40, FR-41

d on a company's profile.
FR-18: Epic 3 - A user can view a list of opportunities, sorted by relevance or other criteria (e.g., deadline).
FR-19: Epic 3 - A user can view the detailed information for a single opportunity.
FR-20: Epic 3 - A user can save or "star" opportunities for future reference.
FR-21: Epic 4 - The system can generate a one-page executive summary for a given tender document.
FR-22: Epic 4 - The system can extract a structured list of mandatory requirements from a tender document to form a compliance checklist.
FR-23: Epic 4 - The system can identify and flag potentially high-risk clauses within a tender document.
FR-24: Epic 4 - A user can upload tender documents (PDF, DOCX) for the system to analyze.
FR-25: Epic 4 - The system can simulate a potential score for a proposal based on the tender's evaluation criteria.
FR-26: Epic 5 - A user can create a new proposal associated with a specific opportunity.
FR-27: Epic 5 - The system can generate an AI-assisted first draft of a proposal.
FR-28: Epic 5 - A user can edit proposal content in a rich text editor.
FR-29: Epic 5 - The system can maintain a version history of proposal drafts.
FR-30: Epic 5 - A user can restore a previous version of a proposal.
FR-31: Epic 7 - A user can lock a specific section of a proposal to prevent concurrent edits. (Post-MVP)
FR-32: Epic 7 - Users can add and resolve comments on proposal sections. (Post-MVP)
FR-33: Epic 5 - A user can export a final proposal to PDF and DOCX formats.
FR-34: Epic 6 - The system can validate a proposal against its auto-generated compliance checklist.
FR-35: Epic 6 - A platform admin can create and manage regulatory compliance frameworks (e.g., ZOP).
FR-36: Epic 6 - The system can generate a pre-filled European Single Procurement Document (ESPD) from a company's profile.
FR-37: Epic 7 - A user can create and manage tasks associated with a proposal. (Post-MVP)
FR-38: Epic 7 - A user can define dependencies between tasks. (Post-MVP)
FR-39: Epic 7 - The system can enforce a multi-stage approval workflow for a proposal before submission. (Post-MVP)
FR-40: Epic 9 - A user can configure and receive email digests for new, relevant opportunities.
FR-41: Epic 9 - The system can sync opportunity deadlines with a user's external calendar (Google/Outlook). (Post-MVP)
FR-42: Epic 8 - A platform admin can manage tenants, subscriptions, and system-wide settings through a secure admin portal.
FR-43: Epic 8 - The system maintains an immutable audit trail of all significant user and system actions.
FR-44: Epic 8 - An enterprise user can access platform data via a secure REST API.

## Epic List

{{epics_list}}


## Epic 1: User Onboarding and Identity
Goal: Users can securely register, log in, and manage their individual and company profiles.

### Story 1.1: User Registration with Email and Password
As a new user,
I want to register using my email and password,
So that I can create an account on the platform.

**Acceptance Criteria:**
**Given** I am on the registration page
**When** I enter a valid email, a secure password, and submit the form
**Then** an account is created
**And** a verification email is sent to me.

### Story 1.2: User Registration with Google OAuth
As a new user,
I want to register using my Google account,
So that I can quickly sign up without a new password.

**Acceptance Criteria:**
**Given** I am on the registration page
**When** I choose to sign up with Google and authenticate
**Then** an account is created linked to my Google identity
**And** I am logged into the platform.

### Story 1.3: User Login and Logout
As a registered user,
I want to log in and out of my account securely,
So that my session and data remain protected.

**Acceptance Criteria:**
**Given** I have a registered account
**When** I enter my credentials on the login page
**Then** I am authenticated via RS256 JWT
**And** I am redirected to the dashboard.
**And** clicking logout terminates my session securely.

### Story 1.4: Email Verification
As a newly registered user,
I want to verify my email address,
So that I can access the core features of the platform.

**Acceptance Criteria:**
**Given** I have received a verification email
**When** I click the verification link
**Then** my email is marked as verified in the system
**And** I gain access to the core platform features.

### Story 1.5: Company Profile Management
As an admin user,
I want to create and update my company profile,
So that the platform can use my company's capabilities and sectors for matching and ESPD.

**Acceptance Criteria:**
**Given** I am an authenticated admin user
**When** I navigate to the company profile settings
**Then** I can input and save my company details, CPV codes, and key personnel
**And** the profile completeness meter updates accordingly.

## Epic 2: Subscription & Billing Management
Goal: Users can subscribe to paid tiers, experience free trials, and manage billing.

### Story 2.1: Free Trial Auto-Enrollment
As a new user,
I want to be automatically enrolled in a 14-day free trial of the Professional tier,
So that I can evaluate the platform's advanced features risk-free.

**Acceptance Criteria:**
**Given** I have just registered and verified my email
**When** I log in for the first time
**Then** I am placed on the Professional tier trial for 14 days
**And** a trial countdown is visible in the top bar.

### Story 2.2: Stripe Subscription Upgrade
As a user,
I want to upgrade my tier using my credit card via Stripe,
So that I can continue using premium features post-trial.

**Acceptance Criteria:**
**Given** I am on the billing page
**When** I select a paid tier and complete checkout via Stripe
**Then** my account is upgraded to the selected tier
**And** the webhook updates my subscription status idempotently.

### Story 2.3: Feature Gating by Tier
As a platform,
I want to automatically restrict features based on the user's active tier,
So that users are incentivized to upgrade and paywall rules are enforced.

**Acceptance Criteria:**
**Given** I am a user on a specific tier
**When** I attempt to access a feature
**Then** the `TierGate` dependency checks my tier
**And** restricts or grants access accordingly with a paywall UI if restricted.

### Story 2.4: Subscription Self-Service Management
As a paying user,
I want to manage my subscription through a self-service portal,
So that I can easily upgrade, downgrade, or cancel without contacting support.

**Acceptance Criteria:**
**Given** I have an active subscription
**When** I click to manage my billing
**Then** I am redirected to the Stripe customer portal
**And** any changes made are synced back via webhooks.

### Story 2.5: One-Time Add-On Purchases
As a user,
I want to purchase specific premium features on a per-bid basis,
So that I can access advanced capabilities for critical bids without a full subscription.

**Acceptance Criteria:**
**Given** I am viewing an opportunity
**When** I elect to purchase a per-bid add-on
**Then** I can pay via Stripe checkout
**And** the specific feature is unlocked for that opportunity only.

## Epic 3: Tender Discovery and Data Ingestion
Goal: Users can find relevant public procurement opportunities via automated ingestion and powerful search.

### Story 3.1: Automated Ingestion from AOP and TED
As the platform data pipeline,
I want to automatically ingest new opportunities from AOP and TED,
So that the database contains the latest public procurement records.

**Acceptance Criteria:**
**Given** the Celery beat schedule is active
**When** the crawler job runs
**Then** it fetches new records from the sources
**And** saves them into the `pipeline.opportunities` table without duplication.

### Story 3.2: Full-Text Search and Filtering
As a user,
I want to search and filter opportunities by keywords, CPV, country, region, and budget,
So that I can narrow down the list to relevant tenders.

**Acceptance Criteria:**
**Given** I am on the opportunities list
**When** I apply filters or enter search terms
**Then** the list updates instantly to show matching results
**And** the URL updates to reflect the filter state.

### Story 3.3: AI-Generated Relevance Scoring
As a user,
I want to see an AI-generated relevance score for each opportunity,
So that I can quickly gauge if a tender is a good fit for my company.

**Acceptance Criteria:**
**Given** my company profile is populated
**When** new opportunities are ingested
**Then** the AI calculates a relevance score (0-100)
**And** this score is displayed alongside the opportunity.

### Story 3.4: Opportunity Detailed View
As a user,
I want to view the full details of a single opportunity,
So that I can review its deadlines, budget, and source documents.

**Acceptance Criteria:**
**Given** I click on an opportunity in the list
**When** the detail page loads
**Then** I see the comprehensive data and tabs for overview, analysis, and documents.

### Story 3.5: Saving Opportunities
As a user,
I want to save or star opportunities,
So that I can easily track and return to them later.

**Acceptance Criteria:**
**Given** I am viewing an opportunity
**When** I click the star icon
**Then** the opportunity is added to my saved list
**And** I can filter my view to see only saved opportunities.

## Epic 4: AI Tender Analysis
Goal: Users can upload documents and instantly receive AI-generated summaries, checklists, and risk flags.

### Story 4.1: Tender Document Upload
As a user,
I want to upload tender documents (PDF, DOCX) to the platform,
So that they can be analyzed by the AI agents.

**Acceptance Criteria:**
**Given** I am on the opportunity detail page
**When** I upload a document
**Then** it is scanned by ClamAV and stored securely
**And** becomes available for AI processing.

### Story 4.2: AI Executive Summary Generation
As a user,
I want the AI to generate a one-page executive summary of the tender document,
So that I can grasp the core scope without reading the entire file.

**Acceptance Criteria:**
**Given** a tender document is uploaded
**When** I request a summary
**Then** the AI Gateway streams the summary response (TTFB < 500ms)
**And** the text appears progressively in the UI.

### Story 4.3: Requirements Extraction and Checklist
As a user,
I want the AI to extract mandatory requirements into a checklist,
So that I know exactly what documents and conditions must be met.

**Acceptance Criteria:**
**Given** a tender document is uploaded
**When** the extraction agent runs
**Then** a structured list of requirements is generated
**And** each item links back to the source clause in the document.

### Story 4.4: High-Risk Clause Flagging
As a user,
I want the AI to identify and flag high-risk clauses in the tender,
So that I can mitigate potential legal or delivery risks.

**Acceptance Criteria:**
**Given** a tender document is uploaded
**When** the risk agent analyzes it
**Then** potentially risky clauses are highlighted with severity levels
**And** displayed in a dedicated risks tab.

### Story 4.5: Proposal Score Simulation
As a user,
I want the system to simulate an evaluation score based on the tender criteria,
So that I can estimate my chances of winning.

**Acceptance Criteria:**
**Given** an opportunity with evaluation criteria
**When** I run the score simulator
**Then** it outputs an estimated score (0-100)
**And** provides specific suggestions to improve the bid.

## Epic 5: AI-Assisted Proposal Drafting
Goal: Users can generate AI drafts, edit them in a rich-text environment, and export finished proposals.

### Story 5.1: Create New Proposal Workspace
As a user,
I want to create a new proposal linked to a specific opportunity,
So that I can begin drafting my response.

**Acceptance Criteria:**
**Given** I am on an opportunity page
**When** I click "Create Proposal"
**Then** a new proposal entity is created
**And** I am taken to the split-pane proposal editor.

### Story 5.2: AI First Draft Generation
As a user,
I want the AI to generate a first draft of the proposal,
So that I don't have to start from a blank page.

**Acceptance Criteria:**
**Given** I am in an empty proposal editor
**When** I click "Generate Draft"
**Then** the AI streams a structured draft based on my company profile and the tender requirements
**And** I can accept, reject, or regenerate the content.

### Story 5.3: Rich Text Editing
As a user,
I want to edit the proposal content in a rich text editor,
So that I can refine the formatting and text of my bid.

**Acceptance Criteria:**
**Given** I am viewing the proposal draft
**When** I type and format text in the Tiptap editor
**Then** the content updates immediately
**And** autosaves periodically.

### Story 5.4: Version History and Restoration
As a user,
I want to view the version history of my proposal and restore previous versions,
So that I can recover from mistakes or unwanted edits.

**Acceptance Criteria:**
**Given** a proposal with multiple saved versions
**When** I open the version history panel
**Then** I can see past snapshots
**And** clicking restore reverts the active document to that state.

### Story 5.5: Document Export
As a user,
I want to export the final proposal to PDF or DOCX,
So that I can submit it to the contracting authority.

**Acceptance Criteria:**
**Given** I have a completed proposal
**When** I select "Export" and choose a format
**Then** a background task generates the file
**And** prompts me to download it securely.

## Epic 6: Automated Compliance & Validation
Goal: Users can validate their proposals against auto-generated checklists and regulatory frameworks.

### Story 6.1: Proposal Compliance Validation
As a user,
I want to validate my proposal draft against the extracted requirements checklist,
So that I can ensure no mandatory elements are missing.

**Acceptance Criteria:**
**Given** I am editing a proposal
**When** I run the compliance check
**Then** the AI compares the draft against the checklist
**And** surfaces any gaps or pass/fail statuses inline.

### Story 6.2: Regulatory Framework Management
As an admin operator,
I want to create and manage regulatory compliance frameworks (like ZOP rules),
So that the compliance engine has the latest legal constraints.

**Acceptance Criteria:**
**Given** I am in the admin portal
**When** I navigate to Compliance Frameworks
**Then** I can add, edit, or version compliance rules with effective dates
**And** these rules are immediately used by the compliance checking agent.

### Story 6.3: ESPD Generation
As a user,
I want to automatically generate a pre-filled European Single Procurement Document (ESPD),
So that I don't have to manually enter my company details into complex forms.

**Acceptance Criteria:**
**Given** my company profile is complete
**When** I request an ESPD for a specific opportunity
**Then** the system maps my profile data to the ESPD XML format
**And** provides it for download as XML and PDF.

## Epic 7: Team Collaboration and Workflow
Goal: Teams can collaborate on proposals with roles, section locking, comments, and approvals.

### Story 7.1: Workspace Member Invitations
As an admin user,
I want to invite other users to my workspace,
So that my team can collaborate on bids together.

**Acceptance Criteria:**
**Given** I am in the workspace settings
**When** I enter emails and select roles to invite
**Then** the users receive invitation emails
**And** upon acceptance, they join the workspace with the specified permissions.

### Story 7.2: Role-Based Access Control (RBAC) Matrix
As an admin user,
I want to assign granular roles (Admin, Bid Manager, Contributor, Reviewer) to my team members,
So that I can control who can view, edit, or approve proposals.

**Acceptance Criteria:**
**Given** I am managing workspace members
**When** I change a user's role
**Then** their permissions update instantly across the platform
**And** unauthorized actions are blocked with a 403 or 404 response.

### Story 7.3: Proposal Section Locking
As a contributor,
I want to lock a specific section of a proposal while I am editing it,
So that I prevent concurrent overwrite conflicts with my teammates.

**Acceptance Criteria:**
**Given** a multi-user proposal
**When** I start editing a section
**Then** the section is locked for other users
**And** they see a read-only view with my name indicating the lock owner.

### Story 7.4: Inline Comments and Mentions
As a team member,
I want to add comments and @mention colleagues on specific parts of the proposal,
So that we can discuss and resolve feedback directly in context.

**Acceptance Criteria:**
**Given** I am reviewing a proposal
**When** I highlight text and add a comment with an @mention
**Then** the comment is anchored to the text
**And** the mentioned user receives a notification.

### Story 7.5: Task Management and Dependencies
As a Bid Manager,
I want to create tasks, assign them, and define dependencies,
So that I can track the progress of the entire proposal effort.

**Acceptance Criteria:**
**Given** I am viewing a proposal's workflow tab
**When** I create a task and set a dependency
**Then** it appears on the team's task list
**And** dependent tasks unlock only when predecessors are complete.

### Story 7.6: Multi-Stage Approval Workflow
As a Bid Manager,
I want to enforce an approval workflow before a proposal can be marked as submitted,
So that quality is guaranteed by designated reviewers.

**Acceptance Criteria:**
**Given** a completed proposal draft
**When** I click "Send for Approval"
**Then** the designated Reviewer is notified
**And** they can "Approve" or "Request Changes" via a single click.

## Epic 8: Enterprise Admin and Integrations
Goal: Enterprise admins can manage sub-tenants, access the API, and monitor the audit trail.

### Story 8.1: Sub-Tenant Workspace Management
As an enterprise admin,
I want to create and manage multiple isolated client workspaces under my parent account,
So that I can keep different clients' data strictly segregated.

**Acceptance Criteria:**
**Given** I have an Enterprise subscription
**When** I create a new workspace
**Then** a new logically isolated tenant is provisioned
**And** I can switch contexts using the workspace switcher in the UI.

### Story 8.2: Immutable Audit Trail
As a platform,
I want to maintain an immutable audit log of all significant user and system actions,
So that enterprise customers can satisfy compliance and forensic requirements.

**Acceptance Criteria:**
**Given** a mutation event occurs
**When** the action completes
**Then** an append-only record is written to `shared.audit_log` securely via a background task.

### Story 8.3: Secure REST API Access
As an enterprise developer,
I want to access platform data via a secure REST API,
So that I can integrate EU Solicit with our internal systems (e.g., CRM, BI).

**Acceptance Criteria:**
**Given** I have generated an API key in the enterprise settings
**When** I make a valid request to an Enterprise API endpoint
**Then** I receive the correct JSON data adhering to strict rate limits
**And** the API documentation is available via an OpenAPI explorer.

### Story 8.4: Platform Admin Portal
As an internal EU Solicit operator,
I want to use a dedicated admin portal to manage tenants and monitor system health,
So that I can resolve operational issues securely without touching the public app.

**Acceptance Criteria:**
**Given** I am on the internal VPN with MFA
**When** I log into the admin portal
**Then** I can view crawler statuses, manage subscriptions, and view aggregated metrics.

## Epic 9: Notifications and Scheduling
Goal: Users receive timely alerts and calendar syncs for deadlines and opportunity updates.

### Story 9.1: Email Digests for Opportunities
As a user,
I want to configure and receive email digests of newly matched opportunities,
So that I don't miss relevant tenders even when I'm not logged in.

**Acceptance Criteria:**
**Given** I have opted into daily or weekly digests
**When** the scheduled time arrives
**Then** the notification service compiles matching opportunities
**And** sends a well-formatted email to my registered address.

### Story 9.2: External Calendar Sync
As a Pro+ user,
I want to sync opportunity deadlines and proposal tasks to my Google/Outlook calendar,
So that my schedule is integrated with my daily workflow tool.

**Acceptance Criteria:**
**Given** I have connected my external calendar in settings
**When** an opportunity is saved or a task is assigned to me with a deadline
**Then** an event is automatically created in my Google/Outlook calendar
**And** updates if the deadline changes in EU Solicit.

