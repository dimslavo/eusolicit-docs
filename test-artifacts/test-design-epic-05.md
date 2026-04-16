---
stepsCompleted: ['step-01-detect-mode', 'step-02-load-context', 'step-03-risk-and-testability', 'step-04-coverage-plan', 'step-05-generate-output']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-15'
workflowType: 'testarch-test-design'
mode: 'epic-level'
epicNumber: 5
inputDocuments:
  - 'eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md'
  - 'eusolicit-docs/test-artifacts/test-design-architecture.md'
  - 'eusolicit-docs/test-artifacts/test-design-qa.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-04.md'
  - 'eusolicit-docs/test-artifacts/test-design-progress.md'
  - 'eusolicit/_bmad/bmm/config.yaml'
---

# Test Design: Epic 5 — Data Pipeline & Opportunity Ingestion

**Date:** 2026-04-15
**Author:** TEA Master Test Architect
**Status:** Draft
**Epic:** E05 | **Sprint:** 3–4 | **Points:** 34 | **Dependencies:** E01, E04

---

## Executive Summary

**Scope:** Epic-level test design for E05 — Data Pipeline & Opportunity Ingestion. Covers the full lifecycle of the Celery-based asynchronous pipeline that orchestrates scheduled crawling of three public procurement portals (AOP, TED, EU Grants), normalizes and deduplicates opportunity records via AI Gateway agents, scores each opportunity against stored company profiles, generates portal-specific submission guides, and publishes `opportunities.ingested` / `opportunities.expired` events to Redis Streams. Functional scope spans: pipeline schema migration (S05.01), Celery Beat bootstrap (S05.02), AI Gateway client with retry/circuit-breaker (S05.03), three crawler tasks (S05.04–S05.06), relevance scoring chain (S05.07), submission guide generation (S05.08), Redis event publishing (S05.09), cleanup task (S05.10), enrichment queue worker (S05.11), and E2E integration tests + observability (S05.12).

This epic is the **primary data source for every tenant-facing feature**: opportunity listings, proposal scoring, guide generation, and analytics all originate here. Silent failures (missed events, incomplete deduplication, broken scoring chains) propagate invisibly through downstream services.

**Risk Summary:**

- Total risks identified: 9
- High-priority risks (≥6): 3 (dedup race condition, AI Gateway cascade failure, Redis Stream publish loss)
- Critical categories: DATA (dedup race, Redis loss), TECH (AI Gateway cascade), PERF (chord starvation)

**Coverage Summary:**

- P0 scenarios: 9 (~18–30 hours)
- P1 scenarios: 30 (~25–40 hours)
- P2 scenarios: 16 (~10–18 hours)
- P3 scenarios: 5 (~3–6 hours)
- **Total effort:** ~56–94 hours (~1.5–2.5 weeks, 1 QA)

---

## Not in Scope

| Item | Reasoning | Mitigation |
|------|-----------|------------|
| **Direct testing of KraftData agents** | External dependency; credentials-gated; not controlled by EU Solicit | Use `respx`/`pytest-mock` for all outbound AI Gateway calls in CI; real-credential smoke tests tagged `@skip-ci` |
| **AI Gateway (E04) internal logic** | E04 owns its own circuit breaker, SSE proxy, and webhook receiver; E05 only tests the `ai_gateway_client` module it wraps | Mock `ai_gateway_client` at the module boundary for crawler task tests; E04 integration tested separately |
| **Redis Streams consumer validation** | Downstream epics own consumer logic; E05 only owns the publish side | Verify `XADD` is called with correct stream name and fully-formed payload; consumer behavior out-of-scope |
| **Prometheus scrape infrastructure / Alertmanager** | Platform/DevOps concern; E05 verifies only that metrics are exported at `/metrics` | Verify endpoint returns 200 with expected metric names and labels; scrape config is a deployment artifact |
| **Kubernetes Beat scheduling accuracy** | Infra-level cron fidelity is a deployment concern | Unit-test cron expressions; confirm `CELERY_TIMEZONE=UTC`; Beat accuracy at cluster level out-of-scope |
| **LLM semantic quality of submission guides** | EU Solicit proxies KraftData output; content quality is a KraftData responsibility | Structural shape of `steps` JSONB is validated; semantic correctness is a UAT/product concern |
| **Frontend opportunity display** | E05 is a backend service; no direct UI integration in this epic | Opportunity listing UI is owned by a later epic and tested end-to-end there |

---

## Risk Assessment

