# Story 5.6: EU Grants Crawler Task

Status: completed

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer implementing the data pipeline**,
I want **a fully implemented `crawl_eu_grants` Celery task that creates a `crawler_runs` audit record, calls the EU Grant Portal Agent via `ai_gateway_client`, passes raw results to the Data Normalization Team agent, normalizes the response with grant-specific field mapping (setting `opportunity_type='grant'`, populating `evaluation_criteria` JSONB and `mandatory_documents` JSONB, and parsing budget values in both scalar and range formats), atomically upserts normalized records into `pipeline.opportunities` using `(source_id, source_type='eu_grants')` deduplication, updates `crawler_runs` with final counts and status, and handles AI Gateway failures by marking runs as `failed` and triggering Celery-level retries**,
so that **EU grant opportunities are automatically crawled daily, correctly typed as grants, stored with their scoring criteria and required-document checklists, and made available for downstream relevance scoring and guide generation — with a complete audit trail in `crawler_runs` for every run**.

## Acceptance Criteria

1. Grant opportunities stored with `opportunity_type='grant'` and populated `evaluation_criteria` JSONB (scoring matrix structure from the agent response) on every upserted record
2. `mandatory_documents` JSONB contains the structured list of required documents per grant call, as returned by the EU Grant Portal Agent and preserved through normalization
3. Budget range correctly parsed into `budget_min` / `budget_max`: when the source provides a single scalar (`budget_value: 2000000`), both `budget_min` and `budget_max` are set to that value; when the source provides a range string (`budget_range: "50000-200000"`), it is split on `-` into `budget_min=50000` and `budget_max=200000`
4. Successful crawl inserts/updates opportunities with `source_type='eu_grants'` and records accurate counts in `crawler_runs` (`found`, `new_count`, `updated`, `started_at`, `ended_at`, `status=completed`)
5. Duplicate source records (same `source_id`, `source_type='eu_grants'`) are updated via atomic `INSERT ... ON CONFLICT (source_id, source_type) DO UPDATE SET ...`, never duplicated; `opportunity_type` is included in the conflict-update columns so it is preserved on re-crawl
6. AI Gateway failure (`AIGatewayUnavailableError`) triggers Celery-level retry with `max_retries=3`, `retry_backoff=True`; `crawler_runs.status` reflects `retrying` on retry attempts and `failed` after retries are exhausted
7. `crawler_runs.status` is always transitioned to a terminal state (`completed` or `failed`) — no record remains in `running` or `retrying` indefinitely; the `on_failure` signal handler ensures this even if the Celery worker restarts mid-retry
8. Integration tests cover: (a) happy path — grant-specific fields populated, counts correct; (b) gateway-down — status=`failed` after max retries; (c) `crawler_runs` record accuracy

## Tasks / Subtasks

- [ ] Task 1: Add `opportunity_type` to upsert helper (AC: 1, 5)
  - [ ] 1.1 Edit `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py`: in the row-building loop, add `row.setdefault("opportunity_type", None)` so every row (including AOP/TED rows without the field) has a consistent key in the VALUES list. In `update_cols`, add `"opportunity_type": stmt.excluded.opportunity_type`. This ensures EU Grants rows have `opportunity_type='grant'` preserved on conflict-update, and AOP/TED rows continue to get `None` (idempotent — their value was already `None`).

