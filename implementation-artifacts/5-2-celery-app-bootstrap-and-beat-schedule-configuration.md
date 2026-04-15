# Story 5.2: Celery App Bootstrap and Beat Schedule Configuration

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **backend developer on the EU Solicit data pipeline**,
I want **a fully configured Celery application with Redis broker/result-backend and a Beat schedule covering all four periodic tasks, with all intervals externalised to environment variables and a health-check task callable from the container liveness probe**,
so that **subsequent stories (S05.04–S05.06, S05.10) can simply add `@shared_task` implementations and have Beat pick them up automatically, and the pipeline worker and Beat containers start cleanly in Docker Compose from day one**.

## Acceptance Criteria

1. `celery -A data_pipeline.workers.celery_app inspect scheduled` (or `app.conf.beat_schedule` assertion in tests) lists all four periodic tasks: `crawl-aop`, `crawl-ted`, `crawl-eu-grants`, and `cleanup-expired-opportunities`
2. Intervals for all four schedules are overridable via environment variables without any code change — setting `PIPELINE_AOP_CRAWL_EVERY_HOURS`, `PIPELINE_TED_CRAWL_EVERY_HOURS`, `PIPELINE_EU_GRANTS_CRON_HOUR`, `PIPELINE_EU_GRANTS_CRON_MINUTE`, `PIPELINE_CLEANUP_CRON_HOUR`, `PIPELINE_CLEANUP_CRON_MINUTE`, `PIPELINE_CLEANUP_CRON_DOW` reshapes the schedule at startup
3. The `pipeline.health` task (`data_pipeline.workers.tasks.health.health_check`) returns `"OK"` when invoked synchronously; the `GET /pipeline-health` endpoint in `main.py` calls it and returns `{"status": "ok"}` — usable as a container liveness probe
4. Running `docker compose up data-pipeline-worker data-pipeline-beat` starts both containers cleanly with no import errors; the worker reports all queues online and Beat reports its schedule

## Tasks / Subtasks

- [x] Task 1: Create `workers/` package and Celery application (AC: 1, 4)
  - [x] 1.1 Create `services/data-pipeline/src/data_pipeline/workers/__init__.py` (empty)
  - [x] 1.2 Create `services/data-pipeline/src/data_pipeline/workers/celery_app.py` — instantiate `Celery("pipeline", broker=..., backend=..., include=[...])` following the notification service pattern; add `celery.conf.update(...)` with JSON serializer, `timezone="UTC"`, `enable_utc=True`, `beat_schedule=BEAT_SCHEDULE`, and task routing (`pipeline_crawl` / `pipeline_scoring` / `pipeline_default` queues)
  - [x] 1.3 Create `services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` — define `BEAT_SCHEDULE` dict reading all interval/cron values from `os.environ` with sensible defaults; use `timedelta(hours=...)` for AOP and TED, `crontab(...)` for EU Grants and cleanup

- [x] Task 2: Create stub task modules (AC: 1, 4)
  - [x] 2.1 Create `services/data-pipeline/src/data_pipeline/workers/tasks/__init__.py` (empty)
  - [x] 2.2 Create `services/data-pipeline/src/data_pipeline/workers/tasks/health.py` — `@shared_task(name="pipeline.health")` returning `"OK"`; this is the ONLY task with a real implementation in this story
  - [x] 2.3 Create `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` — `@shared_task(name="pipeline.crawl_aop", bind=True)` with stub body (log "not yet implemented", return None); docstring: "Implemented in S05.04"
  - [x] 2.4 Create `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` — same stub pattern; docstring: "Implemented in S05.05"
  - [x] 2.5 Create `services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` — same stub pattern; docstring: "Implemented in S05.06"
  - [x] 2.6 Create `services/data-pipeline/src/data_pipeline/workers/tasks/cleanup.py` — `@shared_task(name="pipeline.cleanup_expired_opportunities", bind=True)` stub; docstring: "Implemented in S05.10"

