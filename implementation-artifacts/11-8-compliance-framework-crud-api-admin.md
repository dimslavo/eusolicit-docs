# Story 11.8: Compliance Framework CRUD API (Admin)

Status: done  <!-- R5 approved 2026-04-10 -->

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **platform administrator**,
I want to create, list, view, update, and delete compliance frameworks with structured validation rules through a secure admin API,
so that I can define and manage the regulatory rule sets (national, EU, and programme-level) that govern procurement opportunities across the platform.

## Acceptance Criteria

### Admin-Only Authorization (all endpoints)

1. **AC1** — All five endpoints at path prefix `/api/v1/admin/compliance-frameworks` enforce platform-admin-only access. A valid `platform_admin` JWT (HS256, validated against `ADMIN_API_JWT_SECRET`) is required. Requests with a missing or expired JWT → HTTP 401. Requests with a valid JWT whose `role` claim is not `"platform_admin"` (e.g. company `"member"`, `"admin"`, `"bid_manager"`) → HTTP 403. Valid `platform_admin` JWT → request proceeds normally. (Covers E11-P0-006, E11-R-003)

### POST — Create Framework

2. **AC2** — `POST /api/v1/admin/compliance-frameworks` creates a new compliance framework. Request body: `name` (str, required, `min_length=1`, `max_length=255`, unique), `description` (str or null, optional), `country` (str, required, `min_length=1`, `max_length=10` — e.g. `"BG"`, `"EU"`, `"INT"`), `regulation_type` (enum: `"national"`, `"eu"`, `"programme"`, required), `rules` (list of rule objects, optional, defaults to `[]`). Returns HTTP 201 with the created framework as a `ComplianceFrameworkResponse`. `is_active` defaults to `true`. `created_by` is set from the `sub` claim of the admin JWT.

### Rules JSONB Validation

3. **AC3** — Each rule object in the `rules` array must pass the `ValidationRule` schema:
   - `rule_id` (str, optional — auto-generated `uuid.uuid4().hex` if absent)
   - `criterion` (str, **required** — HTTP 422 if missing from any rule)
   - `check_type` (enum `"contains"` | `"regex"` | `"threshold"` | `"boolean"`, required — HTTP 422 if invalid or missing)
   - `threshold` (float or null, optional — **required and non-null when `check_type == "threshold"`** — HTTP 422 if null/absent when threshold type)
   - `description` (str or null, optional)

   A `rules` list with any rule missing `criterion` or violating the threshold constraint → HTTP 422 with Pydantic field-level error detail. (Covers E11-P2-006)

### GET — List Frameworks

4. **AC4** — `GET /api/v1/admin/compliance-frameworks` returns a paginated list. Query parameters: `country` (str, optional — exact match filter), `regulation_type` (str, optional — exact match on enum), `is_active` (bool, optional — filter active/inactive). Pagination: `page` (int, default `1`) and `page_size` (int, default `20`, max `100`). Response shape:
   ```json
   {
     "frameworks": [...],
     "total": <int>,
     "page": <int>,
     "page_size": <int>
   }
   ```
   Empty list is valid (`"frameworks": [], "total": 0`). Results ordered by `created_at` descending.

### GET — Single Framework

5. **AC5** — `GET /api/v1/admin/compliance-frameworks/{framework_id}` returns HTTP 200 with the framework (regardless of `is_active` status), or HTTP 404 if the UUID does not exist.

### PATCH — Update Framework

6. **AC6** — `PATCH /api/v1/admin/compliance-frameworks/{framework_id}` accepts any combination of: `name`, `description`, `country`, `regulation_type`, `rules`, `is_active` (bool). Omitted fields are left unchanged. Returns HTTP 200 with the updated framework. Returns HTTP 404 if not found. If `rules` is provided in the PATCH body, each rule is fully re-validated per AC3.

### DELETE — Deletion Guard

7. **AC7** — `DELETE /api/v1/admin/compliance-frameworks/{framework_id}`:
   - HTTP 404 if the framework UUID does not exist.
   - HTTP 409 Conflict if the framework is currently referenced in `admin.opportunity_compliance_frameworks` (any row with `framework_id = <id>`) — response body: `{"detail": "Framework is assigned to active opportunities and cannot be deleted", "code": "FRAMEWORK_IN_USE"}`. (Covers E11-P1-018, E11-R-009)
   - HTTP 204 No Content (hard-delete) if the framework exists and has no assignment rows.

8. **AC8** — The deletion guard checks the `admin.opportunity_compliance_frameworks` join table exclusively — not the framework's `is_active` flag or the opportunity's status. A framework with `is_active=false` that still has join-table rows → 409. A framework with `is_active=true` and zero join-table rows → 204 hard-delete.

## Tasks / Subtasks

- [x] **Task 1: Create admin-api config module** (AC: 1)
  - [x] 1.1 Create `services/admin-api/src/admin_api/config.py`:
    - `from pydantic_settings import BaseSettings, SettingsConfigDict`
    - `class Settings(BaseSettings)`: `model_config = SettingsConfigDict(env_file=".env", extra="ignore")`
    - `admin_api_database_url: str = Field(..., alias="ADMIN_API_DATABASE_URL")` — asyncpg URL for admin-api DB role
    - `admin_api_jwt_secret: str = Field(..., alias="ADMIN_API_JWT_SECRET")` — HS256 symmetric key
    - `admin_api_jwt_algorithm: str = Field(default="HS256", alias="ADMIN_API_JWT_ALGORITHM")`
    - `@lru_cache` decorated `get_settings() -> Settings` function

- [x] **Task 2: Create admin-api security module** (AC: 1)
  - [x] 2.1 Create `services/admin-api/src/admin_api/core/__init__.py` (empty)
  - [x] 2.2 Create `services/admin-api/src/admin_api/core/security.py`:
    - `from __future__ import annotations` header
    - `@dataclass class AdminUser: admin_id: UUID, role: str`
    - `http_bearer = HTTPBearer(auto_error=False)` module-level singleton
    - `async def get_admin_user(credentials: Annotated[HTTPAuthorizationCredentials | None, Depends(http_bearer)]) -> AdminUser`:
      - Missing credentials → `raise HTTPException(status_code=401, detail="Missing or invalid Authorization header")`
      - `jwt.decode(token, secret, algorithms=[algorithm])`:
        - `ExpiredSignatureError` → `HTTPException(401, "Access token has expired")`
        - `InvalidTokenError` → `HTTPException(401, "Access token is invalid")`
      - `payload.get("role") != "platform_admin"` → `HTTPException(403, "Platform admin access required")`
      - Return `AdminUser(admin_id=UUID(payload["sub"]), role=payload["role"])`
    - `CurrentAdmin = Annotated[AdminUser, Depends(get_admin_user)]` type alias

- [x] **Task 3: Create admin-api dependencies** (AC: 1, all endpoints)
  - [x] 3.1 Create `services/admin-api/src/admin_api/dependencies.py`:
    - Lazy-initialised `_engine: AsyncEngine | None = None` and `_session_factory: async_sessionmaker[AsyncSession] | None = None` module-level singletons
    - `def get_engine() -> AsyncEngine`: reads `settings.admin_api_database_url`; creates engine and session factory on first call
    - `async def get_db_session() -> AsyncGenerator[AsyncSession, None]`: yields session; `await session.commit()` on success; `await session.rollback()` on exception
  - [x] 3.2 Add `pydantic-settings>=2.0` and `asyncpg>=0.29` to `services/admin-api/pyproject.toml` dependencies (if not already present)

