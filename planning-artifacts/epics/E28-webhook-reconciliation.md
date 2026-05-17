# E28: Webhook & Reconciler Hardening

> **Naming note (2026-05-15 IR fix):** Renamed from "Webhook & Run-State Reconciliation" to make explicit that this epic is *operational hardening* of the webhook/reconciler infrastructure already scaffolded in E04 amendment (S04.25 receiver, S04.26 reconciler). The hardening items (subscription bootstrap, HMAC rotation Beat, DLQ admin surface, observability, degraded-mode banner, cross-tenant negative tests) are NOT already-done — only the underlying scaffold is.

**Sprint:** post-pivot S+1..S+2 (parallel-trackable with E04 amendment) | **Points:** 21 | **Dependencies:** E04 amendment (S04.21 schema, S04.25 receiver, S04.26 reconciler scaffold) | **Milestone:** SirmaAI Pivot

> **Source:** `sprint-change-proposal-2026-05-12-sirmaai.md`, `prd-amendment-2026-05-12-sirmaai.md` (FR-54, FR-55, NFR-26), `architecture-amendment-2026-05-12-sirmaai.md` (ADR-018).

## Goal

Make SirmaAI → EU Solicit event delivery reliable, idempotent, and self-healing. Manage Standard Webhooks subscriptions in code (no manual SirmaAI dashboard config); verify HMAC signatures with `hmac.compare_digest()`; deduplicate via a Redis-backed idempotency cache (7-day TTL); route events to internal Redis Streams for downstream consumers (`data-pipeline`, `client-api`, `notification`, `gateway`); converge non-terminal `gateway.workflow_runs` rows via a 5-minute reconciler that polls `GET /jobs/{jobId}/status`; quarantine poison events in a dead-letter queue with admin alerting.

The **reconciler is authoritative truth** for run lifecycle; webhooks are latency optimisation. This invariant matters because SirmaAI is the second critical external dependency (after Stripe) and the platform cannot accept silent run loss.

E28 is the **operational hardening** of the webhook infrastructure scaffolded in E04 amendment (S04.25 receiver, S04.26 reconciler). It owns subscription bootstrap, HMAC secret rotation, DLQ surfacing, observability, and the cross-substrate idempotency discipline that webhook handlers must follow.

## Acceptance Criteria

- [ ] Standard Webhooks subscriptions for the EU Solicit Organisation registered in code at deploy time: `workflow.completed`, `agent.run.completed`, `storage.file.processed`, `policy.violation`
- [ ] HMAC shared secret stored Fernet-encrypted in `gateway.webhook_subscriptions.hmac_secret_encrypted`
- [ ] HMAC secret rotation Celery Beat (90-day cadence): generate new secret → register new subscription → verify reachability → deregister old subscription. Double-validation overlap minimum 24h.
- [ ] Webhook receiver endpoint `POST /webhooks/sirmaai` on `sirmaai-gateway` public ingress (TLS, authenticated only by HMAC signature):
  - [ ] HMAC SHA-256 verified via `hmac.compare_digest()` (Rule 48 — never `==`)
  - [ ] Signature mismatch → 401 + audit-log entry (potential attack signal)
  - [ ] Idempotency cache: Redis SETNX on `webhook_event:{sirmaai_event_id}` with 7-day TTL; duplicate → 200 (already processed)
  - [ ] Event payload validated against expected schema; malformed → DLQ + admin alert
  - [ ] Successful events routed to internal Redis Streams by event-type-to-stream mapping (per amendment §5.3)
- [ ] DLQ table `gateway.webhook_dlq` for events that failed processing after N retries; admin endpoint `GET /api/v1/admin/webhook-dlq` lists; `POST /api/v1/admin/webhook-dlq/{id}/replay` re-enqueues
- [ ] 5-minute run-state reconciler (Celery Beat): scans `gateway.workflow_runs` partial index for non-terminal rows older than 30s, polls SirmaAI `GET /jobs/{jobId}/status`, converges status. Wins race against webhook (idempotent updates via `UPDATE ... WHERE status IN ('pending','running')` guard)
- [ ] Reconciler observability: Prometheus gauges `sirmaai_workflow_runs_nonterminal_total` (by `run_type`), `sirmaai_webhook_dlq_depth_total`, `sirmaai_reconciler_polls_per_minute`, histogram `sirmaai_run_to_terminal_seconds`
- [ ] Tenant-visible degraded-mode banner: on circuit-breaker open >5 minutes against SirmaAI, surface a status banner via existing notification channel — clears on recovery
- [ ] Test scenarios: webhook dropped + reconciler recovers; webhook duplicate (idempotency hits); malformed payload → DLQ; signature mismatch → 401; HMAC secret rotation mid-stream without event loss
- [ ] Cross-tenant negative: SirmaAI event for tenant A's Project routes only to tenant-A-scoped consumers; tenant B can't subscribe to A's events

