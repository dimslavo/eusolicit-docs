# EU Solicit — Product Requirements Brief v4

**Platform**: EU Solicit | **Domain**: EUSolicit.com | **Version**: 4.0 | **Date**: 2026-04-04 | **Status**: Draft

---

## 1. Product Vision

**EU Solicit** is a SaaS platform that automates the full lifecycle of public procurement and EU grant applications — from opportunity discovery through AI-guided proposal preparation. The platform targets **Bulgaria and EU-wide opportunities** as its initial coverage scope.

EU Solicit is powered by an **Agentic AI backend** (KraftData / Sirma AI) that orchestrates multi-step workflows through specialized AI agents, while a **central data store** serves as the single source of truth for all opportunity data consumed by the client-facing application at **EUSolicit.com**.

> **MVP Scope Note**: Electronic bid submission and direct portal integration (CAIS, TED eSender) are **out of scope** for the MVP. Clients bid/apply on their own using guidance and generated materials from EU Solicit.

---

## 2. Problem Statement

Companies and municipalities in Bulgaria and across Europe face significant friction when engaging with public procurement and EU funding:

- Discovering relevant tenders requires manual monitoring across multiple portals (AOP, TED, national grant databases).
- Procurement documentation packages are large, complex, and time-consuming to parse.
- Writing competitive proposals is expensive — firms routinely pay consultants thousands of euros per bid.
- Compliance validation is error-prone and manual.
- EU grant applications add additional complexity: logframes, budget models, co-financing rules, consortium requirements.

General-purpose LLMs (e.g., ChatGPT) cannot replace a purpose-built system because they lack structured RFP template libraries, scoring models, compliance checkers, and persistent institutional memory.

---

## 3. Target Customers

| Segment | Pain Point | Willingness to Pay |
|---------|-----------|-------------------|
| Construction firms | High volume of public works tenders, complex documentation | High |
| IT companies | Frequent RFP responses, EU digital programme grants | High |
| Engineering firms | Infrastructure tenders, technical proposal complexity | High |
| Consulting firms | Multi-client bid management, grant writing as a service | Very High |
| Municipalities | EU structural fund applications, limited in-house capacity | Medium-High |
| Grant agencies | Programme management, application quality assessment | Medium |

---

## 4. Pricing Model & Tier Access Controls

### 4.1 Tier Definitions

| Tier | Price (EUR/month) | Target |
|------|-------------------|--------|
| **Free** | 0 | Any registered company |
| **Starter** | 29 | Small firms |
| **Professional** | 59 | Mid-size firms |
| **Enterprise** | 109+ | Large firms / consultancies |
| **Per-bid add-on** | Variable | All paid tiers |

### 4.2 Free Tier — Limited Visibility

Free-tier users can browse the EU Solicit opportunity catalogue but see only restricted metadata per opportunity:

- Opportunity / grant name
- Due date / submission deadline
- Location / contracting authority region
- Opportunity type (tender vs. grant)
- Status (open / closing soon / closed)

All other data (full documentation, budget details, evaluation criteria, AI analysis, proposal tools) is gated behind paid tiers. The free tier serves as a lead-generation funnel.

### 4.3 Tier Access Limitations

Paid tiers are differentiated by **region**, **industry**, and **budget range** access:

| Dimension | Starter | Professional | Enterprise |
|-----------|---------|-------------|-----------|
| **Region** | Bulgaria only | Bulgaria + 3 EU countries | All EU (Bulgaria + EU-wide) |
| **Industry** | 1 CPV sector | Up to 3 CPV sectors | Unlimited CPV sectors |
| **Budget range** | Up to €500K | Up to €5M | Unlimited |
| **AI summaries** | 10 / month | 50 / month | Unlimited |
| **Proposal drafts** | — | 5 / month | Unlimited |
| **Compliance checks** | — | 5 / month | Unlimited |
| **Team members** | 1 user | Up to 5 users | Unlimited |
| **Calendar sync** | iCal feed only | Google + Outlook sync | Google + Outlook sync |
| **API access** | — | — | Full API + white-label |

### 4.4 Self-Service Bidding Model

Clients bid and apply for grants on their own. EU Solicit provides:

- AI-generated proposal drafts, checklists, and compliance reports as downloadable documents (PDF / DOCX).
- Step-by-step guided workflows walking the user through the submission process for the specific opportunity type.
- Portal-specific submission instructions (e.g., "submit via CAIS portal at [URL], using form X").
- Deadline reminders via email alerts and calendar sync.
- Document completeness checks before the user submits externally.

