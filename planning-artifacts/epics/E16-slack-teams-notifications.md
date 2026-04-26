# E16: Slack & Teams Notifications

**Sprint**: 17–18 | **Points**: 8 | **Dependencies**: E14, E15 | **Milestone**: Pro-tier integration foothold
**Source:** PRD v1.1 §6 FR11.4–11.5; architecture-evaluation §2 Change-3 (sequenced ahead of CRM per architect recommendation)

## Goal

Bootstrap the new `integrations-api` service with the smallest viable integration first — Slack and Microsoft Teams incoming webhooks for new-opportunity alerts, deadline reminders, proposal-generation completion, and compliance-check failures. Pro tier feature (lower bar than Pro+ to maximise reach). Architecturally lighter than CRM (5% the engineering scope) but proves out the new service skeleton for Epic 17 to inherit.

## Acceptance Criteria

- [ ] New `integrations-api` FastAPI service deployed to K8s with Helm chart, ingress route, Fernet-encryption-ready token vault (token storage stays in `client.crm_connections` schema for future Epic 17 reuse — Slack/Teams use simple webhook URLs, no OAuth)
- [ ] `integrations` PostgreSQL schema created; `integrations.sync_logs` table available for both Slack/Teams and future CRM tracking
- [ ] Slack/Teams webhook configuration UI in workspace settings (per workspace; tier-gated to Professional and above via TierGate Depends)
- [ ] Notifications delivered via incoming webhooks (no SDK / interactive bots in this epic — deferred to future)
- [ ] Triggers: new opportunity matching saved filter, bid deadlines (24h / 48h / 7d ahead), proposal generation completion, compliance-check failure
- [ ] Throttling: max 1 notification per opportunity per event-type per workspace (no flood on bulk re-sync)
- [ ] Notification content respects locale (BG/EN); i18n key parity check passes
- [ ] All outbound HTTP to Slack/Teams uses two-layer resilience pattern (project-context Rule 47): circuit breaker (outer, per-workspace logical name) wrapping retry with exponential backoff
- [ ] Workspace-scoped negative test: workspace W1's webhook URL receives notifications only for W1's opportunities, never W2's (even within same tenant)
- [ ] Free/Starter tier users see upgrade-prompt 403 when attempting to configure
- [ ] `integrations-api` health check + Prometheus metrics endpoint live (joins Epic 21 SLO dashboards)

## Stories

### S16.00: integrations-api Service Bootstrap + Slack/Teams Webhook Configuration + Alert Routing
**Points**: 8 | **Type**: fullstack

**Service scaffold:**
- New `services/integrations-api/` following existing service module pattern (FastAPI app, Helm chart, Dockerfile multi-stage build < 200MB, ruff/mypy/pytest configured, eusolicit-common imports)
- New `integrations_api_role` PostgreSQL role with CRUD on `integrations` schema only, read-only on `client.crm_connections` (token vault location for future Epic 17 reuse), write to `shared.audit_log`
- Internal ClusterIP only; not internet-facing (matches AI Gateway exposure pattern)
- Helm chart with PDB + min replicas 2 (Epic 21 reliability prerequisite)

**Database (Alembic migration in `integrations-api`):**
- Create `integrations` schema
- `integrations.sync_logs` table: `id`, `workspace_id`, `provider` (slack/teams/hubspot/...), `direction` (inbound/outbound), `entity_type`, `entity_id`, `status`, `error`, `occurred_at`
- (Conflict log table deferred to Epic 17 since Slack/Teams have no bi-directional sync)

**Configuration UI (frontend in `apps/client/app/[locale]/workspace/[workspaceId]/settings/integrations/`):**
- Provider list: Slack, Microsoft Teams (CRMs greyed-out with "Coming in Epic 17")
- Webhook-URL input + test-message button per provider
- Per-event-type toggles (new opportunity / deadline / proposal generated / compliance failed)
- TierGate Depends Pro+ tier (correction: per spec Pro tier — confirm with Deb if Pro vs Pro+; PRD locks Pro)

**Alert routing:**
- Domain events (existing EventBus pattern Epic 9): `opportunities.matched`, `proposal.deadline_approaching`, `proposal.generation_completed`, `compliance_check.failed` consumed by integrations-api worker
- Worker assembles message from i18n template (BG/EN) + dispatches to configured webhook URLs via httpx with two-layer resilience (circuit breaker per workspace+provider; retry with exponential backoff)
- Throttle: Redis SETNX with 1-hour TTL per `(workspace_id, opportunity_id, event_type)` key — prevents flood
- All deliveries logged to `integrations.sync_logs` (success + failure)

**Tests:**
- Workspace-scoped negative test (W1 webhook never receives W2 events)
- Two-layer resilience test (circuit-breaker opens on 5xx; recovers on 2xx)
- Throttle test (10 rapid `opportunities.matched` events for same opportunity → 1 notification)
- i18n parity test (BG and EN templates have matching key set)
- Source-inspection ATDD: webhook handler uses `httpx.AsyncClient` with circuit-breaker decorator (NOT raw `requests` or `urllib`)
- Tier-gate E2E (Playwright): Free user attempting `/settings/integrations` → 403 with upgrade prompt; Professional user can configure
