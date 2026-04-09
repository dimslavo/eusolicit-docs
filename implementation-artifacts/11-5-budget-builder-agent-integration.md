# Story 11.5: Budget Builder Agent Integration

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **company admin or bid manager**,
I want to submit a project description and parameters and receive an AI-generated EU-compliant budget,
so that I can build a grant application budget with correct cost categories, overhead calculation, co-financing split, and (when applicable) a per-partner breakdown — ready to include in a submission package.

## Acceptance Criteria

1. **AC1** — `POST /api/v1/grants/budget-builder` accepts a JSON request body with:
   - `project_description` (str, required — the narrative of the proposed project)
   - `duration_months` (int, required, `≥ 1` — project duration in months)
   - `total_requested_funding` (float, required, `> 0` — total EU funding requested in EUR)
   - `consortium_size` (int, optional, `≥ 1`, default `1` — number of consortium partners)
   - `overhead_rate` (float, optional, `0.0–1.0` expressed as a decimal, default `null` — e.g. `0.25` for 25%; if null the agent applies programme-default overhead rules)
   - `target_programme` (str, optional, default `null` — e.g. `"horizon_europe"`, `"digital_europe"`, `"structural_funds"`, `"interreg"`; if null the agent applies generic EU cost structure rules)
   Returns HTTP 200 with a `BudgetBuilderResponse`. Any authenticated user may call this endpoint — no minimum role restriction beyond a valid JWT with `company_id`.

2. **AC2** — The AI Gateway agent payload sent to logical agent name `budget-builder` (URL: `{AIGW_BASE_URL}/agents/budget-builder/run`) must be structured as:
   ```json
   {
     "project_description": "<string>",
     "duration_months": <int>,
     "total_requested_funding": <float>,
     "consortium_size": <int>,
     "overhead_rate": <float or null>,
     "target_programme": "<string or null>",
     "company_id": "<company UUID as string>"
   }
   ```
   The request must include the header `X-Caller-Service: client-api`. The timeout is 30 seconds (configurable via `AIGW_TIMEOUT_SECONDS`). The client-api does NOT retry — the AI Gateway (E04) handles retry and circuit-breaker logic internally.

3. **AC3** — The agent response is parsed into a `BudgetBuilderResponse` containing:
   - `cost_categories` (list of `CostCategory`, must be non-empty): each item has:
     - `name` (str, required — EU cost category label e.g. `"Personnel"`, `"Subcontracting"`, `"Travel"`, `"Equipment"`, `"Other Direct Costs"`, `"Indirect Costs"`)
     - `amount` (float, required, `≥ 0`)
     - `justification` (str or null)
   - `total_budget` (float): the budget total as declared by the agent
   - `overhead_calculation` (`OverheadCalculation`): object with:
     - `base_direct_costs` (float) — sum of direct cost categories (i.e. all categories excluding `"Indirect Costs"`)
     - `overhead_rate` (float) — the rate applied (as decimal; echoed from request or agent-determined default)
     - `overhead_amount` (float) — the resulting overhead monetary amount
   - `co_financing_split` (`CoFinancingSplit`): object with:
     - `eu_contribution` (float)
     - `own_contribution` (float)
     - `eu_rate` (float) — EU share as decimal (e.g. `0.80`)
     - `total_requested_funding` (float) — echoed from the request
   - `per_partner_breakdown` (list of `PartnerBudget` or null): present and non-empty when `consortium_size > 1`; each item has:
     - `partner_name` (str)
     - `partner_country` (str or null)
     - `cost_categories` (list of `CostCategory`)
     - `subtotal` (float)
   - `target_programme` (str or null): echoed from request
   - `duration_months` (int): echoed from request
   - `consortium_size` (int): echoed from request

4. **AC4** — Budget arithmetic validation: after parsing the agent response the service MUST verify the following arithmetic invariants before returning the response to the caller. A tolerance of `0.01` EUR is applied for all floating-point comparisons.
   - **Line-item sum**: `abs(sum(cat.amount for cat in cost_categories) - total_budget) < 0.01`
   - **Co-financing sum**: `abs(co_financing_split.eu_contribution + co_financing_split.own_contribution - co_financing_split.total_requested_funding) < 0.01`
   - **Overhead consistency** (only when `overhead_rate` is not null in the parsed response): `abs(overhead_calculation.overhead_amount - overhead_calculation.overhead_rate * overhead_calculation.base_direct_costs) < 0.01`
   - **Per-partner sum** (only when `per_partner_breakdown` is not null and non-empty): `abs(sum(p.subtotal for p in per_partner_breakdown) - total_budget) < 0.01`
   
   If any invariant fails, return HTTP 422 with:
   ```json
   {"detail": "Agent returned an arithmetically inconsistent budget. Please retry.", "code": "BUDGET_ARITHMETIC_INCONSISTENT"}
   ```
   Do not return the malformed budget to the caller.

