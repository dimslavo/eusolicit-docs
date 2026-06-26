---
stepsCompleted: ["step-01-init", "step-02-discovery", "step-02b-vision", "step-02c-executive-summary", "step-03-success", "step-04-journeys", "step-05-domain", "step-06-innovation", "step-07-project-type", "step-08-scoping", "step-09-functional", "step-10-nonfunctional", "step-11-polish", "step-12-complete"]
releaseMode: "phased"
inputDocuments: [
  "eusolicit-docs/planning-artifacts/research/market-eu-govwin-tender-intelligence-research-2026-04-25.md",
  "eusolicit-docs/project-context.md",
  "eusolicit-docs/EU_Solicit_Solution_Architecture_v5.md",
  "eusolicit-docs/EU_Solicit_Requirements_Brief_v5.md",
  "eusolicit-docs/EU_Solicit_UX_Supplement_v1.md",
  "eusolicit-docs/planning-artifacts/epics.md",
  "eusolicit-docs/architecture-vs-requirements-gap-analysis.md",
  "eusolicit-docs/EU_Solicit_PRD_v2.md",
  "eusolicit-docs/planning-artifacts/product-brief.md"
]
workflowType: 'prd'
classification:
  projectType: 'SaaS B2B Platform'
  domain: 'GovTech'
  complexity: 'High'
  projectContext: 'Brownfield'
---

# Product Requirements Document - eusolicit

**Author:** Deb
**Date:** 2026-04-27

## Executive Summary

EU Solicit is a SaaS platform designed to automate the full lifecycle of public procurement and EU grant applications. It targets Bulgarian and EU-wide opportunities, aiming to replace manual tender monitoring, fragmented tooling, and expensive bid consultants with a single, AI-powered platform. The vision is to democratize access to public procurement by providing an affordable, AI-driven system that automates the entire bidding lifecycle for companies of all sizes.

The core differentiator is a unified, end-to-end workflow powered by AgenticSAI's sophisticated multi-agent AI system. This approach replaces a fragmented market of single-point solutions and manual consulting services. The core insight is that modern Agentic AI is now capable of handling the complex, unstructured data and nuanced analytical tasks of tender evaluation and proposal writing, which previously required significant, high-cost human expertise.

## Project Classification

*   **Project Type**: SaaS B2B Platform
*   **Domain**: GovTech
*   **Complexity**: High
*   **Project Context**: Brownfield

## Success Criteria

### User Success

*   **Time-to-Value:** A new user can find a relevant, qualified opportunity and generate a first-draft proposal within one session, without prior training.
*   **Confidence:** Users feel confident submitting bids prepared with the platform, knowing they have passed automated compliance and quality checks, reducing the fear of administrative rejection.
*   **Win Rate:** Users report a measurable increase in their bid win rates and a reduction in the time and cost per bid.
*   **Empowerment:** Small businesses and NGOs feel empowered to compete for tenders and grants that were previously out of reach due to complexity and cost.

### Business Success

*   **Active Paid Customers:** Achieve 50 active paid customers within 6 months post-launch.
*   **Net Revenue Retention (NRR):** Maintain an NRR of >= 110%, indicating strong customer satisfaction and expansion.
*   **Conversion Rate:** Achieve a free-to-paid conversion rate of 5% and a trial-to-paid conversion rate of 20%.
*   **Customer Satisfaction:** Maintain a CSAT score of >= 90% and an NPS score of > 40.
*   **Strategic Goal:** Become the leading bid intelligence and proposal automation platform for the Bulgarian market and a top 3 player in the CEE region within 3 years.

### Technical Success

*   **Availability:** Platform availability of >= 99.5% for all services.
*   **Performance:** API p95 latency of < 200ms and AI generation TTFB (Time to First Byte) of < 500ms.
*   **Data Integrity:** Zero instances of cross-tenant data leakage. Full audit trail for all critical actions.
*   **Scalability:** The system can scale to handle 10,000+ active tenders and concurrent AI agent execution per tenant without performance degradation.
*   **AI Quality:** Maintain >90% accuracy on critical data extraction tasks (deadlines, budgets) and achieve a human-review-rejection rate of <10% for AI-generated content.

