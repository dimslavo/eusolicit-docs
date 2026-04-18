# Story 6.3: Usage Metering Middleware (Redis)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer building the opportunity access control layer**,
I want **a `UsageGate` FastAPI dependency that atomically checks and increments per-user, per-feature usage counters in Redis using a Lua script, enforces subscription-tier quotas, sets `X-Usage-Remaining` response headers, and raises HTTP 429 with a structured `usage_limit_exceeded` error body when the quota is hit**,
so that **AI summary generation (S06.08) and any future metered features are enforced against subscription-tier limits without race conditions, establishing the atomic usage control foundation for the opportunity discovery epic**.

## Acceptance Criteria

1. `UsageGateContext` is a Python `@dataclass` in `core/usage_gate.py` with fields `user_id: UUID`, `user_tier: str`, `_upgrade_url: str`; it exposes an async method `check_and_increment(feature: str, redis: aioredis.Redis, response: Response) -> None`

2. `check_and_increment` uses a **single atomic Redis Lua script** (registered as a module-level constant `_USAGE_LUA`) that: reads `GET` on `user:{user_id}:usage:{feature}:{billing_period}`, if `current >= limit` returns `{current, 1}` (exceeded flag), otherwise runs `INCR` + conditionally sets `EXPIRE` (only when the key is first created, i.e., new_count == 1), returns `{new_count, 0}`; the two operations (GET and INCR) must never be separate round-trips — atomicity prevents the E06-R-002 race condition

3. The Redis key pattern is `user:{user_id}:usage:{feature}:{billing_period}` where `billing_period` is the current UTC calendar month as `YYYY-MM` (e.g., `user:abc-123:usage:ai_summary:2026-04`); the `EXPIRE` TTL is set to the integer seconds remaining until midnight UTC of the first day of the following month (end-of-billing-period TTL)

4. Tier usage limits for the `ai_summary` feature are: `free=0`, `starter=10`, `professional=50`, `enterprise=None` (unlimited); these are defined as the module-level constant `FEATURE_TIER_LIMITS: dict[str, dict[str, int | None]]`; Enterprise (`None`) bypasses the Lua script entirely — no Redis key is written and no header is set

5. On **successful** increment (limit not exceeded), `check_and_increment` sets response header `X-Usage-Remaining: {limit - new_count}`; for Enterprise (unlimited), no header is set

6. On **limit exceeded**, `check_and_increment` sets response header `X-Usage-Remaining: 0` and raises:
   ```python
   AppException(
       "Usage limit exceeded for this billing period.",
       error="usage_limit_exceeded",
       status_code=429,
       details={
           "limit": limit,
           "used": current_count,   # value that triggered the block (≥ limit)
           "resets_at": resets_at,  # ISO8601 UTC datetime of next billing period start
           "upgrade_url": self._upgrade_url,
       },
   )
   ```

7. `get_usage_gate` is an async FastAPI dependency that: accepts `current_user: CurrentUser = Depends(get_current_user)` and `tier_gate: OpportunityTierGateContext = Depends(get_opportunity_tier_gate)` (reuses the per-request subscription resolution — no second DB query); constructs and returns `UsageGateContext(user_id=current_user.user_id, user_tier=tier_gate.user_tier, _upgrade_url=...)` where `_upgrade_url` is `get_settings().frontend_url.rstrip("/") + "/billing/upgrade"`

8. Unit tests in `tests/unit/test_usage_gate.py` using **fakeredis** (serial only — no concurrency) verify:
   - Starter tier (limit=10): first call sets counter to 1 and header `X-Usage-Remaining: 9`; tenth call sets counter to 10 and header `X-Usage-Remaining: 0`; eleventh call raises 429 with `error="usage_limit_exceeded"`, `details.limit=10`, `details.used=10`, `details.resets_at` non-empty ISO8601, `details.upgrade_url` non-empty
   - Free tier (limit=0): first call immediately raises 429 with `used=0, limit=0`
   - Enterprise tier (unlimited): no Redis key written, no exception raised, no header set
   - Professional tier (limit=50): call 50 succeeds (`X-Usage-Remaining: 0`); call 51 raises 429
   - Redis key TTL is ≤ seconds remaining in billing period after first INCR (E06-P1-008)
   - Sequential calls return correct `X-Usage-Remaining` values (E06-P1-009): after 3 calls against Starter, headers are `9, 8, 7`
   - `_billing_period_ttl()` helper returns correct YYYY-MM, TTL, and resets_at across month boundaries (Dec→Jan year rollover)

9. Concurrency/integration test in `tests/integration/test_usage_gate_concurrency.py` using **testcontainers Redis** (not fakeredis — required for E06-P0-004) verifies the atomic race guard:
   - Pre-set counter to `limit - 1` (e.g., 9 for Starter); fire two concurrent `check_and_increment` calls via `asyncio.gather` with `return_exceptions=True`; assert **exactly one** returns `None` (success) and **exactly one** raises `AppException` with status 429; assert Redis counter value is exactly `limit` (10) after both complete

10. Module-level `__all__` exports `["UsageGateContext", "get_usage_gate", "FEATURE_TIER_LIMITS"]`

## Tasks / Subtasks

