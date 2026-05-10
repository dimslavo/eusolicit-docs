# Story onprem-04: Disk and Resource Monitoring

Status: ready-for-dev

**Origin:** Onprem pivot decision (eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md). New launch-blocker; no prior PE.* counterpart (AWS-managed services masked the host-level concern).
**Priority:** P0 — launch-blocker. Root partition is currently at **83–85%**. An out-of-disk event would take prod down with no warning.
**Epic:** Post-Epic-21 / Onprem Launch
**Points:** 1 | **Type:** infra / observability

## Story

As a **platform-engineering operator**,
I want **host-level alerts on www1 covering disk, memory, swap, container restart loops, postgres WAL growth, Docker daemon health, and NTP drift**,
so that **the single-host posture is operationally defensible — surprises (disk fills, container crash loops, Docker daemon hang) reach the on-call before they take prod down**.

## Acceptance Criteria

1. **`node_exporter` running on www1** — scraped by the Prometheus deployed in `onprem-03`. Provides standard host metrics: filesystem, memory, swap, CPU, load.

2. **`cAdvisor` (or equivalent) for container metrics** — covers per-container CPU, memory, restart counts. Scraped by Prometheus.

3. **Alert: `/` filesystem > 85%** — current measured state is 83%. Alert fires at 85% (warning), pages at 90% (critical). Runbook: `eusolicit-docs/runbooks/disk-cleanup-root.md` (NEW).

4. **Alert: `/home` filesystem > 90%** — currently 15%; ample headroom. Mostly guards against backup-induced fill (onprem-01 writes to `/home/docker/backups/eusolicit/`). Runbook: `eusolicit-docs/runbooks/disk-cleanup-home.md` (NEW).

5. **Alert: available memory < 2 GB** — currently 21 GB available of 31 GB total. Alert at <2 GB sustained 5 min. Runbook: `eusolicit-docs/runbooks/memory-pressure.md` (NEW).

6. **Alert: swap usage > 50%** — currently 3.1 GB of 63 GB. Alert at >32 GB sustained 10 min (indicates serious memory pressure or leak). Runbook: shared with memory-pressure.md.

7. **Alert: container restart loop** — any `eusolicit-app-*` container with >3 restarts in a 5-min window. Runbook: `eusolicit-docs/runbooks/container-restart-loop.md` (NEW).

8. **Alert: postgres `pg_wal/` directory growth > 5 GB/h** — indicates WAL archiving has stalled (which would silently break RPO commitment in `onprem-01`). Scraped via postgres_exporter custom query. Runbook: `eusolicit-docs/runbooks/wal-archiving-stalled.md` (NEW).

9. **Alert: Docker daemon down** — `up{job="docker"}` == 0. Critical, pages immediately. Runbook: `eusolicit-docs/runbooks/docker-daemon-recovery.md` (NEW).

10. **Alert: NTP drift > 30s** — `node_timex_offset_seconds > 30 OR < -30`. Important because backup timestamps + alembic versions + audit logs all assume sane time. Runbook: `eusolicit-docs/runbooks/ntp-drift.md` (NEW).

11. **Every alert ships with a runbook URL** — enforced by the existing PE.06 lint gate `scripts/check_runbook_url_coverage.py`. The CI job that runs this gate must pass after this story.

12. **Alertmanager routing tested** — each alert fires correctly in a contrived staging scenario (e.g., fill `/tmp` to push `/` over 85% temporarily; verify alert fires + email + Telegram delivered).

## Tasks / Subtasks

- [ ] Task 1: Add `node_exporter` + `cadvisor` to the observability docker-compose (from `onprem-03`).
- [ ] Task 2: Author 7 new runbooks under `eusolicit-docs/runbooks/` (one per alert family).
- [ ] Task 3: Add Prometheus rules to `infra/observability/prometheus/rules/host-alerts.yaml`.
- [ ] Task 4: Update `check_runbook_url_coverage.py` allowlist if needed; verify CI gate passes.
- [ ] Task 5: Fire each alert via a contrived scenario in staging or a sacrificial moment on www1; verify routing.

## Dev Notes

### Current www1 baseline (verified live)

| Resource | Current | Alert threshold |
|---|---|---|
| `/` partition | 85% used (13 GB free of 86 GB) | >85% warning, >90% page |
| `/home` partition | 15% used (256 GB free of 319 GB) | >90% warning |
| Memory | 21 GB available of 31 GB total | <2 GB available |
| Swap | 3.1 GB of 63 GB | >32 GB sustained |
| Containers | 14 EU Solicit + N co-tenants, all healthy | restart loops |

### Why `/` is already at 85%

Docker image storage + log retention is on `/var/lib/docker` which lives on `/`. Pruning dangling images (already part of deploy.sh §Cleanup) helps but doesn't address the underlying scaling concern. Long-term fix: move docker storage root to `/home` (config in `/etc/docker/daemon.json`). That's a separate operator task — track in `onprem-06`.

### KraftData-dependent alerts

Per architecture.md line 762, KraftData incidents are excluded from SLA. Alerts on KraftData-dependent SLOs (`slo_target="kraftdata-dependent"`) route to `slack/info` (ticket), not `page`. The existing rules from Story 21-5 already encode this — preserved verbatim.

### References

- ADR-010 rewrite: `architecture.md` §ADR-010 (2026-05-11)
- Decision record: `onprem-pivot-decision-2026-05-11.md`
- onprem-03 (dependency): provides Prometheus + Alertmanager
- pe-06 (parallel): provides email + Telegram receivers
- Existing runbook lint gate: `scripts/check_runbook_url_coverage.py` (Story 21-6)
- Existing runbook directory: `eusolicit-docs/runbooks/` (15 runbooks from Story 21-6)

### Out of scope

- Application-layer alerts (covered by Story 21-5 dashboards + rules, already retained).
- Per-tenant alerts (cardinality concern at small tenant count; revisit if needed).
- Log-based alerts (deferred with Loki).
