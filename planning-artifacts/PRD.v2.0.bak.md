---
stepsCompleted: ["step-01-init", "step-02-discovery", "step-03-success", "step-04-journeys", "step-05-domain", "step-06-innovation", "step-07-project-type", "step-08-scoping", "step-09-functional", "step-10-nonfunctional", "step-11-polish", "step-12-complete"]
inputDocuments: ["product-brief.md", "project-context.md", "architecture.md", "ux-spec.md", "gap-resolution-requirements-p1.md", "gap-resolution-requirements-p2.md", "epics/*"]
workflowType: 'prd'
project_name: 'EU Solicit'
user_name: 'Deb'
date: '2026-04-26'
version: '2.0'
status: 'Approved (Canonical Refresh)'
---

# Product Requirements Document — EU Solicit

**Author:** Deb (Agentic Workflow)
**Date:** 2026-04-26
**Version:** 2.0 (Canonical Refresh)
**Status:** Approved

> **Note on inputs:** The on-disk product brief (`product-brief.md`) was a single-line drift sentinel ("Drift detected. Review requirements.") at the time of this refresh. This PRD therefore consolidates the canonical product specification from the prior approved PRD v1.0, the Architecture Decision Document, the UX Specification, the P1/P2 Gap Resolution Requirements, the in-flight epic specs (E01–E21), and the patterns/anti-patterns recorded in `project-context.md` from epics shipped to date. It is the single source of truth for product scope, functional/non-functional requirements, feature-level stories, and acceptance criteria.

---

## 1. Executive Summary

**EU Solicit** is a multi-tenant SaaS platform that automates the lifecycle of public procurement and EU grant applications, initially targeting Bulgaria-based bidders pursuing AOP, TED, and EU structural-fund opportunities. It transforms a multi-week, paper-heavy bid response process into an AI-assisted workflow spanning opportunity discovery, document analysis, drafting, compliance validation, collaboration, approvals, and submission-ready export.

The platform is built on a Python microservices backend (FastAPI, Celery, PostgreSQL 16, Redis 7) and a Next.js 14 Turborepo frontend. AI capabilities are delivered through the **KraftData Agentic AI** platform via the internal **AI Gateway** service, which provides circuit-breaker-protected, rate-limited, streaming access to agents for crawling, parsing, scoring, drafting, and compliance validation.

The MVP delivers a self-service experience across four subscription tiers (Free, Starter, Professional, Enterprise) gated by Stripe. It does **not** include direct e-procurement portal submission, digital signatures, or CAIS/TED eSender integration — those are explicitly out of scope and tracked as future-phase items.

### Goals

1. **Reduce bid preparation time** by ≥60% versus manual workflows on tenders ≤200 pages.
2. **Increase win-rate** for Professional/Enterprise customers by ≥15% via AI scoring, compliance validation, and institutional-memory reuse.
3. **Achieve unit economics** of ≤€0.40 KraftData spend per €1 of subscription revenue at GA, enforced by per-company usage caps and atomic Lua-script metering.
4. **Operate within EU data residency** boundaries (GDPR Article 44 et seq.) with full audit traceability.

### Non-Goals (MVP)

- Direct submission to AOP, TED eSender, CAIS, or any e-procurement portal.
- Qualified electronic signatures (QES) or digital seals.
- CRM/ERP synchronisation (Salesforce, HubSpot, SAP) — gated to Phase-2 (E17).
- Native mobile apps. The web app must be responsive (≥1024px) but mobile-first interaction patterns are deprioritised.
- Pricing-engine optimisation (auto-bid pricing). Pricing assistance is advisory only.

---

## 2. Target Audience & Personas

### Persona 1 — **Bid Manager** (Professional / Enterprise primary buyer)

- **Role:** Owns the proposal lifecycle from go/no-go through submission.
- **Goals:** Coordinate distributed contributors, enforce compliance, hit deadlines, raise win-rate.
- **Pain Points:** Re-keying boilerplate, hunting for compliance clauses, version-control conflicts, last-minute deadline panic.
- **Key Interactions:** Reviews AI executive summaries, makes Bid/No-Bid decisions, triggers compliance validation, approves submissions, reviews dashboards.

### Persona 2 — **Contributor / Subject-Matter Expert** (technical author)

- **Role:** Drafts technical, methodology, financial, or capability sections.
- **Goals:** A distraction-free editor, contextual access to RFP requirements, no overwrite conflicts.
- **Pain Points:** Lost edits, unclear which requirement a section is responding to, hand-offs to other authors.
- **Key Interactions:** Split-pane proposal editor, AI streaming drafts, section-level locking, comment threads.

### Persona 3 — **Reviewer / Executive** (decision-maker, occasional user)

- **Role:** Decides which tenders to pursue and signs off on final submissions.
- **Goals:** Rapidly synthesise 200-page RFPs; understand risks, win probability, ROI.
- **Pain Points:** No time to read the full RFP; needs trusted summarisation and risk flagging.
- **Key Interactions:** One-page AI summaries, Bid/No-Bid Decision Matrix, risk reports, approval workflow.

### Persona 4 — **Platform Admin** (per-company)

- **Role:** Manages company users, roles, integrations, and subscription tier.
- **Goals:** Enforce least-privilege RBAC, configure regulatory frameworks, monitor usage.
- **Key Interactions:** User management, role assignment, RBAC audit, regulatory framework selection (e.g., ZOP), Stripe billing portal.

### Persona 5 — **EU Solicit System Operator** (internal admin, separate `admin-api` service)

- **Role:** Operates the platform across all tenants.
- **Goals:** Diagnose ingestion failures, manage opportunity dataset, support customers, monitor system health.
- **Key Interactions:** Admin portal at port 3001, VPN/IP-allowlisted, dedicated audit trail.

### Target Customer Segments

- **Construction / IT / Engineering Firms** (~250 prospects in BG; high tender volume).
- **Consulting Firms** managing bids on behalf of clients (require white-label and per-client workspaces — E14).
- **Municipalities & Public Bodies** filing EU structural-fund applications (grants).
- **NGOs and Universities** filing Horizon-Europe grant applications.

---

## 3. Product Vision & Scope

### 3.1 Vision

EU Solicit is the **end-to-end intelligent co-pilot** for any organisation responding to public procurement or EU grants in the EU. The AI handles the mechanical work — discovery, parsing, summarisation, drafting, compliance — so that humans spend their time on strategy, pricing, and partnership.

### 3.2 In Scope (MVP — Sprints 1 through 13)

