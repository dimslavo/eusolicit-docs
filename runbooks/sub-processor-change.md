# Runbook: Sub-Processor Change DPA Notification Flow

**Story**: 18-2 — Sub-Processor Change DPA Notification Flow (Notification Service Extension)  
**Regulation**: GDPR Article 28 — Sub-processor advance notice obligation  
**Last updated**: 2026-05-04

---

## Overview

When `infra/sub-processors.yaml` changes on `main`, the CI pipeline automatically:
1. Publishes a `SubprocessorChanged` event to Redis Stream `eu-solicit:notifications`
2. The notification service (`SubprocessorConsumer`) fans out DPA notification emails to all
   active admins of every paid-tier (`starter`, `professional`, `pro_plus`, `enterprise`)
   active/trialing company.

This runbook covers: adding/removing sub-processors, investigating notification failures,
re-triggering events, and verifying delivery.

---

## Sub-Processor Lifecycle

### Adding a Sub-Processor

**GDPR Art. 28 requires ≥ 30 days advance notice before the new sub-processor starts processing.**

1. Edit `infra/sub-processors.yaml`:
   ```yaml
   - name: "NewVendor"
     purpose: "Analytics processing"
     region: "EU"
     effective_date: "YYYY-MM-DD"   # must be ≥ today + 30 days
     dpa_url: "https://newvendor.com/dpa"   # optional
   ```

2. Validate locally:
   ```bash
   python scripts/validate_subprocessors.py --enforce-advance-notice \
     --diff-base origin/main infra/sub-processors.yaml
   ```

3. Run changelog generator:
   ```bash
   python scripts/generate_subprocessor_changelog.py
   ```

4. Commit both `infra/sub-processors.yaml` and `infra/sub-processors-changelog.md`.

5. Open a PR. CI enforces:
   - Schema validation (`validate-sub-processors-yaml`)
   - 30-day advance-notice (`enforce-subprocessor-advance-notice`)
   - Changelog freshness (`check-subprocessor-changelog`)

6. On merge to main, CI publishes `SubprocessorChanged` event; DPA notification emails go out.

### Removing a Sub-Processor

No advance notice required (Art. 28 applies only to additions).

1. Delete the entry from `infra/sub-processors.yaml`.
2. Run changelog generator, commit both files, open PR.
3. On merge, notification emails are sent (same fan-out as additions).

### Modifying a Sub-Processor

Modified rows are **not** subject to the 30-day rule.

1. Update the entry, run changelog generator, commit, PR.
2. On merge, a notification email is sent with the change.

---

## Notification Email Recipients

**Who receives the email**: All users where:
- `company.subscription.tier ∈ {starter, professional, pro_plus, enterprise}` AND
- `company.subscription.status ∈ {active, trialing}` AND
- `membership.role == admin` AND
- `membership.accepted_at IS NOT NULL` AND
- `user.is_active == True`

**Locale routing**: email locale = `admin.locale_preference` (`bg`/`en`). Unset → `bg` (default).

**DPO contact**: email body contains `NOTIFICATION_DPO_CONTACT_EMAIL` value.

---

## Idempotency

The consumer uses SETNX (per-recipient, per-content-hash) to prevent duplicate sends:

- Key: `notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}`
- TTL: 86400s (24 hours)
- `changelog_sha` = first 8 hex chars of SHA-256 of YAML content (stable across re-runs with same content)

**Effect**: if the same YAML change is re-pushed (e.g., CI re-run, pod restart), each admin receives
at most ONE email per 24-hour window per content-hash.

---

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `NOTIFICATION_SENDGRID_TEMPLATE_SUBPROCESSOR_CHANGE_BG` | SendGrid template ID (Bulgarian) | `d-subprocessor-change-bg` |
| `NOTIFICATION_SENDGRID_TEMPLATE_SUBPROCESSOR_CHANGE_EN` | SendGrid template ID (English) | `d-subprocessor-change-en` |
| `NOTIFICATION_DPO_CONTACT_EMAIL` | DPO email in notification body | `dpo@eusolicit.com` |
| `REDIS_URL_NOTIFICATIONS` | CI secret: Redis URL for publisher script | (required in CI) |

---

## Investigating a Failed/Missing Notification

### Step 1: Check the CI job

1. Go to GitHub Actions → most recent push-to-main run.
2. Find `publish-subprocessor-changed-event` job.
3. Check exit code and logs. Expected success log:
   ```
   subprocessor_event_published event_id=... changelog_sha=... added_count=1 removed_count=0 modified_count=0
   ```

