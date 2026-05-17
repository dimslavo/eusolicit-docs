# Story 26.05: User-Trigger Endpoint POST /opportunities/{id}/analyze

**Status:** backlog
**Epic:** E26 — Agent-Driven Ingestion & Analysis
**Points:** 3
**Type:** backend
**Dependencies:** S26.03 (workflow), S26.04 (analyses schema + service)
**Blocks:** S26.08 (UI consumes this)
**Created:** 2026-05-15
**Source:** E26 epic §S26.05

## Story

As **Elena viewing an opportunity in her tenant**,
I want **a "Run analysis" button that fires off qualification (or quantification, Pro+) on demand and lets the UI show progress**,
so that **I can re-run analysis after profile updates or get quantification on demand without waiting for the auto-cascade**.

## Acceptance Criteria

1. **Endpoint** `POST /api/v1/opportunities/{id}/analyze` (`client-api`) accepts body `{analysis_type: "qualification" | "quantification" | "both", force?: boolean=false}`. Auth: workspace-scoped (existing scope-check middleware).
2. **Tier gating**: `quantification` or `both` requires Pro+ tier (TierGate Depends). Free / Starter get 402 with upgrade CTA.
3. **Idempotency**: `force=false` returns existing completed analysis if < 1h old (per S26.04 service). Returns 200 with `cached=true`.
4. **Trigger flow**: call `sirmaai_gateway.trigger_workflow("opportunity-analysis-v1", projectId, opportunityId, analysis_type, force, eusolicit_run_id)` — generates `eusolicit_run_id` per analysis_type; persists pending `client.opportunity_analyses` row(s).
5. **Response**: 202 Accepted with body `{run_ids: {qualification: "uuid", quantification: "uuid"|null}, cached: false, message: "Analysis dispatched."}`. Returns within 200ms p95 (no blocking wait).
6. **Workspace scope**: cannot analyze an opportunity the caller's company doesn't have access to. Cross-tenant access returns 404.
7. **Rate-limit**: per-tenant max 50 concurrent in-flight analyses (atomic Redis Lua); 429 with Retry-After on exceedance.
8. **Audit log**: each trigger writes `shared.audit_log` row with `action_type='analysis_triggered'`, details include user_id, opportunity_id, analysis_type, force.
9. **Polling endpoint** `GET /api/v1/opportunities/{id}/analyses` returns the latest analyses per type with status + result_payload (if completed).

## Dev Notes

### Pattern reuse
- Workspace-scope auth: existing middleware.
- TierGate Depends: existing.
- Atomic Redis Lua rate-limit: Epic 6 pattern.

### Files likely touched
- `services/client-api/src/client_api/routers/opportunities.py` (extend with `/analyze` and `/analyses`)
- `services/client-api/src/client_api/services/opportunity_analysis_service.py` (extend with `trigger_via_workflow`)
- `services/sirmaai-gateway/src/sirmaai_gateway/services/workflow_trigger.py` (already exposes `trigger_workflow`)
- `services/client-api/tests/integration/test_opportunity_analyze_endpoint.py`

### Out of scope
- Auto-trigger on ingestion (S26.06).
- SSE stream for progress (S26.07).
- Frontend (S26.08).

## Risks

- **R1**: Bulk-trigger abuse — rate-limit handles; consider exposing a "bulk-analyze-saved-opportunities" admin endpoint post-launch.

## Testing

- Unit: tier-gate + scope-check + idempotency.
- Integration: 50 concurrent triggers → only 1 dispatch, 50 receive same run_id.
- Cross-tenant: tenant A triggering tenant B's opportunity → 404.

## See also

- Epic file §S26.05
- PRD amendment FR-47, FR-48
- S26.04 (idempotent service)