## New Schema

```sql
-- =====================================================================
-- WEBHOOK DEAD-LETTER QUEUE (E28)
-- =====================================================================

CREATE TABLE gateway.webhook_dlq (
    id                          UUID PRIMARY KEY,
    sirmaai_event_id            TEXT NOT NULL,
    sirmaai_event_type          TEXT NOT NULL,
    payload                     JSONB NOT NULL,
    received_at                 TIMESTAMPTZ NOT NULL DEFAULT now(),
    failed_reason               TEXT NOT NULL,
    retry_count                 INT NOT NULL DEFAULT 0,
    last_retry_at               TIMESTAMPTZ,
    resolved_at                 TIMESTAMPTZ
);
CREATE INDEX ix_webhook_dlq_unresolved
    ON gateway.webhook_dlq(received_at)
    WHERE resolved_at IS NULL;
```

(`gateway.webhook_subscriptions` and `gateway.workflow_runs` already defined in E04 amendment / architecture amendment §4.1.)

## Stories

### S28.01: Subscription bootstrap + lifecycle management
**Points:** 3 | **Type:** backend

At service deploy (or on first start if no subscriptions exist), call `POST /api/webhooks/subscriptions` with the 4 event types. Persist `gateway.webhook_subscriptions` row. Idempotent: re-deploy with existing subscriptions = no-op. Admin endpoint `GET /api/v1/admin/webhook-subscriptions` for ops visibility.

**Acceptance:**
- Cold-start with no subscriptions → 4 subscriptions registered
- Hot-restart with existing subscriptions → no duplicates
- Admin endpoint surfaces subscription status (active, HMAC last rotated, etc.)

---

### S28.02: HMAC secret rotation Celery Beat
**Points:** 3 | **Type:** backend

Beat schedule (90-day cadence): for each `gateway.webhook_subscriptions` row, if `hmac_rotated_at` is >83 days old, initiate rotation. Steps: (1) generate new secret; (2) register new subscription with new secret; (3) verify by sending a test event via `POST /api/webhooks/subscriptions/{id}/test`; (4) on verification success, deregister old subscription; (5) update `hmac_secret_encrypted` + `hmac_rotated_at`. Overlap minimum 24h to absorb in-flight events signed with old secret. Receiver accepts both during overlap.

**Acceptance:**
- Rotation succeeds end-to-end on staging schedule
- Receiver accepts both secrets during overlap window
- Rotation failure (e.g. SirmaAI rejects new subscription) does not deregister old subscription
- Audit log records rotation events

---

### S28.03: Webhook receiver hardening + idempotency cache
**Points:** 5 | **Type:** backend

Public endpoint `POST /webhooks/sirmaai` (TLS, no auth header — HMAC is the auth). Verification flow: parse Standard Webhooks signature header → look up secret(s) per subscription → `hmac.compare_digest()` against expected → on failure 401 + audit. On success: SETNX on `webhook_event:{event_id}` with 7-day TTL via Redis → duplicate returns 200 immediately. Otherwise: validate schema → route to internal Redis Stream → 200. Malformed schema → DLQ insert + admin alert + 202.

**Acceptance:**
- Signature mismatch → 401, audit row, no Stream publication
- Duplicate event → 200, no second Stream publication, idempotency-hit counter increments
- Malformed payload → row in `webhook_dlq`, no Stream publication, 202 (acknowledge receipt to prevent SirmaAI retry storm)
- Rate-limited inbound (extremely unlikely but defensive): if Redis SETNX fails (Redis down), fail-closed 503 to encourage SirmaAI retry rather than risk dropped event

---

### S28.04: Internal event routing
**Points:** 3 | **Type:** backend

Map SirmaAI event types to internal Redis Streams per architecture amendment §5.3:
- `sirmaai.workflow.completed` → `data-pipeline.opportunity_completion` (E26 consumer) + `client-api.user_notifications` (notification surface)
- `sirmaai.agent.run.completed` → `gateway.workflow_runs_converge` (S04.26 reconciler input) + `notification.run_completion_alerts`
- `sirmaai.kb.file.processed` → `client-api.kb_file_processed` (E25 S25.03 consumer)
- `sirmaai.policy.violation` → `notification.policy_alerts` + `shared.audit_log`

Idempotency-discipline rule: all consumers MUST guard updates with `UPDATE ... WHERE status IN ('pending','running')` since the reconciler may have already converged the row.

