---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
lastStep: step-04-generate-tests
lastSaved: '2026-04-09'
workflowType: bmad-testarch-atdd
story_id: 11-1-espd-profile-compliance-framework-db-schema-migrations
inputDocuments:
  - eusolicit-docs/implementation-artifacts/11-1-espd-profile-compliance-framework-db-schema-migrations.md
  - eusolicit-docs/test-artifacts/test-design-epic-11.md
  - eusolicit-app/services/client-api/tests/integration/test_002_migration.py
  - eusolicit-app/services/client-api/tests/conftest.py
  - eusolicit-app/services/client-api/src/client_api/models/company.py
  - eusolicit-app/services/client-api/src/client_api/models/espd_profile.py
  - eusolicit-app/services/admin-api/alembic/env.py
  - eusolicit-app/services/admin-api/alembic/versions/001_initial.py
  - eusolicit-app/services/admin-api/tests/conftest.py
  - _bmad/bmm/config.yaml
outputFiles:
  - eusolicit-app/services/client-api/tests/integration/test_009_migration.py
  - eusolicit-app/services/admin-api/tests/integration/test_002_migration.py
tddPhase: RED
detectedStack: backend
generationMode: ai-generation
---

# ATDD Checklist — Story 11.1: ESPD Profile & Compliance Framework DB Schema + Migrations

**Date:** 2026-04-09
**Author:** TEA Master Test Architect
**Story:** 11-1 — ESPD Profile & Compliance Framework DB Schema + Migrations
**Status:** 🔴 RED PHASE — All tests written; all will FAIL until implementation is complete
**Stack:** backend (Python / SQLAlchemy / Alembic / PostgreSQL)
**Generation Mode:** AI Generation (no recording needed — backend-only story)

---

## Step 1: Preflight & Context

### Stack Detection

- **Detected stack:** `backend`
- **Indicators:** `pyproject.toml` with SQLAlchemy/Alembic deps; `conftest.py` with async SQLAlchemy engines; no `playwright.config.ts` or frontend framework indicators in service roots

### Prerequisites Verified

- [x] Story approved with clear acceptance criteria (AC1–AC8 fully specified)
- [x] Backend test framework configured: `conftest.py` with `pytest-asyncio`, async SQLAlchemy engines present in both `services/client-api/tests/` and `services/admin-api/tests/`
- [x] Existing migration test pattern available: `services/client-api/tests/integration/test_002_migration.py`

### Context Loaded

- **Story:** 11-1 ESPD Profile & Compliance Framework DB Schema + Migrations
- **Epic test design:** `test-design-epic-11.md` — coverage strategy, P0-P3 test IDs
- **Existing pattern:** `test_002_migration.py` — alembic CLI subprocess pattern, async SQLAlchemy fixtures, RED phase `pytest.fail()` on missing migrations
- **ORM models reviewed:** `company.py` (companies.name NOT NULL), `espd_profile.py` (current schema: field_values + version)

---

## Step 2: Generation Mode

**Mode selected:** AI Generation

**Reason:** `{detected_stack}` = `backend`. No browser automation needed. All scenarios are database schema/migration tests amenable to direct SQL queries via SQLAlchemy + `information_schema` + `pg_*` system catalog queries.

---

## Step 3: Test Strategy

### AC → Test ID Mapping

