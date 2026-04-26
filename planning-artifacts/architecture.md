---
stepsCompleted:
  - step-01-init
  - step-02-context
  - step-03-starter
  - step-04-decisions
  - step-05-patterns
  - step-06-structure
  - step-07-validation
  - step-08-complete
inputDocuments:
  - eusolicit-docs/planning-artifacts/PRD.md
  - eusolicit-docs/planning-artifacts/epics.md
  - eusolicit-docs/planning-artifacts/ux-design-specification.md
  - eusolicit-docs/planning-artifacts/gap-coverage-matrix.md
  - eusolicit-docs/planning-artifacts/project-context.md
  - eusolicit-docs/project-context.md
workflowType: architecture
project_name: EU Solicit
user_name: Deb
date: '2026-04-25'
version: '2.1'
status: canonical
supersedes: '2.0 (2026-04-24)'
---

# Architecture Document — EU Solicit

**Author:** Deb
**Date:** 2026-04-25
**Version:** 2.1 (Canonical — supersedes v2.0 of 2026-04-24)
**Status:** Final
**Companion docs:** `PRD.md` v2.0, `ux-design-specification.md`, `epics.md`

> **Note on scope:** This document is the authoritative technical architecture for EU Solicit. It is the implementation-level counterpart to canonical PRD v2.0. Where the PRD states *what* and *why*, this document states *how*. Any divergence between the two must be resolved in favour of the PRD for product scope and in favour of this document for implementation shape.

---

## 1. System Overview

EU Solicit is a multi-tenant SaaS platform that automates the full lifecycle of public procurement and EU grant applications — from opportunity discovery through AI-guided proposal preparation, compliance validation, and post-award reporting. Initial coverage targets Bulgaria and EU-wide opportunities (AOP, TED, EU Funding Portal).

The platform is built on three architectural pillars:

1. **Decoupled Python microservices** (`client-api`, `admin-api`, `data-pipeline`, `ai-gateway`, `notification`) — each owning a dedicated PostgreSQL schema and communicating across thin HTTP boundaries or through Redis Streams.
2. **Server-rendered, strongly typed Next.js frontend** with a dual-app monorepo (`apps/client` for tenants, `apps/admin` for internal operators) sharing UI primitives, config, and generated OpenAPI types.
3. **Agentic AI execution plane** fronted by the AI Gateway, brokering SSE-streamed runs to KraftData / Sirma AI with strict resilience envelopes (circuit-breaker-wrapping-retry, idle timeouts, heartbeats, semaphore-bounded concurrency).

Strict per-schema data isolation, an immutable `shared.audit_log`, two-layer resilience on every outbound call, and revenue-critical gating at the route level are foundational — not optional — across the system.

---

## 2. Component Diagram

```text
                                +----------------------------------------+
                                |             External Systems            |
                                |                                          |
                                |  KraftData / Sirma AI  · Stripe · VIES  |
                                |  Google Calendar · Outlook · SendGrid   |
                                |  AOP · TED · EU Funding Portal · ClamAV |
                                +-----------------+------------------------+
                                                  |
                                 HTTPS / Webhooks / SSE upstream
                                                  |
 +---------------------+          +---------------v--------------+
 |  Next.js Frontend   |          |  Ingress (NGINX / ALB, TLS)  |
 |  apps/client :3000  +--REST/SSE+                              |
 |  apps/admin  :3001  |          |   /api/v1  → client-api      |
 |  (turborepo, Node)  |          |   /admin   → admin-api (VPN) |
 +----------+----------+          |   /ai      → ai-gateway (SSE)|
            |                     +---+------+-------+------+----+
        (cookies, CSRF)              |      |       |      |
            |                        |      |       |      |
            +------------------------+      |       |      |
                                            |       |      |
  +----------------+  +------------------+  |  +----v----+ |
  |  client-api    |  |  admin-api       |  |  |  ai-    | |
  |  :8001         |  |  :8002 (VPN ACL) |  |  |gateway  | |
  |  Auth · Bids   |  |  Tenants · Ops   |  |  | :8004   | |
  |  Proposals     |  |  Compliance cfg  |  |  | Agent   | |
  |  ESPD · RBAC   |  |                  |  |  |registry |
  |  Billing (Stripe)|  Bid/No-Bid audit |  |  |SSE bridge
  +-------+--------+  +---------+--------+  |  +----+----+
          |                     |           |       |
          |                     |           |       |
          +-----------+---------+           |       |
                      |                     |       |
   event publish / consume                  |       |
                      |                     |       |
         +------------v------------+        |       |
         |  Event Bus (Redis 7)    |<-------+-------+
         |  Redis Streams · Celery |
         |  Cache · Rate limit     |
         |  Quota counters (Lua)   |
         +------+-------------+----+
                |             |
   +------------v--+     +----v---------------+
   | data-pipeline |     | notification :8005 |
   | :8003         |     | Email · In-app     |
   | Celery workers|     | Slack · Teams      |
   | Crawlers      |     | Stream consumer    |
   | Matchers      |     |                    |
   +------+--------+     +----------+---------+
          |                         |
          +------------+------------+
                       |
            +----------v-----------------------------------+
            |            PostgreSQL 16 (single DB)         |
            | [client]  [admin]  [pipeline]  [gateway]     |
            | [notification]           [shared]            |
            |  · per-service role · schema-local CRUD      |
            |  · migration_role has DDL · shared read-only |
            |  · shared.audit_log (immutable)              |
            +----------------------------------------------+
                       |
            +----------v-----------------------------------+
            |       Object Storage (S3 / MinIO dev)        |
            | Uploads (100 MB/file · 500 MB/package) ·     |
            | Exports (PDF/DOCX) · Versioned artifacts     |
            | All writes pass ClamAV scan → status: clean  |
            +----------------------------------------------+
```

**Traffic planes:**

- **North–south:** Browser → Ingress → service. TLS terminated at ingress; cookies `Secure; HttpOnly; SameSite=Lax`.
- **East–west (sync):** Service-to-service over `httpx` behind the two-layer `circuit_breaker(retry(http))` envelope.
- **East–west (async):** Redis Streams events with `EventPublisher` / `EventConsumer` envelopes defined in `eusolicit-common`.
- **AI plane:** Client → `client-api` issues a run → `ai-gateway` opens SSE upstream to KraftData and re-streams tokens with heartbeats; cancellation propagates via `asyncio` task cancellation.
- **Admin plane:** `admin-api` sits behind an IP allowlist middleware (VPN-restricted) — never publicly routable.

