# Story 2.8: Company Profile CRUD

Status: done

## Story

As a company admin or bid manager,
I want to retrieve and update my company's profile (name, tax ID, address, industry, CPV sectors, regions, description, certifications),
so that my company information is accurate and complete for EU procurement activities.

## Acceptance Criteria

1. **AC1** — `GET /api/v1/companies/{id}` returns the full company profile (HTTP 200) for any authenticated member of that company. A request from a user belonging to a different company returns 403.
2. **AC2** — `PUT /api/v1/companies/{id}` replaces the entire profile and returns the updated company (HTTP 200). Only admin and bid_manager roles may execute it; contributor, reviewer, and read_only receive 403.
3. **AC3** — `PATCH /api/v1/companies/{id}` merges partial updates — only fields present in the request body are written to the database; absent fields remain unchanged. Returns the updated company (HTTP 200). Same role restriction as PUT.
4. **AC4** — CPV sector codes in `cpv_sectors` are individually validated against the regex `^\d{8}-\d$` (e.g., `"12345678-9"`). Any invalid code returns HTTP 422 before reaching the service layer.
5. **AC5** — When an `address` object is provided, all four sub-fields (`street`, `city`, `postal_code`, `country`) are required. A partial address (missing any required sub-field) returns HTTP 422.
6. **AC6** — Every PUT and PATCH that modifies the company is audit-logged to `shared.audit_log` with: `action_type="update"`, `entity_type="company"`, `entity_id=<company_id>`, `before=<snapshot_before_update>`, `after=<snapshot_after_update>`, `user_id=<current_user.user_id>`.
7. **AC7** — Unauthenticated requests to any company endpoint return 401. Authenticated requests targeting another company's `{id}` return 403.

## Tasks / Subtasks

- [x] Task 1 — Create `src/client_api/schemas/company.py` with Pydantic models (AC: 1–5)
  - [x] 1.1 Create the file with `from __future__ import annotations` at the top
  - [x] 1.2 Define `AddressSchema(BaseModel)`: required fields `street: str = Field(min_length=1, max_length=500)`, `city: str = Field(min_length=1, max_length=100)`, `postal_code: str = Field(min_length=1, max_length=20)`, `country: str = Field(min_length=1, max_length=100)`
  - [x] 1.3 Define `CertificationSchema(BaseModel)`: `name: str = Field(min_length=1, max_length=255)`, `issuer: str = Field(min_length=1, max_length=255)`, `valid_until: date | None = None`
  - [x] 1.4 Define `CompanyProfileResponse(BaseModel)` with `model_config = ConfigDict(from_attributes=True)`: fields `id: UUID`, `name: str`, `registration_number: str | None`, `tax_id: str | None`, `address: dict | None`, `industry: str | None`, `cpv_sectors: list | None`, `regions: list | None`, `description: str | None`, `certifications: list | None`, `created_at: datetime`, `updated_at: datetime`
  - [x] 1.5 Define `CompanyProfilePutRequest(BaseModel)`: required `name: str = Field(min_length=1, max_length=255)`; optional `tax_id: str | None = None`, `address: AddressSchema | None = None`, `industry: str | None = None`, `cpv_sectors: list[str] | None = None`, `regions: list[str] | None = None`, `description: str | None = None`, `certifications: list[CertificationSchema] | None = None`; add `@field_validator("cpv_sectors")` that validates each code against `re.fullmatch(r"^\d{8}-\d$", code)` and raises `ValueError` on mismatch
  - [x] 1.6 Define `CompanyProfilePatchRequest(BaseModel)`: same set of fields as `PutRequest` but ALL optional (`name: str | None = Field(None, min_length=1, max_length=255)`); reuse the same CPV `@field_validator`; this model is used for partial updates and `model_fields_set` determines which fields are applied

- [x] Task 2 — Create `src/client_api/services/company_service.py` (AC: 1–6)
  - [x] 2.1 Create file with `from __future__ import annotations`, structlog logger (`log = structlog.get_logger()`), and all required imports
  - [x] 2.2 Implement helper `async def _get_company_or_raise(company_id: UUID, current_user: CurrentUser, session: AsyncSession) -> Company`
  - [x] 2.3 Implement `_company_to_snapshot(company: Company) -> dict`
  - [x] 2.4 Implement `async def get_company_profile(company_id: UUID, current_user: CurrentUser, session: AsyncSession) -> Company`
  - [x] 2.5 Implement `async def update_company_full(...)` with audit logging
  - [x] 2.6 Implement `async def update_company_partial(...)` with audit logging

