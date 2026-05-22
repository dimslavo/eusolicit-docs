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

## Drill 4 — Redis AOF replay (≤ 10 min) — closes pe-04 AC2

Verifies Redis Streams consumer-group offsets + AOF persistence + redis-py / Celery keepalive reconnect (per E21 hardening).

```bash
# 1. Sentinel + baseline offsets (run BEFORE kill)
docker exec eusolicit-app-redis-1 redis-cli SET chaos-sentinel "drill-$(date +%s)"

# Capture baseline consumer-group offsets across all known streams.
# These offsets MUST survive AOF replay — that's the load-bearing invariant.
mkdir -p /tmp/chaos-redis-pre
for STREAM in eusolicit.events notification.dispatch pipeline.opportunities.ingested \
              gateway.workflow_runs_converge subscription.changed kb.file.uploaded ; do
  docker exec eusolicit-app-redis-1 redis-cli XINFO GROUPS "$STREAM" \
    > /tmp/chaos-redis-pre/${STREAM//./_}.txt 2>/dev/null || echo "stream $STREAM not present yet"
done

# 2. Hard kill
T0=$(date +%s)
docker kill -s KILL eusolicit-app-redis-1

# 3. Wait for restart + AOF replay. The container restart policy brings it back;
#    Redis logs should show "Loading data from AOF" then "DB loaded from append only file".
for i in $(seq 1 30); do
  if docker exec eusolicit-app-redis-1 redis-cli ping 2>/dev/null | grep -q PONG; then
    T1=$(date +%s)
    echo "Redis back in $((T1 - T0))s"
    break
  fi
  sleep 1
done

# Verify AOF actually replayed (log evidence)
docker logs --tail=30 eusolicit-app-redis-1 | grep -iE "(loading|aof|append only)" \
  || echo "WARNING: no AOF-loading log line found — was AOF really enabled?"

# 4. Sentinel survived?
docker exec eusolicit-app-redis-1 redis-cli GET chaos-sentinel \
  | grep -q "drill-" && echo "✅ AOF replayed sentinel intact" \
  || echo "❌ AOF failed — sentinel lost"

# 5. Consumer-group offset comparison (pre vs post)
mkdir -p /tmp/chaos-redis-post
for STREAM in eusolicit.events notification.dispatch pipeline.opportunities.ingested \
              gateway.workflow_runs_converge subscription.changed kb.file.uploaded ; do
  docker exec eusolicit-app-redis-1 redis-cli XINFO GROUPS "$STREAM" \
    > /tmp/chaos-redis-post/${STREAM//./_}.txt 2>/dev/null || true
done

for f in /tmp/chaos-redis-pre/*.txt; do
  base=$(basename "$f")
  if diff -q "$f" "/tmp/chaos-redis-post/$base" >/dev/null 2>&1; then
    echo "✅ offsets preserved: $base"
  else
    echo "⚠️  offset diff in $base — INVESTIGATE"
    diff "$f" "/tmp/chaos-redis-post/$base" || true
  fi
done

# 6. Service reconnect — all 6 services /healthz must return 200
#    (redis-py keepalive + Celery broker_connection_retry_on_startup should make this automatic)
for port in 18001 18002 18003 18004 18005 18007; do
  curl -sf http://127.0.0.1:$port/healthz && echo " port $port OK" || echo " port $port FAIL"
done

# 7. Celery worker reconnect — pick a known queue and check it's draining
docker exec eusolicit-app-data-pipeline-1 celery -A data_pipeline.celery_app inspect ping 2>/dev/null \
  || echo "WARNING: celery inspect ping failed — investigate worker reconnect"
```

**Pass criterion (all must hold for AC2 close):**
1. `T1 - T0 ≤ 30s` — redis-unavailable window
2. AOF replay log line observed
3. Sentinel value intact post-restart (AOF persistence working)
4. **Zero offset drift** in any consumer-group XINFO diff (pre vs post)
5. All 6 services /healthz 200
6. Celery worker inspect ping succeeds

If any check fails, file a post-mortem at `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-redis.md` BEFORE re-drilling.

---

## Drill 5 — Observability container restart-loop (≤ 10 min) — closes pe-04 AC3

