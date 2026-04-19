# Story 8.14: Subscription Changed Event & Tier Cache Invalidation

Status: complete

## Story

As a **platform engineer ensuring real-time tier enforcement**,
I want a **Redis cache layer for company tier lookups and a `subscription.changed` stream consumer that invalidates stale tier entries**,
so that **feature access gates reflect subscription changes immediately after a plan upgrade, downgrade, or trial expiry — with zero stale-cache permission bypass**.

## Acceptance Criteria

1. **AC1 — Redis Tier Cache Service** — A new `tier_cache.py` module at `services/client-api/src/client_api/core/tier_cache.py` exposes:
   - `TIER_CACHE_PREFIX = "tier"` — key prefix constant
   - `tier_cache_key(company_id: UUID) -> str` — returns `f"tier:{company_id}"`
   - `get_cached_tier(redis_client, company_id) -> str | None` — reads from Redis; returns `None` on miss or error (never raises)
   - `set_cached_tier(redis_client, company_id, tier, ttl=60) -> None` — writes with TTL=60s; fire-and-forget (never raises)
   - `invalidate_cached_tier(redis_client, company_id) -> None` — deletes the key; fire-and-forget (never raises)

2. **AC2 — `opportunity_tier_gate.py` Uses Cache** — `get_opportunity_tier_gate` is updated to implement Redis read-through:
   - Calls `get_cached_tier(redis_client, company_id)` first
   - On cache HIT: returns the cached tier without a DB query
   - On cache MISS: executes the existing DB SELECT, then calls `set_cached_tier` to populate cache
   - Remove the `# TODO(Story-8-14)` comment
   - Structlog debug event `"opportunity_tier_gate.cache_hit"` when cache HIT, `"opportunity_tier_gate.cache_miss"` when DB fallback

3. **AC3 — `tier_gate.py` Uses Cache** — `require_paid_tier`, `require_professional_plus_tier`, and `require_enterprise_tier` are updated to use a shared `_get_tier_with_cache()` helper:
   - Same read-through pattern: cache first, DB on miss, populate cache on miss
   - Helper signature: `async def _get_tier_with_cache(company_id, redis_client, session) -> str | None`
   - Returns the tier string for the company's active/trialing subscription, or `None` if none exists
   - Each `require_*` function applies its own tier set check against the returned value
   - Structlog debug event `"tier_gate.cache_hit"` / `"tier_gate.cache_miss"`

4. **AC4 — `subscription.changed` Consumer Background Task** — A consumer loop function `run_tier_cache_invalidator(app: FastAPI)` is added to a new module `services/client-api/src/client_api/services/tier_cache_consumer.py`:
   - Creates consumer group `client-api:tier-cache-invalidator` on stream `subscription.changed` (MKSTREAM on first create; `BUSYGROUP` error silently ignored)
   - Consumer name: `client-api-invalidator-1`
   - Reads up to 10 events per poll, blocks 2000ms
   - For each event: extracts `company_id` from payload, calls `invalidate_cached_tier`
   - Calls `consumer.ack()` after invalidation
   - Handles pending messages on startup (calls `consumer.process_pending` for DLQ migration after 3 retries)
   - Runs in an infinite loop; catches all exceptions and logs at WARNING — never terminates the loop on error
   - `structlog` log event `"tier_cache_invalidator.invalidated"` per company invalidated

5. **AC5 — Consumer Registered in Lifespan** — `main.py` lifespan is updated to:
   - Create a `asyncio.create_task(run_tier_cache_invalidator(app))` on startup (after Stripe SDK setup)
   - Store the task reference in `app.state.tier_cache_invalidator_task`
   - On shutdown (after `yield`): cancel the task and await it (swallow `CancelledError`)
   - Log `"tier_cache_invalidator_started"` at startup and `"tier_cache_invalidator_stopped"` at shutdown

6. **AC6 — DB Fallback Guarantees Consistency** — On any Redis error (connection refused, timeout), `get_cached_tier` returns `None` (cache miss path), and the DB query always executes as fallback. This is verified by a unit test that injects a Redis client that raises `redis.RedisError` and asserts the DB path is taken.

