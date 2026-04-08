# Story 2.12: ESPD Profile CRUD

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a company admin or bid manager,
I want to create, retrieve, update, and browse version history for my company's ESPD (European Single Procurement Document) profile,
so that my company can maintain a reusable set of pre-mapped answers for EU procurement exclusion/selection criteria across all tender submissions.

## Acceptance Criteria

1. **AC1** — `GET /api/v1/companies/{id}/espd-profile` returns the latest ESPD profile (the row with the highest `version` number for that `company_id`) as HTTP 200 for any authenticated company member. Returns 404 if no profile has been created yet. A cross-company request returns 403.

2. **AC2** — `PUT /api/v1/companies/{id}/espd-profile` accepts a full `field_values` object containing all four required top-level sections (`exclusion_grounds`, `economic_standing`, `technical_ability`, `quality_assurance`), inserts a new `espd_profiles` row with `version = (current_max_version + 1)`, and returns the new profile (HTTP 200). If no previous profile exists, `version = 1`. A PUT with any of the four required sections missing returns HTTP 422.

3. **AC3** — `PATCH /api/v1/companies/{id}/espd-profile` accepts a partial `field_values` object (any subset of top-level keys), merges it into the latest existing profile's `field_values` at the top level (overwriting provided keys, leaving absent keys unchanged), inserts a new row with the next version number, and returns the updated profile (HTTP 200). Returns 404 if no existing profile is found to merge into.

4. **AC4** — Only `admin` and `bid_manager` roles may call PUT or PATCH; `contributor`, `reviewer`, and `read_only` receive 403. GET is accessible to all authenticated company members.

5. **AC5** — The `version` field is a monotonic counter per company: the first write produces `version = 1`, each subsequent write increments it by 1. The GET endpoint always returns the row with the highest version.

6. **AC6** — `GET /api/v1/companies/{id}/espd-profile/versions` returns all stored version rows for that company ordered by `version` ascending (HTTP 200). Returns 404 if no versions exist.

7. **AC7** — Every PUT and PATCH call is audit-logged to `shared.audit_log` via `write_audit_entry` with `entity_type="espd_profile"`, `entity_id=<new_row.id>`. For the first write (no prior profile): `action_type="create"`, `before=None`, `after=<field_values_snapshot>`. For subsequent writes: `action_type="update"`, `before=<previous_field_values_snapshot>`, `after=<new_field_values_snapshot>`. `user_id` and `ip_address` are always set.

8. **AC8** — Unauthenticated requests return 401. Cross-company requests return 403. These checks must apply to all four ESPD endpoints.

## Tasks / Subtasks

- [ ] Task 1 — Create `src/client_api/schemas/espd.py` (AC: 2, 3, 6)
  - [ ] 1.1 Add `from __future__ import annotations` header
  - [ ] 1.2 Define `ESPDFieldValues(BaseModel)` — four **required** fields with no defaults: `exclusion_grounds: dict`, `economic_standing: dict`, `technical_ability: dict`, `quality_assurance: dict`; add `model_config = ConfigDict(extra="allow")` to accept additional top-level keys without rejection
  - [ ] 1.3 Define `ESPDFieldValuesPartial(BaseModel)` — all four sections optional with `None` default: `exclusion_grounds: dict | None = None`, `economic_standing: dict | None = None`, `technical_ability: dict | None = None`, `quality_assurance: dict | None = None`; same `ConfigDict(extra="allow")`; this model drives PATCH
  - [ ] 1.4 Define `ESPDProfilePutRequest(BaseModel)` with field `field_values: ESPDFieldValues`
  - [ ] 1.5 Define `ESPDProfilePatchRequest(BaseModel)` with field `field_values: ESPDFieldValuesPartial`
  - [ ] 1.6 Define `ESPDProfileResponse(BaseModel)` with `model_config = ConfigDict(from_attributes=True)`: `id: UUID`, `company_id: UUID`, `field_values: dict`, `version: int`, `created_at: datetime`, `updated_at: datetime`
  - [ ] 1.7 Define `ESPDProfileVersionsResponse(BaseModel)` with `versions: list[ESPDProfileResponse]` and `total: int`

