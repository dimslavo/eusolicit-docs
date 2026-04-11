# Story 11.9: Framework Assignment & Auto-Suggestion API (Admin)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **platform administrator**,
I want to assign compliance frameworks to procurement opportunities, view and remove those assignments, trigger the Framework Suggestion Agent to auto-suggest applicable frameworks for new opportunities, and accept or reject AI-generated suggestions,
so that each opportunity has the correct regulatory frameworks applied efficiently — with AI assistance for discovery and full admin control for final decisions.

## Acceptance Criteria

### Admin-Only Authorization (all endpoints in this story)

1. **AC1** — All endpoints in this story enforce platform-admin-only access using the existing `get_admin_user` dependency from S11.08 (`services/admin-api/src/admin_api/core/security.py`). A missing or expired JWT → HTTP 401. A valid JWT with `role != "platform_admin"` (e.g. company `"member"`, `"admin"`, `"bid_manager"`) → HTTP 403. Valid `platform_admin` JWT → request proceeds. (Covers E11-P0-007, E11-P0-006, E11-R-003)

### Migration 003 — `framework_suggestions` table

2. **AC2** — Alembic migration `003_framework_suggestions.py` in `services/admin-api` runs cleanly (`upgrade head` from revision `002`) and `downgrade` to `002` restores prior state. After upgrade, `admin.framework_suggestions` table exists with columns: `id (UUID PK DEFAULT gen_random_uuid())`, `opportunity_id (UUID NOT NULL)`, `framework_id (UUID NOT NULL FK→admin.compliance_frameworks(id) ON DELETE CASCADE)`, `confidence (FLOAT NOT NULL)`, `status (VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK IN ('pending','accepted','rejected'))`, `reviewed_by (VARCHAR(255), nullable)`, `reviewed_at (TIMESTAMPTZ, nullable)`, `created_at (TIMESTAMPTZ DEFAULT now())`. Two non-unique indexes: `ix_framework_suggestions_opportunity_id` on `(opportunity_id)` and `ix_framework_suggestions_status` on `(status)`.

### Framework Assignment CRUD

3. **AC3** — `POST /api/v1/admin/opportunities/{opportunity_id}/compliance-frameworks` assigns one or more frameworks to an opportunity. Request body: `{ "framework_ids": ["<uuid>", ...] }` (list, min 1 item). For each `framework_id`: verify it exists in `admin.compliance_frameworks` → HTTP 404 `{"detail": "Compliance framework not found"}` if absent. If the `(opportunity_id, framework_id)` pair already exists in `admin.opportunity_compliance_frameworks`, skip silently (idempotent — no 409 on duplicate). `assigned_by` is set from `str(admin_user.admin_id)`. Returns HTTP 201 with a list of `OpportunityFrameworkAssignmentResponse` (one per framework_id processed).

4. **AC4** — `GET /api/v1/admin/opportunities/{opportunity_id}/compliance-frameworks` returns HTTP 200 with all frameworks currently assigned to the opportunity as a JSON array of `ComplianceFrameworkWithAssignmentResponse` (full framework fields + `assigned_by: str | None` and `assigned_at: datetime`). Returns empty list `[]` if no assignments exist.

5. **AC5** — `DELETE /api/v1/admin/opportunities/{opportunity_id}/compliance-frameworks/{framework_id}` removes a specific assignment. HTTP 404 `{"detail": "Assignment not found"}` if no matching `(opportunity_id, framework_id)` row exists. HTTP 204 No Content on success. This endpoint removes only the *assignment* — the framework itself is not deleted.

### Auto-Suggestion Flow

6. **AC6** — `POST /api/v1/admin/compliance-frameworks/suggest` accepts request body `SuggestFrameworksRequest`: `{ "opportunity_id": "<uuid>", "opportunity_metadata": { "country": "<str|null>", "source": "<str|null>", "funding_type": "<str|null>", "programme": "<str|null>" } }`. Sends the payload to the Framework Suggestion Agent via the admin-api AI Gateway client at logical agent name `"framework-suggestion"` (URL: `{ADMIN_API_AIGW_BASE_URL}/agents/framework-suggestion/run`). Request header: `X-Caller-Service: admin-api`. Timeout: 30 seconds (configurable via `ADMIN_API_AIGW_TIMEOUT_SECONDS`). On agent timeout or HTTP 5xx from gateway → HTTP 503 with standard body `{"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"}`. (Covers E11-P0-008, E11-R-001)

7. **AC7** — The agent payload must be structured as:
   ```json
   {
     "opportunity_id": "<UUID as string>",
     "metadata": {
       "country": "<value or null>",
       "source": "<value or null>",
       "funding_type": "<value or null>",
       "programme": "<value or null>"
     }
   }
   ```
   The agent response is parsed: extract `suggestions` (list, default `[]` if key absent). Each item has `framework_id (UUID string)` and `confidence (float, clamped to 0.0–1.0)`. Only suggestions whose `framework_id` exists in `admin.compliance_frameworks` are stored — **invalid framework_ids from the agent response are silently skipped** (do not raise 404). Valid suggestions are stored in `admin.framework_suggestions` with `status="pending"`. Returns HTTP 200 with list of stored `FrameworkSuggestionResponse`.

8. **AC8** — `GET /api/v1/admin/framework-suggestions` returns a paginated list. Query params: `status` (`"pending"` | `"accepted"` | `"rejected"`, optional — filter by status), `opportunity_id (UUID, optional — filter by opportunity)`, `page (int, default 1, ge=1)`, `page_size (int, default 20, ge=1, le=100)`. Response: `FrameworkSuggestionListResponse` with `suggestions`, `total`, `page`, `page_size`. Results ordered by `created_at` descending.

9. **AC9** — `PATCH /api/v1/admin/framework-suggestions/{suggestion_id}` accepts `{ "action": "accept" | "reject", "override_framework_id": "<uuid or null>" }`. HTTP 404 if suggestion not found. HTTP 409 `{"detail": "Suggestion has already been processed", "code": "SUGGESTION_ALREADY_PROCESSED"}` if `status != "pending"` (covers both accept and reject idempotency protection). On **`reject`**: update `status="rejected"`, `reviewed_by=str(admin_user.admin_id)`, `reviewed_at=now(UTC)`. No assignment created. On **`accept`**: `effective_fw_id = override_framework_id if provided else suggestion.framework_id`. If `override_framework_id` provided: verify it exists in `admin.compliance_frameworks` → HTTP 404 `{"detail": "Override framework not found"}` if absent. Within a **single transaction**: create `OpportunityComplianceFramework(opportunity_id=suggestion.opportunity_id, framework_id=effective_fw_id, assigned_by=str(admin_user.admin_id))` (skip if already exists — idempotent), then update suggestion `status="accepted"`, `reviewed_by`, `reviewed_at`. Returns HTTP 200 with updated `FrameworkSuggestionResponse` including `effective_framework_id` populated. (Covers E11-P1-014, E11-R-007)

## Tasks / Subtasks

### Task 1: Migration 003 — `framework_suggestions` table (AC: 2)

- [x] **Task 1: Create Alembic migration 003** (AC: 2)
  - [x] 1.1 Create `services/admin-api/alembic/versions/003_framework_suggestions.py`:
    - `revision = "003"`, `down_revision = "002"`
    - `upgrade()`: Create `admin.framework_suggestions` table using `op.create_table(...)` in `admin` schema:
      - `sa.Column("id", sa.UUID(as_uuid=True), primary_key=True, server_default=sa.text("gen_random_uuid()"))`
      - `sa.Column("opportunity_id", sa.UUID(as_uuid=True), nullable=False)`
      - `sa.Column("framework_id", sa.UUID(as_uuid=True), nullable=False)` + FK: `sa.ForeignKeyConstraint(["framework_id"], ["admin.compliance_frameworks.id"], ondelete="CASCADE")`
      - `sa.Column("confidence", sa.Float, nullable=False)`
      - `sa.Column("status", sa.String(20), nullable=False, server_default="pending")` + check constraint: `sa.CheckConstraint("status IN ('pending', 'accepted', 'rejected')", name="ck_framework_suggestions_status")`
      - `sa.Column("reviewed_by", sa.String(255), nullable=True)`
      - `sa.Column("reviewed_at", sa.TIMESTAMP(timezone=True), nullable=True)`
      - `sa.Column("created_at", sa.TIMESTAMP(timezone=True), nullable=False, server_default=sa.text("now()"))`
      - `schema="admin"`
    - After `create_table`: add `op.create_index("ix_framework_suggestions_opportunity_id", "framework_suggestions", ["opportunity_id"], schema="admin")`
    - After: add `op.create_index("ix_framework_suggestions_status", "framework_suggestions", ["status"], schema="admin")`
    - `downgrade()`: `op.drop_index(...)` both indexes, then `op.drop_table("framework_suggestions", schema="admin")`