7. **AC7 — Tests** — Tests cover:
   - **Unit** (`tests/unit/test_tier_cache.py`): `get_cached_tier` hit/miss/error; `set_cached_tier` sets correct TTL; `invalidate_cached_tier` calls `delete`; `tier_cache_key` format
   - **Unit** (`tests/unit/test_tier_cache_consumer.py`): consumer processes event → calls `invalidate_cached_tier`; no-op on missing `company_id` field; exception in invalidate does not stop loop
   - **Integration** (`tests/integration/test_subscription_changed_cache_invalidation.py`): test ID `8.14-INT-001` (P1, R-006) — publish a `subscription.changed` event, assert cache key is evicted, assert next tier gate call reads from DB (fakeredis + real DB)

## Tasks / Subtasks

### Task 1 — Create `tier_cache.py` (AC: 1)

- [x] **1.1** Create `services/client-api/src/client_api/core/tier_cache.py`:
  ```python
  """Redis tier cache helpers — Story 8-14.

  Provides read-through caching for company subscription tier lookups.
  All functions are fire-and-forget (never raise to callers) so a Redis
  outage falls back to DB queries without disrupting the request path.

  Cache key format: tier:{company_id}
  Default TTL: 60 seconds (per TODO(Story-8-14) in opportunity_tier_gate.py)

  Invalidation: DELETE on subscription.changed event (handled by tier_cache_consumer.py).
  """
  from __future__ import annotations

  from uuid import UUID

  import redis.asyncio
  import structlog

  log = structlog.get_logger()

  TIER_CACHE_PREFIX = "tier"
  TIER_CACHE_TTL_SECONDS = 60


  def tier_cache_key(company_id: UUID) -> str:
      """Return the Redis key for a company's cached tier."""
      return f"{TIER_CACHE_PREFIX}:{company_id}"


  async def get_cached_tier(
      redis_client: redis.asyncio.Redis,
      company_id: UUID,
  ) -> str | None:
      """Read cached tier for a company. Returns None on miss or Redis error."""
      try:
          value = await redis_client.get(tier_cache_key(company_id))
          return value.decode() if isinstance(value, bytes) else value
      except Exception:  # noqa: BLE001
          log.warning(
              "tier_cache.get_error",
              company_id=str(company_id),
              exc_info=True,
          )
          return None


  async def set_cached_tier(
      redis_client: redis.asyncio.Redis,
      company_id: UUID,
      tier: str,
      ttl: int = TIER_CACHE_TTL_SECONDS,
  ) -> None:
      """Write cached tier for a company with TTL. Never raises."""
      try:
          await redis_client.set(tier_cache_key(company_id), tier, ex=ttl)
      except Exception:  # noqa: BLE001
          log.warning(
              "tier_cache.set_error",
              company_id=str(company_id),
              tier=tier,
              exc_info=True,
          )


  async def invalidate_cached_tier(
      redis_client: redis.asyncio.Redis,
      company_id: UUID,
  ) -> None:
      """Delete cached tier for a company. Never raises."""
      try:
          await redis_client.delete(tier_cache_key(company_id))
          log.debug(
              "tier_cache.invalidated",
              company_id=str(company_id),
          )
      except Exception:  # noqa: BLE001
          log.warning(
              "tier_cache.invalidate_error",
              company_id=str(company_id),
              exc_info=True,
          )
  ```

### Task 2 — Update `opportunity_tier_gate.py` (AC: 2)

- [x] **2.1** Open `services/client-api/src/client_api/core/opportunity_tier_gate.py`
- [x] **2.2** Add imports at the top (after existing imports):
  ```python
  from client_api.core.tier_cache import get_cached_tier, set_cached_tier
  from client_api.dependencies import get_redis_client
  ```
