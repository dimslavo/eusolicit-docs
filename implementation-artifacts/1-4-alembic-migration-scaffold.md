# Story 1.4: Alembic Migration Scaffold

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **developer on the EU Solicit platform**,
I want **each of the 5 services to have its own Alembic migration directory with isolated `env.py` targeting its own schema, an initial no-op migration, and a `make migrate-all` command**,
so that **service teams can evolve their database schemas independently via version-controlled migrations, with Alembic's version tracking isolated per-schema to prevent cross-service collisions, and all migrations runnable in a single deterministic command**.

## Acceptance Criteria

1. Each service has `alembic.ini` and `alembic/` directory under its root
2. Each service's `env.py` is configured to use `search_path` set to its own schema + `shared`
3. `alembic upgrade head` runs successfully for each service against the Docker Compose PostgreSQL
4. Each service has one initial migration (e.g., `001_initial.py`) that is a no-op or creates a `_migrations_meta` table in its schema
5. Migration naming convention: `NNN_descriptive_name.py` (sequential, not timestamp-based)
6. A `make migrate-all` command runs all 5 services' migrations in dependency order

## Tasks / Subtasks

- [x] Task 1: Create `alembic/` directory structure for all 5 services (AC: 1, 2, 5)
  - [x] 1.1 Create `services/<svc>/alembic/env.py` for each service with schema-aware config
  - [x] 1.2 Create `services/<svc>/alembic/script.py.mako` with sequential naming template
  - [x] 1.3 Create `services/<svc>/alembic/versions/` directory (empty, with `.gitkeep`)
  - [x] 1.4 Update existing `alembic.ini` in each service to use `migration_role` connection string via env var
- [x] Task 2: Create initial migration for each service (AC: 3, 4, 5)
  - [x] 2.1 Generate `001_initial.py` for each service — create `_migrations_meta` table in the service's own schema
  - [x] 2.2 Verify `alembic upgrade head` succeeds for each service against Docker Compose PostgreSQL
  - [x] 2.3 Verify `alembic downgrade base` succeeds for each service (rollback works)
- [x] Task 3: Add `make migrate-all` Makefile target (AC: 6)
  - [x] 3.1 Add `migrate-all` target that runs migrations sequentially: shared-dependent services first
  - [x] 3.2 Add `migrate-service` target for individual service migration (usage: `make migrate-service SVC=client-api`)
  - [x] 3.3 Add `migrate-status` target showing current migration head for each service
- [x] Task 4: Add `alembic` dependency to all service pyproject.toml files (AC: 3)
  - [x] 4.1 Add `"alembic>=1.13"` to `dependencies` in each service's `pyproject.toml`
  - [x] 4.2 Add `"psycopg2-binary>=2.9"` to `dependencies` (sync PG driver for Alembic migrations)
- [x] Task 5: Integration tests (AC: 2, 3)
  - [x] 5.1 Test: `alembic upgrade head` succeeds for each service
  - [x] 5.2 Test: Each service's `alembic_version` table lives in its own schema (not `public`)
  - [x] 5.3 Test: Each service's `_migrations_meta` table exists in its own schema after migration
  - [x] 5.4 Test: `make migrate-all` runs all 5 migrations without error

## Dev Notes

### CRITICAL: Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/*/alembic.ini` | Placeholder config exists in ALL 5 services | **MODIFY IN-PLACE** — update `sqlalchemy.url` to read from env var |
| `docker-compose.yml` | PostgreSQL 16 + all service containers | **DO NOT TOUCH** |
| `.env.example` | All role passwords + connection strings defined | **DO NOT TOUCH** |
| `Makefile` | Docker + test targets | **APPEND ONLY** — add new migration targets |
| `infra/postgres/init/01-init-schemas-and-roles.sql` | Creates 6 schemas, 6 roles, grants, `shared.tenants` | **DO NOT TOUCH** |
| `tests/conftest.py` | DB fixtures using `migration_role` | **DO NOT TOUCH** |
| `packages/eusolicit-test-utils/.../db.py` | `ALL_SCHEMAS`, `SERVICE_SCHEMA_MAP`, `apply_migrations()` helper | **DO NOT TOUCH** — reuse these |

### Files to CREATE (New)

