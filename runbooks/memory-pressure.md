# Runbook: Memory Pressure / Swap Use

**Triggers:** `HostMemoryPressure` (available <2 GiB) | `HostSwapHigh` (swap >32 GiB)
**Story:** onprem-04 | **SLO:** platform

## Symptoms

Available memory low, swap rising. OOM-kills imminent. Latency may already be degraded as the kernel pages out hot working sets.

## Triage

```bash
# Per-process memory (top 10)
ps aux --sort=-rss | head -11

# Per-container memory + swap
docker stats --no-stream --format "table {{.Name}}\t{{.MemUsage}}\t{{.MemPerc}}"

# Cgroup OOM kills (recent)
sudo journalctl -k --since "1 hour ago" | grep -iE "oom-kill|out of memory"

# Swap accumulation per container
for id in $(docker ps -q); do
  name=$(docker inspect --format '{{.Name}}' "$id" | sed 's|/||')
  swap=$(awk 'NR==1{print $2}' "/sys/fs/cgroup/system.slice/docker-${id}.scope/memory.swap.current" 2>/dev/null || echo 0)
  echo "$name: $swap bytes swap"
done | sort -k2 -nr | head -5
```

## Fixes

```bash
# 1. Find the leaker. If it's a single eusolicit container leaking, restart it:
docker compose -f /home/debian/Projects/eusolicit/eusolicit-app/docker-compose.prod.yml \
  restart <service-name>

# 2. If many containers are bloated → host-level pressure (likely co-tenant
#    or a slow accumulation). Plan a controlled restart cycle in a maintenance
#    window.

# 3. Emergency: kill the largest non-critical co-tenant process via the
#    appropriate tenant's runbook. Do NOT kill -9 eusolicit containers — let
#    docker compose restart them so volumes flush cleanly.
```

## Long-term

- Cap individual containers via `deploy.resources.limits.memory` (already partially done in `docker-compose.prod.yml`).
- Consider adding a swap-pressure metric to per-service Grafana dashboards.

## References

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
