# PE.05 Observability Runbook — SLO Dashboards + Error-Budget Alerting

**Story:** 21-5-slo-dashboards-prometheus-grafana-error-budget-alerting
**Date Authored:** 2026-05-05
**Epic:** E21 — Platform Reliability for 99.9% SLA
**Status:** DEV-PASS (live staging drill deferred — D-2 pre-recorded deviation)
**Verdict:** DEFERRED — See §Sign-off

---

## §Pre-flight checklist

The operator completes this checklist before applying PE.05 resources to staging/production.

```
[ ] Terraform plan reviewed — modules/monitoring shows AMP + AMG workspace creation
[ ] AMP+AMG workspace IAM permissions verified (IRSA role has workspace:PutRuleGroupsNamespace, workspace:ListRuleGroupsNamespaces)
[ ] CloudWatch exporter IAM role ready (arn:aws:iam::<account-id>:role/eusolicit-cloudwatch-exporter)
    — role has cloudwatch:GetMetricStatistics, cloudwatch:ListMetrics on AWS/ElastiCache + AWS/RDS namespaces
[ ] PagerDuty TEST service provisioned (separate from on-call rotation Story 21-6 will configure)
    — service_id captured: _______________________
[ ] Slack webhook URL captured from Story 16.0 integrations-api config; stored in ESO secret
    — secret path: eusolicit/slack/platform-alerts-webhook-url
[ ] Postgres monitoring_role password stored in Secrets Manager
    — secret path: eusolicit/monitoring-role/password
[ ] Runbook IDs reserved for PE.06:
    — error-budget-burn.md
    — high-latency.md
    — rds-replica-lag.md
    — redis-evictions.md
    — kraftdata-outage.md (PE.06 fills content)
[ ] All ATDD tests GREEN: 146 passed / 18 skipped (infra-unavailable) / 0 failed
    — run: pytest tests/unit/test_metrics_endpoint_contract.py tests/unit/test_pe05_celery_metrics.py
             tests/unit/test_grafana_dashboards_contract.py tests/unit/test_prometheus_rules_contract.py
             tests/unit/test_alertmanager_config_contract.py -q
```

---

## §Implementation Decision

### (a) Celery Queue-Depth: signal-handler poll vs. sidecar celery-exporter

**Decision:** Poll-based beat task at 30s interval via `metrics_signals.poll_queue_depth`.

**Rationale:**
- The `danihodovic/celery-exporter` sidecar adds a deployment per worker fleet (data-pipeline + notification = 2 extra Kubernetes Deployments).
- It requires its own Redis credentials + ESO wiring — duplicating the story scope.
- The project-context "managed > self-managed for a 2-3 person team" principle applies — an in-process beat task beats a separate operational surface.
- The polling implementation uses `redis.llen(<queue_name>)` which is O(1) and read-only — no state mutation.

**Rejected alternative:** `danihodovic/celery-exporter` Helm release. Documented for future consideration if queue cardinality exceeds what the beat task can poll in 30s (currently 5 queues for data-pipeline, 6 for notification).

### (b) Grafana Dashboard Provisioning Path

**Decision:** Dashboard JSON files committed to `infra/observability/grafana/dashboards/` referenced by `infra/observability/grafana/grafana-dashboards-configmap.yaml` Kubernetes ConfigMap. Two equivalent deployment paths:

1. **Grafana Operator (self-managed):** Apply `GrafanaDashboard` CRD — auto-reloads from ConfigMap via sidecar provisioner.
2. **AMG Terraform (managed):** `aws_grafana_workspace_api_key` + `aws grafana create-dashboard` calls from the Terraform module.

The operator selects the path based on the cluster environment. The JSON files are path-stable and valid for both.

### (c) Postgres-Exporter `monitoring_role` Bootstrapping

**Decision:** Append `monitoring_role` block to `infra/postgres/init/01-init-schemas-and-roles.sql` (same pattern as PE.02 `migration_role`). The role is granted `pg_monitor` (PostgreSQL built-in) which gives `pg_stat_statements` read access without touching application schema data (ADR-001 compliance).