---

## 3. Technology Choices with Rationale

### 3.1 Backend

| Choice | Version | Rationale |
|---|---|---|
| Python | 3.12+ | `type` statements, `match/case`, PEP 695 generics; async ergonomics mature enough for FastAPI at scale. |
| FastAPI + Starlette | ≥ 0.110 | Native `async`, first-class SSE, dependency-injection ergonomics for per-route tier and RBAC gating. |
| async SQLAlchemy 2.x + `asyncpg` | 2.x | 2.0-style typed queries; asyncpg is the fastest Python Postgres driver. |
| Alembic | 1.13+ | De-facto migration tool; per-service migration directories keep schema ownership clean. |
| Pydantic v2 | 2.x | Fast validation, discriminated unions, strict mode; used for DTOs and `BaseServiceSettings`. |
| `pydantic-settings` | 2.x | Each service subclasses `BaseServiceSettings` with its own `env_prefix` (e.g. `CLIENT_API_`); `get_settings()` returns a cached singleton. |
| Redis 7 + `redis-py` | 7.x | Streams, pub/sub, Lua-atomic quota counters, rate limiting, OAuth nonces, tier cache, SSE semaphore state. |
| Celery | 5.x | Scheduled (Celery Beat) + queued async work for crawlers, stuck-state reapers, email delivery. |
| structlog | 24.x | JSON logs with request-id / tenant-id / user-id correlation, configured once in `eusolicit-common`. |
| OpenTelemetry | 1.26+ | Distributed tracing across Client API → AI Gateway → KraftData for every streamed run. |
| Prometheus client | 0.20+ | `/metrics` on every service; histograms for latency, counters for errors, gauges for SSE concurrency. |
| pytest + testcontainers + `respx` | — | Real Postgres/Redis in CI; `respx` for deterministic HTTP mocking. |
| ruff + mypy | ruff latest, mypy strict | Line length 120; rules `I E W F UP`; strict type checks on services and packages. |

### 3.2 Frontend

| Choice | Version | Rationale |
|---|---|---|
| Next.js App Router | 14.x | Server Components, nested layouts, streaming SSR, middleware for auth gating. |
| TypeScript (strict) | 5.x | Type-safe boundary with backend via `openapi-typescript` codegen. |
| `openapi-typescript` | latest | Single source of truth: backend OpenAPI → frontend types. Hand-written mirrors are prohibited. |
| TanStack Query v5 | 5.x | Server-state caching, request deduplication, `<QueryGuard>` for loading/error UX. |
| Zustand | 4.x | Persistent local state; namespaced keys (`eusolicit-client-auth-store`, `eusolicit-admin-auth-store`). |
| React Hook Form + Zod | latest | `useZodForm` + `<FormField>` wrappers; Zod schemas derivable from OpenAPI. |
| shadcn/ui + Tailwind CSS | latest | Headless primitives in `packages/ui`; Tailwind slate neutral + indigo `#4f46e5` primary. |
| TipTap | 2.x | Rich-text editor for the split-pane proposal canvas; extensible for AI diff blocks, mentions, slash commands. |
| `next-intl` | 3.x | EN/BG at MVP; DE/FR/RO behind feature flag. `pnpm check:i18n` enforces string parity. |
| Turborepo + pnpm | latest | Monorepo build graph; incremental builds for CI. |
| Playwright | 1.45+ | E2E tests across Chromium/Firefox/WebKit. |

### 3.3 Infrastructure

| Choice | Rationale |
|---|---|
| Docker + Docker Compose | Local dev parity — postgres:5432, redis:6379, minio:9000, clamav:3310 match production contracts. |
| Kubernetes (target) + Helm 3 | Reproducible deployments, per-service HPA, rolling updates, network policies for admin isolation. |
| Terraform ≥ 1.5 | Cloud infra as code (VPC, managed DB, managed Redis, object storage, secrets, DNS). |
| Managed Postgres (AWS RDS / GCP Cloud SQL) | Backups, PITR, read-replica scaling; pin to Postgres 16. |
| Managed Redis (ElastiCache / MemoryStore) | Multi-AZ failover; Streams durable persistence enabled. |
| Object storage (S3 / compatible) | Presigned uploads, lifecycle rules for versioned artifacts, server-side encryption (SSE-KMS) in production. |
| ClamAV as sidecar | Every upload scanned before `status: clean`; timeouts transition to `failed` — never stuck `pending`. |
| SendGrid / SES | SMTP relay with DKIM / SPF / DMARC alignment; transactional and digest email. |

### 3.4 What we deliberately did not choose

- **Kafka / RabbitMQ** — rejected in favour of Redis Streams. Delivery semantics (at-least-once via consumer groups + ack) are sufficient for the scale envelope at GA, and it removes an operational component.
- **gRPC for east–west** — rejected. Service count is small; OpenAPI-driven REST with type generation already gives us end-to-end typing and better debuggability.
- **Shared ORM across services** — rejected. Each service owns its SQLAlchemy models; shared DTOs live in `eusolicit-models` and are intentionally thin.
- **Cross-schema joins** — forbidden by policy (see ADR-001). When a join feels natural, the answer is an API call or an event.
- **Client-only tier enforcement** — rejected. Tier gating is a revenue-critical dependency (`Depends(TierGate)`) rendered on the server path and validated in response serializers (see ADR-004).

---

## 4. System Components

### 4.1 `client-api` (port 8001) — end-user-facing backend

- **Owns schema:** `client`
- **Responsibilities:** Authentication (RS256 JWT + Google OAuth2), user & company management, proposals and content blocks, opportunity surfacing and tier-gated serialization, ESPD auto-fill, billing glue (Stripe webhooks), calendar (iCal + Google/Outlook 2-way), analytics, task/DAG workflows, RBAC enforcement.
- **Key dependencies:** Postgres (`client` + read-only `shared`), Redis (tier cache, session nonces, quota counters), AI Gateway (internal), `notification` (via events), Stripe SDK (wrapped in `asyncio.to_thread`), VIES SOAP client.
- **Critical pattern:** `check_entity_access()` dependency factory for entity-level RBAC — returns 404 (not 403) on cross-tenant access to avoid metadata leakage.

