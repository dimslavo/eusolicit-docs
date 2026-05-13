---
stepsCompleted: ['step-01-load-context']
lastStep: 'step-01-load-context'
lastSaved: '2026-05-13'
coverageBasis: 'acceptance_criteria'
oracleConfidence: 'high'
oracleResolutionMode: 'formal_requirements'
oracleSources:
  - 'eusolicit-docs/implementation-artifacts/onprem-01-postgres-backup-and-recovery.md'
  - 'eusolicit-docs/implementation-artifacts/onprem-02-redis-persistence-and-recovery.md'
  - 'eusolicit-docs/implementation-artifacts/onprem-03-monitoring-on-www1.md'
  - 'eusolicit-docs/implementation-artifacts/onprem-04-disk-and-resource-monitoring.md'
  - 'eusolicit-docs/implementation-artifacts/onprem-05-deploy-guardrails.md'
  - 'eusolicit-docs/implementation-artifacts/onprem-06-www1-itself-as-code.md'
externalPointerStatus: 'not_used'
---

# Traceability Matrix for Epic 22 - On-Prem Launch

## Resolved Coverage Oracle

The coverage oracle for Epic 22 is derived from the Acceptance Criteria (ACs) explicitly defined within the following 6 story markdown files:

1.  `onprem-01-postgres-backup-and-recovery.md`
2.  `onprem-02-redis-persistence-and-recovery.md`
3.  `onprem-03-monitoring-on-www1.md`
4.  `onprem-04-disk-and-resource-monitoring.md`
5.  `onprem-05-deploy-guardrails.md`
6.  `onprem-06-www1-itself-as-code.md`

This approach provides a **high confidence** oracle, as these documents are the formal requirements for each story in Epic 22. No other formal requirements, contract specifications (e.g., OpenAPI), or external pointers were identified as relevant to the core Epic 22 scope.

## Extracted Acceptance Criteria

### Story: onprem-01: Postgres Backup and Recovery

*   **AC1**: Backup script at `eusolicit-app/scripts/onprem/postgres-backup.sh` - runs `pg_basebackup` against `eusolicit-app-postgres-1` via `docker exec`, archives WAL since the last basebackup, writes to `/home/docker/backups/eusolicit/<YYYY-MM-DD>/`. Encrypted at rest via `restic` (or `borg` — choose at impl time). Log to `/var/log/eusolicit/backup.log` with structured fields (`start_ts`, `end_ts`, `bytes`, `status`, `wal_segments`).
*   **AC2**: Off-site replication to Hetzner Storage Box - credentials live in `/home/debian/eusolicit-overrides/.env.backup` (host-only per project memory note "www1 host-only state"), NEVER committed. `rclone` or `restic` push from the local backup directory to Hetzner Storage Box, EU region, retention policy `7d local + 35d off-site`. Replication step must complete in ≤30 min for a baseline DB size (≤10 GB).
*   **AC3**: WAL archiving configured - `postgresql.conf` mounted into the postgres container sets `archive_mode = on` + `archive_command = 'rclone copy %p hetzner:eusolicit-wal/'` (or `wal-g wal-push %p`). RPO ≤ 24h is achieved by combining a daily basebackup + continuous WAL push.
*   **AC4**: Daily cron at 03:00 - registered on www1 via `/etc/cron.d/eusolicit-backup` (or systemd timer; choose at impl). Aligns with the existing co-tenant pattern (`/home/docker/backups/*_0300/`). Runs as `debian` user; outputs to syslog with tag `eusolicit-backup`.
*   **AC5**: Weekly automated restore-test - Sunday 04:00 cron. Spins up a temporary postgres container on port `25432` (loopback-only), restores the latest off-site backup, runs:
    *   `pg_isready -h 127.0.0.1 -p 25432` → must return 0
    *   `psql -h 127.0.0.1 -p 25432 -U eusolicit -d eusolicit -c "SELECT count(*) FROM client.users;"` → must return a non-error
    *   `psql -h 127.0.0.1 -p 25432 -U eusolicit -d eusolicit -c "SELECT version_num FROM client.alembic_version;"` → must match `071` (or current head per `make migrate-status`)
    Tear down the temp container. Report success/failure via alertmanager (depends on `onprem-03`).
*   **AC6**: Restore runbook at `eusolicit-docs/runbooks/postgres-restore.md` - step-by-step recovery for the operator-on-call, including: emergency vs. routine restore paths, decision tree for partial vs. full restore, credential retrieval, network reconfiguration if restoring to a different host. Must include a runbook URL that passes `scripts/check_runbook_url_coverage.py` (PE.06 lint gate).
*   **AC7**: Local retention 7 days, off-site retention 35 days - enforced via `restic forget --keep-daily 7` (local) and `restic forget --keep-daily 35` (Hetzner). Pruning runs in a separate cron at 04:30.
*   **AC8**: First full restore drill - execute the restore runbook end-to-end against a fresh docker volume on a different host (or with a different container name on www1). Measure wall-clock time from "start" to "EU Solicit application reads/writes restored DB successfully." Target: ≤4h. Document the result in `eusolicit-docs/runbooks/postgres-restore.md` §First Drill Results.

