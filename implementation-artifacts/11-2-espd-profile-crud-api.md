# Story 11.2: ESPD Profile CRUD API

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **company admin or bid manager**,
I want to create, list, view, update, and delete named ESPD profiles for my company,
so that my company can maintain multiple reusable ESPD documents mapped to EU ESPD Parts II–V, ready to use across different procurement tender submissions.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/espd-profiles` creates a new ESPD profile for the authenticated user's company. Accepts `profile_name` (string, required, non-empty, max 255 chars) and `espd_data` (JSONB dict, optional, defaults to `{}`). Returns HTTP 201 with the created profile. The profile's `company_id` is derived exclusively from the JWT — any `company_id` field supplied in the request body is ignored. Requires `admin` or `bid_manager` role; `contributor`, `reviewer`, and `read_only` receive HTTP 403.

2. **AC2** — `GET /api/v1/espd-profiles` returns HTTP 200 with a list of all ESPD profiles belonging to the authenticated user's company, ordered by `created_at` descending. Each item includes `id`, `company_id`, `profile_name`, `espd_data`, `created_at`, `updated_at`. Returns `{"profiles": [], "total": 0}` if no profiles exist. Accessible to all authenticated company members.

3. **AC3** — `GET /api/v1/espd-profiles/{profile_id}` returns HTTP 200 with a single ESPD profile if it exists and belongs to the authenticated user's company. Returns HTTP 404 if the profile does not exist OR if it belongs to a different company (no HTTP 403 — 404 prevents UUID enumeration). Accessible to all authenticated company members.

4. **AC4** — `PATCH /api/v1/espd-profiles/{profile_id}` accepts an optional `profile_name` (string) and/or an optional `espd_data` (dict). Updates the profile in place: `profile_name` is replaced if provided; `espd_data` is merged at the top ESPD Part level (provided Part keys replace the existing Part content; absent Part keys are left unchanged). Returns HTTP 200 with the updated profile. Returns HTTP 404 if not found or cross-company. Requires `admin` or `bid_manager` role.

5. **AC5** — `DELETE /api/v1/espd-profiles/{profile_id}` permanently deletes the specified profile and returns HTTP 204 No Content. Returns HTTP 404 if not found or cross-company. Requires `admin` or `bid_manager` role.

6. **AC6** — Company-scoped RLS enforcement on all 5 endpoints: `company_id` is always derived from the authenticated JWT and is never user-configurable. `GET /espd-profiles/{id}`, `PATCH /espd-profiles/{id}`, and `DELETE /espd-profiles/{id}` return HTTP 404 — not HTTP 403 — when the profile UUID does not belong to the JWT company. HTTP 404 prevents cross-company profile enumeration via UUID guessing.

7. **AC7** — `espd_data` PATCH merge semantics: the top-level ESPD Part keys (`part_ii`, `part_iii`, `part_iv`, `part_v`) are merged as units. A PATCH providing only `{"part_iii": {...}}` updates `part_iii` while leaving `part_ii`, `part_iv`, and `part_v` exactly as they were. Merge is not recursive — the entire Part sub-dict is replaced when the key is present in the PATCH body.

8. **AC8** — `espd_data` structure validation: if `espd_data` is provided in POST or PATCH, it must be a JSON object (dict). If any recognised Part key (`part_ii`, `part_iii`, `part_iv`, `part_v`) is present, its value must be a JSON object (dict); a non-dict value returns HTTP 422 with a field-level error detail. Additional top-level keys outside the four Part keys are accepted without error (permissive for forward compatibility).

9. **AC9** — Unauthenticated requests to all 5 endpoints return HTTP 401. `POST`, `PATCH`, and `DELETE` require `admin` or `bid_manager` role; `contributor`, `reviewer`, and `read_only` roles receive HTTP 403. `GET` (list and detail) is accessible to all authenticated company members regardless of role.

## Tasks / Subtasks

- [x] Task 1: Rewrite `src/client_api/schemas/espd.py` (AC: 1, 2, 3, 4, 8)
  - [x] 1.1 Add `from __future__ import annotations` header
  - [x] 1.2 Remove all S2.12 schema classes (`ESPDFieldValues`, `ESPDFieldValuesPartial`, `ESPDProfilePutRequest`, `ESPDProfilePatchRequest`, `ESPDProfileResponse`, `ESPDProfileVersionsResponse`) — these are superseded
  - [x] 1.3 Define `ESPDData(BaseModel)` with `model_config = ConfigDict(extra="allow")` and four optional Part fields, each validated as `dict | None = None`: `part_ii: dict | None = None`, `part_iii: dict | None = None`, `part_iv: dict | None = None`, `part_v: dict | None = None`. Add a `@field_validator("part_ii", "part_iii", "part_iv", "part_v", mode="before")` that raises `ValueError("must be a JSON object")` if the value is not None and not a dict.
  - [x] 1.4 Define `ESPDProfileCreateRequest(BaseModel)` with `profile_name: str = Field(..., min_length=1, max_length=255)` (required) and `espd_data: ESPDData = Field(default_factory=ESPDData)` (optional). Ignore any `company_id` field — do not include it in the model.
  - [x] 1.5 Define `ESPDProfilePatchRequest(BaseModel)` with `profile_name: str | None = Field(None, min_length=1, max_length=255)` and `espd_data: ESPDData | None = None`. Both optional.
  - [x] 1.6 Define `ESPDProfileResponse(BaseModel)` with `model_config = ConfigDict(from_attributes=True)`: `id: UUID`, `company_id: UUID`, `profile_name: str`, `espd_data: dict`, `created_at: datetime`, `updated_at: datetime`
  - [x] 1.7 Define `ESPDProfileListResponse(BaseModel)` with `profiles: list[ESPDProfileResponse]` and `total: int`

- [x] Task 2: Rewrite `src/client_api/services/espd_service.py` (AC: 1–9)
  - [x] 2.1 Add `from __future__ import annotations` header, `structlog` logger, and all required imports: `UUID`, `AsyncSession`, `select`, `ESPDProfile`, `CurrentUser`, `ForbiddenError`, `HTTPException`
  - [x] 2.2 Remove all S2.12 functions (`_get_latest_or_none`, `_get_next_version`, `_espd_to_snapshot`, `upsert_espd_profile_full`, `upsert_espd_profile_partial`, `get_espd_profile`, `get_espd_profile_versions`) — these are superseded by the multi-profile CRUD functions below
  - [x] 2.3 Implement helper `def _assert_company_owns_profile(profile: ESPDProfile, current_user: CurrentUser) -> None` — raises `HTTPException(status_code=404, detail="ESPD profile not found")` if `profile.company_id != current_user.company_id`. Returns 404 (not 403) to prevent enumeration (AC6).
  - [x] 2.4 Implement `async def create_espd_profile(request: ESPDProfileCreateRequest, current_user: CurrentUser, session: AsyncSession) -> ESPDProfile`:
    - Compute `espd_data_dict = request.espd_data.model_dump(exclude_none=True, mode="json")` — yields only provided Parts, dropping None values
    - Create `profile = ESPDProfile(company_id=current_user.company_id, profile_name=request.profile_name, espd_data=espd_data_dict)`
    - `session.add(profile)` → `await session.flush()` → `await session.refresh(profile)`
    - Log `espd_profile.created` with `company_id`, `profile_id`, `profile_name`
    - Return `profile`
  - [x] 2.5 Implement `async def list_espd_profiles(current_user: CurrentUser, session: AsyncSession) -> list[ESPDProfile]`:
    - Execute `SELECT * FROM client.espd_profiles WHERE company_id = :cid ORDER BY created_at DESC`
    - Return list (may be empty)
  - [x] 2.6 Implement `async def get_espd_profile(profile_id: UUID, current_user: CurrentUser, session: AsyncSession) -> ESPDProfile`:
    - `SELECT * FROM client.espd_profiles WHERE id = :profile_id` → `scalar_one_or_none()`
    - Raise `HTTPException(404, "ESPD profile not found")` if result is None
    - Call `_assert_company_owns_profile(profile, current_user)` — raises 404 if cross-company
    - Return `profile`
  - [x] 2.7 Implement `async def update_espd_profile(profile_id: UUID, request: ESPDProfilePatchRequest, current_user: CurrentUser, session: AsyncSession) -> ESPDProfile`:
    - Fetch profile (same fetch-then-ownership pattern as `get_espd_profile`)
    - If `request.profile_name is not None`: `profile.profile_name = request.profile_name`
    - If `request.espd_data is not None`: merge Part-level — `patch_parts = request.espd_data.model_dump(exclude_none=True, mode="json")`; `profile.espd_data = {**profile.espd_data, **patch_parts}` — this replaces provided Parts, leaves absent Parts unchanged
    - `await session.flush()` → `await session.refresh(profile)`
    - Log `espd_profile.updated` with `company_id`, `profile_id`, `changed_keys`
    - Return `profile`
  - [x] 2.8 Implement `async def delete_espd_profile(profile_id: UUID, current_user: CurrentUser, session: AsyncSession) -> None`:
    - Fetch profile (same fetch-then-ownership pattern)
    - `await session.delete(profile)` → `await session.flush()`
    - Log `espd_profile.deleted` with `company_id`, `profile_id`

- [x] Task 3: Rewrite `src/client_api/api/v1/espd.py` (AC: 1–9)
  - [x] 3.1 Add `from __future__ import annotations` header
  - [x] 3.2 Change router definition to `router = APIRouter(prefix="/espd-profiles", tags=["espd"])` — replaces the old `/companies` prefix from S2.12
  - [x] 3.3 Implement `POST /` — inject `body: ESPDProfileCreateRequest`, `session`, `current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]`; call `espd_service.create_espd_profile(body, current_user, session)`; return `ESPDProfileResponse.model_validate(profile)` with `status_code=201`
  - [x] 3.4 Implement `GET /` — inject `session`, `current_user: Annotated[CurrentUser, Depends(get_current_user)]`; call `espd_service.list_espd_profiles(current_user, session)`; return `ESPDProfileListResponse(profiles=[ESPDProfileResponse.model_validate(p) for p in profiles], total=len(profiles))`
  - [x] 3.5 Implement `GET /{profile_id}` — inject `profile_id: UUID`, `session`, `current_user: Annotated[CurrentUser, Depends(get_current_user)]`; call `espd_service.get_espd_profile(profile_id, current_user, session)`; return `ESPDProfileResponse.model_validate(profile)`
  - [x] 3.6 Implement `PATCH /{profile_id}` — inject `profile_id: UUID`, `body: ESPDProfilePatchRequest`, `session`, `current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]`; call `espd_service.update_espd_profile(profile_id, body, current_user, session)`; return `ESPDProfileResponse.model_validate(profile)`
  - [x] 3.7 Implement `DELETE /{profile_id}` — inject `profile_id: UUID`, `session`, `current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]`; call `espd_service.delete_espd_profile(profile_id, current_user, session)`; return `Response(status_code=204)`. Import `Response` from `fastapi`.

- [x] Task 4: Verify `src/client_api/main.py` router registration (AC: all)
  - [x] 4.1 Confirm `api_v1_router.include_router(espd_v1.router)` is already present — no change needed; the router prefix change in espd.py from `/companies` to `/espd-profiles` automatically changes all URLs

- [x] Task 5: Replace `tests/api/test_espd_profile.py` with S11.02 tests (AC: 1–9)
  - [x] 5.1 Replace entire file contents — the old S2.12 tests (`TestAC1GetEspdProfile`, `TestAC2PutEspdProfile`, `TestAC3PatchEspdProfile`, `TestAC4RoleEnforcement`, `TestAC6VersionHistory`, `TestCrossTenantEspd`) test the old versioned API which no longer exists; they must be removed and replaced with tests for the new multi-profile CRUD API
  - [x] 5.2 Define fixture `espd_client_and_session` — follow the `company_client_and_session` pattern from `test_company_profile.py` exactly (register → verify email via SQL UPDATE → login → yield `(client, session, access_token, company_id_str)`)
  - [x] 5.3 Define helper `_register_and_verify_with_role(session, client, role)` — reuse the pattern from `test_company_profile.py` to create a user with a specified role in the same company (for role enforcement tests)
  - [x] 5.4 Define constants `ESPD_CREATE_PAYLOAD` and `ESPD_FULL_DATA` covering all four Part keys:
    ```python
    ESPD_FULL_DATA = {
        "part_ii": {"operator_name": "Test Corp", "registration_number": "BG123456789"},
        "part_iii": {"criminal_convictions": False, "corruption": False, "fraud": False},
        "part_iv": {"economic_financial": {"annual_turnover": 5000000}, "technical_professional": {}},
        "part_v": {"reduction_of_candidates": {}}
    }
    ESPD_CREATE_PAYLOAD = {"profile_name": "Test ESPD Profile", "espd_data": ESPD_FULL_DATA}
    ```
  - [x] 5.5 **Test class `TestAC1CreateEspdProfile`** (AC1, AC8, AC9, E11-P1-008)
    - `test_post_creates_profile_with_all_parts_returns_201` — POST with full espd_data → 201, response has `id`, `profile_name`, `espd_data` matching input, `company_id` matches JWT
    - `test_post_with_empty_espd_data_returns_201` — POST with `profile_name` only, no `espd_data` → 201, `espd_data == {}`
    - `test_post_missing_profile_name_returns_422` — POST without `profile_name` → 422 (AC8)
    - `test_post_empty_profile_name_returns_422` — POST with `profile_name: ""` → 422 (AC8)
    - `test_post_profile_name_over_255_chars_returns_422` — POST with `profile_name: "x" * 256` → 422 (AC8)
    - `test_post_invalid_part_type_returns_422` — POST with `espd_data: {"part_iii": "not_a_dict"}` → 422 (AC8)
    - `test_unauthenticated_post_returns_401` — POST without token → 401 (AC9)
  - [x] 5.6 **Test class `TestAC2ListEspdProfiles`** (AC2, AC9, E11-P1-008)
    - `test_get_list_empty_returns_200_with_empty_list` — GET before any POST → 200, `profiles == []`, `total == 0`
    - `test_get_list_returns_profiles_for_company` — POST two profiles → GET list → 200, `total == 2`, both in `profiles`
    - `test_get_list_ordered_by_created_at_desc` — POST profile A then profile B → GET list → profile B appears first
    - `test_unauthenticated_list_returns_401` — GET without token → 401 (AC9)
  - [x] 5.7 **Test class `TestAC3GetEspdProfile`** (AC3, AC9, E11-P1-008, E11-P0-001)
    - `test_get_detail_returns_200_with_correct_data` — POST profile → GET /{id} → 200, data matches
    - `test_get_nonexistent_profile_returns_404` — GET with random UUID → 404 (AC6)
    - `test_unauthenticated_get_returns_401` — GET without token → 401 (AC9)
  - [x] 5.8 **Test class `TestAC4PatchEspdProfile`** (AC4, AC7, AC9, E11-P1-008, E11-P2-005)
    - `test_patch_profile_name_only_returns_200` — POST profile → PATCH with only `profile_name` → 200, `profile_name` updated, `espd_data` unchanged (AC4)
    - `test_patch_espd_data_part_level_merge` — POST profile with all 4 Parts → PATCH with only `{"espd_data": {"part_iii": {"criminal_convictions": True}}}` → 200, `part_iii` updated, `part_ii`/`part_iv`/`part_v` unchanged (AC7, E11-P2-005)
    - `test_patch_both_fields_returns_200` — POST profile → PATCH with both `profile_name` and `espd_data` → 200, both updated
    - `test_patch_empty_body_returns_200_unchanged` — POST profile → PATCH with `{}` → 200, no changes (no-op PATCH acceptable)
    - `test_patch_nonexistent_profile_returns_404` — PATCH with random UUID → 404 (AC6)
    - `test_unauthenticated_patch_returns_401` — PATCH without token → 401 (AC9)
  - [x] 5.9 **Test class `TestAC5DeleteEspdProfile`** (AC5, AC9, E11-P1-008)
    - `test_delete_profile_returns_204` — POST profile → DELETE → 204; subsequent GET → 404
    - `test_delete_nonexistent_profile_returns_404` — DELETE with random UUID → 404 (AC6)
    - `test_unauthenticated_delete_returns_401` — DELETE without token → 401 (AC9)
  - [x] 5.10 **Test class `TestAC6CrossCompanyRLS`** (AC6, E11-P0-001, E11-P2-004)
    - `test_company_a_cannot_get_company_b_profile_returns_404` — Company A creates profile; Company B token GET /espd-profiles/{company_a_profile_id} → 404 (E11-P0-001)
    - `test_company_a_cannot_patch_company_b_profile_returns_404` — Company A creates profile; Company B token PATCH → 404 (E11-P2-004)
    - `test_company_a_cannot_delete_company_b_profile_returns_404` — Company A creates profile; Company B token DELETE → 404 (E11-P2-004)
    - `test_post_company_id_in_body_is_ignored` — POST with explicit `company_id` of Company B in body → 201, created profile belongs to JWT company (Company A), not Company B (AC1, AC6)
    - `test_list_only_returns_own_company_profiles` — Company A creates 2 profiles; Company B creates 1 profile; Company A GET list → `total == 2` (only Company A's profiles)
  - [x] 5.11 **Test class `TestAC9RoleEnforcement`** (AC9)
    - `test_low_privilege_roles_cannot_post` (parametrized: contributor, reviewer, read_only) — 403 on POST
    - `test_low_privilege_roles_cannot_patch` (parametrized: contributor, reviewer, read_only) — 403 on PATCH
    - `test_low_privilege_roles_cannot_delete` (parametrized: contributor, reviewer, read_only) — 403 on DELETE
    - `test_bid_manager_can_post_and_patch_and_delete` — 201/200/204 for all write operations
    - `test_all_roles_can_get_list_and_detail` (parametrized: contributor, reviewer, read_only) — 200 on GET list and detail

## Dev Notes

### Critical Context: This Story Replaces the S2.12 ESPD API

**S2.12 (ESPD Profile CRUD) implemented a versioned, single-profile-per-company API at `/api/v1/companies/{company_id}/espd-profile`.** Story 11.01 changed the `client.espd_profiles` ORM model from a versioned-row design (`field_values` + `version`) to a named multi-profile design (`profile_name` + `espd_data`). As a result, the existing S2.12 implementation files are now broken — they reference columns that no longer exist. **This story must completely rewrite all three ESPD files** and replace the old test suite.

**Current state of files to rewrite (verify before starting):**
- `services/client-api/src/client_api/schemas/espd.py` — references `ESPDFieldValues`, `field_values`, `version` (all obsolete)
- `services/client-api/src/client_api/services/espd_service.py` — references `ESPDProfile.version`, `ESPDProfile.field_values` (columns no longer exist after migration 009)
- `services/client-api/src/client_api/api/v1/espd.py` — old prefix `/companies`, old endpoints (PUT full-replace, versioned PATCH)
- `services/client-api/tests/api/test_espd_profile.py` — tests old versioned API (PUT, GET /versions, cross-company 403 pattern); must be fully replaced

### Updated ORM Model (from Story 11.01)

The `ESPDProfile` model now has `profile_name` and `espd_data` instead of `field_values` and `version`. Use exactly as defined in `models/espd_profile.py`:

```python
class ESPDProfile(Base):
    __tablename__ = "espd_profiles"
    __table_args__ = (
        sa.Index("ix_espd_profiles_company_id", "company_id"),
        {"schema": "client"},
    )

    id: Mapped[uuid.UUID]            # UUID PK, default uuid4
    company_id: Mapped[uuid.UUID]    # FK → client.companies.id ON DELETE CASCADE
    profile_name: Mapped[str]        # VARCHAR(255), NOT NULL, default 'Default Profile'
    espd_data: Mapped[dict]          # JSONB, NOT NULL, default '{}'
    created_at: Mapped[datetime]     # TIMESTAMPTZ, server_default now()
    updated_at: Mapped[datetime]     # TIMESTAMPTZ, server_default now(), onupdate now()
