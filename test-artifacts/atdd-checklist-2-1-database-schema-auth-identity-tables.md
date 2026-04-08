---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-07'
workflowType: 'testarch-atdd'
storyId: '2-1-database-schema-auth-identity-tables'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/2-1-database-schema-auth-identity-tables.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-02.md'
  - 'eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py'
  - 'eusolicit-app/services/client-api/tests/conftest.py'
  - 'eusolicit-app/services/client-api/alembic/env.py'
  - 'eusolicit-app/services/client-api/pyproject.toml'
---

# ATDD Checklist: Story 2.1 — Database Schema Auth & Identity Tables

## Story Summary

**Epic:** E02 — Authentication & Identity
**Story:** 2.1 — Database Schema Auth & Identity Tables
**Sprint:** 1 | **Priority:** P2 (Schema foundation for all Epic 2 stories)
**Risk-Driven:** Yes — E02-R-007 (Audit log completeness and append-only enforcement, Score: 4 — MEDIUM)

**As a** developer on the EU Solicit platform,
**I want** Alembic migration `002_auth_identity_tables` to create all core auth and identity tables in the `client` schema (plus `shared.audit_log` in the `shared` schema), backed by SQLAlchemy ORM models in `app/models/`,
**So that** every subsequent Epic 2 story has a ready, version-controlled database foundation with correct column types, constraints, indexes, and enums — and `shared.audit_log` is available for all mutation-logging across the platform.

---

## TDD Red Phase (Current)

**Status:** RED — All 42 tests fail. No pre-existing passes.

| Test Class | File | Test Functions | Expanded Tests | AC | Priority | Status |
|------------|------|---------------|----------------|----|----------|--------|
| `TestAC1MigrationExecution` | `tests/integration/test_002_migration.py` | 4 | 4 | AC1 | P2 | 4 FAIL |
| `TestAC2TableStructure` | `tests/integration/test_002_migration.py` | 3 | 9 | AC2 | P2 | 9 FAIL |
| `TestAC3CompanyRoleEnum` | `tests/integration/test_002_migration.py` | 2 | 2 | AC3 | P2 | 2 FAIL |
| `TestAC4EntityPermissionEnum` | `tests/integration/test_002_migration.py` | 2 | 2 | AC4 | P2 | 2 FAIL |
| `TestAC5ForeignKeys` | `tests/integration/test_002_migration.py` | 4 | 4 | AC5 | P2 | 4 FAIL |
| `TestAC6Indexes` | `tests/integration/test_002_migration.py` | 3 | 3 | AC6 | P2 | 3 FAIL |
| `TestAC7AuditLogSchema` | `tests/integration/test_002_migration.py` | 1 | 1 | AC7 | P2 | 1 FAIL |
| `TestAC8OrmModels` | `tests/integration/test_002_migration.py` | 6 | 15 | AC8 | P2 | 15 FAIL |
| `TestAC9AppendOnlyEnforcement` | `tests/integration/test_002_migration.py` | 2 | 2 | AC9 | P2 | 2 FAIL |
| **Total** | **1 file** | **27 functions** | **42 test cases** | **AC1–AC9** | | **42 FAIL / 0 PASS** |

---

## Acceptance Criteria Coverage

