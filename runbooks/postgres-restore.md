# Postgres Restore Runbook (Story onprem-01)

**Audience:** on-call engineer in an incident
**Story:** `eusolicit-docs/implementation-artifacts/onprem-01-postgres-backup-and-recovery.md`
**RTO target:** ≤ 4h | **RPO target:** ≤ 24h (off-site daily) or ≤ 15 min (off-site WAL push if cron healthy)

---

## When to use which mode

```
DECISION TREE — pick the first applicable branch:

Q1: Is the prod postgres container running and reachable?
  YES → No restore needed. This runbook does not apply. Check `runbooks/full-disk-on-pg.md` or `runbooks/error-budget-burn.md`.
  NO  → continue.

Q2: Is the pgdata docker volume present and intact?
  YES → run `docker compose -f docker-compose.prod.yml start postgres` and re-evaluate.
        If postgres still fails to start, the volume is corrupted → continue to Q3.
  NO  → continue to Q3.

Q3: Is www1 itself still reachable (SSH works, docker daemon responds)?
  YES → §Same-host restore (most common — restore into pgdata on www1)
  NO  → §Cross-host restore (the whole machine is gone — full rebuild)
```

The story's RTO commitment of ≤ 4h presumes the **same-host** path. The cross-host path also depends on `onprem-06` (www1-as-code) being ready; without that, RTO drifts to "however long the engineer takes to remember everything."

---

## Prerequisites

Before running any restore:

1. **`restic` is installed on the restore host.** If not:
   ```bash
   sudo apt-get update && sudo apt-get install -y restic
   ```
2. **`/home/debian/eusolicit-overrides/.env.backup` exists and is populated** with real Hetzner Storage Box credentials (see `infra/host/eusolicit-overrides/.env.backup.example`).
3. **The Hetzner SSH key is present** at the path implied by the SFTP `RESTIC_REPOSITORY` URL (typically `/home/debian/.ssh/id_ed25519`). Verify with `ssh u123456@u123456.your-storagebox.de` — should connect without prompting for a password.
4. **A short list of recent restic snapshots** so you know what's available:
   ```bash
   source /home/debian/eusolicit-overrides/.env.backup
   restic snapshots --tag basebackup --compact
   ```
   Pick the snapshot ID to restore from (default: `latest`).

---

## Drill / dry-run path — `--to-fresh-volume`

Use this every Sunday (cron-driven, AC 5) and before any real prod restore to confirm the snapshot is recoverable. **Non-destructive** — prod is untouched.

```bash
cd /home/debian/Projects/eusolicit/eusolicit-app
bash scripts/onprem/postgres-restore.sh --to-fresh-volume
# optional: --snapshot <id>  to restore an older snapshot
```

What it does:
1. Fetches the snapshot from Hetzner into a tempdir on www1.
2. Creates a fresh docker volume `eusolicit-restore-temp-data`.
3. Untars the basebackup into that volume.
4. Starts a temporary postgres container `eusolicit-restore-temp` on port `127.0.0.1:25432`.
5. Runs smoke queries: `pg_isready`, `SELECT version_num FROM client.alembic_version`, `SELECT count(*) FROM client.users`.

Verify manually:
```bash
psql -h 127.0.0.1 -p 25432 -U eusolicit -d eusolicit
# inside psql:
\du                            # 7 roles should appear
SELECT version_num FROM client.alembic_version;
SELECT version_num FROM admin.alembic_version;
SELECT version_num FROM pipeline.alembic_version;
SELECT version_num FROM gateway.alembic_version;
SELECT version_num FROM notification.alembic_version;
SELECT count(*) FROM client.users;
\q
```

Compare the alembic heads to the values in the snapshot's `alembic-heads.txt` (captured at backup time):
```bash
ls /home/docker/backups/eusolicit/<date>/alembic-heads.txt   # local copy, if still within 7d
```

Tear down:
```bash
docker rm -f eusolicit-restore-temp
docker volume rm eusolicit-restore-temp-data
```

---

## Production restore path — `--to-prod`

> **⚠ DESTRUCTIVE.** Replaces the prod pgdata volume. Run only during a declared incident.

### Pre-restore steps

1. **Declare the incident.** Page on-call (per `runbooks/branch-protection-policy.md` or `incident-management/incident-response-process.md`).
2. **Capture the current state for forensics** (if the existing volume is recoverable at all):
   ```bash
   sudo cp -a /var/lib/docker/volumes/eusolicit-app_pgdata /home/debian/forensics/pgdata-$(date -u +%Y%m%dT%H%M%S)
   ```
3. **Run the drill mode first** (`--to-fresh-volume`) to confirm the off-site backup is intact BEFORE wiping the volume. ~5 min. Worth it.
4. **Notify customers** via the status page (`runbooks/status-page-comms-templates.md`).

### Restore

```bash
cd /home/debian/Projects/eusolicit/eusolicit-app
bash scripts/onprem/postgres-restore.sh --to-prod
# optional: --snapshot <id>  for point-in-time recovery
```

The script will:
1. Prompt for confirmation (type `I-UNDERSTAND` to proceed).
2. Stop the eusolicit stack via `docker compose stop`.
3. Remove the `eusolicit-app_pgdata` volume.
4. Recreate it from the restic snapshot.
5. Re-run `scripts/deploy.sh` to start the stack.

### Post-restore validation

