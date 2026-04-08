---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-generate-output'
lastSaved: '2026-04-06'
workflowType: 'testarch-atdd'
storyId: '1-4-alembic-migration-scaffold'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-4-alembic-migration-scaffold.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py'
  - 'eusolicit-app/tests/conftest.py'
---

# ATDD Checklist: Story 1.4 — Alembic Migration Scaffold

## Story Summary

**Epic:** E01 — Infrastructure & Monorepo Foundation
**Story:** 1.4 — Alembic Migration Scaffold
**Sprint:** 1 | **Priority:** P1 (Foundation for all future schema evolution)
**Risk-Driven:** Yes — E01-R-004 (Alembic migration ordering conflict, Score: 4 — MEDIUM)

**As a** developer on the EU Solicit platform,
**I want** each of the 5 services to have its own Alembic migration directory with isolated `env.py` targeting its own schema, an initial no-op migration, and a `make migrate-all` command,
**So that** service teams can evolve their database schemas independently via version-controlled migrations, with Alembic's version tracking isolated per-schema to prevent cross-service collisions, and all migrations runnable in a single deterministic command.

---

## TDD Red Phase (Current)

**Status:** RED — 154 of 166 tests fail. 12 tests pass due to pre-existing state (alembic.ini files from Story 1.1, Makefile service references, schema map consistency).

| Test Class | File | Test Functions | Expanded Tests | AC | Priority | Status |
|------------|------|---------------|----------------|----|----|--------|
| `TestAC1AlembicDirectoryStructure` | `tests/integration/test_alembic_migrations.py` | 5 | 25 | AC1 | P1 | 20 FAIL / 5 PASS |
| `TestAC2EnvPySchemaConfig` | `tests/integration/test_alembic_migrations.py` | 7 | 35 | AC2 | P1 | 35 FAIL |
| `TestAC3AlembicUpgradeHead` | `tests/integration/test_alembic_migrations.py` | 4 | 20 | AC3 | P1 | 20 FAIL |
| `TestAC4InitialMigration` | `tests/integration/test_alembic_migrations.py` | 8 | 40 | AC4 | P1 | 40 FAIL |
| `TestAC5MigrationNamingConvention` | `tests/integration/test_alembic_migrations.py` | 3 | 15 | AC5 | P1 | 15 FAIL |
| `TestAC6MakeMigrateAll` | `tests/integration/test_alembic_migrations.py` | 8 | 8 | AC6 | P1 | 7 FAIL / 1 PASS |
| `TestAlembicIniConfiguration` | `tests/integration/test_alembic_migrations.py` | 2 | 10 | AC1 | P2 | 5 FAIL / 5 PASS |
| `TestPyprojectDependencies` | `tests/integration/test_alembic_migrations.py` | 2 | 10 | AC3 | P1 | 10 FAIL |
| `TestMigrationSchemaIsolation` | `tests/integration/test_alembic_migrations.py` | 3 | 3 | AC3 | P1 | 2 FAIL / 1 PASS |
| **Total** | **1 file** | **42 functions** | **166 test cases** | **AC1–AC6** | | **154 FAIL / 12 PASS** |

---

## Acceptance Criteria Coverage