| AC | Description | Test ID(s) | Level | Priority | File |
|----|-------------|------------|-------|----------|------|
| AC1 | Migration 009 runs cleanly from 008 baseline | E11-DB-001 | Integration | P0 | client-api |
| AC1 | espd_profiles has exactly 6 new columns, no old columns | E11-DB-002 | Integration | P0 | client-api |
| AC1 | uq_espd_profiles_company_version constraint removed | E11-DB-003 | Integration | P0 | client-api |
| AC1 | profile_name NOT NULL + DEFAULT 'Default Profile' | E11-DB-004 | Integration | P0 | client-api |
| AC1 | espd_data NOT NULL JSONB + DEFAULT '{}' | E11-DB-005 | Integration | P0 | client-api |
| AC1 | Insert with only company_id triggers server defaults | E11-DB-006 | Integration | P1 | client-api |
| AC2 | Data migration: field_values → espd_data['part_iii'], profile_name = 'Migrated Profile' | E11-DB-007 | Integration | P0 | client-api |
| AC4 | ix_espd_profiles_company_id index retained after migration | E11-DB-008 | Integration | P0 | client-api |
| AC1 | Downgrade 008 restores field_values, version, constraint | E11-DB-009 | Integration | P0 | client-api |
| AC3 | admin-api migration 002 runs cleanly from 001 baseline | E11-DB-010 | Integration | P0 | admin-api |
| AC3 | All 3 admin tables exist in admin schema | E11-DB-011 | Integration | P0 | admin-api |
| AC3 | regulation_type enum = {national, eu, programme} | E11-DB-012 | Integration | P0 | admin-api |
| AC3 | compliance_frameworks correct columns + NOT NULL + defaults | E11-DB-013 | Integration | P0 | admin-api |
| AC3 | platform_settings unique key constraint enforced | E11-DB-014 | Integration | P0 | admin-api |
| AC3 | opportunity_compliance_frameworks composite PK enforced | E11-DB-015 | Integration | P0 | admin-api |
| AC3 | FK cascade: delete framework cascades to assignments | E11-DB-016 | Integration | P0 | admin-api |
| AC4 | All 6 required indexes exist in pg_indexes | E11-DB-017 | Integration | P0 | admin-api |
| AC3 | Downgrade 001 drops all 3 tables + regulation_type enum | E11-DB-018 | Integration | P0 | admin-api |

### Test Level Selection

- **All tests:** Integration level — direct database schema verification via SQLAlchemy async engine + PostgreSQL system catalogs (`information_schema`, `pg_constraint`, `pg_indexes`, `pg_enum`, `pg_type`)
- **No E2E tests:** Backend-only story; no frontend/API endpoints involved
- **No unit tests:** Migration correctness is a property of the DB state, not of Python logic

### Edge Cases Covered

| Edge Case | Covered By |
|-----------|-----------|
| Empty field_values row data migration (field_values = '{}' → espd_data = '{}', not wrapped) | E11-DB-007b |
| Different opportunity_ids can share same framework_id | E11-DB-015b |
| Deleting framework A doesn't affect framework B's assignments | E11-DB-016b |
| ix_platform_settings_key is specifically a UNIQUE index (not just an index) | E11-DB-017b |
| admin._migrations_meta (from migration 001) is preserved after downgrade 001 | E11-DB-018 |
| Idempotency: second `alembic upgrade head` is a no-op | E11-DB-001, E11-DB-010 |
| alembic check (ORM ↔ DB sync) | AC8 via test_alembic_check_shows_no_pending_changes |

---

## Step 4: Test Files Generated

### TDD RED Phase Status

🔴 **All tests are in RED phase** — they will FAIL until the following is implemented:

**For `test_009_migration.py` to pass:**
- [ ] `services/client-api/alembic/versions/009_espd_profile_v2.py` created with `down_revision = "008"`
- [ ] Migration drops `uq_espd_profiles_company_version` constraint
- [ ] Migration adds `profile_name VARCHAR(255) NOT NULL DEFAULT 'Default Profile'`
- [ ] Migration adds `espd_data JSONB NOT NULL DEFAULT '{}'`
- [ ] Data migration: `espd_data = jsonb_build_object('part_iii', field_values)` for non-empty rows
- [ ] Migration drops `field_values` and `version` columns
- [ ] Downgrade reverses all steps exactly
- [ ] `services/client-api/src/client_api/models/espd_profile.py` updated (removes `field_values`/`version`, adds `profile_name`/`espd_data`)

**For `test_002_migration.py` (admin-api) to pass:**
- [ ] `services/admin-api/alembic/versions/002_compliance_framework_schema.py` created with `down_revision = "001"`
- [ ] Migration creates `regulation_type` PostgreSQL enum in `admin` schema
- [ ] Migration creates `admin.compliance_frameworks` with all required columns + indexes
- [ ] Migration creates `admin.platform_settings` with unique key constraint
- [ ] Migration creates `admin.opportunity_compliance_frameworks` with composite PK + FK cascade
- [ ] Downgrade drops all 3 tables + regulation_type enum
- [ ] `services/admin-api/src/admin_api/models/` directory created with all ORM model files
- [ ] `services/admin-api/alembic/env.py` updated: `target_metadata = Base.metadata`

---

## Test File Registry

### File 1: `services/client-api/tests/integration/test_009_migration.py`

**Coverage:** E11-DB-001 through E11-DB-009 + AC8 (alembic check)
**Test classes:** 8
**Total test functions:** 15