- [x] **2.3** Replace the `get_opportunity_tier_gate` function body (from the `# TODO(Story-8-14)` comment onward) to implement Redis read-through:
  ```python
  async def get_opportunity_tier_gate(
      current_user: Annotated[CurrentUser, Depends(get_current_user)],
      session: Annotated[AsyncSession, Depends(get_db_session)],
  ) -> OpportunityTierGateContext:
      """FastAPI dependency: resolve the user's subscription tier with Redis cache.

      Cache hit: returns cached tier without DB query (TTL=60s).
      Cache miss: executes DB SELECT, populates cache, returns tier.
      Redis error: falls back to DB transparently.

      Story 8-3 (NIT-4): status IN ('active', 'trialing') — trial users get paid tier.
      Story 8-14: per-request tier cache via Redis (60s TTL, invalidated on subscription.changed).
      """
      company_id = current_user.company_id
      redis_client = get_redis_client()

      # Cache read-through
      cached_tier = await get_cached_tier(redis_client, company_id)
      if cached_tier is not None:
          log.debug(
              "opportunity_tier_gate.cache_hit",
              company_id=str(company_id),
              cached_tier=cached_tier,
          )
          effective_tier = cached_tier if cached_tier in _PAID_TIERS else "free"
      else:
          # Cache miss — hit DB
          stmt = (
              select(Subscription.tier)
              .where(Subscription.company_id == company_id)
              .where(Subscription.status.in_(_ALLOWED_STATUSES))
              .limit(1)
          )
          result = await session.execute(stmt)
          current_tier: str | None = result.scalar_one_or_none()
          effective_tier = current_tier if current_tier in _PAID_TIERS else "free"

          # Populate cache (fire-and-forget — cache miss path always returns the DB value)
          tier_to_cache = current_tier if current_tier is not None else "free"
          await set_cached_tier(redis_client, company_id, tier_to_cache)

          log.debug(
              "opportunity_tier_gate.cache_miss",
              company_id=str(company_id),
              db_tier=current_tier,
              effective_tier=effective_tier,
          )

      settings = get_settings()
      upgrade_url = settings.frontend_url.rstrip("/") + "/billing/upgrade"

      return OpportunityTierGateContext(
          user_tier=effective_tier,
          _upgrade_url=upgrade_url,
      )
  ```
- [x] **2.4** Remove the old `# TODO(Story-8-14)` comment and the original DB-only query code it preceded.

### Task 3 — Update `tier_gate.py` (AC: 3)

- [x] **3.1** Open `services/client-api/src/client_api/core/tier_gate.py`
- [x] **3.2** Add imports after existing imports:
  ```python
  from client_api.core.tier_cache import get_cached_tier, set_cached_tier
  from client_api.dependencies import get_redis_client
  ```
- [x] **3.3** Add helper `_get_tier_with_cache` after the `_ALLOWED_STATUSES` constant and before `require_paid_tier`:
  ```python
  async def _get_tier_with_cache(
      company_id,  # UUID
      redis_client: "redis.asyncio.Redis",
      session: AsyncSession,
  ) -> str | None:
      """Read company tier from Redis cache (miss → DB → repopulate cache).

      Returns None when the company has no active/trialing subscription.
      Redis errors fall through to DB transparently.

      Story 8-14: shared helper used by all three tier-gate dependencies.
      """
      cached = await get_cached_tier(redis_client, company_id)
      if cached is not None:
          log.debug("tier_gate.cache_hit", company_id=str(company_id), cached_tier=cached)
          return cached

      # Cache miss — DB query
      stmt = (
          select(Subscription.tier)
          .where(Subscription.company_id == company_id)
          .where(Subscription.status.in_(_ALLOWED_STATUSES))
          .limit(1)
      )
      result = await session.execute(stmt)
      current_tier: str | None = result.scalar_one_or_none()

      # Populate cache (fire-and-forget)
      tier_to_cache = current_tier if current_tier is not None else "free"
      await set_cached_tier(redis_client, company_id, tier_to_cache)

      log.debug(
          "tier_gate.cache_miss",
          company_id=str(company_id),
          db_tier=current_tier,
      )
      return current_tier
  ```
- [x] **3.4** Update `require_paid_tier` to call `_get_tier_with_cache`:
  ```python
  async def require_paid_tier(
      current_user: Annotated[CurrentUser, Depends(get_current_user_or_internal)],
      session: Annotated[AsyncSession, Depends(get_db_session)],
  ) -> CurrentUser:
      """Require active/trialing paid subscription. Uses Redis tier cache (Story 8-14)."""
      redis_client = get_redis_client()
      tier = await _get_tier_with_cache(current_user.company_id, redis_client, session)
      if tier not in PAID_TIERS:
          log.info("tier_gate.access_denied", company_id=str(current_user.company_id), ...)
          raise ForbiddenError("This feature requires a paid subscription.", details={"upgrade_required": True})
      return current_user
  ```
- [x] **3.5** Update `require_professional_plus_tier` similarly (check against `PROFESSIONAL_PLUS_TIERS`).
- [x] **3.6** Update `require_enterprise_tier` similarly (check against `ENTERPRISE_TIERS`).

### Task 4 — Create `tier_cache_consumer.py` (AC: 4)