| AC | Description | Test Class(es) | Tests | Priority | Coverage |
|----|-------------|-----------------|-------|----------|----------|
| AC1 | Each service has `alembic.ini` + `alembic/` directory | `TestAC1AlembicDirectoryStructure`, `TestAlembicIniConfiguration` | 35 | P1/P2 | Full — alembic.ini, alembic/ dir, env.py, script.py.mako, versions/ dir for each of 5 services |
| AC2 | env.py uses search_path = own schema + shared | `TestAC2EnvPySchemaConfig` | 35 | P1 | Full — SERVICE_SCHEMA, SHARED_SCHEMA, version_table_schema, search_path, DATABASE_URL, asyncpg→psycopg2 replacement, include_schemas for each of 5 services |
| AC3 | `alembic upgrade head` succeeds for each service | `TestAC3AlembicUpgradeHead`, `TestPyprojectDependencies`, `TestMigrationSchemaIsolation` | 33 | P1 | Full — upgrade head, version table in own schema, downgrade base, current reports head, alembic + psycopg2 deps, no public schema version table, independent version tracking |
| AC4 | Initial migration creates `_migrations_meta` table | `TestAC4InitialMigration` | 40 | P1 | Full — file exists, revision 001, down_revision None, references _migrations_meta, specifies schema, table created in DB, service identity seeded, downgrade cleans up |
| AC5 | NNN_descriptive_name.py naming convention | `TestAC5MigrationNamingConvention` | 15 | P1 | Full — file pattern matches `^\d{3}_[a-z][a-z0-9_]*\.py$`, file_template in alembic.ini, revision is sequential string "001" |
| AC6 | `make migrate-all` runs in dependency order | `TestAC6MakeMigrateAll` | 8 | P1 | Full — migrate-all/migrate-service/migrate-status targets in Makefile, MIGRATION_ORDER defined, make migrate-all succeeds, all services at head, all _migrations_meta tables exist |

**Total coverage: 166 test cases across 6 acceptance criteria — 100% AC coverage.**

---

## Risk Mitigation Verification

### E01-R-004: Alembic Migration Ordering Conflict (Score: 4 — MEDIUM)

| Mitigation | Test Coverage | Status |
|------------|--------------|--------|
| `make migrate-all` enforces dependency order | `test_makefile_defines_migration_order`, `test_make_migrate_all_succeeds` | RED |
| Each service uses own `alembic_version` table in its schema | `test_alembic_version_table_in_own_schema` (×5), `test_no_alembic_version_in_public_schema` | RED |
| Independent version tracking per service | `test_each_service_has_independent_version_tracking` | RED |
| CI runs migrations sequentially | `test_make_migrate_all_succeeds` (verifies sequential execution) | RED |

**Quality gate:** P1 tests >= 95% pass rate. Migration ordering verified.

---

## Test Strategy

### Stack Detection

- **Detected stack:** `fullstack` (Python backend + Node.js/Playwright frontend)
- **Story scope:** Backend only — Alembic migration infrastructure
- **Test framework:** pytest + pytest-asyncio + asyncpg (DB verification) + subprocess (alembic CLI)
- **No E2E/browser tests:** This story has no UI components

### Generation Mode

- **Mode:** AI generation (backend infrastructure story, no browser recording needed)

### Test Level Selection

| Level | Justification | Count |
|-------|--------------|-------|
| Integration (DB) | Migration execution, version table placement, _migrations_meta table creation — requires live PostgreSQL | 63 |
| Integration (subprocess) | Alembic CLI commands, make targets — requires project filesystem + optional DB | 28 |
| Unit (file-based) | Directory structure, file content, config validation — no DB needed | 75 |

### Priority Distribution

| Priority | Tests | Criteria |
|----------|-------|----------|
| **P1** | 151 | All AC-linked tests — foundation for schema evolution, E01-R-004 mitigation |
| **P2** | 15 | alembic.ini validation (script_location, sqlalchemy.url) — already partially passing from pre-existing config |

---

## Failing Tests Created (RED Phase)

### Integration + Unit Tests (166 test cases across 42 functions)

**File:** `eusolicit-app/tests/integration/test_alembic_migrations.py`

#### AC1: Alembic Directory Structure (25 tests)

- **`TestAC1AlembicDirectoryStructure::test_alembic_ini_exists[×5]`** (5 tests)
  - Status: PASS (pre-existing from Story 1.1)
  - Verifies: `alembic.ini` file exists in each service root

- **`TestAC1AlembicDirectoryStructure::test_alembic_directory_exists[×5]`** (5 tests)
  - Status: RED — `alembic/` directory does not exist
  - Verifies: `alembic/` directory exists under each service

- **`TestAC1AlembicDirectoryStructure::test_env_py_exists[×5]`** (5 tests)
  - Status: RED — env.py not created
  - Verifies: `alembic/env.py` exists for each service

