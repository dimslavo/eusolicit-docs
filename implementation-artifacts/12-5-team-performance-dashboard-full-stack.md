# Story 12.5: Team Performance Dashboard (Full Stack)

Status: done

## Story

As a **company admin or team member on the EU Solicit platform**,
I want **a team performance dashboard at `/analytics/team`**,
so that **I can track per-user metrics (bids submitted, win rate, average preparation time, proposals generated) via a sortable leaderboard and individual user drill-down cards, and view team-level activity trends over time**.

## Acceptance Criteria

### Backend — Team Performance Endpoints

1. **AC1** — `GET /api/v1/analytics/team/leaderboard` returns per-user performance metrics for the authenticated user's company. Response body is a paginated list: `{"items": [...], "total": N, "page": P, "page_size": S}`. Each item contains:
   - `user_id` (UUID)
   - `user_name` (string) — fetched from `users` table via JOIN or subquery
   - `bids_submitted` (integer) — `SUM(mv_team_performance.bids_submitted)`
   - `win_count` (integer) — `SUM(mv_team_performance.win_count)`
   - `win_rate` (Decimal string) — `(SUM(win_count) / SUM(bids_submitted) * 100)` computed in Python
   - `avg_preparation_hours` (Decimal string) — `SUM(total_preparation_hours) / SUM(bids_submitted)`
   - `proposals_generated` (integer) — `SUM(mv_team_performance.proposals_generated)`

   Supports `date_from` and `date_to` filters on `mv_team_performance.month`. Supports `sort_by` (choices: `bids_submitted`, `win_rate`, `avg_preparation_hours`, `proposals_generated`, `user_name`; default: `win_rate`) and `sort_dir` (`asc` | `desc`; default: `desc`).

2. **AC2** — `GET /api/v1/analytics/team/user/{user_id}` returns individual user detail metrics aggregated over the selected date range. Returns a single object with the same metric fields as AC1.

3. **AC3** — `GET /api/v1/analytics/team/activity` returns team-level bids submitted per month for the activity bar chart. Response body is `{"items": [...]}` where each item contains `month` (ISO date) and `bids_submitted` (integer sum across all users in company).

4. **AC4 — Access Control**:
   - Company admins can query the leaderboard and any `user_id` in their company.
   - Regular users can query `/leaderboard` but it returns **only their own row** (or they are restricted from the leaderboard entirely and redirected to their own detail — implementation should allow regular users to see the leaderboard but perhaps with limited data/anonymized if configured; for MVP, **regular users see only their own metrics** in both endpoints).
   - If a regular user requests a `user_id` that is not their own, return 403 Forbidden.

5. **AC5 — Tenant Isolation**: Every query MUST filter by `company_id = current_user.company_id` as the primary clause (E12-R-001).

6. **AC6 — Cache & Performance**: Endpoints include `Cache-Control: public, max-age=1800`. Materialized view `mv_team_performance` is used for all queries.

7. **AC7 — Unit Tests**: Cover happy path for all 3 endpoints; admin vs regular user permissions; cross-tenant isolation; date filtering; sort/pagination logic; empty result handling.

---

### Frontend — Team Performance Dashboard

8. **AC8** — A new page is created at `apps/client/app/[locale]/(protected)/analytics/team/page.tsx`. It is a server component shell that renders `<TeamPerformanceDashboard />`.

9. **AC9 — Filter Panel**: Renders at the top with `data-testid="team-filters"`. Includes:
   - **Date From / To** inputs.
   - **User Selector** (dropdown) — visible to admins only, allows drilling down into a specific user's metrics. Default is "All Team Members".
   - **Apply / Clear** buttons.

10. **AC10 — User Performance Cards**: When a specific user is selected (or for the current user if not admin), render KPI cards using `data-testid="team-user-cards"`. Cards: Bids Submitted, Win Rate, Avg Prep Time, Proposals Generated.

11. **AC11 — Leaderboard Table**: Renders `data-testid="team-leaderboard-section"`. Sortable columns for all metrics. Rows link to user detail view or update the dashboard filter.

12. **AC12 — Activity Bar Chart**: Renders `data-testid="team-activity-chart-section"`. Recharts `<BarChart>` showing `bids_submitted` per month.

13. **AC13 — i18n**: Add `analytics.team` keys to `en.json` and `bg.json`.

14. **AC14 — Unit Tests**: `apps/client/__tests__/team-performance-s12-5.test.ts` verifies structural existence of components, testids, API functions, and i18n keys.

---

## Tasks / Subtasks

### Backend

- [x] Task 1: Pydantic Team schemas — append to `schemas/analytics.py`
  - [x] Define `TeamLeaderboardItem`, `UserPerformanceResponse`, `TeamActivityItem`.
- [x] Task 2: Team performance service — `services/analytics_team_service.py`
  - [x] Implement `get_team_leaderboard`, `get_user_performance`, `get_team_activity`.
  - [x] Ensure `company_id` scoping and user-level permission checks.
