# EU Solicit — Solution Architecture v5 (Service-Oriented)

**Platform**: EU Solicit | **Domain**: EUSolicit.com | **Version**: 5.0 | **Date**: 2026-04-22 | **Status**: Draft
**Changelog**: v4.0 closes all gaps identified in the Architecture vs Requirements Gap Analysis (2026-04-05). See Sections 1.11–1.15 for new ADRs.

---

## 1. Architecture Decision Record

### 1.1 Backend: Python 3.12 + FastAPI

**Decision**: Python + FastAPI over Java/Spring Boot or PHP/Laravel.

**Rationale**: AI coding assistants have the deepest Python training data, accelerating AI-driven development. FastAPI's async runtime maps directly to the SSE streaming pattern required for KraftData agent integration. Pydantic provides strict validation across all service boundaries. OpenAPI auto-generation powers the Enterprise-tier public API.

**Trade-offs**: Python is slower than Java for raw compute, but EU Solicit is I/O-bound (DB queries, HTTP calls to KraftData, SSE streaming). CPU-intensive work is offloaded to KraftData agents.

### 1.2 Architecture Pattern: Hybrid SOA (5 Services)

**Decision**: Service-oriented architecture with 5 services from day one, not a monolith or full microservices.

**Rationale**: The domain reveals two fundamentally different user populations (admin vs client) with different scaling profiles, security postures, and release cadences. The data pipeline should fail independently without taking the client API down. KraftData integration warrants an isolated gateway to absorb third-party outages.

Splitting every domain module into its own service would create 10+ services for a 2-3 person team — too much coordination overhead. Five services hit the sweet spot: critical isolation benefits with manageable operational complexity.

### 1.3 Inter-Service Communication: REST + Async Events

**Decision**: Synchronous REST for queries, async Redis Streams for events.

**Rationale**: REST is simple and debuggable for synchronous operations (e.g., Client API calling AI Gateway). Redis Streams provides at-least-once event delivery for decoupled workflows (e.g., Data Pipeline notifying Notification Service of new opportunities). No need for Kafka or RabbitMQ at this scale.

### 1.4 Frontend: React 18 + Next.js 14

**Decision**: Next.js App Router with React Server Components. SSR for SEO on public tender listings, client-side interactivity for dashboards and proposal editors.

### 1.5 Database: PostgreSQL 16, Shared Instance with Schema Isolation

**Decision**: Single PostgreSQL instance, 6 schemas, per-service DB roles with enforced access boundaries.

**Rationale**: Avoids distributed database complexity while maintaining data ownership clarity. Each service has its own schema and DB role. Post-MVP, high-traffic schemas can be extracted to separate instances by updating connection strings only.

### 1.6 Deployment: Docker + Kubernetes

**Decision**: Containerized services on Kubernetes with Helm charts. Each service has its own Dockerfile, Helm chart, and CI pipeline.

### 1.7 KraftData Agent & Vector Store Management

**Decision**: All KraftData agents, teams, and workflows are created and configured manually via the KraftData admin interface. Vector stores (storage resources) are provisioned and managed through the KraftData API (`/client/api/v1/storage-resources/*`). EU Solicit does not maintain its own vector search infrastructure.

**Rationale**: KraftData's Agentic AI platform owns the full AI lifecycle — agent definitions, prompt tuning, vector indexing, and evaluation. EU Solicit consumes these capabilities through the AI Gateway. The platform's responsibility is orchestration and tier-gated access, not AI model management.

### 1.8 MVP Scope Boundaries

**In scope**: Opportunity intelligence (Bulgaria + EU), document analysis, AI-assisted proposals, EU grant specialization, compliance framework management, alert digests (email), calendar sync (Google + Outlook), analytics dashboards, Stripe billing, white-label (Enterprise).

**Out of scope for MVP**: Electronic bid submission, Microsoft OAuth2 (deferred post-MVP), e-procurement portal integration, ERP/CRM integration, Slack/Teams notifications, digital signatures (eIDAS), consultant marketplace.

### 1.9 Data Residency & GDPR Compliance

**Decision**: All tender data and client information stored within EU data centres. Infrastructure hosted in EU region (AWS eu-central-1 or equivalent).

**Rationale**: GDPR compliance is non-negotiable for a platform handling EU public procurement data and company profiles. This applies to PostgreSQL, Redis, S3/MinIO, and any managed service used by EU Solicit. KraftData's data residency must also be verified to be EU-hosted.

### 1.10 Document Retention Policy

**Decision**: Documents (uploaded tender files, generated proposals, exports) are retained for the lifetime of the opportunity + 2 years. Retention period is configurable per tenant at the Admin API level.

**Implementation**: A Celery Beat task in the Data Pipeline Service runs weekly to identify and soft-delete expired documents. Hard deletion (S3 purge) follows a 30-day grace period after soft-delete. Audit log entries are never deleted.

### 1.11 Data Pipeline Crawlers: Local Celery Tasks

**Decision**: All crawlers (AOP, TED, EU Grant portals) are implemented as local Celery tasks within the Data Pipeline service, not as KraftData agents.

**Rationale**: Crawlers are deterministic HTTP scrapers, not AI reasoning. Running them locally gives full control over error handling, rate limiting, and portal-specific quirks. KraftData agent cost model is per-call; paying per-crawl for 10K+ opportunities/day would be uneconomical. Portal authentication, cookies, and session state are best handled in trusted local code.

**Trade-offs**: Increases maintenance burden on the local data pipeline for parsing changes in target portals, but significantly reduces operational cost and dependency on KraftData for basic ingestion.

### 1.12 Task Orchestration & Approval Gates

**Decision**: The Client API owns a task orchestration engine and configurable approval workflow system.

**Rationale**: The platform manages multi-week bid preparation lifecycles involving multiple team members. Without task orchestration, users must track work externally (spreadsheets, project management tools), reducing platform stickiness. Approval gates are required by Enterprise clients who need governance sign-offs before submission.

**Implementation**: Tasks are modeled as a directed graph (task + dependencies). Approval workflows are company-configurable stage sequences attached to proposals. Both live in the `client` schema.

### 1.13 Proposal Collaboration & Entity-Level RBAC

**Decision**: Proposals support multi-user collaboration with per-entity role assignments. RBAC is enforced at two levels: company-wide role (sets the ceiling) and entity-level permission (narrows access per opportunity/proposal).

**Rationale**: Professional and Enterprise tiers serve teams of 5+ users working on the same proposals. A global "bid manager" role is insufficient — a user may be bid manager on one proposal and read-only on another. Pessimistic locking (one editor per section at a time) is chosen over real-time CRDT for MVP simplicity.

### 1.14 Billing: Trials, Add-Ons, EU VAT

**Decision**: The billing model supports three flows beyond basic subscriptions: (1) 14-day free trial of Professional tier without credit card, (2) per-bid add-on purchases as Stripe one-time charges, (3) EU VAT compliance via Stripe Tax.

**Rationale**: The trial drives conversion from Free to paid. Per-bid add-ons let users pay for premium AI features (full proposal generation, deep compliance audit) without upgrading their entire subscription. EU VAT handling is legally required for a SaaS platform selling to EU businesses.

### 1.15 Self-Service Bidding: Guided Submission & ESPD

**Decision**: The platform provides structured submission guidance and ESPD auto-fill as core features of the self-service bidding model.

**Rationale**: Since the MVP does not submit bids on behalf of clients, the platform's value depends on guiding users through external submission. Portal-specific step-by-step instructions and pre-filled ESPD forms bridge the gap between AI-generated proposal content and actual bid submission.

---

## 2. System Architecture Overview