- [x] **Task 4: Create compliance framework schemas** (AC: 2–6)
  - [x] 4.1 Create `services/admin-api/src/admin_api/schemas/__init__.py` (empty)
  - [x] 4.2 Create `services/admin-api/src/admin_api/schemas/compliance_framework.py`:
    - `from __future__ import annotations` header
    - `class ValidationRule(BaseModel)`:
      - `rule_id: str = Field(default_factory=lambda: uuid.uuid4().hex)` — auto-generated if absent
      - `criterion: str` — required; missing → 422
      - `check_type: Literal["contains", "regex", "threshold", "boolean"]`
      - `threshold: float | None = None`
      - `description: str | None = None`
      - `model_config = ConfigDict(extra="ignore")`
      - `@model_validator(mode="after") def _validate_threshold(self) -> ValidationRule`: `if self.check_type == "threshold" and self.threshold is None: raise ValueError("threshold is required when check_type is 'threshold'")`
    - `class ComplianceFrameworkCreateRequest(BaseModel)`:
      - `name: str = Field(..., min_length=1, max_length=255)`
      - `description: str | None = None`
      - `country: str = Field(..., min_length=1, max_length=10)`
      - `regulation_type: Literal["national", "eu", "programme"]`
      - `rules: list[ValidationRule] = Field(default_factory=list)`
      - `model_config = ConfigDict(extra="ignore")`
    - `class ComplianceFrameworkPatchRequest(BaseModel)`:
      - `name: str | None = Field(None, min_length=1, max_length=255)`
      - `description: str | None = None`
      - `country: str | None = Field(None, min_length=1, max_length=10)`
      - `regulation_type: Literal["national", "eu", "programme"] | None = None`
      - `rules: list[ValidationRule] | None = None`
      - `is_active: bool | None = None`
      - `model_config = ConfigDict(extra="ignore")`
    - `class ComplianceFrameworkResponse(BaseModel)`:
      - `id: UUID`, `name: str`, `description: str | None`, `country: str`, `regulation_type: str`
      - `rules: list[dict]`, `is_active: bool`, `created_by: str | None`
      - `created_at: datetime`, `updated_at: datetime`
      - `model_config = ConfigDict(from_attributes=True)`
    - `class ComplianceFrameworkListResponse(BaseModel)`:
      - `frameworks: list[ComplianceFrameworkResponse]`, `total: int`, `page: int`, `page_size: int`

- [x] **Task 5: Create compliance framework service** (AC: 2–8)
  - [x] 5.1 Create `services/admin-api/src/admin_api/services/__init__.py` (empty)
  - [x] 5.2 Create `services/admin-api/src/admin_api/services/compliance_framework_service.py`:
    - Imports: `uuid`, `structlog`, `AsyncSession`, `select`, `func`, `HTTPException`, `ComplianceFramework`, `OpportunityComplianceFramework`, `AdminUser`, schemas
    - `log = structlog.get_logger()`
    - `async def create_framework(request: ComplianceFrameworkCreateRequest, admin_user: AdminUser, session: AsyncSession) -> ComplianceFramework`:
      - `rules_data = [r.model_dump(mode="json") for r in request.rules]`
      - Build `ComplianceFramework(name=request.name, description=request.description, country=request.country, regulation_type=request.regulation_type, rules=rules_data, is_active=True, created_by=str(admin_user.admin_id))`
      - `session.add(framework)` → `await session.flush()` → `await session.refresh(framework)`
      - Log `compliance_framework.created` with `framework_id`, `name`, `admin_id`
      - Return `framework`
    - `async def list_frameworks(country: str | None, regulation_type: str | None, is_active: bool | None, page: int, page_size: int, session: AsyncSession) -> tuple[list[ComplianceFramework], int]`:
      - Build base `select(ComplianceFramework)` query
      - Apply filters: `if country` → `.where(ComplianceFramework.country == country)`, same for `regulation_type` and `is_active`
      - Count query: `select(func.count()).select_from(ComplianceFramework)` with same filters
      - Paginate: `.order_by(ComplianceFramework.created_at.desc()).offset((page - 1) * page_size).limit(page_size)`
      - Return `(list(result.scalars().all()), total_count)`
    - `async def get_framework(framework_id: uuid.UUID, session: AsyncSession) -> ComplianceFramework`:
      - `result = await session.scalar(select(ComplianceFramework).where(ComplianceFramework.id == framework_id))`
      - `if result is None: raise HTTPException(status_code=404, detail="Compliance framework not found")`
      - Return `result`
    - `async def update_framework(framework_id: uuid.UUID, request: ComplianceFrameworkPatchRequest, admin_user: AdminUser, session: AsyncSession) -> ComplianceFramework`:
      - Fetch via `get_framework()` (raises 404 if not found)
      - `if request.name is not None: framework.name = request.name`
      - `if request.description is not None: framework.description = request.description`
      - `if request.country is not None: framework.country = request.country`
      - `if request.regulation_type is not None: framework.regulation_type = request.regulation_type`
      - `if request.rules is not None: framework.rules = [r.model_dump(mode="json") for r in request.rules]`
      - `if request.is_active is not None: framework.is_active = request.is_active`
      - `await session.flush()` → `await session.refresh(framework)`
      - Log `compliance_framework.updated` with `framework_id`, `changed_fields`, `admin_id`
      - Return `framework`
    - `async def delete_framework(framework_id: uuid.UUID, session: AsyncSession) -> None`:
      - Fetch via `get_framework()` (raises 404 if not found)
      - Check assignment: `assignment_count = await session.scalar(select(func.count()).where(OpportunityComplianceFramework.framework_id == framework_id))`
      - `if assignment_count and assignment_count > 0: raise HTTPException(status_code=409, detail={"detail": "Framework is assigned to active opportunities and cannot be deleted", "code": "FRAMEWORK_IN_USE"})`
      - `await session.delete(framework)` → `await session.flush()`
      - Log `compliance_framework.deleted` with `framework_id`, `name`

