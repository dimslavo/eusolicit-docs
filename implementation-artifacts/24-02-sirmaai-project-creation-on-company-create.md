# Story 24.02: SirmaAI Project Creation on Company Create

**Status:** backlog
**Epic:** E24 — SirmaAI Tenant Provisioning
**Points:** 5
**Type:** backend
**Dependencies:** S24.01 (schema + state machine) done
**Blocks:** S24.03 (template seed), S24.04 (MCP stubs), S24.05 (reconciliation), S24.06 (archive), S24.08 (backfill); E25 S25.02 (KB upload needs Project to exist)
**Created:** 2026-05-15
**Source:** `eusolicit-docs/planning-artifacts/epics/E24-sirmaai-tenant-provisioning.md` §S24.02

## Story

As a **new company admin who just signed up**,
I want **my SirmaAI Project + per-tenant API key to be created automatically within 30 seconds**,
so that **AI features (qualification, drafting, KB) are available to my team without manual operator intervention**.

## Acceptance Criteria

1. Hook into the existing `client.companies` create flow (post-commit of `register` endpoint): dispatch Celery task `provision_sirmaai_project(company_id)` to the `client-api` worker queue.
2. The Celery task body executes in this order:
   - (a) `POST /api/organizations/{orgId}/projects` with body `{name: "eusolicit-company-{company_id}", description: "EU Solicit company '{display_name}' created at {created_at}"}` via `sirmaai-gateway` admin client. On success, capture `sirmaai_project_id`.
   - (b) `POST /api/organizations/{orgId}/keys` to issue a Project-scoped api-key.
   - (c) **Fernet-encrypt** the api-key (using `eusolicit_common.crypto.fernet.encrypt`) and persist to `client.sirmaai_projects.api_key_encrypted`.
   - (d) Insert `client.sirmaai_projects` row with `provisioning_status='pending'` BEFORE step (a); update to `provisioned` after step (c).
3. Provisioning runs in the background — the company-create API response does NOT block on SirmaAI calls. Response returns within existing latency budget; provisioning_status visible via `GET /api/v1/companies/{id}` returns `pending` until task completes.
4. **30-second p95 SLA**: company.created event → `provisioning_status='provisioned'` ≤ 30s under steady state. Measured via Prometheus histogram `sirmaai_provisioning_duration_seconds`.
5. **Idempotency**: re-invocation of `provision_sirmaai_project(company_id)` with the same `company_id` is a no-op when `provisioning_status='provisioned'`. Verified via concurrent-dispatch test.
6. **Exponential-backoff retry** on transient SirmaAI failures (HTTP 5xx, network timeout): attempts at 1m, 5m, 30m, 2h, 4h. After 5 failed attempts, set `provisioning_status='failed'` + `last_error` + emit admin alert via Redis Stream `notification.admin_alerts`.
7. **Cross-tenant negative**: company A's provisioning failure does NOT block or delay company B's provisioning. Verified via integration test that interleaves two company creates with one mocked SirmaAI failure.
8. **Partial-state recovery**: if step (a) succeeds but (b) fails, retry must NOT create a duplicate Project — check `sirmaai_project_id` is null before re-calling step (a). Idempotency key: `company_id`.
9. **Failure-mode coverage** in unit tests: SirmaAI API key revoked mid-provisioning, SirmaAI 429 rate-limit mid-provisioning, network timeout on each of (a)/(b)/(c).
10. Audit log: every transition (`pending → provisioned`, `pending → failed`, retry attempts) writes `shared.audit_log` row.

## Dev Notes

### Pattern reuse
- `sirmaai-gateway` admin client: `services/sirmaai-gateway/src/sirmaai_gateway/clients/admin_client.py` (existing per S04.20 rename — leverage the `OrganizationApi` class for projects + keys).
- Celery task pattern: copy from `services/client-api/src/client_api/tasks/email_verification.py`.
- Fernet encryption: `packages/eusolicit-common/src/eusolicit_common/crypto/fernet.py`.
- Exponential backoff: Celery's `autoretry_for=(SirmaAITransientError,)` + `retry_backoff=True` + `retry_kwargs={'max_retries': 5}`.

### Files likely touched
- `services/client-api/src/client_api/tasks/sirmaai_provisioning.py` (new)
- `services/client-api/src/client_api/services/auth_service.py` — hook into post-commit of register
- `services/client-api/src/client_api/celery_app.py` — register new task
- `services/sirmaai-gateway/src/sirmaai_gateway/clients/admin_client.py` — extend with `create_project`, `create_project_key`
- `services/client-api/tests/unit/tasks/test_sirmaai_provisioning.py` (new)
- `services/client-api/tests/integration/test_sirmaai_provisioning_flow.py` (new)

### Prometheus metrics to add
- `sirmaai_provisioning_duration_seconds` (Histogram, no labels — single per-task path)
- `sirmaai_provisioning_attempts_total` (Counter, labels: `outcome ∈ {success, transient_failure, terminal_failure}`)
- `sirmaai_provisioning_failures_total` (Counter, labels: `step ∈ {project_create, key_create, key_encrypt}`)

### Out of scope
- Template seeding (S24.03)
- MCP stub registration (S24.04)
- Reconciler (S24.05) — that catches steady-state orphans; this story handles synchronous-best-effort
- Admin reprovision endpoint (S24.05)

## Risks

- **R1**: SirmaAI rate-limit during signup surge — exponential backoff + reconciler-as-safety-net (S24.05) covers it. No need to throttle company creates.
- **R2**: Fernet key rotation during provisioning — Fernet supports multi-key mode; ensure encryption uses current key only.
- **R3**: Network partition during step (c) (encrypt) — purely local op; unlikely. Still wrap in try/except.

## Testing

- Unit: each transient + terminal failure mode mocked; idempotency under concurrent dispatch; retry-count increments correctly.
- Integration: real Celery worker + mocked SirmaAI returning success + delay; assert `provisioning_status='provisioned'` within timeout.
- Cross-tenant: 2 concurrent company creates, one mocked to fail; both retry/converge correctly.
- Performance: provisioning histogram p95 < 30s on staging baseline.

## See also

- Epic file §S24.02
- PRD amendment FR-45
- S24.05 (reconciliation) — pairs with this story to provide steady-state convergence
