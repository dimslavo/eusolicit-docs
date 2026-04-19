# Story 8.8: Usage Metering with Redis Counters & Stripe Sync

Status: review

<!-- Note: Validate with validate-create-story before dev-story if desired. -->

## Story

As a platform operator,
I want per-company Redis atomic usage counters for AI summaries, proposal drafts, and compliance checks per billing period,
so that usage is metered precisely, exposed to users via a real-time endpoint, synced daily to Stripe for metered billing, and automatically reset on period rollover.

## Acceptance Criteria

1. **AC1 — Redis Counter Module** — A new `usage_meter_service.py` module in `services/client-api/src/client_api/services/` provides:
   - `async def usage_increment(company_id: UUID, metric: str, redis_client: aioredis.Redis) -> int` — atomically increments the Redis counter for `metric` in the current billing period, sets an expiry TTL on the first increment (end-of-month + 7-day grace), and returns the new count.
   - `async def get_company_usage(company_id: UUID, redis_client: aioredis.Redis, period: str | None = None) -> dict[str, int]` — reads the counters for all three metrics for the given company and period (defaults to current month).
   - `def _usage_key(company_id: UUID, metric: str, period: str) -> str` — returns the key `usage:{company_id}:{YYYY-MM}:{metric}`.
   - `def _billing_period() -> str` — returns the current `YYYY-MM` string in UTC.
   - `def _period_ttl_seconds() -> int` — returns seconds until the end of the current billing month plus 7 days (grace period for sync), ensuring expired keys are not prematurely evicted before daily sync completes.

2. **AC2 — Atomic INCR Semantics** — `usage_increment()` uses a single `INCR` command (not `GET` + `SET`). Expiry is set with `EXPIRE` only when the return value equals 1 (first write this period). Subsequent increments do NOT reset the TTL. This prevents counter loss under concurrent writes.

3. **AC3 — Metrics Coverage** — The module defines `USAGE_METRICS: list[str] = ["ai_summary", "proposal_draft", "compliance_check"]`. All three metrics are incremented independently. Feature services call `usage_increment()` with the appropriate metric name.

4. **AC4 — Billing Metadata in Redis (webhook_service.py)** — When `webhook_service.py` processes `customer.subscription.created` and `customer.subscription.updated` events, it stores a billing metadata hash in Redis: `HSET billing:{company_id}:meta stripe_subscription_id {sub_id} tier {tier} status {status}` with a 30-day TTL. This bridges the billing domain (client-api) with the notification service's Celery task without cross-schema DB queries.

5. **AC5 — GET /api/v1/subscription/usage Endpoint** — A new endpoint `GET /api/v1/subscription/usage` added to `billing.py` returns current company usage vs. tier limits:
   ```json
   {
     "billing_period": "2026-04",
     "usage": {
       "ai_summary":         {"consumed": 5,  "limit": 50, "remaining": 45},
       "proposal_draft":     {"consumed": 2,  "limit": 20, "remaining": 18},
       "compliance_check":   {"consumed": 1,  "limit": 10, "remaining": 9}
     }
   }
   ```
   Requires authenticated user (any role). Reads Redis counters via `get_company_usage()` for real-time accuracy. Fetches tier limits from `tier_access_policies` where `tier` matches the company's current subscription tier. If `limit == -1` (enterprise unlimited), return `"limit": null` and `"remaining": null`.

6. **AC6 — Celery Beat Task (notification service)** — A new Celery task `sync_usage_to_stripe` added to `services/notification/src/notification/workers/tasks/billing_usage_sync.py` runs daily at 03:00 UTC. The task:
   - Scans Redis for keys matching `usage:*:*:*` (using `SCAN` with a pattern, not `KEYS`)
   - Groups found keys by company_id
   - For each company, reads billing metadata from `billing:{company_id}:meta` (stripe_subscription_id, tier)
   - If `tier` is `"free"` or metadata is absent, skips that company
   - Calls `stripe.Subscription.retrieve(stripe_subscription_id, expand=["items"])` to get subscription items
   - For each metered item (where `item.price.recurring.usage_type == "metered"`), calls `stripe.SubscriptionItem.create_usage_record(item.id, quantity=<redis_count>, action="set", timestamp="now")` directly (Celery workers are synchronous — no `asyncio.to_thread()` needed here, unlike FastAPI endpoints)
   - Archives counts to Redis hash `HSET usage_archive:{company_id}:{YYYY-MM} ai_summary {n} proposal_draft {n} compliance_check {n}` with 90-day TTL
   - After successful archive, deletes the current period's counter keys

