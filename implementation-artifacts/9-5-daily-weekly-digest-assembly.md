# Story: 9-5-daily-weekly-digest-assembly

## Epic
Epic 09: Notifications, Alerts & Calendar

## Metadata
- **Points**: 3
- **Type**: backend
- **Module**: Notification Service

## Description
Implement Celery Beat periodic tasks for daily digest (runs at 07:00 UTC) and weekly digest (runs Monday 07:00 UTC). Each task: queries `client.alert_preferences` for active preferences with matching schedule, collects opportunities ingested since last digest (tracked via `notification.alert_log.sent_at`), matches each opportunity against user preferences using the same logic as S09.04, groups matched opportunities per user into a digest payload (sorted by relevance/deadline), and dispatches a single digest email per user via the SendGrid task. Skip users with zero matches. Record in `alert_log` with `digest_type = 'daily'` or `'weekly'` and all matched `opportunity_ids`.

## Acceptance Criteria
- [x] Daily digest task triggers at 07:00 UTC and processes all daily-schedule preferences
- [x] Weekly digest task triggers Monday 07:00 UTC and processes all weekly-schedule preferences
- [x] Digest groups multiple matched opportunities into a single email per user
- [x] Users with zero matches receive no email
- [x] `alert_log` records all opportunity IDs included in each digest
- [x] Digest window is correctly calculated from last sent digest per user (no duplicates, no gaps)

## Implementation Notes
- Use `celery.schedules.crontab` for scheduling.
- For the digest window, query `MAX(sent_at)` from `alert_log` per user per digest_type; fall back to 24h/7d ago if no prior digest.
- Batch DB queries: load all active preferences, then batch-fetch opportunities in the window, then match in Python to avoid N+1 queries.

## Test Expectations (from Epic Test Design)
- **Medium-Priority Risk R-004 (Score 4)**: Daily digest matching logic CPU/DB spike.
  - **Mitigation**: In-memory preference caching, batch queries.
- **P0 Critical Path Test (API/Worker)**: Daily/Weekly Digest.
  - **Steps**: Validate accurate opportunity grouping & exclusion of zero-match users.
  - **Notes**: Run on every commit. Owner: QA.

## Tasks/Subtasks
- [x] Task 1: Create daily and weekly digest Celery periodic tasks
- [x] Task 2: Implement preference and opportunity batch fetching logic
- [x] Task 3: Implement matching and grouping logic for digests
- [x] Task 4: Integrate email dispatch and `alert_log` recording

## Senior Developer Review
**Status:** Changes Requested

**Findings:**
1. **Missing Test Coverage for FIX-2 (Race Condition Upper Bound):** While the implementation correctly adds the `Opportunity.created_at <= now` upper bound to prevent race conditions, the unit test for this fix is entirely missing from `test_digest.py`. The module docstring mentions that a test for FIX-2 was added, but the file only contains tests for FIX-1 and FIX-3. Please add a dedicated test to verify that opportunities created *after* the `now` snapshot are correctly excluded from the current digest run.

### Previous Findings
**Status:** Resolved

**Findings:**
1. **OOM / CPU Spike Vulnerability (R-004 Violation):** Users with zero matches do not get an `AlertLog` record, meaning their `last_sent` cursor never advances. Over time, the window for these users will grow indefinitely, causing the system to load months of opportunities into memory on every run.
   - ✅ **Resolved:** `AlertLog` (with empty `opportunity_ids`) is now written for ALL users on every digest run — including zero-match users — to advance the cursor. When the opportunity window is entirely empty, cursor-advance logs are still written for every active user before returning.

2. **Race Condition (Duplicate Emails):** The query `Opportunity.created_at > min_start` lacks an upper bound (`<= now`). Opportunities inserted *during* the execution window will be included in the current digest and then included *again* on the next run because the next run's `min_start` is set to the `now` of the previous run.
   - ✅ **Resolved:** Added `Opportunity.created_at <= now` filter to the opportunity query. The `now` snapshot is taken once at the start of `_process_digest` and used as both the upper bound for the DB query and the `sent_at` timestamp in all `AlertLog` records written during this run.

