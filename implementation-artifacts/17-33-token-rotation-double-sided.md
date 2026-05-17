# Story 17.33: Token Rotation Double-Sided

**Status:** backlog
**Epic:** E17 amendment / E27 — CRM via SirmaAI MCP
**Points:** 5
**Type:** backend
**Dependencies:** S17.32 (OAuth + token vault)
**Blocks:** safe long-term CRM ops
**Created:** 2026-05-15
**Source:** E17 amendment §Inject

## Story

As a **platform engineer**,
I want **a Celery Beat task that refreshes expiring provider OAuth tokens AND pushes refreshed tokens to SirmaAI MCP-server secrets atomically**,
so that **MCP-tool invocations never fail because either side has an expired token**.

## Acceptance Criteria

1. Celery Beat task `rotate_crm_oauth_tokens` runs every 6 hours.
2. Scans `client.crm_connections` WHERE `expires_at < now() + INTERVAL '7 days'` AND `refresh_token_encrypted IS NOT NULL`.
3. For each row:
   - (a) Decrypt refresh token.
   - (b) Call provider's token-refresh endpoint (HubSpot OAuth refresh, Dynamics OAuth refresh).
   - (c) Receive new `access_token` + (optionally) new `refresh_token`.
   - (d) Push new access token to SirmaAI MCP-server secrets.
   - (e) On SirmaAI push success: update `client.crm_connections` with new tokens + new `expires_at`.
   - (f) On SirmaAI push failure: abort rotation, alert admin, do NOT update EU Solicit row.
4. **Double-validation overlap**: MCP-server side accepts both tokens during a brief window (handled by SirmaAI's MCP-server config — verify).
5. **Provider 4xx (refresh token revoked)** → set `client.crm_connections.invalidated_at=now()`; alert tenant admin via in-app notification ("Your CRM connection has expired — please reconnect").
6. **Audit log**: each rotation attempt writes audit row.
7. **Integration test** on staging with synthetic 1-min expiry.

## Dev Notes

### Pattern reuse
- Celery Beat schedule.
- Provider OAuth refresh: see existing Google Calendar refresh in E09.

### Files likely touched
- `services/client-api/src/client_api/tasks/crm_token_rotation.py` (new)
- `services/client-api/src/client_api/celery_app.py` (Beat entry)
- `services/client-api/src/client_api/services/crm_oauth_service.py` (extend with `refresh_token`)

### Out of scope
- User re-connect flow on revoke (uses existing S17.32 endpoint).

## Risks

- **R1**: Provider rate-limits on refresh-token endpoint — back off.

## Testing

- Integration: synthetic short expiry on staging.
- Failure: provider revoke / SirmaAI push failure.

## See also

- E17 amendment §S17.33
- PRD amendment NFR-24
