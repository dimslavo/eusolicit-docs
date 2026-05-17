# Story 24.08: One-Off Backfill — Provision SirmaAI Projects for Existing Companies

**Status:** backlog
**Epic:** E24 — SirmaAI Tenant Provisioning
**Points:** 3
**Type:** backend + ops
**Dependencies:** S24.01..S24.06 done
**Blocks:** Sprint-5 launch hygiene (all existing tenants must be provisioned before declaring E24 complete)
**Created:** 2026-05-15
**Source:** E24 epic §S24.08

## Story

As an **operator running the SirmaAI cutover**,
I want **a manually-triggered, throttled, idempotent backfill task that provisions SirmaAI Projects for every non-archived existing company**,
so that **the post-pivot platform has zero pre-existing-tenant orphans, and the cutover doesn't depend on customer-driven re-registration**.

## Acceptance Criteria

1. Admin endpoint `POST /api/v1/admin/sirmaai-projects/backfill` triggers a one-off Celery task `backfill_sirmaai_projects()` on the client-api worker. Endpoint is admin-only + VPN-gated.
2. The task enumerates `client.companies` WHERE `archived_at IS NULL` AND no corresponding `client.sirmaai_projects` row exists. Iterates the list.
3. For each company, invokes the SAME standard provisioning flow as S24.02 (no separate code path — call `provision_sirmaai_project(company_id)` task synchronously per iteration).
4. **Throttle**: 1 concurrent provisioning per minute (configurable via env var `SIRMAAI_BACKFILL_THROTTLE_SECONDS=60`). Prevents SirmaAI rate-limit storm.
5. **Audit row per successful provisioning** with `action_type='backfill_provisioned'`, `details->>'company_id'={cid}`.
6. **Failure during one company does NOT block others**: per-company exception is caught, logged, surfaced via existing S24.05 orphan list. Backfill continues.
7. **Pre-flight admin endpoint** `GET /api/v1/admin/sirmaai-projects/backfill/preflight` returns `{eligible_companies: N, estimated_duration_minutes: M}` without side effects. `M = N * throttle_seconds / 60`.
8. **Idempotency**: re-triggering the backfill mid-progress resumes — eligible-companies query naturally excludes already-provisioned tenants. Crash-recovery is automatic.
9. **Runbook** at `eusolicit-docs/runbooks/sirmaai-backfill.md` covering:
   - Pre-flight check (verify SirmaAI quota will accommodate N provisionings).
   - Throttle override flag for staging speed-runs.
   - Monitoring during backfill (Prometheus gauges from S24.05 + S24.02).
   - Failure-recovery procedure (manual reprovision via S24.05 endpoint).
10. **Staging dry-run**: runbook procedure tested on staging with a synthetic 50-company batch before production cutover.

## Dev Notes

### Pattern reuse
- Reuse the `provision_sirmaai_project(company_id)` task from S24.02 — DO NOT fork the provisioning code path.
- Throttle pattern: `time.sleep(throttle_seconds)` between iterations within a single Celery task. Single task instance enforces serialisation; no need for distributed locking.
- Admin endpoint pattern: see existing admin-api routers.

### Files likely touched
- `services/admin-api/src/admin_api/routers/sirmaai_backfill.py` (new)
- `services/client-api/src/client_api/tasks/sirmaai_backfill.py` (new — Celery task wrapping iteration)
- `eusolicit-docs/runbooks/sirmaai-backfill.md` (new)
- `services/client-api/tests/integration/test_sirmaai_backfill.py`

### Out of scope
- Auto-trigger on schedule (this is one-off; reconciler handles steady-state).
- UI for backfill progress — operator runs from CLI / admin API directly.

## Risks

- **R1**: SirmaAI rate-limit despite throttle — operator can pause via env var or restart task.
- **R2**: Long-running task (50 companies × 60s = 50 min; 5000 × 60s = ~3.5 days) — split across multiple operator-supervised windows if needed.

## Testing

- Unit: eligibility-query correctness; throttle observance.
- Integration: synthetic 5-company batch on staging; all 5 provisioned.
- Operator drill: 50-company staging run; populate runbook §Drill Results.

## See also

- Epic file §S24.08 (closes readiness Concern #6)
- S24.02 (provisioning task this backfill reuses)
- S24.05 (orphan surfacing for failed backfill entries)
- IR remediation Concern #6 (which this story closes)
