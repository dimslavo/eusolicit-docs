---
stepsCompleted: ['step-01-preflight-and-context', 'step-02-generation-mode', 'step-03-test-strategy', 'step-04-generate-tests', 'step-05-validate-and-complete']
lastStep: 'step-05-validate-and-complete'
lastSaved: '2026-04-06'
workflowType: 'testarch-atdd'
storyId: '1-3-postgresql-schema-design-role-based-access'
inputDocuments:
  - 'eusolicit-docs/implementation-artifacts/1-3-postgresql-schema-design-role-based-access.md'
  - 'eusolicit-docs/test-artifacts/test-design-epic-01.md'
  - 'eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py'
  - 'eusolicit-app/tests/conftest.py'
---

# ATDD Checklist: Story 1.3 — PostgreSQL Schema Design & Role-Based Access

## Story Summary

**Epic:** E01 — Infrastructure & Monorepo Foundation
**Story:** 1.3 — PostgreSQL Schema Design & Role-Based Access
**Sprint:** 1 | **Priority:** P0 (Database schema isolation is prerequisite for all future data access)
**Risk-Driven:** Yes — E01-R-001 (Schema isolation enforcement failure, Score: 6 — HIGH)

**As a** developer on the EU Solicit platform
**I want** a PostgreSQL initialization script that creates 6 schemas, per-service database roles with strict access boundaries, and a `shared.tenants` foundational table
**So that** each service is enforced at the database level to only access its own data, preventing accidental or malicious cross-service data access, and establishing the multi-tenancy foundation for all future epics.

---

## TDD Red Phase (Current)

**Status:** RED — All tests use `@pytest.mark.skip(reason="ATDD RED: Init script not implemented — Story 1.3")` and will be skipped until the init SQL script is implemented.

| Test Class | File | Test Functions | Expanded Tests | AC | Priority | Status |
|------------|------|---------------|----------------|----|----|--------|
| `TestAC1SchemasExist` | `tests/integration/test_db_schema_isolation.py` | 2 | 7 | AC1 | P0 | Skipped (RED) |
| `TestAC2ServiceRolesExist` | `tests/integration/test_db_schema_isolation.py` | 3 | 17 | AC2 | P0 | Skipped (RED) |
| `TestAC3ServiceRoleCrudOwnSchema` | `tests/integration/test_db_schema_isolation.py` | 4 | 20 | AC3 | P0 | Skipped (RED) |
| `TestAC3SharedSchemaSelectAccess` | `tests/integration/test_db_schema_isolation.py` | 4 | 20 | AC3 | P0 | Skipped (RED) |
| `TestAC4MigrationRoleDDL` | `tests/integration/test_db_schema_isolation.py` | 4 | 24 | AC4 | P1 | Skipped (RED) |
| `TestAC5SharedTenantsTable` | `tests/integration/test_db_schema_isolation.py` | 8 | 8 | AC5 | P0 | Skipped (RED) |
| `TestAC6NegativeAccessCrossSchemaWriteDenied` | `tests/integration/test_db_schema_isolation.py` | 5 | 62 | AC6 | P0 | Skipped (RED) |
| `TestAC7DockerInitScript` | `tests/integration/test_db_schema_isolation.py` | 7 | 7 | AC7 | P1 | Skipped (RED) |
| `TestDefaultPrivilegesForMigrationRole` | `tests/integration/test_db_schema_isolation.py` | 1 | 5 | AC3/AC4 | P1 | Skipped (RED) |
| `TestIdempotency` | `tests/integration/test_db_schema_isolation.py` | 2 | 2 | AC7 | P2 | Skipped (RED) |
| **Total** | **1 file** | **40 functions** | **172 test cases** | **AC1–AC7** | | **All skipped** |

---

## Acceptance Criteria Coverage

