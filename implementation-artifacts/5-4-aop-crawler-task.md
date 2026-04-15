# Story 5.4: AOP Crawler Task

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer implementing the data pipeline**,
I want **a fully implemented `crawl_aop` Celery task that creates a `crawler_runs` audit record, calls the AOP Crawler Agent and Data Normalization Team agent via `ai_gateway_client`, atomically upserts normalized opportunity records into `pipeline.opportunities` using `(source_id, source_type='aop')` deduplication, updates `crawler_runs` with accurate counts and final status, and handles AI Gateway failures by marking runs as `failed` and triggering Celery-level retries**,
so that **AOP procurement opportunities are automatically crawled every 6 hours, deduplicated correctly, and made available for downstream scoring and guide generation — with a complete, accurate audit trail in `crawler_runs` for every run**.

## Acceptance Criteria

1. Successful crawl inserts/updates opportunities and records accurate counts in `crawler_runs` (`found`, `new_count`, `updated`, `started_at`, `ended_at`, `status=completed`)
2. Duplicate source records (same `source_id`, `source_type='aop'`) are updated via atomic `INSERT ... ON CONFLICT (source_id, source_type) DO UPDATE SET ...`, never duplicated
3. AI Gateway failure (`AIGatewayUnavailableError`) triggers Celery-level retry with `max_retries=3`, `retry_backoff=True`; `crawler_runs.status` reflects `retrying` on retry attempts and `failed` after retries are exhausted
4. `crawler_runs.status` is always transitioned to a terminal state (`completed` or `failed`) — no record remains in `running` or `retrying` indefinitely; `on_failure` handler ensures this even if the Celery worker process restarts
5. Integration test with mocked AI Gateway covers: (a) happy path — 10 opportunities inserted, counts correct; (b) gateway-down — status=`failed` after max retries, no partial data committed

## Tasks / Subtasks

- [ ] Task 1: Implement the upsert helper (AC: 2)
  - [ ] 1.1 Create `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py` — module-level `upsert_opportunities(session, records: list[dict], source_type: str) -> tuple[int, int]` function that returns `(new_count, updated_count)`. Use a single `sqlalchemy.dialects.postgresql.insert(...).on_conflict_do_update(index_elements=["source_id", "source_type"], set_={...})` statement. The `set_` dict must include all mutable columns (`title`, `description`, `status`, `deadline`, `budget_min`, `budget_max`, `currency`, `country`, `region`, `contracting_authority`, `cpv_codes`, `evaluation_criteria`, `mandatory_documents`, `raw_data`, `published_at`, `updated_at`). Detect new vs updated by comparing `inserted_primary_key` result: rows where `created_at == updated_at` are new. Return `(new_count, updated_count)`.
  - [ ] 1.2 Add `services/data-pipeline/src/data_pipeline/workers/tasks/__init__.py` re-export of `upsert_opportunities` so sibling tasks (S05.05, S05.06) can import from the shared helper.

- [ ] Task 2: Implement the normalizer helper (AC: 1)
  - [ ] 2.1 Create `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py` — function `normalize_aop_response(raw_output: dict) -> list[dict]` that extracts the `opportunities` list from the AI Gateway agent's `output` dict and maps each item to the `pipeline.opportunities` column schema. Required field mappings (agent field → DB column): `id` → `source_id`, `"aop"` (constant) → `source_type`, `title` → `title`, `description` → `description`, `status` → `status`, `deadline` → `deadline` (ISO 8601 string to `datetime`), `budget.min` → `budget_min`, `budget.max` → `budget_max`, `budget.currency` → `currency`, `country` → `country`, `region` → `region`, `contracting_authority` → `contracting_authority`, `cpv_codes` → `cpv_codes`, `raw_agent_response` (full dict) → `raw_data`. Missing optional fields default to `None`; missing `cpv_codes` defaults to `[]`.