| File | Purpose |
|------|---------|
| `services/client-api/alembic/env.py` | Schema-aware Alembic environment for `client` schema |
| `services/client-api/alembic/script.py.mako` | Migration template with sequential naming |
| `services/client-api/alembic/versions/001_initial.py` | No-op initial migration |
| `services/admin-api/alembic/env.py` | Schema-aware Alembic environment for `admin` schema |
| `services/admin-api/alembic/script.py.mako` | Migration template with sequential naming |
| `services/admin-api/alembic/versions/001_initial.py` | No-op initial migration |
| `services/data-pipeline/alembic/env.py` | Schema-aware Alembic environment for `pipeline` schema |
| `services/data-pipeline/alembic/script.py.mako` | Migration template with sequential naming |
| `services/data-pipeline/alembic/versions/001_initial.py` | No-op initial migration |
| `services/ai-gateway/alembic/env.py` | Schema-aware Alembic environment for `gateway` schema |
| `services/ai-gateway/alembic/script.py.mako` | Migration template with sequential naming |
| `services/ai-gateway/alembic/versions/001_initial.py` | No-op initial migration |
| `services/notification/alembic/env.py` | Schema-aware Alembic environment for `notification` schema |
| `services/notification/alembic/script.py.mako` | Migration template with sequential naming |
| `services/notification/alembic/versions/001_initial.py` | No-op initial migration |
| `tests/integration/test_alembic_migrations.py` | Integration tests for migration scaffold |

### Files to MODIFY (Existing)

| File | Change |
|------|--------|
| `services/client-api/alembic.ini` | Update `sqlalchemy.url` to use `DATABASE_URL` env var |
| `services/admin-api/alembic.ini` | Same |
| `services/data-pipeline/alembic.ini` | Same |
| `services/ai-gateway/alembic.ini` | Same |
| `services/notification/alembic.ini` | Same |
| `services/client-api/pyproject.toml` | Add `alembic>=1.13`, `asyncpg>=0.29` to dependencies |
| `services/admin-api/pyproject.toml` | Same |
| `services/data-pipeline/pyproject.toml` | Same |
| `services/ai-gateway/pyproject.toml` | Same |
| `services/notification/pyproject.toml` | Same |
| `Makefile` | Add `migrate-all`, `migrate-service`, `migrate-status` targets |

### Service-to-Schema Mapping (MUST match exactly)

| Service Dir | Service Name | Own Schema | `version_table_schema` | `search_path` | DB Role |
|-------------|-------------|------------|------------------------|---------------|---------|
| `services/client-api` | `client-api` | `client` | `client` | `client, shared` | `migration_role` |
| `services/admin-api` | `admin-api` | `admin` | `admin` | `admin, shared` | `migration_role` |
| `services/data-pipeline` | `data-pipeline` | `pipeline` | `pipeline` | `pipeline, shared` | `migration_role` |
| `services/ai-gateway` | `ai-gateway` | `gateway` | `gateway` | `gateway, shared` | `migration_role` |
| `services/notification` | `notification` | `notification` | `notification` | `notification, shared` | `migration_role` |

This table is derived from `SERVICE_SCHEMA_MAP` in `packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py`. Match it exactly.

### alembic.ini Update Pattern

The existing `alembic.ini` files use `%(DB_USER)s` interpolation which is awkward. Replace the `sqlalchemy.url` line to use an environment variable override in `env.py` instead:

```ini
# alembic.ini — set a dummy URL; env.py overrides from DATABASE_URL env var
[alembic]
script_location = alembic
# URL set programmatically in env.py from DATABASE_URL environment variable
sqlalchemy.url = postgresql+asyncpg://migration_role:migration_password@localhost:5432/eusolicit
```

The key is that `env.py` reads `DATABASE_URL` from the environment and passes it to Alembic, overriding whatever is in `alembic.ini`. This is the standard pattern.

### env.py Pattern (CRITICAL — All 5 services follow this pattern)

Each service's `env.py` MUST configure:

1. **`version_table_schema`** — places `alembic_version` table inside the service's own schema (avoids collisions)
2. **`include_schemas`** — tells Alembic to operate on the service's schema + shared
3. **`search_path`** — PostgreSQL session `search_path` set to service schema + shared
4. **Connection from env var** — reads `DATABASE_URL` environment variable

