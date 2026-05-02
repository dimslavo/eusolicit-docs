---
title: "EU Solicit — Architecture Document"
status: "complete"
date: "2026-04-27"
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
workflowType: 'architecture'
lastStep: 8
completedAt: '2026-04-27'
version: "v2.0"
inputDocuments:
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/PRD.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/ux-spec.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/project-context.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/project-context.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/architecture-evaluation-2026-04-25.md"
  - "/home/debian/Projects/eusolicit/eusolicit-docs/planning-artifacts/prd-amendment-2026-04-25.md"
---

# EU Solicit — Architecture Document

**Author:** Winston (System Architect)
**Date:** 2026-04-27
**Status:** Living document — updated through epic retrospectives. v2.0 supersedes v1.x and incorporates the 2026-04-25 PRD amendment (multi-client workspaces, per-bid SKU, Pro+ tier, integrations-api, Trust Center, 99.9% SLA, outcome telemetry).

---

## 1. System Overview

EU Solicit is a multi-tenant SaaS platform that automates the lifecycle of EU public procurement and EU grant applications: opportunity discovery, bid/no-bid decisioning, AI-assisted proposal drafting, compliance checking, scoring simulation, collaborative editing, ESPD generation, and submission export.

### 1.1 Architectural Style

- **Domain-driven microservices** behind a single public API surface, sharing one PostgreSQL instance with **schema-per-service logical isolation**.
- **Event-driven asynchrony** via Redis Streams (custom `EventPublisher`/`EventConsumer` abstraction with at-least-once delivery and DLQ).
- **Human-in-the-loop AI** orchestration: an AI Gateway brokers all calls to KraftData's Agentic AI platform (multi-agent system); the user always reviews/approves AI output.
- **Multi-tenant by design**: every tenant-bound row carries `company_id` (tenant root) and (post-Epic 14) `workspace_id` (sub-tenant for client-engagement isolation by consulting firms).
- **Strict separation of customer-facing and internal admin surfaces**: a separate `admin-api` + `apps/admin` deployment, IP-allowlisted and MFA-gated.
- **Resilient outbound integration**: every external HTTP call is wrapped in `circuit_breaker(retry(http_factory))` with explicit `httpx` timeouts.
- **Observability-first**: structlog JSON → Loki, Prometheus metrics, Jaeger traces. Every service exports `/metrics`, `/health`, and `/admin/*` introspection endpoints.

### 1.2 Architectural Drivers (from PRD)