- [ ] Task 2 — Create `src/client_api/services/espd_service.py` (AC: 1–8)
  - [ ] 2.1 Add `from __future__ import annotations` header, structlog logger (`log = structlog.get_logger()`), and all required imports: `UUID`, `AsyncSession`, `select`, `func`, `ESPDProfile`, `CurrentUser`, `write_audit_entry`, `ForbiddenError`, and HTTP exception
  - [ ] 2.2 Implement helper `_assert_own_company(company_id: UUID, current_user: CurrentUser) -> None` — raises `ForbiddenError("You do not have access to this company")` if `current_user.company_id != company_id`; call this at the start of every service function before any DB query (cross-tenant guard, AC8)
  - [ ] 2.3 Implement helper `async def _get_latest_or_none(company_id: UUID, session: AsyncSession) -> ESPDProfile | None` — executes `SELECT * FROM client.espd_profiles WHERE company_id=:id ORDER BY version DESC LIMIT 1`; returns scalar or None
  - [ ] 2.4 Implement helper `async def _get_next_version(company_id: UUID, session: AsyncSession) -> int` — executes `SELECT MAX(version) FROM client.espd_profiles WHERE company_id=:id`; returns `(max or 0) + 1`
  - [ ] 2.5 Implement helper `_espd_to_snapshot(profile: ESPDProfile) -> dict` — returns `{"field_values": profile.field_values, "version": profile.version}`; used to build `before`/`after` for audit log
  - [ ] 2.6 Implement `async def get_espd_profile(company_id: UUID, current_user: CurrentUser, session: AsyncSession) -> ESPDProfile`:
    - Call `_assert_own_company(company_id, current_user)`
    - Call `_get_latest_or_none(company_id, session)`
    - Raise `HTTPException(status_code=404, detail="ESPD profile not found")` if result is None
    - Return the found profile
  - [ ] 2.7 Implement `async def get_espd_profile_versions(company_id: UUID, current_user: CurrentUser, session: AsyncSession) -> list[ESPDProfile]`:
    - Call `_assert_own_company(company_id, current_user)`
    - Execute `SELECT * FROM client.espd_profiles WHERE company_id=:id ORDER BY version ASC`
    - Raise 404 if result is empty
    - Return list of profiles
  - [ ] 2.8 Implement `async def upsert_espd_profile_full(company_id: UUID, request: ESPDProfilePutRequest, current_user: CurrentUser, session: AsyncSession, ip_address: str | None = None) -> ESPDProfile`:
    - Call `_assert_own_company(company_id, current_user)`
    - Fetch `existing = await _get_latest_or_none(company_id, session)` for before-snapshot and to determine action_type
    - Set `before_snapshot = _espd_to_snapshot(existing) if existing else None`
    - Set `action_type = "create" if existing is None else "update"`
    - Compute `next_version = await _get_next_version(company_id, session)`
    - Create `new_profile = ESPDProfile(company_id=company_id, field_values=request.field_values.model_dump(mode="json"), version=next_version)`
    - `session.add(new_profile)` then `await session.flush()` then `await session.refresh(new_profile)`
    - Call `await write_audit_entry(session, user_id=current_user.user_id, action_type=action_type, entity_type="espd_profile", entity_id=new_profile.id, before=before_snapshot, after=_espd_to_snapshot(new_profile), ip_address=ip_address)`
    - Return `new_profile`
  - [ ] 2.9 Implement `async def upsert_espd_profile_partial(company_id: UUID, request: ESPDProfilePatchRequest, current_user: CurrentUser, session: AsyncSession, ip_address: str | None = None) -> ESPDProfile`:
    - Call `_assert_own_company(company_id, current_user)`
    - Fetch `existing = await _get_latest_or_none(company_id, session)` — raise `HTTPException(404, "ESPD profile not found")` if None
    - Set `before_snapshot = _espd_to_snapshot(existing)`
    - Merge: `merged_values = {**existing.field_values, **request.field_values.model_dump(exclude_none=True, mode="json")}`
    - Compute `next_version = await _get_next_version(company_id, session)`
    - Create `new_profile = ESPDProfile(company_id=company_id, field_values=merged_values, version=next_version)`
    - `session.add(new_profile)` → `await session.flush()` → `await session.refresh(new_profile)`
    - Call `await write_audit_entry(session, ..., action_type="update", entity_type="espd_profile", entity_id=new_profile.id, before=before_snapshot, after=_espd_to_snapshot(new_profile), ip_address=ip_address)`
    - Return `new_profile`

