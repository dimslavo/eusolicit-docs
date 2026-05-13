---
storyId: drift-recovery-story
storyKey: drift-recovery-story
storyFile: eusolicit-docs/implementation-artifacts/drift-recovery-story.md
generationMode: bmad-dev-story pass (AC4 + AC5)
tddPhase: GREEN (red phase skipped — coordinator story; tests written after wraps land)
generatedTestFiles:
  - eusolicit-app/services/client-api/tests/unit/test_stripe_resilience.py
  - eusolicit-app/services/client-api/tests/unit/test_billing_metrics_emissions.py
lastSaved: '2026-05-13'
---

# ATDD Checklist — drift-recovery-story (AC4 + AC5)

## AC4 — Stripe Outbound Circuit-Breaker (Owned by this story)

### Unit (`test_stripe_resilience.py`)

- [x] **Scenario 1 — 5 consecutive 5xx transient errors open the circuit.**
  `test_five_consecutive_5xx_opens_circuit` — five `APIConnectionError` raises increment failures to threshold (5); 6th call rejects without invoking `fn` and raises `StripeCircuitOpenError`.

- [x] **Scenario 2 — 4 transient failures keep the circuit closed.**
  `test_four_transient_failures_keep_circuit_closed` — sub-threshold failures do NOT open. `failures == 4`, `state == CLOSED`.

- [x] **Scenario 3 — Half-open probe after cooldown.**
  `test_open_circuit_transitions_to_half_open_after_cooldown` — force open with 50 ms cooldown, sleep 60 ms, assert state transitions to `HALF_OPEN`, then a successful probe closes the circuit (`failures == 0`).

- [x] **Scenario 4 — 4xx CardError does NOT increment the failure counter (OBS-001 fix).**
  `test_4xx_card_error_does_not_increment_failure_counter` — 10 consecutive `CardError` raises propagate but `failures == 0` and `state == CLOSED`. Closes Epic 5 OBS-001 anti-pattern for the billing path.

- [x] **Scenario 5 — Successful call resets failure counter.**
  `test_successful_call_records_success_and_resets_failures` — 3 transient failures (under threshold) followed by a successful call: `failures == 0`, `state == CLOSED`.

- [x] **Operational snapshot helper.**
  `test_circuit_snapshot_returns_per_op_state` — `get_circuit_snapshot()` returns per-op state for both failed and succeeded ops.

### Integration (deferred to follow-up)

- [ ] **`testcontainers` + `respx` — end-to-end through `billing_service.create_checkout_session()`** with simulated 5xx storm. Deferred per story-execution scope cap; tracked as follow-up. Reference pattern: `test_stripe_webhook_flow.py` (8.4-API-004).

- [ ] **Endpoint behavior — 503 + `Retry-After` when circuit OPEN.** Currently callers catch `StripeCircuitOpenError` and translate to `{"error": "stripe_error", ...}` HTTP-500. A follow-up should add an explicit 503 response with a `Retry-After` header derived from `cooldown_seconds`. Out of scope here; flagged in the dev pass completion notes.

### Production wrap evidence (9/9 sites)

- [x] `billing_service.provision_stripe_customer` — `op="customer.create"` (line 151)
- [x] `billing_service.provision_professional_trial` — `op="subscription.create.trial"` (line 330)
- [x] `billing_service.create_addon_checkout_session` — `op="checkout.session.create.addon"` (line 498)
- [x] `billing_service.create_checkout_session` — `op="checkout.session.create.subscription"` (line 621)
- [x] `billing_service.create_portal_session` — `op="billing_portal.session.create"` (line 734)
- [x] `billing_service.report_seat_count_to_stripe` — `op="subscription_item.modify.seats"` (line 883)
- [x] `billing_service.create_per_bid_checkout_session` — `op="checkout.session.create.per_bid"` (line 1006)
- [x] `vies_service.validate_and_sync_company_vat` — `op="customer.modify.vat"` (line 243)
- [x] `api/v1/billing.validate_vat_endpoint` — `op="customer.modify.vat"` (line 506)

## AC5 — Billing Prometheus Metrics (Owned by this story)

### Unit (`test_billing_metrics_emissions.py`)

- [x] **register_billing_metrics is idempotent.** `test_register_is_idempotent` — calling register twice with the same registry returns the same `_BillingMetrics` instance (avoids `Duplicated timeseries` error in tests rebuilding the app).

