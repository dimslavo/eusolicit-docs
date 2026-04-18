# Story 5.8: Submission Guide Generation Task

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer implementing the data pipeline**,
I want **a `generate_submission_guides` Celery task chained after `score_opportunities` that, for each opportunity ID in the crawl result set, dispatches individual `_generate_single_guide` sub-tasks (via Celery `group`, routed to a dedicated `pipeline_guides` queue) that each load the opportunity, check whether a current guide already exists (skipping re-generation when `opportunity.updated_at <= existing_guide.created_at`), call the Submission Guide Agent via `ai_gateway_client` with the opportunity data and `source_type` to generate portal-specific step-by-step submission instructions, persist the result in `pipeline.submission_guides` with `reviewed=false` and `generated_by` populated from the agent's `execution_id`, and on any failure create an `EnrichmentQueueItem` with `enrichment_type='submission_guide'` without propagating the exception**,
so that **every newly crawled or updated opportunity automatically receives portal-specific submission instructions stored in the database, unchanged opportunities are not wastefully re-processed, agent traceability is captured in `generated_by`, and transient failures are queued for automatic retry by S05.11 without interrupting guide generation for the rest of the batch**.

## Acceptance Criteria

1. `submission_guides` row created for each new opportunity with structured `steps` JSONB (list of step objects from the agent), `reviewed=false`, and `source_portal` set to the opportunity's `source_type`
2. Existing guides are **not** regenerated when the opportunity data is unchanged: if a `SubmissionGuide` row already exists for an `opportunity_id` AND `opportunity.updated_at <= existing_guide.created_at`, the opportunity is skipped with no AI Gateway call made; skipped opportunities are logged at DEBUG level with `reason="guide_current"`
3. Existing guides **are** regenerated when the underlying opportunity has been updated: if `opportunity.updated_at > existing_guide.created_at`, the agent is called and the existing `SubmissionGuide` row is updated in-place (`steps`, `generated_by`, `reviewed=False`, `updated_at`); no duplicate guide row is created
4. `generated_by` field on each `SubmissionGuide` row is populated with the agent's `execution_id` returned in the AI Gateway response; fallback to `f"submission-guide-agent:{correlation_id}"` when `execution_id` is `None`
5. A single-opportunity guide generation failure (AI Gateway error or any exception within `_generate_single_guide`) does not propagate to abort the group — the failing sub-task returns a failure dict without re-raising, and the remaining opportunities continue processing normally
6. When `_generate_single_guide` fails, an `EnrichmentQueueItem` record is created with `enrichment_type='submission_guide'`, `status='pending'`, `attempts=0`, and the correct `opportunity_id`; the item is retrievable by S05.11 for later retry
7. `generate_submission_guides` is automatically triggered after each successful `score_opportunities` dispatch: `score_opportunities` calls `generate_submission_guides.apply_async(args=[crawl_result])` at the end of its execution, passing the same crawl result dict (containing `opportunity_ids` and `run_id`)
8. All structured log lines emitted by guide generation tasks include `task_name`, `opportunity_id`, and `correlation_id` fields

## Tasks / Subtasks

- [x] Task 1: Chain `generate_submission_guides` from `score_opportunities` (AC: 7)
  - [x] 1.1 Edit `services/data-pipeline/src/data_pipeline/workers/tasks/score_opportunities.py`: after the `scoring_group.apply_async()` call and before the `return` statement, add:
    ```python
    generate_submission_guides.apply_async(args=[crawl_result])
    ```
    Import `generate_submission_guides` from `data_pipeline.workers.tasks.generate_submission_guides`. Add a DEBUG-level log line `"score_opportunities.guides_dispatched"` with `opportunity_count=len(opportunity_ids)`. Ensure the call is placed inside the main task body but outside any `on_failure` handler — it fires only on the success path.

