# E05: Data Pipeline & Opportunity Ingestion

**Sprint**: 3--4 | **Points**: 34 | **Dependencies**: E01, E04 | **Milestone**: Demo

## Goal

Stand up the Celery-based data pipeline that orchestrates scheduled crawling of AOP, TED, and EU grant portals via KraftData agents (called through the AI Gateway), normalizes and deduplicates the returned opportunity data, scores each opportunity for relevance against stored company profiles, generates portal-specific submission guides, and publishes `opportunities.ingested` events to Redis Streams so downstream services can react in real time. By the end of Sprint 4 the pipeline must run autonomously on its Beat schedule with full observability through the `crawler_runs` audit table.

## Acceptance Criteria

- [ ] Celery Beat schedules fire correctly: AOP every 6 h, TED every 12 h, EU Grants daily
- [ ] Each crawl task calls the corresponding KraftData crawler agent through AI Gateway and persists raw results
- [ ] Data Normalization Team agent is called via AI Gateway; output conforms to the `pipeline.opportunities` schema
- [ ] Upsert logic deduplicates on `(source_id, source_type)` -- updates existing rows, inserts new ones
- [ ] Relevance Scoring Agent is invoked for every new or updated opportunity; scores stored in `relevance_scores` JSONB
- [ ] Submission Guide Agent generates step-by-step instructions stored in `pipeline.submission_guides`
- [ ] `opportunities.ingested` event published to Redis Streams after each successful crawl cycle with the list of affected `opportunity_ids`
- [ ] Weekly cleanup task soft-deletes opportunities past deadline + configured retention period
- [ ] `crawler_runs` table records start/end times, status, counts of found/new/updated opportunities, and errors
- [ ] All AI Gateway calls use structured retry with exponential back-off (max 3 retries)
- [ ] Pipeline survives AI Gateway downtime gracefully -- tasks retry via Celery and do not duplicate data
- [ ] Integration tests cover the full crawl-to-event flow using mocked AI Gateway responses

## Stories

### S05.01: Pipeline schema migration and model layer
**Points**: 3 | **Type**: backend

Create an Alembic migration for the `pipeline` schema with four tables: `opportunities`, `submission_guides`, `crawler_runs`, and `enrichment_queue` as defined in the data model. Add SQLAlchemy models with proper column types (`text[]` for `cpv_codes`, `JSONB` for `evaluation_criteria`, `mandatory_documents`, `relevance_scores`, `raw_data`, `steps`). Include indexes on `(source_id, source_type)` unique, `deadline`, `status`, and `crawler_type`. Add `deleted_at` soft-delete filter as a default query option on the `opportunities` model.

**Acceptance Criteria**:
- [ ] `alembic upgrade head` creates all four tables in the `pipeline` schema
- [ ] `alembic downgrade -1` cleanly drops them
- [ ] Unique constraint on `(source_id, source_type)` enforced at DB level
- [ ] SQLAlchemy models pass type-checking and basic CRUD unit tests

---

### S05.02: Celery app bootstrap and Beat schedule configuration
**Points**: 2 | **Type**: backend

Configure the Celery application with Redis as broker and result backend. Define the Beat schedule with three periodic entries: `crawl-aop` (every 6 h), `crawl-ted` (every 12 h), `crawl-eu-grants` (daily at 02:00 UTC). Add a fourth entry `cleanup-expired-opportunities` (weekly, Sunday 04:00 UTC). Externalise all intervals and cron expressions to environment variables with sensible defaults. Include a health-check task (`pipeline.health`) that the container liveness probe can poll.

**Acceptance Criteria**:
- [ ] `celery -A pipeline inspect scheduled` lists all four periodic tasks
- [ ] Intervals are overridable via env vars without code changes
- [ ] Health-check task returns OK and is callable from the container probe endpoint
- [ ] Celery worker and Beat start cleanly in Docker Compose

---

### S05.03: AI Gateway client module with retry logic
**Points**: 2 | **Type**: backend

Create a shared `ai_gateway_client` module used by all pipeline tasks to call KraftData agents through the AI Gateway. The client wraps HTTP calls with: structured retry (exponential back-off, max 3 attempts, jitter), request/response logging at DEBUG level, timeout configuration per agent type, and circuit-breaker pattern (open after 5 consecutive failures, half-open after 60 s). Return typed Pydantic response models. Raise `AIGatewayUnavailableError` on exhausted retries so Celery can handle task-level retry separately.

