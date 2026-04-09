# Story 11.4: Grant Eligibility Agent Integration

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **company admin or bid manager**,
I want to trigger a Grant Eligibility check that maps my company profile against active EU funding programmes,
so that I receive a ranked list of matched programmes with eligibility scores, requirements summaries, and gap analyses to guide my EU grant application strategy.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/grants/eligibility-check` accepts an optional JSON request body with:
   - `programme_type` (optional string — one of `"horizon_europe"`, `"digital_europe"`, `"structural_funds"`, `"cap"`, `"interreg"`; any string is accepted and forwarded to the agent)
   - `funding_range_min_eur` (optional float — minimum EU funding amount in EUR)
   - `funding_range_max_eur` (optional float — maximum EU funding amount in EUR)
   Loads the authenticated company's profile from the `client.companies` table using `company_id` from the JWT. Sends the company profile and optional filters to the Grant Eligibility Agent via the AI Gateway. Returns HTTP 200 with a `GrantEligibilityResponse`. An empty request body `{}` is accepted (all filter fields default to `null`). Any authenticated company user may call this endpoint — no minimum role restriction beyond a valid JWT.

2. **AC2** — The AI Gateway agent payload sent to logical agent name `grant-eligibility` (URL: `{AIGW_BASE_URL}/agents/grant-eligibility/run`) must be structured as:
   ```json
   {
     "company_profile": {
       "company_id": "<company UUID as string>",
       "name": "<company name>",
       "industry": "<industry or null>",
       "cpv_sectors": ["<cpv code>", ...],
       "regions": ["<region>", ...],
       "country": "<from address.country if present, else null>",
       "description": "<description or null>",
       "certifications": [{"name": "...", "issuer": "...", "valid_until": "..."}, ...]
     },
     "filters": {
       "programme_type": "<value or null>",
       "funding_range_min_eur": <float or null>,
       "funding_range_max_eur": <float or null>
     }
   }
   ```
   The request must include the header `X-Caller-Service: client-api`. The timeout is 30 seconds (configurable via `AIGW_TIMEOUT_SECONDS`). The client-api does NOT retry — the AI Gateway (E04) handles retry and circuit-breaker logic internally.

3. **AC3** — The agent response is parsed into a `GrantEligibilityResponse` containing:
   - `matched_programmes` (list of `MatchedProgramme`, may be empty): each item contains:
     - `programme_name` (str, required)
     - `call_reference` (str or null)
     - `eligibility_score` (int, 0–100, required)
     - `requirements_summary` (str, required)
     - `gap_analysis` (str or null)
   - `total_matched` (int): count of items in `matched_programmes`
   - `company_id` (UUID): the company UUID echoed from the JWT
   - `checked_at` (datetime ISO 8601): server timestamp at which the check was performed
   The result is returned directly (stateless — nothing is persisted to the database on success).

4. **AC4** — AI Gateway error handling: if the gateway times out (> 30 s) or returns HTTP 5xx, the endpoint returns HTTP 503 with the standardised error body:
   ```json
   {"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"}
   ```
   No data is persisted on agent failure. The raw agent response is never forwarded to the caller.

5. **AC5** — If the company record for `current_user.company_id` does not exist in the database, return HTTP 404 with `{"detail": "Company not found"}`. (Edge case — should not occur in normal operation because company_id is embedded in a valid JWT.)

6. **AC6** — Unauthenticated requests (missing or expired token) return HTTP 401. All authenticated users (read_only, contributor, reviewer, bid_manager, admin) may call this endpoint without restriction.

7. **AC7** — The endpoint does NOT apply any client-side filtering or re-ranking of the agent's response. Filter params (`programme_type`, `funding_range_min_eur`, `funding_range_max_eur`) are forwarded verbatim in the `filters` object to the agent. If a filter field is absent from the request body, its value in the payload is `null`. The agent is responsible for applying filter logic against EU programme data (Horizon Europe, Digital Europe, structural funds, CAP, Interreg).

## Tasks / Subtasks

- [x] Task 1 — Create `src/client_api/schemas/grants.py` (AC: 1, 3)
  - [x] 1.1 Add `from __future__ import annotations` header and all imports: `datetime`, `UUID`, `BaseModel`, `ConfigDict`, `Field`
  - [x] 1.2 Define `GrantEligibilityRequest(BaseModel)` with all optional fields (defaults `None`):
    - `programme_type: str | None = None`
    - `funding_range_min_eur: float | None = None`
    - `funding_range_max_eur: float | None = None`
    - Add `model_config = ConfigDict(extra="ignore")` to silently discard unknown fields
  - [x] 1.3 Define `MatchedProgramme(BaseModel)` with:
    - `programme_name: str`
    - `call_reference: str | None = None`
    - `eligibility_score: int = Field(..., ge=0, le=100)`
    - `requirements_summary: str`
    - `gap_analysis: str | None = None`
  - [x] 1.4 Define `GrantEligibilityResponse(BaseModel)` with:
    - `matched_programmes: list[MatchedProgramme]`
    - `total_matched: int`
    - `company_id: UUID`
    - `checked_at: datetime`

- [x] Task 2 — Create `src/client_api/services/grants_service.py` (AC: 1–7)
  - [x] 2.1 Add `from __future__ import annotations` header, structlog logger (`log = structlog.get_logger()`), and imports: `UUID`, `datetime`, `timezone`, `AsyncSession`, `select`, `HTTPException`, `Company`, `CurrentUser`, `AiGatewayClient`, `AiGatewayTimeoutError`, `AiGatewayUnavailableError`, `GrantEligibilityRequest`, `GrantEligibilityResponse`, `MatchedProgramme`
  - [x] 2.2 Define `async def _load_company(company_id: UUID, session: AsyncSession) -> Company`:
    - Execute `SELECT * FROM client.companies WHERE id = :company_id`
    - Raise `HTTPException(status_code=404, detail="Company not found")` if result is `None`
    - Return the `Company` ORM instance
    - Note: no cross-company RLS check needed here — `company_id` always comes from the JWT, never from user input
  - [x] 2.3 Define `def _build_company_payload(company: Company) -> dict`:
    - Extract `country` from `company.address.get("country")` if `company.address` is not `None`, else `None`
    - Return dict:
      ```python
      {
          "company_id": str(company.id),
          "name": company.name,
          "industry": company.industry,
          "cpv_sectors": company.cpv_sectors or [],
          "regions": company.regions or [],
          "country": country,
          "description": company.description,
          "certifications": company.certifications or [],
      }
      ```
  - [x] 2.4 Define `def _parse_matched_programme(raw: dict) -> MatchedProgramme`:
    - Extract fields from the agent's raw programme dict using `.get()` with safe defaults
    - Return `MatchedProgramme(programme_name=raw.get("programme_name", "Unknown"), call_reference=raw.get("call_reference"), eligibility_score=min(100, max(0, int(raw.get("eligibility_score", 0)))), requirements_summary=raw.get("requirements_summary", ""), gap_analysis=raw.get("gap_analysis"))`
    - Wrap in try/except `(KeyError, ValueError, TypeError)` → log warning and return `None`; caller filters out `None` results
  - [x] 2.5 Implement `async def check_grant_eligibility(request: GrantEligibilityRequest, current_user: CurrentUser, session: AsyncSession, gw_client: AiGatewayClient) -> GrantEligibilityResponse`:
    - Load company: `company = await _load_company(current_user.company_id, session)` (AC5)
    - Build payload:
      ```python
      payload = {
          "company_profile": _build_company_payload(company),
          "filters": {
              "programme_type": request.programme_type,
              "funding_range_min_eur": request.funding_range_min_eur,
              "funding_range_max_eur": request.funding_range_max_eur,
          },
      }
      ```
    - Call agent (AC2, AC4):
      ```python
      try:
          agent_response = await gw_client.run_agent("grant-eligibility", payload)
      except AiGatewayTimeoutError:
          log.warning("grants.eligibility_check.timeout", company_id=str(current_user.company_id))
          raise HTTPException(
              status_code=503,
              detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"},
          )
      except AiGatewayUnavailableError:
          log.warning("grants.eligibility_check.unavailable", company_id=str(current_user.company_id))
          raise HTTPException(
              status_code=503,
              detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"},
          )
      ```
    - Parse response (AC3): extract `raw_programmes = agent_response.get("matched_programmes", [])` (default to `[]` if key absent); map through `_parse_matched_programme`; filter out `None` values
    - Log completion: `log.info("grants.eligibility_check.completed", company_id=str(current_user.company_id), total_matched=len(parsed), programme_type=request.programme_type)`
    - Return `GrantEligibilityResponse(matched_programmes=parsed, total_matched=len(parsed), company_id=current_user.company_id, checked_at=datetime.now(timezone.utc))`

- [x] Task 3 — Create `src/client_api/api/v1/grants.py` router (AC: 1, 6)
  - [x] 3.1 Add `from __future__ import annotations` header and imports: `Annotated`, `AsyncSession`, `Depends`, `APIRouter`, `CurrentUser`, `get_current_user`, `get_db_session`, `AiGatewayClient`, `get_ai_gateway_client`, `GrantEligibilityRequest`, `GrantEligibilityResponse`, `grants_service`
  - [x] 3.2 Create `router = APIRouter(prefix="/grants", tags=["grants"])`
  - [x] 3.3 Implement `POST /eligibility-check` endpoint:
    ```python
    @router.post("/eligibility-check", response_model=GrantEligibilityResponse)
    async def eligibility_check(
        body: GrantEligibilityRequest,
        session: Annotated[AsyncSession, Depends(get_db_session)],
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
    ) -> GrantEligibilityResponse | JSONResponse:
        """..."""
        try:
            return await grants_service.check_grant_eligibility(
                body, current_user, session, gw_client
            )
        except HTTPException as exc:
            if exc.status_code == 503 and isinstance(exc.detail, dict):
                return JSONResponse(status_code=503, content=exc.detail)
            raise
    ```
  - [x] 3.4 Note: use `Depends(get_current_user)` (not `require_role`) so all authenticated roles are permitted (AC6); router catches 503 HTTPException and returns flat JSON body matching AC4 spec

- [x] Task 4 — Register the grants router in `src/client_api/main.py` (AC: 1)
  - [x] 4.1 Add import: `from client_api.api.v1 import grants as grants_v1` (after the existing espd import line)
  - [x] 4.2 Add `api_v1_router.include_router(grants_v1.router)` immediately after `api_v1_router.include_router(espd_v1.router)`

- [x] Task 5 — Write tests in `tests/api/test_grant_eligibility.py` (AC: 1–7; E11-P0-004, E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-001)
  - [x] 5.1 Add module docstring listing epic test IDs covered and link to story file
  - [x] 5.2 Add `respx = pytest.importorskip("respx", reason="...")` guard (same pattern as `test_espd_autofill_export.py`)
  - [x] 5.3 Define module-level constants:
    ```python
    AIGW_TEST_BASE_URL = "http://test-aigw.local:8000"
    AIGW_ELIGIBILITY_URL = f"{AIGW_TEST_BASE_URL}/agents/grant-eligibility/run"
    ENDPOINT = "/api/v1/grants/eligibility-check"
    AGENT_UNAVAILABLE_CODE = "AGENT_UNAVAILABLE"
    AGENT_UNAVAILABLE_MESSAGE = "AI features are temporarily unavailable. Please try again."
    ```
  - [x] 5.4 Define deterministic fixture responses:
    ```python
    MOCK_FULL_MATCH_RESPONSE = {
        "matched_programmes": [
            {
                "programme_name": "Horizon Europe",
                "call_reference": "HORIZON-EIC-2026-PATHFINDERCHALLENGES",
                "eligibility_score": 92,
                "requirements_summary": "Open to SMEs and research institutions in EU member states.",
                "gap_analysis": None,
            },
            {
                "programme_name": "Digital Europe Programme",
                "call_reference": "DIGITAL-2026-CLOUD-AI",
                "eligibility_score": 85,
                "requirements_summary": "AI and cloud infrastructure deployment projects.",
                "gap_analysis": "Company lacks GDPR certification.",
            },
        ]
    }
    MOCK_PARTIAL_MATCH_RESPONSE = {
        "matched_programmes": [
            {
                "programme_name": "Interreg Central Europe",
                "call_reference": None,
                "eligibility_score": 61,
                "requirements_summary": "Cross-border cooperation projects.",
                "gap_analysis": "Company lacks partner organisations in at least 3 member states.",
            }
        ]
    }
    MOCK_NO_MATCH_RESPONSE = {"matched_programmes": []}
    ```
  - [x] 5.5 Add session-scoped `aigw_env_setup` fixture (same pattern as `test_espd_autofill_export.py`): set `CLIENT_API_AIGW_BASE_URL = AIGW_TEST_BASE_URL`, clear `get_settings` and `get_ai_gateway_client` LRU caches, restore on teardown
  - [x] 5.6 Add `pytest_asyncio.fixture` `eligibility_client_and_session` (function scope):
    - Create and start an `httpx.AsyncClient(app=app, base_url="http://test")` (import `app` from `client_api.main`)
    - Register a new company user via `POST /api/v1/auth/register` with unique email/company name (use `uuid.uuid4().hex[:8]` suffix)
    - Verify email via SQL: `UPDATE client.users SET email_verified = TRUE WHERE id = :id`
    - Login via `POST /api/v1/auth/login` → extract `access_token`
    - Yield `(client, session, access_token, company_id_str)`
    - Teardown: close client
  - [x] 5.7 **`TestAC1AC2HappyPath`** — Full-match success path (E11-P0-004, E11-P0-009):
    - `test_eligibility_check_returns_200_with_response_structure` — mock `AIGW_ELIGIBILITY_URL` with `MOCK_FULL_MATCH_RESPONSE` using `respx.MockRouter`; POST `{}` → assert 200; response body contains `matched_programmes`, `total_matched == 2`, `company_id`, `checked_at`
    - `test_eligibility_check_agent_payload_has_company_profile_and_filters` — intercept agent call with `respx.MockRouter`; inspect `request.content` (parsed JSON); assert `company_profile.company_id` is present as string; assert `filters.programme_type` is `null` when not provided
    - `test_eligibility_check_with_programme_type_filter` — POST `{"programme_type": "horizon_europe"}`; assert captured payload has `filters.programme_type == "horizon_europe"`
    - `test_eligibility_check_with_funding_range_filter` — POST `{"funding_range_min_eur": 100000.0, "funding_range_max_eur": 5000000.0}`; assert captured payload has `filters.funding_range_min_eur == 100000.0` and `filters.funding_range_max_eur == 5000000.0`
    - `test_eligibility_check_x_caller_service_header_sent` — assert captured request headers contain `x-caller-service: client-api`
  - [x] 5.8 **`TestAC3ResponseParsing`** — Match scenario variants (E11-P1-001):
    - `test_full_match_programmes_have_required_fields` — mock full-match response; POST `{}`; assert each programme in `matched_programmes` has `programme_name` (non-empty str), `eligibility_score` (int, 0–100), `requirements_summary` (non-empty str)
    - `test_partial_match_scenario` — mock `MOCK_PARTIAL_MATCH_RESPONSE`; POST `{}`; assert `total_matched == 1`; assert `matched_programmes[0].eligibility_score == 61`
    - `test_no_match_scenario` — mock `MOCK_NO_MATCH_RESPONSE`; POST `{}`; assert 200; `matched_programmes == []`; `total_matched == 0`
    - `test_agent_response_missing_matched_programmes_key_defaults_to_empty_list` — mock response `{}` (no `matched_programmes` key); POST `{}`; assert 200; `matched_programmes == []`; `total_matched == 0` (parser graceful degradation)
  - [x] 5.9 **`TestAC4AgentErrorHandling`** — Error scenarios (E11-P0-004, E11-P0-008, E11-P0-010):
    - `test_gateway_timeout_returns_503_agent_unavailable` — mock `AIGW_ELIGIBILITY_URL` to raise `httpx.ReadTimeout`; POST `{}`; assert 503; response body has `code == "AGENT_UNAVAILABLE"` and `message == AGENT_UNAVAILABLE_MESSAGE`
    - `test_gateway_500_returns_503_not_500` — mock `AIGW_ELIGIBILITY_URL` to return HTTP 500; POST `{}`; assert 503; assert `code == "AGENT_UNAVAILABLE"`
    - `test_gateway_503_returns_503_with_standard_body` — mock `AIGW_ELIGIBILITY_URL` to return HTTP 503; POST `{}`; assert 503; assert `code == "AGENT_UNAVAILABLE"`
  - [x] 5.10 **`TestAC6Authorization`** — Auth scenarios:
    - `test_unauthenticated_returns_401` — POST without `Authorization` header; assert 401
    - `test_read_only_role_allowed` — create user with `read_only` role in same company (using `_register_and_verify_with_role` helper pattern from `test_espd_autofill_export.py`); mock full-match response; POST `{}`; assert 200 (not 403) — all roles permitted (AC6)

## Dev Notes

### Architecture Context

This story adds the first endpoint in the new `/grants` resource group. The `AiGatewayClient` (S11.03, `services/ai_gateway_client.py`) and the `ClientApiSettings.aigw_*` config fields are already implemented — reuse without modification.

### File Creation Checklist

| File | Action |
|------|--------|
| `src/client_api/schemas/grants.py` | **Create new** |
| `src/client_api/services/grants_service.py` | **Create new** |
| `src/client_api/api/v1/grants.py` | **Create new** |
| `src/client_api/main.py` | **Edit** — add 2 lines |
| `tests/api/test_grant_eligibility.py` | **Create new** |

### Key Patterns from S11.03 to Follow

1. **AI Gateway errors → HTTP 503**: catch both `AiGatewayTimeoutError` and `AiGatewayUnavailableError`; raise `HTTPException(503, detail={...})` with the `AGENT_UNAVAILABLE` body structure. Never forward raw gateway errors.

2. **Stateless endpoint**: unlike S11.03 (which creates a snapshot profile), this endpoint does NOT write anything to the database on success. `session` is injected (for the `_load_company` query) but no `session.flush()` or `session.commit()` is called in the grants service.

3. **Payload serialisation**: UUIDs must be serialised as strings (`str(company.id)`) before including in the agent payload dict — `httpx` JSON serialisation does not handle `uuid.UUID` objects.

4. **Parser defensive defaults**: use `.get()` with safe defaults throughout `_parse_matched_programme`. If the agent returns a programme dict with a missing `requirements_summary`, default to `""` rather than raising. Log a warning for unexpected shapes; never raise unhandled exceptions from parser code.

5. **Empty body accepted**: all fields in `GrantEligibilityRequest` default to `None`. FastAPI will accept `Content-Type: application/json` with body `{}` and use defaults. In tests, POST with `json={}` to verify the no-filter path.

### Company Data Mapping Note

The Grant Eligibility Agent expects company attributes including `size`, `legal_form`, and `turnover`. The current `client.companies` table (S2.08) does not have dedicated columns for these. The `_build_company_payload` function should include all available Company ORM fields. If `size`, `legal_form`, or `turnover` are added to the Company model in a future story, the payload builder should be extended accordingly. For now, the agent must work with the available fields (`industry`, `cpv_sectors`, `regions`, `country`, `description`, `certifications`).

### Router Registration in `main.py`

Add the following two lines (the import and router registration) adjacent to the existing espd router lines:

```python
# existing lines (for context):
from client_api.api.v1 import espd as espd_v1
from client_api.api.v1 import grants as grants_v1   # ADD THIS LINE

