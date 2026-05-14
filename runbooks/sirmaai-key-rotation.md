# Runbook: SirmaAI Per-Project API Key Rotation

**Severity**: SEV-3 (silent data-quality; escalate to SEV-2 if rotation backlog > 500 rows)
**Last updated**: 2026-05-14
**Story**: S04.22 — Per-Project API Key Vault and Rotation

---

## Overview

Each EU Solicit company provisioned onto SirmaAI has a unique bearer token stored
encrypted in `client.sirmaai_projects.api_key_encrypted` (Fernet, key from
`SIRMAAI_FERNET_KEY`).  A Celery Beat task (`sirmaai_rotate_project_keys`) runs
daily and rotates keys older than `SIRMAAI_KEY_ROTATION_INTERVAL_DAYS` (default 90)
days, one batch of up to `SIRMAAI_KEY_ROTATION_BATCH_SIZE` rows (default 100) per
tick.

---

## Prometheus Alerts

| Metric | Description | Threshold |
|--------|-------------|-----------|
| `sirmaai_key_rotations_total{outcome="smoke_test_failed"}` | New key passed creation but failed auth smoke-test | > 0 in 5 min |
| `sirmaai_key_rotations_total{outcome="revoke_failed"}` | Old key not revoked after DB swap | > 5 in 1 h |
| `sirmaai_orphan_keys_total` | Keys created but never persisted | > 0 in 5 min |
| `sirmaai_keys_due_for_rotation` | Rows overdue for rotation | > 200 |

---

## Diagnosing a Stalled Rotation

### 1. Check Celery Beat logs
```bash
docker logs sirmaai-gateway-worker 2>&1 | grep sirmaai_rotate
```
Expected: `sirmaai_rotate_project_keys.start` and `sirmaai_rotate_project_keys.done`
once per day.

### 2. Check the backlog
```sql
SELECT count(*)
FROM client.sirmaai_projects
WHERE api_key_rotated_at < now() - INTERVAL '90 days'
   OR api_key_rotated_at IS NULL;
```

### 3. Check orphan stash (crashed rotations)
```sql
SELECT company_id, sirmaai_previous_key_ref_id, updated_at
FROM client.sirmaai_projects
WHERE sirmaai_previous_key_ref_id IS NOT NULL
ORDER BY updated_at DESC
LIMIT 20;
```
Rows with a non-NULL `sirmaai_previous_key_ref_id` indicate a rotation that
crashed between the DB swap and the old-key revoke.  The next automatic tick
will retry the deferred revoke (recovery preamble, AC8).  If the SirmaAI API is
reachable and the preamble still fails, escalate to SirmaAI support with the
`sirmaai_previous_key_ref_id` value(s).

---

## Manual Key Rotation (Single Company)

Use the admin endpoint exposed when `SIRMAAI_GATEWAY_ENABLED=true`:

```bash
curl -s -X POST https://<host>/admin/sirmaai/rotate-key \
  -H "Content-Type: application/json" \
  -d '{"company_id": "<uuid>"}' | jq .
```

Expected response (HTTP 200):
```json
{
  "company_id": "...",
  "sirmaai_project_id": "...",
  "rotated_at": "2026-05-14T12:00:00+00:00",
  "smoke_test_passed": true,
  "old_key_revoked": true
}
```

| HTTP | Meaning |
|------|---------|
| 200 | Rotation succeeded |
| 404 | Company not provisioned (`TenantNotProvisionedError`) |
| 409 | Row locked by concurrent rotation — retry in 30 s |
| 502 | SirmaAI smoke-test failed — check SirmaAI API status |

---

## Orphan-Key Cleanup

If `sirmaai_orphan_keys_total` is non-zero, keys were created on SirmaAI but never
persisted in the DB (smoke-test failure during rotation).  These keys are not live and
not referenced in the DB.

1. Identify the SirmaAI project IDs from the rotation logs:
   ```bash
   docker logs sirmaai-gateway-worker 2>&1 | grep smoke_test_failed | jq '{key_ref_id, sirmaai_project_id}'
   ```
2. Revoke the orphan keys via the SirmaAI admin dashboard or API:
   `DELETE /client/api/v1/projects/{projectRefId}/keys/{keyRefId}`
3. Reset the orphan counter (it resets on worker restart; Prometheus counter is
   cumulative — check for new increments rather than absolute value).

---

## Alert Thresholds (recommended)

```yaml
# prometheus/rules/sirmaai-key-rotation-rules.yaml
- alert: SirmaAIKeyOrphan
  expr: increase(sirmaai_orphan_keys_total[5m]) > 0
  severity: warning
  annotations:
    summary: "SirmaAI orphan key created — smoke-test failed during rotation"

- alert: SirmaAIKeyRotationBacklog
  expr: sirmaai_keys_due_for_rotation > 200
  severity: warning
  annotations:
    summary: "{{ $value }} SirmaAI keys are overdue for rotation"

- alert: SirmaAIKeyRevokeFailed
  expr: increase(sirmaai_key_rotations_total{outcome="revoke_failed"}[1h]) > 5
  severity: warning
  annotations:
    summary: "Old SirmaAI keys not being revoked — check SirmaAI API"
```

---

## Related Runbooks

- `kraftdata-outage.md` — circuit-breaker patterns that also apply to SirmaAI calls
- `redis-failover.md` — Celery broker and key-rotated event stream share Redis