### High-Priority Risks (Score ≥6)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner | Timeline |
|---------|----------|-------------|-------------|--------|-------|------------|-------|----------|
| **E05-R-001** | **DATA** | Deduplication race condition — AOP, TED, and EU Grants crawlers may run overlapping Beat-triggered windows and attempt concurrent `INSERT ... ON CONFLICT` upserts for the same `(source_id, source_type)` row; if the upsert is not implemented as a single atomic PostgreSQL statement, concurrent transactions can violate the unique constraint or, worse, silently insert a duplicate before the constraint check fires; double-rows produce double-scoring entries, inflated `crawler_runs` counts, and duplicated submission guides | 2 | 3 | **6** | Implement upsert as a single `INSERT ... ON CONFLICT (source_id, source_type) DO UPDATE SET ...` SQL statement (never application-level check-then-insert); add DB-level unique constraint (S05.01 AC); integration test: run two concurrent upsert workers with identical payload, verify single row in DB | Backend / QA | Sprint 3 |
| **E05-R-002** | **TECH** | AI Gateway cascade failure leaving orphaned crawler_runs — all three crawler tasks plus scoring and guide generation chain through AI Gateway (E04); if AI Gateway is unavailable mid-cycle, Celery retries via `autoretry_for=(AIGatewayUnavailableError,)` with `max_retries=3`; however, if the process restarts between retry attempts, the `crawler_runs` record remains in `running` or `retrying` state permanently with no recovery path; downstream scoring and event-publish steps are never triggered, causing silent data gaps | 2 | 3 | **6** | Ensure crawler task `on_failure` handler always transitions `crawler_runs` to `failed` status with error message regardless of how the task terminates; add periodic `cleanup_stale_runs` health check (or incorporate into S05.11) that marks runs stuck in `running/retrying` for >2× the expected crawl window; integration test: mock AI Gateway returning 5xx, verify `crawler_runs.status = failed` after max_retries | Backend / QA | Sprint 3–4 |
| **E05-R-003** | **DATA** | Redis Streams publish-or-lose gap — `opportunities.ingested` event is published after the full crawl-normalize-score-guide chain completes (S05.09); if `XADD` fails (Redis unavailable, network partition), S05.09 specifies isolated retry of the publish step only; however, if all retries are exhausted, the data is persisted in PostgreSQL but the downstream event is permanently lost — consumers never learn about the ingested opportunities, breaking any real-time feature that depends on the stream | 2 | 3 | **6** | Wrap `XADD` in try/except with ERROR-level structured log containing `run_id` and `opportunity_ids` list; after exhausted retries, write a `stream_publish_failures` entry (or enrich `crawler_runs.errors` JSONB) for compensating recovery; add observability criteria for alerting on missed stream events; acceptance test: mock Redis failure → crawl data persisted, ERROR logged, no crawl re-execution | Backend / QA | Sprint 3 |

### Medium-Priority Risks (Score 3–5)

| Risk ID | Category | Description | Probability | Impact | Score | Mitigation | Owner |
|---------|----------|-------------|-------------|--------|-------|------------|-------|
| E05-R-004 | TECH | TED pagination incomplete consumption — TED crawler loops via `next_page_token` sentinel; if the agent returns an inconsistent token type (null vs. empty string vs. missing key) for the final page, the loop exits prematurely; partial data is stored with correct-looking `crawler_runs` counts (only the fetched pages), silently omitting the remainder | 2 | 2 | 4 | Normalise sentinel check: treat `null`, `""`, and missing key as end-of-pages; unit test with 3-page mock (tokens: "page2", "page3", null); verify `crawler_runs` totals accumulate across all pages | Backend / QA |
| E05-R-005 | PERF | Celery chord starvation — `score_opportunities` uses `chord`/`group` with default concurrency 5; a large crawl batch (200+ opportunities) may saturate worker slots for the duration of the scoring chord, blocking Celery Beat from dispatching the next scheduled crawl; no priority queue exists between crawl tasks and scoring tasks | 2 | 2 | 4 | Route scoring tasks to a dedicated `pipeline_scoring` queue separate from `pipeline_crawl`; document queue routing in deployment config; P2 test: verify crawl task acquires worker while scoring chord is active on the default queue | Backend |
| E05-R-006 | SEC | Soft-delete filter bypass — `deleted_at IS NULL` must be enforced as a SQLAlchemy model-level default criterion (not only at application call sites); raw queries, admin tools, and any new code that bypasses the application layer accidentally return expired opportunities; deleted opportunities re-scored or re-guided waste AI Gateway quota and produce misleading data | 2 | 2 | 4 | Implement `deleted_at IS NULL` as a `__mapper_args__` default criterion on the `Opportunities` model; unit test: insert 1 deleted + 1 active opportunity, assert model query returns exactly 1; code review gate: any raw SQL touching `pipeline.opportunities` must explicitly opt-in to include soft-deleted rows | Backend / QA |
| E05-R-007 | DATA | Enrichment queue unbounded failure accumulation — failed scoring and guide items enter `enrichment_queue`; after 3 failed retries the item is marked `failed` and `enrichment.failed` event emitted; if the event emitter also fails (Redis down) or no consumer is wired to act on the alert, permanently failed items accumulate silently; no TTL, archival, or manual recovery path is defined | 2 | 2 | 4 | Emit `enrichment.failed` at best-effort with fallback ERROR log containing item ID and `enrichment_type`; document a manual recovery runbook (requeue failed items via admin task); P1 test: simulate 3 consecutive failures → item marked `failed`, event emitted or error logged; add `enrichment_queue` depth to Prometheus gauge (already in S05.12) | Backend / QA |

### Low-Priority Risks (Score 1–2)