### Measurable Outcomes

*   **Demo Milestone:** A functional end-to-end demo is ready within 16 weeks, covering registration, opportunity discovery, and AI proposal drafting.
*   **MVP Launch:** Public MVP launched within 28 weeks, with 100 registered companies and 25 active paid subscribers in the first 90 days.
*   **Proposal Generation:** 200 proposals generated through the platform within the first 90 days post-launch.

## Project Scoping & Phased Development

The project will be developed in phases to de-risk the investment, validate the core value proposition quickly, and build momentum. The approach is to launch a feature-focused MVP that solves the most critical pain point for a specific user segment, then expand functionality and market reach in subsequent phases.

### MVP Strategy & Philosophy

*   **MVP Approach:** Problem-Solving MVP. The primary goal is to validate that an AI-assisted workflow can dramatically reduce the time and effort required to produce a competitive bid. We will focus on the "speed and quality" value proposition for the Professional User persona (Elena).
*   **Target Market:** The initial market is small-to-medium sized Bulgarian consulting firms and businesses bidding on national public tenders.
*   **Resource Requirements:** The MVP can be delivered by a single full-stack development team (4-6 engineers), a product manager, and a UI/UX designer, leveraging the managed services of the AgenticSAI AI platform.

### Phase 1: MVP Feature Set

**Core User Journey Supported:** Elena's journey from discovering a tender to submitting a high-quality, AI-assisted proposal.

**Must-Have Capabilities:**
*   **User & Company:** Secure user registration, login, and a simple company profile.
*   **Data Pipeline:** Automated daily crawling of AOP and TED portals.
*   **Opportunity Discovery:** Searchable and filterable list of opportunities.
*   **AI Analysis:** AI-powered executive summary and requirements checklist generation.
*   **Proposal Generation:** AI-assisted first draft generation.
*   **Editing & Compliance:** A single-user rich text editor (Tiptap) to refine the draft, and a basic compliance check against extracted requirements.
*   **Billing:** Integration with Stripe for manual subscription to Starter and Professional tiers. 14-day free trial flow.

**Key Exclusions at MVP:** Advanced collaboration (team roles, approvals), EU Grant specific features, white-labeling, and deep analytics.

### Post-MVP Features

**Phase 2: Growth (Post-MVP)**
This phase focuses on expanding the user base by adding collaboration features and widening the scope to include EU Grants.
*   **Collaboration:** Full implementation of the multi-user collaborative editor with section locking, comments, @mentions, and version history.
*   **Workflow:** Task management and approval workflows.
*   **EU Grants:** Introduction of the grant eligibility matcher and budget builder.
*   **Notifications:** Calendar sync (Google/Outlook) and advanced email alerts.
*   **Analytics:** Launch of the initial set of user-facing dashboards (ROI tracker, team performance).

**Phase 3: Expansion (Vision)**
This phase aims to solidify market leadership and begin to realize the long-term vision of a fully autonomous platform.
*   **Full Automation:** Introduction of more advanced AI agents that can suggest revisions and refinements with less human input.
*   **Enterprise Features:** White-labeling, Enterprise API for external integrations, and advanced tenant management features.
*   **Marketplace & Ecosystem:** Building out the consultant marketplace and deep integrations with CRM/ERP systems.
*   **Global Expansion:** Adapting the platform for new countries and procurement law systems.

### Risk Mitigation Strategy

*   **Technical Risk (AI Quality):** The core risk is that the AI's output is not good enough.
    *   **Mitigation:** The MVP is a **human-in-the-loop** system. The AI assists, but the user is always in control. This allows us to launch and provide value even if the AI is not perfect, while gathering the data needed to improve it.