- [x] **4.1** Create `services/client-api/src/client_api/services/tier_cache_consumer.py`:
  ```python
  """Tier cache invalidator — Story 8-14.

  Consumes subscription.changed events from Redis Streams and deletes
  the tier:{company_id} cache key so the next tier-gate request reads
  from DB (always consistent after a plan change).

  Consumer group: client-api:tier-cache-invalidator
  Consumer name:  client-api-invalidator-1
  DLQ threshold:  3 retries (via EventConsumer.process_pending)

  Architecture:
  - Runs as a persistent asyncio background task (asyncio.create_task) in lifespan.
  - Never raises — all exceptions caught and logged at WARNING.
  - cancel() safe: loop exits cleanly on asyncio.CancelledError.
  """
  from __future__ import annotations

  import asyncio
  from uuid import UUID

  import structlog
  from eusolicit_common.events.consumer import EventConsumer
  from fastapi import FastAPI

  from client_api.core.tier_cache import invalidate_cached_tier
  from client_api.dependencies import get_redis_client

  log = structlog.get_logger()

  _STREAM = "subscription.changed"
  _GROUP = "client-api:tier-cache-invalidator"
  _CONSUMER = "client-api-invalidator-1"
  _BLOCK_MS = 2000
  _COUNT = 10
  _DLQ_MAX_RETRIES = 3


  async def _ensure_consumer_group(redis_client) -> None:
      """Create consumer group if it does not exist. MKSTREAM creates stream if absent."""
      try:
          await redis_client.xgroup_create(
              _STREAM, _GROUP, id="0", mkstream=True
          )
      except Exception as exc:  # noqa: BLE001
          # BUSYGROUP error = group already exists — not a problem
          if "BUSYGROUP" in str(exc):
              return
          log.warning(
              "tier_cache_invalidator.group_create_error",
              error=str(exc),
              exc_info=True,
          )


  async def run_tier_cache_invalidator(app: FastAPI) -> None:  # noqa: ARG001
      """Persistent background task: consume subscription.changed → invalidate tier cache."""
      redis_client = get_redis_client()
      consumer = EventConsumer(redis_client)

      await _ensure_consumer_group(redis_client)
      log.info("tier_cache_invalidator.started", stream=_STREAM, group=_GROUP)

      while True:
          try:
              # Process any pending messages that exceeded retry threshold first
              await consumer.process_pending(
                  _STREAM, _GROUP, _CONSUMER, max_retries=_DLQ_MAX_RETRIES
              )

              # Read new messages
              events = await consumer.consume(
                  _STREAM,
                  _GROUP,
                  _CONSUMER,
                  count=_COUNT,
                  block_ms=_BLOCK_MS,
              )

              for event in events:
                  msg_id = event.get("_message_id")
                  company_id_str = event.get("company_id")

                  if not company_id_str:
                      log.warning(
                          "tier_cache_invalidator.missing_company_id",
                          message_id=str(msg_id),
                      )
                      if msg_id:
                          await consumer.ack(_STREAM, _GROUP, msg_id)
                      continue

                  try:
                      company_id = UUID(company_id_str)
                      await invalidate_cached_tier(redis_client, company_id)
                      log.info(
                          "tier_cache_invalidator.invalidated",
                          company_id=company_id_str,
                          message_id=str(msg_id),
                      )
                  except Exception:  # noqa: BLE001
                      log.warning(
                          "tier_cache_invalidator.invalidate_error",
                          company_id=company_id_str,
                          message_id=str(msg_id),
                          exc_info=True,
                      )
                  finally:
                      if msg_id:
                          await consumer.ack(_STREAM, _GROUP, msg_id)

          except asyncio.CancelledError:
              log.info("tier_cache_invalidator.cancelled")
              raise  # Re-raise so the task exits cleanly
          except Exception:  # noqa: BLE001
              log.warning(
                  "tier_cache_invalidator.loop_error",
                  exc_info=True,
              )
              await asyncio.sleep(5)  # back-off before retry
  ```

### Task 5 — Update `main.py` Lifespan (AC: 5)

- [x] **5.1** Open `services/client-api/src/client_api/main.py`
- [x] **5.2** Add import (after other service imports):
  ```python
  from client_api.services.tier_cache_consumer import run_tier_cache_invalidator
  ```