- [x] Task 1: Create `core/usage_gate.py` with `UsageGateContext` dataclass, Lua script, tier limits, and FastAPI dependency (AC: 1–7, 10)
  - [x] 1.1 Create `eusolicit-app/services/client-api/src/client_api/core/usage_gate.py`:

    ```python
    """Usage metering dependency — Story S06.03.

    Provides UsageGateContext (returned by the get_usage_gate FastAPI dependency)
    and atomic Redis Lua-script-based usage quota enforcement for metered features
    (initial feature: "ai_summary").

    This module is separate from core/tier_gate.py (binary allow/deny) and
    core/opportunity_tier_gate.py (per-item field visibility + scope) because
    usage metering is a per-call atomic counter concern distinct from both.

    Tier usage limits for ai_summary (per billing period = calendar month):
      free:         0 (no AI summaries)
      starter:      10
      professional: 50
      enterprise:   None (unlimited — bypass metering entirely)

    Redis key pattern: user:{user_id}:usage:{feature}:{billing_period}
      e.g. user:abc-123:usage:ai_summary:2026-04

    Atomicity guarantee:
      The Lua script runs GET + conditional INCR as a single atomic Redis command.
      This prevents the E06-R-002 race condition where two concurrent requests both
      read the same counter value, both pass the limit check, and both increment —
      which would allow over-quota consumption.

    Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#s0603
    Risk mitigation: eusolicit-docs/test-artifacts/test-design-epic-06.md#e06-r-002
    """
    from __future__ import annotations

    import calendar
    from dataclasses import dataclass
    from datetime import UTC, datetime
    from typing import Annotated
    from uuid import UUID

    import structlog
    from eusolicit_common.exceptions import AppException
    from fastapi import Depends, Response
    from redis.asyncio import Redis

    from client_api.config import get_settings
    from client_api.core.opportunity_tier_gate import (
        OpportunityTierGateContext,
        get_opportunity_tier_gate,
    )
    from client_api.core.security import CurrentUser, get_current_user
    from client_api.dependencies import get_redis_client

    log = structlog.get_logger()

    # ---------------------------------------------------------------------------
    # Feature tier limits — add new metered features here
    # ---------------------------------------------------------------------------

    #: Per-feature, per-tier monthly usage limits.
    #: None = unlimited (Enterprise); 0 = blocked (Free).
    FEATURE_TIER_LIMITS: dict[str, dict[str, int | None]] = {
        "ai_summary": {
            "free": 0,
            "starter": 10,
            "professional": 50,
            "enterprise": None,  # unlimited
        },
    }

    # ---------------------------------------------------------------------------
    # Atomic Lua script: GET + conditional INCR (E06-R-002 mitigation)
    # ---------------------------------------------------------------------------

    #: Atomic Lua script for usage gate.
    #:
    #: KEYS[1]: usage counter Redis key
    #: ARGV[1]: tier limit (integer; must be ≥ 0)
    #: ARGV[2]: TTL seconds until end of billing period
    #:
    #: Returns: table {count, exceeded_flag}
    #:   exceeded_flag = 0 → INCR performed; count = new value after INCR
    #:   exceeded_flag = 1 → limit already reached; count = current value (no INCR)
    _USAGE_LUA = """
    local key   = KEYS[1]
    local limit = tonumber(ARGV[1])
    local ttl   = tonumber(ARGV[2])

    local current = tonumber(redis.call('GET', key) or 0)

    if current >= limit then
        return {current, 1}
    end

    local new_count = redis.call('INCR', key)
    if new_count == 1 then
        redis.call('EXPIRE', key, ttl)
    end

    return {new_count, 0}
    """

    # ---------------------------------------------------------------------------
    # Billing period helpers
    # ---------------------------------------------------------------------------


    def _billing_period_ttl(now: datetime) -> tuple[str, int, str]:
        """Return (billing_period, ttl_seconds, resets_at_iso8601) for the given UTC datetime.

        billing_period: YYYY-MM string for current calendar month
        ttl_seconds:    integer seconds until midnight UTC on the 1st of next month (min 1)
        resets_at:      ISO8601 UTC string of next month's first second
        """
        billing_period = now.strftime("%Y-%m")

        # First second of the following month (UTC)
        if now.month == 12:
            resets_dt = datetime(now.year + 1, 1, 1, tzinfo=UTC)
        else:
            resets_dt = datetime(now.year, now.month + 1, 1, tzinfo=UTC)

        ttl = max(1, int((resets_dt - now).total_seconds()))
        return billing_period, ttl, resets_dt.isoformat()


    # ---------------------------------------------------------------------------
    # UsageGateContext dataclass
    # ---------------------------------------------------------------------------


    @dataclass
    class UsageGateContext:
        """Carries the resolved user_id and tier for per-feature usage metering.

        Instantiated once per request by get_usage_gate.
        Not meant to be constructed directly in production code.
        """

        user_id: UUID
        """The authenticated user's UUID (used to scope the Redis counter key)."""

        user_tier: str
        """Effective subscription plan: 'free' | 'starter' | 'professional' | 'enterprise'."""

        _upgrade_url: str

        async def check_and_increment(
            self,
            feature: str,
            redis: Redis,
            response: Response,
        ) -> None:
            """Check usage limit and atomically increment the counter.

            For Enterprise tier (unlimited): returns immediately, no Redis write, no header.

            For all other tiers:
              1. Resolves the tier limit for the requested feature.
              2. Computes the Redis key and end-of-billing-period TTL.
              3. Executes the atomic Lua script (GET + conditional INCR).
              4. On success: sets ``X-Usage-Remaining`` response header.
              5. On limit exceeded: sets ``X-Usage-Remaining: 0`` and raises 429.

            Parameters
            ----------
            feature:
                Feature key matching an entry in FEATURE_TIER_LIMITS (e.g. "ai_summary").
            redis:
                Async Redis client (injected from get_redis_client dependency).
            response:
                FastAPI Response object for setting response headers.

            Raises
            ------
            AppException (error="usage_limit_exceeded", status_code=429)
                When the user's quota for this feature in the current billing period
                is at or above the tier limit.
            """
            tier_limits = FEATURE_TIER_LIMITS.get(feature, {})
            limit = tier_limits.get(self.user_tier)

            if limit is None:
                # Enterprise (unlimited): skip metering entirely
                log.debug(
                    "usage_gate.unlimited",
                    user_id=str(self.user_id),
                    user_tier=self.user_tier,
                    feature=feature,
                )
                return

            now = datetime.now(UTC)
            billing_period, ttl, resets_at = _billing_period_ttl(now)
            key = f"user:{self.user_id}:usage:{feature}:{billing_period}"

            result: list[int] = await redis.eval(  # type: ignore[assignment]
                _USAGE_LUA,
                1,          # numkeys
                key,        # KEYS[1]
                str(limit), # ARGV[1]
                str(ttl),   # ARGV[2]
            )
            count, exceeded = int(result[0]), int(result[1])

            if exceeded:
                response.headers["X-Usage-Remaining"] = "0"
                log.info(
                    "usage_gate.limit_exceeded",
                    user_id=str(self.user_id),
                    user_tier=self.user_tier,
                    feature=feature,
                    limit=limit,
                    used=count,
                    resets_at=resets_at,
                )
                raise AppException(
                    "Usage limit exceeded for this billing period.",
                    error="usage_limit_exceeded",
                    status_code=429,
                    details={
                        "limit": limit,
                        "used": count,
                        "resets_at": resets_at,
                        "upgrade_url": self._upgrade_url,
                    },
                )

            remaining = max(0, limit - count)
            response.headers["X-Usage-Remaining"] = str(remaining)

            log.debug(
                "usage_gate.incremented",
                user_id=str(self.user_id),
                user_tier=self.user_tier,
                feature=feature,
                count=count,
                remaining=remaining,
            )


    # ---------------------------------------------------------------------------
    # FastAPI dependency
    # ---------------------------------------------------------------------------


    async def get_usage_gate(
        current_user: Annotated[CurrentUser, Depends(get_current_user)],
        tier_gate: Annotated[OpportunityTierGateContext, Depends(get_opportunity_tier_gate)],
    ) -> UsageGateContext:
        """FastAPI dependency: resolve user identity and tier for usage metering.

        Reuses OpportunityTierGateContext (already resolved per request) to avoid a
        second subscription DB query.  FastAPI's DI system ensures get_opportunity_tier_gate
        is called at most once per request even when both this dependency and the route
        handler inject it.

        Returns
        -------
        UsageGateContext
            Contains user_id and user_tier ready for check_and_increment calls.

        Note: This dependency does NOT call Redis itself — it only resolves identity.
        The caller is responsible for passing the Redis client and Response to
        check_and_increment().  This keeps the dependency testable without a Redis fixture.
        """
        settings = get_settings()
        upgrade_url = settings.frontend_url.rstrip("/") + "/billing/upgrade"

        return UsageGateContext(
            user_id=current_user.user_id,
            user_tier=tier_gate.user_tier,
            _upgrade_url=upgrade_url,
        )


    __all__ = ["UsageGateContext", "get_usage_gate", "FEATURE_TIER_LIMITS"]
    ```

