# Story 5.7: Relevance Scoring Post-Processing Task

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer implementing the data pipeline**,
I want **a `score_opportunities` Celery task that is automatically chained after each crawler task (AOP, TED, EU Grants) and, for every opportunity ID in the crawl result set, dispatches parallel `_score_single_opportunity` sub-tasks (via Celery `group`, routed to a dedicated `pipeline_scoring` queue with configurable concurrency via `SCORING_CONCURRENCY` env var, default 5) that each load the opportunity and all active company profiles, call the Relevance Scoring Agent via `ai_gateway_client` once per company, and store the returned per-company scores in `relevance_scores` JSONB keyed by `company_id` — with single-opportunity failures logged and queued in `enrichment_queue` (not propagated to abort the batch)**,
so that **every crawled opportunity is automatically scored against all active company profiles in the background, the scores are immediately available for opportunity-listing and analytics features, and transient scoring failures are tracked in the enrichment queue for automatic retry by S05.11 without any data loss or batch interruption**.

## Acceptance Criteria

1. `relevance_scores` JSONB on `pipeline.opportunities` is populated with per-company scores in the structure `{company_id: score, ...}` for every opportunity processed by the scoring task; all active company profiles are scored for each opportunity
2. Scoring is dispatched as a Celery `group` of `_score_single_opportunity` sub-tasks routed to the `pipeline_scoring` queue; the `SCORING_CONCURRENCY` env var (default 5) controls the worker concurrency for that queue (documented in deployment config)
3. A single-opportunity scoring failure (AI Gateway error or any exception within `_score_single_opportunity`) does not propagate to abort the group — the failing task returns a failure dict without re-raising, the remaining opportunities in the batch continue scoring normally
4. When `_score_single_opportunity` raises an unhandled exception, an `EnrichmentQueueItem` record is created with `enrichment_type='relevance_scoring'`, `status='pending'`, `attempts=0`, and the correct `opportunity_id`; the item is retrievable by S05.11 for later retry
5. `score_opportunities` is automatically triggered after each successful crawl cycle: `crawl_aop`, `crawl_ted`, and `crawl_eu_grants` each call `score_opportunities.apply_async(args=[result])` at the end of their successful execution path, passing the crawl result dict (which now includes `opportunity_ids`)
6. The crawl task return dict is extended with `opportunity_ids: list[str]` (UUIDs as strings) for all upserted opportunities (new and updated) so `score_opportunities` and S05.09 (event publishing) can consume them
7. Integration test verifies the partial-failure scenario: 5 opportunities submitted for scoring, mock AI Gateway returns success for opportunities 1, 3, 5 and raises `AIGatewayUnavailableError` for opportunities 2 and 4 — assert 3 `relevance_scores` JSONB fields populated, 2 `enrichment_queue` entries created, no exception propagated to the test
8. All structured log lines emitted by scoring tasks include `task_name`, `opportunity_id`, and `correlation_id` fields

## Tasks / Subtasks

- [ ] Task 1: Extend `upsert_opportunities` to return upserted opportunity IDs (AC: 6)
  - [ ] 1.1 Edit `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py`: add `.returning(Opportunity.id)` to the `INSERT ... ON CONFLICT` statement so the execute call returns the IDs of all upserted rows (both inserted and updated). Collect them into a `list[str]`: `opportunity_ids = [str(row.id) for row in result]`. Change the function signature return type from `tuple[int, int]` to `tuple[int, int, list[str]]` and update the return statement to `return new_count, updated_count, opportunity_ids`.
  - [ ] 1.2 Update callers in `crawl_aop.py`, `crawl_ted.py`, and `crawl_eu_grants.py`: change `new_count, updated_count = upsert_opportunities(...)` to `new_count, updated_count, opportunity_ids = upsert_opportunities(...)` in all three files. Add `"opportunity_ids": opportunity_ids` to each crawler task's return dict alongside the existing `run_id`, `found`, `new_count`, `updated` keys.