- [ ] Task 3 — Create `src/client_api/api/v1/espd.py` (AC: 1–8)
  - [ ] 3.1 Add `from __future__ import annotations` header
  - [ ] 3.2 Define `router = APIRouter(prefix="/companies", tags=["espd"])` — same prefix as companies.py; FastAPI merges routes at include time with no conflict
  - [ ] 3.3 Add `GET /{company_id}/espd-profile` — inject `session`, `current_user: Annotated[CurrentUser, Depends(get_current_user)]`; call `espd_service.get_espd_profile()`; return `ESPDProfileResponse.model_validate(profile)`
  - [ ] 3.4 Add `PUT /{company_id}/espd-profile` — inject `body: ESPDProfilePutRequest`, `http_request: Request`, `session`, `current_user: Annotated[CurrentUser, Depends(require_role("bid_manager"))]`; extract `ip_address = http_request.client.host if http_request.client else None`; call `espd_service.upsert_espd_profile_full()`; return `ESPDProfileResponse.model_validate(profile)`
  - [ ] 3.5 Add `PATCH /{company_id}/espd-profile` — same signature as PUT but uses `ESPDProfilePatchRequest` and calls `espd_service.upsert_espd_profile_partial()`
  - [ ] 3.6 Add `GET /{company_id}/espd-profile/versions` — inject `session`, `current_user: Annotated[CurrentUser, Depends(get_current_user)]`; call `espd_service.get_espd_profile_versions()`; return `ESPDProfileVersionsResponse(versions=[ESPDProfileResponse.model_validate(p) for p in profiles], total=len(profiles))`
  - [ ] 3.7 **IMPORTANT:** The `/versions` route path (`/{company_id}/espd-profile/versions`) must be registered **before** any route that also matches on `/{company_id}/espd-profile` to prevent route shadowing. In practice, since they are different paths, this is fine in FastAPI, but list the GET versions route first in the file.

- [ ] Task 4 — Register ESPD router in `src/client_api/main.py` (AC: all)
  - [ ] 4.1 Add `from client_api.api.v1 import espd as espd_v1` import
  - [ ] 4.2 Add `api_v1_router.include_router(espd_v1.router)` immediately after the companies router include line
  - [ ] 4.3 Verify FastAPI does not complain about duplicate prefix `/companies` — it will not; separate `include_router` calls with same prefix are merged correctly

- [ ] Task 5 — Alembic migration `008_espd_profile_unique_version.py` (AC: 5 — version integrity)
  - [ ] 5.1 Create file `alembic/versions/008_espd_profile_unique_version.py`; set `revision = "008"`, `down_revision = "007"`
  - [ ] 5.2 In `upgrade()`: add UNIQUE constraint: `op.create_unique_constraint("uq_espd_profiles_company_version", "espd_profiles", ["company_id", "version"], schema="client")` — enforces versioning invariant at DB level (resolves deferred-work item from Story 2.1)
  - [ ] 5.3 Also in `upgrade()`: add FK index: `op.create_index("ix_espd_profiles_company_id", "espd_profiles", ["company_id"], schema="client")` — resolves deferred-work item from Story 2.1 ("Missing FK indexes on espd_profiles.company_id")
  - [ ] 5.4 In `downgrade()`: `op.drop_index("ix_espd_profiles_company_id", table_name="espd_profiles", schema="client")` then `op.drop_constraint("uq_espd_profiles_company_version", "espd_profiles", schema="client")`
  - [ ] 5.5 Run `alembic upgrade head` in CI to confirm migration applies cleanly

