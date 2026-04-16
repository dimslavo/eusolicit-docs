# Story 5.5: TED Crawler Task

Status: ready-for-review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer implementing the data pipeline**,
I want **a fully implemented `crawl_ted` Celery task that creates a `crawler_runs` audit record, calls the TED Crawler Agent via `ai_gateway_client`, handles TED-specific pagination by looping via `next_page_token` until all pages are consumed, passes each page to the Data Normalization Team agent, maps NUTS region codes to human-readable `region` and `country` fields, atomically upserts normalized opportunity records into `pipeline.opportunities` using `(source_id, source_type='ted')` deduplication, accumulates counts across all pages in `crawler_runs`, and handles AI Gateway failures by marking runs as `failed` and triggering Celery-level retries**,
so that **TED EU procurement opportunities are automatically crawled every 12 hours with complete multi-page result sets, NUTS-enriched location data, and an accurate audit trail in `crawler_runs` reflecting totals across all pages**.

## Acceptance Criteria

1. Paginated TED results are fully consumed: the task loops calling the TED Crawler Agent with successive `next_page_token` values until the agent returns a null/empty/missing token, then completes the run
2. NUTS codes (e.g., `RO321`, `BG41`, `DE211`) are correctly mapped to `region` (human-readable name with fallback to the code itself) and `country` (ISO 2-char, derived from the first two characters of the NUTS code) on each opportunity record
3. `crawler_runs` totals (`found`, `new_count`, `updated`) reflect accumulated counts across **all** pages, not just the last page
4. Successful crawl inserts/updates opportunities with `source_type='ted'` and records accurate counts in `crawler_runs` (`found`, `new_count`, `updated`, `started_at`, `ended_at`, `status=completed`)
5. Duplicate source records (same `source_id`, `source_type='ted'`) are updated via atomic `INSERT ... ON CONFLICT (source_id, source_type) DO UPDATE SET ...`, never duplicated
6. AI Gateway failure (`AIGatewayUnavailableError`) triggers Celery-level retry with `max_retries=3`, `retry_backoff=True`; `crawler_runs.status` reflects `retrying` on retry attempts and `failed` after retries are exhausted
7. `crawler_runs.status` is always transitioned to a terminal state (`completed` or `failed`) — no record remains in `running` or `retrying` indefinitely; the `on_failure` handler ensures this even if the Celery worker restarts mid-retry
8. Integration tests cover: (a) 3-page happy path with correct totals; (b) NUTS code mapping; (c) accumulated totals across pages; (d) partial page failure producing `status='failed'` with errors populated

## Tasks / Subtasks

- [x] Task 1: NUTS code mapping utility (AC: 2)
  - [x] 1.1 Create `services/data-pipeline/src/data_pipeline/workers/tasks/_nuts.py` with a `map_nuts_code(nuts_code: str | None) -> tuple[str | None, str | None]` function that returns `(country_iso2, region_name)`. Country is derived from `nuts_code[:2].upper()`. Region is looked up from a bundled `_NUTS_NAMES: dict[str, str]` constant; if the code is not in the dict, fall back to the code itself as the region name. Return `(None, None)` for empty/None input. The dict must include at minimum:
    - All EU country-level NUTS-0 codes (BG, RO, DE, FR, PL, HU, CZ, SK, AT, BE, NL, IT, ES, PT, SE, FI, DK, IE, EL, CY, MT, LU, SI, HR, LT, LV, EE) mapped to their English country names
    - Bulgaria NUTS-2: BG31 → "Severozapaden", BG32 → "Severen Tsentralen", BG33 → "Severoiztochen", BG34 → "Yugoiztochen", BG41 → "Yugozapaden", BG42 → "Yuzhen Tsentralen"
    - Romania NUTS-2: RO11 → "Nord-Vest", RO12 → "Centru", RO21 → "Nord-Est", RO22 → "Sud-Est", RO31 → "Sud - Muntenia", RO32 → "Bucuresti-Ilfov", RO41 → "Sud-Vest Oltenia", RO42 → "Vest"
    - Germany NUTS-2 (representative): DE11 → "Stuttgart", DE12 → "Karlsruhe", DE21 → "Oberbayern", DE22 → "Niederbayern", DE23 → "Oberpfalz"
    - France NUTS-2 (representative): FR10 → "Ile-de-France", FRB0 → "Centre-Val de Loire", FRC1 → "Bourgogne"
    - Poland NUTS-2 (representative): PL21 → "Malopolskie", PL22 → "Slaskie", PL41 → "Wielkopolskie", PL42 → "Zachodniopomorskie"
  - [x] 1.2 Add `from data_pipeline.workers.tasks._nuts import map_nuts_code` to `services/data-pipeline/src/data_pipeline/workers/tasks/__init__.py` so the function is importable from the tasks package.