> **Note**: EU Solicit does NOT submit bids or applications on behalf of clients. Electronic submission integration is out of scope for the MVP.

### 4.5 Billing & Subscription Management

- **Billing provider**: Stripe (supports EUR, recurring subscriptions, usage-based metering, customer portal).
- **Subscription lifecycle**: Free registration → optional upgrade to paid tier → Stripe Checkout for payment → automatic renewal → Stripe Customer Portal for self-service management (upgrade, downgrade, cancel, update payment method).
- **Usage metering**: AI summaries, proposal drafts, and compliance checks are tracked per billing period. When a user reaches their monthly limit, the platform blocks further requests for that feature and presents an upgrade prompt.
- **Trial period**: 14-day free trial of Professional tier for new registrations (no credit card required).
- **Invoicing**: Stripe-generated invoices with EU VAT handling. Enterprise tier supports custom invoicing.

---

## 5. Architecture Overview

### 5.1 Agentic AI Platform (KraftData / Sirma AI)

All AI-driven intelligence, analysis, and generation tasks are orchestrated through the **KraftData Agentic AI platform** (`stage.sirma.ai`). The platform provides:

- **Agents** — Purpose-built AI agents for specific tasks (document analysis, proposal generation, compliance checking, market intelligence).
- **Teams** — Coordinated multi-agent groups that handle complex workflows requiring sequential or parallel agent collaboration.
- **Workflows** — Multi-step orchestrated pipelines that chain agents, data retrieval, validation, and output generation.
- **Storage Resources** — Vector stores for RAG (Retrieval Augmented Generation) enabling domain-specific knowledge bases (RFP templates, past proposals, regulatory texts).
- **Sessions** — Persistent conversation contexts that maintain state across multi-turn interactions.
- **Evaluations** — Built-in eval framework for measuring agent accuracy and output quality.
- **Policies** — Multi-level governance controls (organization, project, team, agent) for output safety and compliance.
- **Webhooks** — Event-driven notifications for asynchronous workflow completion.

### 5.2 Data Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  DATA SOURCE PIPELINES                       │
│  (Orchestrated by Agentic AI Workflow Agents)                │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │ AOP      │  │ TED      │  │ EU Grant │                  │
│  │ Crawler  │  │ Crawler  │  │ Portals  │                  │
│  │ Agent    │  │ Agent    │  │ Agent    │                  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │
│       │              │              │                        │
│       └──────────────┴──────┬───────┘                        │
│                             │                                │
│                    ┌────────▼────────┐                       │
│                    │  Normalization  │                       │
│                    │  & Enrichment   │                       │
│                    │  Agent Team     │                       │
│                    └────────┬────────┘                       │
└─────────────────────────────┼───────────────────────────────┘
                              │
                     ┌────────▼────────┐
                     │  CENTRAL DATA   │
                     │  STORE (DB)     │
                     │  PostgreSQL 16  │
                     └────────┬────────┘
                              │
                     ┌────────▼────────┐
                     │  EU SOLICIT     │
                     │  EUSolicit.com  │
                     └─────────────────┘
