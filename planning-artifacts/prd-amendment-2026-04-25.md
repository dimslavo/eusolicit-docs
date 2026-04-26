---
documentType: 'prd-amendment'
sourceResearch: 'eusolicit-docs/planning-artifacts/research/market-eu-govwin-tender-intelligence-research-2026-04-25.md'
basePRD: 'eusolicit-docs/planning-artifacts/PRD.md'
basePRDversion: '1.0 (2026-04-25)'
amendmentVersion: '1.0'
date: '2026-04-25'
author: 'Mary (Business Analyst), facilitated by Deb'
status: 'Draft — pending architect evaluation'
nextOwner: 'Winston (Architect)'
nextSkills: ['bmad-agent-architect', 'bmad-correct-course', 'bmad-create-epics-and-stories']
---

# PRD Amendment 2026-04-25 — "Tenders + Grants for EU Consulting Firms"

## Purpose of This Document

This amendment scopes **six PRD-level changes** and **three GTM-level changes** into the canonical PRD (`eusolicit-docs/planning-artifacts/PRD.md` v1.0). It is the bridge between the market-research findings (see `sourceResearch` above) and the architecture/epic-creation workflow.

It is structured so that:
1. **Winston (Architect)** can evaluate technical feasibility section-by-section.
2. **The PM (or Mary continuing as facilitator)** can fold approved sections into the canonical PRD verbatim.
3. **The epic/story creation skill** (`bmad-create-epics-and-stories`) can read this directly to draft the new epics and inject them into the sprint plan.

**Important:** This document does NOT duplicate the full market research. For *why* each change is recommended, refer to the research artefact. This amendment focuses on *what* to change in the PRD and *what* the architect needs to decide.

---

## Strategic Reframing (Top-Line)

### Repositioning Statement

The platform's headline positioning is changing from a generic "EU procurement automation platform" to a **dual-segment, consulting-firm-first** product:

> *"EU Solicit is the only EU-native platform that helps consulting firms win both public tenders and EU grants — combining multilingual opportunity intelligence, AI-driven proposal generation, and EU-grade compliance into one workspace per client."*

**Driver of change:** Competitive analysis (research §Step 5) confirmed Altura (NL) raised €8M Series A in June 2025, achieved ISO 27001:2022 certification, acquired Berlin-based Tendara in July 2025, and is actively expanding into the German €500B procurement market with the same "EU's GovWin" positioning. The dual-segment (tenders + grants) angle is the only structurally unique whitespace remaining.

### Affected PRD Sections

| PRD section | Change | Effort |
|---|---|---|
| §1 Executive Summary & Vision | Rewrite headline + value prop | Editorial |
| §2 Problem Statement & Value Proposition | Expand to dual-segment framing | Editorial |
| §3 Target Audience & Personas | Reorder: consulting firms first; refine consulting-firm description | Editorial |
| §4 Product Scope (MVP) | Add multi-client workspace, per-bid SKU, CRM integrations to in-scope; revise pricing model | Material |
| §6 Functional Requirements | Add FR8 (Multi-Client Workspace), FR9 (Outcome Telemetry), FR10 (Trust Center & Compliance Posture), FR11 (CRM/Comms Integrations) | Material |
| §7 Non-Functional Requirements | Tighten uptime to 99.9%; commit ISO 27001 timeline | Material |
| §8 User Stories | Add stories for new functional requirements | Material |
| §9 Next Steps | Update sequencing to reflect amended scope | Editorial |

---

## Change 1 — Multi-Client Workspace Data Model

### Why
Consulting firms (the primary ICP) manage 10–50 client engagements concurrently. The current data model is org-scoped: a `company` is a single tenant. This is a fundamental mismatch with the consulting-firm workflow. Loopio and Responsive both have explicit per-client workspace models. Without this, EU Solicit caps out at the SME-bidder market and cannot credibly claim consulting firms as the primary ICP.

### What changes in PRD

**§3 Target Audience — Replace consulting firm description:**