| Risk ID | Category | Description | Probability | Impact | Score | Action |
|---------|----------|-------------|-------------|--------|-------|--------|
| E05-R-008 | OPS | Celery Beat UTC/DST misconfiguration — `crawl-eu-grants` (02:00 UTC) and `cleanup-expired-opportunities` (Sunday 04:00 UTC) use cron expressions; if `CELERY_TIMEZONE` defaults to server local time, DST transitions shift execution windows 1 hour; in CI, incorrect timezone produces false assertion failures in Beat schedule tests | 1 | 2 | 2 | Set `CELERY_TIMEZONE = UTC` explicitly in Celery config; assert timezone in P3 Beat schedule unit test; document in deployment runbook |
| E05-R-009 | OPS | testcontainers startup latency in CI — S05.12 integration tests require PostgreSQL + Redis containers; 30–60s cold-start adds to CI time; combined with E04 testcontainers (same sprint), CI job may approach the 10-minute limit | 2 | 1 | 2 | Share testcontainers session scope across test modules; use Docker layer caching; enforce `test execution < 60s` AC from S05.12 with `@pytest.mark.timeout(60)` on the integration suite |

### Risk Category Legend

- **TECH**: Technical/Architecture (flaws, integration, scalability)
- **SEC**: Security (access controls, auth, data exposure)
- **PERF**: Performance (SLA violations, degradation, resource limits)
- **DATA**: Data Integrity (loss, corruption, inconsistency)
- **BUS**: Business Impact (UX harm, logic errors, revenue)
- **OPS**: Operations (deployment, config, monitoring)

### System-Level Risk Inheritance

- **E05-R-001** (dedup race) and **E05-R-003** (Redis publish loss) → both extend system **R-05** (Tender Sync Failure, score 6). The system-level risk is about sync reliability in general; E05 adds two concrete failure modes: concurrent write race and event bus delivery gap.
- **E05-R-006** (soft-delete bypass) → new epic-level risk on data visibility; raises the same tenant data leakage concern as system **R-01** (Multi-tenancy Isolation) if a future multi-tenant indexing query accidentally returns other tenants' expired opportunities. Flag for architecture review if multi-tenancy is extended to the pipeline schema.

---

## Entry Criteria

- [ ] E01 complete: monorepo scaffold operational; `services/pipeline/` directory structure exists; `pyproject.toml` with Celery, SQLAlchemy, alembic, pytest-celery, respx, testcontainers dependencies present
- [ ] E04 complete: AI Gateway running in Docker Compose; `ai_gateway_client` module importable with configured base URL and auth headers
- [ ] PostgreSQL and Redis available in CI (Docker Compose services or testcontainers session)
- [ ] `respx` mock router configured for all outbound AI Gateway HTTP calls (no real KraftData credentials required for CI); at least one mock fixture set per crawler type (AOP, TED, EU Grants)
- [ ] Alembic migration chain intact from E01 baseline; `alembic upgrade head` successfully creates `pipeline` schema tables
- [ ] `eusolicit-common` shared library available for structured JSON logging and Redis Streams event bus abstraction
- [ ] At least 5 active company profiles seeded in DB for relevance scoring fixture

## Exit Criteria

- [ ] All P0 tests passing (100%)
- [ ] All P1 tests passing (≥95%; any failures triaged and accepted with PM sign-off)
- [ ] No open high-severity bugs in deduplication logic, AI Gateway cascade handling, or Redis Stream publishing
- [ ] Line coverage ≥80% on `services/pipeline/tasks/` and `services/pipeline/models/`
- [ ] Integration test suite (S05.12) passes in under 60 seconds in CI
- [ ] All 4 Prometheus metrics (`pipeline_crawl_duration_seconds`, `pipeline_opportunities_total`, `pipeline_enrichment_queue_depth`, `pipeline_agent_call_duration_seconds`) exported and scrapeable at `/metrics`
- [ ] `crawler_runs` table correctly records start/end times, status, and found/new/updated/error counts for all three crawler types

---

## Test Coverage Plan

> **Note:** P0/P1/P2/P3 denote **priority and risk level**, NOT execution timing. Execution scheduling is defined separately in the Execution Strategy section.

### P0 (Critical)