Verifies the `ContainerRestartLoop` PromQL alert fires when observability containers (prometheus + cadvisor + grafana) themselves cycle. Failure here is an observability blind spot — without it, *every other alert is downstream-untrustworthy*.

```bash
# Goal: ≥3 restarts within 5min per ContainerRestartLoop rule at
#       infra/observability/prometheus/rules/host-alerts.yaml:81-91

for CONTAINER in eusolicit-app-prometheus-1 eusolicit-app-cadvisor-1 eusolicit-app-grafana-1 ; do
  echo "=== Drilling $CONTAINER ==="
  for i in 1 2 3 4; do
    docker kill "$CONTAINER" 2>/dev/null
    sleep 20   # let restart policy bring it back
    docker inspect --format='{{.State.Status}}' "$CONTAINER"
  done
done

# Wait for alert to fire (rule has a `for: 2m` clause typically)
echo "Waiting 4 minutes for ContainerRestartLoop alert..."
sleep 240

# Verify alert is in firing state
curl -s http://127.0.0.1:19090/api/v1/alerts | \
  jq '.data.alerts[] | select(.labels.alertname == "ContainerRestartLoop")'

# Verify Alertmanager routed it (per pe-06 closed 2026-05-13):
#   severity=page → page-email-and-telegram receiver
# Operator confirms delivery in:
#   - Email inbox (ONCALL_EMAIL env on www1)
#   - Telegram bot chat
# Capture screenshots / log extracts for the post-mortem.

# Recovery check: once you stop killing, restart-loop subsides
echo "Letting containers stabilise. Wait 3 minutes."
sleep 180
docker ps --filter "name=eusolicit-app-prometheus-1" \
          --filter "name=eusolicit-app-cadvisor-1" \
          --filter "name=eusolicit-app-grafana-1" \
          --format "table {{.Names}}\t{{.Status}}"

# Alert should resolve (send_resolved: true expected)
curl -s http://127.0.0.1:19090/api/v1/alerts | \
  jq '.data.alerts[] | select(.labels.alertname == "ContainerRestartLoop") | .state'
```

**Pass criterion (all must hold for AC3 close):**
1. `ContainerRestartLoop` alert fires within 5min of the third kill
2. Severity routes to `page-email-and-telegram` per pe-06 alertmanager.yaml
3. Operator receives email AND Telegram messages (capture both in post-mortem)
4. After kills stop, `restart: unless-stopped` brings containers back without manual `docker compose up`
5. Alert auto-resolves once restart-loop subsides (`send_resolved: true` honoured)

Variant of Drill 1 — same `docker kill` mechanic but targeted at observability containers specifically; pass criterion is the alert path, not just container recovery.

---

## Drill 6 — Network partition vs AgenticSAI (≤ 15 min) — closes pe-04 AC4

Verifies the AgenticSAI circuit-breaker + tenant-visible degraded-mode banner (E04 amendment S04.27 / E28 S28.07).

> **Dependency:** AC4 verification of the tenant-visible banner requires E28 S28.07 to have landed. If running this drill BEFORE E28 lands, AC4 closure is conditional: circuit-breaker open + degraded-mode log evidence alone satisfies the minimum bar; banner verification is deferred until E28 ships.