> **Consulting Firms (primary ICP):** Engineering, IT, management, and EU-grant specialist consultancies managing 10–50 client engagements concurrently. Buys for: opportunity discovery + qualification, proposal generation, multi-client workspace separation, white-label client deliverables, CRM integration, outcome dashboards for client reporting. Highest WTP segment per RB v5 §3. Bid Director is economic buyer; Bid Manager is daily user.

**§4 Product Scope (MVP) — Add to In Scope:**

> - Multi-client workspace data model with per-client isolation, RBAC, and audit-trail scoping. Each consulting-firm tenant can create N client workspaces; each workspace is its own scope for opportunities, proposals, content blocks, audit log entries, and compliance frameworks.

**§6 Functional Requirements — Add new FR8:**

> ### FR8: Multi-Client Workspace Management (Consulting-Firm ICP)
> - **FR8.1:** A consulting-firm tenant must be able to create, rename, archive, and delete client workspaces (subject to retention policy).
> - **FR8.2:** Each client workspace must scope: opportunities, proposals, uploaded tender documents, generated content, AI-summary usage metering, audit-log entries, compliance framework assignments, and outcome telemetry.
> - **FR8.3:** Users at the tenant level must have RBAC roles per workspace (admin, bid_manager, contributor, reviewer, read_only — matching company-level roles per RB v5 §6.10).
> - **FR8.4:** Cross-workspace search, content reuse, and analytics must be available to tenant-level admin users.
> - **FR8.5:** A workspace must support an "external collaborator" role to invite the end-client to review drafts (read-only or comment-only) without consuming a paid seat.
> - **FR8.6:** Per-workspace tier metering must roll up to the tenant subscription (one Stripe subscription, N workspaces, shared metered usage with optional per-workspace soft caps).

### Architectural Questions for Winston

1. **Schema model:** Does `client_workspace` become a new first-class entity above `company`, or does `company` get a `parent_company_id` (tenant) and `is_workspace` flag? **Recommended evaluation criterion:** ease of migrating existing single-tenant accounts to multi-workspace tenants without data loss, and minimal RBAC code churn.
2. **Database schema isolation:** Currently each service owns one PostgreSQL schema. Multi-workspace tenancy is row-level, not schema-level. Confirm RLS approach is viable and audit-able under existing patterns.
3. **RBAC extension:** Current RBAC is two-tier (company role × entity permission). Adding a workspace tier makes it three-tier. Confirm `check_entity_access()` factory in `client_api/core/rbac.py` can extend to workspace scope without a rewrite.
4. **Audit trail scoping:** `shared.audit_log` per RB v5 §6.7 must record `workspace_id` for every mutation. Confirm migration path.
5. **External collaborator authentication:** Magic-link + scoped JWT, or full Google/email signup with limited role? **Recommended:** magic-link with 30-day expiry, no password, no Stripe seat consumption.

### Implied Epic Preview
- **Epic 14: Multi-Client Workspace Data Model & RBAC Extension** — schema migration, workspace CRUD, RBAC extension, scoped audit trail, external-collaborator auth, cross-workspace admin search.

---

## Change 2 — Per-Bid Pricing SKU

### Why
Consulting firms bill clients per pursuit (€3–8K/bid). Per-seat subscription forces them to absorb cost; per-bid pricing lets them pass through. Per-bid pricing is a market-novel SKU among profiled competitors (research §Step 5 pricing intelligence). It is the unlock for the Segment A beachhead (BG specialist tender consultancies).

### What changes in PRD

**§4 Product Scope (MVP) — Update tier description:**

> - Tiered access model: Free, Starter, Professional, **Pro+ (new tier — for consulting firms with multi-client workspace, CRM integrations, ISO 27001 posture, dedicated CSM)**, Enterprise. **Plus a Per-Bid SKU available as add-on or standalone, priced per opportunity tracked through end-to-end pursuit (€99–€299/bid depending on tier and bid value).**

**§6 Functional Requirements — Add to FR2:**

