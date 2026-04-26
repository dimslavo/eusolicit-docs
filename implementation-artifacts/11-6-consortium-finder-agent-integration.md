# Story 11.6: Consortium Finder Agent Integration

Status: done  <!-- reconciled 2026-04-25 retro — sprint-status.yaml authoritative -->

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **company admin or bid manager**,
I want to search for potential consortium partners by describing my project and specifying required capabilities and target countries,
so that I receive a ranked list of organisations with relevant expertise and past EU project participation history that I can use to build a competitive grant consortium.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/grants/consortium-finder` accepts a JSON request body with:
   - `project_description` (str, required, `min_length=1` — summary of the proposed project)
   - `required_capabilities` (list[str], required, min 1 item — e.g. `["machine learning", "clinical trials", "IoT sensors"]`)
   - `target_countries` (list[str], optional, default `null` — ISO 3166-1 alpha-2 country codes e.g. `["BG", "DE", "FR"]`; if null the agent searches across all countries)
   - `max_results` (int, optional, default `10`, min `1`, max `50` — maximum number of partner suggestions to return)
   Returns HTTP 200 with a `ConsortiumFinderResponse`. Any authenticated user may call this endpoint — no minimum role restriction beyond a valid JWT with `company_id`.

2. **AC2** — The AI Gateway agent payload sent to logical agent name `consortium-finder` (URL: `{AIGW_BASE_URL}/agents/consortium-finder/run`) must be structured as:
   ```json
   {
     "project_description": "<string>",
     "required_capabilities": ["<capability>", ...],
     "target_countries": ["<country_code>", ...] or null,
     "max_results": <int>,
     "company_id": "<company UUID as string>"
   }
   ```
   The request must include the header `X-Caller-Service: client-api`. The timeout is 30 seconds (configurable via `AIGW_TIMEOUT_SECONDS`). The client-api does NOT retry — the AI Gateway (E04) handles retry and circuit-breaker logic internally.

3. **AC3** — The agent response is parsed into a `ConsortiumFinderResponse` containing:
   - `partners` (list of `PartnerSuggestion`, may be empty): each item has:
     - `organisation_name` (str, required)
     - `country` (str or null — ISO 3166-1 alpha-2; null if absent from agent response)
     - `organisation_type` (str or null — e.g. `"SME"`, `"University"`, `"Research Institute"`, `"NGO"`, `"Large Enterprise"`; null if absent)
     - `relevant_capabilities` (list[str], may be empty — matched capabilities from `required_capabilities`; empty list if absent)
     - `past_projects` (list of `PastProject`, may be empty): each item has `project_name` (str) and `programme` (str); empty list if absent from agent response
     - `collaboration_score` (float, 0.0–100.0 — ranking score for fit to the project)
     - `contact_info` (str or null — free-form contact information; null if absent from agent response)
   - `total_results` (int): count of items in `partners`
   - `page` (int, value `1`): current page indicator
   - `page_size` (int): number of results returned (equals `len(partners)`)

4. **AC4** — Pagination contract: the endpoint returns all results from the agent in a single response. `page` is always `1` and `page_size` equals `len(partners)`. Full server-side pagination against a persistent store is deferred to a future story once the Consortium Partners Store has a local database cache.

5. **AC5** — Result ordering: the `partners` list in the response MUST be in descending order of `collaboration_score` (highest first). The parser sorts by `collaboration_score` after parsing — the agent's order is not assumed to be ranked.

6. **AC6** — Partial agent response — graceful field degradation: if any optional field (`country`, `organisation_type`, `contact_info`, `past_projects`, `relevant_capabilities`) is absent from a partner dict in the agent response, the parser MUST return `null` (scalar fields) or `[]` (list fields) rather than raising an error or dropping the partner from the result. See E11-R-012.

7. **AC7** — AI Gateway error handling: if the AI Gateway times out (> 30 s) or returns HTTP 5xx, the endpoint returns HTTP 503 to the caller with:
   ```json
   {"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"}
   ```
   No data is persisted on agent failure. The raw agent response is never forwarded to the caller.

8. **AC8** — Unauthenticated requests (missing or expired JWT) return HTTP 401. All authenticated roles (`read_only`, `contributor`, `reviewer`, `bid_manager`, `admin`) may call this endpoint without restriction.

9. **AC9** — The endpoint is stateless — no data is written to the database. The `company_id` from the JWT is included in the agent payload for agent-side context but no results are persisted by this endpoint.

## Tasks / Subtasks

- [x] Task 1: Add Consortium Finder schemas to `src/client_api/schemas/grants.py` (AC: 1, 3, 4, 5, 6)
  - [x] 1.1 Append the story banner comment `# ---------------------------------------------------------------------------` + `# Story 11.6 — Consortium Finder Agent Integration` + `# ---------------------------------------------------------------------------` after the existing S11.5 schemas
  - [x] 1.2 Define `PastProject(BaseModel)`:
    - `project_name: str`
    - `programme: str`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.3 Define `PartnerSuggestion(BaseModel)`:
    - `organisation_name: str`
    - `country: str | None = None`
    - `organisation_type: str | None = None`
    - `relevant_capabilities: list[str] = Field(default_factory=list)`
    - `past_projects: list[PastProject] = Field(default_factory=list)`
    - `collaboration_score: float = Field(..., ge=0.0, le=100.0)`
    - `contact_info: str | None = None`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.4 Define `ConsortiumFinderRequest(BaseModel)`:
    - `project_description: str = Field(..., min_length=1)`
    - `required_capabilities: list[str] = Field(..., min_length=1)`
    - `target_countries: list[str] | None = None`
    - `max_results: int = Field(default=10, ge=1, le=50)`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.5 Define `ConsortiumFinderResponse(BaseModel)`:
    - `partners: list[PartnerSuggestion]`
    - `total_results: int`
    - `page: int = 1`
    - `page_size: int`

