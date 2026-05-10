---
title: 'EU Solicit Infra Pivot — Single-Host On-Premise Docker for Launch'
date: 2026-05-11
status: Ratified
author: 'John (PM)'
supersedes: 'ADR-010 (2026-04-25)'
---

# EU Solicit Infra Pivot — Single-Host On-Premise Docker for Launch

## Summary

EU Solicit is pivoting away from the 2026-04-25 ADR-010 commitment to managed AWS infrastructure (RDS Multi-AZ + ElastiCache + EKS + AMP/AMG). The production launch will ship on **`www1.endigitalx.com` as a single Docker host**, with PostgreSQL 16 and Redis 7 running as containers backed by host-mounted volumes. Backups go to Hetzner Storage Box; paging via email + Telegram bot; PagerDuty cancelled; the 99.9% SLA promise is withdrawn for launch in favor of an RTO ≤ 4h / RPO ≤ 24h commitment paired with a "Service is in beta. Best-effort availability." disclosure on the Trust Center.

## The trigger

Pre-flight review of the 2026-05-12 `pe-02-rds-multi-az-cutover` (the first scheduled AWS-migration operator phase) surfaced 5 blockers:

1. `prod/main.tf` hardcoded `engine_version='16'` (fails module regex validator) and `backup_retention_period=30` (violates PE.02 epic line 56 mandate of 35). Tfvars overrides were silently ignored because the variables were never declared. **Fixed 2026-05-10** by 📋 John (vars declared, module call rewired in `environments/{prod,staging}/main.tf`; both `terraform validate` clean) — but this fix is now moot since the AWS Terraform is being deleted.
2. `staging/main.tf` had identical disconnect. Also fixed 2026-05-10, also now moot.
3. Source postgres on www1 has `wal_level=replica` (verified live), so the runbook's preferred Method A (logical replication) is impossible without a postgres restart.
4. No live `terraform plan` had ever been run against any AWS account (D-2 deviation). The runbook's "expected plan" output is unverified.
5. **The kubernetes module is a TODO stub.** `modules/kubernetes/main.tf` has every resource block commented out; outputs return literal `""`. No EKS cluster exists in any environment. The entire "managed RDS in EKS" migration path the runbook describes assumes a prerequisite that has never been built.

Blocker 5 was the decision-changer. Closing it requires implementing the kubernetes module (~2–3 days), provisioning EKS in staging then prod, then running the existing PE.02 work — total ~2–3 weeks slip — at ~€2,000–2,500/mo ongoing cost.

## Why the pivot

The 2026-05-04 outage that left prod down for 4 hours was caused by `chore: auto-sync 2026-05-09 18:31:59` shipping unverified code through a deploy pipeline with no CI gate. **EKS would not have prevented that outage.** Fixing the deploy gate — a one-day job — would have. The recurring failure mode is the release pipeline, not the runtime infrastructure.

Independent of that, ADR-011 (Trust Center as static Next.js) and ADR-001 (single Postgres with logical schema isolation) already encoded a preference for "boring tech that the team can operate." The 2026-04-25 ADR-010 was the outlier — it imported managed-AWS complexity in exchange for a 99.9% uptime number that the team had no measurement of and no operator burden to defend.

The simpler launch path:
- Removes the EKS prerequisite (the kubernetes module stays a stub; will be deleted).
- Preserves what works (www1 docker stack has been running production for the last several weeks; foundations are reasonable).
- Saves ~€2,000+/mo annually.
- Ships in ~3 weeks instead of ~6.
- Honestly states the availability posture instead of overpromising.

## Options considered

| Option | Description | Effort | Monthly cost | Earliest launch |
|---|---|---|---|---|
| A | Stay the course on 2026-04-25 ADR-010 (RDS in EKS). | 2–3 wks | €2,000–2,500 | 2026-05-26 to 06-02 |
| B | Decouple DB-HA from compute-HA: RDS Multi-AZ + ElastiCache without EKS, keep www1 compute. | 1 wk | €300–500 (RDS + VPC) | ~2026-05-18 |
| **C (chosen)** | **Drop AWS managed services entirely. Launch on single-host Docker on www1.** | ~3 wks (split across 6 new `onprem-*` stories) | **€10–20 (Hetzner Storage Box + Telegram)** | **2026-06-01** |
| D | Defer the launch entirely until full AWS migration is delivered. | 6+ wks | — | indefinite |