*   **Market Risk (Willingness to Pay):** The risk that users will not pay for an AI-assisted tool.
    *   **Mitigation:** The 14-day free trial of the Professional tier allows users to experience the full value proposition before committing. The tiered pricing model provides a low-cost entry point (Starter tier) to further reduce the barrier to adoption.
*   **Resource Risk (Scope Creep):** The risk that the project becomes too large to deliver the MVP in a timely manner.
    *   **Mitigation:** This phased approach provides a clear and disciplined focus for the development team. Any feature not in the "Must-Have Capabilities" list for the MVP is deferred by default.

## User Journeys

### Journey 1: Elena - The Overwhelmed Bid Manager (Professional User - Success Path)

*   **Persona**: Elena is a bid manager at a mid-sized consulting firm in Sofia. She's smart and ambitious but completely swamped. Her team of 5 is trying to manage 10-15 bids at any given time, and the process is a chaotic mess of spreadsheets, shared drives, and endless email chains. They've lost bids not on merit, but because they missed a deadline or a mandatory document. She feels like she's failing her team and the company.

*   **Opening Scene**: It's 7 PM on a Tuesday. Elena is staring at a 150-page tender document that just dropped for a crucial project. The deadline is in two weeks. Her stomach sinks. It will take her and a junior analyst at least two full days just to read, summarize, and create a compliance checklist. That's time they don't have.

*   **Rising Action**: The next morning, a colleague mentions a new platform, EU Solicit, that he saw on LinkedIn. Skeptical but desperate, Elena signs up for the Professional tier free trial.
    1.  **Onboarding**: She completes the company profile, inputting their sectors of expertise (CPV codes) and key personnel.
    2.  **Discovery**: She uploads the massive tender PDF. Within 90 seconds, a one-page AI executive summary appears, along with a structured requirements checklist and a list of flagged high-risk clauses. Her jaw drops. This would have taken days.
    3.  **Collaboration**: She invites her team. For the new tender, she clicks "Generate Proposal Draft." The AI, using their company profile and the tender requirements, streams a coherent first draft directly into the split-pane editor.
    4.  **Drafting**: She assigns sections to her team members. They work in parallel. Elena locks the "Financials" section while she works on it, preventing conflicts. Comments and @mentions fly back and forth.
    5.  **Validation**: A week before the deadline, she runs the "Compliance Check." It flags two missing attachments and a section that exceeds the page limit. They fix the issues instantly.

*   **Climax**: Two days before the deadline, the final proposal is complete. Elena runs the "Score Simulator." The AI gives them an 85/100 and suggests strengthening the "Methodology" section with more specific examples. The team makes the changes, and the score jumps to 92/100. For the first time, Elena feels confident, not just hopeful.

*   **Resolution**: They submit the bid with a day to spare. Elena looks at her dashboard. They have three other proposals in progress, all on track. There's no chaos. No panic. She's not just a bid manager anymore; she's a bid strategist. She approves the monthly subscription. Two weeks later, they win the contract.

*   **Journey Requirements Summary**: This journey highlights the need for: AI Document Analysis (summary, checklist, risk flagging), AI Proposal Generation, a collaborative multi-user editor with versioning and locking, automated compliance validation, and scoring simulation. It validates the core value proposition of the Professional tier.

### Journey 2: Maria - The Cautious Solopreneur (Free Explorer - Evaluation Path)

*   **Persona**: Maria is a freelance environmental consultant. She has deep expertise but is a one-person shop. She knows there are EU grants and smaller public tenders she could win, but the world of public procurement is opaque and intimidating. She can't afford expensive consultants or complex software.

*   **Opening Scene**: Maria is browsing a government portal, trying to find opportunities. It's a frustrating experience with poor search functionality and an overwhelming amount of information. She finds a tender that looks promising but can't access all the documents without registering on yet another portal. She gives up, frustrated.

