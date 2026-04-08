# Story 2.1: Database Schema — Auth & Identity Tables

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **developer on the EU Solicit platform**,
I want **Alembic migration `002_auth_identity_tables` to create all core auth and identity tables in the `client` schema (plus `shared.audit_log` in the `shared` schema), backed by SQLAlchemy ORM models in `app/models/`**,
so that **every subsequent Epic 2 story has a ready, version-controlled database foundation with correct column types, constraints, indexes, enums, and foreign keys — and the `shared.audit_log` table is available for all mutation-logging across the platform**.

## Acceptance Criteria

1. Alembic migration `002_auth_identity_tables.py` runs cleanly (`upgrade head`) on a fresh database with only `001_initial.py` applied, and `downgrade base` rolls back completely
2. All 7 tables exist in the correct schemas after migration:
   - `client.users` — id, email (unique), hashed_password, full_name, google_sub, is_active, email_verified, created_at, updated_at
   - `client.companies` — id, name, tax_id, address JSONB, industry, cpv_sectors JSONB, regions JSONB, description, certifications JSONB, created_at, updated_at
   - `client.company_memberships` — user_id (FK→users), company_id (FK→companies), role (enum), invited_at, accepted_at
   - `client.subscriptions` — id, company_id (FK→companies), plan, status, current_period_start, current_period_end (skeleton only)
   - `client.entity_permissions` — id, user_id (FK→users), company_id (FK→companies), entity_type, entity_id, permission (enum), granted_by (FK→users), created_at
   - `client.espd_profiles` — id, company_id (FK→companies), field_values JSONB, version (int), created_at, updated_at
   - `shared.audit_log` — id, timestamp, user_id, action_type, entity_type, entity_id, before JSONB, after JSONB, ip_address
3. Role enum (`company_role`) includes exactly: `admin`, `bid_manager`, `contributor`, `reviewer`, `read_only`
4. Permission enum (`entity_permission`) includes exactly: `read`, `write`, `manage`
5. Foreign key relationships are correct: `company_memberships.user_id → users.id`, `company_memberships.company_id → companies.id`; all FK columns use `ON DELETE CASCADE` or `ON DELETE RESTRICT` as specified in dev notes
6. Required indexes are present: `users.email` (unique), composite index on `entity_permissions(user_id, entity_type, entity_id)`, composite index on `audit_log(entity_type, entity_id)`
7. `shared.audit_log` lives in the `shared` schema (verified via `information_schema.tables`)
8. SQLAlchemy ORM models are created in `services/client-api/src/client_api/models/` and `env.py` updated to reference `Base.metadata` for autogenerate support
9. Integration tests pass: migration runs clean, all tables + constraints + indexes verified, enum values verified, schema isolation verified

## Tasks / Subtasks

- [x] Task 1: Create SQLAlchemy model base and enums (AC: 3, 4, 8)
  - [x] 1.1 Create `services/client-api/src/client_api/models/__init__.py` — exports `Base` and all model classes
  - [x] 1.2 Create `services/client-api/src/client_api/models/base.py` — `DeclarativeBase` subclass with `metadata` bound to the `client` schema default; import from `sqlalchemy.orm`
  - [x] 1.3 Create `services/client-api/src/client_api/models/enums.py` — `CompanyRole` (`admin`, `bid_manager`, `contributor`, `reviewer`, `read_only`) and `EntityPermission` (`read`, `write`, `manage`) as `str, Enum` Python enums
  - [x] 1.4 Create `services/client-api/src/client_api/models/user.py` — `User` ORM model (see dev notes for full column spec)
  - [x] 1.5 Create `services/client-api/src/client_api/models/company.py` — `Company` ORM model
  - [x] 1.6 Create `services/client-api/src/client_api/models/company_membership.py` — `CompanyMembership` ORM model with `CompanyRole` FK enum column
  - [x] 1.7 Create `services/client-api/src/client_api/models/subscription.py` — `Subscription` ORM model (skeleton — no billing logic)
  - [x] 1.8 Create `services/client-api/src/client_api/models/entity_permission.py` — `EntityPermission` ORM model
  - [x] 1.9 Create `services/client-api/src/client_api/models/espd_profile.py` — `ESPDProfile` ORM model
  - [x] 1.10 Create `services/client-api/src/client_api/models/audit_log.py` — `AuditLog` ORM model in `shared` schema (`__table_args__ = {"schema": "shared"}`)