```bash
# 1. Capture baseline: circuit-breaker state should be 'closed' / healthy
curl -s http://127.0.0.1:18004/metrics | grep -E "agenticsai_circuit_breaker_state|agenticsai_outbound_requests_total"

# 2. Resolve AgenticSAI host to IPs (both A and AAAA if dual-stack)
HOST=agenticsai.endigitalx.com
AGENTICSAI_IPV4=$(dig +short A "$HOST")
AGENTICSAI_IPV6=$(dig +short AAAA "$HOST")
echo "Will block egress to: ${AGENTICSAI_IPV4} ${AGENTICSAI_IPV6}"

# 3. Apply egress block (requires sudo on www1)
T0=$(date +%s)
for IP in $AGENTICSAI_IPV4; do
  sudo iptables -I OUTPUT -d "$IP" -j REJECT --reject-with icmp-net-unreachable
done
for IP in $AGENTICSAI_IPV6; do
  sudo ip6tables -I OUTPUT -d "$IP" -j REJECT --reject-with icmp6-no-route 2>/dev/null || true
done

# 4. Force some outbound traffic to AgenticSAI to trigger circuit-breaker
#    Trigger via an admin-API endpoint that calls agenticsai-gateway (or wait for next scheduled run).
#    Watch the breaker state transition:
for i in $(seq 1 60); do
  STATE=$(curl -s http://127.0.0.1:18004/metrics | \
          awk '/^agenticsai_circuit_breaker_state/ {print $2}')
  echo "$(date +%H:%M:%S) breaker_state=${STATE}"
  if [ "${STATE%.*}" = "1" ] || [ "${STATE%.*}" = "2" ]; then  # 1=open, 2=half-open
    T1=$(date +%s)
    echo "Circuit opened after $((T1 - T0))s"
    break
  fi
  sleep 5
done

# 5. Verify degraded-mode event published to Redis Streams
docker exec eusolicit-app-redis-1 redis-cli XREVRANGE notification.degraded_mode_events + - COUNT 5

# 6. (Post-E28 S28.07) Verify tenant-visible banner
#    Hit the system-status endpoint that the frontend layout polls:
curl -s http://127.0.0.1:18001/api/v1/system/status | jq '.degraded_mode'
# Expected: { "degraded_mode": true, "since": "...", "feature": "agenticsai_ai_analysis" }

# (Manual) Open client app in a browser; verify banner renders within 1min.

# 7. Restore connectivity
for IP in $AGENTICSAI_IPV4; do
  sudo iptables -D OUTPUT -d "$IP" -j REJECT --reject-with icmp-net-unreachable 2>/dev/null || true
done
for IP in $AGENTICSAI_IPV6; do
  sudo ip6tables -D OUTPUT -d "$IP" -j REJECT --reject-with icmp6-no-route 2>/dev/null || true
done
T_RECOVER=$(date +%s)

# 8. Watch breaker re-close + banner clear (1-min hysteresis per S28.07)
for i in $(seq 1 30); do
  STATE=$(curl -s http://127.0.0.1:18004/metrics | \
          awk '/^agenticsai_circuit_breaker_state/ {print $2}')
  echo "$(date +%H:%M:%S) breaker_state=${STATE}"
  if [ "${STATE%.*}" = "0" ]; then  # 0=closed
    T_HEAL=$(date +%s)
    echo "Circuit healed in $((T_HEAL - T_RECOVER))s"
    break
  fi
  sleep 10
done
```

**Pass criterion (all must hold for AC4 close):**
1. Circuit-breaker transitions `closed → open` within ≤ 5min of partition
2. `platform.degraded_mode` (or equivalent) event lands on Redis Streams
3. (Post-E28 S28.07) Tenant-visible banner renders within 1min of degraded-mode event
4. On partition removal, circuit-breaker recovers and banner clears within 2min hysteresis
5. No silent failures — every AgenticSAI call attempt during the partition logs a structured error referencing the circuit state

If E28 has NOT landed: AC4 closes conditionally on points 1, 2, 4, 5; point 3 deferred and re-drilled post-E28.

---

### Drill ordering recommendation

Run drills in this order within a single drill window (or across multiple if needed):
1. **Drill 1** (container kill, client-api) — warm-up, low risk
2. **Drill 2** (disk fill) — easy cleanup
3. **Drill 3** (postgres crash) — stateful but recoverable
4. **Drill 4** (redis AOF) — stateful; offset-preservation matters
5. **Drill 5** (observability restart-loop) — depends on Drills 1-4 being clean (otherwise alerts will be drowned)
6. **Drill 6** (AgenticSAI partition) — most disruptive to user-facing AI features; do last


---

## §Drill Results

| Date | Drill | Outcome | Recovery time | Issues found |
|---|---|---|---|---|
| TBD | Drill 1 (container kill) | — | — | — |
| TBD | Drill 2 (disk fill) | — | — | — |
| TBD | Drill 3 (postgres crash) | — | — | — |
| TBD | Drill 4 (redis AOF) | — | — | — |
| TBD | Drill 5 (observability restart-loop) | — | — | — |
| TBD | Drill 6 (AgenticSAI partition) | — | — | — |

After each drill, append a row. After all six pass — covering pe-04 AC1-AC4 — the rescoped `pe-04-chaos-drill-execution` entry in sprint-status can flip to `done`.

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
