---
stepsCompleted: ["step-01-init", "step-02-context", "step-03-starter", "step-04-decisions", "step-05-patterns", "step-06-structure", "step-07-validation", "step-08-complete"]
inputDocuments: ["PRD.md", "project-context.md"]
workflowType: 'architecture'
project_name: 'EU Solicit'
date: '2026-04-26'
---

# Architecture Decision Document

## 1. System Overview
**EU Solicit** is a SaaS platform designed to automate the lifecycle of public procurement and EU grant applications. The platform leverages KraftData's Agentic AI platform for multi-step workflows, parsing, scoring, and proposal generation. The architecture follows a microservices pattern with strict data isolation, utilizing an asynchronous event-driven approach to handle long-running AI tasks, document parsing, and external integrations (e.g., Stripe, VIES).

## 2. Component Diagram

```text
                                 +------------------+
                                 |  External APIs   |
                                 | (KraftData,      |
                                 |  Stripe, VIES,   |
                                 |  SendGrid)       |
                                 +--------+---------+
                                          |
+-------------------+            +--------v---------+
|                   |            |                  |
|  Client Frontend  | <--------> |   AI Gateway     |
|  (Next.js 14)     |    SSE     |   Service        |
|                   |            |                  |
+--------+----------+            +--------+---------+
         |                                |
         | REST API                       | REST API
         v                                v
+-------------------+            +------------------+
|                   |            |                  |
|  Client API       | <--------> |  Data Pipeline   |
|  Service          |   Redis    |  Service         |
|                   |  Streams   |                  |
+--------+----------+            +--------+---------+
         |                                |
         v                                v
+-------------------+            +------------------+
|                   |            |                  |
|  Notification     |            |  Admin API       | <--- VPN / Allowlist
|  Service          |            |  Service         |
|                   |            |                  |
+-------------------+            +------------------+
         |
+--------v------------------------------------------+
|                  PostgreSQL 16                    |
|  [client] [admin] [pipeline] [ai] [notification]  |
|                  [shared]                         |
+---------------------------------------------------+
```

## 3. Technology Choices with Rationale

### Frontend
- **Framework:** Next.js 14 App Router. *Rationale:* Built-in support for React Server Components, optimal routing, and strong performance for SaaS platforms.
- **Language:** TypeScript (strict mode). *Rationale:* Ensures type safety across the monorepo.
- **State Management:** Zustand (for persistent auth/UI state, namespaced) & TanStack Query v5 (for remote data). *Rationale:* Clean separation of remote and local state; established standard for caching and hydration.
- **Forms & UI:** React Hook Form + Zod, shadcn/ui. *Rationale:* Predictable, accessible, and easily translatable form handling (`useZodForm`).
- **i18n:** next-intl v3. *Rationale:* Deep integration with Next.js App Router for BG/EN locales.

### Backend
- **Language:** Python 3.12+. *Rationale:* First-class support for AI/ML ecosystem, native async capabilities.
- **Framework:** FastAPI with Starlette. *Rationale:* High-performance async framework, easy generation of OpenAPI specs, ideal for streaming (SSE).
- **ORM & DB:** SQLAlchemy (async), Alembic, asyncpg, PostgreSQL 16. *Rationale:* Robust relational data modeling, strong support for isolated schemas and JSONB.
- **Event Bus & Caching:** Redis 7. *Rationale:* Fast, reliable streaming via Redis Streams for microservice decoupling; supports Lua scripting for atomic rate-limiting.

### Infrastructure & Operations
- **Containerization:** Docker multi-stage builds. *Rationale:* Small image sizes (<200MB) and reproducible environments.
- **IaC & Orchestration:** Terraform >= 1.5, Helm 3. *Rationale:* Industry standard for cloud resource provisioning and Kubernetes deployment.
- **Observability:** structlog (JSON), Prometheus (`CollectorRegistry`). *Rationale:* Essential for monitoring SLA-bound AI tasks and business-critical flows.