### 4.2 `admin-api` (port 8002) — VPN-restricted internal ops

- **Owns schema:** `admin`
- **Responsibilities:** Tenant management, subscription overrides, compliance framework CRUD, Bid/No-Bid override audit, system-wide observability surfaces.
- **Key dependencies:** Postgres (`admin` + read `shared`), Redis, Stripe admin endpoints.
- **Critical pattern:** IP allowlist middleware applied to every route; no fallback to public. Audit log rows always stamped with admin user and source IP.

### 4.3 `data-pipeline` (port 8003) — ingestion & matching

- **Owns schema:** `pipeline`
- **Responsibilities:** Celery workers that crawl AOP, TED, and EU Funding Portal; normalization; dedup on `(source_id, source)`; relevance matching against company profiles; `crawler_runs` lifecycle (`running → succeeded|failed`); `pipeline_health` endpoint.
- **Key dependencies:** Postgres (`pipeline`), Redis (Celery broker + result backend on isolated DB), external portal APIs through `circuit_breaker(retry(http))`.
- **Critical pattern:** Idempotent upsert; crawler runs emit freshness events; 24 h SLA breach makes `pipeline_health` return 503.

### 4.4 `ai-gateway` (port 8004) — agentic execution broker

- **Owns schema:** `gateway`
- **Responsibilities:** Agent registry (`config/agents.yaml`, logical name → external UUID), SSE bridge to KraftData, idle timeout (120 s) and heartbeat (15 s) enforcement, circuit breaker per agent, rate limiter, quota bookkeeping (refund on 5xx), semaphore-bounded concurrent streams (10/pod).
- **Key dependencies:** KraftData / Sirma AI upstream, Redis (quota Lua, semaphore state, SSE concurrency gauge), Postgres (`gateway` for run records).
- **Critical pattern:** Every stream guarantees a terminal event (`done`, `cancelled`, or `failed`); generators close in `finally` to release semaphore permits; fire-and-forget DB writes via `asyncio.create_task()` to avoid blocking the stream close path.

### 4.5 `notification` (port 8005) — delivery plane

- **Owns schema:** `notification`
- **Responsibilities:** Email (SendGrid/SES), in-app notifications, Slack/Teams webhooks, digest assembly, per-user delivery log (`sent_at`, `opportunity_ids`, channel). Consumes Redis Streams (`stream: notifications`, consumer group per service).
- **Critical pattern:** At-least-once consume + idempotent delivery dedup key; failed sends retry with bounded attempts; dead-letter stream `notifications.dlq`.

### 4.6 Shared packages (co-located, same repo, installed editable)

- `eusolicit-common` — `BaseServiceSettings`, logging, exception handlers, middleware (request-id, CORS, IP allowlist), `EventBus` with `EventPublisher` / `EventConsumer`, resilience primitives (`circuit_breaker`, `retry`, `http_factory`).
- `eusolicit-models` — Shared DTOs, `StrEnum` enums, Redis Stream event schemas.
- `eusolicit-kraftdata` — KraftData HTTP client with native SSE parser.
- `eusolicit-test-utils` — `ServiceClient`, factories (`UserFactory`, `CompanyFactory`, `OpportunityFactory`), `register_and_verify_with_role`, `create_company_pair`, RBAC helpers.

### 4.7 Frontend apps (Turborepo)

- `apps/client` (Next.js, port 3000) — tenant portal: dashboard, opportunity catalogue, proposal split-pane editor, ESPD wizard, calendar, billing self-service.
- `apps/admin` (Next.js, port 3001) — internal portal: tenant ops, compliance framework editor, system health, audit log viewer.
- `packages/ui` — shadcn/ui primitives and domain components (AI diff block, compliance metric card, quota usage widget).
- `packages/config` — shared ESLint / Prettier / tsconfig / Tailwind preset.

---

## 5. Data Model & Storage

### 5.1 Schema isolation policy

Single PostgreSQL 16 database. Six schemas, each owned by a distinct Postgres role:

| Schema | Owning service | Role permissions |
|---|---|---|
| `client` | client-api | CRUD on `client`; SELECT on `shared` |
| `admin` | admin-api | CRUD on `admin`; SELECT on `shared` |
| `pipeline` | data-pipeline | CRUD on `pipeline`; SELECT on `shared` |
| `gateway` | ai-gateway | CRUD on `gateway`; SELECT on `shared` |
| `notification` | notification | CRUD on `notification`; SELECT on `shared` |
| `shared` | migration_role (DDL) | All services have SELECT; no service has INSERT except client-api for `audit_log` (scoped rule) |

**Policy:**

- Application code must never issue a cross-schema join. Cross-domain access is an API call (sync) or an event (async).
- Only `migration_role` has DDL. Service roles cannot alter their own tables at runtime.
- `shared.audit_log` is append-only; row-level `DELETE` is denied by a RULE.

### 5.2 Core entities per schema

**`client` schema (client-api) — primary business state:**

- `users` — `id`, `email`, `password_hash` (bcrypt, max_length=128 pre-hash), `is_active`, `mfa_secret`, `google_subject`, timestamps.
- `companies` — tenant root; `vat_id`, `vat_validation_status` (`pending|verified|invalid`), `tier`, `stripe_customer_id`, white-label subdomain.
- `company_users` — membership join with `role` enum (`admin | bid_manager | contributor | reviewer | read_only`).
- `entity_permissions` — per-entity overrides for RBAC ceiling (`entity_type`, `entity_id`, `user_id`, `permission`).
- `opportunities_mirror` — projected subset of pipeline data cached per tenant for fast tier-gated serialization (refreshed via event).
- `proposals` — `id`, `company_id`, `opportunity_id`, `status`, `content_hash` (SHA-256 for optimistic locking), `version`, FTS tsvector.
- `proposal_sections` — structured sections with their own `content_hash`/`version`.
- `content_blocks` — reusable vault-backed snippets (sanitized on persist and before AI forwarding).
- `company_profile_vault` — versioned credentials, CVs, certifications, financial statements.
- `bid_history` — submitted bids with outcome, feedback, lessons learned.
- `tasks` + `task_dependencies` — DAG workflow with assignments.
- `bid_no_bid_scores` — scorecard with admin override audit pointer.
- `calendar_tokens` — Fernet-encrypted OAuth tokens for Google / Outlook.
- `espd_submissions` — structured state + generated XML references.
- `webhook_events` — idempotency table with `stripe_event_id UNIQUE` (and room for other providers).
- `subscriptions` — local mirror of Stripe subscription state.