- [x] Task 3 — Create `src/client_api/api/v1/companies.py` (AC: 1–7)
  - [x] 3.1 Create file with `from __future__ import annotations`
  - [x] 3.2 Define `router = APIRouter(prefix="/companies", tags=["companies"])`
  - [x] 3.3 Add `GET /{company_id}`
  - [x] 3.4 Add `PUT /{company_id}`
  - [x] 3.5 Add `PATCH /{company_id}`

- [x] Task 4 — Register companies router in `main.py`
  - [x] 4.1 Add `from client_api.api.v1 import companies as companies_v1` to `main.py`
  - [x] 4.2 Add `api_v1_router.include_router(companies_v1.router)` immediately after the auth router line

- [x] Task 5 — Tests (23/23 passing; SQL fixes applied for asyncpg compatibility)

## Dev Notes

### Architecture Constraints (MUST FOLLOW)

**1. No new Alembic migration needed.**
The `client.companies` table was created in `002_auth_identity_tables.py` and the Company ORM model (`src/client_api/models/company.py`) already defines all required columns. The latest migration is `006_password_reset_tokens.py`. No new migrations for this story.

**2. Files to create (NEW):**
```
src/client_api/
  schemas/
    company.py                           ← NEW
  services/
    company_service.py                   ← NEW
  api/
    v1/
      companies.py                       ← NEW
tests/
  api/
    test_company_profile.py              ← NEW
```

**3. Files to modify (EXISTING):**
```
src/client_api/main.py                   ← add companies_v1 router
```

**4. Do NOT create a companies-specific service file for audit service.** Write AuditLog rows directly in `company_service.py` using the existing `AuditLog` model imported from `client_api.models`. Story 2.11 will refactor this into a dedicated audit middleware; for now, explicit calls in the service are correct. The `AuditLog` model is in `shared` schema — `client_api_role` has INSERT-only permissions on it (enforced at the DB level).

**5. Cross-tenant guard is critical (E02-R-002).** The JWT `company_id` claim is the authoritative source of which company a user belongs to. The guard in `_get_company_or_raise()` must compare `current_user.company_id != company_id` as UUIDs (not strings). The `CurrentUser.company_id` is already a `UUID` instance from `core/security.py`. The path parameter `company_id: UUID` is automatically parsed by FastAPI.

**6. `require_role("bid_manager")` enforces admin + bid_manager.** The existing `require_role()` factory in `core/security.py` returns a dependency that checks `ROLE_HIERARCHY[user.role] >= ROLE_HIERARCHY["bid_manager"]` (score 4). Admin (5) and bid_manager (4) both pass; contributor (3), reviewer (2), read_only (1) all fail with `ForbiddenError` (403). Import: `from client_api.core.security import CurrentUser, get_current_user, require_role`.

**7. PATCH partial update via `model_fields_set`.** Pydantic v2 tracks which fields were explicitly provided in the request body via `request.model_fields_set`. A field absent from the JSON body will NOT appear in `model_fields_set`, so absent fields must NOT be applied to the ORM object. Example:
```python
for field_name in request.model_fields_set:
    if field_name == "address":
        company.address = request.address.model_dump(mode="json") if request.address else None
    elif field_name == "certifications":
        company.certifications = [c.model_dump(mode="json") for c in request.certifications] if request.certifications else None
    else:
        setattr(company, field_name, getattr(request, field_name))
```
This ensures `description` won't be wiped when only `name` is PATCHed.

**8. JSONB serialization — use `mode="json"`.** The `address` and `certifications` fields on `Company` are `JSONB`. ALWAYS use `model_dump(mode="json")` (not plain `model_dump()`) when converting Pydantic nested models to dicts for JSONB storage. `mode="json"` ensures that `date` fields in `CertificationSchema.valid_until` are serialized as ISO-8601 strings, not Python `datetime.date` objects, which are not JSON-serializable and will cause a runtime error when SQLAlchemy tries to serialize them for PostgreSQL. Address fields are all `str` types so either mode works for address, but use `mode="json"` consistently.