# ...

api_v1_router.include_router(espd_v1.router)
api_v1_router.include_router(grants_v1.router)  # ADD THIS LINE
```

### Test Fixture Pattern

The `eligibility_client_and_session` fixture mirrors `espd_client_and_session` from `test_espd_profile.py`:
- Import `app` from `client_api.main`
- Use `httpx.AsyncClient(app=app, base_url="http://test")` (ASGI transport)
- Create a unique company per test function (function scope) to avoid cross-test pollution

The `aigw_env_setup` fixture is session-scoped and sets `CLIENT_API_AIGW_BASE_URL` to `http://test-aigw.local:8000` — this ensures `respx` can intercept calls at a predictable URL. Clear `get_settings.cache_clear()` and `get_ai_gateway_client.cache_clear()` after setting the env var.

### Epic Test ID Coverage

| Epic Test ID | Description | Test Method | Priority |
|-------------|-------------|-------------|----------|
| **E11-P0-004** | Grant Eligibility: structured 503 on timeout; structured list on success | `test_eligibility_check_returns_200_with_response_structure`, `test_gateway_timeout_returns_503_agent_unavailable` | P0 |
| **E11-P0-008** | grant-eligibility endpoint returns `AGENT_UNAVAILABLE` on gateway 5xx | `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` | P0 |
| **E11-P0-009** | grant-eligibility agent mock returns deterministic fixture in CI | `test_eligibility_check_returns_200_with_response_structure` | P0 |
| **E11-P0-010** | 30s timeout enforced; request does not hang past timeout | `test_gateway_timeout_returns_503_agent_unavailable` (via ReadTimeout mock) | P0 |
| **E11-P1-001** | Full-match, partial-match, no-match scenarios; filter params applied | `test_full_match_programmes_have_required_fields`, `test_partial_match_scenario`, `test_no_match_scenario`, `test_eligibility_check_with_programme_type_filter`, `test_eligibility_check_with_funding_range_filter` | P1 |