**`admin` schema (admin-api):**

- `admin_users`, `tenant_notes`, `compliance_frameworks`, `framework_rules`, `opportunity_framework_assignments`, `bid_no_bid_overrides`.

**`pipeline` schema (data-pipeline):**

- `crawler_runs` — state machine (`running → succeeded|failed`); `last_success_at` surfaced by `pipeline_health`.
- `opportunities` — authoritative store (title, buyer, CPV codes array, country, value, currency, deadlines, source, source_id, raw payload JSONB).
- `opportunity_documents` — attachments (S3 keys, sizes, virus scan status).
- `relevance_scores` — per-company match scores with rationale.
- `recurrent_cycles` — forecast of re-publishing patterns.

**`gateway` schema (ai-gateway):**

- `agent_runs` — `id`, `agent_logical_name`, `agent_external_id`, `company_id`, `user_id`, `status` (`running|done|cancelled|failed`), `started_at`, `finished_at`, `tokens_used`, `error`.
- `agent_quota_ledger` — immutable ledger of quota debits/refunds (truth for dashboards).

**`notification` schema:**

- `notification_deliveries` — `user_id`, `channel`, `status`, `dedup_key`, `sent_at`, `payload_ref`.
- `digest_subscriptions` — cadence (daily/weekly), filters.

**`shared` schema:**

- `audit_log` — `id`, `entity_type`, `entity_id`, `action_type`, `before_json`, `after_json`, `user_id`, `company_id`, `ip_address`, `request_id`, `created_at`. Every mutating route writes a row *in the same transaction* as the mutation.
- `countries`, `cpv_codes`, `currencies`, `programmes` — reference data; read-only for services.

### 5.3 Redis usage map

| DB | Usage |
|---|---|
| 0 | Application caches (tier cache, OAuth state, rate limit counters, SSE semaphore state, quota counters via Lua) |
| 1 | Test-only; flushed by `clean_redis` fixture before/after every test |
| 2 | Celery broker |
| 3 | Celery result backend |

Redis Streams are on DB 0 under namespaced keys (`stream:notifications`, `stream:ingest.opportunities`, `stream:ai.run.completed`).

### 5.4 Object storage

S3-compatible bucket(s) with versioning enabled and server-side encryption (SSE-KMS in prod). Key schema:

```
tenants/{company_id}/uploads/{kind}/{yyyy}/{mm}/{uuid}-{filename}
tenants/{company_id}/exports/{proposal_id}/{yyyy}/{mm}/{uuid}.{pdf|docx}
tenants/{company_id}/vault/{doc_id}/v{version}/{filename}
```

Size limits: 100 MB/file, 500 MB/package. Uploads use presigned PUT; a `ClamAV` scan task transitions status `pending → clean | infected | failed`. No file is referenced as `clean` without a scan record.

---

## 6. API Boundaries

### 6.1 Browser ↔ backend

- **REST** (JSON) under `/api/v1/...`. Auth: Bearer JWT (RS256, 15 m access + 7 d refresh) in `Authorization`; refresh tokens in `Secure; HttpOnly; SameSite=Lax` cookies; CSRF tokens for cookie-authenticated endpoints. `User.is_active` checked on every authenticated path.
- **SSE** under `/api/v1/ai/run-stream/{run_id}`. Headers: `Cache-Control: no-cache`, `X-Accel-Buffering: no`, `Connection: keep-alive`. Heartbeat every 15 s; idle timeout 120 s; terminal event always emitted.
- **OpenAPI 3.1** is the source of truth. `openapi-typescript` generates the frontend `types.ts`. Manual Pydantic mirrors in frontend code are prohibited and caught by CI.
- **Error envelope:** `{ "error_code": "snake_case", "message": "human-readable", "details": { ... } }`. Standard codes per domain (`tier_exceeded`, `quota_exhausted`, `stale_content_hash`, `vat_validation_pending`, `rbac_denied`).

### 6.2 Service ↔ service (east–west)

- **Sync HTTP (`httpx`)**: every call is composed as `circuit_breaker(retry(http_factory))` (ADR-003). Timeouts are explicit on every call (no default). 4xx responses *do not* increment the breaker failure counter (client errors are not provider faults).
- **Async events (Redis Streams)**: publishers and consumers use `EventPublisher` / `EventConsumer` envelopes in `eusolicit-common`. Each event carries `event_id`, `event_type`, `schema_version`, `occurred_at`, `tenant_id`, `causation_id`, `correlation_id`. Consumer groups per service; retries with exponential backoff; dead-letter stream per channel.

Canonical streams at GA:

| Stream | Producer | Consumers | Purpose |
|---|---|---|---|
| `stream:ingest.opportunities` | data-pipeline | client-api | Refresh `opportunities_mirror`, trigger relevance scoring |
| `stream:ai.run.completed` | ai-gateway | client-api, notification | Finalize proposal generation, emit completion notification |
| `stream:billing.events` | client-api (Stripe webhook ingest) | notification, admin-api | Subscription lifecycle, tier changes |
| `stream:notifications` | any service | notification | Fan-out delivery |
| `stream:audit.mirror` | n/a (reserved) | — | Future cross-region audit replication |

### 6.3 External integrations

| External | Protocol | Notes |
|---|---|---|
| KraftData / Sirma AI | REST + SSE | Sole AI backbone at MVP. Idle timeout 120 s, heartbeat 15 s, semaphore 10 streams/pod. |
| Stripe | REST + webhooks | SDK is sync — wrapped in `asyncio.to_thread` inside `async def`. Webhooks: ECDSA/HMAC validated on raw bytes via `hmac.compare_digest()`; idempotent via `webhook_events.stripe_event_id UNIQUE`. `stripe_customer_id` provisioned as `BackgroundTask` post-201. |
| VIES (VAT) | SOAP | Fail-open to `vat_validation_status: pending` — registration never blocks on VIES availability. |
| Google Calendar / Outlook | OAuth2 + REST | Fernet-encrypted tokens; automatic refresh; permanent failure surfaces a re-authorize CTA. |
| SendGrid / SES | SMTP / REST | DKIM / SPF / DMARC alignment. Digest cadence controlled per user. |
| AOP / TED / EU Funding Portal | REST / scraping | Behind `circuit_breaker(retry(http))`; per-source scheduled job; idempotent upserts. |
| ClamAV | TCP (clamd) | Scan sidecar; timeouts mark status `failed`, never stuck `pending`. |

