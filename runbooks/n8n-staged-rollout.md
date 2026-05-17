# N8N Staged Rollout — Operator Runbook

**Story:** 5.24 — N8N Staged Rollout Enforcement  
**Service:** admin-api (management), sirmaai-gateway (evaluation)  
**Table:** `client.feature_flags`  
**Last updated:** 2026-05-15

---

## Overview

EU Solicit uses a **feature flag gate** in every N8N workflow to control which
tenants receive a new workflow version (e.g. `crawl-aop-v2`).  When a workflow
starts, its first node calls:

```
GET http://sirmaai-gateway:8004/api/internal/sirmaai/feature-flags/check
    ?company_id=<UUID>&flag_name=<flag>
    Authorization: X-Internal-Secret: <N8N_INTERNAL_SECRET>
```

If the response is `{"enabled": true}` the workflow proceeds; otherwise it
records a `skipped_by_feature_flag` event and exits cleanly.

Three rollout tiers are supported:

| Tier | How it works |
|------|-------------|
| **Canary** | Create a flag row for each canary `company_id` with `is_active=true`, `rollout_percentage=null`. Only those exact tenants are enabled. |
| **Percentage** | Set `rollout_percentage=10` (or any 1–99). The bucket is deterministic: `md5(company_id.bytes)[:4] % 100 < pct`. The same company always lands in or out. |
| **Full rollout** | Set `rollout_percentage=100` (or delete the flag, then platform-wide default handling applies). |

---

## Prerequisites

- Admin API JWT for the `platform_admin` role (`ADMIN_API_JWT_SECRET`).
- `company_id` UUIDs of target tenants (from admin UI or DB query).
- N8N internal secret configured: `N8N_INTERNAL_SECRET` env var on sirmaai-gateway.

---

## Step 1 — Canary rollout (specific tenants)

Create one flag row per canary tenant:

```bash
# Replace <ADMIN_JWT> and <COMPANY_UUID> with real values.
curl -s -X POST http://admin-api:8002/api/v1/admin/feature-flags \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "company_id": "<COMPANY_UUID>",
    "flag_name": "n8n_workflow_crawl_aop_v2",
    "is_active": true,
    "rollout_percentage": null
  }'
```

Verify the flag is evaluated correctly:

```bash
curl -s "http://sirmaai-gateway:8004/api/internal/sirmaai/feature-flags/check\
?company_id=<COMPANY_UUID>&flag_name=n8n_workflow_crawl_aop_v2" \
  -H "X-Internal-Secret: <N8N_INTERNAL_SECRET>"
# Expected: {"enabled": true}
```

Non-canary companies return `{"enabled": false}` because no flag row exists for them.

---

## Step 2 — Percentage-based rollout (10% → 50% → 100%)

### 2a. Identify which companies are in the target bucket

The bucket for a company is: `md5(company_id.bytes)[:4] big-endian % 100`.
Companies with `bucket < percentage` are enabled.

```bash
# Quick Python helper to preview which companies are in the 10% bucket:
python3 -c "
import hashlib, uuid
company_ids = ['<UUID1>', '<UUID2>']  # paste UUIDs here
for cid in company_ids:
    u = uuid.UUID(cid)
    bucket = int.from_bytes(hashlib.md5(u.bytes, usedforsecurity=False).digest()[:4], 'big') % 100
    print(f'{cid}: bucket={bucket}, in_10pct={bucket<10}')
"
```

### 2b. Create a percentage flag for all tenants

Instead of per-company rows, create **one flag per percentage tier** using
a special global flag name. Individual tenant rows take priority via the
existing schema (the gateway checks by `company_id + flag_name`).

For a true platform-wide percentage rollout, you can bulk-insert flag rows
for all active tenants:

```sql
-- Insert a 10% rollout flag for all active companies (run via migration_role):
INSERT INTO client.feature_flags (id, company_id, flag_name, is_active, rollout_percentage)
SELECT gen_random_uuid(), c.id, 'n8n_workflow_crawl_aop_v2', TRUE, 10
FROM client.companies c
ON CONFLICT (company_id, flag_name) DO NOTHING;
```

### 2c. Increase percentage (10 → 50)

```bash
# List flags for a sample company to find the flag_id:
curl -s "http://admin-api:8002/api/v1/admin/feature-flags\
?flag_name=n8n_workflow_crawl_aop_v2" \
  -H "Authorization: Bearer <ADMIN_JWT>"

# Update a specific flag:
curl -s -X PUT \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"rollout_percentage": 50}'
```

---

## Step 3 — Full rollout (100%)

```bash
curl -s -X PUT \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"rollout_percentage": 100}'
```

After full rollout is stable, you may clean up old flag rows:

```bash
curl -s -X DELETE \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>"
```

---

## Rollback / Disabling a Flag

To immediately stop all tenants from receiving the new workflow version:

```bash
curl -s -X PUT \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"is_active": false}'
```

All subsequent N8N workflow executions will see `{"enabled": false}` and take
the safe (previous-stable) path.

---

## Admin API Reference

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/admin/feature-flags` | List all flags (optional `?company_id=&flag_name=` filters) |
| `GET` | `/api/v1/admin/feature-flags/{flag_id}` | Get a single flag |
| `POST` | `/api/v1/admin/feature-flags` | Create a flag (body: `company_id`, `flag_name`, `is_active`, `rollout_percentage`) |
| `PUT` | `/api/v1/admin/feature-flags/{flag_id}` | Update (partial — only provided fields changed) |
| `DELETE` | `/api/v1/admin/feature-flags/{flag_id}` | Delete a flag (204 No Content) |

---

## Troubleshooting

### Flag returns `{"enabled": false}` unexpectedly

1. Verify the flag row exists: `GET /api/v1/admin/feature-flags?company_id=<UUID>&flag_name=<name>`
2. Check `is_active = true` in the response.
3. If `rollout_percentage` is set, compute the bucket:
   ```
   bucket = md5(company_id.bytes)[:4] big-endian % 100
   enabled = bucket < rollout_percentage
   ```
4. Check sirmaai-gateway logs for `feature_flag.evaluated` or `feature_flag.not_found`.

### Gateway returns 401 on the internal endpoint

The `N8N_INTERNAL_SECRET` env var is not set or doesn't match the value
configured in the N8N workflow.  Update the workflow's HTTP Request node headers
and/or set the env var on sirmaai-gateway and restart.

### Gateway returns 503

The sirmaai-gateway database session factory is unavailable (DB down or not
initialised).  Check `make infra` is running and the gateway can reach PostgreSQL.
N8N should route to the safe path automatically in this case.
