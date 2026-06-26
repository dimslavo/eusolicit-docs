# Redis Restore Runbook (Story onprem-02)

**Audience:** on-call engineer in an incident
**Story:** `eusolicit-docs/implementation-artifacts/onprem-02-redis-persistence-and-recovery.md`
**RTO target:** ≤ 15 min (Redis is small; restore is fast)
**RPO target:** ≤ 1 hour (RDB hourly snapshot) or ≤ 1 second (AOF, local only)

---

## When to use which mode

```
DECISION TREE:

Q1: Did Redis crash but the volume is intact?
  YES → docker compose restart redis. AOF + RDB replay automatically on start.
        ETA: ≤30s. No restore action needed.
  NO  → continue.

Q2: Is the redisdata volume corrupted or missing?
  YES (volume gone but RDB file recoverable from `docker cp` of a backup container) → use --to-prod with the latest snapshot
  NO  → check `docker volume inspect eusolicit-app_redisdata`

Q3: Is the host itself lost?
  YES → restore goes hand-in-hand with cross-host postgres restore (onprem-06).
```

Redis losing data is *less* catastrophic than postgres because:
- Cache contents regenerate naturally from postgres queries.
- Celery in-flight tasks: idempotent consumers + Redis-Streams DLQ pattern (ADR-003) means re-delivery is safe.
- Redis-Streams consumer groups: offset is captured in RDB; replay from a slightly-old offset re-processes a few events (idempotent → safe).

What you DO lose with an RDB rollback: any state written between the last RDB snapshot and the crash. With `save 3600 1` that's up to 1h of cache + stream-consumer-offset advancement.

---

## Drill mode (--to-fresh-volume)

Non-destructive. Use this whenever you want to verify the off-site snapshot.

```bash
cd /home/debian/Projects/eusolicit/eusolicit-app
bash scripts/onprem/redis-restore.sh --to-fresh-volume
```

Spins up `eusolicit-redis-restore-temp` on port `127.0.0.1:26379`. Smoke check:

```bash
redis-cli -p 26379 ping
redis-cli -p 26379 INFO keyspace
redis-cli -p 26379 XINFO STREAMS eusolicit.events   # if streams exist
```

Tear down:
```bash
docker rm -f eusolicit-redis-restore-temp
docker volume rm eusolicit-redis-restore-temp-data
```

---

## Prod restore (--to-prod)

> Destructive. Wipes `eusolicit-app_redisdata` volume.

```bash
bash scripts/onprem/redis-restore.sh --to-prod
# type 'I-UNDERSTAND' to proceed
```

Steps:
1. Confirmation prompt.
2. `docker compose stop redis`.
3. Wipe + recreate `eusolicit-app_redisdata` volume.
4. Copy the restored `dump.rdb` into the new volume.
5. `docker compose up -d` to start the stack with the new volume.

Post-restore validation:
```bash
docker exec eusolicit-app-redis-1 redis-cli ping
docker exec eusolicit-app-redis-1 redis-cli DBSIZE
docker exec eusolicit-app-redis-1 redis-cli XINFO STREAMS <stream-name>  # for each event-bus stream

# Verify Celery beat tasks resume (look at celery-worker logs)
docker logs --tail=50 eusolicit-app-notification-worker-1
docker logs --tail=50 eusolicit-app-data-pipeline-worker-1
```

---

## Controlled-restart drill (Story onprem-02 AC 4 — first-drill)

Run this drill at a low-traffic window to verify `restart: unless-stopped` + `pool_pre_ping=True` + redis-py resilience hardening (`socket_keepalive`, `health_check_interval`) all work as intended.

```bash
# 1. Capture a sentinel key BEFORE the drill
docker exec eusolicit-app-redis-1 redis-cli SET drill-sentinel "before-$(date +%s)"

# 2. Note in-flight celery / stream state
docker exec eusolicit-app-redis-1 redis-cli XINFO STREAMS eusolicit.events 2>/dev/null || echo "(no streams)"

# 3. Restart redis (≤30s window)
T0=$(date +%s)
docker compose -f /home/debian/Projects/eusolicit/eusolicit-app/docker-compose.prod.yml restart redis

# 4. Watch reconnect on the 6 services
for svc in client-api admin-api data-pipeline agenticsai-gateway notification integrations-api; do
  docker logs --since=30s --tail=5 "eusolicit-app-${svc}-1" 2>&1 | grep -iE "redis|reconnect" || true
done

# 5. Verify sentinel survived
docker exec eusolicit-app-redis-1 redis-cli GET drill-sentinel

# 6. Time-to-recovery
T1=$(date +%s)
echo "Total redis-unavailable window: $((T1 - T0))s (target ≤30s)"
```

Record results in §First Drill Results below.

### First Drill Results

| Date | Redis-unavailable window | Sentinel intact | Streams resumed | Notes |
|---|---|---|---|---|
| TBD | — | — | — | (To be filled by operator on first drill.) |

---

## References

- Story: `eusolicit-docs/implementation-artifacts/onprem-02-redis-persistence-and-recovery.md`
- Backup script: `eusolicit-app/scripts/onprem/redis-backup.sh`
- Restore script: `eusolicit-app/scripts/onprem/redis-restore.sh`
- Compose config: `docker-compose.prod.yml` (redis service `command:` block)
- ADR-003: Redis Streams as primary event bus
- ADR-010 (2026-05-11): On-prem pivot