### Story: onprem-02: Redis Persistence and Recovery

*   **AC1**: AOF persistence enabled - `redis.conf` mounted into `eusolicit-app-redis-1` sets `appendonly yes` and `appendfsync everysec` (default balance: ≤1s data loss on crash; no per-write fsync overhead). `auto-aof-rewrite-percentage 100` and `auto-aof-rewrite-min-size 64mb` so the AOF doesn't grow unboundedly.
*   **AC2**: RDB snapshots on hourly cadence - `save 3600 1` (snapshot if ≥1 change in last hour) in `redis.conf`. RDB file at `/data/dump.rdb` (default).
*   **AC3**: Off-site RDB replication - `dump.rdb` is included in the daily 03:00 backup (same Hetzner Storage Box target as `onprem-01`). Path: `hetzner:eusolicit/redis/<date>/dump.rdb`. Retention matches: 7d local + 35d off-site.
*   **AC4**: Controlled-restart drill - `docker compose restart redis` is executed on a sentinel test write set. Verify:
    *   Redis Streams consumer groups recover via `XINFO STREAMS <stream>` (consumer-group state must be preserved across restart).
    *   Celery beat tasks fire on schedule post-restart (verify with the `reset_stuck_proposals_task` if `dw-02` is complete, or a tracer task).
    *   The 6 services' `redis-py` clients auto-reconnect via the existing resilience hardening (`socket_keepalive=True`, `health_check_interval=30`, retry policy from Story 21-3 — RETAINED in code per ADR-010 rewrite).
    *   Cache hits resume within 10s of restart.
*   **AC5**: Restore runbook at `eusolicit-docs/runbooks/redis-restore.md` - documents two restore paths: (a) replay from local AOF (zero off-site dependency, fastest), (b) restore RDB from Hetzner Storage Box if local volume is lost. Includes credential retrieval, network reconfig, and post-restore validation queries.
*   **AC6**: First controlled-restart drill executed - runbook executed against the live www1 stack at a low-traffic window. Document measured "Redis-unavailable window" (target ≤30s) and any service-level errors observed. Add a §First Drill Results section.

### Story: onprem-03: Monitoring on www1

*   **AC1**: Implementation choice documented at Task 0 - pick one of: (a) Reuse the co-tenant `lifematch-dev-*` observability stack by adding EU Solicit scrape targets + dashboards into the existing Grafana/Prometheus running on www1, OR (b) Deploy a separate `eusolicit-observability` docker-compose stack under a dedicated docker network. The choice is documented in `eusolicit-docs/runbooks/observability-stack.md` with the trade-off rationale.
*   **AC2**: Prometheus scrape config covers all 7 EU Solicit services - `client-api` (port 18001), `admin-api` (18002), `data-pipeline` (18003), `ai-gateway` (18004), `notification` (18005), `integrations-api` (18007), plus `frontend` (13000 — Next.js metrics if any). Job names match the existing Grafana dashboard variable assumptions (`job="client-api"`, etc.).
*   **AC3**: Grafana dashboard JSONs loaded verbatim - the 7 per-service + 1 cross-cutting `platform-slo.json` files at `infra/observability/grafana/dashboards/` are mounted into Grafana via `grafana-dashboards.yaml` provisioning config (already in repo from Story 21-5). Dashboard variables resolved against the new on-host Prometheus datasource.
*   **AC4**: Prometheus rules loaded verbatim - the rule files at `infra/observability/prometheus/rules/` are mounted into the Prometheus container. The `slo_target` label routing (`"platform"` vs `"kraftdata-dependent"`) keeps working — KraftData-dependent SLO violations still don't page (per architecture.md line 762 carry-forward).
*   **AC5**: Postgres + Redis exporters as docker containers - `prom/postgres-exporter` scrapes `eusolicit-app-postgres-1` via `pg_stat_statements` + standard metrics. Auth as `monitoring_role` (per Story 21-5). `oliver006/redis_exporter` scrapes `eusolicit-app-redis-1`. Replaces the AWS CloudWatch exporter approach from Story 21-5.
*   **AC6**: Alertmanager configured - `infra/observability/alertmanager/alertmanager.yaml` (already in repo from Story 21-5) is loaded, with two routing changes vs. the original: Remove `pagerduty` receivers (PagerDuty cancelled in pivot). Add an `email` receiver + a `telegram-bot` webhook receiver (per `pe-06` decision). The `slack` receiver for tickets is kept if a Slack workspace is in use.
*   **AC7**: Reachable via reverse proxy - Grafana served behind the existing nginx on www1 at `grafana.eusolicit.internal` (or similar private subdomain), with basic-auth or SSO behind it. Not exposed on the public internet.
*   **AC8**: `/metrics` endpoint contract regression test passes - the existing test at `tests/unit/test_metrics_endpoint_contract.py` (146 pass / 18 skip / 0 fail per Story 21-5) continues to pass after this change. No application code changes expected; this AC is a guardrail.