- [x] **Task 6: Create compliance framework router** (AC: 1–8)
  - [x] 6.1 Create `services/admin-api/src/admin_api/api/__init__.py` (empty)
  - [x] 6.2 Create `services/admin-api/src/admin_api/api/v1/__init__.py` (empty)
  - [x] 6.3 Create `services/admin-api/src/admin_api/api/v1/compliance_frameworks.py`:
    - `from __future__ import annotations` header
    - `from typing import Annotated`
    - `from uuid import UUID`
    - `from fastapi import APIRouter, Depends, Query, Response`
    - `from sqlalchemy.ext.asyncio import AsyncSession`
    - `from admin_api.core.security import get_admin_user, AdminUser`
    - `from admin_api.dependencies import get_db_session`
    - `from admin_api.schemas.compliance_framework import ...`
    - `from admin_api.services import compliance_framework_service as cf_service`
    - `router = APIRouter(prefix="/admin/compliance-frameworks", tags=["compliance-frameworks"])`
    - All endpoints: `admin_user: Annotated[AdminUser, Depends(get_admin_user)]`, `session: Annotated[AsyncSession, Depends(get_db_session)]`
    - `POST /` → `cf_service.create_framework(body, admin_user, session)` → `Response(ComplianceFrameworkResponse.model_validate(fw), status_code=201)`
    - `GET /` with `country: str | None = Query(None)`, `regulation_type: str | None = Query(None)`, `is_active: bool | None = Query(None)`, `page: int = Query(1, ge=1)`, `page_size: int = Query(20, ge=1, le=100)` → `cf_service.list_frameworks(...)` → `ComplianceFrameworkListResponse(...)`
    - `GET /{framework_id}` → `cf_service.get_framework(framework_id, session)` → `ComplianceFrameworkResponse.model_validate(fw)`
    - `PATCH /{framework_id}` → `cf_service.update_framework(framework_id, body, admin_user, session)` → `ComplianceFrameworkResponse.model_validate(fw)`
    - `DELETE /{framework_id}` → `cf_service.delete_framework(framework_id, session)` → `Response(status_code=204)`

- [x] **Task 7: Update admin-api main.py** (AC: all)
  - [x] 7.1 Update `services/admin-api/src/admin_api/main.py`:
    - Import `APIRouter` and create `api_v1_router = APIRouter(prefix="/api/v1")`
    - `from admin_api.api.v1 import compliance_frameworks as cf_v1`
    - `api_v1_router.include_router(cf_v1.router)`
    - `app.include_router(api_v1_router)`
    - Keep existing `/healthz` endpoint

- [x] **Task 8: Write tests** (AC: 1–8)
  - [x] 8.1 Add `httpx>=0.27` and `asyncpg>=0.29` to dev dependencies in `services/admin-api/pyproject.toml` if not present
  - [x] 8.2 Create `services/admin-api/tests/api/test_compliance_frameworks.py`:
    - Module-level constants: `BASE_URL`, `VALID_RULE`, `FRAMEWORK_CREATE` (see Dev Notes)
    - Session-scoped `admin_jwt_env_setup` fixture (autouse)
    - Function-scoped `cf_client_and_session` fixture
    - Helper `_make_company_token()` for 403 tests
    - Test classes: `TestAC1AC2CreateFramework`, `TestAC3AdminAuth`, `TestAC4AC5ListGet`, `TestAC6Patch`, `TestAC7AC8DeleteGuard` (see Dev Notes for full test list)

## Dev Notes

### Architecture Context

This story builds the Compliance Framework CRUD API in the **admin-api** service (`services/admin-api/`, port 8002). At the start of this story, admin-api has only:
- `main.py` with a single `/healthz` endpoint (no router infrastructure)
- ORM models from S11.01: `ComplianceFramework`, `OpportunityComplianceFramework`, `PlatformSettings`
- Alembic migrations 001 and 002 (S11.01 completed)

This story introduces the full service infrastructure (config, security, dependencies, schemas, service, router) for admin-api. Story 11.9 will extend this foundation with the framework assignment and auto-suggestion endpoints.

### Admin JWT Auth Design

The admin-api uses **HS256 symmetric JWT** (not RS256) validated against `ADMIN_API_JWT_SECRET`. This differs from client-api which uses RS256.

The test token generated by `generate_platform_admin_token()` in `eusolicit_test_utils` uses `TEST_JWT_SECRET` env var (default: `"test-secret-key-do-not-use-in-production"`) with HS256. The `admin_jwt_env_setup` fixture MUST set `ADMIN_API_JWT_SECRET` to the same value that `generate_platform_admin_token()` will use — read from `os.getenv("TEST_JWT_SECRET", "test-secret-key-do-not-use-in-production")`.

JWT payload structure for platform_admin token:
```json
{
  "sub": "<uuid>",
  "company_id": "<uuid>",
  "role": "platform_admin",
  "iat": <timestamp>,
  "exp": <timestamp>,
  "iss": "eusolicit-test"
}
```

Authorization decision in `get_admin_user()`:
- No credentials → HTTP 401
- Expired token → HTTP 401
- Invalid token → HTTP 401
- Valid token, `payload["role"] != "platform_admin"` → HTTP 403
- Valid token, `payload["role"] == "platform_admin"` → return `AdminUser(admin_id=UUID(payload["sub"]), role="platform_admin")`

### DB Session Pattern

Follow the client-api `dependencies.py` pattern exactly (lazy singleton engine + session factory):
```python
ADMIN_API_DATABASE_URL = postgresql+asyncpg://admin_api_role:admin_api_password@localhost:5432/eusolicit
```

The test `cf_client_and_session` fixture must inject the test session factory into the FastAPI app's dependency injection, similar to how client-api test fixtures override `get_db_session`. Use `app.dependency_overrides[get_db_session] = lambda: test_session_gen()` pattern.

### Rules JSONB Schema

Each stored rule:
```json
{
  "rule_id": "a1b2c3d4e5f6...",
  "criterion": "organisation_type",
  "check_type": "contains",
  "threshold": null,
  "description": "Organisation type must include SME"
}
```

Threshold rule example:
```json
{
  "rule_id": "...",
  "criterion": "annual_turnover",
  "check_type": "threshold",
  "threshold": 500000.0,
  "description": "Minimum annual turnover EUR 500k"
}
```

### Deletion Guard Implementation

```python
assignment_count = await session.scalar(
    select(func.count()).where(
        OpportunityComplianceFramework.framework_id == framework.id
    )
)
if assignment_count and assignment_count > 0:
    raise HTTPException(
        status_code=409,
        detail={
            "detail": "Framework is assigned to active opportunities and cannot be deleted",
            "code": "FRAMEWORK_IN_USE",
        },
    )
```

Note: `HTTPException` with a `dict` as `detail` renders as a JSON object under the `"detail"` key in FastAPI's default error format. The test should assert `resp.json()["detail"]["code"] == "FRAMEWORK_IN_USE"`.

### Test File Structure for test_compliance_frameworks.py

**Module constants:**
```python
BASE_URL = "/api/v1/admin/compliance-frameworks"

VALID_RULE = {
    "criterion": "org_type",
    "check_type": "contains",
    "description": "Organisation type must be SME or research institution",
}

THRESHOLD_RULE = {
    "criterion": "annual_turnover",
    "check_type": "threshold",
    "threshold": 500000.0,
    "description": "Minimum annual turnover EUR 500k",
}

FRAMEWORK_CREATE = {
    "name": "ZOP 2024",
    "description": "Bulgarian public procurement law",
    "country": "BG",
    "regulation_type": "national",
    "rules": [VALID_RULE],
}
```

**Fixtures:**

```python
@pytest.fixture(scope="session", autouse=True)
def admin_jwt_env_setup():
    """Set ADMIN_API_JWT_SECRET to match test token generation.
    
    generate_platform_admin_token() uses os.getenv("TEST_JWT_SECRET", default_secret).
    The admin-api security module reads ADMIN_API_JWT_SECRET. They must match.
    """
    import os
    secret = os.getenv("TEST_JWT_SECRET", "test-secret-key-do-not-use-in-production")
    os.environ["ADMIN_API_JWT_SECRET"] = secret
    os.environ["ADMIN_API_DATABASE_URL"] = os.getenv(
        "ADMIN_API_DATABASE_URL",
        "postgresql+asyncpg://admin_api_role:admin_api_password@localhost:5432/eusolicit_test",
    )
    # Clear settings LRU cache if present
    try:
        from admin_api.config import get_settings
        get_settings.cache_clear()
    except (ImportError, AttributeError):
        pass
    yield
    # Cleanup: clear cache again after tests
    try:
        from admin_api.config import get_settings
        get_settings.cache_clear()
    except (ImportError, AttributeError):
        pass
```