1. **Opportunity Intelligence & Discovery** (E02, E05, E06)
2. **Document Analysis & Parsing** (E06, E07)
3. **AI Proposal Generation** (E07) — streaming SSE, content-blocks, version history, rich-text export.
4. **Compliance Validation** (E11) — ZOP and EU directive frameworks; ESPD generator; budget calculator.
5. **Collaboration, Tasks, Approvals** (E10) — task DAG, dependency gating, templates.
6. **Subscription Billing & Tier Gating** (E08) — Stripe, VIES, four tiers, usage metering.
7. **Notifications, Alerts, Calendar Sync** (E09) — Email + in-app + Slack/Teams (E16) + Google/Outlook calendar.
8. **Authentication, Identity & RBAC** (E02) — Email + Google OAuth; RS256 JWT; company-level roles + entity overrides.
9. **Frontend Shell & Design System** (E03) — Next.js 14, shadcn/ui, BG/EN i18n.
10. **Admin Platform** (E12) — internal operator portal, KPI dashboards, opportunity moderation.
11. **Multi-Client / White-Label Workspace** (E14) — for consulting firms.

### 3.3 Out of Scope (Deferred)

- **E-procurement submission** (AOP electronic, TED eSender, CAIS).
- **Qualified electronic signatures.**
- **Generic CRM/ERP integrations** (Salesforce, HubSpot, SAP) — E17, Phase-2.
- **Native mobile apps** — web is responsive only.
- **Per-bid SKU pricing** — E15, Phase-2.
- **Trust Center / SOC 2 Type II compliance pack** — E18, Phase-2.

### 3.4 Success Metrics

| Metric | Target (GA + 90 days) | Measurement |
|---|---|---|
| Time-to-first-proposal (signup → first complete draft) | < 30 min | Analytics event timing |
| AI cost ratio (KraftData spend / subscription revenue) | ≤ 0.40 | Daily Prometheus rollup |
| Tender→Bid conversion (No-Bid rate) | ≤ 70% No-Bid | Decision Matrix logs |
| Compliance-pass rate at first validation run | ≥ 75% | Compliance reports |
| Customer NPS (Pro + Enterprise) | ≥ 35 | E20 survey |
| Platform availability (rolling 30-day) | ≥ 99.5% | Prometheus uptime |
| SSE TTFB p95 | ≤ 500 ms | k6 baseline + RUM |
| REST p95 latency | ≤ 200 ms | Prometheus histogram |

---

## 4. Functional Requirements

Functional requirements are grouped by epic; each requirement is identified `FR-Eee.xx` where `Eee` is the epic and `xx` is the sequence within the epic. Stories that satisfy each requirement are referenced where helpful.

### 4.1 Authentication, Identity & RBAC (E02)

- **FR-02.1** Users can register with email + password (max length 128, bcrypt-hashed via `run_in_executor`) or via Google OAuth (RS256-signed ID token).
- **FR-02.2** Email verification is mandatory before accessing any paid-tier feature; an unverified user has read-only Free-tier access for 7 days then is locked.
- **FR-02.3** A user belongs to one or more companies via `client.memberships` with a single role per (user, company): `admin`, `bid_manager`, `contributor`, `reviewer`, `read_only`.
- **FR-02.4** Tokens are RS256 JWTs; refresh tokens are rotating with reuse detection; `User.is_active` is checked on every protected endpoint and on token refresh.
- **FR-02.5** Entity-level RBAC overrides (`client.entity_permissions`) can grant a user broader access on a specific opportunity, proposal, or workspace beyond the role ceiling, but never lower than the role floor.
- **FR-02.6** All cross-tenant access attempts return **404** (existence-leakage safe), never 403, except where the resource is global by design.
- **FR-02.7** Admin-API is reachable only from the corporate VPN / IP allowlist enforced at FastAPI middleware (not just at the ingress).

### 4.2 Opportunity Data Pipeline (E05)

- **FR-05.1** A scheduled crawler ingests opportunities daily from AOP (Bulgaria), TED (EU-wide), and the Bulgarian EU funds portal. Crawler runs are tracked in `pipeline.crawler_runs` with start/finish/status.
- **FR-05.2** Documents up to 500 MB per package and 100 MB per file are downloaded, virus-scanned via ClamAV, and persisted to MinIO (S3-compatible) under `pipeline/documents/<opportunity_id>/<file_id>`.
- **FR-05.3** The pipeline parses CPV codes, deadlines, budgets, contracting authority, region, and document set into normalised `pipeline.opportunities`.
- **FR-05.4** Parsed opportunities are emitted to a Redis Stream `opportunity.created` for downstream consumers (matching, notifications, AI Gateway).
- **FR-05.5** A 24-hour freshness SLA applies: 95% of newly published AOP opportunities are searchable in EU Solicit within 24 hours of public posting.
- **FR-05.6** Failed crawls are retried with exponential backoff (3 attempts, 1m/5m/15m); persistent failure raises an alert and writes to dead-letter table.
- **FR-05.7** Idempotency: each opportunity has a stable upstream identifier; re-ingestion updates existing rows (DB UNIQUE constraint + Redis SETNX).
- **FR-05.8** Crawler workers use `SELECT FOR UPDATE SKIP LOCKED` to prevent duplicate processing across replicas. A Celery Beat cleanup task marks runs stuck in `running` for >2× expected duration as `failed`.
- **FR-05.9** Atomic per-step DB transactions: each (download, scan, parse, persist) step is wrapped in its own transaction so a crash never leaves status `running` or `processing` indefinitely.

### 4.3 Opportunity Discovery & Matching (E06)

- **FR-06.1** Users can search opportunities by full-text query (PostgreSQL FTS), CPV codes, regions, budget bands, deadline windows, contracting authority, and award type.
- **FR-06.2** AI relevance scoring: each user/company has a profile (capabilities, certifications, regional focus, financial capacity) used to compute a 0–100 relevance score per opportunity per company.
- **FR-06.3** Free tier sees only 6 fields per opportunity (`OpportunityFreeResponse`: title, contracting authority, deadline, CPV summary, budget band, region). Starter+ sees full detail. **Tier enforcement is field-level Pydantic, not just a route check.**
- **FR-06.4** Filter and sort state is URL-driven (`useSearchParams()` + `router.replace()`) — back/forward/share works.
- **FR-06.5** Pagination uses cursor-based pagination at the API; page-size capped at 100.
- **FR-06.6** "My matches" inbox shows opportunities scored ≥ user's threshold, sorted by score desc then deadline asc.
- **FR-06.7** Opportunity detail view shows: AI executive summary, requirements checklist, risk highlights, similar past wins, recommended next actions.
- **FR-06.8** Star/save and bid/no-bid actions are available to `bid_manager` and above; the no-bid reason is captured for ML training.

### 4.4 AI Gateway Service (E04)

