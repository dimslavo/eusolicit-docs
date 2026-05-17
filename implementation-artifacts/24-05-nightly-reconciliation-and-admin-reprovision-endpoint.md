# Story 24.05: Nightly Reconciliation + Admin Reprovision Endpoint

**Status:** backlog
**Epic:** E24 — SirmaAI Tenant Provisioning
**Points:** 3
**Type:** backend
**Dependencies:** S24.02 (Project creation), S24.03 (template seed)
**Blocks:** none (operational safety surface — but Slice 1 DoD requires it)
**Created:** 2026-05-15
**Source:** E24 epic §S24.05

## Story

As **Operator Ivan**,
I want **a nightly reconciliation job that verifies every active EU Solicit company has a healthy SirmaAI Project AND a one-click "Reprovision" admin endpoint for orphans**,
so that **provisioning failures don't sit invisibly forever and I can recover failed tenants without dev intervention**.

## Acceptance Criteria

1. Celery Beat task `reconcile_sirmaai_projects` runs daily at 03:00 UTC on the `client-api` worker.
2. Reconciler scans all `client.companies` rows where `archived_at IS NULL`. For each:
   - Verify a corresponding `client.sirmaai_projects` row exists with `provisioning_status='provisioned'`.
   - Verify the SirmaAI-side Project still exists via `GET /api/organizations/{orgId}/projects/{projectId}` returns 200. If 404, the Project was deleted server-side (incident or accident) — surface as orphan.
3. Two orphan categories:
   - **EU-Solicit-side orphan**: company exists, no `client.sirmaai_projects` row OR `provisioning_status != 'provisioned'`.
   - **SirmaAI-side orphan**: row exists with `provisioned` status, but SirmaAI returns 404.
4. Both orphan types are persisted to a transient summary table OR Redis Hash for the admin UI:
   - `client.sirmaai_orphans` (TTL-cleared on next reconciler run) with `company_id`, `orphan_type`, `last_attempt_at`, `last_error`.
5. **Prometheus gauges** exported by reconciler completion:
   - `sirmaai_provisioning_orphans_total{type="eu_solicit_side"}`
   - `sirmaai_provisioning_orphans_total{type="sirmaai_side"}`
   - `sirmaai_reconciler_last_run_timestamp_seconds`
6. **Admin API endpoint** `GET /api/v1/admin/sirmaai-projects/orphans` returns the orphan list as JSON. Auth: admin role + VPN per existing admin-api auth pattern.
7. **Admin API endpoint** `POST /api/v1/admin/companies/{id}/reprovision` triggers the `provision_sirmaai_project(company_id)` task (S24.02) idempotently. Returns 202 with `task_id`. If company is not in orphan list, returns 409.
8. **Audit log**: every reprovision-endpoint invocation writes `shared.audit_log` row with admin user_id, target company_id, and orphan reason at trigger time.
9. **Alert rule** (Prometheus): `sirmaai_provisioning_orphans_total > 0` for > 1 hour fires a `ticket`-severity alert via `alertmanager.yaml` Slack receiver (per pe-06). NOT page-severity — orphans are operational, not customer-facing.
10. **Integration test**: synthetic orphan (manually drop a `client.sirmaai_projects` row for a non-archived company) → reconciler runs → orphan appears in admin endpoint → reprovision endpoint clears it → next reconciler run shows 0 orphans.

## Dev Notes

### Pattern reuse
- Celery Beat scheduling: `services/client-api/src/client_api/celery_app.py` BEAT_SCHEDULE block. Pattern from existing `expired_opportunity_cleanup` task.
- Admin-API endpoint pattern: `services/admin-api/src/admin_api/routers/*.py` — copy from tenants router auth shape.
- Reconciler should use the partial index from S24.01 (`WHERE provisioning_status <> 'provisioned'`) for efficient scan.

### Files likely touched
- `services/client-api/src/client_api/tasks/sirmaai_reconciliation.py` (new)
- `services/client-api/src/client_api/celery_app.py` (Beat schedule entry)
- `services/admin-api/src/admin_api/routers/sirmaai_projects.py` (new — orphan list + reprovision)
- `services/client-api/src/client_api/api/v1/admin/companies.py` (extend with reprovision)
- `infra/observability/prometheus/rules/sirmaai-alerts.yaml` (new)
- `services/client-api/tests/integration/test_sirmaai_reconciliation.py`

### Out of scope
- Auto-reprovision on orphan detection — by design, manual operator action (audit-traceable).
- Cross-substrate orphan repair (SirmaAI side has Project that EU Solicit doesn't know about) — that's an admin investigation case, not auto-repair.

## Risks

- **R1**: Reconciler scan O(N) on companies — at 10k companies, the SirmaAI 404-check is 10k HTTP calls. Mitigate: skip the SirmaAI-side check if `provisioning_status='provisioned'` and last_verified_at < 24h (add column if needed).
- **R2**: Race with concurrent provisioning — reconciler sees mid-state during S24.02 retry. Mitigate: reconciler ignores rows with `provisioning_status='pending'` and `created_at > now() - 4h` (within S24.02 retry budget).

## Testing

- Unit: orphan detection logic; reprovision endpoint auth + 409 path.
- Integration: synthetic orphan → reconciler → admin endpoint → reprovision → cleared.
- Performance: reconciler completes in < 10 min for 10k companies.

## See also

- Epic file §S24.05
- PRD amendment FR-45
- E22 onprem-03 (Alertmanager wiring — reconciler alerts feed through it)