5. **AC5** — Per-partner breakdown requirement: when the request's `consortium_size > 1`, the parser MUST expect `per_partner_breakdown` to be present in the agent response. If the agent response is missing `per_partner_breakdown` (or it is null) when `consortium_size > 1`, return HTTP 422 with:
   ```json
   {"detail": "Agent did not return a per-partner breakdown for a consortium budget. Please retry.", "code": "MISSING_PARTNER_BREAKDOWN"}
   ```
   When `consortium_size == 1`, `per_partner_breakdown` is accepted as null or absent.

6. **AC6** — AI Gateway error handling: if the AI Gateway times out (> 30 s) or returns HTTP 5xx, the endpoint returns HTTP 503 to the caller with:
   ```json
   {"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"}
   ```
   No data is persisted on agent failure. The raw agent response is never forwarded to the caller.

7. **AC7** — Unauthenticated requests (missing or expired JWT) return HTTP 401. All authenticated roles (read_only, contributor, reviewer, bid_manager, admin) may call this endpoint without restriction.

8. **AC8** — The endpoint is stateless — no budget data is written to the database. A `company_id` from the JWT is included in the agent payload for agent-side context but no snapshot or audit record is persisted by this endpoint. (Budget snapshot persistence is deferred to a future story once a `budget_snapshots` table is defined.)

## Tasks / Subtasks

- [x] Task 1: Add Budget Builder schemas to `src/client_api/schemas/grants.py` (AC: 1, 3, 4, 5)
  - [x] 1.1 Define `CostCategory(BaseModel)`:
    - `name: str`
    - `amount: float = Field(..., ge=0.0)`
    - `justification: str | None = None`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.2 Define `OverheadCalculation(BaseModel)`:
    - `base_direct_costs: float = Field(..., ge=0.0)`
    - `overhead_rate: float = Field(..., ge=0.0, le=1.0)`
    - `overhead_amount: float = Field(..., ge=0.0)`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.3 Define `CoFinancingSplit(BaseModel)`:
    - `eu_contribution: float = Field(..., ge=0.0)`
    - `own_contribution: float = Field(..., ge=0.0)`
    - `eu_rate: float = Field(..., ge=0.0, le=1.0)`
    - `total_requested_funding: float = Field(..., gt=0.0)`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.4 Define `PartnerBudget(BaseModel)`:
    - `partner_name: str`
    - `partner_country: str | None = None`
    - `cost_categories: list[CostCategory]`
    - `subtotal: float = Field(..., ge=0.0)`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.5 Define `BudgetBuilderRequest(BaseModel)`:
    - `project_description: str = Field(..., min_length=1)`
    - `duration_months: int = Field(..., ge=1)`
    - `total_requested_funding: float = Field(..., gt=0.0)`
    - `consortium_size: int = Field(default=1, ge=1)`
    - `overhead_rate: float | None = Field(default=None, ge=0.0, le=1.0)`
    - `target_programme: str | None = None`
    - `model_config = ConfigDict(extra="ignore")`
  - [x] 1.6 Define `BudgetBuilderResponse(BaseModel)`:
    - `cost_categories: list[CostCategory]`
    - `total_budget: float`
    - `overhead_calculation: OverheadCalculation`
    - `co_financing_split: CoFinancingSplit`
    - `per_partner_breakdown: list[PartnerBudget] | None = None`
    - `target_programme: str | None = None`
    - `duration_months: int`
    - `consortium_size: int`