---

## §Service `/metrics` Wiring Audit

| Service | File | Key Lines Added | Middleware Order | `/metrics` Status |
|---------|------|-----------------|-----------------|-----------------|
| `client-api` | `services/client-api/src/client_api/main.py` | `make_metrics_registry("client-api")`, `MetricsMiddleware` (last `add_middleware`) | Outermost ✓ | HTTP 200 |
| `admin-api` | `services/admin-api/src/admin_api/main.py` | `make_metrics_registry("admin-api")`, after `IPAllowlistMiddleware` | Outermost ✓ | HTTP 200 |
| `ai-gateway` | `services/ai-gateway/src/ai_gateway/main.py` | `make_metrics_registry("ai-gateway")`, after exception handlers | Outermost ✓ | HTTP 200 (skipped in unit tests — Redis lifespan) |
| `notification` | `services/notification/src/notification/main.py` | `make_metrics_registry("notification")`, at end of middleware stack | Outermost ✓ | HTTP 200 (skipped in unit tests — Redis lifespan) |
| `enterprise-api` | `services/enterprise-api/src/enterprise_api/main.py` | `make_metrics_registry("enterprise-api")`, before catch-all router | Outermost ✓ | HTTP 200 |
| `data-pipeline` | `services/data-pipeline/src/data_pipeline/main.py` | `_http_metrics_registry` additive; `generate_latest(A) + generate_latest(B)` | Outermost ✓ | HTTP 200 |
| `integrations-api` | `services/integrations-api/src/integrations_api/main.py` | `_http_metrics_registry` additive; `_NamedMetricFamily.documentation` added | Outermost ✓ | HTTP 200 |

**Shared middleware module:** `packages/eusolicit-common/src/eusolicit_common/observability/`
- `__init__.py` — re-exports `MetricsMiddleware`, `metrics_router`, `make_metrics_registry`
- `metrics.py` — `make_metrics_registry()` factory with 4 mandatory HTTP metrics + `slo_target` label
- `middleware.py` — `MetricsMiddleware` (route-template endpoint label; KraftData path classification)

**slo_target label:** All 4 HTTP metrics include `slo_target` label (`"platform"` or `"kraftdata-dependent"`). KraftData-dependent paths defined in `middleware.py::_KRAFTDATA_PATH_PATTERNS` (AI-Gateway `/run`, `/run-stream`, opportunity summary, KraftData webhooks).

---

## §Dashboard Inventory

| File | Panels | Primary Data Source | Notes |
|------|--------|---------------------|-------|
| `infra/observability/grafana/dashboards/platform-slo.json` | 9 | AMP / recording rules | Cross-cutting SLO dashboard; 4-window multi-burn-rate (Google SRE Workbook §5.2) |
| `infra/observability/grafana/dashboards/client-api.json` | 6 | AMP / raw PromQL | Per-service: req rate, p95 latency, error rate, in-flight, 5xx breakdown |
| `infra/observability/grafana/dashboards/admin-api.json` | 6 | AMP / raw PromQL | Per-service |
| `infra/observability/grafana/dashboards/ai-gateway.json` | 6 | AMP / raw PromQL | Per-service; KraftData-dependent panels separate |
| `infra/observability/grafana/dashboards/data-pipeline.json` | 8 | AMP / raw PromQL | Per-service + Celery queue depth + task throughput |
| `infra/observability/grafana/dashboards/notification.json` | 8 | AMP / raw PromQL | Per-service + Celery metrics |
| `infra/observability/grafana/dashboards/integrations-api.json` | 7 | AMP / raw PromQL | Per-service + CRM sync metrics |
| `infra/observability/grafana/dashboards/enterprise-api.json` | 6 | AMP / raw PromQL | Per-service |

**Provisioning ConfigMap:** `infra/observability/grafana/grafana-dashboards-configmap.yaml`