**Criteria:** Blocks core pipeline data flow + High risk (≥6) + No workaround for downstream features

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E05-P0-001 | Upsert deduplication — same data submitted twice produces zero new inserts, only updates where data changed | S05.01, S05.04 | Integration | E05-R-001 | Run crawl twice with identical mocked response; assert `COUNT(*)` unchanged |
| E05-P0-002 | Concurrent upsert safety — two workers upsert same `(source_id, source_type)` simultaneously; verify single row in DB | S05.01, S05.04 | Integration | E05-R-001 | Use `threading.Thread` or `asyncio.gather` to fire two upserts in parallel |
| E05-P0-003 | AOP crawl happy path — mock AI Gateway returns 10 opportunities; verify `crawler_runs` counts (found=10, new=10, updated=0), all records upserted in `pipeline.opportunities` | S05.04 | Integration | E05-R-002 | Uses respx mock for AOP agent + normalization agent calls |
| E05-P0-004 | AOP crawl AI Gateway down — gateway returns 5xx on all retries; verify Celery retries triggered (max 3), `crawler_runs.status = failed`, error message populated | S05.04 | Integration | E05-R-002 | Freeze `AIGatewayUnavailableError` after first call; assert no partial data committed |
| E05-P0-005 | TED full pagination — mock AI Gateway returns 3 pages (tokens: "p2", "p3", null); verify all pages consumed, `crawler_runs` totals = sum across all pages | S05.05 | Integration | E05-R-004 | Also validates E05-R-004 mitigation (sentinel normalization) |
| E05-P0-006 | EU Grants field completeness — mock agent response includes `evaluation_criteria` JSONB and `mandatory_documents` list; verify both fields populated correctly in `pipeline.opportunities` | S05.06 | Integration | E05-R-001 | Validate JSONB structure shape, not semantic content |
| E05-P0-007 | Redis Streams event published — after successful AOP crawl cycle, verify `opportunities.ingested` event on correct stream with `crawler_type`, `run_id`, `opportunity_ids`, `timestamp`, `summary` fields | S05.09 | Integration | E05-R-003 | Use Redis test client to read stream after task completes |
| E05-P0-008 | Redis Streams publish isolated retry — mock `XADD` failure; verify opportunity data persisted in DB, only publish step retried (crawl not re-executed), ERROR logged with `run_id` | S05.09 | Integration | E05-R-003 | Assert `crawler_runs` is already `completed` before publish retry attempts |
| E05-P0-009 | End-to-end pipeline flow — full crawl → normalize → score → guide → publish chain with mocked AI Gateway responses for one crawler type; verify all downstream records created and event published | S05.12 | Integration | E05-R-002 | Covers acceptance criterion: "integration tests cover the full crawl-to-event flow" |

**Total P0: 9 tests, ~18–30 hours**

---

### P1 (High)