- [x] Task 2: Implement budget builder service in `src/client_api/services/grants_service.py` (AC: 1–8)
  - [x] 2.1 Add imports to the existing `grants_service.py`: `BudgetBuilderRequest`, `BudgetBuilderResponse`, `CostCategory`, `OverheadCalculation`, `CoFinancingSplit`, `PartnerBudget` from `client_api.schemas.grants`
  - [x] 2.2 Define `_parse_cost_category(raw: dict) -> CostCategory | None`:
    - Extract fields with `.get()` safe defaults: `name = raw.get("name", "Unknown")`, `amount = float(raw.get("amount", 0.0))`, `justification = raw.get("justification")`
    - Return `CostCategory(name=name, amount=max(0.0, amount), justification=justification)`
    - Wrap in `try/except (KeyError, ValueError, TypeError)`: log warning via `log.warning("grants.budget_builder.parse_cost_category.error", raw=raw)`; return `None`
  - [x] 2.3 Define `_parse_partner_budget(raw: dict) -> PartnerBudget | None`:
    - Extract `partner_name`, `partner_country`, `cost_categories` (list of raw dicts), `subtotal`
    - Map `cost_categories` through `_parse_cost_category`; filter out `None`
    - Return `PartnerBudget(partner_name=raw.get("partner_name", "Unknown"), partner_country=raw.get("partner_country"), cost_categories=parsed_cats, subtotal=max(0.0, float(raw.get("subtotal", 0.0))))`
    - Wrap in `try/except (KeyError, ValueError, TypeError)`: log warning; return `None`
  - [x] 2.4 Define `_validate_budget_arithmetic(response: BudgetBuilderResponse, request: BudgetBuilderRequest) -> list[str]`:
    - Returns a list of violation strings (empty list = all checks pass)
    - **Check 1 — line-item sum**: `computed = sum(cat.amount for cat in response.cost_categories)`. If `abs(computed - response.total_budget) >= 0.01`: append `f"Line items sum ({computed:.2f}) does not match total_budget ({response.total_budget:.2f})"`
    - **Check 2 — co-financing sum**: `split = response.co_financing_split`. If `abs(split.eu_contribution + split.own_contribution - split.total_requested_funding) >= 0.01`: append violation string
    - **Check 3 — overhead** (only if `response.overhead_calculation.overhead_rate > 0`): `oc = response.overhead_calculation`. If `abs(oc.overhead_amount - oc.overhead_rate * oc.base_direct_costs) >= 0.01`: append violation string
    - **Check 4 — per-partner sum** (only if `response.per_partner_breakdown`): `partner_total = sum(p.subtotal for p in response.per_partner_breakdown)`. If `abs(partner_total - response.total_budget) >= 0.01`: append violation string
    - Return the list
  - [x] 2.5 Implement `async def build_budget(request: BudgetBuilderRequest, current_user: CurrentUser, gw_client: AiGatewayClient) -> BudgetBuilderResponse`:
    - Build payload (AC2):
      ```python
      payload = {
          "project_description": request.project_description,
          "duration_months": request.duration_months,
          "total_requested_funding": request.total_requested_funding,
          "consortium_size": request.consortium_size,
          "overhead_rate": request.overhead_rate,
          "target_programme": request.target_programme,
          "company_id": str(current_user.company_id),
      }
      ```
    - Call agent (AC6):
      ```python
      try:
          agent_response = await gw_client.run_agent("budget-builder", payload)
      except AiGatewayTimeoutError:
          log.warning("grants.budget_builder.timeout", company_id=str(current_user.company_id))
          raise HTTPException(
              status_code=503,
              detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"},
          )
      except AiGatewayUnavailableError:
          log.warning("grants.budget_builder.unavailable", company_id=str(current_user.company_id))
          raise HTTPException(
              status_code=503,
              detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"},
          )
      ```
    - Parse cost_categories (AC3): `raw_cats = agent_response.get("cost_categories", [])`; map through `_parse_cost_category`; filter `None`
    - Parse overhead_calculation (AC3): extract `raw_oc = agent_response.get("overhead_calculation", {})`; build `OverheadCalculation(base_direct_costs=float(raw_oc.get("base_direct_costs", 0.0)), overhead_rate=float(raw_oc.get("overhead_rate", 0.0)), overhead_amount=float(raw_oc.get("overhead_amount", 0.0)))`
    - Parse co_financing_split (AC3): extract `raw_cfs = agent_response.get("co_financing_split", {})`; build `CoFinancingSplit(eu_contribution=float(raw_cfs.get("eu_contribution", 0.0)), own_contribution=float(raw_cfs.get("own_contribution", 0.0)), eu_rate=float(raw_cfs.get("eu_rate", 0.0)), total_requested_funding=float(raw_cfs.get("total_requested_funding", request.total_requested_funding)))`
    - Parse per_partner_breakdown (AC3, AC5): if `agent_response.get("per_partner_breakdown")`, map raw list through `_parse_partner_budget`; filter `None`; else `None`
    - Per-partner requirement check (AC5): if `request.consortium_size > 1` and `per_partner_breakdown is None or len(per_partner_breakdown) == 0`: raise `HTTPException(status_code=422, detail={"detail": "Agent did not return a per-partner breakdown for a consortium budget. Please retry.", "code": "MISSING_PARTNER_BREAKDOWN"})`
    - Build `BudgetBuilderResponse` from parsed components
    - Arithmetic validation (AC4): `violations = _validate_budget_arithmetic(budget_response, request)`. If violations: log warning (`log.warning("grants.budget_builder.arithmetic_inconsistency", violations=violations)`); raise `HTTPException(status_code=422, detail={"detail": "Agent returned an arithmetically inconsistent budget. Please retry.", "code": "BUDGET_ARITHMETIC_INCONSISTENT"})`
    - Log success: `log.info("grants.budget_builder.completed", company_id=str(current_user.company_id), total_budget=budget_response.total_budget, consortium_size=request.consortium_size, target_programme=request.target_programme)`
    - Return `budget_response`

