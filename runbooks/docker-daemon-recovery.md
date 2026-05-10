# Runbook: Docker Daemon Down / Recovery

**Trigger:** `DockerDaemonDown` (cAdvisor scrape failing → up{job="docker-daemon"}=0)
**Story:** onprem-04 | **SLO:** platform

## Symptoms

Docker daemon unreachable. ALL services on www1 are inaccessible (this is the substrate). Full prod outage.

## Triage (immediate)

```bash
# Is dockerd actually down?
sudo systemctl status docker

# Recent docker journal
sudo journalctl -u docker --since "30 min ago" --no-pager | tail -50

# Disk full? (common dockerd-kill cause)
df -h /var/lib/docker /
```

## Fixes (in order, with restart progression)

```bash
# 1. Try a graceful restart
sudo systemctl restart docker

# 2. Wait 30s, check
sleep 30
sudo systemctl status docker
docker ps 2>&1 | head -3

# 3. If still down, check for corrupted state
sudo journalctl -u docker --since "5 min ago" --no-pager

# 4. Common fix: free space on /var/lib/docker (see runbooks/disk-cleanup-root.md)
sudo du -sh /var/lib/docker/overlay2/* 2>/dev/null | sort -h | tail -5

# 5. Nuclear: stop, clean, restart (LAST RESORT — destroys running container state)
sudo systemctl stop docker
sudo systemctl start docker
```

## After daemon is back

Bring up EU Solicit stack:
```bash
cd /home/debian/Projects/eusolicit/eusolicit-app
bash scripts/deploy.sh --no-build   # uses existing images
```

Verify:
```bash
docker ps --filter "name=eusolicit-app-" --format "table {{.Names}}\t{{.Status}}"
for port in 18001 18002 18003 18004 18005 18007; do
  curl -sf http://127.0.0.1:$port/healthz && echo " port $port OK" || echo " port $port FAIL"
done
```

## If Docker can't be recovered

This is a www1-level disaster. Trigger cross-host recovery:
- Provision a fresh Debian 13 host per `runbooks/www1-rebuild.md` (lands with onprem-06).
- Restore postgres + redis from off-site backups per `runbooks/postgres-restore.md` + `runbooks/redis-restore.md`.
- Point DNS at the new host.

## References

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
- runbooks/disk-cleanup-root.md, runbooks/www1-rebuild.md (onprem-06)