```bash
# 1. All 6 services healthy
for port in 18001 18002 18003 18004 18005 18007; do
  curl -sf "http://127.0.0.1:${port}/healthz" || echo "port ${port}: FAIL"
done

# 2. Alembic heads consistent
docker exec eusolicit-app-postgres-1 \
  psql -U eusolicit -d eusolicit -c \
  "SELECT 'client', version_num FROM client.alembic_version
   UNION ALL SELECT 'admin', version_num FROM admin.alembic_version
   UNION ALL SELECT 'pipeline', version_num FROM pipeline.alembic_version
   UNION ALL SELECT 'gateway', version_num FROM gateway.alembic_version
   UNION ALL SELECT 'notification', version_num FROM notification.alembic_version;"

# 3. Public site responds
curl -sLf https://www.eusolicit.com | grep -qi "EU Solicit" && echo "public OK"

# 4. Application-layer smoke test
curl -sf https://www.eusolicit.com/admin-api/healthz
curl -sf https://www.eusolicit.com/api/v1/auth/me   # expect 401 (auth gate working)
```

If any of the above fails: investigate before declaring incident resolved. Do NOT close the incident on assumed-good state.

### Document what happened

Open a post-mortem ticket using `eusolicit-docs/post-mortems/<template>` template within 48h. Update `incident-management/auto-sync-incidents.md` if the restore was triggered by an auto-sync-class failure.

---

## Cross-host restore (www1 lost)

Used when www1 itself is unrecoverable (hardware failure, DC fire, etc.).

### Prerequisites

- A new Debian 13 host with `onprem-06` (www1-as-code) playbooks applied. Without those, RTO is "however long it takes the engineer to remember everything."
- SSH access to the Hetzner Storage Box restored (same SSH key, or rotate via Hetzner UI).
- The `.env.backup` file recovered (from a password manager, offline backup, or memorized values).

### Steps (high level)

1. **Provision new host** per `eusolicit-docs/runbooks/www1-rebuild.md` (lands in `onprem-06`).
2. **Clone the eusolicit-app repo** to the new host.
3. **Restore `.env.backup`** to `/home/debian/eusolicit-overrides/.env.backup`.
4. **Run** `bash scripts/onprem/postgres-restore.sh --to-fresh-volume` to confirm the snapshot is intact.
5. **Run** `bash scripts/onprem/postgres-restore.sh --to-prod` to populate the pgdata volume on the new host.
6. **Start the eusolicit stack**: `bash scripts/deploy.sh`.
7. **Point DNS at the new host** (operator action; out of script scope).

---

## Point-in-time recovery (PITR)

When a corruption was introduced at a known timestamp (e.g., a bad migration ran at 14:32 UTC), restore the most recent basebackup BEFORE that timestamp and replay WAL up to the target.

```bash
# 1. Find the right basebackup snapshot
source /home/debian/eusolicit-overrides/.env.backup
restic snapshots --tag basebackup --compact

# 2. Restore the basebackup
bash scripts/onprem/postgres-restore.sh --to-fresh-volume --snapshot <pre-corruption-snapshot-id>

# 3. Configure recovery target on the temp container
docker exec eusolicit-restore-temp bash -c \
  "echo \"recovery_target_time = '2026-05-09 14:31:00 UTC'\" >> /var/lib/postgresql/data/postgresql.conf
   echo \"recovery_target_action = 'pause'\" >> /var/lib/postgresql/data/postgresql.conf
   touch /var/lib/postgresql/data/recovery.signal"
docker restart eusolicit-restore-temp

# 4. Pull WAL segments from restic and place them in the pg_wal directory
#    (left as operator detail — depends on which WAL push tag is being used)

# 5. Verify data state at target time. If correct, promote out of recovery
#    and proceed with --to-prod, OR cherry-pick the data needed and copy it
#    to the live prod DB.
```

This path is non-trivial and rarely needed. Prefer `--to-prod` from a clean snapshot unless PITR is the only option.

---

## First drill — measure RTO

> **Operator action.** Required to close AC 8.

1. Confirm `restic`, `.env.backup`, and SSH key are in place.
2. `bash scripts/onprem/postgres-backup.sh` — produce a fresh snapshot (counts as a manual run; cron should already be daily).
3. `bash scripts/onprem/postgres-restore.sh --to-fresh-volume` — restore and validate.
4. **Time the wall-clock duration** from "start of restore" to "smoke queries return expected values."
5. Record the result in this runbook under §First Drill Results below.

### First Drill Results

| Date | Restore mode | Snapshot ID | Wall-clock duration | Smoke queries | Issues observed |
|---|---|---|---|---|---|
| TBD | — | — | — | — | (To be filled by operator on first drill.) |

Target: ≤ 4h. Anything significantly longer indicates we need to tighten the runbook (or invest in faster off-site bandwidth, parallel restore, or pre-staged hot standby).

---

## References

- Story: `eusolicit-docs/implementation-artifacts/onprem-01-postgres-backup-and-recovery.md`
- Backup script: `eusolicit-app/scripts/onprem/postgres-backup.sh`
- Restore script: `eusolicit-app/scripts/onprem/postgres-restore.sh`
- Restore-test (weekly cron): `eusolicit-app/scripts/onprem/postgres-restore-test.sh`
- Cron file: `eusolicit-app/infra/host/cron.d/eusolicit-backup`
- Env template: `eusolicit-app/infra/host/eusolicit-overrides/.env.backup.example`
- ADR-010 (2026-05-11): `eusolicit-docs/planning-artifacts/architecture.md`
- Pivot decision: `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md`
- Companion: `onprem-02-redis-persistence-and-recovery.md` (Redis side)