**Acceptance Criteria**:
- [ ] Unit tests verify retry counts, back-off timing, and circuit-breaker state transitions
- [ ] Timeout and retry parameters configurable via environment variables
- [ ] All outbound calls include a correlation ID header for tracing
- [ ] `AIGatewayUnavailableError` raised after retries exhausted; Celery sees it as a retriable exception

---

### S05.04: AOP crawler task
**Points**: 3 | **Type**: backend

Implement the `crawl_aop` Celery task. On execution: (1) create a `crawler_runs` record with status `running`, (2) call the AOP Crawler Agent via `ai_gateway_client`, (3) pass raw results to the Data Normalization Team agent, (4) upsert normalized records into `pipeline.opportunities` using `(source_id, source_type='aop')` dedup, (5) update `crawler_runs` with counts and final status. On any unrecoverable error, mark the run as `failed` with the error message. Task binds `autoretry_for=(AIGatewayUnavailableError,)` with `max_retries=3` and `retry_backoff=True`.

**Acceptance Criteria**:
- [ ] Successful crawl inserts/updates opportunities and records accurate counts in `crawler_runs`
- [ ] Duplicate source records are updated, not duplicated
- [ ] AI Gateway failure triggers Celery-level retry; `crawler_runs` status reflects `retrying`
- [ ] Integration test with mocked AI Gateway covers happy path and gateway-down scenario

---

### S05.05: TED crawler task
**Points**: 3 | **Type**: backend

Implement the `crawl_ted` Celery task following the same pattern as S05.04 but calling the TED Crawler Agent. TED responses include CPV codes and NUTS region codes -- ensure the normalization step maps NUTS codes to human-readable `region` and `country` fields. Handle TED-specific pagination: the agent may return a `next_page_token`; the task must loop until all pages are consumed before completing the run.

**Acceptance Criteria**:
- [ ] Paginated TED results are fully consumed across multiple agent calls
- [ ] NUTS codes correctly mapped to `region` and `country` on opportunity records
- [ ] `crawler_runs` totals reflect all pages, not just the last
- [ ] Test covers multi-page scenario with two mocked pages

---

### S05.06: EU Grants crawler task
**Points**: 3 | **Type**: backend

Implement the `crawl_eu_grants` Celery task calling the EU Grant Portal Agent. EU grant opportunities use `opportunity_type='grant'` and include specific fields: `evaluation_criteria` (scoring matrix as JSONB) and `mandatory_documents` (list of required attachments). Ensure these fields are populated during normalization. Budget fields should handle both single-value and range formats from the source.

**Acceptance Criteria**:
- [ ] Grant opportunities stored with `opportunity_type='grant'` and populated `evaluation_criteria`
- [ ] `mandatory_documents` JSONB contains structured list of required documents per grant call
- [ ] Budget range correctly parsed into `budget_min` / `budget_max` even when source provides a single figure
- [ ] `crawler_runs` record accurately reflects the grant crawl cycle

---

### S05.07: Relevance scoring post-processing task
**Points**: 3 | **Type**: backend

Implement a `score_opportunities` Celery task that is chained after each crawl task completes. For every opportunity ID in the crawl result set, call the Relevance Scoring Agent via AI Gateway, passing the opportunity data and each active company profile. Store returned scores in the `relevance_scores` JSONB column keyed by `company_id`. Use Celery `chord` or `group` to parallelise scoring calls (configurable concurrency, default 5). If scoring fails for a single opportunity, log the error and continue -- do not fail the entire batch.

**Acceptance Criteria**:
- [ ] `relevance_scores` JSONB populated with per-company scores for each scored opportunity
- [ ] Scoring runs in parallel with configurable concurrency limit
- [ ] Single-opportunity scoring failure does not block the rest of the batch
- [ ] Failed scoring attempts are queued in `enrichment_queue` for later retry
- [ ] Test verifies partial failure scenario (3 of 5 succeed, 2 fail and land in enrichment queue)

---

### S05.08: Submission guide generation task
**Points**: 3 | **Type**: backend

