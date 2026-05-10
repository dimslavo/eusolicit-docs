# Observability Stack Runbook (Story onprem-03)

**Story:** `eusolicit-docs/implementation-artifacts/onprem-03-monitoring-on-www1.md`
**Stack:** Prometheus + Grafana + Alertmanager + node-exporter + cadvisor + postgres-exporter + redis-exporter, all docker containers on www1.
**Compose file:** `eusolicit-app/infra/observability/docker-compose.observability.yml`

## Decision: standalone stack vs co-tenant reuse

**Chosen:** standalone. We do **not** reuse the existing `lifematch-dev-*` Grafana/Prom on www1. Reasoning: independent restart cycles, no cross-tenant blast radius, clearer ownership for EU Solicit on-call.

Cost: ~600 MB additional RAM on www1 (Prometheus + Grafana baseline; manageable given 21 GB free).

## Bring up

```bash
# 1. Operator sets up secrets (one time)
cp /home/debian/Projects/eusolicit/eusolicit-app/infra/host/eusolicit-overrides/.env.alertmanager.example \
   /home/debian/eusolicit-overrides/.env.alertmanager
chmod 600 /home/debian/eusolicit-overrides/.env.alertmanager
# edit to fill in SMTP, Telegram, Slack values

mkdir -p /home/debian/eusolicit-overrides/secrets
echo 'YOUR_SMTP_PASSWORD'   > /home/debian/eusolicit-overrides/secrets/smtp_password
echo 'YOUR_SLACK_WEBHOOK'   > /home/debian/eusolicit-overrides/secrets/slack_webhook
chmod 600 /home/debian/eusolicit-overrides/secrets/*

# 2. Provision the monitoring_role in postgres (run once)
docker exec eusolicit-app-postgres-1 psql -U eusolicit -d eusolicit -c \
  "CREATE ROLE monitoring_role LOGIN PASSWORD 'CHANGE_ME';
   GRANT pg_monitor TO monitoring_role;
   GRANT CONNECT ON DATABASE eusolicit TO monitoring_role;"

# 3. Start the stack
cd /home/debian/Projects/eusolicit/eusolicit-app/infra/observability
docker compose -f docker-compose.observability.yml --env-file /home/debian/eusolicit-overrides/.env.alertmanager up -d

# 4. Verify
curl -sf http://127.0.0.1:19090/-/healthy   # prometheus
curl -sf http://127.0.0.1:19093/-/healthy   # alertmanager
curl -sf http://127.0.0.1:13030/api/health  # grafana
```

## Reverse-proxy reach (nginx on www1)

Grafana should be reachable at `grafana.internal.eusolicit.com` (private). Add to `/etc/nginx/sites-enabled/eusolicit.com` (or new site file):

```nginx
server {
    listen 443 ssl;
    server_name grafana.internal.eusolicit.com;
    # cert via certbot --nginx
    location / {
        # Basic-auth in front of grafana for defence in depth
        auth_basic "EU Solicit Observability";
        auth_basic_user_file /etc/nginx/.htpasswd-grafana;
        proxy_pass http://127.0.0.1:13030;
        include snippets/proxy-params.conf;
    }
}
```

## Verifying scrape targets

```bash
curl -s http://127.0.0.1:19090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

All 7 app services + node + cadvisor + postgres + redis should report `health: "up"`. If any is `down`, check that container is on the shared `eusolicit-app_default` docker network.

## Tear down

```bash
cd /home/debian/Projects/eusolicit/eusolicit-app/infra/observability
docker compose -f docker-compose.observability.yml down
# data preserved in named volumes (prometheus-data, grafana-data, alertmanager-data)
# add --volumes to delete data.
```

## References

- Story: `onprem-03-monitoring-on-www1.md`
- Prometheus scrape config: `infra/observability/prometheus/prometheus.yml`
- Alertmanager config: `infra/observability/alertmanager/alertmanager.yaml`
- Grafana dashboards (loaded verbatim from Story 21-5): `infra/observability/grafana/dashboards/`
- Host alerts (Story onprem-04): `infra/observability/prometheus/rules/host-alerts.yaml`