- [ ] Task 2: Chain `score_opportunities` from each crawler task (AC: 5)
  - [ ] 2.1 In `crawl_aop.py`, after the Step 6 `CrawlerRun` terminal-state update and commit, add:
    ```python
    score_opportunities.apply_async(args=[result])
    ```
    where `result` is the dict that will be returned. Import `score_opportunities` from `data_pipeline.workers.tasks.score_opportunities`. Place the call inside the main task body but outside the `on_failure` handler. Add a DEBUG-level log line `"crawl_aop.scoring_dispatched"` with `opportunity_count=len(opportunity_ids)`.
  - [ ] 2.2 Apply the same change to `crawl_ted.py` with a `"crawl_ted.scoring_dispatched"` log.
  - [ ] 2.3 Apply the same change to `crawl_eu_grants.py` with a `"crawl_eu_grants.scoring_dispatched"` log.

- [ ] Task 3: Implement `score_opportunities` orchestrator task (AC: 2, 5)
  - [ ] 3.1 Create `services/data-pipeline/src/data_pipeline/workers/tasks/score_opportunities.py`. Add module-level constant `SCORING_CONCURRENCY = int(os.getenv("SCORING_CONCURRENCY", "5"))`. Implement `score_opportunities` as:
    ```python
    @shared_task(
        name="pipeline.score_opportunities",
        bind=True,
        queue="pipeline_scoring",
    )
    def score_opportunities(self, crawl_result: dict[str, Any]) -> dict[str, Any]:
    ```
    Steps:
    - Extract `opportunity_ids: list[str] = crawl_result.get("opportunity_ids", [])`. If empty, log INFO `"score_opportunities.skipped"` with `reason="no_opportunity_ids"` and return `{"scored": 0, "failed": 0, "dispatched": 0}`.
    - Bind structlog context: `task_name="pipeline.score_opportunities"`, `opportunity_count=len(opportunity_ids)`, `run_id=crawl_result.get("run_id")`.
    - Build a Celery `group` of `_score_single_opportunity.s(opp_id)` for each `opp_id` in `opportunity_ids`.
    - Call `scoring_group.apply_async()` to dispatch asynchronously (fire-and-forget from the orchestrator's perspective).
    - Log INFO `"score_opportunities.dispatched"` with `dispatched=len(opportunity_ids)`.
    - Return `{"dispatched": len(opportunity_ids), "run_id": crawl_result.get("run_id")}`.

- [ ] Task 4: Implement `_score_single_opportunity` sub-task (AC: 1, 3, 4, 8)
  - [ ] 4.1 Add `_score_single_opportunity` to `score_opportunities.py` as a sibling task:
    ```python
    @shared_task(
        name="pipeline._score_single_opportunity",
        bind=True,
        queue="pipeline_scoring",
    )
    def _score_single_opportunity(self, opportunity_id: str) -> dict[str, Any]:
    ```
    Steps:
    - Generate `correlation_id = str(uuid4())` for AI Gateway calls. Bind structlog: `task_name="pipeline._score_single_opportunity"`, `opportunity_id=opportunity_id`, `correlation_id=correlation_id`.
    - Wrap the entire body in `try/except Exception as exc` to prevent failure propagation.
    - **Happy path**:
      - Open a sync session (`get_sync_session()`); load `opp = session.get(Opportunity, UUID(opportunity_id))`. If `None`, log WARNING `"score_single_opportunity.opportunity_not_found"` and return `{"opportunity_id": opportunity_id, "status": "not_found"}`.
      - Query active company profiles: `companies = session.execute(select(CompanyProfile).where(CompanyProfile.is_active.is_(True))).scalars().all()`. If empty, log WARNING `"score_single_opportunity.no_active_companies"` and return `{"opportunity_id": opportunity_id, "status": "no_companies"}`.
      - Build opportunity payload:
        ```python
        opp_payload = {
            "id": str(opp.id),
            "source_type": opp.source_type,
            "title": opp.title,
            "description": opp.description,
            "cpv_codes": opp.cpv_codes,
            "country": opp.country,
            "region": opp.region,
            "deadline": opp.deadline.isoformat() if opp.deadline else None,
            "budget_min": str(opp.budget_min) if opp.budget_min is not None else None,
            "budget_max": str(opp.budget_max) if opp.budget_max is not None else None,
            "evaluation_criteria": opp.evaluation_criteria,
        }
        ```
      - For each company in `companies`, call:
        ```python
        client = get_client()
        response = client.call_agent(
            "relevance-scoring-agent",
            payload={"opportunity": opp_payload, "company_profile": company.to_scoring_dict()},
            agent_type=AgentType.SCORING,
            correlation_id=correlation_id,
        )
        scores[str(company.id)] = response.output.get("score")
        ```
        Collect into `scores: dict[str, Any] = {}`.
      - Open a new sync session; execute `UPDATE pipeline.opportunities SET relevance_scores = :scores WHERE id = :id`; commit.
      - Log INFO `"score_single_opportunity.completed"` with `company_count=len(scores)`.
      - Return `{"opportunity_id": opportunity_id, "status": "scored", "company_count": len(scores)}`.
    - **Except path** (any exception, including `AIGatewayUnavailableError`):
      - Log ERROR `"score_single_opportunity.failed"` with `error=str(exc)`.
      - Open a new sync session; create and add `EnrichmentQueueItem(opportunity_id=UUID(opportunity_id), enrichment_type="relevance_scoring", status="pending", attempts=0)`; commit.
      - Return `{"opportunity_id": opportunity_id, "status": "failed", "error": str(exc)}` — do NOT re-raise.

- [ ] Task 5: Register `pipeline_scoring` queue in Celery app routing (AC: 2)
  - [ ] 5.1 Edit `services/data-pipeline/src/data_pipeline/workers/celery_app.py`: add `pipeline_scoring` to `task_routes` so scoring tasks are routed correctly:
    ```python
    app.conf.task_routes = {
        "pipeline.crawl_*": {"queue": "pipeline_crawl"},
        "pipeline.score_*": {"queue": "pipeline_scoring"},
        "pipeline._score_*": {"queue": "pipeline_scoring"},
        "pipeline.cleanup_*": {"queue": "pipeline_crawl"},
    }
    ```
    Add `pipeline_scoring` to the `app.conf.task_queues` list if it is explicitly declared. Document in a comment that the `pipeline_scoring` queue worker should be started with `--concurrency=$SCORING_CONCURRENCY` (default 5) to honour the configured limit.
  - [ ] 5.2 Add `score_opportunities` and `_score_single_opportunity` to the Celery autodiscovery imports in `celery_app.py` (or ensure they are auto-discovered by the existing `autodiscover_tasks` call).

- [ ] Task 6: Write unit tests (AC: 1, 2, 3, 4, 7, 8)
  - [ ] 6.1 Create `services/data-pipeline/tests/unit/test_score_opportunities.py`:
    - **E05-P2-008** `test_relevance_scores_keyed_by_company_id` — call `_score_single_opportunity` in eager mode with a mocked AI Gateway response returning `{"score": 0.87}`; assert `relevance_scores` JSONB stored in DB has structure `{str(company_id): 0.87, ...}` (dict keyed by company UUID string), not a flat list or nested object.
    - `test_score_opportunities_skips_when_no_opportunity_ids` — call `score_opportunities` with `crawl_result={"run_id": "some-id", "opportunity_ids": []}`; assert return dict has `dispatched=0` and no sub-tasks are dispatched (verify `group.apply_async` not called, or verify 0 `_score_single_opportunity` tasks queued).
    - `test_score_opportunities_dispatches_group_for_all_ids` — call `score_opportunities` with `crawl_result={"opportunity_ids": ["id-1", "id-2", "id-3"]}`; assert `dispatched=3` in return dict; verify the Celery group contained 3 signatures (mock `group` and assert call args).
    - `test_score_single_opportunity_creates_enrichment_queue_entry_on_failure` — mock `get_client().call_agent` to raise `AIGatewayUnavailableError`; call `_score_single_opportunity("some-uuid")` via `.apply()`; assert `EnrichmentQueueItem` exists with `enrichment_type='relevance_scoring'`, `status='pending'`; assert task returned `{"status": "failed", ...}` without raising.
    - `test_score_single_opportunity_skips_missing_opportunity` — call with a UUID that doesn't exist in DB; assert return dict has `status='not_found'`; assert no AI Gateway call made; assert no `EnrichmentQueueItem` created.

- [ ] Task 7: Write integration tests (AC: 1, 3, 4, 7)
  - [ ] 7.1 Create `services/data-pipeline/tests/integration/test_score_opportunities.py`. Use the same session-scoped `pg_container` and `eager_celery`/`eager_celery_no_propagate` fixtures from `conftest.py`. Add helpers: `_seed_opportunities(n)` to insert N opportunities directly via psycopg2, `_seed_company_profiles(n)` to insert N active company profiles, `_get_relevance_scores(pg_container, opportunity_id)` to read `relevance_scores` JSONB for one opportunity, `_enrichment_queue_count(pg_container)` to count `enrichment_queue` rows.
    - **E05-P1-018** `test_score_opportunities_parallelism` — seed 10 opportunities and 3 active company profiles; mock AI Gateway (respx) returning `{"score": 0.75}` for all calls; call `score_opportunities.apply(args=[{"opportunity_ids": [str(id) for id in opp_ids], "run_id": "test-run"}])`; assert all 10 opportunities have non-null `relevance_scores` JSONB with 3 keys (one per company); assert total AI Gateway calls = 30 (10 opportunities × 3 companies); assert return dict has `dispatched=10`.
    - **E05-P1-019** `test_single_opportunity_failure_does_not_block_batch` — seed 10 opportunities and 2 active company profiles; mock AI Gateway to return success for opportunities 1–8 and raise `AIGatewayUnavailableError` for opportunities 9 and 10 (use `respx` side_effect conditional on request URL/body); call `score_opportunities.apply(...)`; assert 8 opportunities have `relevance_scores` populated; assert 2 opportunities have `relevance_scores = NULL`; assert `enrichment_queue` has 2 pending entries; assert no exception raised from `score_opportunities`.
    - **E05-P1-020** `test_failed_scoring_queued_in_enrichment_queue` — seed 1 opportunity and 1 active company profile; mock AI Gateway to raise `AIGatewayUnavailableError`; call `_score_single_opportunity.apply(args=[str(opp_id)])`; assert `enrichment_queue` contains exactly 1 row with `enrichment_type='relevance_scoring'`, `status='pending'`, `attempts=0`, and `opportunity_id` matching the seeded opportunity.
    - **E05-P2-009** `test_partial_failure_batch_3_of_5` — seed 5 opportunities (IDs indexed 1–5) and 2 active company profiles; mock AI Gateway to raise `AIGatewayUnavailableError` for opportunities with index 2 and 4, succeed for 1, 3, 5; call `score_opportunities.apply(...)`; assert `scored_count` (opportunities with non-null `relevance_scores`) = 3; assert `enrichment_queue` row count = 2; assert none of the 5 sub-task calls propagated an exception.

## Dev Notes

### Architecture Context

S05.07 is the first **post-processing task** in the pipeline chain — it sits between the crawl/normalize tasks (S05.04–S05.06) and the guide generation + event publishing tasks (S05.08–S05.09). Its output (`relevance_scores` JSONB) is the primary input for opportunity ranking in the discovery UI (a later epic) and the analytics dashboards (E12).

**Pipeline execution chain after S05.07:**

```
Beat → crawl_aop (pipeline_crawl queue)
           ├── calls AI Gateway × 2 (crawler + normalization agents)
           ├── upserts pipeline.opportunities
           ├── updates crawler_runs to completed
           └── calls score_opportunities.apply_async([result])
                   └── dispatches group of _score_single_opportunity tasks
                           │  (pipeline_scoring queue, concurrency=SCORING_CONCURRENCY)
                           ├── _score_single_opportunity("opp-id-1")
                           │     └── calls relevance-scoring-agent × N_companies
                           │     └── writes opportunity.relevance_scores JSONB
                           ├── _score_single_opportunity("opp-id-2")
                           │     └── (failure) → enrichment_queue entry
                           └── ...
```

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.07]