- [x] Task 2: Implement consortium finder service in `src/client_api/services/grants_service.py` (AC: 1–9)
  - [x] 2.1 Add imports to the existing `grants_service.py`: `ConsortiumFinderRequest`, `ConsortiumFinderResponse`, `PartnerSuggestion`, `PastProject` from `client_api.schemas.grants`
  - [x] 2.2 Define `_parse_past_project(raw: dict) -> PastProject | None`:
    - Extract `project_name = raw.get("project_name", "")` and `programme = raw.get("programme", "")`
    - Return `PastProject(project_name=project_name, programme=programme)`
    - Wrap in `try/except (KeyError, ValueError, TypeError)`: log warning via `log.warning("grants.consortium_finder.parse_past_project.error", raw=raw)`; return `None`
  - [x] 2.3 Define `_parse_partner_suggestion(raw: dict) -> PartnerSuggestion | None`:
    - Extract all fields using `.get()` with safe defaults (AC6):
      - `organisation_name = raw.get("organisation_name", "Unknown")`
      - `country = raw.get("country")` (None if absent)
      - `organisation_type = raw.get("organisation_type")` (None if absent)
      - `relevant_capabilities = raw.get("relevant_capabilities") or []` (empty list if absent or null)
      - `raw_past = raw.get("past_projects") or []`; map through `_parse_past_project`; filter `None`
      - `collaboration_score = min(100.0, max(0.0, float(raw.get("collaboration_score", 0.0))))`
      - `contact_info = raw.get("contact_info")` (None if absent)
    - Return `PartnerSuggestion(organisation_name=organisation_name, country=country, organisation_type=organisation_type, relevant_capabilities=relevant_capabilities, past_projects=parsed_past_projects, collaboration_score=collaboration_score, contact_info=contact_info)`
    - Wrap in `try/except (KeyError, ValueError, TypeError)`: log warning via `log.warning("grants.consortium_finder.parse_partner.error", raw=raw)`; return `None`
  - [x] 2.4 Implement `async def find_consortium_partners(request: ConsortiumFinderRequest, current_user: CurrentUser, gw_client: AiGatewayClient) -> ConsortiumFinderResponse`:
    - Build payload (AC2):
      ```python
      payload = {
          "project_description": request.project_description,
          "required_capabilities": request.required_capabilities,
          "target_countries": request.target_countries,
          "max_results": request.max_results,
          "company_id": str(current_user.company_id),
      }
      ```
    - Call agent (AC7):
      ```python
      try:
          agent_response = await gw_client.run_agent("consortium-finder", payload)
      except AiGatewayTimeoutError:
          log.warning("grants.consortium_finder.timeout", company_id=str(current_user.company_id))
          raise HTTPException(
              status_code=503,
              detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"},
          )
      except AiGatewayUnavailableError:
          log.warning("grants.consortium_finder.unavailable", company_id=str(current_user.company_id))
          raise HTTPException(
              status_code=503,
              detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"},
          )
      ```
    - Parse partners (AC3, AC6): `raw_partners = agent_response.get("partners", [])` (default `[]` if key absent); map through `_parse_partner_suggestion`; filter out `None` values
    - Sort descending by collaboration_score (AC5): `parsed.sort(key=lambda p: p.collaboration_score, reverse=True)`
    - Log success: `log.info("grants.consortium_finder.completed", company_id=str(current_user.company_id), total_results=len(parsed), target_countries=request.target_countries, max_results=request.max_results)`
    - Return `ConsortiumFinderResponse(partners=parsed, total_results=len(parsed), page=1, page_size=len(parsed))`

