# Story 10.10: Bid/No-Bid Decision API & AI Integration

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-10-bid-no-bid-decision-api-ai-integration
- **Points:** 5
- **Type:** backend
- **Module:** Client API (`client_api.api.v1.bid_decisions` (new), `client_api.models.bid_decision` (new), `client_api.schemas.bid_decisions` (new), `client_api.services.bid_decision_service` (new), new Alembic migration `044_bid_decisions_runtime_columns`) + AI Gateway agent registry (`services/ai-gateway/config/agents.yaml` — new `bid-no-bid-decision` entry).
- **Priority:** P1 (Epic 10 AI-assisted decision-support surface — unblocks Story 10.11 Bid Outcome & Lessons Learned Integration, which depends on a persisted `bid_decisions` row per opportunity; also the backend contract for Story 10.16's Bid/No-Bid Decision UI radar chart and override form).
- **Depends On:** Story 11.3 (`AiGatewayClient` + `AiGatewayTimeoutError` / `AiGatewayUnavailableError` + `get_ai_gateway_client` — the shared E04 HTTP client and its error contract E11-R-001), Story 1.3 / Migration 011 (the analytics-source `client.bid_decisions` stub table this story upgrades in-place with runtime columns), Story 2.4 (`get_current_user` + `CurrentUser`), Story 2.10 (`require_role` company-role gate — `admin` / `bid_manager` allow-list), Story 2.11 (`audit_service.write_audit_entry`), Story 6.1 / 6.5 (the `pipeline.opportunities` read-only Core Table and the `opportunity_type` / `deadline` / `cpv_codes` / `budget_max` fields used to build the agent payload), Story 7.1 (`client.proposals` + `proposals.status` — historical win/loss context join), Story 4.2 / 4.3 (`agents.yaml` registry — this story adds the `bid-no-bid-decision` logical name), Story 11.3 `AiGatewayClient.run_agent` contract (sync agent invocation — no SSE streaming for this story).

## Story

As a **company admin or bid_manager evaluating a newly surfaced opportunity**,
I want **to run an AI-assisted bid/no-bid analysis that scores the opportunity across five dimensions (strategic_fit, technical_capability, financial_viability, competitive_position, resource_availability), record my final decision alongside any override justification, and read back both the AI recommendation and the recorded decision on demand**,
so that **my team has a consistent, auditable decision-support record before committing preparation effort — rather than ad-hoc spreadsheets or gut-feel choices — and the persisted decision feeds downstream analytics (Story 12.2 Market Intelligence), the outcome recording flow (Story 10.11), and the approval pipeline UI (Story 10.16).**

## Description

This story adds the runtime backend for EU Solicit's bid/no-bid decision surface. It upgrades the existing analytics-source stub `client.bid_decisions` table (created read-only in Migration 011 with only 5 columns: `id`, `company_id`, `opportunity_id`, `decision`, `created_at` — it backs `mv_market_intelligence`) into a fully-featured per-opportunity decision record by adding runtime columns via Migration 044 without breaking the existing `mv_market_intelligence` materialized view. It ships a thin service module that calls the E04 AI Gateway's new `bid-no-bid-decision` agent, and exposes three endpoints:

1. `POST /api/v1/opportunities/{opportunity_id}/bid-decision/evaluate` — gather opportunity + company + historical context, call the agent, store the returned scorecard under `ai_recommendation`, and return it to the caller.
2. `POST /api/v1/opportunities/{opportunity_id}/bid-decision` — record the final human decision (`bid` / `no_bid` / `conditional`) with optional override justification.
3. `GET /api/v1/opportunities/{opportunity_id}/bid-decision` — return the current persisted row (scorecard + decision, if any).

Because Migration 011's table already exists in production databases, this story does NOT drop and recreate — it runs `ALTER TABLE ... ADD COLUMN` statements and reshapes the `decision` column's domain via a named CHECK constraint (replacing the previous implicit `VARCHAR(20)` with an enumerated domain). Existing rows created by any pre-Story-10.10 seed fixtures are backfilled with sensible defaults: `decision='pending'`, `ai_recommendation=NULL`, `user_override=FALSE`, etc.

Seven design decisions worth calling out:

**(a) One row per `(company_id, opportunity_id)` — upsert semantics.** The bid/no-bid decision is a per-opportunity-per-company artefact. The runtime contract is that re-evaluating is idempotent: a second `POST /evaluate` replaces `ai_recommendation` + `evaluated_at` on the existing row; a subsequent `POST /bid-decision` updates `decision` + `decided_by` + `decided_at` + override fields. Implemented via a DB-level unique constraint `uq_bid_decisions_company_opportunity` on `(company_id, opportunity_id)` and an application-level `INSERT ... ON CONFLICT (company_id, opportunity_id) DO UPDATE` pattern (or SQLAlchemy's `postgresql.insert(...).on_conflict_do_update`). Re-evaluation does NOT reset `decision` / `decided_by` / `decided_at` — the human decision persists across re-runs of the scorecard (AC4 / AC5 spell this out).

**(b) Evaluate and decide are separate endpoints.** The epic spec allows them to coexist ("the user can override with justification"). Splitting them mirrors the UI flow in Story 10.16: the user first triggers evaluation, reviews the radar chart, then submits their decision (possibly overriding the AI's suggested verdict). Collapsing both into a single endpoint would force the UI to either submit empty decisions just to see the scorecard or to re-fetch after an async evaluation — worse UX and a fuzzier audit trail. The decoupling also lets tests mock the agent call in isolation without entangling decision-recording logic.

**(c) `user_override` is computed server-side, not trusted from the client.** The request body for `POST /bid-decision` accepts `{decision, override_justification?}`. The server compares the caller's decision to the AI recommendation's suggested verdict (derived from the scorecard's top-level `recommendation` field: `bid | no_bid | conditional`) and sets `user_override = True` iff they differ. This keeps the override flag honest — a client cannot silently mark a decision as "aligned with AI" when it contradicts the scorecard. If no `ai_recommendation` exists on the row (pure human decision, no prior evaluation), `user_override` is `False`.

**(d) Override justification is required when `user_override = True`.** Validated in a Pydantic `model_validator(mode="after")` that runs the same server-side override detection: if the caller's decision disagrees with the stored AI verdict, `override_justification` must be non-empty after `.strip()` and ≤ 5000 chars. Rejection at validation time returns 422 with detail `"Override justification is required when your decision differs from the AI recommendation"`. For decisions aligned with the AI (or absent an AI recommendation), `override_justification` is optional and persisted as-is if provided.

**(e) The AI agent is `bid-no-bid-decision` — registry entry added in this story.** The agent is not yet in `services/ai-gateway/config/agents.yaml`. Story 10.10 adds a new entry with `type: agent`, a placeholder `kraftdata_id` UUID (to be rotated per E04-R-009 before production deployment), `description: "Scores a bid opportunity across five strategic dimensions and recommends bid / no_bid / conditional"`, and `timeout_override: 120` (scorecard generation can touch multiple historical proposals and run ~60–90 s at the KraftData backend; keep headroom). The logical name `bid-no-bid-decision` is what `AiGatewayClient.run_agent("bid-no-bid-decision", payload)` will resolve to.

**(f) Agent payload shape — explicit contract, not a passthrough.** The service constructs a structured payload from three sources:
- `opportunity` — loaded from `pipeline.opportunities` via the existing Core Table (`client_api.models.pipeline_opportunity.opportunities_table`). Fields: `id`, `title`, `description`, `opportunity_type`, `deadline`, `budget_min`, `budget_max`, `currency`, `country`, `cpv_codes`, `contracting_authority`, `evaluation_criteria` (JSONB), `mandatory_documents` (JSONB). Serialised to JSON with UUIDs as strings and datetimes as ISO-8601.
- `company_profile` — loaded from `client.companies` for `current_user.company_id`. Fields: `id`, `name`, `description`, `cpv_sectors`, `address` (JSONB). Excludes internal fields (Stripe IDs, billing metadata).
- `historical_context` — derived from `client.bid_outcomes` + `client.proposals` for the caller's company. A lightweight rollup: `{win_count: int, loss_count: int, withdrawn_count: int, similar_opportunities_count: int}` where "similar" means "same opportunity_type AND at least one overlapping `cpv_codes` entry" (a best-effort signal; if `cpv_codes` overlap is expensive, fall back to `opportunity_type` only and log a `bid_decision.historical_context.fallback` breadcrumb). Bounded to the last 24 months of `bid_outcomes.created_at` to keep the payload compact. If the company has no outcome history, all counts are 0 and the payload still includes the key (the agent handles empty history).

The agent is expected to return a JSON object matching `BidDecisionScorecard`:
```
{
  "recommendation": "bid" | "no_bid" | "conditional",
  "confidence": 0.0..1.0,
  "dimensions": {
    "strategic_fit":          {"score": 1..10, "rationale": str},
    "technical_capability":   {"score": 1..10, "rationale": str},
    "financial_viability":    {"score": 1..10, "rationale": str},
    "competitive_position":   {"score": 1..10, "rationale": str},
    "resource_availability":  {"score": 1..10, "rationale": str}
  },
  "summary": str,
  "generated_at": ISO-8601  (optional — server fills with now() if absent)
}
```
The service validates the shape via `BidDecisionScorecard.model_validate(...)`. A malformed agent response is wrapped as 503 `{"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"}` (consistent with the Story 11.3 E11-R-001 error contract) and logged at WARN with the raw response body truncated to 500 chars. No row is inserted/updated on a malformed response.

**(g) Cross-tenant safety follows the existence-leakage pattern.** Opportunity access is not company-owned (opportunities are global in `pipeline.opportunities`) — but the bid_decisions row IS company-scoped. GET returns 404 if no row exists for `(current_user.company_id, opportunity_id)` (not 403 — the existence of another company's decision is not leaked). POST (both endpoints) validate the opportunity exists in `pipeline.opportunities` and is not soft-deleted (`deleted_at IS NULL`); missing opportunity → 404. Cross-company reads/writes are structurally impossible because `company_id` is always derived from `current_user.company_id`.

The router is mounted at `/api/v1/opportunities/{opportunity_id}/bid-decision` and included in `client_api/main.py` beside `opportunities_v1.router`.

## Acceptance Criteria

1. [x] **AC1 — Migration `044_bid_decisions_runtime_columns.py` upgrades the existing `client.bid_decisions` table in place.** Revision `"044"`, `down_revision = "043"`. `upgrade()` MUST:

    - Add `ai_recommendation JSONB NULL` column.
    - Add `user_override BOOLEAN NOT NULL DEFAULT FALSE` column.
    - Add `override_justification TEXT NULL` column (max length enforced at the Pydantic layer, not DB).
    - Add `decided_by UUID NULL` column (soft link to `client.users.id`, no FK — match the Story 10.7 / 10.8 / 10.9 pattern).
    - Add `evaluated_at TIMESTAMPTZ NULL` column.
    - Add `decided_at TIMESTAMPTZ NULL` column.
    - Add `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()` column.
    - Alter the existing `decision` column: set `DEFAULT 'pending'` and backfill any pre-existing NULL or analytics-seed rows to `'pending'` via `UPDATE client.bid_decisions SET decision = 'pending' WHERE decision NOT IN ('bid','no_bid','conditional','pending')` (defensive — the column is currently `VARCHAR(20) NOT NULL` so in practice all rows already have a value, but the analytics seed migration made no promises about the domain).
    - Add named CHECK constraint `ck_bid_decisions_decision` with predicate `decision IN ('bid','no_bid','conditional','pending')`.
    - Add unique constraint `uq_bid_decisions_company_opportunity` on `(company_id, opportunity_id)`. If a duplicate exists at migration time (analytics seed fixtures may have inserted duplicates), the migration MUST fail loudly with a clear message — do NOT attempt to deduplicate silently.
    - Add composite index `ix_bid_decisions_company_opportunity` on `(company_id, opportunity_id)` for the hot GET / upsert path. Unique constraint already creates an index; declare the composite index only if it adds a non-redundant sort order (it does not, so skip — the unique constraint index suffices; call this out in the migration docstring).
    - Add `GRANT SELECT, INSERT, UPDATE, DELETE ON client.bid_decisions TO client_api_role` (the Migration 011 grants were SELECT-only to `notification_role` — add the client_api_role grants required for the runtime write surface). Preserve existing notification_role grants.

    `downgrade()` drops the CHECK constraint, unique constraint, and all seven added columns — restoring the table to the Migration 011 analytics shape. Do NOT drop the table itself (the analytics MV depends on it).

2. [x] **AC2 — `BidDecision` ORM model (`client_api/models/bid_decision.py`).** Declare the full table: columns mirror Migration 011's original 5 columns + the seven AC1 additions. `__table_args__` includes `ck_bid_decisions_decision`, `uq_bid_decisions_company_opportunity`, and `{"schema": "client"}`. No relationships (Proposal / Opportunity / User linkages are soft — match Story 10.7 / 10.8 / 10.9 precedent). Export from `client_api/models/__init__.py` so alembic autogenerate's MetaData sees the full column set on subsequent migrations.

   The model must declare `__tablename__ = "bid_decisions"`. Because the table is also present in the Migration 011 analytics-source allow-list (`alembic/env.py` `_EXCLUDED_TABLE_NAMES`), remove `"bid_decisions"` from that frozenset as part of this story — the table is now ORM-managed end-to-end. Do NOT remove `"bid_outcomes"` or `"bid_preparation_logs"` (those remain analytics-only until Story 10.11).

3. [x] **AC3 — Pydantic schemas (`client_api/schemas/bid_decisions.py`).** Implement:
   - `BidDecisionType(str, Enum)`: `bid | no_bid | conditional`. (The DB CHECK constraint additionally accepts `pending` for rows created by `/evaluate` before any human decision; `pending` is NOT exposed in the request enum — it is only ever set server-side.)
   - `ScorecardDimension(BaseModel)`: `score: int` (ge=1, le=10), `rationale: str` (min_length=1, max_length=2000).
   - `BidDecisionScorecard(BaseModel)`: `recommendation: BidDecisionType`, `confidence: float` (ge=0.0, le=1.0), `dimensions: dict[Literal["strategic_fit","technical_capability","financial_viability","competitive_position","resource_availability"], ScorecardDimension]` — validator asserts all five keys are present and no extras, `summary: str` (max_length=5000), `generated_at: datetime | None = None` (server fills if absent).
   - `BidDecisionEvaluateResponse(BaseModel)`: `opportunity_id: UUID`, `scorecard: BidDecisionScorecard`, `evaluated_at: datetime`. `model_config = ConfigDict(from_attributes=True)`.
   - `BidDecisionRequest(BaseModel)`: `decision: BidDecisionType`, `override_justification: str | None` (max_length=5000). No `user_override` field — server-computed. A `@model_validator(mode="after")` is a no-op at the schema layer (the AI-vs-caller check requires DB state); the service-layer validator enforces the "justification required on override" rule (AC5) and returns 422 via `HTTPException(422, ...)`.
   - `BidDecisionResponse(BaseModel)`: `id: UUID`, `company_id: UUID`, `opportunity_id: UUID`, `decision: str` (the DB value — may be `pending`), `ai_recommendation: BidDecisionScorecard | None`, `user_override: bool`, `override_justification: str | None`, `decided_by: UUID | None`, `evaluated_at: datetime | None`, `decided_at: datetime | None`, `created_at: datetime`, `updated_at: datetime`. `model_config = ConfigDict(from_attributes=True)`.

4. [x] **AC4 — `POST /api/v1/opportunities/{opportunity_id}/bid-decision/evaluate`.** Gated by `Depends(require_role("bid_manager"))` (Story 2.10 gate — both `admin` and `bid_manager` company roles pass; lower roles → 403). No request body.

   Service behaviour (single DB transaction):
   - Load opportunity from `pipeline.opportunities` where `id = :opportunity_id AND deleted_at IS NULL`. Missing → 404 `"Opportunity not found"`.
   - Load company from `client.companies` for `current_user.company_id`. Always present (the current_user chain guarantees this); if missing raise 500.
   - Compute `historical_context` rollup (design decision (f)): `SELECT status, COUNT(*) FROM client.bid_outcomes WHERE company_id = :cid AND created_at >= now() - interval '24 months' GROUP BY status` → win_count / loss_count / withdrawn_count; `similar_opportunities_count` via a join on `pipeline.opportunities` filtered by overlapping `cpv_codes` or same `opportunity_type` fallback.
   - Build agent payload per design decision (f) — strictly JSON-serialisable dict.
   - Call `gw_client.run_agent("bid-no-bid-decision", payload)` via the Story 11.3 `AiGatewayClient`. On `AiGatewayTimeoutError` or `AiGatewayUnavailableError`: raise `HTTPException(503, detail={"message": "AI features are temporarily unavailable. Please try again.", "code": "AGENT_UNAVAILABLE"})`. No row is inserted/updated. Log at WARN with `opportunity_id`, `company_id`, exception class.
   - Parse the response via `BidDecisionScorecard.model_validate(response)`. On `ValidationError`: raise the same 503 body; log `bid_decision.scorecard.malformed` at WARN with raw response truncated to 500 chars.
   - Upsert the `bid_decisions` row (AC6 specifies the exact clause) with `ai_recommendation = scorecard.model_dump(mode="json")`, `evaluated_at = now()`, `updated_at = now()`. Preserve `decision`, `decided_by`, `decided_at`, `user_override`, `override_justification` on update (do NOT reset them — see design decision (a)). On insert, set `decision = 'pending'`.
   - Write one audit row: `entity_type="bid_decision"`, `action_type="create"` on first insert OR `"update"` on upsert, `entity_id=row.id`, `after={"opportunity_id": str, "ai_recommendation_recommendation": scorecard.recommendation.value, "evaluated_at": iso}`.
   - Commit (the `get_db_session` dep handles this).
   - Return `BidDecisionEvaluateResponse(opportunity_id=..., scorecard=scorecard_with_generated_at, evaluated_at=row.evaluated_at)` with HTTP 200.

5. [x] **AC5 — `POST /api/v1/opportunities/{opportunity_id}/bid-decision`.** Gated by `Depends(require_role("bid_manager"))` — same allow-list as AC4.

   Body: `BidDecisionRequest`. Service behaviour (single DB transaction):
   - Validate opportunity exists and is not soft-deleted (same query as AC4). Missing → 404.
   - Fetch the existing `bid_decisions` row for `(current_user.company_id, opportunity_id)` if any.
   - **Override detection (design decision (c)):** If a row exists AND `row.ai_recommendation IS NOT NULL`, extract `ai_verdict = row.ai_recommendation["recommendation"]`. Compare to `request.decision.value`. If different → `user_override = True`; else `user_override = False`. If no row yet OR `ai_recommendation IS NULL` → `user_override = False` regardless.
   - **Override justification validation (design decision (d)):** If `user_override == True` AND `request.override_justification` is None or strips to empty → raise `HTTPException(422, detail="Override justification is required when your decision differs from the AI recommendation")`. If `override_justification` is provided but `user_override == False`, persist it as-is (no error; the UI may legitimately send a rationale for an AI-aligned decision).
   - Upsert (AC6): set `decision = request.decision.value`, `decided_by = current_user.user_id`, `decided_at = now()`, `user_override = computed_flag`, `override_justification = request.override_justification`, `updated_at = now()`. Preserve `ai_recommendation` and `evaluated_at` on update.
   - Write one audit row: `entity_type="bid_decision"`, `action_type="create"` on insert OR `"update"` on upsert, `entity_id=row.id`, `after={"opportunity_id": str, "decision": decision_value, "user_override": bool, "decided_by": str(user_id)}`.
   - Return 201 on insert / 200 on update with `BidDecisionResponse` (the full persisted row, scorecard re-validated via `BidDecisionScorecard` if non-null else `None`).

6. [x] **AC6 — Upsert semantics via `ON CONFLICT (company_id, opportunity_id) DO UPDATE`.** Both AC4 and AC5 service functions MUST use `from sqlalchemy.dialects.postgresql import insert as pg_insert; stmt = pg_insert(BidDecision).values(...).on_conflict_do_update(index_elements=["company_id","opportunity_id"], set_={...}).returning(BidDecision)`. The `set_` dict on AC4 includes `ai_recommendation`, `evaluated_at`, `updated_at` ONLY — decision/override/decided_* are untouched. The `set_` dict on AC5 includes `decision`, `decided_by`, `decided_at`, `user_override`, `override_justification`, `updated_at` ONLY — ai_recommendation/evaluated_at are untouched. Write tests for each field's preserve-on-update behaviour (AC10).

7. [x] **AC7 — `GET /api/v1/opportunities/{opportunity_id}/bid-decision`.** Gated by `Depends(require_role("bid_manager"))` — same allow-list (viewing a bid decision is not more privileged than making one; there is no "read_only viewer" for this endpoint in Epic 10). Service behaviour:
   - Validate opportunity exists AND `deleted_at IS NULL` in `pipeline.opportunities`; missing → 404.
   - Load `bid_decisions` row for `(current_user.company_id, opportunity_id)`. If none → 404 `"No bid decision recorded for this opportunity"` (NOT 204 / empty — the distinct 404 lets the UI render an empty state clearly).
   - Return 200 with `BidDecisionResponse` (`ai_recommendation` is the stored JSONB re-validated via `BidDecisionScorecard.model_validate(row.ai_recommendation)` if non-null else `None`).

8. [x] **AC8 — Agent registry entry in `services/ai-gateway/config/agents.yaml`.** Add a new YAML key `bid-no-bid-decision`:
    ```yaml
    bid-no-bid-decision:
      kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000030"
      type: agent
      description: "Scores a bid opportunity across five strategic dimensions (strategic_fit, technical_capability, financial_viability, competitive_position, resource_availability) and recommends bid / no_bid / conditional"
      timeout_override: 120
    ```
    The UUID is a placeholder and MUST be rotated per E04-R-009 before production deployment (same caveat as the existing 29 entries). Run the agent_registry's duplicate-key check via the existing startup path — no code changes to the registry loader required; the entry is additive.

    Add a corresponding entry to the Story 4.3 agent registry spec tests (search `tests/unit/test_agent_registry.py` in `services/ai-gateway/tests/` for the list of logical names asserted present; include `bid-no-bid-decision` there if the test exists and enumerates names).

9. [x] **AC9 — Router wiring + `main.py` include.** Add `services/client-api/src/client_api/api/v1/bid_decisions.py` with router `prefix="/opportunities/{opportunity_id}/bid-decision"`, `tags=["bid-decisions"]`. Register three routes:
   - `POST /evaluate` → `BidDecisionEvaluateResponse` (200)
   - `POST ""` → `BidDecisionResponse` (201 / 200)
   - `GET ""` → `BidDecisionResponse` (200)

   Import as `from client_api.api.v1 import bid_decisions as bid_decisions_v1` in `client_api/main.py` and include via `api_v1_router.include_router(bid_decisions_v1.router)` beside `opportunities_v1`. Verify resulting paths:
   - `POST /api/v1/opportunities/{opportunity_id}/bid-decision/evaluate`
   - `POST /api/v1/opportunities/{opportunity_id}/bid-decision`
   - `GET  /api/v1/opportunities/{opportunity_id}/bid-decision`

   Write one `test_router_no_doubled_prefix` test (mirrors Story 10.9's pattern) that inspects `app.routes` and asserts no `/opportunities/opportunities/` double-prefix.

10. [x] **AC10 — Unit and API tests (≥ 28 scenarios).** Split between `services/client-api/tests/api/test_bid_decisions.py` (router + HTTP-layer) and `services/client-api/tests/unit/test_bid_decision_service.py` (service-layer + schema validator + upsert semantics). Follow Story 11.3's `respx.MockRouter` fixture pattern for AI Gateway mocks. No browser tests required (backend story).

    **Migration (AC1) — integration:**
    - `test_migration_044_adds_runtime_columns` — live DB; all seven columns present with correct types + nullability.
    - `test_migration_044_check_constraint_rejects_invalid_decision` — `INSERT ... decision='maybe'` raises `IntegrityError`.
    - `test_migration_044_unique_constraint_on_company_opportunity` — duplicate `(company_id, opportunity_id)` raises `IntegrityError`.
    - `test_migration_044_downgrade_reverses` — downgrade removes the seven columns + constraints, leaving the Migration 011 shape.
    - `test_migration_044_revision_chain` — `revision == "044"`, `down_revision == "043"`.

    **ORM + schema (AC2, AC3) — unit:**
    - `test_orm_bid_decision_importable` — `from client_api.models import BidDecision` succeeds; schema is `client`.
    - `test_orm_bid_decision_removed_from_env_excluded` — `"bid_decisions"` NOT in `alembic/env.py::_EXCLUDED_TABLE_NAMES`.
    - `test_schema_bid_decision_type_enum` — `bid | no_bid | conditional`; `pending` NOT in `BidDecisionType`.
    - `test_schema_scorecard_requires_all_five_dimensions` — missing any key → `ValidationError`.
    - `test_schema_scorecard_score_range_1_to_10` — score=0 or score=11 → `ValidationError`.
    - `test_schema_scorecard_confidence_0_to_1` — confidence=-0.1 or 1.1 → `ValidationError`.
    - `test_schema_request_override_justification_max_5000` — 5001 chars → `ValidationError`.

    **Happy paths (AC4, AC5, AC7) — API:**
    - `test_evaluate_success_returns_200_with_scorecard` — mock gateway success → POST evaluate → 200; response has `opportunity_id`, `scorecard.recommendation`, `evaluated_at`; DB has one `bid_decisions` row with `decision='pending'`, `ai_recommendation` JSONB populated.
    - `test_evaluate_agent_called_with_correct_payload` — inspect `respx` recorded request body; assert `opportunity`, `company_profile`, `historical_context` keys present with values from DB fixtures.
    - `test_decide_without_prior_evaluation_persists_decision` — POST decide on an opportunity with no prior `/evaluate` → 201; row has `decision='bid'`, `ai_recommendation IS NULL`, `user_override=False`, `decided_by=current_user`.
    - `test_decide_aligned_with_ai_no_override` — seed row with `ai_recommendation.recommendation='bid'`; POST decide `decision='bid'` → 200/201; `user_override=False`; justification optional.
    - `test_decide_against_ai_requires_justification` — seed row with `ai_recommendation.recommendation='bid'`; POST decide `decision='no_bid'` without `override_justification` → 422 `"Override justification is required ..."`; no row update.
    - `test_decide_against_ai_with_justification_succeeds` — seed row with `ai_recommendation.recommendation='bid'`; POST decide `decision='no_bid', override_justification='Client pulled out'` → 200; `user_override=True`; `override_justification` persisted.
    - `test_decide_preserves_ai_recommendation_on_update` — seed row with ai_recommendation populated + evaluated_at set; POST decide → 200; verify `ai_recommendation` unchanged, `evaluated_at` unchanged; `decision` / `decided_at` updated.
    - `test_evaluate_preserves_decision_on_re_run` — seed row with `decision='bid'`, `decided_by=user_x`; POST evaluate → 200; scorecard updated, but `decision` / `decided_by` / `decided_at` / `user_override` preserved.
    - `test_get_returns_200_with_full_row` — seed fully-populated row; GET → 200 matches `BidDecisionResponse` shape.
    - `test_get_no_row_returns_404` — opportunity exists but no decision row → 404 `"No bid decision recorded for this opportunity"`.

    **Gate failures (AC4, AC5, AC7):**
    - `test_evaluate_non_existent_opportunity_returns_404` — unknown opportunity_id → 404.
    - `test_decide_soft_deleted_opportunity_returns_404` — opportunity with `deleted_at` non-null → 404 on both POST decide and POST evaluate.
    - `test_evaluate_unauthenticated_returns_401`.
    - `test_evaluate_technical_writer_role_returns_403` — non-admin / non-bid_manager → 403.
    - `test_get_cross_company_returns_404` — Company B user GETs a decision owned by Company A → 404 (existence-leakage safe; the row exists but for a different company_id).
    - `test_decide_invalid_decision_value_returns_422` — `decision='maybe'` → 422 Pydantic enum.
    - `test_decide_override_justification_over_5000_chars_returns_422`.

    **Agent failure handling (AC4):**
    - `test_evaluate_gateway_timeout_returns_503` — mock `respx` raising `httpx.TimeoutException` → 503 body `{"message": "...", "code": "AGENT_UNAVAILABLE"}`; no row inserted/updated.
    - `test_evaluate_gateway_503_returns_503_with_standard_body`.
    - `test_evaluate_gateway_malformed_response_returns_503` — agent returns a dict missing `dimensions` → 503; row NOT updated; WARN log captured.
    - `test_evaluate_no_row_created_on_gateway_failure` — pre-call row count; mock timeout; post-call count unchanged.
    - `test_evaluate_existing_row_ai_recommendation_not_mutated_on_failure` — seed row with ai_recommendation=X; mock timeout on re-evaluate; verify row.ai_recommendation still X.

    **Upsert (AC6) — unit:**
    - `test_service_upsert_evaluate_uses_on_conflict_do_update` — inspect compiled SQL (or mock the execute() call) and assert `ON CONFLICT` clause targets `(company_id, opportunity_id)` and `set_` contains only the evaluate fields.
    - `test_service_upsert_decide_uses_on_conflict_do_update` — same pattern for the decide fields.
    - `test_service_similar_opportunities_fallback_on_error` — historical-context helper when `cpv_codes` join errors → fallback to `opportunity_type` only; `bid_decision.historical_context.fallback` breadcrumb logged.

    **Audit trail:**
    - `test_evaluate_writes_audit_row` — one `shared.audit_log` row with `entity_type="bid_decision"`, `action_type="create"` or `"update"`, `after` contains `opportunity_id`, `ai_recommendation_recommendation`, `evaluated_at`.
    - `test_decide_writes_audit_row` — one row with `action_type` reflecting insert vs update, `after` contains `opportunity_id`, `decision`, `user_override`, `decided_by`.

    **Router wiring (AC9):**
    - `test_router_paths_exist` — all three paths resolve (non-404 for a well-formed but unauthenticated request).
    - `test_router_no_doubled_prefix` — inspect `app.routes`; assert no `/opportunities/opportunities/` double-prefix.

## Dev Notes

- **Test Expectations (from Epic-level test design):** `eusolicit-docs/test-artifacts/test-design-epic-10.md` is **not** available — Epic 10 was never epic-test-designed (the `test-artifacts/` directory for Epic 10 contains only per-story ATDD checklists 10-1 through 10-9; no epic-level design doc exists; same gap noted in Story 10.9). Coverage strategy therefore follows the established patterns: `httpx.AsyncClient` + `async with AsyncSession` for API tests; `pytest-asyncio` with `asyncio_mode="auto"`; cross-company isolation returns 404 (NOT 403) for existence-leakage safety; AI Gateway mocks use `respx.MockRouter` per Story 11.3's `test_espd_autofill_export.py` fixture pattern. Use the arrange/act/assert block format from `tests/api/test_approvals.py` (Story 10.9) and `tests/api/test_espd_autofill_export.py` (Story 11.3). For DB-backed migration tests, mirror Story 10.9's `tests/integration/test_migration_043.py` structure — tests marked with an `integration` mark and skipped in unit-only runs. The ATDD RED-phase skip markers (`@pytest.mark.skip(reason="RED PHASE: ...")`) are expected to be applied up-front so the entire test suite compiles green at story-creation time; they come off phase-by-phase per the TDD activation sequence.

- **AI Gateway client reuse — no new dependency code.** Import `AiGatewayClient`, `AiGatewayTimeoutError`, `AiGatewayUnavailableError`, `get_ai_gateway_client` from `client_api.services.ai_gateway_client` (Story 11.3 artefact). Do NOT re-implement the client, retry policy, or timeout handling in this story — the E11-R-001 error contract is shared across all client-api AI integrations.

- **Story 10.11 coupling.** Story 10.11 (Bid Outcome & Lessons Learned Integration) assumes a `bid_decisions` row already exists for `(company_id, opportunity_id)` when outcome recording happens. Our insert-on-first-call pattern (either POST evaluate or POST decide bootstraps the row) guarantees that invariant. Story 10.11's outcome recording is free to reference `bid_decisions.id` for traceability joins.

- **Analytics MV compatibility (Migration 011 source preservation).** The `mv_market_intelligence` materialized view defined in Migration 011 joins `client.bid_decisions bd ON bd.opportunity_id = o.id` and groups by `bd.company_id`. None of our added columns nor the CHECK constraint affects this view — the added columns are simply not referenced. The view refreshes unaffected. Run `REFRESH MATERIALIZED VIEW CONCURRENTLY client.mv_market_intelligence` post-migration is NOT required (MVs are stable across column additions on source tables); document in the migration header that no MV refresh is needed.

- **Why extend, not replace.** An earlier draft considered dropping `client.bid_decisions` and recreating it as a "real" runtime table. Rejected because: (i) Migration 011 is already applied in production and the analytics MV is live and depends on the table being stable; (ii) a drop-and-create would require deprecating the MV's `bd.company_id` / `bd.opportunity_id` references for the duration of the migration (a cross-service breakage window); (iii) the analytics columns (`id`, `company_id`, `opportunity_id`, `decision`, `created_at`) are exactly the columns we want as the stable primary key + identity — Migration 011 effectively pre-seeded the shape we need. Strictly additive `ALTER TABLE ADD COLUMN` is the low-risk path.

- **`pending` as a decision value.** The enum includes `pending` at the DB / ORM / response layer but not in `BidDecisionType` (the request enum). Reason: `pending` is a server-only sentinel meaning "the row exists because `/evaluate` was called, but no human decision has been recorded yet." It lets the GET endpoint return a 200 with a meaningful `decision` value rather than a contextual null. A client that wants to treat `pending` as "no decision yet" can check `decided_at IS None`. The CHECK constraint permits `pending` so Migration 044's re-POST-insert path (`ON CONFLICT ... INSERT`) does not need to override the default.

- **Override-flag computation is NOT retroactive.** If a user decides first (no prior `/evaluate`), then later runs `/evaluate`, the row's `user_override` flag remains `False` — it was honest at decision time. Running `/evaluate` AFTER a decision does NOT re-assess override alignment. Product-wise this is the right call: the AI recommendation is a forward-looking decision-support artefact, not a retroactive judge. Document this in the endpoint docstring.

- **Historical-context payload performance.** The `similar_opportunities_count` with `cpv_codes` overlap is a PostgreSQL `array_length(array_intersect(...), 1) > 0` check. If the opportunity count is large (> 10k) the query may be slow; apply a `LIMIT 1000` on the inner `pipeline.opportunities` scan since the downstream agent only needs an order-of-magnitude signal. Log the true count only if it would have exceeded the LIMIT (so the fallback decision is observable).

- **AI Gateway agent timeout tuning.** Start at `timeout_override: 120` seconds (matches `executive-summary` / `tender-summarizer`). If P95 latency at scale is lower, trim in a follow-up config-only change — no code impact. If timeouts are excessive, the E11-R-001 contract already converts them to 503 at the client-api boundary.

- **Out of scope:**
  - Event emission for bid/no-bid decisions (Epic 10 does not specify a `BidDecisionMade` event in `eusolicit_models.events.py`; adding one would require a Story 9.10-style notification-consumer story). If a product need emerges, add `BidDecisionMade(BaseEvent)` to the schema and introduce a post-commit emitter in a follow-up story. This story's audit-log row is the durable decision record.
  - Streaming the AI scorecard via SSE (the agent is invoked synchronously — `run_agent`, not `stream_agent`; scorecards are short enough to fit a single JSON payload within the 120 s timeout).
  - Overriding the AI recommendation with rationale that references external documents (justification is free-text only; attachments are Story 10.11 / 7.10 territory).
  - Bulk evaluation across multiple opportunities (the agent supports one opportunity per call; bulk UI is a future story).
  - Frontend UI (Story 10.16 — radar chart + override form).
  - Recording bid outcome / lessons learned (Story 10.11).
  - Any changes to `client.bid_outcomes` or `client.bid_preparation_logs` beyond read-only joins for the historical context helper (Story 10.11 owns those).

- **Future-compatibility note (Story 12.2 Market Intelligence):** Story 12.2's dashboard reads from `mv_market_intelligence`, which groups by `bd.company_id` and `o.cpv_codes[1]`. Our new `decision IN ('bid','no_bid','conditional','pending')` values MAY warrant a future MV filter (e.g. `WHERE bd.decision = 'bid'` to count only "bids pursued" rather than all opportunities considered). This story does NOT alter the MV — Story 12.2 can decide whether to filter. The shape of `bid_decisions` remains stable for all decision values.

- **Router include pattern note (AC9):** the router prefix `/opportunities/{opportunity_id}/bid-decision` is fully embedded in `APIRouter(prefix=...)`, so `include_router` takes no additional prefix (same pattern as Story 10.9's `/proposals/{proposal_id}/approvals`). The `POST ""` / `GET ""` routes intentionally register the bare prefix as the path so `/bid-decision` (not `/bid-decision/`) is the canonical URL. Confirm FastAPI's `redirect_slashes=True` default does not cause a 307 loop in tests; if it does, register the path explicitly as `/` and document.

## Change Log

- 2026-04-21: Initial story context created. (bmad-create-story)

## Known Deviations

### Detected by `2-dev-story` at 2026-04-21T02:12:25Z

- The pre-written ATDD tests (`test_bid_decisions.py`) had several environment-specific issues including incorrect role usage (`technical_writer` in `company_memberships`), lack of `superuser` privileges for seeding `pipeline.opportunities`, and missing required columns (`source_id`, `source_type`) in seed SQL. These were fixed to ensure test suite integrity. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_

### Detected by `3-code-review` at 2026-04-21T02:19:06Z (session dd012bad-e157-449b-b979-82e7bfd974e5)

- `POST /api/v1/opportunities/{opportunity_id}/bid-decision` always returns 201, never 200 on update, contradicting AC5's explicit 201-on-insert / 200-on-update contract. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_
- `build_historical_context` primary `similar_opportunities_count` query uses `opportunity_type OR cpv_codes &&` instead of the `AND` semantic specified in design-decision (f); the documented fallback branch only runs on exception. _(type: `ARCHITECTURAL_DRIFT`; severity: `deferrable`)_
- `POST /api/v1/opportunities/{opportunity_id}/bid-decision` always returns 201, never 200 on update, contradicting AC5's explicit 201-on-insert / 200-on-update contract. _(type: `CONTRADICTORY_SPEC`)_
- `build_historical_context` primary `similar_opportunities_count` query uses `opportunity_type OR cpv_codes &&` instead of the `AND` semantic specified in design-decision (f); the documented fallback branch only runs on exception. _(type: `CONTRADICTORY_SPEC`; severity: `deferrable`)_

## Senior Developer Review

### Reviewed by `3-code-review` at 2026-04-21

**Verdict: Changes Requested**

The core design is sound: schema/model/migration/router layering matches the spec, upsert semantics are correctly partitioned between evaluate (touches `ai_recommendation`/`evaluated_at`) and decide (touches `decision`/`decided_by`/`decided_at`/override fields), existence-leakage-safe 404s are implemented consistently, and gateway failure handling returns the shared E11-R-001 503 contract without mutating state. Tests are broad and hit the important cross-company and override-detection edges.

However, two implementation-level deviations from the acceptance criteria must be corrected before approval, plus one fragility concern:

#### Blocking findings

1. **AC5 violated — `POST /bid-decision` always returns 201, never 200.**
   `services/client-api/src/client_api/api/v1/bid_decisions.py:60` hardcodes `status_code=status.HTTP_201_CREATED` on the decorator. AC5 explicitly says *"Return 201 on insert / 200 on update."* The service already distinguishes insert vs. update for the audit row; the route needs to differentiate the HTTP status too. Recommended fix: have the service return a tuple `(row, was_insert)` (or set a flag on the service result), then in the route return `JSONResponse(content=BidDecisionResponse.model_validate(row).model_dump(mode="json"), status_code=201 if was_insert else 200)`. Several tests already assert `status_code in (200, 201)`, which masks the bug — add a dedicated "update returns 200, insert returns 201" test to pin the contract once fixed.

2. **Deviation from AC design-decision (f): `similar_opportunities_count` uses OR instead of AND, and no fallback breadcrumb is ever emitted by the primary path.**
   `services/client-api/src/client_api/services/bid_decision_service.py:122-125` builds the primary query as `o.opportunity_type = :opp_type OR o.cpv_codes && :cpv_codes`. The story specifies *"same opportunity_type **AND** at least one overlapping `cpv_codes` entry"* with a fallback to "opportunity_type only" (and a `bid_decision.historical_context.fallback` log) only when the CPV overlap check is expensive or errors. The current implementation is strictly broader than either mode — it will consistently overcount. The fallback branch only fires on an exception (which is rare for a well-typed text[]/&& check), so the breadcrumb is effectively dead code in practice. Fix: change the primary predicate to `AND`, keep the fallback on exception as written. _(type: `ARCHITECTURAL_DRIFT`; severity: `deferrable`)_

#### Non-blocking issues worth addressing

3. **`action = "create" / "update"` detection via `row.created_at >= now - timedelta(seconds=1)` is racy.** `bid_decision_service.py:282,362`. On a fast DB cycle this works, but it is a time-based heuristic on a row just read back from `RETURNING` and will misclassify updates that happen within one second of row creation (possible in fixtures and dev loops). Prefer `RETURNING (xmax = 0) AS inserted` via `stmt.returning(BidDecision, literal_column("xmax = 0").label("inserted"))`, or fetch `existing_row` before the upsert (decide already does this — reuse that for the audit action and for AC5's 201/200 split). Also unifies the create/update signal with finding #1.

4. **Timezone hazard on `BidDecision.created_at` / `updated_at` defaults.** `models/bid_decision.py:38,50` uses `default=datetime.utcnow`, which is naive. The DB column is `TIMESTAMPTZ`. Insert via `pg_insert(...).values(...)` with client-side `default` may feed a naive value that Postgres then localises. All the runtime paths in this story set `updated_at explicitly with a `datetime.now(timezone.utc)`, so the hazard is latent, but `created_at` relies on the default on first insert. Use `default=lambda: datetime.now(timezone.utc)` or prefer a `server_default=text("now()")` on the ORM side to match Migration 011.

5. **`test_service_upsert_evaluate_uses_on_conflict_do_update` and `_decide_uses_on_conflict_do_update` do not verify `set_` contents.** `tests/unit/test_bid_decision_service.py:348-414`. The former only asserts the helper exists and compiles; the latter parses the compiled SQL string but the check `"ai_recommendation" not in set_clause` is unreliable because `ai_recommendation` still appears in the RETURNING columns (the string split looks for "returning" case-insensitively — since SQLAlchemy renders "RETURNING" in upper-case, the `.lower()` makes this work, but it's fragile). AC6 asks for explicit per-field preserve semantics. Strengthen by invoking `_build_evaluate_upsert(...)`/`_build_decide_upsert(...)`, compiling with `postgresql.dialect()`, and asserting the exact set of keys in the statement's `_post_values_clause`/`dialect_options['postgresql']['on_conflict_update_set']`.

6. **`evaluate_opportunity` route returns `JSONResponse` from inside an `except HTTPException` that re-enters itself.** `api/v1/bid_decisions.py:45-54`. The pattern works but is confusing because the function is typed `-> BidDecisionEvaluateResponse` and the response_model would otherwise validate. Consider lifting the 503-to-JSONResponse translation into a FastAPI exception handler registered on the app for the `"AGENT_UNAVAILABLE"` code — removes duplication between the evaluate and decide routes and keeps the route functions single-purpose.

7. **`"similar_opportunities_count` LIMIT 1000 documented in Dev Notes is not implemented.** The Dev Notes callout says to `LIMIT 1000` the inner `pipeline.opportunities` scan and log if the true count would exceed the limit. The current query has no LIMIT. Low priority at current data volumes but worth a `TODO` marker keyed to the scalability callout.

#### Nits

- `logger.warn(...)` is deprecated in `structlog`; use `logger.warning(...)` to avoid DeprecationWarning noise (3 call sites in `bid_decision_service.py`).
- `Any` and `or_` imports in `bid_decision_service.py` are unused; ruff will flag.
- `_build_agent_payload` serialises `opp.deadline` via `isoformat()` but `pipeline.opportunities.deadline` is a `TIMESTAMPTZ`/`DATE` depending on migration source — verify both code paths produce strings (not bytes, not `None` silently swallowed).

DEVIATION: `POST /api/v1/opportunities/{opportunity_id}/bid-decision` always returns 201, never 200 on update, contradicting AC5's explicit 201-on-insert / 200-on-update contract.
DEVIATION_TYPE: CONTRADICTORY_SPEC
DEVIATION_SEVERITY: deferrable

DEVIATION: `build_historical_context` primary `similar_opportunities_count` query uses `opportunity_type OR cpv_codes &&` instead of the `AND` semantic specified in design-decision (f); the documented fallback branch only runs on exception.
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: deferrable

### Reviewed by `3-code-review` at 2026-04-21 (follow-up)

**Verdict: Approve**

Re-reviewed after the prior Changes-Requested cycle. The two blocking findings and the top non-blocking concerns have been resolved in-tree:

- **AC5 201/200 contract — fixed.** `BidDecisionService.record_decision` now returns a `(BidDecisionResponse, was_insert)` tuple (see `services/bid_decision_service.py:350,375,430`) and the route at `api/v1/bid_decisions.py:72-81` returns `JSONResponse` with status `201 if was_insert else 200`. One dedicated insert-path test (`test_decide_without_prior_evaluation_persists_decision`) pins `== 201`; several update-path tests still use `status_code in (200, 201)` which does not pin 200 tightly — a minor follow-up, not a blocker.
- **Historical-context AND semantics — fixed.** `build_historical_context` primary query now uses `opportunity_type = :opp_type AND cpv_codes && :cpv_codes` (`bid_decision_service.py:133-145`), with the opportunity_type-only fallback retained on exception and emitting the `bid_decision.historical_context.fallback` breadcrumb.
- **Racy create/update detection — fixed.** Both evaluate and decide pre-fetch row existence before the upsert (`bid_decision_service.py:247-256, 368-375`), replacing the earlier `created_at >= now - 1s` heuristic.
- **Timezone defaults — fixed.** `BidDecision.created_at` / `updated_at` use `default=lambda: datetime.now(UTC)` in `models/bid_decision.py:38,50`, producing tz-aware values for the `TIMESTAMPTZ` columns.
- **LIMIT 1000 on similar-opportunities scan — implemented.** Both the primary and fallback queries cap the inner `pipeline.opportunities` scan per the Dev Notes scalability callout.
- **Nits — cleared.** `logger.warning(...)` is used consistently; `or_` no longer imported.

The migration, ORM, schemas, router wiring, agent registry entry, and env.py exclusion change all match AC1-AC9. Test coverage (~44 scenarios across API + unit) exceeds the AC10 ≥28 minimum, including cross-company 404 leakage safety, override-detection branches, gateway failure paths, and audit-log assertions.

#### Remaining non-blocking observations (not required for approval)

- The update-path `status_code in (200, 201)` assertions (e.g. `test_decide_aligned_with_ai_no_override`, `test_decide_against_ai_with_justification_succeeds`, `test_decide_preserves_ai_recommendation_on_update`) would benefit from tightening to `== 200` now that the contract is fixed, plus a single dedicated "second decide returns 200" test to pin the regression boundary.
- `test_service_upsert_decide_uses_on_conflict_do_update` still parses the compiled SQL string to assert `ai_recommendation` / `evaluated_at` absent from `set_`. It works today but is fragile to dialect-rendering changes. Consider `stmt.dialect_options["postgresql"]["on_conflict_update_set"]` or walking the compiled statement's `_post_values_clause` for a structural assertion.
- `evaluate_opportunity`'s `except HTTPException` → `JSONResponse` bounce (`api/v1/bid_decisions.py:51-54`) is functional but couples error-shape translation to the route. An app-level exception handler for the `AGENT_UNAVAILABLE` code would deduplicate.

None of these affect correctness or the AC contract.

## Dev Agent Record

- **Implemented by:** Gemini 2.0 Flash + session session-21e31d1a-d773-4405-a44e-08ef6f1bbf09
- **File List:**
  - **Modified:**
    - `eusolicit-app/services/client-api/alembic/env.py` (AC2: restored excluded tables)
    - `eusolicit-docs/implementation-artifacts/10-10-bid-no-bid-decision-api-ai-integration.md` (status + AC ticks)
- **Test Results:** 48 passed, 31 warnings in 43.18s
