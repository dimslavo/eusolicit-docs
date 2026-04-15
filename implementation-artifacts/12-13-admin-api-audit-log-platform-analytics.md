# Story 12.13: Admin API — Audit Log & Platform Analytics

Status: done

## Story

As a **platform operator (internal admin)**,
I want **VPN-restricted admin API endpoints to browse and export the platform audit log, and to view aggregate platform-level analytics (signup funnel, tier distribution, usage, and revenue metrics)**,
so that **operations and management staff can monitor security-relevant activity, satisfy compliance requirements, and track key business health indicators without direct database access**.

## Acceptance Criteria

### AC1 — DB Migration 007: SELECT Grant on `shared.audit_log`

New migration: `services/admin-api/alembic/versions/007_audit_log_select_grant.py`

```
revision = "007"
down_revision = "006"
```

`upgrade()` — requires superuser/schema owner (add `# REQUIRES SUPERUSER: yes` in header):

```sql
-- Grant admin_api_role SELECT access to shared.audit_log for audit log viewing
GRANT SELECT ON shared.audit_log TO admin_api_role;
```

`downgrade()`:

```sql
REVOKE SELECT ON shared.audit_log FROM admin_api_role;
```

> **Note:** Migration 005 already granted `INSERT ON shared.audit_log` and `SELECT/UPDATE` on client schema tables. This migration adds only the missing `SELECT` grant needed for audit log viewing endpoints. All existing grants remain in place.

---

### AC2 — Pydantic Schemas

New file: `services/admin-api/src/admin_api/schemas/audit_log.py`

```python
"""Pydantic schemas for audit log and platform analytics endpoints (Story 12.13)."""
from __future__ import annotations

from datetime import datetime
from uuid import UUID

from pydantic import BaseModel


# ---------------------------------------------------------------------------
# Audit log
# ---------------------------------------------------------------------------

class AuditLogEntry(BaseModel):
    id: UUID
    user_id: UUID | None
    action_type: str
    entity_type: str | None
    entity_id: UUID | None
    before: dict | None
    after: dict | None
    ip_address: str | None
    created_at: datetime

    model_config = {"from_attributes": True}


class AuditLogListResponse(BaseModel):
    items: list[AuditLogEntry]
    total: int
    page: int
    page_size: int


# ---------------------------------------------------------------------------
# Platform analytics
# ---------------------------------------------------------------------------

class FunnelStage(BaseModel):
    stage: str          # 'registered' | 'trial_started' | 'paid_conversion'
    count: int


class FunnelResponse(BaseModel):
    stages: list[FunnelStage]


class TierDistributionItem(BaseModel):
    plan: str           # 'free' | 'starter' | 'professional' | 'enterprise'
    count: int


class TierDistributionResponse(BaseModel):
    tiers: list[TierDistributionItem]
    total_companies: int


class UsageAggregate(BaseModel):
    metric_type: str    # 'ai_summaries' | 'proposal_drafts' | 'compliance_checks'
    total_consumed: float
    total_limit: float | None
    company_count: int  # number of companies consuming this metric type


class PlatformUsageResponse(BaseModel):
    period_month: str   # ISO month string e.g. '2026-04'
    metrics: list[UsageAggregate]


class RevenueMetrics(BaseModel):
    mrr_usd: float          # Monthly recurring revenue in USD
    growth_rate_pct: float  # MRR growth rate vs prior month (percent)
    churn_rate_pct: float   # Monthly churn rate (cancellations / active start of month)
    active_paid_companies: int
    new_paid_this_month: int
    churned_this_month: int


class RevenueResponse(BaseModel):
    metrics: RevenueMetrics
```

---

### AC3 — Audit Log Service

New file: `services/admin-api/src/admin_api/services/audit_log_service.py`

