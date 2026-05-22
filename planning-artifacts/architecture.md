---
title: "EU Solicit — Architecture Document"
status: "complete"
date: "2026-04-27"
lastValidated: "2026-05-14"
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
workflowType: 'architecture'
lastStep: 8
completedAt: '2026-04-27'
version: "v3.0"
inputDocuments:
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/ux-spec.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/project-context.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/project-context.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture-evaluation-2026-04-25.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-agenticsai.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-agenticsai.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md"
revalidationInputs:
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-04-30.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-03.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-04.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/implementation-readiness-report-2026-05-03.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-12-agenticsai.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/sprint-change-proposal-2026-05-13-pe-orphan-cleanup.md"
revalidationOutcome: "v3.0 — folds in 2026-05-12 AgenticSAI integration pivot (ADRs 018/019/020 + ADR-004 addendum + topology rewrite + new tables + run-state reconciler). v2.0 archived to architecture.v2.0.bak.md. Foundational invariants unchanged (schema isolation, two-layer resilience, per-route Depends(), SSE lifecycle, fire-and-forget audit, event-bus discipline)."
---

# EU Solicit — Architecture Document

**Author:** Winston (System Architect)
**Date:** 2026-04-27 (v2.0); 2026-05-14 (v3.0 consolidation)
**Status:** Living document — updated through epic retrospectives. **v3.0** consolidates the 2026-05-12 AgenticSAI integration pivot (ADRs 018/019/020) into v2.0; the prior v2.0 is preserved at `architecture.v2.0.bak.md`. v2.0 (2026-04-27) superseded v1.x and incorporated the 2026-04-25 PRD amendment (multi-client workspaces, per-bid SKU, Pro+ tier, `integrations-api`, Trust Center, 99.9% SLA, outcome telemetry). v3.0 reshapes the AI/agent surface from a thin AgenticSAI proxy to a **AgenticSAI-as-substrate** topology with per-tenant Project, shared org-scoped N8N, and split canonical-data ownership (structured in EU Solicit Postgres, unstructured in AgenticSAI storage-resources).

---

## 1. System Overview

EU Solicit is a multi-tenant SaaS platform that automates the lifecycle of EU public procurement and EU grant applications: opportunity discovery, bid/no-bid decisioning, AI-assisted proposal drafting, compliance checking, scoring simulation, collaborative editing, ESPD generation, and submission export.

### 1.1 Architectural Style

- **Domain-driven microservices** behind a single public API surface, sharing one PostgreSQL instance with **schema-per-service logical isolation**.
- **Event-driven asynchrony** via Redis Streams (custom `EventPublisher`/`EventConsumer` abstraction with at-least-once delivery and DLQ).
- **Human-in-the-loop AI orchestration with AgenticSAI as the agentic substrate** (per ADR-018, 2026-05-12): a per-tenant AgenticSAI Project owns agent definitions, runs, traces, memories, and the knowledge base; **`agenticsai-gateway`** brokers EU Solicit ↔ AgenticSAI calls, holds tenant↔Project mapping, hosts the Standard Webhooks receiver, and reconciles run state. **N8N** (shared org-scoped, provisioned by AgenticSAI for the EU Solicit Organisation) orchestrates multi-step workflows with agents as native steps. The user always reviews/approves AI output.
- **Multi-tenant by design**: every tenant-bound row carries `company_id` (tenant root) and (post-Epic 14) `workspace_id` (sub-tenant for client-engagement isolation by consulting firms).
- **Two-substrate canonical-data model** (per ADR-019, 2026-05-12): structured records (opportunities, proposals, billing, RBAC) canonical in EU Solicit Postgres; unstructured artefacts (tenders, profiles, past proposals, ESPD templates, rubrics) canonical in AgenticSAI storage-resources per tenant Project. Tenant operations (provisioning, archive, Right-to-Erasure, export) traverse both substrates with explicit cross-substrate completion proof in `shared.audit_log`.
- **Strict separation of customer-facing and internal admin surfaces**: a separate `admin-api` + `apps/admin` deployment, IP-allowlisted and MFA-gated.
- **Resilient outbound integration**: every external HTTP call is wrapped in `circuit_breaker(retry(http_factory))` with explicit `httpx` timeouts (see ADR-004 + 2026-05-12 AgenticSAI addendum).
- **Observability-first**: structlog JSON → Loki, Prometheus metrics, Jaeger traces. Every service exports `/metrics`, `/health`, and `/admin/*` introspection endpoints.

### 1.2 Architectural Drivers (from PRD)