### CRITICAL: Two-Queue Separation (E05-R-005 Mitigation)

The epic test design identifies Celery chord starvation (E05-R-005, score 4) as a risk: scoring tasks on the default queue can saturate worker slots and prevent Beat from dispatching the next crawl. The mitigation is **queue isolation**:

- `pipeline_crawl` queue: `crawl_aop`, `crawl_ted`, `crawl_eu_grants`, `cleanup_expired_opportunities`
- `pipeline_scoring` queue: `score_opportunities`, `_score_single_opportunity`

Deploy the scoring worker separately with:
```bash
celery -A data_pipeline.workers.celery_app worker \
  --queues=pipeline_scoring \
  --concurrency=$SCORING_CONCURRENCY  # default 5
```

The crawl worker must only consume from `pipeline_crawl` so it is never blocked by scoring tasks:
```bash
celery -A data_pipeline.workers.celery_app worker \
  --queues=pipeline_crawl \
  --concurrency=4
```

[Source: test-design-epic-05.md#E05-R-005 — Celery chord starvation (Score 4)]

### CRITICAL: `_score_single_opportunity` Must Never Re-raise (AC: 3)

The Celery group pattern does not automatically suppress individual sub-task failures — a task that raises propagates to the chord callback (if any) or surfaces as a failed result. Since `score_opportunities` dispatches a fire-and-forget `group` (no chord callback), each sub-task must explicitly catch all exceptions, enqueue to `enrichment_queue`, and return a failure dict:

```python
except Exception as exc:
    logger.error("score_single_opportunity.failed", opportunity_id=opportunity_id, error=str(exc))
    with get_sync_session() as session:
        session.add(EnrichmentQueueItem(
            opportunity_id=UUID(opportunity_id),
            enrichment_type="relevance_scoring",
            status="pending",
            attempts=0,
        ))
        session.commit()
    return {"opportunity_id": opportunity_id, "status": "failed", "error": str(exc)}
    # ← return, never raise
```

This matches the epic AC: "If scoring fails for a single opportunity, log the error and continue — do not fail the entire batch."

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.07]
[Source: test-design-epic-05.md#E05-P1-019]

### `upsert_opportunities` Return Signature Change

The `RETURNING id` clause is added to the existing PostgreSQL `INSERT ... ON CONFLICT DO UPDATE` statement. The SQLAlchemy pattern:

```python
stmt = insert(Opportunity).values(records)
stmt = stmt.on_conflict_do_update(
    index_elements=["source_id", "source_type"],
    set_={...},
).returning(Opportunity.id)

result = session.execute(stmt)
opportunity_ids = [str(row.id) for row in result]
```

The `RETURNING` clause returns IDs for **all rows** that were either inserted or updated. This is the full set needed by `score_opportunities` and S05.09 (event publishing).

**Updated return signature:**
```python
def upsert_opportunities(
    session: Session,
    records: list[dict[str, Any]],
    source_type: str,
) -> tuple[int, int, list[str]]:
    ...
    return new_count, updated_count, opportunity_ids
```

All three crawl tasks (`crawl_aop.py`, `crawl_ted.py`, `crawl_eu_grants.py`) must be updated to unpack the 3-tuple.

### Company Profile Access Pattern

The Relevance Scoring Agent needs company profile data. The pipeline service reads active company profiles directly from the shared PostgreSQL database. The `CompanyProfile` model is provided by the `eusolicit-models` shared package (built in S01.07).

```python
from eusolicit_models.company import CompanyProfile

with get_sync_session() as session:
    companies = session.execute(
        select(CompanyProfile).where(CompanyProfile.is_active.is_(True))
    ).scalars().all()
```

If `eusolicit-models` does not export `CompanyProfile`, use a lightweight read model in `data_pipeline/models/company_profile.py`:

```python
class CompanyProfile(Base):
    __tablename__ = "company_profiles"
    __table_args__ = {"schema": "auth"}

    id: Mapped[UUID] = mapped_column(pg.UUID(as_uuid=True), primary_key=True)
    is_active: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default="true")
    profile_data: Mapped[dict | None] = mapped_column(pg.JSONB)
    # Add other fields as needed for the scoring payload
```

The `to_scoring_dict()` helper on `CompanyProfile` builds the agent payload:
```python
def to_scoring_dict(self) -> dict[str, Any]:
    return {
        "id": str(self.id),
        "profile": self.profile_data or {},
    }
```

### Relevance Scoring Agent Call Shape

```python
response = client.call_agent(
    "relevance-scoring-agent",
    payload={
        "opportunity": {
            "id": str(opp.id),
            "source_type": opp.source_type,
            "title": opp.title,
            "description": opp.description,
            "cpv_codes": opp.cpv_codes,
            "country": opp.country,
            "region": opp.region,
            "deadline": opp.deadline.isoformat() if opp.deadline else None,
            "budget_min": str(opp.budget_min) if opp.budget_min is not None else None,
            "budget_max": str(opp.budget_max) if opp.budget_max is not None else None,
            "evaluation_criteria": opp.evaluation_criteria,
        },
        "company_profile": company.to_scoring_dict(),
    },
    agent_type=AgentType.SCORING,
    correlation_id=correlation_id,
)
score = response.output.get("score")  # float 0.0–1.0
```

Expected agent response:
```json
{
  "execution_id": "ex-score-1",
  "status": "completed",
  "output": {
    "score": 0.87,
    "reasoning": "High match on CPV codes and sector alignment.",
    "matched_criteria": ["sector", "cpv_codes", "budget_range"]
  }
}
```

The `score` value (float 0.0–1.0) is stored in `relevance_scores` keyed by `company_id`. The full `output` dict may be stored if richer data is desired, but the AC only requires the score value:
```json
{
  "3fa85f64-5717-4562-b3fc-2c963f66afa6": 0.87,
  "7c9e6679-7425-40de-944b-e07fc1f90ae7": 0.34
}
```

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.07]