| AC | Description | Test Class(es) | Tests | Priority | Epic Test ID | Coverage |
|----|-------------|-----------------|-------|----------|--------------|----------|
| AC1 | Migration 002 runs cleanly (upgrade/downgrade/idempotent) | `TestAC1MigrationExecution` | 4 | P2 | E02-P2-001 | Full — file exists, upgrade head returncode, idempotency, downgrade base |
| AC2 | All 7 tables exist in correct schemas with correct column types | `TestAC2TableStructure` | 9 | P2 | E02-P2-001 | Full — 7 parametrized table existence tests + users columns + JSONB column types |
| AC3 | company_role enum = {admin, bid_manager, contributor, reviewer, read_only} | `TestAC3CompanyRoleEnum` | 2 | P2 | E02-P2-002 | Full — exact count (5) and exact value set |
| AC4 | entity_permission enum = {read, write, manage} | `TestAC4EntityPermissionEnum` | 2 | P2 | — | Full — exact count (3) and exact value set |
| AC5 | FK relationships correct with ON DELETE CASCADE / SET NULL rules | `TestAC5ForeignKeys` | 4 | P2 | — | Full — information_schema FK check, cascade delete test, SET NULL test, composite PK |
| AC6 | Required indexes: unique email, entity_permissions lookup, audit_log entity | `TestAC6Indexes` | 3 | P2 | — | Full — duplicate email rejection, pg_indexes for both composite indexes |
| AC7 | shared.audit_log in the shared schema | `TestAC7AuditLogSchema` | 1 | P2 | E02-P2-003 | Full — verify shared schema, verify NOT in client schema |
| AC8 | ORM models created, env.py updated to Base.metadata | `TestAC8OrmModels` | 15 | P2 | — | Full — directory, 10 model files, Base class, enums values, audit_log schema, env.py |
| AC9 | Integration tests pass + append-only enforcement | `TestAC9AppendOnlyEnforcement` | 2 | P2 | E02-P2-014 | Full — UPDATE denied, DELETE denied for client_api_role |

**Total coverage: 42 test cases across 9 acceptance criteria — 100% AC coverage.**

---

## Epic-Level Test ID Mapping

| Epic Test ID | Description | Test Class | Tests |
|--------------|-------------|------------|-------|
| **E02-P2-001** | Alembic migration creates all E02 tables with correct column types, constraints, indexes | `TestAC1MigrationExecution`, `TestAC2TableStructure` | 13 |
| **E02-P2-002** | Role enum includes exactly: admin, bid_manager, contributor, reviewer, read_only | `TestAC3CompanyRoleEnum` | 2 |
| **E02-P2-003** | `shared.audit_log` lives in the `shared` schema | `TestAC7AuditLogSchema` | 1 |
| **E02-P2-014** | Audit log append-only — application code cannot UPDATE or DELETE rows | `TestAC9AppendOnlyEnforcement` | 2 |

**Quality gate:** 100% P2 pass rate for 4 mapped epic test IDs (E02-P2-001, E02-P2-002, E02-P2-003, E02-P2-014) + all 42 integration tests green.

---

## Risk Mitigation Verification

### E02-R-007: Audit Log Completeness and Append-Only Enforcement (Score: 4 — MEDIUM)

| Mitigation | Test Coverage | Status |
|------------|--------------|--------|
| REVOKE UPDATE ON shared.audit_log FROM client_api_role | `test_client_api_role_cannot_update_audit_log_rows` | RED |
| REVOKE DELETE ON shared.audit_log FROM client_api_role | `test_client_api_role_cannot_delete_audit_log_rows` | RED |
| audit_log in shared schema (not client) | `test_audit_log_lives_in_shared_schema_not_client` | RED |
| JSONB before/after columns exist for state capture | `test_jsonb_columns_have_jsonb_data_type` | RED |

---

## Test Strategy

### Stack Detection

- **Detected stack:** `fullstack` (Python backend + Node.js/Playwright frontend)
- **Story scope:** Backend only — Alembic migration + SQLAlchemy ORM models
- **Test framework:** pytest + pytest-asyncio + sqlalchemy[asyncio] + asyncpg + subprocess
- **No E2E/browser tests:** This story has no UI components

### Generation Mode

- **Mode:** AI generation (backend migration infrastructure, no browser recording needed)

### Test Level Selection

| Level | Justification | Count |
|-------|--------------|-------|
| Integration (DB + async) | Table existence, column types, FK constraints, indexes, enum values, append-only — all require live PostgreSQL | 29 |
| Integration (subprocess) | Alembic CLI migration execution, idempotency, downgrade — require project filesystem + DB | 3 |
| Unit (filesystem) | ORM model files exist, content correctness, env.py updated — no DB needed | 11 |