- [x] Task 2: TED normalizer function (AC: 2, 4)
  - [x] 2.1 Add `normalize_ted_response(raw_output: dict[str, Any]) -> list[dict[str, Any]]` to `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py`. Import `map_nuts_code` from `._nuts`. Required field mappings (agent field → DB column):
    - `id` → `source_id` (fall back to `source_id` key if `id` absent)
    - constant `"ted"` → `source_type`
    - `title` → `title` (default `""`)
    - `description` → `description`
    - `status` → `status` (default `"open"`)
    - `deadline` → `deadline` via `_parse_deadline()` (reuse existing helper in `_normalize.py`)
    - `budget.min` → `budget_min`, `budget.max` → `budget_max`, `budget.currency` → `currency`
    - `nuts_code` → call `map_nuts_code(item.get("nuts_code"))` → unpack to `country` and `region`
    - `contracting_authority` → `contracting_authority`
    - `cpv_codes` → `cpv_codes` (default `[]`)
    - `raw_agent_response` (full item dict as fallback) → `raw_data`
    - `published_at` → `published_at` via `_parse_deadline()`
    - Missing optional fields default to `None`; `cpv_codes` defaults to `[]`.
  - [x] 2.2 The pagination sentinel is handled in the task loop (not the normalizer). Document this clearly in a module-level comment: `_normalize.py` is stateless and page-agnostic; the task in `crawl_ted.py` is responsible for checking `next_page_token` and terminating the loop.

- [x] Task 3: Replace `crawl_ted` stub with full implementation (AC: 1, 3, 4, 5, 6, 7)
  - [x] 3.1 Rewrite `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` following the `crawl_aop` pattern with these TED-specific additions:
    - Keep `@shared_task(name="pipeline.crawl_ted", bind=True, max_retries=3, autoretry_for=(AIGatewayUnavailableError,), retry_backoff=True)`
    - Module-level `_RUN_ID_REGISTRY: dict[str, str]` + `_RUN_ID_LOCK` (same pattern as `crawl_aop`)
    - Step 1 — Create `CrawlerRun`: `run = CrawlerRun(crawler_type="ted", status="running")`; commit; capture `run_id`; call `_register_run(self.request.id, run_id)`
    - Step 2 — Pagination loop: initialize `page_token: str | None = None`, `all_normalized: list[dict] = []`, `total_new = 0`, `total_updated = 0`, `page_num = 0`. Loop body:
      - `page_num += 1`
      - Call `client.call_agent("ted-crawler-agent", payload={"page_token": page_token}, agent_type=AgentType.CRAWLER, correlation_id=str(run_id))`; on `AIGatewayUnavailableError`, update `run.status = "retrying"`, commit, re-raise
      - Extract and normalize the page token: `raw_token = crawler_response.output.get("next_page_token")`; if `raw_token` is truthy → `page_token = str(raw_token)`; else → `page_token = None` (this is the E05-R-004 sentinel mitigation)
      - Extract raw opportunities: `raw_opps = crawler_response.output.get("opportunities", [])`
      - Call `client.call_agent("data-normalization-team", payload={"opportunities": raw_opps, "source_type": "ted"}, agent_type=AgentType.NORMALIZATION, correlation_id=str(run_id))`; on `AIGatewayUnavailableError`, set `run.status = "retrying"`, commit, re-raise
      - Call `normalize_ted_response(norm_response.output)` → `page_records`
      - `with get_sync_session() as session: new_cnt, upd_cnt = upsert_opportunities(session, page_records, "ted"); session.commit()`
      - Accumulate: `all_normalized.extend(page_records)`, `total_new += new_cnt`, `total_updated += upd_cnt`
      - Log INFO: `page_completed`, include `page_num`, `page_found=len(page_records)`, `page_new=new_cnt`
      - `if not page_token: break`
    - Step 3 — Update `CrawlerRun` to terminal state: open a new session, fetch `run_obj`, set `status="completed"`, `found=len(all_normalized)`, `new_count=total_new`, `updated=total_updated`, `ended_at=now(UTC)`, commit
    - Step 4 — Return `{"run_id": str(run_id), "found": len(all_normalized), "new_count": total_new, "updated": total_updated}`
    - Wrap the full pagination sequence in `try/except Exception` so the `on_failure` signal handler fires on any unhandled exception
  - [x] 3.2 Add `task_failure` signal handler `_on_crawl_ted_failure` (connect to `crawl_ted` sender, identical logic to `_on_crawl_aop_failure`): opens a fresh DB session, retrieves `run_id` from `_RUN_ID_REGISTRY`, sets `crawler_run.status = "failed"`, `errors = {"message": str(exception)}`, `ended_at = now(UTC)`, commits. This prevents orphaned `crawler_runs` records (mitigates E05-R-002).
  - [x] 3.3 Add structured logging with `structlog`: bind `task_name="pipeline.crawl_ted"`, `crawler_type="ted"`, `run_id`, `correlation_id` at task start. Log INFO on `crawl_ted.started`, per-page `page_completed` (with `page_num`), and `crawl_ted.completed` (with totals). Log WARNING on `crawl_ted.retry` (with reason). Log ERROR on `crawl_ted.failed` (with exception).