### `GrantEligibilityResponse` JSON Shape Example

```json
{
  "matched_programmes": [
    {
      "programme_name": "Horizon Europe",
      "call_reference": "HORIZON-EIC-2026-PATHFINDERCHALLENGES",
      "eligibility_score": 92,
      "requirements_summary": "Open to SMEs and research institutions in EU member states.",
      "gap_analysis": null
    },
    {
      "programme_name": "Digital Europe Programme",
      "call_reference": "DIGITAL-2026-CLOUD-AI",
      "eligibility_score": 85,
      "requirements_summary": "AI and cloud infrastructure deployment projects.",
      "gap_analysis": "Company lacks GDPR certification."
    }
  ],
  "total_matched": 2,
  "company_id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "checked_at": "2026-04-09T12:34:56.789Z"
}
```

### Related Files (Reference Only — Do Not Modify)

- `services/client-api/src/client_api/services/ai_gateway_client.py` — `AiGatewayClient`, `get_ai_gateway_client`, `AiGatewayTimeoutError`, `AiGatewayUnavailableError` (S11.03)
- `services/client-api/src/client_api/config.py` — `ClientApiSettings.aigw_base_url`, `aigw_timeout_seconds` (S11.03)
- `services/client-api/src/client_api/core/security.py` — `CurrentUser`, `get_current_user`, `require_role`
- `services/client-api/src/client_api/models/company.py` — `Company` ORM model (available fields)
- `services/client-api/tests/api/test_espd_autofill_export.py` — reference for `aigw_env_setup` fixture and `respx` mock patterns