### Task 2: `FrameworkSuggestion` ORM Model (AC: 2)

- [x] **Task 2: Create ORM model** (AC: 2)
  - [x] 2.1 Create `services/admin-api/src/admin_api/models/framework_suggestion.py`:
    ```python
    from __future__ import annotations
    import uuid as _uuid
    from datetime import datetime
    import sqlalchemy as sa
    from sqlalchemy.orm import Mapped, mapped_column
    from admin_api.models.base import Base

    class FrameworkSuggestion(Base):
        __tablename__ = "framework_suggestions"
        __table_args__ = (
            sa.CheckConstraint(
                "status IN ('pending', 'accepted', 'rejected')",
                name="ck_framework_suggestions_status",
            ),
            {"schema": "admin"},
        )
        id: Mapped[_uuid.UUID] = mapped_column(
            sa.UUID(as_uuid=True), primary_key=True, default=_uuid.uuid4
        )
        opportunity_id: Mapped[_uuid.UUID] = mapped_column(
            sa.UUID(as_uuid=True), nullable=False, index=True
        )
        framework_id: Mapped[_uuid.UUID] = mapped_column(
            sa.UUID(as_uuid=True),
            sa.ForeignKey("admin.compliance_frameworks.id", ondelete="CASCADE"),
            nullable=False,
        )
        confidence: Mapped[float] = mapped_column(sa.Float, nullable=False)
        status: Mapped[str] = mapped_column(
            sa.String(20), nullable=False, server_default="pending", index=True
        )
        reviewed_by: Mapped[str | None] = mapped_column(sa.String(255), nullable=True)
        reviewed_at: Mapped[datetime | None] = mapped_column(
            sa.TIMESTAMP(timezone=True), nullable=True
        )
        created_at: Mapped[datetime] = mapped_column(
            sa.TIMESTAMP(timezone=True),
            nullable=False,
            server_default=sa.text("now()"),
        )
    ```
  - [x] 2.2 Update `services/admin-api/src/admin_api/models/__init__.py` to import and export `FrameworkSuggestion`

### Task 3: Schemas (AC: 3–9)

- [x] **Task 3: Create request/response schemas** (AC: 3–9)
  - [x] 3.1 Create `services/admin-api/src/admin_api/schemas/framework_assignment.py`:
    - `from __future__ import annotations` header + imports: `UUID`, `datetime`, `BaseModel`, `ConfigDict`, `Field`
    - `class AssignFrameworksRequest(BaseModel)`:
      - `framework_ids: list[UUID] = Field(..., min_length=1)` — minimum 1 UUID required
      - `model_config = ConfigDict(extra="ignore")`
    - `class OpportunityFrameworkAssignmentResponse(BaseModel)`:
      - `opportunity_id: UUID`, `framework_id: UUID`, `assigned_by: str | None`, `assigned_at: datetime`
      - `model_config = ConfigDict(from_attributes=True)`
    - `class ComplianceFrameworkWithAssignmentResponse(BaseModel)`:
      - All fields from `ComplianceFrameworkResponse` (import it): `id`, `name`, `description`, `country`, `regulation_type`, `rules`, `is_active`, `created_by`, `created_at`, `updated_at`
      - Plus: `assigned_by: str | None`, `assigned_at: datetime`
      - `model_config = ConfigDict(from_attributes=True)`
  - [x] 3.2 Create `services/admin-api/src/admin_api/schemas/framework_suggestion.py`:
    - `class OpportunityMetadata(BaseModel)`:
      - `country: str | None = None`, `source: str | None = None`, `funding_type: str | None = None`, `programme: str | None = None`
      - `model_config = ConfigDict(extra="ignore")`
    - `class SuggestFrameworksRequest(BaseModel)`:
      - `opportunity_id: UUID`, `opportunity_metadata: OpportunityMetadata`
      - `model_config = ConfigDict(extra="ignore")`
    - `class FrameworkSuggestionResponse(BaseModel)`:
      - `id: UUID`, `opportunity_id: UUID`, `framework_id: UUID`, `confidence: float`, `status: str`
      - `reviewed_by: str | None`, `reviewed_at: datetime | None`, `created_at: datetime`
      - `effective_framework_id: UUID | None = None` — populated on accept responses only; `None` on list/pending responses
      - `model_config = ConfigDict(from_attributes=True)`
    - `class FrameworkSuggestionListResponse(BaseModel)`:
      - `suggestions: list[FrameworkSuggestionResponse]`, `total: int`, `page: int`, `page_size: int`
    - `class PatchSuggestionRequest(BaseModel)`:
      - `action: Literal["accept", "reject"]`
      - `override_framework_id: UUID | None = None`
      - `model_config = ConfigDict(extra="ignore")`

### Task 4: AI Gateway Client for admin-api (AC: 6)

