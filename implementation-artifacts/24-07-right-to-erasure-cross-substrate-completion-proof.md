# Story 24.07: Right-to-Erasure Cross-Substrate Completion Proof

**Status:** backlog
**Epic:** E24 — SirmaAI Tenant Provisioning
**Points:** 3
**Type:** backend
**Dependencies:** S24.06 (Project archive), S25.01 (`client.sirmaai_kb_files` schema), S25.05 (KB lifecycle)
**Blocks:** FR-56 closure (numbered FR added 2026-05-15)
**Created:** 2026-05-15
**Source:** E24 epic §S24.07; PRD amendment FR-56

## Story

As a **Data Protection Officer at a customer company invoking GDPR Article 17 (right-to-erasure)**,
I want **EU Solicit to delete my data across every substrate it has reached (EU Solicit Postgres, SirmaAI Project, SirmaAI KB files, MCP-server secrets, CRM tokens at provider) and produce a two-ACK audit-log certificate of completion**,
so that **my erasure request is provably complete and EU Solicit can defend GDPR Art. 17 compliance under audit**.

## Acceptance Criteria

1. Extend existing GDPR erasure flow (from Epic 1/9) to chain the SirmaAI cleanup AFTER the Postgres-row deletion path.
2. After Postgres rows are erased for company_id, write `shared.audit_log` row with `action_type='erasure_step'`, `details->>'erasure_step'='postgres_rows_deleted'`, `details->>'company_id'=company_id`, `details->>'rows_count'=N`.
3. Iterate `client.sirmaai_kb_files` rows for company_id (those WHERE `archived_at IS NULL` AND those WHERE `archived_at IS NOT NULL`):
   - For each, call `DELETE /storage-resources/{vector_store_id}/files/{sirmaai_file_id}` via `sirmaai-gateway`.
   - Track per-file deletion outcome.
4. After all KB file deletes complete (or after S24.06 Project archive completes for the SirmaAI-side cleanup), write `shared.audit_log` row with `action_type='erasure_step'`, `details->>'erasure_step'='sirmaai_files_deleted'`, `details->>'company_id'=company_id`, `details->>'files_count'=N`, `details->>'project_archived'=true|false`.
5. **Erasure certificate** is the audit-log query: `SELECT * FROM shared.audit_log WHERE details->>'company_id'='{cid}' AND action_type='erasure_step' ORDER BY timestamp` returning BOTH ACK rows (`postgres_rows_deleted` AND `sirmaai_files_deleted`).
6. **Erasure not certified** until both ACKs are present. An admin endpoint `GET /api/v1/admin/companies/{id}/erasure-status` returns `{status: 'in_progress'|'certified', acks: [...]}`.
7. **Nightly sweep job** scans erasure attempts older than 48h that have only one ACK (`postgres_rows_deleted` present, `sirmaai_files_deleted` missing) and surfaces them as `sirmaai_erasure_stale_total` Prometheus gauge with an alertmanager rule (severity=ticket).
8. **Negative test**: SirmaAI ACK never written (simulate by mocking the gateway client to throw) → erasure status remains `in_progress` indefinitely; alerts fire after 48h.
9. **Performance**: erasure-status admin query must return < 500ms on a `shared.audit_log` table with 10M rows. Verify EXPLAIN ANALYZE shows index hit, NOT Seq Scan.
10. **GIN index** (or expression index on `details->>'erasure_step'`) added to `shared.audit_log` if EXPLAIN ANALYZE shows Seq Scan on the sweep query. Migration is part of this story.

## Dev Notes

### Pattern reuse
- Existing GDPR erasure flow: `services/client-api/src/client_api/services/gdpr_erasure.py` (or wherever Epic 1/9 landed it — search for "erasure" or "right_to_be_forgotten").
- `shared.audit_log` write helper: `eusolicit_common.services.audit_service.log_event()`.
- Index addition migration: Alembic op `op.create_index(... postgresql_using='gin')` for the expression index.

### Files likely touched
- `services/client-api/src/client_api/services/gdpr_erasure.py` (extend with SirmaAI chain)
- `services/client-api/src/client_api/tasks/sirmaai_erasure.py` (new)
- `services/admin-api/src/admin_api/routers/erasure.py` (new — admin endpoint + sweep status)
- `services/client-api/alembic/versions/<next>_audit_log_erasure_index.py` (new — GIN/expression index)
- `services/client-api/tests/integration/test_gdpr_erasure_cross_substrate.py`

### Out of scope
- CRM OAuth token revocation at provider (S17.36 handles)
- Workspace-archive flow distinction (S24.06 handles archive; erasure is a STRICTER subset — Postgres rows are deleted, not just soft-deleted)
- Self-service erasure-request UI (admin-only for now per GDPR-Art-17 practice)

## Risks

- **R1**: A long-archived company (archived_at < erasure_request_at) — Project is already soft-deleted but KB files may persist. Erasure must explicitly call delete on files even if Project is archived.
- **R2**: SirmaAI delete returns 404 on already-deleted files — treat as success.
- **R3**: Partial deletion (5 of 50 files deleted, then SirmaAI 500) — sweep + manual operator retry path required.

## Testing

- Unit: ACK ordering, sweep query correctness.
- Integration: full erasure run with 10 KB files mocked → both ACKs present.
- Performance: EXPLAIN ANALYZE on admin endpoint query.

## See also

- Epic file §S24.07
- PRD amendment FR-56 (numbered cross-substrate erasure FR)
- IR remediation Concern #9 (performance gate on audit_log query)