| Class | Test Function | Test ID | Priority |
|-------|--------------|---------|----------|
| `TestE11DB001MigrationLifecycle` | `test_migration_009_file_exists_in_versions_dir` | — | P1 |
| `TestE11DB001MigrationLifecycle` | `test_upgrade_head_from_008_baseline_lands_at_009` | E11-DB-001 | P0 |
| `TestE11DB001MigrationLifecycle` | `test_upgrade_head_is_idempotent` | E11-DB-001 | P1 |
| `TestE11DB007DataMigration` | `test_field_values_migrated_to_espd_data_part_iii` | E11-DB-007 | P0 |
| `TestE11DB007DataMigration` | `test_empty_field_values_row_gets_empty_espd_data` | E11-DB-007 | P1 |
| `TestE11DB009Downgrade` | `test_downgrade_008_restores_prior_schema` | E11-DB-009 | P0 |
| `TestE11DB002TableColumnStructure` | `test_espd_profiles_has_exactly_required_columns` | E11-DB-002 | P0 |
| `TestE11DB002TableColumnStructure` | `test_espd_data_column_is_jsonb_type` | E11-DB-002 | P0 |
| `TestE11DB002TableColumnStructure` | `test_profile_name_is_varchar_255` | E11-DB-002 | P1 |
| `TestE11DB002TableColumnStructure` | `test_espd_profiles_table_still_exists` | E11-DB-002 | P0 |
| `TestE11DB003UniqueConstraintRemoval` | `test_uq_espd_profiles_company_version_removed` | E11-DB-003 | P0 |
| `TestE11DB004And005ColumnConstraints` | `test_profile_name_is_not_null_with_default_profile_name` | E11-DB-004 | P0 |
| `TestE11DB004And005ColumnConstraints` | `test_espd_data_is_not_null_with_empty_jsonb_default` | E11-DB-005 | P0 |
| `TestE11DB006ServerDefaultBehavior` | `test_insert_with_only_company_id_applies_server_defaults` | E11-DB-006 | P1 |
| `TestE11DB008IndexRetention` | `test_ix_espd_profiles_company_id_index_retained` | E11-DB-008 | P0 |
| `TestE11DB008IndexRetention` | `test_ix_espd_profiles_company_id_covers_correct_column` | E11-DB-008 | P1 |
| `TestE11DB008IndexRetention` | `test_alembic_check_shows_no_pending_changes` | AC8 | P1 |

---

### File 2: `services/admin-api/tests/integration/test_002_migration.py`

**Coverage:** E11-DB-010 through E11-DB-018 + AC5 (model files) + AC8 (alembic check)
**Test classes:** 10
**Total test functions:** 22