```

### 5.3 Integration Pattern with KraftData API

The EU Solicit platform integrates with KraftData via the **Client API v1** (`/client/api/v1/*`) using API key authentication. Key integration points:

| Platform Capability | KraftData API Endpoint | Purpose |
|---|---|---|
| Run analysis agent | `POST /client/api/v1/agents/{agentId}/run` | Execute document analysis, relevance scoring |
| Stream proposal generation | `POST /client/api/v1/agents/{agentId}/run-stream` | Real-time proposal drafting with SSE |
| Execute multi-agent workflows | `POST /client/api/v1/workflows/{workflowId}/run-stream` | End-to-end pipeline execution |
| Run agent teams | `POST /client/api/v1/teams/{teamId}/run` | Coordinated multi-agent tasks |
| Manage knowledge bases | `POST /client/api/v1/storage-resources` | Upload RFP templates, past proposals |
| Upload reference documents | `POST /client/api/v1/storage-resources/{id}/files` | Feed vector stores for RAG |
| Evaluate agent quality | `POST /client/api/v1/eval-runs` | Continuous quality monitoring |
| Webhook notifications | `PUT /api/webhooks/subscriptions/{id}` | Async completion callbacks |

---

## 6. High-Level Features

### 6.1 Opportunity Intelligence Engine

**Data pipeline agents** (orchestrated via KraftData Workflows) continuously crawl and normalize tender data into the central DB.

- **Multi-source monitoring** (MVP: Bulgaria + EU): Automated crawling of AOP (Bulgarian Agency for Public Procurement), TED (Tenders Electronic Daily), and EU funding portals. Each source has a dedicated crawler agent. Additional national portals can be added post-MVP.
- **Data normalization agent team**: A KraftData Team that standardizes heterogeneous data formats (HTML, PDF, XML) into a unified schema in the central data store.
- **Smart matching agent**: AI agent that scores opportunity relevance against company profile, past bids, capabilities, certifications, and financial capacity.
- **Configurable email alerts**: Users configure alert preferences (CPV sectors, regions, budget range, deadline proximity, contracting authority). The platform sends matching opportunity digest emails on a configurable schedule (immediate, daily, weekly). In-app and Slack notification channels are deferred to post-MVP.
- **Competitor tracking agent**: Monitors publicly available award data to identify competitor bidding patterns, win rates, and pricing. Results are displayed in a dedicated Competitor Intelligence view accessible to Professional and Enterprise tiers.
- **Pipeline forecasting agent**: Predicts upcoming tenders based on historical procurement cycles, budget planning documents, and framework agreement renewal schedules. Forecasts are shown as a "Predicted Opportunities" section on the dashboard.

### 6.2 Document Analysis & Intelligence

Powered by dedicated KraftData agents with access to vector stores containing regulatory and template knowledge.

- **AI document parser agent**: Ingests full tender packages (PDF, DOCX, ZIP) and extracts structured data — requirements, evaluation criteria, deadlines, mandatory documents, financial thresholds.
- **Executive summary agent**: Generates one-page summaries of complex 200+ page procurement documents, highlighting key risks and requirements.
- **Requirement checklist builder**: Auto-generates compliance checklists mapping every mandatory requirement to a trackable item.
- **Clause risk flagging agent**: Identifies unusual or high-risk contractual clauses (unlimited liability, IP assignment, penalty structures) and flags for legal review.
- **Multi-language processing**: The KraftData agents process source documents in their original language (Bulgarian, English, German, French, Romanian) and generate outputs (summaries, checklists, proposals) in the user's preferred language. The platform UI supports Bulgarian and English in the MVP, with additional languages post-MVP.
- **Document storage & management**: Tender documents are uploaded to S3-compatible object storage, linked to the opportunity record. Maximum file size: 100MB per file, 500MB per tender package. Documents are retained for the lifetime of the opportunity + 2 years (configurable). Virus scanning via ClamAV on upload.

### 6.3 Proposal Generation & Optimization

Orchestrated as a KraftData Workflow chaining multiple specialized agents, with streaming output (`run-stream`) for real-time user feedback.

- **Template library**: Pre-built, sector-specific proposal templates in vector storage resources, accessible via RAG.
- **AI draft generator workflow**: Multi-agent workflow that generates first-draft proposals from tender requirements + company profile, including technical approach, methodology, team composition, and timeline.
- **Scoring model simulator agent**: Predicts evaluator scoring against published criteria and suggests specific improvements to maximize points. Results are presented as a scorecard with per-criterion breakdown and improvement suggestions.
- **Compliance validator agent**: Automated pre-submission check — mandatory sections addressed, page limits respected, required attachments present, formatting rules followed. Validation runs against the compliance framework assigned to that opportunity.
- **Pricing assistant agent**: Suggests competitive pricing based on historical award data, budget ceilings, and market benchmarks.
- **Version control & collaboration**: Multi-user editing with role-based access (bid manager, technical writer, financial analyst, legal reviewer).
- **Document export**: Generated proposals, checklists, and compliance reports exportable as PDF and DOCX.

### 6.4 EU Grant Specialization Module

A dedicated KraftData agent team configured for EU programme-specific knowledge.

- **Grant eligibility matcher agent**: Maps company/municipality profile against active EU programmes (Horizon Europe, Digital Europe, structural funds, CAP, Interreg).
- **Budget builder agent**: AI-assisted budget construction following EU cost category rules, overhead calculations, and co-financing requirements.
- **Consortium finder**: Suggests potential partners from a database of organizations that have participated in similar grants.
- **Logframe generator agent**: Auto-generates logical frameworks, work packages, Gantt charts, and deliverable tables from project narrative descriptions.
- **Reporting template generator**: Post-award periodic report and financial statement templates pre-filled from project data.

### 6.5 Knowledge Base & Institutional Memory

Leverages KraftData **Storage Resources** (vector stores) for persistent, searchable organizational knowledge.

- **Company profile vault**: Centralized repository of company credentials, past project references, team CVs, certifications, financial statements — auto-populated into proposals via RAG.
- **Bid history & outcome tracking**: Users record bid outcomes (won, lost, withdrawn) with optional evaluator feedback and scores. This data feeds the Lessons Learned Agent and analytics dashboards.
- **Bid history analytics agent**: Tracks win/loss rates, average scores, common feedback themes, and improvement trends. Generates periodic reports and recommendations.
- **Reusable content blocks**: Library of approved boilerplate sections (company overview, quality management, sustainability policy) stored in vector stores.
- **Lessons learned engine**: Post-bid-outcome analysis agent that captures what worked, what didn't, and generates improvement recommendations. Insights are fed back into the Proposal Generator Workflow to improve future drafts.

### 6.6 Workflow & Deadline Management

- **Bid/no-bid decision agent**: Structured scoring tool evaluating strategic fit, win probability, resource availability, and margin potential. Presents a recommendation (bid / no-bid / conditional) with a score breakdown and key risk factors. Users can override the recommendation with a justification that is logged.
- **Task orchestration**: Automated workflow from opportunity identification through final submission, with task assignments, dependencies, and deadline tracking.
- **Calendar integration (MVP)**:
  - **iCal feed**: All tiers get a subscribable iCal URL that includes deadlines for tracked opportunities. Works with any calendar application.
  - **Google Calendar sync**: Professional and Enterprise tiers. Two-way sync via Google Calendar API (OAuth2). Creates/updates/deletes calendar events for tender deadlines, clarification periods, and site visit dates.
  - **Microsoft Outlook sync**: Professional and Enterprise tiers. Two-way sync via Microsoft Graph API (OAuth2). Same functionality as Google Calendar.
- **Approval gates**: Configurable review and sign-off stages before submission (technical, legal, management).

### 6.7 Compliance & Regulatory Intelligence

- **Admin-configurable compliance framework**: Platform administrators can assign the applicable compliance and regulatory framework to each individual opportunity. Framework selection is driven by:
  - **Country or region** of the contracting authority (e.g., Bulgarian ZOP for national tenders).
  - **EU-level regulation** when the opportunity is funded or governed by EU programmes (e.g., EU Public Procurement Directives 2014/24/EU, 2014/25/EU for EU-funded tenders).
  - **Programme-specific rules** for EU grants (Horizon Europe, structural funds, etc.).
  - Hybrid scenarios where both national and EU regulations apply simultaneously.
- **Framework auto-suggestion**: The platform suggests an applicable framework based on the opportunity's country, source, and funding type. The admin confirms or overrides.
- **Regulation tracker agent**: Monitors changes to Bulgarian ZOP, EU procurement directives, and grant programme rules.
- **ESPD auto-fill**: Pre-populates European Single Procurement Document from stored company data.
- **Compliance validation against assigned framework**: The Compliance Checker Agent validates proposals against the specific regulatory framework assigned to that opportunity by the admin — not a generic ruleset.
- **Audit trail**: Full traceability of all platform actions — compliance framework assignments, proposal edits, approval decisions, bid/no-bid overrides, and document downloads. Stored in an immutable audit log with timestamp, user ID, action type, entity reference, and before/after values.

> **MVP Scope**: Digital signature integration and electronic bid submission are out of scope. Users submit externally following platform-generated guidance.

### 6.8 Analytics & Reporting Dashboard

- **Market intelligence agent**: Sector-level analytics on procurement volumes, average contract values, most active contracting authorities, seasonal trends. Presented as interactive charts and filterable tables.
- **ROI tracker**: Calculates return on bid investment (preparation time + cost vs. contract value won).
- **Team performance metrics**: Productivity per bid manager — bids submitted, win rate, average preparation time.
- **Competitor intelligence view**: Aggregated competitor bidding patterns, win rates, and pricing benchmarks (Professional and Enterprise tiers).
- **Pipeline forecasting view**: Predicted upcoming opportunities based on historical patterns.
- **Usage dashboard**: Current billing period usage vs. tier limits (AI summaries consumed, proposal drafts remaining, etc.).
- **Custom reporting**: Exportable reports for management — pipeline value, success rate trends, upcoming deadlines.

### 6.9 Integration Layer

**MVP integrations:**

- **KraftData API**: Full integration via REST API with API key auth, SSE streaming, webhook callbacks.
- **Stripe**: Subscription billing, usage metering, customer portal, invoicing.
- **Google Calendar API**: Two-way calendar sync for Professional and Enterprise tiers (OAuth2).
- **Microsoft Graph API**: Two-way Outlook calendar sync for Professional and Enterprise tiers (OAuth2).
- **Google OAuth2**: Social login for authentication.
- **SMTP / Transactional email**: Email alerts and notifications (SendGrid or AWS SES).
- **S3-compatible object storage**: Tender document files, generated proposals, exports.

**MVP webhook handling**: EU Solicit registers webhook subscriptions with KraftData to receive async completion events for long-running agent/workflow executions. Webhook handler verifies signatures, processes events, and updates relevant records.

**Public API for enterprise clients**: RESTful API for embedding tender intelligence into third-party systems (Enterprise tier only). Documented via auto-generated OpenAPI spec.

**Post-MVP integrations:**

- E-procurement portal connectors (CAIS, TED eSender) for direct electronic submission.
- ERP integration (SAP, 1C, local Bulgarian ERPs) via webhook-based adapter pattern.
- CRM integration (Salesforce, HubSpot, Pipedrive) via OAuth2 + bidirectional sync.
- Slack / Microsoft Teams notification channels.
- Digital signature / QES integration (eIDAS-compliant, ETSI standards).
- Consultant marketplace module.

### 6.10 Platform & Commercial Features

- **Multi-tenant architecture**: White-label option for consulting firms (Enterprise tier). Includes custom subdomain (`clientname.eusolicit.com`), configurable logo/brand colors, and custom email sender domain.
- **Role-based access control**: Admin, bid manager, contributor, reviewer, read-only — with per-tender permission granularity.
- **KraftData Policy enforcement**: Organization and project-level AI governance via KraftData's multi-level policy system.
- **Authentication**: Email/password registration with Google OAuth2 social login. Microsoft OAuth2 deferred to post-MVP.

---

## 7. Pre-Built Agent Inventory (KraftData)

The following agents/teams/workflows need to be configured on the KraftData platform:

| Agent / Team / Workflow | Type | Purpose | KraftData Capability Used |
|---|---|---|---|
| AOP Crawler Agent | Agent | Crawl Bulgarian procurement portal | Agent + HTTP Requests |
| TED Crawler Agent | Agent | Crawl EU tenders portal | Agent + HTTP Requests |
| EU Grant Portal Agent | Agent | Crawl EU funding portals | Agent + HTTP Requests |
| Data Normalization Team | Team | Standardize multi-source data | Team (multi-agent) |
| Relevance Scoring Agent | Agent | Match opportunities to company profiles | Agent + Storage Resources (RAG) |
| Document Parser Agent | Agent | Extract structured data from tender docs | Agent + Storage Resources |
| Executive Summary Agent | Agent | Generate one-page tender summaries | Agent |
| Compliance Checker Agent | Agent | Validate proposal against assigned framework | Agent + Storage Resources |
| Clause Risk Agent | Agent | Flag high-risk contract clauses | Agent |
| Proposal Generator Workflow | Workflow | End-to-end proposal drafting pipeline | Workflow (chained agents) |
| Scoring Simulator Agent | Agent | Predict evaluator scores with scorecard | Agent + Storage Resources |
| Pricing Assistant Agent | Agent | Benchmark-based pricing suggestions | Agent + Storage Resources |
| Grant Eligibility Agent | Agent | Match profiles to EU programmes | Agent + Storage Resources |
| Budget Builder Agent | Agent | EU-compliant budget construction | Agent |
| Logframe Generator Agent | Agent | Generate logical frameworks | Agent |
| Bid/No-Bid Decision Agent | Agent | Score and recommend bid/no-bid with rationale | Agent |
| Market Intelligence Agent | Agent | Sector analytics and trend analysis | Agent + Storage Resources |
| Regulation Tracker Agent | Agent | Monitor regulatory changes | Agent + HTTP Requests |
| Lessons Learned Agent | Agent | Post-bid analysis and improvement recs | Agent + Storage Resources |
| Competitor Analysis Agent | Agent | Track competitor bidding patterns | Agent + Storage Resources |
| Pipeline Forecasting Agent | Agent | Predict upcoming tenders from history | Agent + Storage Resources |
| Framework Suggestion Agent | Agent | Auto-suggest compliance framework per opportunity | Agent + Storage Resources |

---

## 8. Vector Store / Knowledge Base Requirements

The following **Storage Resources** need to be provisioned on KraftData:

| Storage Resource | Content | Used By |
|---|---|---|
| RFP Templates Store | Sector-specific proposal templates | Proposal Generator Workflow |
| Regulatory Knowledge Store | ZOP, EU directives, programme rules, compliance frameworks | Compliance Checker, Regulation Tracker, Framework Suggestion Agent |
| Company Profiles Store | Per-client credentials, CVs, references | All proposal-related agents |
| Historical Bids Store | Past proposals, outcomes, evaluator feedback | Scoring Simulator, Lessons Learned |
| Market Data Store | Award data, pricing benchmarks, competitor info | Pricing Assistant, Market Intelligence, Competitor Analysis |
| EU Programmes Store | Programme guides, eligibility criteria, templates | Grant Eligibility, Budget Builder |

---

## 9. Non-Functional Requirements

- **Availability**: 99.5% uptime SLA for the EU Solicit client platform.
- **Latency**: Streaming responses (`run-stream`) for all user-facing AI operations; no blocking waits for proposal generation.
- **Data residency**: All tender data and client information stored within EU (GDPR compliance). Infrastructure hosted in EU region.
- **Security**: API key rotation, JWT RS256 auth, audit logging, encryption at rest (AES-256) and in transit (TLS 1.3). Virus scanning on all file uploads.
- **Scalability**: Central DB must handle 10K+ active tenders with full-text search; KraftData agents must support concurrent execution per tenant.
- **Monitoring**: KraftData eval runs for continuous agent quality measurement; alerting on degradation.
- **Authentication**: Email/password + Google OAuth2 (MVP). RS256 JWT with 15-min access token + 7-day refresh token.
- **Compliance**: GDPR-compliant data handling. Right to erasure. Data processing agreements with KraftData.

---

## 10. Strategic Differentiation

The moat is not AI text generation — it is the combination of:

1. **Structured procurement data** (CPV codes, ESPD schemas, scoring matrices) in purpose-built vector stores.
2. **Domain-specific compliance logic** enforced through specialized KraftData agents with admin-configurable regulatory frameworks per opportunity.
3. **Institutional memory** per client, accumulated over time in dedicated storage resources, with feedback loops improving proposal quality.
4. **Agentic orchestration** — multi-agent teams and workflows that execute complex procurement tasks end-to-end, not just answer questions.
5. **Tiered access model** with a free discovery funnel that converts to paid via progressive feature unlocking across region, industry, and budget dimensions.

---

## 11. MVP Scope Summary

| In Scope | Out of Scope (Post-MVP) |
|----------|------------------------|
| Bulgaria + EU-wide opportunity coverage | Other non-EU countries |
| AOP, TED, EU grant portal crawlers | National portals of individual EU member states |
| Free tier with limited metadata visibility | — |
| Paid tiers gated by region, industry, budget | — |
| AI analysis, proposal generation, compliance checks | — |
| Self-service bidding with AI-generated guidance | Electronic bid submission on behalf of clients |
| Admin-configurable compliance frameworks per opportunity | — |
| Audit trail for all platform actions | — |
| KraftData agent/workflow integration | — |
| KraftData webhook handling | — |
| Email alerts (configurable digest) | Slack / Teams notification channels |
| Stripe billing with usage metering | — |
| Google Calendar + Outlook sync (Pro/Enterprise) | — |
| iCal feed (all tiers) | — |
| Google OAuth2 social login | Microsoft OAuth2 |
| Email/password authentication | — |
| Bid outcome tracking + lessons learned feedback loop | — |
| Bid/no-bid decision with scorecard + override | — |
| Competitor intelligence view (Pro/Enterprise) | — |
| Pipeline forecasting view | — |
| Document upload with virus scanning + retention | — |
| White-label subdomain (Enterprise) | Full white-label customization |
| PDF/DOCX export of proposals + reports | — |
| — | Digital signature / QES integration |
| — | E-procurement portal connectors (CAIS, TED eSender) |
| — | Consultant marketplace |
| — | ERP integrations (SAP, 1C) |
| — | CRM integrations (Salesforce, HubSpot, Pipedrive) |

---

*Document version: 4.0 | Date: 2026-04-04 | Status: Draft*
