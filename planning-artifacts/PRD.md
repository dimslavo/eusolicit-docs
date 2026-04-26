---
stepsCompleted: ["step-01-init", "step-12-complete", "amendment-2026-04-25"]
inputDocuments: ["project-context.md", "config.yaml", "EU_Solicit_Requirements_Brief_v5.md", "research/market-eu-govwin-tender-intelligence-research-2026-04-25.md", "prd-amendment-2026-04-25.md"]
workflowType: 'prd'
---

# Product Requirements Document - EU Solicit

**Author:** Deb (with Mary, Business Analyst, for the 2026-04-25 amendment)
**Date:** 2026-04-25
**Version:** 1.1 (incorporates PRD Amendment 2026-04-25 — "Tenders + Grants for EU Consulting Firms")

> **Amendment notice:** This version (v1.1) folds in the PRD-amendment brief dated 2026-04-25 (`prd-amendment-2026-04-25.md`), driven by the market-research study `research/market-eu-govwin-tender-intelligence-research-2026-04-25.md`. The amendment repositions the platform to a dual-segment (tenders + grants), consulting-firm-first product, and adds six PRD-level requirement changes. Architectural evaluation by Winston is pending; epic decomposition will follow via `bmad-create-epics-and-stories`.

## 1. Executive Summary & Vision

**EU Solicit** is the only EU-native SaaS platform that helps **consulting firms** win **both public tenders and EU grants** — combining multilingual opportunity intelligence (AOP, TED, EU Funding & Tenders Portal), AI-driven proposal generation orchestrated by KraftData (Sirma AI) agents, and EU-grade compliance posture (GDPR + ISO 27001 + EU data residency) into one workspace per client engagement.

The platform's strategic differentiator is the **dual-segment cross-sell**: where Loopio/Responsive/AutogenAI specialise in tender-side proposal generation and Welcome Europe/PNO specialise in grant-side advisory, EU Solicit unifies both for the consulting-firm ICP that serves the same client across both pursuits. By abstracting the friction of monitoring 2,000+ EU procurement portals and manually drafting compliant tender and grant documents, EU Solicit enables consulting firms to scale pursuit volume without linear headcount growth.

## 2. Problem Statement & Value Proposition

**Problem:**
Consulting firms in the EU lose ~€11–15K per bid-manager FTE per year on information work alone (19% of working time on portal scanning across 2,000+ EU procurement sources). EU single-bidder rates have nearly doubled in a decade (23.5% → 41.8%), confirming opportunity-discovery — not competitor analysis — is the sharpest pain. Horizon Europe applications consume hundreds of hours per proposal at a 12% success rate with a €7,500 median consultancy fee. General-purpose LLMs cannot replace this workflow because they lack structured procurement data (CPV codes, ESPD schemas), EU programme-specific compliance frameworks, and persistent institutional memory per consulting-client engagement.

**Value Proposition:**
EU Solicit replaces the manual portal-scanning grind with multilingual AI-driven discovery + qualification, replaces ad-hoc proposal drafting with agentic generation backed by client-specific institutional memory, and replaces generic compliance checking with admin-configurable regulatory frameworks per opportunity (Bulgarian ZOP, EU Directives 2014/24/EU and 2014/25/EU, Horizon Europe rules). For consulting firms specifically, multi-client workspaces, white-label deliverables, CRM integration, and outcome dashboards make it the operational backbone — not a side tool — of a modern bid-pursuit practice.

## 3. Target Audience & Personas

EU Solicit prioritises **consulting firms** as the primary ICP, with construction/engineering, IT, and municipalities as secondary segments. Personas mapped to the dual-segment (tenders + grants) cross-sell:

- **Consulting Firms (Primary ICP, Highest WTP):** Engineering, IT, management, and EU-grant specialist consultancies. 5–500+ staff. Manage 10–50 client engagements concurrently. Need: multi-client workspace, opportunity discovery + qualification, AI proposal generation, CRM integration (HubSpot/Salesforce/Pipedrive), white-label client deliverables, ISO 27001 + GDPR posture, outcome dashboards. Buyer committee: Bid Director (economic buyer), Bid Manager (daily user), IT/Security (gate), Finance/Procurement (gate). Sales cycle: 4–8 weeks (small specialists), 12–28 weeks (mid-large firms).

