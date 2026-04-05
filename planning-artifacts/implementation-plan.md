# EU Solicit — Implementation Plan

**Date**: 2026-04-05 | **Status**: Draft | **Sprint cadence**: 2 weeks

**Source Documents**: [PRD v1](../EU_Solicit_PRD_v1.md) | [Architecture v4](../EU_Solicit_Solution_Architecture_v4.md) | [UX Supplement v1](../EU_Solicit_UX_Supplement_v1.md) | [Deployment Strategy v1](../EU_Solicit_Deployment_Branch_Strategy.md)

---

## 1. Epic Overview

| Epic | Name | Sprint | Dependencies | Milestone |
|------|------|--------|-------------|-----------|
| E01 | Infrastructure & Monorepo Foundation | 1–2 | None | Demo |
| E02 | Authentication & Identity | 1–2 | E01 | Demo |
| E03 | Frontend Shell & Design System | 1–2 | None | Demo |
| E04 | AI Gateway Service | 3–4 | E01 | Demo |
| E05 | Data Pipeline & Opportunity Ingestion | 3–4 | E01, E04 | Demo |
| E06 | Opportunity Discovery & Intelligence | 5–6 | E02, E03, E05 | Demo |
| E07 | Proposal Generation & Document Intelligence | 7–8 | E04, E06 | Demo |
| E08 | Subscription & Billing | 9–10 | E02, E06 | Beta |
| E09 | Notifications, Alerts & Calendar | 9–10 | E01, E05, E06 | Beta |
| E10 | Collaboration, Tasks & Approvals | 11–12 | E02, E07 | MVP |
| E11 | EU Grant Specialization & Compliance | 11–12 | E04, E06, E07 | MVP |
| E12 | Analytics, Reporting & Admin Platform | 13–14 | E05, E06, E07, E08 | MVP |

---

## 2. Dependency Graph

```
E01 ─────────────────┬──────────┬──────────┐
(Infrastructure)     │          │          │
                     ▼          ▼          │
                    E02        E04         │
                   (Auth)   (AI GW)        │
                     │          │          │
                     │     ┌────┴────┐     │
                     │     ▼         ▼     ▼
                     │    E05       E09
                     │  (Pipeline) (Notif)
                     │     │
E03 ─────────────────┼─────┤
(Frontend)           │     │
                     ▼     ▼
                    E06 ◄──┘
                (Opportunity)
                     │
              ┌──────┴──────┐
              ▼              ▼
             E07            E08
          (Proposal)     (Billing)
              │              │
         ┌────┴────┐        │
         ▼         ▼        │
        E10       E11       │
     (Collab)  (Grants)     │
                             │
              ┌──────────────┘
              ▼
             E12
          (Analytics)
```

---

## 3. Implementation Phases

The 12 epics group into **5 phases** based on technical dependencies, demo-first priority, and parallelism. Each phase has a clear entry gate (what must be done before it starts) and exit gate (what it delivers).

```
 PHASE 1          PHASE 2          PHASE 3           PHASE 4          PHASE 5
 Foundation       Data & AI        Core Product       Commercial       Scale & Launch
 Sprint 1–2       Sprint 3–4       Sprint 5–8         Sprint 9–10      Sprint 11–14
 ──────────       ──────────       ──────────         ──────────       ──────────
 ┌─────┐          ┌─────┐         ┌─────┐            ┌─────┐         ┌─────┐
 │ E01 │──────────│ E04 │────┐    │ E06 │────────────│ E08 │────────│ E12 │
 │Infra│    ┌────│AI GW│    │    │Disc.│    ┌──────│Bill.│    ┌──│Anlyt│
 └─────┘    │     └─────┘    │    └──┬──┘    │       └─────┘    │   └─────┘
 ┌─────┐    │     ┌─────┐    │       │       │       ┌─────┐    │   ┌─────┐
 │ E02 │────┘    │ E05 │────┘       │       │       │ E09 │    │   │ E10 │
 │Auth │         │Pipe.│            ▼       │       │Notif│    │   │Collab│
 └─────┘         └─────┘         ┌─────┐    │       └─────┘    │   └─────┘
 ┌─────┐                         │ E07 │────┘                  │   ┌─────┐
 │ E03 │─────────────────────────│Prop.│                       │   │ E11 │
 │Front│                         └─────┘                       │   │Grant│
 └─────┘                                                       │   └─────┘
                                  ▲ DEMO                        │
                                                    ▲ BETA      ▲ MVP LAUNCH
```

