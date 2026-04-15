# Story 5.1: Pipeline Schema Migration and Model Layer

Status: done

## Story

As a **backend developer on the EU Solicit platform**,
I want **an Alembic migration that creates all four `pipeline` schema tables (`opportunities`, `submission_guides`, `crawler_runs`, `enrichment_queue`) with correct column types, indexes, and constraints, together with SQLAlchemy ORM models that expose these tables with full type annotations and a soft-delete default filter on `opportunities`**,
so that **every subsequent pipeline story (S05.02–S05.12) has a stable, type-safe data access layer and the deduplication, scoring, and cleanup logic can be built on top of a correct DB foundation from day one**.

## Acceptance Criteria

1. `alembic upgrade head` (run from `services/data-pipeline/`) creates all four tables — `pipeline.opportunities`, `pipeline.submission_guides`, `pipeline.crawler_runs`, `pipeline.enrichment_queue` — in the `pipeline` schema
2. `alembic downgrade -1` cleanly drops all four tables with no residual schema artifacts
3. Unique constraint on `(source_id, source_type)` is enforced at the database level; attempting a duplicate insert raises `IntegrityError`
4. SQLAlchemy ORM models for all four tables pass `mypy --strict` type-checking with no errors
5. Basic CRUD unit tests (create / read / update / delete) pass for the `Opportunity`, `SubmissionGuide`, `CrawlerRun`, and `EnrichmentQueueItem` models using a real PostgreSQL testcontainer
6. The `Opportunity` model's default query option filters out soft-deleted rows (`deleted_at IS NULL`) automatically — a query that includes a soft-deleted row must return only the active row without the caller passing an explicit filter

## Tasks / Subtasks

- [x] Task 1: Write Alembic migration `002_pipeline_tables.py` (AC: 1, 2, 3)
  - [x] 1.1 Create `services/data-pipeline/alembic/versions/002_pipeline_tables.py` with `down_revision = "001"`
  - [x] 1.2 `upgrade()`: create `pipeline.opportunities` table with all columns, UNIQUE constraint on `(source_id, source_type)`, and indexes on `deadline`, `status`, `deleted_at`
  - [x] 1.3 `upgrade()`: create `pipeline.submission_guides` table with FK to `opportunities.id` and index on `opportunity_id`
  - [x] 1.4 `upgrade()`: create `pipeline.crawler_runs` table with indexes on `crawler_type`, `status`, `started_at`
  - [x] 1.5 `upgrade()`: create `pipeline.enrichment_queue` table with index on `(status, created_at)` for FIFO processing
  - [x] 1.6 `downgrade()`: drop all four tables in reverse FK-safe order (enrichment_queue, submission_guides first, then opportunities, crawler_runs last)

- [x] Task 2: Create SQLAlchemy ORM models (AC: 4, 5)
  - [x] 2.1 Create `services/data-pipeline/src/data_pipeline/models/__init__.py`
  - [x] 2.2 Create `services/data-pipeline/src/data_pipeline/models/base.py` — declare `Base = DeclarativeBase()` and `SCHEMA = "pipeline"`
  - [x] 2.3 Create `services/data-pipeline/src/data_pipeline/models/opportunity.py` — `Opportunity` model with all columns; use `ARRAY(Text)` for `cpv_codes` and `JSONB` for `evaluation_criteria`, `mandatory_documents`, `relevance_scores`, `raw_data`
  - [x] 2.4 Add soft-delete default criterion to `Opportunity` via `with_loader_criteria` using `@event.listens_for(Session, "do_orm_execute")` pattern
  - [x] 2.5 Create `services/data-pipeline/src/data_pipeline/models/submission_guide.py` — `SubmissionGuide` model with `steps` JSONB and FK to `Opportunity`
  - [x] 2.6 Create `services/data-pipeline/src/data_pipeline/models/crawler_run.py` — `CrawlerRun` model with status enum fields and count columns
  - [x] 2.7 Create `services/data-pipeline/src/data_pipeline/models/enrichment_queue.py` — `EnrichmentQueueItem` model with `enrichment_type`, `status`, `attempts` columns
  - [x] 2.8 Export all four models from `models/__init__.py`

- [x] Task 3: Wire `env.py` to import `target_metadata` (AC: 1)
  - [x] 3.1 Update `services/data-pipeline/alembic/env.py`: import `Base` from `data_pipeline.models.base` and set `target_metadata = Base.metadata` so Alembic can reflect the schema on `alembic check`

- [x] Task 4: Update `pyproject.toml` with test dependencies (AC: 5)
  - [x] 4.1 Add `testcontainers[postgres]>=4.8` and `pytest-timeout>=2.3` to `[project.optional-dependencies] dev`
  - [x] 4.2 Add `asyncpg>=0.29` to `[project.optional-dependencies] dev` (needed for testcontainer async DB sessions in tests)
  - [x] 4.3 Add `[tool.pytest.ini_options]` to `pyproject.toml` with `asyncio_mode = "auto"` and `timeout = 60`

