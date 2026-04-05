# EU Solicit — Service Decomposition Analysis

**Platform**: EU Solicit | **Domain**: EUSolicit.com | **Version**: 1.1 | **Date**: 2026-04-04 | **Status**: Decision Document

---

## 1. Context & Decision

The current architecture defines a modular monolith with 9 domain modules. This document analyzes whether to decompose into services, identifies the natural cut points, and recommends a deployment strategy.

### 1.1 Why Consider Splitting Now

The current module map reveals two fundamentally different user populations with different scaling profiles, security postures, and release cadences:

- **Admin surface**: Platform administrators managing compliance frameworks, crawler configuration, tenant oversight, audit logs. Low traffic, high privilege, slow release cycle.
- **Client surface**: End users searching opportunities, generating proposals, configuring alerts, syncing calendars. High traffic, tier-gated, fast iteration cycle.

Beyond this, the **data ingestion pipeline** (crawlers + normalization) is a background process with no user-facing latency requirement — it should scale independently and fail without taking the client API down.

### 1.2 Key Architectural Constraints

- **KraftData agent management**: All agents, teams, workflows, and vector stores (storage resources) are created and configured manually in the KraftData admin interface. EU Solicit consumes them via the AI Gateway; it does not manage AI model definitions.
- **GDPR / Data residency**: All tender data and client information must be stored within EU data centres. This applies to PostgreSQL, Redis, S3/MinIO, and managed services.
- **Document retention**: Documents retained for opportunity lifetime + 2 years (configurable per tenant). Audit logs are never deleted.
- **MVP scope boundary**: Electronic bid submission, Microsoft OAuth2, e-procurement integration, ERP/CRM, Slack/Teams, digital signatures, and consultant marketplace are all out of scope for MVP.

### 1.3 Recommendation: Hybrid — Core Services Now, Consolidate the Rest

Start with **5 services** from day one. This gives you the critical isolation benefits (admin vs client, pipeline vs API) without the operational overhead of 10+ microservices. Each service has a clear scaling profile, failure domain, and team ownership boundary.

**Rationale against full monolith for MVP**: Given that you're deploying to Kubernetes from the start and using AI-driven development (fast code generation), the marginal cost of splitting into 5 services is low. The operational complexity is manageable with Helm charts. The benefit — independent scaling, isolated failure, and clean security boundaries — pays off immediately.

**Rationale against full microservices**: Splitting every domain module into its own service (opportunities, proposals, compliance, alerts, calendar, analytics, audit...) would create 10+ services for a 2-3 person team. The coordination overhead, distributed transaction complexity, and debugging difficulty would slow down MVP delivery.

---

## 2. Service Decomposition

### 2.1 Service Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTERNET                                       │
└──────┬──────────────────────────────────────────────────────────────────────┘
       │
┌──────▼──────┐
│  CloudFlare │
│  CDN + WAF  │
└──────┬──────┘
       │
