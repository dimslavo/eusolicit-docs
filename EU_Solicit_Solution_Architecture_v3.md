# EU Solicit — Solution Architecture v3 (Service-Oriented)

**Platform**: EU Solicit | **Domain**: EUSolicit.com | **Version**: 3.1 | **Date**: 2026-04-04 | **Status**: Draft

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

**Owns**: Opportunity search/listing/detail (tier-gated), proposal generation/versioning/export, bid/no-bid workflow, bid outcome recording, alert preference management, calendar connection management + iCal feed, analytics dashboards, document upload/download, company profiles, user auth (email/password + Google OAuth2), subscription management + Stripe, tier gating + usage metering.

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

**Owns**: Celery Beat scheduler, AOP/TED/EU Grant crawler tasks, normalization (via AI Gateway), enrichment (via AI Gateway), opportunity upsert, expired opportunity cleanup. Publishes `opportunities.ingested` events to Redis Streams.

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
| `compliance_framework.assigned` | Admin API | Client API, AI Gateway | `{opportunity_id, framework_id}` |
| `crawler.schedule_updated` | Admin API | Data Pipeline | `{crawler_type, new_schedule}` |

### 5.3 Redis Streams Configuration

```python
STREAMS = {
    "eu-solicit:opportunities": "Data pipeline events",
    "eu-solicit:agents": "AI Gateway execution events",
    "eu-solicit:subscriptions": "Subscription lifecycle events",
    "eu-solicit:notifications": "Notification trigger events",
    "eu-solicit:admin": "Admin action events",
}

CONSUMER_GROUPS = {
    "client-api": ["eu-solicit:opportunities", "eu-solicit:agents", "eu-solicit:subscriptions"],
    "notification-svc": ["eu-solicit:opportunities", "eu-solicit:notifications"],
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
│   │   ├── subscriptions/      # Tier gating, usage metering, Stripe
│   │   ├── companies/          # Profiles, team members, credentials
│   │   ├── alerts/             # Preference management (delivery delegated)
│   │   ├── calendar/           # Connection management, iCal feed
│   │   ├── analytics/          # Dashboards, competitor, forecast, usage
│   │   └── audit/              # Write-only audit logging
│   ├── application/
│   │   ├── commands/           # register_company, create_proposal, bid_decision, etc.
│   │   ├── queries/            # search_opportunities, get_analytics, etc.
│   │   └── dtos/
│   ├── infrastructure/
│   │   ├── api/
│   │   │   ├── app.py
│   │   │   ├── middleware/     # auth, tier_gate, usage_gate, rate_limit, audit
│   │   │   └── routers/       # opportunities, proposals, companies, subscriptions,
│   │   │                      # alerts, calendar, analytics, bid_decisions, documents
│   │   ├── persistence/       # SQLAlchemy models for client + shared schemas
│   │   ├── ai_gateway_client/ # HTTP client to call AI Gateway service
│   │   ├── billing/           # Stripe client + webhook handler
│   │   ├── storage/           # S3 client + ClamAV
│   │   ├── event_bus/         # Redis Streams publisher + consumer
│   │   └── auth/              # JWT + Google OAuth2
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
│   │   ├── crawling/           # Crawler orchestration logic
│   │   ├── normalization/      # Data standardization rules
│   │   └── enrichment/         # AI enrichment orchestration
│   ├── infrastructure/
│   │   ├── crawlers/
│   │   │   ├── aop_crawler.py
│   │   │   ├── ted_crawler.py
│   │   │   └── eu_grants_crawler.py
│   │   ├── persistence/       # SQLAlchemy models for pipeline schema
│   │   ├── ai_gateway_client/ # HTTP client to AI Gateway
│   │   └── event_bus/         # Redis Streams publisher
│   ├── workers/
│   │   ├── celery_app.py
│   │   ├── beat_schedule.py
│   │   └── tasks/
│   │       ├── crawl_opportunities.py
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
│   ├── subscriptions
│   ├── tier_access_policies
│   ├── proposals / proposal_versions
│   ├── bid_decisions
│   ├── bid_outcomes
│   ├── alert_preferences
│   ├── calendar_connections
│   ├── calendar_events
│   ├── documents
│   ├── competitor_records
│   └── whitelabel_configs
│
├── Schema: pipeline            ← Data Pipeline Service (read/write)
│   ├── opportunities           ← Central data store
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

All entity definitions from v2 remain unchanged. For reference, the core entities and their schema assignments:

| Entity | Schema | Owner Service |
|---|---|---|
| `opportunities` | `pipeline` | Data Pipeline |
| `compliance_frameworks` | `admin` | Admin API |
| `users`, `companies` | `client` | Client API |
| `subscriptions`, `tier_access_policies` | `client` | Client API |
| `proposals`, `proposal_versions` | `client` | Client API |
| `bid_decisions`, `bid_outcomes` | `client` | Client API |
| `alert_preferences` | `client` | Client API |
| `calendar_connections`, `calendar_events` | `client` | Client API |
| `documents` | `client` | Client API |
| `competitor_records` | `client` | Client API |
| `whitelabel_configs` | `client` | Client API |
| `alert_log`, `email_log`, `sync_log` | `notification` | Notification Svc |
| `agent_executions`, `webhook_log` | `gateway` | AI Gateway |
| `audit_log` | `shared` | All services (write) |
| `usage_meters` | `shared` | All services (write) |

Full field-level entity definitions: see [Solution Architecture v2, Sections 5.1–5.9](./EU_Solicit_Solution_Architecture_v2.md#5-data-model--core-entities).

---

## 10. Tier Gating & Usage Enforcement

Unchanged from v2. The `TierGate` and `UsageGate` middleware live in the Client API service. The `eusolicit-common` shared library provides the base middleware classes so Admin API can reuse the audit middleware.

Key patterns: free-tier returns `OpportunityFreeResponse` (limited metadata), paid tiers return `OpportunityFullResponse` gated by region/CPV/budget. Usage limits enforced via atomic Redis INCR with rollback on limit exceeded (429 response with upgrade prompt).

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

All agents, teams, and workflows listed below are created and maintained manually via the KraftData admin interface. EU Solicit's AI Gateway references them by logical name mapped to KraftData agent IDs in the agent registry.

| # | Name | Type | KraftData Entity | Calling Service(s) |
|---|---|---|---|---|
| 1 | Document Parser Agent | Agent | Agent | Data Pipeline |
| 2 | Executive Summary Agent | Agent | Agent | Client API |
| 3 | Compliance Checker Agent | Agent | Agent | Client API, Admin API |
| 4 | Clause Risk Analyzer Agent | Agent | Agent | Client API |
| 5 | Relevance Scoring Agent | Agent | Agent | Data Pipeline |
| 6 | Pricing Assistant Agent | Agent | Agent | Client API |
| 7 | Scoring Simulator Agent | Agent | Agent | Client API |
| 8 | Grant Eligibility Agent | Agent | Agent | Client API |
| 9 | Budget Builder Agent | Agent | Agent | Client API |
| 10 | Logframe Generator Agent | Agent | Agent | Client API |
| 11 | Bid/No-Bid Decision Agent | Agent | Agent | Client API |
| 12 | Market Intelligence Agent | Agent | Agent | Client API |
| 13 | Regulation Tracker Agent | Agent | Agent | Admin API (scheduled) |
| 14 | Lessons Learned Agent | Agent | Agent | Client API |
| 15 | Competitor Analysis Agent | Agent | Agent | Data Pipeline, Client API |
| 16 | Pipeline Forecasting Agent | Agent | Agent | Client API |
| 17 | Framework Suggestion Agent | Agent | Agent | Admin API |
| 18 | Requirement Checklist Agent | Agent | Agent | Client API |
| 19 | Win Theme Extractor Agent | Agent | Agent | Client API |
| 20 | Data Normalization Team | Team | Team | Data Pipeline |
| 21 | Proposal Generator Workflow | Workflow | Workflow | Client API (SSE) |

### 11.3 KraftData Storage Resources (Vector Stores)

Vector stores are provisioned and managed through the KraftData API (`/client/api/v1/storage-resources/*`). EU Solicit does not maintain its own vector search infrastructure — all semantic search and retrieval-augmented generation is handled by KraftData agents referencing these stores.

| # | Storage Resource | Purpose | Populated By |
|---|---|---|---|
| 1 | RFP Templates Store | Reference RFP structures and past winning proposals | Admin upload via AI Gateway |
| 2 | Regulatory Knowledge Store | ZOP, EU directives, procurement regulations | Regulation Tracker Agent |
| 3 | Company Profiles Store | Client company capabilities, past performance | Client API upload via AI Gateway |
| 4 | Historical Bids Store | Past bid outcomes, scoring data, lessons learned | Lessons Learned Agent |
| 5 | Market Data Store | Market intelligence, competitor data, pricing benchmarks | Market Intelligence Agent |
| 6 | EU Programmes Store | EU funding programme details, eligibility criteria | Data Pipeline (EU Grant crawler) |

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

All functional architectures (Stripe billing lifecycle, alert digest system, calendar sync, document upload with ClamAV) remain as defined in v2. The SOA split changes WHERE each piece runs, not HOW it works:

| Feature | Configuration/UI | Execution |
|---|---|---|
| Stripe billing | Client API (checkout, webhooks) | Notification Svc (usage sync to Stripe) |
| Email alerts | Client API (preference CRUD) | Notification Svc (matching, digest, SendGrid) |
| Calendar sync | Client API (OAuth flow, connection CRUD, iCal feed) | Notification Svc (periodic sync via Celery) |
| Documents | Client API (upload, download, metadata) | Client API (ClamAV scan, S3 storage) |

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
- User and company profile CRUD
- Subscription/tier model + tier gating + usage gate middleware
- Stripe integration (checkout, webhooks, customer portal)
- Audit trail middleware + shared audit log
- Admin API scaffold + platform admin auth
- Redis Streams event bus setup

### Phase 2 — Data Pipeline + AI Gateway (Weeks 5–7)
- AI Gateway service with KraftData client, circuit breaker, execution logging
- AOP crawler agent integration (Data Pipeline → AI Gateway → KraftData)
- TED crawler + EU grant portal crawler
- Data normalization team integration
- Central data store schema + upsert pipeline
- Celery Beat scheduling in Data Pipeline
- `opportunities.ingested` event publishing
- Document upload + S3 storage + ClamAV scanning

### Phase 3 — Core Platform (Weeks 8–11)
- Opportunity search + listing (free-tier limited view, paid-tier gated)
- Opportunity detail with tier access enforcement
- AI summary generation (via AI Gateway) + usage metering
- Alert preferences UI + email digest system in Notification Service
- Compliance framework admin UI in Admin API + per-opportunity assignment + auto-suggestion
- Compliance checker agent integration (via AI Gateway)
- iCal feed generation in Client API

### Phase 4 — Proposal & Intelligence Tools (Weeks 12–15)
- Proposal generator workflow (Client API → AI Gateway SSE proxy)
- Requirement checklist builder
- Scoring simulator with scorecard UI
- Pricing assistant
- Bid/no-bid decision workflow with AI scorecard + user override
- Bid outcome tracking + Lessons Learned Agent integration
- Document export (PDF/DOCX)
- Google Calendar sync in Notification Service
- Microsoft Outlook sync in Notification Service

### Phase 5 — Analytics & Launch (Weeks 16–20)
- Analytics dashboards (market intelligence, ROI tracker, team performance)
- Competitor intelligence view + Pipeline forecasting view
- Usage dashboard
- White-label configuration in Admin API (subdomain + branding)
- Performance optimization + load testing
- Security audit (including network policy verification)
- API documentation (auto-generated OpenAPI spec for Enterprise tier)
- User guides + onboarding flow

**Estimated MVP timeline: 20 weeks** with a small team (2-3 developers) using AI-assisted development.

---

*Document version: 3.1 | Date: 2026-04-04 | Status: Draft*
