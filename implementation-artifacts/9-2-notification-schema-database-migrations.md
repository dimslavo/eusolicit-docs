# Story 9.2: Notification Schema Database Migrations

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer on the EU Solicit notification service**,
I want **Alembic migration 002 to create `notification.alert_log`, `notification.email_log`, and `notification.sync_log` tables with correct column types, enums, indexes, and SQLAlchemy ORM models**,
so that **Stories S09.04–S09.11 (alert matching, email delivery, calendar sync, digest assembly, Stripe usage sync) have a fully-indexed, schema-scoped database schema ready to write to from day one**.

## Acceptance Criteria

1. **AC1** — Alembic migration `002_notification_tables.py` in `services/notification` runs cleanly (`alembic upgrade head`) against a database with migration 001 applied, and `alembic downgrade -1` (to revision `001_initial`) drops all three tables and enum types with no orphaned objects. Re-running `upgrade head` succeeds.

2. **AC2** — After upgrade, `notification.alert_log` exists with:
   - `id UUID PK DEFAULT gen_random_uuid()`
   - `user_id UUID NOT NULL` — soft reference to `client.users.id` (no cross-schema FK per project pattern)
   - `opportunity_ids UUID[] NOT NULL DEFAULT '{}'` — ARRAY of opportunity UUIDs included in this alert
   - `digest_type notification_digest_type NOT NULL` — enum: `immediate`, `daily`, `weekly`
   - `sent_at TIMESTAMPTZ NOT NULL DEFAULT now()`
   - `sendgrid_message_id TEXT` — nullable (populated after SendGrid delivery in S09.06)

3. **AC3** — After upgrade, `notification.email_log` exists with:
   - `id UUID PK DEFAULT gen_random_uuid()`
   - `recipient_email TEXT NOT NULL`
   - `template_type TEXT NOT NULL`
   - `sendgrid_message_id TEXT` — nullable (populated on successful send)
   - `status notification_email_status NOT NULL DEFAULT 'sent'` — enum: `sent`, `delivered`, `bounced`, `failed`
   - `sent_at TIMESTAMPTZ NOT NULL DEFAULT now()`

4. **AC4** — After upgrade, `notification.sync_log` exists with:
   - `id UUID PK DEFAULT gen_random_uuid()`
   - `calendar_connection_id UUID NOT NULL` — soft reference to `client.calendar_connections.id` (no cross-schema FK)
   - `provider notification_calendar_provider NOT NULL` — enum: `google`, `microsoft`
   - `sync_type notification_sync_type NOT NULL` — enum: `full`, `incremental`
   - `events_created INTEGER NOT NULL DEFAULT 0`
   - `events_updated INTEGER NOT NULL DEFAULT 0`
   - `events_deleted INTEGER NOT NULL DEFAULT 0`
   - `started_at TIMESTAMPTZ NOT NULL DEFAULT now()`
   - `completed_at TIMESTAMPTZ` — nullable (NULL if sync is still running or errored)
   - `error_message TEXT` — nullable

5. **AC5** — All required indexes are present after migration:
   - `ix_alert_log_user_id` on `notification.alert_log(user_id)` — supports per-user alert history queries
   - `ix_alert_log_sent_at` on `notification.alert_log(sent_at)` — supports digest window lookups (last sent per user)
   - `ix_email_log_sendgrid_message_id` on `notification.email_log(sendgrid_message_id)` — supports webhook status update lookups
   - `ix_email_log_status` on `notification.email_log(status)` — supports filtering by delivery state
   - `ix_sync_log_calendar_connection_id` on `notification.sync_log(calendar_connection_id)` — supports per-connection sync history

6. **AC6** — Four PostgreSQL enum types are created, all scoped to `schema='notification'`:
   - `notification_digest_type` — values: `immediate`, `daily`, `weekly`
   - `notification_email_status` — values: `sent`, `delivered`, `bounced`, `failed`
   - `notification_calendar_provider` — values: `google`, `microsoft`
   - `notification_sync_type` — values: `full`, `incremental`

7. **AC7** — SQLAlchemy ORM models created in `services/notification/src/notification/models/`:
   - `notification/models/alert_log.py` — `AlertLog` ORM model, `notification` schema
   - `notification/models/email_log.py` — `EmailLog` ORM model, `notification` schema
   - `notification/models/sync_log.py` — `SyncLog` ORM model, `notification` schema
   - `notification/models/__init__.py` — exports all three models
   - `alembic check` runs without error (note: `target_metadata = None` in `env.py` means autogenerate is disabled — this is by design; "no pending changes" is vacuously satisfied; ORM models are for query use only, not autogenerate input)

8. **AC8** — Integration tests pass: migration runs clean on `eusolicit_test`, all three tables + enum types + columns + indexes verified via `information_schema` and `pg_catalog` queries, and `downgrade` restores the prior state. Tests at `services/notification/tests/integration/test_002_migration.py` (test IDs E09-DB-001 through E09-DB-012 per Dev Notes).

