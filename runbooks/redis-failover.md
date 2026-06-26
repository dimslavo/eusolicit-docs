# Runbook: Redis ElastiCache Failover

**Severity**: SEV-1 (if primary unreachable) / SEV-2 (if planned/manual failover)
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6) — lifts operator-facing content from PE.03 `pe-03-cutover-runbook.md` §Failover Drill Steps + §Lua-Script Re-Run

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| `RedisReplicationLagHigh` alert co-firing before failover | Slack `#platform-alerts` |
| `RedisEvictionsObserved` alert co-firing (memory pressure → failover) | Slack `#platform-alerts` |
| Services reporting Redis connection errors simultaneously | Per-service logs |
| Session / rate-limit cache misses (authentication errors, usage-metering gaps) | client-api / notification logs |
| Celery task queue stall (data-pipeline cannot dequeue tasks) | data-pipeline logs |
| AWS ElastiCache Console showing primary node as `degraded` | AWS Console |

---

## Triage

1. **Check ElastiCache replication group status**:
   ```bash
   aws elasticache describe-replication-groups \
     --replication-group-id eusolicit-redis \
     --query 'ReplicationGroups[0].{Status:Status,NodeGroups:NodeGroups}'
   ```
   Expected post-failover: primary node changes; overall status transitions `modifying` → `available`.

2. **Check for AWS-initiated failover event**:
   ```bash
   aws elasticache describe-events \
     --source-identifier eusolicit-redis \
     --duration 60
   ```
   Look for: `"Failover to replica node eusolicit-redis-001 completed"`.

3. **Check per-service Redis connectivity**:
   ```bash
   for svc in client-api admin-api agenticsai-gateway data-pipeline notification integrations-api; do
     kubectl logs -n eusolicit deploy/$svc --since=5m | grep -i "redis" | tail -5
   done
   ```
   Expected: brief connection errors followed by reconnect within ≤10s.

4. **Check consumer-group offsets** (Redis Streams used by notification service):
   ```bash
   redis-cli -h <new-primary-endpoint> -p 6379 --tls \
     XINFO GROUPS <stream-name>
   ```
   Consumer groups should still exist on the new primary (data is replicated).

---

## Resolution

### Branch A — Automated failover (AWS triggered; primary failed)

ElastiCache handles promotion automatically with `automatic_failover_enabled = true` (PE.03).

1. **Wait for AWS to complete promotion** (typically 10–60 seconds; Multi-AZ deployment).
2. **Verify per-service reconnect** — all 6 services MUST reconnect within ≤10s per PE.03 §Failover Drill Steps:
   ```bash
   kubectl get events -n eusolicit --sort-by='.lastTimestamp' | grep -i redis | tail -10
   ```
3. **Run health checks**:
   ```bash
   for svc in client-api admin-api agenticsai-gateway data-pipeline notification integrations-api; do
     kubectl exec -n eusolicit deploy/$svc -- python -c \
       "import redis; r = redis.Redis(host='${REDIS_URL}', ssl=True); print(r.ping())" \
       && echo "$svc: Redis OK" || echo "$svc: Redis FAIL"
   done
   ```

4. **Verify Lua-script atomicity** — usage-metering Lua scripts must re-run correctly after failover per PE.03 §Lua-Script Re-Run:
   ```bash
   # Check notification service usage-metering logs for Lua script errors
   kubectl logs -n eusolicit deploy/notification --since=5m | grep -i "lua\|EVALSHA\|NOSCRIPT"
   ```
   If `NOSCRIPT` errors appear → Lua scripts need to be re-uploaded (scripts are registered on connection; new connection auto-registers them via the `register_scripts()` call pattern).

### Branch B — Manual failover (operator-initiated; primary degraded but not failed)

1. **Trigger failover via AWS CLI**:
   ```bash
   aws elasticache test-failover \
     --replication-group-id eusolicit-redis \
     --node-group-id 0001
   ```
   ⚠️ This is the same API used in the PE.03 §Failover Drill. All in-flight write operations may be lost.

2. **Monitor reconnect timer** — all services expected to reconnect within ≤10s.

3. **Proceed with Branch A verification steps** above.

### Branch C — Memory-pressure-induced failover (evictions co-firing)

If `RedisEvictionsObserved` co-fires with failover:

1. Address memory pressure per `redis-evictions.md` AFTER confirming services are reconnected.
2. Do not trigger manual failover during active eviction pressure (risks dual instability).

---

## Verification

1. **All services reconnect within ≤10s** (PE.03 §Failover Drill Steps invariant):
   - Acceptable reconnect window: ≤10s total per service.

2. **Consumer-group offsets intact** (notification service Redis Streams):
   ```bash
   redis-cli -h <new-primary-endpoint> -p 6379 --tls \
     XINFO GROUPS platform-events
   ```
   Consumer groups must exist with correct `last-delivered-id` — no event replay gap.

3. **Lua scripts functioning** (usage-metering atomicity):
   - Check notification logs for `NOSCRIPT` errors for 5 minutes post-failover.
   - If absent: Lua scripts operational.

4. **Rate-limit cache intact**: test a rate-limited endpoint and confirm counter is consistent.

5. **Celery task queue healthy** (data-pipeline):
   ```bash
   kubectl exec -n eusolicit deploy/data-pipeline -- celery -A tasks inspect active
   ```
   Active tasks should be running within 30s of failover completion.

6. **`RedisReplicationLagHigh` alert clears** after new replica catches up.

---

## Rollback

There is no "undo" for an ElastiCache failover — the former replica is now primary.

1. **If Lua scripts error after failover**: application-level reconnect will re-register them automatically (the `register_scripts()` pattern handles `NOSCRIPT`). If not, rolling-restart the affected service:
   ```bash
   kubectl rollout restart deploy/notification -n eusolicit
   ```

2. **If consumer-group offset loss is detected**: check if events were produced but not consumed during the failover window. The notification service consumer-group `XREADGROUP` will resume from `last-delivered-id` — any events produced during the outage window will be delivered.

3. **If a manual failover worsened the situation**: do NOT immediately re-trigger. Monitor for 5 minutes. Contact AWS Support if both nodes show instability.

---

## Related

- `redis-evictions.md` — memory pressure (often precedes failover)
- `error-budget-burn.md` — burn-rate impact during reconnect window
- `high-latency.md` — latency spike during reconnect
- `deploy-rollback.md` — if application rollback is also needed
- Story 21-3 `pe-03-cutover-runbook.md` §Failover Drill Steps + §Lua-Script Re-Run — reconnect SLA evidence + Lua atomicity check (dev-side audit trail)