### Phase 1: Foundation (Sprint 1–2, Weeks 1–4)

**Purpose**: Establish all infrastructure, authentication, and the frontend shell so that all subsequent phases have a working local dev environment and deployable services.

**Entry gate**: Project kickoff.
**Exit gate**: All 5 services start in Docker Compose; a user can register, log in, and see the app shell.

| Epic | Name | Parallelism | Stories |
|------|------|-------------|---------|
| **E01** | Infrastructure & Monorepo Foundation | Starts immediately | 10 |
| **E02** | Authentication & Identity | Starts immediately; depends on E01 DB schemas mid-sprint | 12 |
| **E03** | Frontend Shell & Design System | Fully independent; starts immediately | 12 |

**Why these together**: E01, E02, and E03 are the only epics with no upstream dependencies. They form the foundation that every other epic builds on. E02 depends on E01's DB schemas, but the team can parallelize: one dev sets up infra (E01) while another builds the frontend shell (E03) and a third scaffolds auth models (E02) — connecting them mid-sprint when schemas land.

**Key deliverables**:
- Monorepo with 5 service directories, 3 shared packages, Docker Compose
- PostgreSQL 6 schemas + Redis Streams (7 streams, 4 consumer groups)
- GitHub Actions CI matrix
- JWT RS256 auth (email/password + Google OAuth2)
- Company/user profiles, entity-level RBAC middleware, audit trail
- Next.js app shell (sidebar, topbar, i18n, auth pages, design system)

---

### Phase 2: Data & AI Layer (Sprint 3–4, Weeks 5–8)

**Purpose**: Stand up the AI integration backbone and start populating the database with real opportunity data. After this phase, the platform has content to display.

**Entry gate**: Phase 1 complete (services run, auth works, event bus operational).
**Exit gate**: KraftData agents callable via AI Gateway; AOP/TED/EU Grant crawlers producing opportunity records in the database.

| Epic | Name | Parallelism | Stories |
|------|------|-------------|---------|
| **E04** | AI Gateway Service | Starts at sprint 3 open; no dependency on E02/E03 | 10 |
| **E05** | Data Pipeline & Opportunity Ingestion | Starts at sprint 3; depends on E04 for KraftData calls | 12 |

**Why these together**: E04 (AI Gateway) is the prerequisite for all AI agent calls across the platform. E05 (Data Pipeline) is its first consumer. Building them together ensures the KraftData integration is validated end-to-end with real crawler agents before the client-facing features depend on it.

**Key deliverables**:
- AI Gateway: KraftData HTTP client, agent registry (29 entries), circuit breaker, SSE proxy, webhook receiver
- Data Pipeline: Celery Beat scheduler, crawler orchestration (3 KraftData agents), normalization, enrichment
- 500+ opportunities in the database from AOP + TED + EU grants
- Submission guide auto-generation per opportunity

---

### Phase 3: Core Product (Sprint 5–8, Weeks 9–16) → DEMO MILESTONE

**Purpose**: Build the two features that define the product: opportunity discovery and AI-powered proposal generation. This phase delivers the demoable end-to-end flow.

**Entry gate**: Phase 2 complete (AI Gateway operational, opportunities in database).
**Exit gate**: A user can search opportunities, view AI summaries, generate a proposal draft, run compliance checks, and export as PDF.

| Epic | Name | Sprint | Stories |
|------|------|--------|---------|
| **E06** | Opportunity Discovery & Intelligence | 5–6 | 14 |
| **E07** | Proposal Generation & Document Intelligence | 7–8 | 16 |

**Why this sequence**: E06 must land before E07 because proposal generation requires an opportunity context (selected opportunity, uploaded documents, company profile). E06 also validates the full stack end-to-end (frontend → Client API → AI Gateway → KraftData) before the more complex proposal workflow builds on it.

**E06 deliverables** (Sprint 5–6):
- Opportunity search, listing, detail with tier gating (free vs paid views)
- Document upload with S3 + ClamAV virus scanning
- AI executive summaries (SSE streaming) with usage metering
- Smart matching (relevance scores), submission guide display
- Upgrade prompt when tier/usage limits hit