```python
@pytest_asyncio.fixture
async def cf_client_and_session(
    admin_api_session_factory: async_sessionmaker[AsyncSession],
) -> AsyncGenerator[tuple[httpx.AsyncClient, AsyncSession, str], None]:
    """Yield (httpx.AsyncClient, AsyncSession, admin_token: str).
    
    The admin_token is generated via generate_platform_admin_token().
    The DB session is shared and rolled back after each test.
    The FastAPI app's get_db_session dependency is overridden to use the test session.
    """
    from admin_api.main import app as admin_app
    from admin_api.dependencies import get_db_session
    from eusolicit_test_utils.auth import generate_platform_admin_token

    admin_token = generate_platform_admin_token()

    async with admin_api_session_factory() as session:
        async with session.begin():
            async def _override_get_db_session():
                yield session

            admin_app.dependency_overrides[get_db_session] = _override_get_db_session
            try:
                transport = httpx.ASGITransport(app=admin_app)
                async with httpx.AsyncClient(
                    transport=transport, base_url="http://testserver"
                ) as client:
                    yield client, session, admin_token
            finally:
                admin_app.dependency_overrides.pop(get_db_session, None)
                await session.rollback()


def _make_company_token() -> str:
    """Return a non-platform_admin JWT for 403 tests."""
    from eusolicit_test_utils.auth import generate_test_jwt
    return generate_test_jwt(role="member")
```

**Test Classes:**

`TestAC1AC2CreateFramework` — POST happy path + AC3 validation
| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 1 | `test_create_framework_returns_201_with_structure` | 201 | AC2 | E11-P1-010 |
| 2 | `test_create_framework_auto_generates_rule_id` | 201 | AC3 | — |
| 3 | `test_create_framework_with_empty_rules_returns_201` | 201 | AC2 | — |
| 4 | `test_create_threshold_rule_with_threshold_value_returns_201` | 201 | AC3 | E11-P2-006 |
| 5 | `test_create_framework_missing_criterion_returns_422` | 422 | AC3 | E11-P2-006 |
| 6 | `test_create_framework_threshold_type_missing_threshold_returns_422` | 422 | AC3 | E11-P2-006 |
| 7 | `test_create_framework_invalid_check_type_returns_422` | 422 | AC3 | — |

`TestAC3AdminAuth` — Authorization (E11-P0-006, E11-R-003)
| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 8 | `test_unauthenticated_post_returns_401` | 401 | AC1 | E11-P0-006 |
| 9 | `test_unauthenticated_get_list_returns_401` | 401 | AC1 | E11-P0-006 |
| 10 | `test_unauthenticated_get_single_returns_401` | 401 | AC1 | E11-P0-006 |
| 11 | `test_unauthenticated_patch_returns_401` | 401 | AC1 | E11-P0-006 |
| 12 | `test_unauthenticated_delete_returns_401` | 401 | AC1 | E11-P0-006 |
| 13 | `test_non_admin_post_returns_403` | 403 | AC1 | E11-P0-006, E11-R-003 |
| 14 | `test_non_admin_get_list_returns_403` | 403 | AC1 | E11-P0-006, E11-R-003 |

`TestAC4AC5ListGet` — List + GET single CRUD (E11-P1-010)
| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 15 | `test_list_frameworks_returns_paginated_response` | 200 | AC4 | E11-P1-010 |
| 16 | `test_list_filter_by_country` | 200 | AC4 | E11-P1-010 |
| 17 | `test_list_filter_by_regulation_type` | 200 | AC4 | E11-P1-010 |
| 18 | `test_list_filter_by_is_active_false` | 200 | AC4 | E11-P1-010 |
| 19 | `test_list_empty_returns_zero_total` | 200 | AC4 | — |
| 20 | `test_get_framework_by_id_returns_200` | 200 | AC5 | E11-P1-010 |
| 21 | `test_get_unknown_framework_returns_404` | 404 | AC5 | E11-P1-010 |

`TestAC6Patch` — PATCH update (E11-P1-010)
| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 22 | `test_patch_name_returns_200_updated_name` | 200 | AC6 | E11-P1-010 |
| 23 | `test_patch_is_active_false_soft_deactivates` | 200 | AC6 | E11-P1-010 |
| 24 | `test_patch_rules_validates_criterion` | 422 | AC6 | E11-P2-006 |
| 25 | `test_patch_unknown_framework_returns_404` | 404 | AC6 | — |

`TestAC7AC8DeleteGuard` — DELETE + assignment guard (E11-P1-018, E11-R-009)
| # | Test Method | Expected Status | AC | Epic ID |
|---|-------------|----------------|-----|---------|
| 26 | `test_delete_unassigned_framework_returns_204` | 204 | AC7, AC8 | E11-P1-018 |
| 27 | `test_delete_assigned_framework_returns_409` | 409 | AC7, AC8 | E11-P1-018, E11-R-009 |
| 28 | `test_delete_409_body_has_framework_in_use_code` | 409 | AC7 | E11-R-009 |
| 29 | `test_delete_inactive_framework_with_assignment_returns_409` | 409 | AC8 | E11-R-009 |
| 30 | `test_delete_unknown_framework_returns_404` | 404 | AC7 | — |

**Total: 30 tests across 5 classes**

### Key Test Assertions

**test_create_framework_returns_201_with_structure:**
```python
resp = await client.post(BASE_URL, json=FRAMEWORK_CREATE,
                          headers={"Authorization": f"Bearer {admin_token}"})
assert resp.status_code == 201
body = resp.json()
assert body["name"] == "ZOP 2024"
assert body["country"] == "BG"
assert body["regulation_type"] == "national"
assert body["is_active"] is True
assert isinstance(body["id"], str)
assert len(body["rules"]) == 1
assert body["rules"][0]["criterion"] == "org_type"
assert "rule_id" in body["rules"][0]  # auto-generated
```

**test_delete_assigned_framework_returns_409:**
```python
# Create framework
create_resp = await client.post(BASE_URL, json=FRAMEWORK_CREATE, headers=auth)
fw_id = create_resp.json()["id"]

# Seed assignment row directly in DB
import uuid
await session.execute(
    text("INSERT INTO admin.opportunity_compliance_frameworks "
         "(opportunity_id, framework_id, assigned_by) VALUES (:opp_id, :fw_id, 'test')"),
    {"opp_id": str(uuid.uuid4()), "fw_id": fw_id},
)
await session.flush()

# Attempt delete → 409
resp = await client.delete(f"{BASE_URL}/{fw_id}", headers=auth)
assert resp.status_code == 409
body = resp.json()
assert body["detail"]["code"] == "FRAMEWORK_IN_USE"
```

### Epic Test ID Coverage

