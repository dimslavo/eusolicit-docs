# Story 12.11: Admin API — Tenant Management

Status: done

## Story

As a **platform operator (internal admin)**,
I want **VPN-restricted admin API endpoints to list, inspect, and override the subscription tier of any tenant company**,
so that **support and operations staff can manage tenant accounts without touching the database directly**.

## Acceptance Criteria

### AC1 — IP Allowlist Middleware

All endpoints under `/api/v1/admin/tenants/*` (and globally, all admin-api routes) must return **HTTP 403** for requests arriving from any IP not in the configured allowlist.

- New Starlette middleware class: `services/admin-api/src/admin_api/middleware/ip_allowlist.py`
- Reads comma-separated CIDRs/IPs from `ADMIN_API_IP_ALLOWLIST` env var (e.g., `"10.0.0.0/8,192.168.1.100"`).
- Uses Python's `ipaddress` stdlib to test `client_ip in network`.
- Client IP resolved from `request.client.host` (trust X-Forwarded-For only if `ADMIN_API_TRUST_PROXY_HEADERS=true`; default false).
- When `ADMIN_API_IP_ALLOWLIST` is empty/unset: **allow all** (local dev default — do not block).
- Returns `{"detail": "Forbidden: your IP address is not on the admin allowlist"}` on rejection.
- Registered in `main.py` **before** all routers: `app.add_middleware(IPAllowlistMiddleware)`.
- New config fields in `services/admin-api/src/admin_api/config.py`:

  ```python
  admin_api_ip_allowlist: str = Field(default="", alias="ADMIN_API_IP_ALLOWLIST")
  admin_api_trust_proxy_headers: bool = Field(default=False, alias="ADMIN_API_TRUST_PROXY_HEADERS")
  ```

---

### AC2 — Cross-Schema Database Access

Admin-api requires read access to `client` schema tables and write access to `shared.audit_log`. This is achieved via a **second async session factory** that uses the same PostgreSQL instance as client-api, pointed at a DB user with appropriate cross-schema grants.

**Config additions** (`services/admin-api/src/admin_api/config.py`):

```python
admin_api_client_db_url: str = Field(
    ...,
    alias="ADMIN_API_CLIENT_DB_URL",
    description="PostgreSQL URL for read access to the client schema (same instance as client-api).",
)
```

**New file: `services/admin-api/src/admin_api/client_db.py`**

```python
"""Second async session factory for cross-schema reads of the client/shared schemas."""
import functools
from collections.abc import AsyncGenerator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from admin_api.config import get_settings

_client_session_factory: async_sessionmaker[AsyncSession] | None = None


def get_client_engine():
    global _client_session_factory
    if _client_session_factory is None:
        settings = get_settings()
        engine = create_async_engine(
            settings.admin_api_client_db_url,
            echo=False,
            pool_size=5,
            max_overflow=10,
        )
        _client_session_factory = async_sessionmaker(engine, expire_on_commit=False)
    return _client_session_factory


async def get_client_db_session() -> AsyncGenerator[AsyncSession, None]:
    """Yield a client-schema DB session. Commit on success, rollback on failure."""
    factory = get_client_engine()
    async with factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

**New dependency alias** — add to `services/admin-api/src/admin_api/dependencies.py`:

```python
from admin_api.client_db import get_client_db_session as _get_client_db_session
from collections.abc import AsyncGenerator
from sqlalchemy.ext.asyncio import AsyncSession

async def get_client_db_session() -> AsyncGenerator[AsyncSession, None]:
    async for session in _get_client_db_session():
        yield session
```

---

### AC3 — Migration 005: Cross-Schema Grants

New migration: `services/admin-api/alembic/versions/005_tenant_management_grants.py`

```
revision = "005"
down_revision = "004"
```

`upgrade()` — run as superuser or DB owner (add `execute_if` guard for idempotency):

```sql
-- Grant admin_api_role read access to client schema tables needed for tenant management
GRANT USAGE ON SCHEMA client TO admin_api_role;
GRANT SELECT ON client.companies TO admin_api_role;
GRANT SELECT ON client.subscriptions TO admin_api_role;
GRANT SELECT ON client.users TO admin_api_role;
GRANT SELECT ON client.refresh_tokens TO admin_api_role;
GRANT SELECT ON client.mv_usage_consumption TO admin_api_role;
GRANT SELECT ON client.mv_team_performance TO admin_api_role;