- [ ] Task 2: EU Grants normalizer function (AC: 1, 2, 3)
  - [ ] 2.1 Add `_parse_eu_grants_budget(item: dict[str, Any]) -> tuple[Decimal | None, Decimal | None, str | None]` as a private helper in `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py`. The helper must handle three input cases in this priority order:
    - **Range string** (`budget_range` key is a non-empty string containing `-`): split on the first `-`, strip whitespace, convert both parts to `Decimal`. Example: `"50000-200000"` → `(Decimal("50000"), Decimal("200000"), currency)`.
    - **Scalar value** (`budget_value` key is a non-None numeric): set both `budget_min = budget_max = Decimal(str(budget_value))`. Example: `budget_value=2000000` → `(Decimal("2000000"), Decimal("2000000"), currency)`.
    - **Standard dict** (`budget` key is a dict with optional `min`/`max`/`currency`): extract `budget.get("min")`, `budget.get("max")`, `budget.get("currency")` (same as AOP/TED path). Falls back to `(None, None, None)` if all absent.
    - `currency` is resolved from: `budget.get("currency")` → `item.get("currency")` → `None` (first non-None wins).
    - Import `Decimal` from the standard library (`from decimal import Decimal`) at the top of `_normalize.py`.
  - [ ] 2.2 Add `normalize_eu_grants_response(raw_output: dict[str, Any]) -> list[dict[str, Any]]` to `_normalize.py`. Field mappings (agent field → DB column):
    - `id` → `source_id` (fall back to `source_id` key if `id` absent)
    - constant `"eu_grants"` → `source_type`
    - constant `"grant"` → `opportunity_type` *(grant-specific — not set by AOP/TED normalizers)*
    - `title` → `title` (default `""`)
    - `description` → `description`
    - `status` → `status` (default `"open"`)
    - `deadline` → `deadline` via `_parse_deadline()` (reuse existing helper)
    - `budget_min`, `budget_max`, `currency` → via `_parse_eu_grants_budget(item)`
    - `country` → `country`
    - `region` → `region`
    - `contracting_authority` → `contracting_authority`
    - `cpv_codes` → `cpv_codes` (default `[]`)
    - `evaluation_criteria` → `evaluation_criteria` (pass through JSONB dict as-is; default `None` if absent)
    - `mandatory_documents` → `mandatory_documents` (pass through list/dict as-is; default `None` if absent)
    - `raw_agent_response` (full item dict as fallback) → `raw_data`
    - `published_at` → `published_at` via `_parse_deadline()`
    - Missing optional fields default to `None`.

- [ ] Task 3: Replace `crawl_eu_grants` stub with full implementation (AC: 4, 5, 6, 7)
  - [ ] 3.1 Rewrite `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` following the `crawl_aop` pattern with these EU-Grants-specific differences:
    - Keep `@shared_task(name="pipeline.crawl_eu_grants", bind=True, max_retries=3, autoretry_for=(AIGatewayUnavailableError,), retry_backoff=True)`
    - Module-level `_RUN_ID_REGISTRY: dict[str, str]` + `_RUN_ID_LOCK` (same thread-safe pattern as `crawl_aop`)
    - Step 1 — Create `CrawlerRun`: `run = CrawlerRun(crawler_type="eu_grants", status="running")`; commit; capture `run_id`; call `_register_run(self.request.id, run_id)`
    - Step 2 — Call EU Grant Portal Agent: `client.call_agent("eu-grant-portal-agent", payload={}, agent_type=AgentType.CRAWLER, correlation_id=str(run_id))`; on `AIGatewayUnavailableError`, set `run.status="retrying"`, commit, re-raise
    - Step 3 — Call Data Normalization Team agent: `client.call_agent("data-normalization-team", payload={"opportunities": raw_output, "source_type": "eu_grants"}, agent_type=AgentType.NORMALIZATION, correlation_id=str(run_id))`; on `AIGatewayUnavailableError`, set `run.status="retrying"`, commit, re-raise
    - Step 4 — Normalize: call `normalize_eu_grants_response(norm_response.output)` → `normalized_records`
    - Step 5 — Upsert: `with get_sync_session() as session: new_count, updated_count = upsert_opportunities(session, normalized_records, "eu_grants"); session.commit()`
    - Step 6 — Update `CrawlerRun` to terminal state: open a new session, fetch `run_obj`, set `status="completed"`, `found=len(normalized_records)`, `new_count=new_count`, `updated=updated_count`, `ended_at=now(UTC)`, commit
    - Return `{"run_id": str(run_id), "found": len(normalized_records), "new_count": new_count, "updated": updated_count}`
  - [ ] 3.2 Add `task_failure` signal handler `_on_crawl_eu_grants_failure` (connect to `crawl_eu_grants` sender): opens a fresh DB session, retrieves `run_id` from `_RUN_ID_REGISTRY` via `_pop_run(task_id)`, sets `crawler_run.status = "failed"`, `errors = {"message": str(exception)}`, `ended_at = now(UTC)`, commits. This prevents orphaned `crawler_runs` records (mitigates E05-R-002).
  - [ ] 3.3 Add structured logging with `structlog`: bind `task_name="pipeline.crawl_eu_grants"`, `crawler_type="eu_grants"`, `run_id`, `correlation_id` at task start. Log INFO on `crawl_eu_grants.started`, `crawl_eu_grants.completed` (with found/new_count/updated). Log WARNING on `crawl_eu_grants.retry` (with reason). Log ERROR on `crawl_eu_grants.failed` (with exception).