Implement a `generate_submission_guides` Celery task chained after scoring. For each new opportunity (not previously processed), call the Submission Guide Agent via AI Gateway with the opportunity data and `source_type` to generate portal-specific step-by-step submission instructions. Store the result in `pipeline.submission_guides` with `reviewed=false`. Skip opportunities that already have a guide unless the opportunity data has changed (`updated_at` is newer than the guide's `created_at`).

**Acceptance Criteria**:
- [ ] `submission_guides` row created for each new opportunity with structured `steps` JSONB
- [ ] Existing guides are not regenerated unless the underlying opportunity was updated
- [ ] `generated_by` field records the agent version/ID returned by AI Gateway
- [ ] Guide generation failure queues the opportunity in `enrichment_queue` with `enrichment_type='submission_guide'`

---

### S05.09: Redis Streams event publishing
**Points**: 2 | **Type**: backend

After each complete crawl cycle (crawl + normalize + score + guide), publish an `opportunities.ingested` event to the Redis Streams event bus. The event payload must include: `crawler_type`, `run_id`, `opportunity_ids` (list of all new and updated IDs), `timestamp`, and `summary` (counts of new, updated, unchanged). Use the event bus abstraction from E01. Ensure at-least-once delivery semantics -- if publishing fails, the task retries the publish step only (data is already persisted).

**Acceptance Criteria**:
- [ ] Event published to the correct Redis Stream with all required payload fields
- [ ] Downstream consumer (integration test) receives the event and can deserialize the opportunity ID list
- [ ] Publish failure triggers isolated retry without re-running the crawl
- [ ] Event payload `summary` counts match `crawler_runs` record counts

---

### S05.10: Expired opportunity cleanup task
**Points**: 2 | **Type**: backend

Implement the `cleanup_expired_opportunities` weekly Celery Beat task. Soft-delete (set `deleted_at = now()`) all opportunities where `deadline < now() - retention_period`. The retention period defaults to 30 days and is configurable via environment variable. Also cascade soft-delete related `submission_guides`. Log the count of affected records. Publish an `opportunities.expired` event to Redis Streams with the list of expired IDs.

**Acceptance Criteria**:
- [ ] Opportunities past deadline + retention period have `deleted_at` set
- [ ] Related `submission_guides` also soft-deleted
- [ ] Retention period configurable via `OPPORTUNITY_RETENTION_DAYS` env var
- [ ] `opportunities.expired` event published with affected IDs
- [ ] Default query filters on the `opportunities` model exclude soft-deleted rows

---

### S05.11: Enrichment queue worker
**Points**: 3 | **Type**: backend

Implement a `process_enrichment_queue` Celery Beat task (runs every 15 minutes) that picks up pending items from `pipeline.enrichment_queue` and retries the failed enrichment step (either relevance scoring or submission guide generation). Items are processed in FIFO order with a configurable batch size (default 20). After 3 failed attempts, mark the item as `failed` and emit an `enrichment.failed` event for alerting. On success, remove the item from the queue and ensure the parent opportunity record is updated.

**Acceptance Criteria**:
- [ ] Pending enrichment items retried automatically every 15 minutes
- [ ] Items marked `failed` after 3 attempts and `enrichment.failed` event emitted
- [ ] Successful retry updates the opportunity record (score or guide) and deletes the queue item
- [ ] Batch size configurable via `ENRICHMENT_BATCH_SIZE` env var
- [ ] Test covers: item succeeds on retry, item fails 3 times and is marked permanently failed

---

### S05.12: End-to-end integration tests and pipeline observability
**Points**: 5 | **Type**: backend

Write integration tests that exercise the full pipeline flow from Beat trigger through event publication, using mocked AI Gateway responses (one fixture set per crawler type). Verify: crawler_runs accounting, dedup correctness (run same data twice), relevance scores populated, submission guides created, events published. Add structured logging across all tasks with correlation IDs. Add Prometheus metrics: `pipeline_crawl_duration_seconds` (histogram), `pipeline_opportunities_total` (counter by source_type and action), `pipeline_enrichment_queue_depth` (gauge), `pipeline_agent_call_duration_seconds` (histogram by agent type).

**Acceptance Criteria**:
- [ ] Integration test suite runs in CI with mocked AI Gateway (no external dependencies)
- [ ] Dedup test: second run with same data produces zero new inserts, only updates where data changed
- [ ] All four Prometheus metrics exported and scrapeable at `/metrics`
- [ ] Every log line includes `correlation_id`, `task_name`, and `crawler_type` where applicable
- [ ] Test execution time under 60 seconds with mocked dependencies