- [x] Task 2: Write unit tests using fakeredis (AC: 8)
  - [x] 2.1 Create `eusolicit-app/services/client-api/tests/unit/test_usage_gate.py`:

    ```python
    """Unit tests for UsageGateContext — Story S06.03.

    Uses fakeredis (serial, no concurrency) for all tests.
    The concurrent race-condition test (E06-P0-004) is in tests/integration/.

    Test IDs covered:
        E06-P0-005  429 body schema: error, limit, used, resets_at, upgrade_url
        E06-P1-008  INCR sets EXPIRE ≤ seconds remaining in billing period
        E06-P1-009  X-Usage-Remaining decrements correctly across sequential calls
        E06-P3-004  fakeredis serial parity — sequential counter values match expected

    Source: eusolicit-docs/test-artifacts/test-design-epic-06.md
    """
    from __future__ import annotations

    import re
    import uuid
    from datetime import UTC, datetime
    from unittest.mock import MagicMock

    import fakeredis.aioredis as fakeredis
    import pytest
    import pytest_asyncio

    from client_api.core.usage_gate import (
        FEATURE_TIER_LIMITS,
        UsageGateContext,
        _billing_period_ttl,
    )
    from eusolicit_common.exceptions import AppException

    # ---------------------------------------------------------------------------
    # Helpers
    # ---------------------------------------------------------------------------

    _TEST_UPGRADE_URL = "https://eusolicit.test/billing/upgrade"


    def _make_context(tier: str) -> UsageGateContext:
        return UsageGateContext(
            user_id=uuid.uuid4(),
            user_tier=tier,
            _upgrade_url=_TEST_UPGRADE_URL,
        )


    def _make_response() -> MagicMock:
        """Mock FastAPI Response that records header assignments."""
        resp = MagicMock()
        resp.headers = {}
        return resp


    @pytest_asyncio.fixture
    async def fake_redis():
        """Provide an in-memory async fakeredis client."""
        r = fakeredis.FakeRedis(decode_responses=True)
        yield r
        await r.aclose()


    # ---------------------------------------------------------------------------
    # _billing_period_ttl helper tests
    # ---------------------------------------------------------------------------

    def test_billing_period_ttl_mid_month():
        """Billing period is YYYY-MM; TTL is positive; resets_at is ISO8601."""
        now = datetime(2026, 4, 17, 12, 0, 0, tzinfo=UTC)
        period, ttl, resets_at = _billing_period_ttl(now)

        assert period == "2026-04"
        assert ttl > 0
        # Remaining time to 2026-05-01T00:00:00+00:00 ≈ 13.5 days = ~1,166,400s
        assert ttl <= int((datetime(2026, 5, 1, tzinfo=UTC) - now).total_seconds())
        assert resets_at.startswith("2026-05-01T00:00:00")


    def test_billing_period_ttl_december_year_rollover():
        """December billing period resets at Jan 1 of next year."""
        now = datetime(2026, 12, 15, 10, 0, 0, tzinfo=UTC)
        period, ttl, resets_at = _billing_period_ttl(now)

        assert period == "2026-12"
        assert ttl > 0
        assert resets_at.startswith("2027-01-01T00:00:00")


    def test_billing_period_ttl_last_second_of_month():
        """TTL is at least 1 second even at the very end of the month."""
        now = datetime(2026, 4, 30, 23, 59, 59, tzinfo=UTC)
        _, ttl, _ = _billing_period_ttl(now)
        assert ttl >= 1


    # ---------------------------------------------------------------------------
    # E06-P1-009 / E06-P3-004: Sequential X-Usage-Remaining decrements (Starter)
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_starter_sequential_remaining_header(fake_redis):
        """E06-P1-009 / E06-P3-004: Starter (limit=10) — X-Usage-Remaining decrements 9, 8, 7."""
        ctx = _make_context("starter")

        for expected_remaining in [9, 8, 7]:
            resp = _make_response()
            await ctx.check_and_increment("ai_summary", fake_redis, resp)
            assert resp.headers["X-Usage-Remaining"] == str(expected_remaining), (
                f"Expected X-Usage-Remaining={expected_remaining}, "
                f"got {resp.headers.get('X-Usage-Remaining')!r}"
            )


    @pytest.mark.asyncio
    async def test_starter_tenth_call_remaining_zero(fake_redis):
        """Starter tenth call: counter = limit; header = X-Usage-Remaining: 0 (no error yet)."""
        ctx = _make_context("starter")
        limit = FEATURE_TIER_LIMITS["ai_summary"]["starter"]

        for _ in range(limit - 1):
            await ctx.check_and_increment("ai_summary", fake_redis, _make_response())

        resp = _make_response()
        await ctx.check_and_increment("ai_summary", fake_redis, resp)  # 10th call
        assert resp.headers["X-Usage-Remaining"] == "0"


    # ---------------------------------------------------------------------------
    # E06-P0-005: 429 body schema on limit exceeded
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_starter_eleventh_call_raises_429(fake_redis):
        """E06-P0-005 / AC6: 11th call raises 429 usage_limit_exceeded with correct body."""
        ctx = _make_context("starter")
        limit = FEATURE_TIER_LIMITS["ai_summary"]["starter"]  # 10

        # Exhaust quota
        for _ in range(limit):
            await ctx.check_and_increment("ai_summary", fake_redis, _make_response())

        resp = _make_response()
        with pytest.raises(AppException) as exc_info:
            await ctx.check_and_increment("ai_summary", fake_redis, resp)

        exc = exc_info.value
        assert exc.status_code == 429
        assert exc.error == "usage_limit_exceeded"
        assert exc.details is not None

        # Body fields per epic spec
        assert exc.details["limit"] == limit
        assert exc.details["used"] >= limit  # used ≥ limit when blocked
        assert exc.details["resets_at"]  # non-empty ISO8601 string
        assert re.match(r"\d{4}-\d{2}-\d{2}T", exc.details["resets_at"])
        assert exc.details["upgrade_url"] == _TEST_UPGRADE_URL

        # Header must also be set to "0" before raise
        assert resp.headers.get("X-Usage-Remaining") == "0"


    @pytest.mark.asyncio
    async def test_free_tier_first_call_raises_429(fake_redis):
        """Free tier (limit=0): first call immediately raises 429 with used=0, limit=0."""
        ctx = _make_context("free")
        resp = _make_response()

        with pytest.raises(AppException) as exc_info:
            await ctx.check_and_increment("ai_summary", fake_redis, resp)

        exc = exc_info.value
        assert exc.status_code == 429
        assert exc.error == "usage_limit_exceeded"
        assert exc.details["limit"] == 0
        assert exc.details["used"] == 0
        assert resp.headers.get("X-Usage-Remaining") == "0"


    # ---------------------------------------------------------------------------
    # Enterprise (unlimited)
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_enterprise_no_metering_no_header_no_exception(fake_redis):
        """AC4, AC5: Enterprise tier — no Redis key written, no header set, no exception."""
        ctx = _make_context("enterprise")
        resp = _make_response()

        await ctx.check_and_increment("ai_summary", fake_redis, resp)

        # No X-Usage-Remaining header for unlimited tier
        assert "X-Usage-Remaining" not in resp.headers

        # No Redis key created
        keys = await fake_redis.keys("user:*")
        assert len(keys) == 0, f"Enterprise should write no Redis keys; found: {keys}"


    @pytest.mark.asyncio
    async def test_enterprise_multiple_calls_no_exception(fake_redis):
        """Enterprise: 100 calls with no exception (unlimited quota)."""
        ctx = _make_context("enterprise")
        for _ in range(100):
            await ctx.check_and_increment("ai_summary", fake_redis, _make_response())


    # ---------------------------------------------------------------------------
    # Professional tier
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_professional_50th_call_succeeds_51st_raises(fake_redis):
        """Professional (limit=50): 50th call succeeds; 51st raises 429."""
        ctx = _make_context("professional")
        limit = FEATURE_TIER_LIMITS["ai_summary"]["professional"]  # 50

        for _ in range(limit):
            await ctx.check_and_increment("ai_summary", fake_redis, _make_response())

        with pytest.raises(AppException) as exc_info:
            await ctx.check_and_increment("ai_summary", fake_redis, _make_response())

        assert exc_info.value.status_code == 429
        assert exc_info.value.details["limit"] == limit


    # ---------------------------------------------------------------------------
    # E06-P1-008: INCR sets EXPIRE ≤ end-of-billing-period TTL
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_expire_set_to_billing_period_ttl(fake_redis):
        """E06-P1-008: After first INCR, Redis key TTL ≤ seconds remaining in billing period."""
        ctx = _make_context("starter")
        await ctx.check_and_increment("ai_summary", fake_redis, _make_response())

        key_pattern = f"user:{ctx.user_id}:usage:ai_summary:*"
        keys = await fake_redis.keys(key_pattern)
        assert len(keys) == 1, f"Expected 1 usage key; found: {keys}"

        ttl = await fake_redis.ttl(keys[0])
        assert ttl > 0, "TTL must be positive after first INCR"

        # TTL must not exceed seconds until end of current billing period
        now = datetime.now(UTC)
        _, expected_max_ttl, _ = _billing_period_ttl(now)
        assert ttl <= expected_max_ttl, (
            f"TTL {ttl}s exceeds billing period TTL {expected_max_ttl}s"
        )


    @pytest.mark.asyncio
    async def test_expire_not_reset_on_subsequent_incr(fake_redis):
        """EXPIRE is set only on first INCR (new_count == 1); not reset on subsequent calls."""
        ctx = _make_context("starter")

        await ctx.check_and_increment("ai_summary", fake_redis, _make_response())
        key = (await fake_redis.keys(f"user:{ctx.user_id}:usage:ai_summary:*"))[0]
        ttl_after_first = await fake_redis.ttl(key)

        await ctx.check_and_increment("ai_summary", fake_redis, _make_response())
        ttl_after_second = await fake_redis.ttl(key)

        # TTL should be roughly the same (slightly less due to elapsed time)
        # It must NOT be reset to the full billing period TTL
        assert ttl_after_second <= ttl_after_first, (
            "EXPIRE must not be reset on subsequent INCR calls"
        )


    # ---------------------------------------------------------------------------
    # Key pattern correctness
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_redis_key_contains_user_id_feature_and_billing_period(fake_redis):
        """AC3: Redis key pattern is user:{user_id}:usage:{feature}:{YYYY-MM}."""
        ctx = _make_context("starter")
        await ctx.check_and_increment("ai_summary", fake_redis, _make_response())

        keys = await fake_redis.keys("user:*")
        assert len(keys) == 1

        key = keys[0]
        expected_period = datetime.now(UTC).strftime("%Y-%m")
        assert str(ctx.user_id) in key
        assert "ai_summary" in key
        assert expected_period in key
        assert key == f"user:{ctx.user_id}:usage:ai_summary:{expected_period}"


    # ---------------------------------------------------------------------------
    # FEATURE_TIER_LIMITS completeness check
    # ---------------------------------------------------------------------------

    def test_feature_tier_limits_has_all_tiers():
        """AC4: FEATURE_TIER_LIMITS has entries for all four subscription tiers."""
        for feature, limits in FEATURE_TIER_LIMITS.items():
            for tier in ("free", "starter", "professional", "enterprise"):
                assert tier in limits, (
                    f"FEATURE_TIER_LIMITS['{feature}'] is missing tier '{tier}'"
                )
            assert limits["enterprise"] is None, (
                "Enterprise must be None (unlimited) in FEATURE_TIER_LIMITS"
            )
            assert limits["free"] == 0, (
                "Free tier must be 0 (blocked) in FEATURE_TIER_LIMITS"
            )
    ```