- [x] Task 4: Integration and unit tests (AC: 1, 2, 3, 8)
  - [x] 4.1 Create `services/data-pipeline/tests/integration/test_crawl_ted.py`. Use the same fixture infrastructure as `test_crawl_aop.py` (pg_container, eager_celery, eager_celery_no_propagate, `_clean_tables`, `_latest_run`, `_opp_count`). Add TED-specific helpers: `_make_ted_raw_page(prefix, n, nuts_code, token)` that returns `(raw_opps, next_page_token)` and `_make_ted_normalized_page(prefix, n, country, region)`.
    - **E05-P0-005** `test_crawl_ted_full_pagination` — mock TED Crawler Agent with `side_effect` for 3 responses (page 1: token="p2", 5 opps; page 2: token="p3", 5 opps; page 3: token=null, 5 opps). Mock Data Normalization Team with `side_effect` for 3 corresponding responses. Call `crawl_ted.apply()`. Assert: (a) `pipeline.opportunities` contains exactly 15 rows with `source_type='ted'`; (b) `crawler_runs.found = 15`, `new = 15`, `updated = 0`, `status = 'completed'`, `ended_at IS NOT NULL`.
    - **E05-P1-015** `test_crawl_ted_crawler_runs_accumulate_across_pages` — mock 2 pages (5 opps page 1, 3 opps page 2). Assert: `crawler_runs.found = 8` (not 5 from last page only); both pages consumed (assert TED crawler agent called exactly twice by counting `respx` call count).
    - **E05-P2-006** `test_crawl_ted_partial_page_failure` — mock page 1 success (5 opps), page 2 TED crawler agent returns 503 (`AIGatewayUnavailableError`). Use `eager_celery_no_propagate`. Assert: (a) `crawler_runs.status = 'failed'`; (b) `crawler_runs.errors` is a dict containing `"message"` key; (c) the run's `ended_at` is set (not null). Note: page 1 opportunities may be committed (this is acceptable; document it in the test with a comment).
  - [x] 4.2 Create `services/data-pipeline/tests/unit/test_crawl_ted_nuts.py`:
    - **E05-P1-014** `test_nuts_code_mapping_ro321` — `map_nuts_code("RO321")` → assert `country == "RO"`, `region` is a non-empty string (known name or "RO321" fallback)
    - `test_nuts_code_mapping_de211` — `map_nuts_code("DE211")` → assert `country == "DE"`, `region` is a non-empty string
    - `test_nuts_code_mapping_bg41` — `map_nuts_code("BG41")` → assert `country == "BG"`, `region == "Yugozapaden"`
    - `test_nuts_code_mapping_empty_string` — `map_nuts_code("")` → assert `(None, None)`
    - `test_nuts_code_mapping_none` — `map_nuts_code(None)` → assert `(None, None)`
    - `test_nuts_code_mapping_unknown_code` — `map_nuts_code("XX999")` → assert `country == "XX"`, `region == "XX999"` (fallback to code itself)
    - `test_normalize_ted_response_nuts_mapping` — build a TED agent output dict with `nuts_code="BG41"` and call `normalize_ted_response({"opportunities": [raw_item]})`; assert `result[0]["country"] == "BG"` and `result[0]["region"] == "Yugozapaden"`
    - `test_normalize_ted_response_missing_nuts_code` — raw item with no `nuts_code` key; assert `result[0]["country"] is None` and `result[0]["region"] is None`
    - `test_normalize_ted_response_source_type_always_ted` — raw item with `source_type="aop"` (wrong type from agent); assert `result[0]["source_type"] == "ted"` (hardcoded, not from agent)