- [x] Task 5: Write unit / integration tests (AC: 3, 5, 6)
  - [x] 5.1 Create `services/data-pipeline/tests/conftest.py` with session-scoped `pg_container` fixture (PostgreSqlContainer, runs `alembic upgrade head` via subprocess) and function-scoped async `db_session` fixture (AsyncSession against the testcontainer)
  - [x] 5.2 Create `services/data-pipeline/tests/integration/test_pipeline_migration.py`:
    - Test E05-P1-001: `alembic upgrade head` creates all 4 tables in the `pipeline` schema
    - Test E05-P2-001: `alembic downgrade -1` drops all 4 tables, no artifacts remain
    - Test E05-P2-002: `deadline`, `status`, `deleted_at` indexes present on `pipeline.opportunities`; `crawler_type` index on `pipeline.crawler_runs`; `(status, created_at)` index on `pipeline.enrichment_queue`
  - [x] 5.3 Create `services/data-pipeline/tests/unit/test_opportunity_model.py`:
    - Test E05-P1-002: direct duplicate insert raises `IntegrityError` on `(source_id, source_type)` unique constraint
    - Test E05-P1-003: CRUD round-trip for `Opportunity` — create with `cpv_codes`, `evaluation_criteria`, `relevance_scores`; read back, verify JSONB and ARRAY types preserved; update `status`; verify `updated_at` changes
    - Test E05-P1-004: insert 1 soft-deleted row (`deleted_at` set) + 1 active row; `select(Opportunity)` returns exactly 1 row (soft-deleted excluded automatically)
    - Test E05-P2-012: insert 2 soft-deleted + 1 active; all model-level `select(Opportunity)` queries return 1
  - [x] 5.4 Create `services/data-pipeline/tests/unit/test_submission_guide_model.py`:
    - CRUD round-trip for `SubmissionGuide` — create with structured `steps` JSONB; read back, verify FK resolves; verify `reviewed` defaults to `False`
  - [x] 5.5 Create `services/data-pipeline/tests/unit/test_crawler_run_model.py`:
    - CRUD round-trip — create with `status='running'`; update to `status='completed'`, set `ended_at`, update `found`/`new`/`updated` counts; read back and assert values
  - [x] 5.6 Create `services/data-pipeline/tests/unit/test_enrichment_queue_model.py`:
    - CRUD round-trip — create item with `enrichment_type='relevance_scoring'`, `status='pending'`; increment `attempts`; update `status='failed'`; delete item; verify clean

## Dev Notes

### Architecture: How This Story Fits

S05.01 is the **data layer foundation** for Epic 5. It produces no runnable pipeline logic — only the migration and the ORM model layer. Every subsequent story depends on it:

- S05.02 imports `CrawlerRun` to record Beat-triggered crawl cycles
- S05.04–S05.06 import `Opportunity` for the atomic upsert
- S05.07 reads/writes `Opportunity.relevance_scores`
- S05.08 creates `SubmissionGuide` rows
- S05.09 reads `CrawlerRun.id` for the Redis event payload
- S05.10 sets `Opportunity.deleted_at` and cascades to `SubmissionGuide`
- S05.11 reads/writes `EnrichmentQueueItem`

The migration chain must stay intact: `001 → 002`. Do not squash or renumber.

### CRITICAL: Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/data-pipeline/alembic/env.py` | Pipeline schema Alembic env, `target_metadata = None` | **MODIFY** — set `target_metadata = Base.metadata` |
| `services/data-pipeline/alembic/versions/001_initial.py` | Creates `pipeline._migrations_meta` table | **DO NOT TOUCH** |
| `services/data-pipeline/alembic.ini` | Points to `alembic/` directory, psycopg2 URL pattern | **DO NOT TOUCH** |
| `services/data-pipeline/pyproject.toml` | Has `alembic>=1.13`, `psycopg2-binary>=2.9`, `sqlalchemy[asyncio]>=2.0`, `celery[redis]>=5.3` | **APPEND ONLY** — add dev test deps |
| `services/data-pipeline/src/data_pipeline/main.py` | Minimal FastAPI stub | **DO NOT TOUCH** |
| `infra/postgres/init/01-init-schemas-and-roles.sql` | Creates `pipeline` schema + `pipeline_role` with DDL rights for `migration_role` | **DO NOT TOUCH** |
| `docker-compose.yml` | PostgreSQL 16 container | **DO NOT TOUCH** |

### Files to CREATE (New)