- [x] Task 3: Add `POST /grants/consortium-finder` endpoint to `src/client_api/api/v1/grants.py` (AC: 1, 7, 8)
  - [x] 3.1 Add imports to the existing `grants.py`: `ConsortiumFinderRequest`, `ConsortiumFinderResponse` from `client_api.schemas.grants`
  - [x] 3.2 Implement `POST /consortium-finder` endpoint in the existing `router`:
    ```python
    @router.post("/consortium-finder", response_model=ConsortiumFinderResponse)
    async def consortium_finder(
        body: ConsortiumFinderRequest,
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
    ) -> ConsortiumFinderResponse | JSONResponse:
        """Search for consortium partners via Consortium Finder Agent.

        Stateless — results are not persisted (AC9). Requires authentication (all roles, AC8).

        AC7: AI Gateway errors (timeout or 5xx) return 503 with flat body:
          {"message": "...", "code": "AGENT_UNAVAILABLE"}
        """
        try:
            return await grants_service.find_consortium_partners(body, current_user, gw_client)
        except HTTPException as exc:
            if exc.status_code == 503 and isinstance(exc.detail, dict):
                return JSONResponse(status_code=503, content=exc.detail)
            raise
    ```
  - [x] 3.3 Note: no `AsyncSession` dependency needed — `find_consortium_partners` does not query the DB (stateless, AC9). `get_current_user` provides `company_id` for the agent payload.
  - [x] 3.4 Note: use `Depends(get_current_user)` (not `require_role`) so all authenticated roles are permitted (AC8).
  - [x] 3.5 Note: **no changes required to `main.py`** — the `/grants` router is already registered from S11.04.

