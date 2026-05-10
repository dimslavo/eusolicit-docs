# Story onprem-02: Redis Persistence and Recovery

Status: ready-for-dev

**Origin:** Onprem pivot decision (eusolicit-docs/planning-artifacts/onprem-pivot-decision-2026-05-11.md). Replaces `pe-03-elasticache-cutover`.
**Priority:** P0 — launch-blocker. Redis is the event bus (ADR-003) + Celery broker + cache. Loss without recovery breaks the application.
**Epic:** Post-Epic-21 / Onprem Launch
**Points:** 1 | **Type:** infra

## Story

As a **platform-engineering operator**,
I want **the EU Solicit Redis container on www1 to persist to disk (AOF + RDB), replicate snapshots off-site to Hetzner Storage Box, and survive a controlled restart without losing in-flight Celery tasks or Redis Streams consumer-group state**,
so that **a Redis container crash or host restart does not lose user-visible work and the RPO ≤ 24h commitment for the cache-and-events layer holds**.

## Acceptance Criteria

1. **AOF persistence enabled** — `redis.conf` mounted into `eusolicit-app-redis-1` sets `appendonly yes` and `appendfsync everysec` (default balance: ≤1s data loss on crash; no per-write fsync overhead). `auto-aof-rewrite-percentage 100` and `auto-aof-rewrite-min-size 64mb` so the AOF doesn't grow unboundedly.

2. **RDB snapshots on hourly cadence** — `save 3600 1` (snapshot if ≥1 change in last hour) in `redis.conf`. RDB file at `/data/dump.rdb` (default).

3. **Off-site RDB replication** — `dump.rdb` is included in the daily 03:00 backup (same Hetzner Storage Box target as `onprem-01`). Path: `hetzner:eusolicit/redis/<date>/dump.rdb`. Retention matches: 7d local + 35d off-site.

4. **Controlled-restart drill** — `docker compose restart redis` is executed on a sentinel test write set. Verify:
   - Redis Streams consumer groups recover via `XINFO STREAMS <stream>` (consumer-group state must be preserved across restart).
   - Celery beat tasks fire on schedule post-restart (verify with the `reset_stuck_proposals_task` if `dw-02` is complete, or a tracer task).
   - The 6 services' `redis-py` clients auto-reconnect via the existing resilience hardening (`socket_keepalive=True`, `health_check_interval=30`, retry policy from Story 21-3 — RETAINED in code per ADR-010 rewrite).
   - Cache hits resume within 10s of restart.

5. **Restore runbook at `eusolicit-docs/runbooks/redis-restore.md`** — documents two restore paths: (a) replay from local AOF (zero off-site dependency, fastest), (b) restore RDB from Hetzner Storage Box if local volume is lost. Includes credential retrieval, network reconfig, and post-restore validation queries.

6. **First controlled-restart drill executed** — runbook executed against the live www1 stack at a low-traffic window. Document measured "Redis-unavailable window" (target ≤30s) and any service-level errors observed. Add a §First Drill Results section.

## Tasks / Subtasks

- [ ] Task 1: Add `redis.conf` snippet to `infra/redis/redis.conf` (NEW directory) with AOF + RDB + maxmemory settings.
- [ ] Task 2: Mount `redis.conf` into the redis container via `docker-compose.prod.yml` volume.
- [ ] Task 3: Extend `scripts/onprem/postgres-backup.sh` (or write a sibling `redis-backup.sh`) to include `dump.rdb` in the off-site push.
- [ ] Task 4: Write `scripts/onprem/redis-restore.sh` for the off-site restore path.
- [ ] Task 5: Author `eusolicit-docs/runbooks/redis-restore.md` runbook.
- [ ] Task 6: Execute the controlled-restart drill at a low-traffic window; record results.
- [ ] Task 7: Verify all 6 services reconnect cleanly post-restart (visible via `onprem-03` monitoring once that lands).

## Dev Notes

### What's already in code (retained from Story 21-3)

The `redis-py` resilience hardening landed by Story 21-3 is unchanged by this pivot. Every `redis.from_url(...)` call site in the codebase has:

- `socket_keepalive=True`
- `health_check_interval=30`
- `retry=Retry(ExponentialBackoff(cap=10, base=1), 3)`
- `retry_on_error=[ConnectionError, TimeoutError]`
- `socket_connect_timeout=5`
- `socket_timeout=10`

Both `celery_app.py` files have:
- `broker_connection_retry_on_startup=True`
- `broker_transport_options` with resilience keys

This story does NOT change application code. It only changes the redis container config + adds the backup/restore path.

### Why AOF + RDB and not just RDB

RDB alone has a per-snapshot RPO (1 hour, in our config). AOF reduces RPO to ~1s for any write that lands between snapshots. The combination matches the project's "everything matters" approach to data — even cache state (Redis stream consumer offsets, in-flight Celery tasks) is worth ≤1s of loss, not 1 hour.

### Drill timing

Schedule the first controlled-restart drill at the same low-traffic window as future chaos drills (descoped `pe-04-chaos-drill-execution`). Coordinate with that story.

### References

- ADR-003: Redis Streams as primary event bus (`architecture.md` §ADR-003)
- ADR-010 rewrite: `architecture.md` §ADR-010 (2026-05-11)
- Story 21-3 (closed, superseded): retained code lives in `services/*/src/*/dependencies.py` and `celery_app.py` files
- Archived migration runbook: `eusolicit-docs/implementation-artifacts/.archive/pe-03-cutover-runbook.md` (§Connection Audit lists all 15 `redis-py` call sites with hardening verified)
- Hetzner credentials: shared with `onprem-01` (.env.backup on www1)

### Out of scope

- Redis Sentinel (deferred to future Phase-2 HA).
- Redis Cluster sharding (rejected — same reasoning as PE.03 Story 21-1 §PE.03).
- Cross-region replica (not justified at single-host launch posture).