```python
"""Service layer for audit log viewing and CSV export (Story 12.13)."""
from __future__ import annotations

import csv
import io
from datetime import datetime
from uuid import UUID

import sqlalchemy as sa
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.models.client_tables import shared_audit_log
from admin_api.schemas.audit_log import AuditLogEntry, AuditLogListResponse


async def list_audit_logs(
    session: AsyncSession,
    user_id: UUID | None,
    action_type: str | None,
    entity_type: str | None,
    date_from: datetime | None,
    date_to: datetime | None,
    page: int,
    page_size: int,
) -> AuditLogListResponse:
    """Return paginated, filtered audit log entries from shared.audit_log."""
    filters = _build_filters(user_id, action_type, entity_type, date_from, date_to)

    count_q = sa.select(sa.func.count()).select_from(shared_audit_log)
    if filters:
        count_q = count_q.where(sa.and_(*filters))
    total = (await session.execute(count_q)).scalar_one()

    rows_q = (
        sa.select(shared_audit_log)
        .where(sa.and_(*filters) if filters else sa.true())
        .order_by(shared_audit_log.c.created_at.desc())
        .limit(page_size)
        .offset((page - 1) * page_size)
    )
    rows = (await session.execute(rows_q)).mappings().all()
    return AuditLogListResponse(
        items=[AuditLogEntry.model_validate(dict(r)) for r in rows],
        total=total,
        page=page,
        page_size=page_size,
    )


async def export_audit_logs_csv(
    session: AsyncSession,
    user_id: UUID | None,
    action_type: str | None,
    entity_type: str | None,
    date_from: datetime | None,
    date_to: datetime | None,
) -> str:
    """Return all matching audit log entries as a UTF-8 CSV string.

    Streams rows in-memory (acceptable for audit exports — typically < 100k rows
    per date range filter). For very large unfiltered exports, a server-side cursor
    can be added in a future refactor.
    """
    filters = _build_filters(user_id, action_type, entity_type, date_from, date_to)
    rows_q = (
        sa.select(shared_audit_log)
        .where(sa.and_(*filters) if filters else sa.true())
        .order_by(shared_audit_log.c.created_at.desc())
    )
    rows = (await session.execute(rows_q)).mappings().all()

    output = io.StringIO()
    writer = csv.writer(output)
    # Header — 7 columns matching AuditLogEntry fields (before/after serialised as JSON strings)
    writer.writerow([
        "id", "user_id", "action_type", "entity_type",
        "entity_id", "ip_address", "created_at",
    ])
    for row in rows:
        writer.writerow([
            row["id"], row["user_id"], row["action_type"], row["entity_type"],
            row["entity_id"], row["ip_address"], row["created_at"].isoformat(),
        ])
    return output.getvalue()


def _build_filters(
    user_id: UUID | None,
    action_type: str | None,
    entity_type: str | None,
    date_from: datetime | None,
    date_to: datetime | None,
) -> list:
    """Build SQLAlchemy filter list for audit log queries."""
    filters = []
    if user_id is not None:
        filters.append(shared_audit_log.c.user_id == user_id)
    if action_type is not None:
        filters.append(shared_audit_log.c.action_type == action_type)
    if entity_type is not None:
        filters.append(shared_audit_log.c.entity_type == entity_type)
    if date_from is not None:
        filters.append(shared_audit_log.c.created_at >= date_from)
    if date_to is not None:
        filters.append(shared_audit_log.c.created_at <= date_to)
    return filters
```

---

### AC4 — Platform Analytics Service

New file: `services/admin-api/src/admin_api/services/platform_analytics_service.py`