### Story: onprem-04: Disk and Resource Monitoring

*   **AC1**: `node_exporter` running on www1 - scraped by the Prometheus deployed in `onprem-03`. Provides standard host metrics: filesystem, memory, swap, CPU, load.
*   **AC2**: `cAdvisor` (or equivalent) for container metrics - covers per-container CPU, memory, restart counts. Scraped by Prometheus.
*   **AC3**: Alert: `/` filesystem > 85% - current measured state is 83%. Alert fires at 85% (warning), pages at 90% (critical). Runbook: `eusolicit-docs/runbooks/disk-cleanup-root.md` (NEW).
*   **AC4**: Alert: `/home` filesystem > 90% - currently 15%; ample headroom. Mostly guards against backup-induced fill (onprem-01 writes to `/home/docker/backups/eusolicit/`). Runbook: `eusolicit-docs/runbooks/disk-cleanup-home.md` (NEW).
*   **AC5**: Alert: available memory < 2 GB - currently 21 GB available of 31 GB total. Alert at <2 GB sustained 5 min. Runbook: `eusolicit-docs/runbooks/memory-pressure.md` (NEW).
*   **AC6**: Alert: swap usage > 50% - currently 3.1 GB of 63 GB. Alert at >32 GB sustained 10 min (indicates serious memory pressure or leak). Runbook: shared with memory-pressure.md.
*   **AC7**: Alert: container restart loop - any `eusolicit-app-*` container with >3 restarts in a 5-min window. Runbook: `eusolicit-docs/runbooks/container-restart-loop.md` (NEW).
*   **AC8**: Alert: postgres `pg_wal/` directory growth > 5 GB/h - indicates WAL archiving has stalled (which would silently break RPO commitment in `onprem-01`). Scraped via postgres_exporter custom query. Runbook: `eusolicit-docs/runbooks/wal-archiving-stalled.md` (NEW).
*   **AC9**: Alert: Docker daemon down - `up{job="docker"}` == 0. Critical, pages immediately. Runbook: `eusolicit-docs/runbooks/docker-daemon-recovery.md` (NEW).
*   **AC10**: Alert: NTP drift > 30s - `node_timex_offset_seconds > 30 OR < -30`. Important because backup timestamps + alembic versions + audit logs all assume sane time. Runbook: `eusolicit-docs/runbooks/ntp-drift.md` (NEW).
*   **AC11**: Every alert ships with a runbook URL - enforced by the existing PE.06 lint gate `scripts/check_runbook_url_coverage.py`. The CI job that runs this gate must pass after this story.
*   **AC12**: Alertmanager routing tested - each alert fires correctly in a contrived staging scenario (e.g., fill `/tmp` to push `/` over 85% temporarily; verify alert fires + email + Telegram delivered).

### Story: onprem-05: Deploy Guardrails

*   **AC1**: Branch protection on `main` requires CI green - GitHub repo settings → Branches → main → Require status checks to pass before merging → "CI / check" required. Configured manually via the GitHub UI (operator action) and documented in `eusolicit-docs/runbooks/branch-protection-policy.md` (NEW).
*   **AC2**: `deploy.yml` only fires after CI passes - currently `deploy.yml` and `ci.yml` run in parallel on push to main; the deploy job has no `needs:` dependency on CI. Change: add `concurrency` + `if: github.event.workflow_run.conclusion == 'success'` pattern, OR convert deploy.yml to trigger on `workflow_run: workflows: [CI]` instead of `push: branches: [main]`. Test by intentionally pushing a failing change and confirming deploy does NOT run.
*   **AC3**: `chore: auto-sync` PR policy - auto-sync commits must land via PR (not direct push). The auto-sync source is the Orchestrator (lives outside `eusolicit-app/`), not a `.github/workflows/` file in this repo. Therefore this AC is closed by changes in `Orchestrator/`, not here:
    *   The Orchestrator config that currently pushes `chore: auto-sync` directly to `main` is modified to open a PR against `main` instead.
    *   The PR template states "Requires human review before merge — auto-sync is unverified code per project memory."
    *   The PR author (Orchestrator bot identity) cannot self-approve (GitHub branch-protection setting from AC 1: require review from someone other than the author — this already covers it).
    *   **Cross-repo coordination:** the Orchestrator-side change is tracked separately; this AC closes when the *first* `chore: auto-sync` PR is opened against `main` and merged through human review.