### Priority Distribution

| Priority | Tests | Criteria |
|----------|-------|----------|
| **P2** | 43 | All tests — schema validation story; no P0/P1 assigned per epic test design |

---

## Failing Tests Created (RED Phase)

### Integration + Unit Tests (42 test cases across 27 functions)

**File:** `eusolicit-app/services/client-api/tests/integration/test_002_migration.py`

---

#### AC1: Migration Execution (4 tests)

- **`TestAC1MigrationExecution::test_migration_002_file_exists_with_correct_revision`**
  - Status: RED — `002_*.py` not found in `alembic/versions/`
  - Verifies: File matching `002_*.py` exists; `down_revision = "001"` declared

- **`TestAC1MigrationExecution::test_upgrade_head_from_001_baseline_succeeds`**
  - Status: RED — `alembic upgrade head` returns non-zero (no 002 migration)
  - Verifies: `alembic upgrade head` returncode == 0 from 001 baseline

- **`TestAC1MigrationExecution::test_upgrade_head_is_idempotent`**
  - Status: RED — second upgrade fails for same reason
  - Verifies: Second `alembic upgrade head` returncode == 0; `alembic current` shows 002

- **`TestAC1MigrationExecution::test_downgrade_base_rolls_back_completely`**
  - Status: RED — cannot downgrade what doesn't exist
  - Verifies: `alembic downgrade base` returncode == 0; current does not show 002

---

#### AC2: Table Structure (9 tests)

- **`TestAC2TableStructure::test_table_exists_in_correct_schema[×7]`** (7 parametrized)
  - Status: RED — migration not applied; tables don't exist
  - Parameters: `(client, users)`, `(client, companies)`, `(client, company_memberships)`, `(client, subscriptions)`, `(client, entity_permissions)`, `(client, espd_profiles)`, `(shared, audit_log)`
  - Verifies: `information_schema.tables` returns count=1 for each (schema, table_name)

- **`TestAC2TableStructure::test_users_table_has_all_9_required_columns`**
  - Status: RED — table doesn't exist
  - Verifies: All 9 columns present: id, email, hashed_password, full_name, google_sub, is_active, email_verified, created_at, updated_at

- **`TestAC2TableStructure::test_jsonb_columns_have_jsonb_data_type`**
  - Status: RED — tables don't exist
  - Verifies: 6 JSONB columns across companies + audit_log have `data_type = 'jsonb'`

---

#### AC3: company_role Enum (2 tests)

- **`TestAC3CompanyRoleEnum::test_company_role_enum_has_exactly_5_values`**
  - Status: RED — enum `client.company_role` does not exist
  - Verifies: `SELECT COUNT FROM pg_enum WHERE enumtypid = 'client.company_role'::regtype = 5`

- **`TestAC3CompanyRoleEnum::test_company_role_enum_values_exact_set`**
  - Status: RED — enum does not exist
  - Verifies: Exact set `{admin, bid_manager, contributor, reviewer, read_only}` — no extras, no missing

---

#### AC4: entity_permission Enum (2 tests)

- **`TestAC4EntityPermissionEnum::test_entity_permission_enum_has_exactly_3_values`**
  - Status: RED — enum `client.entity_permission` does not exist
  - Verifies: `SELECT COUNT FROM pg_enum WHERE enumtypid = 'client.entity_permission'::regtype = 3`

- **`TestAC4EntityPermissionEnum::test_entity_permission_enum_values_exact_set`**
  - Status: RED — enum does not exist
  - Verifies: Exact set `{read, write, manage}`

---

#### AC5: Foreign Key Constraints (4 tests)