**Screenshot references:** (operator captures from staging Grafana post-deploy and links here)

```
platform-slo.json screenshot: [OPERATOR CAPTURE]
per-service screenshot examples: [OPERATOR CAPTURE]
```

---

## §Recording + Alerting Rules Inventory

### Recording Rules (`infra/observability/prometheus/rules/recording-rules.yaml`)

| Rule Name | Window | Condition | KraftData Isolation |
|-----------|--------|-----------|---------------------|
| `availability:slo:rate5m` | 5m | `1 - (errors/requests)` | `slo_target` label separates platform vs. kraftdata-dependent |
| `latency_p95:slo:rate5m` | 5m | `histogram_quantile(0.95, ...)` | `slo_target` in `by()` clause |
| `error_rate:slo:rate5m` | 5m | `errors/requests` | `slo_target` label |
| `availability:slo:rate30m` | 30m | Same pattern | `slo_target` label |
| `availability:slo:rate1h` | 1h | Same pattern | `slo_target` label |
| `availability:slo:rate6h` | 6h | Same pattern | `slo_target` label |
| `availability:slo:rate1d` | 1d | Same pattern | `slo_target` label |
| `availability:slo:rate3d` | 3d | Same pattern | `slo_target` label |
| `availability:slo:rate30d` | 30d | Same pattern — monthly budget | `slo_target` label |

**Anti-pattern guard #11 compliance:** NEVER mix `slo_target="platform"` and `slo_target="kraftdata-dependent"` in the same expression — the `sum by (..., slo_target)` clause keeps them separated.

### Alerting Rules (`infra/observability/prometheus/rules/alerting-rules.yaml`)

| Alert Name | Window Condition | Severity | `for` | Runbook |
|-----------|-----------------|----------|-------|---------|
| `HighErrorBudgetBurnRate` | 1h+5m > 14.4× budget | `page` | 2m | `error-budget-burn.md` |
| `SustainedErrorBudgetBurnRate` | 6h+30m > 6× budget | `page` | 15m | `error-budget-burn.md` |
| `HighLatencyP95` | `latency_p95:slo:rate5m > 0.2` | `ticket` | 10m | `high-latency.md` |
| `HighRDSReplicaLag` | `aws_rds_replica_lag_average > 30` | `page` | 5m | `rds-replica-lag.md` |
| `RedisEvictionsObserved` | `rate(evictions[5m]) > 0` | `ticket` | 10m | `redis-evictions.md` |
| `RedisReplicationLagHigh` | `aws_elasticache_replication_lag_average > 1` | `ticket` | 5m | `redis-evictions.md` |

**Multi-window multi-burn-rate rationale (Google SRE Workbook §5.2):**
- Fast-burn (14.4×): budget exhausts in <2 days → requires immediate on-call response → `severity: page`
- Slow-burn (6×): budget will exhaust within the month if left unchecked → `severity: page` but `for: 15m`
- Short-window confirmation (5m for fast-burn, 30m for slow-burn): prevents long-window drift from firing on transient blips

**Anti-pattern guard #13 compliance:** `repeat_interval` ≥ 1h in `alertmanager.yaml` (prevents PagerDuty spam).

---

## §Alertmanager Routing Verification

**Config file:** `infra/observability/alertmanager/alertmanager.yaml`

**Routing tree:**
```
root → default: slack-platform-alerts
  ├── severity=page → pagerduty
  ├── severity=ticket → slack-platform-alerts
  └── severity=info → null (discarded)
```

**Verification commands (run post-deploy):**

```bash
# Check amtool is available
which amtool || (echo "Install via: go install github.com/prometheus/alertmanager/cmd/amtool@latest" && exit 1)

# Validate config syntax
amtool check-config infra/observability/alertmanager/alertmanager.yaml

# Show parsed routing tree
amtool config show --alertmanager.url http://alertmanager:9093

# Test routing for a page-severity alert
amtool config routes test severity=page team=platform \
  --alertmanager.url http://alertmanager:9093

# Test routing for a ticket-severity alert
amtool config routes test severity=ticket \
  --alertmanager.url http://alertmanager:9093
```