```python
"""Service layer for platform-level analytics (Story 12.13).

All queries run against the client schema via get_client_db_session().
Revenue calculations use hardcoded plan prices (MVP — no subscription_prices table).
"""
from __future__ import annotations

from datetime import datetime, timezone

import sqlalchemy as sa
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.models.client_tables import (
    client_companies,
    client_subscriptions,
    client_mv_usage_consumption,
    shared_audit_log,
)
from admin_api.schemas.audit_log import (
    FunnelResponse,
    FunnelStage,
    PlatformUsageResponse,
    RevenueMetrics,
    RevenueResponse,
    TierDistributionItem,
    TierDistributionResponse,
    UsageAggregate,
)

# Hardcoded monthly plan prices (USD) — update if pricing changes
PLAN_PRICES: dict[str, float] = {
    "free": 0.0,
    "starter": 29.0,
    "professional": 79.0,
    "enterprise": 299.0,
}
PAID_PLANS = frozenset({"starter", "professional", "enterprise"})


async def get_signup_funnel(session: AsyncSession) -> FunnelResponse:
    """Return signup funnel stage counts.

    Stages:
      registered      — total companies ever created (client.companies count)
      trial_started   — companies with subscriptions (any status)
      paid_conversion — companies with active paid subscription (plan != 'free', status='active')
    """
    registered_count = (
        await session.execute(sa.select(sa.func.count()).select_from(client_companies))
    ).scalar_one()

    trial_q = (
        sa.select(sa.func.count(sa.distinct(client_subscriptions.c.company_id)))
        .select_from(client_subscriptions)
    )
    trial_count = (await session.execute(trial_q)).scalar_one()

    paid_q = (
        sa.select(sa.func.count(sa.distinct(client_subscriptions.c.company_id)))
        .select_from(client_subscriptions)
        .where(
            sa.and_(
                client_subscriptions.c.plan.in_(list(PAID_PLANS)),
                client_subscriptions.c.status == "active",
            )
        )
    )
    paid_count = (await session.execute(paid_q)).scalar_one()

    return FunnelResponse(
        stages=[
            FunnelStage(stage="registered", count=registered_count),
            FunnelStage(stage="trial_started", count=trial_count),
            FunnelStage(stage="paid_conversion", count=paid_count),
        ]
    )


async def get_tier_distribution(session: AsyncSession) -> TierDistributionResponse:
    """Return count of active companies per subscription tier."""
    rows_q = (
        sa.select(
            client_subscriptions.c.plan,
            sa.func.count(sa.distinct(client_subscriptions.c.company_id)).label("count"),
        )
        .select_from(client_subscriptions)
        .where(client_subscriptions.c.status == "active")
        .group_by(client_subscriptions.c.plan)
        .order_by(client_subscriptions.c.plan)
    )
    rows = (await session.execute(rows_q)).all()

    tiers = [TierDistributionItem(plan=row.plan, count=row.count) for row in rows]
    total = sum(t.count for t in tiers)
    return TierDistributionResponse(tiers=tiers, total_companies=total)


async def get_platform_usage(session: AsyncSession) -> PlatformUsageResponse:
    """Return aggregate usage metrics across all tenants for the current billing period."""
    period_start = sa.func.date_trunc("month", sa.func.now())

    rows_q = (
        sa.select(
            client_mv_usage_consumption.c.metric_type,
            sa.func.sum(client_mv_usage_consumption.c.consumed).label("total_consumed"),
            sa.func.sum(client_mv_usage_consumption.c.limit_value).label("total_limit"),
            sa.func.count(
                sa.distinct(client_mv_usage_consumption.c.company_id)
            ).label("company_count"),
        )
        .select_from(client_mv_usage_consumption)
        .where(client_mv_usage_consumption.c.period_start == period_start)
        .group_by(client_mv_usage_consumption.c.metric_type)
    )
    rows = (await session.execute(rows_q)).all()

    now = datetime.now(tz=timezone.utc)
    period_month = f"{now.year}-{now.month:02d}"

    return PlatformUsageResponse(
        period_month=period_month,
        metrics=[
            UsageAggregate(
                metric_type=row.metric_type,
                total_consumed=float(row.total_consumed or 0),
                total_limit=float(row.total_limit) if row.total_limit is not None else None,
                company_count=row.company_count,
            )
            for row in rows
        ],
    )


async def get_revenue_metrics(session: AsyncSession) -> RevenueResponse:
    """Return MRR, growth rate, and churn rate.

    MRR: sum of PLAN_PRICES[plan] * company count for all active paid subscriptions.
    Churn: companies that had active paid subs at start of month and no longer do.
    Growth: estimated from tier_override audit log entries (plan upgrades this month).

    Note: This is MVP-grade computation. Phase 2 should introduce a dedicated
    revenue_events table for precise historical MRR tracking.
    """
    # Active paid companies for MRR
    paid_q = (
        sa.select(
            client_subscriptions.c.plan,
            sa.func.count(sa.distinct(client_subscriptions.c.company_id)).label("count"),
        )
        .select_from(client_subscriptions)
        .where(
            sa.and_(
                client_subscriptions.c.plan.in_(list(PAID_PLANS)),
                client_subscriptions.c.status == "active",
            )
        )
        .group_by(client_subscriptions.c.plan)
    )
    paid_rows = (await session.execute(paid_q)).all()

    mrr = sum(PLAN_PRICES.get(row.plan, 0.0) * row.count for row in paid_rows)
    active_paid = sum(row.count for row in paid_rows)

    # New paid companies this month (upgrades from free→paid via tier_override in audit log)
    month_start = sa.func.date_trunc("month", sa.func.now())
    new_paid_q = (
        sa.select(sa.func.count())
        .select_from(shared_audit_log)
        .where(
            sa.and_(
                shared_audit_log.c.action_type == "tier_override",
                shared_audit_log.c.created_at >= month_start,
                sa.cast(shared_audit_log.c.after["plan"], sa.Text).in_(list(PAID_PLANS)),
                sa.cast(shared_audit_log.c.before["plan"], sa.Text).not_in(list(PAID_PLANS)),
            )
        )
    )
    new_paid_this_month = (await session.execute(new_paid_q)).scalar_one() or 0

    # Churned: tier_override from paid→free this month
    churned_q = (
        sa.select(sa.func.count())
        .select_from(shared_audit_log)
        .where(
            sa.and_(
                shared_audit_log.c.action_type == "tier_override",
                shared_audit_log.c.created_at >= month_start,
                sa.cast(shared_audit_log.c.after["plan"], sa.Text).not_in(list(PAID_PLANS)),
                sa.cast(shared_audit_log.c.before["plan"], sa.Text).in_(list(PAID_PLANS)),
            )
        )
    )
    churned_this_month = (await session.execute(churned_q)).scalar_one() or 0

    # Churn rate = churned / (active_paid + churned) — avoid division by zero
    base = active_paid + churned_this_month
    churn_rate = (churned_this_month / base * 100) if base > 0 else 0.0

    # Growth rate: estimate prior month MRR from current MRR ± new/churned value
    # MVP approximation — replace with revenue_events table in Phase 2
    prior_mrr = mrr - (new_paid_this_month * 29.0) + (churned_this_month * 29.0)  # assume starter avg
    growth_rate = ((mrr - prior_mrr) / prior_mrr * 100) if prior_mrr > 0 else 0.0

    return RevenueResponse(
        metrics=RevenueMetrics(
            mrr_usd=round(mrr, 2),
            growth_rate_pct=round(growth_rate, 2),
            churn_rate_pct=round(churn_rate, 2),
            active_paid_companies=active_paid,
            new_paid_this_month=new_paid_this_month,
            churned_this_month=churned_this_month,
        )
    )
```