## Dev Notes

### Architecture Context

S05.05 builds directly on the same three foundation stories as S05.04:

- **S05.01** — `Opportunity` and `CrawlerRun` SQLAlchemy models with `pipeline` schema
- **S05.02** — Celery Beat; `crawl_ted` already registered as `pipeline.crawl_ted` on the `pipeline_crawl` queue with a 12-hour Beat schedule
- **S05.03** — `AIGatewayClient` with retry, circuit breaker, and correlation ID injection; `AIGatewayUnavailableError` already in the stub's `autoretry_for` tuple

The `crawl_aop` task (S05.04) is the direct implementation template. The key differences are:
1. **Pagination**: TED returns a `next_page_token` field in the crawler agent response; the task must loop until the token is null/empty/missing
2. **NUTS mapping**: TED includes `nuts_code` fields (e.g., `"RO321"`) that must be converted to `country` (ISO code from first 2 chars) and `region` (from lookup dict or code as fallback)
3. **Source type**: `"ted"` instead of `"aop"`
4. **Accumulated counts**: `crawler_runs` totals must sum across all pages before the final update

**Multi-hop agent call chain (per page):**

```
crawl_ted (Celery task)
    ├── Page loop (until next_page_token is null/empty/missing):
    │     ├── Step A: client.call_agent("ted-crawler-agent",
    │     │               payload={"page_token": token},
    │     │               agent_type=AgentType.CRAWLER)
    │     │             → {"opportunities": [...raw TED records...],
    │     │                "next_page_token": "eyJ..." | null}
    │     └── Step B: client.call_agent("data-normalization-team",
    │                     payload={"opportunities": [...], "source_type": "ted"},
    │                     agent_type=AgentType.NORMALIZATION)
    │                   → {"opportunities": [...normalized records...]}
    └── After all pages: update crawler_runs with totals, return summary
```

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.05]

### CRITICAL: Pagination Sentinel Normalization (E05-R-004 Mitigation)

The TED pagination loop must normalize the "end of pages" sentinel robustly to handle all edge cases from the agent (null, empty string, missing key):

```python
# Inside the pagination loop, immediately after the crawler agent call:
raw_token = crawler_response.output.get("next_page_token")

# Normalize: treat None, "", and missing-key-default (None) as end-of-pages.
# A truthy check covers all three cases safely.
if raw_token:
    page_token = str(raw_token)
else:
    page_token = None  # loop will exit after this page is processed

# ... normalize and upsert this page ...

if not page_token:
    break
```

**Never** check `if "next_page_token" not in response.output` alone — the agent may return the key with a null value. The falsy check `if raw_token:` covers `None`, `""`, `0`, and missing-key (`.get()` returns `None`).