**Acceptance:**
- Each event type → expected consumer Stream(s)
- Consumer idempotency test: webhook + reconciler race → final state correct exactly once
- Stream consumer-group lag exported as Prometheus metric

---

### S28.05: Reconciler hardening + observability
**Points:** 3 | **Type:** backend

Builds on E04 S04.26 (which delivered the basic 5-min reconciler). Hardenings: skip rows where `last_polled_at < now() - 30s` (prevent thundering herd on staggered deploys); exponential backoff on SirmaAI rate-limit (back off to 10-min cadence per affected tenant on 429); export histogram `sirmaai_run_to_terminal_seconds` and gauges named above. Alert: PagerDuty (per ADR-010 — email + Telegram for current launch posture) on `sirmaai_workflow_runs_nonterminal_total > threshold` sustained for 30 minutes.

**Acceptance:**
- Reconciler completes scan within 5-minute cadence at 10K non-terminal rows (load test)
- Rate-limit handling: 429 from SirmaAI → backoff applied; metric `sirmaai_reconciler_rate_limit_backoff_active` set
- Alert fires on sustained backlog

---

### S28.06: `gateway.webhook_dlq` migration + DLQ admin surface
**Points:** 3 | **Type:** backend + frontend

**Alembic migration** creating `gateway.webhook_dlq` per the epic schema section above (with `received_at` index partial on `resolved_at IS NULL` for hot-row scans). Closes readiness Concern #2 (DLQ table migration).

Admin endpoints:
- `GET /api/v1/admin/webhook-dlq?status=unresolved&limit=50` — list with payload excerpt
- `GET /api/v1/admin/webhook-dlq/{id}` — full payload + failure reason history
- `POST /api/v1/admin/webhook-dlq/{id}/replay` — re-enqueue (idempotency cache cleared first); records replay in audit
- `POST /api/v1/admin/webhook-dlq/{id}/mark-resolved` — manual close (e.g. legacy event no longer valid)

Admin UI page lists DLQ entries with filter + replay button.

**Acceptance:**
- Migration creates table + partial index; downgrade clean
- Replay round-trip: malformed event → DLQ → fix schema bug → replay → successfully processed
- Mark-resolved is auditable (`resolved_by`, `resolved_at`, `resolution_note`)

---

### S28.07: Tenant-visible degraded-mode banner
**Points:** 2 | **Type:** backend + frontend

On circuit-breaker open >5 minutes against SirmaAI (existing circuit-breaker state gauge per ADR-004), publish `platform.degraded_mode` event to Redis Streams. `client-api` exposes `GET /api/v1/system/status` endpoint with degraded-mode state; frontend layout subscribes (poll every 30s) and renders a banner: "AI analysis temporarily unavailable. Existing analyses are unaffected; new analyses will resume automatically." Banner clears on circuit-breaker recovery + 1-min hysteresis.

**Acceptance:**
- Forced circuit-open on staging → banner appears within 1 minute
- Circuit recovery → banner clears within 2 minutes
- WCAG 2.1 AA: banner `role="status"` with `aria-live="polite"`, contrast OK

---

### S28.08: Cross-tenant negative + idempotency-rule tests
**Points:** 2 | **Type:** integration

Test suite covering:
- Webhook for tenant A's Project routed only to tenant-A-scoped consumers
- Tenant B subscriber cannot read tenant A's events from Redis Streams (existing scope-filter pattern)
- Webhook + reconciler race → consumer guard prevents double-update
- Webhook dropped + reconciler recovers within 5 minutes
- HMAC rotation in-flight: events signed with old secret during 24h overlap accepted; events signed with neither rejected after overlap

**Acceptance:**
- 10+ negative scenarios pass
- Load test: 1000 concurrent webhooks + reconciler scan → no data corruption

## Salvaged patterns

- HMAC verification via `hmac.compare_digest()` (Rule 48, existing canonical pattern from Stripe webhooks).
- Redis SETNX idempotency (Epic 9 dual-layer pattern; same approach as existing webhook dedup).
- Fernet at-rest secret encryption (Epic 9 canonical).
- Celery Beat scheduling discipline (no `_RUN_ID_REGISTRY` anti-pattern from E05 — Beat tasks remain stateless).
- Circuit-breaker state gauge for degraded-mode signal (existing ADR-004 telemetry).

## Out of scope

- Webhook delivery from EU Solicit → SirmaAI (one-way from SirmaAI to us for now; if SirmaAI ever needs us to push events, separate epic).
- Multi-region webhook receiver redundancy (single-host launch per ADR-010 — Hetzner Storage Box has the backup story).
- Self-service webhook subscription per tenant (subscriptions are platform-level under the singleton Org per ADR-018; tenants don't manage their own).