**9. `updated_at` is handled by SQLAlchemy ORM's `onupdate` clause.** The `Company` model defines `onupdate=sa.func.now()`, so `updated_at` is automatically included in the SQL UPDATE statement when the ORM flushes a modified Company. No manual `company.updated_at = datetime.now(UTC)` is needed.

**10. Audit snapshot serialization.** The `AuditLog.before` and `.after` fields are `JSONB`. Convert UUID values to strings before storing:
```python
def _company_to_snapshot(company: Company) -> dict:
    return {
        "name": company.name,
        "tax_id": company.tax_id,
        "address": company.address,
        "industry": company.industry,
        "cpv_sectors": company.cpv_sectors,
        "regions": company.regions,
        "description": company.description,
        "certifications": company.certifications,
    }
```
Capture `before` BEFORE any ORM attribute mutation; capture `after` AFTER `await session.flush()`.

**11. `from __future__ import annotations` at top of every new file.** Already the established pattern in the codebase — maintain it.

**12. Use `structlog` for all logging.** No `print()` or `logging.getLogger()`. Pattern: `import structlog; log = structlog.get_logger()`.

**13. `get_db_session` handles commit/rollback.** Do NOT call `session.commit()` inside service functions. The dependency wrapper in `dependencies.py` commits on success and rolls back on exception.

**14. Router parameter naming.** In `companies.py`, the route body parameter MUST NOT be named `request` (that name conflicts with the Starlette `Request` object needed for IP extraction). Use `body: CompanyProfilePutRequest` / `body: CompanyProfilePatchRequest`. For the Starlette Request, use `http_request: Request`. Import: `from starlette.requests import Request` (already used in `auth.py`).

### Code Skeletons

#### `schemas/company.py`

```python
"""Pydantic schemas for company profile endpoints."""
from __future__ import annotations

import re
from datetime import date, datetime
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field, field_validator


class AddressSchema(BaseModel):
    """Structured address for a company."""

    street: str = Field(min_length=1, max_length=500)
    city: str = Field(min_length=1, max_length=100)
    postal_code: str = Field(min_length=1, max_length=20)
    country: str = Field(min_length=1, max_length=100)


class CertificationSchema(BaseModel):
    """A single certification held by a company."""

    name: str = Field(min_length=1, max_length=255)
    issuer: str = Field(min_length=1, max_length=255)
    valid_until: date | None = None


_CPV_PATTERN = re.compile(r"^\d{8}-\d$")


class CompanyProfileResponse(BaseModel):
    """Full company profile returned from GET/PUT/PATCH endpoints."""

    model_config = ConfigDict(from_attributes=True)

    id: UUID
    name: str
    registration_number: str | None = None
    tax_id: str | None = None
    address: dict | None = None
    industry: str | None = None
    cpv_sectors: list | None = None
    regions: list | None = None
    description: str | None = None
    certifications: list | None = None
    created_at: datetime
    updated_at: datetime


class CompanyProfilePutRequest(BaseModel):
    """Full-replace request body for PUT /companies/{id}."""

    name: str = Field(min_length=1, max_length=255)
    tax_id: str | None = None
    address: AddressSchema | None = None
    industry: str | None = None
    cpv_sectors: list[str] | None = None
    regions: list[str] | None = None
    description: str | None = None
    certifications: list[CertificationSchema] | None = None

    @field_validator("cpv_sectors")
    @classmethod
    def validate_cpv_sectors(cls, v: list[str] | None) -> list[str] | None:
        if v is None:
            return v
        for code in v:
            if not _CPV_PATTERN.fullmatch(code):
                raise ValueError(
                    f"Invalid CPV code '{code}'. Expected format: 8 digits, dash, 1 digit (e.g. '12345678-9')"
                )
        return v


class CompanyProfilePatchRequest(BaseModel):
    """Partial-update request body for PATCH /companies/{id}.

    Only fields present in the JSON body (tracked via model_fields_set) are applied.
    """

    name: str | None = Field(None, min_length=1, max_length=255)
    tax_id: str | None = None
    address: AddressSchema | None = None
    industry: str | None = None
    cpv_sectors: list[str] | None = None
    regions: list[str] | None = None
    description: str | None = None
    certifications: list[CertificationSchema] | None = None

    @field_validator("cpv_sectors")
    @classmethod
    def validate_cpv_sectors(cls, v: list[str] | None) -> list[str] | None:
        if v is None:
            return v
        for code in v:
            if not _CPV_PATTERN.fullmatch(code):
                raise ValueError(
                    f"Invalid CPV code '{code}'. Expected format: 8 digits, dash, 1 digit (e.g. '12345678-9')"
                )
        return v
```