- [x] Task 3: Team performance router — `api/v1/analytics_team.py`
- [x] Task 4: Register router in `main.py`.
- [x] Task 5: Backend unit tests — `tests/api/test_analytics_team.py`.

### Frontend

- [x] Task 6: Update `lib/api/analytics.ts` with Team interfaces and fetch functions.
- [x] Task 7: Create TanStack Query hooks in `lib/queries/use-team-analytics.ts`.
- [x] Task 8: Add i18n keys to `en.json` and `bg.json`.
- [x] Task 9: Create components: `TeamFilters.tsx`, `UserPerformanceCards.tsx`, `TeamLeaderboardTable.tsx`, `TeamActivityChart.tsx`.
- [x] Task 10: Create dashboard orchestrator `TeamPerformanceDashboard.tsx`.
- [x] Task 11: Create page shell `page.tsx`.
- [x] Task 12: Frontend unit tests `__tests__/team-performance-s12-5.test.ts`.

---

## Dev Notes

### Architecture & Data
- Materialized view: `client.mv_team_performance`.
- Formula for `win_rate`: `(win_count / bids_submitted * 100)` if `bids_submitted > 0` else `0`.
- Formula for `avg_prep_time`: `total_preparation_hours / bids_submitted` if `bids_submitted > 0` else `0`.
- User names must be joined from the `users` table using `user_id`.
- Page route: `/analytics/team`.

### Security & Access Control (E12-R-001, E12-R-002)
- **Cross-tenant isolation (E12-R-001)**: Every SQL query MUST include `WHERE company_id = current_user.company_id` as the primary filter.
- **User-level isolation**: Regular users (`role != 'admin'`) MUST only see their own data. The API service layer should automatically inject `WHERE user_id = current_user.id` for non-admin requests.
- **Tier Gating (E12-R-002)**: Ensure `require_paid_tier` dependency is used. Verify that it is no longer a stub in the core library.

### Performance & Testing (E12-PERF)
- **Latency target**: Analytics reads must aim for p95 < 500ms. Use `EXPLAIN ANALYZE` if queries are slow.
- **Test Seeding**: Use the `migration_role` pattern for seeding `mv_team_performance` in integration tests, as it is a read-only materialized view for the application role.
- **Edge Cases**: Ensure zero-bid scenarios return 0% win rate and 0 hours, not division errors.

---

## Dev Agent Record

### Implementation Plan

Post-review verification pass: validated that all 4 Senior Developer Review findings (3 must-fix + 1 recommended) were addressed in a prior implementation cycle. Confirmed through code inspection and test execution.

### Completion Notes

**Review Finding Resolution (2026-04-13):**
- ✅ Resolved review finding [HIGH]: Pagination mismatch — `useTeamLeaderboard` hook now accepts `pageSize` parameter (default 10) and forwards it through `fetchTeamLeaderboard` to the API. Dashboard passes `pageSize = 10` consistently.
- ✅ Resolved review finding [HIGH]: Input validation — `sort_by` typed as `Literal["bids_submitted", "win_rate", "avg_preparation_hours", "proposals_generated", "user_name"]` and `sort_dir` as `Literal["asc", "desc"]` in `analytics_team.py`. Tests `test_leaderboard_invalid_sort_by_returns_422` and `test_leaderboard_invalid_sort_dir_returns_422` confirm 422 on invalid values.
- ✅ Resolved review finding [HIGH]: Code quality — `HTTPException` imported at module top (line 21); no leftover deliberation comments in the codebase.
- ✅ Resolved review finding [Deferrable]: Backend test coverage — Added 8 additional tests covering cross-tenant isolation (2 tests), date filtering (2 tests), sort/pagination logic (1 test), invalid sort_by/sort_dir validation (2 tests), and empty result handling (3 tests).

**Test Summary:**
- Backend: 16/16 tests PASS in `test_analytics_team.py`
- Frontend ATDD: 73/73 tests PASS in `team-performance-s12-5.test.ts`
- Regression: 0 regressions introduced (8 pre-existing failures in other stories confirmed unrelated)

**Definition of Done:**
- All 12 tasks/subtasks marked complete [x]
- All 14 acceptance criteria satisfied (AC1–AC14)
- Backend: 3 endpoints operational with tenant isolation, user isolation, pagination, sorting, caching
- Frontend: 5 components, page shell, TanStack Query hooks, i18n (en+bg), data-testids
- Code review findings: all 4 items addressed and verified

---

## File List

### Backend
- `services/client-api/src/client_api/schemas/analytics.py` — Team schemas (TeamLeaderboardItem, UserPerformanceResponse, TeamActivityItem)
- `services/client-api/src/client_api/services/analytics_team_service.py` — Service layer (get_team_leaderboard, get_user_performance, get_team_activity)
- `services/client-api/src/client_api/api/v1/analytics_team.py` — Router with Literal-typed sort params, Cache-Control headers
- `services/client-api/src/client_api/main.py` — Router registration
- `services/client-api/tests/api/test_analytics_team.py` — 16 backend unit tests