Option C accepts a single compute SPoF and gives up the 99.9% uptime promise. In exchange: 100x cheaper run rate, no AWS account dependency, simpler operator model, and a launch that actually ships.

## Ratified sub-decisions (2026-05-11)

| Question | Decision |
|---|---|
| Off-site backup destination | **Hetzner Storage Box** (EU residency, dedicated capacity, likely already in stack) |
| Public SLA wording | **Drop public SLA entirely for launch.** Disclosure on Trust Center: "Service is in beta. Best-effort availability." Internal RTO/RPO targets: 4h / 24h. |
| AWS Terraform disposition | **Delete outright.** No archive; accept re-implementation cost if a Phase-2 HA migration ever happens. |
| Paging platform | **Replace PagerDuty with email + Telegram bot.** ~€0/mo. Single-engineer rotation as a calendar reminder, not a platform. |

## What's retained from the PE.* work

All of the application-layer hardening from Stories 21-1 through 21-6 is **kept verbatim** — it applies to any deployment topology:

- `pool_pre_ping=True` across all 6 services (Postgres reconnect resilience)
- `redis-py` resilience hardening: `socket_keepalive=True`, `health_check_interval=30`, `retry=Retry(ExponentialBackoff(cap=10, base=1), 3)`, `retry_on_error=[ConnectionError, TimeoutError]`, `socket_connect_timeout=5`, `socket_timeout=10`
- Celery `broker_connection_retry_on_startup=True` + `broker_transport_options` resilience keys
- `/metrics` endpoints via `eusolicit_common.observability` middleware (5 services + data-pipeline + integrations-api)
- 7 Grafana dashboard JSONs at `infra/observability/grafana/dashboards/`
- 4 Prometheus rule files at `infra/observability/prometheus/rules/`
- 15 runbooks at `eusolicit-docs/runbooks/` + 4 incident-management docs + post-mortem skeleton
- 7 k6 baseline scripts at `tests/load/` + populated `load-test-results.md`
- Canonical 7-role init SQL at `infra/postgres/init/01-init-schemas-and-roles.sql`
- The CI lint gate `scripts/check_runbook_url_coverage.py` (runbook URL coverage)

## What's removed

- `infra/terraform/modules/` — all 7 modules: database, redis, kubernetes, monitoring, oncall, storage, networking (the networking module had no real downstream consumer either)
- `infra/terraform/environments/` — prod, staging, dev
- (To be decided in a separate pass:) `infra/helm/eusolicit-service/` chart, with PDBs/HPAs/ESO/NetworkPolicy — all k8s-specific and unused in single-host docker
- (To be decided in a separate pass:) `pe-02-cutover-runbook.md` and `pe-03-cutover-runbook.md` — AWS-specific migration runbooks that no longer apply; recommendation is to mark `SUPERSEDED` and move to `implementation-artifacts/.archive/` rather than delete, to preserve the design reasoning

## What's new (the 6 `onprem-*` launch-blocker stories)

| ID | Purpose | Effort | Owner |
|---|---|---|---|
| `onprem-01-postgres-backup-and-recovery` | Daily `pg_basebackup` + WAL archiving; off-site replication to Hetzner Storage Box; weekly automated restore-test; restore runbook. Acceptance: full restore to a fresh docker volume completes in ≤4h. | ~1 day | platform-engineering |
| `onprem-02-redis-persistence-and-recovery` | AOF tuning, RDB snapshots, off-site RDB replication, controlled-restart drill verifying Celery + Redis Streams survive. | ~½ day | platform-engineering |
| `onprem-03-monitoring-on-www1` | Prometheus + Grafana + Alertmanager as docker containers on www1 (either reusing the co-tenant `lifematch-dev-*` observability stack or deploying a separate eusolicit-observability compose). Loads the 7 Grafana dashboard JSONs + Prometheus rules verbatim. | ~1 day | platform-engineering |
| `onprem-04-disk-and-resource-monitoring` | Host-level alerts (/ > 85%, /home > 90%, mem < 2GB available, container restart loops, postgres pg_wal/ growth, Docker daemon down, NTP drift). Every alert ships with runbook URL. | ~½ day | platform-engineering |
| `onprem-05-deploy-guardrails` | Branch protection on main + require CI green + chore:auto-sync PR policy + deploy.yml auto-rollback on smoke-test failure + dated deploy.log. **Closes the auto-sync outage class.** | ~½ day | platform-engineering |
| `onprem-06-www1-itself-as-code` | Codify host-only state (authorized_keys, eusolicit-overrides, .env.prod, /etc/nginx, certbot) as Ansible or shell scripts under `infra/host/`. Required to honestly defend a 4h RTO. | ~1–2 days | platform-engineering |