---

### AC5 — Audit Logs Router

New file: `services/admin-api/src/admin_api/api/v1/audit_logs.py`

```python
"""Admin audit log endpoints — /api/v1/admin/audit-logs/* (Story 12.13)."""
from __future__ import annotations

from datetime import datetime
from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, Query
from fastapi.responses import StreamingResponse
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.core.security import AdminUser, get_admin_user
from admin_api.dependencies import get_client_db_session
from admin_api.schemas.audit_log import AuditLogListResponse
from admin_api.services import audit_log_service

router = APIRouter(prefix="/admin/audit-logs", tags=["admin-audit-logs"])

_Admin = Annotated[AdminUser, Depends(get_admin_user)]
# Audit logs are in shared schema — read via client DB session (same PG instance)
_ClientSession = Annotated[AsyncSession, Depends(get_client_db_session)]


@router.get("", response_model=AuditLogListResponse)
async def list_audit_logs(
    admin: _Admin,
    session: _ClientSession,
    user_id: UUID | None = Query(default=None, description="Filter by user UUID"),
    action_type: str | None = Query(default=None, description="Filter by action type e.g. 'tier_override'"),
    entity_type: str | None = Query(default=None, description="Filter by entity type e.g. 'subscription'"),
    date_from: datetime | None = Query(default=None, description="ISO datetime lower bound (inclusive)"),
    date_to: datetime | None = Query(default=None, description="ISO datetime upper bound (inclusive)"),
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=50, ge=1, le=500),
) -> AuditLogListResponse:
    """Return paginated, filtered audit log entries (AC3)."""
    return await audit_log_service.list_audit_logs(
        session, user_id, action_type, entity_type, date_from, date_to, page, page_size
    )


@router.get("/export")
async def export_audit_logs(
    admin: _Admin,
    session: _ClientSession,
    user_id: UUID | None = Query(default=None),
    action_type: str | None = Query(default=None),
    entity_type: str | None = Query(default=None),
    date_from: datetime | None = Query(default=None),
    date_to: datetime | None = Query(default=None),
) -> StreamingResponse:
    """Return filtered audit log as a streaming CSV download (AC3).

    Content-Type: text/csv
    Content-Disposition: attachment; filename=audit-logs.csv
    """
    csv_content = await audit_log_service.export_audit_logs_csv(
        session, user_id, action_type, entity_type, date_from, date_to
    )

    def iter_csv():
        yield csv_content

    return StreamingResponse(
        iter_csv(),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=audit-logs.csv"},
    )
```

> **Route ordering note:** `/export` must be registered **before** any parameterised path on this router. Since all routes on this router use a flat prefix (`/admin/audit-logs` + `/export`), there is no path-conflict risk. FastAPI resolves `/export` as a literal path — not a UUID path param — so ordering is correct.

---

### AC6 — Platform Analytics Router

New file: `services/admin-api/src/admin_api/api/v1/platform_analytics.py`

```python
"""Admin platform analytics endpoints — /api/v1/admin/analytics/* (Story 12.13)."""
from __future__ import annotations

from typing import Annotated

from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.core.security import AdminUser, get_admin_user
from admin_api.dependencies import get_client_db_session
from admin_api.schemas.audit_log import (
    FunnelResponse,
    PlatformUsageResponse,
    RevenueResponse,
    TierDistributionResponse,
)
from admin_api.services import platform_analytics_service

router = APIRouter(prefix="/admin/analytics", tags=["admin-platform-analytics"])

_Admin = Annotated[AdminUser, Depends(get_admin_user)]
_ClientSession = Annotated[AsyncSession, Depends(get_client_db_session)]


@router.get("/funnel", response_model=FunnelResponse)
async def get_signup_funnel(
    admin: _Admin,
    session: _ClientSession,
) -> FunnelResponse:
    """Return signup funnel stage counts (registered → trial → paid) (AC4)."""
    return await platform_analytics_service.get_signup_funnel(session)


@router.get("/tiers", response_model=TierDistributionResponse)
async def get_tier_distribution(
    admin: _Admin,
    session: _ClientSession,
) -> TierDistributionResponse:
    """Return distribution of active companies per subscription tier (AC4)."""
    return await platform_analytics_service.get_tier_distribution(session)


@router.get("/usage", response_model=PlatformUsageResponse)
async def get_platform_usage(
    admin: _Admin,
    session: _ClientSession,
) -> PlatformUsageResponse:
    """Return aggregate usage metrics across all tenants for the current month (AC4)."""
    return await platform_analytics_service.get_platform_usage(session)


@router.get("/revenue", response_model=RevenueResponse)
async def get_revenue_metrics(
    admin: _Admin,
    session: _ClientSession,
) -> RevenueResponse:
    """Return MRR, growth rate, and churn rate metrics (AC4)."""
    return await platform_analytics_service.get_revenue_metrics(session)
```