3. **Dual-Write Vulnerability:** The `send_email.delay` Celery tasks are dispatched *before* the `session.commit()`. If the DB commit fails (e.g., due to a constraint or network drop), the emails are still queued and sent, but no `AlertLog` is recorded. The next run will duplicate the digest. Emails must be dispatched after a successful commit.
   - ✅ **Resolved:** All email payloads are now collected during the per-user loop, then `session.commit()` is called, and only after a successful commit are emails dispatched via `asyncio.to_thread(send_email.delay, ...)`.

## Review Follow-ups (AI)
- [x] [AI-Review] FIX-1: Write empty AlertLog for all users on every digest run to advance cursor
- [x] [AI-Review] FIX-2: Add `Opportunity.created_at <= now` upper bound to prevent race condition
- [x] [AI-Review] FIX-3: Collect email payloads first, commit, then dispatch (prevent dual-write)
- [x] [AI-Review] FIX-2-TEST: Add unit test verifying opportunities created after ``now`` are excluded from digest opportunity query (`test_digest_opportunity_query_has_upper_bound`)

## Known Deviations

### Detected by `3-code-review` at 2026-04-19T10:11:45Z

- The acceptance criteria require calculating the digest window from the 'last sent digest' and dictate that 'Users with zero matches receive no email', but fail to specify how the 'last checked' cursor is advanced for zero-match users. This omission directly led to the implementation skipping `alert_log` records for these users, creating a critical infinite-lookback bug that will cause OOM and DB spikes over time. _(type: `ACCEPTANCE_GAP`; severity: `blocking`)_

### Resolution at 2026-04-19
The AC gap was addressed by always writing an `AlertLog` (with empty `opportunity_ids`) for every user on every digest run, including zero-match users. This is consistent with the spirit of AC6 ("no duplicates, no gaps") extended to cursor management.

## Dev Agent Record

### Implementation Plan
All 4 original tasks were already implemented. This session addressed 3 critical bugs identified in the first Senior Developer Review and 1 missing test identified in the second Senior Developer Review:

1. **FIX-1 (OOM/Cursor):** Added `AlertLog` writes for zero-match users in both the "no opportunities" early-return path and the per-user matching loop.
2. **FIX-2 (Race Condition):** Added `Opportunity.created_at <= now` upper bound; the `now` snapshot is taken once at function entry.
3. **FIX-3 (Dual-Write):** Refactored loop to collect `email_payloads` list, then commit, then dispatch — ensuring atomicity between DB persistence and email queuing.
4. **FIX-2-TEST:** Added `test_digest_opportunity_query_has_upper_bound` — inspects the compiled SQLAlchemy SELECT statement for the opportunity fetch and asserts `created_at <=` is present in the WHERE clause.

Additional refactors (from prior session):
- Moved `import asyncio` from inline (inside function body) to module-level imports.
- Replaced `datetime(9999, 12, 31, tzinfo=UTC)` sort sentinel with `float("inf")` via `deadline.timestamp() if deadline else float("inf")` — avoids mock interaction issues in tests.
- Removed now-unused `TYPE_CHECKING` import guard (was only guarding `AsyncSession` which was removed).

### Completion Notes
- ✅ All 4 review findings resolved (FIX-1, FIX-2, FIX-3 + FIX-2-TEST)
- ✅ Resolved review finding [High]: Missing unit test for FIX-2 race condition upper bound
- ✅ 7 unit tests pass (4 regression + 3 new: FIX-1, FIX-2, FIX-3)
- ✅ Full notification service test suite: 267 passed, 0 failures
- ✅ Linting: ruff clean

## File List
- `eusolicit-app/services/notification/src/notification/workers/tasks/digest.py` — modified (3 bug fixes + refactors)
- `eusolicit-app/services/notification/tests/worker/test_digest.py` — modified (updated zero-match test + 2 new tests)

## Change Log
- 2026-04-19: Initial implementation (Tasks 1–4) — daily/weekly digest Celery tasks
- 2026-04-19: Code review findings resolved — FIX-1 cursor advance, FIX-2 upper bound, FIX-3 commit-before-dispatch; updated and extended unit tests (6 tests total)
- 2026-04-19: Second code review finding resolved — FIX-2-TEST: added `test_digest_opportunity_query_has_upper_bound` to verify SQL `created_at <= now` upper bound; 7 unit tests total, all pass

## Status
review
