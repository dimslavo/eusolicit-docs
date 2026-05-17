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

---

## 2026-05-12 Amendment — CRM Scope Swap: Dynamics 365 + HubSpot via SirmaAI MCP

> Trigger: `sprint-change-proposal-2026-05-12-sirmaai.md` (approved 2026-05-12). Pairs with `architecture-amendment-2026-05-12-sirmaai.md` (ADR-020) and `prd-amendment-2026-05-12-sirmaai.md` (FR-53 + Integrations rewrite).
> Original Epic 17 shipped (`epic-17: done` in sprint-status): HubSpot direct adapter (S17.01), Pipedrive (S17.02), Salesforce (S17.03). This amendment **scope-swaps** the v1 CRM lineup: HubSpot retained, Dynamics 365 added, Pipedrive + Salesforce deferred to post-launch. Delivery vehicle changes from direct provider adapters in `integrations-api` to **SirmaAI MCP servers registered per Project**.

### Amended Title

`E17: CRM Integrations via SirmaAI MCP — Dynamics 365 + HubSpot` (was: `CRM Integrations (HubSpot, Pipedrive, Salesforce)`)

### Amended Goal

Deliver bi-directional CRM connectivity for **Microsoft Dynamics 365** and **HubSpot** exposed as **SirmaAI MCP servers** registered per tenant Project. Each MCP server exposes a stable tool surface — `find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note` — callable by SirmaAI agents during qualification, opportunity-lifecycle transitions, and on-demand enrichment. EU Solicit hosts the OAuth callback and stores access + refresh tokens Fernet-encrypted in `client.crm_connections`; tokens are injected into MCP-server config at registration (and on rotation) via SirmaAI's secrets API. Pipedrive and Salesforce **deferred to post-launch** as additional MCP-server registrations following the same pattern (no architectural change required).

### Amended Acceptance Criteria

- [ ] Two MCP server registrations per tenant Project at provisioning time (per E24): `dynamics365`, `hubspot`. Initial state `inactive` until OAuth-connected by tenant admin.
- [ ] OAuth callback flow per provider: redirect to provider auth URL → callback handler verifies state nonce → token vault stores encrypted refresh + access tokens with rotation tracking. EU Solicit hosts the callback; SirmaAI never touches the OAuth dance.
- [ ] On successful OAuth: tokens pushed to the corresponding SirmaAI MCP-server's secrets via `POST /api/organizations/{orgId}/secrets`; `client.sirmaai_mcp_servers.status` set to `registered`.
- [ ] MCP tools exposed (5 minimum per provider): `find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`. Each tool's schema captured in the MCP-server registration.
- [ ] SirmaAI agents (qualification, lifecycle-transition, on-demand enrichment) can call MCP tools mid-run via `POST .../mcp-servers/{id}/tools/{toolName}/call`.
- [ ] Token rotation Celery Beat (every 6h): for tokens approaching expiry, refresh via provider OAuth → re-push to SirmaAI secrets → verify reachability → only then update `client.crm_connections.expires_at` (double-sided rotation per ADR-020).
- [ ] Conflict resolution via Last-Write-Wins: `integrations.conflict_log` repurposed to log MCP-tool invocation outcomes that conflict with subsequent CRM-webhook reflections; same audit shape, new source semantics.
- [ ] Tier-gated to Pro+ and above (existing TierGate Depends, Epic 6 pattern); Professional tier sees upgrade prompt on CRM connect UI.
- [ ] Per-provider rate-limit state moves to SirmaAI side (per ADR-020); EU Solicit's circuit-breaker now applies only to the EU Solicit → SirmaAI call (sirmaai-gateway, ADR-004 addendum).
- [ ] Cross-tenant negative test: tenant A's agent cannot invoke MCP tools in tenant B's Project (enforced by SirmaAI api-key scope; EU Solicit verifies via post-call audit trail).
- [ ] Token revocation on workspace archive: `client.crm_connections.status = revoked` triggers MCP-server secret deletion in SirmaAI within 24 hours.
- [ ] CRM dashboard widget per workspace: connection health (token validity, last rotation), MCP-tool invocation count last 7 days, conflict log, agent runs that consumed CRM tools.

### Stories — Amendment Delta

**Retire:**

| Story | Reason |
|---|---|
| S17.01 HubSpot Adapter — Full Bi-Directional Sync (Deals + Contacts) — **direct HTTP adapter portions** | HTTP-to-HubSpot now lives in SirmaAI MCP server, not `integrations-api` |
| S17.02 Pipedrive Adapter | **Deferred to post-launch** as additional MCP-server registration |
| S17.03 Salesforce Adapter | **Deferred to post-launch** as additional MCP-server registration |
| S17.00 portions: per-provider Celery sync queue, direct provider adapter base class | Per-provider HTTP now SirmaAI's responsibility |