- [x] Task 3: Write concurrency integration test using testcontainers Redis (AC: 9)
  - [x] 3.1 Create `eusolicit-app/services/client-api/tests/integration/test_usage_gate_concurrency.py`:

    ```python
    """Concurrency integration test for UsageGateContext — Story S06.03 / E06-P0-004.

    Uses testcontainers Redis (NOT fakeredis) to test the atomic Lua script under
    concurrent load.  E06-R-002 risk mitigation: two concurrent requests at limit-1
    must result in exactly one success and one 429 — never two successes.

    Test ID covered:
        E06-P0-004  UsageGate atomic counter: concurrent requests at limit-1 (testcontainers)

    Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#e06-p0-004
    Risk:   eusolicit-docs/test-artifacts/test-design-epic-06.md#e06-r-002
    """
    from __future__ import annotations

    import asyncio
    import uuid
    from datetime import UTC, datetime
    from unittest.mock import MagicMock

    import pytest
    import pytest_asyncio
    import redis.asyncio as aioredis

    from client_api.core.usage_gate import (
        FEATURE_TIER_LIMITS,
        UsageGateContext,
        _billing_period_ttl,
    )
    from eusolicit_common.exceptions import AppException

    # ---------------------------------------------------------------------------
    # testcontainers Redis fixture (module-scoped for performance)
    # ---------------------------------------------------------------------------

    @pytest_asyncio.fixture(scope="module")
    async def redis_container_url():
        """Start a real Redis container for concurrency tests.

        Requires testcontainers-redis to be installed.
        Skips gracefully if Docker is unavailable.
        """
        try:
            from testcontainers.redis import RedisContainer  # noqa: PLC0415

            with RedisContainer("redis:7-alpine") as container:
                host = container.get_container_host_ip()
                port = container.get_exposed_port(6379)
                yield f"redis://{host}:{port}/3"
        except Exception as exc:
            pytest.skip(f"testcontainers Redis unavailable: {exc}")


    @pytest_asyncio.fixture
    async def real_redis(redis_container_url):
        """Async Redis client connected to testcontainers Redis instance."""
        client = aioredis.Redis.from_url(redis_container_url, decode_responses=True)
        yield client
        # Clean up all keys created during the test
        await client.flushdb()
        await client.aclose()


    # ---------------------------------------------------------------------------
    # E06-P0-004: Atomic race condition test
    # ---------------------------------------------------------------------------

    @pytest.mark.asyncio
    async def test_concurrent_requests_at_limit_minus_one(real_redis):
        """E06-P0-004 / AC9 / E06-R-002:

        Two concurrent check_and_increment calls when counter is at limit-1 (9 of 10
        for Starter tier) must result in EXACTLY ONE success and EXACTLY ONE 429.

        Failure mode (if Lua script not used):
          Both goroutines read count=9, both pass the limit check, both INCR → count=11.
          Two successes instead of one success + one 429.
        """
        ctx = UsageGateContext(
            user_id=uuid.uuid4(),
            user_tier="starter",
            _upgrade_url="https://eusolicit.test/billing/upgrade",
        )
        limit = FEATURE_TIER_LIMITS["ai_summary"]["starter"]  # 10

        # Pre-set counter to limit - 1 (9)
        now = datetime.now(UTC)
        billing_period, ttl, _ = _billing_period_ttl(now)
        counter_key = f"user:{ctx.user_id}:usage:ai_summary:{billing_period}"
        await real_redis.set(counter_key, limit - 1)
        await real_redis.expire(counter_key, ttl)

        def _make_response() -> MagicMock:
            resp = MagicMock()
            resp.headers = {}
            return resp

        # Fire two concurrent requests
        resp_a, resp_b = _make_response(), _make_response()
        results = await asyncio.gather(
            ctx.check_and_increment("ai_summary", real_redis, resp_a),
            ctx.check_and_increment("ai_summary", real_redis, resp_b),
            return_exceptions=True,
        )

        successes = [r for r in results if not isinstance(r, Exception)]
        failures = [r for r in results if isinstance(r, AppException)]

        # Exactly one succeeds, exactly one fails
        assert len(successes) == 1, (
            f"Expected 1 success from concurrent requests at limit-1; "
            f"got {len(successes)} successes. "
            f"This indicates the Lua script is not atomic — check _USAGE_LUA."
        )
        assert len(failures) == 1, (
            f"Expected 1 failure (429) from concurrent requests at limit-1; "
            f"got {len(failures)} failures."
        )
        assert failures[0].status_code == 429
        assert failures[0].error == "usage_limit_exceeded"

        # Counter must be exactly at limit (not limit+1)
        final_count = int(await real_redis.get(counter_key))
        assert final_count == limit, (
            f"Counter must be exactly {limit} after one success + one block; "
            f"got {final_count}. Race condition detected!"
        )


    @pytest.mark.asyncio
    async def test_burst_of_concurrent_requests_respects_limit(real_redis):
        """Fire N concurrent requests where N > limit; verify exactly `limit` succeed.

        This stress-tests the Lua script under higher concurrency than E06-P0-004.
        """
        ctx = UsageGateContext(
            user_id=uuid.uuid4(),
            user_tier="starter",
            _upgrade_url="https://eusolicit.test/billing/upgrade",
        )
        limit = FEATURE_TIER_LIMITS["ai_summary"]["starter"]  # 10
        num_concurrent = limit + 5  # 15 requests for 10-slot limit

        def _make_response() -> MagicMock:
            resp = MagicMock()
            resp.headers = {}
            return resp

        responses = [_make_response() for _ in range(num_concurrent)]
        results = await asyncio.gather(
            *[ctx.check_and_increment("ai_summary", real_redis, r) for r in responses],
            return_exceptions=True,
        )

        successes = [r for r in results if not isinstance(r, Exception)]
        failures = [r for r in results if isinstance(r, AppException)]

        assert len(successes) == limit, (
            f"Expected exactly {limit} successes for limit={limit}; "
            f"got {len(successes)}. Race condition or off-by-one error."
        )
        assert len(failures) == num_concurrent - limit
    ```