- **`TestAC1AlembicDirectoryStructure::test_script_py_mako_exists[×5]`** (5 tests)
  - Status: RED — mako template not created
  - Verifies: `alembic/script.py.mako` exists for each service

- **`TestAC1AlembicDirectoryStructure::test_versions_directory_exists[×5]`** (5 tests)
  - Status: RED — versions/ directory does not exist
  - Verifies: `alembic/versions/` directory exists for each service

#### AC2: env.py Schema Configuration (35 tests)

- **`TestAC2EnvPySchemaConfig::test_env_py_declares_service_schema[×5]`** (5 tests)
  - Status: RED — env.py does not exist
  - Verifies: `SERVICE_SCHEMA = "{own_schema}"` matches SERVICE_SCHEMA_MAP

- **`TestAC2EnvPySchemaConfig::test_env_py_declares_shared_schema[×5]`** (5 tests)
  - Status: RED
  - Verifies: `SHARED_SCHEMA = "shared"` in env.py

- **`TestAC2EnvPySchemaConfig::test_env_py_sets_version_table_schema[×5]`** (5 tests)
  - Status: RED
  - Verifies: `version_table_schema` configured to service's own schema

- **`TestAC2EnvPySchemaConfig::test_env_py_sets_search_path[×5]`** (5 tests)
  - Status: RED
  - Verifies: PostgreSQL `SET search_path TO` present in env.py

- **`TestAC2EnvPySchemaConfig::test_env_py_reads_database_url_from_env[×5]`** (5 tests)
  - Status: RED
  - Verifies: `DATABASE_URL` environment variable read in env.py

- **`TestAC2EnvPySchemaConfig::test_env_py_replaces_asyncpg_with_sync_driver[×5]`** (5 tests)
  - Status: RED
  - Verifies: asyncpg URL rewritten to psycopg2 for Alembic compatibility

- **`TestAC2EnvPySchemaConfig::test_env_py_includes_schemas_enabled[×5]`** (5 tests)
  - Status: RED
  - Verifies: `include_schemas=True` set in Alembic context.configure()

#### AC3: alembic upgrade head (20 tests)

- **`TestAC3AlembicUpgradeHead::test_alembic_upgrade_head_succeeds[×5]`** (5 tests)
  - Status: RED — alembic command fails (no alembic/ directory)
  - Verifies: `alembic upgrade head` returncode == 0

- **`TestAC3AlembicUpgradeHead::test_alembic_version_table_in_own_schema[×5]`** (5 tests)
  - Status: RED — no version table created
  - Verifies: `alembic_version` table in service's own schema, NOT `public` (E01-R-004)

- **`TestAC3AlembicUpgradeHead::test_alembic_downgrade_base_succeeds[×5]`** (5 tests)
  - Status: RED — cannot downgrade without prior upgrade
  - Verifies: Rollback works for each service

- **`TestAC3AlembicUpgradeHead::test_alembic_current_reports_head[×5]`** (5 tests)
  - Status: RED — no migrations applied
  - Verifies: `alembic current` shows revision 001 after upgrade

#### AC4: Initial Migration _migrations_meta (40 tests)

- **`TestAC4InitialMigration::test_initial_migration_file_exists[×5]`** (5 tests)
  - Status: RED — no 001_*.py file in versions/
  - Verifies: `001_initial.py` (or `001_*.py`) exists

- **`TestAC4InitialMigration::test_initial_migration_has_revision_001[×5]`** (5 tests)
  - Status: RED
  - Verifies: `revision = '001'` declared in migration file

- **`TestAC4InitialMigration::test_initial_migration_has_down_revision_none[×5]`** (5 tests)
  - Status: RED
  - Verifies: `down_revision = None` (initial migration)

- **`TestAC4InitialMigration::test_initial_migration_references_migrations_meta[×5]`** (5 tests)
  - Status: RED
  - Verifies: Migration file references `_migrations_meta` table