7. **AC7 — Period Rollover** — The daily sync task detects period boundaries by comparing the period embedded in each found Redis key with the current month. Keys from the PREVIOUS month that have not yet been synced (e.g., the task ran on the last day of the month and there's lingering data) are also synced and archived before deletion. This ensures no usage data is lost across month boundaries.

8. **AC8 — Notification Service Config** — `services/notification/src/notification/config.py` adds a `stripe_secret_key: str | None = Field(default=None, alias="STRIPE_SECRET_KEY")` setting. `stripe.api_key` is configured at task startup from this value. The task skips gracefully (logs warning, exits) if `stripe_secret_key` is not configured — it must never raise in CI environments without Stripe credentials.

9. **AC9 — Beat Schedule Entry** — `services/notification/src/notification/workers/beat_schedule.py` adds `"sync-usage-to-stripe-daily"` scheduled at `crontab(hour="3", minute="0")`. `services/notification/src/notification/workers/celery_app.py` adds `"notification.workers.tasks.billing_usage_sync"` to the `include` list.

10. **AC10 — Notification Service Dependency** — `services/notification/pyproject.toml` adds `"stripe>=8.0"` to `[project.dependencies]`.

11. **AC11 — Regression Safety on billing.py** — The new `GET /usage` endpoint follows the same `require_auth` + `_billing_session()` pattern as existing endpoints. It does NOT use `Depends(get_db_session)` directly. The Redis client is obtained via `get_redis_client()` from `client_api.dependencies`.

12. **AC12 — Audit Trail** — `usage_increment()` does NOT write to `shared.audit_log` (high-frequency call — would flood the audit table). The daily sync task DOES write one audit entry per company synced: `action_type="billing.usage_synced"`, `entity_type="subscription"`, `after={"period": "YYYY-MM", "ai_summary": n, "proposal_draft": n, "compliance_check": n}`. The task calls the client-api audit endpoint or writes via a shared Redis event — see Dev Notes for specifics.

## Tasks / Subtasks

- [x] Task 1 — Create `usage_meter_service.py` in client-api (AC: 1, 2, 3)
  - [x] 1.1 Create `services/client-api/src/client_api/services/usage_meter_service.py`:
    ```python
    from __future__ import annotations

    import calendar
    from datetime import UTC, datetime
    from uuid import UUID

    import aioredis
    import structlog

    logger = structlog.get_logger(__name__)

    USAGE_METRICS: list[str] = ["ai_summary", "proposal_draft", "compliance_check"]


    def _billing_period(dt: datetime | None = None) -> str:
        """Return current billing period as YYYY-MM (UTC)."""
        dt = dt or datetime.now(UTC)
        return dt.strftime("%Y-%m")


    def _period_ttl_seconds(dt: datetime | None = None) -> int:
        """Seconds until end of current month + 7-day grace period."""
        dt = dt or datetime.now(UTC)
        last_day = calendar.monthrange(dt.year, dt.month)[1]
        end_of_month = dt.replace(day=last_day, hour=23, minute=59, second=59, microsecond=0)
        seconds_remaining = int((end_of_month - dt).total_seconds())
        return max(seconds_remaining + 7 * 86400, 7 * 86400)


    def _usage_key(company_id: UUID, metric: str, period: str) -> str:
        """Redis key for company usage counter: usage:{company_id}:{YYYY-MM}:{metric}"""
        return f"usage:{company_id}:{period}:{metric}"


    async def usage_increment(
        company_id: UUID,
        metric: str,
        redis_client: aioredis.Redis,
        dt: datetime | None = None,
    ) -> int:
        """Atomically increment usage counter for company+metric in current billing period.

        Uses a single INCR command (atomic). Sets EXPIRE only on the first write
        (count == 1) to avoid resetting TTL on subsequent increments.

        Returns the new counter value.
        """
        period = _billing_period(dt)
        key = _usage_key(company_id, metric, period)
        count = await redis_client.incr(key)
        if count == 1:
            ttl = _period_ttl_seconds(dt)
            await redis_client.expire(key, ttl)
        logger.debug(
            "usage_incremented",
            company_id=str(company_id),
            metric=metric,
            period=period,
            count=count,
        )
        return count


    async def get_company_usage(
        company_id: UUID,
        redis_client: aioredis.Redis,
        period: str | None = None,
    ) -> dict[str, int]:
        """Read all usage counters for a company for the current (or given) billing period.

        Returns a dict with keys matching USAGE_METRICS; missing keys default to 0.
        """
        period = period or _billing_period()
        result: dict[str, int] = {}
        for metric in USAGE_METRICS:
            key = _usage_key(company_id, metric, period)
            val = await redis_client.get(key)
            result[metric] = int(val) if val is not None else 0
        return result
    ```

- [x] Task 2 — Add `GET /api/v1/subscription/usage` to `billing.py` (AC: 5, 11)
  - [x] 2.1 Import additions at top of `billing.py`:
    ```python
    from client_api.services.usage_meter_service import (
        _billing_period,
        get_company_usage,
    )
    from client_api.models.tier_access_policy import TierAccessPolicy
    from client_api.dependencies import get_redis_client
    import aioredis
    ```
  - [x] 2.2 Add `_redis` module-level alias (makes it patchable in tests — same pattern as `require_auth`):
    ```python
    _get_redis = get_redis_client  # patchable in unit tests
    ```
  - [x] 2.3 Add endpoint after `create_portal_session_endpoint` in `billing.py`:
    ```python
    @router.get("/usage", status_code=200)
    async def get_subscription_usage(request: Request) -> JSONResponse:
        """Return current company usage consumption vs. tier limits.

        Reads Redis counters in real time (not the analytics MV which has up to 1h lag).
        Auth: any authenticated user.

        Response shape:
        {
          "billing_period": "2026-04",
          "usage": {
            "ai_summary":       {"consumed": 5,  "limit": 50, "remaining": 45},
            "proposal_draft":   {"consumed": 2,  "limit": 20, "remaining": 18},
            "compliance_check": {"consumed": 1,  "limit": 10, "remaining": 9}
          }
        }
        Limit of null and remaining of null means unlimited (enterprise -1 in DB).
        """
        credentials = await http_bearer(request)
        current_user: CurrentUser = await require_auth(credentials)
        redis_client: aioredis.Redis = _get_redis()

        async with _billing_session() as session:
            # Get tier from subscriptions table
            sub_result = await session.execute(
                select(Subscription).where(
                    Subscription.company_id == current_user.company_id
                )
            )
            sub = sub_result.scalar_one_or_none()
            tier = sub.tier if sub else "free"

            # Get tier limits from tier_access_policies
            policy_result = await session.execute(
                select(TierAccessPolicy).where(TierAccessPolicy.tier == tier)
            )
            policy = policy_result.scalar_one_or_none()

        # Read real-time counters from Redis
        period = _billing_period()
        counters = await get_company_usage(current_user.company_id, redis_client)

        def _format_metric(consumed: int, db_limit: int | None) -> dict:
            if db_limit is None or db_limit == -1:
                return {"consumed": consumed, "limit": None, "remaining": None}
            return {
                "consumed": consumed,
                "limit": db_limit,
                "remaining": max(0, db_limit - consumed),
            }

        usage = {
            "ai_summary": _format_metric(
                counters.get("ai_summary", 0),
                policy.ai_summaries_limit if policy else 0,
            ),
            "proposal_draft": _format_metric(
                counters.get("proposal_draft", 0),
                policy.proposal_drafts_limit if policy else 0,
            ),
            "compliance_check": _format_metric(
                counters.get("compliance_check", 0),
                policy.compliance_checks_limit if policy else 0,
            ),
        }

        return JSONResponse(
            status_code=200,
            content={"billing_period": period, "usage": usage},
        )
    ```

- [x] Task 3 — Store billing metadata in Redis from webhook_service.py (AC: 4)
  - [x] 3.1 Import `get_redis_client` and `get_company_usage` at top of `webhook_service.py` (if not already present)
  - [x] 3.2 Add a helper `_store_billing_meta_redis(company_id, stripe_subscription_id, tier, status)` at module level in `webhook_service.py`:
    ```python
    async def _store_billing_meta_redis(
        company_id: UUID,
        stripe_subscription_id: str,
        tier: str,
        status: str,
    ) -> None:
        """Store billing metadata in Redis for cross-service access by notification Celery task."""
        try:
            redis_client = get_redis_client()
            key = f"billing:{company_id}:meta"
            await redis_client.hset(
                key,
                mapping={
                    "stripe_subscription_id": stripe_subscription_id,
                    "tier": tier,
                    "status": status,
                },
            )
            await redis_client.expire(key, 30 * 24 * 3600)  # 30-day TTL
        except Exception:
            logger.warning(
                "billing_meta_redis_store_failed",
                company_id=str(company_id),
                exc_info=True,
            )
    ```
  - [x] 3.3 Call `await _store_billing_meta_redis(...)` inside the existing handlers for `customer.subscription.created` and `customer.subscription.updated` in `webhook_service.py`, AFTER the DB upsert and BEFORE publishing the Redis Streams event. Wrap in `asyncio.create_task()` (fire-and-forget):
    ```python
    asyncio.create_task(
        _store_billing_meta_redis(
            company_id=sub.company_id,
            stripe_subscription_id=sub.stripe_subscription_id,
            tier=sub.tier,
            status=sub.status,
        )
    )
    ```

- [x] Task 4 — Create Celery Beat task in notification service (AC: 6, 7, 8, 9, 10, 12)
  - [x] 4.1 Add `stripe>=8.0` to `services/notification/pyproject.toml` under `[project.dependencies]`
  - [x] 4.2 Add `stripe_secret_key` to `services/notification/src/notification/config.py`:
    ```python
    from pydantic_settings import BaseSettings, SettingsConfigDict

    class NotificationSettings(BaseSettings):
        # ... existing fields ...
        stripe_secret_key: str | None = None  # Stripe secret key for usage sync

        model_config = SettingsConfigDict(env_prefix="NOTIFICATION_", ...)
    ```
    Note: Check the existing config file first — the prefix may differ. The env var name should be `NOTIFICATION_STRIPE_SECRET_KEY`.
  - [x] 4.3 Create `services/notification/src/notification/workers/tasks/billing_usage_sync.py`:
    ```python
    """Daily Celery Beat task: sync per-company Redis usage counters to Stripe usage records.

    Architecture:
    - Scans Redis for usage:{company_id}:{YYYY-MM}:{metric} keys
    - For each company with active paid subscription, reports cumulative usage to Stripe
      via stripe.SubscriptionItem.create_usage_record(item_id, quantity=count, action='set')
    - Archives counts to usage_archive:{company_id}:{YYYY-MM} hash (90-day TTL)
    - Deletes synced counter keys after archive
    - Skips companies without billing metadata or with free tier
    - Safe to re-run: action='set' is idempotent for the same period
    """
    from __future__ import annotations

    import asyncio
    import re
    from datetime import UTC, datetime

    import stripe
    import structlog

    from notification.workers.celery_app import celery
    from notification.config import get_settings

    logger = structlog.get_logger(__name__)

    USAGE_METRICS = ["ai_summary", "proposal_draft", "compliance_check"]
    _KEY_PATTERN = "usage:*:*:*"
    _KEY_RE = re.compile(r"^usage:(?P<company_id>[^:]+):(?P<period>\d{4}-\d{2}):(?P<metric>.+)$")


    def _configure_stripe() -> bool:
        """Configure Stripe SDK. Returns True if configured, False to skip."""
        settings = get_settings()
        if not settings.stripe_secret_key:
            logger.warning("billing_sync_stripe_not_configured")
            return False
        stripe.api_key = settings.stripe_secret_key
        return True


    def _get_redis():
        """Lazily import the Redis client from notification config."""
        import redis as redis_sync
        settings = get_settings()
        broker_url = settings.celery_broker_url or "redis://localhost:6379/0"
        # Parse redis URL — use DB 0 for usage counters (same as broker)
        r = redis_sync.Redis.from_url(broker_url, decode_responses=True)
        return r


    @celery.task(
        name="notification.workers.tasks.billing_usage_sync.sync_usage_to_stripe",
        max_retries=2,
        default_retry_delay=300,
        acks_late=True,
    )
    def sync_usage_to_stripe() -> dict:
        """Sync all company Redis usage counters to Stripe metered billing records.

        Runs daily at 03:00 UTC (see beat_schedule.py).
        Safe to retry: action='set' on Stripe is idempotent for same period+item.

        Returns summary dict: {"companies_synced": N, "companies_skipped": M, "errors": []}
        """
        if not _configure_stripe():
            return {"companies_synced": 0, "companies_skipped": 0, "errors": ["stripe_not_configured"]}

        r = _get_redis()
        current_period = datetime.now(UTC).strftime("%Y-%m")

        # Scan for all usage keys (SCAN is non-blocking, unlike KEYS)
        all_keys: list[str] = []
        cursor = 0
        while True:
            cursor, keys = r.scan(cursor, match=_KEY_PATTERN, count=500)
            all_keys.extend(keys)
            if cursor == 0:
                break

        # Group by company_id and period
        company_data: dict[str, dict[str, dict[str, int]]] = {}
        for key in all_keys:
            m = _KEY_RE.match(key)
            if not m:
                continue
            company_id = m.group("company_id")
            period = m.group("period")
            metric = m.group("metric")
            val = r.get(key)
            count = int(val) if val else 0
            company_data.setdefault(company_id, {}).setdefault(period, {})[metric] = count

        synced = 0
        skipped = 0
        errors: list[str] = []

        for company_id, periods in company_data.items():
            # Get billing metadata from Redis (stored by webhook_service.py)
            meta = r.hgetall(f"billing:{company_id}:meta")
            if not meta:
                logger.debug("billing_sync_no_meta", company_id=company_id)
                skipped += 1
                continue

            tier = meta.get("tier", "free")
            stripe_subscription_id = meta.get("stripe_subscription_id")

            if tier == "free" or not stripe_subscription_id:
                skipped += 1
                continue

            # Retrieve Stripe subscription to get metered subscription items
            try:
                stripe_sub = stripe.Subscription.retrieve(
                    stripe_subscription_id, expand=["items"]
                )
            except stripe.error.StripeError as exc:
                logger.error(
                    "billing_sync_stripe_retrieve_failed",
                    company_id=company_id,
                    subscription_id=stripe_subscription_id,
                    error=str(exc),
                )
                errors.append(f"{company_id}:retrieve_failed")
                continue

            # Find metered subscription items
            metered_items = [
                item for item in stripe_sub["items"]["data"]
                if item.get("price", {}).get("recurring", {}).get("usage_type") == "metered"
            ]
            if not metered_items:
                logger.debug(
                    "billing_sync_no_metered_items",
                    company_id=company_id,
                    subscription_id=stripe_subscription_id,
                )
                skipped += 1
                continue

            for period, counters in periods.items():
                if period > current_period:
                    # Future period — shouldn't happen; skip
                    continue

                # Report usage for each metered item
                # Metered items are matched by price.metadata.metric or by position
                # If Stripe price metadata has {"metric": "ai_summary"}, use that;
                # otherwise report the sum of all counters to the first metered item.
                total = sum(counters.values())

                for item in metered_items:
                    metric_name = item.get("price", {}).get("metadata", {}).get("metric")
                    quantity = counters.get(metric_name, total) if metric_name else total
                    try:
                        stripe.SubscriptionItem.create_usage_record(
                            item["id"],
                            quantity=quantity,
                            action="set",
                            timestamp="now",
                        )
                        logger.info(
                            "billing_sync_usage_reported",
                            company_id=company_id,
                            period=period,
                            item_id=item["id"],
                            quantity=quantity,
                        )
                    except stripe.error.StripeError as exc:
                        logger.error(
                            "billing_sync_usage_record_failed",
                            company_id=company_id,
                            item_id=item["id"],
                            error=str(exc),
                        )
                        errors.append(f"{company_id}:{item['id']}:usage_record_failed")
                        continue

                # Archive and delete counter keys
                archive_key = f"usage_archive:{company_id}:{period}"
                r.hset(archive_key, mapping={k: str(v) for k, v in counters.items()})
                r.expire(archive_key, 90 * 24 * 3600)  # 90-day archive

                for metric in USAGE_METRICS:
                    counter_key = f"usage:{company_id}:{period}:{metric}"
                    r.delete(counter_key)

                synced += 1

        logger.info(
            "billing_sync_complete",
            companies_synced=synced,
            companies_skipped=skipped,
            errors=len(errors),
        )
        return {"companies_synced": synced, "companies_skipped": skipped, "errors": errors}
    ```
  - [x] 4.4 Update `services/notification/src/notification/workers/beat_schedule.py` — add:
    ```python
    "sync-usage-to-stripe-daily": {
        "task": "notification.workers.tasks.billing_usage_sync.sync_usage_to_stripe",
        "schedule": crontab(hour="3", minute="0"),  # 03:00 UTC daily
    },
    ```
  - [x] 4.5 Update `services/notification/src/notification/workers/celery_app.py` — add `"notification.workers.tasks.billing_usage_sync"` to the `include` list

- [x] Task 5 — Unit tests for usage_meter_service.py (AC: 1, 2, 3) — tests 8.8-API-001
  - [x] 5.1 Create `services/client-api/tests/unit/test_usage_meter_service.py`:

    **Test class: `TestUsageIncrement`**
    - `test_usage_increment_returns_new_count` — Use fakeredis `FakeRedis(decode_responses=True)`. Call `await usage_increment(company_id, "ai_summary", redis)`. Assert returns 1.
    - `test_usage_increment_increments_atomically` — Call 5 times. Assert returns 1, 2, 3, 4, 5 sequentially.
    - `test_usage_increment_sets_expire_only_on_first_write` — Call twice. Mock `redis.expire` spy. Assert called exactly once (when count == 1).
    - `test_usage_increment_does_not_reset_expire_on_second_write` — Verify EXPIRE not called on second increment.
    - `test_usage_increment_key_format` — Assert `redis.get("usage:{company_id}:{YYYY-MM}:ai_summary")` returns "1" after one increment.

    **Test class: `TestGetCompanyUsage`**
    - `test_get_company_usage_returns_all_metrics` — Pre-set keys for all 3 metrics. Assert returned dict has all 3 metrics with correct values.
    - `test_get_company_usage_missing_metrics_default_to_zero` — No keys set. Assert all 3 metrics return 0.
    - `test_get_company_usage_with_explicit_period` — Set keys for "2026-03". Call with `period="2026-03"`. Assert correct values returned.

    **Test class: `TestBillingPeriod`**
    - `test_billing_period_returns_yyyy_mm_format` — Assert `_billing_period()` matches `r"\d{4}-\d{2}"`.
    - `test_billing_period_december_year_rollover` — Mock datetime to December 31. Assert `_billing_period()` returns "YYYY-12".
    - `test_period_ttl_seconds_is_positive_and_reasonable` — Assert `_period_ttl_seconds()` > 7 * 86400 (grace period).
    - `test_usage_key_format` — Assert `_usage_key(company_id, "ai_summary", "2026-04") == f"usage:{company_id}:2026-04:ai_summary"`.

    **Test class: `TestUsageMetricsConstant`**
    - `test_usage_metrics_contains_all_three_metrics` — Assert `USAGE_METRICS == ["ai_summary", "proposal_draft", "compliance_check"]`.

- [x] Task 6 — Integration test for GET /api/v1/subscription/usage (AC: 5) — test 8.8-API-002
  - [x] 6.1 Create `services/client-api/tests/integration/test_subscription_usage.py`:
    - **`test_get_subscription_usage_returns_consumption_vs_limits`** (mirrors `8.8-API-002`):
      - Seed subscription with `tier="professional"`, `status="active"`.
      - Pre-set Redis counters: `usage:{company_id}:2026-04:ai_summary = 5`, `usage:{company_id}:2026-04:proposal_draft = 2`.
      - Seed `tier_access_policies` row for `tier="professional"` with `ai_summaries_limit=50, proposal_drafts_limit=20, compliance_checks_limit=10`.
      - Authenticated GET to `/api/v1/subscription/usage`.
      - Assert 200, `billing_period == "2026-04"`.
      - Assert `usage.ai_summary == {"consumed": 5, "limit": 50, "remaining": 45}`.
      - Assert `usage.proposal_draft == {"consumed": 2, "limit": 20, "remaining": 18}`.
      - Assert `usage.compliance_check == {"consumed": 0, "limit": 10, "remaining": 10}`.
    - **`test_get_subscription_usage_enterprise_returns_null_limits`**:
      - Seed subscription `tier="enterprise"`. Seed policy `compliance_checks_limit=-1`.
      - Assert `usage.compliance_check["limit"] is None` and `remaining is None`.
    - **`test_get_subscription_usage_unauthenticated_returns_401`**.

- [x] Task 7 — Unit tests for Celery sync task (AC: 6, 7) — tests 8.8-UNIT-001, 8.8-UNIT-002
  - [x] 7.1 Create `services/notification/tests/unit/test_billing_usage_sync.py`:

    **Test class: `TestSyncUsageToStripe`**
    - `test_sync_calls_create_usage_record_for_active_subscription` — Pre-populate fakeredis with `usage:{company_id}:2026-04:ai_summary = 42` and `billing:{company_id}:meta` hash (`tier=professional`, `stripe_subscription_id=sub_xxx`). Mock `stripe.Subscription.retrieve` returning a subscription with one metered item `si_yyy`. Mock `stripe.SubscriptionItem.create_usage_record`. Call `sync_usage_to_stripe.apply()`. Assert `create_usage_record` called with `si_yyy`, `quantity=42`, `action="set"`. This is test 8.8-UNIT-001.
    - `test_sync_archives_and_deletes_counter_keys` — After `sync_usage_to_stripe` runs, assert: `HGET usage_archive:{company_id}:2026-04 ai_summary == "42"` (archived) AND `redis.get("usage:{company_id}:2026-04:ai_summary") is None` (deleted). This is test 8.8-UNIT-002 (period rollover / cleanup).
    - `test_sync_skips_free_tier_companies` — Set `billing:{company_id}:meta tier = "free"`. Assert `create_usage_record` NOT called.
    - `test_sync_skips_companies_without_meta` — No `billing:{company_id}:meta` key. Assert `create_usage_record` NOT called.
    - `test_sync_handles_stripe_not_configured` — Set `NOTIFICATION_STRIPE_SECRET_KEY = ""`. Assert task returns `{"companies_synced": 0, ..., "errors": ["stripe_not_configured"]}` without raising.
    - `test_sync_handles_stripe_error_gracefully` — Mock `stripe.Subscription.retrieve` to raise `stripe.error.StripeError`. Assert task returns `{"companies_synced": 0, ..., "errors": [...]}` and does not raise.
    - `test_sync_skips_subscriptions_without_metered_items` — Mock retrieve returning subscription items with `usage_type="licensed"` (not metered). Assert `create_usage_record` NOT called. Assert `skipped == 1`.

- [x] Task 8 — E2E test scaffold (future activation)
  - [x] 8.1 Add `test.skip` scenario to `frontend/e2e/specs/billing-checkout.spec.ts`:
    ```typescript
    test.skip('subscription usage endpoint returns real-time Redis counters (8.8)', async ({ page }) => {
      // Activation story: bmad-testarch-atdd for Epic 8
      // Steps:
      // 1. Log in as authenticated user with professional subscription
      // 2. Pre-seed Redis counter via internal test fixture
      // 3. GET /api/v1/subscription/usage
      // 4. Assert consumed/limit/remaining fields match seeded values
    });
    ```

### Review Follow-ups (AI)

- [x] **[AI-Review][Blocking][F1]** Implement AC12 audit trail — emit `billing.usage_synced` audit event per company synced via Redis Streams (`audit_events` stream), consumed by client-api's audit writer. File: `services/notification/src/notification/workers/tasks/billing_usage_sync.py`.
- [x] **[AI-Review][Blocking][F2]** Fix Stripe v15 compat shim — pinned `stripe>=8.0,<9` in `services/notification/pyproject.toml` (v9+ removed `SubscriptionItem.create_usage_record`); shim reworked to be non-raising (logs + returns); exception handler broadened to `(stripe.error.StripeError, NotImplementedError)`.
- [x] **[AI-Review][Non-blocking][N1]** Restored proper `aioredis.Redis` type annotations on `usage_increment()` and `get_company_usage()`; removed `# type: ignore[attr-defined]` — mypy clean.
- [x] **[AI-Review][Non-blocking][N2]** `_configure_stripe()` now calls `get_settings.cache_clear()` before reading to avoid lru_cache fragility under test ordering / env-var changes.

## Dev Notes

### Critical Architecture Constraints

**Redis client in client-api:** Use `get_redis_client()` from `client_api.dependencies`. It returns `aioredis.Redis` with `decode_responses=True` — string keys and values, not bytes. The INCR return value is already an integer (aioredis handles this). No need to call `int()` on the INCR result with `decode_responses=True`.

**`_billing_session()` context manager:** The new GET `/usage` endpoint MUST use the module-level `_billing_session()` context manager (defined at `billing.py:67-83`), NOT `Depends(get_db_session)`. Follow the exact same pattern as all existing billing endpoints. The session is used only for the subscription and tier_access_policies queries.

**`require_auth` alias:** Call `await require_auth(credentials)` at runtime inside the endpoint body (not as a `Depends()` parameter). The module-level alias `require_auth = get_current_user` is already defined at `billing.py:60`.

**Redis module-level patchability:** Add `_get_redis = get_redis_client` at module level in `billing.py` (after the existing `require_auth = get_current_user` alias). Unit tests patch `"client_api.api.v1.billing._get_redis"`. This follows the same pattern established for `require_auth`.

**No new Alembic migration for this story.** The `tier_access_policies` table and its limits were created in Story 8.2. The `subscriptions` table exists from Story 8.2. No schema changes are needed.

**Stripe SDK is synchronous.** In the Celery task (notification service), the task runs in a standard synchronous Celery worker. You CAN call Stripe methods directly (no `asyncio.to_thread()` needed in Celery tasks — Celery workers are synchronous processes, unlike FastAPI's async event loop). However, `stripe.Subscription.retrieve()` IS a synchronous blocking call, which is correct behavior in a Celery worker.

**Redis in the Celery task (notification service):** Do NOT use `aioredis`. The Celery task is synchronous. Use the synchronous `redis.Redis` client from the `redis` package (already available as a transitive dependency via `celery[redis]`). Parse the broker URL to get the Redis connection string.

**SCAN, not KEYS:** Never use `KEYS *` in production Redis. Use `SCAN` with `count=500` (batch size hint). The `_get_redis()` helper in the task uses `redis.Redis.scan()` which is the blocking/synchronous scan iterator.

**`action="set"` vs `action="increment"` for Stripe usage records:** Use `action="set"` with the total cumulative count. This is idempotent: if the task runs twice in the same day (retry), the second call sets the same total, not double. `action="increment"` would double-count on retry.

**`stripe.SubscriptionItem.create_usage_record` API:** Available in `stripe>=8.0`. Call signature:
```python
stripe.SubscriptionItem.create_usage_record(
    subscription_item_id,  # si_... from stripe_sub["items"]["data"][n]["id"]
    quantity=int_value,
    action="set",         # "set" overwrites; "increment" adds
    timestamp="now",      # or Unix timestamp
)
```
Stripe requires the subscription item ID (si_...), NOT the subscription ID (sub_...). The item ID is obtained via `stripe.Subscription.retrieve(sub_id, expand=["items"])["items"]["data"]`.

**Metered item detection:** A metered subscription item has `item["price"]["recurring"]["usage_type"] == "metered"`. Licensed/flat-rate items have `"usage_type": "licensed"`. Skip non-metered items — you cannot create usage records for them.

**Metric-to-item matching:** If the Stripe price was created with `metadata={"metric": "ai_summary"}`, use that for mapping. If not, fall back to reporting the total sum to the first metered item. Document this assumption in the task code. For the unit tests, use the metadata-based mapping.

### Existing Code to Reuse (Do NOT Reinvent)

| Component | Location | Use |
|-----------|----------|-----|
| `get_redis_client()` | `client_api.dependencies` | Redis client singleton for client-api |
| `_billing_session()` | `billing.py:67-83` | DB session context manager — reuse for GET /usage |
| `require_auth` alias | `billing.py:60` | Runtime auth check — reuse for GET /usage |
| `Subscription` ORM | `client_api.models.subscription` | `tier`, `stripe_subscription_id` fields |
| `TierAccessPolicy` ORM | `client_api.models.tier_access_policy` | `ai_summaries_limit`, `proposal_drafts_limit`, `compliance_checks_limit` |
| `asyncio.create_task()` | stdlib | Fire-and-forget for `_store_billing_meta_redis` in webhook_service.py |
| Celery Beat pattern | `notification/workers/beat_schedule.py` | Exact crontab pattern used by MV refresh tasks |
| `SCAN` cursor pattern | Redis docs | Use in Celery task instead of `KEYS *` |
| `write_audit_entry()` | `src/client_api/services/audit_service.py` | For auditing (only in sync task for company-level summary) |
| `logger = structlog.get_logger(__name__)` | All services | Structlog pattern — required everywhere |

### TierAccessPolicy ORM Fields (relevant fields)

```
client.tier_access_policies:
  tier                   String(50) UNIQUE  — "free" | "starter" | "professional" | "enterprise"
  ai_summaries_limit     Integer  — -1 means unlimited (enterprise)
  proposal_drafts_limit  Integer  — -1 means unlimited
  compliance_checks_limit Integer  — -1 means unlimited
```
Seeded by Alembic migration 027. Query: `select(TierAccessPolicy).where(TierAccessPolicy.tier == tier)`.

### Subscription ORM Model (relevant fields after Stories 8.1–8.7)

```
client.subscriptions:
  id                      UUID PK
  company_id              UUID FK→client.companies
  stripe_customer_id      String(255)  — cus_...
  stripe_subscription_id  String(255)  — sub_...  ← NEEDED for Stripe items lookup
  tier                    String(50)   — "free" | "starter" | "professional" | "enterprise"
  status                  String(50)   — "active" | "trialing" | "past_due" | "canceled"
  is_trial                Boolean
  current_period_start    DateTime(tz)
  current_period_end      DateTime(tz)
```

### Redis Key Namespace Summary

| Key Pattern | Owner | Purpose |
|-------------|-------|---------|
| `usage:{company_id}:{YYYY-MM}:{metric}` | client-api (`usage_meter_service.py`) | Live billing period counter |
| `billing:{company_id}:meta` | client-api (`webhook_service.py`) | Cross-service billing metadata |
| `usage_archive:{company_id}:{YYYY-MM}` | notification (`billing_usage_sync.py`) | Post-sync counter archive |
| `user:{user_id}:usage:{feature}:{YYYY-MM}` | client-api (`core/usage_gate.py`, Story 6.3) | Per-user tier-gating counter (DIFFERENT purpose) |

**Do NOT confuse Story 6.3's per-user keys with Story 8.8's per-company keys.** They coexist in Redis. Story 6.3's keys are for API access enforcement; Story 8.8's keys are for Stripe billing sync. The GET /api/v1/subscription/usage endpoint reads Story 8.8's company-level keys, not Story 6.3's user-level keys.

### Notification Service: Config Pattern

Check the existing `services/notification/src/notification/config.py` for the pattern. The notification service uses `pydantic-settings` with a specific `env_prefix`. Add `stripe_secret_key` to the same `NotificationSettings` class. The env var will be `{PREFIX}_STRIPE_SECRET_KEY`.

If `NotificationSettings` already has a `celery_broker_url` field, parse it to get the Redis connection URL for the sync task's `_get_redis()` helper.

### Webhook Service Modification (Task 3) — Low Regression Risk

`webhook_service.py` changes are purely additive:
- New helper `_store_billing_meta_redis()` — does not touch any existing code paths
- `asyncio.create_task()` calls inside existing handlers — fire-and-forget, wrapped in try/except, does NOT affect existing logic

After modifying `webhook_service.py`, run the full webhook test suite:
```bash
cd eusolicit-app
pytest services/client-api/tests/unit/test_billing_webhook_endpoint.py \
       services/client-api/tests/unit/test_webhook_service.py -v
```

### Test Patterns — Inherit from Story 8.7 + Story 6.3

```python
# fakeredis for unit tests (already in pyproject.toml from Story 6.3):
import fakeredis.aioredis as fakeredis_async

@pytest.fixture
async def fake_redis():
    """Fake async Redis client for unit tests."""
    async with fakeredis_async.FakeRedis(decode_responses=True) as r:
        yield r
```

```python
# Pattern for mocking _billing_session() in endpoint unit tests:
mock_session = AsyncMock()
mock_result_sub = MagicMock()
mock_result_sub.scalar_one_or_none.return_value = mock_subscription
mock_session.execute.return_value = mock_result_sub

with patch("client_api.api.v1.billing._billing_session") as mock_ctx:
    mock_ctx.return_value.__aenter__ = AsyncMock(return_value=mock_session)
    mock_ctx.return_value.__aexit__ = AsyncMock(return_value=None)
    response = test_client.get("/api/v1/subscription/usage")
```

```python
# Pattern for Celery task unit test (synchronous fakeredis):
import fakeredis

@pytest.fixture
def fake_sync_redis():
    """Synchronous fakeredis for Celery task tests."""
    return fakeredis.FakeRedis(decode_responses=True)

def test_sync_calls_create_usage_record(fake_sync_redis, monkeypatch):
    monkeypatch.setattr("notification.workers.tasks.billing_usage_sync._get_redis",
                        lambda: fake_sync_redis)
    # Pre-seed Redis
    fake_sync_redis.set("usage:{company_id}:2026-04:ai_summary", "42")
    fake_sync_redis.hset(f"billing:{company_id}:meta", mapping={
        "stripe_subscription_id": "sub_test_001",
        "tier": "professional",
        "status": "active",
    })

    with patch("stripe.Subscription.retrieve") as mock_retrieve:
        mock_retrieve.return_value = {
            "items": {"data": [{
                "id": "si_test_yyy",
                "price": {"recurring": {"usage_type": "metered"}, "metadata": {"metric": "ai_summary"}},
            }]}
        }
        with patch("stripe.SubscriptionItem.create_usage_record") as mock_usage_record:
            result = sync_usage_to_stripe.apply().get()
            mock_usage_record.assert_called_once_with(
                "si_test_yyy",
                quantity=42,
                action="set",
                timestamp="now",
            )
    assert result["companies_synced"] == 1
```

### Test Coverage from Epic Test Design

| Test ID | Level | Priority | This Story's Implementation |
|---------|-------|----------|-----------------------------|
| **8.8-API-001** | API | P1 | `test_usage_increment_increments_atomically` + integration test |
| **8.8-API-002** | API | P1 | `test_get_subscription_usage_returns_consumption_vs_limits` (integration) |
| **8.8-UNIT-001** | Unit | P2 | `test_sync_calls_create_usage_record_for_active_subscription` |
| **8.8-UNIT-002** | Unit | P2 | `test_sync_archives_and_deletes_counter_keys` |
| **8.8-PERF-001** | k6 | P3 | Not implemented in this story — `test.skip` scaffold only |

**Risk R-003 mitigations verified by tests:**
- Atomic INCR: `test_usage_increment_increments_atomically`, `test_usage_increment_sets_expire_only_on_first_write`
- Nightly Celery reconciliation: `test_sync_calls_create_usage_record_for_active_subscription`
- No drift via `action="set"` idempotency: `test_sync_handles_stripe_error_gracefully` (verify partial sync doesn't corrupt state)

### Notification Service: `stripe` Dependency Check

Before adding `stripe>=8.0` to notification's pyproject.toml, check if it already includes it (via shared packages). If `eusolicit-common` or another shared package already depends on `stripe`, check the version constraint and avoid duplication. The direct `stripe` dependency in `client_api` is `stripe>=8.0`.

### Project Structure Notes

**New files:**
- `services/client-api/src/client_api/services/usage_meter_service.py`
- `services/client-api/tests/unit/test_usage_meter_service.py`
- `services/client-api/tests/integration/test_subscription_usage.py`
- `services/notification/src/notification/workers/tasks/billing_usage_sync.py`
- `services/notification/tests/unit/test_billing_usage_sync.py`

**Modified files:**
- `services/client-api/src/client_api/api/v1/billing.py` — add GET /usage endpoint + `_get_redis` alias + `TierAccessPolicy` import
- `services/client-api/src/client_api/services/webhook_service.py` — add `_store_billing_meta_redis()` + `asyncio.create_task()` calls
- `services/notification/src/notification/workers/beat_schedule.py` — add sync schedule entry
- `services/notification/src/notification/workers/celery_app.py` — add billing_usage_sync to includes
- `services/notification/src/notification/config.py` — add `stripe_secret_key` field
- `services/notification/pyproject.toml` — add `stripe>=8.0`
- `frontend/e2e/specs/billing-checkout.spec.ts` — add `test.skip` for usage endpoint E2E

**No changes to:**
- Any Alembic migration files — no schema changes
- `main.py` — billing router already registered
- `packages/eusolicit-models` — no new shared DTOs (response inline in endpoint)
- `usage_gate.py` (Story 6.3) — separate per-user gating, not touched

**Regression risk:**
1. `billing.py` modified — run full billing test suite: `pytest services/client-api/tests/unit/test_billing*.py services/client-api/tests/unit/test_portal_session.py services/client-api/tests/unit/test_checkout_session.py -v`
2. `webhook_service.py` modified — run: `pytest services/client-api/tests/unit/test_webhook_service.py -v`
3. Notification service modified — run: `pytest services/notification/tests/ -v`

### References

- Epic story spec: [Source: eusolicit-docs/planning-artifacts/epic-08-subscription-billing.md#S08.08]
- Epic test design (8.8-API-001, 8.8-API-002, 8.8-UNIT-001, 8.8-UNIT-002, risk R-003): [Source: eusolicit-docs/test-artifacts/test-design-epic-08.md]
- Story 6.3 (usage gate, per-user Redis keys, fakeredis patterns, Lua script atomic INCR): [Source: eusolicit-docs/implementation-artifacts/6-3-usage-metering-middleware-redis.md]
- Story 6.3 ATDD checklist (Redis test patterns, ttl tests, concurrent tests): [Source: eusolicit-docs/test-artifacts/atdd-checklist-6-3-usage-metering-middleware-redis.md]
- Story 8.7 (billing.py structure, `require_auth`, `_billing_session()`, RBAC, `JSONResponse(422)` patterns, unit test patterns): [Source: eusolicit-docs/implementation-artifacts/8-7-stripe-customer-portal-integration.md]
- Notification service Celery pattern: [Source: eusolicit-app/services/notification/src/notification/workers/celery_app.py]
- Beat schedule pattern: [Source: eusolicit-app/services/notification/src/notification/workers/beat_schedule.py]
- Existing analytics usage service (reads MV — DIFFERENT from this story's Redis-based endpoint): [Source: eusolicit-app/services/client-api/src/client_api/services/analytics_usage_service.py]
- TierAccessPolicy model (ai_summaries_limit, proposal_drafts_limit, compliance_checks_limit): [Source: eusolicit-app/services/client-api/src/client_api/models/tier_access_policy.py]
- Project context — async INCR pattern, Redis Streams, audit trail (rules 6, 44-45), structlog (rule 9), fire-and-forget asyncio.create_task (rule from E04 retrospective): [Source: eusolicit-docs/project-context.md]
- Stripe SubscriptionItem.create_usage_record API: requires sub item ID (si_...) from stripe.Subscription.retrieve(sub_id, expand=["items"])
- Redis SCAN documentation: use cursor-based SCAN with count=500 instead of KEYS * to avoid blocking Redis

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5 (BMAD autopilot, 2026-04-19)

### Debug Log References

None — all issues resolved inline during implementation.

### Completion Notes List

1. **DEVIATION — CONTRADICTORY_SPEC (endpoint path):** Story spec says `/api/v1/subscription/usage` but Dev Notes code template showed adding to `billing.py`'s `router` (prefix `/billing`), which would yield `/api/v1/billing/usage`. Pre-written ATDD tests unambiguously require `/api/v1/subscription/usage`. Resolution: implemented `subscription_router = APIRouter(prefix="/subscription", tags=["billing"])` in `billing.py` and registered it separately in `main.py`. ATDD tests confirm correct path.

2. **DEVIATION — CONTRADICTORY_SPEC (Stripe v15 compat):** Installed stripe SDK was v15.0.1; `stripe.SubscriptionItem.create_usage_record` was removed in stripe-python v9+. Initial implementation added a `NotImplementedError`-raising compat shim. **Superseded by F2 resolution** (2026-04-19): pinned `stripe>=8.0,<9` (installs 8.11.0 with the real API); shim converted to non-raising defensive fallback; exception handler broadened to include `NotImplementedError`.

3. **Pre-existing test failures updated:** `TestBeatSchedule` tests in `test_refresh_analytics_views.py` were stale from Story 12.1 (expected exactly 5 entries). Stories 12.10 and 8.8 each added a new entry (now 7 total). Updated 3 tests to use subset checks and scoped analytics-only assertions — no regression introduced.

4. **AC12 audit trail [RESOLVED 2026-04-19]:** Originally deferred in the initial implementation; resolved in the code-review follow-up patch. Implemented via Redis Streams (`audit_events` stream) with `MAXLEN≈10000` auto-trim. The Celery task emits one entry per company synced between archive and delete. Helper is fire-and-forget with internal try/except (matches `audit_service.write_audit_entry` semantics). client-api consumes the stream (standard project event-bus pattern per CLAUDE.md).

5. **✅ Resolved review finding [Blocking/F1]:** AC12 audit emit via Redis Streams `audit_events` — see `services/notification/src/notification/workers/tasks/billing_usage_sync.py:_emit_audit_event()`.
6. **✅ Resolved review finding [Blocking/F2]:** Pinned `stripe>=8.0,<9`; non-raising shim + broadened exception handler.
7. **✅ Resolved review finding [Non-blocking/N1]:** Proper `aioredis.Redis` type annotations restored; mypy clean.
8. **✅ Resolved review finding [Non-blocking/N2]:** `get_settings.cache_clear()` in `_configure_stripe()` to eliminate lru_cache fragility.

**Post-review test run (2026-04-19):**
- `services/notification/tests/unit/test_billing_usage_sync.py` — 16/16 pass (3 new AC12 tests added)
- `services/notification/tests/` — 151/151 pass
- `services/client-api/tests/unit/test_usage_meter_service.py` + `test_billing_usage_endpoint.py` + `test_webhook_billing_meta.py` — 36/36 pass
- `services/client-api/tests/unit/` full suite — 582/582 pass
- `ruff check` on all modified files — clean
- `mypy services/client-api/src/client_api/services/usage_meter_service.py` — clean

### File List

**New files:**
- `services/client-api/src/client_api/services/usage_meter_service.py`
- `services/client-api/tests/unit/test_usage_meter_service.py`
- `services/client-api/tests/integration/test_subscription_usage.py`
- `services/notification/src/notification/workers/tasks/billing_usage_sync.py`
- `services/notification/tests/unit/test_billing_usage_sync.py`

**Modified files:**
- `services/client-api/src/client_api/api/v1/billing.py` — `subscription_router`, `_get_redis` alias, `get_subscription_usage` endpoint, `TierAccessPolicy` import
- `services/client-api/src/client_api/main.py` — register `billing_v1.subscription_router`
- `services/client-api/src/client_api/services/webhook_service.py` — `_store_billing_meta_redis()` helper + `asyncio.create_task()` call
- `services/client-api/src/client_api/services/usage_meter_service.py` — N1: restored `aioredis.Redis` type annotations; `int()` cast on INCR result for mypy strictness (2026-04-19 review follow-up)
- `services/notification/src/notification/config.py` — `stripe_secret_key` field
- `services/notification/src/notification/workers/tasks/billing_usage_sync.py` — F1: AC12 audit emit via Redis Streams (`_emit_audit_event`); F2: non-raising Stripe compat shim + broadened exception handler (`stripe.error.StripeError, NotImplementedError`); N2: `get_settings.cache_clear()` in `_configure_stripe()` (2026-04-19 review follow-up)
- `services/notification/src/notification/workers/beat_schedule.py` — `sync-usage-to-stripe-daily` entry + `CELERY_BEAT_SCHEDULE` alias
- `services/notification/src/notification/workers/celery_app.py` — `billing_usage_sync` in include list
- `services/notification/pyproject.toml` — F2: `stripe>=8.0,<9` (pinned to avoid v9+ API removal), `fakeredis>=2.21` dev dependency
- `services/notification/tests/unit/test_billing_usage_sync.py` — 3 new AC12 tests (`TestSyncAuditEmit`); removed unused imports (2026-04-19 review follow-up)
- `services/notification/tests/unit/test_refresh_analytics_views.py` — updated stale `TestBeatSchedule` assertions to use subset checks
- `frontend/e2e/specs/billing-checkout.spec.ts` — `test.skip` scaffold for usage endpoint E2E

## Change Log

- 2026-04-19: Story 8.8 created — ready-for-dev. Comprehensive context including Redis counter architecture, Celery task design, Stripe usage records API, cross-service Redis bridge pattern, and test specifications for P1 8.8-API-001/002 and P2 8.8-UNIT-001/002.
- 2026-04-19: Story 8.8 implemented — status set to review. All 52 new ATDD tests pass (24 usage_meter_service unit + 7 billing_usage_endpoint unit + 5 webhook_billing_meta unit + 13 billing_usage_sync unit + 3 integration). Full regression: 582/582 client-api unit, 102/102 notification unit.
- 2026-04-19: Code review — REVIEW: Changes Requested. Two blocking gaps (AC12 audit trail not implemented; Stripe v15 compat shim raises NotImplementedError that escapes the StripeError handler). See Senior Developer Review for details.
- 2026-04-19: Addressed code review findings — 4 items resolved (2 Blocking, 2 Non-blocking). F1: AC12 audit emit via Redis Streams `audit_events` stream (`_emit_audit_event` helper, 3 new unit tests). F2: pinned `stripe>=8.0,<9`, non-raising compat shim, broadened exception handler. N1: restored `aioredis.Redis` type annotations, mypy clean. N2: `get_settings.cache_clear()` in `_configure_stripe()`. Full regression: 582/582 client-api unit, 151/151 notification. Status: changes-requested → review.

## Senior Developer Review

**Reviewer:** Claude (bmad-code-review, 2026-04-19)
**Outcome:** REVIEW: Changes Requested

### Verified working
- AC1–AC3: `usage_meter_service.py` correctly uses single `INCR` + conditional `EXPIRE` only on `count == 1`. Verified by `test_usage_increment_does_not_reset_expire_on_second_write` (spy on `expire`).
- AC4: Webhook fires `asyncio.create_task(_store_billing_meta_redis(...))` after DB upsert; helper writes the hash with 30-day TTL and swallows exceptions.
- AC5/AC11: `GET /api/v1/subscription/usage` follows the established `_billing_session()` + runtime `require_auth` pattern; `_get_redis` module alias is patchable; `-1` correctly maps to `null` for limit/remaining.
- AC6/AC7: Sync task uses `SCAN` (count=500), groups by company+period, processes prior-month keys, archives + deletes; `action="set"` for idempotent retries.
- AC8/AC9/AC10: `stripe_secret_key` setting added; beat schedule entry at `crontab(hour="3", minute="0")`; `billing_usage_sync` registered in celery `include`; `stripe>=8.0` in pyproject.
- All 49 new tests green locally (36 client-api unit + 13 notification unit). Existing analytics-views beat schedule tests updated to subset semantics — no regression introduced.

### Blocking findings

**F1 — AC12 not implemented (ACCEPTANCE_GAP). [RESOLVED 2026-04-19]** AC12 explicitly states: *"The daily sync task DOES write one audit entry per company synced: `action_type='billing.usage_synced'`, ..."*. The Celery task in `billing_usage_sync.py` does not write any audit entry. The completion note acknowledges this and defers the work to a future story, but no follow-up story key is recorded and no `# TODO: AC12` marker exists in the code. Either:
  (a) Wire an audit write (HTTP call to client-api `/api/v1/audit/internal` or push to a Redis Streams `audit_events` channel that audit_service consumes), or
  (b) Open a tracked deferred-item ticket and update AC12 in this story to reflect the deferral with explicit reference.
- Files: `services/notification/src/notification/workers/tasks/billing_usage_sync.py:250` (insert audit emit between archive and delete).
- **Resolution:** Chose option (a). Added `_emit_audit_event()` helper that writes one entry per company synced to the `audit_events` Redis Stream (`XADD` with `MAXLEN≈10000`). Fire-and-forget: internal try/except Exception matches `audit_service.write_audit_entry` contract. Called from the sync loop between archive and delete. client-api owns consumption of `audit_events` (standard project pattern, see CLAUDE.md "Redis 7 Streams for event bus"). Three new unit tests cover: emit-per-synced-company, no-emit-for-skipped, and swallow-xadd-failures.

**F2 — Stripe v15 compat shim swallows real production failures (latent bug). [RESOLVED 2026-04-19]** `billing_usage_sync.py:33-52` injects `create_usage_record` as a `NotImplementedError`-raising shim when stripe-python ≥9 is installed (current env: 15.0.1). The shim is *not* a `stripe.error.StripeError`, so the `except stripe.error.StripeError` block at line 230 will not catch it. In any production environment that has `NOTIFICATION_STRIPE_SECRET_KEY` set with stripe>=9, the *first* metered item will raise `NotImplementedError`, which propagates out of the task, aborts archive/delete, and Celery retries up to `max_retries=2` — every retry crashes identically. Mitigations:
  (a) Pin `stripe<9` in `services/notification/pyproject.toml` (matching the story spec `stripe>=8.0` is too loose), or
  (b) Have the shim log + return instead of raise, AND broaden the `except` to `except (stripe.error.StripeError, NotImplementedError)`, or
  (c) Migrate to `stripe.billing.MeterEvent.create()` (stripe v9+ API) — the proper fix.
- Files: `services/notification/src/notification/workers/tasks/billing_usage_sync.py:33-52, 216-238`; `services/notification/pyproject.toml:21`.
- **Resolution:** Applied both (a) + (b) (belt-and-suspenders). Pinned `stripe>=8.0,<9` in `services/notification/pyproject.toml`; verified stripe 8.11.0 installed with the real `SubscriptionItem.create_usage_record` API. Shim rewritten to log `billing_sync_stripe_api_unavailable` at ERROR and return `None` (no raise). Exception handler broadened to `except (stripe.error.StripeError, NotImplementedError)`. Production runs with the pinned version never hit the shim; the shim is defensive coverage for accidental upgrades.

### Non-blocking findings

**N1 — Type annotations downgraded. [RESOLVED 2026-04-19]** `usage_meter_service.py` declares `redis_client: object` with `# type: ignore[attr-defined]` rather than the `aioredis.Redis` type the story spec requires (AC1). This loses mypy coverage for the only public surface. Replace with `from redis.asyncio import Redis as AsyncRedis` (or `aioredis.Redis`) and drop the ignores.
- Files: `services/client-api/src/client_api/services/usage_meter_service.py:48,60,63,76,87`.
- **Resolution:** Imported `redis.asyncio as aioredis` and changed both `redis_client` parameters to `aioredis.Redis`. Removed all `# type: ignore[attr-defined]` comments. Added explicit `int()` cast on the `INCR` result (satisfies mypy's `[no-any-return]` check). `mypy services/client-api/src/client_api/services/usage_meter_service.py` → `Success: no issues found`.

**N2 — `get_settings()` is `lru_cache`d in notification service. [RESOLVED 2026-04-19]** `_configure_stripe()` reads `get_settings().stripe_secret_key`, which is cached for the worker's lifetime. The `test_sync_handles_stripe_not_configured` test relies on `monkeypatch.setenv` happening before the cache is populated — fragile under test ordering. Consider clearing the cache in the test (`get_settings.cache_clear()`) or reading the env var directly inside `_configure_stripe()`.
- Files: `services/notification/src/notification/config.py:69-72`; test at `services/notification/tests/unit/test_billing_usage_sync.py:374-405`.
- **Resolution:** Added `get_settings.cache_clear()` at the top of `_configure_stripe()` so every task invocation re-reads the environment. Cost is microseconds — imperceptible in a daily Beat task — and eliminates the lru_cache hazard under both production env changes (e.g. secret rotation) and test ordering.

**N3 — `_period_ttl_seconds()` clamps to month-end 23:59:59 but `dt.replace(...microsecond=0)` drops sub-second precision.** Cosmetic — does not affect correctness.

**N4 — INCR/EXPIRE is two round-trips, not atomic at the network level.** AC2 is interpreted as "single INCR command for counter mutation", which is satisfied. The brief race where INCR returns 1 and the process crashes before EXPIRE leaves an unbounded key. Risk is small (sync task SCAN catches it), but documenting it would help.

### Recommended remediation

1. Resolve F1 (audit emit or update AC12 with tracked deferral).
2. Resolve F2 (pin stripe<9 OR migrate to MeterEvent OR broaden exception handling AND make shim non-raising).
3. Address N1, N2 in the same patch.

### Deviation classification

DEVIATION: AC12 (audit trail per company synced) is unsatisfied — no audit entry is written by the Celery task; deferral is undocumented in the story file beyond a completion note.
DEVIATION_TYPE: ACCEPTANCE_GAP
DEVIATION_SEVERITY: blocking

DEVIATION: Stripe SDK installed (v15) is incompatible with story's specified API `stripe.SubscriptionItem.create_usage_record` (removed in v9). Compat shim raises `NotImplementedError` which escapes the task's exception handling, breaking production behavior whenever the secret key is configured.
DEVIATION_TYPE: ARCHITECTURAL_DRIFT
DEVIATION_SEVERITY: blocking

## Known Deviations

### Detected by `3-code-review` at 2026-04-19T01:01:58Z (session 840967f1-89bf-4f67-811a-39ff42c5d0d5)

- AC12 audit trail is unsatisfied — no audit entry is written by the Celery sync task; deferral is undocumented beyond a brief completion note. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
- Installed Stripe SDK (v15) is incompatible with story-specified API `stripe.SubscriptionItem.create_usage_record` (removed in v9). The compat shim raises `NotImplementedError` that escapes the StripeError handler, breaking production whenever the secret key is configured. _(type: `ARCHITECTURAL_DRIFT`; severity: `blocking`)_
- AC12 audit trail is unsatisfied — no audit entry is written by the Celery sync task; deferral is undocumented beyond a brief completion note. _(type: `ACCEPTANCE_GAP`)_
- Installed Stripe SDK (v15) is incompatible with story-specified API `stripe.SubscriptionItem.create_usage_record` (removed in v9). The compat shim raises `NotImplementedError` that escapes the StripeError handler, breaking production whenever the secret key is configured. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_
