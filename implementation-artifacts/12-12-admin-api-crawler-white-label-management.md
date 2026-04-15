# Story 12.12: Admin API — Crawler & White-Label Management

Status: done

## Story

As a **platform operator (internal admin)**,
I want **VPN-restricted admin API endpoints to inspect crawler run history, manage crawler schedules, trigger manual crawls, and configure per-tenant white-label branding**,
so that **operations staff can monitor and control data ingestion pipelines and configure custom branding for enterprise tenants without touching the database or Celery config directly**.

## Acceptance Criteria

### AC1 — DB Migration 006: Crawler & White-Label Tables

New migration: `services/admin-api/alembic/versions/006_crawler_whitelabel_schema.py`

```
revision = "006"
down_revision = "005"
```

Creates three new tables in the `admin` schema (admin-api owns this schema — no additional grants needed):

**`admin.crawler_runs`** — historical record of each crawl execution:

```sql
CREATE TABLE admin.crawler_runs (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    crawler_type TEXT NOT NULL,          -- 'ted', 'find_a_tender', 'merps', 'grants_gov'
    status      TEXT NOT NULL DEFAULT 'pending',   -- pending | running | completed | failed
    triggered_by TEXT NOT NULL DEFAULT 'scheduled', -- 'scheduled' | 'manual'
    triggered_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at   TIMESTAMPTZ,
    completed_at TIMESTAMPTZ,
    result_count INTEGER,                -- opportunities found/updated
    error_detail TEXT,                   -- error message if status='failed'
    celery_task_id TEXT                  -- Celery async_result id for polling
);
CREATE INDEX idx_crawler_runs_type_triggered ON admin.crawler_runs (crawler_type, triggered_at DESC);
CREATE INDEX idx_crawler_runs_status ON admin.crawler_runs (status);
```

**`admin.crawler_schedules`** — per-type schedule configuration (source of truth for Celery Beat):

```sql
CREATE TABLE admin.crawler_schedules (
    crawler_type    TEXT PRIMARY KEY,    -- 'ted', 'find_a_tender', 'merps', 'grants_gov'
    cron_minute     TEXT NOT NULL DEFAULT '0',
    cron_hour       TEXT NOT NULL DEFAULT '1',
    cron_day_of_week TEXT NOT NULL DEFAULT '*',
    cron_day_of_month TEXT NOT NULL DEFAULT '*',
    cron_month_of_year TEXT NOT NULL DEFAULT '*',
    enabled         BOOLEAN NOT NULL DEFAULT TRUE,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_by_admin_id UUID
);
-- Seed default schedules for known crawler types
INSERT INTO admin.crawler_schedules (crawler_type, cron_hour, cron_minute) VALUES
    ('ted',              '1', '0'),
    ('find_a_tender',    '1', '15'),
    ('merps',            '1', '30'),
    ('grants_gov',       '1', '45');
```

**`admin.white_label_settings`** — per-tenant branding configuration:

```sql
CREATE TABLE admin.white_label_settings (
    company_id         UUID PRIMARY KEY,   -- references client.companies(id) logically
    custom_subdomain   TEXT UNIQUE,        -- uniqueness enforced via DB constraint
    logo_url           TEXT,
    brand_primary_color TEXT,              -- hex string e.g. '#2563eb'
    brand_accent_color  TEXT,
    email_sender_domain TEXT,
    dns_verified        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_white_label_subdomain ON admin.white_label_settings (custom_subdomain)
    WHERE custom_subdomain IS NOT NULL;
```

`downgrade()` drops all three tables (indexes dropped automatically) and their seed data.

---

### AC2 — SQLAlchemy ORM Models

New file: `services/admin-api/src/admin_api/models/crawler.py`

```python
"""ORM models for crawler run history and schedule configuration (Story 12.12)."""
from __future__ import annotations

import uuid
from datetime import datetime

import sqlalchemy as sa
from sqlalchemy.orm import Mapped, mapped_column

from admin_api.models.base import Base

SCHEMA = "admin"


class CrawlerRun(Base):
    __tablename__ = "crawler_runs"
    __table_args__ = {"schema": SCHEMA}

    id: Mapped[uuid.UUID] = mapped_column(
        sa.UUID(as_uuid=True), primary_key=True, default=uuid.uuid4
    )
    crawler_type: Mapped[str] = mapped_column(sa.Text, nullable=False)
    status: Mapped[str] = mapped_column(sa.Text, nullable=False, default="pending")
    triggered_by: Mapped[str] = mapped_column(sa.Text, nullable=False, default="scheduled")
    triggered_at: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()
    )
    started_at: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True))
    completed_at: Mapped[datetime | None] = mapped_column(sa.DateTime(timezone=True))
    result_count: Mapped[int | None] = mapped_column(sa.Integer)
    error_detail: Mapped[str | None] = mapped_column(sa.Text)
    celery_task_id: Mapped[str | None] = mapped_column(sa.Text)


class CrawlerSchedule(Base):
    __tablename__ = "crawler_schedules"
    __table_args__ = {"schema": SCHEMA}

    crawler_type: Mapped[str] = mapped_column(sa.Text, primary_key=True)
    cron_minute: Mapped[str] = mapped_column(sa.Text, nullable=False, default="0")
    cron_hour: Mapped[str] = mapped_column(sa.Text, nullable=False, default="1")
    cron_day_of_week: Mapped[str] = mapped_column(sa.Text, nullable=False, default="*")
    cron_day_of_month: Mapped[str] = mapped_column(sa.Text, nullable=False, default="*")
    cron_month_of_year: Mapped[str] = mapped_column(sa.Text, nullable=False, default="*")
    enabled: Mapped[bool] = mapped_column(sa.Boolean, nullable=False, default=True)
    updated_at: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()
    )
    updated_by_admin_id: Mapped[uuid.UUID | None] = mapped_column(sa.UUID(as_uuid=True))
```

New file: `services/admin-api/src/admin_api/models/white_label.py`