- [x] Task 3: Add `POST /grants/budget-builder` endpoint to `src/client_api/api/v1/grants.py` (AC: 1, 7)
  - [x] 3.1 Add imports to the existing `grants.py`: `BudgetBuilderRequest`, `BudgetBuilderResponse` from `client_api.schemas.grants`
  - [x] 3.2 Implement `POST /budget-builder` endpoint in the existing `router`:
    ```python
    @router.post("/budget-builder", response_model=BudgetBuilderResponse)
    async def budget_builder(
        body: BudgetBuilderRequest,
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        gw_client: Annotated[AiGatewayClient, Depends(get_ai_gateway_client)],
    ) -> BudgetBuilderResponse | JSONResponse:
        """Generate EU-compliant grant budget via Budget Builder Agent."""
        try:
            return await grants_service.build_budget(body, current_user, gw_client)
        except HTTPException as exc:
            if exc.status_code == 503 and isinstance(exc.detail, dict):
                return JSONResponse(status_code=503, content=exc.detail)
            if exc.status_code == 422 and isinstance(exc.detail, dict):
                return JSONResponse(status_code=422, content=exc.detail)
            raise
    ```
  - [x] 3.3 Note: no `AsyncSession` dependency needed — `build_budget` does not query the DB (stateless, AC8). `get_current_user` provides `company_id` for the agent payload.
  - [x] 3.4 Note: use `Depends(get_current_user)` (not `require_role`) so all authenticated roles are permitted (AC7).