- [x] Task 2: Implement `generate_submission_guides` orchestrator task (AC: 5, 7, 8)
  - [x] 2.1 Create `services/data-pipeline/src/data_pipeline/workers/tasks/generate_submission_guides.py`. Implement:
    ```python
    @shared_task(
        name="pipeline.generate_submission_guides",
        bind=True,
        queue="pipeline_guides",
    )
    def generate_submission_guides(self, crawl_result: dict[str, Any]) -> dict[str, Any]:
    ```
    Steps:
    - Extract `opportunity_ids: list[str] = crawl_result.get("opportunity_ids", [])`. If empty, log INFO `"generate_submission_guides.skipped"` with `reason="no_opportunity_ids"` and return `{"dispatched": 0, "run_id": crawl_result.get("run_id")}`.
    - Bind structlog context: `task_name="pipeline.generate_submission_guides"`, `opportunity_count=len(opportunity_ids)`, `run_id=crawl_result.get("run_id")`.
    - Build a Celery `group` of `_generate_single_guide.s(opp_id)` for each `opp_id` in `opportunity_ids`.
    - Call `guide_group.apply_async()` to dispatch asynchronously (fire-and-forget from the orchestrator's perspective).
    - Log INFO `"generate_submission_guides.dispatched"` with `dispatched=len(opportunity_ids)`.
    - Return `{"dispatched": len(opportunity_ids), "run_id": crawl_result.get("run_id")}`.

- [x] Task 3: Implement `_generate_single_guide` sub-task (AC: 1, 2, 3, 4, 5, 6, 8)
  - [x] 3.1 Add `_generate_single_guide` to `generate_submission_guides.py` as a sibling task:
    ```python
    @shared_task(
        name="pipeline._generate_single_guide",
        bind=True,
        queue="pipeline_guides",
    )
    def _generate_single_guide(self, opportunity_id: str) -> dict[str, Any]:
    ```
    Steps:
    - Generate `correlation_id = str(uuid4())`. Bind structlog: `task_name="pipeline._generate_single_guide"`, `opportunity_id=opportunity_id`, `correlation_id=correlation_id`.
    - Wrap the entire body in `try/except Exception as exc` to prevent failure propagation.
    - **Happy path**:
      - Open a sync session (`get_sync_session()`); load `opp = session.get(Opportunity, UUID(opportunity_id))`. If `None`, log WARNING `"generate_single_guide.opportunity_not_found"` and return `{"opportunity_id": opportunity_id, "status": "not_found"}`.
      - Check for existing guide:
        ```python
        existing_guide = session.execute(
            select(SubmissionGuide).where(
                SubmissionGuide.opportunity_id == UUID(opportunity_id)
            )
        ).scalar_one_or_none()
        ```
      - If `existing_guide` is not `None` **and** `opp.updated_at <= existing_guide.created_at`:
        - Log DEBUG `"generate_single_guide.skipped"` with `reason="guide_current"`, `guide_id=str(existing_guide.id)`.
        - Return `{"opportunity_id": opportunity_id, "status": "skipped", "reason": "guide_current"}`.
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
            "mandatory_documents": opp.mandatory_documents,
        }
        ```
      - Call Submission Guide Agent:
        ```python
        client = get_client()
        response = client.call_agent(
            "submission-guide-agent",
            payload={"opportunity": opp_payload, "source_type": opp.source_type},
            agent_type=AgentType.GUIDE,
            correlation_id=correlation_id,
        )
        ```
      - Extract `steps = response.output.get("steps", [])`.
      - Build `generated_by = response.execution_id or f"submission-guide-agent:{correlation_id}"`.
      - Persist in a **new** sync session (the original session may be stale after the agent call):
        - **Update path** (when `existing_guide` is not None, i.e., opportunity was updated):
          ```python
          with get_sync_session() as session:
              guide = session.get(SubmissionGuide, existing_guide.id)
              guide.steps = steps
              guide.generated_by = generated_by
              guide.reviewed = False
              session.commit()
          action = "updated"
          guide_id = str(existing_guide.id)
          ```
        - **Create path** (when `existing_guide` is None):
          ```python
          with get_sync_session() as session:
              guide = SubmissionGuide(
                  opportunity_id=UUID(opportunity_id),
                  source_portal=opp.source_type,
                  steps=steps,
                  generated_by=generated_by,
                  reviewed=False,
              )
              session.add(guide)
              session.commit()
          action = "created"
          guide_id = str(guide.id)
          ```
      - Log INFO `"generate_single_guide.completed"` with `action=action`, `guide_id=guide_id`.
      - Return `{"opportunity_id": opportunity_id, "status": action, "guide_id": guide_id}`.
    - **Except path** (any exception, including `AIGatewayUnavailableError`):
      - Log ERROR `"generate_single_guide.failed"` with `error=str(exc)`.
      - Open a new sync session; create and add:
        ```python
        EnrichmentQueueItem(
            opportunity_id=UUID(opportunity_id),
            enrichment_type="submission_guide",
            status="pending",
            attempts=0,
        )
        ```
        Commit. Return `{"opportunity_id": opportunity_id, "status": "failed", "error": str(exc)}` — do **NOT** re-raise.

- [x] Task 4: Register `pipeline_guides` queue in Celery app routing (AC: 5)
  - [x] 4.1 Edit `services/data-pipeline/src/data_pipeline/workers/celery_app.py`: extend `task_routes` to include:
    ```python
    "pipeline.generate_*": {"queue": "pipeline_guides"},
    "pipeline._generate_*": {"queue": "pipeline_guides"},
    ```
    Add `pipeline_guides` to `app.conf.task_queues` if explicitly declared. Document in a comment that the `pipeline_guides` queue worker should be started with `--queues=pipeline_guides --concurrency=4`.
  - [x] 4.2 Ensure `generate_submission_guides` and `_generate_single_guide` are included in Celery's autodiscovery (add to `autodiscover_tasks` module list in `celery_app.py` if needed, or confirm the task file is within an auto-discovered package).

- [x] Task 5: Add `AgentType.GUIDE` if absent (AC: 1)
  - [x] 5.1 Check `services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py` for the `AgentType` enum. If `GUIDE` is not defined, add:
    ```python
    GUIDE = "guide"
    ```
    This enum value selects the per-agent timeout/retry configuration. If a distinct timeout for guide agents is not yet configured, map it to the same timeout as `AgentType.NORMALIZATION` until configured separately.

- [x] Task 6: Write unit tests (AC: 1, 2, 3, 4, 5, 6, 8)
  - [x] 6.1 Create `services/data-pipeline/tests/unit/test_generate_submission_guides.py`:
    - `test_generate_submission_guides_skips_when_no_opportunity_ids` — call `generate_submission_guides` with `crawl_result={"run_id": "some-id", "opportunity_ids": []}`; assert return dict has `dispatched=0` and no sub-tasks dispatched (verify `group.apply_async` not called, or verify 0 `_generate_single_guide` tasks queued).
    - `test_generate_submission_guides_dispatches_group_for_all_ids` — call with `crawl_result={"opportunity_ids": ["id-1", "id-2", "id-3"]}`; assert `dispatched=3` in return dict; verify the Celery group contained 3 `_generate_single_guide` signatures (mock `group` and assert call args).
    - `test_generate_single_guide_creates_enrichment_queue_entry_on_failure` — seed 1 opportunity in DB; mock `get_client().call_agent` to raise `AIGatewayUnavailableError`; call `_generate_single_guide.apply(args=[str(opp_id)])`; assert `EnrichmentQueueItem` exists with `enrichment_type='submission_guide'`, `status='pending'`; assert task returned `{"status": "failed", ...}` without raising.
    - `test_generate_single_guide_skips_missing_opportunity` — call with a UUID that doesn't exist in DB; assert return dict has `status='not_found'`; assert no AI Gateway call made; assert no `EnrichmentQueueItem` created.
    - **E05-P2-010** `test_generate_single_guide_generated_by_populated` — seed 1 opportunity; mock agent response with `execution_id="ex-guide-001"`; call `_generate_single_guide.apply(args=[str(opp_id)])`; assert `SubmissionGuide.generated_by == "ex-guide-001"`.
    - `test_generate_single_guide_generated_by_fallback_when_execution_id_none` — mock agent response with `execution_id=None`; call `_generate_single_guide.apply(...)`; assert `SubmissionGuide.generated_by` starts with `"submission-guide-agent:"`.

- [x] Task 7: Write integration tests (AC: 1, 2, 3, 5, 6)
  - [x] 7.1 Create `services/data-pipeline/tests/integration/test_generate_submission_guides.py`. Use session-scoped `pg_container` and `eager_celery` fixtures from `conftest.py`. Add helpers: `_seed_opportunity(**kwargs)` to insert one opportunity directly via psycopg2 (accepts `updated_at` override), `_seed_guide(opportunity_id, created_at)` to insert a `submission_guides` row with a specific `created_at`, `_get_guide(pg_container, opportunity_id)` to read the `submission_guides` row for an opportunity (returns dict or None), `_enrichment_queue_count(pg_container)` to count `enrichment_queue` rows with `enrichment_type='submission_guide'`.
    - **E05-P1-021** `test_submission_guide_created_for_new_opportunity` — seed 1 opportunity with no existing guide; mock Submission Guide Agent response returning structured `steps` (`[{"step_number": 1, "title": "Register", "instruction": "..."}]`) and `execution_id="ex-guide-001"`; call `generate_submission_guides.apply(args=[{"opportunity_ids": [str(opp_id)], "run_id": str(uuid4())}])`; assert `_get_guide` returns a non-None row; assert `guide["reviewed"] is False`; assert `guide["steps"]` is a non-empty list; assert `guide["source_portal"] == opp.source_type`.
    - **E05-P1-022** `test_existing_guide_not_regenerated_when_opportunity_unchanged` — seed 1 opportunity with `updated_at = T`; seed an existing guide for that opportunity with `created_at = T` (i.e., `updated_at == created_at`, so `<=` condition is true); mock AI Gateway (assert no call made); call `generate_submission_guides.apply(...)`; assert guide row is unchanged (same `steps`, same `generated_by`); assert AI Gateway mock received 0 calls.
    - **E05-P1-022b** `test_existing_guide_regenerated_when_opportunity_updated` — seed 1 opportunity with `updated_at = T2`; seed an existing guide with `created_at = T1` where `T1 < T2`; mock agent returning new `steps` with different content; call `generate_submission_guides.apply(...)`; assert existing guide row is updated (new `steps` content, `reviewed=false`); assert no new guide row created (still exactly 1 guide row for the opportunity).
    - **E05-P1-023** `test_guide_generation_failure_queues_enrichment_entry` — seed 1 opportunity; mock Submission Guide Agent to raise `AIGatewayUnavailableError` (or use `patch` on `get_client().call_agent`); call `_generate_single_guide.apply(args=[str(opp_id)])`; assert `_enrichment_queue_count` == 1; assert queue entry has `enrichment_type='submission_guide'`, `status='pending'`, `attempts=0`, `opportunity_id` matching seeded opportunity; assert no `submission_guides` row exists for the opportunity.
    - `test_guide_generation_failure_does_not_block_batch` — seed 5 opportunities; mock agent to fail for opportunity at index 2 and succeed for the other 4; call `generate_submission_guides.apply(args=[{"opportunity_ids": [str(id) for id in opp_ids], "run_id": str(uuid4())}])`; assert exactly 4 `submission_guides` rows exist; assert `_enrichment_queue_count` == 1; assert no exception propagated from the orchestrator task.

## Dev Notes

### Architecture Context

S05.08 is the second **post-processing task** in the pipeline enrichment chain, sitting between `score_opportunities` (S05.07) and Redis event publishing (S05.09). Guide generation is independent of relevance scoring — it reads raw opportunity data and `source_type` from the database and does not consume `relevance_scores` JSONB. Scoring and guide generation therefore run **concurrently**: both are dispatched as fire-and-forget tasks by `score_opportunities`.

**Pipeline execution chain after S05.08:**

```
Beat → crawl_{source} (pipeline_crawl queue)
           ├── calls AI Gateway × 2 (crawler + normalization agents)
           ├── upserts pipeline.opportunities
           ├── updates crawler_runs to completed
           └── calls score_opportunities.apply_async([result])
                   ├── dispatches group(_score_single_opportunity) → pipeline_scoring queue
                   │       └── writes opportunity.relevance_scores JSONB
                   └── calls generate_submission_guides.apply_async([crawl_result])
                           └── dispatches group(_generate_single_guide) → pipeline_guides queue
                                   ├── _generate_single_guide("opp-id-1")
                                   │     └── calls submission-guide-agent
                                   │     └── creates/updates pipeline.submission_guides
                                   ├── _generate_single_guide("opp-id-2")
                                   │     └── (skip: guide_current)
                                   └── _generate_single_guide("opp-id-3")
                                         └── (failure) → enrichment_queue entry