## Tasks / Subtasks

- [x] Task 1: Create Alembic migration 002 (AC: 1, 2, 3, 4, 5, 6)
  - [x] 1.1 Create `services/notification/alembic/versions/002_notification_tables.py` with `down_revision = "001"` and `revision = "002"`
  - [x] 1.2 `upgrade()` step A — Create 4 enum types in `notification` schema: `notification_digest_type`, `notification_email_status`, `notification_calendar_provider`, `notification_sync_type`
  - [x] 1.3 `upgrade()` step B — Create `notification.alert_log` table with all columns (UUID PK, user_id, opportunity_ids ARRAY, digest_type, sent_at, sendgrid_message_id)
  - [x] 1.4 `upgrade()` step C — Create `notification.email_log` table with all columns
  - [x] 1.5 `upgrade()` step D — Create `notification.sync_log` table with all columns
  - [x] 1.6 `upgrade()` step E — Create all 5 indexes
  - [x] 1.7 `downgrade()` — Drop indexes → drop tables → drop enum types in exact reverse order

- [x] Task 2: Create ORM base and `AlertLog` model (AC: 7)
  - [x] 2.1 Create `services/notification/src/notification/models/` directory (new)
  - [x] 2.2 Create `services/notification/src/notification/models/base.py` with `DeclarativeBase` — shared base for all notification ORM models
  - [x] 2.3 Create `services/notification/src/notification/models/alert_log.py` with `AlertLog` mapped to `notification.alert_log`

- [x] Task 3: Create `EmailLog` ORM model (AC: 7)
  - [x] 3.1 Create `services/notification/src/notification/models/email_log.py` with `EmailLog` mapped to `notification.email_log`

- [x] Task 4: Create `SyncLog` ORM model (AC: 7)
  - [x] 4.1 Create `services/notification/src/notification/models/sync_log.py` with `SyncLog` mapped to `notification.sync_log`

- [x] Task 5: Update `models/__init__.py` to export all models (AC: 7)
  - [x] 5.1 Add `from .alert_log import AlertLog`
  - [x] 5.2 Add `from .email_log import EmailLog`
  - [x] 5.3 Add `from .sync_log import SyncLog`
  - [x] 5.4 Add `__all__` list with all three exports

- [x] Task 6: Write integration tests — migration 002 (AC: 1, 2, 3, 4, 5, 6, 8)
  - [x] 6.1 Create `services/notification/tests/integration/test_002_migration.py` covering test IDs E09-DB-001 through E09-DB-012 (see Dev Notes)

## Dev Notes

### CRITICAL: Existing Infrastructure — Do NOT Modify

| Path | What Exists | Action |
|------|-------------|--------|
| `services/notification/alembic/versions/001_initial.py` | Creates `notification._migrations_meta` | **DO NOT TOUCH** |
| `services/notification/alembic/env.py` | Configured with `search_path = 'notification, shared'` | **DO NOT TOUCH** |
| `services/notification/src/notification/workers/` | Celery app, tasks — all S09.01 work | **DO NOT TOUCH** |
| `infra/postgres/init/01-init-schemas-and-roles.sql` | Creates `notification` schema + `notification_role` | **DO NOT TOUCH** |

**No `models/` directory exists yet** in the notification service. Task 2 creates it. This is correct — no models existed before this story.

### Migration File — Full Implementation

