# Runbook: AgenticSAI Per-Project API Key Rotation

**Severity**: SEV-3 (silent data-quality; escalate to SEV-2 if rotation backlog > 500 rows)
**Last updated**: 2026-05-14
**Story**: S04.22 — Per-Project API Key Vault and Rotation

---

## Overview

Each EU Solicit company provisioned onto AgenticSAI has a unique bearer token stored
encrypted in `client.agenticsai_projects.api_key_encrypted` (Fernet, key from
`AGENTICSAI_FERNET_KEY`).  A Celery Beat task (`agenticsai_rotate_project_keys`) runs
daily and rotates keys older than `AGENTICSAI_KEY_ROTATION_INTERVAL_DAYS` (default 90)
days, one batch of up to `AGENTICSAI_KEY_ROTATION_BATCH_SIZE` rows (default 100) per
tick.

---

## Prometheus Alerts

| Metric | Description | Threshold |
|--------|-------------|-----------|
| `agenticsai_key_rotations_total{outcome="smoke_test_failed"}` | New key passed creation but failed auth smoke-test | > 0 in 5 min |
| `agenticsai_key_rotations_total{outcome="revoke_failed"}` | Old key not revoked after DB swap | > 5 in 1 h |
| `agenticsai_orphan_keys_total` | Keys created but never persisted | > 0 in 5 min |
| `agenticsai_keys_due_for_rotation` | Rows overdue for rotation | > 200 |

---

## Diagnosing a Stalled Rotation

### 1. Check Celery Beat logs
```bash
docker logs agenticsai-gateway-worker 2>&1 | grep agenticsai_rotate
```
Expected: `agenticsai_rotate_project_keys.start` and `agenticsai_rotate_project_keys.done`
once per day.

### 2. Check the backlog
```sql
SELECT count(*)
FROM client.agenticsai_projects
WHERE api_key_rotated_at < now() - INTERVAL '90 days'
   OR api_key_rotated_at IS NULL;
```

### 3. Check orphan stash (crashed rotations)
```sql
SELECT company_id, agenticsai_previous_key_ref_id, updated_at
FROM client.agenticsai_projects
WHERE agenticsai_previous_key_ref_id IS NOT NULL
ORDER BY updated_at DESC
LIMIT 20;
```
Rows with a non-NULL `agenticsai_previous_key_ref_id` indicate a rotation that
crashed between the DB swap and the old-key revoke.  The next automatic tick
will retry the deferred revoke (recovery preamble, AC8).  If the AgenticSAI API is
reachable and the preamble still fails, escalate to AgenticSAI support with the
`agenticsai_previous_key_ref_id` value(s).

---

## Manual Key Rotation (Single Company)

Use the admin endpoint exposed when `AGENTICSAI_GATEWAY_ENABLED=true`:

```bash
curl -s -X POST https://<host>/admin/agenticsai/rotate-key \
  -H "Content-Type: application/json" \
  -d '{"company_id": "<uuid>"}' | jq .
```

Expected response (HTTP 200):
```json
{
  "company_id": "...",
  "agenticsai_project_id": "...",
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
| 502 | AgenticSAI smoke-test failed — check AgenticSAI API status |

---

## Orphan-Key Cleanup

If `agenticsai_orphan_keys_total` is non-zero, keys were created on AgenticSAI but never
persisted in the DB (smoke-test failure during rotation).  These keys are not live and
not referenced in the DB.

1. Identify the AgenticSAI project IDs from the rotation logs:
   ```bash
   docker logs agenticsai-gateway-worker 2>&1 | grep smoke_test_failed | jq '{key_ref_id, agenticsai_project_id}'
   ```
2. Revoke the orphan keys via the AgenticSAI admin dashboard or API:
   `DELETE /client/api/v1/projects/{projectRefId}/keys/{keyRefId}`
3. Reset the orphan counter (it resets on worker restart; Prometheus counter is
   cumulative — check for new increments rather than absolute value).

---

## Alert Thresholds (recommended)

```yaml
# prometheus/rules/agenticsai-key-rotation-rules.yaml
- alert: AgenticSAIKeyOrphan
  expr: increase(agenticsai_orphan_keys_total[5m]) > 0
  severity: warning
  annotations:
    summary: "AgenticSAI orphan key created — smoke-test failed during rotation"

- alert: AgenticSAIKeyRotationBacklog
  expr: agenticsai_keys_due_for_rotation > 200
  severity: warning
  annotations:
    summary: "{{ $value }} AgenticSAI keys are overdue for rotation"

- alert: AgenticSAIKeyRevokeFailed
  expr: increase(agenticsai_key_rotations_total{outcome="revoke_failed"}[1h]) > 5
  severity: warning
  annotations:
    summary: "Old AgenticSAI keys not being revoked — check AgenticSAI API"
```

---

## Related Runbooks

- `agenticsai-outage.md` — circuit-breaker patterns that also apply to AgenticSAI calls
- `redis-failover.md` — Celery broker and key-rotated event stream share Redis