- [x] Task 2: Create Alembic migration (AC: 1, 2, 3, 4, 5, 6, 7)
  - [x] 2.1 Create `services/client-api/alembic/versions/002_auth_identity_tables.py` with `down_revision = "001"`
  - [x] 2.2 Add `upgrade()` — create PostgreSQL enum types `company_role` and `entity_permission` in `client` schema using `postgresql.ENUM(..., name="company_role", schema="client")`
  - [x] 2.3 Create `client.users` table with all columns, unique constraint on `email`, and `updated_at` server default
  - [x] 2.4 Create `client.companies` table with JSONB columns for `address`, `cpv_sectors`, `regions`, `certifications`
  - [x] 2.5 Create `client.company_memberships` table with composite PK `(user_id, company_id)` and `role` using the `company_role` enum
  - [x] 2.6 Create `client.subscriptions` table (skeleton — minimal columns only, no Stripe logic)
  - [x] 2.7 Create `client.entity_permissions` table with `entity_permission` enum column and composite FK to users (user_id and granted_by)
  - [x] 2.8 Create `client.espd_profiles` table with JSONB `field_values` and integer `version`
  - [x] 2.9 Ensure `shared` schema exists (`op.execute("CREATE SCHEMA IF NOT EXISTS shared")`) before creating `shared.audit_log`
  - [x] 2.10 Create `shared.audit_log` table with all columns including JSONB `before` and `after`
  - [x] 2.11 Create all required indexes: unique on `users.email`; composite on `entity_permissions(user_id, entity_type, entity_id)`; composite on `audit_log(entity_type, entity_id)`
  - [x] 2.12 Add `downgrade()` — drop all tables and enum types in reverse creation order (audit_log, espd_profiles, entity_permissions, subscriptions, company_memberships, companies, users; then drop enums)

- [x] Task 3: Update Alembic `env.py` to use ORM metadata (AC: 8)
  - [x] 3.1 Import `Base` from `client_api.models` in `services/client-api/alembic/env.py`
  - [x] 3.2 Set `target_metadata = Base.metadata` (replaces `target_metadata = None`)
  - [x] 3.3 Verify `alembic check` runs without error (no pending autogenerate changes after migration is applied)

- [x] Task 4: Add missing dependencies to `client-api` (AC: 1, 8)
  - [x] 4.1 Add `asyncpg>=0.29` to `services/client-api/pyproject.toml` dependencies if not already present (needed for async SQLAlchemy sessions in later stories)
  - [x] 4.2 Add `eusolicit-test-utils` to `[project.optional-dependencies] dev` (needed for integration tests — per deferred-work.md)

- [x] Task 5: Write integration tests (AC: 1–9)
  - [x] 5.1 `services/client-api/tests/integration/test_002_migration.py` existed (pre-written ATDD tests — all 42 tests now pass)
  - [x] 5.2 Test: `alembic upgrade head` from `001` baseline succeeds ✓
  - [x] 5.3 Test: all 7 tables exist in correct schemas ✓
  - [x] 5.4 Test: `shared.audit_log` schema = `shared` (E02-P2-003) ✓
  - [x] 5.5 Test: `company_role` enum values match exactly `{admin, bid_manager, contributor, reviewer, read_only}` (E02-P2-002) ✓
  - [x] 5.6 Test: `entity_permission` enum values match exactly `{read, write, manage}` ✓
  - [x] 5.7 Test: unique index on `users.email` — insert duplicate email raises `IntegrityError` ✓
  - [x] 5.8 Test: composite index on `entity_permissions(user_id, entity_type, entity_id)` exists in `pg_indexes` ✓
  - [x] 5.9 Test: `alembic downgrade base` removes all tables and enum types cleanly ✓
  - [x] 5.10 Test: `alembic upgrade head` is idempotent when re-run ✓

### Review Follow-ups (AI)

- [x] [AI-Review][Patch] Fix missing GRANT INSERT on shared.audit_log for client_api_role — add `GRANT INSERT ON shared.audit_log TO client_api_role` before the REVOKE statement; add integration test verifying client_api_role CAN INSERT [002_auth_identity_tables.py]
- [x] [AI-Review][Patch] Remove duplicate unique enforcement on users.email — remove `op.create_index("ix_users_email", ...)` (UniqueConstraint already provides it); remove corresponding `op.drop_index` from downgrade [002_auth_identity_tables.py]
- [x] [AI-Review][Patch] Add `checkfirst=True` to enum `.create()` calls in upgrade() to prevent crash on partial failure re-run [002_auth_identity_tables.py]

## Dev Notes