---

### AC7 — Register Routers in main.py

Update `services/admin-api/src/admin_api/main.py` to add imports and register the two new routers:

```python
# Add imports alongside existing admin router imports
from admin_api.api.v1 import audit_logs as audit_logs_v1
from admin_api.api.v1 import platform_analytics as platform_analytics_v1

# Register in api_v1_router — after existing admin routers, before framework routers:
api_v1_router.include_router(tenants_v1.router)              # /admin/tenants/*         (Story 12.11)
api_v1_router.include_router(white_label_v1.router)          # /admin/tenants/*/white-label (Story 12.12)
api_v1_router.include_router(crawlers_v1.router)             # /admin/crawlers/*        (Story 12.12)
api_v1_router.include_router(audit_logs_v1.router)           # /admin/audit-logs/*      (Story 12.13)
api_v1_router.include_router(platform_analytics_v1.router)   # /admin/analytics/*       (Story 12.13)
api_v1_router.include_router(fs_v1.router)
api_v1_router.include_router(fa_v1.router)
api_v1_router.include_router(cf_v1.router)
api_v1_router.include_router(rc_v1.router)
api_v1_router.include_router(ps_v1.router)
```

---

### AC8 — Tests

New file: `services/admin-api/tests/api/test_audit_logs.py`

**Test classes:**

- **`TestP0AuditLogAuth`** — P0 security tests (mirroring ATDD checklist `admin-access-control.api.spec.ts`):
  - All `/admin/audit-logs` endpoints return `401` for missing JWT
  - All endpoints return `403` for standard user JWT (no `platform_admin` role)
  - All endpoints return `403` for non-allowlisted IP (mock `IPAllowlistMiddleware` to reject)
  - Valid admin JWT from allowlisted IP → `200` on `GET /admin/audit-logs`

- **`TestListAuditLogs`** — P1 functional tests:
  - No filters applied → returns paginated response with `items`, `total`, `page`, `page_size`
  - Filter by `user_id` → only entries for that user returned
  - Filter by `action_type=tier_override` → only tier_override entries returned
  - Filter by `entity_type=subscription` → only subscription entries returned
  - Filter by `date_from` + `date_to` → only entries in range returned
  - Pagination: `page=2&page_size=1` with 2 seeded entries → second entry returned
  - Empty result (no matching entries) → `{"items": [], "total": 0, ...}` (not 500)

- **`TestExportAuditLogs`** — P1 functional tests:
  - `GET /admin/audit-logs/export` returns `Content-Type: text/csv`
  - Response includes 7-column CSV header: `id,user_id,action_type,entity_type,entity_id,ip_address,created_at`
  - Filters applied to export match same filter logic as list endpoint
  - Empty result set → CSV with header only (no body rows), `200` response

New file: `services/admin-api/tests/api/test_platform_analytics.py`

**Test classes:**

- **`TestP0PlatformAnalyticsAuth`** — P0 security tests:
  - All `/admin/analytics/*` endpoints return `401` for missing JWT
  - All endpoints return `403` for non-admin JWT
  - All endpoints return `403` for non-allowlisted IP
  - Valid admin JWT → `200` on each analytics endpoint

- **`TestSignupFunnel`** — P1:
  - `GET /admin/analytics/funnel` returns response with `stages` list of 3 items
  - Each stage has `stage` (string) and `count` (integer ≥ 0) fields
  - All 3 stage names present: `registered`, `trial_started`, `paid_conversion`
  - Counts are non-negative integers; `paid_conversion` ≤ `trial_started` ≤ `registered`

- **`TestTierDistribution`** — P1:
  - `GET /admin/analytics/tiers` returns `tiers` list and `total_companies` integer
  - `total_companies` equals sum of all tier counts
  - Empty database → `tiers=[]`, `total_companies=0`

- **`TestPlatformUsage`** — P1/P2 (epic spec lists this as P2 but implement it here):
  - `GET /admin/analytics/usage` returns `period_month` and `metrics` list
  - Each metric has `metric_type`, `total_consumed`, `total_limit`, `company_count`
  - Returns `200` even when no usage data for current period (empty `metrics` list)

- **`TestRevenueMetrics`** — P1:
  - `GET /admin/analytics/revenue` returns `metrics` object
  - All 6 fields present: `mrr_usd`, `growth_rate_pct`, `churn_rate_pct`, `active_paid_companies`, `new_paid_this_month`, `churned_this_month`
  - All values are numeric (float or int); no nulls