- [ ] Task 6 — API integration tests `tests/api/test_espd_profile.py` (AC: 1–8)
  - [ ] 6.1 Define fixture `espd_client_and_session` — follow `company_client_and_session` pattern from `test_company_profile.py` exactly (register → verify email via SQL UPDATE → login → yield `(client, session, access_token, company_id_str)`)
  - [ ] 6.2 **Test E02-AC1/E02-P2-007** — GET before any PUT returns 404; PUT with all 4 sections → 200, `version == 1`; GET after PUT → 200, returns same profile; second PUT → `version == 2`
  - [ ] 6.3 **Test E02-P2-008** — PUT full profile; PATCH with only `exclusion_grounds` updated → 200, `exclusion_grounds` is new value, other 3 sections unchanged from original; version = 2
  - [ ] 6.4 **Test E02-P2-009** — 3 successive PUTs → GET /versions returns list of 3 items with `version` values 1, 2, 3 in ascending order; `total == 3`
  - [ ] 6.5 **Test E02-P2-010** — PUT missing `exclusion_grounds` → 422; PUT missing `economic_standing` → 422; PUT missing all 4 sections → 422; PUT with all 4 → 200 (boundary)
  - [ ] 6.6 **Test AC4/E02-P1-014** — Use `_register_and_verify_with_role` helper from `test_company_profile.py`; parametrize over contributor, reviewer, read_only → each receives 403 on PUT and PATCH; each receives 200 on GET; admin and bid_manager receive 200 on PUT and PATCH
  - [ ] 6.7 **Test AC3 — PATCH on non-existent profile** — PATCH before any PUT → 404
  - [ ] 6.8 **Test E02-P0-012 (cross-tenant ESPD)** — Create Company A + Company B (two separate fixtures); with Company A token: `GET /companies/{b_id}/espd-profile` → 403; `PUT /companies/{b_id}/espd-profile` → 403; `PATCH /companies/{b_id}/espd-profile` → 403; `GET /companies/{b_id}/espd-profile/versions` → 403
  - [ ] 6.9 **Test AC8 (unauthenticated)** — All four endpoints without Authorization header → 401

- [ ] Task 7 — Extend `tests/api/test_audit_trail.py` with ESPD mutations (AC: 7, E02-P1-018)
  - [ ] 7.1 Add test: first PUT `/espd-profile` → query `shared.audit_log` for most recent row with `entity_type="espd_profile"` → assert `action_type="create"`, `before` is None, `after` is non-null dict, `user_id` is set, `ip_address` is non-null
  - [ ] 7.2 Add test: second PUT `/espd-profile` → assert `action_type="update"`, `before` is non-null (contains previous `field_values`), `after` is non-null (contains new `field_values`)
  - [ ] 7.3 Add test: PATCH `/espd-profile` → assert `action_type="update"`, `before` and `after` non-null, `before["field_values"]` differs from `after["field_values"]` in the patched key

## Dev Notes

### Architecture Constraints (MUST FOLLOW)

**1. `from __future__ import annotations` at the top of every new file.**
Project-wide rule. Omitting it breaks forward-reference type hints on Python 3.12+.

**2. `structlog` for all logging. No `print()` or `logging.getLogger()`.**
```python
import structlog
log = structlog.get_logger()
```