| Class | Test Function | Test ID | Priority |
|-------|--------------|---------|----------|
| `TestE11DB010MigrationLifecycle` | `test_migration_002_file_exists_in_versions_dir` | — | P1 |
| `TestE11DB010MigrationLifecycle` | `test_upgrade_head_from_001_baseline_lands_at_002` | E11-DB-010 | P0 |
| `TestE11DB010MigrationLifecycle` | `test_upgrade_head_is_idempotent` | E11-DB-010 | P1 |
| `TestE11DB010MigrationLifecycle` | `test_orm_models_directory_created` | AC5 | P1 |
| `TestE11DB018Downgrade` | `test_downgrade_001_drops_all_tables_and_enum` | E11-DB-018 | P0 |
| `TestE11DB011AdminTablesExist` | `test_admin_table_exists_in_admin_schema[compliance_frameworks]` | E11-DB-011 | P0 |
| `TestE11DB011AdminTablesExist` | `test_admin_table_exists_in_admin_schema[platform_settings]` | E11-DB-011 | P0 |
| `TestE11DB011AdminTablesExist` | `test_admin_table_exists_in_admin_schema[opportunity_compliance_frameworks]` | E11-DB-011 | P0 |
| `TestE11DB012RegulationTypeEnum` | `test_regulation_type_enum_has_exactly_3_values` | E11-DB-012 | P0 |
| `TestE11DB012RegulationTypeEnum` | `test_regulation_type_enum_values_exact_set` | E11-DB-012 | P0 |
| `TestE11DB013ComplianceFrameworkColumns` | `test_compliance_frameworks_has_all_required_columns` | E11-DB-013 | P0 |
| `TestE11DB013ComplianceFrameworkColumns` | `test_compliance_frameworks_not_null_columns` | E11-DB-013 | P0 |
| `TestE11DB013ComplianceFrameworkColumns` | `test_compliance_frameworks_nullable_columns` | E11-DB-013 | P1 |
| `TestE11DB013ComplianceFrameworkColumns` | `test_compliance_frameworks_rules_is_jsonb_with_default` | E11-DB-013 | P0 |
| `TestE11DB013ComplianceFrameworkColumns` | `test_compliance_frameworks_is_active_default_true` | E11-DB-013 | P1 |
| `TestE11DB014PlatformSettingsUniqueKey` | `test_platform_settings_key_unique_constraint_exists` | E11-DB-014 | P0 |
| `TestE11DB014PlatformSettingsUniqueKey` | `test_duplicate_platform_settings_key_raises_integrity_error` | E11-DB-014 | P0 |
| `TestE11DB015CompositePrimaryKey` | `test_composite_pk_enforced_on_duplicate_pair` | E11-DB-015 | P0 |
| `TestE11DB015CompositePrimaryKey` | `test_different_opportunity_same_framework_allowed` | E11-DB-015 | P1 |
| `TestE11DB016FKCascade` | `test_deleting_framework_cascades_to_assignments` | E11-DB-016 | P0 |
| `TestE11DB016FKCascade` | `test_deleting_one_framework_does_not_affect_other_assignments` | E11-DB-016 | P1 |
| `TestE11DB017RequiredIndexes` | `test_required_index_exists[ix_espd_profiles_company_id]` | E11-DB-017 | P0 |
| `TestE11DB017RequiredIndexes` | `test_required_index_exists[ix_compliance_frameworks_country]` | E11-DB-017 | P0 |
| `TestE11DB017RequiredIndexes` | `test_required_index_exists[ix_compliance_frameworks_regulation_type]` | E11-DB-017 | P0 |
| `TestE11DB017RequiredIndexes` | `test_required_index_exists[ix_compliance_frameworks_is_active]` | E11-DB-017 | P0 |
| `TestE11DB017RequiredIndexes` | `test_required_index_exists[ix_platform_settings_key]` | E11-DB-017 | P0 |
| `TestE11DB017RequiredIndexes` | `test_required_index_exists[ix_opportunity_cf_framework_id]` | E11-DB-017 | P0 |
| `TestE11DB017RequiredIndexes` | `test_ix_platform_settings_key_is_unique_index` | E11-DB-017 | P1 |
| `TestE11DB017RequiredIndexes` | `test_alembic_check_shows_no_pending_changes` | AC8 | P1 |

---

## Summary Statistics

| Metric | Value |
|--------|-------|
| Total test files | 2 |
| Total test classes | 18 |
| Total test functions | 39 |
| P0 tests | 26 |
| P1 tests | 13 |
| Self-contained classes (no fixture) | 5 |
| Module-fixture classes | 13 |
| Coverage per AC | AC1: 9 tests, AC2: 2 tests, AC3: 16 tests, AC4: 8 tests, AC5: 1 test, AC8: 2 tests |
| Test IDs covered | E11-DB-001 through E11-DB-018 (all 18) |

---

## TDD Red Phase Requirements

All tests are written to **fail before implementation**. Here is how each major failure mechanism works:

| Failure Type | Tests Affected | Failure Mechanism |
|---|---|---|
| Migration file missing (`009_*.py`) | E11-DB-001 | `migrated_db` fixture calls `pytest.fail()` when revision not at 009 |
| Migration file missing (`002_*.py`) | E11-DB-010 | `migrated_db` fixture calls `pytest.fail()` when revision not at 002 |
| Column not present | E11-DB-002, E11-DB-003, E11-DB-004, E11-DB-005 | `assert` on `information_schema.columns` count/values |
| Constraint not removed | E11-DB-003 | `assert count == 0` on `pg_constraint` |
| Data not migrated | E11-DB-007 | `assert "part_iii" in espd_data` + value equality |
| Table not created | E11-DB-011, E11-DB-013 | `assert count == 1` on `information_schema.tables`/`columns` |
| Enum not created | E11-DB-012 | `assert count == 3` on `pg_enum` |
| Unique constraint not enforced | E11-DB-014 | `pytest.raises(IntegrityError)` on duplicate insert |
| Composite PK not enforced | E11-DB-015 | `pytest.raises(IntegrityError)` on duplicate pair insert |
| FK cascade not configured | E11-DB-016 | `assert remaining == 0` after cascade delete |
| Index not created | E11-DB-017 | `assert count == 1` on `pg_indexes` |
| Downgrade incomplete | E11-DB-009, E11-DB-018 | Column/table/enum absence assertions post-downgrade |
| ORM out of sync | AC8 | `assert result.returncode == 0` on `alembic check` |

