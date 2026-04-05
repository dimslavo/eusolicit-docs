# EU Solicit — Architecture vs Requirements Gap Analysis

**Ref**: Requirements Brief v4 ↔ Solution Architecture v3.1 | **Date**: 2026-04-05 | **Status**: Resolved
**Resolution**: All 15 gaps closed in Solution Architecture v4.0 (2026-04-05).

---

## Summary

The Solution Architecture v3.1 covered the majority of the Requirements Brief v4. **15 gaps** were identified — 6 features with no architectural coverage, 7 partially covered areas needing more design detail, and 2 mismatches between the agent inventories. **All gaps have been resolved in Architecture v4.0.**

| Severity | Count | Status |
|----------|-------|--------|
| **Missing** | 6 | All resolved — ADRs 1.11–1.15, new domain modules, tables, agents added |
| **Underspecified** | 7 | All resolved — data models, middleware, billing flows fully specified |
| **Inventory mismatch** | 2 | Resolved — agent inventory reconciled (29 entries), crawlers confirmed as KraftData agents |

---

## 1. Missing — No Architectural Coverage

### GAP-01: Task Orchestration & Workflow Engine

**Requirement** (6.6): "Automated workflow from opportunity identification through final submission, with task assignments, dependencies, and deadline tracking."

**Architecture**: No task management tables in any schema. No task module in any service's module map. The UX Supplement defines a Task Board screen, but there is no backend for it.

**Impact**: This is a core workflow feature for Professional and Enterprise users managing multiple concurrent bids. Without task orchestration, the platform is a set of tools rather than a workflow system.

**Recommendation**: Add a `tasks` domain module to Client API with tables: `tasks`, `task_dependencies`, `task_templates`. Add a task assignment model linked to `users` and `opportunities`. Consider whether task templates should be opportunity-type-specific.

---

### GAP-02: Approval Gates

**Requirement** (6.6): "Configurable review and sign-off stages before submission (technical, legal, management)."

**Architecture**: No approval workflow tables, no approval module. The UX Supplement defines an Approval Pipeline screen with no backend counterpart.

**Impact**: Without approval gates, the platform can't enforce governance workflows that Enterprise/consulting clients need before proposal submission.

**Recommendation**: Add an `approvals` domain module to Client API with tables: `approval_workflows` (configurable stage definitions per company), `approval_stages`, `approval_decisions`. Link to `proposals` and `companies`. This is closely coupled with GAP-01 (task orchestration).

---

### GAP-03: Consortium Finder

**Requirement** (6.4): "Suggests potential partners from a database of organizations that have participated in similar grants."

**Architecture**: No Consortium Finder agent in the KraftData agent inventory (Section 11.2). No partner/organization data model. No storage resource for consortium data.

**Impact**: A differentiating feature for the EU Grant Specialization module. Without it, grant applicants must source consortium partners externally.

**Recommendation**: Add a Consortium Finder Agent to the KraftData inventory. Add a storage resource for partner organization data (sourced from historical EU grant award data). Consider whether the Client API needs a `partner_organizations` table or if this is purely a KraftData RAG search.

---

### GAP-04: Reporting Template Generator (Post-Award)

**Requirement** (6.4): "Post-award periodic report and financial statement templates pre-filled from project data."

**Architecture**: Not listed in agent inventory. No post-award data model (project tracking after a bid is won).

**Impact**: This is a retention feature — keeping clients engaged after they win a grant. Without it, the platform's value ends at bid submission.

**Recommendation**: Add a Reporting Template Generator Agent to the KraftData inventory. Decide whether post-award project management is MVP scope or can be deferred. If MVP, the `bid_outcomes` table needs expansion to track ongoing project data (milestones, reporting periods, financial draws).

---

### GAP-05: ESPD Auto-Fill

**Requirement** (6.7): "Pre-populates European Single Procurement Document from stored company data."

**Architecture**: No ESPD agent, no ESPD data model, no ESPD-related feature in any service module. The UX Supplement defines an ESPD Builder screen.