- **`TestAC4InitialMigration::test_initial_migration_specifies_schema[×5]`** (5 tests)
  - Status: RED
  - Verifies: Migration targets the service's own schema explicitly

- **`TestAC4InitialMigration::test_migrations_meta_table_created_in_schema[×5]`** (5 tests)
  - Status: RED — table not created
  - Verifies: `_migrations_meta` exists in service's schema after upgrade

- **`TestAC4InitialMigration::test_migrations_meta_contains_service_identity[×5]`** (5 tests)
  - Status: RED
  - Verifies: `_migrations_meta` seeded with `('service', '{schema}')` row

- **`TestAC4InitialMigration::test_downgrade_removes_migrations_meta[×5]`** (5 tests)
  - Status: RED
  - Verifies: `_migrations_meta` dropped after `alembic downgrade base`

#### AC5: Sequential Naming Convention (15 tests)

- **`TestAC5MigrationNamingConvention::test_migration_files_use_sequential_naming[×5]`** (5 tests)
  - Status: RED — no migration files exist
  - Verifies: Files match `^\d{3}_[a-z][a-z0-9_]*\.py$` pattern

- **`TestAC5MigrationNamingConvention::test_alembic_ini_has_file_template[×5]`** (5 tests)
  - Status: RED — no `file_template` in alembic.ini
  - Verifies: `file_template` configured in alembic.ini [alembic] section

- **`TestAC5MigrationNamingConvention::test_migration_revision_is_sequential_string[×5]`** (5 tests)
  - Status: RED — no migration files
  - Verifies: `revision = '001'` (not hex hash like `a1b2c3d4e5f6`)

#### AC6: make migrate-all (8 tests)

- **`TestAC6MakeMigrateAll::test_makefile_has_migrate_all_target`** (1 test)
  - Status: RED — no `migrate-all:` target in Makefile
  - Verifies: Makefile defines `migrate-all` target

- **`TestAC6MakeMigrateAll::test_makefile_has_migrate_service_target`** (1 test)
  - Status: RED — no `migrate-service:` target in Makefile
  - Verifies: Makefile defines `migrate-service` target

- **`TestAC6MakeMigrateAll::test_makefile_has_migrate_status_target`** (1 test)
  - Status: RED — no `migrate-status:` target in Makefile
  - Verifies: Makefile defines `migrate-status` target

- **`TestAC6MakeMigrateAll::test_makefile_migrate_all_references_all_services`** (1 test)
  - Status: PASS (service names already appear in Makefile from Story 1.1)
  - Verifies: All 5 service names present in Makefile

- **`TestAC6MakeMigrateAll::test_makefile_defines_migration_order`** (1 test)
  - Status: RED — no MIGRATION_ORDER variable
  - Verifies: `MIGRATION_ORDER` variable defined in Makefile

- **`TestAC6MakeMigrateAll::test_make_migrate_all_succeeds`** (1 test)
  - Status: RED — target does not exist
  - Verifies: `make migrate-all` completes without error (P1 from epic test design)

- **`TestAC6MakeMigrateAll::test_all_services_at_head_after_migrate_all`** (1 test)
  - Status: RED — cannot run migrate-all
  - Verifies: All 5 services at revision 001 after migrate-all

- **`TestAC6MakeMigrateAll::test_all_migrations_meta_tables_exist_after_migrate_all`** (1 test)
  - Status: RED — cannot run migrate-all
  - Verifies: All 5 `_migrations_meta` tables exist after migrate-all

#### Cross-Cutting: alembic.ini Config (10 tests)

- **`TestAlembicIniConfiguration::test_alembic_ini_script_location[×5]`** (5 tests)
  - Status: PASS (pre-existing `script_location = alembic`)
  - Verifies: `script_location = alembic` in each alembic.ini

- **`TestAlembicIniConfiguration::test_alembic_ini_has_sqlalchemy_url[×5]`** (5 tests)
  - Status: RED — existing alembic.ini uses `%(DB_USER)s` interpolation which configparser cannot resolve
  - Verifies: `sqlalchemy.url` is readable by configparser (env.py override pattern)