| AC | Description | Test Class(es) | Expanded Tests | Priority | Coverage |
|----|-------------|-----------------|----------------|----------|----------|
| AC1 | 6 schemas created | `TestAC1SchemasExist` | 7 | P0 | Full — each schema verified individually + cross-check with test-utils constant |
| AC2 | 6 service roles created | `TestAC2ServiceRolesExist` | 17 | P0 | Full — existence, LOGIN privilege, connection test per role |
| AC3 | CRUD on own schema + SELECT on shared | `TestAC3ServiceRoleCrudOwnSchema`, `TestAC3SharedSchemaSelectAccess` | 40 | P0 | Full — INSERT/SELECT/UPDATE/DELETE per role on own schema; SELECT + write-denied on shared |
| AC4 | migration_role DDL on all schemas | `TestAC4MigrationRoleDDL`, `TestDefaultPrivilegesForMigrationRole` | 29 | P1 | Full — CREATE/ALTER/DROP + CRUD per schema; default privilege inheritance |
| AC5 | shared.tenants table structure | `TestAC5SharedTenantsTable` | 8 | P0 | Full — table existence, all 5 columns, types, constraints, defaults |
| AC6 | Negative access (cross-schema write denied) | `TestAC6NegativeAccessCrossSchemaWriteDenied` | 62 | P0 | Full — explicit tests for client→admin and client→pipeline per story AC; parametrized INSERT/UPDATE/DELETE across all 20 role×schema combinations |
| AC7 | Init script in docker-entrypoint | `TestAC7DockerInitScript`, `TestIdempotency` | 9 | P1/P2 | Full — file existence, naming, content inspection, idempotency |

**Total coverage: 172 expanded test cases across 7 acceptance criteria — 100% AC coverage.**

---

## Risk Mitigation Verification

### E01-R-001: Schema Isolation Enforcement Failure (Score: 6 — HIGH)

| Mitigation | Test Coverage | Status |
|------------|--------------|--------|
| Negative access tests per role | 62 tests: every service role tested against INSERT/UPDATE/DELETE on every non-owned schema | RED |
| `ALTER DEFAULT PRIVILEGES` verified | 5 tests: migration_role creates table, service role verifies CRUD access | RED |
| `shared.tenants` SELECT-only for service roles | 20 tests: 5 roles × (SELECT + no-INSERT + no-UPDATE + no-DELETE) | RED |
| Schemas match architecture constants | Cross-checked against `ALL_SCHEMAS` from `eusolicit-test-utils` | RED |

**Quality gate:** P0 schema isolation tests must pass 100%. Security scenarios (SEC category) pass 100%. No waivers.

---

## Test Strategy

### Stack Detection

- **Detected stack:** `fullstack` (Python backend + Node.js/Playwright frontend)
- **Story scope:** Backend only — PostgreSQL schema/role configuration
- **Test framework:** pytest + pytest-asyncio + asyncpg (direct role connections)
- **No E2E/browser tests:** This story has no UI components

### Generation Mode

- **Mode:** AI generation (backend story, no browser recording needed)

### Test Level Selection

All tests are **integration tests** requiring a running PostgreSQL instance:

| Level | Justification | Count |
|-------|--------------|-------|
| Integration | Database operations, role permissions, schema existence — requires live PostgreSQL | 165 |
| Unit (file-based) | Init script file existence and content validation — no DB needed | 7 |

### Priority Distribution

| Priority | Tests | Criteria |
|----------|-------|----------|
| **P0** | 134 | Schema existence, role existence, CRUD boundaries, cross-schema write denial, shared.tenants structure — security-critical, blocks all future epics |
| **P1** | 36 | Migration role DDL, default privileges, init script content validation — important for Story 1.4 (Alembic) |
| **P2** | 2 | Idempotency — operational safety |

---

## Failing Tests Created (RED Phase)

### Integration Tests (172 test cases across 40 functions)

**File:** `eusolicit-app/tests/integration/test_db_schema_isolation.py`

#### P0 Tests — Schema Existence (AC1)