- [x] **Task 4: Create AI Gateway client** (AC: 6–7)
  - [x] 4.1 Create `services/admin-api/src/admin_api/core/ai_gateway.py` (follow exact pattern from `services/client-api/src/client_api/core/ai_gateway.py`):
    - `class AiGatewayTimeoutError(Exception): pass`
    - `class AiGatewayUnavailableError(Exception): pass`
    - `class AiGatewayClient`:
      - `def __init__(self, base_url: str, timeout: float = 30.0)` — store `self._base_url = base_url.rstrip("/")`, create `self._client = httpx.AsyncClient(timeout=httpx.Timeout(timeout))`
      - `async def run_agent(self, agent_name: str, payload: dict) -> dict`: POST to `{self._base_url}/agents/{agent_name}/run` with JSON payload + header `{"X-Caller-Service": "admin-api"}`; on `httpx.TimeoutException` → raise `AiGatewayTimeoutError`; on `response.status_code >= 500` → raise `AiGatewayUnavailableError`; return `response.json()`
      - `async def aclose(self) -> None`: `await self._client.aclose()`
    - `_gw_client: AiGatewayClient | None = None` module-level singleton
    - `async def get_ai_gateway_client() -> AsyncGenerator[AiGatewayClient, None]`:
      - Read `settings = get_settings()`
      - Use module-level singleton (lazy-init); on first call `_gw_client = AiGatewayClient(base_url=settings.admin_api_aigw_base_url, timeout=settings.admin_api_aigw_timeout_seconds)`
      - Yield client
  - [x] 4.2 Update `services/admin-api/src/admin_api/config.py` — add fields to `Settings`:
    - `admin_api_aigw_base_url: str = Field(..., alias="ADMIN_API_AIGW_BASE_URL")`
    - `admin_api_aigw_timeout_seconds: float = Field(default=30.0, alias="ADMIN_API_AIGW_TIMEOUT_SECONDS")`
  - [x] 4.3 Add `httpx>=0.27` to `[project] dependencies` in `services/admin-api/pyproject.toml` (if not already present from S11.08 dev dependencies — move to main deps since it's now a runtime dependency)

### Task 5: `framework_assignment_service.py` (AC: 3–5)

- [x] **Task 5: Create framework assignment service** (AC: 3–5)
  - [x] 5.1 Create `services/admin-api/src/admin_api/services/framework_assignment_service.py`:
    - Imports: `uuid`, `structlog`, `AsyncSession`, `select`, `HTTPException`, `ComplianceFramework`, `OpportunityComplianceFramework`, `AdminUser`
    - `log = structlog.get_logger()`
    - `async def assign_frameworks(opportunity_id: UUID, framework_ids: list[UUID], admin_user: AdminUser, session: AsyncSession) -> list[OpportunityComplianceFramework]`:
      - For each `framework_id` in `framework_ids`:
        - Verify framework: `await session.scalar(select(ComplianceFramework).where(ComplianceFramework.id == framework_id))` → `HTTPException(404, "Compliance framework not found")` if `None`
        - Check existing: `existing = await session.scalar(select(OpportunityComplianceFramework).where(OpportunityComplianceFramework.opportunity_id == opportunity_id, OpportunityComplianceFramework.framework_id == framework_id))`
        - If `existing is None`: create and `session.add(OpportunityComplianceFramework(opportunity_id=opportunity_id, framework_id=framework_id, assigned_by=str(admin_user.admin_id)))` — collect in results list
        - If `existing` already there: collect `existing` in results list (idempotent)
      - `await session.flush()`
      - Log `framework_assignment.assigned` with `opportunity_id`, `framework_count=len(framework_ids)`, `admin_id=str(admin_user.admin_id)`
      - Return collected assignment list
    - `async def list_assigned_frameworks(opportunity_id: UUID, session: AsyncSession) -> list[tuple[ComplianceFramework, OpportunityComplianceFramework]]`:
      - Query: `select(ComplianceFramework, OpportunityComplianceFramework).join(OpportunityComplianceFramework, ComplianceFramework.id == OpportunityComplianceFramework.framework_id).where(OpportunityComplianceFramework.opportunity_id == opportunity_id).order_by(OpportunityComplianceFramework.assigned_at.asc())`
      - Return `list(result.all())`
    - `async def remove_assignment(opportunity_id: UUID, framework_id: UUID, admin_user: AdminUser, session: AsyncSession) -> None`:
      - Fetch: `row = await session.scalar(select(OpportunityComplianceFramework).where(...))` → `HTTPException(404, "Assignment not found")` if `None`
      - `await session.delete(row)` → `await session.flush()`
      - Log `framework_assignment.removed` with `opportunity_id`, `framework_id`, `admin_id`

### Task 6: `framework_suggestion_service.py` (AC: 6–9)

- [x] **Task 6: Create framework suggestion service** (AC: 6–9)
  - [x] 6.1 Create `services/admin-api/src/admin_api/services/framework_suggestion_service.py`:
    - Imports: `uuid`, `datetime`, `timezone`, `structlog`, `AsyncSession`, `select`, `func`, `HTTPException`, `ComplianceFramework`, `OpportunityComplianceFramework`, `FrameworkSuggestion`, `AdminUser`, `AiGatewayClient`, `AiGatewayTimeoutError`, `AiGatewayUnavailableError`, `SuggestFrameworksRequest`, `PatchSuggestionRequest`
    - `log = structlog.get_logger()`
    - `_AGENT_ERROR_DETAIL = {"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"}`
    - `async def suggest_frameworks(request: SuggestFrameworksRequest, admin_user: AdminUser, session: AsyncSession, gw_client: AiGatewayClient) -> list[FrameworkSuggestion]`:
      - Build payload (AC7):
        ```python
        payload = {
            "opportunity_id": str(request.opportunity_id),
            "metadata": {
                "country": request.opportunity_metadata.country,
                "source": request.opportunity_metadata.source,
                "funding_type": request.opportunity_metadata.funding_type,
                "programme": request.opportunity_metadata.programme,
            },
        }
        ```
      - Call agent in try/except:
        ```python
        try:
            agent_response = await gw_client.run_agent("framework-suggestion", payload)
        except (AiGatewayTimeoutError, AiGatewayUnavailableError):
            log.warning("framework_suggestion.agent_unavailable", opportunity_id=str(request.opportunity_id))
            raise HTTPException(status_code=503, detail=_AGENT_ERROR_DETAIL)
        ```
      - Parse `raw_suggestions = agent_response.get("suggestions", [])` (default `[]`)
      - For each raw suggestion (wrap in try/except `(KeyError, ValueError, TypeError)` → log warning, skip):
        - Extract `fw_id_str = item.get("framework_id")` and parse `UUID(fw_id_str)`
        - Extract `confidence = max(0.0, min(1.0, float(item.get("confidence", 0.0))))` 
        - Verify framework exists: `await session.scalar(select(ComplianceFramework).where(ComplianceFramework.id == fw_uuid))` → if `None`: `log.warning("framework_suggestion.unknown_framework", ...)` and `continue` (skip)
        - Create `FrameworkSuggestion(opportunity_id=request.opportunity_id, framework_id=fw_uuid, confidence=confidence, status="pending")`; `session.add(suggestion)`; collect in list
      - `await session.flush()`
      - Log `framework_suggestion.created` with `opportunity_id`, `suggestion_count`
      - Return list of created `FrameworkSuggestion` objects
    - `async def list_suggestions(status: str | None, opportunity_id: UUID | None, page: int, page_size: int, session: AsyncSession) -> tuple[list[FrameworkSuggestion], int]`:
      - Build base query + count query; apply filters; paginate with `order_by(FrameworkSuggestion.created_at.desc())`
      - Return `(list(scalars), total_count)`
    - `async def process_suggestion(suggestion_id: UUID, request: PatchSuggestionRequest, admin_user: AdminUser, session: AsyncSession) -> tuple[FrameworkSuggestion, UUID | None]`:
      - Fetch suggestion: → `HTTPException(404, "Framework suggestion not found")` if `None`
      - Guard: `if suggestion.status != "pending": raise HTTPException(409, {"detail": "Suggestion has already been processed", "code": "SUGGESTION_ALREADY_PROCESSED"})`
      - `now_utc = datetime.now(timezone.utc)`
      - If `action == "reject"`:
        - `suggestion.status = "rejected"`, `suggestion.reviewed_by = str(admin_user.admin_id)`, `suggestion.reviewed_at = now_utc`
        - `await session.flush()` → return `(suggestion, None)`
      - If `action == "accept"`:
        - `effective_fw_id = request.override_framework_id if request.override_framework_id else suggestion.framework_id`
        - If `request.override_framework_id`:
          - Verify: `fw = await session.scalar(select(ComplianceFramework).where(ComplianceFramework.id == effective_fw_id))` → `HTTPException(404, "Override framework not found")` if `None`
        - Check existing assignment: `existing = await session.scalar(select(OpportunityComplianceFramework).where(...))` — if `None`: create and `session.add(OpportunityComplianceFramework(opportunity_id=suggestion.opportunity_id, framework_id=effective_fw_id, assigned_by=str(admin_user.admin_id)))`
        - Update suggestion: `suggestion.status = "accepted"`, `suggestion.reviewed_by = str(admin_user.admin_id)`, `suggestion.reviewed_at = now_utc`
        - `await session.flush()` (atomicity — both writes in the same session/transaction)
        - Log `framework_suggestion.accepted` with `suggestion_id`, `effective_fw_id`, `override_used=bool(request.override_framework_id)`
        - Return `(suggestion, effective_fw_id)`

### Task 7: Routers (AC: 3–9)

- [x] **Task 7: Create routers** (AC: 3–9)
  - [x] 7.1 Create `services/admin-api/src/admin_api/api/v1/framework_assignments.py`:
    - `from __future__ import annotations` header
    - `router = APIRouter(prefix="/admin/opportunities", tags=["framework-assignments"])`
    - All endpoints: `admin_user: Annotated[AdminUser, Depends(get_admin_user)]`, `session: Annotated[AsyncSession, Depends(get_db_session)]`
    - `POST /{opportunity_id}/compliance-frameworks`:
      - `body: AssignFrameworksRequest` → `framework_assignment_service.assign_frameworks(opportunity_id, body.framework_ids, admin_user, session)` → `JSONResponse(status_code=201, content=[...])`
      - Build response: `[OpportunityFrameworkAssignmentResponse.model_validate(a).model_dump(mode="json") for a in assignments]` — serialize `UUID` and `datetime` properly
    - `GET /{opportunity_id}/compliance-frameworks`:
      - → `framework_assignment_service.list_assigned_frameworks(opportunity_id, session)` → build `ComplianceFrameworkWithAssignmentResponse` for each `(fw, asgn)` tuple
      - Response shape: `[{"id": ..., "name": ..., ..., "assigned_by": ..., "assigned_at": ...}, ...]`
    - `DELETE /{opportunity_id}/compliance-frameworks/{framework_id}`:
      - → `framework_assignment_service.remove_assignment(opportunity_id, framework_id, admin_user, session)` → `Response(status_code=204)`
  - [x] 7.2 Create `services/admin-api/src/admin_api/api/v1/framework_suggestions.py`:
    - `from __future__ import annotations` header
    - `router = APIRouter(prefix="/admin", tags=["framework-suggestions"])`
    - All endpoints: `admin_user: Annotated[AdminUser, Depends(get_admin_user)]`, `session: Annotated[AsyncSession, Depends(get_db_session)]`
    - `POST /compliance-frameworks/suggest`:
      - Additional dep: `gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)]`
      - `body: SuggestFrameworksRequest` → `framework_suggestion_service.suggest_frameworks(body, admin_user, session, gw_client)` → JSON 200
      - Catch 503 `HTTPException` with dict detail → return `JSONResponse(status_code=503, content=exc.detail)`
    - `GET /framework-suggestions`:
      - Query params: `status: str | None = Query(None)`, `opportunity_id: UUID | None = Query(None)`, `page: int = Query(1, ge=1)`, `page_size: int = Query(20, ge=1, le=100)`
      - → `framework_suggestion_service.list_suggestions(status, opportunity_id, page, page_size, session)` → `FrameworkSuggestionListResponse`
    - `PATCH /framework-suggestions/{suggestion_id}`:
      - `body: PatchSuggestionRequest` → `framework_suggestion_service.process_suggestion(suggestion_id, body, admin_user, session)` → `(suggestion, effective_fw_id)`
      - Build response: `FrameworkSuggestionResponse.model_validate(suggestion)` with `effective_framework_id = effective_fw_id`
      - Catch 409 HTTPException with dict detail → `JSONResponse(status_code=409, content=exc.detail)`

### Task 8: Update `main.py` (AC: all endpoints)

- [x] **Task 8: Register new routers** (AC: all)
  - [x] 8.1 Update `services/admin-api/src/admin_api/main.py`:
    - Add: `from admin_api.api.v1 import framework_assignments as fa_v1`
    - Add: `from admin_api.api.v1 import framework_suggestions as fs_v1`
    - Add: `api_v1_router.include_router(fa_v1.router)` after existing router inclusions
    - Add: `api_v1_router.include_router(fs_v1.router)` after `fa_v1`
    - **ROUTING ORDER CAUTION**: The `fs_v1` router registers `POST /admin/compliance-frameworks/suggest`. The existing S11.08 `compliance_frameworks` router registers `POST /admin/compliance-frameworks/` (creates a framework). These are different paths (one has a trailing literal segment) so no conflict. However, register `fs_v1` **before** the S11.08 `cf_v1` router to ensure FastAPI matches the literal `/suggest` path before any parameterized path variants.

### Task 9: Write tests (AC: 1–9)

- [x] **Task 9: Framework assignment tests** (AC: 1, 3–5)
  - [x] 9.1 Create `services/admin-api/tests/api/test_framework_assignments.py`:
    - Module constants:
      ```python
      BASE_ASSIGN_URL = "/api/v1/admin/opportunities"

      FRAMEWORK_CREATE_NATIONAL = {
          "name": "ZOP S11.9 Test National",
          "country": "BG",
          "regulation_type": "national",
          "rules": [],
      }
      FRAMEWORK_CREATE_EU = {
          "name": "Horizon Europe S11.9 Test",
          "country": "EU",
          "regulation_type": "eu",
          "rules": [],
      }
      ```
    - Session-scoped `admin_jwt_env_setup_assignment` fixture (autouse) — same pattern as `admin_jwt_env_setup` in `test_compliance_frameworks.py`
    - Function-scoped `assign_client_and_session` fixture using `admin_api_session_factory` from conftest
    - Helper `_create_framework(client, admin_token, payload) -> str` — POSTs to `/api/v1/admin/compliance-frameworks` and returns the `id`
    - **`TestAC1AdminAuth`** — Admin-only (E11-P0-006, E11-P0-007, E11-R-003):
      | # | Method | Expected | AC |
      |---|--------|----------|----|
      | 1 | `test_unauthenticated_post_returns_401` | 401 | AC1 |
      | 2 | `test_company_jwt_post_returns_403` | 403 | AC1 |
      | 3 | `test_unauthenticated_get_returns_401` | 401 | AC1 |
      | 4 | `test_company_jwt_get_returns_403` | 403 | AC1 |
      | 5 | `test_unauthenticated_delete_returns_401` | 401 | AC1 |
      | 6 | `test_company_jwt_delete_returns_403` | 403 | AC1 |
    - **`TestAC3AssignFrameworks`** (E11-P1-011, E11-P1-012, E11-R-010):
      | # | Method | Expected | AC |
      |---|--------|----------|----|
      | 7 | `test_assign_single_framework_returns_201_with_assignment` | 201 | AC3 |
      | 8 | `test_assign_multiple_frameworks_returns_201_all_assigned` | 201 | AC3 |
      | 9 | `test_assign_hybrid_national_and_eu_frameworks_to_same_opportunity` | 201 | AC3, E11-P1-012 |
      | 10 | `test_assign_nonexistent_framework_id_returns_404` | 404 | AC3 |
      | 11 | `test_assign_same_framework_twice_is_idempotent_no_409` | 201 | AC3 |
      | 12 | `test_assign_empty_framework_ids_list_returns_422` | 422 | AC3 |
    - **`TestAC4ListAssigned`** (E11-P1-011, E11-P1-012, E11-R-010):
      | # | Method | Expected | AC |
      |---|--------|----------|----|
      | 13 | `test_list_returns_assigned_frameworks_with_assignment_metadata` | 200 | AC4 |
      | 14 | `test_list_empty_returns_empty_list` | 200 | AC4 |
      | 15 | `test_list_returns_both_frameworks_after_hybrid_assignment` | 200 | AC4, E11-P1-012 |
      | 16 | `test_list_only_shows_frameworks_for_requested_opportunity` | 200 | AC4 |
    - **`TestAC5RemoveAssignment`** (E11-P1-011, E11-R-010):
      | # | Method | Expected | AC |
      |---|--------|----------|----|
      | 17 | `test_delete_assignment_returns_204` | 204 | AC5 |
      | 18 | `test_delete_nonexistent_assignment_returns_404` | 404 | AC5 |
      | 19 | `test_delete_one_of_two_frameworks_leaves_other_intact` | 204+200 | AC5, E11-R-010 |
    - **Total: 19 tests** across 4 classes

- [x] **Task 9.2: Framework suggestion tests** (AC: 1, 6–9)
  - [x] 9.2 Create `services/admin-api/tests/api/test_framework_suggestions.py`:
    - `respx = pytest.importorskip("respx", reason="respx required for AI Gateway mocking")` guard
    - Module constants:
      ```python
      AIGW_TEST_BASE_URL = "http://test-aigw-admin.local:8001"
      AIGW_SUGGEST_URL = f"{AIGW_TEST_BASE_URL}/agents/framework-suggestion/run"
      BASE_SUGGEST_URL = "/api/v1/admin/compliance-frameworks/suggest"
      BASE_SUGGESTIONS_URL = "/api/v1/admin/framework-suggestions"
      AGENT_UNAVAILABLE_CODE = "AGENT_UNAVAILABLE"

      MOCK_SUGGEST_OPP_ID = "00000000-0000-0000-0000-000000000001"
      MOCK_SUGGESTION_REQUEST = {
          "opportunity_id": MOCK_SUGGEST_OPP_ID,
          "opportunity_metadata": {
              "country": "BG",
              "source": "TED",
              "funding_type": "grant",
              "programme": "horizon_europe",
          },
      }
      ```
    - Session-scoped `aigw_admin_env_setup` fixture (autouse):
      - Set `ADMIN_API_AIGW_BASE_URL = AIGW_TEST_BASE_URL`
      - Clear `get_settings.cache_clear()` and reset `admin_api.core.ai_gateway._gw_client = None`
      - Yield; restore on teardown
    - Function-scoped `suggest_client_and_session` fixture (same pattern as `cf_client_and_session`)
    - Helper `_create_framework_in_db(session, name="Test FW") -> UUID` — directly inserts via ORM
    - **`TestAC1Authorization`** (E11-P0-007, E11-R-003):
      | # | Test | Status | AC |
      |---|------|--------|----|
      | 1 | `test_unauthenticated_post_suggest_returns_401` | 401 | AC1 |
      | 2 | `test_company_jwt_post_suggest_returns_403` | 403 | AC1 |
      | 3 | `test_unauthenticated_get_suggestions_returns_401` | 401 | AC1 |
      | 4 | `test_company_jwt_get_suggestions_returns_403` | 403 | AC1, E11-P0-007 |
      | 5 | `test_unauthenticated_patch_suggestion_returns_401` | 401 | AC1 |
      | 6 | `test_company_jwt_patch_suggestion_returns_403` | 403 | AC1, E11-P0-007 |
    - **`TestAC6AC7SuggestEndpoint`** (E11-P1-013, E11-P0-008, E11-R-001):
      | # | Test | Status | AC |
      |---|------|--------|----|
      | 7 | `test_suggest_sends_correct_payload_to_agent` | 200 | AC6, AC7 |
      | 8 | `test_suggest_stores_suggestions_with_pending_status` | 200 | AC7, E11-P1-013 |
      | 9 | `test_suggest_x_caller_service_header_is_admin_api` | 200 | AC6 |
      | 10 | `test_suggest_agent_timeout_returns_503_agent_unavailable` | 503 | AC6, E11-P0-008 |
      | 11 | `test_suggest_agent_500_returns_503_not_500` | 503 | E11-P0-008 |
      | 12 | `test_suggest_invalid_framework_id_from_agent_is_skipped` | 200 | AC7 |
      | 13 | `test_suggest_unknown_framework_id_from_agent_is_silently_skipped` | 200 | AC7 |
    - **`TestAC8ListSuggestions`** (E11-P1-013):
      | # | Test | Status |
      |---|------|--------|
      | 14 | `test_list_suggestions_returns_paginated_response` | 200 |
      | 15 | `test_list_filter_by_status_pending` | 200 |
      | 16 | `test_list_filter_by_opportunity_id` | 200 |
      | 17 | `test_list_empty_returns_zero_total` | 200 |
    - **`TestAC9AcceptSuggestion`** (E11-P1-014, E11-R-007, E11-P2-007):
      | # | Test | Status | AC |
      |---|------|--------|----|
      | 18 | `test_accept_suggestion_returns_200_with_accepted_status` | 200 | AC9 |
      | 19 | `test_accept_suggestion_creates_assignment_in_db` | 200 | AC9, E11-P1-014 |
      | 20 | `test_accept_creates_assignment_atomically_status_and_row_together` | 200 | E11-R-007 |
      | 21 | `test_accept_with_override_framework_assigns_override` | 200 | AC9, E11-P2-007 |
      | 22 | `test_accept_with_invalid_override_framework_id_returns_404` | 404 | AC9 |
      | 23 | `test_accept_already_accepted_returns_409` | 409 | AC9 |
      | 24 | `test_accept_already_rejected_returns_409` | 409 | AC9 |
    - **`TestAC9RejectSuggestion`** (E11-P1-015):
      | # | Test | Status | AC |
      |---|------|--------|----|
      | 25 | `test_reject_suggestion_returns_200_with_rejected_status` | 200 | AC9 |
      | 26 | `test_reject_does_not_create_assignment` | 200 | AC9, E11-P1-015 |
      | 27 | `test_reject_already_rejected_returns_409` | 409 | AC9 |
      | 28 | `test_reject_unknown_suggestion_returns_404` | 404 | AC9 |
    - **Total: 28 tests** across 5 classes

### Review Findings

Code review conducted 2026-04-10 using parallel adversarial layers (Blind Hunter, Edge Case Hunter, Acceptance Auditor). 47 tests pass, 0 regressions. All acceptance criteria met on the happy path. Findings below are edge-case hardening and spec conformance items.

**Patch** (6 items — fix before shipping):

- [x] [Review][Patch] **P1 — NaN/Infinity confidence from AI agent corrupts DB and crashes JSON serialization** [`services/framework_suggestion_service.py:77`] — `max(0.0, min(1.0, float('nan')))` returns `nan` on CPython (IEEE 754). NaN is stored in PostgreSQL FLOAT column; subsequent `model_dump(mode="json")` raises `ValueError: Out of range float values are not JSON compliant`, permanently breaking reads of that row. Fix: add `import math` and guard with `if not math.isfinite(raw_conf): raw_conf = 0.0` before clamping. (Sources: edge+blind)
- [x] [Review][Patch] **P2 — Non-list `suggestions` field from agent causes unhandled TypeError** [`services/framework_suggestion_service.py:70`] — If the agent returns `{"suggestions": null}`, the line `for item in raw_suggestions:` raises `TypeError: 'NoneType' is not iterable`. The `try/except` block is inside the loop body and does not catch this. A string value iterates char-by-char, emitting one warning per character. Fix: add `if not isinstance(raw_suggestions, list): raw_suggestions = []` after the `.get()` call. (Sources: edge)
- [x] [Review][Patch] **P3 — Duplicate UUIDs in `framework_ids` list causes IntegrityError/500** [`services/framework_assignment_service.py:41`] — `AssignFrameworksRequest` validates `min_length=1` but not uniqueness. If `[uuid_A, uuid_A]` is sent, the first iteration `session.add()`s a new row; the second iteration queries the DB (pre-flush) and finds nothing, so it adds a duplicate. `await session.flush()` hits the composite PK constraint on `(opportunity_id, framework_id)` → unhandled `IntegrityError` → 500. Fix: deduplicate at the top of `assign_frameworks()`: `framework_ids = list(dict.fromkeys(framework_ids))`. (Sources: edge)
- [x] [Review][Patch] **P4 — `status` query parameter on GET /framework-suggestions accepts arbitrary strings** [`api/v1/framework_suggestions.py:62`] — AC8 specifies `status ("pending" | "accepted" | "rejected")` but the endpoint uses `status: str | None = Query(None)`. Invalid values (e.g., `?status=typo`) silently return empty results with 200 OK instead of 422, violating the documented contract. Fix: change to `status: Literal["pending", "accepted", "rejected"] | None = Query(None)`. (Sources: edge+blind+auditor)
- [x] [Review][Patch] **P5 — Unused imports** [`services/framework_assignment_service.py:8`, `schemas/framework_assignment.py:9`] — `import uuid` (line 8) is never used in the assignment service. `ComplianceFrameworkResponse` (line 9) is imported but never referenced in the schema file. Fix: remove both. (Sources: blind)
- [x] [Review][Patch] **P6 — 503 error response tests do not assert `message` field** [`tests/api/test_framework_suggestions.py:356,376`] — AC6 mandates the 503 body include both `message` and `code` keys. The implementation correctly sets both in `_AGENT_ERROR_DETAIL`, but tests only assert `body.get("code")`, leaving the `message` value unverified. Fix: add `assert body.get("message") == "AI features are temporarily unavailable. Please try again."` to both timeout and 5xx tests. (Sources: auditor)

**Deferred** (5 items — real but out of scope or pre-existing):

- [x] [Review][Defer] **D1 — `ADMIN_API_AIGW_BASE_URL` defaults to empty string instead of required** [`config.py:31`] — Spec Task 4.2 says `Field(...)` (required). Implementation uses `default=""` to avoid breaking S11.08 `test_compliance_frameworks.py` tests that don't set this env var. An empty URL causes `httpx.InvalidURL` at runtime on first AI Gateway call (not at startup). This is a documented trade-off (Dev Notes). Fixing requires updating all existing admin-api test fixtures to set this env var. — deferred, cross-story impact
- [x] [Review][Defer] **D2 — Race condition on concurrent suggestion accepts** [`services/framework_suggestion_service.py:174`] — Two concurrent PATCH /accept requests can both read status="pending" (READ COMMITTED isolation). No `SELECT ... FOR UPDATE` serializes access. The OCF composite PK prevents duplicate assignments, but an unhandled `IntegrityError` surfaces as 500. Low risk: admin operations are single-user. Fix (if needed later): add `.with_for_update()` to the suggestion select. — deferred, concurrency hardening
- [x] [Review][Defer] **D3 — Gateway 4xx responses silently treated as empty success** [`core/ai_gateway.py:62`] — Only `status_code >= 500` is caught. If the gateway returns 401, 403, 429, or 422, the JSON body is treated as a valid agent response. `agent_response.get("suggestions", [])` returns `[]`, and the caller sees 200 OK with zero suggestions. Spec only mandates handling timeout and 5xx. — deferred, spec gap
- [x] [Review][Defer] **D4 — Uncaught `httpx.TransportError` subtypes escape as 500** [`core/ai_gateway.py:49-60`] — `httpx.ReadError`, `RemoteProtocolError`, `WriteError` are `TransportError` subclasses but not `TimeoutException` or `ConnectError`. They propagate unhandled as 500 instead of 503. Follows same pattern as client-api. — deferred, follows existing convention
- [x] [Review][Defer] **D5 — AsyncClient singleton never closed** [`core/ai_gateway.py:84`] — The module-level `_gw_client` is never `aclose()`d. Requires a FastAPI lifespan event handler. The process-lifetime singleton pattern is acceptable for now. — deferred, infrastructure concern

**Dismissed** (2 items — false positives):

- `server_default="pending"` in ORM model (Blind Hunter) — SQLAlchemy 2.0 `mapped_column` auto-quotes string `server_default` values. Migration uses `sa.text("'pending'")`. Both are correct.
- Missing FK on `opportunity_id` (Blind Hunter) — Documented architectural decision: opportunities live in the pipeline service schema, admin-api has no cross-schema FK.

**Verdict: 6 patch, 5 deferred, 2 dismissed.**

### Senior Developer Review — 2026-04-10

**REVIEW: Approve**

Second-pass adversarial review (Blind Hunter, Edge Case Hunter, Acceptance Auditor) confirms all 6 prior patch items have been resolved in code. All 9 acceptance criteria fully implemented with 47 passing tests (19 assignment + 28 suggestion). No new patch or blocking items found.

Re-review triage: 18 raw findings → 14 unique after dedup → **0 patch, 0 decision-needed, 7 defer (all pre-existing/documented), 7 dismissed (false positives or noise).**

Notable false positives dismissed:
- Nil UUID truthiness on override_framework_id — `uuid.UUID` is always truthy in Python; check is correct.
- Float precision concern — FLOAT is spec-mandated; `max(0.0, min(1.0, ...))` clamping is correct.
- Empty list after dedup — `min_length=1` schema validation fires before dedup; impossible edge case.

All prior deferred items (D1–D5) remain acknowledged by design. No regression from patch application.

## Dev Notes

### Architecture Context

This story extends the **admin-api** service (`services/admin-api/`, port 8002) built in S11.08. At the start of this story, admin-api has:
- Full CRUD infrastructure from S11.08: `config.py`, `core/security.py`, `dependencies.py`, schemas, services, router for compliance frameworks
- ORM models: `ComplianceFramework`, `OpportunityComplianceFramework`, `PlatformSettings` (from S11.01)
- Alembic migrations `001` (init) and `002` (compliance framework schema, S11.01)
- All R3/R4 patches from S11.08 verified resolved in code (`pydantic-settings>=2.0` + `asyncpg>=0.29` in pyproject.toml; `regulation_type` typed as `Literal` in the list endpoint)
- This story introduces: migration 003, FrameworkSuggestion ORM model, AI Gateway client, assignment service + router, suggestion service + router

### NEW: `framework_suggestions` Table (Not in S11.01 Migration)

The `framework_suggestions` table was described in the epic (S11.09) but was **NOT included** in S11.01's migration 002. This story must create it via migration **003** in admin-api. Do not attempt to add it to migration 002.

### `opportunity_id` Has No FK in admin-api

The `opportunity_compliance_frameworks` and `framework_suggestions` tables store `opportunity_id` as a plain UUID with **no FK constraint** to an opportunities table. Opportunities are managed in the pipeline service (separate service/schema), and admin-api has no cross-schema FK to them. Accept any UUID for `opportunity_id` without DB-level validation.

### AI Gateway Client Setup for admin-api

This is the **first AI Gateway invocation in admin-api**. Client-api has its AI Gateway client at `services/client-api/src/client_api/core/ai_gateway.py`. Replicate that module's exact pattern in `services/admin-api/src/admin_api/core/ai_gateway.py`, changing:
- Header `X-Caller-Service: admin-api` (not `client-api`)
- Env var `ADMIN_API_AIGW_BASE_URL` (not `CLIENT_API_AIGW_BASE_URL`)
- Env var `ADMIN_API_AIGW_TIMEOUT_SECONDS` (not `AIGW_TIMEOUT_SECONDS`)
- Import from `admin_api.config` (not `client_api.config`)

### Framework Suggestion Agent — Response Parsing Safety

The Framework Suggestion Agent may return `framework_id` values that are:
1. Malformed (non-UUID strings) — wrap in `try/except (ValueError, AttributeError)` and skip
2. Valid UUIDs that don't exist in `admin.compliance_frameworks` — verify each before storing, skip if absent

This prevents corrupted data in the `framework_suggestions` table. Never raise 404 to the client for agent-returned framework IDs — always skip silently and log a warning.

### Atomicity in Accept Flow (E11-R-007)

The accept flow has two writes that must succeed together:
1. Insert `OpportunityComplianceFramework` row (or confirm it already exists)
2. Update `FrameworkSuggestion.status = "accepted"`

Since the `process_suggestion` service function operates within a single `AsyncSession` that is wrapped by `get_db_session` (which commits on success and rolls back on exception), both writes in `await session.flush()` are part of the same transaction that commits when the router function returns. **Do not** call `await session.commit()` inside the service function — that is handled by the dependency. If a `flush()` fails, the exception propagates and `get_db_session` rolls back both writes.

Test this atomicity: after a successful accept, assert both DB rows exist. Cannot simulate a true mid-transaction failure in unit tests, but the integration test should verify the post-condition (both rows created) which proves they are in the same session scope.

### Admin JWT Auth Pattern (from S11.08)

Admin-api uses **HS256 symmetric JWT** with secret `ADMIN_API_JWT_SECRET`. All new endpoints use the same `get_admin_user` dependency and `CurrentAdmin` type alias from `admin_api.core.security`:
```python
from admin_api.core.security import get_admin_user, AdminUser
```

For 403 tests, use `_make_company_token()` helper (generates a non-platform_admin JWT).

### Test Fixture Pattern (from S11.08)

All test fixtures in admin-api follow this pattern from `test_compliance_frameworks.py`:
1. Session-scoped `admin_jwt_env_setup` (autouse) — sets `ADMIN_API_JWT_SECRET` and `ADMIN_API_DATABASE_URL`
2. Function-scoped `cf_client_and_session(admin_api_session_factory)` — yields `(client, session, admin_token)`
3. `app.dependency_overrides[get_db_session] = lambda: test_session_gen()` — injects test session
4. `async with session.begin()` wraps each test, rolled back in teardown

The `admin_api_session_factory: async_sessionmaker[AsyncSession]` fixture is expected in `conftest.py` (created as part of S11.08 test infrastructure). If it is missing, create a `services/admin-api/tests/conftest.py` with:
```python
@pytest_asyncio.fixture(scope="session")
async def admin_api_session_factory() -> async_sessionmaker[AsyncSession]:
    """Session-scoped async_sessionmaker for admin-api tests."""
    from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
    db_url = os.getenv(
        "ADMIN_API_DATABASE_URL",
        "postgresql+asyncpg://admin_api_role:admin_api_password@localhost:5432/eusolicit_test",
    )
    engine = create_async_engine(db_url)
    return async_sessionmaker(engine, expire_on_commit=False)
```

### AI Gateway Mocking in Tests

Use `respx` (same pattern as client-api tests). The `aigw_admin_env_setup` fixture must:
1. Set `os.environ["ADMIN_API_AIGW_BASE_URL"] = AIGW_TEST_BASE_URL`
2. Clear the `get_settings` LRU cache: `from admin_api.config import get_settings; get_settings.cache_clear()`
3. Reset the module-level singleton: `import admin_api.core.ai_gateway as _gw; _gw._gw_client = None`
4. Restore on teardown

In tests using `respx`:
```python
async with respx.MockRouter() as mock_router:
    mock_router.post(AIGW_SUGGEST_URL).mock(
        return_value=httpx.Response(200, json={"suggestions": [
            {"framework_id": str(fw_id), "confidence": 0.87},
        ]})
    )
    resp = await client.post(BASE_SUGGEST_URL, json=MOCK_SUGGESTION_REQUEST,
                              headers={"Authorization": f"Bearer {admin_token}"})
    assert resp.status_code == 200
```

For timeout test: `mock_router.post(AIGW_SUGGEST_URL).mock(side_effect=httpx.ReadTimeout)`.

### Routing: No Conflict between fs_v1 and cf_v1 Routers

The `POST /api/v1/admin/compliance-frameworks/suggest` (from `fs_v1` router) is distinct from `POST /api/v1/admin/compliance-frameworks` and `GET /api/v1/admin/compliance-frameworks/{framework_id}` (from `cf_v1` router):
- Different HTTP method+path combinations
- FastAPI matches by method first, then path
- The only potential confusion is `GET /admin/compliance-frameworks/suggest` (not defined here) vs `GET /admin/compliance-frameworks/{framework_id}` — this doesn't arise since no GET /suggest endpoint exists in this story

Register `fs_v1` router **before** `cf_v1` router in `main.py` to ensure literal `/suggest` is checked before any parameterized routes on the same prefix.

### Key Test Assertions

**test_assign_hybrid_national_and_eu_frameworks_to_same_opportunity:**
```python
fw_nat_id = await _create_framework(client, admin_token, FRAMEWORK_CREATE_NATIONAL)
fw_eu_id = await _create_framework(client, admin_token, FRAMEWORK_CREATE_EU)
opp_id = str(uuid.uuid4())

# Assign both frameworks to same opportunity
resp = await client.post(
    f"{BASE_ASSIGN_URL}/{opp_id}/compliance-frameworks",
    json={"framework_ids": [fw_nat_id, fw_eu_id]},
    headers={"Authorization": f"Bearer {admin_token}"},
)
assert resp.status_code == 201
assert len(resp.json()) == 2

# Verify list shows both
list_resp = await client.get(
    f"{BASE_ASSIGN_URL}/{opp_id}/compliance-frameworks",
    headers={"Authorization": f"Bearer {admin_token}"},
)
assert list_resp.status_code == 200
fw_ids_in_list = {f["id"] for f in list_resp.json()}
assert fw_nat_id in fw_ids_in_list
assert fw_eu_id in fw_ids_in_list
```

**test_accept_creates_assignment_atomically_status_and_row_together:**
```python
# Setup: suggestion in DB with pending status
fw_id = await _create_framework_in_db(session)
suggestion = FrameworkSuggestion(
    opportunity_id=uuid.uuid4(), framework_id=fw_id,
    confidence=0.9, status="pending"
)
session.add(suggestion)
await session.flush()
suggestion_id = str(suggestion.id)
opp_id = suggestion.opportunity_id

# Accept suggestion
resp = await client.patch(
    f"{BASE_SUGGESTIONS_URL}/{suggestion_id}",
    json={"action": "accept"},
    headers={"Authorization": f"Bearer {admin_token}"},
)
assert resp.status_code == 200
body = resp.json()
assert body["status"] == "accepted"
assert body["reviewed_by"] is not None

# Verify assignment row exists in DB
assignment = await session.scalar(
    select(OpportunityComplianceFramework).where(
        OpportunityComplianceFramework.opportunity_id == opp_id,
        OpportunityComplianceFramework.framework_id == fw_id,
    )
)
assert assignment is not None  # Both writes succeeded atomically
```

**test_accept_with_override_framework_assigns_override:**
```python
# suggestion.framework_id = fw_id_original
# override: fw_id_override (different framework)
resp = await client.patch(
    f"{BASE_SUGGESTIONS_URL}/{suggestion_id}",
    json={"action": "accept", "override_framework_id": str(fw_id_override)},
    headers={"Authorization": f"Bearer {admin_token}"},
)
assert resp.status_code == 200
body = resp.json()
assert body["effective_framework_id"] == str(fw_id_override)  # override used
# Verify assignment uses override, not original
assignment = await session.scalar(select(OpportunityComplianceFramework).where(
    OpportunityComplianceFramework.framework_id == fw_id_override
))
assert assignment is not None
```

**test_reject_does_not_create_assignment:**
```python
resp = await client.patch(
    f"{BASE_SUGGESTIONS_URL}/{suggestion_id}",
    json={"action": "reject"},
    headers={"Authorization": f"Bearer {admin_token}"},
)
assert resp.status_code == 200
assert resp.json()["status"] == "rejected"
# No assignment created
count = await session.scalar(
    select(func.count()).select_from(OpportunityComplianceFramework).where(
        OpportunityComplianceFramework.opportunity_id == opp_id
    )
)
assert count == 0
```

### Epic Test ID Coverage

| Epic Test ID | Priority | Covered By Story 11.9 |
|-------------|----------|----------------------|
| **E11-P0-007** | P0 | `TestAC1Authorization` tests 4 (GET suggestions 403) and 6 (PATCH suggestion 403) |
| **E11-P0-008** | P0 | `test_suggest_agent_timeout_returns_503_agent_unavailable`, `test_suggest_agent_500_returns_503_not_500` |
| **E11-P1-011** | P1 | `TestAC3AssignFrameworks` + `TestAC4ListAssigned` + `TestAC5RemoveAssignment` (full assignment CRUD) |
| **E11-P1-012** | P1 | `test_assign_hybrid_national_and_eu_frameworks_to_same_opportunity`, `test_list_returns_both_frameworks_after_hybrid_assignment` |
| **E11-P1-013** | P1 | `test_suggest_stores_suggestions_with_pending_status`, `test_list_suggestions_returns_paginated_response` |
| **E11-P1-014** | P1 | `test_accept_suggestion_creates_assignment_in_db`, `test_accept_creates_assignment_atomically_status_and_row_together` |
| **E11-P1-015** | P1 | `test_reject_does_not_create_assignment`, `test_reject_suggestion_returns_200_with_rejected_status` |
| **E11-P2-007** | P2 | `test_accept_with_override_framework_assigns_override`, `test_accept_with_invalid_override_framework_id_returns_404` |

### Risk Coverage

| Risk ID | Description | Test Coverage in this Story |
|---------|-------------|----------------------------|
| **E11-R-001** | AI Gateway error handling — 30s timeout, structured 503 | `test_suggest_agent_timeout_returns_503_agent_unavailable` (timeout), `test_suggest_agent_500_returns_503_not_500` (5xx) |
| **E11-R-003** | Admin endpoint authorization gaps | Full `TestAC1Authorization` in both test files: company JWT → 403; missing JWT → 401 for all 6 new endpoints |
| **E11-R-007** | Suggestion queue state atomicity | `test_accept_creates_assignment_atomically_status_and_row_together`: post-condition check confirms both rows; `test_reject_does_not_create_assignment`: reject does not auto-assign |
| **E11-R-010** | Hybrid national+EU assignment edge cases | `test_assign_hybrid_national_and_eu_frameworks_to_same_opportunity`, `test_delete_one_of_two_frameworks_leaves_other_intact` |

### Red Phase Failure Mode

All 47 new tests will fail with `404 Not Found` until routers are registered in `main.py`. Once registered but before security is added, auth tests (401/403) will return 200/201 instead. The `admin_jwt_env_setup` and `aigw_admin_env_setup` fixtures handle `ImportError` / `AttributeError` gracefully so collection does not fail in RED phase.

**Primary RED phase failure:** `404 Not Found` on all endpoints.

### Project Structure Notes

```
services/admin-api/
├── alembic/versions/
│   ├── 001_initial.py          # DO NOT TOUCH
│   ├── 002_compliance_framework_schema.py  # DO NOT TOUCH
│   └── 003_framework_suggestions.py       # CREATE (Task 1)
├── src/admin_api/
│   ├── main.py                 # MODIFY — add fa_v1 and fs_v1 routers (Task 8)
│   ├── config.py               # MODIFY — add AIGW settings (Task 4.2)
│   ├── dependencies.py         # DO NOT TOUCH
│   ├── models/
│   │   ├── __init__.py         # MODIFY — export FrameworkSuggestion (Task 2.2)
│   │   ├── base.py             # DO NOT TOUCH
│   │   ├── enums.py            # DO NOT TOUCH
│   │   ├── compliance_framework.py         # DO NOT TOUCH
│   │   ├── opportunity_compliance_framework.py  # DO NOT TOUCH
│   │   ├── platform_settings.py            # DO NOT TOUCH
│   │   └── framework_suggestion.py         # CREATE (Task 2.1)
│   ├── core/
│   │   ├── __init__.py         # DO NOT TOUCH
│   │   ├── security.py         # DO NOT TOUCH
│   │   └── ai_gateway.py       # CREATE (Task 4.1)
│   ├── schemas/
│   │   ├── __init__.py         # DO NOT TOUCH
│   │   ├── compliance_framework.py  # DO NOT TOUCH
│   │   ├── framework_assignment.py  # CREATE (Task 3.1)
│   │   └── framework_suggestion.py  # CREATE (Task 3.2)
│   └── services/
│       ├── __init__.py         # DO NOT TOUCH
│       ├── compliance_framework_service.py  # DO NOT TOUCH
│       ├── framework_assignment_service.py  # CREATE (Task 5)
│       └── framework_suggestion_service.py  # CREATE (Task 6)
│   api/v1/
│       ├── __init__.py         # DO NOT TOUCH
│       ├── compliance_frameworks.py  # DO NOT TOUCH (S11.08 router)
│       ├── framework_assignments.py  # CREATE (Task 7.1)
│       └── framework_suggestions.py  # CREATE (Task 7.2)
└── tests/api/
    ├── conftest.py             # VERIFY EXISTS (admin_api_session_factory); CREATE if missing
    ├── test_compliance_frameworks.py  # DO NOT TOUCH (S11.08 tests)
    ├── test_framework_assignments.py  # CREATE (Task 9.1)
    └── test_framework_suggestions.py  # CREATE (Task 9.2)
```

### References

- Epic S11.09 spec: [Source: eusolicit-docs/planning-artifacts/epic-11-grants-compliance.md#S11.09]
- DB schema (OCF join table): [Source: eusolicit-docs/implementation-artifacts/11-1-espd-profile-compliance-framework-db-schema-migrations.md#AC3]
- S11.08 admin-api infrastructure (config, security, deps, session pattern, test patterns): [Source: eusolicit-docs/implementation-artifacts/11-8-compliance-framework-crud-api-admin.md#Dev Notes]
- AI Gateway client pattern (client-api): [Source: eusolicit-docs/implementation-artifacts/11-4-grant-eligibility-agent-integration.md#Dev Notes]
- Epic test design — test IDs E11-P0-007, E11-P1-011..015, E11-P2-007, risks E11-R-001/003/007/010: [Source: eusolicit-docs/test-artifacts/test-design-epic-11.md]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

- Migration ran against wrong DB initially (alembic.ini targets `eusolicit`; test DB needs `DATABASE_URL` env var pointing to `eusolicit_test`). Fixed by running: `DATABASE_URL="postgresql+asyncpg://migration_role:migration_password@localhost:5432/eusolicit_test" python3 -m alembic upgrade head`
- `ADMIN_API_AIGW_BASE_URL` made optional (default `""`) in Settings to avoid breaking existing `test_compliance_frameworks.py` tests that don't set this env var.
- client-api's AI Gateway client is at `services/client-api/src/client_api/services/ai_gateway_client.py` (not `core/`) — used as pattern reference. Admin-api's client follows same structure but placed at `core/ai_gateway.py` per story spec, uses persistent `AsyncClient` (vs. client-api's per-call approach), and exposes as async generator Depends.

### Completion Notes List

- ✅ **Task 1 (Migration 003)**: Created `alembic/versions/003_framework_suggestions.py` with `admin.framework_suggestions` table, check constraint on `status`, FK to `compliance_frameworks` with CASCADE delete, and two indexes. Applied to both `eusolicit` and `eusolicit_test` databases.
- ✅ **Task 2 (ORM Model)**: Created `FrameworkSuggestion` model at `models/framework_suggestion.py`; updated `models/__init__.py` to export it.
- ✅ **Task 3 (Schemas)**: Created `schemas/framework_assignment.py` (AssignFrameworksRequest, OpportunityFrameworkAssignmentResponse, ComplianceFrameworkWithAssignmentResponse) and `schemas/framework_suggestion.py` (OpportunityMetadata, SuggestFrameworksRequest, FrameworkSuggestionResponse, FrameworkSuggestionListResponse, PatchSuggestionRequest).
- ✅ **Task 4 (AI Gateway Client)**: Created `core/ai_gateway.py` with AiGatewayClient (persistent AsyncClient, AiGatewayTimeoutError, AiGatewayUnavailableError, module-level singleton, async generator dependency). Updated `config.py` with `admin_api_aigw_base_url` (default `""`) and `admin_api_aigw_timeout_seconds` (default 30.0). httpx was already in main deps.
- ✅ **Task 5 (Assignment Service)**: `framework_assignment_service.py` — assign_frameworks (idempotent per-pair), list_assigned_frameworks (JOIN query), remove_assignment (404 guard).
- ✅ **Task 6 (Suggestion Service)**: `framework_suggestion_service.py` — suggest_frameworks (agent call + safe parsing + DB storage), list_suggestions (paginated + filtered), process_suggestion (accept/reject with 409 guard and atomicity).
- ✅ **Task 7 (Routers)**: `api/v1/framework_assignments.py` (POST/GET/DELETE for assignment CRUD) and `api/v1/framework_suggestions.py` (POST suggest, GET list, PATCH accept/reject).
- ✅ **Task 8 (main.py)**: Registered `fs_v1` before `cf_v1` to ensure literal `/suggest` path takes precedence, then `fa_v1`.
- ✅ **Task 9 (Tests)**: 19 tests in `test_framework_assignments.py` and 28 tests in `test_framework_suggestions.py`. All 47 new + 40 existing = **87 tests pass, 0 failures, 0 regressions**.
- ✅ **Resolved review finding [Patch] P1**: Added `import math` + `if not math.isfinite(raw_conf): raw_conf = 0.0` guard in `framework_suggestion_service.py` before confidence clamping — prevents NaN/Infinity from corrupting DB and JSON serialization.
- ✅ **Resolved review finding [Patch] P2**: Added `if not isinstance(raw_suggestions, list): raw_suggestions = []` after `.get("suggestions", [])` call — prevents `TypeError` when agent returns null/non-list.
- ✅ **Resolved review finding [Patch] P3**: Added `framework_ids = list(dict.fromkeys(framework_ids))` dedup at start of `assign_frameworks()` — prevents IntegrityError on duplicate UUIDs in request.
- ✅ **Resolved review finding [Patch] P4**: Changed `status: str | None = Query(None)` to `status: Literal["pending", "accepted", "rejected"] | None = Query(None)` in framework_suggestions router — now returns 422 on invalid status values.
- ✅ **Resolved review finding [Patch] P5**: Removed unused `import uuid` from `framework_assignment_service.py` and unused `from admin_api.schemas.compliance_framework import ComplianceFrameworkResponse` from `schemas/framework_assignment.py`.
- ✅ **Resolved review finding [Patch] P6**: Added `assert body.get("message") == "AI features are temporarily unavailable. Please try again."` to both 503 timeout and 503 5xx tests in `test_framework_suggestions.py`.
- **Post-patch test run**: 87/87 API + unit tests pass, 0 regressions. Integration tests (alembic CLI) remain in pre-existing failing state (environment issue, not caused by this story).

### File List

services/admin-api/alembic/versions/003_framework_suggestions.py
services/admin-api/src/admin_api/config.py
services/admin-api/src/admin_api/core/ai_gateway.py
services/admin-api/src/admin_api/models/__init__.py
services/admin-api/src/admin_api/models/framework_suggestion.py
services/admin-api/src/admin_api/schemas/framework_assignment.py
services/admin-api/src/admin_api/schemas/framework_suggestion.py
services/admin-api/src/admin_api/services/framework_assignment_service.py
services/admin-api/src/admin_api/services/framework_suggestion_service.py
services/admin-api/src/admin_api/api/v1/framework_assignments.py
services/admin-api/src/admin_api/api/v1/framework_suggestions.py
services/admin-api/src/admin_api/main.py
services/admin-api/tests/api/test_framework_assignments.py
services/admin-api/tests/api/test_framework_suggestions.py

### Change Log

- 2026-04-10: Story 11.9 implemented — Framework Assignment & Auto-Suggestion API (Admin). Added migration 003, FrameworkSuggestion ORM model, AI Gateway client for admin-api, framework assignment service (assign/list/remove), framework suggestion service (suggest/list/accept/reject), two new routers, main.py router registration, and 47 new tests (87 total pass, 0 regressions).
- 2026-04-10: Addressed code review findings — 6 patch items resolved (P1: NaN/Infinity confidence guard; P2: non-list suggestions guard; P3: duplicate UUID dedup; P4: Literal status query param; P5: unused imports removed; P6: 503 message assertion added to tests). 87/87 API+unit tests pass after patches.
