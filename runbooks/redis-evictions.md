# Runbook: Redis ElastiCache Evictions

**Severity**: SEV-2
**SLA-Scope**: in-scope
**Last updated**: 2026-05-05
**Story**: PE.06 (21-6) | **Alertmanager routing**: `severity=ticket` → Slack `#platform-alerts`

---

## Symptoms

| Signal | Where to look |
|--------|---------------|
| `RedisEvictionsObserved` alert fires (`aws_elasticache_evictions_sum` > 0 for 10m) | Slack `#platform-alerts` |
| `RedisReplicationLagHigh` alert co-fires | Slack `#platform-alerts` |
| Cache-miss rate spike (services falling back to DB for every request) | Per-service Grafana dashboard |
| DB connection pool pressure increasing (reads no longer served from cache) | `aws_rds_database_connections_average` CloudWatch |
| Usage-metering Lua scripts returning unexpected results | Notification service logs |
| Customer reports of slow response times (latency degradation from cache miss) | Support channel |

**Alert source**: `infra/observability/prometheus/rules/alerting-rules.yaml` lines 100–124 (PE.05).

---

## Triage

1. **Check ElastiCache memory utilisation**:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/ElastiCache \
     --metric-name DatabaseMemoryUsagePercentage \
     --dimensions Name=ReplicationGroupId,Value=eusolicit-redis \
     --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Average
   ```
   Target: < 80%. Above 80% → eviction pressure increasing.

2. **Check current eviction policy**:
   ```bash
   redis-cli -h <elasticache-primary-endpoint> -p 6379 --tls CONFIG GET maxmemory-policy
   ```
   Default `noeviction` → OOM errors instead of evictions (bad for web cache). `allkeys-lru` → evicts LRU keys.
   `volatile-lru` → only evicts keys with TTL set.

3. **Audit large keys** (read-only — no mutation):
   ```bash
   redis-cli -h <elasticache-primary-endpoint> -p 6379 --tls --bigkeys
   ```
   Look for keys consuming >1MB. Large keys are the most common eviction pressure source.

4. **Check key-count trend**: is the cache growing unboundedly (no TTL on keys)?
   ```bash
   redis-cli -h <elasticache-primary-endpoint> -p 6379 --tls INFO keyspace
   ```
   `keys` count growing while `expires` count is not → missing TTL on some key patterns.

5. **Check eviction rate vs. eviction policy**: if policy is `noeviction` and evictions are occurring → data corruption risk; escalate to SEV-1 immediately.

---

## Resolution

### Branch A — Memory pressure (usage > 80%, eviction policy is LRU-based)

Evictions are occurring but LRU policy is protecting the most-used data. This is expected LRU behaviour.

1. **Short-term**: ensure `maxmemory-policy` is `allkeys-lru` (best for a caching workload):
   ```bash
   # Note: ElastiCache parameter group change (not redis-cli CONFIG SET — ElastiCache requires param group)
   aws elasticache modify-replication-group \
     --replication-group-id eusolicit-redis \
     --cache-parameter-group-name eusolicit-redis-params-lru
   ```
   ⚠️ Param group changes require a maintenance window restart unless `apply-immediately` flag is set.

2. **Medium-term (vertical scaling)**: increase node type to reduce eviction pressure:
   - Current: `cache.t3.medium` (staging) / `cache.r6g.large` (prod).
   - Next tier: `cache.r6g.xlarge` (doubles memory).
   - Follow `redis-failover.md` for the scaling procedure to avoid downtime.

3. **Medium-term (TTL audit)**: identify keys without TTL and add appropriate TTLs in the application code.
   - Large-object cached results (e.g., opportunity-list pages) should have TTL ≤ 5 minutes.

### Branch B — Unbounded key growth (no-eviction policy + key count growing)

⚠️ This is **SEV-1** — `noeviction` policy means writes fail when memory is full.

1. Immediately switch to `allkeys-lru` via parameter group (see Branch A step 1).
2. Emergency key flush for identified large key pattern (ONLY if business-safe):
   ```bash
   redis-cli -h <endpoint> -p 6379 --tls --scan --pattern "<pattern>:*" | xargs redis-cli -h <endpoint> -p 6379 --tls DEL
   ```
   ⚠️ Test pattern match before DEL. Coordinate with application team before bulk-deleting.

3. Add TTL audit to the sprint backlog immediately.

### Branch C — Replication lag co-firing (failover risk from evictions)

→ If `RedisReplicationLagHigh` co-fires AND memory > 85% → consider proactive failover.
→ Follow `redis-failover.md` §Resolution.

---

## Verification

1. **Eviction rate drops to 0**:
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/ElastiCache --metric-name Evictions \
     --dimensions Name=ReplicationGroupId,Value=eusolicit-redis \
     --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 60 --statistics Sum
   ```
   Target: 0 or trending to 0.

2. **Memory utilisation < 80%** (after scaling or key purge).

3. **Cache-hit rate recovering** — per-service Grafana dashboard cache-hit panel returns to baseline.

4. **DB connection pressure declining** — `aws_rds_database_connections_average` returns to baseline.

5. **Alert resolves** — `RedisEvictionsObserved` resolves after evictions drop to 0 for 10m.

---

## Rollback

If parameter-group change causes unexpected issues:

1. Revert to previous parameter group:
   ```bash
   aws elasticache modify-replication-group \
     --replication-group-id eusolicit-redis \
     --cache-parameter-group-name eusolicit-redis-params-default
   ```
2. If a failover was triggered during scaling → follow `redis-failover.md` §Verification.
3. If key flush caused application-layer issues → check service logs; cache will repopulate from DB within minutes.

---

## Related

- `redis-failover.md` — ElastiCache failover procedure (if scaling or eviction triggers failover)
- `high-latency.md` — latency regression from cache-miss cascade
- `error-budget-burn.md` — burn-rate impact from cache miss → DB pressure → latency
- PE.05 `alerting-rules.yaml` lines 100–124 — alert definitions
- Story 21-3 `pe-03-cutover-runbook.md` §Failover Drill Steps + §Lua-Script Re-Run