---

## Implementation Checklist for Developer

### client-api (story tasks 1–2, 7)

- [ ] **Task 1.1** — Create `services/client-api/alembic/versions/009_espd_profile_v2.py` with `down_revision = "008"`
- [ ] **Task 1.2** — `upgrade()` step A: drop `uq_espd_profiles_company_version` constraint
- [ ] **Task 1.3** — `upgrade()` step B: add `profile_name VARCHAR(255) NOT NULL DEFAULT 'Default Profile'`
- [ ] **Task 1.4** — `upgrade()` step C: add `espd_data JSONB NOT NULL DEFAULT '{}'`
- [ ] **Task 1.5** — `upgrade()` step D: data migration (`UPDATE SET espd_data = jsonb_build_object('part_iii', field_values), profile_name = 'Migrated Profile' WHERE field_values != '{}'`)
- [ ] **Task 1.6** — `upgrade()` step E: drop `field_values` column
- [ ] **Task 1.7** — `upgrade()` step F: drop `version` column
- [ ] **Task 1.8** — `downgrade()`: reverse all steps in exact reverse order
- [ ] **Task 2.1–2.4** — Update `services/client-api/src/client_api/models/espd_profile.py` with `profile_name` + `espd_data` mapped columns

### admin-api (story tasks 3–5, 8)

- [ ] **Task 3.1** — Create `services/admin-api/src/admin_api/models/__init__.py`
- [ ] **Task 3.2** — Create `services/admin-api/src/admin_api/models/base.py`
- [ ] **Task 3.3** — Create `services/admin-api/src/admin_api/models/enums.py` with `RegulationType`
- [ ] **Task 3.4** — Update `services/admin-api/alembic/env.py`: `target_metadata = Base.metadata`
- [ ] **Task 4.1** — Create `compliance_framework.py` ORM model
- [ ] **Task 4.2** — Create `platform_settings.py` ORM model
- [ ] **Task 4.3** — Create `opportunity_compliance_framework.py` ORM model
- [ ] **Task 5.1** — Create `services/admin-api/alembic/versions/002_compliance_framework_schema.py` with `down_revision = "001"`
- [ ] **Task 5.2** — `upgrade()`: create `regulation_type` PostgreSQL enum in `admin` schema
- [ ] **Task 5.3** — `upgrade()`: create `admin.compliance_frameworks` with all columns + 3 indexes
- [ ] **Task 5.4** — `upgrade()`: create `admin.platform_settings` with unique key constraint
- [ ] **Task 5.5** — `upgrade()`: create `admin.opportunity_compliance_frameworks` with composite PK + FK + index
- [ ] **Task 5.6** — `downgrade()`: drop tables in reverse order, then drop enum

### Run Tests to Confirm RED Phase

```bash
# Confirm all client-api migration tests FAIL (red phase)
cd eusolicit-app/services/client-api
pytest tests/integration/test_009_migration.py -v 2>&1 | grep -E "PASSED|FAILED|ERROR"

# Confirm all admin-api migration tests FAIL (red phase)
cd eusolicit-app/services/admin-api
pytest tests/integration/test_002_migration.py -v 2>&1 | grep -E "PASSED|FAILED|ERROR"
```

Expected output: all tests show `FAILED` or `ERROR` (no `PASSED` in RED phase).

---

## Epic Test Design Coverage Cross-Reference

| Epic Test ID | Story Dependency | Satisfied By |
|---|---|---|
| E11-P0-001 (ESPD RLS) | `client.espd_profiles` company_id FK | E11-DB-002 (column present) |
| E11-P0-006 (Compliance Framework admin-only) | `admin.compliance_frameworks` exists | E11-DB-011 |
| E11-P1-008 (ESPD CRUD smoke) | New profile_name + espd_data schema | E11-DB-002, E11-DB-004, E11-DB-005 |
| E11-P1-010 (Compliance Framework CRUD) | `admin.compliance_frameworks` + regulation_type enum | E11-DB-011, E11-DB-012, E11-DB-013 |
| E11-P1-011/P1-012 (hybrid assignment) | `admin.opportunity_compliance_frameworks` join table | E11-DB-015, E11-DB-016 |
| E11-R-005 (ESPD company-scoped RLS) | `company_id` FK integrity | E11-DB-002, E11-DB-006 |
