---
stepsCompleted:
  - step-01-preflight-and-context
  - step-02-generation-mode
  - step-03-test-strategy
  - step-04-generate-tests
  - step-04c-aggregate
  - step-05-validate-and-complete
lastStep: step-05-validate-and-complete
lastSaved: '2026-04-19'
inputDocuments:
  - eusolicit-docs/implementation-artifacts/9-1-notification-service-scaffold-celery-configuration.md
  - eusolicit-docs/test-artifacts/test-design-epic-09.md
  - eusolicit-app/services/notification/tests/conftest.py
  - eusolicit-app/services/notification/pyproject.toml
  - eusolicit-app/services/notification/src/notification/workers/celery_app.py
  - eusolicit-app/services/notification/src/notification/workers/beat_schedule.py
  - _bmad/bmm/config.yaml
---

# ATDD Checklist: Story 9-1 — Notification Service Scaffold & Celery Configuration

**Date:** 2026-04-19
**Story status:** ready-for-dev
**TDD phase:** 🔴 RED (failing tests generated — implementation pending)
**Stack:** backend (Python / Celery / pytest)
**Generation mode:** AI generation (no browser recording needed)

---

## TDD Red Phase — Current State

All tests are marked `@pytest.mark.skip(reason="TDD RED PHASE: ...")`.
They define the **expected behaviour before implementation** and will be unskipped one
acceptance-criterion at a time as development lands.

| Test file | Location |
|---|---|
| `test_celery_config.py` | `eusolicit-app/services/notification/tests/unit/test_celery_config.py` |

**Total tests generated:** 34  
**All tests skipped (red phase):** ✅  
**E2E tests:** 0 (backend-only story — no browser tests required)

---

## Acceptance Criteria Coverage

### AC1 — Celery worker S09 task includes + stub modules importable

| Test ID | Description | Priority | Expected to fail |
|---|---|---|---|
| `test_s09_health_task_in_celery_includes` | health module in include list | P0 | ✅ |
| `test_s09_alert_matching_task_in_celery_includes` | alert_matching in include list | P0 | ✅ |
| `test_s09_send_email_task_in_celery_includes` | send_email in include list | P0 | ✅ |
| `test_s09_digest_task_in_celery_includes` | digest in include list | P0 | ✅ |
| `test_s09_calendar_sync_task_in_celery_includes` | calendar_sync in include list | P0 | ✅ |
| `test_health_task_module_importable` | health module importable | P0 | ✅ |
| `test_all_s09_stub_modules_importable` | all 5 stubs importable | P0 | ✅ |
| `test_health_task_returns_ok` | `health_check.apply().get() == "OK"` | P1 | ✅ |
| `test_health_task_name_is_notification_health` | task name == notification.health | P1 | ✅ |

**Why they fail now:** `notification.workers.tasks.health`, `alert_matching`, `send_email`, `digest`, `calendar_sync` modules do not exist. The `include=[]` list in `celery_app.py` does not include S09 modules.

---

### AC2 — Beat schedule has all S09 periodic entries (E09-INFRA-001)

| Test ID | Description | Priority | Expected to fail |
|---|---|---|---|
| `test_beat_schedule_contains_notification_heartbeat` | heartbeat key present | P0 | ✅ |
| `test_beat_schedule_contains_alert_digest_daily` | daily digest key present | P0 | ✅ |
| `test_beat_schedule_contains_alert_digest_weekly` | weekly digest key present | P0 | ✅ |
| `test_beat_schedule_contains_calendar_sync_periodic` | calendar sync key present | P0 | ✅ |
| `test_notification_heartbeat_schedule_is_60_seconds` | heartbeat == timedelta(seconds=60) | P0 | ✅ |
| `test_calendar_sync_periodic_default_interval_15_minutes` | calendar == timedelta(minutes=15) | P0 | ✅ |
| `test_calendar_sync_interval_env_override` | env var overrides interval | P2 | ✅ |
| `test_daily_digest_hour_env_override` | env var overrides digest hour | P2 | ✅ |
| `test_stripe_sync_entry_not_duplicated` | sync-usage-to-stripe-daily appears exactly once | P1 | ✅ |
| `test_celery_timezone_is_utc` | timezone==UTC and enable_utc==True | P0 | ✅ |

**Why they fail now:** `beat_schedule.py` does not yet contain S09 entries (`notification-heartbeat`, `alert-digest-daily`, `alert-digest-weekly`, `calendar-sync-periodic`). The env-var override pattern with `_CALENDAR_SYNC_MINUTES` has not been implemented.

---

### AC3 — Task routing delivers to correct queues (E09-INFRA-002)

