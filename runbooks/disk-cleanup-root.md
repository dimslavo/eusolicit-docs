# Runbook: Root Partition Cleanup

**Triggers:** `HostRootDiskWarning` (>85%) | `HostRootDiskCritical` (>90%)
**Story:** onprem-04 | **SLO:** platform

## Symptoms

`/` partition near full. Docker writes will fail, journald log rotation may pause, package upgrades will fail.

## Triage (5 min)

```bash
# What's eating space?
sudo du -shx /var/lib/docker /var/log /var /usr 2>/dev/null | sort -h
sudo du -shx /var/lib/docker/* 2>/dev/null | sort -h

# Docker image / container / volume / build cache footprint
docker system df
```

## Fixes (in order of safety)

```bash
# 1. Prune dangling Docker images (always safe)
docker image prune -f

# 2. Prune stopped containers
docker container prune -f

# 3. Prune build cache (rebuilds will be slower next deploy; acceptable)
docker builder prune -f

# 4. Truncate large journald logs (caps to last 200MB)
sudo journalctl --vacuum-size=200M

# 5. Rotate /var/log/syslog and others
sudo logrotate -f /etc/logrotate.conf
```

## Long-term fix (Story onprem-06)

Move Docker storage root to `/home/docker` (319 GB partition, plenty of room). Edit `/etc/docker/daemon.json`:

```json
{ "data-root": "/home/docker" }
```

Then: `sudo systemctl stop docker; sudo rsync -a /var/lib/docker/ /home/docker/; sudo systemctl start docker`.

This is captured in `onprem-06-www1-itself-as-code` Task 5. Schedule a maintenance window.

## References

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
- onprem-06: `eusolicit-docs/implementation-artifacts/onprem-06-www1-itself-as-code.md`