```

S05.09 (Redis Streams event publishing) will be chained from `generate_submission_guides` — the event is published after guide generation is dispatched, since scoring and guide generation are concurrent enrichment steps rather than strictly sequential.

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.08]

### CRITICAL: Three-Queue Separation (Extends E05-R-005 Mitigation)

S05.08 adds a third dedicated queue, keeping crawl, scoring, and guide tasks fully isolated:

| Queue | Tasks | Deploy command |
|-------|-------|----------------|
| `pipeline_crawl` | `crawl_aop`, `crawl_ted`, `crawl_eu_grants`, `cleanup_expired_opportunities` | `celery worker --queues=pipeline_crawl --concurrency=4` |
| `pipeline_scoring` | `score_opportunities`, `_score_single_opportunity` | `celery worker --queues=pipeline_scoring --concurrency=$SCORING_CONCURRENCY` |
| `pipeline_guides` | `generate_submission_guides`, `_generate_single_guide` | `celery worker --queues=pipeline_guides --concurrency=4` |

No shared queue between crawl/scoring/guide workers — a large scoring chord cannot block guide generation, and vice versa.

[Source: test-design-epic-05.md#E05-R-005 — Celery chord starvation (Score 4)]

### CRITICAL: `_generate_single_guide` Must Never Re-raise (AC: 5)

Identical pattern to `_score_single_opportunity` from S05.07 — the sub-task must catch all exceptions, write to `enrichment_queue`, and return a failure dict:

```python
except Exception as exc:
    logger.error(
        "generate_single_guide.failed",
        opportunity_id=opportunity_id,
        error=str(exc),
    )
    with get_sync_session() as session:
        session.add(EnrichmentQueueItem(
            opportunity_id=UUID(opportunity_id),
            enrichment_type="submission_guide",
            status="pending",
            attempts=0,
        ))
        session.commit()
    return {"opportunity_id": opportunity_id, "status": "failed", "error": str(exc)}
    # ← return, never raise