```

**IMPORTANT:** There is no `version` column. There is no `field_values` column. Never reference these in the new implementation.

### Architecture Constraints (MUST FOLLOW)

**1. `from __future__ import annotations` at the top of every file.** Project-wide rule.

**2. `structlog` for all logging. No `print()` or `logging.getLogger()`:**
```python
import structlog
log = structlog.get_logger()
```

**3. `http_request: Request` injection pattern.** Not needed for ESPD CRUD as S11.02 does not write audit log entries (no `write_audit_entry` calls). The new design uses standard CRUD without versioned audit snapshots.

**4. `session.flush()` then `session.refresh()` after mutations.** Same pattern used in company_service.py. Do NOT call `session.commit()`.

**5. `model_dump(exclude_none=True, mode="json")` for JSONB serialization.** When building the `espd_data` dict from the `ESPDData` Pydantic model, always use `exclude_none=True` to drop omitted Parts, and `mode="json"` to ensure UUID/datetime serialization.

**6. No audit log in S11.02.** The ESPD CRUD in this story does not write `shared.audit_log` entries (unlike S2.12 which did). If audit logging of ESPD mutations is required in future, it can be added without breaking the current API contract.

### New API Endpoint URLs

```
POST   /api/v1/espd-profiles            → create profile (admin/bid_manager)
GET    /api/v1/espd-profiles            → list company profiles (any auth)
GET    /api/v1/espd-profiles/{id}       → get profile detail (any auth)
PATCH  /api/v1/espd-profiles/{id}       → update profile (admin/bid_manager)
DELETE /api/v1/espd-profiles/{id}       → delete profile (admin/bid_manager)
```

The old S2.12 endpoints at `/api/v1/companies/{company_id}/espd-profile` and `/api/v1/companies/{company_id}/espd-profile/versions` are removed.

### New Files to Create

None — all three implementation files already exist from S2.12 and will be rewritten in place.

### Files to REWRITE (Replace Content Entirely)

| File | Action |
|------|--------|
| `services/client-api/src/client_api/schemas/espd.py` | Full rewrite — new S11.02 schema classes |
| `services/client-api/src/client_api/services/espd_service.py` | Full rewrite — multi-profile CRUD functions |
| `services/client-api/src/client_api/api/v1/espd.py` | Full rewrite — new prefix `/espd-profiles`, 5 endpoints |
| `services/client-api/tests/api/test_espd_profile.py` | Full rewrite — replace S2.12 tests with S11.02 tests |

### Files NOT Changed

| File | Reason |
|------|--------|
| `src/client_api/main.py` | Already includes `espd_v1.router`; prefix change in router is sufficient |
| `src/client_api/models/espd_profile.py` | Updated by S11.01; correct model already in place |
| `alembic/versions/` | All migrations handled by S11.01 |

### espd_data JSONB Structure

The `espd_data` column stores the EU ESPD Parts II–V as a structured dict. All Parts are optional in S11.02 (Part III completeness is validated at export time in S11.03):

```json
{
  "part_ii": {
    "operator_name": "...",
    "address": "...",
    "contact_person": "...",
    "registration_number": "..."
  },
  "part_iii": {
    "exclusion_grounds": {},
    "criminal_convictions": false,
    "corruption": false,
    "fraud": false,
    "terrorist_offences": false
  },
  "part_iv": {
    "economic_financial": {},
    "technical_professional": {},
    "quality_assurance": {}
  },
  "part_v": {
    "reduction_of_candidates": {}
  }
}
```

A profile may have any subset of Parts (including an empty `{}`). Do not enforce that all four Parts are present in CRUD operations.

### PATCH Merge Example

```python
# Existing espd_data on profile:
existing_espd_data = {
    "part_ii": {"operator_name": "Test Corp"},
    "part_iii": {"criminal_convictions": False},
    "part_iv": {"economic_financial": {}},
    "part_v": {},
}