- [x] Task 4: Write tests in `tests/api/test_consortium_finder.py` (AC: 1–9; E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-003, E11-P2-002)
  - [x] 4.1 Add module docstring listing epic test IDs covered and link to this story file
  - [x] 4.2 Add `respx = pytest.importorskip("respx", reason="respx required for AI Gateway mocking")` guard
  - [x] 4.3 Define module-level constants:
    ```python
    AIGW_TEST_BASE_URL = "http://test-aigw.local:8000"
    AIGW_CONSORTIUM_URL = f"{AIGW_TEST_BASE_URL}/agents/consortium-finder/run"
    ENDPOINT = "/api/v1/grants/consortium-finder"
    AGENT_UNAVAILABLE_CODE = "AGENT_UNAVAILABLE"
    AGENT_UNAVAILABLE_MESSAGE = "AI features are temporarily unavailable. Please try again."
    ```
  - [x] 4.4 Define deterministic fixture responses:
    ```python
    # MOCK_RANKED_PARTNERS_RESPONSE: 3 partners with collaboration_scores intentionally
    # out of order (85.0, 72.0, 91.0) to verify AC5 sort (expect 91.0, 85.0, 72.0 in response).
    MOCK_RANKED_PARTNERS_RESPONSE = {
        "partners": [
            {
                "organisation_name": "Sofia Tech University",
                "country": "BG",
                "organisation_type": "University",
                "relevant_capabilities": ["machine learning", "data science"],
                "past_projects": [
                    {"project_name": "AIHealthBG", "programme": "Horizon Europe"},
                    {"project_name": "DataDrivenBG", "programme": "Digital Europe"},
                ],
                "collaboration_score": 85.0,
                "contact_info": "partnerships@sofia-tech.bg",
            },
            {
                "organisation_name": "Berlin IoT GmbH",
                "country": "DE",
                "organisation_type": "SME",
                "relevant_capabilities": ["IoT sensors", "embedded systems"],
                "past_projects": [
                    {"project_name": "SmartCityDE", "programme": "Horizon Europe"},
                ],
                "collaboration_score": 72.0,
                "contact_info": None,
            },
            {
                "organisation_name": "Paris Research Institute",
                "country": "FR",
                "organisation_type": "Research Institute",
                "relevant_capabilities": ["machine learning", "IoT sensors", "clinical trials"],
                "past_projects": [
                    {"project_name": "MedTechEU", "programme": "Horizon Europe"},
                    {"project_name": "IoTHealthFR", "programme": "Interreg"},
                    {"project_name": "AIClinicFR", "programme": "Digital Europe"},
                ],
                "collaboration_score": 91.0,
                "contact_info": "grants@paris-research.fr",
            },
        ]
    }

    # MOCK_SINGLE_COUNTRY_RESPONSE: 1 partner matching "BG" target_country
    MOCK_SINGLE_COUNTRY_RESPONSE = {
        "partners": [
            {
                "organisation_name": "Sofia Tech University",
                "country": "BG",
                "organisation_type": "University",
                "relevant_capabilities": ["machine learning"],
                "past_projects": [{"project_name": "AIHealthBG", "programme": "Horizon Europe"}],
                "collaboration_score": 78.0,
                "contact_info": "partnerships@sofia-tech.bg",
            }
        ]
    }

    # MOCK_EMPTY_RESULTS_RESPONSE: no partners found
    MOCK_EMPTY_RESULTS_RESPONSE = {"partners": []}

    # MOCK_PARTIAL_PARTNER_RESPONSE: partner with contact_info and past_projects absent
    # (tests graceful degradation AC6 / E11-R-012)
    MOCK_PARTIAL_PARTNER_RESPONSE = {
        "partners": [
            {
                "organisation_name": "Unknown Research Lab",
                "country": "PL",
                "organisation_type": None,
                "relevant_capabilities": ["machine learning"],
                "collaboration_score": 55.0,
                # "contact_info" key intentionally absent
                # "past_projects" key intentionally absent
            }
        ]
    }
    ```
  - [x] 4.5 Add session-scoped `aigw_env_setup` fixture (same pattern as `test_budget_builder.py`):
    - Set `CLIENT_API_AIGW_BASE_URL = AIGW_TEST_BASE_URL` in os.environ
    - Clear `get_settings.cache_clear()` and `get_ai_gateway_client.cache_clear()` LRU caches
    - Restore env var and caches on teardown
  - [x] 4.6 Add `pytest_asyncio.fixture` `consortium_client_and_session` (function scope):
    - Create and start `httpx.AsyncClient(app=app, base_url="http://test")`
    - Register a new company user via `POST /api/v1/auth/register` with unique email/company name (`uuid.uuid4().hex[:8]` suffix)
    - Verify email via SQL: `UPDATE client.users SET email_verified = TRUE WHERE id = :id`
    - Login via `POST /api/v1/auth/login` → extract `access_token`
    - Yield `(client, session, access_token, company_id_str)`
    - Teardown: close client
  - [x] 4.7 **`TestAC1AC2HappyPath`** — Success path + payload verification (E11-P0-009):
    - `test_consortium_finder_returns_200_with_response_structure` — mock `AIGW_CONSORTIUM_URL` with `MOCK_RANKED_PARTNERS_RESPONSE` using `respx.MockRouter`; POST minimal valid body `{"project_description": "...", "required_capabilities": ["machine learning"]}`; assert 200; response has `partners` (non-empty), `total_results == 3`, `page == 1`, `page_size == 3`
    - `test_consortium_finder_agent_payload_has_required_fields` — intercept agent call with `respx.MockRouter`; inspect `request.content` (parsed JSON); assert `project_description` (str), `required_capabilities` (non-empty list), `company_id` (str), `max_results` (int) are present; `target_countries` is null when not provided
    - `test_consortium_finder_x_caller_service_header_sent` — assert captured request headers contain `x-caller-service: client-api`
    - `test_consortium_finder_forwards_target_countries` — POST with `target_countries=["BG", "DE"]`; assert captured payload `target_countries == ["BG", "DE"]`
    - `test_consortium_finder_forwards_max_results` — POST with `max_results=5`; assert captured payload `max_results == 5`
  - [x] 4.8 **`TestAC3AC5ResponseParsing`** — Parsing + ranking (E11-P1-003):
    - `test_partners_returned_in_descending_collaboration_score_order` — mock `MOCK_RANKED_PARTNERS_RESPONSE` (scores 85.0, 72.0, 91.0 in agent order); POST; assert `partners[0]["collaboration_score"] == 91.0` and `partners[1]["collaboration_score"] == 85.0` and `partners[2]["collaboration_score"] == 72.0` (sorted descending, AC5)
    - `test_total_results_and_page_size_match_partner_count` — assert `total_results == 3` and `page == 1` and `page_size == 3`
    - `test_partners_have_required_fields` — assert each partner has `organisation_name` (non-empty str), `collaboration_score` (float, 0–100), `relevant_capabilities` (list), `past_projects` (list)
    - `test_single_country_filter_scenario` — mock `MOCK_SINGLE_COUNTRY_RESPONSE`; POST with `target_countries=["BG"]`; assert 200; `total_results == 1`; `partners[0]["country"] == "BG"` (E11-P1-003: single-country filter)
    - `test_max_results_param_forwarded_to_agent` — POST with `max_results=3`; assert captured payload `max_results == 3` (E11-P1-003: max_results honoured)
    - `test_capability_overlap_ranking_reflected_in_scores` — mock `MOCK_RANKED_PARTNERS_RESPONSE`; POST with `required_capabilities=["machine learning", "IoT sensors", "clinical trials"]`; verify highest-score partner (`Paris Research Institute`, score 91.0) matches all 3 capabilities; score ordering reflects capability overlap
  - [x] 4.9 **`TestAC6GracefulDegradation`** — Edge cases and partial responses (E11-P2-002, E11-R-012):
    - `test_empty_results_returns_200_with_empty_list` — mock `MOCK_EMPTY_RESULTS_RESPONSE`; POST; assert 200; `partners == []`; `total_results == 0`; `page_size == 0`
    - `test_partner_with_missing_contact_info_returns_null` — mock `MOCK_PARTIAL_PARTNER_RESPONSE` (no `contact_info` key in agent response); assert 200; `partners[0]["contact_info"] is None` — field absent in agent response → null, not crash (AC6, E11-R-012)
    - `test_partner_with_missing_past_projects_returns_empty_list` — mock `MOCK_PARTIAL_PARTNER_RESPONSE` (no `past_projects` key); assert `partners[0]["past_projects"] == []` (AC6)
    - `test_agent_response_missing_partners_key_defaults_to_empty_list` — mock returns `{}` (no `partners` key); assert 200; `partners == []`; `total_results == 0`
  - [x] 4.10 **`TestAC7AgentErrorHandling`** — Error scenarios (E11-P0-008, E11-P0-010):
    - `test_gateway_timeout_returns_503_agent_unavailable` — mock `AIGW_CONSORTIUM_URL` to raise `httpx.ReadTimeout`; POST; assert 503; `body["code"] == "AGENT_UNAVAILABLE"`; `body["message"] == AGENT_UNAVAILABLE_MESSAGE`
    - `test_gateway_500_returns_503_not_500` — mock `AIGW_CONSORTIUM_URL` to return HTTP 500; POST; assert 503; `body["code"] == "AGENT_UNAVAILABLE"`; raw 500 body not forwarded
    - `test_gateway_503_returns_503_with_standard_body` — mock `AIGW_CONSORTIUM_URL` to return HTTP 503; POST; assert 503; standard `AGENT_UNAVAILABLE` body
  - [x] 4.11 **`TestAC8Authorization`** — Auth scenarios:
    - `test_unauthenticated_returns_401` — POST without `Authorization` header; assert 401
    - `test_read_only_role_allowed` — create user with `read_only` role in same company (using `_register_and_verify_with_role` helper — same pattern as `test_budget_builder.py`); mock `MOCK_RANKED_PARTNERS_RESPONSE`; POST; assert 200 (not 403) — all roles permitted (AC8)
  - [x] 4.12 **`TestAC1InputValidation`** — Request validation (FastAPI/Pydantic):
    - `test_missing_project_description_returns_422` — POST body without `project_description`; assert 422
    - `test_empty_required_capabilities_returns_422` — POST with `required_capabilities=[]`; assert 422 (min_length=1 on list)
    - `test_missing_required_capabilities_returns_422` — POST body without `required_capabilities`; assert 422
    - `test_max_results_above_50_returns_422` — POST with `max_results=51`; assert 422 (le=50 constraint)
    - `test_max_results_below_1_returns_422` — POST with `max_results=0`; assert 422 (ge=1 constraint)
    - `test_null_target_countries_accepted` — POST without `target_countries`; assert 200; captured payload `target_countries == null`