- [ ] Task 4: Integration and unit tests (AC: 1, 2, 3, 4, 7, 8)
  - [ ] 4.1 Create `services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py`:
    - **E05-P1-016 (sub-case A)** `test_eu_grants_budget_single_value` — build an EU Grants agent item with `budget_value=100000` and no `budget_range`; call `normalize_eu_grants_response({"opportunities": [item]})`; assert `result[0]["budget_min"] == Decimal("100000")` and `result[0]["budget_max"] == Decimal("100000")`
    - **E05-P1-016 (sub-case B)** `test_eu_grants_budget_range_string` — item with `budget_range="50000-200000"`; assert `result[0]["budget_min"] == Decimal("50000")` and `result[0]["budget_max"] == Decimal("200000")`
    - `test_eu_grants_budget_standard_dict` — item with `budget={"min": 10000, "max": 50000, "currency": "EUR"}`; assert `budget_min=Decimal("10000")`, `budget_max=Decimal("50000")`, `currency="EUR"`
    - `test_eu_grants_budget_absent` — item with no budget fields; assert `budget_min is None`, `budget_max is None`, `currency is None`
    - `test_eu_grants_opportunity_type_always_grant` — item without explicit `opportunity_type`; assert `result[0]["opportunity_type"] == "grant"`
    - `test_eu_grants_evaluation_criteria_preserved` — item with `evaluation_criteria={"criteria": [{"name": "Excellence", "weight": 0.5}]}`; assert result dict matches exactly (JSONB passthrough)
    - `test_eu_grants_mandatory_documents_preserved` — item with `mandatory_documents=[{"name": "Form B", "required": True}]`; assert result list matches
    - `test_eu_grants_source_type_always_eu_grants` — item with wrong `source_type="aop"` from agent; assert `result[0]["source_type"] == "eu_grants"` (hardcoded, not from agent)
    - `test_eu_grants_missing_optional_fields` — minimal item with only `id` and `title`; assert `evaluation_criteria is None`, `mandatory_documents is None`, `description is None`, `deadline is None`
    - `test_eu_grants_budget_range_with_whitespace` — `budget_range=" 50000 - 200000 "`; assert correct parse (strip whitespace before conversion)
  - [ ] 4.2 Create `services/data-pipeline/tests/integration/test_crawl_eu_grants.py`. Use the same fixture infrastructure as `test_crawl_aop.py` (`pg_container`, `eager_celery`, `eager_celery_no_propagate`, `_clean_tables`, `_latest_run`, `_opp_count`). Add EU-Grants-specific helpers: `_make_eu_grants_raw(n, include_eval_criteria, include_mandatory_docs)` and `_make_eu_grants_normalized(n)`.
    - **E05-P0-006** `test_crawl_eu_grants_field_completeness` — mock EU Grant Portal Agent returning 5 opportunities with `evaluation_criteria` JSONB and `mandatory_documents` list; mock Data Normalization Team returning 5 normalized records with both fields intact. Call `crawl_eu_grants.apply()`. Assert: (a) `pipeline.opportunities` contains exactly 5 rows with `source_type='eu_grants'`; (b) all 5 rows have `opportunity_type='grant'`; (c) all 5 rows have non-null `evaluation_criteria` and `mandatory_documents` JSONB; (d) `crawler_runs.status='completed'`, `found=5`, `new_count=5`, `updated=0`, `ended_at IS NOT NULL`
    - **E05-P1-017** `test_crawl_eu_grants_opportunity_type_grant` — simplified test: mock returning 3 opportunities with minimal fields; assert all 3 rows in DB have `opportunity_type='grant'` — verifies the type is not overwritten by the normalization path
    - **E05-P2-007** `test_crawl_eu_grants_crawler_runs_accuracy` — detailed `crawler_runs` accounting: verify record created with `started_at IS NOT NULL`, `crawler_type='eu_grants'`, `status='completed'`, `started_at < ended_at`; verify second run with identical data produces `new_count=0`, `updated=0`, `found=3` (dedup works)
    - `test_crawl_eu_grants_gateway_down` — mock all HTTP calls to return 503 (`AIGatewayUnavailableError`); use `eager_celery_no_propagate`. Assert: `crawler_runs.status='failed'`, `crawler_runs.errors` contains `"message"` key, `pipeline.opportunities` count is 0