*   **Rising Action**: She hears about EU Solicit's free tier. She signs up with just her email.
    1.  **Exploration**: She immediately gets access to a clean, searchable list of tenders. She filters by "environment" and "Bulgaria."
    2.  **Limited View**: For each tender, she can see the name, deadline, and contracting authority. The budget, full documents, and AI analysis are locked. A tooltip says "Upgrade to Starter to see full details."
    3.  **The "Aha!" Moment**: She finds a tender for an "Environmental Impact Assessment" with a deadline in 3 weeks. The free view gives her just enough information to know it's a perfect fit for her skills.

*   **Climax**: Maria decides to take a small risk. She upgrades to the Starter tier for €29 for one month to work on this single opportunity. The moment she pays, all the fields unlock. She sees the budget is €25,000 - a great project for her. She uses the AI summary to quickly understand the core requirements.

*   **Resolution**: While the free tier didn't let her *do* the work, it allowed her to *find* the work and make an informed decision to upgrade. She successfully prepares and submits the bid using the Starter tier features. She realizes that for the cost of a few coffees, she has a tool that can find her next project. She keeps the subscription.

*   **Journey Requirements Summary**: This journey demonstrates the critical role of the freemium model. It requires a clear distinction between free and paid features, effective paywall messaging, and a seamless upgrade path. The free tier must provide enough value to prove the platform's potential without giving everything away.

### Journey 3: The Platform Admin (Internal User - Operations)

*   **Persona**: An internal operator at EU Solicit responsible for maintaining the platform's data quality and configuration.

*   **Opening Scene**: A new EU-wide regulation on data privacy in grant applications has just been announced. The platform's compliance frameworks need to be updated.

*   **Rising Action**:
    1.  **Access**: The admin logs into the secure, IP-restricted admin portal.
    2.  **Framework Management**: They navigate to the "Compliance Frameworks" section.
    3.  **Update**: They edit the "Horizon Europe" framework, adding three new rules related to the new regulation, complete with descriptions and severity levels.
    4.  **Crawler Monitoring**: While in the portal, they check the "Crawler Status" dashboard. They notice the Bulgarian AOP crawler has a higher-than-average error rate for the past 24 hours. They drill down into the logs and see the portal has changed its HTML structure slightly.
    5.  **Action**: They file a high-priority ticket for the engineering team to update the crawler's selectors.

*   **Climax**: A Professional tier user (Elena) starts a new proposal for a Horizon Europe grant. When she runs the compliance check, the new data privacy rules are immediately applied, flagging a section of her proposal that needs revision.

*   **Resolution**: The admin's quick action ensured the platform's compliance intelligence remained up-to-date, providing immediate value to customers and protecting them from potential non-compliance. The monitoring tools allowed for proactive maintenance of the data pipeline.

*   **Journey Requirements Summary**: This journey illustrates the need for a secure admin portal with dedicated tools for managing core platform data like compliance frameworks. It also shows the importance of operational dashboards for monitoring system health (e.g., crawlers) and enabling proactive maintenance.

### Journey 4: The Enterprise Developer (API Consumer)

*   **Persona**: A developer at a large consulting firm that has an Enterprise plan with API access. They want to integrate EU Solicit's opportunity data into their internal Salesforce CRM.

*   **Opening Scene**: The firm's partners want a custom "High-Priority Tenders" view in their Salesforce dashboard, fed by EU Solicit's AI-scored opportunities.

*   **Rising Action**:
    1.  **API Keys**: The developer is given an API key from their company's EU Solicit admin panel.
    2.  **Documentation**: They access the OpenAPI (Swagger) documentation for the Enterprise API.
    3.  **Endpoint Integration**: Using the documentation, they write a script that calls the `/opportunities` endpoint once an hour, filtering for opportunities with a score >= 80 and a status of "new".
    4.  **CRM Sync**: The script then pushes these high-priority leads into a custom object in their Salesforce instance.

*   **Climax**: A partner logs into Salesforce and sees a new tender, scored at 95, appear in their dashboard in near real-time. They click a link that takes them directly to that opportunity within the EU Solicit platform to begin the bid process.