**Criteria:** Important feature correctness + Medium risk (3–5) + Common execution paths

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E05-P1-001 | Schema migration up — `alembic upgrade head` creates all 4 tables in `pipeline` schema | S05.01 | Integration | — | Run against testcontainers PostgreSQL |
| E05-P1-002 | Unique constraint enforced — attempt direct DB insert of duplicate `(source_id, source_type)`, verify `IntegrityError` raised | S05.01 | Unit | E05-R-001 | Tests DB-level enforcement independent of application upsert |
| E05-P1-003 | SQLAlchemy CRUD round-trip — create, read, update, delete on `Opportunity` model; assert column types (text[], JSONB, timestamps) | S05.01 | Unit | — | Includes `cpv_codes`, `evaluation_criteria`, `relevance_scores` JSONB |
| E05-P1-004 | Soft-delete default filter — insert 1 deleted opportunity (`deleted_at` set) + 1 active; assert model query returns exactly 1 | S05.01 | Unit | E05-R-006 | Tests `__mapper_args__` default criterion |
| E05-P1-005 | Beat schedule lists all 4 periodic tasks — `celery inspect scheduled` equivalent returns `crawl-aop`, `crawl-ted`, `crawl-eu-grants`, `cleanup-expired-opportunities` | S05.02 | Unit | — | Use `app.conf.beat_schedule` dict assertion |
| E05-P1-006 | Schedule intervals overridable via env vars — patch env vars for AOP interval to 1h, verify Beat schedule reflects override without code change | S05.02 | Unit | — | Important for staging/dev environment flexibility |
| E05-P1-007 | Health-check task returns OK — invoke `pipeline.health` task synchronously, assert return value is `"OK"` or equivalent truthy | S05.02 | Unit | — | Container liveness probe dependency |
| E05-P1-008 | AI Gateway client retry on 5xx — configure client with max_retries=3; mock 2 failures then success; verify 3 total calls made with exponential back-off delays | S05.03 | Unit | E05-R-002 | Use `freezegun` or mock `asyncio.sleep` to assert back-off timing |
| E05-P1-009 | Circuit breaker state transitions — open after 5 consecutive failures; subsequent calls rejected immediately (no HTTP call made); half-open after 60s (next call attempted) | S05.03 | Unit | E05-R-002 | Mock clock for 60s elapsed; verify `OPEN → HALF_OPEN → CLOSED` |
| E05-P1-010 | `AIGatewayUnavailableError` raised after retries exhausted — mock 3 consecutive 5xx; assert correct exception type raised; verify Celery sees it as retriable | S05.03 | Unit | E05-R-002 | Test both `max_retries` and exception propagation |
| E05-P1-011 | Correlation ID header in all outbound calls — verify every request via `ai_gateway_client` includes `X-Correlation-ID` or equivalent header | S05.03 | Unit | — | Inspect `respx` request object in mock assertions |
| E05-P1-012 | AOP `crawler_runs` accounting — verify record created with `status=running` at start; updated with `status=completed`, accurate `found`/`new`/`updated` counts and timestamps at end | S05.04 | Integration | — | Covers AC: "crawler_runs records start/end times, status, counts" |
| E05-P1-013 | AOP unrecoverable failure marks run as `failed` — mock non-retryable exception; verify `crawler_runs.status = failed` with `error_message` set | S05.04 | Integration | E05-R-002 | Distinguishes from retrying state |
| E05-P1-014 | TED NUTS code mapping — mock TED response with NUTS code `RO321`; verify `region` and `country` fields populated on opportunity record | S05.05 | Unit | — | Test NUTS lookup table / mapping function directly |
| E05-P1-015 | TED crawler_runs accumulate totals across pages — mock 2-page response (5 opps page 1, 3 opps page 2); verify `crawler_runs.found = 8`, not 3 (last page only) | S05.05 | Integration | E05-R-004 | Direct test of accumulation logic |
| E05-P1-016 | EU Grants budget range parsing — mock response with single `budget_value = 100000`; verify `budget_min = budget_max = 100000`; mock range `"50000-200000"` → correct split | S05.06 | Unit | — | Two sub-cases in one parameterized test |
| E05-P1-017 | EU Grants `opportunity_type = grant` — verify all EU Grant opportunities stored with correct type field | S05.06 | Integration | — | Ensures portal-specific type is not overwritten by normalization |
| E05-P1-018 | Relevance scoring parallelism — mock 10 opportunities, concurrency=5; verify scoring calls dispatched as group/chord, not sequential; verify `relevance_scores` JSONB populated for all 10 | S05.07 | Integration | E05-R-005 | Use Celery eager mode or task signature inspection |
| E05-P1-019 | Single-opportunity scoring failure does not block batch — mock failure for opportunities 3 and 7 of 10; verify remaining 8 scored and stored; batch result not failed | S05.07 | Integration | E05-R-007 | Core resilience requirement |
| E05-P1-020 | Failed scoring queued in enrichment_queue — mock scoring failure; verify `enrichment_queue` entry created with `enrichment_type = relevance_scoring` and failed opportunity ID | S05.07 | Integration | E05-R-007 | Verify queue entry, not retry behavior (tested in S05.11) |
| E05-P1-021 | Submission guide created for new opportunity — mock Submission Guide Agent response with structured `steps`; verify `pipeline.submission_guides` row created, `reviewed=false`, `steps` JSONB populated | S05.08 | Integration | — | Core guide generation path |
| E05-P1-022 | Existing guide not regenerated unless opportunity updated — run guide generation twice with unchanged opportunity; verify only 1 guide row and only 1 AI Gateway call | S05.08 | Integration | — | Tests `updated_at` vs guide `created_at` comparison |
| E05-P1-023 | Guide generation failure queues in enrichment_queue with correct type — mock agent failure; verify `enrichment_queue` entry has `enrichment_type = submission_guide` | S05.08 | Integration | E05-R-007 | Symmetric with P1-020 |
| E05-P1-024 | Event payload `summary` matches `crawler_runs` counts — assert `summary.new + summary.updated + summary.unchanged = crawler_runs.found` | S05.09 | Integration | E05-R-003 | Cross-validates two data sources |
| E05-P1-025 | Cleanup soft-deletes opportunities past deadline + retention — insert 3 opportunities: 1 within retention window, 2 past it; run cleanup task; verify `deleted_at` set on the 2 expired | S05.10 | Integration | — | Core cleanup behavior |
| E05-P1-026 | Cascade soft-delete to submission_guides — run cleanup on opportunity with guide; verify both opportunity and guide have `deleted_at` set | S05.10 | Integration | — | Prevents orphaned guide records |
| E05-P1-027 | `opportunities.expired` event published with affected IDs — run cleanup; verify event on Redis Stream contains list of expired opportunity IDs | S05.10 | Integration | E05-R-003 | At-least-once semantics same as ingestion event |
| E05-P1-028 | Enrichment queue FIFO processing — insert 5 items at different timestamps; verify processed in insertion order with configurable batch size | S05.11 | Integration | E05-R-007 | FIFO contract |
| E05-P1-029 | Item marked `failed` after 3 attempts — mock 3 consecutive failures for one enrichment item; verify `status = failed`, `enrichment.failed` event emitted (or ERROR logged) | S05.11 | Integration | E05-R-007 | Covers "fails 3 times" AC from S05.11 |
| E05-P1-030 | Successful enrichment retry updates opportunity + deletes queue item — mock successful retry on 2nd attempt; verify opportunity score updated, queue item deleted | S05.11 | Integration | E05-R-007 | Covers "item succeeds on retry" AC from S05.11 |

**Total P1: 30 tests, ~25–40 hours**

---

### P2 (Medium)