### CRITICAL: Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/client-api/alembic/env.py` | Configured, schema-aware, `target_metadata = None` | **MODIFY** — set `target_metadata = Base.metadata` only |
| `services/client-api/alembic/versions/001_initial.py` | Creates `client._migrations_meta` | **DO NOT TOUCH** |
| `infra/postgres/init/01-init-schemas-and-roles.sql` | Creates 6 schemas + roles + grants | **DO NOT TOUCH** — `shared` schema already exists |
| `docker-compose.yml` | PostgreSQL 16 | **DO NOT TOUCH** |
| `.env.example` | All connection strings defined | **DO NOT TOUCH** |
| `packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py` | `ALL_SCHEMAS`, `SERVICE_SCHEMA_MAP`, `apply_migrations()` | **DO NOT TOUCH** — reuse |
| `tests/conftest.py` | DB fixtures using `migration_role` | **DO NOT TOUCH** |

### Files to CREATE (New)

| File | Purpose |
|------|---------|
| `services/client-api/src/client_api/models/__init__.py` | Package exports: Base + all model classes |
| `services/client-api/src/client_api/models/base.py` | `DeclarativeBase` with schema-aware metadata |
| `services/client-api/src/client_api/models/enums.py` | `CompanyRole`, `EntityPermission` Python enums |
| `services/client-api/src/client_api/models/user.py` | `User` ORM model |
| `services/client-api/src/client_api/models/company.py` | `Company` ORM model |
| `services/client-api/src/client_api/models/company_membership.py` | `CompanyMembership` ORM model |
| `services/client-api/src/client_api/models/subscription.py` | `Subscription` ORM model (skeleton) |
| `services/client-api/src/client_api/models/entity_permission.py` | `EntityPermission` ORM model |
| `services/client-api/src/client_api/models/espd_profile.py` | `ESPDProfile` ORM model |
| `services/client-api/src/client_api/models/audit_log.py` | `AuditLog` ORM model (schema="shared") |
| `services/client-api/alembic/versions/002_auth_identity_tables.py` | Migration creating all 7 tables |
| `services/client-api/tests/integration/test_002_migration.py` | Integration tests for this migration |

### Files to MODIFY (Existing)

| File | Change |
|------|--------|
| `services/client-api/alembic/env.py` | Set `target_metadata = Base.metadata` |
| `services/client-api/pyproject.toml` | Add `asyncpg>=0.29` to deps; add `eusolicit-test-utils` to dev extras |

### Column Specification

#### `client.users`

```python
id: UUID, primary_key=True, default=uuid4
email: String(255), unique=True, nullable=False
hashed_password: Text, nullable=True  # null for Google-only accounts
full_name: String(255), nullable=False
google_sub: String(255), nullable=True, unique=True
is_active: Boolean, nullable=False, server_default="true"
email_verified: Boolean, nullable=False, server_default="false"
created_at: DateTime(timezone=True), server_default=func.now()
updated_at: DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
```

Indexes: unique on `email`, unique on `google_sub` (sparse — only if not null, but standard unique index is acceptable for MVP)

#### `client.companies`

```python
id: UUID, primary_key=True, default=uuid4
name: String(255), nullable=False
tax_id: String(100), nullable=True
address: JSONB, nullable=True                 # {street, city, postal_code, country}
industry: String(100), nullable=True
cpv_sectors: JSONB, nullable=True             # list[str]
regions: JSONB, nullable=True                 # list[str]
description: Text, nullable=True
certifications: JSONB, nullable=True          # list[{name, issuer, valid_until}]
created_at: DateTime(timezone=True), server_default=func.now()
updated_at: DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
```

#### `client.company_memberships`

Composite PK `(user_id, company_id)` — a user can only have one role per company.

```python
user_id: UUID, ForeignKey("client.users.id", ondelete="CASCADE"), primary_key=True
company_id: UUID, ForeignKey("client.companies.id", ondelete="CASCADE"), primary_key=True
role: Enum(CompanyRole, name="company_role", schema="client"), nullable=False
invited_at: DateTime(timezone=True), server_default=func.now()
accepted_at: DateTime(timezone=True), nullable=True  # null = pending invite
```

#### `client.subscriptions` (skeleton)

```python
id: UUID, primary_key=True, default=uuid4
company_id: UUID, ForeignKey("client.companies.id", ondelete="CASCADE"), nullable=False
plan: String(50), nullable=True
status: String(50), nullable=True
current_period_start: DateTime(timezone=True), nullable=True
current_period_end: DateTime(timezone=True), nullable=True
```

No billing logic. Billing columns (stripe_subscription_id, etc.) are added in E08.

#### `client.entity_permissions`

