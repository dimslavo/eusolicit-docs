# Story 1.3: PostgreSQL Schema Design & Role-Based Access

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **developer on the EU Solicit platform**,
I want **a PostgreSQL initialization script that creates 6 schemas, per-service database roles with strict access boundaries, and a `shared.tenants` foundational table**,
so that **each service is enforced at the database level to only access its own data, preventing accidental or malicious cross-service data access, and establishing the multi-tenancy foundation for all future epics**.

## Acceptance Criteria

1. 6 schemas created: `client`, `pipeline`, `admin`, `notification`, `gateway`, `shared`
2. 6 service roles created: `client_api_role`, `admin_api_role`, `data_pipeline_role`, `ai_gateway_role`, `notification_role`, `migration_role`
3. Each service role has full CRUD on its own schema and SELECT on `shared` schema
4. `migration_role` has DDL privileges on all schemas (used by Alembic)
5. `shared.tenants` table exists with columns: `id UUID PK`, `name`, `slug UNIQUE`, `created_at`, `updated_at`
6. A test script verifies that `client_api_role` cannot write to `admin` schema (negative access test)
7. Init script runs automatically on first `docker compose up` via PostgreSQL's `/docker-entrypoint-initdb.d/`

## Tasks / Subtasks

- [x] Task 1: Create idempotent PostgreSQL init SQL script (AC: 1, 2, 3, 4, 5, 7)
  - [x] 1.1 Create `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql`
  - [x] 1.2 Create 6 schemas with `CREATE SCHEMA IF NOT EXISTS`
  - [x] 1.3 Create 6 roles with `CREATE ROLE ... IF NOT EXISTS` with LOGIN and passwords matching `.env.example`
  - [x] 1.4 For each service role: GRANT USAGE + CRUD on own schema, GRANT USAGE + SELECT on `shared` schema
  - [x] 1.5 Use `ALTER DEFAULT PRIVILEGES` for EACH schema owner so future tables auto-inherit permissions
  - [x] 1.6 Grant `migration_role` full DDL (CREATE, ALTER, DROP, USAGE, SELECT, INSERT, UPDATE, DELETE) on ALL 6 schemas with `ALTER DEFAULT PRIVILEGES`
  - [x] 1.7 Create `shared.tenants` table: `id UUID DEFAULT gen_random_uuid() PRIMARY KEY`, `name VARCHAR(255) NOT NULL`, `slug VARCHAR(255) NOT NULL UNIQUE`, `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`
  - [x] 1.8 Wrap entire script in idempotent blocks: `DO $$ ... $$` for conditional logic, `IF NOT EXISTS` for DDL
- [x] Task 2: Create negative access test script (AC: 6)
  - [x] 2.1 Create `eusolicit-app/tests/integration/test_db_schema_isolation.py`
  - [x] 2.2 Test: `client_api_role` can SELECT from `shared.tenants`
  - [x] 2.3 Test: `client_api_role` can INSERT/UPDATE/DELETE in `client` schema (create temp table to verify)
  - [x] 2.4 Test: `client_api_role` CANNOT INSERT into `admin` schema (expect permission denied)
  - [x] 2.5 Test: `client_api_role` CANNOT INSERT into `pipeline` schema (expect permission denied)
  - [x] 2.6 Test: Each service role CANNOT write to other service schemas
  - [x] 2.7 Test: `migration_role` CAN create tables in any schema
  - [x] 2.8 Test: All 6 schemas exist
  - [x] 2.9 Test: `shared.tenants` table exists with correct columns and constraints
- [x] Task 3: Verify Docker Compose integration (AC: 7)
  - [x] 3.1 Confirm init script runs on `docker compose up` (file in `infra/postgres/init/` already mounted to `/docker-entrypoint-initdb.d/`)
  - [x] 3.2 Verify `make reset-db` re-runs init script (volumes destroyed + recreated)
  - [x] 3.3 Test idempotency: run init twice without errors

## Dev Notes

### CRITICAL: Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `docker-compose.yml` | PostgreSQL service already mounts `./infra/postgres/init:/docker-entrypoint-initdb.d` | **DO NOT TOUCH** — forward-compatible mount from Story 1.2 |
| `.env.example` | Role passwords already defined (see below) | **DO NOT TOUCH** — passwords must match init SQL |
| `Makefile` | Docker targets including `make reset-db` | **DO NOT TOUCH** |
| `tests/conftest.py` | DB fixtures using `migration_role` connection | **DO NOT TOUCH** |
| `packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py` | Schema constants + test helpers | **DO NOT TOUCH** — your schemas must match `ALL_SCHEMAS` tuple |