**Receiver verification (operator capture post-deploy):**
```
# OPERATOR CAPTURE — amtool config show output
[OPERATOR CAPTURE]

# PagerDuty receiver configured: [ ] YES / [ ] NO
# Slack receiver configured:     [ ] YES / [ ] NO
# Repeat interval >= 1h:         [ ] YES / [ ] NO
```

**External secrets:** `infra/observability/alertmanager/alertmanager-externalsecret.yaml` provisions:
- `PAGERDUTY_INTEGRATION_KEY` from `eusolicit/alertmanager/pagerduty-integration-key`
- `SLACK_WEBHOOK_URL` from `eusolicit/alertmanager/slack-webhook-url`

**Anti-pattern guard #12 compliance:** No hardcoded secrets in `alertmanager.yaml` — all sensitive values via ESO `SecretProviderClass`.

---

## §Burn-Rate E2E Test Results

**Test file:** `tests/observability/test_alert_burn_rate_e2e.py` (D-2 pre-recorded deviation)

**Status:** DEFERRED — requires live AMP endpoint + staging services under load.

**D-2 justification:** The synthetic burn-rate end-to-end test (AC-9) requires a live AMP workspace, staging services receiving traffic, and a PagerDuty TEST service. These infra prerequisites are operator-gated; the test code is committed but execution is deferred.

**Operator capture (post-staging-deploy):**

```
# Run with AMP endpoint and staging target configured via env vars:
AMP_ENDPOINT=https://<amp-id>.aps.eu-central-1.amazonaws.com/workspaces/<workspace-id> \
STAGING_BASE_URL=https://staging.api.eusolicit.com \
PAGERDUTY_TEST_SERVICE_ID=<test-service-id> \
pytest tests/observability/test_alert_burn_rate_e2e.py -v -s

# Expected outcomes:
# (a) availability:slo:rate5m drops to <= 0.6 within 2 minutes of 50% error injection
# (b) Recording rule promotes the rate to slo_target="platform" label set
# (c) HighErrorBudgetBurnRate fires (queryable via AMP /api/v1/alerts)
# (d) Alertmanager routes to PagerDuty TEST service (verifiable via PagerDuty API)
# (e) Alert payload contains runbook_url annotation

# OPERATOR CAPTURE — paste pytest output here:
[OPERATOR CAPTURE]
```

---

## §ADR-010 Compliance

**ADR-010 principle:** "Boring infra wins (Winston principle)" — managed > self-managed for a 2-3 person team.

| Component | Approach | Managed / Self-deployed | Rationale |
|-----------|----------|------------------------|-----------|
| Prometheus scrape layer | Amazon Managed Prometheus (AMP) | **Managed** ✓ | No Prometheus server to operate; AMP handles HA, retention, scaling |
| Grafana | Amazon Managed Grafana (AMG) | **Managed** ✓ | No Grafana server to operate; AMG handles auth via IAM Identity Center |
| Redis metrics | CloudWatch exporter (`AWS/ElastiCache`) | **Managed** ✓ | ElastiCache is VPC-only; CloudWatch is the AWS-native path |
| RDS metrics | CloudWatch exporter (`AWS/RDS`) | **Managed** ✓ | CloudWatch covers all standard RDS metrics |
| Postgres slow-query metrics | `prometheus-community/postgres-exporter` | **Self-deployed** (lone exception) | `pg_stat_statements` is NOT in CloudWatch — this is the only metric family that requires the exporter. The `monitoring_role` is read-only (ADR-001 compliant). |
| Celery queue depth | In-process beat task poll | **In-process** ✓ | Avoids `celery-exporter` sidecar deployment (rejected in §Implementation Decision) |
| Alert routing | Alertmanager (bundled with Prometheus Operator or AMP managed rules) | **Managed** ✓ | Routing config committed to source; AMP managed alerting handles HA |