```python
"""ORM model for per-tenant white-label branding settings (Story 12.12)."""
from __future__ import annotations

import uuid
from datetime import datetime

import sqlalchemy as sa
from sqlalchemy.orm import Mapped, mapped_column

from admin_api.models.base import Base

SCHEMA = "admin"


class WhiteLabelSettings(Base):
    __tablename__ = "white_label_settings"
    __table_args__ = {"schema": SCHEMA}

    company_id: Mapped[uuid.UUID] = mapped_column(
        sa.UUID(as_uuid=True), primary_key=True
    )
    custom_subdomain: Mapped[str | None] = mapped_column(sa.Text, unique=True)
    logo_url: Mapped[str | None] = mapped_column(sa.Text)
    brand_primary_color: Mapped[str | None] = mapped_column(sa.Text)
    brand_accent_color: Mapped[str | None] = mapped_column(sa.Text)
    email_sender_domain: Mapped[str | None] = mapped_column(sa.Text)
    dns_verified: Mapped[bool] = mapped_column(sa.Boolean, nullable=False, default=False)
    created_at: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), nullable=False, server_default=sa.func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        sa.DateTime(timezone=True), nullable=False,
        server_default=sa.func.now(), onupdate=sa.func.now()
    )
```

Add both models to `services/admin-api/src/admin_api/models/__init__.py`:

```python
from admin_api.models.crawler import CrawlerRun, CrawlerSchedule
from admin_api.models.white_label import WhiteLabelSettings
```

(Add these imports alongside the existing ones.)

---

### AC3 — Pydantic Schemas

New file: `services/admin-api/src/admin_api/schemas/crawler.py`

```python
"""Pydantic schemas for crawler management endpoints (Story 12.12)."""
from __future__ import annotations

from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, Field


# ---------------------------------------------------------------------------
# Crawler runs
# ---------------------------------------------------------------------------

class CrawlerRunSummary(BaseModel):
    id: UUID
    crawler_type: str
    status: str
    triggered_by: str
    triggered_at: datetime
    started_at: datetime | None
    completed_at: datetime | None
    result_count: int | None

    model_config = {"from_attributes": True}


class CrawlerRunDetail(CrawlerRunSummary):
    error_detail: str | None
    celery_task_id: str | None


class CrawlerRunListResponse(BaseModel):
    items: list[CrawlerRunSummary]
    total: int
    page: int
    page_size: int


# ---------------------------------------------------------------------------
# Crawler schedules
# ---------------------------------------------------------------------------

class CrawlerScheduleResponse(BaseModel):
    crawler_type: str
    cron_minute: str
    cron_hour: str
    cron_day_of_week: str
    cron_day_of_month: str
    cron_month_of_year: str
    enabled: bool
    updated_at: datetime
    updated_by_admin_id: UUID | None

    model_config = {"from_attributes": True}


class CrawlerScheduleUpdateRequest(BaseModel):
    cron_minute: str = Field(default="0", description="Cron minute field (e.g. '0', '*/15')")
    cron_hour: str = Field(default="1", description="Cron hour field (e.g. '1', '*/6')")
    cron_day_of_week: str = Field(default="*", description="Cron day-of-week (e.g. '*', '1')")
    cron_day_of_month: str = Field(default="*", description="Cron day-of-month (e.g. '*', '1')")
    cron_month_of_year: str = Field(default="*", description="Cron month field (e.g. '*', '1-6')")
    enabled: bool = Field(default=True, description="Whether the schedule is active")


# ---------------------------------------------------------------------------
# Crawler trigger
# ---------------------------------------------------------------------------

class CrawlerTriggerResponse(BaseModel):
    run_id: UUID
    crawler_type: str
    status: str          # always 'pending' at creation time
    triggered_at: datetime
```

New file: `services/admin-api/src/admin_api/schemas/white_label.py`

```python
"""Pydantic schemas for white-label configuration endpoints (Story 12.12)."""
from __future__ import annotations

import re
from datetime import datetime
from uuid import UUID

from pydantic import BaseModel, Field, field_validator


HEX_COLOR_RE = re.compile(r"^#[0-9a-fA-F]{6}$")


class WhiteLabelResponse(BaseModel):
    company_id: UUID
    custom_subdomain: str | None
    logo_url: str | None
    brand_primary_color: str | None
    brand_accent_color: str | None
    email_sender_domain: str | None
    dns_verified: bool
    updated_at: datetime

    model_config = {"from_attributes": True}


class WhiteLabelUpdateRequest(BaseModel):
    custom_subdomain: str | None = Field(
        default=None,
        description="Lowercase alphanumeric subdomain (e.g. 'acme'). Must be unique.",
        pattern=r"^[a-z0-9][a-z0-9\-]{0,61}[a-z0-9]$",
    )
    logo_url: str | None = Field(default=None, max_length=2048)
    brand_primary_color: str | None = Field(
        default=None, description="Hex color string e.g. '#2563eb'"
    )
    brand_accent_color: str | None = Field(
        default=None, description="Hex color string e.g. '#7c3aed'"
    )
    email_sender_domain: str | None = Field(default=None, max_length=255)

    @field_validator("brand_primary_color", "brand_accent_color", mode="before")
    @classmethod
    def validate_hex_color(cls, v: str | None) -> str | None:
        if v is not None and not HEX_COLOR_RE.match(v):
            raise ValueError("Color must be a valid 6-digit hex string e.g. '#2563eb'")
        return v
```

---

### AC4 — Crawler Service

New file: `services/admin-api/src/admin_api/services/crawler_service.py`

