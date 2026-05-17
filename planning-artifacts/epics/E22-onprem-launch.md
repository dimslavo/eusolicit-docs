# E22: On-Prem Launch

**Sprint**: 2026-05-13 → 2026-05-22 | **Points**: ~6 days execution + drills | **Dependencies**: ADR-010 pivot decision (2026-05-11); E21 application-layer hardening (kept verbatim) | **Milestone**: Launch 2026-06-01, best-effort availability

**Source**: `eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md`; `eusolicit-docs/implementation-artifacts/onprem-launch-sprint-plan-2026-05-11.md`; `eusolicit-docs/planning-artifacts/architecture.md` §ADR-010 (rewrite).

## Goal

Ship EU Solicit production launch on `www1.endigitalx.com` as a single Docker host. PostgreSQL 16 + Redis 7 run as containers backed by host-mounted volumes. Off-site backups to Hetzner Storage Box. Paging via email + Telegram bot (no PagerDuty). Public posture: "Service is in beta. Best-effort availability." with internal RTO ≤ 4h / RPO ≤ 24h.

This epic exists because the 2026-05-11 ADR-010 pivot abandoned managed-AWS infrastructure (RDS Multi-AZ + ElastiCache + EKS + AMP/AMG) in favor of the simpler launch path. The 6 stories codify what's needed to honestly defend a 4h RTO promise on a single-host topology.

## Acceptance Criteria

- [ ] Daily Postgres `pg_basebackup` + WAL archiving running on www1 with off-site replication to Hetzner Storage Box; weekly automated restore-test passing; full restore-from-backup completes in ≤4h (drill executed and recorded)
- [ ] Redis AOF + RDB persistence configured; off-site RDB replication to Hetzner Storage Box; controlled-restart drill verifying Celery + Redis Streams reconnect within 10s (drill executed and recorded)
- [ ] Prometheus + Grafana + Alertmanager observability stack running on www1 (decision: standalone vs reuse `lifematch-dev-*` stack — chosen and documented); 7 Grafana dashboards + 4 Prometheus rule files loaded verbatim from E21 deliverables
- [ ] Host-level alerts firing for: / > 85%, /home > 90%, mem < 2GB available, container restart loops, `pg_wal/` growth, Docker daemon down, NTP drift; every alert has a `runbook_url` annotation resolving to a runbook
- [ ] Branch protection on `main` enforced in GitHub UI; CI green required for merge; `chore: auto-sync` PR policy documented; `deploy.yml` auto-rollback step verified; dated headers in `deploy.log` confirmed
- [ ] `infra/host/` codifies host-only state (authorized_keys, eusolicit-overrides, .env.prod, /etc/nginx, certbot) as Ansible playbooks or shell scripts; practice rebuild on a sacrificial Debian 13 VM completes successfully (drill executed and recorded)
- [ ] AWS Terraform deleted (`infra/terraform/modules/` and `infra/terraform/environments/`); `infra/helm/` moved to `infra/helm/.archive/` to preserve design for hypothetical Phase-2 HA migration
- [ ] No public 99.9% SLA promised. Trust Center disclosure live (owned by E23 `public-sla-announcement-soak-gate`): "Service is in beta. Best-effort availability." + RTO ≤ 4h / RPO ≤ 24h commitment
- [ ] All application-layer resilience from E21 retained verbatim (`pool_pre_ping=True`, redis-py keepalive + health-check + retry, Celery `broker_connection_retry_on_startup`, `/metrics` endpoints, 15 runbooks, k6 baseline)
- [ ] N8N instance on www1 is version-pinned (no `latest`), backed up via Hetzner Storage Box, has restore + upgrade runbooks, and is observable via Prometheus/Grafana/Alertmanager (per `onprem-07`, added 2026-05-15)

## Stories

### onprem-01: Postgres Backup & Recovery (~1 day, platform-engineering)

Daily `pg_basebackup` + WAL archiving; off-site replication to Hetzner Storage Box; weekly automated restore-test; restore runbook. Acceptance: full restore to a fresh docker volume completes in ≤4h. Source story: `implementation-artifacts/onprem-01-postgres-backup-and-recovery.md` (already at `ready-for-dev` with partial code landed).

### onprem-02: Redis Persistence & Recovery (~½ day, platform-engineering)

AOF tuning, RDB snapshots, off-site RDB replication, controlled-restart drill verifying all 6 services (client-api, admin-api, ai-gateway, data-pipeline-worker, notification-worker, integrations-api) reconnect cleanly and Redis Streams consumer-group offsets survive. Source story: `implementation-artifacts/onprem-02-redis-persistence-and-recovery.md`.

### onprem-03: Monitoring on www1 (~1 day, platform-engineering)

