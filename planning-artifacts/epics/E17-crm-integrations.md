# E17: CRM Integrations (HubSpot, Pipedrive, Salesforce)

**Sprint**: 18–22 | **Points**: 55 | **Dependencies**: E14, E15, E16 | **Milestone**: Pro+ tier deal-blocker resolution
**Source:** PRD v1.1 §6 FR11.1–11.3 + §8 US12; architecture-evaluation §2 Change-3, §11 locked decision §11.6 (HubSpot → Pipedrive → Salesforce order)

## Goal

Deliver bi-directional CRM sync for the three providers consulting firms in BG/CEE/SME use most: HubSpot (first; highest BG/CEE/SME density), Pipedrive (second; lower complexity, fast follow), Salesforce (third; most complex, daily-quota-aware, ~6-week add). Without CRM integration, EU Solicit becomes shelfware in consulting-firm bid-pursuit workflows where Loopio's #1 sales talking point is Salesforce sync. Per-workspace OAuth2 connections store tokens via Fernet encryption (project-context Epic 9 pattern). Bi-directional sync handles conflict resolution via Last-Write-Wins with full audit-trail entries (ISO 27001 evidence). Tier-gated to Pro+ and above.

## Acceptance Criteria

- [ ] `client.crm_connections` table stores per-workspace OAuth tokens encrypted at rest via Fernet (project-context Epic 9: "Fernet encryption canonical module for OAuth tokens")
- [ ] OAuth2 connection flow per provider: redirect to provider auth URL → callback handler verifies state → token vault stores encrypted refresh + access tokens with rotation tracking
- [ ] On opportunity added to workspace, optionally auto-create corresponding Deal/Opportunity in CRM (configurable per workspace: auto / prompt / off)
- [ ] On opportunity status change (qualified / bid / submitted / won / lost), CRM Deal stage updates within 60 seconds
- [ ] On CRM Deal/Opportunity update (stage / contact / value), EU Solicit reflects change within 5 minutes (poll-based + webhook-based hybrid)
- [ ] Conflict resolution: Last-Write-Wins with `integrations.conflict_log` entry per conflict; user-visible conflict log per workspace
- [ ] Tier-gate Pro+ enforced via TierGate Depends (Epic 6 pattern); Professional users see upgrade prompt
- [ ] All outbound HTTP to CRMs uses two-layer resilience pattern (project-context Rule 47): circuit breaker per workspace+provider wrapping retry with exponential backoff
- [ ] Per-provider rate-limit handling: HubSpot 100/10s, Salesforce daily quota tracking, Pipedrive 100/2s — circuit breaker opens on 429s with vendor-specific backoff
- [ ] OAuth tokens rotated on schedule via Celery Beat task (per project-context Epic 9)
- [ ] Workspace-scoped negative test (W1's HubSpot connection cannot sync W2's opportunities even within same tenant)
- [ ] Token revocation on workspace archive
- [ ] `integrations.sync_logs` records every sync direction (inbound/outbound), entity, status; 30-day retention
- [ ] CRM dashboard widget per workspace shows: connection health, last sync, conflict log, rate-limit status

## Stories

### S17.00: CRM Connection Model + Fernet Token Vault + Sync Engine Scaffold + Conflict-Resolution Framework
**Points**: 13 | **Type**: backend

**Database:**
- Create `client.crm_connections` table: `id`, `workspace_id` FK, `provider` (enum: hubspot/salesforce/pipedrive), `encrypted_oauth` (Fernet-encrypted blob containing access_token, refresh_token, expires_at), `status` (active/revoked/error), `last_synced_at`, `created_at`, UNIQUE(`workspace_id`, `provider`)
- Create `integrations.conflict_log` table: `id`, `crm_connection_id`, `entity_type`, `entity_id`, `eu_solicit_value` JSONB, `crm_value` JSONB, `resolution` (lww_won_local/lww_won_remote), `occurred_at`

**Service-side scaffold (in `integrations-api`):**
- Generic `CRMAdapter` abstract base class with methods: `authenticate()`, `refresh_token()`, `create_deal()`, `update_deal()`, `read_deal()`, `webhook_handler()`, `rate_limit_status()`
- Per-provider implementations stub-able for S17.01–S17.03 to inherit
- Sync engine: Celery worker consumes `opportunity.created` / `opportunity.status_changed` events from EventBus → routes to provider adapter → logs to `integrations.sync_logs`
- Reverse sync: Celery Beat task polls active CRMs every 15 min for changed Deals → reconciles with EU Solicit opportunities → fires `crm_deal_changed` event for downstream processing
- Conflict detection: each sync compares timestamps; if both sides changed since last sync, create `conflict_log` entry, apply LWW (latest `updated_at` wins), emit `subscription.changed`-style event for UI surfacing
- Token rotation Celery Beat task (every 6 hours) refreshes tokens approaching expiry