```python
id: UUID, primary_key=True, default=uuid4
user_id: UUID, ForeignKey("client.users.id", ondelete="CASCADE"), nullable=False
company_id: UUID, ForeignKey("client.companies.id", ondelete="CASCADE"), nullable=False
entity_type: String(100), nullable=False      # e.g., "opportunity", "proposal"
entity_id: UUID, nullable=False               # ID of the specific entity
permission: Enum(EntityPermission, name="entity_permission", schema="client"), nullable=False
granted_by: UUID, ForeignKey("client.users.id", ondelete="SET NULL"), nullable=True
created_at: DateTime(timezone=True), server_default=func.now()
```

Composite index: `(user_id, entity_type, entity_id)` — supports the RBAC query in S02.10.

#### `client.espd_profiles`

```python
id: UUID, primary_key=True, default=uuid4
company_id: UUID, ForeignKey("client.companies.id", ondelete="CASCADE"), nullable=False
field_values: JSONB, nullable=False, server_default="{}"
version: Integer, nullable=False, server_default="1"
created_at: DateTime(timezone=True), server_default=func.now()
updated_at: DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
```

Each update inserts a new row (version as monotonic counter) rather than updating in place. The latest version is `MAX(version)` for a given `company_id`.

#### `shared.audit_log`

```python
__table_args__ = {"schema": "shared"}
id: UUID, primary_key=True, default=uuid4
timestamp: DateTime(timezone=True), server_default=func.now(), nullable=False
user_id: UUID, nullable=True                  # null for system-generated events
action_type: String(100), nullable=False      # e.g., "create", "update", "delete", "auth.login"
entity_type: String(100), nullable=True       # e.g., "company", "user", "espd_profile"
entity_id: UUID, nullable=True
before: JSONB, nullable=True                  # snapshot before mutation
after: JSONB, nullable=True                   # snapshot after mutation
ip_address: String(45), nullable=True         # IPv4 or IPv6
```

Composite index: `(entity_type, entity_id)` — supports audit history queries. No FK on `user_id` (cross-schema FK avoided; user ID validated at application layer). Append-only enforcement is at the DB level: `REVOKE UPDATE, DELETE ON shared.audit_log FROM client_api_role` (see migration task 2.10).

### Migration File Pattern

```python
"""Auth & identity tables — users, companies, memberships, subscriptions,
entity_permissions, espd_profiles, shared.audit_log.

Revision ID: 002
Revises: 001
Create Date: 2026-04-07
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = "002"
down_revision = "001"
branch_labels = None
depends_on = None

CLIENT_SCHEMA = "client"
SHARED_SCHEMA = "shared"


def upgrade() -> None:
    # 1. Ensure shared schema exists (already created by init SQL, but be safe)
    op.execute("CREATE SCHEMA IF NOT EXISTS shared")

    # 2. Create enum types
    company_role = postgresql.ENUM(
        "admin", "bid_manager", "contributor", "reviewer", "read_only",
        name="company_role", schema=CLIENT_SCHEMA,
    )
    company_role.create(op.get_bind())

    entity_permission = postgresql.ENUM(
        "read", "write", "manage",
        name="entity_permission", schema=CLIENT_SCHEMA,
    )
    entity_permission.create(op.get_bind())

    # 3. Create tables ...
    # (see individual table specs above)

    # 4. Create indexes
    op.create_index("ix_users_email", "users", ["email"], unique=True, schema=CLIENT_SCHEMA)
    op.create_index(
        "ix_entity_permissions_lookup",
        "entity_permissions",
        ["user_id", "entity_type", "entity_id"],
        schema=CLIENT_SCHEMA,
    )
    op.create_index(
        "ix_audit_log_entity",
        "audit_log",
        ["entity_type", "entity_id"],
        schema=SHARED_SCHEMA,
    )

    # 5. Append-only enforcement on audit_log (application role cannot UPDATE or DELETE)
    op.execute(f"REVOKE UPDATE, DELETE ON {SHARED_SCHEMA}.audit_log FROM client_api_role")


def downgrade() -> None:
    # Drop in reverse order
    op.drop_index("ix_audit_log_entity", table_name="audit_log", schema=SHARED_SCHEMA)
    op.drop_index("ix_entity_permissions_lookup", table_name="entity_permissions", schema=CLIENT_SCHEMA)
    op.drop_index("ix_users_email", table_name="users", schema=CLIENT_SCHEMA)

    op.drop_table("audit_log", schema=SHARED_SCHEMA)
    op.drop_table("espd_profiles", schema=CLIENT_SCHEMA)
    op.drop_table("entity_permissions", schema=CLIENT_SCHEMA)
    op.drop_table("subscriptions", schema=CLIENT_SCHEMA)
    op.drop_table("company_memberships", schema=CLIENT_SCHEMA)
    op.drop_table("companies", schema=CLIENT_SCHEMA)
    op.drop_table("users", schema=CLIENT_SCHEMA)

    # Drop enum types
    sa.Enum(name="entity_permission", schema=CLIENT_SCHEMA).drop(op.get_bind(), checkfirst=True)
    sa.Enum(name="company_role", schema=CLIENT_SCHEMA).drop(op.get_bind(), checkfirst=True)
```