*   **Resolution**: The developer has successfully integrated two systems, creating a seamless workflow for the partners and demonstrating the value of the Enterprise tier's API access.

*   **Journey Requirements Summary**: This journey validates the need for a well-documented, secure, and robust REST API for enterprise customers. It requires API key management, clear OpenAPI/Swagger documentation, and endpoints that support filtering and pagination for programmatic access to data.

## Functional Requirements

### User and Tenant Management

*   **FR-1:** A new user can register for an account using an email and password.
*   **FR-2:** A new user can register for an account using their Google identity.
*   **FR-3:** A registered user can log in and out of the system.
*   **FR-4:** A user must verify their email address before accessing core features.
*   **FR-5:** An admin user can create and manage their company profile.
*   **FR-6:** An admin user can invite other users to their company workspace.
*   **FR-7:** An admin user can assign and modify roles for users within their workspace (Admin, Bid Manager, Contributor, Reviewer, Read-Only).
*   **FR-8:** An enterprise admin can manage multiple, isolated client workspaces under a single parent company account.
*   **FR-9:** A user can belong to multiple company workspaces and switch between them.

### Billing & Subscription Management

*   **FR-10:** A user can subscribe to a paid tier (Starter, Professional, Enterprise) using a credit card via Stripe.
*   **FR-11:** The system automatically gates features based on the user's active subscription tier and usage limits.
*   **FR-12:** A new user is automatically enrolled in a 14-day free trial of the Professional tier.
*   **FR-13:** A user can manage their subscription (upgrade, downgrade, cancel) through a self-service customer portal (Stripe).
*   **FR-14:** The system can process one-time add-on purchases for specific premium features.

### Data Pipeline & Opportunity Discovery

*   **FR-15:** The system can automatically ingest new procurement opportunities from configured public sources (AOP, TED).
*   **FR-16:** A user can search for opportunities using full-text search and faceted filters (e.g., CPV code, region, budget).
*   **FR-17:** The system can calculate and display an AI-generated relevance score for each opportunity based on a company's profile.
*   **FR-18:** A user can view a list of opportunities, sorted by relevance or other criteria (e.g., deadline).
*   **FR-19:** A user can view the detailed information for a single opportunity.
*   **FR-20:** A user can save or "star" opportunities for future reference.

### AI-Powered Analysis & Insights

*   **FR-21:** The system can generate a one-page executive summary for a given tender document.
*   **FR-22:** The system can extract a structured list of mandatory requirements from a tender document to form a compliance checklist.
*   **FR-23:** The system can identify and flag potentially high-risk clauses within a tender document.
*   **FR-24:** A user can upload tender documents (PDF, DOCX) for the system to analyze.
*   **FR-25:** The system can simulate a potential score for a proposal based on the tender's evaluation criteria.

### Proposal Generation & Collaboration

*   **FR-26:** A user can create a new proposal associated with a specific opportunity.
*   **FR-27:** The system can generate an AI-assisted first draft of a proposal.
*   **FR-28:** A user can edit proposal content in a rich text editor.
*   **FR-29:** The system can maintain a version history of proposal drafts.
*   **FR-30:** A user can restore a previous version of a proposal.
*   **FR-31:** A user can lock a specific section of a proposal to prevent concurrent edits. (Post-MVP)
*   **FR-32:** Users can add and resolve comments on proposal sections. (Post-MVP)
*   **FR-33:** A user can export a final proposal to PDF and DOCX formats.

### Compliance & Workflow

*   **FR-34:** The system can validate a proposal against its auto-generated compliance checklist.
*   **FR-35:** A platform admin can create and manage regulatory compliance frameworks (e.g., ZOP).
*   **FR-36:** The system can generate a pre-filled European Single Procurement Document (ESPD) from a company's profile.
*   **FR-37:** A user can create and manage tasks associated with a proposal. (Post-MVP)
*   **FR-38:** A user can define dependencies between tasks. (Post-MVP)
*   **FR-39:** The system can enforce a multi-stage approval workflow for a proposal before submission. (Post-MVP)