-- Grant UPDATE on subscriptions for tier-override
GRANT UPDATE ON client.subscriptions TO admin_api_role;

-- Grant WRITE access to shared.audit_log for tier-override audit trail
GRANT USAGE ON SCHEMA shared TO admin_api_role;
GRANT INSERT ON shared.audit_log TO admin_api_role;
```

`downgrade()` — revoke all grants in reverse order.

> **Note:** This migration must be applied with a DB user that owns the `client` and `shared` schemas (typically `postgres` or the migration superuser). Add to the migration header: `# REQUIRES SUPERUSER: yes`.

---

### AC4 — Client Schema Table References

New file: `services/admin-api/src/admin_api/models/client_tables.py`

SQLAlchemy **Core** Table definitions (not ORM) mirroring client schema, used only for read queries and the single UPDATE to `subscriptions.plan`.

```python
import sqlalchemy as sa

metadata = sa.MetaData()

client_companies = sa.Table(
    "companies", metadata,
    sa.Column("id", sa.UUID(as_uuid=True), primary_key=True),
    sa.Column("name", sa.String(255), nullable=False),
    sa.Column("registration_number", sa.String(100)),
    sa.Column("industry", sa.String(100)),
    sa.Column("created_at", sa.TIMESTAMP(timezone=True)),
    schema="client",
)

client_subscriptions = sa.Table(
    "subscriptions", metadata,
    sa.Column("id", sa.UUID(as_uuid=True), primary_key=True),
    sa.Column("company_id", sa.UUID(as_uuid=True), nullable=False),
    sa.Column("plan", sa.String(50)),
    sa.Column("status", sa.String(50)),
    sa.Column("current_period_start", sa.TIMESTAMP(timezone=True)),
    sa.Column("current_period_end", sa.TIMESTAMP(timezone=True)),
    schema="client",
)

client_users = sa.Table(
    "users", metadata,
    sa.Column("id", sa.UUID(as_uuid=True), primary_key=True),
    sa.Column("company_id", sa.UUID(as_uuid=True), nullable=False),
    sa.Column("email", sa.String(255)),
    sa.Column("company_role", sa.String(50)),
    sa.Column("is_active", sa.Boolean()),
    schema="client",
)

client_refresh_tokens = sa.Table(
    "refresh_tokens", metadata,
    sa.Column("id", sa.UUID(as_uuid=True), primary_key=True),
    sa.Column("user_id", sa.UUID(as_uuid=True), nullable=False),
    sa.Column("last_used_at", sa.TIMESTAMP(timezone=True)),
    sa.Column("revoked", sa.Boolean(), nullable=False),
    schema="client",
)

client_mv_usage_consumption = sa.Table(
    "mv_usage_consumption", metadata,
    sa.Column("company_id", sa.UUID(as_uuid=True), nullable=False),
    sa.Column("metric_type", sa.Text(), nullable=False),
    sa.Column("consumed", sa.Numeric(), nullable=False),
    sa.Column("limit_value", sa.Numeric()),
    sa.Column("remaining", sa.Numeric()),
    sa.Column("period_start", sa.TIMESTAMP(timezone=True)),
    sa.Column("period_end", sa.TIMESTAMP(timezone=True)),
    schema="client",
)

client_mv_team_performance = sa.Table(
    "mv_team_performance", metadata,
    sa.Column("company_id", sa.UUID(as_uuid=True), nullable=False),
    sa.Column("user_id", sa.UUID(as_uuid=True)),
    sa.Column("month", sa.TIMESTAMP(timezone=True)),
    sa.Column("bids_submitted", sa.Integer()),
    sa.Column("proposals_generated", sa.Integer()),
    schema="client",
)

# Shared audit log — cross-schema INSERT for tier-override entries
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

---

### AC5 — Pydantic Schemas

New file: `services/admin-api/src/admin_api/schemas/tenant.py`

```python
from __future__ import annotations
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel, Field