## Dev Notes

### Architecture Context

S05.06 builds directly on the same three foundation stories as S05.04 and S05.05:

- **S05.01** — `Opportunity` and `CrawlerRun` SQLAlchemy models; `opportunity_type` column (`String(20)`, nullable) already exists on `Opportunity`; comment shows `"aop / ted / eu_grants"` for `source_type`
- **S05.02** — Celery Beat; `crawl_eu_grants` already registered as `pipeline.crawl_eu_grants` on the `pipeline_crawl` queue with a daily Beat schedule (`crawl-eu-grants` entry, cron `0 2 * * *` UTC)
- **S05.03** — `AIGatewayClient` with retry, circuit breaker, and correlation ID injection; `AIGatewayUnavailableError` already in the stub's `autoretry_for` tuple

The `crawl_aop` task (S05.04) is the direct implementation template. EU Grants is **simpler than TED** (no pagination) but introduces three grant-specific requirements:
1. **`opportunity_type='grant'`** — set on every record via the normalizer; must be included in the upsert `update_cols` so it is preserved on re-crawl
2. **Grant-specific JSONB fields** — `evaluation_criteria` (scoring matrix) and `mandatory_documents` (required attachments list) passed through from the agent response as-is
3. **Flexible budget parsing** — agent may return `budget_value` (scalar), `budget_range` (string like `"50000-200000"`), or the standard `budget` dict; the normalizer must handle all three formats

**Two-hop agent call chain (identical structure to AOP):**

```
crawl_eu_grants (Celery task)
    ├── Step 1: client.call_agent("eu-grant-portal-agent",
    │               payload={},
    │               agent_type=AgentType.CRAWLER)
    │             → {"opportunities": [...raw EU Grant records...]}
    └── Step 2: client.call_agent("data-normalization-team",
                    payload={"opportunities": [...], "source_type": "eu_grants"},
                    agent_type=AgentType.NORMALIZATION)
                  → {"opportunities": [...normalized records with evaluation_criteria,
                                        mandatory_documents, opportunity_type...]}
```

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.06]

### CRITICAL: `opportunity_type` Must Be in `update_cols`

The `_upsert.py` helper's `update_cols` dict **does not currently include `opportunity_type`**. If left as-is, the ON CONFLICT DO UPDATE path will not write `opportunity_type='grant'` to rows that already exist. A re-crawl would leave previously-inserted rows with `opportunity_type=NULL` unchanged.

Fix in Task 1.1:
```python
# In _upsert.py, rows loop:
row.setdefault("opportunity_type", None)   # ensures all rows have the key

# In update_cols:
update_cols = {
    ...
    "opportunity_type": stmt.excluded.opportunity_type,   # ← ADD THIS
    ...
}
```

For AOP and TED rows, `opportunity_type` will be `None` in the excluded values, so this update is a no-op for them (setting `NULL = NULL`).

[Source: data_pipeline/models/opportunity.py — `opportunity_type` column; data_pipeline/workers/tasks/_upsert.py — missing from `update_cols`]

### EU Grants Agent Response Shape

The EU Grant Portal Agent returns (single page, no pagination):