| Test ID | Description | Priority | Expected to fail |
|---|---|---|---|
| `test_task_routes_health_to_notification_default` | health → notification-default | P0 | ✅ |
| `test_task_routes_match_alerts_to_alerts_queue` | match_alerts → alerts | P0 | ✅ |
| `test_task_routes_send_email_to_emails_queue` | send_email → emails | P0 | ✅ |
| `test_task_routes_sync_calendars_to_calendar_sync_queue` | sync_calendars → calendar-sync | P0 | ✅ |
| `test_task_routes_daily_digest_to_alerts_queue` | send_daily_digest → alerts | P0 | ✅ |
| `test_task_routes_weekly_digest_to_alerts_queue` | send_weekly_digest → alerts | P0 | ✅ |
| `test_task_routes_billing_sync_to_usage_reporting_queue` | billing sync → usage-reporting | P1 | ✅ |

**Why they fail now:** `celery.conf.task_routes` dict is not set in `celery_app.py` — `conf.update()` block does not include `task_routes`.

---

### AC4 — Retry policy / worker settings (E09-CELERY-001)

| Test ID | Description | Priority | Expected to fail |
|---|---|---|---|
| `test_worker_prefetch_multiplier_is_1` | prefetch_multiplier == 1 | P2 | ✅ |
| `test_task_acks_late_is_true` | acks_late is True | P2 | ✅ |
| `test_task_retry_backoff_is_true` | retry_backoff is True | P2 | ✅ |
| `test_task_retry_backoff_max_is_600` | retry_backoff_max == 600 | P2 | ✅ |
| `test_task_max_retries_is_5` | max_retries == 5 | P2 | ✅ |
| `test_stub_tasks_have_max_retries_5` | stub decorators set max_retries=5 | P2 | ✅ |
| `test_stub_tasks_have_default_retry_delay_60` | stub decorators set delay=60 | P2 | ✅ |

**Why they fail now:** `task_routes`, `worker_prefetch_multiplier`, `task_acks_late`, `task_retry_backoff`, `task_retry_backoff_max`, `task_max_retries` not yet in `celery.conf.update()` block. Stub task modules don't exist.

---

### AC5 — Dead-letter signal handler publishes to `notification:dead_letter`

| Test ID | Description | Priority | Expected to fail |
|---|---|---|---|
| `test_task_failure_signal_handler_registered` | on_task_failure connected to signal | P1 | ✅ |
| `test_dead_letter_handler_pushes_to_redis_list` | LPUSH entry to notification:dead_letter | P1 | ✅ |
| `test_dead_letter_handler_sets_7_day_ttl` | TTL set to 604800s (7 days) | P1 | ✅ |
| `test_dead_letter_handler_never_raises_on_redis_failure` | silently swallows Redis errors | P1 | ✅ |

**Why they fail now:** `on_task_failure` function and `@task_failure.connect` decorator not yet present in `celery_app.py`. `redis.from_url` import also missing.

---

### AC6 — Alembic migration 001 runs cleanly

| Test ID | Description | Priority | Expected to fail |
|---|---|---|---|
| `test_alembic_env_sets_notification_search_path` | env.py has search_path = notification | P1 | ✅ |
| `test_alembic_001_initial_migration_exists` | 001_*.py exists and creates _migrations_meta | P1 | ✅ |

**Why they fail now:** Tests assert the existing migration files contain specific strings (`search_path`, `_migrations_meta`). Will pass if files exist and are correctly scoped — verify content after reviewing actual alembic files.

---

### AC7 — Docker Compose notification-worker and notification-beat services

| Test ID | Description | Priority | Expected to fail |
|---|---|---|---|
| `test_docker_compose_has_notification_worker_service` | notification-worker: service block present | P1 | ✅ |
| `test_docker_compose_has_notification_beat_service` | notification-beat: service block present | P1 | ✅ |
| `test_docker_compose_worker_command_includes_all_s09_queues` | worker -Q has all 5 queues | P1 | ✅ |
| `test_docker_compose_beat_service_has_restart_on_failure` | beat has restart: on-failure | P1 | ✅ |
| `test_docker_compose_beat_command_uses_persistent_scheduler` | beat uses PersistentScheduler | P1 | ✅ |
| `test_pyproject_has_notification_beat_entry_point` | notification-beat in pyproject.toml scripts | P1 | ✅ |
| `test_pyproject_dev_deps_include_pytest_timeout` | pytest-timeout in dev deps | P2 | ✅ |
| `test_pyproject_dev_deps_include_freezegun` | freezegun in dev deps | P2 | ✅ |

**Why they fail now:** `notification-worker` and `notification-beat` services not yet in `docker-compose.yml`. `notification-beat` entry point not in `pyproject.toml`. `pytest-timeout` and `freezegun` not in dev deps.

---

## Epic Test Design Cross-Reference

| Epic Test ID | Coverage in this ATDD | Notes |
|---|---|---|
| E09-INFRA-001 | ✅ `test_beat_schedule_contains_*` (×4 keys) | All P0 Beat schedule entries |
| E09-INFRA-002 | ✅ `test_task_routes_*` (×7 routes) | All task → queue mappings |
| E09-INFRA-003 | ✅ `test_health_task_returns_ok` | health_check.apply().get() == "OK" |
| E09-INFRA-004 | ✅ `test_calendar_sync_interval_env_override` | importlib.reload pattern |
| E09-CELERY-001 | ✅ `test_worker_prefetch_multiplier_is_1` + backoff tests | Retry / prefetch config |