- **Construction & Engineering Firms (Secondary):** High-volume public works tenders, complex documentation. Need: requirement extraction, compliance checklists, pricing assistant, scoring simulator. Buyer: Bid Manager + Operations Director.

- **IT Companies (Secondary):** Frequent RFP responses, EU digital programme grants. Need: rapid proposal generation, content reuse, grant eligibility matching. Buyer: Sales Director + Bid Manager.

- **Municipalities (Tertiary, Long Sales Cycle):** EU structural funds, limited in-house capacity. Need: grant eligibility matching, logframe generators, budget builders, programme rule guidance. Buyer: Public Procurement Office + Mayor's Office.

## 4. Product Scope (MVP)

### In Scope:
- Bulgaria (AOP/EOP.bg) and EU-wide (TED, EU Funding & Tenders Portal) opportunity monitoring with multilingual coverage.
- **Multi-client workspace data model** with per-client isolation, RBAC scoping, audit-trail scoping, and cross-workspace admin search (consulting-firm ICP foundational requirement).
- Tiered access model: Free, Starter, Professional, **Pro+ (new tier)**, Enterprise. Plus a **Per-Bid SKU** (€99–€299/bid) available as add-on or standalone for consulting firms passing cost to client engagements.
- AI analysis, proposal generation, and compliance checks via KraftData agentic workflows.
- Self-service bidding with AI-generated guidance and document exports (PDF/DOCX).
- Admin-configurable compliance frameworks per opportunity (BG ZOP, EU Directives 2014/24/EU + 2014/25/EU, Horizon Europe + structural fund rules).
- Stripe billing with subscription, usage metering, and one-time per-bid SKU.
- **CRM integrations** (HubSpot, Salesforce, Pipedrive — read-write sync of opportunities, win/loss outcomes, contact records). Tier-gated to Pro+ and above.
- **Slack and Microsoft Teams notification integrations** (deal alerts, deadlines, bid-team comments). Tier-gated to Professional and above.
- Email alerts and Google/Outlook Calendar sync (Pro/Enterprise).
- Authentication via Email/Password and Google OAuth2; magic-link auth for external collaborators.
- Bid outcome tracking, lessons learned feedback loop, and bid/no-bid decision scorecard.
- **Outcome telemetry & renewal proof brief**: workspace-level dashboard tracking platform-attributed bids, win-rate, content reuse, hours saved; auto-generated monthly Outcome Brief PDF.
- **Trust Center page (`/trust`)** with publicly downloadable security and compliance artefacts (DPA, security overview, ISO 27001 roadmap, sub-processor list, pen-test summary, data-residency confirmation).
- **G2 / Capterra / Gartner Peer Insights** product listings with in-product NPS prompt + opt-in review routing.
- **In-product onboarding programme** with "First Bid in 14 Days" SLA tracking and CSM-assisted milestones.
- Audit trail for all platform actions, scoped to workspace.