```python
"""Service layer for crawler management (Story 12.12)."""
from __future__ import annotations

import uuid
from datetime import datetime, timezone

import sqlalchemy as sa
from fastapi import HTTPException
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.models.crawler import CrawlerRun, CrawlerSchedule
from admin_api.schemas.crawler import (
    CrawlerRunDetail,
    CrawlerRunListResponse,
    CrawlerRunSummary,
    CrawlerScheduleResponse,
    CrawlerScheduleUpdateRequest,
    CrawlerTriggerResponse,
)


# Known crawler types — used for validation
KNOWN_CRAWLER_TYPES = frozenset({"ted", "find_a_tender", "merps", "grants_gov"})


async def list_crawler_runs(
    session: AsyncSession,
    crawler_type: str | None,
    status: str | None,
    page: int,
    page_size: int,
) -> CrawlerRunListResponse:
    """Return paginated crawler run history, optionally filtered by type/status."""
    filters = []
    if crawler_type:
        filters.append(CrawlerRun.crawler_type == crawler_type)
    if status:
        filters.append(CrawlerRun.status == status)

    count_q = sa.select(sa.func.count()).select_from(CrawlerRun)
    if filters:
        count_q = count_q.where(*filters)
    total = (await session.execute(count_q)).scalar_one()

    rows_q = (
        sa.select(CrawlerRun)
        .where(*filters)
        .order_by(CrawlerRun.triggered_at.desc())
        .limit(page_size)
        .offset((page - 1) * page_size)
    )
    rows = (await session.execute(rows_q)).scalars().all()
    return CrawlerRunListResponse(
        items=[CrawlerRunSummary.model_validate(r) for r in rows],
        total=total,
        page=page,
        page_size=page_size,
    )


async def get_crawler_run(session: AsyncSession, run_id: uuid.UUID) -> CrawlerRunDetail:
    """Return full detail for a single crawler run by ID."""
    row = await session.get(CrawlerRun, run_id)
    if row is None:
        raise HTTPException(status_code=404, detail=f"Crawler run {run_id} not found")
    return CrawlerRunDetail.model_validate(row)


async def get_crawler_schedule(
    session: AsyncSession, crawler_type: str
) -> CrawlerScheduleResponse:
    """Return schedule config for a crawler type."""
    _validate_crawler_type(crawler_type)
    row = await session.get(CrawlerSchedule, crawler_type)
    if row is None:
        raise HTTPException(
            status_code=404,
            detail=f"No schedule found for crawler type '{crawler_type}'",
        )
    return CrawlerScheduleResponse.model_validate(row)


async def update_crawler_schedule(
    session: AsyncSession,
    crawler_type: str,
    body: CrawlerScheduleUpdateRequest,
    admin_id: uuid.UUID,
) -> CrawlerScheduleResponse:
    """Update the Celery Beat schedule for a crawler type (persists to DB)."""
    _validate_crawler_type(crawler_type)
    row = await session.get(CrawlerSchedule, crawler_type)
    if row is None:
        # Auto-create if missing (allows seeding new crawler types via API)
        row = CrawlerSchedule(crawler_type=crawler_type)
        session.add(row)

    row.cron_minute = body.cron_minute
    row.cron_hour = body.cron_hour
    row.cron_day_of_week = body.cron_day_of_week
    row.cron_day_of_month = body.cron_day_of_month
    row.cron_month_of_year = body.cron_month_of_year
    row.enabled = body.enabled
    row.updated_at = datetime.now(tz=timezone.utc)
    row.updated_by_admin_id = admin_id

    await session.flush()
    await session.refresh(row)
    return CrawlerScheduleResponse.model_validate(row)


async def trigger_crawler_run(
    session: AsyncSession, crawler_type: str
) -> CrawlerTriggerResponse:
    """Create a crawler_runs entry (pending) and dispatch Celery task.

    Returns run_id for status polling via GET /admin/crawlers/runs/{run_id}.
    """
    _validate_crawler_type(crawler_type)

    run = CrawlerRun(
        id=uuid.uuid4(),
        crawler_type=crawler_type,
        status="pending",
        triggered_by="manual",
        triggered_at=datetime.now(tz=timezone.utc),
    )
    session.add(run)
    await session.flush()

    # Dispatch Celery task to data-pipeline worker without importing its code.
    # Task name follows data-pipeline convention: data_pipeline.tasks.crawl.<type>
    task_name = f"data_pipeline.tasks.crawl.run_{crawler_type}_crawler"
    try:
        from admin_api.worker import celery_app
        result = celery_app.send_task(
            task_name,
            kwargs={"run_id": str(run.id)},
            queue="crawler",
        )
        run.celery_task_id = result.id
    except Exception:
        # Non-fatal: run entry is created; task dispatch failure is logged but
        # does not roll back the run record (ops can retry manually).
        run.celery_task_id = None

    await session.flush()
    return CrawlerTriggerResponse(
        run_id=run.id,
        crawler_type=run.crawler_type,
        status=run.status,
        triggered_at=run.triggered_at,
    )


def _validate_crawler_type(crawler_type: str) -> None:
    if crawler_type not in KNOWN_CRAWLER_TYPES:
        raise HTTPException(
            status_code=400,
            detail=f"Unknown crawler type '{crawler_type}'. "
                   f"Valid types: {sorted(KNOWN_CRAWLER_TYPES)}",
        )
```

---

### AC5 — White-Label Service

New file: `services/admin-api/src/admin_api/services/white_label_service.py`

```python
"""Service layer for white-label configuration (Story 12.12)."""
from __future__ import annotations

import asyncio
import socket
import uuid

import sqlalchemy as sa
from fastapi import HTTPException
from sqlalchemy.exc import IntegrityError
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.models.white_label import WhiteLabelSettings
from admin_api.schemas.white_label import WhiteLabelResponse, WhiteLabelUpdateRequest


async def get_white_label(session: AsyncSession, company_id: uuid.UUID) -> WhiteLabelResponse:
    """Return white-label settings for a company (404 if none configured yet)."""
    row = await session.get(WhiteLabelSettings, company_id)
    if row is None:
        raise HTTPException(
            status_code=404,
            detail=f"No white-label settings found for company {company_id}",
        )
    return WhiteLabelResponse.model_validate(row)


async def upsert_white_label(
    session: AsyncSession,
    company_id: uuid.UUID,
    body: WhiteLabelUpdateRequest,
) -> WhiteLabelResponse:
    """Create or update white-label settings for a company.

    Validates subdomain uniqueness (409 on conflict) and resets dns_verified
    flag when custom_subdomain changes.
    """
    row = await session.get(WhiteLabelSettings, company_id)

    if row is None:
        row = WhiteLabelSettings(company_id=company_id)
        session.add(row)

    # Track if subdomain is changing so we reset dns_verified
    subdomain_changed = (body.custom_subdomain is not None) and (
        body.custom_subdomain != row.custom_subdomain
    )

    if body.custom_subdomain is not None:
        row.custom_subdomain = body.custom_subdomain
    if body.logo_url is not None:
        row.logo_url = body.logo_url
    if body.brand_primary_color is not None:
        row.brand_primary_color = body.brand_primary_color
    if body.brand_accent_color is not None:
        row.brand_accent_color = body.brand_accent_color
    if body.email_sender_domain is not None:
        row.email_sender_domain = body.email_sender_domain

    if subdomain_changed:
        row.dns_verified = False
        # Perform async DNS readiness check
        if body.custom_subdomain:
            dns_ok = await _check_subdomain_dns(body.custom_subdomain)
            row.dns_verified = dns_ok

    try:
        await session.flush()
    except IntegrityError as exc:
        await session.rollback()
        if "white_label_settings_custom_subdomain_key" in str(exc) or \
           "idx_white_label_subdomain" in str(exc):
            raise HTTPException(
                status_code=409,
                detail=f"Subdomain '{body.custom_subdomain}' is already in use by another tenant.",
            )
        raise

    await session.refresh(row)
    return WhiteLabelResponse.model_validate(row)


async def _check_subdomain_dns(subdomain: str) -> bool:
    """Non-blocking DNS readiness check.

    Attempts to resolve <subdomain>.eusolicit.com CNAME/A record.
    Returns True if resolvable, False if NXDOMAIN or timeout.
    This is best-effort — actual DNS propagation may lag minutes to hours.
    """
    fqdn = f"{subdomain}.eusolicit.com"
    loop = asyncio.get_event_loop()
    try:
        await asyncio.wait_for(
            loop.run_in_executor(None, socket.getaddrinfo, fqdn, None),
            timeout=3.0,
        )
        return True
    except (OSError, asyncio.TimeoutError):
        return False
```

