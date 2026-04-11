# Story 12.1: Analytics Materialized Views & Refresh Infrastructure

Status: in-progress

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer on the EU Solicit platform**,
I want **PostgreSQL materialized views defined for each analytics domain and Celery Beat tasks configured in the Notification Service to refresh them on schedule**,
so that **the analytics dashboard API stories (S12.02–S12.08) have a performant, non-blocking, company-scoped data foundation ready to query without introducing read-lock contention**.

## Acceptance Criteria

1. **AC1** — Materialized views exist in the `client` schema for all 5 analytics domains: `mv_market_intelligence`, `mv_roi_tracker`, `mv_team_performance`, `mv_competitor_intelligence`, and `mv_usage_consumption`. Each view includes `company_id` as a grouped/selectable column and is populated with data (`WITH DATA`) on creation.

2. **AC2** — Celery Beat schedule in the Notification Service is configured: daily refresh (2:00–2:45 AM staggered) for `mv_market_intelligence`, `mv_roi_tracker`, `mv_team_performance`, `mv_competitor_intelligence`; hourly refresh (minute `:00`) for `mv_usage_consumption`.

3. **AC3** — `REFRESH MATERIALIZED VIEW CONCURRENTLY` is used in every refresh task. Each materialized view has a `UNIQUE INDEX` on its natural composite key (required for CONCURRENT refresh to work — CONCURRENT refresh fails without a unique index). Dashboard reads are never blocked during refresh.

4. **AC4** — Alembic migration `011_analytics_materialized_views.py` in `services/client-api` applies cleanly (`upgrade head`) after migration `010` and is fully reversible (`downgrade` to `010` drops all 5 views, their indexes, and reverts grants).

5. **AC5** — A management command `scripts/refresh_analytics_views.py` (or equivalent CLI script) triggers an ad-hoc full refresh of one or all materialized views via Celery. Runnable as `python scripts/refresh_analytics_views.py [--view VIEW_NAME | --all]`.

## Tasks / Subtasks