### `EnrichmentQueueItem` Model (from S05.01)

```python
class EnrichmentQueueItem(Base):
    __tablename__ = "enrichment_queue"
    __table_args__ = (
        Index("ix_enrichment_queue_status_created", "status", "created_at"),
        {"schema": "pipeline"},
    )

    id: Mapped[UUID] = mapped_column(pg.UUID(as_uuid=True), primary_key=True, default=uuid4)
    opportunity_id: Mapped[UUID] = mapped_column(
        pg.UUID(as_uuid=True),
        ForeignKey("pipeline.opportunities.id"),
        nullable=False,
    )
    enrichment_type: Mapped[str] = mapped_column(String(30), nullable=False)
    # values: 'relevance_scoring' | 'submission_guide'
    status: Mapped[str] = mapped_column(String(20), nullable=False, server_default="pending")
    # values: 'pending' | 'processing' | 'failed'
    attempts: Mapped[int] = mapped_column(Integer, nullable=False, server_default="0")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
```

[Source: eusolicit-docs/implementation-artifacts/5-1-pipeline-schema-migration-and-model-layer.md#enrichment_queue]

### Mock Fixture Design for Integration Tests

```python
# In test_score_opportunities.py
import respx
import httpx

AI_GW_BASE = "http://ai-gateway:8004"
SCORING_URL = f"{AI_GW_BASE}/workflows/relevance-scoring-agent/run"

def _make_score_response(score: float = 0.75) -> dict:
    return {
        "execution_id": f"ex-score-{uuid4()}",
        "status": "completed",
        "output": {"score": score, "reasoning": "Test mock"},
    }

# For E05-P1-018 (all succeed):
with respx.mock(base_url=AI_GW_BASE, assert_all_called=False) as mock:
    mock.post("/workflows/relevance-scoring-agent/run").mock(
        return_value=httpx.Response(200, json=_make_score_response(0.75))
    )
    result = score_opportunities.apply(args=[{
        "run_id": str(uuid4()),
        "opportunity_ids": [str(opp_id) for opp_id in seeded_opp_ids],
    }])

assert result.successful()
assert result.result["dispatched"] == 10

# Verify JSONB populated for all 10:
for opp_id in seeded_opp_ids:
    scores = _get_relevance_scores(pg_container, opp_id)
    assert scores is not None
    assert len(scores) == len(seeded_company_ids)  # 3 companies
    assert all(isinstance(v, float) for v in scores.values())

# For E05-P2-009 (partial failure — 2 of 5 fail):
failure_ids = {str(opp_ids[1]), str(opp_ids[3])}  # index 1 and 3 (0-based → opps 2 and 4)

def _selective_error(request):
    body = json.loads(request.content)
    opp_id = body.get("input", {}).get("opportunity", {}).get("id")
    if opp_id in failure_ids:
        return httpx.Response(503)
    return httpx.Response(200, json=_make_score_response(0.75))

with respx.mock(base_url=AI_GW_BASE, assert_all_called=False) as mock:
    mock.post("/workflows/relevance-scoring-agent/run").mock(side_effect=_selective_error)
    result = score_opportunities.apply(args=[{
        "run_id": str(uuid4()),
        "opportunity_ids": [str(id) for id in opp_ids],
    }])

assert result.successful()
assert _enrichment_queue_count(pg_container) == 2
scored = sum(1 for oid in opp_ids if _get_relevance_scores(pg_container, oid) is not None)
assert scored == 3
```

### `respx` Mock for Selective Failure

The selective-failure pattern requires `side_effect` on the respx mock route. In `respx >= 0.21`, a callable side_effect receives the `httpx.Request` object:

```python
import json

def _selective_failure(request: httpx.Request) -> httpx.Response:
    body = json.loads(request.content)
    opp_id = body.get("input", {}).get("opportunity", {}).get("id", "")
    if opp_id in failing_ids:
        return httpx.Response(503)  # triggers AIGatewayUnavailableError via retry exhaust
    return httpx.Response(200, json=_make_score_response())

mock.post("/workflows/relevance-scoring-agent/run").mock(side_effect=_selective_failure)
```

Note: a single 503 won't exhaust retries — the `ai_gateway_client` will retry 3 times. Use `respx` with `side_effect` that always returns 503 for the failing IDs so all 3 retries fail, triggering `AIGatewayUnavailableError`. Alternatively, mock the client directly:

```python
# Simpler approach: mock at the ai_gateway_client level
from unittest.mock import patch, MagicMock

def _mock_call_agent(agent_name, payload, *, agent_type, correlation_id=None):
    opp_id = payload.get("opportunity", {}).get("id", "")
    if opp_id in failing_ids:
        raise AIGatewayUnavailableError(agent_name=agent_name, attempts=3, last_status_code=503)
    mock_resp = MagicMock()
    mock_resp.output = {"score": 0.75}
    return mock_resp

with patch("data_pipeline.workers.tasks.score_opportunities.get_client") as mock_client:
    mock_client.return_value.call_agent.side_effect = _mock_call_agent
    result = score_opportunities.apply(...)
```

This is the recommended approach for unit tests (Task 6). Integration tests (Task 7) should use `respx` for realism.

### File Locations

| Purpose | Path |
|---------|------|
| New scoring task module | `services/data-pipeline/src/data_pipeline/workers/tasks/score_opportunities.py` |
| Upsert helper (modify return type) | `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py` |
| AOP crawler (modify return + chain) | `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` |
| TED crawler (modify return + chain) | `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` |
| EU Grants crawler (modify return + chain) | `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` |
| Celery app (add queue routing) | `services/data-pipeline/src/data_pipeline/workers/celery_app.py` |
| Opportunity model (read `relevance_scores`) | `services/data-pipeline/src/data_pipeline/models/opportunity.py` |
| EnrichmentQueueItem model (insert on failure) | `services/data-pipeline/src/data_pipeline/models/enrichment_queue.py` |
| Company profile model (read, or from eusolicit-models) | `services/data-pipeline/src/data_pipeline/models/company_profile.py` (create if not in eusolicit-models) |
| AI Gateway client (reuse) | `services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py` |
| Sync DB session (reuse) | `services/data-pipeline/src/data_pipeline/db.py` |
| Unit tests | `services/data-pipeline/tests/unit/test_score_opportunities.py` |
| Integration tests | `services/data-pipeline/tests/integration/test_score_opportunities.py` |
| conftest fixtures (reuse `pg_container`, `eager_celery`) | `services/data-pipeline/tests/conftest.py` |

### Test Design Traceability

Tests in this story directly verify the following scenarios from the Epic 5 test design:

| Test ID | Priority | Scenario |
|---------|----------|----------|
| E05-P1-018 | P1 | Relevance scoring parallelism — 10 opps × 3 companies, all scored, `relevance_scores` JSONB populated for all 10 |
| E05-P1-019 | P1 | Single-opportunity scoring failure does not block batch — 8 of 10 succeed, 2 fail without blocking |
| E05-P1-020 | P1 | Failed scoring queued in `enrichment_queue` with `enrichment_type = 'relevance_scoring'` |
| E05-P2-008 | P2 | `relevance_scores` JSONB structure is `{company_id: score}` dict, not a flat list |
| E05-P2-009 | P2 | Partial failure: 3 of 5 succeed, 2 fail → `enrichment_queue` entries = 2 |

Mitigations verified:

- **E05-R-005** (Celery chord starvation, Score 4): mitigated by `pipeline_scoring` queue separation (Task 5); crawl tasks continue unblocked on `pipeline_crawl` queue
- **E05-R-007** (Enrichment queue unbounded failure, Score 4): S05.07 creates `enrichment_queue` entries on scoring failure with `status='pending'`; S05.11 (enrichment queue worker) handles the 3-retry-then-permanent-failure logic; E05-P1-020 and E05-P2-009 verify queue entry creation

Deferred mitigations (handled by other stories):

- **E05-R-001** (Dedup race): S05.01/S05.04 own the upsert atomicity; S05.07 only reads the already-upserted rows
- **E05-R-002** (AI Gateway cascade / orphaned runs): scoring tasks are fully independent of `crawler_runs`; Gateway failure → `enrichment_queue` entry (not orphaned run)
- **E05-R-003** (Redis publish loss): S05.09 owns event publishing; S05.07 only enriches `relevance_scores`

[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md]

---

## Senior Developer Review

### Review Findings

- [x] [Review][Defer] Silent data loss when enrichment queue DB write also fails [score_opportunities.py:244-259] — deferred, pre-existing design tradeoff from S05.01 enrichment_queue pattern; next crawl cycle re-processes
- [x] [Review][Defer] No deduplication guard on enrichment_queue entries (missing unique constraint on opportunity_id + enrichment_type + status) — deferred, pre-existing; S05.11 enrichment queue worker should handle idempotent retries
- [x] [Review][Defer] Large batch scalability — group dispatch creates one Celery message per opportunity with no chunking — deferred, not a correctness issue at current scale; add batch chunking if crawl volumes exceed ~500/cycle
- [x] [Review][Defer] Per-company partial scoring discarded on failure — deferred, by design per AC1 requiring ALL profiles scored; re-computing on retry is correct behavior

**Review Summary:** 0 `decision-needed`, 0 `patch`, 4 `defer`, 2 dismissed.

All acceptance criteria (AC1–AC8) verified against implementation. All test design scenarios (E05-P1-018, E05-P1-019, E05-P1-020, E05-P2-008, E05-P2-009) have corresponding tests. Queue isolation (E05-R-005) and enrichment queue failure handling (E05-R-007) mitigations confirmed.

**Result: APPROVED — no blocking or actionable findings.**

---

## Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-16 | BMad Orchestrator (bmad-create-story) | Story file created from epic-05 S05.07 spec and test-design-epic-05.md |
| 2026-04-16 | BMad Code Review (bmad-code-review) | Senior developer review: APPROVED — 0 patches, 4 deferred (pre-existing) |