- **`TestAC5ForeignKeys::test_all_required_fk_constraints_exist_in_information_schema`**
  - Status: RED — tables don't exist
  - Verifies: 7 FK relationships present in `information_schema` (company_memberships ×2, entity_permissions ×3, subscriptions ×1, espd_profiles ×1)

- **`TestAC5ForeignKeys::test_company_memberships_user_id_fk_cascades_on_user_delete`**
  - Status: RED — table doesn't exist
  - Verifies: DELETE user → company_memberships row cascade-deleted (ON DELETE CASCADE)

- **`TestAC5ForeignKeys::test_entity_permissions_granted_by_fk_set_null_on_grantor_delete`**
  - Status: RED — table doesn't exist
  - Verifies: DELETE grantor user → `entity_permissions.granted_by` becomes NULL (ON DELETE SET NULL)

- **`TestAC5ForeignKeys::test_company_memberships_composite_pk_rejects_duplicate_user_company_pair`**
  - Status: RED — table doesn't exist
  - Verifies: Duplicate (user_id, company_id) INSERT → `IntegrityError` (composite PK enforced)

---

#### AC6: Required Indexes (3 tests)

- **`TestAC6Indexes::test_unique_index_on_users_email_prevents_duplicates`**
  - Status: RED — table/index doesn't exist
  - Verifies: Duplicate email INSERT → `IntegrityError` (unique index enforced)

- **`TestAC6Indexes::test_composite_index_entity_permissions_lookup_exists_in_pg_indexes`**
  - Status: RED — index doesn't exist
  - Verifies: `ix_entity_permissions_lookup` in `pg_indexes` on `client.entity_permissions`

- **`TestAC6Indexes::test_composite_index_audit_log_entity_exists_in_pg_indexes`**
  - Status: RED — index doesn't exist
  - Verifies: `ix_audit_log_entity` in `pg_indexes` on `shared.audit_log`

---

#### AC7: shared.audit_log Schema (1 test)

- **`TestAC7AuditLogSchema::test_audit_log_lives_in_shared_schema_not_client`**
  - Status: RED — migration not applied
  - Verifies: `information_schema.tables` shows `audit_log` with `table_schema = 'shared'`; NOT in `client` schema

---

#### AC8: ORM Models + env.py (15 tests)

- **`TestAC8OrmModels::test_models_directory_exists`** (1 test)
  - Status: RED — `models/` directory does not exist
  - Verifies: `services/client-api/src/client_api/models/` directory exists

- **`TestAC8OrmModels::test_orm_model_file_exists[×10]`** (10 parametrized tests)
  - Status: RED — none of the model files exist
  - Parameters: `__init__.py`, `base.py`, `enums.py`, `user.py`, `company.py`, `company_membership.py`, `subscription.py`, `entity_permission.py`, `espd_profile.py`, `audit_log.py`
  - Verifies: Each model file exists in `models/` directory

- **`TestAC8OrmModels::test_base_py_defines_declarative_base_subclass`** (1 test)
  - Status: RED — `base.py` does not exist
  - Verifies: `DeclarativeBase` imported; `class Base` defined

- **`TestAC8OrmModels::test_enums_py_defines_company_role_and_entity_permission_with_correct_values`** (1 test)
  - Status: RED — `enums.py` does not exist
  - Verifies: `CompanyRole` and `EntityPermission` classes present; all 5 + 3 enum values present

- **`TestAC8OrmModels::test_audit_log_model_declares_shared_schema_in_table_args`** (1 test)
  - Status: RED — `audit_log.py` does not exist
  - Verifies: `__table_args__` declared; `'shared'` referenced in schema declaration

- **`TestAC8OrmModels::test_env_py_sets_target_metadata_to_base_dot_metadata`** (1 test)
  - Status: RED — `env.py` has `target_metadata = None` (confirmed pre-migration state)
  - Verifies: `'target_metadata = None'` NOT in env.py; `'Base.metadata'` IS in env.py

---

#### AC9: Append-Only Enforcement — E02-P2-014 (2 tests)