- **FR-04.1** The AI Gateway is the **only** path from any service to KraftData. Direct service→KraftData calls are forbidden.
- **FR-04.2** Agent registry is YAML-driven (`config/agents.yaml`); logical names map to KraftData agent UUIDs and runtime config (timeout, max-tokens, model).
- **FR-04.3** Outbound calls use the canonical two-layer resilience pattern: `circuit_breaker(retry(http_factory))`. A 4xx response **must not** increment the circuit-breaker failure counter.
- **FR-04.4** Streaming is offered via `run-stream` (SSE); 120s idle timeout, 600s total timeout, 15s heartbeat. Endpoints return `Cache-Control: no-cache` and `X-Accel-Buffering: no`.
- **FR-04.5** Every inbound call must carry an `X-Caller-Service` header naming the originator; missing header returns 400.
- **FR-04.6** Per-company usage limits are enforced atomically via a Redis Lua script (`UsageGate`); the limit is decremented **before** the streaming response is opened.
- **FR-04.7** When KraftData is unavailable, the gateway returns a flat 503 body `{"message": "...", "code": "AGENT_UNAVAILABLE"}` (not nested under `detail`). Frontend renders an explicit recovery UI.
- **FR-04.8** Redis-gated features (UsageGate, tier-cache) **fail-open** to allow the request when Redis is unavailable, with structured logging at WARN level.

### 4.5 Document Analysis & Summarisation (E06 / E07)

- **FR-07.1** Tender document upload supports PDF and DOCX up to 100 MB / file, 500 MB / package; ClamAV scans before persistence; rejected files surface a typed error.
- **FR-07.2** A KraftData parser agent extracts: deadlines, contracting authority, budget, mandatory requirements list, evaluation criteria, contractual risk clauses.
- **FR-07.3** A summarisation agent produces a 1-page executive summary streamed via SSE.
- **FR-07.4** Risk analysis flags clauses that are unusual or above a configurable risk threshold (penalty clauses, unlimited liability, abnormal payment terms, unrealistic timelines).
- **FR-07.5** A compliance checklist is auto-generated from extracted requirements; each item links back to the source span in the original document for human verification.
- **FR-07.6** ClamAV timeouts on stuck documents are reconciled by a Celery Beat cleanup task (configurable TTL).

### 4.6 AI Proposal Generation (E07)

- **FR-07.7** A proposal is owned by exactly one opportunity and one company. Multiple drafts may exist as **versions** (`proposal_versions` table) with a `current_version_number` pointer.
- **FR-07.8** Proposals are composed of **content blocks** (TipTap JSON nodes) addressable individually for AI drafting, manual editing, locking, comments, and review.
- **FR-07.9** AI drafting is streamed via SSE using native `fetch` + `ReadableStream` on the frontend (Axios is forbidden for SSE; EventSource is forbidden because it is GET-only and silently caches).
- **FR-07.10** Drafted text appears in a Git-style diff view with explicit Accept / Reject actions; nothing is persisted until the user accepts.
- **FR-07.11** Score Simulation: the AI predicts evaluator scoring on each section and surfaces improvement suggestions.
- **FR-07.12** Version history: every accepted change creates a new version row; users can diff and restore any prior version.
- **FR-07.13** Optimistic locking on content save: client sends a SHA-256 `content_hash` of the prior version; backend uses `SELECT FOR UPDATE` + hash compare; mismatch returns **409 Conflict** with structured body `{conflict: {server_version, client_version, diff_url}}` and the frontend opens a conflict-resolution dialog.
- **FR-07.14** Section-level lock: when user A is editing block X, users B/C see a "Locked by A" banner; lock auto-releases on idle (90 s) or on save.
- **FR-07.15** Export: PDF (WeasyPrint) and DOCX (python-docx) — wrapped in `ThreadPoolExecutor` to keep the FastAPI loop free. Exports are tier-gated (Professional+).
- **FR-07.16** A Celery Beat task `reset_stuck_proposals_task` marks proposals stuck in `generating` for >TTL as `failed` so the UI never spins forever.
- **FR-07.17** Content-block bodies sent to AI Gateway are sanitised against prompt-injection (allowlist of Markdown / TipTap nodes; strip script-like tokens) **before** persistence and **before** forwarding.

### 4.7 Compliance & Grants (E11)

- **FR-11.1** Companies can be assigned one or more **regulatory frameworks** (e.g., ZOP for BG public procurement, EU Directive 2014/24/EU). Frameworks are admin-managed JSONB rule sets.
- **FR-11.2** A compliance check runs the assigned framework against a proposal version and returns: pass/fail per rule, severity, citation, suggested remediation.
- **FR-11.3** ESPD (European Single Procurement Document) generator produces an XML/PDF ESPD from the company profile + opportunity reference.
- **FR-11.4** EU grant budget calculator validates total budget against EU funding caps, co-financing rules, eligibility windows.
- **FR-11.5** Gantt-chart view of grant work-packages and milestones (E11.07).
- **FR-11.6** Float arithmetic uses a module-level tolerance constant `_ARITHMETIC_TOLERANCE = 0.01` for budget comparisons.
- **FR-11.7** Optional response fields use `null` to signal absence (UI suppresses the component) and `[]` to signal "present but empty" (UI shows empty state). This distinction is preserved through Pydantic schema, parser, and frontend conditional render.

### 4.8 Collaboration, Tasks & Approvals (E10)

- **FR-10.1** Tasks are first-class entities with title, description, assignee, priority (P1–P4), status (pending / in_progress / blocked / completed), due-date, optional ties to opportunity and proposal.
- **FR-10.2** Tasks can declare dependencies (`finish_to_start` or `finish_to_finish`) forming a DAG. Cycle detection runs DFS from the candidate target before insertion; cycles return 422.
- **FR-10.3** Status transitions enforce gates: `→ completed` is rejected if any `finish_to_start` upstream is not `completed`, or any `finish_to_finish` upstream is neither `completed` nor `in_progress`.
- **FR-10.4** Task templates are reusable factories: a JSONB array of stage definitions, each with relative-days-before-deadline, role, dependency indices. Apply-template instantiates N tasks + M dependencies in one DB transaction.
- **FR-10.5** Approval workflows: a Bid Manager can require sign-off from a Reviewer before a proposal moves from `draft` to `submitted` state.
- **FR-10.6** Comments and @mentions are supported on opportunities, proposals, and content-blocks; @mention triggers an in-app + email notification.
- **FR-10.7** Audit: every create/update/delete of a task, dependency, template, or approval writes one row to `shared.audit_log` (fire-and-forget via `asyncio.create_task`).

### 4.9 Subscription Billing & Tier Gating (E08)

