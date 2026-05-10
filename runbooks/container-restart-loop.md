# Runbook: Container Restart Loop

**Trigger:** `ContainerRestartLoop` (>3 restarts of an `eusolicit-app-*` container in 5 min)
**Story:** onprem-04 | **SLO:** platform

## Symptoms

A service container is repeatedly crashing + being restarted by `restart: unless-stopped`. End-user requests likely failing intermittently.

## Triage

```bash
# Which container?
docker ps --filter "name=eusolicit-app-" --format "table {{.Names}}\t{{.Status}}"

# Recent restart count (last hour)
docker events --since 1h --filter "type=container" --filter "event=start" --filter "name=eusolicit-app-" 2>&1 | tail -20

# Crash logs from the troubled container
SVC=<container-name>
docker logs --tail=100 "$SVC" 2>&1 | tail -80

# Healthcheck status
docker inspect --format='{{json .State.Health}}' "$SVC" | jq .
```

## Common causes + fixes

1. **Bad recent deploy** — check `git log --oneline -5` for the most recent commit. If a `chore: auto-sync` or fresh feature commit just landed, **roll back**:
   ```bash
   cd /home/debian/Projects/eusolicit/eusolicit-app
   git log --oneline -5
   git reset --hard <previous-known-good-sha>
   bash scripts/deploy.sh --rollback
   ```

2. **Database connection refused** — postgres might be down or password mismatch. Check `docker logs eusolicit-app-postgres-1` and `.env.prod` for credential drift.

3. **OOM kill** — see `runbooks/memory-pressure.md`. The container's memory limit may be too tight; raise in `docker-compose.prod.yml`.

4. **Bad config / migration** — alembic migration failure on cold start. See `deploy.log` for `Migration failed for <svc>` errors.

5. **Disk full** — see `runbooks/disk-cleanup-root.md`. Out-of-disk causes mystery crashes.

## Escalation

If restarts continue after triage:
- Increment severity to S1 if user-facing service is the affected container (`client-api`, `frontend`).
- Open a post-mortem in `eusolicit-docs/post-mortems/`.
- If trigger commit is `chore: auto-sync`, append a row to `incident-management/auto-sync-incidents.md`.

## References

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
- Project memory: "Auto-sync ships unverified code"