---

### AC6 — Crawlers Router

New file: `services/admin-api/src/admin_api/api/v1/crawlers.py`

```python
"""Admin crawler management endpoints — /api/v1/admin/crawlers/* (Story 12.12)."""
from __future__ import annotations

from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.core.security import AdminUser, get_admin_user
from admin_api.dependencies import get_db_session
from admin_api.schemas.crawler import (
    CrawlerRunDetail,
    CrawlerRunListResponse,
    CrawlerScheduleResponse,
    CrawlerScheduleUpdateRequest,
    CrawlerTriggerResponse,
)
from admin_api.services import crawler_service

router = APIRouter(prefix="/admin/crawlers", tags=["admin-crawlers"])

_Admin = Annotated[AdminUser, Depends(get_admin_user)]
_Session = Annotated[AsyncSession, Depends(get_db_session)]


@router.get("/runs", response_model=CrawlerRunListResponse)
async def list_crawler_runs(
    admin: _Admin,
    session: _Session,
    crawler_type: str | None = Query(default=None, description="Filter by crawler type"),
    status: str | None = Query(
        default=None, description="Filter by status: pending|running|completed|failed"
    ),
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=20, ge=1, le=100),
) -> CrawlerRunListResponse:
    """Return paginated crawler run history (AC4)."""
    return await crawler_service.list_crawler_runs(
        session, crawler_type, status, page, page_size
    )


@router.get("/runs/{run_id}", response_model=CrawlerRunDetail)
async def get_crawler_run(
    run_id: UUID,
    admin: _Admin,
    session: _Session,
) -> CrawlerRunDetail:
    """Return status, result counts, and error details for a single run (AC4)."""
    return await crawler_service.get_crawler_run(session, run_id)


@router.get("/schedule/{crawler_type}", response_model=CrawlerScheduleResponse)
async def get_crawler_schedule(
    crawler_type: str,
    admin: _Admin,
    session: _Session,
) -> CrawlerScheduleResponse:
    """Return current Celery Beat schedule config for a crawler type (AC4)."""
    return await crawler_service.get_crawler_schedule(session, crawler_type)


@router.put("/schedule/{crawler_type}", response_model=CrawlerScheduleResponse)
async def update_crawler_schedule(
    crawler_type: str,
    body: CrawlerScheduleUpdateRequest,
    admin: _Admin,
    session: _Session,
) -> CrawlerScheduleResponse:
    """Update Celery Beat schedule for a crawler type (AC4)."""
    return await crawler_service.update_crawler_schedule(
        session, crawler_type, body, admin.admin_id
    )


@router.post("/trigger/{crawler_type}", response_model=CrawlerTriggerResponse, status_code=202)
async def trigger_crawler_run(
    crawler_type: str,
    admin: _Admin,
    session: _Session,
) -> CrawlerTriggerResponse:
    """Trigger a manual crawl run; returns run_id for status polling (AC4)."""
    return await crawler_service.trigger_crawler_run(session, crawler_type)
```

---

### AC7 — White-Label Router

New file: `services/admin-api/src/admin_api/api/v1/white_label.py`

```python
"""Admin white-label configuration endpoints — /api/v1/admin/tenants/{id}/white-label (Story 12.12)."""
from __future__ import annotations

from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.core.security import AdminUser, get_admin_user
from admin_api.dependencies import get_db_session
from admin_api.schemas.white_label import WhiteLabelResponse, WhiteLabelUpdateRequest
from admin_api.services import white_label_service

# NOTE: prefix matches /admin/tenants so both the tenants router (12.11)
# and this white-label router can share the path root without conflict.
# FastAPI routes are matched in registration order — tenants_v1 router is
# registered first in main.py; white_label_v1 registered after.
router = APIRouter(prefix="/admin/tenants", tags=["admin-white-label"])

_Admin = Annotated[AdminUser, Depends(get_admin_user)]
_Session = Annotated[AsyncSession, Depends(get_db_session)]


@router.get("/{company_id}/white-label", response_model=WhiteLabelResponse)
async def get_white_label(
    company_id: UUID,
    admin: _Admin,
    session: _Session,
) -> WhiteLabelResponse:
    """Return white-label settings for a company (404 if not yet configured) (AC5)."""
    return await white_label_service.get_white_label(session, company_id)


@router.put("/{company_id}/white-label", response_model=WhiteLabelResponse)
async def upsert_white_label(
    company_id: UUID,
    body: WhiteLabelUpdateRequest,
    admin: _Admin,
    session: _Session,
) -> WhiteLabelResponse:
    """Create or update white-label settings; validates subdomain uniqueness (AC5)."""
    return await white_label_service.upsert_white_label(session, company_id, body)
```

---

### AC8 — Register Routers in main.py

Update `services/admin-api/src/admin_api/main.py`:

```python
# Add imports
from admin_api.api.v1 import crawlers as crawlers_v1
from admin_api.api.v1 import white_label as white_label_v1

# Register after existing routers — order matters for path conflict resolution:
# white_label_v1 uses /admin/tenants/{company_id}/white-label which is more
# specific than the tenants_v1 route /admin/tenants/{company_id} (FastAPI
# matches on full path so no conflict; white-label MUST be registered AFTER
# tenants_v1 to preserve existing tenant route matching).
api_v1_router.include_router(tenants_v1.router)       # /admin/tenants/*      (Story 12.11)
api_v1_router.include_router(white_label_v1.router)   # /admin/tenants/*/white-label (Story 12.12)
api_v1_router.include_router(crawlers_v1.router)      # /admin/crawlers/*     (Story 12.12)
api_v1_router.include_router(fs_v1.router)
api_v1_router.include_router(fa_v1.router)
api_v1_router.include_router(cf_v1.router)
api_v1_router.include_router(rc_v1.router)
api_v1_router.include_router(ps_v1.router)
```

> **Router registration note:** FastAPI resolves `/{company_id}` (tenants_v1) and `/{company_id}/white-label` (white_label_v1) on different path lengths — there is no ambiguity. Both routers use prefix `/admin/tenants` and FastAPI matches the most-specific path first regardless of registration order. Registration order only matters for exact-same-path conflicts (none here). Still, register white_label_v1 immediately after tenants_v1 for clarity.

---

### AC9 — Tests

New file: `services/admin-api/tests/api/test_crawlers.py`

**Test classes to implement:**

- **`TestP0CrawlerAuth`** — P0 security tests:
  - All crawler endpoints return `401` for missing JWT
  - All crawler endpoints return `403` for standard user JWT (no `platform_admin` role)
  - All crawler endpoints return `403` for non-allowlisted IP (mock allowlist)
  - `GET /admin/crawlers/runs` with valid admin JWT → `200`

- **`TestListCrawlerRuns`** — P1 functional tests:
  - Empty runs → `{"items": [], "total": 0, "page": 1, "page_size": 20}`
  - Filter by `crawler_type=ted` → returns only `ted` runs
  - Filter by `status=failed` → returns only failed runs
  - Pagination: `page=2&page_size=1` with 2 seeded runs → returns second run

- **`TestGetCrawlerRun`** — P1:
  - Valid `run_id` → response includes all fields (`error_detail`, `celery_task_id`)
  - Non-existent `run_id` → `404`

- **`TestCrawlerSchedule`** — P1:
  - `GET /admin/crawlers/schedule/ted` → returns seeded schedule
  - `PUT /admin/crawlers/schedule/ted` with new cron values → response reflects update; GET confirms change
  - `GET /admin/crawlers/schedule/unknown_type` → `400` (invalid type)
  - `PUT` with `enabled=false` → schedule disabled

- **`TestTriggerCrawlerRun`** — P1:
  - `POST /admin/crawlers/trigger/ted` → `202`, `run_id` is valid UUID, `status="pending"`
  - Seeded run record exists after trigger (check via `GET /admin/crawlers/runs/{run_id}`)
  - `POST /admin/crawlers/trigger/unknown` → `400`

New file: `services/admin-api/tests/api/test_white_label.py`

**Test classes to implement:**

- **`TestP0WhiteLabelAuth`** — P0 security tests:
  - All white-label endpoints return `401` for missing JWT
  - All white-label endpoints return `403` for non-admin JWT
  - Valid admin JWT → `200`/`404` (not auth error)

- **`TestGetWhiteLabel`** — P1:
  - `GET /admin/tenants/{id}/white-label` with no settings → `404`
  - After upsert, `GET` returns all configured fields

- **`TestUpsertWhiteLabel`** — P1:
  - `PUT` with `custom_subdomain`, logo_url, colors → `200`, all fields in response
  - Second `PUT` updates existing settings (idempotent upsert)
  - `dns_verified` is set to `False` immediately when subdomain changes
  - Invalid hex color → `422`

- **`TestSubdomainUniqueness`** — P1/P2 (Risk R12.12):
  - Two companies with same `custom_subdomain` → second PUT returns `409`
  - Same company re-PUTting same subdomain → no conflict (idempotent)

**Testing patterns to follow:**
- Use `unittest.mock.AsyncMock` + `app.dependency_overrides[get_db_session]` to mock the DB session — do NOT require a real DB connection for unit tests
- Use `httpx.AsyncClient(app=app, base_url="http://test")` for async HTTP testing
- Mock `crawler_service.trigger_crawler_run` to avoid Celery dispatch in unit tests
- Mock `white_label_service._check_subdomain_dns` to return `True` or `False` deterministically
- Follow the exact env var setup pattern from `test_tenants.py` (`TestJwtEnvSetup` fixture)

---

## Tasks / Subtasks

### Review Findings