## 4. Data Model & Isolation

The database relies on strict PostgreSQL schema isolation per service:
- **`client` schema:** Users, Companies, Memberships, Subscriptions, Opportunities, Bids.
- **`admin` schema:** Tenant configurations, Global Settings, Admin Users.
- **`pipeline` schema:** Ingested AOP/TED data, Crawl Jobs, Raw Documents.
- **`ai` schema:** KraftData mappings, Generated Proposals, Compliance Reports.
- **`notification` schema:** Email Templates, Dispatch Logs, Materialized Views for Analytics.
- **`shared` schema:** `audit_log` (append-only by all services).

*Rule:* No cross-schema joins from application code. Services must communicate via API or Redis Streams to exchange bounded context data.

## 5. API Boundaries & Integration Patterns

- **RESTful Endpoints:** Standard JSON over HTTP for CRUD.
- **SSE (Server-Sent Events):** Used for AI Generation (`run-stream`). Endpoints enforce 120s idle timeout, 600s total timeout, and 15s heartbeats to prevent proxy buffering issues. Requires `Cache-Control: no-cache` and `X-Accel-Buffering: no`.
- **Outbound HTTP Calls:** All calls to external APIs (KraftData, Stripe) must be wrapped in a two-layer resilience pattern: an outer Circuit Breaker (per-logical-name) and an inner Retry mechanism (exponential backoff).
- **Inbound Webhooks:** Must be validated using `hmac.compare_digest()` on raw request bytes to prevent timing attacks.

## 6. Deployment Topology

- **Kubernetes (K8s):** Workloads run in K8s, deployed via Helm.
- **Ingress:** Nginx Ingress routing traffic to `/api/v1/*` (Client API) and mapping subdomains for the Frontend App.
- **Admin Access:** `admin-api` is exposed internally or via strict IP allowlist/VPN access.
- **Background Workers:** Celery workers consume Redis streams for asynchronous processing (e.g., crawling, document parsing, report generation).
- **Managed Services:** Managed PostgreSQL and Redis instances for high availability and automated backups.

## 7. Key Architectural Decisions (ADRs)

### ADR-001: Strict Database Schema Isolation
**Decision:** Each microservice owns a dedicated PostgreSQL schema.
**Rationale:** Prevents tight coupling at the database layer. Enforces domain boundaries and allows services to evolve independently.

### ADR-002: Redis Streams for Async Inter-Service Communication
**Decision:** Use Redis Streams (via `eusolicit-common` EventPublisher/Consumer) for cross-service events.
**Rationale:** Provides at-least-once delivery, consumer groups, and DLQ handling without the overhead of Kafka/RabbitMQ.

### ADR-003: Dual-Layer Frontend Auth Guard
**Decision:** Protect routes using both a client-side layout `<AuthGuard>` (with a 5-second hydration timeout) and a server-side `middleware.ts` cookie check.
**Rationale:** Ensures secure, flicker-free routing while protecting against corrupt client-side storage states.

### ADR-004: Circuit Breaker and Retry for Outbound Calls
**Decision:** Wrap all outbound HTTP client calls with an outer circuit breaker and inner exponential retry.
**Rationale:** Protects internal resources from cascading failures when third-party services (e.g., VIES, KraftData) degrade.

### ADR-005: Atomic Upsert and pessimistic locking for Concurrent Tasks
**Decision:** Use `SELECT ... FOR UPDATE SKIP LOCKED` for DB-driven queues and Redis `SETNX` + DB Constraints for webhook idempotency.
**Rationale:** Prevents race conditions, double-billing via Stripe, and duplicate task execution in multi-replica deployments.

### ADR-006: Component Reusability and Query Guards
**Decision:** All UI components reside in `packages/ui` and all data fetching must be wrapped in `<QueryGuard>`.
**Rationale:** Ensures consistent loading/error states and eliminates ad-hoc spinner logic scattered across feature components.
