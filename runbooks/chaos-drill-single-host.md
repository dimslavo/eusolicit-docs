# Chaos Drill Runbook — Single-Host Docker (Story pe-04 rescoped)

**Audience:** platform-engineering operator running a scheduled drill
**Story:** rescoped `pe-04-chaos-drill-execution` in sprint-status post-pivot
**Frequency:** quarterly minimum; recommended monthly during the launch quarter
**Coordination:** schedule at a low-traffic window; announce in `#platform-alerts`

## Why drill at all

The on-prem posture trades horizontal HA for operational simplicity. The compensating control is rehearsed recovery: every failure mode the cluster doesn't fix automatically, we must rehearse manually so the actual incident isn't the first time we run the procedure.

Four drills, runnable independently. Aim to run all four within one quarter.

---

## Drill 1 — Container kill + auto-restart (≤ 5 min)

Verifies `restart: unless-stopped` + healthchecks + service-level reconnect (`pool_pre_ping`, redis-py keepalive).

```bash
SVC=client-api
T0=$(date +%s)

# Kill the container ungracefully
docker kill eusolicit-app-${SVC}-1

# Watch auto-restart
for i in $(seq 1 30); do
  STATE=$(docker inspect --format='{{.State.Status}}' eusolicit-app-${SVC}-1)
  HEALTH=$(docker inspect --format='{{.State.Health.Status}}' eusolicit-app-${SVC}-1 2>/dev/null)
  echo "$(date +%H:%M:%S) state=${STATE} health=${HEALTH}"
  if [ "${STATE}" = "running" ] && [ "${HEALTH}" = "healthy" ]; then
    T1=$(date +%s)
    echo "Recovery time: $((T1 - T0))s"
    break
  fi
  sleep 2
done

# Verify external reach
curl -sf http://127.0.0.1:18001/healthz
```

**Pass criterion:** auto-restart + healthy within 60s, no manual intervention.

Repeat for each of the 6 services. Document one-line note per service in §Drill Results.

---

## Drill 2 — Disk fill (≤ 15 min)

Verifies `HostRootDiskWarning` / `HostRootDiskCritical` alerts fire AND that fill protection doesn't break services.

```bash
# 1. Capture baseline
df -h / | tail -1

# 2. Fill / temporarily — write a sparse file targeting 90% of remaining free
AVAIL_KB=$(df / | tail -1 | awk '{print $4}')
TARGET_KB=$((AVAIL_KB * 9 / 10))  # 90% of free space
sudo fallocate -l ${TARGET_KB}K /tmp/chaos-disk-fill.bin

# 3. Verify alert fires (Prometheus + Alertmanager)
sleep 60
curl -s http://127.0.0.1:19090/api/v1/alerts | jq '.data.alerts[] | select(.labels.alertname | startswith("HostRootDisk"))'

# 4. Verify services still healthy under disk pressure
for port in 18001 18002 18003 18004 18005 18007; do
  curl -sf http://127.0.0.1:$port/healthz && echo " port $port OK" || echo " port $port FAIL"
done

# 5. Clean up — IMPORTANT
sudo rm /tmp/chaos-disk-fill.bin
df -h / | tail -1
```

**Pass criterion:** alert fires within 5min of fill, no service degradation observed, cleanup restores free space.

---

## Drill 3 — Postgres crash recovery (≤ 10 min)

Verifies pg WAL replay + reconnect via `pool_pre_ping`.

```bash
# 1. Capture sentinel — write a row before crash
docker exec eusolicit-app-postgres-1 psql -U eusolicit -d eusolicit -c \
  "INSERT INTO shared.audit_log (entity_type, entity_id, action_type, before, after, user_id, ip_address)
   VALUES ('chaos-drill', gen_random_uuid(), 'chaos-sentinel-$(date +%s)', null, '{}', null, '127.0.0.1');"

# 2. Crash postgres
T0=$(date +%s)
docker kill -s KILL eusolicit-app-postgres-1

# 3. Watch auto-restart + WAL replay
docker logs --tail=20 -f eusolicit-app-postgres-1 &
LOGPID=$!
sleep 30
kill $LOGPID

# 4. Service reconnect — verify all 5 services' /healthz return 200
for port in 18001 18002 18003 18004 18005 18007; do
  curl -sf http://127.0.0.1:$port/healthz && echo " port $port OK"
done

# 5. Verify sentinel survived
docker exec eusolicit-app-postgres-1 psql -U eusolicit -d eusolicit -c \
  "SELECT action_type FROM shared.audit_log WHERE action_type LIKE 'chaos-sentinel-%' ORDER BY timestamp DESC LIMIT 1;"

T1=$(date +%s)
echo "Total postgres-unavailable window: $((T1 - T0))s"
```

**Pass criterion:** postgres back to healthy within 60s, WAL replay clean, sentinel intact, all 6 services reconnect without manual intervention.

---

## Drill 4 — Redis AOF replay (≤ 10 min)

Verifies Redis Streams consumer groups + Celery beat survive a hard kill.

```bash
# 1. Sentinel
docker exec eusolicit-app-redis-1 redis-cli SET chaos-sentinel "drill-$(date +%s)"
PRE_STREAMS=$(docker exec eusolicit-app-redis-1 redis-cli XINFO STREAMS eusolicit.events 2>/dev/null || echo none)

# 2. Hard kill
T0=$(date +%s)
docker kill -s KILL eusolicit-app-redis-1

# 3. Wait for restart + AOF replay
sleep 15
docker exec eusolicit-app-redis-1 redis-cli ping

# 4. Sentinel survived?
docker exec eusolicit-app-redis-1 redis-cli GET chaos-sentinel

# 5. Consumer groups
docker exec eusolicit-app-redis-1 redis-cli XINFO STREAMS eusolicit.events 2>/dev/null

# 6. Service reconnect
for port in 18001 18002 18003 18004 18005 18007; do
  curl -sf http://127.0.0.1:$port/healthz
done

T1=$(date +%s)
echo "Total redis-unavailable window: $((T1 - T0))s"
```

**Pass criterion:** redis-unavailable window ≤30s, sentinel intact (AOF working), consumer-group offsets preserved, all services reconnect.

---

## §Drill Results

| Date | Drill | Outcome | Recovery time | Issues found |
|---|---|---|---|---|
| TBD | Drill 1 | — | — | — |
| TBD | Drill 2 | — | — | — |
| TBD | Drill 3 | — | — | — |
| TBD | Drill 4 | — | — | — |

After each drill, append a row. After all four pass, the rescoped `pe-04-chaos-drill-execution` entry in sprint-status can flip to `done`.

## After a failed drill

Open a post-mortem in `eusolicit-docs/post-mortems/` documenting:
- which drill failed
- root cause (config? code? dependency? human?)
- fix
- re-drill date

## References

- Story: rescoped `pe-04-chaos-drill-execution` in sprint-status
- Project memory: "Auto-sync ships unverified code"
- ADR-010 (2026-05-11)
- Host alerts: `infra/observability/prometheus/rules/host-alerts.yaml`
- Original PE.04 runbook (k8s; superseded): `eusolicit-docs/implementation-artifacts/.archive/pe-04-chaos-drill-runbook.md`
