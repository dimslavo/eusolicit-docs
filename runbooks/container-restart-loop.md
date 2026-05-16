# Runbook: Container Restart Loop

**Trigger:** `ContainerRestartLoop` (>3 restarts of an `eusolicit-app-*` container in 5 min)
**Story:** onprem-04 | **SLO:** platform
**SLA-Scope**: in-scope

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

## Resolution

Identify the cause and apply the matching fix:

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

## Verification

```bash
# No new restarts; container Up and healthy
docker ps --filter "name=eusolicit-app-" --format "table {{.Names}}\t{{.Status}}"
docker inspect --format='{{json .State.Health}}' "$SVC" | jq .

# Confirm no further start events for this container
docker events --since 10m --filter "type=container" --filter "event=start" --filter "name=$SVC" 2>&1 | tail -5
```

`ContainerRestartLoop` clears once the container stays up for the alert window with no further restarts and the healthcheck reports `healthy`.

## Rollback

For cause #1 the Resolution step **is** a rollback (`git reset --hard` + `scripts/deploy.sh --rollback`). If the rollback itself misbehaves, follow `deploy-rollback.md` §Rollback. Causes #2–#5 are diagnostic/forward-only — there is no state to revert; if a config or memory-limit change made things worse, revert that single edit in `docker-compose.prod.yml` / `.env.prod` and `docker compose up -d <service>` again.

## Related

- Alert rule: `infra/observability/prometheus/rules/host-alerts.yaml`
- Project memory: "Auto-sync ships unverified code"
- `memory-pressure.md`, `disk-cleanup-root.md`, `deploy-rollback.md`

### Escalation

If restarts continue after triage:
- Increment severity to S1 if the user-facing service is the affected container (`client-api`, `frontend`).
- Open a post-mortem in `eusolicit-docs/post-mortems/`.
- If the trigger commit is `chore: auto-sync`, append a row to `incident-management/auto-sync-incidents.md`.