## Dev Agent Record

### Implementation Plan

Implemented Story 11.4 Grant Eligibility Agent Integration as a stateless `POST /api/v1/grants/eligibility-check` endpoint. Key design decisions:

1. **Schemas** (`schemas/grants.py`): Three Pydantic models — `GrantEligibilityRequest` (all-optional filters with `extra="ignore"`), `MatchedProgramme` (with `eligibility_score` validated 0–100 via `Field`), and `GrantEligibilityResponse` (stateless response).

2. **Service** (`services/grants_service.py`): Four helper functions follow story spec exactly. `_build_company_payload` serialises UUID to string (Dev Notes §3). `_parse_matched_programme` uses defensive `.get()` with safe defaults and returns `None` on errors (Dev Notes §4). `check_grant_eligibility` is stateless — no DB writes on success.

3. **Router** (`api/v1/grants.py`): Uses `Depends(get_current_user)` (not `require_role`) for all-role access (AC6). The router wraps the service call in a try/except: when the service raises `HTTPException(503, detail=dict)`, the router converts it to a `JSONResponse(503, content=exc.detail)` so the response body is the flat `{"message": ..., "code": ...}` format required by AC4 (not `{"detail": {...}}`). This is the correct HTTP transformation layer.

4. **main.py**: Added 2 lines — import and `include_router`.

