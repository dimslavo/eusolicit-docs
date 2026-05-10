# Story onprem-03: Monitoring on www1

Status: ready-for-dev

**Origin:** Onprem pivot decision (eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md). Adapts Story 21-5 (PE.05) from AMP/AMG to on-host Prometheus + Grafana + Alertmanager.
**Priority:** P0 — launch-blocker. Without monitoring, the RTO ≤ 4h commitment is undefendable; without alertmanager, `onprem-04` and `pe-04` have no destination.
**Epic:** Post-Epic-21 / Onprem Launch
**Points:** 2 | **Type:** infra / observability

## Story

As a **platform-engineering operator**,
I want **Prometheus + Grafana + Alertmanager running as docker containers on www1, scraping the per-service `/metrics` endpoints already wired by Story 21-5 and consuming the Grafana dashboard JSONs + Prometheus rules already in the repo**,
so that **we have a working observability surface for launch, without depending on AMP/AMG or any AWS service, and the existing PE.05 work is preserved rather than discarded**.

## Acceptance Criteria

1. **Implementation choice documented at Task 0** — pick one of:
   - **(a) Reuse the co-tenant `lifematch-dev-*` observability stack** by adding EU Solicit scrape targets + dashboards into the existing Grafana/Prometheus running on www1 (verified live: `lifematch-dev-{grafana,prometheus,loki,tempo,alertmanager}` are healthy and up 10+ minutes per current `docker ps`). Pros: no new containers, lowest disk/RAM cost, can leverage existing grafana auth. Cons: blast-radius mixing with another tenant.
   - **(b) Deploy a separate `eusolicit-observability` docker-compose stack** under a dedicated docker network. Pros: isolation, ownership clarity. Cons: another stack to maintain.
   The choice is documented in `eusolicit-docs/runbooks/observability-stack.md` with the trade-off rationale.

2. **Prometheus scrape config covers all 7 EU Solicit services** — `client-api` (port 18001), `admin-api` (18002), `data-pipeline` (18003), `ai-gateway` (18004), `notification` (18005), `integrations-api` (18007), plus `frontend` (13000 — Next.js metrics if any). Job names match the existing Grafana dashboard variable assumptions (`job="client-api"`, etc.).

3. **Grafana dashboard JSONs loaded verbatim** — the 7 per-service + 1 cross-cutting `platform-slo.json` files at `infra/observability/grafana/dashboards/` are mounted into Grafana via `grafana-dashboards.yaml` provisioning config (already in repo from Story 21-5). Dashboard variables resolved against the new on-host Prometheus datasource.

4. **Prometheus rules loaded verbatim** — the rule files at `infra/observability/prometheus/rules/` are mounted into the Prometheus container. The `slo_target` label routing (`"platform"` vs `"kraftdata-dependent"`) keeps working — KraftData-dependent SLO violations still don't page (per architecture.md line 762 carry-forward).

5. **Postgres + Redis exporters as docker containers** —
   - `prom/postgres-exporter` scrapes `eusolicit-app-postgres-1` via `pg_stat_statements` + standard metrics. Auth as `monitoring_role` (per Story 21-5).
   - `oliver006/redis_exporter` scrapes `eusolicit-app-redis-1`.
   Replaces the AWS CloudWatch exporter approach from Story 21-5.

6. **Alertmanager configured** — `infra/observability/alertmanager/alertmanager.yaml` (already in repo from Story 21-5) is loaded, with two routing changes vs. the original:
   - Remove `pagerduty` receivers (PagerDuty cancelled in pivot).
   - Add an `email` receiver + a `telegram-bot` webhook receiver (per `pe-06` decision).
   - The `slack` receiver for tickets is kept if a Slack workspace is in use.

7. **Reachable via reverse proxy** — Grafana served behind the existing nginx on www1 at `grafana.eusolicit.internal` (or similar private subdomain), with basic-auth or SSO behind it. Not exposed on the public internet.

8. **`/metrics` endpoint contract regression test passes** — the existing test at `tests/unit/test_metrics_endpoint_contract.py` (146 pass / 18 skip / 0 fail per Story 21-5) continues to pass after this change. No application code changes expected; this AC is a guardrail.

## Tasks / Subtasks

- [ ] Task 0: Decide reuse vs separate stack (AC 1). Document in observability-stack.md.
- [ ] Task 1: Author `infra/observability/prometheus/prometheus.yml` scrape config.
- [ ] Task 2: Provision Prometheus + Grafana + Alertmanager + postgres-exporter + redis-exporter via docker-compose (under `infra/observability/docker-compose.observability.yml` if separate, or extend the existing co-tenant stack if reuse).
- [ ] Task 3: Mount existing Grafana dashboard JSONs (verify variable substitution works against new datasource).
- [ ] Task 4: Mount existing Prometheus rule files.
- [ ] Task 5: Update `alertmanager.yaml`: remove PagerDuty, add email + Telegram-bot (depends partly on `pe-06`).
- [ ] Task 6: nginx reverse proxy + auth for Grafana.
- [ ] Task 7: Verify all 7 service jobs are `up=1` in Prometheus and dashboards render.
- [ ] Task 8: Run the existing `/metrics` contract test.

## Dev Notes

### What's already in code (retained from Story 21-5)

- `eusolicit_common.observability` middleware wires the per-service `/metrics` endpoint with HTTP histograms including `slo_target` label.
- `PIPELINE_METRICS_REGISTRY` (data-pipeline) and `METRICS_REGISTRY` (integrations-api) preserved verbatim with HTTP histograms added additively.
- Celery worker metrics via `metrics_signals.py` modules in data-pipeline + notification.
- Beat task queue-depth poll every 30s.
- All 7 Grafana dashboard JSONs committed.
- Prometheus rule files in plain AMP format (also valid for self-hosted Prometheus).

### Co-tenant observability stack on www1 (verified live)

Per current `docker ps`:
```
lifematch-dev-grafana-1       Up 10 minutes (healthy)
lifematch-dev-prometheus-1    Up 10 minutes (healthy)
lifematch-dev-loki-1          Up 10 minutes (healthy)
lifematch-dev-tempo-1         Up 10 minutes (healthy)
lifematch-dev-alertmanager-1  Up 10 minutes (healthy)
```

If reuse is chosen, coordinate scrape-target additions with the lifematch team. Document the boundary clearly (separate Prometheus jobs, separate Grafana folders/orgs, separate Alertmanager routing).

### Telegram bot webhook setup (cross-references `pe-06`)

Alertmanager Telegram receivers use a webhook to a small bot relay. Telegram Bot API is free; the relay can be a tiny Python/Go service or a public alertmanager-to-telegram bridge like `metalmatze/alertmanager-bot`. Decide at impl time.

### References

- ADR-010 rewrite: `architecture.md` §ADR-010 (2026-05-11)
- Decision record: `onprem-pivot-decision-2026-05-11.md`
- Story 21-5 deliverables (retained): `eusolicit-app/infra/observability/`
- Existing co-tenant stack: `docker ps | grep lifematch-dev`
- pe-04 chaos drill: depends on this story for alert routing
- pe-06: depends on this story for alertmanager receivers

### Out of scope

- Loki for logs (deferred — current docker logs + journald is sufficient at launch).
- Tempo for tracing (deferred — application doesn't emit traces yet).
- Long-term metrics storage / Thanos (deferred — Prometheus local retention sufficient for now).