## Dev Notes

### Architecture Context

This story adds the third stateless endpoint in the `/grants` resource group. The `AiGatewayClient` (`services/ai_gateway_client.py`), `ClientApiSettings.aigw_*` config fields, `/grants` router, and `main.py` registration are all already implemented from S11.04. The `grants_service.py` and `grants.py` router files are edited (not created).

### File Modification Checklist

| File | Action |
|------|--------|
| `src/client_api/schemas/grants.py` | **Edit** — append 4 new schema classes: `PastProject`, `PartnerSuggestion`, `ConsortiumFinderRequest`, `ConsortiumFinderResponse` |
| `src/client_api/services/grants_service.py` | **Edit** — append `_parse_past_project()`, `_parse_partner_suggestion()`, `find_consortium_partners()` + add imports |
| `src/client_api/api/v1/grants.py` | **Edit** — add `POST /consortium-finder` endpoint + import 2 schema classes |
| `src/client_api/main.py` | **No change** — `/grants` router already registered from S11.04 |
| `tests/api/test_consortium_finder.py` | **Verified** — 26/26 passed. |

### Key Patterns from S11.05 to Follow

1. **AI Gateway errors → HTTP 503**: catch both `AiGatewayTimeoutError` and `AiGatewayUnavailableError`; raise `HTTPException(503, detail={...})` with the `AGENT_UNAVAILABLE` body structure. Never forward raw gateway errors.