### Notifications and Administration

*   **FR-40:** A user can configure and receive email digests for new, relevant opportunities.
*   **FR-41:** The system can sync opportunity deadlines with a user's external calendar (Google/Outlook). (Post-MVP)
*   **FR-42:** A platform admin can manage tenants, subscriptions, and system-wide settings through a secure admin portal.
*   **FR-43:** The system maintains an immutable audit trail of all significant user and system actions.
*   **FR-44:** An enterprise user can access platform data via a secure REST API.

## Non-Functional Requirements

### Performance

*   **NFR-1 (API Response Time):** All user-facing API endpoints must have a 95th percentile (p95) response time of less than 200ms.
*   **NFR-2 (AI Generation):** The Time to First Byte (TTFB) for streaming AI-generated content (e.g., summaries, proposal drafts) must be less than 500ms.
*   **NFR-3 (Page Load):** Core application pages must achieve a Lighthouse Performance score of 90 or higher.
*   **NFR-4 (Concurrency):** The system must support 100 concurrent active users during the MVP phase without performance degradation.

### Security

*   **NFR-5 (Authentication):** User authentication must be handled via JWTs with short-lived access tokens and securely stored refresh tokens.
*   **NFR-6 (Encryption):** All data must be encrypted in transit using TLS 1.3+ and at rest using AES-256. Sensitive data like external API keys must be additionally encrypted at the application level.
*   **NFR-7 (Data Isolation):** The architecture must enforce strict data isolation between tenants. No data access should be possible across workspace boundaries.
*   **NFR-8 (Admin Security):** Access to the admin portal must be restricted by IP allow-listing and require multi-factor authentication (MFA).
*   **NFR-9 (Dependencies):** The system must not have any known critical or high-severity vulnerabilities in its third-party dependencies. A regular process for scanning and updating dependencies must be in place.

### Scalability

*   **NFR-10 (Vertical Scalability):** The system's services must be designed to scale vertically (increasing CPU/RAM) to handle increased load.
*   **NFR-11 (Horizontal Scalability):** The application services and AI gateways must be stateless to allow for horizontal scaling by adding more instances.
*   **NFR-12 (Database Scalability):** The database architecture must support read replicas to handle increased read traffic as the user base grows.
*   **NFR-13 (Growth Target):** The system must be designed to scale to support 10,000 active companies and 1,000,000 total opportunities within 2 years with less than 20% performance degradation.

### Reliability

*   **NFR-14 (Uptime):** The platform must achieve >= 99.5% uptime, measured monthly.
*   **NFR-15 (Data Integrity):** The system must guarantee zero data loss for user-generated content (e.g., proposal drafts). All data must be backed up daily, with a point-in-time recovery capability of 24 hours.
*   **NFR-16 (Idempotency):** Critical financial operations, such as creating a new subscription, must be idempotent to prevent duplicate charges on retries.
*   **NFR-17 (Disaster Recovery):** The system must have a documented disaster recovery plan to restore service within 4 hours in case of a full regional outage.

### Accessibility

*   **NFR-18 (WCAG Compliance):** All user-facing interfaces must conform to Web Content Accessibility Guidelines (WCAG) 2.1 Level AA.
*   **NFR-19 (Keyboard Navigation):** The entire application must be fully navigable and operable using only a keyboard.
*   **NFR-20 (Screen Reader Support):** All content must be compatible with modern screen readers (e.g., JAWS, NVDA, VoiceOver), with appropriate ARIA attributes.

### Maintainability

*   **NFR-21 (Code Quality):** Backend code must pass `ruff` and `mypy` checks. Frontend code must pass `eslint` and `tsc` checks. No linting or type errors are allowed in the main branch.
*   **NFR-22 (Test Coverage):** A minimum of 80% unit test coverage and 90% E2E test coverage for critical user journeys must be maintained.
*   **NFR-23 (Logging & Monitoring):** All services must produce structured logs. Key system metrics (e.g., latency, error rates, resource usage) must be exported to a central monitoring system (Prometheus/Grafana).