- [x] Task 4: Write tests in `tests/api/test_budget_builder.py` (AC: 1–8; E11-P0-005, E11-P0-008, E11-P0-009, E11-P0-010, E11-P1-002, E11-P2-001)
  - [x] 4.1 Add module docstring listing covered test IDs: `E11-P0-005`, `E11-P0-008`, `E11-P0-009`, `E11-P0-010`, `E11-P1-002`, `E11-P2-001` and link to this story file
  - [x] 4.2 Add `respx = pytest.importorskip("respx", reason="respx required for AI Gateway mocking")` guard
  - [x] 4.3 Define module-level constants:
    ```python
    AIGW_TEST_BASE_URL = "http://test-aigw.local:8000"
    AIGW_BUDGET_URL = f"{AIGW_TEST_BASE_URL}/agents/budget-builder/run"
    ENDPOINT = "/api/v1/grants/budget-builder"
    AGENT_UNAVAILABLE_CODE = "AGENT_UNAVAILABLE"
    AGENT_UNAVAILABLE_MESSAGE = "AI features are temporarily unavailable. Please try again."
    ARITHMETIC_INCONSISTENT_CODE = "BUDGET_ARITHMETIC_INCONSISTENT"
    MISSING_PARTNER_BREAKDOWN_CODE = "MISSING_PARTNER_BREAKDOWN"
    ```
  - [x] 4.4 Define deterministic fixture responses:
    ```python
    MOCK_SOLO_BUDGET_RESPONSE = {
        "cost_categories": [
            {"name": "Personnel", "amount": 750000.0, "justification": "3 FTE researchers × 24 months"},
            {"name": "Subcontracting", "amount": 200000.0, "justification": "External software development"},
            {"name": "Travel", "amount": 50000.0, "justification": "Consortium and dissemination travel"},
            {"name": "Equipment", "amount": 100000.0, "justification": "High-performance computing servers"},
            {"name": "Other Direct Costs", "amount": 50000.0, "justification": "Consumables and publications"},
            {"name": "Indirect Costs", "amount": 287500.0, "justification": "Flat-rate 25% overhead on direct costs"},
        ],
        "total_budget": 1437500.0,
        "overhead_calculation": {
            "base_direct_costs": 1150000.0,
            "overhead_rate": 0.25,
            "overhead_amount": 287500.0,
        },
        "co_financing_split": {
            "eu_contribution": 1150000.0,
            "own_contribution": 287500.0,
            "eu_rate": 0.80,
            "total_requested_funding": 1437500.0,
        },
        "per_partner_breakdown": None,
    }

    MOCK_CONSORTIUM_BUDGET_RESPONSE = {
        "cost_categories": [
            {"name": "Personnel", "amount": 1200000.0, "justification": "5 FTE across 3 partners"},
            {"name": "Subcontracting", "amount": 300000.0, "justification": ""},
            {"name": "Travel", "amount": 80000.0, "justification": ""},
            {"name": "Equipment", "amount": 200000.0, "justification": ""},
            {"name": "Other Direct Costs", "amount": 70000.0, "justification": ""},
            {"name": "Indirect Costs", "amount": 462500.0, "justification": "Flat-rate 25%"},
        ],
        "total_budget": 2312500.0,
        "overhead_calculation": {
            "base_direct_costs": 1850000.0,
            "overhead_rate": 0.25,
            "overhead_amount": 462500.0,
        },
        "co_financing_split": {
            "eu_contribution": 1850000.0,
            "own_contribution": 462500.0,
            "eu_rate": 0.80,
            "total_requested_funding": 2312500.0,
        },
        "per_partner_breakdown": [
            {
                "partner_name": "Lead Organisation",
                "partner_country": "BG",
                "cost_categories": [
                    {"name": "Personnel", "amount": 600000.0, "justification": ""},
                    {"name": "Indirect Costs", "amount": 150000.0, "justification": ""},
                ],
                "subtotal": 750000.0,
            },
            {
                "partner_name": "Partner 2",
                "partner_country": "DE",
                "cost_categories": [
                    {"name": "Personnel", "amount": 400000.0, "justification": ""},
                    {"name": "Equipment", "amount": 200000.0, "justification": ""},
                    {"name": "Indirect Costs", "amount": 150000.0, "justification": ""},
                ],
                "subtotal": 750000.0,
            },
            {
                "partner_name": "Partner 3",
                "partner_country": "FR",
                "cost_categories": [
                    {"name": "Personnel", "amount": 200000.0, "justification": ""},
                    {"name": "Subcontracting", "amount": 300000.0, "justification": ""},
                    {"name": "Travel", "amount": 80000.0, "justification": ""},
                    {"name": "Other Direct Costs", "amount": 70000.0, "justification": ""},
                    {"name": "Indirect Costs", "amount": 162500.0, "justification": ""},
                ],
                "subtotal": 812500.0,
            },
        ],
    }

    MOCK_INCONSISTENT_BUDGET_RESPONSE = {
        "cost_categories": [
            {"name": "Personnel", "amount": 750000.0, "justification": ""},
            {"name": "Indirect Costs", "amount": 187500.0, "justification": ""},
        ],
        "total_budget": 999999.0,  # does NOT equal 750000 + 187500 = 937500
        "overhead_calculation": {
            "base_direct_costs": 750000.0,
            "overhead_rate": 0.25,
            "overhead_amount": 187500.0,
        },
        "co_financing_split": {
            "eu_contribution": 750000.0,
            "own_contribution": 187500.0,
            "eu_rate": 0.80,
            "total_requested_funding": 937500.0,
        },
        "per_partner_breakdown": None,
    }
    ```
  - [x] 4.5 Add session-scoped `aigw_env_setup` fixture (same pattern as `test_grant_eligibility.py`): set `CLIENT_API_AIGW_BASE_URL = AIGW_TEST_BASE_URL`; clear `get_settings` and `get_ai_gateway_client` LRU caches; restore on teardown
  - [x] 4.6 Add `pytest_asyncio.fixture` `budget_client_and_session` (function scope):
    - Create and start `httpx.AsyncClient(app=app, base_url="http://test")`
    - Register a unique company user via `POST /api/v1/auth/register` with `uuid.uuid4().hex[:8]` suffix
    - Verify email via SQL: `UPDATE client.users SET email_verified = TRUE WHERE id = :id`
    - Login via `POST /api/v1/auth/login` → extract `access_token`
    - Yield `(client, session, access_token, company_id_str)`
    - Teardown: close client

  - [x] 4.7 **`TestAC1AC2HappyPath`** — Solo budget success (E11-P0-009):
    - `test_budget_builder_returns_200_with_response_structure` — mock `AIGW_BUDGET_URL` with `MOCK_SOLO_BUDGET_RESPONSE` via `respx.MockRouter`; POST with `{"project_description": "AI research", "duration_months": 24, "total_requested_funding": 1437500.0}`; assert 200; response body contains `cost_categories` (6 items), `total_budget`, `overhead_calculation`, `co_financing_split`; `per_partner_breakdown` is null
    - `test_budget_builder_agent_payload_has_required_fields` — intercept agent call with `respx.MockRouter`; parse `request.content` JSON; assert `project_description`, `duration_months`, `total_requested_funding`, `consortium_size`, `company_id` (as string) all present; assert `overhead_rate` is null when not supplied in request
    - `test_budget_builder_x_caller_service_header_sent` — assert captured request headers contain `x-caller-service: client-api`
    - `test_budget_builder_with_all_optional_params` — POST with `overhead_rate=0.25`, `target_programme="horizon_europe"`, `consortium_size=1`; assert captured payload has `overhead_rate == 0.25` and `target_programme == "horizon_europe"`

  - [x] 4.8 **`TestAC3ResponseParsing`** — Response structure (AC3):
    - `test_cost_categories_have_required_fields` — mock solo response; POST; assert each item in `cost_categories` has `name` (non-empty str), `amount` (float ≥ 0), and `justification` (str or null)
    - `test_overhead_calculation_fields_present` — mock solo response; POST; assert `overhead_calculation` has `base_direct_costs`, `overhead_rate`, `overhead_amount` as numeric values
    - `test_co_financing_split_fields_present` — mock solo response; POST; assert `co_financing_split` has `eu_contribution`, `own_contribution`, `eu_rate`, `total_requested_funding`
    - `test_agent_response_missing_cost_categories_key_returns_graceful_error` — mock agent returning `{}` (no `cost_categories`); POST; assert 422 (arithmetic check fails because totals are zero)

  - [x] 4.9 **`TestAC4ArithmeticValidation`** — Budget arithmetic (E11-P0-005):
    - `test_valid_solo_budget_arithmetic_passes` — mock `MOCK_SOLO_BUDGET_RESPONSE`; POST; assert 200 (arithmetic is consistent)
    - `test_inconsistent_line_item_total_returns_422` — mock `MOCK_INCONSISTENT_BUDGET_RESPONSE`; POST with `total_requested_funding=937500.0`; assert 422; response body has `code == "BUDGET_ARITHMETIC_INCONSISTENT"`
    - `test_inconsistent_co_financing_sum_returns_422` — construct agent response where `eu_contribution + own_contribution != total_requested_funding`; mock and POST; assert 422; `code == "BUDGET_ARITHMETIC_INCONSISTENT"`
    - `test_overhead_inconsistency_returns_422` — construct agent response where `overhead_amount != overhead_rate * base_direct_costs` (e.g. overhead_rate=0.25, base=1000, overhead_amount=300); mock and POST; assert 422; `code == "BUDGET_ARITHMETIC_INCONSISTENT"`

  - [x] 4.10 **`TestAC5ConsortiumBudget`** — Per-partner breakdown (E11-P1-002):
    - `test_consortium_budget_returns_per_partner_breakdown` — mock `MOCK_CONSORTIUM_BUDGET_RESPONSE`; POST with `consortium_size=3`; assert 200; `per_partner_breakdown` has 3 items; each has `partner_name`, `subtotal`, `cost_categories`
    - `test_consortium_budget_per_partner_arithmetic_passes` — mock consortium response (partner subtotals sum = total_budget = 2312500.0); assert 200 (validation passes)
    - `test_consortium_budget_missing_per_partner_returns_422` — mock response with `per_partner_breakdown: null`; POST with `consortium_size=3`; assert 422; `code == "MISSING_PARTNER_BREAKDOWN"`
    - `test_solo_budget_accepts_null_per_partner` — mock solo response (`per_partner_breakdown: null`); POST with `consortium_size=1` (default); assert 200

  - [x] 4.11 **`TestAC6AgentErrorHandling`** — AI Gateway failures (E11-P0-008, E11-P0-010):
    - `test_gateway_timeout_returns_503_agent_unavailable` — mock `AIGW_BUDGET_URL` to raise `httpx.ReadTimeout`; POST; assert 503; `code == "AGENT_UNAVAILABLE"` and `message == AGENT_UNAVAILABLE_MESSAGE`
    - `test_gateway_500_returns_503_not_500` — mock `AIGW_BUDGET_URL` to return HTTP 500; POST; assert 503 (not 500); `code == "AGENT_UNAVAILABLE"`
    - `test_gateway_503_returns_503_with_standard_body` — mock `AIGW_BUDGET_URL` to return HTTP 503; POST; assert 503; `code == "AGENT_UNAVAILABLE"`

  - [x] 4.12 **`TestAC7Authorization`** — Auth scenarios:
    - `test_unauthenticated_returns_401` — POST without `Authorization` header; assert 401
    - `test_read_only_role_allowed` — create user with `read_only` role; mock full success response; POST; assert 200 (all roles permitted, AC7)

  - [x] 4.13 **`TestAC2OptionalParams`** — Optional field handling (E11-P2-001):
    - `test_missing_overhead_rate_accepted` — POST with no `overhead_rate` field; mock solo response; assert 200; captured payload has `overhead_rate == null`
    - `test_missing_target_programme_accepted` — POST with no `target_programme`; mock solo response; assert 200; captured payload has `target_programme == null`
    - `test_missing_consortium_size_defaults_to_one` — POST without `consortium_size`; mock solo response (`per_partner_breakdown: null`); assert 200; captured payload has `consortium_size == 1`
    - `test_invalid_overhead_rate_above_one_returns_422` — POST with `overhead_rate=1.5`; assert 422 (FastAPI Pydantic validation, before any agent call)
    - `test_missing_project_description_returns_422` — POST without `project_description`; assert 422
    - `test_missing_total_requested_funding_returns_422` — POST without `total_requested_funding`; assert 422

