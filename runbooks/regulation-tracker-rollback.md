# Rollback Runbook: Regulation Tracker N8N Workflow

**Story:** 11.21 — Regulation Tracker N8N Workflow + Celery Retire
**Feature Flag:** `regulation-tracker-n8n`
**Last updated:** 2026-05-16

---

## Overview

This runbook describes how to roll back from the N8N-based regulation tracker workflow to the legacy Celery Beat task.

The rollout is controlled by the `regulation-tracker-n8n` feature flag (managed via the agenticsai-gateway internal feature-flag service). When this flag is active, the N8N workflow runs. When it is inactive, the N8N workflow exits via the graceful "Skipped (Flag Off)" branch, and the legacy Celery Beat task at `services/admin-api/src/admin_api/tasks/regulation_tracker.py` continues as the system of record.

> **Rollback window:** The legacy Celery task (`run_regulation_tracker_task`) is marked
> **DEPRECATED** in Story 11.21 and is scheduled for removal in the S04.30 cleanup pass.
> Rollback is only viable until that cleanup pass ships. After S04.30, only the N8N path
> remains and this runbook is no longer applicable.

---

## Step 1: Disable the N8N Workflow Feature Flag

To immediately disable the new N8N workflow, set the `regulation-tracker-n8n` feature flag
to inactive via admin-api.

**1a. Find the flag ID:**

```bash
curl -s "http://admin-api:8002/api/v1/admin/feature-flags?flag_name=regulation-tracker-n8n" \
  -H "Authorization: Bearer <ADMIN_JWT>"
```

This returns a JSON payload containing the flag's details, including its `id`.

**1b. Disable the flag:**

```bash
curl -s -X PUT \
  "http://admin-api:8002/api/v1/admin/feature-flags/<FLAG_ID>" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  -H "Content-Type: application/json" \
  -d '{"is_active": false}'
```

All subsequent N8N workflow executions will see `{"enabled": false}` from the
agenticsai-gateway feature-flag check and exit via the graceful skip path.

---

## Step 2: Verify Rollback

**2a. Confirm the flag is inactive via agenticsai-gateway:**

```bash
curl -s \
  "http://agenticsai-gateway:8004/api/internal/agenticsai/feature-flags/check?flag_name=regulation-tracker-n8n&company_id=<PLATFORM_ORG_COMPANY_ID>" \
  -H "X-Internal-Secret: <N8N_INTERNAL_SECRET>"
# Expected: {"enabled": false}
```

**2b. Verify the Celery Beat task is scheduled:**

The legacy Celery task is unconditionally scheduled in `admin_api/worker.py`
(`beat_schedule` key `run-regulation-tracker`, Monday 08:00 UTC). After disabling
the N8N flag, the Celery task resumes as the sole regulation-tracker execution path.

```bash
# Check the Celery Beat schedule is active
docker compose exec admin-api celery -A admin_api.worker inspect scheduled
```

**2c. Monitor Celery logs** to confirm the task runs on its next scheduled window:

```bash
docker compose logs -f admin-api | grep regulation_tracker
```

---

## Step 3: Verify Regulation Changes Still Flow

After rollback, confirm that `admin.regulation_changes` continues to receive rows:

```bash
curl -s "http://admin-api:8002/api/v1/admin/regulatory-changes" \
  -H "Authorization: Bearer <ADMIN_JWT>" \
  | jq '. | length'
```

---

## See Also

- `eusolicit-docs/runbooks/managing-feature-flags.md` — general feature flag management
- Story 11.21 implementation artifact for N8N template and webhook consumer details
- S04.30 cleanup story for final Celery task retirement