---

## 7. Cross-Cutting Concerns

### 7.1 AuthN / AuthZ

- **AuthN:** RS256 JWT (`eusolicit-common.jwt`). Keys rotated via k8s secret re-mount; `kid` header selects the verifying key. Google OAuth2 flow writes a `google_subject` to `users`; an unlinked provider email cannot hijack an existing account.
- **AuthZ (role ceiling):** `check_entity_access(role_required)` dependency factory for every company-scoped route. Entity-level overrides (`entity_permissions`) can grant above the role ceiling but never above `admin`.
- **Tier enforcement:** `Depends(TierGate(tier_required, feature_flag))` per route. `OpportunityFreeResponse` hard-codes the 6 permitted fields; there is no "strip fields in middleware" path.
- **Cross-tenant probe:** returns `404 Not Found` with empty body, never `403` with resource metadata.

### 7.2 Resilience

- **Two-layer envelope** (ADR-003): `circuit_breaker(retry(http_factory))` on every outbound call. Configurable per destination.
- **Consumer fail-open** for non-revenue paths: Redis outage → tier/usage gating falls back to the tenant's current tier's read paths (never blocks paid users). Hard-revenue paths (checkout, webhook processing) fail closed.
- **Stuck-state reapers:** Celery Beat jobs sweep in-progress resources (proposal generations, uploads, crawler runs) past a TTL and transition them to `failed`.
- **Streaming guarantees:** every SSE generator closes in `finally`; a terminal event (`done | cancelled | failed`) is always emitted; semaphore permits always released.

### 7.3 Observability

- **Metrics (Prometheus):** every service exposes `/metrics`. Canonical series: `tier_active_users{tier=}`, `webhook_latency_seconds{provider=,event_type=}`, `stripe_api_errors_total`, `usage_sync_drift_seconds`, `sse_concurrent_streams{agent=}`, `circuit_breaker_state{target=}`, `quota_consumed_total{tier=,feature=}`.
- **Tracing (OpenTelemetry):** spans across client-api → ai-gateway → KraftData per streamed run. Trace ID propagated on the event envelope.
- **Logging (structlog, JSON):** request-id, tenant-id, user-id correlation on every log record. No secrets, no payloads over 8 KB inline.
- **Audit (`shared.audit_log`):** every POST/PUT/PATCH/DELETE writes one row in the same transaction as the mutation. Regression test asserts row-count increment for every mutating route.

### 7.4 Security

- **Webhooks:** ECDSA/HMAC validated on raw bytes via `hmac.compare_digest()`. `==` on secrets is prohibited and enforced by grep check in CI.
- **Password policy:** `max_length=128` pre-bcrypt truncation defense; bcrypt hashing runs in executor.
- **File scanning:** ClamAV before `status: clean`; infected files quarantined under a `quarantine/` key prefix and never surfaced in UI.
- **Input sanitization:** Content blocks pass a sanitizer before persistence and again before AI forwarding (defends against prompt injection in user-authored reusable content).
- **Transport:** TLS 1.3 only at ingress; HSTS enabled; cookies `Secure`; no mixed-content.
- **At-rest encryption:** AES-256 on managed Postgres and object store; Fernet for OAuth tokens.
- **Secrets:** `.env` for local; k8s Secrets / cloud secret manager in prod; rotation is a documented operational procedure.
- **Frontend:** dual-layer guard — client-side `<AuthGuard>` blocks hydration under unauthenticated state; server-side `middleware.ts` validates the cookie before layout render.

### 7.5 Quota & billing integrity

- **Quota:** Redis Lua script performs `GET + conditional INCR + EXPIRE` atomically. Debits happen only after a 200 OK; refunds emitted on downstream 5xx. An immutable `agent_quota_ledger` is the truth source for dashboards.
- **Tier cache invalidation on subscription change:** `DELETE` the tier key — never `SET` from the webhook handler — so the next request re-reads the authoritative DB value. Webhook handlers are idempotent (`webhook_events.stripe_event_id UNIQUE`).
- **VIES fail-open:** registration path never blocks on VIES. `vat_validation_status` stays `pending` if VIES is unreachable; a background Celery task reconciles.

### 7.6 Internationalization

- **Strings:** `next-intl` with `locales/{en,bg}/*.json`. CI runs `pnpm check:i18n` — missing keys fail the build.
- **Backend error codes:** stable machine-readable; human messages localized in frontend.
- **Document parsing:** source documents in BG/EN/DE/FR/RO; output BG/EN at MVP (DE/FR/RO behind feature flag).

---

## 8. Deployment Topology

### 8.1 Environments

| Env | Purpose | Infra |
|---|---|---|
| Local | Developer laptops | `docker-compose.yml`: postgres:5432, redis:6379, minio:9000, clamav:3310, all FastAPI services hot-reload, Next.js via pnpm. |
| CI | PR verification | GitHub Actions: `testcontainers` for Postgres + Redis, Playwright with Chromium, `openapi-typescript` diff check, `pnpm check:i18n`, ruff, mypy, coverage ≥ 80%. |
| Staging | Pre-prod mirror | k8s cluster (prod mirror), seeded tenants, Stripe test mode, KraftData sandbox keys. |
| Production | GA | Multi-AZ k8s cluster (EU region), managed Postgres (primary + async read replica), managed Redis (multi-AZ), S3 with SSE-KMS, SendGrid prod. |

### 8.2 Kubernetes layout (production target)

- **Namespaces:** `eusolicit-prod`, `eusolicit-staging`, `observability`, `ingress-system`.
- **Deployments:** one per service; HPA on CPU + custom metric (SSE concurrency for `ai-gateway`, queue depth for `data-pipeline` workers).
- **Services:** ClusterIP for east–west. Only ingress (NGINX or cloud ALB) is externally routable.
- **NetworkPolicy:** `admin-api` accepts traffic only from the admin ingress, which requires IP allowlist (VPN). Egress to KraftData/Stripe/VIES allowed from designated service accounts only.
- **Secrets:** k8s Secrets synced from cloud secret manager (External Secrets Operator). Rotated by out-of-band runbooks.
- **Config:** environment-prefixed settings injected via env vars consumed by `BaseServiceSettings` subclasses.
- **Jobs & CronJobs:** Alembic migrations run as Helm pre-upgrade hook (serialized); Celery Beat as a singleton deployment; stuck-state reapers scheduled.