**E07 deliverables** (Sprint 7–8):
- Proposal Generator Workflow (SSE streaming, multi-agent)
- Tiptap rich text editor with section-based editing
- Requirement checklist, compliance checker, clause risk flagging
- Scoring simulator scorecard, pricing assistant, win theme extractor
- Content blocks library, version history
- PDF/DOCX export

**DEMO MILESTONE at end of Sprint 8**: Register → browse opportunities → view AI summary → generate proposal → compliance check → export PDF.

---

### Phase 4: Commercial Readiness (Sprint 9–10, Weeks 17–20) → BETA MILESTONE

**Purpose**: Make the platform production-ready for paying customers. Add live billing, email notifications, and calendar integration. After this phase, real users can sign up, pay, and get value.

**Entry gate**: Phase 3 complete (demo milestone achieved, core product functional).
**Exit gate**: Users can subscribe via Stripe, receive alert emails, sync deadlines to their calendar.

| Epic | Name | Parallelism | Stories |
|------|------|-------------|---------|
| **E08** | Subscription & Billing | Fully parallel with E09 | 14 |
| **E09** | Notifications, Alerts & Calendar | Fully parallel with E08 | 14 |

**Why these together**: E08 and E09 are independent of each other (no cross-dependency) but both depend on the core product from Phase 3. They represent the "pay & engage" layer — without billing, there's no revenue; without notifications, there's no retention. Doing both in one phase lets you launch beta with both.

**E08 deliverables**:
- Stripe subscriptions (4 tiers), Checkout, Customer Portal
- 14-day free trial (no credit card), graceful downgrade on expiry
- Usage metering (Redis counters → Stripe), per-bid add-on purchases
- EU VAT via Stripe Tax, Enterprise custom invoicing
- Pricing page, subscription management UI, trial banner

**E09 deliverables**:
- Notification Service (Celery workers + Beat)
- Alert preferences, email digest assembly, SendGrid delivery
- iCal feed, Google Calendar sync, Microsoft Outlook sync
- Task/approval email notifications, trial expiry reminders
- Usage sync to Stripe

**BETA MILESTONE at end of Sprint 10**: Closed beta with 10–20 companies using live billing.

---

### Phase 5: Advanced Features & Launch (Sprint 11–14, Weeks 21–28) → MVP LAUNCH

**Purpose**: Complete all remaining product capabilities — collaboration, EU grants, compliance admin, analytics, and the admin platform — then polish and launch.

**Entry gate**: Phase 4 complete (billing live, notifications flowing).
**Exit gate**: Full-featured platform launched at EUSolicit.com.

This phase splits into two waves based on dependencies:

#### Wave A (Sprint 11–12): Collaboration + Grants + Compliance

| Epic | Name | Parallelism | Stories |
|------|------|-------------|---------|
| **E10** | Collaboration, Tasks & Approvals | Parallel with E11 | 16 |
| **E11** | EU Grant Specialization & Compliance | Parallel with E10 | 16 |

**Why together**: E10 and E11 are independent of each other but both depend on E07 (proposals). They represent depth features for Professional/Enterprise users — E10 adds team workflows, E11 adds grant-specific tools and compliance admin. Building both in parallel fills out the product breadth before the final analytics/admin layer.

**E10 deliverables**: Per-proposal roles, section locking, comments, task board (kanban), task templates, approval workflows, bid/no-bid decisions, bid outcomes, preparation time logging.

**E11 deliverables**: Grant eligibility, budget builder, consortium finder, logframe generator, reporting templates, ESPD auto-fill, compliance framework admin, regulation tracker.

#### Wave B (Sprint 13–14): Analytics, Admin & Launch Polish

| Epic | Name | Stories |
|------|------|---------|
| **E12** | Analytics, Reporting & Admin Platform | 18 |

**Why last**: E12 depends on data generated by nearly every other epic — opportunities (E05), proposals (E07), billing (E08), bid outcomes (E10), competitor data (E05). It also includes launch polish (performance, security audit, onboarding) which must happen at the end.

**E12 deliverables**: Market intelligence, ROI tracker, team performance, competitor intelligence, pipeline forecast, usage dashboards. Report generation + export. Admin platform (tenants, crawlers, white-label, audit log). Enterprise API. Performance optimization, security audit, onboarding wizard.

**MVP LAUNCH at end of Sprint 14**: Public launch at EUSolicit.com.

---

## 4. Phase Summary

