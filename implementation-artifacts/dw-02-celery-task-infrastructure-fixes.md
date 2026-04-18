# Story DW-02: Celery Task Infrastructure Fixes

Status: draft

**Origin:** Deferred work from code review of S07.05 (2026-04-17)
**Priority:** High — `reset_stuck_proposals_task` cannot be discovered by Celery Beat in production
**Epic:** Post-E07 cleanup (no assigned epic)
**Points:** 2 | **Type:** backend

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **platform operator**,
I want the **Celery Beat task that resets stuck proposal generation status to actually run on schedule**,
so that proposals stuck in `generating` state are automatically recovered instead of blocking users from re-triggering generation indefinitely.

## Acceptance Criteria

1. **`@celery_app.task` decorator on `reset_stuck_proposals_task`** — The function in `services/client-api/src/client_api/tasks/proposal_tasks.py` must be decorated with `@celery_app.task(name="client_api.reset_stuck_proposals_task")` so Celery Beat can discover and schedule it by name. Currently the function exists but has no task decorator, making it invisible to the scheduler.

2. **Correct async execution pattern** — The task currently uses `asyncio.run()` which fails when an event loop is already running (Celery async workers). Replace with a sync-safe pattern:
   - If the worker pool is `prefork` (current infrastructure): use `asyncio.run()` only if no event loop is running; use `loop.run_until_complete()` otherwise. Detect with `try: loop = asyncio.get_event_loop() / except RuntimeError: loop = asyncio.new_event_loop()`.
   - **Preferred**: Refactor `reset_stuck_generating_proposals(session)` service function to have a thin sync wrapper that creates a new event loop explicitly: `asyncio.new_event_loop().run_until_complete(...)`. This is prefork-safe.
   - The cleanest solution: add a dedicated sync entry point `def reset_stuck_proposals_sync() -> int` that wraps the async service function, and call that from the Celery task.

3. **Beat schedule registration** — Add the task to the Celery Beat schedule in the client-api Celery app configuration:
   ```python
   beat_schedule = {
       "reset-stuck-proposals": {
           "task": "client_api.reset_stuck_proposals_task",
           "schedule": 60.0,  # every 60 seconds, as specified in S07.05 AC7
       }
   }
   ```
   Verify this is registered in `services/client-api/src/client_api/celery_app.py` (or wherever the Beat schedule is configured in this service).

4. **Tests pass in both eager and non-eager mode** — The existing tests in `tests/api/test_proposal_generate.py` that cover E07-P0-002 (`generation_status` stuck cleanup) currently run with `CELERY_TASK_ALWAYS_EAGER=True`. After this fix:
   - Eager mode tests must still pass (decorator does not break eager execution).
   - Add one new test that directly calls `reset_stuck_proposals_task.delay()` (non-eager) and asserts the task succeeds when an event loop is not running in the calling context.

5. **No regression in proposal generate tests** — `pytest services/client-api/tests/api/test_proposal_generate.py -v` passes with all tests green after this fix.

## Tasks / Subtasks

- [ ] Task 1: Add `@celery_app.task` decorator to `reset_stuck_proposals_task` (AC: 1)
  - [ ] 1.1 Open `services/client-api/src/client_api/tasks/proposal_tasks.py`
  - [ ] 1.2 Import `celery_app` from the client-api Celery app module
  - [ ] 1.3 Add `@celery_app.task(name="client_api.reset_stuck_proposals_task", bind=False)` decorator
  - [ ] 1.4 Verify `celery -A client_api.celery_app inspect registered` lists the task (test in Docker Compose environment)

- [ ] Task 2: Fix async execution pattern (AC: 2)
  - [ ] 2.1 Replace `asyncio.run(reset_stuck_generating_proposals(...))` with a sync wrapper:
    ```python
    def _run_stuck_cleanup() -> int:
        loop = asyncio.new_event_loop()
        try:
            async def _inner():
                async with session_factory() as session:
                    return await reset_stuck_generating_proposals(session)
            return loop.run_until_complete(_inner())
        finally:
            loop.close()
    ```
  - [ ] 2.2 The task body calls `_run_stuck_cleanup()` instead of `asyncio.run(...)`
  - [ ] 2.3 Do NOT use `asyncio.get_event_loop()` — it is deprecated in Python 3.12; always create a new loop

- [ ] Task 3: Register Beat schedule (AC: 3)
  - [ ] 3.1 Open `services/client-api/src/client_api/celery_app.py`
  - [ ] 3.2 Add the `beat_schedule` entry for `reset-stuck-proposals` (60s interval)
  - [ ] 3.3 Verify other existing Beat tasks in this file are not disturbed

- [ ] Task 4: Update tests (AC: 4, 5)
  - [ ] 4.1 In `tests/api/test_proposal_generate.py`, find the existing stuck-cleanup test (E07-P0-002)
  - [ ] 4.2 Add assertion that after `reset_stuck_proposals_task()` runs, the proposal's `generation_status` is reset to `idle`
  - [ ] 4.3 Run `pytest services/client-api/tests/api/test_proposal_generate.py -v` and confirm all tests pass

## Dev Notes

### Why `asyncio.run()` Fails in Celery Workers

`asyncio.run()` creates a new event loop, runs the coroutine, then closes the loop. If the Celery worker (even prefork) has already created an event loop in the main thread (e.g., from FastAPI startup lifespan or other background tasks), `asyncio.run()` raises `RuntimeError: This event loop is already running`.

The `asyncio.new_event_loop().run_until_complete()` pattern with explicit close avoids this because it never touches the running loop — it creates a private loop in the function scope.

### Finding the Celery App Instance

The existing Beat schedule for the notification service is in `services/notification/src/notification/celery_app.py` as a reference. The client-api Celery app should follow the same pattern. Check `services/client-api/src/client_api/celery_app.py` for the `app = Celery(...)` instantiation.

### Verify Task Registration

In Docker Compose:
```bash
docker compose exec client-api celery -A client_api.celery_app inspect registered
```
The task `client_api.reset_stuck_proposals_task` should appear in the list.

### References

- [Source: eusolicit-docs/implementation-artifacts/deferred-work.md] — DW-02 origin item
- [Source: eusolicit-docs/implementation-artifacts/7-5-ai-draft-generation-backend-sse-integration.md#AC7] — original task spec
- [Source: eusolicit-app/services/client-api/src/client_api/tasks/proposal_tasks.py] — target file
- [Source: eusolicit-app/services/client-api/src/client_api/celery_app.py] — Beat schedule config
- [Source: eusolicit-app/services/notification/src/notification/celery_app.py] — reference Beat schedule pattern
