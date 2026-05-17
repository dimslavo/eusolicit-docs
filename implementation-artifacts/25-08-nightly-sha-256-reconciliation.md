# Story 25.08: Nightly SHA-256 Reconciliation

**Status:** backlog
**Epic:** E25 — Knowledge Base Lifecycle
**Points:** 3
**Type:** backend
**Dependencies:** S25.02
**Blocks:** none (data-integrity layer; doesn't block Slice 2 DoD)
**Created:** 2026-05-15
**Source:** E25 epic §S25.08

## Story

As **Operator Ivan**,
I want **a nightly cross-substrate hash reconciliation that flags any KB file whose SirmaAI-side hash differs from EU Solicit's recorded hash**,
so that **silent corruption or tampering is detected within 24 hours and not at the moment a customer notices their agent output is wrong**.

## Acceptance Criteria

1. Celery Beat task `reconcile_kb_sha256` runs daily at 04:00 UTC on the `client-api` worker.
2. Scans `client.sirmaai_kb_files` WHERE `archived_at IS NULL`. For each, queries SirmaAI for the file's hash (via SirmaAI metadata endpoint).
3. **Mismatch handling**:
   - Write `shared.audit_log` row with `action_type='kb_hash_mismatch'`, `details->>'kb_file_id'`, `details->>'expected_sha256'`, `details->>'actual_sha256'`.
   - Increment Prometheus gauge `sirmaai_kb_hash_mismatches_total{company_id}`.
   - Emit admin alert via `notification.admin_alerts` Redis Stream.
4. **Performance**: completes within 10 minutes for 10K files across 100 tenants (load-test baseline).
5. **Concurrency**: process files in batches (e.g. 100 at a time) with bounded parallel HTTP requests (10 concurrent) to avoid SirmaAI rate-limit.
6. **Idempotency**: re-running the task multiple times in a day is safe — it's read-only on EU Solicit side; only writes audit + metric on mismatch.
7. **Alert rule** (Prometheus): `sirmaai_kb_hash_mismatches_total > 0` for > 1 hour fires `ticket`-severity alert via Slack receiver.
8. **Integration test**: insert a synthetic mismatch (modify `client.sirmaai_kb_files.sha256` directly to wrong value) → run reconciler → audit row + gauge increment observed.

## Dev Notes

### Pattern reuse
- Celery Beat: existing pattern in `services/client-api/src/client_api/celery_app.py`.
- Bounded-parallel HTTP: `asyncio.Semaphore(10)`.

### Files likely touched
- `services/client-api/src/client_api/tasks/sirmaai_kb_reconciliation.py` (new)
- `services/client-api/src/client_api/celery_app.py` (Beat schedule entry)
- `services/sirmaai-gateway/src/sirmaai_gateway/clients/storage_client.py` (extend with `get_file_metadata`)
- `infra/observability/prometheus/rules/sirmaai-alerts.yaml` (extend)
- `services/client-api/tests/integration/test_kb_sha256_reconciliation.py`

### Out of scope
- Auto-remediation on mismatch (operator decides — could mean tampering, could mean SirmaAI re-encoded the file).
- File-content reconciliation (only metadata hash — body re-download to verify would be too expensive).

## Risks

- **R1**: SirmaAI metadata API may not expose hash — coordinate with SirmaAI team; if not available, skip this story or downgrade to file-existence-only check.
- **R2**: Large fleet → 10-min budget may be exceeded; tune batch size and concurrency.

## Testing

- Unit: mismatch detection; alert path.
- Integration: synthetic mismatch.
- Performance: 7-day rolling load test.

## See also

- Epic file §S25.08
- PRD amendment FR-52