#### `services/company_service.py`

```python
"""Company profile service — CRUD business logic."""
from __future__ import annotations

from uuid import UUID

import structlog
from fastapi import HTTPException
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

from eusolicit_common.exceptions import ForbiddenError

from client_api.core.security import CurrentUser
from client_api.models import AuditLog, Company
from client_api.schemas.company import CompanyProfilePatchRequest, CompanyProfilePutRequest

log = structlog.get_logger()


def _company_to_snapshot(company: Company) -> dict:
    """Serialize mutable company profile fields to a plain dict for audit storage."""
    return {
        "name": company.name,
        "tax_id": company.tax_id,
        "address": company.address,
        "industry": company.industry,
        "cpv_sectors": company.cpv_sectors,
        "regions": company.regions,
        "description": company.description,
        "certifications": company.certifications,
    }


async def _get_company_or_raise(
    company_id: UUID,
    current_user: CurrentUser,
    session: AsyncSession,
) -> Company:
    """Fetch company by id, enforcing cross-tenant access control."""
    if current_user.company_id != company_id:
        raise ForbiddenError("You do not have access to this company")
    result = await session.execute(select(Company).where(Company.id == company_id))
    company = result.scalar_one_or_none()
    if company is None:
        raise HTTPException(status_code=404, detail="Company not found")
    return company


async def get_company_profile(
    company_id: UUID,
    current_user: CurrentUser,
    session: AsyncSession,
) -> Company:
    """Return company profile for an authenticated member."""
    return await _get_company_or_raise(company_id, current_user, session)


async def update_company_full(
    company_id: UUID,
    request: CompanyProfilePutRequest,
    current_user: CurrentUser,
    session: AsyncSession,
    ip_address: str | None = None,
) -> Company:
    """Replace entire company profile and write an audit log entry."""
    company = await _get_company_or_raise(company_id, current_user, session)
    before = _company_to_snapshot(company)

    company.name = request.name
    company.tax_id = request.tax_id
    company.address = request.address.model_dump(mode="json") if request.address else None
    company.industry = request.industry
    company.cpv_sectors = request.cpv_sectors
    company.regions = request.regions
    company.description = request.description
    company.certifications = (
        [c.model_dump(mode="json") for c in request.certifications] if request.certifications else None
    )

    await session.flush()
    after = _company_to_snapshot(company)

    session.add(
        AuditLog(
            user_id=current_user.user_id,
            action_type="update",
            entity_type="company",
            entity_id=company_id,
            before=before,
            after=after,
            ip_address=ip_address,
        )
    )
    await session.flush()
    log.info("company.profile_updated_full", company_id=str(company_id), user_id=str(current_user.user_id))
    return company


async def update_company_partial(
    company_id: UUID,
    request: CompanyProfilePatchRequest,
    current_user: CurrentUser,
    session: AsyncSession,
    ip_address: str | None = None,
) -> Company:
    """Merge partial updates into company profile and write an audit log entry."""
    company = await _get_company_or_raise(company_id, current_user, session)
    before = _company_to_snapshot(company)

    for field_name in request.model_fields_set:
        if field_name == "address":
            company.address = request.address.model_dump(mode="json") if request.address else None
        elif field_name == "certifications":
            company.certifications = (
                [c.model_dump(mode="json") for c in request.certifications] if request.certifications else None
            )
        else:
            setattr(company, field_name, getattr(request, field_name))

    await session.flush()
    after = _company_to_snapshot(company)

    session.add(
        AuditLog(
            user_id=current_user.user_id,
            action_type="update",
            entity_type="company",
            entity_id=company_id,
            before=before,
            after=after,
            ip_address=ip_address,
        )
    )
    await session.flush()
    log.info("company.profile_updated_partial", company_id=str(company_id), user_id=str(current_user.user_id))
    return company
```