### Out of Scope (Post-MVP):
- Direct electronic bid submission (e.g., CAIS, TED eSender integration).
- Digital signature / QES integration.
- ERP integrations (SAP, 1C).
- Consultant marketplace.
- Microsoft OAuth2 login.
- Public buyer-side procurement module (mirroring Mercell's two-sided model).

## 5. Architecture & Integrations

- **Backend:** Python 3.12+, FastAPI, PostgreSQL 16 (strict per-service schema isolation, multi-tenant row-level scoping for workspaces), Redis 7 (Event Streams).
- **Frontend:** Next.js 14 App Router, Zustand, TanStack Query v5, shadcn/ui. New top-level workspace switcher and `/[locale]/workspace/[workspaceId]/...` routing.
- **AI Platform (KraftData):** Orchestrates 26 specialised agents/teams/workflows. Uses SSE for streaming responses and vector stores for RAG (RFP templates, past proposals, regulatory texts) — vector stores partitioned per consulting-firm tenant and per-client workspace.
- **Billing:** Stripe (subscriptions, usage metering, customer portal, **one-time per-bid SKU**).
- **Integrations:** New `integrations-api` service candidate (architect to decide vs. extending `client-api`). OAuth2 token storage via Fernet encryption (per project-context.md Epic 9 standard). Two-layer resilience pattern (circuit breaker + retry) on all outbound HTTP.
- **Compliance certifications:** GDPR (compliant from day 1). **ISO/IEC 27001:2022 audit commences by Month 6 with target certification by Month 12.** SOC 2 deferred (US-aligned; secondary in EU per HST Solutions analysis).
- **Testing & Quality:** Exhaustive ATDD-first approach, Playwright for E2E, and TEA (Test Engineering Architecture) automation. Workspace-scoped negative tests mandatory for all new endpoints (extension of existing cross-tenant pattern).

## 6. Functional Requirements (FRs)

### FR1: Multi-Source Monitoring & Data Pipeline
- **FR1.1:** The system must automatically crawl AOP, TED, and EU Funding & Tenders Portal using background Celery tasks.
- **FR1.2:** The system must normalize extracted data into a unified database schema using a KraftData Agent Team.
- **FR1.3:** The system must match and score opportunities against the user's company profile and past bids, scoped to the active workspace.

### FR2: Tiered Access, Billing & Usage Metering
- **FR2.1:** The system must restrict free-tier users to limited opportunity metadata (name, deadline, location, type, status).
- **FR2.2:** The system must enforce region, CPV sector, and budget constraints based on the user's active Stripe subscription tier.
- **FR2.3:** The system must track usage of AI summaries, drafts, and compliance checks via atomic Redis metering, blocking requests when limits are exceeded. Per-workspace metering rolls up to the tenant subscription.
- **FR2.4:** The system must support a **Per-Bid Pricing SKU**. A "bid" is defined as an opportunity that the user has formally engaged with (bid/no-bid: bid; or proposal draft generated). Per-bid SKUs unlock per-opportunity all features that would otherwise require a paid tier (AI summaries, drafts, compliance checks, calendar sync). Per-bid SKU is purchasable via Stripe one-time payment or as a metered add-on to any subscription tier.
- **FR2.5:** Per-bid SKU must integrate with usage metering: when a user activates a per-bid SKU on opportunity X, all metered-feature consumption on opportunity X bypasses tier-level monthly caps for the lifetime of that opportunity.
- **FR2.6:** Per-bid SKU pricing must be configurable by admin (default tiers: €99 standard / €199 with grant module / €299 with full Enterprise feature stack).
- **FR2.7:** The system must support an in-product NPS prompt (tier-gated; Pro+ and above; quarterly cadence) with opt-in routing to G2/Capterra/Gartner review forms.

### FR3: AI Document Analysis & Intelligence
- **FR3.1:** The system must ingest tender documents (PDF, DOCX) up to 100MB per file and extract structured requirements.
- **FR3.2:** The system must generate one-page executive summaries highlighting key risks, deadlines, and financial thresholds.
- **FR3.3:** The system must automatically generate compliance checklists mapping mandatory requirements to trackable items.

### FR4: Proposal Generation & Optimization
- **FR4.1:** The system must generate initial proposal drafts utilizing the KraftData Proposal Generator Workflow with real-time SSE streaming.
- **FR4.2:** The system must provide a Scoring Simulator Agent to predict evaluator scoring and suggest improvements.
- **FR4.3:** The system must allow multi-user editing with role-based access control (RBAC), scoped to the active workspace.

### FR5: Compliance & Regulatory Framework
- **FR5.1:** Administrators must be able to assign specific regulatory frameworks (e.g., ZOP, 2014/24/EU, Horizon Europe rules) to individual opportunities.
- **FR5.2:** The Compliance Validator Agent must evaluate the proposal solely against the opportunity's assigned regulatory framework.
- **FR5.3:** All state mutations must be logged to an immutable audit trail (`shared.audit_log`), scoped per workspace.

### FR6: Institutional Memory (Knowledge Base)
- **FR6.1:** Users must be able to upload company credentials, past references, and CVs to a vector storage vault, scoped per workspace.
- **FR6.2:** The system must track bid outcomes and execute the Lessons Learned Agent to update the knowledge base.

### FR7: Analytics & Forecasting
- **FR7.1:** The system must display pipeline forecasting, competitor intelligence, and ROI tracking via daily-refreshed materialized views.
- **FR7.2:** The system must execute materialized view refreshes concurrently to avoid read-locks.

### FR8: Multi-Client Workspace Management (Consulting-Firm ICP)
- **FR8.1:** A consulting-firm tenant must be able to create, rename, archive, and delete client workspaces (subject to retention policy).
- **FR8.2:** Each client workspace must scope: opportunities, proposals, uploaded tender documents, generated content, AI-summary usage metering, audit-log entries, compliance framework assignments, and outcome telemetry.
- **FR8.3:** Users at the tenant level must have RBAC roles per workspace (admin, bid_manager, contributor, reviewer, read_only — matching company-level roles per RB v5 §6.10).
- **FR8.4:** Cross-workspace search, content reuse, and analytics must be available to tenant-level admin users.
- **FR8.5:** A workspace must support an "external collaborator" role to invite the end-client to review drafts (read-only or comment-only) without consuming a paid seat. External collaborators authenticate via magic-link with 30-day expiry.
- **FR8.6:** Per-workspace tier metering must roll up to the tenant subscription (one Stripe subscription, N workspaces, shared metered usage with optional per-workspace soft caps).

### FR9: Outcome Telemetry & Renewal Proof
- **FR9.1:** Every opportunity must support a "platform-attributed" tag: was the opportunity discovered via platform feed (vs. external)? This tag persists through bid lifecycle and is queryable for analytics.
- **FR9.2:** Bid outcomes (won, lost, withdrawn) must be capturable per opportunity, with optional evaluator score, contract value, and effort estimate.
- **FR9.3:** A workspace-level Outcome Dashboard must aggregate: bids tracked, bids submitted, bids won, win rate (rolling 12-month), platform-attributed bids and their conversion, content-block reuse rate, AI-summary usage trend, hours-saved estimate (based on documented benchmarks).
- **FR9.4:** A monthly auto-generated "Outcome Brief" (PDF) must be available for download per workspace: top metrics, trend charts, ROI calculation (using configurable assumptions: hourly rate, hours-saved-per-bid baseline).
- **FR9.5:** Outcome metrics must be available via API (Pro+ tier and above) for embedding in customer's own QBR decks.
- **FR9.6:** The Outcome Dashboard must include onboarding-progress tracking: "First Bid in 14 Days" milestone with sub-tasks (workspace created, content uploaded, first opportunity reviewed, first AI summary generated, CRM connected, first bid/no-bid decision logged). CSM is alerted on stalls >7 days.

### FR10: Trust Center & Compliance Posture
- **FR10.1:** A publicly accessible `/trust` page must list current compliance posture: GDPR (compliant), ISO 27001 (in progress with target date), SOC 2 (planned/N-A), data residency (EU only), encryption at rest (AES-256) and in transit (TLS 1.3).
- **FR10.2:** The Trust Center must provide downloadable PDF artefacts: GDPR Data Processing Agreement (DPA), Security Overview, Sub-Processor List, latest pen-test summary (redacted), Business Continuity Plan summary, Data Residency confirmation.
- **FR10.3:** Sub-Processor List must auto-update from a structured source (e.g., `infra/sub-processors.yaml`) so changes are tracked in version control with effective-date entries.
- **FR10.4:** All Trust Center artefacts must be versioned with change-log entries visible to subscribers (and via email to active customer DPAs when sub-processors change, per GDPR Art. 28).

### FR11: CRM and Communications Integrations
- **FR11.1:** The system must support OAuth2-based integration with HubSpot, Salesforce, and Pipedrive. Each integration must be configurable per workspace.
- **FR11.2:** When an opportunity is added to a workspace, the system must optionally create a corresponding CRM record (Deal/Opportunity). When the opportunity status changes (qualified, bid, submitted, won, lost), the CRM record must be updated.
- **FR11.3:** When a CRM Deal/Opportunity is updated externally, the system must reflect the change in EU Solicit (read-back sync) within 5 minutes.
- **FR11.4:** The system must support Slack and Microsoft Teams webhook integrations for: new opportunity alerts matching saved filters, bid deadlines (24h, 48h, 7d ahead), proposal generation completion, compliance-check failures.
- **FR11.5:** All integrations must be tier-gated: Slack/Teams available from Professional tier; CRM available from Pro+ tier.

## 7. Non-Functional Requirements (NFRs)

- **Performance & Latency:** Real-time SSE streaming endpoints must enforce an idle timeout (120s), total timeout (600s), and a 15s heartbeat. Analytics queries must complete in under 500ms.
- **Availability:** **99.9% uptime SLA for the client platform** (revised from 99.5% v1.0). Measurement window: rolling 30 days. Excluded: maintenance windows announced ≥48h in advance, force-majeure (per Stripe-equivalent SLA terms). Service-credit policy applies on breach: 10% credit for ≥99.0% but <99.9%; 25% credit for ≥95.0% but <99.0%; 50% credit for <95.0%.
- **Resilience:** All outbound HTTP calls (including KraftData, Stripe, HubSpot/Salesforce/Pipedrive, Slack/Teams webhooks) must utilize a two-layer resilience pattern (Circuit Breaker wrapping Exponential Backoff Retry).
- **Security:** Strict DB schema isolation per service; row-level scoping for workspaces. HMAC ECDSA signature validation for webhooks. Passwords must be hashed via bcrypt (`run_in_executor`) with a max length of 128 chars. OAuth2 tokens for CRM integrations encrypted at rest via Fernet (per project-context.md Epic 9 standard).
- **Idempotency:** Webhook and event consumers must utilize dual-layer idempotency (DB unique constraint + Redis SETNX).
- **Data Residency & Compliance:** All data must be hosted in the EU, complying with GDPR requirements (including Right to Erasure, PII redaction in logs, and DPA notifications on sub-processor changes).
- **Compliance Certifications:** GDPR-compliant from day 1. **ISO/IEC 27001:2022 audit must commence by Month 6 of platform launch with target certification by Month 12.** A public Trust Center page (per FR10) must be live by Month 2 with the certification roadmap visible.

## 8. User Stories & Acceptance Criteria

### Feature Area: Multi-Client Workspace
**US7: As a Consulting-Firm Bid Director, I want to create and switch between client workspaces so I can manage 30 client engagements without filing-system hacks.**
- **AC1:** User with `tenant_admin` role can create, rename, archive, and delete workspaces.
- **AC2:** Workspace switcher in top-level UI persists active workspace in URL and Zustand store.
- **AC3:** All opportunities, proposals, uploads, content blocks, audit log, and analytics are scoped to the active workspace; cross-workspace access requires `tenant_admin` role.
- **AC4:** Cross-tenant negative test passes: User in tenant A cannot access workspace in tenant B (403).
- **AC5:** Cross-workspace negative test passes: User without `tenant_admin` role assigned only to workspace W1 cannot access workspace W2 within the same tenant (403).

### Feature Area: External Collaborator Invitation
**US8: As a Bid Manager, I want to invite my end-client to review a draft proposal as a comment-only collaborator without consuming a paid seat.**
- **AC1:** Bid Manager (in workspace W) generates a magic-link invite scoped to a specific proposal in W with `comment_only` role.
- **AC2:** Magic-link expires 30 days after issuance; consumed on first use; new link required on expiry.
- **AC3:** External collaborator authenticates via magic-link, sees only the invited proposal (no other workspace data), can comment but not edit or download exports.
- **AC4:** External collaborator does not consume a paid seat; activity is logged in audit trail with `external_collaborator` flag.

### Feature Area: Per-Bid SKU
**US9: As a Consulting-Firm Bid Manager, I want to purchase a per-bid SKU on opportunity X so I can pursue it for one client engagement without upgrading my subscription.**
- **AC1:** Per-bid SKU is purchasable via Stripe one-time payment from the opportunity detail page (€99/€199/€299 by feature stack).
- **AC2:** On successful payment, opportunity X is flagged `per_bid_sku_active=true` for its lifetime.
- **AC3:** All metered-feature requests on opportunity X bypass tier-level monthly caps; usage is still recorded for analytics.
- **AC4:** Per-bid SKU activation is recorded in audit log and visible in workspace billing summary.

### Feature Area: Trust Center & Compliance Posture
**US10: As a Procurement-Committee Member at a prospective customer, I want to download EU Solicit's GDPR DPA, security overview, and ISO 27001 roadmap so I can complete our vendor due diligence.**
- **AC1:** `/trust` page is publicly accessible (no auth required) with current compliance posture summary.
- **AC2:** GDPR DPA, Security Overview, Sub-Processor List, BCP Summary, Pen-Test Summary, and Data Residency Confirmation are downloadable as PDFs.
- **AC3:** ISO 27001 roadmap shows quarterly milestones with current status (Audit Engaged → Gap Analysis → Remediation → Certification Awarded).
- **AC4:** Sub-Processor List is generated from `infra/sub-processors.yaml`; changes trigger versioned change-log entries and customer DPA email notifications (per GDPR Art. 28).

### Feature Area: Outcome Telemetry & Renewal Proof
**US11: As a Bid Director, I want a monthly Outcome Brief PDF for each workspace so I can demonstrate ROI to leadership at renewal.**
- **AC1:** Outcome Dashboard per workspace shows: bids tracked, submitted, won, win rate (rolling 12mo), platform-attributed bid conversion, content-reuse rate, AI-summary usage, hours-saved estimate.
- **AC2:** "First Bid in 14 Days" onboarding milestone tracker is visible; CSM is alerted via Slack/email on milestone stalls >7 days.
- **AC3:** Monthly auto-generated Outcome Brief PDF available for download (1st of each month, covering prior month).
- **AC4:** Outcome metrics available via authenticated API (Pro+ tier) for customer's own QBR decks.
- **AC5:** Hourly rate and hours-saved-per-bid baseline are workspace-configurable; sensible defaults (€60K loaded cost, 19% search-time-savings) pre-populated.

### Feature Area: CRM Integration
**US12: As a Bid Manager on Pro+ tier, I want EU Solicit to sync opportunities and outcomes with my HubSpot CRM so my team's pipeline data stays consistent.**
- **AC1:** OAuth2 connection to HubSpot configurable per workspace by `tenant_admin` or `bid_manager`.
- **AC2:** When an opportunity is added to a workspace, a corresponding HubSpot Deal is created (configurable: auto-create, prompt, off).
- **AC3:** Opportunity status changes (qualified, bid, submitted, won, lost) propagate to HubSpot Deal stage within 60s.
- **AC4:** HubSpot Deal updates (stage, contact, value) propagate back to EU Solicit within 5 minutes.
- **AC5:** Conflict resolution: last-write-wins with full audit-trail entry; user notification on conflict.
- **AC6:** OAuth2 tokens encrypted at rest via Fernet; rotation supported; revocation on workspace archive.

### Feature Area: Slack/Teams Notifications
**US13: As a Bid Team Lead on Professional tier, I want Slack notifications for new opportunities matching my saved filters so my team responds within hours.**
- **AC1:** Slack and Teams incoming-webhook configurable per workspace by `bid_manager`.
- **AC2:** Notifications triggered on: new matching opportunity, deadline 7d/48h/24h ahead, proposal generation completion, compliance-check failure.
- **AC3:** Notification content includes: opportunity name, deadline, link to EU Solicit detail page; respects locale (BG/EN).
- **AC4:** Throttling: max 1 notification per opportunity per event-type per workspace; no flood on bulk re-sync.

### Feature Area: NPS & Review Routing
**US14: As a Pro+ customer who's seen real ROI, I want to be invited to leave a G2/Capterra review at the right moment.**
- **AC1:** Quarterly NPS prompt (configurable, opt-out by tenant admin); shown to active users (≥1 login/week).
- **AC2:** Score 9–10: opt-in routing to G2/Capterra review form (deep link with prefilled product version).
- **AC3:** Score ≤6: routed to CSM with structured feedback collection.
- **AC4:** NPS history retained per workspace (analytics, not surveillance — opt-in disclosure).

### Existing User Stories (carried forward unchanged from v1.0)

#### Feature Area: Opportunity Monitoring
**US1: As a user, I want to receive email alerts for relevant tenders so I don't miss opportunities.**
- **AC1:** User can configure alert preferences (CPV, Region, Budget) per workspace.
- **AC2:** The system sends a digest email (immediate, daily, or weekly) matching the user's preferences.
- **AC3:** Free tier users receive alerts with redacted details linking to a paywall.

#### Feature Area: AI Document Summary
**US2: As a Bid Manager, I want the AI to summarize a 200-page tender document so I can quickly assess if it is worth pursuing.**
- **AC1:** User uploads a ZIP or PDF document (up to 100MB) into the active workspace.
- **AC2:** The system calls the KraftData Executive Summary Agent.
- **AC3:** The system displays a one-page summary highlighting risks, financial thresholds, and key deadlines.
- **AC4:** Usage is incremented in Redis (workspace-scoped, rolled up to tenant); if the tier limit is exceeded and no per-bid SKU is active, a 429 is returned.

#### Feature Area: Proposal Draft Generation
**US3: As a Technical Writer, I want the system to generate a first-draft proposal based on my company's profile and the tender requirements.**
- **AC1:** User initiates the draft generation workflow for a specific opportunity in the active workspace.
- **AC2:** The frontend receives the draft via SSE streaming (`run-stream`), rendering text in real-time.
- **AC3:** The generation logic retrieves relevant past content from the workspace's vector store (RAG).
- **AC4:** If the connection drops, the system correctly cleans up the semaphore permit within 120s.

#### Feature Area: Compliance Validation
**US4: As a Reviewer, I want the system to validate my proposal against the designated regulatory framework before I export it.**
- **AC1:** The system verifies the proposal against the admin-assigned framework (e.g., ZOP, 2014/24/EU, Horizon Europe rules).
- **AC2:** A compliance report is generated, flagging missing mandatory sections or non-compliant formatting.
- **AC3:** The user can export the final proposal and the compliance report as a PDF or DOCX file.

#### Feature Area: Bid/No-Bid Decision
**US5: As a Bid Manager, I want a structured scoring tool to help me decide whether to bid on an opportunity.**
- **AC1:** The system evaluates strategic fit, win probability, and resource availability using the Bid/No-Bid Decision Agent.
- **AC2:** A recommendation (bid / no-bid / conditional) is presented with a score breakdown.
- **AC3:** The user can manually override the recommendation, and the justification is logged in the workspace audit trail.

#### Feature Area: Calendar Synchronization
**US6: As a Bid Manager on the Professional tier, I want my tender deadlines synced to my Google Calendar.**
- **AC1:** User authenticates via Google OAuth2 to grant calendar access (per workspace).
- **AC2:** The system creates and automatically updates calendar events for tracked opportunity deadlines.
- **AC3:** Deleting a tracked opportunity removes the event from the user's calendar.
- **AC4:** Starter tier users attempting to access this feature receive a 403 error with an upgrade prompt.

## 9. Next Steps

1. **Architectural evaluation by Winston** of the PRD-amendment brief (`prd-amendment-2026-04-25.md`) — specifically the cross-cutting and per-change architectural questions. Outputs: feasibility verdict, sequencing, dependency map, infra-impact estimate.
2. **Sprint-change-proposal** (via `bmad-correct-course`) to formally inject the amendment into the sprint plan, including a coordinator story for the multi-workspace migration and dependency-managed rollout of subsequent epics.
3. **Epic & Story decomposition** (via `bmad-create-epics-and-stories`) producing Epics 14–20 per the recommended sequencing in the amendment brief.
4. **TEA validation** of the amended PRD against test architectural standards; generate updated traceability matrices.
5. **ISO 27001 audit-prep programme** to be initiated as a parallel sprint-spanning workstream, separate from feature epics.
6. Validate amended PRD against `prd-amendment-2026-04-25.md` (companion document) to ensure no requirement was lost in the fold.
