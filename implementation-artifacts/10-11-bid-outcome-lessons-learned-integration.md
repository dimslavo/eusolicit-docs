# Story 10.11: Bid Outcome & Lessons Learned Integration

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Epic
Epic 10: Collaboration, Tasks & Approvals

## Metadata
- **Story Key:** 10-11-bid-outcome-lessons-learned-integration
- **Points:** 3
- **Type:** backend
- **Module:** Client API (`client_api.api.v1.bid_outcomes` (new), `client_api.api.v1.preparation_logs` (new), `client_api.models.bid_outcome` (new — runtime ORM model upgrading the Migration 011 analytics stub), `client_api.models.bid_preparation_log` (new — runtime ORM model upgrading the Migration 011 analytics stub), `client_api.schemas.bid_outcomes` (new), `client_api.schemas.preparation_logs` (new), `client_api.services.bid_outcome_service` (new), `client_api.services.preparation_log_service` (new), new Alembic migration `045_bid_outcomes_preparation_logs_runtime_columns`) + AI Gateway agent registry (`services/ai-gateway/config/agents.yaml` — new `lessons-learned` entry) + shared events schema (`packages/eusolicit-models/src/eusolicit_models/events.py` — new `BidOutcomeRecorded` event class appended to `ServiceEvent` discriminated union).
- **Priority:** P1 (Epic 10 outcome-tracking surface — closes the bid lifecycle loop started by Story 10.10 Bid/No-Bid Decision and feeds Story 12.4 ROI Tracker + Story 12.5 Team Performance dashboards via the analytics MVs `mv_roi_tracker` and `mv_team_performance`, both of which already read from `client.bid_outcomes` and `client.bid_preparation_logs`).
- **Depends On:** Story 11.3 (`AiGatewayClient.run_agent` + `AiGatewayTimeoutError` / `AiGatewayUnavailableError` + `get_ai_gateway_client` — the shared E04 HTTP client and the E11-R-001 error contract this story reuses for the asynchronous Lessons Learned call), Story 1.3 / Migration 011 (the analytics-source `client.bid_outcomes` and `client.bid_preparation_logs` stub tables this story upgrades in-place with runtime columns; the downstream `mv_roi_tracker` and `mv_team_performance` MVs MUST continue to refresh unaffected), Story 2.4 (`get_current_user` + `CurrentUser`), Story 2.10 (`require_role` company-role gate — `admin` / `bid_manager` allow-list), Story 2.11 (`audit_service.write_audit_entry`), Story 6.5 (`pipeline.opportunities` read-only Core Table for opportunity existence + soft-delete validation), Story 7.1 / 7.2 (`client.proposals` — the `proposal_id` foreign-key target; cross-company and ownership validation joins through `proposals.company_id`), Story 10.10 (`client.bid_decisions` — the optional traceability join target; an outcome MAY reference an existing bid-decision row but is not required to), Story 4.3 / 11.3 `AiGatewayClient.run_agent` contract (sync agent invocation driven from a background task — no SSE streaming for this story), `eusolicit_common.events.publisher.EventPublisher` (the Story 1.5 Redis Streams publisher; pattern established in Story 10.9 `approval_decision_service.emit_approval_decided`).

## Story

As a **company admin or bid_manager closing out a submitted proposal**,
I want **to record the final outcome of a bid (won / lost / withdrawn) with optional contract value, evaluator feedback, and per-criterion evaluator scores — and to log each team member's preparation time and cost against that proposal — so that the system emits a `BidOutcomeRecorded` event, asynchronously invokes the Lessons Learned Agent to surface key takeaways and improvement areas, and exposes aggregated preparation effort (total hours, total cost) for the proposal**,
so that **my organisation closes the bid lifecycle loop started by the Story 10.10 bid/no-bid decision; the persisted outcome feeds the Story 12.4 ROI Tracker and Story 12.5 Team Performance dashboards; the AI-generated lessons are retrievable for future bids on similar opportunities; and per-user preparation cost is attributable for chargeback, win-rate-ROI calculations, and retrospective analysis.**

## Description

This story adds the runtime backend for EU Solicit's post-submission bid lifecycle surface. It upgrades two existing analytics-source stubs (`client.bid_outcomes` and `client.bid_preparation_logs`, both created read-only in Migration 011) into fully-featured runtime tables by adding the columns required for the outcome-recording + preparation-logging flows and the AI Lessons Learned integration — without breaking the downstream `mv_roi_tracker` and `mv_team_performance` materialized views. It ships two thin service modules, two routers, and registers a new `lessons-learned` agent with the AI Gateway plus a new `BidOutcomeRecorded` event on the shared events schema.

The endpoints added are:

1. `POST /api/v1/opportunities/{opportunity_id}/outcome` — record an outcome row for a (company_id, opportunity_id, proposal_id) triple; persist evaluator feedback and per-criterion scores; enqueue a Redis-Streams `BidOutcomeRecorded` event; if outcome is `won` or `lost`, schedule an asynchronous Lessons Learned Agent call that writes the AI response back onto the same outcome row.
2. `GET /api/v1/opportunities/{opportunity_id}/outcome` — return the recorded outcome (including lessons, if already resolved).
3. `POST /api/v1/proposals/{proposal_id}/preparation-logs` — append a time/cost entry for the current user on a proposal.
4. `GET /api/v1/proposals/{proposal_id}/preparation-logs` — list entries with rollup aggregation (total_hours, total_cost_eur, per-user breakdown).

Because Migrations 011 already created both tables (each owned by `notification_role` with SELECT grants, referenced by `mv_roi_tracker` and `mv_team_performance`), this story does NOT drop and recreate — it runs `ALTER TABLE ... ADD COLUMN` statements, widens the `status` column's domain via a named CHECK constraint, and grants `SELECT, INSERT, UPDATE, DELETE` to `client_api_role`. The existing analytics column set remains intact so the MVs refresh unaffected.

Eight design decisions worth calling out:

**(a) One outcome per `(proposal_id)` — uniqueness lives on proposal, not opportunity.** A given company may submit multiple proposals for one opportunity over time (revised bids, consortium splits), and each proposal has its own outcome. The unique constraint is `uq_bid_outcomes_proposal_id` on `(proposal_id)` — NOT `(company_id, opportunity_id)`. Duplicate outcome POST for the same proposal returns 409 `"Outcome already recorded for this proposal"` (guarded by AC6). The existence-leakage-safe cross-company behaviour still returns 404 (not 409) when the caller's company does not own the proposal — the 409 only fires for a genuine duplicate within the caller's own company scope.

**(b) Outcome is recorded via POST only — no PATCH / PUT.** Once recorded, an outcome is append-only for the row itself (corrections require a separate admin endpoint not in scope here). The AI Lessons Learned response IS mutated back onto the same row post-record, but that's a server-internal update triggered by the background task — the HTTP contract does not expose an update verb. This matches the retrospective / historical semantics: outcomes are facts, not drafts.

**(c) Lessons Learned Agent runs asynchronously via `asyncio.create_task`, NOT inline in the request handler.** The Story 10.10 `run_agent` timeout_override for complex agents is 120s; a synchronous call would block the outcome POST and give an ugly UX for a non-time-critical analysis. The handler persists the outcome row, commits the transaction, enqueues the `BidOutcomeRecorded` event, spawns an `asyncio.create_task` that re-opens its own DB session (`AsyncSessionLocal()`) — task-local, independent of the request session — calls `gw_client.run_agent("lessons-learned", payload)`, then UPDATEs the outcome row's `lessons_learned` JSONB column and `lessons_generated_at` timestamp. On agent failure the task logs `bid_outcome.lessons_learned.failed` at WARN with `outcome_id`, `opportunity_id`, exception class and EXITS cleanly — the outcome row is never rolled back (the outcome is the source of truth, lessons are optional enrichment). A `lessons_learned_status` column captures the async state: `pending` / `completed` / `failed` (see AC1).

**(d) Lessons Learned is only triggered for `won` and `lost` outcomes — NOT `withdrawn`.** A withdrawn proposal did not proceed to evaluation and has no evaluator feedback to learn from; the agent call would be wasteful and would produce low-value output. The service-layer check is explicit: `if outcome.status in ("won", "lost"): asyncio.create_task(...)`. For `withdrawn`, `lessons_learned_status` is set to `not_applicable` at record time and never transitions.

**(e) Agent payload shape for `lessons-learned` — explicit contract.** The background task loads three data bundles:
- `outcome` — the just-recorded row: `{status, contract_value, evaluator_feedback, evaluator_scores}` (scores is the JSONB breakdown added in AC1).
- `proposal_summary` — a read-only projection of `client.proposals`: `{id, title, opportunity_id, final_version_id, submitted_at, status}`. The final version's content is NOT sent in full (too large for a single agent payload); instead, if the proposal has a `final_version_id`, the task loads the `proposal_versions.content` JSONB and extracts the first 5000 chars of any `executive_summary` / `methodology` / `win_themes` section. Log `bid_outcome.lessons_learned.content_truncated` at DEBUG when truncation occurs.
- `opportunity_context` — a slim projection of `pipeline.opportunities`: `{title, opportunity_type, cpv_codes, country, deadline, contracting_authority, evaluation_criteria}` (the same structured criteria that drove Story 7.6 Requirement Checklist).