| Phase | Sprints | Weeks | Epics | Stories | Exit Milestone |
|-------|---------|-------|-------|---------|----------------|
| **1. Foundation** | 1–2 | 1–4 | E01, E02, E03 | 34 | Services run, auth works, app shell |
| **2. Data & AI** | 3–4 | 5–8 | E04, E05 | 22 | AI Gateway live, opportunities in DB |
| **3. Core Product** | 5–8 | 9–16 | E06, E07 | 30 | **DEMO** — e2e flow |
| **4. Commercial** | 9–10 | 17–20 | E08, E09 | 28 | **BETA** — billing + notifications |
| **5. Advanced & Launch** | 11–14 | 21–28 | E10, E11, E12 | 50 | **MVP LAUNCH** |
| **Total** | **14** | **28** | **12** | **164** | |

---

## 5. Sprint Allocation Detail

### Demo Milestone — Sprints 1–8 (Weeks 1–16)

**Goal**: End-to-end demoable flow: register → browse opportunities → generate proposal → export PDF.

**Sprint 1–2: Foundation (E01 + E02 + E03 in parallel)**

| Work | Epic | Team |
|------|------|------|
| Monorepo scaffold (5 services, 3 packages, frontend, infra) | E01 | Backend |
| Docker Compose for local dev (all services + PG + Redis + MinIO + ClamAV) | E01 | Backend |
| PostgreSQL schemas + Alembic migrations scaffold (all 6 schemas) | E01 | Backend |
| Redis Streams event bus setup (7 streams, 4 consumer groups) | E01 | Backend |
| GitHub Actions CI scaffold (lint, type check, test matrix) | E01 | Backend |
| `eusolicit-common` library (config, logging, middleware base, health checks, exceptions) | E01 | Backend |
| `eusolicit-models` library (shared Pydantic DTOs, event schemas) | E01 | Backend |
| `eusolicit-kraftdata` library (KraftData client types, not implementation) | E01 | Backend |
| Email/password registration + login (Client API) | E02 | Backend |
| Google OAuth2 social login | E02 | Backend |
| JWT RS256 token issuance + refresh (15-min access, 7-day refresh) | E02 | Backend |
| Company + user profile CRUD | E02 | Backend |
| Entity-level RBAC middleware (company role + entity permission) | E02 | Backend |
| Audit trail middleware + shared audit log table | E02 | Backend |
| Next.js 14 app setup (client + admin shells) | E03 | Frontend |
| Design system: Tailwind config, shadcn/ui setup, design tokens | E03 | Frontend |
| App shell: sidebar, topbar, responsive layout, collapsible nav | E03 | Frontend |
| i18n setup (next-intl, BG + EN) | E03 | Frontend |
| Auth pages: login, register, forgot password, OAuth callback | E03 | Frontend |
| Company profile setup wizard | E03 | Frontend |

**Sprint 3–4: Data & AI Layer (E04 + E05)**

| Work | Epic | Team |
|------|------|------|
| AI Gateway service scaffold (FastAPI, internal ClusterIP) | E04 | Backend |
| KraftData HTTP client (httpx async, API key auth) | E04 | Backend |
| Agent registry (logical name → KraftData ID mapping, config-driven) | E04 | Backend |
| Circuit breaker + retry logic (fail-open after 5 failures, 30s retry) | E04 | Backend |
| SSE stream proxy (KraftData SSE → Client API SSE) | E04 | Backend |
| KraftData webhook receiver (signature verification, event routing) | E04 | Backend |
| Execution logging (latency, success/failure, agent type → gateway schema) | E04 | Backend |
| Agent run endpoints: `/agents/{id}/run`, `/agents/{id}/run-stream` | E04 | Backend |
| Team run endpoint: `/teams/{id}/run` | E04 | Backend |
| Workflow run endpoints: `/workflows/{id}/run`, `/workflows/{id}/run-stream` | E04 | Backend |
| Storage resource endpoint: `/storage/{id}/files` | E04 | Backend |
| Data Pipeline service scaffold (Celery workers + Beat) | E05 | Backend |
| Crawl orchestration: trigger AOP Crawler Agent via AI Gateway | E05 | Backend |
| Crawl orchestration: trigger TED Crawler Agent via AI Gateway | E05 | Backend |
| Crawl orchestration: trigger EU Grant Portal Agent via AI Gateway | E05 | Backend |
| Crawl result ingestion: process agent output → opportunity upsert | E05 | Backend |
| Data Normalization Team integration (via AI Gateway) | E05 | Backend |
| Opportunity enrichment: Relevance Scoring Agent | E05 | Backend |
| Celery Beat schedule configuration (crawl frequencies) | E05 | Backend |
| `opportunities.ingested` event publishing to Redis Streams | E05 | Backend |
| Submission Guide Agent integration + guide storage | E05 | Backend |

