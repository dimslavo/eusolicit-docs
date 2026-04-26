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

**Tests:** Failover test in staging — verify all services reconnect within 30s; no data loss; transaction-rollback fixtures still work. SLI: PG availability target 99.95% (gives headroom for the platform 99.9% SLA).

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