**3. New files to create (none exist yet — verify before creating):**
```
eusolicit-app/services/client-api/
  src/client_api/
    schemas/espd.py                        ← NEW
    services/espd_service.py               ← NEW
    api/v1/espd.py                         ← NEW
  alembic/versions/008_espd_profile_unique_version.py  ← NEW
  tests/api/test_espd_profile.py           ← NEW
```

**4. Files to modify (EXISTING):**
```
src/client_api/main.py                     ← register espd_v1.router
tests/api/test_audit_trail.py              ← extend with ESPD audit assertions (Tasks 7.1–7.3)
```

**5. ESPDProfile ORM model already exists — do NOT recreate.**
`src/client_api/models/espd_profile.py` defines the ORM class with `__tablename__ = "espd_profiles"` and `__table_args__ = {"schema": "client"}`. It is already exported from `client_api.models`. Import it as:
```python
from client_api.models import ESPDProfile
```

**6. Versioning by INSERT, never UPDATE.**
The ESPD profile uses an append-only version history. Every PUT or PATCH **inserts a new row** — it never updates the existing row. The "latest" profile is always `SELECT ... ORDER BY version DESC LIMIT 1`. There is no `UPDATE` path in the service layer. This is intentional per the epic spec: "Store versions by inserting a new row on each update rather than overwriting."

**7. Computing next version — race condition mitigation.**
Migration 008 adds a UNIQUE constraint on `(company_id, version)`. If two concurrent requests try to insert the same version, the second insert will raise `IntegrityError`. The service does **not** need to handle this edge case explicitly — let the DB error propagate (becomes 500). The constraint is a correctness guardrail, not a hot path.

```python
# Correct pattern for next version:
result = await session.execute(
    select(func.max(ESPDProfile.version)).where(ESPDProfile.company_id == company_id)
)
next_version = (result.scalar() or 0) + 1
```

**8. PATCH merge — top-level dict merge only.**
PATCH merges at the top level of `field_values` only — it does NOT deep-merge nested dicts within a section. If the existing `economic_standing` is `{"revenue": 1000000}` and the patch provides `{"economic_standing": {"employees": 50}}`, the result is `{"economic_standing": {"employees": 50}}` (the entire section is replaced). This is intentional — deep merge would be overly complex and ambiguous.

```python
# Correct PATCH merge pattern:
patch_dict = request.field_values.model_dump(exclude_none=True, mode="json")
merged_values = {**existing.field_values, **patch_dict}
```

**9. `action_type` for ESPD audit entries.**
- First PUT (no prior profile exists): `action_type = "create"`, `before = None`
- Subsequent PUT/PATCH (prior profile exists): `action_type = "update"`, `before = _espd_to_snapshot(existing)`
- The distinction is determined at the service layer by checking whether `_get_latest_or_none()` returns a profile.

**10. `_assert_own_company` cross-tenant guard.**
Every service function must call this at the top before any DB query. Do NOT skip it for GET operations. The `company_memberships` check is already embedded in the JWT's `company_id` claim — the guard is a simple equality check, not a DB query.

```python
def _assert_own_company(company_id: UUID, current_user: CurrentUser) -> None:
    if current_user.company_id != company_id:
        raise ForbiddenError("You do not have access to this company")
```

**11. `http_request: Request` injection pattern (NOT `request`).**
Project rule established in Story 2.4 and enforced in members.py. Injecting as `request` conflicts with FastAPI's internal body parameter handling. Always use:
```python
http_request: Request,
```
IP extraction:
```python
ip_address = http_request.client.host if http_request.client else None
```

**12. `write_audit_entry` — never raises, no commit.**
Import from `client_api.services.audit_service`. It is wrapped in `try/except Exception` — it will never raise. Do not call `session.commit()` after it — the session is committed by `get_db_session` dependency wrapper.