class TenantListItem(BaseModel):
    company_id: UUID
    name: str
    registration_number: str | None
    industry: str | None
    plan: str | None           # subscription plan / tier
    sub_status: str | None     # subscription status
    period_end: datetime | None
    created_at: datetime | None

    model_config = {"from_attributes": True}


class TenantListResponse(BaseModel):
    items: list[TenantListItem]
    total: int
    page: int
    page_size: int


class UsageMetric(BaseModel):
    metric_type: str
    consumed: float
    limit_value: float | None
    remaining: float | None
    period_start: datetime | None
    period_end: datetime | None


class TenantActivitySummary(BaseModel):
    last_login: datetime | None
    bids_submitted_this_month: int
    proposals_generated_this_month: int
    active_user_count: int


class TenantDetailResponse(BaseModel):
    company_id: UUID
    name: str
    registration_number: str | None
    industry: str | None
    plan: str | None
    sub_status: str | None
    period_start: datetime | None
    period_end: datetime | None
    usage: list[UsageMetric]
    activity: TenantActivitySummary


class TierOverrideRequest(BaseModel):
    new_plan: str = Field(
        ...,
        pattern="^(free|starter|professional|enterprise)$",
        description="New subscription plan to set.",
    )
    reason: str = Field(..., min_length=10, max_length=500, description="Reason for manual override.")


class TierOverrideResponse(BaseModel):
    company_id: UUID
    previous_plan: str | None
    new_plan: str
    reason: str
    overridden_by_admin_id: UUID
    overridden_at: datetime
```

---

### AC6 — Tenant Service

New file: `services/admin-api/src/admin_api/services/tenant_service.py`

**`list_tenants(session, search, tier, page, page_size)`**

- Query: `JOIN client.companies c ON c.id = s.company_id` — left join subscriptions.
- `search` (optional): `WHERE c.name ILIKE :search` (wrap in `%search%`).
- `tier` (optional): `AND s.plan = :tier`.
- Count query for pagination total.
- `LIMIT page_size OFFSET (page - 1) * page_size`.
- `ORDER BY c.name ASC`.
- Returns `TenantListResponse`.

**`get_tenant_detail(session, company_id)`**

- Fetch `client.companies` row; raise `HTTPException(404)` if not found.
- Fetch `client.subscriptions` where `company_id = :company_id`.
- Fetch current-period usage: `SELECT * FROM client.mv_usage_consumption WHERE company_id = :company_id AND period_start = date_trunc('month', NOW())`.
- Fetch activity:
  - `last_login`: `SELECT MAX(rt.last_used_at) FROM client.refresh_tokens rt JOIN client.users u ON u.id = rt.user_id WHERE u.company_id = :company_id AND rt.revoked = false`
  - `bids_submitted_this_month`: `SELECT COALESCE(SUM(bids_submitted), 0) FROM client.mv_team_performance WHERE company_id = :company_id AND month >= date_trunc('month', NOW())`
  - `proposals_generated_this_month`: `SELECT COALESCE(SUM(proposals_generated), 0) FROM client.mv_team_performance WHERE company_id = :company_id AND month >= date_trunc('month', NOW())`
  - `active_user_count`: `SELECT COUNT(*) FROM client.users WHERE company_id = :company_id AND is_active = true`
- Returns `TenantDetailResponse`.

**`override_tier(session, company_id, request, admin_id)`**

- Fetch `client.subscriptions` row for company_id; raise `404` if company doesn't exist.
- Snapshot `previous_plan = sub.plan`.
- `UPDATE client.subscriptions SET plan = :new_plan WHERE company_id = :company_id`.
- Write audit log entry to `shared.audit_log`:
  ```python
  await session.execute(
      sa.insert(shared_audit_log).values(
          id=uuid4(),
          user_id=admin_id,         # admin_id from AdminUser
          action_type="tier_override",
          entity_type="subscription",
          entity_id=sub_id,
          before={"plan": previous_plan},
          after={"plan": request.new_plan, "reason": request.reason},
      )
  )
  ```
- Returns `TierOverrideResponse`.

---

### AC7 — Tenants Router

New file: `services/admin-api/src/admin_api/api/v1/tenants.py`

```python
"""Admin tenant management endpoints — /api/v1/admin/tenants/* (Story 12.11)."""
from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, Query
from sqlalchemy.ext.asyncio import AsyncSession

