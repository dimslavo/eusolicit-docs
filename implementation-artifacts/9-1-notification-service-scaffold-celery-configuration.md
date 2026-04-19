# Story 9.1: Notification Service Scaffold & Celery Configuration

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer on the EU Solicit notification service**,
I want **the existing Notification Service Celery app extended with full S09 queue routing, retry policies, dead-letter handling, S09-specific Beat schedule entries, stub tasks for all S09 consumers, and worker/beat Docker Compose services**,
so that **subsequent S09 stories (S09.04–S09.11) can add `@shared_task` implementations and have Beat pick them up automatically, and the notification worker and Beat containers start cleanly in Docker Compose from day one**.

## Acceptance Criteria

1. Celery worker starts and connects to Redis broker — confirmed by `celery -A notification.workers.celery_app inspect ping` returning a response
2. Celery Beat scheduler is configured and triggers a test periodic task (`notification.health`) on a heartbeat schedule (every 60 seconds)
3. Task routing delivers tasks to correct dedicated queues: `alerts` (alert matching/dispatch), `emails` (SendGrid delivery), `calendar-sync` (Google/Microsoft sync), `usage-reporting` (Stripe usage), `notification-default` (health check)
4. Retry policy applies exponential backoff with max 5 retries on task failure — `@shared_task(max_retries=5, default_retry_delay=60)` set on all stub tasks with `autoretry_for` wired for S09 domain exceptions
5. Dead-letter handling: a `task_failure` signal handler publishes failed task metadata to Redis list `notification:dead_letter` (name, args, traceback, timestamp) for inspection — tasks exhausting retries are logged at ERROR level via structlog
6. Alembic migration 001_initial.py already creates the `notification._migrations_meta` table (existing — verify it runs cleanly); no new tables in this story (S09.02 adds alert_log/email_log/sync_log)
7. Docker Compose adds `notification-worker` and `notification-beat` services (worker runs `alerts,emails,calendar-sync,usage-reporting,notification-default` queues; beat is singleton)

## Tasks / Subtasks

- [ ] Task 1: Extend `workers/celery_app.py` with S09 configuration (AC: 1, 3, 4, 5)
  - [ ] 1.1 Add S09 stub task includes to the `include=[]` list in the existing `Celery("notification", ...)` instantiation — `notification.workers.tasks.health`, `notification.workers.tasks.alert_matching`, `notification.workers.tasks.send_email`, `notification.workers.tasks.digest`, `notification.workers.tasks.calendar_sync`
  - [ ] 1.2 Add `celery.conf.update(...)` block (currently missing) with: `task_routes` dict for S09 queues, `worker_prefetch_multiplier=1`, `task_acks_late=True`, `task_max_retries=5`, `task_default_retry_delay=60`, `task_retry_backoff=True`, `task_retry_backoff_max=600`
  - [ ] 1.3 Register `task_failure` signal handler that publishes dead-letter metadata to Redis list `notification:dead_letter` with 7-day TTL; use `redis.from_url(CELERY_BROKER_URL)` — fire-and-forget, never raises

- [ ] Task 2: Extend `workers/beat_schedule.py` with S09 periodic tasks (AC: 2)
  - [ ] 2.1 Add S09 Beat schedule entries to the existing `BEAT_SCHEDULE` dict: `alert-digest-daily` (07:00 UTC via crontab), `alert-digest-weekly` (Monday 07:00 UTC via crontab with day_of_week=1), `calendar-sync-periodic` (every 15 min via timedelta), `notification-heartbeat` (every 60 seconds via timedelta for AC 2)
  - [ ] 2.2 All S09 cron values read from `os.environ` with defaults — `NOTIFICATION_DAILY_DIGEST_HOUR`, `NOTIFICATION_WEEKLY_DIGEST_HOUR`, `NOTIFICATION_CALENDAR_SYNC_INTERVAL_MINUTES`
  - [ ] 2.3 Note: `sync-usage-to-stripe-daily` already exists from S08.08 at 03:00 UTC — do NOT duplicate; S09.11 will move it to the `usage-reporting` queue