> - **FR2.4:** The system must support a Per-Bid pricing SKU. A "bid" is defined as an opportunity that the user has formally engaged with (bid/no-bid: bid; or proposal draft generated). Per-bid SKUs unlock per-opportunity all features that would otherwise require a paid tier (AI summaries, drafts, compliance checks, calendar sync). Per-bid SKU is purchasable via Stripe one-time payment or as a metered add-on to any subscription tier.
> - **FR2.5:** Per-bid SKU must integrate with usage metering: when a user activates a per-bid SKU on opportunity X, all metered-feature consumption on opportunity X bypasses tier-level monthly caps for the lifetime of that opportunity.
> - **FR2.6:** Per-bid SKU pricing must be configurable by admin (default tiers: €99 standard / €199 with grant module / €299 with full Enterprise feature stack).

### Architectural Questions for Winston

1. **Stripe integration:** Per-bid SKU is one-time payment (not recurring) but ties to a recurring subscription for billing-period reconciliation. Confirm Stripe's `one-time` + `subscription` combined model works without webhook flow rewrite.
2. **Metering bypass:** When a per-bid SKU is purchased, the metering layer must check both subscription tier AND per-bid activation per opportunity. Confirm Redis Lua script (`_USAGE_LUA`) can extend without contention or redesign.
3. **Refund policy:** What happens if the bid is abandoned? Recommend **non-refundable** to keep simple; flag for product/legal review.
4. **Identification of "bid":** Trigger event (e.g., bid/no-bid agent invocation) should atomically activate the SKU. Confirm event-driven activation is consistent with current `EventBus` patterns.

### Implied Epic Preview
- **Epic 15: Per-Bid Pricing SKU + Pro+ Tier Launch** — Stripe one-time-payment integration, metering bypass logic, per-bid activation events, admin-configurable pricing, Pro+ tier feature flagging.

---

## Change 3 — CRM, Slack, Teams Integrations

### Why
Consulting-firm bid pipelines live in CRM (HubSpot, Salesforce, Pipedrive) and team-comm (Slack, Teams). Without integration, EU Solicit becomes shelfware. CRM integration is a Tier-1 deal blocker for mid-large consultancies (research §Step 4 decision factors). Loopio's #1 sales talking point with consultancies is Salesforce sync.

### What changes in PRD

**§4 Product Scope (MVP) — Move from Out-of-Scope to In-Scope:**

> - **CRM integrations (HubSpot, Salesforce, Pipedrive — read-write sync of opportunities, win/loss outcomes, contact records).**
> - **Slack and Microsoft Teams notification integrations (deal alerts, deadline reminders, bid-team comments forwarded to channels).**

**§6 Functional Requirements — Add new FR11:**

> ### FR11: CRM and Communications Integrations
> - **FR11.1:** The system must support OAuth2-based integration with HubSpot, Salesforce, and Pipedrive. Each integration must be configurable per workspace.
> - **FR11.2:** When an opportunity is added to a workspace, the system must optionally create a corresponding CRM record (Deal/Opportunity). When the opportunity status changes (qualified, bid, submitted, won, lost), the CRM record must be updated.
> - **FR11.3:** When a CRM Deal/Opportunity is updated externally, the system must reflect the change in EU Solicit (read-back sync) within 5 minutes.
> - **FR11.4:** The system must support Slack and Microsoft Teams webhook integrations for: new opportunity alerts matching saved filters, bid deadlines (24h, 48h, 7d ahead), proposal generation completion, compliance-check failures.
> - **FR11.5:** All integrations must be tier-gated: Slack/Teams available from Professional tier; CRM available from Pro+ tier.

### Architectural Questions for Winston

1. **Service location:** New `integrations-api` service, or extend `client-api`? Recommend new service for separation of concerns and isolated rate-limit behaviour, but defer to architect.
2. **Sync model:** Webhook-driven (CRM → us) + polled (us → CRM) or full bi-directional event-stream? Token storage uses Fernet encryption (per project-context.md §Patterns Epic 9).
3. **Rate limit handling:** HubSpot/Salesforce/Pipedrive each have different rate limit characteristics. Confirm circuit-breaker pattern (project-context.md Rule 47) applies cleanly.
4. **Conflict resolution:** When same Deal is edited in CRM and EU Solicit simultaneously, who wins? Recommend **last-write-wins with audit trail**; flag for architect.
5. **Slack/Teams:** Use official SDKs or webhooks? Webhooks simpler, no OAuth; SDKs richer interactivity. Recommend **start with incoming webhooks** (M0), add interactive bots as Phase 2.

