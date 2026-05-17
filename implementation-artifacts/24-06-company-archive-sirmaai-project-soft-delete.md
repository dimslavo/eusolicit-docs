# Story 24.06: Company Archive → SirmaAI Project Soft-Delete + Key Revocation

**Status:** backlog
**Epic:** E24 — SirmaAI Tenant Provisioning
**Points:** 2
**Type:** backend
**Dependencies:** S24.02, S24.04 (MCP stubs to clean up)
**Blocks:** S24.07 (erasure flow consumes this), S17.36 (workspace archive MCP secret deletion); FR-46 closure
**Created:** 2026-05-15
**Source:** E24 epic §S24.06

## Story

As a **company admin archiving my workspace**,
I want **the corresponding SirmaAI Project to be soft-deleted and its API key revoked within 24 hours**,
so that **my proprietary data, agent traces, and KB content are no longer accessible after I close the account**.

## Acceptance Criteria

1. Subscriber on existing Redis Stream `company.archived` (already published by company-archive flow): on each event, dispatch Celery task `archive_sirmaai_project(company_id)`.
2. The archive task executes in this order:
   - (a) Revoke api-key via `DELETE /api/organizations/{orgId}/keys/{keyId}` using the encrypted key from `client.sirmaai_projects.api_key_encrypted` (decrypt → call → discard plaintext).
   - (b) For each `client.sirmaai_mcp_servers` row associated with company_id: delete MCP-server secrets server-side via `DELETE /api/organizations/{orgId}/projects/{projectId}/mcp-servers/{id}/secrets`. Set `client.sirmaai_mcp_servers.status='inactive'`.
   - (c) Soft-delete the SirmaAI Project via `DELETE /api/organizations/{orgId}/projects/{projectId}` (SirmaAI-side soft-delete).
   - (d) Set `client.sirmaai_projects.archived_at = now()` and `provisioning_status='archived'`.
3. **24-hour SLA**: from `company.archived` event → all 4 steps complete with `archived_at` set ≤ 24h. Measured via Prometheus histogram `sirmaai_archive_duration_seconds`.
4. **Audit log**: each of the 4 steps writes a `shared.audit_log` row with `action_type='sirmaai_archive_step_{key|mcp|project|state}'`.
5. **Failure path**: if any step fails (e.g. SirmaAI 500 on Project delete), DO NOT mark `archived_at` or transition status to `archived`. Instead:
   - Increment retry counter; exponential-backoff retry (1h, 4h, 12h, 24h).
   - After 4 failed attempts, set `provisioning_status='failed'` (with a clear context flag `archive_failed=true` in `last_error`), publish admin alert, and DO NOT block the company-archive flow itself (EU Solicit-side archive can proceed; SirmaAI cleanup is async).
6. **Idempotency**: re-running the task on an already-archived Project (status='archived') is a no-op.
7. **Cross-tenant**: tenant A's archive doesn't affect tenant B's MCP-server rows or Project state. Verified via integration test.
8. **Reconciler interop**: archived companies are exempt from S24.05 reconciler — `archived_at IS NOT NULL` filter applied.

## Dev Notes

### Pattern reuse
- Redis Stream subscriber: copy pattern from `services/notification/src/notification/consumers/subscription_changed.py`.
- Fernet decryption for the API key: `eusolicit_common.crypto.fernet.decrypt`.
- Exponential backoff: Celery `autoretry_for=(SirmaAITransientError,)` + `retry_backoff=True`.

### Files likely touched
- `services/client-api/src/client_api/tasks/sirmaai_archival.py` (new)
- `services/client-api/src/client_api/consumers/company_archived.py` (new)
- `services/sirmaai-gateway/src/sirmaai_gateway/clients/admin_client.py` (extend with `revoke_key`, `delete_mcp_server_secrets`, `delete_project`)
- `services/client-api/tests/integration/test_sirmaai_archival_flow.py`

### Out of scope
- Right-to-erasure cross-substrate proof (S24.07 builds on this; this story only covers the company-archive lifecycle).
- CRM OAuth-token revocation at provider (S17.36 — separate story for the provider-side revoke).
- Frontend confirmation modal (UX amendment §5.15).

## Risks

- **R1**: SirmaAI Project delete may have a quirk where in-flight agent runs are dropped — accept (already-archiving company should not be using agents). Document as known behavior.
- **R2**: API key revoke before Project delete leaves a brief window where the Project exists but is uncallable — fine for archive (no further calls expected).

## Testing

- Unit: 4-step ordering, idempotency, failure-mode retry budget.
- Integration: archive flow end-to-end with mocked SirmaAI.
- Cross-tenant: 2 companies, archive one, verify the other's state unchanged.

## See also

- Epic file §S24.06
- PRD amendment FR-46
- S24.07 (erasure builds on this)
- S17.36 (workspace archive MCP secret deletion at provider side)