**13. Alembic migration naming and schema.**
New migration: `008_espd_profile_unique_version.py`. Must use `schema="client"` in all `op.*` calls. Must set `down_revision = "007"`. Verify the latest migration file is `007_invitations.py` before creating 008.

**14. Router prefix duplication is safe.**
Both `companies.py` and `espd.py` use `prefix="/companies"`. FastAPI's `APIRouter.include_router()` simply concatenates routes — multiple routers with the same prefix work correctly without conflict.

**15. GET /espd-profile/versions — 404 semantics.**
If a company has never created an ESPD profile, both `GET /espd-profile` and `GET /espd-profile/versions` return 404. This is the correct behavior — do NOT return an empty list for versions (404 signals the resource doesn't exist yet).

**16. `model_dump(mode="json")` for JSONB serialization.**
When serializing Pydantic models to store in JSONB columns, always use `model_dump(mode="json")` to ensure UUIDs and datetimes are serialized as strings rather than native Python objects.

```python
# Correct:
field_values=request.field_values.model_dump(mode="json")
# Wrong (stores Python UUID objects in JSONB):
field_values=request.field_values.model_dump()
```

**17. ESPD field schema scope (from epic notes).**
The 4 required top-level sections mirror ESPD XML structure at a high level only. Each section is a freeform dict — do NOT try to validate nested keys within each section. The `extra="allow"` on `ESPDFieldValues` ensures additional top-level keys (beyond the 4 required) are also stored without error.

**18. No new `espd_profiles` columns or model changes needed.**
The ESPDProfile ORM model in `models/espd_profile.py` already has all required columns: `id`, `company_id`, `field_values` (JSONB), `version`, `created_at`, `updated_at`. Migration 002 already created the table. Migration 008 only adds a constraint and index.

### Project Structure Notes

New files (absolute paths from repo root):
- `eusolicit-app/services/client-api/src/client_api/schemas/espd.py`
- `eusolicit-app/services/client-api/src/client_api/services/espd_service.py`
- `eusolicit-app/services/client-api/src/client_api/api/v1/espd.py`
- `eusolicit-app/services/client-api/alembic/versions/008_espd_profile_unique_version.py`
- `eusolicit-app/services/client-api/tests/api/test_espd_profile.py`

Modified files:
- `eusolicit-app/services/client-api/src/client_api/main.py` (add espd_v1 router)
- `eusolicit-app/services/client-api/tests/api/test_audit_trail.py` (extend with ESPD audit tests)

No changes needed:
- `eusolicit-app/services/client-api/src/client_api/models/espd_profile.py` — model is complete
- `eusolicit-app/services/client-api/alembic/versions/002_auth_identity_tables.py` — table already created

### Test Fixture Pattern

Follow `test_company_profile.py` and `test_team_members.py` fixture patterns exactly:
- Use `company_client_and_session` fixture pattern (register → verify email via SQL UPDATE → login)
- Cross-company test: spin up two separate instances of the fixture (Company A and Company B)
- Role tests: use `_register_and_verify_with_role` helper from `test_company_profile.py`
- Audit assertions: query `shared.audit_log` directly in the test session:

```python
from sqlalchemy import select, text
from client_api.models import AuditLog

result = await session.execute(
    select(AuditLog)
    .where(AuditLog.entity_type == "espd_profile")
    .order_by(AuditLog.timestamp.desc())
    .limit(1)
)
row = result.scalar_one()
assert row.action_type == "create"
assert row.before is None
assert row.after is not None
assert row.user_id == expected_user_id
```

Full ESPD payload for tests (all 4 required sections):
```python
ESPD_FULL_PAYLOAD = {
    "field_values": {
        "exclusion_grounds": {"corruption": False, "fraud": False, "tax_obligations": True},
        "economic_standing": {"annual_turnover": 5000000, "currency": "EUR"},
        "technical_ability": {"staff_count": 42, "certifications": ["ISO 9001"]},
        "quality_assurance": {"iso_certified": True, "last_audit": "2025-01-15"},
    }
}
```

### Test Expectations from Epic-Level Test Design

The following tests from `test-design-epic-02.md` are assigned to this story and MUST be implemented:

| Test ID | Level | Scenario | Expected Result |
|---------|-------|----------|-----------------|
| **E02-P0-012** (partial) | API | Company A token → GET/PUT/PATCH Company B's `/espd-profile` and `/espd-profile/versions` | All return 403 (cross-tenant isolation) |
| **E02-P1-018** (ESPD extension) | API | PUT `/espd-profile` → verify `audit_log` row with `entity_type="espd_profile"`, `action_type="create"`, `before=None`, `after` non-null, `user_id` set, `ip_address` non-null | First write produces create audit; second write produces update audit with before/after snapshots |
| **E02-P2-007** | API | PUT `/espd-profile` twice | Second version = first version + 1 (version = 2) |
| **E02-P2-008** | API | PATCH `/espd-profile` with only `exclusion_grounds` | `exclusion_grounds` updated; other 3 sections unchanged from prior PUT |
| **E02-P2-009** | API | 3 successive PUTs → GET `/espd-profile/versions` | Returns 3 entries with ascending version numbers (1, 2, 3); `total = 3` |
| **E02-P2-010** | API | PUT with missing required top-level ESPD sections | Returns 422 for each missing required key (`exclusion_grounds`, `economic_standing`, `technical_ability`, `quality_assurance`) |

**Note on E02-P0-012 scope for this story:** The full cross-tenant sweep is split across Stories 2.8, 2.9, 2.10, and 2.12. This story must add the ESPD-specific portion: Company A cannot access Company B's ESPD endpoints. The tests for company profile and membership cross-tenant isolation were already added in Stories 2.8 and 2.9 respectively. Add a dedicated test class `TestCrossTenantEspd` in `test_espd_profile.py`.

**Note on E02-P1-018 scope extension:** Story 2.11's dev notes explicitly state "ESPD profile mutations (Story 2.12) will add their own audit assertions. The test file `test_audit_trail.py` should be designed so S02.12 can extend it." Add new test methods to the existing `TestAuditTrail` class (or a new `TestAuditTrailEspd` class) in `test_audit_trail.py`. Do NOT create a new test file for these assertions.

### References

- Story S02.12 requirements: [Source: eusolicit-docs/planning-artifacts/epics/E02-authentication-identity.md#S02.12]
- ESPDProfile ORM model: [Source: eusolicit-app/services/client-api/src/client_api/models/espd_profile.py]
- `espd_profiles` table (migration 002): [Source: eusolicit-app/services/client-api/alembic/versions/002_auth_identity_tables.py]
- Company service patterns (snapshot, audit, cross-tenant guard): [Source: eusolicit-app/services/client-api/src/client_api/services/company_service.py]
- Companies API route patterns (PUT/PATCH, require_role, http_request): [Source: eusolicit-app/services/client-api/src/client_api/api/v1/companies.py]
- `write_audit_entry` canonical audit helper: [Source: eusolicit-app/services/client-api/src/client_api/services/audit_service.py]
- `require_role` and `get_current_user` dependencies: [Source: eusolicit-app/services/client-api/src/client_api/core/security.py]
- `get_db_session` and `get_session_factory` dependency providers: [Source: eusolicit-app/services/client-api/src/client_api/dependencies.py]
- Test fixture patterns (`company_client_and_session`, `_register_and_verify_with_role`): [Source: eusolicit-app/services/client-api/tests/api/test_company_profile.py]
- Audit assertion patterns for `shared.audit_log`: [Source: eusolicit-app/services/client-api/tests/api/test_audit_trail.py]
- Test conftest (RSA keys, Redis, app, DB session factory): [Source: eusolicit-app/services/client-api/tests/conftest.py]
- Deferred-work items resolved in migration 008 (unique constraint + FK index): [Source: eusolicit-docs/implementation-artifacts/deferred-work.md#Deferred from: code review of story 2-1]
- Previous story patterns (ip_address injection, audit pattern): [Source: eusolicit-docs/implementation-artifacts/2-11-audit-trail-middleware.md#Dev Notes]
- Epic-level test design risks (E02-R-002 cross-tenant, E02-R-007 audit completeness, E02-R-008 ESPD JSONB gaps): [Source: eusolicit-docs/test-artifacts/test-design-epic-02.md#Risk Assessment]
- Project-wide rules (structlog, no commit, `__future__` annotations, `http_request` naming): [Source: eusolicit-docs/project-context.md#Critical Implementation Rules]

## Senior Developer Review

**Review Date:** 2026-04-07
**Verdict:** ✅ APPROVED — Clean review. Zero patch or decision-needed items.
**Review Layers:** Blind Hunter, Edge Case Hunter, Acceptance Auditor (3/3 passed)

### Review Findings

- [x] [Review][Defer] No size/depth limit on `field_values` JSONB payload — deferred, pre-existing (already tracked in deferred-work.md as "no payload size validation", LOW priority)
- [x] [Review][Defer] No-op PATCH (empty `field_values: {}`) creates phantom version rows — deferred, not a spec violation; AC3 does not prohibit no-op patches. Future optimization.
- [x] [Review][Defer] `IntegrityError` on concurrent version collision propagates raw DB details in 500 response — deferred, intentional per Dev Note 7 ("let the DB error propagate"); error message sanitization is a valid future concern
- [x] [Review][Defer] Redundant queries in upsert functions (`_get_latest_or_none` + `_get_next_version` query same table) — deferred, minor optimization; could derive version from `existing.version + 1`
- [x] [Review][Defer] Pre-existing model issues: `espd_profile.py` missing `from __future__ import annotations`, `onupdate=sa.func.now()` dead code in append-only design — deferred, from Story 2.1 (model not modified in this story)
- [x] [Review][Defer] No test assertion for AC7 `entity_id` matching new profile row's UUID — deferred, implementation correctly passes `entity_id=new_profile.id`; minor test coverage gap

### Review Stats

| Metric | Count |
|--------|-------|
| Findings raised | 20 |
| Dismissed (noise/intentional) | 14 |
| Deferred (pre-existing) | 6 |
| Patch (code fix needed) | 0 |
| Decision needed | 0 |

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (claude-code)

### Debug Log References

None — implementation proceeded without errors. All tests passed on first run.

### Completion Notes List

- All 7 tasks completed in full.
- Alembic migration 008 applied cleanly (`alembic upgrade head`).
- 35 ATDD tests in `test_espd_profile.py` activated (skip decorators removed) — all PASS.
- 3 ESPD audit tests in `test_audit_trail.py` activated (skip decorators removed) — all PASS.
- Full test suite: **373 passed, 2 warnings** (warnings are pre-existing RBAC unit test artefacts, unrelated to this story).

### File List

**New files created:**
- `eusolicit-app/services/client-api/src/client_api/schemas/espd.py`
- `eusolicit-app/services/client-api/src/client_api/services/espd_service.py`
- `eusolicit-app/services/client-api/src/client_api/api/v1/espd.py`
- `eusolicit-app/services/client-api/alembic/versions/008_espd_profile_unique_version.py`

**Modified files:**
- `eusolicit-app/services/client-api/src/client_api/main.py` — registered `espd_v1.router`
- `eusolicit-app/services/client-api/tests/api/test_espd_profile.py` — removed 30 `@pytest.mark.skip` decorators; updated header to GREEN PHASE
- `eusolicit-app/services/client-api/tests/api/test_audit_trail.py` — removed 3 `@pytest.mark.skip` decorators from `TestAuditTrailEspd`; updated `_SKIP_REASON_212` variable