**Criteria:** Secondary flows + Low risk (1–2) + Edge cases and configuration validation

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E05-P2-001 | `alembic downgrade -1` drops all 4 tables cleanly | S05.01 | Integration | — | Verify no schema artifacts remain |
| E05-P2-002 | Indexes present — `deadline`, `status`, `crawler_type` indexes exist on `pipeline.opportunities` | S05.01 | Unit | — | Use `sqlalchemy.inspect` to assert index names |
| E05-P2-003 | Celery worker and Beat start cleanly in Docker Compose | S05.02 | Integration | E05-R-009 | Health-check ping; not a full functional test |
| E05-P2-004 | AI Gateway client timeout configurable per agent type — set different timeouts for crawler vs. scoring agents; verify requests use correct timeout values | S05.03 | Unit | — | Test client configuration layer |
| E05-P2-005 | AOP crawl `retrying` status reflected in `crawler_runs` mid-retry — verify status transitions `running → retrying` on first retry attempt | S05.04 | Integration | E05-R-002 | Tests status granularity, not just final state |
| E05-P2-006 | TED partial page failure handled gracefully — mock page 2 returning 5xx; verify task retries and does not store partial data | S05.05 | Integration | E05-R-004 | Pagination resilience edge case |
| E05-P2-007 | EU Grants `crawler_runs` record accuracy — verify record reflects full grant crawl cycle with correct source_type label | S05.06 | Integration | — | Parity check with AOP/TED accounting |
| E05-P2-008 | Relevance scores keyed by company_id — verify `relevance_scores` JSONB structure is `{company_id: score, ...}` not a flat list | S05.07 | Unit | — | JSON schema shape test |
| E05-P2-009 | Partial failure batch: 3 of 5 succeed, 2 fail to enrichment_queue — mock failures on IDs 2 and 4; assert scored_count=3, enrichment_queue entries=2 | S05.07 | Integration | E05-R-007 | Explicit AC from S05.07 |
| E05-P2-010 | Submission guide `generated_by` records agent version — verify `submission_guides.generated_by` is populated from AI Gateway response metadata | S05.08 | Unit | — | Traceability field |
| E05-P2-011 | Retention period configurable via env var — patch `OPPORTUNITY_RETENTION_DAYS=7`; run cleanup; verify 7-day window used instead of 30-day default | S05.10 | Unit | — | Environment variable override |
| E05-P2-012 | Default filter excludes soft-deleted rows from model queries — insert 2 soft-deleted + 1 active; assert all model-level queries return 1 (not 3) | S05.10 | Integration | E05-R-006 | Tests query-time enforcement after cleanup |
| E05-P2-013 | `ENRICHMENT_BATCH_SIZE` env var controls batch size — patch to 3; insert 7 pending items; verify only 3 processed per task invocation | S05.11 | Unit | — | Configuration contract |
| E05-P2-014 | Dedup second-run produces zero new inserts — run same mock payload twice; assert `crawler_runs.new = 0` on second run; check `updated` count for changed fields | S05.12 | Integration | E05-R-001 | Explicit AC from S05.12: "second run with same data produces zero new inserts" |
| E05-P2-015 | All 4 Prometheus metrics exported at `/metrics` — hit metrics endpoint; assert `pipeline_crawl_duration_seconds`, `pipeline_opportunities_total`, `pipeline_enrichment_queue_depth`, `pipeline_agent_call_duration_seconds` present with expected label dimensions | S05.12 | Integration | — | Structural metric validation, not value thresholds |
| E05-P2-016 | Structured log lines include required fields — trigger a crawl task with test logger; assert each log record contains `correlation_id`, `task_name`, and `crawler_type` keys | S05.12 | Integration | — | Log schema compliance |

**Total P2: 16 tests, ~10–18 hours**

---

### P3 (Low)

**Criteria:** Nice-to-have + Exploratory + Benchmarks and configuration correctness

| Test ID | Requirement / Scenario | Story | Test Level | Risk Link | Notes |
|---------|------------------------|-------|------------|-----------|-------|
| E05-P3-001 | Celery Beat UTC timezone assertion — verify `app.conf.timezone == "UTC"` and EU Grants cron `"0 2 * * *"` resolves to 02:00 UTC | S05.02 | Unit | E05-R-008 | Prevents DST-shifted execution |
| E05-P3-002 | Integration test suite executes in under 60 seconds — time full S05.12 integration suite run; assert elapsed < 60s | S05.12 | Benchmark | E05-R-009 | `@pytest.mark.timeout(60)` on integration suite |
| E05-P3-003 | `pipeline_crawl_duration_seconds` histogram has correct label — trigger crawl, verify histogram bucket labels include `crawler_type` dimension | S05.12 | Unit | — | Label schema validation |
| E05-P3-004 | `pipeline_agent_call_duration_seconds` histogram per agent type — trigger scoring and guide calls; verify label `agent_type` distinguishes between `relevance_scoring` and `submission_guide` | S05.12 | Unit | — | Per-agent observability contract |
| E05-P3-005 | Enrichment queue depth gauge matches actual pending items — insert 7 pending, 2 failed, 1 processing; assert gauge value = 7 (pending only) | S05.12 | Unit | — | Gauge accuracy |

**Total P3: 5 tests, ~3–6 hours**

---

## Execution Strategy

**Philosophy:** Run all functional tests (P0–P2) on every PR. The full pytest suite with testcontainers is expected to complete under 60 seconds (S05.12 acceptance criterion). P3 benchmarks and manual credential-gated smoke tests are the only exceptions.

| When | What | Expected Duration |
|------|------|------------------|
| **Every PR** | All P0, P1, P2 tests (pytest with testcontainers or Docker Compose services) | ~3–5 minutes |
| **Every PR** | P3 unit tests (timezone, histogram label, gauge accuracy) | +~1 minute |
| **Weekly / Manual** | Real-credential smoke tests against KraftData stage agents (tagged `@skip-ci`) | ~10–20 minutes |

No nightly-only tier is required: the backend-only nature of E05, combined with testcontainers, keeps the full suite well under 15 minutes. Defer to nightly only if container startup time degrades CI significantly (tracked under E05-R-009).

---

## Resource Estimates

