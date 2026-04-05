# E12: Analytics, Reporting & Admin Platform

**Sprint**: 13-14 | **Points**: 55 | **Dependencies**: E05, E06, E07, E08 | **Milestone**: MVP Launch

## Goal

Deliver the analytics, reporting, admin, and enterprise API capabilities that complete the EU Solicit platform for launch. Users gain actionable business intelligence through market, ROI, team, competitor, and pipeline dashboards powered by PostgreSQL materialized views refreshed on schedule by Celery Beat. Company admins configure scheduled PDF/DOCX reports. Internal operators manage tenants, crawlers, white-label branding, and audit logs through a VPN-restricted admin platform. Enterprise clients integrate via a rate-limited, API-key-authenticated OpenAPI endpoint. The epic concludes with performance optimization, load testing, a security audit, and a guided onboarding flow -- bringing the platform to launch readiness.

## Acceptance Criteria

- [ ] Market intelligence dashboard displays procurement volumes, average contract values, top contracting authorities, and seasonal trends with interactive Recharts charts and filterable tables
- [ ] ROI tracker aggregates bid_preparation_logs and bid_outcomes into per-bid and aggregate investment-return views
- [ ] Team performance dashboard shows per-user bids submitted, win rate, avg preparation time, and proposals generated with leaderboard and individual views
- [ ] Competitor intelligence dashboard (Professional+ tier gate) surfaces competitor bidding patterns, win rates, and pricing benchmarks
- [ ] Pipeline forecasting dashboard (Professional+ tier gate) presents Pipeline Forecasting Agent predictions with confidence scores
- [ ] Usage dashboard displays current-period consumption vs tier limits for AI summaries, proposal drafts, and compliance checks with visual meters
- [ ] PostgreSQL materialized views exist for each analytics domain; Celery Beat refreshes market/ROI/team/competitor daily and usage hourly
- [ ] Scheduled reports (weekly/monthly) are generated as PDF (reportlab) and DOCX (python-docx) and emailed via SendGrid
- [ ] On-demand reports are generated asynchronously with download link returned to the user
- [ ] Admin API is VPN/IP-restricted and provides tenant management, crawler management, white-label configuration, audit log viewing, and platform analytics
- [ ] Enterprise API is routed through api.eusolicit.com/v1/*, authenticated via API key, rate-limited, and auto-documented via Swagger UI or Redoc
- [ ] Load testing (Locust or k6) passes against staging with acceptable response times under target concurrency
- [ ] Security audit completes OWASP checklist, network policy verification, and secret rotation
- [ ] First-run onboarding wizard guides new users through company profile, first search, first opportunity, and a feature tour

## Stories

### S12.01: Analytics Materialized Views & Refresh Infrastructure

**Points**: 4 | **Type**: backend

Define PostgreSQL materialized views for each analytics domain: market intelligence (procurement volumes, avg contract values, top authorities, seasonal trends), ROI tracking (bid cost vs outcome aggregation), team performance (per-user metrics), competitor intelligence (bidding patterns, win rates), and usage consumption. Create corresponding Celery Beat tasks in the Notification Service: daily refresh for market/ROI/team/competitor views, hourly refresh for usage view. Include a `REFRESH MATERIALIZED VIEW CONCURRENTLY` strategy so dashboard reads are not blocked during refresh. Add DB migration scripts for all views and indexes on the materialized view columns used for filtering.

**Acceptance Criteria**:
- [ ] Materialized views created for market, ROI, team, competitor, and usage analytics
- [ ] Celery Beat schedule configured: daily for market/ROI/team/competitor, hourly for usage
- [ ] `REFRESH MATERIALIZED VIEW CONCURRENTLY` used with unique indexes to avoid read locks
- [ ] DB migration scripts included and reversible
- [ ] Manual refresh management command available for ad-hoc use

---

### S12.02: Market Intelligence Dashboard API

**Points**: 3 | **Type**: backend

Build Client API endpoints for the market intelligence dashboard. Endpoints query the market materialized view and return: procurement volume by sector, average contract values by sector, most active contracting authorities (ranked), and seasonal trend data (monthly aggregates). Support query parameters for date range, sector filter, and country filter. All paid tiers can access this dashboard.

**Acceptance Criteria**:
- [ ] `GET /analytics/market/volume` returns procurement volume by sector with date range and country filters
- [ ] `GET /analytics/market/values` returns average contract values by sector
- [ ] `GET /analytics/market/authorities` returns top contracting authorities ranked by activity
- [ ] `GET /analytics/market/trends` returns monthly aggregate trend data
- [ ] Responses are paginated where applicable and include cache headers
- [ ] Unit tests cover filter combinations and empty-result edge cases

---

### S12.03: Market Intelligence Dashboard Frontend

**Points**: 3 | **Type**: frontend

Build the market intelligence dashboard page. Bar chart (Recharts) for procurement volume by sector. Line chart for trends over time. Sortable, filterable table for top contracting authorities. Date range picker and sector/country filter controls. Loading skeletons while data fetches. Responsive layout -- charts stack vertically on mobile.

**Acceptance Criteria**:
- [ ] Bar chart renders procurement volume grouped by sector
- [ ] Line chart renders monthly trend data with hover tooltips
- [ ] Table displays top contracting authorities with sorting and pagination
- [ ] Date range, sector, and country filters update all visualizations
- [ ] Loading skeletons shown during data fetch
- [ ] Layout is responsive (charts stack on mobile)

---

### S12.04: ROI Tracker Dashboard (Full Stack)

**Points**: 4 | **Type**: fullstack

Build the ROI tracker API and frontend. API endpoints query the ROI materialized view: total invested (sum of bid_preparation_logs cost), total won (sum of bid_outcomes contract value), ROI percentage, per-bid breakdown, and trend over time. Frontend displays summary cards (total invested, total won, ROI %), a per-bid table with investment and outcome columns, and a trend line chart. Date range filter and team member filter available.

**Acceptance Criteria**:
- [ ] `GET /analytics/roi/summary` returns total invested, total won, ROI %
- [ ] `GET /analytics/roi/bids` returns per-bid investment vs outcome with pagination
- [ ] `GET /analytics/roi/trends` returns ROI trend data over time
- [ ] Frontend summary cards display aggregate metrics
- [ ] Per-bid table is sortable by investment, outcome, and ROI
- [ ] Trend chart renders with date range filter support

---

### S12.05: Team Performance Dashboard (Full Stack)

**Points**: 3 | **Type**: fullstack

Build the team performance API and frontend. API endpoints query the team materialized view: per-user bids submitted, win rate, average preparation time, proposals generated. Supports company-level aggregation and individual user drill-down. Frontend renders user cards with key metrics, a sortable leaderboard table, and an activity bar chart. Company admin can view all team members; regular users see only their own metrics.

**Acceptance Criteria**:
- [ ] `GET /analytics/team/leaderboard` returns per-user metrics sorted by win rate (default)
- [ ] `GET /analytics/team/user/{user_id}` returns individual user detail metrics
- [ ] Company admins see all team members; regular users see only themselves
- [ ] Frontend leaderboard table is sortable by any metric column
- [ ] User cards display bids submitted, win rate, avg prep time, proposals generated
- [ ] Activity chart shows team-level bids submitted over time

---

### S12.06: Competitor Intelligence Dashboard (Full Stack, Professional+ Tier)

**Points**: 4 | **Type**: fullstack

Build the competitor intelligence API and frontend, gated to Professional+ tier. API endpoints query the competitor materialized view: competitor profiles (name, bid count, estimated win rate, sectors active in), bidding pattern data (frequency, timing), and pricing benchmarks. Frontend renders competitor profile cards, a comparison table, and a pattern chart (e.g., bidding frequency over time). Tier gate middleware returns 403 with upgrade prompt for lower tiers.

**Acceptance Criteria**:
- [ ] `GET /analytics/competitors/profiles` returns paginated competitor profiles
- [ ] `GET /analytics/competitors/{competitor_id}/patterns` returns bidding pattern data
- [ ] `GET /analytics/competitors/benchmarks` returns pricing benchmark aggregations
- [ ] Tier gate returns 403 with upgrade message for non-Professional+ users
- [ ] Frontend competitor cards show name, bid count, estimated win rate, active sectors
- [ ] Comparison table allows selecting 2-4 competitors side by side
- [ ] Pattern chart renders bidding frequency trends

---

### S12.07: Pipeline Forecasting Dashboard (Full Stack, Professional+ Tier)

**Points**: 3 | **Type**: fullstack

Build the pipeline forecasting dashboard, gated to Professional+ tier. API endpoint returns Pipeline Forecasting Agent predictions: predicted upcoming opportunities with title, sector, estimated value range, predicted publication date, and confidence score. Frontend renders a calendar/timeline view of predicted opportunities with color-coded confidence indicators (high/medium/low). Filterable by sector, value range, and confidence threshold.

**Acceptance Criteria**:
- [ ] `GET /analytics/pipeline/forecast` returns predicted opportunities with confidence scores
- [ ] Tier gate enforced (Professional+ only), 403 with upgrade prompt otherwise
- [ ] Frontend timeline/calendar view plots predicted opportunities on a time axis
- [ ] Confidence indicators use color coding: green (high), amber (medium), red (low)
- [ ] Sector, value range, and confidence threshold filters functional
- [ ] Empty state displayed when no predictions are available

---

### S12.08: Usage Dashboard (Full Stack)

**Points**: 3 | **Type**: fullstack

Build the usage dashboard API and frontend for all paid tiers. API endpoint queries the usage materialized view and returns current billing period consumption vs tier limits for: AI summaries consumed/remaining, proposal drafts consumed/remaining, compliance checks consumed/remaining. Frontend renders circular progress meters (or radial gauges) for each usage type. When usage exceeds 80% of a limit, show a warning indicator. When at or above limit, show an upgrade CTA linking to billing/subscription page.

**Acceptance Criteria**:
- [ ] `GET /analytics/usage` returns current-period consumption and limits per usage type
- [ ] Response includes billing period start/end dates and per-type consumed/limit/remaining
- [ ] Frontend circular progress meters render for each usage type
- [ ] Warning indicator shown when usage exceeds 80% of limit
- [ ] Upgrade CTA displayed when any usage type is at or above its limit
- [ ] Dashboard available to all paid tiers

---

### S12.09: Report Generation Engine (PDF & DOCX)

**Points**: 4 | **Type**: backend

Build the report generation engine using reportlab (PDF) and python-docx (DOCX). Define report templates for each report type: pipeline summary, bid performance summary, team activity, and custom date range. Each template accepts structured data and produces a formatted document with headers, summary sections, data tables, and charts (rendered as static images for PDF). Reports are generated asynchronously via Celery task. Output stored in S3 with signed download URL (expiry: 24h). Reuse the same document generation stack from the proposal export feature (E07).

**Acceptance Criteria**:
- [ ] PDF generation via reportlab produces formatted reports with headers, tables, and embedded chart images
- [ ] DOCX generation via python-docx produces formatted reports with headers, tables, and embedded chart images
- [ ] Templates defined for pipeline summary, bid performance summary, team activity, and custom date range
- [ ] Reports generated asynchronously via Celery task
- [ ] Output stored in S3 with signed download URL (24h expiry)
- [ ] Shared generation code with proposal export (E07) is extracted and reused

---

### S12.10: Scheduled & On-Demand Report Delivery

**Points**: 3 | **Type**: fullstack

Build the scheduled and on-demand report configuration and delivery. Company admins configure report schedules (weekly on Monday, monthly on 1st) specifying report type, format (PDF/DOCX), and recipient email list. Celery Beat triggers generation at the scheduled time; the completed report is emailed via SendGrid. On-demand reports: user clicks "Generate Report" on an analytics page, selects type and format, and receives an async job -- the download link is shown in-app when ready (polling or WebSocket notification). Frontend: report configuration form in settings, "Generate Report" button on each analytics page, and a reports list page showing past reports with download links.

**Acceptance Criteria**:
- [ ] Company admins can configure weekly/monthly report schedule with type, format, and recipients
- [ ] Celery Beat triggers scheduled report generation and emails result via SendGrid
- [ ] On-demand report generation triggered from analytics pages with async job tracking
- [ ] Download link shown in-app when report is ready
- [ ] Reports list page displays past reports with download links and generation timestamps
- [ ] Email delivery includes report as attachment with summary in body

---

### S12.11: Admin API -- Tenant Management

**Points**: 3 | **Type**: backend

Build the admin API tenant management endpoints, restricted to VPN/IP allowlist. Endpoints: list companies (paginated, searchable by name, filterable by tier), view company detail (subscription details, usage metrics for current period, activity summary -- last login, bids submitted, proposals generated), and manual tier override (for support scenarios, with audit log entry). Admin authentication uses internal SSO or dedicated admin JWT with admin role claim.

**Acceptance Criteria**:
- [ ] `GET /admin/tenants` returns paginated company list with search and tier filter
- [ ] `GET /admin/tenants/{company_id}` returns subscription, usage, and activity summary
- [ ] `POST /admin/tenants/{company_id}/tier-override` updates tier with reason field and creates audit log entry
- [ ] All admin endpoints reject requests from non-allowlisted IPs (403)
- [ ] Admin authentication requires admin role claim in JWT
- [ ] Responses include pagination metadata (total count, page, page_size)

---

### S12.12: Admin API -- Crawler & White-Label Management

**Points**: 3 | **Type**: backend

Build admin endpoints for crawler management and white-label configuration. Crawler management: list crawler_runs with status/results/timestamps (paginated), view/update Celery Beat schedule per crawler type, trigger manual crawl run (returns run_id for status polling), and view individual run status and result counts. White-label configuration: get/set per-tenant settings (custom subdomain, logo URL, brand primary color, brand accent color, email sender domain). Validate subdomain uniqueness and DNS readiness. All endpoints VPN/IP-restricted.

**Acceptance Criteria**:
- [ ] `GET /admin/crawlers/runs` returns paginated crawler run history
- [ ] `GET/PUT /admin/crawlers/schedule/{crawler_type}` reads/updates Celery Beat schedule
- [ ] `POST /admin/crawlers/trigger/{crawler_type}` starts a manual crawl and returns run_id
- [ ] `GET /admin/crawlers/runs/{run_id}` returns run status, result counts, and error details
- [ ] `GET/PUT /admin/tenants/{company_id}/white-label` reads/updates branding settings
- [ ] Subdomain uniqueness validated; DNS readiness check included
- [ ] All endpoints VPN/IP-restricted

---

### S12.13: Admin API -- Audit Log & Platform Analytics

**Points**: 3 | **Type**: backend

Build admin endpoints for audit log viewing and platform-level analytics. Audit log: searchable, filterable endpoint (filter by user, action type, entity type, date range), paginated, with CSV export option (returns streaming CSV response). Platform analytics: signup funnel metrics (registered, trial started, paid conversion counts), tier distribution, usage aggregates across all tenants, churn rate (monthly), and revenue metrics (MRR, growth rate). Return data structured for chart rendering.

**Acceptance Criteria**:
- [ ] `GET /admin/audit-logs` returns paginated, filtered audit log entries
- [ ] Filters supported: user_id, action_type, entity_type, date_from, date_to
- [ ] `GET /admin/audit-logs/export` returns streaming CSV of filtered results
- [ ] `GET /admin/analytics/funnel` returns signup funnel stage counts
- [ ] `GET /admin/analytics/tiers` returns tier distribution counts
- [ ] `GET /admin/analytics/usage` returns aggregate usage metrics across all tenants
- [ ] `GET /admin/analytics/revenue` returns MRR, growth rate, and churn rate

---

### S12.14: Admin Frontend Pages

**Points**: 4 | **Type**: frontend

Build all admin frontend pages. Tenant management page: searchable company table with tier badges, detail drawer (slide-over) showing subscription info, usage meters, activity summary, and tier override form. Crawler management page: run history table with status badges (success/failed/running), schedule configuration form per crawler type, manual trigger button with confirmation dialog, and run detail modal. White-label configuration page: per-tenant form with subdomain input, logo upload (with preview), color pickers for primary and accent colors, email sender domain input, and a live preview panel showing how the branded UI would look. Audit log page: filter bar with user search, action type dropdown, entity type dropdown, and date range picker; paginated results table; CSV export button. Platform analytics page: funnel visualization (horizontal bar or stepped funnel), tier distribution pie chart, revenue line chart (MRR over time), and churn metric summary cards.

**Acceptance Criteria**:
- [ ] Tenant management table is searchable and filterable with detail drawer functional
- [ ] Tier override form submits with reason field and reflects change immediately
- [ ] Crawler run history table shows status badges; manual trigger works with confirmation
- [ ] White-label form includes live preview panel updating in real time as settings change
- [ ] Audit log filter bar filters results and CSV export downloads the filtered set
- [ ] Platform analytics page renders funnel, pie chart, revenue chart, and metric cards
- [ ] All admin pages are protected by admin role route guard

---

### S12.15: Enterprise API -- API Key Auth, Rate Limiting & Documentation

**Points**: 4 | **Type**: backend

Configure the Enterprise API surface. Set up routing for `api.eusolicit.com/v1/*` that proxies to Client API endpoints. Implement API key authentication middleware: enterprise clients authenticate via `X-API-Key` header (API keys stored hashed in DB, associated with company_id and tier). API key CRUD endpoints for enterprise admins (create, list, revoke). Implement per-API-key rate limiting using Redis (token bucket algorithm) with configurable limits per tier. Auto-generate OpenAPI spec from FastAPI routes. Serve Swagger UI at `/v1/docs` and Redoc at `/v1/redoc`. Include request/response examples in OpenAPI schema annotations.

**Acceptance Criteria**:
- [ ] `api.eusolicit.com/v1/*` routes resolve to Client API endpoints
- [ ] `X-API-Key` header authentication validates against hashed keys in DB
- [ ] API key CRUD: `POST /v1/api-keys`, `GET /v1/api-keys`, `DELETE /v1/api-keys/{key_id}`
- [ ] Per-key rate limiting enforced via Redis token bucket; 429 returned when exceeded
- [ ] Rate limit headers included in responses (X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset)
- [ ] Swagger UI served at `/v1/docs`, Redoc at `/v1/redoc`
- [ ] OpenAPI spec includes request/response examples for all endpoints

---

### S12.16: Enterprise API Documentation Page & Usage Frontend

**Points**: 2 | **Type**: frontend

Build the Enterprise API documentation and management frontend. API documentation page embeds Swagger UI or Redoc (iframe or React component) for enterprise clients to explore endpoints interactively. API key management page (accessible to company admins on Enterprise tier): list active API keys (masked, showing last 4 characters), create new key (display full key once on creation), revoke key with confirmation dialog. Display current rate limit tier and usage stats.

**Acceptance Criteria**:
- [ ] API documentation page renders embedded Swagger UI or Redoc
- [ ] API key management page lists active keys with masked display
- [ ] New key creation shows full key exactly once with copy-to-clipboard
- [ ] Key revocation requires confirmation and takes effect immediately
- [ ] Rate limit tier and current usage stats displayed
- [ ] Page gated to Enterprise tier company admins

---

### S12.17: Performance Optimization, Load Testing & Security Audit

**Points**: 4 | **Type**: backend

Execute pre-launch technical hardening. Performance optimization: run `EXPLAIN ANALYZE` on all analytics queries, add missing indexes, optimize N+1 queries in API endpoints, enable response compression, configure CDN cache headers for static assets. Load testing: write Locust or k6 test scripts covering key user flows (search, opportunity detail, proposal generation, analytics dashboards), run against staging, document results (p50/p95/p99 latencies, throughput, error rates), and fix bottlenecks found. Security audit: complete OWASP Top 10 checklist, verify Kubernetes network policies (service-to-service isolation), rotate all secrets (DB passwords, API keys, JWT signing keys), verify CORS configuration, and confirm rate limiting on all public endpoints.

**Acceptance Criteria**:
- [ ] All analytics queries analyzed; slow queries optimized with appropriate indexes
- [ ] N+1 queries identified and resolved in API endpoints
- [ ] Load test scripts cover search, opportunity detail, proposal generation, and analytics flows
- [ ] Load test results documented with p50/p95/p99 latencies meeting targets (p95 < 500ms for reads)
- [ ] OWASP Top 10 checklist completed with all items addressed or risk-accepted
- [ ] Network policies verified for service isolation
- [ ] All production secrets rotated and documented in secret manager
- [ ] CORS and rate limiting configuration verified on all public endpoints

---

### S12.18: User Onboarding Flow & Launch Polish

**Points**: 3 | **Type**: fullstack

Build the first-run onboarding wizard and final launch polish. Onboarding: detect first login (user.onboarding_completed flag), launch a step-by-step overlay wizard: (1) complete company profile, (2) perform first opportunity search, (3) view first opportunity detail, (4) guided feature tour highlighting key UI areas (analytics, proposals, settings). Each step highlights the relevant UI area with a spotlight effect and instructional tooltip. User can skip or dismiss; completion sets the flag. Launch polish: review and fix any broken empty states across the app, ensure all error pages (404, 500, 403) are branded, verify email templates render correctly across clients, and add meta tags / OG tags for public pages.

**Acceptance Criteria**:
- [ ] First login triggers onboarding wizard overlay
- [ ] Wizard steps: company profile, first search, first opportunity, feature tour
- [ ] Each step spotlights the relevant UI area with instructional tooltip
- [ ] User can skip/dismiss wizard; completion sets onboarding_completed flag
- [ ] Wizard does not reappear after completion or dismissal
- [ ] Empty states reviewed and fixed across all pages
- [ ] Error pages (404, 500, 403) use branded templates
- [ ] Email templates verified across major email clients
- [ ] Public pages include correct meta tags and OG tags