## Dev Notes

### Architecture Context

This story adds the `POST /grants/budget-builder` endpoint to the existing `/grants` resource group created in S11.04. Both the `AiGatewayClient` (implemented in S11.03) and the `grants` router (created in S11.04) already exist — this story extends them rather than creating new files from scratch.

The Budget Builder Agent (E11-R-004) is the highest-risk parser in this epic because budget arithmetic errors propagate directly to EU grant submissions. The `_validate_budget_arithmetic` function is the primary mitigation and must be called before any response is served to the caller.

### File Modification Checklist

| File | Action |
|------|--------|
| `src/client_api/schemas/grants.py` | **Edit** — append 6 new schema classes |
| `src/client_api/services/grants_service.py` | **Edit** — append 4 new functions + add imports |
| `src/client_api/api/v1/grants.py` | **Edit** — add 1 new endpoint + add imports |
| `tests/api/test_budget_builder.py` | **Create new** |

No changes required to `main.py` — the `/grants` router is already registered from S11.04.

### Key Patterns from S11.04 to Follow

1. **AI Gateway errors → HTTP 503**: Same catch block for `AiGatewayTimeoutError` and `AiGatewayUnavailableError`. The router catches `HTTPException(503)` with a `dict` detail and converts it to a flat `JSONResponse` (not a FastAPI validation error envelope).