- [x] **5.3** Update `lifespan` to start/stop the consumer task:
  ```python
  @asynccontextmanager
  async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
      """FastAPI lifespan — run startup/shutdown hooks."""
      settings = get_settings()
      if settings.stripe_secret_key:
          import stripe as stripe_sdk  # noqa: PLC0415
          stripe_sdk.api_key = settings.stripe_secret_key
          stripe_sdk.api_version = settings.stripe_api_version
          logger.info("stripe_sdk_configured", api_version=settings.stripe_api_version)
      else:
          logger.warning(
              "stripe_secret_key_missing",
              msg="Stripe provisioning will be skipped — set CLIENT_API_STRIPE_SECRET_KEY",
          )

      # Story 8-14: start tier cache invalidator background task
      task = asyncio.create_task(run_tier_cache_invalidator(app))
      app.state.tier_cache_invalidator_task = task
      logger.info("tier_cache_invalidator_started")

      yield  # Application running

      # Shutdown: cancel the background task gracefully
      task.cancel()
      try:
          await task
      except asyncio.CancelledError:
          pass
      logger.info("tier_cache_invalidator_stopped")
  ```
- [x] **5.4** Add `import asyncio` at the top of `main.py` if not already present.

### Task 6 — Write Unit Tests (AC: 7)

- [x] **6.1** Create `services/client-api/tests/unit/test_tier_cache.py`:
  ```python
  """Unit tests for core/tier_cache.py — Story 8-14 AC1/AC6."""
  ```
  Test cases:
  - `test_tier_cache_key_format` — `tier_cache_key(UUID)` returns `"tier:{uuid}"`
  - `test_get_cached_tier_hit` — mock Redis `get()` returns bytes → decoded string returned
  - `test_get_cached_tier_miss` — mock Redis `get()` returns `None` → returns `None`
  - `test_get_cached_tier_redis_error` — mock Redis raises `redis.RedisError` → returns `None` (no raise)
  - `test_set_cached_tier_sets_ttl` — mock Redis `set()` called with `ex=60`
  - `test_set_cached_tier_redis_error` — mock raises → no exception propagated
  - `test_invalidate_cached_tier_calls_delete` — mock Redis `delete()` called with correct key
  - `test_invalidate_cached_tier_redis_error` — mock raises → no exception propagated

- [x] **6.2** Create `services/client-api/tests/unit/test_tier_cache_consumer.py`:
  ```python
  """Unit tests for services/tier_cache_consumer.py — Story 8-14 AC4."""
  ```
  Test cases:
  - `test_consumer_invalidates_on_event` — mock EventConsumer returns one event with `company_id` → `invalidate_cached_tier` called
  - `test_consumer_acks_after_invalidation` — `consumer.ack()` called after invalidate
  - `test_consumer_skips_missing_company_id` — event without `company_id` → ack called, no invalidate
  - `test_consumer_continues_after_invalidate_error` — invalidate raises → loop continues, no unhandled exception

### Task 7 — Write Integration Test 8.14-INT-001 (AC: 7)

- [x] **7.1** Create `services/client-api/tests/integration/test_subscription_changed_cache_invalidation.py`:
  ```
  Test ID: 8.14-INT-001 (P1, R-006)
  Level: Integration (fakeredis + real DB via testcontainers)
  Scenario: subscription.changed event → tier cache key evicted → next tier gate reads from DB
  ```
  Test plan:
  1. Seed subscription row for test company (tier=`professional`, status=`active`)
  2. Populate Redis cache: `set tier:{company_id} starter` (simulate stale cache from prior session)
  3. Publish `subscription.changed` event to Redis Streams via `EventPublisher`
  4. Run `run_tier_cache_invalidator` for one iteration (non-blocking, count=1, block_ms=0)
  5. Assert: `tier:{company_id}` key does NOT exist in Redis (evicted)
  6. Call `get_opportunity_tier_gate(...)` → verify it hits DB → returns `professional` (from DB, not stale `starter`)
  7. Assert: `tier:{company_id}` key now set to `professional` (cache repopulated)

## Dev Notes

### Critical Context: What's Already Implemented

The `subscription.changed` Redis Streams **publishing** is **already complete** in two places — do NOT re-implement it:

1. `webhook_service.py` → `_publish_subscription_changed()` — called after `session.commit()` for all subscription lifecycle webhook events
2. `billing_service.py` — publishes on trial provisioning (AC4)

Story 8-14's scope is exclusively the **consumer side**: reading these events and invalidating the tier cache, plus adding the tier cache layer to the gate dependencies.

### Current State of Tier Gates (No Cache Today)

All three tier gate modules currently do raw DB queries per request:

- `core/tier_gate.py` — `require_paid_tier`, `require_professional_plus_tier`, `require_enterprise_tier` each issue a `SELECT FROM client.subscriptions ... LIMIT 1`
- `core/opportunity_tier_gate.py` — `get_opportunity_tier_gate` does the same; has explicit `# TODO(Story-8-14)` marker pointing here
- `core/usage_gate.py` — depends on `get_opportunity_tier_gate` via FastAPI `Depends()` — benefits automatically from the opportunity gate cache, no direct change needed

### File Structure

```
services/client-api/
├── src/client_api/
│   ├── core/
│   │   ├── tier_cache.py                    NEW — Redis cache helpers
│   │   ├── opportunity_tier_gate.py         MODIFIED — add cache read-through, remove TODO
│   │   └── tier_gate.py                     MODIFIED — add _get_tier_with_cache helper
│   ├── services/
│   │   └── tier_cache_consumer.py           NEW — subscription.changed consumer loop
│   └── main.py                              MODIFIED — start/stop consumer in lifespan
└── tests/
    ├── unit/
    │   ├── test_tier_cache.py               NEW — unit tests for AC1/AC6
    │   └── test_tier_cache_consumer.py      NEW — unit tests for AC4
    └── integration/
        └── test_subscription_changed_cache_invalidation.py   NEW — 8.14-INT-001
```

### Cache Key Design

- **Prefix**: `tier` (constant `TIER_CACHE_PREFIX`)
- **Key format**: `tier:{company_id}` — e.g., `tier:550e8400-e29b-41d4-a716-446655440000`
- **Value**: tier string — one of `"free"`, `"starter"`, `"professional"`, `"enterprise"`
- **TTL**: 60 seconds (per the TODO comment in `opportunity_tier_gate.py`)
- **Invalidation**: DELETE (not SET to "free") — forces a DB read on next request to get the accurate new tier

Do NOT use `SET tier:{company_id} free` as the invalidation strategy — this would create a 60-second window where a newly-upgraded user still sees "free". Always DELETE.

### Consumer Architecture: asyncio.create_task + lifespan (Not Celery)

Project rule: **No Celery workers** — use `BackgroundTasks` or `asyncio.create_task`. For a persistent stream consumer, `asyncio.create_task` in the lifespan is the correct pattern. The task runs for the lifetime of the FastAPI process.

The `EventConsumer` class in `eusolicit_common.events.consumer` already implements:
- `consume(stream, group, consumer_name, *, count, block_ms)` — wraps `XREADGROUP`
- `ack(stream, group, message_id)` — wraps `XACK`
- `process_pending(stream, group, consumer_name, max_retries=3)` — DLQ migration via `XPENDING` + `XCLAIM`

Import path: `from eusolicit_common.events.consumer import EventConsumer`

### Consumer Group Bootstrap

Redis `XGROUP CREATE` returns `BUSYGROUP Consumer Group name already exists` if the group exists. This is not an error — silently swallow it. The `mkstream=True` parameter creates the stream itself if it doesn't exist yet (avoids ordering dependency between publisher first-run and consumer startup).

### opportunity_tier_gate.py vs tier_gate.py: Different Consumers

`get_opportunity_tier_gate` returns an `OpportunityTierGateContext` (not `CurrentUser`) with `user_tier` + `_upgrade_url`. The `tier_gate.py` functions return `CurrentUser`. Their caching logic is identical but their return types differ — share the cache helper `_get_tier_with_cache` across both files by importing it from `tier_cache.py` rather than duplicating.

Actually, `_get_tier_with_cache` should be in `tier_gate.py` (as a private module helper) since both `opportunity_tier_gate.py` and `tier_gate.py` do the same DB query. Alternatively, put it in `tier_cache.py`. Either works — choose the cleaner separation.

**Decision**: Put `_get_tier_with_cache` directly in `tier_cache.py` as `async def get_or_fetch_tier(redis_client, company_id, session) -> str | None` — this avoids cross-file imports between the two gate modules.

### Redis Client Access Pattern

In `opportunity_tier_gate.py` and `tier_gate.py`, the Redis client is accessed via `get_redis_client()` (module-level alias, same as in `billing.py`). This is the established pattern for test patchability.

Do NOT inject Redis as a FastAPI `Depends()` parameter into the tier gate dependencies — the existing pattern calls `get_redis_client()` at runtime inside the function body.

### Handling `usage_gate.py`