| Priority | Test Count | Estimated Effort | Notes |
|----------|------------|-----------------|-------|
| P0 | 9 | ~18–30 hours | Integration tests; testcontainers + respx mock setup; async task execution |
| P1 | 30 | ~25–40 hours | Mix of unit (faster) and integration (slower) tests; some parameterized |
| P2 | 16 | ~10–18 hours | Mostly configuration and edge case assertions; lower setup overhead |
| P3 | 5 | ~3–6 hours | Benchmarks and label validation; fast to write and run |
| **Total** | **60** | **~56–94 hours** | **~1.5–2.5 weeks, 1 QA** |

### Prerequisites

**Test Fixtures and Factories:**

- `OpportunityFactory` — faker-based, generates valid `pipeline.opportunities` records with all JSONB fields; configurable `source_type`, `deadline`, `deleted_at`
- `CrawlerRunFactory` — generates `crawler_runs` records in any terminal state
- `CompanyProfileFactory` — generates active company profiles for relevance scoring fixtures
- `EnrichmentQueueFactory` — generates queue items in pending/failed states
- Fixture set per crawler type: one `respx` mock router configuration per portal (AOP, TED, EU Grants) with paginated and error response variants

**Tooling:**

- `pytest` + `pytest-celery` (or Celery `task_always_eager` mode for unit tests)
- `testcontainers` (PostgreSQL + Redis) — shared session scope
- `respx` for mocking `httpx`-based `ai_gateway_client` HTTP calls
- `pytest-asyncio` for async task and client tests
- `freezegun` for mocking back-off timing and circuit breaker `time.time()` assertions
- `prometheus_client` test utilities for metric value assertions

**Environment:**

- `CELERY_BROKER_URL` / `CELERY_RESULT_BACKEND` pointing to testcontainers Redis
- `DATABASE_URL` pointing to testcontainers PostgreSQL
- `AI_GATEWAY_BASE_URL` pointing to `respx` mock router (no real gateway)
- `CELERY_TIMEZONE=UTC`
- `OPPORTUNITY_RETENTION_DAYS=30` (override in P2-011 test)
- `ENRICHMENT_BATCH_SIZE=20` (override in P2-013 test)

---

## Quality Gate Criteria

### Pass/Fail Thresholds

- **P0 pass rate:** 100% (no exceptions — pipeline has no alternative data source)
- **P1 pass rate:** ≥95% (any failures must be triaged and PM-accepted before release)
- **P2/P3 pass rate:** ≥90% (informational; failures tracked as tech debt)
- **High-risk mitigations complete:** E05-R-001, E05-R-002, E05-R-003 all verified by P0 tests before release

### Coverage Targets

- **Task logic (`services/pipeline/tasks/`):** ≥80% line coverage
- **Model layer (`services/pipeline/models/`):** ≥85% line coverage
- **AI Gateway client module:** ≥85% line coverage
- **Security scenarios (soft-delete filter):** 100%

### Non-Negotiable Requirements

- [ ] All P0 tests pass before Sprint 4 close
- [ ] `crawler_runs` audit trail tested for all three crawler types
- [ ] Deduplication correctness verified (zero duplicate rows after concurrent upserts)
- [ ] Redis Stream events verified for both `opportunities.ingested` and `opportunities.expired`
- [ ] All 4 Prometheus metrics present and labelled correctly at `/metrics`

---

## Mitigation Plans

### E05-R-001: Deduplication Race Condition (Score: 6)

**Mitigation Strategy:**
1. Implement upsert as a single atomic `INSERT INTO pipeline.opportunities (...) ON CONFLICT (source_id, source_type) DO UPDATE SET ...` — never application-level check-then-insert
2. Verify unique constraint on `(source_id, source_type)` exists at DB level (S05.01 migration)
3. Integration test E05-P0-001 (sequential dedup) and E05-P0-002 (concurrent dedup) must pass before any crawler task is considered done

**Owner:** Backend / QA
**Timeline:** Sprint 3
**Status:** Planned
**Verification:** E05-P0-001 (zero new inserts on re-run) and E05-P0-002 (single row after concurrent writes) passing in CI

---

### E05-R-002: AI Gateway Cascade Failure / Orphaned crawler_runs (Score: 6)

**Mitigation Strategy:**
1. Implement `on_failure` Celery signal handler that unconditionally transitions `crawler_runs.status` to `failed` with error message — no terminal `running` or `retrying` states should persist after task finalization
2. Add `cleanup_stale_runs` check (as part of S05.11 enrichment worker or a dedicated health task) that marks `crawler_runs` records stuck in non-terminal states for >2× their expected window
3. Integration test E05-P0-004 (gateway down → `failed` status) must pass; also E05-P0-009 (full chain with one mock failure recovery)

**Owner:** Backend / QA
**Timeline:** Sprint 3–4
**Status:** Planned
**Verification:** After 3 exhausted retries, `crawler_runs.status = failed` and `error_message` is non-null; no records remain in `running` state after task finalization

---

### E05-R-003: Redis Streams Publish-or-Lose (Score: 6)

**Mitigation Strategy:**
1. Wrap `XADD` in try/except; on failure, log at ERROR level with `run_id`, `stream_name`, and `opportunity_ids` count so operations can identify and manually replay
2. Populate `crawler_runs.errors` JSONB (or a dedicated `stream_publish_failures` field) when publish exhausts retries — enables compensating recovery via admin task
3. Publish failure must NOT re-trigger the crawl; only the isolated publish step retries (at-least-once semantics per S05.09 AC)
4. Integration test E05-P0-007 (happy path publish) and E05-P0-008 (isolated publish retry) must pass