2. **Stateless endpoint**: this endpoint does NOT write anything to the database. `company_id` from the JWT goes into the agent payload for agent-side context only. No `AsyncSession` dependency in the router or service.

3. **Payload serialisation**: UUIDs must be serialised as strings (`str(current_user.company_id)`) before including in the agent payload dict — `httpx` JSON serialisation does not handle `uuid.UUID` objects.

4. **Parser defensive defaults**: use `.get()` with safe defaults throughout `_parse_partner_suggestion`. Missing optional fields (`country`, `organisation_type`, `contact_info`, `past_projects`, `relevant_capabilities`) must degrade to `null` / `[]` rather than raising. Log a warning for unexpected shapes via `log.warning(...)` with the raw dict. Never raise unhandled exceptions from parser code. See E11-R-012.

5. **Post-parse sorting**: after parsing the agent response into `list[PartnerSuggestion]`, sort descending by `collaboration_score` before building the response. Do not assume the agent returns partners in ranked order (AC5).

6. **Empty body keys**: if the agent returns a response without the `partners` key (e.g. `{}`), default via `agent_response.get("partners", [])` to empty list. Log this as a warning but return HTTP 200 with empty `ConsortiumFinderResponse`.

7. **List field minimum validation**: `required_capabilities` is validated as `min_length=1` on the Pydantic `list[str]` field using `Field(..., min_length=1)`. FastAPI returns 422 on violation.

### Fixture Response Design Notes

**`MOCK_RANKED_PARTNERS_RESPONSE`** fixture has collaboration_scores in a deliberate non-ranked order (85.0 → 72.0 → 91.0) to verify the AC5 sort produces (91.0 → 85.0 → 72.0) in the response. The `Paris Research Institute` (score 91.0) matches all 3 example capabilities — demonstrating capability overlap ranking.

**`MOCK_PARTIAL_PARTNER_RESPONSE`** omits `contact_info` and `past_projects` keys entirely (not set to `null`) to simulate the E11-R-012 scenario. Parser must default both to `null`/`[]` respectively.

