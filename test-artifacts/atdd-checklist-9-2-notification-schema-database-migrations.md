---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-19'
inputDocuments:
  - eusolicit-docs/implementation-artifacts/9-2-notification-schema-database-migrations.md
  - eusolicit-docs/test-artifacts/test-design-epic-09.md
  - eusolicit-app/services/notification/tests/conftest.py
  - eusolicit-app/services/client-api/tests/integration/test_020_migration.py
  - eusolicit-app/services/notification/alembic/versions/001_initial.py
  - eusolicit-docs/_bmad/bmm/config.yaml
---

# ATDD Checklist: Story 9.2 — Notification Schema Database Migrations

**Story:** 9-2-notification-schema-database-migrations
**Date Generated:** 2026-04-19
**TDD Phase:** 🔴 RED (All tests designed to FAIL — implementation pending)
**Stack:** Backend (Python / SQLAlchemy / Alembic / PostgreSQL)
**Test Level:** Integration + Static Filesystem

---

## Summary

| Metric | Value |
|--------|-------|
| Total Tests Generated | 44 (incl. parametrized) |
| Integration Tests | 37 |
| Static / Filesystem Tests | 7 |
| E2E Tests | 0 (backend-only story) |
| TDD Phase | RED |
| All Tests Will Fail | ✅ Yes (migration file + ORM models don't exist) |
| Output File | `services/notification/tests/integration/test_002_migration.py` |

---

## Generated Test File

```
services/notification/tests/integration/test_002_migration.py
```

Run with:
```bash
# From eusolicit-app/
make test-integration SVC=notification
# Or directly:
pytest -m integration services/notification/tests/integration/test_002_migration.py -v
```

---

## Acceptance Criteria Coverage

| AC | Description | Test IDs | Priority | Status |
|----|-------------|----------|----------|--------|
| AC1 | Migration 002 upgrade/downgrade/idempotency | E09-DB-001, E09-DB-002, E09-DB-003 | P3 | 🔴 RED |
| AC2 | `notification.alert_log` — 6 columns + ARRAY | E09-DB-004, E09-DB-012 | P1 | 🔴 RED |
| AC3 | `notification.email_log` — 6 columns | E09-DB-005 | P1 | 🔴 RED |
| AC4 | `notification.sync_log` — 10 columns | E09-DB-006 | P1 | 🔴 RED |
| AC5 | All 5 indexes in pg_indexes | E09-DB-011 | P1 | 🔴 RED |
| AC6 | 4 enum types schema-scoped to 'notification' | E09-DB-007, E09-DB-008, E09-DB-009, E09-DB-010 | P1 | 🔴 RED |
| AC7 | ORM models: AlertLog, EmailLog, SyncLog | AC7-ORM-001 through AC7-ORM-006 | P2 | 🔴 RED |
| AC8 | Integration tests at specified path | (this file itself) | P1 | 🔴 RED |

---

## Test Inventory

### TestE09DB001MigrationLifecycle (E09-DB-001 / AC1) — P3

| Test Method | Description | Fails Because |
|-------------|-------------|---------------|
| `test_migration_002_file_exists_with_correct_revision` | 002_notification_tables.py exists with revision="002", down_revision="001" | File doesn't exist |
| `test_upgrade_head_from_001_baseline_succeeds` | alembic upgrade head from 001 returns code 0 | Migration file missing |
| `test_upgrade_head_is_idempotent` | Second upgrade head is a no-op, stays at (head) | Migration file missing |

### TestE09DB004AlertLogTable (E09-DB-004 / AC2) — P1

| Test Method | Description | Fails Because |
|-------------|-------------|---------------|
| `test_alert_log_table_exists_in_notification_schema` | notification.alert_log in information_schema.tables | Table not created |
| `test_alert_log_has_all_6_required_columns` | All 6 columns present: id, user_id, opportunity_ids, digest_type, sent_at, sendgrid_message_id | Table not created |
| `test_alert_log_id_is_uuid_primary_key` | id is UUID NOT NULL | Table not created |
| `test_alert_log_user_id_is_uuid_not_null` | user_id is UUID NOT NULL (soft reference, no FK) | Table not created |
| `test_alert_log_sendgrid_message_id_is_nullable` | sendgrid_message_id is TEXT nullable | Table not created |

### TestE09DB005EmailLogTable (E09-DB-005 / AC3) — P1

| Test Method | Description | Fails Because |
|-------------|-------------|---------------|
| `test_email_log_table_exists_in_notification_schema` | notification.email_log in information_schema.tables | Table not created |
| `test_email_log_has_all_6_required_columns` | All 6 columns per AC3 | Table not created |
| `test_email_log_recipient_email_is_text_not_null` | recipient_email is TEXT NOT NULL | Table not created |
| `test_email_log_status_has_server_default_sent` | status NOT NULL DEFAULT 'sent' | Table not created |
| `test_email_log_sendgrid_message_id_is_nullable` | sendgrid_message_id nullable | Table not created |

### TestE09DB006SyncLogTable (E09-DB-006 / AC4) — P1

| Test Method | Description | Fails Because |
|-------------|-------------|---------------|
| `test_sync_log_table_exists_in_notification_schema` | notification.sync_log in information_schema.tables | Table not created |
| `test_sync_log_has_all_10_required_columns` | All 10 columns per AC4 | Table not created |
| `test_sync_log_calendar_connection_id_is_uuid_not_null` | calendar_connection_id UUID NOT NULL | Table not created |
| `test_sync_log_events_counters_have_default_zero` | events_created/updated/deleted INTEGER NOT NULL DEFAULT 0 | Table not created |
| `test_sync_log_completed_at_and_error_message_are_nullable` | completed_at and error_message nullable | Table not created |

### Enum Tests (E09-DB-007 to E09-DB-010 / AC6) — P1

| Test Class | Test Method | Expected Values | Fails Because |
|------------|-------------|-----------------|---------------|
| `TestE09DB007DigestTypeEnum` | `test_notification_digest_type_enum_values_correct` | immediate, daily, weekly | Enum not created |
| `TestE09DB008EmailStatusEnum` | `test_notification_email_status_enum_values_correct` | sent, delivered, bounced, failed | Enum not created |
| `TestE09DB009CalendarProviderEnum` | `test_notification_calendar_provider_enum_values_correct` | google, microsoft | Enum not created |
| `TestE09DB010SyncTypeEnum` | `test_notification_sync_type_enum_values_correct` | full, incremental | Enum not created |
| `TestE09DB006AllEnumsSchemaScoped` | `test_enum_exists_in_notification_schema_not_public[*]` (×4 parametrized) | Enum in 'notification', NOT in 'public' | Enums not created |

### TestE09DB011AllIndexesExist (E09-DB-011 / AC5) — P1

| Test Method | Indexes Covered | Fails Because |
|-------------|-----------------|---------------|
| `test_ac5_index_exists_in_pg_indexes[*]` (×5 parametrized) | ix_alert_log_user_id, ix_alert_log_sent_at, ix_email_log_sendgrid_message_id, ix_email_log_status, ix_sync_log_calendar_connection_id | Indexes not created |
| `test_all_5_ac5_indexes_exist_collectively` | All 5 collectively | Indexes not created |

### TestE09DB012OpportunityIdsArray (E09-DB-012 / AC2) — P1

| Test Method | Description | Fails Because |
|-------------|-------------|---------------|
| `test_opportunity_ids_is_array_data_type` | data_type = 'ARRAY', NOT NULL | Column not created |
| `test_opportunity_ids_element_type_is_uuid` | udt_name = '_uuid' (PostgreSQL ARRAY(uuid) encoding) | Column not created |
| `test_opportunity_ids_server_default_is_empty_uuid_array` | column_default contains '{}' | Column not created |

### TestE09DB002Downgrade (E09-DB-002 / AC1) — P3

| Test Method | Description | Fails Because |
|-------------|-------------|---------------|
| `test_downgrade_to_001_drops_all_migration_002_objects` | Downgrade removes all tables, enums, indexes | Migration file missing (skipped gracefully) |
| `test_migration_002_is_fully_reversible` | Full round-trip: upgrade → downgrade → upgrade | Migration file missing (skipped gracefully) |

### TestAC7ORMModels (AC7) — P2

| Test Method | Description | Fails Because |
|-------------|-------------|---------------|
| `test_models_directory_exists` | `src/notification/models/` directory created | Directory doesn't exist |
| `test_alert_log_model_file_exists_and_defines_alertlog` | alert_log.py with AlertLog class, correct schema/table | File doesn't exist |
| `test_email_log_model_file_exists_and_defines_emaillog` | email_log.py with EmailLog class | File doesn't exist |
| `test_sync_log_model_file_exists_and_defines_synclog` | sync_log.py with SyncLog class | File doesn't exist |
| `test_models_init_exports_all_three_models` | `__init__.py` exports AlertLog, EmailLog, SyncLog + `__all__` | File doesn't exist |
| `test_base_model_file_exists` | base.py with `class Base(DeclarativeBase)` | File doesn't exist |
| `test_alembic_check_no_error` | alembic check returns exit code 0 | Migration not applied |

---

## Fixture Design

### `migrated_db` (module-scoped)

- Uses `migration_role` credentials (DDL permissions)
- Attempts `alembic upgrade 002` at setup
- Calls `pytest.fail()` if migration file missing (hard RED phase failure)
- Tears down with `downgrade 001` → `upgrade head` to restore consistent state
- All DB assertion tests depend on this fixture

### `migration_engine` (module-scoped)

- Raw async engine for direct SQL inspection
- Connected as `migration_role` via `TEST_DATABASE_URL`

### Migration lifecycle tests (standalone)

- Invoke `_run_alembic()` helper directly
- No fixture dependency — manage their own Alembic state

---

## Implementation Checklist for Developer

After implementing the story, the developer should:

1. **Create** `services/notification/alembic/versions/002_notification_tables.py`
   - Exact code provided in story Dev Notes
   - `revision = "002"`, `down_revision = "001"`
   - Creates 4 enum types with `schema='notification'`
   - Creates alert_log, email_log, sync_log tables
   - Creates all 5 indexes

2. **Create** `services/notification/src/notification/models/` directory with:
   - `base.py` — `class Base(DeclarativeBase): pass`
   - `alert_log.py` — `AlertLog` ORM model
   - `email_log.py` — `EmailLog` ORM model
   - `sync_log.py` — `SyncLog` ORM model
   - `__init__.py` — exports all three + `__all__`

3. **Apply migration** to test DB:
   ```bash
   cd eusolicit-app && make migrate-service SVC=notification
   ```

4. **Remove `pytest.skip()` / verify RED→GREEN** by running:
   ```bash
   make test-integration SVC=notification
   ```

5. **Commit** once all 29 tests pass.

---

## TDD Red Phase Compliance

All tests are designed to fail because:

- **`test_migration_002_file_exists_*`** — direct file assertion on nonexistent path
- **`migrated_db` fixture** — calls `pytest.fail()` when `alembic upgrade 002` fails
- **DB assertion tests** — `information_schema` / `pg_catalog` queries return empty results
- **ORM static tests** — `Path.is_file()` / `Path.is_dir()` assertions on nonexistent paths

No test uses placeholder assertions (`assert True`). All assert the **exact expected behavior** from the acceptance criteria.

---

## Risk Coverage from Epic Test Design

| Risk ID | Risk | Coverage in This Story |
|---------|------|------------------------|
| R-001 | Redis stream event loss for immediate notifications (Score: 6) | `alert_log` table (E09-DB-004) provides durable audit trail for all dispatched alerts |
| R-002 | Stripe Usage over/under-reporting (Score: 6) | `email_log` table (E09-DB-005) provides audit trail for send attempts |

---

## Epic P0 Entry Criteria Satisfied (After GREEN)

From `test-design-epic-09.md`:

| P0 Test Scenario | Required Table | Satisfied By |
|------------------|---------------|--------------|
| Immediate Notification Match (E2E) | `notification.alert_log` | E09-DB-004 ✅ |
| Daily/Weekly Digest (API) | `notification.alert_log` | E09-DB-004 ✅ |
| SendGrid Webhook (P1) | `notification.email_log` | E09-DB-005 ✅ |
| Calendar Sync diff (P2) | `notification.sync_log` | E09-DB-006 ✅ |

All P0 story dependencies (S09.04, S09.05, S09.06, S09.08, S09.09) blocked on this migration are covered.

---

## Next Steps (TDD Green Phase)

After implementing the story:

1. Remove `pytest.skip()` guards (none used in this file — tests fail naturally)
2. Run: `make test-integration SVC=notification`
3. All 29 tests should **PASS** (green phase)
4. If any tests fail:
   - Compare test assertion against story Dev Notes spec
   - Fix implementation (not the test) unless spec was ambiguous
5. Run `alembic check` to confirm ORM-migration alignment
6. Commit implementation + confirm test suite clean

---

**Generated by:** BMad TEA Agent — ATDD Skill
**Workflow:** `bmad-testarch-atdd`
**Story Key:** 9-2-notification-schema-database-migrations
**Epic Test Design Source:** `eusolicit-docs/test-artifacts/test-design-epic-09.md`