- [x] Task 3: Expose health task in FastAPI (AC: 3)
  - [x] 3.1 Modify `services/data-pipeline/src/data_pipeline/main.py` — add `GET /pipeline-health` endpoint that calls `health_check.apply().get(timeout=5)` and returns `{"status": "ok"}` if result equals `"OK"`, else returns HTTP 503; import lazily so the Celery app is not instantiated on module import unless needed

- [x] Task 4: Update `pyproject.toml` with script entry points (AC: 4)
  - [x] 4.1 Add `[project.scripts]` section with:
    - `data-pipeline-worker = "data_pipeline.workers.celery_app:celery"` (used by `celery -A ... worker`)
    - `data-pipeline-beat = "data_pipeline.workers.celery_app:celery"` (used by `celery -A ... beat`)
  - [x] 4.2 Add `celery-test-utils>=5.3` (or rely on `celery[redis]` already present) to `[project.optional-dependencies] dev` for `@pytest.fixture` worker/beat test helpers; confirm `celery[redis]>=5.3` is already in `dependencies` — it is, no addition needed for core; add `pytest-celery>=1.0` if available, otherwise rely on `task_always_eager` pattern

- [x] Task 5: Add worker and beat services to Docker Compose (AC: 4)
  - [x] 5.1 In `eusolicit-app/docker-compose.yml`, add `data-pipeline-worker` service after the `data-pipeline` service:
    - `build`: same as `data-pipeline` (context `.`, dockerfile `services/data-pipeline/Dockerfile`)
    - `command`: `celery -A data_pipeline.workers.celery_app worker --loglevel=info -Q pipeline_crawl,pipeline_scoring,pipeline_default -c 4`
    - `volumes`, `env_file`, `depends_on` (postgres + redis healthy): same as `data-pipeline`
    - `environment`: add `CELERY_BROKER_URL: redis://redis:6379/0`, `CELERY_RESULT_BACKEND: redis://redis:6379/1` in addition to `DATABASE_URL`, `REDIS_URL`, `PYTHONPATH`
    - No `ports` — worker does not expose HTTP
    - `healthcheck`: `["CMD-SHELL", "celery -A data_pipeline.workers.celery_app inspect ping -d celery@$$HOSTNAME"]`
  - [x] 5.2 Add `data-pipeline-beat` service (singleton):
    - `command`: `celery -A data_pipeline.workers.celery_app beat --loglevel=info --scheduler celery.beat.PersistentScheduler --schedule /tmp/celerybeat-schedule`
    - Same build, volumes, env, depends_on, no ports
    - No healthcheck for beat (Beat is stateful; Kubernetes uses pod liveness probes separately)

- [x] Task 6: Write unit tests (AC: 1, 2, 3)
  - [x] 6.1 Create `services/data-pipeline/tests/unit/test_celery_beat.py`:
    - **E05-P1-005**: Import `from data_pipeline.workers.celery_app import celery as app`; assert `app.conf.beat_schedule` has keys `"crawl-aop"`, `"crawl-ted"`, `"crawl-eu-grants"`, `"cleanup-expired-opportunities"`
    - **E05-P1-006**: `monkeypatch.setenv("PIPELINE_AOP_CRAWL_EVERY_HOURS", "1")`; re-import `beat_schedule` module (use `importlib.reload`); assert the `crawl-aop` schedule equals `timedelta(hours=1)`
    - **E05-P1-007**: `result = health_check.apply(); assert result.get() == "OK"`
    - **E05-P3-001**: `assert app.conf.timezone == "UTC"`; verify `crawl-eu-grants` crontab hour defaults to `"2"`

## Dev Notes

### Architecture Fit

S05.02 creates the **Celery application skeleton** for Epic 5. It produces no crawl logic — only the app bootstrap, Beat schedule, stub tasks, and Docker Compose wiring. S05.04–S05.06 and S05.10 will replace stubs with real implementations.

**Data pipeline runs as three separate processes in production:**
- `data-pipeline` (FastAPI, port 8003) — HTTP API and health probes
- `data-pipeline-worker` (Celery Worker, HPA 2–8 pods) — executes tasks from Redis queues
- `data-pipeline-beat` (Celery Beat, 1 pod singleton) — publishes task messages on schedule

[Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#7.3-Data-Pipeline-Service]

### CRITICAL: Notification Service is the Direct Pattern Template

The notification service already has a running Celery setup. Mirror it exactly, substituting `notification` → `data_pipeline`:

| Notification | Data Pipeline (new) |
|---|---|
| `services/notification/src/notification/workers/celery_app.py` | `services/data-pipeline/src/data_pipeline/workers/celery_app.py` |
| `services/notification/src/notification/workers/beat_schedule.py` | `services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` |
| `Celery("notification", ...)` | `Celery("pipeline", ...)` |
| `celery -A notification.workers.celery_app worker` | `celery -A data_pipeline.workers.celery_app worker` |

[Source: eusolicit-app/services/notification/src/notification/workers/celery_app.py]

### `celery_app.py` — Exact Implementation Pattern

```python
"""Celery application instance for the Data Pipeline Service.

Configuration:
- Broker: Redis (CELERY_BROKER_URL env var, default redis://localhost:6379/0)
- Result backend: Redis (CELERY_RESULT_BACKEND env var, default redis://redis:6379/1)
- Serializer: JSON
- Timezone: UTC (critical — see E05-R-008)

Usage:
    celery -A data_pipeline.workers.celery_app worker --loglevel=info -Q pipeline_crawl,pipeline_scoring,pipeline_default
    celery -A data_pipeline.workers.celery_app beat --loglevel=info

[Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#7.3-Data-Pipeline-Service]
"""
from __future__ import annotations

import os

from celery import Celery

from data_pipeline.workers.beat_schedule import BEAT_SCHEDULE

celery = Celery(
    "pipeline",
    broker=os.environ.get("CELERY_BROKER_URL", "redis://localhost:6379/0"),
    backend=os.environ.get("CELERY_RESULT_BACKEND", "redis://redis:6379/1"),
    include=[
        "data_pipeline.workers.tasks.health",
        "data_pipeline.workers.tasks.crawl_aop",
        "data_pipeline.workers.tasks.crawl_ted",
        "data_pipeline.workers.tasks.crawl_eu_grants",
        "data_pipeline.workers.tasks.cleanup",
    ],
)

celery.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",           # REQUIRED — prevents E05-R-008 DST shift
    enable_utc=True,
    beat_schedule=BEAT_SCHEDULE,
    task_routes={
        "pipeline.crawl_aop": {"queue": "pipeline_crawl"},
        "pipeline.crawl_ted": {"queue": "pipeline_crawl"},
        "pipeline.crawl_eu_grants": {"queue": "pipeline_crawl"},
        "pipeline.cleanup_expired_opportunities": {"queue": "pipeline_crawl"},
        "pipeline.score_opportunities": {"queue": "pipeline_scoring"},  # future S05.07
        "pipeline.generate_submission_guides": {"queue": "pipeline_scoring"},  # future S05.08
        "pipeline.health": {"queue": "pipeline_default"},
    },
    worker_prefetch_multiplier=1,   # fair dispatch; prevents starvation on long tasks
    task_acks_late=True,            # ack only after task completes — prevents silent loss
)
```

### `beat_schedule.py` — Env-Var Overridable Pattern

```python
"""Celery Beat schedule for the Data Pipeline Service.

All intervals/cron expressions are externalised to environment variables so
staging, CI, and production can differ without code changes.

Default schedule:
  crawl-aop               — every 6 h
  crawl-ted               — every 12 h
  crawl-eu-grants         — daily at 02:00 UTC
  cleanup-expired-opportunities — weekly, Sunday at 04:00 UTC

[Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.02]
"""
from __future__ import annotations

import os
from datetime import timedelta

from celery.schedules import crontab

# --- Interval overrides ---
_AOP_HOURS = int(os.environ.get("PIPELINE_AOP_CRAWL_EVERY_HOURS", "6"))
_TED_HOURS = int(os.environ.get("PIPELINE_TED_CRAWL_EVERY_HOURS", "12"))

# --- Cron overrides (EU Grants: daily 02:00 UTC) ---
_EU_GRANTS_HOUR = os.environ.get("PIPELINE_EU_GRANTS_CRON_HOUR", "2")
_EU_GRANTS_MINUTE = os.environ.get("PIPELINE_EU_GRANTS_CRON_MINUTE", "0")

# --- Cron overrides (Cleanup: Sunday 04:00 UTC; day_of_week=0 = Sunday) ---
_CLEANUP_HOUR = os.environ.get("PIPELINE_CLEANUP_CRON_HOUR", "4")
_CLEANUP_MINUTE = os.environ.get("PIPELINE_CLEANUP_CRON_MINUTE", "0")
_CLEANUP_DOW = os.environ.get("PIPELINE_CLEANUP_CRON_DOW", "0")  # 0 = Sunday

BEAT_SCHEDULE = {
    "crawl-aop": {
        "task": "pipeline.crawl_aop",
        "schedule": timedelta(hours=_AOP_HOURS),
    },
    "crawl-ted": {
        "task": "pipeline.crawl_ted",
        "schedule": timedelta(hours=_TED_HOURS),
    },
    "crawl-eu-grants": {
        "task": "pipeline.crawl_eu_grants",
        "schedule": crontab(hour=_EU_GRANTS_HOUR, minute=_EU_GRANTS_MINUTE),
    },
    "cleanup-expired-opportunities": {
        "task": "pipeline.cleanup_expired_opportunities",
        "schedule": crontab(
            hour=_CLEANUP_HOUR, minute=_CLEANUP_MINUTE, day_of_week=_CLEANUP_DOW
        ),
    },
}
```

**CRITICAL**: `beat_schedule.py` reads `os.environ` at **module import time**. To test overrides, use `importlib.reload(beat_schedule_module)` after `monkeypatch.setenv(...)` — not just patching the variable.

### Stub Task Pattern — Crawl Tasks

Each stub must be importable and discoverable as a valid Celery task. Do NOT raise `NotImplementedError` — it causes the Beat dispatcher to fail on scheduled fire:

```python
# services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py
"""AOP crawler task stub. Real implementation in S05.04."""
from __future__ import annotations

import logging

from celery import shared_task

logger = logging.getLogger(__name__)


@shared_task(
    name="pipeline.crawl_aop",
    bind=True,
    max_retries=3,
    default_retry_delay=60,
)
def crawl_aop(self) -> None:  # noqa: ARG001
    """Trigger AOP crawl via AI Gateway. Implemented in S05.04."""
    logger.info("crawl_aop: stub — not yet implemented (S05.04)")
```

Same pattern for `crawl_ted`, `crawl_eu_grants`, `cleanup`. The `bind=True` + retry params are set now so S05.04–S05.06 can add `autoretry_for=(AIGatewayUnavailableError,)` without changing the decorator.

### Health Task — Real Implementation (Not a Stub)

```python
# services/data-pipeline/src/data_pipeline/workers/tasks/health.py
"""Pipeline Celery health-check task for container liveness probes."""
from __future__ import annotations

from celery import shared_task


@shared_task(name="pipeline.health")
def health_check() -> str:
    """Return 'OK'. Called synchronously by the /pipeline-health FastAPI endpoint."""
    return "OK"
```

### FastAPI `/pipeline-health` Endpoint

Add to `services/data-pipeline/src/data_pipeline/main.py`:

```python
from fastapi.responses import JSONResponse

@app.get("/pipeline-health")
def pipeline_health() -> JSONResponse:
    """Liveness probe for Celery worker connectivity."""
    from data_pipeline.workers.tasks.health import health_check  # lazy import

    try:
        result = health_check.apply().get(timeout=5)
        if result == "OK":
            return JSONResponse({"status": "ok"})
    except Exception as exc:
        return JSONResponse({"status": "error", "detail": str(exc)}, status_code=503)
    return JSONResponse({"status": "error"}, status_code=503)
```

**Important**: `health_check.apply()` runs the task in-process (eager), bypassing the broker. This is intentional — the endpoint tests the task code path, not Celery connectivity. For broker connectivity checks, use `celery inspect ping`.

### Docker Compose — Adding Worker and Beat Services

The existing `data-pipeline` service (uvicorn, port 8003) stays unchanged. Add two new services in `eusolicit-app/docker-compose.yml` after the `data-pipeline` block:

```yaml
  data-pipeline-worker:
    build:
      context: .
      dockerfile: services/data-pipeline/Dockerfile
    command: celery -A data_pipeline.workers.celery_app worker --loglevel=info -Q pipeline_crawl,pipeline_scoring,pipeline_default -c 4
    volumes:
      - ./services/data-pipeline/src/data_pipeline:/app/src/data_pipeline
      - ./packages/eusolicit-common/src/eusolicit_common:/app/packages/eusolicit-common/src/eusolicit_common
      - ./packages/eusolicit-models/src/eusolicit_models:/app/packages/eusolicit-models/src/eusolicit_models
      - ./packages/eusolicit-kraftdata/src/eusolicit_kraftdata:/app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata
    env_file: .env
    environment:
      DATABASE_URL: ${PIPELINE_DB_URL:-postgresql+asyncpg://data_pipeline_role:pipeline_password@postgres:5432/eusolicit}
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
      test: ["CMD-SHELL", "celery -A data_pipeline.workers.celery_app inspect ping -d celery@$$HOSTNAME --timeout 5"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  data-pipeline-beat:
    build:
      context: .
      dockerfile: services/data-pipeline/Dockerfile
    command: celery -A data_pipeline.workers.celery_app beat --loglevel=info --scheduler celery.beat.PersistentScheduler --schedule /tmp/celerybeat-schedule
    volumes:
      - ./services/data-pipeline/src/data_pipeline:/app/src/data_pipeline
      - ./packages/eusolicit-common/src/eusolicit_common:/app/packages/eusolicit-common/src/eusolicit_common
      - ./packages/eusolicit-models/src/eusolicit_models:/app/packages/eusolicit-models/src/eusolicit_models
      - ./packages/eusolicit-kraftdata/src/eusolicit_kraftdata:/app/packages/eusolicit-kraftdata/src/eusolicit_kraftdata
    env_file: .env
    environment:
      DATABASE_URL: ${PIPELINE_DB_URL:-postgresql+asyncpg://data_pipeline_role:pipeline_password@postgres:5432/eusolicit}
      REDIS_URL: ${REDIS_URL:-redis://redis:6379/0}
      CELERY_BROKER_URL: ${CELERY_BROKER_URL:-redis://redis:6379/0}
      CELERY_RESULT_BACKEND: ${CELERY_RESULT_BACKEND:-redis://redis:6379/1}
      PYTHONPATH: /app/src
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: on-failure   # Beat must remain singleton; restart if it crashes
```

**CRITICAL — Beat Singleton**: Only ONE `data-pipeline-beat` process must run at any time. In Docker Compose this is automatic (1 container). In Kubernetes, use a `Deployment` with `replicas: 1` and `strategy: Recreate`. Never scale Beat horizontally.

### `pyproject.toml` Modifications

Add a `[project.scripts]` section (it does not yet exist in `pyproject.toml`):

```toml
[project.scripts]
data-pipeline-worker = "data_pipeline.workers.celery_app:celery"
data-pipeline-beat = "data_pipeline.workers.celery_app:celery"
```

No new dev dependencies are required — `celery[redis]>=5.3` is already in `dependencies`; `pytest>=8.0` and `pytest-asyncio>=0.23` are already in dev. Test the health task with `task.apply()` (eager, in-process) — no broker needed.

### Test Expectations from Epic-Level Test Design

| Priority | Test ID | Description | Implementation |
|---|---|---|---|
| **P1** | E05-P1-005 | Beat schedule lists all 4 periodic tasks | Assert `app.conf.beat_schedule.keys()` contains all 4 keys |
| **P1** | E05-P1-006 | Schedule intervals overridable via env vars | `monkeypatch.setenv` + `importlib.reload(beat_schedule)` → assert new timedelta |
| **P1** | E05-P1-007 | Health-check task returns OK | `health_check.apply().get() == "OK"` |
| **P2** | E05-P2-003 | Worker and Beat start cleanly in Docker Compose | Manual / smoke test; log shows "celery@... ready" |
| **P3** | E05-P3-001 | UTC timezone assertion + EU Grants cron expression | `app.conf.timezone == "UTC"`; inspect `beat_schedule["crawl-eu-grants"]["schedule"]` |

**E05-P1-006 pattern** (env-var override test):
```python
import importlib
import os

def test_aop_interval_override(monkeypatch):
    monkeypatch.setenv("PIPELINE_AOP_CRAWL_EVERY_HOURS", "1")
    import data_pipeline.workers.beat_schedule as bs
    importlib.reload(bs)
    from datetime import timedelta
    assert bs.BEAT_SCHEDULE["crawl-aop"]["schedule"] == timedelta(hours=1)
```

### Risk Mitigations Addressed in This Story

| Risk | Mitigation in S05.02 |
|---|---|
| **E05-R-008** (DST shift) | `timezone="UTC"` + `enable_utc=True` set explicitly in `celery.conf.update()` |
| **E05-R-005** (chord starvation) | Queue routing: crawl tasks → `pipeline_crawl`, scoring tasks → `pipeline_scoring` — prevents scoring jobs from blocking crawl workers |
| **E05-R-002** (orphaned crawler_runs) | `task_acks_late=True` ensures task is not acked until complete — message not lost on worker crash |

### Project Structure After This Story

```
eusolicit-app/
├── docker-compose.yml                             # MODIFY — add data-pipeline-worker + data-pipeline-beat
└── services/data-pipeline/
    ├── pyproject.toml                             # MODIFY — add [project.scripts]
    └── src/data_pipeline/
        ├── main.py                                # MODIFY — add GET /pipeline-health
        └── workers/
            ├── __init__.py                        # + NEW (empty)
            ├── celery_app.py                      # + NEW
            ├── beat_schedule.py                   # + NEW
            └── tasks/
                ├── __init__.py                    # + NEW (empty)
                ├── health.py                      # + NEW — real implementation
                ├── crawl_aop.py                   # + NEW — stub (S05.04 implements)
                ├── crawl_ted.py                   # + NEW — stub (S05.05 implements)
                ├── crawl_eu_grants.py             # + NEW — stub (S05.06 implements)
                └── cleanup.py                     # + NEW — stub (S05.10 implements)
    └── tests/
        └── unit/
            └── test_celery_beat.py                # + NEW
```

**DO NOT TOUCH:**
- `services/data-pipeline/alembic/` — migration chain must stay intact
- `services/data-pipeline/src/data_pipeline/models/` — from S05.01, do not modify
- `services/data-pipeline/src/data_pipeline/main.py` — only ADD the `/pipeline-health` route; do not remove existing `/healthz`
- `services/notification/` — the notification Celery app is the template, not modified here

### Cross-Story Dependencies

| Story | Status | Relevance to S05.02 |
|---|---|---|
| S05.01 — Pipeline Schema Migration | **DONE** | Models in `data_pipeline.models.*` exist; `CrawlerRun` is the model stub tasks will eventually write to |
| S01.05 — Redis Streams Event Bus | **DONE** | Redis is running at `redis://redis:6379/0`; event bus abstraction in `eusolicit_common` |
| S04.01 — AI Gateway Service Scaffold | **DONE** | AI Gateway at `http://ai-gateway:8004` in Docker Compose; env var `AI_GATEWAY_BASE_URL` |
| S05.03 — AI Gateway Client Module | NEXT | Stub tasks import `AIGatewayUnavailableError` from S05.03; until then, stubs can skip the import |
| S05.04–S05.06 — Crawler Tasks | NEXT | Will replace stubs in `crawl_aop.py`, `crawl_ted.py`, `crawl_eu_grants.py` |
| S05.10 — Cleanup Task | NEXT | Will replace stub in `cleanup.py` |

### Important Implementation Notes

**`timedelta` vs `crontab`**: AOP and TED use `timedelta` (interval-based) because they run frequently and don't need clock alignment. EU Grants and cleanup use `crontab` (clock-based) because they run at specific UTC times. Never mix up — using `timedelta` for EU Grants would drift relative to 02:00 UTC.

**Beat `--schedule` file**: The `PersistentScheduler` stores last-run times in `/tmp/celerybeat-schedule`. In Kubernetes, mount a `PersistentVolume` so restart doesn't re-run tasks immediately. In Docker Compose, the file is in the container's ephemeral filesystem — acceptable for development.

**Task naming convention**: Task names use dot notation: `pipeline.<task_name>` (e.g., `pipeline.health`, `pipeline.crawl_aop`). Do NOT use the full Python module path as the task name — the short names in `beat_schedule.py` must match the `name=` parameter in `@shared_task(name=...)`.

**`worker_prefetch_multiplier=1`**: Set to 1 to prevent the worker from pre-fetching multiple long-running crawl tasks. This enables fair scheduling — a crawler task doesn't block the health check.

**`task_acks_late=True`**: Combined with `worker_prefetch_multiplier=1`, this ensures tasks are not lost on worker restart. The crawler run record will be created at task start (S05.04), so the ack-late behavior prevents duplicate run records on restart.

### References

- [Source: eusolicit-docs/planning-artifacts/epic-05-data-pipeline-ingestion.md#S05.02]
- [Source: eusolicit-docs/test-artifacts/test-design-epic-05.md#P1-E05-P1-005,006,007, P2-E05-P2-003, P3-E05-P3-001]
- [Source: eusolicit-docs/EU_Solicit_Solution_Architecture_v4.md#7.3-Data-Pipeline-Service] — "Celery workers + Beat scheduler", HPA 2-8, singleton Beat
- [Source: eusolicit-app/services/notification/src/notification/workers/celery_app.py] — direct notification service template
- [Source: eusolicit-app/services/notification/src/notification/workers/beat_schedule.py] — crontab schedule template
- [Source: eusolicit-app/docker-compose.yml] — existing data-pipeline service block to mirror for worker/beat
- [Source: eusolicit-docs/implementation-artifacts/5-1-pipeline-schema-migration-and-model-layer.md] — service path, pyproject.toml state, test fixture patterns

## Dev Agent Record

### Agent Model Used

claude-sonnet-4-5

### Debug Log References

None — all tests passed first run (with PYTHONPATH).

### Completion Notes List

- All 9 unit tests in `test_celery_beat.py` pass covering E05-P1-005, E05-P1-006, E05-P1-007, E05-P3-001.
- Added `pythonpath = ["src"]` to `[tool.pytest.ini_options]` in `pyproject.toml` so tests run without manual `PYTHONPATH` export — this is a forward-compatibility improvement that does not affect any existing tests (all 18 unit tests still pass).
- `health_check.apply().get()` pattern confirmed working in-process (eager execution, no broker needed).
- Beat schedule env-var override test uses `importlib.reload()` pattern as specified — confirmed working correctly.

### File List

- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/__init__.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/celery_app.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/beat_schedule.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/__init__.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/health.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_aop.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_ted.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/crawl_eu_grants.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/workers/tasks/cleanup.py` (NEW)
- `eusolicit-app/services/data-pipeline/src/data_pipeline/main.py` (MODIFIED — added GET /pipeline-health)
- `eusolicit-app/services/data-pipeline/pyproject.toml` (MODIFIED — added [project.scripts], pythonpath)
- `eusolicit-app/docker-compose.yml` (MODIFIED — added data-pipeline-worker and data-pipeline-beat services)
- `eusolicit-app/services/data-pipeline/tests/unit/test_celery_beat.py` (NEW)