#### `api/v1/companies.py`

```python
"""Company profile API router."""
from __future__ import annotations

from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from starlette.requests import Request

from client_api.core.security import CurrentUser, get_current_user, require_role
from client_api.dependencies import get_db_session
from client_api.schemas.company import (
    CompanyProfilePatchRequest,
    CompanyProfilePutRequest,
    CompanyProfileResponse,
)
from client_api.services import company_service

router = APIRouter(prefix="/companies", tags=["companies"])


@router.get("/{company_id}", status_code=200, response_model=CompanyProfileResponse)
async def get_company_profile(
    company_id: UUID,
    session: Annotated[AsyncSession, Depends(get_db_session)],
    current_user: Annotated[CurrentUser, Depends(get_current_user)],
) -> CompanyProfileResponse:
    """Return full company profile for any authenticated company member."""
    company = await company_service.get_company_profile(company_id, current_user, session)
    return CompanyProfileResponse.model_validate(company)


@router.put("/{company_id}", status_code=200, response_model=CompanyProfileResponse)
async def put_company_profile(
    company_id: UUID,
    body: CompanyProfilePutRequest,
    http_request: Request,
    session: Annotated[AsyncSession, Depends(get_db_session)],
    current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
) -> CompanyProfileResponse:
    """Replace entire company profile. Requires admin or bid_manager role."""
    ip_address = http_request.client.host if http_request.client else None
    company = await company_service.update_company_full(
        company_id, body, current_user, session, ip_address
    )
    return CompanyProfileResponse.model_validate(company)


@router.patch("/{company_id}", status_code=200, response_model=CompanyProfileResponse)
async def patch_company_profile(
    company_id: UUID,
    body: CompanyProfilePatchRequest,
    http_request: Request,
    session: Annotated[AsyncSession, Depends(get_db_session)],
    current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))],
) -> CompanyProfileResponse:
    """Merge partial updates into company profile. Requires admin or bid_manager role."""
    ip_address = http_request.client.host if http_request.client else None
    company = await company_service.update_company_partial(
        company_id, body, current_user, session, ip_address
    )
    return CompanyProfileResponse.model_validate(company)
```

#### `main.py` change (diff)

```python
# BEFORE:
from client_api.api.v1 import auth as auth_v1

# AFTER:
from client_api.api.v1 import auth as auth_v1
from client_api.api.v1 import companies as companies_v1

# BEFORE:
api_v1_router.include_router(auth_v1.router)

# AFTER:
api_v1_router.include_router(auth_v1.router)
api_v1_router.include_router(companies_v1.router)
```

### Test Fixture Pattern

The `company_client_and_session` fixture follows the same shared-session pattern as `password_reset_client_and_session` in `test_auth_password_reset.py`. Shape:

```python
@pytest_asyncio.fixture
async def company_client_and_session(
    client_api_session_factory,
    test_redis_client,
) -> AsyncGenerator[tuple[httpx.AsyncClient, AsyncSession, str, str], None]:
    """
    Registers a unique admin user + company, verifies email, logs in.
    Yields (client, session, access_token, company_id_str).
    Rolls back all DB changes on exit.
    """
    from client_api.dependencies import get_db_session, get_email_service_dep, get_redis_client
    from client_api.main import app as fastapi_app
    from client_api.services.email_service import StubEmailService

    async with client_api_session_factory() as session:
        async with session.begin():
            try:
                # 1. Override dependencies
                async def override_db():
                    yield session

                fastapi_app.dependency_overrides[get_db_session] = override_db
                fastapi_app.dependency_overrides[get_email_service_dep] = lambda: StubEmailService()
                fastapi_app.dependency_overrides[get_redis_client] = lambda: test_redis_client

                async with httpx.AsyncClient(
                    transport=httpx.ASGITransport(app=fastapi_app),
                    base_url="http://test",
                    headers={"Content-Type": "application/json"},
                ) as client:
                    # 2. Register
                    import uuid
                    unique = uuid.uuid4().hex[:8]
                    reg_resp = await client.post("/api/v1/auth/register", json={
                        "email": f"admin_{unique}@test.com",
                        "password": "AdminPass1",
                        "full_name": "Test Admin",
                        "company_name": f"TestCo {unique}",
                    })
                    assert reg_resp.status_code == 201
                    company_id = reg_resp.json()["company"]["id"]
                    email = f"admin_{unique}@test.com"

                    # 3. Verify email via SQL
                    await session.execute(
                        text("UPDATE client.users SET email_verified = true WHERE email = :email"),
                        {"email": email},
                    )

                    # 4. Login
                    login_resp = await client.post("/api/v1/auth/login", json={
                        "email": email,
                        "password": "AdminPass1",
                    })
                    assert login_resp.status_code == 200
                    access_token = login_resp.json()["access_token"]

                    yield client, session, access_token, company_id

            finally:
                fastapi_app.dependency_overrides.clear()
                await session.rollback()
```