- [ ] Task 3: Replace `crawl_aop` stub with full implementation (AC: 1, 2, 3, 4)
  - [ ] 3.1 Rewrite `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py`:
    - Keep `@shared_task(name="pipeline.crawl_aop", bind=True, max_retries=3, autoretry_for=(AIGatewayUnavailableError,), retry_backoff=True)`
    - Step 1 — Create `CrawlerRun` record: `run = CrawlerRun(crawler_type="aop", status="running")`; `session.add(run)`; `session.commit()`; capture `run_id = run.id` before any agent call
    - Step 2 — Call AOP Crawler Agent: `client.call_agent("aop-crawler-agent", payload={}, agent_type=AgentType.CRAWLER, correlation_id=str(run_id))`; on `AIGatewayUnavailableError`, update `run.status = "retrying"`, commit, then re-raise so Celery handles retry
    - Step 3 — Call Data Normalization Team agent: `client.call_agent("data-normalization-team", payload={"opportunities": raw_output, "source_type": "aop"}, agent_type=AgentType.NORMALIZATION, correlation_id=str(run_id))`; on error, follow same retry pattern
    - Step 4 — Upsert: call `upsert_opportunities(session, normalized_records, "aop")`; capture `(new_count, updated_count)`
    - Step 5 — Update `CrawlerRun`: `run.status = "completed"`, `run.found = len(normalized_records)`, `run.new_count = new_count`, `run.updated = updated_count`, `run.ended_at = datetime.now(UTC)`; commit
    - Wrap the entire sequence in `try/except Exception` so the `on_failure` path (Task 3.2) always fires
  - [ ] 3.2 Add Celery `on_failure` signal callback at module level (or as a `bind=True` method): when the task transitions to `FAILURE` state, open a new DB session, query `CrawlerRun` by `run_id` (stored in task state via `self.update_state`), set `status="failed"`, `errors={"message": str(exc)}`, `ended_at=now(UTC)`, commit. This ensures no `CrawlerRun` record is left in a non-terminal state even if the worker process restarts between retries.
  - [ ] 3.3 Add structured logging with `structlog` (matching the `ai_gateway_client` pattern): bind `task_name="pipeline.crawl_aop"`, `crawler_type="aop"`, `run_id=str(run_id)`, `correlation_id` to all log calls. Log at INFO on run start and completion; log at WARNING on retry; log at ERROR on failure with exception traceback.
  - [ ] 3.4 Add `get_session()` helper import: the task needs a synchronous SQLAlchemy session (not async). Create or import `services/data-pipeline/src/data_pipeline/db.py` `get_sync_session()` context manager that reads `DATABASE_URL` from env, converts `postgresql+asyncpg://` to `postgresql://` if necessary, and returns a `sqlalchemy.orm.Session` (using `sessionmaker`).

- [ ] Task 4: Create `services/data-pipeline/src/data_pipeline/db.py` (AC: 1, 3, 4)
  - [ ] 4.1 Implement `get_sync_session()` as a `contextmanager` that creates a synchronous `sqlalchemy.orm.Session` from `DATABASE_URL` env var. Convert async URL driver (`+asyncpg`) to sync (`+psycopg2`) automatically. Session is scoped to the task invocation and committed/closed by the caller. This module is the sync DB access layer used by all Celery tasks.

- [ ] Task 5: Write integration tests (AC: 1, 2, 3, 4, 5)
  - [ ] 5.1 Create `services/data-pipeline/tests/integration/test_crawl_aop.py`:
    - **E05-P0-003** `test_crawl_aop_happy_path` — use `respx` to mock both the AOP Crawler Agent (`POST /workflows/aop-crawler-agent/run`) and the Data Normalization Team agent (`POST /workflows/data-normalization-team/run`) returning 10 normalized opportunities. Call `crawl_aop.apply()` (Celery eager mode). Assert: (a) `pipeline.opportunities` contains exactly 10 rows with `source_type='aop'`; (b) `crawler_runs` record has `status='completed'`, `found=10`, `new_count=10`, `updated=0`, `ended_at IS NOT NULL`.
    - **E05-P0-004** `test_crawl_aop_gateway_down` — mock all 4 HTTP calls (3 retries + original) to return 503; configure `task_always_eager=True` with `task_eager_propagates=False`. Assert: (a) `crawler_runs.status = 'failed'`; (b) `crawler_runs.errors` JSONB contains `message` key; (c) `pipeline.opportunities` count is 0 (no partial data).
    - **E05-P0-001** `test_crawl_aop_dedup_no_new_inserts` — run `crawl_aop.apply()` twice with identical mocked response. Assert: after second run, `COUNT(*)` on `pipeline.opportunities` is still 10 (not 20); second `crawler_runs` record has `new_count=0`, `updated=0` (data unchanged).
    - **E05-P0-002** `test_concurrent_upsert_safety` — use `threading.Thread` to call `upsert_opportunities` from two threads simultaneously with identical payloads. Assert: final `COUNT(*)` = 1 (single row, no race condition duplicate).
    - **E05-P1-012** `test_crawl_aop_crawler_runs_accounting` — detailed assertion: verify `crawler_runs` record is created with `status='running'` before agent calls complete; verify final `started_at < ended_at`; verify `crawler_type='aop'`.
    - **E05-P1-013** `test_crawl_aop_unrecoverable_failure_marks_failed` — mock a non-retryable exception (e.g. `ValueError`) raised inside the task. Assert `crawler_runs.status = 'failed'` and `error_message` is set.
    - **E05-P2-005** `test_crawl_aop_retrying_status_mid_retry` — mock first call to return `AIGatewayUnavailableError`, second call to succeed. Assert `crawler_runs` transitions through `running → retrying` before reaching `completed`.