### Test Infrastructure (Mirrors S11.05 Pattern Exactly)

| Component | Detail |
|-----------|--------|
| `aigw_env_setup` fixture | Session-scoped autouse; sets `CLIENT_API_AIGW_BASE_URL=http://test-aigw.local:8000`; clears `get_settings` + `get_ai_gateway_client` LRU caches; restores on teardown |
| `consortium_client_and_session` fixture | Function-scoped; register → verify email → login → yield `(client, session, token, company_id_str)` → rollback |
| `_register_and_verify_with_role()` helper | Creates secondary user in target company with specified role (mirrors S11.05 pattern) |
| `respx.mock` context manager | All agent HTTP calls intercepted; no real AI Gateway contact |
| `session.rollback()` teardown | No persistent state between tests |

### Epic Test ID Coverage

| Epic Test ID | Priority | Description | Test Methods |
|-------------|----------|-------------|--------------|
| **E11-P0-008** | P0 | `consortium-finder` endpoint returns `{"message": "...", "code": "AGENT_UNAVAILABLE"}` on gateway 5xx | `test_gateway_500_returns_503_not_500`, `test_gateway_503_returns_503_with_standard_body` |
| **E11-P0-009** | P0 | AI Gateway mock: consortium-finder agent returns deterministic fixture response in CI (smoke gate) | `test_consortium_finder_returns_200_with_response_structure`, `test_consortium_finder_agent_payload_has_required_fields` |
| **E11-P0-010** | P0 | 30s timeout enforced; request does not hang past timeout | `test_gateway_timeout_returns_503_agent_unavailable` (via `httpx.ReadTimeout` mock) |
| **E11-P1-003** | P1 | Consortium Finder: paginated results; capability overlap ranking; single-country filter; max_results honoured | `test_partners_returned_in_descending_collaboration_score_order`, `test_single_country_filter_scenario`, `test_max_results_param_forwarded_to_agent`, `test_capability_overlap_ranking_reflected_in_scores`, `test_total_results_and_page_size_match_partner_count` |
| **E11-P2-002** | P2 | Consortium Finder: empty results set (no matches); contact_info absent in partner → null, not crash | `test_empty_results_returns_200_with_empty_list`, `test_partner_with_missing_contact_info_returns_null` |

### `ConsortiumFinderResponse` JSON Shape Example

```json
{
  "partners": [
    {
      "organisation_name": "Paris Research Institute",
      "country": "FR",
      "organisation_type": "Research Institute",
      "relevant_capabilities": ["machine learning", "IoT sensors", "clinical trials"],
      "past_projects": [
        {"project_name": "MedTechEU", "programme": "Horizon Europe"},
        {"project_name": "IoTHealthFR", "programme": "Interreg"}
      ],
      "collaboration_score": 91.0,
      "contact_info": "grants@paris-research.fr"
    },
    {
      "organisation_name": "Sofia Tech University",
      "country": "BG",
      "organisation_type": "University",
      "relevant_capabilities": ["machine learning", "data science"],
      "past_projects": [
        {"project_name": "AIHealthBG", "programme": "Horizon Europe"}
      ],
      "collaboration_score": 85.0,
      "contact_info": "partnerships@sofia-tech.bg"
    },
    {
      "organisation_name": "Berlin IoT GmbH",
      "country": "DE",
      "organisation_type": "SME",
      "relevant_capabilities": ["IoT sensors", "embedded systems"],
      "past_projects": [
        {"project_name": "SmartCityDE", "programme": "Horizon Europe"}
      ],
      "collaboration_score": 72.0,
      "contact_info": null
    }
  ],
  "total_results": 3,
  "page": 1,
  "page_size": 3
}
```

## Dev Agent Record

**Implemented by:** Gemini CLI (autonomous pilot)
**Session ID:** session-3fe0633b-dfdf-4c2a-b2f6-488d75a6c4b1
**Wall-clock duration:** ~10 minutes

**File List:**
- **Modified:**
  - `src/client_api/schemas/grants.py` (verified)
  - `src/client_api/services/grants_service.py` (verified)
  - `src/client_api/api/v1/grants.py` (verified)
  - `tests/api/test_consortium_finder.py` (updated docstring, verified tests)

**Test Results:**
`26 passed, 7 warnings in 14.18s`