| Driver | Source | Implication |
|---|---|---|
| AI generation TTFB < 500ms (NFR-2) | PRD §NFR | SSE streaming, single-point error sanitization, pre-generation quota check before HTTP 200 headers |
| API p95 < 200ms (NFR-1) | PRD §NFR | Async I/O end-to-end, Redis cache for tier policy, fire-and-forget audit writes |
| 99.5% MVP / 99.9% post-amendment uptime (NFR-14) | PRD §NFR + amendment | Stateless services, HPA + PDB, managed Postgres Multi-AZ + Redis Sentinel (Epic 21) |
| EU-only data residency (Domain-Specific) | PRD §Domain | AWS eu-central-1 single-region; KraftData EU-region storage resources only |
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
                                       (CloudFront / CloudFlare CDN)
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
                  │              AWS ALB / Ingress (TLS 1.3)      │         │
                  └──┬────────┬─────────┬────────┬────────┬───────┘         │
                     │        │         │        │        │                 │
                     ▼        ▼         ▼        ▼        ▼                 │
              ┌──────────┐┌────────┐┌────────┐┌──────┐┌──────────┐         │
              │client-api││admin-  ││ai-     ││data- ││integ-    │         │
              │  :8001   ││api     ││gateway ││pipe- ││rations-  │◄────────┤
              │ FastAPI  ││:8002   ││:8004   ││line  ││api :8007 │         │
              │ + Stripe │└────────┘└───┬────┘│:8003 │└────┬─────┘         │
              │ + RBAC   │              │      └──┬───┘     │               │
              │ + Tier   │              │         │         │               │
              │   Gates  │              │         │         │               │
              └────┬─────┘              │         │         │               │
                   │            ┌───────▼─────┐   │         │               │
                   │            │  KraftData  │   │         │               │
                   │            │ Agentic AI  │   │         │               │
                   │            │ (EU region) │   │         │               │
                   │            └─────────────┘   │         │               │
                   │                              │         │               │
                   │                       ┌──────▼─────┐   │               │
                   │                       │  AOP / TED │   │               │
                   │                       │  crawlers  │   │               │
                   │                       │ (Celery)   │   │               │
                   │                       └────────────┘   │               │
                   │                                  ┌─────▼──────┐        │
                   │                                  │ HubSpot /  │        │
                   │                                  │ Salesforce │        │
                   │                                  │ Pipedrive /│        │
                   │                                  │ Slack /    │        │
                   │                                  │ MS Teams   │        │
                   │                                  └────────────┘        │
                   │                                                        │
                   ▼                                                        │
       ┌──────────────────────┐         ┌──────────────────────────┐       │
       │  notification :8005  │◄────────┤ Redis Streams            │       │
       │  Email / Slack /     │         │ (events)                 │       │
       │  Teams / SP-change / │         │ + DLQ                    │       │
       │  onboarding alerts   │         └──────────┬───────────────┘       │
       └──────────┬───────────┘                    │                       │
                  │                                │                       │
                  │   ┌────────────────────────────┴─────────────────┐    │
                  │   │              Redis 7 (single logical cluster) │◄──┘
                  │   │  - Caching (tier policy, RBAC, session)        │
                  │   │  - Rate limiting + atomic usage Lua            │
                  │   │  - Event Bus (Streams) + DLQ                   │
                  │   │  - Idempotency keys (SETNX)                    │
                  │   └────────────────────────────────────────────────┘
                  │
                  │
                  ▼
        ┌─────────────────────────────────────────────────────────────────┐
        │              PostgreSQL 16 (Multi-AZ — Epic 21)                 │
        │  Schemas (logical isolation; cross-schema queries forbidden):   │
        │  ┌──────────┬──────────┬──────────┬─────────┬────────────────┐  │
        │  │  client  │  admin   │ pipeline │ gateway │  notification  │  │
        │  ├──────────┴──────────┴──────────┴─────────┴────────────────┤  │
        │  │  integrations  (NEW Epic 17)                              │  │
        │  ├───────────────────────────────────────────────────────────┤  │
        │  │  shared (audit_log, lookups)                               │  │
        │  └───────────────────────────────────────────────────────────┘  │
        │  Roles: client_api_role, admin_api_role, gateway_role,           │
        │         pipeline_role, notification_role, integrations_role,     │
        │         migration_role (DDL only)                                │
        └─────────────────────────────────────────────────────────────────┘

        ┌─────────────────────────────────────────────────────────────────┐
        │   Object storage:  Amazon S3 (eu-central-1)                     │
        │   - Tender documents (ClamAV scanned before access)             │
        │   - Generated proposals (PDF/DOCX exports)                      │
        │   - Trust Center artefacts (signed URLs)                        │
        └─────────────────────────────────────────────────────────────────┘