# PATCH request body:
patch_request = {"espd_data": {"part_iii": {"criminal_convictions": True, "fraud": False}}}

# After merge (part_iii replaced, others untouched):
result_espd_data = {
    "part_ii": {"operator_name": "Test Corp"},           # unchanged
    "part_iii": {"criminal_convictions": True, "fraud": False},  # replaced
    "part_iv": {"economic_financial": {}},               # unchanged
    "part_v": {},                                        # unchanged
}

# Implementation:
patch_parts = request.espd_data.model_dump(exclude_none=True, mode="json")
profile.espd_data = {**profile.espd_data, **patch_parts}
```

### Company-Scoped RLS Pattern

The new API design does NOT include `company_id` in the URL (unlike the old S2.12 design). The company is always determined from the JWT. The ownership guard uses `_assert_company_owns_profile()` which returns **HTTP 404** (not 403) to prevent enumeration:

```python
def _assert_company_owns_profile(profile: ESPDProfile, current_user: CurrentUser) -> None:
    """Raise 404 if the profile does not belong to the current user's company.
    
    Returns 404 (not 403) to prevent UUID enumeration by company users.
    See E11-R-005 mitigation in test-design-epic-11.md.
    """
    if profile.company_id != current_user.company_id:
        raise HTTPException(status_code=404, detail="ESPD profile not found")