### Implied Epic Preview
- **Epic 16: CRM Integrations (HubSpot, Salesforce, Pipedrive)** — OAuth2 token storage, bi-directional sync engine, conflict resolution, webhook handlers.
- **Epic 17: Slack & Teams Notifications** — incoming-webhook configuration UI, alert routing, message templating, channel mapping.

---

## Change 4 — ISO 27001 Roadmap & Trust Center

### Why
ISO 27001 is the **preferred certification in EU procurement gates** (research §Step 4 — confirmed via HST Solutions analysis: EU government contracts and GDPR-conscious enterprises prefer ISO 27001 over SOC 2). Altura already has ISO/IEC 27001:2022. Without a public commitment with target date in a Trust Center page, mid-large EU consulting-firm deals stall 4–8 weeks at the procurement gate.

### What changes in PRD

**§7 Non-Functional Requirements — Add commitment line:**

> - **Compliance Certifications:** GDPR-compliant from day 1 (per RB v5 §9). **ISO/IEC 27001:2022 audit must commence by Month 6 of platform launch with target certification by Month 12.** A public Trust Center page must be live by Month 2 of platform launch with downloadable: GDPR DPA, security overview, ISO 27001 roadmap with quarterly milestones, sub-processor list, current pen-test summary, data-residency confirmation.

**§4 Product Scope (MVP) — Add to In Scope:**

> - **Trust Center page** with publicly downloadable security and compliance artefacts (DPA, security overview, ISO 27001 roadmap with milestones, sub-processor list, pen-test summary, data-residency confirmation).

**§6 Functional Requirements — Add new FR10:**

> ### FR10: Trust Center & Compliance Posture
> - **FR10.1:** A publicly accessible `/trust` page must list current compliance posture: GDPR (compliant), ISO 27001 (in progress with target date), SOC 2 (planned/N-A), data residency (EU only), encryption at rest (AES-256) and in transit (TLS 1.3).
> - **FR10.2:** The Trust Center must provide downloadable PDF artefacts: GDPR Data Processing Agreement (DPA), Security Overview, Sub-Processor List, latest pen-test summary (redacted), Business Continuity Plan summary, Data Residency confirmation.
> - **FR10.3:** Sub-Processor List must auto-update from a structured source (e.g., `infra/sub-processors.yaml`) so changes are tracked in version control with effective-date entries.
> - **FR10.4:** All Trust Center artefacts must be versioned with change-log entries visible to subscribers (and via email to active customer DPAs when sub-processors change, per GDPR Art. 28).

### Architectural Questions for Winston

1. **Content management:** Static rendering at build time, or DB-backed CMS for the Trust Center? Recommend **static rendering from version-controlled YAML/MDX** for maximum auditability.
2. **PDF generation:** Reuse existing WeasyPrint pipeline (per project-context.md Epic 7 patterns) or accept human-curated PDFs? Recommend **human-curated for legal artefacts (DPA), generated for living artefacts (sub-processor list, security overview)**.
3. **Notification of sub-processor changes:** Email distribution to all active DPAs is a GDPR requirement. Confirm `notification` service can handle this without a dedicated transactional flow.
4. **ISO 27001 audit scope:** Scope must include all services + infra + KraftData integration + customer data handling. Confirm with auditor early; do NOT under-scope and re-audit.

### Implied Epic Preview
- **Epic 18: Trust Center Page & Compliance Posture** — `/trust` route, downloadable artefact pipeline, sub-processor YAML + auto-render, customer notification on changes.
- **Cross-cutting epic / programme:** ISO 27001 audit preparation — controls inventory, evidence collection, gap analysis, auditor engagement. Likely a sprint-spanning programme rather than a single epic.

---

## Change 5 — Uptime SLA Tightened to 99.9%

