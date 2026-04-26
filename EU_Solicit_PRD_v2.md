# EU Solicit — Product Requirements Document v2

**Platform**: EU Solicit | **Domain**: EUSolicit.com | **Date**: 2026-04-22 | **Status**: Draft

**Source Documents**:
- [Requirements Brief v5](./EU_Solicit_Requirements_Brief_v5.md) — Feature specifications, pricing model, MVP scope
- [Solution Architecture v5](./EU_Solicit_Solution_Architecture_v5.md) — Technical architecture, data models, service definitions
- [UX Supplement v1](./EU_Solicit_UX_Supplement_v1.md) — Design system, screen specifications, UX patterns
- [User Journeys v1](./EU_Solicit_User_Journeys_and_Workflows_v1.md) — Personas, user flows, E2E dataflows
- [Deployment & Branch Strategy v1](./EU_Solicit_Deployment_Branch_Strategy.md) — Monorepo, trunk-based dev, CI/CD

---

## 1. Product Summary

EU Solicit is a SaaS platform that automates the full lifecycle of public procurement and EU grant applications — from opportunity discovery through AI-guided proposal preparation. Powered by KraftData's Agentic AI platform, it targets Bulgarian and EU-wide opportunities as initial scope.

**Core value proposition**: Replace manual tender monitoring, expensive bid consultants, and fragmented tooling with a single AI-powered platform that discovers, analyzes, and helps prepare competitive bids.

**Business model**: Freemium SaaS with 4 tiers (Free → Starter → Professional → Enterprise) gated by region, industry, budget range, and AI feature limits. Per-bid add-ons for premium features. 14-day free trial of Professional tier.

---

## 2. Target Users & Personas

| Persona | Tier | Core Job-to-be-Done |
|---------|------|---------------------|
| **Free Explorer** (Maria) | Free | Evaluate platform value by browsing tenders with limited metadata |
| **Starter User** (Georgi) | Starter | Monitor Bulgarian tenders under €500K, use AI summaries to pre-qualify |
| **Professional User** (Elena) | Professional | Full proposal lifecycle across BG + 2 EU countries with team collaboration |
| **Enterprise User** (Nikolai) | Enterprise | White-label platform for multi-client bid management across all EU |
| **Platform Admin** (Internal) | Admin | Manage compliance frameworks, crawlers, tenants, audit |

Full persona definitions: [User Journeys v1, Section 1](./EU_Solicit_User_Journeys_and_Workflows_v1.md).

---

## 3. Feature Areas & Acceptance Criteria

### 3.1 Opportunity Discovery & Intelligence

Users discover relevant procurement and grant opportunities through automated crawling, smart matching, and configurable alerts.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Multi-source monitoring | AOP, TED, and EU grant portals crawled by local Celery tasks on schedule; new opportunities appear within 24h of publication | All |
| Opportunity listing | Searchable, filterable by CPV, region, budget, deadline, status; paginated results | All |
| Free-tier limited view | Free users see name, deadline, location, type, status only; all other fields gated | Free |
| Paid-tier full view | Paid users see full documentation, budget, evaluation criteria, AI analysis (within tier limits) | Paid |
| AI summary generation | One-page executive summary of tender documents generated via KraftData agent; streaming response | Starter+ |
| Smart matching | Relevance score (0-100) computed against company profile, displayed on listing and detail | Starter+ |
| Configurable email alerts | Users set CPV sectors, regions, budget range, deadline proximity; digest sent on schedule | Starter+ |
| Competitor tracking | Publicly available award data aggregated; competitor win rates and pricing displayed | Professional+ |
| Pipeline forecasting | Predicted upcoming tenders based on historical procurement cycles shown on dashboard | Professional+ |
| Document upload | Tender documents (PDF, DOCX, ZIP) uploaded to S3; virus scanned via ClamAV; max 100MB/file, 500MB/package | Starter+ |
| Submission guides | Portal-specific step-by-step submission instructions displayed on opportunity detail | Paid |

### 3.2 Document Analysis & Intelligence