**Impact**: ESPD is a mandatory document for EU public procurement tenders above threshold values. This is a high-value automation for all paid tiers.

**Recommendation**: This can be implemented as either (a) a dedicated ESPD Agent on KraftData that takes company profile data and generates a pre-filled ESPD XML/PDF, or (b) a structured form in the Client API that maps company profile fields to ESPD schema fields. Option (b) is more reliable for a structured document. Add ESPD-related tables or a generation module to Client API.

---

### GAP-06: Guided Submission Workflows

**Requirement** (4.4): "Step-by-step guided workflows walking the user through the submission process for the specific opportunity type" and "Portal-specific submission instructions."

**Architecture**: No submission guidance model, no workflow engine for guided steps, no data structure for portal-specific instructions.

**Impact**: This is the core of the "self-service bidding" model — the platform's alternative to directly submitting bids. Without it, the user gets AI-generated documents but no structured guidance on how to actually submit them.

**Recommendation**: Add a `submission_guides` table to the pipeline or admin schema, populated per opportunity type/source. Link to `opportunities`. The Admin API should manage guide templates; the Client API should render them as a step-by-step checklist for users. This may also need a KraftData agent to auto-generate portal-specific instructions from crawled data.

---

## 2. Underspecified — Needs More Architectural Detail

### GAP-07: 14-Day Free Trial

**Requirement** (4.5): "14-day free trial of Professional tier for new registrations (no credit card required)."

**Architecture**: The subscription model mentions Stripe but doesn't address trial state management.

**What's missing**: Trial start/end date fields in the subscription model, trial-to-paid conversion flow, behavior when trial expires (graceful downgrade to Free tier), trial eligibility rules (one per company? per user?).

**Recommendation**: Extend `subscriptions` table with `trial_start`, `trial_end`, `trial_converted_at`. Add trial state to tier gating middleware. Define Stripe trial subscription flow (Stripe natively supports trials).

---

### GAP-08: Per-Bid Add-On Pricing

**Requirement** (4.1): Per-bid add-on with variable pricing for all paid tiers.

**Architecture**: Tier gating covers subscription tiers but doesn't model one-time or per-bid add-on purchases.

**What's missing**: Data model for add-on purchases, Stripe one-time charge integration (Checkout Sessions or Payment Intents), association of add-on credits to specific opportunities or proposals.

**Recommendation**: Add `add_on_purchases` table to the client schema. Decide whether add-ons are pre-purchased credit packs or per-bid charges. Extend the billing module to support Stripe one-time payments alongside subscriptions.

---

### GAP-09: EU VAT Handling

**Requirement** (4.5): "Stripe-generated invoices with EU VAT handling. Enterprise tier supports custom invoicing."

**Architecture**: Stripe is integrated for subscriptions but VAT configuration is not addressed.

**What's missing**: Tax ID collection at registration/upgrade, Stripe Tax or manual VAT rate configuration, reverse charge mechanics for B2B cross-border within EU, custom invoice generation for Enterprise tier.

**Recommendation**: Use Stripe Tax for automated EU VAT calculation. Add `tax_id` to the company profile. Configure Stripe customer tax settings. For Enterprise custom invoicing, define whether this is manual (admin-generated) or automated with custom terms.

---

### GAP-10: Proposal Collaboration Model

**Requirement** (6.3): "Multi-user editing with role-based access (bid manager, technical writer, financial analyst, legal reviewer)."

**Architecture**: Has `proposals` and `proposal_versions` tables. Has `role-based access` mentioned in platform features. No detail on how collaboration works.

**What's missing**: Per-proposal role assignments, concurrent editing strategy (lock-based or CRDT), comment/annotation model on proposal sections, review workflow (draft → review → approved), notification on assignment/mention.

**Recommendation**: Add `proposal_collaborators` table (user, proposal, role). Decide on editing model — for MVP, pessimistic locking (one editor at a time per section) is simpler than real-time co-editing. Add `proposal_comments` table. The Tiptap editor in the frontend supports collaboration via Y.js, but the backend needs a corresponding persistence and sync layer.