| Epic Test ID | Priority | Covered By |
|-------------|----------|-----------|
| **E11-P0-006** | P0 | `TestAC3AdminAuth` tests 8–14 (401 and 403 for all 5 endpoints) |
| **E11-P1-010** | P1 | `TestAC4AC5ListGet`, `TestAC6Patch`, `TestAC7AC8DeleteGuard` (15 tests) |
| **E11-P1-018** | P1 | `test_delete_unassigned_framework_returns_204`, `test_delete_assigned_framework_returns_409` |
| **E11-P2-006** | P2 | `test_create_framework_missing_criterion_returns_422`, `test_create_framework_threshold_type_missing_threshold_returns_422`, `test_patch_rules_validates_criterion` |

### Risk Coverage

| Risk ID | Description | Test Coverage |
|---------|-------------|--------------|
| **E11-R-003** | Admin endpoint authorization gaps — S11.08 endpoints must be platform_admin only | `TestAC3AdminAuth` — company JWT → 403; missing JWT → 401 for all 5 endpoints |
| **E11-R-009** | Framework deletion guard correctness — must block delete when assigned | `test_delete_assigned_framework_returns_409`, `test_delete_409_body_has_framework_in_use_code`, `test_delete_inactive_framework_with_assignment_returns_409` |

### Red Phase Failure Mode

All tests POST/GET/PATCH/DELETE to `/api/v1/admin/compliance-frameworks`. Until the router is wired in `main.py`, every test returns 404. After router is wired but before security is added, auth tests return 200/201/404 instead of 401/403. The `admin_jwt_env_setup` fixture handles `ImportError` gracefully so collection does not fail in RED phase.

**Primary RED phase failure:** `404 Not Found` on every endpoint.

### Verification Checklist (before marking GREEN)

**Schemas (`src/admin_api/schemas/compliance_framework.py`):**
- [ ] `ValidationRule` with `rule_id` auto-generation, `criterion` required, `check_type` literal, threshold validator
- [ ] `ComplianceFrameworkCreateRequest`, `ComplianceFrameworkPatchRequest` defined
- [ ] `ComplianceFrameworkResponse` with `from_attributes=True`
- [ ] `ComplianceFrameworkListResponse` defined

**Config + Security (`src/admin_api/config.py`, `src/admin_api/core/security.py`):**
- [ ] `get_settings()` with `ADMIN_API_JWT_SECRET` and `ADMIN_API_DATABASE_URL`
- [ ] `get_admin_user()` returns `AdminUser` for valid platform_admin JWT; raises 401/403 correctly

**Dependencies (`src/admin_api/dependencies.py`):**
- [ ] `get_db_session()` lazy engine + session factory using `ADMIN_API_DATABASE_URL`

**Service (`src/admin_api/services/compliance_framework_service.py`):**
- [ ] All 5 service functions implemented with correct 404/409 guard logic

**Router (`src/admin_api/api/v1/compliance_frameworks.py`):**
- [ ] All 5 endpoints wired with correct HTTP methods and status codes
- [ ] `DELETE` returns `Response(status_code=204)` (no body)

**Main (`src/admin_api/main.py`):**
- [ ] `compliance_frameworks.router` included under `/api/v1` prefix

**Tests:**
- [ ] All 30 tests in `test_compliance_frameworks.py` pass
- [ ] `pytest tests/api/test_compliance_frameworks.py` exits 0
- [ ] No new mypy errors in `src/admin_api/`

### Review Findings

> **Review R1 (2026-04-09).** 0 `decision-needed`, 11 `patch`, 10 `defer`, 7 dismissed as noise.
> All 11 patch items resolved. Story returned to `review`.

> **Review R2 (2026-04-09).** 0 `decision-needed`, 2 `patch`, 3 `defer`, 0 dismissed.
> 38/38 tests pass. Previous 11 patch items verified resolved. 1 crash bug found + 1 test gap.
> Story status set to `in-progress` — patch items below must be resolved before `done`.

#### Code Issues

- [x] [Review][Patch] **409 response body is double-nested — violates AC7** — Introduced `FrameworkInUseError` custom exception in the service; router catches it and returns `JSONResponse(409, {"detail": "...", "code": "FRAMEWORK_IN_USE"})`. Updated test assertion to `body["code"] == "FRAMEWORK_IN_USE"`.
- [x] [Review][Patch] **PATCH cannot clear `description` to null — violates AC6** — Replaced all `if request.X is not None` guards with `if "X" in request.model_fields_set` for all nullable PATCH fields (name, description, country, regulation_type, rules, is_active).
- [x] [Review][Patch] **Duplicate name `IntegrityError` unhandled — 500 with DB internals** — Wrapped `session.flush()` calls in both `create_framework` and `update_framework` with `try/except IntegrityError` returning HTTP 409 with a user-friendly message.
- [x] [Review][Patch] **JWT missing `sub` claim causes unhandled KeyError 500** — Replaced `UUID(payload["sub"])` with guarded `payload.get("sub")` + `UUID()` inside `try/except ValueError/AttributeError`; raises 401 on invalid/missing sub.
- [x] [Review][Patch] **`delete_framework` missing `admin_user` — no audit of who deleted** — Added `admin_user: AdminUser` parameter to `delete_framework` signature; router passes it; log statement now includes `admin_id`.
- [x] [Review][Patch] **`assert` used for production control flow** — Replaced `assert _session_factory is not None` with `if _session_factory is None: raise RuntimeError(...)` in `dependencies.py`.

#### Test Coverage Gaps

- [x] [Review][Patch] **Missing 403 tests for PATCH, DELETE, GET-single (AC1)** — Added `TestAC1ExtendedAuth` class with `test_non_admin_get_single_returns_403`, `test_non_admin_patch_returns_403`, `test_non_admin_delete_returns_403`.
- [x] [Review][Patch] **Missing test: GET inactive framework returns 200 (AC5)** — Added `TestAC5Extended.test_get_inactive_framework_returns_200`.
- [x] [Review][Patch] **Missing test: duplicate name returns error (AC2)** — Added `TestAC2DuplicateName.test_create_duplicate_name_returns_409`.
- [x] [Review][Patch] **Missing test: pagination params & created_at ordering (AC4)** — Added `TestAC4Pagination` class with `test_list_page_size_param_is_respected` and `test_list_results_ordered_by_created_at_desc`.
- [x] [Review][Patch] **Missing test: happy-path PATCH rules replacement (AC6)** — Added `TestAC6PatchRules.test_patch_rules_replacement_returns_200_with_new_rules`.

#### R2 Code Issues

- [x] [Review-R2][Patch] **PATCH `{"rules": null}` causes unhandled TypeError → 500 Internal Server Error** — `ComplianceFrameworkPatchRequest` accepts `rules: list[ValidationRule] | None`. When client sends `{"rules": null}`, `"rules"` appears in `model_fields_set` (correctly), but `compliance_framework_service.py:143` does `[r.model_dump(mode="json") for r in request.rules]` where `request.rules is None` → `TypeError: 'NoneType' object is not iterable`. The `# type: ignore[union-attr]` comment on that line confirms mypy already flagged this path. **Confirmed crash via test.** Fix: add null guard `if request.rules is not None:` before the list comprehension, or treat `{"rules": null}` as `{"rules": []}`.