- [x] **`billing_webhook_processing_duration_seconds` Histogram emission.** `test_webhook_processing_duration_emits` — `.observe(0.123)` on labels `{event_type=customer.subscription.updated}` produces a `_count == 1` sample.

- [x] **`billing_usage_sync_drift_total` Counter increment.** `test_usage_sync_drift_increments` — `.inc()` on labels `{feature=proposal_generation, drift_direction=permanent_failure}` produces a `_total == 1` sample.

- [x] **`billing_stripe_api_errors_total` Counter increment (cumulative).** `test_stripe_api_errors_increments` — two `.inc()` calls produce a `_total == 2` sample.

- [x] **`billing_active_subscriptions_total` Gauge `.set` (replaces, not adds).** `test_active_subscriptions_gauge_set` — `.set(42)` then `.set(17)` produces value 17 (not 59).

- [x] **`billing_trial_to_paid_conversions_total` Counter increment.** `test_trial_to_paid_conversions_increments` — `.inc()` on `{tier=professional}` produces a `_total == 1` sample.

### Integration (deferred — fold into webhook integration suite)

- [ ] **Fire `customer.subscription.updated` (trialing→active) → scrape `/metrics` → assert `trial_to_paid_conversions{tier=professional} == 1` AND `webhook_processing_duration_count{event_type=customer.subscription.updated} == 1`.** Pattern: `test_stripe_webhook_flow.py`. Deferred — integration suite expansion tracked as follow-up.

- [ ] **Curl smoke (manual)**: `curl http://localhost:8001/metrics | grep ^billing_` lists 5 metric families. To be verified post-deploy on www1 once monitoring stack is live (onprem-03 dependency).

### Production emission evidence (5/5 metrics)

- [x] `webhook_processing_duration` — `webhook_service.process_stripe_webhook` (timing wrapper around `_process_stripe_webhook_impl`)
- [x] `usage_sync_drift` — `billing_usage_sync.sync_usage_to_stripe` via `_emit_drift` helper (`feature=metric_name`, `drift_direction ∈ {transient_failure, permanent_failure}`)
- [x] `stripe_api_errors` — `stripe_resilience.resilient_stripe_call` on every transient/non-retryable failure (`circuit_state`, `error_type` labels)
- [x] `active_subscriptions` — `webhook_service._refresh_active_subscriptions_gauge` after each subscription event commit (GROUP BY tier; zeroes unseen tiers)
- [x] `trial_to_paid_conversions` — `webhook_service.process_stripe_webhook` for `customer.subscription.updated` where `previous_attributes.status == "trialing"` and `current.status == "active"` and tier is paid

## AC4/AC5 Grafana Dashboard

- [x] `eusolicit-app/infra/observability/grafana/dashboards/billing-overview.json` — 5 panels (webhook p95, usage-sync drift rate, active subscriptions by tier, trial→paid conversion rate, Stripe API errors by op + state). Follows the PE.05 / Story 21-5 provisioning pattern (file-on-disk, auto-loaded by `infra/observability/grafana/provisioning/dashboards/`).

## Out-of-Scope (per Story §"Out-of-Scope" — explicitly fenced off)

- AC1 (Dependabot, inj-01) — DONE per sprint-status.
- AC2 (k6 baseline, inj-02) — DONE via Story 21-1 (k6-baseline-closure).
- AC3 (TEA review backlog, inj-03) — DEFERRED (requires interactive bmad-tea passes, not bmad-dev-story).
- AC6 (Coordinator gate close → epic-13 → done) — partial: AC4 + AC5 close-out evidence above; FINAL gate flip blocked on AC3 closure.

## Latent bug fixed in this pass

- **`from billing_metrics import BILLING_METRICS` captured `None` at import time** in `stripe_resilience.py` (the seed-pass `stripe_api_errors` emission was silently dead code because `register_billing_metrics` runs at `main.py` line 112, AFTER service modules import on lines 30+). Fixed by switching to `from client_api.services import billing_metrics as _billing_metrics_module` and referencing `_billing_metrics_module.BILLING_METRICS` at call time so the runtime lookup sees the registered singleton. Applied to `stripe_resilience.py` and the new emission sites in `webhook_service.py`.