`usage_gate.py` imports and uses `get_opportunity_tier_gate` via `Depends(get_opportunity_tier_gate)`. Since FastAPI deduplicates identical dependencies within a request, updating `get_opportunity_tier_gate` to use the cache automatically benefits all endpoints that depend on `UsageGateContext`. No changes needed in `usage_gate.py`.

### Critical Mistakes to Prevent

1. **Do NOT call `session.commit()` in the tier gate helpers** — they are called inside a request's DB session lifecycle managed by `get_db_session`. `set_cached_tier` is fire-and-forget for Redis only.

2. **Do NOT block the lifespan yield on the consumer** — `asyncio.create_task()` is non-blocking. The app starts serving requests immediately. The consumer runs concurrently.

3. **Do NOT `await task` before yield** — the consumer task runs indefinitely. `yield` is what makes the lifespan work; blocking before it means the app never starts.

4. **Do NOT forget `asyncio.CancelledError` re-raise** — in the consumer loop, `CancelledError` must be re-raised after catching it, otherwise the task shutdown on lifespan exit hangs.

5. **Do NOT reinvent the EventConsumer** — it's already in `eusolicit-common`. Use it exactly as `webhook_service.py` and `billing_service.py` use `EventPublisher`.

6. **Do NOT break existing tests** — `test_tier_gate.py`, `test_tier_gate_trial.py`, `test_tier_gate_context.py` use `AsyncMock` sessions. After Task 3, the mocks need to also patch `get_redis_client` to return a mock Redis client. Verify all existing tier gate tests still pass.

7. **Cache value for companies with no subscription** — when DB returns `None` (no active/trialing subscription), cache the value `"free"` (not `None`). Caching `None` would prevent the Redis key from being set (no-op), causing a DB query on every request. Cache `"free"` with TTL=60s.

8. **`usage_gate.py` imports `get_opportunity_tier_gate`** — do not accidentally change the function signature of `get_opportunity_tier_gate`. It must remain a FastAPI dependency with the same `Annotated[CurrentUser, Depends(...)]` and `Annotated[AsyncSession, Depends(...)]` parameters.

### Test Coverage Map (from Epic 8 Test Design)

| Test Design ID | Priority | Risk | Scenario | Coverage in This Story |
|---|---|---|---|---|
| **8.14-INT-001** | P1 | R-006 | `subscription.changed` invalidates tier cache; next request reads from DB | `test_subscription_changed_cache_invalidation.py` integration test |

R-006 mitigation (from test design): "Redis Streams `subscription.changed` consumer invalidates cache; DB fallback on cache miss (never trust stale cache)". This story closes R-006.

### Previous Story Learnings (S08.13)

From Story 8.13:
- ATDD tests for backend stories use **pytest + AsyncMock** (not Vitest/Node `fs`)
- All test files follow `from __future__ import annotations` at the top
- `fakeredis.aioredis` is available (`import fakeredis.aioredis as _fakeredis_async`) for Redis mocking in integration tests — see `test_subscription_usage.py`
- Tests use `@pytest.mark.skip` with `SKIP_REASON` comment during ATDD RED phase; remove `skip` decorators once implementation is done
- The `_billing_session()` context manager pattern from `billing.py` is for DB sessions; do not use it in `tier_cache.py` (which is Redis-only)

### Testing Pattern Reference

From `test_subscription_usage.py` (S08.08 integration test):
```python
try:
    import fakeredis.aioredis as _fakeredis_async
    _FAKEREDIS_OK = True
except ImportError:
    _fakeredis_async = None
    _FAKEREDIS_OK = False
```
Use this same guard for fakeredis imports in `test_subscription_changed_cache_invalidation.py`.

From `test_webhook_service.py` (unit test pattern for async functions):
- Use `AsyncMock()` for all async functions
- Patch via `unittest.mock.patch` at the module level (e.g., `"client_api.core.tier_cache.get_cached_tier"`)

### References