```

This matches the epic AC: "Guide generation failure queues the opportunity in `enrichment_queue` with `enrichment_type='submission_guide'`."

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.08]
[Source: test-design-epic-05.md#E05-P1-023]

### Guide Freshness Check Logic (AC: 2, 3)

The skip/regenerate decision uses `<=` (not `<`) for timestamp comparison:

```python
existing_guide = session.execute(
    select(SubmissionGuide).where(
        SubmissionGuide.opportunity_id == UUID(opportunity_id)
    )
).scalar_one_or_none()

if existing_guide is not None and opp.updated_at <= existing_guide.created_at:
    # Guide is current — opportunity has not changed since guide was generated
    logger.debug(
        "generate_single_guide.skipped",
        reason="guide_current",
        guide_id=str(existing_guide.id),
    )
    return {"opportunity_id": opportunity_id, "status": "skipped", "reason": "guide_current"}
```

**Why `<=`**: If `updated_at == created_at` (the opportunity was crawled and the guide was generated in the same cycle without subsequent modification), the guide is still considered current. Using `<` would incorrectly trigger regeneration.

**No duplicate rows**: when `existing_guide` is not None but the freshness check fails (opportunity updated), the existing row is **updated in place** — `session.get(SubmissionGuide, existing_guide.id)` loads it, then `guide.steps`, `guide.generated_by`, and `guide.reviewed` are set and committed. This guarantees at most one `submission_guides` row per opportunity.

### Submission Guide Agent Call Shape

```python
response = client.call_agent(
    "submission-guide-agent",
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
            "mandatory_documents": opp.mandatory_documents,
        },
        "source_type": opp.source_type,
    },
    agent_type=AgentType.GUIDE,
    correlation_id=correlation_id,
)
```

Expected agent response:
```json
{
  "execution_id": "ex-guide-001",
  "status": "completed",
  "output": {
    "steps": [
      {
        "step_number": 1,
        "title": "Register on the portal",
        "instruction": "Navigate to the AOP portal and create an account using your company credentials.",
        "portal_url": "https://aop.example.com/register",
        "form_reference": "Form AOP-1A",
        "tips": ["Use a company email address", "Have your NACE code ready"]
      },
      {
        "step_number": 2,
        "title": "Submit Expression of Interest",
        "instruction": "Complete and upload the EoI template before the deadline.",
        "portal_url": "https://aop.example.com/eoi",
        "form_reference": null,
        "tips": []
      }
    ],
    "portal": "aop",
    "version": "1.0"
  }
}
```

`generated_by` population:
```python
generated_by = response.execution_id or f"submission-guide-agent:{correlation_id}"
```

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.08]

### `SubmissionGuide` Model (from S05.01)

```python
class SubmissionGuide(Base):
    __tablename__ = "submission_guides"
    __table_args__ = (
        Index("ix_submission_guide_opportunity_id", "opportunity_id"),
        {"schema": SCHEMA},
    )

    id: Mapped[UUID] = mapped_column(pg.UUID(as_uuid=True), primary_key=True, default=uuid4)
    opportunity_id: Mapped[UUID] = mapped_column(
        pg.UUID(as_uuid=True),
        ForeignKey(f"{SCHEMA}.opportunities.id", ondelete="CASCADE"),
        nullable=False,
    )
    source_portal: Mapped[str] = mapped_column(String(20), nullable=False)  # aop / ted / eu_grants
    steps: Mapped[dict] = mapped_column(pg.JSONB, nullable=False)
    # steps: ordered list of {step_number, title, instruction, portal_url, form_reference, tips}
    generated_by: Mapped[str | None] = mapped_column(Text)  # agent execution_id / version string
    reviewed: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default="false")
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    opportunity: Mapped["Opportunity"] = relationship(back_populates="submission_guides")
```

[Source: eusolicit-docs/implementation-artifacts/5-1-pipeline-schema-migration-and-model-layer.md#SubmissionGuide]

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
    # 'relevance_scoring' | 'submission_guide'
    status: Mapped[str] = mapped_column(String(20), nullable=False, server_default="pending")
    # 'pending' | 'processing' | 'done' | 'failed'
    attempts: Mapped[int] = mapped_column(Integer, nullable=False, server_default="0")
    last_error: Mapped[str | None] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
```

