# E27: CRM via SirmaAI MCP — Dynamics 365 + HubSpot

**Sprint:** post-pivot S+3 | **Points:** 34 | **Dependencies:** E04 amendment, E24, E25 | **Milestone:** SirmaAI Pivot

> **Source:** `sprint-change-proposal-2026-05-12-sirmaai.md`, `prd-amendment-2026-05-12-sirmaai.md` (FR-53), `architecture-amendment-2026-05-12-sirmaai.md` (ADR-020). Implementation overlaps with **E17 amendment** — E27 is the new-build narrative for the same scope. Whichever file owns delivery, the other references it; suggest **E27 is the authoritative source for the post-pivot CRM lifecycle** and E17 amendment is the bridging delta on the original epic record.

## Goal

Deliver bi-directional CRM connectivity for **Microsoft Dynamics 365** and **HubSpot** through SirmaAI **MCP servers** registered per tenant Project. Each MCP server exposes a stable tool surface (`find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`) callable by SirmaAI agents during qualification (lead enrichment in-flight), opportunity-lifecycle transitions (deal stage updates), and on-demand enrichment (user-triggered "enrich this account" from opportunity detail). EU Solicit hosts the OAuth callback and stores tokens Fernet-encrypted in `client.crm_connections`; tokens are injected into MCP-server config at registration via SirmaAI's secrets API. Pipedrive and Salesforce deferred to post-launch as additional MCP-server registrations.

This epic and the **E17 amendment** describe overlapping work. E27 is the **forward-looking authoritative spec** for the post-pivot CRM lifecycle; E17 amendment is the **bridge** that records the scope swap on the original epic file. Stories in this file are the canonical execution unit.

## Acceptance Criteria

(See **E17 amendment §Acceptance Criteria** — same set, repeated here for completeness.)

- [ ] Two MCP server registrations per tenant Project at provisioning: `dynamics365`, `hubspot` (status `inactive` until OAuth-connected) — implemented in E24 S24.04
- [ ] OAuth callback flow per provider — EU Solicit-hosted; SirmaAI never sees OAuth dance
- [ ] On successful OAuth: tokens pushed to SirmaAI MCP-server secrets; `client.sirmaai_mcp_servers.status='registered'`
- [ ] 5 MCP tools exposed per provider: `find_account`, `create_deal`, `update_deal_stage`, `enrich_contact`, `attach_note`
- [ ] Agents (qualification, lifecycle-transition, on-demand enrichment) call MCP tools mid-run
- [ ] Token rotation Celery Beat (6h cadence; double-sided rotation per ADR-020)
- [ ] LWW conflict resolution via repurposed `integrations.conflict_log`
- [ ] Tier-gate Pro+
- [ ] Per-provider rate-limit state on SirmaAI side; EU Solicit's circuit-breaker only on the EU Solicit → SirmaAI path
- [ ] Cross-tenant negative test
- [ ] Token revocation on workspace archive
- [ ] CRM dashboard widget per workspace

## Stories

(See **E17 amendment §Stories — Amendment Delta** — canonical list:)

| Story | Pts | Type | Description |
|---|---|---|---|
| **S27.01 = S17.30** Dynamics 365 MCP server — tool spec + registration | 8 | backend + integration | Author Dynamics tool spec; register on tenant provisioning |
| **S27.02 = S17.31** HubSpot MCP server — tool spec + registration | 5 | backend + integration | Author HubSpot tool spec; register on tenant provisioning |
| **S27.03 = S17.32** OAuth callback hosting + Fernet token vault + SirmaAI secret push | 5 | backend | OAuth flow; on success, push tokens to SirmaAI MCP-server secrets |
| **S27.04 = S17.33** Token rotation double-sided | 5 | backend | 6h Beat; refresh OAuth → push to SirmaAI → verify → update expires_at |
| **S27.05 = S17.34** MCP-tool invocation log + conflict resolution | 3 | backend | Repurpose `integrations.conflict_log` for MCP-tool outcomes; LWW |
| **S27.06 = S17.35** Tier-gate Pro+ + workspace CRM dashboard | 3 | full-stack | TierGate Depends; UI widget |
| **S27.07 = S17.36** Workspace archive → MCP secret deletion | 2 | backend | On archive: revoke tokens at provider + delete SirmaAI secrets + transition status |
| **S27.08** Agent-side MCP-tool consumption patterns | 3 | backend + integration | Document how qualification + quantification + on-demand enrichment agents call MCP tools; eval-runs with mocked CRM responses |

**Total: ~34 pts** (matches sprint change proposal §3.1 estimate).

## Story numbering note

E27 stories use S27.* prefix in this file for clarity. The E17 amendment uses S17.30-S17.36 for the same work in its delta. **Implementation tickets should pick one convention** — recommend S17.30-S17.36 since E17 already has a story-numbering namespace and E17 amendment is what the orchestrator dispatches against. E27 stays as the design narrative.

## Out of scope

- Pipedrive and Salesforce MCP servers (post-launch, separate epics — same pattern).
- Direct CRM HTTP from EU Solicit's `integrations-api` (deliberately retired per ADR-020).
- Custom MCP tools beyond the 5 named (e.g. `query_opportunities`, `merge_duplicates`) — extension path documented but not v1 scope.

## See also

- `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-sirmaai.md` § ADR-020
- `eusolicit-docs/planning-artifacts/prd-amendment-2026-05-12-sirmaai.md` § FR-53
- `eusolicit-docs/planning-artifacts/epics/E17-crm-integrations.md` § 2026-05-12 Amendment (where the work actually lives for orchestrator dispatch)