## Domain-Specific Requirements (GovTech)

As a high-complexity GovTech platform, EU Solicit must adhere to stringent requirements beyond standard SaaS applications. The core of the domain revolves around trust, compliance, security, and accessibility when dealing with public sector data and processes.

### Compliance & Regulatory

*   **Public Procurement Law:** The platform must be fully compliant with the Bulgarian Public Procurement Act (ZOP) and relevant EU directives. This includes correct handling of procurement types, deadlines, and documentation formats like the ESPD.
*   **GDPR:** All personal data processing must be strictly compliant with the General Data Protection Regulation (GDPR). This includes data residency within the EU, the right to erasure, and maintaining detailed data processing records.
*   **Data Residency:** All customer data, without exception, must be stored and processed within EU data centers (specifically AWS eu-central-1 as per architecture).

### Technical Constraints

*   **Enhanced Security:**
    *   **Admin Access:** The platform's administrative functions must be inaccessible from the public internet, restricted via VPN and/or IP allowlists.
    *   **Audit Trail:** An immutable, append-only audit log must record all significant actions (mutations, access changes, downloads) for accountability and forensic analysis.
    *   **Data Encryption:** All data must be encrypted at rest (AES-256) and in transit (TLS 1.3). Sensitive credentials like OAuth tokens must have an additional layer of encryption.
*   **High Availability:** Given the time-sensitive nature of bid submissions, the platform must maintain >= 99.5% availability, with a robust disaster recovery plan.
*   **Data Isolation:** Strict logical separation between tenants (companies/workspaces) is mandatory. No query or process should ever cross tenant boundaries.

### Integration Requirements

*   **Government Portals:** The data pipeline must reliably integrate with official procurement portals like Bulgaria's AOP and the EU's TED. This requires robust crawlers that are actively maintained to handle changes in the source portals.
*   **Standardized Formats:** The platform must be able to generate documents in legally mandated formats, such as the XML-based ESPD (European Single Procurement Document).

### Accessibility

*   **WCAG 2.1 AA:** The entire user-facing application must conform to the Web Content Accessibility Guidelines (WCAG) 2.1 Level AA. This ensures the platform is usable by people with disabilities, a common requirement for government-facing software. This includes keyboard navigability, sufficient color contrast, and ARIA labels for all interactive components.

## Innovation & Novel Patterns

### Detected Innovation Areas

*   **Agentic AI Workflow Automation**: This is the core innovation. Instead of providing discrete tools (a parser, a writer), EU Solicit orchestrates a team of specialized AI agents (e.g., Document Parser, Summarizer, Draft Generator, Compliance Checker, Score Simulator) to manage the end-to-end bid lifecycle. This moves beyond simple "co-pilot" functionality to a proactive, automated project management paradigm.
*   **Democratization of Expertise**: The platform codifies the tacit knowledge of expensive bid consultants into a scalable SaaS model. By using AI to perform tasks that traditionally require years of human experience (e.g., identifying risky clauses, simulating evaluator scores), it makes high-level bid strategy accessible to smaller organizations.
*   **Unified Tender & Grant Environment**: While some tools exist for tender monitoring and others for grant writing, EU Solicit is novel in creating a single, unified platform that addresses both, recognizing the significant operational overlap for consulting firms and NGOs that pursue both funding types.

### Market Context & Competitive Landscape

The current market is fragmented:
1.  **Tender Databases**: Simple search portals that provide lists of opportunities but offer no analysis or workflow support.
2.  **General-Purpose AI Writers**: Tools like ChatGPT can draft text but lack the domain-specific context, compliance knowledge, and multi-step workflow orchestration required for a complete proposal.
3.  **Human Consultants**: Highly effective but expensive, unscalable, and inaccessible to the long tail of the market.

EU Solicit is creating a new category by combining the discovery of (1), the generative power of (2), and the expert knowledge of (3) into a single, affordable product.