### Files to CREATE (New)

| File | Purpose |
|------|---------|
| `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql` | Idempotent init script — schemas, roles, grants, shared.tenants |
| `eusolicit-app/tests/integration/test_db_schema_isolation.py` | Negative access tests + schema verification |

### Role-to-Password Mapping (from .env.example — MUST match exactly)

| Role | Password | Own Schema | Connection String Pattern |
|------|----------|-----------|---------------------------|
| `client_api_role` | `client_api_password` | `client` | `postgresql+asyncpg://client_api_role:client_api_password@...` |
| `admin_api_role` | `admin_api_password` | `admin` | `postgresql+asyncpg://admin_api_role:admin_api_password@...` |
| `data_pipeline_role` | `pipeline_password` | `pipeline` | `postgresql+asyncpg://data_pipeline_role:pipeline_password@...` |
| `ai_gateway_role` | `gateway_password` | `gateway` | `postgresql+asyncpg://ai_gateway_role:gateway_password@...` |
| `notification_role` | `notification_password` | `notification` | `postgresql+asyncpg://notification_role:notification_password@...` |
| `migration_role` | `migration_password` | ALL (DDL) | `postgresql+asyncpg://migration_role:migration_password@...` |

### SQL Init Script Pattern

The file MUST be named with a numeric prefix (e.g., `01-`) because PostgreSQL's docker-entrypoint runs scripts in alphabetical order. Future stories may add `02-*.sql` etc.

**Idempotency approach:**
```sql
-- Roles: use DO $$ block with IF NOT EXISTS pattern
DO $$
BEGIN
  IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = 'client_api_role') THEN
    CREATE ROLE client_api_role LOGIN PASSWORD 'client_api_password';
  END IF;
END
$$;

-- Schemas: native IF NOT EXISTS
CREATE SCHEMA IF NOT EXISTS client;

-- Grants: re-running GRANT is safe (idempotent by nature)
GRANT USAGE ON SCHEMA client TO client_api_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA client TO client_api_role;

-- CRITICAL: ALTER DEFAULT PRIVILEGES so future tables inherit grants
ALTER DEFAULT PRIVILEGES IN SCHEMA client GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO client_api_role;

-- Tables: IF NOT EXISTS
CREATE TABLE IF NOT EXISTS shared.tenants ( ... );
```

### GRANT Strategy (PostgreSQL-Specific)

**Two levels of grants are required:**

1. **Existing tables**: `GRANT ... ON ALL TABLES IN SCHEMA <schema> TO <role>` — applies to tables that exist NOW
2. **Future tables**: `ALTER DEFAULT PRIVILEGES IN SCHEMA <schema> GRANT ... ON TABLES TO <role>` — applies to tables created LATER by Alembic migrations

Both are needed. Without `ALTER DEFAULT PRIVILEGES`, any table created by Story 1.4 (Alembic) or later would NOT be accessible by service roles.