```
┌───────────────────────────────────────────────────────────────────────────────────┐
│                                   INTERNET                                        │
└───────┬──────────────────┬──────────────────┬─────────────────────┬───────────────┘
        │                  │                  │                     │
 ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐      ┌──────▼──────┐
 │  CloudFlare │    │  Google     │    │  Microsoft  │      │  KraftData  │
 │  CDN + WAF  │    │  OAuth2 +   │    │  Graph API  │      │  Platform   │
 │             │    │  Calendar   │    │  (Outlook)  │      │  (Sirma AI) │
 └──────┬──────┘    └──────┬──────┘    └──────┬──────┘      └──────┬──────┘
        │                  │                  │                     │
 ┌──────▼──────┐           │                  │                     │
 │  NGINX      │           │                  │                     │
 │  INGRESS    │           │                  │                     │
 └──────┬──────┘           │                  │                     │
        │                  │                  │                     │
 ┌──────┴──────────────────┴──────────────────┴──── K8s Cluster ───┴──────────────┐
 │                                                                                │
 │   ┌────────────────────────────────────────────────────────────────────────┐   │
 │   │                         INGRESS ROUTING                                │   │
 │   │                                                                        │   │
 │   │  eusolicit.com          ──► Frontend (Next.js)                         │   │
 │   │  eusolicit.com/api/v1/* ──► SERVICE 1: Client API                      │   │
 │   │  api.eusolicit.com/*    ──► SERVICE 1: Client API (Enterprise API)     │   │
 │   │  admin.eusolicit.com/*  ──► SERVICE 2: Admin API   (VPN-restricted)    │   │
 │   │  *.eusolicit.com        ──► Frontend (white-label subdomains)          │   │
 │   └────────────────────────────────────────────────────────────────────────┘   │
 │                                                                                │
 │   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
 │   │ SVC 1:       │  │ SVC 2:       │  │ SVC 3:       │  │ SVC 4:       │     │
 │   │ CLIENT API   │  │ ADMIN API    │  │ DATA         │  │ AI           │     │
 │   │              │  │              │  │ PIPELINE     │  │ GATEWAY      │     │
 │   │ FastAPI      │  │ FastAPI      │  │              │  │              │     │
 │   │ 3-20 pods    │  │ 1-3 pods     │  │ Celery       │  │ FastAPI      │     │
 │   │              │  │ VPN only     │  │ 2-8 workers  │  │ 2-10 pods    │     │
 │   │ Opportunities│  │ Compliance   │  │              │  │              │     │
 │   │ Proposals    │  │ Crawl mgmt   │  │ Crawlers     │  │ KraftData    │     │
 │   │ Billing      │  │ Tenants      │  │ Normalize    │  │ client       │     │
 │   │ Alerts       │  │ Audit viewer │  │ Enrich       │  │ Agent exec   │     │
 │   │ Calendar     │  │ White-label  │  │ Upsert       │  │ SSE proxy    │     │
 │   │ Analytics    │  │ Platform     │  │              │  │ Webhooks     │     │
 │   │ Documents    │  │ settings     │  │              │  │ Circuit      │     │
 │   │ Auth (user)  │  │ Auth (admin) │  │              │  │ breaking     │     │
 │   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
 │          │                 │                  │                  │             │
 │          │          ┌──────┴──────────────────┴──────────────────┘             │
 │          │          │                                                         │
 │   ┌──────▼──────────▼─────┐                                                  │
 │   │ SVC 5: NOTIFICATION   │                                                  │
 │   │                       │                                                  │
 │   │ Celery 1-4 workers    │                                                  │
 │   │ Email delivery        │──► SendGrid                                      │
 │   │ Alert digest          │──► Google Calendar API                            │
 │   │ Calendar sync         │──► Microsoft Graph API                            │
 │   │ Usage sync to Stripe  │──► Stripe API                                     │
 │   └───────────────────────┘                                                  │
 │                                                                               │
 │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                         │
 │   │ PostgreSQL  │  │ Redis 7     │  │ S3 / MinIO  │                         │
 │   │ 16          │  │             │  │             │                         │
 │   │ 6 schemas   │  │ Event bus   │  │ Documents   │                         │
 │   │ Per-service │  │ Celery      │  │ Proposals   │                         │
 │   │ DB roles    │  │ Rate limits │  │ Exports     │                         │
 │   │             │  │ Usage ctrs  │  │             │                         │
 │   └─────────────┘  └─────────────┘  └─────────────┘                         │
 │                                                                              │
 │   ┌─────────────┐                                                           │
 │   │ ClamAV      │ (internal, client-api only)                               │
 │   └─────────────┘                                                           │
 └──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Service Definitions

### Service 1: Client API

**Purpose**: All client-facing (end-user) functionality. The primary revenue-generating surface.

| Aspect | Detail |
|---|---|
| **Framework** | FastAPI |
| **Scaling** | HPA 3-20 pods. Latency-sensitive. |
| **Exposure** | `eusolicit.com/api/v1/*`, `api.eusolicit.com/v1/*` (Enterprise) |
| **DB access** | `client` schema (read/write), `pipeline` schema (read-only: opportunities), `shared` schema (write: audit, usage) |

**Owns**: Opportunity search/listing/detail (tier-gated), proposal generation/versioning/export, proposal collaboration (per-entity roles, section locking, comments), task orchestration (assignments, dependencies, deadlines), approval workflows (configurable stage sequences, sign-off decisions), bid/no-bid workflow, bid outcome recording + preparation time/cost logging, alert preference management, calendar connection management + iCal feed, analytics dashboards (market, ROI, team performance, competitor, forecast, usage) + report export, document upload/download, company profiles, reusable content blocks library, ESPD auto-fill (structured form mapped to company profile), submission guides (portal-specific step-by-step instructions), entity-level RBAC (per-opportunity/proposal permission grants), user auth (email/password + Google OAuth2), subscription management + Stripe (trials, add-ons, VAT), tier gating + usage metering.

**Delegates to**: AI Gateway (all KraftData calls), Notification Service (email delivery, calendar sync execution).

### Service 2: Admin API

**Purpose**: Platform administration, compliance management, operational oversight. Admin-only access.

| Aspect | Detail |
|---|---|
| **Framework** | FastAPI |
| **Scaling** | HPA 1-3 pods. Security-critical. |
| **Exposure** | `admin.eusolicit.com/api/v1/*` (VPN/IP-restricted) |
| **DB access** | `admin` schema (read/write), `client` schema (read-only), `pipeline` schema (read-only), `shared` schema (read/write) |
| **Network** | Ingress restricted to VPN/office IPs only |

**Owns**: Compliance framework CRUD + assignment, framework auto-suggestion config, crawler management, tenant management, white-label config, audit log viewer, platform analytics, regulation tracker agent management.

### Service 3: Data Pipeline

**Purpose**: Continuous crawling, normalization, enrichment, and ingestion of opportunity data.

| Aspect | Detail |
|---|---|
| **Framework** | Celery workers + Beat scheduler |
| **Scaling** | HPA 2-8 workers. Burst during crawl windows. |
| **Exposure** | None (no ingress, outbound only) |
| **DB access** | `pipeline` schema (read/write), `shared` schema (write: audit) |

**Owns**: Celery Beat scheduler, crawl orchestration (triggers AOP/TED/EU Grant crawler agents via AI Gateway — crawling logic lives in KraftData, not locally), normalization (via AI Gateway), enrichment (via AI Gateway), opportunity upsert, submission guide ingestion (auto-generates portal-specific instructions from crawled metadata), expired opportunity cleanup. Publishes `opportunities.ingested` events to Redis Streams.

**Failure isolation**: If the pipeline goes down, Client API keeps serving existing data. Users notice stale results, not an outage.

### Service 4: AI Gateway

**Purpose**: Single integration point with KraftData. Insulates all services from KraftData API changes, outages, and rate limits.

| Aspect | Detail |
|---|---|
| **Framework** | FastAPI |
| **Scaling** | HPA 2-10 pods. Scale based on active SSE stream count. |
| **Exposure** | Internal ClusterIP only. Not internet-facing. |
| **DB access** | `gateway` schema (read/write: execution logs, latency metrics), `shared` schema (write: audit) |
| **Network** | Ingress from client-api, admin-api, data-pipeline only |

**Owns**: KraftData HTTP client, agent/team/workflow execution (sync + streaming), storage resource file upload (via KraftData API), KraftData webhook receiver, SSE stream proxying, agent ID registry (maps logical names to externally-managed KraftData agent IDs), retry logic + circuit breaking, eval-run execution.

**Why separate?** (1) Blast radius — KraftData outages are absorbed here. (2) Rate limit management — single connection pool. (3) Contract stability — KraftData API changes affect only this service. (4) SSE stream isolation — long-lived connections don't starve the REST request pool.

### Service 5: Notification Service

**Purpose**: All outbound communication and scheduled background tasks (excluding data ingestion).

| Aspect | Detail |
|---|---|
| **Framework** | Celery workers + Beat scheduler |
| **Scaling** | HPA 1-4 workers. Burst during digest windows. |
| **Exposure** | None (no ingress, event-driven) |
| **DB access** | `notification` schema (read/write), `client` schema (read-only: alerts, calendar, opportunities) |

**Owns**: Alert digest assembly + SendGrid delivery, calendar sync execution (Google Calendar API, Microsoft Graph API), OAuth token refresh, usage meter sync to Stripe, scheduled report generation.

---

## 4. Technology Stack

### 4.1 Backend (All Services)

| Component | Technology | Version | Purpose |
|---|---|---|---|
| Runtime | Python | 3.12 | Core language |
| Web Framework | FastAPI | 0.115+ | Client API, Admin API, AI Gateway |
| ASGI Server | Uvicorn | 0.30+ | Production server |
| Task Queue | Celery | 5.4+ | Data Pipeline, Notification Service |
| ORM | SQLAlchemy | 2.0+ (async) | All services |
| Migrations | Alembic | 1.13+ | Per-service migrations |
| Validation | Pydantic | 2.x | Request/response, inter-service DTOs |
| HTTP Client | httpx | 0.27+ | Inter-service REST, KraftData, Google, Microsoft |
| Auth | PyJWT + authlib | 2.8+ | JWT RS256 + Google OAuth2 |
| Billing | stripe SDK | latest | Subscription, metering, webhooks |
| Email | sendgrid SDK | latest | Transactional email |
| Calendar | google-api-python-client + msgraph-sdk | latest | Calendar sync |
| iCal | icalendar | latest | Feed generation |
| File scanning | pyclamd | latest | ClamAV integration |
| Doc Generation | python-docx + reportlab | latest | Proposal export |
| Logging | structlog | 24.x | Structured JSON |
| Testing | pytest + pytest-asyncio | 8.x | Async tests |
| Linting | ruff + mypy | latest | Lint + type check |

### 4.2 Frontend

| Component | Technology | Purpose |
|---|---|---|
| Next.js 14 | App Router, SSR/SSG | Two UIs: client (`eusolicit.com`) + admin (`admin.eusolicit.com`) |
| React 18 | Component architecture | — |
| Tailwind CSS + shadcn/ui | Styling + components | — |
| Zustand | Client state | — |
| TanStack Query | Server state, SSE subscriptions | — |
| React Hook Form + Zod | Form validation | — |
| Tiptap | Proposal rich text editor | — |
| Recharts | Analytics dashboards | — |
| next-intl | i18n (Bulgarian, English) | — |

### 4.3 Database & Storage

| Component | Technology | Purpose |
|---|---|---|
| PostgreSQL 16 | Shared instance, 6 schemas | Relational data, JSONB, FTS |
| Redis 7 | Event bus (Streams), Celery broker, rate limits, usage counters | — |
| S3 / MinIO | Tender documents, proposals, exports | — |
| Elasticsearch 8.x (post-MVP) | Advanced faceted search | — |

### 4.4 Infrastructure & DevOps

| Component | Technology | Purpose |
|---|---|---|
| Docker | Multi-stage builds | Per-service Dockerfiles |
| Kubernetes | Orchestration | Auto-scaling, network policies |
| Helm 3 | Package management | Reusable chart template with per-service values |
| GitHub Actions | CI/CD | Matrix build for all services in parallel |
| Terraform | IaC | Cloud resources |
| Cloudflare | CDN + WAF | SSL for EUSolicit.com |
| nginx-ingress | Ingress controller | Domain routing, rate limiting |
| External Secrets Operator | Secrets | Per-service secret sets from AWS Secrets Manager |
| Prometheus + Grafana | Monitoring | Metrics, dashboards, alerting |
| Loki | Logging | Centralized log aggregation |
| OpenTelemetry + Jaeger | APM | Distributed tracing across all 5 services |
| ClamAV | Virus scanning | File upload scanning (internal sidecar) |

---

## 5. Inter-Service Communication

### 5.1 Communication Matrix

| From → To | Method | Use Case |
|---|---|---|
| Client API → AI Gateway | REST (sync) + SSE proxy | Agent execution, proposal generation |
| Client API → Notification Svc | Redis Streams (async) | `alert_preference_updated`, `calendar_connected` |
| Admin API → AI Gateway | REST (sync) | Compliance checker, framework suggestion |
| Admin API → Data Pipeline | Redis Streams (async) | `crawler_schedule_updated`, `framework_assigned` |
| Data Pipeline → AI Gateway | REST (sync) | Normalization team, enrichment agents |
| Data Pipeline → Notification Svc | Redis Streams (async) | `opportunities_ingested` |
| Stripe → Client API | Webhook (HTTP) | Subscription lifecycle events |
| KraftData → AI Gateway | Webhook (HTTP) | Agent/workflow completion events |
| AI Gateway → Client API | Redis Streams (async) | `agent_completed`, `agent_failed` |

### 5.2 Event Catalog

| Event | Publisher | Subscribers | Payload |
|---|---|---|---|
| `opportunities.ingested` | Data Pipeline | Notification Svc, Client API (cache) | `{opportunity_ids, source}` |
| `opportunity.status_changed` | Data Pipeline | Notification Svc | `{opportunity_id, old_status, new_status}` |
| `agent.execution.completed` | AI Gateway | Client API | `{execution_id, agent_type, result_summary}` |
| `agent.execution.failed` | AI Gateway | Client API, Admin API | `{execution_id, agent_type, error}` |
| `alert_preference.updated` | Client API | Notification Svc | `{preference_id, user_id}` |
| `calendar.connected` | Client API | Notification Svc | `{connection_id, user_id, provider}` |
| `subscription.changed` | Client API | All services (tier cache) | `{company_id, old_tier, new_tier}` |
| `subscription.trial_expiring` | Client API | Notification Svc | `{company_id, user_id, trial_end}` |
| `task.assigned` | Client API | Notification Svc | `{task_id, assigned_to, opportunity_id}` |
| `task.overdue` | Client API | Notification Svc | `{task_id, assigned_to, due_date}` |
| `approval.requested` | Client API | Notification Svc | `{proposal_id, stage_id, required_role}` |
| `approval.decided` | Client API | Notification Svc | `{proposal_id, stage_id, decision, decided_by}` |
| `addon.purchased` | Client API | Client API (unlock) | `{company_id, add_on_type, opportunity_id}` |
| `compliance_framework.assigned` | Admin API | Client API, AI Gateway | `{opportunity_id, framework_id}` |
| `crawler.schedule_updated` | Admin API | Data Pipeline | `{crawler_type, new_schedule}` |

### 5.3 Redis Streams Configuration

```python
STREAMS = {
    "eu-solicit:opportunities": "Data pipeline events",
    "eu-solicit:agents": "AI Gateway execution events",
    "eu-solicit:subscriptions": "Subscription + trial lifecycle events",
    "eu-solicit:notifications": "Notification trigger events (alerts, calendar)",
    "eu-solicit:tasks": "Task assignment, overdue, and approval events",
    "eu-solicit:billing": "Add-on purchase events",
    "eu-solicit:admin": "Admin action events",
}

CONSUMER_GROUPS = {
    "client-api": ["eu-solicit:opportunities", "eu-solicit:agents", "eu-solicit:subscriptions", "eu-solicit:billing"],
    "notification-svc": ["eu-solicit:opportunities", "eu-solicit:notifications", "eu-solicit:tasks", "eu-solicit:subscriptions"],
    "data-pipeline": ["eu-solicit:admin"],
    "ai-gateway": ["eu-solicit:admin"],
}
```

---

## 6. Shared Libraries (Internal Python Packages)

```
packages/
├── eusolicit-common/           # Auth, middleware, config, logging, exceptions, health
├── eusolicit-kraftdata/        # KraftData client library (used by AI Gateway, importable for types)
└── eusolicit-models/           # Shared Pydantic DTOs for inter-service contracts + events
```

Distributed as private pip packages (Git URL or private PyPI). Each service's `pyproject.toml` declares its dependencies.

---

## 7. Per-Service Module Maps

### 7.1 Client API Service

```
services/client-api/
├── src/
│   ├── domain/
│   │   ├── opportunities/      # Search, detail, tier-gated access
│   │   ├── proposals/          # Generation, versioning, export
│   │   │   ├── collaboration/  # Per-proposal roles, section locking, comments
│   │   │   └── content_blocks/ # Reusable boilerplate library (CRUD, tagging, versioning)
│   │   ├── tasks/              # Task orchestration: assignments, dependencies, deadlines
│   │   ├── approvals/          # Configurable approval workflows + stage sign-off decisions
│   │   ├── subscriptions/      # Tier gating, usage metering, Stripe (trials, add-ons, VAT)
│   │   ├── companies/          # Profiles, team members, credentials, ESPD profiles
│   │   ├── alerts/             # Preference management (delivery delegated)
│   │   ├── calendar/           # Connection management, iCal feed
│   │   ├── analytics/          # Dashboards, competitor, forecast, usage, ROI, team perf
│   │   │   └── reporting/      # Scheduled report generation + PDF/DOCX export
│   │   ├── espd/               # ESPD auto-fill (structured form → company profile mapping)
│   │   ├── submission_guides/  # Portal-specific step-by-step submission instructions
│   │   └── audit/              # Write-only audit logging
│   ├── application/
│   │   ├── commands/           # register_company, create_proposal, bid_decision,
│   │   │                      # assign_task, approve_stage, purchase_addon, etc.
│   │   ├── queries/            # search_opportunities, get_analytics,
│   │   │                      # get_task_board, get_approval_pipeline, etc.
│   │   └── dtos/
│   ├── infrastructure/
│   │   ├── api/
│   │   │   ├── app.py
│   │   │   ├── middleware/     # auth, tier_gate, usage_gate, rate_limit, audit,
│   │   │   │                  # entity_permission (per-opportunity/proposal RBAC)
│   │   │   └── routers/       # opportunities, proposals, companies, subscriptions,
│   │   │                      # alerts, calendar, analytics, bid_decisions, documents,
│   │   │                      # tasks, approvals, content_blocks, espd, submission_guides
│   │   ├── persistence/       # SQLAlchemy models for client + shared schemas
│   │   ├── ai_gateway_client/ # HTTP client to call AI Gateway service
│   │   ├── billing/           # Stripe client + webhook handler (subscriptions, trials,
│   │   │                      # add-on purchases, VAT via Stripe Tax)
│   │   ├── storage/           # S3 client + ClamAV
│   │   ├── event_bus/         # Redis Streams publisher + consumer
│   │   └── auth/              # JWT + Google OAuth2 + entity-level permission checks
│   └── config/
├── tests/
├── Dockerfile
├── helm/
└── pyproject.toml
```

### 7.2 Admin API Service

```
services/admin-api/
├── src/
│   ├── domain/
│   │   ├── compliance/         # Framework CRUD, assignment, suggestion
│   │   ├── tenants/            # Company/subscription oversight
│   │   ├── crawlers/           # Crawler scheduling config
│   │   ├── whitelabel/         # White-label configuration
│   │   └── audit/              # Audit log reading + writing
│   ├── application/
│   │   ├── commands/           # assign_framework, update_crawler, configure_whitelabel
│   │   ├── queries/            # get_audit_log, get_tenant_list, get_platform_analytics
│   │   └── dtos/
│   ├── infrastructure/
│   │   ├── api/
│   │   │   ├── app.py
│   │   │   ├── middleware/     # admin auth, audit
│   │   │   └── routers/       # compliance, crawlers, tenants, audit, whitelabel, health
│   │   ├── persistence/       # SQLAlchemy models for admin + shared schemas
│   │   ├── ai_gateway_client/ # HTTP client to AI Gateway
│   │   └── event_bus/         # Redis Streams publisher
│   └── config/
├── tests/
├── Dockerfile
├── helm/
└── pyproject.toml
```

### 7.3 Data Pipeline Service

```
services/data-pipeline/
├── src/
│   ├── domain/
│   │   ├── crawling/           # Crawl orchestration: scheduling, result ingestion
│   │   │                      # (crawling logic lives in KraftData agents, not here)
│   │   ├── normalization/      # Data standardization rules
│   │   ├── enrichment/         # AI enrichment orchestration
│   │   └── submission_guides/  # Auto-generate portal-specific instructions from crawl metadata
│   ├── infrastructure/
│   │   ├── persistence/       # SQLAlchemy models for pipeline schema
│   │   ├── ai_gateway_client/ # HTTP client to AI Gateway (triggers crawler agents,
│   │   │                      # normalization team, enrichment agents)
│   │   └── event_bus/         # Redis Streams publisher
│   ├── workers/
│   │   ├── celery_app.py
│   │   ├── beat_schedule.py
│   │   └── tasks/
│   │       ├── trigger_crawl.py       # Triggers KraftData crawler agents via AI Gateway
│   │       ├── ingest_crawl_results.py # Processes crawler agent output → opportunity upsert
│   │       ├── generate_submission_guides.py  # Builds portal-specific instructions
│   │       └── cleanup_expired.py
│   └── config/
├── tests/
├── Dockerfile
├── helm/
└── pyproject.toml
```

### 7.4 AI Gateway Service

```
services/ai-gateway/
├── src/
│   ├── domain/
│   │   └── execution/          # Agent execution tracking, circuit breaking
│   ├── infrastructure/
│   │   ├── api/
│   │   │   ├── app.py
│   │   │   └── routers/
│   │   │       ├── agents.py       # POST /agents/{id}/run, /run-stream
│   │   │       ├── teams.py        # POST /teams/{id}/run
│   │   │       ├── workflows.py    # POST /workflows/{id}/run, /run-stream
│   │   │       ├── storage.py      # Storage resource management
│   │   │       ├── webhooks.py     # KraftData inbound webhooks
│   │   │       └── health.py
│   │   ├── kraftdata/
│   │   │   ├── client.py          # httpx async client to KraftData API
│   │   │   ├── circuit_breaker.py # Circuit breaking + retry logic
│   │   │   └── agent_registry.py  # Logical name → KraftData agent ID mapping
│   │   ├── persistence/          # Execution log, latency metrics
│   │   └── event_bus/            # Redis Streams publisher
│   └── config/
├── tests/
├── Dockerfile
├── helm/
└── pyproject.toml
```

### 7.5 Notification Service

```
services/notification/
├── src/
│   ├── domain/
│   │   ├── alerts/             # Alert matching logic, digest assembly
│   │   └── calendar_sync/      # Sync orchestration
│   ├── infrastructure/
│   │   ├── email/
│   │   │   ├── sendgrid_client.py
│   │   │   └── templates/
│   │   ├── calendar_providers/
│   │   │   ├── google_calendar.py
│   │   │   └── microsoft_graph.py
│   │   ├── persistence/       # SQLAlchemy models for notification schema
│   │   └── event_bus/         # Redis Streams consumer
│   ├── workers/
│   │   ├── celery_app.py
│   │   ├── beat_schedule.py
│   │   └── tasks/
│   │       ├── generate_alerts.py
│   │       ├── sync_calendars.py
│   │       ├── report_usage.py
│   │       └── generate_reports.py
│   └── config/
├── tests/
├── Dockerfile
├── helm/
└── pyproject.toml
```

---

## 8. Database Strategy — Schema Isolation

```
PostgreSQL 16 Instance
│
├── Schema: client              ← Client API Service (read/write)
│   ├── users
│   ├── companies
│   ├── subscriptions           (+ trial_start, trial_end, trial_converted_at)
│   ├── tier_access_policies
│   ├── add_on_purchases        ★ NEW — per-bid add-on one-time charges
│   ├── proposals / proposal_versions
│   ├── proposal_collaborators  ★ NEW — per-proposal role assignments (user, proposal, role)
│   ├── proposal_comments       ★ NEW — section-level comments/annotations
│   ├── proposal_section_locks  ★ NEW — pessimistic locking for concurrent editing
│   ├── content_blocks          ★ NEW — reusable boilerplate sections (tagged, versioned)
│   ├── tasks                   ★ NEW — task assignments with deadlines
│   ├── task_dependencies       ★ NEW — directed dependency graph between tasks
│   ├── task_templates          ★ NEW — per-opportunity-type task templates
│   ├── approval_workflows      ★ NEW — company-configurable stage definitions
│   ├── approval_stages         ★ NEW — ordered stages within a workflow
│   ├── approval_decisions      ★ NEW — per-stage sign-off records (user, decision, timestamp)
│   ├── entity_permissions      ★ NEW — per-opportunity/proposal RBAC grants
│   ├── espd_profiles           ★ NEW — pre-mapped ESPD field values per company
│   ├── bid_decisions
│   ├── bid_outcomes
│   ├── bid_preparation_logs    ★ NEW — time/cost tracking per user per proposal (for ROI)
│   ├── alert_preferences
│   ├── calendar_connections
│   ├── calendar_events
│   ├── documents
│   ├── competitor_records
│   └── whitelabel_configs
│
├── Schema: pipeline            ← Data Pipeline Service (read/write)
│   ├── opportunities           ← Central data store
│   ├── submission_guides       ★ NEW — portal-specific step-by-step instructions per opportunity
│   ├── crawler_runs
│   └── enrichment_queue
│
├── Schema: admin               ← Admin API Service (read/write)
│   ├── compliance_frameworks
│   └── platform_settings
│
├── Schema: notification        ← Notification Service (read/write)
│   ├── alert_log
│   ├── email_log
│   └── sync_log
│
├── Schema: gateway             ← AI Gateway Service (read/write)
│   ├── agent_executions
│   └── webhook_log
│
└── Schema: shared              ← All services (write)
    ├── audit_log               (append-only, immutable)
    └── usage_meters            (Redis-backed, persisted daily)
```

### DB Role Access Matrix

| Service | DB Role | Own Schema | Read-Only Access | Shared Write |
|---|---|---|---|---|
| Client API | `client_api_role` | `client` | `pipeline` (opportunities) | `shared` |
| Admin API | `admin_api_role` | `admin` | `client`, `pipeline` | `shared` |
| Data Pipeline | `pipeline_role` | `pipeline` | — | `shared` |
| AI Gateway | `gateway_role` | `gateway` | — | `shared` |
| Notification Svc | `notification_role` | `notification` | `client` (alerts, calendar), `pipeline` (opportunities) | `shared` |

---

## 9. Data Models

All entity definitions from v2 remain unchanged except for new entities added in v4 (marked ★). For reference, the core entities and their schema assignments:

| Entity | Schema | Owner Service |
|---|---|---|
| `opportunities` | `pipeline` | Data Pipeline |
| `submission_guides` ★ | `pipeline` | Data Pipeline |
| `compliance_frameworks` | `admin` | Admin API |
| `users`, `companies` | `client` | Client API |
| `subscriptions`, `tier_access_policies` | `client` | Client API |
| `add_on_purchases` ★ | `client` | Client API |
| `proposals`, `proposal_versions` | `client` | Client API |
| `proposal_collaborators` ★, `proposal_comments` ★, `proposal_section_locks` ★ | `client` | Client API |
| `content_blocks` ★ | `client` | Client API |
| `tasks` ★, `task_dependencies` ★, `task_templates` ★ | `client` | Client API |
| `approval_workflows` ★, `approval_stages` ★, `approval_decisions` ★ | `client` | Client API |
| `entity_permissions` ★ | `client` | Client API |
| `espd_profiles` ★ | `client` | Client API |
| `bid_decisions`, `bid_outcomes` | `client` | Client API |
| `bid_preparation_logs` ★ | `client` | Client API |
| `alert_preferences` | `client` | Client API |
| `calendar_connections`, `calendar_events` | `client` | Client API |
| `documents` | `client` | Client API |
| `competitor_records` | `client` | Client API |
| `whitelabel_configs` | `client` | Client API |
| `alert_log`, `email_log`, `sync_log` | `notification` | Notification Svc |
| `agent_executions`, `webhook_log` | `gateway` | AI Gateway |
| `audit_log` | `shared` | All services (write) |
| `usage_meters` | `shared` | All services (write) |

Full field-level entity definitions for v2 entities: see [Solution Architecture v2, Sections 5.1–5.9](./EU_Solicit_Solution_Architecture_v2.md#5-data-model--core-entities).

### 9.1 New Entity Definitions (v4)

**`tasks`**: `id`, `company_id`, `opportunity_id` (nullable), `proposal_id` (nullable), `title`, `description`, `assigned_to` (user_id), `created_by` (user_id), `status` (pending/in_progress/completed/blocked), `priority` (p1–p4), `due_date`, `completed_at`, `template_id` (nullable, FK to task_templates), `created_at`, `updated_at`.

**`task_dependencies`**: `id`, `task_id` (FK), `depends_on_task_id` (FK), `dependency_type` (finish_to_start/finish_to_finish). Unique constraint on (task_id, depends_on_task_id).

**`task_templates`**: `id`, `company_id`, `name`, `opportunity_type` (tender/grant), `stages` (JSONB — ordered list of template task definitions with relative deadlines). Allows companies to define repeatable task sequences per opportunity type.

**`approval_workflows`**: `id`, `company_id`, `name`, `is_default` (boolean), `created_at`. One company may have multiple workflows (e.g., different governance for tenders vs. grants).

**`approval_stages`**: `id`, `workflow_id` (FK), `name` (e.g., "Technical Review", "Legal Review", "Management Sign-off"), `order` (integer), `required_role` (the entity-level role that can approve this stage), `auto_advance` (boolean — advance to next stage on approval, or wait for manual trigger).

**`approval_decisions`**: `id`, `proposal_id` (FK), `stage_id` (FK), `decided_by` (user_id), `decision` (approved/rejected/returned_for_revision), `comment`, `decided_at`.

**`proposal_collaborators`**: `id`, `proposal_id` (FK), `user_id` (FK), `role` (bid_manager/technical_writer/financial_analyst/legal_reviewer/read_only), `granted_by` (user_id), `granted_at`. Unique constraint on (proposal_id, user_id).

**`proposal_comments`**: `id`, `proposal_id` (FK), `version_id` (FK to proposal_versions), `section_key` (string — identifies the proposal section), `user_id`, `body`, `resolved` (boolean), `resolved_by` (user_id), `created_at`.

**`proposal_section_locks`**: `id`, `proposal_id` (FK), `section_key`, `locked_by` (user_id), `locked_at`, `expires_at` (auto-release after 15 min inactivity). Unique constraint on (proposal_id, section_key).

**`content_blocks`**: `id`, `company_id`, `title`, `category` (company_overview/quality_management/sustainability/methodology/team/custom), `body` (rich text), `tags` (text[]), `version` (integer), `is_approved` (boolean), `approved_by` (user_id), `created_by` (user_id), `created_at`, `updated_at`.

**`entity_permissions`**: `id`, `user_id` (FK), `entity_type` (opportunity/proposal), `entity_id` (UUID), `permission` (view/comment/edit/manage), `granted_by` (user_id), `granted_at`. Unique constraint on (user_id, entity_type, entity_id). Company-wide role sets the ceiling; entity permission narrows or overrides within that ceiling.

**`espd_profiles`**: `id`, `company_id` (FK), `profile_name` (e.g., "Default", "Joint Venture with X"), `espd_data` (JSONB — structured mapping of company fields to ESPD schema sections: exclusion grounds, selection criteria, economic capacity, technical capacity), `created_at`, `updated_at`. Companies may maintain multiple ESPD profiles for different legal entities or consortium arrangements.

**`submission_guides`**: `id`, `opportunity_id` (FK), `source_portal` (aop/ted/eu_grants), `steps` (JSONB — ordered list of {step_number, title, instruction, portal_url, form_reference, tips}), `generated_by` (agent_execution_id), `reviewed` (boolean — admin-reviewed flag), `created_at`, `updated_at`. Auto-generated during crawl ingestion; Client API serves read-only to users.

**`add_on_purchases`**: `id`, `company_id` (FK), `purchased_by` (user_id), `add_on_type` (proposal_generation/deep_compliance_audit/pricing_analysis), `opportunity_id` (nullable — links to specific opportunity if applicable), `stripe_payment_intent_id`, `amount_eur` (integer, cents), `status` (pending/succeeded/failed/refunded), `created_at`.

**`bid_preparation_logs`**: `id`, `proposal_id` (FK), `user_id` (FK), `activity_type` (drafting/review/research/meeting/other), `hours` (decimal), `cost_eur` (decimal, optional — for external costs like consultant fees), `logged_at`, `notes`. Aggregated for ROI tracker and team performance analytics.

---

## 10. Tier Gating & Usage Enforcement

The `TierGate`, `UsageGate`, and `EntityPermission` middleware live in the Client API service. The `eusolicit-common` shared library provides the base middleware classes so Admin API can reuse the audit middleware.

Key patterns: free-tier returns `OpportunityFreeResponse` (limited metadata), paid tiers return `OpportunityFullResponse` gated by region/CPV/budget. Usage limits enforced via atomic Redis INCR with rollback on limit exceeded (429 response with upgrade prompt).

### 10.1 Trial Management

New registrations automatically start a 14-day free trial of the Professional tier (no credit card required). Trial state is managed in the `subscriptions` table:

- `trial_start` (timestamp) — set at registration.
- `trial_end` (timestamp) — `trial_start + 14 days`.
- `trial_converted_at` (timestamp, nullable) — set when user upgrades to a paid tier during or after trial.

**Trial lifecycle**: Registration → trial active (Professional features unlocked) → trial expiration → graceful downgrade to Free tier (data preserved, features gated) → upgrade prompt. If the user upgrades during trial, the remaining trial days are not credited — billing starts immediately at the chosen tier.

**Stripe integration**: Trial subscriptions are created with `trial_end` set on the Stripe subscription object. No `payment_method` is required during trial creation. Stripe sends `customer.subscription.trial_will_end` webhook 3 days before expiry; the Notification Service sends a reminder email. On trial expiry, Stripe transitions the subscription to `past_due` (no payment method) — the Client API webhook handler downgrades the user to Free tier.

**Eligibility**: One trial per company (not per user). Checked against `subscriptions.trial_start IS NOT NULL` for the company.

### 10.2 Per-Bid Add-On Purchases

Paid-tier users can purchase per-bid add-ons for premium AI features without upgrading their subscription. Add-on types:

| Add-On | Description | Availability |
|--------|-------------|--------------|
| Full Proposal Generation | Complete AI-drafted proposal for a specific opportunity | Professional, Enterprise |
| Deep Compliance Audit | Comprehensive compliance check against assigned framework | Starter, Professional, Enterprise |
| Pricing Analysis | AI-driven competitive pricing recommendation | Professional, Enterprise |

**Flow**: User selects add-on from opportunity detail → Stripe Checkout Session (one-time payment, `mode: payment`) → on `checkout.session.completed` webhook, create `add_on_purchases` record with `status: succeeded` → unlock the feature for that specific opportunity.

**Tier interaction**: Add-on usage does not count against monthly subscription limits. Add-ons are opportunity-scoped — purchasing a proposal generation add-on for Opportunity A does not unlock it for Opportunity B.

### 10.3 EU VAT Handling

**Stripe Tax**: Enabled on all subscription and add-on charges. Stripe Tax automatically calculates the correct VAT rate based on the customer's country, business status (B2B vs. B2C), and the EU reverse charge mechanism.

**Tax ID collection**: At registration or first upgrade, companies provide their EU VAT number (optional). Stored in `companies.tax_id` and synced to the Stripe Customer object via `customer.tax_ids`. Stripe validates VAT numbers via VIES and applies reverse charge where applicable.

**Invoice requirements**: All Stripe-generated invoices include: seller VAT ID, buyer VAT ID (if provided), applicable VAT rate, net/gross amounts, and the legal basis for any exemption (e.g., "Reverse charge — Article 196 Council Directive 2006/112/EC").

**Enterprise custom invoicing**: Enterprise tier clients can opt for manual invoicing with custom payment terms (NET 30/60). The Admin API generates invoice records; payment is tracked manually. Stripe Invoicing API used for generation; manual payment reconciliation in admin panel.

---

## 11. KraftData Integration (AI Gateway)

The AI Gateway exposes an internal REST API that mirrors KraftData's capabilities:

| Internal Endpoint | Upstream KraftData Endpoint | Callers |
|---|---|---|
| `POST /agents/{id}/run` | `POST /client/api/v1/agents/{agentId}/run` | Client API, Admin API, Data Pipeline |
| `POST /agents/{id}/run-stream` | `POST /client/api/v1/agents/{agentId}/run-stream` | Client API (SSE proxy) |
| `POST /workflows/{id}/run` | `POST /client/api/v1/workflows/{workflowId}/run` | Data Pipeline |
| `POST /workflows/{id}/run-stream` | `POST /client/api/v1/workflows/{workflowId}/run-stream` | Client API (SSE proxy) |
| `POST /teams/{id}/run` | `POST /client/api/v1/teams/{teamId}/run` | Data Pipeline |
| `POST /storage/{id}/files` | `POST /client/api/v1/storage-resources/{id}/files` | Client API |
| `POST /webhooks/kraftdata` | — (inbound) | KraftData platform |

The gateway adds: circuit breaking (fail-open after 5 consecutive failures, retry after 30s), request queuing (respect KraftData rate limits), execution logging (latency, success/failure, agent type), and SSE connection tracking (for HPA scaling metric).

### 11.2 KraftData Agent Inventory

All agents, teams, workflows, and vector stores listed below are created and maintained by KraftData via their admin interface. EU Solicit's AI Gateway references them by logical name mapped to KraftData entity IDs in the agent registry.

| # | Name | Type | Ownership | Consumer Service |
|---|---|---|---|---|
| 1 | Document Parser Agent | Agent | KraftData | Data Pipeline |
| 2 | Executive Summary Agent | Agent | KraftData | Client API |
| 3 | Compliance Checker Agent | Agent | KraftData | Client API, Admin API |
| 4 | Clause Risk Analyzer Agent | Agent | KraftData | Client API |
| 5 | Relevance Scoring Agent | Agent | KraftData | Data Pipeline |
| 6 | Pricing Assistant Agent | Agent | KraftData | Client API |
| 7 | Scoring Simulator Agent | Agent | KraftData | Client API |
| 8 | Grant Eligibility Agent | Agent | KraftData | Client API |
| 9 | Budget Builder Agent | Agent | KraftData | Client API |
| 10 | Logframe Generator Agent | Agent | KraftData | Client API |
| 11 | Consortium Finder Agent | Agent | KraftData | Client API |
| 12 | Reporting Template Generator Agent | Agent | KraftData | Client API |
| 13 | Bid/No-Bid Decision Agent | Agent | KraftData | Client API |
| 14 | Market Intelligence Agent | Agent | KraftData | Client API |
| 15 | Regulation Tracker Agent | Agent | KraftData | Admin API (scheduled) |
| 16 | Lessons Learned Agent | Agent | KraftData | Client API |
| 17 | Bid History Analytics Agent | Agent | KraftData | Client API (scheduled) |
| 18 | Competitor Analysis Agent | Agent | KraftData | Data Pipeline, Client API |
| 19 | Pipeline Forecasting Agent | Agent | KraftData | Client API |
| 20 | Framework Suggestion Agent | Agent | KraftData | Admin API |
| 21 | Requirement Checklist Agent | Agent | KraftData | Client API |
| 22 | Win Theme Extractor Agent | Agent | KraftData | Client API |
| 23 | ESPD Auto-Fill Agent | Agent | KraftData | Client API |
| 24 | Submission Guide Agent | Agent | KraftData | Data Pipeline |
| 25 | Data Normalization Team | Team | KraftData | Data Pipeline |
| 26 | Proposal Generator Workflow | Workflow | KraftData | Client API (SSE) |

### 11.3 KraftData Storage Resources (Vector Stores)

Vector stores are provisioned and managed through the KraftData API (`/client/api/v1/storage-resources/*`). EU Solicit does not maintain its own vector search infrastructure — all semantic search and retrieval-augmented generation is handled by KraftData agents referencing these stores.

| # | Storage Resource | Purpose | Populated By |
|---|---|---|---|
| 1 | RFP Templates Store | Reference RFP structures and past winning proposals | Admin upload via AI Gateway |
| 2 | Regulatory Knowledge Store | ZOP, EU directives, procurement regulations, ESPD schemas | Regulation Tracker Agent |
| 3 | Company Profiles Store | Client company capabilities, past performance, reusable content blocks | Client API upload via AI Gateway |
| 4 | Historical Bids Store | Past bid outcomes, scoring data, lessons learned | Lessons Learned Agent |
| 5 | Market Data Store | Market intelligence, competitor data, pricing benchmarks | Market Intelligence Agent |
| 6 | EU Programmes Store | EU funding programme details, eligibility criteria | Data Pipeline (EU Grant crawler agent) |
| 7 | Consortium Partners Store | Organizations that have participated in EU grants, their domains, past roles, countries | Data Pipeline (EU Grant crawler agent), Admin upload |

---

## 12. Authentication & Authorization

Unchanged from v2. JWT RS256 tokens issued by Client API's auth module. Admin API verifies the same JWT but requires `platform_admin` role. The `eusolicit-common` library provides the JWT verification dependency so all FastAPI services can validate tokens consistently.

| Auth Flow | Service |
|---|---|
| Email/password registration + login | Client API |
| Google OAuth2 social login | Client API |
| JWT token issuance + refresh | Client API |
| JWT verification | All services (via shared library) |
| Admin role enforcement | Admin API |

---

## 13. Billing, Alerts, Calendar, Documents

Functional architectures from v2 remain, with v4 additions for billing (trials, add-ons, VAT), proposal collaboration, analytics computation, and submission guidance. The SOA split changes WHERE each piece runs, not HOW it works:

| Feature | Configuration/UI | Execution |
|---|---|---|
| Stripe billing (subscriptions) | Client API (checkout, webhooks) | Notification Svc (usage sync to Stripe) |
| Stripe billing (trials) | Client API (trial creation at registration, downgrade on expiry webhook) | Notification Svc (trial expiry reminder email) |
| Stripe billing (add-ons) | Client API (Checkout Session, payment webhook) | Client API (unlock feature on payment success) |
| Stripe billing (VAT) | Client API (tax ID collection, Stripe Tax config) | Stripe Tax (automatic calculation) |
| Email alerts | Client API (preference CRUD) | Notification Svc (matching, digest, SendGrid) |
| Calendar sync | Client API (OAuth flow, connection CRUD, iCal feed) | Notification Svc (periodic sync via Celery) |
| Documents | Client API (upload, download, metadata) | Client API (ClamAV scan, S3 storage) |
| Proposal collaboration | Client API (collaborator assignment, section lock/unlock, comments) | Client API (pessimistic lock enforcement) |
| Task orchestration | Client API (task CRUD, dependency graph, templates) | Client API (deadline notifications delegated to Notification Svc) |
| Approval workflows | Client API (workflow config, stage sign-off) | Notification Svc (approval request/decision emails) |
| Submission guides | Data Pipeline (auto-generation from crawl metadata) | Client API (read-only serving to users) |
| ESPD auto-fill | Client API (profile management, form generation) | AI Gateway (ESPD Agent for AI-assisted field mapping) |
| Content blocks | Client API (CRUD, tagging, approval workflow) | AI Gateway (blocks fed to Proposal Generator via RAG) |
| Analytics & reporting | Client API (dashboard queries, materialized views) | Notification Svc (scheduled report generation + export) |

### 13.1 Proposal Collaboration Architecture

**Editing model**: Pessimistic section-level locking. When a user begins editing a proposal section, the Client API acquires a lock in `proposal_section_locks`. The lock has a 15-minute TTL (auto-released on inactivity). Other users see the section as locked with the editor's name displayed. Attempting to edit a locked section returns a `423 Locked` response with the lock holder's identity.

**Comments**: Threaded comments are anchored to a specific proposal version and section. When a new version is created, existing unresolved comments carry forward. Resolved comments are archived.

**Permission enforcement**: The `entity_permission` middleware checks `proposal_collaborators` for the requesting user's role on the target proposal. Role hierarchy: `bid_manager` > `technical_writer` = `financial_analyst` = `legal_reviewer` > `read_only`. Bid managers can edit all sections, assign collaborators, and approve stages. Other roles can edit only their designated sections.

### 13.2 Task Orchestration Architecture

**Task lifecycle**: pending → in_progress → completed (or blocked). Tasks are linked to opportunities and/or proposals. Dependencies are modeled as a DAG in `task_dependencies` — a task cannot transition to `in_progress` if any of its `finish_to_start` dependencies are incomplete.

**Templates**: Companies can define task templates per opportunity type (tender vs. grant). When a user starts working on a new opportunity, they can apply a template, which creates a full set of tasks with relative deadlines computed from the opportunity's submission deadline (e.g., "Technical review due 7 days before deadline").

**Notifications**: Task assignments and deadline reminders are published as events to Redis Streams and consumed by the Notification Service for email delivery. Overdue tasks generate daily digest alerts.

### 13.3 Analytics Computation Model

**Dashboard data**: Analytics dashboards (market intelligence, ROI, team performance, competitor, forecast, usage) are powered by PostgreSQL materialized views refreshed on a schedule by a Celery Beat task in the Notification Service.

| Dashboard | Data Source | Refresh Frequency |
|-----------|-------------|-------------------|
| Market Intelligence | `opportunities` + `bid_outcomes` + Market Intelligence Agent | Daily |
| ROI Tracker | `bid_outcomes` + `bid_preparation_logs` + `subscriptions` | Daily |
| Team Performance | `bid_preparation_logs` + `proposals` + `bid_outcomes` per user | Daily |
| Competitor Intelligence | `competitor_records` + Competitor Analysis Agent | Daily |
| Pipeline Forecast | `opportunities` + Pipeline Forecasting Agent | Daily |
| Usage | `usage_meters` (Redis → materialized view) | Hourly |

**Report export**: The Notification Service's `generate_reports` Celery task produces PDF/DOCX reports from materialized view data using the same `python-docx` + `reportlab` stack used for proposal export. Reports can be scheduled (weekly/monthly) or generated on demand via the Client API.

---

## 14. Deployment Architecture

### 14.1 Kubernetes Topology

```
Namespace: eu-solicit
│
├── Deployments
│   ├── frontend                (Next.js)           — 2 pods, HPA 2-10
│   ├── client-api              (FastAPI)           — 3 pods, HPA 3-20
│   ├── admin-api               (FastAPI)           — 1 pod,  HPA 1-3
│   ├── ai-gateway              (FastAPI)           — 2 pods, HPA 2-10
│   ├── data-pipeline-worker    (Celery Worker)     — 2 pods, HPA 2-8
│   ├── data-pipeline-beat      (Celery Beat)       — 1 pod   (singleton)
│   ├── notification-worker     (Celery Worker)     — 1 pod,  HPA 1-4
│   ├── notification-beat       (Celery Beat)       — 1 pod   (singleton)
│   └── clamav                  (Virus scanner)     — 1 pod
│
├── StatefulSets
│   ├── postgresql              — 1 primary + 1 replica (or managed RDS)
│   └── redis                   — 1 pod (or managed ElastiCache)
│
├── Services (ClusterIP)
│   ├── frontend-svc
│   ├── client-api-svc
│   ├── admin-api-svc
│   ├── ai-gateway-svc          ← Internal only
│   ├── clamav-svc              ← Internal only
│   └── redis-svc
│
├── Ingress (nginx)
│   ├── eusolicit.com                → frontend-svc
│   ├── eusolicit.com/api/v1/*       → client-api-svc
│   ├── api.eusolicit.com/v1/*       → client-api-svc (Enterprise API)
│   ├── admin.eusolicit.com/*        → admin-api-svc  (IP-restricted)
│   └── *.eusolicit.com              → frontend-svc   (white-label)
│
├── NetworkPolicies
│   ├── admin-api:    ingress from VPN/office IPs only
│   ├── ai-gateway:   ingress from client-api, admin-api, data-pipeline only
│   ├── clamav:       ingress from client-api only
│   └── data-pipeline: no ingress (outbound only)
│
├── CronJobs
│   └── db-backup — daily pg_dump to S3
│
└── Secrets (External Secrets Operator → AWS Secrets Manager)
    ├── client-api-secrets
    ├── admin-api-secrets
    ├── ai-gateway-secrets      (KraftData API key)
    ├── data-pipeline-secrets
    └── notification-secrets    (SendGrid, Google OAuth, Microsoft OAuth)
```

### 14.2 Domain Configuration

| Domain | Purpose | Service |
|---|---|---|
| `eusolicit.com` | Main platform | Frontend |
| `eusolicit.com/api/v1/*` | Client REST API | Client API |
| `api.eusolicit.com` | Enterprise public API | Client API |
| `admin.eusolicit.com` | Admin panel | Admin API |
| `*.eusolicit.com` | White-label subdomains | Frontend |
| `stage.eusolicit.com` | Staging | All services |

### 14.3 Environment Strategy

| Environment | Purpose | Infra | KraftData |
|---|---|---|---|
| `local` | Developer workstation | Docker Compose (all 5 services) | KraftData stage |
| `staging` | Integration testing, QA | K8s (small cluster) | KraftData stage |
| `production` | Live SaaS at EUSolicit.com | K8s (HA cluster) | KraftData production |

### 14.4 CI/CD Pipeline (GitHub Actions)

```
PR opened  → lint + type check + unit tests (matrix: all 5 services in parallel)
PR merged  → integration tests → build Docker images (matrix) → push to registry
Tag v*.*.*  → deploy to stage.eusolicit.com → smoke tests → manual approval → deploy to eusolicit.com
```

Each service is independently versioned and deployable. A change to Admin API does not trigger a Client API rebuild.

---

## 15. Observability

| Layer | Tool | What It Captures |
|---|---|---|
| Metrics | Prometheus + Grafana | Per-service API latency, error rates, DB pool, Celery queue depth, usage meters, SSE stream count (AI Gateway) |
| Logs | structlog → Loki → Grafana | Structured JSON, searchable by request_id, company_id, tier, service_name |
| Traces | OpenTelemetry → Jaeger | Distributed traces across all 5 services + KraftData call duration |
| Uptime | Grafana Alerting | Per-service health checks, SLA tracking (99.5%) |
| KraftData | KraftData eval-runs | Agent quality scores, drift detection |
| Business | Custom Grafana | Signup funnel, tier conversion, usage patterns, churn |

Trace propagation: `X-Request-ID` and `traceparent` headers propagated across all inter-service HTTP calls. Redis Stream events include `trace_id` in payload.

---

## 16. Post-MVP Integration Readiness

| Integration | Design Pattern | Preparation in MVP |
|---|---|---|
| E-procurement portals | `SubmissionPort` in Client API | Interface defined |
| ERP (SAP, 1C) | Webhook adapter in Client API | Generic webhook receiver |
| CRM (Salesforce, HubSpot) | `CRMProviderPort` in Client API | OAuth2 infra from calendar |
| Slack / Teams | `NotificationChannelPort` in Notification Svc | Alert system is channel-agnostic |
| Digital signatures (eIDAS) | `SignatureProviderPort` in Client API | — |
| Microsoft OAuth2 | `authlib` in Client API | Same pattern as Google |
| Consultant marketplace | New domain in Client API | Company profiles vault exists |

---

## 17. MVP Development Phases

### Phase 1 — Foundation + Service Scaffold (Weeks 1–4)
- Monorepo setup with 5 service directories + shared libraries
- Per-service Dockerfiles, Helm charts, GitHub Actions matrix
- Docker Compose for local development (all services)
- Auth system (email/password + Google OAuth2 + JWT RS256) in Client API
- Entity-level RBAC middleware (`entity_permissions` table + `EntityPermission` middleware)
- User and company profile CRUD (including `tax_id` for VAT, `espd_profiles`)
- Subscription/tier model + tier gating + usage gate middleware
- 14-day trial flow (Stripe trial subscriptions, downgrade on expiry)
- Stripe integration (checkout, webhooks, customer portal, Stripe Tax for EU VAT)
- Per-bid add-on purchase flow (Stripe Checkout one-time payments)
- Audit trail middleware + shared audit log
- Admin API scaffold + platform admin auth
- Redis Streams event bus setup (7 streams, 4 consumer groups)

### Phase 2 — Data Pipeline + AI Gateway (Weeks 5–7)
- AI Gateway service with KraftData client, circuit breaker, execution logging
- AOP crawler agent integration (Data Pipeline triggers KraftData agent via AI Gateway)
- TED crawler agent + EU grant portal crawler agent (same pattern)
- Data normalization team integration
- Central data store schema + upsert pipeline
- Submission guide auto-generation from crawl metadata (via Submission Guide Agent)
- Celery Beat scheduling in Data Pipeline
- `opportunities.ingested` event publishing
- Document upload + S3 storage + ClamAV scanning

### Phase 3 — Core Platform (Weeks 8–12)
- Opportunity search + listing (free-tier limited view, paid-tier gated)
- Opportunity detail with tier access enforcement + submission guides display
- AI summary generation (via AI Gateway) + usage metering
- Alert preferences UI + email digest system in Notification Service
- Compliance framework admin UI in Admin API + per-opportunity assignment + auto-suggestion
- Compliance checker agent integration (via AI Gateway)
- ESPD auto-fill (structured form + ESPD Agent for AI-assisted field mapping)
- iCal feed generation in Client API
- Task orchestration engine (task CRUD, dependency graph, templates, deadline notifications)
- Approval workflow system (configurable stages, sign-off decisions, notifications)
- Reusable content blocks library (CRUD, tagging, versioning, approval)

### Phase 4 — Proposal & Intelligence Tools (Weeks 13–17)
- Proposal generator workflow (Client API → AI Gateway SSE proxy)
- Proposal collaboration (per-proposal roles, section locking, comments)
- Requirement checklist builder
- Scoring simulator with scorecard UI
- Pricing assistant
- Bid/no-bid decision workflow with AI scorecard + user override
- Bid outcome tracking + preparation time/cost logging + Lessons Learned Agent
- Consortium finder agent integration (EU grant proposals)
- Document export (PDF/DOCX) for proposals, checklists, compliance reports
- Google Calendar sync in Notification Service
- Microsoft Outlook sync in Notification Service

### Phase 5 — Analytics, Reporting & Launch (Weeks 18–22)
- Analytics dashboards (market intelligence, ROI tracker, team performance)
- Competitor intelligence view + Pipeline forecasting view
- Usage dashboard
- Analytics materialized views + scheduled refresh
- Report generation + export (PDF/DOCX, scheduled + on-demand)
- Bid History Analytics Agent integration
- Reporting Template Generator for post-award (grant winners)
- White-label configuration in Admin API (subdomain + branding)
- Performance optimization + load testing
- Security audit (including network policy verification, entity RBAC audit)
- API documentation (auto-generated OpenAPI spec for Enterprise tier)
- User guides + onboarding flow (including trial-to-paid conversion UX)

**Estimated MVP timeline: 22 weeks** with a small team (2-3 developers) using AI-assisted development. The 2-week increase from v3 accounts for task orchestration, approval workflows, proposal collaboration, trial/add-on billing, and submission guides.

---

*Document version: 4.0 | Date: 2026-04-05 | Status: Draft*