```python
# services/notification/alembic/versions/002_notification_tables.py
"""Notification audit tables: alert_log, email_log, sync_log.

Revision ID: 002
Revises: 001
Create Date: 2026-04-19

These three tables are the audit backbone of the Notification Service:
- alert_log: Records every alert dispatched to a user (immediate, daily, weekly digests)
- email_log: Records every email send attempt with SendGrid message ID and delivery status
- sync_log: Records every calendar sync run (Google/Microsoft) with event counts

All enum types are schema-scoped to 'notification' to avoid conflicts with
enums in 'client' or 'pipeline' schemas.

[Source: eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md#S09.02]
"""
from __future__ import annotations

import sqlalchemy as sa
from alembic import op

revision = "002"
down_revision = "001"
branch_labels = None
depends_on = None

SCHEMA = "notification"

# Enum type definitions — scoped to notification schema
DIGEST_TYPE = sa.Enum(
    "immediate", "daily", "weekly",
    name="notification_digest_type",
    schema=SCHEMA,
)
EMAIL_STATUS = sa.Enum(
    "sent", "delivered", "bounced", "failed",
    name="notification_email_status",
    schema=SCHEMA,
)
CALENDAR_PROVIDER = sa.Enum(
    "google", "microsoft",
    name="notification_calendar_provider",
    schema=SCHEMA,
)
SYNC_TYPE = sa.Enum(
    "full", "incremental",
    name="notification_sync_type",
    schema=SCHEMA,
)


def upgrade() -> None:
    # Step A: Create enum types first (tables reference them)
    DIGEST_TYPE.create(op.get_bind(), checkfirst=True)
    EMAIL_STATUS.create(op.get_bind(), checkfirst=True)
    CALENDAR_PROVIDER.create(op.get_bind(), checkfirst=True)
    SYNC_TYPE.create(op.get_bind(), checkfirst=True)

    # Step B: Create notification.alert_log
    op.create_table(
        "alert_log",
        sa.Column(
            "id",
            sa.UUID(as_uuid=True),
            primary_key=True,
            server_default=sa.text("gen_random_uuid()"),
        ),
        sa.Column("user_id", sa.UUID(as_uuid=True), nullable=False),
        sa.Column(
            "opportunity_ids",
            sa.ARRAY(sa.UUID(as_uuid=True)),
            nullable=False,
            server_default=sa.text("'{}'::uuid[]"),
        ),
        sa.Column("digest_type", DIGEST_TYPE, nullable=False),
        sa.Column(
            "sent_at",
            sa.TIMESTAMP(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column("sendgrid_message_id", sa.Text, nullable=True),
        schema=SCHEMA,
    )

    # Step C: Create notification.email_log
    op.create_table(
        "email_log",
        sa.Column(
            "id",
            sa.UUID(as_uuid=True),
            primary_key=True,
            server_default=sa.text("gen_random_uuid()"),
        ),
        sa.Column("recipient_email", sa.Text, nullable=False),
        sa.Column("template_type", sa.Text, nullable=False),
        sa.Column("sendgrid_message_id", sa.Text, nullable=True),
        sa.Column("status", EMAIL_STATUS, nullable=False, server_default="sent"),
        sa.Column(
            "sent_at",
            sa.TIMESTAMP(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        schema=SCHEMA,
    )

    # Step D: Create notification.sync_log
    op.create_table(
        "sync_log",
        sa.Column(
            "id",
            sa.UUID(as_uuid=True),
            primary_key=True,
            server_default=sa.text("gen_random_uuid()"),
        ),
        sa.Column("calendar_connection_id", sa.UUID(as_uuid=True), nullable=False),
        sa.Column("provider", CALENDAR_PROVIDER, nullable=False),
        sa.Column("sync_type", SYNC_TYPE, nullable=False),
        sa.Column("events_created", sa.Integer, nullable=False, server_default="0"),
        sa.Column("events_updated", sa.Integer, nullable=False, server_default="0"),
        sa.Column("events_deleted", sa.Integer, nullable=False, server_default="0"),
        sa.Column(
            "started_at",
            sa.TIMESTAMP(timezone=True),
            nullable=False,
            server_default=sa.func.now(),
        ),
        sa.Column("completed_at", sa.TIMESTAMP(timezone=True), nullable=True),
        sa.Column("error_message", sa.Text, nullable=True),
        schema=SCHEMA,
    )

    # Step E: Create indexes
    op.create_index(
        "ix_alert_log_user_id",
        "alert_log",
        ["user_id"],
        schema=SCHEMA,
    )
    op.create_index(
        "ix_alert_log_sent_at",
        "alert_log",
        ["sent_at"],
        schema=SCHEMA,
    )
    op.create_index(
        "ix_email_log_sendgrid_message_id",
        "email_log",
        ["sendgrid_message_id"],
        schema=SCHEMA,
    )
    op.create_index(
        "ix_email_log_status",
        "email_log",
        ["status"],
        schema=SCHEMA,
    )
    op.create_index(
        "ix_sync_log_calendar_connection_id",
        "sync_log",
        ["calendar_connection_id"],
        schema=SCHEMA,
    )


def downgrade() -> None:
    # Reverse order: indexes → tables → enum types
    op.drop_index("ix_sync_log_calendar_connection_id", table_name="sync_log", schema=SCHEMA)
    op.drop_index("ix_email_log_status", table_name="email_log", schema=SCHEMA)
    op.drop_index("ix_email_log_sendgrid_message_id", table_name="email_log", schema=SCHEMA)
    op.drop_index("ix_alert_log_sent_at", table_name="alert_log", schema=SCHEMA)
    op.drop_index("ix_alert_log_user_id", table_name="alert_log", schema=SCHEMA)

    op.drop_table("sync_log", schema=SCHEMA)
    op.drop_table("email_log", schema=SCHEMA)
    op.drop_table("alert_log", schema=SCHEMA)

    SYNC_TYPE.drop(op.get_bind(), checkfirst=True)
    CALENDAR_PROVIDER.drop(op.get_bind(), checkfirst=True)
    EMAIL_STATUS.drop(op.get_bind(), checkfirst=True)
    DIGEST_TYPE.drop(op.get_bind(), checkfirst=True)
```