**OAuth flow:**
- `GET /api/v1/workspaces/:id/crm/:provider/connect` → returns provider auth URL with state nonce
- `GET /api/v1/crm/oauth/callback` → validates state nonce against session store, exchanges code for tokens, stores encrypted, redirects to settings page

**Resilience:**
- Two-layer pattern (Rule 47): `circuit_breaker(retry(http_factory))` per (workspace, provider) logical name
- Per-provider rate-limit configuration: HubSpot 100/10s sliding window, Salesforce daily-quota-aware (separate state machine), Pipedrive 100/2s
- Circuit breaker opens on 429s, holds for vendor-specific cooldown, surfaces "Rate-limit reached, paused for N minutes" in workspace UI

**Tests:**
- Workspace-scoped negative tests for all CRM endpoints
- Token-encryption round-trip test
- LWW conflict-resolution test with timestamp ties (favours remote per design)
- Circuit-breaker integration test (5xx storm → opens → recovers)

---

### S17.01: HubSpot Adapter — Full Bi-Directional Sync (Deals + Contacts)
**Points**: 13 | **Type**: backend | **First provider per locked decision**

Implement `HubSpotAdapter(CRMAdapter)`:
- OAuth2 flow with HubSpot's `auth.hubspot.com` (state nonce verified per project-context Rule 39: "OAuth callback must validate the state parameter")
- API client uses `hubspot-api-client` SDK or direct httpx with HubSpot v3 API; wrapped in two-layer resilience
- Deal create on EU Solicit `opportunity.created` event: maps `opportunity.title → deal.dealname`, `opportunity.value_eur → deal.amount`, `opportunity.deadline → deal.closedate`, `opportunity.status → deal.dealstage` (configurable stage mapping per workspace)
- Deal update on `opportunity.status_changed` event: stage transitions follow workspace-configured mapping table
- Reverse sync: webhook handler at `POST /webhooks/crm/hubspot` validates webhook secret (HMAC, project-context Rule 48 — `hmac.compare_digest`), processes `deal.propertyChange` events, updates corresponding EU Solicit opportunity
- Contact sync: bi-directional sync of associated contacts with email/phone/role
- Rate-limit handling: HubSpot's 100/10s sliding window with token-bucket; circuit breaker opens on 429s with 60-second cooldown
- Stage mapping configurable per workspace via Admin API; default mapping seeded

**Tests:**
- Full bi-directional sync E2E with HubSpot sandbox account
- Webhook signature validation test (project-context Rule 48 timing-safe comparison)
- Rate-limit recovery test (simulated 429 → backoff → recover)
- Conflict scenario: edit Deal in HubSpot AND update opportunity in EU Solicit within same window → LWW resolution + `conflict_log` entry

---

### S17.02: Pipedrive Adapter
**Points**: 13 | **Type**: backend | **Second provider — lower complexity**

Mirror S17.01 pattern with Pipedrive specifics:
- OAuth2 with `oauth.pipedrive.com`
- API uses `pipedrive` SDK or direct httpx
- Entity mapping: opportunity → deal, with workspace-configured pipeline mapping
- Rate-limit: 100/2s — different from HubSpot, so per-provider config in CRMAdapter base class
- Webhook subscriptions via Pipedrive webhook API; HMAC signature validation
- Same conflict-resolution framework (S17.00) applies

Tests parallel to S17.01.

---

### S17.03: Salesforce Adapter — Daily-Quota-Aware
**Points**: 16 | **Type**: backend | **Third provider — most complex; ~6-week add**

Mirror S17.01–S17.02 pattern with Salesforce's tighter constraints:
- OAuth2 with `login.salesforce.com` (or sandbox-specific URL); refresh-token grant with extended TTL
- API uses `simple-salesforce` SDK or direct httpx with REST API; Apex callouts NOT used (out of scope)
- Entity mapping: opportunity → Opportunity object, with workspace-configured stage mapping
- **Daily-quota tracking:** dedicated state machine in `integrations.sync_logs` aggregates per-org per-day call counts; circuit breaker opens at 80% of daily quota with explicit "approaching quota" warning to workspace UI; hard-stops at 100% with "quota exhausted, resumes at midnight UTC" message
- Streaming API (CometD) for inbound change capture optional — defer to post-MVP if scope tight; webhook + 15-min polling sufficient for MVP
- Higher complexity: Salesforce custom-fields, multi-currency, sandbox-vs-production handling
- Per-org rate-limit (different from per-user limits): tracked separately

Tests:
- Quota state machine: simulated 80% threshold → warning event; 100% → hard-stop + recovery on next day rollover
- Sandbox-vs-production environment switching
- Custom-field mapping configurable
