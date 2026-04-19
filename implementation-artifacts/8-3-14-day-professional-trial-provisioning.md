# Story 8.3: 14-Day Professional Trial Provisioning

Status: review

## Story

As a platform engineer,
I want a 14-day Professional trial subscription to be automatically provisioned when a new company registers (no payment method required),
so that new users can experience Professional-tier features immediately without any friction or upfront billing commitment.

## Acceptance Criteria

1. **AC1** — When a new company registers, a Stripe subscription is created on the Professional price with `trial_end` set to `now() + 14 days` (Unix timestamp), `payment_behavior="default_incomplete"` (no payment method required), and the customer set to the `stripe_customer_id` provisioned by Story 8.1. The call uses idempotency key `f"trial-provisioning-{company_id}"`.

2. **AC2** — The local `client.subscriptions` row is updated atomically: `stripe_subscription_id=<from Stripe>`, `tier="professional"`, `status="trialing"`, `trial_start=datetime.now(UTC)`, `trial_end=datetime.now(UTC)+timedelta(days=14)`, `is_trial=True`, `started_at=datetime.now(UTC)`. The existing `stripe_customer_id` (set by Story 8.1) MUST remain intact (use `existing.stripe_subscription_id = ...` and `session.flush()` on the existing row, not a new INSERT).

3. **AC3** — One-trial-per-company is enforced: if `client.subscriptions` already has a non-null `stripe_subscription_id`, trial provisioning is skipped with structlog INFO `"trial_already_provisioned"`. No second Stripe subscription is created.

4. **AC4** — A `subscription.changed` event is published to Redis Streams (stream key `"subscription.changed"`, `event_type="SubscriptionChanged"`) with payload `{"company_id": str(company_id), "old_tier": "free", "new_tier": "professional", "timestamp": <ISO8601>}` via `EventPublisher` from `eusolicit_common.events.publisher`. The event is published **after** `session.commit()` to avoid publishing for rolled-back transactions.

5. **AC5** — Graceful degradation: if `stripe_customer_id` is None (Story 8.1 customer provisioning failed), trial provisioning is skipped with WARNING `"trial_provisioning_skipped_no_customer"`. If `CLIENT_API_STRIPE_PROFESSIONAL_PRICE_ID` is not configured, trial provisioning is skipped with WARNING `"trial_provisioning_skipped_no_price_id"`. Stripe API failures are caught, logged at ERROR (`"trial_provisioning_failed"`), and swallowed (registration response already sent).