from admin_api.client_db import get_client_db_session
from admin_api.core.security import AdminUser, get_admin_user
from admin_api.schemas.tenant import (
    TenantDetailResponse,
    TenantListResponse,
    TierOverrideRequest,
    TierOverrideResponse,
)
from admin_api.services import tenant_service

router = APIRouter(prefix="/admin/tenants", tags=["admin-tenants"])

_Admin = Annotated[AdminUser, Depends(get_admin_user)]
_ClientSession = Annotated[AsyncSession, Depends(get_client_db_session)]


@router.get("/", response_model=TenantListResponse)
async def list_tenants(
    admin: _Admin,
    session: _ClientSession,
    search: str | None = Query(default=None, description="Case-insensitive name search"),
    tier: str | None = Query(default=None, description="Filter by subscription plan"),
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=20, ge=1, le=100),
) -> TenantListResponse:
    """List companies with optional search and tier filter (AC6)."""
    return await tenant_service.list_tenants(session, search, tier, page, page_size)


@router.get("/{company_id}", response_model=TenantDetailResponse)
async def get_tenant_detail(
    company_id: UUID,
    admin: _Admin,
    session: _ClientSession,
) -> TenantDetailResponse:
    """Return subscription details, current usage metrics, and activity summary (AC6)."""
    return await tenant_service.get_tenant_detail(session, company_id)


@router.post("/{company_id}/tier-override", response_model=TierOverrideResponse)
async def override_tenant_tier(
    company_id: UUID,
    body: TierOverrideRequest,
    admin: _Admin,
    session: _ClientSession,
) -> TierOverrideResponse:
    """Manually override a tenant's subscription plan; writes an audit log entry (AC6)."""
    return await tenant_service.override_tier(session, company_id, body, admin.admin_id)
```

---

### AC8 — Register Middleware and Router in main.py

Update `services/admin-api/src/admin_api/main.py`:

```python
# Add import
from admin_api.api.v1 import tenants as tenants_v1
from admin_api.middleware.ip_allowlist import IPAllowlistMiddleware

# Add middleware BEFORE app.include_router(api_v1_router)
app.add_middleware(IPAllowlistMiddleware)

