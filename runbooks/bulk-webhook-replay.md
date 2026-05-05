# Runbook: Bulk Webhook Replay

**Severity**: SEV-2 (post-incident recovery action)
**SLA-Scope**: in-scope (platform owns the recovery flow)
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6)

---

> **Context**: This runbook covers the procedure for replaying missed webhooks after a
> platform downtime or a provider-side outage (Stripe, Slack, Teams). The platform owns
> the recovery flow (replay procedure, idempotency verification) even when the upstream
> event source (Stripe) was the cause of the missed events.

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| Invoice-state divergence after Stripe outage recovery (invoices stuck `open`) | Admin API → billing panel |
| Slack `#platform-incidents` alert notifications missing for an outage window | Slack channel history |
| Teams notification delivery gaps | Microsoft Teams channel history |
| Stripe webhook DLQ depth > 0 after Stripe recovery | Redis DLQ key |
| billing-service logs showing `event_id already processed` (dedup working) | client-api logs |
| `stripe events list` returning events with `pending` delivery status | Stripe CLI |

---

## Triage

1. **Identify the outage window** (start + end time):
   ```bash
   # Example: Stripe outage from 14:00 to 15:30 UTC on 2026-MM-DD
   OUTAGE_START="2026-MM-DDTHH:MM:SSZ"
   OUTAGE_END="2026-MM-DDTHH:MM:SSZ"
   ```

2. **Identify which webhook types need replay**:
   - **Stripe**: invoice events (`invoice.paid`, `invoice.payment_failed`, `customer.subscription.updated`)
   - **Slack**: platform alert notifications (best-effort; Slack does not support webhook replay)
   - **Teams**: platform alert notifications (best-effort; Teams does not support webhook replay)

3. **Check Stripe's automatic retry** — Stripe retries failed webhook deliveries for up to 72 hours:
   ```bash
   stripe events list \
     --created.gte=<outage_start_unix> \
     --created.lte=<outage_end_unix> \
     --type=invoice.paid \
     --limit 100
   ```
   If events show `delivery_success: false` and `pending_webhooks > 0` → Stripe will retry automatically.
   If delivery has already been abandoned → manual replay needed.

4. **Check billing DLQ for unprocessed events**:
   ```bash
   redis-cli -h <endpoint> -p 6379 --tls LLEN billing:webhook:dlq
   redis-cli -h <endpoint> -p 6379 --tls LRANGE billing:webhook:dlq 0 -1
   ```

5. **Verify idempotency is active** — `handle_stripe_webhook` uses Stripe `event_id` dedup (Story 8):
   - Safe to replay events that were already processed → they will be silently skipped.
   - NEVER skip idempotency verification before replay.

---

## Resolution

### Stripe Webhook Replay

1. **Retrieve missed event IDs** from Stripe:
   ```bash
   stripe events list \
     --created.gte=<outage_start_unix> \
     --created.lte=<outage_end_unix> \
     --limit 100 \
     --output json > missed_events.json
   ```

2. **Replay each event** via Stripe CLI:
   ```bash
   cat missed_events.json | jq -r '.[].id' | while read event_id; do
     echo "Replaying: $event_id"
     stripe events resend $event_id \
       --account <stripe_account_id> \
       --webhook-endpoint <platform_webhook_url>
   done
   ```
   ⚠️ Test with 1 event first. Verify the webhook endpoint is reachable and idempotency is working.

3. **Alternative: Stripe Dashboard replay** — Stripe Dashboard → Developers → Webhooks → select the endpoint → resend individual events.

4. **Verify billing-service processed the replayed events**:
   ```bash
   kubectl logs -n eusolicit deploy/client-api --since=10m | grep "stripe_webhook\|event_id"
   ```
   Expected: `"event_id": "<evt_xxx>", "status": "processed"` (or `"duplicate_skipped"` for already-processed events).

5. **Reconcile invoice state**:
   ```bash
   # Compare Stripe invoices vs local state
   stripe invoices list --status=open --created.gte=<outage_start_unix>
   ```
   Any invoice that Stripe shows as `paid` but local state shows as `open` → manually update via admin API.

### Slack Webhook Replay

Slack does not support webhook replay — the events are best-effort delivery.

1. **Identify missed alerts** from Prometheus / PagerDuty history during the outage window.
2. **Manually post a summary** to `#platform-incidents`:
   > "Alert notification gap during <start>–<end> UTC. The following alerts fired during this window and were not delivered to Slack: <list alerts>. Incident <ID> was handled by on-call. No action required."

### Teams Webhook Replay (integrations-api)

Teams webhooks are signed and delivered via `integrations-api` (Story 16.0).

1. **Check integrations-api logs for failed deliveries**:
   ```bash
   kubectl logs -n eusolicit deploy/integrations-api --since=<outage_duration>m | grep -i "teams\|webhook\|retry"
   ```

2. **Resign and re-deliver** failed messages using the `integrations-api` retry endpoint:
   ```bash
   curl -X POST https://integrations-api.eusolicit.eu/internal/webhook/retry \
     -H "Authorization: Bearer <internal-token>" \
     -d '{"provider": "teams", "window_start": "<timestamp>", "window_end": "<timestamp>"}'
   ```
   (Endpoint is internal; requires internal service token from Secrets Manager.)

---

## Verification

1. **Stripe invoice state consistent** — spot-check 10 invoices from the outage window:
   - Stripe state = `paid` → local state = `paid` ✓
   - Stripe state = `open` → local state = `open` (awaiting payment; correct)

2. **Billing DLQ empty**:
   ```bash
   redis-cli -h <endpoint> -p 6379 --tls LLEN billing:webhook:dlq
   ```
   Expected: 0.

3. **No duplicate billing events** — confirm no double-charges or double-credits via Stripe Dashboard.

4. **Idempotency verified** — replayed events show `"duplicate_skipped"` for previously-processed events.

5. **Slack/Teams alert gaps acknowledged** in the relevant channels.

---

## Rollback

If webhook replay causes duplicate billing charges:

1. **Immediately stop the replay**:
   - Cancel the CLI replay loop if still running (`Ctrl+C`).

2. **Identify affected invoices** in Stripe Dashboard → check for double-charges.

3. **Issue refunds via Stripe Dashboard** for confirmed duplicate charges.

4. **Audit the idempotency logic**: if `event_id` dedup failed, escalate to a code fix in `billing_service.handle_stripe_webhook` as a P0 bug.

---

## Related

- `stripe-outage.md` — upstream Stripe outage (context for why replay is needed)
- `severity-definitions.md` — SEV-2 response SLA (bulk replay is a post-incident recovery action)
- `deploy-rollback.md` — if a deploy caused webhook-processing failures
- Story 8 — `client_api.services.billing_service.handle_stripe_webhook` idempotency pattern
- Story 16.0 — integrations-api Slack/Teams webhook signing logic