---

### GAP-11: Per-Tender RBAC Granularity

**Requirement** (6.10): "Admin, bid manager, contributor, reviewer, read-only — with per-tender permission granularity."

**Architecture**: Auth system issues JWT with roles. No detail on per-tender (per-opportunity or per-proposal) permission enforcement.

**What's missing**: Permission model that allows a user to be "bid manager" on Tender A but "read-only" on Tender B. Table structure for per-entity role assignments. Middleware that checks entity-level permissions, not just global role.

**Recommendation**: This overlaps with GAP-10. A unified `entity_permissions` table or a `proposal_collaborators` + `opportunity_access` pair. Middleware checks should cascade: global role → company role → entity-level permission.

---

### GAP-12: Reusable Content Blocks Library

**Requirement** (6.5): "Library of approved boilerplate sections (company overview, quality management, sustainability policy) stored in vector stores."

**Architecture**: Company Profiles Store and RFP Templates Store exist. No dedicated content blocks feature in any service module.

**What's missing**: A content blocks management CRUD in Client API (create, tag, version, approve boilerplate sections). A mechanism to feed these into the Proposal Generator Workflow via RAG.

**Recommendation**: Add a `content_blocks` domain sub-module within the proposals or companies module. Blocks are stored in the Company Profiles Store (already exists). Client API needs CRUD endpoints + search. The Proposal Generator Workflow's RAG context should include the company's content blocks.

---

### GAP-13: Bid History Analytics & Custom Reporting

**Requirement** (6.5): "Bid history analytics agent: Tracks win/loss rates, average scores, common feedback themes, and improvement trends. Generates periodic reports."

**Requirement** (6.8): "Custom reporting: Exportable reports for management — pipeline value, success rate trends, upcoming deadlines." Also ROI tracker and team performance metrics.

**Architecture**: Analytics module exists in Client API. `bid_outcomes` table captures outcomes. But no specific analytics computation model, no report generation pipeline, no data structures for preparation time/cost tracking (needed for ROI).

**What's missing**: How analytics are computed (real-time queries vs. materialized views vs. pre-computed), report export mechanism (same PDF/DOCX pipeline as proposals, or separate), data capture for preparation time and cost per bid (needed for ROI tracker and team performance).

**Recommendation**: Add `bid_preparation_logs` table (tracks time/cost per user per proposal). Use PostgreSQL materialized views for dashboard aggregations. Extend the Notification Service's `generate_reports` task to support scheduled analytics reports. The Bid History Analytics Agent on KraftData can generate narrative insights from the raw data.

---

## 3. Inventory Mismatches

### GAP-14: KraftData Agent Inventory Discrepancies

**Requirements Brief lists 22 entries** in Section 7 (Pre-Built Agent Inventory).
**Architecture lists 21 entries** in Section 11.2.