- [ ] Task 3: Create S09 stub task modules (AC: 3, 4)
  - [ ] 3.1 Create `src/notification/workers/tasks/health.py` — `@shared_task(name="notification.health")` returning `"OK"`; real implementation (liveness probe)
  - [ ] 3.2 Create `src/notification/workers/tasks/alert_matching.py` — `@shared_task(name="notification.match_alerts", bind=True, max_retries=5, default_retry_delay=60, queue="alerts")` stub; docstring: "Implemented in S09.04 — processes opportunities.ingested Redis Stream events"
  - [ ] 3.3 Create `src/notification/workers/tasks/send_email.py` — `@shared_task(name="notification.send_email", bind=True, max_retries=5, default_retry_delay=60, queue="emails")` stub; docstring: "Implemented in S09.06 — dispatches email via SendGrid v3 API"
  - [ ] 3.4 Create `src/notification/workers/tasks/digest.py` — two stubs: `notification.send_daily_digest` and `notification.send_weekly_digest` (both `bind=True, max_retries=5, queue="alerts"`); docstrings: "Implemented in S09.05"
  - [ ] 3.5 Create `src/notification/workers/tasks/calendar_sync.py` — `@shared_task(name="notification.sync_calendars", bind=True, max_retries=5, default_retry_delay=60, queue="calendar-sync")` stub; docstring: "Implemented in S09.08 and S09.09 — syncs Google/Microsoft calendar events"
  - [ ] 3.6 **DO NOT** stub `billing_usage_sync.sync_usage_to_stripe` — it's real and in `usage-reporting` queue routing; just add queue route for it in Task 1.2

