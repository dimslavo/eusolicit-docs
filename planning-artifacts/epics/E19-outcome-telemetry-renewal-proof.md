# E19: Outcome Telemetry & Renewal Proof Brief

**Sprint**: 17–18 | **Points**: 21 | **Dependencies**: E12 (Analytics), E14 (workspace), E07 (WeasyPrint) | **Milestone**: Renewal engine
**Source:** PRD v1.1 §6 FR9 + §8 US11; architecture-evaluation §2 Change-6 (stays in client-api per Rule of Three)

## Goal

Ship the renewal engine: every workspace gains an Outcome Dashboard quantifying platform value (€/hours saved, win-rate uplift, content reuse, platform-attributed bid conversion) plus a monthly auto-generated Outcome Brief PDF for download. Without instrumentation, retention is anecdotal — Loopio's commentary "you'll be surprised how many customers churn" applies precisely here. Materialized views extend the existing FR7.2 pattern (concurrent refresh to avoid read-locks). PDF generation reuses WeasyPrint pipeline from Epic 7. Onboarding milestone tracking ("First Bid in 14 Days") rolls in here because S20's Epic-20 NPS work depends on it.

## Acceptance Criteria

- [ ] Every opportunity supports a `platform_attributed` boolean tag (default false; set true when discovered via platform feed); persists through bid lifecycle; queryable by analytics
- [ ] Bid outcomes capturable per opportunity: `won` / `lost` / `withdrawn` with optional `evaluator_score`, `contract_value_eur`, `effort_hours`
- [ ] Workspace-level Outcome Dashboard widget renders: bids tracked, submitted, won, win rate (rolling 12-month), platform-attributed bid conversion, content-reuse rate, AI-summary usage trend, hours-saved estimate
- [ ] Monthly auto-generated Outcome Brief PDF available for download per workspace (1st of each month, prior month covered)
- [ ] Outcome metrics available via authenticated API at `api.eusolicit.com/v1/workspaces/:id/outcome` (Pro+ tier-gated for QBR embedding in customer's own decks)
- [ ] Hourly rate and hours-saved-per-bid baseline workspace-configurable; defaults to research-derived values (€60K loaded cost, 19% search-time savings → ~€11.4K baseline savings)
- [ ] Onboarding milestone tracker visible on dashboard: workspace_created / content_uploaded / first_opp_reviewed / first_ai_summary / crm_connected / first_bid_decision
- [ ] CSM is alerted via Slack/email on milestone stalls >7 days
- [ ] Materialized views refresh concurrently per FR7.2 pattern (no read-locks)
- [ ] Workspace-scoped negative tests on all endpoints
- [ ] Cross-workspace analytics work for `tenant_admin` role (cross-workspace bypass per Epic 14)

## Stories

### S19.00: Platform-Attributed Flag + Bid-Outcome Capture UI + Materialized Views
**Points**: 8 | **Type**: backend + frontend

**Database:**
- `client.opportunities` add column `platform_attributed BOOLEAN DEFAULT FALSE` (Alembic migration)
- `client.bid_outcomes` add columns `evaluator_score INTEGER`, `contract_value_eur INTEGER`
- New materialized views in `client` schema:
  - `mv_workspace_outcome_stats(workspace_id, month, bids_tracked, bids_submitted, bids_won, win_rate, platform_attributed_count, avg_value)` — refreshed daily
  - `mv_workspace_content_reuse_stats(workspace_id, content_block_id, usage_count, win_rate_when_used)` — refreshed daily
  - `mv_workspace_onboarding_milestones(workspace_id, milestone, completed_at)` — refreshed hourly
- Refresh runs `CONCURRENTLY` per FR7.2 pattern; index on `(workspace_id, month)` for outcome stats

**Backend:**
- `POST /api/v1/workspaces/:id/opportunities/:oid/outcome` (existing endpoint extended) — accepts won/lost/withdrawn + optional evaluator_score, contract_value_eur, effort_hours; writes to `bid_outcomes`; emits `bid.outcome_recorded` event
- Platform-attributed flag set automatically when opportunity created via platform feed (vs. external upload); default false otherwise
- Lessons-Learned Agent (existing E11 integration) consumes outcome events as-is

**Frontend:**
- Outcome capture form on opportunity detail page (post-submission): radio buttons + structured fields
- Form uses `useZodForm(schema)` + `<FormField>` (existing pattern)

**Tests:**
- Workspace-scoped negative test (W1's outcome endpoint cannot read W2's data)
- Materialized-view refresh under concurrent inserts (no read-lock)
- ATDD source-inspection: form uses `useZodForm` and `<FormField>` (NOT direct `useForm`)

---

### S19.01: Outcome Dashboard Frontend Widgets + Workspace-Config Overrides
**Points**: 8 | **Type**: frontend + backend

**Backend:**
- New `client.workspace_outcome_config` table: `workspace_id PK`, `hourly_rate_eur INTEGER nullable`, `hours_saved_per_bid INTEGER nullable`
- New `client.tenant_outcome_config` table: `company_id PK`, default values per tenant
- `GET /api/v1/workspaces/:id/outcome/dashboard` — returns aggregated metrics from materialized views with hourly-rate / hours-saved baseline applied
- `PATCH /api/v1/workspaces/:id/outcome/config` — admin or bid-manager role; updates per-workspace overrides
- Defaults: €60K loaded cost / 19% search-time savings (configurable as env vars per tenant)

**Frontend:**
- Outcome Dashboard widget composed of: KPI cards (bids tracked, submitted, won, win rate), trend chart (Recharts), platform-attributed funnel, content-reuse leaderboard
- Configuration drawer: workspace admin can override hourly rate / hours-saved baseline; defaults visible
- Renders inside existing `<AppShell>` workspace home page
- TanStack Query keys include `workspace_id` (cache isolation per Epic 14)

**Tests:**
- Workspace-scoped read isolation
- Configuration override persistence
- ATDD source-inspection: dashboard wraps queries in `<QueryGuard>` (existing pattern)
- i18n parity (BG/EN)

---

### S19.02: Monthly Outcome Brief PDF Generation + Onboarding Milestone Tracker + CSM Stall Alerts
**Points**: 5 | **Type**: backend + integration

**Outcome Brief PDF (Celery Beat task, monthly 1st of month):**
- Iterates over all active workspaces, generates per-workspace PDF via WeasyPrint pipeline (Epic 7 pattern; project-context Rule: ThreadPoolExecutor mandatory for CPU-bound)
- Content: top metrics for prior month, trend charts (rendered to PNG embedded in PDF), ROI calculation using configurable assumptions, platform-attributed bid examples
- Stored in S3 with workspace-scoped path; signed URL (24-hour TTL) returned to frontend on demand
- `GET /api/v1/workspaces/:id/outcome/brief?month=YYYY-MM` returns signed URL

**Onboarding milestones table:**
- `client.onboarding_milestones` (workspace_id, milestone, completed_at) seeded with 6 milestone rows on workspace creation: workspace_created (auto-completed), content_uploaded, first_opp_reviewed, first_ai_summary, crm_connected, first_bid_decision
- Milestone events fire from existing flows: workspace creation (Epic 14), content upload (Epic 7 docs), first opportunity view (Epic 6), first AI summary (Epic 6), CRM connection (Epic 17), first bid decision (Epic 10)

**CSM stall alerts (Celery Beat task, daily):**
- Query `onboarding_milestones` for rows where `completed_at IS NULL` AND workspace_age > 7 days
- Per-workspace digest event published to `eu-solicit:notifications` with payload `{workspace_id, stalled_milestones: [...]}`
- Notification service consumes → emails CSM contact (configured per tenant) AND posts to Slack #onboarding (config via Epic 16 webhook)

**Tests:**
- PDF generation under load (10 concurrent workspaces, no event-loop blocking — verified via timing)
- Stall-alert trigger test (workspace stuck on milestone 3 → alert fires after 7 days)
- WeasyPrint runs in `run_in_executor` (source-inspection ATDD per Epic 7)
- Workspace-scoped: W1 stall events do not leak into W2 reports
