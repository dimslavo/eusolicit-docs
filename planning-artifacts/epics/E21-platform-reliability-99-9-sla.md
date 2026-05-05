# E21: Platform Reliability for 99.9% SLA

**Sprint**: 14–17 (parallel to feature epics) | **Points**: 34 | **Dependencies**: Epic 13 carry-forwards | **Milestone**: 99.9% SLA externally publishable
**Source:** PRD v1.1 §7 NFRs; architecture-evaluation §2 Change-5 (architect surfaced this as a separate workstream)

## Goal

Build the infrastructure to back the 99.9% uptime SLA promised in PRD v1.1 §7. Without this work, the SLA cannot credibly be published — current PostgreSQL and Redis are single-instance SPoFs and the k6 baseline carry-forward (now 6+ epics deferred) means we have no measured baseline. **This epic is parallel to Epics 14–20 feature work but gates the public SLA announcement.** PE.01 (k6 baseline) is critical-path: cannot declare any SLO without it. Re-homes Epic 13 carry-forwards (k6 baseline → PE.01; Prometheus /metrics → PE.05); Dependabot remains in Epic 13's set (out of scope here).

99.9% = 43min downtime/month max. Achievable target with managed PostgreSQL HA + managed Redis HA + PodDisruptionBudgets + min-replica enforcement + SLO observability + on-call.

## Acceptance Criteria

- [ ] k6 performance baseline closed for client-api, AI-gateway, data-pipeline endpoints with documented p50/p95/p99 latency and throughput numbers (PE.01 — critical path)
- [ ] PostgreSQL migrated to managed HA (RDS Multi-AZ or equivalent) with automated failover; tested cutover in staging; production cutover during maintenance window
- [ ] Redis migrated to managed HA (Sentinel or managed cluster) with automated failover
- [ ] Every service Helm chart has PodDisruptionBudget with `minAvailable: 1` and minimum replicas ≥ 2 in production
- [ ] Prometheus + Grafana SLO dashboards live with error-budget burn-rate alerting (re-homes Epic 13 Prometheus carry-forward)
- [ ] PagerDuty (or Opsgenie) on-call rotation configured; runbook-density baseline (≥10 runbooks covering top failure modes); incident-management process documented
- [ ] Public SLA announcement gated: cannot publish until PE.01 + PE.02 + PE.03 + PE.04 ship
- [ ] AI-Gateway-dependent endpoints exempted from SLA during KraftData incidents (existing Epic 4 circuit breaker pattern absorbs upstream failures)
- [ ] All HTTP outbound from new infra workstream uses two-layer resilience pattern (Rule 47) — no exceptions in PE stories

## Stories

### PE.01: k6 Baseline Closure (Carries Forward from Epic 13 — CRITICAL PATH)
**Points**: 5 | **Type**: platform-engineering | **Critical path; first PE story**

Close the k6 performance baseline that has been carry-forward for 6 epics (E03 → E05 → E06 → E07 → E08 → E13 per project-context anti-pattern). **Cannot declare any SLO without measured baseline.**