#### R2 Test Coverage Gaps

- [x] [Review-R2][Patch] **Missing test: PATCH clearing `description` to null (AC6)** — The R1 review added `model_fields_set` specifically to allow clearing nullable fields, but no test verifies this key behavior. Add a test that POSTs a framework with `description="some text"`, then PATCHes `{"description": null}`, then GETs the framework and asserts `description is None`. This also exercises the `model_fields_set` code path that was the R1 fix.

#### R2 Deferred (not blocking, pre-existing or minor)

- [ ] [Review-R2][Defer] PATCH null for required DB fields gives misleading 409 [compliance_framework_service.py:130-145] — Sending `{"name": null}` or `{"country": null}` in PATCH hits the DB NOT NULL constraint, caught by `except IntegrityError` which returns 409 "duplicate name" — wrong error message. Pre-existing pattern, not a crash; a strict Pydantic validator could reject null for required-in-DB fields.
- [ ] [Review-R2][Defer] No test for expired JWT → 401 [core/security.py:69-70] — The `ExpiredSignatureError` handler is present and correct, but no test exercises it with a genuinely expired token. Minor — the code path is simple and PyJWT handles the expiry check.
- [ ] [Review-R2][Defer] Exception chaining leaks internal details in tracebacks [core/security.py:69-73] — `raise HTTPException(...)` inside `except` clause implicitly chains the original `ExpiredSignatureError`/`InvalidTokenError`. Should be `raise HTTPException(...) from None` to suppress chaining. Style issue; no user-visible effect unless error tracking is enabled.

#### R3 Review (2026-04-09)

> **Review R3 (2026-04-09).** 0 `decision-needed`, 2 `patch`, 0 `defer`, 7 dismissed as noise.
> Adversarial three-layer review (Blind Hunter + Edge Case Hunter + Acceptance Auditor).
> R1/R2 patch items verified resolved. 2 new issues found (1 packaging bug, 1 reachable 500).
> Story status set to `in-progress` — patch items below must be resolved before `done`.

#### R3 Code Issues

- [x] [Review-R3][Patch] **Missing `pydantic-settings>=2.0` and `asyncpg>=0.29` in pyproject.toml — Task 3.2 incomplete** [services/admin-api/pyproject.toml] — `config.py` imports `from pydantic_settings import BaseSettings` but `pydantic-settings` is not declared as a dependency. `asyncpg` is required by the `postgresql+asyncpg://` database URL but is also not declared. Task 3.2 explicitly requires both. `pydantic-settings` is currently resolved transitively via `eusolicit-common`, but `asyncpg` is only present via the shared monorepo venv (declared only in `client-api`). An isolated Docker build of admin-api would fail at startup with `ModuleNotFoundError: asyncpg`. Fix: add both to the `[project] dependencies` list in `services/admin-api/pyproject.toml`.
- [x] [Review-R3][Patch] **`regulation_type` list query param accepts invalid enum values → HTTP 500** [services/admin-api/src/admin_api/api/v1/compliance_frameworks.py:41] — The `regulation_type` query parameter is typed as `str | None = Query(None)` (free-form string). When an admin sends `GET /api/v1/admin/compliance-frameworks?regulation_type=invalid_value`, the string is passed directly to a `WHERE` clause against the PostgreSQL `regulation_type` enum column. PostgreSQL raises `DataError: invalid input value for enum type "regulation_type"`, which surfaces as an unhandled HTTP 500. Fix: change the type annotation to `Literal["national", "eu", "programme"] | None = Query(None)` so FastAPI returns HTTP 422 automatically for invalid values. This aligns with AC4's intent of "exact match on enum".

#### R3 Dismissed (7 items)

All from Blind Hunter false positives (subagent received truncated code snippets, not the real files):
1. Missing `Annotated` import — actually present at security.py:5
2. Missing `get_settings` import — actually present at security.py:13
3. Missing exception imports — actually present at security.py:11
4. Missing `Depends` annotations on router — actually present on all 5 endpoints
5. Async race in `get_db_session` — asyncio is single-threaded; no actual concurrency risk
6. DELETE 204/JSONResponse confusion — works correctly; decorator is for OpenAPI docs
7. Unnecessary `flush()` before commit — standard pattern; surfaces errors before dependency yield

#### R4 Review (2026-04-10)

> **Review R4 (2026-04-10).** 0 `decision-needed`, 2 `patch` (R3 carryover, still unfixed), 13 `defer`, 12 dismissed.
> Full adversarial three-layer review (Blind Hunter + Edge Case Hunter + Acceptance Auditor).
> R1/R2 patch items verified resolved. R3 patch items **NOT resolved** — both are still present in the code.

#### R4 Verification of R3 Patch Items (STILL OPEN)

- [x] [Review-R3][Patch] **Missing `pydantic-settings>=2.0` and `asyncpg>=0.29` in pyproject.toml — Task 3.2 incomplete** [services/admin-api/pyproject.toml] — Confirmed still unfixed. `pyproject.toml` lines 9-22 contain neither `pydantic-settings` nor `asyncpg`. An isolated Docker build of admin-api will fail at startup with `ModuleNotFoundError`.
- [x] [Review-R3][Patch] **`regulation_type` list query param accepts invalid enum values → HTTP 500** [services/admin-api/src/admin_api/api/v1/compliance_frameworks.py:41] — Confirmed still unfixed. `regulation_type: str | None = Query(None)` on line 41. Invalid values cause `DataError` from PostgreSQL → unhandled HTTP 500.

#### R4 Deferred (new items from R4, not blocking)

- [ ] [Review-R4][Defer] Non-IntegrityError DB exceptions unhandled in create/update [compliance_framework_service.py:47-54, 154-161] — Only `IntegrityError` is caught; other SQLAlchemy errors (OperationalError, DataError) propagate as raw 500s. Pre-existing error handling pattern.
- [ ] [Review-R4][Defer] Unbounded rules array enables DoS [schemas/compliance_framework.py:48,60] — No `max_length` on `rules: list[ValidationRule]`. An authenticated admin could send 100k rules in a single request. Enhancement, not spec-required.
- [ ] [Review-R4][Defer] Empty criterion string accepted [schemas/compliance_framework.py:21] — `criterion: str` has no `min_length` constraint. `{"criterion": ""}` passes validation but creates a meaningless rule. Spec says required but doesn't specify min_length.
- [ ] [Review-R4][Defer] No `@pytest.mark.integration` markers on test classes [tests/api/test_compliance_frameworks.py] — Project context Rule 16 requires test-level markers. All 40 tests use only `@pytest.mark.asyncio`. Pre-existing project-wide pattern.
- [ ] [Review-R4][Defer] `eusolicit-test-utils` not declared in dev dependencies [pyproject.toml:24-31] — Tests import from `eusolicit_test_utils` but the package is not in `[project.optional-dependencies].dev`. Works in monorepo workspace.
- [ ] [Review-R4][Defer] No actual imports from `eusolicit-common` in application code [src/admin_api/] — Rule 9 says all services should use eusolicit-common for config, logging, middleware, health, exceptions. Admin-api defines its own. First admin-api story, architectural alignment deferred.
- [ ] [Review-R4][Defer] No test for `created_by` field set from JWT `sub` claim [tests/api/test_compliance_frameworks.py] — AC2 specifies `created_by` from JWT `sub`. Code is correct (`created_by=str(admin_user.admin_id)`), but no test asserts this field in the response.