| File | Purpose |
|------|---------|
| `services/data-pipeline/alembic/versions/002_pipeline_tables.py` | Alembic migration — creates 4 pipeline tables |
| `services/data-pipeline/src/data_pipeline/models/__init__.py` | Package init — exports all 4 models + `Base` |
| `services/data-pipeline/src/data_pipeline/models/base.py` | `DeclarativeBase`, `SCHEMA = "pipeline"` |
| `services/data-pipeline/src/data_pipeline/models/opportunity.py` | `Opportunity` ORM model with soft-delete criterion |
| `services/data-pipeline/src/data_pipeline/models/submission_guide.py` | `SubmissionGuide` ORM model |
| `services/data-pipeline/src/data_pipeline/models/crawler_run.py` | `CrawlerRun` ORM model |
| `services/data-pipeline/src/data_pipeline/models/enrichment_queue.py` | `EnrichmentQueueItem` ORM model |
| `services/data-pipeline/tests/conftest.py` | Session-scoped testcontainer + async DB session fixtures |
| `services/data-pipeline/tests/integration/test_pipeline_migration.py` | Migration up/down + index assertions |
| `services/data-pipeline/tests/unit/test_opportunity_model.py` | Opportunity CRUD + soft-delete + unique constraint tests |
| `services/data-pipeline/tests/unit/test_submission_guide_model.py` | SubmissionGuide CRUD tests |
| `services/data-pipeline/tests/unit/test_crawler_run_model.py` | CrawlerRun CRUD tests |
| `services/data-pipeline/tests/unit/test_enrichment_queue_model.py` | EnrichmentQueueItem CRUD tests |

### Files to MODIFY (Existing)

| File | Change |
|------|--------|
| `services/data-pipeline/alembic/env.py` | Import `Base` from `data_pipeline.models.base`; set `target_metadata = Base.metadata` |
| `services/data-pipeline/pyproject.toml` | Add `testcontainers[postgres]>=4.8`, `asyncpg>=0.29`, `pytest-timeout>=2.3` to dev deps; add `[tool.pytest.ini_options]` |

### Full Data Model Definitions

#### `pipeline.opportunities` — Central Opportunity Store

```python
class Opportunity(Base):
    __tablename__ = "opportunities"
    __table_args__ = (
        UniqueConstraint("source_id", "source_type", name="uq_opportunity_source"),
        Index("ix_opportunity_deadline", "deadline"),
        Index("ix_opportunity_status", "status"),
        Index("ix_opportunity_deleted_at", "deleted_at"),
        {"schema": SCHEMA},
    )

    id: Mapped[UUID] = mapped_column(pg.UUID(as_uuid=True), primary_key=True, default=uuid4)
    source_id: Mapped[str] = mapped_column(Text, nullable=False)
    source_type: Mapped[str] = mapped_column(String(20), nullable=False)  # aop / ted / eu_grants
    title: Mapped[str] = mapped_column(Text, nullable=False)
    description: Mapped[str | None] = mapped_column(Text)
    opportunity_type: Mapped[str | None] = mapped_column(String(20))  # tender / grant
    status: Mapped[str] = mapped_column(String(20), nullable=False, server_default="open")
    deadline: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    budget_min: Mapped[Decimal | None] = mapped_column(Numeric(18, 2))
    budget_max: Mapped[Decimal | None] = mapped_column(Numeric(18, 2))
    currency: Mapped[str | None] = mapped_column(String(3))
    country: Mapped[str | None] = mapped_column(Text)
    region: Mapped[str | None] = mapped_column(Text)
    contracting_authority: Mapped[str | None] = mapped_column(Text)
    cpv_codes: Mapped[list[str]] = mapped_column(pg.ARRAY(Text), nullable=False, server_default="{}")
    evaluation_criteria: Mapped[dict | None] = mapped_column(pg.JSONB)
    mandatory_documents: Mapped[dict | None] = mapped_column(pg.JSONB)
    relevance_scores: Mapped[dict | None] = mapped_column(pg.JSONB)       # {company_id: score}
    raw_data: Mapped[dict | None] = mapped_column(pg.JSONB)               # raw agent response
    published_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    # Relationships
    submission_guides: Mapped[list["SubmissionGuide"]] = relationship(back_populates="opportunity")
    enrichment_items: Mapped[list["EnrichmentQueueItem"]] = relationship(back_populates="opportunity")
```

#### `pipeline.submission_guides` — Portal-Specific Submission Instructions

```python
class SubmissionGuide(Base):
    __tablename__ = "submission_guides"
    __table_args__ = (
        Index("ix_submission_guide_opportunity_id", "opportunity_id"),
        {"schema": SCHEMA},
    )

    id: Mapped[UUID] = mapped_column(pg.UUID(as_uuid=True), primary_key=True, default=uuid4)
    opportunity_id: Mapped[UUID] = mapped_column(
        pg.UUID(as_uuid=True), ForeignKey(f"{SCHEMA}.opportunities.id", ondelete="CASCADE"), nullable=False
    )
    source_portal: Mapped[str] = mapped_column(String(20), nullable=False)  # aop / ted / eu_grants
    steps: Mapped[dict] = mapped_column(pg.JSONB, nullable=False)
    # steps structure: ordered list of {step_number, title, instruction, portal_url, form_reference, tips}
    generated_by: Mapped[str | None] = mapped_column(Text)  # agent_execution_id / version string
    reviewed: Mapped[bool] = mapped_column(Boolean, nullable=False, server_default="false")
    deleted_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    opportunity: Mapped["Opportunity"] = relationship(back_populates="submission_guides")
```