Prometheus + Grafana + Alertmanager as Docker containers on www1. Decision pending: reuse co-tenant `lifematch-dev-*` observability stack vs. deploy a separate `eusolicit-observability` compose. Loads the 7 Grafana dashboard JSONs + 4 Prometheus rule files verbatim from E21 deliverables. Replaces PagerDuty receivers with email + Telegram-bot receivers in `alertmanager.yaml`. Source story: `implementation-artifacts/onprem-03-monitoring-on-www1.md`.

### onprem-04: Disk & Resource Monitoring (~½ day, platform-engineering)

Host-level alerts: / > 85%, /home > 90%, mem < 2GB available, container restart loops, postgres `pg_wal/` growth, Docker daemon down, NTP drift. Every alert ships with `runbook_url` annotation. `scripts/check_runbook_url_coverage.py` CI lint gate (existing) extended to cover new host-alert rules. Source story: `implementation-artifacts/onprem-04-disk-and-resource-monitoring.md`.

### onprem-05: Deploy Guardrails (~½ day, platform-engineering)

Branch protection on main + require CI green + `chore: auto-sync` PR policy + `deploy.yml` `workflow_run` gate + auto-rollback on smoke-test failure + dated `deploy.log` headers. **Closes the auto-sync outage class** (the 2026-05-04 4-hour outage). AC 3 (auto-sync PR policy) was rewritten 2026-05-11 to point at `Orchestrator/` rather than `eusolicit-app/.github/workflows/` — the auto-sync source is the orchestrator, not a GitHub Actions workflow. Source story: `implementation-artifacts/onprem-05-deploy-guardrails.md`.

### onprem-06: www1-as-Code (~1-2 days, platform-engineering)

Codify host-only state (`authorized_keys`, `eusolicit-overrides`, `.env.prod`, `/etc/nginx`, certbot) as Ansible playbooks or shell scripts under `infra/host/`. Required to honestly defend a 4h RTO commitment. Includes practice rebuild on a sacrificial Debian 13 VM (Hetzner Cloud trial or local KVM). Source story: `implementation-artifacts/onprem-06-www1-itself-as-code.md`.

### onprem-07: N8N Instance Ops (NEW 2026-05-15 via IR remediation, ~1 day, platform-engineering)

Codify operational ownership of the shared EU Solicit-owned N8N instance on www1: version pinning (no `latest` tag); host-mounted volume backup to Hetzner Storage Box; weekly workflow JSON export committed to `infra/n8n/workflows-export/`; restore + upgrade runbooks; Prometheus `/metrics` scrape + Grafana dashboard + Alertmanager rules with `runbook_url` annotations; practice rebuild on a sacrificial Debian 13 VM (AP22-D1 discipline). Without this story the N8N substrate has no upgrade/backup/SLO owner — a risk class equivalent to auto-sync defects per project memory. E05 amendment owns workflow *templates*; this story owns the N8N *container itself*. Source story: `implementation-artifacts/onprem-07-n8n-instance-ops.md` with 9 ACs.

## Tests

- Each `*-First Drill Results` section in the per-story runbooks must be populated from a real drill execution (not synthetic) — that's the closure signal.
- CI: `scripts/check_runbook_url_coverage.py` must remain green.
- Smoke: post-restore tests verify schema isolation (per-service roles only have CRUD on their own schema).

## Anti-patterns / known deviations

- **AP22-D1** Practice rebuild on the production www1 host — never. Use a sacrificial VM. (onprem-06 AC8)
- **AP22-D2** Storage-root migration in production during launch sprint — requires separate maintenance window decision; flagged in onprem-06 sub-task.
- **AP22-D3** Reusing the co-tenant `lifematch-dev-*` observability stack without explicit decision — onprem-03 AC1 forces an explicit decision.
- **AP22-D4** Leaving `infra/observability/cloudwatch-exporter/` on disk post-pivot — folded into onprem-03 cleanup.
- **AP22-D5** Editing `pe-02-cutover-runbook.md` / `pe-03-cutover-runbook.md` — already archived under `.archive/` with SUPERSEDED markers; do not touch.

## Cross-epic dependencies

- **E23 dependency**: `pe-06-pagerduty-rotation-provisioning` consumes onprem-03's alertmanager email/Telegram receivers; `pe-04-chaos-drill-execution` consumes onprem-03 + onprem-04 monitoring + alerting.
- **E23 gate**: `public-sla-announcement-soak-gate` cannot close until E22 launch is live + 2 weeks soak.

## Out-of-scope

- ISO 27001 audit preparation (separate parallel programme).
- Phase-2 HA migration (deliberately deferred per ADR-010; `infra/helm/.archive/` preserves the design).
- SOC 2 Type II availability clauses (contractual carveout for enterprise prospects, separate concern).