- [ ] Task 1: Alembic migration 011 — create materialized views in `client` schema (AC: 1, 3, 4)
  - [ ] 1.1 Create `services/client-api/alembic/versions/011_analytics_materialized_views.py` with `down_revision = "010"`
  - [ ] 1.2 `upgrade()` — grant `USAGE` on `client` schema + `SELECT` on the 6 required tables to `notification_role` (see Dev Notes for exact tables; needed so `notification_role` can run the views' SELECT queries on REFRESH)
  - [ ] 1.3 `upgrade()` — grant `USAGE` on `pipeline` schema + `SELECT` on `pipeline.opportunities` to `notification_role`
  - [ ] 1.4 `upgrade()` — create `mv_market_intelligence` materialized view (see Dev Notes for column spec and source tables) — `WITH DATA`
  - [ ] 1.5 `upgrade()` — create `UNIQUE INDEX uq_mv_market_intelligence` on `(company_id, sector, country, month)` for CONCURRENT refresh support; create `INDEX ix_mv_market_intelligence_company_id` on `(company_id)` for filter queries
  - [ ] 1.6 `upgrade()` — create `mv_roi_tracker` materialized view (see Dev Notes) — `WITH DATA`
  - [ ] 1.7 `upgrade()` — create `UNIQUE INDEX uq_mv_roi_tracker` on `(company_id, proposal_id)`; create `INDEX ix_mv_roi_tracker_company_id` on `(company_id)`
  - [ ] 1.8 `upgrade()` — create `mv_team_performance` materialized view (see Dev Notes) — `WITH DATA`
  - [ ] 1.9 `upgrade()` — create `UNIQUE INDEX uq_mv_team_performance` on `(company_id, user_id, month)`; create `INDEX ix_mv_team_performance_company_id` on `(company_id)`
  - [ ] 1.10 `upgrade()` — create `mv_competitor_intelligence` materialized view (see Dev Notes) — `WITH DATA`
  - [ ] 1.11 `upgrade()` — create `UNIQUE INDEX uq_mv_competitor_intelligence` on `(company_id, competitor_name, sector)`; create `INDEX ix_mv_competitor_intelligence_company_id` on `(company_id)`
  - [ ] 1.12 `upgrade()` — create `mv_usage_consumption` materialized view (see Dev Notes) — `WITH DATA`
  - [ ] 1.13 `upgrade()` — create `UNIQUE INDEX uq_mv_usage_consumption` on `(company_id, metric_type, period_start)`; create `INDEX ix_mv_usage_consumption_company_id` on `(company_id)`
  - [ ] 1.14 `upgrade()` — transfer ownership of all 5 views to `notification_role`: `ALTER MATERIALIZED VIEW client.<view_name> OWNER TO notification_role` (× 5) — this allows notification_role to REFRESH them without needing MAINTAIN privilege
  - [ ] 1.15 `downgrade()` — drop indexes and views in reverse order (5 views + 10 indexes); revoke `notification_role` grants on `pipeline` and `client` source tables (REVOKE USAGE ON SCHEMA)

- [ ] Task 2: SQLAlchemy ORM "reflect" models for materialized views in client-api (AC: 1)
  - [ ] 2.1 Create `services/client-api/src/client_api/models/analytics_views.py` with 5 read-only `Table` definitions (use `sa.Table` or `mapped_column` with `__table_args__ = {"schema": "client"}`) — no `__tablename__` triggers DDL conflicts; use `autoload_with` pattern or define columns explicitly
  - [ ] 2.2 ORM models must NOT define `CreateTable` in metadata (views, not tables); use `info={"is_view": True}` on the table arg to flag for Alembic's `include_object` filter — prevents `alembic check` from flagging unmapped views
  - [ ] 2.3 Export all 5 view Table objects from `services/client-api/src/client_api/models/__init__.py`

- [ ] Task 3: Notification Service — create Celery workers infrastructure (AC: 2)
  - [ ] 3.1 Create `services/notification/src/notification/workers/__init__.py` (empty)
  - [ ] 3.2 Create `services/notification/src/notification/workers/celery_app.py` — `Celery` instance named `"notification"` with `broker` from `CELERY_BROKER_URL` env var (default: `redis://localhost:6379/0`) and `backend` from `CELERY_RESULT_BACKEND` env var; `task_serializer="json"`, `accept_content=["json"]`, `timezone="UTC"`, `enable_utc=True`
  - [ ] 3.3 Create `services/notification/src/notification/workers/tasks/__init__.py` (empty)
  - [ ] 3.4 Create `services/notification/src/notification/workers/beat_schedule.py` — `app.conf.beat_schedule` dict with 5 entries (see Dev Notes for full schedule spec)
  - [ ] 3.5 Update `services/notification/pyproject.toml` `[project.scripts]` or `[tool.setuptools]` to expose the Celery app entry point: `notification-worker = "notification.workers.celery_app:celery"` (for `celery -A notification.workers.celery_app worker` invocation)

- [ ] Task 4: Refresh task implementation (AC: 2, 3)
  - [ ] 4.1 Create `services/notification/src/notification/workers/tasks/refresh_analytics_views.py`
  - [ ] 4.2 Add `DATABASE_URL` env var reading (use `NOTIFICATION_DATABASE_URL` — same PostgreSQL DSN the notification service uses, connecting as `notification_role`)
  - [ ] 4.3 Implement helper `_refresh_view(view_name: str, concurrently: bool = True)` — executes `REFRESH MATERIALIZED VIEW CONCURRENTLY client.<view_name>` using a synchronous `sqlalchemy.create_engine` connection (Celery tasks are sync by default; use `with engine.connect() as conn: conn.execute(text(...)); conn.commit()`)
  - [ ] 4.4 Implement 5 Celery tasks: `refresh_market_intelligence`, `refresh_roi_tracker`, `refresh_team_performance`, `refresh_competitor_intelligence`, `refresh_usage_consumption` — each calls `_refresh_view("<view_name>")` and logs duration via `structlog.get_logger()`
  - [ ] 4.5 Add `autoretry_for=(Exception,)`, `max_retries=2`, `retry_backoff=True`, `retry_backoff_max=60` on all 5 tasks (transient DB connection issues during REFRESH should retry with backoff, not silently drop)
  - [ ] 4.6 Bind `app.conf.beat_schedule` from `beat_schedule.py` at module import: `from notification.workers.beat_schedule import BEAT_SCHEDULE; celery.conf.beat_schedule = BEAT_SCHEDULE`

- [ ] Task 5: Management command — ad-hoc refresh script (AC: 5)
  - [ ] 5.1 Create `scripts/refresh_analytics_views.py` — argparse CLI with `--view` (choices: `market`, `roi`, `team`, `competitor`, `usage`, `all`; default: `all`) and `--sync` flag (run synchronously without Celery, useful in CI/migration contexts)
  - [ ] 5.2 `--sync` mode: directly calls `_refresh_view()` via SQLAlchemy (bypasses Celery broker)
  - [ ] 5.3 Default mode: dispatches Celery tasks via `.delay()` and waits for results with `AsyncResult.get(timeout=300)` per view
  - [ ] 5.4 Print timing and success/failure summary per view; exit code 1 on any failure

- [ ] Task 6: Unit tests — Celery task refresh logic (AC: 2, 3)
  - [ ] 6.1 Create `services/notification/tests/unit/test_refresh_analytics_views.py`
  - [ ] 6.2 Test: `_refresh_view("mv_market_intelligence")` executes `REFRESH MATERIALIZED VIEW CONCURRENTLY client.mv_market_intelligence` — mock `engine.connect()`, capture SQL string, assert correct statement
  - [ ] 6.3 Test: each of the 5 Celery tasks calls `_refresh_view` with its correct view name — mock `_refresh_view`, assert called with expected argument
  - [ ] 6.4 Test: task retries on `OperationalError` (DB connection failure) — mock `_refresh_view` to raise on first call then succeed; assert task retried and succeeded on second call
  - [ ] 6.5 Test: Beat schedule has correct task names and schedules — assert `BEAT_SCHEDULE` dict has 5 entries; verify `mv_usage_consumption` uses `crontab(minute="0")` and daily views use `crontab(hour=...)` with distinct hours

- [ ] Task 7: Integration tests — migration and view schema (AC: 1, 3, 4)
  - [ ] 7.1 Create `services/client-api/tests/integration/test_011_migration.py`
  - [ ] 7.2 Test: `alembic upgrade head` from `010` baseline succeeds (E12-DB-001)
  - [ ] 7.3 Test: all 5 materialized views exist in `pg_matviews` with `matviewschema = 'client'` after migration (E12-DB-002)
  - [ ] 7.4 Test: each view has a unique index — query `pg_indexes` for each `uq_mv_*` index name; verify `indisunique = true` (E12-DB-003)
  - [ ] 7.5 Test: each view has a `company_id` index — query `pg_indexes` for each `ix_mv_*_company_id` (E12-DB-004)
  - [ ] 7.6 Test: `notification_role` is the owner of all 5 views — query `pg_matviews` WHERE `matviewowner = 'notification_role'` for each view (E12-DB-005)
  - [ ] 7.7 Test: `REFRESH MATERIALIZED VIEW CONCURRENTLY client.mv_usage_consumption` (and one other view) succeeds — execute as `notification_role` connection, assert no exception (E12-DB-006)
  - [ ] 7.8 Test: concurrent read during refresh returns data without blocking — launch REFRESH in a thread, query the view in the main thread simultaneously; assert query returns within 2 seconds (E12-DB-007)
  - [ ] 7.9 Test: `alembic downgrade 010` drops all 5 views and their indexes; verify `pg_matviews` is empty for these view names (E12-DB-008)
  - [ ] 7.10 Test: `alembic check` produces no pending autogenerate changes after migration applied (ORM models in sync — E12-DB-009)

## Dev Notes

### Architecture Decision: Views in `client` Schema, Owned by `notification_role`

**Why `client` schema?** Materialized views are queried by Client API analytics endpoints (S12.02–S12.08). Placing them in `client` schema means `client_api_role` reads them via the existing default privileges — no extra grants needed on the read side.

**Why transfer ownership to `notification_role`?** The Notification Service's Celery Beat task must `REFRESH MATERIALIZED VIEW CONCURRENTLY`. PostgreSQL 16's `MAINTAIN` privilege would be an alternative, but transferring ownership is simpler and avoids future issues if the DB version changes. The migration uses `ALTER MATERIALIZED VIEW client.<view_name> OWNER TO notification_role` immediately after `CREATE MATERIALIZED VIEW`.

**Cross-schema grants in migration:** `notification_role` currently has no access to the `client` or `pipeline` schemas (the init SQL only grants `notification_role` on `notification` schema + `shared` SELECT). The migration must add: `GRANT USAGE ON SCHEMA client TO notification_role` + `SELECT` on the 6 specific source tables, and `GRANT USAGE ON SCHEMA pipeline TO notification_role` + `SELECT ON pipeline.opportunities`. These are additive grants — the downgrade REVOKEs them.

[Source: eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql#2b]

### CRITICAL: Company-ID Scoping in Every View (E12-R-001, Risk Score 6)

**Every materialized view MUST have `company_id` as a GROUP BY column and as a selectable output column.** This is the primary defence against cross-tenant analytics leakage (E12-R-001, highest risk in this epic, score 6). Dashboard API queries (S12.02–S12.08) MUST always include `WHERE company_id = :company_id` when reading from these views.

The P0 test in `e2e/specs/analytics/cross-tenant-isolation.api.spec.ts` seeds Company A and Company B, authenticates as A, queries all 6 analytics domains, and asserts zero Company B records are returned. That test depends entirely on both (a) the view having `company_id` and (b) the API query filtering by it.

[Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#E12-R-001, test-artifacts/atdd-checklist-e12-p0.md]

### REFRESH CONCURRENTLY Requirement — Unique Index is Mandatory

`REFRESH MATERIALIZED VIEW CONCURRENTLY` requires a unique index on the view. Without it, PostgreSQL falls back to a blocking full-table lock refresh (E12-R-006). The unique index must be on the natural composite key that uniquely identifies a row in the view. **Do NOT skip the unique index** — CONCURRENT refresh silently fails (reverts to blocking mode) if the unique index is absent or not maintained.

Verify after migration: `SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'mv_<name>' AND schemaname = 'client'` should show both a `uq_mv_*` (unique) and `ix_mv_*_company_id` (non-unique) index.

[Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#E12-R-006]

### Migration File Location and Chain

- **File**: `services/client-api/alembic/versions/011_analytics_materialized_views.py`
- **Revision chain**: `down_revision = "010"` (010_proposals.py is the current head)
- **Run by**: `migration_role` (has DDL + full CRUD on all schemas + GRANT privileges)
- **Alembic env.py**: Already configured in client-api — do NOT modify

[Source: eusolicit-app/services/client-api/alembic/versions/010_proposals.py]

### Materialized View SQL Specifications

**Inspect actual column names** before writing SQL: run `\d client.<table_name>` against the dev DB (after applying migrations 001–010) to verify column names. Architecture provides entity fields for `bid_preparation_logs`; other tables (bid_outcomes, competitor_records, usage_meters) are not yet fully defined in migration files — infer from architecture entity descriptions below.

#### Source Table Column References

| Table | Schema | Key Columns (from architecture) |
|-------|--------|----------------------------------|
| `bid_decisions` | `client` | `id`, `company_id`, `opportunity_id`, `decision` (bid/no_bid), `created_at` |
| `bid_outcomes` | `client` | `id`, `company_id`, `proposal_id`, `opportunity_id`, `status` (won/lost/withdrawn), `contract_value_eur`, `won_at`, `created_at` |
| `bid_preparation_logs` | `client` | `id`, `proposal_id` (FK), `user_id` (FK), `activity_type`, `hours` (decimal), `cost_eur` (decimal, nullable), `logged_at` |
| `proposals` | `client` | `id`, `company_id`, `opportunity_id`, `created_by`, `status`, `created_at` |
| `competitor_records` | `client` | `id`, `company_id`, `competitor_name`, `sector`, `bid_count` (integer), `estimated_win_rate` (decimal 0–1), `avg_bid_value_eur` (decimal nullable), `last_seen_at` |
| `usage_meters` | `shared` | `id`, `company_id`, `metric_type` (ai_summaries/proposal_drafts/compliance_checks), `consumed` (integer), `limit_value` (integer), `period_start` (date), `period_end` (date), `updated_at` |
| `opportunities` | `pipeline` | `id`, `title`, `cpv_codes` (text[]), `country` (varchar), `contracting_authority` (text), `estimated_value_eur` (decimal), `deadline_at` (timestamptz) |

**If any column does not exist:** check the latest migration for that table and use the actual column name. Do not invent column names.

#### `mv_market_intelligence` — Daily Refresh

Data: market-level aggregates from opportunities the company has engaged with (bid decisions).

```sql
CREATE MATERIALIZED VIEW client.mv_market_intelligence AS
SELECT
    bd.company_id,
    COALESCE(o.cpv_codes[1], 'unknown')                   AS sector,
    COALESCE(o.country, 'unknown')                         AS country,
    date_trunc('month', o.deadline_at)::date               AS month,
    COUNT(DISTINCT o.id)                                   AS opportunity_count,
    AVG(o.estimated_value_eur)                             AS avg_contract_value_eur,
    SUM(o.estimated_value_eur)                             AS total_value_eur,
    o.contracting_authority                                AS authority_name
FROM client.bid_decisions bd
JOIN pipeline.opportunities o ON o.id = bd.opportunity_id
WHERE bd.company_id IS NOT NULL
GROUP BY
    bd.company_id,
    o.cpv_codes[1],
    o.country,
    date_trunc('month', o.deadline_at),
    o.contracting_authority
WITH DATA;
```

Unique index: `(company_id, sector, country, month, authority_name)` — add `authority_name` if needed for uniqueness; alternatively define a surrogate row hash. Simplest approach: `(company_id, sector, country, month)` if `authority_name` is not needed in the key (verify that the GROUP BY produces unique rows for this 4-column key; if not, add `authority_name` to the unique index).

#### `mv_roi_tracker` — Daily Refresh

Data: per-company, per-proposal investment vs. outcome.

```sql
CREATE MATERIALIZED VIEW client.mv_roi_tracker AS
SELECT
    p.company_id,
    p.id                                                    AS proposal_id,
    date_trunc('month', p.created_at)::date                AS month,
    COALESCE(SUM(bpl.hours * 50 + COALESCE(bpl.cost_eur, 0)), 0)
                                                           AS total_invested_eur,
    COALESCE(MAX(CASE WHEN bo.status = 'won' THEN bo.contract_value_eur ELSE 0 END), 0)
                                                           AS total_won_eur,
    CASE
        WHEN COALESCE(SUM(bpl.hours * 50 + COALESCE(bpl.cost_eur, 0)), 0) = 0 THEN 0
        ELSE ROUND(
            (COALESCE(MAX(CASE WHEN bo.status = 'won' THEN bo.contract_value_eur ELSE 0 END), 0) -
             COALESCE(SUM(bpl.hours * 50 + COALESCE(bpl.cost_eur, 0)), 0)) /
             COALESCE(SUM(bpl.hours * 50 + COALESCE(bpl.cost_eur, 0)), 1) * 100,
            2)
    END                                                    AS roi_pct
FROM client.proposals p
LEFT JOIN client.bid_preparation_logs bpl ON bpl.proposal_id = p.id
LEFT JOIN client.bid_outcomes bo ON bo.proposal_id = p.id
GROUP BY p.company_id, p.id, date_trunc('month', p.created_at)
WITH DATA;
```

Note: `hours * 50` is a placeholder rate (EUR/hour). This should be configurable or replaced with actual logged `cost_eur`. Validate with product before hardcoding.

Unique index: `(company_id, proposal_id)`.

#### `mv_team_performance` — Daily Refresh

Data: per-company, per-user, per-month performance metrics.

```sql
CREATE MATERIALIZED VIEW client.mv_team_performance AS
SELECT
    p.company_id,
    bpl.user_id,
    date_trunc('month', bpl.logged_at)::date               AS month,
    COUNT(DISTINCT p.id)                                   AS bids_submitted,
    COUNT(DISTINCT CASE WHEN bo.status = 'won' THEN p.id END)
                                                           AS win_count,
    CASE
        WHEN COUNT(DISTINCT p.id) = 0 THEN 0
        ELSE ROUND(
            COUNT(DISTINCT CASE WHEN bo.status = 'won' THEN p.id END)::numeric /
            COUNT(DISTINCT p.id) * 100, 2)
    END                                                    AS win_rate,
    COALESCE(SUM(bpl.hours), 0)                            AS total_preparation_hours,
    CASE
        WHEN COUNT(DISTINCT p.id) = 0 THEN 0
        ELSE ROUND(COALESCE(SUM(bpl.hours), 0) / COUNT(DISTINCT p.id), 2)
    END                                                    AS avg_preparation_hours,
    COUNT(DISTINCT p.id)                                   AS proposals_generated
FROM client.bid_preparation_logs bpl
JOIN client.proposals p ON p.id = bpl.proposal_id
LEFT JOIN client.bid_outcomes bo ON bo.proposal_id = p.id
GROUP BY p.company_id, bpl.user_id, date_trunc('month', bpl.logged_at)
WITH DATA;
```

Unique index: `(company_id, user_id, month)`.

#### `mv_competitor_intelligence` — Daily Refresh

Data: per-company competitor intelligence snapshot (from competitor_records).

```sql
CREATE MATERIALIZED VIEW client.mv_competitor_intelligence AS
SELECT
    company_id,
    competitor_name,
    COALESCE(sector, 'unknown')                            AS sector,
    SUM(bid_count)                                         AS total_bid_count,
    AVG(estimated_win_rate)                                AS avg_win_rate,
    AVG(avg_bid_value_eur)                                 AS avg_bid_value_eur
FROM client.competitor_records
WHERE company_id IS NOT NULL
GROUP BY company_id, competitor_name, sector
WITH DATA;
```

Unique index: `(company_id, competitor_name, sector)`.

#### `mv_usage_consumption` — Hourly Refresh

Data: current billing-period consumption vs. tier limits per company per metric type.

```sql
CREATE MATERIALIZED VIEW client.mv_usage_consumption AS
SELECT
    company_id,
    metric_type,
    period_start,
    period_end,
    consumed,
    limit_value,
    GREATEST(limit_value - consumed, 0)                    AS remaining,
    updated_at
FROM shared.usage_meters
WHERE company_id IS NOT NULL
WITH DATA;
```

Unique index: `(company_id, metric_type, period_start)`.

**Note**: `shared.usage_meters` is in the `shared` schema. `notification_role` already has `SELECT` on `shared` tables (granted in init SQL). No additional grant needed for this view's REFRESH.

### Notification Service Workers Directory — Does Not Yet Exist

The Notification Service currently has only a stub FastAPI `main.py`. The `workers/` directory does NOT exist and must be created from scratch by this story. Architecture target structure:

```
services/notification/src/notification/
├── workers/
│   ├── __init__.py
│   ├── celery_app.py
│   ├── beat_schedule.py
│   └── tasks/
│       ├── __init__.py
│       └── refresh_analytics_views.py
└── main.py   ← existing stub, DO NOT MODIFY
```

[Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#7.5-Notification-Service]

### Celery Beat Schedule Specification

```python
# services/notification/src/notification/workers/beat_schedule.py
from celery.schedules import crontab

BEAT_SCHEDULE = {
    "refresh-market-intelligence-daily": {
        "task": "notification.workers.tasks.refresh_analytics_views.refresh_market_intelligence",
        "schedule": crontab(hour="2", minute="0"),       # 02:00 UTC daily
    },
    "refresh-roi-tracker-daily": {
        "task": "notification.workers.tasks.refresh_analytics_views.refresh_roi_tracker",
        "schedule": crontab(hour="2", minute="15"),      # 02:15 UTC daily
    },
    "refresh-team-performance-daily": {
        "task": "notification.workers.tasks.refresh_analytics_views.refresh_team_performance",
        "schedule": crontab(hour="2", minute="30"),      # 02:30 UTC daily
    },
    "refresh-competitor-intelligence-daily": {
        "task": "notification.workers.tasks.refresh_analytics_views.refresh_competitor_intelligence",
        "schedule": crontab(hour="2", minute="45"),      # 02:45 UTC daily
    },
    "refresh-usage-consumption-hourly": {
        "task": "notification.workers.tasks.refresh_analytics_views.refresh_usage_consumption",
        "schedule": crontab(minute="0"),                 # :00 every hour
    },
}
```

Stagger daily refreshes by 15 minutes to avoid DB contention. Usage is hourly because it's used for live tier-limit checks; others are daily aggregates.

[Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#13.3-Analytics-Computation-Model]

### Celery App Configuration Reference

Notification service env vars (add to `services/notification/` docker-compose section and `.env.example`):
- `CELERY_BROKER_URL` — Redis connection string (e.g., `redis://redis:6379/0`)
- `CELERY_RESULT_BACKEND` — Redis connection string for results (e.g., `redis://redis:6379/1`)
- `NOTIFICATION_DATABASE_URL` — PostgreSQL DSN as `notification_role` (e.g., `postgresql+psycopg2://notification_role:notification_password@postgres:5432/eusolicit`)

The synchronous `psycopg2` driver is required for Celery tasks (async `asyncpg` does not work in sync Celery workers). `notification` service already has `psycopg2-binary>=2.9` in pyproject.toml.

[Source: eusolicit-app/services/notification/pyproject.toml]

### Alembic Migration Pattern for Materialized Views

Alembic does not have built-in support for materialized views. Use raw SQL via `op.execute()`:

```python
def upgrade() -> None:
    # Step 1: Grant cross-schema access to notification_role
    op.execute("GRANT USAGE ON SCHEMA client TO notification_role")
    op.execute("GRANT SELECT ON client.bid_decisions TO notification_role")
    op.execute("GRANT SELECT ON client.bid_outcomes TO notification_role")
    op.execute("GRANT SELECT ON client.bid_preparation_logs TO notification_role")
    op.execute("GRANT SELECT ON client.proposals TO notification_role")
    op.execute("GRANT SELECT ON client.competitor_records TO notification_role")
    op.execute("GRANT USAGE ON SCHEMA pipeline TO notification_role")
    op.execute("GRANT SELECT ON pipeline.opportunities TO notification_role")

    # Step 2: Create views
    op.execute("""
        CREATE MATERIALIZED VIEW client.mv_market_intelligence AS
        ... (full SQL)
        WITH DATA
    """)

    # Step 3: Unique index (required for CONCURRENT refresh)
    op.execute("CREATE UNIQUE INDEX uq_mv_market_intelligence ON client.mv_market_intelligence (company_id, sector, country, month)")
    op.execute("CREATE INDEX ix_mv_market_intelligence_company_id ON client.mv_market_intelligence (company_id)")

    # Step 4: Transfer ownership to notification_role
    op.execute("ALTER MATERIALIZED VIEW client.mv_market_intelligence OWNER TO notification_role")

    # ... repeat for remaining 4 views


def downgrade() -> None:
    # Drop in reverse order
    op.execute("DROP MATERIALIZED VIEW IF EXISTS client.mv_market_intelligence CASCADE")
    # ... remaining 4 views

    # Revoke cross-schema grants
    op.execute("REVOKE SELECT ON client.bid_decisions FROM notification_role")
    # ... remaining revokes
    op.execute("REVOKE USAGE ON SCHEMA client FROM notification_role")
    op.execute("REVOKE SELECT ON pipeline.opportunities FROM notification_role")
    op.execute("REVOKE USAGE ON SCHEMA pipeline FROM notification_role")
```

Do NOT use `op.create_table()` for views — that generates a regular table DDL.

### ORM "View" Model — Alembic autogenerate Filter

Alembic autogenerate will try to drop materialized views because they are not in the ORM metadata. Add `is_view` info to the `Table` object and update `env.py` to skip them:

```python
# In 011 migration: views are created via raw SQL, not ORM
# In client-api/alembic/env.py, update include_object to exclude views:
def include_object(object, name, type_, reflected, compare_to):
    if type_ == "table" and reflected and compare_to is None:
        return False  # skip auto-drop of reflected-only objects (views)
    return True
```

Check if `env.py` already has an `include_object` filter (likely yes, from earlier stories). If so, verify it handles reflected materialized views correctly.

### Test Data Requirements for P0/P1 Tests (from ATDD Checklist)

The following factories need to be added to `packages/eusolicit-test-utils/src/eusolicit_test_utils/factories.py` for S12.02+ stories. Document them here as prerequisite for downstream test development:

```python
def AnalyticsDataFactory(company_id: str, **overrides) -> dict:
    """Seed materialized view source records for a company (bid_decisions, bid_outcomes,
    bid_preparation_logs, competitor_records, usage_meters) for analytics test scenarios."""
    ...

def CrossTenantPairFactory() -> tuple[dict, dict]:
    """Return two companies (A, B) with seeded analytics data in all 5 view domains.
    Used for E12-R-001 cross-tenant isolation P0 tests."""
    ...
```

[Source: eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md#Fixture-Needs]

### Test Coverage Alignment (from Test Design Epic 12)

This story directly enables:

**P0 prerequisites (from atdd-checklist-e12-p0.md):**
- `cross-tenant-isolation.api.spec.ts` (2 tests): materialized views must exist with company_id scoping before these tests can go GREEN
- `tier-gate-enforcement.api.spec.ts` (7 tests): views not directly needed but must be deployed for S12.02–S12.07 endpoints to function

**P1 tests enabled by this story (from test-design-epic-12.md):**
- "Materialized views — all 5 domains created with correct schema" → verified by integration tests E12-DB-002
- "Celery Beat — daily refresh for market/ROI/team/competitor, hourly for usage" → verified by E12-DB-006 (manual REFRESH succeeds as notification_role)
- "`REFRESH CONCURRENTLY` — reads not blocked during refresh" → verified by E12-DB-007 (concurrent thread test)

**Test commands to run after implementation:**
```bash
# Migration integration tests
cd services/client-api && pytest tests/integration/test_011_migration.py -v

# Notification unit tests
cd services/notification && pytest tests/unit/test_refresh_analytics_views.py -v

# Run all E12 P0 (currently all skipped — GREEN after S12.02–S12.08 implemented):
npx playwright test e2e/specs/analytics/cross-tenant-isolation.api.spec.ts
```

[Source: eusolicit-docs/test-artifacts/test-design-epic-12.md, eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md]

### Files to CREATE (New)

| File | Purpose |
|------|---------|
| `services/client-api/alembic/versions/011_analytics_materialized_views.py` | Migration: 5 materialized views + grants + ownership transfer |
| `services/client-api/src/client_api/models/analytics_views.py` | Read-only ORM Table definitions for the 5 views |
| `services/client-api/tests/integration/test_011_migration.py` | Integration tests E12-DB-001 through E12-DB-009 |
| `services/notification/src/notification/workers/__init__.py` | Celery workers package init |
| `services/notification/src/notification/workers/celery_app.py` | Celery application instance + beat_schedule binding |
| `services/notification/src/notification/workers/beat_schedule.py` | BEAT_SCHEDULE dict with 5 crontab entries |
| `services/notification/src/notification/workers/tasks/__init__.py` | Tasks sub-package init |
| `services/notification/src/notification/workers/tasks/refresh_analytics_views.py` | 5 Celery tasks + `_refresh_view` helper |
| `services/notification/tests/unit/test_refresh_analytics_views.py` | Unit tests for refresh tasks |
| `scripts/refresh_analytics_views.py` | Ad-hoc management command |

### Files to MODIFY (Existing)

| File | Change |
|------|--------|
| `services/client-api/src/client_api/models/__init__.py` | Export 5 view Table objects from `analytics_views.py` |
| `services/client-api/alembic/env.py` | Add/extend `include_object` filter to skip materialized view auto-drop |
| `services/notification/pyproject.toml` | Add `[project.scripts]` entry for Celery worker + Beat entry points |

### Files NOT to Touch

| File | Reason |
|------|--------|
| `infra/postgres/init/01-init-schemas-and-roles.sql` | Cross-schema grants handled in migration, not init SQL |
| `services/client-api/alembic/versions/001–010.py` | Existing migration chain — do NOT modify |
| `services/notification/src/notification/main.py` | Stub FastAPI app — not part of Celery workers |
| `services/notification/alembic/versions/001_initial.py` | Notification schema tables — unrelated to this story |

### Project Structure Notes

- Analytics domain in client-api: `services/client-api/src/client_api/domain/analytics/` (per architecture 7.1). This story creates the DB layer only. S12.02 creates the API routes in `services/client-api/src/client_api/infrastructure/api/routers/analytics.py`.
- Workers follow the same pattern as `services/data-pipeline/src/data_pipeline/workers/` — verify that service for an existing pattern to copy.
- Beat schedule is loaded in `celery_app.py`, not in a separate `celery_config.py` — keep consistent with Data Pipeline service pattern.

[Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#7.1-Client-API-Service, #7.5-Notification-Service]

### References

- [Source: eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md#S12.01]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#13.3-Analytics-Computation-Model]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#7.5-Notification-Service]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#8-Database-Strategy]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#P0, #P1, #E12-R-001, #E12-R-006]
- [Source: eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md#Infrastructure-Requirements]
- [Source: eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql]
- [Source: eusolicit-app/services/client-api/alembic/versions/010_proposals.py]
- [Source: eusolicit-app/services/notification/pyproject.toml]
- [Source: eusolicit-app/services/notification/src/notification/main.py]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

N/A — implementation completed cleanly with 19/19 unit tests passing.

### Completion Notes List

1. **Migration 011** created with all 5 materialized views, cross-schema grants for `notification_role`, unique indexes for CONCURRENT refresh, non-unique company_id indexes, and ownership transfer. Downgrade drops all 5 views with CASCADE and revokes all granted permissions.

2. **ORM models** (`analytics_views.py`) use a separate `sa.MetaData()` instance (NOT `Base.metadata`), preventing Alembic autogenerate from seeing them via ORM reflection. The `include_object` filter in `env.py` is a belt-and-suspenders guard that explicitly skips the 5 `mv_*` view names.

3. **mv_market_intelligence unique index** uses `(company_id, sector, country, month, authority_name)` — `authority_name` was added to the key because `contracting_authority` is in the GROUP BY clause, making rows potentially non-unique on 4 columns alone.

4. **Celery workers** follow the pattern specified in architecture: separate `celery_app.py`, `beat_schedule.py`, and `tasks/` sub-package. The `_engine` is lazily initialized (not at import time) to avoid DB connection attempts during test collection.

5. **Unit tests** run without a live DB or Redis broker — all 19 tests pass using mocks.

6. **Integration tests** (`test_011_migration.py`) require a live PostgreSQL DB with migration_role and notification_role credentials. They use the same patterns as existing `test_009_migration.py`.

7. **Unique index deviation note**: The story spec suggests `(company_id, sector, country, month)` for `mv_market_intelligence` but notes "add `authority_name` if needed for uniqueness." Since `contracting_authority` is in GROUP BY, it was included in the unique index to guarantee uniqueness as required for CONCURRENT refresh.

### File List

- `services/client-api/alembic/versions/011_analytics_materialized_views.py` — CREATED
- `services/client-api/src/client_api/models/analytics_views.py` — CREATED
- `services/client-api/tests/integration/test_011_migration.py` — CREATED
- `services/notification/src/notification/workers/__init__.py` — CREATED
- `services/notification/src/notification/workers/celery_app.py` — CREATED
- `services/notification/src/notification/workers/beat_schedule.py` — CREATED
- `services/notification/src/notification/workers/tasks/__init__.py` — CREATED
- `services/notification/src/notification/workers/tasks/refresh_analytics_views.py` — CREATED
- `services/notification/tests/unit/test_refresh_analytics_views.py` — CREATED
- `scripts/refresh_analytics_views.py` — CREATED
- `services/client-api/src/client_api/models/__init__.py` — MODIFIED (added 5 view exports)
- `services/client-api/alembic/env.py` — MODIFIED (added `_MATERIALIZED_VIEW_NAMES` exclusion to `include_object`)
- `services/notification/pyproject.toml` — MODIFIED (added `[project.scripts]` entry)

## Senior Developer Review

**Reviewer:** Claude Code (adversarial code review)
**Date:** 2026-04-11
**Unit Tests:** 19/19 PASS (after `notification` package install + lint fix)
**Lint:** PASS (ruff — 1 unused import fixed)

### Review Findings

- [ ] [Review][Decision] **Migration GRANT/OWNER privilege model incompatible with `migration_role`** — The upgrade executes `GRANT USAGE ON SCHEMA client TO notification_role` (line 30) and `ALTER MATERIALIZED VIEW ... OWNER TO notification_role` (lines 79, 119, 162, 192, 224). Both require privileges `migration_role` does not have per `01-init-schemas-and-roles.sql`: GRANT on a schema requires schema ownership or WITH GRANT OPTION (migration_role has neither — schemas owned by postgres superuser); ALTER OWNER requires SET ROLE to the target role (needs `GRANT notification_role TO migration_role` membership, which doesn't exist) and the target role needs CREATE on the schema (notification_role only has USAGE). Makefile confirms `MIGRATION_DB_URL` uses `migration_role`. **Violates AC4** (upgrade head applies cleanly). Root cause: the story spec prescribes this pattern but assumes more privilege than init SQL provides. **Resolution options:** (A) Add `GRANT notification_role TO migration_role` and `GRANT CREATE ON SCHEMA client TO notification_role` in init SQL — but story says "do not touch init SQL". (B) Create a prerequisite migration that adds these grants. (C) Document that migration 011 requires superuser and update Makefile accordingly. (D) Use PostgreSQL 16's `GRANT MAINTAIN ON` privilege instead of ownership transfer.
- [ ] [Review][Patch] **Downgrade DROP fails — migration_role cannot DROP views owned by notification_role** [011_analytics_materialized_views.py:230-234] — Same root cause as above. After upgrade transfers ownership to `notification_role`, the downgrade's `DROP MATERIALIZED VIEW ... CASCADE` runs as `migration_role` which is not the owner. PostgreSQL requires owner/schema-owner/superuser to DROP. Fix: once privilege issue is resolved, add `ALTER MATERIALIZED VIEW ... OWNER TO CURRENT_USER` before each DROP in downgrade.
- [x] [Review][Patch] **Unused import `call` in unit test** [test_refresh_analytics_views.py:14] — FIXED. Removed unused `call` from `unittest.mock` import. ruff F401 now clean.
- [x] [Review][Defer] **No view_name whitelist in `_refresh_view` (SQL injection defence-in-depth)** [refresh_analytics_views.py:48, scripts/refresh_analytics_views.py:109] — deferred, pre-existing pattern. All callers pass hardcoded literals or argparse-validated choices. Zero external attack surface. Adding an `ALLOWED_VIEWS` set check would improve defence-in-depth but is not actionable for this story.

### Review Summary

| Category | Count |
|----------|-------|
| Decision needed | 1 |
| Patch (applied) | 1 |
| Patch (pending — blocked on decision) | 1 |
| Deferred | 1 |
| Dismissed | 2 |

**Code quality:** Excellent. Clean separation of concerns, proper lazy initialization, comprehensive docstrings with source references, retry policies, structured logging. All 5 views match spec SQL exactly. Beat schedule matches spec exactly. ORM models use separate MetaData (not Base.metadata) which is the correct approach for view-only objects.

**Test coverage:** Unit tests (19/19 PASS) thoroughly cover: SQL statement correctness, task-to-view mapping, retry-on-error behavior, beat schedule validation. Integration tests (9 test cases) cover: migration lifecycle, view existence, index verification, ownership, REFRESH CONCURRENTLY execution, concurrent read non-blocking, downgrade, and alembic check. Integration tests could not be verified (require live PostgreSQL).

**Architecture alignment:** Implementation matches architecture docs for: Notification Service worker structure, Celery Beat schedule, materialized view placement in `client` schema, company_id scoping (E12-R-001), CONCURRENT refresh with unique indexes (E12-R-006).