5. **Tests**: Pre-written ATDD tests already existed in GREEN PHASE (15 skip decorators removed). All 15 tests pass.

### Debug Log

- AC4 response format: Tests expected `{"code": ..., "message": ...}` at top level. Service raises `HTTPException(503, detail={...})` which FastAPI would wrap as `{"detail": {...}}`. Resolution: router catches the 503 `HTTPException` and returns `JSONResponse(503, content=exc.detail)` — this is distinct from the ESPD auto-fill pattern (which uses `{"detail": {...}}` wrapping) by deliberate design per AC4 spec.

### Completion Notes

- ✅ All 16 tests pass (15 original ATDD + 1 new F1 regression test — 16 passed in 7.85s)
- ✅ Full regression suite: 279 API tests passing (3 pre-existing failures in `test_audit_trail.py::TestAuditTrailEspd` unrelated to this story — `PUT /espd-profile` 404, confirmed failing before this story)
- ✅ All 7 Acceptance Criteria satisfied
- ✅ All 5 Tasks with subtasks marked complete
- ✅ Stateless endpoint — no DB writes on success
- ✅ Flat `{"message", "code"}` 503 body per AC4 spec
- ✅ All roles permitted via `get_current_user` (no `require_role`) per AC6

**Review follow-up actions completed (2026-04-09):**
- ✅ Resolved review finding [Medium]: F1 — Added type guard `if not isinstance(raw, dict)` to `_parse_matched_programme` (Option B — explicit guard, preferred per review); prevents `AttributeError` on non-dict agent responses; added regression test `test_non_dict_items_in_matched_programmes_filtered_out` that verifies mixed list `["bad string", null, {valid dict}]` → HTTP 200, `total_matched == 1`
- ✅ Resolved review finding [Low]: F3 — Updated 5 stale class-level docstrings from `🔴 RED` to `🟢 GREEN` in `test_grant_eligibility.py` (classes: `TestAC1AC2HappyPath`, `TestAC3ResponseParsing`, `TestAC4AgentErrorHandling`, `TestAC5CompanyNotFound`, `TestAC6Authorization`)
- ℹ️ F2 (duplicate 503 blocks) — deferred; non-blocking per review
- ℹ️ F4 (advisory — covered by new F1 test), F5 (advisory — ESPD vs Grants 503 format) — deferred as non-blocking