- **`TestAC9AppendOnlyEnforcement::test_client_api_role_cannot_update_audit_log_rows`**
  - Status: RED — migration not applied; REVOKE not issued; table doesn't exist
  - Verifies: `UPDATE shared.audit_log` as `client_api_role` → `ProgrammingError("permission denied")`

- **`TestAC9AppendOnlyEnforcement::test_client_api_role_cannot_delete_audit_log_rows`**
  - Status: RED — migration not applied; REVOKE not issued; table doesn't exist
  - Verifies: `DELETE FROM shared.audit_log` as `client_api_role` → `ProgrammingError("permission denied")`

---

## Data Factories Created

N/A — Tests use direct SQL inserts with generated UUIDs (`uuid.uuid4()`). No application-level data factories required for this schema-only migration story. All test data is rolled back via `await conn.rollback()` within each test.

---

## Fixtures Created

| Fixture | Scope | Purpose |
|---------|-------|---------|
| `migration_engine` | module | Async SQLAlchemy engine connected as `migration_role` (DDL access) |
| `client_api_engine` | module | Async SQLAlchemy engine connected as `client_api_role` (restricted access) |
| `migrated_db` | module | Applies `alembic upgrade head` once per module; teardown runs `alembic downgrade 001` |
| `seeded_audit_log_row` | function | Seeds a `shared.audit_log` row for append-only tests; auto-cleans via migration_role |

**Helper functions:**
- `_run_alembic(service_root, *args)` — Runs Alembic CLI via subprocess with `DATABASE_URL` injection; 60s timeout; returns `CompletedProcess`

---

## Mock Requirements

N/A — All tests run against real PostgreSQL (`eusolicit_test` database). No mocks required — the purpose is to verify actual migration execution, database state, and permission enforcement.

---

## Required data-testid Attributes

N/A — No UI components in this story.

---

## Dependencies Before Running Tests

| Requirement | Source | Status |
|-------------|--------|--------|
| `asyncpg>=0.29` in client-api dependencies | Story AC: Task 4.1 | Must add to `pyproject.toml` |
| `eusolicit-test-utils` in dev extras | Story AC: Task 4.2 | Must add to `pyproject.toml` optional-deps |
| PostgreSQL running with `eusolicit_test` DB | `docker-compose.yml` | `make up` |
| `migration_role` has DDL on client+shared schemas | `infra/postgres/init/01-init-schemas-and-roles.sql` | Pre-existing from Story 1.3 |
| `client_api_role` has restricted access to `shared.audit_log` | Migration 002 REVOKE | Created by this story's migration |
| `001_initial.py` applied before running tests | Story 1.4 (DONE) | Pre-existing |

---

## Implementation Checklist

### Task 1: Create SQLAlchemy Models (AC: 3, 4, 8)

**Tests that will pass when complete:**