### Why
Loopio and Responsive offer 99.9%; Mercell offers 99.9%+. Procurement gates flag 99.5% as below market norm. Tightening to 99.9% is a procurement-stake change with limited engineering impact (mostly observability and incident-response discipline).

### What changes in PRD

**§7 Non-Functional Requirements — Update Availability:**

> - **Availability:** **99.9% uptime SLA for the client platform** (revised from 99.5% v1.0). Measurement window: rolling 30 days. Excluded: maintenance windows announced ≥48h in advance, force-majeure (per Stripe-equivalent SLA terms). Service-credit policy applies on breach: 10% credit for ≥99.0% but <99.9%; 25% credit for ≥95.0% but <99.0%; 50% credit for <95.0%.

### Architectural Questions for Winston

1. **Current platform availability under existing SLO:** What does k6/Prometheus baseline say? (Note: k6 baseline is a known anti-pattern carry-forward per project-context.md Epic 8 — this is now newly urgent.)
2. **Single points of failure:** PostgreSQL (one instance), Redis (one instance), individual services without HPA/PDB. Confirm what 99.9% requires in infra changes.
3. **Incident response:** PagerDuty/Opsgenie commitment, runbook density, on-call rotation. 99.9% = ~43min downtime/month max.

### Implied Epic Preview
- **No new epic** — but cross-cutting refinement to existing observability/reliability work and the carry-forward k6 baseline story.

---

## Change 6 — Outcome Telemetry & Renewal Proof Memo

### Why
Renewal conversations require quantified outcome (€/hours saved, win-rate uplift, content reuse). Without instrumentation, retention is anecdotal. **"You'll be surprised how many customers churn"** in this category (Loopio commentary, research §Step 4). Outcome telemetry is the renewal engine.

### What changes in PRD

**§6 Functional Requirements — Add new FR9:**

> ### FR9: Outcome Telemetry & Renewal Proof
> - **FR9.1:** Every opportunity must support a "platform-attributed" tag: was the opportunity discovered via platform feed (vs. external)? This tag persists through bid lifecycle and is queryable for analytics.
> - **FR9.2:** Bid outcomes (won, lost, withdrawn) must be capturable per opportunity, with optional evaluator score, contract value, and effort estimate.
> - **FR9.3:** A workspace-level Outcome Dashboard must aggregate: bids tracked, bids submitted, bids won, win rate (rolling 12-month), platform-attributed bids and their conversion, content-block reuse rate, AI-summary usage trend, hours-saved estimate (based on documented benchmarks).
> - **FR9.4:** A monthly auto-generated "Outcome Brief" (PDF) must be available for download per workspace: top metrics, trend charts, ROI calculation (using configurable assumptions: hourly rate, hours-saved-per-bid baseline).
> - **FR9.5:** Outcome metrics must be available via API (Pro+ tier and above) for embedding in customer's own QBR decks.

### Architectural Questions for Winston

1. **Analytics service location:** Extend existing analytics in `client-api`, or new `analytics-api` service? Volume is low (event-rate <100/sec across all customers for years); recommend **stay in client-api with a dedicated module** for now, split when justified.
2. **Materialized view refresh:** Per FR7.2, materialized views refresh concurrently. Confirm new outcome views fit existing pattern.
3. **PDF generation:** Reuse WeasyPrint pipeline; outcome brief is a structured report well-suited to existing patterns.
4. **Configurability:** Hourly rate and hours-saved baseline must be per-workspace; default to industry benchmarks (€60K loaded cost / 19% search-time → ~€11.4K baseline savings) but tenant-overridable.

### Implied Epic Preview
- **Epic 19: Outcome Telemetry & Renewal Proof Brief** — opportunity tagging, outcome capture UI, dashboard widgets, monthly auto-brief generation, API endpoints.

---

## Change 7 — GTM Functional Requirements (smaller, but PRD-scope)

### Why
Three GTM-level items have engineering-touchpoint scope and must appear in the PRD even though most of the work is content/marketing/sales:

### What changes in PRD