┌──────▼──────────────────────────────────────── K8s Cluster ─────────────────┐
│                                                                             │
│  ┌───────────────────┐                                                     │
│  │  NGINX INGRESS    │                                                     │
│  │                   │                                                     │
│  │  eusolicit.com    │──► Frontend (Next.js)                               │
│  │  eusolicit.com/api│──► Client API Service                               │
│  │  admin.eusolicit  │──► Admin API Service                                │
│  │  api.eusolicit    │──► Client API Service (Enterprise public API)       │
│  └───────────────────┘                                                     │
│                                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐ │
│  │ 1. CLIENT   │  │ 2. ADMIN    │  │ 3. DATA     │  │ 4. AI            │ │
│  │    API      │  │    API      │  │    PIPELINE  │  │    GATEWAY       │ │
│  │    SERVICE  │  │    SERVICE  │  │    SERVICE   │  │    SERVICE       │ │
│  │             │  │             │  │              │  │                  │ │
│  │ Opps search │  │ Compliance  │  │ Crawlers     │  │ KraftData client │ │
│  │ Proposals   │  │ frameworks  │  │ Normalization│  │ Agent execution  │ │
│  │ Alerts      │  │ Crawler     │  │ Enrichment   │  │ SSE proxy        │ │
│  │ Calendar    │  │ mgmt        │  │ Upsert       │  │ Webhook handler  │ │
│  │ Bid/no-bid  │  │ Tenant      │  │              │  │ Storage mgmt     │ │
│  │ Analytics   │  │ mgmt        │  │              │  │                  │ │
│  │ Documents   │  │ Audit logs  │  │              │  │                  │ │
│  │ Auth (user) │  │ White-label │  │              │  │                  │ │
│  │ Billing     │  │ Auth (admin)│  │              │  │                  │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬───── ┘  └────────┬─────────┘ │
│         │                │                │                   │           │
│         │     ┌──────────┴───────────┐    │                   │           │
│         │     │ 5. NOTIFICATION      │    │                   │           │
│         │     │    SERVICE           │    │                   │           │
│         │     │                      │    │                   │           │
│         │     │ Email delivery       │    │                   │           │
│         │     │ Alert digest         │    │                   │           │
│         │     │ Calendar sync        │    │                   │           │
│         │     │ (Celery workers)     │    │                   │           │
│         │     └──────────┬───────────┘    │                   │           │
│         │                │                │                   │           │
│     ┌───┴────────────────┴────────────────┴───────────────────┘           │
│     │                                                                     │
│  ┌──▼───────────┐  ┌────────────┐  ┌────────────┐  ┌──────────────┐     │
│  │ PostgreSQL   │  │ Redis      │  │ S3 / MinIO │  │ KraftData    │     │
│  │ (shared DB   │  │ (shared)   │  │            │  │ Platform     │     │
│  │  with schema │  │            │  │            │  │ (external)   │     │
│  │  isolation)  │  │            │  │            │  │              │     │
│  └──────────────┘  └────────────┘  └────────────┘  └──────────────┘     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Service Definitions

### Service 1: Client API Service

**Purpose**: All client-facing (end-user) functionality. This is the primary revenue-generating surface.

**Owns**:
- Opportunity search, listing, detail (tier-gated)
- Proposal generation, versioning, export
- Bid/no-bid decision workflow
- Bid outcome recording
- Alert preference management
- Calendar connection management + iCal feed
- Analytics dashboards (market intelligence, competitor, forecast, usage)
- Document upload/download
- Company profile management
- User authentication (email/password + Google OAuth2)
- Subscription management + Stripe integration
- Tier gating + usage metering

**Does NOT own**:
- Compliance framework CRUD (admin)
- Crawler configuration (admin)
- Tenant/white-label management (admin)
- Actual AI agent execution (delegates to AI Gateway)
- Email sending, calendar sync execution (delegates to Notification Service)
- Data crawling/ingestion (delegates to Data Pipeline)

**Scaling profile**: High traffic, horizontally scaled. 3-20 pods via HPA. Latency-sensitive.

**Database**: Read/write access to `client` schema (opportunities, proposals, companies, subscriptions, alerts, calendar, analytics, documents, bid_decisions, bid_outcomes, usage_meters).

**API base**: `eusolicit.com/api/v1/*` and `api.eusolicit.com/v1/*` (Enterprise public API).

---

### Service 2: Admin API Service

**Purpose**: Platform administration, compliance management, and operational oversight. Accessed only by platform admins.

**Owns**:
- Compliance framework CRUD + assignment to opportunities
- Framework auto-suggestion configuration
- Crawler management (enable/disable, scheduling)
- Tenant management (view all companies, subscriptions, usage)
- White-label configuration
- Audit log viewer
- Platform-level analytics (signup funnel, churn, agent quality)
- Admin authentication (separate JWT with `platform_admin` role)
- Regulation tracker agent management

**Does NOT own**:
- Client-facing opportunity views
- Proposal generation
- User self-service

**Scaling profile**: Low traffic. 1-2 pods, no HPA. Security-critical — hardened network policy.

**Database**: Read/write access to `admin` schema (compliance_frameworks, whitelabel_configs, audit_log). Read-only access to `client` schema for reporting.