| Driver | Source | Implication |
|---|---|---|
| AI generation TTFB < 500ms (NFR-2) | PRD §NFR | SSE streaming, single-point error sanitization, pre-generation quota check before HTTP 200 headers |
| API p95 < 200ms (NFR-1) | PRD §NFR | Async I/O end-to-end, Redis cache for tier policy, fire-and-forget audit writes |
| Launch posture: **best-effort availability** per ADR-010 (2026-05-11). RTO ≤ 4h / RPO ≤ 24h. Original NFR-14 targets (99.5% MVP / 99.9% post-amendment) **deferred to a future Phase-2 HA-migration epic**; no public SLA promised at launch. | PRD §NFR + amendment + ADR-010 | Single Docker host on `www1.endigitalx.com`; `pg_basebackup` + WAL archiving + Hetzner Storage Box off-site replication (Epic 22 onprem-01); Redis AOF + RDB persistence (Epic 22 onprem-02); `restart: unless-stopped` + healthchecks; Trust Center "Service is in beta. Best-effort availability." disclosure (E23). HPA/PDB/Multi-AZ/Sentinel removed in pivot — see ADR-010 §Removed. |
| EU-only data residency (Domain-Specific) | PRD §Domain | On-prem single-host `www1.endigitalx.com` (per ADR-010); AgenticSAI EU-region storage resources only for the EU Solicit Organisation — **contractually confirmed** as launch-blocking due-diligence (see §11.3 risk #1). |
| Zero cross-tenant leakage (NFR-7) | PRD §NFR | DB schema isolation + row-level `company_id`/`workspace_id` scoping + RBAC `Depends()` factories + cross-tenant negative tests as story ACs |
| GDPR + ZOP + WCAG 2.1 AA | PRD §Domain | Immutable audit log, encryption at rest (AES-256) and in transit (TLS 1.3), right-to-erasure flow, accessibility-first UI primitives |
| 10K active companies / 1M opportunities at <20% degradation (NFR-13) | PRD §NFR | Horizontal scaling, read replicas, materialized views with `REFRESH CONCURRENTLY`, atomic Redis Lua for usage metering |

---

## 2. Component Diagram

### 2.1 High-Level Topology (text/ASCII)

```
                                            ┌──────────────────────────┐
                                            │       INTERNET            │
                                            └─┬────────┬────────┬───────┘
                                              │        │        │
                                          (CloudFlare CDN — Trust Center only)
                                              │        │        │
                              ┌───────────────┼────────┼────────┼───────────┐
                              │               │        │        │           │
                              ▼               ▼        ▼        ▼           ▼
                        ┌─────────────┐  ┌─────────┐  ┌────────────┐  ┌────────────┐
                        │ apps/client │  │apps/    │  │ /trust     │  │  Stripe    │
                        │ Next.js 14  │  │admin    │  │ (public    │  │  webhooks  │
                        │ :3000       │  │Next.js  │  │  static)   │  │ (HMAC)     │
                        └──────┬──────┘  │:3001    │  └────────────┘  └─────┬──────┘
                               │         │ (VPN +  │                        │
                               │         │ IP-AL)  │                        │
                               │         └────┬────┘                        │
                               │              │                             │
                  ┌────────────┴──────────────────────────────────┐         │
                  │      host nginx on www1 (TLS 1.3 + certbot)   │         │
                  └──┬────────┬─────────┬────────┬────────┬───────┘         │
                     │        │         │        │        │                 │
                     ▼        ▼         ▼        ▼        ▼                 │
              ┌──────────┐┌────────┐┌───────────┐┌──────┐┌──────────┐      │
              │client-api││admin-  ││agenticsai-   ││data- ││integ-    │      │
              │  :8001   ││api     ││gateway    ││pipe- ││rations-  │◄─────┤
              │ FastAPI  ││:8002   ││:8004      ││line  ││api :8007 │      │
              │ + Stripe │└────────┘│ (renamed) ││:8003 │└────┬─────┘      │
              │ + RBAC   │          │ tenant→   ││(thin,│     │            │
              │ + Tier   │          │  Project  ││ recvr│     │            │
              │ + tenant │          │  mapping  ││ only)│     │            │
              │  provis. │          │  + WH recv│└──┬───┘     │            │
              └────┬─────┘          │  + reconc.│   │         │            │
                   │                └─────┬─────┘   │         │            │
                   │                      │         │         │            │
                   │                      ▼         ▼         ▼            │
                   │       ╔═══════════════════════════════════════════╗  │
                   │       ║              AgenticSAI (EXTERNAL)            ║  │
                   │       ║      https://agenticsai.endigitalx.com/   ║  │
                   │       ║                                            ║  │
                   │       ║  ┌──────────────────────────────────────┐ ║  │
                   │       ║  │ EU Solicit Organisation (singleton)  │ ║  │
                   │       ║  │ ┌──────────────────────────────────┐ │ ║  │
                   │       ║  │ │ Project per EU Solicit company   │ │ ║  │
                   │       ║  │ │  - Agents (ESPD, qual, quant…)   │ │ ║  │
                   │       ║  │ │  - Teams + Workflows             │ │ ║  │
                   │       ║  │ │  - Storage-resources (KB) ◄──────┼─┼─║──┐ artefact uploads
                   │       ║  │ │  - MCP servers (Dynamics,HubSpot)│ │ ║  │ from client-api
                   │       ║  │ │  - Traces, memories, eval-runs   │ │ ║  │
                   │       ║  │ │  - Per-Project api-key (Fernet)  │ │ ║  │
                   │       ║  │ └──────────────────────────────────┘ │ ║  │
                   │       ║  │  ┌────────────────────────────────┐  │ ║  │
                   │       ║  │  │ Shared N8N (org-scoped)        │  │ ║  │
                   │       ║  │  │  Workflow templates by         │  │ ║  │
                   │       ║  │  │  projectId — AOP/TED/EUGrants/ │  │ ║  │
                   │       ║  │  │  Qualification/Quantification/ │  │ ║  │
                   │       ║  │  │  CRM-enrichment                │  │ ║  │
                   │       ║  │  └────────────────────────────────┘  │ ║  │
                   │       ║  └──────────────────────────────────────┘ ║  │
                   │       ║                                            ║  │
                   │       ║  Webhooks (Standard Webhooks spec):       ║  │
                   │       ║   workflow.completed                       ║  │
                   │       ║   agent.run.completed                      ║──┴──► HTTPS in to
                   │       ║   storage.file.processed                   ║      agenticsai-gateway
                   │       ║   policy.violation                         ║      /webhooks/agenticsai
                   │       ╚═══════════════════════════════════════════╝
                   │
                   │                                  ┌────────────┐
                   │                                  │ Microsoft  │
                   │                                  │ Dynamics + │◄── via MCP from
                   │                                  │ HubSpot    │    AgenticSAI agents
                   │                                  │ (CRM)      │    (Pipedrive/SF
                   │                                  └────────────┘     deferred)
                   │
                   ▼
       ┌──────────────────────┐         ┌──────────────────────────┐
       │  notification :8005  │◄────────┤ Redis Streams            │
       │  Email / Slack /     │         │ (events)                 │
       │  Teams / SP-change / │         │ + DLQ                    │
       │  onboarding alerts   │         └──────────┬───────────────┘
       └──────────┬───────────┘                    │
                  │                                │
                  │   ┌────────────────────────────┴─────────────────┐
                  │   │              Redis 7 (single container)       │◄──┘
                  │   │  - Caching (tier policy, RBAC, session)        │
                  │   │  - Rate limiting + atomic usage Lua            │
                  │   │  - Event Bus (Streams) + DLQ                   │
                  │   │  - Idempotency keys (SETNX) — webhook dedup    │
                  │   └────────────────────────────────────────────────┘
                  │
                  ▼
        ┌─────────────────────────────────────────────────────────────────┐
        │   PostgreSQL 16 (single container on www1, daily Hetzner BU)    │
        │  Schemas (logical isolation; cross-schema queries forbidden):   │
        │  ┌──────────┬──────────┬──────────┬─────────┬────────────────┐  │
        │  │  client  │  admin   │ pipeline │ gateway │  notification  │  │
        │  ├──────────┴──────────┴──────────┴─────────┴────────────────┤  │
        │  │  integrations                                              │  │
        │  ├───────────────────────────────────────────────────────────┤  │
        │  │  shared (audit_log, lookups)                               │  │
        │  └───────────────────────────────────────────────────────────┘  │
        │  Roles: client_api_role, admin_api_role, gateway_role,           │
        │         pipeline_role, notification_role, integrations_role,     │
        │         migration_role (DDL only)                                │
        │                                                                  │
        │  NEW (per 2026-05-12 AgenticSAI pivot):                             │
        │   client.agenticsai_projects        (tenant↔Project mapping)       │
        │   client.agenticsai_kb_files        (artefact metadata + SHA256)   │
        │   client.agenticsai_mcp_servers     (MCP registration state)       │
        │   gateway.webhook_subscriptions  (Standard Webhooks subs)       │
        │   gateway.workflow_runs          (run lifecycle + reconciler)   │
        └─────────────────────────────────────────────────────────────────┘

        ┌─────────────────────────────────────────────────────────────────┐
        │   Local object storage on www1 (Trust Center artefacts)         │
        │   - Tender raw downloads cached briefly before upload to        │
        │     AgenticSAI storage-resources                                   │
        │   - Trust Center artefacts (signed URLs)                        │
        │   NOTE: S3/MinIO no longer canonical for tender/proposal        │
        │   content (per ADR-019 — AgenticSAI KB is canonical)               │
        └─────────────────────────────────────────────────────────────────┘
```

### 2.2 Service Inventory

| Service | Port | Schema(s) owned | Purpose |
|---|---|---|---|
| `client-api` | 8001 | `client` (rw), `pipeline` (read-only via dual-session) | Primary user-facing API: auth, RBAC, workspaces, opportunities (read), proposals, ESPD, billing, calendar, analytics, per-bid metering |
| `admin-api` | 8002 | `admin` (rw), `client` (read-only) | Internal admin portal API: tenant management, compliance frameworks, crawler oversight, pricing-tier configuration |
| `data-pipeline` | 8003 | `pipeline` (rw) | **Re-scoped (2026-05-12).** Webhook event router + opportunity normaliser + canonical `pipeline.opportunities` writer. Crawler responsibility (AOP/TED/EUGrants) moves to N8N workflow templates calling AgenticSAI crawler agents; per-pipeline `ai_gateway_client` retry module retired. Retains: `crawler_runs` history (read-only), `opportunities.ingested` Redis Streams publication |
| `agenticsai-gateway` | 8004 | `gateway` (rw) | Single broker for AgenticSAI. Tenant↔Project mapping cache (Fernet-encrypted per-Project api-keys), Standard Webhooks receiver, async-run + job-poll, run-status reconciler (5-min cadence), SSE proxy for streaming agent runs. **Retires (2026-05-12):** `agents.yaml` logical-name registry, hard-coded AgenticSAI routes |
| `notification` | 8005 | `notification` (rw) | Redis Streams consumer. Email/in-app/Slack/Teams delivery, sub-processor change DPAs, onboarding-stall alerts, materialized-view refresh orchestration |
| `integrations-api` | 8007 | `integrations` (rw), `client.crm_connections` (read-only) | **Re-scoped (2026-05-12).** OAuth callback hosting for Dynamics 365 + HubSpot; Fernet token storage in `client.crm_connections`; MCP-server registration + token-injection to AgenticSAI; token rotation Celery Beat. Slack/Teams incoming-webhook templates **retained**. **Retires:** HubSpot/Pipedrive/Salesforce direct HTTP adapters, per-provider sync Celery queues for CRM (Slack/Teams Celery queues retained) |
| `enterprise-api` | (gateway) | proxied | Public REST API for Enterprise tier. Routes through `client-api` with `X-API-Key` auth and stricter rate limits |

**Frontend apps** (`frontend/apps/`):

| App | Port | Audience |
|---|---|---|
| `apps/client` | 3000 | End users — Elena, Maria, contributors, reviewers, tenant admins |
| `apps/admin` | 3001 | Internal EU Solicit operators — IP-allowlisted, MFA-required |

**Frontend shared packages** (`frontend/packages/`):

- `packages/ui` — shadcn/ui-based design system (`<Select>`, `<Dialog>`, `<Sheet>`, `<Tabs>`, `<FormField>`, `<QueryGuard>`, etc.). Native HTML controls are forbidden where a `packages/ui` equivalent exists (Epic 11 anti-pattern).
- `packages/config` — shared ESLint/Prettier/TypeScript config.

**Backend shared packages** (`packages/`):

- `eusolicit-common` — `BaseServiceSettings`, structlog setup, exception handlers, middleware (request ID, tenant scope, IP allowlist), `EventBus` (Redis Streams).
- `eusolicit-models` — Cross-service Pydantic DTOs, `StrEnum` enums, Redis Stream event schemas.
- `eusolicit-agenticsai` — Typed Python client for AgenticSAI Org/Project/Agent/Team/Workflow/Storage/MCP/Webhook APIs. Generated from `eusolicit-docs/agenticsai-reference-docs/api-docs v3.json`. Pins AgenticSAI base URL via env (`AGENTICSAI_BASE_URL`). Boring-tech directive: thin generated client + a handful of hand-written convenience wrappers — no business logic in the client package.
- `eusolicit-test-utils` — Canonical test fixtures (`UserFactory`, `register_and_verify_with_role`, `create_company_pair`, `ServiceClient`, RBAC helpers). **Mandatory** — bespoke fixture rebuilds are a BLOCKING code review finding (project-context Epic 14).

---

## 3. Technology Choices & Rationale

### 3.1 Backend

| Technology | Version | Role | Rationale |
|---|---|---|---|
| **Python** | 3.12 | Backend language | Native asyncio improvements (PEP 692, exception groups); `StrEnum`; broad AI/ML library ecosystem; team fluency |
| **FastAPI** | latest stable | HTTP framework | Async-first, dependency injection (`Depends()` for tier gates and RBAC), auto-generated OpenAPI for codegen, mature SSE support |
| **SQLAlchemy 2.x async** | 2.x | ORM | Async I/O, schema-aware MetaData, clean dual-session pattern for cross-schema reads (`get_pipeline_readonly_session` for read, `get_db_session` for write — Epic 6 pattern) |
| **Alembic** | latest | Migrations | Each service ships its own migration tree; `migration_role` has DDL rights; partial-unique WHERE predicates supported (Epic 14 pattern) |
| **Celery** | 5.x | Background tasks (data-pipeline, notification, integrations-api) | Beat scheduling for crawlers, OAuth refresh, materialized-view refresh, stuck-state cleanup; **Always use `asyncio.new_event_loop()` in `try/finally loop.close()`** — never `asyncio.get_event_loop()` (Python 3.12 deprecated) or `asyncio.run()` (fails with a running loop) — Epic 13 anti-pattern |
| **structlog** | latest | Structured logging | JSON output to stdout → Loki; **session-scoped autouse `configure_structlog_stdlib` fixture** required in every service's `conftest.py` so pytest `caplog` captures records (Epic 13 pattern) |
| **httpx** | latest | Outbound HTTP | Async; explicit timeouts mandatory (CLAUDE.md rule); composes with circuit breaker + retry |
| **Pydantic 2** | 2.x | Schema validation | Strict typing on requests/responses; `Literal[...]` / `StrEnum` mandatory for status fields (Epic 13 anti-pattern: bare `str` permits unknown values) |
| **WeasyPrint + python-docx** | latest | Document export | PDF (proposals, Outcome Briefs, Trust Center artefacts) and DOCX export. **Must run inside `loop.run_in_executor(executor, fn, *args)`** — CPU-bound; event-loop blocking is invisible in unit tests, only manifests under load (Epic 7 pattern) |
| **Tiptap** (frontend) + ProseMirror | latest | Rich-text editor | Block-based proposal editor; supports collaborative cursors (post-MVP); content-hash optimistic locking on save (Epic 7 pattern) |
| **`asyncio.to_thread()`** | stdlib | Sync-SDK adapter | Mandatory for synchronous SDKs in `async def` handlers (Stripe Python SDK, VIES SOAP, ClamAV pyclamd) — Epic 8 pattern |

### 3.2 Frontend

| Technology | Version | Role | Rationale |
|---|---|---|---|
| **Next.js** | 14 (App Router) | React meta-framework | RSC, route groups, middleware for dual-layer auth guard (Epic 3 pattern); `/[locale]/workspace/[workspaceId]/...` routing for Epic 14 |
| **TypeScript strict** | 5.x | Language | Required by NFR-21; `tsc` is a CI gate |
| **TanStack Query** | v5 | Server state | Wrapped in `<QueryGuard>` for consistent loading/error/empty UI (Epic 3 pattern); query keys are explicit cross-story constraints (Epic 6 anti-pattern) |
| **Zustand** | latest | Global client state | Namespaced persist keys (`eusolicit-client-auth-store-v2`, `eusolicit-admin-auth-store`); migration shim reads v1 → writes v2 → deletes v1 (Epic 14) |
| **React Hook Form + Zod** | latest | Forms | `useZodForm` + `<FormField>` wrappers; **Retry buttons MUST route through `form.handleSubmit()`** — direct `.mutate()` bypasses validation (Epic 11 anti-pattern) |
| **shadcn/ui + Tailwind + Radix** | latest | Design system | Accessibility-first (WCAG 2.1 AA per NFR-18); themeable via CSS variables; **native `<select>` etc. forbidden — must use `<Select>` from `@eusolicit/ui`** (Epic 11) |
| **next-intl** | latest | i18n (BG/EN MVP) | `pnpm check:i18n` is a CI gate; `no-literal-text` ESLint rule for `apps/client` and `apps/admin` (Epic 11 anti-pattern: hardcoded English in fix passes) |
| **openapi-typescript** | latest | Type codegen | **Frontend response types MUST be codegen'd from backend OpenAPI** — manual duplication is a BLOCKING review finding (Epic 7 anti-pattern) |
| **Vitest** | latest | Frontend unit tests | Source-inspection ATDD scales to multi-panel components without runtime rendering; RTL/JSDOM reserved for user interaction flows |
| **Playwright** | latest | E2E | Multi-browser; **E2E specs with ≥80% `test.skip` are NONE coverage, not PARTIAL** — activation is a blocking sub-task (Epic 8 anti-pattern) |
| **Turborepo + pnpm** | latest | Monorepo build | Cached builds across `apps/`, `packages/`; `pnpm dev` starts both Next.js apps |

### 3.3 Data & Messaging

| Technology | Version | Role | Rationale |
|---|---|---|---|
| **PostgreSQL** | 16 | Primary RDBMS | Schemas as logical isolation boundary; JSONB for AI scorecards, content blocks; GIN indexes for full-text search; `REFRESH MATERIALIZED VIEW CONCURRENTLY` for outcome telemetry; `SELECT FOR UPDATE` for pessimistic locks; UNIQUE WHERE predicates for soft-delete tables |
| **Redis** | 7 | Cache + event bus + rate limiter | Streams (`XADD`/`XREADGROUP`) for at-least-once async messaging; atomic Lua scripts for usage metering (`_USAGE_LUA` — Epic 6 mandatory pattern); SETNX for idempotency keys; `tier_cache` with **DELETE-on-change** (not SET — Epic 8 pattern) |
| **Amazon S3** (eu-central-1) | — | Object storage | Tender documents, exports, Trust Center artefacts. ClamAV scan before allowing read (`pyclamd`, async-wrapped); signed URLs for time-limited access |
| **MinIO** (local dev) | latest | S3-compatible local object store | Mirrors S3 API for `make up` local dev; identical code path |
| **ClamAV** (clamd) | 1.x | Virus scanning | All user-uploaded documents; ClamAV timeout-to-failed transition required (Epic 6 anti-pattern: stuck `pending` documents) |
| **testcontainers (Postgres + Redis)** | latest | Integration test infra | Mandatory pair with `respx` for backend integration tests (Epic 4 standard) — never fakeredis where atomicity is asserted |

### 3.4 AI Platform (rewritten 2026-05-12 per ADR-018)

| Component | Choice | Rationale |
|---|---|---|
| **AgenticSAI agentic substrate** | external managed at `https://agenticsai.endigitalx.com/` | Multi-agent runtime (per-Project): agents, teams, workflows, vector storage-resources (KB), traces, memories, eval-runs, policies. Per ADR-018 |
| **Tenancy mapping** | Singleton EU Solicit Org + Project-per-company (Topology A) | ADR-018. `client.agenticsai_projects` table holds (`company_id ↔ agenticsai_project_id`, Fernet-encrypted api-key, n8n-subdomain, agent_map JSONB) |
| **`agenticsai-gateway` abstraction** | in-house FastAPI service | Single broker per ADR-018: tenant↔Project mapping cache, per-Project api-key vault, Standard Webhooks receiver, run-state reconciler, SSE proxy. Logical agent names resolved per `(eusolicit_logical_name, agenticsai_project_id)` from `agent_map` |
| **Calling conventions** | `AgenticsaiGatewayClient.run_agent(logical_name, payload, company_id)` (sync), `run_agent_stream(...)` (SSE), `submit_agent_run_async(...)` + `get_job_status(job_id)` (async-poll) | Frozen contract. Flat 503 body `{"message", "code": "AGENT_UNAVAILABLE"}` retained (Epic 11 standard). `X-Caller-Service` header retained. **No client-side retry**, gateway owns retry/backoff per ADR-004 |
| **N8N orchestration** | shared org-scoped (one instance for EU Solicit Org) | Workflow templates parameterised by `projectId`. Per-template versioning (semver) + staged rollout (canary tenant → 10% → 100%) per ADR-018 *Consequences*. Templates are production code: PR review + rollback plan |
| **Knowledge Base** | AgenticSAI storage-resources per Project | Per ADR-019. Artefacts canonical in AgenticSAI; EU Solicit holds `(agenticsai_storage_resource_id, agenticsai_file_id, sha256)` metadata only |
| **CRM tooling** | AgenticSAI MCP servers per Project (Dynamics + HubSpot v1) | Per ADR-020. OAuth tokens custodied in EU Solicit Fernet vault, injected into MCP-server config at registration + rotation |
| **Webhooks (AgenticSAI → EU Solicit)** | Standard Webhooks specification | Subscribed event types: `workflow.completed`, `agent.run.completed`, `storage.file.processed`, `policy.violation`. HMAC SHA-256 verified via `hmac.compare_digest()`. 7-day Redis idempotency cache. DLQ for poison events |
| **Run-state reconciliation** | scheduled job in `agenticsai-gateway`, 5-min cadence | Polls `GET /jobs/{jobId}/status` for non-terminal rows in `gateway.workflow_runs` and converges to terminal. **Reconciler is authoritative**; webhooks are latency optimisation. AgenticSAI is the second critical external dependency (after Stripe) — silent run loss is unacceptable |
| **Streaming (SSE)** | unchanged per ADR-005 | SSE lifecycle invariants (quota check before `StreamingResponse`, generator closed in finally, terminal event, fresh `session_factory`, single-point error sanitization, `except asyncio.CancelledError: raise`) all retained. Upstream URL shifts to AgenticSAI `/agents/{id}/run/stream` |

#### 3.4.1 N8N workflow template source-of-truth

N8N workflow templates are **canonical in EU Solicit's Git repo** at `eusolicit-app/infra/n8n-templates/<workflow-name>-v<semver>.json` (exported JSON from AgenticSAI N8N). Templates are versioned (semver-tagged) and PR-reviewed like production code. Deploy mechanism: a deploy-time script (`scripts/sync_n8n_templates.py`) diffs committed JSON against the AgenticSAI N8N instance via the AgenticSAI N8N API and applies updates; refuses to deploy on drift unless `--force` flag is set (with audit log). Rollback: revert the commit + re-run the script to restore the previous template version. Per-tenant feature flag (`enable_n8n_workflow_<source>_<version>`) gates which version a tenant runs against (per E05 amendment).

### 3.5 Auth & Security

| Component | Choice | Rationale |
|---|---|---|
| **JWT** | RS256 (asymmetric) | Stateless inter-service auth; short-lived access (15 min) + rotating refresh tokens stored as httpOnly secure SameSite=lax cookies (NFR-5) |
| **Google OAuth** | OAuth 2.0 PKCE | FR-2; single-button registration |
| **Magic-link external collaborators** | scoped JWT, 30-day expiry, one-shot | Epic 14. `accepted_at IS NOT NULL → 410 Gone` semantics; claims omit `company_id` and `subscription_tier` so `get_current_user` rejects with 401 (claim-shape isolation, Epic 14 pattern) |
| **MFA (TOTP)** | mandatory for `apps/admin` | NFR-8; admin portal also IP-allowlisted via middleware |
| **Webhook signature verification** | HMAC SHA-256 + `hmac.compare_digest()` | CLAUDE.md security rule; Stripe uses ECDSA on raw body, fail-closed (Epic 9) |
| **Secret encryption at rest** | Fernet (symmetric) | OAuth tokens, third-party API keys; canonical module per Epic 9 |
| **Field-level data encryption** | AES-256 (libsodium) | Sensitive PII columns above and beyond at-rest disk encryption |

---

## 4. Data Model

### 4.1 Schema Layout

One PostgreSQL 16 instance, six service schemas + `shared` + (post-Epic 17) `integrations`. Each schema is owned by a single service role with CRUD; `migration_role` owns DDL. Application code **must not** issue cross-schema queries — cross-schema reads use the dual-session pattern (`MetaData(schema=...)` per schema, separate read-only session in the consuming service).

| Schema | Owning service | Purpose |
|---|---|---|
| `client` | `client-api` | Tenant root. Companies, users, workspaces, memberships, RBAC, proposals, opportunities (denormalized snapshot from pipeline), documents, content blocks, billing, subscriptions, add-ons, ESPDs, calendar tokens, CRM connections (token vault) |
| `admin` | `admin-api` | Compliance frameworks, pricing-tier policies, internal user accounts, system flags |
| `pipeline` | `data-pipeline` | Source procurement records (AOP, TED), enrichment queue, crawler runs, raw documents pre-ingestion |
| `gateway` | `agenticsai-gateway` | Agent execution logs, prompt template versions, evaluation scorecards, rate-limit & circuit-breaker state mirrors, AgenticSAI webhook subscriptions, workflow-run reconciliation state |
| `notification` | `notification` | Email templates, delivery records, in-app notification feed, materialized views for analytics, webhook dispatch logs |
| `integrations` (NEW) | `integrations-api` | CRM sync logs, conflict logs, webhook subscriptions; **token vault stays in `client.crm_connections`** for billing scope (Epic 17 design decision) |
| `shared` | `migration_role` (DDL) — read by all | Immutable `audit_log` (append-only), global lookups (CPV codes, country codes) |

### 4.2 Core Entities (illustrative; not exhaustive)

```sql
-- =====================================================================
-- TENANT, USERS, WORKSPACES (Epic 1, 2, 14)
-- =====================================================================

CREATE TABLE client.companies (
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,
    country         TEXT NOT NULL,                -- ISO-3166 alpha-2
    cpv_codes       TEXT[] NOT NULL DEFAULT '{}',
    profile         JSONB NOT NULL DEFAULT '{}',
    stripe_customer_id   TEXT UNIQUE,             -- nullable until provisioned (Epic 8)
    vat_number      TEXT,
    vat_validation_status TEXT NOT NULL DEFAULT 'pending', -- pending|valid|invalid (Epic 8 fail-open)
    subscription_tier  TEXT NOT NULL DEFAULT 'free',       -- free|starter|professional|pro_plus|enterprise (Epic 15 +pro_plus)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client.users (
    id              UUID PRIMARY KEY,
    email           TEXT NOT NULL UNIQUE,
    password_hash   TEXT,                          -- nullable for OAuth-only users
    google_sub      TEXT UNIQUE,
    is_active       BOOLEAN NOT NULL DEFAULT true,  -- MUST be checked in all auth paths (CLAUDE.md)
    email_verified_at TIMESTAMPTZ,
    mfa_secret      TEXT,                           -- Fernet-encrypted
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client.company_memberships (
    company_id      UUID REFERENCES client.companies(id),
    user_id         UUID REFERENCES client.users(id),
    role            TEXT NOT NULL,                  -- admin|bid_manager|contributor|reviewer|read_only
    is_tenant_admin BOOLEAN NOT NULL DEFAULT false, -- cross-workspace bypass (Epic 14)
    PRIMARY KEY (company_id, user_id)
);

CREATE TABLE client.client_workspaces (                          -- Epic 14
    id              UUID PRIMARY KEY,
    company_id      UUID NOT NULL REFERENCES client.companies(id),
    name            TEXT NOT NULL,
    archived_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ix_client_workspaces_company_name_active
    ON client.client_workspaces (company_id, name)
    WHERE archived_at IS NULL;                                   -- Epic 14 pattern

CREATE TABLE client.workspace_memberships (
    workspace_id    UUID REFERENCES client.client_workspaces(id),
    user_id         UUID REFERENCES client.users(id),
    role            TEXT NOT NULL,
    PRIMARY KEY (workspace_id, user_id)
);

CREATE TABLE client.external_collaborators (                     -- Epic 14
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL REFERENCES client.client_workspaces(id),
    proposal_id     UUID NOT NULL REFERENCES client.proposals(id),
    email           TEXT NOT NULL,
    magic_link_jti  TEXT NOT NULL UNIQUE,
    expires_at      TIMESTAMPTZ NOT NULL,
    accepted_at     TIMESTAMPTZ,
    role            TEXT NOT NULL CHECK (role IN ('read_only', 'comment_only')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ix_external_collaborators_active_email
    ON client.external_collaborators (proposal_id, lower(email))
    WHERE accepted_at IS NULL;                                   -- Epic 14 pattern

-- Entity-level RBAC overrides (Epic 2)
CREATE TABLE client.entity_permissions (
    entity_type     TEXT NOT NULL,    -- 'proposal' | 'opportunity'
    entity_id       UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES client.users(id),
    permission      TEXT NOT NULL,    -- 'read' | 'write' | 'comment' | 'approve'
    PRIMARY KEY (entity_type, entity_id, user_id, permission)
);

-- =====================================================================
-- BILLING (Epic 8, 15)
-- =====================================================================

CREATE TABLE client.subscriptions (
    id              UUID PRIMARY KEY,
    company_id      UUID NOT NULL UNIQUE REFERENCES client.companies(id),
    stripe_subscription_id TEXT UNIQUE,
    tier            TEXT NOT NULL,
    status          TEXT NOT NULL,    -- trialing|active|past_due|canceled|unpaid
    current_period_end  TIMESTAMPTZ,
    trial_end       TIMESTAMPTZ
);

CREATE TABLE client.add_on_purchases (                           -- Epic 8 + 15
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL REFERENCES client.client_workspaces(id),
    opportunity_id  UUID NOT NULL,
    add_on_pricing_tier_id UUID NOT NULL,
    stripe_session_id TEXT NOT NULL UNIQUE,
    status          TEXT NOT NULL,    -- pending|succeeded|refunded
    purchased_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client.add_on_pricing_tiers (                       -- Epic 15
    id              UUID PRIMARY KEY,
    name            TEXT NOT NULL,    -- 'standard'|'grant_module'|'enterprise_stack'
    price_eur_cents INTEGER NOT NULL,
    stripe_price_id TEXT NOT NULL UNIQUE,
    feature_stack   JSONB NOT NULL,   -- which features unlock per opportunity
    active          BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE client.webhook_events (                             -- Epic 8 mandatory idempotency
    stripe_event_id TEXT PRIMARY KEY,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ
);

-- =====================================================================
-- OPPORTUNITIES + DOCUMENTS (Epic 5, 6)
-- =====================================================================

CREATE TABLE pipeline.opportunities (
    id              UUID PRIMARY KEY,
    source          TEXT NOT NULL,           -- 'aop'|'ted'
    source_ref      TEXT NOT NULL,           -- de-dup key
    title           TEXT NOT NULL,
    contracting_authority TEXT,
    cpv_codes       TEXT[],
    country         TEXT,
    budget_eur_cents BIGINT,
    deadline        TIMESTAMPTZ,
    raw_metadata    JSONB,
    fts             tsvector GENERATED ALWAYS AS (...) STORED,  -- GIN-indexed
    UNIQUE (source, source_ref)
);

CREATE TABLE client.opportunity_snapshots (                      -- denormalized for tenant scope
    id              UUID PRIMARY KEY,
    pipeline_opportunity_id UUID NOT NULL,   -- soft FK; cross-schema enforced at app layer
    workspace_id    UUID,                    -- nullable until Epic 14 backfill
    relevance_score INTEGER,                 -- 0-100, AI-computed (Epic 6)
    is_starred_by   UUID[],
    platform_attributed BOOLEAN NOT NULL DEFAULT false,  -- Epic 19 outcome telemetry
    snapshot_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client.documents (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL REFERENCES client.client_workspaces(id),
    proposal_id     UUID,
    s3_key          TEXT NOT NULL,
    filename        TEXT NOT NULL,
    mime_type       TEXT NOT NULL,
    clamav_status   TEXT NOT NULL DEFAULT 'pending',  -- pending|clean|infected|failed
    uploaded_by     UUID NOT NULL REFERENCES client.users(id),
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =====================================================================
-- PROPOSALS (Epic 7)
-- =====================================================================

CREATE TABLE client.proposals (
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL REFERENCES client.client_workspaces(id),
    opportunity_snapshot_id UUID NOT NULL,
    title           TEXT NOT NULL,
    status          TEXT NOT NULL,           -- draft|active|archived (Literal/StrEnum, Epic 13)
    generation_status TEXT,                  -- idle|generating|complete|failed
    current_version_number INTEGER NOT NULL DEFAULT 1,
    content_hash    TEXT NOT NULL,           -- SHA-256 for optimistic locking (Epic 7)
    created_by      UUID NOT NULL REFERENCES client.users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE client.proposal_versions (
    proposal_id     UUID REFERENCES client.proposals(id),
    version_number  INTEGER,
    content_blocks  JSONB NOT NULL,
    saved_by        UUID NOT NULL REFERENCES client.users(id),
    saved_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (proposal_id, version_number)
);

CREATE TABLE client.proposal_section_locks (                     -- post-MVP
    proposal_id     UUID REFERENCES client.proposals(id),
    section_id      TEXT NOT NULL,
    locked_by       UUID NOT NULL REFERENCES client.users(id),
    locked_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at      TIMESTAMPTZ NOT NULL,    -- atomic upsert with TTL renewal
    PRIMARY KEY (proposal_id, section_id)
);

CREATE TABLE client.content_blocks (                             -- shared boilerplate
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL,
    tenant_scope    BOOLEAN NOT NULL DEFAULT false,             -- Epic 14: cross-workspace library
    block_type      TEXT NOT NULL,
    content         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- =====================================================================
-- AGENTICSAI GATEWAY (Epic 4)
-- =====================================================================

CREATE TABLE gateway.agent_runs (
    id              UUID PRIMARY KEY,
    correlation_id  UUID NOT NULL,
    agent_name      TEXT NOT NULL,           -- logical (resolved via agents.yaml)
    caller_service  TEXT NOT NULL,           -- X-Caller-Service header
    workspace_id    UUID,
    status          TEXT NOT NULL,           -- running|complete|failed|cancelled
    input_size_bytes INTEGER,
    output_size_bytes INTEGER,
    duration_ms     INTEGER,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at     TIMESTAMPTZ
);

-- =====================================================================
-- INTEGRATIONS (Epic 17)
-- =====================================================================

CREATE TABLE client.crm_connections (                            -- token vault stays in client schema
    id              UUID PRIMARY KEY,
    workspace_id    UUID NOT NULL REFERENCES client.client_workspaces(id),
    provider        TEXT NOT NULL CHECK (provider IN ('hubspot','salesforce','pipedrive')),
    encrypted_oauth TEXT NOT NULL,           -- Fernet-encrypted token blob
    status          TEXT NOT NULL DEFAULT 'active',
    last_synced_at  TIMESTAMPTZ,
    UNIQUE (workspace_id, provider)
);

CREATE SCHEMA integrations;
CREATE TABLE integrations.sync_logs ( ... );
CREATE TABLE integrations.conflict_log ( ... );

-- =====================================================================
-- AUDIT TRAIL (Epic 5)
-- =====================================================================

CREATE TABLE shared.audit_log (
    id              BIGSERIAL PRIMARY KEY,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    actor_user_id   UUID,                    -- nullable for system events
    company_id      UUID,
    workspace_id    UUID,                    -- Epic 14
    action          TEXT NOT NULL,           -- e.g. 'proposal.publish'
    entity_type     TEXT NOT NULL,
    entity_id       TEXT NOT NULL,
    metadata        JSONB,                   -- structured, no free-form PII
    correlation_id  UUID
);
-- Append-only enforced via GRANT (no UPDATE/DELETE for application roles).
-- Writes are fire-and-forget: asyncio.create_task(_write_audit(...)) on .finally
-- (Epic 13 canonical form — never await on TTFB path).

-- =====================================================================
-- AGENTICSAI TENANT MAPPING (per ADR-018, 2026-05-12)
-- =====================================================================

CREATE TABLE client.agenticsai_projects (
    id                    UUID PRIMARY KEY,
    company_id            UUID NOT NULL UNIQUE REFERENCES client.companies(id),
    agenticsai_org_id        TEXT NOT NULL,                    -- the singleton EU Solicit Org
    agenticsai_project_id    TEXT NOT NULL UNIQUE,
    api_key_encrypted     BYTEA NOT NULL,                   -- Fernet, Epic 9 canonical module
    api_key_rotated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    n8n_subdomain         TEXT,                              -- nullable; org-shared but project-routed
    agent_map             JSONB NOT NULL DEFAULT '{}',      -- {logical_name: agenticsai_agent_uuid}
    provisioning_status   TEXT NOT NULL DEFAULT 'pending',  -- pending | provisioned | failed | archived
    provisioning_error    TEXT,                              -- last error if failed
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    archived_at           TIMESTAMPTZ
);
CREATE INDEX ix_agenticsai_projects_status ON client.agenticsai_projects(provisioning_status)
    WHERE provisioning_status != 'provisioned';   -- partial: hot rows only

-- =====================================================================
-- KB ARTEFACT METADATA (per ADR-019 — bodies live in AgenticSAI)
-- =====================================================================

CREATE TABLE client.agenticsai_kb_files (
    id                          UUID PRIMARY KEY,
    company_id                  UUID NOT NULL REFERENCES client.companies(id),
    agenticsai_storage_resource_id TEXT NOT NULL,
    agenticsai_file_id             TEXT NOT NULL UNIQUE,
    filename                    TEXT NOT NULL,
    content_type                TEXT NOT NULL,
    size_bytes                  BIGINT NOT NULL,
    sha256                      TEXT NOT NULL,                   -- for reconciliation
    artefact_category           TEXT NOT NULL,                   -- tender|profile|proposal|espd_template|rubric|other
    parsed_text_available_at    TIMESTAMPTZ,                     -- nullable; set when AgenticSAI completes processing
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    archived_at                 TIMESTAMPTZ
);
CREATE INDEX ix_agenticsai_kb_files_company_category
    ON client.agenticsai_kb_files (company_id, artefact_category)
    WHERE archived_at IS NULL;

-- =====================================================================
-- CRM MCP REGISTRATION (per ADR-020)
-- =====================================================================

CREATE TABLE client.agenticsai_mcp_servers (
    id                          UUID PRIMARY KEY,
    company_id                  UUID NOT NULL REFERENCES client.companies(id),
    provider                    TEXT NOT NULL,                  -- dynamics365|hubspot (v1); pipedrive|salesforce post-launch
    agenticsai_mcp_server_id       TEXT NOT NULL UNIQUE,
    crm_connection_id           UUID REFERENCES client.crm_connections(id),  -- token source
    status                      TEXT NOT NULL DEFAULT 'inactive',  -- inactive|registered|error
    last_token_pushed_at        TIMESTAMPTZ,
    last_error                  TEXT,
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (company_id, provider)
);

-- =====================================================================
-- WEBHOOK SUBSCRIPTIONS + RUN-STATE RECONCILIATION (per ADR-018)
-- =====================================================================

CREATE TABLE gateway.webhook_subscriptions (
    id                          UUID PRIMARY KEY,
    agenticsai_subscription_id     TEXT NOT NULL UNIQUE,
    event_types                 TEXT[] NOT NULL,                -- subscribed Standard Webhooks event types
    hmac_secret_encrypted       BYTEA NOT NULL,                 -- Fernet
    hmac_rotated_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE gateway.workflow_runs (
    id                          UUID PRIMARY KEY,
    company_id                  UUID NOT NULL REFERENCES client.companies(id),
    eusolicit_run_id            UUID NOT NULL UNIQUE,           -- our correlation id
    agenticsai_run_id              TEXT,                            -- null until AgenticSAI assigns
    agenticsai_job_id              TEXT,                            -- async-run job id, if applicable
    run_type                    TEXT NOT NULL,                  -- agent|team|workflow
    status                      TEXT NOT NULL,                  -- pending|running|completed|failed|cancelled
    started_at                  TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at                TIMESTAMPTZ,
    last_polled_at              TIMESTAMPTZ,                    -- for reconciler
    error_message               TEXT,
    payload_excerpt             JSONB                            -- redacted, bounded ≤16KB; for forensics
);
CREATE INDEX ix_workflow_runs_nonterminal
    ON gateway.workflow_runs(last_polled_at)
    WHERE status IN ('pending','running');  -- partial: reconciler scan target
```

**Design notes (AgenticSAI tables):**

- `client.agenticsai_projects.agent_map JSONB` is the **adapter pattern** retiring `agents.yaml`: logical names survive in EU Solicit code (`run_agent("proposal_drafter", ...)`), per-tenant resolution at call time. Cheap, debuggable.
- Partial indexes on `provisioning_status != 'provisioned'` and `status IN ('pending','running')`: hot rows only. The 5-min reconciler scans this index; full-table scans on `workflow_runs` would be a real cost at scale.
- `gateway.workflow_runs.payload_excerpt JSONB` is redacted, **bounded size** (≤16KB) — for forensics on failed runs. Full payloads live in AgenticSAI's traces; we keep a breadcrumb.
- No FKs from `client.agenticsai_projects` or `gateway.workflow_runs` to anything in `pipeline` or `gateway` schemas beyond `client.companies` — schema-isolation invariant (ADR-001) preserved.

### 4.3 Materialized Views (Epic 19)

```sql
-- Refreshed CONCURRENTLY (no read-locks; pattern from FR7.2 / Epic 19).
CREATE MATERIALIZED VIEW client.mv_workspace_outcome_stats AS ...;
CREATE MATERIALIZED VIEW client.mv_workspace_content_reuse_stats AS ...;
CREATE MATERIALIZED VIEW client.mv_workspace_onboarding_milestones AS ...;
CREATE MATERIALIZED VIEW client.mv_opportunity_match_quality AS ...;
```

Refresh cadence: hourly for milestones, daily for outcome and reuse stats. Orchestrated via Celery Beat in `notification` service.

### 4.4 Key Data Patterns

- **Tenant scoping**: every domain table carries `company_id` (and post-Epic 14 `workspace_id`). RBAC `Depends()` factory injects scope into queries; cross-tenant negative tests are mandatory story ACs.
- **Optimistic locking**: `content_hash` (SHA-256) on `proposals`. Client sends hash in PATCH body; server does `SELECT FOR UPDATE` + hash compare in transaction; mismatch returns 409 Conflict with structured body and triggers frontend conflict-resolution dialog (Epic 7).
- **Pessimistic locking** (post-MVP): atomic upsert into `proposal_section_locks` with TTL renewal.
- **Idempotency**: webhook dedup table with `event_id` UNIQUE; UNIQUE-violation returns 200 (already processed). Redis SETNX for short-lived idempotency keys (Epic 9 dual-layer).
- **Audit trail**: fire-and-forget `asyncio.create_task(_write_audit(...))` from `.finally`; task owns its own session with `async with session.begin()`; logs `audit_write_failed` at ERROR on exception. Never `await` on the TTFB path (Epic 13 Rule 45).
- **Soft delete uniqueness**: `UNIQUE (...) WHERE archived_at IS NULL` partial indexes; `IntegrityError → 409` translation in service layer (Epic 14).
- **Tier-aware Redis keys**: workspace-scoped key first (`{workspace_id}:{resource}`), fallback to company-scoped (`{company_id}:{resource}`) — asymmetric keys are silent bypass failures (Epic 15 pattern).
- **Atomic usage metering**: `_USAGE_LUA` script (`GET + conditional INCR + EXPIRE` in single Lua) is mandatory for any rate-limit/quota counter. **testcontainers Redis** (not fakeredis) for atomicity proof; concurrency proof asserts `len(set(results)) == N` not `max == N` (Epic 15).
- **Migration safety**: any downgrade recreating a UNIQUE/PK constraint must include a regression test inserting ≥2 rows under the same grouping key before downgrade (Epic 11).
- **Determinism**: lists derived from frozensets for SQL `IN()` clauses must be **explicit literals** (`["pro_plus","enterprise"]`), not `list(frozenset)` — non-deterministic order across Python versions (Epic 15).
- **Per-Project AgenticSAI API key as tenant credential boundary** (ADR-018, 2026-05-12): Fernet-encrypted at rest in `client.agenticsai_projects.api_key_encrypted`. Rotation on 90-day cadence with overlap: new key issued → smoke test → old key revoked. Rotation event publishes `agenticsai.key_rotated` to Redis Streams for downstream observability.
- **Per-Project agent-name resolution** (ADR-018 *Consequences*): `agent_map JSONB` on `client.agenticsai_projects`. Logical-name lookup at call time; missing key triggers re-sync against AgenticSAI Project agent inventory before falling back to 503.
- **KB artefact integrity via SHA-256 reconciliation** (ADR-019): `client.agenticsai_kb_files.sha256` recorded at upload; nightly reconciliation against AgenticSAI inventory; mismatches → admin alert + audit entry.
- **Cross-substrate Right-to-Erasure** (ADR-019): erasure flow has two ACK steps in `shared.audit_log` (`erasure_step: postgres_rows_deleted`, `erasure_step: agenticsai_files_deleted`); erasure not certified complete until both present.
- **Workflow-run reconciliation as authoritative** (ADR-018 *Consequences*): `gateway.workflow_runs.status` is converged by the 5-min reconciler polling `GET /jobs/{jobId}/status`; webhooks are latency optimisations and may be lost without correctness impact. Test design must include `webhook_dropped, reconciler_recovers` scenario. Both webhook and reconciler may converge the same row — use `UPDATE ... WHERE status IN ('pending','running')` guards rather than blind state writes.

---

## 5. API Boundaries

### 5.1 External APIs

| Surface | Audience | Auth | Notes |
|---|---|---|---|
| `https://app.eusolicit.com/api/*` (`client-api`) | End users via `apps/client` | RS256 JWT in `Authorization` header (or httpOnly cookie) | REST + SSE (`/proposals/{id}/draft/stream`); OpenAPI generated; tier gates and RBAC enforced via per-route `Depends()`; CORS pinned to known origins |
| `https://api.eusolicit.com/v1/*` (Enterprise API) | Customer dev integrations | `X-API-Key` (per-workspace) + scoped JWT | Routed through `client-api` with stricter rate limits and reduced field set; Enterprise tier-gated |
| `https://app.eusolicit.com/trust/*` | Public (no auth) | — | Static-rendered Next.js (Epic 18); MDX + sub-processors YAML; PDF artefacts via signed S3 URLs |
| `https://admin.eusolicit.com/*` (`apps/admin` + `admin-api`) | Internal ops | RS256 JWT + MFA + IP allowlist middleware | All admin endpoints VPN-restricted |
| Stripe webhooks | Stripe | ECDSA on raw body, fail-closed | `webhook_events` UNIQUE-constraint dedup |
| **AgenticSAI** (`https://agenticsai.endigitalx.com/`) | Outbound + inbound webhooks | Per-Project api-key (Bearer) outbound; HMAC SHA-256 inbound (Standard Webhooks spec) | All outbound wrapped in `circuit_breaker(retry(http_factory))` with explicit `httpx` timeout. Logical-name circuit-breaker keys = `(eusolicit_logical_name, agenticsai_project_id)`. Inbound webhooks: signature verified via `hmac.compare_digest()`, 7-day idempotency cache, DLQ for poison events. Per ADR-018 |
| AOP / TED / Stripe / VIES / Google OAuth / Microsoft Dynamics / HubSpot / Slack / MS Teams (Pipedrive/Salesforce deferred) | Outbound | Provider-specific | All wrapped in `circuit_breaker(retry(http_factory))`. Dynamics + HubSpot are invoked via AgenticSAI MCP servers (per ADR-020); EU Solicit's direct HTTP only used by OAuth-callback flow + token-rotation push to AgenticSAI secrets |

### 5.2 Internal APIs

| Caller | Callee | Mechanism | Notes |
|---|---|---|---|
| `client-api`, `admin-api` | `agenticsai-gateway` | Internal REST (cluster DNS) | `AgenticsaiGatewayClient.run_agent(logical_name, payload, company_id)` + `X-Caller-Service` header. Flat 503 body `{"message", "code"}` (Epic 11 standard). Async-poll variant for long runs: `submit_agent_run_async` + `get_job_status` |
| `client-api` | `pipeline` schema | **Read-only async session** (`get_pipeline_readonly_session`) | Dual-session pattern (Epic 6); separate `MetaData(schema=...)`; **no FK** across schemas |
| `data-pipeline`, `client-api`, `notification` | each other | Redis Streams events | At-least-once; consumer groups; DLQ; idempotent handlers |
| `notification` | external email/Slack/Teams | Outbound REST | Per-provider circuit breaker |
| `integrations-api` | external CRM/Slack/Teams | Outbound REST + per-provider Celery worker queue | OAuth refresh on Beat schedule; bi-directional sync with LWW conflict resolution |

### 5.3 Event Catalog (selected — Redis Streams)

Stream → consumer → semantics:

- `opportunity.ingested` → `client-api` (relevance scoring), `notification` (digest emails)
- `opportunity.matched` → `notification` (alert dispatch)
- `proposal.created` / `proposal.published` → `notification`, `audit_log` writer
- `addon.opportunity_engaged` → billing service (per-bid SKU offer trigger; Epic 15)
- `subscription.changed` → `tier_cache` invalidation (DELETE; Epic 8)
- `subprocessor.changed` → `notification` (DPA email to customers; Epic 18)
- `crm.connection_created` / `crm.sync_failed` → `notification` (workspace alerts; Epic 17)
- `onboarding.milestone_reached` → analytics + CSM stall tracker (Epic 19)
- `tenant.provisioned` (NEW 2026-05-12; published by `client-api` after E24 S24.02) → `notification` (welcome email enriched with KB-upload CTA), `client-api` (UI badge update for provisioning completion)
- `agenticsai.key_rotated` (NEW 2026-05-12; published by `agenticsai-gateway` after E04 S04.22 rotation) → `agenticsai-gateway` mapping cache invalidation, `notification` (admin audit notification on rotation)

**AgenticSAI inbound webhooks → internal Redis Streams** (per ADR-018, Standard Webhooks spec):

- AgenticSAI `workflow.completed` → internal `agenticsai.workflow.completed` → `data-pipeline` (opportunity-ingestion completion handler), `client-api` (user-facing notification)
- AgenticSAI `agent.run.completed` → internal `agenticsai.agent.run.completed` → `gateway` (workflow_runs status converge), `notification` (user-facing run-completion alerts)
- AgenticSAI `storage.file.processed` → internal `agenticsai.kb.file.processed` → `client-api` (set `client.agenticsai_kb_files.parsed_text_available_at`)
- AgenticSAI `policy.violation` → internal `agenticsai.policy.violation` → `notification` (admin alert), `shared.audit_log` (immutable record)

**Event handler discipline** (project-context):

- Narrow `except` clauses — never swallow `celery.exceptions.Retry` or `asyncio.CancelledError` (Epic 9, Epic 13).
- Idempotency mandatory: every handler keys on a stable event_id, uses dedup table or SETNX.
- 4xx errors must NOT increment circuit-breaker failure counters (Epic 5 OBS-001 fix).
- **AgenticSAI-origin events must be idempotent against the reconciler** — both webhook and reconciler may converge the same `workflow_runs` row. Use `UPDATE ... WHERE status IN ('pending','running')` guards rather than blind state writes.

---

## 6. Deployment Topology

### 6.1 Environments

| Environment | Purpose | Notes |
|---|---|---|
| `local` | Developer laptops | `make up` brings up all services + infra via docker-compose. Postgres :5432, Redis :6379, MinIO :9000, ClamAV :3310 |
| `dev` | Shared integration / preview | Per-PR preview deployments via GitHub Actions |
| `staging` | Pre-production | Full prod parity with reduced replica counts; sanitized prod data snapshots |
| `production` | Customer-serving | AWS eu-central-1 (hard residency requirement, NFR + Domain) |

### 6.2 Production Topology (single-host on-prem, per ADR-010)

> **2026-05-11 ADR-010 supersedes the prior AWS managed-services topology.** What follows is the launch posture; the AWS topology is preserved in §ADR-010 *Historical* and in `architecture.v2.0.bak.md` for reference. Phase-2 HA migration is out of scope until a future epic.

- **Host**: `www1.endigitalx.com` — single VPS (Hetzner, EU region — Frankfurt or Falkenstein). Co-tenant with `lifematch-*`, `celthrac.com`, etc.; noisy-neighbour risk managed by docker `mem_limit` + cpu reservations.
- **Compute**: docker-compose (`docker-compose.prod.yml`); each microservice as a single container with `restart: unless-stopped` + healthcheck. **No Kubernetes; no horizontal replicas.** Deploy via GitHub Actions over SSH.
- **Ingress**: host nginx on www1 (TLS 1.3, certbot-managed Let's Encrypt). HSTS preload. Reverse-proxy to per-service ports (8001/8002/8004/8005/8007). nginx config under `infra/nginx/` in repo; deploy is a manual `sudo cp` on www1 (per project memory: nginx + certbot are manual, not in `deploy.yml`).
- **Database**: PostgreSQL 16 as a single container with host-mounted volume. **No Multi-AZ; no read replica.** Daily logical backup (`pg_dump`) + WAL archiving to `/home/docker/backups/eusolicit/`, replicated off-site to **Hetzner Storage Box** (EU residency). PITR window: 7 days from latest base backup.
- **Redis**: Redis 7 as a single container with AOF + RDB persistence. **No Sentinel.** Separate logical DB indices: `0` for application cache/streams, `1` for tests.
- **Object storage**: local volume on www1 for Trust Center artefacts (Git-tracked PDFs + WeasyPrint output). Tender raw downloads cached briefly on disk before upload to AgenticSAI storage-resources (per ADR-019 — AgenticSAI KB is canonical, S3 is no longer in production scope).
- **Secrets**: sourced from `/home/docker/eusolicit-overrides/.env.prod` on www1 with file-permission isolation; not a managed secrets store. Includes DB credentials, Stripe keys, per-Project AgenticSAI API keys (Fernet-encrypted in `client.agenticsai_projects.api_key_encrypted`), Standard Webhooks HMAC secrets, OAuth tokens for Dynamics + HubSpot. Rotation cadences in ADR-004 / ADR-018 / ADR-020.
- **CDN**: CloudFlare in front of the public `/trust/*` route only (per Trust Center §ADR-011). The authenticated app routes are not CDN-fronted at launch.
- **DNS**: provider's nameservers; no Route53.
- **Paging**: email + Telegram bot (PagerDuty cancelled, per ADR-010).

### 6.3 CI/CD (GitHub Actions)

Pipelines in `.github/workflows/`:

1. **PR pipeline**: matrix lint (ruff, mypy, eslint, tsc, `pnpm check:i18n`, `no-literal-text`) → unit tests → integration tests (testcontainers) → Vitest → Playwright (chromium first, then full matrix on `main`) → `make coverage` (≥80% gate) → docker image build → preview deployment.
2. **`main` pipeline** (per ADR-010): full matrix → docker image build → push to GHCR → SSH-based deploy to `www1.endigitalx.com` via `deploy.yml` workflow (docker-compose pull + recreate). nginx + certbot changes remain manual on www1 (per project memory). Helm/ArgoCD/ECR removed in 2026-05-11 pivot.
3. **Cron jobs**: nightly Dependabot security scan; weekly Trivy container scan; weekly k6 baseline load test (Epic 21 PE.01).

**Quality gates** (CI-enforced; aligned with project-context standards):

- `ruff check`, `mypy`, `eslint`, `tsc`, `pnpm check:i18n`, `no-literal-text` — zero errors on `main`
- ≥80% backend unit + integration coverage; ≥90% frontend E2E coverage on critical journeys
- TEA review score ≥80/100 as a story-level gate (project-context Epic 14 fix — embedded as story AC, not separate story)
- Senior code review: `REVIEW: Approve` required + quoted pytest/Playwright execution output — approval without quoted execution result is invalid (Epic 13)
- Story-close: orchestrator verifies `Status: done` in story file AND `REVIEW: Approve` in review section before writing `done` to sprint-status.yaml (Epic 13/14/15 — recurring anti-pattern)

### 6.4 Observability

- **Metrics**: Prometheus scraping `/metrics` on every service. Custom `CollectorRegistry` per service (avoids global-state test pollution). Mandatory metrics (project-context retros):
  - REST p50/p95/p99 latency histograms
  - SSE TTFB histogram (target <500ms)
  - Outbound provider call latency + error rate (Stripe, AgenticSAI, VIES, AOP, TED, Dynamics, HubSpot, Slack, Teams)
  - Circuit-breaker state gauge per provider
  - Webhook processing latency histogram (Epic 8)
  - `billing_usage_sync_drift_total` gauge, active-tier distribution gauge, trial-to-paid conversion counter (Epic 8 retro)
  - Pipeline throughput (24h freshness SLA — Epic 5)
- **Dashboards**: Grafana per service + cross-service SLO dashboards with error-budget burn-rate alerting.
- **Logging**: structlog JSON → stdout → Promtail → Loki. Mandatory fields: `service`, `request_id`, `correlation_id`, `actor_user_id`, `company_id`, `workspace_id`. Never log secrets; never `str(exc)` in user-facing 5xx responses (Epic 5 anti-pattern: connection-string leakage).
- **Tracing**: OpenTelemetry → Jaeger; instrument FastAPI middleware, SQLAlchemy, httpx, Celery task boundaries.
- **Alerting**: email + Telegram bot per ADR-010 (PagerDuty cancelled). SLO burn-rate alerts (1h fast-burn at 14.4× budget; 6h slow-burn at 3×) reframed as best-effort beta posture, not a public SLA.

### 6.5 Data Backup, DR, Residency

- **Backups** (per ADR-010): daily `pg_dump` + WAL archiving to `/home/docker/backups/eusolicit/` on www1; nightly `rsync` to Hetzner Storage Box (EU residency). Redis AOF + RDB persistence. **No cross-region replication** (residency); single-host = single point of failure for compute.
- **DR**: documented runbook (Epic 22 onprem-* stories). RTO ≤ 4h (NFR-17); RPO ≤ 24h (NFR-15) — **gated on backup-restore drill measurement**. Annual DR test mandatory; 99.9% SLA promise withdrawn for launch ("Service is in beta. Best-effort availability.")
- **AgenticSAI substrate** (per ADR-019): unstructured artefacts (tenders, profiles, past proposals, ESPD templates, rubrics) live in AgenticSAI storage-resources, not on www1. EU residency for the AgenticSAI EU Solicit Organisation is a contractual prerequisite (§11.3 risk #1). Cross-substrate erasure requires both `postgres_rows_deleted` and `agenticsai_files_deleted` ACKs in `shared.audit_log`.
- **Right-to-erasure (GDPR Art. 17)**: User-data soft-delete cascade to all tenant-scoped tables; audit log retention preserved under Art. 17.3.b legal-obligation exception (documented in Trust Center). Audit entries reference IDs only — never free-form PII.

---

## 7. Key Architectural Decisions (ADRs)

The ADRs below are the foundational decisions; each has been validated through one or more shipped epics. Any deviation requires a written, reviewed ADR amendment.

### ADR-001 — Schema-per-service logical isolation in a single PostgreSQL instance

**Status:** Accepted (Epic 1)
**Decision:** Six service schemas (`client`, `admin`, `pipeline`, `gateway`, `notification`, `integrations`) plus `shared` in a single Postgres 16 instance. Each schema owned by one service role with CRUD; `migration_role` owns DDL; cross-schema queries are forbidden. Cross-schema reads use the **dual-session pattern** with separate `MetaData(schema=...)` and read-only sessions (Epic 6).
**Rationale:** Operational simplicity (single Postgres, single backup, single failover) with strong logical boundaries enforced at the role-permission level. Avoids the operational cost of N managed databases at MVP scale while still preserving migration paths to physical isolation later.
**Consequences:** No cross-schema FKs; consuming services hold a soft FK plus a denormalized snapshot when they need to query (e.g. `client.opportunity_snapshots`).

### ADR-002 — Two-layer authentication and granular RBAC

**Status:** Accepted (Epic 1, 2; extended Epic 14)
**Decision:** Dual-layer auth: Next.js middleware (`apps/client/middleware.ts`) for route-level pre-checks + client-side layout guards for state-aware redirects. Backend RS256 JWT with `Depends(get_current_user)`. RBAC is a **company-role ceiling** (`admin|bid_manager|contributor|reviewer|read_only`) plus optional **per-entity `EntityPermission`** rows that can override the ceiling on specific resources. Post-Epic 14: RBAC factored across `WorkspaceScope` + `check_entity_access()` with cross-workspace `tenant_admin` bypass. External collaborator JWTs use a separate `ExternalCollaboratorPrincipal` dataclass — non-substitutable for `CurrentUser` at the type level (Epic 14 pattern).
**Rationale:** Aligns with PRD §FR-7 / §SaaS RBAC; extension via per-entity permissions accommodates "Bid Manager on this proposal, Reviewer on that one." Two-layer auth catches bypass attempts that single-layer middleware would miss.
**Consequences:** Every cross-tenant endpoint requires a 403/404 negative test as a story AC (CLAUDE.md mandate). 404 (not 403) for cross-user scoped resources to avoid existence leakage (Epic 9).

### ADR-003 — Redis Streams as primary event bus (not Celery as message broker)

**Status:** Accepted (Epic 4, validated through Epics 5–9)
**Decision:** Custom `EventPublisher` / `EventConsumer` abstraction over Redis Streams (`XADD` / `XREADGROUP`). Consumer groups per service, per-stream DLQ on N retries. Celery is used only for time-driven tasks (Beat schedules) and CPU-heavy batch jobs — not as the application event bus.
**Rationale:** At-least-once with explicit DLQ visibility, lower operational footprint than Kafka at MVP scale, native to the Redis we already run for cache.
**Consequences:** Idempotent consumers mandatory; `webhook_events`-style dedup tables for any consumer with side effects; never broaden `except` to swallow `celery.exceptions.Retry` (Epic 9).

### ADR-004 — Two-layer outbound resilience: `circuit_breaker(retry(http_factory))`

**Status:** Accepted (Epic 4, reused Epic 5/8/9/15); 2026-05-12 AgenticSAI addendum below.
**Decision:** All outbound HTTP calls (AgenticSAI, Stripe, VIES, AOP, TED, HubSpot, Microsoft Dynamics, Slack, Teams, calendar) use composed resilience: outer **circuit breaker** wrapping inner **exponential backoff retry** wrapping the typed `httpx` client factory. Explicit `httpx` timeouts mandatory.
**Rationale:** Keeps protections layered correctly — circuit breaker counts only network/5xx failures (4xx must NOT increment failure counter, Epic 5 OBS-001), retry handles transient network hiccups, factory ensures a single typed client per provider.
**Consequences:** Stripe must adopt this pattern in `billing_service.py` and `vies_service.py` (Epic 8 carry-forward). Open-circuit fallback varies per call: payment paths fail-closed; AI-summary paths fail-open with degraded result; VIES fails-open to `vat_validation_status: pending`.

> **Addendum 2026-05-12 (per ADR-018):** Outbound calls to **AgenticSAI** (replacing prior "AgenticSAI" framing) use the same two-layer composition. The pattern is unchanged. Logical-name keys for circuit-breaker buckets shift from agent-name-from-yaml-registry (retired) to a composite of `(eusolicit_logical_name, agenticsai_project_id)` resolved at call time by `agenticsai-gateway`. Open-circuit fallback for AI paths remains **fail-open with degraded result + tenant-visible banner** for outages >5 minutes; payment-path circuit breakers (Stripe) remain **fail-closed**.

### ADR-005 — Streaming AI via Server-Sent Events with strict lifecycle controls

**Status:** Accepted (Epic 4, hardened Epic 6, 13)
**Decision:** AI generation uses SSE (`text/event-stream`). Native `fetch` + `ReadableStream` on the frontend (NOT `EventSource`, which is GET-only and silently caches; NOT `Axios`, which buffers the body — Epic 6). Lifecycle invariants:

1. Quota check **before** `StreamingResponse` creation (Epic 6 anti-pattern: leaked permits).
2. Generator closed in `finally`.
3. Terminal event guaranteed (success or sanitized error).
4. Fresh `session_factory` inside the generator body (not the request-scoped session).
5. Single-point error sanitization helper per endpoint (Epic 13 — never inline error frame construction).
6. `except asyncio.CancelledError: raise` in any `async with / try / finally` — Python 3.8+ `CancelledError` inherits `BaseException`, so bare `except Exception` silently drops cancel signals (Epic 13).

**Rationale:** TTFB <500ms NFR; SSE keeps the protocol simple, browser-native, behind the existing TLS/ALB stack. Streaming POST bodies require native `fetch` because `EventSource` cannot send a request body.
**Consequences:** Each SSE endpoint requires lifecycle ATDD assertions; integration tests must include cancel-mid-stream scenarios.

### ADR-006 — Tier gating and usage metering as per-route `Depends()`, never as middleware

**Status:** Accepted (Epic 6, hardened Epic 8/15)
**Decision:** `TierGate(min_tier=Tier.PROFESSIONAL)` and `UsageGate(feature=Feature.AI_SUMMARY)` are FastAPI `Depends()` factories applied per-route. Field-level Pydantic models (e.g. `OpportunityFreeResponse` with exactly the 6 free-tier fields) prevent free-tier field leakage. Atomic Redis Lua (`_USAGE_LUA`) is mandatory for metering — `GET + conditional INCR + EXPIRE` in a single script. Per-bid add-on bypass: tier cap is skipped if `add_on_purchases.opportunity_id` matches and `status='succeeded'` (Epic 15). Tier cache uses **DELETE-on-change**, not SET (Epic 8).
**Rationale:** Middleware-based gating leaks fields silently when serialization changes; per-route `Depends()` makes the tier requirement co-located with the endpoint and visible in OpenAPI. ATDD assertions on field-set are stable.
**Consequences:** Every new endpoint must declare `min_tier` explicitly; Vitest source-inspection tests check both `<TierGate>` and the response model field set. Workspace-scoped Redis keys with company-scoped fallback (Epic 15) — asymmetric keys are silent bypass failures.

### ADR-007 — Multi-client workspaces under `company`, not as a renamed tenant

**Status:** Accepted (Epic 14, ratified 2026-04-25 architecture evaluation)
**Decision:** Keep `company` as the tenant boundary (no rename — would touch ~50% of the codebase). Introduce `client_workspaces` as child of `company`. Existing companies auto-backfill with one default workspace at migration time. Atomic single-transaction migration (S14.00). RBAC extends with `WorkspaceScope` `Depends()` plus a tenant-level `tenant_admin` role for cross-workspace operations. **No PostgreSQL RLS for correctness** — the existing `check_entity_access()` factory plus `WorkspaceScope` is sufficient. RLS deferred to ISO 27001 audit prep as defence-in-depth, not blocking.
**Rationale:** Avoids 13-epic-regression risk of renaming `company`. Workspaces are the third instance of scope-bearing concept (after company and per-entity permissions) — Rule of Three holds. Schema-isolation pattern preserved verbatim.
**Consequences:** Frontend route refactor `/[locale]/workspace/[workspaceId]/...` (S14.03 — two ATDD-heavy sprints). Zustand persist namespace migrates v1→v2 with non-destructive shim. Stripe seat counting reads only `company_memberships`, never `external_collaborators` (table-as-contract; Epic 14 pattern). External collaborator magic-link JWTs are claim-shape-isolated.

### ADR-008 — Per-bid SKU as one-time Stripe charges, not subscription tier upgrade

**Status:** Accepted (v5 §1.14, Epic 15 refinement)
**Decision:** Per-bid SKU (`add_on_purchases`) issued via Stripe Checkout Session in `mode: payment`. Three pricing tiers (€99 / €199 / €299 — locked 2026-04-25) seeded into `add_on_pricing_tiers` with Stripe Price IDs. Triggered via existing `bid_decisions` flow (`bid_decision.recommendation = 'bid'` → `addon.opportunity_engaged` event). Pro+ tier added between Professional and Enterprise (€199/user/month — locked 2026-04-25). Non-refundable with 24h cancellation window. EU VAT via Stripe Tax with separate product mapping for per-bid vs. subscription SKUs.
**Rationale:** Per-bid is one-time revenue (not ARR); subscription tier model is the wrong primitive. Reusing `add_on_purchases` (already in v5 Sol Arch) avoids new payment primitives.
**Consequences:** Metering bypass is one Lua-script edit, not a redesign (Epic 15 pattern: workspace-scoped key first, fallback to company-scoped). Stripe success/cancel URLs MUST take `locale` and `workspace_id` as parameters (Epic 15 anti-pattern: hardcoded `/bg/workspace/default/`).

### ADR-009 — `integrations-api` as a separate service

**Status:** Accepted (Epic 16/17, ratified 2026-04-25)
**Decision:** Net-new service `integrations-api` (port 8007) for HubSpot/Salesforce/Pipedrive/Slack/Teams. Reasons mirroring agenticsai-gateway split:

1. Blast radius: provider outages or rate-limits don't cascade into `client-api`.
2. Per-provider rate-limit characteristics warrant per-provider Celery worker queues.
3. Sync workers belong in their own deployment (decouples web-pod scaling from sync-worker scaling).
4. Three concurrent providers (HubSpot + Salesforce + Pipedrive) at launch tip the Rule of Three.

Slack/Teams (incoming-webhook templates) ships **before** CRM (bi-directional sync) — smaller scope, daily-felt value, larger addressable Pro tier vs. Pro+ tier.
**Rationale:** Decomposition stays at 6+1=7 services post-amendment — within the 5–10 sweet spot.
**Consequences:** New `integrations` schema; OAuth token vault stays in `client.crm_connections` for billing scope; `last-write-wins` conflict resolution with workspace-visible conflict log to `shared.audit_log`.

### ADR-010 — Single-host on-premise Docker for launch (supersedes 2026-04-25)

**Status:** Accepted (2026-05-11). Supersedes the 2026-04-25 version which committed to managed AWS (RDS Multi-AZ + ElastiCache + EKS + AMP/AMG). The prior version's "Implementation status (2026-05-04..05)" notes below are PRESERVED as historical record but are NOT current state — Stories 21-2 / 21-3 / 21-4 Terraform deliverables are being deleted in this pivot; what they describe was never provisioned in AWS, only authored as code.

**Decision:** EU Solicit ships its production launch on **`www1.endigitalx.com` as a single Docker host**. PostgreSQL 16 and Redis 7 run as containers backed by host-mounted volumes. Backups go daily to `/home/docker/backups/eusolicit/` and are replicated off-site to **Hetzner Storage Box** (EU residency). Service-level redundancy is provided by `restart: unless-stopped` + healthcheck-driven container restarts; there is **no** horizontal replica layer and **no** Multi-AZ. Paging uses **email + Telegram bot** (PagerDuty cancelled). The 99.9% SLA promise is **withdrawn for launch**; the public posture is "Service is in beta. Best-effort availability."

**Rationale:** The 2026-04-25 ADR committed to a 2–3 week AWS migration with hidden EKS prerequisite work and ~€2,000–2,500/mo ongoing cost. Pre-flight review on 2026-05-11 surfaced 5 blockers including a kubernetes module that was a TODO stub — i.e., the EKS-cluster prerequisite the migration assumed did not exist. With a small operating team and the existing single-host docker stack already running well enough (the recent 4-hour outage was a deploy-pipeline issue, not an infra-layer issue), the simpler launch posture beats the theoretical-HA path. The launch happens; HA is a future-Phase-2 question.

**Consequences:**
- ~€10–20/mo run rate increment (Hetzner Storage Box + Telegram-bot tier) vs. the original +€250–600/mo AWS estimate. ~€2,000+/mo savings annually.
- Single physical host = single point of failure for compute. RTO ≤ 4h, RPO ≤ 24h (gated on backup-restore drill measurement).
- No horizontal scaling. When traffic exceeds www1 capacity: bigger VPS, or open a future Phase-2 HA-migration epic.
- ISO 27001 achievable; SOC 2 Type II availability commitments may need a contractual carveout.
- Co-tenancy on www1 with `lifematch-*`, `celthrac.com`, etc. — noisy-neighbour risk managed by docker `mem_limit` + cpu reservations.
- New launch-blocker stories (sprint-status development_status keys `onprem-01..onprem-06`) replace `pe-02` / `pe-03` and rescope `pe-04` / `pe-06` / `public-sla-announcement-soak-gate`.

**Retained from prior PE.* work:** `pool_pre_ping=True` across all services, `redis-py` resilience hardening (`socket_keepalive`, `health_check_interval`, retry policy), Celery `broker_connection_retry_on_startup`, `/metrics` endpoints via `eusolicit_common.observability` middleware, the 7 Grafana dashboard JSONs at `infra/observability/grafana/dashboards/`, the 15 runbooks at `eusolicit-docs/runbooks/`, the 7 k6 baseline scripts at `tests/load/`.

**Removed in this pivot:** AWS Terraform modules and environments, the Helm chart with PDBs/HPAs/ESO/NetworkPolicy, AMP/AMG, PagerDuty Terraform integration. AWS-specific cutover runbooks remain in `implementation-artifacts/` as historical reference.

**Decision record:** Full deliberation, options matrix, and ratified sub-decisions captured in `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md`.

---

#### Historical: ADR-010 (2026-04-25, superseded by 2026-05-11)

> **Status:** Accepted (Epic 21, Plan 2026-04-25), **superseded 2026-05-11**.
> **Decision:** Migrate to managed Multi-AZ RDS (Aurora candidate; RDS Postgres baseline) and managed Redis with Sentinel. PDBs with `minAvailable: 1` and ≥2 replicas per service in production. SLO dashboards with error-budget burn-rate alerting. PagerDuty on-call rotation with runbook density. **Do not publish 99.9% SLA before infra uplift completes** (Phase A → B → C).
> **Rationale:** Patroni-on-K8s is operational debt for a small team. Managed databases with documented Multi-AZ failover semantics are auditable and predictable.
> **Consequences:** ~+€250–600/mo run rate. k6 baseline closure (PE.01) is the **first** Epic 21 story — has been deferred 6 epics; cannot publish any SLO without it. AgenticSAI incidents excluded from SLA scope (isolated by agenticsai-gateway circuit breaker).

**Implementation status (2026-05-04):** Story 21-2 closed. Production RDS Multi-AZ provisioned (eu-central-1, db.r6g.large, 35d backup retention, Performance Insights enabled). Migration `M_PE02_opportunities_tsv_gin_index` shipped (data-pipeline rev 003). FTS plan flipped from `Seq Scan` to `Bitmap Index Scan on ix_opportunities_tsv` — see `implementation-artifacts/load-test-results.md` §EXPLAIN ANALYZE Results — Post-PE.02 Migration for verbatim evidence. Multi-AZ failover drill (staging) documented in `implementation-artifacts/pe-02-cutover-runbook.md` §Failover Drill Results; production drill pending operator-on-call execution per D-1 pre-recorded deviation.

**Implementation status (2026-05-04) — PE.03 (Story 21-3):** Story 21-3 closed (dev pass). Production ElastiCache for Redis Replication Group provisioned via Terraform (eu-central-1, cache.r6g.large × 3 nodes Multi-AZ + automatic failover, cluster-mode-disabled, 7d snapshot retention, at-rest + transit encryption with auth_token via Secrets Manager per AP-GUARD-11). Per-service Redis connection strings via External Secrets Operator (Shape A — extending Story 21-2 `externalsecret.yaml` pattern; activates via `.Values.externalSecret.redis.enabled`). All `redis-py` `from_url` call sites hardened with `socket_keepalive=True` + `health_check_interval=30` (pool_pre_ping equivalent) + `retry=Retry(ExponentialBackoff(cap=10, base=1), 3)` + `retry_on_error=[ConnectionError, TimeoutError]` + `socket_connect_timeout=5` + `socket_timeout=10`. Both Celery celery_app.py files updated with `broker_connection_retry_on_startup=True` + `broker_transport_options` resilience keys. Multi-AZ failover drill and production cutover deferred to operator-action follow-up per D-1 pre-recorded deviation; §Failover Drill Results and §Staging Rehearsal Timing to be populated by operator. See `implementation-artifacts/pe-03-cutover-runbook.md` for the canonical cutover runbook + §Connection Audit (15 sites, all hardened) + §ESO Wiring Decision (Shape A chosen) + §Failover Drill Steps (procedure pre-documented for operator execution).

**Implementation status (2026-05-05) — PE.04 (Story 21-4):** Story 21-4 dev pass. PDB `minAvailable: 1` enforced across all 6 production services (client-api, admin-api, agenticsai-gateway, data-pipeline, notification, integrations-api); HPA `minReplicas` floors per epic line 105 met (admin-api 1→2, notification 1→2; integrations-api new at 2; client-api/agenticsai-gateway/data-pipeline already at floors). NEW `infra/helm/values/integrations-api.yaml` Helm values file per ADR-009 (port 8007). CI lint gate `scripts/check_helm_pdb_and_minreplicas.py` + `scripts/pe04_min_replica_floors.py` added — rejects future values files without HA primitives; runs on every push and PR. Chaos-drill runbook pre-documented at `implementation-artifacts/pe-04-chaos-drill-runbook.md` (§Drain Procedure + §Per-Service Drill + §PDB-Behaviour Evidence + §HPA Sizing Review + §Network-Policy Verification); live staging drill deferred to operator-on-call execution per D-1. NetworkPolicy HA verification: all 7 values files use `podSelector`/`namespaceSelector`/`ipBlock`/`to: []` — no literal pod IPs (verified). nginx-ingress PDB verification commands documented in runbook §Pre-flight; conditional override file at `infra/helm/values/ingress-nginx-overrides.yaml`. **SLA-publication gate `PE.01 + PE.02 + PE.03 + PE.04` advances to 4/4 done** from a code-and-config standpoint (PE.05 SLO dashboards + PE.06 on-call rotation are parallel/non-gating per epic line 20).

**Implementation status (2026-05-05) — PE.05 (Story 21-5):** Story 21-5 dev pass. Per-service `/metrics` endpoints wired via shared `eusolicit_common.observability` middleware (5 services: client-api, admin-api, agenticsai-gateway, notification, enterprise-api); existing data-pipeline `PIPELINE_METRICS_REGISTRY` (Story 5.12) and integrations-api `METRICS_REGISTRY` (Story 17.0) preserved verbatim with HTTP-layer histograms added additively. All 4 mandatory HTTP metrics include `slo_target` label (`"platform"` / `"agenticsai-dependent"`) per AC-6.4 + architecture.md line 762 AgenticSAI isolation. Celery worker metrics wired via per-service `metrics_signals.py` modules (data-pipeline + notification) with `task_prerun` / `task_postrun` / `task_failure` / `task_retry` signal handlers + beat task queue-depth poll every 30s. Redis metrics via CloudWatch exporter (`AWS/ElastiCache`). Postgres metrics via CloudWatch exporter (`AWS/RDS`) + dedicated `postgres-exporter` Helm release authenticated as `monitoring_role`. Grafana dashboards committed as JSON at `infra/observability/grafana/dashboards/` (7 per-service + 1 cross-cutting `platform-slo.json` with 4-window multi-burn-rate layout per Google SRE Workbook §5.2). Prometheus recording + alerting rules at `infra/observability/prometheus/rules/` (plain AMP format). Alertmanager routing config at `infra/observability/alertmanager/alertmanager.yaml` (page → PagerDuty, ticket → Slack, info → null; secrets via ESO). AMP+AMG provisioned via `infra/terraform/modules/monitoring/`. `/metrics` contract regression test at `tests/unit/test_metrics_endpoint_contract.py` covering all 7 services (146 pass / 18 skip / 0 fail). Burn-rate e2e test (`tests/observability/test_alert_burn_rate_e2e.py`) deferred to live staging operator action per D-2. Observability runbook at `implementation-artifacts/pe-05-observability-runbook.md`.

### ADR-011 — Trust Center as static-rendered Next.js, not a CMS

**Status:** Accepted (Epic 18)
**Decision:** `apps/client/app/(public)/trust/page.tsx` — public route, no auth. Source: MDX + `sub-processors.yaml` in Git. PDF artefacts in `infra/trust/artefacts/` (Git-tracked for legal artefacts; WeasyPrint-generated for living artefacts). Sub-processor changes detected via Git hook → CI emits `subprocessor.changed` → `notification` service emails active customer DPAs.
**Rationale:** Git-driven workflow is auditable (which is exactly what ISO 27001 wants for change management). Boring tech beats clever CMS at weekly cadence.
**Consequences:** Code deploy on every Trust Center change — acceptable at expected cadence; revisit if pace exceeds weekly.

### ADR-012 — Pydantic `Literal[...]` / `StrEnum` mandatory for status fields; no bare `str`

**Status:** Accepted (Epic 13)
**Decision:** Any Pydantic response/request schema with a known value set MUST use `Literal[...]` or `StrEnum`. Bare `str` is a BLOCKING review finding. Generated OpenAPI exposes the enum to frontend codegen (`openapi-typescript`).
**Rationale:** Bare `str` masks unknown values, breaks frontend exhaustiveness, eliminates OpenAPI enum schema for codegen, allows test data to drift.
**Consequences:** Backend story template includes "all enumerable status fields use `Literal[...]` or `StrEnum`" as an AC. Frontend types codegen'd from OpenAPI — manual duplication is a BLOCKING finding (Epic 7).

### ADR-013 — Fire-and-forget audit writes via `asyncio.create_task` from `.finally`

**Status:** Accepted (Epic 13 Rule 45, evolved from Epic 4)
**Decision:** Audit writes are scheduled via `asyncio.create_task(_write_audit(...))` in the `finally` block of the request handler. The audit task owns its own session (`async with session.begin()`), catches all exceptions, logs `audit_write_failed` at ERROR. **Never `await write_audit()` + `await session.commit()` inline on the TTFB path.**
**Rationale:** Audit writes can add 5–50ms to every mutation; awaiting them violates the p95 <200ms NFR for high-frequency endpoints.
**Consequences:** A small window exists where the request returns 200 but the audit write fails — accepted as a logged ERROR with explicit operator alerting; correctness of the application state is not affected. Append-only `shared.audit_log` enforced via GRANT (no UPDATE/DELETE for app roles).

### ADR-014 — `asyncio.to_thread()` / `loop.run_in_executor()` for sync SDKs and CPU-bound work

**Status:** Accepted (Epic 7, Epic 8)
**Decision:**

- **CPU-bound** library calls in FastAPI (WeasyPrint, python-docx, Pillow, lxml): `await loop.run_in_executor(executor, fn, *args)`.
- **Synchronous I/O SDKs** in `async def` handlers (Stripe Python SDK, VIES SOAP, ClamAV pyclamd): `asyncio.to_thread(sync_fn, *args)`.

ATDD checklists for export/render stories must assert "function uses `run_in_executor`."
**Rationale:** Event-loop blocking is invisible in unit tests and only manifests under load — must be caught at story-design time.

### ADR-015 — Frontend response types codegen'd from backend OpenAPI; manual duplication forbidden

**Status:** Accepted (Epic 7)
**Decision:** Frontend TypeScript response types generated by `openapi-typescript` from each service's OpenAPI spec. Manual hand-rolling of Pydantic-equivalent TS types is a BLOCKING code review finding.
**Rationale:** Eliminates schema drift (Epic 7: `ProposalResponse` missing `current_version_number`/`generation_status` only caught in TEA review); makes adding fields to the contract a single-source change.
**Consequences:** CI step regenerates types and fails build on drift; PR descriptions reference both backend schema change and codegen output.

### ADR-016 — TEA test review as a story AC, not a separate "TEA backlog" story

**Status:** Accepted (Epic 14, after 10 consecutive epics of TEA under-execution)
**Decision:** TEA review is a step in the dev-story prompt template — runs before senior code review. Score ≥80/100 is required for `review → done` transition. Standalone `inj-03-tea-review-backlog`-style stories are **prohibited** — they get pre-empted by feature work (10 epics of evidence).
**Rationale:** Separate stories never execute. Embedding the review in the per-story flow is the only configuration that succeeds.

### ADR-017 — Coordinator story pattern for multi-concern hardening epics

**Status:** Accepted (Epic 13)
**Decision:** When a hardening epic has multiple independent concerns (e.g. observability, security, idempotency, perf), use a **coordinator story** that owns end-to-end AC acceptance and the close-out gate, plus narrow sub-stories that each deliver one slice. Coordinator cannot be `done` until all sub-stories' ACs are GREEN.
**Rationale:** Prevents mega-story scope explosion; preserves traceability per concern; close-out gate ensures no slice gets dropped.

### ADR-018 — AgenticSAI as agentic substrate, Topology A (singleton Org, Project-per-company)

**Status:** Accepted (2026-05-12). Supersedes §3.4 framing of "agenticsai-gateway abstraction" as the integration locus. ADR-018 narrows the gateway's responsibility from "broker every AgenticSAI call" to "broker every AgenticSAI call, hold per-tenant mapping, host webhooks, reconcile run state."

**Decision.** EU Solicit delegates its agent runtime to **AgenticSAI** (at `https://agenticsai.endigitalx.com/`). Tenancy is modelled as **Topology A**: a **single AgenticSAI Organisation** owned by EU Solicit, with **one AgenticSAI Project per EU Solicit company**. Per-Project API keys are the tenant credential boundary, stored Fernet-encrypted in `client.agenticsai_projects.api_key_encrypted`. AgenticSAI-provisioned N8N runs at the Organisation level — **one shared N8N instance** for the whole platform, workflow templates parameterised by `projectId`.

**Options considered.**

| Option | Tenancy mapping | Cost profile | Isolation | N8N | Decision |
|---|---|---|---|---|---|
| **A. Singleton Org, Project-per-company** | 1 Org + N Projects | Low — single billing line | Project-scope (agents, KB, traces, memories) | Shared (org-scoped) | **Selected** |
| B. Org-per-company | N Orgs + N Projects | High — per-Org overhead | Org-scope (incl. N8N + rate-limits + audit) | Per-tenant | Rejected — cost + provisioning surface |
| C. Hybrid by tier | Free/Starter → A; Pro/Enterprise → B | Medium | Mixed | Mixed | Rejected — two code paths, tier-upgrade migration is hostile |

**Rationale.** *Boring choice for the boring question.* Topology B is the "right" answer if money were free and AgenticSAI were our crown jewel; it isn't. We're a single-team platform with a single-host on-prem launch (ADR-010). Operating N N8N instances and N billing relationships in lockstep with EU Solicit's company lifecycle is operational debt that doesn't buy what the product needs at launch. Project-level isolation in AgenticSAI is genuine — agents, KB vector stores, traces, memories, eval-runs, policies, and run logs are all Project-scoped. The N8N concession is the only meaningful give. *Per-Project API key as tenant boundary* fits the existing Fernet pattern (ADR-009 / Epic 9). Rule of Three holds: company secrets, OAuth tokens, AgenticSAI keys — third use of the pattern is the trigger to elevate it from "vendor-specific" to "canonical Fernet vault" in the project context.

**Consequences.**

- *N8N as shared infrastructure is a known blast-radius concession.* One badly-authored workflow template can affect every tenant simultaneously. Mitigation: workflow versioning + staged rollout (canary tenants → 10% → 100%) + per-tenant feature flag gate on new template versions. **Story AC in E26**, not hand-wave. Treating workflow templates as production code (PR review, semver, rollback plan) is non-negotiable.
- *AgenticSAI is now the second critical external dependency* (after Stripe). EU Solicit's effective availability becomes `min(EU Solicit, AgenticSAI)` for any user flow that crosses the boundary. Mitigation: run-state reconciler is authoritative; async-run + job-poll model means transient AgenticSAI outages don't lose runs; graceful-degradation UX banner ("AI analysis temporarily unavailable") for outages >5 minutes.
- *Tenant provisioning becomes synchronous with company create.* Auto-create AgenticSAI Project + seed KB + register MCP stubs within 30s of company create (FR-45). Failure → reconciliation path with retry; admin-API "re-provision" endpoint for manual recovery.
- *EU residency is now an ownership-split question.* EU Solicit data on www1 is EU; AgenticSAI residency for the EU Solicit Organisation must be **contractually confirmed**. See §11.3 risk row #1 update.
- *The `agents.yaml` logical-name registry retires.* AgenticSAI Projects own agent identity natively. Logical agent names survive as an **adapter pattern**: `agenticsai-gateway` resolves `("proposal_drafter", project_id)` → AgenticSAI agent UUID via a per-Project agent-name index built at provisioning time. The lookup table lives in `client.agenticsai_projects.agent_map` JSONB.

### ADR-019 — Knowledge Base ownership split: AgenticSAI canonical for unstructured artefacts, EU Solicit canonical for structured records

**Status:** Accepted (2026-05-12)

**Decision.** EU Solicit's data tier is **split by content type**:

- **EU Solicit Postgres remains canonical for structured records**: opportunities, proposals, ESPD profiles, compliance frameworks, billing, subscriptions, users, companies, workspaces, memberships, audit log.
- **AgenticSAI storage-resources is canonical for unstructured artefacts**: tender PDFs, ESPD template documents, company profile documents (org charts, capability statements, certifications), past proposals (uploaded as reference), qualification rubrics.

Vector embeddings and parsed-text representations live exclusively in AgenticSAI; EU Solicit holds metadata pointers (`agenticsai_file_id`, `agenticsai_storage_resource_id`, `parsed_text_available_at`) but not the artefact bodies after upload.

**Options considered.**

| Option | Where artefacts live | Search | Agent grounding | Decision |
|---|---|---|---|---|
| **A. KB canonical in AgenticSAI** | AgenticSAI storage-resources only | AgenticSAI semantic search | Native (agents read KB in Project scope) | **Selected** |
| B. Dual write (EU Solicit S3 + AgenticSAI KB) | Both | AgenticSAI for semantic, S3 for raw retrieval | Native | Rejected — sync gap risk, double cost |
| C. KB canonical in EU Solicit S3 + agents pull at runtime | S3 | Custom semantic layer | Per-agent custom retriever | Rejected — re-implements what AgenticSAI does natively |

**Rationale.** *The platform that does the inference should own the index it queries.* Mirroring artefacts in EU Solicit S3 buys nothing the user notices and costs us: storage duplication, sync drift, two retention policies, two delete paths for Right-to-Erasure. AgenticSAI's storage-resources gives parsed-text download (`/files/{fileId}/parsed-text/download`) and signed-URL file download — both are sufficient for any EU Solicit-side use case (re-export, audit, legal hold). *Right-to-Erasure (GDPR Art. 17) needs an explicit cross-substrate path.* When a tenant exercises erasure: EU Solicit Postgres rows are deleted/anonymised in the existing flow; **a sibling job must call AgenticSAI `DELETE /storage-resources/{id}/files/{fileId}` for every artefact in the tenant's Project**. Story AC in E24 archival flow.

**Consequences.**

- *No more in-house parsed-text or vector indexing for unstructured content.* Saves the build of a separate retrieval substrate — and saves the bug surface that comes with it.
- *Right-to-Erasure traverses two substrates.* New cross-substrate erasure-completion proof required in audit log: `erasure_step: postgres_rows_deleted`, `erasure_step: agenticsai_files_deleted`. Single missing step = erasure not certified.
- *Data export for tenant offboarding crosses two substrates.* Export job pulls structured records from EU Solicit Postgres + iterates AgenticSAI `/storage-resources/{id}/files` to fetch artefacts and parsed text. Bundled into a single tarball. New endpoint: `POST /api/v1/companies/{id}/export-archive`.
- *Operational pain shifts to "what if AgenticSAI loses a file?"* Mitigation: pre-upload SHA-256 hash recorded in `client.agenticsai_kb_files.sha256`; periodic reconciliation job verifies AgenticSAI inventory against the EU Solicit hash table. Mismatch → admin alert. Storage-resources analogue of the run-state reconciler.
- *Local-dev story is uglier.* `make up` cannot stand up a fake AgenticSAI cheaply. Two options: (1) point local dev at AgenticSAI staging with a shared dev Org; (2) write a `agenticsai-mock` thin FastAPI that implements the dozen endpoints we care about with in-memory state. Recommendation: **(1) for early E24/E25/E26 work, (2) later when test-isolation pain forces it**. Don't pre-build the mock.

### ADR-020 — CRM integration via AgenticSAI MCP servers (Dynamics 365 + HubSpot v1; Pipedrive + Salesforce deferred)

**Status:** Accepted (2026-05-12). Supersedes §3.4 and ADR-009's CRM-related scope (HubSpot → Pipedrive → Salesforce as direct adapters in `integrations-api`). ADR-009 itself **remains accepted** for Slack/Teams + `integrations` schema + OAuth callback hosting; CRM-specific direct-adapter language is retired by this ADR.

**Decision.** CRM connectivity in v1 ships as **two MCP servers per AgenticSAI Project**: **Microsoft Dynamics 365** and **HubSpot**. Each MCP server exposes a stable tool surface — `find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note` — callable by AgenticSAI agents during qualification, lifecycle transitions, and on-demand enrichment. EU Solicit hosts the OAuth callback and stores access + refresh tokens Fernet-encrypted in `client.crm_connections` (existing table). Tokens are injected into the MCP-server configuration **at registration time** (and on rotation) via AgenticSAI's `secrets` API; AgenticSAI's secrets store becomes a downstream extension of EU Solicit's trust boundary for these credentials.

**Pipedrive** and **Salesforce** are deferred to post-launch as additional MCP-server registrations following the same pattern (no architectural change required).

**Options considered.**

| Option | Where CRM HTTP lives | Agent integration | Per-provider rate-limit state | Decision |
|---|---|---|---|---|
| **A. MCP server per provider, per Project** | AgenticSAI MCP server | Native (agent invokes MCP tool) | AgenticSAI-side | **Selected** |
| B. Direct adapter in `integrations-api` (the v1 plan) | EU Solicit `integrations-api` | Indirect — agent → agenticsai-gateway → integrations-api | EU Solicit-side, per-provider Celery queues | Rejected — agents can't enrich in-flight |
| C. N8N HTTP-request nodes in workflows | N8N (org-shared) | Workflow-step only | N8N-side | Rejected — agents need tool access, not workflow steps; credential isolation harder in shared N8N |

**Rationale.** *Agents need to enrich leads during qualification, not after it.* In Option B, qualification → workflow ends → separate enrichment step → workflow restarts → re-qualification with enriched data. Round-trip + state machinery + sync drift. In Option A, the qualification agent calls `enrich_contact` mid-run, gets richer context, and folds it into the same output. The bid/no-bid recommendation comes out the other side already-enriched. *Pipedrive + Salesforce deferral is a v1 scope call, not a permanent architectural cut.* The previous E17 plan correctly identified Salesforce as a deal-blocker for the Loopio-shaped competitive set. With on-prem launch ADR-010 in beta posture and the 8–10 week launch slip from the pivot, Salesforce is a Phase 2 add — same pattern, additional MCP server registration, no contract change. **Dynamics 365 is net-new** to the plan and reflects the rebrand opportunity for the v1 enterprise pursuit list.

**Consequences.**

- *OAuth tokens leave EU Solicit's Fernet vault into AgenticSAI's secrets store at registration.* Deliberate trust-boundary expansion: EU Solicit remains the custodian (rotation, revocation, audit) but AgenticSAI gets a copy needed for MCP-server runtime. Rotation must be **double-sided**: rotate in EU Solicit's vault → push to AgenticSAI secrets → verify reachability → revoke old token. Per FR-53.
- *Conflict resolution remains Last-Write-Wins (LWW) with audit.* Repurposes the existing `integrations.conflict_log` table; conflict source changes from "EU Solicit direct adapter vs CRM" to "AgenticSAI MCP tool call vs CRM webhook reflection." Same shape.
- *Per-provider rate limits now belong to AgenticSAI.* EU Solicit's existing per-provider circuit-breaker + Celery-queue machinery for CRM (`integrations-api` S17.00) **retires for these two providers**. The two-layer resilience pattern (ADR-004) applies to the EU Solicit → AgenticSAI call only.
- *Audit trail spans two substrates.* MCP-tool invocations are traced inside AgenticSAI (per-Project traces); EU Solicit's `shared.audit_log` records the trigger (`agent_run_initiated`, `mcp_tool_invoked_via_agent`) and the user-visible outcome. Both are retrievable.
- *Slack / Teams remain native EU Solicit integrations* (per ADR-009, unchanged). They are notification surfaces, not agent-callable tools. If we ever expose them as agent tools, that's a separate MCP-server addition — same pattern.

---

## 8. Cross-Cutting Concerns

### 8.1 Multi-Tenancy

Tenant context propagates through every layer:

1. JWT carries `company_id`, `subscription_tier`, optionally `workspace_id` for workspace-scoped routes.
2. FastAPI middleware materializes `CurrentUser` from JWT.
3. `WorkspaceScope` `Depends()` factory injects `workspace_id` into the request scope; mismatch with route `{workspaceId}` → 404 (existence leakage protection — Epic 9, Epic 14).
4. SQLAlchemy queries always filter by `workspace_id` (and `company_id` where applicable) — bare-table queries without scope are a BLOCKING review finding.
5. Stripe metadata stores `company_id` and `workspace_id` so webhook handlers can scope updates correctly.
6. **AgenticSAI Project scope** (per ADR-018) propagates as a sixth tenant-context dimension: `agenticsai-gateway` resolves `(company_id) → agenticsai_project_id + api_key` from the `client.agenticsai_projects` cache (Redis-backed, 5-min TTL, invalidated on key rotation event). Every outbound AgenticSAI call carries the per-Project bearer token; **a request with a mismatched Project token is a server-side bug, not a tenant-isolation violation** — the api-key is the tenant boundary at the AgenticSAI side.
7. **KB artefact upload scope**: artefact uploads from `client-api` to AgenticSAI `storage-resources` use the calling company's Project api-key. `client.agenticsai_kb_files` rows carry `company_id` and are scoped by the existing RBAC `Depends()` factories (Epic 2 pattern, unchanged).
8. **MCP-tool invocations** carry the Project scope implicitly — agents run inside the Project; MCP tools the agent calls execute against that Project's MCP-server config (which holds *this* tenant's OAuth tokens, not a sibling tenant's). Cross-tenant negative tests for MCP invocations are a story AC in E27.

### 8.2 Authentication & Authorization Flow

```
Browser                 Next.js middleware           client-api
   │                          │                          │
   │  POST /auth/login        │                          │
   ├─────────────────────────►│                          │
   │                          ├─────────────────────────►│
   │                          │   verify creds           │
   │                          │   (User.is_active req.)  │
   │                          │                          │
   │                          │◄─────────────────────────┤
   │                          │   JWT + refresh cookie   │
   │◄─────────────────────────┤                          │
   │                          │                          │
   │  GET /workspace/X/...    │                          │
   ├─────────────────────────►│                          │
   │                          │  middleware: pre-check   │
   │                          │  (route requires auth?)  │
   │                          ├─────────────────────────►│
   │                          │                          │ Depends(get_current_user)
   │                          │                          │    → JWT verify (RS256)
   │                          │                          │    → load User, check is_active
   │                          │                          │ Depends(WorkspaceScope)
   │                          │                          │    → load membership, role
   │                          │                          │    → 404 if mismatch
   │                          │                          │ Depends(TierGate)
   │                          │                          │    → check subscription_tier
   │                          │                          │    → 402 / 403 if below floor
   │                          │                          │ Depends(check_entity_access)
   │                          │                          │    → role ceiling vs entity perm
   │                          │                          │ → handler
   │                          │                          │ → fire-and-forget audit
```

### 8.3 Configuration & Secrets

- Each service subclasses `BaseServiceSettings` (pydantic-settings) with its own `env_prefix` (e.g. `CLIENT_API_`, `AI_GATEWAY_`).
- `get_settings()` returns a cached singleton.
- Secrets sourced from env vars (production: AWS Secrets Manager via External Secrets Operator) or `.env` (local).
- Negative-path config tests required (Epic 4 anti-pattern: missing-env-var paths untested).

### 8.4 Internationalization

- Backend produces machine codes (`AGENT_UNAVAILABLE`, `WORKSPACE_NAME_TAKEN`); frontend resolves to user-facing strings via `next-intl`.
- BG and EN required at MVP; locale carried in URL (`/[locale]/...`).
- `pnpm check:i18n` is a CI gate; `no-literal-text` ESLint rule enforced for `apps/client` and `apps/admin` (Epic 11 anti-pattern: hardcoded English in fix passes).

### 8.5 Accessibility (WCAG 2.1 AA)

- shadcn/ui + Radix Primitives baseline meets most requirements.
- Mandatory: 4.5:1 contrast for body text, keyboard navigation across all interactive elements (esp. split-pane editor), `aria-live` regions for dynamic state announcements ("AI generation started", "Section locked by Elena", "Compliance check complete: 3 issues found"), color independence (status icons + text), `prefers-reduced-motion` respected.
- Playwright accessibility checks via `@axe-core/playwright` on critical pages.

### 8.6 Performance Engineering

- p95 <200ms REST, TTFB <500ms SSE, Lighthouse ≥90 — all enforced via k6 baseline (Epic 21 PE.01) and Prometheus histograms in CI.
- ThreadPoolExecutor for CPU-bound (Epic 7) / `asyncio.to_thread` for sync I/O (Epic 8) — both event-loop-blocking is invisible without load tests.
- Materialized view refresh CONCURRENTLY (no read-locks).
- Over-fetch + scope-filter pattern for related items: fetch N, filter to max via `is_in_scope()` (Epic 6 — avoids complex scoped-count SQL when result set is small).
- Frontend: URL-driven filter/sort/pagination state via `useSearchParams()` + `router.replace()` with `useRef(onChange)` for stale-closure prevention; zero `useState` for URL-representable state (Epic 6).

---

## 9. Project Structure

```
eusolicit-app/
├── Makefile                           # All make targets (test, infra, migrate, lint, coverage)
├── docker-compose.yml                 # Local dev: postgres, redis, minio, clamav
├── pyproject.toml                     # Root Python project + ruff config
├── ruff.toml                          # Python 3.12 target, line 120, rules I E W F UP
│
├── services/
│   ├── client-api/                    # :8001 — primary user-facing FastAPI
│   ├── admin-api/                     # :8002 — internal admin FastAPI
│   ├── data-pipeline/                 # :8003 — Celery + FastAPI ingestion
│   ├── agenticsai-gateway/               # :8004 — AgenticSAI broker (renamed 2026-05-12 from agenticsai-gateway)
│   ├── notification/                  # :8005 — email/Slack/Teams + materialized-view refresh
│   ├── enterprise-api/                # public REST proxy (gateway routes through client-api)
│   └── integrations-api/              # :8007 — CRM + Slack/Teams (Epic 16/17)
│
├── packages/                          # Shared Python packages
│   ├── eusolicit-common/              # BaseServiceSettings, structlog, EventBus, middleware
│   ├── eusolicit-models/              # Cross-service Pydantic DTOs, Redis Stream event schemas
│   ├── eusolicit-agenticsai/             # Typed AgenticSAI client (renamed 2026-05-12 from eusolicit-agenticsai)
│   └── eusolicit-test-utils/          # Canonical fixtures (UserFactory, ServiceClient, etc.)
│
├── frontend/
│   ├── apps/
│   │   ├── client/                    # :3000 — Next.js 14 customer app
│   │   └── admin/                     # :3001 — Next.js 14 admin app (IP-allowlisted)
│   └── packages/
│       ├── ui/                        # @eusolicit/ui — shadcn/ui design system
│       └── config/                    # ESLint/Prettier/TS configs
│
├── tests/                             # Cross-service pytest (unit | integration | api | cross_service)
├── e2e/                               # Playwright (chromium, firefox, webkit)
├── infra/
│   ├── nginx/                         # host-nginx config for www1 (per ADR-010)
│   ├── postgres/                      # init scripts (schema + role provisioning)
│   ├── n8n-templates/                 # AgenticSAI N8N workflow JSON, semver-tagged (per §3.4.1)
│   ├── stripe-config.yaml             # Stripe Price IDs, products, tiers
│   └── trust/artefacts/               # Trust Center PDFs (legal-curated, Git-tracked)
│   # NOTE: helm/ + terraform/ removed in 2026-05-11 on-prem pivot (ADR-010);
│   # AWS-specific cutover runbooks retained in implementation-artifacts/ as historical.
│
└── _bmad/                             # BMAD pipeline state (sprint-status.yaml etc.)
```

**Naming conventions:**

- Python: `snake_case`; modules under `services/<svc>/<svc>/...` (mirroring package name).
- TypeScript: `kebab-case` for files, `PascalCase` for components, `camelCase` for hooks/utils.
- Migrations: numbered (`0046_workspace_unique_active.py`), descriptive, atomic.
- Test markers: `@pytest.mark.unit | integration | api | cross_service`.

---

## 10. Implementation Patterns (Living)

### 10.1 Source of Truth

Patterns and anti-patterns are codified in `eusolicit-docs/planning-artifacts/project-context.md` and `eusolicit-docs/project-context.md`, generated from epic retrospectives. These files are **the** definitive guide for implementation details. **All AI agents and human developers MUST treat them as authoritative.** This architecture document and the ADRs above are the framework; project-context is the lived practice.

### 10.2 High-Frequency Patterns (from Epics 1–15 retros)

**Backend**

- Two-layer resilience: `circuit_breaker(retry(http_factory))` — universal for outbound HTTP.
- Atomic Redis Lua for all metering operations.
- Webhook dedup table with `event_id` UNIQUE — never in-memory state.
- Tier cache invalidation via DELETE, not SET.
- `asyncio.to_thread()` for sync SDKs; `loop.run_in_executor()` for CPU-bound.
- Fire-and-forget audit via `asyncio.create_task` from `.finally`.
- Single-point SSE error sanitization helper per endpoint.
- Reset-stuck background task (Celery Beat) for any resource with an in-progress status field.
- Partial-unique WHERE predicate for soft-delete uniqueness; `IntegrityError → 409` translation in service layer.
- `Literal[...]` / `StrEnum` for all enumerable status fields.
- Workspace-scoped Redis keys with company-scoped fallback (Epic 15).
- Explicit literal lists, not `list(frozenset)`, for SQL `IN()` and display ordering.

**Frontend**

- `<QueryGuard>` wraps every TanStack Query usage.
- `useZodForm` + `<FormField>`; retry routes through `form.handleSubmit()`.
- URL-driven filter/sort/pagination state.
- Native `fetch` + `ReadableStream` for SSE POST.
- `cancelledIdsRef` checked at every async boundary in multi-step uploads.
- `<Select>` / `<Dialog>` / `<Sheet>` / `<Tabs>` from `@eusolicit/ui` only — no native equivalents.
- Codegen response types from OpenAPI — never hand-roll.

**Testing**

- testcontainers Postgres + Redis + respx for backend integration tests.
- Vitest source-inspection ATDD for complex multi-panel components; RTL/JSDOM only for user-interaction flows.
- Playwright E2E with quoted `N passed` summary in completion notes — `test.skip ≥80%` is NONE coverage, not PARTIAL.
- Canonical fixtures from `eusolicit-test-utils` mandatory; bespoke rebuilds are BLOCKING.
- structlog stdlib routing fixture in every service's `conftest.py`.
- Cross-tenant negative tests (403/404) for every cross-tenant endpoint.

### 10.3 Anti-Patterns (BLOCKING on detection)

(Selected from project-context — see source for full list.)

- Bespoke test fixtures rebuilt despite canonical helpers in `eusolicit-test-utils`.
- Story marked `done` in sprint-status without `Status: done` in story file AND `REVIEW: Approve` in review section.
- Frontend response types manually duplicated from backend Pydantic (must codegen from OpenAPI).
- Documentation claims a fix or test exists when it doesn't on disk (review must `grep` the File List).
- Deleting out-of-scope files in a review-fix pass without running an import smoke test.
- Hardcoded English in fix passes (use `no-literal-text` ESLint rule).
- Native `<select>` instead of `<Select>` from `@eusolicit/ui`.
- E2E spec marked done without execution evidence (need quoted Playwright `N passed`).
- TEA review absent (must be a story AC, not a backlog item).
- Carry-forward stories deprioritized behind new feature work — must be P0 priority and execute first.

### 10.4 Enforcement

- **CI gates** (zero-tolerance on `main`): ruff, mypy, eslint, tsc, `pnpm check:i18n`, `no-literal-text`, ≥80% coverage, codegen-drift check.
- **Story-level gates**: ATDD GREEN before `done`, TEA review ≥80/100, senior code review `Approve` with quoted execution output.
- **Sprint-level gates**: orchestrator story-close verifies story-file status before writing sprint-status.
- **Epic-level gates**: NFR assessment + retrospective at close; carry-forward `[ACTION]` items create verification tasks for the next epic's kickoff.

---

## 11. Validation, Risks, and Open Carry-Forwards

### 11.1 Architecture validation

- **Coherence:** Decisions, technology stack, schema layout, and patterns are mutually consistent. Epics 1–15 have shipped on this architecture without redesign — five extensions of existing patterns; no foundational rewrites.
- **Requirements coverage:** Every PRD FR maps to a service or ADR (FR-1..14 → `client-api` + Stripe; FR-15..20 → `data-pipeline` + `client-api` + N8N workflows; FR-21..25 → `agenticsai-gateway`; FR-26..33 → `client-api` + `agenticsai-gateway`; FR-34..39 → `client-api` + `admin-api`; FR-40..44 → `notification` + `admin-api`; FR-45..55 → `agenticsai-gateway` + `client-api` + `integrations-api` per ADR-018/019/020). NFRs are addressed via specific ADRs (NFR-1/2 → ADR-005, ADR-014; NFR-7 → ADR-001/002/007; NFR-14 → ADR-010; NFR-15/17 → §6.5; NFR-21..23 → §6.3, §10; NFR-24..26 → ADR-018 + §11.3 risks #11/#12).
- **Implementation readiness:** Mature project context, canonical fixtures, established CI gates. Ready.
- **2026-05-04 re-validation pass (autopilot):** Re-checked v2.0 against four post-2026-04-27 sprint-change proposals (v17 of 04-30; v18 of 05-03; v19 of 05-04) and IR-v3 (05-03). All four documents explicitly state "no PRD/architecture/epics/ux-spec edits required" — pure sequencing/process interventions. Epic 18 (Trust Center) implementation introduces `eusolicit_common.document_generation.weasyprint_renderer`, `infra/trust/artefacts.yaml`, `eusolicit_common.aws.s3_client`, and the public route `GET /api/v1/trust/artefacts/{slug}` (302 → signed S3) — all conform to ADR-011 (static-rendered Trust Center) and the existing public-route ingress + S3 artefact pipeline already documented in §5.1 / §6.5; no new ADR required. Sole architecture-adjacent risk note: WeasyPrint, markdown-it-py, python-frontmatter shipped unscanned because `inj-01` (Dependabot configuration) remains a deferred carry-forward — already tracked in §11.2 item 2 and §11.3 item N/A; remains a process/tooling gate, not an architecture change. **No content changes to §1–§10 required.**
- **2026-05-12 AgenticSAI pivot pass (v3.0 consolidation):** Re-validated coherence and requirements coverage against the post-pivot architecture. PRD amendment (`prd-amendment-2026-05-12-agenticsai.md`) introduces FR-45 through FR-55 and NFR-24 through NFR-26; each maps to a service/epic/ADR per the traceability matrix in that document. Architecture readiness for the post-pivot scope is **DEFERRED PENDING `bmad-check-implementation-readiness`** against the modified E04/E05/E11/E17 + new E24-E28 epic set. No retreat from the underlying invariants (schema isolation, two-layer resilience, per-route Depends(), SSE lifecycle, fire-and-forget audit, event-bus discipline) — the pivot reshapes the upstream surface, not the platform's spine. Cross-document consistency check: PRD FR-45 ↔ ADR-018 + `client.agenticsai_projects` ✓; FR-49..52 (KB) ↔ ADR-019 + `client.agenticsai_kb_files` ✓; FR-53 (CRM via MCP) ↔ ADR-020 + `client.agenticsai_mcp_servers` ✓; FR-54..55 (webhooks + reconciler) ↔ ADR-018 + `gateway.webhook_subscriptions` + `gateway.workflow_runs` ✓; NFR-24 (key rotation) ↔ §4.4 + `api_key_rotated_at` ✓; NFR-26 (AgenticSAI as external dep) ↔ §11.3 risk #12 ✓.

### 11.2 Open carry-forwards (highest priority)

Five items have crossed multiple epic boundaries and now block the next NFR/SLA milestone. Locked as Epic 21 (Platform Reliability):

1. **k6 performance baseline** — deferred 6 epics. Cannot publish 99.9% SLA, cannot validate p95 <200ms / TTFB <500ms.
2. **Dependabot configuration** — deferred 6 epics. Stripe SDK, Stripe.js, VIES, WeasyPrint, Tiptap, recharts, dnd-kit, pyclamd, boto3, fakeredis, testcontainers all unscanned.
3. **Stripe outbound circuit breaker** — Epic 4 pattern not extended to `billing_service.py` / `vies_service.py`.
4. **Billing Prometheus metrics** — webhook latency, sync-drift gauge, Stripe error counter, active-tier gauge, trial-to-paid counter.
5. **R-001 optimistic-locking GREEN test verification** — implementation exists; tests exist; some not GREEN.

### 11.3 Risks (from architecture-evaluation-2026-04-25 §8)

| # | Risk | Mitigation |
|---|---|---|
| 1 | **AgenticSAI EU data residency / GDPR sub-processor evidence** (2026-05-12 update) | **Launch-blocking**: contractually confirm EU-only data residency for the EU Solicit Organisation, including storage-resources, traces, and N8N execution. Add AgenticSAI to the sub-processor list with the confirmed residency posture. ADR-010 on-prem pivot's GDPR rationale is wasted if AgenticSAI residency cannot be confirmed |
| 2 | Vector store cost scaling under per-workspace partitioning | Per ADR-019, vector storage is AgenticSAI-side; confirm AgenticSAI billing model; consider tenant-scoped vector stores with workspace-tagged content as fallback |
| 3 | External collaborator GDPR data flow (DPA chain-of-processing) | Update DPA template; flag in legal review |
| 4 | Stripe per-bid SKU + EU VAT MOSS reporting | Confirm Stripe Tax product configuration |
| 5 | BGN settlement preference for BG buyers | Confirm Stripe currency setup early in Epic 15 |
| 6 | ARR vs. one-time revenue tracking distinction | Add reporting category to internal billing dashboards |
| 7 | Workspace deletion vs. GDPR Right to Erasure | Document Art. 17.3.b legal-obligation exception; ensure audit entries reference IDs not free-form PII |
| 8 | Cross-workspace data leakage in shared content blocks | `tenant_scope=true` only on explicit user opt-in; PII detection at upload (Epic 19/20) |
| 9 | k6 baseline closure on critical path | First Epic 21 story (PE.01) |
| 10 | CRM provider rate-limit surprises at scale (Slack/Teams; CRM moved to AgenticSAI MCP per ADR-020) | Per-provider Celery queue with backoff for Slack/Teams; "rate-limit reached, paused" UX state |
| 11 | **N8N org-scope blast radius** (per ADR-018) | One badly-authored workflow template can affect every tenant. Mitigation: workflow versioning (semver) + canary-tenant + 10% + 100% staged rollout + per-tenant feature flag gate on new template versions. **Story AC in E26**, not hand-wave |
| 12 | **AgenticSAI as second critical external dependency** | Effective availability = `min(EU Solicit, AgenticSAI)`. Mitigations: run-state reconciler is authoritative; async-run + job-poll preserves runs across transient outages; tenant-visible degraded-mode banner on outages >5 minutes; AI summary paths fail-open with degraded result, payment paths remain fail-closed |
| 13 | **AgenticSAI secrets-store as extended trust boundary** (per ADR-020) | OAuth tokens for Dynamics + HubSpot leave EU Solicit's Fernet vault into AgenticSAI secrets at MCP registration. Mitigations: rotation is double-sided (EU Solicit vault → AgenticSAI push → verify → old-key revoke); audit log records both sides; periodic verification job (weekly) confirms AgenticSAI MCP-server config reachability |
| 14 | **Cross-substrate Right-to-Erasure completion** (per ADR-019) | Erasure spans EU Solicit Postgres + AgenticSAI storage-resources. Risk: partial erasure → compliance breach. Mitigation: two-ACK audit-log pattern (`erasure_step: postgres_rows_deleted`, `erasure_step: agenticsai_files_deleted`); erasure not certified until both present; nightly job sweeps for stale erasure rows missing the second ACK |
| 15 | **Tenant provisioning failure modes** (per ADR-018, FR-45) | AgenticSAI unavailable at company create → `provisioning_status='pending'` with retry. Bounded retry window (24h); after that, admin-flagged for manual recovery. UX: trial-tier signups can land in `pending` and degrade gracefully (no AI features until provisioned), paid signups must block at billing until provisioning succeeds |

---

## 12. Architecture Readiness Assessment

| Item | Status |
|---|---|
| **Overall Status** | **READY FOR IMPLEMENTATION** |
| **Confidence** | High — battle-tested across 15 shipped epics |
| **Strengths** | Proven patterns; coherent stack; living project-context; strict gate discipline (when followed) |
| **Watch-outs** | Carry-forward backlog; sprint-status integrity; TEA execution discipline |
| **Net-new for v2.0** | Workspaces (Epic 14), Per-bid SKU + Pro+ (Epic 15), `integrations-api` (Epics 16/17), Trust Center + ISO 27001 prep (Epic 18 + parallel programme), Outcome Telemetry (Epic 19), NPS (Epic 20), Platform Reliability (Epic 21) |
| **Net-new for v3.0 (2026-05-12 AgenticSAI pivot)** | ADR-018 AgenticSAI Topology A; ADR-019 KB ownership split; ADR-020 CRM via MCP servers; ADR-004 AgenticSAI addendum; `agenticsai-gateway`; 5 new tables (`client.agenticsai_projects`, `client.agenticsai_kb_files`, `client.agenticsai_mcp_servers`, `gateway.webhook_subscriptions`, `gateway.workflow_runs`); Standard Webhooks receiver + 5-min run-state reconciler; N8N templates as production code with semver+canary rollout (E04/E05/E11/E17 refactor + new E24-E28) |

### Implementation Handoff — for AI Agents and Developers

**Primary directive:** Adhere strictly to the ADRs in this document AND the patterns in `project-context.md` files.
**Conflict resolution:** When this document and `project-context.md` disagree, **`project-context.md` takes precedence for implementation details** — it is generated from lived experience. Discrepancies must be raised in the next epic retrospective and resolved by amending one or both documents.
**Process:** Standard BMAD development workflow — Implementation Readiness check before each epic, Validate Story before each story, Story Review after each story in multi-story epics, Epic Review for complex/interdependent epics, Post-Review after code review for all epics.

---

**End of Architecture Document v3.0** (2026-05-14 consolidation of v2.0 + 2026-05-12 AgenticSAI amendment). Prior v2.0 archived to `architecture.v2.0.bak.md`. Standalone amendment at `architecture-amendment-2026-05-12-agenticsai.md` is now historical — all content folded into the sections above.