- **FR-08.1** Four subscription tiers — **Free, Starter, Professional, Enterprise** — with feature, usage, and seat caps managed centrally and surfaced via the `TierGate` FastAPI dependency.
- **FR-08.2** Stripe is the system of record for subscriptions. Each company has at most one Stripe Customer; provisioning happens **after** registration returns 201, as a `BackgroundTask`. `stripe_customer_id` is nullable until provisioned.
- **FR-08.3** Tier-gated endpoints use `Depends(TierGate(min_tier="professional"))` per route — never global middleware. This guarantees field-level enforcement and testability.
- **FR-08.4** Stripe webhook validation uses `hmac.compare_digest()` on the raw body. Idempotency uses an `event_id` UNIQUE column on `shared.webhook_events` inside the processing transaction (a duplicate insert returns 200 immediately).
- **FR-08.5** On subscription change events, the tier cache is **DELETEd** (not SET) so the next request reads authoritative DB state and repopulates with a 60s TTL.
- **FR-08.6** VIES VAT validation is on the registration critical path but **fails open** to `vat_validation_status: pending` on SOAP timeout/5xx; reverse-charge applies only when `status: valid`.
- **FR-08.7** Usage metering uses an atomic Redis Lua script (`_USAGE_LUA`): GET + conditional INCR + EXPIRE in a single round trip. Counters reset at the start of each billing period.
- **FR-08.8** Stripe SDK calls (synchronous) are wrapped in `asyncio.to_thread()` from inside `async def` handlers.
- **FR-08.9** Invoice events (`invoice.payment_failed`, `customer.subscription.deleted`, `customer.subscription.updated`) drive tier changes; integration tests must fire mock webhooks end-to-end.
- **FR-08.10** Customer-facing billing portal: link to Stripe Customer Portal for invoice download, payment-method changes, cancellation.

### 4.10 Notifications, Alerts & Calendar (E09)

- **FR-09.1** Notification channels: email (transactional via SendGrid), in-app, Slack (E16), Teams (E16). Per-user channel preferences with per-event opt-out.
- **FR-09.2** Daily tender digest at a per-user configurable hour, summarising new matches scored ≥ threshold.
- **FR-09.3** Calendar sync to Google Calendar and Outlook (Pro/Enterprise only) via OAuth; tokens encrypted at rest with Fernet.
- **FR-09.4** iCal feed (read-only) is available on all paid tiers as a fallback.
- **FR-09.5** Inbound webhook validations (Slack, Teams) use ECDSA on the raw body, fail-closed.
- **FR-09.6** Notification dispatch is idempotent via DB constraint (`notification_id` UNIQUE) plus Redis SETNX in the consumer.
- **FR-09.7** Cross-user resource access through notification deep-links returns **404** (not 403) on mismatch, to prevent existence leakage.

### 4.11 Multi-Client / White-Label (E14)

- **FR-14.1** A consulting-firm parent company can manage N child workspaces, each isolated by `workspace_id` filter at the data layer.
- **FR-14.2** Branding (logo, colour, sender name) is per-workspace.
- **FR-14.3** Cross-workspace access requires explicit grant; no implicit visibility from parent to child or sibling.
- **FR-14.4** Billing rolls up to the parent; usage is tracked per workspace for cost allocation.

### 4.12 Frontend Shell & Design System (E03)

- **FR-03.1** Two Next.js 14 App Router apps in a Turborepo monorepo: `apps/client` (port 3000, end-user) and `apps/admin` (port 3001, internal).
- **FR-03.2** Shared `packages/ui` (shadcn/ui-based) and `packages/config` (ESLint, Prettier, TS).
- **FR-03.3** Data fetching: TanStack Query v5 wrapped in `<QueryGuard>` (handles loading, error, empty, populated). No raw `useQuery` in feature pages.
- **FR-03.4** Forms: `useZodForm` + `<FormField>` wrappers (React Hook Form + Zod schemas generated from backend OpenAPI).
- **FR-03.5** State: Zustand with namespaced persist keys (`eusolicit-client-auth-store`, `eusolicit-admin-auth-store`).
- **FR-03.6** i18n: `next-intl`; **all** user-visible strings keyed; `pnpm check:i18n` is a CI gate; ESLint `no-literal-text` rule enforced for `apps/client` and `apps/admin`.
- **FR-03.7** Rich text: TipTap (matches backend content-block schema).
- **FR-03.8** Frontend response types are generated via `openapi-typescript` codegen — manual duplication of Pydantic response shapes is forbidden.
- **FR-03.9** Design-system components (`<Select>`, `<Dialog>`, `<Sheet>`, `<Tabs>`, etc.) are mandatory — native HTML equivalents are forbidden in feature code.

### 4.13 Admin Platform (E12)

- **FR-12.1** Internal admin portal at port 3001, behind VPN/IP allowlist, separate `admin-api` service and `admin` schema.
- **FR-12.2** Tenant management: list companies, view/edit subscription tier, suspend/reactivate, view usage.
- **FR-12.3** Opportunity moderation: hide spam, fix mis-parsed CPV, re-trigger AI parser per opportunity.
- **FR-12.4** KPI dashboards (Recharts): MRR, active companies, AI cost ratio, SSE TTFB p95, crawler health.
- **FR-12.5** RBAC, regulatory framework editor, audit log viewer with structured filters.

### 4.14 Future-Phase Features (Post-MVP, listed for traceability)

- E15 Per-bid SKU pricing (Pro+ tier add-on).
- E16 Slack & Teams notifications (channel surfaces; partial in MVP).
- E17 CRM integrations (Salesforce, HubSpot, Pipedrive).
- E18 Trust Center / SOC 2 Type II readiness pack.
- E19 Outcome telemetry & renewal proof (tender result tracking).
- E20 NPS, reviews, onboarding tours.
- E21 99.9% SLA infrastructure (multi-region, Redis Sentinel/Cluster).

---

## 5. Non-Functional Requirements

NFR identifiers map to gates in TEA reviews and the test design.

### 5.1 Availability & Reliability

- **NFR-AV-1** Platform availability ≥ **99.5%** rolling 30 days at MVP / GA; ≥ 99.9% at E21 milestone.
- **NFR-AV-2** Crawler 24-hour ingestion freshness on AOP opportunities (95th percentile).
- **NFR-AV-3** Idempotency at every async boundary: Redis Streams consumers use DB UNIQUE + Redis SETNX dual-layer. No event is ever processed twice end-to-end.
- **NFR-AV-4** Long-running async state machines (generation, scanning, export, crawler-run) must have a Celery Beat cleanup task marking stuck rows `failed` after a configurable TTL. *Reference: `reset_stuck_proposals_task`.*
- **NFR-AV-5** Outbound calls implement the canonical `circuit_breaker(retry(http_factory))` pattern. 4xx responses must **not** increment the circuit-breaker failure counter.
- **NFR-AV-6** Redis-gated features fail **open** (UsageGate, tier-cache, rate-limiters); the request proceeds with a structured WARN log.
- **NFR-AV-7** External validation services on the registration path (VIES) fail **open** to a `pending` status — never block user registration.