**Sprint 5–6: Core Discovery Experience (E06)**

| Work | Epic | Team |
|------|------|------|
| Opportunity search API (full-text search, CPV, region, budget, deadline filters) | E06 | Backend |
| Opportunity listing API (paginated, tier-gated: free vs paid response shapes) | E06 | Backend |
| Tier gating middleware (region, CPV sector, budget range checks per tier) | E06 | Backend |
| Usage gate middleware (Redis INCR, atomic check, 429 + upgrade prompt) | E06 | Backend |
| Opportunity detail API (full data for paid, limited for free) | E06 | Backend |
| Document upload API (S3 presigned URL, ClamAV scan, metadata record) | E06 | Backend |
| Document download API (S3 presigned URL, access control) | E06 | Backend |
| AI summary generation (Executive Summary Agent via AI Gateway, SSE streaming) | E06 | Backend |
| AI summary usage metering (count against tier limit) | E06 | Backend |
| Smart matching: Relevance Scoring Agent display on listing/detail | E06 | Backend |
| Submission guide API (read-only, served from pipeline schema) | E06 | Backend |
| Opportunity listing page (table + card views, filters, pagination) | E06 | Frontend |
| Free-tier limited opportunity card (name, deadline, location, type, status) | E06 | Frontend |
| Paid-tier full opportunity card (all metadata, relevance score) | E06 | Frontend |
| Opportunity detail page (tabbed: overview, documents, requirements, AI analysis) | E06 | Frontend |
| AI summary panel (streaming display with loading state) | E06 | Frontend |
| Document upload component (drag-drop, progress, virus scan status) | E06 | Frontend |
| Submission guide panel (step-by-step accordion) | E06 | Frontend |
| Search + filter bar (CPV multi-select, region, budget range slider, deadline picker) | E06 | Frontend |
| Upgrade prompt component (shown when tier/usage limits hit) | E06 | Frontend |

**Sprint 7–8: Proposal Engine (E07)**

| Work | Epic | Team |
|------|------|------|
| Proposal Generator Workflow integration (SSE streaming via AI Gateway) | E07 | Backend |
| Proposal CRUD API (create from opportunity, list, get, update metadata) | E07 | Backend |
| Proposal versioning API (create version, list versions, get version, diff) | E07 | Backend |
| Proposal content save API (section-based, auto-save) | E07 | Backend |
| Requirement Checklist Agent integration | E07 | Backend |
| Compliance Checker Agent integration | E07 | Backend |
| Clause Risk Analyzer Agent integration | E07 | Backend |
| Scoring Simulator Agent integration (scorecard response) | E07 | Backend |
| Pricing Assistant Agent integration | E07 | Backend |
| Win Theme Extractor Agent integration | E07 | Backend |
| Content blocks CRUD API (create, list, search by tag/category, update, approve) | E07 | Backend |
| Document export API (PDF via reportlab, DOCX via python-docx) | E07 | Backend |
| Proposal workspace page (full-width editor + side panels) | E07 | Frontend |
| Tiptap rich text editor integration (section-based editing) | E07 | Frontend |
| AI draft generation panel (trigger, streaming progress, insert into editor) | E07 | Frontend |
| Requirement checklist panel (checkable items, linked to proposal sections) | E07 | Frontend |
| Compliance check results panel (pass/fail per criterion) | E07 | Frontend |
| Scoring simulator scorecard UI (per-criterion breakdown, improvement suggestions) | E07 | Frontend |
| Pricing assistant panel | E07 | Frontend |
| Version history sidebar (version list, diff viewer, rollback action) | E07 | Frontend |
| Content blocks library modal (search, preview, insert into editor) | E07 | Frontend |
| PDF/DOCX export button with format selection | E07 | Frontend |
| Clause risk highlights in editor (inline annotations) | E07 | Frontend |

**--- DEMO MILESTONE (end of Sprint 8) ---**

### Beta Milestone — Sprints 9–10 (Weeks 17–20)

**Goal**: Production-ready with live billing, notifications, and calendar sync for closed beta (Phase 4).