```python
"""Alembic environment configuration for {service_name}."""
import os
from logging.config import fileConfig

from alembic import context
from sqlalchemy import create_engine, pool, text

# Alembic Config object
config = context.config

# Logging
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

# Schema configuration — MUST match SERVICE_SCHEMA_MAP
SERVICE_SCHEMA = "{own_schema}"   # e.g., "client"
SHARED_SCHEMA = "shared"
VERSION_TABLE_SCHEMA = SERVICE_SCHEMA

# No SQLAlchemy MetaData target_metadata for now (manual migrations)
target_metadata = None


def get_url() -> str:
    """Get database URL from environment, falling back to alembic.ini."""
    url = os.environ.get("DATABASE_URL")
    if url:
        # Alembic requires sync driver — replace asyncpg with psycopg2
        return url.replace("+asyncpg", "+psycopg2").replace("+aiopg", "+psycopg2")
    return config.get_main_option("sqlalchemy.url", "").replace("+asyncpg", "+psycopg2")


def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode (SQL script generation)."""
    context.configure(
        url=get_url(),
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        version_table_schema=VERSION_TABLE_SCHEMA,
        include_schemas=True,
    )
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    """Run migrations in 'online' mode (direct database connection)."""
    connectable = create_engine(get_url(), poolclass=pool.NullPool)

    with connectable.connect() as connection:
        # Set search_path so unqualified table references resolve to service schema
        connection.execute(text(f"SET search_path TO {SERVICE_SCHEMA}, {SHARED_SCHEMA}"))

        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            version_table_schema=VERSION_TABLE_SCHEMA,
            include_schemas=True,
        )

        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

**CRITICAL NOTES:**
- Alembic does NOT support async drivers. The `get_url()` function MUST replace `+asyncpg` with `+psycopg2` in the URL. This means `psycopg2-binary` (or `psycopg2`) must be available. Add `"psycopg2-binary>=2.9"` to each service's `pyproject.toml` dependencies.
- The `connection.execute(text(f"SET search_path TO ..."))` ensures unqualified table names in migrations resolve to the service's schema.
- `version_table_schema=SERVICE_SCHEMA` places the `alembic_version` table INSIDE the service's schema. This is how 5 services share one PostgreSQL database without colliding.
- `target_metadata = None` for now — there are no SQLAlchemy ORM models yet. When models are added in later stories, this will be set to `Base.metadata`.

### script.py.mako Template (Sequential Naming)

The epic requires `NNN_descriptive_name.py` format instead of Alembic's default hex revision IDs. Use this template:

```mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