6. **AC6** — `auth.py` is updated to call `provision_new_company_billing_bg()` (the new combined function) instead of `provision_stripe_customer_bg()`. The combined function runs both customer creation (Story 8.1 logic) and trial provisioning (this story's logic) in sequence within a single background task and single DB session. `provision_stripe_customer_bg()` is retained in `billing_service.py` but deprecated (marked in docstring).

7. **AC7** — `core/tier_gate.py` is updated: all three gate functions (`require_paid_tier`, `require_professional_plus_tier`, `require_enterprise_tier`) change `.where(Subscription.status == "active")` to `.where(Subscription.status.in_(_ALLOWED_STATUSES))` where `_ALLOWED_STATUSES: list[str] = ["active", "trialing"]` is a module-level constant. This allows trial Professional users to access Professional-tier features immediately. Module docstring and inline comments updated accordingly.

8. **AC8** — Unit tests cover: happy path (trial created, DB updated, event published), idempotency (existing stripe_subscription_id → skip, no Stripe call), no stripe_customer_id (skip with WARNING), no price ID configured (skip with WARNING), Stripe API error (logged, swallowed, no exception propagated), tier_gate allows `status="trialing"` users through all three gates. Integration test verifies full registration → background task → DB state (tier="professional", status="trialing", stripe_subscription_id set).

## Tasks / Subtasks

- [x] Task 1 — Add `stripe_professional_price_id` to `ClientApiSettings` (AC: 1, 5)
  - [x] 1.1 Edit `services/client-api/src/client_api/config.py` — add after `stripe_api_version`:
    ```python
    stripe_professional_price_id: str | None = None  # price_... for 14-day trial subscription
    ```
    Env var: `CLIENT_API_STRIPE_PROFESSIONAL_PRICE_ID=price_...` (populated from Stripe Dashboard product/price for Professional tier).
    Missing value must NOT crash the app — skip trial provisioning with WARNING (AC5).

- [x] Task 2 — Implement `provision_professional_trial()` in `billing_service.py` (AC: 1, 2, 3, 5)
  - [x] 2.1 Add imports to `billing_service.py`:
    ```python
    from datetime import UTC, datetime, timedelta
    from eusolicit_common.events.publisher import EventPublisher
    from client_api.config import get_settings
    ```
  - [x] 2.2 Implement core trial provisioning function:
    ```python
    async def provision_professional_trial(
        company_id: UUID,
        stripe_customer_id: str | None,
        session: AsyncSession,
    ) -> bool:
        """Provision a 14-day Professional trial Stripe subscription.

        AC3: Enforces one-trial-per-company by checking for existing stripe_subscription_id.
        AC5: Skips gracefully when stripe_customer_id is None or price ID is not configured.
        Returns True if trial was provisioned (caller should publish subscription.changed event),
        False if skipped or failed.

        Must be called AFTER provision_stripe_customer() so stripe_customer_id is available.
        """
        # AC5: Skip if no customer ID (Story 8.1 failed silently)
        if not stripe_customer_id:
            logger.warning(
                "trial_provisioning_skipped_no_customer",
                company_id=str(company_id),
            )
            return False

        settings = get_settings()
        # AC5: Skip if Professional price ID is not configured
        if not settings.stripe_professional_price_id:
            logger.warning(
                "trial_provisioning_skipped_no_price_id",
                company_id=str(company_id),
            )
            return False

        # AC3: Idempotency — check for existing trial subscription
        result = await session.execute(
            select(Subscription).where(Subscription.company_id == company_id)
        )
        existing = result.scalar_one_or_none()

        if existing is not None and existing.stripe_subscription_id:
            logger.info(
                "trial_already_provisioned",
                company_id=str(company_id),
                stripe_subscription_id=existing.stripe_subscription_id,
            )
            return False

        if existing is None:
            logger.error(
                "trial_provisioning_failed_no_subscription_row",
                company_id=str(company_id),
                msg="subscription row must exist before trial provisioning (Story 8.1 creates it)",
            )
            return False

        # AC1: Call Stripe API (sync SDK wrapped in executor for async safety)
        now = datetime.now(UTC)
        trial_end_ts = int((now + timedelta(days=14)).timestamp())

        try:
            stripe_sub = await asyncio.to_thread(
                stripe.Subscription.create,
                customer=stripe_customer_id,
                items=[{"price": settings.stripe_professional_price_id}],
                trial_end=trial_end_ts,
                payment_behavior="default_incomplete",
                payment_settings={"save_default_payment_method": "on_subscription"},
                metadata={"company_id": str(company_id)},
                idempotency_key=f"trial-provisioning-{company_id}",
            )
        except stripe.error.StripeError as exc:
            logger.error(
                "trial_provisioning_failed",
                company_id=str(company_id),
                error=str(exc),
                error_type=type(exc).__name__,
            )
            return False

        # AC2: Update local subscription row
        trial_end_dt = now + timedelta(days=14)
        existing.stripe_subscription_id = stripe_sub.id
        existing.tier = "professional"
        existing.status = "trialing"
        existing.trial_start = now
        existing.trial_end = trial_end_dt
        existing.is_trial = True
        existing.started_at = now
        await session.flush()

        logger.info(
            "trial_provisioned",
            company_id=str(company_id),
            stripe_subscription_id=stripe_sub.id,
            trial_end=trial_end_dt.isoformat(),
        )

        # AC8: Emit audit trail (uses caller's session per audit_service.py docstring)
        await write_audit_entry(
            session,
            action_type="billing.trial_provisioned",
            entity_type="subscription",
            entity_id=existing.id,
            after={
                "tier": "professional",
                "status": "trialing",
                "trial_end": trial_end_dt.isoformat(),
                "stripe_subscription_id": stripe_sub.id,
            },
            company_id=company_id,
        )

        return True  # Signal caller to publish subscription.changed event
    ```

- [x] Task 3 — Implement `provision_new_company_billing_bg()` in `billing_service.py` (AC: 4, 6)
  - [x] 3.1 Add the combined background task entry point that sequences customer + trial provisioning and publishes the Redis Streams event after DB commit:
    ```python
    async def provision_new_company_billing_bg(
        company_id: UUID,
        company_name: str,
        user_email: str,
    ) -> None:
        """Combined background task: Stripe customer provisioning + trial provisioning.

        Replaces provision_stripe_customer_bg() as the registration hook.
        Sequences both operations in a single DB session for efficiency.
        Publishes subscription.changed Redis Streams event AFTER commit (AC4).

        Called by FastAPI BackgroundTasks after HTTP 201 response is sent.
        """
        from client_api.dependencies import get_redis_client  # avoid circular at module level

        session_factory = get_session_factory()
        redis_client = get_redis_client()
        trial_provisioned = False

        async with session_factory() as session:
            try:
                # Step 1: Provision Stripe customer (Story 8.1 logic)
                stripe_customer_id = await provision_stripe_customer(
                    company_id, company_name, user_email, session
                )
                # Step 2: Provision 14-day Professional trial (Story 8.3 logic)
                trial_provisioned = await provision_professional_trial(
                    company_id, stripe_customer_id, session
                )
                await session.commit()
            except Exception as exc:  # noqa: BLE001
                await session.rollback()
                logger.exception(
                    "new_company_billing_bg_failed",
                    company_id=str(company_id),
                    error=str(exc),
                    error_type=type(exc).__name__,
                )
                return

        # AC4: Publish subscription.changed AFTER commit (never for rolled-back transactions)
        if trial_provisioned:
            publisher = EventPublisher(redis_client)
            try:
                await publisher.publish(
                    stream="subscription.changed",
                    event_type="SubscriptionChanged",
                    payload={
                        "company_id": str(company_id),
                        "old_tier": "free",
                        "new_tier": "professional",
                        "timestamp": datetime.now(UTC).isoformat(),
                    },
                    source_service="client-api",
                    tenant_id=str(company_id),
                )
                logger.info(
                    "subscription_changed_event_published",
                    company_id=str(company_id),
                    new_tier="professional",
                )
            except Exception as exc:  # noqa: BLE001
                # Event publishing failure is non-fatal — tier cache will fall back to DB
                logger.warning(
                    "subscription_changed_event_publish_failed",
                    company_id=str(company_id),
                    error=str(exc),
                    error_type=type(exc).__name__,
                )
    ```

- [x] Task 4 — Update `auth.py` to use `provision_new_company_billing_bg()` (AC: 6)
  - [x] 4.1 Edit `services/client-api/src/client_api/api/v1/auth.py`:
    - Replace the import: `from client_api.services.billing_service import provision_stripe_customer_bg`
      with: `from client_api.services.billing_service import provision_new_company_billing_bg`
    - Update the `background_tasks.add_task(...)` call in the `register()` endpoint:
      ```python
      background_tasks.add_task(
          provision_new_company_billing_bg,      # ← was provision_stripe_customer_bg
          response.company.id,
          request.company_name,
          str(request.email),
      )
      ```
    - Update the endpoint docstring to mention both customer and trial provisioning:
      ```
      Post-registration: enqueues Stripe customer + trial provisioning as a BackgroundTask
      (non-blocking — runs after HTTP 201 response is sent). Story 8-1/8-3.
      ```

- [x] Task 5 — Update `core/tier_gate.py` to allow trialing status (AC: 7)
  - [x] 5.1 Edit `services/client-api/src/client_api/core/tier_gate.py`:
    - Update module docstring: replace `AND status = 'active'` with `AND status IN ('active', 'trialing')` in all three dependency chain descriptions.
    - Add `_ALLOWED_STATUSES` module-level constant after the precomputed list constants (after `_ENTERPRISE_TIERS_LIST`):
      ```python
      # Story 8-3: Allow 'trialing' status — trial Professional users must access paid features.
      # Status 'active' = paid subscription confirmed by invoice.
      # Status 'trialing' = 14-day trial, no payment method required yet.
      _ALLOWED_STATUSES: list[str] = ["active", "trialing"]
      ```
    - In `require_paid_tier()`: replace `.where(Subscription.status == "active")` with `.where(Subscription.status.in_(_ALLOWED_STATUSES))`
    - In `require_professional_plus_tier()`: same replacement.
    - In `require_enterprise_tier()`: same replacement.
    - Update the inline docstring/comments in each function to reflect the new status filter.
    - Update `__all__` to export `_ALLOWED_STATUSES` if not already (it starts with `_` so it's private — do NOT export, just add it as module-level constant).
    - **IMPORTANT**: Also update the `opportunity_tier_gate.py` `.where(Subscription.status == "active")` filter to `.where(Subscription.status.in_(["active", "trialing"]))`. Add a local `_ALLOWED_STATUSES = ["active", "trialing"]` constant there as well (or import from tier_gate).

- [x] Task 6 — Unit tests for trial provisioning (AC: 8)
  - [x] 6.1 Create `services/client-api/tests/unit/test_trial_provisioning.py`:
    - `test_trial_provisioned_creates_stripe_subscription_and_updates_db` — mock `stripe.Subscription.create`, seed existing subscription row (with `stripe_customer_id` set, `stripe_subscription_id=None`), call `provision_professional_trial(company_id, "cus_test", session)` → verify `stripe.Subscription.create` called with correct params, `existing.stripe_subscription_id` set, `existing.tier == "professional"`, `existing.status == "trialing"`, `existing.is_trial is True`, returns `True`.
    - `test_trial_provisioning_idempotent_existing_subscription` — seed subscription with `stripe_subscription_id="sub_existing"` → verify Stripe NOT called, returns `False`.
    - `test_trial_provisioning_skips_when_no_customer_id` — call with `stripe_customer_id=None` → verify WARNING logged, Stripe NOT called, returns `False`.
    - `test_trial_provisioning_skips_when_no_price_id` — mock settings with `stripe_professional_price_id=None` → verify WARNING logged, Stripe NOT called, returns `False`.
    - `test_trial_provisioning_stripe_error_logged_and_swallowed` — mock `stripe.Subscription.create` raises `stripe.error.StripeError` → verify ERROR logged, no exception propagated, returns `False`.
    - `test_trial_provisioning_uses_correct_idempotency_key` — verify `stripe.Subscription.create` called with `idempotency_key=f"trial-provisioning-{company_id}"`.
    - `test_trial_provisioning_trial_end_is_14_days_from_now` — verify `trial_end` Unix timestamp passed to Stripe is within 1 second of `now + 14 days`.
    - `test_trial_provisioning_skips_when_no_subscription_row` — no existing row in DB → verify ERROR logged (missing row), returns `False`.

  - [x] 6.2 Create or update `services/client-api/tests/unit/test_tier_gate_trial.py` (new tests for trialing status):
    - `test_require_paid_tier_allows_trialing_status` — mock session returning a row with `tier="professional"` and `status="trialing"` → verify no ForbiddenError raised.
    - `test_require_professional_plus_tier_allows_trialing_status` — same for Professional+ gate.
    - `test_require_enterprise_tier_allows_trialing_status` — same for Enterprise gate.
    - `test_allowed_statuses_constant_contains_active_and_trialing` — assert `_ALLOWED_STATUSES == ["active", "trialing"]` (or contains both).

  - [x] 6.3 Create `services/client-api/tests/unit/test_new_company_billing_bg.py`:
    - `test_provision_new_company_billing_bg_calls_customer_then_trial` — mock `provision_stripe_customer` returns `"cus_test"`, mock `provision_professional_trial` returns `True` → verify both called in sequence, session committed.
    - `test_provision_new_company_billing_bg_publishes_event_after_commit` — verify `EventPublisher.publish()` called after `session.commit()` when trial_provisioned=True.
    - `test_provision_new_company_billing_bg_no_event_when_trial_skipped` — mock `provision_professional_trial` returns `False` → verify `EventPublisher.publish()` NOT called.
    - `test_provision_new_company_billing_bg_rollback_on_exception` — mock `provision_stripe_customer` raises exception → verify `session.rollback()` called, exception logged, no event published.

- [x] Task 7 — Integration test for full registration → trial flow (AC: 8)
  - [x] 7.1 Create `services/client-api/tests/integration/test_trial_provisioning_flow.py`:
    - `test_register_triggers_trial_provisioning` — mock both `stripe.Customer.create` (returns `MagicMock(id="cus_test")`) and `stripe.Subscription.create` (returns `MagicMock(id="sub_test")`), mock Redis `xadd`, POST to `POST /api/v1/auth/register` → wait for BackgroundTask → query DB and verify:
      - `subscription.stripe_customer_id == "cus_test"`
      - `subscription.stripe_subscription_id == "sub_test"`
      - `subscription.tier == "professional"`
      - `subscription.status == "trialing"`
      - `subscription.is_trial is True`
      - `subscription.trial_start is not None`
      - `subscription.trial_end is not None`
      - `subscription.started_at is not None`
    - `test_second_registration_attempt_does_not_duplicate_subscription` — seed existing subscription with `stripe_subscription_id="sub_existing"`, call `provision_new_company_billing_bg` → verify `stripe.Subscription.create` NOT called second time.

## Dev Notes

### Critical Architecture Constraints

- **BackgroundTasks only — no Celery workers.** Project rule from CLAUDE.md: "No Celery workers — async tasks use FastAPI BackgroundTasks or Celery Beat for scheduling only." The combined billing task runs as `background_tasks.add_task(provision_new_company_billing_bg, ...)`. [Source: CLAUDE.md#Critical Conventions]
- **Stripe SDK is sync — always use `asyncio.to_thread()`.** Python 3.12+ canonical idiom (established in Story 8.1). Never use `get_event_loop().run_in_executor()` — it is deprecated. Pattern: `await asyncio.to_thread(stripe.Subscription.create, **kwargs)`. [Source: story 8-1-stripe-customer-provisioning.md#Dev Notes]
- **Background task MUST own its DB session.** Do NOT share the request-scoped `get_db_session()` session. Use `get_session_factory()` to create an independent session in `provision_new_company_billing_bg`. [Source: story 8-1 Dev Notes; dependencies.py:51]
- **Event published AFTER commit.** `subscription.changed` event via `EventPublisher.publish()` must be called outside the `async with session_factory()` block, after a successful commit. This prevents publishing events for rolled-back transactions. Event failure is non-fatal (log WARNING, swallow). [Source: eusolicit_common.events.publisher; project architecture pattern]
- **SQLAlchemy async only.** All DB reads/writes use `AsyncSession`, `await session.execute(...)`, `await session.flush()`. No sync ORM. [Source: CLAUDE.md#Critical Conventions]
- **structlog everywhere.** `logger = structlog.get_logger()` at module level. Log every skip/success/failure with structured keys. [Source: project-context.md#Shared Packages rule 9]
- **Audit trail for subscription mutations.** Call `write_audit_entry()` after updating the subscription row with `action_type="billing.trial_provisioned"`. Uses caller's session (not dedicated session) — the `rbac.py` pattern is the ONLY exception. [Source: project-context.md rule 44; story 8-1 Dev Notes]
- **Schema isolation.** All subscription rows are in `client` schema. ORM model `Subscription` has `__table_args__ = {"schema": "client"}`. No direct SQL string queries — use ORM model. [Source: project-context.md#Database rule 1]
- **No new Alembic migration in Story 8.3.** All columns (`tier`, `status`, `trial_start`, `trial_end`, `is_trial`, `started_at`, `stripe_subscription_id`) were added by migration 027 in Story 8.2. Story 8.3 is pure service logic on top of the existing schema.
- **Pin Stripe SDK version.** `stripe>=8.0` is in `services/client-api/pyproject.toml`. Pin `stripe.api_version = "2024-06-20"` in `main.py` startup (already done by Story 8.1). Do NOT change this. [Source: test-design-epic-08.md#Risks to Plan — R-007]
- **NIT-4 from Story 8.2 MUST be resolved here.** The `tier_gate.py` `status == "active"` filter (AC7) was explicitly deferred to this story. If not fixed, trial Professional users will get HTTP 403 on all Professional-tier features immediately after registration — a broken UX. [Source: story 8-2 review NIT-4]

### Existing Code to Reuse (Do NOT Reinvent)

| Component | Location | Use |
|-----------|----------|-----|
| `provision_stripe_customer()` | `src/client_api/services/billing_service.py:35` | Story 8.1 function — call as Step 1 in combined BG task; do NOT duplicate |
| `get_session_factory()` | `src/client_api/dependencies.py:51` | Independent DB session for background tasks |
| `get_redis_client()` | `src/client_api/dependencies.py:89` | Singleton Redis client for EventPublisher |
| `EventPublisher` | `packages/eusolicit-common/src/eusolicit_common/events/publisher.py` | Publish `subscription.changed` to Redis Streams |
| `write_audit_entry()` | `src/client_api/services/audit_service.py` | Post-update audit log (caller's session) |
| `Subscription` ORM | `src/client_api/models/subscription.py` | Updated in 8.2 — has tier, status, trial_start, trial_end, is_trial, started_at, stripe_subscription_id columns |
| `SubscriptionStatus` enum | `eusolicit_models.enums.SubscriptionStatus` | Contains `trialing = "trialing"` — use as string `"trialing"` in DB writes (stored as String(50)) |
| `tier_gate.py` | `src/client_api/core/tier_gate.py` | UPDATE: add `_ALLOWED_STATUSES` and fix all three gate WHERE clauses |
| `opportunity_tier_gate.py` | `src/client_api/core/opportunity_tier_gate.py` | UPDATE: same status filter fix |
| `ClientApiSettings` | `src/client_api/config.py` | ADD `stripe_professional_price_id` field |

### Subscription ORM Model — Current State After Story 8.2

After migration 027, `client.subscriptions` has:
```
id                      UUID PK
company_id              UUID NOT NULL FK→client.companies.id (UNIQUE constraint added in 8.2)
stripe_customer_id      String(255) nullable  ← set by Story 8.1
stripe_subscription_id  String(255) nullable  ← Story 8.3 sets this
tier                    String(50) NOT NULL default "free"  ← Story 8.3 updates to "professional"
status                  String(50) nullable   ← Story 8.3 sets "trialing"
trial_start             DateTime(tz) nullable ← Story 8.3 sets now()
trial_end               DateTime(tz) nullable ← Story 8.3 sets now()+14d
is_trial                Boolean NOT NULL default false ← Story 8.3 sets True
started_at              DateTime(tz) nullable ← Story 8.3 sets now()
expires_at              DateTime(tz) nullable ← Story 8.5 will use this
plan                    String(50) nullable   ← DEPRECATED (keep, do not set)
current_period_start    DateTime(tz) nullable ← Story 8.4 webhook sync will populate
current_period_end      DateTime(tz) nullable ← Story 8.4 webhook sync will populate
```

**IMPORTANT**: When updating the subscription row in Story 8.3:
- The row WILL already exist (created by `provision_stripe_customer` in Story 8.1 Step 4)
- DO NOT INSERT a new row — find the existing row and UPDATE its fields via `existing.<field> = value`
- The `plan` deprecated column must NOT be touched in 8.3

### EventPublisher API

```python
from eusolicit_common.events.publisher import EventPublisher
import redis.asyncio

publisher = EventPublisher(redis_client)  # redis_client from get_redis_client()

msg_id = await publisher.publish(
    stream="subscription.changed",        # Redis Stream key
    event_type="SubscriptionChanged",     # Discriminator string
    payload={
        "company_id": str(company_id),
        "old_tier": "free",
        "new_tier": "professional",
        "timestamp": datetime.now(UTC).isoformat(),
    },
    source_service="client-api",
    tenant_id=str(company_id),           # Multi-tenant context
)
```

The envelope stored in Redis Streams contains: `event_id` (UUID4), `event_type`, `payload` (JSON), `timestamp` (ISO8601), `correlation_id` (auto-UUID4), `source_service`, `tenant_id`. [Source: eusolicit_common.events.publisher:publish()]

### Stripe Subscription Create — Correct API Call

```python
# asyncio.to_thread wraps the blocking Stripe SDK call
stripe_sub = await asyncio.to_thread(
    stripe.Subscription.create,
    customer=stripe_customer_id,         # "cus_..." from Story 8.1
    items=[{"price": settings.stripe_professional_price_id}],  # "price_..." from config
    trial_end=int((datetime.now(UTC) + timedelta(days=14)).timestamp()),  # Unix timestamp
    payment_behavior="default_incomplete",  # No payment method required for trial
    payment_settings={"save_default_payment_method": "on_subscription"},  # Prompt for PM on upgrade
    metadata={"company_id": str(company_id)},
    idempotency_key=f"trial-provisioning-{company_id}",
)
# Access: stripe_sub.id → "sub_..."
```

**`payment_behavior="default_incomplete"`** — critical for no-card trial: Stripe creates the subscription without requiring a payment method. If the customer has no saved payment method, the subscription goes to `trialing` state rather than failing. [Source: Stripe API docs — Subscription create]

**Stripe SDK error handling:**
```python
try:
    stripe_sub = await asyncio.to_thread(stripe.Subscription.create, **kwargs)
except stripe.error.StripeError as exc:
    logger.error("trial_provisioning_failed", company_id=str(company_id),
                 error=str(exc), error_type=type(exc).__name__)
    return False
```

### tier_gate.py Fix — Required Change (NIT-4 from Story 8.2)

Current (broken for trial users):
```python
.where(Subscription.status == "active")
```

Required (allows trialing Professional users):
```python
# _ALLOWED_STATUSES = ["active", "trialing"] at module level
.where(Subscription.status.in_(_ALLOWED_STATUSES))
```

This affects all three gate functions. Without this fix:
- Registration succeeds (HTTP 201 ✓)
- Trial subscription provisioned in background (Stripe ✓, DB updated ✓)
- User tries to use any Professional feature → HTTP 403 (broken!)
- User sees "paid subscription required" despite being on a trial

The fix must be in **both** `tier_gate.py` (for market intelligence, competitor intelligence, enterprise API gates) and `opportunity_tier_gate.py` (for opportunity-level feature gating, AI summaries, proposal drafts).

### Background Task Sequencing — Why Combined Function

Story 8.1 registered `provision_stripe_customer_bg` as the post-registration background task. Story 8.3 needs trial provisioning to run AFTER customer provisioning (needs `stripe_customer_id`).

**Option chosen: Combined `provision_new_company_billing_bg` function.** This:
- Runs both operations in a single DB session (one connection, one commit)
- Guarantees sequential execution (customer before trial)
- Single exception handler for the full billing setup lifecycle
- `provision_stripe_customer_bg` is kept (marked deprecated in docstring) for backward compat

**Why NOT two separate background tasks:** FastAPI `BackgroundTasks` run after response is sent but do NOT guarantee ordering between independently registered tasks. Two separate tasks could run concurrently, causing a race where trial provisioning starts before `stripe_customer_id` is set.

### Test Pattern for Background Task + Redis Mock

In integration tests, FastAPI's `httpx.AsyncClient` with ASGI transport runs BackgroundTasks within the test (synchronously before returning). Mock Stripe and Redis before the request:

```python
from unittest.mock import AsyncMock, MagicMock, patch

@pytest.mark.integration
async def test_register_triggers_trial_provisioning(client, session):
    mock_redis = AsyncMock()
    mock_redis.xadd = AsyncMock(return_value=b"1234-0")

    with (
        patch("stripe.Customer.create", return_value=MagicMock(id="cus_test")),
        patch("stripe.Subscription.create", return_value=MagicMock(id="sub_test")),
        patch("client_api.dependencies._redis_client", mock_redis),
    ):
        response = await client.post("/api/v1/auth/register", json={...})

    assert response.status_code == 201
    # Background task has already run (ASGI transport executes it synchronously)
    result = await session.execute(
        select(Subscription).where(Subscription.company_id == ...)
    )
    sub = result.scalar_one()
    assert sub.stripe_subscription_id == "sub_test"
    assert sub.tier == "professional"
    assert sub.status == "trialing"
    assert sub.is_trial is True
```

### Known Deferred Items from Story 8.2 (Resolve Here)

| Item | Where | Resolution |
|------|-------|------------|
| NIT-4: `status == "active"` filter in `tier_gate.py` | `core/tier_gate.py:101,152,203` | Fix in Task 5 (AC7) |
| NIT-7: `enterprise_api_keys.py:80` queries `Subscription.plan` | `api/v1/enterprise_api_keys.py:80` | **Out of scope for 8.3** — deferred to cleanup story (admin-api + client-api plan→tier cleanup). Fallback `or "enterprise"` keeps it functional. |
| NIT-5: admin-api `platform_analytics_service.py` queries `plan` | `admin-api` | **Out of scope for 8.3** — deferred to cleanup story. |

### Test Expectations from Epic Test Design

Story 8.3 directly implements the following test scenarios:

| Test ID | Level | Description | Priority |
|---------|-------|-------------|----------|
| **8.3-E2E-001** | E2E | Auto-provision 14-day Professional trial on new registration (no payment method; assert `status: trialing` in DB + Redis Stream event published) | **P0** |
| **8.3-API-001** | API | Second trial attempt rejected with clear error (AC3 idempotency check) | P2 |

**P0 risk mitigations enabled by this story:**
- **R-005 (Trial manipulation)**: AC3 enforces `stripe_subscription_id` presence check + DB unique constraint `uq_subscriptions_company_id` (from Story 8.2) → second trial creation is impossible.
- The smoke test criterion "Register company → triggers Trial creation" requires 8.1 ✅ + 8.2 ✅ + 8.3 (this story).

**Key test assertions from epic test design:**
- `status: trialing` in `client.subscriptions` row — direct AC2 output
- Redis Stream event published on `subscription.changed` — AC4 output
- Attempt to create second trial → skipped (no new Stripe subscription) — AC3 output

[Source: eusolicit-docs/test-artifacts/test-design-epic-08.md#P0 Tests; #P2 Tests]

### Project Structure Notes

- `billing_service.py` gains `provision_professional_trial()` and `provision_new_company_billing_bg()` — add after the existing functions, not before. Keep `provision_stripe_customer_bg()` at its current location (deprecated).
- No new files needed for service logic — all goes into existing `billing_service.py`.
- New test files: `tests/unit/test_trial_provisioning.py`, `tests/unit/test_tier_gate_trial.py`, `tests/unit/test_new_company_billing_bg.py`, `tests/integration/test_trial_provisioning_flow.py`.
- The `opportunity_tier_gate.py` fix (status filter) requires a small update but no test file changes — existing `test_opportunity_tier_gate.py` tests should be updated to mock `status="trialing"` as a passing case.

### References

- Epic story spec: [Source: eusolicit-docs/planning-artifacts/epic-08-subscription-billing.md#S08.03]
- Epic test design (P0 test 8.3-E2E-001, P2 test 8.3-API-001, risk R-005): [Source: eusolicit-docs/test-artifacts/test-design-epic-08.md]
- Story 8.1 (billing_service.py, BackgroundTasks pattern, asyncio.to_thread idiom): [Source: eusolicit-docs/implementation-artifacts/8-1-stripe-customer-provisioning-on-company-registration.md]
- Story 8.2 (schema, ORM model current state, NIT-4 deferral): [Source: eusolicit-docs/implementation-artifacts/8-2-subscription-tier-database-schema.md]
- billing_service.py current state: [Source: eusolicit-app/services/client-api/src/client_api/services/billing_service.py]
- tier_gate.py current state: [Source: eusolicit-app/services/client-api/src/client_api/core/tier_gate.py]
- opportunity_tier_gate.py: [Source: eusolicit-app/services/client-api/src/client_api/core/opportunity_tier_gate.py]
- EventPublisher: [Source: eusolicit-app/packages/eusolicit-common/src/eusolicit_common/events/publisher.py]
- auth.py register endpoint: [Source: eusolicit-app/services/client-api/src/client_api/api/v1/auth.py:32]
- dependencies.py (get_session_factory, get_redis_client): [Source: eusolicit-app/services/client-api/src/client_api/dependencies.py]
- ClientApiSettings: [Source: eusolicit-app/services/client-api/src/client_api/config.py]
- Project context rules: [Source: eusolicit-docs/project-context.md]

## Dev Agent Record

### Agent Model Used

Claude Sonnet 4.6

### Debug Log References

- Integration test FK violation: `client.subscriptions.company_id_fkey` required pre-seeding a committed company row before calling `provision_new_company_billing_bg`. Fixed by adding `INSERT INTO client.companies (id, name) ... ON CONFLICT DO NOTHING` + commit in `_run_provision_new_company_billing_bg` helper and the idempotency test.
- Migration 027 FK on `client.opportunities`: `client.opportunities` table does not exist in `eusolicit_test` DB. Applied migrations 026 and 027 SQL manually, skipping `add_on_purchases` table (FK to `client.opportunities`). Alembic version stamped to `027`.
- `alembic_version.version_num VARCHAR(32)` overflow: expanded to `VARCHAR(128)` before stamping `025_audit_log_company_correlation`.
- `opportunity_tier_gate.py` parameter name `db` renamed to `session` to match ATDD test expectation (3 existing tests in `test_opportunity_tier_gate.py` updated accordingly).

### Completion Notes List

- All 7 tasks implemented (Tasks 1–5 = production code; Tasks 6–7 = pre-written ATDD tests made GREEN).
- AC7 (NIT-4 from Story 8.2): `tier_gate.py` and `opportunity_tier_gate.py` both updated from `status == "active"` to `status.in_(["active", "trialing"])` with `_ALLOWED_STATUSES` module-level constant.
- `provision_stripe_customer_bg()` retained but marked DEPRECATED in docstring; backward-compat preserved.
- Integration test helper `_run_provision_new_company_billing_bg` fixed: added committed company seeding (missing from pre-written test as noted in its own comment).
- 474 unit tests pass; 5/5 integration tests pass; ruff clean on all modified files.
- No new Alembic migration created for Story 8.3 (all schema columns added by migration 027 in Story 8.2).

### File List

- `services/client-api/src/client_api/config.py` — added `stripe_professional_price_id: str | None = None`
- `services/client-api/src/client_api/services/billing_service.py` — added `provision_professional_trial()`, `provision_new_company_billing_bg()`; updated module docstring; marked `provision_stripe_customer_bg()` DEPRECATED
- `services/client-api/src/client_api/api/v1/auth.py` — updated register endpoint to use `provision_new_company_billing_bg`
- `services/client-api/src/client_api/core/tier_gate.py` — added `_ALLOWED_STATUSES = ["active", "trialing"]`; all three gate functions updated from `== "active"` to `.in_(_ALLOWED_STATUSES)`
- `services/client-api/src/client_api/core/opportunity_tier_gate.py` — same status filter fix; parameter `db` renamed to `session`; added `_ALLOWED_STATUSES`
- `services/client-api/tests/unit/test_opportunity_tier_gate.py` — updated 3 call sites from `db=mock_db` to `session=mock_db`
- `services/client-api/tests/integration/test_trial_provisioning_flow.py` — added committed company seeding to `_run_provision_new_company_billing_bg` helper and idempotency test

## Change Log

| Date | Version | Description | Author |
|------|---------|-------------|--------|
| 2026-04-18 | 1.0 | Story 8.3 implementation complete — 14-day Professional trial provisioning, tier gate trialing status fix, 479 tests GREEN | Claude Sonnet 4.6 |

## Senior Developer Review

**Outcome:** ✅ **Approve**

**Reviewer:** bmad-code-review (Claude Sonnet)
**Date:** 2026-04-18

**AC coverage:** All 8 ACs satisfied with evidence in code + tests.
- AC1 (Stripe call shape, idempotency key format) — verified in `billing_service.provision_professional_trial` and unit tests.
- AC2 (atomic row UPDATE, `stripe_customer_id` preserved) — verified; test `test_stripe_customer_id_preserved_after_trial_provisioning` pins this.
- AC3 (one-trial-per-company guard) — explicit check + DB unique constraint from Story 8.2 as belt-and-braces.
- AC4 (event AFTER commit) — `EventPublisher.publish()` correctly placed outside `async with session_factory()`; publish failures caught and logged WARNING.
- AC5 (graceful degradation) — three skip branches wired and tested.
- AC6 (combined BG task + auth.py rewiring) — `provision_stripe_customer_bg` retained with DEPRECATED docstring; auth.py import updated.
- AC7 (NIT-4 resolved) — `_ALLOWED_STATUSES` added to both `tier_gate.py` and `opportunity_tier_gate.py`; all three gate functions + opportunity gate use `.in_()`.
- AC8 — unit + integration coverage for each AC and for idempotency end-to-end.

**Project convention compliance:**
- ✅ SQLAlchemy async only, `async_sessionmaker` via `get_session_factory()`
- ✅ `asyncio.to_thread()` for sync Stripe SDK (per Story 8.1 idiom)
- ✅ structlog with structured keys for every skip/success/failure
- ✅ Audit trail via `write_audit_entry` on caller's session (rule 44)
- ✅ BackgroundTasks only — no Celery
- ✅ Pydantic v2 via `eusolicit_models` (no inline DTOs introduced)
- ✅ No new migration (schema already in place from migration 027)

**Minor observations (non-blocking, not required for approval):**
1. `logger.error("trial_provisioning_failed_no_subscription_row", ..., msg="...")` uses `msg=` kwarg — `reason=` would be more idiomatic for structlog; purely cosmetic.
2. `write_audit_entry` calls `session.flush()` internally, so the prior explicit `await session.flush()` in `provision_professional_trial` is effectively flushed twice. Harmless; keep for clarity.
3. Allowing `status='trialing'` in `require_enterprise_tier` is spec-compliant forward-compat (AC7), even though no Enterprise trials are provisioned today. Acceptable given the explicit AC.
4. Integration-level wiring test `test_register_endpoint_background_task_is_provision_new_company_billing_bg` is an import-level assertion; the epic's P0 E2E test (8.3-E2E-001) lives at the Playwright level and is tracked by the epic test design, not this story.

**No deviations detected.** No scope creep, no missing requirements, no architectural drift, no acceptance gaps.

**Recommendation:** Mark story `done` and proceed to Story 8.4.