- [x] [Review][Patch] F1 — Add logging for Celery dispatch failure in `trigger_crawler_run` [crawler_service.py:140-143] — Comment says "is logged" but no `logger.exception()` call exists; runs silently stuck in pending forever with no operator visibility
- [x] [Review][Patch] F2 — Add cron field validation to `CrawlerScheduleUpdateRequest` [schemas/crawler.py:57-62] — Arbitrary strings accepted for all cron fields with no regex, no length limit; invalid expressions will crash Celery Beat when consumed
- [x] [Review][Patch] F3 — Replace deprecated `asyncio.get_event_loop()` with `asyncio.get_running_loop()` [white_label_service.py:90] — Deprecated since Python 3.10; raises RuntimeError in certain contexts on 3.12+
- [x] [Review][Patch] F4 — Make filter query consistent in `list_crawler_runs` [crawler_service.py:45-47] — `count_q` conditionally applies `.where()` only when filters exist, but `rows_q` always calls `.where(*filters)` unconditionally; inconsistency could break on future SQLAlchemy versions
- [x] [Review][Patch] F5 — Remove redundant column-level UNIQUE on `custom_subdomain` in migration [006_crawler_whitelabel_schema.py:76] — `TEXT UNIQUE` creates an implicit unique index AND explicit `CREATE UNIQUE INDEX` (partial, WHERE NOT NULL) is added on line 87; two indexes on same column; keep only the partial index
- [x] [Review][Patch] F6 — Add URL format validation on `logo_url` field [schemas/white_label.py:32] — No URL format check; accepts `javascript:` or `data:` URI schemes which are XSS vectors when rendered; add Pydantic `AnyHttpUrl` or URL regex
- [x] [Review][Patch] F7 — Explicitly set `updated_at` in `upsert_white_label` service [white_label_service.py:27-79] — Relies on ORM `onupdate=sa.func.now()` which may not fire on no-op updates; inconsistent with `crawler_service.py:102` which explicitly sets `updated_at`
- [x] [Review][Patch] F8 — Remove unused `import sqlalchemy as sa` in migration file [006_crawler_whitelabel_schema.py:9] — Ruff F401 lint error
- [x] [Review][Patch] F9 — Fix test RuntimeWarning from unawaited coroutine on mock `session.add()` [test_white_label.py: TestUpsertWhiteLabel::test_dns_check_true_sets_dns_verified] — AsyncMock session's `.add()` returns a coroutine that is never awaited; use `MagicMock()` for `.add()` or configure side_effect
- [x] [Review][Defer] D1 — Cannot clear/unset white-label fields once set [white_label_service.py:48-57] — deferred, design decision for PATCH-vs-PUT semantics
- [x] [Review][Defer] D2 — IntegrityError string matching fragility [white_label_service.py:70-71] — deferred, pre-existing pattern; consider ON CONFLICT in future refactor
- [x] [Review][Defer] D3 — No domain format validation on `email_sender_domain` [schemas/white_label.py:39] — deferred, pre-existing; relevant when email integration story is implemented
- [x] [Review][Defer] D4 — No validation on `crawler_type`/`status` query params in list endpoint [crawlers.py:31-33] — deferred, non-blocking; returns empty results on invalid filter values
- [x] [Review][Defer] D5 — Test mocks bypass real service logic for upsert tests [test_white_label.py] — deferred, integration tests needed separately for service-layer code paths
- [x] [Review][Defer] D6 — No rate limiting on manual crawler trigger endpoint [crawler_service.py:110-151] — deferred, infrastructure concern; add duplicate-run guard in future story
- [x] [Review][Defer] D7 — Empty body PUT creates phantom white-label row [white_label_service.py] — deferred, minor; add minimum-one-field validation in future

- [x] Task 1 — DB Migration 006 (AC1)
  - [x] 1.1 Create `services/admin-api/alembic/versions/006_crawler_whitelabel_schema.py`
  - [x] 1.2 Create `admin.crawler_runs` table with indexes
  - [x] 1.3 Create `admin.crawler_schedules` table with seeded default rows
  - [x] 1.4 Create `admin.white_label_settings` table with partial unique index
  - [x] 1.5 Write reversible `downgrade()` that drops all 3 tables
  - [x] 1.6 Verify: `alembic upgrade head` runs cleanly; `alembic downgrade -1` also clean

- [x] Task 2 — ORM Models (AC2)
  - [x] 2.1 Create `services/admin-api/src/admin_api/models/crawler.py` (`CrawlerRun`, `CrawlerSchedule`)
  - [x] 2.2 Create `services/admin-api/src/admin_api/models/white_label.py` (`WhiteLabelSettings`)
  - [x] 2.3 Add imports to `models/__init__.py`

- [x] Task 3 — Pydantic Schemas (AC3)
  - [x] 3.1 Create `services/admin-api/src/admin_api/schemas/crawler.py` (all 6 schema classes)
  - [x] 3.2 Create `services/admin-api/src/admin_api/schemas/white_label.py` (response + update request with validators)

- [x] Task 4 — Crawler Service (AC4)
  - [x] 4.1 Create `services/admin-api/src/admin_api/services/crawler_service.py`
  - [x] 4.2 Implement `list_crawler_runs()` with type/status filter, pagination, count query
  - [x] 4.3 Implement `get_crawler_run()` with 404 on missing run_id
  - [x] 4.4 Implement `get_crawler_schedule()` — fetch from DB, 404 if type not found
  - [x] 4.5 Implement `update_crawler_schedule()` — upsert in DB, stamp updated_at + admin_id
  - [x] 4.6 Implement `trigger_crawler_run()` — create pending run + `celery_app.send_task()` dispatch
  - [x] 4.7 Handle Celery dispatch failure gracefully (non-fatal, run record still created)

- [x] Task 5 — White-Label Service (AC5)
  - [x] 5.1 Create `services/admin-api/src/admin_api/services/white_label_service.py`
  - [x] 5.2 Implement `get_white_label()` — fetch by company_id, 404 if missing
  - [x] 5.3 Implement `upsert_white_label()` — create-or-update with subdomain uniqueness handling (409 on IntegrityError)
  - [x] 5.4 Implement `_check_subdomain_dns()` — async socket.getaddrinfo with 3s timeout
  - [x] 5.5 Reset `dns_verified=False` on subdomain change; set to True if DNS resolves

- [x] Task 6 — Crawlers Router (AC6)
  - [x] 6.1 Create `services/admin-api/src/admin_api/api/v1/crawlers.py`
  - [x] 6.2 Implement 5 endpoints wired to crawler_service + dependencies
  - [x] 6.3 Use `status_code=202` on the trigger POST endpoint

- [x] Task 7 — White-Label Router (AC7)
  - [x] 7.1 Create `services/admin-api/src/admin_api/api/v1/white_label.py`
  - [x] 7.2 Implement GET + PUT endpoints wired to white_label_service
  - [x] 7.3 Verify prefix `/admin/tenants` does not conflict with tenants_v1

- [x] Task 8 — Register routers (AC8)
  - [x] 8.1 Add `crawlers_v1` and `white_label_v1` imports to `main.py`
  - [x] 8.2 Register both routers in `api_v1_router` in correct order (see AC8 note)

- [x] Task 9 — Tests (AC9)
  - [x] 9.1 Create `services/admin-api/tests/api/test_crawlers.py` — P0 auth + P1 functional tests
  - [x] 9.2 Create `services/admin-api/tests/api/test_white_label.py` — P0 auth + P1 functional + uniqueness tests
  - [x] 9.3 Run full test suite: `pytest services/admin-api/tests/ -v` — zero regressions

## Dev Notes

### Service Architecture