**Note:** For role enforcement tests (Task 5.7), do NOT go through the invite flow (Story 2.9). Instead, insert a `CompanyMembership` row directly via the session with the target role:

```python
from client_api.models import CompanyMembership, User
from client_api.models.enums import CompanyRole
import bcrypt, uuid

# Create a new user directly
new_user = User(
    email=f"contrib_{uuid.uuid4().hex[:8]}@test.com",
    hashed_password=bcrypt.hashpw(b"Contrib123", bcrypt.gensalt(rounds=12)).decode(),
    full_name="Contributor User",
    email_verified=True,
    is_active=True,
)
session.add(new_user)
await session.flush()

membership = CompanyMembership(
    user_id=new_user.id,
    company_id=uuid.UUID(company_id),
    role=CompanyRole.contributor,
    accepted_at=datetime.now(UTC),
)
session.add(membership)
await session.flush()

# Login as this user to get a token, then test PUT/PATCH → expect 403
```

### Project Structure Notes

- The `companies.py` router lives in `services/client-api/src/client_api/api/v1/` alongside the existing `auth.py`.
- The `company_service.py` lives in `services/client-api/src/client_api/services/` alongside `auth_service.py` and `email_service.py`.
- The `schemas/company.py` lives in `services/client-api/src/client_api/schemas/` alongside `auth.py`.
- The existing `schemas/__init__.py` exports auth schemas; **do not need to update it** — `companies.py` imports directly from `client_api.schemas.company`.
- No `from client_api.services import company_service` import update needed — Python resolves this at runtime. However, if a `services/__init__.py` needs updating, check `src/client_api/services/__init__.py` (currently appears empty/minimal).

### Epic-Level Test Coverage Mapping (from test-design-epic-02.md)

| AC | Epic Test ID | Test Class | Description |
|----|-------------|-----------|-------------|
| AC1 (GET) | E02-P2-004 | `TestAC1GetCompanyProfile` | Any member can GET; unauthenticated → 401; wrong company → 403 |
| AC2 (PUT) | E02-P1-014 | `TestAC2PutCompanyProfile` | admin/bid_manager → 200; others → 403 |
| AC3 (PATCH) | E02-P2-006 | `TestAC3PatchCompanyProfile` | Partial merge; unchanged fields stay |
| AC4 (CPV) | E02-P2-005 | `TestAC4CPVValidation` | Invalid CPV → 422; valid CPV → 200 |
| AC5 (address) | — | `TestAC5AddressValidation` | Partial address → 422; complete → 200 |
| AC6 (audit) | E02-P1-018 | `TestAC7AuditLog` | PUT logged with before/after snapshots |
| AC7 (roles) | E02-P1-014 | `TestAC6RoleEnforcement` | contributor/reviewer/read_only → 403; bid_manager → 200 |

**Note:** E02-P0-012 (cross-tenant isolation sweep) is partially satisfied by AC7 / `TestAC1GetCompanyProfile` — the fixture that calls GET with a random non-matching UUID exercises the cross-tenant guard.

### Security Notes (from Epic Test Design E02-R-001, E02-R-002)