AI agents analyze uploaded tender documents to extract structured data and flag risks.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Document parser | Extracts requirements, evaluation criteria, deadlines, mandatory documents, financial thresholds from tender packages | Starter+ |
| Requirement checklist | Auto-generated compliance checklist mapping every mandatory requirement to a trackable item | Professional+ |
| Clause risk flagging | High-risk clauses (unlimited liability, IP assignment, penalties) identified and flagged for legal review | Professional+ |
| Multi-language processing | Source documents processed in BG, EN, DE, FR, RO; outputs generated in user's preferred language | All |

### 3.3 Proposal Generation & Optimization

AI-powered proposal drafting with scoring simulation and compliance validation.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| AI draft generator | Multi-agent workflow generates first-draft proposal from tender requirements + company profile; SSE streaming | Professional+ (or add-on) |
| Template library | Sector-specific proposal templates available via RAG in the Proposal Generator Workflow | Professional+ |
| Scoring simulator | Predicted evaluator scores per criterion with improvement suggestions; scorecard UI | Professional+ |
| Compliance validator | Pre-submission check: mandatory sections, page limits, required attachments, formatting rules | Professional+ (or add-on) |
| Pricing assistant | Competitive pricing suggestion based on historical award data and budget ceilings | Professional+ |
| Proposal editor | Rich text editor (Tiptap) with versioning; section-level editing | Professional+ |
| Version control | Proposal versions tracked with diff capability; rollback to previous versions | Professional+ |
| Document export | Proposals, checklists, compliance reports exportable as PDF and DOCX | Professional+ |
| Content blocks library | Reusable boilerplate sections (company overview, quality management, etc.) stored, tagged, versioned | Professional+ |
| Win theme extractor | AI identifies key win themes from tender requirements for proposal positioning | Professional+ |

### 3.4 Proposal Collaboration & Workflow

Multi-user editing, task management, and governance workflows for bid preparation.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Per-proposal roles | Users assigned roles (bid manager, technical writer, financial analyst, legal reviewer, read-only) per proposal | Professional+ |
| Entity-level RBAC | Access controlled per opportunity/proposal; company role sets ceiling, entity permission narrows | Professional+ |
| Section locking | Pessimistic locking: one editor per section; 15-min TTL; locked sections show editor name | Professional+ |
| Proposal comments | Threaded comments anchored to section + version; resolvable; carry forward across versions | Professional+ |
| Task orchestration | Tasks with assignments, deadlines, dependencies (DAG); templates per opportunity type; overdue alerts | Professional+ |
| Approval workflows | Configurable stage sequences (technical → legal → management); sign-off decisions logged | Enterprise |
| Bid/no-bid decision | Structured scoring (strategic fit, win probability, resources, margin); AI recommendation with override | Professional+ |
| Bid outcome tracking | Record won/lost/withdrawn with optional evaluator feedback and scores | Professional+ |
| Preparation time logging | Users log hours and costs per proposal for ROI tracking | Professional+ |

### 3.5 EU Grant Specialization

Dedicated tools for EU programme-specific applications.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Grant eligibility matcher | Company/municipality profile mapped against active EU programmes (Horizon Europe, Digital Europe, structural funds, etc.) | Professional+ |
| Budget builder | AI-assisted budget following EU cost category rules, overhead calculations, co-financing requirements | Professional+ |
| Consortium finder | Partner suggestions from database of organizations that have participated in similar grants | Professional+ |
| Logframe generator | Auto-generated logical frameworks, work packages, Gantt charts, deliverable tables from narrative | Professional+ |
| Reporting template generator | Post-award periodic report and financial statement templates pre-filled from project data | Enterprise |

### 3.6 Compliance & Regulatory Intelligence

Admin-configurable compliance frameworks with AI-powered validation.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Compliance framework CRUD | Platform admins create/edit compliance frameworks (Bulgarian ZOP, EU directives, programme rules) | Admin |
| Per-opportunity assignment | Admin assigns applicable framework to each opportunity; hybrid scenarios supported | Admin |
| Framework auto-suggestion | Platform suggests framework based on country, source, funding type; admin confirms/overrides | Admin |
| ESPD auto-fill | Pre-populated European Single Procurement Document from stored company data; structured form | Professional+ |
| Regulation tracker | AI agent monitors changes to ZOP, EU procurement directives, and grant programme rules | Admin |
| Audit trail | Immutable log of all platform actions with timestamp, user, action type, entity ref, before/after values | All |