- [ ] Task 6: Add `psycopg2-binary` dependency (AC: 4)
  - [ ] 6.1 Verify `psycopg2-binary` is present in `[project.dependencies]` in `services/data-pipeline/pyproject.toml`. If absent, add it. This is required for the synchronous `get_sync_session()` DB access from Celery task threads.

## Dev Notes

### Architecture Context

S05.04 implements the first production-ready Celery crawler task. It builds directly on three prior stories:

- **S05.01** — `Opportunity` and `CrawlerRun` SQLAlchemy models with `pipeline` schema tables
- **S05.02** — Celery app bootstrap; `crawl_aop` is already registered as `pipeline.crawl_aop` on the `pipeline_crawl` queue with a 6-hour Beat schedule
- **S05.03** — `AIGatewayClient` with retry, circuit breaker, and correlation ID injection; `AIGatewayUnavailableError` is already wired into the task stub's `autoretry_for` tuple

**Two-hop agent call chain for AOP:**

```
crawl_aop (Celery task)
    ├── Step 1: AIGatewayClient.call_agent("aop-crawler-agent", payload={}, agent_type=AgentType.CRAWLER)
    │             → returns {"opportunities": [...raw AOP records...]}
    └── Step 2: AIGatewayClient.call_agent("data-normalization-team", payload={"opportunities": [...], "source_type": "aop"}, agent_type=AgentType.NORMALIZATION)
                  → returns {"opportunities": [...normalized records conforming to pipeline.opportunities schema...]}
```

The normalization agent is called as a second hop because AOP raw data contains non-standard field names and formats. The `data-normalization-team` agent returns records already mapped to EU Solicit's canonical schema.

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.04]

### CRITICAL: Sync vs Async

Celery tasks run in **synchronous threads**. The `crawl_aop` implementation must:
- Use **synchronous `httpx`** via `AIGatewayClient` (already sync from S05.03)
- Use **synchronous SQLAlchemy** (`sqlalchemy.orm.Session` + `psycopg2`), NOT `AsyncSession`/`asyncpg`
- Use `threading.Lock` if any shared state is needed

The existing `conftest.py` uses `AsyncSession` for testing models — the crawler task tests must set up a separate sync session fixture (or use `psycopg2` directly for assertions).

### CRITICAL: Atomic Upsert for Dedup Safety

The deduplication constraint (AC 2) **must** be implemented as a single PostgreSQL statement:

```python
from sqlalchemy.dialects.postgresql import insert

stmt = insert(Opportunity).values(records)
stmt = stmt.on_conflict_do_update(
    index_elements=["source_id", "source_type"],
    set_={
        "title": stmt.excluded.title,
        "updated_at": func.now(),
        # ... all mutable columns
    },
)
session.execute(stmt)
session.commit()
```

**Never** use application-level check-then-insert (SELECT then INSERT). This pattern avoids the race condition documented in E05-R-001 where two concurrent workers upsert the same row.