#### Cross-Cutting: Dependency Verification (10 tests)

- **`TestPyprojectDependencies::test_alembic_dependency_present[×5]`** (5 tests)
  - Status: RED — no `alembic>=1.13` in pyproject.toml
  - Verifies: `alembic>=1.13` in dependencies

- **`TestPyprojectDependencies::test_psycopg2_dependency_present[×5]`** (5 tests)
  - Status: RED — no `psycopg2-binary>=2.9` in pyproject.toml
  - Verifies: `psycopg2-binary>=2.9` for Alembic sync driver

#### Cross-Cutting: Schema Isolation / E01-R-004 (3 tests)

- **`TestMigrationSchemaIsolation::test_no_alembic_version_in_public_schema`** (1 test)
  - Status: RED — cannot run migrations to verify
  - Verifies: No `alembic_version` in `public` schema after all migrations

- **`TestMigrationSchemaIsolation::test_each_service_has_independent_version_tracking`** (1 test)
  - Status: RED — cannot run migrations to verify
  - Verifies: 5 separate `alembic_version` tables in 5 separate schemas

- **`TestMigrationSchemaIsolation::test_service_schema_map_consistency`** (1 test)
  - Status: PASS — test config matches SERVICE_SCHEMA_MAP from eusolicit-test-utils
  - Verifies: Test constants align with architecture constants

---

## Data Factories Created

N/A — This story tests migration infrastructure (Alembic directories, env.py config, migration execution). No application-level data factories are needed. The `_migrations_meta` table data is created by the migration itself.

---

## Fixtures Created

N/A — Tests use module-level helper functions:
- `_service_dir(service)` — Returns absolute path to service directory
- `_alembic_dir(service)` — Returns absolute path to alembic/ directory
- `_connect_as_migration()` — Raw asyncpg connection as `migration_role`
- `_run_alembic(service, *args)` — Runs alembic CLI via subprocess with DATABASE_URL injection

These follow the same pattern as Story 1.3's test helpers.

---

## Mock Requirements

N/A — All tests run against real filesystem and real PostgreSQL. No mocks required — the purpose is to verify actual Alembic configuration and migration execution.

---

## Required data-testid Attributes

N/A — No UI components in this story.

---

## Implementation Checklist

### Task 1: Create Alembic Directory Structure (AC: 1, 2, 5)

**Files to create (15 files):**
- `services/{client-api,admin-api,data-pipeline,ai-gateway,notification}/alembic/env.py`
- `services/{client-api,admin-api,data-pipeline,ai-gateway,notification}/alembic/script.py.mako`
- `services/{client-api,admin-api,data-pipeline,ai-gateway,notification}/alembic/versions/.gitkeep`

**Tests that will pass when complete:**

- [ ] `TestAC1AlembicDirectoryStructure` (20 remaining) — alembic/ dirs, env.py, script.py.mako, versions/
- [ ] `TestAC2EnvPySchemaConfig` (35 tests) — SERVICE_SCHEMA, SHARED_SCHEMA, version_table_schema, search_path, DATABASE_URL, psycopg2 driver, include_schemas
- [ ] `TestAC5MigrationNamingConvention::test_alembic_ini_has_file_template` (5 tests) — file_template in alembic.ini

### Task 2: Create Initial Migrations (AC: 3, 4, 5)

**Files to create (5 files):**
- `services/{client-api,admin-api,data-pipeline,ai-gateway,notification}/alembic/versions/001_initial.py`

**Tests that will pass when complete:**

- [ ] `TestAC4InitialMigration` (40 tests) — file exists, revision 001, down_revision None, _migrations_meta, schema reference, table creation, service identity, downgrade cleanup
- [ ] `TestAC5MigrationNamingConvention` (10 remaining) — sequential naming, sequential revision string
- [ ] `TestAC3AlembicUpgradeHead` (20 tests) — upgrade head, version table in own schema, downgrade base, current reports head

