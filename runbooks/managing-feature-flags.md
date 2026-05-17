# Managing Feature Flags — Operator Runbook

**Story:** 5.24 — N8N Staged Rollout Enforcement  
**Service:** admin-api (management), sirmaai-gateway (evaluation)  
**Table:** `client.feature_flags`  
**Last updated:** 2026-05-15

---

## Overview

EU Solicit uses a **feature flag gate** in every N8N workflow to control which
tenants receive a new workflow version (e.g. `crawl-aop-v2`).  When an N8N
workflow starts, its first HTTP node calls the sirmaai-gateway to evaluate
whether the new logic should run for the current tenant:

```
GET http://sirmaai-gateway:8004/api/internal/sirmaai/feature-flags/check
    ?company_id=<UUID>&flag_name=<flag>
    X-Internal-Secret: <N8N_INTERNAL_SECRET>
```

If the response is `{"enabled": true}` the workflow proceeds with the new logic.
If `{"enabled": false}` the workflow routes to a graceful skip path that records
`skipped_by_feature_flag` and exits cleanly.

Three rollout tiers are supported:

| Tier | Configuration |
|------|--------------|
| **Canary** | One flag row per canary `company_id` with `is_active=true`, `rollout_percentage=null`. Only those exact tenants are enabled. |
| **Percentage** | `rollout_percentage=10` (or 1–99). Bucket: `md5(company_id.bytes)[:4] % 100 < pct`. Same company always in or out. |
| **Full rollout** | `rollout_percentage=100`. All tenants enabled. |

---

## Prerequisites

- Admin API JWT for the `platform_admin` role (`ADMIN_API_JWT_SECRET`).
- `company_id` UUIDs of target tenants (from admin UI or DB query).
- N8N internal secret configured: `N8N_INTERNAL_SECRET` env var on sirmaai-gateway.

---

## Step 1 — Create a feature flag (canary rollout)

Create one flag row per canary tenant:

```bash
curl -s -X POST http://admin-api:8002/api/v1/admin/feature-flags \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{
    "company_id": "<COMPANY_UUID>",
    "flag_name": "crawl_aop_v2",
    "is_active": true,
    "rollout_percentage": null
  }'
```

Verify:

```bash
curl -s "http://sirmaai-gateway:8004/api/internal/sirmaai/feature-flags/check\
?company_id=<COMPANY_UUID>&flag_name=crawl_aop_v2" \
  -H "X-Internal-Secret: <N8N_INTERNAL_SECRET>"
# Expected: {"enabled": true}
```

Non-canary companies return `{"enabled": false}` because no flag row exists for them.

---

## Step 2 — List and inspect flags

```bash
# List all flags
curl -s "http://admin-api:8002/api/v1/admin/feature-flags" \
  -H "Authorization: Bearer <ADMIN_JWT>"

# Filter by company
curl -s "http://admin-api:8002/api/v1/admin/feature-flags?company_id=<UUID>" \
  -H "Authorization: Bearer <ADMIN_JWT>"

# Filter by flag name
curl -s "http://admin-api:8002/api/v1/admin/feature-flags?flag_name=crawl_aop_v2" \
  -H "Authorization: Bearer <ADMIN_JWT>"

# Get a single flag
curl -s "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>"
```

---

## Step 3 — Percentage-based rollout (10% → 50% → 100%)

### 3a. Preview which companies fall in the bucket

```bash
python3 -c "
import hashlib, uuid
company_ids = ['<UUID1>', '<UUID2>']
for cid in company_ids:
    u = uuid.UUID(cid)
    bucket = int.from_bytes(hashlib.md5(u.bytes, usedforsecurity=False).digest()[:4], 'big') % 100
    print(f'{cid}: bucket={bucket}, in_10pct={bucket<10}')
"
```

### 3b. Bulk-insert a percentage rollout for all active tenants

```sql
-- Run via migration_role or admin_api_role:
INSERT INTO client.feature_flags (id, company_id, flag_name, is_active, rollout_percentage)
SELECT gen_random_uuid(), c.id, 'crawl_aop_v2', TRUE, 10
FROM client.companies c
ON CONFLICT (company_id, flag_name) DO NOTHING;
```