## File List

New files created:
- `eusolicit-app/services/client-api/src/client_api/schemas/grants.py`
- `eusolicit-app/services/client-api/src/client_api/services/grants_service.py`
- `eusolicit-app/services/client-api/src/client_api/api/v1/grants.py`

Modified files:
- `eusolicit-app/services/client-api/src/client_api/main.py` — added grants import and router registration
- `eusolicit-app/services/client-api/src/client_api/services/grants_service.py` — F1 fix: added type guard to `_parse_matched_programme`; updated function signature to `raw: object`
- `eusolicit-app/services/client-api/tests/api/test_grant_eligibility.py` — removed 15 `@pytest.mark.skip` decorators; updated module docstring to GREEN PHASE; added `test_non_dict_items_in_matched_programmes_filtered_out`; updated 5 class docstrings from RED to GREEN
- `eusolicit-docs/implementation-artifacts/11-4-grant-eligibility-agent-integration.md` — task checkboxes, dev agent record, file list, change log, status
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — status updated

## Senior Developer Review

**Verdict: APPROVED**
**Reviewer: Senior Dev Code Review (adversarial, re-review)**
**Date: 2026-04-09**

### Prior Review Status

First review (2026-04-09) requested changes for 2 findings:
- ✅ **F1 (Medium)** — `_parse_matched_programme` non-dict crash: **FIXED** — type guard added (`isinstance(raw, dict)`), regression test added (`test_non_dict_items_in_matched_programmes_filtered_out`)
- ✅ **F3 (Low)** — Stale RED PHASE docstrings: **FIXED** — 5 class docstrings updated to GREEN
- ℹ️ F2, F4, F5 — deferred (non-blocking), confirmed still non-blocking