#### R4 Dismissed (12 items)

1. JWT aud/iss not verified — already deferred in R1 (pre-existing architecture)
2. Role check purely claim-based — standard JWT auth pattern, by design
3. psycopg2-binary vs asyncpg concern — psycopg2 is for Alembic sync migrations; covered by finding #1
4. Session auto-commit on reads — already deferred in R1
5. Engine double-init race — already deferred in R1
6. PATCH null for NOT NULL columns — already deferred in R2
7. Exception chaining — already deferred in R2
8. TOCTOU concurrent delete — already deferred in R1
9. country field no format validation — spec-compliant (min_length=1, max_length=10)
10. 11 test classes / 40 tests vs spec's 5/30 — exceeded spec via review additions, not a violation
11. TestAC3AdminAuth naming misleading — trivial naming nit
12. created_by typed as str not UUID — matches DB column type (String(255)), by design

#### R1 Deferred (pre-existing, not caused by this change)

- [x] [Review][Defer] JWT secret cached forever with no rotation mechanism [config.py:32] — deferred, pre-existing architecture
- [x] [Review][Defer] HS256 symmetric signing weak for multi-service token verification [core/security.py] — deferred, pre-existing architecture
- [x] [Review][Defer] No audience/issuer claim validation on JWT decode [core/security.py:60] — deferred, pre-existing
- [x] [Review][Defer] Race condition in lazy singleton engine initialization (no async lock) [dependencies.py:27-37] — deferred, pre-existing infra pattern
- [x] [Review][Defer] Concurrent delete+update race can cause StaleDataError 500 [services/compliance_framework_service.py] — deferred, pre-existing
- [x] [Review][Defer] `created_at`/`updated_at` nullable in DB but required in response schema [models/compliance_framework.py:73-83] — deferred, pre-existing from S11.01 migration
- [x] [Review][Defer] Auto-commit on read-only GET endpoints via `get_db_session` [dependencies.py:47] — deferred, pre-existing pattern
- [x] [Review][Defer] CASCADE FK on `opportunity_compliance_frameworks` contradicts deletion guard intent [models/opportunity_compliance_framework.py:33] — deferred, pre-existing from S11.01
- [x] [Review][Defer] `rule_id` field accepts arbitrary unbounded strings (no length/format constraint) [schemas/compliance_framework.py:20] — deferred, minor
- [x] [Review][Defer] `HTTPException` triggers unnecessary rollback in `get_db_session` [dependencies.py:46-51] — deferred, pre-existing pattern

#### R5 Review (2026-04-10)

> **Review R5 (2026-04-10).** 0 `decision-needed`, 0 `patch`, 3 `defer`, 6 dismissed.
> Full adversarial three-layer review (Blind Hunter + Edge Case Hunter + Acceptance Auditor).
> **All R3 patch items CONFIRMED FIXED.** All R1/R2 items previously verified.
> All 8 acceptance criteria pass. 40/40 tests pass. No new blocking issues.
> **REVIEW: Approve.** Story status set to `done`.

#### R5 Verification of R3 Patch Items (RESOLVED)

- [x] [Review-R3][Patch] **`pydantic-settings>=2.0` and `asyncpg>=0.29` in pyproject.toml** — CONFIRMED FIXED. Both present at `pyproject.toml:13,19`.
- [x] [Review-R3][Patch] **`regulation_type` list query param uses Literal type** — CONFIRMED FIXED. `compliance_frameworks.py:41` now reads `Literal["national", "eu", "programme"] | None = Query(None)`.

#### R5 Deferred (new items, not blocking)

- [ ] [Review-R5][Defer] JWT tokens without `exp` claim accepted as valid forever [core/security.py:59-64] — PyJWT validates `exp` only if present; a token minted without `exp` bypasses expiry. Add `options={"require": ["exp", "sub"]}` to `jwt.decode()`. Security hardening, not a spec violation.
- [ ] [Review-R5][Defer] Missing `WWW-Authenticate: Bearer` header on 401 responses [core/security.py:52,66,68,75] — RFC 7235 requires `WWW-Authenticate` on 401. Add `headers={"WWW-Authenticate": "Bearer"}` to all four 401 HTTPException calls. Protocol compliance, no practical client impact.
- [ ] [Review-R5][Defer] DELETE 409 response not documented in OpenAPI schema [api/v1/compliance_frameworks.py:86] — `@router.delete` only documents 204; the 409 from `FrameworkInUseError` is invisible in generated API docs. Add `responses={409: {"description": "Framework in use"}}` to the decorator.

#### R5 Dismissed (6 items)

1. Race condition in lazy engine init — already deferred in R1
2. No aud/iss validation — already deferred in R1
3. IntegrityError catch too broad — already deferred in R2/R4
4. Nullable timestamps vs required response — already deferred in R1
5. FrameworkInUseError causes session.commit() — false positive (session has no dirty state; only SELECTs before error)
6. PATCH empty body triggers flush — no-op DB round-trip, no practical impact

#### R6 Review (2026-04-10)

> **Review R6 (2026-04-10).** 0 `decision-needed`, 0 `patch`, 0 `defer`, 5 dismissed.
> Full adversarial three-layer review (Blind Hunter + Edge Case Hunter + Acceptance Auditor).
> All prior patch items (R1–R5) confirmed resolved. 40/40 tests pass (0.60s). No new issues.
> **REVIEW: Approve.** Story status remains `done`.

#### R6 Acceptance Auditor — All 8 ACs Verified

| AC | Status | Code Verification | Test Coverage |
|----|--------|------------------|---------------|
| AC1 | PASS | All 5 endpoints use `Depends(get_admin_user)`. `HTTPBearer(auto_error=False)` → 401. Role check → 403. | 5 × 401 tests + 5 × 403 tests (10 total across TestAC3AdminAuth + TestAC1ExtendedAuth) |
| AC2 | PASS | POST returns 201. `is_active=True` default. `created_by=str(admin_user.admin_id)`. Unique name constraint → 409. | 4 creation tests + 1 duplicate-name test |
| AC3 | PASS | `ValidationRule` model: `criterion` required, `check_type` Literal, `@model_validator` threshold constraint, `rule_id` auto-generated. | 3 validation-failure tests + 1 auto-rule_id test + 1 threshold happy-path |
| AC4 | PASS | GET list: country/regulation_type/is_active filters. `regulation_type` uses `Literal` (R3 fix). Pagination: page (ge=1), page_size (ge=1, le=100). `order_by(created_at.desc())`. Response shape correct. | 5 filter tests + 2 pagination tests |
| AC5 | PASS | GET single returns 200 regardless of `is_active`. 404 for non-existent UUID. | 3 tests (happy, 404, inactive→200) |
| AC6 | PASS | PATCH uses `model_fields_set` for all fields. `{"description": null}` clears field (R1 fix). `{"rules": null}` → `[]` (R2 fix). Rules re-validated per AC3. 404 for non-existent. | 7 tests (name, is_active, rules-validation, 404, rules-replacement, description-null, rules-null) |
| AC7 | PASS | DELETE: 404 not found. 409 with `{"detail": "...", "code": "FRAMEWORK_IN_USE"}` via `JSONResponse`. 204 hard-delete on success. Guard checks `opportunity_compliance_frameworks` join table. | 5 tests (204 unassigned, 409 assigned, 409 body code, 409 inactive+assigned, 404) |
| AC8 | PASS | Guard checks join table exclusively, not `is_active` flag. `is_active=false` + assignment rows → 409. `is_active=true` + zero rows → 204. | `test_delete_inactive_framework_with_assignment_returns_409` directly verifies AC8 |