### SQLAlchemy Base Pattern

```python
# models/base.py
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    """Shared declarative base for all client-api ORM models."""
    pass
```

The ORM models use `__table_args__ = {"schema": "client"}` for client-schema tables and `{"schema": "shared"}` for `AuditLog`. UUIDs use `sa.UUID(as_uuid=True)` with `default=uuid4` at the Python layer (not DB-level `gen_random_uuid()`) for predictability in tests.

### JSONB Handling

Use `postgresql.JSONB` (not `sa.JSON`) for all JSONB columns to get PostgreSQL-specific operators. Import: `from sqlalchemy.dialects.postgresql import JSONB`.

### Updated_at Trigger Note

Per the deferred-work note from Story 1.3: `updated_at` with `server_default=func.now()` only fires on INSERT. For ORM-managed updates, use `onupdate=func.now()` in the column definition. This is sufficient for MVP; a DB-level trigger is a production hardening item.

### Cross-Schema FK Considerations

`entity_permissions.user_id` and `audit_log.user_id` reference `client.users`. For `audit_log` (in `shared` schema), avoid a hard FK constraint since the `shared` schema is used by multiple services. Use application-level user_id validation only. Document this in the migration with a comment.

### Test File Pattern

```python
# tests/integration/test_002_migration.py
import subprocess
import pytest
from sqlalchemy import text
from eusolicit_test_utils.db import apply_migrations

# Tests use TEST_DATABASE_URL (migration_role against eusolicit_test)
# Each test applies migration, asserts, then tears down

@pytest.fixture(autouse=True)
def apply_and_teardown():
    apply_migrations("client-api")  # upgrade head
    yield
    # downgrade to 001 to leave DB clean
    subprocess.run(
        ["alembic", "downgrade", "001"],
        cwd="services/client-api",
        check=True,
        env={**os.environ, "DATABASE_URL": os.environ["TEST_DATABASE_URL"]},
    )
```

Key assertions (aligned with E02-P2-001, E02-P2-002, E02-P2-003, E02-P2-014):

- Query `information_schema.tables` to verify all 7 tables exist in correct schemas
- Query `pg_enum` to verify `company_role` values exactly match `{admin, bid_manager, contributor, reviewer, read_only}`
- Insert duplicate `users.email` → assert `IntegrityError`
- Query `pg_indexes` → composite indexes on `entity_permissions` and `audit_log` present
- Attempt `UPDATE shared.audit_log SET action_type='tampered'` as `client_api_role` → assert `PermissionError` / PG exception

### Test Expectations (from Epic-Level Test Design)

This story maps directly to the following test scenarios from `test-design-epic-02.md`:

| Priority | Test ID | Test Description | Risk Link | Verification |
|----------|---------|-----------------|-----------|--------------|
| **P2** | E02-P2-001 | Alembic migration creates all E02 tables with correct column types, constraints, and indexes | — | Query `information_schema` after `upgrade head`; assert all 7 tables with correct column types |
| **P2** | E02-P2-002 | Role enum includes exactly: admin, bid_manager, contributor, reviewer, read_only | — | Query `pg_enum WHERE enumtypid = 'client.company_role'::regtype` → exactly 5 values |
| **P2** | E02-P2-003 | `shared.audit_log` lives in the `shared` schema | — | `information_schema.tables` WHERE table_schema = 'shared' AND table_name = 'audit_log'` → 1 row |
| **P2** | E02-P2-014 | Audit log append-only — application code cannot UPDATE or DELETE rows | E02-R-007 | `REVOKE UPDATE, DELETE ON shared.audit_log FROM client_api_role` verified; confirm attempt raises PG permission denied |

**Quality gate**: All P2 tests for this story must pass. Since this is a schema-only story with no P0/P1 test assignments, quality gate = 100% P2 pass rate for the 4 tests listed above + migration idempotency verification.

### Cross-Story Dependencies

- **Story 1.3** (DONE): `01-init-schemas-and-roles.sql` — schemas (`client`, `shared`) exist; `migration_role` has DDL on all schemas; `client_api_role` has CRUD on `client`; `shared` schema exists and `client_api_role` has INSERT-only access after this migration adds the REVOKE.
- **Story 1.4** (DONE): Alembic scaffold set up; `001_initial.py` applies cleanly; `make migrate-service SVC=client-api` works.
- **Story 2.2** (NEXT): Registration endpoint — depends on `users`, `companies`, `company_memberships` tables.
- **Story 2.3** (NEXT): Login / JWT — depends on `users`, `refresh_tokens` (token tables added in 2.3).
- **Story 2.10** (FUTURE): RBAC middleware — depends on `entity_permissions` table + `company_role` enum.
- **Story 2.11** (FUTURE): Audit trail middleware — depends on `shared.audit_log` table.
- **Story 2.12** (FUTURE): ESPD profile CRUD — depends on `espd_profiles` table.

### Previous Story Intelligence

Key learnings from Story 1.4 (Alembic scaffold) relevant to this story:

1. **Alembic does NOT support async drivers**: `env.py` already handles the `asyncpg → psycopg2` URL rewrite via `get_url()`. The migration itself must use `psycopg2`.
2. **`target_metadata = None` is the current state**: Story 1.4 set it to `None` because no ORM models existed. Update it to `Base.metadata` once models are created so autogenerate works.
3. **Explicit `schema=` parameter in `op.create_table`**: The `env.py` sets `search_path` to `client, shared`, but always pass `schema=CLIENT_SCHEMA` or `schema=SHARED_SCHEMA` explicitly to avoid ambiguity.
4. **`version_table_schema = "client"`**: The Alembic version table lives in the `client` schema. `migration_role` has full DDL access. This is correctly configured in the existing `env.py`.
5. **PostgreSQL ENUM creation order**: Create enum types BEFORE the tables that reference them. Drop tables BEFORE dropping enum types in downgrade.
6. **Test DB cleanup**: Teardown should downgrade to `001` (not `base`) to preserve the `_migrations_meta` table created in Story 1.4's initial migration, leaving the test DB in a clean state for other test suites.

### Project Structure After This Story

```
eusolicit-app/services/client-api/
├── alembic/
│   └── versions/
│       ├── 001_initial.py                       # EXISTING — do not modify
│       └── + 002_auth_identity_tables.py        # NEW — this story
├── src/client_api/
│   ├── main.py                                  # EXISTING — do not modify
│   └── + models/                                # NEW directory
│       ├── __init__.py                          # exports Base + all models
│       ├── base.py                              # DeclarativeBase
│       ├── enums.py                             # CompanyRole, EntityPermission
│       ├── user.py                              # User ORM model
│       ├── company.py                           # Company ORM model
│       ├── company_membership.py                # CompanyMembership ORM model
│       ├── subscription.py                      # Subscription ORM model (skeleton)
│       ├── entity_permission.py                 # EntityPermission ORM model
│       ├── espd_profile.py                      # ESPDProfile ORM model
│       └── audit_log.py                         # AuditLog ORM model (schema=shared)
└── tests/
    └── + integration/
        └── test_002_migration.py                # NEW — migration integration tests
