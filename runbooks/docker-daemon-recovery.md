# Runbook: Docker Daemon Down / Recovery

**Trigger:** `DockerDaemonDown` (cAdvisor scrape failing → up{job="docker-daemon"}=0)
**Story:** onprem-04 | **SLO:** platform
**SLA-Scope**: in-scope

## Symptoms

Docker daemon unreachable. ALL services on www1 are inaccessible (this is the substrate). Full prod outage.

## Triage

Immediate:

```bash
# Is dockerd actually down?
sudo systemctl status docker

# Recent docker journal
sudo journalctl -u docker --since "30 min ago" --no-pager | tail -50

# Disk full? (common dockerd-kill cause)
df -h /var/lib/docker /
```

## Resolution

Apply in order, with restart progression:

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

Once the daemon is back, bring up the EU Solicit stack:

```bash
cd /home/debian/Projects/eusolicit/eusolicit-app
bash scripts/deploy.sh --no-build   # uses existing images
```

## Verification

```bash
# Daemon active
sudo systemctl is-active docker

# EU Solicit stack healthy
docker ps --filter "name=eusolicit-app-" --format "table {{.Names}}\t{{.Status}}"
for port in 18001 18002 18003 18004 18005 18007; do
  curl -sf http://127.0.0.1:$port/healthz && echo " port $port OK" || echo " port $port FAIL"
done
```

`DockerDaemonDown` clears once cAdvisor can scrape the daemon again (`up{job="docker-daemon"}=1`).

## Rollback

Daemon recovery is **forward-only** — there is no prior state to restore; the goal *is* to get dockerd running. The nuclear step (#5) intentionally destroys running container state; containers are recreated by `scripts/deploy.sh --no-build` from existing images. If the daemon went down because of a `/etc/docker/daemon.json` change (e.g. the disk-cleanup-root `data-root` migration), revert that file to its previous contents and `sudo systemctl restart docker`. If Docker cannot be recovered at all, this is a www1-level disaster:

- Provision a fresh Debian 13 host per `runbooks/www1-rebuild.md` (lands with onprem-06).
- Restore postgres + redis from off-site backups per `runbooks/postgres-restore.md` + `runbooks/redis-restore.md`.
- Point DNS at the new host.

## Related

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
- `runbooks/disk-cleanup-root.md`, `runbooks/www1-rebuild.md` (onprem-06)
- `runbooks/postgres-restore.md`, `runbooks/redis-restore.md`