- **Target service:** `services/admin-api` (port 8002), all endpoints under `/api/v1/admin/`.
- **Auth:** Already in place from S12.11 — `get_admin_user` dependency + `IPAllowlistMiddleware`. No changes needed.
- **DB session for ORM (admin schema):** Use `get_db_session()` from `dependencies.py` — this connects to `ADMIN_API_DATABASE_URL` and accesses the `admin` schema. The new tables are in `admin` schema, so no cross-schema session is needed.
- **NOT using `get_client_db_session()`** — white-label settings are stored in `admin` schema for this story, not `client` schema. Simplifies permissions.

### Celery Beat Schedule Management

The `admin.crawler_schedules` table is the **source of truth** for schedule configuration. The actual Celery Beat cron schedule running in the data-pipeline worker reads from this table at startup (data-pipeline implementation is out of scope for this story — that's handled in a future story when crawlers are fully implemented). For S12.12, the admin-api exposes read/write of this table; data-pipeline will consume it.

**Important:** The `PUT /admin/crawlers/schedule/{type}` endpoint only updates the DB table — it does **not** hot-reload a running Celery Beat worker. Live schedule updates require the data-pipeline worker to be restarted or to poll the DB for changes (out of scope). This is documented behavior for MVP.

### Crawler Trigger — Celery Dispatch Details

```python
from admin_api.worker import celery_app
result = celery_app.send_task(
    "data_pipeline.tasks.crawl.run_ted_crawler",   # task name in data-pipeline worker
    kwargs={"run_id": str(run.id)},
    queue="crawler",
)
run.celery_task_id = result.id
```

- Task names follow convention `data_pipeline.tasks.crawl.run_{crawler_type}_crawler`
- Queue `"crawler"` is the dedicated Celery queue for crawler tasks (data-pipeline worker listens on this queue)
- If Celery broker is unreachable, `send_task()` raises; catch the exception, log a warning, and return the run record with `celery_task_id=None` — the run stays in `status="pending"` indefinitely (ops can diagnose via Celery dashboard or logs)
- In **unit tests**, mock `celery_app.send_task` to return a `MagicMock` with `.id = "mock-celery-id"`

### White-Label Subdomain Uniqueness

Two mechanisms enforce uniqueness:
1. **DB-level:** `UNIQUE INDEX idx_white_label_subdomain` (partial, `WHERE custom_subdomain IS NOT NULL`) — raises `IntegrityError` on duplicate insert/update
2. **API-level:** `IntegrityError` caught in `upsert_white_label()` and converted to `HTTPException(409)`

The DB constraint is the authoritative guard (prevents race conditions). The API error message must include the conflicting subdomain for operator clarity.

### DNS Readiness Check

- `_check_subdomain_dns()` runs a blocking `socket.getaddrinfo()` in a thread pool executor via `asyncio.run_in_executor()` to avoid blocking the event loop
- Timeout: 3 seconds (fast fail — DNS propagation can take hours, so this is advisory only)
- `dns_verified` field is informational — it does not block the upsert; the settings are saved regardless
- When subdomain is cleared (`custom_subdomain=None`), `dns_verified` is set to `False` without triggering a DNS check

### Router Prefix Conflict Resolution

Both `tenants_v1` and `white_label_v1` routers use prefix `/admin/tenants`. FastAPI handles this correctly because the paths are distinct:
- Tenants: `GET/POST /admin/tenants/`, `GET /admin/tenants/{company_id}`, `POST /admin/tenants/{company_id}/tier-override`
- White-label: `GET/PUT /admin/tenants/{company_id}/white-label`

FastAPI matches routes on full path and does not short-circuit on prefix alone. No conflict exists. Register `white_label_v1` immediately after `tenants_v1` in `main.py` for visual grouping.

### ORM Base & Model Registration

Admin-api uses SQLAlchemy's declarative Base in `services/admin-api/src/admin_api/models/base.py`. New models inherit from `Base` and are automatically picked up by Alembic's `autogenerate` (though S12.12 uses manual migration authoring — do not rely on autogenerate). Import new models in `models/__init__.py` to ensure they are registered before `Base.metadata` is used.

### Column Verification Before Writing Queries

The `CrawlerRun` and `CrawlerSchedule` models are **new** (created by migration 006 in this story) — no pre-existing column naming surprises. `WhiteLabelSettings` is also new. No cross-referencing of existing tables is required for this story's ORM queries.

### Known Crawler Types

Defined in `crawler_service.KNOWN_CRAWLER_TYPES = frozenset({"ted", "find_a_tender", "merps", "grants_gov"})`. Any API call with a `crawler_type` not in this set returns HTTP 400. This is a constant for MVP; future stories may make this DB-driven.

### Pagination Pattern

Same envelope as S12.11:
```python
{"items": [...], "total": count, "page": page, "page_size": page_size}
```
Use explicit `sa.func.count()` select for the total count (not `.count()` on the ORM — breaks with async).

### Previous Story Context

Story 12.11 (Tenant Management) established:
- `IPAllowlistMiddleware` — registered globally in `main.py`; all admin-api routes are already IP-restricted. S12.12 does not need to re-add middleware.
- `get_admin_user` dependency — all new endpoints use this dependency unchanged.
- `get_db_session()` / `get_client_db_session()` in `dependencies.py` — S12.12 uses only `get_db_session()` (admin schema). No new session factory needed.
- `client_tables.py` — not used by S12.12 (all new tables are in admin schema).
- Deviation log: `shared.audit_log.timestamp` (not `created_at`), `users` linked via `company_memberships` — not relevant for S12.12 which uses only `admin` schema tables.

### test_design Reference

From `test-design-epic-12.md` — **P0 tests for S12.12** (must pass before review):

| Test | Expected |
|------|----------|
| Non-allowlisted IP → all crawler/white-label endpoints | 403 `{"detail": "Forbidden: your IP address..."}` |
| Standard user JWT (no `platform_admin` claim) → all endpoints | 403 `"Platform admin access required"` |
| Missing JWT → all endpoints | 401 `"Missing or invalid Authorization header"` |

From `atdd-checklist-e12-p0.md` — S12.12 maps to the **Admin API Access Control** P0 test group (4 tests in `e2e/specs/admin/admin-access-control.api.spec.ts` — currently RED/skipped). Implementing S12.12 completes the backend for those tests. After implementation, remove `test.skip()` from that spec file and verify GREEN phase.

**P1 tests for S12.12** (core functionality — 9 tests from test design):

| Test | Expected |
|------|----------|
| `GET /admin/crawlers/runs` | Paginated run history with status + timestamps |
| `GET/PUT /admin/crawlers/schedule/ted` | Read; update; re-read confirms change |
| `POST /admin/crawlers/trigger/ted` | 202, valid UUID `run_id`, `status="pending"` |
| `GET /admin/crawlers/runs/{run_id}` | All detail fields including error_detail |
| `GET/PUT /admin/tenants/{id}/white-label` | Read/update branding; persisted |
| Duplicate subdomain → PUT | 409 with message |

**P2 test for S12.12**:
- DNS readiness check returns informative error when subdomain CNAME unresolvable (mock `_check_subdomain_dns` to return `False`; assert `dns_verified=False` in response)

### Project Structure — New Files

```
services/admin-api/
├── alembic/versions/
│   └── 006_crawler_whitelabel_schema.py    ← new
├── src/admin_api/
│   ├── api/v1/
│   │   ├── crawlers.py                     ← new
│   │   └── white_label.py                  ← new
│   ├── models/
│   │   ├── crawler.py                      ← new
│   │   ├── white_label.py                  ← new
│   │   └── __init__.py                     ← modify (add imports)
│   ├── schemas/
│   │   ├── crawler.py                      ← new
│   │   └── white_label.py                  ← new
│   ├── services/
│   │   ├── crawler_service.py              ← new
│   │   └── white_label_service.py          ← new
│   └── main.py                             ← modify (register 2 new routers)
└── tests/
    └── api/
        ├── test_crawlers.py                ← new
        └── test_white_label.py             ← new
```

### References

- IP allowlist middleware (already active): `services/admin-api/src/admin_api/middleware/ip_allowlist.py`
- AdminUser + get_admin_user: `services/admin-api/src/admin_api/core/security.py`
- ORM Base: `services/admin-api/src/admin_api/models/base.py`
- Existing ORM model example: `services/admin-api/src/admin_api/models/platform_settings.py`
- Session factory (get_db_session): `services/admin-api/src/admin_api/dependencies.py`
- Worker / Celery app: `services/admin-api/src/admin_api/worker.py`
- Router pattern: `services/admin-api/src/admin_api/api/v1/platform_settings.py`
- Test fixture pattern: `services/admin-api/tests/api/test_tenants.py`
- Notification beat schedule (reference for cron patterns): `services/notification/src/notification/workers/beat_schedule.py`
- Epic-12 test design (S12.12 P0/P1 coverage): `eusolicit-docs/test-artifacts/test-design-epic-12.md#S12.12`
- P0 ATDD checklist (admin access control spec): `eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md#admin-access-control`
- Risk R12.2 (admin API privilege escalation): `eusolicit-docs/test-artifacts/test-design-epic-12.md#R12.2`
- Risk R12.12 (subdomain uniqueness): `eusolicit-docs/test-artifacts/test-design-epic-12.md#low-priority-risks`

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5)