[Source: test-design-epic-05.md#E05-R-001 — Dedup Race Condition (Score 6)]

### `crawler_runs` Status State Machine

```
created → running → (on AIGatewayUnavailableError) retrying
                  ↓ (on success) completed
                  ↓ (on non-retryable error or max_retries exhausted) failed
```

The `on_failure` Celery handler must **always** set `status='failed'` regardless of how the task terminates. This is the mitigation for E05-R-002 (AI Gateway cascade failure leaving orphaned `crawler_runs`). The handler must open a **new** DB session (the task's session may be in an invalid state after an exception).

[Source: test-design-epic-05.md#E05-R-002 — AI Gateway Cascade Failure (Score 6)]

### `get_sync_session()` Pattern

```python
# services/data-pipeline/src/data_pipeline/db.py
from contextlib import contextmanager
import os
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker

def _get_sync_url() -> str:
    url = os.environ.get("DATABASE_URL", "postgresql://...")
    # Convert asyncpg URL to psycopg2 if needed (CI testcontainers use asyncpg URLs)
    return url.replace("postgresql+asyncpg://", "postgresql://")

_engine = None

def _get_engine():
    global _engine
    if _engine is None:
        _engine = create_engine(_get_sync_url(), pool_pre_ping=True)
    return _engine

@contextmanager
def get_sync_session():
    factory = sessionmaker(bind=_get_engine(), expire_on_commit=False)
    session: Session = factory()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()
```

### Agent Names for This Task

| Call | Logical Agent Name | `AgentType` |
|------|--------------------|-------------|
| Crawler call | `aop-crawler-agent` | `AgentType.CRAWLER` |
| Normalization call | `data-normalization-team` | `AgentType.NORMALIZATION` |

These names are registered in `eusolicit-app/services/ai-gateway/config/agents.yaml`. Do not hardcode the KraftData UUID — always use the logical name registered in the agent registry.

[Source: eusolicit-docs/implementation-artifacts/5-3-ai-gateway-client-module-with-retry-logic.md#Agent-Names]

### Mock Fixture Design for Integration Tests

Tests use `respx` to intercept `httpx` calls at the transport layer. Pattern:

```python
import respx
import httpx

@pytest.fixture
def aop_mock_10_opportunities(respx_mock):
    """Mock: AOP crawl returns 10 opportunities; normalization returns 10 normalized records."""
    raw_opps = [{"id": f"AOP-{i:04d}", "title": f"AOP Tender {i}", ...} for i in range(10)]
    normalized = [{"source_id": f"AOP-{i:04d}", "source_type": "aop", "title": f"AOP Tender {i}", ...} for i in range(10)]

    respx_mock.post("http://ai-gateway:8004/workflows/aop-crawler-agent/run").mock(
        return_value=httpx.Response(200, json={"execution_id": "ex-1", "status": "completed", "output": {"opportunities": raw_opps}})
    )
    respx_mock.post("http://ai-gateway:8004/workflows/data-normalization-team/run").mock(
        return_value=httpx.Response(200, json={"execution_id": "ex-2", "status": "completed", "output": {"opportunities": normalized}})
    )
    return normalized
```

Use `CELERY_TASK_ALWAYS_EAGER=True` in test environment via `celery_config` fixture (already in `tests/conftest.py`).

### File Locations

| Purpose | Path |
|---------|------|
| Task stub to replace | `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` |
| New upsert helper | `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py` |
| New normalize helper | `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py` |
| New sync DB session | `services/data-pipeline/src/data_pipeline/db.py` |
| Integration tests | `services/data-pipeline/tests/integration/test_crawl_aop.py` |
| Opportunity model | `services/data-pipeline/src/data_pipeline/models/opportunity.py` |
| CrawlerRun model | `services/data-pipeline/src/data_pipeline/models/crawler_run.py` |
| AI Gateway client | `services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py` |

### Test Design Traceability

Tests in this story directly verify the following scenarios from the Epic 5 test design:

| Test ID | Priority | Scenario |
|---------|----------|----------|
| E05-P0-001 | P0 | Dedup: second run with same data → zero new inserts |
| E05-P0-002 | P0 | Concurrent upsert safety → single row in DB |
| E05-P0-003 | P0 | AOP crawl happy path → 10 opps, `crawler_runs` counts correct |
| E05-P0-004 | P0 | AOP crawl AI Gateway down → `crawler_runs.status=failed`, no partial data |
| E05-P1-012 | P1 | `crawler_runs` accounting: `status=running` at start, `completed` at end |
| E05-P1-013 | P1 | Unrecoverable failure → `crawler_runs.status=failed` with `error_message` |
| E05-P2-005 | P2 | `retrying` status reflected in `crawler_runs` mid-retry |

Mitigations verified:
- **E05-R-001** (Dedup race, Score 6): covered by E05-P0-001 + E05-P0-002
- **E05-R-002** (AI Gateway cascade / orphaned runs, Score 6): covered by E05-P0-004 + E05-P1-013

[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md]