[Source: eusolicit-docs/implementation-artifacts/5-1-pipeline-schema-migration-and-model-layer.md#enrichment_queue]

### Mock Fixture Design for Integration Tests

```python
# In test_generate_submission_guides.py
import respx
import httpx
from uuid import uuid4

AI_GW_BASE = "http://ai-gateway:8004"

def _make_guide_response(steps: list | None = None, execution_id: str | None = None) -> dict:
    return {
        "execution_id": execution_id or f"ex-guide-{uuid4()}",
        "status": "completed",
        "output": {
            "steps": steps or [
                {
                    "step_number": 1,
                    "title": "Register on portal",
                    "instruction": "Create an account on the portal.",
                    "portal_url": "https://portal.example.com/register",
                    "form_reference": None,
                    "tips": [],
                }
            ],
            "portal": "aop",
            "version": "1.0",
        },
    }

# E05-P1-021: New opportunity, happy path
with respx.mock(base_url=AI_GW_BASE, assert_all_called=False) as mock:
    mock.post("/workflows/submission-guide-agent/run").mock(
        return_value=httpx.Response(200, json=_make_guide_response())
    )
    result = generate_submission_guides.apply(args=[{
        "run_id": str(uuid4()),
        "opportunity_ids": [str(opp_id)],
    }])

assert result.successful()
guide = _get_guide(pg_container, opp_id)
assert guide is not None
assert guide["reviewed"] is False
assert isinstance(guide["steps"], list) and len(guide["steps"]) > 0

# E05-P1-022: Existing guide, opportunity not updated — zero AI Gateway calls
with respx.mock(base_url=AI_GW_BASE, assert_all_called=False) as mock:
    route = mock.post("/workflows/submission-guide-agent/run").mock(
        return_value=httpx.Response(200, json=_make_guide_response())
    )
    result = generate_submission_guides.apply(...)

assert route.call_count == 0  # No AI Gateway call for current guide

# E05-P1-023: Failure → enrichment_queue (simpler mock at client level)
from unittest.mock import patch

with patch("data_pipeline.workers.tasks.generate_submission_guides.get_client") as mock_client:
    mock_client.return_value.call_agent.side_effect = AIGatewayUnavailableError(
        agent_name="submission-guide-agent", attempts=3, last_status_code=503
    )
    result = _generate_single_guide.apply(args=[str(opp_id)])

assert result.successful()  # task returns failure dict, does not raise
assert result.result["status"] == "failed"
assert _enrichment_queue_count(pg_container) == 1
```

### File Locations

| Purpose | Path |
|---------|------|
| New guide generation task module | `services/data-pipeline/src/data_pipeline/workers/tasks/generate_submission_guides.py` |
| Score opportunities (add chain call to Task 1) | `services/data-pipeline/src/data_pipeline/workers/tasks/score_opportunities.py` |
| Celery app (add `pipeline_guides` queue routing) | `services/data-pipeline/src/data_pipeline/workers/celery_app.py` |
| AI Gateway client (add `AgentType.GUIDE` if absent) | `services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py` |
| SubmissionGuide model (read/write) | `services/data-pipeline/src/data_pipeline/models/submission_guide.py` |
| EnrichmentQueueItem model (insert on failure) | `services/data-pipeline/src/data_pipeline/models/enrichment_queue.py` |
| Opportunity model (read `updated_at`, `source_type`) | `services/data-pipeline/src/data_pipeline/models/opportunity.py` |
| Sync DB session (reuse) | `services/data-pipeline/src/data_pipeline/db.py` |
| Unit tests | `services/data-pipeline/tests/unit/test_generate_submission_guides.py` |
| Integration tests | `services/data-pipeline/tests/integration/test_generate_submission_guides.py` |
| conftest fixtures (reuse `pg_container`, `eager_celery`) | `services/data-pipeline/tests/conftest.py` |

### Test Design Traceability

Tests in this story directly verify the following scenarios from the Epic 5 test design:

| Test ID | Priority | Scenario |
|---------|----------|----------|
| E05-P1-021 | P1 | Submission guide created for new opportunity — mock agent response with structured `steps`; verify `pipeline.submission_guides` row created, `reviewed=false`, `steps` JSONB populated |
| E05-P1-022 | P1 | Existing guide not regenerated unless opportunity updated — two runs with unchanged opportunity; assert 1 guide row and 0 AI Gateway calls on second run |
| E05-P1-023 | P1 | Guide generation failure queues opportunity in `enrichment_queue` with `enrichment_type='submission_guide'` |
| E05-P2-010 | P2 | `generated_by` field records agent `execution_id` returned from AI Gateway response |

Mitigations verified:

- **E05-R-005** (Celery chord starvation, Score 4): `pipeline_guides` queue isolation (Task 4) extends the two-queue mitigation from S05.07 to a full three-queue separation — crawl, scoring, and guide workers cannot starve each other
- **E05-R-007** (Enrichment queue unbounded failure, Score 4): S05.08 creates `enrichment_queue` entries on guide generation failure with `enrichment_type='submission_guide'`; E05-P1-023 verifies the queue entry; S05.11 owns the 3-retry-then-permanent-failure logic

Deferred mitigations (handled by other stories):

- **E05-R-001** (Dedup race): S05.01/S05.04–06 own the upsert atomicity; S05.08 only reads already-upserted rows
- **E05-R-002** (AI Gateway cascade / orphaned runs): guide tasks are independent of `crawler_runs`; Gateway failure → `enrichment_queue` entry, not an orphaned run
- **E05-R-003** (Redis publish loss): S05.09 owns event publishing; S05.08 only creates/updates `submission_guides` rows

[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md]

---

## Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-16 | BMad Orchestrator (bmad-create-story) | Story file created from epic-05 S05.08 spec and test-design-epic-05.md |
| 2026-04-17 | bmad-dev-story | Implemented: generate_submission_guides.py (Tasks 2+3), score_opportunities.py chain (Task 1), celery_app.py pipeline_guides routing (Task 4). Used AgentType.SUBMISSION_GUIDE (pre-existing) instead of GUIDE (spec artifact). 90/90 tests passing. |

## Known Deviations

### Detected by `2-dev-story` at 2026-04-16T21:11:13Z (session 365fca9a-3820-4c89-af64-8708be325a65)

- Story spec references `AgentType.GUIDE` but codebase already has `AgentType.SUBMISSION_GUIDE` with a 90-second timeout configured. Used the pre-existing enum value. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
- Story spec references `AgentType.GUIDE` but codebase already has `AgentType.SUBMISSION_GUIDE` with a 90-second timeout configured. Used the pre-existing enum value. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