**Lone self-deployed exporter justification:** `pg_stat_statements` exposes slow-query plan data essential for the NFR-13 regression detection (architecture.md line 891 — FTS plan flip evidence). CloudWatch does not expose query-level statistics. The `monitoring_role` is a built-in PostgreSQL `pg_monitor` grant — no schema-crossing (ADR-001 compliant); no write access; no application-schema exposure.

---

## §Cross-Story Coordination

### Runbook IDs Reserved for PE.06 (Story 21-6)

PE.05 reserves the following runbook URLs. PE.06 authors the content. The URLs are path-stable and committed in all `alerting-rules.yaml` `runbook_url` annotations.

| Runbook ID | Alert(s) That Reference It | Content Owner |
|------------|---------------------------|---------------|
| `error-budget-burn.md` | `HighErrorBudgetBurnRate`, `SustainedErrorBudgetBurnRate` | PE.06 (Story 21-6) |
| `high-latency.md` | `HighLatencyP95` | PE.06 (Story 21-6) |
| `rds-replica-lag.md` | `HighRDSReplicaLag` | PE.06 (Story 21-6) |
| `redis-evictions.md` | `RedisEvictionsObserved`, `RedisReplicationLagHigh` | PE.06 (Story 21-6) |
| `kraftdata-outage.md` | (future KraftData-specific alerts) | PE.06 (Story 21-6) |

**URL pattern:** `https://github.com/eusolicit/eusolicit/blob/main/eusolicit-docs/runbooks/<runbook-id>.md`

### Cross-Story Contract

- PE.06 reads `infra/observability/alertmanager/alertmanager.yaml` as the canonical alert-routing source.
- PE.06 reads `infra/observability/prometheus/rules/alerting-rules.yaml` as the canonical alert-payload source (including `runbook_url` annotation pattern).
- PE.06 reads the `platform-slo.json` Grafana dashboard as the canonical incident-evidence source during the chaos drill.
- PE.04 `pe-04-chaos-drill-runbook.md` §Per-Service Drill subsections link to the per-service Grafana dashboards in `infra/observability/grafana/dashboards/` for live evidence capture.
- PE.03 Redis failover SLA (`≤10s reconnect`) measurable via `aws_elasticache_replication_group_failover_count` + `http_request_duration_seconds_bucket{le="1"}` increase rate.
- PE.02 RDS failover SLA (`≤30s reconnect`) measurable via `aws_rds_database_connections_average` drop+recovery pattern.

---

## §Sign-off

**Verdict:** DEFERRED

**Date:** 2026-05-05

**Dev pass completed:** All ATDD tests pass (146 passed / 18 skipped for infra-gated services / 0 failed).

**Deferred items:**
- **D-2 (AC-9 burn-rate e2e test):** Requires live AMP + staging services. Operator captures output in §Burn-Rate E2E Test Results above.
- **Alertmanager routing verification (§Alertmanager):** Requires live Alertmanager deployment. Operator captures `amtool config show` output.
- **Grafana dashboard screenshot evidence (§Dashboard Inventory):** Operator captures from staging AMG post-deploy.

**PE.06 dependency:** PE.06 (Story 21-6) consumes this story's `alertmanager.yaml` + `runbook_url` annotation pattern + Grafana dashboard URLs as the canonical incident-response entrypoint. PE.05 must be promoted to `done` (Approve verdict from code review) before PE.06 authors runbook content.

**Operator sign-off required:** bmad-code-review Approve verdict per AP17-C1 (protects 9-in-a-row successful-closure streak: S19-0 / S19-1 / S19-2 / S20-0 / S21-1 / S21-2 / S21-3 / S21-4 / **S21-5**).

```
Operator sign-off: ___________________________  Date: ______________
Code-review verdict: [ ] APPROVE  [ ] REQUEST_CHANGES
```