# revision identifiers, used by Alembic.
revision: str = ${repr(up_revision)}
down_revision: Union[str, None] = ${repr(down_revision)}
branch_labels: Union[str, Sequence[str], None] = ${repr(branch_labels)}
depends_on: Union[str, Sequence[str], None] = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}
```

For sequential naming, configure `alembic.ini`:
```ini
[alembic]
# Use sequential revision IDs (not hex)
revision_environment = false
file_template = %%(rev)s_%%(slug)s
```

Then generate the initial migration with an explicit rev ID: `alembic revision --rev-id 001 -m "initial"`. This produces `001_initial.py`.

**Alternative (simpler)**: Manually create `001_initial.py` with `revision = "001"` and `down_revision = None`. This is cleaner for the scaffold since we control the content.

### Initial Migration Pattern (001_initial.py)

Each service's initial migration creates a `_migrations_meta` table in its schema. This proves connectivity + schema access.

```python
"""Initial migration — verify schema access.

Revision ID: 001
Revises: None
Create Date: 2026-04-06
"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa

revision: str = "001"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

# Schema set by env.py search_path, but be explicit for clarity
SCHEMA = "{own_schema}"  # e.g., "client"


def upgrade() -> None:
    op.create_table(
        "_migrations_meta",
        sa.Column("key", sa.String(100), primary_key=True),
        sa.Column("value", sa.Text, nullable=True),
        sa.Column("created_at", sa.DateTime(timezone=True), server_default=sa.func.now()),
        schema=SCHEMA,
    )
    # Seed with service identity
    op.execute(
        sa.text(
            f"INSERT INTO {SCHEMA}._migrations_meta (key, value) "
            f"VALUES ('service', '{SCHEMA}'), ('initialized_at', now()::text)"
        )
    )


def downgrade() -> None:
    op.drop_table("_migrations_meta", schema=SCHEMA)
```

### make migrate-all Dependency Order

The `shared` schema has no Alembic service (its tables are created by the init SQL or by services that reference it). Migration order for the 5 services doesn't have hard dependencies today, but use a logical order:

```makefile
MIGRATION_ORDER = data-pipeline client-api admin-api ai-gateway notification
MIGRATION_DB_URL ?= postgresql+psycopg2://migration_role:migration_password@localhost:5432/eusolicit

migrate-all: ## Run Alembic migrations for all services in dependency order
	@for svc in $(MIGRATION_ORDER); do \
		echo "🔄 Migrating $$svc..."; \
		cd services/$$svc && DATABASE_URL=$(MIGRATION_DB_URL) alembic upgrade head && cd ../..; \
		echo "✅ $$svc migrated"; \
	done

migrate-service: ## Run migrations for one service (usage: make migrate-service SVC=client-api)
	@if [ -z "$(SVC)" ]; then echo "Usage: make migrate-service SVC=client-api"; exit 1; fi
	cd services/$(SVC) && DATABASE_URL=$(MIGRATION_DB_URL) alembic upgrade head

migrate-status: ## Show migration status for all services
	@for svc in $(MIGRATION_ORDER); do \
		echo "📊 $$svc:"; \
		cd services/$$svc && DATABASE_URL=$(MIGRATION_DB_URL) alembic current 2>/dev/null || echo "  (no migrations run)"; \
		cd ../..; \
	done
```

**Note**: The Makefile uses `psycopg2` (sync driver) in `MIGRATION_DB_URL` because Alembic requires a synchronous connection. The `.env.example` URLs use `asyncpg` — do NOT use those directly for Alembic.

### psycopg2-binary Dependency

Alembic runs migrations synchronously. The existing `alembic.ini` uses `postgresql+asyncpg://` URLs, but Alembic's migration runner needs a sync driver. The `env.py` rewrites URLs to use `psycopg2`. Add this dependency:

```toml
dependencies = [
    # ... existing deps ...
    "alembic>=1.13",
    "psycopg2-binary>=2.9",  # Sync PG driver for Alembic migrations
]
```

Do NOT remove the existing `sqlalchemy[asyncio]` or `asyncpg` — those are used by the application code. `psycopg2-binary` is only for migration tooling.

### Testing Strategy

**Test file**: `tests/integration/test_alembic_migrations.py`

Tests require Docker Compose PostgreSQL running with init script applied (Story 1.3 prerequisite).

**Key tests:**
1. **Migration runs**: For each service, `alembic upgrade head` succeeds
2. **Version table isolation**: `alembic_version` table is in service's own schema, NOT `public`
3. **Migration meta table**: `_migrations_meta` table created in service schema
4. **Downgrade**: `alembic downgrade base` succeeds and cleans up
5. **Make target**: `make migrate-all` runs without error

**Connection for tests**: Use `TEST_DATABASE_URL` (migration_role against eusolicit_test) or construct per-service URLs. The existing `apply_migrations()` helper in `eusolicit-test-utils/db.py` runs `alembic upgrade head` in a service directory — reuse it.

**Important**: Tests should clean up after themselves. Run `alembic downgrade base` per service in teardown to leave the test database clean for other test suites.

### Test Expectations (from Epic-Level Test Design)

This story maps to the following test scenarios from `test-design-epic-01.md`:

| Priority | Test Description | Risk Link | Verification |
|----------|-----------------|-----------|--------------|
| **P1** | Alembic `upgrade head` for all 5 services | E01-R-004 (Score 4) | `make migrate-all` succeeds; `alembic_version` in each service's schema |

**Risk E01-R-004 (MEDIUM, Score 4):** Alembic migration ordering conflict — 5 services running migrations against shared PostgreSQL may conflict if run out of order. Mitigated by: `make migrate-all` enforcing dependency order, each service using own `alembic_version` table in its schema, CI running migrations sequentially.

**Quality gate**: P1 tests ≥95% pass rate. Migration ordering verified.

### Cross-Story Dependencies

- **Story 1.1** (DONE): Created monorepo structure, `services/*/` directories
- **Story 1.2** (DONE): Created `docker-compose.yml` with PostgreSQL, `.env.example` with all role passwords and DB URLs
- **Story 1.3** (DONE): Created `01-init-schemas-and-roles.sql` — schemas, roles, grants. Especially:
  - `migration_role` with DDL on all schemas + `GRANT CREATE ON DATABASE`
  - `ALTER DEFAULT PRIVILEGES FOR ROLE migration_role` ensuring service roles can access migration-created tables
  - All 6 schemas exist before Alembic runs
- **Story 1.5** (NEXT): Redis Streams — no dependency on this story
- **Story 1.6**: eusolicit-common health checks may query `_migrations_meta` — depends on Alembic scaffold existing

### Previous Story Intelligence (from Story 1.3)

Key learnings from Story 1.3 implementation:

1. **`migration_role` privileges**: Story 1.3 granted `migration_role` both `CREATE ON DATABASE` (for schema DDL) and full CRUD + DDL on all schemas. It also set `ALTER DEFAULT PRIVILEGES FOR ROLE migration_role` so tables created by `migration_role` auto-grant correct permissions to service roles. This means Alembic migrations running as `migration_role` will automatically have correct access for both DDL and future service role queries.

2. **Test database**: The init SQL script creates both `eusolicit` and `eusolicit_test` databases with identical schema/role setup. Alembic tests should run against `eusolicit_test`.

3. **`psql` meta-commands**: The init script uses `\c` to switch databases. Alembic env.py should NOT use psql — it uses SQLAlchemy engine connections.

4. **Idempotency pattern**: Story 1.3 used `IF NOT EXISTS` throughout. Alembic migrations should be idempotent where practical, but Alembic's version tracking handles this inherently (won't re-run already-applied migrations).

5. **653 tests in regression suite**: Run full test suite after implementation to verify no regressions.

### Project Structure Notes

**Target directory tree changes from this story** (new items marked with `+`):

```
eusolicit-app/
├── services/
│   ├── client-api/
│   │   ├── alembic.ini                          # MODIFY — update sqlalchemy.url
│   │   ├── alembic/                             # + NEW directory
│   │   │   ├── env.py                           # + NEW — schema: client
│   │   │   ├── script.py.mako                   # + NEW — migration template
│   │   │   └── versions/                        # + NEW directory
│   │   │       └── 001_initial.py               # + NEW — no-op initial migration
│   │   └── pyproject.toml                       # MODIFY — add alembic, psycopg2-binary
│   │
│   ├── admin-api/                               # Same structure as client-api
│   │   ├── alembic.ini                          # MODIFY
│   │   ├── alembic/                             # + NEW
│   │   │   ├── env.py                           # + NEW — schema: admin
│   │   │   ├── script.py.mako                   # + NEW
│   │   │   └── versions/
│   │   │       └── 001_initial.py               # + NEW
│   │   └── pyproject.toml                       # MODIFY
│   │
│   ├── data-pipeline/                           # Same structure
│   │   ├── alembic.ini                          # MODIFY
│   │   ├── alembic/                             # + NEW
│   │   │   ├── env.py                           # + NEW — schema: pipeline
│   │   │   ├── script.py.mako                   # + NEW
│   │   │   └── versions/
│   │   │       └── 001_initial.py               # + NEW
│   │   └── pyproject.toml                       # MODIFY
│   │
│   ├── ai-gateway/                              # Same structure
│   │   ├── alembic.ini                          # MODIFY
│   │   ├── alembic/                             # + NEW
│   │   │   ├── env.py                           # + NEW — schema: gateway
│   │   │   ├── script.py.mako                   # + NEW
│   │   │   └── versions/
│   │   │       └── 001_initial.py               # + NEW
│   │   └── pyproject.toml                       # MODIFY
│   │
│   └── notification/                            # Same structure
│       ├── alembic.ini                          # MODIFY
│       ├── alembic/                             # + NEW
│       │   ├── env.py                           # + NEW — schema: notification
│       │   ├── script.py.mako                   # + NEW
│       │   └── versions/
│       │       └── 001_initial.py               # + NEW
│       └── pyproject.toml                       # MODIFY
│
├── tests/
│   └── integration/
│       └── + test_alembic_migrations.py         # + NEW — migration integration tests
│
└── Makefile                                     # MODIFY — add migrate-all, migrate-service, migrate-status
```

### References

- [Source: eusolicit-docs/planning-artifacts/epic-01-infrastructure-foundation.md#S01.04]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section-8-Database-Strategy]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section-4.2-Core-Technology-Stack (Alembic 1.13+)]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-01.md#P1-alembic-upgrade-head, E01-R-004]
- [Source: eusolicit-docs/implementation-artifacts/1-3-postgresql-schema-design-role-based-access.md#migration-role-privileges]
- [Source: eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py#SERVICE_SCHEMA_MAP, apply_migrations]

## Dev Agent Record

### Agent Model Used

Claude Opus 4 (claude-opus-4-20250514)

### Debug Log References

- SQLAlchemy 2.0 autobegin issue: Initial env.py pattern had `connection.execute(SET search_path)` before `context.begin_transaction()`, causing nested savepoint instead of real commit. Fixed by moving SET search_path inside Alembic-managed transaction.
- Migration file type annotations: ATDD tests use regex `revision\s*[:=]\s*["\']001["\']` which doesn't match Python type-annotated `revision: str = "001"`. Simplified to `revision = "001"`.
- alembic.ini URL: Story 1.1 test expected `asyncpg` in alembic.ini URL. Kept `asyncpg` in alembic.ini (env.py rewrites to psycopg2 at runtime).

### Completion Notes List

- Created 5 schema-aware Alembic environments (client, admin, pipeline, gateway, notification) each with isolated `version_table_schema` and `search_path`
- Created 5 initial migrations (001_initial.py) creating `_migrations_meta` table with service identity in each service's schema
- Updated 5 alembic.ini files with `file_template` for sequential naming and `DATABASE_URL` override pattern
- Added `alembic>=1.13` and `psycopg2-binary>=2.9` to all 5 service pyproject.toml files
- Added `migrate-all`, `migrate-service`, and `migrate-status` Makefile targets with MIGRATION_ORDER enforcement
- All 166 pre-written ATDD tests pass (test_alembic_migrations.py)
- No regressions introduced (53 pre-existing failures all from missing fastapi/eusolicit_kraftdata modules, not from this story)

### File List

**New files:**
- `eusolicit-app/services/client-api/alembic/env.py`
- `eusolicit-app/services/client-api/alembic/script.py.mako`
- `eusolicit-app/services/client-api/alembic/versions/.gitkeep`
- `eusolicit-app/services/client-api/alembic/versions/001_initial.py`
- `eusolicit-app/services/admin-api/alembic/env.py`
- `eusolicit-app/services/admin-api/alembic/script.py.mako`
- `eusolicit-app/services/admin-api/alembic/versions/.gitkeep`
- `eusolicit-app/services/admin-api/alembic/versions/001_initial.py`
- `eusolicit-app/services/data-pipeline/alembic/env.py`
- `eusolicit-app/services/data-pipeline/alembic/script.py.mako`
- `eusolicit-app/services/data-pipeline/alembic/versions/.gitkeep`
- `eusolicit-app/services/data-pipeline/alembic/versions/001_initial.py`
- `eusolicit-app/services/ai-gateway/alembic/env.py`
- `eusolicit-app/services/ai-gateway/alembic/script.py.mako`
- `eusolicit-app/services/ai-gateway/alembic/versions/.gitkeep`
- `eusolicit-app/services/ai-gateway/alembic/versions/001_initial.py`
- `eusolicit-app/services/notification/alembic/env.py`
- `eusolicit-app/services/notification/alembic/script.py.mako`
- `eusolicit-app/services/notification/alembic/versions/.gitkeep`
- `eusolicit-app/services/notification/alembic/versions/001_initial.py`

**Modified files:**
- `eusolicit-app/services/client-api/alembic.ini`
- `eusolicit-app/services/admin-api/alembic.ini`
- `eusolicit-app/services/data-pipeline/alembic.ini`
- `eusolicit-app/services/ai-gateway/alembic.ini`
- `eusolicit-app/services/notification/alembic.ini`
- `eusolicit-app/services/client-api/pyproject.toml`
- `eusolicit-app/services/admin-api/pyproject.toml`
- `eusolicit-app/services/data-pipeline/pyproject.toml`
- `eusolicit-app/services/ai-gateway/pyproject.toml`
- `eusolicit-app/services/notification/pyproject.toml`
- `eusolicit-app/Makefile`

**Story/status files:**
- `eusolicit-docs/implementation-artifacts/1-4-alembic-migration-scaffold.md`
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml`

## Senior Developer Review

**Date:** 2026-04-06
**Reviewer:** Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** APPROVE

### Acceptance Criteria Verification

| AC | Description | Status |
|----|-------------|--------|
| AC1 | Each service has `alembic.ini` and `alembic/` directory | PASS |
| AC2 | Each service's `env.py` uses `search_path` to own schema + `shared` | PASS |
| AC3 | `alembic upgrade head` runs successfully per service | PASS (166 ATDD tests) |
| AC4 | Each service has `001_initial.py` creating `_migrations_meta` | PASS |
| AC5 | Sequential naming convention `NNN_descriptive_name.py` | PASS |
| AC6 | `make migrate-all` runs all 5 services in dependency order | PASS |

Schema mapping verified correct across all 5 services (env.py, 001_initial.py, test SERVICE_CONFIG, db.py SERVICE_SCHEMA_MAP). DO NOT TOUCH files confirmed untouched.

### Review Findings

**Patch (recommended improvements, non-blocking):**

- [ ] [Review][Patch] Makefile migrate-all/migrate-status: bare `cd` causes cascading directory corruption on failure [Makefile:166-170, 177-181] — When `alembic upgrade head` fails, `&& cd ../..` is skipped, corrupting cwd for subsequent loop iterations. Fix: use subshell `(cd services/$$svc && DATABASE_URL=... alembic upgrade head) || exit 1`. Note: implementation faithfully follows the spec's own Makefile pattern, so this is a spec-level defect carried forward.
- [ ] [Review][Patch] Offline mode `run_migrations_offline()` does not emit `SET search_path` [services/*/alembic/env.py] — Online mode sets `search_path` (line 62), but offline mode (SQL script generation via `--sql`) omits it. Generated scripts will target wrong schema for unqualified table references. Low practical impact since offline mode is rarely used and `001_initial.py` uses explicit `schema=SCHEMA`.
- [ ] [Review][Patch] migrate-status swallows all stderr with `2>/dev/null` [Makefile:179] — Auth failures, connection errors, and config errors silently produce "(no migrations run)" instead of the real error. Remove `2>/dev/null` or replace with targeted error handling.

**Deferred (pre-existing patterns or future improvements):**

- [x] [Review][Defer] `get_url()` returns empty string with no validation when both `DATABASE_URL` and `sqlalchemy.url` are unset — produces cryptic SQLAlchemy error. Add `RuntimeError` guard. — deferred, low-impact edge case
- [x] [Review][Defer] No `connect_timeout` on migration engine (`create_engine()`) — migrations hang indefinitely if DB unreachable. Add `connect_args={"connect_timeout": 10}`. — deferred, CI/operability improvement
- [x] [Review][Defer] No advisory lock for concurrent migration safety — two simultaneous `upgrade head` runs could corrupt `alembic_version`. — deferred, not in scope for scaffold story
- [x] [Review][Defer] `psycopg2-binary` in main dependencies — psycopg2 maintainers recommend `psycopg2` (source build) for production. — deferred, production-readiness concern
- [x] [Review][Defer] `script.py.mako` template doesn't include `SCHEMA` constant or schema-targeting reminder — future migration authors may omit `schema=` parameter. — deferred, developer experience improvement
- [x] [Review][Defer] Makefile `MIGRATION_DB_URL` not quoted in shell command — passwords with `&`, `;`, or spaces cause shell interpretation errors. — deferred, edge case

**Dismissed (14 findings):** False positives (script.py.mako existence, async test markers with auto mode), spec-compliant patterns (hardcoded dev credentials in alembic.ini/Makefile per spec), hardcoded constants misidentified as injection vectors, and pre-existing code in DO NOT TOUCH files.

### Summary

| Metric | Value |
|--------|-------|
| Review layers completed | 3/3 (Blind Hunter, Edge Case, Acceptance Auditor) |
| Total findings raised | 23 |
| Dismissed as noise/FP | 14 |
| Patch (non-blocking) | 3 |
| Deferred | 6 |
| Decision needed | 0 |

Implementation is solid. All acceptance criteria met. Schema isolation verified. 166 ATDD tests passing. The 3 patch items are hardening improvements that can be addressed in a follow-up commit.

## Change Log

- 2026-04-06: Story 1.4 implemented — Alembic migration scaffold for all 5 services with schema isolation, initial migrations, Makefile targets, and dependency updates. 166 ATDD tests passing.
- 2026-04-06: Senior Developer Review — APPROVE. 3 non-blocking patch recommendations, 6 deferred, 14 dismissed. All ACs verified PASS.