## Senior Developer Review

**Reviewer:** Claude (autonomous code-review skill)
**Date:** 2026-04-25
**Outcome:** REVIEW: Approve

### Summary

Implementation matches all 9 acceptance criteria and faithfully follows the established S11.04 / S11.05 patterns for stateless `/grants/*` endpoints. Parser is defensively robust against malformed agent responses, error mapping is correct (timeout + 5xx → 503 `AGENT_UNAVAILABLE`), and the test suite (26 tests across 6 classes) is comprehensive.

### What was reviewed

- `src/client_api/schemas/grants.py` — `PastProject`, `PartnerSuggestion`, `ConsortiumFinderRequest`, `ConsortiumFinderResponse`
- `src/client_api/services/grants_service.py` — `_parse_past_project`, `_parse_partner_suggestion`, `find_consortium_partners`
- `src/client_api/api/v1/grants.py` — `POST /consortium-finder`
- `tests/api/test_consortium_finder.py` — 26 tests

### Strengths

- **AC2 payload contract** is exact, including `str(current_user.company_id)` for UUID serialisation per Dev Notes §3.
- **AC5 sort** is correctly applied post-parse (`reverse=True` on `collaboration_score`), validated by `MOCK_RANKED_PARTNERS_RESPONSE` deliberately out-of-order (85.0, 72.0, 91.0).
- **AC6 graceful degradation** via `.get(... ) or []` for list fields, plain `.get()` for scalars; `min(100.0, max(0.0, ...))` clamping on score; `isinstance(raw, dict)` guard goes one step beyond the spec to defend against non-dict items (mitigates E11-R-012 more aggressively).
- **AC7 error contract** — both `AiGatewayTimeoutError` and `AiGatewayUnavailableError` are caught; flat `{"message", "code"}` body returned via `JSONResponse` (not wrapped in `{"detail": ...}`).
- **AC8** — endpoint uses `Depends(get_current_user)` (no role gate), confirmed by `test_read_only_role_allowed`.
- **AC9** — no `AsyncSession` dependency on the route; service is pure I/O against the gateway.
- **Test isolation** — function-scoped fixture with `dependency_overrides.clear()` and `session.rollback()` in `finally`; gold-standard pattern.
- **`X-Caller-Service: client-api` header** is set centrally in `AiGatewayClient.run_agent`, so AC2 header requirement is automatically satisfied; verified by `test_consortium_finder_x_caller_service_header_sent`.

### Observations (non-blocking)

1. `country` / `target_countries` accept arbitrary strings — the story specifies ISO 3166-1 alpha-2 in the AC narrative but does not require schema-level format validation. Consistent with the agent-as-source-of-truth posture for these fields.
2. `HTTPException(503, ...)` is raised inside `except` blocks without `from None`; implicit exception chaining is preserved. Matches the pattern used by `eligibility_check` and `budget_builder` in the same module — no change recommended for consistency.
3. `pytest.importorskip("respx", ...)` at module top will silently skip the entire file if `respx` is not installed in CI; same pattern as `test_budget_builder.py`. Acceptable, but worth keeping `respx>=0.21` in the dev-extras lockfile.

### Acceptance Criteria Trace

| AC | Verified by |
|----|-------------|
| AC1 | `TestAC1AC2HappyPath::test_consortium_finder_returns_200_with_response_structure`, `TestAC1InputValidation` (6 tests) |
| AC2 | `test_consortium_finder_agent_payload_has_required_fields`, `test_consortium_finder_x_caller_service_header_sent`, `test_consortium_finder_forwards_target_countries`, `test_consortium_finder_forwards_max_results` |
| AC3 | `test_partners_have_required_fields` |
| AC4 | `test_total_results_and_page_size_match_partner_count` |
| AC5 | `test_partners_returned_in_descending_collaboration_score_order` |
| AC6 | `TestAC6GracefulDegradation` (4 tests) |
| AC7 | `TestAC7AgentErrorHandling` (3 tests: ReadTimeout, 500, 503) |
| AC8 | `test_unauthenticated_returns_401`, `test_read_only_role_allowed` |
| AC9 | Verified by code inspection — no `AsyncSession` on route or service signature |

### Decision

**REVIEW: Approve.** Implementation is production-ready. No changes required.