**Sprint 9–10: Billing + Notifications (E08 + E09 in parallel)**

| Work | Epic | Team |
|------|------|------|
| Stripe subscription model (tier definitions, access policies) | E08 | Backend |
| Stripe Checkout integration (upgrade flow) | E08 | Backend |
| Stripe webhook handler (subscription lifecycle events) | E08 | Backend |
| Stripe Customer Portal integration (self-service management) | E08 | Backend |
| 14-day trial flow (auto-create at registration, downgrade on expiry) | E08 | Backend |
| Usage metering persistence (Redis counters → Stripe usage records) | E08 | Backend |
| Per-bid add-on purchase flow (Stripe Checkout one-time) | E08 | Backend |
| EU VAT configuration (Stripe Tax, tax ID collection, VIES validation) | E08 | Backend |
| Enterprise custom invoicing (Stripe Invoicing API) | E08 | Backend |
| Pricing page UI (tier comparison, upgrade CTA) | E08 | Frontend |
| Checkout flow UI (Stripe Checkout redirect) | E08 | Frontend |
| Subscription management UI (current plan, usage, portal link) | E08 | Frontend |
| Trial banner + countdown + upgrade prompt | E08 | Frontend |
| Add-on purchase UI (opportunity detail integration) | E08 | Frontend |
| Notification Service scaffold (Celery workers + Beat) | E09 | Backend |
| Alert preference CRUD API (CPV, regions, budget, deadline proximity, schedule) | E09 | Backend |
| Alert digest assembly (match preferences against new opportunities) | E09 | Backend |
| SendGrid email delivery (digest templates, transactional) | E09 | Backend |
| iCal feed generation (tracked opportunities → .ics endpoint) | E09 | Backend |
| Google Calendar sync (OAuth2 flow, event CRUD, periodic sync task) | E09 | Backend |
| Microsoft Outlook sync (OAuth2 flow, Graph API, periodic sync task) | E09 | Backend |
| Task/approval event consumers (email notifications) | E09 | Backend |
| Trial expiry reminder (3-day warning email) | E09 | Backend |
| Usage sync to Stripe (Celery periodic task) | E09 | Backend |
| Alert preferences settings page | E09 | Frontend |
| Calendar connection management page (connect/disconnect, iCal URL copy) | E09 | Frontend |
| Notification preferences UI | E09 | Frontend |

### MVP Milestone — Sprints 11–14 (Weeks 21–28)

**Goal**: Full-featured launch at EUSolicit.com (Phase 5, Wave A + Wave B).

**Sprint 11–12: Collaboration + Grants + Compliance (E10 + E11 in parallel) — Wave A**

| Work | Epic | Team |
|------|------|------|
| Proposal collaborators API (assign/remove roles per proposal) | E10 | Backend |
| Section locking API (acquire, release, auto-expire at 15 min) | E10 | Backend |
| Proposal comments API (create, resolve, list by section/version) | E10 | Backend |
| Task CRUD API (create, assign, update status, list by opportunity/company) | E10 | Backend |
| Task dependency API (add/remove dependencies, DAG validation) | E10 | Backend |
| Task templates API (CRUD, apply template to opportunity) | E10 | Backend |
| Approval workflow CRUD API (define stages per company) | E10 | Backend |
| Approval decision API (approve/reject/return, log decision) | E10 | Backend |
| Bid/no-bid decision API (AI scorecard + user override + log) | E10 | Backend |
| Bid outcome recording API (won/lost/withdrawn, feedback, scores) | E10 | Backend |
| Bid preparation log API (log hours/costs per user per proposal) | E10 | Backend |
| Collaborator management UI (invite, role selector, remove) | E10 | Frontend |
| Section lock indicators in editor (locked by name, unlock on timeout) | E10 | Frontend |
| Comments panel in proposal editor (thread view, resolve action) | E10 | Frontend |
| Task board page (kanban columns: pending/in-progress/completed/blocked) | E10 | Frontend |
| Approval pipeline page (stage progress, sign-off actions) | E10 | Frontend |
| Bid/no-bid decision page (scorecard display, override form) | E10 | Frontend |
| Bid outcome recording form | E10 | Frontend |
| Grant Eligibility Agent integration | E11 | Backend |
| Budget Builder Agent integration | E11 | Backend |
| Consortium Finder Agent integration + partner display | E11 | Backend |
| Logframe Generator Agent integration | E11 | Backend |
| Reporting Template Generator Agent integration | E11 | Backend |
| ESPD profile CRUD API (manage ESPD field mappings per company) | E11 | Backend |
| ESPD auto-fill API (generate pre-filled ESPD from profile + agent) | E11 | Backend |
| Compliance framework CRUD API (Admin API) | E11 | Backend |
| Per-opportunity framework assignment API (Admin API) | E11 | Backend |
| Framework auto-suggestion agent integration | E11 | Backend |
| Regulation Tracker Agent integration (Admin API, scheduled) | E11 | Backend |
| EU grant tools page (eligibility, budget, consortium, logframe) | E11 | Frontend |
| ESPD builder page (structured form, auto-fill action) | E11 | Frontend |
| Compliance framework admin page | E11 | Frontend |
| Regulation tracker admin page | E11 | Frontend |