### 5.2 Performance

- **NFR-PF-1** REST p95 latency ≤ **200 ms** at expected MVP load (10 concurrent users / company, ≤ 25 companies).
- **NFR-PF-2** SSE TTFB p95 ≤ **500 ms**; idle timeout 120 s; total timeout 600 s; heartbeat 15 s.
- **NFR-PF-3** PostgreSQL FTS queries on opportunities (10K+ rows) p95 ≤ 300 ms; budgeted by k6 baseline (mandatory artifact, not a backlog item).
- **NFR-PF-4** SSE concurrency cap ≤ 10 simultaneous generations per Kubernetes pod; verified by k6.
- **NFR-PF-5** Cycle-check on task-dependency graphs ≤ 50 ms p95 for graphs up to 500 tasks.

### 5.3 Security

- **NFR-SE-1** Passwords: bcrypt; explicit `max_length=128` Pydantic validator on every password field. Hashing runs in `run_in_executor`, never on the request thread.
- **NFR-SE-2** All inbound webhook signature validations use `hmac.compare_digest()` on raw bytes — never `==` on strings.
- **NFR-SE-3** Data at rest: AES-256 (managed Postgres + S3-compatible MinIO with SSE).
- **NFR-SE-4** Data in transit: TLS 1.3 minimum. Internal cluster traffic also TLS where the mesh permits.
- **NFR-SE-5** OAuth tokens (Google, Outlook, Slack, Teams) encrypted at rest with Fernet; key rotation procedure documented.
- **NFR-SE-6** `User.is_active` is checked on every protected endpoint **and** on every token refresh.
- **NFR-SE-7** Tier-gating uses FastAPI `Depends()` per route — never global middleware. Field-level Pydantic enforcement on tier-restricted response shapes.
- **NFR-SE-8** Cross-tenant access returns 404 (existence-leakage safe), not 403.
- **NFR-SE-9** Admin-API routes are behind a VPN / IP allowlist enforced by `test_ip_allowlist.py` middleware in addition to ingress-level controls.
- **NFR-SE-10** Content blocks sent to AI agents are sanitised against prompt-injection (allowlist of Markdown / TipTap nodes; remove script-like tokens) **before** persistence and **before** forwarding.
- **NFR-SE-11** Dependabot is configured for all language ecosystems (Python via `pyproject.toml`, Node via `pnpm`) and is a hard release gate.
- **NFR-SE-12** Error responses must never echo internal detail (no `str(exc)` in 5xx bodies; no connection strings in `/health`).

### 5.4 Data, Privacy & Compliance

- **NFR-DC-1** All customer data resides in EU regions (GDPR Article 44+). Documented in the Trust Center (E18).
- **NFR-DC-2** Schema isolation: six PostgreSQL schemas (`client`, `admin`, `pipeline`, `gateway`, `notification`, `shared`); each service role has CRUD on its own schema only; `migration_role` has DDL. **No cross-schema joins from application code.**
- **NFR-DC-3** Append-only `shared.audit_log` captures: actor, IP, before/after JSON, route, correlation-id, for every POST/PATCH/DELETE.
- **NFR-DC-4** GDPR data-subject rights: data export and erasure endpoints for each company; erasure tombstones audit rows but never deletes them.
- **NFR-DC-5** Data retention policy per resource type, enforced by Celery Beat retention sweeps.

### 5.5 Concurrency & Consistency

- **NFR-CC-1** Celery worker queues use `SELECT FOR UPDATE SKIP LOCKED` for database-driven queues to prevent duplicate processing across replicas.
- **NFR-CC-2** Optimistic locking via SHA-256 `content_hash` on collaborative document edits (proposals, content blocks). Mismatch → 409 with structured body and frontend conflict resolution.
- **NFR-CC-3** Atomic Lua scripts for usage metering, rate limiting, and any "read–test–write" Redis op (no GET-then-INCR sequences).
- **NFR-CC-4** Webhook deduplication uses DB UNIQUE constraint inside the processing transaction (not in-memory state, not Redis TTL).
- **NFR-CC-5** Multi-row migration safety: any downgrade that recreates a UNIQUE/PK constraint must include a regression test inserting ≥2 rows under the same grouping key before downgrading.

### 5.6 Observability

- **NFR-OB-1** `structlog` is the canonical logging library; configured with stdlib `LoggerFactory()` so pytest `caplog` works. JSON output in production, human-readable in dev.
- **NFR-OB-2** Prometheus `/metrics` is bootstrapped on every service. Mandatory metrics include: REST latency histogram, SSE TTFB histogram, AI Gateway error counter, billing webhook latency, tier distribution gauge, trial-to-paid counter, `billing_usage_sync_drift_total` gauge, crawler success/failure counter.
- **NFR-OB-3** Every request carries an `X-Correlation-ID` (generated if absent) propagated through Redis Streams events and into every log line.
- **NFR-OB-4** Structured introspection endpoints on every stateful service (e.g., `/internal/circuit-breaker/state`) — not exposed publicly, available on cluster IP only, and **must not** echo connection strings or hostnames in error bodies.
- **NFR-OB-5** k6 performance baselines are checked into `tests/load/` and executed on a release-candidate cadence; results committed to `load-test-results.md`.

### 5.7 Accessibility (UI)

- **NFR-AX-1** WCAG 2.1 AA: 4.5:1 contrast for body text; 3:1 for large text and UI component boundaries.
- **NFR-AX-2** Full keyboard navigability including the split-pane editor; visible focus rings (`:focus-visible`, 2px solid indigo, 2px offset).
- **NFR-AX-3** ARIA labels on interactive components; `aria-live` regions announce streaming AI updates and state changes ("Section locked by Deb").
- **NFR-AX-4** Status colour is always paired with an icon and text label.
- **NFR-AX-5** `prefers-reduced-motion` honoured: 0 ms transitions; static placeholders instead of animated loaders.

### 5.8 Internationalisation

- **NFR-I18-1** All user-visible strings are keyed via next-intl. Source language: English. Target launch languages: Bulgarian + English.
- **NFR-I18-2** `pnpm check:i18n` parity is a CI release gate; missing keys fail the build.
- **NFR-I18-3** Dates, numbers, and currencies render via `Intl.*` with the active locale.

### 5.9 Quality & Test Architecture