**API base**: `admin.eusolicit.com/api/v1/*` (VPN or IP-restricted in production).

**Network policy**: Only accessible from admin frontend or VPN. No public internet exposure.

---

### Service 3: Data Pipeline Service

**Purpose**: Continuous crawling, normalization, enrichment, and ingestion of opportunity data into the central store.

**Owns**:
- Celery Beat scheduler (crawl schedules)
- AOP, TED, EU Grant crawler tasks
- Data normalization (delegates to KraftData Normalization Team via AI Gateway)
- AI enrichment (delegates summary/tagging to KraftData agents via AI Gateway)
- Opportunity upsert into PostgreSQL
- Expired opportunity cleanup
- Alert trigger events (publishes "new_opportunities" events to Redis Streams)

**Does NOT own**:
- Serving opportunity data to users (that's Client API)
- Any user-facing endpoints

**Scaling profile**: Background processing. 2-8 Celery workers via HPA. Burst scaling during crawl windows. Failure-tolerant — if pipeline goes down, client API keeps serving stale data.

**Database**: Read/write access to `pipeline` schema (opportunities table write, crawler_runs for tracking). No access to client tables.

**Communication**: Publishes events to Redis Streams when new opportunities are ingested. Client API and Notification Service subscribe.

---

### Service 4: AI Gateway Service

**Purpose**: Single point of integration with KraftData. Wraps all agent, team, workflow, and storage resource API calls. Provides a stable internal API that insulates other services from KraftData API changes.

**Owns**:
- KraftData HTTP client (all `/client/api/v1/*` calls)
- Agent execution (run, run-stream)
- Team execution
- Workflow execution
- Storage resource file upload (via KraftData API — vector stores are provisioned externally)
- KraftData webhook receiver (inbound)
- SSE stream proxying
- Agent ID registry (maps logical names to externally-managed KraftData agent IDs)
- Retry logic, circuit breaking, timeout management for KraftData calls
- KraftData eval-run execution for quality monitoring

**Does NOT own**:
- Business logic about WHAT to analyze or generate (that's Client API or Admin API)
- Deciding WHEN to run agents (that's the caller's responsibility)

**Scaling profile**: Medium traffic, I/O-bound. 2-10 pods. Each SSE stream holds a long-lived connection, so connection pooling and pod count matter. Scale based on active SSE stream count.

**Database**: Minimal. Writes to a `gateway` schema for logging agent execution history and latency metrics. No access to opportunity or client data.

**API base**: Internal ClusterIP only. Not exposed to the internet.

**Why a separate service?**

This is the most important split after admin vs client. Reasons:

1. **Blast radius**: If KraftData has an outage, the AI Gateway absorbs the failure. Client API continues serving cached data, search, and static features. Without this isolation, a KraftData timeout could cascade into the entire platform going down.
2. **Rate limiting**: KraftData has its own rate limits. The gateway manages a single connection pool, applies backpressure, and queues requests. Without centralization, each service would independently hammer KraftData.
3. **Contract stability**: If KraftData changes their API (new auth scheme, endpoint changes, payload format), only the gateway needs updating. Client API and Admin API are unaffected.
4. **SSE stream management**: Long-lived SSE connections need different scaling characteristics than short-lived REST calls. Isolating them prevents stream-heavy workloads from starving the request pool.

---

### Service 5: Notification Service

**Purpose**: All outbound communication and scheduled background tasks that aren't data ingestion.

**Owns**:
- Alert digest email assembly and delivery (via SendGrid)
- Email templates and rendering
- Calendar sync execution (Google Calendar API, Microsoft Graph API)
- Celery workers for:
  - Alert digest generation (immediate, daily, weekly)
  - Calendar sync (every 15 min)
  - Usage meter sync to Stripe (daily)
  - Scheduled report generation
- OAuth token refresh for Google and Microsoft

**Does NOT own**:
- Alert preference configuration (that's Client API)
- Calendar connection setup (that's Client API — OAuth redirect flow)
- Deciding what to send (Client API / Data Pipeline publishes events; Notification Service reacts)

**Scaling profile**: Burst-oriented. Low baseline, scales up during digest windows (8 AM UTC daily, Monday weekly). 1-4 Celery workers.

**Database**: Read access to `client` schema (alert_preferences, calendar_connections, calendar_events, opportunities). Write access to `notification` schema (alert_log, email_log, sync_log).

**Communication**: Subscribes to Redis Streams for "new_opportunities" events from Data Pipeline. Also runs on Celery Beat schedule.

---

## 4. Inter-Service Communication

### 4.1 Communication Matrix

| From → To | Method | Use Case |
|---|---|---|
| Client API → AI Gateway | REST (sync) + SSE proxy | Agent execution, proposal generation |
| Client API → Notification Service | Redis Streams (async) | "alert_preference_updated", "calendar_connected" |
| Admin API → AI Gateway | REST (sync) | Compliance checker, framework suggestion agent |
| Admin API → Data Pipeline | Redis Streams (async) | "crawler_schedule_updated", "framework_assigned" |
| Data Pipeline → AI Gateway | REST (sync) | Normalization team, enrichment agents |
| Data Pipeline → Notification Service | Redis Streams (async) | "new_opportunities_ingested" |
| Stripe → Client API | Webhook (HTTP) | Subscription events |
| KraftData → AI Gateway | Webhook (HTTP) | Agent/workflow completion events |
| AI Gateway → Client API | Redis Streams (async) | "agent_completed", "workflow_completed" |

### 4.2 Event Catalog

| Event | Publisher | Subscribers | Payload |
|---|---|---|---|
| `opportunities.ingested` | Data Pipeline | Notification Service, Client API (cache invalidation) | `{opportunity_ids: [...], source: "aop"}` |
| `opportunity.status_changed` | Data Pipeline | Notification Service | `{opportunity_id, old_status, new_status}` |
| `agent.execution.completed` | AI Gateway | Client API | `{execution_id, agent_type, result_summary}` |
| `agent.execution.failed` | AI Gateway | Client API, Admin API (alerting) | `{execution_id, agent_type, error}` |
| `alert_preference.updated` | Client API | Notification Service | `{preference_id, user_id}` |
| `calendar.connected` | Client API | Notification Service | `{connection_id, user_id, provider}` |
| `subscription.changed` | Client API | All services (tier cache refresh) | `{company_id, old_tier, new_tier}` |
| `compliance_framework.assigned` | Admin API | Client API, AI Gateway | `{opportunity_id, framework_id}` |
| `crawler.schedule_updated` | Admin API | Data Pipeline | `{crawler_type, new_schedule}` |

### 4.3 Redis Streams Configuration

```python
# Shared event bus via Redis Streams
STREAMS = {
    "eu-solicit:opportunities": "Data pipeline opportunity events",
    "eu-solicit:agents": "AI Gateway agent execution events",
    "eu-solicit:subscriptions": "Subscription lifecycle events",
    "eu-solicit:notifications": "Notification trigger events",
    "eu-solicit:admin": "Admin action events",
}

# Consumer groups — each service has its own consumer group
# ensuring at-least-once delivery per service
CONSUMER_GROUPS = {
    "client-api": ["eu-solicit:opportunities", "eu-solicit:agents", "eu-solicit:subscriptions"],
    "notification-svc": ["eu-solicit:opportunities", "eu-solicit:notifications"],
    "data-pipeline": ["eu-solicit:admin"],
    "ai-gateway": ["eu-solicit:admin"],
}
```

---

## 5. Database Strategy

### 5.1 Shared Database, Schema Isolation

For the MVP, all 5 services share **one PostgreSQL instance** but operate on **isolated schemas** with enforced access boundaries. This avoids the operational complexity of multiple databases while maintaining data ownership clarity.

```
PostgreSQL 16 Instance
├── Schema: client
│   ├── users
│   ├── companies
│   ├── subscriptions
│   ├── tier_access_policies
│   ├── proposals
│   ├── proposal_versions
│   ├── bid_decisions
│   ├── bid_outcomes
│   ├── alert_preferences
│   ├── calendar_connections
│   ├── calendar_events
│   ├── documents
│   ├── competitor_records
│   └── whitelabel_configs
│
├── Schema: pipeline
│   ├── opportunities        ← Central data store
│   ├── crawler_runs         ← Crawl execution log
│   └── enrichment_queue     ← Pending AI enrichment jobs
│
├── Schema: admin
│   ├── compliance_frameworks
│   └── platform_settings
│
├── Schema: notification
│   ├── alert_log
│   ├── email_log
│   └── sync_log
│
├── Schema: gateway
│   ├── agent_executions     ← Execution log + latency metrics
│   └── webhook_log
│
└── Schema: shared
    ├── audit_log            ← Immutable, all services write
    └── usage_meters         ← Redis-backed, persisted daily
```

### 5.2 Access Control via DB Roles

| Service | DB Role | Schemas |
|---|---|---|
| Client API | `client_api_role` | `client` (read/write), `pipeline` (read-only: opportunities), `shared` (write: audit, usage) |
| Admin API | `admin_api_role` | `admin` (read/write), `client` (read-only), `pipeline` (read-only), `shared` (read/write) |
| Data Pipeline | `pipeline_role` | `pipeline` (read/write), `shared` (write: audit) |
| AI Gateway | `gateway_role` | `gateway` (read/write), `shared` (write: audit) |
| Notification Service | `notification_role` | `notification` (read/write), `client` (read-only: alerts, calendar), `pipeline` (read-only: opportunities) |

### 5.3 Evolution Path

Post-MVP, high-traffic schemas (`pipeline`, `notification`) can be extracted to their own PostgreSQL instances without code changes — only connection strings need updating, since each service already has its own DB role and connection pool.

---

## 6. Shared Libraries (Python Packages)

To avoid code duplication across 5 services, extract common code into internal Python packages:

```
packages/
├── eusolicit-common/
│   ├── auth/               # JWT verification, RBAC decorators
│   ├── middleware/          # Rate limiting, audit logging
│   ├── events/             # Redis Streams publisher/consumer
│   ├── config/             # Pydantic Settings base classes
│   ├── logging/            # structlog configuration
│   ├── exceptions/         # Domain exception hierarchy
│   └── health/             # Health check endpoint factory
│
├── eusolicit-kraftdata/    # KraftData client library
│   ├── client.py           # HTTP client wrapper
│   ├── agents.py
│   ├── teams.py
│   ├── workflows.py
│   └── storage.py
│
└── eusolicit-models/       # Shared Pydantic DTOs for inter-service contracts
    ├── events.py           # Event payload schemas
    ├── opportunity.py      # Shared opportunity DTOs
    └── subscription.py     # Tier/subscription DTOs
```

Packaged as private pip packages (installed via Git URL or private PyPI). Each service's `pyproject.toml` declares its dependencies on these shared packages.

---

## 7. Deployment Topology (Kubernetes)

```
Namespace: eu-solicit
│
├── Deployments
│   ├── frontend            (Next.js)              — 2 pods, HPA 2-10
│   ├── client-api          (FastAPI)              — 3 pods, HPA 3-20
│   ├── admin-api           (FastAPI)              — 1 pod,  HPA 1-3
│   ├── ai-gateway          (FastAPI)              — 2 pods, HPA 2-10
│   ├── data-pipeline-worker (Celery Worker)       — 2 pods, HPA 2-8
│   ├── data-pipeline-beat   (Celery Beat)         — 1 pod  (singleton)
│   ├── notification-worker  (Celery Worker)       — 1 pod,  HPA 1-4
│   ├── notification-beat    (Celery Beat)         — 1 pod  (singleton)
│   └── clamav              (Virus scanner)        — 1 pod
│
├── StatefulSets
│   ├── postgresql          — 1 primary + 1 replica (or managed RDS)
│   └── redis               — 1 pod (or managed ElastiCache)
│
├── Services (ClusterIP)
│   ├── frontend-svc
│   ├── client-api-svc
│   ├── admin-api-svc
│   ├── ai-gateway-svc      ← Internal only, no ingress
│   ├── clamav-svc           ← Internal only
│   └── redis-svc
│
├── Ingress (nginx)
│   ├── eusolicit.com                → frontend-svc
│   ├── eusolicit.com/api/v1/*       → client-api-svc
│   ├── api.eusolicit.com/v1/*       → client-api-svc (Enterprise API)
│   ├── *.eusolicit.com              → frontend-svc (white-label)
│   └── admin.eusolicit.com/*        → admin-api-svc (IP-restricted)
│
├── NetworkPolicies
│   ├── admin-api: ingress from VPN/office IPs only
│   ├── ai-gateway: ingress from client-api, admin-api, data-pipeline only
│   ├── clamav: ingress from client-api only
│   └── data-pipeline: no ingress (outbound only to AI gateway + DB)
│
└── Secrets
    └── External Secrets Operator → AWS Secrets Manager
        ├── client-api-secrets
        ├── admin-api-secrets
        ├── ai-gateway-secrets (KraftData API key)
        ├── data-pipeline-secrets
        └── notification-secrets (SendGrid, Google, Microsoft OAuth)
```

---

## 8. Monolith → Services Migration Strategy

Since the current codebase is a modular monolith with clean domain boundaries, the migration is mechanical:

### Phase 0: Prepare (Week 1)
- Extract shared libraries (`eusolicit-common`, `eusolicit-kraftdata`, `eusolicit-models`).
- Set up monorepo structure with service directories.
- Configure per-service Dockerfiles and Helm charts.

### Phase 1: AI Gateway extraction (Week 2)
- Move `infrastructure/kraftdata/` into its own FastAPI service.
- Expose internal REST API matching current domain interface methods.
- Update Client API to call AI Gateway via HTTP instead of direct import.
- **Validation**: All AI operations (summary, proposal gen, compliance check) work via gateway.

### Phase 2: Data Pipeline extraction (Week 2-3)
- Move `workers/` and `infrastructure/crawlers/` into its own service.
- Set up Redis Streams for `opportunities.ingested` events.
- Client API subscribes to cache invalidation events.
- **Validation**: Crawlers run independently; new opportunities appear in client search.

### Phase 3: Admin API extraction (Week 3)
- Move admin routers (`compliance.py`, `audit.py`) and admin-specific logic into its own FastAPI service.
- Set up `admin.eusolicit.com` ingress with IP restriction.
- **Validation**: Admin can manage frameworks; client API unaffected by admin deployments.

### Phase 4: Notification Service extraction (Week 4)
- Move alert digest, calendar sync, and email tasks into their own Celery workers.
- Subscribe to Redis Streams events from Data Pipeline.
- **Validation**: Alert emails and calendar sync work end-to-end.

---

## 9. Decision Matrix: What Stays Together, What Splits

| Domain Module | Service Assignment | Rationale |
|---|---|---|
| Opportunities (search, detail) | **Client API** | Core user-facing query surface. High traffic. |
| Proposals | **Client API** | Tightly coupled to opportunities and user context. |
| Bid/No-Bid | **Client API** | Part of the user bidding workflow. |
| Analytics | **Client API** | Serves user dashboards. |
| Companies | **Client API** | User self-service. |
| Subscriptions / Billing | **Client API** | Stripe webhooks + user checkout flow. |
| Alert preferences | **Client API** | User configuration. |
| Calendar connections | **Client API** | OAuth flow is user-initiated. |
| Documents | **Client API** | Upload/download is user-facing. |
| Compliance frameworks | **Admin API** | Admin-only CRUD. Different security posture. |
| Audit log | **Admin API** (viewer) / **Shared** (writer) | All services write, admin reads. |
| White-label config | **Admin API** | Tenant configuration. |
| Crawlers | **Data Pipeline** | Background, independently scaled. |
| Normalization + Enrichment | **Data Pipeline** → **AI Gateway** | Pipeline orchestrates, gateway executes. |
| All KraftData calls | **AI Gateway** | Single integration point. Circuit breaking. |
| Email delivery | **Notification Service** | Background, burst-oriented. |
| Calendar sync | **Notification Service** | Scheduled background task. |
| Alert digest | **Notification Service** | Scheduled background task. |

---

## 10. Trade-off Analysis

### Benefits of This Split

1. **Admin isolation**: Admin service is VPN-restricted, independently deployable, and can't be DDoS'd via the public client API.
2. **Pipeline resilience**: Crawler failures don't impact the client API. Users keep searching and generating proposals from cached data.
3. **AI blast radius**: KraftData outages are absorbed by the AI Gateway. The client API degrades gracefully (disables AI features, serves cached summaries).
4. **Independent scaling**: Client API scales for user traffic. Data Pipeline scales for crawl volume. AI Gateway scales for SSE stream count. No waste.
5. **Targeted deployments**: Ship a proposal UI fix without restarting crawlers. Update a compliance framework without touching billing.

### Costs

1. **Operational overhead**: 5 services × (Dockerfile + Helm chart + CI pipeline + health monitoring) = more infrastructure to maintain.
2. **Distributed debugging**: Cross-service request tracing requires OpenTelemetry investment. A single monolith stack trace is simpler.
3. **Event consistency**: Redis Streams provides at-least-once delivery, not exactly-once. Services must be idempotent.
4. **Shared DB coupling**: Schema isolation is a convention, not a hard boundary. A rogue migration in one service could impact others. Mitigated by per-service DB roles and migration governance.

### Mitigations

- **OpenTelemetry**: Already planned. Trace IDs propagated in all inter-service HTTP headers.
- **Shared libraries**: Reduce code duplication to near-zero.
- **Helm chart templates**: Reusable chart with per-service values files.
- **GitHub Actions matrix**: Single CI workflow that builds all services in parallel.

---

## 11. Comparison: Monolith vs Hybrid SOA vs Full Microservices

| Factor | Modular Monolith | Hybrid SOA (5 services) | Full Microservices (10+) |
|---|---|---|---|
| MVP delivery speed | Fastest | Moderate (+2 weeks) | Slowest (+6 weeks) |
| Operational complexity | Low | Moderate | High |
| Admin isolation | Middleware only | Network-level isolation | Network-level isolation |
| Pipeline resilience | Shared process | Independent failure domain | Independent failure domain |
| AI blast radius | Shared process | Isolated gateway | Isolated gateway |
| Independent scaling | Not possible | Per-service HPA | Per-module HPA |
| Team scaling | 1-3 devs | 2-5 devs | 5+ devs |
| Distributed debugging | N/A | Moderate (5 services) | Hard (10+ services) |
| Database complexity | Single schema | Schema isolation | Separate databases |
| Event infrastructure | Not needed | Redis Streams | Kafka / RabbitMQ |

**Verdict**: Hybrid SOA (5 services) is the right choice for EU Solicit given the K8s deployment target, the clear admin/client security boundary, and the KraftData integration risk that warrants an isolated gateway.

---

## 12. Updated Timeline Impact

| Phase | Monolith Timeline | SOA Timeline | Delta |
|---|---|---|---|
| Phase 1: Foundation | 3 weeks | 4 weeks (+shared libs, service scaffold) | +1 week |
| Phase 2: Data Pipeline | 3 weeks | 3 weeks (already a separate service) | 0 |
| Phase 3: Core Platform | 4 weeks | 4.5 weeks (inter-service events) | +0.5 weeks |
| Phase 4: Proposal Tools | 4 weeks | 4 weeks (AI Gateway absorbs complexity) | 0 |
| Phase 5: Analytics & Launch | 4 weeks | 4.5 weeks (cross-service dashboards) | +0.5 weeks |
| **Total** | **18 weeks** | **20 weeks** | **+2 weeks** |

The 2-week overhead is a worthwhile investment given the architectural benefits. It also makes Phase 5 (analytics) cleaner since the data pipeline is already decoupled and event-driven.

---

*Document version: 1.1 | Date: 2026-04-04 | Status: Decision Document*