### 3c. Increase percentage

```bash
# Find the flag ID:
curl -s "http://admin-api:8002/api/v1/admin/feature-flags?flag_name=crawl_aop_v2" \
  -H "Authorization: Bearer <ADMIN_JWT>"

# Update rollout_percentage to 50:
curl -s -X PUT \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"rollout_percentage": 50}'
```

---

## Step 4 — Full rollout (100%)

```bash
curl -s -X PUT \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"rollout_percentage": 100}'
```

---

## Step 5 — Rollback / disable a flag

To immediately disable the new workflow version for all tenants:

```bash
curl -s -X PUT \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"is_active": false}'
```

All subsequent N8N workflow executions will see `{"enabled": false}` and exit
via the graceful skip path.

---

## Step 6 — Delete a flag (cleanup)

After a successful full rollout, old flag rows can be removed:

```bash
curl -s -X DELETE \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>"
# Returns: HTTP 204 No Content
```

---

## Admin API Reference

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/admin/feature-flags` | List all flags (`?company_id=` and `?flag_name=` filters) |
| `GET` | `/api/v1/admin/feature-flags/{id}` | Get a single flag by ID |
| `POST` | `/api/v1/admin/feature-flags` | Create a flag (body: `company_id`, `flag_name`, `is_active`, `rollout_percentage`) |
| `PUT` | `/api/v1/admin/feature-flags/{id}` | Update (partial — only provided fields changed) |
| `DELETE` | `/api/v1/admin/feature-flags/{id}` | Delete a flag (HTTP 204) |

---

## Evaluation Logic Reference

The gateway endpoint implements the following logic (AC #7):

| Condition | Result |
|-----------|--------|
| Flag row not found | `{"enabled": false}` (fail-safe) |
| `is_active = false` | `{"enabled": false}` |
| `is_active = true`, `rollout_percentage = NULL` | `{"enabled": true}` (pure boolean) |
| `is_active = true`, `rollout_percentage = 100` | `{"enabled": true}` |
| `is_active = true`, `rollout_percentage = 0` | `{"enabled": false}` |
| `is_active = true`, `rollout_percentage = N` (1–99) | `md5(company_id.bytes)[:4] % 100 < N` |

The bucket computation is **deterministic**: the same `company_id` always maps
to the same bucket regardless of call order, time, or infrastructure state.
Increasing `rollout_percentage` never un-enables a previously enabled company
(monotonicity property).

---

## Troubleshooting

### Flag returns `{"enabled": false}` unexpectedly

1. Verify the flag row exists:
   `GET /api/v1/admin/feature-flags?company_id=<UUID>&flag_name=<name>`
2. Check `is_active = true` in the response.
3. If `rollout_percentage` is set, compute the bucket:
   ```
   bucket = md5(company_id.bytes)[:4] big-endian % 100
   enabled = bucket < rollout_percentage
   ```
4. Check sirmaai-gateway logs for `feature_flag.evaluated` or `feature_flag.not_found`.

### Gateway returns 401 on the internal endpoint

The `N8N_INTERNAL_SECRET` env var is not set on sirmaai-gateway, or the value
in the N8N workflow's `X-Internal-Secret` header does not match. Update the
workflow header expression (`{{$env.N8N_INTERNAL_SECRET}}`) and/or set the env
var on sirmaai-gateway and restart.

### Gateway returns 503

The sirmaai-gateway cannot reach PostgreSQL. Check `make infra` is running and
the gateway's `DATABASE_URL` is correct. N8N routes to the safe skip path on 503.

---

## See Also

- **Detailed rollout guide:** `eusolicit-docs/runbooks/n8n-staged-rollout.md`
- **Migration 075:** `eusolicit-app/services/client-api/alembic/versions/075_create_feature_flags.py`
- **Gateway endpoint:** `eusolicit-app/services/sirmaai-gateway/src/sirmaai_gateway/routers/feature_flags.py`
- **Evaluation logic:** `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/feature_flags.py`