### 8.3 Data plane

- **Postgres:** Primary in one AZ, synchronous commit; async read replica for analytics read paths. PITR backups retained 30 days.
- **Redis:** Multi-AZ with automatic failover. Streams persisted via AOF (appendfsync everysec). `maxmemory-policy noeviction` on DB 0 so cache eviction never drops stream entries.
- **Object storage:** S3 with versioning, lifecycle rules (transition to IA at 60 days, expire `quarantine/` at 30 days), cross-region replication disabled at MVP (EU residency).

### 8.4 Routing

```
client.eusolicit.com         → NGINX ingress → apps/client (Next.js)
admin.eusolicit.com          → NGINX ingress (IP allowlist) → apps/admin
api.eusolicit.com/api/v1     → client-api
api.eusolicit.com/ai         → ai-gateway (SSE-friendly upstream config)
admin-api.internal           → admin-api (VPN only, no public DNS record)
{tenant}.eusolicit.com       → apps/client w/ tenant context (Enterprise white-label)
```

### 8.5 CI/CD

- **Pipelines:** PR → unit + integration + E2E + lint + type-check + OpenAPI diff + i18n check + coverage. Main → build images → push to registry → deploy to staging (auto) → deploy to prod (manual approval).
- **Migrations:** run before image rollout via Helm hook; rollback playbook documented per service.
- **Schema drift guard:** `make migrate-all` dry-run in CI fails the build if a service has uncommitted autogenerate.
- **Dependabot:** enabled across Python and JS ecosystems (tracked as a release gate).

---

## 9. Testing Architecture

- **Markers:** `@pytest.mark.unit` (no I/O), `@pytest.mark.integration` (Postgres + Redis), `@pytest.mark.api` (services running), `@pytest.mark.cross_service` (all services up).
- **Isolation:**
  - DB: per-test transaction rollback via `db_session` fixture. Tests *never* commit.
  - Redis: `clean_redis` fixture flushes DB 1 before/after each test; application runs on DB 0.
  - Service-level tests: override `get_db_session` and `get_redis_client` on the FastAPI app; clear `dependency_overrides` in `finally`.
- **Cross-tenant negative tests:** required for every company-scoped endpoint. Expect 404 when company A accesses company B resource.
- **ATDD:** every story has a red-phase acceptance test before implementation (including infrastructure/schema stories — no exemptions).
- **Type safety:** `openapi-typescript` runs in CI; hand-written frontend Pydantic mirrors are caught by a grep check.
- **Load baseline:** k6 suite covering SSE TTFB (< 500 ms p95), REST p95 < 200 ms at 10 K active opportunities, Redis metering under 10 K concurrent `INCR`. Required before GA.
- **TEA gate:** minimum three scored TEA reviews per epic (≥ 80/100); blocks `review → done`.
- **Coverage floor:** 80% project-wide.

---

## 10. Architectural Decisions (ADRs)

Each ADR is stated as **Decision → Context → Rationale → Consequences**.

### ADR-001: Strict schema isolation per microservice

- **Decision:** Each microservice operates within its own PostgreSQL schema with a dedicated DB role; `migration_role` is the only role with DDL; cross-schema joins in application code are forbidden.
- **Context:** Six cooperating services share one Postgres cluster; we need operational simplicity without paying the "big ball of mud" cost of a shared database.
- **Rationale:** Schema-level isolation gives us per-service ownership, independent migrations, and a clean contract for future extraction. Running one cluster is dramatically cheaper and easier to back up than per-service clusters.
- **Consequences:** Cross-domain data requires an API call or event; no easy ad-hoc joins. `shared` is deliberately minimal (reference data + audit log).

### ADR-002: Redis Streams for async event bus

- **Decision:** Inter-service async communication uses Redis Streams with consumer groups, standardized via `EventPublisher` / `EventConsumer`.
- **Context:** We need at-least-once async delivery for ingestion, notification fan-out, and billing event propagation. We already depend on Redis for caching and rate limiting.
- **Rationale:** Reusing Redis collapses one operational component. Streams provide durable, acknowledgeable delivery with consumer groups — sufficient for our scale envelope and simpler to operate than Kafka/RabbitMQ at MVP volumes.
- **Consequences:** Redis becomes a critical dependency; multi-AZ managed Redis is mandatory in prod. Re-evaluate Kafka if event volume exceeds single-Redis capacity.

### ADR-003: Two-layer resilience (`circuit_breaker(retry(http))`) on all outbound calls

- **Decision:** Every outbound HTTP call is wrapped `circuit_breaker(retry(http_factory))` with explicit timeouts; 4xx does not increment breaker failure count.
- **Context:** KraftData, Stripe, VIES, and portal APIs all fail in distinct modes: transient network blips, sustained outages, and 4xx client errors.
- **Rationale:** Retrying fixes transient faults; the circuit breaker prevents retry storms during sustained outages; excluding 4xx from breaker math prevents our own client bugs from tripping service-wide fail-open.
- **Consequences:** Every new outbound call must adopt the envelope; no raw `httpx.AsyncClient()` usage at the route level.

### ADR-004: Tier enforcement via route-level `Depends(TierGate)`, not middleware

- **Decision:** Tier enforcement and tier-specific response shaping live in per-route `Depends(TierGate)` + a tier-specific response model (e.g., `OpportunityFreeResponse`).
- **Context:** Tier gating is revenue-critical; a silent field leak to the Free tier is a serious compliance incident.
- **Rationale:** Dependency-injected gates are unit-testable, auditable per route, and tied to the exact response serializer. A global middleware path would have to strip fields on the way out and is easy to bypass accidentally with a new endpoint.
- **Consequences:** Every new revenue-gated endpoint must declare its `TierGate` explicitly; a linter/CI check flags endpoints on `/opportunities/*` without one.

### ADR-005: Dual-layer frontend auth guard