Scope:
- k6 scripts written, executed, and committed for: client-api search/list/detail endpoints, AI-Gateway run/run-stream endpoints, data-pipeline ingestion throughput, billing checkout flow
- Document p50/p95/p99 latency and throughput numbers in `eusolicit-docs/load-test-results.md` (the unfilled template that's been sitting deferred)
- Tests run on staging-equivalent environment; results reproducible
- 10K concurrent INCR test for Redis usage metering (Epic 8 carry-forward 8.8-PERF-001) included
- PostgreSQL FTS under 10K+ opportunities tested (Epic 6 carry-forward)
- SSE concurrency cap (10/pod) validated under load
- Results inform PE.02–PE.04 sizing decisions

**Tests:** k6 results checked into `eusolicit-docs/`; CI step that re-runs nightly against staging with regression alarm (>20% degradation triggers issue).

---

### PE.02: PostgreSQL HA Migration (Managed RDS Multi-AZ or Equivalent)
**Points**: 8 | **Type**: platform-engineering

Boring tech wins (Winston principle): migrate to managed PostgreSQL HA — RDS Multi-AZ (synchronous replica + automated failover) or Aurora PostgreSQL. Don't run Patroni in-cluster for a 2-3 person team.

Scope:
- Provision new managed PG instance with Multi-AZ in EU region
- Cutover plan: pg_dump + restore OR logical replication for zero-downtime
- Test cutover in staging; document rollback steps
- Production cutover during planned 30-minute maintenance window
- All services' DB connection strings updated via External Secrets Operator
- Per-service DB roles preserved (no role-permission churn)
- Backup retention: 35 days point-in-time
- Failover tested in staging post-cutover
- **REQUIRED migration `M_PE02_opportunities_tsv_gin_index`** — add a stored
  generated `tsv TSVECTOR` column on `pipeline.opportunities` plus a GIN
  index, and rewrite `client_api.services.opportunity_service._build_fts_condition`
  to query the stored column rather than building `to_tsvector()` at query
  time. **This is a hard prerequisite, not an optional optimization** —
  PE.01 (Story 21.1) captured `EXPLAIN ANALYZE` against the local 10K-row
  dataset and confirmed `Seq Scan on opportunities` with execution time
  289 ms; linear-scan extrapolation to 1M rows gives ~28 s p50 which would
  fail NFR-13 (`<20% degradation at 10K active companies / 1M opportunities`).
  See `eusolicit-docs/implementation-artifacts/load-test-results.md`
  §EXPLAIN ANALYZE Results (verbatim PostgreSQL 16.13 plan) and §Sizing
  Recommendations for PE.02 for the supporting evidence. Expected
  post-migration p95 at 1M rows: 30–80 ms (Bitmap Index Scan on the GIN
  index, top-N heapsort over matching ~0.5–2% of rows). Without this
  migration the 99.9% SLA cannot be published per AC-2.4 of Story 21.1.

**Tests:** Failover test in staging — verify all services reconnect within 30s; no data loss; transaction-rollback fixtures still work. SLI: PG availability target 99.95% (gives headroom for the platform 99.9% SLA). The migration's `EXPLAIN ANALYZE` evidence file MUST show `Bitmap Index Scan` (not `Seq Scan`) — this is the regression-test for the PE.01 deviation tracked here.

**Implementation:** Story 21-2 (`21-2-postgresql-ha-migration-managed-rds-multi-az-or-equivalent`) — done 2026-05-04. See `implementation-artifacts/pe-02-cutover-runbook.md` for cutover evidence and `implementation-artifacts/load-test-results.md` §EXPLAIN ANALYZE Results — Post-PE.02 Migration for the FTS plan flip (Seq Scan → Bitmap Index Scan on ix_opportunities_tsv). Terraform module `infra/terraform/modules/database/` fully implemented (aws_db_instance Multi-AZ + 35d PITR + Performance Insights + CloudWatch logs + KMS encryption + parameter group with pg_stat_statements). Migration M_PE02_opportunities_tsv_gin_index shipped as data-pipeline rev 003. ESO ExternalSecret CRDs added to Helm chart. Staged failover drill and production cutover pending operator execution (D-1 pre-recorded deviation).

---

### PE.03: Redis HA Migration (Sentinel or Managed Cluster)
**Points**: 5 | **Type**: platform-engineering

Migrate Redis from single-instance to managed Redis HA — managed Redis Cluster (AWS ElastiCache for Redis) or Sentinel. Boring tech preferred.

Scope:
- Provision new managed Redis HA instance (1 primary + 2 replicas) in EU region
- Cutover plan: dual-write window then cutover; or planned 5-minute maintenance window for stop-the-world cutover (Redis state is ephemeral except event-stream offsets — those replicate)
- All services' Redis connection strings updated via External Secrets Operator
- Sentinel-aware redis-py client configuration
- Backup retention: 7 days
- Failover tested in staging post-cutover

**Tests:** Sentinel failover test — verify all services reconnect within 10s; event-stream consumer groups resume correctly; usage metering Lua scripts continue to function under failover.

**Implementation:** Story 21-3 (`21-3-redis-ha-migration-sentinel-or-managed-cluster`) — dev pass 2026-05-04; pending bmad-code-review Approve verdict (AP17-C1). Terraform `modules/redis/` fully activated: `aws_elasticache_replication_group` (cluster-mode-disabled, 1 primary + 2 replicas for prod, eu-central-1, cache.r6g.large, Multi-AZ + automatic failover, 7d snapshot retention, at-rest + transit encryption). Per-service Redis connection strings via ESO ExternalSecret (Shape A — extending `externalsecret.yaml`; `CELERY_BROKER_URL` + `CELERY_RESULT_BACKEND` exposed for data-pipeline + notification per AP-GUARD-6). All 15 `redis-py` `from_url` call sites hardened with `health_check_interval=30` + `socket_keepalive=True` + `retry=Retry(ExponentialBackoff(cap=10, base=1), 3)` + `retry_on_error=[ConnectionError, TimeoutError]`. Both Celery apps updated with `broker_connection_retry_on_startup=True` + `broker_transport_options` resilience keys. See `implementation-artifacts/pe-03-cutover-runbook.md` for cutover evidence + §Connection Audit + §ESO Wiring Decision. Failover drill results and staging rehearsal timing to be populated by operator (D-1/D-2 deviations pre-recorded).

---

### PE.04: PodDisruptionBudgets + Min-Replica Enforcement Across All Services
**Points**: 5 | **Type**: platform-engineering

Apply PDB + min-replica patterns to all 6 services (5 existing + new integrations-api).

Scope:
- Each service Helm chart updated with PodDisruptionBudget (`minAvailable: 1`)
- Production min-replica counts: client-api=3, admin-api=2, ai-gateway=2, data-pipeline-worker=2, notification-worker=2, integrations-api=2
- HPA configurations reviewed: max replicas appropriate to PE.01 baseline numbers
- nginx-ingress already 2-replica per typical Helm pattern; verify; add PDB if missing
- Network policies confirm HA failover paths
- Cluster auto-scaler configured to handle replica scale-out under load

**Tests:** Chaos test — drain a node hosting a replica of each service; verify continuity. Helm-chart-lint CI step rejects future services without PDB.

**PE.04 Implementation (Story 21-4 — 2026-05-05):** Dev pass complete. PDB `minAvailable: 1` enforced across all 6 production services. HPA `minReplicas` floors per epic line 105: admin-api 1→2, notification 1→2, integrations-api new at 2, client-api/ai-gateway/data-pipeline already at floors. NEW `infra/helm/values/integrations-api.yaml` created per ADR-009 (port 8007, HPA 2-4). CI lint gate `scripts/check_helm_pdb_and_minreplicas.py` wired into `.github/workflows/ci.yml` (every push + PR) — rejects future services without HA primitives (epic line 112). Chaos-drill runbook at `implementation-artifacts/pe-04-chaos-drill-runbook.md`; live staging drill deferred to operator per D-1. NetworkPolicy HA verification: all rules use label-selectors (no literal IPs). **SLA-publication gate PE.01+PE.02+PE.03+PE.04 = 4/4 complete** from code+config standpoint. PE.05/PE.06 non-gating per epic line 20.

---

### PE.05: SLO Dashboards (Prometheus + Grafana) + Error-Budget Alerting
**Points**: 8 | **Type**: platform-engineering | **Re-homes Epic 13 Prometheus /metrics carry-forward**

Bootstrap Prometheus /metrics endpoints across all services (Epic 13 carry-forward) and build Grafana SLO dashboards.

Scope:
- Every FastAPI service exposes `/metrics` Prometheus endpoint with: request count, request duration histogram (p50/p95/p99), in-flight requests, error rate by endpoint
- Celery workers expose worker-level metrics: task throughput, success rate, queue depth
- Redis metrics: connection count, command rate, evictions
- PostgreSQL metrics: connection pool, slow queries, replication lag
- Grafana dashboards per service + cross-cutting SLO dashboard with: availability (target 99.9%), latency p95, error rate, error-budget burn rate
- Alertmanager rules: page on-call when error-budget burn rate >2x target (would exhaust budget within 30 days)
- KraftData-dependent SLOs separated (per Epic 4 isolation pattern) so AI-Gateway outages don't trip platform SLA alerts

**Tests:** Synthetic load test triggers alert; alert routes to PagerDuty/Opsgenie test schedule; runbook link included in alert payload.

**Implementation:** Story 21-5 (`21-5-slo-dashboards-prometheus-grafana-error-budget-alerting`).
See `implementation-artifacts/pe-05-observability-runbook.md`. Re-homes Epic 13 Prometheus
/metrics carry-forward. Shared `eusolicit_common.observability` middleware wired into all 7 services;
Celery worker metrics (data-pipeline + notification); CloudWatch exporter (ElastiCache + RDS);
Grafana dashboards as JSON; Prometheus recording + alerting rules (AMP format); Alertmanager
routing config (PagerDuty + Slack via ESO secrets); AMP+AMG Terraform module activated.

---

### PE.06: On-Call Rotation + Runbook Authoring + Incident-Management Process
**Points**: 3 | **Type**: process

Operational process — not pure engineering, but gates the SLA promise.

Scope:
- PagerDuty (or Opsgenie) configured with on-call schedule (initially shared rotation; can split engineering/platform later)
- Top-10 runbooks authored covering: PG failover, Redis failover, KraftData outage (already partial from Epic 4), Stripe outage, ClamAV outage, ingress controller restart, full-disk on PG, OAuth provider outage (per Epic 9), bulk webhook replay, deploy rollback
- Incident-management process documented: severity definitions (SEV-1 / SEV-2 / SEV-3), response-time SLAs, post-mortem cadence
- Post-mortem template authored (blameless format per existing project-context Epic 13 retro patterns)
- First chaos-test exercise (drain a node, watch alerts/runbooks/response) executed and post-mortemed

**Acceptance:** First simulated incident post-mortemed. Runbooks verified by being followed during the chaos drill. On-call schedule active for ≥2 weeks before public SLA announcement.

---

## PE.06 Implementation Record (Story 21-6 Dev-Pass 2026-05-05)

**Status at dev-pass:** DEV-PASS — awaiting bmad-code-review Approve verdict (AP17-C1) + D-3 operator soak gate.

### Terraform Module Provisioned

- `infra/terraform/modules/oncall/` — PagerDuty `~> 3.0` provider; `pagerduty_user` × N, `pagerduty_schedule` (2 layers: primary weekly + backup offset 24h, Europe/Berlin), `pagerduty_escalation_policy` (3 levels: 5min → 10min → 30min CTO), `pagerduty_service` × 2 (`eusolicit-platform-prod` high-urgency + `eusolicit-test-burn-rate-alert` low-urgency), `pagerduty_service_integration` × 2 (events_api_v2). All resources: `count = var.enabled ? 1 : 0`.
- `infra/terraform/modules/oncall/aws-secrets.tf` — writes integration keys to `eusolicit/<env>/observability/pagerduty-key` + `eusolicit/<env>/observability/pagerduty-test-key`.
- `infra/terraform/environments/dev/terraform.tfvars` — `enable_oncall = false` (anti-pattern guard #1).
- `infra/terraform/environments/staging/terraform.tfvars` + `prod/terraform.tfvars` — `enable_oncall = true`.

### Runbooks Authored (12 new)

All at `eusolicit-docs/runbooks/`. Each has 6 mandatory sections + `**SLA-Scope**:` header.

| Runbook | SLA-Scope | Notes |
|---------|-----------|-------|
| `error-budget-burn.md` | in-scope | PE.05-reserved; fast-burn + slow-burn branches |
| `high-latency.md` | in-scope | PE.05-reserved; NFR-2 p95 violation |
| `rds-replica-lag.md` | in-scope | PE.05-reserved; Multi-AZ failover decision tree |
| `redis-evictions.md` | in-scope | PE.05-reserved; memory pressure + unbounded key growth |
| `kraftdata-outage.md` | EXEMPT (vendor outage) | PE.05-reserved; consolidates Epic 4 partial content |
| `pg-failover.md` | in-scope | Lifts PE.02 §Failover Drill Steps; ≤30s reconnect SLA |
| `redis-failover.md` | in-scope | Lifts PE.03 §Failover Drill Steps + §Lua-Script Re-Run; ≤10s |
| `node-drain.md` | in-scope | Lifts PE.04 §Drain Procedure + §Per-Service Drill |
| `stripe-outage.md` | EXEMPT (vendor outage) | Billing webhook backoff + idempotent handler |
| `clamav-outage.md` | EXEMPT (vendor outage) | Queue-vs-reject decision; `CLAMAV_FALLBACK_MODE` |
| `ingress-controller-restart.md` | in-scope | crashloop + planned restart + PDB blocking |
| `full-disk-on-pg.md` | in-scope | VACUUM FULL decision; storage auto-scaling; RDS emergency |
| `oauth-provider-outage.md` | EXEMPT (vendor outage) | JWT 24h grace; email-password fallback |
| `bulk-webhook-replay.md` | in-scope | Stripe events resend + Teams integrations-api retry |
| `deploy-rollback.md` | in-scope | `helm rollback`; migration-rollback gotcha; `alembic downgrade` |

### Incident-Management Docs (4 new files)

- `eusolicit-docs/incident-management/severity-definitions.md` — SEV-1/2/3 + response SLAs + SLA-scope table + Alertmanager label mapping.
- `eusolicit-docs/incident-management/incident-response-process.md` — 9-step IC-led flow.
- `eusolicit-docs/incident-management/post-mortem-template.md` — blameless; no "who" column; mirrors Epic 13 retro.
- `eusolicit-docs/incident-management/status-page-comms-templates.md` — SEV-1 Initial/Update/Resolved + EXEMPT vendor template.

### Post-Mortem Repository Established

- `eusolicit-docs/post-mortems/2026-MM-DD-pe-04-chaos-drill.md` — first entry; skeleton with YAML frontmatter (drill_rota for 6 services) + all sections with operator-capture placeholders + 2-week soak gate note. Operator renames + populates after live drill.

### CI Lint Gate

- `eusolicit-app/scripts/check_runbook_url_coverage.py` — scans `infra/observability/prometheus/rules/*.yaml`; asserts every `runbook_url` annotation resolves to a file with all 6 sections + `**SLA-Scope**:` header.
- `.github/workflows/ci.yml` — `runbook-url-coverage` job added after `helm-pdb-lint`; no `if:` condition (runs every PR — anti-pattern guard #19).
- **ATDD result**: 15/15 tests GREEN (`tests/unit/test_runbook_url_coverage.py`).
- **Lint gate result**: `✅ PE.06 runbook coverage lint PASSED — runbook coverage: 7/7 URLs resolve, 5/5 structural checks pass`.

### Evidence File

`eusolicit-docs/implementation-artifacts/pe-06-incident-readiness-runbook.md` — 7-section evidence file with Pre-flight checklist, Implementation Decision, On-Call Rotation Verification, Runbook Inventory + URL Coverage Audit, Incident Process Verification, Chaos-Drill Post-Mortem Reference, and 2-week soak gate.

### Deferred Operator Actions (D-3 gate)

- D-3a: `terraform apply` modules/oncall to staging → capture PagerDuty schedule screenshot → record in pe-06 runbook §On-Call Rotation Verification.
- D-3b: Execute PE.04 chaos drill against staging using authored runbooks → populate + sign `2026-MM-DD-pe-04-chaos-drill.md`.
- D-3c: 2-week soak gate — on-call rotation active ≥2 weeks + ≥1 real page received → record in pe-06 runbook §2-Week Soak Gate → public 99.9% SLA announcement unblocked.

---

## Amendments

### Amendment 2026-05-04 — Story 21.1 Pass-7 review-fix B10 (PE.02 Seq Scan dependency tracker)

PE.01 (Story 21.1) AC-2.4 demanded the FTS `EXPLAIN ANALYZE` plan show
`Bitmap Index Scan on ix_opportunities_tsv`. The captured plan against the
local 10K-row dataset (PostgreSQL 16.13) shows `Seq Scan on opportunities`
— the GIN index does not exist because the current implementation builds
`to_tsvector()` at query time from a runtime expression rather than from
a stored generated column. AC-2.4 explicitly states: *"if it shows Seq
Scan, the index is missing or the planner cost model is mis-tuned; HALT
and file an issue under PE.02 (sizing decision) before proceeding."*

This amendment is the formal tracker for that PE.02 dependency. The PE.02
scope above has been extended with a new bullet ("REQUIRED migration
`M_PE02_opportunities_tsv_gin_index`") plus a new test requirement that
PE.02's `EXPLAIN ANALYZE` evidence file MUST show `Bitmap Index Scan` —
this is the regression test that closes the PE.01 deviation.

**Evidence reference**: `eusolicit-docs/implementation-artifacts/load-test-results.md`
§EXPLAIN ANALYZE Results (verbatim PostgreSQL 16.13 plan, 10K rows local
docker-compose, 2026-05-04) and §Sizing Recommendations for PE.02–PE.04.

**Sprint-status reconciliation**: PE.02's row in `sprint-status.yaml`
should pick up the migration name once the story is created — no
sprint-status edit required as part of this amendment (the requirement
is now in the epic file which PE.02's `bmad-create-story` workflow reads
as the source of truth).