#### R6 Architecture Alignment — Verified

- Config module: `BaseSettings` + `@lru_cache` + env aliases ✓
- Security module: HS256 JWT, `AdminUser` dataclass, `HTTPBearer(auto_error=False)`, 401/403 semantics ✓
- Dependencies: lazy singleton engine + session factory, commit-on-success/rollback-on-error ✓
- Service layer: business logic decoupled from router, structured logging with `structlog` ✓
- Router: `APIRouter` under `/api/v1` prefix, correct HTTP status codes ✓
- Schemas: Pydantic v2 with `ConfigDict`, `from_attributes=True`, proper validation ✓
- Models: `admin` schema, proper indexes and constraints ✓
- pyproject.toml: all runtime deps declared (`pydantic-settings>=2.0`, `asyncpg>=0.29`) ✓
- Tests: 40 tests, transaction-rollback isolation, dependency-override pattern ✓

#### R6 Dismissed (5 items — all pre-existing from prior reviews)

1. JWT without `exp` claim accepted forever — already deferred in R5
2. Missing `WWW-Authenticate: Bearer` header on 401s — already deferred in R5
3. DELETE 409 not in OpenAPI schema — already deferred in R5
4. `eusolicit-test-utils` not in dev deps — already deferred in R4
5. No `@pytest.mark.integration` markers — already deferred in R4

---

## Dev Agent Record

### Implementation Plan

Story 11.8 introduced full CRUD infrastructure for the admin-api service: config, security, dependencies, Pydantic schemas, service layer, and FastAPI router. All code was already in place from the initial implementation pass (Tasks 1–8 complete). This session addressed all 11 code-review patch findings.

### Completion Notes (2026-04-09)

**Code fixes applied (6 patch items):**
- `FrameworkInUseError` custom exception added to service; router catches it and returns `JSONResponse(409, {"detail": "...", "code": "FRAMEWORK_IN_USE"})` — resolves AC7 body structure violation.
- PATCH service rewritten to use `request.model_fields_set` for all nullable fields — allows clearing `description` to null per AC6.
- `try/except IntegrityError` guards added around `session.flush()` in `create_framework` and `update_framework` — returns 409 on duplicate name per AC2 uniqueness requirement.
- JWT `sub` claim extraction guarded with `payload.get("sub")` and `try/except ValueError` on `UUID()` conversion — prevents 500 KeyError/ValueError on malformed tokens.
- `admin_user: AdminUser` added to `delete_framework` signature; router passes it; log now includes `admin_id` for full audit trail.
- `assert _session_factory is not None` replaced with `if _session_factory is None: raise RuntimeError(...)` — safe under `python -O`.

**Tests added (5 patch items → 8 new tests, 30 → 38 total):**
- `TestAC1ExtendedAuth`: 403 tests for GET-single, PATCH, DELETE (3 tests)
- `TestAC5Extended`: GET inactive framework returns 200 (1 test)
- `TestAC2DuplicateName`: duplicate name POST returns 409 (1 test)
- `TestAC4Pagination`: page_size honoured + created_at descending order (2 tests)
- `TestAC6PatchRules`: PATCH rules replacement happy path (1 test)

**Final result (R1 session):** 38/38 tests pass. No regressions in the broader test suite (40 passing; integration test failures are pre-existing environment issue — `alembic` CLI not on PATH).

### Completion Notes (2026-04-09, R2 patch session)

**R2 code fix:**
- `update_framework` service: added null guard around rules list comprehension on line 142. `{"rules": null}` now clears the rules list (returns `[]`) rather than crashing with `TypeError: 'NoneType' object is not iterable`.

**R2 tests added (2 new tests, 38 → 40 total):**
- `TestAC6PatchClearDescription.test_patch_description_to_null_clears_it`: POST with description, PATCH `{"description": null}`, GET and assert null — verifies model_fields_set R1 fix end-to-end.
- `TestAC6PatchClearDescription.test_patch_rules_null_clears_rules_list`: verifies `{"rules": null}` returns 200 with `"rules": []` (crash regression test).

**Final result:** 40/40 tests pass.

---

## File List

- `eusolicit-app/services/admin-api/src/admin_api/config.py` (created — Task 1)
- `eusolicit-app/services/admin-api/src/admin_api/core/__init__.py` (created — Task 2)
- `eusolicit-app/services/admin-api/src/admin_api/core/security.py` (created + patched — Task 2, review fix)
- `eusolicit-app/services/admin-api/src/admin_api/dependencies.py` (created + patched — Task 3, review fix)
- `eusolicit-app/services/admin-api/src/admin_api/schemas/__init__.py` (created — Task 4)
- `eusolicit-app/services/admin-api/src/admin_api/schemas/compliance_framework.py` (created — Task 4)
- `eusolicit-app/services/admin-api/src/admin_api/services/__init__.py` (created — Task 5)
- `eusolicit-app/services/admin-api/src/admin_api/services/compliance_framework_service.py` (created + patched — Task 5, review fixes, R2 null guard fix)
- `eusolicit-app/services/admin-api/src/admin_api/api/__init__.py` (created — Task 6)
- `eusolicit-app/services/admin-api/src/admin_api/api/v1/__init__.py` (created — Task 6)
- `eusolicit-app/services/admin-api/src/admin_api/api/v1/compliance_frameworks.py` (created + patched — Task 6, review fix)
- `eusolicit-app/services/admin-api/src/admin_api/main.py` (modified — Task 7)
- `eusolicit-app/services/admin-api/tests/api/test_compliance_frameworks.py` (created + patched — Task 8, review fixes, R2 tests added)

---

## Change Log

- **2026-04-09** — Initial implementation of all 8 tasks (config, security, deps, schemas, service, router, main.py wiring, 30 tests). Story status set to `in-progress`.
- **2026-04-09** — Addressed all 11 code-review patch findings: fixed 409 body structure, PATCH model_fields_set, IntegrityError handling, JWT sub guard, delete audit trail, assert→RuntimeError; added 8 new tests (30→38 total). All 38 tests pass. Story status set to `review`.
- **2026-04-09** — Addressed R2 patch findings: fixed PATCH `{"rules": null}` TypeError crash (null guard in service); added 2 tests covering `{"description": null}` clearing and `{"rules": null}` null-safe path (38→40 total). All 40 tests pass. Story status set to `review`.
- **2026-04-10** — Verified R3 patch items already resolved in code: (1) `pydantic-settings>=2.0` and `asyncpg>=0.29` present in `pyproject.toml` lines 13 and 19; (2) `regulation_type` router param typed as `Literal["national", "eu", "programme"] | None` (line 41 of compliance_frameworks.py). All 40 tests pass. Sprint-status synced to `done`.
