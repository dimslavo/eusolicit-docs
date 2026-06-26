# AgenticSAI Agent Inventory Runbook

Story: S04.23 — Logical-Name Resolution via `agent_map`
Owner: Platform Team

---

## Overview

`AgenticSAIAgentResolver` maps EU Solicit logical agent names (e.g. `"proposal_drafter"`)
to per-tenant AgenticSAI Project agent UUIDs by reading `client.agenticsai_projects.agent_map`
via the 5-minute Redis-cached `ProjectCache`.

On a cache miss, the resolver calls `GET /client/api/v1/agents` against the AgenticSAI
Project API (per-tenant bearer token) to fetch the current inventory, merges the result
into the DB row, and invalidates the cache entry.  A second-pass miss returns HTTP 503
with `Retry-After: 60`.

---

## Manually Trigger a Re-sync

If a tenant's `agent_map` is stale or empty, you can force an inventory refresh by:

1. **Invalidating the Redis cache entry** (forces next resolve to hit the DB, then
   re-sync if the agent is still missing):

   ```bash
   redis-cli -u "$REDIS_URL" DEL "agenticsai:project:by-company:<company_uuid>"
   ```

2. **Clearing `agent_map` in the DB** (forces a full re-sync on the next resolve):

   ```bash
   # Run as migration_role or a superuser — ai_gateway_role can only UPDATE agent_map,
   # not DELETE or TRUNCATE rows.
   psql "$DATABASE_URL" -c "UPDATE client.agenticsai_projects SET agent_map = '{}' WHERE company_id = '<company_uuid>'"
   ```

3. **Trigger resolve via an API call** (the resolver re-syncs automatically on miss):

   ```bash
   curl -s -X POST "https://app.eusolicit.eu/api/v1/agents/proposal-drafter/run" \
     -H "Authorization: Bearer <user_jwt>" \
     -H "X-Company-Id: <company_uuid>" \
     -H "Content-Type: application/json" \
     -d '{"message": "test"}' | jq .
   ```

   - If the agent resolves: re-sync succeeded.
   - If you get `503 AGENT_UNAVAILABLE`: check AgenticSAI Project API health and the
     circuit breaker state (see below).

---

## Spot a Stuck `"agenticsai_agent_inventory"` Circuit

The inventory client uses a single shared circuit breaker keyed `"agenticsai_agent_inventory"`.
When the circuit trips OPEN, ALL tenant inventory re-syncs fail until the cooldown expires.

**Check via Prometheus:**

```promql
# Circuit is open when failure_count >= circuit_breaker_threshold
agenticsai_agent_inventory_resyncs_total{outcome="failed"}
agenticsai_agent_resolver_misses_total{outcome="resync_miss"}
```

**Check via the admin API:**

```bash
curl -s "http://agenticsai-gateway:8004/admin/circuits" | jq '.[] | select(.agent_name == "agenticsai_agent_inventory")'
```

Look for `"circuit_state": "open"` and `"failure_count"` near or above the threshold
(`CIRCUIT_BREAKER_THRESHOLD` env var, default 5).

**Reset the circuit (if AgenticSAI is healthy again):**

```bash
# The circuit automatically transitions to HALF_OPEN after CIRCUIT_BREAKER_COOLDOWN
# seconds (default 30s).  The next successful call closes it.
# If needed, restart the gateway pod to force a clean circuit state.
docker compose restart agenticsai-gateway
```

---

## What to Do When `resync_miss` Rises

Monitor:

```promql
rate(agenticsai_agent_resolver_misses_total{outcome="resync_miss"}[5m])
```

If this metric is non-zero and rising:

1. **Check AgenticSAI Project provisioning** for the affected tenants.
   Log event `agent_resolver.terminal_miss` includes `company_id` and `logical_name`.
   Look for entries in structured logs:

   ```bash
   kubectl logs -l app=agenticsai-gateway | jq 'select(.event == "agent_resolver.terminal_miss")'
   ```

2. **Check that the logical name exists in the AgenticSAI Project.**
   Each tenant has a distinct AgenticSAI Project; the `slug` or normalised `name` in that
   Project must match the EU Solicit logical name.  If the agent was never created in
   AgenticSAI, the re-sync will always miss.

3. **Escalate to Epic E24 (tenant provisioning)** if the AgenticSAI Project was not
   correctly provisioned with the required agents.  The `agent_map` is populated
   during provisioning; a missing entry usually indicates a provisioning defect.

4. **Circuit open?** See the circuit-breaker section above.  A stuck circuit causes
   ALL re-syncs to fail, not just one tenant.  Distinguish: if `resync_miss` rises for
   many tenants simultaneously → likely a circuit or AgenticSAI outage.  If only one
   tenant → likely a mis-provisioning.

---

## Key Prometheus Counters

| Counter | Labels | Meaning |
|---|---|---|
| `agenticsai_agent_resolver_hits_total` | `outcome=cache_hit` | Resolved from cache without DB/API call |
| `agenticsai_agent_resolver_hits_total` | `outcome=resync_hit` | Resolved after inventory re-sync |
| `agenticsai_agent_resolver_misses_total` | `outcome=resync_attempted` | Re-sync triggered (map miss) |
| `agenticsai_agent_resolver_misses_total` | `outcome=resync_miss` | Still missing after re-sync → 503 |
| `agenticsai_agent_inventory_resyncs_total` | `outcome=success` | Inventory fetched and DB updated |
| `agenticsai_agent_inventory_resyncs_total` | `outcome=failed` | Inventory call failed (circuit/timeout/API) |
| `agenticsai_agent_inventory_resyncs_total` | `outcome=locked` | Row locked by concurrent re-sync (SKIP LOCKED) |

---

## Related Runbooks

- `agenticsai-key-rotation.md` — per-Project API key rotation (S04.22)
- `agenticsai-provisioning.md` — tenant provisioning pipeline (E24)

## References

- Story: `eusolicit-docs/implementation-artifacts/4-23-logical-name-resolution-via-agent-map.md`
- Architecture: `eusolicit-docs/planning-artifacts/architecture-amendment-2026-05-12-agenticsai.md` §4.4
- AgenticSAI API: `eusolicit-docs/agenticsai-reference-docs/api-docs v3.json` operationId `listAgents`