- [ ] Task 4: Update `pyproject.toml` (AC: 7)
  - [ ] 4.1 Add `notification-beat = "notification.workers.celery_app:celery"` to `[project.scripts]` (current only has `notification-worker`)
  - [ ] 4.2 Add to `[project.optional-dependencies] dev`: `"pytest-timeout>=2.3"`, `"freezegun>=1.5"` (fakeredis>=2.21 already present)
  - [ ] 4.3 Add `"redis>=5.0"` to `dependencies` if not already present (required for dead-letter signal handler's `redis.from_url()`)

- [ ] Task 5: Add worker and beat services to Docker Compose (AC: 7)
  - [ ] 5.1 In `eusolicit-app/docker-compose.yml`, add `notification-worker` service after the `notification` service block (after line ~304):
    - Build: same context/dockerfile as `notification`
    - Command: `celery -A notification.workers.celery_app worker --loglevel=info -Q alerts,emails,calendar-sync,usage-reporting,notification-default -c 4`
    - Volumes/env_file/depends_on: same as `notification`; add `CELERY_BROKER_URL` and `CELERY_RESULT_BACKEND` env vars
    - No ports; healthcheck: `celery -A notification.workers.celery_app inspect ping -d celery@$$HOSTNAME --timeout 5`
  - [ ] 5.2 Add `notification-beat` service (singleton): command `celery -A notification.workers.celery_app beat --loglevel=info --scheduler celery.beat.PersistentScheduler --schedule /tmp/celerybeat-schedule`; `restart: on-failure`; no healthcheck

- [ ] Task 6: Write unit tests (AC: 1, 2, 3, 4, 5)
  - [ ] 6.1 Create `tests/unit/test_celery_config.py`:
    - Assert `app.conf.beat_schedule.keys()` contains S09 entries: `alert-digest-daily`, `alert-digest-weekly`, `calendar-sync-periodic`, `notification-heartbeat`
    - Assert `app.conf.timezone == "UTC"` and `app.conf.enable_utc is True`
    - Assert `app.conf.worker_prefetch_multiplier == 1`
    - Assert `app.conf.task_acks_late is True`
    - Assert `app.conf.task_routes["notification.health"] == {"queue": "notification-default"}`
    - Assert `app.conf.task_routes["notification.match_alerts"] == {"queue": "alerts"}`
    - Assert `app.conf.task_routes["notification.send_email"] == {"queue": "emails"}`
    - Assert `app.conf.task_routes["notification.sync_calendars"] == {"queue": "calendar-sync"}`
    - `monkeypatch.setenv("NOTIFICATION_CALENDAR_SYNC_INTERVAL_MINUTES", "5")` + `importlib.reload(beat_schedule)` → assert calendar-sync schedule == `timedelta(minutes=5)`
    - `result = health_check.apply(); assert result.get() == "OK"`
  - [ ] 6.2 All tests must pass without a Redis broker (`task_always_eager=True` already in `tests/conftest.py`)

## Dev Notes

### CRITICAL: Service Already Exists — Extend, Do Not Recreate

The notification service was partially scaffolded during Epics 8 and 12. **Do NOT create a new service directory or new `celery_app.py` from scratch.** Instead, modify existing files:

| File | Action |
|---|---|
| `src/notification/workers/celery_app.py` | MODIFY — add task includes and `celery.conf.update()` block |
| `src/notification/workers/beat_schedule.py` | MODIFY — add S09 Beat entries to existing BEAT_SCHEDULE dict |
| `src/notification/workers/tasks/health.py` | NEW |
| `src/notification/workers/tasks/alert_matching.py` | NEW — stub |
| `src/notification/workers/tasks/send_email.py` | NEW — stub |
| `src/notification/workers/tasks/digest.py` | NEW — stub |
| `src/notification/workers/tasks/calendar_sync.py` | NEW — stub |
| `pyproject.toml` | MODIFY — add notification-beat entry point + dev deps |
| `docker-compose.yml` | MODIFY — add notification-worker and notification-beat services |
| `tests/unit/test_celery_config.py` | NEW |

**DO NOT TOUCH:**
- `src/notification/workers/tasks/billing_usage_sync.py` — real implementation from S08.08
- `src/notification/workers/tasks/refresh_analytics_views.py` — real implementation from S12.01
- `src/notification/tasks/report_generation.py` — real implementation from S12.09
- `src/notification/tasks/scheduled_report_delivery.py` — real implementation from S12.10
- `src/notification/alembic/` — migration chain is intact; S09.02 adds the next migration
- `src/notification/config.py` — SendGrid/Stripe/S3 settings already configured

[Source: eusolicit-app/services/notification/src/notification/workers/celery_app.py]
[Source: eusolicit-app/services/notification/src/notification/workers/beat_schedule.py]

### Current `celery_app.py` — Gaps to Fill

The existing file (as of 2026-04-19) is:

```python
celery = Celery(
    "notification",
    broker=os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/0"),
    backend=os.environ.get("CELERY_RESULT_BACKEND", "redis://localhost:6379/1"),
    include=[
        "notification.workers.tasks.refresh_analytics_views",
        "notification.tasks.report_generation",
        "notification.tasks.scheduled_report_delivery",
        "notification.workers.tasks.billing_usage_sync",
    ],
)

celery.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
    beat_schedule=BEAT_SCHEDULE,
)
```

**What is MISSING** (add these):
- `include` for S09 stub tasks (health, alert_matching, send_email, digest, calendar_sync)
- `task_routes` dict assigning tasks to queues
- `worker_prefetch_multiplier=1` — prevents starvation on long-running tasks
- `task_acks_late=True` — prevents message loss on worker crash (pairs with manual ACK)
- `task_max_retries=5` — global default (individual tasks set their own via decorator)
- `task_retry_backoff=True` — exponential backoff enabled
- `task_retry_backoff_max=600` — cap at 10 minutes between retries

### Target `celery_app.py` — Full Updated Version

```python
"""Celery application instance for the Notification Service.

Configuration:
- Broker: Redis (CELERY_BROKER_URL env var, default redis://localhost:6379/0)
- Result backend: Redis (CELERY_RESULT_BACKEND env var, default redis://redis:6379/1)
- Serializer: JSON
- Timezone: UTC

Queues (S09.01):
  alerts           — alert matching, immediate dispatch, digest assembly
  emails           — SendGrid email delivery (S09.06)
  calendar-sync    — Google/Microsoft calendar periodic sync (S09.08/09)
  usage-reporting  — Stripe usage metering sync (S08.08/S09.11)
  notification-default — health check, misc

Usage:
    celery -A notification.workers.celery_app worker --loglevel=info \\
           -Q alerts,emails,calendar-sync,usage-reporting,notification-default
    celery -A notification.workers.celery_app beat --loglevel=info

[Source: eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md#S09.01]
"""
from __future__ import annotations

import os

import redis as redis_client
import structlog
from celery import Celery
from celery.signals import task_failure

from notification.workers.beat_schedule import BEAT_SCHEDULE

logger = structlog.get_logger(__name__)

_BROKER_URL = os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/0")

celery = Celery(
    "notification",
    broker=_BROKER_URL,
    backend=os.environ.get("CELERY_RESULT_BACKEND", "redis://redis:6379/1"),
    include=[
        # Epic 12 — analytics view refreshes (existing)
        "notification.workers.tasks.refresh_analytics_views",
        # Epic 12 — report generation (existing)
        "notification.tasks.report_generation",
        "notification.tasks.scheduled_report_delivery",
        # Epic 8 — Stripe usage billing sync (existing)
        "notification.workers.tasks.billing_usage_sync",
        # S09.01 — new health + S09 domain stubs
        "notification.workers.tasks.health",
        "notification.workers.tasks.alert_matching",
        "notification.workers.tasks.send_email",
        "notification.workers.tasks.digest",
        "notification.workers.tasks.calendar_sync",
    ],
)

celery.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",          # REQUIRED — prevents DST shift bugs
    enable_utc=True,
    beat_schedule=BEAT_SCHEDULE,
    # S09.01: queue routing — each S09 domain gets an isolated queue
    task_routes={
        "notification.health": {"queue": "notification-default"},
        "notification.match_alerts": {"queue": "alerts"},
        "notification.send_daily_digest": {"queue": "alerts"},
        "notification.send_weekly_digest": {"queue": "alerts"},
        "notification.send_email": {"queue": "emails"},
        "notification.sync_calendars": {"queue": "calendar-sync"},
        # S08.08 billing sync moves to dedicated queue
        "notification.workers.tasks.billing_usage_sync.sync_usage_to_stripe": {
            "queue": "usage-reporting"
        },
        # Analytics refreshes stay on notification-default (low priority, fast)
        "notification.workers.tasks.refresh_analytics_views.refresh_market_intelligence": {
            "queue": "notification-default"
        },
        "notification.workers.tasks.refresh_analytics_views.refresh_roi_tracker": {
            "queue": "notification-default"
        },
        "notification.workers.tasks.refresh_analytics_views.refresh_team_performance": {
            "queue": "notification-default"
        },
        "notification.workers.tasks.refresh_analytics_views.refresh_competitor_intelligence": {
            "queue": "notification-default"
        },
        "notification.workers.tasks.refresh_analytics_views.refresh_usage_consumption": {
            "queue": "notification-default"
        },
        "notification.tasks.scheduled_report_delivery.check_scheduled_reports": {
            "queue": "notification-default"
        },
    },
    worker_prefetch_multiplier=1,   # fair dispatch; prevents long tasks starving health check
    task_acks_late=True,            # ack only after completion — prevents silent loss on crash
    task_max_retries=5,             # global default; tasks can override with their own decorator
    task_retry_backoff=True,        # exponential backoff between retries
    task_retry_backoff_max=600,     # cap at 10 minutes
)


# ---------------------------------------------------------------------------
# Dead-letter handler — stores failed tasks in Redis for inspection
# ---------------------------------------------------------------------------

@task_failure.connect
def on_task_failure(sender, task_id, exception, args, kwargs, traceback, einfo, **extra):
    """Publish failed task metadata to Redis dead-letter list.

    Dead-letter entries expire after 7 days. Never raises — failure here
    must not mask the original exception or block Celery's error handling.
    """
    try:
        import json
        from datetime import UTC, datetime

        r = redis_client.from_url(_BROKER_URL, socket_connect_timeout=2)
        entry = json.dumps({
            "task_name": sender.name,
            "task_id": task_id,
            "args": str(args),
            "kwargs": str(kwargs),
            "exception": str(exception),
            "timestamp": datetime.now(UTC).isoformat(),
        })
        r.lpush("notification:dead_letter", entry)
        r.expire("notification:dead_letter", 60 * 60 * 24 * 7)  # 7 days
    except Exception as dlq_exc:
        logger.error("dead_letter_handler_failed", exc=str(dlq_exc))
```

### Target `beat_schedule.py` — S09 Additions

Add these entries to the existing `BEAT_SCHEDULE` dict (keep all existing entries):

```python
# --- S09.01: env-var overridable notification schedule ---
_DAILY_DIGEST_HOUR = os.environ.get("NOTIFICATION_DAILY_DIGEST_HOUR", "7")
_WEEKLY_DIGEST_HOUR = os.environ.get("NOTIFICATION_WEEKLY_DIGEST_HOUR", "7")
_CALENDAR_SYNC_MINUTES = int(os.environ.get("NOTIFICATION_CALENDAR_SYNC_INTERVAL_MINUTES", "15"))

# ... inside BEAT_SCHEDULE dict, ADD:

    "alert-digest-daily": {
        "task": "notification.send_daily_digest",
        "schedule": crontab(hour=_DAILY_DIGEST_HOUR, minute="0"),  # 07:00 UTC
    },
    "alert-digest-weekly": {
        "task": "notification.send_weekly_digest",
        "schedule": crontab(hour=_WEEKLY_DIGEST_HOUR, minute="0", day_of_week=1),  # Mon 07:00 UTC
    },
    "calendar-sync-periodic": {
        "task": "notification.sync_calendars",
        "schedule": timedelta(minutes=_CALENDAR_SYNC_MINUTES),  # every 15 min rolling
    },
    "notification-heartbeat": {
        "task": "notification.health",
        "schedule": timedelta(seconds=60),   # liveness probe — confirms Beat→broker→worker path
    },
```

**IMPORTANT**: Import `from datetime import timedelta` at the top of `beat_schedule.py` — it uses only `crontab` currently.

**Do NOT add** a Stripe usage entry — it already exists as `sync-usage-to-stripe-daily` at 03:00 UTC from S08.08.

### Stub Task Pattern

All stubs must be importable and never raise `NotImplementedError` (Beat fires these on schedule — a NotImplementedError crashes the dispatcher):

```python
# src/notification/workers/tasks/alert_matching.py
"""Alert matching task stub. Real implementation in S09.04."""
from __future__ import annotations

import structlog
from celery import shared_task

logger = structlog.get_logger(__name__)


@shared_task(
    name="notification.match_alerts",
    bind=True,
    max_retries=5,
    default_retry_delay=60,
)
def match_alerts(self, event_data: dict | None = None) -> None:  # noqa: ARG001
    """Process opportunities.ingested events and dispatch immediate alerts.

    Real implementation in S09.04:
    - Reads Redis Stream eu-solicit:opportunities consumer group
    - Matches against client.alert_preferences (schedule='immediate')
    - Dispatches send_email task for each matching user
    """
    logger.info("match_alerts: stub — not yet implemented (S09.04)")
```

Use the same pattern for `send_email`, `calendar_sync`, `digest` stubs.

### Health Task — Real Implementation

```python
# src/notification/workers/tasks/health.py
"""Notification Service Celery health-check task."""
from __future__ import annotations
from celery import shared_task


@shared_task(name="notification.health")
def health_check() -> str:
    """Return 'OK'. Used as liveness probe and Beat heartbeat confirmation."""
    return "OK"
```

### Docker Compose — Notification Worker and Beat Services

Add after the `notification` service block (currently ends ~line 304 of docker-compose.yml):

```yaml
  notification-worker:
    build:
      context: .
      dockerfile: services/notification/Dockerfile
    command: >
      celery -A notification.workers.celery_app worker
      --loglevel=info
      -Q alerts,emails,calendar-sync,usage-reporting,notification-default
      -c 4
    volumes:
      - ./services/notification/src/notification:/app/src/notification
      - ./packages/eusolicit-common/src/eusolicit_common:/app/packages/eusolicit-common/src/eusolicit_common
      - ./packages/eusolicit-models/src/eusolicit_models:/app/packages/eusolicit-models/src/eusolicit_models
      - ./packages/eusolicit-kraftdata/src/eusolicit_kraftdata:/app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata
    env_file: .env
    environment:
      DATABASE_URL: ${NOTIFICATION_DB_URL:-postgresql+asyncpg://notification_role:notification_password@postgres:5432/eusolicit}
      REDIS_URL: ${REDIS_URL:-redis://redis:6379/0}
      CELERY_BROKER_URL: ${CELERY_BROKER_URL:-redis://redis:6379/0}
      CELERY_RESULT_BACKEND: ${CELERY_RESULT_BACKEND:-redis://redis:6379/1}
      PYTHONPATH: /app/src
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "celery -A notification.workers.celery_app inspect ping -d celery@$$HOSTNAME --timeout 5"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  notification-beat:
    build:
      context: .
      dockerfile: services/notification/Dockerfile
    command: >
      celery -A notification.workers.celery_app beat
      --loglevel=info
      --scheduler celery.beat.PersistentScheduler
      --schedule /tmp/celerybeat-schedule
    volumes:
      - ./services/notification/src/notification:/app/src/notification
      - ./packages/eusolicit-common/src/eusolicit_common:/app/packages/eusolicit-common/src/eusolicit_common
      - ./packages/eusolicit-models/src/eusolicit_models:/app/packages/eusolicit-models/src/eusolicit_models
      - ./packages/eusolicit-kraftdata/src/eusolicit_kraftdata:/app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata
    env_file: .env
    environment:
      DATABASE_URL: ${NOTIFICATION_DB_URL:-postgresql+asyncpg://notification_role:notification_password@postgres:5432/eusolicit}
      REDIS_URL: ${REDIS_URL:-redis://redis:6379/0}
      CELERY_BROKER_URL: ${CELERY_BROKER_URL:-redis://redis:6379/0}
      CELERY_RESULT_BACKEND: ${CELERY_RESULT_BACKEND:-redis://redis:6379/1}
      PYTHONPATH: /app/src
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: on-failure    # Beat singleton must restart on crash
```

**CRITICAL — Beat Singleton**: Only ONE `notification-beat` process must run. In Docker Compose this is automatic. In Kubernetes, use `Deployment` with `replicas: 1` and `strategy: Recreate`.

### Alembic Migration — Existing, No New Migration Needed

The `alembic/versions/001_initial.py` already creates `notification._migrations_meta`. The `notification` schema is provisioned during DB role setup (Epic 1 / S01.03). The `alembic/env.py` sets `search_path` to the `notification` schema inside migrations.

**S09.01 Alembic AC** is satisfied by verifying `make migrate-service SVC=notification` runs cleanly. S09.02 will add the next migration (002) for `alert_log`, `email_log`, `sync_log`.

**Verify with**: `cd eusolicit-app && make migrate-service SVC=notification` — should show "Running upgrade 001 → (nothing)" if already applied, no errors.

### pyproject.toml — Changes

Current `[project.scripts]` only has `notification-worker`. Add:
```toml
[project.scripts]
notification-worker = "notification.workers.celery_app:celery"
notification-beat = "notification.workers.celery_app:celery"    # ADD THIS
```

Add to `[project.optional-dependencies] dev`:
```toml
dev = [
    ...existing entries...
    "pytest-timeout>=2.3",
    "freezegun>=1.5",
]
```

Add to `[project] dependencies` if `redis` package is not already there:
```toml
"redis>=5.0",   # for dead-letter signal handler (redis.from_url)
```

Note: `celery[redis]>=5.3` already installs `redis` as a dependency, so this addition may be redundant — verify before adding.

### Dead-Letter Implementation Notes

**Celery DLQ pattern used here**: `task_failure` signal → publish to Redis list. This is different from RabbitMQ DLX or AWS SQS DLQ. The `notification:dead_letter` Redis list acts as a simple inspection queue.

**Key decisions**:
- Use `LPUSH` (prepend) so newest failures are at the front
- 7-day expiry via `EXPIRE` — prevents unbounded growth
- `socket_connect_timeout=2` on the Redis client — prevents the DLQ handler from blocking if Redis is unreachable during a cascade failure
- `try/except Exception: logger.error(...)` wraps the entire handler — DLQ failure must never propagate

**Inspection commands** (for ops):
```bash
# View latest dead-letter entries
redis-cli LRANGE notification:dead_letter 0 9

# Count
redis-cli LLEN notification:dead_letter
```

### Test Patterns — `test_celery_config.py`

Tests use `task_always_eager=True` (configured in `tests/conftest.py::celery_config`). The env-var override test requires `importlib.reload`:

```python
import importlib
import os
from datetime import timedelta

def test_calendar_sync_interval_override(monkeypatch):
    monkeypatch.setenv("NOTIFICATION_CALENDAR_SYNC_INTERVAL_MINUTES", "5")
    import notification.workers.beat_schedule as bs
    importlib.reload(bs)
    assert bs.BEAT_SCHEDULE["calendar-sync-periodic"]["schedule"] == timedelta(minutes=5)
```

**CRITICAL**: `beat_schedule.py` reads `os.environ` at **module import time**. Always use `importlib.reload()` after `monkeypatch.setenv()` — not just patching the variable.

Test for health task:
```python
from notification.workers.tasks.health import health_check

def test_health_task_returns_ok():
    result = health_check.apply()
    assert result.get() == "OK"
```

### Cross-Story Dependencies

| Story | Status | Relevance |
|---|---|---|
| S01.05 — Redis Streams Event Bus | done | Redis at `redis://redis:6379/0` confirmed working |
| S08.08 — Usage Metering/Stripe Sync | done | `billing_usage_sync.sync_usage_to_stripe` task is real; just add queue routing |
| S12.01 — Analytics MV Refresh | done | `refresh_analytics_views` tasks are real; add queue routing to `notification-default` |
| S09.02 — Notification Schema Migrations | backlog | Next story — adds alert_log/email_log/sync_log tables |
| S09.04 — Alert Matching & Immediate Dispatch | backlog | Replaces `alert_matching.py` stub |
| S09.05 — Daily/Weekly Digest Assembly | backlog | Replaces `digest.py` stubs |
| S09.06 — SendGrid Email Delivery | backlog | Replaces `send_email.py` stub |
| S09.08/09 — Calendar OAuth2 & Sync | backlog | Replaces `calendar_sync.py` stub |

### Risk Mitigations (from Test Design)

| Risk | Mitigation in S09.01 |
|---|---|
| **R-001** Redis stream event loss for immediate notifications (Score: 6) | `task_acks_late=True` + consumer group pattern (consumer in S09.04 will use manual XACK) |
| **R-004** OAuth token exposure | Config settings via `NotificationSettings` (already in config.py); Fernet encryption key setup deferred to S09.08 |

### Project Structure After This Story

```
eusolicit-app/
├── docker-compose.yml                                   # MODIFY — add notification-worker + notification-beat
└── services/notification/
    ├── pyproject.toml                                   # MODIFY — add notification-beat entry point, dev deps
    └── src/notification/
        └── workers/
            ├── celery_app.py                            # MODIFY — add includes, task_routes, retry config, DLQ handler
            ├── beat_schedule.py                         # MODIFY — add S09 Beat entries
            └── tasks/
                ├── health.py                            # + NEW — real implementation
                ├── alert_matching.py                    # + NEW — stub (S09.04 implements)
                ├── send_email.py                        # + NEW — stub (S09.06 implements)
                ├── digest.py                            # + NEW — 2 stubs (S09.05 implements)
                └── calendar_sync.py                     # + NEW — stub (S09.08/09 implements)
    └── tests/
        └── unit/
            └── test_celery_config.py                    # + NEW
```

### Test Expectations from Epic-Level Test Design

| Priority | Test ID | Description | Implementation |
|---|---|---|---|
| **P2** | E09-CELERY-001 | Celery Retry Backoff | Mock failing task, assert retry count and delay follows exponential backoff |
| **P0** | E09-INFRA-001 | Beat schedule has all S09 periodic tasks | Assert `app.conf.beat_schedule.keys()` contains S09 entries |
| **P0** | E09-INFRA-002 | Queue routing correct | Assert `app.conf.task_routes` maps each task to expected queue |
| **P1** | E09-INFRA-003 | Health task returns OK | `health_check.apply().get() == "OK"` |
| **P2** | E09-INFRA-004 | Env-var override works | `importlib.reload()` pattern for calendar sync interval |

[Source: eusolicit-docs/test-artifacts/test-design-epic-09.md#P2-Celery-Retry-Backoff]

### References

- [Source: eusolicit-docs/planning-artifacts/epic-09-notifications-alerts-calendar.md#S09.01]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-09.md]
- [Source: eusolicit-app/services/notification/src/notification/workers/celery_app.py] — current state to extend
- [Source: eusolicit-app/services/notification/src/notification/workers/beat_schedule.py] — current state to extend
- [Source: eusolicit-app/services/notification/pyproject.toml] — current dependencies
- [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/workers/celery_app.py] — canonical pattern for task_routes, worker_prefetch_multiplier, task_acks_late
- [Source: eusolicit-app/services/data-pipeline/src/data_pipeline/workers/beat_schedule.py] — env-var override pattern
- [Source: eusolicit-docs/implementation-artifacts/5-2-celery-app-bootstrap-and-beat-schedule-configuration.md] — direct template for this story
- [Source: eusolicit-app/docker-compose.yml] — existing notification service block to reference for worker/beat

## Dev Agent Record

### Agent Model Used

claude-opus-4-5

### Debug Log References

### Completion Notes List

### File List