# Register tenant router (literal paths before parameterised; no conflict here)
api_v1_router.include_router(tenants_v1.router)
```

Router registration order for `api_v1_router` (ensure `tenants_v1` is included before any router with overlapping path prefixes):

```python
api_v1_router.include_router(tenants_v1.router)   # /admin/tenants/*
api_v1_router.include_router(fs_v1.router)
api_v1_router.include_router(fa_v1.router)
api_v1_router.include_router(cf_v1.router)
api_v1_router.include_router(rc_v1.router)
api_v1_router.include_router(ps_v1.router)
```

---

### AC9 — Docker Compose / Environment Variables

`ADMIN_API_CLIENT_DB_URL` and `ADMIN_API_IP_ALLOWLIST` must be added to:
- `docker-compose.yml` (or `docker-compose.override.yml`) under the `admin-api` service `environment` block.
- `.env.example` file (if one exists).

In local dev, `ADMIN_API_IP_ALLOWLIST` is left empty (allow all). In staging/prod, set to VPN CIDR (e.g., `10.8.0.0/16`).

`ADMIN_API_CLIENT_DB_URL` in local dev uses the same PostgreSQL host as `CLIENT_API_DATABASE_URL` but must connect with a DB user that has the grants from AC3 applied (e.g., `admin_api_role` after running migration 005).

---

## Tasks / Subtasks

- [ ] Task 1 — Migration 005: cross-schema grants (AC3)
  - [ ] 1.1 Create `services/admin-api/alembic/versions/005_tenant_management_grants.py`
  - [ ] 1.2 Add `GRANT USAGE ON SCHEMA client/shared` + all table SELECT/UPDATE/INSERT grants to `admin_api_role`
  - [ ] 1.3 Write reversible `downgrade()` that revokes all grants
  - [ ] 1.4 Verify migration runs cleanly: `alembic upgrade head` in admin-api service

- [ ] Task 2 — Config & cross-schema session factory (AC2)
  - [ ] 2.1 Add `admin_api_client_db_url`, `admin_api_ip_allowlist`, `admin_api_trust_proxy_headers` to `config.py`
  - [ ] 2.2 Create `services/admin-api/src/admin_api/client_db.py` with second async session factory
  - [ ] 2.3 Add `get_client_db_session()` re-export to `dependencies.py`
  - [ ] 2.4 Add `ADMIN_API_CLIENT_DB_URL` and `ADMIN_API_IP_ALLOWLIST` to `docker-compose.yml`

- [ ] Task 3 — IP Allowlist middleware (AC1)
  - [ ] 3.1 Create `services/admin-api/src/admin_api/middleware/ip_allowlist.py`
  - [ ] 3.2 Implement `IPAllowlistMiddleware(BaseHTTPMiddleware)` using `ipaddress` stdlib
  - [ ] 3.3 Handle `ADMIN_API_TRUST_PROXY_HEADERS` for X-Forwarded-For parsing
  - [ ] 3.4 When allowlist is empty → pass all requests (local dev safety valve)
  - [ ] 3.5 Register middleware in `main.py`

- [ ] Task 4 — Client schema Table references (AC4)
  - [ ] 4.1 Create `services/admin-api/src/admin_api/models/client_tables.py`
  - [ ] 4.2 Define Core Table objects: `client_companies`, `client_subscriptions`, `client_users`, `client_refresh_tokens`, `client_mv_usage_consumption`, `client_mv_team_performance`, `shared_audit_log`
  - [ ] 4.3 Verify column names against existing migrations (see Dev Notes)

- [ ] Task 5 — Pydantic schemas (AC5)
  - [ ] 5.1 Create `services/admin-api/src/admin_api/schemas/tenant.py`
  - [ ] 5.2 Implement `TenantListItem`, `TenantListResponse`, `TenantDetailResponse`, `UsageMetric`, `TenantActivitySummary`, `TierOverrideRequest`, `TierOverrideResponse`

- [ ] Task 6 — Tenant service (AC6)
  - [ ] 6.1 Create `services/admin-api/src/admin_api/services/tenant_service.py`
  - [ ] 6.2 Implement `list_tenants()` with search/tier filter, pagination, count query
  - [ ] 6.3 Implement `get_tenant_detail()` with usage + activity aggregation
  - [ ] 6.4 Implement `override_tier()` with UPDATE + shared.audit_log INSERT in same transaction
  - [ ] 6.5 Unit tests: `services/admin-api/tests/services/test_tenant_service.py`

- [ ] Task 7 — Router (AC7)
  - [ ] 7.1 Create `services/admin-api/src/admin_api/api/v1/tenants.py`
  - [ ] 7.2 Implement all 3 endpoints wiring service + dependencies
  - [ ] 7.3 Register router in `main.py` (AC8)

- [ ] Task 8 — Tests
  - [ ] 8.1 Unit tests for `IPAllowlistMiddleware` (allowed IP, blocked IP, empty allowlist, CIDR range, X-Forwarded-For when proxy trust enabled)
  - [ ] 8.2 Unit tests for tenant_service (mock DB session, assert query patterns)
  - [ ] 8.3 API tests: `GET /admin/tenants` — search filter, tier filter, pagination metadata
  - [ ] 8.4 API tests: `GET /admin/tenants/{id}` — subscription + usage + activity fields
  - [ ] 8.5 API tests: `POST /admin/tenants/{id}/tier-override` — tier updated, audit row created, reason required
  - [ ] 8.6 Negative tests (P0): all 3 endpoints return 403 for blocked IP (test with mocked allowlist)
  - [ ] 8.7 Negative tests (P0): all 3 endpoints return 403 for standard user JWT (no `platform_admin` role)
  - [ ] 8.8 Negative tests (P0): missing JWT header returns 401

## Dev Notes

### Service Architecture

- **Target service:** `services/admin-api` (port 8002), NOT client-api.
- **Auth:** HS256 JWT, `platform_admin` role claim. Dependency: `get_admin_user` → `AdminUser(admin_id, role)`. Already implemented — do NOT reinvent.
- **Admin-api router prefix pattern:** `/api/v1/admin/<resource>` — new router prefix is `/api/v1/admin/tenants`.

### Cross-Schema Access Pattern

Admin-api accesses only its own `admin` schema normally. This story introduces a **second session factory** (`client_db.py`) for `client`/`shared` schema reads/writes. This is intentional and clean — no code in admin-api should import from `client_api` package; instead use the local `client_tables.py` Core Table definitions.

The `get_client_db_session()` dependency is used in the tenant router **instead of** `get_db_session()`. Do not mix the two sessions in a single endpoint — each endpoint uses only one session.

### Column Names to Verify (from client-api migrations)

Cross-check these against actual migration files before writing queries:
- `client.companies`: `id`, `name`, `registration_number`, `industry`, `created_at` — ✅ confirmed from exploration
- `client.subscriptions`: `id`, `company_id`, `plan`, `status`, `current_period_start`, `current_period_end` — ✅ confirmed
- `client.users`: `id`, `company_id`, `email`, `company_role`, `is_active` — verify `company_role` column name (may be `role`)
- `client.refresh_tokens`: `id`, `user_id`, `last_used_at`, `revoked` — verify `last_used_at` column exists (may be `used_at` or `last_used`)
- `client.mv_usage_consumption`: columns vary by view definition — check `011_analytics_materialized_views.py`
- `client.mv_team_performance`: check same migration for exact column names (`bids_submitted`, `proposals_generated`)
- `shared.audit_log`: check column names for `user_id`, `action_type`, `entity_type`, `entity_id`, `before`, `after` — look at `client_api/services/audit_service.py` for `AuditLog` ORM model definition

> **IMPORTANT:** Before writing any query, read the actual migration files and model files from `services/client-api/alembic/versions/` and `services/client-api/src/client_api/models/` to confirm exact column names. Do NOT assume they match the Table definitions in AC4 exactly.

### IP Allowlist Middleware Implementation Details

```python
import ipaddress
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse

class IPAllowlistMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, settings=None):
        super().__init__(app)
        from admin_api.config import get_settings
        s = settings or get_settings()
        raw = s.admin_api_ip_allowlist.strip()
        self._networks = [
            ipaddress.ip_network(entry.strip(), strict=False)
            for entry in raw.split(",") if entry.strip()
        ] if raw else []
        self._trust_proxy = s.admin_api_trust_proxy_headers

    async def dispatch(self, request: Request, call_next):
        if not self._networks:
            return await call_next(request)
        if self._trust_proxy:
            forwarded_for = request.headers.get("X-Forwarded-For", "")
            client_ip_str = forwarded_for.split(",")[0].strip() if forwarded_for else (request.client.host if request.client else "")
        else:
            client_ip_str = request.client.host if request.client else ""
        try:
            client_ip = ipaddress.ip_address(client_ip_str)
        except ValueError:
            return JSONResponse({"detail": "Forbidden: invalid client IP"}, status_code=403)
        for network in self._networks:
            if client_ip in network:
                return await call_next(request)
        return JSONResponse(
            {"detail": "Forbidden: your IP address is not on the admin allowlist"},
            status_code=403,
        )
```

### Tier Override Audit Log

The audit log entry for tier-override uses `action_type="tier_override"` (not `"update"`) to distinguish admin overrides from normal subscription changes. The `after` dict **must include the `reason` field** so the audit record is self-documenting.

Audit log schema uses `JSON` (not `JSONB`) columns for `before`/`after` — check `shared.audit_log` table definition; if it uses `JSONB`, cast accordingly.

### Pagination Pattern

Follow the same pagination envelope used in client-api reports:
```python
{"items": [...], "total": count, "page": page, "page_size": page_size}
```
Use a count query (`SELECT COUNT(*) FROM ... WHERE ...`) with the same filters before fetching the page. Do not use `.count()` on the ORM — execute explicit `sa.func.count()` select for correct async behavior.

### test_design Reference

From `test-design-epic-12.md` — **P0 tests** (must pass before review):

| Test | Expected |
|------|----------|
| Non-allowlisted IP → all 3 tenant endpoints | 403 `{"detail": "Forbidden: your IP address..."}` |
| Standard user JWT (no `platform_admin` claim) → all 3 tenant endpoints | 403 `"Platform admin access required"` |
| Missing JWT header → all 3 tenant endpoints | 401 |

**P1 tests** (core functionality):

| Test | Expected |
|------|----------|
| `GET /admin/tenants?search=acme` | Returns only companies with "acme" in name (case-insensitive) |
| `GET /admin/tenants?tier=starter` | Returns only companies with `subscriptions.plan = 'starter'` |
| `GET /admin/tenants` with no filters | Correct `total`, `page`, `page_size` in response |
| `GET /admin/tenants/{id}` | Response includes `plan`, `sub_status`, `usage` list, `activity` object |
| `POST /admin/tenants/{id}/tier-override` | `subscriptions.plan` updated in DB + `shared.audit_log` row with `action_type="tier_override"` |
| `POST /admin/tenants/{id}/tier-override` without `reason` | 422 |

**P2 tests**:
- Pagination metadata (`total`, `page`, `page_size`) all present and correct on list endpoint.

### No Existing Admin Tenant Code

There is **no existing tenant management code** in admin-api. This story creates all tenant management files from scratch. Do not confuse with:
- `services/client-api/api/v1/companies.py` — client-facing company CRUD (different service, different auth)
- `services/admin-api/api/v1/framework_assignments.py` — uses `opportunity_id` soft FK, unrelated

### Previous Story Context

Story 12.10 (scheduled/on-demand report delivery) established:
- Celery producer pattern in client-api: `celery_producer.send_task()` (task dispatch without importing worker code)
- `client.report_schedules` and `client.report_jobs` tables (migrations 013, 014)
- Audit log pattern: `write_audit_entry(session, user_id=..., action_type=..., entity_type=..., entity_id=..., before=..., after=...)`

For this story the audit pattern is the same conceptually, but admin-api writes directly to `shared.audit_log` via Core Table INSERT (no `write_audit_entry` helper available in admin-api — that helper lives in client-api's service layer).

### Project Structure Notes

New files follow existing admin-api conventions:
```
services/admin-api/
├── alembic/versions/
│   └── 005_tenant_management_grants.py      ← new
├── src/admin_api/
│   ├── api/v1/
│   │   └── tenants.py                        ← new
│   ├── middleware/
│   │   └── ip_allowlist.py                   ← new (create middleware/ dir)
│   ├── models/
│   │   └── client_tables.py                  ← new (Core Tables, not ORM)
│   ├── schemas/
│   │   └── tenant.py                         ← new
│   ├── services/
│   │   └── tenant_service.py                 ← new
│   ├── client_db.py                          ← new
│   ├── config.py                             ← modify (3 new fields)
│   ├── dependencies.py                       ← modify (add get_client_db_session)
│   └── main.py                               ← modify (middleware + router)
└── tests/
    ├── middleware/
    │   └── test_ip_allowlist.py              ← new
    ├── services/
    │   └── test_tenant_service.py            ← new
    └── api/v1/
        └── test_tenants.py                   ← new