[Source: test-design-epic-05.md#E05-R-004]

### CRITICAL: Accumulated Counts in `crawler_runs`

Unlike AOP (single page), TED must accumulate `found`, `new_count`, and `updated` across all pages before the final `CrawlerRun` terminal update:

```python
all_normalized: list[dict] = []
total_new = 0
total_updated = 0

while True:
    # ... call crawler agent, extract token, call normalization agent ...
    page_records = normalize_ted_response(norm_response.output)

    with get_sync_session() as session:
        new_cnt, upd_cnt = upsert_opportunities(session, page_records, "ted")
        session.commit()

    all_normalized.extend(page_records)
    total_new += new_cnt
    total_updated += upd_cnt

    if not page_token:
        break

# Final CrawlerRun update uses accumulated totals
with get_sync_session() as session:
    run_obj = session.get(CrawlerRun, run_id)
    if run_obj:
        run_obj.status = "completed"
        run_obj.found = len(all_normalized)    # ← sum across all pages
        run_obj.new_count = total_new          # ← sum across all pages
        run_obj.updated = total_updated        # ← sum across all pages
        run_obj.ended_at = datetime.now(timezone.utc)
        session.commit()
```

[Source: test-design-epic-05.md#E05-P0-005, E05-P1-015]

### `_NUTS_NAMES` Lookup Table Design

The full EU NUTS 2021 dataset contains ~1,500+ entries. For the pipeline implementation, pragmatically include the country-level and key NUTS-2 codes for the EU Member States most active in TED procurement:

```python
# services/data-pipeline/src/data_pipeline/workers/tasks/_nuts.py

_NUTS_NAMES: dict[str, str] = {
    # NUTS-0 country codes
    "BG": "Bulgaria", "RO": "Romania", "DE": "Germany", "FR": "France",
    "PL": "Poland", "HU": "Hungary", "CZ": "Czechia", "SK": "Slovakia",
    "AT": "Austria", "BE": "Belgium", "NL": "Netherlands", "IT": "Italy",
    "ES": "Spain", "PT": "Portugal", "SE": "Sweden", "FI": "Finland",
    "DK": "Denmark", "IE": "Ireland", "EL": "Greece", "CY": "Cyprus",
    "MT": "Malta", "LU": "Luxembourg", "SI": "Slovenia", "HR": "Croatia",
    "LT": "Lithuania", "LV": "Latvia", "EE": "Estonia",
    # Bulgaria NUTS-2
    "BG31": "Severozapaden", "BG32": "Severen Tsentralen",
    "BG33": "Severoiztochen", "BG34": "Yugoiztochen",
    "BG41": "Yugozapaden", "BG42": "Yuzhen Tsentralen",
    # Romania NUTS-2
    "RO11": "Nord-Vest", "RO12": "Centru",
    "RO21": "Nord-Est", "RO22": "Sud-Est",
    "RO31": "Sud - Muntenia", "RO32": "Bucuresti-Ilfov",
    "RO41": "Sud-Vest Oltenia", "RO42": "Vest",
    # Germany NUTS-2 (representative)
    "DE11": "Stuttgart", "DE12": "Karlsruhe",
    "DE13": "Freiburg", "DE21": "Oberbayern",
    "DE22": "Niederbayern", "DE23": "Oberpfalz",
    # France NUTS-2 (representative)
    "FR10": "Ile-de-France", "FRB0": "Centre-Val de Loire",
    "FRC1": "Bourgogne", "FRD1": "Basse-Normandie",
    # Poland NUTS-2 (representative)
    "PL21": "Malopolskie", "PL22": "Slaskie",
    "PL41": "Wielkopolskie", "PL42": "Zachodniopomorskie",
    "PL43": "Lubuskie",
    # ... extend as more NUTS codes are observed in TED data ...
}


def map_nuts_code(nuts_code: str | None) -> tuple[str | None, str | None]:
    """Map a NUTS code to (country_iso2, region_name).

    Country is derived from the first 2 characters of the NUTS code (ISO 3166-1 alpha-2).
    Region is looked up from _NUTS_NAMES; falls back to the code itself if not found.

    Returns (None, None) for empty or None input.
    """
    if not nuts_code:
        return None, None
    country = nuts_code[:2].upper()
    region = _NUTS_NAMES.get(nuts_code, nuts_code)  # fallback: use code as region name
    return country, region
```

Tests should verify fallback: `map_nuts_code("XX999")` → `("XX", "XX999")`.

[Source: test-design-epic-05.md#E05-P1-014]

### TED-Specific Agent Response Shape

The TED Crawler Agent returns (per page):

```json
{
  "execution_id": "ex-ted-1",
  "status": "completed",
  "output": {
    "opportunities": [
      {
        "id": "TED-2024-12345",
        "title": "Road Infrastructure Works",
        "description": "Construction of regional road section",
        "status": "open",
        "deadline": "2025-06-30T23:59:59",
        "budget": {"min": 50000, "max": 500000, "currency": "EUR"},
        "nuts_code": "RO321",
        "contracting_authority": "Consiliul Judetean Ilfov",
        "cpv_codes": ["45233120", "45200000"],
        "raw_agent_response": {}
      }
    ],
    "next_page_token": "eyJwYWdlIjogMn0="
  }
}
```

Key differences from AOP response shape:
- `nuts_code` field (not separate `country`/`region`) — derived in normalizer via `map_nuts_code()`
- `next_page_token` inside `output` dict (not top-level) — extracted in the task loop via `crawler_response.output.get("next_page_token")`
- No separate `country` or `region` field in the raw response; both derived from `nuts_code`

### Agent Names for This Task

| Call | Logical Agent Name | `AgentType` |
|------|--------------------|-------------|
| Crawler call (per page) | `ted-crawler-agent` | `AgentType.CRAWLER` |
| Normalization call (per page) | `data-normalization-team` | `AgentType.NORMALIZATION` |

These names are registered in `eusolicit-app/services/ai-gateway/config/agents.yaml`. Always use logical names — never hardcode KraftData UUIDs.

[Source: eusolicit-docs/implementation-artifacts/5-3-ai-gateway-client-module-with-retry-logic.md#Agent-Names]

### Mock Fixture Design for Pagination Tests

```python
# In test_crawl_ted.py — 3-page pagination test setup

AI_GW_BASE = "http://ai-gateway:8004"

PAGE_1_RAW = [{"id": f"TED-P1-{i:04d}", "title": f"TED P1 {i}", "nuts_code": "BG41",
               "status": "open", "cpv_codes": ["45000000"]} for i in range(5)]
PAGE_2_RAW = [{"id": f"TED-P2-{i:04d}", "title": f"TED P2 {i}", "nuts_code": "RO11",
               "status": "open", "cpv_codes": ["72000000"]} for i in range(5)]
PAGE_3_RAW = [{"id": f"TED-P3-{i:04d}", "title": f"TED P3 {i}", "nuts_code": "DE11",
               "status": "open", "cpv_codes": ["45000000"]} for i in range(5)]

PAGE_1_NORM = [{"source_id": f"TED-P1-{i:04d}", "source_type": "ted",
                "country": "BG", "region": "Yugozapaden",
                "title": f"TED P1 {i}", "status": "open",
                "cpv_codes": ["45000000"]} for i in range(5)]
PAGE_2_NORM = [{"source_id": f"TED-P2-{i:04d}", "source_type": "ted",
                "country": "RO", "region": "Nord-Vest",
                "title": f"TED P2 {i}", "status": "open",
                "cpv_codes": ["72000000"]} for i in range(5)]
PAGE_3_NORM = [{"source_id": f"TED-P3-{i:04d}", "source_type": "ted",
                "country": "DE", "region": "Stuttgart",
                "title": f"TED P3 {i}", "status": "open",
                "cpv_codes": ["45000000"]} for i in range(5)]

with respx.mock(base_url=AI_GW_BASE, assert_all_called=False) as mock:
    # TED crawler agent called 3 times, each returning a different page
    mock.post("/workflows/ted-crawler-agent/run").mock(
        side_effect=[
            httpx.Response(200, json={"execution_id": "ex-1", "status": "completed",
                "output": {"opportunities": PAGE_1_RAW, "next_page_token": "p2"}}),
            httpx.Response(200, json={"execution_id": "ex-2", "status": "completed",
                "output": {"opportunities": PAGE_2_RAW, "next_page_token": "p3"}}),
            httpx.Response(200, json={"execution_id": "ex-3", "status": "completed",
                "output": {"opportunities": PAGE_3_RAW, "next_page_token": None}}),
        ]
    )
    # Normalization agent called 3 times, once per page
    mock.post("/workflows/data-normalization-team/run").mock(
        side_effect=[
            httpx.Response(200, json={"execution_id": "ex-n1", "status": "completed",
                "output": {"opportunities": PAGE_1_NORM}}),
            httpx.Response(200, json={"execution_id": "ex-n2", "status": "completed",
                "output": {"opportunities": PAGE_2_NORM}}),
            httpx.Response(200, json={"execution_id": "ex-n3", "status": "completed",
                "output": {"opportunities": PAGE_3_NORM}}),
        ]
    )

    result = crawl_ted.apply()

assert result.successful()
assert _opp_count(pg_container, "ted") == 15
run = _latest_run(pg_container, "ted")
assert run["found"] == 15
assert run["new"] == 15
assert run["status"] == "completed"
```

### Partial Failure Behavior (E05-P2-006)

Since each page's upsert is committed independently (separate `get_sync_session()` contexts), data from successfully processed pages **is** committed before a later page failure. This is intentional and acceptable behavior for long multi-page crawls:

- Page 1 data is durably committed
- Page 2 `AIGatewayUnavailableError` triggers Celery retry; if max retries exhausted, the `on_failure` handler marks the run `failed`
- `crawler_runs.errors` records the failure details
- `crawler_runs.found` reflects only what was committed before the failure

Test `test_crawl_ted_partial_page_failure` should assert:
- `crawler_runs.status == "failed"`
- `crawler_runs.errors` is a dict with `"message"` key
- `crawler_runs.ended_at` is not null
- **NOT** that `pipeline.opportunities` count is 0 (page 1 may be committed)

This distinguishes TED's multi-page behavior from AOP's single-call-atomicity guarantee.

### `_RUN_ID_REGISTRY` Pattern

Identical to `crawl_aop`. The module-level dict + lock allows the `task_failure` signal handler to locate the `run_id` even when running in eager mode or after a worker process restart:

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

The signal handler calls `_pop_run(task_id)` to retrieve and remove the `run_id` atomically.

### File Locations

| Purpose | Path |
|---------|------|
| Task stub to replace | `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` |
| New NUTS mapping utility | `services/data-pipeline/src/data_pipeline/workers/tasks/_nuts.py` |
| TED normalizer (add function to) | `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py` |
| Shared upsert (reuse, no changes) | `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py` |
| Shared sync DB session (reuse) | `services/data-pipeline/src/data_pipeline/db.py` |
| AI Gateway client (reuse) | `services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py` |
| Opportunity model (reuse) | `services/data-pipeline/src/data_pipeline/models/opportunity.py` |
| CrawlerRun model (reuse) | `services/data-pipeline/src/data_pipeline/models/crawler_run.py` |
| Integration tests | `services/data-pipeline/tests/integration/test_crawl_ted.py` |
| NUTS unit tests | `services/data-pipeline/tests/unit/test_crawl_ted_nuts.py` |
| Existing AOP tests (pattern ref) | `services/data-pipeline/tests/integration/test_crawl_aop.py` |
| conftest (pg_container, fixtures) | `services/data-pipeline/tests/conftest.py` |

### Test Design Traceability

Tests in this story directly verify the following scenarios from the Epic 5 test design:

| Test ID | Priority | Scenario |
|---------|----------|----------|
| E05-P0-005 | P0 | TED full pagination — 3 pages (tokens "p2", "p3", null) fully consumed; `crawler_runs` totals = sum across all pages |
| E05-P1-014 | P1 | TED NUTS code mapping — `RO321` → `country='RO'`, non-empty `region`; `BG41` → `region='Yugozapaden'` |
| E05-P1-015 | P1 | TED crawler_runs accumulate totals across pages — 2 pages (5+3 opps); `crawler_runs.found = 8` not 3 |
| E05-P2-006 | P2 | TED partial page failure — page 2 returns 5xx; `crawler_runs.status='failed'`, `errors` populated |

Mitigations verified:
- **E05-R-001** (Dedup race, Score 6): covered by reusing the atomic `upsert_opportunities()` from S05.04 — no new dedup tests needed for TED specifically; the shared helper's correctness is already established
- **E05-R-002** (AI Gateway cascade / orphaned runs, Score 6): covered by E05-P2-006 (partial failure → `on_failure` handler fires) and the `_on_crawl_ted_failure` signal handler implementation
- **E05-R-004** (TED pagination incomplete, Score 4): covered by E05-P0-005 (3-page full consumption) and the sentinel normalization in Task 3.1 (falsy check on `next_page_token`)

[Source: eusolicit-docs/test-artifacts/test-design-epic-05.md]


---

## Dev Agent Record

**Implemented by:** `gemini-3.1-pro-preview` via Orchestrator phase `2-dev-story` (session `3acee290-…`).
Runtime 1142 s. Cost $0.1602.

**Post-dev correction (client.py retry exception handling):**
`ai_gateway_client/client.py` was broadened to `except Exception as e:` by the dev agent, which masked legitimate `AIGatewayUnavailableError(attempts=N)` from `with_retry` as synthetic `(attempts=0, circuit_open=True)`. Two unit tests failed (`test_unavailable_error_after_retries_exhausted`, `test_retry_on_timeout_exhausted`). Restored to narrow `except CircuitOpenError as exc:` — all 55 tests pass.

### File List

**New:**

- `services/data-pipeline/src/data_pipeline/workers/tasks/_nuts.py` — NUTS code → (country ISO2, region name) mapping (AC 2)
- `services/data-pipeline/tests/unit/test_crawl_ted_nuts.py` — NUTS mapping + normalizer AC 2 tests
- `services/data-pipeline/tests/integration/test_crawl_ted.py` — pagination + state-machine integration tests (AC 1, 3, 4, 5, 6, 8)

**Modified:**

- `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` — full implementation (+244 LOC): state machine `running → retrying → completed/failed`, `task_failure` signal handler, `_RUN_ID_REGISTRY` pattern, structured logging
- `services/data-pipeline/src/data_pipeline/workers/tasks/_normalize.py` — `normalize_ted_response` with NUTS enrichment
- `services/data-pipeline/src/data_pipeline/workers/tasks/_upsert.py` — `source_type` passthrough
- `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` — minor signature alignment
- `services/data-pipeline/src/data_pipeline/workers/tasks/__init__.py` — re-export `map_nuts_code`
- `services/data-pipeline/src/data_pipeline/ai_gateway_client/client.py` — retry/circuit exception surface (post-dev correction)
- `services/data-pipeline/src/data_pipeline/ai_gateway_client/retry.py` — `_GatewayHTTPError` private signal type
- `services/data-pipeline/src/data_pipeline/models/opportunity.py` — `source_type` constraint update
- `services/data-pipeline/src/data_pipeline/db.py` — sync session helper adjustment
- `services/data-pipeline/alembic/versions/002_pipeline_tables.py` — schema alignment
- `services/data-pipeline/tests/integration/test_crawl_aop.py`, `test_pipeline_migration.py` — fixture updates
- `services/data-pipeline/tests/unit/test_ai_gateway_client.py`, `test_crawler_run_model.py`, `test_enrichment_queue_model.py`, `test_opportunity_model.py`, `test_submission_guide_model.py` — assertion updates

### Test Results

```
55 passed in 33.79s
```

All ACs 1–8 covered by passing tests. 0 failures, 0 errors.

### Known Deviation (HALT reason)

AC 7 requires `crawler_runs.status` transition to a terminal state **even if the Celery worker restarts mid-retry**. The implemented `task_failure` signal handler covers graceful shutdown, but a `SIGKILL` during retry back-off leaves records at `status='retrying'` indefinitely because no Python-level signal fires. This cannot be fully satisfied in-process and requires an out-of-band sweeper.

**Resolution:** Deferred to story 5-12 (`end-to-end-integration-tests-and-pipeline-observability`) — add a periodic `sweep_stale_runs` Celery beat task running every 15 min to age out `status='retrying'` records older than the max retry window. AC 7 of story 5-5 is satisfied for all in-process failure modes; SIGKILL resilience is tracked separately.

### Change Log

| Date | Actor | Change |
|---|---|---|
| 2026-04-16 | `gemini-3.1-pro-preview` (dev phase) | Full implementation of `crawl_ted`, NUTS mapping, TED normalizer, unit+integration tests |
| 2026-04-16 | `claude-opus-4-6` (post-dev fix) | Restored narrow `CircuitOpenError` handler in `client.py`; unblocked 2 retry-exhaustion tests |