---

## TDD Green Phase — Implementation Checklist

After implementation is complete, remove `@pytest.mark.skip` markers in this order:

### Phase 1 — Task modules (unblock other tests)
```bash
# Implement Task 3.1-3.5 first, then unskip:
# test_health_task_module_importable
# test_all_s09_stub_modules_importable
# test_health_task_returns_ok
# test_health_task_name_is_notification_health
```

### Phase 2 — celery_app.py task includes (Task 1.1)
```bash
# After adding S09 modules to include=[]:
# test_s09_*_in_celery_includes (×5 tests)
```

### Phase 3 — Beat schedule S09 entries (Task 2.1 + 2.2)
```bash
# After adding S09 Beat entries to beat_schedule.py:
# test_beat_schedule_contains_* (×4 key tests)
# test_notification_heartbeat_schedule_is_60_seconds
# test_calendar_sync_periodic_default_interval_15_minutes
# test_calendar_sync_interval_env_override
# test_daily_digest_hour_env_override
# test_stripe_sync_entry_not_duplicated
# test_celery_timezone_is_utc
```

### Phase 4 — celery.conf.update() block (Task 1.2)
```bash
# After adding task_routes, retry config to conf.update():
# test_task_routes_* (×7 tests)
# test_worker_prefetch_multiplier_is_1
# test_task_acks_late_is_true
# test_task_retry_backoff_is_true
# test_task_retry_backoff_max_is_600
# test_task_max_retries_is_5
# test_stub_tasks_have_max_retries_5
# test_stub_tasks_have_default_retry_delay_60
```

### Phase 5 — Dead-letter signal handler (Task 1.3)
```bash
# After implementing on_task_failure:
# test_task_failure_signal_handler_registered
# test_dead_letter_handler_pushes_to_redis_list
# test_dead_letter_handler_sets_7_day_ttl
# test_dead_letter_handler_never_raises_on_redis_failure
```

### Phase 6 — Docker Compose + pyproject.toml (Tasks 4 + 5)
```bash
# After adding services to docker-compose.yml and updating pyproject.toml:
# test_docker_compose_has_notification_worker_service
# test_docker_compose_has_notification_beat_service
# test_docker_compose_worker_command_includes_all_s09_queues
# test_docker_compose_beat_service_has_restart_on_failure
# test_docker_compose_beat_command_uses_persistent_scheduler
# test_pyproject_has_notification_beat_entry_point
# test_pyproject_dev_deps_include_pytest_timeout
# test_pyproject_dev_deps_include_freezegun
```

### Phase 7 — Alembic migration (verify existing files)
```bash
# Verify alembic files contain expected strings, then unskip:
# test_alembic_env_sets_notification_search_path
# test_alembic_001_initial_migration_exists
```

---

## Running the Tests

```bash
# From eusolicit-app/ — run all (currently all skipped):
make test-service SVC=notification

# Run only unit tests:
cd services/notification && python -m pytest tests/unit/test_celery_config.py -v

# Run with skip summary to see what's pending:
cd services/notification && python -m pytest tests/unit/test_celery_config.py -v --tb=short -rs

# After removing skip markers, verify green phase:
cd services/notification && python -m pytest tests/unit/test_celery_config.py -v --tb=short
```

**Note:** Tests use `task_always_eager=True` from `tests/conftest.py` (`celery_config` fixture). No live Redis broker required for unit tests. Dead-letter handler tests use `fakeredis`.

---

## Risk Mitigations Covered

| Risk | Mitigation test |
|---|---|
| R-001: Redis stream event loss | `test_task_acks_late_is_true` — acks only after completion |
| DLQ handler masking errors | `test_dead_letter_handler_never_raises_on_redis_failure` |
| Beat singleton crash | `test_docker_compose_beat_service_has_restart_on_failure` |
| Env-var not overriding schedule | `test_calendar_sync_interval_env_override` |
| Duplicate Stripe sync entry | `test_stripe_sync_entry_not_duplicated` |

---

## Assumptions & Notes

1. `conftest.py` `celery_config` fixture with `task_always_eager=True` is inherited by all tests in
   `tests/unit/` — no broker needed for `health_check.apply()` test.
2. `fakeredis>=2.21` already in dev deps — dead-letter tests use it directly.
3. `importlib.reload()` pattern is required for env-var override tests because `beat_schedule.py`
   reads `os.environ` at **module import time**.
4. Docker Compose tests use file parsing (string search) — no Docker daemon required.
5. Alembic tests use file content inspection — no DB connection required.
6. `on_task_failure` is the function name expected in `celery_app.py` — the signal receiver
   identity test checks `str(receiver)` for this name.

---

**Generated by:** BMad TEA Agent — ATDD Workflow  
**Workflow:** `bmad-testarch-atdd`  
**Story:** 9-1-notification-service-scaffold-celery-configuration  
**Next recommended workflow:** `bmad-agent-dev` (implement Story 9-1 tasks)