- **`TestAC1SchemasExist::test_schema_exists[client|pipeline|admin|notification|gateway|shared]`** (6 tests)
  - Status: RED — schemas do not exist (init script not created)
  - Verifies: Each of 6 schemas exists in `pg_namespace`

- **`TestAC1SchemasExist::test_schemas_match_test_utils_constant`** (1 test)
  - Status: RED — schemas not created
  - Verifies: DB schemas match `ALL_SCHEMAS` from `eusolicit_test_utils.db`

#### P0 Tests — Role Existence (AC2)

- **`TestAC2ServiceRolesExist::test_role_exists[×6]`** (6 tests)
  - Status: RED — roles do not exist
  - Verifies: Each role in `pg_roles`

- **`TestAC2ServiceRolesExist::test_role_has_login_privilege[×6]`** (6 tests)
  - Status: RED — roles not created
  - Verifies: `rolcanlogin = true` for each role

- **`TestAC2ServiceRolesExist::test_service_role_can_connect[×5]`** (5 tests)
  - Status: RED — roles not created, connection refused
  - Verifies: Each service role can establish asyncpg connection

#### P0 Tests — CRUD on Own Schema (AC3)

- **`TestAC3ServiceRoleCrudOwnSchema::test_service_role_can_insert_own_schema[×5]`** (5 tests)
  - Status: RED — roles/schemas not created
  - Verifies: INSERT permission on own schema

- **`TestAC3ServiceRoleCrudOwnSchema::test_service_role_can_select_own_schema[×5]`** (5 tests)
  - Status: RED
  - Verifies: SELECT permission on own schema

- **`TestAC3ServiceRoleCrudOwnSchema::test_service_role_can_update_own_schema[×5]`** (5 tests)
  - Status: RED
  - Verifies: UPDATE permission on own schema

- **`TestAC3ServiceRoleCrudOwnSchema::test_service_role_can_delete_own_schema[×5]`** (5 tests)
  - Status: RED
  - Verifies: DELETE permission on own schema

#### P0 Tests — SELECT on Shared Schema (AC3)

- **`TestAC3SharedSchemaSelectAccess::test_service_role_can_select_shared_tenants[×5]`** (5 tests)
  - Status: RED — shared.tenants doesn't exist
  - Verifies: SELECT access to shared.tenants (Task 2.2)

- **`TestAC3SharedSchemaSelectAccess::test_service_role_cannot_insert_shared[×5]`** (5 tests)
  - Status: RED
  - Verifies: INSERT to shared raises `InsufficientPrivilegeError`

- **`TestAC3SharedSchemaSelectAccess::test_service_role_cannot_update_shared[×5]`** (5 tests)
  - Status: RED
  - Verifies: UPDATE to shared raises `InsufficientPrivilegeError`

- **`TestAC3SharedSchemaSelectAccess::test_service_role_cannot_delete_shared[×5]`** (5 tests)
  - Status: RED
  - Verifies: DELETE from shared raises `InsufficientPrivilegeError`

#### P0 Tests — shared.tenants Structure (AC5)

- **`TestAC5SharedTenantsTable::test_shared_tenants_table_exists`** — Table in `information_schema`
- **`TestAC5SharedTenantsTable::test_id_column_uuid_primary_key`** — UUID type + PK constraint
- **`TestAC5SharedTenantsTable::test_id_default_gen_random_uuid`** — `gen_random_uuid()` default
- **`TestAC5SharedTenantsTable::test_name_column_varchar_not_null`** — VARCHAR(255) NOT NULL
- **`TestAC5SharedTenantsTable::test_slug_column_varchar_unique_not_null`** — VARCHAR(255) NOT NULL UNIQUE
- **`TestAC5SharedTenantsTable::test_created_at_column_timestamptz_not_null`** — TIMESTAMPTZ NOT NULL DEFAULT now()
- **`TestAC5SharedTenantsTable::test_updated_at_column_timestamptz_not_null`** — TIMESTAMPTZ NOT NULL DEFAULT now()
- **`TestAC5SharedTenantsTable::test_shared_tenants_has_exactly_five_columns`** — Exact column list match