| In Requirements, NOT in Architecture | In Architecture, NOT in Requirements |
|--------------------------------------|--------------------------------------|
| AOP Crawler Agent | Win Theme Extractor Agent (#19) |
| TED Crawler Agent | Requirement Checklist Agent (#18) |
| EU Grant Portal Agent | — |
| Consortium Finder (implied) | — |
| Reporting Template Generator (implied) | — |

**Crawler agents**: The requirements describe crawlers as KraftData agents. The architecture implements them as local Celery tasks (`aop_crawler.py`, `ted_crawler.py`, `eu_grants_crawler.py`). This is a valid architectural decision (reducing KraftData dependency for infrastructure tasks), but the deviation should be explicitly documented.

**Win Theme Extractor & Requirement Checklist agents**: These appear in the architecture but not in the requirements. They may have been added during architecture design. If they're planned, the requirements should be updated to include them. If they were speculative, remove from architecture.

**Recommendation**: Reconcile the two inventories. Document the crawler implementation decision. Add Consortium Finder and Reporting Template Generator if they're MVP scope (see GAP-03, GAP-04). Update requirements to include Win Theme Extractor and Requirement Checklist agents if desired.

---

### GAP-15: Crawler Implementation Strategy Mismatch

**Requirements** (5.2, 6.1): Data pipeline agents are "orchestrated by Agentic AI Workflow Agents" on KraftData. Crawler agents are listed as KraftData agents.

**Architecture** (Service 3, Section 7.3): Crawlers are implemented as local Python modules (`aop_crawler.py`, `ted_crawler.py`, `eu_grants_crawler.py`) in the Data Pipeline service, executed as Celery tasks.

**Why this matters**: If crawlers are local Python code, the team owns the crawling logic, error handling, and portal-specific parsing. If they're KraftData agents, that complexity is pushed to the AI platform. The current architecture's approach is pragmatic (crawlers are deterministic HTTP scraping, not AI tasks), but this contradicts the requirements.

**Recommendation**: Decide and document. Likely keep crawlers as local Celery tasks (architecture's approach is correct for deterministic scraping). Update the requirements to reflect this — crawlers are local, normalization/enrichment uses KraftData agents. Remove crawler agents from requirements Section 7.

---

## 4. Alignment Confirmation

The following areas are well-covered with no gaps:

- **5-service SOA decomposition** — clean split, well-reasoned isolation
- **PostgreSQL schema isolation** with per-service DB roles
- **Redis Streams event bus** with clear event catalog
- **AI Gateway abstraction** — circuit breaking, SSE proxy, agent registry
- **Stripe billing** — checkout, webhooks, customer portal, usage metering
- **Calendar sync** — iCal, Google, Outlook with correct service split
- **Email alerts** — preference CRUD in Client API, delivery in Notification Service
- **Document upload** — S3 + ClamAV + retention policy
- **Compliance framework management** — admin CRUD + per-opportunity assignment + auto-suggestion
- **Audit trail** — shared schema, append-only, immutable
- **Authentication** — email/password + Google OAuth2 + JWT RS256
- **White-label** — subdomain + branding in Admin API
- **Tier gating + usage enforcement** — middleware + Redis counters
- **KraftData integration** — comprehensive REST + SSE + webhook pattern
- **Observability** — Prometheus, Grafana, Loki, Jaeger, structlog
- **Deployment** — K8s, Helm, GitHub Actions, environment strategy
- **Non-functional requirements** — availability, latency, GDPR, security all addressed

---

## 5. Prioritized Resolution Order

For implementation readiness, resolve gaps in this order:

| Priority | Gap | Rationale |
|----------|-----|-----------|
| **P1** | GAP-01 Task Orchestration | Core workflow backbone — many features depend on it |
| **P1** | GAP-02 Approval Gates | Tightly coupled with task orchestration |
| **P1** | GAP-06 Guided Submission Workflows | Core value proposition of the self-service model |
| **P1** | GAP-07 14-Day Free Trial | Blocks user onboarding flow design |
| **P1** | GAP-10 Proposal Collaboration | Multi-user editing is a key Professional/Enterprise feature |
| **P2** | GAP-11 Per-Tender RBAC | Security model must be designed before implementation |
| **P2** | GAP-08 Per-Bid Add-On Pricing | Affects billing module scope |
| **P2** | GAP-09 EU VAT Handling | Affects billing module scope |
| **P2** | GAP-12 Reusable Content Blocks | Relatively small addition to proposals module |
| **P2** | GAP-13 Analytics & Reporting | Drives dashboard implementation |
| **P2** | GAP-05 ESPD Auto-Fill | High-value feature for procurement tenders |
| **P2** | GAP-14 Agent Inventory Reconciliation | Documentation alignment |
| **P2** | GAP-15 Crawler Strategy Documentation | Documentation alignment |
| **P3** | GAP-03 Consortium Finder | Grant-specific, can be deferred |
| **P3** | GAP-04 Reporting Template Generator | Post-award, can be deferred |

---

*Analysis date: 2026-04-05 | Source documents: Requirements Brief v4 (2026-04-04), Solution Architecture v3.1 (2026-04-04)*