#### `pipeline.crawler_runs` — Crawl Audit Log

```python
class CrawlerRun(Base):
    __tablename__ = "crawler_runs"
    __table_args__ = (
        Index("ix_crawler_run_crawler_type", "crawler_type"),
        Index("ix_crawler_run_status", "status"),
        Index("ix_crawler_run_started_at", "started_at"),
        {"schema": SCHEMA},
    )

    id: Mapped[UUID] = mapped_column(pg.UUID(as_uuid=True), primary_key=True, default=uuid4)
    crawler_type: Mapped[str] = mapped_column(String(20), nullable=False)  # aop / ted / eu_grants
    status: Mapped[str] = mapped_column(String(20), nullable=False, server_default="running")
    # status values: running / retrying / completed / failed
    started_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    ended_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    found: Mapped[int] = mapped_column(Integer, nullable=False, server_default="0")
    new: Mapped[int] = mapped_column(Integer, nullable=False, server_default="0")
    updated: Mapped[int] = mapped_column(Integer, nullable=False, server_default="0")
    errors: Mapped[dict] = mapped_column(pg.JSONB, nullable=False, server_default="{}")
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

> **Note**: `new` is a reserved word in many languages but valid as a PostgreSQL column name. Use `mapped_column(name="new")` if there are quoting issues — or rename to `new_count` / `inserted_count` if linters complain. The migration must quote it: `sa.Column("new", sa.Integer, ...)`.

#### `pipeline.enrichment_queue` — Failed Enrichment Retry Queue

```python
class EnrichmentQueueItem(Base):
    __tablename__ = "enrichment_queue"
    __table_args__ = (
        Index("ix_enrichment_queue_status_created", "status", "created_at"),
        {"schema": SCHEMA},
    )

    id: Mapped[UUID] = mapped_column(pg.UUID(as_uuid=True), primary_key=True, default=uuid4)
    opportunity_id: Mapped[UUID] = mapped_column(
        pg.UUID(as_uuid=True), ForeignKey(f"{SCHEMA}.opportunities.id", ondelete="CASCADE"), nullable=False
    )
    enrichment_type: Mapped[str] = mapped_column(String(30), nullable=False)
    # enrichment_type values: relevance_scoring / submission_guide
    status: Mapped[str] = mapped_column(String(20), nullable=False, server_default="pending")
    # status values: pending / processing / done / failed
    attempts: Mapped[int] = mapped_column(Integer, nullable=False, server_default="0")
    last_error: Mapped[str | None] = mapped_column(Text)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    opportunity: Mapped["Opportunity"] = relationship(back_populates="enrichment_items")
```

### Alembic Migration Pattern (`002_pipeline_tables.py`)

Build on the scaffold from `001_initial.py`. The migration uses `schema=SCHEMA` on every `op.create_table()` call to ensure tables land in the `pipeline` schema:

```python
"""Pipeline tables — opportunities, submission_guides, crawler_runs, enrichment_queue.

Revision ID: 002
Revises: 001
Create Date: 2026-04-14
"""
from typing import Sequence, Union
from uuid import uuid4

import sqlalchemy as sa
from alembic import op
from sqlalchemy.dialects import postgresql as pg

revision: str = "002"
down_revision: Union[str, None] = "001"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

SCHEMA = "pipeline"


