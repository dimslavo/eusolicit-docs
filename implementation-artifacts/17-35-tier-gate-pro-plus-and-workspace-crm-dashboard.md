# Story 17.35: Tier-Gate Pro+ + Workspace CRM Dashboard

**Status:** backlog
**Epic:** E17 amendment / E27 — CRM via SirmaAI MCP
**Points:** 3
**Type:** fullstack
**Dependencies:** S17.32 (OAuth), S17.34 (conflict log), UX amendment §5.14
**Blocks:** Slice 4 DoD
**Created:** 2026-05-15
**Source:** E17 amendment §Inject

## Story

As **Elena (Pro+ workspace admin)**,
I want **a Settings → CRM Connections page that shows my connected CRMs, their health, tool-invocation counts, and any conflicts**,
so that **I can verify the AI is enriching opportunities correctly AND troubleshoot when something looks wrong**.

## Acceptance Criteria

1. **TierGate Depends** on all CRM-connect endpoints (existing pattern).
2. **Endpoint** `GET /api/v1/workspaces/{id}/crm/status` returns per-provider: `{provider, status, connected_since, last_health_check_at, token_expires_at, tool_invocations_30d}`.
3. **Endpoint** `GET /api/v1/workspaces/{id}/crm/conflicts?limit=50` returns conflict-log entries (existing pattern; surfaced from S17.34).
4. **Frontend page** at `/workspace/[workspaceId]/settings/crm-connections` per UX amendment §5.14:
   - Per-provider tile with status, health, token-expiry countdown.
   - "Connect" / "Disconnect" CTAs (disconnect = typed-confirmation modal warning that SirmaAI secrets will be deleted).
   - Conflict-log link → side panel with sortable conflict list.
   - Empty state: "Free / Starter tier — upgrade to Pro+ to connect CRMs".
5. **OAuth callback redirect**: lands user back on the dashboard with toast confirming success/failure.
6. **WCAG 2.1 AA**: keyboard nav, focus management in modals.
7. **E2E Playwright test**: Pro+ user connects HubSpot → status flips to Active → disconnects via typed confirmation → status flips to Inactive.

## Dev Notes

### Pattern reuse
- TierGate Depends.
- Settings page layout pattern.
- Typed-confirmation modal: see workspace-archive (S24.06).

### Files likely touched
- `services/client-api/src/client_api/routers/crm.py` (extend with status + conflicts endpoints)
- `frontend/apps/client/app/(protected)/workspace/[workspaceId]/settings/crm-connections/page.tsx` (new)
- `frontend/packages/ui/src/components/crm/CrmConnectionTile.tsx` (new)
- `tests/e2e/crm-connections.spec.ts`

### Out of scope
- Conflict-resolution UI (S17.34 backend).
- Tool-invocation log detailed view (admin-only post-launch).

## Risks

- **R1**: Token-expiry countdown timezone — display in user's local TZ.

## Testing

- E2E: full connect/disconnect flow.
- Cross-tenant: tenant-A page never shows tenant-B status.

## See also

- E17 amendment §S17.35
- UX amendment §5.14
- PRD amendment FR-53
