# Story onprem-01: Postgres Backup and Recovery

Status: ready-for-dev

**Origin:** Onprem pivot decision (eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md). Replaces `pe-02-rds-multi-az-cutover`.
**Priority:** P0 — launch-blocker. Without this, the RTO ≤ 4h / RPO ≤ 24h commitment is undefendable.
**Epic:** Post-Epic-21 / Onprem Launch
**Points:** 2 | **Type:** backend / infra

## Story

As a **platform-engineering operator**,
I want **daily `pg_basebackup` + WAL archiving for the EU Solicit Postgres container on www1, replicated off-site to Hetzner Storage Box, with a weekly automated restore-test and a written restore runbook**,
so that **we can defend the RTO ≤ 4h / RPO ≤ 24h commitment that ADR-010 (2026-05-11) makes in lieu of a numeric SLA, with measured evidence from a real restore drill**.

## Acceptance Criteria

1. **Backup script at `eusolicit-app/scripts/onprem/postgres-backup.sh`** — runs `pg_basebackup` against `eusolicit-app-postgres-1` via `docker exec`, archives WAL since the last basebackup, writes to `/home/docker/backups/eusolicit/<YYYY-MM-DD>/`. Encrypted at rest via `restic` (or `borg` — choose at impl time). Log to `/var/log/eusolicit/backup.log` with structured fields (`start_ts`, `end_ts`, `bytes`, `status`, `wal_segments`).

2. **Off-site replication to Hetzner Storage Box** — credentials live in `/home/debian/eusolicit-overrides/.env.backup` (host-only per project memory note "www1 host-only state"), NEVER committed. `rclone` or `restic` push from the local backup directory to Hetzner Storage Box, EU region, retention policy `7d local + 35d off-site`. Replication step must complete in ≤30 min for a baseline DB size (≤10 GB).

3. **WAL archiving configured** — `postgresql.conf` mounted into the postgres container sets `archive_mode = on` + `archive_command = 'rclone copy %p hetzner:eusolicit-wal/'` (or `wal-g wal-push %p`). RPO ≤ 24h is achieved by combining a daily basebackup + continuous WAL push.

4. **Daily cron at 03:00** — registered on www1 via `/etc/cron.d/eusolicit-backup` (or systemd timer; choose at impl). Aligns with the existing co-tenant pattern (`/home/docker/backups/*_0300/`). Runs as `debian` user; outputs to syslog with tag `eusolicit-backup`.

5. **Weekly automated restore-test** — Sunday 04:00 cron. Spins up a temporary postgres container on port `25432` (loopback-only), restores the latest off-site backup, runs:
   - `pg_isready -h 127.0.0.1 -p 25432` → must return 0
   - `psql -h 127.0.0.1 -p 25432 -U eusolicit -d eusolicit -c "SELECT count(*) FROM client.users;"` → must return a non-error
   - `psql -h 127.0.0.1 -p 25432 -U eusolicit -d eusolicit -c "SELECT version_num FROM client.alembic_version;"` → must match `071` (or current head per `make migrate-status`)
   Tear down the temp container. Report success/failure via alertmanager (depends on `onprem-03`).

6. **Restore runbook at `eusolicit-docs/runbooks/postgres-restore.md`** — step-by-step recovery for the operator-on-call, including: emergency vs. routine restore paths, decision tree for partial vs. full restore, credential retrieval, network reconfiguration if restoring to a different host. Must include a runbook URL that passes `scripts/check_runbook_url_coverage.py` (PE.06 lint gate).

7. **Local retention 7 days, off-site retention 35 days** — enforced via `restic forget --keep-daily 7` (local) and `restic forget --keep-daily 35` (Hetzner). Pruning runs in a separate cron at 04:30.

8. **First full restore drill** — execute the restore runbook end-to-end against a fresh docker volume on a different host (or with a different container name on www1). Measure wall-clock time from "start" to "EU Solicit application reads/writes restored DB successfully." Target: ≤4h. Document the result in `eusolicit-docs/runbooks/postgres-restore.md` §First Drill Results.

## Tasks / Subtasks

- [ ] Task 1: Choose backup tool (`restic` vs `borg` vs raw `pg_basebackup` + WAL-G). Document choice in the runbook.
- [ ] Task 2: Provision Hetzner Storage Box account; create `.env.backup` on www1 with credentials (do NOT commit).
- [ ] Task 3: Write `scripts/onprem/postgres-backup.sh` (idempotent; safe to re-run).
- [ ] Task 4: Add WAL archiving to `infra/postgres/postgresql.conf.snippet` (mounted into the postgres container via `docker-compose.prod.yml` volume).
- [ ] Task 5: Install cron entries (03:00 daily backup, 04:00 Sunday restore-test, 04:30 prune).
- [ ] Task 6: Write `scripts/onprem/postgres-restore.sh` (mirror of backup script; restore-from-off-site path).
- [ ] Task 7: Author `eusolicit-docs/runbooks/postgres-restore.md` runbook.
- [ ] Task 8: Execute first full restore drill; record measured RTO + any surprises.
- [ ] Task 9: Wire restore-test failure → alertmanager (deps `onprem-03`).

## Dev Notes

### Where the postgres container lives

- Container name: `eusolicit-app-postgres-1` (per `docker ps`)
- Image: `postgres:16` (per `docker-compose.prod.yml`)
- Volume: docker-managed volume (verify name via `docker inspect`); contains `/var/lib/postgresql/data/`
- Port mapping: `127.0.0.1:15432->5432/tcp`
- Master user: `eusolicit`, default DB: `eusolicit`

### Backup credentials pattern

Pattern matches the project memory note about www1 host-only state. `.env.backup` lives at `/home/debian/eusolicit-overrides/.env.backup` and is sourced by the backup script. Operator MUST manually create this file post-impl with Hetzner credentials. Document this in `onprem-06-www1-itself-as-code` so it ends up in the host-as-code playbook.

### Why pg_basebackup over pg_dump

`pg_basebackup` captures a binary snapshot + WAL = consistent point-in-time recovery. `pg_dump` is logical, slower, and loses WAL context. For PITR / RPO ≤ 24h, basebackup + continuous WAL archiving is the standard.

### Restore-time validation queries

After restore, verify alembic heads are consistent with the application code (per the earlier verification done in this pivot review):

| Schema | Expected head |
|---|---|
| `client.alembic_version` | matches latest in `services/client-api/.../alembic/versions/` |
| `admin.alembic_version` | latest in admin-api migrations |
| `pipeline.alembic_version` | latest in data-pipeline migrations |
| `gateway.alembic_version` | latest in ai-gateway migrations |
| `notification.alembic_version` | latest in notification migrations |

### References

- ADR-010 rewrite (2026-05-11): `eusolicit-docs/planning-artifacts/architecture.md` §ADR-010
- Decision record: `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md`
- Existing co-tenant backup pattern: `/home/docker/backups/<date>_0300/` (MariaDB) — same time-of-day, same path root
- Init SQL (consumed at restore-validation): `infra/postgres/init/01-init-schemas-and-roles.sql`
- Postgres container: `eusolicit-app/docker-compose.prod.yml` (postgres service block)
- Project memory: "www1 host-only state" + "Auto-sync ships unverified code" (both relevant)

### Out of scope

- Multi-region replication (single off-site copy is sufficient for RPO ≤ 24h).
- Per-tenant point-in-time restore (single-DB restore; tenant data isolation is logical via schema).
- Live-streaming replication to a hot standby (deferred to a future Phase-2 HA epic if ever).