All 8 tests: Status RED — table doesn't exist

#### P0 Tests — Cross-Schema Write Denied (AC6)

- **`TestAC6NegativeAccessCrossSchemaWriteDenied::test_client_api_role_cannot_write_admin_schema`** (1 test)
  - Status: RED — explicit test per AC (Task 2.4)
  - Verifies: `asyncpg.InsufficientPrivilegeError` on INSERT to admin schema

- **`TestAC6NegativeAccessCrossSchemaWriteDenied::test_client_api_role_cannot_write_pipeline_schema`** (1 test)
  - Status: RED — explicit test per AC (Task 2.5)
  - Verifies: `asyncpg.InsufficientPrivilegeError` on INSERT to pipeline schema

- **`TestAC6NegativeAccessCrossSchemaWriteDenied::test_service_role_cannot_insert_other_schema[×20]`** (20 tests)
  - Status: RED — parametrized across all 5 roles × 4 non-owned schemas
  - Verifies: INSERT denied for every cross-schema combination (Task 2.6)

- **`TestAC6NegativeAccessCrossSchemaWriteDenied::test_service_role_cannot_update_other_schema[×20]`** (20 tests)
  - Status: RED
  - Verifies: UPDATE denied for every cross-schema combination

- **`TestAC6NegativeAccessCrossSchemaWriteDenied::test_service_role_cannot_delete_other_schema[×20]`** (20 tests)
  - Status: RED
  - Verifies: DELETE denied for every cross-schema combination

#### P1 Tests — Migration Role DDL (AC4)

- **`TestAC4MigrationRoleDDL::test_migration_role_can_create_table[×6]`** — CREATE TABLE in each schema (Task 2.7)
- **`TestAC4MigrationRoleDDL::test_migration_role_can_alter_table[×6]`** — ALTER TABLE ADD COLUMN
- **`TestAC4MigrationRoleDDL::test_migration_role_can_drop_table[×6]`** — DROP TABLE
- **`TestAC4MigrationRoleDDL::test_migration_role_crud_any_schema[×6]`** — Full INSERT/SELECT/UPDATE/DELETE

All 24 tests: Status RED

#### P1 Tests — Default Privileges Inheritance

- **`TestDefaultPrivilegesForMigrationRole::test_migration_created_table_accessible_by_service_role[×5]`** (5 tests)
  - Status: RED
  - Verifies: Tables created by migration_role inherit service role grants (critical for Story 1.4 Alembic)

#### P1 Tests — Init Script Content (AC7)

- **`TestAC7DockerInitScript::test_init_script_file_exists`** — File at expected path
- **`TestAC7DockerInitScript::test_init_script_has_numeric_prefix`** — `01-` prefix
- **`TestAC7DockerInitScript::test_init_script_contains_schema_creation`** — CREATE SCHEMA for all 6
- **`TestAC7DockerInitScript::test_init_script_contains_role_creation`** — All 6 roles mentioned
- **`TestAC7DockerInitScript::test_init_script_uses_if_not_exists`** — Idempotency pattern
- **`TestAC7DockerInitScript::test_init_script_contains_alter_default_privileges`** — Future table grants
- **`TestAC7DockerInitScript::test_init_script_contains_shared_tenants_creation`** — Table DDL present

All 7 tests: Status RED

#### P2 Tests — Idempotency

- **`TestIdempotency::test_schemas_survive_reapplication`** — CREATE SCHEMA IF NOT EXISTS twice
- **`TestIdempotency::test_shared_tenants_survives_reapplication`** — CREATE TABLE IF NOT EXISTS twice

Both tests: Status RED

---

## Data Factories Created