The agent is expected to return:
```
{
  "key_takeaways":      [str, str, ...],      (1–10 items, each ≤ 500 chars)
  "improvement_areas":  [str, str, ...],      (1–10 items, each ≤ 500 chars)
  "strengths":          [str, str, ...],      (0–10 items, each ≤ 500 chars)
  "recommended_actions":[str, str, ...],      (0–10 items, each ≤ 500 chars)
  "summary":            str,                  (≤ 2000 chars)
  "generated_at":       ISO-8601              (optional — server fills with now() if absent)
}
```
Validated via `LessonsLearnedResponse.model_validate(...)`. On malformed response: log `bid_outcome.lessons_learned.malformed` at WARN, set `lessons_learned_status='failed'`, leave `lessons_learned` NULL, re-raise nothing (the task swallows to avoid propagating to the event loop). On `AiGatewayTimeoutError` or `AiGatewayUnavailableError`: set `lessons_learned_status='failed'`, log as above.

**(f) `BidOutcomeRecorded` event is emitted INSIDE the request transaction — before the async task spawn — so event delivery is tied to successful row persistence.** If the DB commit rolls back, no event fires (the publish is after commit but before the HTTP response is assembled; this matches Story 10.9's `emit_approval_decided` pattern). The Redis publish itself is best-effort wrapped in `try/except` that logs `bid_outcome.event_emission_failed` at WARN — we do NOT 500 the request on a Redis outage (the durable record is the DB row + the audit log; downstream consumers that missed the event can reconcile from the DB). The event payload is:
```
{
  "event_type": "BidOutcomeRecorded",
  "outcome_id": str(row.id),
  "proposal_id": str(row.proposal_id),
  "opportunity_id": str(row.opportunity_id),
  "company_id": str(row.company_id),
  "status": "won" | "lost" | "withdrawn",
  "contract_value_eur": float | None,
  "recorded_by": str(user_id)
}
```
Stream: `eu-solicit:bid-outcomes`. The event class is added to `packages/eusolicit-models/src/eusolicit_models/events.py` and wired into the `ServiceEvent` discriminated union (AC9).

**(g) Preparation-log writes are append-only per user per proposal with NO uniqueness constraint.** Multiple entries per user per day are legitimate (e.g. "2h drafting" + "1h review" logged as two rows). The POST is always an INSERT; no upsert. The GET aggregation returns:
```
{
  "entries": [BidPreparationLogResponse, ...],     (ordered by logged_at DESC)
  "aggregation": {
    "total_hours": Decimal,
    "total_cost_eur": Decimal | None,              (None only if NO entry has cost_eur; any single entry contributes)
    "entry_count": int,
    "per_user": [
      {"user_id": UUID, "user_name": str | None, "hours": Decimal, "cost_eur": Decimal | None},
      ...
    ]
  }
}
```
User names are joined from `client.users`; if a user has been deactivated/deleted the row shows `user_name=null` but the user_id remains (historical log rows MUST survive user deletion — this is the same pattern as Story 2.11 audit log immutability). Cost aggregation uses `SUM(cost_eur) FILTER (WHERE cost_eur IS NOT NULL)` so partially-logged costs don't silently zero-fill.

**(h) RBAC — proposal-level gate, not just company-role.** Because preparation logs and outcomes are tied to a specific proposal, the authorisation gate is stricter than bid_decisions (which only checked `require_role("bid_manager")`). Outcome recording requires `require_role("bid_manager")` AND ownership: the proposal must belong to the caller's company, enforced via `proposals.company_id = current_user.company_id` (or 404 under existence-leakage). Preparation-log writes DO NOT require the bid_manager role — any non-read_only collaborator can log their own hours (via the Story 10.2 `require_proposal_role` gate for all proposal collaborator roles except `read_only`). Preparation-log reads are similarly Story-10.2-gated — all proposal collaborators INCLUDING `read_only` can see rollup aggregation (viewing historical effort is a transparency feature, not a mutation).

The two routers are mounted at:
- `/api/v1/opportunities/{opportunity_id}/outcome` (routes: POST, GET) — included beside `bid_decisions_v1.router` in `client_api/main.py`.
- `/api/v1/proposals/{proposal_id}/preparation-logs` (routes: POST, GET) — included beside `proposals_v1` / `approvals_v1`.

## Acceptance Criteria

1. [x] **AC1 — Migration `045_bid_outcomes_preparation_logs_runtime_columns.py` upgrades both `client.bid_outcomes` and `client.bid_preparation_logs` in place.** Revision `"045"`, `down_revision = "044"`. `upgrade()` MUST:

    For `client.bid_outcomes`:
    - Add `evaluator_feedback TEXT NULL` column.
    - Add `evaluator_scores JSONB NULL` column (per-criterion score breakdown; schema-layer enforced to be a dict of `{criterion_name: {score: float, max_score: float | None, rationale: str | None}}` — DB stays permissive JSONB).
    - Add `lessons_learned JSONB NULL` column (stores the full `LessonsLearnedResponse` serialisation from AC7).
    - Add `lessons_learned_status VARCHAR(20) NOT NULL DEFAULT 'not_applicable'` column (domain: `pending | completed | failed | not_applicable`).
    - Add `lessons_generated_at TIMESTAMPTZ NULL` column.
    - Add `recorded_by UUID NULL` column (soft link to `client.users.id`, no FK — matches the Story 10.7 / 10.8 / 10.9 / 10.10 soft-link precedent).
    - Add `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()` column.
    - Alter the existing `status` column: widen the implicit `VARCHAR(20)` domain with a named CHECK constraint `ck_bid_outcomes_status` predicate `status IN ('won','lost','withdrawn')`. Backfill any pre-existing rows whose status is outside this set with a defensive UPDATE (the analytics-seed fixtures in dev may have used loose values); if any row remains in violation after the backfill, fail loudly.
    - Add a named CHECK constraint `ck_bid_outcomes_lessons_status` predicate `lessons_learned_status IN ('pending','completed','failed','not_applicable')`.
    - Add unique constraint `uq_bid_outcomes_proposal_id` on `(proposal_id)` — one outcome per proposal. If duplicates exist at migration time, fail loudly (do NOT silently deduplicate — design decision (a)).
    - Add composite index `ix_bid_outcomes_company_opportunity` on `(company_id, opportunity_id)` for the GET-by-opportunity path (NOT redundant with the per-proposal unique index since lookups happen via opportunity).
    - Add index `ix_bid_outcomes_status` on `(status)` to accelerate the Story 12.5 team-performance and Story 10.10 historical-context aggregation queries (both group by status).
    - `GRANT SELECT, INSERT, UPDATE, DELETE ON client.bid_outcomes TO client_api_role`. Preserve existing `notification_role` SELECT grants.

    For `client.bid_preparation_logs`:
    - Add `opportunity_id UUID NULL` column (convenience denormalisation for the Story 12.4 ROI dashboard join path; soft link — no FK).
    - Add `company_id UUID NULL` column (same rationale — the row is reachable via `proposal_id → proposals.company_id` but the direct column lets ROI aggregation avoid a join).
    - Add `note TEXT NULL` column (free-form description; the existing `activity_type VARCHAR(100)` column is too narrow for notes).
    - Add `created_at TIMESTAMPTZ NOT NULL DEFAULT now()` column (separate from `logged_at`, which is the user-supplied activity date; `created_at` is the server-insert timestamp for audit ordering).
    - Backfill `company_id` and `opportunity_id` for any pre-existing rows by joining `client.proposals` (if no proposals row exists, leave NULL — defensive).
    - Add composite index `ix_bid_preparation_logs_proposal_user` on `(proposal_id, user_id)` for the per-user aggregation hot path.
    - Add index `ix_bid_preparation_logs_company_opportunity` on `(company_id, opportunity_id)` for Story 12.4 ROI joins.
    - `GRANT SELECT, INSERT, UPDATE, DELETE ON client.bid_preparation_logs TO client_api_role`. Preserve existing `notification_role` SELECT grants.

    `downgrade()` drops all added columns, CHECK constraints, unique constraints, and indices on both tables — restoring the Migration 011 analytics shape. Do NOT drop either table itself (MVs depend on them).

    Migration docstring MUST state explicitly that `mv_roi_tracker` and `mv_team_performance` remain unaffected (strictly additive columns; no type changes on analytics-referenced columns) and that no `REFRESH MATERIALIZED VIEW` is required post-migration.

2. [x] **AC2 — `BidOutcome` and `BidPreparationLog` ORM models.**

    `client_api/models/bid_outcome.py` declares the full `client.bid_outcomes` table: the Migration 011 original columns (`id`, `company_id`, `proposal_id`, `opportunity_id`, `status`, `contract_value_eur`, `won_at`, `created_at`) plus the seven AC1 additions. `__table_args__` includes `ck_bid_outcomes_status`, `ck_bid_outcomes_lessons_status`, `uq_bid_outcomes_proposal_id`, the two new indices, and `{"schema": "client"}`. No relationships (Proposal / Opportunity / User linkages are soft — match Story 10.10 `BidDecision` precedent).

    `client_api/models/bid_preparation_log.py` declares the full `client.bid_preparation_logs` table: the Migration 011 original columns (`id`, `proposal_id`, `user_id`, `activity_type`, `hours`, `cost_eur`, `logged_at`) plus the four AC1 additions. `__table_args__` includes the two new indices and `{"schema": "client"}`.

    Both models export from `client_api/models/__init__.py`. Remove `"bid_outcomes"` AND `"bid_preparation_logs"` from `alembic/env.py::_EXCLUDED_TABLE_NAMES` as part of this story — they are now ORM-managed end-to-end (leaving `"competitor_records"` and `"usage_meters"` in the exclusion set; those remain analytics-only).

    Both models use `default=lambda: datetime.now(UTC)` for `created_at` / `updated_at` to produce timezone-aware values (avoiding the naive-datetime hazard called out in the Story 10.10 code review).

3. [x] **AC3 — Pydantic schemas.**

    `client_api/schemas/bid_outcomes.py`:
    - `BidOutcomeStatus(str, Enum)`: `won | lost | withdrawn`.
    - `LessonsLearnedStatus(str, Enum)`: `pending | completed | failed | not_applicable`.
    - `EvaluatorCriterionScore(BaseModel)`: `score: float` (ge=0), `max_score: float | None` (ge=0), `rationale: str | None` (max_length=2000). A `@model_validator(mode="after")` asserts `score <= max_score` when both are set.
    - `EvaluatorScores(RootModel[dict[str, EvaluatorCriterionScore]])`: keys are free-form criterion names (max 20 keys, each key ≤ 100 chars).
    - `LessonsLearnedResponse(BaseModel)`: `key_takeaways: list[str]` (min_length=1, max_length=10, each item ≤ 500 chars), `improvement_areas: list[str]` (same bounds), `strengths: list[str]` (max_length=10, each ≤ 500 chars, may be empty), `recommended_actions: list[str]` (max_length=10, each ≤ 500 chars, may be empty), `summary: str` (max_length=2000), `generated_at: datetime | None = None`.
    - `BidOutcomeCreateRequest(BaseModel)`: `proposal_id: UUID`, `outcome: BidOutcomeStatus`, `contract_value_eur: Decimal | None` (ge=0, ≤ 999,999,999.99), `evaluator_feedback: str | None` (max_length=10000), `evaluator_scores: EvaluatorScores | None`. A `@model_validator(mode="after")` enforces: `outcome=won` permits `contract_value_eur` (may still be None for confidential awards); `outcome in {lost, withdrawn}` → `contract_value_eur` MUST be None (422 with detail `"contract_value_eur is only valid for 'won' outcomes"`).
    - `BidOutcomeResponse(BaseModel)`: `id: UUID`, `company_id: UUID`, `proposal_id: UUID`, `opportunity_id: UUID`, `status: BidOutcomeStatus`, `contract_value_eur: Decimal | None`, `evaluator_feedback: str | None`, `evaluator_scores: EvaluatorScores | None`, `lessons_learned: LessonsLearnedResponse | None`, `lessons_learned_status: LessonsLearnedStatus`, `lessons_generated_at: datetime | None`, `recorded_by: UUID | None`, `won_at: datetime | None`, `created_at: datetime`, `updated_at: datetime`. `model_config = ConfigDict(from_attributes=True)`.

    `client_api/schemas/preparation_logs.py`:
    - `BidPreparationLogCreateRequest(BaseModel)`: `activity_type: str` (min_length=1, max_length=100), `hours: Decimal` (gt=0, le=24 per single entry — sanity bound), `cost_eur: Decimal | None` (ge=0, le=1,000,000), `note: str | None` (max_length=2000), `logged_at: datetime | None = None` (server defaults to `now()` if absent).
    - `BidPreparationLogResponse(BaseModel)`: `id: UUID`, `proposal_id: UUID`, `user_id: UUID`, `user_name: str | None`, `activity_type: str`, `hours: Decimal`, `cost_eur: Decimal | None`, `note: str | None`, `logged_at: datetime`, `created_at: datetime`.
    - `PerUserAggregation(BaseModel)`: `user_id: UUID`, `user_name: str | None`, `hours: Decimal`, `cost_eur: Decimal | None`.
    - `PreparationLogAggregation(BaseModel)`: `total_hours: Decimal`, `total_cost_eur: Decimal | None`, `entry_count: int`, `per_user: list[PerUserAggregation]`.
    - `PreparationLogListResponse(BaseModel)`: `entries: list[BidPreparationLogResponse]`, `aggregation: PreparationLogAggregation`.

4. [x] **AC4 — `POST /api/v1/opportunities/{opportunity_id}/outcome`.** Gated by `Depends(require_role("bid_manager"))`. Body: `BidOutcomeCreateRequest`.

    Service behaviour (`bid_outcome_service.record_outcome`, single DB transaction):
    - Validate opportunity exists AND `deleted_at IS NULL` in `pipeline.opportunities`. Missing → 404 `"Opportunity not found"`.
    - Validate proposal: `SELECT id, company_id, opportunity_id, status, final_version_id FROM client.proposals WHERE id = :proposal_id`. Missing OR `proposals.company_id != current_user.company_id` → 404 `"Proposal not found"` (existence-leakage safe — no 403). `proposals.opportunity_id != :opportunity_id` → 400 `"Proposal does not belong to this opportunity"`.
    - Check for duplicate: `SELECT id FROM client.bid_outcomes WHERE proposal_id = :proposal_id`. Non-null → 409 `"Outcome already recorded for this proposal"`.
    - Insert the `bid_outcomes` row with: `company_id = current_user.company_id`, `proposal_id = request.proposal_id`, `opportunity_id = :opportunity_id`, `status = request.outcome.value`, `contract_value_eur = request.contract_value_eur`, `evaluator_feedback = request.evaluator_feedback`, `evaluator_scores = request.evaluator_scores.model_dump() if present else None`, `recorded_by = current_user.user_id`, `won_at = now() if status == "won" else None`, `lessons_learned_status = 'pending' if status in ('won','lost') else 'not_applicable'`, `created_at = updated_at = now()`. Use `RETURNING *`.
    - Write one audit row: `entity_type="bid_outcome"`, `action_type="create"`, `entity_id=row.id`, `after={"proposal_id": str, "opportunity_id": str, "status": status_value, "recorded_by": str(user_id), "contract_value_eur": float_or_none}`.
    - Commit (via `get_db_session` dep).
    - Publish `BidOutcomeRecorded` event to stream `eu-solicit:bid-outcomes` via `EventPublisher`, wrapped in `try/except` that logs `bid_outcome.event_emission_failed` at WARN and swallows (design decision (f)).
    - If `status in ("won", "lost")`: spawn `asyncio.create_task(trigger_lessons_learned_agent(outcome_id=row.id, opportunity_id=opportunity_id, ...))` — the background task specification is AC7.
    - Return 201 with `BidOutcomeResponse` (`lessons_learned` will be None at this point; the client polls GET to observe resolution).

5. [x] **AC5 — `GET /api/v1/opportunities/{opportunity_id}/outcome`.** Gated by `Depends(require_role("bid_manager"))`.

    Service behaviour:
    - Validate opportunity exists AND `deleted_at IS NULL`; missing → 404.
    - Load outcome: `SELECT * FROM client.bid_outcomes WHERE opportunity_id = :opportunity_id AND company_id = :cid`. None → 404 `"No outcome recorded for this opportunity"` (existence-leakage safe — cross-company returns 404 not 403).
    - Return 200 with `BidOutcomeResponse`. If `lessons_learned` is non-null, re-validate via `LessonsLearnedResponse.model_validate(row.lessons_learned)` before shaping into the response.

6. [x] **AC6 — Duplicate outcome guard via unique constraint + application check.** The `uq_bid_outcomes_proposal_id` unique index is the DB-level backstop. The service layer does a `SELECT` check FIRST (AC4) for a user-friendly 409 before attempting the INSERT — this keeps the `IntegrityError` path as a belt-and-suspenders for the rare race where two concurrent POSTs reach the INSERT simultaneously. On `IntegrityError` with the unique constraint name, catch and re-raise as `HTTPException(409, detail="Outcome already recorded for this proposal")` — matches the Story 10.1 `proposal_collaborators` unique-constraint handling precedent.

    Write one test for each path: (a) second POST after first completes returns 409 via the SELECT check; (b) simulated concurrent INSERT (by patching the SELECT to return None and then triggering a second `pg_insert`) verifies the `IntegrityError` fallback surfaces the 409.

7. [x] **AC7 — Lessons Learned background task (`trigger_lessons_learned_agent`).** Signature: `async def trigger_lessons_learned_agent(outcome_id: UUID) -> None`. Lives in `client_api.services.bid_outcome_service`.

    Behaviour:
    - Open a NEW `AsyncSession` via `AsyncSessionLocal()` — NOT the request session (which is already closed after the request transaction commits). Close in `finally`.
    - Load outcome by `outcome_id`. If missing (shouldn't happen, but defensive) → log WARN + return.
    - If `outcome.status not in ("won", "lost")` → log DEBUG + return (defensive guard; the caller already gates on this but task idempotency matters).
    - Load `proposal_summary` via `SELECT id, title, opportunity_id, final_version_id, submitted_at, status FROM client.proposals WHERE id = :proposal_id`.
    - If `final_version_id` present, load `proposal_versions.content` and truncate per design decision (e) to the first 5000 chars of `executive_summary`, `methodology`, `win_themes` sections (or any present subset).
    - Load `opportunity_context` via `SELECT title, opportunity_type, cpv_codes, country, deadline, contracting_authority, evaluation_criteria FROM pipeline.opportunities WHERE id = :opportunity_id AND deleted_at IS NULL`.
    - Build agent payload (strict JSON-serialisable dict, UUIDs as strings, datetimes as ISO-8601).
    - Call `gw_client = await get_ai_gateway_client(); response = await gw_client.run_agent("lessons-learned", payload)`. On `AiGatewayTimeoutError` / `AiGatewayUnavailableError`: UPDATE `bid_outcomes` SET `lessons_learned_status='failed'`, `updated_at=now()` WHERE `id = :outcome_id`; log `bid_outcome.lessons_learned.failed` at WARN with outcome_id, exception class; commit; return.
    - Parse response: `LessonsLearnedResponse.model_validate(response)`. On `ValidationError`: same `failed` UPDATE + WARN log (`bid_outcome.lessons_learned.malformed`, raw body truncated to 500 chars); commit; return.
    - UPDATE `bid_outcomes` SET `lessons_learned = parsed.model_dump(mode="json")`, `lessons_learned_status='completed'`, `lessons_generated_at = now()`, `updated_at = now()` WHERE `id = :outcome_id`. Commit.
    - Write one audit row: `entity_type="bid_outcome"`, `action_type="update"`, `entity_id=outcome_id`, `after={"lessons_learned_status": "completed", "lessons_generated_at": iso}` (the audit captures the AI enrichment event for retrospective traceability).

    The task MUST NOT raise to the caller — any unexpected exception is caught at the outer `except Exception` and logged at ERROR with `bid_outcome.lessons_learned.unexpected_error`. The outcome row itself is never rolled back by a task failure.

8. [x] **AC8 — `POST /api/v1/proposals/{proposal_id}/preparation-logs` and `GET /api/v1/proposals/{proposal_id}/preparation-logs`.**

    Router mounted at `/proposals/{proposal_id}/preparation-logs`, tags=`["preparation-logs"]`.

    POST gated by `Depends(require_proposal_role(*PROPOSAL_WRITE_ROLES))` — any non-`read_only` collaborator (bid_manager, technical_writer, financial_analyst, legal_reviewer). Body: `BidPreparationLogCreateRequest`.

    POST service behaviour:
    - Validate proposal exists + belongs to current_user.company_id (the `require_proposal_role` gate already did the collaborator check, but the ownership and existence validation runs server-side for 404 consistency). Missing → 404.
    - INSERT one row: `proposal_id = path_param`, `user_id = current_user.user_id`, `company_id = current_user.company_id`, `opportunity_id = proposals.opportunity_id`, `activity_type = request.activity_type`, `hours = request.hours`, `cost_eur = request.cost_eur`, `note = request.note`, `logged_at = request.logged_at or now()`, `created_at = now()`.
    - Write one audit row: `entity_type="bid_preparation_log"`, `action_type="create"`, `entity_id=row.id`, `after={"proposal_id": str, "user_id": str, "hours": float, "cost_eur": float_or_none, "activity_type": str}`.
    - Return 201 with `BidPreparationLogResponse`.

    GET gated by `Depends(require_proposal_role(*PROPOSAL_ALL_ROLES))` — INCLUDES `read_only` (transparency feature; design decision (h)). Query params: `?user_id=...` (optional filter).

    GET service behaviour:
    - Validate proposal exists + ownership (404 otherwise).
    - Load entries: `SELECT bpl.*, u.email AS user_name FROM client.bid_preparation_logs bpl LEFT JOIN client.users u ON u.id = bpl.user_id WHERE bpl.proposal_id = :pid AND (:user_id IS NULL OR bpl.user_id = :user_id) ORDER BY bpl.logged_at DESC`. (Use `users.email` as `user_name` — consistent with Story 2.8/2.9 user display precedent; if `users.full_name` exists, prefer that.)
    - Compute aggregation: `SELECT SUM(hours) AS total_hours, SUM(cost_eur) FILTER (WHERE cost_eur IS NOT NULL) AS total_cost_eur, COUNT(*) AS entry_count FROM client.bid_preparation_logs WHERE proposal_id = :pid AND (:user_id IS NULL OR user_id = :user_id)`.
    - Compute per-user rollup: `SELECT user_id, SUM(hours) AS hours, SUM(cost_eur) FILTER (WHERE cost_eur IS NOT NULL) AS cost_eur FROM client.bid_preparation_logs WHERE proposal_id = :pid GROUP BY user_id`. Join `client.users` for names (LEFT JOIN to preserve deleted users per design decision (g)).
    - If no entries exist, return `{"entries": [], "aggregation": {"total_hours": 0, "total_cost_eur": None, "entry_count": 0, "per_user": []}}` with 200 (NOT 404 — empty lists are valid).

9. [x] **AC9 — `BidOutcomeRecorded` event schema + AI agent registry entries.**

    `packages/eusolicit-models/src/eusolicit_models/events.py`:
    - Append a new class `BidOutcomeRecorded(BaseEvent)` with `event_type: Literal["BidOutcomeRecorded"] = "BidOutcomeRecorded"`, fields: `outcome_id: str`, `proposal_id: str`, `opportunity_id: str`, `company_id: str`, `status: Literal["won", "lost", "withdrawn"]`, `contract_value_eur: float | None = None`, `recorded_by: str`.
    - Wire it into the `ServiceEvent` discriminated union.
    - Update `packages/eusolicit-models/build/lib/eusolicit_models/events.py` in parity if that path is under source control (otherwise skip — the build/ copy is artefact).
    - Add unit tests to `packages/eusolicit-models/tests/test_events.py` (if present; otherwise co-located schema tests): `test_bid_outcome_recorded_event_type_literal`, `test_bid_outcome_recorded_in_service_event_union`, `test_bid_outcome_recorded_serialisation_roundtrip`.

    `services/ai-gateway/config/agents.yaml`:
    ```yaml
    lessons-learned:
      kraftdata_id: "a1b2c3d4-1234-4abc-8def-000000000031"
      type: agent
      description: "Analyses a completed bid outcome (won or lost) with evaluator feedback and returns key takeaways, improvement areas, strengths, and recommended actions for future bids"
      timeout_override: 120
    ```
    The UUID is a placeholder — rotate per E04-R-009 before production deployment. If `tests/unit/test_agent_registry.py` enumerates expected logical names, include `lessons-learned` in that list.

10. [x] **AC10 — Router wiring + `main.py` include.**

    Add `services/client-api/src/client_api/api/v1/bid_outcomes.py` with router `prefix="/opportunities/{opportunity_id}/outcome"`, `tags=["bid-outcomes"]`. Register two routes: `POST ""` → `BidOutcomeResponse` (201); `GET ""` → `BidOutcomeResponse` (200).

    Add `services/client-api/src/client_api/api/v1/preparation_logs.py` with router `prefix="/proposals/{proposal_id}/preparation-logs"`, `tags=["preparation-logs"]`. Register two routes: `POST ""` → `BidPreparationLogResponse` (201); `GET ""` → `PreparationLogListResponse` (200).

    Import both in `client_api/main.py`:
    ```python
    from client_api.api.v1 import bid_outcomes as bid_outcomes_v1
    from client_api.api.v1 import preparation_logs as preparation_logs_v1
    ...
    api_v1_router.include_router(bid_outcomes_v1.router)
    api_v1_router.include_router(preparation_logs_v1.router)
    ```

    Verify resulting paths:
    - `POST /api/v1/opportunities/{opportunity_id}/outcome`
    - `GET  /api/v1/opportunities/{opportunity_id}/outcome`
    - `POST /api/v1/proposals/{proposal_id}/preparation-logs`
    - `GET  /api/v1/proposals/{proposal_id}/preparation-logs`

    Write one `test_router_no_doubled_prefix` test per router mirroring Story 10.9 / 10.10 — inspect `app.routes` and assert no `/opportunities/opportunities/` or `/proposals/proposals/` double-prefix.

11. [x] **AC11 — Unit and API tests (≥ 32 scenarios).** Split between `services/client-api/tests/api/test_bid_outcomes.py`, `services/client-api/tests/api/test_preparation_logs.py`, `services/client-api/tests/unit/test_bid_outcome_service.py`, and `services/client-api/tests/unit/test_preparation_log_service.py`. Follow Story 10.10's `respx.MockRouter` fixture pattern for the async Lessons Learned Agent mock (apply against the background task via `asyncio.get_event_loop().run_until_complete(task)` or `await task` in the test after the POST returns — see the Story 11.3 `stream_agent` deferred-task test pattern for precedent). Apply `@pytest.mark.skip(reason="RED PHASE: ...")` markers upfront so the suite compiles green at story-creation time; markers come off per the TDD activation sequence.

    **Migration (AC1) — integration (marked `integration`):**
    - `test_migration_045_adds_bid_outcomes_runtime_columns` — all seven new columns present with correct types + nullability; existing MV `mv_roi_tracker` still refreshes.
    - `test_migration_045_adds_preparation_logs_runtime_columns` — all four new columns present; backfill of `company_id` / `opportunity_id` succeeded for seeded rows.
    - `test_migration_045_bid_outcomes_status_check_constraint` — `INSERT ... status='draft'` raises `IntegrityError`.
    - `test_migration_045_bid_outcomes_unique_proposal_id` — duplicate `proposal_id` raises `IntegrityError`.
    - `test_migration_045_lessons_status_check_constraint`.
    - `test_migration_045_downgrade_reverses`.
    - `test_migration_045_revision_chain` — `revision == "045"`, `down_revision == "044"`.
    - `test_migration_045_mv_roi_tracker_refreshes_post_migration` — execute `REFRESH MATERIALIZED VIEW client.mv_roi_tracker` after upgrade; no errors.

    **ORM + schema (AC2, AC3) — unit:**
    - `test_orm_bid_outcome_importable` + `test_orm_bid_preparation_log_importable`.
    - `test_orm_bid_outcomes_removed_from_env_excluded` + `test_orm_bid_preparation_logs_removed_from_env_excluded` — neither name is in `_EXCLUDED_TABLE_NAMES`.
    - `test_schema_bid_outcome_status_enum` — `won | lost | withdrawn` only.
    - `test_schema_contract_value_rejected_for_lost_withdrawn` — `outcome=lost` + `contract_value_eur=1000` → `ValidationError`.
    - `test_schema_evaluator_score_validator` — `score > max_score` → `ValidationError`.
    - `test_schema_lessons_learned_bounds` — key_takeaways with 0 items or > 10 items → `ValidationError`.
    - `test_schema_preparation_log_hours_bounds` — `hours=25` → `ValidationError`; `hours=0` → `ValidationError` (gt=0).

    **Outcome happy paths (AC4, AC5) — API:**
    - `test_record_outcome_won_returns_201_and_triggers_lessons` — POST; row has `status='won'`, `won_at` set, `lessons_learned_status='pending'`; mock gateway returns valid `LessonsLearnedResponse`; await background task; verify row transitions to `lessons_learned_status='completed'` and `lessons_learned` JSONB populated.
    - `test_record_outcome_lost_triggers_lessons`.
    - `test_record_outcome_withdrawn_skips_lessons` — `status='withdrawn'`; `lessons_learned_status='not_applicable'`; `respx` gateway mock is NEVER called (assert `mock_router.calls.call_count == 0`).
    - `test_record_outcome_emits_bid_outcome_recorded_event` — patch `EventPublisher.publish`; verify called once with `event_type="BidOutcomeRecorded"`, correct payload fields, stream `eu-solicit:bid-outcomes`.
    - `test_record_outcome_event_emission_failure_does_not_500` — patch `EventPublisher.publish` to raise `redis.ConnectionError`; POST still returns 201; WARN log captured.
    - `test_get_outcome_returns_200_with_lessons` — seed outcome row with lessons_learned populated; GET → 200 with full `BidOutcomeResponse` matching shape.
    - `test_get_outcome_pending_lessons_returns_200_with_null_lessons` — seed `lessons_learned_status='pending'`, `lessons_learned=NULL`; GET → 200; `lessons_learned=None`, `lessons_learned_status='pending'`.

    **Outcome gate failures (AC4, AC5):**
    - `test_record_outcome_non_existent_opportunity_returns_404`.
    - `test_record_outcome_soft_deleted_opportunity_returns_404`.
    - `test_record_outcome_wrong_company_proposal_returns_404` — Company B user POSTs with a proposal_id owned by Company A → 404 (existence-leakage safe).
    - `test_record_outcome_proposal_opportunity_mismatch_returns_400` — proposal belongs to a different opportunity → 400.
    - `test_record_outcome_duplicate_returns_409` — second POST for same proposal → 409.
    - `test_record_outcome_concurrent_duplicate_surfaces_409_via_integrity_error` — patch SELECT to return None; the pg_insert's IntegrityError is caught and re-raised as 409.
    - `test_record_outcome_unauthenticated_returns_401`.
    - `test_record_outcome_technical_writer_role_returns_403` — non-admin / non-bid_manager.
    - `test_record_outcome_invalid_status_value_returns_422`.
    - `test_get_outcome_cross_company_returns_404`.

    **Lessons Learned background task (AC7) — unit:**
    - `test_lessons_agent_success_updates_row` — mock `run_agent` returns valid payload; after task completes, DB row has `lessons_learned_status='completed'`, `lessons_generated_at` set, `lessons_learned` JSONB matches.
    - `test_lessons_agent_timeout_sets_status_failed` — mock raises `AiGatewayTimeoutError`; row updates to `lessons_learned_status='failed'`; WARN log captured; task does NOT raise to caller.
    - `test_lessons_agent_malformed_response_sets_status_failed` — response missing `key_takeaways`; same failed behaviour; WARN log with truncated raw body.
    - `test_lessons_agent_skipped_for_withdrawn` — task invoked directly with withdrawn outcome → early return; no `run_agent` call.
    - `test_lessons_agent_writes_audit_row_on_success` — one audit row with `entity_type="bid_outcome"`, `action_type="update"`, `after.lessons_learned_status="completed"`.
    - `test_lessons_agent_payload_shape` — inspect `respx` recorded request body; assert `outcome`, `proposal_summary`, `opportunity_context` keys present; proposal content truncated to ≤ 5000 chars.
    - `test_lessons_agent_uses_independent_db_session` — mock `AsyncSessionLocal` and assert the task opens a fresh session (not the request session).
    - `test_lessons_agent_outcome_row_survives_task_failure` — force the task to raise an unexpected `RuntimeError`; outcome row is still present and not rolled back; ERROR log captured.

    **Preparation-log happy paths (AC8) — API:**
    - `test_log_preparation_returns_201_and_persists` — technical_writer collaborator POSTs `{activity_type: "Drafting", hours: 2.5, cost_eur: 200}`; row persisted; `user_id = current_user`; audit row written.
    - `test_log_preparation_logged_at_defaults_to_now` — omit `logged_at`; row has `logged_at ≈ now()`.
    - `test_list_preparation_logs_returns_entries_and_aggregation` — seed 3 entries (2 users); GET returns entries ordered by `logged_at DESC`, aggregation with correct `total_hours`, `total_cost_eur`, per-user breakdown.
    - `test_list_preparation_logs_empty_returns_empty_lists_not_404` — proposal exists with zero logs; GET → 200 with `entries=[]`, `aggregation.total_hours=0`, `per_user=[]`.
    - `test_list_preparation_logs_user_id_filter` — `GET ?user_id=<uid>` returns only that user's entries; aggregation reflects the filter.
    - `test_list_preparation_logs_cost_aggregation_skips_null` — 2 entries, one with cost_eur=100, one with cost_eur=None; `total_cost_eur=100` (not 0, not None).
    - `test_list_preparation_logs_zero_cost_entries_total_none` — all entries have `cost_eur=NULL`; `total_cost_eur=None`.
    - `test_list_preparation_logs_survives_deleted_user` — seed entry then delete user row; GET still returns entry with `user_name=None`.
    - `test_log_preparation_read_only_collaborator_cannot_write_returns_403`.
    - `test_list_preparation_logs_read_only_collaborator_can_read_returns_200`.
    - `test_log_preparation_non_collaborator_returns_403`.
    - `test_log_preparation_cross_company_proposal_returns_404` (existence-leakage safe via `require_proposal_role`).
    - `test_log_preparation_hours_over_24_returns_422`.
    - `test_log_preparation_negative_cost_returns_422`.

    **Event wiring (AC9) — unit:**
    - `test_bid_outcome_recorded_event_type_literal`.
    - `test_bid_outcome_recorded_in_service_event_union` — `ServiceEvent` can parse a payload with `event_type="BidOutcomeRecorded"` into the correct subclass.
    - `test_bid_outcome_recorded_serialisation_roundtrip`.

    **Router wiring (AC10):**
    - `test_router_bid_outcomes_paths_exist` + `test_router_preparation_logs_paths_exist`.
    - `test_router_bid_outcomes_no_doubled_prefix` + `test_router_preparation_logs_no_doubled_prefix`.

## Dev Notes

- **Test Expectations (from Epic-level test design):** `eusolicit-docs/test-artifacts/test-design-epic-10.md` is **not** available — Epic 10 was never epic-test-designed (the `test-artifacts/` directory for Epic 10 contains only per-story ATDD checklists 10-1 through 10-10; no epic-level design doc exists; same gap noted in Stories 10.9 and 10.10). Coverage strategy therefore follows the established per-story ATDD patterns: `httpx.AsyncClient` + `async with AsyncSession` for API tests; `pytest-asyncio` with `asyncio_mode="auto"`; cross-company isolation returns 404 (NOT 403) for existence-leakage safety; AI Gateway mocks use `respx.MockRouter` per Story 11.3's `test_espd_autofill_export.py` fixture pattern; event-publisher mocks patch `eusolicit_common.events.publisher.EventPublisher.publish` per Story 10.9 precedent. Use the arrange/act/assert block format from `tests/api/test_bid_decisions.py` (Story 10.10) and `tests/api/test_approvals.py` (Story 10.9). For DB-backed migration tests, mirror `tests/integration/test_migration_044.py` — `integration` mark and skipped in unit-only runs. For the async background-task tests, follow the `asyncio.wait_for` + explicit-await pattern from Story 11.3's `test_espd_autofill_export.py::test_autofill_completes_asynchronously` (the task reference is returned from the service / captured via a test-scoped fixture; the test awaits it explicitly rather than sleeping). The RED-phase `@pytest.mark.skip(reason="RED PHASE: ...")` markers are applied upfront so the entire test suite compiles green at story-creation time; they come off phase-by-phase per the TDD activation sequence.

- **AI Gateway client reuse — no new dependency code.** Import `AiGatewayClient`, `AiGatewayTimeoutError`, `AiGatewayUnavailableError`, `get_ai_gateway_client` from `client_api.services.ai_gateway_client` (Story 11.3 artefact; reused by Story 10.10). The `run_agent` synchronous call pattern is the right primitive here — the scorecard- / lessons-learned-style agents return a single JSON payload within their `timeout_override`. No SSE streaming.

- **Event emitter reuse.** Follow the Story 10.9 `approval_decision_service.emit_approval_decided` shape exactly: `EventPublisher(redis_client)`, stream-per-domain (`eu-solicit:approvals` → `eu-solicit:bid-outcomes`), best-effort `try/except` that logs WARN on failure. The `BidOutcomeRecorded` event class adds to `packages/eusolicit-models/src/eusolicit_models/events.py` and extends the `ServiceEvent` `Annotated[... | BidOutcomeRecorded, Discriminator("event_type")]` union.

- **Story 10.10 coupling — `bid_decisions.id` traceability join.** A `bid_outcomes` row may reference a prior `bid_decisions` row (same `company_id` + `opportunity_id`) for retrospective "did we pursue what we decided to pursue?" analysis. This story does NOT enforce the link (no FK, no required field) — the join is lazy at report time. If a future story adds a `bid_decision_id` column, that is additive. For now the `company_id + opportunity_id` tuple on both tables is the implicit join key.

- **Analytics MV compatibility (Migration 011 source preservation).** `mv_roi_tracker` joins `client.bid_preparation_logs bpl ON bpl.proposal_id = p.id` and aggregates `SUM(bpl.cost_eur)`; `mv_team_performance` aggregates `COUNT(DISTINCT bo.id) FILTER (WHERE bo.status = 'won')` from `client.bid_outcomes`. Our added columns (lessons_learned, evaluator_feedback, evaluator_scores, recorded_by, company_id, opportunity_id, note, etc.) are STRICTLY ADDITIVE — neither MV references them. The `ck_bid_outcomes_status` CHECK constraint tightens the domain to `('won','lost','withdrawn')`; `mv_team_performance`'s `WHERE bo.status = 'won'` filter is unaffected. No `REFRESH MATERIALIZED VIEW` is required post-migration (column additions do not invalidate the existing materialized rows). Document in the migration header.

- **Why extend, not replace.** Same reasoning as Story 10.10: Migration 011 is already applied in production; the MVs are live; a drop-and-create would require a deprecating-the-MV window. Strictly additive `ALTER TABLE ADD COLUMN` is the low-risk path. Unlike Story 10.10's `bid_decisions` where the entire runtime semantics were new, here the MV stubs had a minimal-but-correct schema that we extend rather than discard.

- **`lessons_learned_status = 'not_applicable'` for `withdrawn`.** Setting the status explicitly at insert time (rather than leaving it NULL) lets the GET response always return a typed value and lets Story 12.5's team-performance dashboard count "how many completed outcomes have AI lessons available" cleanly via `WHERE lessons_learned_status = 'completed'`. The CHECK constraint permits all four values.

- **Background-task error containment.** The `asyncio.create_task(trigger_lessons_learned_agent(...))` is fire-and-forget; FastAPI's request lifecycle does NOT await it. On a long-running agent call the HTTP response has already been sent and the client has moved on (polling GET for lesson resolution). The task must therefore: (a) use its own DB session (`AsyncSessionLocal()`); (b) catch all exceptions at the outer boundary so a stray `RuntimeError` does not crash the event loop; (c) never mutate any entity outside `bid_outcomes.lessons_learned*` and the one audit row on success. A crashed task leaves the outcome row at `lessons_learned_status='pending'` — NOT ideal (the row is effectively orphaned in pending state) but acceptable for a first cut; a follow-up story can add a Celery-beat reaper that retries `pending` rows older than 1 hour. Document this as a known-small-gap.

- **`hours` precision.** Stored as `NUMERIC` (the Migration 011 column type); at the Pydantic layer use `Decimal` to preserve precision over the wire. `SUM(hours)` returns `Decimal` from asyncpg — do NOT cast to float in the aggregation path (fractional-hour accuracy matters for chargeback). In the response JSON, serialise as a decimal string (Pydantic `Decimal` default) — the frontend can format.

- **`cost_eur` optionality.** Not every org tracks per-entry cost (some only track hours and let finance compute cost centrally). The schema accepts `cost_eur=None`; the aggregation uses `SUM(... FILTER (WHERE cost_eur IS NOT NULL))` so a single NULL entry does not zero the sum. If ALL entries have NULL cost, the aggregation returns `total_cost_eur=None` (not 0 — distinguishing "no data" from "zero cost" matters).

- **User name source — `users.email` fallback.** The Story 2.8 `users` table has `email` (required) and may have `full_name` (optional). Use `COALESCE(u.full_name, u.email)` in the per-user aggregation join. If `users.full_name` column does not exist (check Story 2.8 migration), fall back to `u.email` unconditionally; leave a TODO to upgrade when `full_name` lands.

- **Truncating proposal content for the lessons agent payload.** The full proposal JSONB in `proposal_versions.content` can be 100 KB+; passing all of it to the agent bloats the request and risks agent timeout. We extract the 3 most-informative sections (`executive_summary`, `methodology`, `win_themes`) and truncate each to 5000 chars. If the proposal shape has none of these (e.g. an early-draft proposal), send `{"sections": []}` and let the agent operate on evaluator feedback + opportunity context alone. Log `bid_outcome.lessons_learned.content_truncated` at DEBUG.

- **Idempotency on re-trigger.** Calling the task twice for the same outcome (which shouldn't happen — the POST is guarded by the duplicate 409 — but defensive) should overwrite the prior lessons with the new result. The UPDATE is unconditional on `outcome_id`. A follow-up story could introduce a `manual_retrigger_lessons` admin endpoint if the product team wants it; not in scope here.

- **Out of scope:**
  - PATCH / PUT on bid outcomes (append-only post-record — corrections require a separate admin endpoint not specified in Epic 10).
  - DELETE on bid_outcomes or bid_preparation_logs (the historical record is meant to be durable; deletion, if ever required, is a separate admin-audit-logged flow).
  - A Celery-beat reaper for `lessons_learned_status='pending'` rows older than N hours (follow-up story if task crashes prove common in practice).
  - Manual re-trigger endpoint for lessons learned (implicit in the re-record behaviour; explicit UX is a future polish).
  - Frontend UI (Story 10.16 — outcome recording form + lessons display; preparation-log UI is also in 10.16 or a follow-up).
  - Cross-proposal rollups of preparation effort at the opportunity / company level (Story 12.4 ROI Tracker owns that via `mv_roi_tracker`).
  - Integration of evaluator scores into the Story 7.7 compliance/risk scoring feedback loop (future story).
  - Changes to `client.bid_decisions` beyond the optional implicit join (Story 10.10 owns that table).
  - Emission of a separate `LessonsLearnedReady` event when the background task completes (consumers can poll GET; adding a second event doubles the integration surface for marginal value).

- **Future-compatibility notes:**
  - Story 12.5 Team Performance Dashboard reads `mv_team_performance`, which aggregates wins per user via `bid_outcomes + bid_preparation_logs`. Our new `recorded_by` column does NOT feed the MV yet (current MV uses `bpl.user_id`). A future MV upgrade could use `recorded_by` to attribute the outcome to the user who closed it — that's a Story 12.x concern, not this one.
  - Story 12.4 ROI Tracker reads `mv_roi_tracker` grouped by `company_id` + `opportunity_id`. Our added `company_id` / `opportunity_id` denormalised columns on `bid_preparation_logs` let the MV avoid a three-way join; 12.4 can adopt them without requiring changes here.
  - The `LessonsLearnedResponse` JSONB is intentionally generous (key_takeaways / improvement_areas / strengths / recommended_actions) to accommodate agent-prompt evolution. If a future agent-prompt version returns an unexpected key, `model_validate` with `extra="ignore"` (the Pydantic default) preserves forward compatibility; we do NOT set `extra="forbid"`.

- **Router include pattern note (AC10):** the prefixes `/opportunities/{opportunity_id}/outcome` and `/proposals/{proposal_id}/preparation-logs` are fully embedded in `APIRouter(prefix=...)`, so `include_router` takes no additional prefix (same pattern as Story 10.9 / 10.10). The `POST ""` / `GET ""` routes intentionally register the bare prefix as the path so `/outcome` and `/preparation-logs` (not `/outcome/` and `/preparation-logs/`) are the canonical URLs. Confirm FastAPI's `redirect_slashes=True` default does not cause a 307 loop in tests; if it does, register the path explicitly as `/` and document.

## Change Log

- 2026-04-21: Initial story context created. (bmad-create-story)

## Known Deviations

- **AC3 (Partial Infeasibility):** `LessonsLearnedResponse.improvement_areas` lower bound relaxed to `min_length=0`. Intentional to avoid spurious `lessons_learned_status='failed'` rows on strong-score wins where no improvements are necessary.
- **AC7 (Schema Gap):** Story assumes `Proposal.final_version_id` and `Proposal.submitted_at` which do not exist in the current schema. Implemented using `Proposal.current_version_id` and omitted `submitted_at`.
- **Migration logic:** `alembic/env.py` was initially not updated to remove the table exclusions, causing unit tests to fail. This has been corrected.

### Detected by `2-dev-story` at 2026-04-21T03:21:31Z

- Story assumes Proposal.final_version_id and Proposal.submitted_at which do not exist in the current schema. Implemented using Proposal.current_version_id and omitted submitted_at. _(type: `MISSING_REQUIREMENT`; severity: `deferrable`)_

### Detected by `3-code-review` at 2026-04-21T04:09:44Z (session e3bf0860-16e7-4531-898e-1a62f54af104)

- `LessonsLearnedResponse.improvement_areas` uses `min_length=0` instead of AC3's implied `min_length=1` (spec says "same bounds" as `key_takeaways`); intentional to avoid spurious `lessons_learned_status='failed'` on strong-score wins, documented inline in code but not yet appended to `## Known Deviations`. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_
- `LessonsLearnedResponse.improvement_areas` uses `min_length=0` instead of AC3's implied `min_length=1` (spec says "same bounds" as `key_takeaways`); intentional to avoid spurious `lessons_learned_status='failed'` on strong-score wins, documented inline in code but not yet appended to `## Known Deviations`. _(type: `ACCEPTANCE_GAP`; severity: `deferrable`)_

## Senior Developer Review

### Follow-up review performed at 2026-04-21T05:00Z (bmad-code-review)

**Outcome: Approve**

Re-reviewed after the developer addressed the prior review's findings. All blockers resolved and the test suite now passes cleanly (`pytest tests/unit/test_bid_outcome_service.py tests/unit/test_preparation_log_service.py tests/unit/test_bid_outcome_models_and_schemas.py tests/api/test_bid_outcomes.py tests/api/test_preparation_logs.py` → **83 passed**). The migration integration suite (`tests/integration/test_migration_045.py`) contributes 8 more scenarios, bringing total active coverage well above the AC11 ≥32 target.

#### Resolution of prior findings

| Ref | Finding | Status |
|-----|---------|--------|
| B1 | `EventPublisher.publish` signature mismatch | **Fixed** — `bid_outcome_service.py:120-134` now passes `stream=`, `event_type="BidOutcomeRecorded"`, `payload=`, `source_service="client-api"`, `tenant_id=str(company_id)`; test `test_record_outcome_emits_bid_outcome_recorded_event` asserts the correct kwargs. `test_record_outcome_event_emission_failure_does_not_500` verifies the Redis-failure path returns 201. |
| B2 | AC11 test coverage gap | **Fixed** — 32 active tests in `test_bid_outcome_service.py`, 17 in `test_preparation_log_service.py`, 16 in API `test_bid_outcomes.py`, 10 in API `test_preparation_logs.py`, 8 in `test_bid_outcome_models_and_schemas.py`, and 8 migration integration tests. All enumerated AC11 scenarios are present (gate failures, background-task paths, aggregation filters, deleted-user handling, router prefix checks). |
| H1 | `improvement_areas` `min_length=1` too strict | **Resolved by relaxation** — schema now `min_length=0` with inline comment explaining why (won outcomes with unanimously strong scores may legitimately have no areas to improve). This is a deliberate deviation from AC3's "same bounds" wording — captured as a known deviation below. |
| M1 | `evaluator_scores.model_dump()` omits `mode="json"` | **Fixed** — `bid_outcome_service.py:79`. |
| M2 | `generated_at` never auto-filled | **Fixed** — `bid_outcome_service.py:292-293` uses `parsed.model_copy(update={"generated_at": datetime.now(UTC)})`. |
| L1 | Audit entry omits `user_id` kwarg | **Fixed** — both services now pass `user_id=user_id` (`bid_outcome_service.py:104`, `preparation_log_service.py:65`). |
| L2 | `get_outcome` missing defensive re-validate | **Fixed** — `bid_outcome_service.py:170-179` re-validates `lessons_learned` JSONB on read and synthesises `failed` state + WARN log on corruption. |
| L3 | `get_session_factory()` vs. `AsyncSessionLocal()` | Acceptable — matches project session-maker idiom; `test_lessons_agent_uses_independent_db_session` patches `get_session_factory`. |
| L4 | `evaluator_scores` key `max_length=100` not enforced | **Fixed** — `schemas/bid_outcomes.py:45-49` explicitly loops the RootModel keys and raises on overflow; covered by `test_schema_evaluator_scores_key_max_length`. |

#### Remaining small notes (non-blocking)

- **Spec deviation — `LessonsLearnedResponse.improvement_areas` lower bound.** AC3 literally reads "`improvement_areas: list[str] (same bounds)`" i.e. `min_length=1`. Implementation uses `min_length=0` (via `default_factory=list, max_length=10`) as agreed in the prior-review H1 guidance. Recommend appending a one-line entry to `## Known Deviations` documenting this so the next reviewer does not re-flag it.
- **Aggregation filter pattern.** `preparation_log_service.list_preparation_logs` uses plain `func.sum(cost_eur)` rather than AC8's explicit `SUM(cost_eur) FILTER (WHERE cost_eur IS NOT NULL)`. Functionally equivalent because SQL `SUM` already ignores NULLs, and the two cost-aggregation tests (`test_list_preparation_logs_cost_aggregation_skips_null`, `test_list_preparation_logs_zero_cost_entries_total_none`) confirm the observable behaviour matches the spec. No action required.

#### Alignment with operational playbooks

The `improvement_areas` relaxation is a bounded, documented deviation from AC3 (type: `ACCEPTANCE_GAP`, severity: `deferrable`). The previously-recorded `final_version_id → current_version_id` deviation remains correctly handled and captured. No new blocking deviations detected.

DEVIATION: `LessonsLearnedResponse.improvement_areas` uses `min_length=0` instead of AC3's implied `min_length=1`; intentional to avoid spurious `lessons_learned_status='failed'` rows on strong-score wins, but not yet recorded under `## Known Deviations`.
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: deferrable

---

### Initial review performed at 2026-04-21 (bmad-code-review)

**Outcome: Changes Requested**

The core scaffolding (migration 045, ORM models, schemas, service modules, routers, event class, agent-registry entry, main.py wiring) is in place and structurally aligned with the AC spec. However, two defects and one large coverage gap prevent approval.

---

#### BLOCKING — B1. `EventPublisher.publish()` invoked with the wrong signature (AC4 / AC9)

`services/client-api/src/client_api/services/bid_outcome_service.py:131`

```python
await event_publisher.publish("eu-solicit:bid-outcomes", event.model_dump(mode="json"))
```

`EventPublisher.publish` (from `eusolicit_common/events/publisher.py:23`) is:

```python
async def publish(
    self,
    stream: str,
    event_type: str,          # positional 2 — a string discriminator
    payload: dict[str, Any],  # positional 3 — the body dict
    *,
    source_service: str,      # REQUIRED keyword-only
    tenant_id: str | None = None,
    correlation_id: str | None = None,
) -> str:
```

The current call passes the entire event dump as `event_type` (a dict where a str is expected) and omits the required keyword-only `source_service`. At runtime this raises `TypeError: missing required keyword argument 'source_service'` — which is silently swallowed by the surrounding `try/except Exception`. **Net effect: the `BidOutcomeRecorded` event is never emitted to Redis in production**, while the code logs a single WARN and the HTTP response returns 201. This directly defeats AC4 step 7 and AC9.

The test `test_record_outcome_emits_bid_outcome_recorded_event` (tests/api/test_bid_outcomes.py:222) passes only because it patches `EventPublisher.publish` with an `AsyncMock` that has no signature enforcement; the assertion `args[1]["event_type"] == "BidOutcomeRecorded"` coincidentally succeeds against the dict payload and hides the bug.

**Fix:** mirror the Story 10.9 `emit_approval_decided` shape:

```python
await event_publisher.publish(
    stream="eu-solicit:bid-outcomes",
    event_type="BidOutcomeRecorded",
    payload={
        "outcome_id": str(outcome.id),
        "proposal_id": str(outcome.proposal_id),
        "opportunity_id": str(outcome.opportunity_id),
        "company_id": str(outcome.company_id),
        "status": outcome.status,
        "contract_value_eur": float(outcome.contract_value_eur) if outcome.contract_value_eur is not None else None,
        "recorded_by": str(user_id),
    },
    source_service="client-api",
    tenant_id=str(company_id),
)
```

Then update the test to assert the correct kwargs (`kwargs["event_type"] == "BidOutcomeRecorded"`, `kwargs["payload"]["status"] == "won"`, `kwargs["source_service"] == "client-api"`) so the real signature is verified.

---

#### BLOCKING — B2. AC11 test coverage substantially incomplete (≈13 active vs. ≥ 32 required)

AC11 mandates ≥ 32 scenarios with the explicit note that "markers come off per the TDD activation sequence." The current state:

- `tests/api/test_bid_outcomes.py`: **4** active tests
- `tests/api/test_preparation_logs.py`: **2** active tests
- `tests/unit/test_bid_outcome_service.py`: **0** active tests (all ~21 tests still `@pytest.mark.skip(reason="RED PHASE: ...")`)
- `tests/unit/test_preparation_log_service.py`: **0** active tests (all ~14 tests still `@pytest.mark.skip(reason="RED PHASE: ...")`)
- `tests/unit/test_bid_outcome_models_and_schemas.py`: **7** active tests

Specifically missing (not implemented at all, or still skip-marked when the production code supports them):

- Outcome gate failures: `test_record_outcome_non_existent_opportunity_returns_404`, `..._soft_deleted_opportunity_returns_404`, `..._wrong_company_proposal_returns_404`, `..._proposal_opportunity_mismatch_returns_400`, `..._concurrent_duplicate_surfaces_409_via_integrity_error`, `..._unauthenticated_returns_401`, `..._technical_writer_role_returns_403`, `..._invalid_status_value_returns_422`, `test_get_outcome_cross_company_returns_404`.
- `test_record_outcome_withdrawn_skips_lessons` — would have caught a separate regression: verifies `respx_mock.calls.call_count == 0` when `status='withdrawn'`.
- `test_record_outcome_event_emission_failure_does_not_500` — would have caught B1 above.
- Background task: the eight AC7-required tests (`test_lessons_agent_success_updates_row`, `..._timeout_sets_status_failed`, `..._malformed_response_sets_status_failed`, `..._skipped_for_withdrawn`, `..._writes_audit_row_on_success`, `..._payload_shape`, `..._uses_independent_db_session`, `..._outcome_row_survives_task_failure`) — still skip-marked.
- Preparation-log API: 8 of the 14 AC8 scenarios (`..._logged_at_defaults_to_now`, `..._user_id_filter`, `..._cost_aggregation_skips_null`, `..._zero_cost_entries_total_none`, `..._survives_deleted_user`, `..._read_only_collaborator_cannot_write_returns_403`, `..._read_only_collaborator_can_read_returns_200`, `..._non_collaborator_returns_403`, `..._cross_company_proposal_returns_404`, `..._hours_over_24_returns_422`, `..._negative_cost_returns_422`).
- Router wiring (AC10): `test_router_bid_outcomes_no_doubled_prefix`, `test_router_preparation_logs_no_doubled_prefix`, `test_router_bid_outcomes_paths_exist`, `test_router_preparation_logs_paths_exist`.
- Event-schema unit tests (AC9): `test_bid_outcome_recorded_event_type_literal`, `test_bid_outcome_recorded_in_service_event_union`, `test_bid_outcome_recorded_serialisation_roundtrip` — present but skip-marked.
- Migration integration suite (AC1): no `tests/integration/test_migration_045*.py` file exists at all.

**Fix:** de-skip the tests whose production code exists (the majority of the `RED PHASE` list), author the missing scenarios enumerated in AC11, and add `tests/integration/test_migration_045_*.py` mirroring the Story 10.10 `test_migration_044.py` pattern for the eight AC1 migration assertions.

---

#### HIGH — H1. `LessonsLearnedResponse` `improvement_areas` lower bound is brittle (AC7)

`schemas/bid_outcomes.py:48` — `improvement_areas: list[str] = Field(min_length=1, max_length=10)`.

A perfectly valid agent response for a `won` outcome may have no clear improvement areas (e.g. unanimously strong evaluator scores). A `min_length=1` requirement forces the agent to fabricate an "area" or the row transitions to `failed`. The story AC3 text is unambiguous on this — (`"improvement_areas: list[str] (same bounds)"` as `key_takeaways` — i.e. `min_length=1`), so this matches the spec, but it is worth flagging because it will manifest as spurious `lessons_learned_status='failed'` rows once the agent is live. Recommend either relaxing to `min_length=0` (align with `strengths`/`recommended_actions`) and documenting the deviation, or handling the validation fallback more gracefully than marking the outcome as `failed`.

---

#### MEDIUM — M1. `evaluator_scores.model_dump()` omits `mode="json"` (AC4)

`services/bid_outcome_service.py:80` serialises the nested `EvaluatorCriterionScore` objects with the default `.model_dump()`. Today all fields are already JSON-native (float / str), so JSONB insertion succeeds — but if any future evaluator-score subfield becomes `Decimal` or `datetime` (e.g. a future `assessed_at` field), the silent asyncpg adapter behaviour will differ from the `mode="json"` path and cause an insert error. Use `request.evaluator_scores.model_dump(mode="json")` for parity with the lessons_learned write path (`services/bid_outcome_service.py:291`).

---

#### MEDIUM — M2. `LessonsLearnedResponse` `generated_at` never auto-filled (AC7 step 7)

The story AC7 contract says "`generated_at` ISO-8601 (optional — server fills with now() if absent)." The background task validates the agent response but does not populate `generated_at` when the agent omits it before storing to JSONB. The resulting `lessons_learned` JSONB will have `generated_at = null`, which is then exposed via GET. This is a small but user-visible deviation. Add a one-liner in `trigger_lessons_learned_agent`:

```python
parsed = LessonsLearnedResponse.model_validate(response)
if parsed.generated_at is None:
    parsed = parsed.model_copy(update={"generated_at": datetime.now(UTC)})
```

---

#### LOW — L1. Audit entry in `record_outcome` omits `user_id` kwarg (AC4 step 5)

`services/bid_outcome_service.py:100-112` calls `write_audit_entry(..., entity_type=..., entity_id=..., after={...})` but does not pass the `user_id` kwarg that `audit_service.write_audit_entry` accepts explicitly (and which captures the acting user in the `shared.audit_log.user_id` column — not just the `after` dict). The acting-user UUID ends up denormalised only inside `after["recorded_by"]`, which makes audit-table-level filtering by actor harder. Same in `preparation_log_service.log_preparation:60`. Add `user_id=user_id` to both call sites.

---

#### LOW — L2. `get_outcome` re-validation of `lessons_learned` omitted (AC5)

AC5 last line: "If `lessons_learned` is non-null, re-validate via `LessonsLearnedResponse.model_validate(row.lessons_learned)` before shaping into the response." The current `get_outcome` service just returns the ORM row and relies on FastAPI's response_model coercion. In practice Pydantic will raise if the stored JSONB is malformed — surfacing a 500. A defensive re-validate + `lessons_learned=None` fallback (with `lessons_learned_status='failed'` synthesised) would keep the GET contract usable even if a historical row's JSONB was corrupted.

---

#### LOW — L3. `trigger_lessons_learned_agent` uses `get_session_factory` instead of spec'd `AsyncSessionLocal()` (AC7)

The AC7 wording says the task "Open a NEW `AsyncSession` via `AsyncSessionLocal()`." The implementation uses `get_session_factory()` which is equivalent in effect and already aliases the project's session maker — so the behaviour is correct. Worth noting in Dev Notes that the deviation is intentional (follows the project's dependency pattern) but the test `test_lessons_agent_uses_independent_db_session` (still skipped) needs to patch `get_session_factory` rather than `AsyncSessionLocal` when activated.

---

#### LOW — L4. `evaluator_scores` key-level `max_length=100` is likely not enforced (AC3)

`schemas/bid_outcomes.py:38` uses `RootModel[dict[Annotated[str, Field(max_length=100)], EvaluatorCriterionScore]]`. Pydantic v2 does not reliably enforce `Annotated` metadata on dict keys — a known quirk. Add a `@model_validator(mode="after")` that iterates `self.root.keys()` and asserts `len(k) <= 100` explicitly, matching the spec.

---

### Alignment with operational playbooks

No new requirements deviations detected during this review. The previously-recorded deviation (Proposal `current_version_id` vs. spec's `final_version_id`) is correctly handled by the implementation and already captured in `## Known Deviations`.

---

### Summary

Structural work is solid; the two blockers are a live production-behaviour bug (silently-swallowed event publish) and a quantitative coverage shortfall against AC11. Fix B1, activate / author the enumerated AC11 tests (especially `test_record_outcome_event_emission_failure_does_not_500` which would have caught B1), and re-request review.

---

**REVIEW: Changes Requested** (superseded by follow-up approval above)

---

**REVIEW: Approve**

## Dev Agent Record

- **Implemented by:** Gemini 2.0 Flash (session-2490820f-e0af-474a-8a90-8f3edce748c3)
- **File List:**
    - **New:**
        - `eusolicit-app/services/client-api/src/client_api/api/v1/bid_outcomes.py`
        - `eusolicit-app/services/client-api/src/client_api/api/v1/preparation_logs.py`
        - `eusolicit-app/services/client-api/src/client_api/models/bid_outcome.py`
        - `eusolicit-app/services/client-api/src/client_api/models/bid_preparation_log.py`
        - `eusolicit-app/services/client-api/src/client_api/schemas/bid_outcomes.py`
        - `eusolicit-app/services/client-api/src/client_api/schemas/preparation_logs.py`
        - `eusolicit-app/services/client-api/src/client_api/services/bid_outcome_service.py`
        - `eusolicit-app/services/client-api/src/client_api/services/preparation_log_service.py`
        - `eusolicit-app/services/client-api/alembic/versions/045_bid_outcomes_preparation_logs_runtime_columns.py`
        - `eusolicit-app/services/client-api/tests/api/test_bid_outcomes.py`
        - `eusolicit-app/services/client-api/tests/api/test_preparation_logs.py`
        - `eusolicit-app/services/client-api/tests/unit/test_bid_outcome_models_and_schemas.py`
        - `eusolicit-app/services/client-api/tests/unit/test_bid_outcome_service.py`
        - `eusolicit-app/services/client-api/tests/unit/test_preparation_log_service.py`
        - `eusolicit-app/services/client-api/tests/unit/test_router_wiring_10_11.py`
        - `eusolicit-app/services/client-api/tests/integration/test_migration_045.py`
    - **Modified:**
        - `eusolicit-app/services/client-api/src/client_api/api/v1/__init__.py`
        - `eusolicit-app/services/client-api/src/client_api/models/__init__.py`
        - `eusolicit-app/services/client-api/src/client_api/main.py`
        - `eusolicit-app/services/client-api/alembic/env.py`
        - `eusolicit-app/packages/eusolicit-models/src/eusolicit_models/events.py`
        - `eusolicit-app/services/ai-gateway/config/agents.yaml`
- **Test Results:** `95 passed, 72 warnings in 23.41s`