### Task 3: Update alembic.ini Files (AC: 1, 5)

**Files to modify (5 files):**
- `services/{client-api,admin-api,data-pipeline,ai-gateway,notification}/alembic.ini`

**Tests that will pass when complete:**

- [ ] `TestAlembicIniConfiguration::test_alembic_ini_has_sqlalchemy_url` (5 tests) — sqlalchemy.url readable
- [ ] `TestAC5MigrationNamingConvention::test_alembic_ini_has_file_template` (5 tests) — file_template option

### Task 4: Add Dependencies to pyproject.toml (AC: 3)

**Files to modify (5 files):**
- `services/{client-api,admin-api,data-pipeline,ai-gateway,notification}/pyproject.toml`

**Tests that will pass when complete:**

- [ ] `TestPyprojectDependencies::test_alembic_dependency_present` (5 tests) — alembic>=1.13
- [ ] `TestPyprojectDependencies::test_psycopg2_dependency_present` (5 tests) — psycopg2-binary>=2.9

### Task 5: Add Makefile Targets (AC: 6)

**File to modify:** `Makefile`

**Tests that will pass when complete:**

- [ ] `TestAC6MakeMigrateAll` (7 remaining) — migrate-all, migrate-service, migrate-status targets, MIGRATION_ORDER, make migrate-all succeeds, all at head, all _migrations_meta exist

### Task 6: Verify Schema Isolation (E01-R-004)

**No files to create — verification only**

**Tests that will pass when all above tasks complete:**

- [ ] `TestMigrationSchemaIsolation` (2 remaining) — no public schema version table, independent version tracking

**Estimated Total Effort:** 4–6 hours

---

## Running Tests