- [ ] `TestAC8OrmModels::test_models_directory_exists` — models/ directory created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[__init__.py]` — package init created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[base.py]` + `test_base_py_defines_declarative_base_subclass` — Base class created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[enums.py]` + `test_enums_py_defines_company_role_and_entity_permission_with_correct_values` — enums created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[user.py]` — User model created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[company.py]` — Company model created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[company_membership.py]` — CompanyMembership model created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[subscription.py]` — Subscription model created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[entity_permission.py]` — EntityPermission model created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[espd_profile.py]` — ESPDProfile model created
- [ ] `TestAC8OrmModels::test_orm_model_file_exists[audit_log.py]` + `test_audit_log_model_declares_shared_schema_in_table_args` — AuditLog model with schema='shared'

### Task 2: Create Alembic Migration 002 (AC: 1, 2, 3, 4, 5, 6, 7)

**Tests that will pass when complete:**

- [ ] `TestAC1MigrationExecution::test_migration_002_file_exists_with_correct_revision` — 002_auth_identity_tables.py created
- [ ] `TestAC1MigrationExecution::test_upgrade_head_from_001_baseline_succeeds` — upgrade head returncode 0
- [ ] `TestAC1MigrationExecution::test_upgrade_head_is_idempotent` — second upgrade no-op
- [ ] `TestAC1MigrationExecution::test_downgrade_base_rolls_back_completely` — downgrade base cleans up
- [ ] `TestAC2TableStructure::test_table_exists_in_correct_schema[×7]` — all 7 tables created
- [ ] `TestAC2TableStructure::test_users_table_has_all_9_required_columns` — users columns complete
- [ ] `TestAC2TableStructure::test_jsonb_columns_have_jsonb_data_type` — 6 JSONB columns created
- [ ] `TestAC3CompanyRoleEnum::test_company_role_enum_has_exactly_5_values` — enum created with 5 values
- [ ] `TestAC3CompanyRoleEnum::test_company_role_enum_values_exact_set` — exact enum values match
- [ ] `TestAC4EntityPermissionEnum::test_entity_permission_enum_has_exactly_3_values` — enum created
- [ ] `TestAC4EntityPermissionEnum::test_entity_permission_enum_values_exact_set` — exact values match
- [ ] `TestAC5ForeignKeys::test_all_required_fk_constraints_exist_in_information_schema` — 7 FKs present
- [ ] `TestAC5ForeignKeys::test_company_memberships_user_id_fk_cascades_on_user_delete` — cascade verified
- [ ] `TestAC5ForeignKeys::test_entity_permissions_granted_by_fk_set_null_on_grantor_delete` — SET NULL verified
- [ ] `TestAC5ForeignKeys::test_company_memberships_composite_pk_rejects_duplicate_user_company_pair` — PK enforced
- [ ] `TestAC6Indexes::test_unique_index_on_users_email_prevents_duplicates` — unique email enforced
- [ ] `TestAC6Indexes::test_composite_index_entity_permissions_lookup_exists_in_pg_indexes` — ix present
- [ ] `TestAC6Indexes::test_composite_index_audit_log_entity_exists_in_pg_indexes` — ix present
- [ ] `TestAC7AuditLogSchema::test_audit_log_lives_in_shared_schema_not_client` — schema verified
- [ ] `TestAC9AppendOnlyEnforcement::test_client_api_role_cannot_update_audit_log_rows` — REVOKE UPDATE verified
- [ ] `TestAC9AppendOnlyEnforcement::test_client_api_role_cannot_delete_audit_log_rows` — REVOKE DELETE verified

### Task 3: Update alembic/env.py (AC: 8)

**Tests that will pass when complete:**

- [ ] `TestAC8OrmModels::test_env_py_sets_target_metadata_to_base_dot_metadata` — `target_metadata = Base.metadata`

### Task 4: Add Missing Dependencies (AC: 1, 8)

**Changes to `services/client-api/pyproject.toml`:**

- Add `asyncpg>=0.29` to `[project] dependencies`
- Add `eusolicit-test-utils` to `[project.optional-dependencies] dev`

**Tests that become runnable when complete:** All async integration tests (currently fail earlier due to missing asyncpg driver at runtime)

---

## Running Tests

```bash
# Run all ATDD tests for this story (all 43 will fail in RED phase)
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v

# Run with short traceback (recommended for RED phase diagnostics)
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v --tb=short

# Run only filesystem/unit tests (no DB required, no migration needed)
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v -k "TestAC1 and file_exists or TestAC8"

# Run only DB integration tests (requires Docker Compose PostgreSQL running)
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v -k "TestAC2 or TestAC3 or TestAC4 or TestAC5 or TestAC6 or TestAC7 or TestAC9"

# Run only migration execution tests
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v -k "TestAC1"

# Run only append-only enforcement tests
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v -k "TestAC9"

# Run full suite with Docker Compose (full environment)
make reset-db && make up && cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/test_002_migration.py -v