## Dev Notes

### Architecture Context

S06.03 is the **usage quota enforcement layer** for Epic 6. It sits orthogonal to `OpportunityTierGate` (field visibility + scope) — both apply to metered endpoints like S06.08, but they enforce different dimensions:

| Gate | Dimension | Enforcement |
|------|-----------|-------------|
| `OpportunityTierGate` | Field visibility + geographic/budget scope | Per-item 403 or field masking |
| `UsageGate` | Per-user monthly quota for metered features | Per-request 429 with counter |

The two dependencies compose on the AI summary endpoint (S06.08):
```python
@router.post("/ai-summary")
async def generate_ai_summary(
    opportunity_id: UUID,
    current_user: Annotated[CurrentUser, Depends(get_current_user)],
    tier_gate: Annotated[OpportunityTierGateContext, Depends(get_opportunity_tier_gate)],
    usage_gate: Annotated[UsageGateContext, Depends(get_usage_gate)],
    redis: Annotated[Redis, Depends(get_redis_client)],
    response: Response,
    session: Annotated[AsyncSession, Depends(get_db_session)],
):
    # 1. TierGate: free users can't reach AI summary (detail endpoint already blocked)
    # 2. UsageGate: check + decrement before calling AI Gateway
    await usage_gate.check_and_increment("ai_summary", redis, response)
    ...
```