2. **Stateless endpoint**: No `AsyncSession` dependency in the endpoint or service. `company_id` comes from the JWT via `CurrentUser`; it is included in the agent payload but nothing is written to the database (AC8).

3. **Payload serialisation**: `str(current_user.company_id)` for UUID → string conversion before including in payload dict. Floats from the request are safe to include directly as Python floats.

4. **Parser defensive defaults**: All agent response keys accessed via `.get()` with numeric defaults. Individual `_parse_*` helpers return `None` on parse error and are filtered from the result list. Never raise unhandled exceptions from parser code.

5. **422 from arithmetic validation vs FastAPI validation**: The `_validate_budget_arithmetic` function raises `HTTPException(422, detail=dict)`. The router's `except HTTPException` block must explicitly handle `status_code == 422 and isinstance(exc.detail, dict)` to return a flat `JSONResponse` (consistent with the 503 handling pattern). FastAPI's own Pydantic validation errors use the standard `{"detail": [...]}` envelope and are not caught by this block.

### Arithmetic Tolerance

All floating-point equality checks use a tolerance of `0.01` EUR (one Euro cent). This is sufficient for EU grant budgets (which are expressed in whole or half Euros in practice) while avoiding false positives from floating-point rounding. The tolerance is defined as a module-level constant `_ARITHMETIC_TOLERANCE = 0.01` in `grants_service.py` to allow easy adjustment.

### Test Fixture Arithmetic