- [Source: eusolicit-docs/planning-artifacts/epic-08-subscription-billing.md#S08.14] — story definition
- [Source: eusolicit-docs/test-artifacts/test-design-epic-08.md#8.14-INT-001, R-006] — P1 test + risk mitigation target
- [Source: eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py#TODO(Story-8-14)] — explicit TODO comment to be resolved
- [Source: eusolicit-app/services/client-api/src/client_api/core/tier_gate.py] — all three gate deps to update
- [Source: eusolicit-app/services/client-api/src/client_api/services/webhook_service.py#_publish_subscription_changed] — publisher already implemented (do not re-implement)
- [Source: eusolicit-app/services/client-api/src/client_api/services/billing_service.py#subscription.changed] — second publish site (already implemented)
- [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/consumer.py] — EventConsumer: `consume`, `ack`, `process_pending`
- [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/publisher.py] — EventPublisher (already used by webhook_service)
- [Source: eusolicit-app/services/client-api/src/client_api/main.py#lifespan] — add task start/stop here
- [Source: eusolicit-app/services/client-api/src/client_api/dependencies.py] — `get_redis_client()` call pattern
- [Source: eusolicit-app/services/client-api/tests/integration/test_subscription_usage.py] — fakeredis import pattern for integration tests
- [Source: eusolicit-app/services/client-api/tests/unit/test_tier_gate.py] — tier gate unit test structure to follow

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (2026-04-19)

### Debug Log References

- **Existing test regressions after Task 3**: `test_tier_gate.py` and `test_tier_gate_trial.py` used `return_value=uuid.uuid4()` (simulating "row found" for old `SELECT id` query). After Task 3 changed the query to `SELECT tier`, mocks had to return `"professional"` / `"enterprise"` strings. Also patched `get_redis_client` in all tier gate unit tests to avoid live Redis calls.
- **Integration test fixture column mismatch**: `seeded_professional_subscription` initially referenced `created_at`/`updated_at` columns that don't exist in `client.subscriptions`. Fixed to use `current_period_start`/`current_period_end`.
- **EventPublisher envelope format**: The integration test called `publisher.publish(_STREAM, {dict})` — wrong positional arg. EventPublisher wraps `payload` as a JSON string in an envelope field, so `event.get("company_id")` in the consumer returned None. Fixed by: (1) updating the integration test to use the correct `publish(stream=..., event_type=..., payload=..., source_service=...)` call, and (2) updating the consumer to also look for `company_id` inside the `payload` JSON string (EventPublisher envelope format) in addition to top-level (unit-test mock format).
- **Ruff lint error in tier_gate.py**: `redis_client: "redis.asyncio.Redis"` forward reference with unimported `redis`. Fixed to untyped with a comment.

### Completion Notes List

1. All 5 implementation tasks complete: `tier_cache.py` (Task 1), `opportunity_tier_gate.py` (Task 2), `tier_gate.py` (Task 3), `tier_cache_consumer.py` (Task 4), `main.py` lifespan (Task 5).
2. ATDD GREEN phase: all `@pytest.mark.skip` removed from `test_tier_cache.py`, `test_tier_cache_consumer.py`, and `test_subscription_changed_cache_invalidation.py`.
3. 656 tests pass (653 unit + 3 integration); 0 failures; 0 regressions.
4. Lint (`ruff`) and type check (`mypy`) pass clean on all 5 modified/created source files.
5. Consumer reads `company_id` from both top-level event field (unit test mocks) and from `payload` JSON (real EventPublisher envelope) — dual-lookup for compatibility.
6. R-006 closed: end-to-end verified by `8.14-INT-001` — stale cache → consumer invalidates → gate reads DB → cache repopulated.

### File List

**Created:**
- `services/client-api/src/client_api/core/tier_cache.py`
- `services/client-api/src/client_api/services/tier_cache_consumer.py`

**Modified:**
- `services/client-api/src/client_api/core/opportunity_tier_gate.py` — Redis read-through cache added, TODO removed
- `services/client-api/src/client_api/core/tier_gate.py` — `_get_tier_with_cache` helper + all three gate functions updated
- `services/client-api/src/client_api/main.py` — lifespan starts/stops `run_tier_cache_invalidator` background task
- `services/client-api/tests/unit/test_tier_cache.py` — `@pytest.mark.skip` removed (12 tests active)
- `services/client-api/tests/unit/test_tier_cache_consumer.py` — `@pytest.mark.skip` removed (8 tests active)
- `services/client-api/tests/unit/test_tier_gate.py` — `get_redis_client` patches + mock return values corrected
- `services/client-api/tests/unit/test_tier_gate_trial.py` — `get_redis_client` patches + mock return values corrected
- `services/client-api/tests/integration/test_subscription_changed_cache_invalidation.py` — `@pytest.mark.skip` removed, `EventPublisher.publish()` call corrected
- `eusolicit-docs/implementation-artifacts/sprint-status.yaml` — status updated to `complete`
