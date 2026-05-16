# Runbook: Postgres WAL Archiving Stalled

**Trigger:** `PostgresWalArchivingStalled` (no archived count increase in 15 min + last archive >15 min ago)
**Story:** onprem-04 | **SLO:** platform
**SLA-Scope**: in-scope

## Symptoms

Postgres `archive_command` not running successfully. WAL segments accumulating in `/wal_archive` on www1; off-site RPO commitment is breached (no new WAL pushed to Hetzner).

## Triage

```bash
# Inside container: is archive_mode actually on?
docker exec eusolicit-app-postgres-1 psql -U eusolicit -d eusolicit -c "SHOW archive_mode; SHOW archive_command; SHOW archive_timeout;"

# Archive stats from postgres
docker exec eusolicit-app-postgres-1 psql -U eusolicit -d eusolicit -c "
  SELECT archived_count, last_archived_wal, last_archived_time,
         failed_count, last_failed_wal, last_failed_time, stats_reset
  FROM pg_stat_archiver;"

# What's the host-mounted /wal_archive looking like?
ls -lah /home/debian/eusolicit-overrides/pg_wal_archive | head -20
df -h /home

# Is the WAL-push cron running?
grep wal-only /var/log/eusolicit/cron.log | tail -10
```

## Resolution

Identify the cause and apply the matching fix:

1. **`/wal_archive` is full or unwritable.** Free space:
   ```bash
   # Last-resort: WAL segments older than the most recent basebackup can be dropped
   # but ONLY if basebackup is recent and verified.
   ls -t /home/debian/eusolicit-overrides/pg_wal_archive | tail -20
   # Verify before deletion.
   ```

2. **archive_command failing.** Check postgres container logs:
   ```bash
   docker logs eusolicit-app-postgres-1 2>&1 | grep -iE "archive|wal" | tail -30
   ```
   Look for "archive command failed with exit code…" — the message points at the failing step (cp permissions? disk full? path missing?).

3. **WAL-push cron broken** — check `infra/host/cron.d/eusolicit-backup` is installed (`ls /etc/cron.d/eusolicit-backup`). If installed but not running, check `restic` is installed and `.env.backup` is reachable.

4. **Hetzner reachability** — the WAL-push step talks to Hetzner Storage Box. If Hetzner is down, push fails but local archiving continues. Manual:
   ```bash
   ssh u123456@u123456.your-storagebox.de "echo ok"
   ```

After the underlying cause is fixed, run a one-shot push to catch up:

```bash
bash /home/debian/Projects/eusolicit/eusolicit-app/scripts/onprem/postgres-backup.sh --wal-only
```

## Verification

```bash
# archived_count increasing, last_archived_time recent, failed_count not growing
docker exec eusolicit-app-postgres-1 psql -U eusolicit -d eusolicit -c "
  SELECT archived_count, last_archived_time, failed_count, last_failed_time
  FROM pg_stat_archiver;"

# Off-site copy caught up (Hetzner Storage Box listing is recent)
ls -lah /home/debian/eusolicit-overrides/pg_wal_archive | tail -5
```

`PostgresWalArchivingStalled` clears once `archived_count` advances and `last_archived_time` is within the last 15 min.

## Rollback

The catch-up push (`postgres-backup.sh --wal-only`) is **idempotent and forward-only** — re-running it is safe and there is nothing to undo. The one irreversible action is manual deletion of old WAL segments (Resolution #1); only do it after verifying a recent basebackup exists. If WAL needed for point-in-time recovery was deleted in error, the off-site Hetzner copy is authoritative — recover per `postgres-restore.md`. No change here alters running Postgres.

## Related

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
- Backup script: `eusolicit-app/scripts/onprem/postgres-backup.sh`
- onprem-01 story for full backup architecture
- `postgres-restore.md`, `disk-cleanup-home.md`