**Owner:** Backend / QA
**Timeline:** Sprint 3
**Status:** Planned
**Verification:** E05-P0-008: after XADD failure, DB data persisted, ERROR log contains `run_id`, crawler not re-executed

---

## Assumptions and Dependencies

### Assumptions

1. All AI Gateway calls from E05 go through the `ai_gateway_client` module (S05.03) — no direct `httpx` calls in crawler tasks; this is the mock boundary for all CI tests
2. `respx` mock router is sufficient for CI (no live KraftData agent calls required)
3. Celery `task_always_eager` mode or `pytest-celery` worker mode is available for synchronous task execution in unit tests without a running Redis broker
4. PostgreSQL supports `INSERT ... ON CONFLICT DO UPDATE` (requires PostgreSQL 9.5+, confirmed by E01 baseline)
5. `eusolicit-common` event bus abstraction from E01 is stable and its `publish()` method can be mocked at the module level
6. testcontainers PostgreSQL and Redis can be shared across all E05 test modules via session scope (no per-test container restart)

### Dependencies

| Dependency | Required By | Status |
|------------|-------------|--------|
| E01 complete: monorepo, PostgreSQL schema, Redis Streams event bus | Sprint 3 start | Assumed complete |
| E04 complete: AI Gateway operational; `ai_gateway_client` importable | Sprint 3, S05.03 | Assumed concurrent |
| `respx` mock fixtures per crawler type (AOP, TED, EU Grants) | Sprint 3, S05.04–S05.06 | QA creates as part of test dev |
| 5+ active company profiles seeded for relevance scoring tests | Sprint 3, S05.07 | QA creates via factory |
| `pytest-celery` or Celery eager mode confirmed compatible | Sprint 3 start | Tech spike needed |

### Risks to Plan

- **Risk:** E04 AI Gateway not fully stable when E05 S05.03 begins (same sprint)
  - **Impact:** `ai_gateway_client` integration tests blocked until E04 health probe is reliable
  - **Contingency:** Use `respx` mock exclusively for E05 CI; defer real-gateway smoke tests to post-sprint manual run tagged `@skip-ci`

- **Risk:** testcontainers startup time exceeds 60s combined with E04 tests in same CI job
  - **Impact:** E05-P3-002 benchmark fails; CI job may time out
  - **Contingency:** Separate E04 and E05 integration test jobs; share Docker layer cache; escalate to DevOps if >90s combined

---

## Interworking & Regression

| Service / Component | Impact | Regression Scope |
|--------------------|--------|-----------------|
| **E01 — Redis Streams / Event Bus** | E05 publishes `opportunities.ingested` and `opportunities.expired` events using E01's event bus abstraction | Verify event bus `publish()` API unchanged; stream name constants not renamed |
| **E01 — PostgreSQL / Alembic** | E05 adds `pipeline` schema tables on top of E01 baseline migration | `alembic upgrade head` and `downgrade -1` must not break E01 schema; migration chain integrity test |
| **E04 — AI Gateway** | E05 is the primary consumer of AI Gateway; all crawl, scoring, and guide tasks route through it | E04 health probe; `/execute/sync` and `/execute/stream` endpoints unchanged in signature; `agents.yaml` KraftData UUIDs still valid for E05 agents |
| **E02 — Auth / Company Profiles** | Company profiles used by relevance scoring must be retrievable; `company_id` scoping must be correct | Verify `active` company profile query returns expected rows; no RBAC breakage on read path |
| **E11 — ESPD / Proposal** | Submission guides generated by E05 are consumed by the proposal workflow | `pipeline.submission_guides` schema backward compatible; `steps` JSONB structure stable; `reviewed` flag respected by proposal service |
| **Future Analytics (E12)** | `opportunities.ingested` events are the upstream source for opportunity analytics | Stream name and payload schema (including `summary` counts) must remain stable |

---

## Follow-on Workflows

- Run `/bmad-testarch-atdd` to generate failing P0 tests (separate workflow; not auto-run)
- Run `/bmad-testarch-automate` for broader coverage expansion once E05 stories are implemented
- Run `/bmad-testarch-ci` to wire E05 test suite into the CI pipeline alongside E04

---

## Appendix

### Knowledge Base References

- `risk-governance.md` — Risk classification framework
- `probability-impact.md` — P×I scoring methodology
- `test-levels-framework.md` — Unit / Integration / E2E level selection
- `test-priorities-matrix.md` — P0–P3 prioritization criteria

### Related Documents

- Epic: `eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md`
- System Architecture Test Design: `eusolicit-docs/test-artifacts/test-design-architecture.md`
- QA Test Design (System-Level): `eusolicit-docs/test-artifacts/test-design-qa.md`
- E04 Test Design (Dependency): `eusolicit-docs/test-artifacts/test-design-epic-04.md`

---

**Generated by:** BMad TEA Agent — Master Test Architect
**Workflow:** `bmad-testarch-test-design`
**Version:** 4.0 (BMad v6)