The `MOCK_SOLO_BUDGET_RESPONSE` fixture is pre-verified to be arithmetically consistent:
- Line items: 750000 + 200000 + 50000 + 100000 + 50000 + 287500 = **1 437 500** ✓ matches `total_budget`
- Overhead: 1 150 000 × 0.25 = **287 500** ✓ matches `overhead_amount`; base = sum of direct categories = 750000+200000+50000+100000+50000 = 1 150 000 ✓
- Co-financing: 1 150 000 + 287 500 = **1 437 500** ✓ matches `total_requested_funding`

The `MOCK_CONSORTIUM_BUDGET_RESPONSE` partner subtotals sum: 750 000 + 750 000 + 812 500 = **2 312 500** ✓ matches `total_budget`.

Both fixtures are designed to pass all arithmetic validation checks so `TestAC1AC2HappyPath` and `TestAC5ConsortiumBudget` happy-path tests do not produce false 422s.

### Agent URL Convention

Following the pattern established in S11.03 and S11.04, the `AiGatewayClient.run_agent("budget-builder", payload)` call resolves to `POST {AIGW_BASE_URL}/agents/budget-builder/run`. The test constant `AIGW_BUDGET_URL` must be set to `{AIGW_TEST_BASE_URL}/agents/budget-builder/run` for `respx` interception to work correctly.

### Test Coverage Mapping

| Test Class | Epic Test IDs Covered |
|------------|----------------------|
| `TestAC1AC2HappyPath` | E11-P0-009 (smoke gate — budget-builder agent mock works in CI) |
| `TestAC4ArithmeticValidation` | E11-P0-005 (line-item sum + co-financing sum + overhead + per-partner sum assertions) |
| `TestAC5ConsortiumBudget` | E11-P1-002 (per-partner breakdown present when consortium_size > 1) |
| `TestAC6AgentErrorHandling` | E11-P0-008 (structured 503 on agent unavailable), E11-P0-010 (30s timeout enforced) |
| `TestAC2OptionalParams` | E11-P2-001 (missing optional params → defaults or clear validation error) |

## Dev Agent Record

### Implementation Plan

Extended the existing `/grants` resource group from S11.04 with a new stateless endpoint. Three files edited, one new test file promoted to GREEN phase.

### Completion Notes

**Date:** 2026-04-09

**Tasks implemented:**

1. **Task 1 — Schemas** (`src/client_api/schemas/grants.py`): Appended 6 new Pydantic v2 models: `CostCategory`, `OverheadCalculation`, `CoFinancingSplit`, `PartnerBudget`, `BudgetBuilderRequest`, `BudgetBuilderResponse`. All use `ConfigDict(extra="ignore")` for forward-compatible parsing.

2. **Task 2 — Service** (`src/client_api/services/grants_service.py`): Added `_ARITHMETIC_TOLERANCE = 0.01` constant, `_parse_cost_category()`, `_parse_partner_budget()`, `_validate_budget_arithmetic()`, and `build_budget()` async function. All parser helpers are defensive (.get() with defaults, None-filtered). Arithmetic validation covers all 4 invariants from AC4.

3. **Task 3 — Endpoint** (`src/client_api/api/v1/grants.py`): Added `POST /budget-builder` route using `get_current_user` (no `require_role`) for all-roles access (AC7). Exception handler converts 503/422 HTTPException with dict detail into flat JSONResponse (consistent with S11.04 pattern).

4. **Task 4 — Tests** (`tests/api/test_budget_builder.py`): Removed all 28 `@pytest.mark.skip` RED PHASE decorators. Updated docstring to GREEN PHASE. Fixed 2 E501 ruff violations in pre-existing comment lines. All 27 tests pass.

**Test results:** 27/27 budget builder tests pass; 16/16 grant eligibility regression tests pass. Ruff clean. Mypy clean.

### File List

- `eusolicit-app/services/client-api/src/client_api/schemas/grants.py` — modified (appended 6 schema classes)
- `eusolicit-app/services/client-api/src/client_api/services/grants_service.py` — modified (appended 4 functions + imports)
- `eusolicit-app/services/client-api/src/client_api/api/v1/grants.py` — modified (added 1 endpoint + imports)
- `eusolicit-app/services/client-api/tests/api/test_budget_builder.py` — modified (removed skip decorators → GREEN phase)

### Change Log

- 2026-04-09: Implemented Story 11.5 — Budget Builder Agent Integration. Added `POST /api/v1/grants/budget-builder` stateless endpoint with arithmetic validation, per-partner breakdown enforcement, AI Gateway error handling, and full ATDD test coverage (27 tests, all passing).