```

### References

- [Source: eusolicit-docs/planning-artifacts/epic-02-authentication-identity.md#S02.01]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-02.md#E02-P2-001, E02-P2-002, E02-P2-003, E02-P2-014]
- [Source: eusolicit-docs/implementation-artifacts/1-4-alembic-migration-scaffold.md]
- [Source: eusolicit-docs/implementation-artifacts/deferred-work.md#deferred-from-story-1-3 (updated_at trigger)]
- [Source: eusolicit-app/packages/eusolicit-test-utils/src/eusolicit_test_utils/db.py#SERVICE_SCHEMA_MAP, apply_migrations]
- [Source: eusolicit-app/infra/postgres/init/01-init-schemas-and-roles.sql#phase-2 (shared schema, client_api_role grants)]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5)

### Debug Log References

- **Bug found and fixed**: `sa.Enum(..., create_type=False)` does NOT propagate `create_type` to the PostgreSQL dialect implementation via `_make_enum_kw()` in SQLAlchemy 2.0.48. The fix was to use `postgresql.ENUM(..., create_type=False)` directly for column type definitions in `op.create_table()` calls, ensuring the `create_type=False` flag is on the PostgreSQL-specific type object itself.
- Both enum types (`company_role`, `entity_permission`) are explicitly created via `postgresql.ENUM.create()` before the tables that reference them, and referenced in table definitions using `postgresql.ENUM(..., create_type=False)` to prevent double-creation.

### Completion Notes List

- All 42 integration tests pass (100% pass rate, 3.83s runtime)
- Quality gate satisfied: all E02-P2-001, E02-P2-002, E02-P2-003, E02-P2-014 tests pass
- `client-api` package installed in editable mode; `eusolicit-test-utils` and all sibling packages also installed
- Migration upgrade/downgrade cycle fully verified: `base → 001 → 002` and `002 → 001 → base` both clean
- `shared.audit_log` append-only enforcement verified: `client_api_role` receives PostgreSQL "permission denied" on UPDATE and DELETE
- ✅ Resolved review finding [Patch]: Added `GRANT INSERT ON shared.audit_log TO client_api_role` before the REVOKE statement; `client_api_role` now has true append-only access (INSERT + SELECT, no UPDATE/DELETE). New test `test_client_api_role_can_insert_into_audit_log` added and passing.
- ✅ Resolved review finding [Patch]: Removed redundant `op.create_index("ix_users_email", ...)` — `UniqueConstraint("email", name="uq_users_email")` already enforces uniqueness via an implicit index. Removed corresponding `op.drop_index` from downgrade to prevent failure on rollback. All 43 tests pass.
- ✅ Resolved review finding [Patch]: Added `checkfirst=True` to `company_role.create()` and `entity_permission.create()` in upgrade() — prevents crash on re-run after partial failure.
- Total post-review: 43 tests pass (100% pass rate, 3.81s runtime)

### File List

**Created:**
- `services/client-api/src/client_api/models/__init__.py`
- `services/client-api/src/client_api/models/base.py`
- `services/client-api/src/client_api/models/enums.py`
- `services/client-api/src/client_api/models/user.py`
- `services/client-api/src/client_api/models/company.py`
- `services/client-api/src/client_api/models/company_membership.py`
- `services/client-api/src/client_api/models/subscription.py`
- `services/client-api/src/client_api/models/entity_permission.py`
- `services/client-api/src/client_api/models/espd_profile.py`
- `services/client-api/src/client_api/models/audit_log.py`
- `services/client-api/alembic/versions/002_auth_identity_tables.py`
- `services/client-api/tests/integration/test_002_migration.py`

**Modified:**
- `services/client-api/alembic/env.py` — added `from client_api.models import Base`; set `target_metadata = Base.metadata`
- `services/client-api/pyproject.toml` — added `asyncpg>=0.29` to deps; added `eusolicit-test-utils` to dev extras

## Senior Developer Review

**Review Date:** 2026-04-07
**Reviewer:** Claude Opus 4.6 (code-review skill — Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Verdict:** ~~Changes Requested~~ → **Approved** (re-review 2026-04-07)

**Round 1 Summary:** 3 `patch`, 10 `defer`, 8 dismissed as noise. All 3 patches resolved.

**Round 2 (Re-review) Summary:** 0 `patch`, 0 `defer` (new), 24 dismissed. All ACs pass. Clean review.

Re-review conducted with full three-layer adversarial analysis (Blind Hunter: 15 raw findings, Edge Case Hunter: 15 raw findings, Acceptance Auditor: 4 observations → 24 unique after dedup). Every finding was either already tracked in `deferred-work.md` (10 items) or dismissed as noise/by-design/infra-dependency (14 items). All 3 prior patch items verified as correctly resolved in code and tests. 43/43 integration tests pass. All 9 acceptance criteria satisfied. No new actionable findings.

### Review Findings (Round 1 — All Resolved)

- [x] [Review][Patch] **Missing GRANT INSERT on shared.audit_log for client_api_role** — The init SQL (`01-init-schemas-and-roles.sql`) grants only SELECT on the shared schema to service roles. The migration REVOKEs UPDATE/DELETE (which were never granted — a no-op), but never GRANTs INSERT. Result: `client_api_role` has SELECT-only access to `shared.audit_log`, not append-only. Every audit log write from client-api will fail with "permission denied". The story cross-dependency section explicitly states: "`client_api_role` has INSERT-only access after this migration adds the REVOKE." Fix: add `op.execute(f"GRANT INSERT ON {SHARED_SCHEMA}.audit_log TO client_api_role")` before the REVOKE statement, and add an integration test verifying client_api_role CAN INSERT into shared.audit_log. [002_auth_identity_tables.py:288-292] ✅ Fixed
- [x] [Review][Patch] **Duplicate unique enforcement on users.email** — `sa.UniqueConstraint("email", name="uq_users_email")` at line 76 creates a unique constraint (which PostgreSQL implements as a unique index). Then `op.create_index("ix_users_email", ..., unique=True)` at line 264 creates a second unique index on the same column. This is redundant (double index maintenance cost) and the downgrade drops only `ix_users_email`, leaving `uq_users_email` behind — partial rollback. Fix: remove the explicit `create_index` call for `ix_users_email` (the UniqueConstraint already provides it) and update the downgrade to not reference that index. Alternatively, remove the UniqueConstraint from `create_table` and keep only the named index. [002_auth_identity_tables.py:76,264-270] ✅ Fixed
- [x] [Review][Patch] **Enum types created without checkfirst in upgrade** — `company_role.create(op.get_bind())` and `entity_permission.create(op.get_bind())` in upgrade() lack `checkfirst=True`. If a partial failure leaves enums created but the version_table row uncommitted, re-running upgrade crashes with "type already exists". The downgrade correctly uses `checkfirst=True`. Fix: add `checkfirst=True` to both `.create()` calls. [002_auth_identity_tables.py:35,44] ✅ Fixed
- [x] [Review][Defer] **Type annotations use `Mapped[sa.UUID]`/`Mapped[sa.DateTime]` instead of `Mapped[uuid.UUID]`/`Mapped[datetime.datetime]`** — Cosmetic; mypy may flag but functionally works with SQLAlchemy 2.0 type inference. Defer to code quality pass. [all model files]
- [x] [Review][Defer] **`onupdate=sa.func.now()` is ORM-only, not a DB trigger** — Raw SQL updates bypass it. Documented as deferred work from Story 1.3 (see deferred-work.md). [user.py:56, company.py:61, espd_profile.py:48]
- [x] [Review][Defer] **No CHECK constraint ensuring at least one auth method (hashed_password OR google_sub)** — Application-layer concern. Revisit in Story 2.2 (registration) or Story 2.6 (Google OAuth). [user.py]
- [x] [Review][Defer] **No `relationship()` declarations on ORM models** — ORM convenience; not required for this schema-only story. Add when writing CRUD endpoints in Stories 2.2+. [all model files]
- [x] [Review][Defer] **No unique constraint on `entity_permissions(user_id, company_id, entity_type, entity_id, permission)`** — RBAC design detail. Duplicate grants possible without it. Revisit in Story 2.10 (RBAC middleware). [entity_permission.py, 002_auth_identity_tables.py:162-203]
- [x] [Review][Defer] **No unique constraint on `espd_profiles(company_id, version)`** — Versioning invariant not enforced at DB level. Concurrent inserts can produce duplicate versions. Revisit in Story 2.12 (ESPD CRUD). [espd_profile.py, 002_auth_identity_tables.py:206-238]
- [x] [Review][Defer] **Missing FK indexes on `subscriptions.company_id` and `espd_profiles.company_id`** — PostgreSQL does not auto-create indexes on FK columns. Joins/filters by company_id will be sequential scans. Performance optimization; defer to query tuning. [subscription.py, espd_profile.py]
- [x] [Review][Defer] **Subscription table missing `created_at`/`updated_at` columns** — Every other entity table has timestamps. Skeleton table; billing expansion in Epic 8 will add them. [subscription.py]
- [x] [Review][Defer] **No unique constraint on `subscriptions.company_id`** — Multiple subscription rows per company allowed. Billing design detail for Epic 8. [subscription.py]
- [x] [Review][Defer] **`audit_log.user_id` is a soft reference (no FK)** — Cross-schema FK intentionally avoided per dev notes. Orphaned user_ids accumulate after user deletion. Documented design tradeoff. [audit_log.py]

## Change Log

- 2026-04-07: Story 2.1 created — Database schema for auth & identity tables (users, companies, company_memberships, subscriptions skeleton, entity_permissions, espd_profiles, shared.audit_log). Epic-level test design (E02-P2-001, E02-P2-002, E02-P2-003, E02-P2-014) mapped to integration test tasks.
- 2026-04-07: Addressed code review findings — 3 patch items resolved: (1) GRANT INSERT on shared.audit_log for client_api_role (true append-only access); (2) removed redundant ix_users_email create_index (UniqueConstraint already sufficient); (3) added checkfirst=True to enum .create() calls. 43/43 tests pass.
- 2026-04-07: Re-review approved — full three-layer adversarial review (Blind Hunter + Edge Case Hunter + Acceptance Auditor). 24 findings raised, all dismissed (10 already tracked in deferred-work.md, 14 noise/by-design). All 3 prior patches verified. All 9 ACs pass. Story status → done.