### CRITICAL: Enum Scoping

The epic says "Use `sa.Enum` for digest_type, status, provider, sync_type with `schema='notification'` to scope enums." This is **mandatory** — without `schema=SCHEMA`, Alembic creates enum types in the **public schema** which:
1. Conflicts with same-named enums if ever created in other services
2. Is not cleaned up by `alembic downgrade` scoped to `notification` schema
3. Fails if the migration role lacks `CREATE` permission on `public` (which it shouldn't)

Use the pattern above where enum objects are declared at module level with `schema=SCHEMA` and called via `.create()` / `.drop()` explicitly. Do **NOT** define enums inline in `sa.Column(...)` — the `create_constraint=False` / `schema=` interaction is fragile and drops silently.

### CRITICAL: ARRAY Column Type

The `opportunity_ids` column uses PostgreSQL native ARRAY type:
```python
sa.Column(
    "opportunity_ids",
    sa.ARRAY(sa.UUID(as_uuid=True)),
    nullable=False,
    server_default=sa.text("'{}'::uuid[]"),
)
```
The `server_default` **must** use `sa.text("'{}'::uuid[]")` — not `sa.text("ARRAY[]::uuid[]")` which PostgreSQL rejects as a column default. Always pass the cast explicitly.

### ORM Models — Full Implementation

SQLAlchemy async ORM models use `DeclarativeBase` (SQLAlchemy 2.x pattern). Note that `target_metadata = None` in `alembic/env.py` means Alembic does NOT autogenerate from models — models exist for query use only, not for migration generation.

```python
# services/notification/src/notification/models/__init__.py
"""SQLAlchemy ORM models for the notification service.

These models map to tables in the 'notification' PostgreSQL schema.
All models use SQLAlchemy 2.x mapped_column() style.

Note: target_metadata in alembic/env.py is None — migrations are written
manually, not autogenerated from these models. alembic check will show
"Nothing new to generate" because autogenerate is disabled.
"""
from __future__ import annotations

from .alert_log import AlertLog
from .email_log import EmailLog
from .sync_log import SyncLog

__all__ = ["AlertLog", "EmailLog", "SyncLog"]
```

```python
# services/notification/src/notification/models/alert_log.py
"""AlertLog ORM model — notification.alert_log table."""
from __future__ import annotations

import uuid
from datetime import datetime

from sqlalchemy import ARRAY, Enum, Text, UUID
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

from notification.models.base import Base


class AlertLog(Base):
    """Records every alert dispatched (immediate, daily, weekly digest).

    Populated by S09.04 (alert matching) and S09.05 (digest assembly).
    Read by S09.05 to determine the last-sent timestamp for digest windows.
    """
    __tablename__ = "alert_log"
    __table_args__ = {"schema": "notification"}

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
        server_default="gen_random_uuid()",
    )
    user_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    opportunity_ids: Mapped[list[uuid.UUID]] = mapped_column(
        ARRAY(UUID(as_uuid=True)), nullable=False, default=list, server_default="'{}'::uuid[]"
    )
    digest_type: Mapped[str] = mapped_column(
        Enum("immediate", "daily", "weekly", name="notification_digest_type", schema="notification"),
        nullable=False,
    )
    sent_at: Mapped[datetime] = mapped_column(nullable=False, server_default="now()")
    sendgrid_message_id: Mapped[str | None] = mapped_column(Text, nullable=True)
```

```python
# services/notification/src/notification/models/base.py
"""Shared DeclarativeBase for notification ORM models."""
from __future__ import annotations

from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass
```

```python
# services/notification/src/notification/models/email_log.py
"""EmailLog ORM model — notification.email_log table."""
from __future__ import annotations

import uuid
from datetime import datetime

from sqlalchemy import Enum, Text, UUID
from sqlalchemy.orm import Mapped, mapped_column

from notification.models.base import Base


class EmailLog(Base):
    """Records every email send attempt via SendGrid.

    Populated by S09.06 (send_email task). Status updated by SendGrid webhook handler.
    """
    __tablename__ = "email_log"
    __table_args__ = {"schema": "notification"}

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
        server_default="gen_random_uuid()",
    )
    recipient_email: Mapped[str] = mapped_column(Text, nullable=False)
    template_type: Mapped[str] = mapped_column(Text, nullable=False)
    sendgrid_message_id: Mapped[str | None] = mapped_column(Text, nullable=True)
    status: Mapped[str] = mapped_column(
        Enum("sent", "delivered", "bounced", "failed", name="notification_email_status", schema="notification"),
        nullable=False,
        server_default="sent",
    )
    sent_at: Mapped[datetime] = mapped_column(nullable=False, server_default="now()")
```

```python
# services/notification/src/notification/models/sync_log.py
"""SyncLog ORM model — notification.sync_log table."""
from __future__ import annotations

import uuid
from datetime import datetime

from sqlalchemy import Enum, Integer, Text, UUID
from sqlalchemy.orm import Mapped, mapped_column

from notification.models.base import Base


class SyncLog(Base):
    """Records every Google/Microsoft calendar sync run.

    Populated by S09.08 (Google Calendar) and S09.09 (Microsoft Outlook) sync tasks.
    """
    __tablename__ = "sync_log"
    __table_args__ = {"schema": "notification"}

    id: Mapped[uuid.UUID] = mapped_column(
        UUID(as_uuid=True),
        primary_key=True,
        default=uuid.uuid4,
        server_default="gen_random_uuid()",
    )
    calendar_connection_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), nullable=False)
    provider: Mapped[str] = mapped_column(
        Enum("google", "microsoft", name="notification_calendar_provider", schema="notification"),
        nullable=False,
    )
    sync_type: Mapped[str] = mapped_column(
        Enum("full", "incremental", name="notification_sync_type", schema="notification"),
        nullable=False,
    )
    events_created: Mapped[int] = mapped_column(Integer, nullable=False, default=0, server_default="0")
    events_updated: Mapped[int] = mapped_column(Integer, nullable=False, default=0, server_default="0")
    events_deleted: Mapped[int] = mapped_column(Integer, nullable=False, default=0, server_default="0")
    started_at: Mapped[datetime] = mapped_column(nullable=False, server_default="now()")
    completed_at: Mapped[datetime | None] = mapped_column(nullable=True)
    error_message: Mapped[str | None] = mapped_column(Text, nullable=True)
```

### No Cross-Schema FKs

The epic spec says "user_id UUID FK" and "calendar_connection_id UUID FK" — **do not create actual database foreign key constraints across schemas**. The project pattern is explicit: "Cross-service reads go through APIs, never direct DB joins." These are soft UUID references enforced at the application layer only. The migration code above correctly omits `sa.ForeignKeyConstraint` on both columns.

### File Structure After This Story

```
eusolicit-app/services/notification/
├── alembic/
│   └── versions/
│       ├── 001_initial.py          # EXISTING — do not touch
│       └── 002_notification_tables.py   # + NEW — this story
└── src/notification/
    ├── models/                     # + NEW directory
    │   ├── __init__.py             # + NEW — exports AlertLog, EmailLog, SyncLog
    │   ├── base.py                 # + NEW — DeclarativeBase
    │   ├── alert_log.py            # + NEW — AlertLog ORM model
    │   ├── email_log.py            # + NEW — EmailLog ORM model
    │   └── sync_log.py             # + NEW — SyncLog ORM model
    └── workers/                    # EXISTING — do not touch
        └── tasks/                  # EXISTING — all S09.01 stubs intact
```

### Integration Test Structure — E09-DB-001 through E09-DB-012

File: `services/notification/tests/integration/test_002_migration.py`

Follow the same patterns as `services/client-api/tests/integration/test_020_migration.py` (Story 7.1), adapted for the notification service. Tests run via `sys.executable -m alembic` subprocess calls and async SQLAlchemy queries via `information_schema` and `pg_catalog`.

| Test ID | AC | Description | Level |
|---------|-----|-------------|-------|
| E09-DB-001 | AC1 | Upgrade from 001 to 002 completes without error | integration |
| E09-DB-002 | AC1 | Downgrade from 002 to 001 drops tables and enums | integration |
| E09-DB-003 | AC1 | Re-run upgrade after downgrade succeeds (idempotent) | integration |
| E09-DB-004 | AC2 | `notification.alert_log` exists with all 6 columns and correct types | integration |
| E09-DB-005 | AC3 | `notification.email_log` exists with all 6 columns and correct types | integration |
| E09-DB-006 | AC4 | `notification.sync_log` exists with all 10 columns and correct types | integration |
| E09-DB-007 | AC6 | `notification_digest_type` enum has values `immediate, daily, weekly` | integration |
| E09-DB-008 | AC6 | `notification_email_status` enum has values `sent, delivered, bounced, failed` | integration |
| E09-DB-009 | AC6 | `notification_calendar_provider` enum has values `google, microsoft` | integration |
| E09-DB-010 | AC6 | `notification_sync_type` enum has values `full, incremental` | integration |
| E09-DB-011 | AC5 | All 5 indexes present in `pg_indexes` with correct table/column | integration |
| E09-DB-012 | AC2 | `opportunity_ids` column is PostgreSQL ARRAY of uuid (data_type = ARRAY, udt_name = uuid) | integration |

**Test skeleton** (pattern from Story 7.1, adapted for notification):

```python
# services/notification/tests/integration/test_002_migration.py
"""Integration tests for migration 002 — notification tables.

Requires: running PostgreSQL with eusolicit_test DB and notification schema.
Run via: pytest -m integration services/notification/tests/integration/test_002_migration.py

Test IDs: E09-DB-001 through E09-DB-012
"""
from __future__ import annotations

import subprocess
import sys
from pathlib import Path

import pytest
import pytest_asyncio
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy import text

NOTIFICATION_SVC_ROOT = Path(__file__).parents[2]  # services/notification/
REPO_ROOT = Path(__file__).parents[4]              # eusolicit-app/

pytestmark = pytest.mark.integration


@pytest.fixture(scope="module")
def run_alembic():
    """Run alembic commands from the notification service root."""
    def _run(*args):
        result = subprocess.run(
            [sys.executable, "-m", "alembic", *args],
            cwd=NOTIFICATION_SVC_ROOT,
            capture_output=True,
            text=True,
        )
        assert result.returncode == 0, f"alembic {args} failed:\n{result.stderr}"
        return result
    return _run


@pytest.fixture(scope="module", autouse=True)
def migration_state(run_alembic):
    """Ensure test DB is at revision 001 before tests, 002 after."""
    run_alembic("downgrade", "001")  # reset to known state
    run_alembic("upgrade", "002")    # apply migration under test
    yield
    # Leave at 002 for dev iteration (do not downgrade after tests)


@pytest.mark.asyncio
async def test_e09_db_001_upgrade_succeeds(run_alembic):
    """E09-DB-001: Migration 002 upgrade completes cleanly."""
    result = run_alembic("current")
    assert "002" in result.stdout


@pytest.mark.asyncio
async def test_e09_db_011_indexes_present(notification_session: AsyncSession):
    """E09-DB-011: All 5 required indexes exist."""
    result = await notification_session.execute(
        text("""
            SELECT indexname FROM pg_indexes
            WHERE schemaname = 'notification'
            AND indexname IN (
                'ix_alert_log_user_id',
                'ix_alert_log_sent_at',
                'ix_email_log_sendgrid_message_id',
                'ix_email_log_status',
                'ix_sync_log_calendar_connection_id'
            )
        """)
    )
    found = {row[0] for row in result.fetchall()}
    expected = {
        "ix_alert_log_user_id",
        "ix_alert_log_sent_at",
        "ix_email_log_sendgrid_message_id",
        "ix_email_log_status",
        "ix_sync_log_calendar_connection_id",
    }
    assert found == expected, f"Missing indexes: {expected - found}"

# ... complete remaining test IDs following the same pattern
```

### Previous Story Intelligence (S09.01)

From Story 9.1 completion notes:
- **Alembic chain**: `001_initial.py` creates `notification._migrations_meta`. The chain is clean; `alembic upgrade head` on a fresh DB runs migration 001 first, then 002. Confirm with `make migrate-service SVC=notification`.
- **env.py** sets `search_path = 'notification, shared'` inside the Alembic transaction. This means migration code referencing `SCHEMA = "notification"` in `schema=SCHEMA` kwargs is correct and explicit.
- **`target_metadata = None`** — autogenerate is disabled. Alembic does NOT generate migrations from ORM models. The ORM models in Task 2–4 are for query use, not autogenerate input.
- **No `models/` directory** — confirmed by S09.01 file list. This story creates it fresh.
- **pyproject.toml** already has `alembic>=1.13`, `psycopg2-binary>=2.9`, `sqlalchemy[asyncio]>=2.0` — no new dependencies needed.

**Learnings from similar migration story (7-1)**:
- Always verify migration with both `alembic upgrade head` AND explicit `alembic current` check
- Downgrade test must verify tables are truly gone (query `information_schema.tables`)
- Enum types must be dropped in reverse dependency order
- Integration tests use `pytest -m integration` marker — confirm `conftest.py` has `notification_session` fixture (it does — see `tests/conftest.py`)

### Cross-Story Dependencies

| Story | Status | Relevance to S09.02 |
|-------|--------|---------------------|
| S01.03 — PostgreSQL Schema & Roles | done | `notification` schema exists; `notification_role` has CRUD on `notification.*` |
| S01.04 — Alembic Scaffold | done | Alembic configured in notification service; `001_initial.py` is baseline |
| S09.01 — Notification Service Scaffold | done | Celery + Beat + stub tasks set up; `002` is the next migration in the chain |
| S09.04 — Alert Matching & Immediate Dispatch | backlog | **Reads and writes `notification.alert_log`** — requires this migration |
| S09.05 — Daily/Weekly Digest Assembly | backlog | **Reads `notification.alert_log`** (digest window calc) — requires this migration |
| S09.06 — SendGrid Email Delivery | backlog | **Writes `notification.email_log`** on every send — requires this migration |
| S09.08/09 — Calendar OAuth2 & Sync | backlog | **Writes `notification.sync_log`** after each sync run — requires this migration |
| S09.11 — Trial Expiry & Stripe Usage Sync | backlog | **Reads `notification.email_log`** indirectly (audit trail) |

### Test Expectations from Epic-Level Test Design

From `test-design-epic-09.md`:

The three tables created in this story are **prerequisite infrastructure** for ALL P0 tests in the epic:

| Test Design Scenario | Depends on S09.02 | Notes |
|---------------------|-------------------|-------|
| P0: Immediate Notification Match (E2E) | ✅ `notification.alert_log` | Alert logged with `digest_type='immediate'` |
| P0: Daily/Weekly Digest (API) | ✅ `notification.alert_log` | Digest window from `MAX(sent_at)` per user |
| P0: Stripe Usage Sync (API) | indirect | `email_log` used for audit trail in S09.06 |
| P0: Alert Preferences CRUD (E2E) | indirect | Preferences in `client` schema (S09.03) |
| P1: SendGrid Webhook (API) | ✅ `notification.email_log` | Status updated via webhook handler |
| P2: Calendar Sync diff (API) | ✅ `notification.sync_log` | Sync results logged here |

**Entry criteria from test design** (must be satisfied before S09.04 ATDD):
- `notification.alert_log` created with `user_id`, `opportunity_ids ARRAY(UUID)`, `digest_type` enum
- `notification.email_log` created with `sendgrid_message_id` and `status` enum
- `notification.sync_log` created with `calendar_connection_id`, `provider`, `sync_type` enums

### Risk Mitigations

| Risk ID | Description | Mitigation in S09.02 |
|---------|-------------|---------------------|
| **R-001** | Redis stream event loss for immediate notifications (Score: 6) | `alert_log` table provides the durable audit trail — every sent alert is recorded regardless of stream ACK state |
| **R-002** | Stripe Usage over/under-reporting (Score: 6) | `email_log` indirectly supports audit trail for send attempts used in usage tracking |

### Verification Commands

```bash
# Apply migration
cd eusolicit-app && make migrate-service SVC=notification

# Check current revision
cd services/notification && python -m alembic current

# Verify tables exist (in psql)
\dt notification.*

# Run integration tests (requires running postgres)
cd eusolicit-app && make test-integration SVC=notification
```

### Project Structure Notes

- Alembic uses **sync psycopg2** driver (`env.py` replaces `+asyncpg` with `+psycopg2`) — migrations always use sync connections even though the app uses async SQLAlchemy.
- All table and column names use **snake_case** per PostgreSQL convention.
- ORM models use `TIMESTAMP(timezone=True)` / `TIMESTAMPTZ` — never naive `TIMESTAMP`.
- The `notification` schema is scoped to this service. No other service should create tables here.

### References

- [Source: eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md#S09.02]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-09.md]
- [Source: eusolicit-app/services/notification/alembic/versions/001_initial.py]
- [Source: eusolicit-app/services/notification/alembic/env.py]
- [Source: eusolicit-app/services/notification/tests/conftest.py] — `notification_session` fixture for integration tests
- [Source: eusolicit-docs/implementation-artifacts/9-1-notification-service-scaffold-celery-configuration.md] — previous story, migration chain context
- [Source: eusolicit-docs/implementation-artifacts/7-1-proposal-version-db-schema-migrations.md] — reference pattern for migration story structure and integration tests
- [Source: eusolicit-app/services/client-api/tests/integration/test_020_migration.py] — canonical migration integration test pattern

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (Claude Code, 2026-04-19)

### Debug Log References

- **Issue 1**: `sa.Enum` does not have a `create_type` attribute in SQLAlchemy 2.0.48. When `sa.Enum` objects are used as column types in `op.create_table()`, SQLAlchemy fires a `before_create` event that tries to auto-`CREATE TYPE` within the same transaction — even after we explicitly called `.create(checkfirst=True)`. This causes `DuplicateObject` error.
  - **Fix**: Separated enum objects into two sets: `sa.Enum` objects for lifecycle management (`.create()` / `.drop()`), and `postgresql.ENUM(create_type=False)` objects for column definitions inside `op.create_table()`. The `postgresql.ENUM` class supports the `create_type` parameter; `sa.Enum` does not.

- **Issue 2 (DEVIATION)**: `test_alembic_check_no_error` expects `alembic check` to return exit code 0 when `target_metadata = None` in env.py. In SQLAlchemy 2.x / Alembic, `alembic check` requires a MetaData object and always returns 255 with "Can't proceed with --autogenerate option" when `target_metadata = None`. The Dev Notes explicitly prohibit modifying env.py and state ORM models are for "query use only, not autogenerate input." This is an `ACCEPTANCE_GAP` in the test design — the correct verification for manual migrations is `alembic current | grep "(head)"`, not `alembic check`. Test AC7-ORM-006 fails (43/44 pass). DEVIATION_TYPE: ACCEPTANCE_GAP. DEVIATION_SEVERITY: deferrable.

### Completion Notes List

- **Migration 002** created at `services/notification/alembic/versions/002_notification_tables.py`. Runs clean from 001 → 002. Round-trip downgrade→upgrade verified. `alembic current` shows `002 (head)`.
- **4 enum types** created schema-scoped to `notification`: `notification_digest_type`, `notification_email_status`, `notification_calendar_provider`, `notification_sync_type`. All confirmed NOT in `public` schema.
- **3 tables** created: `notification.alert_log` (6 cols), `notification.email_log` (6 cols), `notification.sync_log` (10 cols). ARRAY column `opportunity_ids` with `'{}'::uuid[]` default confirmed.
- **5 indexes** created and verified in `pg_indexes`.
- **ORM models** created under `src/notification/models/`: `base.py`, `alert_log.py`, `email_log.py`, `sync_log.py`, `__init__.py`. SQLAlchemy 2.x `mapped_column()` style used throughout.
- **Integration tests**: 43/44 PASS. 1 test (`test_alembic_check_no_error`) fails due to ACCEPTANCE_GAP — `alembic check` requires `target_metadata` which is `None` by design in this service.
- **Unit tests**: 162/162 PASS. Zero regressions in notification service.
- **Linting**: `ruff check` — all checks passed after auto-fixing import sort order on model files.

**Review Follow-up (2026-04-19):**
- ✅ Resolved review finding [blocking]: ORM models now explicitly use `TIMESTAMP(timezone=True)` for all datetime columns (`sent_at` in `alert_log.py` and `email_log.py`; `started_at` and `completed_at` in `sync_log.py`). Architectural requirement satisfied — no naive TIMESTAMP mappings remain.
- ✅ Resolved review finding [deferrable]: Replaced `test_alembic_check_no_error` with `test_target_metadata_none_and_db_at_head` which verifies the AC7 intent by: (1) asserting `env.py` has `target_metadata = None` (autogenerate disabled by design), and (2) asserting `alembic current` shows `(head)`. **44/44 integration tests PASS. 240/240 total tests PASS.** Zero regressions.

### File List

services/notification/alembic/versions/002_notification_tables.py (new)
services/notification/src/notification/models/__init__.py (new)
services/notification/src/notification/models/base.py (new)
services/notification/src/notification/models/alert_log.py (new — updated: TIMESTAMP(timezone=True) for sent_at)
services/notification/src/notification/models/email_log.py (new — updated: TIMESTAMP(timezone=True) for sent_at)
services/notification/src/notification/models/sync_log.py (new — updated: TIMESTAMP(timezone=True) for started_at, completed_at)
services/notification/tests/integration/test_002_migration.py (new — updated: replaced test_alembic_check_no_error with test_target_metadata_none_and_db_at_head)

## Change Log

- **2026-04-19** (claude-sonnet-4-5): Initial implementation — migration 002, ORM models (base, alert_log, email_log, sync_log), integration tests E09-DB-001 through E09-DB-012, AC7 ORM tests.
- **2026-04-19** (claude-sonnet-4-7): Addressed code review findings — 2 items resolved: (1) [blocking] Added explicit `TIMESTAMP(timezone=True)` to all datetime ORM columns (`sent_at`, `started_at`, `completed_at`); (2) [deferrable] Replaced `test_alembic_check_no_error` with `test_target_metadata_none_and_db_at_head` — verifies AC7 intent via env.py content check + `alembic current (head)`. 44/44 integration tests PASS; 240/240 total tests PASS.

## Senior Developer Review

**Status:** Changes Requested

### Findings
1. **DEVIATION (ACCEPTANCE_GAP):** `test_alembic_check_no_error` in `services/notification/tests/integration/test_002_migration.py` expects `alembic check` to succeed but fails because `target_metadata = None` in `env.py`. This is a gap in the test design.
2. **DEVIATION (ARCHITECTURAL_DRIFT):** The ORM models (`alert_log.py`, `email_log.py`, `sync_log.py`) use `Mapped[datetime]` without explicitly defining the timezone-aware timestamp type (`TIMESTAMP(timezone=True)`). SQLAlchemy 2 maps `datetime` to naive `TIMESTAMP` by default, violating the explicit architectural instruction: "ORM models use TIMESTAMP(timezone=True) / TIMESTAMPTZ — never naive TIMESTAMP."

## Known Deviations

### Detected by `3-code-review` at 2026-04-19T08:34:12Z

- `test_alembic_check_no_error` expects `alembic check` to succeed but it fails since `target_metadata = None` in `env.py`. The acceptance criteria are untestable using this command in SQLAlchemy 2 / Alembic without metadata. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- ORM models (`alert_log.py`, `email_log.py`, `sync_log.py`) use `Mapped[datetime]` without explicitly defining the type as `TIMESTAMP(timezone=True)`. This maps to naive timestamps in SQLAlchemy 2, violating the explicit architectural instruction. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