- **Decision:** Protected Next.js routes use both a client-side `<AuthGuard>` (blocks hydration) and server-side `middleware.ts` (cookie validation before layout render).
- **Context:** Either guard alone has a failure mode: client-only leaks flash-of-unauth content; server-only allows stale client state post-logout.
- **Rationale:** Belt and braces. Server stops unauthenticated responses at the network edge; client stops stale state from rendering after token revocation.
- **Consequences:** Small boilerplate on every protected route group; acceptable cost for zero flash-of-content leaks.

### ADR-006: HMAC / ECDSA webhook validation on raw bytes via `hmac.compare_digest()`

- **Decision:** All webhook signature validation reads the raw request body (not parsed JSON), uses `hmac.compare_digest()`, and is idempotent via a provider-keyed uniqueness table.
- **Context:** FastAPI's automatic JSON parsing can subtly change bytes and break signature verification. `==` comparison is vulnerable to timing attacks.
- **Rationale:** Raw-bytes + constant-time compare is the only correct path. Idempotency via `webhook_events(stripe_event_id UNIQUE)` handles replay and duplicate delivery.
- **Consequences:** Webhook routes must declare `request: Request` and read `await request.body()` before any Pydantic parsing.

### ADR-007: Optimistic locking on collaborative documents via `content_hash`

- **Decision:** Proposals and sections carry a `content_hash` (SHA-256) and `version`; mutations supply the expected hash; mismatch returns `409 Conflict` with `{ server_hash, server_version, base_version }` for the resolution dialog.
- **Context:** Multiple users edit a proposal concurrently; pessimistic locking breaks collaboration UX; last-write-wins silently loses work.
- **Rationale:** Optimistic locking is the right trade-off for human-paced editing; the structured 409 body lets the frontend render a deterministic merge UI.
- **Consequences:** Every mutation on lockable entities must accept and verify `content_hash`; tests verify two concurrent edits deterministically resolve.

### ADR-008: OpenAPI is the source of truth; frontend types are generated

- **Decision:** Backend Pydantic + FastAPI produce the canonical OpenAPI 3.1 spec; frontend runs `openapi-typescript` in CI to generate `types.ts`. Hand-maintained mirror types are prohibited.
- **Context:** Prior projects drifted when frontend and backend both hand-maintained request/response types.
- **Rationale:** Single source of truth eliminates drift and keeps refactors honest. Breaking changes surface as TypeScript errors in the frontend build.
- **Consequences:** Backend must keep OpenAPI tidy (response_model, explicit status codes); CI fails if spec and generated types diverge.

### ADR-009: Admin API VPN-restricted via IP allowlist middleware

- **Decision:** `admin-api` is served only through an ingress whose middleware enforces a corporate VPN IP allowlist; no public DNS record.
- **Context:** Admin-plane surfaces tenant data and compliance overrides — it is the highest-trust surface in the system.
- **Rationale:** Network-layer isolation in addition to AuthN/AuthZ is the right defence-in-depth for internal ops tooling.
- **Consequences:** Admin users must be on VPN; CI tests include an IP allowlist middleware suite (`test_ip_allowlist.py`).

### ADR-010: Redis Lua for atomic quota accounting

- **Decision:** Quota debits use a single Lua script implementing `GET + conditional INCR + EXPIRE`; debits only after 200 OK; refunds emitted on downstream 5xx.
- **Context:** Race conditions between concurrent AI runs caused over-consumption and quota refund loss before the Lua switch.
- **Rationale:** Lua is the only way to make the three ops atomic inside Redis. Pairing with the immutable `agent_quota_ledger` gives us an auditable truth source for dashboards.
- **Consequences:** Any new quota-gated feature must adopt the Lua primitive; no in-Python compare-and-set.

### ADR-011: SSE concurrency bounded per pod via semaphore

- **Decision:** `ai-gateway` caps concurrent SSE streams per pod (10 at MVP) using an `asyncio.Semaphore`; permit release is guaranteed in `finally`. HPA scales pods on the `sse_concurrent_streams` gauge.
- **Context:** Unbounded SSE concurrency starves the event loop and causes TTFB regressions for all streams.
- **Rationale:** A small, predictable per-pod cap gives us reliable TTFB and makes autoscaling decisions legible. Per-pod (rather than global Redis-coordinated) keeps the hot path local.
- **Consequences:** Global capacity is `pods × 10`; we observe and tune the per-pod cap as the Python interpreter's async budget evolves.

### ADR-012: Fire-and-forget stream-lifecycle DB writes via `asyncio.create_task`