**§4 Product Scope (MVP) — Add to In Scope:**

> - **G2 / Capterra / Gartner Peer Insights** product listings with review-collection mechanism (in-app NPS prompt + opt-in to review).
> - **In-product onboarding programme** with "First Bid in 14 Days" SLA tracking and CSM-assisted milestones.

**§6 Functional Requirements — Append to existing FRs:**

> - **FR2.7:** The system must support an in-product NPS prompt (tier-gated; Pro+ and above; quarterly cadence) with opt-in routing to G2/Capterra/Gartner review forms.
> - **FR9.6:** The Outcome Dashboard must include onboarding-progress tracking: "First Bid in 14 Days" milestone with sub-tasks (workspace created, content uploaded, first opportunity reviewed, first AI summary generated, CRM connected, first bid/no-bid decision logged). CSM is alerted on stalls >7 days.

### Architectural Questions for Winston

1. **NPS implementation:** In-house, or third-party (Delighted, Wootric)? Recommend **third-party SDK** for SOC 2 / ISO 27001 evidence trail (audit-ready).
2. **Review-routing:** Deep links to G2/Capterra forms with prefilled context (customer logo, product version) — out-of-band, no privacy issue.
3. **Onboarding milestone tracking:** Implement as a structured `onboarding_milestone` model per workspace, fired by domain events. Aligned with EventBus pattern.

### Implied Epic Preview
- **Epic 20: In-Product NPS, Review Collection & Onboarding Milestone Tracking** — NPS prompt, review-routing deep links, onboarding milestone model + dashboard widget, CSM alerting on stalls.

---

## Cross-Cutting Architectural Questions for Winston

These span multiple changes and should be answered before epic decomposition begins:

1. **Tenant model migration:** With multi-client workspace + per-bid SKU + Pro+ tier, the tenancy model is materially more complex. Should there be a coordinated migration story (e.g., S14.00) that delivers the new data model + back-fills existing single-company tenants as single-workspace tenants in one atomic switch?

2. **Tier-gate matrix update:** Existing TierGate (project-context.md Epic 6) enforces Free/Starter/Pro/Enterprise. Adding **Pro+** + **per-bid SKU bypass** creates a 5-dimension matrix. Confirm `TierGate as FastAPI per-route Depends()` extends cleanly or requires refactor.

3. **Service decomposition impact:** Two new candidate services (`integrations-api`, possibly `analytics-api`). Confirm decomposition aligns with existing decomposition analysis (`EU_Solicit_Service_Decomposition_Analysis.md`).

4. **Frontend impact:** Multi-client workspace requires new top-level navigation (workspace switcher), URL routing changes (`/[locale]/workspace/[workspaceId]/...`), and Zustand store namespace updates. Significant frontend refactor — confirm sequencing.

5. **Backwards compatibility:** Existing customers (if any during this rollout) must transition to the new model without data loss or downtime. Confirm migration approach.

6. **Compliance audit dependency chain:** ISO 27001 audit cannot complete until the Trust Center is live AND all controls are evidenced in code/infra/process. Sequence the certification programme accordingly — the audit is a **Q3-Q4 2026** activity if PRD amendment lands in Q2.

---

## Updated PRD §3 Personas — Drop-In Replacement

To save the PM/architect time, here is the §3 replacement text consolidating ICP changes:

```markdown
## 3. Target Audience & Personas

EU Solicit prioritises **consulting firms** as the primary ICP, with construction/engineering, IT, and municipalities as secondary segments. Personas mapped to the dual-segment (tenders + grants) cross-sell:

- **Consulting Firms (Primary ICP, Highest WTP):** Engineering, IT, management, and EU-grant specialist consultancies. 5–500+ staff. Manage 10–50 client engagements concurrently. Need: multi-client workspace, opportunity discovery + qualification, AI proposal generation, CRM integration (HubSpot/Salesforce/Pipedrive), white-label client deliverables, ISO 27001 + GDPR posture, outcome dashboards. Buyer committee: Bid Director (economic buyer), Bid Manager (daily user), IT/Security (gate), Finance/Procurement (gate). Sales cycle: 4–8 weeks (small specialists), 12–28 weeks (mid-large firms).

- **Construction & Engineering Firms (Secondary):** High-volume public works tenders, complex documentation. Need: requirement extraction, compliance checklists, pricing assistant, scoring simulator. Buyer: Bid Manager + Operations Director.

- **IT Companies (Secondary):** Frequent RFP responses, EU digital programme grants. Need: rapid proposal generation, content reuse, grant eligibility matching. Buyer: Sales Director + Bid Manager.

- **Municipalities (Tertiary, Long Sales Cycle):** EU structural funds, limited in-house capacity. Need: grant eligibility matching, logframe generators, budget builders, programme rule guidance. Buyer: Public Procurement Office + Mayor's Office.
```