N/A — This story tests database infrastructure (schemas, roles, permissions). No application-level data factories are needed. Test tables are created ad-hoc by `migration_role` during test setup and cleaned up in `finally` blocks.

---

## Fixtures Created

N/A — Tests use helper functions (`_connect_as_service_role`, `_connect_as_migration`) defined in the test file. These provide direct `asyncpg` connections as specific database roles, which is necessary to test per-role permission boundaries. The existing root `conftest.py` fixtures (SQLAlchemy-based, using `migration_role`) are not suitable for per-role connection testing.

---

## Mock Requirements

N/A — All tests run against a real PostgreSQL instance. No mocks required — the purpose is to verify actual database-level permission enforcement.

---

## Required data-testid Attributes

N/A — No UI components in this story.

---

## Implementation Checklist

### Task 1: Create Init SQL Script

**File to create:** `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql`

**Tests that will pass when complete:**

- [ ] `TestAC1SchemasExist` (7 tests) — Create 6 schemas with `CREATE SCHEMA IF NOT EXISTS`
- [ ] `TestAC2ServiceRolesExist` (17 tests) — Create 6 roles with `CREATE ROLE ... LOGIN PASSWORD`
- [ ] `TestAC3ServiceRoleCrudOwnSchema` (20 tests) — GRANT USAGE + CRUD on own schema
- [ ] `TestAC3SharedSchemaSelectAccess` (20 tests) — GRANT USAGE + SELECT on shared; NO write grants
- [ ] `TestAC4MigrationRoleDDL` (24 tests) — GRANT ALL on all schemas to migration_role
- [ ] `TestAC5SharedTenantsTable` (8 tests) — CREATE TABLE shared.tenants with exact DDL
- [ ] `TestAC6NegativeAccessCrossSchemaWriteDenied` (62 tests) — Ensure NO cross-schema write grants
- [ ] `TestDefaultPrivilegesForMigrationRole` (5 tests) — `ALTER DEFAULT PRIVILEGES FOR ROLE migration_role`
- [ ] `TestAC7DockerInitScript` (7 tests) — File exists with correct content
- [ ] `TestIdempotency` (2 tests) — Use IF NOT EXISTS throughout

**Estimated Effort:** 3–4 hours

### Task 2: Remove Skip Markers and Verify

- [ ] Remove `@pytest.mark.skip` from all test classes in `test_db_schema_isolation.py`
- [ ] Start PostgreSQL with init script: `make reset-db && make up`
- [ ] Run: `cd eusolicit-app && pytest tests/integration/test_db_schema_isolation.py -v`
- [ ] All 172 test cases pass (GREEN phase)

**Estimated Effort:** 30 minutes

---

## Running Tests

```bash
# Run all failing tests for this story (all will be skipped in RED phase)
cd eusolicit-app && pytest tests/integration/test_db_schema_isolation.py -v

# Run with skip report to see all skipped tests
cd eusolicit-app && pytest tests/integration/test_db_schema_isolation.py -v -rs

# Run only P0 tests (after removing skip markers)
cd eusolicit-app && pytest tests/integration/test_db_schema_isolation.py -v -k "AC1 or AC2 or AC3 or AC5 or AC6"

# Run only negative access tests
cd eusolicit-app && pytest tests/integration/test_db_schema_isolation.py -v -k "NegativeAccess"

# Run with verbose output showing parametrized test IDs
cd eusolicit-app && pytest tests/integration/test_db_schema_isolation.py -v --tb=long

# Run integration tests with Docker Compose (full environment)
make reset-db && make up && cd eusolicit-app && pytest tests/integration/test_db_schema_isolation.py -v
```

---

## Red-Green-Refactor Workflow

### RED Phase (Complete)

**TEA Agent Responsibilities:**

- All 172 test cases written and skipped (RED phase)
- Tests organized by acceptance criteria with clear priority tags
- Cross-schema negative access tests provide comprehensive security coverage
- Default privilege tests validate Alembic compatibility (Story 1.4 dependency)
- Test helpers use `asyncpg` for direct per-role connection testing