FastAPI's DI system caches `get_opportunity_tier_gate` per request, so the subscription DB query runs only once even when both `get_usage_gate` and the route handler inject `OpportunityTierGateContext`.

### CRITICAL: Why Lua Script, Not Separate GET + INCR (E06-R-002)

**NEVER implement usage metering as two separate Redis commands:**
```python
# WRONG — race condition E06-R-002:
count = await redis.get(key)
if count < limit:
    await redis.incr(key)
```

Two concurrent requests both read `count=9`, both pass `9 < 10`, both INCR to 10 and 11. Result: 11 summaries consumed against a 10-limit quota.

**The `_USAGE_LUA` script executes GET + conditional INCR as a single atomic Redis transaction.** Redis executes Lua scripts without interruption between any other commands. This is the standard Redis pattern for atomic check-and-modify operations.

**MULTI/EXEC is insufficient here** because it requires WATCH to guard against concurrent modification, which is complex with `redis.asyncio` and error-prone. The Lua script is simpler, well-supported, and fully atomic.

### CRITICAL: AppException for 429 (No `TooManyRequestsError` Subclass)

`eusolicit_common.exceptions` does not have a `TooManyRequestsError` subclass. The correct pattern (established by S06.02 using `AppException` directly for `tier_limit`) is:

```python
raise AppException(
    "Usage limit exceeded for this billing period.",
    error="usage_limit_exceeded",
    status_code=429,
    details={
        "limit": limit,
        "used": current_count,
        "resets_at": resets_at,
        "upgrade_url": self._upgrade_url,
    },
)
```

The `app_exception_handler` in `eusolicit_common.exceptions` handles this and returns:
```json
{
  "error": "usage_limit_exceeded",
  "message": "Usage limit exceeded for this billing period.",
  "details": {"limit": 10, "used": 10, "resets_at": "2026-05-01T00:00:00+00:00", "upgrade_url": "..."},
  "correlation_id": "..."
}
```

**In integration tests, assert `response.json()["details"]["limit"]`, NOT `response.json()["limit"]`** — the epic spec shorthand `{"error": "...", "limit": N, ...}` is documentation notation; in the actual response body, these fields are inside `details`.

### CRITICAL: `redis.eval()` Call Signature (redis-py async)

The correct way to call `eval` with `redis.asyncio`:

```python
result = await redis.eval(
    _USAGE_LUA,
    1,          # numkeys (number of KEYS[] arguments)
    key,        # KEYS[1]
    str(limit), # ARGV[1] — must be string, not int
    str(ttl),   # ARGV[2] — must be string, not int
)
```

`result` is a list of two integers `[count, exceeded_flag]`. Cast with `int()` before comparison since redis-py may return them as strings or bytes depending on `decode_responses` setting.

With `decode_responses=True` (as set in `get_redis_client()`), values are strings. With fakeredis, behaviour may differ — always cast with `int(result[0])`.

### CRITICAL: `get_redis_client()` Is Already Available

`dependencies.py` already exports `get_redis_client()` which returns a singleton `redis.asyncio.Redis` client configured via `CLIENT_API_REDIS_URL` (inherited from `BaseServiceSettings.redis_url`). Do not create a new Redis client — use the existing dependency:

```python
from client_api.dependencies import get_redis_client
```

The `get_usage_gate` dependency does NOT inject Redis directly — Redis is passed to `check_and_increment()` by the calling route handler. This keeps the context object testable without a Redis fixture (unit tests pass fakeredis directly to `check_and_increment()`).

### CRITICAL: fakeredis for Unit Tests, testcontainers for Concurrency Tests

- `fakeredis.aioredis.FakeRedis(decode_responses=True)` — for all serial unit tests in `tests/unit/test_usage_gate.py`
- Real Redis via `testcontainers.redis.RedisContainer` — ONLY for concurrency tests in `tests/integration/test_usage_gate_concurrency.py` (E06-P0-004 and the burst test)
- **Do NOT use fakeredis for E06-P0-004** — per E06-R-010, fakeredis's concurrent atomicity may differ from real Redis; testcontainers Redis is required for the race condition test