### 3.7 Notifications, Alerts & Calendar

Outbound communications and deadline management.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Email alerts | Configurable digest (immediate/daily/weekly) matching user's alert preferences; sent via SendGrid | Starter+ |
| iCal feed | Subscribable URL with deadlines for tracked opportunities; works with any calendar app | All |
| Google Calendar sync | Two-way sync via Google Calendar API; creates/updates/deletes events for deadlines | Professional+ |
| Outlook sync | Two-way sync via Microsoft Graph API; same functionality as Google Calendar | Professional+ |
| Task notifications | Email alerts on task assignment, approaching deadlines, and overdue tasks | Professional+ |
| Approval notifications | Email alerts when approval is requested and when decisions are made | Enterprise |
| Trial expiry reminder | Email 3 days before trial ends with upgrade prompt | Trial users |

### 3.8 Analytics & Reporting

Dashboards and exportable reports for decision-making.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Market intelligence | Sector-level analytics: procurement volumes, avg contract values, active authorities, seasonal trends | Professional+ |
| ROI tracker | Return on bid investment: preparation time/cost vs. contract value won | Professional+ |
| Team performance | Per-user metrics: bids submitted, win rate, average preparation time | Professional+ |
| Competitor intelligence | Aggregated competitor bidding patterns, win rates, pricing benchmarks | Professional+ |
| Pipeline forecast | Predicted upcoming opportunities based on historical patterns | Professional+ |
| Usage dashboard | Current period usage vs. tier limits (AI summaries, proposal drafts, compliance checks) | All paid |
| Report export | Management reports (pipeline value, success rates, deadlines) as PDF/DOCX; scheduled or on-demand | Professional+ |

### 3.9 Subscription & Billing

Stripe-powered billing with tiered access, trials, and add-ons.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Tier definitions | Free, Starter (€29), Professional (€59), Enterprise (€109+) with correct feature/limit mappings | All |
| Stripe Checkout | Upgrade flow via Stripe Checkout; subscription created with correct tier | All |
| Customer portal | Self-service upgrade, downgrade, cancel, update payment method via Stripe Customer Portal | Paid |
| 14-day trial | New registrations get Professional tier for 14 days, no credit card; graceful downgrade to Free on expiry | New users |
| Usage metering | AI summaries, proposal drafts, compliance checks tracked per period; blocked on limit with upgrade prompt | Paid |
| Per-bid add-ons | One-time purchases for premium AI features on specific opportunities; via Stripe Checkout | Paid |
| EU VAT | Automatic VAT calculation via Stripe Tax; tax ID collection; reverse charge for B2B | All paid |
| Enterprise invoicing | Custom invoicing with NET 30/60 payment terms for Enterprise tier | Enterprise |

### 3.10 Platform Administration

Internal tools for platform operations.

| Feature | Acceptance Criteria | Tier |
|---------|--------------------|------|
| Admin authentication | Separate admin auth with `platform_admin` role; VPN/IP-restricted access | Admin |
| Tenant management | View/manage companies, subscriptions, usage | Admin |
| Crawler management | View crawler run history, configure schedules, trigger manual runs | Admin |
| White-label configuration | Custom subdomain, logo, brand colors, email sender domain for Enterprise clients | Admin |
| Audit log viewer | Searchable, filterable audit log with export capability | Admin |
| Platform analytics | Signup funnel, tier conversion, usage patterns, churn metrics | Admin |
| Enterprise API | RESTful API documented via auto-generated OpenAPI spec for Enterprise tier clients | Enterprise |

---

## 4. Non-Functional Requirements