**Verification:**

- All tests are skipped with clear reason: "ATDD RED: Init script not implemented — Story 1.3"
- Tests assert expected behavior based on story ACs and architecture specs
- Tests will fail due to missing init script, not test bugs

---

### GREEN Phase (DEV Team — Next Steps)

**DEV Agent Responsibilities:**

1. Create `eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql` following the patterns in Dev Notes
2. Remove `@pytest.mark.skip(reason=_SKIP_REASON)` from all test classes
3. Run `make reset-db && make up` to apply the init script
4. Run `pytest tests/integration/test_db_schema_isolation.py -v`
5. Fix any failures (implementation bugs, not test bugs)
6. All 172 tests pass — GREEN

**Key Implementation Notes:**

- Two levels of GRANT required: existing tables + `ALTER DEFAULT PRIVILEGES` for future tables
- `ALTER DEFAULT PRIVILEGES FOR ROLE migration_role` is CRITICAL for Story 1.4 (Alembic)
- Idempotency: Use `IF NOT EXISTS`, `DO $$ ... $$` blocks
- Passwords MUST match `.env.example` exactly
- `shared` schema is SELECT-only for service roles (write access deferred to later stories)

---

### REFACTOR Phase (DEV Team — After All Tests Pass)

1. Verify all 172 tests pass consistently
2. Review SQL script for readability and DRY patterns
3. Ensure comments explain the GRANT strategy (existing + future tables)
4. Run `make reset-db && make up` twice to verify idempotency manually
5. Ready for code review

---

## Next Steps

1. **Review this checklist** — Verify test coverage aligns with team expectations
2. **Implement Story 1.3** — Create init SQL script using the implementation checklist as guide
3. **Remove skip markers** — When init script is ready, remove `@pytest.mark.skip` decorators
4. **Run tests** — `pytest tests/integration/test_db_schema_isolation.py -v`
5. **Verify GREEN** — All 172 tests pass
6. **Story 1.4 depends on this** — Alembic migrations require migration_role DDL + default privileges

---

## Knowledge Base References Applied

- **data-factories.md** — Factory patterns for test data (adapted: test tables created by migration_role as setup)
- **test-quality.md** — Deterministic, isolated, self-cleaning test design; no hard waits
- **test-healing-patterns.md** — Error diagnostic patterns (asyncpg.InsufficientPrivilegeError for negative tests)
- **test-levels-framework.md** — Integration level selected: tests database operations requiring live PostgreSQL
- **test-priorities-matrix.md** — P0 for security-critical schema isolation (E01-R-001 Score: 6)
- **ci-burn-in.md** — CI integration patterns for integration test execution

See `tea-index.csv` for complete knowledge fragment mapping.

---

## Notes

- **Test database:** Tests connect to `eusolicit_test` database (matching `TEST_DATABASE_URL` in conftest.py). The init script runs against the main `eusolicit` database via Docker; for tests, the same roles/schemas must exist in the test database.
- **Test isolation:** Each test creates temporary tables (`_atdd_*` prefix) via migration_role, runs assertions, then drops them in `finally` blocks. No test data persists between tests.
- **asyncpg vs SQLAlchemy:** Tests use raw `asyncpg` connections (not SQLAlchemy) because we need to connect as specific database roles. The root conftest's SQLAlchemy fixtures use migration_role only.
- **Parametrized coverage:** The 20 cross-schema INSERT denial tests cover all 5 roles × 4 non-owned service schemas. UPDATE and DELETE denial tests mirror this, totaling 60 parametrized negative access tests.
- **SERVICE_SCHEMA_MAP:** Imported from `eusolicit_test_utils.db` and cross-referenced to ensure test role-schema mappings match the architecture.

---

**Generated by BMad TEA Agent** — 2026-04-06