### CRITICAL: `X-Usage-Remaining` Header Value

- On success: `X-Usage-Remaining = limit - new_count` (remaining usages after this call)
  - After 1st call (Starter): `10 - 1 = 9`
  - After 10th call (Starter): `10 - 10 = 0`
- On 429: `X-Usage-Remaining = 0` (set BEFORE raising the exception)
- For Enterprise (unlimited): header NOT set at all

### Billing Period and TTL Calculation

- **Billing period**: `YYYY-MM` (e.g., `"2026-04"`) — calendar month in UTC
- **TTL**: `int((first_day_of_next_month - now).total_seconds())` — minimum 1 second
- **resets_at**: ISO8601 UTC datetime string of `datetime(year, month+1, 1, tzinfo=UTC)` (or `datetime(year+1, 1, 1)` for December)
- Use `_billing_period_ttl(datetime.now(UTC))` from `usage_gate.py` — this is also exported for testing

### S06.02 Review Patches — Apply Here Too

Two patches were deferred from S06.02 code review that affect `opportunity_tier_gate.py`. They are recorded in `eusolicit-docs/implementation-artifacts/deferred-work.md`. The dev agent implementing S06.03 should check `deferred-work.md` and apply any patch items that are not yet done before committing.

### Interaction with S06.08 (AI Summary)

S06.03 provides `UsageGateContext` + `get_usage_gate` dependency. S06.08 is responsible for:
1. Injecting `usage_gate: UsageGateContext = Depends(get_usage_gate)` and `redis = Depends(get_redis_client)` in the AI summary route
2. Calling `await usage_gate.check_and_increment("ai_summary", redis, response)` BEFORE calling the AI Gateway (not after)
3. Skipping `check_and_increment` when returning a cached summary (< 24h) — this is the E06-P0-008 "no decrement for cached summaries" behavior
4. Writing the completed summary to `client.ai_summaries` table (S06.08 concern, not S06.03)

**S06.03 does NOT add any new API endpoint.** It only provides the reusable `UsageGateContext` and `get_usage_gate` dependency.

### File Locations

| Purpose | Path |
|---------|------|
| UsageGate dependency + context | `eusolicit-app/services/client-api/src/client_api/core/usage_gate.py` |
| Unit tests (fakeredis, serial) | `eusolicit-app/services/client-api/tests/unit/test_usage_gate.py` |
| Concurrency integration tests (testcontainers) | `eusolicit-app/services/client-api/tests/integration/test_usage_gate_concurrency.py` |
| Redis client dependency (existing) | `eusolicit-app/services/client-api/src/client_api/dependencies.py` |
| Config (redis_url already in BaseServiceSettings) | `eusolicit-app/services/client-api/src/client_api/config.py` |
| Common exceptions (AppException) | `eusolicit-app/packages/eusolicit-common/src/eusolicit_common/exceptions.py` |

### Project Structure Notes

- App root is `eusolicit-app/` (not project root) — all Python service paths begin with `eusolicit-app/`
- The `core/` module holds request-scoped dependencies: `security.py`, `tier_gate.py`, `opportunity_tier_gate.py`, `rbac.py`, `rate_limit.py` — add `usage_gate.py` here
- `tests/unit/` and `tests/unit/__init__.py` already exist (created in S06.02)
- `tests/integration/` and `tests/integration/__init__.py` already exist
- `tests/api/` already exists — use `tests/integration/` for the concurrency test (it requires testcontainers, not the HTTP client)

### Test Design Traceability

| Test ID | Priority | Scenario | AC |
|---------|----------|----------|----|
| E06-P0-004 | P0 | Concurrent race: two requests at limit-1 → exactly one 200, one 429 (testcontainers Redis) | AC2, AC9 |
| E06-P0-005 | P0 | 429 body schema: error, details.limit, details.used, details.resets_at, details.upgrade_url, X-Usage-Remaining: 0 | AC6 |
| E06-P0-008 | P0 | Cached AI summary (< 24h) returned without decrement — S06.08 responsibility, depends on S06.03 | AC1, AC2 |
| E06-P1-008 | P1 | INCR sets EXPIRE ≤ end-of-billing-period TTL (fakeredis) | AC3 |
| E06-P1-009 | P1 | X-Usage-Remaining decrements correctly across sequential calls (9, 8, 7 for Starter) | AC5 |
| E06-P3-004 | P3 | fakeredis serial parity — sequential counter values correct | AC2, AC5 |

Risk mitigations verified by this story:
- **E06-R-002** (Redis counter race — score 6): Lua script atomicity verified by E06-P0-004 with testcontainers Redis; unit tests verify header and 429 schema

### References