```

### 2.2 Service Inventory

| Service | Port | Schema(s) owned | Purpose |
|---|---|---|---|
| `client-api` | 8001 | `client` (rw), `pipeline` (read-only via dual-session) | Primary user-facing API: auth, RBAC, workspaces, opportunities (read), proposals, ESPD, billing, calendar, analytics, per-bid metering |
| `admin-api` | 8002 | `admin` (rw), `client` (read-only) | Internal admin portal API: tenant management, compliance frameworks, crawler oversight, pricing-tier configuration |
| `data-pipeline` | 8003 | `pipeline` (rw) | Celery-driven crawlers (AOP, TED), document ingestion, opportunity matching, enrichment queue |
| `ai-gateway` | 8004 | `gateway` (rw) | Single broker for KraftData & LLM. Agent registry (`config/agents.yaml`), circuit breaker, rate limiter, SSE streaming, `X-Caller-Service` header |
| `notification` | 8005 | `notification` (rw) | Redis Streams consumer. Email/in-app/Slack/Teams delivery, sub-processor change DPAs, onboarding-stall alerts, materialized-view refresh orchestration |
| `integrations-api` | 8007 | `integrations` (rw), `client.crm_connections` (read-only) | NEW (Epic 16/17). HubSpot/Salesforce/Pipedrive bi-directional sync; Slack/Teams incoming-webhook templates; OAuth token vault |
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
- `eusolicit-kraftdata` — Typed client for KraftData Agentic AI.
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

### 3.4 AI Platform

| Component | Choice | Rationale |
|---|---|---|
| **KraftData Agentic AI** | external managed | Multi-agent orchestration (Document Parser, Summarizer, Draft Generator, Compliance Checker, Score Simulator), RAG with EU-region vector storage, GDPR sub-processor with signed DPA |
| **AI Gateway abstraction** | in-house (`ai-gateway` service) | Single broker. Agents addressed by **logical name** (e.g. `proposal_drafter`) via YAML registry (`config/agents.yaml`), decoupling code from external UUIDs (Epic 4 pattern). Logical name + circuit-breaker key + rate-limit bucket all align |
| **Calling convention** | `AiGatewayClient.run_agent(agent_name, payload)` | Frozen Epic 11 standard. Flat 503 body `{"message": "...", "code": "AGENT_UNAVAILABLE"}`, `X-Caller-Service` header, configurable timeout, **no client-side retry** (gateway owns retry/backoff). Deviations block code review approval |
| **Streaming** | SSE (`text/event-stream`) | TTFB < 500ms. Single-point error sanitization helper per endpoint (Epic 13 pattern). Quota check **before** `StreamingResponse` creation; generator closed in `finally`; terminal event guaranteed; `except asyncio.CancelledError: raise` to preserve cancellation semantics (Python 3.8+ — Epic 13) |

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
| `gateway` | `ai-gateway` | Agent execution logs, prompt template versions, evaluation scorecards, rate-limit & circuit-breaker state mirrors |
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
-- AI GATEWAY (Epic 4)
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
```

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
| AOP / TED / KraftData / Stripe / VIES / Google OAuth / HubSpot / Salesforce / Pipedrive / Slack / MS Teams | Outbound | Provider-specific | All wrapped in `circuit_breaker(retry(http_factory))` with explicit `httpx` timeout |

### 5.2 Internal APIs

| Caller | Callee | Mechanism | Notes |
|---|---|---|---|
| `client-api`, `admin-api` | `ai-gateway` | Internal REST (cluster DNS) | `AiGatewayClient.run_agent(name, payload)` + `X-Caller-Service` header. Flat 503 body `{"message", "code"}` (Epic 11 standard) |
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

**Event handler discipline** (project-context):

- Narrow `except` clauses — never swallow `celery.exceptions.Retry` or `asyncio.CancelledError` (Epic 9, Epic 13).
- Idempotency mandatory: every handler keys on a stable event_id, uses dedup table or SETNX.
- 4xx errors must NOT increment circuit-breaker failure counters (Epic 5 OBS-001 fix).

---

## 6. Deployment Topology

### 6.1 Environments

| Environment | Purpose | Notes |
|---|---|---|
| `local` | Developer laptops | `make up` brings up all services + infra via docker-compose. Postgres :5432, Redis :6379, MinIO :9000, ClamAV :3310 |
| `dev` | Shared integration / preview | Per-PR preview deployments via GitHub Actions |
| `staging` | Pre-production | Full prod parity with reduced replica counts; sanitized prod data snapshots |
| `production` | Customer-serving | AWS eu-central-1 (hard residency requirement, NFR + Domain) |