- **NFR-QA-1** TEA test review (≥80 / 100) is a blocking AC on each story's `review → done` transition (not a separate injected story).
- **NFR-QA-2** ATDD: every story with P0/P1 acceptance criteria has a RED-phase test list before development; ≥1 GREEN test per P0 AC before `done`.
- **NFR-QA-3** Frontend types are generated from backend OpenAPI (`openapi-typescript`); manual duplication is prohibited.
- **NFR-QA-4** Story completion gate: `done` requires (1) senior dev review APPROVED, (2) ATDD tests GREEN, (3) i18n parity, (4) no stub references for story-owned endpoints.
- **NFR-QA-5** Coverage: minimum 80% line + branch on services and packages.

---

## 6. User Stories & Acceptance Criteria

User stories are at **feature level**, grouped by epic. Each acceptance-criteria bullet is testable. Story IDs match the in-flight epics where applicable.

### Epic 1 — Workspace & Access Management (E02)

#### Story 1.1 — User Registration
*As a prospective customer, I want to register with email and password (or Google OAuth) so I can evaluate EU Solicit.*

**Acceptance Criteria**
- Email + password (max 128 chars), or Google OAuth ID token (RS256-verified).
- Password hashing via bcrypt in `run_in_executor`; never blocks the event loop.
- API returns 201 within 300 ms; Stripe customer provisioning runs as `BackgroundTask`.
- Email verification message dispatched on success; unverified user has read-only Free-tier access for 7 days then is locked.
- Cross-tenant access (e.g., GET another company's resource) returns **404**.

#### Story 1.2 — Login, Refresh, Logout
*As a user, I want secure session management.*

**Acceptance Criteria**
- RS256 JWT access token (15 min) + rotating refresh token (7 days).
- Refresh-reuse detection invalidates the entire token family.
- `User.is_active` checked on every refresh and protected request.
- Logout revokes the active refresh family.

#### Story 1.3 — Company Roles & Memberships
*As an admin, I want to invite teammates and assign roles.*

**Acceptance Criteria**
- Roles: admin, bid_manager, contributor, reviewer, read_only.
- Entity-level overrides (`entity_permissions`) can grant access above the role floor for a specific resource.
- Removing a user from a company revokes all entity overrides immediately.
- Audit row written for every role change.

#### Story 1.4 — Multi-Client Workspace (E14)
*As a consulting firm, I want to manage multiple isolated client workspaces under one parent company.*

**Acceptance Criteria**
- Parent company can create N child workspaces.
- Branding (logo, colour, sender) is per workspace.
- Cross-workspace queries are filtered by `workspace_id` at the data layer; cross-workspace access returns 404.
- Billing rolls up to the parent; usage is recorded per workspace.

### Epic 2 — Opportunity Data Pipeline (E05)

#### Story 2.1 — Daily AOP / TED Crawl
*As an Operator, I need new tenders ingested daily.*

**Acceptance Criteria**
- Crawler runs on Celery Beat schedule (configurable cron); writes `pipeline.crawler_runs` start/finish/status.
- Each crawl is idempotent: re-ingestion updates rows via DB UNIQUE constraint + Redis SETNX.
- Failed crawls retry 3× with exponential backoff (1m / 5m / 15m); persistent failure raises an alert.
- Crawler workers use `SELECT FOR UPDATE SKIP LOCKED` to prevent duplicate processing across replicas.
- Stuck-run cleanup task marks runs in `running` status >2× expected duration as `failed`.
- 95% of new AOP opportunities searchable in EU Solicit within 24h of public posting.

#### Story 2.2 — Document Download & Virus Scan
*As a Bid Manager, I expect every tender document to be safe and accessible.*

**Acceptance Criteria**
- Documents up to 100 MB / file, 500 MB / package downloaded to MinIO.
- ClamAV scan before persistence; rejected files surface a typed error.
- ClamAV timeout/outage handling: a Celery Beat task transitions documents stuck in `pending` to `failed` after configurable TTL.

#### Story 2.3 — Opportunity Normalisation & Emit
*As downstream services, we need clean structured opportunities on a stream.*

**Acceptance Criteria**
- CPV codes, deadline, budget, contracting authority, region, document set extracted into `pipeline.opportunities`.
- `opportunity.created` Redis Stream event emitted on each successful parse.
- Stream consumption is idempotent (DB UNIQUE + SETNX).

### Epic 3 — Frontend Shell & Design System (E03)

#### Story 3.1 — Turborepo Bootstrap
*As a developer, I want a monorepo with shared UI and config.*

**Acceptance Criteria**
- Two apps (`apps/client`, `apps/admin`); two shared packages (`packages/ui`, `packages/config`).
- `pnpm dev`, `pnpm build`, `pnpm lint`, `pnpm type-check` work via Turborepo pipeline.
- shadcn/ui base components installed in `packages/ui` and consumed by both apps.

#### Story 3.2 — QueryGuard, useZodForm, FormField
*As a feature dev, I want consistent fetching and forms.*

**Acceptance Criteria**
- All feature pages wrap remote state in `<QueryGuard>` (loading, error, empty, populated states).
- Forms use `useZodForm` + `<FormField>`; raw `useForm` is forbidden in feature code.
- `openapi-typescript` codegen run in CI; manual response-type duplication fails review.

#### Story 3.3 — i18n & no-literal-text Lint
*As a BG/EN customer, I want fully localised UI.*

**Acceptance Criteria**
- All user-visible strings keyed in next-intl.
- ESLint `no-literal-text` rule active for `apps/client` and `apps/admin`.
- `pnpm check:i18n` is a CI release gate; missing keys fail the build.

### Epic 4 — AI Gateway Service (E04)

#### Story 4.1 — Agent Registry & Resilient Calls
*As any internal service, I call KraftData only through the gateway.*

**Acceptance Criteria**
- `config/agents.yaml` maps logical names to KraftData UUIDs and runtime config.
- `circuit_breaker(retry(http_factory))` wraps every outbound call; 4xx does not increment breaker counter.
- Missing `X-Caller-Service` header returns 400.
- Two GREEN integration tests via `respx` for: happy path, 503 propagation as `{"message", "code": "AGENT_UNAVAILABLE"}` flat body.

#### Story 4.2 — Streaming Generation
*As a frontend, I receive SSE chunks reliably.*

**Acceptance Criteria**
- `run-stream` endpoint returns `StreamingResponse` with `Cache-Control: no-cache`, `X-Accel-Buffering: no`.
- 120 s idle timeout, 600 s total timeout, 15 s heartbeat.
- UsageGate decremented **before** the streaming body is opened.
- Generator is closed in `finally`; terminal event emitted on upstream EOF; semaphore permit released on timeout.
- Frontend uses native `fetch` + `ReadableStream` (Axios / EventSource forbidden); ATDD assertion enforces this at source level.

### Epic 5 — Document Analysis & Summarisation (E06 / E07)

#### Story 5.1 — Generate Executive Summary
*As a Reviewer, I want a one-page AI summary of a 200-page tender.*

**Acceptance Criteria**
- File upload validated for size (100 MB) and ClamAV scan.
- Frontend uses `<QueryGuard>` for fetch state.
- KraftData call carries `X-Caller-Service: client-api`; response streamed via SSE.
- Audit row written for the upload and for the summary completion.

#### Story 5.2 — Compliance Checklist Extraction
*As a Bid Manager, I need a structured requirements checklist.*

**Acceptance Criteria**
- Each checklist item links to its source span in the original document for verification.
- Items can be marked complete; completion is per proposal version.
- Free-tier shows top-5 items; Starter+ shows full list (field-level Pydantic enforcement).

### Epic 6 — Opportunity Discovery (E06)

#### Story 6.1 — Search & Filter
*As a user, I want to find tenders quickly.*

**Acceptance Criteria**
- PostgreSQL FTS query, filterable by CPV, region, budget, deadline, authority.
- Filter/sort/pagination state in URL (`useSearchParams()` + `router.replace()`); shareable via copy-paste.
- Cursor pagination; page-size capped at 100.
- Free-tier `OpportunityFreeResponse` returns exactly 6 fields.
- 404 (not 403) on cross-tenant ID.

#### Story 6.2 — Match Score & Inbox
*As a Bid Manager, I want a daily-curated inbox.*

**Acceptance Criteria**
- Per-company profile drives a 0–100 score per opportunity.
- "My matches" view shows score ≥ threshold, sorted by score desc then deadline asc.
- Inbox is paginated and URL-driven.

#### Story 6.3 — Opportunity Detail
*As a user, I want everything about a tender on one page.*

**Acceptance Criteria**
- AI executive summary, requirements checklist, risk highlights, similar past wins.
- Bid / No-Bid action available to bid_manager+; no-bid reason is captured.
- Skeleton loaders mirror the populated layout.

### Epic 7 — Proposal Generation (E07)

#### Story 7.1 — Stream a Proposal Draft
*As a Contributor, I want streaming AI drafting per content block.*

**Acceptance Criteria**
- Frontend uses native `fetch` + `ReadableStream`; Axios / EventSource not present in source.
- Backend `StreamingResponse` with required SSE headers; UsageGate decrement before generator yields.
- Streaming output appears in a Git-style diff view; explicit Accept / Reject required.
- Stuck generations beyond TTL are flipped to `failed` by `reset_stuck_proposals_task`.

#### Story 7.2 — Optimistic Locking on Save
*As a collaborator, I never want my edits silently overwritten.*

**Acceptance Criteria**
- Client sends `content_hash` (SHA-256) of the prior version on save.
- Backend `SELECT FOR UPDATE` + hash compare; mismatch returns **409** with structured body `{server_version, client_version, diff_url}`.
- Frontend opens a Conflict Resolution dialog showing 3-way diff.
- ≥1 GREEN integration test confirms the 409 path.

#### Story 7.3 — Section-Level Lock
*As a team, I want clear "who is editing what".*

**Acceptance Criteria**
- Locking a content block surfaces a "Locked by <name>" banner to other users.
- Lock auto-releases after 90 s idle or on save.
- Reads are never blocked.

#### Story 7.4 — Version History & Restore
*As a Bid Manager, I want to see and restore prior drafts.*

**Acceptance Criteria**
- Every Accept creates a new `proposal_versions` row.
- Diff between any two versions; restore creates a new version pointing to the chosen content.
- Audit row on restore.

#### Story 7.5 — Export PDF / DOCX
*As a Bid Manager, I want a submission-ready document.*

**Acceptance Criteria**
- Export tier-gated (Professional+).
- WeasyPrint (PDF) and python-docx (DOCX) wrapped in `ThreadPoolExecutor` (`run_in_executor`).
- Numeric thresholds (page size, margin) listed in the AC's "Implementation Constants" subsection.

### Epic 8 — Subscription Billing & Tier Gating (E08)

#### Story 8.1 — Stripe Customer Provisioning
*As a new user, I want signup to be instant even if Stripe is slow.*

**Acceptance Criteria**
- Signup returns 201 within 300 ms; Stripe customer created in `BackgroundTask`.
- `stripe_customer_id` nullable until provisioned; billing endpoints return 422 with structured error if absent.
- Stripe SDK calls wrapped in `asyncio.to_thread()`.

#### Story 8.2 — Upgrade / Downgrade
*As a Free user, I want to upgrade in one click.*

**Acceptance Criteria**
- Stripe Checkout session created; redirect URL returned.
- Webhook receipt validated with `hmac.compare_digest()` on raw body.
- `event_id` UNIQUE on `webhook_events` ensures idempotency inside the processing tx.
- Tier cache **DELETE** on `customer.subscription.updated`; next request repopulates from DB.

#### Story 8.3 — Usage Metering
*As an operator, I want fair-use enforcement that is race-free.*

**Acceptance Criteria**
- Atomic Lua script (`_USAGE_LUA`) does GET + conditional INCR + EXPIRE in one round trip.
- Concurrency test using testcontainers Redis (not fakeredis) — 10 K concurrent INCR with no over-count.
- Failure mode: Redis outage → fail-open with structured WARN log.

#### Story 8.4 — Webhook Failure Paths
*As an operator, I want failed-payment events to drive tier changes.*

**Acceptance Criteria**
- `invoice.payment_failed` → tier transitions to `past_due`; integration test fires mock event end-to-end.
- `customer.subscription.deleted` → downgrade to Free; tier cache DELETE.
- VIES validation fails open to `pending`; reverse-charge applies only on `valid`.

### Epic 9 — Notifications, Alerts, Calendar (E09 / E16)

#### Story 9.1 — Daily Tender Digest
*As a Bid Manager, I want a daily digest at my chosen hour.*

**Acceptance Criteria**
- Worker queries new opportunities scored ≥ user threshold.
- SendGrid email rendered from a localised template (BG/EN).
- Per-event opt-out honoured.
- Idempotent dispatch (`notification_id` UNIQUE + Redis SETNX).

#### Story 9.2 — Calendar Sync (Google / Outlook)
*As a Pro user, I want deadlines on my calendar.*

**Acceptance Criteria**
- OAuth flow stores tokens encrypted at rest with Fernet.
- Sync is bi-directional for the user's EU Solicit-managed calendar.
- iCal feed available on all paid tiers as a fallback.
- Token refresh has explicit `try / except / asyncio.CancelledError: raise`.

#### Story 9.3 — Slack / Teams Notifications
*As a team channel, I want bid updates posted in real time.*

**Acceptance Criteria**
- Outbound: signed payloads to Slack / Teams webhooks with retry+breaker.
- Inbound (interactivity): ECDSA-validated on raw body, fail-closed.
- Cross-user deep links return 404 on mismatch.

### Epic 10 — Collaboration, Tasks & Approvals

#### Story 10.1 — Tasks CRUD with Dependencies
*As a Bid Manager, I want a Gantt-like task structure.*

**Acceptance Criteria**
- POST/GET/PATCH/DELETE `/api/v1/tasks` with priority, status, due-date, assignee.
- Cycle detection (DFS) on dependency insert; cycles return 422 `"Cycle detected"`.
- `→ completed` rejected when finish_to_start upstreams not complete (422).
- Cross-tenant returns 404; soft-delete via `deleted_at`.

#### Story 10.2 — Task Templates
*As a Bid Manager, I want repeatable bid playbooks.*

**Acceptance Criteria**
- Template stages stored as JSONB array, each with `relative_days_before_deadline` and `dependency_indices[]` strictly less than its own index.
- Apply-template inserts N tasks + M dependencies in one DB tx; rollback on partial failure.
- Warning (non-blocking) when template `opportunity_type` mismatches the target.
- Audit row on apply.

#### Story 10.3 — Approval Workflow
*As a Bid Manager, I want sign-off before submission.*

**Acceptance Criteria**
- `draft → submitted` requires Reviewer approval if approval is configured.
- Approver receives in-app + email notification; can approve, reject, or request changes.
- Audit trail of all approval transitions.

### Epic 11 — Compliance & Grants

#### Story 11.1 — Run Compliance Check (ZOP)
*As a Bid Manager, I want to validate against my regulatory framework.*

**Acceptance Criteria**
- Framework selected from admin-managed JSONB rule set (`ZOP`, `EU-2014-24`, etc.).
- Each rule produces pass/fail/warning with severity and citation.
- Frontend renders progress rings + accordion list; failed mandatory items in red.
- Suggestions surface improvement actions.

#### Story 11.2 — ESPD Generator
*As a bidder, I want my ESPD auto-filled.*

**Acceptance Criteria**
- ESPD XML and PDF generated from company profile + opportunity reference.
- Wizard stepper used for fields requiring confirmation (progressive disclosure).
- Output download auditable.

#### Story 11.3 — EU Grant Budget Calculator
*As a grant applicant, I want my budget validated against EU rules.*

**Acceptance Criteria**
- Co-financing caps, eligibility windows, total caps validated.
- Float comparisons use `_ARITHMETIC_TOLERANCE = 0.01`.
- Optional fields use `null` (suppressed) vs `[]` (empty state) consistently.

### Epic 12 — Admin Platform

#### Story 12.1 — Tenant Management
*As an Operator, I manage companies and tiers.*

**Acceptance Criteria**
- List, suspend, reactivate; tier change via Stripe.
- VPN/IP allowlist enforced at FastAPI middleware.
- All actions audited under `admin` schema.

#### Story 12.2 — KPI Dashboards
*As a leader, I want live business metrics.*

**Acceptance Criteria**
- MRR, active companies, AI cost ratio, SSE TTFB p95, crawler health.
- Recharts charts with EN/BG labels.
- Data refreshed via scheduled materialised views.

### Epic 13 — Hardening & Drift Recovery (in flight)

#### Story 13.1 — Coordinator: Close Carry-Forwards
*As an engineering lead, I want zero rollover technical debt before Beta.*

**Acceptance Criteria**
- Dependabot configured for Python (`pyproject.toml`) + Node (`pnpm-workspace.yaml`).
- k6 baseline scripts written, executed, results committed to `load-test-results.md`.
- Prometheus `/metrics` bootstrapped on every service; mandatory metrics emitted.
- TEA review score ≥ 80/100 documented per closed story.
- All `inj-*` security stories closed.

---

## 7. Open Issues & Future Scope

### Open Issues (track in retrospectives until closed)

1. **Per-instance circuit breaker → Redis-backed** for multi-replica scaling (E21 dependency).
2. **k6 baseline freshness** — must run on each release candidate.
3. **TEA review on every story** — enforce as `done`-gate AC, not as separate story (do-not-regress lesson from E11).
4. **Frontend stub elimination** — `done` gate must verify no stub references for story-owned endpoints.
5. **Retro-action feedback loop** — orchestrator must create verification tasks for every `SEVERITY: critical` ACTION item and check at next retrospective before new items are accepted.

### Future Scope (Post-MVP)

- **E15** — Per-bid SKU pricing (Pro+ add-on).
- **E17** — CRM/ERP connectors (Salesforce, HubSpot, Pipedrive, SAP).
- **E18** — Trust Center / SOC 2 Type II readiness pack.
- **E19** — Outcome telemetry (tender result tracking, win/loss feedback loop into matching ML).
- **E20** — NPS, in-product reviews, onboarding tours.
- **E21** — 99.9% SLA infrastructure: multi-region active-active, Redis Sentinel/Cluster, cross-region Postgres replicas, RTO ≤ 15 min / RPO ≤ 5 min.
- **Direct e-procurement submission** — AOP electronic, TED eSender, CAIS.
- **Qualified electronic signatures (QES)** — eIDAS-compliant signing.

---

## 8. Glossary

| Term | Meaning |
|---|---|
| **AOP** | Bulgarian Public Procurement Agency portal |
| **TED** | Tenders Electronic Daily — EU procurement portal |
| **ZOP** | *Zakon za Obshtestvenite Porachki* — Bulgarian Public Procurement Act |
| **ESPD** | European Single Procurement Document |
| **SSE** | Server-Sent Events |
| **RBAC** | Role-Based Access Control |
| **DAG** | Directed Acyclic Graph (task dependencies) |
| **TTFB** | Time-To-First-Byte |
| **TEA** | Test Engineer Architect (BMAD role) |
| **ATDD** | Acceptance-Test-Driven Development |
| **VIES** | EU VAT Information Exchange System |
| **KraftData** | Vendor-supplied Agentic AI platform powering all AI features |

---

## 9. Document Control

| Version | Date | Author | Notes |
|---|---|---|---|
| 1.0 | 2026-04-26 (earlier) | Deb (Agentic Workflow) | Initial approved PRD |
| 2.0 | 2026-04-26 | Deb (Agentic Workflow) | Canonical refresh consolidating brief drift, gap-resolution requirements, architecture decisions, UX spec, and patterns/anti-patterns from epics shipped to date. Supersedes v1.0. |