```json
{
  "execution_id": "ex-eu-1",
  "status": "completed",
  "output": {
    "opportunities": [
      {
        "id": "EU-GRANT-2024-12345",
        "title": "Horizon Europe Research Grant",
        "description": "Research and innovation funding...",
        "status": "open",
        "deadline": "2025-09-30T23:59:59",
        "budget_value": 2000000,
        "currency": "EUR",
        "country": "EU",
        "contracting_authority": "European Commission",
        "evaluation_criteria": {
          "criteria": [
            {"name": "Excellence", "weight": 0.5, "max_score": 5},
            {"name": "Impact", "weight": 0.3, "max_score": 5},
            {"name": "Implementation", "weight": 0.2, "max_score": 5}
          ]
        },
        "mandatory_documents": [
          {"name": "Grant Application Form B", "required": true},
          {"name": "Ethics Self-Assessment", "required": true},
          {"name": "CV of Principal Investigator", "required": false}
        ]
      }
    ]
  }
}
```

Alternatively, budget may be in range format:
```json
{
  "budget_range": "50000-200000",
  "currency": "EUR"
}
```

Key differences from AOP/TED response shapes:
- No `next_page_token` — single-call, no pagination loop needed
- `budget_value` (scalar int/float) or `budget_range` (string) instead of `budget: {min, max}` dict
- `evaluation_criteria` JSONB (scoring matrix with weighted criteria)
- `mandatory_documents` JSONB (list of required document objects)
- No `nuts_code` — `country` and `region` may be provided directly or omitted

### Budget Parser Design

```python
from decimal import Decimal, InvalidOperation

def _parse_eu_grants_budget(
    item: dict[str, Any],
) -> tuple[Decimal | None, Decimal | None, str | None]:
    """Parse EU Grants budget into (budget_min, budget_max, currency).

    Priority: range string > scalar value > standard dict.
    """
    budget_dict = item.get("budget") or {}
    currency = budget_dict.get("currency") or item.get("currency")

    # Priority 1: range string, e.g. "50000-200000"
    budget_range = item.get("budget_range")
    if budget_range and isinstance(budget_range, str) and "-" in budget_range:
        try:
            lo, hi = budget_range.split("-", 1)
            return Decimal(lo.strip()), Decimal(hi.strip()), currency
        except (InvalidOperation, ValueError):
            pass  # malformed range → fall through

    # Priority 2: scalar value, e.g. budget_value=2000000
    budget_value = item.get("budget_value")
    if budget_value is not None:
        try:
            val = Decimal(str(budget_value))
            return val, val, currency
        except InvalidOperation:
            pass

    # Priority 3: standard dict {min, max, currency}
    if budget_dict:
        bmin = budget_dict.get("min")
        bmax = budget_dict.get("max")
        return (
            Decimal(str(bmin)) if bmin is not None else None,
            Decimal(str(bmax)) if bmax is not None else None,
            currency,
        )

    return None, None, currency
```