### Debug Log References

- Fixed 3 test failures in `test_white_label.py` where `MagicMock` objects (from patched `upsert_white_label`) were returned directly as FastAPI responses — FastAPI's `response_model` validation requires proper Pydantic instances. Fixed by returning `WhiteLabelResponse` Pydantic instances from mocks.
- Ruff auto-fixed 24 linting issues across 6 files: import ordering (I001), `timezone.utc` → `datetime.UTC` (UP017), unused imports (F401), `asyncio.TimeoutError` → `TimeoutError` (UP041).

### Completion Notes List

- **Migration 006**: Created `006_crawler_whitelabel_schema.py` with 3 tables (`admin.crawler_runs`, `admin.crawler_schedules`, `admin.white_label_settings`), 3 indexes, 4 seeded schedule rows, and reversible `downgrade()`.
- **ORM Models**: `CrawlerRun`, `CrawlerSchedule`, `WhiteLabelSettings` following existing `PlatformSettings` pattern with `SCHEMA = "admin"` and `mapped_column` / `Mapped` typed annotations.
- **Pydantic Schemas**: 6 crawler schemas (summary, detail, list response, schedule response, schedule update, trigger response) + 2 white-label schemas (response + update request with hex color validator and subdomain pattern).
- **Crawler Service**: Full CRUD with `KNOWN_CRAWLER_TYPES` frozenset validation (400 on unknown), Celery dispatch via `celery_app.send_task()` with graceful failure fallback (run still created with `celery_task_id=None`).
- **White-Label Service**: Upsert with `IntegrityError` → 409 conversion, async DNS check via `asyncio.wait_for(loop.run_in_executor(...), timeout=3.0)`, `dns_verified` reset on subdomain change.
- **Routers**: Both registered after `tenants_v1` in `main.py`; no path conflicts (confirmed by test suite).
- **Tests**: 38 new tests (22 crawler + 16 white-label) all passing. 231 total tests pass with zero regressions.
- **ATDD E2E Note**: The `admin-access-control.api.spec.ts` ATDD tests remain skipped — those tests also cover S12.13 endpoints (audit-logs, analytics) not yet implemented, so removing `test.skip()` would cause RED phase on unimplemented endpoints. Left as-is pending S12.13 implementation.

### File List

- `eusolicit-app/services/admin-api/alembic/versions/006_crawler_whitelabel_schema.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/models/crawler.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/models/white_label.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/models/__init__.py` — modified (added CrawlerRun, CrawlerSchedule, WhiteLabelSettings imports)
- `eusolicit-app/services/admin-api/src/admin_api/schemas/crawler.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/schemas/white_label.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/services/crawler_service.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/services/white_label_service.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/api/v1/crawlers.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/api/v1/white_label.py` — new
- `eusolicit-app/services/admin-api/src/admin_api/main.py` — modified (registered crawlers_v1 and white_label_v1 routers)
- `eusolicit-app/services/admin-api/tests/api/test_crawlers.py` — new
- `eusolicit-app/services/admin-api/tests/api/test_white_label.py` — new
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — modified (status: in-progress → review)

### Change Log

- 2026-04-13: Story 12.12 implementation complete — crawler run history/schedule/trigger API and white-label branding API added to admin-api service (38 new tests, 231 total passing)
