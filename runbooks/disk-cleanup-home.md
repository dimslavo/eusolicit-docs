# Runbook: /home Partition Cleanup

**Trigger:** `HostHomeDiskWarning` (>90%)
**Story:** onprem-04 | **SLO:** platform
**SLA-Scope**: in-scope

## Symptoms

`/home` partition near full. Backups will fail to land, WAL archiving will stop accepting new segments, container volumes may halt.

## Triage

```bash
# Top space consumers under /home
sudo du -shx /home/* 2>/dev/null | sort -h | tail -10
sudo du -shx /home/docker/backups/* 2>/dev/null | sort -h | tail -10
sudo du -shx /home/debian/eusolicit-overrides/pg_wal_archive 2>/dev/null
```

## Resolution

Apply in order:

```bash
# 1. Backup directory bloat — purge anything older than the local 7d retention
find /home/docker/backups/eusolicit -maxdepth 1 -mindepth 1 -type d -mtime +7 -exec rm -rf {} \;

# 2. WAL archive bloat — the postgres-backup.sh --wal-only cron should be
#    pushing these to Hetzner and pruning. If WAL is piling up, check that
#    cron is actually running:
grep eusolicit /var/log/syslog | tail -20

# 3. Stale restore-test temp volumes
docker volume ls --filter "name=eusolicit-restore" --format "{{.Name}}"
docker volume rm $(docker volume ls -q --filter "name=eusolicit-restore-temp") 2>/dev/null || true

# 4. Co-tenant backup growth (lifematch-*, celthrac, etc.) — coordinate with
#    the relevant team; we share /home/docker/backups/ as a convention.
```

### Long-term

If sustained pressure: expand the storage volume on www1 (provider UI) or move postgres data volume to a dedicated mount.

## Verification

```bash
# /home back under the 90% alert threshold
df -h /home
```

`HostHomeDiskWarning` should clear within one scrape interval once usage drops below 90%. Confirm the next backup lands and WAL archiving resumes (see `wal-archiving-stalled.md` §Verification).

## Rollback

The Resolution steps (deleting aged backups, pruning stale restore-test volumes) are **forward-only and irreversible** — there is nothing to undo, and nothing within the local 7-day retention or any in-use volume is touched. If an aged backup that was just purged is later needed, restore it from the off-site Hetzner copy per `postgres-restore.md`. If co-tenant data was deleted in error (step 4), escalate to that tenant's on-call immediately.

## Related

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
- Backup script: `eusolicit-app/scripts/onprem/postgres-backup.sh`
- `wal-archiving-stalled.md`, `postgres-restore.md`