```bash
# Run all ATDD tests for this story (154 will fail in RED phase)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_alembic_migrations.py -v

# Run with short traceback (recommended for RED phase verification)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_alembic_migrations.py -v --tb=line

# Run only file-structure tests (no DB required)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_alembic_migrations.py -v -k "AC1 or AC2 or AC5 or Pyproject or AlembicIniConfiguration"

# Run only DB-dependent tests (requires Docker Compose PostgreSQL running)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_alembic_migrations.py -v -k "AC3 or AC4 or MigrationSchemaIsolation or migrate_all_succeeds or at_head or meta_tables_exist"

# Run only Makefile target tests (no DB required)
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_alembic_migrations.py -v -k "TestAC6"

# Run with verbose output showing parametrized test IDs
cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_alembic_migrations.py -v --tb=long

# Run integration tests with Docker Compose (full environment)
make reset-db && make up && cd eusolicit-app && .venv/bin/python -m pytest tests/integration/test_alembic_migrations.py -v
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete)

**TEA Agent Responsibilities:**

- 166 test cases written across 42 test functions (154 failing, 12 passing on pre-existing state)
- Tests organized by acceptance criteria with clear priority tags
- E01-R-004 mitigation verified: version table isolation, no public schema, independent tracking
- Tests use subprocess for alembic CLI, asyncpg for DB verification, pathlib for filesystem checks
- Cross-referenced against `SERVICE_SCHEMA_MAP` from `eusolicit-test-utils`

**Verification:**

```
154 failed, 12 passed in 0.23s
```

- 12 pre-existing passes: 5 `alembic.ini` exists, 5 `script_location`, 1 Makefile service references, 1 schema map consistency
- 154 failures: All due to missing alembic directories, env.py files, migrations, Makefile targets, and dependencies
- Tests will fail due to missing implementation, not test bugs

---

### GREEN Phase (DEV Team — Next Steps)

**DEV Agent Responsibilities:**

1. **Task 1:** Create `alembic/env.py` for each service (follow the pattern in story Dev Notes — critical: SERVICE_SCHEMA, version_table_schema, search_path, DATABASE_URL, asyncpg→psycopg2 replacement)
2. **Task 1:** Create `alembic/script.py.mako` for each service
3. **Task 1:** Create `alembic/versions/` directories with `.gitkeep`
4. **Task 2:** Create `alembic/versions/001_initial.py` for each service (creates `_migrations_meta` table in own schema)
5. **Task 3:** Update `alembic.ini` files (sqlalchemy.url, file_template)
6. **Task 4:** Add `alembic>=1.13` and `psycopg2-binary>=2.9` to each service's `pyproject.toml`
7. **Task 5:** Add `migrate-all`, `migrate-service`, `migrate-status` targets to Makefile with `MIGRATION_ORDER`
8. Run `make reset-db && make up` to start PostgreSQL
9. Run `pytest tests/integration/test_alembic_migrations.py -v`
10. All 166 tests pass — GREEN

**Key Implementation Notes:**

- `env.py` MUST replace `+asyncpg` with `+psycopg2` — Alembic requires sync driver
- `version_table_schema=SERVICE_SCHEMA` places `alembic_version` in service's own schema — CRITICAL for E01-R-004
- `SET search_path TO {SERVICE_SCHEMA}, {SHARED_SCHEMA}` in connection setup
- Sequential naming: use `--rev-id 001` or manually set `revision = "001"`
- `alembic.ini` needs `file_template = %%(rev)s_%%(slug)s` for sequential filenames
- Passwords MUST match `.env.example` exactly
- Tests run against `eusolicit_test` database (matching `TEST_DATABASE_URL` default)

---

### REFACTOR Phase (DEV Team — After All Tests Pass)

1. Verify all 166 tests pass consistently (run 3x)
2. Run `make migrate-all` manually to verify deterministic ordering
3. Run full regression suite (653+ existing tests) to verify no regressions
4. Review env.py DRY patterns (consider shared helper module if 5 env.py files are identical except for schema name)
5. Ready for code review

---

## Next Steps

1. **Review this checklist** — Verify test coverage aligns with team expectations
2. **Implement Story 1.4** — Follow the implementation checklist above (Tasks 1–5)
3. **Run tests** — `pytest tests/integration/test_alembic_migrations.py -v`
4. **Verify GREEN** — All 166 tests pass
5. **Story 1.6 depends on this** — eusolicit-common health checks may query `_migrations_meta`

---

## Knowledge Base References Applied

- **test-quality.md** — Deterministic, isolated, self-cleaning test design; subprocess timeout for Alembic CLI
- **test-levels-framework.md** — Integration level for DB operations; unit-equivalent for file structure validation
- **test-priorities-matrix.md** — P1 for migration infrastructure (E01-R-004 Score: 4)
- **test-healing-patterns.md** — Clear assertion messages with diagnostic context (schema names, file paths, stderr output)

---

## Notes

- **Test database:** Tests connect to `eusolicit_test` database (matching `TEST_DATABASE_URL` default in conftest.py). Docker Compose init script creates identical schemas in both `eusolicit` and `eusolicit_test`.
- **Subprocess isolation:** Alembic tests use `subprocess.run()` with captured output and 30s timeouts. The `DATABASE_URL` environment variable is injected per-invocation.
- **asyncpg for verification:** DB state assertions use raw `asyncpg` connections as `migration_role`, matching the pattern from Story 1.3 tests.
- **SERVICE_SCHEMA_MAP:** Cross-referenced between test constants and `eusolicit_test_utils.db.SERVICE_SCHEMA_MAP` to prevent drift.
- **Pre-existing passes (12):** Tests that check `alembic.ini` existence and `script_location` pass because Story 1.1 created placeholder alembic.ini files. The `test_makefile_migrate_all_references_all_services` passes because service names appear in existing Makefile targets. These tests are retained as regression guards.
- **No skip markers:** Unlike Story 1.3's ATDD tests which used `@pytest.mark.skip`, these tests fail immediately with descriptive assertions. This follows the "fail fast with clear diagnostics" pattern — the developer sees exactly what needs to be created.

---

**Generated by BMad TEA Agent** — 2026-04-06