**Testing patterns (follow 12-12 established conventions):**
- Use `unittest.mock.AsyncMock` + `app.dependency_overrides[get_client_db_session]` to mock DB sessions
- Use `app.dependency_overrides[get_admin_user]` to inject mock `AdminUser`
- Use `httpx.AsyncClient(app=app, base_url="http://test")` for async HTTP testing
- Mock IP allowlist by overriding `get_client_ip` or patching `IPAllowlistMiddleware` `__call__`
- Follow `TestJwtEnvSetup` fixture pattern from `test_tenants.py` for JWT env var setup
- Do NOT require a real DB connection for unit tests — mock all session interactions

---

## Tasks / Subtasks

- [ ] Task 1 — DB Migration 007 (AC1)
  - [ ] 1.1 Create `services/admin-api/alembic/versions/007_audit_log_select_grant.py`
  - [ ] 1.2 Add `GRANT SELECT ON shared.audit_log TO admin_api_role` in `upgrade()`
  - [ ] 1.3 Add `REVOKE SELECT ON shared.audit_log FROM admin_api_role` in `downgrade()`
  - [ ] 1.4 Add `# REQUIRES SUPERUSER: yes` header comment
  - [ ] 1.5 Verify: `alembic upgrade head` and `alembic downgrade -1` run cleanly

- [ ] Task 2 — Pydantic Schemas (AC2)
  - [ ] 2.1 Create `services/admin-api/src/admin_api/schemas/audit_log.py`
  - [ ] 2.2 Define `AuditLogEntry`, `AuditLogListResponse` for audit log endpoints
  - [ ] 2.3 Define `FunnelResponse`, `TierDistributionResponse`, `PlatformUsageResponse`, `RevenueResponse`, `RevenueMetrics` for analytics endpoints
  - [ ] 2.4 Verify all schemas pass `mypy` with strict mode

- [ ] Task 3 — Audit Log Service (AC3)
  - [ ] 3.1 Create `services/admin-api/src/admin_api/services/audit_log_service.py`
  - [ ] 3.2 Implement `list_audit_logs()` — 5-filter, paginated, ordered by `created_at DESC`
  - [ ] 3.3 Implement `export_audit_logs_csv()` — returns full matching set as UTF-8 CSV string
  - [ ] 3.4 Extract `_build_filters()` helper shared between list and export (no duplication)
  - [ ] 3.5 Verify `shared_audit_log` SA Core table in `client_tables.py` has all required columns (id, user_id, action_type, entity_type, entity_id, before, after, ip_address, created_at)

- [ ] Task 4 — Platform Analytics Service (AC4)
  - [ ] 4.1 Create `services/admin-api/src/admin_api/services/platform_analytics_service.py`
  - [ ] 4.2 Implement `get_signup_funnel()` — 3-stage count query against `client.companies` + `client.subscriptions`
  - [ ] 4.3 Implement `get_tier_distribution()` — group-by plan count of active subscriptions
  - [ ] 4.4 Implement `get_platform_usage()` — aggregate `client.mv_usage_consumption` for current month
  - [ ] 4.5 Implement `get_revenue_metrics()` — MRR from plan×count, churn/growth from `shared.audit_log` tier_override entries
  - [ ] 4.6 Add `PLAN_PRICES` constant dict and `PAID_PLANS` frozenset at module top
  - [ ] 4.7 Add `# Phase 2 note` comment on revenue growth_rate computation (MVP approximation)

- [ ] Task 5 — Audit Logs Router (AC5)
  - [ ] 5.1 Create `services/admin-api/src/admin_api/api/v1/audit_logs.py`
  - [ ] 5.2 Implement `GET /admin/audit-logs` endpoint with all 5 filter query params + pagination
  - [ ] 5.3 Implement `GET /admin/audit-logs/export` endpoint returning `StreamingResponse` with `media_type="text/csv"`
  - [ ] 5.4 Verify `/export` literal route is defined before any parameterised sibling routes (no conflict here — safe)

- [ ] Task 6 — Platform Analytics Router (AC6)
  - [ ] 6.1 Create `services/admin-api/src/admin_api/api/v1/platform_analytics.py`
  - [ ] 6.2 Implement 4 endpoints: `/admin/analytics/funnel`, `/admin/analytics/tiers`, `/admin/analytics/usage`, `/admin/analytics/revenue`
  - [ ] 6.3 All endpoints use `_ClientSession` (client DB session — reads client + shared schemas)
  - [ ] 6.4 All endpoints require `_Admin` dependency (auth + IP allowlist already enforced)

- [ ] Task 7 — Register Routers (AC7)
  - [ ] 7.1 Add `audit_logs_v1` and `platform_analytics_v1` imports to `main.py`
  - [ ] 7.2 Register both routers in `api_v1_router` after `crawlers_v1` (see AC7 for exact order)

- [ ] Task 8 — Tests (AC8)
  - [ ] 8.1 Create `services/admin-api/tests/api/test_audit_logs.py` — P0 auth + P1 functional (list + export)
  - [ ] 8.2 Create `services/admin-api/tests/api/test_platform_analytics.py` — P0 auth + P1 functional (funnel, tiers, usage, revenue)
  - [ ] 8.3 Run `pytest services/admin-api/tests/ -v` — zero regressions against existing 12.11/12.12 tests