- **Decision:** Writing `agent_runs` lifecycle rows (`started`, `done`, `cancelled`, `failed`) is fire-and-forget via `asyncio.create_task`, not awaited on the stream close path.
- **Context:** Awaiting DB writes on stream close causes last-token latency spikes and can block the generator's `finally`.
- **Rationale:** Lifecycle rows are observability-grade (not the authoritative ledger — that's `agent_quota_ledger`). Fire-and-forget keeps the close path tight; loss is bounded and surfaceable via alerts.
- **Consequences:** Tasks are tracked in a service-level set so they aren't GC'd mid-flight; metrics track the set cardinality.

### ADR-013: ClamAV timeouts transition to `failed`, never stuck `pending`

- **Decision:** ClamAV scan timeouts transition upload status to `failed` (user actionable) instead of leaving it `pending`.
- **Context:** We previously saw uploads stuck `pending` forever when the ClamAV sidecar timed out, silently blocking tenant workflows.
- **Rationale:** `failed` is an explicit, observable, retryable state; `pending` is a black hole. A Celery Beat reaper also sweeps anything truly stuck.
- **Consequences:** UI must render `failed` with a "Retry scan" affordance; tests simulate ClamAV timeout and assert the transition.

### ADR-014: Fernet encryption for OAuth tokens at rest

- **Decision:** Google and Outlook OAuth tokens (`calendar_tokens`) are Fernet-encrypted at rest with a rotating key pinned via `kid`.
- **Context:** OAuth tokens are bearer credentials for external user data; a DB exfiltration without encryption is a direct third-party breach.
- **Rationale:** Fernet gives us authenticated encryption with key rotation in the stdlib-adjacent `cryptography` package; integration is straightforward.
- **Consequences:** Token access goes through a `decrypt_calendar_token()` helper; key material lives in the cloud secret manager.

### ADR-015: VIES fail-open on VAT validation

- **Decision:** If VIES is unavailable, company registration continues with `vat_validation_status: pending`; a Celery task reconciles asynchronously.
- **Context:** VIES SOAP has poor uptime. Blocking registration on VIES would break a hard-revenue path every time VIES hiccups.
- **Rationale:** Registration is top-of-funnel; resilience > strictness there. Billing correctness is preserved downstream: reverse-charge applies only after `verified`.
- **Consequences:** UI communicates pending state; ops monitors reconciliation lag; `vat_validation_status` is a first-class enum, not a boolean.

### ADR-016: Stripe customer provisioning via BackgroundTask after 201 Created

- **Decision:** After user registration returns 201, a `BackgroundTask` provisions the Stripe customer; absence of `stripe_customer_id` returns a structured `422` on billing routes until provisioned.
- **Context:** Blocking registration on Stripe API latency degrades signup UX and couples a revenue dependency into the auth path.
- **Rationale:** Background provisioning keeps registration fast. Gate billing routes explicitly with a typed error so the UI knows to poll / retry.
- **Consequences:** Frontend billing pages handle the `stripe_customer_pending` error code gracefully; tests cover the race.

### ADR-017: i18n string parity enforced in CI

- **Decision:** `pnpm check:i18n` in CI fails if any key is missing across supported locales.
- **Context:** Silent English fallbacks shipped to BG tenants in an earlier epic.
- **Rationale:** String parity is a product-quality gate, not a dev-time suggestion; CI is the only reliable enforcement point.
- **Consequences:** Adding a string means adding all locale entries in the same PR.

### ADR-018: PostgreSQL FTS (not external search engine) at MVP

- **Decision:** Opportunity search uses Postgres FTS with `tsvector` and a GIN index; no Elasticsearch / OpenSearch at MVP.
- **Context:** Catalogue scale at GA is 10 K active opportunities. Sub-second filtered full-text at that scale is well within Postgres's envelope.
- **Rationale:** One fewer moving piece; tight transactional guarantees with the rest of the schema.
- **Consequences:** Re-evaluate when active catalogue exceeds ~100 K or facets become too expensive; migration path is well understood (denormalize to Elasticsearch with a projection consumer on `stream:ingest.opportunities`).

### ADR-019: No electronic bid submission at MVP

- **Decision:** Users submit on their respective portals (CAIS, TED eSender, etc.) themselves. The platform provides ESPD auto-fill and exportable drafts but does not transact on portals.
- **Context:** Portal-side submission integrations are legally and operationally expensive; scoping them in would delay MVP.
- **Rationale:** ESPD auto-fill captures the bulk of the pain. Submission automation is a post-MVP differentiator, not a blocker.
- **Consequences:** UX clearly guides the user to the portal with a deep link; product roadmap tracks submission automation as a follow-on epic.

### ADR-020: EU-only hosting at MVP; no data residency toggles

- **Decision:** All production workloads run in an EU region (primary Frankfurt or equivalent). No tenant-level data-residency toggles at MVP.
- **Context:** GDPR and customer expectations require EU hosting; per-tenant residency is operationally heavy and not in scope.
- **Rationale:** EU-only meets the compliance bar and keeps operations simple.
- **Consequences:** Non-EU tenants still land in the EU region; residency options tracked post-MVP for Enterprise customers.

---

## 11. Non-Functional Targets (restated for traceability)

| Target | Value | Source |
|---|---|---|
| API p95 latency | < 200 ms | NFR2 |
| SSE first-token p95 | < 500 ms | NFR3 |
| Availability (paid) | 99.5% monthly | NFR1 |
| Catalogue scale | 10 K active opportunities, 1 K concurrent sessions | NFR2 |
| Ingestion freshness | p95 < 24 h source → listing | PRD §2 |
| Coverage floor | 80% project-wide | §8.6 |
| SSE per-pod cap | 10 | ADR-011 |
| Upload cap | 100 MB/file, 500 MB/package | FR7 |
| JWT lifetimes | 15 m access / 7 d refresh | NFR7 |

---

## 12. Risks & Open Architectural Questions

| # | Topic | Status | Next step |
|---|---|---|---|
| AR-1 | k6 baseline not yet executed & committed | Carry-forward | Gate Epic 12 on k6 scripts written, executed, committed. |
| AR-2 | Dependabot not yet configured | Carry-forward | Single `.github/dependabot.yml` merged before Epic 11. |
| AR-3 | Billing-path Prometheus metrics partial | Partial | Inject 5 billing metrics into S12.17 before GA. |
| AR-4 | Prompt-injection sanitization on content blocks | Deferred | Sprint 8 story: sanitize on persist *and* before AI forwarding. |
| AR-5 | PostgreSQL FTS vs. external search beyond 100 K opps | Monitor | Re-evaluate at catalogue inflection; migration path via stream consumer. |
| AR-6 | Single-Redis SPOF for streams + cache | Monitor | Multi-AZ managed Redis is mandatory in prod; re-evaluate splitting cache vs. streams at scale. |
| AR-7 | Multi-provider AI routing | Post-MVP | KraftData sole backbone at MVP; abstraction already lives in `ai-gateway`. |
| AR-8 | SSO (SAML) for Enterprise | Post-MVP | Fast-follow after MVP. |
| AR-9 | Data residency toggles beyond EU | Post-MVP | EU-only at MVP; tracked for Enterprise. |

---

## 13. Alignment with PRD v2.0

This architecture satisfies all twelve release-level acceptance criteria in PRD §10:

1. Tier enforcement — §7.1 + ADR-004.
2. Data pipeline — §4.3 + `stream:ingest.opportunities`.
3. AI draft generation — §4.4 + ADR-011/012.
4. Streaming reliability — §7.2.
5. Security — §7.4 + ADR-006/013/014.
6. Calendar — §4.1 + ADR-014.
7. Billing correctness — §7.5 + ADR-006/015/016.
8. Observability — §7.3 + Prometheus / OTel / `shared.audit_log`.
9. RBAC isolation — §7.1 (404, not 403).
10. i18n parity — §7.6 + ADR-017.
11. Type safety — §6.1 + ADR-008.
12. Test gates — §9.

---

*End of Architecture Document v2.1 (2026-04-25). Supersedes v2.0 (2026-04-24). For product scope see `PRD.md` v2.0; for implementable stories see `epics.md`.*
