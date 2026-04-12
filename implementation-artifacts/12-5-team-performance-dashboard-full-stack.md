# Story 12.5: Team Performance Dashboard (Full Stack)

Status: ready-for-dev

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

- [ ] Task 1: Pydantic Team schemas — append to `schemas/analytics.py`
  - [ ] Define `TeamLeaderboardItem`, `UserPerformanceResponse`, `TeamActivityItem`.
- [ ] Task 2: Team performance service — `services/analytics_team_service.py`
  - [ ] Implement `get_team_leaderboard`, `get_user_performance`, `get_team_activity`.
  - [ ] Ensure `company_id` scoping and user-level permission checks.
- [ ] Task 3: Team performance router — `api/v1/analytics_team.py`
- [ ] Task 4: Register router in `main.py`.
- [ ] Task 5: Backend unit tests — `tests/api/test_analytics_team.py`.

### Frontend

- [ ] Task 6: Update `lib/api/analytics.ts` with Team interfaces and fetch functions.
- [ ] Task 7: Create TanStack Query hooks in `lib/queries/use-team-analytics.ts`.
- [ ] Task 8: Add i18n keys to `en.json` and `bg.json`.
- [ ] Task 9: Create components: `TeamFilters.tsx`, `UserPerformanceCards.tsx`, `TeamLeaderboardTable.tsx`, `TeamActivityChart.tsx`.
- [ ] Task 10: Create dashboard orchestrator `TeamPerformanceDashboard.tsx`.
- [ ] Task 11: Create page shell `page.tsx`.
- [ ] Task 12: Frontend unit tests `__tests__/team-performance-s12-5.test.ts`.

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