- **E02-R-002 (cross-tenant isolation):** The `_get_company_or_raise()` guard (`current_user.company_id != company_id`) enforces that Company A's users cannot read or mutate Company B's profile. This is the primary tenant isolation mechanism for company data.
- **E02-R-001 (RBAC ceiling):** `require_role("bid_manager")` ensures that `contributor`, `reviewer`, and `read_only` users (all below bid_manager threshold in the hierarchy) cannot modify company data. GET uses `get_current_user` only (no role minimum) per the epic spec.
- **Audit log append-only:** The DB-level INSERT-only grant on `shared.audit_log` (applied in migration 002) means the application role cannot UPDATE or DELETE audit rows. This is enforced at the PostgreSQL role level, not application code.

### Previous Story Intelligence (Story 2.7)

- `auth_service.py` now contains all auth flow functions including `request_password_reset()` and `confirm_password_reset()`
- `models/__init__.py` exports: `AuditLog`, `Company`, `CompanyMembership`, `EntityPermission`, `EntityPermissionEnum`, `ESPDProfile`, `PasswordResetToken`, `RefreshToken`, `Subscription`, `User`, `VerificationToken`
- `schemas/auth.py` contains: `RegisterRequest`, `UserResponse`, `CompanyResponse`, `RegisterResponse`, `LoginRequest`, `LoginResponse`, `CurrentUserResponse`, `RefreshRequest`, `LogoutRequest`, `PasswordResetRequestRequest`, `PasswordResetConfirmRequest`
- `dependencies.py` provides: `get_db_session`, `get_email_service_dep`, `get_redis_client`, `get_login_rate_limiter`
- **182 tests pass.** Story 2.8 must not regress any of these.
- Latest Alembic migration: `006_password_reset_tokens.py` (revision `"006"`). No new migration needed for Story 2.8.
- `ValueError` (not `eusolicit_common.ValidationError`) maps to HTTP 400 when caught in route handlers. `ForbiddenError` from `eusolicit_common.exceptions` maps to 403 via the registered exception handlers in `main.py` (`register_exception_handlers(app)`).
- The `CapturingEmailService` pattern in `test_auth_password_reset.py` is a clean way to stub external services in tests — adopt a similar pattern if needed.
- Company model column `updated_at` uses `onupdate=sa.func.now()` — no manual update required; SQLAlchemy includes it in the UPDATE statement automatically.

### References