### Re-Review Summary

All prior required fixes verified. Implementation is clean, all 16 tests pass (7.81s), all 7 Acceptance Criteria satisfied. Three parallel adversarial review layers (Blind Hunter, Edge Case Hunter, Acceptance Auditor) executed. Three Low-severity advisory findings identified — none blocking.

### Re-Review Findings

#### ℹ️ R1 — Advisory: Missing `from exc` exception chaining (Severity: Low)

**File:** `src/client_api/services/grants_service.py`, lines 141 and 153
**Detail:** Both `raise HTTPException(status_code=503, ...)` statements do not chain the original exception with `from exc`. The established pattern in `espd_service.py:242` uses `from exc`. Missing chain reduces debuggability in production logs (traceback won't show the original `AiGatewayTimeoutError` / `AiGatewayUnavailableError`). No runtime impact — just a best-practice deviation.

#### ℹ️ R2 — Advisory: Non-list `matched_programmes` iterates unexpectedly (Severity: Low)

**File:** `src/client_api/services/grants_service.py`, line 162
**Detail:** If the AI agent returns `"matched_programmes": "some_string"` (a truthy non-list), `or []` won't trigger and iterating the string character-by-character will produce N calls to `_parse_matched_programme`. The type guard (F1 fix) correctly filters each non-dict character → returns `None`, but this generates N warning logs. Safer: `isinstance(agent_response.get("matched_programmes"), list)` guard. Edge case — only occurs with a misbehaving AI Gateway.

#### ℹ️ R3 — Advisory: Non-dict `agent_response` would crash (Severity: Low)

**File:** `src/client_api/services/grants_service.py`, line 162
**Detail:** If `run_agent()` returns a non-dict JSON response (e.g., a JSON array), `agent_response.get("matched_programmes")` raises `AttributeError`. The `run_agent()` type hint says `dict` and the AI Gateway should always return JSON objects, but `response.json()` can return any JSON type. Edge case — only occurs with a severely misbehaving AI Gateway.

### Triage Summary

| ID | Source | Classification | Severity | Status |
|----|--------|---------------|----------|--------|
| R1 | blind+auditor | advisory | Low | Non-blocking |
| R2 | edge | advisory | Low | Non-blocking |
| R3 | edge | advisory | Low | Non-blocking |
| — | — | dismissed | — | 3 dismissed (cosmetic type hint, pre-existing JSONB design, log granularity preference) |

### Architecture Alignment

| Aspect | Status | Notes |
|--------|--------|-------|
| Dependency injection (FastAPI Depends) | ✅ Aligned | Matches ESPD pattern |
| Service layer isolation | ✅ Aligned | Clean separation: router → service → gateway |
| Router registration in main.py | ✅ Aligned | Adjacent to ESPD lines |
| Error handling (AI Gateway → 503) | ✅ Aligned | Correct error body per AC4 |
| RLS / company isolation | ✅ Aligned | company_id from JWT, no cross-company access |
| Structured logging (structlog) | ✅ Aligned | Contextual fields present |
| Defensive response parsing | ✅ Aligned | Type guards, `.get()` defaults, None filtering |
| Stateless endpoint design | ✅ Aligned | No DB writes on success (per spec) |
| Test patterns (respx, fixtures) | ✅ Aligned | Mirrors test_espd_autofill_export.py |
| RBAC (all roles permitted) | ✅ Aligned | Correct per AC6 — uses get_current_user |

### Verification

- ✅ 16/16 tests passing (7.81s)
- ✅ All 7 ACs satisfied (AC1–AC7)
- ✅ All 5 Epic test IDs covered (E11-P0-004, E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-001)
- ✅ Prior review F1 fix verified — type guard + regression test present and passing
- ✅ Prior review F3 fix verified — all 5 class docstrings updated to GREEN
- ✅ No HIGH or MEDIUM findings in re-review

### Action Required

None — all findings are advisory (Low severity, non-blocking). Recommended for future improvement:
1. R1: Add `from exc` to exception re-raises (align with ESPD pattern)
2. R2–R3: Add `isinstance` guards for non-list/non-dict agent responses (harden edge cases)

## Change Log

- 2026-04-09: Story 11.4 implemented — added `POST /api/v1/grants/eligibility-check` endpoint with Grant Eligibility Agent integration; 15 ATDD tests passing; stateless (no DB writes on success); flat 503 response body per AC4 spec
- 2026-04-09: Senior Developer Review — CHANGES REQUESTED (F1: `_parse_matched_programme` AttributeError bug; F3: stale test docstrings)
- 2026-04-09: Review follow-ups addressed — F1 fixed (type guard + regression test added); F3 fixed (5 class docstrings updated RED→GREEN); 16 tests passing; status → review
- 2026-04-09: Senior Developer Re-Review — APPROVED. All prior fixes verified. 3 advisory findings (R1–R3, all Low) noted as non-blocking. 16/16 tests pass. Status → done