**IMPORTANT**: `ALTER DEFAULT PRIVILEGES` applies to objects created by the role executing the statement. The init script runs as the PostgreSQL superuser (`eusolicit`), so the default privileges apply to tables created BY the superuser. However, Alembic runs as `migration_role`. Therefore, you MUST ALSO run:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE migration_role IN SCHEMA client
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO client_api_role;
```

This ensures tables created by `migration_role` (via Alembic) inherit the correct service role permissions.

### Architecture Cross-Schema Access Matrix

The epic AC says "CRUD on own schema + SELECT on shared". The architecture document specifies ADDITIONAL cross-schema READ access for some roles:

| Role | Own Schema (CRUD) | shared (SELECT) | Additional READ Access |
|------|-------------------|-----------------|----------------------|
| `client_api_role` | `client` | `shared` | `pipeline` (opportunities) |
| `admin_api_role` | `admin` | `shared` | `client`, `pipeline` |
| `data_pipeline_role` | `pipeline` | `shared` | None |
| `ai_gateway_role` | `gateway` | `shared` | None |
| `notification_role` | `notification` | `shared` | `client` (alerts, calendar), `pipeline` (opportunities) |

**For this story**: Implement the EPIC's acceptance criteria (CRUD on own + SELECT on shared). The additional cross-schema reads from the architecture are noted here for future reference but are NOT required by the ACs. They can be added later when actual cross-service queries are implemented in E02+.

**Architecture also specifies** shared schema is WRITABLE by all services (for `audit_log` and `usage_meters` tables). The epic says SELECT-only on shared. Follow the epic's AC (SELECT on shared) — write access to shared tables will be granted when those tables are created in later stories.

### shared.tenants Table Design

```sql
CREATE TABLE IF NOT EXISTS shared.tenants (
    id         UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    name       VARCHAR(255) NOT NULL,
    slug       VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Notes:
- `gen_random_uuid()` is built into PostgreSQL 16 (no extension needed since PG 13)
- `TIMESTAMPTZ` (not `TIMESTAMP`) for timezone-aware storage — architecture mandates EU data residency
- `slug UNIQUE` enforces tenant uniqueness for URL routing
- No `company_id` FK — this IS the foundational tenant table that other schemas reference

### Testing Strategy

**Test file**: `eusolicit-app/tests/integration/test_db_schema_isolation.py`

Tests require a running PostgreSQL instance with the init script applied. Use `psycopg2` (sync) or `asyncpg` directly for role-specific connections — the existing `conftest.py` uses `migration_role` which has full access. Tests need to connect AS each service role to verify boundaries.

**Connection pattern for per-role testing:**
```python
import asyncpg

# Connect as specific role to test permissions
conn = await asyncpg.connect(
    user="client_api_role",
    password="client_api_password",
    host="localhost",
    port=5432,
    database="eusolicit",
)
```

**Negative access test pattern:**
```python
import pytest
import asyncpg

async def test_client_role_cannot_write_admin_schema():
    conn = await asyncpg.connect(user="client_api_role", ...)
    with pytest.raises(asyncpg.InsufficientPrivilegeError):
        await conn.execute("INSERT INTO admin.test_table ...")
    await conn.close()
```

**Important**: Negative tests need a table to exist in the target schema. Either:
- Create test tables via `migration_role` in test setup, OR
- Use the `shared.tenants` table (which exists) and try writing from non-shared roles, OR
- Create temporary tables in each schema during test setup using `migration_role`

Recommended: Use `migration_role` to create a temp test table in each schema during setup, then test CRUD boundaries per role.

### Test Expectations (from Epic-Level Test Design)

This story maps to the following test scenarios from `test-design-epic-01.md`:

| Priority | Test Description | Risk Link | Verification |
|----------|-----------------|-----------|--------------|
| **P0** | PostgreSQL 6 schemas created + role access boundaries enforced | E01-R-001 (Score 6) | Verify each of 6 roles: own schema CRUD + shared SELECT + cross-schema write rejected |
| **P1** | PostgreSQL negative access — client_api_role cannot write to admin schema | E01-R-001 | INSERT to `admin` schema as `client_api_role` → permission denied error |

**Risk E01-R-001 (HIGH, Score 6):** Schema isolation enforcement failure — misconfigured DB roles allow cross-schema writes, violating data boundaries all future epics depend on. This is the HIGHEST-priority security risk in Epic 1. Tests MUST be thorough.

**Quality gate**: P0 schema isolation tests must pass 100%. Security scenarios (SEC category) pass 100%. No waivers.

### Existing Test Infrastructure to Reuse

| Component | Path | Relevance |
|-----------|------|-----------|
| Schema constants | `packages/eusolicit-test-utils/.../db.py` — `ALL_SCHEMAS`, `SERVICE_SCHEMA_MAP` | Import and validate against these |
| Root conftest | `tests/conftest.py` — `database_url` fixture (migration_role) | Use for setup operations |
| DB session fixtures | `tests/conftest.py` — `db_engine`, `db_session` | Use migration_role session for table creation in test setup |

### Cross-Story Dependencies

- **Story 1.1** (DONE): Created monorepo structure, service directories, pyproject.toml files
- **Story 1.2** (DONE): Created `docker-compose.yml` with PostgreSQL mounted to `./infra/postgres/init:/docker-entrypoint-initdb.d` and `.env.example` with all role passwords
- **Story 1.4** (NEXT): Alembic migrations will use `migration_role` to create tables — depends on `migration_role` having DDL on all schemas from this story
- **Story 1.6**: eusolicit-common will use service roles for health checks — depends on roles existing

### PostgreSQL Docker Entrypoint Behavior

PostgreSQL's docker-entrypoint ONLY runs scripts in `/docker-entrypoint-initdb.d/` on FIRST initialization (when the data volume is empty). Re-running `docker compose up` on an existing volume will NOT re-run the init script. To re-trigger:
- `make reset-db` (destroys volumes, then restarts)
- `docker compose down -v && docker compose up -d postgres`

The init script runs as the `POSTGRES_USER` (superuser `eusolicit` per `.env.example`).

### Project Structure Notes

**Target directory tree changes from this story** (new items marked with `+`):

```
eusolicit-app/
├── infra/
│   └── postgres/
│       └── init/
│           ├── .gitkeep                                    # EXISTS
│           └── + 01-init-schemas-and-roles.sql            # NEW — idempotent init script
│
└── tests/
    └── integration/
        ├── __init__.py                                     # EXISTS
        └── + test_db_schema_isolation.py                  # NEW — negative access tests
```

### References

- [Source: eusolicit-docs/planning-artifacts/epic-01-infrastructure-foundation.md#S01.03]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#Section-8-Database-Strategy]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#ADR-1.5-Database-Design]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-01.md#P0-schema-isolation, E01-R-001]
- [Source: eusolicit-docs/implementation-artifacts/1-2-docker-compose-local-development-environment.md#forward-compatible-postgresql-mount]
- [Source: eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py#ALL_SCHEMAS]

## Dev Agent Record

### Agent Model Used

Claude Opus 4 (claude-sonnet-4-20250514)

### Debug Log References

- Initial ATDD run: 171/172 passed — `TestIdempotency.test_schemas_survive_reapplication` failed because `migration_role` lacked `CREATE ON DATABASE` privilege (needed for `CREATE SCHEMA` DDL). Fixed by adding `GRANT CREATE ON DATABASE ... TO migration_role` for both databases.
- Second ATDD run: 172/172 passed.
- Lint fix: Removed unused imports (`Any`, `SERVICE_SCHEMA_MAP`) and extraneous f-string prefix from pre-written ATDD test template.

### Completion Notes List

- **Task 1 (Init SQL Script)**: Created `01-init-schemas-and-roles.sql` with 4-phase structure:
  - Phase 1: Cluster-level role creation (6 roles with LOGIN, passwords matching .env.example)
  - Phase 2: Schema/grant setup for `eusolicit` database (6 schemas, per-role CRUD grants, ALTER DEFAULT PRIVILEGES for both superuser and migration_role created tables, shared.tenants table)
  - Phase 3: Conditional `eusolicit_test` database creation via `\gexec`
  - Phase 4: Identical schema/grant setup replicated for `eusolicit_test` database
  - Includes sequence grants (`GRANT USAGE ON ALL SEQUENCES`) for serial column support
  - Includes `GRANT CREATE ON DATABASE` for migration_role schema DDL
- **Task 2 (ATDD Tests)**: Pre-written ATDD tests activated by removing `@pytest.mark.skip` decorators. 172 tests covering all 7 ACs: schema existence, role existence/LOGIN, own-schema CRUD, shared SELECT-only, migration DDL, shared.tenants structure, negative cross-schema write, Docker init file validation, default privileges for migration_role, and idempotency. All 172 pass.
- **Task 3 (Docker Integration)**: Verified init script runs on `docker compose up` (mounted at `/docker-entrypoint-initdb.d/`), `make reset-db` re-triggers init (volumes destroyed + recreated), and script is idempotent (re-run produces only benign NOTICE, no errors).
- **Full regression**: 653 tests passed, 0 failed.
- **Lint**: ruff check passes clean on modified test file.

### Change Log

- 2026-04-06: Story 1.3 implementation complete — created init SQL script, activated ATDD tests (172 pass), verified Docker integration and idempotency.

### File List

- `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql` — **NEW**: Idempotent PostgreSQL init script (schemas, roles, grants, shared.tenants, test database setup)
- `eusolicit-app/tests/integration/test_db_schema_isolation.py` — **MODIFIED**: Removed `@pytest.mark.skip` decorators and `_SKIP_REASON` constant to activate ATDD tests; fixed unused imports and f-string lint issue
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — **MODIFIED**: Story status updated ready-for-dev → in-progress → review
- `eusolicit-docs/implementation-artifacts/1-3-postgresql-schema-design-role-based-access.md` — **MODIFIED**: Tasks marked complete, Dev Agent Record populated

## Senior Developer Review

**Date:** 2026-04-06
**Verdict:** REVIEW: Approve
**Reviewer:** Claude Opus 4 (bmad-code-review, 3-layer adversarial)

### Review Layers Executed

| Layer | Findings | Status |
|-------|----------|--------|
| Blind Hunter (no-context adversarial) | 13 raw | completed |
| Edge Case Hunter (project-aware) | 15 raw | completed |
| Acceptance Auditor (spec compliance) | 0 — all 7 ACs pass | completed |

### AC Compliance Summary

All 7 acceptance criteria fully satisfied:
- [x] AC1: 6 schemas created (client, pipeline, admin, notification, gateway, shared)
- [x] AC2: 6 service roles created with LOGIN privilege
- [x] AC3: Each service role has CRUD on own schema + SELECT on shared
- [x] AC4: migration_role has DDL on all schemas + GRANT CREATE ON DATABASE
- [x] AC5: shared.tenants table matches exact column spec (UUID PK, name, slug UNIQUE, timestamptz)
- [x] AC6: Negative access tests — comprehensive cross-schema write denial (INSERT/UPDATE/DELETE)
- [x] AC7: Init script at infra/postgres/init/01-init-schemas-and-roles.sql, Docker mount verified

### Triaged Findings

**Patch (2) — minor code quality, non-blocking:**
- [ ] [Review][Patch] Connection leak in TestAC4 finally blocks — `conn.execute(DROP)` before `conn.close()` in finally: if DROP fails, close() is skipped [test_db_schema_isolation.py:441-444,467-468,488-489]
- [ ] [Review][Patch] `test_schemas_match_test_utils_constant` hardcodes `$1-$6` positional params — breaks if `ALL_SCHEMAS` changes [test_db_schema_isolation.py:127-131]

**Deferred (7) — valid observations, out of story scope:**
- [x] [Review][Defer] No `REVOKE` on `public` schema — PG16 mitigates CREATE, USAGE remains — deferred, defense-in-depth
- [x] [Review][Defer] Role creation not idempotent for password rotation — IF NOT EXISTS skips password update — deferred, production concern
- [x] [Review][Defer] No `updated_at` trigger on `shared.tenants` — DEFAULT now() only fires on INSERT — deferred, future story
- [x] [Review][Defer] Orphaned test tables on assertion failure — mitigated by IF NOT EXISTS pattern — deferred, refactoring
- [x] [Review][Defer] No cross-schema SELECT denial test — only write denial tested — deferred, PG16 handles by default
- [x] [Review][Defer] No `search_path` configured for service roles — deferred, application convention concern
- [x] [Review][Defer] `_DB_NAME` vs `TEST_DATABASE_URL` env var split-brain risk — deferred, same defaults

**Dismissed (11):** Hardcoded passwords (by design), psql meta-commands (Docker design), migration_role CRUD (per spec), empty-row negative tests (PG behavior reliable), f-string SQL (hardcoded constants), weak static check (supplementary), no pooling (inappropriate), no CONNECT grant (PG default), db.py f-string (DO NOT TOUCH), per-grantor limitation (speculative), cross-schema SELECT (PG16 handles).

### Quality Assessment

| Dimension | Rating | Notes |
|-----------|--------|-------|
| Correctness | ✅ Excellent | All ACs met, 172 tests passing, 653 full regression |
| Security | ✅ Strong | Schema isolation properly enforced, negative access comprehensive |
| Test Coverage | ✅ Thorough | P0/P1 risk E01-R-001 fully covered, ATDD approach |
| Architecture Alignment | ✅ Aligned | Matches ALL_SCHEMAS, SERVICE_SCHEMA_MAP, .env.example, Docker mount |
| Idempotency | ✅ Verified | IF NOT EXISTS, DO $$ blocks, idempotency tests pass |
| Code Quality | ⚠️ Good | 2 minor patch items noted, not blockers |