# Run all Epic 2 integration tests (after more stories are implemented)
cd eusolicit-app && .venv/bin/python -m pytest services/client-api/tests/integration/ -v
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete)

**TEA Agent Responsibilities:**

- 42 test cases written across 27 test functions (all 43 failing)
- Tests organized by acceptance criteria; 1:1 mapping with AC1–AC9
- Epic test IDs E02-P2-001, E02-P2-002, E02-P2-003, E02-P2-014 explicitly tagged
- E02-R-007 (audit log completeness) mitigated via TestAC9 append-only tests
- Tests use subprocess for Alembic CLI, sqlalchemy async for DB verification, pathlib for filesystem
- Module-scoped `migrated_db` fixture minimizes migration setup/teardown overhead
- `client_api_engine` fixture tests with restricted app-level DB role

**Expected RED phase output:**

```
43 failed, 0 passed
```

Failure modes:
- `TestAC1` (subprocess): returncode != 0 — `002_auth_identity_tables.py` missing from `alembic/versions/`
- `TestAC2`–`TestAC7`, `TestAC9` (DB): `migrated_db` fixture fails → all dependent tests ERROR (migration not applied)
- `TestAC8` (filesystem): `assert MODELS_DIR.is_dir()` fails — `models/` directory does not exist; `target_metadata = None` assertion fails on env.py content check

---

### GREEN Phase (DEV Team — Next Steps)

**DEV Agent Responsibilities:**

1. **Task 1.1–1.10:** Create `models/` directory and all 10 ORM model files
   - `base.py` — `DeclarativeBase` subclass
   - `enums.py` — `CompanyRole` (5 values), `EntityPermission` (3 values)
   - `user.py`, `company.py`, `company_membership.py`, `subscription.py`
   - `entity_permission.py`, `espd_profile.py`, `audit_log.py` (schema='shared')
   - `__init__.py` — exports `Base` and all model classes
2. **Task 2.1–2.12:** Create `alembic/versions/002_auth_identity_tables.py`
   - `down_revision = "001"`, `revision = "002"`
   - Create enum types BEFORE tables that reference them (PostgreSQL requirement)
   - Create all 7 tables with correct columns, FK constraints, ON DELETE rules
   - Create 3 required indexes (ix_users_email, ix_entity_permissions_lookup, ix_audit_log_entity)
   - Issue `REVOKE UPDATE, DELETE ON shared.audit_log FROM client_api_role`
   - Implement `downgrade()` in reverse order (tables → enums)
3. **Task 3.1–3.3:** Update `alembic/env.py`
   - Import `Base` from `client_api.models`
   - Set `target_metadata = Base.metadata` (replace `None`)
4. **Task 4.1–4.2:** Update `pyproject.toml`
   - Add `asyncpg>=0.29` to dependencies
   - Add `eusolicit-test-utils` to dev optional-dependencies
5. Run `make reset-db && make up` to start PostgreSQL
6. Run `pytest services/client-api/tests/integration/test_002_migration.py -v`
7. All 42 tests pass — GREEN

**Key Implementation Notes:**

- `PostgreSQL ENUM creation order`: Create `company_role` and `entity_permission` enum types BEFORE creating tables that reference them. In `downgrade()`, drop tables BEFORE dropping enum types.
- `entity_permissions.granted_by` FK: Use `ForeignKey("client.users.id", ondelete="SET NULL")` — NOT CASCADE. Test `test_entity_permissions_granted_by_fk_set_null_on_grantor_delete` verifies this.
- `audit_log.user_id`: No FK constraint on this column — it's cross-schema (shared → client). Validation at application layer only (per dev notes).
- `REVOKE UPDATE, DELETE`: Must be the last operation in `upgrade()`. In `downgrade()`, this is automatically reversed (the table is dropped).
- `asyncpg` URL handling: `env.py` already rewrites `+asyncpg` → `+psycopg2`. Tests pass `TEST_DATABASE_URL` as DATABASE_URL env var (asyncpg format); env.py handles the rewrite.
- `down_revision = "001"`: Must match the string `"001"`, not a hex hash. `test_migration_002_file_exists_with_correct_revision` verifies this.
- Teardown order: `migrated_db` fixture teardowns to `001` (not `base`) to preserve Story 1.4's `_migrations_meta` table.