- Epic story definition: [Source: eusolicit-docs/planning-artifacts/epic-02-authentication-identity.md#S02.08]
- Epic test design (company profile CRUD): [Source: eusolicit-docs/test-artifacts/test-design-epic-02.md] (E02-P1-014, E02-P1-018, E02-P2-004, E02-P2-005, E02-P2-006)
- Company ORM model: [Source: eusolicit-app/services/client-api/src/client_api/models/company.py]
- AuditLog ORM model: [Source: eusolicit-app/services/client-api/src/client_api/models/audit_log.py]
- CompanyMembership model: [Source: eusolicit-app/services/client-api/src/client_api/models/company_membership.py]
- Role hierarchy and `require_role`: [Source: eusolicit-app/services/client-api/src/client_api/core/security.py#require_role]
- Auth service (service layer pattern): [Source: eusolicit-app/services/client-api/src/client_api/services/auth_service.py]
- Auth API router (route handler pattern): [Source: eusolicit-app/services/client-api/src/client_api/api/v1/auth.py]
- Test conftest (fixture patterns, dependency overrides): [Source: eusolicit-app/services/client-api/tests/conftest.py]
- Password reset test (shared-session fixture pattern): [Source: eusolicit-app/services/client-api/tests/api/test_auth_password_reset.py]
- Project rules: [Source: eusolicit-docs/project-context.md]

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (claude-code)

### Debug Log References

- `session.refresh(company)` required after `session.flush()` to eagerly load server-computed `updated_at` before Pydantic serialization (avoids MissingGreenlet error)
- PostgreSQL `NOW()` returns transaction start time — both audit entries in the same transaction share the same timestamp; audit test query uses `after->>'name' = 'After Name'` filter instead of `ORDER BY timestamp DESC` to avoid ambiguity
- asyncpg rejects `::` PostgreSQL cast syntax after named params (e.g., `:role::client.company_role`); replaced with `CAST(:role AS client.company_role)` and `CAST(:company_id AS uuid)` throughout tests

### Completion Notes List

- All 7 acceptance criteria implemented and verified (28/28 tests pass)
- 210/210 total tests pass (0 regressions vs Story 2.7 baseline of 182)
- New files: `schemas/company.py`, `services/company_service.py`, `api/v1/companies.py`
- Modified files: `main.py` (router registration), `schemas/company.py` (null name guard + max_length for tax_id/industry), `tests/api/test_company_profile.py` (5 new tests for review findings)

### File List

- `services/client-api/src/client_api/schemas/company.py` (NEW)
- `services/client-api/src/client_api/services/company_service.py` (NEW)
- `services/client-api/src/client_api/api/v1/companies.py` (NEW)
- `services/client-api/src/client_api/main.py` (MODIFIED)
- `services/client-api/tests/api/test_company_profile.py` (MODIFIED)

## Senior Developer Review

**Date:** 2026-04-07
**Reviewer:** Code Review (Blind Hunter + Edge Case Hunter + Acceptance Auditor)
**Test Results:** 28/28 story tests pass, 210/210 full suite (0 regressions)
**Verdict:** Approved

### Review Findings

- [x] [Review][Patch] PATCH `{"name": null}` causes unhandled 500 IntegrityError [schemas/company.py:81] — Fixed: added `@field_validator("name")` to `CompanyProfilePatchRequest` that raises `ValueError` when the field is explicitly `null`, returning 422 instead of letting the `NOT NULL` DB constraint surface a 500.
- [x] [Review][Patch] Schema `tax_id` and `industry` missing `max_length` — DB column mismatch [schemas/company.py:53,55] — Fixed: added `Field(None, max_length=100)` to `tax_id` and `industry` in both `CompanyProfilePutRequest` and `CompanyProfilePatchRequest`.
- [x] [Review][Patch] No test for PATCH audit logging — AC6 gap [tests/api/test_company_profile.py] — Fixed: added `test_patch_creates_audit_log_entry_with_required_fields` to `TestAC7AuditLog`.
- [x] [Review][Patch] No test for unauthenticated PUT/PATCH returning 401 — AC7 gap [tests/api/test_company_profile.py] — Fixed: added `test_unauthenticated_put_returns_401` and `test_unauthenticated_patch_returns_401` to new `TestAC7UnauthAndCrossTenant` class.
- [x] [Review][Patch] No test for cross-tenant PUT/PATCH returning 403 — AC7 gap [tests/api/test_company_profile.py] — Fixed: added `test_cross_tenant_put_returns_403` and `test_cross_tenant_patch_returns_403` to `TestAC7UnauthAndCrossTenant`.
- [x] [Review][Defer] Duplicated CPV validator across PutRequest and PatchRequest [schemas/company.py:62-72,90-100] — deferred, cosmetic DRY concern; both validators are identical and in the same file; Pydantic v2 class-method validators cannot easily be shared without a mixin
- [x] [Review][Defer] `setattr` in PATCH loop has no allowlist of mutable fields [company_service.py:119] — deferred, defense-in-depth; currently safe because Pydantic schema fields exactly match ORM columns; consider adding `_PATCHABLE_FIELDS` frozenset when schema/ORM diverge
- [x] [Review][Defer] No concurrency control — lost update race condition [company_service.py:66,108] — deferred, architectural; no optimistic locking or `SELECT FOR UPDATE`; applies to all CRUD operations in the codebase; not specified in this story's requirements
- [x] [Review][Defer] Empty PATCH body creates no-op audit log entry [company_service.py:100-138] — deferred, cosmetic; AC6 says "that modifies"; empty `model_fields_set` produces identical before/after audit entries; Story 2.11 audit middleware will address
- [x] [Review][Defer] Missing size limits on `cpv_sectors`/`regions`/`certifications` lists and `description` length [schemas/company.py] — deferred, defense-in-depth; unbounded JSONB payloads risk storage bloat; not specified in story ACs
- [x] [Review][Defer] `ip_address` from `request.client.host` may be proxy IP, not real client [api/v1/companies.py:43] — deferred, pre-existing pattern shared with auth.py; proxy header handling is a codebase-wide concern
- [x] [Review][Defer] Empty list `[]` vs `None` semantic ambiguity in JSONB columns [schemas/company.py] — deferred, semantic design choice; `cpv_sectors: []` stored as JSON array vs `None` as SQL NULL; not addressed in story spec