**Inject:**

| Story | Pts | Type | Description |
|---|---|---|---|
| **S17.30a Dynamics 365 MCP server — tool spec authoring** *(split 2026-05-15 from former 8pt S17.30 per IR remediation)* | 4 | backend | Author Dynamics 365 MCP-server tool spec for the 5 tools (`find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`). Tool schema captures Dynamics-specific fields (entity types, custom field IDs, OptionSet enums). Per-tool input/output JSON Schema; error-mapping table (Dataverse Web API error codes → MCP error responses). Spec lives at `services/integrations-api/config/sirmaai_mcp_specs/dynamics365.yaml`. PR-reviewed and versioned. |
| **S17.30b Dynamics 365 MCP server — registration flow + sandbox tests** *(split 2026-05-15 from former 8pt S17.30 per IR remediation)* | 4 | backend + integration | Implement the registration flow consuming S17.30a's spec: at tenant provisioning (E24 S24.04 calls this path), register inactive MCP-server in SirmaAI Project; activate on OAuth completion (handoff to S17.32). Integration tests against a Dynamics 365 sandbox cover all 5 tools end-to-end. Negative paths: malformed spec, sandbox 401, sandbox rate-limit. |
| **S17.31 HubSpot MCP server — tool spec + registration** | 5 | backend + integration | Author HubSpot MCP-server tool spec (5 tools). Same flow as S17.30 — HubSpot was simpler in the original E17, simpler here too. Tests against HubSpot sandbox. |
| **S17.32 OAuth callback hosting + Fernet token vault** | 5 | backend | Salvaged from original S17.00. OAuth flow per provider: `GET /api/v1/workspaces/:id/crm/:provider/connect` → returns provider auth URL with state nonce; `GET /api/v1/crm/oauth/callback` → validates state, exchanges code, stores Fernet-encrypted. **Now also pushes tokens to SirmaAI MCP-server secrets at activation.** |
| **S17.33 Token rotation double-sided** | 5 | backend | Celery Beat (every 6h). For tokens within 7 days of expiry: refresh via provider OAuth → push new token to SirmaAI secrets → verify by invoking MCP-server health check → update `client.crm_connections.expires_at`. Rollback path: if SirmaAI push fails, abort rotation, alert admin. |
| **S17.34 MCP-tool invocation log + conflict resolution** | 3 | backend | Repurpose `integrations.conflict_log`: record MCP-tool invocations and their outcomes; on subsequent CRM webhook reflection that conflicts with the tool's intended state change, log + apply LWW. Tenant-visible conflict surface unchanged. |
| **S17.35 Tier-gate Pro+ + workspace CRM dashboard widget** | 3 | full-stack | TierGate Depends on CRM connect endpoints (existing pattern). Frontend widget: connection health, tool-invocation count, conflict log link. |
| **S17.36 Workspace archive → MCP secret deletion** | 2 | backend | On workspace archive event: revoke OAuth tokens at provider + delete MCP-server secrets in SirmaAI + transition `client.sirmaai_mcp_servers.status = inactive`. Audit-logged. |

**Total amendment points:** ~31 (unchanged after 2026-05-15 split of S17.30 8pts → S17.30a 4pts + S17.30b 4pts). Sprint placement: after E04 amendment (needs sirmaai-gateway), E24 (needs Project per company), E25 (KB grounding for enrichment context). Parallel-trackable with E26 (agents that call CRM tools).

### Dependencies

- **Inputs:** E04 amendment (sirmaai-gateway), E24 (Project provisioning + MCP-server registration at provisioning time), E25 (KB context for enrichment agents).
- **Outputs:** Unblocks the CRM-enrichment story arc inside E26 (qualification agents that auto-enrich leads via MCP tools).

### Salvaged from original Epic 17

`client.crm_connections` table + Fernet token vault pattern (Epic 9 canonical, retained verbatim), OAuth state-nonce flow (Rule 39, retained), webhook HMAC validation pattern (Rule 48, retained), `integrations.conflict_log` table (repurposed), two-layer resilience pattern (now applied to EU Solicit → SirmaAI only, not to direct CRM), tier-gate Pro+ pattern, workspace dashboard widget surface.

### Post-launch additions (out of scope for this amendment)

- **E17.B Pipedrive MCP server** — same pattern as S17.30/S17.31. Lift HubSpot tool spec, swap auth flow + endpoints + rate-limit details. ~8 pts.
- **E17.C Salesforce MCP server** — same pattern. Salesforce's daily-quota model needs explicit handling in MCP-server config. ~13 pts (Salesforce remains the most complex).

These do not change the EU Solicit contract; they are SirmaAI-side additions invoked by existing agents.