---

## Recommended Epic Sequencing for Sprint Plan

Based on the implied epics above, the recommended sequencing for `bmad-create-epics-and-stories` to consume:

| Order | Epic | Justification | Dependencies |
|---|---|---|---|
| **1** | **Epic 14: Multi-Client Workspace Data Model & RBAC** | Foundational — all other epics assume workspace-scope | None (but blocks 15–20) |
| **2** | **Epic 18: Trust Center & Compliance Posture** | Procurement gate; relatively isolated; can run parallel to 14 | Independent |
| **3** | **Epic 15: Per-Bid Pricing SKU & Pro+ Tier** | Unlock GTM monetisation; depends on workspace for billing scope | Epic 14 |
| **4** | **Epic 19: Outcome Telemetry & Renewal Proof** | Retention engine; depends on workspace for scope | Epic 14 |
| **5** | **Epic 16: CRM Integrations (HubSpot, Salesforce, Pipedrive)** | Tier-1 deal blocker for Pro+ | Epic 14, Epic 15 |
| **6** | **Epic 17: Slack & Teams Notifications** | Lower priority than CRM; ships as Pro+ feature | Epic 14, Epic 15 |
| **7** | **Epic 20: NPS, Reviews & Onboarding Milestones** | Optimises retention + review pipeline; can ship after core | Epic 14, Epic 19 |
| **Parallel programme** | **ISO 27001 Audit Preparation** | Sprint-spanning; controls inventory + evidence | Epic 18 |
| **NFR cross-cutting** | **99.9% SLA observability uplift** | Refines existing reliability work | k6 baseline carry-forward |

---

## Open Questions Requiring Deb's Decision Before Architect Engagement

1. **Existing customer impact:** Are there any existing paying customers from the v1.0 PRD execution who would be impacted by the multi-client-workspace migration? If yes, sequencing must include data migration ACs.
2. **Pricing for Pro+ tier:** Recommended €199–€399/user/month range. Need final number for §4 PRD update and Stripe configuration.
3. **Per-bid SKU pricing:** Recommended €99 / €199 / €299 by tier. Need confirmation.
4. **Trust Center launch deadline:** Recommended Month 2; flag if business cycle requires earlier (e.g., a known enterprise sales cycle in flight requiring proof now).
5. **ISO 27001 budget approval:** Audit + remediation typically €30–80K all-in for a SaaS of this complexity. Need budget commitment before engaging auditor.
6. **CRM integration prioritisation:** Recommended HubSpot first (most consulting-firm density in BG/CEE/SME), then Pipedrive, then Salesforce. Confirm or override.

---

## Document Status

| Item | Status |
|---|---|
| Research artefact | Complete (`research/market-eu-govwin-tender-intelligence-research-2026-04-25.md`) |
| PRD-amendment brief | **This document — Draft v1.0** |
| Canonical PRD update | Pending (next step in this session) |
| Architectural evaluation | Pending Winston engagement |
| Sprint-change-proposal | Pending (recommend `bmad-correct-course` after architect feedback) |
| Epic decomposition | Pending (`bmad-create-epics-and-stories`) |

**Author:** Mary (Business Analyst)
**Date:** 2026-04-25
**Review owner:** Winston (Architect) — for technical feasibility on every section above; PM (or facilitator) for §3/§4/§7 prose adoption.