```

For list operations, filter by `company_id` in the WHERE clause — do not return all profiles and filter in Python.

### Role Guard Pattern

Same pattern used in companies.py and the old espd.py:

```python
# POST, PATCH, DELETE — admin or bid_manager only:
current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]

# GET list + detail — any authenticated member:
current_user: Annotated[CurrentUser, Depends(get_current_user)]
```

`require_role("bid_manager")` permits `admin` and `bid_manager` roles (it enforces minimum privilege level, not exact role match).

### Test Fixture Cross-Company Pattern

For cross-company RLS tests, create two companies in the same test using the shared session pattern:

```python
# Company A — registered user from the espd_client_and_session fixture
# Company B — registered inline using a second registration call within the test

# Register Company B inline:
uid_b = uuid.uuid4().hex[:8]
reg_b = await client.post("/api/v1/auth/register", json={
    "email": f"company-b-{uid_b}@test.com",
    "password": "SecurePass1",
    "full_name": "Company B Admin",
    "company_name": f"Company B {uid_b}",
})
# Verify email via SQL:
await session.execute(
    text("UPDATE client.users SET email_verified = TRUE WHERE email = :email"),
    {"email": f"company-b-{uid_b}@test.com"},
)
await session.flush()
# Login as Company B:
login_b = await client.post("/api/v1/auth/login", json={
    "email": f"company-b-{uid_b}@test.com", "password": "SecurePass1"
})
token_b = login_b.json()["access_token"]
```

Then use `token_b` to attempt access to Company A's profile IDs.

### ESPDData Schema with Part Validator

```python
from pydantic import BaseModel, ConfigDict, Field, field_validator