### Validation Approach

The core innovative assumption is that an AI-driven workflow can produce proposals that are not just faster, but *better*—leading to higher win rates. This will be validated through a phased approach:

1.  **Demo Milestone**: Validate technical feasibility. Can the AI agents be orchestrated to produce a coherent, plausible proposal draft from a real tender document?
2.  **Beta Milestone**: Validate user acceptance and perceived value with a closed group of 10-20 companies. Key metrics: Time-to-draft, user-reported confidence, and willingness to pay.
3.  **MVP Launch**: Validate market effectiveness. Key metrics: User-reported win rate increase, reduction in cost-per-bid, and free-to-paid conversion rates.

### Risk Mitigation

*   **Innovation Risk**: The primary risk is that the AI-generated content is not of sufficient quality, leading to a lack of user trust and poor outcomes.
*   **Mitigation Strategy**: The platform is designed as a **human-in-the-loop system**, not a fully autonomous one. The AI *assists*, it does not *replace*. Every AI-generated output (summaries, drafts, compliance checks) is presented to the user for review, editing, and final approval. The user always has the final say, which mitigates the risk of AI error leading to a flawed submission. The goal is to make the human expert faster and more effective, not to remove them from the process.

## SaaS B2B Platform Specific Requirements

### Tenant & Subscription Model

*   **Multi-Tenancy:** The platform is architected as a multi-tenant system. Each "Company" or "Workspace" is a distinct tenant. All data is strictly isolated at the database level using a `workspace_id` or `company_id` on all relevant tables, with database policies enforcing this separation. There are no cross-tenant queries at the application layer.
*   **Subscription Tiers:** The business model is built on four subscription tiers, each with specific feature sets and usage limits.
    *   **Free:** Limited access for evaluation.
    *   **Starter:** Aimed at small businesses with access to a single country's tenders.
    *   **Professional:** Full-featured tier for core users, with collaboration and multi-country access.
    *   **Enterprise:** For large firms, offering white-labeling, API access, and advanced administrative features.
*   **Billing System:** Billing is handled via Stripe, supporting recurring subscriptions, one-time add-on purchases, and a 14-day free trial of the Professional tier. The system must also handle EU VAT correctly via Stripe Tax.

### Permissions & RBAC Model

*   **Role-Based Access Control (RBAC):** A robust RBAC matrix defines user permissions. Key roles include:
    *   **Admin:** Full control over the company's workspace, including user management and billing.
    *   **Bid Manager:** Can manage proposals and assign tasks.
    *   **Contributor:** Can edit assigned proposals.
    *   **Reviewer:** Can comment and approve/reject proposals.
    *   **Read-Only:** Can view opportunities and proposals but cannot make changes.
*   **Granular Permissions:** The permission model extends beyond global roles, allowing for per-entity access control. For example, a user can be a "Bid Manager" on one proposal but only a "Reviewer" on another, providing fine-grained control for sensitive projects.

### Integrations

*   **Data Ingestion:** Core integrations with public procurement portals (AOP, TED) are essential for the data pipeline. These are handled by scheduled crawlers.
*   **User Productivity:** Integrations with Google Calendar and Outlook Calendar for deadline syncing are provided for paid tiers.
*   **Notifications:** Real-time notifications for team collaboration can be sent to Slack and Microsoft Teams channels (Post-MVP).
*   **Enterprise API:** An enterprise-grade REST API is available for the highest tier, allowing customers to integrate EU Solicit data (like scored opportunities) into their own internal systems (e.g., CRM, BI tools).

### Compliance Considerations

*   **Auditability:** As a B2B platform dealing with critical business workflows, all significant actions must be logged in an immutable audit trail. This is crucial for security, compliance, and internal governance for enterprise customers.
*   **Admin & Governance:** The platform includes a separate, secure admin portal for internal EU Solicit operators to manage tenants, monitor system health, and oversee compliance frameworks. This ensures that customer-facing operations are separate from platform administration.