### 6.2 Production Topology (eu-central-1)

- **Region**: AWS eu-central-1 (Frankfurt) — single region; data residency requirement permits no cross-region replication outside EU.
- **Compute**: Amazon EKS (Kubernetes 1.29+); each microservice as a Helm chart in `infra/helm/`; provisioned via Terraform (`infra/terraform/`).
- **Ingress**: AWS ALB → nginx-ingress (≥2 replicas, PDB `minAvailable: 1`). TLS 1.3 termination at ALB; HSTS preload.
- **Service replicas (production)**: minimum 2 replicas per service; HPA on CPU/memory/RPS; `PodDisruptionBudget` `minAvailable: 1` mandatory (Epic 21).
- **Database**: Amazon RDS for PostgreSQL 16 Multi-AZ (Epic 21 — replaces single-instance). 1 read replica for analytics workloads. Daily automated snapshots; PITR window 7 days.
- **Redis**: Amazon ElastiCache Redis 7 with Redis Sentinel (1 primary + 2 replicas). Separate logical DB indices: `0` for application cache/streams, `1` for tests.
- **Object storage**: S3 with bucket-level KMS encryption (AES-256), lifecycle rules archiving generated exports >90 days to S3 Glacier IR.
- **Secrets**: AWS Secrets Manager for DB credentials, Stripe keys, KraftData API keys; injected via External Secrets Operator into K8s `Secret` objects.
- **CDN**: CloudFront in front of `apps/client` and `apps/admin` Next.js builds; `/trust/*` cached aggressively.
- **DNS**: Route53.

### 6.3 CI/CD (GitHub Actions)

Pipelines in `.github/workflows/`:

1. **PR pipeline**: matrix lint (ruff, mypy, eslint, tsc, `pnpm check:i18n`, `no-literal-text`) → unit tests → integration tests (testcontainers) → Vitest → Playwright (chromium first, then full matrix on `main`) → `make coverage` (≥80% gate) → docker image build → preview deployment.
2. **`main` pipeline**: full matrix → image push to ECR → Helm chart bump → ArgoCD reconcile to `staging` → integration smoke → manual approval → reconcile to `production`.
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
  - Outbound provider call latency + error rate (Stripe, KraftData, VIES, AOP, TED, CRMs)
  - Circuit-breaker state gauge per provider
  - Webhook processing latency histogram (Epic 8)
  - `billing_usage_sync_drift_total` gauge, active-tier distribution gauge, trial-to-paid conversion counter (Epic 8 retro)
  - Pipeline throughput (24h freshness SLA — Epic 5)
- **Dashboards**: Grafana per service + cross-service SLO dashboards with error-budget burn-rate alerting.
- **Logging**: structlog JSON → stdout → Promtail → Loki. Mandatory fields: `service`, `request_id`, `correlation_id`, `actor_user_id`, `company_id`, `workspace_id`. Never log secrets; never `str(exc)` in user-facing 5xx responses (Epic 5 anti-pattern: connection-string leakage).
- **Tracing**: OpenTelemetry → Jaeger; instrument FastAPI middleware, SQLAlchemy, httpx, Celery task boundaries.
- **Alerting**: PagerDuty rotation (Epic 21 PE.06). SLO burn-rate alerts (1h fast-burn at 14.4× budget; 6h slow-burn at 3×).

### 6.5 Data Backup, DR, Residency

- **Backups**: RDS automated daily snapshots; PITR 7 days. Cross-AZ replication automatic. **No cross-region replication** (residency).
- **DR**: documented runbook (Epic 21 PE.06). RTO ≤ 4h (NFR-17); RPO ≤ 24h (NFR-15). Annual DR test mandatory.
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