```

Existing admin-api router files use `router = APIRouter(prefix="/admin/<resource>", tags=["..."])` — follow the same pattern.

Models use ORM (`Base` declarative) for admin schema objects, but Core Table definitions (not ORM) are used for cross-schema read-only references in `client_tables.py` — matching the pattern in client-api's `models/report_job.py`.

### References

- Admin-api router pattern: [Source: services/admin-api/src/admin_api/api/v1/platform_settings.py]
- AdminUser + get_admin_user: [Source: services/admin-api/src/admin_api/core/security.py]
- Admin-api config pattern: [Source: services/admin-api/src/admin_api/config.py]
- Admin-api main.py (router registration): [Source: services/admin-api/src/admin_api/main.py]
- Cross-schema grants examples: [Source: services/client-api/alembic/versions/011_analytics_materialized_views.py]
- client.companies + client.subscriptions columns: [Source: services/client-api/src/client_api/models/company.py, subscription.py]
- client-api audit log service: [Source: services/client-api/src/client_api/services/audit_service.py]
- Epic-12 test design (S12.11 P0/P1 coverage): [Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#S12.11]
- IP allowlist risk: R12.2 (SEC) — [Source: eusolicit-docs/test-artifacts/test-design-epic-12.md#R12.2]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (Claude Code)

### Debug Log References

- **DEVIATION (deferrable):** `client.refresh_tokens` has `is_revoked` not `revoked`, and no `last_used_at` column. Implementation uses `is_revoked` for filter and `created_at` as last-login proxy.
- **DEVIATION (deferrable):** `client.users` has no `company_id`/`company_role`. User-company link is via `client.company_memberships`. Active-user and last-login queries JOIN `company_memberships`.
- **DEVIATION (deferrable):** `shared.audit_log` uses `timestamp` column (not `created_at`), and `before`/`after` are JSONB. `client_tables.py` corrected accordingly.
- **DEVIATION (deferrable):** `mv_usage_consumption.period_start/period_end` are `Date` not `TIMESTAMP`; `consumed`/`limit_value`/`remaining` are `Integer` not `Numeric`. `mv_team_performance.month` is `Date`, `bids_submitted`/`proposals_generated` are `BigInteger`.

### Completion Notes List

- All 9 acceptance criteria implemented (AC1–AC9).
- 45 new tests created: 11 middleware unit tests, 9 service unit tests, 25 API integration tests.
- 193 total tests pass (0 regressions in existing test suite).
- Column names in `client_tables.py` verified against actual ORM models and migration files.
- Middleware tests use `httpx.AsyncClient` with `ASGITransport(client=...)` to control the ASGI client IP (Starlette TestClient sets non-IP `"testclient"` host which would always trigger the invalid-IP path).

### File List

- `services/admin-api/alembic/versions/005_tenant_management_grants.py` — new migration
- `services/admin-api/src/admin_api/config.py` — added 3 new fields
- `services/admin-api/src/admin_api/client_db.py` — new: second async session factory
- `services/admin-api/src/admin_api/dependencies.py` — added `get_client_db_session`
- `services/admin-api/src/admin_api/middleware/__init__.py` — new directory
- `services/admin-api/src/admin_api/middleware/ip_allowlist.py` — new middleware
- `services/admin-api/src/admin_api/models/client_tables.py` — new Core Table definitions
- `services/admin-api/src/admin_api/schemas/tenant.py` — new Pydantic schemas
- `services/admin-api/src/admin_api/services/tenant_service.py` — new service
- `services/admin-api/src/admin_api/api/v1/tenants.py` — new router
- `services/admin-api/src/admin_api/main.py` — registered middleware + router
- `docker-compose.yml` — added ADMIN_API_CLIENT_DB_URL and ADMIN_API_IP_ALLOWLIST
- `services/admin-api/tests/middleware/__init__.py` — new
- `services/admin-api/tests/middleware/test_ip_allowlist.py` — new: 11 tests
- `services/admin-api/tests/services/__init__.py` — new
- `services/admin-api/tests/services/test_tenant_service.py` — new: 9 tests
- `services/admin-api/tests/api/test_tenants.py` — new: 25 tests