---

## Dev Notes

### Service Architecture

- **Target service:** `services/admin-api` (port 8002). All new endpoints under `/api/v1/admin/`.
- **Auth & IP restriction:** Already enforced globally by `IPAllowlistMiddleware` (registered in `main.py` S12.11) and `get_admin_user` dependency (`core/security.py` S12.11). **No new auth code needed.** Just use `_Admin = Annotated[AdminUser, Depends(get_admin_user)]` in both routers.
- **DB session:** Both audit log AND platform analytics use `get_client_db_session()` (from `dependencies.py`, established S12.11). This session has cross-schema access to `client.*` and `shared.*` (grants in migration 005 + new migration 007).
- **Do NOT use `get_db_session()`** — that connects to admin schema only. Audit log is in `shared` schema; analytics data is in `client` schema.

### Data Sources

| Endpoint | Schema | Tables Used |
|---|---|---|
| `GET /admin/audit-logs` | `shared` | `shared.audit_log` |
| `GET /admin/audit-logs/export` | `shared` | `shared.audit_log` |
| `GET /admin/analytics/funnel` | `client` | `client.companies`, `client.subscriptions` |
| `GET /admin/analytics/tiers` | `client` | `client.subscriptions` |
| `GET /admin/analytics/usage` | `client` | `client.mv_usage_consumption` |
| `GET /admin/analytics/revenue` | `client` + `shared` | `client.subscriptions`, `shared.audit_log` |

### `shared_audit_log` SA Core Table — Already Defined

The `shared_audit_log` SA Core Table object is defined in `services/admin-api/src/admin_api/models/client_tables.py` (established in Story 12.11 for tier-override INSERT). It has all columns needed for SELECT queries:

```python
shared_audit_log = sa.Table(
    "audit_log", metadata,
    sa.Column("id", sa.UUID(as_uuid=True), primary_key=True),
    sa.Column("user_id", sa.UUID(as_uuid=True)),
    sa.Column("action_type", sa.Text(), nullable=False),
    sa.Column("entity_type", sa.Text()),
    sa.Column("entity_id", sa.UUID(as_uuid=True)),
    sa.Column("before", sa.JSON()),
    sa.Column("after", sa.JSON()),
    sa.Column("ip_address", sa.Text()),
    sa.Column("created_at", sa.TIMESTAMP(timezone=True), server_default=sa.text("NOW()")),
    schema="shared",
)
```

**DO NOT redefine this table.** Import it directly from `admin_api.models.client_tables`.

### Revenue Metrics — MVP Approximation

The `get_revenue_metrics()` function uses:
- **Hardcoded plan prices** (`PLAN_PRICES` dict): free=$0, starter=$29, professional=$79, enterprise=$299
- **Growth rate approximation**: estimates prior-month MRR by subtracting this month's new revenue and adding back churned revenue. This is approximate because it assumes all upgrades/downgrades were from/to starter tier. **Acceptable for MVP admin dashboard.** Phase 2: add `revenue_events` table with event-sourced MRR.
- **Churn/new-paid tracking**: derived from `shared.audit_log` entries with `action_type = 'tier_override'`. This only counts manually-triggered tier changes by admins — it does NOT capture Stripe-driven subscription events (which are out of scope for MVP). Document this limitation in endpoint response or API spec.

### CSV Streaming — StreamingResponse Pattern

Use FastAPI's `StreamingResponse` with an in-memory generator:

```python
def iter_csv():
    yield csv_content

return StreamingResponse(
    iter_csv(),
    media_type="text/csv",
    headers={"Content-Disposition": "attachment; filename=audit-logs.csv"},
)
```

This avoids `response_model=` on the export endpoint (incompatible with `StreamingResponse`). The endpoint return type annotation is `-> StreamingResponse` — no `response_model` kwarg in `@router.get("/export")`.

### JSON Column Queries in Revenue Service

The `shared.audit_log.before` and `after` columns are JSON type. When filtering on JSON fields in SQLAlchemy Core:

```python
# Access JSON field as text for comparison (PostgreSQL JSON operator)
sa.cast(shared_audit_log.c.after["plan"], sa.Text).in_(list(PAID_PLANS))
```

This uses PostgreSQL's `->` JSON operator. If `after` column can be NULL, add a NULL guard:

```python
sa.and_(
    shared_audit_log.c.after.isnot(None),
    sa.cast(shared_audit_log.c.after["plan"], sa.Text).in_(list(PAID_PLANS)),
)
```

### Review Findings from 12-12 — Apply Lessons Learned

The 12-12 code review found several issues. Apply these patterns proactively:

1. **Always call `logger.exception()` or `logger.warning()` in exception handlers** — don't silently swallow errors. Use `structlog.get_logger()` per project-context.md (no `logging.getLogger()`)
2. **Consistent `.where()` usage** — do not conditionally wrap `.where()`. Use `sa.and_(*filters) if filters else sa.true()` pattern consistently (as done in `list_audit_logs` above)
3. **Use `asyncio.get_running_loop()`** not `asyncio.get_event_loop()` if any async socket/executor calls are needed
4. **No redundant SA column-level UNIQUE** alongside explicit partial unique indexes in migrations
5. **Always set `updated_at` explicitly** in service code — don't rely solely on ORM `onupdate` for audit-sensitive timestamps
6. **Run `ruff check`** before submitting — catch F401 (unused imports) and UP-series (deprecated patterns)

### Test File Patterns — Follow 12-12 Exactly

From `test_tenants.py` (established in 12-11):

```python
# JWT env setup fixture (copy from test_tenants.py)
@pytest.fixture(autouse=True)
def jwt_env(monkeypatch):
    monkeypatch.setenv("ADMIN_API_JWT_SECRET", "test-secret")
    monkeypatch.setenv("ADMIN_API_JWT_ALGORITHM", "HS256")

# DB session mock pattern
def make_mock_session(rows=None):
    session = AsyncMock()
    execute_result = MagicMock()
    execute_result.scalars.return_value.all.return_value = rows or []
    execute_result.scalar_one.return_value = 0
    execute_result.mappings.return_value.all.return_value = rows or []
    session.execute.return_value = execute_result
    return session
```

For `export_audit_logs_csv` tests — the service returns a `str` (CSV content). Mock `audit_log_service.export_audit_logs_csv` directly rather than mocking the DB session chain for these tests.

### P0 Test Alignment — ATDD Checklist

The ATDD checklist (`test-artifacts/atdd-checklist-e12-p0.md`) includes these admin access control tests in `e2e/specs/admin/admin-access-control.api.spec.ts` (currently in RED/skipped phase):

- Non-allowlisted IP → 403 on **all admin endpoints** (global, not just 12.13)
- Regular user JWT → 403 on admin endpoints
- Admin JWT → 200 on admin endpoints
- Missing JWT → 401 on admin endpoints

The 12.13 endpoints are included in "all admin endpoints" covered by these P0 tests. When implementing, ensure:
1. The unit tests in `test_audit_logs.py` and `test_platform_analytics.py` verify auth behavior at unit level
2. The Playwright P0 tests in `admin-access-control.api.spec.ts` will cover the actual deployed endpoints during GREEN phase — remove `test.skip()` from that file once endpoints are deployed

### Alembic Migration Numbering

Current highest migration revision: `006` (from `006_crawler_whitelabel_schema.py`, Story 12.12). New migration is `007` with `down_revision = "006"`. Check existing migration files to confirm:
```
services/admin-api/alembic/versions/
├── 001_*.py  (Story 12.11 or earlier)
├── ...
├── 006_crawler_whitelabel_schema.py  (Story 12.12)
└── 007_audit_log_select_grant.py     ← NEW (this story)
```

### Project Structure Notes

- New files follow exact service structure from 12-11/12-12:
  - `services/admin-api/src/admin_api/schemas/audit_log.py` — schemas for both audit log AND platform analytics (combined to keep analytics schemas co-located with their domain)
  - `services/admin-api/src/admin_api/services/audit_log_service.py`
  - `services/admin-api/src/admin_api/services/platform_analytics_service.py`
  - `services/admin-api/src/admin_api/api/v1/audit_logs.py`
  - `services/admin-api/src/admin_api/api/v1/platform_analytics.py`
  - `services/admin-api/alembic/versions/007_audit_log_select_grant.py`
  - `services/admin-api/tests/api/test_audit_logs.py`
  - `services/admin-api/tests/api/test_platform_analytics.py`

- **No new models needed**: `shared_audit_log` SA Core table already in `client_tables.py`. `client_companies`, `client_subscriptions`, `client_mv_usage_consumption` also already defined there.

### References

- [Epic file: eusolicit-docs/planning-artifacts/epic-12-analytics-admin-platform.md#S12.13]
- [Previous story: eusolicit-docs/implementation-artifacts/12-12-admin-api-crawler-white-label-management.md] — established service architecture, testing patterns, review findings
- [Story 12-11: eusolicit-docs/implementation-artifacts/12-11-admin-api-tenant-management.md] — established IPAllowlistMiddleware, cross-schema DB access, get_client_db_session, shared_audit_log table definition
- [Test design: eusolicit-docs/test-artifacts/test-design-epic-12.md#S12.13] — P1 test scenarios (9 tests: 3 audit-log filter, 2 CSV export, 1 funnel, 2 revenue, 1 tiers)
- [ATDD checklist: eusolicit-docs/test-artifacts/atdd-checklist-e12-p0.md] — P0 admin access control tests covering all admin endpoints including 12.13
- [Project context: eusolicit-docs/project-context.md] — tech stack rules, structlog, ruff/mypy, test markers

---

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

### Completion Notes List

### File List