---

### REFACTOR Phase (DEV Team — After All Tests Pass)

1. Verify all 43 tests pass consistently (run 3×)
2. Run `alembic check` — verify no pending autogenerate changes after migration applied
3. Run `make migrate-all` to verify the full migration pipeline still works (001 → 002 for client-api)
4. Run the full regression suite to verify no regressions from env.py change
5. Review model file DRY patterns (e.g., `created_at`/`updated_at` as a mixin)
6. Ready for code review

---

## Next Steps

1. **Review this checklist** — Verify test coverage aligns with team expectations
2. **Implement Story 2.1** — Follow the implementation checklist above (Tasks 1–4)
3. **Add asyncpg + eusolicit-test-utils** to `pyproject.toml` (Task 4) before running async tests
4. **Run tests** — `pytest services/client-api/tests/integration/test_002_migration.py -v`
5. **Verify GREEN** — All 42 tests pass
6. **Story 2.2 depends on this** — Registration endpoint requires `users`, `companies`, `company_memberships` tables

---

## Knowledge Base References Applied

- **test-quality.md** — Deterministic, isolated, self-cleaning test design; module-scope fixture minimizes DB setup overhead
- **test-levels-framework.md** — Integration level for DB state; unit-equivalent for filesystem assertions; subprocess for Alembic CLI
- **test-priorities-matrix.md** — P2 for migration/schema validation; all 4 epic test IDs (E02-P2-001/002/003/014) covered
- **test-healing-patterns.md** — All assertions include diagnostic context (schema names, expected vs actual values, SQL queries used)
- **data-factories.md** — Direct SQL inserts with UUID generation; rollback for test isolation (no persistent test data)

---

## Notes

- **Test database:** Tests connect to `eusolicit_test` database. `TEST_DATABASE_URL` defaults to `migration_role` connection; `CLIENT_API_DATABASE_URL` defaults to `client_api_role` connection.
- **Async test runner:** `pytest-asyncio` is already in `client-api` dev dependencies. Tests use `@pytest.mark.asyncio` decorator. Ensure `asyncio_mode = "auto"` or individual markers are respected (check `pyproject.toml` for `[tool.pytest.ini_options]`).
- **asyncpg dependency:** The test file uses `postgresql+asyncpg://` URLs with SQLAlchemy async engine. `asyncpg>=0.29` must be added to `pyproject.toml` (Story Task 4.1) before tests run successfully.
- **No skip markers:** Tests fail immediately with descriptive assertion messages (not `pytest.mark.skip`). This follows the "fail fast with clear diagnostics" RED phase pattern established in Story 1.4.
- **Module-scoped fixture ordering:** `TestAC1` tests (subprocess-based) run without `migrated_db` dependency. `TestAC2`–`TestAC9` tests trigger `migrated_db` setup on first use. `TestAC1::test_downgrade_base_rolls_back_completely` upgrades back to head at the end of its body to avoid interfering with module-scoped state.
- **env.py confirmation:** `services/client-api/alembic/env.py` currently contains `target_metadata = None` (confirmed pre-implementation). `TestAC8OrmModels::test_env_py_sets_target_metadata_to_base_dot_metadata` will fail with: _"env.py still has 'target_metadata = None'"_.
- **Pre-existing passes:** None. All 42 tests fail in the current state (no 002 migration, no models/, env.py has None). Unlike Story 1.4 (12 pre-passes), this story starts at 0/43 GREEN.

---

**Generated by BMad TEA Agent** — 2026-04-07