**Status:** Accepted (Epic 4, reused Epic 5/8/9/15)
**Decision:** All outbound HTTP calls (KraftData, Stripe, VIES, AOP, TED, HubSpot, Salesforce, Pipedrive, Slack, Teams, calendar) use composed resilience: outer **circuit breaker** wrapping inner **exponential backoff retry** wrapping the typed `httpx` client factory. Explicit `httpx` timeouts mandatory.
**Rationale:** Keeps protections layered correctly — circuit breaker counts only network/5xx failures (4xx must NOT increment failure counter, Epic 5 OBS-001), retry handles transient network hiccups, factory ensures a single typed client per provider.
**Consequences:** Stripe must adopt this pattern in `billing_service.py` and `vies_service.py` (Epic 8 carry-forward). Open-circuit fallback varies per call: payment paths fail-closed; AI-summary paths fail-open with degraded result; VIES fails-open to `vat_validation_status: pending`.

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
**Decision:** Net-new service `integrations-api` (port 8007) for HubSpot/Salesforce/Pipedrive/Slack/Teams. Reasons mirroring AI Gateway split:

1. Blast radius: provider outages or rate-limits don't cascade into `client-api`.
2. Per-provider rate-limit characteristics warrant per-provider Celery worker queues.
3. Sync workers belong in their own deployment (decouples web-pod scaling from sync-worker scaling).
4. Three concurrent providers (HubSpot + Salesforce + Pipedrive) at launch tip the Rule of Three.

Slack/Teams (incoming-webhook templates) ships **before** CRM (bi-directional sync) — smaller scope, daily-felt value, larger addressable Pro tier vs. Pro+ tier.
**Rationale:** Decomposition stays at 6+1=7 services post-amendment — within the 5–10 sweet spot.
**Consequences:** New `integrations` schema; OAuth token vault stays in `client.crm_connections` for billing scope; `last-write-wins` conflict resolution with workspace-visible conflict log to `shared.audit_log`.

### ADR-010 — Boring infra for 99.9% SLA: Multi-AZ Postgres + Redis Sentinel, not Patroni

**Status:** Accepted (Epic 21, Plan 2026-04-25)
**Decision:** Migrate to managed Multi-AZ RDS (Aurora candidate; RDS Postgres baseline) and managed Redis with Sentinel. PDBs with `minAvailable: 1` and ≥2 replicas per service in production. SLO dashboards with error-budget burn-rate alerting. PagerDuty on-call rotation with runbook density. **Do not publish 99.9% SLA before infra uplift completes** (Phase A → B → C).
**Rationale:** Patroni-on-K8s is operational debt for a small team. Managed databases with documented Multi-AZ failover semantics are auditable and predictable.
**Consequences:** ~+€250–600/mo run rate. k6 baseline closure (PE.01) is the **first** Epic 21 story — has been deferred 6 epics; cannot publish any SLO without it. KraftData incidents excluded from SLA scope (isolated by AI Gateway circuit breaker).

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

---

## 8. Cross-Cutting Concerns

### 8.1 Multi-Tenancy

Tenant context propagates through every layer:

1. JWT carries `company_id`, `subscription_tier`, optionally `workspace_id` for workspace-scoped routes.
2. FastAPI middleware materializes `CurrentUser` from JWT.
3. `WorkspaceScope` `Depends()` factory injects `workspace_id` into the request scope; mismatch with route `{workspaceId}` → 404 (existence leakage protection — Epic 9, Epic 14).
4. SQLAlchemy queries always filter by `workspace_id` (and `company_id` where applicable) — bare-table queries without scope are a BLOCKING review finding.
5. Stripe metadata stores `company_id` and `workspace_id` so webhook handlers can scope updates correctly.

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
│   ├── ai-gateway/                    # :8004 — KraftData broker
│   ├── notification/                  # :8005 — email/Slack/Teams + materialized-view refresh
│   ├── enterprise-api/                # public REST proxy (gateway routes through client-api)
│   └── integrations-api/              # :8007 — CRM + Slack/Teams (Epic 16/17)
│
├── packages/                          # Shared Python packages
│   ├── eusolicit-common/              # BaseServiceSettings, structlog, EventBus, middleware
│   ├── eusolicit-models/              # Cross-service Pydantic DTOs, Redis Stream event schemas
│   ├── eusolicit-kraftdata/           # Typed KraftData client
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
│   ├── helm/                          # Per-service Helm charts
│   ├── terraform/                     # AWS provisioning (eu-central-1)
│   ├── postgres/                      # init scripts (schema + role provisioning)
│   ├── stripe-config.yaml             # Stripe Price IDs, products, tiers
│   └── trust/artefacts/               # Trust Center PDFs (legal-curated, Git-tracked)
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
- **Requirements coverage:** Every PRD FR maps to a service or ADR (FR-1..14 → `client-api` + Stripe; FR-15..20 → `data-pipeline` + `client-api`; FR-21..25 → `ai-gateway`; FR-26..33 → `client-api` + `ai-gateway`; FR-34..39 → `client-api` + `admin-api`; FR-40..44 → `notification` + `admin-api`). NFRs are addressed via specific ADRs (NFR-1/2 → ADR-005, ADR-014; NFR-7 → ADR-001/002/007; NFR-14 → ADR-010; NFR-15/17 → §6.5; NFR-21..23 → §6.3, §10).
- **Implementation readiness:** Mature project context, canonical fixtures, established CI gates. Ready.

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
| 1 | KraftData EU residency / GDPR sub-processor evidence | Confirm DPA before Epic 18 ships; add to sub-processor list |
| 2 | Vector store cost scaling under per-workspace partitioning | Confirm KraftData billing model; consider tenant-scoped vector stores with workspace-tagged content as fallback |
| 3 | External collaborator GDPR data flow (DPA chain-of-processing) | Update DPA template; flag in legal review |
| 4 | Stripe per-bid SKU + EU VAT MOSS reporting | Confirm Stripe Tax product configuration |
| 5 | BGN settlement preference for BG buyers | Confirm Stripe currency setup early in Epic 15 |
| 6 | ARR vs. one-time revenue tracking distinction | Add reporting category to internal billing dashboards |
| 7 | Workspace deletion vs. GDPR Right to Erasure | Document Art. 17.3.b legal-obligation exception; ensure audit entries reference IDs not free-form PII |
| 8 | Cross-workspace data leakage in shared content blocks | `tenant_scope=true` only on explicit user opt-in; PII detection at upload (Epic 19/20) |
| 9 | k6 baseline closure on critical path | First Epic 21 story (PE.01) |
| 10 | CRM provider rate-limit surprises at scale | Per-provider Celery queue with backoff; "rate-limit reached, paused" UX state |

---

## 12. Architecture Readiness Assessment

| Item | Status |
|---|---|
| **Overall Status** | **READY FOR IMPLEMENTATION** |
| **Confidence** | High — battle-tested across 15 shipped epics |
| **Strengths** | Proven patterns; coherent stack; living project-context; strict gate discipline (when followed) |
| **Watch-outs** | Carry-forward backlog; sprint-status integrity; TEA execution discipline |
| **Net-new for v2.0** | Workspaces (Epic 14), Per-bid SKU + Pro+ (Epic 15), `integrations-api` (Epics 16/17), Trust Center + ISO 27001 prep (Epic 18 + parallel programme), Outcome Telemetry (Epic 19), NPS (Epic 20), Platform Reliability (Epic 21) |

### Implementation Handoff — for AI Agents and Developers

**Primary directive:** Adhere strictly to the ADRs in this document AND the patterns in `project-context.md` files.
**Conflict resolution:** When this document and `project-context.md` disagree, **`project-context.md` takes precedence for implementation details** — it is generated from lived experience. Discrepancies must be raised in the next epic retrospective and resolved by amending one or both documents.
**Process:** Standard BMAD development workflow — Implementation Readiness check before each epic, Validate Story before each story, Story Review after each story in multi-story epics, Epic Review for complex/interdependent epics, Post-Review after code review for all epics.

---

**End of Architecture Document v2.0.**
