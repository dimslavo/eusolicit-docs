# Story 26.04: Opportunity Analyses Schema + Service Layer

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 3
**Type:** backend
**Dependencies:** S24.01 (sirmaai_projects FK reference)
**Blocks:** S26.05, S26.07, S26.08
**Created:** 2026-05-15
**Source:** E26 epic §S26.04 + epic-file schema block

## Story

As a **platform engineer**,
I want **a typed `client.opportunity_analyses` table + service layer with idempotent `start_analysis` semantics**,
so that **concurrent triggers don't produce duplicate runs and analysis results are reliably persisted with run-history visibility**.

## Acceptance Criteria

1. Alembic migration creates `client.opportunity_analyses`:
   - `id` UUID PK
   - `company_id` UUID FK → `client.companies(id)`
   - `opportunity_id` UUID NOT NULL (no FK due to cross-schema — `pipeline.opportunities`)
   - `analysis_type` TEXT — StrEnum `qualification|quantification|post_processing`
   - `sirmaai_run_id` TEXT
   - `eusolicit_run_id` UUID UNIQUE FK → `gateway.workflow_runs(eusolicit_run_id)` (cross-schema reference managed in migration)
   - `status` TEXT — StrEnum `pending|running|completed|failed`
   - `result_payload` JSONB
   - `kb_citations` JSONB DEFAULT `'[]'::jsonb`
   - `started_at` TIMESTAMPTZ NOT NULL DEFAULT now()
   - `completed_at` TIMESTAMPTZ
   - `error_message` TEXT
2. **Partial index** `(company_id, opportunity_id, analysis_type) WHERE status = 'completed'` — drives idempotency check + admin reprovision view.
3. **Service-layer functions** in `services/client-api/src/client_api/services/opportunity_analysis_service.py`:
   - `start_analysis(company_id, opportunity_id, analysis_type, force=False) -> StartAnalysisResult` — idempotent: if a completed row exists within 1 hour AND `force=False`, returns the existing row's `eusolicit_run_id` + `cached=True`. Otherwise creates a pending row + returns the new `eusolicit_run_id`.
   - `get_latest_completed(company_id, opportunity_id, analysis_type) -> Optional[OpportunityAnalysis]`
   - `record_completion(eusolicit_run_id, payload, kb_citations) -> None`
   - `record_failure(eusolicit_run_id, error_message) -> None`
4. **Idempotency under concurrency**: 10 concurrent `start_analysis` calls for the same (opp, type) within 1h → 1 actual run, 10 callers return same `eusolicit_run_id`. Use `INSERT ... ON CONFLICT DO NOTHING RETURNING` or `SELECT ... FOR UPDATE SKIP LOCKED`.
5. **Result-payload schema validation**: `record_completion` validates payload via Pydantic against the analysis_type's structured-output schema (S26.01 + S26.02). Invalid payload → `record_failure` instead with structured error.
6. **EXPLAIN ANALYZE** verifies index hit on idempotency-check query.
7. Schema-isolation: `client_role` has CRUD; verified.

## Dev Notes

### Pattern reuse
- StrEnum: per ADR-012.
- Idempotency via `INSERT ... ON CONFLICT`: see existing E04 amendment `gateway.workflow_runs` for the pattern.
- Cross-schema FK works because `gateway` schema is in the same Postgres database.

### Files likely touched
- `services/client-api/alembic/versions/<next>_opportunity_analyses.py`
- `services/client-api/src/client_api/models/opportunity_analysis.py` (new)
- `services/client-api/src/client_api/services/opportunity_analysis_service.py` (new)
- `services/client-api/tests/unit/services/test_opportunity_analysis_service.py`
- `services/client-api/tests/integration/test_opportunity_analysis_concurrency.py`

### Out of scope
- User-trigger endpoint (S26.05).
- Auto-trigger consumer (S26.06).
- Webhook-persistence completer (S26.07).
- Frontend (S26.08).

## Risks

- **R1**: Cross-schema FK + `migration_role` permissions — verify `gateway.workflow_runs` is readable by `client_role` for FK target reference at migration time.
- **R2**: 1h idempotency window — what if user wants to force? `force=True` parameter handles it.

## Testing

- Unit: idempotency under concurrent dispatch.
- Integration: EXPLAIN ANALYZE; cross-schema FK migration works.

## See also

- Epic file §S26.04
- E04 amendment S04.24 (gateway.workflow_runs reference)