## Rescoped existing entries

| ID | Change |
|---|---|
| `pe-02-rds-multi-az-cutover` | Closed `done` — superseded by `onprem-01`. |
| `pe-03-elasticache-cutover` | Closed `done` — superseded by `onprem-02`. |
| `pe-04-chaos-drill-execution` | Kept `ready-for-dev`. Scope adapted to single-host: kill-and-recover container, fill-disk drill, postgres crash recovery, Redis AOF replay. |
| `pe-06-pagerduty-rotation-provisioning` | Kept `ready-for-dev`. Scope adapted: implement email + Telegram-bot paging via alertmanager; keep the 15 runbooks; remove PagerDuty Terraform. |
| `public-sla-announcement-soak-gate` | Kept `ready-for-dev`. Scope replaced: publish "Service is in beta. Best-effort availability." disclosure on the Trust Center with RTO/RPO commitment. No numeric SLA. |

## Calendar

| Date | Milestone |
|---|---|
| 2026-05-11 | Decision ratified; sprint-status + ADR-010 updated; AWS Terraform deleted; this document committed. |
| 2026-05-13 → 2026-05-15 | `onprem-01` + `onprem-02` (backup + redis persistence) |
| 2026-05-15 → 2026-05-18 | `onprem-03` + `onprem-04` (monitoring + alerting) |
| 2026-05-15 → 2026-05-18 (parallel) | `onprem-05` (deploy guardrails) |
| 2026-05-18 → 2026-05-22 | `onprem-06` (www1-as-code) |
| 2026-05-18 → 2026-05-22 (parallel) | `drift-recovery` AC4 (Stripe circuit-breakers) + `inj-01` (dependabot) + `dw-01..03` |
| 2026-05-22 → 2026-05-25 | First restore drill + first chaos drill (rescoped `pe-04`) |
| 2026-05-25 → 2026-05-29 | `inj-03` (6 TEA reviews) + `drift-recovery` AC5 (billing metrics) |
| **2026-06-01** | **Launch target.** RTO/RPO promise, no SLA number, status page live. |

## Risks explicitly accepted

- **Single physical host = single point of failure for compute.** An www1 hardware failure or DC issue takes the service offline until restore-from-backup completes. RPO ≤ 24h, RTO ≤ 4h with rehearsed runbooks.
- **No horizontal scaling.** Capacity ceiling = www1's specs (currently 31 GiB RAM, ~258 GB free on /home).
- **Co-tenancy noisy-neighbour exposure.** Mitigated by docker `mem_limit` + cpu reservations.
- **ISO 27001 achievable; SOC 2 Type II availability** clauses may need contractual carveout for enterprise prospects.
- **AWS Terraform re-implementation cost** if a Phase-2 HA migration ever happens. The deleted code is ~1,200 lines plus the kubernetes module that never worked anyway.

## References

- `eusolicit-docs/planning-artifacts/architecture.md` §ADR-010 (rewrite, this commit; historical version preserved under same heading)
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` (entries `pe-02..pe-06`, `public-sla-announcement-soak-gate`, `onprem-01..onprem-06`)
- `eusolicit-docs/implementation-artifacts/pe-02-cutover-runbook.md` (historical reference; AWS migration runbook)
- `eusolicit-docs/implementation-artifacts/pe-03-cutover-runbook.md` (historical reference)
- 📋 John (PM) is the agent of record for this pivot