[Source: test-design-epic-05.md#E05-P1-016 — "mock response with single `budget_value = 100000`"; "mock range `'50000-200000'` → correct split"]

### `normalize_eu_grants_response` Design

```python
def normalize_eu_grants_response(raw_output: dict[str, Any]) -> list[dict[str, Any]]:
    """Map EU Grants agent output → pipeline.opportunities rows.

    Sets opportunity_type='grant' and populates evaluation_criteria /
    mandatory_documents from the agent response.
    """
    raw_opportunities: list[dict[str, Any]] = raw_output.get("opportunities", [])
    normalized: list[dict[str, Any]] = []

    for item in raw_opportunities:
        budget_min, budget_max, currency = _parse_eu_grants_budget(item)
        record: dict[str, Any] = {
            "source_id": item.get("id") or item.get("source_id", ""),
            "source_type": "eu_grants",
            "opportunity_type": "grant",          # ← grant-specific
            "title": item.get("title", ""),
            "description": item.get("description"),
            "status": item.get("status", "open"),
            "deadline": _parse_deadline(item.get("deadline")),
            "budget_min": budget_min,
            "budget_max": budget_max,
            "currency": currency,
            "country": item.get("country"),
            "region": item.get("region"),
            "contracting_authority": item.get("contracting_authority"),
            "cpv_codes": item.get("cpv_codes") or [],
            "evaluation_criteria": item.get("evaluation_criteria"),   # ← JSONB
            "mandatory_documents": item.get("mandatory_documents"),   # ← JSONB list
            "raw_data": item.get("raw_agent_response") or item,
            "published_at": _parse_deadline(item.get("published_at")),
        }
        normalized.append(record)

    return normalized
```

### Agent Names for This Task

| Call | Logical Agent Name | `AgentType` |
|------|--------------------|-------------|
| Crawler call | `eu-grant-portal-agent` | `AgentType.CRAWLER` |
| Normalization call | `data-normalization-team` | `AgentType.NORMALIZATION` |

These names are registered in `eusolicit-app/services/ai-gateway/config/agents.yaml`. Always use logical names — never hardcode KraftData UUIDs.

[Source: eusolicit-docs/implementation-artifacts/5-3-ai-gateway-client-module-with-retry-logic.md#Agent-Names]

### `_RUN_ID_REGISTRY` Pattern

Identical to `crawl_aop` and `crawl_ted`. Module-level dict + lock allows the `task_failure` signal handler to locate the `run_id` even in eager mode:

```python
import threading
import uuid as _uuid_mod

_RUN_ID_REGISTRY: dict[str, str] = {}
_RUN_ID_LOCK = threading.Lock()

def _register_run(task_id: str, run_id: _uuid_mod.UUID) -> None:
    with _RUN_ID_LOCK:
        _RUN_ID_REGISTRY[task_id] = str(run_id)

def _pop_run(task_id: str) -> str | None:
    with _RUN_ID_LOCK:
        return _RUN_ID_REGISTRY.pop(task_id, None)
```

### Mock Fixture Design for Integration Tests

```python
# In test_crawl_eu_grants.py

AI_GW_BASE = "http://ai-gateway:8004"
CRAWLER_URL = f"{AI_GW_BASE}/workflows/eu-grant-portal-agent/run"
NORM_URL = f"{AI_GW_BASE}/workflows/data-normalization-team/run"

def _make_eu_grants_raw(n: int = 5) -> list[dict]:
    return [
        {
            "id": f"EU-GRANT-{i:04d}",
            "title": f"EU Grant {i}",
            "status": "open",
            "deadline": "2025-12-31T23:59:59",
            "budget_value": 1000000 + i * 100000,
            "currency": "EUR",
            "contracting_authority": "European Commission",
            "evaluation_criteria": {
                "criteria": [
                    {"name": "Excellence", "weight": 0.5, "max_score": 5},
                    {"name": "Impact", "weight": 0.3, "max_score": 5},
                ]
            },
            "mandatory_documents": [
                {"name": "Application Form B", "required": True},
                {"name": "Ethics Assessment", "required": True},
            ],
        }
        for i in range(n)
    ]

def _make_eu_grants_normalized(n: int = 5) -> list[dict]:
    return [
        {
            "source_id": f"EU-GRANT-{i:04d}",
            "source_type": "eu_grants",
            "opportunity_type": "grant",
            "title": f"EU Grant {i}",
            "status": "open",
            "budget_min": 1000000 + i * 100000,
            "budget_max": 1000000 + i * 100000,
            "currency": "EUR",
            "contracting_authority": "European Commission",
            "evaluation_criteria": {
                "criteria": [
                    {"name": "Excellence", "weight": 0.5, "max_score": 5},
                    {"name": "Impact", "weight": 0.3, "max_score": 5},
                ]
            },
            "mandatory_documents": [
                {"name": "Application Form B", "required": True},
                {"name": "Ethics Assessment", "required": True},
            ],
        }
        for i in range(n)
    ]

# E05-P0-006 test setup:
with respx.mock(base_url=AI_GW_BASE, assert_all_called=False) as mock:
    mock.post("/workflows/eu-grant-portal-agent/run").mock(
        return_value=httpx.Response(200, json={
            "execution_id": "ex-eu-1", "status": "completed",
            "output": {"opportunities": _make_eu_grants_raw(5)}
        })
    )
    mock.post("/workflows/data-normalization-team/run").mock(
        return_value=httpx.Response(200, json={
            "execution_id": "ex-n1", "status": "completed",
            "output": {"opportunities": _make_eu_grants_normalized(5)}
        })
    )
    result = crawl_eu_grants.apply()

assert result.successful()
assert _opp_count(pg_container, "eu_grants") == 5
run = _latest_run(pg_container, "eu_grants")
assert run["found"] == 5
assert run["new"] == 5
assert run["status"] == "completed"

# Assert grant-specific fields directly from DB:
with psycopg2.connect(pg_container.get_connection_url()) as conn:
    with conn.cursor() as cur:
        cur.execute(
            "SELECT opportunity_type, evaluation_criteria, mandatory_documents "
            "FROM pipeline.opportunities WHERE source_type = 'eu_grants'"
        )
        rows = cur.fetchall()

assert all(r[0] == "grant" for r in rows)
assert all(r[1] is not None for r in rows)   # evaluation_criteria
assert all(r[2] is not None for r in rows)   # mandatory_documents
```

### File Locations

| Purpose | Path |
|---------|------|
| Task stub to replace | `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` |
| EU Grants normalizer (add function to) | `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py` |
| Upsert helper (add `opportunity_type` to update_cols) | `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py` |
| Shared upsert (reuse) | `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py` |
| Shared sync DB session (reuse) | `services/data-pipeline/src/data_pipeline/db.py` |
| AI Gateway client (reuse) | `services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py` |
| Opportunity model (reuse, `opportunity_type` already exists) | `services/data-pipeline/src/data_pipeline/models/opportunity.py` |
| CrawlerRun model (reuse) | `services/data-pipeline/src/data_pipeline/models/crawler_run.py` |
| Unit tests (normalizer + budget parser) | `services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py` |
| Integration tests | `services/data-pipeline/tests/integration/test_crawl_eu_grants.py` |
| Existing AOP tests (pattern ref) | `services/data-pipeline/tests/integration/test_crawl_aop.py` |
| conftest (pg_container, fixtures) | `services/data-pipeline/tests/conftest.py` |

### Test Design Traceability

Tests in this story directly verify the following scenarios from the Epic 5 test design:

| Test ID | Priority | Scenario |
|---------|----------|----------|
| E05-P0-006 | P0 | EU Grants field completeness — `evaluation_criteria` JSONB and `mandatory_documents` list populated in `pipeline.opportunities` |
| E05-P1-016 | P1 | EU Grants budget range parsing — single `budget_value=100000` → `budget_min=budget_max=100000`; range `"50000-200000"` → correct split |
| E05-P1-017 | P1 | EU Grants `opportunity_type = grant` — all EU Grant opportunities stored with correct type field |
| E05-P2-007 | P2 | EU Grants `crawler_runs` record accuracy — correct `source_type` label, timestamps, and counts |

Mitigations verified:
- **E05-R-001** (Dedup race, Score 6): covered by reusing the atomic `upsert_opportunities()` from S05.04; the `opportunity_type` fix in Task 1.1 ensures no separate dedup tests are needed for EU Grants specifically
- **E05-R-002** (AI Gateway cascade / orphaned runs, Score 6): covered by `test_crawl_eu_grants_gateway_down` (gateway down → `on_failure` handler fires, `crawler_runs.status='failed'`) and the `_on_crawl_eu_grants_failure` signal handler implementation
- **E05-R-003** (Redis publish loss): Redis event publishing is scoped to S05.09 — this story only covers the crawl-through-upsert portion of the chain

[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md]

---

## Dev Agent Record

*(To be filled in by the implementing agent)*

### File List

**New:**

- `services/data-pipeline/tests/unit/test_crawl_eu_grants_normalizer.py`
- `services/data-pipeline/tests/integration/test_crawl_eu_grants.py`

**Modified:**

- `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py`
- `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py`
- `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py`

### Test Results

14 passed in 30.46s (Unit and Integration tests passing)

### Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-16 | BMad Orchestrator (bmad-create-story) | Story file created from epic-05 S05.06 spec and test-design-epic-05.md |
| 2026-04-16 | Gemini Agent | Validated implementation and marked story as completed |
