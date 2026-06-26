# Runbook: Stripe API Outage

**Severity**: SEV-2 (platform-visible) / SEV-3 (if brief)
**SLA-Scope**: EXEMPT (vendor outage)
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6)

---

> **SLA-EXEMPT notice**: Stripe is an upstream payment vendor. Stripe API outages are
> **explicitly excluded from the platform 99.9% SLA scope** per the Epic 4 AgenticSAI
> isolation precedent applied to Stripe (architecture.md line 762 pattern).
> The platform owns the recovery flow (webhook replay, reconciliation) but the
> Stripe API availability itself is not in the platform SLA.
> See `severity-definitions.md` SLA-scope table.

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| Stripe API 5xx errors elevated in client-api logs | `kubectl logs -n eusolicit deploy/client-api` |
| Billing webhook DLQ depth growing | CloudWatch / Redis DLQ queue depth |
| Customer reports of failed subscription checkouts or payment failures | Support channel |
| `client_api.services.billing_service` retry/circuit-breaker logging 429 or 5xx | client-api logs |
| Stripe status page reporting an incident | https://www.stripe.com/status |
| Invoice-state divergence (invoices stuck in `open` after expected payment) | Admin API → billing panel |

---

## Triage

1. **Check Stripe status page first**: https://www.stripe.com/status
   - Confirmed Stripe incident → EXEMPT status confirmed. Log it and monitor.
   - No Stripe incident → may be a platform-side misconfiguration or rate-limit. Investigate further.

2. **Check client-api billing-service logs**:
   ```bash
   kubectl logs -n eusolicit deploy/client-api --since=10m | grep -i "stripe\|billing"
   ```
   - `stripe.error.APIConnectionError` → network/DNS issue → check client-api → Stripe connectivity.
   - `stripe.error.RateLimitError` → platform is sending too many requests → billing service throttle bug.
   - `stripe.error.APIError (5xx)` → confirmed Stripe-side issue.

3. **Check billing webhook DLQ**:
   ```bash
   # DLQ depth (key name may vary by implementation)
   redis-cli -h <endpoint> -p 6379 --tls LLEN billing:webhook:dlq
   ```
   Growing DLQ → webhooks are arriving but cannot be processed (possibly Stripe retry storm).

4. **Confirm the existing retry/circuit-breaker is active**:
   - `client_api.services.billing_service` implements retry + idempotent webhook handling.
   - Check that retries are happening (logs show backoff with increasing delay).
   - If retries stopped prematurely → investigate circuit breaker cooldown.

---

## Resolution

**Primary action: WAIT.** Stripe is upstream; the platform cannot resolve their outage.

1. **Acknowledge the alert** in PagerDuty / Slack `#platform-incidents` with status:
   > "Stripe API outage confirmed at <time> per https://www.stripe.com/status. Billing is degraded (SLA-EXEMPT per architecture.md line 762). Monitoring for Stripe recovery."

2. **Inform affected customers** (if checkout failures are visible to users):
   - Use `status-page-comms-templates.md` SEV-1 Initial template (scope: "payment processing temporarily unavailable due to Stripe upstream outage").
   - Recommend customers retry after Stripe reports recovery.

3. **Let the billing service retry automatically** — `handle_stripe_webhook` is idempotent (Stripe `event_id` dedup in Story 8). Do NOT manually replay webhooks while Stripe is still degraded.

4. **Once Stripe recovers**: replay any missed webhooks per `bulk-webhook-replay.md`:
   - Stripe retries missed webhook deliveries automatically for up to 72 hours.
   - For invoice-state divergence, use `stripe events list --created.gte=<outage_start>` to identify missed events.

5. **Reconcile invoice state** if Stripe recovery doesn't backfill all missed events:
   - Compare local invoice state with Stripe `stripe invoices list --status=open`.
   - Update local invoice state via admin API if diverged.

---

## Verification

1. **Stripe status page shows "All Systems Operational"**: https://www.stripe.com/status

2. **client-api billing-service logs show successful Stripe API calls**:
   ```bash
   kubectl logs -n eusolicit deploy/client-api --since=5m | grep "stripe" | grep -v "error"
   ```

3. **Webhook DLQ depth returns to 0**:
   ```bash
   redis-cli -h <endpoint> -p 6379 --tls LLEN billing:webhook:dlq
   ```

4. **Invoice state consistent** — spot-check 5 recent invoices against Stripe dashboard.

5. **Platform SLO unaffected** — confirm `slo_target="platform"` burn rate < 1×.

---

## Rollback

Not applicable — Stripe outage recovery is handled by:
1. Stripe's automatic webhook retry (72-hour window).
2. Manual reconciliation via `bulk-webhook-replay.md`.

No platform-side rollback action is needed.

---

## Related

- `bulk-webhook-replay.md` — webhook replay procedure post-Stripe recovery
- `severity-definitions.md` — SLA-scope table (Stripe listed as EXEMPT)
- `error-budget-burn.md` — if Stripe outage somehow impacts platform SLO (should not per carve-out)
- architecture.md line 762 — vendor isolation precedent
- Stripe status page: https://www.stripe.com/status
- Story 8 `billing_service.handle_stripe_webhook` — idempotent webhook handler