### Frontend
- `frontend/apps/client/lib/api/analytics.ts` — Team interfaces and fetch functions
- `frontend/apps/client/lib/queries/use-team-analytics.ts` — TanStack Query hooks (useTeamLeaderboard with pageSize, useUserPerformance, useTeamActivity)
- `frontend/apps/client/messages/en.json` — analytics.team i18n keys (EN)
- `frontend/apps/client/messages/bg.json` — analytics.team i18n keys (BG)
- `frontend/apps/client/app/[locale]/(protected)/analytics/team/page.tsx` — Server component page shell
- `frontend/apps/client/app/[locale]/(protected)/analytics/team/components/TeamPerformanceDashboard.tsx` — Dashboard orchestrator
- `frontend/apps/client/app/[locale]/(protected)/analytics/team/components/TeamFilters.tsx` — Filter panel (data-testid="team-filters")
- `frontend/apps/client/app/[locale]/(protected)/analytics/team/components/UserPerformanceCards.tsx` — KPI cards (data-testid="team-user-cards")
- `frontend/apps/client/app/[locale]/(protected)/analytics/team/components/TeamLeaderboardTable.tsx` — Sortable leaderboard (data-testid="team-leaderboard-section")
- `frontend/apps/client/app/[locale]/(protected)/analytics/team/components/TeamActivityChart.tsx` — Recharts bar chart (data-testid="team-activity-chart-section")
- `frontend/apps/client/__tests__/team-performance-s12-5.test.ts` — 73 frontend ATDD tests

---

## Change Log

- **2026-04-13**: Post-review verification pass — confirmed all 4 Senior Developer Review findings addressed. All tests passing (16 backend + 73 frontend ATDD). Status → review.

---

## Senior Developer Review

**Review Date**: 2026-04-12
**Verdict**: CHANGES REQUESTED

### Must Fix

1. **BUG — Pagination mismatch (HIGH)**
   - `TeamPerformanceDashboard.tsx:38` sets `pageSize = 10` but `useTeamLeaderboard` hook does not accept or forward `pageSize` to `fetchTeamLeaderboard` (which defaults to 20). The API returns 20 items per page while the UI paginates assuming 10-item pages. Pagination is visibly broken for teams with >10 members.
   - **Fix**: Add `pageSize` parameter to `useTeamLeaderboard` and pass it through to `fetchTeamLeaderboard`. Ensure dashboard and API agree on the same value.

2. **Input validation — `sort_by` / `sort_dir` accept arbitrary strings**
   - `api/v1/analytics_team.py:47-48`: Both parameters are typed as `str`. AC1 specifies allowed choices. Invalid values silently fall back instead of returning 422.
   - **Fix**: Type `sort_by` as `Literal["bids_submitted", "win_rate", "avg_preparation_hours", "proposals_generated", "user_name"]` and `sort_dir` as `Literal["asc", "desc"]`.

3. **Code quality cleanup in `analytics_team.py`**
   - Line 112: `from fastapi import HTTPException` is imported inside the function body — move to module top.
   - Lines 104-111: Leftover deliberation comments ("But wait, AC2 says...", "Actually, let's just...") should be removed.

### Recommended (Deferrable)

4. **Backend test coverage gaps (AC7)**
   - Tests cover happy path and admin/regular-user permissions but are missing: cross-tenant isolation test, date filtering test, sort/pagination logic test, and empty result handling test. All four are explicitly required by AC7.

### Passed Checks

- ✅ Tenant isolation (E12-R-001): `company_id` scoping on all queries
- ✅ User isolation: non-admins restricted to own data on leaderboard + user detail
- ✅ Zero-bids safety: `CASE WHEN` prevents division by zero
- ✅ Tier gating: `require_paid_tier` dependency used on all endpoints
- ✅ Cache-Control headers set correctly
- ✅ i18n complete for en + bg
- ✅ All required components, testids, and page shell in place
- ✅ Recharts activity bar chart correctly implemented
- ✅ Router registered in main.py

---

### Re-Review (2026-04-13)

**Verdict**: APPROVED

All 4 prior findings verified resolved. Full adversarial re-review performed (Blind Hunter, Edge Case Hunter, Acceptance Auditor).

**Prior Findings — Verified Fixed:**
- [x] Pagination mismatch: `useTeamLeaderboard` now accepts `pageSize`, dashboard passes 10 consistently
- [x] Input validation: `sort_by`/`sort_dir` use `Literal` types, 422 tests confirm rejection of invalid values
- [x] Code quality: `HTTPException` at module top, no leftover comments
- [x] Test coverage: 16 backend tests covering cross-tenant, date filtering, sort/pagination, validation, empty results

**New Findings:** None.

**Test Results (re-verified):**
- Backend: 16/16 PASS
- Frontend ATDD: 73/73 PASS

**Architecture Alignment:**
- ✅ E12-R-001 (Tenant Isolation): company_id primary clause on all queries
- ✅ E12-R-002 (Tier Gating): require_paid_tier on all endpoints
- ✅ Materialized view mv_team_performance used exclusively
- ✅ User isolation at API (403) and service (WHERE) layers
- ✅ All 14 acceptance criteria satisfied (AC1–AC14)