def upgrade() -> None:
    op.create_table(
        "opportunities",
        sa.Column("id", pg.UUID(as_uuid=True), primary_key=True),
        sa.Column("source_id", sa.Text, nullable=False),
        sa.Column("source_type", sa.String(20), nullable=False),
        sa.Column("title", sa.Text, nullable=False),
        sa.Column("description", sa.Text),
        sa.Column("opportunity_type", sa.String(20)),
        sa.Column("status", sa.String(20), nullable=False, server_default="open"),
        sa.Column("deadline", sa.DateTime(timezone=True)),
        sa.Column("budget_min", sa.Numeric(18, 2)),
        sa.Column("budget_max", sa.Numeric(18, 2)),
        sa.Column("currency", sa.String(3)),
        sa.Column("country", sa.Text),
        sa.Column("region", sa.Text),
        sa.Column("contracting_authority", sa.Text),
        sa.Column("cpv_codes", pg.ARRAY(sa.Text), nullable=False, server_default="{}"),
        sa.Column("evaluation_criteria", pg.JSONB),
        sa.Column("mandatory_documents", pg.JSONB),
        sa.Column("relevance_scores", pg.JSONB),
        sa.Column("raw_data", pg.JSONB),
        sa.Column("published_at", sa.DateTime(timezone=True)),
        sa.Column("deleted_at", sa.DateTime(timezone=True)),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.UniqueConstraint("source_id", "source_type", name="uq_opportunity_source"),
        schema=SCHEMA,
    )
    op.create_index("ix_opportunity_deadline", "opportunities", ["deadline"], schema=SCHEMA)
    op.create_index("ix_opportunity_status", "opportunities", ["status"], schema=SCHEMA)
    op.create_index("ix_opportunity_deleted_at", "opportunities", ["deleted_at"], schema=SCHEMA)

    op.create_table(
        "submission_guides",
        sa.Column("id", pg.UUID(as_uuid=True), primary_key=True),
        sa.Column("opportunity_id", pg.UUID(as_uuid=True),
                  sa.ForeignKey(f"{SCHEMA}.opportunities.id", ondelete="CASCADE"), nullable=False),
        sa.Column("source_portal", sa.String(20), nullable=False),
        sa.Column("steps", pg.JSONB, nullable=False),
        sa.Column("generated_by", sa.Text),
        sa.Column("reviewed", sa.Boolean, nullable=False, server_default="false"),
        sa.Column("deleted_at", sa.DateTime(timezone=True)),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        schema=SCHEMA,
    )
    op.create_index("ix_submission_guide_opportunity_id", "submission_guides", ["opportunity_id"], schema=SCHEMA)

    op.create_table(
        "crawler_runs",
        sa.Column("id", pg.UUID(as_uuid=True), primary_key=True),
        sa.Column("crawler_type", sa.String(20), nullable=False),
        sa.Column("status", sa.String(20), nullable=False, server_default="running"),
        sa.Column("started_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("ended_at", sa.DateTime(timezone=True)),
        sa.Column("found", sa.Integer, nullable=False, server_default="0"),
        sa.Column("new", sa.Integer, nullable=False, server_default="0"),
        sa.Column("updated", sa.Integer, nullable=False, server_default="0"),
        sa.Column("errors", pg.JSONB, nullable=False, server_default="{}"),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        schema=SCHEMA,
    )
    op.create_index("ix_crawler_run_crawler_type", "crawler_runs", ["crawler_type"], schema=SCHEMA)
    op.create_index("ix_crawler_run_status", "crawler_runs", ["status"], schema=SCHEMA)
    op.create_index("ix_crawler_run_started_at", "crawler_runs", ["started_at"], schema=SCHEMA)

    op.create_table(
        "enrichment_queue",
        sa.Column("id", pg.UUID(as_uuid=True), primary_key=True),
        sa.Column("opportunity_id", pg.UUID(as_uuid=True),
                  sa.ForeignKey(f"{SCHEMA}.opportunities.id", ondelete="CASCADE"), nullable=False),
        sa.Column("enrichment_type", sa.String(30), nullable=False),
        sa.Column("status", sa.String(20), nullable=False, server_default="pending"),
        sa.Column("attempts", sa.Integer, nullable=False, server_default="0"),
        sa.Column("last_error", sa.Text),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        sa.Column("updated_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        schema=SCHEMA,
    )
    op.create_index("ix_enrichment_queue_status_created", "enrichment_queue",
                    ["status", "created_at"], schema=SCHEMA)


def downgrade() -> None:
    # Drop in reverse FK-safe order
    op.drop_table("enrichment_queue", schema=SCHEMA)
    op.drop_table("submission_guides", schema=SCHEMA)
    op.drop_table("crawler_runs", schema=SCHEMA)
    op.drop_table("opportunities", schema=SCHEMA)
```

### Soft-Delete Default Filter Pattern

SQLAlchemy 2.0 supports `with_loader_criteria` / `__mapper_args__` for model-level filtering. Use the `with_loader_criteria` event approach registered on the mapper, which applies to all ORM-level `select()` queries automatically:

```python
from sqlalchemy import event
from sqlalchemy.orm import Session

# In opportunity.py, after the Opportunity class definition:
from sqlalchemy.orm import with_loader_criteria

# Preferred: use __mapper_args__ with a polymorphic identity filter
# For SQLAlchemy 2.x, the cleanest approach is the `enable_if` loader criteria:

@event.listens_for(Session, "do_orm_execute")
def _add_soft_delete_filter(execute_state):
    if (
        execute_state.is_select
        and not execute_state.execution_options.get("include_deleted", False)
    ):
        execute_state.statement = execute_state.statement.options(
            with_loader_criteria(
                Opportunity,
                Opportunity.deleted_at.is_(None),
                include_aliases=True,
            )
        )
```

**IMPORTANT** — Register this event handler at application startup (in `main.py` or via the model module import). The event listener must be attached to the `Session` class, not an instance.

**Alternatively (simpler for this story)**, use `__mapper_args__` with `polymorphic_on` + a class-level query default. The most pragmatic SQLAlchemy 2.x approach that works consistently with async sessions is the session event approach above. Verify it works with `AsyncSession` from `sqlalchemy.ext.asyncio` — it does, because `do_orm_execute` fires for both sync and async.

**Caller opt-out** (for cleanup task): `session.execute(select(Opportunity).execution_options(include_deleted=True))`.

### Updating `env.py` for `target_metadata`

```python
# In services/data-pipeline/alembic/env.py — REPLACE the target_metadata line:

# OLD:
# target_metadata = None

# NEW:
import sys
import os
# Ensure src/ is on the path for the Alembic process
sys.path.insert(0, os.path.join(os.path.dirname(__file__), "..", "src"))
from data_pipeline.models.base import Base  # noqa: E402
target_metadata = Base.metadata
```

This enables `alembic check` (drift detection) and autogenerate — important for later stories.

### Test Fixtures Pattern (`conftest.py`)

```python
import subprocess, sys, os
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker

PIPELINE_SCHEMA = "pipeline"

@pytest.fixture(scope="session")
def pg_container():
    """Start a PostgreSQL 16 testcontainer and run Alembic migrations."""
    with PostgresContainer("postgres:16-alpine") as pg:
        # Create pipeline schema (testcontainers DB starts with only public schema)
        engine_sync = sqlalchemy.create_engine(pg.get_connection_url())
        with engine_sync.connect() as conn:
            conn.execute(text("CREATE SCHEMA IF NOT EXISTS pipeline"))
            conn.execute(text("CREATE SCHEMA IF NOT EXISTS shared"))
            conn.commit()
        engine_sync.dispose()

        # Run Alembic migrations
        env = {**os.environ, "DATABASE_URL": pg.get_connection_url()}
        result = subprocess.run(
            [sys.executable, "-m", "alembic", "upgrade", "head"],
            cwd=os.path.join(os.path.dirname(__file__), ".."),
            env=env,
            capture_output=True,
            text=True,
        )
        assert result.returncode == 0, f"Alembic upgrade failed:\n{result.stderr}"
        yield pg

@pytest.fixture
async def db_session(pg_container):
    """Async SQLAlchemy session against the testcontainer DB."""
    url = pg_container.get_connection_url().replace("postgresql://", "postgresql+asyncpg://")
    engine = create_async_engine(url)
    async_session = async_sessionmaker(engine, expire_on_commit=False)
    async with async_session() as session:
        yield session
        await session.rollback()
    await engine.dispose()
```

**Note on schema creation**: The PostgreSQL testcontainer starts with only the `public` schema. The real Docker Compose DB creates `pipeline` and `shared` via `01-init-schemas-and-roles.sql`. The `conftest.py` must create these schemas before running Alembic, since the `env.py` does `SET search_path TO pipeline, shared`.

### Column Naming Gotcha: `new`

`new` is a reserved word in Python but **valid** as a PostgreSQL column name. SQLAlchemy will quote it correctly in SQL. However, Python attribute name `new` may conflict with class internals. Options:
1. **Use `new` as the column name** — set the mapped attribute to `new_count` with `mapped_column("new", ...)` to disambiguate
2. **Rename to `new_count`** — simpler, no gotchas

Recommended: use attribute name `new_count` with `mapped_column("new", ...)` so the DB column stays `new` (matching the epic spec) but the Python attribute is unambiguous.

### Test Expectations (from Epic-Level Test Design)

This story satisfies the following test scenarios from `test-design-epic-05.md`:

| Priority | Test ID | Description | Risk Link |
|----------|---------|-------------|-----------|
| **P1** | E05-P1-001 | `alembic upgrade head` creates all 4 tables | — |
| **P1** | E05-P1-002 | Direct DB insert of duplicate `(source_id, source_type)` raises `IntegrityError` | E05-R-001 |
| **P1** | E05-P1-003 | CRUD round-trip for `Opportunity` including `cpv_codes`, `evaluation_criteria`, `relevance_scores` JSONB | — |
| **P1** | E05-P1-004 | Soft-delete filter: 1 deleted + 1 active → query returns exactly 1 | E05-R-006 |
| **P2** | E05-P2-001 | `alembic downgrade -1` drops all 4 tables cleanly | — |
| **P2** | E05-P2-002 | `deadline`, `status`, `crawler_type` indexes exist (verified via `sqlalchemy.inspect`) | — |
| **P2** | E05-P2-012 | After soft-delete via cleanup task, model queries still exclude deleted rows | E05-R-006 |

**Key risk mitigations tested here**:
- **E05-R-001** (dedup race): The `UNIQUE(source_id, source_type)` DB-level constraint is the primary safeguard. E05-P1-002 verifies it is enforced at the DB layer — not just application logic. Later stories (S05.04) build the `INSERT ... ON CONFLICT DO UPDATE SET ...` atomic upsert on top.
- **E05-R-006** (soft-delete bypass): The `deleted_at IS NULL` default criterion must be a model-level enforcement, not a caller-side responsibility. E05-P1-004 and E05-P2-012 prove it cannot be bypassed by accident.

### Cross-Story Dependencies

| Story | Status | Dependency |
|-------|--------|------------|
| S01.04 — Alembic Migration Scaffold | DONE | `services/data-pipeline/alembic/` directory, `001_initial.py`, `alembic.ini`, Alembic + psycopg2-binary deps |
| S01.03 — PostgreSQL Schema Design | DONE | `pipeline` schema exists, `migration_role` has DDL rights, `pipeline_role` has DML rights |
| S01.07 — eusolicit-models / eusolicit-kraftdata | DONE | `eusolicit_models.enums.OpportunityStatus` available for reference (but ORM models define their own column constraints) |
| S04.10 — AI Gateway Integration Tests | DONE | Testcontainer pattern established in `services/ai-gateway/tests/integration/conftest.py` — reuse the same fixture structure |
| S05.02 — Celery Bootstrap | NEXT | Needs `CrawlerRun` model |
| S05.04–S05.06 — Crawler Tasks | NEXT | Need `Opportunity`, `CrawlerRun` models |

### Project Structure After This Story

```
services/data-pipeline/
├── alembic/
│   ├── env.py                                 # MODIFY — target_metadata = Base.metadata
│   └── versions/
│       ├── 001_initial.py                     # UNTOUCHED
│       └── 002_pipeline_tables.py             # + NEW
├── src/data_pipeline/
│   ├── main.py                                # UNTOUCHED
│   └── models/                                # + NEW package
│       ├── __init__.py                        # + NEW
│       ├── base.py                            # + NEW — Base, SCHEMA
│       ├── opportunity.py                     # + NEW — Opportunity + soft-delete filter
│       ├── submission_guide.py                # + NEW
│       ├── crawler_run.py                     # + NEW
│       └── enrichment_queue.py               # + NEW
├── tests/
│   ├── conftest.py                            # + NEW — testcontainer fixtures
│   ├── integration/
│   │   ├── __init__.py                        # exists
│   │   └── test_pipeline_migration.py         # + NEW
│   └── unit/
│       ├── __init__.py                        # exists
│       ├── test_opportunity_model.py          # + NEW
│       ├── test_submission_guide_model.py     # + NEW
│       ├── test_crawler_run_model.py          # + NEW
│       └── test_enrichment_queue_model.py    # + NEW
└── pyproject.toml                             # MODIFY — add test deps
```

### References

- `eusolicit-docs/planning-artifacts/epics/E05-data-pipeline-ingestion.md#S05.01`
- `eusolicit-docs/test-artifacts/test-design-epic-05.md#P1-E05-P1-001–004, P2-E05-P2-001–002,012`
- `eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#9.1-New-Entity-Definitions-submission_guides`
- `eusolicit-docs/implementation-artifacts/1-4-alembic-migration-scaffold.md` — env.py pattern, migration structure
- `eusolicit-docs/implementation-artifacts/4-10-integration-tests-and-end-to-end-validation.md` — testcontainer fixture pattern
- `eusolicit-app/services/data-pipeline/alembic/env.py` — existing pipeline Alembic env
- `eusolicit-app/services/data-pipeline/alembic/versions/001_initial.py` — existing initial migration
- `eusolicit-app/packages/eusolicit-models/src/eusolicit_models/enums.py` — `OpportunityStatus` enum values

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6 (claude-code)

### Debug Log References

- Fixed psycopg2 URL scheme issue in conftest.py: `PostgresContainer(driver="asyncpg")` returns `postgresql+asyncpg://` URL; plain psycopg2 connection needed `postgresql://` scheme. Fixed by replacing `+asyncpg` prefix before direct connections.
- Fixed SQLAlchemy cascade delete behavior: without `passive_deletes=True` on Opportunity relationships, SQLAlchemy attempted to NULL out FK columns before DB CASCADE fired, violating NOT NULL constraint. Added `passive_deletes=True` to `submission_guides` and `enrichment_items` relationships.
- Fixed mypy strict `dict` generic type errors: replaced bare `dict` and `dict | None` annotations with `dict[str, Any]` and `dict[str, Any] | None` in JSONB-mapped columns across opportunity, submission_guide, and crawler_run models.

### Completion Notes List

- **Migration (002_pipeline_tables.py)**: Creates all 4 pipeline tables with correct column types, UNIQUE constraint on `(source_id, source_type)`, FK with `ondelete="CASCADE"` for child tables, and all required indexes. Downgrade drops in FK-safe order. Migration chain intact: `001 → 002`.
- **ORM Models**: Four models created (`Opportunity`, `SubmissionGuide`, `CrawlerRun`, `EnrichmentQueueItem`) with full type annotations including `dict[str, Any]` for JSONB columns. `CrawlerRun.new_count` maps Python attribute to DB column `new` to avoid keyword clash.
- **Soft-delete filter**: Implemented via `@event.listens_for(Session, "do_orm_execute")` with `with_loader_criteria`. Works with both sync and async sessions. Callers opt-out with `.execution_options(include_deleted=True)`.
- **env.py**: Updated to insert `src/` on sys.path and import `Base.metadata` for Alembic drift detection / autogenerate support.
- **Tests**: 12 tests total (3 integration migration + 9 unit model). All 12 pass against a real PostgreSQL 16 testcontainer. mypy --strict reports 0 errors on all 6 model files.
- **AC coverage**: AC1 ✅ (upgrade head creates tables), AC2 ✅ (downgrade drops cleanly), AC3 ✅ (IntegrityError on duplicate source_id/source_type), AC4 ✅ (mypy strict passes), AC5 ✅ (CRUD round-trips all 4 models), AC6 ✅ (soft-delete auto-filter verified E05-P1-004 and E05-P2-012).

### File List

- `eusolicit-app/services/data-pipeline/alembic/versions/002_pipeline_tables.py` (NEW)
- `eusolicit-app/services/data-pipeline/alembic/env.py` (MODIFIED — added sys.path, Base import, target_metadata)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/models/__init__.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/models/base.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/models/opportunity.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/models/submission_guide.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/models/crawler_run.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/models/enrichment_queue.py` (NEW)
- `eusolicit-app/services/data-pipeline/tests/conftest.py` (MODIFIED — replaced legacy fixtures with testcontainer fixtures)
- `eusolicit-app/services/data-pipeline/tests/integration/test_pipeline_migration.py` (NEW)
- `eusolicit-app/services/data-pipeline/tests/unit/test_opportunity_model.py` (NEW)
- `eusolicit-app/services/data-pipeline/tests/unit/test_submission_guide_model.py` (NEW)
- `eusolicit-app/services/data-pipeline/tests/unit/test_crawler_run_model.py` (NEW)
- `eusolicit-app/services/data-pipeline/tests/unit/test_enrichment_queue_model.py` (NEW)
- `eusolicit-app/services/data-pipeline/pyproject.toml` (MODIFIED — added testcontainers, asyncpg, pytest-timeout deps + pytest ini_options)

## Senior Developer Review

### Review Findings

**Review date:** 2026-04-14
**Review layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor (3-layer adversarial review)
**Raw findings:** 39 → 22 after dedup → 2 patch, 5 defer, 0 decision-needed, 17 dismissed

- [ ] [Review][Patch] conftest.py `psycopg2.connect()` failure masks real error — if `connect()` raises, `conn` is unbound and `finally: conn.close()` raises `NameError` masking the original `OperationalError`. Fix: initialize `conn = None` before try block and guard close with `if conn is not None`. [`tests/conftest.py:49-56`]
- [ ] [Review][Patch] test_downgrade re-upgrade not in try/finally — if assertion fails after downgrade but before re-upgrade, all subsequent session-scoped tests run against a schema missing its 4 tables. Fix: wrap the re-upgrade in a `finally` block or add `pytest-order` dependency with `@pytest.mark.order("last")`. [`tests/integration/test_pipeline_migration.py:142-168`]
- [x] [Review][Defer] Soft-delete event listener not idempotent on module reload — `@event.listens_for(Session, "do_orm_execute")` registers unconditionally; double-fires if module reloaded. Guard with `event.contains()` check. Benign today. [`models/opportunity.py:82`] — deferred, low-risk improvement
- [x] [Review][Defer] Lazy-load of soft-deleted parent returns None for non-Optional `Mapped[Opportunity]` — `SubmissionGuide.opportunity` and `EnrichmentQueueItem.opportunity` typed non-Optional but soft-delete filter can make lazy load return None. [`models/submission_guide.py:46`, `models/enrichment_queue.py:46`] — deferred, future story S05.10 concern
- [x] [Review][Defer] `cpv_codes` and `errors` columns lack Python-side `default=` — pre-flush attribute is None despite `Mapped[list[str]]`/`Mapped[dict]` type. Add `default=list`/`default=dict`. [`models/opportunity.py:47`, `models/crawler_run.py:40`] — deferred, low-risk
- [x] [Review][Defer] `db_session` fixture uses rollback-only isolation — any test calling `commit()` permanently dirties the shared testcontainer. Consider savepoint-based wrapper. [`tests/conftest.py:80-97`] — deferred, latent risk
- [x] [Review][Defer] `include_deleted` execution option is a magic string with no typed constant — easy to misspell silently. [`models/opportunity.py:87`] — deferred, improvement

### Review Verdict

**REVIEW: Approve**

All 6 acceptance criteria verified and met. Implementation matches spec precisely. 2 minor patch findings (test reliability improvements) do not block approval — they are non-functional improvements to test infrastructure robustness. 5 deferred findings are pre-existing patterns or future-story concerns.

## Change Log

- 2026-04-14: Story 5.1 created — pipeline schema migration and ORM model layer for E05 data pipeline
- 2026-04-14: Story 5.1 implemented — all 5 tasks complete, 12/12 tests pass, mypy strict clean, status → review
- 2026-04-14: Senior developer review — APPROVE (2 patch, 5 defer, 17 dismissed). All 6 ACs met.