class ESPDData(BaseModel):
    """Structured ESPD Parts II–V. All parts optional."""
    model_config = ConfigDict(extra="allow")

    part_ii: dict | None = None
    part_iii: dict | None = None
    part_iv: dict | None = None
    part_v: dict | None = None

    @field_validator("part_ii", "part_iii", "part_iv", "part_v", mode="before")
    @classmethod
    def part_must_be_dict_or_none(cls, v: object) -> object:
        if v is not None and not isinstance(v, dict):
            raise ValueError("ESPD Part values must be JSON objects (dicts)")
        return v
```

### DELETE Endpoint Response

FastAPI requires returning `Response(status_code=204)` for 204 No Content — NOT `None` or an empty dict:

```python
from fastapi import Response

@router.delete("/{profile_id}", status_code=204)
async def delete_espd_profile(...) -> Response:
    await espd_service.delete_espd_profile(profile_id, current_user, session)
    return Response(status_code=204)
```

### Test Coverage Alignment (from test-design-epic-11.md)

This story implements the following test scenario IDs from the epic test design. Tests in `test_espd_profile.py` must cover:

| Epic Test ID | Priority | Scenario | Expected |
|-------------|----------|----------|----------|
| **E11-P0-001** | P0 | Company A JWT → GET/PATCH/DELETE Company B profile UUID | 404 (not 403) for all three; see `TestAC6CrossCompanyRLS` |
| **E11-P1-008** | P1 | ESPD CRUD smoke — create, list, get, update, delete | All 5 endpoints functional, correct HTTP status codes |
| **E11-P2-004** | P2 | Company A cannot PATCH/DELETE Company B profile | 404 expected (enumeration prevention) |
| **E11-P2-005** | P2 | PATCH only `part_iii` → other Parts unchanged | Part-level merge correctness |

**Note on E11-P1-009 (espd_data validation, missing Part III → 422):** This test scenario is likely scoped to the auto-fill export endpoint in S11.03, not to the CRUD API. For S11.02 CRUD, Part III is optional (profiles may be populated incrementally). The AC8 validator in S11.02 covers the "invalid Part type" case (non-dict value → 422). The "missing Part III" validation is deferred to S11.03.

**Note on E11-R-005 (ESPD RLS, score 6):** The mitigation for this risk is implemented in AC6. Tests in `TestAC6CrossCompanyRLS` and `TestAC3GetEspdProfile` directly verify 404 (not 403) for cross-company access. This must pass 100% (P0 severity per QA gate).

### Relevant Existing Files (for pattern reference)

| File | Why Relevant |
|------|-------------|
| `services/client-api/src/client_api/services/company_service.py` | CRUD service pattern (`session.flush`, `session.refresh`, structlog) |
| `services/client-api/src/client_api/api/v1/companies.py` | Router pattern (`APIRouter`, `require_role`, `get_current_user`, `Depends`) |
| `services/client-api/tests/api/test_company_profile.py` | Test fixture pattern (`company_client_and_session`, `_register_and_verify_with_role`, cross-tenant pattern) |
| `services/client-api/tests/api/test_team_members.py` | Multi-step test pattern with role checks |
| `services/client-api/tests/conftest.py` | `client_api_session_factory`, `test_redis_client` fixtures |
| `services/client-api/src/client_api/core/security.py` | `require_role`, `get_current_user`, `CurrentUser` |
| `services/client-api/src/client_api/dependencies.py` | `get_db_session` |
| `eusolicit-docs/implementation-artifacts/11-1-espd-profile-compliance-framework-db-schema-migrations.md` | S11.01 context: updated ORM model, migration details, factory patterns |
| `eusolicit-docs/test-artifacts/test-design-epic-11.md` | Epic test scenarios, risk IDs, E11-P0-001, E11-P1-008, E11-P2-004, E11-P2-005 |

### pyproject.toml Test Config

No changes needed. Tests run with:
```bash
cd eusolicit-app/services/client-api
pytest tests/api/test_espd_profile.py -v
```

Same `pytest-asyncio`, `httpx`, `pytest_asyncio` stack as existing tests.