**Sprint 13–14: Analytics + Admin + Polish (E12) — Wave B**

| Work | Epic | Team |
|------|------|------|
| Analytics materialized views (PG views, Celery refresh task) | E12 | Backend |
| Market intelligence API (sector analytics, authority rankings, trends) | E12 | Backend |
| ROI tracker API (bid investment vs. contract value) | E12 | Backend |
| Team performance API (per-user metrics) | E12 | Backend |
| Competitor intelligence API (patterns, win rates, pricing) | E12 | Backend |
| Pipeline forecast API (predicted opportunities) | E12 | Backend |
| Usage dashboard API (current period vs. limits) | E12 | Backend |
| Report generation task (PDF/DOCX, scheduled + on-demand) | E12 | Backend |
| Admin API: tenant management (companies, subscriptions, usage overview) | E12 | Backend |
| Admin API: crawler management (run history, schedule config, manual trigger) | E12 | Backend |
| Admin API: white-label configuration (subdomain, branding, email domain) | E12 | Backend |
| Admin API: audit log viewer (searchable, filterable, exportable) | E12 | Backend |
| Admin API: platform analytics (signup funnel, conversion, churn) | E12 | Backend |
| Enterprise API: OpenAPI auto-generated spec + documentation | E12 | Backend |
| Market intelligence dashboard page (charts, tables, filters) | E12 | Frontend |
| ROI tracker dashboard page | E12 | Frontend |
| Team performance dashboard page | E12 | Frontend |
| Competitor intelligence dashboard page | E12 | Frontend |
| Pipeline forecast dashboard page | E12 | Frontend |
| Usage dashboard page (meters, limits, upgrade prompts) | E12 | Frontend |
| Report generation UI (schedule configuration, download) | E12 | Frontend |
| Admin: tenant management page | E12 | Frontend |
| Admin: crawler management page | E12 | Frontend |
| Admin: white-label configuration page | E12 | Frontend |
| Admin: audit log viewer page | E12 | Frontend |
| Performance optimization + load testing | E12 | Backend |
| Security audit (OWASP, network policies, secret rotation) | E12 | Backend |
| User onboarding flow (guided first-run experience) | E12 | Frontend |

---

## 6. Risk Register

| Risk | Impact | Mitigation |
|------|--------|-----------|
| KraftData agent quality insufficient for demo | Demo blocked | Run eval-runs in Sprint 3; have manual fallback prompts ready |
| Crawler agents fail on portal changes | Stale data | AI Gateway circuit breaker; manual seed data for demo |
| Stripe integration complexity (VAT, trials) | Beta delay | Start with simple subscription in Sprint 9; add VAT/trials incrementally |
| Tiptap collaboration complexity | Feature cut | MVP uses pessimistic locking only; real-time CRDT deferred to post-MVP |
| Team capacity (2-3 devs) | Timeline slip | AI-assisted development; strict scope control; demo milestone is priority |

---

## 7. Definition of Done

A story is "done" when:
1. Code is implemented, linted (`ruff`), type-checked (`mypy`), and passes unit tests
2. API endpoints have integration tests (pytest-asyncio)
3. Frontend components have basic interaction tests
4. Alembic migration included if schema changes are needed
5. API documented in OpenAPI (auto-generated by FastAPI)
6. PR reviewed and merged to `main`
7. Acceptance criteria verified in local Docker Compose environment

An epic is "done" when all stories are done and the feature is end-to-end functional in the staging environment.

---

*Document version: 1.0 | Date: 2026-04-05 | Status: Draft*