*   **AC4**: Auto-rollback on smoke-test failure - `deploy.yml`'s smoke-test step (already present, currently only logs) is extended to:
    *   On failure: `git reset --hard <previous-sha>` on www1, re-run `bash scripts/deploy.sh` to redeploy the previous version, exit non-zero with a clear "ROLLED BACK to <sha>" message.
    *   The `<previous-sha>` is captured BEFORE the new deploy starts (e.g., `git rev-parse HEAD` and saved to a tempfile or env var).
    *   Alertmanager (deps `onprem-03`) receives the rollback event.
*   **AC5**: `deploy.log` includes dated headers - current log lines start with `[HH:MM:SS]` only (per direct file inspection on www1). Update `scripts/deploy.sh` to emit `[YYYY-MM-DD HH:MM:SS]` so log archeology across days works. Add a section divider line `========== Deploy <SHA> at <DATE> ==========` at the start of each deploy invocation.
*   **AC6**: Deploy approval annotation - every `deploy.yml` run posts a GitHub deployment status to the relevant commit, including the deploy SHA, smoke-test results, and rollback status if applicable. This gives `git log` linkable evidence of "what was actually deployed when."
*   **AC7**: Auto-sync incident recurrence: zero in 30 days post-merge - measurable acceptance criterion. Track via a `eusolicit-docs/incident-management/auto-sync-incidents.md` log; if any auto-sync commit causes a prod outage between merge of this story and 30 days later, the story fails its long-term AC and a follow-up ticket opens.

### Story: onprem-06: www1 Itself as Code

*   **AC1**: `infra/host/` directory created (NEW) with the following structure:
    ```
    infra/host/
      README.md                              # operator entry point
      inventory.yaml                          # pins www1 specs (Debian 13, kernel, deps)
      playbooks/
        01-base-system.yml                    # apt update, base packages, NTP, ssh
        02-docker.yml                         # docker engine + compose v2 + daemon.json
        03-nginx-and-certbot.yml              # nginx install + certbot + sites
        04-debian-user.yml                    # user, groups, sudoers, authorized_keys
        05-firewall.yml                       # ufw or iptables rules
        06-eusolicit-overrides.yml            # creates /home/debian/eusolicit-overrides/ skeleton
      templates/
        nginx-sites/                          # nginx site templates (eusolicit.com, etc.)
        env.prod.example                      # template for /home/debian/Projects/.../.env.prod
        env.backup.example                    # template for backup credentials
        authorized_keys.example
    ```
    Format choice: **Ansible** if the team is comfortable; **plain shell scripts** if not. Pick at impl, document.
*   **AC2**: nginx site config templated - `/etc/nginx/sites-enabled/eusolicit.com` (the one verified live during pre-flight) is replicated as a template at `infra/host/templates/nginx-sites/eusolicit.com.j2`. Variables: domain names, upstream ports (the `1xxxx` port-collision-avoidance scheme), TLS cert paths.
*   **AC3**: certbot renewal config - `infra/host/playbooks/03-nginx-and-certbot.yml` installs certbot + the cert-renewal hook. Includes documentation on initial cert provisioning (`certbot --nginx --expand -d www.eusolicit.com -d eusolicit.com -d admin.eusolicit.com`) since the initial run can't be automated without manual DNS validation.
*   **AC4**: `debian` user provisioning - playbook creates the `debian` user with: sudo NOPASSWD for `apt`, `systemctl`, `docker`; SSH access via authorized_keys template; ownership of `/home/debian/Projects/eusolicit/`. Template the authorized_keys file (do NOT commit the real keys; commit an `.example`).
*   **AC5**: Docker daemon config - `/etc/docker/daemon.json` codified. Includes: storage driver, log rotation, **storage root moved to `/home/docker/`** (resolves the current `/` partition pressure at 85%).
*   **AC6**: Firewall rules - `ufw` or `iptables` rules: allow 22 (SSH), 80 (HTTP), 443 (HTTPS), deny everything else inbound. Templated.
*   **AC7**: Recovery runbook at `eusolicit-docs/runbooks/www1-rebuild.md` - step-by-step: "you have a fresh Debian 13 box + git clone of eusolicit-app + off-site backup credentials. Restore service in ≤4h." Includes order-of-operations: base system → docker → eusolicit-overrides skeleton → restore postgres from backup → restore redis from backup → start services → verify health.
*   **AC8**: Practice recovery on a sacrificial VM - execute the rebuild runbook end-to-end on a fresh Debian 13 VM (Hetzner Cloud trial or local KVM). Measure wall-clock time. Target: ≤4h. Document result in §First Rebuild Drill Results in the runbook.