If the job **skipped** (`subprocessor_event_skipped_empty_diff`): the YAML diff was empty — no notification is expected.

### Step 2: Check the notification service consumer logs

```bash
# In production (Kubernetes)
kubectl logs -l app=notification --since=1h | grep "subprocessor_consumer"

# In local dev
docker compose logs notification --tail=200 | grep -i subprocessor
```

Look for:
- `subprocessor_consumer.subprocessor_changed_dispatched admin_count=N` — N emails dispatched
- `subprocessor_consumer.no_paid_admins` — no qualifying recipients found
- `subprocessor_consumer.unknown_event_type` — event was routed to subprocessor consumer but has wrong type

### Step 3: Check Redis idempotency keys

If you suspect duplicate-suppression is blocking legitimate re-sends:

```bash
# Connect to Redis
redis-cli -u "$REDIS_URL"

# Check if admin already received this changelog_sha
GET "notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}"
# → "1" if already dispatched, (nil) if not yet

# Force re-send by deleting the key
DEL "notification:dispatched:subprocessor:{changelog_sha}:{admin_user_id}"
```

---

## Re-triggering a SubprocessorChanged Event

If the notification emails were not sent (e.g., Redis was down when `main` was pushed),
re-trigger by running the publisher script manually:

```bash
cd eusolicit-app

# Get the current YAML (new version)
HEAD_YAML=$(cat infra/sub-processors.yaml)

# Get the previous version (from git)
PREV_YAML=$(git show HEAD~1:infra/sub-processors.yaml 2>/dev/null || echo "schema_version: 1
sub_processors: []")

# Publish the event
python scripts/publish_subprocessor_event.py \
  --head-yaml infra/sub-processors.yaml \
  --prev-yaml /tmp/prev-sub-processors.yaml \
  --redis-url "$REDIS_URL"
```

**Note**: The SETNX idempotency guard (24h TTL) will prevent duplicate emails if the event
was already delivered within the last 24 hours. Delete the per-admin Redis keys first if you
need to force re-send.

---

## Verifying Delivery

### Check SendGrid email logs

In the SendGrid Dashboard:
1. Go to Activity → Email Activity.
2. Filter by template ID (`NOTIFICATION_SENDGRID_TEMPLATE_SUBPROCESSOR_CHANGE_BG` or `_EN`).
3. Verify all expected admin recipients appear with status `Delivered`.

### Check notification service email_log table

```sql
-- Connect with migration_role or notification_role (SELECT-only)
SELECT recipient_email, template_type, status, sendgrid_message_id, sent_at
FROM notification.email_log
WHERE template_type = 'subprocessor_change'
ORDER BY sent_at DESC
LIMIT 50;
```

---

## FAQ

**Q: Why did only some admins receive the email?**  
A: Check if those admins are in paid-tier active/trialing companies with accepted memberships.
The resolver query filters: `tier ∈ PAID_TIERS`, `status ∈ {active, trialing}`, `role == admin`,
`accepted_at IS NOT NULL`, `is_active == True`.

**Q: An admin has the wrong locale in their email. How do I fix it?**  
A: Update `locale_preference` in the `client.users` table for that user (`bg` or `en`). The next
SubprocessorChanged event will use the updated preference.

**Q: The 30-day rule CI check is failing on a modification (not addition).**  
A: Modified and removed rows are NOT subject to the 30-day rule. If CI is failing, check whether
the YAML change is being classified as an addition (e.g., a name change creates a remove + add).
To modify without triggering the rule, keep the same `name` field and only change other fields.

**Q: How do I test the notification locally without sending real emails?**  
A: Set `NOTIFICATION_SENDGRID_API_KEY=SG.test-key` (default) — the SendGrid client will fail
gracefully. Or mock the task in a test environment. For integration testing, use
`make infra` + `pytest services/notification/tests -m integration -v`.

---

## Related Documents

- Story: `eusolicit-docs/implementation-artifacts/18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md`
- ATDD checklist: `test_artifacts/atdd-checklist-18-2-sub-processor-change-dpa-notification-flow-notification-service-extension.md`
- infra README: `eusolicit-app/infra/README.md#sub-processor-maintenance`
- GDPR Art. 28 reference: https://gdpr-info.eu/art-28-gdpr/