- [Source: eusolicit-docs/planning-artifacts/epic-06-opportunity-discovery.md#s0603] — S06.03 story spec with Redis key, Lua pipeline requirement, 429 body format
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#e06-r-002] — Redis race condition risk (score 6) and Lua script mitigation strategy
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#e06-p0-004] — P0 concurrent race test spec (testcontainers, asyncio.gather)
- [Source: eusolicit-docs/test-artifacts/test-design-epic-06.md#e06-p0-005] — P0 usage limit 429 body/header test spec
- [Source: eusolicit-docs/implementation-artifacts/6-2-tier-gated-response-serialization.md] — AppException pattern for custom error slugs, OpportunityTierGateContext dependency pattern
- [Source: eusolicit-app/services/client-api/src/client_api/dependencies.py] — get_redis_client() singleton, decode_responses=True
- [Source: eusolicit-app/services/client-api/src/client_api/core/rate_limit.py] — Existing Redis usage pattern (LoginRateLimiter)
- [Source: eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py] — Template for FastAPI dependency + dataclass pattern

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.5 (claude-sonnet-4-5)

### Debug Log References

- fakeredis `EVAL` required `lupa` package (Lua engine) — `fakeredis[lua]` extra added to pyproject.toml instead of `fakeredis[aioredis]`. The `lupa` library is auto-detected by fakeredis when installed.

### Completion Notes List

1. **Usage gate implementation**: Created `core/usage_gate.py` exactly per story spec with `UsageGateContext` dataclass, `_USAGE_LUA` atomic Lua script, `FEATURE_TIER_LIMITS`, `_billing_period_ttl` helper, and `get_usage_gate` FastAPI dependency. All 10 acceptance criteria satisfied.

2. **Test activation**: Removed all `@pytest.mark.skip` decorators from pre-written ATDD test files (22 unit tests, 3 integration tests). Both test files were already written in ATDD red phase and simply needed the skip decorators removed.

3. **fakeredis Lua support**: The `fakeredis[aioredis]` extra does not include Lua support. Changed to `fakeredis[lua]>=2.21` in pyproject.toml dev dependencies to install `lupa` which enables `EVAL` command support in fakeredis.

4. **S06.02 deferred patch applied**: Applied `field(repr=False)` to `_upgrade_url` in `opportunity_tier_gate.py` as tracked in `deferred-work.md`. The `from dataclasses import dataclass, field` import was updated and `_upgrade_url: str` became `_upgrade_url: str = field(repr=False)`. All 42 existing tier gate tests continue to pass.

5. **Integration concurrency tests**: Used testcontainers Redis for E06-P0-004 atomic race guard test. Two concurrent requests at `limit-1` result in exactly one success and one 429 — verifying Lua script atomicity against E06-R-002 race condition.

6. **No deviations detected**: Implementation matches story spec exactly. No scope creep, no missing requirements, no contradictory spec items.

### File List

- `eusolicit-app/services/client-api/src/client_api/core/usage_gate.py` — **Created** (new): UsageGateContext dataclass, _USAGE_LUA Lua script, FEATURE_TIER_LIMITS, _billing_period_ttl helper, get_usage_gate FastAPI dependency
- `eusolicit-app/services/client-api/tests/unit/test_usage_gate.py` — **Modified**: removed @pytest.mark.skip decorators (ATDD red → green), cleaned up _SKIP_REASON variable, updated phase comment
- `eusolicit-app/services/client-api/tests/integration/test_usage_gate_concurrency.py` — **Modified**: removed @pytest.mark.skip decorators (ATDD red → green), cleaned up _SKIP_REASON variable, updated phase comment
- `eusolicit-app/services/client-api/pyproject.toml` — **Modified**: added `fakeredis[lua]>=2.21` and `testcontainers[redis]>=4.7` to dev extras
- `eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py` — **Modified** (S06.02 patch): `_upgrade_url: str = field(repr=False)` to hide upgrade URL from repr()/asdict()

## Senior Developer Review

**Date:** 2026-04-17
**Reviewer:** Code Review Agent (bmad-code-review)
**Verdict:** APPROVE

### Review Findings

- [x] [Review][Patch] Unused import `calendar` — dead import, never referenced in executable code [usage_gate.py:31] — trivial cleanup, non-blocking
- [x] [Review][Patch] Unused import `get_redis_client` — imported but never used (only appears in a docstring) [usage_gate.py:48] — trivial cleanup, non-blocking
- [x] [Review][Defer] Unknown feature/tier → fail-open to unlimited — `FEATURE_TIER_LIMITS.get(feature, {}).get(tier)` returns `None` for unknown keys, same sentinel as Enterprise (unlimited). Inputs are developer-controlled (feature is a literal, tier is normalized upstream). Defensive `ValueError` for unknown feature would harden, but risk is theoretical. — deferred, defensive hardening
- [x] [Review][Defer] No Redis connection error handling around `redis.eval()` — `ConnectionError`/`TimeoutError` propagates as raw 500. A `try/except` with 503 response would be more graceful, but this is a cross-cutting infrastructure concern (same pattern exists in `rate_limit.py`). — deferred, pre-existing pattern
- [x] [Review][Defer] Professional tier test has redundant first loop — `test_professional_50th_call_succeeds_51st_raises` creates two contexts with different UUIDs; the first 50 calls with `ctx` are never used. — deferred, cosmetic test issue

### Triage Summary

| Category | Count |
|----------|-------|
| Patch (non-blocking) | 2 |
| Defer | 3 |
| Dismissed (noise) | 5 |

### Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: UsageGateContext dataclass | ✅ Pass | `@dataclass` with `user_id: UUID`, `user_tier: str`, `_upgrade_url: str`; async `check_and_increment` method |
| AC2: Atomic Lua script | ✅ Pass | `_USAGE_LUA` module constant; GET+conditional INCR in single eval; EXPIRE on new_count==1 |
| AC3: Redis key pattern + TTL | ✅ Pass | `user:{user_id}:usage:{feature}:{YYYY-MM}`; TTL = seconds to end of billing month; tested |
| AC4: Tier limits constant | ✅ Pass | `FEATURE_TIER_LIMITS` with free=0, starter=10, professional=50, enterprise=None |
| AC5: X-Usage-Remaining header on success | ✅ Pass | `limit - new_count`; enterprise: no header. Tested with sequential decrement assertions |
| AC6: 429 on limit exceeded | ✅ Pass | `AppException(error="usage_limit_exceeded", status_code=429, details={limit, used, resets_at, upgrade_url})` |
| AC7: get_usage_gate dependency | ✅ Pass | Reuses `OpportunityTierGateContext` + `CurrentUser`; no second DB query; correct upgrade URL |
| AC8: Unit tests (fakeredis) | ✅ Pass | 22 tests covering all specified scenarios; all green |
| AC9: Concurrency test (testcontainers) | ✅ Pass | E06-P0-004 race guard + burst test + enterprise concurrent; testcontainers Redis |
| AC10: __all__ exports | ✅ Pass | `["UsageGateContext", "get_usage_gate", "FEATURE_TIER_LIMITS"]` |

### Architecture Alignment

- ✅ Follows established `core/` dependency pattern (matches `opportunity_tier_gate.py`, `rate_limit.py`)
- ✅ Uses existing `get_redis_client()` singleton from `dependencies.py` (referenced in docs, injected by route handlers)
- ✅ Lua script prevents E06-R-002 race condition (validated by testcontainers integration test)
- ✅ `AppException` usage matches S06.02 pattern (no 429 subclass needed)
- ✅ S06.02 deferred patch applied (`field(repr=False)` on `_upgrade_url`)
- ✅ Deferred work items from S02.03 (non-atomic INCR+EXPIRE) explicitly avoided via Lua script design