| Requirement | Target | Measurement |
|-------------|--------|-------------|
| Availability | 99.5% uptime | Grafana uptime monitoring |
| API latency (p95) | < 200ms for REST, < 500ms TTFB for SSE | Prometheus histograms |
| Data residency | All data within EU (AWS eu-central-1) | Infrastructure audit |
| Security | JWT RS256, TLS 1.3, AES-256 at rest, ClamAV | Security audit checklist |
| Scalability | 10K+ active tenders, concurrent agent execution per tenant | Load test |
| GDPR compliance | Right to erasure, DPAs with KraftData, data processing records | Compliance checklist |
| Agent quality | KraftData eval-runs for continuous quality monitoring | Eval score dashboards |
| i18n | Bulgarian + English in MVP; AI outputs in user's language | Manual QA |

---

## 5. Success Metrics

### 5.1 Demo Milestone Metrics

| Metric | Target | Notes |
|--------|--------|-------|
| Opportunities loaded | 500+ from AOP + TED | Proves data pipeline works |
| Search-to-detail flow | < 3 clicks | Proves UX works |
| AI summary generation | < 30s for 100-page document | Proves AI integration works |
| Proposal draft generation | < 2 min for first draft | Proves core value proposition |
| End-to-end flow | Registration → opportunity → proposal in one session | Demoable to investors |

### 5.2 MVP Launch Metrics

| Metric | Target | Timeframe |
|--------|--------|-----------|
| Registered companies | 100 | First 30 days |
| Free → paid conversion | 10% | First 60 days |
| Trial → paid conversion | 25% | Ongoing |
| Active paid subscribers | 25 | First 90 days |
| Proposals generated | 200 | First 90 days |
| NPS score | > 40 | First 90 days |

---

## 6. Release Milestones

### 6.1 Demo Milestone (Target: Sprint 8 / Week 16)

**Goal**: Functional end-to-end flow demoable to investors, partners, and early adopters.

**Scope**: Infrastructure + Auth + AI Gateway + Data Pipeline + Opportunity Discovery + Proposal Generation + Frontend. Billing is simulated (tier set manually in DB).

**Demo script**:
1. Register account, complete company profile
2. Browse opportunity listing (pre-loaded from AOP/TED crawls)
3. Search and filter by CPV sector, region, budget
4. Open opportunity detail, view AI-generated summary
5. Upload tender documents, view parsed requirements checklist
6. Generate AI proposal draft (streaming), edit in Tiptap editor
7. Run compliance check, view scorecard
8. Export proposal as PDF

### 6.2 Beta Milestone (Target: Sprint 12 / Week 24)

**Goal**: Production-ready for closed beta with 10-20 invited companies.

**Additional scope**: Stripe billing (live), trial flow, usage metering, email alerts, calendar sync, basic task management, ESPD auto-fill.

### 6.3 MVP Launch (Target: Sprint 14 / Week 28)

**Goal**: Public launch at EUSolicit.com.

**Additional scope**: Collaboration, approval workflows, EU grant specialization, analytics dashboards, admin platform, white-label, Enterprise API.

---

## 7. Out of Scope (Post-MVP)

- Electronic bid submission on behalf of clients
- Microsoft OAuth2 social login
- E-procurement portal connectors (CAIS, TED eSender)
- ERP integrations (SAP, 1C)
- CRM integrations (Salesforce, HubSpot, Pipedrive)
- Slack / Microsoft Teams notification channels
- Digital signatures / QES (eIDAS)
- Consultant marketplace
- Mobile native apps
- Elasticsearch advanced faceted search

---

## 8. Technical Architecture Reference

The full technical architecture is defined in [Solution Architecture v4](./EU_Solicit_Solution_Architecture_v4.md). Key decisions:

- **Backend**: Python 3.12 + FastAPI (5 services)
- **Frontend**: Next.js 14 + React 18 + Tailwind + shadcn/ui
- **Database**: PostgreSQL 16 (6 schemas, shared instance)
- **AI**: KraftData Agentic AI platform (29 agents/teams/workflows, 7 vector stores)
- **Billing**: Stripe (subscriptions, trials, add-ons, VAT)
- **Deployment**: Docker + Kubernetes + Helm + GitHub Actions
- **Monitoring**: Prometheus + Grafana + Loki + Jaeger

---

*Document version: 1.0 | Date: 2026-04-05 | Status: Draft*
